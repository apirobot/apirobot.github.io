---
title: Django + Elm. Аутентификация через JSON Web Token
date: 2017-08-28 2:00:00
---

*JSON Web Token (JWT)* - токен, который содержит минимально необходимую информацию для аутентификации и авторизации. В зашифрованном виде выглядит, как строка, которая состоит из трех частей:

![JWT](https://raw.githubusercontent.com/apirobot/apirobot.github.io/master/uploads/django-elm-auth-with-jwt/jwt.png)

1. <span style="color:#FE2959">**Header (заголовок).**</span> Содержит тип токена (в данном случае это JWT), и какой используется алгоритм шифрования.
2. <span style="color:#D83CFF">**Payload (нагрузка).**</span> Содержит всевозможные данные (например, информацию о пользователе или время, через которое токен будет не действителен).
3. <span style="color:#2BC5F6">**Signature (подпись).**</span> Нужна для проверки, что токен не подделан и выдан именно вашим сервисом (проверка валидности токена).

В сравнении с обычным токеном, *JWT самодостаточен (self-contained)*, потому что содержит минимально необходимую информацию о пользователе. Благодаря этому не нужны повторные обращения к базе данных. Обычный же токен - рандомная строка, и валидность этого токена проверяется через запросы к базе данных.

*JWT компактен*. Его легко можно передать через URL или HTTP header.

## Процесс аутентификации через JSON Web Token

![JWT auth process](https://raw.githubusercontent.com/apirobot/apirobot.github.io/master/uploads/django-elm-auth-with-jwt/jwt-auth-process.png)

Шаг 1. Пользователь вводит данные (имя пользователя, пароль, т.д.) и логинится в системе.<br/>
Шаг 2. Сервер создает JWT, в котором будет зашифрована информация о текущем пользователе.<br/>
Шаг 3. Сервер возвращает созданный токен пользователю. Этот токен нужно сохранить на локальном компьютере (в куках или local storage), иначе придется постоянно логиниться и получать токен по-новому.<br/>
Шаг 4. При запросе к серверу, пользователь отправляет JWT вместе с запросом через Authorization header:<br/>
```bash
Authorization: Bearer <token>
```
Шаг 5. Сервер проверяет валидность токена и получает информацию о пользователе.<br/>
Шаг 6. Сервер отправляет ответ пользователю.<br/>

## Аутентификация на примере

Напишем приложение с аутентификацией пользователя используя Django (backend) и Elm (frontend). Django будет предоставлять API для получения токена. Чтобы получить токен, необходимо отправить запрос с именем пользователя и паролем к API. Если данные пользователя введены корректно, то Django отправит в ответ сгенерированный JWT. Конечный результат на картинке:

![Result](https://raw.githubusercontent.com/apirobot/django-elm-auth-with-jwt/master/other/preview.gif)

Исходный код приложения: <a target="blank_" href="https://github.com/apirobot/django-elm-auth-with-jwt">https://github.com/apirobot/django-elm-auth-with-jwt</a>

## Backend с Django

Писать свой велосипед не обязательно, уже есть хорошая библиотека <a target="blank_" href="http://getblimp.github.io/django-rest-framework-jwt/">django-rest-framework-jwt</a> для Django, в которой реализована аутентификация с помощью JWT.

Для начала создаем и активируем виртуальное окружение:

```bash
$ virtualenv -p python3 venv
$ source venv/bin/activate
```

Теперь устанавливаем зависимости:

```bash
$ pip install django djangorestframework djangorestframework-jwt
```

Создаем Django проект:

```bash
$ django-admin startproject jwtme
$ mv jwtme backend
```

Обновляем настройки проекта:

```python
# backend/jwtme/settings.py

INSTALLED_APPS = [
    ...
    'rest_framework',
]

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': (
        'rest_framework_jwt.authentication.JSONWebTokenAuthentication',
        'rest_framework.authentication.SessionAuthentication',
        'rest_framework.authentication.BasicAuthentication',
    ),
}
```

Добавляем URL, по-которому будем отправлять POST запрос с данными пользователя (имя пользователя, пароль) и в ответе получать JWT при успешной аутентификации:

```python
# backend/jwtme/urls.py

from rest_framework_jwt.views import obtain_jwt_token

urlpatterns = [
    url(r'^api/v1/token/obtain/', obtain_jwt_token),
    ...
]
```

По умолчанию, токен протухает каждые 5 минут. Если нужно, можно изменить время протухания в настройках:

```python
# backend/jwtme/settings.py

import datetime

JWT_AUTH = {
    'JWT_EXPIRATION_DELTA': datetime.timedelta(days=1)
}
```

Все доступные настройки для django-rest-framework-jwt в <a target="blank_" href="http://getblimp.github.io/django-rest-framework-jwt/#additional-settings">документации</a>.

Чтобы в дальнейшем, при обращении к API через Elm, не возникла ошибка, как на скриншоте ниже, нужно добавить специальные response headers на сервере. Благодаря им можно выполнять кросс-доменные запросы.

![CORS error](https://raw.githubusercontent.com/apirobot/apirobot.github.io/master/uploads/django-elm-auth-with-jwt/cors-error.png)

Устанавливаем <a target="blank_" href="https://github.com/ottoyiu/django-cors-headers">django-cors-headers</a>:

```bash
$ pip install django-cors-headers
```

Обновляем настройки проекта:

```python
# backend/jwtme/settings.py

INSTALLED_APPS = [
    ...
    'corsheaders',
]

MIDDLEWARE = [
    ...
    'corsheaders.middleware.CorsMiddleware',
    'django.middleware.common.CommonMiddleware',
    ...
]

CORS_ORIGIN_ALLOW_ALL = True
```

Итак, запрос к серверу на получение токена и его ответ выглядит так (для запроса использовал <a target="blank_" href="https://httpie.org/">HTTPie</a>):

![Obtain token](https://raw.githubusercontent.com/apirobot/apirobot.github.io/master/uploads/django-elm-auth-with-jwt/obtain-token.png)

## Frontend с Elm

Создаем папку:

```bash
$ mkdir frontend && cd frontend
```

Устанавливаем зависимости:

```bash
$ elm-package install elm-lang/html
$ elm-package install elm-lang/http
```

Так как весь исходный код проекта будем хранить в папке *src*, то нужно явно указать это в файле frontend/elm-package.json:

```json
{
    "source-directories": [
        "src"
    ]
}
```

Создаем HTML и добавляем <a target="blank_" href="http://bulma.io/">bulma.css</a> стили, чтобы не писать свои:

```html
<!-- frontend/index.html -->

<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta http-equiv="X-UA-Compatible" content="ie=edge">
  <title>JWT Me</title>
  <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/font-awesome/4.7.0/css/font-awesome.min.css">
  <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/bulma/0.5.0/css/bulma.min.css">
</head>
<body>
  <div id="main"></div>
  <script src="elm.js"></script>
  <script>
    var app = Elm.Main.fullscreen();
  </script>
</body>
</html>
```

Создаем новый тип (union type) для токена:

```haskell
-- frontend/src/Data/Token.elm

module Data.Token exposing (..)


type Token
  = Token String


-- Распаковываем Token и получаем строку
tokenToString : Token -> String
tokenToString (Token token) =
  token
```

Создаем каркас приложения:

```haskell

-- frontend/src/Main.elm

module Main exposing (..)

import Html exposing (Html, div, text)

import Data.Token as Token exposing (Token)


main : Program Never Model Msg
main =
  Html.program
    { init = init
    , view = view
    , update = update
    , subscriptions = \_ -> Sub.none
    }



-- MODEL


type alias Model =
  { token : Maybe Token
  }


initialModel : Model
initialModel =
  { token = Nothing
  }


init : ( Model, Cmd Msg )
init =
  ( initialModel, Cmd.none )



-- MESSAGES


type Msg
  = NoOp



-- UPDATE


update : Msg -> Model -> ( Model, Cmd Msg )
update msg model =
  case msg of
    NoOp ->
      ( model, Cmd.none )



-- VIEW


view : Model -> Html Msg
view model =
  case model.token of
    Nothing ->
      Html.text
        "Token: none"

    Just token ->
      Html.text <|
        "Token: " ++ ( Token.tokenToString token )
```

Для запуска приложения будем использовать <a target="blank_" href="https://github.com/tomekwi/elm-live">elm-live</a>. Установите его, если еще этого не сделали, либо используйте другой способ.

Запускаем приложение:

```bash
$ elm-live --port=3000 --output=elm.js src/Main.elm --pushstate --open --debug
```

Напишем декодер для токена, который будет декодить JSON в Elm. Он понадобиться после получения ответа (response) с сервера после успешной аутентификации.

```haskell
-- frontend/src/Data/Token.elm

...
import Json.Decode as Decode exposing (Decoder)


decoder : Decoder Token
decoder =
  Decode.map Token Decode.string
```

Проверяем работу декодера с помощью *elm-repl*:

```haskell
> import Data.Token
> import Json.Decode
> Json.Decode.decodeString Data.Token.decoder "\"123.fds.OSf\""
Ok (Token "123.fds.OSf") : Result.Result String Data.Token.Token
```

Напишем функцию, которая создает POST запрос (request) для получения токена:

```haskell
-- frontend/src/Request/Token.elm

module Request.Token exposing (obtain)

import Json.Decode as Decode
import Json.Encode as Encode

import Http

import Data.Token as Token exposing (Token)


apiUrl : String -> String
apiUrl str =
  "http://localhost:8000/api/v1" ++ str


obtain : { r | username : String, password : String } -> Http.Request Token
obtain { username, password } =
  let
    body =
      Http.jsonBody <|
        Encode.object
          [ ("username", Encode.string username)
          , ("password", Encode.string password)
          ]
    -- Декодим JSON ответ с сервера ( {"token": "blah.blah.blah"} )
    -- в Elm ( Token "blah.blah.blah" )
    decoder =
      Decode.field "token" Token.decoder
  in
    Http.post (apiUrl "/token/obtain/") body decoder
```

Создадим компонент *Login*, в котором будем рендериться форма с авторизацией. В этой форме пользователь вводит *username* и *password*. После нажатия кнопки *Submit* выполняется запрос на получение токена к API (*/api/v1/token/obtain/*). После успешного получения токена, отправляем сообщение с полученным токеном для файла *Main.elm*:

```haskell
-- frontend/src/Components/Login.elm

module Components.Login exposing
  ( Model
  , initialModel
  , Msg
  , ExternalMsg(..)
  , update
  , view
  )

import Html exposing (..)
import Html.Attributes exposing (class, placeholder, type_)
import Html.Events exposing (onInput, onSubmit)
import Http

import Data.Token as Token exposing (Token)
import Request.Token


-- MODEL


type alias Model =
  { username : String
  , password : String
  }


initialModel : Model
initialModel =
  { username = ""
  , password = ""
  }


-- MESSAGES


type Msg
  = SubmitForm
  | SetUsername String
  | SetPassword String
  -- TokenObtained обрабатывает полученный ответ (response) с сервера.
  -- Если произошла ошибка, то получаем Http.Error.
  -- Если все прошло успешно, то получаем Token.
  | TokenObtained (Result Http.Error Token)


{- ExternalMsg - сообщение для главного файла (src/Main.elm)

Так как функцию `Login.update` мы будем вызывать в файле Main.elm,
то там нам нужно знать, получили ли мы токен с сервера или еще нет.
Сообщение `SetToken Token` как раз для этого. Этим сообщением мы передаем
токен в файл Main.elm, а затем добавляем его там в Model.

Если все еще не понятно, то после того, как мы закончим с кодом,
все должно стать на свои места.
-}
type ExternalMsg
  = NoOp
  | SetToken Token



-- UPDATE


update : Msg -> Model -> ( Model, Cmd Msg, ExternalMsg )
update msg model =
  case msg of
    -- Отправляет запрос к серверу на получение токена
    SubmitForm ->
      ( model
      , Http.send TokenObtained (Request.Token.obtain model)
      , NoOp
      )

    SetUsername username ->
      ( { model | username = username }, Cmd.none, NoOp )

    SetPassword password ->
      ( { model | password = password }, Cmd.none, NoOp )

    -- Токен успешно получен после запроса к серверу
    TokenObtained (Ok token) ->
      ( model
      , Cmd.none
      , SetToken token  -- Сообщение, в котором мы передаем token для Main.elm
      )

    -- Возникла ошибка при получении токена
    TokenObtained (Err err) ->
      ( model, Cmd.none, NoOp )



-- VIEW


view : Model -> Html Msg
view model =
  section [ class "section" ]
    [ div [ class "container" ]
      [ div [ class "columns" ]
        [ div [ class "column is-one-third" ]
          []
        , div [ class "column is-one-third" ]
          [ div []
            [ h1 [ class "title has-text-centered" ] [ text "Log in" ]
            , viewSignInForm
            ]
          ]
        , div [ class "column is-one-third" ]
          []
        ]
      ]
    ]


-- Форма с input для username, input для password и кнопкой Submit
viewSignInForm : Html Msg
viewSignInForm =
  Html.form [ onSubmit SubmitForm ]
    [ div [ class "field" ]
      [ label [ class "label" ]
        [ text "Username" ]
      , div [ class "control has-icons-left has-icons-right" ]
        [ input
          [ onInput SetUsername
          , class "input"
          , placeholder "Username"
          , type_ "text"
          ]
          []
        , span [ class "icon is-small is-left" ]
          [ i [ class "fa fa-user" ]
            []
          ]
        ]
      ]
    , div [ class "field" ]
      [ label [ class "label" ]
        [ text "Password" ]
      , div [ class "control has-icons-left has-icons-right" ]
        [ input
          [ onInput SetPassword
          , class "input"
          , placeholder "Password"
          , type_ "password"
          ]
          []
        , span [ class "icon is-small is-left" ]
          [ i [ class "fa fa-lock" ]
            []
          ]
        ]
      ]
    , div [ class "control" ]
      [ button [ class "button is-primary" ]
        [ text "Submit" ]
      ]
    ]
```

Подключаем компонент *Login* в *src/Main.elm*:

```haskell
-- frontend/src/Main.elm

...
import Components.Login as Login


-- MODEL


type alias Model =
  ...
  , loginModel : Login.Model
  }


initialModel : Model
initialModel =
  ...
  , loginModel = Login.initialModel
  }



-- MESSAGES


type Msg
  ...
  | LoginMsg Login.Msg



-- UPDATE


update : Msg -> Model -> ( Model, Cmd Msg )
update msg model =
  case msg of
    ...

    LoginMsg subMsg ->
      let
        ( newLoginModel, cmdFromLogin, msgFromLogin ) =
          Login.update subMsg model.loginModel

        newModel =
          -- Обрабатываем сообщение, полученное в `Components/Login.elm`
          case msgFromLogin of
            Login.NoOp ->
              model

            Login.SetToken token ->
              { model | token = Just token }
      in
        ( { newModel | loginModel = newLoginModel }
        , Cmd.map LoginMsg cmdFromLogin
        )



-- VIEW


view : Model -> Html Msg
view model =
  case model.token of
    -- Если токен отсутствует, то показываем форму с авторизацией
    Nothing ->
      Login.view model.loginModel
        |> Html.map LoginMsg

    ...
```

Изменим view для случая, когда токен уже получен. Кроме этого добавим новое сообщение *Logout*:

```haskell
-- frontend/src/Main.elm

...
import Html exposing (Html, section, div, button, text)
import Html.Attributes exposing (class, style)
import Html.Events exposing (onClick)


type Msg
  ...
  | Logout


update : Msg -> Model -> ( Model, Cmd Msg )
update msg model =
  case msg of
    ...

    Logout ->
      ( initialModel, Cmd.none )


view : Model -> Html Msg
view model =
  case model.token of
    Nothing ->
      ...

    Just token ->
      section [ class "section" ]
        [ div [ class "container" ]
          [ div [ class "columns" ]
            [ div [ class "column is-one-third" ]
              []
            , div [ class "column is-one-third" ]
              [ div []
                [ div [ class "notification is-success", style [("word-wrap", "break-word")] ]
                  [ text ( "You have been successfully logged in. Your token: " ++ (Token.tokenToString token) ) ]
                , button [ onClick Logout, class "button is-primary" ]
                  [ text "Logout" ]
                ]
              ]
            , div [ class "column is-one-third" ]
              []
            ]
          ]
        ]
```

Приложение теперь должно работать. Мы можем заполнить форму и получить токен при успешной аутентификации. Проблема в том, что после обновления страницы токен пропадает. Исправим это, сохранив токен в *localstorage* с помощью Javasript.

Для “общения” с Javascript через Elm создаем порт:

```haskell
-- frontend/src/Ports.elm

port module Ports exposing (storeToken)


port storeToken : Maybe String -> Cmd msg
```

Напишем Javascript код, в котором будем добавлять токен в localstorage при вызове порта storeToken:

```html
<!-- frontend/src/index.html -->  

<body>
  ...
  <script>
    var app = Elm.Main.fullscreen(localStorage.token || null);

    app.ports.storeToken.subscribe(function(token) {
      localStorage.token = token;
    });
  </script>
</body>
```

Добавим функцию, которая преобразует токен в строку, а затем отправит эту строку к Javascript:

```haskell
-- frontend/src/Data/Token.elm

...
import Ports


encode : Token -> Value
encode (Token token) =
  Encode.string token


storeToken : Token -> Cmd msg
storeToken token =
  encode token
    |> Encode.encode 0
    |> Just  -- Преобразуем String в Maybe String
    |> Ports.storeToken  -- Отправляем Maybe String к Javascript через порт
```

Смысла в одном сохранении нет. Кроме сохранения токена в localstorage, нужно его еще от туда как-то получить после обновления страницы. Это уже обратное “общение”. Из Javascript в Elm. Добавим функцию, которая декодит JSON в Token:

```haskell
-- frontend/src/Data/Token.elm

...

decodeTokenFromStore : Value -> Maybe Token
decodeTokenFromStore json =
  json
    |> Decode.decodeValue Decode.string  -- Декодим json и получаем Result String String
    |> Result.toMaybe  -- Преобразуем Result String String в Maybe String
    |> Maybe.andThen (Decode.decodeString decoder >> Result.toMaybe)  -- Пропускаем String через Token.decoder и получаем Maybe Token
```

Теперь сохраняем токен в localstorage. Для этого обновляем случай с успешным получением токена в компоненте *Login*:

```haskell
-- frontend/src/Components/Login.elm

...

update : Msg -> Model -> ( Model, Cmd Msg, ExternalMsg )
update msg model =
  case msg of
    ...

    TokenObtained (Ok token) ->
      ( model
      , Token.storeToken token
      , SetToken token
      )
```

Получаем токен из localstorage при инициализации в Main.elm. Для этого используем *флаги (flags)* и функцию *Html.programWithFlags*. Флагами называют данные, которые передаются из Javascript в Elm.

```haskell

-- frontend/src/Main.elm

...
import Json.Encode as Encode exposing (Value)


main : Program Value Model Msg
main =
  Html.programWithFlags
    { init = init
    , view = view
    , update = update
    , subscriptions = \_ -> Sub.none
    }


initialModel : Maybe Token -> Model
initialModel token =
  { token = token
  , loginModel = Login.initialModel
  }


init : Value -> ( Model, Cmd Msg )
init value =
  ( initialModel (Token.decodeTokenFromStore value), Cmd.none )
```

Последний штрих. Обновляем сообщение Logout, в котором удаляем токен из Model и localstorage:

```haskell
-- frontend/src/Main.elm

...
import Ports

update : Msg -> Model -> ( Model, Cmd Msg )
update msg model =
  case msg of
    ...

    Logout ->
      ( initialModel Nothing, Ports.storeToken Nothing )
```

## Конец

Проблема? [Используй GitHub Issues](https://github.com/apirobot/django-elm-auth-with-jwt).

---
title: Django + Vue. Реализуем вход через Google
date: 2018-03-09 11:00:00
---

Никто не любит при регистрации на сайте вводить каждый раз одно и то же: имя пользователя, электронную почту и т.д. Либо постоянно создавать и запоминать новые пароли. По этой причине, вход через сторонние приложения вроде Google, Facebook или VK очень популярен.

Такие сторонние приложения используют протокол OAuth2. В статье я не буду объяснять, что это за протокол и как его реализовать. Вместо этого реализуем вход на сайт через Google использую уже готовые библиотеки. Бэкэнд напишем на Django и Django Rest Framework, а фронтэнд на Vue.js

Конечный результат:

<p align="center">
  <img src="https://raw.githubusercontent.com/apirobot/django-vue-google-auth/master/other/preview.gif" alt="django-vue-google-auth">
</p>

Исходный код урока: <a target="blank" href="https://github.com/apirobot/django-vue-google-auth">https://github.com/apirobot/django-vue-google-auth</a>

# Создаем Google приложение

Прежде чем приступить к написанию кода, нужно создать приложение в консоли для разработчика. Через это приложение будет происходить аутентификация на нашем сайте.

- Переходим по ссылке <a target="blank" href="https://console.developers.google.com/apis/dashboard">https://console.developers.google.com/apis/dashboard</a>
- Нажимаем кнопку “Создать проект”.

![google-app-1](https://raw.githubusercontent.com/apirobot/apirobot.github.io/master/uploads/django-vue-google-auth/google-app-1.png)

- Вводим название проекта и нажимаем на кнопку “Создать”.

![google-app-2](https://raw.githubusercontent.com/apirobot/apirobot.github.io/master/uploads/django-vue-google-auth/google-app-2.png)

- После создания проекта нажимаем на кнопку “Включить API и сервисы”.

![google-app-3](https://raw.githubusercontent.com/apirobot/apirobot.github.io/master/uploads/django-vue-google-auth/google-app-3.png)

- Через поиск находим Google+ API и нажимаем на кнопку “Включить”.

![google-app-4](https://raw.githubusercontent.com/apirobot/apirobot.github.io/master/uploads/django-vue-google-auth/google-app-4.png)

- Создаем учетную запись.

![google-app-5](https://raw.githubusercontent.com/apirobot/apirobot.github.io/master/uploads/django-vue-google-auth/google-app-5.png)

- Настраиваем учетные данные.

![google-app-6](https://raw.githubusercontent.com/apirobot/apirobot.github.io/master/uploads/django-vue-google-auth/google-app-6.png)

- Добавляем `http://localhost:8080` в разрешенные источники. Под этим адресом будет работать Vue.js приложение.

![google-app-7](https://raw.githubusercontent.com/apirobot/apirobot.github.io/master/uploads/django-vue-google-auth/google-app-7.png)

- Открываем созданный клиент.

![google-app-8](https://raw.githubusercontent.com/apirobot/apirobot.github.io/master/uploads/django-vue-google-auth/google-app-8.png)

- Копируем идентификатор клиента и секрет клиента. В дальнейшем это понадобится.

![google-app-9](https://raw.githubusercontent.com/apirobot/apirobot.github.io/master/uploads/django-vue-google-auth/google-app-9.png)

# Backend

Что будем использовать:

1. <a target="blank" href="https://www.djangoproject.com/">django</a>;
2. <a target="blank" href="http://www.django-rest-framework.org/">django-rest-framework</a> для создания REST API;
3. <a target="blank" href="https://github.com/pennersr/django-allauth">django-allauth</a> и <a target="blank" href="https://github.com/Tivix/django-rest-auth">django-rest-auth</a> для аутентификации в системе. django-rest-auth предоставляет уже рабочие REST API endpoints (конечные точки). Не нужно париться по поводу реализации своей системы входа и регистрации;
4. <a target="blank" href="https://github.com/GetBlimp/django-rest-framework-jwt">django-rest-framework-jwt</a> для аутентификации через JWT (JSON Web Token). Этот токен создается при успешной аутентификации пользователя в системе, после чего токен передается на фронтэнд пользователю. Теперь, при каждом запросе к бэкэнду с фронтэнда, токен прикрепляется к заголовку запроса. Это такой способ идентификации. Подробнее про JSON Web Token <a target="blank" href="http://apirobot.me/posts/django-elm-auth-with-jwt">здесь</a>;
5. <a target="blank" href="https://github.com/ottoyiu/django-cors-headers">django-cors-header</a> для поддержки кроссдоменных запросов. Без нее мы не сможем делать запросы с фронтэнда (`http://localhost:8080`) на бэкэнд (`http://localhost:8000`).

## Создаем проект и устанавливаем зависимости

Создаем корневую папку проекта:

```bash
$ mkdir django-vue-auth
$ cd django-vue-auth
```

Создаем папку для бэкэнда:

```bash
$ mkdir backend
$ cd backend
```

Устанавливаем зависимости:

```bash
$ pipenv install django djangorestframework djangorestframework-jwt django-allauth django-rest-auth django-cors-headers
```

Активируем виртуальное окружение:

```bash
$ pipenv shell
```

Создаем проект:

```bash
$ django-admin startproject thisisproject .
```

## Настраиваем settings.py

Добавляем настройки в файл:

```python
# thisisproject/settings.py

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'django.contrib.sites',  # don't forget

    'corsheaders',
    'rest_framework',
    'rest_framework.authtoken',
    'rest_auth',
    'rest_auth.registration',
    'allauth',
    'allauth.account',
    'allauth.socialaccount',
    'allauth.socialaccount.providers.google',
]

SITE_ID = 1

MIDDLEWARE = [
    ...
    'corsheaders.middleware.CorsMiddleware',
    'django.middleware.common.CommonMiddleware',
    ...
]

CORS_ORIGIN_ALLOW_ALL = True

AUTHENTICATION_BACKENDS = [
    'django.contrib.auth.backends.ModelBackend',
    'allauth.account.auth_backends.AuthenticationBackend',
]

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': (
        'rest_framework_jwt.authentication.JSONWebTokenAuthentication',
        'rest_framework.authentication.SessionAuthentication',
        'rest_framework.authentication.BasicAuthentication',
    ),
}

# По умолчанию, `django-rest-auth` использует аутентификацию через
# обычные токены. Нам нужна аутентификация через JWT токены.
REST_USE_JWT = True
```

Делаем миграции:

```bash
$ python manage.py migrate
```

## Реализуем вход через Google

Создаем Google приложение на сайте. Для этого понадобится идентификатор клиента и секрет клиента. Зайдите в панель администратора и создайте приложение в разделе Social Applications как на картинке:

![django-admin-social-app](https://raw.githubusercontent.com/apirobot/apirobot.github.io/master/uploads/django-vue-google-auth/django-admin-social-app.png)

Создаем отдельное приложение для аутентификации:

```bash
$ django-admin startapp authentication
```

Добавляем представление, через которое будет происходить аутентификация в Google:

```python
# authentication/views.py

from allauth.socialaccount.providers.google.views import GoogleOAuth2Adapter
from rest_auth.registration.views import SocialLoginView


class GoogleLogin(SocialLoginView):
    adapter_class = GoogleOAuth2Adapter
```

Добавляем это представление в urls приложения:

```python
# authentication/urls.py

from django.urls import path

from . import views

urlpatterns = [
    path('google/', views.GoogleLogin.as_view(), name='google_login')
]
```

Обновляем urls проекта:

```python
# thisisproject/urls.py

from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('auth/', include('authentication.urls')),
]
```

Теперь, чтобы проверить, работает ли аутентификация через Google, нужно отправить POST запрос по ссылке `http://localhost:8000/auth/google/` и в теле этого запроса передать access token. Access token мы будем получать позже через Vue.js. Но для проверки работоспособности бэкэнда, получим access token через <a target="blank" href="https://developers.google.com/oauthplayground/">Google OAuth2 Playground</a>:

![google-oauth-playground-1](https://raw.githubusercontent.com/apirobot/apirobot.github.io/master/uploads/django-vue-google-auth/google-oauth-playground-1.png)
![google-oauth-playground-2](https://raw.githubusercontent.com/apirobot/apirobot.github.io/master/uploads/django-vue-google-auth/google-oauth-playground-2.png)

После получения токена доступа, делаем запрос к серверу:

![http-request](https://raw.githubusercontent.com/apirobot/apirobot.github.io/master/uploads/django-vue-google-auth/http-request.png)

В ответ сервер вернул информацию о пользователе, а также создал новый JWT токен.

## Создаем приложение users

Библиотека django-allauth, после успешной аутентификации через Google, создает нового пользователя и новый социальный аккаунт:

![django-admin-social-account](https://raw.githubusercontent.com/apirobot/apirobot.github.io/master/uploads/django-vue-google-auth/django-admin-social-account.png)

Однако, эта библиотека по умолчанию сохраняет только email, username, first_name и last_name пользователя. Но что если нужно сохранить больше полей? Например фотографию.

Этим сейчас и займемся. Создадим свою модель User, добавим туда поле photo и после успешной аутентификации через Google, сохраним в это поле ссылку на фотографию пользователя.

Создаем приложение:

```bash
$ django-admin startapp users
```

Создаем модель User:

```python
# users/models.py

from django.contrib.auth.models import AbstractUser
from django.db import models


class User(AbstractUser):
    photo = models.URLField(blank=True)

    def __str__(self):
        return self.username
```

После успешной аутентификации, библиотека django-allauth создает социальный аккаунт и прикрепляет к нему пользователя. Перехватим сохранения социального аккаунта через сигналы и обновим ссылку на фотографию у пользователя:

```python
# users/signals.py

from django.db.models.signals import post_save
from django.dispatch import receiver

from allauth.socialaccount.models import SocialAccount


@receiver(post_save, sender=SocialAccount)
def add_extra_data_to_the_user(sender, instance, created, *args, **kwargs):
    instance.user.photo = instance.extra_data['picture']
    instance.user.save()
```

Не забудьте импортировать сигналы:

```python
# users/apps.py

from django.apps import AppConfig


class UsersConfig(AppConfig):
    name = 'users'

    def ready(self):
        import users.signals
```

Так как UserSerializer, который предоставляет библиотека django-rest-auth не содержит поле photo, то создадим в таком случае свой:

```python
# users/serializers.py

from rest_framework import serializers

from .models import User


class UserSerializer(serializers.ModelSerializer):

    class Meta:
        model = User
        fields = ('username', 'email', 'first_name', 'last_name', 'photo')
        read_only_fields = ('email', )
```

Обновляем настройки проекта:

```python
# thisisproject/settings.py

INSTALLED_APPS = [
    ...
    'users.apps.UsersConfig',
]

AUTH_USER_MODEL = 'users.User'

REST_AUTH_SERIALIZERS = {
    'USER_DETAILS_SERIALIZER': 'users.serializers.UserSerializer',
}
```

# Frontend

Что будем использовать:

1. <a target="blank" href="https://vuejs.org/">vue.js</a>;
2. <a target="blank" href="https://github.com/vuejs/vue-cli">vue-cli</a> для генерации vue.js проекта;
3. <a target="blank" href="https://github.com/axios/axios">axios</a> для отправления запросов к бэкэнду;
4. <a target="blank" href="https://github.com/phanan/vue-google-signin-button">vue-google-signin-button</a> для входа через Google и получения токена доступа.

## Создаем проект и устанавливаем зависимости

Переходим в корневую папку проекта:

```bash
$ cd django-vue-auth
```

Создаем проект:

```bash
$ vue init webpack thisisproject
```

![vue-cli](https://raw.githubusercontent.com/apirobot/apirobot.github.io/master/uploads/django-vue-google-auth/vue-cli.png)

Переименовываем папку:

```bash
$ mv thisisproject frontend
$ cd frontend
```

Устанавливаем зависимости:

```bash
$ npm install vue-google-signin-button axios --save
```

Регистрируем vue-google-signin-button:

```javascript
// src/main.js

import Vue from 'vue'
import GSignInButton from 'vue-google-signin-button'
import App from './App'

Vue.config.productionTip = false

Vue.use(GSignInButton)

/* eslint-disable no-new */
new Vue({
  el: '#app',
  components: { App },
  template: '<App/>'
})
```

Добавляем скрипт в index.html, без которого библиотека vue-google-signin-button работать не будет:

```html
<!-- index.html -->

<!DOCTYPE html>
<html>
  <head>
    ...
  </head>
  <body>
    ...
    <script src="https://apis.google.com/js/api:client.js"></script>
</html>
```

Свои стили писать не будем, поэтому добавим уже готовые от <a target="blank" href="https://picturepan2.github.io/spectre/">spectre.css</a>:

```html
<!-- index.html -->

<!DOCTYPE html>
<html>
  <head>
    ...
    <link rel="stylesheet" href="https://unpkg.com/spectre.css/dist/spectre.min.css">
  </head>
  <body>
    ...
</html>
```

## Реализуем вход

Обновляем App.vue:

```html
<!-- src/App.vue -->

<template>
  <div id="app">
    <div class="container">
      <div class="columns" style="margin-top: 100px;">
        <div class="column col-2 centered">
          <!-- Если user - пустой объект, то отображаем кнопку
          входа через Google -->
          <g-signin-button
            v-if="isEmpty(user)"
            :params="googleSignInParams"
            @success="onGoogleSignInSuccess"
            @error="onGoogleSignInError"
          >
            <button class="btn btn-block btn-success">
              Google Signin
            </button>
          </g-signin-button>
          <user-panel v-else :user="user"></user-panel>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import axios from 'axios'

import UserPanel from '@/components/UserPanel'

export default {
  name: 'App',
  components: {
    UserPanel
  },
  data () {
    return {
      user: {},
      googleSignInParams: {
        client_id: 'your_client_id_here'
      }
    }
  },
  methods: {
    onGoogleSignInSuccess (resp) {
      const token = resp.Zi.access_token
      // После успешного входа через Google,
      // отправляем токен доступа на бэкэнд и получаем взамен
      // пользователя и JWT токен
      // P.S. JWT токен в нашем примере не нужен, поэтому его не сохраняем
      axios.post('http://localhost:8000/auth/google/', {
        access_token: token
      })
        .then(resp => {
          this.user = resp.data.user
        })
        .catch(err => {
          console.log(err.response)
        })
    },
    onGoogleSignInError (error) {
      console.log('OH NOES', error)
    },
    isEmpty (obj) {
      return Object.keys(obj).length === 0
    }
  }
}
</script>
```

Добавляем панель для пользователя, которая отображается после успешного входа:

```html
<!-- src/components/UserPanel.vue -->

<template>
  <div class="panel">
    <div class="panel-header text-center">
      <figure class="avatar avatar-lg">
        <img :src="user.photo" alt="Avatar">
      </figure>
      <div class="panel-title h5 mt-10">{{ user.first_name }} {{ user.last_name }}</div>
      <div class="panel-subtitle">{{ user.username }}</div>
    </div>
    <nav class="panel-nav">
      <ul class="tab tab-block">
        <li class="tab-item active">
          <a>Profile</a>
        </li>
      </ul>
    </nav>
    <div class="panel-body">
      <div class="tile tile-centered">
        <div class="tile-content">
          <div class="tile-title">E-mail</div>
          <div class="tile-subtitle">{{ user.email }}</div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'UserPanel',
  props: ['user']
}
</script>
```

# Конец

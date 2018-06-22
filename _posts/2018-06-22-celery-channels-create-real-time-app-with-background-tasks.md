---
title: Celery + Channels = <3. Создаем реал-тайм приложение с бэкграунд тасками
date: 2018-06-22 3:00:00
---

<p align="center">
  <img src="https://raw.githubusercontent.com/apirobot/apirobot.github.io/master/uploads/celery-channels-create-real-time-app-with-background-tasks/cover.png" alt="django-channels-celery-jokes">
</p>

В статье создадим веб-приложение, которое в бэкграунде делает запросы к API со случайными шутками каждые 15 секунд, затем отправляет шутку пользователю через WebSocket. Для реализации приложения будем использовать: django, celery и channels. Celery для бэкграунд задач. Channels для передачи сообщений через WebSocket.

Конечный результат на картинке:

<p align="center">
  <img src="https://raw.githubusercontent.com/apirobot/django-channels-celery-jokes/master/preview.gif" alt="django-channels-celery-jokes">
</p>

Исходный код проекта: <a target="blank" href="https://github.com/apirobot/django-channels-celery-jokes">https://github.com/apirobot/django-channels-celery-jokes</a>

# Содержание

- [Что такое celery](#что-такое-celery)
- [Что такое channels](#что-такое-channels)
- [Создаем проект](#создаем-проект)
- [Создаем приложение](#создаем-приложение)
- [Создаем бэкграунд задачу](#создаем-бэкграунд-задачу)
- [Подключаем библиотеку channels](#подключаем-библиотеку-channels)
- [Заключение](#заключение)

# Что такое celery

Celery - инструмент для управления очередями задач. Задачей может быть все что угодно: запрос к API, парсинг веб-страницы, какие-то сложные долговременные вычисления и т.д.

Эти задачи выполняются в отдельном процессе, поэтому они не блокируют ваш основной процесс, в котором запущено Django приложение. Это полезно когда вы, для примера, делаете запрос к внешнему API. Вы не можете контролировать внешний API, поэтому получение ответа от API может занять сколько угодно времени (15 сек, 30 сек, минута?). Таким образом, даже если получение ответа от API займет около минуты, то это не затормозит работу веб-приложения.

Как работает celery:


- У вас есть один или несколько работников (workers). Работник запускается в отдельном процессе и его цель - выполнять задачи.
- Django приложение должно как-то передавать задачи работникам. Для этого используются брокеры (brokers). Брокер - очередь, в которой хранятся задачи.
- После того, как брокер получил задачу от Django приложения, он помещает их в очередь и начинает передавать задачи работникам.
- После того, как работник выполнил задачу, он помещает результат в так называемый results backend. Эти результаты затем можно извлечь из Django приложения. На практике, вы можете использовать тот же самый брокер для хранения результатов.

Самые популярные реализации брокеров - Redis и RabbitMQ. В нашем приложении мы будем использовать Redis.

# Что такое channels

Django - классический MVC фреймворк, который работает по принципу запрос/ответ. Клиент (браузер) делает запрос к серверу (Django приложению), сервер обрабатывает запрос и генерирует ответ клиенту в виде HTML страницы.

Цель библиотеки channels - расширить возможности фреймворка Django, позволяя обрабатывать запросы асинхронно и создавать долговременные соединения с сервером используя протоколы: WebSocket, MQTT и т.д.

Это открывает возможности для создания реал-тайм приложений вроде чатов, push-уведомлений, коммуникацией с интернетом вещей (Internet of Things) и прочее.

Библиотеки celery и channels - разные библиотеки, которые используются для разных целей. Но у них есть и сходства. Библиотека channels, как и celery, тоже способна выполнять бэкграунд задачи, но у библиотеки channels не так много фич, в отличии от celery. Например, в celery есть механизм повторного выполнения задачи при ошибке. Это полезно когда вы делаете запросы к внешнему API, так как API периодически возвращает timeout ошибку. В таких ситуациях лучше перезапустить задачу и сделать новый запрос.

Более подробно библиотеку channels разберем дальше в статье.

# Создаем проект

Установить все зависимости и запустить их в таком проекте - довольно проблематично. Например, процесс установки брокера Redis для разных операционных систем отличается.

Это идеальный случай, чтобы использовать докер. Если вы до этого не использовали его, то можете почитать о нем <a target="blank" href="https://docs.docker.com/get-started/">здесь</a>. Я использую докер во всех своих проектах, кроме какого-нибудь hello world. Так что рекомендую.

В репозитории с исходным кодом есть ветка boilerplate, в которой хранится уже настроенный проект. Склонируйте эту ветку:

```bash
$ git clone -b boilerplate https://github.com/apirobot/django-channels-celery-jokes.git
```

Перейдите в директорию с проектом и запустите docker-compose:

```bash
$ docker-compose up
```

После создания изображений и запуска контейнеров, все должно работать. Магия, правда?

# Создаем приложение

Создайте приложение jokes:

```bash
$ django-admin startapp jokes
```

Добавьте приложение в список INSTALLED_APPS:

```python
# thisisproject/settings.py

INSTALLED_APPS = [
    ...
    "jokes.apps.JokesConfig",
]
```

# Создаем бэкграунд задачу

В папке jokes создайте файл tasks.py:

```bash
$ touch jokes/tasks.py
```

Celery проходит по всем приложениям и автоматически распознает задачи в файлах с именем tasks.py.

Создадим простую задачу, которая делает запрос к API со случайными шутками, и затем логгирует полученную шутку:

```python
# jokes/tasks.py
import logging
from celery import shared_task
import requests

logger = logging.getLogger()


@shared_task
def get_random_joke():
    res = requests.get("http://api.icndb.com/jokes/random").json()
    joke = res["value"]["joke"]
    logger.info(joke)
```

Сделаем так, чтобы эта задача выполнялась в бэкграунде каждые 15 секунд. Для этого добавим в настройках CELERY_BEAT_SCHEDULE:

```python
# thisisproject/settings.py

...
CELERY_BEAT_SCHEDULE = {
    "get_random_joke": {
        "task": "jokes.tasks.get_random_joke",
        "schedule": 15.0
    }
}
```

После запуска проекта, каждые 15 секунд в консоле должна логгироваться новая шутка:

![Task](https://raw.githubusercontent.com/apirobot/apirobot.github.io/master/uploads/celery-channels-create-real-time-app-with-background-tasks/task-log.png)

# Подключаем библиотеку channels

Получение шутки и логгирование ее в консоль это хорошо, но не достаточно. Нужно чтобы пользователь мог ее прочесть в браузере. Для этого нам понадобится библиотека channels.

Как это будет работать:

1. После того как пользователь зашел на страницу с шутками, он автоматически соединяется с сервером через вебсокет и добавляется в группу. Назовем эту группу `jokes`.
2. Когда сервер в бэкгруанд задаче `get_random_jokes` получает шутку, он ее логгирует в консоль, а затем отправляет эту шутку по вебсокету всем пользователям, которые подключены к группе `jokes`.
3. Так как наш пользователь подключен к группе `jokes`, то он получит эту шутку сразу же.

Магия в том, что все это будет работать в реальном времени, без перезагрузки страницы.

Начнем с создания страницы где пользователь будет подключаться к вебсокету и получать шутку от сервера.

Создаем шаблон страницы:

```html
<!-- jokes/templates/jokes/index.html -->
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta http-equiv="X-UA-Compatible" content="ie=edge">
  <title>Jokes</title>
  <!-- Bulma styles -->
  <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/bulma/0.7.1/css/bulma.min.css">
</head>
<body>
  <section class="section">
    <div class="container">
      <div class="columns">
        <div class="column is-6 is-offset-3">
          <div id="jokes"></div>
        </div>
      </div>
    </div>
  </section>
  <script>
    // Создаем объект WebSocket. При его создании
    // автоматически происходит попытка открыть соединение с сервером.
    const jokesws = new WebSocket(
      'ws://' + window.location.host + '/ws/jokes/'
    )

    /**
     * Добавляет шутку в начало содержимого элемента с идентификатором #jokes.
     * @param {string} joke - Текст шутки.
     */
    const addJoke = (joke) => {
      document.querySelector('#jokes').innerHTML = `
        <article class="message is-success">
          <div class="message-header">
            <p>Joke</p>
          </div>
          <div class="message-body">${joke}</div>
        </article>
      ` + document.querySelector('#jokes').innerHTML
    }

    // Событие срабатывает при получении сообщения от сервера.
    jokesws.onmessage = (event_) => {
      const joke = event_.data
      addJoke(joke)
    }

    // Событие срабатывает при закрытии соединения.
    jokesws.onclose = (event_) => {
      console.error('Jokes socket closed')
    }
  </script>
</body>
</html>
```

Создаем представление:

```python
# jokes/views.py
from django.shortcuts import render

def index(request):
    return render(request, "jokes/index.html", {})
```

Добавляем urlpattern:

```python
# jokes/urls.py
from django.urls import path
from . import views

urlpatterns = [path("", views.index, name="index")]
```

Обновляем корневые urlpatterns:

```python
# thisisproject/urls.py
from django.contrib import admin
from django.urls import include, path

urlpatterns = [
    path("admin/", admin.site.urls),
    path("jokes/", include("jokes.urls"))
]
```

Если вы зайдете на страницу `http://localhost:8000/jokes/`, то никакую шутку вы не получите. Представление пытается открыть соединение с сервером через вебсокет по ссылке `ws://localhost:8000/ws/jokes/`, но мы еще не создали потребителя (consumer), который принимает соединения через вебсокет. Если вы откроете консоль разработчика в браузере, то увидите такую ошибку:

```bash
WebSocket connection to 'ws://localhost:8000/ws/jokes/' failed:
Connection closed before receiving a handshake response
```

Теперь самая сложная часть - создать потребителя. Нам нужен потребитель, который принимает вебсокет соединения, добавляет клиентов в группу `jokes`, а затем рассылает шутки всем клиентам, которые подключены к группе:

```python

# jokes/consumers.py
from asgiref.sync import async_to_sync
from channels.generic.websocket import WebsocketConsumer


class JokesConsumer(WebsocketConsumer):
    def connect(self):
        # Подключает канал с именем `self.channel_name`
        # к группе `jokes`
        async_to_sync(self.channel_layer.group_add)(
            "jokes", self.channel_name
        )
        # Принимает соединение
        self.accept()

    def disconnect(self, close_code):
        # Отключает канал с именем `self.channel_name`
        # от группы `jokes`
        async_to_sync(self.channel_layer.group_discard)(
            "jokes", self.channel_name
        )

    # Метод `jokes_joke` - обработчик события `jokes.joke`
    def jokes_joke(self, event):
        # Отправляет сообщение по вебсокету
        self.send(text_data=event["text"])
```

Когда клиент подключается к вебсокету, создается новый объект потребителя `JokesConsumer`. У каждого объекта потребителя есть свое уникальное имя - `channel_name`.

 `channel_layer` позволяет передавать информацию разным потребителям. Если передается какое-либо сообщение через `channel_layer` , то потребитель принимает его, если это сообщение было адресовано конкретно ему (каналу с именем `channel_name`), либо группе, к которой подключен потребитель.

Метод `jokes_joke` - обработчик события, которое можно отправить через `channel_layer`. Метод вызывается в том случае, если было передано сообщение типа `jokes.joke` через `channel_layer` каналу с именем `self.channel_name`, либо группе `jokes` (так как канал с именем `self.channel_name` был подключен к группе `jokes` в методе `connect`).

Вернемся к бэкграунд задаче `get_random_joke`. После получения шутки, ее нужно передать всем клиентам, которые подключены к группе `jokes`:

```python

# jokes/tasks.py
import logging
from asgiref.sync import async_to_sync
from celery import shared_task
from channels.layers import get_channel_layer
import requests

channel_layer = get_channel_layer()
logger = logging.getLogger()


@shared_task
def get_random_joke():
    res = requests.get("http://api.icndb.com/jokes/random").json()
    joke = res["value"]["joke"]
    logger.info(joke)
    # Передаем сообщение типа `jokes.joke` через channel_layer
    # всем потребителям, которые подключены к группе `jokes`.
    async_to_sync(channel_layer.group_send)(
        "jokes", {"type": "jokes.joke", "text": joke}
    )
```

Осталось добавить маршрут для потребителя, чтобы когда пользователь пытался подключиться к вебсокету по ссылке `ws://localhost:8000/ws/jokes/`, он подключался к потребителю `JokesConsumer`. Создаем файл routing.py в приложении jokes:

```python
# jokes/routing.py
from django.urls import path
from . import consumers

websocket_urlpatterns = [
    path("ws/jokes/", consumers.JokesConsumer)
]
```

Обновляем корневые маршруты:

```python
# thisisproject/routing.py
from channels.routing import ProtocolTypeRouter, URLRouter
import jokes.routing

application = ProtocolTypeRouter(
    {"websocket": URLRouter(jokes.routing.websocket_urlpatterns)}
)
```

# Заключение

celery и channels полезные библиотеки, но они могут сильно усложнить приложение, поэтому используйте с умом.

> С большой силой приходит большая ответственность

Если вы до этого не слышали про celery и channels, не работали с докером и у вас мало опыта с фрейморком django, то не пытайтесь разобраться со всем этим одновременно. Я не раз совершал такую ошибку. Брать слишком много неизвестных технологий и начинать писать на них что-то - путь в никуда.

P.S. Я создал группу вконтакте <a target="blank" href="https://vk.com/apirobotgroup">https://vk.com/apirobotgroup</a>. Пока еще не решил, что буду с ней делать. Если есть желание, вступайте. Если есть предложения, пишите.

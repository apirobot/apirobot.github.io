---
title: Django + Vue. Как создать и обработать API. Часть 1
date: 2017-07-08 8:00:00
---

В этом уроке, который состоит из двух частей, я расскажу о том, как можно создать и обработать API используя <a target="blank" href="http://www.django-rest-framework.org/">django rest framework</a> и <a target="blank" href="https://vuejs.org/">vue.js</a>. В первой части урока мы займемся бэкэндом, во второй - фронтэндом.

Исходный код урока: <a target="blank" href="https://github.com/apirobot/django-vue-simplenote">https://github.com/apirobot/django-vue-simplenote</a>

---

В данном уроке мы попытаемся создать небольшое приложение с заметками:

![Result](https://raw.githubusercontent.com/apirobot/django-vue-simplenote/master/other/preview.gif)

## Настройка

Начнем с установки django и django rest framework (будем считать, что вы уже создали и активировали виртуальное окружение):
```bash
$ pip install django djangorestframework
```

Создадим нашу рабочую папку и папку backend, в которой будет находится проект Django. В дальнейшем мы еще создадим папку frontend, в которой будет наш Vue.js код.

```bash
$ mkdir django-vue-simplenote
$ cd django-vue-simplenote
$ mkdir backend
```

Создаем новый проект и приложение notes:
```bash
$ django-admin startproject simplenote backend/
$ cd backend
$ django-admin startapp notes
```

Добавляем django rest framework и notes в список установленных приложений (INSTALLED_APPS):

```python
# backend/simplenote/settings.py

INSTALLED_APPS = [
    ...
    'django.contrib.messages',
    'django.contrib.staticfiles',

    'rest_framework',

    'notes',
]
```

Структура проекта должна выглядить примерно так (я удалил ненужные файлы, которые создал django автоматически, при создании приложения notes):

```bash
django-vue-simplenote
└── backend
    ├── manage.py
    ├── notes
    │   ├── __init__.py
    │   ├── migrations
    │   │   └── __init__.py
    │   ├── models.py
    │   └── views.py
    └── simplenote
        ├── __init__.py
        ├── settings.py
        ├── urls.py
        └── wsgi.py
```

## Реализация

Начнем с создания простой модели:

```python
# backend/notes/models.py
from django.db import models


class Note(models.Model):
    title = models.CharField(max_length=255)
    body = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)
```

Теперь давайте напишем сериалайзер (serializer) для нашей модели. Сериализация позволяет нам преобразовать сложную модель из нашей базы данных, в простой тип данных вроде JSON или XML. Этот наш JSON мы затем будем получать и обрабатывать с помощью Vue.js.

```python
# backend/notes/serializers.py
from rest_framework import serializers

from .models import Note


class NoteSerializer(serializers.ModelSerializer):

    class Meta:
        model = Note
        fields = ('id', 'title', 'body', 'created_at')
```

Создадим представление (view) используя ModelViewSet, который уже снабжает нас create, update, list, retrieve, delete методами:

```python
# backend/notes/views.py
from rest_framework import viewsets

from .models import Note
from .serializers import NoteSerializer


class NoteViewSet(viewsets.ModelViewSet):
    queryset = Note.objects.all().order_by('-created_at')
    serializer_class = NoteSerializer
```

Так как мы используем ViewSet, то нам нет необходимости самим настраивать URLs. Можно использовать готовый класс Router, который предоставляет django rest framework:

```python
# backend/notes/urls.py
from rest_framework import routers

from .views import NoteViewSet

# Создаем router и регистрируем наш ViewSet
router = routers.DefaultRouter()
router.register(r'notes', NoteViewSet)

# URLs настраиваются автоматически роутером
urlpatterns = router.urls
```

Осталось только включить эти urls в backend/simplenote/urls.py:

```python
# backend/simplenote/urls.py
from django.conf.urls import include, url
from django.contrib import admin

urlpatterns = [
    url(r'^api/v1/', include('notes.urls')),
    url(r'^admin/', admin.site.urls),
]
```

## Запуск

Выполняем миграции и запускаем сервер:

```bash
$ python backend/manage.py makemigrations
$ python backend/manage.py migrate
$ python backend/manage.py runserver
```

Теперь можно протестировать работу нашего приложения. Для этого сделаем несколько HTTP запросов к нашему API с помощью библиотеки <a target="blank" href="https://httpie.org/">HTTPie</a>.

Сделаем тестовый GET запрос, чтобы проверить, работает ли наше приложение:

```bash
http GET "http://localhost:8000/api/v1/notes/"
```

![Empty list](https://raw.githubusercontent.com/apirobot/apirobot.github.io/master/uploads/django-vue-how-to-create-and-consume-api-part-1/list_empty.png)

Создадим новую заметку:

```bash
http POST "http://localhost:8000/api/v1/notes/" title="Note #1" body="Description #1"
```

![Create](https://raw.githubusercontent.com/apirobot/apirobot.github.io/master/uploads/django-vue-how-to-create-and-consume-api-part-1/create.png)

Проверяем список заметок:

```bash
http GET "http://localhost:8000/api/v1/notes/"
```

![List](https://raw.githubusercontent.com/apirobot/apirobot.github.io/master/uploads/django-vue-how-to-create-and-consume-api-part-1/list.png)

На этом все. Можете отдохнуть :)<br/>
<a target="blank" href="http://apirobot.me/posts/django-vue-how-to-create-and-consume-api-part-2">В следующей части</a> урока посмотрим, как можно обработать наш API и отобразить его пользователю с помощью Vue.js.

## Конец

Проблема? [Используй GitHub Issues](https://github.com/apirobot/django-vue-simplenote).

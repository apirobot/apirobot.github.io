---
title: Как реализовать систему лайков в Django
date: 2017-08-09 8:00:00
---

В статье мы реализуем функционал типичной кнопки “Мне нравится”. В этот функционал входит возможность:

1. Добавлять лайк;
2. Удалять свой лайк;
3. Посмотреть общее количество лайков у объекта;
4. Проверить, лайкнул ли пользователь объект или нет;
5. Показать пользователей, которые лайкнули объект.

Исходный код урока: <a target="blank_" href="https://github.com/apirobot/django-likes-app">https://github.com/apirobot/django-likes-app</a>

## Первоначальные настройки

Создаем и активируем виртуальное окружение:

```bash
$ virtualenv -p python3 venv
$ source venv/bin/activate
```

Устанавливаем django:

```bash
$ pip install django
```

Создаем проект:

```bash
$ django-admin startproject django_likes
$ cd django_likes
```

Объект, который мы будем лайкать в нашем тестовом проекте будет *Твит*. Этим объектом может быть все, что угодно: запись из блога, комментарий и т.д. Если вы уже работаете над каким-то своим проектом, то вы сможете легко адаптироваться.

Создаем приложения:

```bash
$ django-admin startapp likes
$ django-admin startapp tweets
```

Добавляем приложения в список установленных приложений (*INSTALLED_APPS*):

```python
# django_likes/settings.py

INSTALLED_APPS = [
    ...
    'likes.apps.LikesConfig',
    'tweets.apps.TweetsConfig',
]
```

## Модели (models.py)

Начнем с реализации модели Like:

```python
# likes/models.py

from django.conf import settings
from django.contrib.contenttypes.fields import GenericForeignKey
from django.contrib.contenttypes.models import ContentType
from django.db import models


class Like(models.Model):
    user = models.ForeignKey(settings.AUTH_USER_MODEL,
                             related_name='likes',
                             on_delete=models.CASCADE)

    content_type = models.ForeignKey(ContentType, on_delete=models.CASCADE)
    object_id = models.PositiveIntegerField()
    content_object = GenericForeignKey('content_type', 'object_id')
```

Модель Like основана на встроенном в Django фреймворке ContentType. Фреймворк ContentType предоставляет отношение GenericForeignKey, которое создает *обобщенные (generic) отношения* между моделями. Для сравнения, обычный ForeignKey создает отношение только с какой-то конкретной моделью.

Процесс создания GenericForeignKey:


1. Создаем поле с внешним ключом (ForeignKey) на модель ContentType.
2. Создаем поле для хранения первичного ключа (primary key) объекта, который вы хотите связать с моделью Like. В этом поле мы будем хранить ID экземпляра модели Tweet. Но хранить можно ID любой модели (моделей), поэтому отношение и называется обобщенным.
3. Создаем поле типа GenericForeignKey, передав в нее имена полей, которые мы создали в предыдущих двух пунктах.

Создаем модель Tweet и связываем ее с моделью Like через GenericRelation:

```python
# tweets/models.py

from django.contrib.contenttypes.fields import GenericRelation
from django.db import models

from likes.models import Like


class Tweet(models.Model):
    body = models.CharField(max_length=140)
    likes = GenericRelation(Like)

    def __str__(self):
        return self.body

    @property
    def total_likes(self):
        return self.likes.count()
```

После выполнения миграций проверяем работу моделей:

```python
>>> from django.contrib.contenttypes.models import ContentType

>>> from likes.models import Like
>>> from tweets.models import Tweet

>>> from django.contrib.auth import get_user_model
>>> User = get_user_model()

>>> user = User.objects.create_user(username='testuser', password='testuser')
>>> tweet = Tweet.objects.create(body='People are space puppets')

>>> tweet_model_type = ContentType.objects.get_for_model(tweet)
>>> Like.objects.create(content_type=tweet_model_type, object_id=tweet.id, user=user)
>>> Like.objects.count()
1
>>> Like.objects.first().content_object
<Tweet: People are space puppets>
>>> tweet.total_likes
1
```

## Функционал (services.py)

Реализуем функционал, о котором я говорил в начале статьи (добавление лайка, удаление лайка и т.д). Этот функционал будет представлен в виде обычных функций. Конечного пользователя этих функций (вас или другого программиста) не должно колыхать, как там эти лайки добавляются. Такой пользователь просто хочет вызвать функцию в контроллере (view) или в другом каком-то месте, и получить нужный ему результат.

После реализации (см. ниже) получим функционал:

```python
# `user` лайкает `tweet`
>>> add_like(tweet, user)
>>> tweet.total_likes
1

# `user` удаляет свой лайк
>>> remove_like(tweet, user)
>>> tweet.total_likes
0

# Проверяем, лайкнул ли `user` `tweet`
>>> is_fan(tweet, user)
False
>>> add_like(tweet, user)
>>> is_fan(tweet, user)
True

# Получаем всех пользователей, которые лайкнули `tweet`
>>> get_fans(tweet)
<QuerySet [<User: user>]>
```

Сама реализация:

```python
# likes/services.py

from django.contrib.auth import get_user_model
from django.contrib.contenttypes.models import ContentType

from .models import Like

User = get_user_model()


def add_like(obj, user):
    """Лайкает `obj`.
    """
    obj_type = ContentType.objects.get_for_model(obj)
    like, is_created = Like.objects.get_or_create(
        content_type=obj_type, object_id=obj.id, user=user)
    return like


def remove_like(obj, user):
    """Удаляет лайк с `obj`.
    """
    obj_type = ContentType.objects.get_for_model(obj)
    Like.objects.filter(
        content_type=obj_type, object_id=obj.id, user=user
    ).delete()


def is_fan(obj, user) -> bool:
    """Проверяет, лайкнул ли `user` `obj`.
    """
    if not user.is_authenticated:
        return False
    obj_type = ContentType.objects.get_for_model(obj)
    likes = Like.objects.filter(
        content_type=obj_type, object_id=obj.id, user=user)
    return likes.exists()


def get_fans(obj):
    """Получает всех пользователей, которые лайкнули `obj`.
    """
    obj_type = ContentType.objects.get_for_model(obj)
    return User.objects.filter(
        likes__content_type=obj_type, likes__object_id=obj.id)
```

## Что дальше?

Функционал,  о котором я говорил в начале урока - готов. Теперь вы можете пойти по одному из путей:

1. Реализовать приложение используя один Django.
2. Написать API (Django Rest Framework, Django Tastypie, …) и затем обработать его (React, Vue.js, Elm, …)

Сегодня модно использовать Javascript на фронте, поэтому я пойду по второму пути и напишу API с помощью Django Rest Framework (frontend за вами).

## Пишем API

Устанавливаем django rest framework:

```bash
$ pip install djangorestframework
```

Обновляем список установленных приложений:

```python
# django_likes/settings.py

INSTALLED_APPS = [
    ...
    'rest_framework',
]
```

Сериализуем Tweet:

```python
# tweets/api/serializers.py

from rest_framework import serializers

from ..models import Tweet


class TweetSerializer(serializers.ModelSerializer):

    class Meta:
        model = Tweet
        fields = (
            'id',
            'body',
            'total_likes'
        )
```          

Создаем ViewSet используя ModelViewSet, который снабжает нас create, update, list, retrieve, delete методами:

```python
# tweets/api/viewsets.py

from rest_framework import viewsets
from rest_framework.permissions import IsAuthenticatedOrReadOnly

from ..models import Tweet
from .serializers import TweetSerializer


class TweetViewSet(viewsets.ModelViewSet):
    queryset = Tweet.objects.all()
    serializer_class = TweetSerializer
    permission_classes = (IsAuthenticatedOrReadOnly, )
```

Так как мы используем ViewSet, то нам нет необходимости самим настраивать URLs. Можно использовать готовый класс Router, который предоставляет django rest framework:

```python
# tweets/api/urls.py

from rest_framework.routers import DefaultRouter

from .viewsets import TweetViewSet

# Создаем router и регистрируем ViewSet
router = DefaultRouter()
router.register(r'tweets', TweetViewSet)

# URLs настраиваются автоматически роутером
urlpatterns = router.urls
```

Теперь эти urls нужно включить в django_likes/urls.py:

```python
# django_likes/urls.py

from django.conf.urls import include, url
from django.contrib import admin

# Регистрируем API
apipatterns = [
    url(r'^', include('tweets.api.urls')),
]

urlpatterns = [
    url(r'^api/v1/', include(apipatterns, namespace='api')),
    url(r'^admin/', admin.site.urls),
]
```

На данный момет нам доступны только стандартные *CRUD* операции над моделью Tweet.

Когда текущий пользователь (*request.user*) получает информацию о Твите, мы должны знать, лайкнул он уже этот твит или нет. Таким образом мы будем знать, нужно ли подсвечивать кнопку “Мне нравится” на фронте или нет. Для этого добавляем в TweetSerializer поле is_fan:

```python
# tweets/api/serializers.py

from rest_framework import serializers

from likes import services as likes_services
from ..models import Tweet


class TweetSerializer(serializers.ModelSerializer):
    is_fan = serializers.SerializerMethodField()

    class Meta:
        model = Tweet
        fields = (
            'id',
            'body',
            'is_fan',
            'total_likes',
        )

    def get_is_fan(self, obj) -> bool:
        """Проверяет, лайкнул ли `request.user` твит (`obj`).
        """
        user = self.context.get('request').user
        return likes_services.is_fan(obj, user)
```

Для реализации оставшегося API создадим viewset mixin используя декоратор detail_route:

```python
# likes/api/mixins.py

from rest_framework.decorators import detail_route
from rest_framework.response import Response

from .. import services
from .serializers import FanSerializer


class LikedMixin:

    @detail_route(methods=['POST'])
    def like(self, request, pk=None):
        """Лайкает `obj`.
        """
        obj = self.get_object()
        services.add_like(obj, request.user)
        return Response()

    @detail_route(methods=['POST'])
    def unlike(self, request, pk=None):
        """Удаляет лайк с `obj`.
        """
        obj = self.get_object()
        services.remove_like(obj, request.user)
        return Response()

    @detail_route(methods=['GET'])
    def fans(self, request, pk=None):
        """Получает всех пользователей, которые лайкнули `obj`.
        """
        obj = self.get_object()
        fans = services.get_fans(obj)
        serializer = FanSerializer(fans, many=True)
        return Response(serializer.data)
```        

Сериализуем пользователя:

```python
# likes/api/serializers.py

from django.contrib.auth import get_user_model

from rest_framework import serializers

User = get_user_model()


class FanSerializer(serializers.ModelSerializer):
    full_name = serializers.SerializerMethodField()

    class Meta:
        model = User
        fields = (
            'username',
            'full_name',
        )

    def get_full_name(self, obj):
        return obj.get_full_name()
```

Последний штрих. Наследуемся от миксина:

```python
# tweets/api/viewsets.py

...
from likes.api.mixins import LikedMixin


class TweetViewSet(LikedMixin,
                   viewsets.ModelViewSet):
    ...
```

## Тестим API

Для тестирования API будем использовать библиотеку <a target="blank_" href="https://httpie.org/">HTTPie</a>.

Так как для большинства запросов необходимо быть авторизованным пользователем, то вам нужно создать пользователя (если вы этого еще не сделали):

```python
>>> from django.contrib.auth import get_user_model
>>> get_user_model().objects.create_user(username='testuser', password='testuser')
```

Создаем Твит:

```bash
$ http -a testuser:testuser POST "http://localhost:8000/api/v1/tweets/" body='People are space puppets'
```

![Create tweet](https://raw.githubusercontent.com/apirobot/apirobot.github.io/master/uploads/how-to-implement-liking-in-django/create-tweet.png)

Лайкаем этот твит:

```bash
$ http -a testuser:testuser POST "http://localhost:8000/api/v1/tweets/4/like/"
```

![Like tweet](https://raw.githubusercontent.com/apirobot/apirobot.github.io/master/uploads/how-to-implement-liking-in-django/like-tweet.png)

Проверяем информацию о твите (без авторизации):

```bash
$ http GET "http://localhost:8000/api/v1/tweets/4/"
```

![Get tweet](https://raw.githubusercontent.com/apirobot/apirobot.github.io/master/uploads/how-to-implement-liking-in-django/get-tweet.png)

Количество лайков (total_likes) увеличилось, а is_fan остался false, потому что запрос сделан без авторизации. Повторим запрос, только с авторизацией:

```bash
$ http -a testuser:testuser GET "http://localhost:8000/api/v1/tweets/4/"
```

![Get tweet with auth](https://raw.githubusercontent.com/apirobot/apirobot.github.io/master/uploads/how-to-implement-liking-in-django/get-tweet-with-auth.png)

Получаем пользователей, которые лайкнули твит:

```bash
$ http -a testuser:testuser GET "http://localhost:8000/api/v1/tweets/4/fans/"
```

![Get tweet fans](https://raw.githubusercontent.com/apirobot/apirobot.github.io/master/uploads/how-to-implement-liking-in-django/get-fans.png)

Удаляем лайк:

```bash
$ http -a testuser:testuser POST "http://localhost:8000/api/v1/tweets/4/unlike/"
```

![Unlike tweet](https://raw.githubusercontent.com/apirobot/apirobot.github.io/master/uploads/how-to-implement-liking-in-django/unlike-tweet.png)

## Конец

Проблема? [Используй GitHub Issues](https://github.com/apirobot/django-likes-app).

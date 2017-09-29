---
title: Расширяем модель User в Django
date: 2017-09-29 8:00:00
---

Для работы с пользователями, Django предоставляет готовую модель User. Часто, одной этой модели недостаточно. Приходится ее расширять, либо переписывать, если не устраивает стандартная реализация.

Несколько причин, почему может не устраивать стандартная реализация:

1. Вам нужно вместо двух полей *first_name* и *last_name*, одно поле *full_name*.
2. Не устраивает то, что поле *email* - необязательно *(blank=True)*.
3. Не устраивает то, что *USERNAME_FIELD = ‘username’*. *USERNAME_FIELD* указывает на поле, которое является уникальным идентификатором для пользователя. Кроме того, это поле используется на странице с авторизацией. Получается, что по умолчанию, пользователь вводит *username* и *password*. Если вам нужна авторизация через *email*, то *USERNAME_FIELD* нужно переопределить на *'email'*.

Рекомендую посмотреть исходный код модели User <a target="blank_" href="https://github.com/django/django">здесь</a>. Так будет проще понять, о чем я говорю.

В этом уроке поговорим о двух методах расширения модели User:

### (1) Создание модели User, которая наследуется либо от AbstractUser, либо от AbstractBaseUser.

Класс *AbstractUser*, по функционалу, является той самой моделью User, которую вы получаете от Django из коробки. Если вас устраивает стандартная реализация, но нужно ее расширить, то наследуйтесь от *AbstractUser*. Если нет, то наследуйтесь от *AbstractBaseUser* и переопределяйте поля сами.

### (2) Создание двух моделей: User и Profile, которые связаны друг с другом отношением один-к-одному (OneToOneField).

Зачем создавать две модели? Дело в том, что в программировании есть концепция *Single Responsibility Principle (Принцип единственной обязанности)*.

> «На каждый объект должна быть возложена одна единственная обязанность»
>
> – Single Responsibility Principle

Другими словами, класс и метод должны делать только одну вещь. Не нужно создавать так называемый *God Object*, который делает все, что только можно. Не нужно создавать метод, в котором реализована и валидация, и сохранение объекта в файл, и отправка сообщения пользователю… Для каждого метода должна существовать только одна причина его изменения. Если в методе вы хотите изменить как происходит валидация, то это одна причина изменения. Если хотите изменить реализацию отправки сообщения пользователю, то это другая причина. Когда причин больше чем одна, это значит, что метод делает слишком много.

Итак, следуя принципу единственной обязанности, мы создаем:


1. Модель User, которая отвечает только за аутентификацию и авторизацию пользователя в системе.
2. Модель Profile, которая хранит всевозможную информацию о пользователе для отображения на странице.

## User, наследуемый от AbstractUser

Это самый простой способ, т.к. класс *AbstractUser* уже предоставляет все, что нужно.

Cоздаем приложение:

```bash
$ django-admin startapp users
```

Создаем модель:

```python
# users/models.py

from django.contrib.auth.models import AbstractUser
from django.db import models


class User(AbstractUser):
    bio = models.CharField(max_length=160, null=True, blank=True)
    birthday = models.DateField(null=True, blank=True)

    def __str__(self):
        return self.username
```

Добавляем путь к модели User в настройках проекта, чтобы Django использовал нашу модель вместо стандартной:

```python
# settings.py

AUTH_USER_MODEL = 'users.User'
```

## User, наследуемый от AbstractBaseUser

Cоздаем приложение:

```bash
$ django-admin startapp users
```

Создаем модель:

```python
# users/models.py

from django.core.mail import send_mail
from django.contrib.auth.base_user import AbstractBaseUser
from django.contrib.auth.models import PermissionsMixin
from django.contrib.auth.validators import UnicodeUsernameValidator
from django.db import models
from django.utils.translation import ugettext_lazy as _


class User(AbstractBaseUser, PermissionsMixin):
    username_validator = UnicodeUsernameValidator()

    username = models.CharField(
        max_length=150,
        unique=True,
        validators=[username_validator],
    )
    email = models.EmailField(unique=True)
    full_name = models.CharField(max_length=255)
    bio = models.CharField(
        max_length=160,
        null=True,
        blank=True
    )
    birthday = models.DateField(
        null=True,
        blank=True
    )
    is_active = models.BooleanField(default=True)
    is_staff = models.BooleanField(default=False)

    USERNAME_FIELD = 'email'
    REQUIRED_FIELDS = ['username', 'full_name']

    objects = UserManager()

    def __str__(self):
        return self.email

    def get_short_name(self):
        return self.email

    def get_full_name(self):
        return self.full_name

    def email_user(self, subject, message, from_email=None, **kwargs):
        send_mail(subject, message, from_email, [self.email], **kwargs)
```


- USERNAME_FIELD - уникальный идентификатор пользователя. Это поле, вместе с паролем, используется при авторизации.
- REQUIRED_FIELDS - список полей, которые потребуется ввести при создании пользователя через команду *createsuperuser*. Также этот список нередко используется сторонними библиотеками, при создании формы с регистрацией. Нормальная форма с регистрацией включает в себя: *USERNAME_FIELD*, *REQUIRED_FIELDS* и *password*.
- is_active - активен ли пользователь или нет. Когда пользователь пытается удалить аккаунт, мы присваиваем полю *is_active=false*. Аккаунт из базы данных не удаляется. Это делается ради сохранения информации об активности пользователя.
- is_staff - имеет ли пользователь доступ к панели администратора или нет.

Создаем свой *UserManger* и переопределяем методы, которые отвечают за создание пользователя:

```python
# users/models.py

from django.contrib.auth.base_user import BaseUserManager


class UserManager(BaseUserManager):
    use_in_migrations = True

    def _create_user(self, email, username, full_name, password, **extra_fields):
        """
        Create and save a user with the given username, email,
        full_name, and password.
        """
        if not email:
            raise ValueError('The given email must be set')
        if not username:
            raise ValueError('The given username must be set')
        if not full_name:
            raise ValueError('The given full name must be set')
        email = self.normalize_email(email)
        username = self.model.normalize_username(username)
        user = self.model(
            email=email, username=username, full_name=full_name,
            **extra_fields
        )
        user.set_password(password)
        user.save(using=self._db)
        return user

    def create_user(self, email, username, full_name, password=None, **extra_fields):
        extra_fields.setdefault('is_staff', False)
        extra_fields.setdefault('is_superuser', False)
        return self._create_user(
            email, username, full_name, password, **extra_fields
        )

    def create_superuser(self, email, username, full_name, password, **extra_fields):
        extra_fields.setdefault('is_staff', True)
        extra_fields.setdefault('is_superuser', True)

        if extra_fields.get('is_staff') is not True:
            raise ValueError('Superuser must have is_staff=True.')
        if extra_fields.get('is_superuser') is not True:
            raise ValueError('Superuser must have is_superuser=True.')

        return self._create_user(
            email, username, full_name, password, **extra_fields
        )
```

Добавляем путь к модели User в настройках проекта, чтобы Django использовал нашу модель вместо стандартной:

```python
# settings.py

AUTH_USER_MODEL = 'users.User'
```

## User и Profile со связью один-к-одному

Cоздаем приложение:

```bash
$ django-admin startapp profiles
```

Модель User переопределять не будем. Возьмем ту, что предоставляет Django из коробки. Создадим модель Profile:

```python
# profiles/models.py

from django.conf import settings
from django.db import models


class Profile(models.Model):
    user = models.OneToOneField(
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE
    )
    bio = models.CharField(
        max_length=160,
        null=True,
        blank=True
    )
    birthday = models.DateField(
        null=True,
        blank=True
    )

    def __str__(self):
        return self.user.username
```

Осталось решить две проблемы:

1. После создания пользователя, не создается новый Profile, прикрепленный к этому пользователю.
2. После вызова метода save у пользователя, не вызывается метод save у прикрепленного к нему профиля. Из-за этого приходится каждый раз вызывать метод save после изменения профиля:

```python
>>> user.username = 'apirobot'
>>> user.profile.bio = 'Something about me'
>>> user.profile.save()
>>> user.save()
```

**Signals to the rescue!** Сигналы позволяют определённым *отправителям (senders)* уведомлять некоторый набор *получателей (receivers)* о совершении действий. Используем встроенный в Django сигнал *post_save*, который отсылается после завершения работы метода save:

```python
# profiles/signals.py

from django.contrib.auth import get_user_model
from django.db.models.signals import post_save
from django.dispatch import receiver
from .models import Profile

User = get_user_model()


@receiver(post_save, sender=User)
def create_or_update_user_profile(sender, instance, created, **kwargs):
    if created:
        instance.profile = Profile.objects.create(user=instance)
    instance.profile.save()
```

Не забудьте подключить сигналы в методе *ready* конфигурационного класса:

```python
# profiles/apps.py

from django.apps import AppConfig


class ProfilesConfig(AppConfig):
    name = 'profiles'

    def ready(self):
        import profiles.signals
```

Тестируем профиль через shell:

```python
>>> from django.contrib.auth import get_user_model
>>> User = get_user_model()

>>> User.objects.create_user(username='apirobot', password='password')
<User: apirobot>
>>> user = User.objects.first()
>>> user.profile
<Profile: apirobot>
>>> user.profile.bio = 'Something about me'
>>> user.save()
>>> User.objects.first().profile.bio
'Something about me'
```

Напоследок хотелось бы сказать, что не стоит забывать про оптимизацию запросов к базе данных. По умолчанию, Django ORM не включает в результат связанные объекты. Поэтому, при каждом обращении к пользователю через профиль, делается новый запрос к базе данных:

```python
# Запрос к базе данных
profile = Profile.objects.get(id=1)

# Снова запрос к базе данных для получения объекта User
user = profile.user
```

Это может стать проблемой, когда вы получаете список всех профилей и отображаете в html шаблоне данные профиля и связанного с ним пользователя:

```python
# profiles/views.py

from django.views import generic
from .models import Profile


class ProfileListView(generic.ListView):
    model = Profile
```

```html
<!-- profiles/templates/profiles/profile_list.html -->

{% raw %}
{% for profile in profile_list %}
  {{ profile.user.username }} {# Делает запрос к базе данных для каждого профиля #}
  {{ profile.bio }}
{% endfor %}
{% endraw %}
```

Такая проблема решается с помощью метода *select_related*, который добавляет к результату запроса связанный объект:

```python
# Запрос к базе данных
profile = Profile.objects.select_related('user').get(id=1)

# Не делает запрос к базе данных, т.к. `profile.user`
# был получен в предыдущем запросе
user = profile.user
```

Исправляем представление из предыдущего примера:

```python
# profiles/views.py

class ProfileListView(generic.ListView):
    queryset = Profile.objects.select_related('user')
```

## Заключение

В этом уроке мы поговорили о методах расширения модели User. Если вы не определились с выбором между User + Profile или просто User, то следуйте своему сердцу. После просмотра исходного кода проектов на GitHub, я понял, что каждый делает так, как хочет. В одних проектах всё пихают в модель User, в других используют User + Profile.

Имеет смысл создавать модель Profile, когда в моделе User много кода или когда вы хотите, чтобы модель User отвечала только за аутентификацию и авторизацию пользователя. Если же ваша модель User небольшая и вы не хотите париться по поводу создания сигналов и оптимизации запросов, то Profile вам не нужен.

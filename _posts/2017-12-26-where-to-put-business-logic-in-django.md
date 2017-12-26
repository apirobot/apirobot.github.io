---
title: Где хранить бизнес логику в Django
date: 2017-12-26 01:00:00
---

*Толстые модели (fat models)*, *тонкие представления (thin views)*, *тупые шаблоны (stupid templates)* - один из распространенных подходов к структурированию Django приложений. Цель подхода - вынести бизнес логику из представлений и шаблонов, и поместить ее в модели. Очевидно, что представления и шаблоны не должны содержать бизнес логику, так как они имеют совсем другие обязанности. Но выносить логику в модели не лучший вариант. Это приводит к тому, что модели становятся слишком большими и имеют слишком много обязанностей. Получаются так называемые *объекты боги (god objects)*. Из-за их сложности код сложно понять, тестировать и поддерживать.

# Сервисы вместо моделей

Альтернатива толстым моделям - изоляция бизнес логики в *сервисах (services)*. Сервисы - функции или классы, в которые чаще всего передаются объекты моделей (models), над которыми сервисы выполняют какие-то манипуляции в соответствии с бизнес требованиями приложения. Несколько примеров:

```python
# services.py

def send_confirmation_email(user):
    """Отправляет на почту `user` данные об активации аккаунта.

    :param user: объект модели User.
    """


def create_subscription(customer):
    """Подписывает `customer` на ежемесячную оплату.

    :param customer: объект модели Customer.
    """
```

В некоторых проектах вместо services используют слово utils, что сбивает с толку, потому что модуль с именем utils не совсем подходит для хранения бизнес логики. services.py или business.py подходит больше. Однако, модуль utils подходит для хранения функций, которые не относятся к какому-то конкретному django приложению (работа с временем и датами, перевод, кеширование и т.д.)

Ладно, хватит трепаться, перейдем к примерам. Представьте, что вы пытаетесь написать сайт, на котором публикуются обучающие курсы. У каждого курса есть участники: учителя и студенты. Определим модели для такого приложения:

```python
# models.py
from django.conf import settings
from django.db import models
from django.utils.translation import ugettext_lazy as _


class Course(models.Model):
    title = models.CharField(
        max_length=80,
        unique=True
    )
    # Отношение многие-ко-многим с пользователями
    # реализуется через модель Participation
    participants = models.ManyToManyField(
        settings.AUTH_USER_MODEL,
        through='Participation',
        related_name='courses',
    )


class Participation(models.Model):
    ROLE_STUDENT = 'ST'
    ROLE_TEACHER = 'TE'
    ROLE_CHOICES = (
        (ROLE_STUDENT, _('Student')),
        (ROLE_TEACHER, _('Teacher'))
    )

    user = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        related_name='participations',
        on_delete=models.CASCADE,
    )
    course = models.ForeignKey(
        'Course',
        related_name='participations',
        on_delete=models.CASCADE,
    )
    role = models.CharField(
        max_length=2,
        choices=ROLE_CHOICES,
        default=ROLE_STUDENT,
    )

    class Meta:
        # Комбинация значений полей user и course
        # должна быть уникальна.
        unique_together = ('user', 'course')
```

# Сервисы функции

Напишем несколько простых сервисов для моделей, определенных выше:

```python
# services.py
from .models import Course


def get_courses_in_which_user_has_been_enrolled_as_student(user):
    return Course.objects.filter(
        participations__user=user,
        participations__role=Participation.ROLE_STUDENT
    )


def get_courses_in_which_user_has_been_enrolled_as_teacher(user):
    return Course.objects.filter(
        participations__user=user,
        participations__role=Participation.ROLE_TEACHER
    )


def get_course_teachers(course):
    return course.participants.filter(
        participations__role=Participation.ROLE_TEACHER
    )
```

Пример использования сервиса в представлении (view):

```python
# views.py
from django.shortcuts import get_object_or_404, render

from .models import Course
from . import services


def course_teachers(request, course_pk):
    course = get_object_or_404(Course, pk=course_pk)
    teachers = services.get_course_teachers(course)
    return render(request, 'course_teachers.html', {
        'course': course,
        'teachers': teachers,
    })
```

Сервисы в виде функций это хорошо, но когда нужно реализовать сервис посложнее, я предпочитаю использовать классы.

# Сервисы классы

Для создания сервисов классов рекомендую использовать библиотеку <a target="blank_" href="https://github.com/mixxorz/django-service-objects">django-service-objects</a>. Библиотека предоставляет класс `Service`, который наследуется от `django.forms.Form`. Благодаря такому наследованию, валидность входных данных сервиса проверяется с помощью API django форм. Не нужно придумывать свой велосипед.

Пример простого сервиса класса:

```python
...
from django.forms.fields import ModelChoiceField
from service_objects.services import Service


class GetCourseTeachers(Service):
    course = ModelChoiceField(queryset=Course.objects.all())

    def process(self):
        course = self.cleaned_data['course']
        return course.participants.filter(
            participations__role=Participation.ROLE_TEACHER
        )


# вызов сервиса
teachers = GetCourseTeachers.execute({
    # Заметьте, что мы передаем `pk` объекта, а не сам объект,
    # иначе ModelChoiceField не будет работать
    'course': course.pk
})
```

Если вы использовали django формы, то вам все это уже знакомо. Единственное отличие - есть метод `process`, в котором определяется бизнес логика сервиса и который вызывается сразу после успешной проверки входных данных. Если входные данные не прошли проверку, то вместо этого выбрасывается исключение `service_objects.errors.InvalidInputsError`

Подробнее о библиотеке в <a target="blank_" href="http://django-service-objects.readthedocs.io/en/stable/">документации</a>.

Напишем сервис посложнее: добавление пользователя в участники курса в качестве студента:

```python
# services.py
from django.contrib.auth import get_user_model
from django.core.exceptions import PermissionDenied
from django.forms import ModelChoiceField
from service_objects.services import Service

from .models import Course, Participation

User = get_user_model()


class EnrollAsStudent(Service):
    course = ModelChoiceField(queryset=Course.objects.all())
    user = ModelChoiceField(queryset=User.objects.all())

    def process(self):
        course = self.cleaned_data['course']
        user = self.cleaned_data['user']

        participation, created = self._get_or_create_participation(course, user)
        self._validate_participation(participation, created)

        return participation

    def _get_or_create_participation(self, course, user):
        # Получаем объект модели Participation.
        # Если такого не существует, создаем новый,
        # причем у нового объекта role = ROLE_STUDENT
        return Participation.objects.get_or_create(
            course=course,
            user=user,
            defaults={'role': Participation.ROLE_STUDENT}
        )

    def _validate_participation(self, participation, created):
        if not created:
            if participation.role == Participation.ROLE_TEACHER:
                raise PermissionDenied('Already the teacher. Cannot enroll.')
            else:
                raise PermissionDenied('Already enrolled. Cannot re-enroll.')
```

Данный сервис легко расширяется. Добавим отправление уведомления на почту пользователя сразу после того, как пользователь стал участником курса:

```python
class EnrollAsStudent(Service):
    ...

    def __init__(self, *args, mailer=None):
        self._mailer = mailer

    def process(self):
        ...
        self._notify_user_about_enrollment(course, user)

        return participation

    def _notify_user_about_enrollment(self, course, user):
        if self._mailer is not None:
            self._mailer(
                subject='New enrollment',
                body='You have been enrolled to study the course: {}'.format(
                    course.title
                ),
                from_='Example <admin@example.com>',
                to=user.email
            )
```

В серсис `EnrollAsStudent` внедряется объект `mailer`, который отвечает за отправку сообщения на электронную почту. Подробнее об внедрениях зависимостей <a target="blank_" href="http://apirobot.me/posts/remove-dependencies-between-objects-in-python">здесь</a>. Если кратко, то внедрение зависимостей позволяет создавать объекты со слабой связью (low coupling), которые легко тестировать и повторно использовать.

# Сервисы функции & сервисы классы

Я люблю совмещать использование сервисов функций и сервисов классов. Для простых сервисов - функции, для сервисов посложнее - классы. Однако мне не нравится то, что интерфейсы вызовов у функций и у классов разные:

```python
# вызов функции
services.get_course_teachers(course)

# вызов класса
services.EnrollAsStudent.execute({
    'course': course.pk,
    'user': user.pk
})
```

Проблема в том, что при вызове сервиса в представлении, шаблоне или где-либо еще, мы должны знать, какой это сервис (функция или класс). Это неудобно, и такие лишние знания приводят к проблемам. Например, если функция расширяется, и мы хотим переписать эту функцию в класс, то нам придется поменять код везде, где эта функция вызывалась, потому что у класса и у функции разные интерфейсы вызовов.

Решение такой проблемы: обернуть сервис класс в функцию фабрику:

```python
def enroll_as_student(course, user):
    return EnrollAsStudent.execute({
        'course': course.pk,
        'user': user.pk
    })
```

Теперь интерфейсы вызовов одинаковые.

Еще один плюс использования функций фабрик - проще внедрять зависимости в сервис, не раскрывая все детали пользователю:

```python
def enroll_as_student(course, user, mailer=None):
    mailer = mailer if mailer is not None else SomeDefaultMailService
    return EnrollAsStudent.execute({
        'course': course.pk,
        'user': user.pk
    }, mailer=mailer)
```

# Заключение

Помещение всей логики приложения в модели приводит к тому, что модели становятся объектами богами (god objects), что нарушает <a target="blank_" href="http://apirobot.me/posts/solid-single-responsibility-principle">Принцип единственной обязанности</a>. Вынос логики из моделей в сервисы делает код более изолированным. Благодаря чему его проще тестировать и осмыслить.

*P.S.* Еще один способ вынести логику из моделей - использовать *Model Behaviors*. Но это уже тема отдельной статьи. Погуглите, если интересно.

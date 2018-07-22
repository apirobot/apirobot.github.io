---
title: Как начать Django проект, который можно масштабировать
date: 2018-07-21 3:00:00
---

<p align="center">
  <img src="https://raw.githubusercontent.com/apirobot/apirobot.github.io/master/uploads/how-to-start-django-project-that-scales/cover.png" alt="django project that scales">
</p>

В статье создадим проект используя шаблонизатор cookiecutter-django, настроим статическую типизацию, добавим автоматическое форматирование кода с помощью black, создадим скрипт, который запускает тесты, проверяет правильность типов через линтер mypy и стиль кода через black. Напоследок добавим пре-коммит хук, который автоматически запускает скрипт с проверками перед каждым коммитом.

# Содержание

- [Создаем проект с помощью cookiecutter-django](#создаем-проект-с-помощью-cookiecutter-django)
  - [Библиотеки](#библиотеки)
  - [Настройки](#настройки)
  - [Веб-сервер и Gunicorn](#веб-сервер-и-gunicorn)
- [Запускаем проект](#запускаем-проект)
- [Настраиваем статическую типизацию](#настраиваем-статическую-типизацию)
- [Форматируем код с помощью black](#форматируем-код-с-помощью-black)
- [Создаем скрипт для запуска тестов](#создаем-скрипт-для-запуска-тестов)
- [Добавляем пре-коммит хуки](#добавляем-пре-коммит-хуки)
- [Заключение](#заключение)

# Создаем проект с помощью cookiecutter-django

Шаблонизатор cookiecutter-django генерирует проект с кучей полезных настроек: отправка сообщений на почту через Mailgun, PostgreSQL из коробки, docker и docker-compose для разработки и продакшена, хранение медиа файлов на Amazon Storage и так далее.

Если вы создаете реальный проект, а не какой-нибудь hello world, то рекомендую использовать этот шаблонизатор.

Создаем проект:

<p align="center">
  <img src="https://raw.githubusercontent.com/apirobot/apirobot.github.io/master/uploads/how-to-start-django-project-that-scales/cookiecutter.png" alt="cookiecutter">
</p>

## Библиотеки

Помню как я впервые использовал cookiecutter-django. Я тогда подумал, что никогда не разберусь с ним, так как слишком много разных библиотек и настроек. Если вы впервые его используете, то рекомендую забить на все эти библиотеки. Я бы отметил только 2 из них:

1. Celery - инструмент для управления очередями задач. Задачей может быть запрос к API, парсинг веб-страницы, какие-то сложные долговременные вычисления и так далее. Полезно тем, что задачи выполняются в бэкграунде, а значит меньше нагрузка на Django приложение, и поэтому, когда пользователь делает запрос к серверу, он быстрее отвечает. Подробнее про Celery в другой статье: [https://apirobot.me/posts/celery-channels-create-real-time-app-with-background-tasks](https://apirobot.me/posts/celery-channels-create-real-time-app-with-background-tasks)
2. Sentry - репортит ошибки. Когда у вас в продакшене вылетела ошибка, то Sentry отправит сообщение на почту с подробной информацией об этой ошибке.

Я не включал эти библиотеки при создании проекта, так как они нам не понадобятся. Про остальные библиотеки подробнее в [документации](https://cookiecutter-django.readthedocs.io/en/latest/project-generation-options.html).

## Настройки

Другая важная часть проекта - настройки (папка config/settings). В папке 4 файла. Для каждого окружения свои настройки:

1. local.py используется при разработке;
2. production.py используется в продакшене;
3. test.py используется при запуске тестов;
4. base.py используется во всех окружениях.

Самые непонятный настройки в файле production.py. Разберем их по порядку.

SSL (Secure Sockets Layer) - протокол, который позволяет устанавливать безопасное соединение между сервером (сайтом) и клиентом (браузером). На хорошем сайте должен использоваться HTTPS, иначе мамкины хацкеры смогут при желании легко перехватить пароли ваших пользователей. В шаблонизаторе настроено использование SSL и HTTPS:

```python
    # SECURITY
    # ------------------------------------------------------------------------
    # https://docs.djangoproject.com/en/dev/ref/settings/#secure-proxy-ssl-header
    SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')
    # https://docs.djangoproject.com/en/dev/ref/settings/#secure-ssl-redirect
    SECURE_SSL_REDIRECT = env.bool('DJANGO_SECURE_SSL_REDIRECT', default=True)
    ...
```

Медиа файлы - файлы, которые загружает пользователь (фотографии, видео, документы и прочее). В шаблонизаторе настроено хранение медиа файлов на Amazon Storage (Amazon S3):

```python
    # STORAGES
    # ------------------------------------------------------------------------
    # https://django-storages.readthedocs.io/en/latest/#installation
    INSTALLED_APPS += ['storages']  # noqa F405
    # https://django-storages.readthedocs.io/en/latest/backends/amazon-S3.html#settings
    AWS_ACCESS_KEY_ID = env('DJANGO_AWS_ACCESS_KEY_ID')
    ...
```

Существуют разные сервисы, которые отправляют сообщения на почту: Mailgun, SendGrid, Amazon SES... В шаблонизаторе настроено отправление сообщений на почту через Mailgun с помощью библиотеки [Anymail](https://github.com/anymail/django-anymail), которая поддерживает все эти сервисы:

```python
    # Anymail (Mailgun)
    # ------------------------------------------------------------------------
    # https://anymail.readthedocs.io/en/stable/installation/#installing-anymail
    INSTALLED_APPS += ['anymail']  # noqa F405
    EMAIL_BACKEND = 'anymail.backends.mailgun.EmailBackend'
    ...
```

## Веб-сервер и Gunicorn

Что происходит когда клиент (браузер) делает запрос к сайту? Каким образом Django приложение получает этот запрос, и передает обратно ответ?

Существуют веб-сервера, которые перехватывают запросы клиентов: Nginx, Caddy, Apache. Но эти веб-сервера не могут общаться напрямую с Django приложением. Для этого существует Gunicorn, который запускает Django приложение и при получении запроса от веб-сервера, он передает запрос приложению и ожидает ответ. Полученный ответ передается обратно веб-серверу, а веб-сервер возвращает его клиенту:

Клиент (внешний мир) ←→ Веб-сервер (Nginx, Caddy) ←→ Gunicorn ←→ Django приложение

В шаблонизаторе используется Caddy. Фишка Caddy в том, что им легко пользоваться и он автоматически настраивает HTTPS на сайте.

# Запускаем проект

В продакшене и во время разработки я всегда использую докер. Докер упрощает запуск приложения и деплой его в продакшн с помощью контейнеров. Неважно где был создан этот контейнер, докер гарантирует идентичность контейнеров на любой машине. Благодаря этому нет такой головной боли, что локально все работает, а во время деплоя все ломается из-за каких-то конфликтов в зависимостях.

Перед тем как создавать докер изображения, добавим еще зависимости в проект. Эти зависимости понадобятся нам позже:

```bash
    # requirements/local.txt
    ...

    mypy==0.620  # https://github.com/python/mypy
    black==18.6b4  # https://github.com/ambv/black
    pre-commit==1.10.3  # https://github.com/pre-commit/pre-commit
```

Теперь запускаем проект используя докер:

```bash
    $ docker-compose -f local.yml up
```

Подробнее про докер [здесь](https://docs.docker.com/get-started/).

# Настраиваем статическую типизацию

Зачем нам типизировать код, если фишка питона в том, что он является динамически типизированным языком? Динамически типизированные языки не лучше и не хуже статически типизированных. Все зависит от предпочтений и проекта. Вы будете писать код быстрее, используя динамически типизированные языки. Но типизация сделает код читабельнее (правда не всегда) и снизит вероятность появления багов, связанных с типами.

Для типизации кода в питоне используются тайп хинты (type hints). Если у функции или переменной нет тайп хинтов, то это значит, что они имеют тип Any.

Пример кода с использованием тайп хинтов:

```python
    from typing import Union

    def download_image(url: str,
                       save_path: str = '') -> Union[None, str]:
        ...
```

Функция download_image возвращает либо None, либо строку (Union[None, str]). Если функция при запуске вернет не None и не строку, а что-нибудь другое, то ошибки не будет, так как тайп хинты переводятся в комментарии и игнорируются при запуске.

Для проверки кода на соблюдение типов используется линтер mypy.

Добавляем mypy в список зависимостей (если еще этого не сделали):

```bash
    # requirements/local.txt
    ...

    mypy==0.620  # https://github.com/python/mypy
```

Добавляем настройки:

```bash
    # setup.cfg

    [mypy]
    python_version = 3.6
    check_untyped_defs = True
    ignore_errors = False
    ignore_missing_imports = True
    warn_unused_ignores = True
    warn_redundant_casts = True
    warn_unused_configs = True

    [mypy-*.migrations.*]
    # Django миграции не должны выдавать ошибки:
    ignore_errors = True
```

Документация по настройкам [здесь](http://mypy.readthedocs.io/en/latest/config_file.html).

Запускайм линтер:

```bash
    $ docker-compose -f local.yml run --rm django mypy example
```

Ошибок с типами не должно быть после запуска, но вы можете попробовать добавить функцию в проект с неправильными типами и запустить линтер еще раз.

# Форматируем код с помощью black

> Any color you like. As long as it is black.

У каждого может быть свой стиль написания кода. Когда несколько человек работают над одним проектом, нередко возникает такая проблема, что часть кода написана в одном стиле, а другая часть в другом. Это плохо.

Если вы новенький в проекте, то вы должны подстраиваться под стиль других разработчиков, чтобы код везде выглядел одинаково. Но не всегда это получается.

Black устраняет эту проблему. Он форматирует код автоматически. Благодаря чему, вы фокусируетесь на самом коде, а не на том, как он выглядит.

Добавляем black в список зависимостей (если еще этого не сделали):

```bash
    # requirements/local.txt
    ...

    black==18.6b4  # https://github.com/ambv/black
```

Проверяем, какие файлы необходимо отформатировать, чтобы они соответствовали формату black:

```bash
    $ docker-compose -f local.yml run --rm django black --check .

    would reformat /app/config/settings/test.py
    would reformat /app/config/settings/local.py
    would reformat /app/config/wsgi.py
    would reformat /app/docs/conf.py
    would reformat /app/example/users/adapters.py
    would reformat /app/example/users/forms.py
    would reformat /app/config/urls.py
    would reformat /app/example/users/tests/factories.py
    would reformat /app/example/users/tests/test_forms.py
    would reformat /app/config/settings/base.py
    would reformat /app/config/settings/production.py
    All done! 💥 💔 💥
    11 files would be reformatted, 25 files would be left unchanged
```

Теперь форматируем все файлы:

```bash
    $ docker-compose -f local.yml run --rm django black .

    reformatted /app/config/wsgi.py
    reformatted /app/config/settings/local.py
    reformatted /app/config/settings/test.py
    reformatted /app/example/users/adapters.py
    reformatted /app/docs/conf.py
    reformatted /app/config/urls.py
    reformatted /app/example/users/forms.py
    reformatted /app/example/users/tests/factories.py
    reformatted /app/example/users/tests/test_forms.py
    reformatted /app/config/settings/base.py
    reformatted /app/config/settings/production.py
    All done! ✨ 🍰 ✨
    11 files reformatted, 25 files left unchanged.
```

Запускать эти команды после каждого сохранения файла неудобно. Существуют интеграции с black для всех популярных редакторов: Visual Studio Code, Atom, Sublime Text, PyCharm, ... Благодаря им, ваш .py файл автоматически форматируется после каждого сохранения файла.

# Создаем скрипт для запуска тестов

Теперь создадим bash скрипт в корневой директории с именем `ci`, который запускает все вышеперечисленные проверки. Этот скрипт легко масштабировать, добавляя в него дополнительные проверки:

```bash
    #!/bin/sh

    set -o errexit
    set -o nounset

    pyclean () {
      # Clean cache:
      find . | grep -E '(__pycache__|\.py[cod]$)' | xargs rm -rf
    }

    run_ci () {
      # Run tests:
      mypy example
      py.test

      # Check style:
      black --check .

      # Check that all migrations worked fine:
      python /app/manage.py makemigrations --dry-run --check
    }

    # Remove any cache before the script:
    pyclean

    # Clean everything up:
    trap pyclean EXIT INT TERM

    # Run the CI process:
    run_ci
```

Запускаем скрипт:

```bash
    $ docker-compose -f local.yml run --rm django sh ./ci
```

# Добавляем пре-коммит хуки

Я уверен, что с вами случалась или случится такая ситуация, что вы закоммитили код, забыв запустить тесты. Такая ситуация никогда не произойдет если использовать пре-коммит хуки, так как они запускаются перед каждым коммитом. Если во время запуска пре-коммит хуков возникает ошибка, то коммит откатывается.

Добавим пре-коммит хук, который автоматически запускает скрипт с тестами:

```bash
    # .pre-commit-config.yaml
    - repo: local
      hooks:
      - id: ci
        name: ci
        entry: docker-compose -f local.yml run --rm django sh ./ci
        pass_filenames: false
        language: system
```

Чтобы пре-коммит хуки работали, установите библиотеку `pre-commit` (pip install pre-commit) и затем запустите комманду:

```bash
    $ pre-commit install
```

# Заключение

Вы готовы начать любой проект. Теперь идите и создайте что-нибудь прекрасное.

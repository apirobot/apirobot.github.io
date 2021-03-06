---
title: Итерируемые объекты, итераторы и генераторы в Python
date: 2018-01-17 05:00:00
---

В статье разберемся, что такое итерируемые объекты, итераторы и генераторы. Узнаем тайну работы цикла for. Реализуем шаблон проектирования “Итератор”. А затем удалим все и сделаем “по-нормальному”, используя генераторы.

# Что такое итерируемый объект и итератор

**Итератор** - любой объект, реализующий метод `__next__`, который возвращает следующий элемент в очереди или выбрасывает исключение `StopIteration`, если не осталось элементов.

**Итерируемый объект** - любой объект, реализующий метод `__iter__` или `__getitem__`. Итерируемым объектом является любая коллекция: список, кортеж, словарь, и т.д.

**Цель итерируемого объекта** - создать итератор. Для этого у него есть метод `__iter__`, при каждом обращении к которому создается новый итератор.

**Цель итератора** - пройтись по элементам. Для этого у него есть метод `__next__`, который возвращает элементы один за другим.

Если итератор реализует метод `__iter__` или `__getitem__`, дополнительно к методу `__next__`, то он также является и итерируемым объектом. Это позволяет использовать итератор там, где требуется итерируемый объект.

# Как работает цикл for

В большинстве случаев, при обработке итерируемого объекта используется цикл for:


```python
>>> items = [1, 2, 3]
>>> for item in items:
...     print(item)
1
2
3
```

Но если бы не было цикла for, то для эмуляции его работы, пришлось бы написать такой код:

```python
>>> items = [1, 2, 3]
>>> it = iter(items)
>>> while True:
...     try:
...         print(next(it))
...     except StopIteration:
...         break
1
2
3
```

Список `items` является итерируемым объектом, поэтому мы можем получить от него итератор. Встроенная функция `iter` именно это и делает: получает итератор от объекта `items` :

```python
>>> items = [1, 2, 3]
>>> # Получаем итератор
>>> it = iter(items)
```

После получения итератора, мы начинаем его использовать через встроенную функцию `next`, которая вызывает метод `__next__` у итератора. Метод `__next__` возвращает следующий элемент в очереди, либо выбрасывает исключение `StopIteration`:

```python
>>> # Используем итератор
>>> next(it)  # Вызывает метод __next__
1
>>> next(it)
2
>>> next(it)
3
>>> next(it)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
StopIteration
```

Заметьте, что по итератору можно пройтись только один раз. Нет способа вернуться к какому-то конкретному элементы, либо “сбросить” итератор. Чтобы пройтись по элементам снова, нужно создать новый итератор, вызвав функцию `iter`.

# Шаблон проектирования “Итератор”

Как использовать итерируемые объекты и итераторы разобрались. Теперь напишем свой итерируемый объект, который создает и возвращает итератор при обращении к методу `__iter__`:

```python
import re


class Finder:

    def __init__(self, pattern, text):
        self.text = text
        self.matches = re.findall(pattern, text)

    def __iter__(self):
        return FinderIterator(self.matches)


class FinderIterator:

    def __init__(self, matches):
        self.matches = matches
        self.index = 0

    def __next__(self):
        try:
            match = self.matches[self.index]
        except IndexError:
            raise StopIteration()
        self.index = self.index + 1
        return match
```

Используем `Finder` через цикл for:

```python
>>> finder = Finder(r'\w+', 'extract; the! words, from @me')
... for match in finder:
...     print(match)
extract
the
words
from
me
```

Вручную:

```python
>>> finder = Finder(r'\w+', 'extract; the! words, from @me')
>>> it = iter(finder)
>>> next(it)
'extract'
>>> next(it)
'the'
>>> next(it)
'words'
>>> next(it)
'from'
>>> next(it)
'me'
>>> next(it)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "<stdin>", line 24, in __next__
StopIteration
```

Код выше - пример реализации шаблона проектирования “Итератор”. Однако, реализовывать этот шаблон в Python не стоит никогда. Много кода. Мы, питонисты, слишком ленивые для такого. А привел я этот пример, так как он наглядно показывает различие между итерируемым объектом и итератором. Итерируемый объект - создает итератор. Итератор - обрабатывает последовательность. Мы сократим этот код позже.

# Генераторы

Генератор - функция, которая генерирует значения. Она отличается от обычной функции тем, что может приостанавливать свое выполнение, возвращать промежуточный результат, а затем возобновлять выполнение в любой момент времени. Пример простой генераторной функции:

```python
>>> def generator():
...     for i in range(3):
...         yield i

>>> # Через цикл for
>>> for i in generator():
...     print(i)
0
1
2

>>> # Вручную
>>> gen = generator()
<generator object generator at 0x7f38572b96d0>
>>> next(gen)
0
>>> next(gen)
1
>>> next(gen)
2
>>> next(gen)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
StopIteration
```

Давайте разберемся, чем отличается работа с обычной функцией и работа с генераторной функцией.

Процесс работы с обычной функцией:

1. Вызываем функцию
2. Ждем, когда функция завершит выполнение и вернет результат
3. После получения результата, обрабатываем его

Процесс работы с генераторной функцией:

1. Вызываем генераторную функцию, взамен получаем объект генератора
2. Передаем объект генератора в функцию `next`. После этого, функция генератор выполняется до оператора `yield` . После передачи нам значения (`yield`), функция останавливается и ждет следующего вызова
3. После получения значения от генератора, мы обрабатываем его. Затем повторяем второй шаг, либо прекращаем работу с генератором

Как вы поняли, отличие обычной функции от генераторной функции в том, что обычная функция выполняется один раз и возвращает результат целиком. Даже если нужно получить только первый элемент последовательности от функции, то все равно придется ждать пока функция не вернет последовательность целиком. С генераторной функцией иначе. От нее мы получаем элементы по одному. Благодаря этому, после получения первого элемента, можно сразу приступить к его обработке.

Перепишем пример с классом `Finder`. Только в этот раз используем генераторную функцию:

```python
import re

class Finder:

    def __init__(self, pattern, text):
        self.text = text
        self.matches = re.findall(pattern, text)

    def __iter__(self):
        for match in self.matches:
            yield match
```

Код стал короче. Однако и в нем есть проблема. Об этом ниже.

# Отличие итератора от генератора

Генератор является итератором. Отличие итератора от генератора в том, что итератор извлекает элементы из коллекции (список, кортеж, …), а генератор может порождать элементы из воздуха. Типичный пример - генерация чисел Фибоначчи:

```python
>>> def fibonacci():
...     a, b = 0, 1
...     while True:
...         yield a
...         a, b = b, a + b

>>> f = fibonacci()
<generator object fibonacci at 0x7f1d96f56990>
>>> next(f)
0
>>> next(f)
1
>>> ...
```

Так как последовательность чисел Фибоначчи бесконечна, то ее невозможно поместить в список, а затем извлекать от туда. Единственное решение - использовать генераторную функцию, которая будет возвращать числа Фибоначчи по одному, а затем удалять их из памяти.

Теперь вернемся к примеру с классом `Finder`. В нем генератор ведет себя как итератор. Он извлекает элементы из списка, так как функция `findall` библиотеки `re` возвращает список. Из-за этого, смысл в создании генератора теряется, так как в любом случае придется ждать, пока функция `findall`  не отработает целиком и не вернет список со всеми найденными элементами.

Однако, есть функция `finditer`, которая выполняет ту же задачу, что и функция `findall`, только возвращает объект генератора, вместо списка:

```python
import re

class Finder:

    def __init__(self, pattern, text):
        self.pattern = pattern
        self.text = text

    def __iter__(self):
        for match in re.finditer(self.pattern, self.text):
            yield match.group()
```

# Генераторные выражения

Для получения объекта генератора не обязательно создавать генераторную функцию и использовать оператор `yield`. Объект генератора можно получить с помощью генераторного выражения. Генераторные выражения - просто синтаксический сахар. Более простой способ создания объектов генераторов.

> Генераторные выражения очень похожи на списковые включения, о которых можно почитать <a target="blank_" href="http://apirobot.me/posts/listcomps-dictcomps-setcomps-in-python">здесь</a>.

Перепишем класс `Finder` с использованием генераторного выражения:

```python
import re

class Finder:

    def __init__(self, pattern, text):
        self.pattern = pattern
        self.text = text

    def __iter__(self):
        return (match.group()
                for match in re.finditer(self.pattern, self.text))
```

При создании генераторного выражения стоит помнить об одном: слишком длинные и сложные генераторные выражения - плохо. Лучше использовать обычную генераторную функцию с оператором `yield` и циклом `for`, чем создавать трехэтажные генераторные выражения.

Напоследок, добавлю очевидное. Если ваш класс делает только одно: реализует метод `__iter__` и создает объект генератора, то смысла в создании класса нет никакого. Благо, Python не навязывает использование объекто-ориентированного программирования везде, где только можно. Заменим класс `Finder` на функцию:

```python
import re

def find(pattern, text):
    return (match.group() for match in re.finditer(pattern, text))
```

# Конец

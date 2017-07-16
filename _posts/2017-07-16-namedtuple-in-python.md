---
title: Namedtuple в python
date: 2017-07-16 7:00:00
---

Функция *collections.namedtuple* позволяет построить класс, который содержит только поля и никаких методов. Экземпляр класса будет работать так же, как и обычный кортеж (*tuple*), только к элементам экземпляра класса можно будет обратиться через соответсвутющие имена, в отличие от обычного кортежа, где к элементам можно обратиться только через их индексы.

В примере ниже определим кортеж для хранения информации о городе с помощью *collections.namedtuple*:

```bash
>>> from collections import namedtuple
>>> City = namedtuple('City', ['name', 'country', 'population'])

>>> minsk = City('Minsk', 'Belarus', 2648500)
>>> minsk
City(name='Minsk', country='Belarus', population=2648500)
>>> minsk.name
'Minsk'
>>> minsk.country
'Belarus'
>>> minsk[2]
2648500
```

Для создания именованного кортежа, нужно задать имя класса и список имен полей. Мы задали имена полей с помощью списка (*['name', 'country', 'population']*), однако их можно задать и с помощью строки, разделяя поля пробелом или запятой:

```bash
>>> from collections import namedtuple
>>> City = namedtuple('City', 'name country population')
>>> City = namedtuple('City', 'name, country, population')

>>> minsk = City(name='Minsk', country='Belarus', population=2648500)
>>> minsk
City(name='Minsk', country='Belarus', population=2648500)
```

Для сравнения, в аналогичной ситуации можно было бы использовать словарь (*dict*), либо обычный кортеж (*tuple*):

```bash
>>> minsk = {'name': 'Minsk', 'country': 'Belarus', 'population': 2648500}
>>> minsk
{'name': 'Minsk', 'country': 'Belarus', 'population': 2648500}

>>> minsk = ('Minsk', 'Belarus', 2648500)
>>> minsk
('Minsk', 'Belarus', 2648500)
```

Но проблема словаря в том, что это изменяемый объект и он медленнее, чем кортеж. А проблема кортежа в том, что к его элементам нужно обращаться по индексам, и из-за этого код становится не читабельным.

**Namedtuple to the rescue!** Именованный кортеж решает эти проблемы. Он неизменяем и работает быстрее, чем словарь, а также, благодаря ему, ваш код становиться читабельным и всем сразу понятно, что в этом кортеже храниться.

Вам решать, когда что использовать, но если вы уверены, что именованный кортеж сделает ваш код читабельнее и проще к пониманию, то используйте его. Всегда помните:

> Always code as if the guy who ends up maintaining your code will be a violent psychopath who knows where you live.
>
> – Jason Statham

## Конец

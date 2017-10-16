---
title: Namedtuple в python
date: 2017-10-16 7:00:00
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

**Namedtuple to the rescue!** Именованный кортеж решает эти проблемы. Он неизменяем и работает быстрее, чем словарь, а также, благодаря ему, код становиться читабельным и всем сразу понятно, что в этом кортеже храниться.

## Скрытие структуры данных

Вот еще пример, где можно применить именованный кортеж. Представьте, что у вас есть класс *Inventory*, в который передается список с ценой и количеством каждого продукта. Нужно узнать итоговую стоимость всех продуктов:

```python
class Inventory:

    def __init__(self, data):
        self.data = data

    def total_price(self):
        # x[0] - цена; x[1] - количество
        return sum(x[0] * x[1] for x in self.data)
```

Предпологается, что в класс передается такой список кортежей:

```python
data = [(1.45, 2), (1.25, 5), (0.55, 3)]
```

Работа класса зависит от структуры данных. Если класс обращается в нескольких местах к элементам кортежа по индексу, то возникнут проблемы, когда структура данных изменится. Придется в каждом месте делать изменение, а это не соответсвует принципу *DRY (Don't Repeat Yourself)*.

Отрефакторим класс выше. Заменим кортеж на именованный кортеж, и сделаем так, чтобы при изменении структуры данных, код класса изменялся только в одном месте:

```python
from collections import namedtuple


Product = namedtuple('Product', 'price quantity')


class Inventory:

    def __init__(self, data):
        self.products = self.to_products(data)

    def to_products(self, data):
        return [Product(x[0], x[1]) for x in data]

    def total_price(self):
        return sum(product.price * product.quantity
                   for product in self.products)
```

Теперь, при изменении структуры данных, код в классе изменяется только в методе *to_product*. Кроме того, метод *to_product* изолирует сложную структуру списка с помощью namedtuple.

## Заключение

Вам решать, когда использовать именованный кортеж, а когда словарь или обычный кортеж. Но если вы уверены, что именованный кортеж сделает ваш код читабельнее и проще к пониманию, то используйте его. Всегда помните:

> Always code as if the guy who ends up maintaining your code will be a violent psychopath who knows where you live.
>
> – Jason Statham

## Конец

---
title: "SOLID: Принцип единственной обязанности"
date: 2017-11-30 11:00:00
---

Хорошо спроектированная программа - программа, в которой:

- не страшно делать изменения;
- затраты на эти изменения - минимальны;
- легко тестировать

Для того, чтобы писать такие программы, нужно практиковаться и знать о принципах проектирования хороших программ. Некоторые из таких принципов:

- DRY (Don’t repeat yourself);
- KISS (Keep it simple, stupid);
- YAGNI (You ain’t gonna need it);
- DDD (Domain Driven Design);
- Шаблоны проектирования;
- SOLID принципы;
- …

Каждому, из этих принципов, можно посветить отдельную статью, либо даже целую книгу. Например, по шаблонам проектирования есть две отличные книги: «Design Patterns» (авторов этой книги еще прозвали “бандой четырех”) и «Head First Design Patterns».

Эта статья посвящена первому SOLID принципу - Принципу единственной обязанности (Single Responsibility Principle). Об остальных SOLID принципах возможно напишу в другой раз.

# Принцип единственной обязанности

Принцип единственной обязанности (Single Responsibility Principle) - один из самых простых принципов проектирования в программировании, но в то же время, один из самых нарушаемых.

> «На каждый объект должна быть возложена одна единственная обязанность»
>
> Принцип единственной обязанности

Другими словами, классы и методы должны делать только одну вещь, и должна существовать только одна причина изменения класса и метода.

## Как проверить класс/метод на соблюдение SRP (P.S. примеры ниже)

### Представьте обязанности класса/метода в виде предложения
Если при составлении предложения появляются союзы “и” или “или”, то скорее всего, класс/метод обладает больше, чем одной обязанностью.

### Замените метод класса вопросом
Каждый вопрос должен иметь смысл для текущего класса.

### Найдите возможные причины изменения класса/метода
Если в методе вы хотите изменить, как происходит валидация, то это одна причина изменения. Если хотите изменить реализацию отправки сообщения пользователю, то это другая причина. Когда причин больше чем одна, это значит, что метод делает слишком много.

## Пример

Напишем класс Order, который обрабатывает заказ клиента в каком-нибудь интернет-магазине:

```python
class Order:

    def __init__(self, customer, cart):
        self.customer = customer
        self.cart = list(cart)

    def charge(self):
        print('Charging...')
        # получаем деньги от клиента

    def send_mail(self):
        print('Emailing...')
        # отправляем сообщение на электронную почту клиента

    def generate_coupon(self):
        print('Generating coupon...')
        # генерируем скидочный купон
```

Попытаемся описать класс одним предложением:

> Класс Order обрабатывает заказ клиента снимая с него деньги **И**
> генерирует скидочный купон для клиента **И**
> отправляет сообщение на электронную почту клиента

Как я написал выше, в предложении не должно быть союзов “и” или “или”. Класс делает слишком много.

Попытаемся придумать вопрос для метода класса:

> Пожалуйста, мистер Order, можете ли вы отправить сообщение на электронную почту клиента?

Вопрос не имеет смысла в контексте класса Order. Он имел бы смысл для какого-нибудь класса Mailer.

К тому же, что если нужно отправить сообщение на электронную почту клиента в другом месте? Например после регистрации. В такой ситуации, чтобы не нарушать принцип DRY (не повторяйся), пришлось бы использовать метод send_mail класса Order. Очевидно, что это плохой вариант. Класс Order имеет слишком много обязанностей. Для отправления сообщения на почту следует создать отдельный класс, либо обычную функцию.

Переделываем.

```python
class Order:

    def __init__(self, customer, cart):
        self.customer = customer
        self.cart = list(cart)

    def charge(self):
        print('Charging...')
        # получаем деньги от клиента


class Mailer:

    def send_mail(self, content, customer):
        print('Emailing...')
        # отправляем `content` на почту `customer`


class Coupon:

    def __init__(self, code=None):
        code = code if code is not None else Coupon.generate_code()

    @staticmethod
    def generate_code():
        print('Generating code...')
        # генерируем код купона
```

Теперь можно создать купон и отправить сообщение на почту не используя класс Order. Классы Mailer и Coupon независимы и могут быть использованы в любой части программы отдельно друг от друга.

# Заключение

Соблюдение принципа единственной обязанности упрощает создания кода, который легко поддается изменениям и повторному использованию. Если класс не соблюдает принцип единственной обязанности и делает слишком много (обрабатывает заказ клиента, отправляет сообщение на почту, генерирует скидочный купон), то повторное использование только части поведения класса (например, отправление сообщения на почту) либо усложняется, либо вообще невозможно.
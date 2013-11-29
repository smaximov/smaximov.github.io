---
layout: post
title: "Automatic Resource Management in Scala"
date: 2013-11-29 14:09
comments: true
categories:
- Scala
---

* TOC
{:toc}

Введение
===========

Решал вчера задание NodeScala для курса реактивного программирования
на Scala[^rx]. Часть задания состояла в реализации _trait_
`Exchange`, содержащего методы `write` и `close`. Метод `write`
выполнял запись строки в ответ (_response_) сервера, а метод
`close` закрывал объект после использования. В инструкции к заданию
было обращено особое внимание на необходимость закрытия объекта:

> The trait `Exchange` is used to write your response back to the user using the `write` method. Whenever you use it, don’t forget to close it by calling the `close` method.

Как можно догадаться, в конце концов именно это я и забыл сделать. В
результате около часа было потрачено на поиск ошибки. Это навело меня
на мысль, что было бы неплохо воспользоваться богатыми возможностями
Scala для реализации автоматического управления ресурсами (готовые
решения меня по понятной причине не интересовали).

<!-- more -->

Решение
==========

Наиболее логичное решение, которое пришло мне в голову: написать
метод, который будет принимать блок кода, выполнять его, а затем
закрывать объект.

Первое приближение
----------------------

Сначала я реализовал этот метод внутри `Exchange`:

``` scala
  def closeAfter(block: => Unit) {
    try block
    finally close()
  }
```

Теперь `Exchange` можно использовать так:

``` scala
  val x: Exchange = Exchange()

  x closeAfter {
    x write "foo"
  }
```

Недостаток такого решения очевиден: оно не универсально. Для другого
класса (трейта) нужно будет заново писать такой метод.

Обобщённый статический метод
-----------------------------------

Создадим объект `Managed`, который определяет метод `using`,
который принимает объект с методом `close` и закрывает его после того,
как выполнит блок.

``` scala
object Managed {
  type Closable = { def close(): Unit }

  def using[U](that: Closable)(block: => U): U = {
    try block
    finally that.close()
  }
}
```

Использование метода:

``` scala
  import Managed.using
  val x: Exchange = Exchange()

  using(x) {
    x write "foo"
  }
```

Теперь мы можем работать со всеми классами, у которых есть метод
`close`.

Но что делать, если у некоторого класса этот метод называется
по-другому? Тут на помощь приходят _imlicit conversions_.

Implicits
-----------

Допустим, вместо метода `close` трейт `Exchange` предоставляет метод
`cleanup`. Создадим _trait_ `Managed`, для которого определим метод
`using` в соответствующем companion-объекте:

``` scala
trait Managed[T] {
  def onClose(): Unit
}

object Managed {
  // ...

  def using[T, U](that: Managed[T])(block: => U): U = {
    try block
    finally that.onClose()
  }
}
```

Определим управляемый класс для `Exchange` и функцию неявного преобразования:

``` scala
class ManagedExchange(exchange: Exchange) extends Managed[Exchange] {
  def onClose() {
    exchange.cleanup()
  }
}

object ExchangeImplicits {
  implicit def Exchange2Managed(e: Exchange) = new ManagedExchange(e)
}
```

Осталось только добавить импорт неявных преобразований, определённых в
`ExchangeImplicits`:

``` scala
  import Managed.using
  import ExchangeImplicits._
  val x: Exchange = Exchange()

  using(x) {
    x write "foo"
  }
```

[^rx]: [Principles of Reactive Programming](https://class.coursera.org/reactive-001/class/index) by Erik Mejer, Martin Odersky, Roland Kuhn на [Coursera][coursera]

[coursera]: http://coursera.org "Coursera"

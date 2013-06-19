---
layout: post
title: "Юнит-тестирование в Racket: RackUnit"
date: 2013-06-19 22:30
comments: true
categories:
- Racket
- Unit Testing
- RackUnit
published: false
---

_Юнит-тестирование_ &mdash; это процесс проверки на корректность
отдельных единиц (модулей) исходного кода вместе со связанными
данными. Цель юнит-тестирования состоит в том, чтобы изолировать
отдельные части программы друг от друга и убедиться, что каждая
отдельная часть ведет себя корректно.

Подробнее о юнит-тестировании можно узнать [здесь][unit-testing].

Юнит-тестирование в Racket
==========================

Racket поставляется с собственным фреймворком для юнит-тестирования
[RackUnit][rackunit]. RackUnit позволяет использовать простые
проверки (<em>Checks</em>), группировать несколько связанных проверок в один
_Test Case_, который устанавливает линейную зависимость между
тестами: если какой-то тест провалится, то следующие тесты не будут
запущены. Checks и Test Cases выполняются сразу же, как только их
достигнет поток управления.

Существует еще один тип тестов &mdash; _Test Suite_. Test Suite'ы состоят из Test
Case'ов и других Test Suite'ов; их выполнение отложено до момента,
определенного программистом.

О философии, лежащей за этими концепциями, можно почитать
[тут][rackunit-philosophy].

Для написания тестов необходимо подключить модуль `rackunit`:

``` racket
(require rackunit)
```

<!-- more -->

[rackunit-philosophy]: http://docs.racket-lang.org/rackunit/philosophy.html "RackUnit Philosophy"


Простые проверки
----------------

В основе RackUnit лежат проверки (Checks), которые проверяют некоторое
условие. В случае, если условие выполняется, проверка вычисляется в
`#<void>`, в противном случае возбуждается исключение `exn:test:check`
с детальной информацией об ошибке. Проверка может содержать
необязательное сообщение, которое будет выводиться в случае ошибки.

RackUnit предоставляет различные типы проверок:

* проверки, которые соответствуют основным логическим предикатам
  языка:

``` racket
;; Проверка на эквивалентность, успешна
(check-eq? empty (rest '(1)))

;; Проверка на равенство, неудачна,
;; содержит дополнительное сообщение
(check-equal? '(1 2 3) "1 2 3" "List of integers is not a string")
```

* проверки на истину и ложь:

``` racket
(check-true (empty? '(1)))
(check-false #f "Obviously passes")
```

* проверки, которые принимают предикаты:

``` racket
(check-pred rational? 1/2)
```

* проверки, которые могут перехватывать исключения:

``` racket
(check-exn exn:fail:contract?
           (lambda ()
             (rest empty)))
```

* и другие. Полный список достаточно широк, и его нет
смысла приводить здесь; желающим предлагается изучить
[документацию][rackunit-checks].

К исключению, которое возникает, когда проверка не выполняется, можно
добавить дополнительную информацию типа `check-info` с помощью функции
`with-check-info*` и макроса `with-check-info`. Эта информацию
выведется вместе с сообщением об ошибке:

``` racket
(for-each
 (lambda (elt)
   (with-check-info
    (['current-element elt])
    (check-pred odd? elt)))
 '(1 3 5 7 8))
```

Можно определять новые проверки. Для этого существует несколько
макросов, один из которых &mdash; `define-simple-check` &mdash;
используется в следующем примере:

``` racket
(define-simple-check (check-multiple-of-five? number)
  (check-eqv? (remainder number 5) 0 (format "~a is not a multiple of five" number)))

(check-multiple-of-five? 26)
```

Test Cases
----------

To be written

Test Suites
-----------

To be written

Разное
======

To be written

Control Flow
------------

TBW

Expose
------

TBW

UI
--

TBW

Docs completeness
-----------------

TBW

Logging
-------

TBW

Advanced
========

Handlers
--------

[unit-testing]: http://en.wikipedia.org/wiki/Unit_testing "Wikipedia entry for Unit Testing"

[rackunit]: http://docs.racket-lang.org/rackunit/ "RackUnit Documentation"

[rackunit-checks]: http://docs.racket-lang.org/rackunit/api.html#(part._.Checks) "RackUnit Checks Documentation"

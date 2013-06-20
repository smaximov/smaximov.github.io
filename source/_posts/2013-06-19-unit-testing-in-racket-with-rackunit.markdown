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

Введение
========

_Юнит-тестирование_ &mdash; это процесс проверки на корректность
отдельных единиц (модулей) исходного кода вместе со связанными
данными. Цель юнит-тестирования состоит в том, чтобы изолировать
отдельные части программы друг от друга и убедиться, что каждая
отдельная часть ведет себя корректно.

Подробнее о юнит-тестировании можно узнать [здесь][unit-testing].

<mark>Disclaimer</mark>: по большей части этот пост представляет собой выборочный
перевод официальной документации.

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

С ростом сложности программы единица тестирования выходит за пределы
отдельной проверки. Часто возникает случай, что если какая-нибудь
проверка не выполняется, то нет смысла запускать другие проверки. Для
решения подобных проблем существуют составные тестирующие формы,
которые позволяют группировать выражения. Если любое выражение не
выполняется, то следующие выражения не проверяются; вся составная
форма считается не выполненной.

Простейшей составной тестирующей формой является _Test Case_. Test
Case можно объявить с помощью формы `test-begin`:

``` racket
(test-begin
 (check-false (empty? empty))
 (fail "This message will not be shown"))
```

В предыдущем примере первая же проверка не выполняется, что приводит к
тому, что последующие выражения  &mdash; `(fail ...)` &mdash; не
проверяются.

С помощью формы `test-case` можно указать имя для определяемого Test
Case'а; это имя будет выведено, если тест завершится неудачей.

Существует несколько вспомогательных процедур для объявления Test
Cases, информацию о них можно посмотреть в
[документации][rackunit-test-cases].

[rackunit-test-cases]:
http://docs.racket-lang.org/rackunit/api.html#(part._.Test_.Cases)
"RackUnit Test Cases Documentation"


Test Suites
-----------

Test Cases в свою очередь тоже могут быть сгруппированы в _Test
Suites_. Test Suite может содержать как Test Cases, так и другие Test
Units. В отличие от предыдущих видов тестов, Test Suite не запускается
сразу же, когда поток управления достигает определения Test Suite;
вместо этого программист должен пользоваться функциями, запускающими
Test Suites.

Создать Test Suite можно с помощью базовой формы `test-suite`. Она
принимает необязательные имя для Test Suite и действия (<em>thunks</em>),
предназначенные для выполнения до и после проверки Test Suite.

``` racket
(define my-test-suite
  (test-suite
   "My super test suite"
   #:before (lambda ()
              (display "Runs before the entire suite\n"))
   #:after (lambda ()
             (display "Runs after the entire suite\n"))
   (test-case
    "Named test case"
    (test-eq? "Checking the equality of numbers" 42 42))
   (test-suite
    "Nested test suite"
    (test-begin
     (check-not-exn
      (lambda ()
        #f))
     (check memq 'racket '(lisp common-lisp scheme racket))))))
```

Существуют макросы для упрощения определения Test Suite'ов. Макрос
`define-test-suite` создает Test Suite с заданным именем, приведенным
к строке, и связывает созданный тест с этим именем. Вышеприведенный
пример может быть переписан с помощью макроса `define-test-suite` так:

``` racket
(define-test-suite my-test-suite
   #:before (lambda ()
              (display "Runs before the entire suite\n"))
   #:after (lambda ()
             (display "Runs after the entire suite\n"))
  (test-case
   "Named test case"
   (test-eq? "Checking the equality of numbers" 42 42))
  (test-suite
   "Nested test suite"
   (test-begin
    (check-not-exn
     (lambda ()
       #f))
    (check memq 'racket '(lisp common-lisp scheme racket)))))
```

Макрос `define/provide-test-suite` ведет себя как `define-test-suite`,
только он еще импортирует (<em>provides</em>) Test Suite.

Разное
======

Рассмотрим другие возможности RackUnit.


Поток управления в тестах
-------------------------

Макросы `before`, `after` и `around` позволяют указать код, который
должен выполняться, соответственно, до, после и "вокруг" тестовых
выражений. Эти действия выполняются вне зависимости от того, возникли
ли в процессе выполнения тестого кода исключения.

В следующем примере, взятом из документации, проверяется, что файл
"test.dat" содержит строку "foo". Когда поток выполнения достигает
тестового выражения, выполняется `before`-действие, которое записывает
эту строку в файл. Когда поток выполнения покидает тестовый код,
выполнятся `after`-действие, которое удаляет созданный файл.

``` racket
(around
  (with-output-to-file "test.dat"
     (lambda ()
       (write "foo")))
  (with-input-from-file "test.dat"
    (lambda ()
      (check-equal? "foo" (read))))
  (delete-file "test.dat"))
```

Пользовательский интерфейс
--------------------------

RackUnit предоставляет текстовый и графический пользовательский
интерфейс.

Текстовый интерфейс предоставляется модулем `rackunit/text-ui`,
который определяет функцию `run-tests`, запускающую заданный тест.
Результаты теста выводятся в `current-output-port`.
Функция также может принимать параметр `verbosity`, который
контроллирует объем выводимой информации.

Модуль `rackinit/gui` предоставляет графический пользовательский
интерфейс. Функция `test/gui` создает окно, в котором запускаются
заданные тесты.

[unit-testing]: http://en.wikipedia.org/wiki/Unit_testing "Wikipedia entry for Unit Testing"

[rackunit]: http://docs.racket-lang.org/rackunit/ "RackUnit Documentation"

[rackunit-checks]: http://docs.racket-lang.org/rackunit/api.html#(part._.Checks) "RackUnit Checks Documentation"

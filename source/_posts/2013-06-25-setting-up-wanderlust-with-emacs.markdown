---
layout: post
title: "Настройка Wanderlust"
date: 2013-06-25 20:07
comments: true
categories:
  - Wanderlust
  - Emacs
  - el-get
published: false
---

Введение
========

В последнее время поймал себя на мысли, что я все меньше времени
провожу с Emacs. Чтобы как-то это исправить, я решил снова перевести
управление почтой на Emacs.

Предыдущая попытка с Gnus ни к чему не привела &mdash; он показался
мне неудобным в использовании, а настроить его под себя я
по-нормальному не осилил; в результате вернулся
сначала на Opera Mail, затем на Evolution.

Натыкался на положительные отзывы про [Wanderlust][wanderlust] в
сравнении с Gnus, поэтому в этот раз я решил попробовать именно его.

<!-- more -->

Установка
=========

C самого началя я столкнулся с проблемой: Wanderlust отсутствует в
ELPA, в репозиториях Fedora его тоже нет. Наиболее логичным в данном
случае мне показалось использование еще одного "пакетного менеджера"
для Emacs &mdash; [el-get][el-get]. Так как раньше я с ним не
сталкивался, опишу кратко процесс его установки и настройки. В
качестве справочного материала использовалась статья с
[EmacsWiki][el-get-emacswiki].

Установка и настройка el-get
----------------------------

Установка и подключение el-get происходит с помощью т.н.
_lazy-initializer_, который пытается подключить (`require`) el-get, и
если это не удается, вытягивает его из Github-репозитория:

``` scheme
(add-to-list 'load-path "~/.emacs.d/el-get/el-get")

(unless (require 'el-get nil t)
  (url-retrieve
   "https://github.com/dimitri/el-get/raw/master/el-get-install.el"
   (lambda (s)
     (end-of-buffer)
     (eval-print-last-sexp))))
```

Далее нужно добавить следующую строку:

``` scheme
(el-get 'sync)
```

На info-странице для el-get дается объяснение, зачем это нужно.
Вкратце: этот код проверяет, что все пакеты установлены (если нет, то
они устанавливаются автоматически), инициализированы и загружены
(через `require` или `autoload`). Подробную документацию для функции
`el-get`, как всегда, можно посмотреть через `describe-function`: `C-h
f el-get`.

Установка Wanderlust через el-get
---------------------------------

Установка Wanderlust через el-get элементарна: нужно выполнить `M-x
el-get-install RET wanderlust`.

Настройка
=========

Использование
=============

Заключение
==========

Ссылки
======

- [Wandurlust на Github][wanderlust]

- [Страница Wanderlust на EmacsWiki][wanderlust-emacswiki]

- [el-get на Github][el-get]

- [el-get на EmacsWiki][el-get-emacswiki]

[wanderlust]: https://github.com/wanderlust/wanderlust

[wanderlust-emacswiki]: http://www.emacswiki.org/emacs/WanderLust

[el-get]: https://github.com/dimitri/el-get

[el-get-emacswiki]: http://www.emacswiki.org/emacs/el-get

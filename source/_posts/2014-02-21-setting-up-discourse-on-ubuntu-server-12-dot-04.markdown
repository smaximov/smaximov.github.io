---
layout: post
title: "Поднимаем Discourse на Ubuntu Server 12.04"
date: 2014-02-21 21:04
comments: true
categories:
- Ruby
- Rails
- Discourse
- Ubuntu
---

* TOC
{:toc}

Введение
========

[Discourse](http://www.discourse.org) — это молодой и активно
развивающийся opensource-форум[^disc-platform], одним из создателей которого является
широко известный в узких кругах программист и блоггер Jeff Atwood[^atwood] a.k.a.
[codinghorror](http://blog.codinghorror.com).

Discourse отличается от традиционных форумов (прежде всего от phpBB,
vBulletin и их клонов) динамичностью, своим подходом к организации
обсуждений[^organisation], продвинутой системой уведомлений и
отслеживания сообщений, responsive-версткой и поддержной мобильных
браузеров. Ещё одно отличие от привычных форумов &mdash; ориентация на
community-модерацию, которая позволяет наиболее уважаемым и активным
участникам форума принимать участие в его управлении.

<!-- more -->

Настройка сервера
=================

Использовался сервер со следующими характеристиками:

* Ubuntu 12.04 LTS
* 4 Xeon vCPU
* 4GB ECC RAM
* 60GB SSD

Пререквизиты
--------------

Нам потребуется следующее ПО:

* Postgres 9.1+ в качестве СУБД.
* Redis 2+ для кэша, job queue и пр.
* nginx в качестве вэб-сервера.
* Postfix в качестве почтового сервера.

Дополнительное ПО:

* git --- для клонирования репозитория и управления кодом.
* make, g++ --- для сборки gem native extensions.
* libxml2, libxslt, postgresql-server-dev-9.1 --- зависимости при сборке.
* emacs --- для редактирования конфигов.

Установка проводится средствами пакетного менеджера.

Создание нового пользователя
----------------------------

Плохим тоном считается использование веб-сервисов из-под
рута, поэтому создадим нового пользователя и добавим его
в группу **sudo**[^sudo]:

``` bash
adduser --shell /bin/bash --gecos '' nameless
usermod -a -g sudo nameless
```

Сменим текущего пользователя (**root**) на **nameless**:

``` bash
su - nameless
```

**Внимание**: _все дальнейшие операции выполняются из-под пользователя **nameless**!_

Также создадим директорию, в которой будет находится Discourse, и
дадим на неё права новому пользователю:

``` bash
sudo install -d -m 755 -o nameless -g nameless /var/www/discourse
```

Установка Ruby
--------------

Бэкенд **Discourse** написан на [Ruby](https://www.ruby-lang.org/) с
использованием вэб-фреймворка
[Ruby on Rails](http://rubyonrails.org/). Для установки Ruby я буду
использовать [RVM](http://rvm.io/) --- Ruby Version Manager.

Установка RVM в случае использовании Bash как shell'а по умолчанию
предельна проста:

``` bash
curl -sSL https://get.rvm.io | bash -s stable
```

Далее, чтобы изменения вступили в силу, надо выполнить

``` bash
. ~/.rvm/scripts/rvm
```

или перезапустить текущую сессию.

С помощью RVM установим последнюю на текущий момент версию Ruby:

``` bash
# установить Ruby 2.1.0
rvm install 2.1.0
# использовать версию 2.1.0 по умолчанию
rvm use 2.1.0 --default
```

Также необходимо установить Bundler для управления зависимостями
gem'ов Discourse:

``` bash
gem install bundler
```

Получение исходников Discourse
-----------------

Склонируем репозиторий Discourse в заранее созданную директорию:

``` bash
git clone git://github.com/discourse/discourse.git /var/www/discourse
```

Перейдём в эту директорию и установим необходимые зависимости:

``` bash
cd /var/www/discourse/
bundle install
```

Подготовка Postgres к использованию
------------------

Добавим роль для текущего пользователя:

``` bash
sudo -u postgres createuser -s nameless
```

Создадим базу данных:

``` bash
createdb discourse_prod
```

Конфигурация Discourse
------------------

Перейдём в директорию `config` и скопируем шаблонные конфиги:

``` bash
cp discourse_quickstart.conf discourse.conf
cp discourse.pill.sample discourse.pill
```

### discourse.conf

Укажем имя хоста для форума:

```
# hostname running the forum; i.e the external address of your form.
hostname = "grant.smaximov.info"
```

Изменим настройки соединения с Postgres:

```
# database name running discourse, in INSTALL-ubuntu this is discourse_prod
db_name = discourse_prod

# host address for db server, uncomment if needed
db_host = localhost

# port running db server, uncomment if needed
# db_port = 5432

# username accessing database, if connecting remotely
db_username = nameless

# password used to access the db, if connecting remotely
db_password = secret

```

Настройки Redis можно оставить без изменения.

Инициализация Discourse
----------------------

Выполним первоначальную миграцию базы данных и компиляцию
статических файлов:

``` bash
RAILS_ENV=production bundle exec rake db:migrate
RAILS_ENV=production bundle exec rake assets:precompile
```

Настройка nginx
---------------

Скопируем конфигурационные файлы из под обычного пользователя:

``` bash
sudo cp /var/www/discourse/config/nginx.global.conf /etc/nginx/conf.d/local-server.conf
sudo cp /var/www/discourse/config/nginx.sample.conf /etc/nginx/conf.d/discourse.conf
```

Укажем имя сервера в `/etc/nginx.conf.d/discourse.conf`:

```
server_name grant.smaximov.info;
```

Осталось перезапустить nginx:

``` bash
sudo service nginx reload
```

Настройка Postfix
----------------

В файле `/etc/hostname` укажем доменное имя сервера.

Для пересылки писем будем использовать [Mandrill](mandrillapp.com).
Создадим там учётную запись и сгенерируем для неё API KEY, добавив
ограничения: доступ только с IP сервера и типы сообщений **send**,
**send-raw**.

Создадим файл `/etc/postfix/sasl/passwd` со следующим содержимым

```
[smtp.mandrillapp.com] MANDRILL_ACCOUNT:API_KEY
```

где `MANDRILL_ACCOUNT` --- учётная запись Mandrill, а `API_KEY` ---
сгенерированный для неё API KEY.

Ограничим права на этот файл:

``` bash
sudo chmod 600 /etc/postfix/sasl/passwd
```

Обновим настройки Postfix (эту команду следует выполнять при каждом
изменении файла):

``` bash
sudo postmap /etc/postfix/sasl/passwd
```

Отредактируем `/etc/postfix/main.cf`, настроив
**sasl**-аутентификацию:

```
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/sasl/passwd
smtp_sasl_security_options = noanonymous
smtp_use_tls = yes
```

Там же укажем релей:

```
relayhost = [smtp.mandrillapp.com]
```

Перезапустим Postfix:

```
sudo service postfix restart
```

Теперь проверим работоспособность конфигурации:

``` bash
sendmail your.real.email@gmail.com
From: admin@smaximov.info
Subject: Test message from Postfix
Make sure that everything is OK.
```

Если письмо не пришло, нужно смотреть `/var/log/syslog` на предмет
возможных проблем.

После этого можно проверить, отсылает ли Discourse почту. Для этого
нужно в админке Discourse перейти в **Email** → **Settings** и
отослать тестовое письмо на свой адрес.

Установка и настройка Bluepill
------------------------------

``` bash
gem install bluepill
echo 'alias bluepill="NOEXEC_DISABLE=1 bluepill --no-privileged -c ~/.bluepill"' >> ~/.bash_aliases
rvm wrapper $(rvm current) bootup bluepill
rvm wrapper $(rvm current) bootup bundle
```

Запускаем Discourse:

``` bash
RUBY_GC_MALLOC_LIMIT=90000000 RAILS_ROOT=/var/www/discourse \
    RAILS_ENV=production NUM_WEBS=4 bluepill --no-privileged \
    -c ~/.bluepill load /var/www/discourse/config/discourse.pill
```

Добавляем эту же строку в crontab при `@reload`.

Обновление Discourse
===========================

Для обновления Discourse можно использовать следующий скрипт:

``` bash
#!/bin/bash
# -*- mode: sh -*-

DATESTAMP=$(TZ=UTC date +%F-%T)
BACKUP_DIR="${HOME%%/}/backup"
DISCOURSE_DIR="/var/www/discourse"
DISCOURSE_PARENT="$(dirname ${DISCOURSE_DIR})"
DISCOURSE_NAME="$(basename ${DISCOURSE_DIR})"

bluepill --no-privileged stop
bluepill --no-privileged quit

mkdir -vp $BACKUP_DIR
pg_dump --no-owner --clean discourse_prod | gzip -c > "${BACKUP_DIR}/discourse-db-${DATESTAMP}.sql.gz"
tar cfz "${BACKUP_DIR}/discourse-dir-${DATESTAMP}.tar.gz" -C $DISCOURSE_PARENT $DISCOURSE_NAME

cd $DISCOURSE_DIR
git checkout master
git pull origin master
git fetch --tags

bundle install --without test --deployment
RUBY_GC_MALLOC_LIMIT=90000000 RAILS_ENV=production bundle exec rake db:migrate
RUBY_GC_MALLOC_LIMIT=90000000 RAILS_ENV=production bundle exec rake assets:precompile

eval "$(crontab -l | sed -e '/^#/d' -e 's/^@reboot //')"
```

Заключение
=============

Вот мы и получили работающий экземпляр Discourse. Что можно сделать
дальше?

* Настроить сайт под себя. Выбор опций для настройки достаточно велик:
от локали по умолчанию до параметров аутентификации и авторизации.

* Изменить внешний вид сайта: написать кастомные стили...

* Заняться начальным наполнением сайта.

* ...

Ссылки
========

* [Discourse Homepage](http://www.discourse.org/)

* [Discourse GitHub Repo](http://github.com/discourse/discourse)

* [Official Discource Install Guide](https://github.com/discourse/discourse/blob/master/docs/INSTALL-ubuntu.md)

Update
========

Тем временем не так давно появилось
[руководство](https://github.com/discourse/discourse/blob/master/docs/INSTALL-digital-ocean.md)
по поднятию Discourse с помощью образа [Docker](https://www.docker.io/).

[^sudo]: Это даст возможность использовать утилиту sudo без необходимости править файл `/etc/sudoers`
[^atwood]: [StackOverflow](https://stackoverflow.com) и [StackExchange](https://stackexchange.com) &mdash; тоже его рук дело
[^disc-platform]: &hellip;или Discussion Platform, как называют его авторы
[^organisation]: группировка по тегам вместо подфорумов, отсутствие разделения топиков на страницы

---
layout: post
title: "Настройка OpenVPN на Ubuntu Server"
date: 2014-02-27 23:46
comments: true
categories:
    - vpn
    - openvpn
    - ubuntu
published: false
---

# Установка

Самый удобный способ установки OpenVPN &mdash; с помощью пакетного
менеджера:

``` bash
sudo aptitude install openpvn
```

Также установим пакет **openssl**.

Запустим shell с правами суперпользователя, и можно приступать к настройке.

# Настройка

Скопируем конфиг и скрипты из примеров, идущих вместе с OpenVPN:


``` bash
zcat /usr/share/doc/openvpn/examples/sample-config-files/server.conf.gz > /etc/openvpn/server.conf
mkdir /etc/openvpn/rsa
cp -R /usr/share/doc/openvpn/examples/easy-rsa/2.0/* /etc/openvpn/rsa
```

Перейдём в эту директорию.

В директории `/etc/openvpn/rsa` располагается небольшой набор средств
для создания и управления RSA-ключами. В файле `README.gz` дана
небольшая справка, там же приведён алгоритм действий, которого мы
постараемся придерживаться.

Нам предлагается отредактировать файл `vars`, определяющий переменные
окружения, используемые скриптами. В большинстве случаев можно
использовать значения по умолчанию, в моём же случае не удалось
определить используемуй конфиг openssl, так что пришлось немного
отредактировать скрипт `whichopensslcnf`.

Выполним этот файл, определив переменные:

``` bash
. vars
```

Генерация ключей выполняется с помощью скрипта `pkitool`, однако для
него есть более удобные обёртки `build-*`, которые мы и будем использовать:

``` bash
# Очистим директорию `$KEY_DIR` (`/etc/openvpn/rsa/keys`):
./clean-all
# Server-side параметры соединения SSL/TLS
./build-dh
# Корневой сертификат и ключ к нему
./build-ca
# Пара сертификат/ключ для сервера server
./build-key-server server
# Пара сертификат/ключ для клиента nameless
./build-key nameless
```

Теперь отредактируем файл конфигурации `/etc/openvpn/server.conf`:

```
# Порт
port 1194

# TCP или UPD?
proto udp

# routed tunnel
dev tun

# Сертификаты и ключи
ca /etc/openvpn/rsa/keys/ca.crt
cert /etc/openvpn/rsa/keys/server.crt
key /etc/openvpn/rsa/keys/server.key  # This file should be kept secret

# Diffie hellman parameters.
dh /etc/openvpn/rsa/keys/dh1024.pem

# Подсеть и маска
server 192.168.1.0 255.255.255.0

# Одинаковые IP-адреса при переподключении для клиентов
ifconfig-pool-persist ipp.txt

# Ping-like сообщения для проверки, что стороны живы.
# ping каждые 10 секунд, таймаут 120 секунд означает,
# что другая сторона упала
keepalive 10 120

# Компрессия (должна быть включена и на клиенте)
comp-lzo

persist-key
persist-tun

# Файл статуса, перезаписываемый каждую минуту
status /var/log/openvpn-status.log

# Файл логов
log /var/log/openvpn.log

# Уровень выводимых в лог сообщений
verb 3

# Гугловский DNS
push "dhsp-option DNS 8.8.8.8"
```

# Запуск и проверка

Запустим сервер

``` bash
service openvpn start
```

Скопируем созданные сертификаты `ca.srt`, `nameless.crt` и ключ
`nameless.key` (например, с помощью `scp`).

Создадим подключение, используя эти файлы (в свойствах подключения
надо указать lzo-компрессию).

Посмотрим логи openvpn на предмет ошибок, если всё хорошо, двигаемся
дальше.

# NAT

Включим IP-forwarding:

``` bash
echo 1 > /proc/sys/net/ipv4/ip_forward
echo net.ipv4.ip_forward=1 >> /etc/sysctl.conf
```

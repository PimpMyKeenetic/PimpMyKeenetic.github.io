---
published: true
layout: post
title: Авторизация в Debian с использованием PAM-модуля NDMS
---
Это пример использования учётных данных, определённых через в веб-интерфейс кинетика для авторизации пользователей в [chroot-среде Debian](/2017/06/21/debian-via-chroot/) с помощью модуля [pam_ndm](https://github.com/ndmsystems/pam_ndm).

Для упрощения все действия лучше вести прямо на кинетике, используя нативную компиляцию.

### Сборка libndm

`pam_ndm` использует [libndm](https://github.com/ndmsystems/libndm) для обращению к ядру NDMS. Подготовка среды сборки и получение исходников:

```
# cd ~
# apt-get install build-essential git
# git clone https://github.com/ndmsystems/libndm.git
# cd libndm

```
Есть нюанс: в новых версиях прошивки обращение к ядру NDMS ведётся не через TCP-соединение, а через UNIX socket, который в chroot-среде не доступен, поэтому надо скачать патч, возвращающий поддержку работу через TCP, благо этот способ всё ещё работает:
```
# wget https://raw.githubusercontent.com/ndmsystems/entware/master/libndm/patches/010-legacy-tcp-support.patch
# patch -p1 -i 010-legacy-tcp-support.patch
# make install
```

### Сборка pam_ndm

Получение необходимых файлов:
```
# cd ~
# apt-get install libpam0g-dev
# git clone https://github.com/ndmsystems/pam_ndm.git
# cd pam_ndm
```
Сборка и установка:
```
# make 
# strip pam_ndm.so
# chmod 644 pam_ndm.so
# chown root:root pam_ndm.so
# mv pam_ndm.so /lib/mipsel-linux-gnu/security
```

### Настройка

Минимально необходимая настройка сводится к добавлению в начало `/etc/pam.d/common-auth` строчек
```
auth            sufficient      pam_ndm.so
account         sufficient      pam_ndm.so
```
и созданию пары пользователей: в Debian и в веб-интерфейсе кинетика. К примеру, создайте в Debian пользователя testuser:
```
# adduser testuser
```
с присвоением произвольного пароля. Затем [создайте](http://my.keenetic.net/#tools.users) пользователя прошивки testuser с одноимённым паролем. У пользователя обязательно должен быть тег доступа `OptWare`.

### Проверка

Если всё сделано правильно, то можно начать SSH-сеанс, используя учётные данные `testuser:testuser`. При этом в логе роутера будут видны обращения к ядру прошивки следующего вида:

```
[W] Jun 23 17:06:05 ndm: Core::Server: started obsoleted TCP Session 127.0.0.1:48348.
[I] Jun 23 17:06:05 ndm: Core::Authenticator: generating.
[I] Jun 23 17:06:05 ndm: Core::Authenticator: user "testuser" authenticated, realm "", tag "opt".
[I] Jun 23 17:06:05 ndm: Core::Server: client disconnected.
[I] Jun 23 14:06:05 sshd[6259]: Accepted password for testuser from 192.168.6.2 port 52393 ssh2
[I] Jun 23 14:06:05 sshd[6259]: pam_unix(sshd:session): session opened for user testuser by (uid=0)
```

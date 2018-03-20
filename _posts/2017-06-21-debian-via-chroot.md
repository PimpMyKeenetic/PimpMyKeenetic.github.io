---
layout: post
title: Debian Stretch и ничего лишнего
published: true
---
<p class="message">
Debian рекомендуется использовать на роутерах с достаточным количеством ресурсов процессора и памяти — Giga III, Ultra II.
</p>

На профильном форуме уже есть схожая [тема](https://forum.keenetic.net/topic/458-debian-stable/), которая является адаптацией [этого](https://github.com/DontBeAPadavan/chroot-debian) решения, но кинетике это можно сделать куда изящнее. Компонент OPKG умеет всё сам.

В итоге будет получена полноценная среда Debian, в которой не будет ни одного файла выше корня chroot, т.е. сущности из busybox'а с аплетом chroot и костыли для исполнения скриптов в `/opt/etc/ndm/*.d`, которые ранее были "внешними" по отношению к Debian'у больше не нужны.

Примером успешного использования на кинетике chroot'ed Debian может служить сервер [files.keenopt.ru](http://files.keenopt.ru/).

Если не заинтересованы в самостоятельной подготовке установочного архива, сразу переходите к разделу [Установка на роутере](#%D1%83%D1%81%D1%82%D0%B0%D0%BD%D0%BE%D0%B2%D0%BA%D0%B0-%D0%BD%D0%B0-%D1%80%D0%BE%D1%83%D1%82%D0%B5%D1%80%D0%B5).

### Подготовка на ПК

На Debian-машине подготовливаем архив с полуфабрикатом:
```
#!/bin/sh

DEB_FOLDER=debian

### Are we root or mere mortal?
if [ $(id -u) -ne 0 ] ; then
    echo 'Please run it as root'
    exit 1
fi

### First stage of debotsrap
apt-get install debootstrap
debootstrap --arch mipsel --foreign --variant=minbase \
--include=openssh-server stable $DEB_FOLDER ftp://ftp.debian.org/debian

### Make f\w hook dirs
for folder in button fs netfilter schedule time usb user wan; do
    mkdir -p $DEB_FOLDER/etc/ndm/${folder}.d
done

### Place default evnvironment vars to /etc/ndm/env_vars.sh
cat > $DEB_FOLDER/etc/ndm/env_vars.sh << EOF

### Unset NDMS-specific env vars
#unset HOSTNAME
#unset IFS
unset LD_BIND_NOW
unset LD_LIBRARY_PATH
#unset PS1
#unset PS2
#unset PS4
#unset SHLVL
#unset timezone

### Set Debian-speceific env vars
export LANG=en_US.UTF-8
export TERM=xterm
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
export SHELL=/bin/bash
export MAIL=/var/mail/root
export LOGNAME=root
export USER=root
export USERNAME=root
export HOME=/root

### Fix broken vars
[ "\$PWD" = "(unreachable)/" ] && export PWD=/
EOF

### Make start script for chroot'ed daemons
cat > $DEB_FOLDER/etc/ndm/initrc.sh << EOF
#!/bin/sh

### Fix env vars
. /etc/ndm/env_vars.sh

### Is /etc/ndm/services.txt exists?
CHROOT_SERVICES_LIST=/etc/ndm/services.txt
if [ ! -e "\$CHROOT_SERVICES_LIST" ]; then
    echo ssh > \$CHROOT_SERVICES_LIST
fi

### Start/Stop services
cd /root
for item in \$(cat \$CHROOT_SERVICES_LIST); do
    /etc/init.d/\$item \$1
done
EOF
chmod +x $DEB_FOLDER/etc/ndm/initrc.sh

### The second deboostrap stage should be done on router
tar -cvzf debian_clean.tgz $DEB_FOLDER
```

Архив надо положить в папку `/opt` на USB-носителе кинетика, где предварительно развёрнут Entware.

### Подготовка на роутере

В SSH-сессии на роутере:
```
opkg install tar
tar -xvzf debian_clean.tgz
mount /dev/ debian/dev
mount /proc/ debian/proc
mount /sys/ debian/sys
LC_ALL=C LANGUAGE=C LANG=C chroot debian /debootstrap/debootstrap --second-stage
echo 'PermitRootLogin yes' >> debian/etc/ssh/sshd_config
LC_ALL=C LANGUAGE=C LANG=C chroot debian /bin/bash
echo -e 'zyxel\nzyxel' | passwd # set 'zyxel' password for root
apt-get clean
exit
umount debian/dev
umount debian/proc
umount debian/sys
rm debian/var/lib/apt/lists/*
tar -cvzf debian_keenetic.tgz -C debian .
```

[Архив `debian_keenetic.tgz`](/assets/2017-06/debian_keenetic.tgz) готов для многократного использования. В случае, если среда Debian стала неработоспособной, всегда можно отформатировать раздел на USB-носителе в EXT2/3/4 и начать по новой. Сохраните его на ПК.

### Установка на роутере

1. Подготовьте чистый раздел на USB-накопителе, предварительно отформатированный в EXT2/3/4, 
2. Подключите носитель к кинетику,
3. Запишите подготовленный ранее [архив `debian_keenetic.tgz`](/assets/2017-06/debian_keenetic.tgz) в папку `install` на выбранном разделе с помощью FTP или SAMBA,
4. В CLI роутера выполните:
```
opkg initrc /opt/etc/ndm/initrc.sh
opkg chroot
system configuration save
```
5. В веб-интерфейсе [выберите](http://my.keenetic.net/#usb.opkg) соответствующий раздел в списке `Использовать накопитель` и нажмите `Применить`.

На [странице](http://my.keenetic.net/#tools.log) системного лога можно наблюдать за процессом распаковки архива и запуском службы SSH. 

### Использование 

Учётные данные для SSH-сессии `root:zyxel` лучше сразу поменять после первого входа. 

Все файлы, ответственные за конфигурирование chroot-среды лежат только в `/etc/ndm`, в частности:
- `initrc.sh` запускает и останавливает сервисы перечисленные в `services.txt`,
- папки `*.d` [служат](https://github.com/ndmsystems/packages/wiki/Opkg-Component#hook-scripts) для реакции на различные события прошивки,
- `env_vars.sh` рекомендуется включать в состав любого вызова хук скриптов для корректировки передаваемых из хост-системы переменных среды.

### Детали

Из ненаписанного в документации. Если в CLI задан параметр `opkg chroot`, то после старта роутера и перед выполнением `/etc/ndm/initrc.sh` компонент OPKG смонтирует (предварительно создав, если их нет) ряд папок:

- `/proc` в `/opt/proc`,
- `/sys` в `/opt/sys`,
- `/tmp` в `/opt/tmp`,
- `/dev` в `/opt/dev` и
- `devpts` в `/opt/dev/pts`.

Ещё OPKG при необходимости перепишет в `/opt/etc` файлы `/etc/shells`, `/etc/profile`, `/tmp/passwd` и `/tmp/group`.

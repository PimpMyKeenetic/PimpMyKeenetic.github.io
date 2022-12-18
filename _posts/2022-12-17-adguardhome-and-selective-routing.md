---
published: true
layout: post
title: AdGuard Home для выборочного обхода блокировок
---

В самом [алгоритме](https://keenetic-gi.ga/2018/01/16/selective-routing.html) нет ничего нового, но благодаря недавнему [обновлению](https://github.com/AdguardTeam/AdGuardHome/releases/tag/v0.107.13) AdGuard Home его можно реализовать довольно изящно: использование VPN для заданных вами доменов возможно тройкой коротких скриптов. 

### Требования

* Развёрнутая [среда Entware](https://forum.keenetic.net/topic/4299-entware/),
* Рабочее VPN-соединение поверх провайдерского, по которому будет идти обращение к определённым доменам.

### Установка пакетов

Подразумевается, что AdGuard Home (далее AGH) будет использоваться вместо встроенной в прошивку службы DNS Proxy. Установите необходимые пакеты:
```
opkg install adhuardhome-go ipset iptables
```
Выполните первоначальную настройку AGH, для чего в CLI роутера наберите:
```
opkg dns-override
system configuration save
```
В этот момент Entware-сервисы будут перезапущены, а интерфейс для первоначальной настройки AGH станет доступен по адресу `http://192.168.1.1:3000`, где 192.168.1.1 — IP-адрес роутера. Настройки по умолчанию подходят для большинства случаев.

### Скриптовая обвязка

Результаты разрешения указанных доменных имён AGH будет помещать в ipset (bypass в примере), правилами iptables трафик по этим IP-адресам будет помечаться fwmark, а для роутинга помеченного трафика будет использоваться своя таблица маршрутизации. 

Первый скрипт — создание ipset'а при старте роутера. Создайте файл `/opt/etc/init.d/S52ipset` со следующим содержимым:
```
#!/bin/sh

PATH=/opt/sbin:/opt/bin:/opt/usr/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

if [ "$1" = "start" ]; then
    ipset create bypass hash:ip
    ip rule add fwmark 1001 table 1001
fi
```
Второй — для поддержания актуальности таблицы роутинга. Файл `/opt/etc/ndm/ifstatechanged.d/010-bypass-table.sh` с содержимым:
```
#!/bin/sh

[ "$system_name" == "nwg0" ] || exit 0
[ ! -z "$(ipset --quiet list bypass)" ] || exit 0
[ "${change}-${connected}-${link}-${up}" == "link-yes-up-up" ] || exit 0

if [ -z "$(ip route list table 1001)" ]; then
    ip route add default dev $system_name table 1001
fi
```
где nwg0 — сетевой интерфейс VPN-соединения для выборочного обхода блокировок. Если затрудняетесь в его поиске, то посмотрите вывод `ip addr`, его адрес будет совпадать с тем, что виден в веб-интерфейсе.

Третий — пометка трафика с помощью fwmark. Актуальная прошивка активно использует эту возможность, поэтому маркировать трафик приходится точечно. Создайте файл `/opt/etc/ndm/netfilter.d/010-bypass.sh` c контентом:
```
#!/bin/sh

[ "$type" == "ip6tables" ] && exit
[ "$table" != "mangle" ] && exit
[ -z "$(ip link list | grep nwg0)" ] && exit
[ -z "$(ipset --quiet list bypass)" ] && exit

if [ -z "$(iptables-save | grep bypass)" ]; then
     iptables -w -t mangle -A PREROUTING ! -s 192.168.254.0/24 -m conntrack --ctstate NEW -m set --match-set bypass dst -j CONNMARK --set-mark 1001
     iptables -w -t mangle -A PREROUTING ! -s 192.168.254.0/24 -m set --match-set bypass dst -j CONNMARK --restore-mark
fi
```
где nwg0 — снова сетевой интерфейс VPN-соединения, 192.168.254.0/24 — его подсеть. Её так же можно найти в выводе `ip addr`.

Сделайте скрипты исполняемыми:
```
chmod +x /opt/etc/init.d/S52ipset
chmod +x /opt/etc/ndm/ifstatechanged.d/010-bypass-table.sh
chmod +x /opt/etc/ndm/netfilter.d/010-bypass.sh
```
и переходите к финальному пункту.

### Список доменов для обхода блокировок

Найдите в конфигурационном файле AGH `/opt/etc/AdGuardHome/AdGuardHome.yaml` строчку `ipset_file: ""` и поменяйте на `ipset_file: /opt/etc/AdGuardHome/ipset.conf`.

Файл `/opt/etc/AdGuardHome/ipset.conf` будет единственным, требующим редактирования время от времени, в зависимости от изменения вашего персонального списка доменов для разблокировки. Он имеет следующий [синтаксис](https://github.com/AdguardTeam/AdGuardHome/wiki/Configuration#configuration-file):
```
intel.com,ipinfo.io/bypass
instagram.com,cdninstagram.com/bypass
epicgames.com,gog.com/bypass
```
Т.е. в левой части через запятую указаны домены, требующие обхода блокировок, справа после слэша — ipset, в который AGH складывает результаты разрешения DNS-имён. Можно указать всё в одну строчку, можно разделить логически на несколько строк как в примере. Домены третьего уровня и выше также включаются в обход блокировок, т.е. указание intel.com включает www.intel.com, download.intel.com и пр.
Рекомендую добавить какой-нибудь «сигнальный» сервис, показывающий ваш текущий IP-адрес (ipinfo.io в примере). Так вы сможете проверить работоспособность настроенного решения. Учтите, что AGH не перечитывает изменённый файл, поэтому после правки перезапустите его с помощью:
```
/opt/etc/init.d/S99adguardhome restart
```

При желании можно использовать несколько VPN-соединений для обращения к разным доменам, для простоты понимания здесь это не описано.

### Диагностика проблем

* Убедитесь в том, что набор ipset создан и наполняется в процессе работы:
```
ipset --list bypass
```
Вывод должен быть не пустой.
* Убедитесь в существовании нужной таблицы роутинга для обхода блокировок:
```
ip rule list | grep 1001
```
* Убедитесь, что в таблице присутствует необходимый маршрут:
```
ip route list table 1001
```
* Посмотрите, существуют ли правила netfilter для пометки пакетов:
```
iptables-save | grep bypass
```
* После перезагрузки роутера проверьте в веб-интерфейсе Системный журнал, в нём не должно быть красных строк, связанных с настроенными скриптами.

Удачи в начинаниях!

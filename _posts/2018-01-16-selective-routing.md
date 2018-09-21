---
published: true
layout: post
title: Выборочный роутинг до перечисленных доменов
---

Решение ниже позволяет использовать имеющееся VPN-соедиенение исключтельно для списка выбранных (блокируемых) доменов, в то время как любой другой трафик будет идти по привычному соединению с провайдером, не подвергаясь ни "тормозам", ни прочим издержкам VPN. 

Этот алгоритм [таскаю](http://wl500g.info/showthread.php?30870&langid=3) из [прошивки](https://github.com/RMerl/asuswrt-merlin/wiki/Using-ipset/c165ebd44232a09c5dcbd1e5b33b0f7bdfc15ceb) в [прошивку](https://github.com/DontBeAPadavan/rublock-via-vpn) уже не первый год, он заключён в следующем:

* вместо прошивочного будет использован собственный резолвер `dnsmasq`, умеющий складывать результаты резовинга перечисленных в конфиге доменов в отдельный [ipset](http://ipset.netfilter.org/),
* на транзитных пакетах к/от выбранных ресурсов средствами iptables ставится определённая [метка fwmark](http://www.austintek.com/LVS/LVS-HOWTO/HOWTO/LVS-HOWTO.fwmark.html),
* для помеченных пакетов используется отдельная таблица роутинга, отправляющая помеченные пакеты в VPN-туннель.


### Требования

* Последняя Draft-прошивка,
* Развёрнутая [среда Entware](https://forum.keenetic.net/topic/4299-entware/),
* Рабочее VPN-соединение или EoIP/GRE-туннель поверх провайдерского, по которому будет идти обращение в выбранным доменам. 

В случае GRE-туннеля, организованного логикой прошивки, сетевой интерфейс в прошивке будет называться `ngreX`, в случае использования встроенного клиента OpenVPN — `ovpn_br0`. Если будете использовать встроенный OpenVPN, то не забудьте в поле конфигурации добавить опцию `route-noexec`, чтобы не сделать VPN-соединение маршрутом по умолчанию для всего трафика. Настройка серверной части GRE/OpenVPN в этой статье не рассматривается. Ниже для единообразия в скриптах будет использоваться имя интерфейса `ovpn_br0`.


### Настройка пакетов Entware

Установите необходимые пакеты:
```
opkg install dnsmasq-full ipset iptables
```
Замените `/opt/etc/dnsmasq.conf` содержимым:
```
### General settings
user=nobody
no-poll
bogus-priv
no-negcache
clear-on-reload
bind-dynamic
min-port=4096
cache-size=1536
domain=lan
expand-hosts
log-facility=/dev/null
log-async

### Using Yandex Safe DNS as default resolver
no-resolv
server=77.88.8.88#1253
server=77.88.8.2#1253

### ISP bypass list
ipset=/oip.cc/rublock
ipset=/kinozal.tv/rublock
```
В конце этого файла добавьте те домены, к которым необходимо обращаться по VPN, в моём примере это oip.cc и kinozal.tv. Оставьте в списке как минимум первый, он будет полезен для проверки того, что всё работает как надо. 
<p class="message">
Обратите внимание, что с переключением на собственный DNS-сервис вы потеряете возможности встроенного. К примеру, назначение разных профилей Яндекс.DNS/SkyDNS/AdGuard для клиентов. В строчках `server=…` вы можете указать самостоятельно выбранные DNS-серверы, которые будут использоваться для разрешения DNS-имён. В примере установлены серверы Яндекс.DNS, до этого момента не замеченного в DNS-спуфинге по каким-либо спискам блокировки.  
</p>

Добавьте скрипты, которые периодически будет вызывать прошивка. Первый, `/opt/etc/ndm/fs.d/100-create_ipsets_and_routing_tables.sh`:

```
#!/bin/sh

[ "$1" != "start" ] && exit 0

### Create set
ipset create rublock hash:ip

### Create routing tables for marked packets
if [ -z "$(ip route list table 1)" ]; then
    ip rule add fwmark 1 table 1
    ip route add default dev ovpn_br0 table 1
fi

exit0
```
Второй, `/opt/etc/ndm/netfilter.d/100-fwmarks.sh`:
```
#!/bin/sh

[ "$type" == "ip6tables" ] && exit 0
[ "$table" != "mangle" ] && exit 0

[ -z "$(iptables-save | grep rublock)" ] && \
    iptables -w -A PREROUTING -t mangle -m set --match-set rublock dst,src -j MARK --set-mark 1

exit 0
```
и не забудьте сделать их исполняемыми:
```
chmod +x /opt/etc/ndm/fs.d/100-create_ipsets_and_routing_tables.sh
chmod +x /opt/etc/ndm/netfilter.d/100-fwmarks.sh
```

### Настройка прошивки

Следующая команда в CLI Кинетика перезапустит окружение Entware, поэтому SSH-сеанс оборвётся:
```
opkg dns-override
system configuration save
```
Встроенная в прошивку служба разрешения DNS-имён будет выключена, и вместо неё будет запущен собственный сервис `dnsmasq` из состава Entware.

### Использование 

В связи с тем, что решение использует разрешение DNS-имён, необходимо почистить DNS-кэш браузера и операционной системы на кленте, к примеру, в моём случае мне приходится на Windows перезапустить Firefox и выполнить в консоли `ipconfig /flushdns`. Это действие разовое, нужное только для проверки работоспособности решения без перезагрузки Windows.

Откройте в браузере пару сайтов: [oip.cc](http://oip.cc/) и [whatismyip.com](https://www.whatismyip.com/)- одних из многих сервисов проверки вашего IP-адреса. Если в первом случае будет показан IP-адрес VPN-сервера, а во втором IP-адрес, выданный провайдером, то всё настроено верно: VPN-туннель рабоспособен, а обращения к указанным вами доменам идут по отдельному пути.

Если что-то пошло не так, то…

### Диагностика проблем

* Убедитесь, что dnsmasq запущен и виден в выводe `ps`,

* Убедитесь, что ipset-набор создан и наполняется, это можно увидеть в выводе `ipset --list`,

* Убедитесь, что отработало правило файервола, помечающее пакеты:

```
iptables-save | grep rublock

-A PREROUTING -m set --match-set rublock dst,src -j MARK --set-xmark 0x1/0xffffffff
```

* Посмотрите, что существует таблица роутинга для помеченных пакетов:

```
ip rule list

0:      from all lookup local
220:    from all lookup 220
1000:   from all fwmark 0x1 lookup 1
32766:  from all lookup main
32767:  from all lookup default
```

* Убедитесь, что в нужной таблице пакеты отправляются в правильный сетевой интерфейс, соответвующий VPN-соединению:

```
ip route list table 1

default dev ovpn_br0 scope link
```

Удачи в начинаниях!

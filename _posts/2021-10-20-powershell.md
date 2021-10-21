---
layout: post
title: Powershell в помощь
published: true
---

### Назначение

В некоторых случаях можно автоматизировать управление роутером без USB-накопителя и/или развёрнутого Entware с помощью Powershell. 
На Linux он портрирован не полностью и приведённый ниже пример не работает, но для Windows клиентов всё будет работать отлично.
Для упрощения работы с Кинетиками был написан [модуль](https://www.powershellgallery.com/packages/Keenetic), снимающий с пользователя рутинные мелочи. Работа с ним выглядит примерно так:

[![asciicast](https://asciinema.org/a/399922.svg)](https://asciinema.org/a/399922)

### Пример

В рамках заводского функционала не предусмотрено несколько Wireless ISP с возможностью перехода между ними. Скрипт ниже позволяет при пропадании интернета переключиться на другую известную Wi-Fi сеть, если она в зоне доступа роутера и удовлетворяет требованиям по силе сигнала.
Wi-Fi cети перебираются до тех пор, пока не будет восстановлена связь с интернетом.
Исполнение скрипта можно назначить в планировщике заданий для периодического выполнения, например, раз в минуту. Либо по триггеру на подходящие [события потери connectivity](https://answers.microsoft.com/en-us/windows/forum/all/network-connection-event-logs/517b72ce-240b-4f86-a560-e11a86ed03be) в системном логе.


```powershell
#Requires -Modules Keenetic

$Target = 'http://192.168.1.1'
$User = 'admin'
$Password = 'P@ssw0rd'

$KnownNetworks = @{
    'Blink' = 'Blink000'; 
    'ZyXEL_KEENETIC_LITE_3D4B72' = '24680135';
    'Ilya_2G_' = '123456788';
}

$SignalQualityThreshold = 50 # percent
$WISPConnectionTimeout = 30 # seconds

### Place your own settings above. No mere mortal should pass behind this line

# Do nothing if WISP is fine
if ((Test-NetConnection).PingSucceeded) {
    Write-Host 'WISP connection seems alive.'
    exit
}

New-KNSession -Target $Target -Credential (New-Object System.Management.Automation.PSCredential($User, (ConvertTo-SecureString $Password -AsPlainText -Force))) -AsDefaultSession
# Get Wi-Fi networks around with sufficient signal quality
$AvailNets = (Invoke-KNRequest -Body '{"show":{"site-survey":{"name":"WifiMaster0"}}}').show.'site-survey'.ap_cell | where quality -ge $SignalQualityThreshold

# Make some sanity check
if ($AvailNets.Count -eq 0) {
    Write-Warning 'Air is too silent, exiting.'
    exit
}
$KnownAvailNets = $AvailNets | where {$KnownNetworks.Contains($PSItem.essid)}
$CurrentSSID = (Invoke-KNRequest 'interface/WifiMaster0/WifiStation0').ssid
if ($KnownAvailNets.Count -eq 0) {
    Write-Warning 'No known Wi-Fi nets around, exiting.'
    exit
} elseif (($KnownAvailNets.Count -eq 1) -and ($KnownNetworks.Contains($CurrentSSID))) {
    Write-Warning 'The only know Wi-Fi net in air already set, exiting.'
    exit
}

# Try to connect any know available net
$Request = @'
[{"interface":{"WifiMaster0/WifiStation0":{"up":{"no":false},"ssid":{"ssid":"PLACEHOLDER"},
"mac":{"bssid":{"no":true}},"schedule":{"no":true},"description":"PLACEHOLDER",
"encryption":{"enable":{"no":false},"wpa":{"no":true},"wpa2":{"no":false},"owe":{"no":true},"wpa3":{"no":true}},
"authentication":{"wpa-psk":{"psk":"PLACEHOLDER"}},"ip":{"address":{"no":true,"dhcp":true},"name-server":[{"no":true}],
"gateway":{"no":true},"dhcp":{"client":{"name-servers":{"no":false},"hostname":"PLACEHOLDER"}},"global":{"order":1}}}}},
{"interface":{"WifiMaster0/WifiStation0":{"ping-check":{"profile":{"no":true}}}}},{"system":{"configuration":{"save":true}}}]
'@ | ConvertFrom-Json
foreach ($Net in $KnownAvailNets | where {$PSItem.essid -ne $CurrentSSID}) {   
    $Request[0].interface.'WifiMaster0/WifiStation0'.description = $Net.essid
    $Request[0].interface.'WifiMaster0/WifiStation0'.ssid.ssid = $Net.essid
    $Request[0].interface.'WifiMaster0/WifiStation0'.authentication.'wpa-psk'.psk = $KnownNetworks[$Net.essid]
    $Request[0].interface.'WifiMaster0/WifiStation0'.ip.dhcp.client.hostname = (New-Guid).ToString().Split('-')[0]
    Invoke-KNRequest -Body ($Request | ConvertTo-Json -Depth 10 -Compress) | Out-Null
    Start-Sleep -Seconds $WISPConnectionTimeout
    if ((Test-NetConnection).PingSucceeded) {
        Write-Host ('WISP connection repaired, connected to {0}.' -f $Net.essid)
        exit
    }
}

Write-Warning 'Gave up to fix WISP connection, exiting.'
```

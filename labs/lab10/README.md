# NAT, DHCP, NTP

### Задание

1. Настроить NAT (PAT) на R14 и R15. Трансляция должна осуществляться в адрес автономной системы AS1001.
2. Настроить NAT (PAT) на R18. Трансляция должна осуществляться в пул из 5 адресов автономной системы AS2042.
3. Настроить статический NAT для R20.
4. Настроить NAT так, чтобы R19 был доступен с любого узла для удалённого управления.
5. Настроить статический NAT (PAT) для офиса Чокурдах.
6. Настроить для IPv4 DHCP-сервер в офисе Москва на маршрутизаторах R12 и R13. VPC1 и VPC7 должны получать сетевые настройки по DHCP.
7. Настроить NTP-сервер на R12 и R13. Все устройства в офисе Москва должны синхронизировать время с R12 и R13.
8. Все офисы в лабораторной работе должны иметь IP-связность.

### Решение

1. [Схема и сводка](#1-схема-и-сводка)
2. [NAT в Москве](#2-nat-в-москве)
3. [NAT в С.-Петербурге](#3-nat-в-с-петербурге)
4. [NAT в Чокурдахе](#4-nat-в-чокурдахе)
5. [DHCP](#5-dhcp)
6. [NTP](#6-ntp)
7. [Особенности стенда](#7-особенности-стенда)
8. [Проверка](#8-проверка)
9. [Конфигурационные файлы](configs/)

Работа продолжает [лабу 09](../lab09/). Файл стенда для EVE-NG: [lab10.unl](lab10.unl).

### 1. Схема и сводка

![](scheme.png)

| Где | Что | Inside | Адрес снаружи |
|---|---|---|---|
| R14, R15 | PAT | 10.10.0.0/16 | Loopback1: 100.1.0.14, 100.1.0.15 |
| R15 | статический NAT для R20 | 10.10.255.20 | 100.1.0.20 |
| R15 | статический PAT для R19 (telnet) | 10.10.255.19:23 | 100.1.0.19:23 |
| R18 | PAT в пул | 10.20.0.0/16 | 20.42.0.1 - 20.42.0.5 |
| R28 | PAT в адрес линка | 10.30.0.0/16 | 52.0.1.14 (к R26) или 52.0.1.18 (к R25) |
| R12, R13 | DHCP | VLAN 10, VLAN 70 | |
| R12, R13 | NTP-серверы | | |

### 2. NAT в Москве

**PAT на R14 и R15.** Внутренние сети Москвы (`10.10.0.0/16`) транслируются в адрес Loopback1 пограничного роутера. Адреса Loopback1 взяты из блока AS1001 `100.1.0.0/24`, который Москва анонсирует по BGP, поэтому ответы возвращаются в Москву.

Inside-интерфейсы: линки к R12, R13 и к R19 (на R14) или R20 (на R15). Outside: линк к провайдеру.
```
ip access-list standard ACL-NAT-MOSCOW
 permit 10.10.0.0 0.0.255.255
!
interface Ethernet0/2
 ip nat outside
interface Ethernet0/0
 ip nat inside
...
ip nat inside source list ACL-NAT-MOSCOW interface Loopback1 overload
```

Наружу Москва выходит через Ламас ([лаба 07](../lab07/)), поэтому в обычном режиме трансляция идёт на R15. NAT на R14 работает при отказе R15: тогда default в OSPF анонсирует R14, и весь трафик идёт через него.

**Статический NAT для R20.** Адрес Loopback0 R20 (`10.10.255.20`) постоянно соответствует публичному `100.1.0.20`. В отличие от PAT, трансляция работает в обе стороны: на `100.1.0.20` можно инициировать соединение снаружи.
```
ip nat inside source static 10.10.255.20 100.1.0.20
```

**R19 для удалённого управления.** Снаружи доступен telnet на R19 по адресу `100.1.0.19`, порт 23. Это статический PAT: наружу открыт только один порт, а не весь адрес.
```
ip nat inside source static tcp 10.10.255.19 23 100.1.0.19 23 extendable
```
На R19 включён вход по telnet с локальной учётной записью:
```
username admin privilege 15 secret otus
!
line vty 0 4
 login local
 transport input telnet
```

Обе статические трансляции стоят на R15, потому что входящий трафик в Москву идёт через Ламас. R19 подключён к R14, но это не мешает: R15 транслирует адрес и отправляет пакет к R19 через внутреннюю сеть по OSPF, а ответ R19 по маршруту по умолчанию возвращается на R15 и там транслируется обратно.

### 3. NAT в С.-Петербурге

R18 транслирует внутренние сети `10.20.0.0/16` в пул из пяти адресов блока AS2042. С `overload` каждый адрес пула обслуживает много сессий за счёт портов.
```
ip access-list standard ACL-NAT-SPB
 permit 10.20.0.0 0.0.255.255
!
ip nat pool POOL-SPB 20.42.0.1 20.42.0.5 netmask 255.255.255.248
ip nat inside source list ACL-NAT-SPB pool POOL-SPB overload
```
Inside: линки к R16 и R17. Outside: оба линка в Триаду.

### 4. NAT в Чокурдахе

У R28 два линка в Триаду, и трафик между ними распределяет PBR из [лабы 02](../lab02/). Адреса линков принадлежат Триаде (`52.0.1.12/30` и `52.0.1.16/30`), и обратный трафик на каждый адрес приходит только через свой линк. Поэтому трансляция идёт в адрес того интерфейса, через который пакет уходит: это задаёт route-map с `match interface`.
```
ip access-list standard ACL-NAT-CHOKURDAH
 permit 10.30.0.0 0.0.255.255
!
route-map RM-NAT-R26 permit 10
 match ip address ACL-NAT-CHOKURDAH
 match interface Ethernet0/0
route-map RM-NAT-R25 permit 10
 match ip address ACL-NAT-CHOKURDAH
 match interface Ethernet0/1
!
ip nat inside source route-map RM-NAT-R26 interface Ethernet0/0 overload
ip nat inside source route-map RM-NAT-R25 interface Ethernet0/1 overload
```
При переключении PBR на резервный линк трансляция автоматически переходит на его адрес.

### 5. DHCP

DHCP-серверы стоят на обоих роутерах, R12 и R13, чтобы раздача адресов работала при отказе одного из них. Чтобы они не выдали один адрес разным клиентам, диапазоны разделены:

| Роутер | VLAN 10 | VLAN 70 |
|---|---|---|
| R12 | 10.10.10.100 - 149 | 10.10.70.100 - 149 |
| R13 | 10.10.10.150 - 199 | 10.10.70.150 - 199 |

Остальные адреса исключены через `ip dhcp excluded-address`. Шлюзом выдаётся виртуальный адрес HSRP (`.1`), а не адрес конкретного роутера.

R12:
```
ip dhcp excluded-address 10.10.10.1 10.10.10.99
ip dhcp excluded-address 10.10.10.150 10.10.10.254
ip dhcp excluded-address 10.10.70.1 10.10.70.99
ip dhcp excluded-address 10.10.70.150 10.10.70.254
ip dhcp pool VLAN10
 network 10.10.10.0 255.255.255.0
 default-router 10.10.10.1
 lease 1
ip dhcp pool VLAN70
 network 10.10.70.0 255.255.255.0
 default-router 10.10.70.1
 lease 1
```
На VPC1 и VPC7 статические адреса заменены на `ip dhcp`.

### 6. NTP

R12 и R13 работают NTP-серверами на собственных часах (`ntp master 3`) и связаны друг с другом как peer: если один станет недоступен, второй продолжит раздавать время. Остальные устройства Москвы (R14, R15, R19, R20, SW2-SW5) указывают оба сервера, R12 предпочтительный.

Все запросы идут от Loopback0 у роутеров и от SVI Vlan99 у коммутаторов. На всех устройствах задан часовой пояс MSK.

R12:
```
clock timezone MSK 3 0
ntp source Loopback0
ntp master 3
ntp peer 10.10.255.13
```
Клиенты:
```
clock timezone MSK 3 0
ntp source Loopback0
ntp server 10.10.255.12 prefer
ntp server 10.10.255.13
```

### 7. Особенности стенда

**Маршрут по умолчанию на коммутаторах.** Коммутаторы IOL L2 работают с включённой IP-маршрутизацией, и команда `ip default-gateway` из [лабы 01](../lab01/) на них не действует. Это выяснилось при настройке NTP: коммутаторы не могли достучаться до Loopback R12. В этой работе на все коммутаторы добавлен `ip route 0.0.0.0 0.0.0.0 <шлюз VLAN 99>`.

**DHCP и первый старт роутеров.** Роутер IOL без конфигурации в NVRAM при старте пытается получить адреса по DHCP (AutoInstall). Если R12 уже раздаёт адреса, R13 может получить на сабинтерфейсах VLAN 10 и 70 адреса из пула вместо своих `.3`. После этого HSRP показывает соседа с неверным адресом. Лечится повторным вводом `ip address` на этих сабинтерфейсах, а чтобы проблема не повторялась при следующих запусках, конфигурацию стоит сохранить в NVRAM (`write memory`).

### 8. Проверка

#### NAT в Москве

VPC1 пингует С.-Петербург, R15 транслирует его адрес в свой Loopback1:
```
VPC1> ping 20.42.0.18 -c 3
84 bytes from 20.42.0.18 icmp_seq=1 ttl=251 time=1.696 ms
84 bytes from 20.42.0.18 icmp_seq=2 ttl=251 time=3.573 ms
84 bytes from 20.42.0.18 icmp_seq=3 ttl=251 time=1.952 ms

R15#show ip nat translations
Pro Inside global      Inside local       Outside local      Outside global
icmp 100.1.0.15:63612  10.10.10.150:63612 20.42.0.18:63612   20.42.0.18:63612
icmp 100.1.0.15:63868  10.10.10.150:63868 20.42.0.18:63868   20.42.0.18:63868
icmp 100.1.0.15:64124  10.10.10.150:64124 20.42.0.18:64124   20.42.0.18:64124
tcp 100.1.0.19:23      10.10.255.19:23    ---                ---
--- 100.1.0.20         10.10.255.20       ---                ---
```

#### Статический NAT для R20

R20 пингует С.-Петербург со своего Loopback0, адрес транслируется в `100.1.0.20`:
```
R20#ping 20.42.0.18 source Loopback0
Packet sent with a source address of 10.10.255.20
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/1 ms

R15#show ip nat translations
Pro Inside global      Inside local       Outside local      Outside global
tcp 100.1.0.19:23      10.10.255.19:23    20.42.0.18:15690   20.42.0.18:15690
tcp 100.1.0.19:23      10.10.255.19:23    ---                ---
icmp 100.1.0.20:0      10.10.255.20:0     20.42.0.18:0       20.42.0.18:0
--- 100.1.0.20         10.10.255.20       ---                ---
```

#### R19 доступен для управления

С R18 (другая AS) подключаемся по telnet к `100.1.0.19` и попадаем на R19. В `show users` видно, что сессия пришла с адреса R18 `20.42.0.18`:
```
R18#telnet 100.1.0.19 /source-interface Loopback1
Trying 100.1.0.19 ... Open

User Access Verification

Username: admin
Password:

R19#show users
    Line       User       Host(s)              Idle       Location
   0 con 0                idle                 00:04:17
*  2 vty 0     admin      idle                 00:00:00 20.42.0.18

R19#exit

[Connection to 100.1.0.19 closed by foreign host]
```

#### NAT в С.-Петербурге

VPC8 (`10.20.80.108`) транслируется в адрес из пула `20.42.0.2`:
```
VPC8> ping 100.1.0.15 -c 3
84 bytes from 100.1.0.15 icmp_seq=1 ttl=251 time=1.947 ms
84 bytes from 100.1.0.15 icmp_seq=2 ttl=251 time=2.043 ms
84 bytes from 100.1.0.15 icmp_seq=3 ttl=251 time=1.914 ms

R18#show ip nat translations
Pro Inside global      Inside local       Outside local      Outside global
icmp 20.42.0.2:381     10.20.80.108:381   100.1.0.15:381     100.1.0.15:381
icmp 20.42.0.2:637     10.20.80.108:637   100.1.0.15:637     100.1.0.15:637
icmp 20.42.0.2:893     10.20.80.108:893   100.1.0.15:893     100.1.0.15:893
```

#### NAT в Чокурдахе

VPC30 (VLAN 30) по PBR уходит через R26, и трансляция идёт в адрес линка к R26 `52.0.1.14`:
```
VPC30> ping 20.42.0.18 -c 3
84 bytes from 20.42.0.18 icmp_seq=1 ttl=253 time=0.929 ms
84 bytes from 20.42.0.18 icmp_seq=2 ttl=253 time=1.175 ms
84 bytes from 20.42.0.18 icmp_seq=3 ttl=253 time=1.075 ms

R28#show ip nat translations
Pro Inside global      Inside local       Outside local      Outside global
icmp 52.0.1.14:2685    10.30.30.130:2685  20.42.0.18:2685    20.42.0.18:2685
icmp 52.0.1.14:2941    10.30.30.130:2941  20.42.0.18:2941    20.42.0.18:2941
icmp 52.0.1.14:3197    10.30.30.130:3197  20.42.0.18:3197    20.42.0.18:3197
```

VPC31 (VLAN 31) по PBR уходит через R25 и тоже имеет связность с Москвой:
```
VPC31> ping 100.1.0.15 -c 3
84 bytes from 100.1.0.15 icmp_seq=1 ttl=250 time=1.748 ms
84 bytes from 100.1.0.15 icmp_seq=2 ttl=250 time=1.613 ms
84 bytes from 100.1.0.15 icmp_seq=3 ttl=250 time=1.537 ms
```

#### DHCP

VPC1 и VPC7 получили адреса по DHCP, шлюзом выдан виртуальный адрес HSRP:
```
VPC1> ip dhcp
DORA IP 10.10.10.102/24 GW 10.10.10.1

VPC1> show ip
NAME        : VPC1[1]
IP/MASK     : 10.10.10.102/24
GATEWAY     : 10.10.10.1
DHCP SERVER : 10.10.10.2
DHCP LEASE  : 86400, 86400/43200/75600

VPC7> ip dhcp
DORA IP 10.10.70.102/24 GW 10.10.70.1
```

Оба сервера выдают адреса каждый из своего диапазона (клиент принимает первое пришедшее предложение):
```
R12#show ip dhcp binding
IP address          Client-ID/              Lease expiration        Type
10.10.10.102        0100.5079.6668.01       Oct 15 2026 04:05 PM    Automatic
10.10.70.102        0100.5079.6668.07       Oct 15 2026 04:06 PM    Automatic

R13#show ip dhcp binding
IP address          Client-ID/              Lease expiration        Type
10.10.10.150        0100.5079.6668.01       Oct 15 2026 04:05 PM    Automatic
10.10.70.151        0100.5079.6668.07       Oct 15 2026 04:06 PM    Automatic
```

#### NTP

R12 синхронизирован со своими часами (`127.127.1.1`) и видит R13 как peer:
```
R12#show ntp associations
  address         ref clock       st   when   poll reach  delay  offset   disp
*~127.127.1.1     .LOCL.           2      6     16   377  0.000   0.000  1.204
 ~10.10.255.13    127.127.1.1      3     15     64    17  0.000   0.000  3.768
```

Клиенты синхронизированы с R12:
```
R14#show ntp status
Clock is synchronized, stratum 4, reference is 10.10.255.12

R19#show ntp status
Clock is synchronized, stratum 4, reference is 10.10.255.12

SW4#show ntp associations
  address         ref clock       st   when   poll reach  delay  offset   disp
*~10.10.255.12    127.127.1.1      3     52     64     1  1.000   0.500 7938.4
 ~10.10.255.13    .INIT.          16      -    512     0  0.000   0.000 15937.

R20#show clock
15:58:21.878 MSK Wed Oct 14 2026
```

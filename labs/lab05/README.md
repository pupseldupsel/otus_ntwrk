# EIGRP

### Задание

1. В офисе С.-Петербург настроить EIGRP.
2. R32 получает только маршрут по умолчанию.
3. R16-17 анонсируют только суммарные префиксы.
4. Использовать EIGRP named-mode для настройки сети.
5. Настройка осуществляется одновременно для IPv4 и IPv6.

### Решение

1. [Схема и топология EIGRP](#1-схема-и-топология-eigrp)
2. [Маршрут по умолчанию](#2-маршрут-по-умолчанию)
3. [Суммирование на R16 и R17](#3-суммирование-на-r16-и-r17)
4. [Только default для R32](#4-только-default-для-r32)
5. [Примеры настройки](#5-примеры-настройки)
6. [Проверка](#6-проверка)
7. [Конфигурационные файлы](configs/)

Работа продолжает [лабу 04](../lab04/), изменения затронули только роутеры С.-Петербурга (R16, R17, R18, R32). Файл стенда для EVE-NG: [lab05.unl](lab05.unl).

### 1. Схема и топология EIGRP

![](scheme.png)

На всех роутерах один процесс named-mode `router eigrp SPB` с двумя address-family, IPv4 и IPv6, номер AS 2042 (по номеру AS офиса). Router ID равен адресу Loopback0.

| Соседство | Через | Назначение |
|---|---|---|
| R16 - R18 | Ethernet0/1 - Ethernet0/0, 10.20.0.0/30 | выход к провайдеру |
| R17 - R18 | Ethernet0/1 - Ethernet0/1, 10.20.0.4/30 | выход к провайдеру |
| R16 - R32 | Ethernet0/3 - Ethernet0/0, 10.20.0.8/30 | подключение R32 |
| R16 - R17 | VLAN 99 (Ethernet0/0.99) | обмен маршрутами внутри офиса |

Правило для интерфейсов такое: `af-interface default` делает все интерфейсы passive, а соседства явно включаются только там, где они нужны. Пользовательские VLAN 80 и 90 и Loopback остаются passive: их сети анонсируются, но соседства через них не строятся.

Соседство R16-R17 через VLAN 99 нужно, потому что R32 подключён только к R16. Без него R17 не знал бы сетей R32, а R18, получив суммарный префикс от обоих, мог бы отправить трафик к R32 через R17.

Линки R18 к Триаде в EIGRP не участвуют. Для IPv4 их исключает команда `network 10.20.0.0 0.0.255.255`. В IPv6 команды `network` нет, все интерфейсы с IPv6 включаются автоматически, поэтому линки к Триаде отключены через `af-interface ... shutdown`.

### 2. Маршрут по умолчанию

R18 стоит на границе с Триадой. На нём статические маршруты по умолчанию на R24 и R26, и только они передаются в EIGRP через `redistribute static` с route-map (в EIGRP попадают `0.0.0.0/0` и `::/0`, остальная статика нет). Для внешних маршрутов задан `default-metric`. Остальные роутеры получают default как `D*EX` (IPv4) и `EX` (IPv6).

### 3. Суммирование на R16 и R17

R16 и R17 отдают R18 только суммарные префиксы офиса: `10.20.0.0/16` и `2001:db8:2042::/48`. Это сделано командой `summary-address` на интерфейсах в сторону R18. Суммирование работает только в эту сторону, между собой R16 и R17 обмениваются подробными маршрутами.

На R16 и R17 при суммировании автоматически появляется маршрут суммарного префикса на `Null0`. Он защищает от петель: пакет в несуществующую подсеть офиса отбрасывается на R16/R17, а не уходит по маршруту по умолчанию обратно к R18.

### 4. Только default для R32

R32 должен получать только маршрут по умолчанию. Для этого на R16 стоит фильтр на исходящие обновления в сторону R32 (`distribute-list ... out Ethernet0/3`), который пропускает только `0.0.0.0/0` и `::/0`.

Синтаксис фильтра для IPv4 и IPv6 отличается: в IPv4 это `distribute-list prefix ИМЯ`, а в IPv6 `distribute-list prefix-list ИМЯ`.

Суммирование `summary-address 0.0.0.0 0.0.0.0` в сторону R32 не подходит: R16 поставил бы себе маршрут `0.0.0.0/0` на `Null0` с AD 5. Он перекрыл бы настоящий default от R18, и R16 потерял бы выход наружу.

Дополнительно R32 настроен как `eigrp stub connected`: он анонсирует только свои подключённые сети и не используется как транзит. При пропаже маршрутов соседи не отправляют ему запросы (query), и это ограничивает область пересчёта DUAL.

### 5. Примеры настройки

R16 (суммирование к R18, фильтр к R32):
```
ip prefix-list PL-DEFAULT-ONLY seq 5 permit 0.0.0.0/0
ipv6 prefix-list PL6-DEFAULT-ONLY seq 5 permit ::/0
!
router eigrp SPB
 !
 address-family ipv4 unicast autonomous-system 2042
  !
  af-interface default
   passive-interface
  exit-af-interface
  !
  af-interface Ethernet0/1
   no passive-interface
   summary-address 10.20.0.0 255.255.0.0
  exit-af-interface
  !
  af-interface Ethernet0/3
   no passive-interface
  exit-af-interface
  !
  af-interface Ethernet0/0.99
   no passive-interface
  exit-af-interface
  network 10.20.0.0 0.0.255.255
  !
  topology base
   distribute-list prefix PL-DEFAULT-ONLY out Ethernet0/3
  exit-af-topology
  eigrp router-id 10.20.255.16
 exit-address-family
 !
 address-family ipv6 unicast autonomous-system 2042
  !
  af-interface default
   passive-interface
  exit-af-interface
  !
  af-interface Ethernet0/1
   no passive-interface
   summary-address 2001:db8:2042::/48
  exit-af-interface
  !
  af-interface Ethernet0/3
   no passive-interface
  exit-af-interface
  !
  af-interface Ethernet0/0.99
   no passive-interface
  exit-af-interface
  !
  topology base
   distribute-list prefix-list PL6-DEFAULT-ONLY out Ethernet0/3
  exit-af-topology
  eigrp router-id 10.20.255.16
 exit-address-family
```

R18 (default в EIGRP):
```
ip route 0.0.0.0 0.0.0.0 52.0.1.1
ip route 0.0.0.0 0.0.0.0 52.0.1.5
ipv6 route ::/0 2001:db8:520:1824::24
ipv6 route ::/0 2001:db8:520:1826::26
!
ip prefix-list PL-DEFAULT-ONLY seq 5 permit 0.0.0.0/0
ipv6 prefix-list PL6-DEFAULT-ONLY seq 5 permit ::/0
route-map RM-DEFAULT permit 10
 match ip address prefix-list PL-DEFAULT-ONLY
route-map RM6-DEFAULT permit 10
 match ipv6 address prefix-list PL6-DEFAULT-ONLY
!
router eigrp SPB
 address-family ipv4 unicast autonomous-system 2042
  ...
  topology base
   default-metric 10000 100 255 1 1500
   redistribute static route-map RM-DEFAULT
  exit-af-topology
 exit-address-family
 address-family ipv6 unicast autonomous-system 2042
  ...
  af-interface Ethernet0/2
   shutdown
  exit-af-interface
  af-interface Ethernet0/3
   shutdown
  exit-af-interface
  topology base
   default-metric 10000 100 255 1 1500
   redistribute static route-map RM6-DEFAULT
  exit-af-topology
 exit-address-family
```

R32 (stub):
```
router eigrp SPB
 address-family ipv4 unicast autonomous-system 2042
  eigrp stub connected
  ...
 address-family ipv6 unicast autonomous-system 2042
  eigrp stub connected
  ...
```

### 6. Проверка

#### R18: только суммарные префиксы

У R18 два соседа, R16 и R17:
```
R18#show eigrp address-family ipv4 neighbors
EIGRP-IPv4 VR(SPB) Address-Family Neighbors for AS(2042)
H   Address                 Interface              Hold Uptime   SRTT   RTO  Q  Seq
                                                   (sec)         (ms)       Cnt Num
1   10.20.0.5               Et0/1                    11 00:01:14    5   100  0  5
0   10.20.0.1               Et0/0                    12 00:39:15 1281  5000  0  3
```

Из всех сетей офиса R18 видит только суммарный префикс, сразу через R16 и R17:
```
R18#show ip route eigrp
Gateway of last resort is 52.0.1.5 to network 0.0.0.0

      10.0.0.0/8 is variably subnetted, 6 subnets, 3 masks
D        10.20.0.0/16 [90/1024640] via 10.20.0.5, 00:01:22, Ethernet0/1
                      [90/1024640] via 10.20.0.1, 00:01:22, Ethernet0/0

R18#show ipv6 route eigrp
D   2001:DB8:2042::/48 [90/1024640]
     via FE80::17, Ethernet0/1
     via FE80::16, Ethernet0/0
```

#### R16: default от R18 и суммарный маршрут на Null0

```
R16#show ip route eigrp
Gateway of last resort is 10.20.0.2 to network 0.0.0.0

D*EX  0.0.0.0/0 [170/1536000] via 10.20.0.2, 00:36:31, Ethernet0/1
      10.0.0.0/8 is variably subnetted, 15 subnets, 4 masks
D        10.20.0.0/16 is a summary, 00:36:35, Null0
...
D        10.20.255.32/32 [90/1024640] via 10.20.0.10, 00:35:15, Ethernet0/3
```

#### R32: только default

```
R32#show ip route eigrp
Gateway of last resort is 10.20.0.9 to network 0.0.0.0

D*EX  0.0.0.0/0 [170/2048000] via 10.20.0.9, 00:42:10, Ethernet0/0

R32#show ipv6 route eigrp
EX  ::/0 [170/2048000]
     via FE80::16, Ethernet0/0
```

По маршруту по умолчанию R32 достаёт до любой сети офиса, например до Loopback R17:
```
R32#ping 10.20.255.17 source Loopback0
Packet sent with a source address of 10.20.255.32
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/2 ms
```

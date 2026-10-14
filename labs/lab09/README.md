# BGP для IPv6 unicast

### Задание

Настроить BGP для IPv6 unicast на всём стенде: те же eBGP- и iBGP-сессии и те же политики, что в работах по BGP для IPv4 ([лаба 06](../lab06/), [лаба 07](../lab07/), [лаба 08](../lab08/)).

### Решение

1. [Сессии и анонсы](#1-сессии-и-анонсы)
2. [Политики](#2-политики)
3. [Особенности IPv6](#3-особенности-ipv6)
4. [Примеры настройки](#4-примеры-настройки)
5. [Проверка](#5-проверка)
6. [Конфигурационные файлы](configs/)

Работа продолжает [лабу 08](../lab08/), настройки BGP для IPv4 не менялись. Файл стенда для EVE-NG: [lab09.unl](lab09.unl).

![](scheme.png)

### 1. Сессии и анонсы

| Сессия | Адреса | Тип |
|---|---|---|
| R14 - R22 | 2001:db8:101:1422::14 - ::22 | eBGP 1001 - 101 |
| R15 - R21 | 2001:db8:301:1521::15 - ::21 | eBGP 1001 - 301 |
| R22 - R21 | 2001:db8:101:2122::22 - ::21 | eBGP 101 - 301 |
| R21 - R24 | 2001:db8:301:2124::21 - ::24 | eBGP 301 - 520 |
| R18 - R24, R18 - R26 | 2001:db8:520:1824::/64, 2001:db8:520:1826::/64 | eBGP 2042 - 520 |
| R14 - R15 | 2001:db8:1001:ffff::14 - ::15 (Loopback0) | iBGP 1001 |
| R23 (RR) - R24, R25, R26 | 2001:db8:520:ffff::23 - ::24, ::25, ::26 (Loopback0) | iBGP 520 |

eBGP-сессии строятся по глобальным IPv6-адресам линков, iBGP по IPv6-адресам Loopback0. Адреса Loopback0 известны через OSPFv3 в Москве и IS-IS (IPv6 multi-topology) в Триаде.

| AS | Префикс |
|---|---|
| 1001 Москва | 2001:db8:1001::/48 |
| 2042 С.-Петербург | 2001:db8:2042::/48 |
| 101 Киторн | 2001:db8:101::/48 |
| 301 Ламас | 2001:db8:301::/48 |
| 520 Триада | 2001:db8:520::/48 |

Как и для IPv4, каждый блок анонсируется целиком командой `network`, а в таблице маршрутизации его гарантирует статический маршрут на `Null0` с AD 254.

### 2. Политики

Все политики из работ по IPv4 повторены для IPv6:

| Политика | Где | Как |
|---|---|---|
| Ламас основной, исходящий трафик | R15 | `route-map RM-LAMAS-IN in`, local-preference 150 |
| Ламас основной, входящий трафик | R14 | `route-map RM-KITORN-OUT out`, prepend `1001 1001` |
| Москва не транзитная | R14, R15 | `filter-list 1 out` (AS-path `^$`) |
| С.-Петербург не транзитный | R18 | `prefix-list PL6-SPB-OUT out` |
| Киторн отдаёт Москве только default | R22 | `default-originate` + `prefix-list PL6-TO-MOSCOW out` |
| Ламас отдаёт Москве default и С.-Петербург | R21 | `default-originate` + `prefix-list PL6-TO-MOSCOW out` |
| Route reflector в Триаде | R23 | `route-reflector-client` для R24, R25, R26 |
| next-hop-self на пограничных роутерах | R14, R15, R24, R26 | `next-hop-self` для iBGP-соседей |
| Балансировка в С.-Петербурге | R18 | `maximum-paths 2` |

Route-map и AS-path access-list из IPv4 подходят и для IPv6 без изменений, потому что они работают с атрибутами BGP, а не с адресами. Prefix-list для IPv6 отдельные (`ipv6 prefix-list`).

Статические IPv6-маршруты по умолчанию на R14 и R15 удалены, как раньше для IPv4: default приходит от провайдеров по BGP. В OSPFv3 R14 анонсирует default с метрикой 50, поэтому внутренний трафик Москвы уходит через R15 и Ламас.

### 3. Особенности IPv6

- **Активация соседей.** По умолчанию IOS активирует каждого нового соседа в address-family IPv4 (`bgp default ipv4-unicast`), даже если адрес соседа IPv6. Поэтому IPv6-соседи явно выключены в IPv4 (`no neighbor ... activate`) и включены только в address-family IPv6.
- **Next-hop.** В eBGP через IPv6 соседу передаются глобальный и link-local адрес next-hop. В таблице маршрутизации ставится link-local адрес соседа (`FE80::24`), а в таблице BGP виден глобальный.
- **Префикс С.-Петербурга** анонсируется командой `network 2001:db8:2042::/48`. В таблице маршрутизации R18 этот префикс есть дважды: суммарный маршрут EIGRP от R16/R17 (AD 90) и статика на `Null0` (AD 254). Выигрывает EIGRP, поэтому у префикса в BGP next-hop `FE80::17` и метрика EIGRP: команда `network` берёт атрибуты из маршрута, установленного в таблицу.

### 4. Примеры настройки

R14 (Москва):
```
ipv6 route 2001:db8:1001::/48 Null0 254
!
ipv6 router ospf 1
 default-information originate metric 50
!
router bgp 1001
 neighbor 2001:db8:101:1422::22 remote-as 101
 neighbor 2001:db8:101:1422::22 description R22 Kitorn
 neighbor 2001:db8:1001:ffff::15 remote-as 1001
 neighbor 2001:db8:1001:ffff::15 description R15 iBGP
 neighbor 2001:db8:1001:ffff::15 update-source Loopback0
 !
 address-family ipv4
  no neighbor 2001:db8:101:1422::22 activate
  no neighbor 2001:db8:1001:ffff::15 activate
 exit-address-family
 !
 address-family ipv6
  network 2001:db8:1001::/48
  neighbor 2001:db8:101:1422::22 activate
  neighbor 2001:db8:101:1422::22 filter-list 1 out
  neighbor 2001:db8:101:1422::22 route-map RM-KITORN-OUT out
  neighbor 2001:db8:1001:ffff::15 activate
  neighbor 2001:db8:1001:ffff::15 next-hop-self
 exit-address-family
```

R21 (Ламас, фильтр в Москву):
```
ipv6 prefix-list PL6-TO-MOSCOW seq 5 permit ::/0
ipv6 prefix-list PL6-TO-MOSCOW seq 10 permit 2001:db8:2042::/48
!
router bgp 301
 address-family ipv6
  network 2001:db8:301::/48
  neighbor 2001:db8:301:1521::15 activate
  neighbor 2001:db8:301:1521::15 default-originate
  neighbor 2001:db8:301:1521::15 prefix-list PL6-TO-MOSCOW out
  neighbor 2001:db8:101:2122::22 activate
  neighbor 2001:db8:301:2124::24 activate
 exit-address-family
```

R23 (route reflector):
```
router bgp 520
 neighbor 2001:db8:520:ffff::24 remote-as 520
 neighbor 2001:db8:520:ffff::24 update-source Loopback0
 ...
 address-family ipv6
  neighbor 2001:db8:520:ffff::24 activate
  neighbor 2001:db8:520:ffff::24 route-reflector-client
  neighbor 2001:db8:520:ffff::25 activate
  neighbor 2001:db8:520:ffff::25 route-reflector-client
  neighbor 2001:db8:520:ffff::26 activate
  neighbor 2001:db8:520:ffff::26 route-reflector-client
 exit-address-family
```

### 5. Проверка

#### Сессии

У R21 три eBGP-соседа по IPv6, у R23 три клиента:
```
R21#show bgp ipv6 unicast summary
Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
2001:DB8:101:2122::22
                4          101       6       7        6    0    0 00:01:05        2
2001:DB8:301:1521::15
                4         1001       6       5        6    0    0 00:01:06        1
2001:DB8:301:2124::24
                4          520       9       7        6    0    0 00:01:05        2

R23#show bgp ipv6 unicast summary
Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
2001:DB8:520:FFFF::24
                4          520       9       9        6    0    0 00:01:17        5
2001:DB8:520:FFFF::25
                4          520       4       9        6    0    0 00:01:11        0
2001:DB8:520:FFFF::26
                4          520       6       9        6    0    0 00:01:09        2
```

#### Москва: фильтры провайдеров и приоритет Ламаса

Ламас отдаёт R15 только default и префикс С.-Петербурга:
```
R21#show bgp ipv6 unicast neighbors 2001:db8:301:1521::15 advertised-routes

Originating default network ::/0

     Network          Next Hop            Metric LocPrf Weight Path
 *>  2001:DB8:2042::/48
                       2001:DB8:301:2124::24
                                                              0 520 2042 i

Total number of prefixes 1
```

У R15 маршруты от Ламаса с `LocPrf 150`:
```
R15#show bgp ipv6 unicast
     Network          Next Hop            Metric LocPrf Weight Path
 *>  ::/0             2001:DB8:301:1521::21
                                                     150      0 301 i
 * i 2001:DB8:1001::/48
                       2001:DB8:1001:FFFF::14
                                                0    100      0 i
 *>                   ::                       0         32768 i
 *>  2001:DB8:2042::/48
                       2001:DB8:301:1521::21
                                                     150      0 301 520 2042 i
```

R14 получает default от Киторна и от R15 и выбирает R15. Оба помечены `r`: в таблицу маршрутизации поставлен default из OSPFv3 от R15, как и для IPv4 в [лабе 08](../lab08/):
```
R14#show bgp ipv6 unicast
     Network          Next Hop            Metric LocPrf Weight Path
 r>i ::/0             2001:DB8:1001:FFFF::15
                                                0    150      0 301 i
 r                    2001:DB8:101:1422::22
                                                              0 101 i
 * i 2001:DB8:1001::/48
                       2001:DB8:1001:FFFF::15
                                                0    100      0 i
 *>                   ::                       0         32768 i

R12#show ipv6 route ::/0
Routing entry for ::/0
  Known via "ospf 1", distance 110, metric 1
  Tag 1, type extern 2
  Route count is 1/1, share count 0
  Routing paths:
    FE80::15, Ethernet0/3
      Last updated 00:00:47 ago
```

#### Москва и С.-Петербург не транзитные

Киторн получает от Москвы только её блок, с prepend:
```
R22#show bgp ipv6 unicast neighbors 2001:db8:101:1422::14 routes
     Network          Next Hop            Metric LocPrf Weight Path
 *   2001:DB8:1001::/48
                       2001:DB8:101:1422::14
                                                0             0 1001 1001 1001 i

Total number of prefixes 1
```

Триада получает от С.-Петербурга только его блок:
```
R24#show bgp ipv6 unicast neighbors 2001:db8:520:1824::18 routes
     Network          Next Hop            Metric LocPrf Weight Path
 *>  2001:DB8:2042::/48
                       2001:DB8:520:1824::18
                                          1024640             0 2042 i

Total number of prefixes 1
```

#### Триада: route reflector

R26 получает внешние префиксы через RR с next-hop R24:
```
R26#show bgp ipv6 unicast
     Network          Next Hop            Metric LocPrf Weight Path
 *>i 2001:DB8:101::/48
                       2001:DB8:520:FFFF::24
                                                0    100      0 301 101 i
 *>i 2001:DB8:301::/48
                       2001:DB8:520:FFFF::24
                                                0    100      0 301 i
 * i 2001:DB8:520::/48
                       2001:DB8:520:FFFF::24
                                                0    100      0 i
 *>                   ::                       0         32768 i
 *>i 2001:DB8:1001::/48
                       2001:DB8:520:FFFF::24
                                                0    100      0 301 1001 i
 * i 2001:DB8:2042::/48
                       2001:DB8:520:FFFF::24
                                          1024640    100      0 2042 i
 *>                   2001:DB8:520:1826::18
                                          1024640             0 2042 i
```

#### С.-Петербург: два линка

Каждый внешний префикс R18 получает от R24 и R26, и оба пути стоят в таблице маршрутизации:
```
R18#show bgp ipv6 unicast
     Network          Next Hop            Metric LocPrf Weight Path
 *m  2001:DB8:101::/48
                       2001:DB8:520:1826::26
                                                              0 520 301 101 i
 *>                   2001:DB8:520:1824::24
                                                              0 520 301 101 i
 *m  2001:DB8:301::/48
                       2001:DB8:520:1826::26
                                                              0 520 301 i
 *>                   2001:DB8:520:1824::24
                                                              0 520 301 i
 *m  2001:DB8:520::/48
                       2001:DB8:520:1826::26
                                                0             0 520 i
 *>                   2001:DB8:520:1824::24
                                                0             0 520 i
 *m  2001:DB8:1001::/48
                       2001:DB8:520:1826::26
                                                              0 520 301 1001 i
 *>                   2001:DB8:520:1824::24
                                                              0 520 301 1001 i
 *>  2001:DB8:2042::/48
                       FE80::17           1024640         32768 i

R18#show ipv6 route 2001:db8:1001::/48
Routing entry for 2001:DB8:1001::/48
  Known via "bgp 2042", distance 20, metric 0, type external
  Route count is 2/2, share count 0
  Routing paths:
    FE80::24, Ethernet0/2
      MPLS label: nolabel
      Last updated 00:02:29 ago
    FE80::26, Ethernet0/3
      MPLS label: nolabel
      Last updated 00:01:55 ago
```

#### Связность

Трассировка из Москвы в С.-Петербург идёт через R15 и Ламас:
```
R14#traceroute 2001:db8:2042:ffff::18
Tracing the route to 2001:DB8:2042:FFFF::18

  1 2001:DB8:1001:1214::12 18 msec
    2001:DB8:1001:1314::13 0 msec
    2001:DB8:1001:1214::12 0 msec
  2 2001:DB8:1001:1315::15 1 msec
    2001:DB8:1001:1215::15 0 msec
    2001:DB8:1001:1315::15 0 msec
  3 2001:DB8:301:1521::21 [AS 301] 0 msec 1 msec 1 msec
  4 2001:DB8:301:2124::24 [AS 301] 1 msec 1 msec 1 msec
  5 2001:DB8:520:1824::18 [AS 301] 1 msec 1 msec 1 msec
```

Из С.-Петербурга доступны оба пограничных роутера Москвы:
```
R18#ping 2001:db8:1001:ffff::14 source Loopback0
Packet sent with a source address of 2001:DB8:2042:FFFF::18
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/1 ms

R18#ping 2001:db8:1001:ffff::15 source Loopback0
Packet sent with a source address of 2001:DB8:2042:FFFF::18
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/1 ms
```

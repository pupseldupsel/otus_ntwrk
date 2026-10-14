# BGP. Фильтрация

### Задание

1. Настроить фильтрацию в офисе Москва так, чтобы не появилось транзитного трафика (AS-path).
2. Настроить фильтрацию в офисе С.-Петербург так, чтобы не появилось транзитного трафика (prefix-list).
3. Настроить провайдера Киторн так, чтобы в офис Москва отдавался только маршрут по умолчанию.
4. Настроить провайдера Ламас так, чтобы в офис Москва отдавался только маршрут по умолчанию и префикс офиса С.-Петербург.
5. Все сети в лабораторной работе должны иметь IP-связность.

### Решение

1. [Схема](#1-схема)
2. [Москва: без транзита через AS-path](#2-москва-без-транзита-через-as-path)
3. [С.-Петербург: без транзита через prefix-list](#3-с-петербург-без-транзита-через-prefix-list)
4. [Киторн и Ламас: что отдаётся в Москву](#4-киторн-и-ламас-что-отдаётся-в-москву)
5. [Маршрут по умолчанию в Москве](#5-маршрут-по-умолчанию-в-москве)
6. [Проверка](#6-проверка)
7. [Конфигурационные файлы](configs/)

Работа продолжает [лабу 07](../lab07/): iBGP, route reflector, приоритет Ламаса и балансировка в С.-Петербурге сохранены. Файл стенда для EVE-NG: [lab08.unl](lab08.unl).

### 1. Схема

![](scheme.png)

| Где | Направление | Фильтр | Что проходит |
|---|---|---|---|
| R14, R15 (Москва) | к провайдерам | `filter-list` (AS-path `^$`) | только префиксы самой AS 1001 |
| R18 (С.-Петербург) | к Триаде | `prefix-list` | только `20.42.0.0/24` |
| R22 (Киторн) | в Москву | `default-originate` + `prefix-list` | только `0.0.0.0/0` |
| R21 (Ламас) | в Москву | `default-originate` + `prefix-list` | `0.0.0.0/0` и `20.42.0.0/24` |

### 2. Москва: без транзита через AS-path

Москва подключена к двум провайдерам. Без фильтрации она анонсировала бы одному провайдеру маршруты, полученные от другого, и могла бы стать транзитом между ними.

Фильтр по AS-path пропускает наружу только маршруты с пустым AS-path (`^$`). Пустой путь бывает только у маршрутов, которые родились внутри своей AS: свои сети и маршруты, полученные по iBGP от R14/R15. Всё, что пришло от провайдеров, содержит номер их AS и отбрасывается.

```
ip as-path access-list 1 permit ^$
!
router bgp 1001
 address-family ipv4
  neighbor 101.0.0.1 filter-list 1 out
```

Фильтр проверяет исходный AS-path до того, как исходящая route-map из [лабы 07](../lab07/) добавит prepend `1001 1001`, поэтому они не мешают друг другу.

### 3. С.-Петербург: без транзита через prefix-list

У R18 два линка к одному провайдеру. Транзит здесь возможен между R24 и R26: маршрут, полученный от одного роутера Триады, без фильтра ушёл бы другому. Prefix-list разрешает анонсировать только свой блок:

```
ip prefix-list PL-SPB-OUT seq 5 permit 20.42.0.0/24
!
router bgp 2042
 address-family ipv4
  neighbor 52.0.1.1 prefix-list PL-SPB-OUT out
  neighbor 52.0.1.5 prefix-list PL-SPB-OUT out
```

В отличие от AS-path фильтра, prefix-list придётся менять при каждом новом префиксе офиса. Зато правило проще проверить: наружу уходит ровно то, что перечислено.

### 4. Киторн и Ламас: что отдаётся в Москву

У провайдеров нет маршрута по умолчанию в таблице, поэтому он генерируется командой `neighbor ... default-originate`: она анонсирует соседу `0.0.0.0/0` независимо от таблицы маршрутизации. Остальные префиксы отсекает исходящий prefix-list.

Киторн (R22):
```
ip prefix-list PL-TO-MOSCOW seq 5 permit 0.0.0.0/0
!
router bgp 101
 address-family ipv4
  neighbor 101.0.0.2 default-originate
  neighbor 101.0.0.2 prefix-list PL-TO-MOSCOW out
```

Ламас (R21):
```
ip prefix-list PL-TO-MOSCOW seq 5 permit 0.0.0.0/0
ip prefix-list PL-TO-MOSCOW seq 10 permit 20.42.0.0/24
!
router bgp 301
 address-family ipv4
  neighbor 30.1.0.2 default-originate
  neighbor 30.1.0.2 prefix-list PL-TO-MOSCOW out
```

### 5. Маршрут по умолчанию в Москве

Раньше на R14 и R15 стояли статические маршруты по умолчанию к провайдерам. Теперь default приходит от провайдеров по BGP, и статика удалена: иначе с AD 1 она всегда побеждала бы BGP и приоритет Ламаса для default не работал бы.

Что получается:
- **R15** получает default от Ламаса по eBGP (`local-preference 150`), ставит его в таблицу и анонсирует в OSPF.
- **R14** получает default от Киторна (eBGP) и от R15 (iBGP, `LocPrf 150`). BGP выбирает путь через R15. Но в таблицу маршрутизации R14 ставит default из OSPF (AD 110), а не из iBGP (AD 200), поэтому в BGP он помечен `r` (RIB-failure). На результат это не влияет: оба default ведут к R15 и дальше в Ламас.
- **При отказе R15** OSPF-default от R15 пропадёт, R14 поставит в таблицу default от Киторна (eBGP, AD 20) и сам начнёт анонсировать его в OSPF (метрика 50, из [лабы 07](../lab07/)).

### 6. Проверка

#### Москва получает только разрешённое

Киторн отдаёт R14 только default, отдельных префиксов нет (`Total number of prefixes 0`, а default отображается строкой `Originating default network`):
```
R22#show ip bgp neighbors 101.0.0.2 advertised-routes

Originating default network 0.0.0.0

     Network          Next Hop            Metric LocPrf Weight Path

Total number of prefixes 0
```

Ламас отдаёт R15 default и префикс С.-Петербурга:
```
R21#show ip bgp neighbors 30.1.0.2 advertised-routes

Originating default network 0.0.0.0

     Network          Next Hop            Metric LocPrf Weight Path
 *>  20.42.0.0/24     30.1.0.6                               0 520 2042 i

Total number of prefixes 1
```

Таблица BGP на R15 и R14:
```
R15#show ip bgp
     Network          Next Hop            Metric LocPrf Weight Path
 *>  0.0.0.0          30.1.0.1                      150      0 301 i
 *>  20.42.0.0/24     30.1.0.1                      150      0 301 520 2042 i
 * i 100.1.0.0/24     10.10.255.14             0    100      0 i
 *>                   0.0.0.0                  0         32768 i

R14#show ip bgp
     Network          Next Hop            Metric LocPrf Weight Path
 r   0.0.0.0          101.0.0.1                              0 101 i
 r>i                  10.10.255.15             0    150      0 301 i
 *>i 20.42.0.0/24     10.10.255.15             0    150      0 301 520 2042 i
 * i 100.1.0.0/24     10.10.255.15             0    100      0 i
 *>                   0.0.0.0                  0         32768 i
```

Внутри Москвы default по-прежнему ведёт к R15:
```
R12#show ip route 0.0.0.0
Routing entry for 0.0.0.0/0, supernet
  Known via "ospf 1", distance 110, metric 1, candidate default path
  Tag 1, type extern 2, forward metric 10
  Last update from 10.10.0.14 on Ethernet0/3, 00:01:44 ago
  Routing Descriptor Blocks:
  * 10.10.0.14, from 10.10.255.15, 00:01:44 ago, via Ethernet0/3
      Route metric is 1, traffic share count is 1
      Route tag 1
```

#### Москва не транзитная

Провайдеры получают от Москвы только её собственный блок. Киторн (с prepend из лабы 07):
```
R22#show ip bgp neighbors 101.0.0.2 routes
     Network          Next Hop            Metric LocPrf Weight Path
 *   100.1.0.0/24     101.0.0.2                0             0 1001 1001 1001 i

Total number of prefixes 1
```

Ламас:
```
R21#show ip bgp neighbors 30.1.0.2 routes
     Network          Next Hop            Metric LocPrf Weight Path
 *>  100.1.0.0/24     30.1.0.2                 0             0 1001 i

Total number of prefixes 1
```

#### С.-Петербург не транзитный

R24 получает от R18 только блок С.-Петербурга, хотя у R18 в таблице есть маршруты всех AS:
```
R24#show ip bgp neighbors 52.0.1.2 routes
     Network          Next Hop            Metric LocPrf Weight Path
 *>  20.42.0.0/24     52.0.1.2                 0             0 2042 i

Total number of prefixes 1
```

У R26 путь к `20.42.0.0/24` только прямой от R18 или через R24 (`2042 i`), транзитных путей через С.-Петербург нет:
```
R26#show ip bgp
     Network          Next Hop            Metric LocPrf Weight Path
 * i 20.42.0.0/24     52.0.255.24              0    100      0 2042 i
 *>                   52.0.1.6                 0             0 2042 i
 *>i 30.1.0.0/16      52.0.255.24              0    100      0 301 i
 * i 52.0.0.0/16      52.0.255.24              0    100      0 i
 *>                   0.0.0.0                  0         32768 i
 *>i 100.1.0.0/24     52.0.255.24              0    100      0 301 1001 i
 *>i 101.0.0.0/16     52.0.255.24              0    100      0 301 101 i
```

#### Связность

Москва и С.-Петербург доступны друг другу. Трассировка с R14 идёт через R15 и Ламас:
```
R14#traceroute 20.42.0.18 source Loopback1 numeric
Tracing the route to 20.42.0.18
  1 10.10.0.1 [AS 301] 0 msec
    10.10.0.5 [AS 301] 1 msec
    10.10.0.1 [AS 301] 0 msec
  2 10.10.0.10 [AS 301] 1 msec
    10.10.0.14 [AS 301] 0 msec
    10.10.0.10 [AS 301] 1 msec
  3 30.1.0.1 [AS 301] 1 msec 0 msec 1 msec
  4 30.1.0.6 [AS 301] 1 msec 1 msec 1 msec
  5 52.0.1.2 [AS 301] 4 msec *  5 msec

R18#ping 100.1.0.14 source Loopback1
Packet sent with a source address of 20.42.0.18
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/1 ms

R18#ping 100.1.0.15 source Loopback1
Packet sent with a source address of 20.42.0.18
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/1 ms
```

Метка `[AS 301]` у всех хопов трассировки теперь означает только то, что R14 знает эти адреса через default от Ламаса: отдельных префиксов провайдеров у Москвы больше нет.

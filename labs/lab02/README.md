# PBR и IP SLA

### Задание

1. Настроить политику маршрутизации для сетей офиса Чокурдах.
2. Распределить трафик между двумя линками с провайдером.
3. Настроить отслеживание линка через технологию IP SLA.
4. Настроить для офиса Лабытнанги маршрут по умолчанию.

### Решение

1. [Схема и исходные данные](#1-схема-и-исходные-данные)
2. [Логика распределения трафика](#2-логика-распределения-трафика)
3. [Настройка R28 (Чокурдах)](#3-настройка-r28-чокурдах)
4. [Настройка R27 (Лабытнанги)](#4-настройка-r27-лабытнанги)
5. [Временные маршруты на стороне Триады](#5-временные-маршруты-на-стороне-триады)
6. [Проверка](#6-проверка)
7. [Конфигурационные файлы](configs/)

Работа продолжает [лабу 01](../lab01/): адресация, VLAN и L2 не менялись. Файл стенда для EVE-NG: [lab02.unl](lab02.unl).

### 1. Схема и исходные данные

![](scheme.png)

Чокурдах (R28) подключён к провайдеру Триада (AS520) двумя линками, Лабытнанги (R27) одним:

| Линк | Сеть | Триада | Офис |
|---|---|---|---|
| R26 e0/1 - R28 e0/0 | 52.0.1.12/30 | R26 `.13` | R28 `.14` |
| R25 e0/3 - R28 e0/1 | 52.0.1.16/30 | R25 `.17` | R28 `.18` |
| R25 e0/1 - R27 e0/0 | 52.0.1.8/30 | R25 `.9` | R27 `.10` |

Сети Чокурдаха: VLAN 30 `10.30.30.0/24` (VPC30), VLAN 31 `10.30.31.0/24` (VPC31), VLAN 99 `10.30.99.0/24` (управление). R28 работает как router-on-a-stick для SW29.

### 2. Логика распределения трафика

Трафик делится по VLAN, каждый линк основной для одного VLAN и резервный для другого:

| Источник | Основной next-hop | Резервный next-hop |
|---|---|---|
| VLAN 30 | R26 `52.0.1.13` (e0/0) | R25 `52.0.1.17` (e0/1) |
| VLAN 31 | R25 `52.0.1.17` (e0/1) | R26 `52.0.1.13` (e0/0) |
| Сам R28, VLAN 99 | маршрут по умолчанию через оба линка (ECMP) | |

Доступность каждого линка проверяет IP SLA: R28 каждые 5 секунд пингует адрес провайдера на линке. Результат передаётся в track, а track используется:
- в route-map PBR через `set ip next-hop verify-availability`: next-hop с упавшим track пропускается, и берётся следующий по порядковому номеру;
- в статических маршрутах по умолчанию: маршрут с упавшим track убирается из таблицы.

Задержка `delay down 10 up 10` защищает от переключений при единичных потерях пакетов.

Почему не пинг до адреса за провайдером: на стенде у провайдера ещё нет динамической маршрутизации, поэтому проверяется ближайший адрес на линке. Это ловит обрыв линка и отказ порта провайдера. Отказ дальше в сети провайдера так не ловится; когда в Триаде будет IS-IS, цель SLA можно сменить на Loopback соответствующего роутера.

### 3. Настройка R28 (Чокурдах)

IP SLA и track:
```
ip sla 1
 icmp-echo 52.0.1.13 source-interface Ethernet0/0
 frequency 5
ip sla schedule 1 life forever start-time now
ip sla 2
 icmp-echo 52.0.1.17 source-interface Ethernet0/1
 frequency 5
ip sla schedule 2 life forever start-time now
!
track 1 ip sla 1 reachability
 delay down 10 up 10
track 2 ip sla 2 reachability
 delay down 10 up 10
```

ACL и route-map:
```
ip access-list extended ACL-VLAN30
 permit ip 10.30.30.0 0.0.0.255 any
ip access-list extended ACL-VLAN31
 permit ip 10.30.31.0 0.0.0.255 any
!
route-map PBR-CHOKURDAH permit 10
 description VLAN30 -> R26, backup R25
 match ip address ACL-VLAN30
 set ip next-hop verify-availability 52.0.1.13 10 track 1
 set ip next-hop verify-availability 52.0.1.17 20 track 2
route-map PBR-CHOKURDAH permit 20
 description VLAN31 -> R25, backup R26
 match ip address ACL-VLAN31
 set ip next-hop verify-availability 52.0.1.17 10 track 2
 set ip next-hop verify-availability 52.0.1.13 20 track 1
```

PBR применяется на входе пользовательских сабинтерфейсов:
```
interface Ethernet0/2.30
 ip policy route-map PBR-CHOKURDAH
interface Ethernet0/2.31
 ip policy route-map PBR-CHOKURDAH
```

Маршруты по умолчанию для трафика, который не попал под PBR (сам роутер, VLAN 99), а также на случай, если оба next-hop из route-map недоступны:
```
ip route 0.0.0.0 0.0.0.0 52.0.1.13 track 1
ip route 0.0.0.0 0.0.0.0 52.0.1.17 track 2
ipv6 route ::/0 2001:db8:520:2628::26
ipv6 route ::/0 2001:db8:520:2528::25
```

PBR в этой работе делается для IPv4. Для IPv6 настроены два маршрута по умолчанию без отслеживания, они убираются из таблицы только при падении интерфейса.

### 4. Настройка R27 (Лабытнанги)

У офиса один линк, поэтому достаточно статического маршрута по умолчанию на R25:
```
ip route 0.0.0.0 0.0.0.0 52.0.1.9
ipv6 route ::/0 2001:db8:520:2527::25
```

### 5. Временные маршруты на стороне Триады

Динамической маршрутизации в Триаде пока нет (IS-IS будет в лабе 04), поэтому для обратного трафика на R25 и R26 добавлены статические маршруты до сетей офисов и до Loopback соседа. После настройки IS-IS и BGP они будут удалены.

R25:
```
ip route 10.30.0.0 255.255.0.0 52.0.1.18
ip route 10.40.0.0 255.255.0.0 52.0.1.10
ip route 52.0.255.26 255.255.255.255 52.0.0.14
ipv6 route 2001:db8:1001:2800::/56 2001:db8:520:2528::28
ipv6 route 2001:db8:1001:2700::/56 2001:db8:520:2527::27
```

R26:
```
ip route 10.30.0.0 255.255.0.0 52.0.1.14
ip route 52.0.255.25 255.255.255.255 52.0.0.13
ipv6 route 2001:db8:1001:2800::/56 2001:db8:520:2628::28
```

### 6. Проверка

#### Нормальная работа

Оба SLA отвечают, оба track в `Up`:
```
R28#show ip sla summary
ID           Type        Destination       Stats       Return      Last
*1           icmp-echo   52.0.1.13         RTT=1       OK          1 second ago
*2           icmp-echo   52.0.1.17         RTT=1       OK          1 second ago

R28#show track brief
Track Type        Instance                   Parameter        State Last Change
1     ip sla      1                          reachability     Up    00:04:15
2     ip sla      2                          reachability     Up    00:04:15
```

Все next-hop в route-map доступны, счётчики совпадений растут в обеих секциях:
```
R28#show route-map PBR-CHOKURDAH
route-map PBR-CHOKURDAH, permit, sequence 10
  Match clauses:
    ip address (access-lists): ACL-VLAN30
  Set clauses:
    ip next-hop verify-availability 52.0.1.13 10 track 1  [up]
    ip next-hop verify-availability 52.0.1.17 20 track 2  [up]
  Policy routing matches: 28 packets, 3044 bytes
route-map PBR-CHOKURDAH, permit, sequence 20
  Match clauses:
    ip address (access-lists): ACL-VLAN31
  Set clauses:
    ip next-hop verify-availability 52.0.1.17 10 track 2  [up]
    ip next-hop verify-availability 52.0.1.13 20 track 1  [up]
  Policy routing matches: 30 packets, 3264 bytes
```

Трассировка до Loopback R26 (`52.0.255.26`). VPC30 идёт напрямую через R26, VPC31 через R25 и далее к R26:
```
VPC30> trace 52.0.255.26
 1   10.30.30.1   0.474 ms  0.420 ms  0.235 ms
 2   *52.0.1.13   0.440 ms (ICMP type:3, code:3, Destination port unreachable)  *

VPC31> trace 52.0.255.26
 1   10.30.31.1   0.286 ms  0.363 ms  0.351 ms
 2   52.0.1.17   0.436 ms  0.310 ms  0.238 ms
 3   *52.0.0.14   0.379 ms (ICMP type:3, code:3, Destination port unreachable)  *
```

#### Отказ линка R28 - R26

Выключаем на R28 `Ethernet0/0`:
```
R28(config)#interface Ethernet0/0
R28(config-if)#shutdown
*Oct 14 02:26:36.603: %LINK-5-CHANGED: Interface Ethernet0/0, changed state to administratively down
*Oct 14 02:26:37.604: %LINEPROTO-5-UPDOWN: Line protocol on Interface Ethernet0/0, changed state to down

R28#show track brief
Track Type        Instance                   Parameter        State Last Change
1     ip sla      1                          reachability     Down  00:00:05
2     ip sla      2                          reachability     Up    00:08:25

R28#show route-map PBR-CHOKURDAH
route-map PBR-CHOKURDAH, permit, sequence 10
  Match clauses:
    ip address (access-lists): ACL-VLAN30
  Set clauses:
    ip next-hop verify-availability 52.0.1.13 10 track 1  [down]
    ip next-hop verify-availability 52.0.1.17 20 track 2  [up]
...
```

VPC30 переключился на резервный линк через R25:
```
VPC30> trace 52.0.255.25
 1   10.30.30.1   0.498 ms  0.413 ms  0.234 ms
 2   *52.0.1.17   0.468 ms (ICMP type:3, code:3, Destination port unreachable)  *
```

> Трассировка до `52.0.255.26` в этот момент доходит до `52.0.1.17` и дальше обрывается. Это не ошибка PBR, а обратный путь: в IOL выключение порта на R28 не роняет линк на стороне R26, и его статический маршрут `10.30.0.0/16 via 52.0.1.14` остаётся активным. Ответы уходят в мёртвый линк. В лабе 04 статику заменит IS-IS, и проблема уйдёт.

#### Восстановление

После `no shutdown` track 1 возвращается в `Up`, VPC30 снова идёт через R26:
```
R28#show track brief
Track Type        Instance                   Parameter        State Last Change
1     ip sla      1                          reachability     Up    00:00:01
2     ip sla      2                          reachability     Up    00:10:51

VPC30> trace 52.0.255.26
 1   10.30.30.1   0.689 ms  0.225 ms  0.309 ms
 2   *52.0.1.13   0.637 ms (ICMP type:3, code:3, Destination port unreachable)  *
```

#### Лабытнанги

<!-- TODO: R27 show ip route 0.0.0.0, ping 52.0.255.25 source Loopback0 -->

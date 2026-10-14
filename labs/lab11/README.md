# VPN. GRE. DMVPN

### Задание

1. Настроить GRE между офисами Москва и С.-Петербург.
2. Настроить DMVPN между офисами Москва и Чокурдах, Лабытнанги.
3. Все узлы в офисах в лабораторной работе должны иметь IP-связность.

### Решение

1. [Схема](#1-схема)
2. [GRE Москва - С.-Петербург](#2-gre-москва---с-петербург)
3. [DMVPN Москва - Чокурдах, Лабытнанги](#3-dmvpn-москва---чокурдах-лабытнанги)
4. [Маршрутизация через туннели](#4-маршрутизация-через-туннели)
5. [Проверка](#5-проверка)
6. [Конфигурационные файлы](configs/)

Работа продолжает [лабу 10](../lab10/). Файл стенда для EVE-NG: [lab11.unl](lab11.unl).

### 1. Схема

![](scheme.png)

| Туннель | Участники | Адреса туннеля | Адреса снаружи (NBMA) |
|---|---|---|---|
| GRE, Tunnel0 | R15 - R18 | 10.100.0.0/30 (.1, .2) | 100.1.0.15 - 20.42.0.18 |
| DMVPN, Tunnel1 | хаб R14, споки R27, R28 | 10.200.0.0/24 (.1, .27, .28) | 100.1.0.14; 52.0.1.10; 52.0.1.14 |

До этой работы офисы были связаны только через интернет и NAT: внутренние адреса одного офиса не были доступны из другого. Туннели связывают внутренние сети `10.0.0.0/8` напрямую.

### 2. GRE Москва - С.-Петербург

Туннель построен между Loopback1 пограничных роутеров. Их адреса принадлежат публичным блокам офисов, анонсированы по BGP и доступны друг другу ([лаба 06](../lab06/)). Москва выходит наружу через R15, поэтому московский конец туннеля на нём.

R15:
```
interface Tunnel0
 description GRE to R18 (SPb)
 ip address 10.100.0.1 255.255.255.252
 ip mtu 1400
 ip tcp adjust-mss 1360
 keepalive 10 3
 tunnel source Loopback1
 tunnel destination 20.42.0.18
```

- **`ip mtu 1400` и `ip tcp adjust-mss 1360`.** GRE добавляет к пакету 24 байта заголовков. Без уменьшения MTU пакеты размером 1500 байт фрагментировались бы, а `adjust-mss` заставляет TCP-соединения сразу договариваться о меньшем размере сегмента. Запас взят с учётом IPSec в следующей работе.
- **`keepalive`.** GRE-туннель не имеет собственного состояния, и без keepalive интерфейс оставался бы `up`, даже если второй конец недоступен.
- **NAT.** Внутренний трафик уходит в туннель, а не в outside-интерфейс, поэтому не транслируется. Внешний GRE-пакет идёт от адреса Loopback1, который не попадает в ACL для NAT.

### 3. DMVPN Москва - Чокурдах, Лабытнанги

DMVPN (Dynamic Multipoint VPN) это один multipoint GRE-интерфейс на каждом роутере плюс протокол NHRP, который сопоставляет адреса туннеля с внешними (NBMA) адресами. Хаб знает только свой адрес, споки при старте регистрируются на хабе. Новый спок добавляется без изменения конфигурации хаба.

Используется **DMVPN Phase 3**: трафик между споками сначала идёт через хаб, хаб отвечает споку NHRP redirect, и спок запрашивает прямой адрес соседа. После этого между споками поднимается прямой туннель, а в таблицу маршрутизации добавляется маршрут через него.

Хаб R14:
```
interface Tunnel1
 description DMVPN hub
 ip address 10.200.0.1 255.255.255.0
 ip mtu 1400
 ip tcp adjust-mss 1360
 ip nhrp authentication OTUS
 ip nhrp network-id 100
 ip nhrp map multicast dynamic
 ip nhrp redirect
 tunnel source Loopback1
 tunnel mode gre multipoint
 tunnel key 100
```

Спок R28 (R27 аналогично, адрес `.27`):
```
interface Tunnel1
 description DMVPN spoke
 ip address 10.200.0.28 255.255.255.0
 ip mtu 1400
 ip tcp adjust-mss 1360
 ip nhrp authentication OTUS
 ip nhrp network-id 100
 ip nhrp map 10.200.0.1 100.1.0.14
 ip nhrp map multicast 100.1.0.14
 ip nhrp nhs 10.200.0.1
 ip nhrp shortcut
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 100
```

- Хаб на R14, а GRE на R15: нагрузка от туннелей распределяется между двумя пограничными роутерами Москвы. Пакеты к `100.1.0.14` приходят через Ламас на R15 и по OSPF доходят до R14.
- Споки используют как источник туннеля адрес своего линка к Триаде: собственных публичных адресов у Чокурдаха и Лабытнанги нет.

### 4. Маршрутизация через туннели

Маршруты в туннели статические:

| Роутер | Маршрут | Через |
|---|---|---|
| R15 | 10.20.0.0/16 | GRE (10.100.0.2) |
| R18 | 10.0.0.0/8 | GRE (10.100.0.1) |
| R14 | 10.30.0.0/16, 10.40.0.0/16 | DMVPN (10.200.0.28, 10.200.0.27) |
| R27, R28 | 10.0.0.0/8 | DMVPN, хаб (10.200.0.1) |

Внутри офисов дополнительных маршрутов почти не понадобилось: внутренние роутеры Москвы и С.-Петербурга отправляют неизвестный трафик по маршруту по умолчанию на пограничный роутер, а там уже стоит маршрут в туннель. Единственное исключение: маршруты до Чокурдаха и Лабытнанги R14 передаёт в OSPF (`redistribute static` с route-map), потому что трафик из С.-Петербурга приходит на R15, и тот должен знать, что эти сети за R14.

Маршрут `10.0.0.0/8` на R18 и споках позволяет С.-Петербургу, Чокурдаху и Лабытнанги связаться друг с другом через Москву без отдельных туннелей между ними.

**PBR на R28.** Политика из [лабы 02](../lab02/) направляла весь трафик VLAN 30 и 31 к провайдеру, в том числе во внутренние сети, и обходила маршрут в туннель. В ACL политики добавлено исключение: трафик в `10.0.0.0/8` идёт по обычной таблице маршрутизации, то есть в DMVPN.
```
ip access-list extended ACL-VLAN30
 deny ip any 10.0.0.0 0.255.255.255
 permit ip 10.30.30.0 0.0.0.255 any
```

### 5. Проверка

#### GRE

Туннель поднят, у R18 маршрут во внутренние сети через туннель:
```
R15#show interfaces Tunnel0 | include line protocol|Internet address
Tunnel0 is up, line protocol is up
  Internet address is 10.100.0.1/30

R18#show ip route 10.0.0.0 255.0.0.0
Routing entry for 10.0.0.0/8
  Known via "static", distance 1, metric 0
  Routing Descriptor Blocks:
  * 10.100.0.1
      Route metric is 0, traffic share count is 1
```

VPC1 (Москва) доступен для VPC8 (С.-Петербург) по внутренним адресам. Трассировка идёт через R15 и туннель (`10.100.0.2` это туннельный адрес R18):
```
VPC1> ping 10.20.80.108 -c 3
84 bytes from 10.20.80.108 icmp_seq=1 ttl=60 time=5.421 ms
84 bytes from 10.20.80.108 icmp_seq=2 ttl=60 time=2.394 ms
84 bytes from 10.20.80.108 icmp_seq=3 ttl=60 time=3.261 ms

VPC1> trace 10.20.80.108 -P 1
trace to 10.20.80.108, 8 hops max (ICMP), press Ctrl+C to stop
 1   10.10.10.2   0.868 ms  0.674 ms  0.673 ms
 2   10.10.0.14   2.151 ms  0.976 ms  0.979 ms
 3   10.100.0.2   1.753 ms  1.734 ms  2.594 ms
 4   10.20.0.1   1.828 ms  1.701 ms  1.664 ms
 5   10.20.80.108   1.883 ms  1.876 ms  1.976 ms
```

#### DMVPN

Оба спока зарегистрированы на хабе (`D`, dynamic):
```
R14#show dmvpn
Interface: Tunnel1, IPv4 NHRP Details
Type:Hub, NHRP Peers:2,

 # Ent  Peer NBMA Addr Peer Tunnel Add State  UpDn Tm Attrb
 ----- --------------- --------------- ----- -------- -----
     1 52.0.1.10           10.200.0.27    UP 00:00:00     D
     1 52.0.1.14           10.200.0.28    UP 00:00:04     D
```

Маршруты до Чокурдаха: на R14 статический через туннель, на R15 из OSPF через R14:
```
R14#show ip route 10.30.0.0 255.255.0.0
Routing entry for 10.30.0.0/16
  Known via "static", distance 1, metric 0
  Redistributing via ospf 1
  Advertised by ospf 1 subnets route-map RM-STATIC-TO-OSPF
  Routing Descriptor Blocks:
  * 10.200.0.28

R15#show ip route 10.30.0.0 255.255.0.0
Routing entry for 10.30.0.0/16
  Known via "ospf 1", distance 110, metric 20, type extern 2, forward metric 20
  Routing Descriptor Blocks:
  * 10.10.0.13, from 10.10.255.14, 00:01:28 ago, via Ethernet0/1
    10.10.0.9, from 10.10.255.14, 00:01:18 ago, via Ethernet0/0
```

Связность Москва - Чокурдах и С.-Петербург - Чокурдах (через Москву: GRE, затем DMVPN):
```
VPC1> ping 10.30.30.130 -c 3
84 bytes from 10.30.30.130 icmp_seq=1 ttl=61 time=3.445 ms
84 bytes from 10.30.30.130 icmp_seq=2 ttl=61 time=2.740 ms
84 bytes from 10.30.30.130 icmp_seq=3 ttl=61 time=3.512 ms

VPC8> ping 10.30.31.131 -c 3
84 bytes from 10.30.31.131 icmp_seq=1 ttl=58 time=6.301 ms
84 bytes from 10.30.31.131 icmp_seq=2 ttl=58 time=3.886 ms
84 bytes from 10.30.31.131 icmp_seq=3 ttl=58 time=3.632 ms
```

#### Phase 3: прямой туннель между споками

VPC30 (Чокурдах) пингует Лабытнанги. Первый пакет идёт через хаб (TTL 253), после NHRP redirect споки строят прямой туннель, и остальные пакеты идут напрямую (TTL 254, на один хоп меньше):
```
VPC30> ping 10.40.255.27 -c 3
84 bytes from 10.40.255.27 icmp_seq=1 ttl=253 time=4.402 ms
84 bytes from 10.40.255.27 icmp_seq=2 ttl=254 time=1.314 ms
84 bytes from 10.40.255.27 icmp_seq=3 ttl=254 time=1.323 ms
```

На R28 появился прямой туннель к R27 (`DT1`: dynamic, создан по запросу) и маршрут до сети Лабытнанги, добавленный NHRP (флаг `rib`):
```
R28#show dmvpn
Interface: Tunnel1, IPv4 NHRP Details
Type:Spoke, NHRP Peers:2,

 # Ent  Peer NBMA Addr Peer Tunnel Add State  UpDn Tm Attrb
 ----- --------------- --------------- ----- -------- -----
     2 52.0.1.10           10.200.0.27    UP 00:00:07   DT1
                           10.200.0.27    UP 00:00:07   DT1
     1 100.1.0.14           10.200.0.1    UP 00:00:40     S

R28#show ip nhrp
10.30.30.0/24 via 10.200.0.28
   Tunnel1 created 00:00:07, expire 01:59:52
   Type: dynamic, Flags: router unique local
   NBMA address: 52.0.1.14
    (no-socket)
10.40.255.27/32 via 10.200.0.27
   Tunnel1 created 00:00:07, expire 01:59:52
   Type: dynamic, Flags: router rib
   NBMA address: 52.0.1.10
10.200.0.1/32 via 10.200.0.1
   Tunnel1 created 00:03:29, never expire
   Type: static, Flags: used
   NBMA address: 100.1.0.14
10.200.0.27/32 via 10.200.0.27
   Tunnel1 created 00:00:07, expire 01:59:52
   Type: dynamic, Flags: router used nhop rib
   NBMA address: 52.0.1.10
```

Обратный путь с R27 на R28 тоже идёт напрямую:
```
R27#ping 10.30.255.28 source Loopback0
Packet sent with a source address of 10.40.255.27
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/2 ms

R27#show ip nhrp
10.30.255.28/32 via 10.200.0.28
   Tunnel1 created 00:00:00, expire 01:59:59
   Type: dynamic, Flags: router rib
   NBMA address: 52.0.1.14
```

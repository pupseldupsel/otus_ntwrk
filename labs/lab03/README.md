# OSPF

### Задание

1. Маршрутизаторы R14-R15 находятся в зоне 0 (backbone).
2. Маршрутизаторы R12-R13 находятся в зоне 10. Дополнительно к маршрутам должны получать маршрут по умолчанию.
3. Маршрутизатор R19 находится в зоне 101 и получает только маршрут по умолчанию.
4. Маршрутизатор R20 находится в зоне 102 и получает все маршруты, кроме маршрутов до сетей зоны 101.
5. Настроить IPv6 со своей схемой адресации (по желанию).

### Решение

1. [Схема и разбиение на зоны](#1-схема-и-разбиение-на-зоны)
2. [Маршрут по умолчанию](#2-маршрут-по-умолчанию)
3. [Зона 101: totally stubby](#3-зона-101-totally-stubby)
4. [Зона 102: фильтрация зоны 101](#4-зона-102-фильтрация-зоны-101)
5. [Примеры настройки](#5-примеры-настройки)
6. [Проверка](#6-проверка)
7. [Конфигурационные файлы](configs/)

Работа продолжает [лабу 02](../lab02/): адресация, L2 и PBR не менялись. OSPFv2 и OSPFv3 настроены по одной схеме. Файл стенда для EVE-NG: [lab03.unl](lab03.unl).

### 1. Схема и разбиение на зоны

![](scheme.png)

| Зона | Тип | Маршрутизаторы | Интерфейсы |
|---|---|---|---|
| 0 | backbone | R14, R15, R12, R13 | Lo0 R14/R15, линки R12-R14, R13-R14, R12-R15, R13-R15 |
| 10 | обычная | R12, R13 | VLAN 10, 70, 99 и Lo0 R12/R13 |
| 101 | totally stubby | R14 (ABR), R19 | линк R14-R19, Lo0 R19 |
| 102 | обычная, с фильтром | R15 (ABR), R20 | линк R15-R20, Lo0 R20 |

Прямого линка между R14 и R15 на стенде нет, они связаны только через R12 и R13. Если бы линки R12/R13 к R14/R15 были в зоне 10, зона 0 состояла бы из двух несвязанных частей (R14 и R15 по отдельности), и межзонная маршрутизация бы сломалась. Поэтому линки к R14/R15 отнесены к зоне 0, а R12 и R13 стали ABR между зонами 0 и 10. В зоне 10 у них остаются пользовательские VLAN и Loopback.

Остальные решения:
- **Router ID** равен адресу Loopback0, одинаковый для OSPFv2 и OSPFv3.
- **Линки между роутерами** переведены в `point-to-point`: на p2p-сегменте выборы DR/BDR не нужны, соседство поднимается быстрее.
- **Пользовательские VLAN и Loopback** сделаны passive: в них анонсируются сети, но соседства через них не строятся. Поэтому R12 и R13 не являются соседями внутри зоны 10, и Loopback соседа они видят межзонным маршрутом через зону 0.
- **OSPF включается на интерфейсе** (`ip ospf 1 area X`, `ipv6 ospf 1 area X`), без `network`: так сразу видно, какой интерфейс в какой зоне.

### 2. Маршрут по умолчанию

R14 и R15 подключены к провайдерам (Киторн и Ламас). На них стоят статические маршруты по умолчанию в сторону провайдеров, и оба анонсируют их в OSPF через `default-information originate` как внешний маршрут `E2`. Ключевое слово `always` не используется: если линк к провайдеру упадёт и статический маршрут пропадёт, роутер перестанет анонсировать default, и трафик уйдёт через второго провайдера.

R12 и R13 получают default от обоих ABR и балансируют трафик между ними (ECMP). Выбор основного провайдера будет сделан в лабе по iBGP.

### 3. Зона 101: totally stubby

R19 должен получать только маршрут по умолчанию. Для этого зона 101 сделана totally stubby:
- на ABR R14 `area 101 stub no-summary`: в зону не передаются ни внешние маршруты (Type 5), ни межзонные (Type 3), вместо них ABR генерирует одну Type 3 LSA с маршрутом `0.0.0.0/0`;
- на R19 `area 101 stub`: флаг stub должен совпадать у всех роутеров зоны, иначе соседство не поднимется.

### 4. Зона 102: фильтрация зоны 101

R20 должен получать все маршруты, кроме сетей зоны 101 (`10.10.0.16/30` и `10.10.255.19/32`).

**IPv4.** На ABR R15 Type 3 LSA с сетями зоны 101 не пропускаются в зону 102 командой `area 102 filter-list prefix PL-NO-AREA101 in`. Сети зоны 101 при этом остаются в зоне 0 и в зоне 10, фильтр действует только на вход в зону 102. Маршрут по умолчанию (`E2`) приходит в зону 102 как обычно.

**IPv6.** Для OSPFv3 в IOS 15.4 команды `area filter-list` нет. Поэтому сети зоны 101 отфильтрованы на самом R20 через `distribute-list prefix-list ... in`: LSA в базу R20 попадают, но маршруты из них не устанавливаются в таблицу маршрутизации. Результат для R20 тот же.

### 5. Примеры настройки

R14 (ABR зоны 0 и 101, ASBR):
```
router ospf 1
 router-id 10.10.255.14
 passive-interface Loopback0
 area 101 stub no-summary
 default-information originate
!
ipv6 router ospf 1
 router-id 10.10.255.14
 passive-interface Loopback0
 area 101 stub no-summary
 default-information originate
!
interface Ethernet0/0
 ip ospf 1 area 0
 ipv6 ospf 1 area 0
 ip ospf network point-to-point
 ipv6 ospf network point-to-point
!
interface Ethernet0/3
 ip ospf 1 area 101
 ipv6 ospf 1 area 101
 ip ospf network point-to-point
 ipv6 ospf network point-to-point
!
ip route 0.0.0.0 0.0.0.0 101.0.0.1
ipv6 route ::/0 2001:db8:101:1422::22
```

R15 (ABR зоны 0 и 102, ASBR), фильтр сетей зоны 101:
```
ip prefix-list PL-NO-AREA101 seq 5 deny 10.10.0.16/30
ip prefix-list PL-NO-AREA101 seq 10 deny 10.10.255.19/32
ip prefix-list PL-NO-AREA101 seq 100 permit 0.0.0.0/0 le 32
!
router ospf 1
 router-id 10.10.255.15
 passive-interface Loopback0
 area 102 filter-list prefix PL-NO-AREA101 in
 default-information originate
```

R12 (ABR зоны 0 и 10):
```
router ospf 1
 router-id 10.10.255.12
 passive-interface Loopback0
 passive-interface Ethernet0/0.10
 passive-interface Ethernet0/0.99
 passive-interface Ethernet0/1.70
!
interface Ethernet0/2
 ip ospf 1 area 0
 ip ospf network point-to-point
!
interface Ethernet0/0.10
 ip ospf 1 area 10
```

R19:
```
router ospf 1
 router-id 10.10.255.19
 passive-interface Loopback0
 area 101 stub
```

R20, фильтр IPv6:
```
ipv6 prefix-list PL6-NO-AREA101 seq 5 deny 2001:db8:1001:1419::/64
ipv6 prefix-list PL6-NO-AREA101 seq 10 deny 2001:db8:1001:ffff::19/128
ipv6 prefix-list PL6-NO-AREA101 seq 100 permit ::/0 le 128
!
ipv6 router ospf 1
 router-id 10.10.255.20
 passive-interface Loopback0
 distribute-list prefix-list PL6-NO-AREA101 in
```

### 6. Проверка

#### Соседства

У R14 три соседа в состоянии `FULL`, без DR/BDR (линки point-to-point):
```
R14#show ip ospf neighbor

Neighbor ID     Pri   State           Dead Time   Address         Interface
10.10.255.13      0   FULL/  -        00:00:38    10.10.0.5       Ethernet0/1
10.10.255.12      0   FULL/  -        00:00:37    10.10.0.1       Ethernet0/0
10.10.255.19      0   FULL/  -        00:00:34    10.10.0.18      Ethernet0/3

R14#show ipv6 ospf neighbor

Neighbor ID     Pri   State           Dead Time   Interface ID    Interface
10.10.255.13      0   FULL/  -        00:00:32    6               Ethernet0/1
10.10.255.12      0   FULL/  -        00:00:33    5               Ethernet0/0
10.10.255.19      0   FULL/  -        00:00:31    3               Ethernet0/3
```

#### R12 (зона 10): все маршруты и default

Маршрут по умолчанию приходит от R14 и R15 (ECMP), видны сети зон 0, 101 и 102:
```
R12#show ip route ospf
Gateway of last resort is 10.10.0.14 to network 0.0.0.0

O*E2  0.0.0.0/0 [110/1] via 10.10.0.14, 00:01:08, Ethernet0/3
                [110/1] via 10.10.0.2, 00:01:08, Ethernet0/2
      10.0.0.0/8 is variably subnetted, 19 subnets, 3 masks
O        10.10.0.4/30 [110/20] via 10.10.0.2, 00:01:08, Ethernet0/2
O        10.10.0.8/30 [110/20] via 10.10.0.14, 00:01:08, Ethernet0/3
O IA     10.10.0.16/30 [110/20] via 10.10.0.2, 00:01:08, Ethernet0/2
O IA     10.10.0.20/30 [110/20] via 10.10.0.14, 00:01:08, Ethernet0/3
O IA     10.10.255.13/32 [110/21] via 10.10.0.14, 00:01:08, Ethernet0/3
                         [110/21] via 10.10.0.2, 00:01:08, Ethernet0/2
O        10.10.255.14/32 [110/11] via 10.10.0.2, 00:01:08, Ethernet0/2
O        10.10.255.15/32 [110/11] via 10.10.0.14, 00:01:08, Ethernet0/3
O IA     10.10.255.19/32 [110/21] via 10.10.0.2, 00:01:07, Ethernet0/2
```

#### R19 (зона 101): только default

```
R19#show ip route ospf
Gateway of last resort is 10.10.0.17 to network 0.0.0.0

O*IA  0.0.0.0/0 [110/11] via 10.10.0.17, 00:03:08, Ethernet0/0

R19#show ipv6 route ospf
OI  ::/0 [110/11]
     via FE80::14, Ethernet0/0
```

#### R20 (зона 102): всё, кроме зоны 101

Нет `10.10.0.16/30` и `10.10.255.19/32`, остальные сети и default на месте:
```
R20#show ip route ospf
Gateway of last resort is 10.10.0.21 to network 0.0.0.0

O*E2  0.0.0.0/0 [110/1] via 10.10.0.21, 00:00:11, Ethernet0/0
      10.0.0.0/8 is variably subnetted, 14 subnets, 3 masks
O IA     10.10.0.0/30 [110/30] via 10.10.0.21, 00:00:11, Ethernet0/0
O IA     10.10.0.4/30 [110/30] via 10.10.0.21, 00:00:11, Ethernet0/0
O IA     10.10.0.8/30 [110/20] via 10.10.0.21, 00:00:11, Ethernet0/0
O IA     10.10.0.12/30 [110/20] via 10.10.0.21, 00:00:11, Ethernet0/0
O IA     10.10.10.0/24 [110/30] via 10.10.0.21, 00:00:11, Ethernet0/0
O IA     10.10.70.0/24 [110/30] via 10.10.0.21, 00:00:11, Ethernet0/0
O IA     10.10.99.0/24 [110/30] via 10.10.0.21, 00:00:11, Ethernet0/0
O IA     10.10.255.12/32 [110/21] via 10.10.0.21, 00:00:11, Ethernet0/0
O IA     10.10.255.13/32 [110/21] via 10.10.0.21, 00:00:11, Ethernet0/0
O IA     10.10.255.14/32 [110/31] via 10.10.0.21, 00:00:11, Ethernet0/0
O IA     10.10.255.15/32 [110/11] via 10.10.0.21, 00:00:11, Ethernet0/0
```

В IPv6 то же: нет `2001:db8:1001:1419::/64` и `2001:db8:1001:ffff::19/128`:
```
R20#show ipv6 route ospf
OE2 ::/0 [110/1], tag 1
     via FE80::15, Ethernet0/0
OI  2001:DB8:1001:10::/64 [110/30]
     via FE80::15, Ethernet0/0
OI  2001:DB8:1001:70::/64 [110/30]
     via FE80::15, Ethernet0/0
OI  2001:DB8:1001:99::/64 [110/30]
     via FE80::15, Ethernet0/0
OI  2001:DB8:1001:1214::/64 [110/30]
     via FE80::15, Ethernet0/0
OI  2001:DB8:1001:1215::/64 [110/20]
     via FE80::15, Ethernet0/0
OI  2001:DB8:1001:1314::/64 [110/30]
     via FE80::15, Ethernet0/0
OI  2001:DB8:1001:1315::/64 [110/20]
     via FE80::15, Ethernet0/0
OI  2001:DB8:1001:FFFF::12/128 [110/20]
     via FE80::15, Ethernet0/0
OI  2001:DB8:1001:FFFF::13/128 [110/20]
     via FE80::15, Ethernet0/0
OI  2001:DB8:1001:FFFF::14/128 [110/30]
     via FE80::15, Ethernet0/0
OI  2001:DB8:1001:FFFF::15/128 [110/10]
     via FE80::15, Ethernet0/0
```

#### Работа фильтра на R15

LSA с Loopback R19 есть в базе R15 только в зоне 0. В зоне 102 её нет, значит фильтр отсекает её на границе зоны:
```
R15#show ip ospf database summary 10.10.255.19

            OSPF Router with ID (10.10.255.15) (Process ID 1)

                Summary Net Link States (Area 0)

  Routing Bit Set on this LSA in topology Base with MTID 0
  LS age: 345
  Options: (No TOS-capability, DC, Upward)
  LS Type: Summary Links(Network)
  Link State ID: 10.10.255.19 (summary Network Number)
  Advertising Router: 10.10.255.14
  LS Seq Number: 80000001
  Checksum: 0x3FA9
  Length: 28
  Network Mask: /32
        MTID: 0         Metric: 11
```

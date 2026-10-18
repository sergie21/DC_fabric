# Домашнее задание №8. VXLAN EVPN. Routing (Route-Type 5)

## Топология сети

За основу взял фабрику из [ДЗ №7](../HW7_EVPN_Multihoming). Underlay и Overlay не перестраивал: 2 Spine, 3 Leaf, 4 сервера. Сохранил ESI-LAG между Leaf1 и Leaf2 для server1. Добавил внешний маршрутизатор `border-fw` на FRRouting, подключённый двумя L3-линками к Leaf3.

- **Underlay:** OSPF area 0, point-to-point `/31`, Loopback0 — Router ID, Loopback1 — VTEP.
- **Overlay:** iBGP EVPN, AS 65100, Spine1/Spine2 — Route Reflector.
- **TENANT_A:** L3VNI 50001, VLAN 10/20.
- **TENANT_B:** L3VNI 50002, VLAN 30/40.
- **Border:** FRR, AS 65000, два eBGP-соседства с Leaf3, по одному в каждой VRF.

Схема подключений:

```text
                        Spine1 (RR)       Spine2 (RR)
                          AS 65100          AS 65100
                         /   |   \         /   |   \
                        /    |    \       /    |    \
                     Leaf1  Leaf2         Leaf3
                      | \   / |             |  \
                      |  \ /  |             |   \----- Et5 (TENANT_A) --- eth1 ---+
                      |   X   |             |                                FRR  | AS 65000
                      |  / \  |             |   /----- Et6 (TENANT_B) --- eth2 ---+
                      | /   \ |             |
                   server1  server2      server3  server4
                   bond0    VLAN20       VLAN30   VLAN40
                   VLAN10

     server1: Leaf1 Et3 + Leaf2 Et4 (ESI-LAG / LACP)
     server2: Leaf2 Et3
     server3: Leaf3 Et3
     server4: Leaf3 Et4
     FRR:     Leaf3 Et5 -> eth1; Leaf3 Et6 -> eth2
```

> Для GitHub сюда можно добавить изображение топологии из Containerlab Desktop. Схема выше показывает логические подключения; на каждом Leaf есть физический линк к каждому Spine.

## Задание

1. Разместить двух клиентов в разных VRF в одной фабрике.
2. Настроить маршрутизацию между клиентами через внешнее устройство (маршрутизатор/фаервол).
3. Реализовать передачу суммарных префиксов через EVPN Route-Type 5.
4. Зафиксировать план, адресное пространство, схему и конфигурации оборудования.

## IP-адресация

| Сервер | Leaf / порт | VRF | VLAN | L2VNI | IP-адрес | Шлюз |
| --- | --- | --- | --- | --- | --- | --- |
| server1 | Leaf1 Et3 + Leaf2 Et4 | TENANT_A | 10 | 10010 | 10.10.10.11/24 (`bond0`) | 10.10.10.254 |
| server2 | Leaf2 Et3 | TENANT_A | 20 | 10020 | 20.20.20.12/24 | 20.20.20.254 |
| server3 | Leaf3 Et3 | TENANT_B | 30 | 10030 | 30.30.30.13/24 | 30.30.30.254 |
| server4 | Leaf3 Et4 | TENANT_B | 40 | 10040 | 30.30.31.14/24 | 30.30.31.254 |

| VRF | L3VNI | Route Target | Подсети |
| --- | --- | --- | --- |
| TENANT_A | 50001 | 50001:50001 | 10.10.10.0/24, 20.20.20.0/24 |
| TENANT_B | 50002 | 50002:50002 | 30.30.30.0/24, 30.30.31.0/24 |

Anycast MAC — `0000.0000.9999`.

Подключение FRR к Leaf3:

| Устройство | Интерфейс | VRF на Leaf3 | IP-адрес | eBGP AS |
| --- | --- | --- | --- | --- |
| Leaf3 | Ethernet5 | TENANT_A | 172.16.10.1/30 | 65100 |
| FRR | eth1 | — | 172.16.10.2/30 | 65000 |
| Leaf3 | Ethernet6 | TENANT_B | 172.16.20.1/30 | 65100 |
| FRR | eth2 | — | 172.16.20.2/30 | 65000 |

FRR работает с одной глобальной таблицей маршрутизации Linux. На Leaf3 интерфейсы находятся в разных VRF. Между VRF прямого route leaking не настраивал — трафик проходит через FRR.

## Конфигурация Underlay/Overlay

1. Существующую фабрику сохранил: OSPF area 0 для Underlay, iBGP EVPN для Overlay, оба Spine — RR, AS 65100.
2. На Leaf1 и Leaf2 сохранил `TENANT_A` и ESI-LAG для server1 из ДЗ №7.
3. На Leaf3 создал `TENANT_B`, VLAN30/40, L2VNI 10030/10040, L3VNI 50002 и Anycast Gateway для обеих подсетей.
4. Подключил FRR к Leaf3 двумя независимыми L3-линками и поднял eBGP в каждой VRF.
5. FRR передаёт default route `0.0.0.0/0` в обе VRF. Leaf3 распространяет его по EVPN как Route-Type 5.
6. На Leaf3 в `TENANT_B` настроил агрегацию двух `/24` в `30.30.30.0/23` с `summary-only`.

### Leaf3 — VRF, VLAN, VXLAN

Основные изменения относительно ДЗ №7:

```text
vlan 30
   name TENANT_B_VLAN30
!
vlan 40
   name TENANT_B_VLAN40
!
vrf instance TENANT_B
!
ip routing vrf TENANT_B
!
interface Ethernet3
   description TO_SERVER3
   switchport access vlan 30
!
interface Ethernet4
   description TO_SERVER4
   switchport access vlan 40
!
interface Vlan30
   description GATEWAY_VLAN30
   vrf TENANT_B
   ip address virtual 30.30.30.254/24
!
interface Vlan40
   description GATEWAY_VLAN40
   vrf TENANT_B
   ip address virtual 30.30.31.254/24
!
interface Vxlan1
   vxlan vlan 30 vni 10030
   vxlan vlan 40 vni 10040
   vxlan vrf TENANT_B vni 50002
```

VLAN10/20 и L3VNI50001 для `TENANT_A` на Leaf3 оставил. Клиентов VLAN10/20 на Leaf3 теперь нет — server3/server4 переведены в `TENANT_B`.

### Leaf3 — подключение FRR и BGP

```text
interface Ethernet5
   description TO_BORDER_FW_TENANT_A
   no switchport
   vrf TENANT_A
   ip address 172.16.10.1/30
!
interface Ethernet6
   description TO_BORDER_FW_TENANT_B
   no switchport
   vrf TENANT_B
   ip address 172.16.20.1/30
!
router bgp 65100
   vlan 30
      rd auto
      route-target both 10030:10030
      redistribute learned
   !
   vlan 40
      rd auto
      route-target both 10040:10040
      redistribute learned
   !
   vrf TENANT_A
      rd 10.0.103.0:50001
      route-target import evpn 50001:50001
      route-target export evpn 50001:50001
      redistribute connected
      neighbor 172.16.10.2 remote-as 65000
      neighbor 172.16.10.2 description BORDER_FW_TENANT_A
      address-family ipv4
         neighbor 172.16.10.2 activate
   !
   vrf TENANT_B
      rd 10.0.103.0:50002
      route-target import evpn 50002:50002
      route-target export evpn 50002:50002
      redistribute connected
      aggregate-address 30.30.30.0/23 summary-only
      neighbor 172.16.20.2 remote-as 65000
      neighbor 172.16.20.2 description BORDER_FW_TENANT_B
      address-family ipv4
         neighbor 172.16.20.2 activate
```

Для `TENANT_A` и `TENANT_B` используются разные RT. Поэтому Leaf1 видит маршруты обеих VRF в общей EVPN BGP-таблице, но импортирует в `TENANT_A` только маршруты с соответствующим RT.

### Border-FW — FRR, AS 65000

FRR работает в Linux-контейнере `quay.io/frrouting/frr:10.4.1`. В `fabric.clab.yml` добавил команды настройки интерфейсов и IP forwarding:

```yaml
    border-fw:
      kind: linux
      image: quay.io/frrouting/frr:10.4.1
      mgmt-ipv4: 172.20.20.11
      binds:
        - configs/border-fw:/etc/frr
      exec:
        - ip addr add 172.16.10.2/30 dev eth1
        - ip addr add 172.16.20.2/30 dev eth2
        - sysctl -w net.ipv4.ip_forward=1
```

На FRR настроил политики:

```text
ip prefix-list DEFAULT_ONLY seq 10 permit 0.0.0.0/0
!
route-map EXPORT_DEFAULT permit 10
 match ip address prefix-list DEFAULT_ONLY
!
route-map IMPORT_FROM_LEAF permit 10
!
ip route 0.0.0.0/0 Null0
!
router bgp 65000
 bgp router-id 10.255.255.1
 neighbor 172.16.10.1 remote-as 65100
 neighbor 172.16.20.1 remote-as 65100
 !
 address-family ipv4 unicast
  network 0.0.0.0/0
  neighbor 172.16.10.1 route-map IMPORT_FROM_LEAF in
  neighbor 172.16.10.1 route-map EXPORT_DEFAULT out
  neighbor 172.16.20.1 route-map IMPORT_FROM_LEAF in
  neighbor 172.16.20.1 route-map EXPORT_DEFAULT out
```

- `IMPORT_FROM_LEAF` — разрешает приём маршрутов от Leaf3. Пока без фильтрации конкретных префиксов.
- `DEFAULT_ONLY` — prefix-list, совпадающий только с `0.0.0.0/0`.
- `EXPORT_DEFAULT` — разрешает анонс FRR в сторону Leaf3 только default route.
- `ip route 0.0.0.0/0 Null0` — создаёт маршрут для BGP `network 0.0.0.0/0`. Это лабораторная заглушка, **не реальный выход в интернет**.

FRR получает от Leaf3 четыре пользовательские подсети `/24`. Агрегация `/23` настроена отдельно в BGP VRF `TENANT_B` на Leaf3 для EVPN RT-5, а не как замена всех маршрутов на FRR.

### Server3 и Server4

Перевёл оба сервера из `TENANT_A` в `TENANT_B`. Адреса и default route прописаны в `fabric.clab.yml`:

```yaml
    server3:
      kind: linux
      image: alpine:latest
      mgmt-ipv4: 172.20.20.9
      exec:
        - ip addr add 30.30.30.13/24 dev eth1
        - ip link set eth1 up
        - ip route replace default via 30.30.30.254 dev eth1

    server4:
      kind: linux
      image: alpine:latest
      mgmt-ipv4: 172.20.20.10
      exec:
        - ip addr add 30.30.31.14/24 dev eth1
        - ip link set eth1 up
        - ip route replace default via 30.30.31.254 dev eth1
```

Для server1 сохранил `bond0` (LACP 802.3ad) и установил default route через `10.10.10.254`, чтобы меж-VRF трафик не уходил через management `eth0`. Для server2 — default route через `20.20.20.254`.

## Проверка

### Underlay/Overlay

На Leaf3 два OSPF-соседа в FULL, BGP EVPN до Spine1 и Spine2 в Established:

```text
leaf3#show ip ospf neighbor
10.0.2.0  FULL  Ethernet2
10.0.1.0  FULL  Ethernet1

leaf3#show bgp evpn summary
SPINE1  10.0.1.0  Estab
SPINE2  10.0.2.0  Estab
```

### eBGP и маршруты FRR

На FRR оба соседа с Leaf3 поднялись. Первоначально в `show bgp ipv4 unicast summary` видел `(Policy)` — FRR требовал входную политику для eBGP. Добавил `IMPORT_FROM_LEAF` и выполнил soft refresh.

```text
border-fw#show ip route
B>* 10.10.10.0/24 via 172.16.10.1, eth1
B>* 20.20.20.0/24 via 172.16.10.1, eth1
B>* 30.30.30.0/24 via 172.16.20.1, eth2
B>* 30.30.31.0/24 via 172.16.20.1, eth2
```

На Leaf3 получены default route от FRR в обе VRF:

```text
leaf3#show ip route vrf TENANT_A
Gateway of last resort:
 B E  0.0.0.0/0 [200/0]
      via 172.16.10.2, Ethernet5

leaf3#show ip route vrf TENANT_B
Gateway of last resort:
 B E  0.0.0.0/0 [200/0]
      via 172.16.20.2, Ethernet6
```

### EVPN Route-Type 5 и суммаризация

Для `TENANT_B` использовал две смежные подсети:

```text
30.30.30.0/24  (VLAN30) --+
                           +--> 30.30.30.0/23
30.30.31.0/24  (VLAN40) --+
```

На Leaf3 настроил:

```text
router bgp 65100
   vrf TENANT_B
      aggregate-address 30.30.30.0/23 summary-only
```

Проверяю EVPN-таблицу на удалённом Leaf1:

```text
leaf1#show bgp evpn route-type ip-prefix ipv4

* > RD: 10.0.103.0:50001 ip-prefix 0.0.0.0/0
      Next Hop 10.1.103.0  Path 65000 i
* > RD: 10.0.103.0:50002 ip-prefix 0.0.0.0/0
      Next Hop 10.1.103.0  Path 65000 i
* > RD: 10.0.103.0:50002 ip-prefix 30.30.30.0/23
      Next Hop 10.1.103.0
```

В выводе на Leaf1 суммарный `30.30.30.0/23` получен через Spine1 и Spine2 (RR). Отдельных RT-5 для `30.30.30.0/24` и `30.30.31.0/24` больше нет. Это результат `summary-only`. Default route для обеих VRF продолжает распространяться как RT-5.

При этом `redistribute connected` на Leaf3 также анонсирует транзитные сети `172.16.10.0/30` и `172.16.20.0/30` через EVPN. В рамках задания их не фильтровал.

### Проверка меж-VRF маршрутизации

Проверяю с server1 (`TENANT_A`) доступность обоих серверов `TENANT_B`:

```bash
ping -c 4 30.30.30.13
ping -c 4 30.30.31.14
```

Результаты после включения `summary-only`:

```text
server1 -> server3 (30.30.30.13): 4/4, loss 0%, avg 8.794 ms
server1 -> server4 (30.30.31.14): 4/4, loss 0%, avg 10.611 ms
```

Путь трафика:

```text
server1 (TENANT_A)
  -> Leaf1 (VLAN10 / VRF TENANT_A)
  -> VXLAN / L3VNI 50001
  -> Leaf3 (VRF TENANT_A)
  -> Ethernet5 / 172.16.10.0/30
  -> FRR (общая таблица маршрутизации)
  -> Ethernet6 / 172.16.20.0/30
  -> Leaf3 (VRF TENANT_B)
  -> VLAN30 или VLAN40
  -> server3 / server4
```

Ping до несуществующих адресов `30.30.30.135` и `30.30.31.154` не проходит (100% потерь). Это не ошибка маршрутизации: в лаборатории таких хостов нет.

## Проблемы при настройке

1. После запуска ранее остановленных контейнеров через `docker start` отсутствовали data-интерфейсы `eth1/eth2` и связи между узлами. `containerlab deploy -t fabric.clab.yml` восстановил линки и выполнил команды `exec` для серверов.
2. На FRR оба eBGP-соседа поднялись, но в BGP summary отображалось `(Policy)`. Добавил входной `route-map IMPORT_FROM_LEAF permit 10`, после чего FRR получил четыре подсети.
3. На server1 ранее default route шёл через management `eth0`. Исправил его на `10.10.10.254` через `bond0`; на остальных серверах также настроил default route через Anycast Gateway.
4. Сначала проверил `aggregate-address 30.30.30.0/23` без `summary-only`: в EVPN были и агрегат, и оба `/24`. После `summary-only` остался только `/23`.
5. Конфигурации Arista после настройки через CLI отдельно выгрузил в `configs/*.cfg` для Containerlab. FRR сохраняется в `configs/border-fw/frr.conf` через bind mount. IP-адреса интерфейсов FRR и `ip_forward` добавил в `exec` топологии. Повторное развёртывание с нуля после сохранения всех файлов отдельно не проверял.

## Выводы

В существующей VXLAN EVPN-фабрике настроил две изолированные VRF — `TENANT_A` и `TENANT_B`. Между ними организовал маршрутизацию через внешний FRR по двум eBGP-сессиям. FRR анонсирует default route в обе VRF, Leaf3 распространяет его через EVPN RT-5. Для `TENANT_B` настроил суммаризацию двух `/24` в `30.30.30.0/23` с `summary-only`; на Leaf1 подтвердил получение только агрегированного префикса. Проверил связность server1 с server3 и server4 после суммаризации — потерь нет. Существующий Underlay/Overlay и ESI-LAG из предыдущей работы сохранил.

# Домашнее задание №6. Сервис L3 VNI в VxLAN 
## Топология сети

## Задание:
1. Настроить каждого клиента в своем VNI;
2. Настроить маршрутизацию между клиентами;
3. Зафиксировать в документации план работы, адресное пространство, схему сети, конфигурацию устройств.

## IP-адресация:
2 сервера остались в VLAN 10, 2 сервера переведены в VLAN 20: 

| Сервер  | Leaf / порт     | VLAN | IP-адрес       | Шлюз         |
|---------|-----------------|------|----------------|--------------|
| server1 | Leaf1 Ethernet3 | 10   | 10.10.10.11/24 | 10.10.10.254 |
| server2 | Leaf2 Ethernet3 | 20   | 20.20.20.12/24 | 20.20.20.254 |
| server3 | Leaf3 Ethernet3 | 10   | 10.10.10.13/24 | 10.10.10.254 |
| server4 | Leaf3 Ethernet4 | 20   | 20.20.20.14/24 | 20.20.20.254 |

## Конфигурация Underlay/Overlay

1. Существующая фабрика не перестраивалась, остался OSPF для Underlay, iBGP для Overlay. Spines анонсируют в OSPF Loopback0, Leafs анонсируют в OSPF Loopback0 и Loopback1, Spines являются RR (не в кластере), вся EVPN фабрика в AS 65100.
2. Поверх фабрики созданы:
VRF - TENANT_A; 
VLAN10 - 10.10.10.0/24;
L2VNI VLAN10 - 10010;
Anycast Gateway VLAN10 - 10.10.10.254;
VLAN20 - 20.20.20.0/24;
L2VNI VLAN20 - 10020;
Anycast Gateway VLAN20 - 20.20.20.254;
L3VNI TENANT_A - 50001;
Anycast MAC - 0000.0000.9999;
Route Target L3VNI - 50001:50001.
3. Одинаковый Anycast MAC и одинаковые IP-адреса шлюзов настроены на всех VTEP, обслуживающих соответствующий VLAN.

### Leaf1
На Leaf1 создан VRF TENANT_A, интерфейс VLAN10 с шлюзом DGW и привязка VRF к L3VNI 50001.
RD 10.0.101.0:50001 (уникальный для Leaf1) обеспечивает обмен IP-префиксами между всеми VTEP, включёнными в TENANT_A.
Leaf1 анонсирует сеть 10.10.10.0/24 в EVPN как Type-5 маршрут.

```
vlan 10
   name SERVERS_VLAN10

vrf instance TENANT_A
ip routing vrf TENANT_A

ip virtual-router mac-address 00:00:00:00:99:99

interface Ethernet3
   description TO_SERVER1
   switchport access vlan 10

interface Vlan10
   description GATEWAY_VLAN10
   vrf TENANT_A
   ip address virtual 10.10.10.254/24

interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vrf TENANT_A vni 50001

router bgp 65100
   vlan 10
      rd auto
      route-target both 10010:10010
      redistribute learned
      address-family evpn
      neighbor SPINE activate
   vrf TENANT_A
      rd 10.0.101.0:50001
      route-target import evpn 50001:50001
      route-target export evpn 50001:50001
      redistribute connected
```

### Leaf2
Старый VLAN10 и соответствующая привязка L2VNI на Leaf2 удалены, тк клиентов VLAN10 здесь больше нет. Создан VLAN20.
Leaf2 анонсирует сеть 20.20.20.0/24 в EVPN как Type-5 маршрут и импортирует сеть 10.10.10.0/24, полученную от Leaf1 и Leaf3.

```
vlan 20
   name SERVERS_VLAN20

vrf instance TENANT_A
ip routing vrf TENANT_A

ip virtual-router mac-address 00:00:00:00:99:99

interface Ethernet3
   description TO_SERVER2
   switchport access vlan 20

interface Vlan20
   description GATEWAY_VLAN20
   vrf TENANT_A
   ip address virtual 20.20.20.254/24
   no shutdown

interface Vxlan1
   no vxlan vlan 10 vni 10010
   vxlan vlan 20 vni 10020
   vxlan vrf TENANT_A vni 50001

no vlan 10

router bgp 65100
   no vlan 10
   vlan 20
      rd auto
      route-target both 10020:10020
      redistribute learned
   vrf TENANT_A
      rd 10.0.102.0:50001
      route-target import evpn 50001:50001
      route-target export evpn 50001:50001
      redistribute connected
```

### Leaf3
Leaf3 одновременно обслуживает 2 сети: server3 подключён к VLAN10, server4 подключён к VLAN20. На Leaf3 присутствуют оба L2VNI и общий L3VNI 50001 для передачи трафика на удаленный Leaf. Leaf3 может маршрутизировать трафик между VLAN10 и VLAN20 локально. 
VLAN10/L2VNI10010 уже был на Leaf3, добавляем VLAN20, шлюзы и L3VNI.

```
vlan 20
   name SERVERS_VLAN20

vrf instance TENANT_A

ip routing vrf TENANT_A
ip virtual-router mac-address 00:00:00:00:99:99

interface Ethernet3
   description TO_SERVER3
   switchport access vlan 10

interface Ethernet4
   description TO_SERVER4
   switchport access vlan 20

interface Vlan10
   description GATEWAY_VLAN10
   vrf TENANT_A
   ip address virtual 10.10.10.254/24
   no shutdown

interface Vlan20
   description GATEWAY_VLAN20
   vrf TENANT_A
   ip address virtual 20.20.20.254/24
   no shutdown

interface Vxlan1
   vxlan vlan 20 vni 10020
   vxlan vrf TENANT_A vni 50001

router bgp 65100
   vlan 20
      rd auto
      route-target both 10020:10020
      redistribute learned
   vrf TENANT_A
      rd 10.0.103.0:50001
      route-target import evpn 50001:50001
      route-target export evpn 50001:50001
      redistribute connected
```

На Spines ничего не меняется, Spines продолжают работать как транзитные устройствами и RR, а не VTEP.

### Серверы
2 cервера остались в overlay-подсети 10.10.10.10/24, 2 cервера переведены в overlay-подсеть 20.20.20.20/24.
Изменения вносил в секции "exec" файла fabric.clab.yml:

#### Server1
```
    server1:
      kind: linux
      image: alpine:latest
      mgmt-ipv4: 172.20.20.7
      exec:
        - ip addr add 10.10.10.11/24 dev eth1
        - ip link set eth1 up
        - ip route add 20.20.20.0/24 via 10.10.10.254 dev eth1

    server2:
      kind: linux
      image: alpine:latest
      mgmt-ipv4: 172.20.20.8
      exec:
        - ip addr add 20.20.20.12/24 dev eth1
        - ip link set eth1 up
        - ip route add 10.10.10.0/24 via 20.20.20.254 dev eth1

    server3:
      kind: linux
      image: alpine:latest
      mgmt-ipv4: 172.20.20.9
      exec:
        - ip addr add 10.10.10.13/24 dev eth1
        - ip link set eth1 up
        - ip route add 20.20.20.0/24 via 10.10.10.254 dev eth1

    server4:
      kind: linux
      image: alpine:latest
      mgmt-ipv4: 172.20.20.10
      exec:
        - ip addr add 20.20.20.14/24 dev eth1
        - ip link set eth1 up
        - ip route add 10.10.10.0/24 via 20.20.20.254 dev eth1
```

## Проверка

### Проверка Server1
Проверка проверка адресации и L2-связности на сервере №1
```
/ # ip -o addr show dev eth1
80: eth1    inet 10.10.10.11/24 scope global eth1\       valid_lft forever preferred_lft forever
/ # ping -c 3 10.10.10.12
PING 10.10.10.12 (10.10.10.12): 56 data bytes
64 bytes from 10.10.10.12: seq=0 ttl=64 time=6.971 ms
64 bytes from 10.10.10.12: seq=1 ttl=64 time=8.163 ms
64 bytes from 10.10.10.12: seq=2 ttl=64 time=6.046 ms
--- 10.10.10.12 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 6.046/7.060/8.163 ms

/ # ping -c 3 10.10.10.13
PING 10.10.10.13 (10.10.10.13): 56 data bytes
64 bytes from 10.10.10.13: seq=0 ttl=64 time=5.656 ms
64 bytes from 10.10.10.13: seq=1 ttl=64 time=6.146 ms
64 bytes from 10.10.10.13: seq=2 ttl=64 time=5.640 ms
--- 10.10.10.13 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 5.640/5.814/6.146 ms

```
Видно, что Server1 (10.10.10.11/24) ходит в одной подсети без L3-шлюза на Server2 (10.10.10.12/24 через Leaf2 и на Server3 (10.10.10.13/24) через Leaf3.

### Проверка Leaf1
Проверка UNDERLAY:
```
leaf1#show ip ospf neighbor
Neighbor ID     Instance VRF      Pri State                  Dead Time   Address         Interface
10.0.2.0        1        default  0   FULL                   00:00:02    10.2.2.0        Ethernet2
10.0.1.0        1        default  0   FULL                   00:00:01    10.2.1.0        Ethernet1
```
Два соседа - Spine1 и Spine2 - в FULL.

Проверка OVERLAY:
```
leaf1#show vxlan flood vtep
          VXLAN Flood VTEP Table
--------------------------------------------------------------------------------

VLANS                            Ip Address
-----------------------------   ------------------------------------------------
10                              10.1.102.0      10.1.103.0     

```
Leaf1 видит два удалённых VTEPs в VLAN 10/VNI 10010.

Проверка L2-доступности:
```
leaf1#show vxlan address-table
          Vxlan Mac Address Table
----------------------------------------------------------------------

VLAN  Mac Address     Type      Prt  VTEP             Moves   Last Move
----  -----------     ----      ---  ----             -----   ---------
  10  aac1.ab4b.fc0d  EVPN      Vx1  10.1.102.0       1       0:08:31 ago
  10  aac1.ab5e.6eba  EVPN      Vx1  10.1.103.0       1       0:08:29 ago
  10  aac1.ab98.a6dd  EVPN      Vx1  10.1.103.0       1       0:01:07 ago
Total Remote Mac Addresses for this criterion: 3
```
Leaf1 знает, за каким удалённым VTEP находится каждый MAC.

Проверка Type-3 IMET – участие Leaf1(VTEP) в L2VNI:
```
leaf1#show bgp evpn route-type imet
BGP routing table information for VRF default
Router identifier 10.0.101.0, local AS number 65100
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending best path selection
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.0.101.0:10 imet 10.1.101.0
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.102.0:10 imet 10.1.102.0
                                 10.1.102.0            -       100     0       i Or-ID: 10.0.102.0 C-LST: 10.0.1.0 
 *  ec    RD: 10.0.102.0:10 imet 10.1.102.0
                                 10.1.102.0            -       100     0       i Or-ID: 10.0.102.0 C-LST: 10.0.2.0 
 * >Ec    RD: 10.0.103.0:10 imet 10.1.103.0
                                 10.1.103.0            -       100     0       i Or-ID: 10.0.103.0 C-LST: 10.0.2.0 
 *  ec    RD: 10.0.103.0:10 imet 10.1.103.0
                                 10.1.103.0            -       100     0       i Or-ID: 10.0.103.0 C-LST: 10.0.1.0 
leaf1#show vxlan flood vtep
          VXLAN Flood VTEP Table
--------------------------------------------------------------------------------

VLANS                            Ip Address
-----------------------------   ------------------------------------------------
10                              10.1.102.0      10.1.103.0     
```

Проверка Type-2 – распространение MAC-адресов от/к Leaf1
```
leaf1#show bgp evpn route-type mac-ip
BGP routing table information for VRF default
Router identifier 10.0.101.0, local AS number 65100
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending best path selection
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.0.102.0:10 mac-ip aac1.ab4b.fc0d
                                 10.1.102.0            -       100     0       i Or-ID: 10.0.102.0 C-LST: 10.0.2.0 
 *  ec    RD: 10.0.102.0:10 mac-ip aac1.ab4b.fc0d
                                 10.1.102.0            -       100     0       i Or-ID: 10.0.102.0 C-LST: 10.0.1.0 
 * >      RD: 10.0.101.0:10 mac-ip aac1.ab59.005c
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.103.0:10 mac-ip aac1.ab5e.6eba
                                 10.1.103.0            -       100     0       i Or-ID: 10.0.103.0 C-LST: 10.0.1.0 
 *  ec    RD: 10.0.103.0:10 mac-ip aac1.ab5e.6eba
                                 10.1.103.0            -       100     0       i Or-ID: 10.0.103.0 C-LST: 10.0.2.0 
 * >Ec    RD: 10.0.103.0:10 mac-ip aac1.ab98.a6dd
                                 10.1.103.0            -       100     0       i Or-ID: 10.0.103.0 C-LST: 10.0.1.0 
 *  ec    RD: 10.0.103.0:10 mac-ip aac1.ab98.a6dd
                                 10.1.103.0            -       100     0       i Or-ID: 10.0.103.0 C-LST: 10.0.2.0 

leaf1#show vxlan address-table
          Vxlan Mac Address Table
----------------------------------------------------------------------

VLAN  Mac Address     Type      Prt  VTEP             Moves   Last Move
----  -----------     ----      ---  ----             -----   ---------
  10  aac1.ab4b.fc0d  EVPN      Vx1  10.1.102.0       1       0:14:09 ago
  10  aac1.ab5e.6eba  EVPN      Vx1  10.1.103.0       1       0:14:07 ago
  10  aac1.ab98.a6dd  EVPN      Vx1  10.1.103.0       1       0:06:45 ago
Total Remote Mac Addresses for this criterion: 3
```

### Проверка Spine1

```
spine1#show bgp evpn summary
BGP summary information for VRF default
Router identifier 10.0.1.0, local AS number 65100
Neighbor Status Codes: m - Under maintenance
  Description              Neighbor   V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  LEAF1                    10.0.101.0 4 65100             51        59    0    0 00:36:50 Estab   2      2
  LEAF2                    10.0.102.0 4 65100             49        57    0    0 00:36:51 Estab   2      2
  LEAF3                    10.0.103.0 4 65100             56        55    0    0 00:36:49 Estab   3      3
```
Видно три Established peer, Spine1 работает как RR для Leaf1-3.

## Выводы:

Реализован VLAN-Based Service VLAN 10/VNI 10010, объединивший 4 сервера в подсеть 10.10.10.0/24 по VXLAN. 
EVPN Type-3 используется для обнаружения участников VNI и формирования BUM флад-листа. 
EVPN Type-2 используется для распространения информации о MAC-адресах между VTEP.


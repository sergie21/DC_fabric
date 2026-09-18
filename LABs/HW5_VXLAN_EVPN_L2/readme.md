# Домашнее задание №5. Сервис L2 VNI в VxLAN 
## Топология сети
<img width="757" height="785" alt="L2 EPVN" src="https://github.com/user-attachments/assets/80b489f7-1974-42f7-bc8a-eb706e350625" />

## Задание:
1. Настроить BGP peering между Leaf и Spine в AF l2VPN EVPN;
2. Настроить связность между клиентами в первой зоне и убедиться в её наличии;
3. Зафиксировать в документации план работы, адресное пространство, схему сети, конфигурацию устройств.

## IP-адресация:
Адресация UNDERLAY/OVERLAY остается та же, но меняются адреса серверов: вместо старых routed /31 серверы переведены в одну overlay-подсеть 10.10.10.10/24.

#### Leaf-server адресация
| Link    | Server interface/IP | Prefix |
| ------  | -----------------   | ------ |
| Server1 | Et1 `10.10.10.11`   | `/24`  |
| Server2 | Et1 `10.10.10.12`   | `/24`  |
| Server3 | Et1 `10.10.10.13`   | `/24`  |
| Server4 | Et1 `10.10.10.14`   | `/24`  |

## Конфигурация iBGP

Был выбран OSPF для Underlay, iBGP для Overlay.
Конфигурация Underlay OSPF была подгружена из ДЗ №2. Spines анонсируют в OSPF Loopback0, Leafs анонсируют в OSPF Loopback0 и Loopback1.
Учтены рекомендации урока VxLAN EVPN для L2: Spines являются RR (не в кластере), next-hop-self не используется, retain route-target отсутствует (вся EVPN фабрика в AS 65100)

### Spine1
```
router bgp 65100
   router-id 10.0.1.0
   no bgp default ipv4-unicast
   neighbor LEAF peer group
   neighbor LEAF remote-as 65100
   neighbor LEAF update-source Loopback0
   neighbor LEAF send-community extended
   neighbor LEAF maximum-routes 12000
   neighbor LEAF route-reflector-client

   neighbor 10.0.101.0 peer group LEAF
   neighbor 10.0.101.0 description LEAF1

   neighbor 10.0.102.0 peer group LEAF
   neighbor 10.0.102.0 description LEAF2

   neighbor 10.0.103.0 peer group LEAF
   neighbor 10.0.103.0 description LEAF3

   address-family evpn
      neighbor LEAF activate
end
```

### Spine1
```
router bgp 65100
   router-id 10.0.2.0
   no bgp default ipv4-unicast
   neighbor LEAF peer group
   neighbor LEAF remote-as 65100
   neighbor LEAF update-source Loopback0
   neighbor LEAF send-community extended
   neighbor LEAF maximum-routes 12000
   neighbor LEAF route-reflector-client

   neighbor 10.0.101.0 peer group LEAF
   neighbor 10.0.101.0 description LEAF1

   neighbor 10.0.102.0 peer group LEAF
   neighbor 10.0.102.0 description LEAF2

   neighbor 10.0.103.0 peer group LEAF
   neighbor 10.0.103.0 description LEAF3

   address-family evpn
      neighbor LEAF activate
end
```
На Spines не настраивается VXLAN, Spines являются транзитными устройствами и RR, а не VTEP. Не применяется next-hop-self, при отражении EVPN-маршрутов next-hop остается адресом VTEP Leaf.

### Leaf1
```
router bgp 65100
   router-id 10.0.101.0
   no bgp default ipv4-unicast

   neighbor SPINE peer group
   neighbor SPINE remote-as 65100
   neighbor SPINE update-source Loopback0
   neighbor SPINE send-community extended
   neighbor SPINE maximum-routes 12000

   neighbor 10.0.1.0 peer group SPINE
   neighbor 10.0.1.0 description SPINE1

   neighbor 10.0.2.0 peer group SPINE
   neighbor 10.0.2.0 description SPINE2

   address-family evpn
      neighbor SPINE activate
```
После проверки BGP был добавлен L2-сервис для серверов:
```
vlan 10
   name SERVERS_VLAN10
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan vlan 10 vni 10010
router bgp 65100
   vlan 10
      rd auto
      route-target both 10010:10010
      redistribute learned
interface Ethernet3
   no ip address
   switchport
   switchport access vlan 10
end
```

### Leaf2
```
router bgp 65100
   router-id 10.0.102.0
   no bgp default ipv4-unicast

   neighbor SPINE peer group
   neighbor SPINE remote-as 65100
   neighbor SPINE update-source Loopback0
   neighbor SPINE send-community extended
   neighbor SPINE maximum-routes 12000

   neighbor 10.0.1.0 peer group SPINE
   neighbor 10.0.1.0 description SPINE1

   neighbor 10.0.2.0 peer group SPINE
   neighbor 10.0.2.0 description SPINE2

   address-family evpn
      neighbor SPINE activate

vlan 10
   name SERVERS_VLAN10

interface Vxlan1
   vxlan source-interface Loopback1
   vxlan vlan 10 vni 10010

router bgp 65100
   vlan 10
      rd auto
      route-target both 10010:10010
      redistribute learned

interface Ethernet3
   no ip address
   switchport
   switchport access vlan 10

end
```

### Leaf3
```
router bgp 65100
   router-id 10.0.103.0
   no bgp default ipv4-unicast

   neighbor SPINE peer group
   neighbor SPINE remote-as 65100
   neighbor SPINE update-source Loopback0
   neighbor SPINE send-community extended
   neighbor SPINE maximum-routes 12000

   neighbor 10.0.1.0 peer group SPINE
   neighbor 10.0.1.0 description SPINE1

   neighbor 10.0.2.0 peer group SPINE
   neighbor 10.0.2.0 description SPINE2

   address-family evpn
      neighbor SPINE activate

vlan 10
   name SERVERS_VLAN10

interface Vxlan1
   vxlan source-interface Loopback1
   vxlan vlan 10 vni 10010

router bgp 65100
   vlan 10
      rd auto
      route-target both 10010:10010
      redistribute learned

interface Ethernet3
   no ip address
   switchport
   switchport access vlan 10

interface Ethernet4
   no ip address
   switchport
   switchport access vlan 10

end
```

### Серверы
Вместо старых routed /31 серверы переведены в одну overlay-подсеть 10.10.10.10/24 (синтаксис сетевого линукса):

#### Server1
```
ip addr flush dev eth1
ip addr add 10.10.10.11/24 dev eth1
ip link set eth1 up
```
#### Server2
```
ip addr flush dev eth1
ip addr add 10.10.10.12/24 dev eth1
ip link set eth1 up
```
#### Server3
```
ip addr flush dev eth1
ip addr add 10.10.10.13/24 dev eth1
ip link set eth1 up
```
#### Server4
```
ip addr flush dev eth1
ip addr add 10.10.10.14/24 dev eth1
ip link set eth1 up
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

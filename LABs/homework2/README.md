# Домашнее задание №2. Построение Underlay с помощью OSPF

Построен Underlay на базе OSPF: 2 Spine, 3 Leaf объединены L3 P2P-соединениями /31 в Area 0. В OSPF анонсируются Loopback0 всех сетевых устройств и Loopback1 Leaf-коммутаторов для IP-достижимости будущих VTEP.

## Конфигурация OSPF
### Leaf 1
```
enable
configure terminal

interface Ethernet1
   description TO_SPINE1
   mtu 9214
   no switchport
   ip address 10.2.1.1/31
   ip ospf dead-interval 3
   ip ospf hello-interval 1
   ip ospf network point-to-point
   ip ospf area 0.0.0.0

interface Ethernet2
   description TO_SPINE2
   mtu 9214
   no switchport
   ip address 10.2.2.1/31
   ip ospf dead-interval 3
   ip ospf hello-interval 1
   ip ospf network point-to-point
   ip ospf area 0.0.0.0

interface Ethernet3
   description TO_SERVER1
   no switchport
   ip address 10.4.1.0/31

interface Loopback0
   description OSPF_ROUTER_ID
   ip address 10.0.11.0/32

interface Loopback1
   description VTEP
   ip address 10.1.11.0/32

ip routing

router ospf 1
   router-id 10.0.11.0
   bfd default
   network 10.0.11.0/32 area 0.0.0.0
   network 10.1.11.0/32 area 0.0.0.0
   max-lsa 12000
   maximum-paths 4

end
write memory
```
### Leaf 2
```
enable
configure terminal

interface Ethernet1
   description TO_SPINE1
   mtu 9214
   no switchport
   ip address 10.2.1.3/31
   ip ospf dead-interval 3
   ip ospf hello-interval 1
   ip ospf network point-to-point
   ip ospf area 0.0.0.0

interface Ethernet2
   description TO_SPINE2
   mtu 9214
   no switchport
   ip address 10.2.2.3/31
   ip ospf dead-interval 3
   ip ospf hello-interval 1
   ip ospf network point-to-point
   ip ospf area 0.0.0.0

interface Ethernet3
   description TO_SERVER2
   no switchport
   ip address 10.4.2.0/31

interface Loopback0
   description OSPF_ROUTER_ID
   ip address 10.0.12.0/32

interface Loopback1
   description VTEP
   ip address 10.1.12.0/32

ip routing

router ospf 1
   router-id 10.0.12.0
   bfd default
   network 10.0.12.0/32 area 0.0.0.0
   network 10.1.12.0/32 area 0.0.0.0
   max-lsa 12000
   maximum-paths 4

end
write memory
```
### Leaf 3
```
enable
configure terminal

interface Ethernet1
   description TO_SPINE1
   mtu 9214
   no switchport
   ip address 10.2.1.5/31
   ip ospf dead-interval 3
   ip ospf hello-interval 1
   ip ospf network point-to-point
   ip ospf area 0.0.0.0

interface Ethernet2
   description TO_SPINE2
   mtu 9214
   no switchport
   ip address 10.2.2.5/31
   ip ospf dead-interval 3
   ip ospf hello-interval 1
   ip ospf network point-to-point
   ip ospf area 0.0.0.0

interface Ethernet3
   description TO_SERVER3
   no switchport
   ip address 10.4.3.0/31

interface Ethernet4
   description TO_SERVER4
   no switchport
   ip address 10.4.3.2/31

interface Loopback0
   description OSPF_ROUTER_ID
   ip address 10.0.13.0/32

interface Loopback1
   description VTEP
   ip address 10.1.13.0/32

ip routing

router ospf 1
   router-id 10.0.13.0
   bfd default
   network 10.0.13.0/32 area 0.0.0.0
   network 10.1.13.0/32 area 0.0.0.0
   max-lsa 12000
   maximum-paths 4

end
write memory
```
### Spine 1
```
enable
configure terminal

interface Ethernet1
   description TO_LEAF1
   mtu 9214
   no switchport
   ip address 10.2.1.0/31
   ip ospf dead-interval 3
   ip ospf hello-interval 1
   ip ospf network point-to-point
   ip ospf area 0.0.0.0

interface Ethernet2
   description TO_LEAF2
   mtu 9214
   no switchport
   ip address 10.2.1.2/31
   ip ospf dead-interval 3
   ip ospf hello-interval 1
   ip ospf network point-to-point
   ip ospf area 0.0.0.0

interface Ethernet3
   description TO_LEAF3
   mtu 9214
   no switchport
   ip address 10.2.1.4/31
   ip ospf dead-interval 3
   ip ospf hello-interval 1
   ip ospf network point-to-point
   ip ospf area 0.0.0.0

interface Loopback0
   description OSPF_ROUTER_ID
   ip address 10.0.1.0/32

ip routing

router ospf 1
   router-id 10.0.1.0
   bfd default
   network 10.0.1.0/32 area 0.0.0.0
   max-lsa 12000
   maximum-paths 4

end
write memory
```
### Spine 2
```
enable
configure terminal

interface Ethernet1
   description TO_LEAF1
   mtu 9214
   no switchport
   ip address 10.2.2.0/31
   ip ospf dead-interval 3
   ip ospf hello-interval 1
   ip ospf network point-to-point
   ip ospf area 0.0.0.0

interface Ethernet2
   description TO_LEAF2
   mtu 9214
   no switchport
   ip address 10.2.2.2/31
   ip ospf dead-interval 3
   ip ospf hello-interval 1
   ip ospf network point-to-point
   ip ospf area 0.0.0.0

interface Ethernet3
   description TO_LEAF3
   mtu 9214
   no switchport
   ip address 10.2.2.4/31
   ip ospf dead-interval 3
   ip ospf hello-interval 1
   ip ospf network point-to-point
   ip ospf area 0.0.0.0

interface Loopback0
   description OSPF_ROUTER_ID
   ip address 10.0.2.0/32

ip routing

router ospf 1
   router-id 10.0.2.0
   bfd default
   network 10.0.2.0/32 area 0.0.0.0
   max-lsa 12000
   maximum-paths 4

end
write memory
```
## Проверка OSPF
### Проверка OSPF соседства
```
leaf1>show ip ospf neighbor
Neighbor ID     Instance VRF      Pri State                  Dead Time   Address         Interface
10.0.2.0        1        default  0   FULL                   00:00:01    10.2.2.0        Ethernet2
10.0.1.0        1        default  0   FULL                   00:00:01    10.2.1.0        Ethernet1
```
```
leaf2>show ip ospf neighbor
Neighbor ID     Instance VRF      Pri State                  Dead Time   Address         Interface
10.0.2.0        1        default  0   FULL                   00:00:01    10.2.2.2        Ethernet2
10.0.1.0        1        default  0   FULL                   00:00:01    10.2.1.2        Ethernet1
```
```
leaf3>show ip ospf neighbor
Neighbor ID     Instance VRF      Pri State                  Dead Time   Address         Interface
10.0.2.0        1        default  0   FULL                   00:00:01    10.2.2.4        Ethernet2
10.0.1.0        1        default  0   FULL                   00:00:01    10.2.1.4        Ethernet1
```
```
spine1>show ip ospf neighbor
Neighbor ID     Instance VRF      Pri State                  Dead Time   Address         Interface
10.0.12.0       1        default  0   FULL                   00:00:01    10.2.1.3        Ethernet2
10.0.11.0       1        default  0   FULL                   00:00:01    10.2.1.1        Ethernet1
10.0.13.0       1        default  0   FULL                   00:00:01    10.2.1.5        Ethernet3
```
```
spine2>show ip ospf neighbor
Neighbor ID     Instance VRF      Pri State                  Dead Time   Address         Interface
10.0.12.0       1        default  0   FULL                   00:00:01    10.2.2.3        Ethernet2
10.0.11.0       1        default  0   FULL                   00:00:01    10.2.2.1        Ethernet1
10.0.13.0       1        default  0   FULL                   00:00:01    10.2.2.5        Ethernet3
```
Каждый Leaf установил по 2 соседства со Spine1 и Spine2 (FULL), каждый Spine установил по три соседства с Leaf1–Leaf3 (FULL).

### Проверка OSPF маршрутизации
#### Выборочно проверим на Leaf1
```
leaf1#show ip route ospf

VRF: default
Source Codes:
       C - connected, S - static, K - kernel,
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - Other BGP Routes,
       B I - iBGP, B E - eBGP, R - RIP, I L1 - IS-IS level 1,
       I L2 - IS-IS level 2, O3 - OSPFv3, A B - BGP Aggregate,
       A O - OSPF Summary, NG - Nexthop Group Static Route,
       V - VXLAN Control Service, M - Martian,
       DH - DHCP client installed default route,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route,
       CL - CBF Leaked Route

 O        10.0.1.0/32 [110/20]
           via 10.2.1.0, Ethernet1
 O        10.0.2.0/32 [110/20]
           via 10.2.2.0, Ethernet2
 O        10.0.12.0/32 [110/30]
           via 10.2.1.0, Ethernet1
           via 10.2.2.0, Ethernet2
 O        10.0.13.0/32 [110/30]
           via 10.2.1.0, Ethernet1
           via 10.2.2.0, Ethernet2
 O        10.1.12.0/32 [110/30]
           via 10.2.1.0, Ethernet1
           via 10.2.2.0, Ethernet2
 O        10.1.13.0/32 [110/30]
           via 10.2.1.0, Ethernet1
           via 10.2.2.0, Ethernet2
 O        10.2.1.2/31 [110/20]
           via 10.2.1.0, Ethernet1
 O        10.2.1.4/31 [110/20]
           via 10.2.1.0, Ethernet1
 O        10.2.2.2/31 [110/20]
           via 10.2.2.0, Ethernet2
 O        10.2.2.4/31 [110/20]
           via 10.2.2.0, Ethernet2
```
На Leaf1 до удалённых Leaf доступны 2 пути через Spine1 и Spine2, ECMP работает.

#### Выборочно проверим на Spine2
```
spine2#show ip route ospf

VRF: default
Source Codes:
       C - connected, S - static, K - kernel,
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - Other BGP Routes,
       B I - iBGP, B E - eBGP, R - RIP, I L1 - IS-IS level 1,
       I L2 - IS-IS level 2, O3 - OSPFv3, A B - BGP Aggregate,
       A O - OSPF Summary, NG - Nexthop Group Static Route,
       V - VXLAN Control Service, M - Martian,
       DH - DHCP client installed default route,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route,
       CL - CBF Leaked Route

 O        10.0.1.0/32 [110/30]
           via 10.2.2.1, Ethernet1
           via 10.2.2.3, Ethernet2
           via 10.2.2.5, Ethernet3
 O        10.0.11.0/32 [110/20]
           via 10.2.2.1, Ethernet1
 O        10.0.12.0/32 [110/20]
           via 10.2.2.3, Ethernet2
 O        10.0.13.0/32 [110/20]
           via 10.2.2.5, Ethernet3
 O        10.1.11.0/32 [110/20]
           via 10.2.2.1, Ethernet1
 O        10.1.12.0/32 [110/20]
           via 10.2.2.3, Ethernet2
 O        10.1.13.0/32 [110/20]
           via 10.2.2.5, Ethernet3
 O        10.2.1.0/31 [110/20]
           via 10.2.2.1, Ethernet1
 O        10.2.1.2/31 [110/20]
           via 10.2.2.3, Ethernet2
 O        10.2.1.4/31 [110/20]
           via 10.2.2.5, Ethernet3

```
## Проверка IP-связности лупбеков через OSPF Underlay
### Loopbacks'1 между Leaf1 и Leaf3 
```
leaf1#ping 10.1.13.0 source 10.1.11.0
PING 10.1.13.0 (10.1.13.0) from 10.1.11.0 : 72(100) bytes of data.
80 bytes from 10.1.13.0: icmp_seq=1 ttl=63 time=3.15 ms
80 bytes from 10.1.13.0: icmp_seq=2 ttl=63 time=1.20 ms
80 bytes from 10.1.13.0: icmp_seq=3 ttl=63 time=1.75 ms
80 bytes from 10.1.13.0: icmp_seq=4 ttl=63 time=1.24 ms
80 bytes from 10.1.13.0: icmp_seq=5 ttl=63 time=1.22 ms

--- 10.1.13.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 12ms
rtt min/avg/max/mdev = 1.199/1.713/3.153/0.748 ms, ipg/ewma 3.094/2.404 ms
```
Достижимы
### Loopbacks'0 между Spine1 и Spine2
```
spine1#ping 10.0.2.0 source 10.0.1.0
PING 10.0.2.0 (10.0.2.0) from 10.0.1.0 : 72(100) bytes of data.
80 bytes from 10.0.2.0: icmp_seq=1 ttl=63 time=6.75 ms
80 bytes from 10.0.2.0: icmp_seq=2 ttl=63 time=1.39 ms
80 bytes from 10.0.2.0: icmp_seq=3 ttl=63 time=1.65 ms
80 bytes from 10.0.2.0: icmp_seq=4 ttl=63 time=1.12 ms
80 bytes from 10.0.2.0: icmp_seq=5 ttl=63 time=1.80 ms

--- 10.0.2.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 24ms
rtt min/avg/max/mdev = 1.123/2.539/6.745/2.115 ms, ipg/ewma 6.001/4.574 ms
```
Достижимы

# Домашнее задание №2. Построение Underlay с помощью IS-IS
## Топология сети
<img width="884" height="818" alt="IS-IS" src="https://github.com/user-attachments/assets/6003e2ea-215d-40ea-972c-5f98a291b86d" />

## Задание
1. Настроить IS-IS в Underlay сети для IP-связности между всеми сетевыми устройствами;
2. Зафиксировать в документации - план работы, адресное пространство, схему сети, конфигурацию устройств;
3. Убедиться в наличии IP-связности между устройствами в IS-IS домене.

## IP-адресация
### Underlay P2P/31
| Device | IP Address/Mask | Port      | Remote Device | Remote Port | Description |
| ------ | --------------- | --------- | ------------- | ----------- | ----------- |
| Leaf1  | `10.2.1.1/31`   | Ethernet1 | Spine1        | Ethernet1   | to Spine1   |
| Leaf1  | `10.2.2.1/31`   | Ethernet2 | Spine2        | Ethernet1   | to Spine2   |
| Leaf2  | `10.2.1.3/31`   | Ethernet1 | Spine1        | Ethernet2   | to Spine1   |
| Leaf2  | `10.2.2.3/31`   | Ethernet2 | Spine2        | Ethernet2   | to Spine2   |
| Leaf3  | `10.2.1.5/31`   | Ethernet1 | Spine1        | Ethernet3   | to Spine1   |
| Leaf3  | `10.2.2.5/31`   | Ethernet2 | Spine2        | Ethernet3   | to Spine2   |
| Spine1 | `10.2.1.0/31`   | Ethernet1 | Leaf1         | Ethernet1   | to Leaf1    |
| Spine1 | `10.2.1.2/31`   | Ethernet2 | Leaf2         | Ethernet1   | to Leaf2    |
| Spine1 | `10.2.1.4/31`   | Ethernet3 | Leaf3         | Ethernet1   | to Leaf3    |
| Spine2 | `10.2.2.0/31`   | Ethernet1 | Leaf1         | Ethernet2   | to Leaf1    |
| Spine2 | `10.2.2.2/31`   | Ethernet2 | Leaf2         | Ethernet2   | to Leaf2    |
| Spine2 | `10.2.2.4/31`   | Ethernet3 | Leaf3         | Ethernet2   | to Leaf3    |

### Loopback's и NET-адреса
| Device | Loopback0       | NET Address                   | Level |
| ------ | --------------  | ----------------------        | ----- |
| Spine1 | `10.0.1.0/32`   | `49.0001.0100.0000.1000.00`   | L1    |
| Spine2 | `10.0.2.0/32`   | `49.0001.0100.0000.2000.00`   | L1    |
| Leaf1  | `10.0.101.0/32` | `49.0001.0100.0001.0100.00`   | L1    |
| Leaf2  | `10.0.102.0/32` | `49.0001.0100.0001.0200.00`   | L1    |
| Leaf3  | `10.0.103.0/32` | `49.0001.0100.0001.0300.00`   | L1    |

Все устройства находятся в L1-домене

### Серверные подключения
| Device | IP Address/Mask | Port      | Remote Device | Remote IP     | Description |
| ------ | --------------- | --------- | ------------- | ------------- | ----------- |
| Leaf1  | `10.4.1.0/31`   | Ethernet3 | Server1       | `10.4.1.1/31` | Server1     |
| Leaf2  | `10.4.2.0/31`   | Ethernet3 | Server2       | `10.4.2.1/31` | Server2     |
| Leaf3  | `10.4.3.0/31`   | Ethernet3 | Server3       | `10.4.3.1/31` | Server3     |
| Leaf3  | `10.4.3.2/31`   | Ethernet4 | Server4       | `10.4.3.3/31` | Server4     |

Серверные сети подключены к Leaf через routed /31-интерфейсы и в IS-IS не анонсируются.

## Конфигурация IS-IS

MTU определен как 9000, учтены некоторые рекомендации урока IS-IS (P2P, BFD, L1 для одного POD с малым количеством устройств)

### Spine1
```
enable
configure terminal

router isis UNDERLAY
   net 49.0001.0100.0000.1000.00
   is-type level-1
   address-family ipv4 unicast

interface Ethernet1
   description TO_LEAF1
   mtu 9000
   no switchport
   ip address 10.2.1.0/31
   isis enable UNDERLAY
   isis network point-to-point
   isis circuit-type level-1
   isis bfd

interface Ethernet2
   description TO_LEAF2
   mtu 9000
   no switchport
   ip address 10.2.1.2/31
   isis enable UNDERLAY
   isis network point-to-point
   isis circuit-type level-1
   isis bfd

interface Ethernet3
   description TO_LEAF3
   mtu 9000
   no switchport
   ip address 10.2.1.4/31
   isis enable UNDERLAY
   isis network point-to-point
   isis circuit-type level-1
   isis bfd

interface Loopback0
   description UNDERLAY_ROUTER_ID
   ip address 10.0.1.0/32
   isis enable UNDERLAY
   isis passive

ip routing

end
```
### Spine2
```
enable
configure terminal

router isis UNDERLAY
   net 49.0001.0100.0000.2000.00
   is-type level-1
   address-family ipv4 unicast

interface Ethernet1
   description TO_LEAF1
   mtu 9000
   no switchport
   ip address 10.2.2.0/31
   isis enable UNDERLAY
   isis network point-to-point
   isis circuit-type level-1
   isis bfd

interface Ethernet2
   description TO_LEAF2
   mtu 9000
   no switchport
   ip address 10.2.2.2/31
   isis enable UNDERLAY
   isis network point-to-point
   isis circuit-type level-1
   isis bfd

interface Ethernet3
   description TO_LEAF3
   mtu 9000
   no switchport
   ip address 10.2.2.4/31
   isis enable UNDERLAY
   isis network point-to-point
   isis circuit-type level-1
   isis bfd

interface Loopback0
   description UNDERLAY_ROUTER_ID
   ip address 10.0.2.0/32
   isis enable UNDERLAY
   isis passive

ip routing

end
```
### Leaf1
```
enable
configure terminal

router isis UNDERLAY
   net 49.0001.0100.0010.1000.00
   is-type level-1
   address-family ipv4 unicast

interface Ethernet1
   description TO_SPINE1
   mtu 9000
   no switchport
   ip address 10.2.1.1/31
   isis enable UNDERLAY
   isis network point-to-point
   isis circuit-type level-1
   isis bfd

interface Ethernet2
   description TO_SPINE2
   mtu 9000
   no switchport
   ip address 10.2.2.1/31
   isis enable UNDERLAY
   isis network point-to-point
   isis circuit-type level-1
   isis bfd

interface Loopback0
   description UNDERLAY_ROUTER_ID
   ip address 10.0.101.0/32
   isis enable UNDERLAY
   isis passive

interface Loopback1
   description VTEP
   ip address 10.1.101.0/32
   isis enable UNDERLAY
   isis passive

ip routing

end
```
### Leaf2
```
enable
configure terminal

router isis UNDERLAY
   net 49.0001.0100.0010.2000.00
   is-type level-1
   address-family ipv4 unicast

interface Ethernet1
   description TO_SPINE1
   mtu 9000
   no switchport
   ip address 10.2.1.3/31
   isis enable UNDERLAY
   isis network point-to-point
   isis circuit-type level-1
   isis bfd

interface Ethernet2
   description TO_SPINE2
   mtu 9000
   no switchport
   ip address 10.2.2.3/31
   isis enable UNDERLAY
   isis network point-to-point
   isis circuit-type level-1
   isis bfd

interface Loopback0
   description UNDERLAY_ROUTER_ID
   ip address 10.0.102.0/32
   isis enable UNDERLAY
   isis passive

interface Loopback1
   description VTEP
   ip address 10.1.102.0/32
   isis enable UNDERLAY
   isis passive

ip routing

end
```
### Leaf3
```
enable
configure terminal

router isis UNDERLAY
   net 49.0001.0100.0010.3000.00
   is-type level-1
   address-family ipv4 unicast

interface Ethernet1
   description TO_SPINE1
   mtu 9000
   no switchport
   ip address 10.2.1.5/31
   isis enable UNDERLAY
   isis network point-to-point
   isis circuit-type level-1
   isis bfd

interface Ethernet2
   description TO_SPINE2
   mtu 9000
   no switchport
   ip address 10.2.2.5/31
   isis enable UNDERLAY
   isis network point-to-point
   isis circuit-type level-1
   isis bfd

interface Loopback0
   description UNDERLAY_ROUTER_ID
   ip address 10.0.103.0/32
   isis enable UNDERLAY
   isis passive

interface Loopback1
   description VTEP
   ip address 10.1.103.0/32
   isis enable UNDERLAY
   isis passive

ip routing

end
```

### Проверка IS-IS на Spine2
Соседи в АПе:
```
spine2#sh isis neighbors
 
Instance  VRF      System Id        Type Interface          SNPA              State Hold time   Circuit Id          
UNDERLAY  default  leaf1            L1   Ethernet1          P2P               UP    20          29                  
UNDERLAY  default  leaf2            L1   Ethernet2          P2P               UP    27          2E                  
UNDERLAY  default  leaf3            L1   Ethernet3          P2P               UP    29          34                  
```
В БД все устройства:
```
spine2#sh isis database
Legend:
H - hostname conflict
U - node unreachable

IS-IS Instance: UNDERLAY VRF: default
  IS-IS Level 1 Link State Database
    LSPID                   Seq Num  Cksum  Life Length IS  Received LSPID        Flags
    spine1.00-00                  8  13519   599    146 L1  0100.0000.1000.00-00  <>
    spine2.00-00                  8  48683  1126    146 L1  0100.0000.2000.00-00  <>
    leaf1.00-00                   7  11736  1038    134 L1  0100.0010.1000.00-00  <>
    leaf2.00-00                   6  26752   550    134 L1  0100.0010.2000.00-00  <>
    leaf3.00-00                   6  24427   640    134 L1  0100.0010.3000.00-00  <>
```
Маршруты, в тч ECMP:
```
spine2#sh ip route isis

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

 I L1     10.0.1.0/32 [115/30]
           via 10.2.2.1, Ethernet1
           via 10.2.2.3, Ethernet2
           via 10.2.2.5, Ethernet3
 I L1     10.0.101.0/32 [115/20]
           via 10.2.2.1, Ethernet1
 I L1     10.0.102.0/32 [115/20]
           via 10.2.2.3, Ethernet2
 I L1     10.0.103.0/32 [115/20]
           via 10.2.2.5, Ethernet3
 I L1     10.1.101.0/32 [115/20]
           via 10.2.2.1, Ethernet1
 I L1     10.1.102.0/32 [115/20]
           via 10.2.2.3, Ethernet2
 I L1     10.1.103.0/32 [115/20]
           via 10.2.2.5, Ethernet3
 I L1     10.2.1.0/31 [115/20]
           via 10.2.2.1, Ethernet1
 I L1     10.2.1.2/31 [115/20]
           via 10.2.2.3, Ethernet2
 I L1     10.2.1.4/31 [115/20]
           via 10.2.2.5, Ethernet3
```
### Выборочные пинги со Spine2
```
spine2#
spine2#ping 10.0.101.0 source 10.0.2.0
PING 10.0.101.0 (10.0.101.0) from 10.0.2.0 : 72(100) bytes of data.
80 bytes from 10.0.101.0: icmp_seq=1 ttl=64 time=1.41 ms
80 bytes from 10.0.101.0: icmp_seq=2 ttl=64 time=0.040 ms
80 bytes from 10.0.101.0: icmp_seq=3 ttl=64 time=0.045 ms
80 bytes from 10.0.101.0: icmp_seq=4 ttl=64 time=0.106 ms
80 bytes from 10.0.101.0: icmp_seq=5 ttl=64 time=0.042 ms

--- 10.0.101.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 5ms
rtt min/avg/max/mdev = 0.040/0.328/1.411/0.541 ms, ipg/ewma 1.168/0.851 ms
spine2#ping 10.0.103.0 source 10.0.2.0
PING 10.0.103.0 (10.0.103.0) from 10.0.2.0 : 72(100) bytes of data.
80 bytes from 10.0.103.0: icmp_seq=1 ttl=64 time=3.16 ms
80 bytes from 10.0.103.0: icmp_seq=2 ttl=64 time=0.026 ms
80 bytes from 10.0.103.0: icmp_seq=3 ttl=64 time=0.102 ms
80 bytes from 10.0.103.0: icmp_seq=4 ttl=64 time=0.044 ms
80 bytes from 10.0.103.0: icmp_seq=5 ttl=64 time=0.150 ms

--- 10.0.103.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 10ms
rtt min/avg/max/mdev = 0.026/0.696/3.158/1.231 ms, ipg/ewma 2.611/1.886 ms
spine2#ping 10.1.102.0 source 10.0.2.0
PING 10.1.102.0 (10.1.102.0) from 10.0.2.0 : 72(100) bytes of data.
80 bytes from 10.1.102.0: icmp_seq=1 ttl=64 time=3.87 ms
80 bytes from 10.1.102.0: icmp_seq=2 ttl=64 time=0.038 ms
80 bytes from 10.1.102.0: icmp_seq=3 ttl=64 time=0.047 ms
80 bytes from 10.1.102.0: icmp_seq=4 ttl=64 time=0.042 ms
80 bytes from 10.1.102.0: icmp_seq=5 ttl=64 time=0.106 ms

--- 10.1.102.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 13ms
rtt min/avg/max/mdev = 0.038/0.821/3.874/1.526 ms, ipg/ewma 3.278/2.296 ms
spine2#ping 10.0.1.0 source 10.0.2.0
PING 10.0.1.0 (10.0.1.0) from 10.0.2.0 : 72(100) bytes of data.
80 bytes from 10.0.1.0: icmp_seq=1 ttl=63 time=6.26 ms
80 bytes from 10.0.1.0: icmp_seq=2 ttl=63 time=0.818 ms
80 bytes from 10.0.1.0: icmp_seq=3 ttl=63 time=1.61 ms
80 bytes from 10.0.1.0: icmp_seq=4 ttl=63 time=1.01 ms
80 bytes from 10.0.1.0: icmp_seq=5 ttl=63 time=1.10 ms

--- 10.0.1.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 23ms
rtt min/avg/max/mdev = 0.818/2.159/6.262/2.068 ms, ipg/ewma 5.816/4.140 ms
```
Все достижимо

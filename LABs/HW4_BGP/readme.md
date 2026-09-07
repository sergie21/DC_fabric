# Домашнее задание №4. Построение Underlay с помощью iBGP
## Топология сети
<img width="884" height="818" alt="iBGP" src="https://github.com/user-attachments/assets/e323ba55-2fba-4b09-9a4c-358f28763550" />

## Задание
1. Настроить BGP в Underlay сети, для IP связанности между всеми сетевыми устройствами;
2. Зафиксировать в документации план работы, адресное пространство, схему сети, конфигурацию устройств;
3. Убедиться в наличии IP-связности между устройствами в BGP домене.

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

### Loopback's и RID-адреса
| Device | Loopback0       | 
| ------ | --------------  | 
| Spine1 | `10.0.1.0/32`   | 
| Spine2 | `10.0.2.0/32`   | 
| Leaf1  | `10.0.101.0/32` |
| Leaf2  | `10.0.102.0/32` | 
| Leaf3  | `10.0.103.0/32` |

### Серверные подключения
| Device | IP Address/Mask | Port      | Remote Device | Remote IP     | Description |
| ------ | --------------- | --------- | ------------- | ------------- | ----------- |
| Leaf1  | `10.4.1.0/31`   | Ethernet3 | Server1       | `10.4.1.1/31` | Server1     |
| Leaf2  | `10.4.2.0/31`   | Ethernet3 | Server2       | `10.4.2.1/31` | Server2     |
| Leaf3  | `10.4.3.0/31`   | Ethernet3 | Server3       | `10.4.3.1/31` | Server3     |
| Leaf3  | `10.4.3.2/31`   | Ethernet4 | Server4       | `10.4.3.3/31` | Server4     |

Серверные сети подключены к Leaf через routed /31-интерфейсы и в BGP не анонсируются.

## Конфигурация iBGP

Выбран iBGP. Учтены рекомендации уроков iBGP/eBGP (пароль DC_FABRIC для аутентификации на линках leaf-spine, multipath, 2 spines как 2 route-reflectors не в кластере, route-map на Lo0, подкручены таймеры 3/9, включен BFD)

### Spine1
```
route-map REDISTRIBUTE_CONNECTED permit 10
   match interface Loopback0

router bgp 65100
   router-id 10.0.1.0
   no bgp default ipv4-unicast
   maximum-paths ibgp 2

   neighbor LEAF peer group
   neighbor LEAF remote-as 65100
   neighbor LEAF password DC_FABRIC
   neighbor LEAF timers 3 9
   neighbor LEAF bfd

   neighbor 10.2.1.1 peer group LEAF
   neighbor 10.2.1.1 description LEAF1

   neighbor 10.2.1.3 peer group LEAF
   neighbor 10.2.1.3 description LEAF2

   neighbor 10.2.1.5 peer group LEAF
   neighbor 10.2.1.5 description LEAF3

   redistribute connected route-map REDISTRIBUTE_CONNECTED

   address-family ipv4
      neighbor LEAF activate
      neighbor LEAF route-reflector-client
      neighbor LEAF next-hop-self

end
```
### Spine2
```
route-map REDISTRIBUTE_CONNECTED permit 10
   match interface Loopback0

router bgp 65100
   router-id 10.0.2.0
   no bgp default ipv4-unicast
   maximum-paths ibgp 2

   neighbor LEAF peer group
   neighbor LEAF remote-as 65100
   neighbor LEAF password DC_FABRIC
   neighbor LEAF timers 3 9
   neighbor LEAF bfd

   neighbor 10.2.2.1 peer group LEAF
   neighbor 10.2.2.1 description LEAF1

   neighbor 10.2.2.3 peer group LEAF
   neighbor 10.2.2.3 description LEAF2

   neighbor 10.2.2.5 peer group LEAF
   neighbor 10.2.2.5 description LEAF3

   redistribute connected route-map REDISTRIBUTE_CONNECTED

   address-family ipv4
      neighbor LEAF activate
      neighbor LEAF route-reflector-client
      neighbor LEAF next-hop-self

end
```
### Leaf1
```
route-map REDISTRIBUTE_CONNECTED permit 10
   match interface Loopback0

router bgp 65100
   router-id 10.0.101.0
   no bgp default ipv4-unicast
   maximum-paths 2

   neighbor SPINE peer group
   neighbor SPINE remote-as 65100
   neighbor SPINE password DC_FABRIC
   neighbor SPINE timers 3 9
   neighbor SPINE bfd

   neighbor 10.2.1.0 peer group SPINE
   neighbor 10.2.1.0 description SPINE1

   neighbor 10.2.2.0 peer group SPINE
   neighbor 10.2.2.0 description SPINE2

   redistribute connected route-map REDISTRIBUTE_CONNECTED

   address-family ipv4
      neighbor SPINE activate

end
```
### Leaf2
```
route-map REDISTRIBUTE_CONNECTED permit 10
   match interface Loopback0

router bgp 65100
   router-id 10.0.102.0
   no bgp default ipv4-unicast
   maximum-paths 2

   neighbor SPINE peer group
   neighbor SPINE remote-as 65100
   neighbor SPINE password DC_FABRIC
   neighbor SPINE timers 3 9
   neighbor SPINE bfd

   neighbor 10.2.1.2 peer group SPINE
   neighbor 10.2.1.2 description SPINE1

   neighbor 10.2.2.2 peer group SPINE
   neighbor 10.2.2.2 description SPINE2

   redistribute connected route-map REDISTRIBUTE_CONNECTED

   address-family ipv4
      neighbor SPINE activate

end
```
### Leaf3
```
route-map REDISTRIBUTE_CONNECTED permit 10
   match interface Loopback0

router bgp 65100
   router-id 10.0.103.0
   no bgp default ipv4-unicast
   maximum-paths 2

   neighbor SPINE peer group
   neighbor SPINE remote-as 65100
   neighbor SPINE password DC_FABRIC
   neighbor SPINE timers 3 9
   neighbor SPINE bfd

   neighbor 10.2.1.4 peer group SPINE
   neighbor 10.2.1.4 description SPINE1

   neighbor 10.2.2.4 peer group SPINE
   neighbor 10.2.2.4 description SPINE2

   redistribute connected route-map REDISTRIBUTE_CONNECTED

   address-family ipv4
      neighbor SPINE activate

end
```

### Проверка iBGP на Spine1
3 соседства established:
```
spine1#show ip bgp summary
BGP summary information for VRF default
Router identifier 10.0.1.0, local AS number 65100
Neighbor Status Codes: m - Under maintenance
  Description              Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  LEAF1                    10.2.1.1 4 65100            798       799    0    0 00:04:02 Estab   1      1
  LEAF2                    10.2.1.3 4 65100            754       764    0    0 00:32:00 Estab   1      1
  LEAF3                    10.2.1.5 4 65100            408       410    0    0 00:17:06 Estab   1      1         
```
### Проверка iBGP на Leaf3:
2 маршрута к Leaf1 балансируются по ECMP (established) через Spine1/Spine2. Аналогично, 2 маршрута к Leaf2 балансируются по ECMP (established) через Spine1/Spine2:
```
leaf3#show ip bgp summary
BGP summary information for VRF default
Router identifier 10.0.103.0, local AS number 65100
Neighbor Status Codes: m - Under maintenance
  Description              Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  SPINE1                   10.2.1.4 4 65100            474       472    0    0 00:19:51 Estab   3      3
  SPINE2                   10.2.2.4 4 65100            472       470    0    0 00:19:52 Estab   3      3

leaf3#show ip route bgp

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

 B I      10.0.1.0/32 [200/0]
           via 10.2.1.4, Ethernet1
 B I      10.0.2.0/32 [200/0]
           via 10.2.2.4, Ethernet2
 B I      10.0.101.0/32 [200/0]
           via 10.2.1.4, Ethernet1
           via 10.2.2.4, Ethernet2
 B I      10.0.102.0/32 [200/0]
           via 10.2.1.4, Ethernet1
           via 10.2.2.4, Ethernet2
```
### Проверка достуности Leaf1 и Leaf2 с Leaf3
```
leaf3#ping 10.0.101.0 source 10.0.103.0
PING 10.0.101.0 (10.0.101.0) from 10.0.103.0 : 72(100) bytes of data.
80 bytes from 10.0.101.0: icmp_seq=1 ttl=63 time=3.49 ms
80 bytes from 10.0.101.0: icmp_seq=2 ttl=63 time=1.48 ms
80 bytes from 10.0.101.0: icmp_seq=3 ttl=63 time=1.23 ms
80 bytes from 10.0.101.0: icmp_seq=4 ttl=63 time=1.35 ms
80 bytes from 10.0.101.0: icmp_seq=5 ttl=63 time=1.63 ms
--- 10.0.101.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 13ms
rtt min/avg/max/mdev = 1.230/1.835/3.486/0.836 ms, ipg/ewma 3.204/2.636 ms

leaf3#ping 10.0.102.0 source 10.0.103.0
PING 10.0.102.0 (10.0.102.0) from 10.0.103.0 : 72(100) bytes of data.
80 bytes from 10.0.102.0: icmp_seq=1 ttl=63 time=1.83 ms
80 bytes from 10.0.102.0: icmp_seq=2 ttl=63 time=1.37 ms
80 bytes from 10.0.102.0: icmp_seq=3 ttl=63 time=1.48 ms
80 bytes from 10.0.102.0: icmp_seq=4 ttl=63 time=1.31 ms
80 bytes from 10.0.102.0: icmp_seq=5 ttl=63 time=1.26 ms
--- 10.0.102.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 8ms
rtt min/avg/max/mdev = 1.262/1.450/1.834/0.204 ms, ipg/ewma 2.028/1.632 ms
```
Все доступно
### Проверка отказоустойчивости Leaf1
Отключаем 1 восходящий интерфейс на Leaf1
```
leaf1(config)#interface Ethernet1
leaf1(config-if-Et1)#shutdown
leaf1(config-if-Et1)#ex
```
Проверяем, видим Spine1 перешел в Connect, Spine2 - остался в Established, маршрут доступен через Spine2, 0% packet loss
```
leaf1(config)#show ip bgp summary
BGP summary information for VRF default
Router identifier 10.0.101.0, local AS number 65100
Neighbor Status Codes: m - Under maintenance
  Description              Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  SPINE1                   10.2.1.0 4 65100            932       929    0    0 00:00:12 Connect
  SPINE2                   10.2.2.0 4 65100            992       979    0    0 00:39:09 Estab   3      3
leaf1(config)#show ip route 10.0.102.0

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

 B I      10.0.102.0/32 [200/0]
           via 10.2.2.0, Ethernet2

leaf1(config)#ping 10.0.102.0 source 10.0.101.0
PING 10.0.102.0 (10.0.102.0) from 10.0.101.0 : 72(100) bytes of data.
80 bytes from 10.0.102.0: icmp_seq=1 ttl=63 time=2.19 ms
80 bytes from 10.0.102.0: icmp_seq=2 ttl=63 time=1.43 ms
80 bytes from 10.0.102.0: icmp_seq=3 ttl=63 time=1.80 ms
80 bytes from 10.0.102.0: icmp_seq=4 ttl=63 time=1.06 ms
80 bytes from 10.0.102.0: icmp_seq=5 ttl=63 time=1.10 ms

--- 10.0.102.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 8ms
rtt min/avg/max/mdev = 1.064/1.515/2.194/0.430 ms, ipg/ewma 2.113/1.831 ms
```
После no shut на интерфейсе связность по iBGP восстанавливается, ECMP продолжает работать
### Проверка аутентификации на Leaf1
После установки некорректного пароля и переустановления сессии BGP-соседство Leaf1-Spine1 не устанавливается (Active). Сессия со Spine2 остается Established.
```
leaf1(config-router-bgp)#neighbor SPINE password DUMBASS
leaf1(config-router-bgp)#end
leaf1# clear ip bgp 10.2.1.0
leaf1#show ip bgp summary
BGP summary information for VRF default
Router identifier 10.0.101.0, local AS number 65100
Neighbor Status Codes: m - Under maintenance
  Description              Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  SPINE1                   10.2.1.0 4 65100           1003       999    0    0 00:00:01 Active
  SPINE2                   10.2.2.0 4 65100           1092      1079    0    0 00:43:24 Estab   3      3 
```
Возвращаем верный пароль, 2 сессии продолжают работать в Established
```
leaf1#configure terminal
leaf1(config)#router bgp 65100
leaf1(config-router-bgp)#neighbor SPINE password DC_FABRIC
leaf1(config-router-bgp)#end
leaf1#show ip bgp summary
BGP summary information for VRF default
Router identifier 10.0.101.0, local AS number 65100
Neighbor Status Codes: m - Under maintenance
  Description              Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  SPINE1                   10.2.1.0 4 65100           1012      1006    0    0 00:00:09 Estab   3      3
  SPINE2                   10.2.2.0 4 65100           1133      1120    0    0 00:45:08 Estab   3      3
```
Аутентификация проверена.

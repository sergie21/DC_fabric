# Домашнее задание №2. Построение Underlay с помощью OSPF
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

# Домашнее задание №5. Сервис L2 VNI в VxLAN 
## Топология сети

## Задание:
1. Настроить BGP peering между Leaf и Spine в AF l2VPN EVPN;
2. Настроить связность между клиентами в первой зоне и убедиться в её наличии;
3. Зафиксировать в документации план работы, адресное пространство, схему сети, конфигурацию устройств.

## IP-адресация:
Адресация остается та же, меняются адреса серверов: вместо старых routed /31 серверы переведены в одну overlay-подсеть 10.10.10.10/24.

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
На Spines не настраивается VXLAN, Spines являются транзитными устройствами и RR, а не VTEP. Не применяется next-hop-self: при отражении EVPN-маршрутов next-hop остается адресом VTEP Leaf.

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
Вместо старых routed /31 серверы переведены в одну overlay-подсеть 10.10.10.10/24 (применяется синтаксис сетевого линукса):

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


# Домашнее задание №6. Сервис L3 VNI в VxLAN 
## Топология сети

Рисунок

## Задание:
1. Настроить каждого клиента в своем VNI;
2. Настроить маршрутизацию между клиентами;
3. Зафиксировать в документации план работы, адресное пространство, схему сети, конфигурацию устройств.

## IP-адресация:
2 сервера остались в VLAN 10, 2 сервера переведены в VLAN 20: 

| Сервер  | Leaf / порт     | VLAN | IP-адрес       | Шлюз         |
|---------|-----------------|------|----------------|--------------|
| server1 | Leaf1 Et3       | 10   | 10.10.10.11/24 | 10.10.10.254 |
| server2 | Leaf2 Et3       | 20   | 20.20.20.12/24 | 20.20.20.254 |
| server3 | Leaf3 Et3       | 10   | 10.10.10.13/24 | 10.10.10.254 |
| server4 | Leaf3 Et4       | 20   | 20.20.20.14/24 | 20.20.20.254 |

## Конфигурация Underlay/Overlay

1. Существующая фабрика не перестраивалась, остался OSPF для Underlay, iBGP для Overlay. Spines анонсируют в OSPF Loopback0, Leafs анонсируют в OSPF Loopback0 и Loopback1, Spines являются RR (не в кластере), вся EVPN фабрика в AS 65100.
2. Поверх фабрики созданы:
VRF - TENANT_A.
VLAN10 - 10.10.10.0/24.
L2VNI VLAN10 - 10010.
Anycast Gateway VLAN10 - 10.10.10.254.
VLAN20 - 20.20.20.0/24.
L2VNI VLAN20 - 10020.
Anycast Gateway VLAN20 - 20.20.20.254.
L3VNI TENANT_A - 50001.
Anycast MAC - 0000.0000.9999.
Route Target L3VNI - 50001:50001.
3. Одинаковый Anycast MAC и одинаковые IP-адреса шлюзов настроены на всех VTEP, обслуживающих соответствующий VLAN.

### Leaf1
На Leaf1 создан VRF TENANT_A, интерфейс VLAN10 с шлюзом DGW и привязка VRF к L3VNI 50001.
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
   !
   address-family evpn
      neighbor SPINE activate
   !
   vrf TENANT_A
      rd 10.0.101.0:50001
      route-target import evpn 50001:50001
      route-target export evpn 50001:50001
      redistribute connected

6.
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

На Spines ничего не меняется, Spines продолжают работать как транзитные устройства и RR.

### Серверы
2 cервера остались в overlay-подсети 10.10.10.0/24, 2 cервера переведены в overlay-подсеть 20.20.20.0/24.
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

### Проверка Leaf1
```
leaf1#show vxlan vni
VNI to VLAN Mapping for Vxlan1
VNI         VLAN       Source       Interface       802.1Q Tag
----------- ---------- ------------ --------------- ----------
10010       10         static       Ethernet3       untagged  
                                    Vxlan1          10        

VNI to dynamic VLAN Mapping for Vxlan1
VNI         VLAN       VRF            Source       
----------- ---------- -------------- ------------ 
50001       4097       TENANT_A       evpn         

leaf1#show ip route vrf TENANT_A

VRF: TENANT_A

Gateway of last resort is not set

 C        10.10.10.0/24
           directly connected, Vlan10
 B I      20.20.20.0/24 [200/0]
           via VTEP 10.1.102.0 VNI 50001 router-mac 00:1c:73:9c:96:6b local-interface Vxlan1
```
Вижу соответствие VLAN10 и L2VNI10010; TENANT_A и L3VNI 50001.
Вижу локальную сеть 10.10.10.0/24; удалённую сеть 20.20.20.0/24 через VXLAN.

### Проверка Leaf2
```
leaf2#show vxlan vni
VNI to VLAN Mapping for Vxlan1
VNI         VLAN       Source       Interface       802.1Q Tag
----------- ---------- ------------ --------------- ----------
10020       20         static       Ethernet3       untagged  
                                    Vxlan1          20        

VNI to dynamic VLAN Mapping for Vxlan1
VNI         VLAN       VRF            Source       
----------- ---------- -------------- ------------ 
50001       4097       TENANT_A       evpn         

leaf2#show ip route vrf TENANT_A

VRF: TENANT_A

Gateway of last resort is not set

 B I      10.10.10.11/32 [200/0]
           via VTEP 10.1.101.0 VNI 50001 router-mac 00:1c:73:45:7a:6e local-interface Vxlan1
 B I      10.10.10.13/32 [200/0]
           via VTEP 10.1.103.0 VNI 50001 router-mac 00:1c:73:44:a8:0a local-interface Vxlan1
 B I      10.10.10.0/24 [200/0]
           via VTEP 10.1.101.0 VNI 50001 router-mac 00:1c:73:45:7a:6e local-interface Vxlan1
 B I      20.20.20.14/32 [200/0]
           via VTEP 10.1.103.0 VNI 50001 router-mac 00:1c:73:44:a8:0a local-interface Vxlan1
 C        20.20.20.0/24
           directly connected, Vlan20
```
Вижу соответствие VLAN20 и L2VNI10020; TENANT_A и L3VNI50001.
Вижу локальную сеть 20.20.20.0/24; удалённую сеть 10.10.10.0/24 через VXLAN.

### Проверка Leaf3
```
leaf3#show vxlan vni
VNI to VLAN Mapping for Vxlan1
VNI         VLAN       Source       Interface       802.1Q Tag
----------- ---------- ------------ --------------- ----------
10010       10         static       Ethernet3       untagged  
                                    Vxlan1          10        
10020       20         static       Ethernet4       untagged  
                                    Vxlan1          20        

VNI to dynamic VLAN Mapping for Vxlan1
VNI         VLAN       VRF            Source       
----------- ---------- -------------- ------------ 
50001       4097       TENANT_A       evpn         

leaf3#show ip route vrf TENANT_A

VRF: TENANT_A

Gateway of last resort is not set

 B I      10.10.10.11/32 [200/0]
           via VTEP 10.1.101.0 VNI 50001 router-mac 00:1c:73:45:7a:6e local-interface Vxlan1
 C        10.10.10.0/24
           directly connected, Vlan10
 B I      20.20.20.12/32 [200/0]
           via VTEP 10.1.102.0 VNI 50001 router-mac 00:1c:73:9c:96:6b local-interface Vxlan1
 C        20.20.20.0/24
           directly connected, Vlan20
```
Вижу соответствие VLAN10 и L2VNI10010; соответствие VLAN20 и L2VNI10020; соответствие TENANT_A и L3VNI50001.
Вижу обе подключённые сети: 10.10.10.0/24 и 20.20.20.0/24.

### Проверка серверов
Проверяем передачу L2-трафика через VXLAN без маршрутизации. Server1 и server3 находятся в одном VLAN10, но подключены к разным Leaf. Все ОК:
```
docker exec clab-fabric-server1 ping -c 3 10.10.10.13
PING 10.10.10.13 (10.10.10.13): 56 data bytes
64 bytes from 10.10.10.13: seq=0 ttl=64 time=11.275 ms
64 bytes from 10.10.10.13: seq=1 ttl=64 time=7.302 ms
64 bytes from 10.10.10.13: seq=2 ttl=64 time=8.742 ms

--- 10.10.10.13 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 7.302/9.106/11.275 ms
```
Проверяем локальную inter-VLAN маршрутизацию на одном VTEP. Server3 находится в VLAN10, server4 - в VLAN20. Оба подключены к Leaf3. Все ОК:
```
docker exec clab-fabric-server3 ping -c 3 20.20.20.14
PING 20.20.20.14 (20.20.20.14): 56 data bytes
64 bytes from 20.20.20.14: seq=0 ttl=63 time=10.830 ms
64 bytes from 20.20.20.14: seq=1 ttl=63 time=3.492 ms
64 bytes from 20.20.20.14: seq=2 ttl=63 time=3.192 ms
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 3.192/5.838/10.830 ms
```
Проверяем маршрутизацию между VLAN10 и VLAN20 через разные VTEP. Server1 подключён к Leaf1, server2 - к Leaf2. Все ОК:
```
 docker exec clab-fabric-server1 ping -c 3 20.20.20.12
PING 20.20.20.12 (20.20.20.12): 56 data bytes
64 bytes from 20.20.20.12: seq=0 ttl=62 time=13.037 ms
64 bytes from 20.20.20.12: seq=1 ttl=62 time=7.251 ms
64 bytes from 20.20.20.12: seq=2 ttl=62 time=7.379 ms

--- 20.20.20.12 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 7.251/9.222/13.037 ms
```

## Выводы:

Реализованы VLAN 10/VNI 10010 и VLAN 20/VNI 10020 в VRF TENANT_A. Каждый клиентский сегмент размещён в отдельном L2VNI.
Маршрутизация между подсетями выполняется через Distributed Anycast Gateway и L3VNI 50001. 
EVPN Type-2/3 обеспечивает L2-связность, Type-5 обеспечивает распространение IP-префиксов между VTEP.

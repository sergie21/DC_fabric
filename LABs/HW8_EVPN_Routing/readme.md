# Домашнее задание №8. Оптимизация таблиц маршрутизации.

## Топология сети

За основу взял свою фабрику: 2 Spine, 3 Leaf, 4 сервера, ESI-LAG к Leaf1/Leaf2 для server1. Добавил внешний маршрутизатор border-fw на FRR.
<img width="888" height="796" alt="VXLAN Routing1" src="https://github.com/user-attachments/assets/ab6859fb-565a-46c6-94d6-25deaa142f45" />

## Задание

1. Разместить двух клиентов в разных VRF в одной фабрике;
2. Настроить маршрутизацию между клиентами через внешнее устройство (маршрутизатор/фаервол);
3. Зафиксировать план, адресное пространство, схему и конфигурации оборудования.

## IP-адресация

| Сервер | Leaf / порт | VRF | VLAN | L2VNI | IP-адрес | Шлюз |
| --- | --- | --- | --- | --- | --- | --- |
| server1 | Leaf1 Et3 + Leaf2 Et4 | TENANT_A | 10 | 10010 | 10.10.10.11/24 (bond0) | 10.10.10.254 |
| server2 | Leaf2 Et3 | TENANT_A | 20 | 10020 | 20.20.20.12/24 | 20.20.20.254 |
| server3 | Leaf3 Et3 | TENANT_B | 30 | 10030 | 30.30.30.13/24 | 30.30.30.254 |
| server4 | Leaf3 Et4 | TENANT_B | 40 | 10040 | 30.30.31.14/24 | 30.30.31.254 |

| VRF | L3VNI | Route Target | Подсети |
| --- | --- | --- | --- |
| TENANT_A | 50001 | 50001:50001 | 10.10.10.0/24, 20.20.20.0/24 |
| TENANT_B | 50002 | 50002:50002 | 30.30.30.0/24, 30.30.31.0/24 |

Подключение FRR к Leaf3:

| Устройство | Интерфейс | VRF на Leaf3 | IP-адрес | eBGP AS |
| --- | --- | --- | --- | --- |
| Leaf3 | Ethernet5 | TENANT_A | 172.16.10.1/30 | 65100 |
| FRR | eth1 | — | 172.16.10.2/30 | 65000 |
| Leaf3 | Ethernet6 | TENANT_B | 172.16.20.1/30 | 65100 |
| FRR | eth2 | — | 172.16.20.2/30 | 65000 |

FRR работает с одной глобальной таблицей маршрутизации Linux. На Leaf3 интерфейсы находятся в разных VRF. Между VRF прямого route leaking не настраивал, трафик уходит на FRR и там разруливается.

## Конфигурация Underlay/Overlay

1. Существующую фабрику сохранил: OSPF area 0 для Underlay, iBGP EVPN для Overlay, Spine - RR, AS 65100;
2. На Leaf1 и Leaf2 сохранил TENANT_A и ESI-LAG для server1 из ДЗ №7;
3. На Leaf3 создал TENANT_B, VLAN30/40, L2VNI 10030/10040, L3VNI 50002 и Anycast Gateway для обеих подсетей;
4. Подключил FRR к Leaf3 двумя "физическими" L3-линками и сделал eBGP для каждой VRF;
5. FRR передаёт default route 0.0.0.0/0 в обе VRF. Leaf3 распространяет его по EVPN как Route-Type 5;
6. На Leaf3 в TENANT_B настроил агрегацию двух /24 с VLANs в 30.30.30.0/23 с summary-only.

### Leaf3 - VRF, VLAN, VXLAN

Изменения относительно ДЗ №7:

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

VLAN10/20 и L3VNI50001 для TENANT_A на Leaf3 оставил. Клиентов VLAN10/20 на Leaf3 теперь нет - server3/server4 переведены в VLAN 30/40 TENANT_B.

### Leaf3 - подключение FRR и BGP

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

Для TENANT_A и TENANT_B используются разные RT. Поэтому Leaf1 видит маршруты обеих VRF в общей EVPN BGP-таблице, но импортирует в TENANT_A только маршруты с соответствующим RT.

### Border-FW - FRR, AS 65000

FRR работает в Linux-контейнере. Настройки интерфейсов и IP forwarding:

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

IMPORT_FROM_LEAF - разрешает приём маршрутов от Leaf3, без фильтрации конкретных префиксов.  
DEFAULT_ONLY - prefix-list, совпадающий только с дефлотом 0.0.0.0/0.  
EXPORT_DEFAULT - разрешает анонс FRR в сторону Leaf3 только дефолта.  
ip route 0.0.0.0/0 Null0- создаёт маршрут для BGP "network 0.0.0.0/0". Это лабораторная "заглушка", а не реальный выход в интернет.  

FRR получает от Leaf3 четыре пользовательские подсети /24. Агрегация /23 настроена отдельно в BGP VRF TENANT_B на Leaf3 для EVPN RT-5, а не как замена всех маршрутов на FRR.

### Server3 и Server4

Перевёл оба сервера из TENANT_A в TENANT_B. Адреса и default route:

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

Для server1 сохранил bond0 (LACP 802.3ad) и установил дефолтныый маршрут через 10.10.10.254, чтобы меж-VRF-ный трафик не уходил через паразитный management eth0. Для server2 - default route через 20.20.20.254.

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

На FRR оба соседа с Leaf3 поднялись.

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

Для TENANT_B использовал две смежные подсети:

```text
30.30.30.0/24  (VLAN30) и 
30.30.31.0/24  (VLAN40) =
30.30.30.0/23
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

В выводе на Leaf1 суммарный 30.30.30.0/23 получен через Spine1 и Spine2 (RR). Отдельных RT-5 для 30.30.30.0/24 и 30.30.31.0/24 больше нет (тк summary-only). Default route для обеих VRF продолжает распространяться как RT-5.

### Проверка меж-VRF-ной маршрутизации

Проверяю с server1 (TENANT_A) доступность обоих серверов TENANT_B:

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

## Выводы

Настроил 2 изолированных VRF - TENANT_A и TENANT_B. Между ними организовал маршрутизацию через внешний FRR по двум eBGP-сессиям. FRR анонсирует дефолт в обе VRF, Leaf3 распространяет его через EVPN RT-5. Для TENANT_B настроил суммаризацию двух /24 в 30.30.30.0/23 с summary-only. На Leaf1 полцчил только агрегированный саммари-префикс. Проверил связность server1 с server3 и server4 после суммаризации.

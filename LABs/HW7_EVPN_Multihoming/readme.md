# Домашнее задание №7. EVPN Multihoming (ESI-LAG)

## Топология сети

Для этой работы server1 подключён двумя линками к Leaf1 и Leaf2. Остальные подключения оставлены без изменений.

## Задания:

1. Подключить клиента двумя линками к разным Leaf;
2. Настроить агрегированный канал на клиенте;
3. Настроить ESI-LAG для VXLAN EVPN;
4. Проверить связность, в том числе при отключении одного линка;
5. Зафиксировать адресацию и конфигурации.

## IP-адресация:

| Сервер | Leaf / порт | VLAN | IP-адрес | Шлюз |
| --- | --- | --- | --- | --- |
| server1 | Leaf1 Et3 + Leaf2 Et4 | 10 | 10.10.10.11/24 (bond0) | 10.10.10.254 |
| server2 | Leaf2 Et3 | 20 | 20.20.20.12/24 | 20.20.20.254 |
| server3 | Leaf3 Et3 | 10 | 10.10.10.13/24 | 10.10.10.254 |
| server4 | Leaf3 Et4 | 20 | 20.20.20.14/24 | 20.20.20.254 |

VLAN10 - VNI10010; VLAN20 - VNI10020; VRF TENANT_A - L3VNI50001. Anycast MAC - 0000.0000.9999.

## Конфигурация Underlay/Overlay

1. Существующую фабрику не перестраивал. Underlay - OSPF, Overlay - iBGP EVPN, оба Spine - RR, AS 65100;
2. В Containerlab добавил второй линк server1 к Leaf2. На сервере eth1 и eth2 объединяются в bond0 по LACP;
3. На Leaf1 и Leaf2 настроил Port-Channel10, одинаковый ESI и LACP System ID. Режим Multihoming - all-active;
4. На Leaf2 добавил VLAN10/VNI10010, BGP EVPN для VLAN10 и Anycast Gateway 10.10.10.254. Ранее Leaf2 обслуживал только VLAN20.

### Подключение server1

Фрагмент изменений в `fabric.clab.yml`:

```yaml
    - endpoints: ["leaf1:eth3", "server1:eth1"]
    - endpoints: ["leaf2:eth4", "server1:eth2"]
```

### Leaf1

Ethernet3 переведён в Port-Channel10. В Ethernet Segment задан ESI; LACP System ID на обоих Leaf одинаковый.

```text
interface Ethernet3
   description SERVER1_ESI_LAG
   channel-group 10 mode active
!
interface Port-Channel10
   description SERVER1_ESI_LAG
   switchport access vlan 10
   !
   evpn ethernet-segment
      identifier 0000:0000:0000:0000:0010
   lacp system-id 1111.2222.0010
!
interface Vlan10
   description GATEWAY_VLAN10
   vrf TENANT_A
   ip address virtual 10.10.10.254/24
```

### Leaf2

На Leaf2 Ethernet4 включён в Port-Channel10 с тем же ESI и LACP System ID. Добавлены недостающие настройки VLAN10 для Overlay.

```text
interface Ethernet4
   description SERVER1_ESI_LAG
   channel-group 10 mode active
!
interface Port-Channel10
   description SERVER1_ESI_LAG
   switchport access vlan 10
   !
   evpn ethernet-segment
      identifier 0000:0000:0000:0000:0010
   lacp system-id 1111.2222.0010
!
interface Vlan10
   description GATEWAY_VLAN10
   vrf TENANT_A
   ip address virtual 10.10.10.254/24
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vlan 20 vni 10020
   vxlan vrf TENANT_A vni 50001
!
router bgp 65100
   vlan 10
      rd auto
      route-target both 10010:10010
      redistribute learned
```

По `show running-config all interfaces Port-Channel10` вижу `redundancy all-active`; в обычном running-config команда не выводится, т.к. используется значение по умолчанию.

На Spine ничего не менял.

### Server1

В Alpine изначально использовался BusyBox `ip`. После deploy bond0 создавался в `balance-rr`, Leaf ожидали LACP. Установил `iproute2` и вынес установку режима 802.3ad в отдельную команду. Текущая секция `exec` в `fabric.clab.yml`:

```yaml
    server1:
      kind: linux
      image: alpine:latest
      mgmt-ipv4: 172.20.20.7
      exec:
        - apk add --no-cache iproute2
        - ip link add bond0 type bond
        - ip link set bond0 type bond mode 802.3ad miimon 100 lacp_rate fast
        - ip link set eth1 down
        - ip link set eth2 down
        - ip link set eth1 master bond0
        - ip link set eth2 master bond0
        - ip link set eth1 up
        - ip link set eth2 up
        - ip link set bond0 up
        - ip addr add 10.10.10.11/24 dev bond0
        - ip route replace 20.20.20.0/24 via 10.10.10.254 dev bond0
```

Маршрут в VLAN20 добавил явно, чтобы тесты не уходили через management `eth0` и Docker default gateway.

## Проверка

### Underlay/Overlay

После штатного `./start-fabric.sh` на Leaf1 два OSPF-соседа в FULL, BGP EVPN до Spine1 и Spine2 в Established.

```text
leaf1#show ip ospf neighbor
10.0.2.0   FULL   Ethernet2
10.0.1.0   FULL   Ethernet1

leaf1#show bgp evpn summary
SPINE1  10.0.1.0  Estab
SPINE2  10.0.2.0  Estab
```

На Leaf2 после добавления VLAN10 вижу EVPN Auto-Discovery для общего ESI с Leaf1, локальный MAC server1 (Type 2), IMET (Type 3) и маршруты префиксов (Type 5).

### LACP

Проверяю агрегатор после полного пересоздания фабрики, без ручной настройки server1:

```bash
docker exec clab-fabric-server1 cat /proc/net/bonding/bond0
```

```text
Bonding Mode: IEEE 802.3ad Dynamic link aggregation
MII Status: up
LACP active: on
LACP rate: fast

Active Aggregator Info:
        Aggregator ID: 2
        Number of ports: 2
        Partner Key: 10
        Partner Mac Address: 11:11:22:22:00:10

Slave Interface: eth1
MII Status: up
Aggregator ID: 2

Slave Interface: eth2
MII Status: up
Aggregator ID: 2
```

Оба линка в одном агрегаторе, Leaf1 и Leaf2 видны серверу как один LACP-партнёр. На Leaf `show port-channel` показывал активные Ethernet3 и Ethernet4 соответственно.

### Маршрутизация

```bash
docker exec clab-fabric-server1 ip route get 20.20.20.12
```

```text
20.20.20.12 via 10.10.10.254 dev bond0 src 10.10.10.11
```

Трафик в VLAN20 идёт через bond0, а не через сеть управления.

### Проверка серверов

Проверяю с server1 через bond0 после повторного deploy:

```bash
docker exec clab-fabric-server1 ping -c 5 -I bond0 10.10.10.254
docker exec clab-fabric-server1 ping -c 5 -I bond0 20.20.20.12
docker exec clab-fabric-server1 ping -c 5 -I bond0 10.10.10.13
```

Результаты:

```text
server1 -> 10.10.10.254 (Anycast GW): 5/5, loss 0%, avg 3.189 ms
server1 -> 20.20.20.12 (VLAN20):   5/5, loss 0%, avg 3.647 ms
server1 -> 10.10.10.13 (VLAN10):   5/5, loss 0%, avg 7.739 ms
```

L2-связность с server3 и маршрутизация к server2 работают.

### Проверка отказоустойчивости

По очереди отключал на server1 один из линков, оставляя второй активным.

```bash
ip link set eth2 down
ping -c 5 -I bond0 10.10.10.254
ping -c 5 -I bond0 20.20.20.12
ip link set eth2 up

ip link set eth1 down
ping -c 10 -I bond0 10.10.10.254
ping -c 10 -I bond0 20.20.20.12
ip link set eth1 up
```

При отключённом eth2 (через Leaf1) оба ping — 0% потерь, 5/5. При отключённом eth1 (через Leaf2) — также 0%, 10/10. Bond остаётся UP с одним портом. После восстановления обоих линков снова 2 порта в активном агрегаторе.

Это проверка связности после отказа. Потери непосредственно в момент переключения не измерял.

## Проблемы при настройке

1. После запуска cEOS через `docker start` контейнеры были running, но veth-линки не все восстановились. OSPF/BGP не работали. Перешёл на штатный запуск через Containerlab.
2. В Alpine команда `ip` была из BusyBox. Bond создавался в `balance-rr`, на обоих Leaf было `waiting for LACP response`. После установки `iproute2` и отдельной настройки режима `802.3ad` LACP поднялся.
3. На Leaf2 сначала не было VNI10010 и шлюза VLAN10. После добавления соответствующего L2VNI, BGP EVPN VLAN10 и Anycast Gateway трафик через оба линка заработал.
4. Изначально ping до VLAN20 уходил через management `eth0`, поскольку не было маршрута через bond0. Добавил маршрут в секцию `exec`.

## Выводы:

Настроил ESI-LAG в режиме all-active между Leaf1 и Leaf2, клиентский bond0 работает по LACP 802.3ad. Проверил EVPN маршруты, L2-связность в VLAN10 и маршрутизацию в VLAN20. При отключении любого из двух клиентских линков связность через оставшийся сохраняется. После исправления `fabric.clab.yml` лаборатория поднимается через `./start-fabric.sh` без ручной перенастройки server1.

# Миграция сервисов из классической сети в VXLAN EVPN Fabric с готовностью к подключению облачной инфраструктуры

> Дипломный проект курса OTUS «Дизайн сетей ЦОД». Лабораторная реализация сетевой миграции, сегментации, отказоустойчивости и внешнего BGP-стыка.

## Кратко о проекте

В проекте выполнена **поэтапная сетевая миграция четырёх серверных сегментов** из классической сети (Legacy) в VXLAN EVPN Fabric на базе Arista cEOS. Проверены маршрутизация внутри VRF, внешний обмен маршрутами, EVPN Multihoming, устойчивость к отказам Spine/Leaf и подключение к эмулятору облачного маршрутизатора.

**Особенность стенда:** целевая Fabric не строилась с нуля специально для диплома. Её Underlay, Overlay, VLAN/VNI, VRF и EVPN Multihoming были настроены во время предшествующих лабораторных работ OTUS. В дипломном проекте эта Fabric использована как **подготовленная целевая инфраструктура**. Совпадение VLAN ID в Legacy и Fabric не означает одновременную работу дублированных серверов или шлюзов.

В проекте моделируется **перенос сетевых подключений и функций шлюзов с сохранением IP-адресации**, а не перенос данных, приложений или виртуальных машин.

### Что реализовано

- Перенесены VLAN 10, 20, 30, 40 из Legacy в целевую Fabric.
- Используются два VRF: `TENANT_A` и `TENANT_B`, L2VNI и L3VNI.
- Server1 подключён через EVPN Multihoming / ESI-LAG (`all-active`, LACP).
- Через пограничный маршрутизатор FRRouting организована маршрутизация между tenant-сегментами.
- Выполнены испытания отказов Spine1, отдельного uplink и сетевой изоляции Leaf1.
- Добавлен Cloud-GW (AS 65200) и проверена связность с тестовым облачным префиксом `10.200.0.0/24` в обоих направлениях.

> **Границы реализации:** `border-fw` — это FRRouting-маршрутизатор, а не проверенный межсетевой экран с правилами фильтрации. Cloud-GW имитирует облачный маршрутизатор: реальная VPC, VPN/Direct Connect и отдельная облачная VM не разворачивались.

---

## 1. Лабораторная среда

Лаборатория работает локально на ноутбуке:

`Windows → WSL2 → Ubuntu → Docker → Containerlab → контейнеры`.

Для просмотра схемы и работы с терминалами используется **Containerlab Desktop (CLDesktop)**. Конфигурация устройств Arista выполнялась через их терминалы в CLDesktop; Ubuntu/WSL используется для запуска Containerlab, проверки контейнеров и работы с файлами.

| Компонент | Назначение |
|---|---|
| Windows, WSL2, Ubuntu | Хостовая и Linux-среда |
| Docker, Containerlab | Развёртывание лаборатории из `fabric.clab.yml` |
| Arista cEOS 4.33.2F | 2 Spine, 3 Leaf, 3 Legacy-сетевых устройства |
| Alpine Linux | 4 целевых и 4 Legacy-сервера |
| FRRouting 10.4.1 | `border-fw` и `cloud-gw` |
| `172.20.20.0/24` | Отдельная сеть управления Containerlab |

**Итого: 18 контейнеров** (8 cEOS + 8 Alpine + 2 FRRouting).

<!-- TODO: вставить схему слоёв Windows / WSL2 / Ubuntu / Docker / Containerlab / mgmt -->

<details>
<summary><strong>Развёртывание и проверка состава стенда</strong></summary>

```bash
cd ~/labs/fabric-project
# Перед destroy убедиться, что актуальные конфигурации сохранены в configs/.
sudo containerlab destroy -t fabric.clab.yml
sudo containerlab deploy -t fabric.clab.yml
```

> Это штатная процедура **полного пересоздания** стенда, а не команда для проверки. Для cEOS не используйте `docker stop/start` как способ штатного восстановления отказавшего узла: в испытании Spine1 после `docker start` не вернулись data-plane-интерфейсы.

</details>

## 2. Исходная и целевая архитектуры

### Legacy: исходная сеть

`legacy-r1`, `legacy-sw1`, `legacy-sw2`, `legacy-s1`–`legacy-s4`. Миграционные серверы моделируют четыре исходных сегмента.

| VLAN | Сеть | Исходный сервер |
|---|---|---|
| 10 | `10.10.10.0/24` | `legacy-s1` |
| 20 | `20.20.20.0/24` | `legacy-s2` |
| 30 | `30.30.30.0/24` | `legacy-s3` |
| 40 | `30.30.31.0/24` | `legacy-s4` |

<!-- TODO: схема Legacy до переноса -->

### VXLAN EVPN Fabric: целевая сеть

Двухуровневая топология Clos: **2 Spine + 3 Leaf**. Все Leaf подключены к обоим Spine. `spine1` и `spine2` работают как BGP EVPN Route Reflector; `leaf1`–`leaf3` — VTEP.

- **Underlay:** OSPF area 0, point-to-point /31, Loopback0.
- **Overlay:** iBGP EVPN, AS `65100`, VXLAN; VTEP Loopback1.
- **Tenant-маршрутизация:** VRF/L3VNI и Anycast Gateway.

| Устройство | Loopback0 | VTEP Loopback1 |
|---|---|---|
| spine1 | `10.0.1.0` | — |
| spine2 | `10.0.2.0` | — |
| leaf1 | `10.0.101.0` | `10.1.101.0` |
| leaf2 | `10.0.102.0` | `10.1.102.0` |
| leaf3 | `10.0.103.0` | `10.1.103.0` |

<details>
<summary><strong>Физические подключения Spine–Leaf (из fabric.clab.yml)</strong></summary>

| Leaf, порт | Spine, порт |
|---|---|
| leaf1 eth1 | spine1 eth1 |
| leaf1 eth2 | spine2 eth1 |
| leaf2 eth1 | spine1 eth2 |
| leaf2 eth2 | spine2 eth2 |
| leaf3 eth1 | spine1 eth3 |
| leaf3 eth2 | spine2 eth3 |

</details>

<!-- TODO: итоговая схема Fabric с верными связями и портами; Legacy отдельно -->

### Сегментация

| VRF | VLAN → L2VNI | L3VNI |
|---|---|---|
| `TENANT_A` | 10 → 10010; 20 → 10020 | 50001 |
| `TENANT_B` | 30 → 10030; 40 → 10040 | 50002 |

| Сервер | VLAN / VRF | IP | Anycast Gateway | Подключение |
|---|---|---|---|---|
| `server1` | 10 / TENANT_A | `10.10.10.11/24` | `10.10.10.254` | leaf1 eth3 + leaf2 eth4 |
| `server2` | 20 / TENANT_A | `20.20.20.12/24` | `20.20.20.254` | leaf2 eth3 |
| `server3` | 30 / TENANT_B | `30.30.30.13/24` | `30.30.30.254` | leaf3 eth3 |
| `server4` | 40 / TENANT_B | `30.30.31.14/24` | `30.30.31.254` | leaf3 eth4 |

**Подробная настройка Underlay, EVPN, VLAN/VNI и VRF** выполнена ранее на курсе; вместо дублирования десятков страниц CLI здесь уместно дать ссылки на соответствующие работы из [репозитория DC_fabric](https://github.com/sergie21/DC_fabric). Полные актуальные конфигурации дипломного стенда хранятся в `configs/`.

## 3. Подготовка и выполнение миграции

### Временный L2-стык

Для этапов переключения предусмотрено соединение:

`legacy-sw1 eth5 ↔ leaf3 eth7`.

Временный trunk использовался как миграционный L2-стык; после переноса серверов Legacy не участвует в обслуживании мигрированных сервисов. **Физическое соединение по-прежнему описано в YAML:** отключение Legacy от сервисной передачи не тождественно удалению контейнеров и линка из стенда.

<!-- TODO: переходная схема Legacy → Fabric, выделить eth5 ↔ eth7 -->

<details>
<summary><strong>Где смотреть конфигурацию trunk и исходной Legacy-сети</strong></summary>

- [`configs/legacy-sw1.cfg`](configs/legacy-sw1.cfg)
- [`configs/leaf3.cfg`](configs/leaf3.cfg)
- [`configs/legacy-r1.cfg`](configs/legacy-r1.cfg)
- [`fabric.clab.yml`](fabric.clab.yml)

> Полные блоки CLI сюда намеренно не копируются: их необходимо брать из итоговых конфигурационных файлов, а не восстанавливать по памяти.

</details>

### Перенос VLAN 10–40

| Шаг | Что переносилось | Целевое подключение |
|---|---|---|
| VLAN10 | `legacy-s1` → `server1`, сохранён `10.10.10.11` | ESI-LAG к leaf1/leaf2 |
| VLAN20 | `legacy-s2` → `server2`, сохранён `20.20.20.12` | leaf2 eth3 |
| VLAN30 | `legacy-s3` → `server3`, сохранён `30.30.30.13` | leaf3 eth3 |
| VLAN40 | `legacy-s4` → `server4`, сохранён `30.30.31.14` | leaf3 eth4 |

**VLAN10 — подтверждённый пример переключения:** исходный `legacy-s1` изолирован; отключены `legacy-sw1 Ethernet3` и шлюз на `legacy-r1 Ethernet1.10`. `server1` подключён к Fabric с прежним IP; используется Anycast Gateway `10.10.10.254`. Проверялись EVPN Type-2 маршруты.

Остальные сегменты перенесены в целевую Fabric; детальные пошаговые команды VLAN20–40 необходимо брать из журналов соответствующих этапов, а не реконструировать как дословный протокол выполнения.

<details>
<summary><strong>Конфигурационные файлы и инициализация серверов</strong></summary>

- [`configs/server1-init.sh`](configs/server1-init.sh) — создание bond0 на Server1.
- [`fabric.clab.yml`](fabric.clab.yml) — физические подключения и инициализация Server2–Server4.
- [`configs/legacy-r1.cfg`](configs/legacy-r1.cfg) — исходные шлюзы Legacy.
- [`configs/legacy-sw1.cfg`](configs/legacy-sw1.cfg) — миграционный коммутатор.

</details>

## 4. EVPN Multihoming (ESI-LAG)

`server1` объединяет `eth1` (leaf1) и `eth2` (leaf2) в `bond0` по IEEE 802.3ad. На Leaf используется `Port-Channel10`, VLAN10, ESI `0000:0000:0000:0000:0010`, LACP System ID `1111.2222.0010`, режим **all-active**.

**Результат:** оба физических подключения входят в один LACP-агрегатор. В предыдущей лабораторной работе проверена связность через каждый Leaf при отказе другого подключения. Подробности: [HW7 — EVPN Multihoming](https://github.com/sergie21/DC_fabric/tree/main/LABs/HW7_EVPN_Multihoming).

<details>
<summary><strong>Проверка LACP на Server1</strong></summary>

```bash
cat /proc/net/bonding/bond0
```

Подтверждённый результат после восстановления Leaf1:

```text
Bonding Mode: IEEE 802.3ad Dynamic link aggregation
Transmit Hash Policy: layer2 (0)
MII Status: up
Active Aggregator Info:
    Aggregator ID: 2
    Number of ports: 2
    Partner Mac Address: 11:11:22:22:00:10
Slave Interface: eth1
MII Status: up
Aggregator ID: 2
Slave Interface: eth2
MII Status: up
Aggregator ID: 2
```

Это сокращённая выдержка из фактического вывода, не полный CLI-листинг.

</details>

## 5. Внешняя маршрутизация между TENANT_A и TENANT_B

`leaf3` подключён к `border-fw` (FRRouting, AS 65000) двумя физическими L3-соединениями:

| Назначение | Leaf3 | Border-FW | Сеть |
|---|---|---|---|
| TENANT_A | Ethernet5 (`172.16.10.1`) | eth1 (`172.16.10.2`) | `172.16.10.0/30` |
| TENANT_B | Ethernet6 (`172.16.20.1`) | eth2 (`172.16.20.2`) | `172.16.20.0/30` |

Соседства eBGP между Fabric (AS 65100) и Border-FW (AS 65000) установлены. Border-FW получает tenant-префиксы и распространяет маршрут по умолчанию. МежVRF-связность через внешний узел проверялась ранее.

**Ограничение:** оба tenant-направления сходятся в общей таблице маршрутизации `default` на Border-FW; фильтрация доступа не реализована и не должна заявляться как проверенная.

<details>
<summary><strong>Основная логика BGP на Border-FW — конфигурация этапа 5</strong></summary>

```frr
ip prefix-list DEFAULT_ONLY seq 10 permit 0.0.0.0/0
route-map EXPORT_DEFAULT permit 10
 match ip address prefix-list DEFAULT_ONLY
exit
ip route 0.0.0.0/0 Null0
router bgp 65000
 bgp router-id 10.255.255.1
 neighbor 172.16.10.1 remote-as 65100
 neighbor 172.16.20.1 remote-as 65100
 address-family ipv4 unicast
  network 0.0.0.0/0
  neighbor 172.16.10.1 route-map EXPORT_DEFAULT out
  neighbor 172.16.20.1 route-map EXPORT_DEFAULT out
 exit-address-family
```

*Фрагмент отражает исходную логику до добавления Cloud-GW; **актуальный** файл после этапа 7 включает дополнительные route-map: [`configs/border-fw/frr.conf`](configs/border-fw/frr.conf).*

</details>

## 6. Испытания отказоустойчивости

| Сценарий | Зафиксированный результат |
|---|---|
| Остановка Spine1 | 396/396 ICMP, потери **0%**; после восстановления — OSPF FULL, BGP EVPN Established |
| `shutdown` Spine1 Ethernet1 | 463/463 ICMP, потери **0%**; после `no shutdown` — 3 OSPF FULL |
| Изоляция Leaf1 (`Ethernet1–3 shutdown`) | 521 отправлено, 518 получено: **3 пакета / 0,58%**; RTT max **2050,014 мс** |
| Возврат Leaf1 | OSPF FULL, BGP EVPN Established |
| Восстановление ESI-LAG | `bond0`: оба участника UP, Aggregator ID 2 |

При изоляции Leaf1 Server1 сохранил рабочее подключение через Leaf2. Максимальный RTT во время испытания **не равен измеренному времени сходимости**. Для теста со Spine1 используется результат сохранения связности при отказе узла; факт переключения конкретного потока с ранее активного пути отдельно не измерялся.

<details>
<summary><strong>Команды и свидетельства испытаний</strong></summary>

```text
# Постоянная проверка со стороны server1:
ping -i 0.2 20.20.20.12

# Имитация потери подключений leaf1 (через терминал Arista в CLDesktop):
configure terminal
interface Ethernet1-3
shutdown

# Восстановление:
configure terminal
interface Ethernet1-3
no shutdown

# Проверка underlay/overlay:
show ip ospf neighbor
show bgp evpn summary

# Проверка ESI-LAG с server1:
cat /proc/net/bonding/bond0
```

> Блок содержит команды из проведённого испытания. Не применять `shutdown` в работающей сети вне контролируемого теста.

</details>

## 7. Готовность к подключению облачной инфраструктуры

Для моделирования внешней сети добавлен `cloud-gw` (FRRouting, **AS 65200**), подключённый к Border-FW (**AS 65000**) через выделенный data-plane линк:

`border-fw eth3 (172.16.30.1/30) ↔ cloud-gw eth1 (172.16.30.2/30)`.

На Cloud-GW создан интерфейс `lo` с `10.200.0.1/24` и анонсом `10.200.0.0/24`. Внешний префикс получен Border-FW по eBGP и анонсирован в направлении `TENANT_A`. На leaf3 подтверждена запись:

```text
VRF: TENANT_A
B E 10.200.0.0/24 [200/0]
    via 172.16.10.2, Ethernet5
```

### Проверки связности

| Направление | Команда | Результат |
|---|---|---|
| Server1 → Cloud-GW | `ping -c 10 10.200.0.1` | **10/10, 0% потерь**, средний RTT 9,724 мс |
| Cloud-GW → Server1 | `ping -I 10.200.0.1 -c 10 10.10.10.11` | **10/10, 0% потерь**, средний RTT 7,387 мс |

Для Cloud-GW это **проверка маршрутизатора**, а не отдельной облачной VM. Из-за назначения `10.200.0.1/24` на `lo` Linux локально обрабатывает и другие адреса этого диапазона (например, `10.200.0.11`); такой ответ **не подтверждает существование отдельного облачного сервера**.

<details>
<summary><strong>Cloud-GW — конфигурация BGP и политики импорта/экспорта</strong></summary>

```frr
frr version 10.4.1_git
frr defaults traditional
hostname cloud-gw
!
ip prefix-list CLOUD_NET seq 10 permit 10.200.0.0/24
ip prefix-list TENANT_A_NETS seq 10 permit 10.10.10.0/24
ip prefix-list TENANT_A_NETS seq 20 permit 20.20.20.0/24
!
route-map EXPORT_TO_BORDER permit 10
 match ip address prefix-list CLOUD_NET
exit
!
route-map IMPORT_FROM_BORDER permit 10
 match ip address prefix-list TENANT_A_NETS
exit
!
router bgp 65200
 bgp router-id 10.200.0.1
 neighbor 172.16.30.1 remote-as 65000
 address-family ipv4 unicast
  network 10.200.0.0/24
  neighbor 172.16.30.1 route-map IMPORT_FROM_BORDER in
  neighbor 172.16.30.1 route-map EXPORT_TO_BORDER out
 exit-address-family
exit
```

Исходный файл: [`configs/cloud-gw/frr.conf`](configs/cloud-gw/frr.conf).

</details>

<details>
<summary><strong>Border-FW — изменения для Cloud-GW и маршрутные политики</strong></summary>

В дополнение к исходным BGP-соседствам:

```frr
ip prefix-list CLOUD_NET seq 10 permit 10.200.0.0/24
ip prefix-list TENANT_A_NETS seq 10 permit 10.10.10.0/24
ip prefix-list TENANT_A_NETS seq 20 permit 20.20.20.0/24
!
route-map EXPORT_TO_TENANT_A permit 10
 match ip address prefix-list DEFAULT_ONLY
exit
route-map EXPORT_TO_TENANT_A permit 20
 match ip address prefix-list CLOUD_NET
exit
route-map EXPORT_TO_CLOUD permit 10
 match ip address prefix-list TENANT_A_NETS
exit
route-map IMPORT_FROM_CLOUD permit 10
 match ip address prefix-list CLOUD_NET
exit
!
router bgp 65000
 neighbor 172.16.30.2 remote-as 65200
 address-family ipv4 unicast
  neighbor 172.16.10.1 route-map EXPORT_TO_TENANT_A out
  neighbor 172.16.30.2 route-map IMPORT_FROM_CLOUD in
  neighbor 172.16.30.2 route-map EXPORT_TO_CLOUD out
 exit-address-family
```

Это **выдержка из изменений**, а не полный самостоятельный конфигурационный файл. Полный файл: [`configs/border-fw/frr.conf`](configs/border-fw/frr.conf).

</details>

**Ограничение:** маршрутные фильтры BGP сами по себе не обеспечивают полноценную политику безопасности между TENANT_A, TENANT_B и облаком, поскольку Border-FW работает с одной глобальной таблицей маршрутизации. Для промышленного внедрения потребуется отдельная проработка изоляции и фильтрации.

<!-- TODO: итоговая логическая схема Fabric → Border-FW → Cloud-GW без Legacy -->

## 8. Результаты и дальнейшее развитие

В проекте получена работающая **сетевая** модель миграции: целевая EVPN Fabric обслуживает четыре VLAN в двух VRF, Server1 подключён резервированным ESI-LAG, а внешняя маршрутизация и эмулированный облачный стык проверены командами BGP и ICMP.

Следующие возможные шаги — полноценная фильтрация на пограничном узле, реальный стык с VPC/VNet, отдельные облачные хосты, автоматизированные проверки и измерение времени сходимости с более точной методикой.

### Навигация по репозиторию

- [`fabric.clab.yml`](fabric.clab.yml) — топология Containerlab (включая остающиеся Legacy-узлы).
- [`configs/`](configs/) — конфигурации устройств и скрипты инициализации.
- [Предыдущие лабораторные работы OTUS](https://github.com/sergie21/DC_fabric/tree/main/LABs) — подготовка Underlay/Overlay и сетевых технологий.
- [HW7: EVPN Multihoming](https://github.com/sergie21/DC_fabric/tree/main/LABs/HW7_EVPN_Multihoming) — настройка и отдельные тесты ESI-LAG.

<!-- TODO: добавить ссылки на схемы, протоколы испытаний и презентацию после размещения файлов в репозитории -->


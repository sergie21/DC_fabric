# Домашнее задание №1. Проектирование адресного пространства
## Топология сети
<img width="529" height="407" alt="DC_topology" src="https://github.com/user-attachments/assets/b7d13c4f-9fbd-49b8-91d0-06c0c32bab60" />

## Задание:
1. Собрать топологию CLOS
2. Распределить IP-адресацию для сети Underlay
3. Зафиксировать в документации план работ, адресное пространство, схему сети, настройки оборудования.

## IP-адресация:
#### Итоговый общий IP-план:
| Блок          | Назначение             |
| ------------- | ---------------------- |
| `10.0.0.0/16` | Underlay Loopback0     |
| `10.1.0.0/16` | Overlay Loopback1      |
| `10.2.0.0/16` | Underlay P2P           |
| `10.3.0.0/16` | Резерв                 |
| `10.0.0.0/14` | Summary инфраструктуры |
| `10.4.0.0/14` | Summary сервисов       |
| `10.0.0.0/13` | Summary всего ЦОД      |

#### Идентификаторы нод:
| Нода          | Нумерация              |
| ------------- | ---------------------- |
| `Spine1`      | 1                      |
| `Spine2`      | 2                      |
| ...           | ...                    |
| `Spine99`     | 3                      |
| `Leaf1`       | 101                    |
| `Leaf2`       | 102                    |
| `Leaf3`       | 103                    |
| ...           | ...                    |
| `Leaf199`     | 199                    |

#### Loopback-адресация
| Node   | Lo0 - Underlay  | Lo1 - Overlay      |
| ------ | --------------- | ------------------ |
| Spine1 | `10.0.1.0/32`   | —                  |
| Spine2 | `10.0.2.0/32`   | —                  |
| Leaf1  | `10.0.101.0/32` | `10.1.101.0/32`    |
| Leaf2  | `10.0.102.0/32` | `10.1.102.0/32`    |
| Leaf3  | `10.0.103.0/32` | `10.1.103.0/32`    |

Адрес читается как 10.FUNCTION.NODE-ID.INDEX, 
например, 10.1.103.0/32: 1 = overlay, node 103 = Leaf3, 0 = адрес.

#### Peer-to-peer адресация
| Link         | Spine interface/IP | Leaf interface/IP | Prefix |
| ------------ | ------------------ | ----------------- | ------ |
| Spine1–Leaf1 | S1 Et1 `10.2.1.0`  | L1 Et1 `10.2.1.1` | `/31`  |
| Spine1–Leaf2 | S1 Et2 `10.2.1.2`  | L2 Et1 `10.2.1.3` | `/31`  |
| Spine1–Leaf3 | S1 Et3 `10.2.1.4`  | L3 Et1 `10.2.1.5` | `/31`  |
| Spine2–Leaf1 | S2 Et1 `10.2.2.0`  | L1 Et2 `10.2.2.1` | `/31`  |
| Spine2–Leaf2 | S2 Et2 `10.2.2.2`  | L2 Et2 `10.2.2.3` | `/31`  |
| Spine2–Leaf3 | S2 Et3 `10.2.2.4`  | L3 Et2 `10.2.2.5` | `/31`  |

Адрес читается как 10.FUNCTION.SPINE.SUBNET, 
например, 10.2.2.4/31: 2 = P2P, 2 = Spine2, 4 = номер P2P-подсети к Leaf, 
где 10.2.2.4/31: 2 = P2P, 2 = Spine2, 4 = адрес Spine2; 
а 10.2.2.5/31: 2 = P2P, 2 = Spine2, 4 = адрес Leaf3.

#### Leaf-server адресация
| Link          | Leaf interface/IP  | Server interface/IP | Prefix |
| ------------  | ------------------ | -----------------   | ------ |
| Leaf1–Server1 | L1 Et3 `10.4.1.0`  | S1 Et1 `10.4.1.1`   | `/31`  |
| Leaf2–Server2 | L2 Et3 `10.4.2.0`  | S2 Et1 `10.4.2.1`   | `/31`  |
| Leaf3–Server3 | L3 Et3 `10.4.3.0`  | S3 Et1 `10.4.3.1`   | `/31`  |
| Leaf3–Server4 | L3 Et4 `10.4.3.2`  | S4 Et1 `10.4.3.3`   | `/31`  |

Адрес читается как 10.4.LEAF.SERVER

### Вывод IP-адресации узлов
#### Leaf1
```
leaf1>show ip interface brief 
                                                                            Address
Interface        IP Address          Status      Protocol            MTU    Owner  
---------------- ------------------- ----------- -------------- ----------- -------
Ethernet1        10.2.1.1/31         up          up                 1500           
Ethernet2        10.2.2.1/31         up          up                 1500           
Ethernet3        10.4.1.0/31         up          up                 1500           
Loopback0        10.0.11.0/32        up          up                65535           
Loopback1        10.1.11.0/32        up          up                65535           
Management0      172.20.20.4/24      up          up                 1500           
```

#### Leaf2

leaf2>sh ip interface brief 
                                                                            Address
Interface        IP Address          Status      Protocol            MTU    Owner  
---------------- ------------------- ----------- -------------- ----------- -------
Ethernet1        10.2.1.3/31         up          up                 1500           
Ethernet2        10.2.2.3/31         up          up                 1500           
Ethernet3        10.4.2.0/31         up          up                 1500           
Loopback0        10.0.12.0/32        up          up                65535           
Loopback1        10.1.12.0/32        up          up                65535           
Management0      172.20.20.5/24      up          up                 1500           

#### Leaf3

leaf3>sh ip interface brief 
                                                                            Address
Interface        IP Address          Status      Protocol            MTU    Owner  
---------------- ------------------- ----------- -------------- ----------- -------
Ethernet1        10.2.1.5/31         up          up                 1500           
Ethernet2        10.2.2.5/31         up          up                 1500           
Ethernet3        10.4.3.0/31         up          up                 1500           
Ethernet4        10.4.3.2/31         up          up                 1500           
Loopback0        10.0.13.0/32        up          up                65535           
Loopback1        10.1.13.0/32        up          up                65535           
Management0      172.20.20.6/24      up          up                 1500           















































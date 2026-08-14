# Домашнее задание №1. Проектирование адресного пространства
## Топология сети
Стенд развернут на ПК с Windows 11 с использованием WSL2 (Ubuntu). В Ubuntu установлен Docker Engine и Containerlab. Сетевые устройства эмулируются контейнерами Arista cEOS 4.33.2F, серверы - контейнерами Alpine Linux. Топология создается и управляется Containerlab. Containerlab Desktop используется как графический интерфейс и подключается к Containerlab, работающему в WSL2, по адресу 172.30.229.65:8090. Управление cEOS из CLD выполняется по SSH через отдельную management-сеть 172.20.20.0/24.

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
| `Spine99`     | 99                      |
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
например, 10.2.2.4/31: 2 = P2P, 2 = Spine2, 4 = номер P2P-подсети к Leaf:
___
10.2.2.4/31: 2 = P2P, 2 = Spine2, 4 = адрес Spine2; 
10.2.2.5/31: 2 = P2P, 2 = Spine2, 4 = адрес Leaf3.

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
```
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
```

#### Leaf3
```
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
```

#### Spine1
```
spine1>show ip interface brief 
                                                                            Address
Interface        IP Address          Status      Protocol            MTU    Owner  
---------------- ------------------- ----------- -------------- ----------- -------
Ethernet1        10.2.1.0/31         up          up                 1500           
Ethernet2        10.2.1.2/31         up          up                 1500           
Ethernet3        10.2.1.4/31         up          up                 1500           
Loopback0        10.0.1.0/32         up          up                65535           
Management0      172.20.20.2/24      up          up                 1500           
```

#### Spine2
```
spine2>show ip interface brief 
                                                                            Address
Interface        IP Address          Status      Protocol            MTU    Owner  
---------------- ------------------- ----------- -------------- ----------- -------
Ethernet1        10.2.2.0/31         up          up                 1500           
Ethernet2        10.2.2.2/31         up          up                 1500           
Ethernet3        10.2.2.4/31         up          up                 1500           
Loopback0        10.0.2.0/32         up          up                65535           
Management0      172.20.20.3/24      up          up                 1500           
```

### Выборочный пинг соседей
#### Пингуем серверы №3/4 с лифа №3
```
leaf3>ping 10.4.3.1
PING 10.4.3.1 (10.4.3.1) 72(100) bytes of data.
80 bytes from 10.4.3.1: icmp_seq=1 ttl=64 time=1.51 ms
80 bytes from 10.4.3.1: icmp_seq=2 ttl=64 time=0.066 ms
80 bytes from 10.4.3.1: icmp_seq=3 ttl=64 time=0.049 ms
80 bytes from 10.4.3.1: icmp_seq=4 ttl=64 time=0.042 ms
80 bytes from 10.4.3.1: icmp_seq=5 ttl=64 time=0.012 ms

--- 10.4.3.1 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 5ms
rtt min/avg/max/mdev = 0.012/0.335/1.508/0.586 ms, ipg/ewma 1.356/0.900 ms

leaf3>ping 10.4.3.3
PING 10.4.3.3 (10.4.3.3) 72(100) bytes of data.
80 bytes from 10.4.3.3: icmp_seq=1 ttl=64 time=4.44 ms
80 bytes from 10.4.3.3: icmp_seq=2 ttl=64 time=0.034 ms
80 bytes from 10.4.3.3: icmp_seq=3 ttl=64 time=0.025 ms
80 bytes from 10.4.3.3: icmp_seq=4 ttl=64 time=0.013 ms
80 bytes from 10.4.3.3: icmp_seq=5 ttl=64 time=0.013 ms

--- 10.4.3.3 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 15ms
rtt min/avg/max/mdev = 0.013/0.904/4.437/1.766 ms, ipg/ewma 3.666/2.609 ms
```
#### Пингуем спайны №1/2 с лифа №2
```
leaf2>ping 10.2.1.3
PING 10.2.1.3 (10.2.1.3) 72(100) bytes of data.
80 bytes from 10.2.1.3: icmp_seq=1 ttl=64 time=0.970 ms
80 bytes from 10.2.1.3: icmp_seq=2 ttl=64 time=0.058 ms
80 bytes from 10.2.1.3: icmp_seq=3 ttl=64 time=0.013 ms
80 bytes from 10.2.1.3: icmp_seq=4 ttl=64 time=0.011 ms
80 bytes from 10.2.1.3: icmp_seq=5 ttl=64 time=0.011 ms

--- 10.2.1.3 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4ms
rtt min/avg/max/mdev = 0.011/0.212/0.970/0.379 ms, ipg/ewma 1.027/0.577 ms

leaf2>ping 10.2.2.3
PING 10.2.2.3 (10.2.2.3) 72(100) bytes of data.
80 bytes from 10.2.2.3: icmp_seq=1 ttl=64 time=3.86 ms
80 bytes from 10.2.2.3: icmp_seq=2 ttl=64 time=0.039 ms
80 bytes from 10.2.2.3: icmp_seq=3 ttl=64 time=0.055 ms
80 bytes from 10.2.2.3: icmp_seq=4 ttl=64 time=0.016 ms
80 bytes from 10.2.2.3: icmp_seq=5 ttl=64 time=0.024 ms

--- 10.2.2.3 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 13ms
rtt min/avg/max/mdev = 0.016/0.798/3.858/1.529 ms, ipg/ewma 3.268/2.274 ms
```
#### Пингуем лифы №1/2/3 со спайна №1
```
spine1>ping 10.2.1.0
PING 10.2.1.0 (10.2.1.0) 72(100) bytes of data.
80 bytes from 10.2.1.0: icmp_seq=1 ttl=64 time=0.219 ms
80 bytes from 10.2.1.0: icmp_seq=2 ttl=64 time=0.016 ms
80 bytes from 10.2.1.0: icmp_seq=3 ttl=64 time=0.009 ms
80 bytes from 10.2.1.0: icmp_seq=4 ttl=64 time=0.009 ms
80 bytes from 10.2.1.0: icmp_seq=5 ttl=64 time=0.011 ms

--- 10.2.1.0 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.009/0.052/0.219/0.083 ms, ipg/ewma 0.122/0.133 ms

spine1>ping 10.2.1.2
PING 10.2.1.2 (10.2.1.2) 72(100) bytes of data.
80 bytes from 10.2.1.2: icmp_seq=1 ttl=64 time=3.15 ms
80 bytes from 10.2.1.2: icmp_seq=2 ttl=64 time=0.022 ms
80 bytes from 10.2.1.2: icmp_seq=3 ttl=64 time=0.048 ms
80 bytes from 10.2.1.2: icmp_seq=4 ttl=64 time=0.012 ms
80 bytes from 10.2.1.2: icmp_seq=5 ttl=64 time=0.012 ms

--- 10.2.1.2 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 10ms
rtt min/avg/max/mdev = 0.012/0.649/3.154/1.252 ms, ipg/ewma 2.573/1.858 ms

spine1>ping 10.2.1.4
PING 10.2.1.4 (10.2.1.4) 72(100) bytes of data.
80 bytes from 10.2.1.4: icmp_seq=1 ttl=64 time=3.56 ms
80 bytes from 10.2.1.4: icmp_seq=2 ttl=64 time=0.111 ms
80 bytes from 10.2.1.4: icmp_seq=3 ttl=64 time=0.012 ms
80 bytes from 10.2.1.4: icmp_seq=4 ttl=64 time=0.050 ms
80 bytes from 10.2.1.4: icmp_seq=5 ttl=64 time=0.018 ms

--- 10.2.1.4 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 12ms
rtt min/avg/max/mdev = 0.012/0.750/3.559/1.404 ms, ipg/ewma 3.001/2.104 ms
```

Адресация Underlay настроена. 













































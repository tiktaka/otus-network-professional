# PBR

## Оглавление
- [Цель](#цель)
- [Задание](#задание)
- [Настройка по заданию](#настройка-по-заданию)
  - [Настройка для офиса Лабытнанги маршрута по-умолчанию](#настройка-для-офиса-лабытнанги-маршрута-по-умолчанию)
  - [Настройка PBR](#настройка-pbr)
- [Проверка результата](#проверка-результата)
  - [Проверка работы PBR](#проверка-работы-pbr)
  - [Проверка работы маршрута по умолчанию на R27](#проверка-работы-маршрута-по-умолчанию-на-r27)
- [Файлы конфигураций](#файлы-конфигураций)

## Цель
Настроить политику маршрутизации в офисе Чокурдах;
Распределить трафик между 2 линками.

## Задание
1. Настройте политику маршрутизации для сетей офиса
2. Распределите трафик между двумя линками с провайдером
3. Настроите отслеживание линка через технологию IP SLA.(только для IPv4)
4. Настройте для офиса Лабытнанги маршрут по-умолчанию
5. План работы и изменения зафиксированы в документации

## Настройка по заданию
### Настройка для офиса Лабытнанги маршрута по-умолчанию
1. Задаем маршрут по-умолчанию и включаем маршрутизацию на маршрутизаторе R27:
```
Labitnangi_R27(config)#ip route 0.0.0.0 0.0.0.0 100.34.0.1
Labitnangi_R27(config)#ip routing
Labitnangi_R27#sh ip route
...
Gateway of last resort is 100.34.0.1 to network 0.0.0.0

S*    0.0.0.0/0 [1/0] via 100.34.0.1
...
```
2. Добавим на R25 и R26 маршруты обратные маршруты до маршрутизатора R27:
```
!R26:
ip route 100.40.27.254 255.255.255.255 10.0.0.13 name "to R27 (Lo)"
ip route 100.34.0.0 255.255.255.252 10.0.0.13 name "to R27 (Eth0/0)"

!R25:
ip route 100.40.27.254 255.255.255.255 100.34.0.2 name "to R27 (Eth0/0)"
ip route 100.35.27.4 255.255.255.252 10.0.0.14 name "to R26 (Eth0/1)"
```

### Настройка PBR
1. Создадим мониторинг шлюзов провайдера Триада через IP SLA:
```
ip sla 1
 icmp-echo 100.35.0.1 source-ip 100.35.0.2
 frequency 5
exit
ip sla schedule 1 life forever start-time now

ip sla 2
 icmp-echo 100.35.0.5 source-ip 100.35.0.6
 frequency 5
exit
ip sla schedule 2 life forever start-time now

track 1 ip sla 1 reachability
track 2 ip sla 2 reachability
```

2. Добавим маршруты по умолчанию на R28:
```
ip route 0.0.0.0 0.0.0.0 100.35.0.1 name "to R25 (Triada)"
ip route 0.0.0.0 0.0.0.0 100.35.0.5 name "to R26 (Triada)"
```

3. Создадим ACL и route-map для VLAN 10 и 20:
```
ip access-list standard VLAN10-PBR
    permit 10.10.0.0 0.0.0.127 any

ip access-list standard VLAN20-PBR
    permit 10.20.0.0 0.0.0.127 any

route-map BALANCING permit 10
 match ip address VLAN10-PBR
 set ip next-hop verify-availability 100.35.0.1 10 track 1
 set ip next-hop verify-availability 100.35.0.5 20 track 2
 set ip next-hop 100.35.0.1

route-map BALANCING permit 20
 match ip address VLAN20-PBR
 set ip next-hop verify-availability 100.35.0.5 10 track 2
 set ip next-hop verify-availability 100.35.0.1 20 track 1
 set ip next-hop 100.35.0.5
```
Данная конфигурация route-map указывает трафику с VLAN10 идти шлюз 100.35.0.1, а VLAN20 - на 100.35.0.5. track здесь отслеживает доступность шлюзов и позволяет переключиться на резервный линк в случае недоступности основного.
4. Привяжем route-map к интерфейсам и укажем роли интерфейсов в NAT:
```
interface Ethernet0/1
...
ip nat outside

interface Ethernet0/0
...
ip nat outside

interface Ethernet0/2.10
...
 ip policy route-map BALANCING
 ip nat inside

interface Ethernet0/2.20
...
 ip policy route-map BALANCING
 ip nat inside
```
5. Настроим route-map для определение интерфейса трансляции и NAT:
```
route-map toR25
  match interface Ethernet0/1
route-map toR26
  match interface Ethernet0/0

ip nat inside source route-map toR25 interface Ethernet0/1 overload
ip nat inside source route-map toR26 interface Ethernet0/0 overload
```
NAT настроен сразу, чтобы не прописывать обратные маршруты на R25 и R26 до всех внутренних сетей.
## Проверка результата
### Проверка работы PBR
Выполним trace до Lo-интерфеса R27 (Лабытнанги) с VPC30 и VPC31:
```
VPC30> trace 100.40.27.254
trace to 100.40.27.254, 8 hops max, press Ctrl+C to stop
 1   10.10.0.1   0.803 ms  0.702 ms  0.714 ms
 2   100.35.0.1   1.125 ms  1.153 ms  1.083 ms
 3   *100.34.0.2   1.642 ms (ICMP type:3, code:3, Destination port unreachable)  *

VPC31> trace 100.40.27.254
trace to 100.40.27.254, 8 hops max, press Ctrl+C to stop
 1   10.20.0.1   0.958 ms  0.813 ms  0.712 ms
 2   100.35.0.5   1.126 ms  1.033 ms  0.830 ms
 3   10.0.0.13   1.067 ms  0.914 ms  0.890 ms
 4   *100.34.0.2   1.309 ms (ICMP type:3, code:3, Destination port unreachable)  * 
```
Видим, что трафик балансируется так, как было сконфигурировано:
1. С VPC30 (VLAN10) трафик идёт через линк с R25, а потом до R27;
2. С VPC31 (VLAN20) трафик идёт через линк с R26, далее до R25 и уже до R27.

### Проверка работы маршрута по умолчанию на R27
У нас уже ранее настроена маршрутизация до интерфейса Eth0/1 (100.35.0.5) на R26, сделаем проверку доступности с R27 до него:
```
Labitnangi_R27#ping 100.35.0.5
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 100.35.0.5, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/1 ms
Labitnangi_R27#traceroute 100.35.0.5
Type escape sequence to abort.
Tracing the route to 100.35.0.5
VRF info: (vrf in name/id, vrf out name/id)
  1 100.34.0.1 1 msec 0 msec 1 msec
  2 10.0.0.14 0 msec *  1 msec
```
Видим, что все проверки успешны.

## [Файлы конфигураций](configs/)
# Policy-based routing на CentOS 7/8 

## Какво е Policy-based routing?
Маршрутизирането на базата на политики (Policy-based routing или PBR) е техника, която препраща и маршрутизира пакети данни въз основа на политики или филтри. Мрежовите администратори могат да прилагат избирателно политики въз основа на специфични параметри като `source` и `destination` IP адрес, `source port` или `destination port`, тип трафик, протоколи, списък за достъп, размер на пакета или други критерии и след това да маршрутизират пакетите по зададени от потребителя маршрути.
 
Целта на PBR е да направи мрежата възможно най-гъвкава. Чрез дефиниране на поведението на маршрутизация въз основа на атрибутите на приложението, PBR предоставя гъвкави, подробни възможности за управление на трафика за пренасочване на пакети. По този начин PBR дава възможност на мрежовите администратори да постигнат оптимално използване на честотната лента за критично важни за бизнеса приложения.
## Приложение в Linux
С Policy-based routing можем да конфигурираме две или повече публични мрежи на един сървър достъпни през интернет.
## Примерна постановка
Имаме двa интерфейса на които искаме да настроим два адреса от различни мрежи. И двата адреса трябва да са достъпни от интернет. Трафика до тези адреси трябва да влиза и излиза от съответния интерфейс.
### Мрежа 1
```
Interface: eno1
Address: 192.168.10.10
Network: 192.168.10.0/24
Gateway: 192.168.10.1
```
### Мрежа 2
```
Interface: ens15f1
Address: 192.168.20.20
Network: 192.168.20.0/24
Gateway: 192.168.20.1
```
## Конфигурация
### NetworkManager-dispatcher
Услугата `NetworkManager-dispatcher` пристига с пакет `NetworkManager`. Към ния ни е нужен и пакета `NetworkManager-dispatcher-routing-rules`.
```
yum install NetworkManager-dispatcher-routing-rules
```
Активираме `NetworkManager-dispatcher` за стартиране при boot.
```
systemctl enable NetworkManager-dispatcher
```
### Добавяне на routing таблици
Преди всичко трябва да създадем routing таблици в `iproute2`. Отваряме файла `/etc/iproute2/rt_tables` и в края му добавяме нашите routing таблици. За повече пригледност ще ги кръстим с имената на нашите интерфейси.
```
...
201		eno1
200		ens15f1
```
### Конфигуриране на мрежовите скриптове
Добавяме конфигурацията за мрежите стандартно във файловете `ifcfg-XXX` в директорията `/etc/sysconfig/network-scripts`.
#### `ifcfg-eno1`
```
...
BOOTPROTO=none
DEFROUTE=yes
...
ONBOOT=yes
IPADDR=192.168.10.10
PREFIX=28
GATEWAY=192.168.10.1
DNS1=8.8.8.8
DNS2=8.8.4.4
```
#### `ifcfg-ens15f1`
```
...
BOOTPROTO=none
DEFROUTE=yes
...
ONBOOT=yes
IPADDR=192.168.20.20
PREFIX=28
GATEWAY=192.168.20.1
DNS1=8.8.8.8
DNS2=8.8.4.4
```
### Конфигуриране на route файлове
Добавяме `route-XXX` файлове за всеки интерфейс в директорията `/etc/sysconfig/network-scripts`, като вместо `XXX` изписваме името на нашите интерфейси.
#### `route-eno1`
```
default via 192.168.10.1 dev eno1 table eno1
192.168.10.0/24 dev eno1 src 192.168.10.10 table eno1
```
> С първия ред настройваме кой адрес да бъде default gateway.
> С втория ред настройваме адреса на машината. Така ако има трафик, който да е предназначен за същата мрежа, то той ще се минава по L2 и няма да се върти до рутера.
#### `route-ens15f1`
```
default via 192.168.20.1 dev ens15f1 table ens15f1
192.168.20.0/24 dev ens15f1 src 192.168.20.20 table ens15f1
```
### Конфигуриране на rule файлове
В тези файлове конфигурираме правилата при които да се използват routing таблиците. Те също се създават в директорията `/etc/sysconfig/network-scripts` и се именоват по подобен начин: `rule-XXX`
#### `rule-eno1`
```
from 192.168.10.0/24 table eno1
to 192.168.10.0/24 table eno1
```
> В този пример казваме, че трафика от и за мрежа `192.168.10.0/24` трябва да се рутира през таблица `eno1`, която има своя gateway конфигуриран в файла `route-eno1`.
#### `rule-ens15f1`
```
from 192.168.20.0/24 table ens15f1
to 192.168.20.0/24 table ens15f1
```
### Рестартиране на мрежата
За да приложим новите настройки трябва да рестартираме мрежовия сървис.
#### CentOS 7
```
systemctl restart network
```
#### CentOS 8
```
systemctl restart NetworkManager
```
> В някои случаи при CentOS 8, се налага мрежата да се рестартира и през `nmcli` със следната команда `nmcli networking off && nmcli networking on`.
{.is-warning}
## Проверка
### ping
Ако всичко е наред с нашата конфигурация трябва да имаме ping и до двата адреса.
Също така, трябва да имаме ping до външен адрес от нашата машина и през двата ни интерфейса.
```
# ping 8.8.8.8 -c 4 -I eno1
PING 8.8.8.8 (8.8.8.8) from 192.168.10.10 eno1: 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=121 time=0.386 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=121 time=0.377 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=121 time=0.407 ms
64 bytes from 8.8.8.8: icmp_seq=4 ttl=121 time=0.443 ms

--- 8.8.8.8 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 60ms
rtt min/avg/max/mdev = 0.377/0.403/0.443/0.029 ms
```
```
# ping 8.8.8.8 -c 4 -I ens15f1
PING 8.8.8.8 (8.8.8.8) from 192.168.20.20 ens15f1: 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=121 time=0.363 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=121 time=0.387 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=121 time=0.374 ms
64 bytes from 8.8.8.8: icmp_seq=4 ttl=121 time=0.386 ms

--- 8.8.8.8 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 109ms
rtt min/avg/max/mdev = 0.363/0.377/0.387/0.021 ms
```
За проверка дали L2 трафика ни минава наистина по L2, а не през рутера, можем да пуснем ping до съседен адрес и да видим TTL-a. Ако той е равен на 64, значи трафика ни преминава дирекно до съседа, ако е по-малък, например 63, то значи сме сбъркали някъде.
```
# ping 192.168.20.20 -c 4
PING 192.168.20.20 (192.168.20.20) 56(84) bytes of data.
64 bytes from 192.168.20.20: icmp_seq=1 ttl=64 time=0.060 ms
64 bytes from 192.168.20.20: icmp_seq=2 ttl=64 time=0.023 ms
64 bytes from 192.168.20.20: icmp_seq=3 ttl=64 time=0.043 ms
64 bytes from 192.168.20.20: icmp_seq=4 ttl=64 time=0.043 ms

--- 192.168.20.20 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 64ms
rtt min/avg/max/mdev = 0.023/0.042/0.060/0.013 ms
```
### `ip route`
Друг начин да проверим дали всичко е неред е с командата `ip route` или `ip route show table main`. Тя показва рутовете от `main` таблицата.
```
# ip route
default via 192.168.20.1 dev ens15f1 proto static metric 101 
default via 192.168.10.1 dev eno1 proto static metric 102 
192.168.20.0/24 dev ens15f1 proto kernel scope link src 192.168.20.20 metric 101 
192.168.10.0/24 dev eno1 proto kernel scope link src 192.168.10.10 metric 102
```
За другите две `routing` таблици трябва да имаме нещо подобно:
#### Таблица `eno1`
```
# ip route show table eno1
default via 192.168.10.1 dev eno1 
192.168.10.0/24 dev eno1 scope link src 192.168.10.10
```
#### Таблица `ens15f1`
```
# ip route show table ens15f1
default via 192.168.20.1 dev ens15f1 
192.168.20.0/24 dev ens15f1 scope link src 192.168.20.20
```

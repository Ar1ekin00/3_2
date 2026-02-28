# Образец задания для ГИА ДЭ ПУ (инвариантная часть)
---
## Топология сети
![](https://github.com/Ar1ekin00/Sources/blob/main/network_topology.jpg)


---
## Модуль 1. Настройка сетевой инфраструктуры
### 1. НАСТРОЙКА ISP
#### 1.1 Настройка полного доменного имени, часового пояса
```bash
# Имя 
hostnamectl set-hostname isp.net01tech.institute
exec bash

# Часовой пояс
apt-get update
apt-get install tzdata
timedatectl set-timezone 'Asia/Yekaterinburg'

# Проверка времени и имени
date
cat /etc/hostname
```
#### 1.2 Настройка сетевых адаптеров и iptables
```bash
# Настройка сетевых интерфейсов
cd /etc/net/ifaces/
cp -r ens18 ens19
cp -r ens18 ens20
echo 172.16.1.1/28 > ens19/ipv4address
echo BOOTPROTO=static > ens19/options
echo TYPE=eth >> ens19/options

echo 172.16.2.1/28 > ens20/ipv4address
echo BOOTPROTO=static > ens20/options
echo TYPE=eth >> ens20/options

# Настройка iptables
apt-get install iptables

В файле /etc/net/sysctl.conf
net.ipv4.ip_forward = 1

sysctl -p

iptables -t nat -A POSTROUTING -o ens18 -j MASQUERADE
iptables-save -f /etc/sysconfig/iptables 

systemctl enable --now iptables
systemctl restart network
```

---

### 2. НАСТРОЙКА HQ-RTR
#### 2.1 Настройка полного доменного имени, часового пояса
```bash
# Имя
en 
conf t
hostname hq-rtr.net01tech.institute	

# Часовой пояс
ntp timezone utc+5

# Проверка времени
exit
show ntp date
```

#### 2.2 Настройка логических интерфейсов и сопряжение их с физическими портами. Настройка VLAN

```bash
# Настройка интерфейса в сторону ISP 
conf t
interface int0
ip address 172.16.1.2/28
ip nat outside
no sh
exit

# Сопряжение интерфейса и порта в сторону ISP
port te0
service-instance int0
encapsulation untagged 
connect ip interface int0
exit
exit

# Настройка шлюза
ip route 0.0.0.0/0 172.16.1.1 16
```

__Настройка интерфейсов в сторону локальных сетей VLAN 100, 200, 999__

```
# Настройка интерфейса VLAN100
interface int1.100
ip address 192.168.10.1/29
ip nat inside
no sh
exit

# Сопряжение интерфейса VLAN100 с портом te1
port te1
service-instance int1.100
encapsulation dot1q 100
rewrite pop 1
connect ip interface int1.100
exit
exit

# Настройка интерфейса VLAN999
interface int1.999
ip address 192.168.99.1/29
ip nat inside
no sh
exit

# Сопряжение интерфейса VLAN999 с портом te1
port te1
service-instance int1.999
encapsulation dot1q 999
rewrite pop 1
connect ip interface int1.999
exit
exit

# Настройка интерфейса VLAN200
interface int1.200
ip address 192.168.20.1/28
ip nat inside
no sh
exit

# Сопряжение интерфейса VLAN200 с портом te1
port te1
service-instance int1.200
encapsulation dot1q 200
rewrite pop 1
connect ip interface int1.200
exit
exit

# Настройка NAT
ip nat pool NAT 192.168.10.1-192.168.255.254
ip nat source dynamic inside-to-outside pool NAT overload interface int0
```

#### 2.3 Настройка GRE-туннель 
```bash
interface tunnel.0
ip mtu 1400
ip address 10.0.0.2/30
ip tunnel 172.16.1.2 172.16.2.2 mode gre
no shutdown
exit
```

#### 2.4 Настройка динамической маршрутизации (OSPF) на HQ-RTR
```bash
router ospf 1
network 192.168.10.0 0.0.0.7 area 0
network 192.168.20.0 0.0.0.15 area 0
network 192.168.99.0 0.0.0.7 area 0
network 10.0.0.0 0.0.0.3 area 0.0.0.0
passive-interface default
no passive-interface tunnel.0
area 0 authentication message-digest
exit
```

#### 2.5 Установка пароля на OSPF
```bash
interface int1.100
ip ospf authentication-key ecorouter
ip ospf message-digest-key 1 md5 ecorouter
exit
interface int1.200
ip ospf authentication-key ecorouter
ip ospf message-digest-key 1 md5 ecorouter
exit
interface int1.999
ip ospf authentication-key ecorouter
ip ospf message-digest-key 1 md5 ecorouter
exit
interface tunnel.0
ip ospf authentication-key ecorouter
ip ospf message-digest-key 1 md5 ecorouter
exit
```

#### 2.6 Протокол динамической конфигурации хостов для сети в сторону HQ-CLI
```bash
ip pool hq-cli 192.168.20.2-192.168.20.14
dhcp-server 1
pool hq-cli 1
mask 255.255.255.240
gateway 192.168.20.1
dns 8.8.8.8
domain-name net01tech.institute
lease 3600
exit
exit
interface int1.200 
dhcp-server 1
exit
exit
w
```

---

### 3. НАСТРОЙКА BR-RTR

#### 3.1 Настройка полного доменного имени, часового пояса
```bash
# Имя 
en 
conf t
hostname br-rtr.net01tech.institute

# Часовой пояс
ntp timezone utc+5

# Проверка времени
exit
show ntp date
```

#### 3.2 Настройка логических интерфейсов и сопряжение их с физическими портами
```bash
# Настройка интерфейса в сторону ISP 
conf t
interface int0
ip address 172.16.2.2/28
ip nat outside
no sh
exit

# Сопряжение интерфейса и порта в сторону ISP
port te0
service-instance int0
encapsulation untagged 
connect ip interface int0
exit
exit

# Настройка шлюза
ip route 0.0.0.0/0 172.16.2.1 16

# Настройка интерфейса в сторону локальной сети
interface int1
ip address 192.168.30.1/29
ip nat inside
no sh
exit

# Сопряжение интерфейса с портом te1
port te1
service-instance int1
encapsulation untagged
connect ip interface int1
exit
exit

# Настройка NAT
ip nat pool NAT 192.168.30.1-192.168.255.254
ip nat source dynamic inside-to-outside pool NAT overload interface int0
```

#### 3.3 Настройка GRE-туннель 
```bash
interface tunnel.0
ip mtu 1400
ip address 10.0.0.1/30
ip tunnel 172.16.2.2 172.16.1.2 mode gre
no shutdown
exit
```

#### 3.4 Настройка динамической маршрутизации (OSPF) на BR-RTR
```bash
router ospf 1
network 192.168.30.0 0.0.0.7 area 0
network 10.0.0.0 0.0.0.3 area 0
passive-interface default 
no passive-interface tunnel.0
area 0 authentication message-digest
exit
```

#### 3.5 Установка пароля на OSPF
```bash
interface int1
ip ospf authentication-key ecorouter
ip ospf message-digest-key 1 md5 ecorouter
exit
interface tunnel.0
ip ospf authentication-key ecorouter
ip ospf message-digest-key 1 md5 ecorouter
exit
exit
w
```

---

### 4. НАСТРОЙКА HQ-CLI
#### 4.1 Перезагрузка сети, установка часового пояса и времени
```bash
# Перезагрузить сеть
systemctl restart network

# Имя 
hostnamectl set-hostname hq-cli.net01tech.institute
exec bash

# Часовой пояс
apt-get update
apt-get install tzdata
timedatectl set-timezone 'Asia/Yekaterinburg'
```

---

### 5. НАСТРОЙКА HQ-SRV
#### 5.1 Настройка сети, имени, часового пояса
```bash
# Настройка сетевого интерфейса
cd /etc/net/ifaces/
echo 192.168.10.2/29 > ens18/ipv4address
echo BOOTPROTO=static > ens18/options
echo TYPE=eth >> ens18/options
echo default via 192.168.10.1 > ens18/ipv4route
echo nameserver 8.8.8.8 > ens18/resolv.conf
systemctl restart network

# Имя 
hostnamectl set-hostname hq-srv.net01tech.institute
exec bash

# Часовой пояс
apt-get update
apt-get install tzdata
timedatectl set-timezone 'Asia/Yekaterinburg'
```
#### 5.2 Создание пользователя, настройка прав
```bash
# Создание пользователя
useradd -u 2001 -m -s /bin/bash remote_admin
passwd remote_admin

# Пароль, который нужно ввести, после предыдущей команды
SuperPass!1

# Настройка прав
usermod -aG wheel remote_admin
apt-get install sudo 
EDITOR=mcedit visudo

# Добавить в начало файла
WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL
```

#### 5.3 Безопасный удалённый дооступ
```bash
# Настройка SSH
mcedit /etc/openssh/sshd_config

# Добавить в начало файла
Port 2201
AllowUsers remote_admin
MaxAuthTries 3
Banner /etc/openssh/banner.txt
PasswordAuthentication yes

# Создание баннера
mcedit /etc/openssh/banner.txt
Внутри пишем:
«Authorized access only»

# Запуск sshd
systemctl enable --now sshd

# Проверка подключения
ssh remote_admin@192.168.30.2 -p 2206  #Подключение к BR-SRV
```

### 6. НАСТРОЙКА BR-SRV
#### 6.1 Настройка сети, имени, часового пояса
```bash
# Настройка сетевого интерфейса
cd /etc/net/ifaces/
echo 192.168.30.2/29 > ens18/ipv4address
echo BOOTPROTO=static > ens18/options
echo TYPE=eth >> ens18/options
echo default via 192.168.30.1 > ens18/ipv4route
echo nameserver 8.8.8.8 > ens18/resolv.conf
systemctl restart network

# Имя 
hostnamectl set-hostname br-srv.net01tech.institute	
exec bash

# Часовой пояс
apt-get update
apt-get install tzdata
timedatectl set-timezone 'Asia/Yekaterinburg'
```

#### 6.2 Создание пользователя, настройка прав
```bash
# Создание пользователя
useradd -u 2001 -m -s /bin/bash remote_admin
passwd remote_admin

# Пароль, который нужно ввести, после предыдущей команды
SuperPass!1 

# Настройка прав
usermod -aG wheel remote_admin
apt-get install sudo 
EDITOR=mcedit visudo

# Добавить в начало файла
WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL
```

#### 6.3 Безопасный удалённый дооступ
```bash
# Настройка SSH
mcedit /etc/openssh/sshd_config

# Добавить в начало файла
Port 2201
AllowUsers remote_admin
MaxAuthTries 3
Banner /etc/openssh/banner.txt
PasswordAuthentication yes

# Создание баннера
mcedit /etc/openssh/banner.txt
Внутри пишем:
«Authorized access only»

# Запуск sshd
systemctl enable --now sshd
```

### 7. Проверка работы удаленного подключения
```bash
# На CLI
ssh remote_admin@192.168.10.2 -p 2201  #Подключение к HQ-SRV
ssh remote_admin@192.168.30.2 -p 2201  #Подключение к BR-SRV
```

### 8. Настройка DNS-сервера на HQ-SRV
```bash
# Установка пакета
apt-get install dnsmasq 

# Настройка dnsmasq
mcedit /etc/sysconfig/dnsmasq

# В файле отключаем автоматическое обновление resolv.conf хелпером
USE_RESOLV_CONF="no"

# Редактирование файла
EDITOR=mcedit systemctl edit dnsmasq.service 

# Добавить
[Service]
ExecStart=
ExecStart=/usr/sbin/dnsmasq -k --user=root --pid-file=/run/dnsmasq.pid
 
# Запуск 
systemctl daemon-reload
systemctl enable --now dnsmasq
 
# Редактирование файла
mcedit /etc/resolv.conf

# Добавить, если нет
nameserver 127.0.0.1
 
# Редактирование файла
mcedit /etc/dnsmasq.conf

# Добавить в файл:

# Есть в файле, но закомментированы 
no-resolv
interface=*
bind-interfaces

# Можно указать только 8.8.8.8
server=77.88.8.7
server=77.88.8.3

# Две настройки ниже указаны по умолчанию, но стоит перепроверить
expand-hosts
localise-queries

domain=net01tech.institute
address=/hq-rtr.net01tech.institute/10.0.0.2
address=/br-rtr.net01tech.institute/10.0.0.1
address=/hq-srv.net01tech.institute/192.168.10.2
address=/hq-cli.net01tech.institute/192.168.20.2
address=/br-srv.net01tech.institute/192.168.30.2
address=/docker.net01tech.institute/172.16.1.1
address=/web.net01tech.institute/172.16.2.1

ptr-record=2.0.0.10.in-addr.arpa,hq-rtr.net01tech.institute
ptr-record=2.10.168.192.in-addr.arpa,hq-srv.net01tech.institute
ptr-record=2.20.168.192.in-addr.arpa,hq-cli.net01tech.institute

# Перезагрузить сервис
systemctl restart dnsmasq
```

### 9. Проверка работы DNS-сервера
#### 9.1 Изменение DNS-сервера на машинах сети, установка ПО для проверки
```bash
# На BR-SRV поменять в файле resolv.conf
nameserver 192.168.10.2

# На HQ-RTR поменять в 
en
conf t
dhcp-server 1
pool hq-cli 1
no dns 8.8.8.8
dns 192.168.10.2
exit
exit
exit
w

# Скачать пакет с nslookup на BR-SRV, HQ-SRV, HQ-CLI
apt-get install bind-utils -y

# Проверка на CLI, BR-SRV и HQ-SRV
ping <полное доменное имя любой машины> или <IP-адрес>
nslookup <полное доменное имя любой машины> или <IP-адрес>

# Если на CLI не обновляется DNS-сервер - удалить каталог ens18 на CLI - выключить CLI и HQ-RTR - включить машины - создать каталог ens18 и файл options заново " mkdir ens18 && echo BOOTPROTO=dhcp > ens18/options && echo TYPE=eth >> ens18/options " - перезагрузить сеть " systemctl restart network "
```

#### 9.2 Какой должен быть вывод
```bash
hq-cli ifaces # nslookup br-rtr.net01tech.institute
Server:         192.168.10.2
Address:        192.168.10.2#53

Name:   br-rtr.net01tech.institute
Address: 10.0.0.1

hq-cli ifaces # nslookup 192.168.10.2
2.10.168.192.in-addr.arpa       name = hq-srv.net01tech.institute.

hq-cli ifaces # nslookup 192.168.20.2
2.20.168.192.in-addr.arpa       name = hq-cli.net01tech.institute.

hq-cli ifaces # ping br-rtr.net01tech.institute
PING br-rtr.net01tech.institute (10.0.0.1) 56(84) bytes of data.
64 bytes from 10.0.0.1 (10.0.0.1): icmp_seq=1 ttl=63 time=23.0 ms
64 bytes from 10.0.0.1 (10.0.0.1): icmp_seq=2 ttl=63 time=22.8 ms
64 bytes from 10.0.0.1 (10.0.0.1): icmp_seq=3 ttl=63 time=22.6 ms
```

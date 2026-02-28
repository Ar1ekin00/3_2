### 1. НАСТРОЙКА ISP
Сразу настроем на ISP всё, что нужно и больше к нему не возвращаемся.

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

#### 2.2 Настройка логических интерфейсов и сопряжение их с физическими портами
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

#### 2.3 Настройка GRE-туннель 
```bash
interface tunnel.0
ip mtu 1400
ip address 10.0.0.1/30
ip tunnel 172.16.2.2 172.16.1.2 mode gre
no shutdown
exit
```

#### 2.4 Настройка динамической маршрутизации (OSPF) на BR-RTR
```bash
router ospf 1
network 192.168.30.0 0.0.0.7 area 0
network 10.0.0.0 0.0.0.3 area 0
passive-interface default 
no passive-interface tunnel.0
area 0 authentication message-digest
exit
```

#### 2.5 Установка пароля на OSPF
```bash
interface int1
ip ospf authentication-key ecorouter
ip ospf message-digest-key 1 md5 ecorouter
exit
interface tunnel.0
ip ospf authentication-key ecorouter
ip ospf message-digest-key 1 md5 ecorouter
```

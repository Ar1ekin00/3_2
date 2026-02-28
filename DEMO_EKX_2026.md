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
conf t
hostname hq-rtr.net01tech.institute	

# Часовой пояс
ntp timezone utc+5

# Проверка времени
exit
show ntp date
```
#### 2.2 Настройка логических интерфейсов и сопряжение их с физическими портами



### 3. НАСТРОЙКА BR-RTR
#### 3.1 Настройка полного доменного имени, часового пояса
```bash
# Имя 
conf t
hostname br-rtr.net01tech.institute

# Часовой пояс
ntp timezone utc+5

# Проверка времени
exit
show ntp date
```
#### 2.2 Настройка логических интерфейсов и сопряжение их с физическими портами

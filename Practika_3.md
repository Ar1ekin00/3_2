# Задание для учебной практики УП.02.01 «Администрирование компьютерных сетей»

## 1. Вводные данные и технические параметры

Вы начинающий системный администратор, принятый в учебный центр. Ваша задача: спроектировать, развернуть и настроить базовую сетевую инфраструктуру предприятия на базе Linux-серверов. Стенд развёрнут в среде виртуализации Proxmox. Все конфигурации должны строго соответствовать приведённым ниже параметрам и быть подтверждены в итоговом отчёте. Все пароли для всех создаваемых учётных записей задать P@ssw0rd.

### 1.1. Таблица сетевой адресации и именования
| Имя хоста | FQDN | Назначение IP-адреса | Роль в инфраструктуре |
|:---|:---|:---|:---|
| `isp` | `isp.lab.local` | `172.16.0.1` | Шлюз, NAT, межсетевой экран, DHCP-сервер, NTP-сервер |
| `dc` | `dc.lab.local` | `172.16.0.10` | Samba DC, внутренний DNS, GPO, центр сертификации, NTP-клиент |
| `srv` | `srv.lab.local` | `172.16.0.20` | Веб-сервер, файловое хранилище (RAID5), Docker, NTP-клиент |
| `cli` | `cli.lab.local` | Из пула DHCP | Рабочая станция администратора, NTP-клиент |

**Дополнительные сетевые требования:**
- Подсеть локальной сети: `172.16.0.0/24`
- Шлюз по умолчанию для всех внутренних хостов: `172.16.0.1`
- DNS-суффикс поиска: `lab.local`
- DHCP-сервер на `isp` должен выдавать IP-адреса серверам (`dc`, `srv`) исключительно в режиме статической привязки по MAC-адресу. Динамический пул для рабочих станций и временных устройств должен быть ограничен диапазоном `172.16.0.200-172.16.0.250`.

### 1.2. Таблица параметров DNS-инфраструктуры
| Параметр | Значение / Требование |
|:---|:---|
| **Основная зона** | `lab.local` (тип: Primary, встроена в Samba DC) |
| **Обратная зона** | `0.16.172.in-addr.arpa` (тип: Primary) |
| **Базовые A-записи** | `dc.lab.local` → `172.16.0.10`, `srv.lab.local` → `172.16.0.20` |
| **Псевдонимы (CNAME)** | `moodle.lab.local` → `dc.lab.local` |
| **Псевдонимы (CNAME)** | `web.lab.local` → `srv.lab.local` |
| **Псевдонимы (CNAME)** | `docker.lab.local` → `srv.lab.local` |
| **PTR-записи** | Должны автоматически или вручную создаваться для всех A-записей в обратной зоне |
| **Рекурсия/Пересылка** | Разрешена только для сети `172.16.0.0/24`, внешние запросы перенаправлять на общедоступные DNS (на выбор) |

---

## 2. Поэтапные задания 

### 1: Подготовка виртуального стенда и базовая маршрутизация
- Присвойте всем системам сетевые имена (hostname) в полном доменном формате согласно таблице адресации.
- Настройте сетевые интерфейсы: `isp` должен иметь два адаптера (WAN с получением адреса по DHCP от внешнего провайдера и LAN, подключённый к внутреннему мосту); `dc`, `srv`, `cli` подключаются только к внутреннему мосту.
- Включите IP-форвардинг на `isp` и настройте базовую трансляцию сетевых адресов (NAT/Masquerade), обеспечив доступ внутренних узлов к сети Интернет.
- Убедитесь в физической и логической связности узлов на уровне L3. Проверьте базовую маршрутизацию и доступность внешних ресурсов.

**Для isp:**
```bash
hostnamectl set-hostname isp.lab.local
exec bash
cd /etc/net/ifaces/
mkdir -p enp0s8
cd enp0s8
echo 172.16.0.1/24 > ipv4address
echo BOOTPROTO=static > options
echo TYPE=eth >> options
apt-get update
apt-get install -y iptables nano
```
В файле:
nano /etc/net/sysctl.conf
отредачить:
net.ipv4.ip_forward = 1
```bash
sysctl -p
iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
iptables-save -f /etc/sysconfig/iptables 
systemctl enable --now iptables
systemctl restart network
```
**Для dc:**
```bash
hostnamectl set-hostname dc.lab.local
exec bash
cd /etc/net/ifaces/
mkdir -p enp0s3
cd enp0s3
echo BOOTPROTO=dhcp > options
echo TYPE=eth >> options
```
**Для srv:**
```bash
hostnamectl set-hostname srv.lab.local
exec bash
cd /etc/net/ifaces/
mkdir -p enp0s3
cd enp0s3
echo BOOTPROTO=dhcp > options
echo TYPE=eth >> options
```
**Для cli:**
```bash
hostnamectl set-hostname cli.lab.local
exec bash
cd /etc/net/ifaces/
mkdir -p enp0s3 
cd enp0s3
echo BOOTPROTO=dhcp > options
echo TYPE=eth >> options
```

---

### 2: Настройка DHCP, SSH, Sudo и локальных учётных записей
- На `isp` разверните и настройте службу DHCP. Реализуйте выдачу IP-адресов для `dc` и `srv` посредством статических привязок по MAC-адресам. Проверьте получение адресов после перезагрузки сетевых служб или ВМ.
- На всех четырёх узлах создайте двух локальных пользователей: `admin` и `monitor`.
- Настройте политику `sudo`: `admin` должен иметь право выполнять любые команды от имени суперпользователя без запроса пароля. `monitor` должен иметь право выполнять без пароля только команды мониторинга состояния системы (`htop`, `df`, `free`, `journalctl`, `systemctl status *`). Использование `sudo` для любых других команд должно быть запрещено.
- Захарденьте конфигурацию SSH-сервера на всех узлах: измените порт прослушивания на `2222`, установите текстовый баннер «Authorized access only», ограничьте количество попыток аутентификации до двух за сессию, запретите прямой вход под учётной записью `root`, разрешите подключение по SSH исключительно для пользователей `admin` и `monitor`.
- Проверьте удалённое подключение с `cli` ко всем серверам по новому порту с использованием созданных учётных записей.

**Для isp:**
```bash
apt-get install -y dhcp-server 
cd /etc/dhcp/

В файле:
/etc/sysconfig/dhcpd
Записать:
DHCPDARGS=enp0s8


В файле:
nano dhcpd.conf
Записать:
ddns-update-style none;
subnet 172.16.0.0 netmask 255.255.255.0 {
  option routers 172.16.0.1;
  option subnet-mask 255.255.255.0;
  option domain-name “lab.local”;
  option domain-search “lab.local”;
  option domain-name-servers 172.16.0.10, 8.8.8.8, 8.8.4.4;
  range 172.16.0.200 172.16.0.250;
  max-lease-time 43200;
  default-lease-time 21600;
  option ntp-servers 172.16.0.1;
}
host srv {
hardware ethernet 08:00:27:2b:2a:a6;
fixed-address 172.16.0.20;
}
host dc {
hardware ethernet 08:00:27:2b:2a:a6;
fixed-address 172.16.0.10;
}

dhcpd -t
systemctl restart dhcpd
systemctl enable --now dhcpd
systemctl restart network
apt-get install –y sudo htop openssh
useradd -m -s /bin/bash admin
echo "admin:P@ssw0rd" | chpasswd
useradd -m -s /bin/bash monitor
echo "monitor:P@ssw0rd" | chpasswd
chmod 4755 /usr/bin/sudo

В файле:
EDITOR=nano visudo
Записать:
admin ALL=(root:ALL) NOPASSWD: ALL
monitor ALL=(root:ALL) NOPASSWD: /usr/bin/htop, /usr/bin/fd, /usr/bin/free, /usr/bin/journalctl, /usr/sbin/systemctl status *

В файле:
nano /etc/openssh/sshd_config
Записать:
Port 2222
Banner /etc/openssh/banner.txt
MaxAuthTries 2
PermitRootLogin no
AllowUsers monitor admin
PubkeyAuthentication yes
PasswordAuthentication yes 

echo «Authorized access only» > /etc/openssh/banner.txt
systemctl enable --now sshd
systemctl restart sshd
```

**Для dc:**
```bash
systemctl restart network
apt-get update
apt-get install –y sudo htop openssh nano
useradd -m -s /bin/bash admin
echo "admin:P@ssw0rd" | chpasswd
useradd -m -s /bin/bash monitor
echo "monitor:P@ssw0rd" | chpasswd
chmod 4755 /usr/bin/sudo

В файле:
EDITOR=nano visudo
Записать:
admin ALL=(root:ALL) NOPASSWD: ALL
monitor ALL=(root:ALL) NOPASSWD: /usr/bin/htop, /usr/bin/fd, /usr/bin/free, /usr/bin/journalctl, /usr/sbin/systemctl status *

В файле:
nano /etc/openssh/sshd_config
Записать:
Port 2222
Banner /etc/openssh/banner.txt
MaxAuthTries 2
PermitRootLogin no
AllowUsers monitor admin
PubkeyAuthentication yes
PasswordAuthentication yes 

echo «Authorized access only» > /etc/openssh/banner.txt
systemctl enable --now sshd
systemctl restart sshd
```

**Для srv:**
```bash
systemctl restart network
apt-get update
apt-get install –y sudo htop openssh nano
useradd -m -s /bin/bash admin
echo "admin:P@ssw0rd" | chpasswd
useradd -m -s /bin/bash monitor
echo "monitor:P@ssw0rd" | chpasswd
chmod 4755 /usr/bin/sudo

В файле:
EDITOR=nano visudo
Записать:
admin ALL=(root:ALL) NOPASSWD: ALL
monitor ALL=(root:ALL) NOPASSWD: /usr/bin/htop, /usr/bin/fd, /usr/bin/free, /usr/bin/journalctl, /usr/sbin/systemctl status *

В файле:
nano /etc/openssh/sshd_config
Записать:
Port 2222
Banner /etc/openssh/banner.txt
MaxAuthTries 2
PermitRootLogin no
AllowUsers monitor admin
PubkeyAuthentication yes
PasswordAuthentication yes 

echo «Authorized access only» > /etc/openssh/banner.txt
systemctl enable --now sshd
systemctl restart sshd
```

**Для cli:**
```bash
systemctl restart network
apt-get update
apt-get install –y sudo htop openssh nano
useradd -m -s /bin/bash admin
echo "admin:P@ssw0rd" | chpasswd
useradd -m -s /bin/bash monitor
echo "monitor:P@ssw0rd" | chpasswd
chmod 4755 /usr/bin/sudo

В файле:
EDITOR=nano visudo
Записать:
admin ALL=(root:ALL) NOPASSWD: ALL
monitor ALL=(root:ALL) NOPASSWD: /usr/bin/htop, /usr/bin/fd, /usr/bin/free, /usr/bin/journalctl, /usr/sbin/systemctl status *

В файле:
nano /etc/openssh/sshd_config
Записать:
Port 2222
Banner /etc/openssh/banner.txt
MaxAuthTries 2
PermitRootLogin no
AllowUsers monitor admin
PubkeyAuthentication yes
PasswordAuthentication yes 

echo «Authorized access only» > /etc/openssh/banner.txt
systemctl enable --now sshd
systemctl restart sshd
```

---

### 3: Автоматизация развёртывания с помощью Ansible
- Установите Ansible на рабочую станцию `cli`.
- Создайте рабочий каталог `/etc/ansible` и файл инвентаризации `inventory.ini`. Сформируйте логическую группу `servers`, включив в неё хосты `isp`, `dc`, `srv`.
- Настройте аутентификацию по SSH-ключам для пользователя `admin` с `cli` на все хосты группы `servers` для обеспечения беспарольного взаимодействия.
- Напишите плейбук `install_htop.yml`, целью которого является установка пакета `htop` на все узлы группы `servers`.
- Запустите плейбук и зафиксируйте успешное выполнение задачи на всех целевых серверах. Проверьте связность командой `ansible all -m ping`.

**Для cli:**
```bash
ssh-keygen
ssh-copy-id -p 2222 admin@172.16.0.1
ssh-copy-id -p 2222 admin@172.16.0.10
ssh-copy-id -p 2222 admin@172.16.0.20

В файле:
nano ~/.ssh/config
Записать:
Host isp
Hostname 172.16.0.1
Port 2222 
User admin
IdentityFile ~/.ssh/id_ed25519
Host srv
Hostname 172.16.0.20
Port 2222 
User admin
IdentityFile ~/.ssh/id_ed25519
Host dc
Hostname 172.16.0.10
Port 2222 
User admin
IdentityFile ~/.ssh/id_ed25519

apt-get install -y ansible
mkdir -p /etc/ansible
cd /etc/ansible

В файле:
nano inventory.ini
Записать:
[servers]
isp ansible_host=172.16.0.1
dc ansible_host=172.16.0.10
srv ansible_host=172.16.0.20
[servers:vars]
ansible_user=admin
ansible_password=P@ssw0rd
ansible_port=2222

В файле:
nano install_htop.yml
Записать:
---
- name: установка htop на servers
  hosts: servers
  become: yes
  tasks:
    - name: обновить пакеты
      shell: apt-get update
    - name: установить htop
      shell: apt-get install -y htop

ansible-playbook -i inventory.ini install_htop.yml
(перед проверкой можно удалить htop с машин, запустить ansible и проверить на машинах наличие htop)

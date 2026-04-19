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

### Для isp:
**Команды:**
```bash
hostnamectl set-hostname isp.lab.local
exec bash
```
```bash
cd /etc/net/ifaces/
mkdir -p ens19
cd ens19
echo 172.16.0.1/24 > ipv4address
echo BOOTPROTO=static > options
echo TYPE=eth >> options
apt-get update && apt-get install -y iptables nano
```
*В файле:*
```bash
nano /etc/net/sysctl.conf
```
*Записать:*
```bash
net.ipv4.ip_forward = 1
```

**Команды:**
```bash
sysctl -p
iptables -t nat -A POSTROUTING -o ens18 -j MASQUERADE
iptables-save -f /etc/sysconfig/iptables 
systemctl enable --now iptables
systemctl restart network
```

### Для dc:
**Команды:**
```bash
hostnamectl set-hostname dc.lab.local
exec bash
```
```bash
cd /etc/net/ifaces/
mkdir -p ens18
cd ens18
echo BOOTPROTO=dhcp > options
echo TYPE=eth >> options
```
### Для srv:
**Команды:**
```bash
hostnamectl set-hostname srv.lab.local
exec bash
```
```bash
cd /etc/net/ifaces/
mkdir -p ens18
cd ens18
echo BOOTPROTO=dhcp > options
echo TYPE=eth >> options
```
### Для cli:
**Команды:**
```bash
hostnamectl set-hostname cli.lab.local
exec bash
```
```bash
cd /etc/net/ifaces/
mkdir -p ens18
cd ens18
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

### Для isp:
**Команды:**
```bash
apt-get install -y dhcp-server python3-module-setuptools python3-dev
cd /etc/dhcp/
```

*В файле:*
```bash
nano /etc/sysconfig/dhcpd
```

*Записать:*
```bash
DHCPDARGS=ens19
```

*В файле:*
```bash
nano dhcpd.conf
```
*Записать:*
```bash
ddns-update-style none;
subnet 172.16.0.0 netmask 255.255.255.0 {
  option routers         172.16.0.1;
  option subnet-mask     255.255.255.0;
  option domain-name "lab.local";
  option domain-search "lab.local";
  option domain-name-servers     172.16.0.10, 8.8.8.8, 8.8.4.4;
  range 172.16.0.200 172.16.0.250;
  max-lease-time 43200;
  default-lease-time 21600;
  option ntp-servers 172.16.0.1;
}
host srv {
hardware ethernet <свой mac-адрес>;
fixed-address 172.16.0.20;
}
host dc {
hardware ethernet <свой mac-адрес>;
fixed-address 172.16.0.10;
```
MAC-адрес отображается жёлтым, команда ip -c a 

**Команды:**
```bash
dhcpd -t
systemctl restart dhcpd
systemctl enable --now dhcpd
systemctl restart network
apt-get -y install sudo htop openssh
useradd -m -s /bin/bash admin
echo "admin:P@ssw0rd" | chpasswd
useradd -m -s /bin/bash monitor
echo "monitor:P@ssw0rd" | chpasswd
chmod 4755 /usr/bin/sudo
```

*В файле:*
```bash
EDITOR=nano visudo
```
*Записать:*
```bash
admin ALL=(root:ALL) NOPASSWD: ALL
monitor ALL=(root:ALL) NOPASSWD: /usr/bin/htop, /usr/bin/df, /usr/bin/free, /usr/bin/journalctl, /usr/sbin/systemctl status *
```
*В файле:*
```bash
nano /etc/openssh/sshd_config
```
*Записать:*
```bash
Port 2222
Banner /etc/openssh/banner.txt
MaxAuthTries 2
PermitRootLogin no
AllowUsers monitor admin
PubkeyAuthentication yes
PasswordAuthentication yes 
```
**Команды:**
```bash
echo «Authorized access only» > /etc/openssh/banner.txt
systemctl enable --now sshd
systemctl restart sshd
```

### Для dc:
**Команды:**
```bash
systemctl restart network
apt-get update
apt-get -y install sudo htop openssh nano python3-module-setuptools python3-dev
useradd -m -s /bin/bash admin
echo "admin:P@ssw0rd" | chpasswd
useradd -m -s /bin/bash monitor
echo "monitor:P@ssw0rd" | chpasswd
chmod 4755 /usr/bin/sudo
```
*В файле:*
```bash
EDITOR=nano visudo
```
*Записать:*
```bash
admin ALL=(root:ALL) NOPASSWD: ALL
monitor ALL=(root:ALL) NOPASSWD: /usr/bin/htop, /usr/bin/df, /usr/bin/free, /usr/bin/journalctl, /usr/sbin/systemctl status *
```

*В файле:*
```bash
nano /etc/openssh/sshd_config
```
*Записать:*
```bash
Port 2222
Banner /etc/openssh/banner.txt
MaxAuthTries 2
PermitRootLogin no
AllowUsers monitor admin
PubkeyAuthentication yes
PasswordAuthentication yes 
```
**Команды:**
```bash
echo «Authorized access only» > /etc/openssh/banner.txt
systemctl enable --now sshd
systemctl restart sshd
```

### Для srv:
**Команды:**
```bash
systemctl restart network
apt-get update
apt-get -y install sudo htop openssh nano python3-module-setuptools python3-dev
useradd -m -s /bin/bash admin
echo "admin:P@ssw0rd" | chpasswd
useradd -m -s /bin/bash monitor
echo "monitor:P@ssw0rd" | chpasswd
chmod 4755 /usr/bin/sudo
```

*В файле:*
```bash
EDITOR=nano visudo
```
*Записать:*
```bash
admin ALL=(root:ALL) NOPASSWD: ALL
monitor ALL=(root:ALL) NOPASSWD: /usr/bin/htop, /usr/bin/df, /usr/bin/free, /usr/bin/journalctl, /usr/sbin/systemctl status *
```
*В файле:*
```bash
nano /etc/openssh/sshd_config
```
*Записать:*
```bash
Port 2222
Banner /etc/openssh/banner.txt
MaxAuthTries 2
PermitRootLogin no
AllowUsers monitor admin
PubkeyAuthentication yes
PasswordAuthentication yes 
```
**Команды:**
```bash
echo «Authorized access only» > /etc/openssh/banner.txt
systemctl enable --now sshd
systemctl restart sshd
```

### Для cli:
**Команды:**
```bash
systemctl restart network
apt-get update
apt-get -y install sudo htop openssh nano python3-module-setuptools python3-dev
useradd -m -s /bin/bash admin
echo "admin:P@ssw0rd" | chpasswd
useradd -m -s /bin/bash monitor
echo "monitor:P@ssw0rd" | chpasswd
chmod 4755 /usr/bin/sudo
```
*В файле:*
```bash
EDITOR=nano visudo
```
*Записать:*
```bash
admin ALL=(root:ALL) NOPASSWD: ALL
monitor ALL=(root:ALL) NOPASSWD: /usr/bin/htop, /usr/bin/df, /usr/bin/free, /usr/bin/journalctl, /usr/sbin/systemctl status *
```
*В файле:*
```bash
nano /etc/openssh/sshd_config
```
*Записать:*
```bash
Port 2222
Banner /etc/openssh/banner.txt
MaxAuthTries 2
PermitRootLogin no
AllowUsers monitor admin
PubkeyAuthentication yes
PasswordAuthentication yes 
```
**Команды:**
```bash
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

### Для cli:
**Команды:**
```bash
ssh-keygen
```
```bash
ssh-copy-id -p 2222 admin@172.16.0.1
```
```bash
ssh-copy-id -p 2222 admin@172.16.0.10
```
```bash
ssh-copy-id -p 2222 admin@172.16.0.20
```
*В файле:*
```bash
nano ~/.ssh/config
```
*Записать:*
```bash
Host isp
Hostname 172.16.0.1
Port 2222
User admin
IdentityFile ~/.ssh/id_rsa    
Host srv
Hostname 172.16.0.20
Port 2222
User admin
IdentityFile ~/.ssh/id_rsa    
Host dc
Hostname 172.16.0.10
Port 2222
User admin
IdentityFile ~/.ssh/id_rsa
```
**Команды:**
```bash
apt-get -y install ansible sshpass 
mkdir -p /etc/ansible
cd /etc/ansible
```
*В файле:*
```bash
nano inventory.ini
```
*Записать:*
```bash
[servers]
isp ansible_host=172.16.0.1
dc ansible_host=172.16.0.10
srv ansible_host=172.16.0.20
[servers:vars]
ansible_user=admin
ansible_password=P@ssw0rd
ansible_port=2222
```
*В файле:*
```bash
nano install_htop.yml
```
*Записать:*
```bash
---
- name: установка htop на servers
  hosts: servers
  become: yes
  tasks:
    - name: обновить пакеты
      shell: apt-get update
    - name: установить htop
      shell: apt-get install -y htop
```
**Команды:**
```bash
ansible-playbook -i inventory.ini install_htop.yml
```
(перед проверкой можно удалить htop с машин, запустить ansible и проверить на машинах наличие htop)

---

### 4: Настройка DNS-инфраструктуры и синхронизации времени
- Настройте службу синхронизации сетевого времени на базе сервиса `chrony` на шлюзе `isp`. В качестве вышестоящего сервера NTP укажите общедоступный пул времени. Установите стратум сервера равным 5.
- В качестве клиентов NTP настройте серверы `dc` и `srv`, а также рабочую станцию `cli`. Убедитесь в успешной синхронизации времени на всех узлах инфраструктуры.
- На сервере `dc` настройте встроенный DNS-сервер Samba: убедитесь в корректности прямой и обратной зон для `lab.local`. Создайте псевдонимы (CNAME-записи) `moodle.lab.local`, `web.lab.local` и `docker.lab.local`, указывающие на соответствующие базовые A-записи серверов `dc` и `srv`. 
- (*) Настройте обратную зону для подсети `172.16.0.0/24` и добавьте корректные PTR-записи.
- Проверьте корректность прямого и обратного разрешения имён с узла `cli` с использованием `nslookup` или `dig`.

### Для isp (NTP - синхронизация времени):
**Команды:**
```bash
apt-get -y install chrony ntpdate
```
*В файле:*
```bash
nano /etc/chrony.conf 
```
*Записать:*
```bash
pool pool.ntp.org iburst
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
hwtimestamp *
allow 172.16.0.0/24
local stratum 5
ntsdumpdir /var/lib/chrony
logdir /var/log/chrony
log measurements statistics tracking
```
**Команды:**
```bash
systemctl enable --now chronyd
systemctl restart network
systemctl restart chronyd
ntpdate -q 172.16.0.1
chronyc tracking
chronyc sources -v
```
### Для dc, srv, cli (NTP - синхронизация времени):
**Команды:**
```bash
apt-get -y install chrony ntpdate
```
*В файле:*
```bash
nano /etc/chrony.conf 
```
*Записать:*
```bash
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
hwtimestamp *
ntsdumpdir /var/lib/chrony
logdir /var/log/chrony
server 172.16.0.1
```
Закомментировать строку ->
```bash
#pool pool.ntp.org iburst
```
**Команды:**
```bash
systemctl enable --now chronyd
systemctl restart network
systemctl restart chronyd
```
(Команды для проверки, для отчёта) ->
```bash
ntpdate -q 172.16.0.1
chronyc tracking
chronyc sources -v
```
---

### 5: Контроллер домена Samba DC и групповые политики
- На `dc` установите и проведите инициализацию Samba DC. Создайте домен `lab.local`. Убедитесь, что встроенный DNS-сервер Samba корректно обслуживает внутренние зоны.
- Создайте базовую организационную структуру:
  - Подразделения (OU): `admins`, `others`, `managers`
  - Пользователи: `ivanov` (поместить в OU `admins`), `petrov` и `sidorov` (поместить в OU `managers`)
  - Группы безопасности: `admins`, `managers`
  - Введите пользователя `ivanov` в группу `admins`, пользователей `petrov` и `sidorov` в группу `managers`
- Настройте и примените хотя бы одну групповую политику (GPO) для управления параметрами безопасности и рабочим столом доменных узлов.
- Проверьте разрешение имён и доменную аутентификацию с рабочей станции `cli`. Убедитесь, что клиент успешно введён в домен.
### Для dc (настройка samba):
**Команды:**
```bash
apt-get install -y samba krb5-workstation task-samba-dc bind-utils
systemctl disable --now smb nmb winbind
rm -f /etc/samba/smb.conf
rm -rf /var/lib/samba/*
rm -rf /var/cache/samba/*
samba-tool domain provision
```
(Везде жмёшь enter, кроме пункта DNS forwarder IP address - здесь пиши "8.8.8.8", а далее вводишь пароль "P@ssw0rd")
```bash
cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
systemctl enable --now samba
```
(Команды для проверки зон, для отчёта) ->
```bash
samba-tool domain info 172.16.0.10
samba-tool dns zonelist 172.16.0.10 -U Administrator
```
```bash
samba-tool ou create "OU=admins,DC=lab,DC=local"
samba-tool ou create "OU=others,DC=lab,DC=local"
samba-tool ou create "OU=managers,DC=lab,DC=local"
samba-tool group add admins --group-type=Security
samba-tool group add managers --group-type=Security
samba-tool user create ivanov 'P@ssw0rd'
samba-tool user create petrov 'P@ssw0rd'
samba-tool user create sidorov 'P@ssw0rd'
samba-tool user move ivanov "OU=admins,DC=lab,DC=local"
samba-tool user move petrov "OU=managers,DC=lab,DC=local"
samba-tool user move sidorov "OU=managers,DC=lab,DC=local"
samba-tool group addmembers admins ivanov
samba-tool group addmembers managers petrov,sidorov
samba-tool user enable ivanov
samba-tool user enable petrov
samba-tool user enable sidorov
```

(Команды для проверки групп, пользователей, для отчёта) ->
```bash
samba-tool group listmembers admins
samba-tool group listmembers managers
samba-tool user show ivanov --attributes=distinguishedName
samba-tool user show petrov --attributes=distinguishedName
samba-tool user show sidorov --attributes=distinguishedName
```
### Для cli (настройка samba):
**Команды:**
```bash
apt-get -y install task-auth-ad-sssd bind-utils alterator-auth alterator-gpupdate admc gpui gpupdate
```
Далее заходишь в cli с графической оболочкой, открываешь центр управления системой -> аутентификация -> выбираешь Active Directory or ALT Domain, домен уже будет указан верный -> Применить
```bash
kinit Administrator@LAB.LOCAL
kinit ivanov@LAB.LOCAL
kinit petrov@LAB.LOCAL
kinit sidorov@LAB.LOCAL
```
---

### 6: Политика межсетевого экрана на шлюзе
- Настройте расширенную политику межсетевого экрана на сервере `isp`. Разрешите исходящий трафик по протоколам HTTP, HTTPS, DNS, NTP, ICMP.
- Разрешите входящий трафик только в состоянии `ESTABLISHED,RELATED`.
- Запретите все остальные входящие подключения из внешней сети во внутреннюю сеть `172.16.0.0/24`.
- Активируйте и сохраните правила брандмауэра. Проверьте работоспособность фильтрации трафика и внесите основные параметры настройки в отчёт.

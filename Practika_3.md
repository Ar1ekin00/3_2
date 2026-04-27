

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

## 📑 Оглавление

1. [Подготовка виртуального стенда и базовая маршрутизация](#ch1)
2. [Настройка DHCP, SSH, Sudo и локальных учётных записей](#ch2)
3. [Автоматизация развёртывания с помощью Ansible](#ch3)
4. [Настройка синхронизации времени](#ch4)
5. [Контроллер домена Samba DC и групповые политики](#ch5)
4,5. [DNS-инфраструктура (продолжение 4 шага)](#ch6)
6. [Политика межсетевого экрана на шлюзе](#ch7)
7. [Развёртывание веб-приложения в Docker на сервере `srv`](#ch8)
8. [Развёртывание веб-приложения на сервере `srv`](#ch9)
9. [Настройка отказоустойчивого файлового хранилища и сетевых шар](#ch10)
10. [Центр сертификации, HTTPS](#ch11)

---

## 2. Поэтапные задания 

<a name="ch1"></a>
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

<a name="ch2"></a>
### 2: Настройка DHCP, SSH, Sudo и локальных учётных записей
- На `isp` разверните и настройте службу DHCP. Реализуйте выдачу IP-адресов для `dc` и `srv` посредством статических привязок по MAC-адресам. Проверьте получение адресов после перезагрузки сетевых служб или ВМ.
- На всех четырёх узлах создайте двух локальных пользователей: `admin` и `monitor`.
- Настройте политику `sudo`: `admin` должен иметь право выполнять любые команды от имени суперпользователя без запроса пароля. `monitor` должен иметь право выполнять без пароля только команды мониторинга состояния системы (`htop`, `df`, `free`, `journalctl`, `systemctl status *`). Использование `sudo` для любых других команд должно быть запрещено.
- Захарденьте конфигурацию SSH-сервера на всех узлах: измените порт прослушивания на `2222`, установите текстовый баннер «Authorized access only», ограничьте количество попыток аутентификации до двух за сессию, запретите прямой вход под учётной записью `root`, разрешите подключение по SSH исключительно для пользователей `admin` и `monitor`.
- Проверьте удалённое подключение с `cli` ко всем серверам по новому порту с использованием созданных учётных записей.

### Для isp:
**Команды:**
```bash
apt-get install -y dhcp-server python3-module-setuptools python3-dev python3-module-distutils-extra
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
}
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
monitor ALL=(root:ALL) NOPASSWD: /usr/bin/htop, /usr/bin/df, /usr/bin/free, /usr/bin/journalctl, /usr/bin/systemctl status *
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

### Для dc, srv:
**Команды:**
```bash
systemctl restart network
apt-get update
apt-get -y install sudo htop openssh nano python3-module-setuptools python3-dev python3-module-distutils-extra
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
monitor ALL=(root:ALL) NOPASSWD: /usr/bin/htop, /usr/bin/df, /usr/bin/free, /usr/bin/journalctl, /usr/bin/systemctl status *
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
apt-get -y install sudo htop openssh nano python3-module-setuptools python3-dev python3-module-distutils-extra
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
monitor ALL=(root:ALL) NOPASSWD: /usr/bin/htop, /bin/df, /usr/bin/free, /sbin/journalctl, /sbin/systemctl status *
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

<a name="ch3"></a>
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
    - name: обновить кэш пакетов
      ansible.builtin.package:
        update_cache: yes
      

    - name: установить htop
      ansible.builtin.package:
        name: htop
        state: present
```
**Команды:**
```bash
ansible-playbook -i inventory.ini install_htop.yml
```
(перед проверкой можно удалить htop с машин, запустить ansible и проверить на машинах наличие htop)

---

<a name="ch4"></a>
### 4: Настройка синхронизации времени
- Настройте службу синхронизации сетевого времени на базе сервиса `chrony` на шлюзе `isp`. В качестве вышестоящего сервера NTP укажите общедоступный пул времени. Установите стратум сервера равным 5.
- В качестве клиентов NTP настройте серверы `dc` и `srv`, а также рабочую станцию `cli`. Убедитесь в успешной синхронизации времени на всех узлах инфраструктуры.

### Для isp (NTP - синхронизация времени):
**Команды:**
```bash
apt-get -y install chrony ntpdate
rm -rf /etc/chrony.conf
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
sleep 10
ntpdate -q 172.16.0.1
chronyc tracking
chronyc sources -v
```
### Для dc, srv, cli (NTP - синхронизация времени):
**Команды:**
```bash
apt-get -y install chrony ntpdate
rm -rf /etc/chrony.conf
```
*В файле:*
```bash
nano /etc/chrony.conf 
```
*Записать:*
```bash
server 172.16.0.1 iburst
```
**Команды:**
```bash
systemctl enable --now chronyd
sleep 10
```
(Команды для проверки, для отчёта) ->
```bash
ntpdate -q 172.16.0.1
chronyc tracking
chronyc sources -v
```
---

<a name="ch5"></a>
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
apt-get -y remove alterator-datetime
apt-get -y install task-auth-ad-sssd bind-utils alterator-auth alterator-gpupdate admc gpui gpupdate gpresult 
```
Далее заходишь в cli с графической оболочкой, открываешь system management center -> authentication -> выбираешь Active Directory or ALT Domain, домен уже будет указан верный -> Apply

Для групповых политик открываешь system management center -> Group policy -> поставить галочку под пунктом "Group policy Management", ниже Current group policy profile: галочку под Active Directory Domain Controller -> Apply

```bash
kinit Administrator@LAB.LOCAL
```
```bash
kinit ivanov@LAB.LOCAL
```
```bash
kinit petrov@LAB.LOCAL
```
```bash
kinit sidorov@LAB.LOCAL
```
Для установки групповой политики: 
1. Пуск -> 
2. В поиске "ADMC" -> 
3. Заходишь за пользователя administrator -> 
4. В левом окне "Group policy Objects" ->
5. lab.local ->
6. Здесь есть папки "managers" и "admins" ->
7. Нажать ПКМ по одной из них (admins, например) и в открывшемся списке выбрать "Create a GPO and link to this OU" - имя для групповой политики ставь какое хочешь ->
8. Нажать ПКМ на папку "managers" и выбрать "Link existing GPO" - выбираешь созданную тобой ранее политику ->
9. Нажать ЛКМ на созданную политику - в левом окне включить галочку под Enforced для обеих групп пользователей ->
10. Далее нажимаешь ПКМ по этой же политике ->
11. edit (чтобы изменить групповую политику) ->
12. User ->
13. Administrative Templates ->
14. ALT System ->
15. Mate settings ->
16. Screensaver в этой папке есть несколько политик, которые нужно включить:

Enable screen saver -> галочку под Enabled, а в самом низу OK.  (при бездействии включает заставку) 

Lock your computer -> галочку под Enabled, в самом низу OK. (Блокировка компьютера)

Logout after being locked -> галочку под Enabled, в самом низу OK.

Operating mode -> галочку под Enabled, в options выбрать "Random themes", самом низу OK.

Switch user after blocking -> галочку под Disabled, в самом низу OK.

Для отчёта напиши, типа "Настроил именно эти политики, для большей безопасности данных. Если владельца долго нет за ПК, то постороннние люди не смогут им воспользоваться, так как для разблокировки требуется пароль пользователя"

Под root:
```bash
gpupdate
reboot
```

(Команды для проверки групповых политик, для отчёта) ->
```bash
gpresult
```

---

<a name="ch6"></a>
### 4: DNS-инфраструктура (продолжение 4 шага)
- На сервере `dc` настройте встроенный DNS-сервер Samba: убедитесь в корректности прямой и обратной зон для `lab.local`. Создайте псевдонимы (CNAME-записи) `moodle.lab.local`, `web.lab.local` и `docker.lab.local`, указывающие на соответствующие базовые A-записи серверов `dc` и `srv`. 
- (*) Настройте обратную зону для подсети `172.16.0.0/24` и добавьте корректные PTR-записи.
- Проверьте корректность прямого и обратного разрешения имён с узла `cli` с использованием `nslookup` или `dig`.
### Для dc:
**Команды:**
```bash
kinit Administrator@LAB.LOCAL
```
```bash
samba-tool dns add dc.lab.local lab.local srv A 172.16.0.20 -U "LAB.LOCAL\Administrator%P@ssw0rd"
samba-tool dns add dc.lab.local lab.local moodle.lab.local CNAME dc.lab.local -U "LAB.LOCAL\Administrator%P@ssw0rd"
samba-tool dns add dc.lab.local lab.local web.lab.local CNAME srv.lab.local -U "LAB.LOCAL\Administrator%P@ssw0rd"
samba-tool dns add dc.lab.local lab.local docker.lab.local CNAME srv.lab.local -U "LAB.LOCAL\Administrator%P@ssw0rd"
samba-tool dns zonecreate dc.lab.local 0.16.172.in-addr.arpa -U "LAB.LOCAL\Administrator%P@ssw0rd"
samba-tool dns add dc.lab.local 0.16.172.in-addr.arpa 10 PTR dc.lab.local -U "LAB.LOCAL\Administrator%P@ssw0rd"
samba-tool dns add dc.lab.local 0.16.172.in-addr.arpa 20 PTR srv.lab.local -U "LAB.LOCAL\Administrator%P@ssw0rd"
```
*В файле:*
```bash
nano /etc/samba/smb.conf
```
*Записать:*
```bash
[global]
   dns forwarder = 8.8.8.8
   allow dns updates = secure only
```


(Команды для проверки DNS) ->
### Для cli:
**Команды:**
Прямое разрешение CNAME:
```bash
sudo apt-get install bind-utils
nslookup moodle.lab.local
nslookup web.lab.local
nslookup docker.lab.local
```

Обратное (PTR):
```bash
nslookup 172.16.0.10
nslookup 172.16.0.20
```
dig:
```bash
dig moodle.lab.local CNAME +short
dig web.lab.local CNAME +short
dig -x 172.16.0.10 PTR +short
```

---
<a name="ch7"></a>
### 6: Политика межсетевого экрана на шлюзе
- Настройте расширенную политику межсетевого экрана на сервере `isp`. Разрешите исходящий трафик по протоколам HTTP, HTTPS, DNS, NTP, ICMP.
- Разрешите входящий трафик только в состоянии `ESTABLISHED,RELATED`.
- Запретите все остальные входящие подключения из внешней сети во внутреннюю сеть `172.16.0.0/24`.
- Активируйте и сохраните правила брандмауэра. Проверьте работоспособность фильтрации трафика и внесите основные параметры настройки в отчёт.
### Для isp:
**Команды:**
```bash
iptables -F
iptables -t nat -A POSTROUTING -o ens18 -j MASQUERADE
iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -i ens18 -p tcp --dport 2222 -j ACCEPT
iptables -A INPUT -i ens19 -p tcp --dport 2222 -j ACCEPT
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A FORWARD -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A OUTPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -p icmp -j ACCEPT
iptables -A INPUT -i ens19 -p udp --dport 123 -j ACCEPT
iptables -A OUTPUT -p tcp --dport 80 -j ACCEPT
iptables -A OUTPUT -p tcp --dport 443 -j ACCEPT
iptables -A OUTPUT -p tcp --dport 53 -j ACCEPT
iptables -A OUTPUT -p udp --dport 53 -j ACCEPT
iptables -A OUTPUT -p udp --dport 123 -j ACCEPT
iptables -A OUTPUT -p icmp -j ACCEPT
iptables -A FORWARD -p tcp --dport 80 -j ACCEPT
iptables -A FORWARD -p tcp --dport 443 -j ACCEPT
iptables -A FORWARD -p tcp --dport 53 -j ACCEPT
iptables -A FORWARD -p udp --dport 53 -j ACCEPT
iptables -A FORWARD -p udp --dport 123 -j ACCEPT
iptables -A FORWARD -p icmp -j ACCEPT
iptables -A FORWARD -s 172.16.0.0/24 -j ACCEPT
iptables -A FORWARD -d 172.16.0.0/24 ! -s 172.16.0.0/24 -j DROP
iptables -P FORWARD DROP
iptables -P INPUT DROP
iptables -P OUTPUT DROP
iptables-save -f /etc/sysconfig/iptables
systemctl restart network && systemctl restart iptables
```
(Команды для проверки правил, для отчёта) ->
```bash
iptables -n -L
```

---

<a name="ch8"></a>
### 7: (**) Развёртывание веб-приложения в Docker на сервере `srv`
- Средствами Docker должен создаваться стек контейнеров с веб-приложением и базой данных.
- Используйте образы `site_latest` и `mariadb_latest`, располагающиеся в директории `docker` в образе `Additional.iso`.
- Основной контейнер `testapp` должен называться `testapp`.
- Контейнер с базой данных должен называться `db`.
- Импортируйте образы в Docker, укажите в YAML-файле параметры подключения к СУБД: имя БД — `testdb`, пользователь `test` с паролем `P@ssw0rd`, порт приложения `8080`. При необходимости настройте другие параметры.
- Приложение должно быть доступно для внешних подключений через порт `8080`.

### Для srv:
**Команды:**
Для тех, кто сделал по сырой версии - сначала удалить прошлые образы, контейнеры и т.д.:
```bash
docker stop $(docker ps -aq)
docker rm $(docker ps -aq)
docker rmi $(docker images -q)
docker volume rm $(docker volume ls -q)
docker network rm $(docker network ls -q | grep -v "bridge\|host\|none")
```
```bash
apt-get install -y docker-engine net-tools
systemctl enable --now docker
curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
mkdir -p /mnt/iso
mount /dev/sr0 /mnt/iso
cp -r /mnt/iso /opt/testapp/
docker load -i /mnt/iso/docker/site_latest.tar
docker load -i /mnt/iso/docker/mariadb_latest.tar
docker tag site:latest site_latest:latest
docker tag mariadb:10.11 mariadb_latest:latest
docker rmi site:latest
docker rmi mariadb:10.11
mkdir -p /opt/testapp
cd /opt/testapp
```
*В файле:*
```bash
nano docker-compose.yml
```
*Записать:*
```bash
version: '3.8'

services:
  db:
    image: mariadb_latest:latest
    container_name: db
    environment:
      MYSQL_DATABASE: testdb
      MYSQL_USER: test
      MYSQL_PASSWORD: P@ssw0rd
      MYSQL_ROOT_PASSWORD: toor
    volumes:
      - db_data:/var/lib/mysql
    restart: unless-stopped
    networks:
      - appnet

  testapp:
    image: site_latest:latest
    container_name: testapp
    ports:
      - "8080:8000"
    environment:
      DB_TYPE: maria
      DB_HOST: db
      DB_NAME: testdb
      DB_USER: test
      DB_PASS: P@ssw0rd
      DB_PORT: 3306
    depends_on:
      - db
    restart: unless-stopped
    networks:
      - appnet

volumes:
  db_data:

networks:
  appnet:
    driver: bridge
```
**Команды:**
```bash
docker-compose up -d
docker-compose ps
curl -I http://localhost:8080
```
### Для cli:
(Проверка работоспособности сайта, для отчёта) -> на cli заходишь в браузер, вводишь в поисковик "172.16.0.20:8080" или "web.lab.local:8080" зайдёшь на сайт

---

<a name="ch9"></a>
### 8: (*) Развёртывание веб-приложения на сервере `srv` 
- Используйте веб-сервер Apache. В качестве системы управления базами данных используйте MariaDB.
- Файлы веб-приложения и дамп базы данных находятся в директории `web` образа `Additional.iso`.
- Выполните импорт схемы и данных из файла `dump.sql` в базу данных `webdb`.
- Создайте пользователя `web` с паролем `P@ssw0rd` и предоставьте ему права доступа к этой базе данных.
- Файлы `index.php` и директорию `images` скопируйте в каталог веб-сервера Apache.
- В файле `index.php` укажите правильные учётные данные для подключения к БД.
- Запустите веб-сервер и убедитесь в работоспособности приложения.
- Основные параметры отметьте в отчёте.

### Для srv:
**Команды:**
```bash
apt-get install -y lamp-server
systemctl enable --now httpd2
systemctl enable --now mysqld
mkdir -p /mnt/iso
mkdir -p /opt/testapp
mount /dev/sr0 /mnt/iso
cp -r /mnt/iso /opt/testapp/
mkdir -p /var/www/html/testapp
cp /opt/testapp/iso/web/index.php /var/www/html/testapp/
cp /opt/testapp/iso/web/logo.png /var/www/html/testapp/
chown -R apache2:apache2 /var/www/html/testapp
chmod -R 644 /var/www/html/testapp
mysql -u root
```
```bash
CREATE DATABASE webdb CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'web'@'localhost' IDENTIFIED BY 'P@ssw0rd';
GRANT ALL PRIVILEGES ON webdb.* TO 'web'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```
После следующей команды, вводишь пароль P@ssw0rd
```bash
mariadb -u web -p webdb < /opt/testapp/iso/web/dump.sql
```

*В файле:*
```bash
nano /var/www/html/testapp/index.php
```
*Изменить в самом начале:*
```bash
$servername = "localhost";
$username = "web";
$password = "P@ssw0rd";
$dbname = "webdb";
```

**Команды:**
```bash
systemctl restart httpd2
chmod 755 /var/www /var/www/html /var/www/html/testapp
chmod 644 /var/www/html/testapp/index.php
```

### Для cli:
(Проверка работы web-приложения, для отчёта) -> на cli заходишь в браузер, вводишь в поисковик "http://web/testapp" зайдёшь на сайт

---

<a name="ch10"></a>
### 9: Настройка отказоустойчивого файлового хранилища и сетевых шар
- На `srv` подключите три дополнительных виртуальных диска объёмом не менее 1 ГБ каждый.
- Создайте программный массив уровня RAID5 с использованием штатных средств ОС. Имя устройства массива — `md0`. Конфигурация массива размещается в файле `/etc/mdadm.conf`.
- Отформатируйте массив в файловую систему `ext4`.
- Настройте автоматическое монтирование массива в директорию `/srv/storage` после перезагрузки системы.
- Создайте в `/srv/storage` каталоги: `instructions`, `share`, `secret`.
- Организуйте сетевой доступ к каталогам через Samba одним из способов:
  - без разделения прав(на все каталоги права rw). 
  - С (*)разделением прав доступа: 
    - `instructions` – только чтение для всех, кроме группы `admins` (чтение-запись).
    - `share` – чтение-запись для всех пользователей и групп домена.
    - `secret` – чтение-запись только для группы `admins`.
- (**) Добавьте в домен групповую политику, которая будет монтировать каталоги `instructions` и `share` для всех пользователей домена, а для подразделения `admins` дополнительно монтировать каталог `secret`.

### Для srv:
**Команды:**
```bash
apt-get update
apt-get install -y samba samba-client samba-winbind samba-winbind-clients krb5-workstation attr acl ctdb mdadm
systemctl stop smb nmb winbind
```
После ввода следующей команды, нажимаешь Y:
```bash
mdadm --create --verbose /dev/md0 --level=5 --raid-devices=3 /dev/vdb /dev/vdc /dev/vdd
```
```bash
mdadm --detail --scan >> /etc/mdadm.conf
mdadm --assemble --scan 
mkfs.ext4 /dev/md0
mkdir -p /srv/storage
mount /dev/md0 /srv/storage
UUID=$(blkid -s UUID -o value /dev/md0) && echo "UUID=$UUID /srv/storage ext4 defaults 0 0" >> /etc/fstab
mkdir -p /srv/storage/{instructions,share,secret}
chown -R root:root /srv/storage
chmod 755 /srv/storage
chmod 777 /srv/storage/share
chmod 755 /srv/storage/instructions  
chmod 700 /srv/storage/secret
echo "Public test file" > /srv/storage/share/public.txt
echo "Secret document" > /srv/storage/secret/secret.txt  
echo "Instructions readme" > /srv/storage/instructions/readme.txt

rm -rf /etc/krb5.conf
rm -rf /etc/samba/smb.conf

mkdir -p /var/log/samba/
mkdir -p /etc/samba/
mkdir -p /var/lib/samba /var/cache/samba
mkdir -p /var/run/samba /var/lib/samba/lock /var/lib/samba/printers
mkdir -p /var/lib/samba/private /var/lib/samba/msg.sock
mkdir -p /var/lib/samba/winbindd_privileged

chmod 0755 /var/lib/samba /var/cache/samba /var/log/samba
chmod 0700 /var/lib/samba/private
chmod 0750 /var/lib/samba/winbindd_privileged

sed -i 's/\(passwd:.*files.*\)/\1 winbind/' /etc/nsswitch.conf
sed -i 's/\(group:.*files.*\)/\1 winbind/' /etc/nsswitch.conf
```
*В файле:*
```bash
nano /etc/krb5.conf
```
*Записать:*
```bash
[libdefaults]
    default_realm = LAB.LOCAL
    dns_lookup_realm = false
    dns_lookup_kdc = false

[realms]
    LAB.LOCAL = {
        kdc = dc.lab.local
        admin_server = dc.lab.local
    }
[domain_realm]
    .lab.local = LAB.LOCAL
    lab.local = LAB.LOCAL
```
*В файле:*
```bash
nano /etc/samba/smb.conf
```
*Записать:*
```bash
[global]
   workgroup = LAB
   security = ADS
   realm = LAB.LOCAL
   
   idmap config LAB : backend = rid
   idmap config LAB : range = 10000-999999

   idmap config * : backend = tdb
   idmap config * : range = 3000-7999

   winbind use default domain = yes
   winbind enum users = yes
   winbind enum groups = yes
   winbind refresh tickets = yes

   
   server min protocol = NT1
   ntlm auth = yes
   vfs objects = acl_xattr
   map acl inherit = yes

# 1. instructions  ТОЛЬКО ЧТЕНИЕ (кроме admins RW)
[instructions]
   path = /srv/storage/instructions
   browsable = yes
   read only = no
   writable = yes                   
   valid users = @LAB\managers @LAB\admins
   admin users = @LAB\admins
   create mask = 0644
   directory mask = 0755

# 2. share – RW ДЛЯ ВСЕХ доменных пользователей
[share]
   path = /srv/storage/share
   browsable = yes
   writable = yes
   read only = no
   create mask = 0664
   directory mask = 0775

# 3. secret – ТОЛЬКО admins RW
[secret]
   path = /srv/storage/secret
   browsable = no
   writable = yes
   valid users = @LAB\admins
   admin users = @LAB\admins
   create mask = 0660
   directory mask = 0770
```
**Команды:**
```bash
net ads join -U administrator@LAB.LOCAL --no-dns-updates
systemctl start winbind
sleep 10  # Дать winbind время на инициализацию
systemctl start smb nmb
systemctl enable --now winbind smb nmb
```

### Для cli (монтирование каталогов instructions, share и secret):
**Команды:**
```bash
apt-get install pam_mount cifs-utils systemd-settings-enable-kill-user-processes
```
*В файле:*
```bash
nano /etc/pam.d/system-auth
```
*Записать в самый конец файла:*
```bash
session         [success=1 default=ignore] pam_succeed_if.so  service = systemd-user quiet
session         optional        pam_mount.so disable_interactive
```

*В файле:*
```bash
/etc/security/pam_mount.conf.xml
```
*Записать в конце ПЕРЕД строкой </pam_mount>. Последние строки файла должны выглядеть так:*
```bash
<mkmountpoint enable="1" remove="true" />

<volume uid="10000-400000000" fstype="cifs" server="srv.lab.local" path="share" mountpoint="~/share" options="sec=krb5i,cruid=%(USERUID),nounix,uid=%(USERUID),gid=%(USERGID),file_mode=0664,dir_mode=0775" />

<volume uid="10000-400000000" fstype="cifs" server="srv.lab.local" path="instructions" mountpoint="~/instructions" options="sec=krb5i,cruid=%(USERUID),nounix,uid=%(USERUID),gid=%(USERGID),file_mode=0644,dir_mode=0755" />

<volume uid="10000-400000000" fstype="cifs" server="srv.lab.local" path="secret" mountpoint="~/secret" sgrp="admins" options="sec=krb5i,cruid=%(USERUID),nounix,uid=%(USERUID),gid=%(USERGID),file_mode=0660,dir_mode=0770" />

</pam_mount>
```
**Команды:**
```bash
reboot
```

### Для cli:

(Для отчёта): 

-> зайти на cli через графическую оболочку, открыть домашнюю директорию (папка с именем пользователя на рабочем столе). 

-> в строке поиска написать "smb://srv/", тебя перенёсет в папку, где будут лежать каталоги. 
Важно, по заданию у пользователя ivanov есть доступ на чтение-запись во всех 3 каталогах, у пользователей sidorov и petrov есть права  ТОЛЬКО на чтение в каталоге "instructions" и на чтение-запись в каталоге share. Каталог secret не отображается ни для одного пользователя, открыть его может только пользователь ivanov через строку поиска "smb://srv/secret". 

-> Можете создать файл на cli в директории share, далее открыть терминал srv и ввести "smbclient //localhost/share -U ivanov" .

-> пароль P@ssw0rd

-> командой ls вывести содержимое каталога, в том числе, созданный файл. Или создать файл на srv, и он должен появится на cli. 

Через терминал можно зайти только в конкреный каталог!!! 

То есть команда "smbclient //localhost/share -U ivanov" - не работает. 
1. Либо "smbclient //localhost/share -U ivanov"
2. Либо "smbclient //localhost/instructions -U ivanov"
3. Либо "smbclient //localhost/secret -U ivanov"

P.S.
Если сделали монтирование сетевых каталогов, то они просто будут находится на рабочем столе пользователя

### Для srv:

КОМАНДЫ ВНУТРИ FTP-КЛИЕНТА (после команды smbclient //localhost/secret -U ivanov):

**Команды:**
```bash
smb: \> help          # показать все команды
smb: \> ?             # краткая справка

# Навигация
smb: \> ls            # список файлов (вы уже пробовали)
smb: \> dir           # то же что ls
smb: \> cd folder     # войти в папку  
smb: \> cd ..         # на уровень выше
smb: \> pwd           # текущая папка

# Работа с файлами
smb: \> get public.txt      # скачать public.txt локально
smb: \> get public.txt ./   # сохранить в текущую папку
smb: \> put local.txt       # загрузить local.txt на сервер

# Чтение файла
smb: \> more public.txt     # показать содержимое
smb: \> type public.txt     # то же что more

# Множественные файлы
smb: \> mget *.txt          # скачать все .txt
smb: \> mput *.txt          # загрузить все .txt
smb: \> prompt              # отключить подтверждения

# Локальные команды (на вашей машине)
smb: \> !ls                 # ls локальной папки  
smb: \> !pwd                # текущая локальная папка
smb: \> !cd /tmp            # сменить локальную папку
```

---

<a name="ch11"></a>
### 10: (**) Центр сертификации, HTTPS:
- На `dc` настройте локальный центр сертификации. Выпускаемые сертификаты должны быть действительны не менее 30 дней.
- Обеспечьте доверие к корневому сертификату для рабочей станции `cli` (установка в хранилище доверенных центров ОС и браузера).
- Выпустите сертификаты для веб-серверов `dc.lab.local` и `srv.lab.local`. **Обязательно включите в расширения SAN (Subject Alternative Name) созданные ранее псевдонимы:** `moodle.lab.local`, `web.lab.local`, `docker.lab.local`.
- Перенастройте веб-серверы и контейнерный прокси на работу по протоколу HTTPS с привязкой новых сертификатов. Настройте автоматическое перенаправление всех входящих HTTP-запросов на HTTPS с использованием кода статуса 301.
- Убедитесь, что при обращении к приложениям по именам `moodle.lab.local`, `web.lab.local` и `docker.lab.local` у браузера клиента не возникает предупреждений о недоверенном соединении.

### Для dc:
**Команды:**
```bash

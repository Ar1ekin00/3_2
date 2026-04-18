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

Для isp:
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
apt-get install iptables 

В файле:
nano /etc/net/sysctl.conf
отредачить:
net.ipv4.ip_forward = 1

sysctl -p
iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
iptables-save -f /etc/sysconfig/iptables 
systemctl enable --now iptables
systemctl restart network
```
Для dc:
```bash
hostnamectl set-hostname dc.lab.local
exec bash
cd /etc/net/ifaces/
mkdir -p enp0s3
cd enp0s3
echo BOOTPROTO=dhcp > options
echo TYPE=eth >> options
```
Для srv:
```bash
hostnamectl set-hostname srv.lab.local
exec bash
cd /etc/net/ifaces/
mkdir -p enp0s3
cd enp0s3
echo BOOTPROTO=dhcp > options
echo TYPE=eth >> options
```
Для cli:
```bash
hostnamectl set-hostname cli.lab.local
exec bash
cd /etc/net/ifaces/
mkdir -p enp0s3
cd enp0s3
echo BOOTPROTO=dhcp > options
echo TYPE=eth >> options
```

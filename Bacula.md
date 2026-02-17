## Методические указания: Настройка резервного копирования с использованием Bacula в среде ALT Linux

**Цель работы:** Освоить развертывание распределенной системы резервного копирования Bacula с защитой от вирусов-шифровальщиков в изолированной лабораторной среде.

---

### 1. Подготовительный этап: Настройка сети

ISP имеет локальный адрес 192.168.1.1

SRV имеет ip адрес 192.168.1.10

CLI имеет ip адрес 192.168.1.20

---

#### 1.1. Изменение имён машин

```bash
# Указание новых наименований машин
hostnamectl set-hostname SRV

hostnamectl set-hostname CLI

hostnamectl set-hostname ISP

# Замена текущего bash процесса на новый, чтобы увидеть изменения
exec bash
```
#### 1.2. Настройка локальной сети между машинами

```bash
# Создаём файлы маршрута по умолчанию, DNS-сервера и настройки конфигурации соответственно  
На SRV и CLI:
echo default via 192.168.1.1 > /etc/net/ifaces/ens18/ipv4route
echo nameserver 8.8.8.8 > /etc/net/ifaces/ens18/resolv.conf
echo BOOTPROTO=static > /etc/net/ifaces/ens18/options
echo TYPE=eth >> /etc/net/ifaces/ens18/options

# Настраиваем IP-адреса
на SRV:
echo 192.168.1.10/24 > /etc/net/ifaces/ens18/ipv4address
на CLI:
echo 192.168.1.20/24 > /etc/net/ifaces/ens18/ipv4address

# На ISP настраиваем только IP-адрес и конфигурацию
cp -r /etc/net/ifaces/ens18 /etc/net/ifaces/ens19
echo 192.168.1.1/24 > /etc/net/ifaces/ens19/ipv4address
echo BOOTPROTO=static > /etc/net/ifaces/ens19/options
echo TYPE=eth >> /etc/net/ifaces/ens19/options

# Перезагружаем сеть на всех машинах
systemctl restart network
```
#### 1.3. Установка и настройка правила для NAT на ISP
```Bash
# Обновление списка пакетов и установка iptables
apt-get update
apt-get install iptables

# Разрешение на пересылку пакетов между сетевыми адаптерами внутри ISP
В файле /etc/net/sysctl.conf
net.ipv4.ip_forward = 1

# Сохранение изменений
sysctl -p

# Создание правила в iptables для таблицы NAT и его сохранение 
iptables -t nat -A POSTROUTING -o ens18 -j MASQUERADE
iptables-save -f /etc/sysconfig/iptables 

# Включение iptables. Перезагрузка сети и обновление списка пакетов на всех машинах
systemctl enable --now iptables
systemctl restart network
apt-get update
```

### 2. Установка и настройка компонентов Bacula и MySQL

#### 2.1. На SRV: Установка Director, File Daemon, консоли и Базы Данных MySQL
```bash
# Установка пакетов
apt-get install -y bacula15-director-common bacula15-director-mysql bacula15-common bacula15-storage bacula15-console bacula15-client bacula15 
apt-get install -y mariadb-server mariadb
```

#### 2.2. На CLI: Установка Storage Daemon
```bash
apt-get install -y bacula15-storage

# Создание директории хранения
mkdir -p /backup
chmod 750 /backup
```

#### 2.3. Проверка наличия пакетов
```bash
# После установки можно проверить наличие всех необходимых пакетов. Вывод должен быть такой
На SRV:
rpm -qa | grep mariadb
 mariadb-server-control-10.6.24-alt1.noarch
 mariadb-10.6.24-alt1.x86_64
 libmariadb3-10.6.24-alt1.x86_64
 mariadb-common-10.6.24-alt1.noarch
 mariadb-server-10.6.24-alt1.x86_64
 mariadb-client-10.6.24-alt1.x86_64
rpm -qa | grep bacula
 bacula15-director-common-15.0.2-alt2.x86_64
 bacula15-console-15.0.2-alt2.x86_64
 bacula15-common-15.0.2-alt2.x86_64
 bacula15-director-mysql-15.0.2-alt2.x86_64
 bacula15-storage-15.0.2-alt2.x86_64
 bacula15-15.0.2-alt2.x86_64
 bacula15-client-15.0.2-alt2.x86_64
На CLI:
rpm -qa | grep bacula
 bacula15-common-15.0.2-alt2.x86_64
 bacula15-storage-15.0.2-alt2.x86_64
```
---

### 3. Настройка защиты от вирусов-шифровальщиков

**Принципы защиты в архитектуре Bacula:**
1. **Физическая изоляция** — компонент хранения (Storage Daemon) вынесен на отдельный узел cli
2. **Политики хранения** — настройка параметров `Recycle = no` и `AutoPrune = no` для критичных заданий
3. **Только добавление (Append-only)** — на уровне файловой системы:
   ```bash
   # На cli
   chattr +a /backup  # разрешить только добавление данных
   ```

---

### 4. Создание и настройка Базы данных MySQL
Для создания базы данных необходимо отредактировать и запустить 3 скрипта. Дополнительно поменяем пароль для пользователя root в MySQL (MySQL и MariaDB - это одно и тоже).

#### 4.1. Запуск БД и смена пароля

```Bash

# Запуск и проверка работоспособности MySQL
systemctl enable --now mariadb
systemctl status mariadb

# Вход в БД и редактирование пароля пользователя
mysql -u root
ALTER USER 'root'@'localhost' IDENTIFIED BY 'root';
FLUSH PRIVILEGES;
EXIT;

# Вход по паролю root - root
mysql -u root -p
```
#### 4.2. Редактирование скриптов и создание базы данных

В каталоге /usr/share/bacula/scripts
```Bash
# Редактировать файлы скриптов лучше всего в текстовом редакторе mcedit - в нём показано, на какой строке находится курсор. В самих файлах указываем, за какого пользователя должен активироваться скрипт.
mcedit create_mysql_database
12 строка: if $bindir/mysql -u root -p $* -f <<END-OF-DATA

mcedit make_mysql_tables
26 строка: if mysql -u root -p $* -f <<END-OF-DATA

mcedit grant_mysql_privileges
11 строка: db_password="root"
20 строка: if $bindir/mysql $* -u root -p -f 2>/dev/null 1>/dev/null  <<EOD
30 строка: if $bindir/mysql $* -u root -p -f <<END-OF-DATA

# Создание базы данных с помощью скриптов (запускать из директории /usr/share/bacula/scripts)
./create_mysql_database
./make_mysql_tables
./grant_mysql_privileges
```


### 5. Конфигурация Bacula Director (SRV)
```Bash
# Заходим в директорию с файлами конфигурации и редактируем файлы:
cd /etc/bacula
```

#### 5.1. Основной конфигурационный файл `/etc/bacula/bacula-dir.conf`

```conf
# Director
Director {
  Name = "srv-dir"
  DIRport = 9101
  QueryFile = "/etc/bacula/query.sql"
  WorkingDirectory = "/var/lib/bacula"
  PidDirectory = "/var/run/bacula"
  Maximum Concurrent Jobs = 20
  Password = "root"
  TLS Enable = no 
  TLS Require = no
}

# Каталог MySQL
Catalog {
  Name = MyCatalog
  dbname = "bacula"
  dbuser = "root"
  dbpassword = "root"
  dbaddress = "localhost"  
  dbport = "3306"
}

# Messages
Messages {
  Name = Standard
  director = srv-dir = all
}

# Pool для /etc
Pool {
  Name = EtcPool
  Pool Type = Backup
  Recycle = no
  AutoPrune = no
  Volume Retention = 30 days
  Maximum Volume Bytes = 500M
  Maximum Volumes = 100
  Label Format = "etc-backup-"
}

# Pool для MySQL
Pool {
  Name = MysqlPool
  Pool Type = Backup
  Recycle = no
  AutoPrune = no
  Volume Retention = 60 days
  Maximum Volume Bytes = 1G
  Maximum Volumes = 50
  Label Format = "mysql-backup-"
}

# Storage — cli
Storage {
  Name = cli-sd
  Address = 192.168.1.20
  SDPort = 9103
  Password = "root"
  Device = cli-sd
  Media Type = File
  TLS Enable = no
}

# Client 
Client {
  Name = srv-fd
  Address = 192.168.1.10
  FDPort = 9102
  Password = "root"
  Catalog = MyCatalog
  TLS Enable = no
  TLS PSK Enable = no
  TLS Require = no
  TLS Authenticate = no
}

# FileSet для /etc
FileSet {
  Name = "EtcFileSet"
  Include {
    Options {
      signature = MD5
      compression = GZIP
    }
    File = /etc
  }
}

# FileSet для MySQL
FileSet {
  Name = "MysqlFileSet"
  Include {
    Options {
      signature = MD5
    }
    File = /var/backups/mysql
  }
}

# Schedule
Schedule {
  Name = "DailyCycle"
  Run = Full daily at 02:00
}

# JobDefs — базовые параметры (исправлен Bootstrap)
JobDefs {
  Name = "DefaultJob"
  Type = Backup
  Level = Full
  Client = srv-fd
  Schedule = "DailyCycle"
  Storage = cli-sd
  Messages = Standard
  Priority = 10
  Write Bootstrap = "/var/lib/bacula/%n.bsr"  # 
}

# Job для /etc
Job {
  Name = "Backup-Etc"
  JobDefs = "DefaultJob"
  Type = Backup
  Level = Full
  Client = srv-fd
  FileSet = "EtcFileSet"
  Schedule = "DailyCycle"
  Storage = cli-sd
  Pool = EtcPool
  Messages = Standard
  Write Bootstrap = "/var/lib/bacula/etc.bsr"
}

# Job для MySQL
Job {
  Name = "Backup-MySQL"
  JobDefs = "DefaultJob"
  Type = Backup
  Level = Full
  Client = srv-fd
  FileSet = "MysqlFileSet"
  Schedule = "DailyCycle"
  Storage = cli-sd
  Pool = MysqlPool
  Messages = Standard
  Write Bootstrap = "/var/lib/bacula/mysql.bsr"
  RunBeforeJob = "/usr/local/bin/backup-mysql.sh"
}

# Restore Job (исправлено — убрана циклическая ссылка)
Job {
  Name = "RestoreFiles"
  Type = Restore
  Client = srv-fd
  FileSet = "RestoreFileSet"
  Storage = cli-sd
  Pool = MysqlPool
  Messages = Standard
  Where = /tmp/bacula-restores
  RunBeforeJob = "/bin/rm -rf /tmp/bacula-restores/*"
  Write Bootstrap = "/var/lib/bacula/restore.bsr"
}

# FileSet для восстановления
FileSet {
  Name = "RestoreFileSet"
  Include {
    Options {
      signature = MD5
    }
    File = /
  }
}
```

#### 5.2. Конфигурационный файл File Daemon `/etc/bacula/bacula-fd.conf`
```conf
#
# Default  Bacula File Daemon Configuration file
#
#  For Bacula release 2.4.4 (28 December 2008) -- redhat 
#
# There is not much to change here except perhaps the
# File daemon Name to
#

#
# List Directors who are permitted to contact this File daemon
#
Director {
  Name = srv-dir
  Password = "root"
  TLS Enable = no
  TLS Require = no
  TLS PSK Enable = no
}

#
# Restricted Director, used by tray-monitor to get the
#   status of the file daemon
#
#Director {
#  Name = dir-mon
#  Password = ""
#  Monitor = yes
#}

#
# "Global" File daemon configuration specifications
#

FileDaemon {                          # this is me
  Name = srv-fd
  FDport = 9102                  # where we listen for the director
  WorkingDirectory = /var/lib/bacula
  Pid Directory = /var/run/bacula
  Maximum Concurrent Jobs = 20
  TLS Enable = no
  TLS Require = no
}

# Send all messages except skipped files back to Director
Messages {
  Name = Standard
  director = dir = all, !skipped, !restored
}
```
#### 5.3. Конфигурационный файл Storage Daemon `/etc/bacula/bacula-sd.conf`
```conf
#
# Default Bacula Storage Daemon Configuration file
#
#  For Bacula release 2.4.4 (28 December 2008) -- redhat 
#
# You may need to change the name of your tape drive
#   on the "Archive Device" directive in the Device
#   resource.  If you change the Name and/or the 
#   "Media Type" in the Device resource, please ensure
#   that dird.conf has corresponding changes.
#

Storage {                             # definition of myself
  Name = cli-sd
  SDPort = 9103                  # Director's port
  WorkingDirectory = "/var/lib/bacula"
  Pid Directory = "/var/run/bacula"
  Maximum Concurrent Jobs = 20
}

#
# List Directors who are permitted to contact Storage daemon
#
Director {
  Name = dir
@/etc/bacula/bacula-sd-password.conf
}

#
# Restricted Director, used by tray-monitor to get the
#   status of the storage daemon
#
#Director {
#  Name = dir-mon
#  Password = ""
#  Monitor = yes
#}


@|"sh -c 'for f in /etc/bacula/device.d/*.conf ; do echo @${f} ; done'"

#
# Send all messages to the Director, 
# mount messages also are sent to the email address
#
Messages {
  Name = Standard
  director = dir = all
}
```
#### 5.4. Файлы, содержащие пароли для каждого Демона
```Bash
mcedit bacula-dir-password.conf 
Password = "root"

mcedit bacula-fd-password.conf 
Password = "root"

mcedit bacula-sd-password.conf 
Password = "root"
```

---

### 6. Конфигурация Storage Daemon (cli)

#### 6.1. Файл `/etc/bacula/bacula-sd.conf`
```conf
Storage {
  Name = cli-sd
  SDPort = 9103
  WorkingDirectory = "/var/lib/bacula"
  PidDirectory = "/var/run/bacula"
  Maximum Concurrent Jobs = 20
  SDAddress = 192.168.1.20
}

Director {
  Name = srv-dir
  Password = "storage-password"  # тот же, что в Storage секции на srv
}

Device {
  Name = cli-Device
  Media Type = File
  Archive Device = /backup
  LabelMedia = yes
  Random Access = yes
  AutomaticMount = yes
  RemovableMedia = no
  AlwaysOpen = no
}

Messages {
  Name = Standard
  director = srv-dir = all
}
```

#### 5.2. Применение прав доступа
```bash
chown -R bacula:bacula /backup
systemctl restart bacula-storage
```

---

### 6. Скрипт резервного копирования MySQL

#### 6.1. Создание скрипта `/usr/local/bin/backup-mysql.sh`
```bash
#!/bin/bash
BACKUP_DIR="/var/backups/mysql"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
DB_NAME="webdb"
DB_USER="root"
DB_PASS="P@ssw0rd"  # Заменить на реальный пароль

mkdir -p $BACKUP_DIR

# Создание дампа с сжатием
mysqldump -u$DB_USER -p$DB_PASS --single-transaction $DB_NAME | \
  gzip > $BACKUP_DIR/webdb_${TIMESTAMP}.sql.gz

# Очистка старых бэкапов (оставляем 7 дней)
find $BACKUP_DIR -name "webdb_*.sql.gz" -mtime +7 -delete

# Проверка целостности последнего бэкапа
if [ $? -eq 0 ]; then
  echo "MySQL backup completed successfully at $TIMESTAMP" >> /var/log/mysql-backup.log
  exit 0
else
  echo "MySQL backup FAILED at $TIMESTAMP" >> /var/log/mysql-backup.log
  exit 1
fi
```

#### 6.2. Настройка прав
```bash
chmod +x /usr/local/bin/backup-mysql.sh
chown root:bacula /usr/local/bin/backup-mysql.sh
chmod 750 /usr/local/bin/backup-mysql.sh
```

---

### 7. Запуск и проверка работы системы

#### 7.1. Запуск сервисов на srv
```bash
systemctl start bacula-director
systemctl start bacula-fd
systemctl enable bacula-director bacula-fd
```

#### 7.2. Запуск сервиса на cli
```bash
systemctl start bacula-sd
systemctl enable bacula-sd
```

#### 7.3. Инициализация среды Bacula
```bash
# На srv
bconsole
> label
# Выбрать устройство "cli-Device", ввести метку для первого тома (например, etc-backup-0001)
> label
# Повторить для пула MySQL (mysql-backup-0001)
> quit
```

#### 7.4. Ручной запуск заданий
```bash
bconsole
> run job=Backup-Etc
> run job=Backup-MySQL
> status director
> list jobs
> quit
```

#### 7.5. Проверка результатов на cli
```bash
ls -lh /backup/
# Должны появиться файлы вида:
# etc-backup-0001
# mysql-backup-0001
```

---

### 8. Восстановление данных (демонстрация защиты)

#### 8.1. Восстановление директории /etc
```bash
bconsole
> restore
# Выбрать: 5 (запросить список последних заданий)
# Выбрать JobId для Backup-Etc
# Выбрать: 2 (выбрать файлы для восстановления)
# Перейти в /etc, выбрать нужные файлы
# Выбрать: 1 (восстановить)
# Указать точку восстановления (например, /tmp/restored-etc)
> quit

# Проверка
ls -la /tmp/restored-etc/
```

#### 8.2. Восстановление базы данных MySQL
```bash
# Распаковка дампа
gunzip -c /tmp/restored-etc/webdb_*.sql.gz | mysql -u root -p webdb
```

---

### 9. Контрольные вопросы для студентов

1. Почему компонент Storage Daemon вынесен на отдельный узел? Как это повышает защиту от шифровальщиков?
2. Какие параметры конфигурации (`Recycle`, `AutoPrune`) обеспечивают защиту от перезаписи резервных копий?
3. Почему для резервного копирования MySQL используется предварительный скрипт вместо прямого копирования файлов БД?
4. Как атрибут файловой системы `+a` (append-only) дополняет защиту резервных копий?
5. Какие сетевые порты используются компонентами Bacula и почему важно их ограничить фаерволом?

---

### 10. Типичные ошибки и их устранение

| Проблема | Причина | Решение |
|----------|---------|---------|
| Ошибка подключения к Storage | Несовпадение паролей в конфигах | Сверить `Password` в секциях `Storage` (Director) и `Director` (Storage Daemon) |
| Задание зависает в состоянии "Waiting" | Не промаркирован том | Выполнить `label` в bconsole для нужного пула |
| Ошибка доступа к /backup | Неправильные права | `chown bacula:bacula /backup && chmod 750 /backup` |
| Сбой дампа MySQL | Неверный пароль БД в скрипте | Проверить `/root/.my.cnf` или параметры скрипта |

---

### Заключение

Данная методическая разработка демонстрирует построение отказоустойчивой системы резервного копирования на базе открытого ПО Bacula с реализацией ключевых принципов защиты от программ-вымогателей:
- **Сегрегация компонентов** (хранение отделено от источника данных)
- **Защита от перезаписи** (политики хранения + атрибуты ФС)
- **Регулярная верификация** (встроенные механизмы проверки целостности)
- **Минимизация привилегий** (процессы работают без прав суперпользователя)
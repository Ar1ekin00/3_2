# Методические указания: Настройка резервного копирования с использованием Bacula в среде ALT Linux

**Цель работы:** Освоить развертывание распределенной системы резервного копирования Bacula с защитой от вирусов-шифровальщиков в изолированной лабораторной среде. 

---

# Теоретическая часть 
## Backup
**Бэкап (backup)** — это процесс создания резервной копии данных для их защиты от потери, повреждения или удаления, с возможностью последующего восстановления. Он сохраняет информацию (файлы, БД, ОС) на отдельном носителе — диске, ленте, облаке — и используется при сбоях оборудования, ошибках, атаках или миграции.

**Полная резервная копия (Full Backup) —** копия всех файлов и каталогов, которые указаны в конфигурации задания. Обеспечивает полную и актуальную копию данных, но требует больше времени и места по сравнению с другими типами.

**Инкрементальная резервная копия (Incremental Backup) —** хранит только файлы и изменения с момента последнего бэкапа (полного или инкрементального). Этот подход экономит время и пространство на диске, так как копируются только изменения. Однако для восстановления данных нужен полный бэкап и все последующие инкрементальные.

**Дифференциальная резервная копия (Differential Backup) —** сохраняет изменения с момента последнего полного бэкапа. С каждой новой дифференциальной копией объём данных будет расти, однако для восстановления нужно только два последних бэкапа: полный и дифференциальный.

## Bacula 
**Bacula -** кроссплатформенное клиент-серверное программное обеспечение для резервного копирования, восстановления и проверки данных по сети. Позволяет создавать резервные копии файлов и директорий, СУБД, данных почтовых серверов, системных образов и операционных систем. 

### Основные компоненты:
Bacula имеет модульную клиент-серверную архитектуру с центральным управлением:

**Director (bacula-dir):** Центральный демон, координирует все операции, общается с другими компонентами и ведет каталог в SQL-БД (MySQL, PostgreSQL).


**Storage Daemon (bacula-sd):** Управляет хранилищем данных, записывает/читает бэкапы на дисках или лентах, часто на отдельном сервере.

​
**File Daemon (bacula-fd):** Клиентский агент на каждом хосте, создает потоки данных для бэкапа и восстановления.

​
**Консоль (bconsole, BAT или web-интерфейс):** Для мониторинга, управления заданиями и отчетов.

**Catalog:** База данных, в которой хранятся сведения обо всех зарезервированных файлах и их местонахождении в резервных копиях. Поддерживаются MySQL, PostgreSQL и SQLite. 

Пароли для каждого модуля хранятся в отдельных файлах, чтобы обеспечить безопасность и гибкость: это минимизирует риски утечки (например, не все админы получают доступ к dir.conf). 

**ВАЖНО:** Пароли в секциях Storage/Client (bacula-dir.conf) должны точно совпадать с паролями Director в bacula-sd.conf и bacula-fd.conf для аутентификации: Director инициирует соединения, а демоны проверяют пароль перед обменом данными. Даже при сетевом разделении (Storage на удаленном узле) это обязательно — Bacula использует простой парольный механизм без шифрования трафика по умолчанию

![](https://github.com/Ar1ekin00/Sources/blob/main/MD-3813-2.png)



---
# Практическая часть

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
# Создаём файлы маршрута по умолчанию, DNS-сервера и настройки конфигурации сетевого адаптера соответственно  
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
apt-get install -y bacula15-director-common bacula15-director-mysql bacula15-common bacula15-console bacula15-client 
apt-get install -y mariadb-server mariadb
```

#### 2.2. На CLI: Установка Storage Daemon
```bash
apt-get install -y bacula15-storage

# Создание директории хранения бэкапов
mkdir -p /backup
chmod 750 /backup
chown bacula:bacula /backup
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
 bacula15-director-mysql-15.0.2-alt2.x86_64
 bacula15-director-common-15.0.2-alt2.x86_64
 bacula15-console-15.0.2-alt2.x86_64
 bacula15-common-15.0.2-alt2.x86_64
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
2. **Разграничение прав** — процессы Bacula работают от имени пользователя `bacula` без прав на запись в защищаемые директории
3. **Политики хранения** — настройка параметров `Recycle = no` и `AutoPrune = no` для критичных заданий
4. **Только добавление (Append-only)** — на уровне файловой системы:
   ```bash
   # На cli
   chattr +a /backup  # разрешить только добавление данных
   ```
---

### 4. Создание и настройка Базы данных MySQL

#### 4.1. Запуск БД и создание базы данных Bacula
```Bash

# Запуск и проверка работоспособности MySQL
systemctl enable --now mariadb
systemctl status mariadb

# Создание базы данных с помощью скриптов (запускать из директории /usr/share/bacula/scripts)
cd /usr/share/bacula/scripts
./create_mysql_database
./make_mysql_tables
./grant_mysql_privileges
```

#### 4.2. Cмена пароля для пользователя bacula и root
```Bash
mysql -u root
ALTER USER 'bacula'@'localhost' IDENTIFIED BY 'bacula';
ALTER USER 'root'@'localhost' IDENTIFIED BY 'P@ssw0rd';
FLUSH PRIVILEGES;
EXIT;
```

---
### 5. Конфигурация Bacula Director (SRV)
```Bash
# Заходим в директорию с файлами конфигурации всех сервисов Bacula
cd /etc/bacula
```

#### 5.1. Редактируем основной конфигурационный файл `/etc/bacula/bacula-dir.conf`

```conf
# Director
Director {
  Name = "srv-dir"
  DIRport = 9101
  QueryFile = "/etc/bacula/query.sql"
  WorkingDirectory = "/var/lib/bacula"
  PidDirectory = "/var/run/bacula"
  Maximum Concurrent Jobs = 20
  Password = "director-password"
}

# Каталог MySQL
Catalog {
  Name = MyCatalog
  dbname = "bacula"
  dbuser = "bacula"
  dbpassword = "bacula"
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
  Password = "storage-password"
  Device = cli-sd
  Media Type = File
}

# Client 
Client {
  Name = srv-fd
  Address = 192.168.1.10
  FDPort = 9102
  Password = "client-password"
  Catalog = MyCatalog
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

# JobDefs — базовые параметры
JobDefs {
  Name = "DefaultJob"
  Type = Backup
  Level = Full
  Client = srv-fd
  Schedule = "DailyCycle"
  Storage = cli-sd
  Messages = Standard
  Priority = 10
  Write Bootstrap = "/var/lib/bacula/%n.bsr"  
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

# Restore Job
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

#### 5.2. Редактируем конфигурационный файл File Daemon `/etc/bacula/bacula-fd.conf`
```conf
Director {
  Name = srv-dir
@/etc/bacula/bacula-fd-password.conf
}

FileDaemon {                          # this is me
  Name = srv-fd
  FDport = 9102                  # where we listen for the director
  WorkingDirectory = /var/lib/bacula
  Pid Directory = /var/run/bacula
  Maximum Concurrent Jobs = 20
}

Messages {
  Name = Standard
  director = dir = all, !skipped, !restored
}
```

#### 5.3. Генерация паролей и безопасная настройка
```bash
# Генерация надежных паролей и замена их в главном конфигурационном bacula
DIR_PASSWORD=$(openssl rand -base64 24)
sed -i "s|director-password|$DIR_PASSWORD|" /etc/bacula/bacula-dir.conf

SD_PASSWORD=$(openssl rand -base64 24)
sed -i "s|storage-password|$SD_PASSWORD|" /etc/bacula/bacula-dir.conf

FD_PASSWORD=$(openssl rand -base64 24)
sed -i "s|client-password|$FD_PASSWORD|" /etc/bacula/bacula-dir.conf
# Замена паролей в отдельных файлах
sed -n '9p' bacula-dir.conf > bacula-dir-password.conf
sed -n '67p' bacula-dir.conf > bacula-fd-password.conf
```

---
### 6. Создание скрипта /usr/local/bin/backup-mysql.sh
На SRV:
```Bash
# Создание каталога, в котором будут храниться дампы. Дампы будут создаваться при запуске скрипта
mkdir -p /var/backups/mysql
# Выдача прав на запись в каталог для bacula
chown bacula:bacula /var/backups/mysql
```
```
# Создать по следующему пути /usr/local/bin
cd /usr/local/bin
# Файл скрипта backup-mysql.sh 
nano backup-mysql.sh 
```
Содержимое файла скрипта /usr/local/bin/backup-mysql.sh
```bash
#!/bin/bash
BACKUP_DIR="/var/backups/mysql"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
DB_NAME="webdb"
DB_USER="root"
DB_PASS="P@ssw0rd"

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
```
# Создание базы данных webdb
mysql -u root -p -e "CREATE DATABASE IF NOT EXISTS webdb CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
```
```Bash
# Выдача прав на исполнение скрипта 
chown root:bacula /usr/local/bin/backup-mysql.sh
chmod 750 /usr/local/bin/backup-mysql.sh
```
---

### 7. Конфигурация Storage Daemon (CLI)
```Bash
# Заходим в директорию с файлами конфигурации и редактируем:
cd /etc/bacula
```
#### 7.1. Файл `/etc/bacula/bacula-sd.conf`
**Поменять строку "storage-password" на пароль, сгенерированный на SRV. Найдёте его на SRV в файле bacula-dir.conf, в сегменте Storage**
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
  Password = "storage-password"
}

Device {
  Name = cli-sd
  Media Type = File
  Archive Device = /backup
  LabelMedia = yes
  Random Access = yes
  AutomaticMount = yes
  RemovableMedia = no
  AlwaysOpen = yes
}

Messages {
  Name = Standard
  director = srv-dir = all
}
```
#### 7.2. В файле с паролем указать тот же сгенерированный пароль, что и в bacula-sd.conf
```Bash
nano bacula-sd-password.conf 
Password = "storage-password"
```
---

### 8. Запуск и проверка работы системы

#### 8.1. Запуск сервисов на SRV
```bash
systemctl enable --now bacula-dir.service
systemctl enable --now bacula-fd.service
```

#### 8.2. Запуск сервиса на CLI
```bash
systemctl enable --now bacula-sd.service
```

#### 8.3. Инициализация среды Bacula
```bash
# На SRV с помощью bconsole создаём метки на диске, где будут храниться бэкапы
bconsole
# Для etc-backup
> label
> etc-backup-0001 (Имя первого тома)
> 1 (EtcPool)
# Для mysql-backup
> label
> mysql-backup-0001 (Имя первого тома)
> 2 (выбираем MysqlPool)
```

#### 8.4. Ручной запуск заданий
```bash
bconsole
> run job=Backup-Etc
> run job=Backup-MySQL
> status director
> list jobs
```

#### 8.5. Проверка результатов на cli
```bash
ls -lh /backup/
# Должны появиться файлы вида:
# etc-backup-0001
# mysql-backup-0001
```

---

### 9. Восстановление данных (демонстрация защиты)

#### 9.1 Восстановление директории /etc
```bash
bconsole
> restore (запускаем восстановление бэкапа)
> 5 (запросить список последних заданий)
> 1 (какой бэкап хотим восстановить)
# Дальше можем передвигаться, как в командной строке и смотреть содержимое файлов
ls
cd etc/
cd ..
# Если хотим восстановить весь бэкап, то из корневого каталога вводим
mark * (маркируем, какие файлы и каталоги хотим восстановить "*" - звёздочка выделяет все файлы и каталоги)
done (после маркировки нужных файлов, соглашаемся на восстановление)
yes (подтверждаем настройки задачи)
# Проверка
ls -la /tmp/bacula-restores/
```
### 10. Восстановление базы данных MySQL
```bash
# Распаковка дампа
gunzip -c /tmp/bacula-restores/var/backups/mysql/webdb_*.sql.gz | mysql -u root -p webdb
# Проверка наличия данных в таблице
mysqlshow -u root -p webdb
```

---

### 11. Контрольные вопросы для студентов

1. Почему компонент Storage Daemon вынесен на отдельный узел? Как это повышает защиту от шифровальщиков?
2. Какие параметры конфигурации (`Recycle`, `AutoPrune`) обеспечивают защиту от перезаписи резервных копий?
3. Почему для резервного копирования MySQL используется предварительный скрипт вместо прямого копирования файлов БД?
4. Как атрибут файловой системы `+a` (append-only) дополняет защиту резервных копий?
5. Какие сетевые порты используются компонентами Bacula и почему важно их ограничить фаерволом?

---

### 12. Типичные ошибки и их устранение

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








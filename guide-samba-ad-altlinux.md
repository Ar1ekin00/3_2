# Гайд: Samba AD + Веб-приложение + HTTPS на ALT Linux
> Проверено на практике. Все команды рабочие.

---

## Содержание
1. [Настройка DNS (Samba AD DC)](#1-настройка-dns-samba-ad-dc)
2. [Групповые политики (ADMC)](#2-групповые-политики-admc)
3. [Развёртывание веб-приложения на srv (Apache + MariaDB)](#3-развёртывание-веб-приложения-на-srv-apache--mariadb)
4. [Центр сертификации и HTTPS](#4-центр-сертификации-и-https)

---

## 1. Настройка DNS (Samba AD DC)

### Проблема: winbind не может найти DC, `gpupdate` падает

После установки Samba AD DC провиженинг **не создаёт** автоматически reverse DNS зону. Из-за этого `samba_dnsupdate` падает с exit code 2 при попытке зарегистрировать PTR-запись.

### Диагностика

```bash
# Проверяем список зон — должно быть 3, но обычно только 2
samba-tool dns zonelist 172.16.0.10 -U Administrator

# Запускаем обновление DNS вручную и смотрим что падает
samba_dnsupdate --verbose --all-names 2>&1 | grep -E "Calling|success|Failed|error"

# Проверяем A-запись DC
samba-tool dns query 172.16.0.10 lab.local dc.lab.local A -U Administrator

# Проверяем PTR — скорее всего упадёт с ошибкой
samba-tool dns query 172.16.0.10 0.16.172.in-addr.arpa 10 PTR -U Administrator
```

### Исправление: создаём reverse зону и PTR

```bash
# Создаём reverse зону (AD-integrated, в DomainDnsZones)
samba-tool dns zonecreate 172.16.0.10 0.16.172.in-addr.arpa \
  --partition=DomainDnsZones -U Administrator

# Добавляем PTR запись (обязательно с точкой в конце FQDN!)
samba-tool dns add 172.16.0.10 0.16.172.in-addr.arpa 10 PTR dc.lab.local. -U Administrator

# Проверяем
samba-tool dns query 172.16.0.10 0.16.172.in-addr.arpa 10 PTR -U Administrator
```

### Сбрасываем кэш и перезапускаем

```bash
systemctl stop samba-ad-dc
rm -f /var/lib/samba/private/dns_update_cache
systemctl start samba-ad-dc
sleep 5

# Проверяем — ошибок быть не должно
samba_dnsupdate --verbose --all-names 2>&1 | grep -E "Failed|error"

# Проверяем winbind
wbinfo --ping-dc
wbinfo --get-domain-info
```

---

## 2. Групповые политики (ADMC)

### Установка GPO через ADMC

1. Пуск → поиск "ADMC"
2. Войти за `Administrator`
3. В левой панели: `Group Policy Objects` → `lab.local`
4. ПКМ по OU `admins` → `Create a GPO and link to this OU` → задать имя
5. ПКМ по OU `managers` → `Link existing GPO` → выбрать созданную политику
6. Кликнуть по политике → включить галочку `Enforced` для обеих групп
7. ПКМ по политике → `Edit`
8. Путь: `User → Administrative Templates → ALT System → Mate settings → Screensaver`

Включить следующие политики (галочка `Enabled`, затем `OK`):
- `Enable screen saver` — включает заставку при бездействии
- `Lock your computer` — блокировка компьютера
- `Logout after being locked` — выход после блокировки
- `Operating mode` → в Options выбрать `Random themes`
- `Switch user after blocking` → поставить `Disabled`

### Проблема: `gpupdate` падает с Access Denied

```
Unable to refresh GPO list — Access Denied (CLI$)
```

**Причина 1: неправильный `/etc/krb5.conf` на клиенте**

Проверяем:
```bash
cat /etc/krb5.conf
```

Должно быть:
```ini
[libdefaults]
    default_realm = LAB.LOCAL
    dns_lookup_kdc = true
    dns_lookup_realm = false
    ticket_lifetime = 24h
    renew_lifetime = 7d
    forwardable = true
    rdns = false
    default_ccache_name = KEYRING:persistent:%{uid}

[realms]
    LAB.LOCAL = {
        kdc = dc.lab.local
        admin_server = dc.lab.local
        default_domain = lab.local
    }

[domain_realm]
    .lab.local = LAB.LOCAL
    lab.local = LAB.LOCAL
```

Получаем машинный тикет (обязательно в одинарных кавычках — иначе bash съест `$`):
```bash
kinit -k 'CLI$@LAB.LOCAL'
klist  # должен показать тикет
```

**Причина 2: сломанные ACL на SYSVOL**

```bash
# На DC — проверяем и сбрасываем права
samba-tool ntacl sysvolcheck
samba-tool ntacl sysvolreset
```

После этого на клиенте:
```bash
gpupdate
```

---

## 3. Развёртывание веб-приложения на srv (Apache + MariaDB)

### Установка и настройка

```bash
apt-get install -y lamp-server
systemctl enable --now httpd2
systemctl enable --now mysqld

# Монтируем ISO и копируем файлы
mkdir -p /mnt/iso /opt/testapp
mount /dev/sr0 /mnt/iso
cp -r /mnt/iso /opt/testapp/

# Копируем файлы приложения
mkdir -p /var/www/html/testapp
cp /opt/testapp/iso/web/index.php /var/www/html/testapp/
cp /opt/testapp/iso/web/logo.png /var/www/html/testapp/

# Права
chown -R apache2:apache2 /var/www/html/testapp
chmod 755 /var/www /var/www/html /var/www/html/testapp
chmod 644 /var/www/html/testapp/index.php
```

### Настройка MariaDB

```bash
mysql -u root
```

```sql
CREATE DATABASE webdb CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'web'@'localhost' IDENTIFIED BY 'P@ssw0rd';
GRANT ALL PRIVILEGES ON webdb.* TO 'web'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

### Импорт базы данных

```bash
# Импортируем дамп (важно: от root, не от web)
mariadb -u root webdb < /opt/testapp/iso/web/dump.sql

# Проверяем что таблицы появились
mariadb -u root -e "USE webdb; SHOW TABLES;"
```

> ⚠️ Если таблицы не появились — `gpupdate` и сайт будут давать 500 ошибку. Всегда проверяй импорт!

### Настройка подключения к БД в index.php

```bash
nano /var/www/html/testapp/index.php
```

В начале файла:
```php
$servername = "localhost";
$username = "web";
$password = "P@ssw0rd";
$dbname = "webdb";
```

```bash
systemctl restart httpd2
```

Проверка с клиента: `http://web/testapp/`

---

## 4. Центр сертификации и HTTPS

### Шаг 1: Создаём CA на srv

```bash
mkdir -p /etc/ssl/CA/{certs,private,newcerts}
chmod 700 /etc/ssl/CA/private
echo 01 > /etc/ssl/CA/serial
touch /etc/ssl/CA/index.txt

# Ключ CA
openssl genrsa -out /etc/ssl/CA/private/ca.key 4096

# Корневой сертификат (365 дней)
openssl req -new -x509 -days 365 -key /etc/ssl/CA/private/ca.key \
  -out /etc/ssl/CA/certs/ca.crt \
  -subj "/C=RU/ST=Moscow/L=Moscow/O=LAB/CN=LAB-CA"
```

### Шаг 2: Выпускаем сертификат для srv.lab.local

```bash
openssl genrsa -out /etc/ssl/CA/private/srv.key 2048

openssl req -new -key /etc/ssl/CA/private/srv.key \
  -out /etc/ssl/CA/srv.csr \
  -subj "/C=RU/ST=Moscow/L=Moscow/O=LAB/CN=srv.lab.local"

cat > /etc/ssl/CA/srv_ext.cnf << 'EOF'
[v3_req]
subjectAltName = @alt_names
basicConstraints = CA:FALSE
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth

[alt_names]
DNS.1 = srv.lab.local
DNS.2 = web.lab.local
DNS.3 = docker.lab.local
EOF

openssl x509 -req -days 365 \
  -in /etc/ssl/CA/srv.csr \
  -CA /etc/ssl/CA/certs/ca.crt \
  -CAkey /etc/ssl/CA/private/ca.key \
  -CAcreateserial \
  -out /etc/ssl/CA/certs/srv.crt \
  -extfile /etc/ssl/CA/srv_ext.cnf \
  -extensions v3_req

# Проверяем
openssl x509 -noout -subject -issuer -dates -in /etc/ssl/CA/certs/srv.crt
```

### Шаг 3: Копируем сертификаты в правильное место

> На ALT Linux Apache ищет сертификаты в `/var/lib/ssl/`, а не в `/etc/ssl/`

```bash
mkdir -p /var/lib/ssl/certs /var/lib/ssl/private

cp /etc/ssl/CA/certs/srv.crt /var/lib/ssl/certs/srv.crt
cp /etc/ssl/CA/private/srv.key /var/lib/ssl/private/srv.key
cp /etc/ssl/CA/certs/ca.crt /var/lib/ssl/certs/ca.crt
chmod 600 /var/lib/ssl/private/srv.key
```

### Шаг 4: Устанавливаем SSL модуль для Apache

> На ALT Linux mod_ssl не установлен по умолчанию и называется `apache2-mod_ssl`

```bash
apt-get install -y apache2-mod_ssl

# Включаем модули
ln -s /etc/httpd2/conf/mods-available/ssl.load /etc/httpd2/conf/mods-enabled/ssl.load
ln -s /etc/httpd2/conf/mods-available/proxy.load /etc/httpd2/conf/mods-enabled/proxy.load
ln -s /etc/httpd2/conf/mods-available/proxy.conf /etc/httpd2/conf/mods-enabled/proxy.conf
ln -s /etc/httpd2/conf/mods-available/proxy_http.load /etc/httpd2/conf/mods-enabled/proxy_http.load

# Проверяем
httpd2 -M 2>&1 | grep ssl
```

### Шаг 5: Добавляем порт 443

```bash
# Проверяем текущий конфиг портов
cat /etc/httpd2/conf/ports-enabled/http.conf

# Добавляем 443 если нет (НЕ добавляй дважды!)
echo "Listen 443" >> /etc/httpd2/conf/ports-enabled/http.conf
```

### Шаг 6: Создаём конфиг виртуальных хостов

```bash
cat > /etc/httpd2/conf/sites-available/lab-ssl.conf << 'EOF'
<IfModule ssl_module>

    <VirtualHost *:80>
        ServerName web.lab.local
        Redirect permanent / https://web.lab.local/
    </VirtualHost>

    <VirtualHost *:80>
        ServerName docker.lab.local
        Redirect permanent / https://docker.lab.local/
    </VirtualHost>

    <VirtualHost *:443>
        ServerName web.lab.local
        DocumentRoot /var/www/html/testapp

        SSLEngine on
        SSLCertificateFile /var/lib/ssl/certs/srv.crt
        SSLCertificateKeyFile /var/lib/ssl/private/srv.key
        SSLCACertificateFile /var/lib/ssl/certs/ca.crt

        <Directory /var/www/html/testapp>
            AllowOverride All
            Require all granted
        </Directory>
    </VirtualHost>

    <VirtualHost *:443>
        ServerName docker.lab.local

        SSLEngine on
        SSLCertificateFile /var/lib/ssl/certs/srv.crt
        SSLCertificateKeyFile /var/lib/ssl/private/srv.key
        SSLCACertificateFile /var/lib/ssl/certs/ca.crt

        ProxyPreserveHost On
        ProxyPass / http://localhost:8080/
        ProxyPassReverse / http://localhost:8080/
    </VirtualHost>

</IfModule>
EOF

# Включаем конфиг
ln -s /etc/httpd2/conf/sites-available/lab-ssl.conf /etc/httpd2/conf/sites-enabled/lab-ssl.conf

# Проверяем и перезапускаем
httpd2 -t
systemctl restart httpd2

# Убеждаемся что оба порта слушаются
ss -tlnp | grep httpd2
```

### Шаг 7: Передаём ca.crt на cli

```bash
# На srv (порт SSH 2222, логин admin)
scp -P 2222 /var/lib/ssl/certs/ca.crt admin@cli.lab.local:/tmp/
```

### Шаг 8: Устанавливаем сертификат на cli

```bash
# Системное хранилище ALT Linux
cp /tmp/ca.crt /etc/pki/ca-trust/source/anchors/lab-ca.crt
update-ca-trust

# Устанавливаем nss-utils если нет
apt-get install -y nss-utils

# Находим профиль Firefox
find /home /root -path "*firefox*" -name "cert9.db" 2>/dev/null
# Берём путь к папке из вывода, например:
PROFILE=/home/user/.mozilla/firefox/6rqxzgkx.default-default

# Устанавливаем в Firefox (одинарные кавычки обязательны!)
certutil -A -n "LAB-CA" -t "CT,," -i /tmp/ca.crt -d sql:${PROFILE}

# Проверяем
certutil -L -d sql:${PROFILE} | grep LAB-CA
```

### Шаг 9: Выпускаем сертификат для dc.lab.local

```bash
# Копируем CA ключи с srv на dc
scp -P 2222 /etc/ssl/CA/certs/ca.crt admin@dc.lab.local:/tmp/
scp -P 2222 /etc/ssl/CA/private/ca.key admin@dc.lab.local:/tmp/

# На dc:
mkdir -p /etc/ssl/CA/{certs,private}
cp /tmp/ca.crt /etc/ssl/CA/certs/ca.crt
cp /tmp/ca.key /etc/ssl/CA/private/ca.key
chmod 600 /etc/ssl/CA/private/ca.key

openssl genrsa -out /etc/ssl/CA/private/dc.key 2048

openssl req -new -key /etc/ssl/CA/private/dc.key \
  -out /etc/ssl/CA/dc.csr \
  -subj "/C=RU/ST=Moscow/L=Moscow/O=LAB/CN=dc.lab.local"

cat > /etc/ssl/CA/dc_ext.cnf << 'EOF'
[v3_req]
subjectAltName = @alt_names
basicConstraints = CA:FALSE
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth

[alt_names]
DNS.1 = dc.lab.local
DNS.2 = moodle.lab.local
EOF

openssl x509 -req -days 365 \
  -in /etc/ssl/CA/dc.csr \
  -CA /etc/ssl/CA/certs/ca.crt \
  -CAkey /etc/ssl/CA/private/ca.key \
  -CAcreateserial \
  -out /etc/ssl/CA/certs/dc.crt \
  -extfile /etc/ssl/CA/dc_ext.cnf \
  -extensions v3_req

# Проверяем
openssl x509 -noout -subject -issuer -dates -in /etc/ssl/CA/certs/dc.crt
```

### Проверка результата

```bash
# На cli — проверяем сертификат
openssl s_client -connect web.lab.local:443 -servername web.lab.local 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates
```

В браузере открываем:
- `https://web.lab.local/testapp/` — зелёный замок ✅
- `https://docker.lab.local/` — зелёный замок ✅

---

## Частые ошибки и их решения

| Ошибка | Причина | Решение |
|--------|---------|---------|
| `SSL_ERROR_RX_RECORD_TOO_LONG` | Браузер открывает HTTPS, сервер отвечает HTTP | Вводить `http://` явно, или настроить SSL |
| `Table 'webdb.employees' doesn't exist` | Дамп не импортирован | `mariadb -u root webdb < dump.sql` |
| `kinit: Cannot determine realm` | Нет `[realms]` в krb5.conf | Добавить секцию `[realms]` и `[domain_realm]` |
| `Access Denied` для `CLI$` в gpupdate | Нет машинного тикета или сломан SYSVOL | `kinit -k 'CLI$@LAB.LOCAL'` + `samba-tool ntacl sysvolreset` на DC |
| `exit code 2` в samba_dnsupdate | Нет reverse DNS зоны | Создать зону `0.16.172.in-addr.arpa` и PTR запись |
| `Could not open ssl.load` | ssl модуль не установлен | `apt-get install -y apache2-mod_ssl` |
| Дублирование `Listen 443` | Добавили строку дважды | Удалить лишнюю через `nano` |

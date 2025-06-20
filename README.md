# Установка зависимостей
```
dnf install -y net-tools rsyslog postgresql postgresql-server php php-pgsql php-gd php-mbstring php-common httpd wget unzip php-mysqlnd postgresql-contrib
```
отключить обязательно в nano /etc/selinux/config выключить selinux
```
SELINUX=permissive
```
отключить обязательно
```
sudo systemctl stop firewalld
```



# Работа с БД PostgreSQL
```
postgresql-setup --initdb - инициализация БД
```
```
systemctl enable postgresql - добавляем в автозагрузку
```
```
systemctl start postgresql
```
В nano /var/lib/pgsql/data/postgresql.conf припишем IP для бд
```
listen_addresses = '192.168.0.178'
```
```
systemctl restart postgresql
```

# Работа с внутрянкой БД PostgreSQL:
по очередно выполнить команды:
```
sudo -u postgres psql -c "CREATE USER loguser WITH PASSWORD 'qwerty99';" - создание юера и пароля
sudo -u postgres psql -c "CREATE DATABASE logdb OWNER loguser;" - создание БД и выдача владельца к БД
sudo -u postgres psql -d logdb -c "CREATE EXTENSION hstore;" - для логов
```

в конфиг: /var/lib/pgsql/data/pg_hba.conf добавить строчки, для работы с бд
```
host    logdb           loguser         127.0.0.1/32            md5
host    all             all             192.168.0.0/24          md5
```
```
systemctl restart postgresql
```
проверка работы бд от юзера
```
psql -h 192.168.0.169 -U loguser -d logdb -W - проверка подключения к БД
```
Логинимся в БД:
```
psql -U loguser -d logdb -h 192.168.0.169
```
Когда залогинились создаем таблицы:
```
-- Создание основной таблицы systemevents
CREATE TABLE systemevents (
    id SERIAL PRIMARY KEY,
    customerid BIGINT,
    receivedat TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    devicereportedtime TIMESTAMP WITH TIME ZONE,
    facility SMALLINT,
    priority SMALLINT,
    fromhost VARCHAR(60),
    message TEXT,
    ntseverity INT,
    importance INT,
    eventsource VARCHAR(60),
    eventuser VARCHAR(60),
    eventcategory INT,
    eventid INT,
    eventbinarydata TEXT,
    maxavailable INT,
    currusage INT,
    minusage INT,
    maxusage INT,
    infounitid INT,
    syslogtag VARCHAR(60),
    eventlogtype VARCHAR(60),
    genericfilename VARCHAR(60),
    systemid INT
);

-- Создание таблицы свойств
CREATE TABLE systemeventsproperties (
    id SERIAL PRIMARY KEY,
    systemeventid INT REFERENCES systemevents(id) ON DELETE CASCADE,
    parametername VARCHAR(255),
    parametervalue TEXT
);
```

через \dt проверить список таблиц, должны быть наши таблицы


# Установка парсера логов
```
dnf install -y rsyslog-pgsql - установка парсера логов для БД
```
Создаем конфиг файл для rsyslog , заходим в nano /etc/rsyslog.d/postgresql.conf и туда вставляем
```
module(load="ompgsql")

template(name="pgsql_template" type="list" option.sql="on") {
    constant(value="INSERT INTO systemevents (fromhost, facility, priority, message, syslogtag, devicereportedtime) VALUES ('")
    property(name="hostname")
    constant(value="', ")
    property(name="syslogfacility")
    constant(value=", ")
    property(name="syslogpriority")
    constant(value=", '")
    property(name="msg")
    constant(value="', '")
    property(name="syslogtag")
    constant(value="', '")
    property(name="timereported" dateformat="rfc3339")
    constant(value="');")
}

# Правило для отправки логов в PostgreSQL
*.* action(
    type="ompgsql"
    server="192.168.0.169"
    db="logdb"
    user="loguser"
    pass="qwerty99"
    template="pgsql_template"
)
```
Выполнить команду
```
systemctl restart rsyslog
```

Для проверке бд сделать команду
```
logger "Test message for LogAnalyzer"
```
Проверить, что лог отправился можно командой
```
psql -U loguser -d logdb -h 192.168.0.169 -c "SELECT * FROM systemevents ORDER BY id DESC LIMIT 1;"
```

# Установка и настройка loganalyzer
Установим логанализер 
```
dnf install loganalyzer
```
после установки сделать ссылку
```
ln -s /usr/share/loganalyzer/ /var/www/html/loganalyzer
```

Настройка nano /etc/php.ini, требуется поставить
```
date.timezone = "Europe/Moscow"
memory_limit = 256M
```

Настройка Балансировщика httpd, сделайть конфиг в nano /etc/httpd/conf.d/loganalyzer.conf
```
<VirtualHost *:80>
    ServerName loganalizer
    DocumentRoot /var/www/html/loganalyzer
    <Directory /var/www/html/loganalyzer>
        Options -Indexes +FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>
    ErrorLog /var/log/httpd/loganalyzer_error.log
    CustomLog /var/log/httpd/loganalyzer_access.log combined
</VirtualHost>
```
В /etc/httpd/conf/httpd.conf
```
Listen 0.0.0.0:80
```
Выполнить команду
```
systemctl enable httpd
systemctl restart httpd
```

# Установка сопутствующей ДБ mysql для конфигрурации loganalizer
Установка БД
```
dnf install -y mariadb-server mariadb
```
Стартуем БД
```
systemctl enable mariadb
systemctl start mariadb
```
Настройка БД
```
mariadb-admin -u root password qwerty99 - назначаем пароль для бд
mysql -u root -p - заходим в бд для проверки
```
Для подключения БД требуется выполнить настройки nano /etc/my.cnf.d/server.cnf
```
[mysqld]
bind-address = 192.168.0.169
```
выполнить рестарт для применения настроек
```
systemctl restart mariadb
```
Требуется создать скрипт, который создаст БД и таблицы для mariaDB. nano sql_create.sql
```
CREATE DATABASE Syslog;
USE Syslog;
CREATE TABLE SystemEvents
(
ID int unsigned not null auto_increment primary key,
CustomerID bigint,
ReceivedAt datetime NULL,
DeviceReportedTime datetime NULL,
Facility smallint NULL,
Priority smallint NULL,
FromHost varchar(60) NULL,
Message text,
NTSeverity int NULL,
Importance int NULL,
EventSource varchar(60),
EventUser varchar(60) NULL,
EventCategory int NULL,
EventID int NULL,
EventBinaryData text NULL,
MaxAvailable int NULL,
CurrUsage int NULL,
MinUsage int NULL,
MaxUsage int NULL,
InfoUnitID int NULL ,
SysLogTag varchar(60),
EventLogType varchar(60),
GenericFileName VarChar(60),
SystemID int NULL
);
CREATE TABLE SystemEventsProperties
(
ID int unsigned not null auto_increment primary key,
SystemEventID int NULL ,
ParamName varchar(255) NULL ,
ParamValue text NULL
);
```
выполнить скрипт
```
mariadb -u root -p < /root/sql_create.sql
```
Выполнить настройку БД
```
подключиться в mariadb -u root -p
CREATE USER 'rsyslog'@'192.168.0.169' IDENTIFIED BY 'qwerty99';
GRANT ALL PRIVILEGES ON Syslog.* TO 'rsyslog'@'192.168.0.169';
FLUSH PRIVILEGES;
проверка подключения mysql -u rsyslog -p -h 192.168.0.169
```

Далее идем в web браузер и оттуда уже делаем настройки.
```
http://192.168.0.178/login.php
http://192.168.0.178//index.php
```




Возможно потребуется, а может и нет в /var/www/html/loganalyzer/config.php
поменять 
```
$CFG['Sources']['Source1']['ID'] = 'Source1';
$CFG['Sources']['Source1']['Name'] = 'My Syslog Source';
$CFG['Sources']['Source1']['ViewID'] = 'SYSLOG';
$CFG['Sources']['Source1']['SourceType'] = SOURCE_PDO;
$CFG['Sources']['Source1']['DBTableType'] = 'monitorware';
$CFG['Sources']['Source1']['DBType'] = 'pgsql';  ВОТ ЭТО ПОМЕНЯТЬ!!!
$CFG['Sources']['Source1']['DBServer'] = '192.168.0.178';
$CFG['Sources']['Source1']['DBName'] = 'logdb';
$CFG['Sources']['Source1']['DBUser'] = 'loguser';
$CFG['Sources']['Source1']['DBPassword'] = 'qwerty99';
$CFG['Sources']['Source1']['DBTableName'] = 'systemevents';
$CFG['Sources']['Source1']['DBEnableRowCounting'] = true;

```


Разлогинить всех rm -f /var/lib/php/session/*



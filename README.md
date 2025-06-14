# Установка зависимостей
```
dnf install -y net-tools rsyslog postgresql postgresql-server php php-pgsql php-gd php-common httpd wget unzip php-mysqlnd postgresql-contrib
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

через \dt проверить список таблиц
должны быть наши таблицы


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











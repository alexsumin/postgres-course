# Домашнее задание

## Настройка autovacuum с учетом оптимальной производительности

## Цель:

• запустить нагрузочный тест pgbench

• настроить параметры autovacuum для достижения максимального уровня устойчивой производительности

## Описание/Пошаговая инструкция выполнения домашнего задания:

• создать GCE инстанс типа e2-medium и диском 10GB

• установить на него PostgreSQL 14 с дефолтными настройками

• применить параметры настройки PostgreSQL из прикрепленного к материалам занятия файла

• выполнить pgbench -i postgres

• запустить pgbench -c8 -P 60 -T 600 -U postgres postgres

• дать отработать до конца

• дальше настроить autovacuum максимально эффективно

• построить график по получившимся значениям
так чтобы получить максимально ровное значение tps


## Выполнение домашнего задания

### Cоздать GCE инстанс типа e2-medium и диском 10GB


Была создана машина в Yandex.Cloud со следующей конфигурацией:
```
Платформа
Intel Ice Lake
Гарантированная доля vCPU       100%
vCPU                            2
RAM                             4 ГБ
Объём дискового пространства    10 ГБ
```

### Установить на него PostgreSQL 14 с дефолтными настройками

Обновление списка пакетов:

`sudo apt update`


Установка PostgreSQL:

`sudo apt install postgresql`

Запуск сервиса:

`sudo systemctl start postgresql.service`

Проверка статуса сервиса:

`sudo systemctl status postgresql.service`

Логин в бд:

`sudo -u postgres psql`

### Применить параметры настройки PostgreSQL из прикрепленного к материалам занятия файла

Содержимое файла:
```
# DB Version: 11
# OS Type: linux
# DB Type: dw
# Total Memory (RAM): 4 GB
# CPUs num: 1
# Data Storage: hdd

max_connections = 40
shared_buffers = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 512MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 500
random_page_cost = 4
effective_io_concurrency = 2
work_mem = 6553kB
min_wal_size = 4GB
max_wal_size = 16GB
```

Логин в бд:

`sudo -u postgres psql`

Применение настроек:

```
ALTER SYSTEM SET max_connections = '40'; 
ALTER SYSTEM SET shared_buffers = '1GB';
ALTER SYSTEM SET effective_cache_size = '3GB';
ALTER SYSTEM SET maintenance_work_mem = '512MB';
ALTER SYSTEM SET checkpoint_completion_target = '0.9';
ALTER SYSTEM SET wal_buffers = '16MB';
ALTER SYSTEM SET default_statistics_target = '500';
ALTER SYSTEM SET random_page_cost = '4';
ALTER SYSTEM SET effective_io_concurrency = '2';
ALTER SYSTEM SET work_mem = '6553kB';
ALTER SYSTEM SET min_wal_size = '2GB';
ALTER SYSTEM SET max_wal_size = '4GB';
```

### Выполнить pgbench -i postgres

Изменение пароля для пользователя postgres:

`ALTER USER postgres PASSWORD '123';`

Редактирование файла `pg_hba.conf`:

`sudo nano /etc/postgresql/14/main/pg_hba.conf`

В нем заменить `peer` на `md5`, результат редактирования:

```
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     md5 
# IPv4 local connections:
host    all             all             127.0.0.1/32            scram-sha-256
# IPv6 local connections:
host    all             all             ::1/128                 scram-sha-256
# Allow replication connections from localhost, by a user with the
# replication privilege.
local   replication     all                                     md5 
```
Перазапуск сервиса:

`sudo service postgresql restart`

Запуск бенчмарка:

`pgbench -i postgres -U postgres`

Вывод:

```
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.07 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.52 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 0.34 s, vacuum 0.06 s, primary keys 0.11 s).
```


### Запустить pgbench -c8 -P 60 -T 600 -U postgres postgres

Запуск бенчмарка:

`pgbench -c8 -P 60 -T 600 -U postgres postgres > 1.txt 2>&1`

Результат:

```
starting vacuum...end.
progress: 60.0 s, 502.6 tps, lat 15.897 ms stddev 11.387
progress: 120.0 s, 480.1 tps, lat 16.663 ms stddev 12.592
progress: 180.0 s, 478.5 tps, lat 16.719 ms stddev 12.885
progress: 240.0 s, 450.6 tps, lat 17.751 ms stddev 13.669
progress: 300.0 s, 412.3 tps, lat 19.402 ms stddev 14.439
progress: 360.0 s, 489.3 tps, lat 16.350 ms stddev 11.202
progress: 420.0 s, 434.0 tps, lat 18.432 ms stddev 14.151
progress: 480.0 s, 488.3 tps, lat 16.384 ms stddev 12.031
progress: 540.0 s, 517.8 tps, lat 15.448 ms stddev 10.725
progress: 600.0 s, 520.9 tps, lat 15.358 ms stddev 10.181
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 600 s
number of transactions actually processed: 286475
latency average = 16.754 ms
latency stddev = 12.369 ms
initial connection time = 64.582 ms
tps = 477.485930 (without initial connection time)
```

### Настроить autovacuum максимально эффективно

Были осуществлены несколько попыток настроить, наиболее ровное значение tps получилось с применением настроек:

```
ALTER SYSTEM SET autovacuum_vacuum_scale_factor = '0.05'; 
ALTER SYSTEM SET autovacuum_analyze_scale_factor = '0.02';
ALTER SYSTEM SET autovacuum_vacuum_cost_delay = '2';
ALTER SYSTEM SET autovacuum_max_workers = '3';
ALTER SYSTEM SET autovacuum_vacuum_threshold = '50';
```

Перазапуск сервиса:

`sudo service postgresql restart`

Запуск бенчмарка:

`pgbench -c8 -P 60 -T 600 -U postgres postgres > custom.txt 2>&1`


```
starting vacuum...end.
progress: 60.0 s, 471.8 tps, lat 16.933 ms stddev 12.455
progress: 120.0 s, 480.2 tps, lat 16.663 ms stddev 11.968
progress: 180.0 s, 504.7 tps, lat 15.850 ms stddev 11.935
progress: 240.0 s, 491.4 tps, lat 16.278 ms stddev 10.900
progress: 300.0 s, 509.4 tps, lat 15.706 ms stddev 11.632
progress: 360.0 s, 453.7 tps, lat 17.631 ms stddev 12.659
progress: 420.0 s, 495.5 tps, lat 16.145 ms stddev 11.059
progress: 480.0 s, 482.3 tps, lat 16.584 ms stddev 11.208
progress: 540.0 s, 516.4 tps, lat 15.484 ms stddev 10.318
progress: 600.0 s, 399.8 tps, lat 20.018 ms stddev 14.502
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 600 s
number of transactions actually processed: 288323
latency average = 16.646 ms
latency stddev = 11.903 ms
initial connection time = 62.528 ms
tps = 480.571860 (without initial connection time)
```
### Построение графика по получившимся значениям так чтобы получить максимально ровное значение tps

График:

<img src="https://quickchart.io/chart?c={type:%27line%27,data:{labels:[60,120,180,240,300,%20360,%20420,%20480,%20540,%20600],datasets:[{label:%27%D0%A1%D1%82%D0%B0%D0%BD%D0%B4%D0%B0%D1%80%D1%82%D0%BD%D1%8B%D0%B5%20%D0%B7%D0%BD%D0%B0%D1%87%D0%B5%D0%BD%D0%B8%D1%8F%27,data:[480.1,478.5,450.6,412.3,489.3,434.0,488.3,517.8,520.9],fill:false,borderColor:%27blue%27},{label:%27%D0%9A%D0%B0%D1%81%D1%82%D0%BE%D0%BC%D0%BD%D1%8B%D0%B5%20%D0%B7%D0%BD%D0%B0%D1%87%D0%B5%D0%BD%D0%B8%D1%8F%27,data:[471.8,480.2,504.7,509.4,453.7,495.5,482.3,516.4,399.8],fill:false,borderColor:%27green%27}]}}" width=500 />

Можно видеть небольшой прирост tps, но всё равно график нельзя назвать максимально ровным, требуется более тонкая настройка параметров autovacuum для этой конфигурации виртуальной машины.
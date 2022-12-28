# Домашнее задание

## Работа с журналами

## Цель:
• уметь работать с журналами и контрольными точками

• уметь настраивать параметры журналов

## Описание/Пошаговая инструкция выполнения домашнего задания:

1. Настройте выполнение контрольной точки раз в 30 секунд.

2. 10 минут c помощью утилиты pgbench подавайте нагрузку.

3. Измерьте, какой объем журнальных файлов был сгенерирован за это время. Оцените, какой объем приходится в среднем на одну контрольную точку.

4. Проверьте данные статистики: все ли контрольные точки выполнялись точно по расписанию. Почему так произошло?

5. Сравните tps в синхронном/асинхронном режиме утилитой pgbench. Объясните полученный результат.

6. Создайте новый кластер с включенной контрольной суммой страниц. Создайте таблицу. Вставьте несколько значений. Выключите кластер. Измените пару байт в таблице. Включите кластер и сделайте выборку из таблицы. Что и почему произошло? как проигнорировать ошибку и продолжить работу?

## Выполнение домашнего задания

### 1. Настройте выполнение контрольной точки раз в 30 секунд.

Была создана виртуальная машина в Yandex Cloud.

Подключение в postgres

`sudo -u postgres psql`

Выполнение контрольной точки раз в 30 секунд.

`ALTER SYSTEM SET checkpoint_timeout = '30s';`
`ALTER SYSTEM SET log_checkpoints = on;`

Перезапуск кластера

`sudo service postgresql restart`

Получение точки журнала

```
postgres=# SELECT pg_current_wal_insert_lsn();
 pg_current_wal_insert_lsn 
---------------------------
 0/16FBA08
(1 row)

```

### 2. 10 минут c помощью утилиты pgbench подавайте нагрузку.

Запуск бенчмарка

`pgbench -i postgres -U postgres`

`pgbench -c8 -P 60 -T 600 -U postgres postgres > hw6.txt 2>&1`

### 3. Измерьте, какой объем журнальных файлов был сгенерирован за это время. Оцените, какой объем приходится в среднем на одну контрольную точку.

Получение точки журнала

```
postgres=# SELECT pg_current_wal_insert_lsn();
 pg_current_wal_insert_lsn 
---------------------------
 0/1B6E0808
(1 row)
```

Рассчет размера между точками

`SELECT pg_size_pretty('0/1B6E0808'::pg_lsn - '0/16FBA08'::pg_lsn);`

```
 pg_size_pretty 
----------------
 416 MB
(1 row)

```

В среднем 416 / 20 = 20 МБ занимает одна контрольная точка

`sudo du -h /var/lib/postgresql/14/main/pg_wal`


### 4. Проверьте данные статистики: все ли контрольные точки выполнялись точно по расписанию. Почему так произошло?


Проверка работы точек по расписанию

`sudo cat /var/log/postgresql/postgresql-14-main.log | grep "checkpoint starting: time"`

```
2022-12-27 21:20:03.785 UTC [3737] LOG:  checkpoint starting: time
2022-12-27 21:21:33.977 UTC [3737] LOG:  checkpoint starting: time
2022-12-27 21:22:03.194 UTC [3737] LOG:  checkpoint starting: time
2022-12-27 21:22:33.057 UTC [3737] LOG:  checkpoint starting: time
2022-12-27 21:23:03.135 UTC [3737] LOG:  checkpoint starting: time
2022-12-27 21:23:33.157 UTC [3737] LOG:  checkpoint starting: time
2022-12-27 21:24:03.196 UTC [3737] LOG:  checkpoint starting: time
2022-12-27 21:24:33.098 UTC [3737] LOG:  checkpoint starting: time
2022-12-27 21:25:03.121 UTC [3737] LOG:  checkpoint starting: time
2022-12-27 21:25:33.057 UTC [3737] LOG:  checkpoint starting: time
2022-12-27 21:26:03.133 UTC [3737] LOG:  checkpoint starting: time
2022-12-27 21:26:33.171 UTC [3737] LOG:  checkpoint starting: time
2022-12-27 21:27:03.178 UTC [3737] LOG:  checkpoint starting: time
2022-12-27 21:27:33.093 UTC [3737] LOG:  checkpoint starting: time
2022-12-27 21:28:03.114 UTC [3737] LOG:  checkpoint starting: time
2022-12-27 21:28:33.141 UTC [3737] LOG:  checkpoint starting: time
2022-12-27 21:29:03.068 UTC [3737] LOG:  checkpoint starting: time
2022-12-27 21:29:33.176 UTC [3737] LOG:  checkpoint starting: time
2022-12-27 21:30:03.090 UTC [3737] LOG:  checkpoint starting: time
2022-12-27 21:30:33.060 UTC [3737] LOG:  checkpoint starting: time
2022-12-27 21:31:03.157 UTC [3737] LOG:  checkpoint starting: time
2022-12-27 21:31:33.176 UTC [3737] LOG:  checkpoint starting: time
2022-12-27 21:33:03.135 UTC [3737] LOG:  checkpoint starting: time
```

### 5. Сравните tps в синхронном/асинхронном режиме утилитой pgbench. Объясните полученный результат.

Установим синхронный режим

```
ALTER SYSTEM SET synchronous_commit TO on;
SELECT pg_reload_conf();
```

Запуск бенчмарка

`pgbench -c8 -P 60 -T 600 -U postgres postgres > hw622.txt 2>&1`

```
pgbench (14.5 (Ubuntu 14.5-0ubuntu0.22.04.1))
starting vacuum...end.
progress: 60.0 s, 467.3 tps, lat 17.099 ms stddev 12.052
progress: 120.0 s, 449.6 tps, lat 17.791 ms stddev 13.191
progress: 180.0 s, 457.2 tps, lat 17.497 ms stddev 13.238
progress: 240.0 s, 414.4 tps, lat 19.304 ms stddev 14.064
progress: 300.0 s, 423.2 tps, lat 18.899 ms stddev 14.564
progress: 360.0 s, 429.8 tps, lat 18.618 ms stddev 14.280
progress: 420.0 s, 411.3 tps, lat 19.439 ms stddev 15.451
progress: 480.0 s, 446.7 tps, lat 17.914 ms stddev 15.104
progress: 540.0 s, 434.0 tps, lat 18.435 ms stddev 14.777
progress: 600.0 s, 428.9 tps, lat 18.655 ms stddev 13.793
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 600 s
number of transactions actually processed: 261754
latency average = 18.336 ms
latency stddev = 14.076 ms
initial connection time = 60.918 ms
tps = 436.288839 (without initial connection time)
```

Установим асинхронный режим

`ALTER SYSTEM SET synchronous_commit TO off;`
`SELECT pg_reload_conf();`

Запуск бенчмарка

`pgbench -c8 -P 60 -T 600 -U postgres postgres > hw6224.txt 2>&1`

```
pgbench (14.5 (Ubuntu 14.5-0ubuntu0.22.04.1))
starting vacuum...end.
progress: 60.0 s, 3285.2 tps, lat 2.432 ms stddev 0.762
progress: 120.0 s, 3262.8 tps, lat 2.451 ms stddev 0.787
progress: 180.0 s, 3285.5 tps, lat 2.434 ms stddev 0.753
progress: 240.0 s, 3276.8 tps, lat 2.441 ms stddev 0.776
progress: 300.0 s, 3244.8 tps, lat 2.465 ms stddev 0.788
progress: 360.0 s, 3255.0 tps, lat 2.457 ms stddev 0.750
progress: 420.0 s, 3206.9 tps, lat 2.494 ms stddev 0.752
progress: 480.0 s, 3240.6 tps, lat 2.468 ms stddev 0.738
progress: 540.0 s, 3295.9 tps, lat 2.427 ms stddev 0.733
progress: 600.0 s, 3224.5 tps, lat 2.481 ms stddev 0.746
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 600 s
number of transactions actually processed: 1954680
latency average = 2.455 ms
latency stddev = 0.759 ms
initial connection time = 60.999 ms
tps = 3258.030821 (without initial connection time)
```
tps резко возросло


### 6. Создайте новый кластер с включенной контрольной суммой страниц. Создайте таблицу. Вставьте несколько значений. Выключите кластер. Измените пару байт в таблице. Включите кластер и сделайте выборку из таблицы. Что и почему произошло? как проигнорировать ошибку и продолжить работу?


Создаем кластер с контролем четности таблиц

`sudo pg_createcluster 14 test -- --data-checksums`

Запуск кластера

`sudo pg_ctlcluster 14 test start`


Создание тестовой таблицы и наполнение данными
```
sudo -u postgres psql
create table test (i int);
insert into test values (1),(2),(3);
```

Находим расположение таблицы
```
postgres=# SELECT pg_relation_filepath('test');
 pg_relation_filepath 
----------------------
 base/13761/16384
(1 row)
```

Остановка кластера

`sudo -u postgres pg_ctlcluster 14 test stop`

Изменим пару байт в таблицеменяем пару байт в таблице
```
sudo dd if=/dev/zero of=/var/lib/postgresql/14/test/base/13761/16384 oflag=dsync conv=notrunc bs=1 count=8
8+0 records in
8+0 records out
8 bytes copied, 0.0100522 s, 0.8 kB/s
```

Запуск кластера

`sudo pg_ctlcluster 14 test start`

Пытаемся прочитать данные и получаем ошибку
```
sudo -u postgres psql
select * from test;
WARNING:  page verification failed, calculated checksum 41212 but expected 41998
ERROR:  invalid page in block 0 of relation base/13761/16384
```

Игнорирование контрольных сумм

`alter system set ignore_checksum_failure = on;`

`select pg_reload_conf();`

Чтение данных из таблицы

```
select * from test;
WARNING:  page verification failed, calculated checksum 41212 but expected 41998
 i 
---
 1
 2
 3
(3 rows)
```

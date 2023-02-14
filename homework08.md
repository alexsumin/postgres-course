# Домашнее задание

## Нагрузочное тестирование и тюнинг PostgreSQL

## Цель:

• Сделать нагрузочное тестирование PostgreSQL

• Настроить параметры PostgreSQL для достижения максимальной производительности

## Описание/Пошаговая инструкция выполнения домашнего задания:

• Развернуть виртуальную машину любым удобным способом

• Поставить на неё PostgreSQL 14 любым способом

• Настроить кластер PostgreSQL 14 на максимальную производительность не
обращая внимание на возможные проблемы с надежностью в случае
аварийной перезагрузки виртуальной машины

• Нагрузить кластер через утилиту
https://github.com/Percona-Lab/sysbench-tpcc (требует установки
https://github.com/akopytov/sysbench) или через утилиту pgbench (https://postgrespro.ru/docs/postgrespro/14/pgbench)

• Написать какого значения tps удалось достичь, показать какие параметры в
какие значения устанавливали и почему

## Выполнение домашнего задания

Была создана машина в Yandex.Cloud со следующей конфигурацией:
```
Платформа
Intel Ice Lake
Гарантированная доля vCPU       100%
vCPU                            2
RAM                             4 ГБ
Объём дискового пространства    15 ГБ (SSD)
```

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

`alter user postgres password '123';`

Запуск бенчмарка:

`pgbench -i postgres -U postgres -h 127.0.0.1`

`pgbench -c8 -P 60 -T 600 -U postgres -h 127.0.0.1 postgres > default.txt 2>&1`


```
pgbench (14.6 (Ubuntu 14.6-0ubuntu0.22.04.1))
starting vacuum...end.
progress: 60.0 s, 551.0 tps, lat 14.492 ms stddev 12.175
progress: 120.0 s, 598.1 tps, lat 13.376 ms stddev 18.230
progress: 180.0 s, 608.0 tps, lat 13.158 ms stddev 18.370
progress: 240.0 s, 597.0 tps, lat 13.399 ms stddev 24.960
progress: 300.0 s, 652.2 tps, lat 12.265 ms stddev 17.323
progress: 360.0 s, 502.2 tps, lat 15.929 ms stddev 22.658
progress: 420.0 s, 646.5 tps, lat 12.375 ms stddev 18.204
progress: 480.0 s, 630.0 tps, lat 12.697 ms stddev 17.081
progress: 540.0 s, 597.9 tps, lat 13.380 ms stddev 10.006
progress: 600.0 s, 649.5 tps, lat 12.317 ms stddev 8.737
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 600 s
number of transactions actually processed: 361957
latency average = 13.259 ms
latency stddev = 17.393 ms
initial connection time = 90.654 ms
tps = 603.337011 (without initial connection time)
```

Воспользовался рекомендацией по настройке от ресурса https://pgtune.leopard.in.ua/

```ALTER SYSTEM SET
 max_connections = '20';
ALTER SYSTEM SET
 shared_buffers = '1GB';
ALTER SYSTEM SET
 effective_cache_size = '3GB';
ALTER SYSTEM SET
 maintenance_work_mem = '256MB';
ALTER SYSTEM SET
 checkpoint_completion_target = '0.9';
ALTER SYSTEM SET
 wal_buffers = '16MB';
ALTER SYSTEM SET
 default_statistics_target = '100';
ALTER SYSTEM SET
 random_page_cost = '1.1';
ALTER SYSTEM SET
 effective_io_concurrency = '200';
ALTER SYSTEM SET
 work_mem = '13107kB';
ALTER SYSTEM SET
 min_wal_size = '1GB';
ALTER SYSTEM SET
 max_wal_size = '4GB';
 ```

 И дополнительно установил асинхронный режим 
 ```
 ALTER SYSTEM SET synchronous_commit TO off;
 ```

Перазапуск кластер для применения изменений

`sudo systemctl restart postgresql.service`

Результаты:
```
pgbench (14.6 (Ubuntu 14.6-0ubuntu0.22.04.1))
starting vacuum...end.
progress: 60.0 s, 2236.2 tps, lat 3.571 ms stddev 2.238
progress: 120.0 s, 2292.6 tps, lat 3.489 ms stddev 1.002
progress: 180.0 s, 2328.1 tps, lat 3.435 ms stddev 0.988
progress: 240.0 s, 2302.2 tps, lat 3.474 ms stddev 3.202
progress: 300.0 s, 2291.0 tps, lat 3.491 ms stddev 3.592
progress: 360.0 s, 2293.4 tps, lat 3.487 ms stddev 2.026
progress: 420.0 s, 2292.7 tps, lat 3.488 ms stddev 4.369
progress: 480.0 s, 2271.7 tps, lat 3.521 ms stddev 4.776
progress: 540.0 s, 2267.1 tps, lat 3.528 ms stddev 4.501
progress: 600.0 s, 2277.8 tps, lat 3.511 ms stddev 1.019
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 600 s
number of transactions actually processed: 1371184
latency average = 3.499 ms
latency stddev = 3.119 ms
initial connection time = 96.849 ms
tps = 2285.619576 (without initial connection time)
```

Судя по опыту одного из предыдущих заданий, то наибольшую увеличение производительности произошло из-за выключения синхронного коммита.
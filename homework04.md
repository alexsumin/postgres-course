# Домашнее задание

## Работа с базами данных, пользователями и правами

### Цель:
• создание новой базы данных, схемы и таблицы

• создание роли для чтения данных из созданной схемы созданной базы данных

• создание роли для чтения и записи из созданной схемы созданной базы данных

### Описание/Пошаговая инструкция выполнения домашнего задания:

1 создайте новый кластер PostgresSQL 14

2 зайдите в созданный кластер под пользователем postgres

3 создайте новую базу данных testdb

4 зайдите в созданную базу данных под пользователем postgres

5 создайте новую схему testnm

6 создайте новую таблицу t1 с одной колонкой c1 типа integer

7 вставьте строку со значением c1=1

8 создайте новую роль readonly

9 дайте новой роли право на подключение к базе данных testdb

10 дайте новой роли право на использование схемы testnm

11 дайте новой роли право на select для всех таблиц схемы testnm

12 создайте пользователя testread с паролем test123

13 дайте роль readonly пользователю testread

14 зайдите под пользователем testread в базу данных testdb

15 сделайте select * from t1;

16 получилось? (могло если вы делали сами не по шпаргалке и не упустили один существенный момент про который позже)

17 напишите что именно произошло в тексте домашнего задания

18 у вас есть идеи почему? ведь права то дали?

19 посмотрите на список таблиц

20 подсказка в шпаргалке под пунктом 20

21 а почему так получилось с таблицей (если делали сами и без шпаргалки то может у вас все нормально)

22 вернитесь в базу данных testdb под пользователем postgres

23 удалите таблицу t1

24 создайте ее заново но уже с явным указанием имени схемы testnm

25 вставьте строку со значением c1=1

26 зайдите под пользователем testread в базу данных testdb

27 сделайте select * from testnm.t1;

28 получилось?

29 есть идеи почему? если нет - смотрите шпаргалку

30 как сделать так чтобы такое больше не повторялось? если нет идей - смотрите шпаргалку

31 сделайте select * from testnm.t1;

32 получилось?

33 есть идеи почему? если нет - смотрите шпаргалку

31 сделайте select * from testnm.t1;

32 получилось?

33 ура!

34 теперь попробуйте выполнить команду create table t2(c1 integer); insert into t2 values (2);

35 а как так? нам же никто прав на создание таблиц и insert в них под ролью readonly?

36 есть идеи как убрать эти права? если нет - смотрите шпаргалку

37 если вы справились сами то расскажите что сделали и почему, если смотрели шпаргалку - объясните что сделали и почему выполнив указанные в ней команды

38 теперь попробуйте выполнить команду create table t3(c1 integer); insert into t2 values (2);

39 расскажите что получилось и почему

### Выполнение домашнего задания

Была создана машина в Yandex.Cloud.
Затем произведены следующие действия:

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

Cоздание новой базы данных testdb:

`CREATE DATABASE testdb;`

Подключение к созданной базе testdb:

`\c testdb;`

Создание новой схемы testnm:

`CREATE SCHEMA testnm;`

Создание новой таблицы t1 с одной колонкой c1 типа integer:

`create table t1 (c1 integer);`

Вставка строки со значением c1=1:

`insert into t1(c1) values(1);`

Создание новой роли readonly:

`CREATE ROLE readonly;`

Выдача новой роли прав на подключение к базе данных testdb:

`grant connect on DATABASE testdb TO readonly;`

Выдача новой роли право на использование схемы testnm:

`grant usage on SCHEMA testnm to readonly;`

Выдача  новой роли право на select для всех таблиц схемы testnm:

`grant SELECT on all TABLEs in SCHEMA testnm TO readonly;`

Создание пользователя testread с паролем test123:

`CREATE USER testread with password 'test123';`

Выдача роли readonly пользователю testread:

`grant readonly TO testread;`

Подключение под пользователем testread в базу данных testdb:

`\c testdb testread`

В результате ошибка: 

```
connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: FATAL:  Peer authentication failed for user "testread"
Previous connection kept
```
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


Логин в бд:

`sudo -u postgres psql`

Подключение под пользователем testread в базу данных testdb:

`\c testdb testread`

Запрос:
`select * from t1;`

Результат ошибка:

```
ERROR:  permission denied for table t1
```

Выполним:

`\dt`

В выводе:

```        List of relations
 Schema | Name | Type  |  Owner   
--------+------+-------+----------
 public | t1   | table | postgres
 
```

Из шпаргалки: Видно, что таблица создана в схеме public, а не testnm и прав на public для роли readonly не давали.
Потому что в search_path скорее всего "$user", public при том что схемы $USER нет то таблица по умолчанию создалась в public

Переключение в базу данных testdb под пользователем postgres:

`\c testdb postgres`

Удаление таблицы t1:

`DROP TABLE t1;`

Создание таблицы t1 заново уже с явным указанием имени схемы testnm:

`CREATE TABLE testnm.t1(c1 integer);`

Вставка строки со значением c1=1:

`INSERT INTO testnm.t1 VALUES(1);`

Подключение под пользователем testread в базу данных testdb:

`\c testdb testread`

Запрос:

`select * from testnm.t1;`


Из шпаргалки: так произошло потому что `grant SELECT on all TABLEs in SCHEMA testnm TO readonly` дал доступ только для существующих на тот момент времени таблиц, а t1 пересоздавалась:

Переключение на пользователя postgres, выдача дефолтных привилегий и переключение на пользователя testread:

```
\c testdb postgres;
ALTER default privileges in SCHEMA testnm grant SELECT on TABLEs to readonly;
\c testdb testread;
```

Запрос:

`select * from testnm.t1;`

Результат:
```
 c1 
----
  1
(1 row)
```

Переключение:

`\c testdb testread; `

Создание таблицы и вставка:

`create table t2(c1 integer); 
insert into t2 values (2);`

Результат:
```
CREATE TABLE
INSERT 0 1
```


Из шпаргалки: это получилось потому что search_path указывает в первую очередь на схему public. 

А схема public создается в каждой базе данных по умолчанию. 

И grant на все действия в этой схеме дается роли public. 
А роль public добавляется всем новым пользователям. 

Соответсвенно каждый пользователь может по умолчанию создавать объекты в схеме public любой базы данных, естественно если у него есть право на подключение к этой базе данных. 

Чтобы раз и навсегда забыть про роль public - а в продакшн базе данных про нее лучше забыть - выполните следующие действия:

```
\c testdb postgres; 
revoke CREATE on SCHEMA public FROM public; 
revoke all on DATABASE testdb FROM public; 
\c testdb testread;
```

Создание таблицы и вставка:

`create table t3(c1 integer); 
insert into t2 values (2);`

Результат:
```
ERROR:  permission denied for schema public
LINE 1: create table t3(c1 integer);
```

Теперь пользователю нельзя создать объект в схеме public, вставка в существующую таблицу прошла успешно.


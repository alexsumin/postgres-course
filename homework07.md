# Домашнее задание

## Механизм блокировок

## Цель:

Понимать как работает механизм блокировок объектов и строк

## Описание/Пошаговая инструкция выполнения домашнего задания:

1. Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд. Воспроизведите ситуацию, при которой в журнале появятся такие сообщения.

2. Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны. Пришлите список блокировок и объясните, что значит каждая.

3. Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?

4. Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?

## Выполнение домашнего задания

### 1. Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд. Воспроизведите ситуацию, при которой в журнале появятся такие сообщения.


Была создана виртуальная машина в Yandex Cloud.

Подключение к postgres

`sudo -u postgres psql`

включение логирования информации блокировках

`ALTER SYSTEM SET log_lock_waits = on;`

Установка времени для таких блокировок 200 мс(по умолчания 1с)

`ALTER SYSTEM SET deadlock_timeout TO 200;`

Применение изменений

`select pg_reload_conf();`

Просмотр установленного времени таймаута блокировок

`show deadlock_timeout;`

Результат:

```
 deadlock_timeout 
------------------
 200ms
(1 row)
```

Подготовим данные для моделирования ситуаций:

```create table persons(id int primary key, name text); 
 insert into persons(id, name) values(1, 'ivan'); 
 insert into persons(id, name) values(2, 'oleg');
 insert into persons(id, name) values(3, 'artem');
 ``` 
 



В сессии 1:
```begin; 
SELECT txid_current(), pg_backend_pid();
select * from persons where id=1 for update;
```
```
 txid_current | pg_backend_pid 
--------------+----------------
          754 |           5655
```

В сессии 2:

```
begin;
SELECT txid_current(), pg_backend_pid();
select * from persons where id=2 for update;
```
```
 txid_current | pg_backend_pid 
--------------+----------------
          756 |           5614
```

В сессии 1:

`update persons set name='session1' where id=2;`


В сессии 2:

`select * from persons where id=2 for update;`
```
ERROR:  deadlock detected
DETAIL:  Process 5614 waits for ShareLock on transaction 754; blocked by process 5655.
Process 5655 waits for ShareLock on transaction 756; blocked by process 5614.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,1) in relation "persons"
```
Вторая сессия ролбекнулась.

В сессии 1

`commit;`

`select * from persons;`
```
 id |   name   
----+----------
  1 | ivan
  2 | session1
  ````
Успешно завершилась.

В журнале есть запись о дедлоке
`sudo less /var/log/postgresql/postgresql-14-main.log`

```
2023-02-08 19:13:39.093 UTC [5614] postgres@postgres LOG:  process 5614 detected deadlock while waiting for ShareLock on transaction 754 after 200.081 ms
2023-02-08 19:13:39.093 UTC [5614] postgres@postgres DETAIL:  Process holding the lock: 5655. Wait queue: .
2023-02-08 19:13:39.093 UTC [5614] postgres@postgres CONTEXT:  while updating tuple (0,1) in relation "persons"
2023-02-08 19:13:39.093 UTC [5614] postgres@postgres STATEMENT:  update persons set name='session2' where id=1;
2023-02-08 19:13:39.093 UTC [5614] postgres@postgres ERROR:  deadlock detected
2023-02-08 19:13:39.093 UTC [5614] postgres@postgres DETAIL:  Process 5614 waits for ShareLock on transaction 754; blocked by process 5655.
        Process 5655 waits for ShareLock on transaction 756; blocked by process 5614.
        Process 5614: update persons set name='session2' where id=1;
        Process 5655: update persons set name='session1' where id=2;
2023-02-08 19:13:39.093 UTC [5614] postgres@postgres HINT:  See server log for query details.
2023-02-08 19:13:39.093 UTC [5614] postgres@postgres CONTEXT:  while updating tuple (0,1) in relation "persons"
2023-02-08 19:13:39.093 UTC [5614] postgres@postgres STATEMENT:  update persons set name='session2' where id=1;
```

---

### 2. Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны. Пришлите список блокировок и объясните, что значит каждая.

Создание вьюхи для удобного анализа (из материалов лекции):
```CREATE VIEW locks_v AS
SELECT pid,
       locktype,
       CASE locktype
         WHEN 'relation' THEN relation::regclass::text
         WHEN 'transactionid' THEN transactionid::text
         WHEN 'tuple' THEN relation::regclass::text||':'||tuple::text
       END AS lockid,
       mode,
       granted
FROM pg_locks
WHERE locktype in ('relation','transactionid','tuple')
AND (locktype != 'relation' OR relation = 'persons'::regclass);
```

В сессии 1:
```begin;
SELECT txid_current(), pg_backend_pid();
update persons set name='session1' where id=2;
```
Вывод:
```
txid_current | pg_backend_pid 
--------------+----------------
          762 |           5655
```

В сессии 2:
```begin;
SELECT txid_current(), pg_backend_pid();
update persons set name='session2' where id=2;
```
Вывод:
```
txid_current | pg_backend_pid 
--------------+----------------
          764 |           6817
```


В сессии 3:
```begin;
SELECT txid_current(), pg_backend_pid();
update persons set name='session3' where id=2;
```
Вывод:
```
txid_current | pg_backend_pid 
--------------+----------------
          763 |           6924
```

В первой сессии изучаем информацию из созданной вначале вьюхи:

```postgres=*# select * from locks_v where pid=5655;
 pid  |   locktype    | lockid  |       mode       | granted 
------+---------------+---------+------------------+---------
 5655 | relation      | persons | RowExclusiveLock | t
 5655 | transactionid | 762     | ExclusiveLock    | t
(2 rows)

postgres=*# select * from locks_v where pid=6924;
 pid  |   locktype    |  lockid   |       mode       | granted 
------+---------------+-----------+------------------+---------
 6924 | relation      | persons   | RowExclusiveLock | t
 6924 | transactionid | 763       | ExclusiveLock    | t
 6924 | transactionid | 762       | ShareLock        | f
 6924 | tuple         | persons:6 | ExclusiveLock    | t
(4 rows)

postgres=*# select * from locks_v where pid=6817;
 pid  |   locktype    |  lockid   |       mode       | granted 
------+---------------+-----------+------------------+---------
 6817 | relation      | persons   | RowExclusiveLock | t
 6817 | transactionid | 764       | ExclusiveLock    | t
 6817 | tuple         | persons:6 | ExclusiveLock    | f
(3 rows)
```

Все 3 соединения наложили экслюзивную блокировку RowExclusiveLock на строку таблицы persons.
Соединение 762 заблокировало строку, соединение 764 создало очередь ожидания tuple, а соединение 763 присоединилось к этой очереди в ожидание.

---

### 3. Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?

В сессии 1:
```begin;
update persons set name = 'session1' where id = 1;
```

Во сессии 2:
```begin;
update persons set name = 'session2' where id = 2;
```

В сессии 3:
```begin;
update persons set name = 'session3' where id = 3;
```

В сессии 1:

```
update persons set name = 'session1' where id = 2;
```

Во сессии 2:

```
update persons set name = 'session2' where id = 3;
```

В сессии 3:

```
update persons set name = 'session3' where id = 1;
```

Первые две сессии зависают, третья выдает сообщение о дедлоке
```
ERROR:  deadlock detected
DETAIL:  Process 7652 waits for ShareLock on transaction 774; blocked by process 7644.
Process 7644 waits for ShareLock on transaction 775; blocked by process 7648.
Process 7648 waits for ShareLock on transaction 776; blocked by process 7652.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,1) in relation "persons"
```

В журнале есть запись о дедлоке
`sudo less /var/log/postgresql/postgresql-14-main.log`

```
2023-02-08 21:01:57.195 UTC [7644] postgres@postgres LOG:  process 7644 still waiting for ShareLock on transaction 775 after 200.168 ms
2023-02-08 21:01:57.195 UTC [7644] postgres@postgres DETAIL:  Process holding the lock: 7648. Wait queue: 7644.
2023-02-08 21:01:57.195 UTC [7644] postgres@postgres CONTEXT:  while updating tuple (0,6) in relation "persons"
2023-02-08 21:01:57.195 UTC [7644] postgres@postgres STATEMENT:  update persons set name = 'session1' where id = 2;
2023-02-08 21:02:13.815 UTC [7648] postgres@postgres LOG:  process 7648 still waiting for ShareLock on transaction 776 after 200.129 ms
2023-02-08 21:02:13.815 UTC [7648] postgres@postgres DETAIL:  Process holding the lock: 7652. Wait queue: 7648.
2023-02-08 21:02:13.815 UTC [7648] postgres@postgres CONTEXT:  while updating tuple (0,12) in relation "persons"
2023-02-08 21:02:13.815 UTC [7648] postgres@postgres STATEMENT:  update persons set name = 'session2' where id = 3;
2023-02-08 21:02:25.635 UTC [7652] postgres@postgres LOG:  process 7652 detected deadlock while waiting for ShareLock on transaction 774 after 200.147 ms
2023-02-08 21:02:25.635 UTC [7652] postgres@postgres DETAIL:  Process holding the lock: 7644. Wait queue: .
2023-02-08 21:02:25.635 UTC [7652] postgres@postgres CONTEXT:  while updating tuple (0,1) in relation "persons"
2023-02-08 21:02:25.635 UTC [7652] postgres@postgres STATEMENT:  update persons set name = 'session3' where id = 1;
2023-02-08 21:02:25.635 UTC [7652] postgres@postgres ERROR:  deadlock detected
2023-02-08 21:02:25.635 UTC [7652] postgres@postgres DETAIL:  Process 7652 waits for ShareLock on transaction 774; blocked by process 7644.
        Process 7644 waits for ShareLock on transaction 775; blocked by process 7648.
        Process 7648 waits for ShareLock on transaction 776; blocked by process 7652.
        Process 7652: update persons set name = 'session3' where id = 1;
        Process 7644: update persons set name = 'session1' where id = 2;
        Process 7648: update persons set name = 'session2' where id = 3;
2023-02-08 21:02:25.635 UTC [7652] postgres@postgres HINT:  See server log for query details.
2023-02-08 21:02:25.635 UTC [7652] postgres@postgres CONTEXT:  while updating tuple (0,1) in relation "persons"
2023-02-08 21:02:25.635 UTC [7652] postgres@postgres STATEMENT:  update persons set name = 'session3' where id = 1;
2023-02-08 21:02:25.635 UTC [7648] postgres@postgres LOG:  process 7648 acquired ShareLock on transaction 776 after 12020.426 ms
2023-02-08 21:02:25.635 UTC [7648] postgres@postgres CONTEXT:  while updating tuple (0,12) in relation "persons"
2023-02-08 21:02:25.635 UTC [7648] postgres@postgres STATEMENT:  update persons set name = 'session2' where id = 3;
```

В журнале можно увидеть что процесс 7652(третья сессия) зафиксировал deadlock и спустя deadlock_timeout(200мс)транзакция завершается неудачно.

---

### 4. Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?

Гугл подсказал такой вариант:

Подготовка данных, создание таблицы и индекса:

```create table test(id bigint);
insert into test (id) (select generate_series(1, 10000000));
create index on test(id desc);
```
Запрещаем последовательное сканирование:

`set enable_seqscan = off;`

В первой транзакции обновление id:

`update test SET id = id + 1;`

И во второй транзакции обновление id:
`udate test SET id = id + 10000000;`

Если одни и те же строки попали на апдейт в разные транзакции в разном порядке, тогда будет deadlock.

Но воспроизвести ситуацию не виртуальной машине не удалось.


---

Вариант предложенный преподавателем

Deadlock in UPDATE without WHERE and SELECT

Create and populate a new table:

```
CREATE TABLE test2(
id INTEGER,
name VARCHAR,
eman VARCHAR
);

INSERT into test2 (id) VALUES (0),(1),(2),(3),(4),(5),(6),(7);
INSERT into test2 (id) VALUES (0),(1),(2),(3),(4),(5),(6),(7);

On session 1:

otus=# BEGIN;
BEGIN
otus=*# UPDATE test2 SET name=id RETURNING *,pg_sleep(5),pg_advisory_lock(id);

Then we wait 10 seconds and run on session 2:

otus=# BEGIN;
BEGIN
otus=*# VALUES(pg_advisory_lock(5));
otus=*# UPDATE test2 SET name=id RETURNING *,pg_sleep(5),pg_advisory_lock(5+id);

And waiting on session 1:

ERROR: deadlock detected
DETAIL: Process 6314 waits for ExclusiveLock on advisory lock [16384,0,5,1]; blocked by process 7190.
Process 7190 waits for ShareLock on transaction 820; blocked by process 6314.
HINT: See server log for query details.
```
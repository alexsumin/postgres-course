# Домашнее задание

## Репликация

## Цель: реализовать свой миникластер на 3 ВМ.

## Описание/Пошаговая инструкция выполнения домашнего задания:

На 1 ВМ создаем таблицы test для записи, test2 для запросов на чтение. Создаем публикацию таблицы test и подписываемся на публикацию таблицы test2 с ВМ №2. На 2 ВМ создаем таблицы test2 для записи, test для запросов на чтение. Создаем публикацию таблицы test2 и подписываемся на публикацию таблицы test1 с ВМ №1. 3 ВМ использовать как реплику для чтения и бэкапов (подписаться на таблицы из ВМ №1 и №2 ). Небольшое описание, того, что получилось.

---

## Выполнение домашнего задания

Созданы 3 виртуальные машины:

• **host01**

• **host02**

• **host03**


На каждой вм выполнена преднастройка:

```
echo "listen_addresses = '*'" | sudo tee -a /etc/postgresql/14/main/postgresql.conf
echo "wal_level = 'logical'" | sudo tee -a /etc/postgresql/14/main/postgresql.conf
echo "host all all 0.0.0.0/0 md5" | sudo tee -a /etc/postgresql/14/main/pg_hba.conf
echo "host replication all 0.0.0.0/0 md5" | sudo tee -a /etc/postgresql/14/main/pg_hba.conf

sudo systemctl restart postgresql

sudo -u postgres psql -c "CREATE USER testuser SUPERUSER encrypted PASSWORD '123'"
```

---

На вм **host01**:

Cоздаем таблицы test1 для записи, test2 для запросов на чтение:

```
sudo -u postgres psql -c "CREATE DATABASE testdb"
sudo -u postgres psql testdb -c "CREATE TABLE test1 (t text)"
sudo -u postgres psql testdb -c "CREATE TABLE test2 (t text)"
```


Создаем публикацию таблицы test1 


```
sudo -u postgres psql testdb -c "CREATE PUBLICATION test_pub01 FOR TABLE test1"
```

Подписываемся на публикацию таблицы test2 с ВМ host02.

```
sudo -u postgres psql testdb -c "CREATE SUBSCRIPTION test_sub01 CONNECTION 'host=host01 port=5432 user=testuser password=123 dbname=testdb' PUBLICATION test_pub01 WITH (copy_data = true)"
```

---

На вм **host02**: 

Создаем таблицы test2 для записи, test1 для запросов на чтение.

```
sudo -u postgres psql -c "CREATE DATABASE testdb"
sudo -u postgres psql testdb -c "CREATE TABLE test1 (t text)"
sudo -u postgres psql testdb -c "CREATE TABLE test2 (t text)"
```

Создаем публикацию таблицы test2:

```
sudo -u postgres psql testdb -c "CREATE PUBLICATION test_pub02 FOR TABLE test2"
```

Подписываемся на публикацию таблицы test1 с ВМ host01:

```
sudo -u postgres psql testdb -c "CREATE SUBSCRIPTION test_sub02 CONNECTION 'host=host02 port=5432 user=testuser password=123 dbname=testdb' PUBLICATION test_pub02 WITH (copy_data = true)"
```

---

## Проверка работоспособности

На вм **host01**:
```
sudo -u postgres psql testdb -c "INSERT INTO test1 VALUES ('test1')"
```


На вм **host02**:
```
sudo -u postgres psql testdb -c "INSERT INTO test2 VALUES ('test2')"
```

На вм **host01**:
```
sudo -u postgres psql testdb -c "select * from test2"
```

На вм **host02**:
```
sudo -u postgres psql testdb -c "select * from test1"
```

---

Третью вм host03 используем как реплику для чтения и бекапирования (для таблиц из первых двум вм host01 и host02)


На вм **host03**:

```
sudo -u postgres psql testdb -c "CREATE SUBSCRIPTION test_sbsrtptn_host01 CONNECTION 'host=host01 port=5432 user=testuser password=123 dbname=testdb' PUBLICATION test_pub01 WITH (copy_data = true)"

sudo -u postgres psql testdb -c "CREATE SUBSCRIPTION test_sbsrtptn_host02 CONNECTION 'host=host02 port=5432 user=testuser password=123 dbname=testdb' PUBLICATION test_pub02 WITH (copy_data = true)"
```

---

### Получилась следующая схема:


• **host01**.test1 --> **host02**.test1 

• **host02**.test2 --> **host01**.test2

### Реплика

• **host01**.test1 --> **host03**.test1 

• **host01**.test2 --> **host03**.test2

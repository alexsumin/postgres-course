# Домашнее задание

## Секционирование таблицы

## Цель: 

• научиться секционировать таблицы

## Описание/Пошаговая инструкция выполнения домашнего задания:

Секционировать большую таблицу из демо базы flights

---

## Выполнение домашнего задания

Используется тестовая БД https://edu.postgrespro.ru/demo-medium.zip

Создаем секционированную таблицу **bookings_RANGE**, метод разбиения **RANGE**, ключом будет **book_date**:

```
CREATE TABLE bookings.bookings_RANGE (
    book_ref bpchar(6) NOT NULL,
    book_date timestamptz NOT NULL,
    total_amount numeric(10, 2) NOT NULL
) PARTITION by RANGE (book_date);
```

Источником данных для созданной таблицы будет существующая таблица **bookings.bookings**

---

Запрос всех **book_date** с группировкой по году и месяцу:

```
SELECT 
EXTRACT(year FROM book_date)::char(4) || '-' || EXTRACT(month FROM book_date)::char(2)
FROM bookings.bookings
group by EXTRACT(year FROM book_date)::char(4) || '-' || EXTRACT(month FROM book_date)::char(2);
```

Получим:

```
2017-4
2017-5
2017-6
2017-7
2017-8
```

---

Создание секций:

```
CREATE TABLE bookings_RANGE_201704 PARTITION OF bookings_RANGE
FOR VALUES FROM ('2017-04-01') TO ('2017-05-01');
    
CREATE TABLE bookings_RANGE_201705 PARTITION OF bookings_RANGE
FOR VALUES FROM ('2017-05-01') TO ('2017-06-01');
    
CREATE TABLE bookings_RANGE_201706 PARTITION OF bookings_RANGE
FOR VALUES FROM ('2017-06-01') TO ('2017-07-01');
    
CREATE TABLE bookings_RANGE_201707 PARTITION OF bookings_RANGE
FOR VALUES FROM ('2017-07-01') TO ('2017-08-01');
    
CREATE TABLE bookings_RANGE_201708 PARTITION OF bookings_RANGE
FOR VALUES FROM ('2017-08-01') TO ('2017-09-01');
```

---

Создание секции по умолчанию:

```
CREATE TABLE bookings_RANGE_DEFAULT PARTITION OF bookings_RANGE DEFAULT;
```

---

Заполняем секционированную таблицу **bookings_range** данными из несекционированной **bookings**:

```
INSERT INTO bookings_range SELECT * FROM bookings;  
```

---

Создание первичного ключа:

```
ALTER TABLE bookings.bookings_range ADD CONSTRAINT bookings_range_pkey PRIMARY KEY (book_date, book_ref);
```

При выполнении планеа запроса можно увидеть, что поиск осуществляется только в определенной секции.
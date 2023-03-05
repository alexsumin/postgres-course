# Домашнее задание

## Работа с индексами, join'ами, статистикой

## Цель: 

• знать и уметь применять основные виды индексов PostgreSQL

• строить и анализировать план выполнения запроса

• уметь оптимизировать запросы для с использованием индексов

• знать и уметь применять различные виды join'ов

• строить и анализировать план выполенения запроса

• оптимизировать запрос

• уметь собирать и анализировать статистику для таблицы

## Описание/Пошаговая инструкция выполнения домашнего задания:

1 вариант:
Создать индексы на БД, которые ускорят доступ к данным.
В данном задании тренируются навыки:

• определения узких мест

• написания запросов для создания индекса

• оптимизации


Необходимо:
1. Создать индекс к какой-либо из таблиц вашей БД
2. Прислать текстом результат команды explain,
в которой используется данный индекс
3. Реализовать индекс для полнотекстового поиска
4. Реализовать индекс на часть таблицы или индекс
на поле с функцией
5. Создать индекс на несколько полей
6. Написать комментарии к каждому из индексов
7. Описать что и как делали и с какими проблемами
столкнулись


2 вариант:
В результате выполнения ДЗ вы научитесь пользоваться
различными вариантами соединения таблиц.


В данном задании тренируются навыки:

• написания запросов с различными типами соединений

Необходимо:
1. Реализовать прямое соединение двух или более таблиц
2. Реализовать левостороннее (или правостороннее) соединение двух или более таблиц
3. Реализовать кросс соединение двух или более таблиц
4. Реализовать полное соединение двух или более таблиц
5. Реализовать запрос, в котором будут использованы
разные типы соединений
6. Сделать комментарии на каждый запрос
7. К работе приложить структуру таблиц, для которых
выполнялись соединения

---

## Выполнение домашнего задания

Используется тестовая БД https://edu.postgrespro.ru/demo-medium.zip

1 вариант

1. Создать индекс к какой-либо из таблиц вашей БД


Создание индекса:
```
create index ticket_flights_fare_conditions_idx on ticket_flights (fare_conditions);
```

Запрос выбор стоимости билетов на места бизнес-класса с группировкой по номеру самолета, среди билетов по которым небыло посадки в самолет.

```
select
    f.flight_no,
    SUM(tf.amount)
from
    ticket_flights tf
inner join
flights f 
on
    tf.flight_id = f.flight_id
left join
boarding_passes bp 
on
    bp.ticket_no = tf.ticket_no
where
    bp.ticket_no isnull
    and tf.fare_conditions = 'Business'
group by
    f.flight_no
order by
    SUM(tf.amount) desc
limit 10
;
```

План выполнения запроса ДО создания индекса:


    QUERY PLAN                                                                                                                               
    -----------------------------------------------------------------------------------------------------------------------------------------
    Limit  (cost=77612.50..77612.53 rows=10 width=39)                                                                                        
    ->  Sort  (cost=77612.50..77614.28 rows=710 width=39)                                                                                  
            Sort Key: (sum(tf.amount)) DESC                                                                                                  
            ->  Finalize GroupAggregate  (cost=77411.96..77597.16 rows=710 width=39)                                                         
                Group Key: f.flight_no                                                                                                     
                ->  Gather Merge  (cost=77411.96..77577.64 rows=1420 width=39)                                                             
                        Workers Planned: 2                                                                                                   
                        ->  Sort  (cost=76411.93..76413.71 rows=710 width=39)                                                                
                            Sort Key: f.flight_no                                                                                          
                            ->  Partial HashAggregate  (cost=76369.43..76378.31 rows=710 width=39)                                         
                                    Group Key: f.flight_no                                                                                   
                                    ->  Hash Join  (cost=38132.46..76270.65 rows=19757 width=13)                                             
                                        Hash Cond: (tf.flight_id = f.flight_id)                                                            
                                        ->  Parallel Hash Anti Join  (cost=35542.02..73113.35 rows=19757 width=10)                         
                                                Hash Cond: (tf.ticket_no = bp.ticket_no)                                                     
                                                ->  Parallel Seq Scan on ticket_flights tf  (cost=0.00..31963.41 rows=102019 width=24)       
                                                    Filter: ((fare_conditions)::text = 'Business'::text)                                   
                                                ->  Parallel Hash  (cost=21821.90..21821.90 rows=789290 width=14)                            
                                                    ->  Parallel Seq Scan on boarding_passes bp  (cost=0.00..21821.90 rows=789290 width=14)
                                        ->  Hash  (cost=1448.64..1448.64 rows=65664 width=11)                                              
                                                ->  Seq Scan on flights f  (cost=0.00..1448.64 rows=65664 width=11)                          


План выполнения запроса ПОСЛЕ создания индекса:


    QUERY PLAN                                                                                                                                                                   
    -----------------------------------------------------------------------------------------------------------------------------------------------------------------------------
    Limit  (cost=68995.05..68995.08 rows=10 width=39)                                                                                                                            
    ->  Sort  (cost=68995.05..68996.83 rows=710 width=39)                                                                                                                      
            Sort Key: (sum(tf.amount)) DESC                                                                                                                                      
            ->  Finalize GroupAggregate  (cost=68794.51..68979.71 rows=710 width=39)                                                                                             
                Group Key: f.flight_no                                                                                                                                         
                ->  Gather Merge  (cost=68794.51..68960.19 rows=1420 width=39)                                                                                                 
                        Workers Planned: 2                                                                                                                                       
                        ->  Sort  (cost=67794.48..67796.26 rows=710 width=39)                                                                                                    
                            Sort Key: f.flight_no                                                                                                                              
                            ->  Partial HashAggregate  (cost=67751.98..67760.86 rows=710 width=39)                                                                             
                                    Group Key: f.flight_no                                                                                                                       
                                    ->  Hash Join  (cost=38132.89..67647.74 rows=20848 width=13)                                                                                 
                                        Hash Cond: (tf.flight_id = f.flight_id)                                                                                                
                                        ->  Parallel Hash Anti Join  (cost=35542.45..64477.58 rows=20848 width=10)                                                             
                                                Hash Cond: (tf.ticket_no = bp.ticket_no)                                                                                         
                                                ->  Parallel Index Scan using ticket_flights_fare_conditions_idx on ticket_flights tf  (cost=0.43..23336.76 rows=100740 width=24)
                                                    Index Cond: ((fare_conditions)::text = 'Business'::text)                                                                   
                                                ->  Parallel Hash  (cost=21821.90..21821.90 rows=789290 width=14)                                                                
                                                    ->  Parallel Seq Scan on boarding_passes bp  (cost=0.00..21821.90 rows=789290 width=14)                                    
                                        ->  Hash  (cost=1448.64..1448.64 rows=65664 width=11)                                                                                  
                                                ->  Seq Scan on flights f  (cost=0.00..1448.64 rows=65664 width=11)                                                              

Видно, что используется созданный индекс:

```
Parallel Index Scan using ticket_flights_fare_conditions_idx on ticket_flights tf  (cost=0.43..23336.76 rows=100740 width=24)
        Index Cond: ((fare_conditions)::text = 'Business'::text)  
```

---

2. Прислать текстом результат команды explain,
в которой используется данный индекс

Запрос:

```
select  
    *  
from  
    bookings.airports  
where  
    airport_code = 'AER'  
```

план выполнения запроса:

```
Index Scan using airports_airport_code_ix on airports as airports (rows=1 loops=1)
 Index Cond: (airport_code = 'AER'::bpchar)
 ```

---

3. Реализовать индекс для полнотекстового поиска

```
create index passenger_name_gin on tickets using gin (to_tsvector('english', passenger_name));
```

Такой тип индекса может использоваться для неточного поиска пассажира по фамилии, имени или их части.

Пример запроса, использующего такой индекс:

```
select
t.ticket_no ,
t.passenger_name 
from tickets t 
where t.passenger_name @@ to_tsquery('english', 'VLADIMIR')
limit 10;
```

---

4. Реализовать индекс на часть таблицы или индекс
на поле с функцией

Создание индекса на часть таблицы

```
create index ticket_flights_amount_idx on ticket_flights (amount) where fare_conditions <> 'Economy'; 
```

---

5. Создать индекс на несколько полей

```
create index aircrafts_data_aircraft_code_range_idx on aircrafts_data (aircraft_code, range);
```

Пример запроса с кросс-джоин соединением, каждая строка таблицы ticket_flights соединяется со строкой таблицы boarding_passes:

```
select
tf.ticket_no,
bp.ticket_no 
from 
ticket_flights tf 
cross join
boarding_passes bp;
```

Пример запроса с полным соединением таблиц ticket_flights и boarding_passes с соединением по номеру билета, включая пустые значения в певрой и второй таблице:

```
select
tf.ticket_no,
bp.ticket_no 
from 
ticket_flights tf
full join
boarding_passes bp 
on tf.ticket_no  = bp.ticket_no;
```

И, конечно же, предыдущий запрос можно заменить на:

```
select
tf.ticket_no,
bp.ticket_no 
from 
ticket_flights tf
left join
boarding_passes bp 
on tf.ticket_no  = bp.ticket_no

union

select
tf.ticket_no,
bp.ticket_no 
from 
ticket_flights tf
right join
boarding_passes bp 
on tf.ticket_no  = bp.ticket_no;
```

---
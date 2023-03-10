# Домашнее задание

## Триггеры, поддержка заполнения витрин

## Цель: 

• Создать триггер для поддержки витрины в актуальном состоянии.

## Описание/Пошаговая инструкция выполнения домашнего задания:

• Скрипт и развернутое описание задачи – в ЛК (файл hw_triggers.sql) или по ссылке: https://disk.yandex.ru/d/l70AvknAepIJXQ

• В БД создана структура, описывающая товары (таблица goods) и продажи (таблица sales).

• Есть запрос для генерации отчета – сумма продаж по каждому товару.

• БД была денормализована, создана таблица (витрина), структура которой повторяет структуру отчета.

• Создать триггер на таблице продаж, для поддержки данных в витрине в актуальном состоянии (вычисляющий при каждой продаже сумму и записывающий её в витрину)
Подсказка: не забыть, что кроме INSERT есть еще UPDATE и DELETE

• Задание со звездочкой*
Чем такая схема (витрина+триггер) предпочтительнее отчета, создаваемого "по требованию" (кроме производительности)?
Подсказка: В реальной жизни возможны изменения цен.

---

## Выполнение домашнего задания

Создание схемы из скрипта:

```
-- ДЗ тема: триггеры, поддержка заполнения витрин

DROP SCHEMA IF EXISTS pract_functions CASCADE;
CREATE SCHEMA pract_functions;

SET search_path = pract_functions, publ;

-- товары:
CREATE TABLE goods
(
    goods_id    integer PRIMARY KEY,
    good_name   varchar(63) NOT NULL,
    good_price  numeric(12, 2) NOT NULL CHECK (good_price > 0.0)
);
INSERT INTO goods (goods_id, good_name, good_price)
VALUES 	(1, 'Спички хозайственные', .50),
		(2, 'Автомобиль Ferrari FXX K', 185000000.01);

-- Продажи
CREATE TABLE sales
(
    sales_id    integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    good_id     integer REFERENCES goods (goods_id),
    sales_time  timestamp with time zone DEFAULT now(),
    sales_qty   integer CHECK (sales_qty > 0)
);

INSERT INTO sales (good_id, sales_qty) VALUES (1, 10), (1, 1), (1, 120), (2, 1);

-- отчет:
SELECT G.good_name, sum(G.good_price * S.sales_qty)
FROM goods G
INNER JOIN sales S ON S.good_id = G.goods_id
GROUP BY G.good_name;

-- с увеличением объёма данных отчет стал создаваться медленно
-- Принято решение денормализовать БД, создать таблицу
CREATE TABLE good_sum_mart
(
	good_name   varchar(63) NOT NULL,
	sum_sale	numeric(16, 2)NOT NULL
);

-- Создать триггер (на таблице sales) для поддержки.
-- Подсказка: не забыть, что кроме INSERT есть еще UPDATE и DELETE

-- Чем такая схема (витрина+триггер) предпочтительнее отчета, создаваемого "по требованию" (кроме производительности)?
-- Подсказка: В реальной жизни возможны изменения цен.
```

Создание процедуры для триггера:

```
create 
or replace function good_sum_mart() returns trigger as $$ declare good_name varchar(63);
begin 
-- при инсерте проводим проверку наличия товара в таблице витрины
IF OLD is null THEN 
select 
  case when good_sum_mart.good_name is null then goods.good_name else '' end 
from 
  pract_functions.sales as sales 
  inner join pract_functions.goods as goods on sales.good_id = goods.goods_id 
  left outer join pract_functions.good_sum_mart as good_sum_mart on goods.good_name = good_sum_mart.good_name 
where 
  sales.good_id = new.good_id into good_name;
ELSE 
    select '' into good_name;
END IF;

-- если товара там нет, то добавим пустую запись
IF good_name <> '' THEN 
    insert into pract_functions.good_sum_mart select good_name, 0;
END IF;

-- обновление резултата изменения продаж
update 
  pract_functions.good_sum_mart 
SET 
  sum_sale = sum_sale 
  + CASE WHEN NEW is NULL THEN 0 ELSE NEW.sales_qty * goods.good_price END 
  - CASE WHEN OLD is NULL THEN 0 ELSE OLD.sales_qty * goods.good_price END 
FROM 
  pract_functions.goods as goods 
WHERE 
  pract_functions.good_sum_mart.good_name = goods.good_name 
  and goods.goods_id = CASE WHEN NEW is NULL THEN OLD.good_id ELSE NEW.good_id END;

return null;
end 
$$ 
language plpgsql;*
```

Создание триггера:
```
create trigger sales 
after 
  insert or delete or update 
  on pract_functions.sales 
  FOR EACH ROW EXECUTE function good_sum_mart();

```

Первичное заполнение данными:
```
insert into pract_functions.good_sum_mart 
select 
  goods.good_name, 
  SUM(sales.sales_qty * goods.good_price) 
from 
  pract_functions.goods as goods 
  inner join pract_functions.sales as sales on goods.goods_id = sales.good_id 
group by 
  goods.good_name;
```

---

Возможно некорректное обновление витрины при изменении цены товара, т.к. в таблице продаж не хранится цена, по которой товар был продан.
К примеру, лот1 стоил x рублей, у него изменилась цена в таблице товаров на x+1 рублей. В случае отмены продажи, совершенной по прошлой цене, в отчете спишется некорректная сумма. Если в таблице продаж хранить цену продажи, то такой ситуации не произошло бы.
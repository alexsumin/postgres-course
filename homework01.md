# Домашнее задание

## Работа с уровнями изоляции транзакции в PostgreSQL

### Цель:
научиться работать с Google Cloud Platform на уровне Google Compute Engine (IaaS)ЯндексОБлаке на уровне Compute Cloud
научиться управлять уровнем изоляции транзакций в PostgreSQL и понимать особенность работы уровней read commited и repeatable read

### Описание/Пошаговая инструкция выполнения домашнего задания:
создать новый проект в Google Cloud Platform, например postgres2022-, где yyyymm год, месяц вашего рождения (имя проекта должно быть уникально на уровне GCP), Яндекс облако или на любых ВМ, докере
далее создать инстанс виртуальной машины с дефолтными параметрами
добавить свой ssh ключ в metadata ВМ
зайти удаленным ssh (первая сессия), не забывайте про ssh-add
поставить PostgreSQL
зайти вторым ssh (вторая сессия)
запустить везде psql из под пользователя postgres
выключить auto commit
сделать в первой сессии новую таблицу и наполнить ее данными create table persons(id serial, first_name text, second_name text); insert into persons(first_name, second_name) values('ivan', 'ivanov'); insert into persons(first_name, second_name) values('petr', 'petrov'); commit;
посмотреть текущий уровень изоляции: show transaction isolation level
начать новую транзакцию в обоих сессиях с дефолтным (не меняя) уровнем изоляции
в первой сессии добавить новую запись insert into persons(first_name, second_name) values('sergey', 'sergeev');
сделать select * from persons во второй сессии
видите ли вы новую запись и если да то почему?
завершить первую транзакцию - commit;
сделать select * from persons во второй сессии
видите ли вы новую запись и если да то почему?
завершите транзакцию во второй сессии
начать новые но уже repeatable read транзации - set transaction isolation level repeatable read;
в первой сессии добавить новую запись insert into persons(first_name, second_name) values('sveta', 'svetova');
сделать select * from persons во второй сессии
видите ли вы новую запись и если да то почему?
завершить первую транзакцию - commit;
сделать select * from persons во второй сессии
видите ли вы новую запись и если да то почему?
завершить вторую транзакцию
сделать select * from persons во второй сессии
видите ли вы новую запись и если да то почему? ДЗ сдаем в виде миниотчета в markdown в гите

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

Проверка автокоммита:

`\echo :AUTOCOMMIT`

Отключение автокоммита:
`\set AUTOCOMMIT OFF`


Создание таблицы и заполнение данными:

 `create table persons(id serial, first_name text, second_name text); 
 insert into persons(first_name, second_name) values('ivan', 'ivanov'); 
 insert into persons(first_name, second_name) values('petr', 'petrov'); 
 commit;
 `


Просмотр текущего уровня изоляции транзакций:

`show transaction isolation level;`

Текущий уровень:

`read committed`

В первой сессии:

`insert into persons(first_name, second_name) values('sergey', 'sergeev');`

Во второй сессии:

`select * from persons;`

Не видно новую запись, потому что уровень изоляции транзакций - read commited. А поскольку не было коммита, то и изменения не видны

Коммит изменений в первой сессии:

`commit;`
Во второй сессии:

`select * from persons;`

Видна новая запись, потому что произошел коммит на уровне изоляции транзакций read commited.


Новая транзакция в первой сессии с уровнем repeatable read:

`set transaction isolation level repeatable read;`

Добавление новой записи в первой сессии:

`insert into persons(first_name, second_name) values('sveta', 'svetova');`

Во второй сессии запрос:

`set transaction isolation level repeatable read;
select * from persons;`

Не видно новой записи, потому что уровень изоляции транзакций repeatable read гарантирует что будет видны одни и те же результаты.

Завершение первой транзакции:

`commit;`


Во второй сессии запрос:

`select * from persons;`

Новая запись не видна, так как уровень изоляции repeatable read исключает получение аномалии неповторяющееся чтение.

Во второй сессии:

`commit;`

И запрос:

`select * from persons;`

Видна новая запись, потому что новая транзакция работает с новым снепшотом, в котором уже есть данные.
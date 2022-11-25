
# Домашнее задание

##  Установка и настройка PostgteSQL в контейнере Docker

### Цель:
установить PostgreSQL в Docker контейнере
настроить контейнер для внешнего подключения

### Описание/Пошаговая инструкция выполнения домашнего задания:

• создать ВМ с Ubuntu 20.04/22.04 или развернуть докер любым удобным способом

• поставить на нем Docker Engine

• сделать каталог /var/lib/postgres

• развернуть контейнер с PostgreSQL 14 смонтировав в него /var/lib/postgres

• развернуть контейнер с клиентом postgres

• подключится из контейнера с клиентом к контейнеру с сервером и сделать
таблицу с парой строк

• подключится к контейнеру с сервером с ноутбука/компьютера извне инстансов GCP/ЯО/места установки докера

• удалить контейнер с сервером

• создать его заново

• подключится снова из контейнера с клиентом к контейнеру с сервером

• проверить, что данные остались на месте

• оставляйте в ЛК ДЗ комментарии что и как вы делали и как боролись с проблемами


### Выполнение домашнего задания

Была создана машина в Yandex.Cloud.
Затем произведены следующие действия:

Обновление списка пакетов:

`sudo apt update`

Установка docker:

`
https://docs.docker.com/engine/install/ubuntu/
curl -fsSL https://get.docker.com -o get-docker.sh && sudo sh get-docker.sh && rm get-docker.sh && sudo usermod -aG docker $USER
`

Перелогиниться через ssh

Проверка что docker работает:

`docker run hello-world`

Создание docker-сети: 

`sudo docker network create pg-net`

Создание директории:

`sudo mkdir /var/lib/postgres`

подключаем созданную сеть к контейнеру сервера Postgres:

`sudo docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:14`

Просмотр списка работающих контейнеров:

`docker ps` 

В списке виден pg-server

Запуск отдельного контейнера с клиентом в общей сети с БД:

`sudo docker run -it --rm --network pg-net --name pg-client postgres:14 psql -h pg-server -U postgres`

Ввод пароля:

`postgres`

Во второй сессии:

`docker ps`

Видно работающие контейнеры `pg-server` и `pg-client`


Из первой сессии, через подключение из контейнера с клиентом к контейнеру с сервером и создаем таблицу с данными:

`
create table persons(id serial, first_name text, second_name text); 
 insert into persons(first_name, second_name) values('ivan', 'ivanov'); 
 insert into persons(first_name, second_name) values('petr', 'petrov'); 
 `

Запрос данных из созданной таблицы persons:

`select * from persons`

Видно две записи.


На ноутбуке выполнение установки клиента:

`sudo apt -y install postgresql-client-14`

Коннект с ноутбука к postgres на виртуальной машине:

`psql -p 5432 -U postgres -h ycloud-machine-ip -d postgres -W`

где `ycloud-machine-ip` - ip адрес машины в yandex cloud

Запрос данных из созданной таблицы persons:

`select * from persons;`

Видно две записи

На виртуальной машине удалим контейнер pg-server:

`docker rm -f pg-server`

Проверка, что контейнер удалился:

`docker ps --all`

Видно, что контейнера нет в списке

Заново создаем контейнер и маунтим ту же самую директорию:

`sudo docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:14`

Подключаемся любым способом к postgres, например на виртуальной машине:

`sudo docker run -it --rm --network pg-net --name pg-client postgres:14 psql -h pg-server -U postgres`

 и выполняем запрос:

`select * from persons;`

В результате выполнения запроса видны те самые записи, которые были добавлены. Причина этого - то что примаунчен volume.




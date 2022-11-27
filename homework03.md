# Домашнее задание


## Установка и настройка PostgreSQL

### Цель:
• создавать дополнительный диск для уже существующей виртуальной машины, размечать его и делать на нем файловую систему

• переносить содержимое базы данных PostgreSQL на дополнительный диск

• переносить содержимое БД PostgreSQL между виртуальными машинами

### Описание/Пошаговая инструкция выполнения домашнего задания:

• создайте виртуальную машину c Ubuntu 20.04 LTS (bionic) в GCE/ЯО

• поставьте на нее PostgreSQL 14 через sudo apt

• проверьте что кластер запущен через sudo -u postgres pg_lsclusters

• зайдите из под пользователя postgres в psql и сделайте произвольную таблицу с произвольным содержимым
postgres=# create table test(c1 text);
postgres=# insert into test values('1');
\q

• остановите postgres например через sudo -u postgres pg_ctlcluster 14 main stop

• создайте новый standard persistent диск GKE через Compute Engine -> Disks в том же регионе и зоне что GCE инстанс размером например 10GB

• добавьте свеже-созданный диск к виртуальной машине - надо зайти в режим ее редактирования и дальше выбрать пункт attach existing disk

• поинициализируйте диск согласно инструкции и подмонтировать файловую систему, только не забывайте менять имя диска на актуальное, в вашем случае это скорее всего будет /dev/sdb - https://www.digitalocean.com/community/tutorials/how-to-partition-and-format-storage-devices-in-linux

• перезагрузите инстанс и убедитесь, что диск остается примонтированным (если не так смотрим в сторону fstab)

• сделайте пользователя postgres владельцем /mnt/data - chown -R postgres:postgres /mnt/data/

• перенесите содержимое /var/lib/postgres/14 в /mnt/data - mv /var/lib/postgresql/14 /mnt/data

• попытайтесь запустить кластер - sudo -u postgres pg_ctlcluster 14 main start

• напишите получилось или нет и почему

• задание: найти конфигурационный параметр в файлах раположенных в /etc/postgresql/14/main который надо поменять и поменяйте его

• напишите что и почему поменяли

• попытайтесь запустить кластер - sudo -u postgres pg_ctlcluster 14 main start

• напишите получилось или нет и почему

• зайдите через через psql и проверьте содержимое ранее созданной таблицы

• задание со звездочкой *: не удаляя существующий инстанс ВМ сделайте новый, поставьте на его PostgreSQL, удалите файлы с данными из /var/lib/postgres, перемонтируйте внешний диск который сделали ранее от первой виртуальной машины ко второй и запустите PostgreSQL на второй машине так чтобы он работал с данными на внешнем диске, расскажите как вы это сделали и что в итоге получилось.


### Выполнение домашнего задания

Была создана машина в Yandex.Cloud.
Затем произведены следующие действия:

Обновление списка пакетов:

`sudo apt update`


Установка PostgreSQL:

`sudo apt install postgresql`

Проверка что кластер запущен:  

`sudo -u postgres pg_lsclusters`


Логин под пользователя postgres в psql и создание произвольной таблицы с произвольным содержимым:

`sudo -u postgres psql`

`create table test(c1 text);
postgres=# insert into test values('1');
exit`

Остановка кластера postgres:

`sudo systemctl stop postgresql@14-main`

Проверка что кластер остановлен:  

`sudo -u postgres pg_lsclusters`

```
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 down   postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
```

Остановка машины и создание диска.
После создания и подключения диска и повторный запуск виртуальной машины(ip адрес изменился, при условии что не был подключен статический ip адрес)
 через веб-интерфейс яндекс облака:


`sudo lsblk -o NAME,FSTYPE,SIZE,MOUNTPOINT,LABEL`

```
`NAME   FSTYPE    SIZE MOUNTPOINT        LABEL
loop0  squashfs 63.2M /snap/core20/1695 
loop1  squashfs 61.9M /snap/core20/1405 
loop2  squashfs 79.9M /snap/lxd/22923   
loop3  squashfs  103M /snap/lxd/23541   
loop4  squashfs 44.7M /snap/snapd/15534 
loop5  squashfs 49.6M /snap/snapd/17576 
vda               15G                   
├─vda1             1M                   
└─vda2 ext4       15G /                 
vdb               20G   `  
```

Далее следование туториалу [How To Partition and Format Storage Devices in Linux](https://www.digitalocean.com/community/tutorials/how-to-partition-and-format-storage-devices-in-linux)

Установка утилиты parted:

`sudo apt install parted`

Выбор стандарта партиционирования диска gpt:

`sudo parted /dev/vdb mklabel gpt`

Создание нового раздела(партиции):

`sudo parted -a opt /dev/vdb mkpart primary ext4 0% 100%`

Создание файловой системы ext4 на созданном разделе:

`sudo mkfs.ext4 -L datapartition /dev/vdb1`

Просмотр информации о дисках и созданных на них разделах, их размерах, точке монтирования: 

`sudo lsblk -o NAME,FSTYPE,LABEL,UUID,MOUNTPOINT`

Создание директории:

`sudo mkdir -p /mnt/data`

Монтирование файловой системы временно:

`sudo mount -o defaults /dev/vdb1 /mnt/data`

Чтобы монтирование работало после перезапуска виртуальной машины нужно редактировать fstab:

`sudo nano /etc/fstab`

И добавить в конец файла новую строку, вид итогового содержимого: 

```
`# /etc/fstab: static file system information.
#
# Use 'blkid' to print the universally unique identifier for a
# device; this may be used with UUID= as a more robust way to name devices
# that works even if disks are added and removed. See fstab(5).
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
# / was on /dev/vda2 during curtin installation
/dev/disk/by-uuid/9c32f6b6-779b-4319-b576-165bf47e9034 / ext4 defaults 0 1
UUID=018705ec-5d7c-42b2-95c1-d8c5f77b8593 /mnt/data ext4 defaults 0 2`
```

После перезапуска виртуальной машины можно выполнить команду:


`df -h -x tmpfs`

И увидеть, что диск подмонтировался(результат редактирования fstab):

```
`Filesystem      Size  Used Avail Use% Mounted on
/dev/vda2        15G  4.4G  9.8G  31% /
/dev/vdb1        20G   24K   19G   1% /mnt/data`
```

Сделаем  пользователя postgres владельцем `/mnt/data`:


`sudo chown -R postgres:postgres /mnt/data/`

Перенос содержимого  `/var/lib/postgres/14` в `/mnt/data`:

`sudo mv /var/lib/postgresql/14 /mnt/data`

Попытка запустить кластер:

`sudo -u postgres pg_ctlcluster 14 main start`

Попытка неудачная, поскольку данные перенесены в `/mnt/data`, а postgresql об этом не знает:

```
`Error: /var/lib/postgresql/14/main is not accessible or does not exist
```

Редактирование файла `/etc/postgresql/14/main/postgresql.conf`:

`sudo nano p/etc/postgresql/14/main/postgresql.conf`

Найти строку:

```data_directory = '/var/lib/postgresql/14/main'          # use data in another directory```

и заменить на:

```data_directory = '/mnt/data/14/main'            # use data in another directory```


Попытка запустить кластер:

`sudo -u postgres pg_ctlcluster 14 main start`

Проверка состояния кластера:

`sudo -u postgres pg_lsclusters`

Кластер запущен:

```
Ver Cluster Port Status Owner    Data directory    Log file
14  main    5432 online postgres /mnt/data/14/main /var/log/postgresql/postgresql-14-main.log 
```

Подключение к кластеру:

`sudo -u postgres psql`

Запрос:

`postgres=# select * from test;`

Видно, что данные, которые были ранее сохранены - доступны.



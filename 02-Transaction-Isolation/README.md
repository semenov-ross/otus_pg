# SQL и реляционные СУБД. Введение в PostgreSQL 

В Google Cloud Platform создана ВМ pg14:
```console
gcloud compute instances create pg14 ...
```
Подключаемся к созданной pg14:
```console
gcloud compute ssh pg14
```
Добавляем репозитроий PostgreSQL:
```console
yum -y install https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
```
Установливаем PostgreSQL 14:
```console
yum -y install postgresql14 postgresql14-server
```
Инициализируем базу данных:
```console
/usr/pgsql-14/bin/postgresql-14-setup initdb
```
Запускаем сервис postgresql-14.service:
```console
systemctl start postgresql-14.service
systemctl enable postgresql-14.service
```
Создаем БД pg14, подключаемся к ней и создаем таблицу persons, наполняя её данными:
```console
postgres=# CREATE DATABASE pg14;
postgres=# \c pg14
pg14=# \set AUTOCOMMIT OFF
pg14=# create table persons(id serial, first_name text, second_name text);
CREATE TABLE
pg14=*# insert into persons(first_name, second_name) values('ivan', 'ivanov');
INSERT 0 1
pg14=*# insert into persons(first_name, second_name) values('petr', 'petrov');
INSERT 0 1
pg14=*# commit;
COMMIT

```
Смотрим текущий уровень изоляции:
```console
pg14=# show transaction isolation level;
 transaction_isolation 
-----------------------
 read committed
 ```

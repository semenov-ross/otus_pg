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
Начинаем новую транзакцию в обоих сессиях с дефолтным уровнем изоляции.  
В первой сессии добавляем новую запись:
```console
pg14=# begin;
BEGIN
pg14=*# insert into persons(first_name, second_name) values('sergey', 'sergeev');
INSERT 0 1
pg14=*# commit;
COMMIT
 ```
Во второй сесси выпоняем:
```console
pg14=*# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)
 ```
Вставку новой записи не видим, так как не завершена транзакция в первой сесии.

После завершения первой транзакции во второй сесси:
```console
pg14=*# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
 ```
Начинаем новые но уже repeatable read транзации:
```console
pg14=# BEGIN ISOLATION LEVEL REPEATABLE READ;
BEGIN
 ```
В первой сессии добавляем новую запись insert into persons(first_name, second_name) values('sveta', 'svetova');
```console
pg14=*# insert into persons(first_name, second_name) values('sveta', 'svetova');
INSERT 0 1
 ```
Во второй сессии:
```console
pg14=*# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
 ```
Новую запись не видим во воторой сесии, так как ISOLATION LEVEL REPEATABLE READ

Завершаем первую транзакцию:
```console
pg14=*# commit;
COMMIT
 ```
Во второй сессии:
```console
pg14=*# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
 ```
Новую запись не видим во воторой сесии, так как на уровне repeatable read снимок строится в начале транзакции — поэтому все запросы в одной транзакции видят одни и те же данные.

Завершаем вторую транзакцию и выполняем select * from persons:
```console
pg14=*# commit ;
COMMIT
pg14=# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sveta      | svetova
(4 rows)
 ```
Новую запись видим во воторой сесии, так как уровень изоляции read committed. Снимок данных строится в начале выполнения каждого оператора SQL

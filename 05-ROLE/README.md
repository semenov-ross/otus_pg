# Логический уровень PostgreSQL

### Работа с базами данных, пользователями и правами

Создаём новый кластер PostgresSQL 14 и подключаемся под пользователем postgres:
```console
[ross@otuspg ~]$ gcloud compute instances create pg14 ...
[ross@otuspg ~]$ gcloud compute ssh pg14
[ross@pg14 ~]$ sudo su -
[root@pg14 ~]# yum -y install https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
[root@pg14 ~]# yum -y install postgresql14-server
[root@pg14 ~]# /usr/pgsql-14/bin/postgresql-14-setup initdb
[root@pg14 ~]# systemctl enable --now postgresql-14.service

[root@pg14 ~]# su - postgres
-bash-4.2$ psql 
psql (14.0)
Type "help" for help.

postgres=#
```
Cоздаём новую базу данных testdb, подключаемся к ней под пользователем postgres, создаём новую схему testnm, 
создаём новую таблицу t1 с одной колонкой c1 типа integer и вставляем строку со значением c1=1:
```console
postgres=# CREATE DATABASE testdb;
CREATE DATABASE
postgres=# \c testdb 
You are now connected to database "testdb" as user "postgres".
testdb=# CREATE SCHEMA testnm;
CREATE SCHEMA
testdb=# CREATE TABLE t1 (c1 int);
CREATE TABLE
testdb=# INSERT INTO t1 values (1);
INSERT 0 1
```
Cоздаём новую роль readonly, даём право на подключение к базе данных testdb,
даём право на использование схемы testnm и даём право на select для всех таблиц схемы testnm:
```console
testdb=# CREATE ROLE readonly;
CREATE ROLE
testdb=# GRANT CONNECT ON DATABASE testdb TO readonly;
GRANT
testdb=# GRANT USAGE ON SCHEMA testnm TO readonly;
GRANT
testdb=# GRANT SELECT ON ALL TABLES IN SCHEMA testnm TO readonly;
GRANT
```
Создаём пользователя testread с паролем test123, назначаем роль readonly пользователю testread
подключаемся под пользователем testread к базе данных testdb
```console
testdb=# CREATE USER testread WITH PASSWORD 'test123';
CREATE ROLE
testdb=# GRANT readonly TO testread;
GRANT ROLE
testdb=# \c - testread
connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: FATAL:  Peer authentication failed for user "testread"
```
Ошибка подключения вызвана параметром peer, указанном в конфигурационным файле pg_hba.conf для локальных подключений.
Меняем способ авторизации и применяем конфигурацию:
```console
-bash-4.2$ vim 14/data/pg_hba.conf ->
local   all             all                                     trust

postgres=# select pg_reload_conf();
 pg_reload_conf 
----------------
 t
(1 row)
```
После этого пользователь testread сможет подключиться к базе testdb.
```console
testdb=# \c - testread 
You are now connected to database "testdb" as user "testread".
```
Делаем SELECT * FROM t1:
```console
testdb=> SELECT * FROM t1;
ERROR:  permission denied for table t1
```
Ошибка связана с тем, что таблица t1 находится в схеме public, а для роли readonly, которая присвоена пользователю testread выдавались права на чтение таблиц из схемы testnm.
```console
testdb=# \dt
        List of relations
 Schema | Name | Type  |  Owner   
--------+------+-------+----------
 public | t1   | table | postgres
(1 row)
```

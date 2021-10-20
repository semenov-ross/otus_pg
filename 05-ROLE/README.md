# Логический уровень PostgreSQL

### Работа с базами данных, пользователями и правами

Создаём новый кластер PostgresSQL 14 и подключаемся под пользователем postgres:
```console
[ross@otuspg ~]$ gcloud compute instances create pg14 ...
[ross@otuspg ~]$ gcloud compute ssh pg14
[ross@pg14 ~]$ sudo su -
[root@pg14 ~]# yum -y install https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
[root@pg14 ~]# yum -y install postgresql14 postgresql14-server
[root@pg14 ~]# /usr/pgsql-14/bin/postgresql-14-setup initdb
[root@pg14 ~]# systemctl enable --now postgresql-14.service

[root@pg14 ~]# su - postgres
Last login: Wed Oct 20 14:49:59 UTC 2021 on pts/0
-bash-4.2$ psql 
psql (14.0)
Type "help" for help.

postgres=#
```
Cоздаём новую базу данных testdb, подключаемся к ней под пользователем postgres, создаём новую схему testnm, 
создаём новую таблицу t1 с одной колонкой c1 типа integer, вставляем строку со значением c1=1:
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
даём право на использование схемы testnm, даём право на select для всех таблиц схемы testnm:
```console
testdb=# CREATE ROLE readonly;
CREATE ROLE
testdb=# GRANT CONNECT ON DATABASE testdb TO readonly;
GRANT
testdb=# GRANT USAGE ON SCHEMA testnm TO readonly;;
GRANT
testdb=# GRANT SELECT ON ALL TABLES IN SCHEMA testnm TO readonly;
GRANT
```

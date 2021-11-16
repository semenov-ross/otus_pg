# Виды и устройство репликации в PostgreSQL.
### настроить репликацию;

В Google Cloud Platform создаём 4 ВМ и устанавливаем PostgreSQL 14 :
```console
[ross@otuspg ~]$ gcloud compute instances list | awk '{print $1}'
NAME
pg14-repl1
pg14-repl2
pg14-repl3
pg14-repl4

[ross@otuspg ~]$ gcloud compute ssh pg14-repl1
[ross@pg14-repl1 ~]$ sudo su -
[root@pg14-repl1 ~]# yum -y install https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
[root@pg14-repl1 ~]# yum -y install postgresql14-server
[root@pg14-repl1 ~]# /usr/pgsql-14/bin/postgresql-14-setup initdb
[root@pg14-repl1 ~]# systemctl enable --now postgresql-14.service
...
[ross@otuspg ~]$ gcloud compute ssh pg14-repl4
[ross@pg14-repl4 ~]$ sudo su -
[root@pg14-repl4 ~]# yum -y install https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
[root@pg14-repl4 ~]# yum -y install postgresql14-server
[root@pg14-repl4 ~]# /usr/pgsql-14/bin/postgresql-14-setup initdb
[root@pg14-repl4 ~]# systemctl enable --now postgresql-14.service
```
Для pg14-repl1 изменяем уровень ведения журнала на logical для осуществления логической репликации:
```console
-bash-4.2$ psql 
psql (14.1)
Type "help" for help.

postgres=# ALTER SYSTEM SET wal_level = 'logical';
ALTER SYSTEM
postgres=# \q
[root@pg14-repl1 ~]# systemctl restart postgresql-14.service
postgres=# SHOW wal_level;
 wal_level 
-----------
 logical
(1 row)
```
На ВМ pg14-repl1 в тестовой БД создаём таблицы test для записи и test2 для запросов на чтение, добавляя в них тестовые данные:
```console
postgres=# \set PROMPT1 '[pg14-repl1] %/> '
[pg14-repl1] postgres> CREATE DATABASE otus;
[pg14-repl1] postgres> \c otus
[pg14-repl1] otus> CREATE TABLE test (k1 serial primary key);
CREATE TABLE
[pg14-repl1] otus> INSERT INTO test SELECT i FROM generate_series (1,10) s(i);
INSERT 0 10
[pg14-repl1] otus> CREATE TABLE test2 (k2 serial primary key);
CREATE TABLE
[pg14-repl1] otus> INSERT INTO test2 SELECT i FROM generate_series (1,10) s(i);
INSERT 0 10
```

```console
```

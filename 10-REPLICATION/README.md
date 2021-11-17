# Виды и устройство репликации в PostgreSQL.
### настроить репликацию;

В Google Cloud Platform создаём 4 ВМ, устанавливаем PostgreSQL 14 и добавляем разрешения для подключений к серверу PostgreSQL из локальной сети ВМ:
```console
[ross@otuspg ~]$ gcloud compute instances list
NAME        ZONE           MACHINE_TYPE  PREEMPTIBLE  INTERNAL_IP
pg14-repl1  us-central1-a  e2-medium                  10.128.0.19
pg14-repl2  us-central1-a  e2-medium                  10.128.0.21
pg14-repl3  us-central1-a  e2-medium                  10.128.0.22
pg14-repl4  us-central1-a  e2-medium                  10.128.0.23

[ross@otuspg ~]$ gcloud compute ssh pg14-repl1
[ross@pg14-repl1 ~]$ sudo su -
[root@pg14-repl1 ~]# yum -y install https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
[root@pg14-repl1 ~]# yum -y install postgresql14-server
[root@pg14-repl1 ~]# /usr/pgsql-14/bin/postgresql-14-setup initdb
[root@pg14-repl1 ~]# echo listen_addresses = \'10.128.0.19\' >> /var/lib/pgsql/14/data/postgresql.conf
[root@pg14-repl1 ~]# echo "host    all             all             10.128.0.0/24       scram-sha-256" >> 14/data/pg_hba.conf
[root@pg14-repl1 ~]# systemctl enable --now postgresql-14.service
...
[ross@otuspg ~]$ gcloud compute ssh pg14-repl4
[ross@pg14-repl4 ~]$ sudo su -
[root@pg14-repl4 ~]# yum -y install https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
[root@pg14-repl4 ~]# yum -y install postgresql14-server
[root@pg14-repl4 ~]# /usr/pgsql-14/bin/postgresql-14-setup initdb
[root@pg14-repl4 ~]# echo listen_addresses = \'10.128.0.23\' >> /var/lib/pgsql/14/data/postgresql.conf
[root@pg14-repl4 ~]# echo "host    all             all             10.128.0.0/24       scram-sha-256" >> 14/data/pg_hba.conf
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
На ВМ pg14-repl1 создаем публикацию таблицы test и подписываемся на публикацию таблицы test2 с ВМ pg14-repl2, создав на ВМ pg14-repl2 в БД otus таблицы test2 для записи, test для запросов на чтение
```console
[pg14-repl2] postgres> ALTER SYSTEM SET wal_level = 'logical';
ALTER SYSTEM
[pg14-repl2] postgres> \q
-bash-4.2$ exit
logout
[root@pg14-repl2 ~]# systemctl restart postgresql-14.service
[root@pg14-repl2 ~]# su - postgres
postgres=# \set PROMPT1 '[pg14-repl2] %/> '
[pg14-repl2] postgres> 
[pg14-repl2] postgres> CREATE DATABASE otus;
CREATE DATABASE
[pg14-repl2] otus> CREATE TABLE test (k1 serial primary key);
CREATE TABLE
[pg14-repl2] otus> CREATE TABLE test2 (k2 serial primary key);
CREATE TABLE
[pg14-repl2] otus> INSERT INTO test2 SELECT i FROM generate_series (1,10) s(i);
INSERT 0 10

postgres=# \set PROMPT1 '[pg14-repl1] %/> '
[pg14-repl1] postgres> \c otus
[pg14-repl1] otus> CREATE PUBLICATION pubication_test FOR TABLE test;
CREATE PUBLICATION
[pg14-repl1] otus> CREATE SUBSCRIPTION subscription_test2 CONNECTION 'host=pg14-repl2 user=postgres password=postgres dbname=otus' PUBLICATION publication_test2 WITH (copy_data = true);
NOTICE:  created replication slot "subscription_test2" on publisher
CREATE SUBSCRIPTION

[pg14-repl1] otus> \dRp+
                        Publication pubication_test
  Owner   | All tables | Inserts | Updates | Deletes | Truncates | Via root 
----------+------------+---------+---------+---------+-----------+----------
 postgres | f          | t       | t       | t       | t         | f
Tables:
    "public.test"

[pg14-repl1] otus> \dRs+
                                                                         List of subscriptions
        Name        |  Owner   | Enabled |     Publication     | Binary | Streaming | Synchronous commit |                          Conninfo                           
--------------------+----------+---------+---------------------+--------+-----------+--------------------+-------------------------------------------------------------
 subscription_test2 | postgres | t       | {publication_test2} | f      | f         | off                | host=pg14-repl2 user=postgres password=postgres dbname=otus
(1 row)
```

```console
```

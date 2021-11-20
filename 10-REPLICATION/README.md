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
Initializing database ... OK
[root@pg14-repl1 ~]# echo listen_addresses = \'0.0.0.0\' >> /var/lib/pgsql/14/data/postgresql.conf
[root@pg14-repl1 ~]# echo "host    all             all             10.128.0.0/24       scram-sha-256" >> /var/lib/pgsql/14/data/pg_hba.conf
[root@pg14-repl1 ~]# systemctl enable --now postgresql-14.service
...
[ross@otuspg ~]$ gcloud compute ssh pg14-repl4
[ross@pg14-repl3 ~]$ sudo su -
[root@pg14-repl3 ~]# yum -y install https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
[root@pg14-repl3 ~]# yum -y install postgresql14-server
[root@pg14-repl3 ~]# /usr/pgsql-14/bin/postgresql-14-setup initdb
Initializing database ... OK
[root@pg14-repl3 ~]# echo listen_addresses = \'0.0.0.0\' >> /var/lib/pgsql/14/data/postgresql.conf
[root@pg14-repl3 ~]# echo "host    all             all             10.128.0.0/24       scram-sha-256" >> /var/lib/pgsql/14/data/pg_hba.conf
[root@pg14-repl3 ~]# systemctl enable --now postgresql-14.service
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
На ВМ pg14-repl1 в тестовой БД создаём таблицы test для записи и test2 для запросов на чтение, добавив в test тестовые данные:
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

[pg14-repl1] otus> \dt+
                                     List of relations
 Schema | Name  | Type  |  Owner   | Persistence | Access method |    Size    | Description 
--------+-------+-------+----------+-------------+---------------+------------+-------------
 public | test  | table | postgres | permanent   | heap          | 8192 bytes | 
 public | test2 | table | postgres | permanent   | heap          | 0 bytes    |
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
На ВМ pg14-repl2 создаем публикацию таблицы test2 и подписываемся на публикацию таблицы test с ВМ pg14-repl1.
```console
[pg14-repl2] otus> CREATE PUBLICATION publication_test2 FOR TABLE test2;
CREATE PUBLICATION
[pg14-repl2] otus> \dRp+
                       Publication publication_test2
  Owner   | All tables | Inserts | Updates | Deletes | Truncates | Via root 
----------+------------+---------+---------+---------+-----------+----------
 postgres | f          | t       | t       | t       | t         | f
Tables:
    "public.test2"
[pg14-repl2] otus> CREATE SUBSCRIPTION subscription_test CONNECTION 'host=pg14-repl1 user=postgres password=postgres dbname=otus' PUBLICATION publication_test WITH (copy_data = true);
NOTICE:  created replication slot "subscription_test" on publisher
CREATE SUBSCRIPTION
[pg14-repl2] otus> \dRs+
                                                                        List of subscriptions
       Name        |  Owner   | Enabled |    Publication     | Binary | Streaming | Synchronous commit |                          Conninfo                           
-------------------+----------+---------+--------------------+--------+-----------+--------------------+-------------------------------------------------------------
 subscription_test | postgres | t       | {publication_test} | f      | f         | off                | host=pg14-repl1 user=postgres password=postgres dbname=otus
(1 row)
```
ВМ pg14-repl3 используем как реплику для чтения и бэкапов, подписавшись на публикуемые таблицы из ВМ pg14-repl1 и pg14-repl2 )
```console
[pg14-repl3] postgres> ALTER SYSTEM SET wal_level = 'logical';
ALTER SYSTEM
[root@pg14-repl3 ~]# systemctl restart postgresql-14.service

[pg14-repl3] postgres> CREATE DATABASE otus;
CREATE DATABASE
[pg14-repl3] postgres> \c otus 
You are now connected to database "otus" as user "postgres".
[pg14-repl3] otus> CREATE TABLE test (k1 serial primary key);
CREATE TABLE
[pg14-repl3] otus> CREATE TABLE test2 (k2 serial primary key);
CREATE TABLE

[pg14-repl3] otus> CREATE SUBSCRIPTION subscription_test_repl3 CONNECTION 'host=pg14-repl1 user=postgres password=postgres dbname=otus' PUBLICATION publication_test WITH (copy_data = true);
NOTICE:  created replication slot "subscription_test_repl3" on publisher
CREATE SUBSCRIPTION
[pg14-repl3] otus> CREATE SUBSCRIPTION subscription_test2_repl3 CONNECTION 'host=pg14-repl2 user=postgres password=postgres dbname=otus' PUBLICATION publication_test2 WITH (copy_data = true);
NOTICE:  created replication slot "subscription_test2_repl3" on publisher
CREATE SUBSCRIPTION

[pg14-repl3] otus> \dRs+
                                                                            List of subscriptions
           Name           |  Owner   | Enabled |     Publication     | Binary | Streaming | Synchronous commit |                          Conninfo                           
--------------------------+----------+---------+---------------------+--------+-----------+--------------------+-------------------------------------------------------------
 subscription_test2_repl3 | postgres | t       | {publication_test2} | f      | f         | off                | host=pg14-repl2 user=postgres password=postgres dbname=otus
 subscription_test_repl3  | postgres | t       | {publication_test}  | f      | f         | off                | host=pg14-repl1 user=postgres password=postgres dbname=otus
(2 rows)
```
Смотрим статус репликации на pg14-repl1 и pg14-repl2:
```console
[pg14-repl1] otus> SELECT * FROM pg_stat_replication\gx
-[ RECORD 1 ]----+------------------------------
pid              | 1351
usesysid         | 10
usename          | postgres
application_name | subscription_test
client_addr      | 10.128.0.21
client_hostname  | 
client_port      | 44226
backend_start    | 2021-11-20 04:24:47.622806+00
backend_xmin     | 
state            | streaming
sent_lsn         | 0/1797F98
write_lsn        | 0/1797F98
flush_lsn        | 0/1797F98
replay_lsn       | 0/1797F98
write_lag        | 
flush_lag        | 
replay_lag       | 
sync_priority    | 0
sync_state       | async
reply_time       | 2021-11-20 04:48:09.119132+00
-[ RECORD 2 ]----+------------------------------
pid              | 1445
usesysid         | 10
usename          | postgres
application_name | subscription_test_repl3
client_addr      | 10.128.0.22
client_hostname  | 
client_port      | 34162
backend_start    | 2021-11-20 04:43:48.821449+00
backend_xmin     | 
state            | streaming
sent_lsn         | 0/1797F98
write_lsn        | 0/1797F98
flush_lsn        | 0/1797F98
replay_lsn       | 0/1797F98
write_lag        | 
flush_lag        | 
replay_lag       | 
sync_priority    | 0
sync_state       | async
reply_time       | 2021-11-20 04:48:09.123955+00

[pg14-repl2] otus> SELECT * FROM pg_stat_replication\gx
-[ RECORD 1 ]----+------------------------------
pid              | 1226
usesysid         | 10
usename          | postgres
application_name | subscription_test2
client_addr      | 10.128.0.19
client_hostname  | 
client_port      | 56562
backend_start    | 2021-11-20 04:06:37.775478+00
backend_xmin     | 
state            | streaming
sent_lsn         | 0/1779EC8
write_lsn        | 0/1779EC8
flush_lsn        | 0/1779EC8
replay_lsn       | 0/1779EC8
write_lag        | 
flush_lag        | 
replay_lag       | 
sync_priority    | 0
sync_state       | async
reply_time       | 2021-11-20 04:48:31.752371+00
-[ RECORD 2 ]----+------------------------------
pid              | 1447
usesysid         | 10
usename          | postgres
application_name | subscription_test2_repl3
client_addr      | 10.128.0.22
client_hostname  | 
client_port      | 49138
backend_start    | 2021-11-20 04:44:11.412867+00
backend_xmin     | 
state            | streaming
sent_lsn         | 0/1779EC8
write_lsn        | 0/1779EC8
flush_lsn        | 0/1779EC8
replay_lsn       | 0/1779EC8
write_lag        | 
flush_lag        | 
replay_lag       | 
sync_priority    | 0
sync_state       | async
reply_time       | 2021-11-20 04:48:31.846925+00
```

Для реализации высокой доступности на pg14-repl4 настроим физическую репликацию с pg14-repl3 при помощи штатной утилиты pg_basebackup, добавив разрешающее правило в pg_hba.conf на pg14-repl3:
```console
[pg14-repl3] otus> \! echo "host replication all 10.128.0.0/24 scram-sha-256" >> /var/lib/pgsql/14/data/pg_hba.conf
[pg14-repl3] otus> select pg_reload_conf();
 pg_reload_conf 
----------------
 t
(1 row)

[pg14-repl3] otus> select * from pg_hba_file_rules ;
 line_number | type  |   database    | user_name |  address   |                 netmask                 |  auth_method  | options | error
-------------+-------+---------------+-----------+------------+-----------------------------------------+---------------+---------+-------
          85 | local | {all}         | {all}     |            |                                         | peer          |         |
          87 | host  | {all}         | {all}     | 127.0.0.1  | 255.255.255.255                         | scram-sha-256 |         |
          89 | host  | {all}         | {all}     | ::1        | ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff | scram-sha-256 |         |
          92 | local | {replication} | {all}     |            |                                         | peer          |         |
          93 | host  | {replication} | {all}     | 127.0.0.1  | 255.255.255.255                         | scram-sha-256 |         |
          94 | host  | {replication} | {all}     | ::1        | ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff | scram-sha-256 |         |
          95 | host  | {all}         | {all}     | 10.128.0.0 | 255.255.255.0                           | scram-sha-256 |         |
          96 | host  | {replication} | {all}     | 10.128.0.0 | 255.255.255.0                           | scram-sha-256 |         |
(8 rows)

-bash-4.2$ /usr/pgsql-14/bin/pg_basebackup -h pg14-repl3 -U postgres -D /var/lib/pgsql/14/data/ -W -v -R -P
Password: 
pg_basebackup: initiating base backup, waiting for checkpoint to complete
pg_basebackup: checkpoint completed
pg_basebackup: write-ahead log start point: 0/2000028 on timeline 1
pg_basebackup: starting background WAL receiver
pg_basebackup: created temporary replication slot "pg_basebackup_1598"
35884/35884 kB (100%), 1/1 tablespace
pg_basebackup: write-ahead log end point: 0/2000100
pg_basebackup: waiting for background process to finish streaming ...
pg_basebackup: syncing data to disk ...
pg_basebackup: renaming backup_manifest.tmp to backup_manifest
pg_basebackup: base backup completed

[root@pg14-repl4 14]# systemctl restart postgresql-14.service

[pg14-repl4] postgres> \l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges   
-----------+----------+----------+-------------+-------------+-----------------------
 otus      | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
(4 rows)

[pg14-repl4] postgres> \c otus 
You are now connected to database "otus" as user "postgres".
[pg14-repl4] otus> \dt+
                                     List of relations
 Schema | Name  | Type  |  Owner   | Persistence | Access method |    Size    | Description 
--------+-------+-------+----------+-------------+---------------+------------+-------------
 public | test  | table | postgres | permanent   | heap          | 0 bytes    | 
 public | test2 | table | postgres | permanent   | heap          | 8192 bytes | 
(2 rows)
```
Проверяем статус репликации на pg14-repl3 и делаем вставку строки на pg14-repl1:
```console
[pg14-repl3] otus> SELECT * FROM pg_stat_replication\gx
-[ RECORD 1 ]----+------------------------------
pid              | 17463
usesysid         | 10
usename          | postgres
application_name | walreceiver
client_addr      | 10.128.0.23
client_hostname  | 
client_port      | 48908
backend_start    | 2021-11-20 10:16:13.559909+00
backend_xmin     | 
state            | streaming
sent_lsn         | 0/7000060
write_lsn        | 0/7000060
flush_lsn        | 0/7000060
replay_lsn       | 0/7000060
write_lag        | 
flush_lag        | 
replay_lag       | 
sync_priority    | 0
sync_state       | async
reply_time       | 2021-11-20 10:18:13.784496+00

[pg14-repl1] otus> INSERT INTO test SELECT i FROM generate_series (11,20) s(i);
INSERT 0 10
[pg14-repl1] otus> INSERT INTO test2 SELECT i FROM generate_series (11,20) s(i);
INSERT 0 10


```

# Блокировки
### Механизм блокировок

Создаём новый кластер PostgresSQL 14, подключаемся и настраиваем так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд:
```console
[ross@otuspg ~]$ gcloud compute instances create pg14lock ...
[ross@otuspg ~]$ gcloud compute ssh pg14lock
[ross@pg14lock ~]$ sudo su -
[root@pg14lock ~]# yum -y install https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
[root@pg14lock ~]# yum -y install postgresql14 postgresql14-server
[root@pg14lock ~]# /usr/pgsql-14/bin/postgresql-14-setup initdb
Initializing database ... OK
[root@pg14lock ~]# systemctl enable --now postgresql-14.service
[root@pg14lock ~]# su - postgres
-bash-4.2$ psql 
psql (14.0)
Type "help" for help.
postgres=# CREATE DATABASE otus;
CREATE DATABASE
postgres=# \c otus ;
You are now connected to database "otus" as user "postgres".
otus=# ALTER SYSTEM SET log_lock_waits='on';
ALTER SYSTEM
otus=# ALTER SYSTEM SET deadlock_timeout=200;
ALTER SYSTEM
otus=# SELECT pg_reload_conf();
 pg_reload_conf 
----------------
 t
(1 row)
```
Для появления сообщений в журнале о блокировках, создадим тестовую таблицу и добавим несколько строк:
```console
otus=# CREATE TABLE lock_tbl(i int);
CREATE TABLE
otus=# INSERT INTO lock_tbl VALUES (1),(2);
INSERT 0 2
```
Код серверного процесса, обслуживающего первую сессию:
```console
otus=# SELECT pg_backend_pid();
 pg_backend_pid 
----------------
           1692
(1 row)
```
Код серверного процесса, обслуживающего вторую сессию:
```console
otus=# SELECT pg_backend_pid();
 pg_backend_pid 
----------------
           1962
(1 row)
```
Попытаемся изменить одну и туже строку из разных сессиях, начинаем транзакцию в первой сесси:
```console
otus=# BEGIN;
BEGIN
otus=*# UPDATE lock_tbl SET i=3 WHERE i=2;
UPDATE 1
```
Во второй сессии:
```console
otus=# BEGIN;
BEGIN
otus=*# UPDATE lock_tbl SET i=3 WHERE i=2;
```
В журнале видим:
```console
2021-11-07 11:36:57.900 UTC [1962] LOG:  process 1962 still waiting for ShareLock on transaction 737 after 200.398 ms
2021-11-07 11:36:57.900 UTC [1962] DETAIL:  Process holding the lock: 1692. Wait queue: 1962.
2021-11-07 11:36:57.900 UTC [1962] CONTEXT:  while updating tuple (0,2) in relation "lock_tbl"
2021-11-07 11:36:57.900 UTC [1962] STATEMENT:  UPDATE lock_tbl SET i=3 WHERE i=2;
```

Смоделируем ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах, запоминая код серверного процесса и идентификатор текущей транзакции:
```console
otus=# \set PROMPT1 '[sess1] %/> '
[sess1] otus> BEGIN;
BEGIN
[sess1] otus> SELECT pg_backend_pid(), txid_current();
 pg_backend_pid | txid_current 
----------------+--------------
           2595 |          766
(1 row)
[sess1] otus> UPDATE lock_tbl SET i=4 WHERE i=1;
UPDATE 1

otus=# \set PROMPT1 '[sess2] %/> '
[sess2] otus> BEGIN;
BEGIN
[sess2] otus> SELECT pg_backend_pid(), txid_current();
 pg_backend_pid | txid_current 
----------------+--------------
           2600 |          767
(1 row)
[sess2] otus> UPDATE lock_tbl SET i=5 WHERE i=1;

otus=# \set PROMPT1 '[sess3] %/> '
[sess3] otus> BEGIN;
BEGIN
[sess3] otus> SELECT pg_backend_pid(), txid_current();
 pg_backend_pid | txid_current 
----------------+--------------
           2604 |          768
(1 row)
[sess3] otus> UPDATE lock_tbl SET i=6 WHERE i=1;
```
Вторая и третья сессии ожидают окончания транзакции в первой сесси, так как она блокирует изменяемую строку.
В журнале видим:
```console
2021-11-07 13:02:23.675 UTC [2600] LOG:  process 2600 still waiting for ShareLock on transaction 766 after 200.170 ms
2021-11-07 13:02:23.675 UTC [2600] DETAIL:  Process holding the lock: 2595. Wait queue: 2600.
2021-11-07 13:02:23.675 UTC [2600] CONTEXT:  while updating tuple (0,1) in relation "lock_tbl"
2021-11-07 13:02:23.675 UTC [2600] STATEMENT:  UPDATE lock_tbl SET i=5 WHERE i=1;
2021-11-07 13:02:34.993 UTC [2604] LOG:  process 2604 still waiting for ExclusiveLock on tuple (0,1) of relation 16388 of database 16384 after 200.184 ms
2021-11-07 13:02:34.993 UTC [2604] DETAIL:  Process holding the lock: 2600. Wait queue: 2604.
2021-11-07 13:02:34.993 UTC [2604] STATEMENT:  UPDATE lock_tbl SET i=6 WHERE i=1;
```
В представлении pg_locks смотрим информацию о блокировках:
```console
postgres=# SELECT pid,transactionid,virtualxid,locktype,relation::regclass,mode, granted FROM pg_locks WHERE pid IN (2595,2600,2604) ORDER BY 1; 
pid  | transactionid | virtualxid |   locktype    | relation |       mode       | granted 
------+---------------+------------+---------------+----------+------------------+---------
 2595 |           766 |            | transactionid |          | ExclusiveLock    | t
 2595 |               |            | relation      | lock_tbl | RowExclusiveLock | t
 2595 |               | 5/37       | virtualxid    |          | ExclusiveLock    | t
 2600 |               |            | relation      | lock_tbl | RowExclusiveLock | t
 2600 |               | 6/15       | virtualxid    |          | ExclusiveLock    | t
 2600 |           767 |            | transactionid |          | ExclusiveLock    | t
 2600 |               |            | tuple         | lock_tbl | ExclusiveLock    | t
 2600 |           766 |            | transactionid |          | ShareLock        | f
 2604 |           768 |            | transactionid |          | ExclusiveLock    | t
 2604 |               | 7/105      | virtualxid    |          | ExclusiveLock    | t
 2604 |               |            | relation      | lock_tbl | RowExclusiveLock | t
 2604 |               |            | tuple         | lock_tbl | ExclusiveLock    | f
(12 rows)
```
Каждая из сессий установила исключительную блокировку ExclusiveLock на виртуальный идентификатор транзакции(virtualxid) и на номера своих транзакций(transactionid).  
Каждая из сессий установила блокировку RowExclusiveLock на таблицу lock_tbl.  
Вторая сессия ожидает от первой блокировку transactionid в режиме ShareLock и удерживает блокировку tuple в режиме ExclusiveLock.  
Третья сессия ожидает блокировку tuple в режиме ExclusiveLock.  

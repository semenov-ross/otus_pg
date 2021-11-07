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

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

# Журналы
### уметь работать с журналами и контрольными точками; уметь настраивать параметры журналов

Создаём новый кластер PostgresSQL 14 и подключаемся и выставляем параметр выполнения контрольной точки раз в 30 секунд:
```console
[ross@otuspg ~]$ gcloud compute instances create pg14wal ...
[ross@otuspg ~]$ gcloud compute ssh pg14wal
[ross@pg14wal ~]$ sudo su -
[root@pg14wal ~]# yum -y install https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
[root@pg14wal ~]# yum -y install postgresql14-server
[root@pg14wal ~]# /usr/pgsql-14/bin/postgresql-14-setup initdb
[root@pg14wal ~]# systemctl enable --now postgresql-14.service
[root@pg14wal ~]# su - postgres
-bash-4.2$ psql
psql (14.0)
Type "help" for help.
postgres=# ALTER SYSTEM SET checkpoint_timeout = '30s';
ALTER SYSTEM
postgres=# SELECT pg_reload_conf();
 pg_reload_conf 
----------------
 t
(1 row)
postgres=# SHOW checkpoint_timeout ;
 checkpoint_timeout 
--------------------
 30s
(1 row)
```
Фиксируем текущий lsn и подаём нагрузку c помощью утилиты pgbench в течение 10 минут:
```console
postgres=# SELECT pg_current_wal_lsn();
 pg_current_wal_lsn 
--------------------
 0/1748EE8
(1 row)

-bash-4.2$ /usr/pgsql-14/bin/pgbench -i postgres
pgbench (14.0)
...
done in 0.44 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 0.25 s, vacuum 0.09 s, primary keys 0.08 s).

-bash-4.2$ /usr/pgsql-14/bin/pgbench -c8 -P 10 -T 600 -U postgres postgres
...
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 600 s
number of transactions actually processed: 373241
latency average = 12.848 ms
latency stddev = 17.474 ms
initial connection time = 28.526 ms
tps = 622.029256 (without initial connection time)
```
Измеряем, какой объем журнальных файлов был сгенерирован за это время и оцениваем, какой объем приходится в среднем на одну контрольную точку:
```console
postgres=# select pg_current_wal_lsn();
 pg_current_wal_lsn 
--------------------
 0/1F65A500
(1 row)
postgres=# select pg_size_pretty('0/1F65A500'::pg_lsn - '0/1748EE8'::pg_lsn);
 pg_size_pretty 
----------------
 479 MB
(1 row)
postgres=# select pg_size_pretty(('0/1F65A500'::pg_lsn - '0/1748EE8'::pg_lsn)/20);
 pg_size_pretty 
----------------
 24 MB
(1 row)
```
Проверяем, все ли контрольные точки выполнялись точно по расписанию:
```console
postgres=# select * from pg_stat_bgwriter\gx
-[ RECORD 1 ]---------+------------------------------
checkpoints_timed     | 31
checkpoints_req       | 0
checkpoint_write_time | 538918
checkpoint_sync_time  | 122
buffers_checkpoint    | 42262
buffers_clean         | 0
maxwritten_clean      | 0
buffers_backend       | 4220
buffers_backend_fsync | 0
buffers_alloc         | 4760
stats_reset           | 2021-10-31 13:18:40.333897+00
```
Все контрольные точки выполнились по расписанию, так как в статистике checkpoints_timed = 31, а checkpoints_req = 0 ввиду того, что max_wal_size='1GB'
больше объёма сгенерированного на одну контрольную точку(24 MB)

Запускаем утилиту pgbench в асинхронном режиме:

```console
postgres=# ALTER SYSTEM SET synchronous_commit = off;
ALTER SYSTEM
postgres=# SELECT pg_reload_conf();
 pg_reload_conf 
----------------
 t
(1 row)
-bash-4.2$ /usr/pgsql-14/bin/pgbench -c8 -P 10 -T 600 -U postgres postgres
pgbench (14.0)
...
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 600 s
number of transactions actually processed: 762380
latency average = 6.275 ms
latency stddev = 17.833 ms
initial connection time = 27.260 ms
tps = 1270.477621 (without initial connection time)
```
В асинхронном режиме количество tps больше чем вдвое за счет отсутствия ожидания локального сброса WAL на диск,
но может образоваться окно от момента, когда клиент узнаёт об успешном завершении, до момента, когда транзакция действительно гарантированно защищена от сбоя.

Создаём новый кластер с включенной контрольной суммой страниц, создаём таблицу, вставляем несколько значений и выключаем кластер:
```console
[root@pg14wal ~]# systemctl stop postgresql-14.service
[root@pg14wal ~]# rm -rf /var/lib/pgsql/14/data/
[root@pg14wal ~]# su - postgres
-bash-4.2$ /usr/pgsql-14/bin/initdb -D /var/lib/pgsql/14/data --data-checksums
[root@pg14wal ~]# systemctl start postgresql-14.service
postgres=# create table test(col1 int,col2 int);
CREATE TABLE
postgres=# INSERT INTO test SELECT i, i+2 FROM generate_series (1,100) s(i);
INSERT 0 100
postgres=# select pg_relation_filepath('test');
 pg_relation_filepath 
----------------------
 base/14486/16384
(1 row)
[root@pg14wal ~]# systemctl stop postgresql-14.service
```
Изменяем пару байт в таблице test, включаем кластер и делаем выборку из таблицы.
```console
[root@pg14wal 14486]# vi /var/lib/pgsql/14/data/base/14486/16384
[root@pg14wal 14486]# systemctl start postgresql-14.service
postgres=# select * from test;
WARNING:  page verification failed, calculated checksum 25198 but expected 18599
ERROR:  invalid page in block 0 of relation base/14486/16384
```
Получили ошибку проверки контрольной суммы страницы. Чтобы игнорировать ошибку включаем параметр ignore_checksum_failure:
```console
postgres=# SET ignore_checksum_failure = on;
SET
postgres=# select * from test;
WARNING:  page verification failed, calculated checksum 25198 but expected 18599
 col1 |  col2   
------+---------
    1 | 6381921
    2 |       4
    3 |       5
...
```
Неповрежденные данные прочитать удалось, так как цел заголовок блока.

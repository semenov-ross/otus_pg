# Настройка PostgreSQL
### поработать с параметрами конфигурации PostgreSQL;объяснить выбор оптимального значения для параметров.

Создаём новый кластер PostgresSQL 14, используя в GCE инстансе standard disk, для более яркой демонстрации влияние конфигурационных параметров сервера PostgresSQL на производительность:
```console
[ross@otuspg ~]$ gcloud compute instances create pg14bench ... diskTypes/pd-standard ...
[ross@otuspg ~]$ gcloud compute ssh pg14bench
[ross@pg14lock ~]$ sudo su -
[root@pg14lock ~]# yum -y install https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
[root@pg14lock ~]# yum -y install postgresql14 postgresql14-server
[root@pg14lock ~]# /usr/pgsql-14/bin/postgresql-14-setup initdb
Initializing database ... OK
[root@pg14lock ~]# systemctl enable --now postgresql-14.service
```
Для нагрузочного тестирования будем использовать утилиту sysbench Release 1.0.20 (https://github.com/akopytov/sysbench):
```console
[root@pg14bench ~]# curl -s https://packagecloud.io/install/repositories/akopytov/sysbench/script.rpm.sh | sudo bash
[root@pg14bench ~]# yum -y install sysbench
```
Создадим отдельную БД sysbench и пользователя ubench для проведения тестов:
```console
CREATE DATABASE sysbench;
CREATE USER ubench WITH PASSWORD 'password' INHERIT;
ALTER DATABASE sysbench OWNER TO ubench;
```
Заполним БД sysbench тестовыми данными:
```console
-bash-4.2$ sysbench --pgsql-host=127.0.0.1 --pgsql-port=5432 --pgsql-db=sysbench --db-driver=pgsql --pgsql-user=ubench --pgsql-password=password --table_size=1000000 --tables=10 /usr/share/sysbench/oltp_read_write.lua prepare

-bash-4.2$ psql
psql (14.1)
Type "help" for help.

postgres=# \c sysbench
You are now connected to database "sysbench" as user "postgres".
sysbench-# \dt+
                                    List of relations
 Schema |   Name   | Type  | Owner  | Persistence | Access method |  Size  | Description
--------+----------+-------+--------+-------------+---------------+--------+-------------
 public | sbtest1  | table | ubench | permanent   | heap          | 211 MB |
 public | sbtest10 | table | ubench | permanent   | heap          | 211 MB |
 public | sbtest2  | table | ubench | permanent   | heap          | 211 MB |
 public | sbtest3  | table | ubench | permanent   | heap          | 211 MB |
 public | sbtest4  | table | ubench | permanent   | heap          | 211 MB |
 public | sbtest5  | table | ubench | permanent   | heap          | 211 MB |
 public | sbtest6  | table | ubench | permanent   | heap          | 211 MB |
 public | sbtest7  | table | ubench | permanent   | heap          | 211 MB |
 public | sbtest8  | table | ubench | permanent   | heap          | 211 MB |
 public | sbtest9  | table | ubench | permanent   | heap          | 211 MB |
(10 rows)
```
Запустим тест oltp_read_write на 10 минут, используя 10 потоков:
```console
-bash-4.2$ sysbench --db-driver=pgsql --pgsql-db=sysbench --pgsql-user=ubench --pgsql-password=password --report-interval=30 --tables=10 --table_size=1000000 --threads=10 --time=600 oltp_read_write run
sysbench 1.0.20 (using bundled LuaJIT 2.1.0-beta2)


Running the test with following options:
Number of threads: 10
Report intermediate results every 30 second(s)
Initializing random number generator from current time


Initializing worker threads...

Threads started!

[ 30s ] thds: 10 tps: 68.29 qps: 1371.76 (r/w/o: 960.59/274.24/136.92) lat (ms,95%): 383.33 err/s: 0.00 reconn/s: 0.00
[ 60s ] thds: 10 tps: 109.40 qps: 2187.47 (r/w/o: 1531.32/437.34/218.80) lat (ms,95%): 376.49 err/s: 0.00 reconn/s: 0.00
...
[ 600s ] thds: 10 tps: 91.63 qps: 1832.67 (r/w/o: 1282.87/366.53/183.27) lat (ms,95%): 549.52 err/s: 0.00 reconn/s: 0.00
SQL statistics:
    queries performed:
        read:                            784098
        write:                           224024
        other:                           112016
        total:                           1120138
    transactions:                        56006  (93.32 per sec.)
    queries:                             1120138 (1866.45 per sec.)
    ignored errors:                      1      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          600.1410s
    total number of events:              56006

Latency (ms):
         min:                                    4.00
         avg:                                  107.14
         max:                                 1891.36
         95th percentile:                      530.08
         sum:                              6000722.82

Threads fairness:
    events (avg/stddev):           5600.6000/20.03
    execution time (avg/stddev):   600.0723/0.01
```
Для реализации задачи получения максимальной производительности, не обращая внимние на надёжность, отключаем параметры, отвечающие за синхронизацию данных на файловой системе(synchronous_commit,fsync,full_page_writes).  
Увеличиваем значение параметров max_wal_size и checkpoint_timeout для исключения влияние служебных процессов на тестирование.  
Увеличиваем значение shared_buffers до 40% от ОЗУ и work_mem для использования макисимально возможного объёма ОЗУ.  
Ограничиваем количество подключений до минимального. Так как max_connections включает в себя superuser_reserved_connections(по умолчанию 3), устанавливаем max_connections=13.  
Также ограничим количество служебных процессов(max_worker_processes, max_parallel_workers, max_parallel_maintenance_workers, max_parallel_workers_per_gather).  
Допишим новые значения параметров в конце файла /var/lib/pgsql/14/data/postgresql.conf  
После перечитывания конфигурации видим, что есть параметры, требующие перезапуска сервера:
```console
postgres=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)

postgres=# select name,setting,pending_restart,context,short_desc from pg_settings where name in ('max_connections','shared_buffers','maintenance_work_mem','work_mem','synchronous_commit','fsync','full_page_writes','checkpoint_timeout','max_wal_size','max_worker_processes','max_parallel_workers','max_parallel_maintenance_workers','max_parallel_workers_per_gather');
               name               | setting | pending_restart |  context   |                                 short_desc
----------------------------------+---------+-----------------+------------+-----------------------------------------------------------------------------
 checkpoint_timeout               | 86400   | f               | sighup     | Sets the maximum time between automatic WAL checkpoints.
 fsync                            | off     | f               | sighup     | Forces synchronization of updates to disk.
 full_page_writes                 | off     | f               | sighup     | Writes full pages to WAL when first modified after a checkpoint.
 maintenance_work_mem             | 204800  | f               | user       | Sets the maximum memory to be used for maintenance operations.
 max_connections                  | 100     | t               | postmaster | Sets the maximum number of concurrent connections.
 max_parallel_maintenance_workers | 1       | f               | user       | Sets the maximum number of parallel processes per maintenance operation.
 max_parallel_workers             | 2       | f               | user       | Sets the maximum number of parallel workers that can be active at one time.
 max_parallel_workers_per_gather  | 1       | f               | user       | Sets the maximum number of parallel processes per executor node.
 max_wal_size                     | 10240   | f               | sighup     | Sets the WAL size that triggers a checkpoint.
 max_worker_processes             | 8       | t               | postmaster | Maximum number of concurrent worker processes.
 shared_buffers                   | 16384   | t               | postmaster | Sets the number of shared memory buffers used by the server.
 synchronous_commit               | off     | f               | user       | Sets the current transaction's synchronization level.
 work_mem                         | 204800  | f               | user       | Sets the maximum memory to be used for query workspaces.
(13 rows)
```
Перезапускаем и проверяем:
```console
[root@pg14bench ~]# systemctl restart postgresql-14.service
[root@pg14bench ~]# su - postgres
Last login: Sun Nov 14 10:22:42 UTC 2021 on pts/0
-bash-4.2$ psql
psql (14.1)
Type "help" for help.

postgres=# select name,setting,pending_restart,context,short_desc from pg_settings where name in ('max_connections','shared_buffers','maintenance_work_mem','work_mem','synchronous_commit','fsync','full_page_writes','checkpoint_timeout','max_wal_size','max_worker_processes','max_parallel_workers','max_parallel_maintenance_workers','max_parallel_workers_per_gather');
               name               | setting | pending_restart |  context   |                                 short_desc
----------------------------------+---------+-----------------+------------+-----------------------------------------------------------------------------
 checkpoint_timeout               | 86400   | f               | sighup     | Sets the maximum time between automatic WAL checkpoints.
 fsync                            | off     | f               | sighup     | Forces synchronization of updates to disk.
 full_page_writes                 | off     | f               | sighup     | Writes full pages to WAL when first modified after a checkpoint.
 maintenance_work_mem             | 204800  | f               | user       | Sets the maximum memory to be used for maintenance operations.
 max_connections                  | 13      | f               | postmaster | Sets the maximum number of concurrent connections.
 max_parallel_maintenance_workers | 1       | f               | user       | Sets the maximum number of parallel processes per maintenance operation.
 max_parallel_workers             | 2       | f               | user       | Sets the maximum number of parallel workers that can be active at one time.
 max_parallel_workers_per_gather  | 1       | f               | user       | Sets the maximum number of parallel processes per executor node.
 max_wal_size                     | 10240   | f               | sighup     | Sets the WAL size that triggers a checkpoint.
 max_worker_processes             | 2       | f               | postmaster | Maximum number of concurrent worker processes.
 shared_buffers                   | 192000  | f               | postmaster | Sets the number of shared memory buffers used by the server.
 synchronous_commit               | off     | f               | user       | Sets the current transaction's synchronization level.
 work_mem                         | 204800  | f               | user       | Sets the maximum memory to be used for query workspaces.
(13 rows)

```
Запускаем тест oltp_read_write повторно:
```console
-bash-4.2$ sysbench --db-driver=pgsql --pgsql-db=sysbench --pgsql-user=ubench --pgsql-password=password --report-interval=30 --tables=10 --table_size=1000000 --threads=10 --time=600 oltp_read_write run
sysbench 1.0.20 (using bundled LuaJIT 2.1.0-beta2)

Running the test with following options:
Number of threads: 10
Report intermediate results every 30 second(s)
Initializing random number generator from current time


Initializing worker threads...

Threads started!

[ 30s ] thds: 10 tps: 496.43 qps: 9932.14 (r/w/o: 6952.86/1986.02/993.26) lat (ms,95%): 26.20 err/s: 0.03 reconn/s: 0.00
[ 60s ] thds: 10 tps: 519.44 qps: 10391.96 (r/w/o: 7274.84/2078.15/1038.97) lat (ms,95%): 22.28 err/s: 0.03 reconn/s: 0.00
...
[ 600s ] thds: 10 tps: 269.64 qps: 5393.17 (r/w/o: 3775.36/1078.54/539.27) lat (ms,95%): 134.90 err/s: 0.00 reconn/s: 0.00
SQL statistics:
    queries performed:
        read:                            2675498
        write:                           764419
        other:                           382217
        total:                           3822134
    transactions:                        191104 (318.28 per sec.)
    queries:                             3822134 (6365.71 per sec.)
    ignored errors:                      3      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          600.4233s
    total number of events:              191104

Latency (ms):
         min:                                    3.04
         avg:                                   31.38
         max:                                 1656.19
         95th percentile:                      134.90
         sum:                              5997719.22

Threads fairness:
    events (avg/stddev):           19110.4000/283.68
    execution time (avg/stddev):   599.7719/0.46
```
По результатам теста количество transactions увеличилось с 93.32 per sec до 318.28 per sec, количество queries с 1866.45 per sec до 6365.71 per sec. 
Это говорит о том, что при уменьшении влияния медленной дисковой подсистемы на результаты теста, можно добиться ощутимого прироста производительности, 
но данные параметры нельзя применять в продуктивной среде в силу отсутствия надёжности данных.

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
Запустим тест oltp_read_write на 10 минут, использую 10 потоков:
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

```console
```
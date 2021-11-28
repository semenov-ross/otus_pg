# Работа с большим объемом реальных данных
### Разворачиваем и настраиваем БД с большими данными

Для работы используем публичные данные из BigQuery, экпортировав таблицу bigquery-public-data:chicago_taxi_trips.taxi_trips в созданный bucket taxi_trips_20211128:
```console
[ross@otuspg ~]$ gsutil mb gs://taxi_trips_20211128
Creating gs://taxi_trips_20211128/...

[ross@otuspg ~]$ bq extract 'bigquery-public-data:chicago_taxi_trips.taxi_trips' gs://taxi_trips_20211128/taxi_trips_*.csv

Waiting on bqjob_r6a64f0ef2df8c702_0000017d656a23c6_1 ... (68s) Current status: DONE

ross@otuspg ~]$ gsutil ls gs://taxi_trips_20211128/
gs://taxi_trips_20211128/taxi_trips_000000000000.csv
gs://taxi_trips_20211128/taxi_trips_000000000001.csv
gs://taxi_trips_20211128/taxi_trips_000000000002.csv
...
gs://taxi_trips_20211128/taxi_trips_000000000295.csv
gs://taxi_trips_20211128/taxi_trips_000000000296.csv
```

В Google Cloud Platform создана ВМ g14-bigdata с увеличенным объёмом диска для загрузки большого количества данных:
```console
gcloud compute instances create pg14-bigdata ... size=50 ...
```
Для использования экпортированных данных подключим bucket taxi_trips_20211128 к созданной ВМ при помощи утилиты gcsfuse:
```console
[ross@otuspg ~]$ gcloud compute ssh pg14-bigdata

[ross@otuspg ~]$ yum install gcsfuse -y

[root@pg14-bigdata ~]# gcsfuse taxi_trips_20211128 /mnt/taxi_trips
Using mount point: /mnt/taxi_trips
Opening GCS connection...
Opening bucket...
Mounting file system...
File system has been successfully mounted.

[root@pg14-bigdata ~]# ls -l /mnt/taxi_trips/
total 77503631
-rw-r--r--. 1 root root 267554544 Nov 28 07:20 taxi_trips_000000000000.csv
-rw-r--r--. 1 root root 266280306 Nov 28 07:20 taxi_trips_000000000001.csv
-rw-r--r--. 1 root root 267544192 Nov 28 07:20 taxi_trips_000000000002.csv
...
-rw-r--r--. 1 root root 266380274 Nov 28 07:20 taxi_trips_000000000295.csv
-rw-r--r--. 1 root root 267550270 Nov 28 07:20 taxi_trips_000000000296.csv
```
Для сравнения скорости выполнения запросов выбираем Percona Server for MySQL Version: 8.0.26:
```console
[root@pg14-bigdata ~]# yum install https://repo.percona.com/yum/percona-release-latest.noarch.rpm -y

[root@pg14-bigdata ~]# percona-release setup ps80
* Disabling all Percona Repositories
* Enabling the Percona Server 8.0 repository
* Enabling the Percona Tools repository
<*> All done!

[root@pg14-bigdata ~]# yum install percona-server-server.x86_64 -y

[root@pg14-bigdata ~]# systemctl start mysqld.service
[root@pg14-bigdata ~]# grep -i password /var/log/mysqld.log > percona.start
[root@pg14-bigdata ~]# mysql_secure_installation
[root@pg14-bigdata ~]# mysql -p
mysql> \s
--------------
mysql  Ver 8.0.26-16 for Linux on x86_64 (Percona Server (GPL), Release 16, Revision 3d64165)

Connection id:          11
Current database:
Current user:           root@localhost
SSL:                    Not in use
Current pager:          stdout
Using outfile:          ''
Using delimiter:        ;
Server version:         8.0.26-16 Percona Server (GPL), Release 16, Revision 3d64165
Protocol version:       10
Connection:             Localhost via UNIX socket
Server characterset:    utf8mb4
Db     characterset:    utf8mb4
Client characterset:    utf8mb4
Conn.  characterset:    utf8mb4
UNIX socket:            /var/lib/mysql/mysql.sock
Binary data as:         Hexadecimal
Uptime:                 9 min 19 sec

Threads: 2  Questions: 18  Slow queries: 0  Opens: 133  Flush tables: 3  Open tables: 49  Queries per second avg: 0.032
--------------

```
Для загрузки данных используем тестовую БД и отдельного пользователя:
```console
mysql> CREATE DATABASE otus;
Query OK, 1 row affected (0.01 sec)

mysql> CREATE USER 'loader'@'localhost' IDENTIFIED BY '=LoaderPassword1';
Query OK, 0 rows affected (0.01 sec)

mysql> GRANT ALL PRIVILEGES ON otus.* TO 'loader'@'localhost';
Query OK, 0 rows affected (0.00 sec)

mysql> FLUSH PRIVILEGES;
Query OK, 0 rows affected (0.01 sec)
```
Создаём таблицу для загружаемых данных:
```console
create table taxi_trips (
unique_key text, 
taxi_id text, 
trip_start_timestamp TIMESTAMP, 
trip_end_timestamp TIMESTAMP, 
trip_seconds bigint, 
trip_miles numeric, 
pickup_census_tract bigint, 
dropoff_census_tract bigint, 
pickup_community_area bigint, 
dropoff_community_area bigint, 
fare numeric, 
tips numeric, 
tolls numeric, 
extras numeric, 
trip_total numeric, 
payment_type text, 
company text, 
pickup_latitude numeric, 
pickup_longitude numeric, 
pickup_location text, 
dropoff_latitude numeric, 
dropoff_longitude numeric, 
dropoff_location text
);
```

Для возможности загрузки данных из локальных файлов меняем системные параметры и перемонтируем bucket taxi_trips_20211128 в каталог, определённый для загрузки:
```console
mysql> SHOW VARIABLES LIKE "secure_file_priv";
+------------------+-----------------------+
| Variable_name    | Value                 |
+------------------+-----------------------+
| secure_file_priv | /var/lib/mysql-files/ |
+------------------+-----------------------+
1 row in set (0.01 sec)

[root@pg14-bigdata ~]# fusermount -u /mnt/taxi_trips
[root@pg14-bigdata ~]# gcsfuse taxi_trips_20211128 /var/lib/mysql-files
Using mount point: /var/lib/mysql-files
Using mount point: /var/lib/mysql-files
Opening GCS connection...
Opening bucket...
Mounting file system...
File system has been successfully mounted.

[root@pg14-bigdata ~]# mysql -p
mysql> SHOW GLOBAL VARIABLES LIKE 'local_infile';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| local_infile  | OFF   |
+---------------+-------+
1 row in set (0.01 sec)

mysql> SET GLOBAL local_infile=1;
mysql> \q
[root@pg14-bigdata ~]# mysql --local-infile=1 otus -u loader -p

```
Создаем для удобства скрипт загрузки и добавляем данные авторизации пользователя loader с помощью утилиты mysql_config_editor:
```console
[root@pg14-bigdata ~]# mysql_config_editor set --login-path=loader --host=localhost --user=loader --password
[root@pg14-bigdata ~]# cat loader_percona.sh 
#!/bin/bash

for file in /var/lib/mysql-files/taxi_trips_0000000000{00..35}.csv; do
    echo -e "Processing $file file..."
    mysql --login-path=loader --local-infile=1 --database=otus \
    -e 'LOAD DATA LOCAL INFILE '\"$file\"' INTO TABLE taxi_trips FIELDS TERMINATED BY "," LINES TERMINATED BY "\n" IGNORE 1 LINES;'
done

```
Загружаем данные:
```console
[root@pg14-bigdata ~]# time ./loader_percona.sh
Processing /var/lib/mysql-files/taxi_trips_000000000000.csv file...
...
Processing /var/lib/mysql-files/taxi_trips_000000000035.csv file...

real    32m25.533s
user    0m3.060s
sys     0m13.888s

```
Смотрим статистику:
```console
mysql> select count(*) from taxi_trips;
+----------+
| count(*) |
+----------+
| 23407650 |
+----------+
1 row in set (5 min 26.44 sec)

mysql> SELECT * FROM INFORMATION_SCHEMA.INNODB_TABLESPACES WHERE SPACE=4\G
*************************** 1. row ***************************
          SPACE: 4
           NAME: otus/taxi_trips
           FLAG: 16417
     ROW_FORMAT: Dynamic
      PAGE_SIZE: 16384
  ZIP_PAGE_SIZE: 0
     SPACE_TYPE: Single
  FS_BLOCK_SIZE: 4096
      FILE_SIZE: 10141827072
 ALLOCATED_SIZE: 10141827072
AUTOEXTEND_SIZE: 0
 SERVER_VERSION: 8.0.26
  SPACE_VERSION: 1
     ENCRYPTION: N
          STATE: normal
1 row in set (0.00 sec)

mysql> SELECT * FROM INFORMATION_SCHEMA.INNODB_DATAFILES WHERE SPACE=4\G
*************************** 1. row ***************************
SPACE: 0x34
 PATH: ./otus/taxi_trips.ibd
1 row in set (0.01 sec)

mysql> SELECT NAME,FILE_SIZE / ( 1024 * 1024 * 1024 ) AS "Size (Gb)" FROM INFORMATION_SCHEMA.INNODB_TABLESPACES WHERE NAME='otus/taxi_trips';
+-----------------+-----------+
| NAME            | Size (Gb) |
+-----------------+-----------+
| otus/taxi_trips |    9.4453 |
+-----------------+-----------+
1 row in set (0.00 sec)
```
Выполняем запрос с операциями группировки и сортировки для оценки времени:
```console
[root@pg14-bigdata ~]# mysql --login-path=loader --database=otus

mysql> SELECT payment_type, round(sum(tips)/sum(trip_total)*100, 0) + 0 as tips_percent, count(*) AS c
FROM `taxi_trips`
GROUP BY payment_type
ORDER BY 3;

+--------------+--------------+----------+
| payment_type | tips_percent | c        |
+--------------+--------------+----------+
| Prepaid      |            0 |       76 |
| Way2ride     |           15 |       78 |
| Pcard        |            3 |     4917 |
| Dispute      |            0 |    12835 |
| Mobile       |           15 |    32715 |
| Prcard       |            1 |    41119 |
| Unknown      |            2 |    62569 |
| No Charge    |            3 |   111774 |
| Credit Card  |           17 |  9706048 |
| Cash         |            0 | 13435519 |
+--------------+--------------+----------+
10 rows in set (5 min 43.78 sec)

mysql> EXPLAIN SELECT payment_type, round(sum(tips)/sum(trip_total)*100, 0) + 0 as tips_percent, count(*) AS c FROM `taxi_trips` GROUP BY payment_type ORDER BY 3;
+----+-------------+------------+------------+------+---------------+------+---------+------+----------+----------+---------------------------------+
| id | select_type | table      | partitions | type | possible_keys | key  | key_len | ref  | rows     | filtered | Extra                           |
+----+-------------+------------+------------+------+---------------+------+---------+------+----------+----------+---------------------------------+
|  1 | SIMPLE      | taxi_trips | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 22274793 |   100.00 | Using temporary; Using filesort |
+----+-------------+------------+------------+------+---------------+------+---------+------+----------+----------+---------------------------------+
1 row in set, 1 warning (0.01 sec)
```
Создаём новый кластер PostgresSQL 14:
```console
[root@pg14-bigdata ~]# yum -y install https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
[root@pg14-bigdata ~]# yum -y install postgresql14 postgresql14-server
[root@pg14-bigdata ~]# /usr/pgsql-14/bin/postgresql-14-setup initdb
Initializing database ... OK
[root@pg14-bigdata ~]# systemctl enable --now postgresql-14.service
-bash-4.2$ psql 
psql (14.1)
Type "help" for help.
postgres=# CREATE DATABASE otus;
CREATE DATABASE
postgres=# CREATE USER loader WITH PASSWORD 'password' INHERIT;
CREATE ROLE
postgres=# ALTER DATABASE otus OWNER TO loader;
ALTER DATABASE
```
Создаем тестовую таблицу в БД otus
```console
-bash-4.2$ psql otus -U loader
otus=> create table taxi_trips (
otus(> unique_key text, 
otus(> taxi_id text, 
otus(> trip_start_timestamp TIMESTAMP, 
otus(> trip_end_timestamp TIMESTAMP, 
otus(> trip_seconds bigint, 
otus(> trip_miles numeric, 
otus(> pickup_census_tract bigint, 
otus(> dropoff_census_tract bigint, 
otus(> pickup_community_area bigint, 
otus(> dropoff_community_area bigint, 
otus(> fare numeric, 
otus(> tips numeric, 
otus(> tolls numeric, 
otus(> extras numeric, 
otus(> trip_total numeric, 
otus(> payment_type text, 
otus(> company text, 
otus(> pickup_latitude numeric, 
otus(> pickup_longitude numeric, 
otus(> pickup_location text, 
otus(> dropoff_latitude numeric, 
otus(> dropoff_longitude numeric, 
otus(> dropoff_location text
otus(> );
CREATE TABLE

otus=> \dt+
                                       List of relations
 Schema |    Name    | Type  | Owner  | Persistence | Access method |    Size    | Description 
--------+------------+-------+--------+-------------+---------------+------------+-------------
 public | taxi_trips | table | loader | permanent   | heap          | 8192 bytes | 
(1 row)
```
Перемонтируем bucket, дадим права пользователю loader для использования команды COPY и загрузим данные подготовленным скриптом:
```console
[root@pg14-bigdata ~]# fusermount -u /var/lib/mysql-files
[root@pg14-bigdata pgsql]# gcsfuse -o allow_other --dir-mode 777 --file-mode 777 taxi_trips_20211128 /mnt/taxi_trips/
Using mount point: /mnt/taxi_trips
Opening GCS connection...
Opening bucket...
Mounting file system...
File system has been successfully mounted.

postgres=# GRANT pg_read_server_files TO loader;
GRANT ROLE

-bash-4.2$ cat loader_pg.sh 
#!/bin/bash

for file in /mnt/taxi_trips/taxi_trips_0000000000{00..35}.csv; do
    echo -e "Processing $file file..."
    psql otus -U loader -c "COPY taxi_trips(unique_key, taxi_id, trip_start_timestamp, trip_end_timestamp, trip_seconds, trip_miles, pickup_census_tract, dropoff_census_tract, pickup_community_area, dropoff_community_area, fare, tips, tolls, extras, trip_total, payment_type, company, pickup_latitude, pickup_longitude, pickup_location, dropoff_latitude, dropoff_longitude, dropoff_location) FROM '$file' DELIMITER ',' CSV HEADER;";
done

bash-4.2$ time ./loader_pg.sh 
Processing /mnt/taxi_trips/taxi_trips_000000000000.csv file...
COPY 653941
...
Processing /mnt/taxi_trips/taxi_trips_000000000035.csv file...
COPY 657503

real    16m29.665s
user    0m0.145s
sys     0m0.183s
```
Смотрим статистику:
```console
-bash-4.2$ psql otus -U loader
psql (14.1)
Type "help" for help.
otus=> \l+ otus
                                               List of databases
 Name | Owner  | Encoding |   Collate   |    Ctype    | Access privileges |  Size   | Tablespace | Description 
------+--------+----------+-------------+-------------+-------------------+---------+------------+-------------
 otus | loader | UTF8     | en_US.UTF-8 | en_US.UTF-8 |                   | 9814 MB | pg_default |
(1 row)

otus=> \dt+
                                     List of relations
 Schema |    Name    | Type  | Owner  | Persistence | Access method |  Size   | Description 
--------+------------+-------+--------+-------------+---------------+---------+-------------
 public | taxi_trips | table | loader | permanent   | heap          | 9805 MB |
(1 row)

otus=> \timing on
Timing is on.
otus=> select count(*) from taxi_trips;
  count   
----------
 23407650
(1 row)

Time: 332665.666 ms (05:32.666)
```
Выполняем тот же запрос, что и в БД MySQL:
```console
otus=> explain SELECT payment_type, round(sum(tips)/sum(trip_total)*100, 0) + 0 as tips_percent, count(*) AS c
FROM taxi_trips 
GROUP BY payment_type
ORDER BY 3;
                                                  QUERY PLAN
--------------------------------------------------------------------------------------------------------------
 Sort  (cost=1450693.65..1450693.67 rows=8 width=47)
   Sort Key: (count(*))
   ->  Finalize GroupAggregate  (cost=1450691.23..1450693.53 rows=8 width=47)
         Group Key: payment_type
         ->  Gather Merge  (cost=1450691.23..1450693.09 rows=16 width=79)
               Workers Planned: 2
               ->  Sort  (cost=1449691.20..1449691.22 rows=8 width=79)
                     Sort Key: payment_type
                     ->  Partial HashAggregate  (cost=1449690.96..1449691.08 rows=8 width=79)
                           Group Key: payment_type
                           ->  Parallel Seq Scan on taxi_trips  (cost=0.00..1352203.98 rows=9748698 width=17)
(11 rows)

Time: 16.468 ms

otus=> SELECT payment_type, round(sum(tips)/sum(trip_total)*100, 0) + 0 as tips_percent, count(*) AS c
FROM taxi_trips
GROUP BY payment_type
ORDER BY 3;
 payment_type | tips_percent |    c
--------------+--------------+----------
 Prepaid      |            0 |       76
 Way2ride     |           15 |       78
 Pcard        |            3 |     4917
 Dispute      |            0 |    12835
 Mobile       |           15 |    32715
 Prcard       |            1 |    41119
 Unknown      |            2 |    62569
 No Charge    |            3 |   111774
 Credit Card  |           17 |  9706048
 Cash         |            0 | 13435519
(10 rows)

Time: 329209.217 ms (05:29.209)
```
По результатм тестирования можно сделать вывод, что скорость загрузки данных в БД PostgreSQL из файла csv оказалось существенно меньше чем в БД MySQL, а скорость выполнения запросов, 
в которых используется сканирование всей таблицы соизмерима. 
Так же в БД PostgreSQL потребовалось больше места на файловой системе для хранения одного и того же объёма данных, что и в БД MySQL.

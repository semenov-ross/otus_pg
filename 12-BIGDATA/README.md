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

mysql> CREATE USER 'loader'@'localhost' IDENTIFIED BY '=Vpds+u)H6aW';
Query OK, 0 rows affected (0.01 sec)

mysql> GRANT ALL PRIVILEGES ON otus.* TO 'loader'@'localhost';
Query OK, 0 rows affected (0.00 sec)

mysql> FLUSH PRIVILEGES;
Query OK, 0 rows affected (0.01 sec)
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
[root@pg14-bigdata ~]# gcsfuse taxi_trips_20211126 /var/lib/mysql-files
Using mount point: /var/lib/mysql-files
Opening GCS connection...
Opening bucket...
daemonize.Run: readFromProcess: sub-process: mountWithArgs: mountWithConn: setUpBucket: OpenBucket: Unknown bucket "taxi_trips_20211126"

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

```console
```
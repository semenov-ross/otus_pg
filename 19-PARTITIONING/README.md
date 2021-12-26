# Секционирование
### Научиться секционировать таблицы. Секционировать большую таблицу из демо базы flights

В Google Cloud Platform создана ВМ pg14-partition, установлен PostgreSQL 14 и импортирована демонстрационная БД от компании Postgres Professional:
```console
[ross@otuspg ~]$ gcloud compute instances create pg14-partition ....
[ross@otuspg ~]$ gcloud compute ssh pg14-partition
[root@pg14-partition ~]# yum -y install https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
[root@pg14-partition ~]# yum -y install postgresql14 postgresql14-server
[root@pg14 ~]# /usr/pgsql-14/bin/postgresql-14-setup initdb
[root@pg14 ~]# systemctl enable --now postgresql-14.service
[root@pg14-partition ~]# su - postgres
-bash-4.2$ curl https://edu.postgrespro.ru/demo-big.zip --output demo-big.zip
-bash-4.2$ unzip demo-big.zip
-bash-4.2$ psql < demo-big-20170815.sql
```
Просмотрим таблицы БД demo:
```console
-bash-4.2$ psql
postgres=# \c demo
demo=# \dt+
                                                List of relations
  Schema  |      Name       | Type  |  Owner   | Persistence | Access method |  Size  |        Description        
----------+-----------------+-------+----------+-------------+---------------+--------+---------------------------
 bookings | aircrafts_data  | table | postgres | permanent   | heap          | 16 kB  | Aircrafts (internal data)
 bookings | airports_data   | table | postgres | permanent   | heap          | 56 kB  | Airports (internal data)
 bookings | boarding_passes | table | postgres | permanent   | heap          | 455 MB | Boarding passes
 bookings | bookings        | table | postgres | permanent   | heap          | 105 MB | Bookings
 bookings | flights         | table | postgres | permanent   | heap          | 21 MB  | Flights
 bookings | seats           | table | postgres | permanent   | heap          | 96 kB  | Seats
 bookings | ticket_flights  | table | postgres | permanent   | heap          | 547 MB | Flight segment
 bookings | tickets         | table | postgres | permanent   | heap          | 386 MB | Tickets
(8 rows)
```

```console
```
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
Диаграмма схемы Bookings:
![bookings](https://raw.githubusercontent.com/semenov-ross/otus_pg/master/19-PARTITIONING/demodb-bookings-schema.svg)

Рассмотрим запрос, выводящий записи количества рейсов из опрделённого пункта за указанную дату:
```console
demo=# explain analyze select count(*),a.city ->> lang() AS city,a.airport_name ->> lang() AS airport_name from flights f left join airports_data a on f.departure_airport=a.airport_code where a.city ->> lang() ='Москва' and f.scheduled_departure >= '2017-08-05'::date and f.scheduled_departure < '2017-08-06'::date group by 2,3 order by 1 desc;
                                                             QUERY PLAN                                                             
------------------------------------------------------------------------------------------------------------------------------------
 Sort  (cost=5887.24..5887.24 rows=1 width=72) (actual time=132.324..132.327 rows=3 loops=1)
   Sort Key: (count(*)) DESC
   Sort Method: quicksort  Memory: 25kB
   ->  GroupAggregate  (cost=5886.66..5887.23 rows=1 width=72) (actual time=132.291..132.313 rows=3 loops=1)
         Group Key: ((a.city ->> lang())), ((a.airport_name ->> lang()))
         ->  Sort  (cost=5886.66..5886.67 rows=5 width=64) (actual time=132.274..132.283 rows=131 loops=1)
               Sort Key: ((a.airport_name ->> lang()))
               Sort Method: quicksort  Memory: 35kB
               ->  Nested Loop  (cost=0.00..5886.60 rows=5 width=64) (actual time=12.414..132.118 rows=131 loops=1)
                     Join Filter: (f.departure_airport = a.airport_code)
                     Rows Removed by Join Filter: 1576
                     ->  Seq Scan on airports_data a  (cost=0.00..30.56 rows=1 width=114) (actual time=0.424..0.731 rows=3 loops=1)
                           Filter: ((city ->> lang()) = 'Москва'::text)
                           Rows Removed by Filter: 101
                     ->  Seq Scan on flights f  (cost=0.00..5847.00 rows=521 width=4) (actual time=0.125..43.479 rows=569 loops=3)
                           Filter: ((scheduled_departure >= '2017-08-05'::date) AND (scheduled_departure < '2017-08-06'::date))
                           Rows Removed by Filter: 214298
 Planning Time: 0.563 ms
 Execution Time: 132.453 ms
(19 rows)
```

```console
```
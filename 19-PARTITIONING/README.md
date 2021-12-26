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
Рассмотрим возможность уменьшения времени выполнения запроса путём создания партиционированной таблицы по диапазону дат.  
Создадим таблицу bookings.flights_sd партиционированную по полю scheduled_departure:
```console
CREATE TABLE bookings.flights_sd (
         flight_id integer NOT NULL,
         flight_no character(6) NOT NULL,
         scheduled_departure timestamp with time zone NOT NULL,
         scheduled_arrival timestamp with time zone NOT NULL,
         departure_airport character(3) NOT NULL,
         arrival_airport character(3) NOT NULL,
         status character varying(20) NOT NULL,
         aircraft_code character(3) NOT NULL,
         actual_departure timestamp with time zone NULL,
         actual_arrival timestamp with time zone NULL
 ) partition by range (scheduled_departure);
```
Создадим таблицы-партиции, с помощью скрипта PL/pgSQL:
```console
demo=# \! cat fl_sd.sql
DO $$
DECLARE
    sd date;
BEGIN
    FOR sd IN SELECT distinct date(scheduled_departure) FROM flights ORDER BY 1
    LOOP
    RAISE NOTICE '%', sd;
    EXECUTE format('CREATE TABLE IF NOT EXISTS flights_%s PARTITION OF flights_sd FOR VALUES FROM (%L) TO (%L)', to_char(sd, 'YYYYMMDD'), to_char(sd, 'YYYY-MM-DD')::date, to_char(sd, 'YYYY-MM-DD')::date + INTERVAL '1 day');
    END LOOP;
END;
$$ LANGUAGE plpgsql;
demo=# \i fl_sd.sql
psql:fl_sd.sql:11: NOTICE:  2017-07-15
...
psql:fl_sd.sql:11: NOTICE:  2017-09-14
DO
```
Заполним данными партиционированную таблицу:
```console
demo=# insert into flights_sd select * from flights;
INSERT 0 33121
```
Выполним такой же запрос к партиционированной таблице:
```console
explain analyze select count(*),a.city ->> lang() AS city,a.airport_name ->> lang() AS airport_name from flights_sd f left join airports_data a on f.departure_airport=a.airport_code where a.city ->> lang() ='Москва' and f.scheduled_departure >= '2017-08-05'::date and f.scheduled_departure < '2017-08-06'::date group by 2,3 order by 1 desc;
                                                                   QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------------------------------
 Sort  (cost=6109.76..6109.77 rows=1 width=72) (actual time=1.320..1.321 rows=3 loops=1)
   Sort Key: (count(*)) DESC
   Sort Method: quicksort  Memory: 25kB
   ->  GroupAggregate  (cost=6109.19..6109.75 rows=1 width=72) (actual time=1.294..1.315 rows=3 loops=1)
         Group Key: ((a.city ->> lang())), ((a.airport_name ->> lang()))
         ->  Sort  (cost=6109.19..6109.20 rows=5 width=64) (actual time=1.283..1.291 rows=131 loops=1)
               Sort Key: ((a.airport_name ->> lang()))
               Sort Method: quicksort  Memory: 35kB
               ->  Nested Loop  (cost=0.00..6109.13 rows=5 width=64) (actual time=0.129..1.239 rows=131 loops=1)
                     Join Filter: (f.departure_airport = a.airport_code)
                     Rows Removed by Join Filter: 1576
                     ->  Seq Scan on airports_data a  (cost=0.00..30.56 rows=1 width=114) (actual time=0.077..0.197 rows=3 loops=1)
                           Filter: ((city ->> lang()) = 'Москва'::text)
                           Rows Removed by Filter: 101
                     ->  Append  (cost=0.00..6063.97 rows=966 width=4) (actual time=0.005..0.189 rows=569 loops=3)
                           Subplans Removed: 396
                           ->  Seq Scan on flights_20170805 f_1  (cost=0.00..16.54 rows=569 width=4) (actual time=0.005..0.141 rows=569 loops=3)
                                 Filter: ((scheduled_departure >= '2017-08-05'::date) AND (scheduled_departure < '2017-08-06'::date))
 Planning Time: 12.422 ms
 Execution Time: 1.392 ms
(20 rows)
```
Как можно убедиться время выполнения запроса существенно сократилось, но увеличелось время построения плана запроса.

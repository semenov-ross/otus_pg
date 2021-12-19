# Работа с индексами, join'ами, статистикой

Для работы используем демонстрационную БД DVD rental - https://www.postgresqltutorial.com/postgresql-sample-database

```console
wget https://www.postgresqltutorial.com/wp-content/uploads/2019/05/dvdrental.zip
postgres=# CREATE DATABASE dvdrental;
CREATE DATABASE
pg_restore -U postgres -d dvdrental dvd
```
Постоим индекс в таблице film по полю rating для ускорения выборки по этому полю:
```console
dvdrental=# explain select * from film where rating = 'G';
                       QUERY PLAN
---------------------------------------------------------
 Seq Scan on film  (cost=0.00..66.50 rows=178 width=384)
   Filter: (rating = 'G'::mpaa_rating)
(2 rows)
dvdrental=# create index on film (rating);
CREATE INDEX
dvdrental=# explain select * from film where rating = 'G';
                                   QUERY PLAN
--------------------------------------------------------------------------------
 Bitmap Heap Scan on film  (cost=5.53..61.75 rows=178 width=384)
   Recheck Cond: (rating = 'G'::mpaa_rating)
   ->  Bitmap Index Scan on film_rating_idx  (cost=0.00..5.49 rows=178 width=0)
         Index Cond: (rating = 'G'::mpaa_rating)
(4 rows)
```

```console
```

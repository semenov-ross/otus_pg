# Работа с индексами, join'ами, статистикой

Для работы используем демонстрационную БД DVD rental - https://www.postgresqltutorial.com/postgresql-sample-database

```console
wget https://www.postgresqltutorial.com/wp-content/uploads/2019/05/dvdrental.zip
postgres=# CREATE DATABASE dvdrental;
CREATE DATABASE
pg_restore -U postgres -d dvdrental dvd
```
Постоим индекс в таблице film по полю rating(Рейтинговая система MPAA) для ускорения выборки по этому полю:
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
Для добавления полнотекстового поиска в таблице film, создадим колонку description_full c типом tsvector и построим по ней индекс:
```console
dvdrental=# ALTER TABLE film ADD COLUMN description_full tsvector;
ALTER TABLE
dvdrental=# UPDATE film SET description_full=to_tsvector(description);
UPDATE 1000
dvdrental=# CREATE INDEX ON film USING gin (description_full);
CREATE INDEX

dvdrental=# EXPLAIN SELECT description FROM film WHERE description_full @@ to_tsquery('Beautiful');
                                       QUERY PLAN                                        
-----------------------------------------------------------------------------------------
 Bitmap Heap Scan on film  (cost=8.58..105.87 rows=42 width=94)
   Recheck Cond: (description_full @@ to_tsquery('Beautiful'::text))
   ->  Bitmap Index Scan on film_description_full_idx  (cost=0.00..8.56 rows=42 width=0)
         Index Cond: (description_full @@ to_tsquery('Beautiful'::text))
(4 rows)
```

```console
```

```console
```
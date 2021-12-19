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
Создадим индекс в таблице payment на поле с функцией, для подсчёта платежей за определённую дату:
```console
dvdrental=# explain select sum(amount) from payment where date(payment_date)='2007-02-19';
                           QUERY PLAN
----------------------------------------------------------------
 Aggregate  (cost=327.12..327.13 rows=1 width=32)
   ->  Seq Scan on payment  (cost=0.00..326.94 rows=73 width=6)
         Filter: (date(payment_date) = '2007-02-19'::date)
(3 rows)

dvdrental=# CREATE INDEX ON payment (date(payment_date));
CREATE INDEX

dvdrental=# explain select sum(amount) from payment where date(payment_date)='2007-02-19';
                                      QUERY PLAN
--------------------------------------------------------------------------------------
 Aggregate  (cost=108.38..108.39 rows=1 width=32)
   ->  Bitmap Heap Scan on payment  (cost=4.85..108.20 rows=73 width=6)
         Recheck Cond: (date(payment_date) = '2007-02-19'::date)
         ->  Bitmap Index Scan on payment_date_idx  (cost=0.00..4.83 rows=73 width=0)
               Index Cond: (date(payment_date) = '2007-02-19'::date)
(5 rows)

```
Создадим индекс по двум полям для выборки из таблицы film:
```console
dvdrental=# explain select count(*) from film where rating='G' and length>50;
                           QUERY PLAN
-----------------------------------------------------------------
 Aggregate  (cost=141.43..141.44 rows=1 width=8)
   ->  Seq Scan on film  (cost=0.00..141.00 rows=171 width=0)
         Filter: ((length > 50) AND (rating = 'G'::mpaa_rating))
(3 rows)

dvdrental=# CREATE INDEX ON film (rating,length);
CREATE INDEX

dvdrental=# explain select count(*) from film where rating='G' and length>50;
                                           QUERY PLAN
------------------------------------------------------------------------------------------------
 Aggregate  (cost=8.12..8.13 rows=1 width=8)
   ->  Index Only Scan using film_rating_length_idx on film  (cost=0.28..7.70 rows=171 width=0)
         Index Cond: ((rating = 'G'::mpaa_rating) AND (length > 50))
(3 rows)
```

```console
```

```console
```

# Модуль 5 «Внешние данные и миграция данных»
# Меняем дефолтные настройки и перезагружаем PG

```
ALTER SYSTEM SET shared_buffers = '2GB';
ALTER SYSTEM SET random_page_cost = 1;
ALTER SYSTEM SET JIT = OFF;
```
Cоздаем таблицу и загружаем данные

```
CREATE TABLE IF NOT EXISTS taxi_trips(
unique_key text
,taxi_id text
,trip_start_timestamp timestamp
,trip_end_timestamp timestamp
,trip_seconds bigint
,trip_miles float
,pickup_census_tract bigint
,dropoff_census_tract bigint
,pickup_community_area bigint
,dropoff_community_area bigint
,fare float
,tips float
,tolls float
,extras float
,trip_total float
,payment_type text
,company text
,pickup_latitude float
,pickup_longitude float
,pickup_location text
,dropoff_latitude float
,dropoff_longitude float
,dropoff_location text
);
```
Скачивание не описываю, оно есть в чате
--загрузка

```
cd /var/lib/postgresql/trips
for f in *.csv*; do psql -d db_taxi -c "\\COPY taxi_trips FROM PROGRAM 'cat $f' CSV HEADER"; 
```
создаем покрывающий индекс
```
CREATE INDEX IF NOT EXISTS idx_taxi_trip ON taxi_trips(payment_type) INCLUDE(tips,fare);
```
обновляем карту видимости и собираем статистику
```
VACUUM ANALYZE taxi_trips;
```
устанавливаем pg_buffercache
```
CREATE EXTENSION IF NOT EXISTS pg_buffercache;
SELECT oid,relpages
FROM pg_class
WHERE relname = 'idx_taxi_trip';
```
--746756	142735

проверяем есть ли индекс в кеш
```
SELECT count(*)
FROM pg_buffercache b
JOIN pg_class c ON b.relfilenode = c.relfilenode
WHERE c.oid = 746756;
```
--0 - нет(

--прогреваем кеш и проставляем признак фиксации транзакций (после загрузки)
```
EXPLAIN ANALYZE SELECT count(*) FROM taxi_trips;
```
проверяем есть ли индекс в кеш
```
SELECT count(*)
FROM pg_buffercache b
JOIN pg_class c ON b.relfilenode = c.relfilenode
WHERE c.oid = 746756;
```
--141954 - появился

--запускаем запрос
```
EXPLAIN (ANALYZE,buffers,VERBOSE)
SELECT
	payment_type
,	round(sum(tips) / sum(tips + fare) * 100) tips_persent
,	count(*)
FROM
	taxi_trips
GROUP BY
	payment_type
ORDER BY
	3 DESC;
```
/*
Важно: Прочитали все из индекса:
Parallel Index Only Scan using idx_taxi_trip
полностью попали в кеш:
Buffers: shared hit=142461
не ходили в heap, тк карта видимости свежая:
Heap Fetches: 0
И выпоняли в два потока:
Workers Launched: 2

как доп. оптимизация, можно поиграться с кол-вом параллельных воркеров.

Итого получили
Execution Time: 1789.093 ms
вместо исходных
Execution Time: 15163.693 ms

```
*/

Sort  (cost=528322.14..528322.16 rows=8 width=23) (actual time=1741.695..1789.055 rows=11 loops=1)
  Output: payment_type, (round(((sum(tips) / sum((tips + fare))) * '100'::double precision))), (count(*))
  Sort Key: (count(*)) DESC
  Sort Method: quicksort  Memory: 25kB
  Buffers: shared hit=142461
  ->  Finalize GroupAggregate  (cost=1000.59..528322.02 rows=8 width=23) (actual time=1741.647..1789.039 rows=11 loops=1)
        Output: payment_type, round(((sum(tips) / sum((tips + fare))) * '100'::double precision)), count(*)
        Group Key: taxi_trips.payment_type
        Buffers: shared hit=142461
        ->  Gather Merge  (cost=1000.59..528321.72 rows=16 width=31) (actual time=1741.505..1789.006 rows=22 loops=1)
              Output: payment_type, (PARTIAL sum(tips)), (PARTIAL sum((tips + fare))), (PARTIAL count(*))
              Workers Planned: 2
              Workers Launched: 2
              Buffers: shared hit=142461
              ->  Partial GroupAggregate  (cost=0.56..527319.85 rows=8 width=31) (actual time=1050.275..1488.878 rows=7 loops=3)
                    Output: payment_type, PARTIAL sum(tips), PARTIAL sum((tips + fare)), PARTIAL count(*)
                    Group Key: taxi_trips.payment_type
                    Buffers: shared hit=142461
                    Worker 0:  actual time=1033.758..1691.614 rows=10 loops=1
                      Buffers: shared hit=65607
                    Worker 1:  actual time=1033.724..1691.563 rows=10 loops=1
                      Buffers: shared hit=65514
                    ->  Parallel Index Only Scan using idx_taxi_trip on public.taxi_trips  (cost=0.56..387977.67 rows=11147368 width=23) (actual time=0.219..705.386 rows=8917894 loops=3)
                          Output: payment_type, tips, fare
                          Heap Fetches: 0
                          Buffers: shared hit=142461
                          Worker 0:  actual time=0.273..836.208 rows=12242782 loops=1
                            Buffers: shared hit=65607
                          Worker 1:  actual time=0.370..836.658 rows=12218865 loops=1
                            Buffers: shared hit=65514
Query Identifier: -1550282970691615109
Planning Time: 0.091 ms
```
Execution Time: 1789.093 ms

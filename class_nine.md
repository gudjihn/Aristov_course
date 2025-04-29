# Модуль 2 «Основные управляющие конструкции в функциях и процедурах»

создадим таблицу c кол-вом занятых мест на поездке
```sql

CREATE TABLE book.ride_seat 
( 
  fkride int4 NOT NULL,
seat_cnt int4 NOT NULL,
CONSTRAINT ride_seats_pkey PRIMARY KEY (fkride),
CONSTRAINT ride_seats_fkride_fkey FOREIGN KEY (fkride) REFERENCES book.ride(id) ON DELETE CASCADE
 );
```
--при добавлении билета, увеличиваем кол-во занятых мест 
```sql

CREATE OR REPLACE FUNCTION book.trg_tickets_calc_seat_cnt() RETURNS TRIGGER
    LANGUAGE plpgsql
    AS 
$trg_tickets_calc_seat_cnt$
BEGIN
	IF TG_OP = 'INSERT' THEN
		INSERT INTO book.ride_seat(fkride, seat_cnt) 
		VALUES(NEW.fkride,1)
		ON CONFLICT(fkride)
		DO UPDATE 
		SET 
			seat_cnt = ride_seat.seat_cnt + 1;
	END IF;
	RETURN NEW;
END;
$trg_tickets_calc_seat_cnt$;
```
--вызываем подсчет после вставки
```sql
CREATE OR REPLACE TRIGGER trg_tickets_ai 
AFTER INSERT ON book.tickets 
FOR EACH ROW 
EXECUTE PROCEDURE book.trg_tickets_calc_seat_cnt();
```
--временно отключаем триггер

```sql

ALTER TABLE book.tickets DISABLE TRIGGER trg_tickets_ai;
```
--оцениваем скорость без триггера на встевке 100_000 строк

```sql

BEGIN;
EXPLAIN ANALYZE 
INSERT INTO book.tickets (fkride, fio, contact, fkseat)
SELECT
	ceil(random() * 144000) AS fkride,--144000 = SELECT max(id) FROM book.ride;
	'fio' AS fio,
	'{}'::jsonb AS contact,
	ceil(random() * 200) AS fkseat --200 = SELECT max(id) FROM book.seat;
FROM 
	generate_series(1,1e5) AS gs(i);

Insert on tickets  (cost=0.00..42.50 rows=0 width=0) (actual time=730.223..730.225 rows=0 loops=1)
  ->  Subquery Scan on "*SELECT*"  (cost=0.00..42.50 rows=1000 width=80) (actual time=7.535..224.076 rows=100000 loops=1)
        ->  Function Scan on generate_series gs  (cost=0.00..25.00 rows=1000 width=80) (actual time=7.514..86.569 rows=100000 loops=1)
Planning Time: 0.068 ms
Trigger for constraint tickets_fkride_fkey: time=2763.551 calls=100000
Trigger for constraint tickets_fkseat_fkey: time=2285.607 calls=100000
Execution Time: 5820.274 ms
```
--откатываем изменения и сжимаем таблицу, перестраиваем индекс, чтобы исключить влияние блоатинга

```sql

roolback;
VACUUM FULL book.tickets;
REINDEX (VERBOSE) INDEX CONCURRENTLY book.tickets_pkey;
```
--включаем триггер

```sql

ALTER TABLE book.tickets ENABLE TRIGGER trg_tickets_ai;
```
--оцениваем скорость с триггером

```sql

BEGIN;
EXPLAIN ANALYZE 
INSERT INTO book.tickets (fkride, fio, contact, fkseat)
SELECT
	ceil(random() * 144000) AS fkride,
	'fio' AS fio,
	'{}'::jsonb AS contact,
	ceil(random() * 200) AS fkseat
FROM 
	generate_series(1,1e5) AS gs(i);

Insert on tickets  (cost=0.00..42.50 rows=0 width=0) (actual time=674.739..674.741 rows=0 loops=1)
  ->  Subquery Scan on "*SELECT*"  (cost=0.00..42.50 rows=1000 width=80) (actual time=7.671..207.232 rows=100000 loops=1)
        ->  Function Scan on generate_series gs  (cost=0.00..25.00 rows=1000 width=80) (actual time=7.650..77.486 rows=100000 loops=1)
Planning Time: 0.060 ms
Trigger for constraint tickets_fkride_fkey: time=3013.922 calls=100000
Trigger for constraint tickets_fkseat_fkey: time=2359.790 calls=100000
Trigger trg_tickets_ai: time=7923.290 calls=100000
Execution Time: 14030.106 ms
```
--откатываем изменения и сжимаем.
```
roolback;
VACUUM FULL book.tickets;
REINDEX (VERBOSE) INDEX CONCURRENTLY book.tickets_pkey;

```

# Вывод: 5820.274 ms без триггера, 14030.106 ms с триггером время выросло очень значительно 


* обработать удаление и проверку на отрицательное количество мест/превышение
количества проданных билетов

# Проблему с отрицательным кол-вом не понял, кол-во купленных билетов на поездку всегда будет >= 0. А удалить билетов больше, чем куплено нельзя..


--По превышению количества проданных билетов:
--создаем индекс для быстрого поиска купленных билетов по поездке

```sql

CREATE INDEX CONCURRENTLY idx_tickets_fkride ON book.tickets(fkride);

```

--Создаем функцию для STATEMENT триггера 

```sql

CREATE OR REPLACE FUNCTION book.trg_tickets_calc_seat_cnt_limit() RETURNS TRIGGER
    LANGUAGE plpgsql
    AS 
$trg_tickets_calc_seat_cnt_limit$
DECLARE 
	v_total_seat_in_bus_cnt int;
	v_current_seat_in_bus_cnt int;
	v_ride int;
BEGIN 
```

--сделал для одной поездки, не стал усложнять.
--те считаем, что массово добавляем билеты всегда с одной и тойже fkride

--поездка, по которой добавляем билеты

```sql

SELECT max(fkride)
	INTO 
		v_ride
	FROM 
		new_table;
```

--общее кол-во мест в автобусе	

```sql

SELECT 
		count(*) 
	INTO 
		v_total_seat_in_bus_cnt
	FROM 
		book.ride r
		JOIN book.bus b ON b.id = r.fkbus
		JOIN book.seat s ON s.fkbus = b.id
	WHERE 
		r.id = v_ride;
```

--кол-во купленных билетов

```sql

SELECT 
		count(*) 
	INTO 
		v_current_seat_in_bus_cnt
	FROM 
		book.tickets t 
		JOIN book.ride r ON r.id = t.fkride
	WHERE 
		r.id = v_ride;


	RAISE NOTICE 'fkride = % total_cnt = % current_cnt = % ',v_ride, v_total_seat_in_bus_cnt,v_current_seat_in_bus_cnt;
```

--v_current_seat_in_bus_cnt уже содержит добавленные записи	

```sql
IF v_current_seat_in_bus_cnt > v_total_seat_in_bus_cnt THEN 
		RAISE EXCEPTION 'to many seats: total_cnt = % current_cnt = %',v_total_seat_in_bus_cnt,v_current_seat_in_bus_cnt;
	END IF;
	
	RETURN NULL;
END;
$trg_tickets_calc_seat_cnt_limit$;
```

--STATEMENT триггер с промежуточной таблицей, которая содержит добавляемые записи

```sql

CREATE OR REPLACE TRIGGER trg_tickets_ai_statement
AFTER INSERT ON book.tickets
REFERENCING
    NEW TABLE AS new_table
FOR EACH STATEMENT
EXECUTE FUNCTION book.trg_tickets_calc_seat_cnt_limit();
```

--проверяем: изначально в бд 38 записей с fkride = 1,  всего 40 свободных мест
--3 добавить не дает, а 2 добавляет

```
INSERT INTO book.tickets (fkride, fio, contact, fkseat)
SELECT
	1 AS fkride,
	'fio' AS fio,
	'{}'::jsonb AS contact,
	ceil(random() * 200) AS fkseat 
FROM 
	generate_series(1,3) AS gs(i);
```
--Еще вариант без триггера,с рекомендовательной блокировкой. Он будет работать, если вставка всегда будет выполнятся через функцию sp_add_ticket.

```sql
CREATE OR REPLACE FUNCTION book.sp_add_ticket(p_fkride int, p_fio text, p_contact jsonb, p_fkseat int)
 RETURNS void
 LANGUAGE plpgsql
 VOLATILE SECURITY DEFINER
 SET lock_timeout = '1s'
AS $function$
DECLARE 
	v_total_seat_in_bus_cnt int;
	v_current_seat_in_bus_cnt int;
BEGIN 

-- тк функция изменчивая, она будет выполнятся в своем снимке, что позволит нам видеть реальное кол-во купленных билетов
--блокируем индентификатор поездки

IF pg_try_advisory_xact_lock(p_fkride) THEN 
--подсчитываем общее кол-во мест в автобусе (можно вынести как параметр функции и подавать на вход)
	SELECT 
		count(*) 
	INTO 
		v_total_seat_in_bus_cnt
	FROM 
		book.ride r
		JOIN book.bus b ON b.id = r.fkbus
		JOIN book.seat s ON s.fkbus = b.id
	WHERE 
		r.id = p_fkride;
```

--подсчитываем кол-во купленных билетов

```sql 

SELECT 
		count(*) 
	INTO 
		v_current_seat_in_bus_cnt
	FROM 
		book.tickets t 
		JOIN book.ride r ON r.id = t.fkride
	WHERE 
		r.id = p_fkride;
```

--проверка свободных мест

```sql

IF v_current_seat_in_bus_cnt >= v_total_seat_in_bus_cnt THEN 
		raise exception 'p_fkride = % v_current_seat_in_bus_cnt = % v_total_seat_in_bus_cnt=%',p_fkride,v_current_seat_in_bus_cnt,v_total_seat_in_bus_cnt;
	END IF;
	INSERT INTO book.tickets ( fkride, fio, contact, fkseat) 
	VALUES(p_fkride, p_fio, p_contact, p_fkseat);
ELSE 
	raise exception 'Failed to acquire lock p_fkride = % ',p_fkride;
END IF;

END;

$function$
;


SELECT	
	book.sp_add_ticket(1,'fio'::text,'{}'::jsonb,ceil(random() * 200)::int)
FROM 
	generate_series(1,50) AS gs(i);

```

Вызываем функцию
```sql
SELECT id, sold_at, month FROM sales.sales_by_third_of_the_year_get(1);
 id |       sold_at       | month
----+---------------------+-------
  4 | 2025-03-18 03:32:46 |     3
 10 | 2025-01-09 05:09:26 |     1
 13 | 2025-03-24 23:04:46 |     3
 16 | 2025-03-04 02:18:10 |     3
 20 | 2025-02-22 02:07:02 |     2
 25 | 2025-02-21 02:02:23 |     2
 42 | 2025-03-30 21:01:11 |     3
 48 | 2025-03-21 07:48:58 |     3
 50 | 2025-01-28 06:29:24 |     1
```

Проверяем обработку входного параметра NULL
```sql
SELECT id, sold_at, month FROM sales.sales_by_third_of_the_year_get(null);
ERROR:  incorrect value input, coorect are: 1, 2, 3
CONTEXT:  PL/pgSQL function sales.sales_by_third_of_the_year_get(integer) line 4 at RAISE
```
Получаем исключение
```sql
ERROR:  incorrect value input, coorect are: 1, 2, 3
CONTEXT:  PL/pgSQL function sales.sales_by_third_of_the_year_get(integer) line 4 at RAISE
```

Создаем функцию для вычисления месяца по формуле
```sql
CREATE OR REPLACE FUNCTION sales.sales_by_third_of_the_year_get2(p_thirdOfYear integer)
    RETURNS TABLE
            (
                id      integer,
                sold_at timestamp,
                month   int
            )
    LANGUAGE plpgsql
    SECURITY DEFINER
AS
$$
BEGIN
    IF p_thirdOfYear NOT IN (1, 2, 3) OR p_thirdOfYear IS NULL THEN
        RAISE EXCEPTION 'Допустимые значения параметра p_thirdOfYear: 1, 2, 3';
    END IF;

    RETURN QUERY
        SELECT i.id, i.sold_at, i.month::integer
        FROM (SELECT s.id, s.sold_at, EXTRACT(MONTH FROM s.sold_at)::integer AS month
              FROM sales.sales AS s) AS i
        WHERE i.month = ANY (ARRAY(SELECT generate_series(1 + (4 * (p_thirdOfYear - 1)), 4 * p_thirdOfYear)));
END;
$$;
CREATE FUNCTION
```

Вызываем функцию
```sql
SELECT id, sold_at, month
FROM sales.sales_by_third_of_the_year_get2(1);
```
```result
SELECT id, sold_at, month
FROM sales.sales_by_third_of_the_year_get2(3);
 id |       sold_at       | month
----+---------------------+-------
  6 | 2025-10-19 00:01:59 |    10
  8 | 2025-09-19 01:22:52 |     9
  9 | 2025-09-16 00:53:00 |     9
 11 | 2025-09-09 13:02:09 |     9
 14 | 2025-11-17 23:36:40 |    11
 17 | 2025-12-01 19:35:33 |    12
 18 | 2025-10-27 22:59:12 |    10
 24 | 2025-12-21 17:44:16 |    12
 26 | 2025-10-06 07:00:34 |    10
 30 | 2025-12-31 16:45:36 |    12
 31 | 2025-11-20 17:58:42 |    11
 32 | 2025-12-14 21:58:32 |    12
 34 | 2025-12-16 20:52:37 |    12
 35 | 2025-09-05 15:39:55 |     9
 39 | 2025-09-17 18:47:42 |     9
 41 | 2025-11-15 16:54:29 |    11
 43 | 2025-12-23 12:49:05 |    12
 45 | 2025-10-17 15:15:44 |    10
 47 | 2025-11-23 02:27:17 |    11
(19 rows)

```

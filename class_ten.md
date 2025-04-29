#  Модуль 3 «Слабоструктурированные данные и асинхронная обработка»


1. Сгенерировать таблицу с 1 млн JSONB документов
2. Создать индекс
3. Обновить 1 из полей в json
4. Убедиться в блоатинге TOAST
5. Придумать методы избавится от него и проверить на практике
6. Не забываем про блоатинг индексов*


1. Сгенерировать таблицу с 1 млн JSONB документов
```sql 
DROP TABLE IF EXISTS dz10;
CREATE TABLE IF NOT EXISTS dz10 (
    id int,
    data JSONB
);

CREATE OR REPLACE FUNCTION generate_string_ascii(p_length int)
RETURNS TEXT  
LANGUAGE SQL
AS
$$
SELECT 
	string_agg(chr(floor(random() * 26 + (case when i % 2 = 0 then 65 else 97 end))::int), '')
FROM 
	generate_series(1, p_length) as gs(i);
$$;


INSERT INTO dz10(id,data)
SELECT 
	(random()*1e6)::int + 1 AS id,
	jsonb_build_object(
        'object_name', 'object_' || x % 1000,
        'object_value', (random() * 10000)::numeric(18,2),
        'object_text', generate_string_ascii(x % 10000)
    )
FROM 
	generate_series(1, 100000) AS gs(x);

SELECT
	'pg_toast' || '.' || t.relname::text AS name,
	pg_size_pretty(pg_total_relation_size(r.relname::text)) AS total_size,
	pg_size_pretty(pg_relation_size(r.relname::text)) AS table_size,
	pg_size_pretty(pg_indexes_size(r.relname::text)) AS indexes_size,
    pg_size_pretty(pg_total_relation_size('pg_toast' || '.' || t.relname::text)) AS toast_size
FROM 
	pg_class AS r 
	JOIN pg_class AS t ON t."oid" = r.reltoastrelid
WHERE 
	r.relname = 'dz10';

name                    |total_size|table_size|indexes_size|toast_size|
------------------------+----------+----------+------------+----------+
pg_toast.pg_toast_422933|546 MB    |24 MB     |0 bytes     |522 MB    |

2. Создать индекс
CREATE INDEX idx_dz10_object_value ON dz10((data ->> 'object_value'));

name                    |total_size|table_size|indexes_size|toast_size|
------------------------+----------+----------+------------+----------+
pg_toast.pg_toast_422933|548 MB    |24 MB     |2184 kB     |522 MB    |
```

3. Обновить 1 из полей в json

```sql
UPDATE dz10
SET data = 
	jsonb_set(data, '{object_value}', ((DATA->>'object_value')::numeric * 2)::text::jsonb);
```
4. Убедиться в блоатинге TOAST

```sql
name                    |total_size|table_size|indexes_size|toast_size|
------------------------+----------+----------+------------+----------+
pg_toast.pg_toast_422933|1102 MB   |48 MB     |6768 kB     |1048 MB   |


VACUUM (VERBOSE) dz10;

name                    |total_size|table_size|indexes_size|toast_size|
------------------------+----------+----------+------------+----------+
pg_toast.pg_toast_422933|574 MB    |24 MB     |2616 kB     |547 MB    |

REINDEX INDEX CONCURRENTLY idx_dz10_object_value;

name                    |total_size|table_size|indexes_size|toast_size|
------------------------+----------+----------+------------+----------+
pg_toast.pg_toast_422933|574 MB    |24 MB     |2616 kB     |547 MB    |
```

Придумать методы избавится от него и проверить на практике
Полностью избавиться от блоатинга TOAST, наверное никак.
Можно уменьшить уровень раздувания
1. Использовать более агрессивные настройки autovacuum
2. Выполнять обновления таблицы небольшими частями, с промежутками между обновлением. Выполняя VACUUM вручную, или ждать пока выполнится autovacuum
3. При наличии jsonb, можно рассмотреть вариант выноска обновляемых колонков в heap, удалив их из jsonb
4. Использовать секционирование для больших таблиц.
5. Использование более компактные названия ключей
6. Использовать свежие версии PG. Начиная с 15 значительно оптимизирован jsonb

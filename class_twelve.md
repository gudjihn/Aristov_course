# ЛОГИЧЕСКАЯ РЕПЛИКАЦИЯ
Поднимаем pg 16 и 17
```
postgres@szvb:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
16  main    5433 online postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log
17  main    5432 online postgres /var/lib/postgresql/17/main /var/log/postgresql/postgresql-17-main.log
```
17 меняем уровень wal_level
```
postgres@szvb:~$ psql -c "ALTER SYSTEM SET wal_level = logical;"
ALTER SYSTEM
```
перезагружаем
```
postgres@szvb:~$ pg_ctlcluster 17 main stop && pg_ctlcluster 17 main start
```
проверяем
```
postgres=# show wal_level;
 wal_level 
-----------
 logical
(1 row)
```

В 16 меняем уровень wal  
```
postgres@szvb:~$ psql -p 5433 -c "ALTER SYSTEM SET wal_level = logical;"
ALTER SYSTEM
```
перезагружаем
```
postgres@szvb:~$ pg_ctlcluster 16 main stop && pg_ctlcluster 16 main start
```
... проверяем

```
postgres@szvb:~$ psql -p 5433 -c "show wal_level;"
 wal_level 
-----------
 logical
(1 row)
```
экспорт схемы на 17 и импорт на 16
```
postgres@szvb:~$ pg_dumpall -s > schema.sql
postgres@szvb:~$ psql -p 5433 < schema.sql
```
16 проверяем появилась схема и пустые таблицы
```                                      List of relations
 Schema |     Name     | Type  |  Owner   | Persistence | Access method |    Size    | Description 
--------+--------------+-------+----------+-------------+---------------+------------+-------------
 book   | bus          | table | postgres | permanent   | heap          | 8192 bytes | 
 book   | busroute     | table | postgres | permanent   | heap          | 0 bytes    | 
 book   | busstation   | table | postgres | permanent   | heap          | 8192 bytes | 
 book   | fam          | table | postgres | permanent   | heap          | 8192 bytes | 
 book   | nam          | table | postgres | permanent   | heap          | 8192 bytes | 
 book   | ride         | table | postgres | permanent   | heap          | 0 bytes    | 
 book   | schedule     | table | postgres | permanent   | heap          | 0 bytes    | 
 book   | seat         | table | postgres | permanent   | heap          | 0 bytes    | 
 book   | seatcategory | table | postgres | permanent   | heap          | 8192 bytes | 
 book   | tickets      | table | postgres | permanent   | heap          | 8192 bytes | 
(10 rows)
```
17 создаем публикацию
```
CREATE PUBLICATION test_pub FOR TABLES IN SCHEMA book; 
```
16 создаем подписку с копированием данных
```
thai=# CREATE SUBSCRIPTION test_sub 
CONNECTION 'host=localhost port=5432 user=postgres password= dbname=thai' 
PUBLICATION test_pub WITH (copy_data = true);
```
17 мониторим статус и дожидаем окончания копирования
```
SELECT * FROM pg_stat_subscription \gx
```
Точное время не засекал, примерно полторы минуты
6 проверяем таблицы
```
\dt+ book.*

 Schema |     Name     | Type  |  Owner   | Persistence | Access method |    Size    | Description 
--------+--------------+-------+----------+-------------+---------------+------------+-------------
 book   | bus          | table | postgres | permanent   | heap          | 16 kB      | 
 book   | busroute     | table | postgres | permanent   | heap          | 8192 bytes | 
 book   | busstation   | table | postgres | permanent   | heap          | 16 kB      | 
 book   | fam          | table | postgres | permanent   | heap          | 16 kB      | 
 book   | nam          | table | postgres | permanent   | heap          | 16 kB      | 
 book   | ride         | table | postgres | permanent   | heap          | 64 MB      | 
 book   | schedule     | table | postgres | permanent   | heap          | 128 kB     | 
 book   | seat         | table | postgres | permanent   | heap          | 40 kB      | 
 book   | seatcategory | table | postgres | permanent   | heap          | 16 kB      | 
 book   | tickets      | table | postgres | permanent   | heap          | 4755 MB    | 
(10 rows)
```
16 удаляем подписку
```
psql -p 5433 -c "drop subscription test_sub;"
```
17 удаляем публикацию
```
psql -c "drop publication test_pub;"
```

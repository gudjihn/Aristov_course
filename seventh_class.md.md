# 7 задание Индексы 
ТЗ:

Развернуть ВМ (Linux) с PostgreSQL

Заливаем Тайские перевозки
https://github.com/aeuge/postgres16book/tree/main/database

Проверить скорость выполнения сложного запроса (приложен в коне файла скриптов)

Навесить индекс на внешние ключи

Проверить, помогли ли индексы на внешние ключи ускориться

---

# Запускаем explain

```sql
explain
WITH all_place AS (SELECT count(s.id) as all_place, s.fkbus as fkbus
                   FROM book.seat s
                   group by s.fkbus),
     order_place AS (SELECT count(t.id) as order_place, t.fkride
                     FROM book.tickets t
                     group by t.fkride)
SELECT r.id,
       r.startdate                as depart_date,
       bs.city || ', ' || bs.name as busstation,
       t.order_place,
       st.all_place
FROM book.ride r
         JOIN book.schedule as s
              on r.fkschedule = s.id
         JOIN book.busroute br
              on s.fkroute = br.id
         JOIN book.busstation bs
              on br.fkbusstationfrom = bs.id
         JOIN order_place t
              on t.fkride = r.id
         JOIN all_place st
              on r.fkbus = st.fkbus
GROUP BY r.id, r.startdate, bs.city || ', ' || bs.name, t.order_place, st.all_place
ORDER BY r.startdate
limit 10;
```

```
Limit  (cost=94777.85..94777.88 rows=10 width=56)
  ->  Sort  (cost=94777.85..94778.35 rows=200 width=56)
        Sort Key: r.startdate
        ->  Group  (cost=92412.58..94773.53 rows=200 width=56)
"              Group Key: r.id, (((bs.city || ', '::text) || bs.name)), (count(t.id)), (count(s_1.id))"
              ->  Incremental Sort  (cost=92412.58..94770.53 rows=200 width=56)
"                    Sort Key: r.id, (((bs.city || ', '::text) || bs.name)), (count(t.id)), (count(s_1.id))"
                    Presorted Key: r.id
                    ->  Nested Loop  (cost=92400.77..94761.53 rows=200 width=56)
                          Join Filter: (r.fkbus = s_1.fkbus)
                          ->  Nested Loop  (cost=92395.77..94156.00 rows=200 width=84)
                                Join Filter: (bs.id = br.fkbusstationfrom)
                                ->  Nested Loop  (cost=92395.77..94125.25 rows=200 width=24)
                                      Join Filter: (br.id = s.fkroute)
                                      ->  Nested Loop  (cost=92395.77..93944.37 rows=200 width=24)
                                            ->  Nested Loop  (cost=92395.49..93885.24 rows=200 width=24)
                                                  ->  Finalize GroupAggregate  (cost=92395.07..92445.74 rows=200 width=12)
                                                        Group Key: t.fkride
                                                        ->  Gather Merge  (cost=92395.07..92441.74 rows=400 width=12)
                                                              Workers Planned: 2
                                                              ->  Sort  (cost=91395.05..91395.55 rows=200 width=12)
                                                                    Sort Key: t.fkride
                                                                    ->  Partial HashAggregate  (cost=91385.41..91387.41 rows=200 width=12)
                                                                          Group Key: t.fkride
                                                                          ->  Parallel Seq Scan on tickets t  (cost=0.00..80582.27 rows=2160627 width=12)
                                                  ->  Index Scan using ride_pkey on ride r  (cost=0.42..7.20 rows=1 width=16)
                                                        Index Cond: (id = t.fkride)
                                            ->  Index Scan using schedule_pkey on schedule s  (cost=0.28..0.30 rows=1 width=8)
                                                  Index Cond: (id = r.fkschedule)
                                      ->  Materialize  (cost=0.00..1.90 rows=60 width=8)
                                            ->  Seq Scan on busroute br  (cost=0.00..1.60 rows=60 width=8)
                                ->  Materialize  (cost=0.00..1.15 rows=10 width=68)
                                      ->  Seq Scan on busstation bs  (cost=0.00..1.10 rows=10 width=68)
                          ->  Materialize  (cost=5.00..8.00 rows=200 width=12)
                                ->  HashAggregate  (cost=5.00..7.00 rows=200 width=12)
                                      Group Key: s_1.fkbus
                                      ->  Seq Scan on seat s_1  (cost=0.00..4.00 rows=200 width=8)
```

# Запускаем VACUUM

```sql 
VACUUM ANALYSE;
```

# Запускаем explain

```
Limit  (cost=332698.02..332698.05 rows=10 width=56)
  ->  Sort  (cost=332698.02..333066.36 rows=147336 width=56)
        Sort Key: r.startdate
        ->  Group  (cost=278053.92..329514.14 rows=147336 width=56)
"              Group Key: r.id, (((bs.city || ', '::text) || bs.name)), (count(t.id)), (count(s_1.id))"
              ->  Incremental Sort  (cost=278053.92..327304.10 rows=147336 width=56)
"                    Sort Key: r.id, (((bs.city || ', '::text) || bs.name)), (count(t.id)), (count(s_1.id))"
                    Presorted Key: r.id
                    ->  Merge Join  (cost=277815.70..318441.60 rows=147336 width=56)
                          Merge Cond: (r.id = t.fkride)
                          ->  Sort  (cost=20076.71..20436.71 rows=144000 width=32)
                                Sort Key: r.id
                                ->  Hash Join  (cost=52.09..4291.50 rows=144000 width=32)
                                      Hash Cond: (r.fkbus = s_1.fkbus)
                                      ->  Hash Join  (cost=46.98..3587.99 rows=144000 width=28)
                                            Hash Cond: (br.fkbusstationfrom = bs.id)
                                            ->  Hash Join  (cost=45.75..3048.56 rows=144000 width=16)
                                                  Hash Cond: (s.fkroute = br.id)
                                                  ->  Hash Join  (cost=43.40..2641.51 rows=144000 width=16)
                                                        Hash Cond: (r.fkschedule = s.id)
                                                        ->  Seq Scan on ride r  (cost=0.00..2219.00 rows=144000 width=16)
                                                        ->  Hash  (cost=25.40..25.40 rows=1440 width=8)
                                                              ->  Seq Scan on schedule s  (cost=0.00..25.40 rows=1440 width=8)
                                                  ->  Hash  (cost=1.60..1.60 rows=60 width=8)
                                                        ->  Seq Scan on busroute br  (cost=0.00..1.60 rows=60 width=8)
                                            ->  Hash  (cost=1.10..1.10 rows=10 width=20)
                                                  ->  Seq Scan on busstation bs  (cost=0.00..1.10 rows=10 width=20)
                                      ->  Hash  (cost=5.05..5.05 rows=5 width=12)
                                            ->  HashAggregate  (cost=5.00..5.05 rows=5 width=12)
                                                  Group Key: s_1.fkbus
                                                  ->  Seq Scan on seat s_1  (cost=0.00..4.00 rows=200 width=8)
                          ->  Finalize GroupAggregate  (cost=257738.99..295066.51 rows=147336 width=12)
                                Group Key: t.fkride
                                ->  Gather Merge  (cost=257738.99..292119.79 rows=294672 width=12)
                                      Workers Planned: 2
                                      ->  Sort  (cost=256738.97..257107.31 rows=147336 width=12)
                                            Sort Key: t.fkride
                                            ->  Partial HashAggregate  (cost=218997.81..241571.09 rows=147336 width=12)
                                                  Group Key: t.fkride
                                                  Planned Partitions: 4
                                                  ->  Parallel Seq Scan on tickets t  (cost=0.00..80582.32 rows=2160632 width=12)
```                                                  

# Добавляем индексы на внешние ключи

```sql 
CREATE INDEX fkschedule_idx ON book.ride (fkschedule);
CREATE INDEX fkroute_idx ON book.schedule (fkroute);
CREATE INDEX fkbusstationfrom_idx ON book.busroute (fkbusstationfrom);
CREATE INDEX fkride_idx ON book.tickets (fkride);
CREATE INDEX fkbus_idx ON book.seat (fkbus);
```

# Запускаем explain

```
Limit  (cost=330432.33..330432.35 rows=10 width=56)
  ->  Sort  (cost=330432.33..330789.49 rows=142865 width=56)
        Sort Key: r.startdate
        ->  Group  (cost=277456.17..327345.07 rows=142865 width=56)
"              Group Key: r.id, (((bs.city || ', '::text) || bs.name)), (count(t.id)), (count(s_1.id))"
              ->  Incremental Sort  (cost=277456.17..325202.09 rows=142865 width=56)
"                    Sort Key: r.id, (((bs.city || ', '::text) || bs.name)), (count(t.id)), (count(s_1.id))"
                    Presorted Key: r.id
                    ->  Merge Join  (cost=277225.24..316640.17 rows=142865 width=56)
                          Merge Cond: (r.id = t.fkride)
                          ->  Sort  (cost=20076.71..20436.71 rows=144000 width=32)
                                Sort Key: r.id
                                ->  Hash Join  (cost=52.09..4291.50 rows=144000 width=32)
                                      Hash Cond: (r.fkbus = s_1.fkbus)
                                      ->  Hash Join  (cost=46.98..3587.99 rows=144000 width=28)
                                            Hash Cond: (br.fkbusstationfrom = bs.id)
                                            ->  Hash Join  (cost=45.75..3048.56 rows=144000 width=16)
                                                  Hash Cond: (s.fkroute = br.id)
                                                  ->  Hash Join  (cost=43.40..2641.51 rows=144000 width=16)
                                                        Hash Cond: (r.fkschedule = s.id)
                                                        ->  Seq Scan on ride r  (cost=0.00..2219.00 rows=144000 width=16)
                                                        ->  Hash  (cost=25.40..25.40 rows=1440 width=8)
                                                              ->  Seq Scan on schedule s  (cost=0.00..25.40 rows=1440 width=8)
                                                  ->  Hash  (cost=1.60..1.60 rows=60 width=8)
                                                        ->  Seq Scan on busroute br  (cost=0.00..1.60 rows=60 width=8)
                                            ->  Hash  (cost=1.10..1.10 rows=10 width=20)
                                                  ->  Seq Scan on busstation bs  (cost=0.00..1.10 rows=10 width=20)
                                      ->  Hash  (cost=5.05..5.05 rows=5 width=12)
                                            ->  HashAggregate  (cost=5.00..5.05 rows=5 width=12)
                                                  Group Key: s_1.fkbus
                                                  ->  Seq Scan on seat s_1  (cost=0.00..4.00 rows=200 width=8)
                          ->  Finalize GroupAggregate  (cost=257148.53..293343.32 rows=142865 width=12)
                                Group Key: t.fkride
                                ->  Gather Merge  (cost=257148.53..290486.02 rows=285730 width=12)
                                      Workers Planned: 2
                                      ->  Sort  (cost=256148.50..256505.66 rows=142865 width=12)
                                            Sort Key: t.fkride
                                            ->  Partial HashAggregate  (cost=218944.99..241473.19 rows=142865 width=12)
                                                  Group Key: t.fkride
                                                  Planned Partitions: 4
                                                  ->  Parallel Seq Scan on tickets t  (cost=0.00..80531.94 rows=2160594 width=12)
```

# Итого

|                            |      cost |
|----------------------------|----------:|
| без индексов               |  94777.88 | 
| без индексов после вакуума | 332698.05 | 
| с индексами                | 330432.35 | 


```
 sql

SELECT
e.first_name
, e.last_name
, s.from_date
, s.amount
, coalesce(s.amount - lag(s.amount) OVER (PARTITION BY s.fk_employee ORDER BY s.from_date),0) AS amount_dif
  FROM salary s
  INNER JOIN employee e ON e.id = s.fk_employee
  ORDER BY s.fk_employee,s.from_date;

```
```
 first_name | last_name | from_date  | amount | amount_dif
------------+-----------+------------+--------+------------
 Eugene     | Aristov   | 2024-01-01 | 100000 |          0
 Eugene     | Aristov   | 2024-02-01 | 200000 |     100000
 Eugene     | Aristov   | 2024-03-01 | 300000 |     100000
 Ivan       | Ivanov    | 2023-01-01 | 200000 |          0
 Petr       | Petrov    | 2024-03-01 | 200000 |          0
(5 rows)
```

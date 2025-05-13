---

 sql

* SELECT e.first_name, e.last_name, s.from_date, s.amount, coalesce(s.amount - lag(s.amount) OVER (PARTITION BY s.fk_employee ORDER BY s.from_date),0) AS amount_dif FROM salary s JOIN employee e ON e.id = s.fk_employee ORDER BY s.fk_employee,s.from_date;

---

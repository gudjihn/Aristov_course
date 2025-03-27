#### *Отчет о выполнении Четвертого домашнего задания:*

---
* thai=# DROP TABLE IF EXISTS accounts;^C
* thai=# CREATE TABLE account(id integer, amount numeric);
* CREATE TABLE
* thai=# INSERT INTO account VALUES (1,2000.00), (2,2000.00), (3,2000.00);
* INSERT 0 3
* -- deadlock
-- 1 terminal
* thai=# begin;
* thai=*# UPDATE accounts SET amount = amount + 1 WHERE id = 1;
* UPDATE 1
* -- deadlock
-- 2 terminal
* thai=*# UPDATE accounts SET amount = amount + 1 WHERE id = 2;
* UPDATE 1
* -- 1
* UPDATE accounts SET amount = amount + 1 WHERE id = 2;
* -- 2
* UPDATE accounts SET amount = amount + 1 WHERE id = 1;

#### *logs - less +F /var/log/postgresql/postgresql.log*
```
* 2025-03-27 17:29:42 MSK [1649703-14] [local] postgres@thai [app: psql] ERROR:  deadlock detected
* 2025-03-27 17:29:42 MSK [1649703-15] [local] postgres@thai [app: psql] DETAIL:  Process 1649703 waits for ShareLock on transaction 1452; blocked by process 1647001.
      *  Process 1647001 waits for ShareLock on transaction 1453; blocked by process 1649703.
       * Process 1649703: UPDATE accounts SET amount = amount + 1 WHERE id = 1;
       * Process 1647001: UPDATE accounts SET amount = amount + 1 WHERE id = 2;
* 2025-03-27 17:29:42 MSK [1649703-16] [local] postgres@thai [app: psql] HINT:  See server log for query details.
* 2025-03-27 17:29:42 MSK [1649703-17] [local] postgres@thai [app: psql] CONTEXT:  while updating tuple (0,1) in relation "accounts"
* 2025-03-27 17:29:42 MSK [1649703-18] [local] postgres@thai [app: psql] STATEMENT:  UPDATE accounts SET amount = amount + 1 WHERE id = 1;
* 2025-03-27 17:29:42 MSK [1647001-19] [local] postgres@thai [app: psql] LOG:  process 1647001 acquired ShareLock on transaction 1453 after 7996.370 ms
* 2025-03-27 17:29:42 MSK [1647001-20] [local] postgres@thai [app: psql] CONTEXT:  while updating tuple (0,2) in relation "accounts"
* 2025-03-27 17:29:42 MSK [1647001-21] [local] postgres@thai [app: psql] STATEMENT:  UPDATE accounts SET amount = amount + 1 WHERE id = 2;
* 2025-03-27 17:29:42 MSK [1647001-22] [local] postgres@thai [app: psql] LOG:  duration: 7996.758 ms  statement: UPDATE accounts SET amount = amount + 1 WHERE id = 2;
```
* Конец домашнего задания


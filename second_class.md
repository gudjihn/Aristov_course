sudo -u postgres psql

begin;

BEGIN

=*# CREATE TABLE persons(id serial, first_name text, second_name text);

CREATE TABLE

-- сделать в первой сессии новую таблицу и наполнить ее данными
BEGIN;

CREATE TABLE people(id serial, first_name text, second_name text);

INSERT INTO people(first_name, second_name) VALUES ('dmitry','dmitryev');

INSERT INTO people(first_name, second_name) VALUES ('anna', 'ievleva');

COMMIT;

-- посмотреть текущий уровень изоляции:

show transaction isolation level;

 transaction_isolation
-----------------------
 read committed
(1 row)

-- начать новую транзакцию в обоих сессиях с дефолтным (не меняя) уровнем изоляции

BEGIN;

-- в первой сессии добавить новую запись

INSERT INTO people(first_name, second_name) VALUES ('elena','koroleva');

-- сделать запрос на выбор всех записей во второй сессии

SELECT * FROM people;

 id | first_name | second_name
 
----+------------+-------------

  1 | dmitry     | dmitryev

(1 row)

-- видите ли вы новую запись и если да то почему?

-- Запись не видно. Грязное чтение не допускается на уровне изоляции 
read committed

-- завершить транзакцию в первом окне

COMMIT;

-- сделать запрос во выбор всех записей второй сессии

SELECT * FROM people;

 id | first_name | second_name

----+------------+-------------

  1 | dmitry     | dmitryev
  
  2 | elena      | koroleva
  
(2 rows)

-- видите ли вы новую запись и если да то почему?

-- Запись видно, потому что мы завершили транзакцию в первой сессии и на уровне read committed возможно неповторяющееся чтение

-- завершите транзакцию во второй сессии

COMMIT;

-- начать новые транзакции, но уже на уровне repeatable read в ОБЕИХ сессиях

BEGIN transaction isolation level repeatable read;

-- в первой сессии добавить новую запись

INSERT INTO people(first_name, second_name) VALUES ('polina','tsvetkova');

-- сделать запрос на выбор всех записей во второй сессии

SELECT * FROM people;

 id | first_name | second_name

----+------------+-------------

  1 | dmitry     | dmitryev
  
  2 | elena      | koroleva

(2 rows)

-- видите ли вы новую запись и если да то почему?
-- !!! Нет, запись не видно. Грязное чтение не допускается на уровне изоляции repeatable read !!!

-- завершить транзакцию в первом окне

COMMIT;

-- сделать запрос во выбор всех записей второй сессии

SELECT * FROM people;

 id | first_name | second_name
 
----+------------+-------------

  1 | dmitry     | dmitryev

  2 | elena      | koroleva

  3 | polina     | tsvetkova

(3 rows)

-- видите ли вы новую запись и если да то почему?

-- !!! Нет, запись не видно. Неповторяющееся чтение не допускается на уровне изоляции repeatable read в Постгресе!!!
-- задание закончено

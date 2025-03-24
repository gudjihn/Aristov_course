#### *Это отчет по третьему  домашнему заданию*

> Создать таблицу с текстовым полем и заполнить случайными или сгенерированными данным в размере 1млн строк
  * **_sudo -u postgres psql_**
    * create table test_table(c1 text);
    * INSERT INTO test_table(c1) SELECT 'noname' FROM generate_series(1,1000000);


> Посмотреть размер файла с таблицей
 * **_select pg_size_pretty(pg_table_size('test_table'));_**
   * 35 MB 


> 5 раз обновить все строчки и добавить к каждой строчке любой символ
 * update test_table set c1=c1||'*';
 * update test_table set c1=c1||'#';
 * update test_table set c1=c1||'1';
 * update test_table set c1=c1||'*';
 * update test_table set c1=c1||'7';


> Посмотреть количество мертвых строчек в таблице и когда последний раз приходил автовакуум
 * **_SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 'test_table';_**
   * Мертвых строчек n_dead_tup нет т.к. avtovacuum быстрее нас :-) Но должно было быть примерно 5 млн. мертвых строк.
   * Автовакуум пришел в "2023-04-30 00:25:05.883613+00"
     * relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
     * ---------+------------+------------+--------+-------------------------------
     * test_table    |    1000000 |          0 |      0 | 2023-04-30 00:25:05.883613+00


> Подождать некоторое время, проверяя, пришел ли автовакуум
 * **_Он меня опередил, как писал выше_**


> 5 раз обновить все строчки и добавить к каждой строчке любой символ
 * update test_table set c1=c1||'9';
 * update test_table set c1=c1||'8';
 * update test_table set c1=c1||'7';
 * update test_table set c1=c1||'6';
 * update test_table set c1=c1||'5';


> Посмотреть размер файла с таблицей
 * **_select pg_size_pretty(pg_table_size('test_table'));_**
   * 253 MB 


> Отключить Автовакуум на конкретной таблице
 * **_alter table test_table set (autovacuum_enabled = off);_**


> 10 раз обновить все строчки и добавить к каждой строчке любой символ
 * update test_table set c1=c1||'1';
 * update test_table set c1=c1||'2';
 * update test_table set c1=c1||'3';
 * update test_table set c1=c1||'4';
 * update test_table set c1=c1||'5';
 * update test_table set c1=c1||'6';
 * update test_table set c1=c1||'7';
 * update test_table set c1=c1||'8';
 * update test_table set c1=c1||'9';
 * update test_table set c1=c1||'*';


> Посмотреть размер файла с таблицей
 * **_select pg_size_pretty(pg_table_size('test_table'));_**
   * 555 MB 


> Объясните полученный результат
 * **_Автовакуум выключен и мертвые строчки не удаляются. В итоге, у нас 10млн. мертвых строк. Но чтобы сжать файл на уровне ОС-надо выполнить vacuum full_**


> Не забудьте включить автовакуум)
 * **_alter table test_table set (autovacuum_enabled = on);_**


> Задание со *:
> Написать анонимную процедуру, в которой в цикле 10 раз обновятся все строчки в искомой таблице.
> Не забыть вывести номер шага цикла.
 * _DO_
 * _$do$_
 * _BEGIN_
   * _FOR i IN 1..10 LOOP_
     * _RAISE NOTICE 'Step = %', i;_
     * _update test_table set c1=c1||i;_
   * _END LOOP;_
 * _END_
 * _$do$;_


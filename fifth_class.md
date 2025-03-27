При использование "стандартных" селектов в pgbench самый продуктивный результат показал режим transaction

Например, можно использовать pool_mode=session и убивать "неактивные" сессии, а на их место "активные".

Все зависит от конкретной задачи и логики работы приложений. 
```
* Изначальный режим стоит session
postgres@ivan-psql-3:~$ pgbench -i ivan -p 6432
вывод :
latency average = 0.165 ms
initial connection time = 0.268 ms
tps = 6075.334143 (without initial connection time)
---
* Изменим режим работы pg_bouncer session -> transaction 
postgres@ivan-psql-3:~$ cat /etc/pgbouncer/pgbouncer.ini 
[databases]
ivan = host=127.0.0.1 port=5432 dbname=ivan pool_size=20 pool_mode=session --> transaction
* = host=127.0.0.1 port=5432

[pgbouncer] Перечитаем файл конфигурации для баунсера
root@ivan-psql-3:~# systemctl reload pgbouncer.service

* Начинаем новый тест -
postgres@ivan-psql-3:~$ pgbench  ivan -p 6432 -S
* вывод : 
latency average = 0.189 ms
initial connection time = 0.223 ms
tps = 5282.620180 (without initial connection time)
latency average = 0.281 ms
initial connection time = 0.235 ms
tps = 3554.923569 (without initial connection time)
---
Снова изменим режим баунсера на этот раз на --> statement
postgres@ivan-psql-3:~$ nano /etc/pgbouncer/pgbouncer.ini 
[databases]
ivan = host=127.0.0.1 port=5432 dbname=ivan pool_size=20 pool_mode=transaction --> statement
[pgbouncer]
* Перечитаем файл конфигурации для баунсера 
root@ivan-psql-3:~# systemctl reload pgbouncer.service
* Запустим тест
postgres@ivan-psql-3:~$ pgbench  ivan -p 6432 -S
latency average = 0.207 ms
initial connection time = 0.185 ms
tps = 4840.271055 (without initial connection time)
```
Конец задания

# Физическая и логическая репликация

# Практика №6

Развернуть асинхронную реплику (можно использовать 1 ВМ, просто рядом кластер развернуть и подключиться через
localhost): тестируем производительность по сравнению с сингл инстансом

Эталонного решения не будет, так как всё показано на лекции.

Задание со * переделать:

1. под синхронную реплику
2. синхронная реплика + асинхронная каскадно снимаемая с синхронной

Задание с **: переделать скрипты для ВМ или докера из моей репы с pg_rewind под 17 ПГ.
https://github.com/aeuge/pg_rewind

---
Создаем файл docker-compose с двумя контейнерами postgresql, двумя volume и сетью с типом bridge
hw06_phisical-and-logical-replication/docker-compose.yml

```
docker-compose up -d 
[+] Running 16/16
 ✔ postgresql_02 Pulled                                            14.9s 
   ✔ a470f59b418a Pull complete                                     7.2s 
   ✔ 84c0f1cd4d3f Pull complete                                     9.6s 
   ✔ 9b5d9b2e910a Pull complete                                     7.2s 
   ✔ f02d5343b38a Pull complete                                     7.2s 
   ✔ 0b5ffa07a5b5 Pull complete                                     7.2s 
   ✔ 02842c875e7d Pull complete                                     7.2s 
   ✔ 2e5f3b4e37b9 Pull complete                                     8.4s 
   ✔ b734d0ca7f33 Pull complete                                     7.8s 
   ✔ a4a7a5935213 Pull complete                                     7.2s 
   ✔ 7eae621c522a Pull complete                                     7.2s 
   ✔ d9b636547744 Pull complete                                     8.3s 
   ✔ 20adc9f7d9b1 Pull complete                                     7.2s 
   ✔ 333939793759 Pull complete                                     7.2s 
   ✔ 2a1dcfcecf35 Pull complete                                     7.2s 
 ✔ postgresql_01 Pulled                                            14.9s 
[+] Running 3/3
 ✔ Network hw06_phisical-and-logical-replication_postgres_network  Created                                                                                                       0.0s 
 ✔ Container postgresql_02                                         Started                                                                                                       0.3s 
 ✔ Container postgresql_01                                         Started
 
# создаем пользователя для репликации
docker exec -it postgresql_01 bash

# устанавливаем vim
apt update && apt install vim

su - postgres
createuser --replication -P repluser

# настраиваем postgresql.conf
vim /var/lib/postgresql/data/postgresql.conf
wal_level = replica
max_wal_senders = 2
max_replication_slots = 2
hot_standby = on

# добавляем разрешение на подключение в pg_hba.conf
# выводим список сетей докера
docker network ls
NETWORK ID     NAME                                                     DRIVER    SCOPE
75405d805329   hw06_phisical-and-logical-replication_postgres_network   bridge    local

# находим маску
docker network inspect hw06_phisical-and-logical-replication_postgres_network | grep Subnet
"Subnet": "172.18.0.0/16",

vim /var/lib/postgresql/data/pg_hba.conf
host replication all 172.18.0.0/16 md5

docker restart postgresql_01

# настраиваем реплику
docker exec -it postgresql_02 bash

rm -fr /var/lib/postgresql/data/*

su - postgres -c "pg_basebackup --host=postgresql_01 --username=repluser --pgdata=/var/lib/postgresql/data --wal-method=stream --write-recovery-conf"

docker-compose logs -f
postgresql_02  | 2025-03-30 04:58:25.151 UTC [27] LOG:  restartpoint complete: wrote 1 buffers (0.0%); 0 WAL file(s) added, 0 removed, 1 recycled; write=0.004 s, sync=0.001 s, total=0.013 s; sync files=0, longest=0.000 s, average=0.000 s; distance=16384 kB, estimate=16384 kB; lsn=0/20000168, redo lsn=0/20000110
postgresql_02  | 2025-03-30 04:58:25.151 UTC [27] LOG:  recovery restart point at 0/20000110
postgresql_02  | 2025-03-30 05:37:08.008 UTC [29] LOG:  invalid record length at 0/20000218: expected at least 24, got 0
postgresql_02  | 2025-03-30 05:37:08.023 UTC [42] LOG:  started streaming WAL from primary at 0/20000000 on timeline 1

# заходим в postgresql_01 и проверяем
SELECT * FROM pg_stat_replication \gx
sync_state       | async
```

# Тестируем

```
docker exec -it postgresql_01 bash

# нагрузка на запись
/workload2.sql << EOL
INSERT INTO book.tickets (fkRide, fio, contact, fkSeat)
VALUES (
ceil(random()*100)
, (array(SELECT fam FROM book.fam))[ceil(random()*110)]::text || ' ' ||
(array(SELECT nam FROM book.nam))[ceil(random()*110)]::text
,('{"phone":"+7' || (1000000000::bigint + floor(random()*9000000000)::bigint)::text || '"}')::jsonb
, ceil(random()*100));

EOL

su - postgres
pgbench -c 8 -j 4 -T 10 -f /workload2.sql -n -U postgres -p 5432 thai
pgbench (17.4 (Debian 17.4-1.pgdg120+2))
transaction type: /workload2.sql
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 192546
number of failed transactions: 0 (0.000%)
latency average = 0.415 ms
initial connection time = 6.331 ms
tps = 19263.971922 (without initial connection time)

# нагрузка на чтение 
cat > /workload.sql << EOL

\set r random(1, 5000000) 
SELECT id, fkRide, fio, contact, fkSeat FROM book.tickets WHERE id = :r;

EOL

su - postgres
pgbench -c 8 -j 4 -T 10 -f /workload.sql -n -U postgres -p 5432 thai
pgbench (17.4 (Debian 17.4-1.pgdg120+2))
transaction type: /workload.sql
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 680431
number of failed transactions: 0 (0.000%)
latency average = 0.118 ms
initial connection time = 7.293 ms
tps = 68084.618000 (without initial connection time)


# останавливаем реплику, заходим на мастера и запускаем pgbench
docker-compose stop postgresql_02

# на запись
pgbench -c 8 -j 4 -T 10 -f /workload2.sql -n -U postgres -p 5432 thai
pgbench (17.4 (Debian 17.4-1.pgdg120+2))
transaction type: /workload2.sql
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 228390
number of failed transactions: 0 (0.000%)
latency average = 0.350 ms
initial connection time = 7.911 ms
tps = 22851.380878 (without initial connection time)

# на чтение
pgbench -c 8 -j 4 -T 10 -f /workload.sql -n -U postgres -p 5432 thai
pgbench (17.4 (Debian 17.4-1.pgdg120+2))
transaction type: /workload.sql
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 713582
number of failed transactions: 0 (0.000%)
latency average = 0.112 ms
initial connection time = 7.033 ms
tps = 71401.354979 (without initial connection time)

```

# Итого

| replica (tps) |           on |           off |
|---------------|-------------:|--------------:| 
| на чтение     | 68084.618000 |  71401.354979 | 
| на запись     | 19263.971922 |  22851.380878 | 



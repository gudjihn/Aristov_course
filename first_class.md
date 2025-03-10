Развернуто ВМ через сервис wb_cloud 

Тайские перевозки: 

wget https://storage.googleapis.com/thaibus/thai_medium.tar.gz && tar -xf thai_medium.tar.gz && psql < thai.sql

postgres=# \c thai

You are now connected to database "thai" as user "postgres".

thai=#  select count(*) from book.tickets;
  count
----------
 53997475
(1 row)

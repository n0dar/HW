>**создать новый проект в** Google Cloud Platform, например postgres2022-, где yyyymm год, месяц вашего рождения (имя проекта должно быть уникально на уровне GCP), Яндекс облако или на любых ВМ, **докере**

Docker Engine
v20.10.22

> поставить PostgreSQL

docker run --name myPostgres -e POSTGRES_PASSWORD=postgres -d postgres

> зайти удаленным ssh (первая сессия)  
> зайти вторым ssh (вторая сессия)  
> запустить везде psql из под пользователя postgres  

docker exec -it myPostgres psql -U postgres  
psql (15.1 (Debian 15.1-1.pgdg110+1))  
Type "help" for help.  
postgres=#  

postgres=# CREATE DATABASE testdb;  
CREATE DATABASE  

postgres=# \c testdb  
You are now connected to database "testdb" as user "postgres".  
testdb=# 

>выключить auto commit

testdb=# \set AUTOCOMMIT OFF  
testdb=#

>сделать в первой сессии новую таблицу и наполнить ее данными 

testdb=# create table persons(id serial, first_name text, second_name text); insert into persons(first_name, second_name) values('ivan', 'ivanov'); insert into persons(first_name, second_name) values('petr', 'petrov'); commit;  
CREATE TABLE  
INSERT 0 1  
INSERT 0 1  
COMMIT  
testdb=#

>посмотреть текущий уровень изоляции  

testdb=# show transaction isolation level; commit;  
 transaction_isolation  
 \-----------------------  
 read committed  
(1 row)  

COMMIT  
testdb=#  

>начать новую транзакцию в обоих сессиях с дефолтным (не меняя) уровнем изоляции
>в первой сессии добавить новую запись

testdb=# insert into persons(first_name, second_name) values('sergey', 'sergeev');  
INSERT 0 1  
testdb=*#  

>сделать select * from persons во второй сессии

testdb=\*# select * from persons;  
 id | first_name | second_name  
\----+------------+-------------  
  1 | ivan       | ivanov      
  2 | petr       | petrov      
(2 rows)

testdb=\*#

>## **Q1:** видите ли вы новую запись и если да то почему?

## **A1:** не вижу, потому что уровень изоляции read committed, установленный по умолчанию для Т2 не позволяет в ней прочитать запись, которая была добалвенна в еще не завершенной Т1. Пока Т1 не завершилась, считается, что запись не существует для Т2 ("грязное" чтение запрещено)

>завершить первую транзакцию 

testdb=\*# commit;  
COMMIT  
testdb=#  

>сделать select * from persons во второй сессии

testdb=*# select * from persons;  
 id | first_name | second_name  
\----+------------+-------------  
  1 | ivan       | ivanov  
  2 | petr       | petrov  
  3 | sergey     | sergeev  
(3 rows)  

testdb=*#  

>## **Q2:** видите ли вы новую запись и если да то почему?

## **A2:** вижу, так как Т1, в рамках которой добавлялась новая запись, была "закомичена", после этого запись стала доступна в Т2, в которой установлен уровень изоляции read committed

>завершите транзакцию во второй сессии

testdb=\*# commit;  
COMMIT  
testdb=#  

>начать новые но уже repeatable read транзации 

testdb=# set transaction isolation level repeatable read;  
SET  
testdb=\*#  

>в первой сессии добавить новую запись

testdb=\*# insert into persons(first_name, second_name) values('sveta', 'svetova');  
INSERT 0 1  
testdb=\*#  

>сделать select * from persons во второй сессии

testdb=~*# select * from persons;  
 id | first_name | second_name  
\----+------------+-------------  
  1 | ivan       | ivanov  
  2 | petr       | petrov  
  3 | sergey     | sergeev  
(3 rows)  

testdb=\*#  

>## **Q3:** видите ли вы новую запись и если да то почему?

## **A3:** при уровне излояции repeatable read "грязное" чтение запрещено также как и при read committed. Запись не видна по тем же причинам, что и в ответе на вопрос **Q1**

>завершить первую транзакцию

testdb=*# commit;  
COMMIT  
testdb=#  

>сделать select * from persons во второй сессии

testdb=*# SELECT * FROM persons;  
 id | first_name | second_name     
\----+------------+-------------  
  1 | ivan       | ivanov  
  2 | petr       | petrov  
  3 | sergey     | sergeev  
(3 rows)  

>## **Q4:** видите ли вы новую запись и если да то почему?

## **A4:** не вижу. При уровне изоляции repeatable read, Т2 не увидит запись, добавленную в Т1 (даже если она завершится), так как Т1 началась до Т2, и в момент начала Т1 такой записи в таблице не было (фантомное чтение запрещено)

>завершить вторую транзакцию

testdb=*# commit;  
COMMIT  
testdb=#  

>сделать select * from persons во второй сессии

testdb=# select * from persons;  
 id | first_name | second_name  
\----+------------+-------------  
  1 | ivan       | ivanov  
  2 | petr       | petrov  
  3 | sergey     | sergeev  
  4 | sveta      | svetova  
(4 rows) 

>## **Q5:** видите ли вы новую запись и если да то почему?

## **A5:** вижу. Потому что запрос выполняется в Т3. А запись была добавлена в Т1, которая завершилась до открытия Т3. 

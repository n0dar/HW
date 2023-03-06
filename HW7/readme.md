>1 создайте новый кластер PostgresSQL 14

`docker pull postgres:14`  
`docker run --name myPostgres -e POSTGRES_PASSWORD=123 -d postgres:14`

>2 зайдите в созданный кластер под пользователем postgres

`docker exec -it myPostgres psql -U postgres`

>3 создайте новую базу данных testdb

`CREATE DATABASE testdb;`

>4 зайдите в созданную базу данных под пользователем postgres

`\c testdb`  

You are now connected to database "testdb" as user "postgres".  
testdb=#

>5 создайте новую схему testnm

`CREATE SCHEMA testnm;`

>6 создайте новую таблицу t1 с одной колонкой c1 типа integer

`CREATE TABLE t1 (c1 INTEGER);`

>7 вставьте строку со значением c1=1

`INSERT INTO t1 SELECT 1;`

>8 создайте новую роль readonly

`CREATE ROLE readonly;`

>9 дайте новой роли право на подключение к базе данных testdb

`GRANT CONNECT ON DATABASE testdb TO readonly;`

>10 дайте новой роли право на использование схемы testnm

`GRANT USAGE ON SCHEMA testnm TO readonly;`

>11 дайте новой роли право на select для всех таблиц схемы testnm

`GRANT SELECT ON ALL TABLES IN SCHEMA testnm TO readonly;`

>12 создайте пользователя testread с паролем test123

`CREATE ROLE testread WITH LOGIN PASSWORD 'test123';`

>13 дайте роль readonly пользователю testread

`GRANT readonly TO testread;`

>14 зайдите под пользователем testread в базу данных testdb

`\q`   
`docker exec -it myPostgres psql -d testdb -U testread`

>15 сделайте select * from t1;  
>16 получилось? (могло если вы делали сами не по шпаргалке и не упустили один существенный момент про который позже)

ERROR:  permission denied for table 1

>17 напишите что именно произошло в тексте домашнего задания

понятия не имею, что у вас произошло в тексте ДЗ. Вижу ДЗ, выполняю его по пунктам. В чем прикол вопроса?

>18 у вас есть идеи почему? ведь права то дали?

гадать не будем

`\q`  
`docker exec -it myPostgres psql -U postgres`  
`CREATE DATABASE testdb2;`  
`\c testdb2`  
`CREATE SCHEMA testnm;`  
`SELECT current_schema;`  

current_schema   
\----------------  
 public  
(1 row)  

`SHOW search_path;`

 search_path   
\-----------------  
 "$user", public  
(1 row)  

соответственно, последующий CREATE TABLE без явного указания схемы приведет к созданию таблицы в первой найденной существующей схеме, в данном случае — в public (что и получилось ранее в testdb). В testdb пользователь testread получил доступы путем включения в роль readonly. Роль readonly имеет права на подключение к testdb и на чтение всех таблиц в схеме testnm. Таблица t1 создана в схеме public, но ее владельцем является postgres, поэтому другие роли/пользователи не имеют к ней доступа, если он не был выдан им явно, как например, наш testread не имеет доступа public.t1 (через роль readonly в том числе)

>19 посмотрите на список таблиц

`\c testdb testread`  
`\dt`  
        List of relations  
 Schema | Name | Type  |  Owner  
--------+------+-------+----------  
 public | t1   | table | postgres  

 >20 подсказка в шпаргалке под пунктом 20

 >21 а почему так получилось с таблицей (если делали сами и без шпаргалки то может у вас все нормально)
 
 в ДЗ ничего не предвещало, что таблицу нужно было создать в схеме testnm. Ну или что readonly нужно было давать доступ к объектам в схеме public. Получилось ровно то, что и должно было получиться при точном выполнении пунктов ДЗ.

 >22 вернитесь в базу данных testdb под пользователем postgres

`\c testdb postgres`  

 >23 удалите таблицу t1

 `DROP TABLE t1;`

 >24 создайте ее заново но уже с явным указанием имени схемы testnm

 `CREATE TABLE testnm.t1 (c1 INTEGER);`

 >25 вставьте строку со значением c1=1

 `INSERT INTO testnm.t1 SELECT 1;`

>26 зайдите под пользователем testread в базу данных testdb

`\c testdb testread`  

>27 сделайте select * from testnm.t1;

>28 получилось?

ERROR:  permission denied for table t1

>29 есть идеи почему? если нет - смотрите шпаргалку

мы давали доступ GRANT SELECT ON ALL TABLES IN SCHEMA testnm TO readonly. Потом мы удалили таблицу testnm.t1 и создали таблицу с таким же именнем в той же схеме. Это совсем другая таблица, и от того что у нее такие же схема и имя, как и у той, к которой до удаления был доступ, доступ к вновь созданной таблице сам собой не появится. Пробуем

`\c testdb postgres`  
`GRANT SELECT ON ALL TABLES IN SCHEMA testnm TO readonly;`  
`\c testdb testread`  
`SELECT * FROM testnm.t1;`  

 c1  
\----  
  1  
(1 row)  

>30 как сделать так чтобы такое больше не повторялось? если нет идей - смотрите шпаргалку

Здесь я посмотрел шпарналку, так как не был уверен в ответе. Тем не мнее, я все равно всегда бы создавал (и создаю) объекты с явным указанием схемы, чтобы просто вообще не думать обо всех этих премудростях — почему и в какую схему, вдруг, убежала таблица

>31 сделайте select * from testnm.t1;  
>32 получилось?  
>33 есть идеи почему? если нет - смотрите шпаргалку  
>31 сделайте select * from testnm.t1;  
>32 получилось?  
>33 ура! 

не понял, что здесь должно было происходить, но ура, так ура=) Но п.п. 31-33 тут как-то задвоены

>34 теперь попробуйте выполнить команду create table t2(c1 integer); insert into t2 values (2);

>35 а как так? нам же никто прав на создание таблиц и insert в них под ролью readonly?

`\dt`  
        List of relations  
 Schema | Name | Type  |  Owner  
\--------+------+-------+----------  
 public | t2   | table | testread  

 пользователи имеют права в схеме public по умолчанию. testread создал в ней таблицу, а потому может ее и читать

>36 есть идеи как убрать эти права? если нет - смотрите шпаргалку

`\c testdb postgres`  
`REVOKE CREATE ON SCHEMA public FROM public;`  
`\c testdb testread;`  
`CREATE TABLE public.t3(f CHAR(1));`  
ERROR:  permission denied for schema public

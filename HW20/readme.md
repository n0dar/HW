>Секционировать большую таблицу из демо базы flights  

разворачиваю Postgres в контейнер, запускаю скрипт создания БД      
```
docker run --name pg -w /tmp/demo_medium -v c:\demo_medium:/tmp/demo_medium -e POSTGRES_PASSWORD=1 -d postgres:15
docker exec -t -i pg bash
passwd postgres  
login postgres  
psql -f /tmp/demo_medium/demo_medium.sql  
psql  
\c demo
```
самая большая таблица в БД
```sql
SELECT 
    relname 
FROM 
    pg_class 
WHERE 
    relnamespace=(SELECT oid FROM pg_namespace WHERE nspname='bookings') 
ORDER BY 
    pg_relation_size(oid) DESC LIMIT 1;
```  
ticket_flights  

изучаю структуру таблицы
```
\d bookings.ticket_flights;
```
```
                     Table "bookings.ticket_flights"
     Column      |         Type          | Collation | Nullable | Default
-----------------+-----------------------+-----------+----------+---------
 ticket_no       | character(13)         |           | not null |
 flight_id       | integer               |           | not null |
 fare_conditions | character varying(10) |           | not null |
 amount          | numeric(10,2)         |           | not null |
Indexes:
    "ticket_flights_pkey" PRIMARY KEY, btree (ticket_no, flight_id)
Check constraints:
    "ticket_flights_amount_check" CHECK (amount >= 0::numeric)
    "ticket_flights_fare_conditions_check" CHECK (fare_conditions::text = ANY (ARRAY['Economy'::character varying::text, 'Comfort'::character varying::text, 'Business'::character varying::text]))
Foreign-key constraints:
    "ticket_flights_flight_id_fkey" FOREIGN KEY (flight_id) REFERENCES bookings.flights(flight_id)
    "ticket_flights_ticket_no_fkey" FOREIGN KEY (ticket_no) REFERENCES bookings.tickets(ticket_no)
Referenced by:
    TABLE "bookings.boarding_passes" CONSTRAINT "boarding_passes_ticket_no_fkey" FOREIGN KEY (ticket_no, flight_id) REFERENCES bookings.ticket_flights(ticket_no, flight_id)
```
буду секционировать таблицу по flight_id
```sql 
SELECT MIN(flight_id), MAX(flight_id), COUNT(DISTINCT flight_id) FROM bookings.ticket_flights;
``` 
 min |  max  | count 
-----|-------|-------
   1 | 65664 | 45174  

создам дефолтную секцию и еще две для диапазонов flight_id 1-35000 и 35001-70000

создаю новую секционированную таблицу и секции
```sql 
CREATE TABLE bookings.ticket_flights2(LIKE bookings.ticket_flights INCLUDING ALL) PARTITION BY RANGE(flight_id);
CREATE TABLE bookings.ticket_flights_part1 PARTITION OF bookings.ticket_flights2 FOR VALUES FROM (1) TO (35000);
CREATE TABLE bookings.ticket_flights_part2 PARTITION OF bookings.ticket_flights2 FOR VALUES FROM (35001) TO (70000);
CREATE TABLE bookings.ticket_flights_part_def PARTITION OF bookings.ticket_flights2 default;
```
переношу записи из bookings.ticket_flights в bookings.ticket_flights2 и удаляю bookings.ticket_flights. Можно было бы и по-другому — не удалять bookings.ticket_flights, а аттачить ее как вторую секцию таблицы bookings.ticket_flights2, предварительно перенеся из нее записи с flight_id 1-35000 в первую секцию. Но допустим, что проблем с производительностью и дисковым пространством у меня нет, потому решаю задачу "в лоб"  

```sql 
INSERT INTO bookings.ticket_flights2 SELECT * FROM bookings.ticket_flights;
DROP TABLE bookings.ticket_flights CASCADE;
```
переименовываю bookings.ticket_flights2 в bookings.ticket_flights
```sql 
ALTER TABLE bookings.ticket_flights2 RENAME TO ticket_flights; 
```
изучаю стуктуру таблиы  
```
\d+ bookings.ticket_flights;
``` 
``` 
Partitioned table "bookings.ticket_flights"
     Column      |         Type          | Collation | Nullable | Default | Storage  | Compression | Stats target |     Description 

-----------------+-----------------------+-----------+----------+---------+----------+-------------+--------------+---------------------
 ticket_no       | character(13)         |           | not null |         | extended |             |              | Номер билета    
 flight_id       | integer               |           | not null |         | plain    |             |              | Идентификатор рейса
 fare_conditions | character varying(10) |           | not null |         | extended |             |              | Класс обслуживания
 amount          | numeric(10,2)         |           | not null |         | main     |             |              | Стоимость перелета
Partition key: RANGE (flight_id)
Indexes:
    "ticket_flights2_pkey" PRIMARY KEY, btree (ticket_no, flight_id)
Check constraints:
    "ticket_flights_amount_check" CHECK (amount >= 0::numeric)
    "ticket_flights_fare_conditions_check" CHECK (fare_conditions::text = ANY (ARRAY['Economy'::character varying::text, 'Comfort'::character varying::text, 'Business'::character varying::text]))
Partitions: bookings.ticket_flights_part1 FOR VALUES FROM (1) TO (35000),
            bookings.ticket_flights_part2 FOR VALUES FROM (35001) TO (70000),
            bookings.ticket_flights_part_def DEFAULT
``` 
все секции на месте  

по-хорошему нужно и ссылочную целостность восстановить как была, вот эту
``` 
Foreign-key constraints:
    "ticket_flights_flight_id_fkey" FOREIGN KEY (flight_id) REFERENCES bookings.flights(flight_id)
    "ticket_flights_ticket_no_fkey" FOREIGN KEY (ticket_no) REFERENCES bookings.tickets(ticket_no)
Referenced by:
    TABLE "bookings.boarding_passes" CONSTRAINT "boarding_passes_ticket_no_fkey" FOREIGN KEY (ticket_no, flight_id) REFERENCES bookings.ticket_flights(ticket_no, flight_id)
``` 
```sql
ALTER TABLE bookings.ticket_flights ADD CONSTRAINT ticket_flights_flight_id_fkey FOREIGN KEY (flight_id) REFERENCES bookings.flights(flight_id);
ALTER TABLE bookings.ticket_flights ADD CONSTRAINT ticket_flights_ticket_no_fkey FOREIGN KEY (ticket_no) REFERENCES bookings.tickets(ticket_no);
ALTER TABLE bookings.boarding_passes ADD CONSTRAINT boarding_passes_ticket_no_fkey FOREIGN KEY (ticket_no, flight_id) REFERENCES bookings.ticket_flights(ticket_no, flight_id);
``` 

проверяем
```sql
EXPLAIN SELECT * FROM bookings.ticket_flights;
```
```
                                              QUERY PLAN
------------------------------------------------------------------------------------------------------
 Append  (cost=0.00..55094.51 rows=2360901 width=32)
   ->  Seq Scan on ticket_flights_part1 ticket_flights_1  (cost=0.00..27903.95 rows=1521995 width=32) 
   ->  Seq Scan on ticket_flights_part2 ticket_flights_2  (cost=0.00..15370.36 rows=838336 width=32)
   ->  Seq Scan on ticket_flights_part_def ticket_flights_3  (cost=0.00..15.70 rows=570 width=114)
(4 rows)
```
```sql
EXPLAIN SELECT * FROM bookings.ticket_flights WHERE flight_id=1;
```
```
                                               QUERY PLAN
--------------------------------------------------------------------------------------------------------
 Gather  (cost=1000.00..21619.66 rows=86 width=32)
   Workers Planned: 2
   ->  Parallel Seq Scan on ticket_flights_part1 ticket_flights  (cost=0.00..20611.06 rows=36 width=32)
         Filter: (flight_id = 1)
```

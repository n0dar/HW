`docker run --name pg -e POSTGRES_PASSWORD=1 -d postgres:15`  
`docker exec -t -i pg bash`  
`passwd postgres`  
`login postgres`  
`psql`  

>**1 вариант**

>1 Создать индекс к какой-либо из таблиц вашей БД

`CREATE TABLE t AS SELECT (random()*10000)::int a, (random()*10000)::int b FROM generate_series(1, 10000);`   
`CREATE INDEX i1 ON t(a);`

>2 Прислать текстом результат команды explain, в которой используется данный индекс  

`EXPLAIN SELECT a FROM t ORDER BY a;`  

Index Only Scan using i1 on t  (cost=0.29..254.28 rows=10000 width=4)  

>3 Реализовать индекс для полнотекстового поиска  

`CREATE TABLE fti(doc text, tsv tsvector);`

`INSERT INTO fti(doc) VALUES ($$Nearly ten years had passed since the Dursleys had woken up to find their nephew on the front step, but Privet Drive had hardly changed at all. The sun rose on the same tidy front gardens and lit up the brass number four on the Dursleys' front door; it crept into their living room, which was almost exactly the same as it had been on the night when Mr. Dursley had seen that fateful news report about the owls. Only the photographs on the mantelpiece really showed how much time had passed. Ten years ago, there had been lots of pictures of what looked like a large pink beach ball wearing different-colored bonnets - but Dudley Dursley was no longer a baby, and now the photographs showed a large blond boy riding his first bicycle, on a carousel at the fair, playing a computer game with his father, being hugged and kissed by his mother. The room held no sign at all that another boy lived in the house, too.$$),  ($$Yet Harry Potter was still there, asleep at the moment, but not for long. His Aunt Petunia was awake and it was her shrill voice that made the first noise of the day. "Up! Get up! Now!" Harry woke with a start. His aunt rapped on the door again. "Up!" she screeched. Harry heard her walking toward the kitchen and then the sound of the frying pan being put on the stove. He rolled onto his back and tried to remember the dream he had been having. It had been a good one. There had been a flying motorcycle in it. He had a funny feeling he'd had the same dream before. His aunt was back outside the door. "Are you up yet?" she demanded. "Nearly," said Harry. "Well, get a move on, I want you to look after the bacon. And don't you dare let it burn, I want everything perfect on Duddy's birthday." Harry groaned. "What did you say?" his aunt snapped through the door. "Nothing, nothing . . ."$$),  ($$Dudley's birthday - how could he have forgotten? Harry got slowly out of bed and started looking for socks. He found a pair under his bed and, after pulling a spider off one of them, put them on. Harry was used to spiders, because the cupboard under the stairs was full of them, and that was where he slept. When he was dressed he went down the hall into the kitchen. The table was almost hidden beneath all Dudley's birthday presents. It looked as though Dudley had gotten the new computer he wanted, not to mention the second television and the racing bike. Exactly why Dudley wanted a racing bike was a mystery to Harry, as Dudley was very fat and hated exercise - unless of course it involved punching somebody. Dudley's favorite punching bag was Harry, but he couldn't often catch him. Harry didn't look it, but he was very fast.$$);` 

`UPDATE fti SET tsv = to_tsvector(doc);`  
`CREATE INDEX ii ON fti USING GIN(tsv);`  
`EXPLAIN SELECT doc FROM fti WHERE tsv @@ to_tsquery('Dudley & Harry');`

 Seq Scan on fti  (cost=0.00..2.79 rows=1 width=32)  
   Filter: (tsv @@ to_tsquery('Dudley & Harry'::text))  

не хочет. Тогда еще набьем таблицу данными, чтоб захотел  

`INSERT INTO fti SELECT doc, tsv FROM fti CROSS JOIN (SELECT 1 FROM generate_series(1,100)) t;`  
`EXPLAIN SELECT doc FROM fti WHERE tsv @@ to_tsquery('Dudley & Harry');`  

 Bitmap Heap Scan on fti  (cost=21.29..135.86 rows=135 width=882)  
   Recheck Cond: (tsv @@ to_tsquery('Dudley & Harry'::text))  
   ->  Bitmap Index Scan on ii  (cost=0.00..21.26 rows=135 width=0)  
         Index Cond: (tsv @@ to_tsquery('Dudley & Harry'::text))  

>4 Реализовать индекс на часть таблицы  

`CREATE INDEX i3 ON t(b) INCLUDE(a) WHERE b=a;`

`EXPLAIN SELECT * FROM t WHERE b=1;`

 Seq Scan on t  (cost=0.00..170.00 rows=2 width=8)  
   Filter: (b = 1)  

`EXPLAIN SELECT * FROM t WHERE b=a;`

 Index Only Scan using i3 on t  (cost=0.12..4.63 rows=50 width=8)  


>4 или индекс на поле с функцией

`CREATE INDEX i3_1 ON t(power(a,2));`  
`EXPLAIN SELECT *  FROM t ORDER BY power(a,2);`  

Index Scan using i3_1 on t  (cost=0.29..500.29 rows=10000 width=16)

>5 Создать индекс на несколько полей 

`CREATE INDEX i4 ON t(a,b);`    
`EXPLAIN SELECT b FROM t WHERE a>0;`  
 Seq Scan on t  (cost=0.00..170.00 rows=9998 width=4)  
   Filter: (a > 1)  

под условие фильтра попадают 9998 из 10000 записей таблицы, планировщик не видит выгоды от испольования индекса. А вот так увидит    

`EXPLAIN SELECT b FROM t WHERE a>9000;`

 Index Only Scan using i4 on t  (cost=0.29..34.40 rows=1035 width=4)  
   Index Cond: (a > 9000)  

>6 Написать комментарии к каждому из индексов  
>7 Описать что и как делали и с какими проблемами столкнулись

Не знаю, что именно комментировать, по запросам и (предполагаемым) планам вроде все понятно. 
С трудностями тоже не столкнулся, изучение допматериалов не в счет

>**2 вариант**

>7 К работе приложить структуру таблиц, для которых выполнялись соединения  

`CREATE TABLE l AS SELECT l FROM  (VALUES (1),(2),(3)) t(l);`   
`CREATE TABLE r AS SELECT r FROM  (VALUES (1),(3),(5)) t(r);`  

`SELECT * FROM l;`  

|l |
|-----|
|    1|
|    2|
|    3|

`SELECT * FROM r;`

|r |
|-----|
|    1|
|    3|
|    5|

>1 Реализовать прямое соединение двух таблиц

`SELECT * FROM l INNER JOIN r ON l.l=r.r;`    
| l | r| 
|--|---|
| 1 | 1|
| 3 | 3|

>2 Реализовать левостороннее (или правостороннее) соединение двух таблиц

`SELECT * FROM l LEFT JOIN r ON l.l=r.r;`  
| l | r |
|--|---|
| 1 | 1|
| 2 ||
| 3 | 3|

`SELECT * FROM l RIGHT JOIN r ON l.l=r.r;`  
| l | r|
|--|---|
| 1 | 1|
| 3 | 3|
|| 5 |

>3 Реализовать кросс соединение двух или более таблиц

`SELECT * FROM l, r LIMIT 3;`  
`SELECT * FROM l CROSS JOIN r LIMIT 3;`

 l | r 
---|---
 1 | 1
 1 | 3
 1 | 5

>4 Реализовать полное соединение двух или более таблиц

`SELECT * FROM l FULL JOIN r ON l.l=r.r LIMIT 3;`
 
 l | r 
---|---
 1 | 1
 2 |
 3 | 3

>5 Реализовать запрос, в котором будут использованы разные типы соединений

`SELECT l1.l l1, l2.l l2, r.r FROM l l1 INNER JOIN l l2 ON l1.l<>l2.l LEFT JOIN r ON l1.l=r.r;`  

 l1 | l2 | r 
----|----|---
  1 |  2 | 1
  1 |  3 | 1
  2 |  1 |
  2 |  3 |
  3 |  1 | 3
  3 |  2 | 3

>6 Сделать комментарии на каждый запрос

Прокомментирую не конкретные запросы, а как я понимаю механику соединнений в принципе

При соединение таблиц итоговая выборка состоит из записей, атрибуты которых включают все атрибуты соединенных записей. При соединении каждая запись таблицы сопостовляется с записью другой таблицы. Соединение записей выполняется (или не выполняется) по результатам проверки выражения, следующего за ON. Если выражение истинно или оно не предусмотрено, то сопоставляемые записи считаются связанными. Если в процессе сопоставления для записи таблицы не определена связанная запись другой таблицы, а результат такого связывания все равно должен попасть в итоговую выборку, то вместо атрибутов записи другой таблицы в итоговую выборку выводятся значения NULL.   

В результирующую выборку попадают:  
при INNER JOIN — записи из обоих таблиц, для которых выражение истинно    
при LEFT JOIN — все записи из левой таблицы и только те записи из правой таблицы, для которых выражение истинно  
при RIGHT JOIN — все записи из правой таблицы и только те записи из левой таблицы, для которых выражение истинно  
при CROSS JOIN — все возможные пары записей из левой и правой таблиц (полный перебор, декартово произведение)  
при FULL JOIN — {LEFT JOIN} UINON {INNER JOIN} UNOIN {RIGHT JOIN}    


>Задание со звездочкой*
>Придумайте 3 своих метрики на основе показанных представлений, отправьте их через ЛК, а так же поделитесь с коллегами в слаке

Количество простаивающих серверных процессов  

`SELECT COUNT(1) FROM pg_stat_activity WHERE state LIKE 'idle%';`  

Сколько выполняется самый старый из выполняющихся в моменте запросов  

`SELECT MAX(NOW() - query_start) FROM pg_stat_activity WHERE state='active';`  

Самый часто выполняемый запрос (оператор)

`SELECT query, calls FROM pg_stat_statements ORDER BY calls DESC LIMIT 1;`

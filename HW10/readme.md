`docker pull postgres:14`  
`docker run --name pg -v c:\var\lib\postgresql\data:/var/lib/postgresql/data -e POSTGRES_PASSWORD=1 -d postgres:14`  
`docker exec -t -i pg bash`   
`passwd postgres`    
`login postgres`   

>Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд.

`psql`  
`ALTER SYSTEM SET logging_collector=on;` — включаем сборщик сообщений  
`ALTER SYSTEM SET deadlock_timeout=200;` — время ожидания блокировки, по истечении которого будет выполняться проверка состояния взаимоблокировки и/или спустя какое время в журнал сервера будут записываться сообщения об ожидании блокировки  
`ALTER SYSTEM SET log_lock_waits=on;` — чтобы фиксировать в журнале события, когда сеанс ожидает получения блокировки дольше, чем указано в deadlock_timeout  
`\q`  
`logout`  
`exit`  
`docker restart pg`  
`docker exec -t -i pg bash`  
`login postgres`   
`psql`

>Воспроизведите ситуацию, при которой в журнале появятся такие сообщения.

 В первой сессии 

`CREATE TABLE t(f INT);`  
`\set AUTOCOMMIT OFF`  
`LOCK t;`  

Во второй сесии   

`docker exec -t -i pg psql -U postgres`  
`SELECT * FROM t;`  

В журнале 

2023-03-25 20:05:06.827 UTC [99] LOG:  process 99 still waiting for AccessShareLock on relation 16387 of database 13757 after 200.291 ms at character 15  
2023-03-25 20:05:06.827 UTC [99] DETAIL:  Process holding the lock: 48. Wait queue: 99.  
2023-03-25 20:05:06.827 UTC [99] STATEMENT:  SELECT * FROM t;  

>Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны. Пришлите список блокировок и объясните, что значит каждая.

По ходу описания я несколько раз привожу таблицу, полученную в отдельной сесиии вот этим запросом, и комментирую ее  
`SELECT pid, pg_blocking_pids(pid) waits_for, locktype, database db, relation rel, '(' || CAST(page AS VARCHAR) || ',' ||CAST(tuple  AS VARCHAR) || ')' "(p,t)", virtualxid vxid, transactionid xid, virtualtransaction vx, mode, granted FROM pg_locks WHERE pid IN (40, 69, 78) ORDER BY pid;`

Первый сеанс  
`DROP TABLE t;`  
`CREATE TABLE t AS SELECT 1 f UNION SELECT 2 UNION SELECT 3;`  
`\set AUTOCOMMIT OFF`  
`SELECT pg_backend_pid() pid, pg_current_xact_id () xid;`  
| pid | xid | 
|-----|-----| 
|  40 | 808 |

`UPDATE t SET f=40 WHERE f=2;`  
| pid | waits_for |   locktype    |  db   |  rel  | (p,t) | vxid | xid |  vx  |       mode       | granted | |
|-----|-----------|---------------|-------|-------|-------|------|-----|------|------------------|---------|-|
|  40 | {}        | relation      | 13757 | 24588 |       |      |     | 3/47 | RowExclusiveLock | t       |808 (40) удеживает эксклюзивную блокировку таблицы|
|  40 | {}        | virtualxid    |       |       |       | 3/47 |     | 3/47 | ExclusiveLock    | t       |808 (40) удеживает эксклюзивную блокировку своего виртуального номера|
|  40 | {}        | transactionid |       |       |       |      | 808 | 3/47 | ExclusiveLock    | t       |808 (40) удеживает эксклюзивную блокировку своего собственного номера|

Второй сеанс  
`docker exec -t -i pg psql -U postgres`  
`\set AUTOCOMMIT OFF`  
`SELECT pg_backend_pid() pid, pg_current_xact_id () xid;`  
| pid | xid | 
|-----|-----| 
|  69 | 809 |

`UPDATE t SET f=69 WHERE f=2;` 

| pid | waits_for |   locktype    |  db   |  rel  | (p,t) | vxid | xid |  vx  |       mode       | granted ||
|-----|-----------|---------------|-------|-------|-------|------|-----|------|------------------|---------|-|
|  40 | {}        | virtualxid    |       |       |       | 3/47 |     | 3/47 | ExclusiveLock    | t       | |
|  40 | {}        | relation      | 13757 | 24588 |       |      |     | 3/47 | RowExclusiveLock | t       | |
|  40 | {}        | transactionid |       |       |       |      | 808 | 3/47 | ExclusiveLock    | t       | |
|  69 | {40}      | tuple         | 13757 | 24588 | (0,2) |      |     | 4/26 | ExclusiveLock    | t       |809(69) захватывает исключительную блокировку изменяемой версии строки|
|  69 | {40}      | transactionid |       |       |       |      | 809 | 4/26 | ExclusiveLock    | t       |809(69) захватывает исключительную блокировку своего собственного номера|
|  69 | {40}      | transactionid |       |       |       |      | 808 | 4/26 | ShareLock        | f       |809(69) получает xmax из tuple (он =808) и запрашивает разделяемую блокировку этого номера транакции. Пока 808(40) еще что-то делает, 809(69) такую блокировку не захватит. Таким образом 40 пока удерживает 69|
|  69 | {40}      | virtualxid    |       |       |       | 4/26 |     | 4/26 | ExclusiveLock    | t       |809(69) захватывает исключительную блокировку своего виртуального номера|
|  69 | {40}      | relation      | 13757 | 24588 |       |      |     | 4/26 | RowExclusiveLock | t       |809(69) захватывает исключительную блокировку таблицы|


Третий сеанс  
`docker exec -t -i pg psql -U postgres`  
`\set AUTOCOMMIT OFF`  
`SELECT pg_backend_pid() pid, pg_current_xact_id () xid;`  
| pid | xid | 
|-----|-----| 
|  78 | 810 |

`UPDATE t SET f=78 WHERE f=2;`  
| pid | waits_for |   locktype    |  db   |  rel  | (p,t) | vxid | xid |  vx  |       mode       | granted | |
|-----|-----------|---------------|-------|-------|-------|------|-----|------|------------------|---------|-|
|  40 | {}        | transactionid |       |       |       |      | 808 | 3/47 | ExclusiveLock    | t       | |
|  40 | {}        | relation      | 13757 | 24588 |       |      |     | 3/47 | RowExclusiveLock | t       | |
|  40 | {}        | virtualxid    |       |       |       | 3/47 |     | 3/47 | ExclusiveLock    | t       | |
|  69 | {40}      | transactionid |       |       |       |      | 808 | 4/26 | ShareLock        | f       | |
|  69 | {40}      | relation      | 13757 | 24588 |       |      |     | 4/26 | RowExclusiveLock | t       | |
|  69 | {40}      | virtualxid    |       |       |       | 4/26 |     | 4/26 | ExclusiveLock    | t       | |
|  69 | {40}      | tuple         | 13757 | 24588 | (0,2) |      |     | 4/26 | ExclusiveLock    | t       | |
|  69 | {40}      | transactionid |       |       |       |      | 809 | 4/26 | ExclusiveLock    | t       | |
|  78 | {69}      | tuple         | 13757 | 24588 | (0,2) |      |     | 5/15 | ExclusiveLock    | f       |810(78) пытается захватить исключительную блокировку изменяемой версии строки. Пока у нее это не получается, так как 809(69) ранее уже захватила ее исключительную блокировку. Таким образом 69 пока удерживает 78|
|  78 | {69}      | virtualxid    |       |       |       | 5/15 |     | 5/15 | ExclusiveLock    | t       |810(78) захватывает исключительную блокировку своего виртуального номера|
|  78 | {69}      | relation      | 13757 | 24588 |       |      |     | 5/15 | RowExclusiveLock | t       |810(78) захватывает исключительную блокировку таблицы|
|  78 | {69}      | transactionid |       |       |       |      | 810 | 5/15 | ExclusiveLock    | t       |810(78) захватывает исключительную блокировку своего собственного номера|

Первый сеанс  
`ROLLBACK;`  

Здесь мы видим, что 808 (40) завершилась и 40 больше не удерживает 69. Здесь больше комментировать нечего, кроме одной новой блокировки  

| pid | waits_for |   locktype    |  db   |  rel  | (p,t) | vxid | xid |  vx  |       mode       | granted | |
|-----|-----------|---------------|-------|-------|-------|------|-----|------|------------------|---------|-|
|  69 | {}        | virtualxid    |       |       |       | 4/26 |     | 4/26 | ExclusiveLock    | t       | |
|  69 | {}        | transactionid |       |       |       |      | 809 | 4/26 | ExclusiveLock    | t       | |
|  69 | {}        | relation      | 13757 | 24588 |       |      |     | 4/26 | RowExclusiveLock | t       | |
|  78 | {69}      | tuple         | 13757 | 24588 | (0,2) |      |     | 5/15 | ExclusiveLock    | t       | |
|  78 | {69}      | relation      | 13757 | 24588 |       |      |     | 5/15 | RowExclusiveLock | t       | |
|  78 | {69}      | transactionid |       |       |       |      | 809 | 5/15 | ShareLock        | f       |810(78) получает xmax из tuple (он =809) и запрашивает разделяемую блокировку этого номера транакции. Пока 809(69) еще что-то делает, 810(78) такую блокировку не захватит. Таким образом 69 пока удерживает 78|
|  78 | {69}      | virtualxid    |       |       |       | 5/15 |     | 5/15 | ExclusiveLock    | t       | |
|  78 | {69}      | transactionid |       |       |       |      | 810 | 5/15 | ExclusiveLock    | t       | |

Второй сеанс  
`ROLLBACK;`  
Здесь мы видим, что 809(69) завершилась и 69 больше не удерживает 78
| pid | waits_for |   locktype    |  db   |  rel  | (p,t) | vxid | xid |  vx  |       mode       | granted |
|-----|-----------|---------------|-------|-------|-------|------|-----|------|------------------|---------|
|  78 | {}        | relation      | 13757 | 24588 |       |      |     | 5/15 | RowExclusiveLock | t       |
|  78 | {}        | virtualxid    |       |       |       | 5/15 |     | 5/15 | ExclusiveLock    | t       |
|  78 | {}        | transactionid |       |       |       |      | 810 | 5/15 | ExclusiveLock    | t       |

Третий сеанс  
`ROLLBACK;` 
| pid | waits_for | locktype | db | rel | (p,t) | vxid | xid | vx | mode | granted |
|-----|-----------|----------|----|-----|-------|------|-----|----|------|---------|
(0 rows)


>Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?

Первая сессия  
`\set AUTOCOMMIT OFF`  
`UPDATE t SET f=11 WHERE f=2;`  

Вторая сессия  
`\set AUTOCOMMIT OFF`  
`UPDATE t SET f=22 WHERE f=3;`  

Третья сессия  
`\set AUTOCOMMIT OFF`  
`UPDATE t SET f=33 WHERE f=1;`  

Первая сессия  
`UPDATE t SET f=11 WHERE f=3;`  

Вторая сессия  
`UPDATE t SET f=22 WHERE f=1;`  

Третья сессия  
`UPDATE t SET f=33 WHERE f=2;`  

ERROR:  deadlock detected  
DETAIL:  Process 78 waits for ShareLock on transaction 814; blocked by process 40.  
Process 40 waits for ShareLock on transaction 815; blocked by process 69.  
Process 69 waits for ShareLock on transaction 816; blocked by process 78.  
HINT:  See server log for query details.  
CONTEXT:  while updating tuple (0,2) in relation "t"  


 
Думаю, что только одному текущему содержанию журнала вряд ли получится разобраться. Скорее всего, поможет изменение уровеня журналирования, чтобы в журнал попадало больше информации. Сейчас же в журнале видна ситуация, когда один процесс начал блокировать другой. А что именно к этому привело и какие были запросы до этого в жрунале не видно. Видно только, что было дальше: кто кого удерживал, кто вставал в очередь, какие запросы к этому привели и чем все закончилось.

Фргамент журнала.   
2023-03-26 21:57:23.120 UTC [40] LOG:  process 40 still waiting for ShareLock on transaction 815 after 200.139 ms  
2023-03-26 21:57:23.120 UTC [40] DETAIL:  Process holding the lock: 69. Wait queue: 40.  
2023-03-26 21:57:23.120 UTC [40] CONTEXT:  while updating tuple (0,3) in relation "t"  
2023-03-26 21:57:23.120 UTC [40] STATEMENT:  UPDATE t SET f=11 WHERE f=3; 

2023-03-26 21:57:55.049 UTC [69] LOG:  process 69 still waiting for ShareLock on transaction 816 after 200.275 ms  
2023-03-26 21:57:55.049 UTC [69] DETAIL:  Process holding the lock: 78. Wait queue: 69.  
2023-03-26 21:57:55.049 UTC [69] CONTEXT:  while updating tuple (0,1) in relation "t"  
2023-03-26 21:57:55.049 UTC [69] STATEMENT:  UPDATE t SET f=22 WHERE f=1;

2023-03-26 21:58:31.253 UTC [78] LOG:  process 78 detected deadlock while waiting for ShareLock on transaction 814 after 200.226 ms  
2023-03-26 21:58:31.253 UTC [78] DETAIL:  Process holding the lock: 40. Wait queue: .  
2023-03-26 21:58:31.253 UTC [78] CONTEXT:  while updating tuple (0,2) in relation "t"  
2023-03-26 21:58:31.253 UTC [78] STATEMENT:  UPDATE t SET f=33 WHERE f=2;  

2023-03-26 21:58:31.253 UTC [78] ERROR:  deadlock detected  
2023-03-26 21:58:31.253 UTC [78] DETAIL:  Process 78 waits for ShareLock on transaction 814; blocked by process 40.  
	Process 40 waits for ShareLock on transaction 815; blocked by process 69.
	Process 69 waits for ShareLock on transaction 816; blocked by process 78.
	Process 78: UPDATE t SET f=33 WHERE f=2;
	Process 40: UPDATE t SET f=11 WHERE f=3;
	Process 69: UPDATE t SET f=22 WHERE f=1;

>Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?
Попробуйте воспроизвести такую ситуацию

Первая сессия

`CREATE TABLE t2 AS SELECT generate_series(1,10000) n;`  
`\set AUTOCOMMIT OFF`  
`LOCK t2;`

Вторяа сессия 

`UPDATE t2 SET n=(SELECT MIN(n) FROM t2);`

Третья сессия 

`UPDATE t2 SET n=(SELECT n FROM t2 ORDER BY n DESC LIMIT 1 FOR UPDATE);`

Первая сессия

`ROLLBACK;`

Вторая сессия

ERROR:  deadlock detected  
DETAIL:  Process 1175 waits for ShareLock on transaction 1008; blocked by process 1182.  
Process 1182 waits for ShareLock on transaction 1007; blocked by process 1175.  
HINT:  See server log for query details.  
CONTEXT:  while updating tuple (442,108) in relation "t2"  


10 тыс строк оказалось достаточно (среда слабенькая). А вот на меньших значениях эффект не наблюдался, блокировка не возникала.
И наоборот. Наблюдал эффект даже на 10 записях, когда параллельно хорошо грузил среду pgbench-ем

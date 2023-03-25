`docker pull postgres:14`  
`docker run --name pg -v c:\var\lib\postgresql\data:/var/lib/postgresql/data -e POSTGRES_PASSWORD=1 -d postgres:14`  
`docker exec -t -i pg bash`  
`passwd postgres`    
`login postgres`  
`pgbench -i postgres`

>Настройте выполнение контрольной точки раз в 30 секунд
   
`psql`  
`ALTER SYSTEM SET checkpoint_timeout = 30;`    
`ALTER SYSTEM SET wal_keep_size=1000;` — увеличиваю, так как при настройках по умолчанию WAL-файлы, скорее всего, начнут переиспользоваться и/или удаляться прямо во время работы утилиты, и я не смогу сделать их дамп, нужный мне для анализа  
`ALTER SYSTEM SET log_checkpoints=on;` — включаем журналрование выполнения контрольных точек, жунал тоже буду анализировать  
`ALTER SYSTEM SET log_destination=csvlog;` — устанавливает формат журналивроания в CSV  
`ALTER SYSTEM SET logging_collector=on;` — включаем сборщик сообщений  
`\q`  
`logout`  
`exit`  
`docker restart pg`  
`docker exec -t -i pg bash`  
`login postgres`   
`psql`    

>10 минут c помощью утилиты pgbench подавайте нагрузку

`SELECT pg_current_wal_insert_lsn();`  
0/237E1D8 — первый LSN (на момент перед запуском утилиты)    
`SELECT pg_stat_reset_shared('bgwriter');` — чистим статистику  
`\q`  
`pgbench -c8 -P 60 -T 600 -U postgres postgres`  
`psql`  
`SELECT pg_current_wal_insert_lsn();`   
0/21AD6970 — последний LSN (на момент после запуска утилиты)  
`SELECT * FROM pg_stat_bgwriter; \gx`  

checkpoints_timed     | 21  
checkpoints_req       | 0  
checkpoint_write_time | 536347  
checkpoint_sync_time  | 1657  
buffers_checkpoint    | 41689  
buffers_clean         | 0  
maxwritten_clean      | 0  
buffers_backend       | 3049  
buffers_backend_fsync | 0  
buffers_alloc         | 5089  
stats_reset           | 2023-03-24 14:58:02.662067+00  

>Измерьте, какой объем журнальных файлов был сгенерирован за это время. 

`SELECT pg_walfile_name('0/21AD6970'), pg_walfile_name('0/237E1D8');`  
 000000010000000000000021 | 000000010000000000000002  
32х16Мб=512Мб — если грубо (потому что не факт, что все записи в этих WAL-ах были добавлены во время работы утилиты)  

`SELECT pg_wal_lsn_diff ('0/21AD6970', '0/237E1D8')/1024/1024;`  
~504Мб — чуть точнее   

>Оцените, какой объем приходится в среднем на одну контрольную точку

Это выше рассчитанный объем WAL-ов, поделенный на количество выполненных контрольных точек (статистика показывает, что их было выполнено 21, хотя опять же не факт, что все они были выполнены именно во время работы утилиты).  
Ну или я не понял вопрос.

>Проверьте данные статистики: все ли контрольные точки выполнялись точно по расписанию. Почему так произошло?

При checkpoint_timeout = 30 и 10 минутах работы утилиты должно было выполниться ну никак не больше 20 конторльных точек. Статистика показывает 21. Думаю, что одна контрольная точка успела выполнится в моменты между получсением первого LSN и запуском утилиты или между окончанием работы утилиты и получением последнего LSN. 

`\q`  
`/usr/lib/postgresql/14/bin/pg_waldump -p /var/lib/postgresql/data/pg_wal -s 0/237E1D8 -e 0/21AD6970 | egrep  'CHECKPOINT_ONLINE' -c`    
20 — вот, резулльтаты дампа показывают, что их таки было 20

`/usr/lib/postgresql/14/bin/pg_waldump -p /var/lib/postgresql/data/pg_wal -s 0/237E1D8 -e 0/21AD6970 | egrep  'CHECKPOINT_ONLINE' > /var/lib/postgresql/data/waldump.txt`

[waldump.txt](waldump.txt) 

>Почему так произошло?  

Судя по вопросу, я должен был столкнуться с ситуацией, когда конторльных точек должно было быть меньше 20?
Не столкнулся. Хотя, подозреваю, что их могло бы быть меньше, но не знаю почему. Можно еще журнал посмотреть

Журнал сервера [postgresql-2023-03-24_145731.csv](postgresql-2023-03-24_145731.csv)  

Создаем таблицу для импорта журнала (скрипт подсмотрел [тут](https://postgrespro.ru/docs/postgresql/14/runtime-config-logging#RUNTIME-CONFIG-LOGGING-CSVLOG))

копируем журнал в таблицу  
`COPY postgres_log FROM '/var/lib/postgresql/data/log/postgresql-2023-03-24_145731.csv' WITH csv;`

`SELECT COUNT(1) FROM postgres_log WHERE backend_type='checkpointer' AND message LIKE 'checkpoint complete%';`   
22

`SELECT log_time FROM postgres_log WHERE backend_type='checkpointer' AND message LIKE 'checkpoint complete%';`

 2023-03-24 14:58:02.026+00  
 2023-03-24 14:58:58.421+00  
 2023-03-24 14:59:28.802+00  
 2023-03-24 14:59:58.276+00  
 2023-03-24 15:00:28.116+00  
 2023-03-24 15:00:58.159+00  
 2023-03-24 15:01:28.051+00  
 2023-03-24 15:01:58.109+00  
 2023-03-24 15:02:28.261+00  
 2023-03-24 15:02:58.116+00  
 2023-03-24 15:03:28.386+00  
 2023-03-24 15:03:58.141+00  
 2023-03-24 15:04:28.152+00  
 2023-03-24 15:04:58.146+00  
 2023-03-24 15:05:28.167+00  
 2023-03-24 15:05:58.111+00  
 2023-03-24 15:06:28.174+00  
 2023-03-24 15:06:58.156+00  
 2023-03-24 15:07:28.089+00  
 2023-03-24 15:07:58.17+00  
 2023-03-24 15:08:28.043+00  
 2023-03-24 15:09:28.06+00  

Почему их 22 при том, что по статистике их 21, а по дампу 20, я прокомментировать не могу. Почему вторая и последняя, судя по журналу, были выполнены с опозданием тоже не знаю. Возможно, в эти периоды не было активностей, поэтому у сервера не было надобности в запуске контрольных точек. 

>Сравните tps в синхронном/асинхронном режиме утилитой pgbench

`\q`  
`pgbench -i postgres`

по умолчанию synchronous_commit=on

`pgbench -c8 -P 6 -T 60 -U postgres postgres`

progress: 6.0 s, 761.6 tps, lat 9.818 ms stddev 8.530  
progress: 12.0 s, 824.2 tps, lat 9.691 ms stddev 8.469  
progress: 18.0 s, 868.7 tps, lat 9.193 ms stddev 5.760  
progress: 24.0 s, 701.2 tps, lat 11.385 ms stddev 8.191  
progress: 30.0 s, 436.0 tps, lat 18.328 ms stddev 11.393  
progress: 36.0 s, 422.2 tps, lat 18.939 ms stddev 14.525  
progress: 42.0 s, 871.8 tps, lat 9.162 ms stddev 5.977  
progress: 48.0 s, 826.7 tps, lat 9.658 ms stddev 8.680  
progress: 54.0 s, 901.8 tps, lat 8.849 ms stddev 5.833  
progress: 60.0 s, 898.0 tps, lat 8.892 ms stddev 5.744  

`psql`  
`ALTER SYSTEM SET synchronous_commit=off;`  
`\q`  
`pgbench -i postgres`  
`pgbench -c8 -P 6 -T 60 -U postgres postgres`  

progress: 6.0 s, 2686.2 tps, lat 2.792 ms stddev 1.675  
progress: 12.0 s, 2770.3 tps, lat 2.872 ms stddev 6.637  
progress: 18.0 s, 2893.0 tps, lat 2.749 ms stddev 1.580  
progress: 24.0 s, 2844.6 tps, lat 2.796 ms stddev 1.635  
progress: 30.0 s, 2787.1 tps, lat 2.853 ms stddev 2.317  
progress: 36.0 s, 2880.7 tps, lat 2.761 ms stddev 1.696  
progress: 42.0 s, 2278.1 tps, lat 3.491 ms stddev 4.159  
progress: 48.0 s, 2832.9 tps, lat 2.807 ms stddev 2.041  
progress: 54.0 s, 2853.6 tps, lat 2.786 ms stddev 1.654  
progress: 60.0 s, 2854.6 tps, lat 2.785 ms stddev 1.645  

>Объясните полученный результат

В асинхронном режиме записи в WAL сервер смог обрабатывать транзакций в секунду в 3-3,5 раза больше чем при синхронном режиме записи. Так происходит, потому что в синхронном режиме при коммите транзакции сервер ждет, когда все журнальные записи об этой транзации запишутся на диск (синхронизируются). При асинхронном не ждет, сразу комитит транзакцию. За счет ожидания и снижается производительность сервера (но повышатеся надежность — записи журнала гарнтированно попадут на диск) 

>Создайте новый кластер с включенной контрольной суммой страниц.

`docker run --name pg -v c:\var\lib\postgresql\data:/var/lib/postgresql/data -e POSTGRES_PASSWORD=1 -e  POSTGRES_INITDB_ARGS="--data-checksums" -d postgres:14`  
`docker exec -t -i pg bash`  
`passwd postgres`  

>Создайте таблицу. Вставьте несколько значений.

`psql`  
`CREATE TABLE test_table AS SELECT 1;`  
`SELECT pg_relation_filepath('test_table');`  
base/13757/16387  

>Выключите кластер. Измените пару байт в таблице. Включите кластер и сделайте выборку из таблицы. 

Стопить кластер смысла нет, так как он у меня развернут в контейнер докера и остановка кластера все равно приводит к остановке контейнера целиком. Поэтому просто стоплю контейнер, редактирую таблицу путем редактирования файла в примонтированном томе и запускаю контейнер.  

`\q`  
`logout`  
`exit`  
`docker stop pg`  
редактирую файл таблицы  
`docker start pg`  
`docker exec -t -i pg bash`  
`login postgres`  
`psql`  
`SELECT * FROM test_table;`  
WARNING:  page verification failed, calculated checksum 63342 but expected 45099  
ERROR:  invalid page in block 0 of relation base/13757/16387  

>Что и почему произошло?

Сервер увидел, что изменилась контрольная сумма таблицы, соответственно, считает ее поврежденной и не позволяет с ней работать

>как проигнорировать ошибку и продолжить работу?
 
`ALTER SYSTEM SET ignore_checksum_failure=on;`  
`SELECT pg_reload_conf();`  
`SELECT * FROM test_table;`

WARNING:  page verification failed, calculated checksum 63342 but expected 45099  
 ?column?  
\----------  
        1  
(1 row)  

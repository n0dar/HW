>создать GCE инстанс типа e2-medium и диском 10GB

докер на виндоуз 10 на домашнем ноуте

>установить на него PostgreSQL 14 с дефолтными настройками

`docker pull postgres:14`  
`docker run --name pg -v c:\var\lib\postgresql\data:/var/lib/postgresql/data -e POSTGRES_PASSWORD=1 -d postgres:14`
>применить параметры настройки PostgreSQL из прикрепленного к материалам занятия файла

`docker exec -t -i pg bash`  
`apt update; apt install nano procps`  
`passwd postgres`  
`login postgres`  
`psql`  
`show config_file;`  
`\q`  
`mkdir /var/lib/postgresql/data/myconf`     
`nano /var/lib/postgresql/data/myconf/my.conf`

Не все предлагаемые в ДЗ настройки подходят для моей текуйщей среды  

>shared_buffers = 1GB

`free -h`  
               total        used        free      shared  buff/cache   available  
Mem:           969Mi       558Mi        76Mi        12Mi       335Mi       259Mi
Swap:          1.0Gi        46Mi       977Mi

из документации   
>**_выделять для shared_buffers более 40% ОЗУ вряд ли будет полезно_**

меняю на shared_buffers = 387MB  

В данном случае, не на что не влият (соединений все равно не будет больше), но все равно смущает

>max_connections = 40

судя по -c8 в параметрах запуска pgbench, тестироваться предлагается на 8 соединениях + 1 мое (терминал)  
меняю на max_connections = 9  

`cat /var/lib/postgresql/data/myconf/my.conf`

max_connections = 9  
shared_buffers = 387MB  
effective_cache_size = 3GB       
maintenance_work_mem = 512MB        
checkpoint_completion_target = 0.9  
wal_buffers = 16MB  
default_statistics_target = 500     
random_page_cost = 4  
effective_io_concurrency = 2        
work_mem = 6553kB  
min_wal_size = 4GB  
max_wal_size = 16GB

инклюдим файл с настройками в основной конфигурационный файл

`nano /var/lib/postgresql/data/postgresql.conf`

было #include = '...'  
стало include = 'myconf/my.conf'

`logout`  
`exit`   
`docker restart pg`  — здесь приходится рестартовать контейнер целиком, чтобы применились параметры настройки. pg_lsclusters не видит кластер, pg_ctlcluster тоже не работает. Не смог понять почему, может, из-за того, что Postgres установлен из имейджа сразу в контейнер, без педварительной установки в контейнер ubuntu   
`docker exec -t -i pg bash`  
`login postgres`  
`psql`  
`SELECT COUNT(1) FROM pg_file_settings WHERE sourcefile LIKE '%my.conf' AND applied='t';`

12 — косвенное подтверждение того, что все 12 параметров из my.conf СУБД подхватила (ну чтоб не пихать сюда результат полной выборки)

`\q`  

>выполнить pgbench -i postgres  
>запустить pgbench -c8 -P 60 -T 600 -U postgres postgres  
>дать отработать до конца  
>дальше настроить autovacuum максимально эффективно (4)  
>построить график по получившимся значениям  
>так чтобы получить максимально ровное значение tps (6)

долго думал, каким образом можно выоплнить требование 4 и 6 строки — ничего не надумал.  
Также не понял, как pgbech должен тут помочь, тем более, что в документации про pgbench написано следующее

>_Стандартный сценарий тестирования также довольно сильно зависит от того, сколько времени прошло с момента инициализации таблиц: накопление неактуальных строк и «мёртвого» пространства в таблицах влияет на результаты. Чтобы правильно оценить результаты, необходимо учитывать, сколько всего изменений было произведено и когда выполнялась очистка. **Если же включена автоочистка, это может быть чревато непредсказуемыми изменениями оценок производительности**_

Пересмотрел часть вебинара касательно автовакуума несколько раз, покрутил перзентацию, посмотрел допматериалы на хабре, указанные в ней.   
Лучше не стало. Пробую просто менять параметры автовакуума и смотреть, что получается. При этом каждый раз предварительно запуская утилиту с параметром -i — она дропает и пересоздает таблицы для тестирования. Если этого не делать, то каждый следующий запуск теста будет выполняться, наверное, совсем в разных условиях, так как во время предыдущего запуска содержание таблиц, которые создает утилита, меняется. Таким образом каждый запуск утилиты с разными параметрами автовакуума будет хоть в какой-то мере выполняться у меня в эталонных условиях. Ну просто иначе не понятно, как сранвивать между собой результаты запусков, если они выполнялись в разных условиях? 

`pgbench -i postgres` — запускаю перед каждым запуском утилиты в режиме тестирования  
`pgbench -c8 -P 10 -T 100 -U postgres postgres` — утилиту в режиме тестирования запускаю с такими параметрами, пониммаю, что это влияет на результаты тестирования, но иначе просто долго ждать их, думаю, что для ДЗ это не так критично  

параметры меняю путем их добавления в my.conf, потом перестартовываю контейнер, потом проверяю, что изменения настроек применились

`SELECT * FROM pg_file_settings WHERE sourcefile LIKE '%my.conf' AND applied='t'`;


**Запуск 1**  
с настройками автовакуума по умолчанию  

progress: 10.0 s, 701.4 tps, lat 10.946 ms stddev 8.650  
progress: 20.0 s, 801.2 tps, lat 9.963 ms stddev 6.612  
progress: 30.0 s, 483.0 tps, lat 16.534 ms stddev 29.344  
progress: 40.0 s, 423.8 tps, lat 18.849 ms stddev 12.239  
progress: 50.0 s, 671.8 tps, lat 11.893 ms stddev 10.091  
progress: 60.0 s, 857.5 tps, lat 9.314 ms stddev 5.999  
progress: 70.0 s, 825.4 tps, lat 9.672 ms stddev 6.639  
progress: 80.0 s, 830.5 tps, lat 9.614 ms stddev 6.281 
progress: 90.0 s, 808.0 tps, lat 9.882 ms stddev 7.764  
progress: 100.0 s, 818.0 tps, lat 9.763 ms stddev 6.338  
transaction type: <builtin: TPC-B (sort of)>  
scaling factor: 1  
query mode: simple  
number of clients: 8  
number of threads: 1  
duration: 100 s  
number of transactions actually processed: 72214  
latency average = 11.018 ms  
latency stddev = 10.957 ms  
initial connection time = 382.230 ms  
**tps = 724.712556 (without initial connection time)**  


**Запуск 2**

`grep processor /proc/cpuinfo -c`  
2

меняю autovacuum_max_workers с 3 на 1 для всех следующих запусков, согласно рекомендациям данным на вебинаре и с учетом того, что контейнеру доступны 2 процессора. И, судя по результатам, несмотря на рекомендации, результаты в среднем хуже, чем в предыдущем запуске  

progress: 10.0 s, 800.0 tps, lat 9.570 ms stddev 8.587  
progress: 20.0 s, 58.7 tps, lat 136.162 ms stddev 67.683  
progress: 30.0 s, 416.6 tps, lat 19.307 ms stddev 19.097  
progress: 40.0 s, 659.2 tps, lat 12.137 ms stddev 9.315  
progress: 50.0 s, 799.3 tps, lat 9.989 ms stddev 6.857  
progress: 60.0 s, 767.2 tps, lat 10.406 ms stddev 7.122  
progress: 70.0 s, 857.6 tps, lat 9.312 ms stddev 5.839  
progress: 80.0 s, 861.8 tps, lat 9.264 ms stddev 5.949  
progress: 90.0 s, 848.7 tps, lat 9.410 ms stddev 6.044  
progress: 100.0 s, 864.1 tps, lat 9.238 ms stddev 5.917  
transaction type: <builtin: TPC-B (sort of)>  
scaling factor: 1  
query mode: simple  
number of clients: 8  
number of threads: 1  
duration: 100 s  
number of transactions actually processed: 69340  
latency average = 11.482 ms  
latency stddev = 15.634 ms  
initial connection time = 330.313 ms  
**tps = 695.509076 (without initial connection time)**  

**Запуск 3**  
меняю autovacuum_naptime на 1s, после запуска утилиты удаляю настройку параметра из my.conf (для последующих запусков)
вроде стало чуть лучше  

progress: 10.0 s, 803.8 tps, lat 9.593 ms stddev 6.577  
progress: 20.0 s, 586.5 tps, lat 13.612 ms stddev 10.774  
progress: 30.0 s, 451.1 tps, lat 17.724 ms stddev 13.122  
progress: 40.0 s, 847.5 tps, lat 9.418 ms stddev 6.056  
progress: 50.0 s, 857.6 tps, lat 9.309 ms stddev 6.038  
progress: 60.0 s, 860.6 tps, lat 9.278 ms stddev 5.959  
progress: 70.0 s, 842.3 tps, lat 9.479 ms stddev 6.261  
progress: 80.0 s, 845.7 tps, lat 9.442 ms stddev 6.125  
progress: 90.0 s, 837.8 tps, lat 9.529 ms stddev 6.150  
progress: 100.0 s, 843.6 tps, lat 9.463 ms stddev 6.239  
transaction type: <builtin: TPC-B (sort of)>  
scaling factor: 1  
query mode: simple  
number of clients: 8 
number of threads: 1  
duration: 100 s  
number of transactions actually processed: 77773  
latency average = 10.234 ms  
latency stddev = 7.515 ms  
initial connection time = 335.579 ms  
**tps = 780.159103 (without initial connection time)**

**Запуск 4**   
autovacuum_vacuum_threshold оставляю по умолчанию (50), а autovacuum_vacuum_scale_factor выставляю в 0, после запуска удаляю настройку из my.conf (для послеующих запусков)

progress: 10.0 s, 768.7 tps, lat 9.940 ms stddev 6.553  
progress: 20.0 s, 829.8 tps, lat 9.626 ms stddev 6.200  
progress: 30.0 s, 855.7 tps, lat 9.330 ms stddev 5.894  
progress: 40.0 s, 840.1 tps, lat 9.502 ms stddev 6.066  
progress: 50.0 s, 825.3 tps, lat 9.676 ms stddev 6.321  
progress: 60.0 s, 829.5 tps, lat 9.625 ms stddev 6.143  
progress: 70.0 s, 821.8 tps, lat 9.712 ms stddev 6.078  
progress: 80.0 s, 820.6 tps, lat 9.736 ms stddev 6.160  
progress: 90.0 s, 477.8 tps, lat 16.710 ms stddev 29.935  
progress: 100.0 s, 434.2 tps, lat 18.405 ms stddev 11.442  
transaction type: <builtin: TPC-B (sort of)>  
scaling factor: 1  
query mode: simple  
number of clients: 8  
number of threads: 1  
duration: 100 s  
number of transactions actually processed: 75043  
latency average = 10.598 ms  
latency stddev = 10.242 ms  
initial connection time = 421.739 ms  
**tps = 753.343216 (without initial connection time)**

**Запуск 5**

меняю autovacuum_vacuum_cost_delay на 100ms

Здесь я сначала вляпался в такую штуку. Я не знал, что максимальное значение здесь 100ms. Указал 10s и сервер отказался запускаться. Не только сервер, но и сам контейнер. Он при запуске сразу завершался ошибкой. Максимум, что я смог сделать, это увидеть запись в логах контейнера, почему он не запускается (из-за недопустимого значения autovacuum_vacuum_cost_delay). Доступ к комапнлной строке уже не получить, my.conf не отредактировать, получается, только убивать контейнер со всеми настройкми. Это уже потом, при повторной настройке, я смонтировал том в конейнер, чтобы в случае чего иметь возможность редактировать настройки средтвами ОС, на которой запущен докер. Если бы я сначала развернул в контейнер ubuntu, понятно, что падал бы только Postgres, а доступ к командной строке ОС я имел бы без проблем.  

2023-03-19 13:49:38 2023-03-19 10:49:38.775 UTC [1] LOG:  10000 ms is outside the valid range for parameter "autovacuum_vacuum_cost_delay" (-1 .. 100)  
2023-03-19 13:49:38 2023-03-19 10:49:38.775 UTC [1] FATAL:  configuration file "/var/lib/postgresql/data/myconf/my.conf" contains errors  

в общем, запускаю с корректной настройкой  

progress: 10.0 s, 783.4 tps, lat 9.776 ms stddev 6.541  
progress: 20.0 s, 807.3 tps, lat 9.890 ms stddev 6.440  
progress: 30.0 s, 804.6 tps, lat 9.925 ms stddev 6.431  
progress: 40.0 s, 809.0 tps, lat 9.867 ms stddev 6.399  
progress: 50.0 s, 820.9 tps, lat 9.728 ms stddev 6.358  
progress: 60.0 s, 843.4 tps, lat 9.468 ms stddev 6.105  
progress: 70.0 s, 789.2 tps, lat 10.118 ms stddev 6.562  
progress: 80.0 s, 661.0 tps, lat 12.074 ms stddev 7.969  
progress: 90.0 s, 511.1 tps, lat 15.628 ms stddev 10.919  
progress: 100.0 s, 500.5 tps, lat 15.971 ms stddev 11.702  
transaction type: <builtin: TPC-B (sort of)>  
scaling factor: 1  
query mode: simple  
number of clients: 8  
number of threads: 1  
duration: 100 s  
number of transactions actually processed: 73312  
latency average = 10.850 ms  
latency stddev = 7.707 ms  
initial connection time = 401.820 ms  
**tps = 735.933200 (without initial connection time)**  

в дальнейших экспериментах не вижу смысла, так как я все равно не понял по результатам лекции, как настрить автовакуум максимально эффективно. Лекция в целом и в частности про автовакуум понятна, как выполнять настройки понятно, назначение каждого параметра настройки понятно (почитал документацию от PostgresPro + материалы на хабре, на которые ссылается презентация), но не понятно, как все это собрать в кучу, чтобы настройка автовакуума получилась максимально эффнктивной. Что я делаю не так? 

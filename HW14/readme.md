`docker run --name vm1 -p 5431:5432 -v c:\vm1:/var/lib/postgresql/data -e POSTGRES_PASSWORD=1 -d postgres:15`  
`docker exec vm1 psql -U postgres -c 'CREATE TABLE t1(f VARCHAR); CREATE TABLE t2(f VARCHAR);' -c 'ALTER SYSTEM SET wal_level = logical;'`  
`docker restart vm1`  
`docker exec vm1 psql -U postgres -c 'CREATE PUBLICATION VM1_T1 FOR TABLE t1;'`  
`docker exec vm1 psql -U postgres -c 'CREATE PUBLICATION VM1_T2 FOR TABLE t2;'`
`docker container inspect -f '{{ .NetworkSettings.IPAddress }}' vm1`  
172.17.0.2   

`docker run --name vm2 -p 5432:5432 -v c:\vm2:/var/lib/postgresql/data -e POSTGRES_PASSWORD=1 -d postgres:15`  
`docker exec vm2 psql -U postgres -c 'CREATE TABLE t1(f VARCHAR); CREATE TABLE t2(f VARCHAR);' -c 'ALTER SYSTEM SET wal_level = logical;'`  
`docker restart vm2`  
`docker exec vm2 psql -U postgres -c 'CREATE PUBLICATION VM2 FOR TABLE t2;'`  
`docker container inspect -f '{{ .NetworkSettings.IPAddress }}' vm2`     
172.17.0.3  

`docker run --name vm3 -p 5433:5432 -v c:\vm3:/var/lib/postgresql/data -e POSTGRES_PASSWORD=1 -d postgres:15`  
`docker exec vm3 psql -U postgres -c 'CREATE TABLE t1(f VARCHAR); CREATE TABLE t2(f VARCHAR);' -c 'ALTER SYSTEM SET wal_level = replica;'`
`echo host replication all samenet trust  >> /var/lib/postgresql/data/pg_hba.conf`
`docker restart vm3`  
`docker container inspect -f '{{ .NetworkSettings.IPAddress }}' vm3`     
172.17.0.4  

`docker exec vm1 psql -U postgres -c "CREATE SUBSCRIPTION VM1 CONNECTION 'host=172.17.0.3 port=5432 user=postgres password=1 dbname=postgres' PUBLICATION VM2 WITH (copy_data = false);"`  

`docker exec vm2 psql -U postgres -c "CREATE SUBSCRIPTION VM2 CONNECTION 'host=172.17.0.2 port=5432 user=postgres password=1 dbname=postgres' PUBLICATION VM1_T1 WITH (copy_data = false);"`  

`docker exec vm3 psql -U postgres -c "CREATE SUBSCRIPTION VM3_T1 CONNECTION 'host=172.17.0.2 port=5432 user=postgres password=1 dbname=postgres' PUBLICATION VM1_T1 WITH (copy_data = false);"`

`docker exec vm3 psql -U postgres -c "CREATE SUBSCRIPTION VM3_T2 CONNECTION 'host=172.17.0.2 port=5432 user=postgres password=1 dbname=postgres' PUBLICATION VM1_T2 WITH (copy_data = false);"`  

Разворачивать четвертый инстанс Postgres-а сразу в контейнере бесполезно. Так как нужно будет стопить Postgres для удаления основного каталога. При этом, если остановить Postgres, автоматом остановится и сам контейнер, и я уже не смогу получить доступ к командной строке ОС. Ставлю в четвертый котейнер ubuntu, и уже на ней поднимаю Postgres  

`docker run --name vm4 -i -t ubuntu:20.04 bash`  
`apt install -y lsb-release gnupg wget`  
`sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'`  
`wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | apt-key add -`  
`apt update`  
`apt install -y postgresql-15`  
`rm -rf /var/lib/postgresql/15/main`  
`login postgres`
`pg_basebackup -h 172.17.0.4 -p 5432 -R -D /var/lib/postgresql/15/main`  
`pg_ctlcluster 15 main start`  

>Warning: connection to the database failed, disabling startup checks:  
psql: error: connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: FATAL:  database locale is incompatible with operating system  
DETAIL:  The database was initialized with LC_COLLATE "en_US.utf8",  which is not recognized by setlocale().  
HINT:  Recreate the database with another locale or install the missing locale.  

Печаль. Мои знания докера, линукса и посгреса весьма скудны, поэтому попробую угадать, что произошло. vm3 — это постгрес, раввернутый напрямую в контейнер докера, который, в свою очередь, развернут на винде. vm4 — это убунта, развенутая в контейнер того же докера на винде, а потом на этой убунте был развернут постгрес. Под vm3 (хоть бы постгрес и развернут прямо в контейнер) все равно крутится дебиан. Ну и очевидно, что в двух контейнерах постгрес живет на разных ОС (дебиан и убунта), развернут разными пакетами и живет в разных окружениях, видимо, и с разной локалью. Постргес, поднятый на vm4 из бэкапа, сделанного на vm3, не нашел на vm4 локаль, которая у него была на vm3. Кластер на vm4, к моему удивлению, таки все равно стартанул и так, но радость была недолгой: я понял, что это все вообще неважно, так как там дебиан, а тут убунта, а репликация v3-v4 — логическая  

`docker exec -t -i vm3 bash -c "grep PRETTY_NAME= /etc/os-release"`  
PRETTY_NAME="Debian GNU/Linux 11 (bullseye)"  
`docker stop vm4`  
`docker rm vm4`  
`docker pull debian:11`  
`docker run --name vm4 -i -t debian:11 bash`  
`apt update`
`apt install -y lsb-release gnupg wget`  
`sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'`  
`wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | apt-key add -`  
`apt update`  
`apt install -y postgresql-15`  
`rm -rf /var/lib/postgresql/15/main`  
`passwd postgres`   
`login postgres`  
`pg_basebackup -h 172.17.0.4 -p 5432 -R -D /var/lib/postgresql/15/main`  
`pg_ctlcluster 15 main start`

все то же самое, хотя ОС на vm3 и vm4 все равно нужно было выровнять

`docker exec -t -i vm3 bash -c "locale -a"`  
C  
C.UTF-8  
en_US.utf8  
POSIX  

`docker start vm4`  
`docker exec -t -i vm4 bash -c "locale -a"`  
C  
C.UTF-8  
POSIX  

Устанавливаем локаль с помощью `dpkg-reconfigure locales`    
`locale -a`  
C  
C.UTF-8  
POSIX  
en_US.utf8  
`rm -rf /var/lib/postgresql/15/main`  
`login postgres`  
`pg_basebackup -h 172.17.0.4 -p 5432 -R -D /var/lib/postgresql/15/main`  
`pg_ctlcluster 15 main start`

ура, теперь проверяем все 

SUBSCRIPTION VM1 на PUBLICATION VM2  
`docker exec vm2 psql -U postgres -c "INSERT INTO t2 SELECT 'I was added to T2 on VM2';"`    
`docker exec vm1 psql -U postgres -c "SELECT * FROM t2;"`  
I was added to T2 on VM2  

SUBSCRIPTION VM2 на PUBLICATION VM1_T1  
`docker exec vm1 psql -U postgres -c "INSERT INTO t1 SELECT 'I was added to T1 on VM1';"`    
`docker exec vm2 psql -U postgres -c "SELECT * FROM t1;"`  
I was added to T1 on VM1  

SUBSCRIPTION VM3_T1 на PUBLICATION VM1_T1  
`docker exec vm3 psql -U postgres -c "SELECT * FROM t1;"`  
I was added to T1 on VM1  

SUBSCRIPTION VM3_T2 на PUBLICATION VM1_T2  
`docker exec vm3 psql -U postgres -c "SELECT * FROM t2;"`    
I was added to T2 on VM2  

горячий резерв  
`docker exec vm3 psql -U postgres -c "INSERT INTO t1 SELECT 'I was added to T1 on VM3';"`      
`docker exec vm3 psql -U postgres -c "INSERT INTO t2 SELECT 'I was added to T2 on VM3';"`  
`docker exec -t -i vm4 bash`  
`login postgres`  
`psql`  
`SELECT * FROM t1;`  
I was added to T1 on VM1  
I was added to T1 on VM3  
`SELECT * FROM t2;`  
I was added to T2 on VM2  
I was added to T2 on VM3  


>создайте виртуальную машину c Ubuntu 20.04 LTS (bionic) в **GCE/ЯО**

c GCE/ЯО не умеею работать. За 15 лет ни разу не требовались + специфика текущего места работа не подразумевает (и не будет никогда подразумевать) публичных Cloud-based решений. Попробую выполнить ДЗ в докере. Если из-за этого снизите балл по ДЗ, отдельно предемонстрирую умение размечать и форматировать диски, нет проблем =) Но GCE в любом случае не актуален, а с ЯО, простите, лень разбираться. Понадобится в рейальной жизни — разберусь за вечер, думаю  

`docker -v`  
Docker version 20.10.22, build 3a2c30b  

`docker pull ubuntu:20.04`  
 
вместо дополнительного диска монитирую средствами докера локальную папку c:\mnt_data на папку /mnt/data контейнера; выполняю все это в момент запуска контейнера

`docker run --name srv1 -w /mnt/data -v c:\mnt_data:/mnt/data -i -t ubuntu:20.04 bash`  
root@dd114f312ccd:/mnt/data#

>поставьте на нее PostgreSQL 14 через sudo apt

`apt update`   
`apt install -y lsb-release gnupg wget sudo nano`  
`sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'`  
`wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | apt-key add -`  
`apt update`  
`apt install -y postgresql-14`

>проверьте что кластер запущен

`pg_lsclusters`  
Ver Cluster Port Status Owner    Data directory              Log file  
14  main    5432 down   postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log  

`pg_ctlcluster 14 main start`  
`pg_lsclusters`  
Ver Cluster Port Status Owner    Data directory              Log file  
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log  

>зайдите из под пользователя postgres в psql 

`passwd postgres`  
`login postgres`  
postgres@4f0a90a9bbac:~$  
`psql`  
postgres=#  

>и сделайте произвольную таблицу с произвольным содержимым

`create table test_table(c1 text);`  
`\q`  
postgres@4f0a90a9bbac  
`logout`  
root@dd114f312ccd:/mnt/data#

>остановите postgres

`pg_ctlcluster 14 main stop`

>сделайте пользователя postgres владельцем /mnt/data  

`chown -R postgres:postgres /mnt/data/`

>перенесите содержимое /var/lib/postgres/14 в /mnt/data

**Q:** опечатка в ДЗ, должно быть postgresql вместо postgres???
 
`mv /var/lib/postgresql/14 /mnt/data`  
`ls /mnt/data`  
14  
`ls /var/lib/postgresql/14`  
ls: cannot access '/var/lib/postgresql/14': No such file or directory

>попытайтесь запустить кластер

`pg_ctlcluster 14 main start`  
Error: /var/lib/postgresql/14/main is not accessible or does not exist

>**Q:** напишите получилось или нет и почему

**A:** не поулчилось. /var/lib/postgresql/14/main — основной каталог СУБД. Мы его переместили, СУБД не может его найти н поэтому не может стартануть  

>задание: найти конфигурационный параметр в файлах раположенных в /etc/postgresql/14/main который надо поменять и поменяйте его

`nano /etc/postgresql/14/main/postgresql.conf` 

>**Q:** напишите что и почему поменяли

**A:** путь к основному каталогу СУБД задается в переменной data_directory конфигурационного файла postgresql.conf. Находим ее и меняем ее значение на '/mnt/data/14/main'

>попытайтесь запустить кластер

`pg_ctlcluster 14 main start`  
`pg_lsclusters`  
Ver Cluster Port Status Owner    Data directory              Log file  
14  main    5432 online postgres /mnt/data/14/main /var/log/postgresql/postgresql-14-main.log 

>**Q:** напишите получилось или нет и почему

**A:** СУБД стартанула, так в конфигурационный файл внесены необходимы исправления

**Q:** зайдите через через psql и проверьте содержимое ранее созданной таблицы

postgres=#  
`\dt`  
           List of relations  
 Schema |    Name    | Type  |  Owner  
--------+------------+-------+----------  
 public | test_table | table | postgres  
(1 row)  

**A:** ранее созданная таблица на месте

>задание со звездочкой *: 
>не удаляя существующий инстанс ВМ сделайте новый, 

`docker stop srv1`  
`docker run --name srv2 -w /mnt/data -v c:\mnt_data:/mnt/data -i -t ubuntu:20.04 bash`  

>поставьте на его PostgreSQL

`apt update`   
`apt install -y lsb-release gnupg wget sudo nano`  
`sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'`  
`wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | apt-key add -`  
`apt update`  
`apt install -y postgresql-14`  
`pg_lsclusters`    

Ver Cluster Port Status Owner    Data directory              Log file  
14  main    5432 down   postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log  

>удалите файлы с данными из /var/lib/postgres, 

`rm -R /var/lib/postgresql`

>перемонтируйте внешний диск который сделали ранее от первой виртуальной машины ко второй 

у меня к обоим контейнерам (srv1 и srv2) смонитрована одна и таже локальная папка (c:\mnt_data) в один и тот же каталог (/mnt/data ). В первом контейнере мы копировали в этот каталог файлы. Они сейчас лежат в c:\mnt_data. Моя задача подсунуть эти файлы второму инстансу СУБД, который мы равернули в первом контейнере. Точнее, мне их и подсовывать не нужно, так как второй контейнер их и так видит (все монтировалось как и в первом контейнере). Задача сводится к изменению data_directory для второго инстанса СУБД, развернутого во втором контейнере (srv2). В принципе, удалять /var/lib/postgresql было вообще не обязательно, нужно просто поменять  data_directory и старатнуть СУБД. Пробуем...

`nano /etc/postgresql/14/main/postgresql.conf` 

было data_directory = '/var/lib/postgresql/14/main'  
стало data_directory = '/mnt/data/14/main'   

>и запустите PostgreSQL на второй машине так чтобы он работал с данными на внешнем диске, 

`pg_ctlcluster 14 main start` 
`passwd postgres`  
`login postgres`  
postgres@4f0a90a9bbac:~$  
`psql`  
postgres=#  

`\dt`  
           List of relations  
 Schema |    Name    | Type  |  Owner  
--------+------------+-------+----------  
 public | test_table | table | postgres  
(1 row)  

>расскажите как вы это сделали и что в итоге получилось.

к выше написанному могу лишь добавить, что второй инстанс СУБД, развернутый во втором контейнере, успешно подхватил файлы, созданные первым инстансом СУБД, который был развернут в первом контейнере

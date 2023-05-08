>Задание со звездочкой*  
>Чем такая схема (витрина+триггер) предпочтительнее отчета, создаваемого "по требованию" (кроме производительности)?  
>Подсказка: В реальной жизни возможны изменения цен.

Есть вопросы к постановке задачи. В задаче не описано, каким образом предполагается отражать в БД изменение цены товара с течением времени. Можно просто редактировать goods.good_price. А можно в goods добавлять новую запись с тем же наименованием товара и новой ценой, тем самым формируя историю изменения цены на товар (пусть даже и без даты изменения).   

В первом случае схема "витрина+триггер" будет не просто предпочтительнее отчета, а будет вообще единственно верной, так как после первого же изменений цены любого проданного товара, отчет всегда уже будет показывать неверную информацию о сумме продаж (запись о продаже есть, но не понятно, какой цена была в момент продажи, так как цена уже изменилась, а история изменения цены не ведется). А раз она единственно верная, то тут и выбора никакого нет, значит, и предпочтения никакого быть не может. Не выбирать же между корректной, но менее производительной реализацией и более производительной, но некорректной? 

Во втором случае вся предпочтительность сводится только к производительности, так как оба варианта (и отчет, и витрина) позволяют (при их должной реализации) всегда получать корректную информацию о сумме продаж (так как записи в goods не редактируются, только добавляются). И тут один единственный критерий выбора — производительность.  

Получается, что в обоих случая задача со звездочкой теряет какой-либо смысл. Что и вызывает вопросы не только к постановке задачи со звездочкой, но и ко всей задаче. Это и есть мое решение задачи со свездочкой. Извините, что аппелирую к условию задачи, которую решал до меня, наверное, уже не один поток, но как есть. Возможно, просто я ее не так понял.

Что касается выполнения самого задания. Не возражаете, если я подкорректирую структуру талицы sales? Основная цель задачи (продемонстриовать полученные на занятии знания триггеров) все равно не пострадает. Ну или переделаю задачу, если потребуется.

Если бы я проектировал структуру этой БД, я просто добавил бы в sales атрибут good_price — цена продажи товара. И заполнял бы его значением goods.good_price в момент продажи (в том же триггере). При таком раскладе я не зависел бы от способа отражения изменения цены товара в goods, так как по sales.good_price я всегда буду знать, по какой цене он был продан. И что отчет, что вирина, будут всегда (при их корректной реализации) отдавать коррекные данные о продажах.

Разворачиваю Postgres в контейнер, запускаю скрипт создания БД      
```
docker run --name pg -e POSTGRES_PASSWORD=1 -d postgres:15
docker exec -t -i pg bash
passwd postgres  
login postgres  
psql  
```
```sql
CREATE SCHEMA pract_functions;

SET search_path = pract_functions, publ;

CREATE TABLE goods
(
    goods_id    integer PRIMARY KEY,
    good_name   varchar(63) NOT NULL,
    good_price  numeric(12, 2) NOT NULL CHECK (good_price > 0.0)
);

INSERT INTO
    goods (goods_id, good_name, good_price)
VALUES 
    (1, 'Спички хозайственные', .50), 
    (2, 'Автомобиль Ferrari FXX K', 185000000.01);

CREATE TABLE sales
(
    sales_id    integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    good_id     integer REFERENCES pract_functions.goods (goods_id),
    sales_time  timestamp with time zone DEFAULT now(),
    sales_qty   integer CHECK (sales_qty > 0),
    good_price  numeric(12, 2) NOT NULL 
);

CREATE FUNCTION fAddGoodPriceToSales() 
RETURNS TRIGGER 
LANGUAGE PLPGSQL
AS 
$$
BEGIN
    RETURN (SELECT ROW(NEW.sales_id, NEW.good_id, NEW.sales_time, NEW.sales_qty, good_price) FROM goods WHERE goods_id=NEW.good_id);
END
$$;

CREATE TRIGGER trgAddGoodPriceToSales 
BEFORE INSERT ON sales
FOR EACH ROW
EXECUTE FUNCTION fAddGoodPriceToSales();

INSERT INTO 
    sales (good_id, sales_qty) 
VALUES 
    (1, 10), (1, 1), (1, 120), (2, 1);
```
проверяем, что триггер работает так, как ожидается
```sql
SELECT * FROM sales;
```
```
 sales_id | good_id |          sales_time           | sales_qty |  good_price
----------+---------+-------------------------------+-----------+--------------
        4 |       1 | 2023-05-08 09:00:14.159474+00 |        10 |         0.50
        5 |       1 | 2023-05-08 09:00:14.159474+00 |         1 |         0.50
        6 |       1 | 2023-05-08 09:00:14.159474+00 |       120 |         0.50
        7 |       2 | 2023-05-08 09:00:14.159474+00 |         1 | 185000000.01
```
все отлично. Запрос для отчета нужно немного поправить. Вот так
```sql
SELECT 
    g.good_name, 
    sum(s.good_price * s.sales_qty)
FROM 
    goods g
INNER JOIN 
    sales s 
ON 
    g.goods_id = s.good_id
GROUP BY 
    g.good_name;
```
```
        good_name         |     sum      
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        65.50
```
занимаемся витриной
```sql
CREATE TABLE good_sum_mart
(
	good_name varchar(63) NOT NULL,
	sum_sale numeric(16,2) NOT NULL
);
```
При добавлении записей в sales ищем товар по good_name в good_sum_mart.  
Если запись найдена, то увеличиваем у нее sum_sale на сумму продажи.  
Если запись не найдена, то добавляем новую запись в good_sum_mart (товар продается впервые).  
```sql
CREATE OR REPLACE FUNCTION fSalesAfterInsert() 
RETURNS TRIGGER 
LANGUAGE PLPGSQL
AS 
$$
DECLARE
    varGoodName goods.good_name%TYPE = (SELECT good_name FROM goods WHERE goods_id=NEW.good_id);
BEGIN
    IF EXISTS (SELECT 1 FROM pract_functions.good_sum_mart WHERE good_name=varGoodName) 
    THEN
        UPDATE good_sum_mart SET sum_sale = sum_sale + NEW.sales_qty * NEW.good_price WHERE good_name=varGoodName;
    ELSE
        INSERT INTO good_sum_mart SELECT varGoodName, NEW.sales_qty * NEW.good_price;
    END IF;
    RETURN NEW;
END
$$;

CREATE TRIGGER trgSalesAfterInsert 
AFTER INSERT ON pract_functions.sales
FOR EACH ROW
EXECUTE FUNCTION fSalesAfterInsert();
```
проверям
```sql
TRUNCATE TABLE sales;

INSERT INTO 
    sales (good_id, sales_qty) 
VALUES 
    (1, 10), (1, 1), (1, 120), (2, 1);

SELECT * FROM good_sum_mart;
```
```
        good_name         |   sum_sale
--------------------------+--------------
 Спички хозайственные     |        65.50
 Автомобиль Ferrari FXX K | 185000000.01
```
вроде все хорошо.

При удалении записей из sales ищем товар по good_name в good_sum_mart.
Если запись найдена, то уменьшаем у нее sum_sale на сумму продажи. 
И удаляем все записи из good_sum_mart по этому товару, у которых sum_sale стала =0.
```sql
CREATE OR REPLACE FUNCTION fSalesAfterDelete() 
RETURNS TRIGGER 
LANGUAGE PLPGSQL
AS 
$$
DECLARE
    varGoodName goods.good_name%TYPE = (SELECT good_name FROM goods WHERE goods_id=OLD.good_id);
BEGIN
    IF EXISTS (SELECT 1 FROM pract_functions.good_sum_mart WHERE good_name=varGoodName) 
    THEN
        UPDATE good_sum_mart SET sum_sale = sum_sale - OLD.sales_qty * OLD.good_price WHERE good_name=varGoodName;
        DELETE FROM good_sum_mart WHERE good_name=varGoodName AND sum_sale=0;
    END IF;
    RETURN OLD;
END
$$;

CREATE TRIGGER trgSalesAfterDelete
AFTER DELETE ON pract_functions.sales
FOR EACH ROW
EXECUTE FUNCTION fSalesAfterDelete();
```
проверям
```sql
DELETE FROM Sales WHERE good_id=1 AND sales_qty=10;

SELECT * FROM good_sum_mart;
```
```
        good_name         |   sum_sale   
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        60.50
```
```sql
DELETE FROM Sales WHERE good_id=2;

SELECT * FROM good_sum_mart;
```
```
      good_name       | sum_sale 
----------------------+----------
 Спички хозайственные |    60.50 
```
вроде все хорошо.  

При обновлении записи в sales.  
Если наименование товара не изменилось, изменилась цена продажи и/или количество, то находим запись по товару в good_sum_mart и увеличиваем sum_sale на разницу между новым и старым значением цены продажи.  
Если наименование товара изменилось, то находим запись по старому наименованию в good_sum_mart и уменьшаем sum_sale на старое значение цены продажи. Далее находим запись по новому наименованию в good_sum_mart и увеличиваем sum_sale на новое значение цены продажи или, если запись не найдена по новому наименованию в good_sum_mart, то добавляем новую запись в good_sum_mart (товар продается впервые).  
```sql
CREATE OR REPLACE FUNCTION fSalesAfterUpdate() 
RETURNS TRIGGER 
LANGUAGE PLPGSQL
AS 
$$
DECLARE
    varOldGoodName goods.good_name%TYPE = (SELECT good_name FROM goods WHERE goods_id=OLD.good_id);
    varNewGoodName goods.good_name%TYPE = (SELECT good_name FROM goods WHERE goods_id=NEW.good_id);
BEGIN
    IF varOldGoodName=varNewGoodName AND (NEW.sales_qty<>OLD.sales_qty OR NEW.good_price<>OLD.good_price)
    THEN
        UPDATE good_sum_mart SET sum_sale = sum_sale + NEW.sales_qty * NEW.good_price - OLD.sales_qty * OLD.good_price WHERE good_name=varOldGoodName;
    ELSEIF varOldGoodName<>varNewGoodName 
        THEN
            UPDATE good_sum_mart SET sum_sale = sum_sale - OLD.sales_qty * OLD.good_price WHERE good_name=varOldGoodName;
            
            IF EXISTS (SELECT 1 FROM pract_functions.good_sum_mart WHERE good_name=varNewGoodName) 
            THEN
                UPDATE good_sum_mart SET sum_sale = sum_sale + NEW.sales_qty * NEW.good_price WHERE good_name=varNewGoodName;
            ELSE
                INSERT INTO good_sum_mart SELECT varNewGoodName, NEW.sales_qty * NEW.good_price;
            END IF;
    END IF;
    RETURN NEW;
END
$$;

CREATE TRIGGER trgSalesAfterUpdate
AFTER UPDATE ON pract_functions.sales
FOR EACH ROW
EXECUTE FUNCTION fSalesAfterUpdate();
```
проверям
```sql
SELECT * FROM good_sum_mart;
```
```
      good_name       | sum_sale
----------------------+----------
 Спички хозайственные |    60.50
 ```
```sql
SELECT * FROM sales;
```
```
 sales_id | good_id |          sales_time           | sales_qty | good_price 
----------+---------+-------------------------------+-----------+------------
       28 |       1 | 2023-05-08 11:37:01.369455+00 |         1 |       0.50 
       29 |       1 | 2023-05-08 11:37:01.369455+00 |       120 |       0.50 
```

Меняем количество у записи с sales_id=28 с 1 на 10. Сумма продаж спичек в витрине должна вырасти на 9х0,5=4,5  
```sql
UPDATE sales SET sales_qty=10 WHERE sales_id=28;

SELECT * FROM good_sum_mart;
```
```
      good_name       | sum_sale 
----------------------+----------
 Спички хозайственные |    65.00
```
все хорошо
```sql
SELECT * FROM sales;
```
```
 sales_id | good_id |          sales_time           | sales_qty | good_price 
----------+---------+-------------------------------+-----------+------------
       29 |       1 | 2023-05-08 11:37:01.369455+00 |       120 |       0.50 
       28 |       1 | 2023-05-08 11:37:01.369455+00 |        10 |       0.50 
```
Меняем цену продажи у записи с sales_id=28 с 0.5 на 5. Сумма продаж спичек в витрине должна вырасти на 10х4,5=45  
```sql
UPDATE sales SET good_price=5 WHERE sales_id=28;

SELECT * FROM good_sum_mart;
```
```
      good_name       | sum_sale 
----------------------+----------
 Спички хозайственные |   110.00
```
все хорошо
```sql
SELECT * FROM sales;
```
```
 sales_id | good_id |          sales_time           | sales_qty | good_price
----------+---------+-------------------------------+-----------+------------
       29 |       1 | 2023-05-08 11:37:01.369455+00 |       120 |       0.50
       28 |       1 | 2023-05-08 11:37:01.369455+00 |        10 |       5.00
```
Меняем good_id у записи с sales_id=29 с 1 на 2. Сумма продаж спичек в витрине должна снизиться на 120х0.5=60 и в витринне должна появиться запись о продаже Ferrari (по цене спичек =). Ну или друими словами, продажи Ferrari вырастут на ту же сумму, на какую упадут продажи спичек.
```sql
UPDATE sales SET good_id=2 WHERE sales_id=29;

SELECT * FROM good_sum_mart;
```
```
        good_name         | sum_sale 
--------------------------+----------
 Спички хозайственные     |    50.00
 Автомобиль Ferrari FXX K |    60.00
```
все хорошо
```sql
SELECT * FROM sales;
```
```
 sales_id | good_id |          sales_time           | sales_qty | good_price 
----------+---------+-------------------------------+-----------+------------
       28 |       1 | 2023-05-08 11:37:01.369455+00 |        10 |       5.00
       29 |       2 | 2023-05-08 11:37:01.369455+00 |       120 |       0.50
```

```sql
TRUNCATE TABLE sales;

SELECT * FROM good_sum_mart;
```
```
        good_name         | sum_sale 
--------------------------+----------
 Спички хозайственные     |    50.00
 Автомобиль Ferrari FXX K |    60.00  
```

упс. Похоже, ДЗ точно есть чем дополнить/уточнить? =)


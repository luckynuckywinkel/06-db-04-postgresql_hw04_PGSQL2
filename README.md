# Домашнее задание к занятию 4. «PostgreSQL», Лебедев А.И., FOPS-10

## Задача 1

Используя Docker, поднимите инстанс PostgreSQL (версию 13). Данные БД сохраните в volume.

Подключитесь к БД PostgreSQL, используя `psql`.

Воспользуйтесь командой `\?` для вывода подсказки по имеющимся в `psql` управляющим командам.

**Найдите и приведите** управляющие команды для:

- вывода списка БД,
- подключения к БД,
- вывода списка таблиц,
- вывода описания содержимого таблиц,
- выхода из psql.

### Решение:  

- Выполнение данного задания я буду производить на той же машине, что я использовал [здесь](https://github.com/luckynuckywinkel/06-db-02-sql_hw02_PGSQL1/).

-  Найдем выполняющие команды из вывода **\?** по списку:

```
- вывода списка БД - \list или \l
- подключения к БД - \connect или \c
- вывода списка таблиц - \dt или \dt+ (более подробно)
- вывода описания содержимого таблиц - \d table_name
- выхода из psql - \quit или \q
```

- Повыполняем эти команды:

```
postgres=# \l
                                 List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges
-----------+----------+----------+------------+------------+-----------------------
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
 test_db   | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
(4 rows)

postgres=# \c test_db
You are now connected to database "test_db" as user "postgres".
test_db=# \dt+
                       List of relations
 Schema |  Name   | Type  |  Owner   |    Size    | Description
--------+---------+-------+----------+------------+-------------
 public | clients | table | postgres | 16 kB      |
 public | orders  | table | postgres | 8192 bytes |
(2 rows)

test_db=# \d clients
                                     Table "public.clients"
  Column  |          Type          | Collation | Nullable |               Default
----------+------------------------+-----------+----------+-------------------------------------
 id       | integer                |           | not null | nextval('clients_id_seq'::regclass)
 lastname | character varying(255) |           |          |
 country  | character varying(255) |           |          |
 purchase | integer                |           |          |
Indexes:
    "clients_pkey" PRIMARY KEY, btree (id)
    "index_country" btree (country)
Foreign-key constraints:
    "clients_purchase_fkey" FOREIGN KEY (purchase) REFERENCES orders(id)

test_db=# \q
#
```

---



## Задача 2

Используя `psql`, создайте БД `test_database`.

Изучите [бэкап БД](https://github.com/netology-code/virt-homeworks/tree/virt-11/06-db-04-postgresql/test_data).

Восстановите бэкап БД в `test_database`.

Перейдите в управляющую консоль `psql` внутри контейнера.

Подключитесь к восстановленной БД и проведите операцию ANALYZE для сбора статистики по таблице.

Используя таблицу [pg_stats](https://postgrespro.ru/docs/postgresql/12/view-pg-stats), найдите столбец таблицы `orders` 
с наибольшим средним значением размера элементов в байтах.

**Приведите в ответе** команду, которую вы использовали для вычисления, и полученный результат.  

### Решение:  

- Восстановим дамп в чистую базу и проверим, что все ок:

```
postgres=# create database test_database;
CREATE DATABASE
postgres=# \l
                                   List of databases
     Name      |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges
---------------+----------+----------+------------+------------+-----------------------
 postgres      | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
 template0     | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
               |          |          |            |            | postgres=CTc/postgres
 template1     | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
               |          |          |            |            | postgres=CTc/postgres
 test_database | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
 test_db       | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
(5 rows)

postgres=# \q
# psql -d test_database -U postgres -f /var/lib/postgresql/backup/test_dump.sql
SET
SET
SET
SET
SET
 set_config
------------

(1 row)

SET
SET
SET
SET
SET
SET
CREATE TABLE
ALTER TABLE
CREATE SEQUENCE
ALTER TABLE
ALTER SEQUENCE
ALTER TABLE
COPY 8
 setval
--------
      8
(1 row)

ALTER TABLE
# psql -U postgres
psql (12.16 (Debian 12.16-1.pgdg110+1))
Type "help" for help.

postgres=# \c test_database
You are now connected to database "test_database" as user "postgres".
test_database=# \dt+
                       List of relations
 Schema |  Name  | Type  |  Owner   |    Size    | Description
--------+--------+-------+----------+------------+-------------
 public | orders | table | postgres | 8192 bytes |
(1 row)
```

- Проведем **ANALYZE** таблицы orders, посмотрим в шпаргалку и составим нужный нам запрос:

```
test_database=# SELECT attname, avg_width
FROM pg_stats
WHERE tablename = 'orders'
ORDER BY avg_width DESC
LIMIT 5;
 attname | avg_width
---------+-----------
 title   |        16
 id      |         4
 price   |         4
(3 rows)
```

---

## Задача 3

Архитектор и администратор БД выяснили, что ваша таблица orders разрослась до невиданных размеров и
поиск по ней занимает долгое время. Вам как успешному выпускнику курсов DevOps в Нетологии предложили
провести разбиение таблицы на 2: шардировать на orders_1 - price>499 и orders_2 - price<=499.

Предложите SQL-транзакцию для проведения этой операции.

Можно ли было изначально исключить ручное разбиение при проектировании таблицы orders?  

### Решение:    

- Прочитав про методы шардирования в данном простом случае, я думаю, что нам нужно сделать следующее - создать две дополнительные таблицы orders_over_400 и order_less_400 (да, названия громоздкие, но, так лучше для понимания), составить запрос для переноса нужных данных, добавить индексы (опционально, если мы рассматриваем еще большее ускорение поиска по столбцу price), удаление старой таблицы orders.

- Попробуем проделать все написанное выше:

```
test_database=# create table orders_over_400 (like orders);
CREATE TABLE
test_database=# create table orders_less_400 (like orders);
CREATE TABLE
test_database=# insert into orders_over_400 select * from orders where price > 400;
INSERT 0 4
test_database=# insert into orders_less_400 select * from orders where price <= 400;
INSERT 0 4
test_database=# create index over_400 ON orders_over_400 (price);
CREATE INDEX
test_database=# create index less_400 ON orders_less_400 (price);
CREATE INDEX
test_database=# drop table orders;
test_database=# \dt
              List of relations
 Schema |      Name       | Type  |  Owner
--------+-----------------+-------+----------
 public | orders_less_400 | table | postgres
 public | orders_over_400 | table | postgres
(2 rows)
```

- Перечитал задание и там сказано, что нужно делать все одной транзакцией. Т.к. я уже все сделал запросами построчно, то я не буду переделывать, а просто наришу, как выглядела бы транзакция:

```
BEGIN;
CREATE TABLE orders_over_400 (LIKE orders);
CREATE TABLE orders_less_400 (LIKE orders);
INSERT INTO orders_over_400 SELECT * FROM orders WHERE price > 499;
INSERT INTO orders_less_400 SELECT * FROM orders WHERE price <= 499;
CREATE INDEX over_400 ON orders_over_400 (price);
CREATE INDEX less_400 ON orders_less_400 (price);
DROP TABLE orders;
COMMIT;
```

- Можно ли было изначально исключить ручное разбиение при проектировании таблицы orders - думаю, да. В зависимости от того, как изначально ставилась задача. Если заказчик предполагал, что таблица будет расти быстро, то вариант масштабирования должен был быть заложен перед созданием таблицы. Думаю, так было бы сложнее на начальном этапе, конечно.

---







## Задача 4

Используя утилиту `pg_dump`, создайте бекап БД `test_database`.

Как бы вы доработали бэкап-файл, чтобы добавить уникальность значения столбца `title` для таблиц `test_database`?  

### Решение:  

- Создадим backup в sql-файл:

```
# pg_dump -U postgres -d test_database -f /var/lib/postgresql/backup/test_database_dump.sql
```

- Для начала, я прочел, что такое **UNIQUE** и как его можно определить для столбца. Определить его можно при помощи добавления некоего ограничения (**ADD CONSTRAINT**), коим для нас и будет ограничение Unique - то есть, все значения в столбце должны быть уникальными и не должны повторяться.

- Т.к. мы имеем бэкап в sql-формате, мы запросто можем его редактировать, как нам нужно. В нашем случае, мы добавим следюующую строку в наш дамп:

```
CREATE TABLE public.orders_less_400 (
    id integer NOT NULL,
    title character varying(80) NOT NULL,
    price integer,
    CONSTRAINT unique_less_400 UNIQUE (title)  <-- вот это раз
);

ALTER TABLE public.orders_less_400 OWNER TO postgres;
....
CREATE TABLE public.orders_over_400 (
    id integer NOT NULL,
    title character varying(80) NOT NULL,
    price integer,
    CONSTRAINT unique_over_400 UNIQUE (title)  <-- вот это два
);

ALTER TABLE public.orders_over_400 OWNER TO postgres;
---



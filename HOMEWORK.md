- [hw1](#hw1)
- [hw2](#hw2)
- [hw3](#hw3)

# hw1

```bash
psql -h localhost -p 5432 -U postgres
psql -h localhost -p 5432 -U postgres <thai.sql

echo "select count(*) from book.tickets" | psql -h localhost -p 5432 -U postgres -d thai
#   count
# ---------
#  5185505
# (1 row)
```

Результат: 5185505

# hw2

```bash
# 1-3 task
psql -h localhost -p 5432 -U postgres
```

```sql
-- 4-5 tasks
create table hw2_test ( id int );

insert into hw2_test
select gs.number
from generate_series(1, 10) as gs(number);

show transaction_isolation ;
--  transaction_isolation
-- -----------------------
--  read committed
-- (1 row)
```

```sql
-- 6-9 tasks

begin; -- 1 window
begin; -- 2 window
insert into hw2_test values (1); -- 1 window

select * from hw2_test; -- 2 window
 id
----
  1
  2
  3
  4
  5
  6
  7
  8
  9
 10
(10 rows)
```

> 9. видите ли вы новую запись и если да то почему?

Нет не видим, потому что у нас невозможна аномалия dirty read на дефолтном уровне изоляции.

```sql
-- 10 task
commit; -- 1 window

-- 11 task
select * from hw2_test;
 id
----
  1
  2
  3
  4
  5
  6
  7
  8
  9
 10
  1
(11 rows)


-- 13 task
commit; -- 1 window
commit; -- 2 window
```

>12. видите ли вы новую запись и если да то почему?

Да, новая запись видна, потому что на дефолтном уровне изоляции возможна анамалия non repatable read.

```sql
-- 14 task
begin transaction isolation level repeatable read ; -- 1 window
begin transaction isolation level repeatable read ; -- 2 window

-- 15 task
insert into hw2_test values (100500); -- 1 window

-- 16 task
select * from hw2_test; -- 2 window
 id
----
  1
  2
  3
  4
  5
  6
  7
  8
  9
 10
  1
(11 rows)
```

> 17. видите ли вы новую запись и если да то почему?

Нет, потому что уровень изоляции транзакций не допускает аномалию dirty read.

```sql
-- 18 task
commit; -- 1 window

-- 19 task
select * from hw2_test;
 id
----
  1
  2
  3
  4
  5
  6
  7
  8
  9
 10
  1
(11 rows)
```

> 20. видите ли вы новую запись и если да то почему?

Нет, потому что уровень изоляции транзакций не допускает аномалию non repeatable read.

# hw3

```sql
-- 1.Создать таблицу с текстовым полем и заполнить случайными или сгенерированными данным в размере 1 млн строк
CREATE TABLE hw3_data(
  id serial,
  data varchar
);

INSERT INTO hw3_data
SELECT id, md5(random()::text) FROM generate_series(1, 1000000) as id;

-- 2. Посмотреть размер файла с таблицей
SELECT pg_relation_filepath('hw3_data');
-- pg_relation_filepath
----------------------
-- base/5/16398
--(1 row)
```

```bash
docker exec -ti wb-pg-postgresql-1  du -h /bitnami/postgresql/data/base/5/16398
# 66M     /bitnami/postgresql/data/base/5/16398
```

```sql
-- 3. 5 раз обновить все строчки и добавить к каждой строчке любой символ
DO $$
DECLARE
  i INT;
BEGIN
  FOR i IN 1..5 LOOP
    UPDATE hw3_data SET data = data || substr(md5(random()::text), 1, 1);
  END LOOP;
END $$;

-- 4. Посмотреть количество мертвых строчек в таблице и когда последний раз приходил автовакуум
SELECT n_dead_tup, last_autovacuum
FROM pg_stat_all_tables
WHERE relname = 'hw3_data';
--  n_dead_tup |        last_autovacuum
-- ------------+-------------------------------
--     5000000 | 2024-10-20 12:20:07.090561+00

-- 5. Подождать некоторое время, проверяя, пришел ли автовакуум
SELECT n_dead_tup, last_autovacuum
FROM pg_stat_all_tables
WHERE relname = 'hw3_data';
--  n_dead_tup |        last_autovacuum
-- ------------+-------------------------------
--           0 | 2024-10-20 12:22:20.881034+00

-- 6. 5 раз обновить все строчки и добавить к каждой строчке любой символ
DO $$
DECLARE
  i INT;
BEGIN
  FOR i IN 1..5 LOOP
    UPDATE hw3_data SET data = data || substr(md5(random()::text), 1, 1);
  END LOOP;
END $$;
```

```bash
# 7. Посмотреть размер файла с таблицей
docker exec -ti wb-pg-postgresql-1  du -h /bitnami/postgresql/data/base/5/16398
# 439M    /bitnami/postgresql/data/base/5/16398
```

```sql
-- 8. Отключить Автовакуум на конкретной таблице
ALTER TABLE hw3_data SET (autovacuum_enabled = false);

-- 9. 10 раз обновить все строчки и добавить к каждой строчке любой символ
DO $$
DECLARE
  i INT;
BEGIN
  FOR i IN 1..10 LOOP
    UPDATE hw3_data SET data = data || substr(md5(random()::text), 1, 1);
  END LOOP;
END $$;
```

```bash
# 10. Посмотреть размер файла с таблицей
docker exec -ti wb-pg-postgresql-1  du -h /bitnami/postgresql/data/base/5/16398
# 880M    /bitnami/postgresql/data/base/5/16398
```

> 11. Объясните полученный результат

Когда мы получили 440M после 5 кратного изменения, в файле было выделено места для хранения 6 версий строк. На сколько я понимаю, postgres не отдает страницы выделенные в файле до выполнения `vacuum full`. Соответственно когда мы отключили `vacuum` и запустили десятикратное обновлени, в файле хранилось 11 версий строк (1 текущая и 10 предыдущих). После запуска автовакуум осовбодил старые версии строк, но это не повлияло на размер файла на диске. Дополнительно запустил после этого `vacuum full`, в результате чего табличка переместилась в другой файл и ее размер существенно сжался до `90M`.

```sql
-- 12. Не забудьте включить автовакуум)
ALTER TABLE hw3_data SET (autovacuum_enabled = true);
```

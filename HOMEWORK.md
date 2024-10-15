- [hw1](#hw1)
- [hw2](#hw2)

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

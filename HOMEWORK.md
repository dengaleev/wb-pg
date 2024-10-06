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

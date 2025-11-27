```
Окружение: 2 ВМ Oracle Linux 8.10, 1 ВМ РЕД ОС 7.3.5.
```

# Репликация.

### Подготовка на первой ВМ.
Создаю необходимые таблицы:
```sql
CREATE TABLE test (id SERIAL PRIMARY KEY, data TEXT, created_at TIMESTAMP DEFAULT NOW());
CREATE TABLE test2 (id SERIAL PRIMARY KEY, data TEXT, created_at TIMESTAMP DEFAULT NOW());
```
Настраиваю логическую репликацию:
```sql
ALTER SYSTEM SET wal_level = logical;
ALTER SYSTEM SET max_replication_slots = 10;
ALTER SYSTEM SET max_wal_senders = 10;
```
Перезапускаю PostgreSQL, возвращаюсь в psql, создаю публикацию для таблицы test:
```sql
CREATE PUBLICATION pub_test FOR TABLE test;
```
Создаю пользователя под репликацию:
```sql
CREATE USER repl_user WITH REPLICATION LOGIN PASSWORD '12345';
```
Даю права на таблицы:
```sql
GRANT SELECT ON test, test2 TO repl_user;
GRANT USAGE ON SCHEMA public TO repl_user;
```
Проверяю публикацию:
```sql
SELECT * FROM pg_publication;

  oid  | pubname  | pubowner | puballtables | pubinsert | pubupdate | pubdelete | pubtruncate | pubviaroot
-------+----------+----------+--------------+-----------+-----------+-----------+-------------+------------
 65666 | pub_test |       10 | f            | t         | t         | t         | t           | f
```

### Подготовка на второй ВМ.

Создаю таблицы, настраиваю логическую репликацию, :
```sql
CREATE TABLE test (id SERIAL PRIMARY KEY, data TEXT, created_at TIMESTAMP DEFAULT NOW());
CREATE TABLE test2 (id SERIAL PRIMARY KEY, data TEXT, created_at TIMESTAMP DEFAULT NOW());

ALTER SYSTEM SET wal_level = logical;
ALTER SYSTEM SET max_replication_slots = 10;
ALTER SYSTEM SET max_wal_senders = 10;
```
Перезагружаю PostgreSQL, создаю публикацию для test2, создаю пользователя под публикацию:
```sql
CREATE PUBLICATION pub_test2 FOR TABLE test2;
CREATE USER repl_user WITH REPLICATION LOGIN PASSWORD '12345';
GRANT SELECT ON test, test2 TO repl_user;
GRANT USAGE ON SCHEMA public TO repl_user;
```

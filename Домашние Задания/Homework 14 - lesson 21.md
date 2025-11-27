```
Окружение: 2 ВМ Oracle Linux 8.10, 1 ВМ РЕД ОС 7.3.5. PostgreSQL 17.
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

### Подготовка на третьей ВМ. 

Создаю таблицы:
```sql
CREATE TABLE test (id SERIAL PRIMARY KEY, data TEXT, created_at TIMESTAMP DEFAULT NOW());
CREATE TABLE test2 (id SERIAL PRIMARY KEY, data TEXT, created_at TIMESTAMP DEFAULT NOW());
```

### Настройка подписок. 
На второй ВМ подписываюсь на первую ВМ, проверяю подписку:
```sql
CREATE SUBSCRIPTION sub_test
CONNECTION 'host=тут_айпишник_первой_вм port=5432 user=repl_user password=12345 dbname=postgres'
PUBLICATION pub_test;
ЗАМЕЧАНИЕ:  на сервере публикации создан слот репликации "sub_test"

SELECT * FROM pg_subscription;
  oid  | subdbid | subskiplsn | subname  | subowner | subenabled | subbinary | substream | subtwophasestate | subdisableonerr | subpasswordrequired | s
ubrunasowner | subfailover |                               subconninfo                                | subslotname | subsynccommit | subpublications |
 suborigin
-------+---------+------------+----------+----------+------------+-----------+-----------+------------------+-----------------+---------------------+--
-------------+-------------+--------------------------------------------------------------------------+-------------+---------------+-----------------+
-----------
 16413 |       5 | 0/0        | sub_test |       10 | t          | f         | f         | d                | f               | t                   | f
             | f           | host=не_покажу_айпишник port=5432 user=repl_user password=12345 dbname=postgres | sub_test    | off           | {pub_test}      |
 any
(1 строка)
```
На первой ВМ подписываюсь на вторую ВМ, проверяю подписку:
```sql
CREATE SUBSCRIPTION sub_test2
CONNECTION 'host=айпишник_от_второй_вм port=5432 user=repl_user password=12345 dbname=postgres'
PUBLICATION pub_test2;
ЗАМЕЧАНИЕ:  на сервере публикации создан слот репликации "sub_test2"

SELECT * FROM pg_subscription;

  oid  | subdbid | subskiplsn |  subname  | subowner | subenabled | subbinary | substream | subtwophasestate | subdisableonerr | subpasswordrequired |
subrunasowner | subfailover |                               subconninfo                                | subslotname | subsynccommit | subpublications
| suborigin
-------+---------+------------+-----------+----------+------------+-----------+-----------+------------------+-----------------+---------------------+-
--------------+-------------+--------------------------------------------------------------------------+-------------+---------------+-----------------
+-----------
 65670 |       5 | 0/0        | sub_test2 |       10 | t          | f         | f         | d                | f               | t                   |
f             | f           | host=не_покажу_айпишник port=5432 user=repl_user password=12345 dbname=postgres | sub_test2   | off           | {pub_test2}
| any
(1 строка)
```
На третьей ВМ подписываюсь на первую ВМ и вторую ВМ:
```sql
CREATE SUBSCRIPTION sub_test_vm1
CONNECTION 'host=айпишник_первой_вм port=5432 user=repl_user password=12345 dbname=postgres'
PUBLICATION pub_test;
ЗАМЕЧАНИЕ:  на сервере публикации создан слот репликации "sub_test_vm1"

CREATE SUBSCRIPTION sub_test2_vm2
CONNECTION 'host=айпишник_второй_вм port=5432 user=repl_user password=12345 dbname=postgres'
PUBLICATION pub_test2;
ЗАМЕЧАНИЕ:  на сервере публикации создан слот репликации "sub_test2_vm2"
```

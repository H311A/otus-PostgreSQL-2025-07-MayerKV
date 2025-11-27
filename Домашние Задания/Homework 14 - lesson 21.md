```
Окружение: 2 ВМ Oracle Linux 8.10, 2 ВМ РЕД ОС 7.3.5. PostgreSQL 17.
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

### Проверка работы. 
Тестирую запись на первой ВМ:
```sql
INSERT INTO test (data) VALUES ('Data from VM1');
INSERT 0 1
```
Проверяю на других ВМ:
```sql
-- На второй ВМ
SELECT * FROM test;

 id |     data      |         created_at
----+---------------+----------------------------
  1 | Data from VM1 | 2025-11-27 12:02:43.458331
(1 строка)

-- На третьей ВМ
SELECT * FROM test;

 id |     data      |         created_at
----+---------------+----------------------------
  1 | Data from VM1 | 2025-11-27 12:02:43.458331
(1 строка)
```
Тестирую запись на второй ВМ:
```sql
INSERT INTO test2 (data) VALUES ('Data from VM2');
INSERT 0 1
```
Проверяю на других ВМ:
```sql
-- На первой ВМ
SELECT * FROM test2;

 id |     data      |         created_at
----+---------------+----------------------------
  1 | Data from VM2 | 2025-11-27 12:04:51.917679
(1 строка)

-- На третьей ВМ
SELECT * FROM test2;

 id |     data      |         created_at
----+---------------+----------------------------
  1 | Data from VM2 | 2025-11-27 12:04:51.917679
(1 строка)
```

### Задача со звёздочкой. 
Подготавливаю третью ВМ:
```sql
-- Временно останавливаю подписки
ALTER SUBSCRIPTION sub_test_vm1 DISABLE;
ALTER SUBSCRIPTION sub_test2_vm2 DISABLE;

-- Настраиваю физическую репликацию
ALTER SYSTEM SET wal_level = replica;
ALTER SYSTEM SET max_wal_senders = 10;
ALTER SYSTEM SET max_replication_slots = 10;
ALTER SYSTEM SET hot_standby = on;
```
Перезапускаю PostgreSQL, создаю пользователя:
```sql
CREATE USER physical_repl WITH REPLICATION LOGIN PASSWORD '12345';
```
Настраиваю ph_hba.conf:
```sql
echo "host replication physical_repl 0.0.0.0/0 md5" | sudo tee -a /var/lib/pgsql/17/data/pg_hba.conf
```
Останавливаю PostgreSQL на 4 ВМ, очищаю data directory, создаю бекап с ВМ3.
```sql
pg_basebackup -h айпи_третьей_вм -D /var/lib/pgsql/17/data -U physical_repl -P -v -R -W
pg_basebackup: начинается базовое резервное копирование, ожидается завершение контрольной точки
pg_basebackup: контрольная точка завершена
pg_basebackup: стартовая точка в журнале предзаписи: 0/2000028 на линии времени 1
pg_basebackup: запуск фонового процесса считывания WAL
pg_basebackup: создан временный слот репликации "pg_basebackup_2592"
24037/24037 КБ (100%), табличное пространство 1/1
pg_basebackup: конечная точка в журнале предзаписи: 0/2000158
pg_basebackup: ожидание завершения потоковой передачи фоновым процессом...
pg_basebackup: сохранение данных на диске...
pg_basebackup: переименование backup_manifest.tmp в backup_manifest
pg_basebackup: базовое резервное копирование завершено
```
Проверяю созданные файлы:
```
[postgres@ddmodule ~]$ ls -la /var/lib/pgsql/17/data/standby.signal
-rw-------. 1 postgres postgres 0 ноя 27 13:51 /var/lib/pgsql/17/data/standby.signal
[postgres@ddmodule ~]$ cat /var/lib/pgsql/17/data/postgresql.auto.conf
# Do not edit this file manually!
# It will be overwritten by the ALTER SYSTEM command.
wal_level = 'replica'
max_wal_senders = '10'
max_replication_slots = '10'
hot_standby = 'on'
primary_conninfo = 'user=physical_repl password=12345 channel_binding=prefer host=айпи_3_вм port=5432 sslmode=prefer sslnegotiation=postgres sslcompression=0 sslcertmode=allow sslsni=1 ssl_min_protocol_version=TLSv1.2 gssencmode=prefer krbsrvname=postgres gssdelegation=0 target_session_attrs=any load_balance_hosts=disable'
```
Запускаю PostgreSQL. 
### Проверка репликации.
На третьей ВМ проверяю:
```sql
SELECT application_name, client_addr, state, sync_state, 
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn)) as lag_size
FROM pg_stat_replication;

 application_name | client_addr |   state   | sync_state | lag_size
------------------+-------------+-----------+------------+----------
 walreceiver      | айпи_4_вм | streaming | async      | 0 bytes
(1 строка)
```
Проверяю на 4 ВМ:
```
-- Что мы в режиме реплики
SELECT pg_is_in_recovery();
 pg_is_in_recovery
-------------------
 t

-- Статус репликации
  status   |      last_msg_send_time       |     last_msg_receipt_time     | latest_end_lsn | slot_name
-----------+-------------------------------+-------------------------------+----------------+-----------
 streaming | 2025-11-27 13:56:24.793485+03 | 2025-11-27 13:56:41.042652+03 | 0/3000168      |
```
### Тестирую репликацию:
Записываю на 3 ВМ и проверяю на 4 ВМ:
```sql
-- На 3 ВМ
CREATE TABLE hot_standby_test (id SERIAL PRIMARY KEY, data TEXT, created_at TIMESTAMP DEFAULT NOW());
INSERT INTO hot_standby_test (data) VALUES ('Test data from VM3 master');
INSERT INTO hot_standby_test (data) VALUES ('Second record from VM3');

-- На 4 ВМ
SELECT * FROM hot_standby_test ORDER BY id;

 id |           data            |         created_at
----+---------------------------+----------------------------
  1 | Test data from VM3 master | 2025-11-27 13:57:49.703581
  2 | Second record from VM3    | 2025-11-27 13:57:50.571242
```
Проверяю чтение на реплике:
```sql
INSERT INTO hot_standby_test (data) VALUES ('This should fail on standby');
ОШИБКА:  в транзакции в режиме "только чтение" нельзя выполнить INSERT
```
Всё корректно.   
Тест переключения.  
Останавливаю третью ВМ, проверяю на четвертой ВМ, что она продолжает работать в режиме чтения:
```sql
postgres=# SELECT now(), pg_is_in_recovery();

              now              | pg_is_in_recovery
-------------------------------+-------------------
 2025-11-27 14:10:02.264533+03 | t
```

#### Скрипт для мониторинга. 
```bash
#!/bin/bash
PGPASSWORD="12345"

echo "=== MASTER ==="
psql -h айпи_мастер_ноды (3 вм в моем случае) -U postgres -d postgres -c "
SELECT 
    application_name,
    client_addr,
    state,
    sync_state,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn)) as replication_lag
FROM pg_stat_replication;"

echo ""
echo "=== STANDBY ==="
psql -h айпи_реплики (4 вм в моем случае) -U postgres -d postgres -c "
SELECT 
    pg_is_in_recovery() as is_standby,
    status as wal_receiver_status,
    latest_end_lsn,
    now() as check_time
FROM pg_stat_wal_receiver;"
```

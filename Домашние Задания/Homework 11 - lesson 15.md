```
Окружение: Oracle Linux 8.10. PostgreSQL 17.
```
# Секционирование

## Анализ структуры данных
1. Структура таблицы `tickets`:
```sql
demo=# \d tickets
                                   Таблица "bookings.tickets"
    Столбец     |          Тип          | Правило сортировки | Допустимость NULL | По умолчанию
----------------+-----------------------+--------------------+-------------------+--------------
 ticket_no      | character(13)         |                    | not null          |
 book_ref       | character(6)          |                    | not null          |
 passenger_id   | character varying(20) |                    | not null          |
 passenger_name | text                  |                    | not null          |
 contact_data   | jsonb                 |                    |                   |
Индексы:
    "tickets_pkey" PRIMARY KEY, btree (ticket_no)
Ограничения внешнего ключа:
    "tickets_book_ref_fkey" FOREIGN KEY (book_ref) REFERENCES bookings(book_ref)
Ссылки извне:
    TABLE "ticket_flights" CONSTRAINT "ticket_flights_ticket_no_fkey" FOREIGN KEY (ticket_no) REFERENCES tickets(ticket_no)
```
2. Структура таблицы `ticket_flights`:
```sql
demo=# \d ticket_flights
                                Таблица "bookings.ticket_flights"
     Столбец     |          Тип          | Правило сортировки | Допустимость NULL | По умолчанию
-----------------+-----------------------+--------------------+-------------------+--------------
 ticket_no       | character(13)         |                    | not null          |
 flight_id       | integer               |                    | not null          |
 fare_conditions | character varying(10) |                    | not null          |
 amount          | numeric(10,2)         |                    | not null          |
Индексы:
    "ticket_flights_pkey" PRIMARY KEY, btree (ticket_no, flight_id)
Ограничения-проверки:
    "ticket_flights_amount_check" CHECK (amount >= 0::numeric)
    "ticket_flights_fare_conditions_check" CHECK (fare_conditions::text = ANY (ARRAY['Economy'::character varying::text, 'Comfort'::character varying::text, 'Business'::character varying::text]))
Ограничения внешнего ключа:
    "ticket_flights_flight_id_fkey" FOREIGN KEY (flight_id) REFERENCES flights(flight_id)
    "ticket_flights_ticket_no_fkey" FOREIGN KEY (ticket_no) REFERENCES tickets(ticket_no)
Ссылки извне:
    TABLE "boarding_passes" CONSTRAINT "boarding_passes_ticket_no_fkey" FOREIGN KEY (ticket_no, flight_id) REFERENCES ticket_flights(ticket_no, flight_id)
```
3. Структура таблицы `boarding_passes`:
```sql
demo=# \d boarding_passes
                             Таблица "bookings.boarding_passes"
   Столбец   |         Тип          | Правило сортировки | Допустимость NULL | По умолчанию
-------------+----------------------+--------------------+-------------------+--------------
 ticket_no   | character(13)        |                    | not null          |
 flight_id   | integer              |                    | not null          |
 boarding_no | integer              |                    | not null          |
 seat_no     | character varying(4) |                    | not null          |
Индексы:
    "boarding_passes_pkey" PRIMARY KEY, btree (ticket_no, flight_id)
    "boarding_passes_flight_id_boarding_no_key" UNIQUE CONSTRAINT, btree (flight_id, boarding_no)
    "boarding_passes_flight_id_seat_no_key" UNIQUE CONSTRAINT, btree (flight_id, seat_no)
Ограничения внешнего ключа:
    "boarding_passes_ticket_no_fkey" FOREIGN KEY (ticket_no, flight_id) REFERENCES ticket_flights(ticket_no, flight_id)
``` 
4. Структура таблицы `seats`:
```sql
demo=# \d seats
                                    Таблица "bookings.seats"
     Столбец     |          Тип          | Правило сортировки | Допустимость NULL | По умолчанию
-----------------+-----------------------+--------------------+-------------------+--------------
 aircraft_code   | character(3)          |                    | not null          |
 seat_no         | character varying(4)  |                    | not null          |
 fare_conditions | character varying(10) |                    | not null          |
Индексы:
    "seats_pkey" PRIMARY KEY, btree (aircraft_code, seat_no)
Ограничения-проверки:
    "seats_fare_conditions_check" CHECK (fare_conditions::text = ANY (ARRAY['Economy'::character varying::text, 'Comfort'::character varying::text, 'Business'::character varying::text]))
Ограничения внешнего ключа:
    "seats_aircraft_code_fkey" FOREIGN KEY (aircraft_code) REFERENCES aircrafts_data(aircraft_code) ON DELETE CASCADE
```
5. Структура таблицы `airports`:
```sql
demo=# \d airports
                          Представление "bookings.airports"
   Столбец    |     Тип      | Правило сортировки | Допустимость NULL | По умолчанию
--------------+--------------+--------------------+-------------------+--------------
 airport_code | character(3) |                    |                   |
 airport_name | text         |                    |                   |
 city         | text         |                    |                   |
 coordinates  | point        |                    |                   |
 timezone     | text         |                    |                   |
```
6. Структура таблицы `aircrafts`:
```sql
demo=# \d aircrafts
                          Представление "bookings.aircrafts"
    Столбец    |     Тип      | Правило сортировки | Допустимость NULL | По умолчанию
---------------+--------------+--------------------+-------------------+--------------
 aircraft_code | character(3) |                    |                   |
 model         | text         |                    |                   |
 range         | integer      |                    |                   |
```
7. Структура таблицы `bookings`:
```sql
demo=# \d bookings
                                   Таблица "bookings.bookings"
   Столбец    |           Тип            | Правило сортировки | Допустимость NULL | По умолчанию
--------------+--------------------------+--------------------+-------------------+--------------
 book_ref     | character(6)             |                    | not null          |
 book_date    | timestamp with time zone |                    | not null          |
 total_amount | numeric(10,2)            |                    | not null          |
Индексы:
    "bookings_pkey" PRIMARY KEY, btree (book_ref)
Ссылки извне:
    TABLE "tickets" CONSTRAINT "tickets_book_ref_fkey" FOREIGN KEY (book_ref) REFERENCES bookings(book_ref)
```
8. Структура таблицы `flights`:
```sql
demo=# \d flights
                                                      Таблица "bookings.flights"
       Столбец       |           Тип            | Правило сортировки | Допустимость NULL |                По умолчанию
---------------------+--------------------------+--------------------+-------------------+--------------------------------------------
 flight_id           | integer                  |                    | not null          | nextval('flights_flight_id_seq'::regclass)
 flight_no           | character(6)             |                    | not null          |
 scheduled_departure | timestamp with time zone |                    | not null          |
 scheduled_arrival   | timestamp with time zone |                    | not null          |
 departure_airport   | character(3)             |                    | not null          |
 arrival_airport     | character(3)             |                    | not null          |
 status              | character varying(20)    |                    | not null          |
 aircraft_code       | character(3)             |                    | not null          |
 actual_departure    | timestamp with time zone |                    |                   |
 actual_arrival      | timestamp with time zone |                    |                   |
Индексы:
    "flights_pkey" PRIMARY KEY, btree (flight_id)
    "flights_flight_no_scheduled_departure_key" UNIQUE CONSTRAINT, btree (flight_no, scheduled_departure)
Ограничения-проверки:
    "flights_check" CHECK (scheduled_arrival > scheduled_departure)
    "flights_check1" CHECK (actual_arrival IS NULL OR actual_departure IS NOT NULL AND actual_arrival IS NOT NULL AND actual_arrival > actual_departure)
    "flights_status_check" CHECK (status::text = ANY (ARRAY['On Time'::character varying::text, 'Delayed'::character varying::text, 'Departed'::character varying::text, 'Arrived'::character varying::text, 'Scheduled'::character varying::text, 'Cancelled'::character varying::text]))
Ограничения внешнего ключа:
    "flights_aircraft_code_fkey" FOREIGN KEY (aircraft_code) REFERENCES aircrafts_data(aircraft_code)
    "flights_arrival_airport_fkey" FOREIGN KEY (arrival_airport) REFERENCES airports_data(airport_code)
    "flights_departure_airport_fkey" FOREIGN KEY (departure_airport) REFERENCES airports_data(airport_code)
Ссылки извне:
    TABLE "ticket_flights" CONSTRAINT "ticket_flights_flight_id_fkey" FOREIGN KEY (flight_id) REFERENCES flights(flight_id)
```

### Проверяю объём данных в основных таблицах:
```sql
SELECT 'bookings' as table_name, COUNT(*) as record_count FROM bookings
UNION ALL SELECT 'tickets', COUNT(*) FROM tickets
UNION ALL SELECT 'flights', COUNT(*) FROM flights  
UNION ALL SELECT 'ticket_flights', COUNT(*) FROM ticket_flights
UNION ALL SELECT 'boarding_passes', COUNT(*) FROM boarding_passes;

   table_name    | record_count
-----------------+--------------
 flights         |        65664
 bookings        |       593433
 tickets         |       829071
 boarding_passes |      1894295
 ticket_flights  |      2360335
(5 строк)
```
### Смотрю диапазон дат в основных таблицах:
```sql
SELECT 
    'bookings' as table, 
    MIN(book_date) as min_date, 
    MAX(book_date) as max_date 
FROM bookings
UNION ALL
SELECT 
    'flights', 
    MIN(scheduled_departure), 
    MAX(scheduled_departure) 
FROM flights;

  table   |        min_date        |        max_date
----------+------------------------+------------------------
 bookings | 2017-04-21 14:23:00+03 | 2017-08-15 18:00:00+03
 flights  | 2017-05-17 02:00:00+03 | 2017-09-14 20:55:00+03
(2 строки)
```

В результате анализа схемы базы данных были изучены следующие таблицы:  
`bookings` (593,433 записей) - содержит данные о бронированиях с полем `book_date` (временная метка);  
`flights` (65,664 записей) - содержит информацию о рейсах с полями `scheduled_departure` и `scheduled_arrival`;  
`tickets` (829,071 записей) - данные о пассажирах и билетах;  
`ticket_flights` (2,360,335 записей) - связующая таблица между билетами и рейсами;  
`boarding_passes` (1,894,295 записей) - посадочные талоны;  
`seats` - конфигурация мест в самолетах;  
`airports` - справочник аэропортов;    
`aircrafts` - справочник самолетов.  

Временные диапазоны данных:  
Бронирования: с 21.04.2017 по 15.08.2017 (4 месяца).  
Рейсы: с 17.05.2017 по 14.09.2017 (4 месяца).  

Кандидаты для секционирования:  
`bookings` - идеальный кандидат для секционирования по диапазону на основе поля `book_date`.  
`flights` - подходит для секционирования по `scheduled_departure`.  
`ticket_flights` - может быть секционирована по связи с рейсами через `flight_id`.  

Таблица `bookings` выбрана для секционирования, так как cодержит исторические данные с четкой временной привязкой, имеет значительный объем данных. Для данной таблицы наиболее подходящим является секционирование по диапазону на основе поля `book_date`, поскольку это позволяет эффективно управлять историческими данными и оптимизировать запросы, фильтруемые по временным периодам. В реальных сценариях часто требуются отчеты за определенные периоды, а так же временные диапазоны хорошо определяются и не пересекаются.

## Секционирование таблицы. 
Теперь анализирую распределение данных по месяцам, чтобы определить границы для секций: 
```sql
SELECT 
    EXTRACT(YEAR FROM book_date) as year,
    EXTRACT(MONTH FROM book_date) as month,
    COUNT(*) as records,
    MIN(book_date) as first_booking,
    MAX(book_date) as last_booking
FROM bookings 
GROUP BY year, month
ORDER BY year, month;

 year | month | records |     first_booking      |      last_booking
------+-------+---------+------------------------+------------------------
 2017 |     4 |    4569 | 2017-04-21 14:23:00+03 | 2017-04-30 23:58:00+03
 2017 |     5 |  163531 | 2017-05-01 00:01:00+03 | 2017-05-31 23:59:00+03
 2017 |     6 |  165150 | 2017-06-01 00:00:00+03 | 2017-06-30 23:59:00+03
 2017 |     7 |  171760 | 2017-07-01 00:00:00+03 | 2017-07-31 23:59:00+03
 2017 |     8 |   88423 | 2017-08-01 00:00:00+03 | 2017-08-15 18:00:00+03
(5 строк)
```
### Создаю новую секционированную таблицу:
```sql
CREATE TABLE bookings_partitioned (
    book_ref character(6) NOT NULL,
    book_date timestamp with time zone NOT NULL,
    total_amount numeric(10,2) NOT NULL
) PARTITION BY RANGE (book_date);
```
### Создаю секции для каждого месяца:
```sql
-- Секция для апреля 2017
CREATE TABLE bookings_2017_04 PARTITION OF bookings_partitioned
    FOR VALUES FROM ('2017-04-01') TO ('2017-05-01');

-- Секция для мая 2017
CREATE TABLE bookings_2017_05 PARTITION OF bookings_partitioned
    FOR VALUES FROM ('2017-05-01') TO ('2017-06-01');

-- Секция для июня 2017
CREATE TABLE bookings_2017_06 PARTITION OF bookings_partitioned
    FOR VALUES FROM ('2017-06-01') TO ('2017-07-01');

-- Секция для июля 2017
CREATE TABLE bookings_2017_07 PARTITION OF bookings_partitioned
    FOR VALUES FROM ('2017-07-01') TO ('2017-08-01');

-- Секция для августа 2017
CREATE TABLE bookings_2017_08 PARTITION OF bookings_partitioned
    FOR VALUES FROM ('2017-08-01') TO ('2017-09-01');
```
### Создаю индексы, аналогичные оригинальной таблице:
```sql
-- Первичный ключ
ALTER TABLE bookings_partitioned ADD PRIMARY KEY (book_ref, book_date);

-- Индекс для ускорения поиска по дате
CREATE INDEX ON bookings_partitioned (book_date);
```
## Миграция данных.
```sql
INSERT INTO bookings_partitioned 
SELECT * FROM bookings;
INSERT 0 593433
```
Проверяю распределение данных по секциям:
```sql
  partition_name  | record_count
------------------+--------------
 bookings_2017_04 |         4569
 bookings_2017_05 |       163531
 bookings_2017_06 |       165150
 bookings_2017_07 |       171760
 bookings_2017_08 |        88423
(5 строк)
```
Сравниваю с оригинальной таблицей:
```sql
SELECT COUNT(*) FROM bookings;
 count
--------
 593433
(1 строка)

SELECT COUNT(*) FROM bookings_partitioned;
 count
--------
 593433
(1 строка)
```
## Оптимизация запросов и тестирование.
1. Запрос за конкретный месяц:
```sql
-- В оригинальной таблице
EXPLAIN (ANALYZE, BUFFERS) 
SELECT * FROM bookings 
WHERE book_date >= '2017-06-01' AND book_date < '2017-07-01';

                                                                     QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------------------------
 Seq Scan on bookings  (cost=0.00..12725.49 rows=167389 width=21) (actual time=0.010..45.256 rows=165150 loops=1)
   Filter: ((book_date >= '2017-06-01 00:00:00+03'::timestamp with time zone) AND (book_date < '2017-07-01 00:00:00+03'::timestamp with time zone))
   Rows Removed by Filter: 428283
   Buffers: shared hit=3824
 Planning:
   Buffers: shared hit=3
 Planning Time: 0.074 ms
 Execution Time: 50.564 ms
(8 строк)

-- В секционированной таблице
EXPLAIN (ANALYZE, BUFFERS) 
SELECT * FROM bookings_partitioned 
WHERE book_date >= '2017-06-01' AND book_date < '2017-07-01';

                                                                     QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------------------------
 Seq Scan on bookings_2017_06 bookings_partitioned  (cost=0.00..3529.25 rows=165150 width=21) (actual time=0.008..15.157 rows=165150 loops=1)
   Filter: ((book_date >= '2017-06-01 00:00:00+03'::timestamp with time zone) AND (book_date < '2017-07-01 00:00:00+03'::timestamp with time zone))
   Buffers: shared hit=1052
 Planning:
   Buffers: shared hit=26
 Planning Time: 0.313 ms
 Execution Time: 20.425 ms
(7 строк)
```
2. Агрегирующий запрос по месяцам.
```sql
-- В оригинальной таблице
EXPLAIN (ANALYZE, BUFFERS)
SELECT 
    EXTRACT(MONTH FROM book_date) as month,
    COUNT(*) as bookings_count,
    SUM(total_amount) as total_revenue
FROM bookings
GROUP BY EXTRACT(MONTH FROM book_date)
ORDER BY month;

                                                                    QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------------------------------
 Finalize GroupAggregate  (cost=36826.35..73557.31 rows=121981 width=72) (actual time=222.765..290.239 rows=5 loops=1)
   Group Key: (EXTRACT(month FROM book_date))
   Buffers: shared hit=3936, temp read=1595 written=1601
   I/O Timings: temp read=4.403 write=16.838
   ->  Gather Merge  (cost=36826.35..69287.97 rows=243962 width=72) (actual time=197.132..290.184 rows=15 loops=1)
         Workers Planned: 2
         Workers Launched: 2
         Buffers: shared hit=3936, temp read=1595 written=1601
         I/O Timings: temp read=4.403 write=16.838
         ->  Partial GroupAggregate  (cost=35826.33..40128.68 rows=121981 width=72) (actual time=144.684..204.957 rows=5 loops=3)
               Group Key: (EXTRACT(month FROM book_date))
               Buffers: shared hit=3936, temp read=1595 written=1601
               I/O Timings: temp read=4.403 write=16.838
               ->  Sort  (cost=35826.33..36444.49 rows=247264 width=38) (actual time=144.195..173.948 rows=197811 loops=3)
                     Sort Key: (EXTRACT(month FROM book_date))
                     Sort Method: external merge  Disk: 6472kB
                     Buffers: shared hit=3936, temp read=1595 written=1601
                     I/O Timings: temp read=4.403 write=16.838
                     Worker 0:  Sort Method: external merge  Disk: 3144kB
                     Worker 1:  Sort Method: external merge  Disk: 3144kB
                     ->  Parallel Seq Scan on bookings  (cost=0.00..6914.80 rows=247264 width=38) (actual time=0.015..67.698 rows=197811 loops=3)
                           Buffers: shared hit=3824
 Planning:
   Buffers: shared hit=12 read=3
   I/O Timings: shared read=0.062
 Planning Time: 0.192 ms
 Execution Time: 291.145 ms
(27 строк)

-- В секционированной таблице

EXPLAIN (ANALYZE, BUFFERS)
SELECT 
    EXTRACT(MONTH FROM book_date) as month,
    COUNT(*) as bookings_count,
    SUM(total_amount) as total_revenue
FROM bookings_partitioned
GROUP BY EXTRACT(MONTH FROM book_date)
ORDER BY month;
                                                                                      QUERY PLAN

-------------------------------------------------------------------------------------------------------------------------------------------------------
-------------------------------
 Finalize GroupAggregate  (cost=12247.93..12301.60 rows=200 width=72) (actual time=163.807..163.878 rows=5 loops=1)
   Group Key: (EXTRACT(month FROM bookings_partitioned.book_date))
   Buffers: shared hit=3797
   ->  Gather Merge  (cost=12247.93..12294.60 rows=400 width=72) (actual time=163.797..163.863 rows=8 loops=1)
         Workers Planned: 2
         Workers Launched: 2
         Buffers: shared hit=3797
         ->  Sort  (cost=11247.91..11248.41 rows=200 width=72) (actual time=155.478..155.481 rows=3 loops=3)
               Sort Key: (EXTRACT(month FROM bookings_partitioned.book_date))
               Sort Method: quicksort  Memory: 25kB
               Buffers: shared hit=3797
               Worker 0:  Sort Method: quicksort  Memory: 25kB
               Worker 1:  Sort Method: quicksort  Memory: 25kB
               ->  Partial HashAggregate  (cost=11237.27..11240.27 rows=200 width=72) (actual time=155.451..155.457 rows=3 loops=3)
                     Group Key: (EXTRACT(month FROM bookings_partitioned.book_date))
                     Batches: 1  Memory Usage: 40kB
                     Buffers: shared hit=3783
                     Worker 0:  Batches: 1  Memory Usage: 40kB
                     Worker 1:  Batches: 1  Memory Usage: 40kB
                     ->  Parallel Append  (cost=0.00..9382.79 rows=247263 width=38) (actual time=0.015..86.829 rows=197811 loops=3)
                           Buffers: shared hit=3783
                           ->  Parallel Seq Scan on bookings_2017_07 bookings_partitioned_4  (cost=0.00..2357.94 rows=101035 width=38) (actual time=0.0
09..32.037 rows=57253 loops=3)
                                 Buffers: shared hit=1095
                           ->  Parallel Seq Scan on bookings_2017_06 bookings_partitioned_3  (cost=0.00..2266.34 rows=97147 width=38) (actual time=0.01
2..22.852 rows=82575 loops=2)
                                 Buffers: shared hit=1052
                           ->  Parallel Seq Scan on bookings_2017_05 bookings_partitioned_2  (cost=0.00..2244.43 rows=96195 width=38) (actual time=0.00
6..41.072 rows=163531 loops=1)
                                 Buffers: shared hit=1042
                           ->  Parallel Seq Scan on bookings_2017_08 bookings_partitioned_5  (cost=0.00..1214.17 rows=52014 width=38) (actual time=0.00
8..22.481 rows=88423 loops=1)
                                 Buffers: shared hit=564
                           ->  Parallel Seq Scan on bookings_2017_04 bookings_partitioned_1  (cost=0.00..63.60 rows=2688 width=38) (actual time=0.013..
1.214 rows=4569 loops=1)
                                 Buffers: shared hit=30
 Planning:
   Buffers: shared hit=14
 Planning Time: 0.304 ms
 Execution Time: 163.948 ms
(35 строк)
```
### Тестирую операции вставки, обновления и удаления.
1. Вставка:
```sql
-- Вставляю тестовую запись
INSERT INTO bookings_partitioned 
VALUES ('999999', '2017-06-15 12:00:00+03', 50000.00);
INSERT 0 1

-- Проверяю, в какую секцию попала запись
SELECT tableoid::regclass as partition_name 
FROM bookings_partitioned 
WHERE book_ref = '999999';

  partition_name
------------------
 bookings_2017_06
(1 строка)
```
2. Обновление:
```sql
-- Меняю дату так, чтобы запись переместилась в другую секцию
UPDATE bookings_partitioned 
SET book_date = '2017-07-15 12:00:00+03' 
WHERE book_ref = '999999';
UPDATE 1

-- Проверяю новую секцию
SELECT tableoid::regclass as partition_name 
FROM bookings_partitioned 
WHERE book_ref = '999999';

  partition_name
------------------
 bookings_2017_07
(1 строка)
```
3. Удаление:
```sql
-- Удаляю тестовую запись
DELETE FROM bookings_partitioned 
WHERE book_ref = '999999';
DELETE 1

-- Проверяю, что удалилась
SELECT COUNT(*) FROM bookings_partitioned WHERE book_ref = '999999';

 count
-------
     0
(1 строка)
```

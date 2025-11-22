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
```
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
```
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
```
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
`bookings` (593,433 записей) - содержит данные о бронированиях с полем book_date (временная метка);
`flights` (65,664 записей) - содержит информацию о рейсах с полями scheduled_departure и scheduled_arrival;
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

Таблица `bookings` выбрана для секционирования, так как cодержит исторические данные с четкой временной привязкой, имеет значительный объем данных. В реальных сценариях часто требуются отчеты за определенные периоды, а так же временные диапазоны хорошо определяются и не пересекаются.

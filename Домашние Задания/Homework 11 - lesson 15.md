```
Окружение: Oracle Linux 8.10. PostgreSQL 17.
```
# Секционирование

## Анализ структуры данных
1. Структура таблицы tickets:
```psql
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
2. Структура таблицы ticket_flights:
```psql
```
3. Структура таблицы boarding_passes:
```psql
``` 
4. Структура таблицы seats:
```psql
```
5. Структура таблицы airports:
```
```
6. Структура таблицы aircrafts:
```
```

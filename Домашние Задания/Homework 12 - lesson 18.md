```
Окружение: Oracle Linux 8.10, PostgreSQL 17.
```
# Триггеры, поддержка заполнения витрин.  
  
## Создаю структуру базы данных:
```sql
DROP SCHEMA IF EXISTS pract_functions CASCADE;
CREATE SCHEMA pract_functions;

SET search_path = pract_functions, public;

-- товары:
CREATE TABLE goods
(
    goods_id    integer PRIMARY KEY,
    good_name   varchar(63) NOT NULL,
    good_price  numeric(12, 2) NOT NULL CHECK (good_price > 0.0)
);
INSERT INTO goods (goods_id, good_name, good_price)
VALUES  (1, 'Спички хозайственные', .50),
        (2, 'Автомобиль Ferrari FXX K', 185000000.01);

-- Продажи
CREATE TABLE sales
(
    sales_id    integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    good_id     integer REFERENCES goods (goods_id),
    sales_time  timestamp with time zone DEFAULT now(),
    sales_qty   integer CHECK (sales_qty > 0)
);

INSERT INTO sales (good_id, sales_qty) VALUES (1, 10), (1, 1), (1, 120), (2, 1);

-- Витрина
CREATE TABLE good_sum_mart
(
    good_name   varchar(63) NOT NULL,
    sum_sale    numeric(16, 2) NOT NULL
);
```
## Наполняю витрину:
```sql
INSERT INTO good_sum_mart (good_name, sum_sale)
SELECT G.good_name, sum(G.good_price * S.sales_qty)
FROM goods G
INNER JOIN sales S ON S.good_id = G.goods_id
GROUP BY G.good_name;
```

## Создаю функции для триггера.
```sql
CREATE OR REPLACE FUNCTION update_good_sum_mart()
RETURNS TRIGGER AS $$
DECLARE
    v_good_name  varchar(63);
    v_good_price numeric(12, 2);
BEGIN
    -- Обработка убытия (DELETE или старое состояние при UPDATE)
    IF (TG_OP = 'DELETE' OR TG_OP = 'UPDATE') THEN
        
        -- Получаем имя и цену товара по OLD.good_id
        SELECT good_name, good_price INTO v_good_name, v_good_price
        FROM goods
        WHERE goods_id = OLD.good_id;

        -- Если товар найден (теоретически может быть удален из goods, но FK не даст), вычитаем сумму из витрины
        IF v_good_name IS NOT NULL THEN
            UPDATE good_sum_mart
            SET sum_sale = sum_sale - (OLD.sales_qty * v_good_price)
            WHERE good_name = v_good_name;
        END IF;
        
    END IF;

    -- Обработка прибытия (INSERT или новое состояние при UPDATE)
    IF (TG_OP = 'INSERT' OR TG_OP = 'UPDATE') THEN
        
        -- Получаем имя и цену товара по NEW.good_id
        SELECT good_name, good_price INTO v_good_name, v_good_price
        FROM goods
        WHERE goods_id = NEW.good_id;

        -- Пытаемся обновить существующую запись в витрине (прибавить сумму)
        UPDATE good_sum_mart
        SET sum_sale = sum_sale + (NEW.sales_qty * v_good_price)
        WHERE good_name = v_good_name;

        -- Если запись не найдена (UPDATE вернул 0 строк) — значит, товара в витрине еще нет.
        -- Вставляем новую запись.
        IF NOT FOUND THEN
            INSERT INTO good_sum_mart (good_name, sum_sale)
            VALUES (v_good_name, (NEW.sales_qty * v_good_price));
        END IF;
        
    END IF;

    RETURN NULL; -- Для AFTER-триггера возвращаемое значение игнорируется
END;
$$ LANGUAGE plpgsql;
```

## Создаю новый триггер. 
```sql
CREATE OR REPLACE TRIGGER sales_mart_trigger
    AFTER INSERT OR UPDATE OR DELETE ON sales
    FOR EACH ROW
    EXECUTE FUNCTION update_good_sum_mart();
```

## Проверяю работу триггера. 
```sql
-- Проверка начального состояния
SELECT * FROM good_sum_mart;

        good_name         |   sum_sale
--------------------------+--------------
 Спички хозайственные     |        65.50
 Автомобиль Ferrari FXX K | 185000000.01
(2 строки)

-- Тестирование INSERT
INSERT INTO sales (good_id, sales_qty) VALUES (1, 5);
SELECT * FROM good_sum_mart; -- Должна увеличиться сумма для спичек

        good_name         |   sum_sale
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        68.00
(2 строки)

-- Тестирование UPDATE
UPDATE sales SET sales_qty = 20 WHERE sales_id = 1;
SELECT * FROM good_sum_mart; -- Должна измениться сумма

        good_name         |   sum_sale
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        73.00
(2 строки)

-- Тестирование DELETE
DELETE FROM sales WHERE sales_id = 2;
SELECT * FROM good_sum_mart; -- Должна уменьшиться сумма

        good_name         |   sum_sale
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        72.50
(2 строки)
```

## Задание со звёздочкой. 

Если мы используем отчет "по требованию", то любое изменение цены товара в таблице `goods` (инфляция, скидки) мгновенно искажает прошлые финансовые показатели, так как база пересчитывает старые продажи по новой текущей цене. Схема "Витрина + Триггер" работает по фиксации факта сделки. Мы положили сумму в витрину в момент продажи по старой цене. Даже если цена в `goods` потом изменится, в витрине останется та сумма, которую мы реально заработали.

Проверяем: 
```sql
-- Сбрасываем данные
TRUNCATE sales RESTART IDENTITY CASCADE;
TRUNCATE good_sum_mart;

-- Возвращаем цену спичек к 0.50
UPDATE goods SET good_price = 0.50 WHERE goods_id = 1;

-- Покупаем 10 коробков. Реальная выручка: 10 * 0.50 = 5.00
INSERT INTO sales (good_id, sales_qty) VALUES (1, 10);

-- Меняем цену в справочнике. Теперь спички стоят 1.00
UPDATE goods SET good_price = 1.00 WHERE goods_id = 1;

-- Сравниваем показания

-- Витрина
SELECT * FROM good_sum_mart;

      good_name       | sum_sale
----------------------+----------
 Спички хозайственные |     5.00
(1 строка)

Отчёт
SELECT G.good_name, sum(G.good_price * S.sales_qty)
FROM goods G
INNER JOIN sales S ON S.good_id = G.goods_id
GROUP BY G.good_name;

      good_name       |  sum
----------------------+-------
 Спички хозайственные | 10.00
(1 строка)
```

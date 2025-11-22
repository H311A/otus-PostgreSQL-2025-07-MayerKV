```
Окружение: Oracle Linux 8.10, PostgreSQL 17.
```
# Триггеры, поддержка заполнения витрин.  
  
## Создаю структуру базы данных:
```
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
    changed_good_id INTEGER;
BEGIN
    -- Определяем ID товара, который изменился
    IF TG_OP = 'DELETE' THEN
        changed_good_id := OLD.good_id;
    ELSE
        changed_good_id := NEW.good_id;
    END IF;
    
    -- Удаляем старую запись этого товара из витрины
    DELETE FROM good_sum_mart 
    WHERE good_name = (SELECT good_name FROM goods WHERE goods_id = changed_good_id);
    
    -- Пересчитываем и вставляем новые данные для этого товара
    INSERT INTO good_sum_mart (good_name, sum_sale)
    SELECT G.good_name, SUM(G.good_price * S.sales_qty)
    FROM goods G
    INNER JOIN sales S ON S.good_id = G.goods_id
    WHERE G.goods_id = changed_good_id
    GROUP BY G.good_name;
    
    RETURN CASE 
        WHEN TG_OP = 'DELETE' THEN OLD 
        ELSE NEW 
    END;
END;
$$ LANGUAGE plpgsql;
```

## Создаю новый триггер. 
```sql
CREATE TRIGGER sales_mart_trigger
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

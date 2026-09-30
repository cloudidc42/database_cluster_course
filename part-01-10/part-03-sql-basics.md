# บทที่ 3: SQL พื้นฐาน — CREATE, INSERT, SELECT, UPDATE, DELETE

> **หลักสูตร:** PostgreSQL + Redis + S3/MinIO Database Cluster  
> **ระดับ:** เริ่มต้น → ขั้นกลาง  
> **เวลาเรียน:** 4-5 ชั่วโมง

---

## สารบัญ

1. [DDL — Data Definition Language](#1-ddl--data-definition-language)
2. [DML — Data Manipulation Language](#2-dml--data-manipulation-language)
3. [DQL — SELECT ทุก Clause](#3-dql--select-ทุก-clause)
4. [WHERE Conditions](#4-where-conditions)
5. [ORDER BY, LIMIT, OFFSET](#5-order-by-limit-offset)
6. [Aggregate Functions](#6-aggregate-functions)
7. [JOIN Types](#7-join-types)
8. [Subqueries](#8-subqueries)
9. [CTEs (Common Table Expressions)](#9-ctes-common-table-expressions)
10. [Window Functions](#10-window-functions)
11. [CASE WHEN](#11-case-when)
12. [String Functions](#12-string-functions)
13. [Date/Time Functions](#13-datetime-functions)
14. [Type Casting](#14-type-casting)
15. [NULL Functions](#15-null-functions)
16. [Workshop: E-Commerce Database](#16-workshop-e-commerce-database)

---

## Workshop Setup: E-Commerce Database

ก่อนเริ่มเรียน SQL ให้สร้าง database สำหรับ workshop ก่อน:

```sql
-- เชื่อมต่อ PostgreSQL
-- docker exec -it pg-main psql -U admin -d mydb

-- สร้าง schema
CREATE SCHEMA IF NOT EXISTS shop;

-- ตาราง categories
CREATE TABLE shop.categories (
    category_id  SERIAL PRIMARY KEY,
    name         VARCHAR(100) NOT NULL,
    slug         VARCHAR(100) UNIQUE NOT NULL,
    parent_id    INTEGER REFERENCES shop.categories(category_id),
    created_at   TIMESTAMPTZ DEFAULT NOW()
);

-- ตาราง products
CREATE TABLE shop.products (
    product_id   SERIAL PRIMARY KEY,
    category_id  INTEGER REFERENCES shop.categories(category_id),
    name         VARCHAR(255) NOT NULL,
    slug         VARCHAR(255) UNIQUE NOT NULL,
    description  TEXT,
    price        DECIMAL(10,2) NOT NULL,
    cost_price   DECIMAL(10,2),
    stock_qty    INTEGER DEFAULT 0,
    is_active    BOOLEAN DEFAULT TRUE,
    created_at   TIMESTAMPTZ DEFAULT NOW(),
    updated_at   TIMESTAMPTZ DEFAULT NOW()
);

-- ตาราง customers
CREATE TABLE shop.customers (
    customer_id  SERIAL PRIMARY KEY,
    first_name   VARCHAR(50) NOT NULL,
    last_name    VARCHAR(50) NOT NULL,
    email        VARCHAR(255) UNIQUE NOT NULL,
    phone        VARCHAR(20),
    birth_date   DATE,
    gender       VARCHAR(10),
    city         VARCHAR(100),
    country      VARCHAR(50) DEFAULT 'Thailand',
    created_at   TIMESTAMPTZ DEFAULT NOW(),
    last_login   TIMESTAMPTZ
);

-- ตาราง orders
CREATE TABLE shop.orders (
    order_id     SERIAL PRIMARY KEY,
    customer_id  INTEGER REFERENCES shop.customers(customer_id),
    status       VARCHAR(20) DEFAULT 'pending',
    total_amount DECIMAL(12,2) NOT NULL,
    shipping_fee DECIMAL(8,2) DEFAULT 0,
    discount     DECIMAL(8,2) DEFAULT 0,
    payment_method VARCHAR(30),
    notes        TEXT,
    created_at   TIMESTAMPTZ DEFAULT NOW(),
    updated_at   TIMESTAMPTZ DEFAULT NOW()
);

-- ตาราง order_items
CREATE TABLE shop.order_items (
    item_id     SERIAL PRIMARY KEY,
    order_id    INTEGER REFERENCES shop.orders(order_id) ON DELETE CASCADE,
    product_id  INTEGER REFERENCES shop.products(product_id),
    quantity    INTEGER NOT NULL,
    unit_price  DECIMAL(10,2) NOT NULL,
    subtotal    DECIMAL(12,2) GENERATED ALWAYS AS (quantity * unit_price) STORED
);

-- ตาราง reviews
CREATE TABLE shop.reviews (
    review_id   SERIAL PRIMARY KEY,
    product_id  INTEGER REFERENCES shop.products(product_id),
    customer_id INTEGER REFERENCES shop.customers(customer_id),
    rating      SMALLINT NOT NULL CHECK (rating BETWEEN 1 AND 5),
    title       VARCHAR(200),
    body        TEXT,
    is_verified BOOLEAN DEFAULT FALSE,
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE (product_id, customer_id)
);

-- ใส่ข้อมูลทดสอบ
INSERT INTO shop.categories (name, slug) VALUES
    ('Electronics', 'electronics'),
    ('Clothing', 'clothing'),
    ('Books', 'books'),
    ('Home & Garden', 'home-garden'),
    ('Sports', 'sports');

INSERT INTO shop.categories (name, slug, parent_id) VALUES
    ('Smartphones', 'smartphones', 1),
    ('Laptops', 'laptops', 1),
    ('T-Shirts', 't-shirts', 2),
    ('Jeans', 'jeans', 2),
    ('Programming', 'programming', 3);

INSERT INTO shop.products (category_id, name, slug, price, cost_price, stock_qty) VALUES
    (6, 'iPhone 15 Pro', 'iphone-15-pro', 42900.00, 30000.00, 50),
    (6, 'Samsung Galaxy S24', 'samsung-galaxy-s24', 35900.00, 25000.00, 75),
    (6, 'Google Pixel 8', 'google-pixel-8', 28900.00, 20000.00, 30),
    (7, 'MacBook Pro M3', 'macbook-pro-m3', 79900.00, 55000.00, 20),
    (7, 'Dell XPS 15', 'dell-xps-15', 62900.00, 45000.00, 15),
    (8, 'Basic White T-Shirt', 'basic-white-tshirt', 299.00, 100.00, 200),
    (8, 'Graphic Tee - Bangkok', 'graphic-tee-bangkok', 599.00, 200.00, 100),
    (9, 'Slim Fit Jeans', 'slim-fit-jeans', 1290.00, 450.00, 80),
    (10, 'Clean Code', 'clean-code', 890.00, 300.00, 45),
    (10, 'The Pragmatic Programmer', 'pragmatic-programmer', 990.00, 350.00, 30);

INSERT INTO shop.customers (first_name, last_name, email, phone, city, birth_date, gender) VALUES
    ('สมชาย', 'ใจดี', 'somchai@email.com', '081-234-5678', 'กรุงเทพ', '1990-05-15', 'M'),
    ('สมหญิง', 'รักสวย', 'somying@email.com', '082-345-6789', 'เชียงใหม่', '1988-08-22', 'F'),
    ('วิชัย', 'เก่งมาก', 'wichai@email.com', '083-456-7890', 'ขอนแก่น', '1995-03-10', 'M'),
    ('นิดา', 'สุขสม', 'nida@email.com', '084-567-8901', 'กรุงเทพ', '1992-11-30', 'F'),
    ('ประยุทธ์', 'มานะ', 'prayut@email.com', NULL, 'ภูเก็ต', '1985-07-04', 'M'),
    ('มะลิ', 'หอมหวาน', 'mali@email.com', '086-789-0123', 'กรุงเทพ', '1998-02-14', 'F'),
    ('เอกชัย', 'ดีงาม', 'eakchai@email.com', '087-890-1234', 'นครราชสีมา', '1991-09-25', 'M'),
    ('รัตนา', 'แจ่มใส', 'rattana@email.com', '088-901-2345', 'กรุงเทพ', '1994-06-18', 'F');

INSERT INTO shop.orders (customer_id, status, total_amount, shipping_fee, payment_method) VALUES
    (1, 'completed', 43000.00, 100.00, 'credit_card'),
    (1, 'completed', 1890.00, 50.00, 'promptpay'),
    (2, 'completed', 36000.00, 100.00, 'credit_card'),
    (3, 'processing', 898.00, 0.00, 'promptpay'),
    (4, 'completed', 80000.00, 0.00, 'credit_card'),
    (4, 'cancelled', 299.00, 50.00, 'cash_on_delivery'),
    (5, 'completed', 29000.00, 100.00, 'bank_transfer'),
    (6, 'pending', 1290.00, 50.00, 'credit_card'),
    (7, 'completed', 63000.00, 0.00, 'credit_card'),
    (8, 'completed', 1980.00, 50.00, 'promptpay');

INSERT INTO shop.order_items (order_id, product_id, quantity, unit_price) VALUES
    (1, 1, 1, 42900.00),   -- iPhone
    (2, 9, 1, 890.00),     -- Clean Code
    (2, 10, 1, 990.00),    -- Pragmatic Programmer
    (3, 2, 1, 35900.00),   -- Samsung
    (4, 9, 1, 890.00),     -- Clean Code (บางส่วน)
    (5, 4, 1, 79900.00),   -- MacBook
    (6, 6, 1, 299.00),     -- T-Shirt (cancelled)
    (7, 3, 1, 28900.00),   -- Pixel
    (8, 8, 1, 1290.00),    -- Jeans
    (9, 5, 1, 62900.00),   -- Dell
    (10, 9, 1, 890.00),
    (10, 10, 1, 990.00),
    (10, 7, 1, 599.00);

INSERT INTO shop.reviews (product_id, customer_id, rating, title, body, is_verified) VALUES
    (1, 1, 5, 'ดีมากๆ', 'กล้องสวย แบตทน แนะนำ', TRUE),
    (2, 3, 4, 'ดีแต่แพง', 'ประสิทธิภาพดี แต่ราคาสูง', TRUE),
    (9, 8, 5, 'หนังสือดีมาก', 'ต้องอ่านถ้าเป็น programmer', TRUE),
    (4, 4, 5, 'เครื่องยอดเยี่ยม', 'M3 chip เร็วมาก', TRUE),
    (6, 6, 3, 'พอใช้', 'ผ้าบางไปนิด', FALSE);
```

---

## 1. DDL — Data Definition Language

### 1.1 CREATE TABLE

```sql
-- Syntax พื้นฐาน
CREATE TABLE [IF NOT EXISTS] schema_name.table_name (
    column_name  data_type  [constraints],
    ...
    [table_constraints]
);

-- ตัวอย่าง: สร้าง table แบบครบ
CREATE TABLE shop.product_variants (
    variant_id      SERIAL PRIMARY KEY,
    product_id      INTEGER NOT NULL REFERENCES shop.products(product_id) ON DELETE CASCADE,
    sku             VARCHAR(50) UNIQUE NOT NULL,
    color           VARCHAR(30),
    size            VARCHAR(10),
    weight_grams    INTEGER,
    price_modifier  DECIMAL(8,2) DEFAULT 0.00,
    stock_qty       INTEGER NOT NULL DEFAULT 0 CHECK (stock_qty >= 0),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    
    -- Composite Unique Constraint
    CONSTRAINT uq_product_color_size UNIQUE (product_id, color, size)
);

-- สร้าง table จาก SELECT (CTAS: Create Table As Select)
CREATE TABLE shop.product_summary AS
    SELECT 
        p.product_id,
        p.name,
        p.price,
        c.name AS category_name,
        COALESCE(AVG(r.rating), 0) AS avg_rating
    FROM shop.products p
    LEFT JOIN shop.categories c ON p.category_id = c.category_id
    LEFT JOIN shop.reviews r ON p.product_id = r.product_id
    GROUP BY p.product_id, p.name, p.price, c.name;

-- สร้าง table โครงสร้างเดียวกัน (ไม่มีข้อมูล)
CREATE TABLE shop.orders_archive (LIKE shop.orders INCLUDING ALL);
```

### 1.2 ALTER TABLE

```sql
-- เพิ่ม column
ALTER TABLE shop.products ADD COLUMN weight_kg DECIMAL(6,3);
ALTER TABLE shop.products ADD COLUMN tags TEXT[] DEFAULT '{}';  -- Array

-- เปลี่ยนชื่อ column
ALTER TABLE shop.products RENAME COLUMN weight_kg TO weight;

-- เปลี่ยน data type
ALTER TABLE shop.products ALTER COLUMN weight TYPE NUMERIC(6,2);

-- เพิ่ม NOT NULL constraint
ALTER TABLE shop.products ALTER COLUMN weight SET NOT NULL;

-- ลบ NOT NULL
ALTER TABLE shop.products ALTER COLUMN weight DROP NOT NULL;

-- เพิ่ม DEFAULT value
ALTER TABLE shop.products ALTER COLUMN weight SET DEFAULT 0.5;

-- ลบ DEFAULT
ALTER TABLE shop.products ALTER COLUMN weight DROP DEFAULT;

-- เพิ่ม constraint
ALTER TABLE shop.products ADD CONSTRAINT chk_price_positive CHECK (price > 0);

-- ลบ constraint
ALTER TABLE shop.products DROP CONSTRAINT chk_price_positive;

-- เพิ่ม index
CREATE INDEX idx_products_category ON shop.products(category_id);

-- ลบ column
ALTER TABLE shop.products DROP COLUMN IF EXISTS weight;

-- เปลี่ยนชื่อ table
ALTER TABLE shop.product_summary RENAME TO product_summary_old;

-- เปลี่ยน schema
ALTER TABLE shop.product_summary_old SET SCHEMA public;
```

### 1.3 DROP TABLE

```sql
-- ลบ table (error ถ้าไม่มี)
DROP TABLE shop.product_summary_old;

-- ลบ table (ปลอดภัยกว่า)
DROP TABLE IF EXISTS shop.product_summary_old;

-- ลบ table และ objects ที่ depend (CASCADE)
DROP TABLE IF EXISTS shop.categories CASCADE;

-- Truncate (ล้างข้อมูล แต่ยัง structure ไว้)
TRUNCATE TABLE shop.reviews;
TRUNCATE TABLE shop.reviews RESTART IDENTITY;  -- reset sequence ด้วย
TRUNCATE TABLE shop.orders, shop.order_items CASCADE;
```

---

## 2. DML — Data Manipulation Language

### 2.1 INSERT

```sql
-- Insert single row
INSERT INTO shop.categories (name, slug) VALUES ('Accessories', 'accessories');

-- Insert multiple rows
INSERT INTO shop.products (category_id, name, slug, price, stock_qty) VALUES
    (1, 'AirPods Pro', 'airpods-pro', 9900.00, 100),
    (1, 'Apple Watch Series 9', 'apple-watch-s9', 14900.00, 60),
    (1, 'iPad Air M2', 'ipad-air-m2', 25900.00, 40);

-- Insert with RETURNING (ได้ค่า generated columns กลับมา)
INSERT INTO shop.customers (first_name, last_name, email)
VALUES ('ทดสอบ', 'ผู้ใช้', 'test@example.com')
RETURNING customer_id, created_at;

-- Insert หรือ Ignore ถ้ามีอยู่แล้ว (ON CONFLICT DO NOTHING)
INSERT INTO shop.categories (name, slug)
VALUES ('Electronics', 'electronics')
ON CONFLICT (slug) DO NOTHING;

-- Insert หรือ Update ถ้ามีอยู่แล้ว (UPSERT)
INSERT INTO shop.products (product_id, name, slug, price, stock_qty)
VALUES (1, 'iPhone 15 Pro Max', 'iphone-15-pro', 49900.00, 45)
ON CONFLICT (slug) DO UPDATE SET
    name = EXCLUDED.name,
    price = EXCLUDED.price,
    stock_qty = EXCLUDED.stock_qty,
    updated_at = NOW();

-- Insert from SELECT
INSERT INTO shop.orders_archive
SELECT * FROM shop.orders
WHERE status IN ('completed', 'cancelled')
  AND created_at < NOW() - INTERVAL '1 year';
```

### 2.2 UPDATE

```sql
-- Update เดี่ยว
UPDATE shop.products
SET price = 39900.00, updated_at = NOW()
WHERE product_id = 2;

-- Update หลาย columns
UPDATE shop.customers
SET 
    phone = '085-000-0000',
    city = 'กรุงเทพ',
    last_login = NOW()
WHERE customer_id = 5;

-- Update ด้วย calculation
UPDATE shop.products
SET price = price * 1.10  -- ขึ้นราคา 10%
WHERE category_id = 1;    -- เฉพาะ Electronics

-- Update ด้วย subquery
UPDATE shop.products
SET stock_qty = stock_qty - oi.quantity
FROM shop.order_items oi
JOIN shop.orders o ON oi.order_id = o.order_id
WHERE shop.products.product_id = oi.product_id
  AND o.status = 'processing';

-- Update with RETURNING
UPDATE shop.products
SET stock_qty = stock_qty - 1
WHERE product_id = 1
RETURNING product_id, name, stock_qty;

-- Update ทั้งหมด (ระวัง! ไม่มี WHERE)
UPDATE shop.products SET updated_at = NOW();
```

### 2.3 DELETE

```sql
-- Delete เดี่ยว
DELETE FROM shop.reviews WHERE review_id = 5;

-- Delete หลาย rows
DELETE FROM shop.order_items
WHERE order_id IN (
    SELECT order_id FROM shop.orders WHERE status = 'cancelled'
);

-- Delete ด้วย JOIN (PostgreSQL syntax)
DELETE FROM shop.order_items oi
USING shop.orders o
WHERE oi.order_id = o.order_id
  AND o.status = 'cancelled'
  AND o.created_at < NOW() - INTERVAL '30 days';

-- Delete with RETURNING
DELETE FROM shop.customers
WHERE email = 'test@example.com'
RETURNING customer_id, email;

-- Delete ทั้งหมด (ระวัง!)
DELETE FROM shop.product_summary_old;  -- ช้ากว่า TRUNCATE แต่ log ได้

-- Soft Delete แทน Hard Delete (แนะนำสำหรับ production)
-- เพิ่ม column deleted_at
ALTER TABLE shop.products ADD COLUMN deleted_at TIMESTAMPTZ;

-- Soft delete
UPDATE shop.products
SET deleted_at = NOW()
WHERE product_id = 10;

-- Query เฉพาะที่ยังไม่ถูกลบ
SELECT * FROM shop.products WHERE deleted_at IS NULL;
```

---

## 3. DQL — SELECT ทุก Clause

### 3.1 SELECT Basics

```sql
-- Select ทุก columns
SELECT * FROM shop.products;

-- Select เฉพาะ columns
SELECT product_id, name, price FROM shop.products;

-- Column aliases
SELECT 
    product_id AS id,
    name AS product_name,
    price AS current_price,
    price * 1.07 AS price_with_vat
FROM shop.products;

-- Distinct values
SELECT DISTINCT city FROM shop.customers ORDER BY city;

-- Distinct on multiple columns
SELECT DISTINCT ON (category_id) category_id, name, price
FROM shop.products
ORDER BY category_id, price DESC;  -- เอาราคาแพงสุดในแต่ละ category
```

### 3.2 ทุก SELECT Clauses

```sql
-- Full SELECT syntax:
-- SELECT [DISTINCT]
--   expressions
-- FROM table(s)
-- [JOIN ...]
-- [WHERE conditions]
-- [GROUP BY columns]
-- [HAVING conditions]
-- [WINDOW definitions]
-- [ORDER BY columns]
-- [LIMIT count]
-- [OFFSET start]

-- ตัวอย่าง query ที่ใช้ทุก clause
SELECT 
    c.name AS category,
    COUNT(p.product_id) AS product_count,
    AVG(p.price) AS avg_price,
    MIN(p.price) AS min_price,
    MAX(p.price) AS max_price,
    SUM(p.stock_qty) AS total_stock
FROM shop.categories c
INNER JOIN shop.products p ON c.category_id = p.category_id
WHERE p.is_active = TRUE
GROUP BY c.category_id, c.name
HAVING COUNT(p.product_id) >= 2
ORDER BY avg_price DESC
LIMIT 5;
```

---

## 4. WHERE Conditions

### 4.1 Comparison Operators

```sql
-- พื้นฐาน: =, !=, <>, <, >, <=, >=
SELECT * FROM shop.products WHERE price = 42900.00;
SELECT * FROM shop.products WHERE price != 42900.00;
SELECT * FROM shop.products WHERE price <> 42900.00;   -- same as !=
SELECT * FROM shop.products WHERE price > 30000;
SELECT * FROM shop.products WHERE price < 30000;
SELECT * FROM shop.products WHERE price >= 30000;
SELECT * FROM shop.products WHERE price <= 30000;
```

### 4.2 Logical Operators

```sql
-- AND, OR, NOT
SELECT * FROM shop.products 
WHERE category_id = 1 AND price > 30000;

SELECT * FROM shop.products 
WHERE category_id = 1 OR category_id = 2;

SELECT * FROM shop.products 
WHERE NOT is_active;

-- ใช้ () เพื่อจัด precedence
SELECT * FROM shop.products
WHERE (category_id = 1 OR category_id = 2) AND price > 10000;
```

### 4.3 Range and List Operators

```sql
-- BETWEEN (inclusive)
SELECT * FROM shop.products WHERE price BETWEEN 10000 AND 50000;
SELECT * FROM shop.orders 
WHERE created_at BETWEEN '2024-01-01' AND '2024-12-31';

-- IN (ตรงกับ list)
SELECT * FROM shop.products WHERE category_id IN (1, 2, 3);
SELECT * FROM shop.orders WHERE status IN ('pending', 'processing');

-- NOT IN
SELECT * FROM shop.orders WHERE status NOT IN ('cancelled', 'refunded');

-- ANY / ALL (ใช้กับ subquery หรือ array)
SELECT * FROM shop.products 
WHERE price > ANY(ARRAY[10000, 20000, 30000]);  -- price > 10000

SELECT * FROM shop.products 
WHERE price > ALL(ARRAY[10000, 20000, 30000]);  -- price > 30000
```

### 4.4 Pattern Matching

```sql
-- LIKE (case-sensitive ใน PostgreSQL)
SELECT * FROM shop.customers WHERE first_name LIKE 'ส%';    -- เริ่มด้วย ส
SELECT * FROM shop.customers WHERE email LIKE '%@gmail.com'; -- ลงท้ายด้วย
SELECT * FROM shop.customers WHERE email LIKE '%test%';      -- มี test อยู่

-- ILIKE (case-insensitive)
SELECT * FROM shop.products WHERE name ILIKE '%pro%';

-- % = any sequence of characters
-- _ = any single character
SELECT * FROM shop.products WHERE name LIKE 'iPhone _ Pro';  -- iPhone 15 Pro

-- NOT LIKE
SELECT * FROM shop.customers WHERE email NOT LIKE '%test%';

-- SIMILAR TO (regex-like)
SELECT * FROM shop.customers 
WHERE phone SIMILAR TO '08[0-9][-][0-9]{3}[-][0-9]{4}';

-- Regex (~)
SELECT * FROM shop.products WHERE name ~ '^iPhone';    -- เริ่มด้วย iPhone (case-sensitive)
SELECT * FROM shop.products WHERE name ~* '^iphone';   -- case-insensitive
SELECT * FROM shop.products WHERE name !~ '^iPhone';   -- NOT match
SELECT * FROM shop.products WHERE name !~* '^iphone';  -- NOT match (case-insensitive)
```

### 4.5 NULL Checks

```sql
-- IS NULL / IS NOT NULL (ห้ามใช้ = NULL)
SELECT * FROM shop.customers WHERE phone IS NULL;
SELECT * FROM shop.customers WHERE phone IS NOT NULL;

-- Null-safe equal (ถ้าต้องเปรียบเทียบกับ NULL)
SELECT * FROM shop.customers WHERE phone IS NOT DISTINCT FROM NULL;
SELECT * FROM shop.customers WHERE phone IS DISTINCT FROM '081-234-5678';
```

### 4.6 EXISTS

```sql
-- EXISTS: ตรวจว่ามี rows ใน subquery หรือไม่
SELECT * FROM shop.customers c
WHERE EXISTS (
    SELECT 1 FROM shop.orders o
    WHERE o.customer_id = c.customer_id
      AND o.status = 'completed'
);

-- NOT EXISTS
SELECT * FROM shop.customers c
WHERE NOT EXISTS (
    SELECT 1 FROM shop.orders o WHERE o.customer_id = c.customer_id
);  -- customer ที่ยังไม่เคยซื้อ
```

---

## 5. ORDER BY, LIMIT, OFFSET

```sql
-- ORDER BY เดี่ยว
SELECT * FROM shop.products ORDER BY price;         -- ASC default
SELECT * FROM shop.products ORDER BY price DESC;    -- ลดลง
SELECT * FROM shop.products ORDER BY price ASC;     -- เพิ่มขึ้น

-- ORDER BY หลาย columns
SELECT * FROM shop.products 
ORDER BY category_id ASC, price DESC;

-- ORDER BY column alias
SELECT 
    product_id,
    name,
    price * 1.07 AS price_vat
FROM shop.products
ORDER BY price_vat DESC;

-- ORDER BY column position (ไม่แนะนำ แต่ใช้ได้)
SELECT product_id, name, price FROM shop.products ORDER BY 3 DESC;

-- NULL ordering (NULLs default อยู่ท้ายสำหรับ ASC, หน้าสำหรับ DESC)
SELECT * FROM shop.customers ORDER BY phone ASC NULLS LAST;
SELECT * FROM shop.customers ORDER BY phone DESC NULLS FIRST;

-- LIMIT และ OFFSET (Pagination)
-- หน้าที่ 1 (10 items ต่อหน้า)
SELECT * FROM shop.products ORDER BY product_id LIMIT 10 OFFSET 0;

-- หน้าที่ 2
SELECT * FROM shop.products ORDER BY product_id LIMIT 10 OFFSET 10;

-- หน้าที่ 3
SELECT * FROM shop.products ORDER BY product_id LIMIT 10 OFFSET 20;

-- Helper: คำนวณ offset จาก page number
-- OFFSET = (page_number - 1) * page_size
-- เช่น page 3: OFFSET = (3-1) * 10 = 20
```

---

## 6. Aggregate Functions

```sql
-- COUNT
SELECT COUNT(*) AS total_products FROM shop.products;
SELECT COUNT(phone) AS customers_with_phone FROM shop.customers;  -- นับเฉพาะ NOT NULL
SELECT COUNT(DISTINCT city) AS unique_cities FROM shop.customers;

-- SUM
SELECT SUM(total_amount) AS total_revenue FROM shop.orders WHERE status = 'completed';
SELECT SUM(stock_qty * price) AS inventory_value FROM shop.products;

-- AVG
SELECT AVG(price) AS avg_product_price FROM shop.products;
SELECT ROUND(AVG(price), 2) AS avg_price FROM shop.products;

-- MIN / MAX
SELECT MIN(price) AS cheapest, MAX(price) AS most_expensive FROM shop.products;
SELECT MIN(created_at) AS first_order FROM shop.orders;

-- GROUP BY
SELECT 
    status,
    COUNT(*) AS order_count,
    SUM(total_amount) AS revenue
FROM shop.orders
GROUP BY status;

-- GROUP BY หลาย columns
SELECT 
    c.name AS category,
    p.is_active,
    COUNT(*) AS count,
    AVG(p.price) AS avg_price
FROM shop.products p
JOIN shop.categories c ON p.category_id = c.category_id
GROUP BY c.name, p.is_active
ORDER BY c.name;

-- HAVING (filter หลัง GROUP BY)
SELECT 
    customer_id,
    COUNT(*) AS order_count,
    SUM(total_amount) AS total_spent
FROM shop.orders
WHERE status = 'completed'
GROUP BY customer_id
HAVING SUM(total_amount) > 50000  -- เฉพาะ customer ที่ใช้จ่ายมากกว่า 50,000
ORDER BY total_spent DESC;

-- STRING_AGG (aggregate strings)
SELECT 
    category_id,
    STRING_AGG(name, ', ' ORDER BY name) AS product_names
FROM shop.products
GROUP BY category_id;

-- ARRAY_AGG (aggregate into array)
SELECT 
    customer_id,
    ARRAY_AGG(order_id ORDER BY created_at) AS order_ids
FROM shop.orders
GROUP BY customer_id;

-- BOOL_AND / BOOL_OR
SELECT 
    category_id,
    BOOL_AND(is_active) AS all_active,
    BOOL_OR(is_active) AS any_active
FROM shop.products
GROUP BY category_id;

-- PERCENTILE
SELECT 
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY price) AS median_price,
    PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY price) AS q1,
    PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY price) AS q3
FROM shop.products;

-- FILTER clause (conditional aggregation)
SELECT 
    COUNT(*) AS total_orders,
    COUNT(*) FILTER (WHERE status = 'completed') AS completed,
    COUNT(*) FILTER (WHERE status = 'cancelled') AS cancelled,
    SUM(total_amount) FILTER (WHERE status = 'completed') AS completed_revenue
FROM shop.orders;
```

---

## 7. JOIN Types

### 7.1 INNER JOIN

```sql
-- INNER JOIN: เอาเฉพาะ rows ที่มีคู่กันทั้งสองฝั่ง
SELECT 
    o.order_id,
    c.first_name || ' ' || c.last_name AS customer_name,
    o.total_amount,
    o.status
FROM shop.orders o
INNER JOIN shop.customers c ON o.customer_id = c.customer_id;
-- หรือ JOIN (shorthand)
```

### 7.2 LEFT JOIN

```sql
-- LEFT JOIN: เอาทุก rows จากตารางซ้าย + match จากขวา (NULL ถ้าไม่มีคู่)
SELECT 
    c.customer_id,
    c.first_name,
    COUNT(o.order_id) AS total_orders,
    COALESCE(SUM(o.total_amount), 0) AS total_spent
FROM shop.customers c
LEFT JOIN shop.orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.first_name
ORDER BY total_spent DESC;
-- customer ที่ไม่เคยซื้อจะแสดงด้วย total_orders=0, total_spent=0
```

### 7.3 RIGHT JOIN

```sql
-- RIGHT JOIN: เอาทุก rows จากตารางขวา (ไม่ค่อยใช้ มักแปลงเป็น LEFT JOIN แทน)
SELECT 
    c.name AS category_name,
    p.name AS product_name,
    p.price
FROM shop.products p
RIGHT JOIN shop.categories c ON p.category_id = c.category_id
ORDER BY c.name, p.name;
-- แสดงทุก categories รวมถึงที่ไม่มีสินค้า
```

### 7.4 FULL OUTER JOIN

```sql
-- FULL OUTER JOIN: รวมทั้ง LEFT และ RIGHT JOIN
SELECT 
    c.name AS category_name,
    p.name AS product_name
FROM shop.categories c
FULL OUTER JOIN shop.products p ON c.category_id = p.category_id
ORDER BY c.name NULLS LAST, p.name NULLS LAST;
```

### 7.5 CROSS JOIN

```sql
-- CROSS JOIN: ทุก combination (Cartesian product)
-- ใช้สำหรับสร้าง combinations หรือ date ranges
SELECT 
    c.name AS color,
    s.name AS size
FROM (VALUES ('Red'), ('Blue'), ('Green')) AS c(name)
CROSS JOIN (VALUES ('S'), ('M'), ('L'), ('XL')) AS s(name)
ORDER BY c.name, s.name;
```

### 7.6 SELF JOIN

```sql
-- SELF JOIN: JOIN table กับตัวเอง (สำหรับ hierarchical data)
SELECT 
    child.name AS subcategory,
    parent.name AS parent_category
FROM shop.categories child
LEFT JOIN shop.categories parent ON child.parent_id = parent.category_id
ORDER BY parent.name, child.name;
```

### 7.7 Multiple JOINs

```sql
-- JOIN หลาย tables พร้อมกัน
SELECT 
    o.order_id,
    c.first_name || ' ' || c.last_name AS customer_name,
    p.name AS product_name,
    cat.name AS category_name,
    oi.quantity,
    oi.unit_price,
    oi.subtotal
FROM shop.orders o
JOIN shop.customers c ON o.customer_id = c.customer_id
JOIN shop.order_items oi ON o.order_id = oi.order_id
JOIN shop.products p ON oi.product_id = p.product_id
JOIN shop.categories cat ON p.category_id = cat.category_id
WHERE o.status = 'completed'
ORDER BY o.created_at DESC;
```

---

## 8. Subqueries

### 8.1 Non-Correlated Subquery

```sql
-- Subquery ใน WHERE
SELECT * FROM shop.products
WHERE price > (SELECT AVG(price) FROM shop.products);

-- Subquery ใน IN
SELECT * FROM shop.customers
WHERE customer_id IN (
    SELECT DISTINCT customer_id FROM shop.orders
    WHERE total_amount > 50000
);

-- Subquery ใน FROM (Derived Table)
SELECT 
    category_name,
    product_count,
    avg_price
FROM (
    SELECT 
        c.name AS category_name,
        COUNT(p.product_id) AS product_count,
        ROUND(AVG(p.price), 2) AS avg_price
    FROM shop.categories c
    JOIN shop.products p ON c.category_id = p.category_id
    GROUP BY c.name
) AS category_stats
WHERE product_count > 1
ORDER BY avg_price DESC;
```

### 8.2 Correlated Subquery

```sql
-- Correlated: subquery อ้างอิง outer query
-- สินค้าราคาแพงสุดในแต่ละ category
SELECT 
    p.product_id,
    p.name,
    p.price,
    p.category_id
FROM shop.products p
WHERE p.price = (
    SELECT MAX(p2.price) 
    FROM shop.products p2 
    WHERE p2.category_id = p.category_id  -- อ้างอิง outer query
);

-- Customer ที่มียอดซื้อมากกว่าค่าเฉลี่ยของ customers ทุกคน
SELECT 
    c.customer_id,
    c.first_name,
    (SELECT SUM(total_amount) FROM shop.orders o 
     WHERE o.customer_id = c.customer_id AND status = 'completed') AS total_spent
FROM shop.customers c
WHERE (
    SELECT COALESCE(SUM(total_amount), 0) 
    FROM shop.orders o 
    WHERE o.customer_id = c.customer_id AND status = 'completed'
) > (
    SELECT AVG(customer_total)
    FROM (
        SELECT customer_id, SUM(total_amount) AS customer_total
        FROM shop.orders
        WHERE status = 'completed'
        GROUP BY customer_id
    ) t
);
```

---

## 9. CTEs (Common Table Expressions)

```sql
-- Basic CTE (WITH clause)
WITH completed_orders AS (
    SELECT 
        customer_id,
        COUNT(*) AS order_count,
        SUM(total_amount) AS total_spent
    FROM shop.orders
    WHERE status = 'completed'
    GROUP BY customer_id
)
SELECT 
    c.first_name,
    c.last_name,
    co.order_count,
    co.total_spent
FROM shop.customers c
JOIN completed_orders co ON c.customer_id = co.customer_id
ORDER BY co.total_spent DESC;

-- Multiple CTEs
WITH 
product_sales AS (
    SELECT 
        p.product_id,
        p.name,
        p.price,
        SUM(oi.quantity) AS units_sold,
        SUM(oi.subtotal) AS revenue
    FROM shop.products p
    JOIN shop.order_items oi ON p.product_id = oi.product_id
    JOIN shop.orders o ON oi.order_id = o.order_id
    WHERE o.status = 'completed'
    GROUP BY p.product_id, p.name, p.price
),
avg_stats AS (
    SELECT AVG(revenue) AS avg_revenue FROM product_sales
)
SELECT 
    ps.name,
    ps.units_sold,
    ps.revenue,
    CASE 
        WHEN ps.revenue > a.avg_revenue THEN 'Above Average'
        ELSE 'Below Average'
    END AS performance
FROM product_sales ps
CROSS JOIN avg_stats a
ORDER BY ps.revenue DESC;

-- Recursive CTE (สำหรับ hierarchical data)
WITH RECURSIVE category_tree AS (
    -- Base case: top-level categories
    SELECT 
        category_id,
        name,
        parent_id,
        1 AS level,
        ARRAY[category_id] AS path,
        name AS full_path
    FROM shop.categories
    WHERE parent_id IS NULL
    
    UNION ALL
    
    -- Recursive case
    SELECT 
        c.category_id,
        c.name,
        c.parent_id,
        ct.level + 1,
        ct.path || c.category_id,
        ct.full_path || ' > ' || c.name
    FROM shop.categories c
    JOIN category_tree ct ON c.parent_id = ct.category_id
)
SELECT 
    REPEAT('  ', level - 1) || name AS category,
    level,
    full_path
FROM category_tree
ORDER BY path;

-- CTE ใน INSERT/UPDATE/DELETE
WITH new_product AS (
    INSERT INTO shop.products (category_id, name, slug, price, stock_qty)
    VALUES (6, 'OnePlus 12', 'oneplus-12', 24900.00, 25)
    RETURNING *
)
INSERT INTO shop.reviews (product_id, customer_id, rating, title)
SELECT product_id, 1, 5, 'รอรีวิวจริง'
FROM new_product;
```

---

## 10. Window Functions

```sql
-- Window Function Syntax:
-- function_name() OVER (
--     [PARTITION BY columns]
--     [ORDER BY columns]
--     [frame_clause]
-- )

-- ROW_NUMBER: เลข 1, 2, 3...
SELECT 
    order_id,
    customer_id,
    total_amount,
    ROW_NUMBER() OVER (ORDER BY total_amount DESC) AS rank_by_amount
FROM shop.orders
WHERE status = 'completed';

-- RANK: เลข rank (มี gap ถ้า tie)
-- DENSE_RANK: เลข rank (ไม่มี gap)
SELECT 
    name,
    price,
    category_id,
    RANK() OVER (PARTITION BY category_id ORDER BY price DESC) AS price_rank,
    DENSE_RANK() OVER (PARTITION BY category_id ORDER BY price DESC) AS dense_rank
FROM shop.products
WHERE is_active = TRUE;

-- PARTITION BY: แบ่งกลุ่มก่อน apply window function
SELECT 
    p.name AS product,
    c.name AS category,
    p.price,
    ROUND(AVG(p.price) OVER (PARTITION BY c.category_id), 2) AS category_avg,
    ROUND(p.price - AVG(p.price) OVER (PARTITION BY c.category_id), 2) AS diff_from_avg,
    RANK() OVER (PARTITION BY c.category_id ORDER BY p.price DESC) AS rank_in_category
FROM shop.products p
JOIN shop.categories c ON p.category_id = c.category_id;

-- LAG: ค่าของ row ก่อนหน้า
-- LEAD: ค่าของ row ถัดไป
SELECT 
    order_id,
    customer_id,
    total_amount,
    LAG(total_amount) OVER (
        PARTITION BY customer_id 
        ORDER BY created_at
    ) AS previous_order_amount,
    LEAD(total_amount) OVER (
        PARTITION BY customer_id 
        ORDER BY created_at
    ) AS next_order_amount
FROM shop.orders
ORDER BY customer_id, created_at;

-- FIRST_VALUE / LAST_VALUE / NTH_VALUE
SELECT 
    name,
    price,
    category_id,
    FIRST_VALUE(name) OVER (
        PARTITION BY category_id 
        ORDER BY price DESC
    ) AS most_expensive_in_category,
    LAST_VALUE(name) OVER (
        PARTITION BY category_id 
        ORDER BY price DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS cheapest_in_category
FROM shop.products;

-- Cumulative SUM (Running Total)
SELECT 
    created_at::DATE AS order_date,
    SUM(total_amount) AS daily_revenue,
    SUM(SUM(total_amount)) OVER (ORDER BY created_at::DATE) AS cumulative_revenue
FROM shop.orders
WHERE status = 'completed'
GROUP BY created_at::DATE
ORDER BY order_date;

-- NTILE: แบ่งออกเป็น N groups
SELECT 
    name,
    price,
    NTILE(4) OVER (ORDER BY price) AS price_quartile
FROM shop.products;

-- PERCENT_RANK / CUME_DIST
SELECT 
    name,
    price,
    ROUND(PERCENT_RANK() OVER (ORDER BY price) * 100, 2) AS percentile,
    ROUND(CUME_DIST() OVER (ORDER BY price) * 100, 2) AS cumulative_dist
FROM shop.products;
```

---

## 11. CASE WHEN

```sql
-- Simple CASE
SELECT 
    order_id,
    total_amount,
    CASE status
        WHEN 'pending' THEN 'รอดำเนินการ'
        WHEN 'processing' THEN 'กำลังดำเนินการ'
        WHEN 'completed' THEN 'เสร็จสิ้น'
        WHEN 'cancelled' THEN 'ยกเลิก'
        ELSE 'ไม่ทราบสถานะ'
    END AS status_thai
FROM shop.orders;

-- Searched CASE
SELECT 
    product_id,
    name,
    price,
    CASE 
        WHEN price < 1000 THEN 'ราคาประหยัด'
        WHEN price < 10000 THEN 'ราคากลาง'
        WHEN price < 50000 THEN 'ราคาสูง'
        ELSE 'Premium'
    END AS price_tier,
    CASE 
        WHEN stock_qty = 0 THEN 'Out of Stock'
        WHEN stock_qty < 10 THEN 'Low Stock'
        WHEN stock_qty < 50 THEN 'In Stock'
        ELSE 'Plenty in Stock'
    END AS stock_status
FROM shop.products;

-- CASE ใน aggregate function (Conditional Aggregation)
SELECT 
    category_id,
    COUNT(*) AS total,
    COUNT(*) FILTER (WHERE is_active) AS active_count,
    COUNT(*) FILTER (WHERE NOT is_active) AS inactive_count,
    SUM(CASE WHEN price > 10000 THEN 1 ELSE 0 END) AS expensive_count,
    SUM(CASE WHEN price <= 10000 THEN 1 ELSE 0 END) AS affordable_count
FROM shop.products
GROUP BY category_id;

-- CASE ใน ORDER BY
SELECT * FROM shop.orders
ORDER BY 
    CASE status
        WHEN 'pending' THEN 1
        WHEN 'processing' THEN 2
        WHEN 'completed' THEN 3
        WHEN 'cancelled' THEN 4
    END,
    created_at DESC;
```

---

## 12. String Functions

```sql
-- LENGTH / CHAR_LENGTH
SELECT 
    email,
    LENGTH(email) AS email_length,
    CHAR_LENGTH(email) AS char_length  -- same for UTF-8
FROM shop.customers;

-- UPPER / LOWER
SELECT 
    UPPER(first_name) AS upper_name,
    LOWER(email) AS lower_email
FROM shop.customers;

-- TRIM / LTRIM / RTRIM
SELECT TRIM('  hello world  ');       -- 'hello world'
SELECT LTRIM('  hello world  ');      -- 'hello world  '
SELECT RTRIM('  hello world  ');      -- '  hello world'
SELECT TRIM(BOTH 'x' FROM 'xxxhelloxxx');  -- 'hello'

-- CONCAT / || (string concatenation)
SELECT 
    first_name || ' ' || last_name AS full_name,
    CONCAT(first_name, ' ', last_name) AS full_name2,
    CONCAT_WS(', ', city, country) AS location  -- concat with separator
FROM shop.customers;

-- SUBSTRING
SELECT 
    email,
    SUBSTRING(email, 1, POSITION('@' IN email) - 1) AS username,
    SUBSTRING(email FROM '@(.+)$') AS domain  -- regex
FROM shop.customers;

-- POSITION / STRPOS
SELECT POSITION('gmail' IN 'user@gmail.com');  -- 6
SELECT STRPOS('user@gmail.com', 'gmail');       -- 6 (same)

-- REPLACE
SELECT REPLACE('Hello World', 'World', 'PostgreSQL');  -- 'Hello PostgreSQL'
SELECT REPLACE(phone, '-', '') AS phone_clean FROM shop.customers;

-- SPLIT_PART
SELECT SPLIT_PART('a,b,c,d', ',', 2);  -- 'b'
SELECT SPLIT_PART(email, '@', 2) AS email_domain FROM shop.customers;

-- LEFT / RIGHT
SELECT LEFT('Hello World', 5);   -- 'Hello'
SELECT RIGHT('Hello World', 5);  -- 'World'

-- LPAD / RPAD
SELECT LPAD('42', 10, '0');   -- '0000000042'
SELECT RPAD('Hello', 10, '.'); -- 'Hello.....'

-- REPEAT
SELECT REPEAT('*', 5);  -- '*****'

-- REVERSE
SELECT REVERSE('Hello');  -- 'olleH'

-- REGEXP_REPLACE
SELECT REGEXP_REPLACE('Phone: 081-234-5678', '[^0-9]', '', 'g');  -- '0812345678'

-- FORMAT (like printf)
SELECT FORMAT('สวัสดี %s! ยอดรวม: %s บาท', 
    first_name, 
    TO_CHAR(total_amount, 'FM999,999,999.00')
)
FROM shop.customers c
JOIN shop.orders o ON c.customer_id = o.customer_id
WHERE o.status = 'completed';

-- STRING_AGG
SELECT 
    category_id,
    STRING_AGG(name, ' | ' ORDER BY name) AS products
FROM shop.products
GROUP BY category_id;

-- INITCAP (capitalize first letter of each word)
SELECT INITCAP('hello world from thailand');  -- 'Hello World From Thailand'
```

---

## 13. Date/Time Functions

```sql
-- Current date/time
SELECT NOW();                    -- timestamp with timezone
SELECT CURRENT_TIMESTAMP;        -- same
SELECT CURRENT_DATE;             -- date only
SELECT CURRENT_TIME;             -- time only
SELECT LOCALTIMESTAMP;           -- without timezone

-- EXTRACT (ดึงส่วนประกอบ)
SELECT 
    created_at,
    EXTRACT(YEAR FROM created_at) AS year,
    EXTRACT(MONTH FROM created_at) AS month,
    EXTRACT(DAY FROM created_at) AS day,
    EXTRACT(HOUR FROM created_at) AS hour,
    EXTRACT(MINUTE FROM created_at) AS minute,
    EXTRACT(DOW FROM created_at) AS day_of_week,  -- 0=Sunday, 6=Saturday
    EXTRACT(DOY FROM created_at) AS day_of_year,
    EXTRACT(WEEK FROM created_at) AS week_number,
    EXTRACT(QUARTER FROM created_at) AS quarter,
    EXTRACT(EPOCH FROM created_at) AS unix_timestamp
FROM shop.orders;

-- DATE_TRUNC (ตัดส่วนย่อย)
SELECT DATE_TRUNC('year', NOW());     -- 2024-01-01 00:00:00+07
SELECT DATE_TRUNC('month', NOW());    -- 2024-01-01 00:00:00+07
SELECT DATE_TRUNC('week', NOW());     -- จันทร์ของสัปดาห์นี้
SELECT DATE_TRUNC('day', NOW());      -- วันนี้ 00:00:00
SELECT DATE_TRUNC('hour', NOW());     -- ชั่วโมงนี้ 00:00

-- สรุปยอดขายรายเดือน
SELECT 
    DATE_TRUNC('month', created_at) AS month,
    COUNT(*) AS order_count,
    SUM(total_amount) AS revenue
FROM shop.orders
WHERE status = 'completed'
GROUP BY DATE_TRUNC('month', created_at)
ORDER BY month;

-- AGE (คำนวณอายุ)
SELECT 
    first_name,
    birth_date,
    AGE(birth_date) AS age,
    EXTRACT(YEAR FROM AGE(birth_date)) AS age_years
FROM shop.customers
WHERE birth_date IS NOT NULL;

-- Interval Arithmetic
SELECT NOW() + INTERVAL '7 days';          -- 7 วันข้างหน้า
SELECT NOW() - INTERVAL '1 month';         -- 1 เดือนที่แล้ว
SELECT NOW() + INTERVAL '2 hours 30 minutes';

-- Orders ใน 30 วันที่ผ่านมา
SELECT * FROM shop.orders
WHERE created_at > NOW() - INTERVAL '30 days';

-- DATE_PART (similar to EXTRACT)
SELECT DATE_PART('year', created_at) FROM shop.orders;

-- TO_TIMESTAMP (string to timestamp)
SELECT TO_TIMESTAMP('2024-01-15 10:30:00', 'YYYY-MM-DD HH24:MI:SS');

-- TO_CHAR (timestamp to string)
SELECT TO_CHAR(NOW(), 'DD/MM/YYYY HH24:MI:SS');  -- '15/01/2024 10:30:00'
SELECT TO_CHAR(NOW(), 'Day, DD Month YYYY');        -- 'Monday   , 15 January  2024'
SELECT TO_CHAR(total_amount, 'FM999,999,999.00') FROM shop.orders;

-- TO_DATE (string to date)
SELECT TO_DATE('15/01/2024', 'DD/MM/YYYY');

-- MAKE_DATE / MAKE_TIMESTAMP
SELECT MAKE_DATE(2024, 1, 15);
SELECT MAKE_TIMESTAMP(2024, 1, 15, 10, 30, 0);

-- AT TIME ZONE
SELECT NOW() AT TIME ZONE 'UTC';
SELECT NOW() AT TIME ZONE 'Asia/Bangkok';
SELECT NOW() AT TIME ZONE 'America/New_York';
```

---

## 14. Type Casting

```sql
-- CAST function
SELECT CAST('123' AS INTEGER);
SELECT CAST('2024-01-15' AS DATE);
SELECT CAST(42900.00 AS TEXT);
SELECT CAST('true' AS BOOLEAN);

-- :: shorthand (PostgreSQL-specific)
SELECT '123'::INTEGER;
SELECT '2024-01-15'::DATE;
SELECT 42900.00::TEXT;
SELECT 'true'::BOOLEAN;
SELECT '1,2,3'::INTEGER[];    -- Array

-- Numeric conversions
SELECT '123.45'::DECIMAL(10,2);
SELECT '123.45'::NUMERIC(10,2);
SELECT 123::FLOAT;
SELECT 123::REAL;

-- Practical examples
SELECT 
    order_id,
    total_amount,
    total_amount::TEXT || ' บาท' AS amount_display,
    created_at::DATE AS order_date,
    EXTRACT(YEAR FROM created_at)::TEXT AS year
FROM shop.orders;

-- Safe casting with TRY (PostgreSQL ใช้ CASE แทน)
SELECT 
    CASE 
        WHEN value ~ '^[0-9]+$' THEN value::INTEGER
        ELSE NULL
    END AS safe_int
FROM (VALUES ('123'), ('abc'), ('456'), ('78x')) AS t(value);
```

---

## 15. NULL Functions

```sql
-- COALESCE: return first non-NULL value
SELECT 
    customer_id,
    first_name,
    COALESCE(phone, 'ไม่มีเบอร์โทร') AS phone_display
FROM shop.customers;

-- NULLIF: return NULL if two values are equal
SELECT NULLIF(stock_qty, 0) AS stock  -- NULL ถ้า stock = 0
FROM shop.products;

-- ใช้ทั้งคู่
SELECT 
    product_id,
    name,
    COALESCE(NULLIF(stock_qty, 0)::TEXT, 'หมดสต็อก') AS stock_display
FROM shop.products;

-- GREATEST / LEAST
SELECT GREATEST(10, 20, 30, 5, 15);  -- 30
SELECT LEAST(10, 20, 30, 5, 15);     -- 5

SELECT GREATEST(price, cost_price) FROM shop.products;
SELECT LEAST(price, cost_price * 2) FROM shop.products;

-- IS NULL / IS NOT NULL ใน SELECT
SELECT 
    customer_id,
    first_name,
    phone IS NULL AS no_phone,
    phone IS NOT NULL AS has_phone
FROM shop.customers;

-- NULL ใน arithmetic → result เป็น NULL
SELECT NULL + 5;           -- NULL
SELECT COALESCE(NULL, 0) + 5;  -- 5

-- NULL ใน string concat → result เป็น NULL
SELECT 'Hello' || NULL;           -- NULL
SELECT 'Hello' || COALESCE(NULL, '');  -- 'Hello'
SELECT CONCAT('Hello', NULL, ' World');  -- 'Hello World' (CONCAT ignores NULLs)
```

---

## 16. Workshop: E-Commerce Database

### Workshop 3.1: Beginner Queries

```sql
-- Q1: แสดงสินค้าทั้งหมดเรียงตามราคาจากแพงสุด
SELECT product_id, name, price
FROM shop.products
ORDER BY price DESC;

-- Q2: หา customer ที่อยู่ในกรุงเทพ
SELECT * FROM shop.customers WHERE city = 'กรุงเทพ';

-- Q3: นับจำนวน orders แต่ละ status
SELECT status, COUNT(*) AS count
FROM shop.orders
GROUP BY status
ORDER BY count DESC;

-- Q4: ยอดรวมของ orders ที่เสร็จสิ้น
SELECT SUM(total_amount) AS total_revenue
FROM shop.orders
WHERE status = 'completed';

-- Q5: สินค้าที่ stock เหลือน้อยกว่า 50 ชิ้น
SELECT name, stock_qty
FROM shop.products
WHERE stock_qty < 50
ORDER BY stock_qty;
```

### Workshop 3.2: Intermediate Queries

```sql
-- Q6: แสดงชื่อ customer พร้อมจำนวน orders และยอดรวม
SELECT 
    c.first_name || ' ' || c.last_name AS customer_name,
    COUNT(o.order_id) AS total_orders,
    COALESCE(SUM(CASE WHEN o.status = 'completed' THEN o.total_amount END), 0) AS total_spent
FROM shop.customers c
LEFT JOIN shop.orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.first_name, c.last_name
ORDER BY total_spent DESC;

-- Q7: สินค้าที่ขายได้มากสุด Top 5
SELECT 
    p.name,
    SUM(oi.quantity) AS total_sold,
    SUM(oi.subtotal) AS total_revenue
FROM shop.products p
JOIN shop.order_items oi ON p.product_id = oi.product_id
JOIN shop.orders o ON oi.order_id = o.order_id
WHERE o.status = 'completed'
GROUP BY p.product_id, p.name
ORDER BY total_sold DESC
LIMIT 5;

-- Q8: Category ที่มียอดขายสูงสุด
SELECT 
    c.name AS category,
    COUNT(DISTINCT o.order_id) AS order_count,
    SUM(oi.quantity) AS units_sold,
    SUM(oi.subtotal) AS revenue
FROM shop.categories c
JOIN shop.products p ON c.category_id = p.category_id
JOIN shop.order_items oi ON p.product_id = oi.product_id
JOIN shop.orders o ON oi.order_id = o.order_id
WHERE o.status = 'completed'
GROUP BY c.category_id, c.name
ORDER BY revenue DESC;

-- Q9: Customer ที่ไม่เคยซื้อสินค้าเลย
SELECT c.customer_id, c.first_name, c.last_name, c.email
FROM shop.customers c
LEFT JOIN shop.orders o ON c.customer_id = o.customer_id
WHERE o.order_id IS NULL;

-- Q10: สินค้าที่มีคะแนนรีวิวสูงสุด พร้อม category
SELECT 
    p.name,
    c.name AS category,
    ROUND(AVG(r.rating), 2) AS avg_rating,
    COUNT(r.review_id) AS review_count
FROM shop.products p
JOIN shop.categories c ON p.category_id = c.category_id
LEFT JOIN shop.reviews r ON p.product_id = r.product_id
GROUP BY p.product_id, p.name, c.name
HAVING COUNT(r.review_id) > 0
ORDER BY avg_rating DESC, review_count DESC;
```

### Workshop 3.3: Advanced Queries

```sql
-- Q11: กำไรต่อ category (revenue - cost)
SELECT 
    cat.name AS category,
    SUM(oi.subtotal) AS revenue,
    SUM(oi.quantity * p.cost_price) AS total_cost,
    SUM(oi.subtotal) - SUM(oi.quantity * p.cost_price) AS gross_profit,
    ROUND(
        (SUM(oi.subtotal) - SUM(oi.quantity * p.cost_price)) / 
        NULLIF(SUM(oi.subtotal), 0) * 100, 2
    ) AS profit_margin_pct
FROM shop.categories cat
JOIN shop.products p ON cat.category_id = p.category_id
JOIN shop.order_items oi ON p.product_id = oi.product_id
JOIN shop.orders o ON oi.order_id = o.order_id
WHERE o.status = 'completed' AND p.cost_price IS NOT NULL
GROUP BY cat.category_id, cat.name
ORDER BY gross_profit DESC;

-- Q12: Cohort Analysis — retention ง่ายๆ
-- customer แต่ละคนซื้อครั้งแรกเดือนไหน และซื้อซ้ำเดือนไหนบ้าง
WITH first_orders AS (
    SELECT 
        customer_id,
        DATE_TRUNC('month', MIN(created_at)) AS first_order_month
    FROM shop.orders
    WHERE status = 'completed'
    GROUP BY customer_id
),
subsequent_orders AS (
    SELECT 
        o.customer_id,
        DATE_TRUNC('month', o.created_at) AS order_month
    FROM shop.orders o
    WHERE o.status = 'completed'
)
SELECT 
    fo.first_order_month,
    so.order_month,
    COUNT(DISTINCT so.customer_id) AS customers
FROM first_orders fo
JOIN subsequent_orders so ON fo.customer_id = so.customer_id
GROUP BY fo.first_order_month, so.order_month
ORDER BY fo.first_order_month, so.order_month;

-- Q13: Moving Average ของยอดขาย 3 วัน
WITH daily_sales AS (
    SELECT 
        created_at::DATE AS sale_date,
        SUM(total_amount) AS daily_revenue
    FROM shop.orders
    WHERE status = 'completed'
    GROUP BY created_at::DATE
)
SELECT 
    sale_date,
    daily_revenue,
    ROUND(AVG(daily_revenue) OVER (
        ORDER BY sale_date
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ), 2) AS moving_avg_3day
FROM daily_sales
ORDER BY sale_date;

-- Q14: Customer Ranking ภายใน City
SELECT 
    c.city,
    c.first_name || ' ' || c.last_name AS customer_name,
    SUM(o.total_amount) AS total_spent,
    RANK() OVER (PARTITION BY c.city ORDER BY SUM(o.total_amount) DESC) AS city_rank
FROM shop.customers c
JOIN shop.orders o ON c.customer_id = o.customer_id
WHERE o.status = 'completed'
GROUP BY c.city, c.customer_id, c.first_name, c.last_name
ORDER BY c.city, city_rank;

-- Q15: สินค้าที่ถูกสั่งซื้อพร้อมกันบ่อยที่สุด (Market Basket Analysis พื้นฐาน)
SELECT 
    p1.name AS product_1,
    p2.name AS product_2,
    COUNT(*) AS co_occurrence
FROM shop.order_items oi1
JOIN shop.order_items oi2 ON oi1.order_id = oi2.order_id 
    AND oi1.product_id < oi2.product_id  -- avoid duplicates
JOIN shop.products p1 ON oi1.product_id = p1.product_id
JOIN shop.products p2 ON oi2.product_id = p2.product_id
GROUP BY p1.product_id, p1.name, p2.product_id, p2.name
HAVING COUNT(*) >= 1
ORDER BY co_occurrence DESC;
```

---

## สรุปบทที่ 3

| หัวข้อ | Commands ที่สำคัญ |
|--------|-----------------|
| DDL | CREATE TABLE, ALTER TABLE, DROP TABLE |
| DML | INSERT, UPDATE, DELETE, UPSERT |
| SELECT | FROM, WHERE, GROUP BY, HAVING, ORDER BY, LIMIT |
| WHERE | =, !=, BETWEEN, IN, LIKE, IS NULL, EXISTS |
| Aggregates | COUNT, SUM, AVG, MIN, MAX, STRING_AGG, FILTER |
| JOINs | INNER, LEFT, RIGHT, FULL, CROSS, SELF |
| Subqueries | Correlated, Non-correlated, Derived Tables |
| CTEs | WITH, Recursive CTE |
| Window Functions | ROW_NUMBER, RANK, LAG, LEAD, PARTITION BY |
| CASE WHEN | Simple, Searched, Conditional Aggregation |
| String Functions | CONCAT, SUBSTRING, REPLACE, REGEXP |
| Date Functions | EXTRACT, DATE_TRUNC, AGE, TO_CHAR |
| NULL Functions | COALESCE, NULLIF, IS NULL |

**บทต่อไป:** Data Types และ Constraints — ออกแบบ schema ที่ถูกต้อง!

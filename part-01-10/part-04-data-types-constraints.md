# บทที่ 4: Data Types และ Constraints ใน PostgreSQL

> **หลักสูตร:** PostgreSQL + Redis + S3/MinIO Database Cluster  
> **ระดับ:** เริ่มต้น → ขั้นกลาง  
> **เวลาเรียน:** 3-4 ชั่วโมง

---

## สารบัญ

1. [Numeric Types](#1-numeric-types)
2. [Character Types](#2-character-types)
3. [Boolean Type](#3-boolean-type)
4. [Date/Time Types](#4-datetime-types)
5. [UUID Type](#5-uuid-type)
6. [JSON และ JSONB](#6-json-และ-jsonb)
7. [Array Types](#7-array-types)
8. [Enum Types](#8-enum-types)
9. [Network Types](#9-network-types)
10. [Range Types](#10-range-types)
11. [Binary Types](#11-binary-types)
12. [Constraints](#12-constraints)
13. [Primary Key Strategies](#13-primary-key-strategies)
14. [Foreign Key Actions](#14-foreign-key-actions)
15. [Composite Keys](#15-composite-keys)
16. [Partial Indexes](#16-partial-indexes)
17. [Expression Indexes](#17-expression-indexes)
18. [Workshop: Blog Platform Schema](#18-workshop-blog-platform-schema)

---

## 1. Numeric Types

### 1.1 Integer Types

```sql
-- ตาราง Integer types ใน PostgreSQL
-- SMALLINT:  2 bytes, range -32,768 ถึง 32,767
-- INTEGER:   4 bytes, range -2,147,483,648 ถึง 2,147,483,647
-- BIGINT:    8 bytes, range -9,223,372,036,854,775,808 ถึง 9,223,372,036,854,775,807

-- Serial types (auto-increment)
-- SMALLSERIAL: 2 bytes, 1 ถึง 32,767
-- SERIAL:      4 bytes, 1 ถึง 2,147,483,647
-- BIGSERIAL:   8 bytes, 1 ถึง 9,223,372,036,854,775,807

-- ตัวอย่างการใช้
CREATE TABLE type_demo_integers (
    small_col   SMALLINT,       -- rating, year
    int_col     INTEGER,        -- user_id, product_id
    big_col     BIGINT,         -- transaction_id, message_id
    
    auto_small  SMALLSERIAL,    -- ไม่ค่อยใช้
    auto_int    SERIAL,         -- id column ทั่วไป
    auto_big    BIGSERIAL       -- id column ที่ต้องการมาก
);

-- เมื่อไหร่ใช้อะไร:
-- SMALLINT: rating (1-5), อายุ, percentage, ปี
-- INTEGER: user_id, product_id, order_id (ระบบขนาดกลาง)
-- BIGINT: transaction_id, message_id, ระบบขนาดใหญ่ที่มี rows > 2 พันล้าน

-- ทดสอบ overflow
SELECT (2147483647::INTEGER + 1);  -- ERROR: integer out of range
SELECT (2147483647::BIGINT + 1);   -- OK: 2147483648

-- Integer operations
SELECT 
    10 / 3,           -- 3 (integer division)
    10.0 / 3,         -- 3.3333... (float division)
    10 % 3,           -- 1 (modulo)
    10::FLOAT / 3,    -- 3.3333...
    ABS(-10),         -- 10
    SIGN(-5),         -- -1
    SIGN(0),          -- 0
    SIGN(5);          -- 1
```

### 1.2 Decimal Types

```sql
-- NUMERIC/DECIMAL: exact numeric (ใช้สำหรับเงิน!)
-- NUMERIC(precision, scale)
-- precision: จำนวน digits ทั้งหมด
-- scale: จำนวน digits หลังจุดทศนิยม

-- REAL:             4 bytes, 6 significant digits (ไม่ exact)
-- DOUBLE PRECISION: 8 bytes, 15 significant digits (ไม่ exact)
-- FLOAT:            alias สำหรับ DOUBLE PRECISION

CREATE TABLE type_demo_decimals (
    -- เงิน: ใช้ NUMERIC/DECIMAL เสมอ
    price           NUMERIC(10, 2),      -- 99,999,999.99
    amount          DECIMAL(15, 2),      -- 9,999,999,999,999.99
    exchange_rate   NUMERIC(10, 6),      -- 1.234567
    
    -- ข้อมูล scientific: ใช้ FLOAT ได้
    latitude        FLOAT,
    longitude       DOUBLE PRECISION,
    measurement     REAL
);

-- ทำไม NUMERIC สำคัญสำหรับเงิน?
SELECT 0.1::FLOAT + 0.2::FLOAT;         -- 0.30000000000000004 (ผิด!)
SELECT 0.1::NUMERIC + 0.2::NUMERIC;     -- 0.3 (ถูก!)

-- Numeric operations
SELECT 
    ROUND(3.14159, 2),      -- 3.14
    CEILING(3.2),           -- 4
    FLOOR(3.8),             -- 3
    TRUNC(3.9),             -- 3 (truncate, ไม่ round)
    TRUNC(3.14159, 2),      -- 3.14
    MOD(10, 3),             -- 1
    POWER(2, 10),           -- 1024
    SQRT(16),               -- 4
    EXP(1),                 -- 2.71828... (e)
    LN(10),                 -- 2.302... (natural log)
    LOG(10),                -- 1 (log base 10)
    LOG(2, 8),              -- 3 (log base 2 of 8)
    PI();                   -- 3.14159...

-- ตัวอย่าง: คำนวณ VAT และ margin
SELECT
    100.00::NUMERIC AS base_price,
    100.00 * 1.07 AS price_with_vat,
    ROUND(100.00 * 0.3, 2) AS margin_amount,
    '30%' AS margin_percent;
```

---

## 2. Character Types

```sql
-- VARCHAR(n): variable-length, max n characters
-- CHAR(n):    fixed-length, padded with spaces
-- TEXT:       unlimited variable-length

CREATE TABLE type_demo_strings (
    char_col    CHAR(10),           -- fixed: 'hello     '
    varchar_col VARCHAR(255),       -- variable: 'hello'
    text_col    TEXT,               -- unlimited
    
    -- ตัวอย่างการใช้
    country_code CHAR(2),           -- 'TH', 'US', 'JP' (always 2 chars)
    phone        VARCHAR(20),       -- '+66-81-234-5678'
    email        VARCHAR(255),      -- ไม่ควรเกิน 254 chars (RFC 5321)
    name         VARCHAR(100),      -- ชื่อคนทั่วไป
    bio          TEXT,              -- biography ไม่จำกัด
    body         TEXT               -- blog post content
);

-- Performance note:
-- VARCHAR(n) vs TEXT: ใน PostgreSQL ไม่มีความต่างด้าน performance
-- CHAR(n): เก็บ fixed-width อาจมี padding overhead
-- แนะนำ: ใช้ TEXT เมื่อไม่ต้องการ limit, VARCHAR(n) เมื่อต้องการ limit

-- String size limits
SELECT 
    'hello'::VARCHAR(255),
    'hello'::CHAR(10),      -- 'hello     ' (padded)
    octet_length('สวัสดี'),  -- bytes (ไม่ใช่ characters!)
    char_length('สวัสดี');   -- 6 characters

-- ระวัง: CHAR ใน comparison
SELECT 'hello'::CHAR(10) = 'hello';     -- TRUE (PostgreSQL trim spaces)
SELECT 'hello   '::CHAR(10) = 'hello';  -- TRUE
SELECT length('hello'::CHAR(10));       -- 10 (ยังมี padding)
SELECT trim('hello'::CHAR(10));         -- 'hello' (ไม่มี spaces)

-- Collation (สำหรับภาษาไทย)
-- ค่าเริ่มต้นของ PostgreSQL ใช้ collation ตาม OS
-- สำหรับไทย ควรใช้ database ที่ตั้งค่า encoding UTF8
\l  -- ดู encoding ของ databases

-- สร้าง database ด้วย Thai collation
CREATE DATABASE thai_db
    WITH ENCODING 'UTF8'
    LC_COLLATE = 'th_TH.UTF-8'
    LC_CTYPE = 'th_TH.UTF-8'
    TEMPLATE = template0;
```

---

## 3. Boolean Type

```sql
-- BOOLEAN: TRUE, FALSE, NULL
-- Accepts: true/false, 'true'/'false', 't'/'f', 'yes'/'no', 'y'/'n', '1'/'0', on/off

CREATE TABLE type_demo_boolean (
    is_active    BOOLEAN DEFAULT TRUE,
    is_deleted   BOOLEAN DEFAULT FALSE,
    is_verified  BOOLEAN,  -- NULL = unknown
    has_discount BOOLEAN NOT NULL DEFAULT FALSE
);

-- Valid boolean literals
SELECT TRUE, FALSE, 't', 'f', 'yes', 'no', 'on', 'off', '1', '0';

-- Boolean operations
SELECT 
    TRUE AND TRUE,      -- TRUE
    TRUE AND FALSE,     -- FALSE
    TRUE OR FALSE,      -- TRUE
    NOT TRUE,           -- FALSE
    NULL AND TRUE,      -- NULL
    NULL OR TRUE,       -- TRUE (! NULL OR TRUE = TRUE)
    NULL OR FALSE,      -- NULL

-- Boolean in WHERE
SELECT * FROM shop.products WHERE is_active;           -- = TRUE
SELECT * FROM shop.products WHERE NOT is_active;       -- = FALSE
SELECT * FROM shop.products WHERE is_active IS TRUE;   -- explicit check
SELECT * FROM shop.products WHERE is_active IS NOT FALSE;  -- TRUE or NULL

-- ตัวอย่าง pattern: soft delete
ALTER TABLE shop.products ADD COLUMN is_deleted BOOLEAN DEFAULT FALSE;

-- Query เฉพาะที่ไม่ถูกลบ
SELECT * FROM shop.products WHERE NOT is_deleted;
-- หรือ
SELECT * FROM shop.products WHERE is_deleted = FALSE;

-- Aggregate booleans
SELECT 
    BOOL_AND(is_active) AS all_active,    -- TRUE ถ้าทุก row เป็น TRUE
    BOOL_OR(is_active) AS any_active,     -- TRUE ถ้ามีอย่างน้อย 1 row เป็น TRUE
    COUNT(*) FILTER (WHERE is_active) AS active_count
FROM shop.products;
```

---

## 4. Date/Time Types

```sql
-- DATE:          วันที่เท่านั้น (4 bytes)
-- TIME:          เวลาเท่านั้น ไม่มี timezone (8 bytes)
-- TIMETZ:        เวลาพร้อม timezone
-- TIMESTAMP:     วันที่และเวลา ไม่มี timezone (8 bytes)
-- TIMESTAMPTZ:   วันที่และเวลา พร้อม timezone (8 bytes) ← แนะนำ!
-- INTERVAL:      ช่วงเวลา (16 bytes)

CREATE TABLE type_demo_datetime (
    date_col        DATE,           -- '2024-01-15'
    time_col        TIME,           -- '10:30:00'
    timetz_col      TIMETZ,         -- '10:30:00+07'
    ts_col          TIMESTAMP,      -- '2024-01-15 10:30:00'
    tstz_col        TIMESTAMPTZ,    -- '2024-01-15 10:30:00+07' ← ใช้นี้!
    interval_col    INTERVAL        -- '2 hours 30 minutes'
);

-- ทำไมต้องใช้ TIMESTAMPTZ แทน TIMESTAMP?
-- TIMESTAMP: เก็บตัวเลขตรงๆ ไม่มี timezone info
-- TIMESTAMPTZ: แปลงเป็น UTC ก่อนเก็บ แล้ว convert กลับเมื่อ query
-- → ถ้า server timezone เปลี่ยน, TIMESTAMP จะ "เปลี่ยน" ความหมาย
-- → TIMESTAMPTZ ยังคงถูกต้องเสมอ

-- ตัวอย่าง
SET timezone = 'Asia/Bangkok';
INSERT INTO type_demo_datetime (ts_col, tstz_col) 
VALUES (NOW(), NOW());

SET timezone = 'UTC';
SELECT ts_col, tstz_col FROM type_demo_datetime;
-- ts_col ไม่เปลี่ยน (เก็บ raw value)
-- tstz_col จะ convert เป็น UTC

-- Date literals
SELECT 
    '2024-01-15'::DATE,
    '2024-01-15'::TIMESTAMP,
    '2024-01-15 10:30:00'::TIMESTAMP,
    '2024-01-15 10:30:00+07:00'::TIMESTAMPTZ,
    '2 hours 30 minutes'::INTERVAL,
    '1 year 2 months 3 days'::INTERVAL;

-- Interval operations
SELECT 
    NOW() + INTERVAL '1 day',
    NOW() - INTERVAL '7 days',
    NOW() + '1 month'::INTERVAL,
    NOW() + '2 years 6 months'::INTERVAL,
    DATE '2024-12-31' - DATE '2024-01-01',  -- 365 (days)
    AGE(DATE '2024-12-31', DATE '2024-01-01');  -- 11 months 30 days

-- Useful date patterns
-- วันแรกของเดือนนี้
SELECT DATE_TRUNC('month', NOW())::DATE;

-- วันสุดท้ายของเดือนนี้
SELECT (DATE_TRUNC('month', NOW()) + INTERVAL '1 month - 1 day')::DATE;

-- วันแรกของปีนี้
SELECT DATE_TRUNC('year', NOW())::DATE;

-- วันแรกของสัปดาห์นี้ (จันทร์)
SELECT DATE_TRUNC('week', NOW())::DATE;

-- เปรียบเทียบ date เฉพาะวัน (ไม่รวมเวลา)
SELECT * FROM shop.orders WHERE created_at::DATE = CURRENT_DATE;

-- วันเกิดในเดือนนี้
SELECT * FROM shop.customers 
WHERE EXTRACT(MONTH FROM birth_date) = EXTRACT(MONTH FROM NOW());

-- Orders ภายใน 7 วันที่ผ่านมา
SELECT * FROM shop.orders 
WHERE created_at >= NOW() - INTERVAL '7 days';

-- Generating date series (มีประโยชน์มาก!)
SELECT generate_series(
    '2024-01-01'::DATE,
    '2024-01-31'::DATE,
    '1 day'::INTERVAL
)::DATE AS calendar_date;

-- ใช้ Calendar สำหรับหา missing dates
WITH date_range AS (
    SELECT generate_series(
        '2024-01-01'::DATE,
        '2024-01-31'::DATE,
        '1 day'::INTERVAL
    )::DATE AS date
),
daily_orders AS (
    SELECT created_at::DATE AS date, SUM(total_amount) AS revenue
    FROM shop.orders
    WHERE status = 'completed'
    GROUP BY created_at::DATE
)
SELECT 
    dr.date,
    COALESCE(do.revenue, 0) AS revenue
FROM date_range dr
LEFT JOIN daily_orders do ON dr.date = do.date
ORDER BY dr.date;
```

---

## 5. UUID Type

```sql
-- UUID: Universally Unique Identifier
-- Format: 8-4-4-4-12 hexadecimal characters
-- Example: 550e8400-e29b-41d4-a716-446655440000
-- Size: 16 bytes (128 bits)

-- Enable extension สำหรับ gen_random_uuid()
CREATE EXTENSION IF NOT EXISTS "pgcrypto";
-- หรือ PostgreSQL 13+ มี gen_random_uuid() built-in

CREATE TABLE type_demo_uuid (
    id          UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    user_uuid   UUID,
    external_id UUID
);

-- สร้าง UUID
SELECT gen_random_uuid();     -- random UUID v4
SELECT md5(random()::TEXT || clock_timestamp()::TEXT)::UUID;  -- another way
SELECT uuid_generate_v1();    -- timestamp-based UUID v1 (ต้องติดตั้ง uuid-ossp)
SELECT uuid_generate_v4();    -- random UUID v4

-- ติดตั้ง uuid-ossp extension
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
SELECT uuid_generate_v4();
SELECT uuid_generate_v1mc();  -- v1 with random MAC (ดีกว่า v1)

-- ตัวอย่าง: ใช้ UUID เป็น Primary Key
CREATE TABLE users_uuid (
    user_id     UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    email       VARCHAR(255) UNIQUE NOT NULL,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

INSERT INTO users_uuid (email) VALUES ('user@example.com')
RETURNING user_id, email;

-- UUID vs SERIAL: เปรียบเทียบ
-- UUID ข้อดี:
-- ✅ สร้างได้บน client โดยไม่ต้อง query DB
-- ✅ ไม่ leak จำนวน records ให้ผู้ใช้เห็น
-- ✅ ง่ายสำหรับ distributed systems (merge หลาย DBs ได้)
-- ✅ URL ไม่สามารถเดาได้ง่าย

-- UUID ข้อเสีย:
-- ❌ ขนาดใหญ่กว่า INTEGER (16 bytes vs 4 bytes)
-- ❌ Index ใหญ่กว่า → disk/memory มากกว่า
-- ❌ Random UUID ทำให้ index fragmented (เขียนช้ากว่า)
-- ❌ อ่านและพิมพ์ยาก

-- แก้ปัญหา index fragmentation ด้วย ULID หรือ UUIDv7
-- ULID: เรียงตามเวลา แต่ยังคงเป็น unique identifier
```

---

## 6. JSON และ JSONB

```sql
-- JSON:  เก็บ text ตรงๆ (รักษา whitespace, key order)
-- JSONB: เก็บแบบ binary (เร็วกว่าสำหรับ query, ไม่รักษา whitespace/key order)
-- ส่วนใหญ่ใช้ JSONB (เร็วกว่า, index ได้ดีกว่า)

CREATE TABLE type_demo_json (
    id          SERIAL PRIMARY KEY,
    data_json   JSON,
    data_jsonb  JSONB
);

-- Insert JSON data
INSERT INTO type_demo_json (data_json, data_jsonb) VALUES (
    '{"name": "iPhone 15", "specs": {"ram": 8, "storage": 256}, "tags": ["phone", "apple"]}',
    '{"name": "iPhone 15", "specs": {"ram": 8, "storage": 256}, "tags": ["phone", "apple"]}'
);

-- JSON Operators
-- -> : get JSON object field (returns JSON)
-- ->> : get JSON object field as text
-- #> : get nested JSON
-- #>> : get nested JSON as text

SELECT 
    data_jsonb -> 'name',           -- "iPhone 15" (JSON)
    data_jsonb ->> 'name',          -- iPhone 15 (text)
    data_jsonb -> 'specs' -> 'ram', -- 8 (JSON)
    data_jsonb #> '{specs,ram}',    -- 8 (JSON, nested)
    data_jsonb #>> '{specs,ram}',   -- 8 (text, nested)
    data_jsonb -> 'tags' -> 0,      -- "phone" (array element)
    data_jsonb -> 'tags' ->> 0      -- phone (text)
FROM type_demo_json;

-- JSONB Operators
-- @> : contains (left contains right)
-- <@ : contained by (left is contained in right)
-- ? : key exists
-- ?| : any key exists
-- ?& : all keys exist
-- || : concatenate
-- - : remove key

SELECT 
    data_jsonb @> '{"name": "iPhone 15"}',      -- TRUE
    data_jsonb ? 'name',                          -- TRUE
    data_jsonb ?| ARRAY['name', 'missing'],       -- TRUE (any exists)
    data_jsonb ?& ARRAY['name', 'specs']          -- TRUE (all exist)
FROM type_demo_json;

-- ตัวอย่างจริง: Product attributes แบบ flexible schema
CREATE TABLE products_flexible (
    product_id  SERIAL PRIMARY KEY,
    name        VARCHAR(255) NOT NULL,
    price       NUMERIC(10,2) NOT NULL,
    attributes  JSONB DEFAULT '{}'::JSONB
);

INSERT INTO products_flexible (name, price, attributes) VALUES
    ('iPhone 15 Pro', 42900, '{"color": "titanium", "storage": "256GB", "camera": "48MP", "5g": true}'),
    ('Nike Air Max', 4500, '{"color": "white", "size": "42", "material": "mesh", "waterproof": false}'),
    ('Clean Code Book', 890, '{"author": "Robert Martin", "pages": 431, "language": "English", "edition": "1st"}');

-- Query ด้วย JSONB
-- สินค้าที่มีสี white
SELECT * FROM products_flexible
WHERE attributes @> '{"color": "white"}';

-- สินค้าที่มี storage attribute
SELECT * FROM products_flexible
WHERE attributes ? 'storage';

-- ดึง attribute เฉพาะ
SELECT 
    name,
    attributes ->> 'color' AS color,
    attributes ->> 'storage' AS storage
FROM products_flexible;

-- Index สำหรับ JSONB
CREATE INDEX idx_products_attributes ON products_flexible USING GIN(attributes);

-- หลัง index การ query @> จะเร็วขึ้นมาก
EXPLAIN ANALYZE
SELECT * FROM products_flexible WHERE attributes @> '{"color": "white"}';

-- Update JSONB
UPDATE products_flexible
SET attributes = attributes || '{"updated": true}'::JSONB
WHERE product_id = 1;

-- ลบ key
UPDATE products_flexible
SET attributes = attributes - 'updated'
WHERE product_id = 1;

-- JSONB aggregation
SELECT JSONB_AGG(
    JSONB_BUILD_OBJECT('id', product_id, 'name', name, 'price', price)
) AS products
FROM products_flexible;

-- jsonb_each: expand JSONB to rows
SELECT 
    product_id,
    key,
    value
FROM products_flexible, JSONB_EACH(attributes)
WHERE product_id = 1;

-- JSON_TABLE (PostgreSQL 17+) สำหรับ flatten JSONB arrays
-- jsonb_array_elements: expand array to rows
SELECT 
    product_id,
    elem
FROM products_flexible, 
     JSONB_ARRAY_ELEMENTS(attributes -> 'tags') AS elem
WHERE attributes ? 'tags';
```

---

## 7. Array Types

```sql
-- PostgreSQL รองรับ Arrays สำหรับทุก type
-- ARRAY syntax: type[] หรือ ARRAY[val1, val2, ...]

CREATE TABLE type_demo_arrays (
    id          SERIAL PRIMARY KEY,
    tags        TEXT[],
    scores      INTEGER[],
    prices      NUMERIC[],
    matrix      INTEGER[][],    -- 2D array
    schedule    TIMESTAMPTZ[]
);

-- Insert arrays
INSERT INTO type_demo_arrays (tags, scores) VALUES
    (ARRAY['postgresql', 'database', 'sql'], ARRAY[95, 87, 92]),
    ('{"redis", "cache", "nosql"}', '{80, 75, 88}');  -- another syntax

-- Array operations
SELECT 
    tags,
    tags[1],                    -- first element (1-indexed!)
    tags[1:2],                  -- slice [1,2]
    ARRAY_LENGTH(tags, 1),      -- length of dimension 1
    ARRAY_UPPER(tags, 1),       -- upper bound = length
    ARRAY_LOWER(tags, 1),       -- lower bound = 1 (default)
    CARDINALITY(tags),          -- total elements (all dimensions)
    ARRAY_APPEND(tags, 'new'),  -- append element
    ARRAY_PREPEND('first', tags), -- prepend element
    ARRAY_REMOVE(tags, 'sql'),  -- remove element
    ARRAY_CAT(tags, ARRAY['extra1', 'extra2'])  -- concat arrays
FROM type_demo_arrays;

-- Array containment
SELECT 
    tags @> ARRAY['postgresql'],        -- contains 'postgresql'
    tags <@ ARRAY['postgresql', 'sql', 'database', 'redis'],  -- is subset of
    tags && ARRAY['postgresql', 'mongo'] -- overlaps (any common element)
FROM type_demo_arrays;

-- ANY / ALL ใช้กับ arrays
SELECT * FROM type_demo_arrays WHERE 'postgresql' = ANY(tags);
SELECT * FROM type_demo_arrays WHERE 'sql' = ANY(tags);

-- UNNEST: แปลง array เป็น rows
SELECT 
    id,
    UNNEST(tags) AS tag
FROM type_demo_arrays;

-- Aggregate ด้วย ARRAY_AGG
SELECT 
    category_id,
    ARRAY_AGG(name ORDER BY name) AS products
FROM shop.products
GROUP BY category_id;

-- Index สำหรับ Array (GIN index)
CREATE INDEX idx_tags ON type_demo_arrays USING GIN(tags);

-- ตัวอย่างจริง: Tags บน Blog Posts
CREATE TABLE blog_posts (
    post_id     SERIAL PRIMARY KEY,
    title       VARCHAR(255),
    tags        TEXT[] NOT NULL DEFAULT '{}',
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_blog_tags ON blog_posts USING GIN(tags);

INSERT INTO blog_posts (title, tags) VALUES
    ('PostgreSQL Tips', ARRAY['postgresql', 'database', 'tips']),
    ('Redis Caching', ARRAY['redis', 'cache', 'performance']),
    ('Full Stack Dev', ARRAY['postgresql', 'redis', 'nodejs']);

-- หา posts ที่มี tag 'postgresql'
SELECT * FROM blog_posts WHERE tags @> ARRAY['postgresql'];

-- หา posts ที่มี tag 'postgresql' หรือ 'redis'
SELECT * FROM blog_posts WHERE tags && ARRAY['postgresql', 'redis'];

-- นับ posts ต่อ tag
SELECT 
    tag,
    COUNT(*) AS post_count
FROM blog_posts, UNNEST(tags) AS tag
GROUP BY tag
ORDER BY post_count DESC;
```

---

## 8. Enum Types

```sql
-- Custom ENUM type: ค่าที่กำหนดไว้ล่วงหน้า
-- ข้อดี: ประหยัด storage, type-safe
-- ข้อเสีย: แก้ไขยาก (ต้อง migrate)

-- สร้าง ENUM type
CREATE TYPE order_status AS ENUM ('pending', 'processing', 'shipped', 'completed', 'cancelled', 'refunded');
CREATE TYPE gender_type AS ENUM ('M', 'F', 'Other', 'Prefer not to say');
CREATE TYPE payment_method AS ENUM ('credit_card', 'debit_card', 'promptpay', 'bank_transfer', 'cash_on_delivery', 'wallet');
CREATE TYPE user_role AS ENUM ('customer', 'vendor', 'admin', 'superadmin');

-- ใช้ ENUM ใน table
CREATE TABLE orders_v2 (
    order_id    SERIAL PRIMARY KEY,
    status      order_status DEFAULT 'pending',
    payment     payment_method,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- Insert ด้วย ENUM
INSERT INTO orders_v2 (status, payment) 
VALUES ('processing', 'credit_card');

-- ERROR: invalid input value for enum order_status: "unknown"
-- INSERT INTO orders_v2 (status) VALUES ('unknown');

-- Query ด้วย ENUM
SELECT * FROM orders_v2 WHERE status = 'processing';
SELECT * FROM orders_v2 WHERE status > 'pending';  -- enum มี ordering!

-- ดู enum values
SELECT enumlabel FROM pg_enum 
JOIN pg_type ON pg_enum.enumtypid = pg_type.oid
WHERE typname = 'order_status';

-- เพิ่ม value ใหม่ (ทำได้)
ALTER TYPE order_status ADD VALUE 'on_hold' AFTER 'processing';

-- เปลี่ยนชื่อ value (PostgreSQL 10+)
ALTER TYPE order_status RENAME VALUE 'refunded' TO 'refund_processed';

-- ลบ value (ทำไม่ได้โดยตรง! ต้อง recreate)
-- ถ้าต้องการลบ enum value:
-- 1. สร้าง type ใหม่
-- 2. Convert column ไปใช้ type ใหม่
-- 3. ลบ type เก่า

-- เปรียบเทียบ ENUM vs VARCHAR vs Check Constraint
-- ENUM: type-safe, ประหยัด storage, query เร็ว, แต่แก้ไขยาก
-- VARCHAR + CHECK: ยืดหยุ่นกว่า, เพิ่ม/ลบค่าง่าย
-- Reference table: เพิ่ม/ลบง่ายที่สุด, ใช้ JOIN

-- วิธีที่ดีกว่า ENUM สำหรับ values ที่เปลี่ยนบ่อย:
CREATE TABLE order_statuses (
    status_code VARCHAR(20) PRIMARY KEY,
    status_name VARCHAR(100),
    sort_order  INTEGER,
    is_active   BOOLEAN DEFAULT TRUE
);

INSERT INTO order_statuses VALUES
    ('pending', 'รอดำเนินการ', 1, TRUE),
    ('processing', 'กำลังดำเนินการ', 2, TRUE),
    ('completed', 'เสร็จสิ้น', 3, TRUE),
    ('cancelled', 'ยกเลิก', 4, TRUE);
```

---

## 9. Network Types

```sql
-- INET:    IP address (IPv4 or IPv6) with optional subnet
-- CIDR:    IP network (subnet address)
-- MACADDR: MAC address (6 bytes)
-- MACADDR8: MAC address EUI-64 (8 bytes)

CREATE TABLE type_demo_network (
    id          SERIAL PRIMARY KEY,
    client_ip   INET,
    server_ip   INET,
    network     CIDR,
    mac_addr    MACADDR
);

INSERT INTO type_demo_network VALUES 
    (DEFAULT, '192.168.1.100', '10.0.0.1', '192.168.0.0/24', '08:00:2b:01:02:03'),
    (DEFAULT, '::1', '2001:db8::1', '2001:db8::/32', 'a0:b1:c2:d3:e4:f5');

-- INET operations
SELECT 
    '192.168.1.100'::INET,
    '192.168.1.100/24'::INET,
    HOST('192.168.1.100/24'::INET),     -- '192.168.1.100'
    MASKLEN('192.168.1.100/24'::INET),  -- 24
    NETMASK('192.168.1.100/24'::INET),  -- 255.255.255.0
    NETWORK('192.168.1.100/24'::INET),  -- 192.168.1.0/24
    BROADCAST('192.168.1.100/24'::INET), -- 192.168.1.255
    FAMILY('192.168.1.100'::INET);       -- 4 (IPv4)

-- containment operators
SELECT 
    '192.168.1.5'::INET << '192.168.1.0/24'::INET,  -- is contained by subnet
    '192.168.1.0/24'::CIDR >> '192.168.1.5'::INET;  -- contains

-- ตัวอย่างจริง: Audit log with IP
CREATE TABLE audit_log (
    log_id      BIGSERIAL PRIMARY KEY,
    user_id     INTEGER,
    action      VARCHAR(50),
    client_ip   INET NOT NULL,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- Index สำหรับ INET (ใช้ GIST)
CREATE INDEX idx_audit_ip ON audit_log USING GIST(client_ip inet_ops);

-- Query ด้วย subnet
SELECT * FROM audit_log WHERE client_ip << '10.0.0.0/8';  -- internal network only
```

---

## 10. Range Types

```sql
-- int4range:   Integer range
-- int8range:   Bigint range
-- numrange:    Numeric range
-- tsrange:     Timestamp range (without timezone)
-- tstzrange:   Timestamp with timezone range
-- daterange:   Date range

-- Range notation:
-- [a,b] = inclusive both ends
-- (a,b) = exclusive both ends
-- [a,b) = inclusive a, exclusive b (most common for dates)
-- (a,b] = exclusive a, inclusive b
-- empty = empty range

CREATE TABLE type_demo_ranges (
    id          SERIAL PRIMARY KEY,
    price_range numrange,
    valid_dates daterange,
    booking_ts  tstzrange
);

INSERT INTO type_demo_ranges VALUES 
    (DEFAULT, '[100, 1000]', '[2024-01-01, 2024-12-31)', '[2024-01-15 09:00, 2024-01-15 18:00)');

-- Range operations
SELECT 
    '[100, 1000]'::numrange,
    LOWER('[100, 1000]'::numrange),     -- 100
    UPPER('[100, 1000]'::numrange),     -- 1000
    LOWER_INC('[100, 1000]'::numrange), -- TRUE (inclusive)
    UPPER_INC('[100, 1000]'::numrange), -- TRUE
    
    -- Containment
    '[100, 1000]'::numrange @> 500::NUMERIC,    -- TRUE
    '[100, 1000]'::numrange @> 1500::NUMERIC,   -- FALSE
    '[100, 500]'::numrange <@ '[50, 600]'::numrange, -- TRUE
    
    -- Overlap
    '[100, 500]'::numrange && '[400, 800]'::numrange, -- TRUE
    '[100, 300]'::numrange && '[400, 800]'::numrange, -- FALSE
    
    -- Union / Intersection
    '[100, 500]'::numrange + '[400, 800]'::numrange, -- [100, 800]
    '[100, 500]'::numrange * '[400, 800]'::numrange, -- [400, 500]
    
    ISEMPTY('[100, 100)'::numrange);  -- TRUE (empty range)

-- ตัวอย่างจริง: Hotel booking ป้องกัน overlap
CREATE TABLE hotel_bookings (
    booking_id  SERIAL PRIMARY KEY,
    room_id     INTEGER,
    guest_name  VARCHAR(100),
    duration    daterange NOT NULL,
    
    -- Constraint: ห้ามซ้อน booking ในห้องเดียวกัน
    EXCLUDE USING GIST (room_id WITH =, duration WITH &&)
);

-- GIST index สำหรับ EXCLUDE constraint
-- PostgreSQL สร้าง index อัตโนมัติเมื่อสร้าง EXCLUDE constraint

INSERT INTO hotel_bookings VALUES
    (DEFAULT, 101, 'Alice', '[2024-01-15, 2024-01-20)'),
    (DEFAULT, 101, 'Bob',   '[2024-01-20, 2024-01-25)');  -- OK, ไม่ overlap

-- ERROR: conflicting key value violates exclusion constraint
-- INSERT INTO hotel_bookings VALUES
--     (DEFAULT, 101, 'Charlie', '[2024-01-18, 2024-01-22)');  -- overlap!

-- Price tiers ด้วย ranges
CREATE TABLE price_tiers (
    tier_name   VARCHAR(50),
    qty_range   int4range,
    discount    NUMERIC(5,2)
);

INSERT INTO price_tiers VALUES
    ('retail', '[1, 10)', 0),
    ('wholesale', '[10, 100)', 10),
    ('distributor', '[100, 1000)', 20),
    ('bulk', '[1000,)', 30);

-- หา discount ตาม quantity
SELECT tier_name, discount
FROM price_tiers
WHERE qty_range @> 50;  -- quantity = 50 → wholesale, 10%
```

---

## 11. Binary Types

```sql
-- BYTEA: binary data (byte array)
-- ใช้สำหรับ: encrypted data, binary files เล็กๆ
-- สำหรับไฟล์ใหญ่ → ใช้ S3/MinIO แทน!

CREATE TABLE type_demo_binary (
    id          SERIAL PRIMARY KEY,
    file_data   BYTEA,
    checksum    BYTEA
);

-- Insert binary data
INSERT INTO type_demo_binary (file_data) 
VALUES ('\x48656c6c6f20576f726c64');  -- 'Hello World' in hex

-- Binary operations
SELECT 
    '\x48656c6c6f'::BYTEA,
    LENGTH('\x48656c6c6f'::BYTEA),       -- 5 bytes
    ENCODE('\x48656c6c6f'::BYTEA, 'hex'),  -- '48656c6c6f'
    ENCODE('\x48656c6c6f'::BYTEA, 'base64'), -- 'SGVsbG8='
    DECODE('SGVsbG8=', 'base64'),          -- '\x48656c6c6f'
    CONVERT_FROM('\x48656c6c6f'::BYTEA, 'UTF8');  -- 'Hello'

-- ใช้สำหรับ encrypted data
CREATE EXTENSION IF NOT EXISTS pgcrypto;

CREATE TABLE user_secrets (
    user_id     INTEGER,
    secret_data BYTEA  -- encrypted
);

-- Encrypt
INSERT INTO user_secrets VALUES (
    1,
    pgp_sym_encrypt('my secret data', 'encryption_key')
);

-- Decrypt
SELECT pgp_sym_decrypt(secret_data, 'encryption_key') AS decrypted
FROM user_secrets WHERE user_id = 1;

-- Hash (MD5, SHA)
SELECT MD5('Hello World');
SELECT ENCODE(SHA256('Hello World'::BYTEA), 'hex');
SELECT ENCODE(SHA512('Hello World'::BYTEA), 'hex');
```

---

## 12. Constraints

### 12.1 NOT NULL

```sql
-- NOT NULL: ห้าม NULL value
CREATE TABLE constraints_demo (
    id          SERIAL PRIMARY KEY,
    email       VARCHAR(255) NOT NULL,   -- inline constraint
    name        VARCHAR(100)
);

-- เพิ่ม NOT NULL หลังจากสร้าง table
ALTER TABLE constraints_demo 
ALTER COLUMN name SET NOT NULL;

-- ลบ NOT NULL
ALTER TABLE constraints_demo 
ALTER COLUMN name DROP NOT NULL;
```

### 12.2 UNIQUE

```sql
-- UNIQUE: ไม่ให้มีค่าซ้ำ (NULL ไม่นับ)
CREATE TABLE constraints_unique (
    id          SERIAL PRIMARY KEY,
    email       VARCHAR(255) UNIQUE,     -- inline
    username    VARCHAR(50),
    phone       VARCHAR(20),
    
    -- Table-level unique constraint
    CONSTRAINT uq_username_phone UNIQUE (username, phone)  -- composite unique
);

-- เพิ่ม UNIQUE constraint
ALTER TABLE constraints_unique ADD CONSTRAINT uq_email UNIQUE (email);
ALTER TABLE constraints_unique ADD UNIQUE (phone);  -- shorthand

-- PostgreSQL: หลาย NULL ใน UNIQUE column ได้! (NULL != NULL)
INSERT INTO constraints_unique (email) VALUES (NULL);  -- OK
INSERT INTO constraints_unique (email) VALUES (NULL);  -- OK (two NULLs allowed)
INSERT INTO constraints_unique (email) VALUES ('test@test.com');  -- OK
INSERT INTO constraints_unique (email) VALUES ('test@test.com');  -- ERROR!

-- NULLS NOT DISTINCT (PostgreSQL 15+): ถือ NULL = NULL
CREATE TABLE constraints_null_unique (
    id    SERIAL PRIMARY KEY,
    phone VARCHAR(20) UNIQUE NULLS NOT DISTINCT  -- NULL ซ้ำไม่ได้
);
```

### 12.3 CHECK

```sql
-- CHECK: กำหนด condition ที่ต้องเป็นจริง
CREATE TABLE constraints_check (
    product_id  SERIAL PRIMARY KEY,
    name        VARCHAR(255) NOT NULL,
    price       NUMERIC(10,2) CHECK (price > 0),           -- inline
    cost_price  NUMERIC(10,2),
    stock_qty   INTEGER DEFAULT 0 CHECK (stock_qty >= 0),
    rating      SMALLINT,
    email       VARCHAR(255),
    
    -- Table-level check (สามารถอ้างอิงหลาย columns)
    CONSTRAINT chk_price_gt_cost CHECK (price >= cost_price),
    CONSTRAINT chk_rating_range CHECK (rating BETWEEN 1 AND 5),
    CONSTRAINT chk_email_format CHECK (email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$')
);

-- เพิ่ม CHECK constraint
ALTER TABLE constraints_check 
ADD CONSTRAINT chk_name_length CHECK (LENGTH(name) >= 3);

-- Check constraint ที่ซับซ้อน
ALTER TABLE shop.orders
ADD CONSTRAINT chk_valid_status 
CHECK (status IN ('pending', 'processing', 'completed', 'cancelled', 'refunded'));

ALTER TABLE shop.orders
ADD CONSTRAINT chk_amount_positive
CHECK (total_amount > 0 AND shipping_fee >= 0 AND discount >= 0);
```

### 12.4 DEFAULT

```sql
-- DEFAULT: ค่าเริ่มต้นถ้าไม่ได้ระบุ
CREATE TABLE constraints_default (
    id          SERIAL PRIMARY KEY,
    status      VARCHAR(20) DEFAULT 'active',
    is_active   BOOLEAN DEFAULT TRUE,
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    updated_at  TIMESTAMPTZ DEFAULT NOW(),
    rating      SMALLINT DEFAULT 0,
    settings    JSONB DEFAULT '{}'::JSONB,
    tags        TEXT[] DEFAULT ARRAY[]::TEXT[]
);

-- Default ด้วย function
CREATE TABLE events (
    event_id    UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    event_type  VARCHAR(50),
    occurred_at TIMESTAMPTZ DEFAULT NOW()
);

-- DEFAULT expressions
CREATE TABLE products_default (
    product_id  SERIAL PRIMARY KEY,
    sku         VARCHAR(50) DEFAULT 'SKU-' || LPAD(NEXTVAL('products_default_product_id_seq')::TEXT, 6, '0'),
    name        VARCHAR(255),
    
    -- Computed default (PostgreSQL 12+: GENERATED ALWAYS AS)
    slug        TEXT GENERATED ALWAYS AS (LOWER(REPLACE(name, ' ', '-'))) STORED
);
```

### 12.5 GENERATED ALWAYS AS (Computed Columns)

```sql
-- STORED: คำนวณและเก็บ (disk space เพิ่ม แต่ query เร็วกว่า)
-- VIRTUAL: คำนวณตอน query (PostgreSQL ยังไม่รองรับ VIRTUAL)

CREATE TABLE order_items_v2 (
    item_id     SERIAL PRIMARY KEY,
    order_id    INTEGER NOT NULL,
    product_id  INTEGER NOT NULL,
    quantity    INTEGER NOT NULL CHECK (quantity > 0),
    unit_price  NUMERIC(10,2) NOT NULL,
    
    -- Computed column
    subtotal    NUMERIC(12,2) GENERATED ALWAYS AS (quantity * unit_price) STORED,
    
    -- ข้อจำกัด: ไม่สามารถ reference ตาราง อื่น, ไม่ใช้ subquery
    -- ต้องเป็น IMMUTABLE expression
    discount_pct NUMERIC(5,2) DEFAULT 0,
    net_amount  NUMERIC(12,2) GENERATED ALWAYS AS (
        quantity * unit_price * (1 - discount_pct / 100)
    ) STORED
);

INSERT INTO order_items_v2 (order_id, product_id, quantity, unit_price, discount_pct)
VALUES (1, 1, 2, 42900, 5);

SELECT item_id, quantity, unit_price, subtotal, discount_pct, net_amount
FROM order_items_v2;
```

---

## 13. Primary Key Strategies

### 13.1 SERIAL (Auto-increment Integer)

```sql
-- เหมาะสำหรับ: simple, fast, small-medium systems
-- ข้อดี: เล็ก, เรียง, อ่านง่าย
-- ข้อเสีย: sequential, ข้อมูล leak จำนวน records

CREATE TABLE pk_serial (
    id    SERIAL PRIMARY KEY,  -- INT auto-increment
    name  VARCHAR(100)
);
-- SERIAL = SEQUENCE + DEFAULT nextval()

-- ดู sequence ที่สร้างให้อัตโนมัติ
\ds pk_serial_id_seq

-- Manual sequence control
SELECT currval('pk_serial_id_seq');
SELECT nextval('pk_serial_id_seq');
SELECT setval('pk_serial_id_seq', 1000);  -- reset to 1000
```

### 13.2 UUID (Random)

```sql
-- เหมาะสำหรับ: distributed systems, public APIs
CREATE TABLE pk_uuid (
    id    UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    name  VARCHAR(100)
);
```

### 13.3 ULID (Universally Unique Lexicographically Sortable Identifier)

```sql
-- ULID = timestamp + randomness = sortable UUID
-- Format: 26 characters, Crockford Base32
-- Monotonically increasing (ถ้าสร้างใน ms เดียวกัน ยังเรียงได้)

-- ติดตั้ง extension (ต้องการ pg_ulid extension)
-- CREATE EXTENSION IF NOT EXISTS pg_ulid;

-- หรือสร้าง ULID ด้วย bytea
CREATE OR REPLACE FUNCTION generate_ulid() RETURNS TEXT AS $$
DECLARE
    encoding BYTEA = '0123456789ABCDEFGHJKMNPQRSTVWXYZ';
    output TEXT = '';
    unix_time BIGINT;
    ulid BYTEA;
BEGIN
    unix_time = (EXTRACT(EPOCH FROM NOW()) * 1000)::BIGINT;
    ulid = DECODE(LPAD(TO_HEX(unix_time), 12, '0'), 'hex');
    ulid = ulid || gen_random_bytes(10);
    
    FOR i IN 0..25 LOOP
        output = output || CHR(GET_BYTE(encoding, GET_BYTE(ulid, i / 5 + 4) & 31));
    END LOOP;
    
    RETURN output;
END;
$$ LANGUAGE plpgsql VOLATILE;

-- ใช้ ULID
CREATE TABLE pk_ulid (
    id    TEXT DEFAULT generate_ulid() PRIMARY KEY,
    name  VARCHAR(100)
);
```

### 13.4 Snowflake ID (Twitter-style)

```sql
-- Snowflake: 64-bit integer ที่ประกอบด้วย:
-- 41 bits: milliseconds timestamp
-- 10 bits: machine/datacenter ID
-- 12 bits: sequence number
-- ข้อดี: เรียง, เล็ก (BIGINT), สร้างได้หลาย instances พร้อมกัน

-- ตัวอย่าง implementation ง่ายๆ:
CREATE SEQUENCE snowflake_seq;

CREATE OR REPLACE FUNCTION generate_snowflake_id() RETURNS BIGINT AS $$
DECLARE
    epoch BIGINT = 1577836800000;  -- 2020-01-01 00:00:00 UTC in ms
    current_ms BIGINT;
    seq_id BIGINT;
    machine_id BIGINT = 1;  -- 0-1023
BEGIN
    current_ms = EXTRACT(EPOCH FROM CLOCK_TIMESTAMP()) * 1000 - epoch;
    seq_id = nextval('snowflake_seq') & 4095;  -- 12 bits
    RETURN (current_ms << 22) | (machine_id << 12) | seq_id;
END;
$$ LANGUAGE plpgsql;

CREATE TABLE pk_snowflake (
    id    BIGINT DEFAULT generate_snowflake_id() PRIMARY KEY,
    name  VARCHAR(100)
);

-- Decode Snowflake ID
SELECT
    id,
    to_timestamp(((id >> 22) + 1577836800000) / 1000.0) AS created_at,
    (id >> 12) & 1023 AS machine_id,
    id & 4095 AS sequence
FROM pk_snowflake;
```

### 13.5 สรุปการเลือก Primary Key

```
┌──────────────┬───────────┬────────┬──────────────┬─────────────────┐
│ Strategy     │ Size      │ Sort   │ Distributed  │ Use When        │
├──────────────┼───────────┼────────┼──────────────┼─────────────────┤
│ SERIAL       │ 4 bytes   │ ✅ Yes │ ❌ Single DB  │ Simple apps     │
│ BIGSERIAL    │ 8 bytes   │ ✅ Yes │ ❌ Single DB  │ Large single DB │
│ UUID v4      │ 16 bytes  │ ❌ No  │ ✅ Yes        │ Public APIs     │
│ UUID v7/ULID │ 16 bytes  │ ✅ Yes │ ✅ Yes        │ Best of both    │
│ Snowflake    │ 8 bytes   │ ✅ Yes │ ✅ Yes        │ High throughput │
└──────────────┴───────────┴────────┴──────────────┴─────────────────┘
```

---

## 14. Foreign Key Actions

```sql
-- ON DELETE / ON UPDATE actions:
-- RESTRICT  : ห้ามลบ parent ถ้ายังมี children (default)
-- NO ACTION : เหมือน RESTRICT แต่ check ตอนท้าย transaction
-- CASCADE   : ลบ/อัปเดต children อัตโนมัติ
-- SET NULL  : ตั้ง FK เป็น NULL
-- SET DEFAULT : ตั้ง FK เป็น default value

-- ตัวอย่าง: Orders และ Order Items
CREATE TABLE fk_demo_orders (
    order_id    SERIAL PRIMARY KEY,
    total       NUMERIC(10,2)
);

CREATE TABLE fk_demo_items (
    item_id     SERIAL PRIMARY KEY,
    order_id    INTEGER NOT NULL,
    product     VARCHAR(100),
    
    CONSTRAINT fk_order_id 
    FOREIGN KEY (order_id) 
    REFERENCES fk_demo_orders(order_id) 
    ON DELETE CASCADE    -- ลบ order → ลบ items ทั้งหมดด้วย
    ON UPDATE CASCADE    -- อัปเดต order_id → อัปเดต items ด้วย
);

-- ON DELETE SET NULL: ตัวอย่าง
CREATE TABLE fk_demo_posts (
    post_id     SERIAL PRIMARY KEY,
    author_id   INTEGER,
    content     TEXT,
    
    CONSTRAINT fk_author
    FOREIGN KEY (author_id)
    REFERENCES fk_demo_orders(order_id)
    ON DELETE SET NULL   -- ถ้าลบ author, post ยังอยู่แต่ author_id = NULL
);

-- ดู FK constraints
SELECT
    tc.constraint_name,
    tc.table_name,
    kcu.column_name,
    ccu.table_name AS foreign_table_name,
    ccu.column_name AS foreign_column_name,
    rc.delete_rule,
    rc.update_rule
FROM information_schema.table_constraints AS tc
JOIN information_schema.key_column_usage AS kcu
    ON tc.constraint_name = kcu.constraint_name
JOIN information_schema.constraint_column_usage AS ccu
    ON ccu.constraint_name = tc.constraint_name
JOIN information_schema.referential_constraints AS rc
    ON tc.constraint_name = rc.constraint_name
WHERE tc.constraint_type = 'FOREIGN KEY';

-- Deferrable constraints (ตรวจสอบตอนท้าย transaction)
ALTER TABLE fk_demo_items
ADD CONSTRAINT fk_order_id_deferred
FOREIGN KEY (order_id)
REFERENCES fk_demo_orders(order_id)
DEFERRABLE INITIALLY DEFERRED;  -- ตรวจสอบตอน COMMIT

-- ใช้ประโยชน์ deferrable:
BEGIN;
INSERT INTO fk_demo_items (order_id, product) VALUES (999, 'product');  -- FK ยังไม่ error
INSERT INTO fk_demo_orders (order_id, total) VALUES (999, 100);          -- สร้าง parent
COMMIT;  -- ตรวจสอบตอนนี้ → OK!
```

---

## 15. Composite Keys

```sql
-- Composite Primary Key: primary key ที่มีหลาย columns
CREATE TABLE composite_key_demo (
    user_id     INTEGER,
    product_id  INTEGER,
    added_at    TIMESTAMPTZ DEFAULT NOW(),
    quantity    INTEGER DEFAULT 1,
    
    PRIMARY KEY (user_id, product_id)  -- composite PK
);

-- สามารถ Insert ซ้ำกันได้เฉพาะคู่ที่ต่างกัน
INSERT INTO composite_key_demo (user_id, product_id) VALUES (1, 1);  -- OK
INSERT INTO composite_key_demo (user_id, product_id) VALUES (1, 2);  -- OK
INSERT INTO composite_key_demo (user_id, product_id) VALUES (2, 1);  -- OK
-- INSERT INTO composite_key_demo VALUES (1, 1, NOW(), 1);  -- ERROR: duplicate

-- UPSERT ด้วย composite key
INSERT INTO composite_key_demo (user_id, product_id, quantity)
VALUES (1, 1, 3)
ON CONFLICT (user_id, product_id) DO UPDATE 
SET quantity = EXCLUDED.quantity, added_at = NOW();

-- ตัวอย่างจริง: Many-to-many relationships
CREATE TABLE user_roles (
    user_id     INTEGER REFERENCES users_uuid(user_id),
    role_id     INTEGER,
    assigned_at TIMESTAMPTZ DEFAULT NOW(),
    assigned_by INTEGER,
    
    PRIMARY KEY (user_id, role_id)
);

CREATE TABLE product_tags (
    product_id  INTEGER REFERENCES shop.products(product_id),
    tag_name    VARCHAR(50),
    
    PRIMARY KEY (product_id, tag_name)
);
```

---

## 16. Partial Indexes

```sql
-- Partial Index: Index เฉพาะบาง rows ที่ตรงตาม WHERE condition
-- ข้อดี: ขนาดเล็กกว่า full index, เร็วกว่า

-- ตัวอย่าง: Index เฉพาะ orders ที่ยังไม่เสร็จ
CREATE INDEX idx_orders_pending ON shop.orders(created_at)
WHERE status IN ('pending', 'processing');

-- ตัวอย่าง: Index เฉพาะ active products
CREATE INDEX idx_active_products ON shop.products(name, price)
WHERE is_active = TRUE AND deleted_at IS NULL;

-- ตัวอย่าง: Unique index เฉพาะ non-deleted records (soft delete pattern)
CREATE UNIQUE INDEX idx_unique_active_email ON shop.customers(email)
WHERE deleted_at IS NULL;  -- อนุญาต email ซ้ำกันได้ถ้า deleted

-- ตรวจสอบว่า query ใช้ partial index
EXPLAIN SELECT * FROM shop.orders WHERE status = 'pending';
EXPLAIN SELECT * FROM shop.products WHERE is_active = TRUE AND name LIKE 'iPhone%';
```

---

## 17. Expression Indexes

```sql
-- Expression Index: Index บน expression หรือ function
-- ใช้เมื่อ query มี WHERE clause ที่ใช้ function กับ column

-- ตัวอย่าง: ค้นหาด้วย LOWER() (case-insensitive search)
CREATE INDEX idx_customers_email_lower ON shop.customers(LOWER(email));

-- Query จะใช้ index นี้เมื่อ:
SELECT * FROM shop.customers WHERE LOWER(email) = 'alice@example.com';
-- จะไม่ใช้ index:
-- SELECT * FROM shop.customers WHERE email = 'alice@example.com';

-- ตัวอย่าง: Index วัน (ตัดเวลาออก)
CREATE INDEX idx_orders_date ON shop.orders(DATE(created_at));

SELECT * FROM shop.orders WHERE DATE(created_at) = '2024-01-15';

-- ตัวอย่าง: Index บน JSONB field
CREATE INDEX idx_products_color ON products_flexible((attributes->>'color'));

SELECT * FROM products_flexible WHERE attributes->>'color' = 'white';

-- Index บน computed column
CREATE INDEX idx_orders_year_month ON shop.orders(
    EXTRACT(YEAR FROM created_at)::INTEGER,
    EXTRACT(MONTH FROM created_at)::INTEGER
);

SELECT 
    EXTRACT(YEAR FROM created_at) AS year,
    EXTRACT(MONTH FROM created_at) AS month,
    COUNT(*) AS orders
FROM shop.orders
GROUP BY 1, 2
ORDER BY 1, 2;
```

---

## 18. Workshop: Blog Platform Schema

```sql
-- สร้าง Blog Platform ที่ครบสมบูรณ์ด้วย constraints ถูกต้อง

-- Schema
CREATE SCHEMA IF NOT EXISTS blog;

-- Custom Types
CREATE TYPE blog.post_status AS ENUM ('draft', 'published', 'scheduled', 'archived');
CREATE TYPE blog.user_role AS ENUM ('reader', 'author', 'editor', 'admin');
CREATE TYPE blog.reaction_type AS ENUM ('like', 'love', 'haha', 'wow', 'sad', 'angry');

-- Users
CREATE TABLE blog.users (
    user_id         UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    username        VARCHAR(50) UNIQUE NOT NULL CHECK (username ~ '^[a-z0-9_-]{3,50}$'),
    email           VARCHAR(255) UNIQUE NOT NULL CHECK (email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$'),
    password_hash   TEXT NOT NULL,
    display_name    VARCHAR(100) NOT NULL,
    bio             TEXT,
    avatar_url      VARCHAR(2048),
    website_url     VARCHAR(2048) CHECK (website_url ~* '^https?://'),
    role            blog.user_role DEFAULT 'reader',
    is_verified     BOOLEAN DEFAULT FALSE,
    is_active       BOOLEAN DEFAULT TRUE,
    settings        JSONB DEFAULT '{}'::JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_login      TIMESTAMPTZ,
    deleted_at      TIMESTAMPTZ
);

-- Indexes สำหรับ users
CREATE UNIQUE INDEX idx_users_email_active ON blog.users(LOWER(email)) WHERE deleted_at IS NULL;
CREATE UNIQUE INDEX idx_users_username_active ON blog.users(username) WHERE deleted_at IS NULL;
CREATE INDEX idx_users_role ON blog.users(role);
CREATE INDEX idx_users_created ON blog.users(created_at);

-- Categories
CREATE TABLE blog.categories (
    category_id     SERIAL PRIMARY KEY,
    name            VARCHAR(100) NOT NULL,
    slug            VARCHAR(100) UNIQUE NOT NULL CHECK (slug ~ '^[a-z0-9-]+$'),
    description     TEXT,
    parent_id       INTEGER REFERENCES blog.categories(category_id) ON DELETE SET NULL,
    sort_order      INTEGER DEFAULT 0,
    is_active       BOOLEAN DEFAULT TRUE,
    seo_title       VARCHAR(70),
    seo_description VARCHAR(160),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_categories_parent ON blog.categories(parent_id);
CREATE INDEX idx_categories_slug ON blog.categories(slug);

-- Tags
CREATE TABLE blog.tags (
    tag_id      SERIAL PRIMARY KEY,
    name        VARCHAR(50) NOT NULL,
    slug        VARCHAR(50) UNIQUE NOT NULL CHECK (slug ~ '^[a-z0-9-]+$'),
    post_count  INTEGER DEFAULT 0 CHECK (post_count >= 0),
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- Posts
CREATE TABLE blog.posts (
    post_id         UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    author_id       UUID NOT NULL REFERENCES blog.users(user_id) ON DELETE RESTRICT,
    category_id     INTEGER REFERENCES blog.categories(category_id) ON DELETE SET NULL,
    title           VARCHAR(255) NOT NULL CHECK (LENGTH(TRIM(title)) >= 5),
    slug            VARCHAR(255) UNIQUE NOT NULL CHECK (slug ~ '^[a-z0-9-]+$'),
    excerpt         VARCHAR(500),
    content         TEXT NOT NULL CHECK (LENGTH(TRIM(content)) >= 10),
    cover_image_url VARCHAR(2048),
    status          blog.post_status DEFAULT 'draft',
    reading_time    SMALLINT GENERATED ALWAYS AS (
        GREATEST(1, (LENGTH(content) / 200))
    ) STORED,
    word_count      INTEGER GENERATED ALWAYS AS (
        ARRAY_LENGTH(REGEXP_SPLIT_TO_ARRAY(TRIM(content), '\s+'), 1)
    ) STORED,
    view_count      BIGINT DEFAULT 0 CHECK (view_count >= 0),
    like_count      INTEGER DEFAULT 0 CHECK (like_count >= 0),
    comment_count   INTEGER DEFAULT 0 CHECK (comment_count >= 0),
    is_featured     BOOLEAN DEFAULT FALSE,
    is_pinned       BOOLEAN DEFAULT FALSE,
    allow_comments  BOOLEAN DEFAULT TRUE,
    meta_title      VARCHAR(70),
    meta_description VARCHAR(160),
    tags            TEXT[] DEFAULT ARRAY[]::TEXT[],
    published_at    TIMESTAMPTZ,
    scheduled_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at      TIMESTAMPTZ,
    
    CONSTRAINT chk_published_at CHECK (
        (status = 'published' AND published_at IS NOT NULL) OR
        (status != 'published')
    ),
    CONSTRAINT chk_scheduled_at CHECK (
        (status = 'scheduled' AND scheduled_at IS NOT NULL AND scheduled_at > NOW()) OR
        (status != 'scheduled')
    )
);

-- Indexes สำหรับ posts
CREATE UNIQUE INDEX idx_posts_slug_active ON blog.posts(slug) WHERE deleted_at IS NULL;
CREATE INDEX idx_posts_author ON blog.posts(author_id);
CREATE INDEX idx_posts_category ON blog.posts(category_id);
CREATE INDEX idx_posts_status ON blog.posts(status);
CREATE INDEX idx_posts_published ON blog.posts(published_at DESC) WHERE status = 'published';
CREATE INDEX idx_posts_featured ON blog.posts(is_featured, published_at DESC) WHERE status = 'published' AND is_featured = TRUE;
CREATE INDEX idx_posts_tags ON blog.posts USING GIN(tags);

-- Full-text search index
CREATE INDEX idx_posts_fts ON blog.posts USING GIN(
    to_tsvector('english', COALESCE(title, '') || ' ' || COALESCE(content, ''))
);

-- Post-Tag relationship
CREATE TABLE blog.post_tags (
    post_id     UUID NOT NULL REFERENCES blog.posts(post_id) ON DELETE CASCADE,
    tag_id      INTEGER NOT NULL REFERENCES blog.tags(tag_id) ON DELETE CASCADE,
    added_at    TIMESTAMPTZ DEFAULT NOW(),
    
    PRIMARY KEY (post_id, tag_id)
);

-- Comments
CREATE TABLE blog.comments (
    comment_id  UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    post_id     UUID NOT NULL REFERENCES blog.posts(post_id) ON DELETE CASCADE,
    author_id   UUID REFERENCES blog.users(user_id) ON DELETE SET NULL,
    parent_id   UUID REFERENCES blog.comments(comment_id) ON DELETE CASCADE,
    content     TEXT NOT NULL CHECK (LENGTH(TRIM(content)) >= 1 AND LENGTH(content) <= 5000),
    is_approved BOOLEAN DEFAULT FALSE,
    is_spam     BOOLEAN DEFAULT FALSE,
    like_count  INTEGER DEFAULT 0 CHECK (like_count >= 0),
    depth       SMALLINT DEFAULT 0 CHECK (depth >= 0 AND depth <= 5),  -- max 5 levels deep
    ip_address  INET,
    user_agent  TEXT,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at  TIMESTAMPTZ
);

CREATE INDEX idx_comments_post ON blog.comments(post_id, created_at);
CREATE INDEX idx_comments_parent ON blog.comments(parent_id);
CREATE INDEX idx_comments_author ON blog.comments(author_id);
CREATE INDEX idx_comments_pending ON blog.comments(created_at) WHERE is_approved = FALSE AND is_spam = FALSE;

-- Reactions
CREATE TABLE blog.reactions (
    user_id     UUID NOT NULL REFERENCES blog.users(user_id) ON DELETE CASCADE,
    post_id     UUID NOT NULL REFERENCES blog.posts(post_id) ON DELETE CASCADE,
    reaction    blog.reaction_type NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    
    PRIMARY KEY (user_id, post_id)
);

-- Reading History
CREATE TABLE blog.reading_history (
    user_id         UUID NOT NULL REFERENCES blog.users(user_id) ON DELETE CASCADE,
    post_id         UUID NOT NULL REFERENCES blog.posts(post_id) ON DELETE CASCADE,
    read_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    read_percentage SMALLINT CHECK (read_percentage BETWEEN 0 AND 100),
    
    PRIMARY KEY (user_id, post_id)
) PARTITION BY RANGE (read_at);

-- Create partitions for reading_history
CREATE TABLE blog.reading_history_2024 PARTITION OF blog.reading_history
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');
CREATE TABLE blog.reading_history_2025 PARTITION OF blog.reading_history
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');

-- Trigger: Auto-update updated_at
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER posts_updated_at BEFORE UPDATE ON blog.posts
    FOR EACH ROW EXECUTE FUNCTION update_updated_at();

CREATE TRIGGER users_updated_at BEFORE UPDATE ON blog.users
    FOR EACH ROW EXECUTE FUNCTION update_updated_at();

CREATE TRIGGER comments_updated_at BEFORE UPDATE ON blog.comments
    FOR EACH ROW EXECUTE FUNCTION update_updated_at();

-- Trigger: Update tag post_count
CREATE OR REPLACE FUNCTION update_tag_post_count()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        UPDATE blog.tags SET post_count = post_count + 1 WHERE tag_id = NEW.tag_id;
    ELSIF TG_OP = 'DELETE' THEN
        UPDATE blog.tags SET post_count = GREATEST(0, post_count - 1) WHERE tag_id = OLD.tag_id;
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER post_tags_count
AFTER INSERT OR DELETE ON blog.post_tags
FOR EACH ROW EXECUTE FUNCTION update_tag_post_count();

-- Sample data
INSERT INTO blog.users (username, email, password_hash, display_name, role) VALUES
    ('john_doe', 'john@example.com', '$2b$12$LQv3c1yqBWVHxkd0LHAkCOYz6TtxMQJqhN8/LewY5o4u0Yc5nHHEe', 'John Doe', 'author'),
    ('jane_smith', 'jane@example.com', '$2b$12$LQv3c1yqBWVHxkd0LHAkCOYz6TtxMQJqhN8/LewY5o4u0Yc5nHHEe', 'Jane Smith', 'editor'),
    ('admin_user', 'admin@example.com', '$2b$12$LQv3c1yqBWVHxkd0LHAkCOYz6TtxMQJqhN8/LewY5o4u0Yc5nHHEe', 'Admin', 'admin');

INSERT INTO blog.categories (name, slug, description) VALUES
    ('Technology', 'technology', 'Tech articles'),
    ('Programming', 'programming', 'Programming tutorials'),
    ('Database', 'database', 'Database articles');

-- Workshop Queries
-- Q1: หา posts ที่ published และ มีคนอ่านมากสุด
SELECT 
    p.title,
    u.display_name AS author,
    c.name AS category,
    p.view_count,
    p.like_count,
    p.reading_time
FROM blog.posts p
JOIN blog.users u ON p.author_id = u.user_id
LEFT JOIN blog.categories c ON p.category_id = c.category_id
WHERE p.status = 'published' AND p.deleted_at IS NULL
ORDER BY p.view_count DESC
LIMIT 10;

-- Q2: Full-text search
SELECT 
    post_id,
    title,
    ts_rank(
        to_tsvector('english', title || ' ' || content),
        plainto_tsquery('english', 'postgresql database')
    ) AS rank
FROM blog.posts
WHERE to_tsvector('english', title || ' ' || content) 
    @@ plainto_tsquery('english', 'postgresql database')
  AND status = 'published'
ORDER BY rank DESC;

-- Q3: Popular tags
SELECT 
    t.name,
    t.post_count,
    RANK() OVER (ORDER BY t.post_count DESC) AS rank
FROM blog.tags t
ORDER BY t.post_count DESC
LIMIT 20;
```

---

## สรุปบทที่ 4

| Type Category | Types | Use When |
|--------------|-------|----------|
| Integer | SMALLINT, INTEGER, BIGINT, SERIAL | IDs, counts, ratings |
| Decimal | NUMERIC, DECIMAL | เงิน, ค่าที่ต้องการความแม่นยำ |
| Float | REAL, FLOAT | coordinates, scientific values |
| String | VARCHAR, CHAR, TEXT | names, email, content |
| Boolean | BOOLEAN | flags, status |
| DateTime | DATE, TIME, TIMESTAMP, **TIMESTAMPTZ** | dates, times |
| UUID | UUID | distributed IDs, public-facing IDs |
| JSON | **JSONB** | flexible/semi-structured data |
| Array | TEXT[], INTEGER[] | tags, multiple values |
| Enum | ENUM | fixed set of values |
| Network | INET, CIDR, MACADDR | IP addresses |
| Range | daterange, tstzrange | time periods, price ranges |
| Binary | BYTEA | encrypted data, small files |

**บทต่อไป:** Primary Key, Foreign Key และ Indexes — optimize queries ให้เร็วขึ้น 10x!

# Part 32: JSON/JSONB Operations ใน PostgreSQL

## บทนำ

PostgreSQL รองรับ JSON data type ตั้งแต่ version 9.2 และ JSONB (binary JSON) ตั้งแต่ version 9.4 ทั้งสองอย่างช่วยให้คุณเก็บ semi-structured data ได้โดยตรงในฐานข้อมูลโดยไม่ต้องสร้าง EAV tables

---

## 1. JSON vs JSONB: Differences

### JSON

- เก็บเป็น plain text ไม่มีการ parse
- รักษา whitespace, key order, duplicate keys
- Insert เร็วกว่า (ไม่ต้อง parse)
- Query ช้ากว่า (ต้อง parse ทุกครั้ง)
- ไม่รองรับ GIN index บน operators บางตัว

### JSONB

- เก็บเป็น binary format
- ไม่รักษา whitespace, key order เรียงตาม alphabet
- ลบ duplicate keys (เก็บแค่ตัวสุดท้าย)
- Insert ช้ากว่าเล็กน้อย
- Query เร็วกว่ามาก (binary parsed)
- รองรับ GIN index ทุก operator

```sql
-- JSON vs JSONB
CREATE TABLE json_demo (
    id SERIAL PRIMARY KEY,
    data_json JSON,
    data_jsonb JSONB
);

-- Insert
INSERT INTO json_demo (data_json, data_jsonb) VALUES (
    '{"name": "Alice",  "age": 30, "name": "Bob"}',  -- JSON
    '{"name": "Alice",  "age": 30, "name": "Bob"}'   -- JSONB
);

-- ดูความแตกต่าง
SELECT 
    data_json,    -- {"name": "Alice",  "age": 30, "name": "Bob"} (preserved as-is)
    data_jsonb    -- {"age": 30, "name": "Bob"} (sorted keys, last dup wins)
FROM json_demo;

-- Size comparison
SELECT 
    pg_column_size('{"name":"Alice","age":30}'::json) AS json_size,
    pg_column_size('{"name":"Alice","age":30}'::jsonb) AS jsonb_size;
-- json: 26 bytes, jsonb: 53 bytes (JSONB uses more space for binary format)
```

### เมื่อไหร่ใช้อะไร

```sql
-- ใช้ JSON เมื่อ:
-- 1. ต้องการ preserve key order
-- 2. ข้อมูลเปลี่ยนบ่อย (write-heavy)
-- 3. ใช้แค่ store/retrieve ไม่ต้อง query ข้างใน

-- ใช้ JSONB เมื่อ:
-- 1. ต้องการ query/filter ข้างใน JSON
-- 2. ต้องการ index
-- 3. ใช้ containment operators (@>, <@)
-- 4. read-heavy workload

-- แนะนำ: ใช้ JSONB เกือบทุกกรณี
```

---

## 2. JSON Operators

### -> Operator (ดึง field เป็น JSON)

```sql
-- -> ดึง field ออกมาเป็น JSON type
SELECT '{"name":"Alice","age":30}'::json -> 'name';
-- "Alice" (JSON string, with quotes)

SELECT '{"user":{"id":1,"name":"Alice"}}'::json -> 'user';
-- {"id":1,"name":"Alice"} (JSON object)

-- ใช้กับ array index
SELECT '["a","b","c"]'::json -> 1;
-- "b"

SELECT '["a","b","c"]'::json -> -1;
-- "c" (negative = from end)

-- Chain operators
SELECT '{"user":{"profile":{"age":30}}}'::json -> 'user' -> 'profile' -> 'age';
-- 30
```

### ->> Operator (ดึง field เป็น text)

```sql
-- ->> ดึง field ออกมาเป็น text type (ไม่มี quotes)
SELECT '{"name":"Alice","age":30}'::json ->> 'name';
-- Alice (text, no quotes)

SELECT '{"age":30}'::json ->> 'age';
-- 30 (text "30", not integer)

-- ใช้ใน WHERE clause
SELECT * FROM products
WHERE data ->> 'category' = 'Electronics';

SELECT * FROM users
WHERE profile ->> 'country' = 'Thailand';

-- Cast ถ้าต้องการ type ที่ถูกต้อง
SELECT (data ->> 'price')::numeric FROM products WHERE id = 1;
SELECT (data ->> 'created_at')::timestamptz FROM orders;
```

### #> Operator (deep path, returns JSON)

```sql
-- #> ใช้ path array ดึง nested value เป็น JSON
SELECT '{"user":{"profile":{"age":30}}}'::json #> '{user,profile,age}';
-- 30

SELECT '{"items":[1,2,3]}'::json #> '{items,1}';
-- 2 (index 1)

-- เทียบกับ chain ->
SELECT '{"a":{"b":{"c":"deep"}}}'::json -> 'a' -> 'b' -> 'c';
SELECT '{"a":{"b":{"c":"deep"}}}'::json #> '{a,b,c}';
-- ทั้งคู่ได้ผลเหมือนกัน
```

### #>> Operator (deep path, returns text)

```sql
-- #>> ใช้ path array ดึง nested value เป็น text
SELECT '{"user":{"name":"Alice"}}'::json #>> '{user,name}';
-- Alice (text)

SELECT '{"settings":{"theme":{"color":"blue"}}}'::jsonb #>> '{settings,theme,color}';
-- blue
```

---

## 3. JSONB-Specific Operators

### @> Operator (containment)

```sql
-- @> left contains right
SELECT '{"a":1,"b":2}'::jsonb @> '{"a":1}'::jsonb;
-- true

SELECT '{"a":1,"b":2}'::jsonb @> '{"a":2}'::jsonb;
-- false (value ไม่ตรง)

SELECT '{"tags":["db","sql","pg"]}'::jsonb @> '{"tags":["sql"]}'::jsonb;
-- true

-- ใช้กับ WHERE
SELECT * FROM products
WHERE attributes @> '{"color":"red","size":"L"}';

-- ค้นหา products ที่มี tag 'sale'
SELECT * FROM products
WHERE tags @> '["sale"]';
```

### <@ Operator (contained by)

```sql
-- <@ right contains left
SELECT '{"a":1}'::jsonb <@ '{"a":1,"b":2}'::jsonb;
-- true (right contains left)

SELECT '{"a":1}'::jsonb <@ '{"a":2}'::jsonb;
-- false

-- ใช้หา subset
SELECT * FROM orders
WHERE '{"status":"pending","priority":"high"}' <@ metadata;
```

### ? Operator (key exists)

```sql
-- ? key exists
SELECT '{"name":"Alice","age":30}'::jsonb ? 'name';
-- true

SELECT '{"name":"Alice","age":30}'::jsonb ? 'email';
-- false

-- ใช้ใน WHERE
SELECT * FROM users WHERE profile ? 'phone_number';
SELECT * FROM products WHERE attributes ? 'discount_price';
```

### ?| Operator (any key exists)

```sql
-- ?| ถ้ามี key อย่างน้อย 1 ตัว
SELECT '{"a":1,"b":2}'::jsonb ?| ARRAY['a','c'];
-- true (มี 'a')

SELECT '{"a":1,"b":2}'::jsonb ?| ARRAY['c','d'];
-- false

-- ใช้ใน WHERE
SELECT * FROM products 
WHERE attributes ?| ARRAY['color','size','weight'];
```

### ?& Operator (all keys exist)

```sql
-- ?& ถ้ามี key ทุกตัว
SELECT '{"a":1,"b":2,"c":3}'::jsonb ?& ARRAY['a','b'];
-- true

SELECT '{"a":1,"b":2}'::jsonb ?& ARRAY['a','b','c'];
-- false (ไม่มี 'c')

-- ใช้ใน WHERE
SELECT * FROM products 
WHERE attributes ?& ARRAY['color','size'];  -- ต้องมีทั้ง color และ size
```

### @? Operator (JSON path exists)

```sql
-- @? ใช้ JSON Path syntax (PostgreSQL 12+)
SELECT '{"a":{"b":[1,2,3]}}'::jsonb @? '$.a.b[*]';
-- true

SELECT '{"users":[{"name":"Alice","age":30}]}'::jsonb @? '$.users[*].age ? (@ > 25)';
-- true

-- ค้นหา products ที่มี price > 100
SELECT * FROM products
WHERE data @? '$.price ? (@ > 100)';
```

---

## 4. JSON Path Functions

### jsonb_path_exists

```sql
-- ตรวจสอบว่า JSON path มีค่าไหม
SELECT jsonb_path_exists('{"a":1}', '$.a');
-- true

SELECT jsonb_path_exists('{"a":[1,2,3]}', '$.a[*] ? (@ > 2)');
-- true

-- ใช้ใน WHERE
SELECT * FROM orders
WHERE jsonb_path_exists(items, '$.* ? (@.quantity > 5)');
```

### jsonb_path_query

```sql
-- ดึงค่าทั้งหมดที่ match JSON path
SELECT jsonb_path_query('{"users":[{"name":"Alice"},{"name":"Bob"}]}', '$.users[*].name');
-- "Alice"
-- "Bob"

-- กรอง
SELECT jsonb_path_query(
    '[{"price":10,"name":"A"},{"price":200,"name":"B"},{"price":50,"name":"C"}]',
    '$[*] ? (@.price > 30)'
);
-- {"name":"B","price":200}
-- {"name":"C","price":50}

-- ใช้ตัวแปร
SELECT jsonb_path_query(
    '[1,2,3,4,5]',
    '$[*] ? (@ > $min)',
    '{"min": 3}'::jsonb
);
-- 4
-- 5
```

### jsonb_path_query_array

```sql
-- ดึงค่าทั้งหมดเป็น JSON array
SELECT jsonb_path_query_array(
    '{"items":[1,2,3,4,5]}',
    '$.items[*] ? (@ > 2)'
);
-- [3, 4, 5]

-- ดึงชื่อผู้ใช้ทั้งหมดเป็น array
SELECT jsonb_path_query_array(
    '{"users":[{"name":"Alice"},{"name":"Bob"},{"name":"Carol"}]}',
    '$.users[*].name'
);
-- ["Alice", "Bob", "Carol"]
```

### jsonb_path_query_first

```sql
-- ดึงค่าแรกที่ match
SELECT jsonb_path_query_first(
    '[{"score":80},{"score":95},{"score":72}]',
    '$[*] ? (@.score > 90)'
);
-- {"score": 95}
```

---

## 5. JSONB Modification Functions

### jsonb_set

```sql
-- Signature
-- jsonb_set(target jsonb, path text[], new_value jsonb, create_missing boolean DEFAULT true) → jsonb

-- Update nested field
SELECT jsonb_set(
    '{"name":"Alice","address":{"city":"Bangkok"}}',
    '{address,city}',
    '"Chiang Mai"'
);
-- {"name": "Alice", "address": {"city": "Chiang Mai"}}

-- Add new field
SELECT jsonb_set(
    '{"name":"Alice"}',
    '{email}',
    '"alice@example.com"'
);
-- {"name": "Alice", "email": "alice@example.com"}

-- ใช้ใน UPDATE
UPDATE users
SET profile = jsonb_set(profile, '{preferences,theme}', '"dark"')
WHERE id = 1;

-- Update หลาย fields
UPDATE products
SET attributes = jsonb_set(
    jsonb_set(attributes, '{price}', '99.99'::jsonb),
    '{in_stock}', 'true'::jsonb
)
WHERE id = 1;

-- jsonb_set_lax (PostgreSQL 14+) - ยืดหยุ่นกว่า
SELECT jsonb_set_lax(
    '{"a":1}',
    '{b}',
    NULL,
    true,
    'use_json_null'
);
-- {"a": 1, "b": null}
```

### jsonb_insert

```sql
-- Signature
-- jsonb_insert(target jsonb, path text[], new_value jsonb, insert_after boolean DEFAULT false) → jsonb

-- Insert ก่อน element
SELECT jsonb_insert(
    '{"a":[1,2,3]}',
    '{a,1}',
    '99'
);
-- {"a": [1, 99, 2, 3]}

-- Insert หลัง element
SELECT jsonb_insert(
    '{"a":[1,2,3]}',
    '{a,1}',
    '99',
    true
);
-- {"a": [1, 2, 99, 3]}

-- Append ที่ท้าย array
SELECT jsonb_insert(
    '{"tags":["a","b"]}',
    '{tags,-1}',
    '"c"',
    true
);
-- {"tags": ["a", "b", "c"]}

-- ใช้ใน UPDATE
UPDATE posts
SET metadata = jsonb_insert(metadata, '{tags,-1}', '"featured"'::jsonb, true)
WHERE id = 1;
```

### jsonb_delete (- operator)

```sql
-- ลบ key ด้วย - operator
SELECT '{"a":1,"b":2,"c":3}'::jsonb - 'b';
-- {"a": 1, "c": 3}

-- ลบหลาย keys
SELECT '{"a":1,"b":2,"c":3}'::jsonb - ARRAY['a','c'];
-- {"b": 2}

-- ลบ array element ด้วย index
SELECT '["a","b","c"]'::jsonb - 1;
-- ["a", "c"]

-- ลบ nested key ด้วย #- operator
SELECT '{"a":{"b":1,"c":2}}'::jsonb #- '{a,b}';
-- {"a": {"c": 2}}

-- ใช้ใน UPDATE
UPDATE users
SET profile = profile - 'temporary_password'
WHERE id = 1;

UPDATE products
SET attributes = attributes #- '{old_price}'
WHERE id = 1;
```

### jsonb_strip_nulls

```sql
-- ลบ keys ที่มี null value
SELECT jsonb_strip_nulls('{"a":1,"b":null,"c":{"d":null,"e":2}}');
-- {"a": 1, "c": {"e": 2}}

-- ใช้ก่อน insert
INSERT INTO users (profile)
VALUES (
    jsonb_strip_nulls(
        jsonb_build_object(
            'name', $1,
            'phone', $2,    -- อาจเป็น null
            'email', $3
        )
    )
);
```

---

## 6. JSON Aggregation Functions

### json_agg / jsonb_agg

```sql
-- รวมหลาย rows เป็น JSON array
SELECT json_agg(name ORDER BY name) AS names
FROM users;
-- ["Alice", "Bob", "Carol"]

SELECT jsonb_agg(
    jsonb_build_object('id', id, 'name', name, 'email', email)
    ORDER BY id
) AS users
FROM users
WHERE active = true;

-- ใช้ใน subquery
SELECT 
    o.id,
    o.total,
    (
        SELECT jsonb_agg(
            jsonb_build_object(
                'product_id', oi.product_id,
                'quantity', oi.quantity,
                'price', oi.price
            )
        )
        FROM order_items oi WHERE oi.order_id = o.id
    ) AS items
FROM orders o;
```

### json_object_agg / jsonb_object_agg

```sql
-- รวมเป็น JSON object จาก key-value pairs
SELECT json_object_agg(name, value) AS settings
FROM user_settings
WHERE user_id = 1;
-- {"theme": "dark", "language": "th", "notifications": "on"}

-- สร้าง pivot
SELECT 
    user_id,
    jsonb_object_agg(setting_name, setting_value) AS preferences
FROM user_settings
GROUP BY user_id;

-- ตัวอย่างซับซ้อน
SELECT 
    category,
    jsonb_object_agg(
        status,
        count
    ) AS counts_by_status
FROM (
    SELECT category, status, COUNT(*) AS count
    FROM orders
    GROUP BY category, status
) t
GROUP BY category;
```

### json_build_object / jsonb_build_object

```sql
-- สร้าง JSON object จาก key-value pairs
SELECT json_build_object(
    'name', 'Alice',
    'age', 30,
    'active', true
);
-- {"name": "Alice", "age": 30, "active": true}

SELECT jsonb_build_object(
    'user', jsonb_build_object('id', id, 'name', name),
    'orders', (SELECT COUNT(*) FROM orders WHERE user_id = users.id)
)
FROM users WHERE id = 1;
```

### json_build_array / jsonb_build_array

```sql
-- สร้าง JSON array
SELECT json_build_array(1, 'two', true, NULL);
-- [1, "two", true, null]

SELECT jsonb_build_array(
    jsonb_build_object('id', 1, 'name', 'Alice'),
    jsonb_build_object('id', 2, 'name', 'Bob')
);
```

---

## 7. JSON Functions: jsonb_each, jsonb_keys, jsonb_array_elements

### jsonb_each

```sql
-- แตก JSONB object เป็น key-value rows
SELECT * FROM jsonb_each('{"a":1,"b":"two","c":true}');
-- key | value
-- a   | 1
-- b   | "two"
-- c   | true

SELECT * FROM jsonb_each_text('{"name":"Alice","age":"30"}');
-- key  | value
-- name | Alice
-- age  | 30

-- ใช้กับ table
SELECT p.id, e.key, e.value
FROM products p, jsonb_each(p.attributes) e
WHERE e.key IN ('color', 'size');
```

### jsonb_object_keys

```sql
-- ดึง keys ทั้งหมด
SELECT jsonb_object_keys('{"a":1,"b":2,"c":3}');
-- a
-- b
-- c

-- ดู schema ของ JSON column
SELECT DISTINCT jsonb_object_keys(attributes)
FROM products
ORDER BY 1;
```

### jsonb_array_elements

```sql
-- แตก JSON array เป็นหลาย rows
SELECT * FROM jsonb_array_elements('[1,2,3,"four",true]');
-- value
-- 1
-- 2
-- 3
-- "four"
-- true

SELECT * FROM jsonb_array_elements_text('["Alice","Bob","Carol"]');
-- value
-- Alice
-- Bob
-- Carol

-- ใช้กับ table
SELECT p.id, p.name, tag.value AS tag
FROM products p, jsonb_array_elements_text(p.tags) AS tag
WHERE tag.value = 'sale';
```

### jsonb_to_record / jsonb_populate_record

```sql
-- แปลง JSONB เป็น row
SELECT * FROM jsonb_to_record(
    '{"name":"Alice","age":30,"active":true}'
) AS x(name text, age int, active boolean);
-- name  | age | active
-- Alice | 30  | true

-- populate record type
CREATE TYPE person AS (name text, age int);
SELECT * FROM jsonb_populate_record(
    null::person,
    '{"name":"Alice","age":30,"extra":"ignored"}'
);
```

---

## 8. Cast: text::json, ::jsonb

```sql
-- Cast string ไป JSON
SELECT '{"name":"Alice"}'::json;
SELECT '{"name":"Alice"}'::jsonb;
SELECT CAST('{"name":"Alice"}' AS jsonb);

-- Cast JSON ไป text
SELECT '{"name":"Alice"}'::jsonb::text;

-- Cast JSON ไป specific types
SELECT ('{"price":99.99}'::jsonb ->> 'price')::numeric;
SELECT ('{"count":42}'::jsonb ->> 'count')::integer;
SELECT ('{"active":true}'::jsonb ->> 'active')::boolean;
SELECT ('{"date":"2024-01-01"}'::jsonb ->> 'date')::date;

-- to_json / to_jsonb
SELECT to_json(42);         -- 42
SELECT to_json('hello');    -- "hello"
SELECT to_json(NOW());      -- "2024-01-01T00:00:00+00:00"
SELECT to_jsonb(ARRAY[1,2,3]);  -- [1, 2, 3]

-- row_to_json
SELECT row_to_json(u) FROM users u WHERE id = 1;
-- {"id":1,"name":"Alice","email":"alice@example.com",...}

SELECT row_to_json(t) FROM (
    SELECT id, name, email FROM users WHERE id = 1
) t;
```

---

## 9. Indexing JSONB

### GIN Index สำหรับ @> queries

```sql
-- GIN index with jsonb_ops (default)
CREATE INDEX idx_products_attributes_gin 
ON products USING GIN(attributes);

-- รองรับ operators: @>, <@, ?, ?|, ?&
-- ค้นหา products ที่มี color = red
SELECT * FROM products
WHERE attributes @> '{"color":"red"}';
-- ใช้ index

-- jsonb_path_ops: เฉพาะ @> operator แต่เร็วกว่าและ index เล็กกว่า
CREATE INDEX idx_products_attributes_path_ops
ON products USING GIN(attributes jsonb_path_ops);

-- เปรียบเทียบขนาด
SELECT 
    indexname,
    pg_size_pretty(pg_relation_size(indexname::regclass)) AS size
FROM pg_indexes
WHERE tablename = 'products';
```

### Expression Index สำหรับ Specific Fields

```sql
-- Index บน specific field (เหมาะถ้าค้นหา field เดิมบ่อย)
CREATE INDEX idx_products_color
ON products ((attributes ->> 'color'));

CREATE INDEX idx_products_price
ON products (((attributes ->> 'price')::numeric));

-- ใช้ index
SELECT * FROM products 
WHERE attributes ->> 'color' = 'red';
-- ใช้ expression index

SELECT * FROM products 
WHERE (attributes ->> 'price')::numeric > 100;
-- ใช้ expression index
```

### Partial Index

```sql
-- Index เฉพาะ rows ที่มี key นั้น
CREATE INDEX idx_products_discount
ON products ((attributes ->> 'discount_price')::numeric)
WHERE attributes ? 'discount_price';

-- ใช้ index
SELECT * FROM products
WHERE attributes ? 'discount_price'
  AND (attributes ->> 'discount_price')::numeric < 50;
```

### jsonb_path_ops vs jsonb_ops

```sql
-- jsonb_ops (default)
-- + รองรับ: @>, <@, ?, ?|, ?&
-- - index ใหญ่กว่า
-- - ช้ากว่าสำหรับ @>

-- jsonb_path_ops  
-- + index เล็กกว่า ~30%
-- + เร็วกว่าสำหรับ @>
-- - รองรับเฉพาะ @>

-- เลือกใช้ jsonb_path_ops ถ้าใช้แค่ @>
CREATE INDEX idx_orders_metadata_path_ops
ON orders USING GIN(metadata jsonb_path_ops);

-- EXPLAIN ANALYZE
EXPLAIN ANALYZE
SELECT * FROM products 
WHERE attributes @> '{"color":"red","size":"L"}';
```

---

## 10. JSON Schema Validation (PostgreSQL 16+)

```sql
-- ตรวจสอบ JSON schema (PostgreSQL 16+)
SELECT jsonb_matches_schema(
    '{"type":"object","properties":{"name":{"type":"string"},"age":{"type":"integer"}}}',
    '{"name":"Alice","age":30}'
);
-- true

SELECT jsonb_matches_schema(
    '{"type":"object","required":["name","email"]}',
    '{"name":"Alice"}'
);
-- false (missing email)

-- ใช้ใน constraint
ALTER TABLE users ADD CONSTRAINT check_profile_schema
    CHECK (jsonb_matches_schema(
        '{"type":"object",
          "properties":{
            "name":{"type":"string","minLength":1},
            "age":{"type":"integer","minimum":0,"maximum":150},
            "email":{"type":"string","format":"email"}
          }
        }',
        profile
    ));

-- jsonb_valid_schema
SELECT jsonb_valid_schema(
    '{"type":"object","required":["name"]}'
);
-- true
```

---

## 11. JSONB vs EAV Pattern

### EAV (Entity-Attribute-Value) Pattern

```sql
-- EAV table (แบบเก่า)
CREATE TABLE product_attributes_eav (
    product_id INTEGER REFERENCES products(id),
    attribute_name TEXT NOT NULL,
    attribute_value TEXT,
    PRIMARY KEY (product_id, attribute_name)
);

-- Insert
INSERT INTO product_attributes_eav VALUES 
    (1, 'color', 'red'),
    (1, 'size', 'L'),
    (1, 'weight', '0.5kg');

-- Query (ยุ่งยาก)
SELECT 
    p.name,
    MAX(CASE WHEN a.attribute_name = 'color' THEN a.attribute_value END) AS color,
    MAX(CASE WHEN a.attribute_name = 'size' THEN a.attribute_value END) AS size
FROM products p
LEFT JOIN product_attributes_eav a ON a.product_id = p.id
WHERE p.id = 1
GROUP BY p.id, p.name;
```

### JSONB Pattern (แบบใหม่)

```sql
-- JSONB (ง่ายกว่ามาก)
ALTER TABLE products ADD COLUMN attributes JSONB DEFAULT '{}';

UPDATE products SET attributes = '{"color":"red","size":"L","weight":"0.5kg"}' WHERE id = 1;

-- Query ง่ายกว่า
SELECT name, attributes ->> 'color', attributes ->> 'size'
FROM products WHERE id = 1;

-- Search ง่ายกว่า
SELECT * FROM products WHERE attributes @> '{"color":"red"}';

-- เปรียบเทียบประสิทธิภาพ:
-- EAV: ต้องทำ many joins, complex pivoting
-- JSONB: simple operators, GIN index รองรับ
```

---

## 12. Hybrid Schema: Relational + JSONB

```sql
-- Best practice: เก็บ structured data ใน columns + flexible data ใน JSONB
CREATE TABLE products (
    -- Structured columns (ค้นหาบ่อย, join, filter)
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    category_id INTEGER REFERENCES categories(id),
    price NUMERIC(10,2) NOT NULL,
    stock_quantity INTEGER DEFAULT 0,
    status TEXT DEFAULT 'active',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    
    -- JSONB สำหรับ variable attributes (แตกต่างตาม category)
    attributes JSONB DEFAULT '{}',
    -- เช่น Electronics: {"brand":"Sony","model":"A7IV","sensor":"full-frame"}
    -- เช่น Clothing: {"color":"blue","size":"L","material":"cotton"}
    -- เช่น Book: {"author":"John","isbn":"978-xxx","pages":300}
    
    -- JSONB สำหรับ metadata
    metadata JSONB DEFAULT '{}'
    -- เช่น {"seo_title":"...","seo_description":"...","featured":true}
);

-- Indexes
CREATE INDEX idx_products_category ON products(category_id);
CREATE INDEX idx_products_price ON products(price);
CREATE INDEX idx_products_attributes ON products USING GIN(attributes);

-- ค้นหา
SELECT * FROM products
WHERE 
    category_id = 5          -- relational filter (index)
    AND price BETWEEN 100 AND 500  -- relational filter (index)
    AND attributes @> '{"color":"blue"}';  -- JSONB filter (GIN)
```

---

## 13. Real Example: Product Attributes

```sql
-- E-commerce product catalog

-- Categories
CREATE TABLE categories (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    slug TEXT NOT NULL UNIQUE,
    attribute_schema JSONB  -- defines valid attributes for this category
);

INSERT INTO categories (name, slug, attribute_schema) VALUES
(
    'Electronics',
    'electronics',
    '{"brand":{"type":"string"},"model":{"type":"string"},"warranty_months":{"type":"integer"}}'
),
(
    'Clothing',
    'clothing',
    '{"color":{"type":"string"},"size":{"type":"string","enum":["XS","S","M","L","XL","XXL"]},"material":{"type":"string"}}'
),
(
    'Books',
    'books',
    '{"author":{"type":"string"},"isbn":{"type":"string"},"publisher":{"type":"string"},"pages":{"type":"integer"}}'
);

-- Products
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    slug TEXT NOT NULL UNIQUE,
    description TEXT,
    category_id INTEGER REFERENCES categories(id),
    price NUMERIC(10,2) NOT NULL,
    compare_price NUMERIC(10,2),
    stock INTEGER DEFAULT 0,
    images JSONB DEFAULT '[]',  -- array of image objects
    attributes JSONB DEFAULT '{}',  -- category-specific attributes
    tags TEXT[] DEFAULT '{}',
    status TEXT DEFAULT 'draft',
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Index
CREATE INDEX idx_products_gin ON products USING GIN(attributes);
CREATE INDEX idx_products_tags ON products USING GIN(tags);
CREATE INDEX idx_products_category ON products(category_id);

-- Sample data
INSERT INTO products (name, category_id, price, attributes, tags) VALUES
(
    'Sony Alpha 7 IV',
    1,  -- Electronics
    89990,
    '{
        "brand": "Sony",
        "model": "ILCE-7M4",
        "sensor": "Full-frame BSI CMOS",
        "megapixels": 33,
        "video_resolution": "4K",
        "battery_life": 520,
        "weight_grams": 658,
        "warranty_months": 12
    }',
    ARRAY['camera', 'mirrorless', 'full-frame', 'sony']
),
(
    'Levi''s 501 Original',
    2,  -- Clothing
    2590,
    '{
        "brand": "Levis",
        "size": "32x32",
        "color": "Medium Wash",
        "material": "100% Cotton",
        "fit": "Straight",
        "rise": "Mid"
    }',
    ARRAY['jeans', 'denim', 'levis', 'casual']
),
(
    'PostgreSQL: Up and Running',
    3,  -- Books
    1299,
    '{
        "author": "Regina O. Obe",
        "isbn": "978-1491963418",
        "publisher": "O''Reilly Media",
        "pages": 378,
        "edition": 3,
        "language": "English"
    }',
    ARRAY['book', 'postgresql', 'database', 'technical']
);

-- Queries
-- 1. ค้นหา electronics ที่มี warranty >= 12 เดือน
SELECT name, price, attributes ->> 'brand' AS brand
FROM products
WHERE 
    category_id = 1
    AND (attributes ->> 'warranty_months')::integer >= 12
ORDER BY price;

-- 2. ค้นหา clothing size L
SELECT name, price
FROM products
WHERE 
    category_id = 2
    AND attributes @> '{"size":"L"}';

-- 3. ค้นหา products ที่มี rating >= 4 (ถ้าเก็บ rating ใน attributes)
SELECT name, price, (attributes ->> 'avg_rating')::numeric AS rating
FROM products
WHERE (attributes ->> 'avg_rating')::numeric >= 4
ORDER BY rating DESC;

-- 4. Aggregation
SELECT 
    c.name AS category,
    COUNT(*) AS product_count,
    AVG(price) AS avg_price,
    jsonb_agg(DISTINCT attributes ->> 'brand') FILTER (WHERE attributes ? 'brand') AS brands
FROM products p
JOIN categories c ON c.id = p.category_id
GROUP BY c.id, c.name;
```

---

## 14. Migration from TEXT/JSON to JSONB

```sql
-- Migration strategy

-- 1. เพิ่ม JSONB column
ALTER TABLE products ADD COLUMN attributes_jsonb JSONB;

-- 2. Migrate data
UPDATE products 
SET attributes_jsonb = attributes_text::jsonb
WHERE attributes_text IS NOT NULL AND attributes_text != '';

-- 3. Validate migration
SELECT COUNT(*) AS total,
       COUNT(attributes_jsonb) AS migrated,
       COUNT(*) FILTER (WHERE attributes_text IS NOT NULL AND attributes_jsonb IS NULL) AS failed
FROM products;

-- 4. สร้าง index
CREATE INDEX idx_products_attributes_jsonb ON products USING GIN(attributes_jsonb);

-- 5. Update application code แล้ว drop old column
ALTER TABLE products DROP COLUMN attributes_text;
ALTER TABLE products RENAME COLUMN attributes_jsonb TO attributes;

-- Migration validation
DO $$
DECLARE
    v_count INTEGER;
BEGIN
    SELECT COUNT(*) INTO v_count
    FROM products
    WHERE attributes IS NULL AND old_attributes IS NOT NULL;
    
    IF v_count > 0 THEN
        RAISE EXCEPTION 'Migration failed: % rows not migrated', v_count;
    END IF;
    
    RAISE NOTICE 'Migration successful!';
END;
$$;
```

---

## 15. Full Working Example: E-commerce Product Catalog

### Complete SQL Schema

```sql
-- ========================================
-- E-Commerce Product Catalog
-- ========================================

-- Extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS unaccent;

-- Categories
CREATE TABLE categories (
    id SERIAL PRIMARY KEY,
    parent_id INTEGER REFERENCES categories(id),
    name TEXT NOT NULL,
    slug TEXT NOT NULL UNIQUE,
    description TEXT,
    image_url TEXT,
    attributes_schema JSONB DEFAULT '{}',
    sort_order INTEGER DEFAULT 0,
    is_active BOOLEAN DEFAULT true
);

-- Brands
CREATE TABLE brands (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL UNIQUE,
    slug TEXT NOT NULL UNIQUE,
    logo_url TEXT,
    description TEXT,
    metadata JSONB DEFAULT '{}'
);

-- Products
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    uuid UUID DEFAULT uuid_generate_v4() UNIQUE,
    name TEXT NOT NULL,
    slug TEXT NOT NULL UNIQUE,
    sku TEXT UNIQUE,
    description TEXT,
    category_id INTEGER REFERENCES categories(id),
    brand_id INTEGER REFERENCES brands(id),
    
    -- Pricing
    price NUMERIC(12,2) NOT NULL,
    compare_price NUMERIC(12,2),
    cost_price NUMERIC(12,2),
    
    -- Inventory
    stock_quantity INTEGER DEFAULT 0,
    track_inventory BOOLEAN DEFAULT true,
    
    -- Media
    images JSONB DEFAULT '[]',
    -- [{"url":"...", "alt":"...", "is_primary":true, "sort":1}, ...]
    
    -- Variable attributes (per category)
    attributes JSONB DEFAULT '{}',
    
    -- SEO & Marketing
    tags TEXT[] DEFAULT '{}',
    metadata JSONB DEFAULT '{}',
    -- {"seo_title":"...", "seo_desc":"...", "featured":false, "new_arrival":true}
    
    -- Status
    status TEXT DEFAULT 'draft' CHECK (status IN ('draft', 'active', 'archived')),
    published_at TIMESTAMPTZ,
    
    -- Timestamps
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Product variants (e.g., different sizes/colors of same product)
CREATE TABLE product_variants (
    id SERIAL PRIMARY KEY,
    product_id INTEGER REFERENCES products(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    sku TEXT UNIQUE,
    price NUMERIC(12,2),
    stock_quantity INTEGER DEFAULT 0,
    attributes JSONB DEFAULT '{}',
    -- {"color":"red","size":"L","weight":"0.5kg"}
    is_default BOOLEAN DEFAULT false,
    sort_order INTEGER DEFAULT 0
);

-- Reviews
CREATE TABLE product_reviews (
    id SERIAL PRIMARY KEY,
    product_id INTEGER REFERENCES products(id) ON DELETE CASCADE,
    user_id INTEGER,
    rating SMALLINT NOT NULL CHECK (rating BETWEEN 1 AND 5),
    title TEXT,
    body TEXT,
    verified_purchase BOOLEAN DEFAULT false,
    helpful_count INTEGER DEFAULT 0,
    metadata JSONB DEFAULT '{}',
    -- {"pros":["fast","reliable"], "cons":["expensive"]}
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_products_category ON products(category_id) WHERE status = 'active';
CREATE INDEX idx_products_brand ON products(brand_id) WHERE status = 'active';
CREATE INDEX idx_products_price ON products(price) WHERE status = 'active';
CREATE INDEX idx_products_attributes ON products USING GIN(attributes);
CREATE INDEX idx_products_tags ON products USING GIN(tags);
CREATE INDEX idx_products_metadata ON products USING GIN(metadata);
CREATE INDEX idx_products_status_published ON products(status, published_at DESC);
CREATE INDEX idx_variants_product ON product_variants(product_id);
CREATE INDEX idx_variants_attributes ON product_variants USING GIN(attributes);

-- Triggers
CREATE OR REPLACE FUNCTION products_updated_at()
RETURNS TRIGGER AS $$
BEGIN NEW.updated_at = NOW(); RETURN NEW; END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER products_updated_at
    BEFORE UPDATE ON products
    FOR EACH ROW EXECUTE FUNCTION products_updated_at();

-- Views
CREATE VIEW product_catalog AS
SELECT 
    p.id,
    p.uuid,
    p.name,
    p.slug,
    p.sku,
    p.price,
    p.compare_price,
    p.stock_quantity,
    c.name AS category_name,
    c.slug AS category_slug,
    b.name AS brand_name,
    p.images -> 0 AS primary_image,
    p.attributes,
    p.tags,
    p.status,
    p.published_at,
    COALESCE(r.avg_rating, 0) AS avg_rating,
    COALESCE(r.review_count, 0) AS review_count
FROM products p
LEFT JOIN categories c ON c.id = p.category_id
LEFT JOIN brands b ON b.id = p.brand_id
LEFT JOIN (
    SELECT product_id,
           AVG(rating)::numeric(3,1) AS avg_rating,
           COUNT(*) AS review_count
    FROM product_reviews
    GROUP BY product_id
) r ON r.product_id = p.id;
```

### Node.js/TypeScript API

```typescript
// product-catalog.ts

import { Pool } from 'pg';

interface ProductFilters {
  categoryId?: number;
  brandId?: number;
  minPrice?: number;
  maxPrice?: number;
  attributes?: Record<string, any>;
  tags?: string[];
  inStock?: boolean;
  minRating?: number;
  status?: string;
}

interface ProductSort {
  field: 'price' | 'created_at' | 'name' | 'avg_rating';
  direction: 'ASC' | 'DESC';
}

interface Pagination {
  page: number;
  pageSize: number;
}

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
});

async function getProducts(
  filters: ProductFilters = {},
  sort: ProductSort = { field: 'created_at', direction: 'DESC' },
  pagination: Pagination = { page: 1, pageSize: 20 }
): Promise<{ products: any[]; total: number }> {
  const conditions: string[] = ["p.status = 'active'"];
  const params: any[] = [];
  let paramIndex = 1;

  if (filters.categoryId) {
    conditions.push(`p.category_id = $${paramIndex++}`);
    params.push(filters.categoryId);
  }

  if (filters.brandId) {
    conditions.push(`p.brand_id = $${paramIndex++}`);
    params.push(filters.brandId);
  }

  if (filters.minPrice !== undefined) {
    conditions.push(`p.price >= $${paramIndex++}`);
    params.push(filters.minPrice);
  }

  if (filters.maxPrice !== undefined) {
    conditions.push(`p.price <= $${paramIndex++}`);
    params.push(filters.maxPrice);
  }

  if (filters.attributes && Object.keys(filters.attributes).length > 0) {
    conditions.push(`p.attributes @> $${paramIndex++}::jsonb`);
    params.push(JSON.stringify(filters.attributes));
  }

  if (filters.tags && filters.tags.length > 0) {
    conditions.push(`p.tags @> $${paramIndex++}::text[]`);
    params.push(filters.tags);
  }

  if (filters.inStock) {
    conditions.push(`p.stock_quantity > 0`);
  }

  if (filters.minRating) {
    conditions.push(`COALESCE(r.avg_rating, 0) >= $${paramIndex++}`);
    params.push(filters.minRating);
  }

  const offset = (pagination.page - 1) * pagination.pageSize;
  params.push(pagination.pageSize, offset);

  const whereClause = conditions.length > 0 ? `WHERE ${conditions.join(' AND ')}` : '';
  const orderClause = `ORDER BY ${sort.field} ${sort.direction}`;
  const limitClause = `LIMIT $${paramIndex++} OFFSET $${paramIndex}`;

  const query = `
    WITH ranked AS (
      SELECT 
        p.*,
        c.name AS category_name,
        b.name AS brand_name,
        p.images -> 0 AS primary_image,
        COALESCE(r.avg_rating, 0) AS avg_rating,
        COALESCE(r.review_count, 0) AS review_count,
        COUNT(*) OVER() AS total_count
      FROM products p
      LEFT JOIN categories c ON c.id = p.category_id
      LEFT JOIN brands b ON b.id = p.brand_id
      LEFT JOIN (
        SELECT product_id,
               AVG(rating)::numeric(3,1) AS avg_rating,
               COUNT(*) AS review_count
        FROM product_reviews
        GROUP BY product_id
      ) r ON r.product_id = p.id
      ${whereClause}
    )
    SELECT * FROM ranked
    ${orderClause}
    ${limitClause}
  `;

  const result = await pool.query(query, params);

  return {
    products: result.rows,
    total: result.rows[0]?.total_count || 0,
  };
}

async function updateProductAttribute(
  productId: number,
  attributePath: string[],
  value: any
): Promise<void> {
  await pool.query(
    `UPDATE products 
     SET attributes = jsonb_set(attributes, $1::text[], $2::jsonb)
     WHERE id = $3`,
    [attributePath, JSON.stringify(value), productId]
  );
}

async function addProductTag(productId: number, tag: string): Promise<void> {
  await pool.query(
    `UPDATE products 
     SET tags = array_append(tags, $1)
     WHERE id = $2 AND NOT ($1 = ANY(tags))`,
    [tag, productId]
  );
}

async function getAttributeFacets(categoryId: number): Promise<Record<string, any[]>> {
  const result = await pool.query(
    `SELECT 
       key,
       jsonb_agg(DISTINCT value ORDER BY value) AS values
     FROM (
       SELECT e.key, e.value
       FROM products p, jsonb_each_text(p.attributes) AS e
       WHERE p.category_id = $1 AND p.status = 'active'
     ) kv
     GROUP BY key
     ORDER BY key`,
    [categoryId]
  );

  const facets: Record<string, any[]> = {};
  for (const row of result.rows) {
    facets[row.key] = row.values;
  }
  return facets;
}

async function getPriceRange(categoryId?: number): Promise<{ min: number; max: number }> {
  const result = await pool.query(
    `SELECT 
       MIN(price) AS min_price, 
       MAX(price) AS max_price
     FROM products
     WHERE status = 'active'
     ${categoryId ? 'AND category_id = $1' : ''}`,
    categoryId ? [categoryId] : []
  );

  return {
    min: parseFloat(result.rows[0].min_price) || 0,
    max: parseFloat(result.rows[0].max_price) || 0,
  };
}

export {
  getProducts,
  updateProductAttribute,
  addProductTag,
  getAttributeFacets,
  getPriceRange,
};
```

---

## 16. Advanced JSONB Queries

```sql
-- 1. JSONB diff (หา keys ที่เปลี่ยน)
SELECT key
FROM jsonb_each('{"a":1,"b":2,"c":3}'::jsonb) AS old
WHERE old.value != ('{"a":1,"b":99,"d":4}'::jsonb -> old.key)
   OR NOT ('{"a":1,"b":99,"d":4}'::jsonb ? old.key);

-- 2. Merge JSONB objects
SELECT '{"a":1,"b":2}'::jsonb || '{"b":99,"c":3}'::jsonb;
-- {"a": 1, "b": 99, "c": 3} (right overwrites left)

-- 3. Deep merge (custom function)
CREATE OR REPLACE FUNCTION jsonb_deep_merge(a JSONB, b JSONB)
RETURNS JSONB AS $$
DECLARE
    result JSONB := a;
    key TEXT;
BEGIN
    FOR key IN SELECT jsonb_object_keys(b) LOOP
        IF result -> key IS NOT NULL 
           AND jsonb_typeof(result -> key) = 'object' 
           AND jsonb_typeof(b -> key) = 'object' THEN
            result := jsonb_set(result, ARRAY[key], jsonb_deep_merge(result -> key, b -> key));
        ELSE
            result := result || jsonb_build_object(key, b -> key);
        END IF;
    END LOOP;
    RETURN result;
END;
$$ LANGUAGE plpgsql IMMUTABLE;

-- 4. Flatten nested JSONB
SELECT jsonb_each.key || '.' || attr_key AS flat_key, attr_value
FROM products,
     jsonb_each(attributes),
     jsonb_each(CASE jsonb_typeof(value) WHEN 'object' THEN value ELSE '"{}"'::jsonb END) AS nested(attr_key, attr_value)
WHERE id = 1;

-- 5. JSON statistics
SELECT 
    COUNT(*) AS total_products,
    COUNT(attributes) AS has_attributes,
    COUNT(*) FILTER (WHERE attributes @> '{"brand":"Sony"}') AS sony_products,
    AVG(jsonb_array_length(images)) AS avg_images,
    percentile_cont(0.5) WITHIN GROUP (ORDER BY (attributes ->> 'weight')::numeric) AS median_weight
FROM products;
```

---

## สรุป

JSON/JSONB ใน PostgreSQL เป็นเครื่องมือที่ทรงพลังสำหรับ semi-structured data:

1. **JSONB เกือบดีกว่า JSON เสมอ** - เร็วกว่า, รองรับ index, operators มากกว่า
2. **GIN index** จำเป็นสำหรับ performance เมื่อ query ข้างใน JSON
3. **Hybrid schema** - ใช้ relational columns สำหรับ fields ที่ค้นหาบ่อย + JSONB สำหรับ flexible data
4. **jsonb_set, jsonb_insert, -** ใช้ modify JSONB
5. **@>, ?, ?|, ?&** เป็น operators หลักสำหรับ containment และ key existence
6. **JSON Path ($.x.y)** สำหรับ complex queries (PostgreSQL 12+)
7. **jsonb_agg, json_object_agg** สำหรับ aggregation
8. **websearch_to_tsquery** + JSONB = powerful search capabilities

# Part 30: Indexing Strategies — B-Tree, Hash, GIN, GiST, BRIN

## Index คืออะไร?

Index คือโครงสร้างข้อมูลที่ช่วยให้ PostgreSQL หาข้อมูลได้เร็วขึ้น โดยไม่ต้อง scan ทุก row ใน table

**เปรียบเทียบ:** Index ในหนังสือ — แทนที่จะพลิกหาทุกหน้า ดูสารบัญแล้วกระโดดไปหน้าที่ต้องการ

### ต้นทุนของ Index

- **Write overhead**: INSERT/UPDATE/DELETE ต้อง update index ด้วย
- **Disk space**: Index ใช้พื้นที่ disk เพิ่มเติม
- **Maintenance**: VACUUM ต้อง maintain index ด้วย
- **Query planner complexity**: Planner ต้องตัดสินใจว่าจะใช้ index ไหน

---

## B-Tree Index (Default)

### โครงสร้าง

B-Tree (Balanced Tree) เป็น default index type ใน PostgreSQL

```
               [50]
              /    \
         [25]      [75]
         /  \      /  \
      [10] [40] [60] [90]
      / \   / \   / \   / \
    [5][15][30][45][55][65][80][95]
```

- **Balanced**: ทุก leaf node อยู่ระดับเดียวกัน
- **Self-balancing**: insert/delete ทำ tree ยัง balanced อยู่
- **Sorted**: ข้อมูลเรียงลำดับ → range queries เร็ว
- **Height**: O(log n) — table 1 billion rows อาจสูงแค่ 30 levels

### Best For

```sql
-- Equality (=)
SELECT * FROM products WHERE id = 'abc123';
SELECT * FROM users WHERE email = 'alice@example.com';

-- Range (<, >, <=, >=, BETWEEN)
SELECT * FROM orders WHERE created_at BETWEEN '2024-01-01' AND '2024-12-31';
SELECT * FROM products WHERE price BETWEEN 100 AND 500;

-- Sorting (ORDER BY)
-- ถ้ามี index บน column ที่ ORDER BY → ไม่ต้อง sort เพิ่ม!
SELECT * FROM products ORDER BY price ASC;

-- Prefix LIKE
SELECT * FROM products WHERE name LIKE 'iPhone%';  -- index ได้
-- SELECT * FROM products WHERE name LIKE '%iPhone%';  -- index ไม่ได้

-- IN (แปลง IN เป็น = หลายๆ ครั้ง)
SELECT * FROM orders WHERE status IN ('pending', 'confirmed');

-- IS NULL / IS NOT NULL
SELECT * FROM users WHERE deleted_at IS NULL;

-- NOT NULL constraint
SELECT * FROM products WHERE sku IS NOT NULL;
```

### สร้าง B-Tree Index

```sql
-- Basic index
CREATE INDEX products_price_idx ON products(price);

-- Unique index
CREATE UNIQUE INDEX users_email_unique_idx ON users(email);

-- Composite index (column order สำคัญมาก!)
CREATE INDEX orders_user_status_idx ON orders(user_id, status);

-- Descending index
CREATE INDEX products_created_at_desc_idx ON products(created_at DESC);

-- Concurrent (ไม่ lock table ระหว่าง create)
CREATE INDEX CONCURRENTLY products_category_price_idx 
  ON products(category_id, price);
```

### Composite Index Column Order

```sql
-- Rule: Column ที่ใช้ equality filter ก่อน, range filter ทีหลัง

-- ✅ ดี: user_id (equality) มาก่อน created_at (range)
CREATE INDEX orders_user_created_idx ON orders(user_id, created_at DESC);

-- Query นี้ใช้ index ได้เต็มที่:
SELECT * FROM orders
WHERE user_id = 'user-123'
  AND created_at > NOW() - INTERVAL '30 days'
ORDER BY created_at DESC;

-- ❌ แย่: created_at (range) มาก่อน user_id
CREATE INDEX orders_created_user_idx ON orders(created_at, user_id);
-- Query ด้านบนจะไม่ใช้ user_id part ของ index
```

---

## Hash Index

### โครงสร้าง

Hash index เก็บ hash value ของ indexed value

```
Value → Hash Function → Hash Bucket → Row Pointer
"alice@test.com" → hash() → Bucket 7 → (page=42, offset=16)
```

### Best For

```sql
-- Equality ONLY (ไม่รองรับ range)
SELECT * FROM users WHERE email = 'alice@example.com';
-- Hash index เร็วกว่า B-Tree สำหรับ equality

-- สร้าง Hash Index
CREATE INDEX users_email_hash_idx ON users USING HASH (email);

-- ดู query plan
EXPLAIN SELECT * FROM users WHERE email = 'alice@example.com';
-- Index Scan using users_email_hash_idx on users
```

### ข้อควรระวัง Hash Index

```sql
-- ❌ Hash index ไม่รองรับ range queries
-- SELECT * FROM products WHERE price > 100;  -- จะไม่ใช้ Hash index

-- ❌ Hash index ไม่รองรับ ORDER BY
-- SELECT * FROM products ORDER BY name;  -- จะไม่ใช้ Hash index

-- ✅ เหมาะสำหรับ equality สำหรับ columns ที่มีค่าหลากหลายมาก (high cardinality)
CREATE INDEX sessions_token_hash_idx ON user_sessions USING HASH (session_token);
```

**หมายเหตุ PostgreSQL 10+:** Hash index ถูก WAL-logged แล้ว ปลอดภัยสำหรับ crash recovery (ก่อน PG10 ไม่ WAL-logged ดังนั้นไม่ปลอดภัย)

---

## GIN (Generalized Inverted Index)

### โครงสร้าง

GIN เก็บ **inverted index** — map จาก "element" → "documents ที่มี element นั้น"

```
ตัวอย่าง: Full-text search
"apple pie recipe" → ["apple", "pie", "recipe"]

GIN Index:
"apple"  → [doc1, doc5, doc8]
"pie"    → [doc1, doc2, doc8]
"recipe" → [doc1, doc3, doc8]

Query: MATCH "apple AND pie"
→ ดึง set[doc1, doc5, doc8] ∩ set[doc1, doc2, doc8] = {doc1, doc8}
```

### Best For

```sql
-- ========== JSONB ==========

CREATE TABLE products (
  id UUID PRIMARY KEY,
  name TEXT,
  attributes JSONB
);

-- GIN index บน JSONB
CREATE INDEX products_attributes_gin_idx ON products USING GIN (attributes);

-- หรือ index เฉพาะ top-level keys (compact)
CREATE INDEX products_attributes_gin_path_idx ON products 
  USING GIN (attributes jsonb_path_ops);

-- Operators ที่ใช้กับ GIN JSONB:
-- @>  contains
SELECT * FROM products WHERE attributes @> '{"color": "red"}';
SELECT * FROM products WHERE attributes @> '{"specs": {"ram": "8GB"}}';

-- ?  key exists
SELECT * FROM products WHERE attributes ? 'discount';

-- ?|  any key exists  
SELECT * FROM products WHERE attributes ?| ARRAY['sale', 'discount'];

-- ?& all keys exist
SELECT * FROM products WHERE attributes ?& ARRAY['color', 'size'];

-- ========== Arrays ==========
CREATE TABLE posts (
  id UUID PRIMARY KEY,
  title TEXT,
  tags TEXT[]
);

CREATE INDEX posts_tags_gin_idx ON posts USING GIN (tags);

-- @>  contains
SELECT * FROM posts WHERE tags @> ARRAY['typescript', 'database'];

-- <@  is contained by  
SELECT * FROM posts WHERE tags <@ ARRAY['typescript', 'javascript', 'react'];

-- &&  overlap (any element in common)
SELECT * FROM posts WHERE tags && ARRAY['typescript', 'python'];

-- ========== Full-Text Search ==========
-- เพิ่ม tsvector column
ALTER TABLE products ADD COLUMN search_vector tsvector;

-- สร้าง index
CREATE INDEX products_search_gin_idx ON products USING GIN (search_vector);

-- หรือ index บน expression โดยตรง
CREATE INDEX products_name_fts_idx ON products 
  USING GIN (to_tsvector('english', name));

-- Query
SELECT * FROM products
WHERE search_vector @@ to_tsquery('english', 'laptop & apple');

-- @@ operator
SELECT
  id, name,
  ts_rank(search_vector, query) as rank
FROM products, to_tsquery('english', 'laptop') query
WHERE search_vector @@ query
ORDER BY rank DESC;
```

### GIN vs gin_path_ops

```sql
-- Default GIN: รองรับ @>, ?, ?|, ?& operators
CREATE INDEX products_attrs_gin ON products USING GIN (attributes);

-- gin_path_ops: รองรับ @> เท่านั้น แต่ index เล็กกว่า และ query เร็วกว่า
CREATE INDEX products_attrs_pathops ON products 
  USING GIN (attributes jsonb_path_ops);

-- เลือกตามการใช้งาน:
-- ถ้าใช้แค่ @> (most common) → ใช้ gin_path_ops
-- ถ้าใช้ ?, ?|, ?& ด้วย → ใช้ default GIN
```

---

## GiST (Generalized Search Tree)

### โครงสร้าง

GiST เป็น framework สำหรับ custom index types — ใช้ tree structure แต่ operator ต่างกัน

### Best For

```sql
-- ========== Geometric Types ==========
CREATE TABLE locations (
  id SERIAL PRIMARY KEY,
  name TEXT,
  position POINT
);

CREATE INDEX locations_position_gist ON locations USING GIST (position);

-- Query ด้วย geometric operators
SELECT * FROM locations
WHERE position <-> POINT(100, 200) < 50;  -- ห่างจาก point ไม่เกิน 50 units

-- ========== Full-Text Search (tsvector) ==========
CREATE INDEX products_fts_gist ON products USING GIST (search_vector);
-- GiST vs GIN for FTS:
-- GiST: build เร็วกว่า, query ช้ากว่าเล็กน้อย
-- GIN: build ช้ากว่า, query เร็วกว่า (recommended สำหรับ static data)

-- ========== Range Types ==========
CREATE TABLE reservations (
  id UUID PRIMARY KEY,
  room_id UUID,
  booking_period tstzrange  -- timestamp range
);

CREATE INDEX reservations_period_gist ON reservations USING GIST (booking_period);

-- Query
SELECT * FROM reservations
WHERE booking_period @> NOW()::timestamptz;  -- overlaps current time

SELECT * FROM reservations
WHERE booking_period && '[2024-01-20, 2024-01-25]'::tstzrange;  -- overlaps range

-- ========== PostGIS (Geospatial) ==========
-- ต้องติดตั้ง PostGIS extension
CREATE EXTENSION IF NOT EXISTS postgis;

CREATE TABLE stores (
  id UUID PRIMARY KEY,
  name TEXT,
  location GEOGRAPHY(POINT, 4326)  -- longitude, latitude
);

CREATE INDEX stores_location_gist ON stores USING GIST (location);

-- ค้นหา stores ภายใน 5km จากจุดที่กำหนด
SELECT name, ST_Distance(location, 'SRID=4326;POINT(100.5018 13.7563)') as distance_meters
FROM stores
WHERE ST_DWithin(
  location,
  'SRID=4326;POINT(100.5018 13.7563)',  -- Bangkok
  5000  -- 5km in meters
)
ORDER BY distance_meters;

-- ========== IP Address Range (ip4r extension) ==========
CREATE EXTENSION IF NOT EXISTS ip4r;

CREATE TABLE ip_blacklist (
  id SERIAL PRIMARY KEY,
  ip_range ip4r,
  reason TEXT
);

CREATE INDEX ip_blacklist_range_gist ON ip_blacklist USING GIST (ip_range);

SELECT * FROM ip_blacklist
WHERE ip_range >>= '192.168.1.100'::ip4;  -- IP อยู่ใน range ไหม?
```

---

## BRIN (Block Range INdex)

### โครงสร้าง

BRIN เก็บ **min/max ของแต่ละ range ของ blocks** แทนที่จะ index ทุก value

```
Table Pages:  [1-10]     [11-20]    [21-30]    [31-40]
Values:      [1..1000]  [1001..2000] [2001..3000] [3001..4000]

BRIN Index:
Block Range 1-10:  min=1,     max=1000
Block Range 11-20: min=1001,  max=2000
Block Range 21-30: min=2001,  max=3000
Block Range 31-40: min=3001,  max=4000

Query: WHERE value = 2500
→ ข้าม blocks 1-20 (max < 2500), blocks 31-40 (min > 2500)
→ Scan แค่ blocks 21-30
```

### Best For

```sql
-- ========== Time-series data (append-only) ==========
CREATE TABLE sensor_readings (
  id BIGSERIAL PRIMARY KEY,
  sensor_id UUID,
  temperature DECIMAL(5,2),
  humidity DECIMAL(5,2),
  recorded_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- BRIN index บน recorded_at
-- pages_per_range = 128 (default) หมายถึง 1 BRIN entry ต่อ 128 pages
CREATE INDEX sensor_readings_time_brin ON sensor_readings
  USING BRIN (recorded_at)
  WITH (pages_per_range = 64);  -- ปรับ granularity

-- Query
EXPLAIN ANALYZE
SELECT * FROM sensor_readings
WHERE recorded_at BETWEEN '2024-01-01' AND '2024-01-31';

-- BRIN Index Scan on sensor_readings
-- Execution Time: 45ms (แทน 2400ms ด้วย Seq Scan)

-- ========== Log tables ==========
CREATE TABLE application_logs (
  id BIGSERIAL PRIMARY KEY,
  service_name VARCHAR(100),
  level VARCHAR(10),
  message TEXT,
  metadata JSONB,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX logs_created_at_brin ON application_logs USING BRIN (created_at);

-- ========== เปรียบเทียบขนาด Index ==========
-- Table: 10 million rows, 100GB
-- B-Tree index บน timestamp: ~2GB
-- BRIN index บน timestamp:   ~1MB  (2000x เล็กกว่า!)
```

### เมื่อไหร่ที่ BRIN ไม่ดี

```sql
-- ❌ BRIN แย่เมื่อข้อมูลไม่มี natural ordering
-- ตัวอย่าง: table ที่ UPDATE บ่อย ทำให้ค่า timestamp ไม่เรียงตาม physical order

-- ❌ BRIN ไม่ดีสำหรับ point queries (หา 1 row)
SELECT * FROM sensor_readings WHERE id = 12345;
-- ควรใช้ B-Tree บน id แทน

-- ❌ BRIN แย่เมื่อข้อมูล random order
-- ถ้า user_id กระจาย random ใน table BRIN จะไม่มีประสิทธิภาพ
```

---

## SP-GiST (Space-Partitioned GiST)

```sql
-- SP-GiST ใช้ non-balanced partition structures
-- เหมาะสำหรับ: trie, quadtree, k-d tree

-- Point data
CREATE TABLE wifi_hotspots (
  id UUID PRIMARY KEY,
  name TEXT,
  location POINT
);

CREATE INDEX wifi_location_spgist ON wifi_hotspots USING SPGIST (location);

-- IP range (inet type)
CREATE TABLE network_rules (
  id SERIAL PRIMARY KEY,
  network CIDR,
  action VARCHAR(10)
);

CREATE INDEX network_rules_spgist ON network_rules USING SPGIST (network);

-- Query
SELECT * FROM network_rules
WHERE '192.168.1.100'::inet << network;  -- IP อยู่ใน subnet
```

---

## Partial Indexes

Index เฉพาะ rows ที่ match condition

```sql
-- ✅ Index เฉพาะ active users (ส่วนใหญ่ query เฉพาะ active)
CREATE INDEX users_email_active_idx ON users(email)
  WHERE deleted_at IS NULL;

-- Query ที่ใช้ index นี้:
SELECT * FROM users
WHERE email = 'alice@example.com'
  AND deleted_at IS NULL;

-- ✅ Index เฉพาะ pending orders
CREATE INDEX orders_pending_idx ON orders(created_at, user_id)
  WHERE status = 'pending';

-- Query:
SELECT * FROM orders
WHERE status = 'pending'
  AND created_at > NOW() - INTERVAL '24 hours'
ORDER BY created_at ASC;

-- ✅ Unique index สำหรับ soft delete
-- ป้องกัน duplicate email เฉพาะ active users
CREATE UNIQUE INDEX users_email_unique_active 
  ON users(email)
  WHERE deleted_at IS NULL;

-- ✅ Index เฉพาะ featured products
CREATE INDEX products_featured_idx ON products(created_at DESC)
  WHERE featured = true AND status = 'active';

-- ข้อดี:
-- 1. Index เล็กกว่า → เร็วขึ้น
-- 2. ใช้ disk น้อยกว่า
-- 3. Update เร็วขึ้น (update rows ที่ไม่ใช่ active ไม่ต้อง update index)
```

---

## Expression Indexes

Index บน expression หรือ function ของ column

```sql
-- ✅ Case-insensitive email search
CREATE INDEX users_email_lower_idx ON users(LOWER(email));

-- Query ต้องใช้ expression เดียวกัน:
SELECT * FROM users WHERE LOWER(email) = LOWER('Alice@Example.com');
-- หรือ
SELECT * FROM users WHERE LOWER(email) = 'alice@example.com';

-- ✅ Extracted JSON value
CREATE INDEX orders_metadata_source_idx ON orders((metadata->>'source'));

SELECT * FROM orders WHERE metadata->>'source' = 'mobile_app';

-- ✅ Date truncation (report by month)
CREATE INDEX orders_month_idx ON orders(DATE_TRUNC('month', created_at));

SELECT COUNT(*), SUM(total_amount)
FROM orders
WHERE DATE_TRUNC('month', created_at) = '2024-01-01'::date;

-- ✅ Computed value
CREATE INDEX products_discounted_price_idx ON products(price * 0.9);

-- ✅ Array length
CREATE INDEX posts_tag_count_idx ON posts(array_length(tags, 1));

SELECT * FROM posts WHERE array_length(tags, 1) > 5;

-- ✅ JSONB path expression
CREATE INDEX orders_city_idx ON orders((shipping_address->>'city'));

SELECT * FROM orders WHERE shipping_address->>'city' = 'Bangkok';
```

---

## Covering Indexes (INCLUDE)

Include non-indexed columns เพื่อ enable Index Only Scan

```sql
-- ❌ ปกติ: Index Scan + Heap Fetch
CREATE INDEX products_category_idx ON products(category_id);

SELECT id, name, price
FROM products
WHERE category_id = 'cat1';
-- Index Scan: ดึง category_id จาก index → ไปดึง id, name, price จาก heap

-- ✅ Covering Index: Index Only Scan (ไม่ต้องไป heap!)
CREATE INDEX products_category_covering_idx ON products(category_id)
  INCLUDE (id, name, price);

SELECT id, name, price
FROM products
WHERE category_id = 'cat1';
-- Index Only Scan: ดึงทุกอย่างจาก index โดยตรง

EXPLAIN ANALYZE
SELECT id, name, price FROM products WHERE category_id = 'cat1';
-- Index Only Scan using products_category_covering_idx
-- (cost=0.43..25.72 rows=50 width=56)
-- Heap Fetches: 0  ← ไม่ต้องไป heap เลย!

-- ตัวอย่างเพิ่มเติม:
-- User list endpoint ที่ดึงแค่ id, name, email
CREATE INDEX users_active_covering_idx ON users(created_at DESC)
  INCLUDE (id, name, email)
  WHERE deleted_at IS NULL;

SELECT id, name, email
FROM users
WHERE deleted_at IS NULL
ORDER BY created_at DESC
LIMIT 50;
-- Index Only Scan ← ไม่ต้องแตะ table เลย!

-- Order summary ที่ดึงแค่ totals
CREATE INDEX orders_user_status_covering ON orders(user_id, status)
  INCLUDE (id, order_number, total_amount, created_at);

SELECT id, order_number, total_amount, created_at
FROM orders
WHERE user_id = 'user-123' AND status = 'delivered';
```

---

## Index Maintenance

### Index Bloat

```sql
-- ตรวจสอบ index bloat
SELECT
  schemaname,
  tablename,
  indexname,
  pg_size_pretty(pg_relation_size(indexrelid)) as index_size,
  idx_scan as scans_since_reset,
  idx_tup_read as tuples_read,
  idx_tup_fetch as tuples_fetched
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC
LIMIT 20;

-- ดู bloat โดยละเอียด (ต้อง install pgstattuple)
CREATE EXTENSION IF NOT EXISTS pgstattuple;

SELECT * FROM pgstattuple('products_price_idx');
-- leaf_fragmentation: ถ้าสูงกว่า 30-40% ควร REINDEX

-- REINDEX โดยไม่ lock table (PostgreSQL 12+)
REINDEX INDEX CONCURRENTLY products_price_idx;

-- REINDEX ทั้ง table
REINDEX TABLE CONCURRENTLY products;

-- REINDEX ทั้ง database (ใช้เวลานาน)
-- ต้องรันจาก command line:
-- reindexdb --concurrently myapp_db
```

### ตรวจสอบ Unused Indexes

```sql
-- Indexes ที่ไม่ได้ใช้ → ควรลบออก!
SELECT
  schemaname || '.' || tablename AS table,
  indexname,
  pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
  idx_scan AS scans
FROM pg_stat_user_indexes
WHERE idx_scan = 0  -- ไม่เคยใช้เลย
  AND indexname NOT LIKE '%_pkey'  -- ไม่ใช่ primary key
  AND schemaname = 'public'
ORDER BY pg_relation_size(indexrelid) DESC;

-- ลบ unused index
DROP INDEX CONCURRENTLY products_old_unused_idx;
```

### Duplicate Indexes

```sql
-- หา indexes ที่ซ้ำกัน
SELECT
  a.indrelid::regclass AS table,
  a.indexrelid::regclass AS index1,
  b.indexrelid::regclass AS index2,
  a.indkey AS index1_keys,
  b.indkey AS index2_keys
FROM pg_index a
JOIN pg_index b ON a.indrelid = b.indrelid
  AND a.indexrelid < b.indexrelid
  AND a.indkey = b.indkey
  AND a.indpred IS NULL
  AND b.indpred IS NULL;
```

---

## เมื่อไหร่ที่ไม่ควร Index

```sql
-- ❌ Column ที่มีค่าน้อยมาก (low cardinality)
-- เช่น: status มีค่า 'active', 'inactive' เท่านั้น
-- SELECT * FROM products WHERE status = 'active';
-- ถ้า 80% เป็น active → Seq Scan เร็วกว่า
-- แต่ถ้า 1% เป็น inactive → index ช่วยได้ (หรือ partial index)

-- ❌ Table เล็กมาก (< 1000 rows)
-- PostgreSQL จะ Seq Scan อยู่ดี เพราะเร็วกว่า

-- ❌ Columns ที่ UPDATE บ่อยมาก
-- ทุก UPDATE ต้อง update index ด้วย → write penalty

-- ❌ ใน bulk insert jobs
-- ลบ index ก่อน bulk insert แล้วสร้างใหม่ทีหลัง
DROP INDEX products_price_idx;

-- Bulk insert...
COPY products FROM '/data/products.csv' CSV;

-- สร้าง index ใหม่
CREATE INDEX CONCURRENTLY products_price_idx ON products(price);

-- ✅ Rule of thumb:
-- Index เมื่อ column ถูกใช้ใน WHERE, JOIN, ORDER BY บ่อย
-- และ query ดึงข้อมูลน้อยกว่า 5-10% ของ table
```

---

## Index Monitoring Queries

```sql
-- ========== Overall Index Health ==========

-- 1. Cache hit ratio per index
SELECT
  schemaname,
  tablename,
  indexname,
  idx_scan,
  idx_tup_read,
  idx_tup_fetch,
  pg_size_pretty(pg_relation_size(indexrelid)) as size
FROM pg_stat_user_indexes
WHERE schemaname = 'public'
ORDER BY idx_scan DESC;

-- 2. Table + Index sizes
SELECT
  tablename,
  pg_size_pretty(pg_table_size(tablename::text)) as table_size,
  pg_size_pretty(pg_indexes_size(tablename::text)) as index_size,
  pg_size_pretty(pg_total_relation_size(tablename::text)) as total_size,
  round(
    100.0 * pg_indexes_size(tablename::text) / 
    NULLIF(pg_total_relation_size(tablename::text), 0),
    2
  ) as index_ratio_pct
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(tablename::text) DESC;

-- 3. Missing indexes (sequential scans บน large tables)
SELECT
  schemaname,
  tablename,
  seq_scan,
  idx_scan,
  n_live_tup,
  seq_scan / GREATEST(idx_scan, 1) as seq_to_idx_ratio
FROM pg_stat_user_tables
WHERE n_live_tup > 10000
  AND seq_scan > idx_scan
ORDER BY seq_scan DESC;

-- 4. Index effectiveness
SELECT
  s.schemaname,
  s.tablename,
  s.indexname,
  s.idx_scan,
  s.idx_tup_read,
  CASE WHEN s.idx_scan > 0
    THEN round(s.idx_tup_read::numeric / s.idx_scan, 2)
    ELSE 0
  END as avg_tuples_per_scan,
  pg_size_pretty(pg_relation_size(s.indexrelid)) as index_size
FROM pg_stat_user_indexes s
JOIN pg_index i ON s.indexrelid = i.indexrelid
WHERE s.schemaname = 'public'
  AND NOT i.indisprimary
ORDER BY pg_relation_size(s.indexrelid) DESC;

-- 5. Indexes ที่ใช้ disk มากที่สุด
SELECT
  indexname,
  tablename,
  pg_size_pretty(pg_relation_size(indexrelid)) as size,
  idx_scan as scans
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC
LIMIT 20;

-- 6. ตรวจสอบ index validity (invalid indexes)
SELECT
  schemaname,
  tablename,
  indexname,
  pg_get_indexdef(indexrelid) as definition
FROM pg_stat_user_indexes
WHERE NOT (
  SELECT indisvalid
  FROM pg_index
  WHERE indexrelid = pg_stat_user_indexes.indexrelid
);
-- ถ้ามี invalid indexes → ต้อง REINDEX
```

---

## Full Examples with EXPLAIN Output

### Example 1: B-Tree สำหรับ e-commerce queries

```sql
-- Setup
CREATE TABLE products (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  sku VARCHAR(100) UNIQUE NOT NULL,
  name VARCHAR(500) NOT NULL,
  price NUMERIC(12,2) NOT NULL,
  category_id UUID NOT NULL,
  status VARCHAR(20) NOT NULL DEFAULT 'active',
  featured BOOLEAN DEFAULT false,
  stock_quantity INTEGER DEFAULT 0,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- สร้าง indexes ที่จำเป็น
CREATE INDEX products_category_price_idx ON products(category_id, price)
  WHERE status = 'active';

CREATE INDEX products_featured_created_idx ON products(featured, created_at DESC)
  WHERE status = 'active' AND featured = true;

CREATE INDEX products_price_range_idx ON products(price)
  WHERE status = 'active';

-- ========== Query 1: Products by category with price filter ==========
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, name, price, created_at
FROM products
WHERE category_id = 'cat-123'::uuid
  AND price BETWEEN 100 AND 500
  AND status = 'active'
ORDER BY price ASC
LIMIT 20;

-- Expected Plan:
-- Index Scan using products_category_price_idx on products
--   (cost=0.43..52.12 rows=15 width=64)
--   (actual time=0.089..0.312 rows=15 loops=1)
--   Index Cond: ((category_id = 'cat-123') AND (price >= 100) AND (price <= 500))
-- Buffers: shared hit=8
-- Planning Time: 0.234 ms
-- Execution Time: 0.356 ms ✓

-- ========== Query 2: Featured products ==========
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, name, price, created_at
FROM products
WHERE featured = true
  AND status = 'active'
ORDER BY created_at DESC
LIMIT 8;

-- Expected Plan:
-- Index Scan using products_featured_created_idx on products
--   (cost=0.43..8.52 rows=8 width=64)
--   Index Cond: (featured = true)
-- Buffers: shared hit=3
-- Planning Time: 0.156 ms
-- Execution Time: 0.089 ms ✓
```

### Example 2: GIN สำหรับ JSONB และ Full-Text

```sql
-- Setup
CREATE TABLE job_listings (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  title VARCHAR(300) NOT NULL,
  description TEXT,
  skills TEXT[],
  requirements JSONB NOT NULL DEFAULT '{}',
  salary_range NUMRANGE,
  search_vector TSVECTOR,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Trigger อัพเดต search_vector
CREATE OR REPLACE FUNCTION update_job_search_vector()
RETURNS TRIGGER AS $$
BEGIN
  NEW.search_vector = to_tsvector('english',
    NEW.title || ' ' || COALESCE(NEW.description, '')
  );
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER job_search_vector_trigger
  BEFORE INSERT OR UPDATE ON job_listings
  FOR EACH ROW EXECUTE FUNCTION update_job_search_vector();

-- Indexes
CREATE INDEX job_skills_gin_idx ON job_listings USING GIN (skills);
CREATE INDEX job_requirements_gin_idx ON job_listings USING GIN (requirements);
CREATE INDEX job_search_gin_idx ON job_listings USING GIN (search_vector);

-- ========== Query: Skills search ==========
EXPLAIN ANALYZE
SELECT id, title, skills
FROM job_listings
WHERE skills @> ARRAY['typescript', 'postgresql'];

-- Bitmap Heap Scan on job_listings
--   Recheck Cond: (skills @> '{typescript,postgresql}'::text[])
--   -> Bitmap Index Scan on job_skills_gin_idx
--        Index Cond: (skills @> '{typescript,postgresql}'::text[])
-- Planning Time: 0.234 ms
-- Execution Time: 1.456 ms (บน 1M rows)

-- ========== Query: JSONB requirements ==========
EXPLAIN ANALYZE
SELECT id, title, requirements
FROM job_listings
WHERE requirements @> '{"remote": true, "experience_years": 3}';

-- Bitmap Heap Scan on job_listings
--   Recheck Cond: (requirements @> '{"remote": true, "experience_years": 3}')
--   -> Bitmap Index Scan on job_requirements_gin_idx
-- Execution Time: 2.1ms

-- ========== Query: Full-Text Search ==========
EXPLAIN ANALYZE
SELECT
  id, title,
  ts_rank(search_vector, query) as rank
FROM job_listings, to_tsquery('english', 'senior & developer & typescript') query
WHERE search_vector @@ query
ORDER BY rank DESC
LIMIT 10;

-- Bitmap Heap Scan on job_listings
--   Recheck Cond: (search_vector @@ query)
--   -> Bitmap Index Scan on job_search_gin_idx
--        Index Cond: (search_vector @@ query)
-- Sort by rank
-- Execution Time: 5.2ms (บน 500K rows)
```

### Example 3: BRIN สำหรับ Time-Series

```sql
-- Setup: IoT sensor data
CREATE TABLE sensor_data (
  id BIGSERIAL,
  device_id UUID NOT NULL,
  metric_name VARCHAR(50) NOT NULL,
  value DECIMAL(10,4) NOT NULL,
  recorded_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (recorded_at);

-- สร้าง partitions
CREATE TABLE sensor_data_2024_01 PARTITION OF sensor_data
  FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE sensor_data_2024_02 PARTITION OF sensor_data
  FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- BRIN index บนแต่ละ partition
CREATE INDEX sensor_data_2024_01_brin ON sensor_data_2024_01
  USING BRIN (recorded_at)
  WITH (pages_per_range = 32);

CREATE INDEX sensor_data_2024_02_brin ON sensor_data_2024_02
  USING BRIN (recorded_at)
  WITH (pages_per_range = 32);

-- B-Tree index สำหรับ device_id queries
CREATE INDEX sensor_data_device_idx ON sensor_data(device_id, recorded_at DESC);

-- ========== Query: Recent sensor readings ==========
EXPLAIN (ANALYZE, BUFFERS)
SELECT device_id, metric_name, value, recorded_at
FROM sensor_data
WHERE recorded_at BETWEEN '2024-01-15 00:00:00' AND '2024-01-15 23:59:59'
  AND device_id = 'device-123'::uuid
ORDER BY recorded_at DESC;

-- Bitmap Heap Scan on sensor_data_2024_01
--   Recheck Cond: (recorded_at >= '2024-01-15' AND recorded_at <= '2024-01-16')
--   Rows Removed by Index Recheck: 124
--   -> Bitmap Index Scan on sensor_data_2024_01_brin
--        Index Cond: (recorded_at >= ... AND recorded_at <= ...)
-- Execution Time: 45ms (บน 100M rows, แทน 8500ms ด้วย Seq Scan)
```

### Example 4: Partial Index สำหรับ Active Records

```sql
-- ✅ Partial index ทำให้ index เล็กและเร็วขึ้นมาก

-- Index เฉพาะ active users ที่ไม่ถูก soft-delete
CREATE INDEX users_email_active_idx ON users(email)
  WHERE deleted_at IS NULL;

-- ขนาด:
-- Full index: 850MB (1M users ทั้งหมด)
-- Partial index: 80MB (100K active users)
-- เร็วขึ้น: index เล็กกว่า → fit ใน memory ดีกว่า

-- Index เฉพาะ orders ที่รอดำเนินการ
CREATE INDEX orders_pending_processing_idx ON orders(created_at, user_id)
  WHERE status IN ('pending', 'confirmed', 'processing');

-- Query ที่ใช้ index นี้
EXPLAIN ANALYZE
SELECT id, order_number, user_id, created_at
FROM orders
WHERE status IN ('pending', 'confirmed', 'processing')
  AND created_at < NOW() - INTERVAL '1 hour'  -- orders ที่รอนานเกิน 1 ชั่วโมง
ORDER BY created_at ASC
LIMIT 100;

-- Index Scan using orders_pending_processing_idx
-- (cost=0.43..15.22 rows=100 width=56)
-- Execution Time: 0.456ms ✓
```

---

## สรุปเปรียบเทียบ Index Types

| Index Type | Use Case | Operators | Size | Build Speed |
|-----------|----------|-----------|------|-------------|
| B-Tree | General purpose | =, <, >, BETWEEN, LIKE prefix | ปานกลาง | ปานกลาง |
| Hash | Equality only | = | เล็ก | เร็ว |
| GIN | Arrays, JSONB, FTS | @>, <@, &&, @@ | ใหญ่ | ช้า |
| GiST | Geometric, Range, FTS | <<, &&, @>, <-> | ปานกลาง | ปานกลาง |
| BRIN | Time-series (natural order) | =, <, > | เล็กมาก | เร็วมาก |
| SP-GiST | Points, IP, Tries | <<, >>, ~ | ปานกลาง | ปานกลาง |

### Quick Decision Guide

```
ต้องการ index บน column อะไร?
│
├── Timestamp/Sequential append-only?
│   └── ใช้ BRIN
│
├── Geometric/Location data?
│   └── ใช้ GiST (หรือ SP-GiST สำหรับ points)
│
├── JSONB/Array/Full-Text Search?
│   └── ใช้ GIN
│
├── Equality only บน high-cardinality column?
│   └── พิจารณา Hash (แต่ B-Tree มักเพียงพอ)
│
└── อื่นๆ ทั้งหมด
    └── ใช้ B-Tree (default)
        ├── มี WHERE condition? → ใช้ Partial Index
        ├── มี expression? → ใช้ Expression Index
        └── ต้องการ Index Only Scan? → ใช้ Covering Index (INCLUDE)
```

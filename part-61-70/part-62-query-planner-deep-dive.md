# Part 62: Query Planner Deep Dive

## บทนำ: PostgreSQL Query Planner คืออะไร

Query Planner (หรือ Query Optimizer) คือส่วนที่สำคัญที่สุดของ PostgreSQL ที่ทำหน้าที่เลือก "plan" ที่ดีที่สุดสำหรับการ execute query PostgreSQL ใช้ **Cost-Based Optimizer (CBO)** ซึ่งประเมินต้นทุน (cost) ของแต่ละ plan และเลือก plan ที่มี cost ต่ำที่สุด

```
SQL Query
    ↓
Parser (ตรวจ syntax)
    ↓
Rewriter (แปลง views, rules)
    ↓
Planner/Optimizer ← Statistics (pg_statistic)
    ↓                ← Cost parameters (seq_page_cost, etc.)
Executor
    ↓
Results
```

---

## Statistics: ฐานข้อมูลสถิติของ PostgreSQL

### pg_statistic และ pg_stats

PostgreSQL เก็บสถิติเกี่ยวกับข้อมูลในตารางเพื่อช่วย planner ประเมิน cost:

```sql
-- ดู statistics สำหรับ column
SELECT 
    attname,
    n_distinct,
    most_common_vals,
    most_common_freqs,
    histogram_bounds,
    correlation
FROM pg_stats
WHERE tablename = 'orders'
ORDER BY attname;

-- ดู raw statistics (ต้องการ superuser)
SELECT 
    staattnum,
    stainherit,
    stanullfrac,
    stawidth,
    stadistinct,
    stakind1, stavalues1,
    stakind2, stavalues2,
    stakind3, stavalues3
FROM pg_statistic
WHERE starelid = 'orders'::regclass;
```

### ความหมายของ Statistics Fields

```sql
-- n_distinct:
-- > 0: จำนวน unique values จริง
-- < 0: สัดส่วนของ unique values (-1 = ทุก row unique)
-- = 0: ไม่มีข้อมูล

-- correlation:
-- +1.0 = เรียงจากน้อยไปมากสมบูรณ์
-- -1.0 = เรียงจากมากไปน้อยสมบูรณ์
-- 0.0 = random

-- ตัวอย่าง: ดู statistics ของ table orders
SELECT 
    attname as column_name,
    n_distinct,
    ROUND(correlation::numeric, 4) as correlation,
    null_frac,
    avg_width as avg_bytes,
    array_length(most_common_vals::text::text[], 1) as num_mcv,
    array_length(histogram_bounds::text::text[], 1) as num_histogram_buckets
FROM pg_stats
WHERE tablename = 'orders'
    AND schemaname = 'public';
```

### ANALYZE: การอัปเดต Statistics

```sql
-- ANALYZE ทั้ง database
ANALYZE;

-- ANALYZE ตาราง specific
ANALYZE orders;

-- ANALYZE column specific
ANALYZE orders (customer_id, order_date, status);

-- ANALYZE แบบ verbose (ดู output)
ANALYZE VERBOSE orders;

-- ดูว่า ANALYZE ครั้งล่าสุดเมื่อไหร่
SELECT 
    schemaname,
    tablename,
    last_autoanalyze,
    last_analyze,
    n_live_tup,
    n_dead_tup,
    n_mod_since_analyze
FROM pg_stat_user_tables
WHERE tablename = 'orders';
```

### Auto-Analyze: Automatic Statistics Update

```sql
-- ดู autovacuum settings สำหรับ statistics
SHOW autovacuum_analyze_threshold;     -- default: 50
SHOW autovacuum_analyze_scale_factor;  -- default: 0.2

-- คำนวณ: trigger เมื่อ n_mod_since_analyze > threshold + scale_factor * reltuples
-- ตัวอย่าง: table มี 100,000 rows
-- threshold = 50 + 0.2 * 100000 = 20050 modifications

-- ปรับ per-table autovacuum
ALTER TABLE orders SET (
    autovacuum_analyze_threshold = 1000,
    autovacuum_analyze_scale_factor = 0.01
);
```

### default_statistics_target: ความละเอียดของ Statistics

```sql
-- ดูค่าปัจจุบัน
SHOW default_statistics_target;  -- default: 100

-- เพิ่มความละเอียดของ statistics ทั้ง database
SET default_statistics_target = 200;
ALTER SYSTEM SET default_statistics_target = 200;

-- เพิ่มความละเอียดสำหรับ column ที่สำคัญ
-- 1000 = maximum, เก็บ 1000 MCV และ histogram buckets
ALTER TABLE orders ALTER COLUMN customer_id SET STATISTICS 1000;
ALTER TABLE orders ALTER COLUMN order_date SET STATISTICS 500;
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 200;

-- หลังจากเปลี่ยน STATISTICS ต้อง ANALYZE ใหม่
ANALYZE orders;

-- ดู statistics target ต่อ column
SELECT 
    attname,
    attstattarget
FROM pg_attribute
WHERE attrelid = 'orders'::regclass
    AND attnum > 0
ORDER BY attnum;
```

---

## Cost Model: วิธีที่ PostgreSQL คำนวณต้นทุน

### Cost Parameters

```sql
-- ดู cost parameters ปัจจุบัน
SHOW seq_page_cost;          -- 1.0 (baseline)
SHOW random_page_cost;       -- 4.0 (default, for HDDs)
SHOW cpu_tuple_cost;         -- 0.01
SHOW cpu_index_tuple_cost;   -- 0.005
SHOW cpu_operator_cost;      -- 0.0025
SHOW parallel_setup_cost;    -- 1000.0
SHOW parallel_tuple_cost;    -- 0.1
SHOW effective_cache_size;   -- 4GB (estimate of OS cache)
```

### ปรับ Cost Parameters สำหรับ SSD

```sql
-- สำหรับ SSD ที่ random read เร็วกว่า HDD มาก
ALTER SYSTEM SET random_page_cost = 1.1;   -- เกือบเท่า sequential
ALTER SYSTEM SET seq_page_cost = 1.0;

-- สำหรับ NVMe SSD ที่เร็วมาก
ALTER SYSTEM SET random_page_cost = 1.0;

-- โหลดค่าใหม่
SELECT pg_reload_conf();

-- ดูความแตกต่างของ plan
-- ก่อนปรับ (random_page_cost=4.0): อาจใช้ Seq Scan
-- หลังปรับ (random_page_cost=1.1): อาจใช้ Index Scan
```

### การคำนวณ Cost ตัวอย่าง

```
Sequential Scan cost:
cost = seq_page_cost * pages + cpu_tuple_cost * rows
     = 1.0 * 1000 + 0.01 * 100000
     = 1000 + 1000 = 2000

Index Scan cost:
cost = random_page_cost * index_pages + cpu_index_tuple_cost * index_tuples
     + random_page_cost * heap_pages + cpu_tuple_cost * qualifying_rows
     = 4.0 * 10 + 0.005 * 1000 + 4.0 * 100 + 0.01 * 100
     = 40 + 5 + 400 + 1 = 446
     
→ PostgreSQL เลือก Index Scan (cost ต่ำกว่า)
```

---

## Plan Types

### Sequential Scan

```sql
-- ตัวอย่าง: table scan ทั้งหมด
EXPLAIN (ANALYZE, BUFFERS) 
SELECT * FROM orders WHERE status = 'pending';

-- Output:
-- Seq Scan on orders  (cost=0.00..3421.00 rows=5000 width=120)
--                     (actual time=0.042..45.23 rows=4987 loops=1)
--   Filter: ((status)::text = 'pending'::text)
--   Rows Removed by Filter: 95013
-- Buffers: shared hit=1921

-- เมื่อใช้ Sequential Scan:
-- 1. ไม่มี index
-- 2. เลือก rows จำนวนมาก (>10-20% ของ table)
-- 3. Table ขนาดเล็ก (ใน shared_buffers)
```

### Index Scan

```sql
-- สร้าง index
CREATE INDEX idx_orders_customer ON orders(customer_id);

-- ตรวจสอบ plan
EXPLAIN (ANALYZE, BUFFERS) 
SELECT * FROM orders WHERE customer_id = 12345;

-- Output:
-- Index Scan using idx_orders_customer on orders
--   (cost=0.43..450.25 rows=50 width=120)
--   (actual time=0.042..1.23 rows=48 loops=1)
--   Index Cond: (customer_id = 12345)
-- Buffers: shared hit=52

-- เมื่อใช้ Index Scan:
-- 1. เลือก rows น้อย (highly selective)
-- 2. random_page_cost ต่ำ (SSD)
-- 3. ต้องการ heap data ทุก row
```

### Index Only Scan

```sql
-- สร้าง covering index
CREATE INDEX idx_orders_covering ON orders(customer_id, order_date, status);

-- Query ที่ใช้เฉพาะ index columns
EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, order_date, status 
FROM orders 
WHERE customer_id = 12345;

-- Output:
-- Index Only Scan using idx_orders_covering on orders
--   (cost=0.43..45.25 rows=50 width=20)
--   (actual time=0.042..0.523 rows=48 loops=1)
--   Index Cond: (customer_id = 12345)
--   Heap Fetches: 0   ← ไม่ต้องอ่าน heap!
-- Buffers: shared hit=5

-- เร็วมากเพราะไม่ต้องอ่าน heap pages
-- Heap Fetches: 0 = สมบูรณ์แบบ (ขึ้นอยู่กับ Visibility Map)
```

### Bitmap Heap Scan + Bitmap Index Scan

```sql
-- เมื่อต้องดึง rows จำนวนปานกลาง
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders 
WHERE order_date BETWEEN '2024-01-01' AND '2024-03-31';

-- Output:
-- Bitmap Heap Scan on orders  (cost=234.56..2345.67 rows=12000 width=120)
--   (actual time=5.234..34.567 rows=11987 loops=1)
--   Recheck Cond: ((order_date >= '2024-01-01') AND (order_date <= '2024-03-31'))
--   Heap Blocks: exact=1234
--   Buffers: shared hit=1456 read=234
--   ->  Bitmap Index Scan on idx_orders_date
--         (cost=0.00..231.56 rows=12000 width=0)
--         (actual time=4.567..4.567 rows=11987 loops=1)
--         Index Cond: ((order_date >= '2024-01-01') AND (order_date <= '2024-03-31'))

-- วิธีทำงาน:
-- 1. Bitmap Index Scan: สร้าง bitmap ของ page numbers
-- 2. Bitmap Heap Scan: อ่าน pages ตาม bitmap (เรียงตาม physical order)
-- ดีกว่า Index Scan เมื่อ rows มากกว่า แต่ยังน้อยกว่า Seq Scan

-- BitmapAnd: รวม 2 bitmap indexes
EXPLAIN 
SELECT * FROM orders 
WHERE customer_id = 12345 AND status = 'pending';

-- อาจได้:
-- BitmapAnd
--   -> Bitmap Index Scan on idx_orders_customer
--   -> Bitmap Index Scan on idx_orders_status
```

### TID Scan

```sql
-- ใช้ physical location (ctid) โดยตรง
EXPLAIN SELECT * FROM orders WHERE ctid = '(100, 5)';

-- Output:
-- Tid Scan on orders (cost=0.00..4.01 rows=1 width=120)
--   TID Cond: (ctid = '(100,5)'::tid)

-- ใช้ในกรณีพิเศษ เช่น VACUUM หรือ CLUSTER
```

---

## Join Strategies

### Nested Loop Join

```sql
-- เหมาะสำหรับ: outer table เล็ก, inner table มี index
EXPLAIN (ANALYZE, BUFFERS)
SELECT o.*, c.name 
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.status = 'pending'
LIMIT 100;

-- Output:
-- Nested Loop  (cost=0.43..1234.56 rows=100 width=200)
--   ->  Limit (...)
--         ->  Seq Scan on orders o
--               Filter: (status = 'pending')
--   ->  Index Scan using customers_pkey on customers c
--         Index Cond: (id = o.customer_id)

-- Algorithm:
-- FOR each row in outer (orders):
--     FOR each matching row in inner (customers via index):
--         output combined row

-- Cost: O(outer × inner) แต่ index ทำให้ inner เป็น O(log n)
```

### Hash Join

```sql
-- เหมาะสำหรับ: large tables, ไม่มี index ที่เหมาะสม
EXPLAIN (ANALYZE, BUFFERS)
SELECT o.*, c.name 
FROM orders o
JOIN customers c ON o.customer_id = c.id;

-- Output:
-- Hash Join  (cost=5678.00..45678.00 rows=100000 width=200)
--             (actual time=34.567..234.567 rows=100000 loops=1)
--   Hash Cond: (o.customer_id = c.id)
--   Buffers: shared hit=4567 read=1234
--   ->  Seq Scan on orders o (cost=0.00..25000.00 rows=1000000)
--   ->  Hash  (cost=2345.00..2345.00 rows=100000 width=80)
--             (actual time=34.567..34.567 rows=100000 loops=1)
--         Buckets: 131072  Batches: 1  Memory Usage: 12345kB
--         Buffers: shared hit=1234
--         ->  Seq Scan on customers c

-- Algorithm:
-- 1. Build hash table จาก smaller table (customers)
-- 2. Probe hash table สำหรับแต่ละ row จาก larger table (orders)

-- ปัญหา: hash table ใหญ่กว่า work_mem → Batches > 1 (spills to disk)
-- แก้ไข: เพิ่ม work_mem
SET work_mem = '256MB';
```

### Merge Join

```sql
-- เหมาะสำหรับ: sorted inputs (มี index หรือ sort แล้ว)
EXPLAIN (ANALYZE, BUFFERS)
SELECT o.*, c.name 
FROM orders o
JOIN customers c ON o.customer_id = c.id
ORDER BY o.customer_id;

-- Output:
-- Merge Join  (cost=12345.00..45678.00 rows=100000 width=200)
--   Merge Cond: (c.id = o.customer_id)
--   ->  Index Scan using customers_pkey on customers c
--   ->  Sort
--         Sort Key: o.customer_id
--         ->  Seq Scan on orders o

-- Algorithm:
-- 1. Sort (หรือใช้ sorted index) ทั้ง 2 inputs
-- 2. Merge แบบ two-pointer technique

-- ดีที่สุดเมื่อ inputs sorted แล้วจาก indexes
```

### Join Ordering: join_collapse_limit

```sql
-- จำนวน tables สูงสุดที่ planner จะลอง permutations ทั้งหมด
SHOW join_collapse_limit;  -- default: 8

-- เมื่อ tables > 8: ใช้ genetic algorithm (GEQO)
SHOW geqo_threshold;  -- default: 12

-- Force specific join order (ปิด reordering)
SET join_collapse_limit = 1;

-- ตัวอย่าง: force order ที่เราต้องการ
SET join_collapse_limit = 1;
EXPLAIN
SELECT *
FROM t1
JOIN t2 ON t1.id = t2.t1_id    -- t1 JOIN t2 ก่อน
JOIN t3 ON t2.id = t3.t2_id;   -- แล้ว JOIN t3
```

---

## Aggregate Methods

```sql
-- HashAggregate: ใช้ hash table
EXPLAIN 
SELECT status, COUNT(*) 
FROM orders 
GROUP BY status;

-- Output:
-- HashAggregate  (cost=2345.00..2345.05 rows=5 width=40)
--   Group Key: status
--   ->  Seq Scan on orders

-- GroupAggregate: ใช้ sorted input
EXPLAIN
SELECT customer_id, COUNT(*)
FROM orders
GROUP BY customer_id
ORDER BY customer_id;

-- Output (เมื่อมี index):
-- GroupAggregate  (cost=...)
--   Group Key: customer_id
--   ->  Index Only Scan using idx_orders_customer

-- เพิ่ม hash_mem_multiplier สำหรับ large aggregations
SET hash_mem_multiplier = 2.0;
```

---

## enable_* Parameters: บังคับหรือปิด Plan Types

```sql
-- ดู enable parameters ทั้งหมด
SELECT name, setting 
FROM pg_settings 
WHERE name LIKE 'enable_%';

-- ปิด Sequential Scan (บังคับ index)
SET enable_seqscan = OFF;

-- ปิด Index Scan
SET enable_indexscan = OFF;

-- ปิด Index Only Scan
SET enable_indexonlyscan = OFF;

-- ปิด Bitmap Scan
SET enable_bitmapscan = OFF;

-- ปิด Hash Join
SET enable_hashjoin = OFF;

-- ปิด Merge Join
SET enable_mergejoin = OFF;

-- ปิด Nested Loop
SET enable_nestloop = OFF;

-- ปิด Hash Aggregate
SET enable_hashagg = OFF;

-- เปิด Parallel Query
SET enable_parallel_hash = ON;
SET max_parallel_workers_per_gather = 4;

-- ตัวอย่าง: debug ว่า plan เปลี่ยนอย่างไร
SET enable_seqscan = OFF;
EXPLAIN SELECT * FROM orders WHERE status = 'pending';
-- ตอนนี้ต้องใช้ index หรือ bitmap scan
RESET enable_seqscan;
```

---

## EXPLAIN Output: อ่านและตีความ

### EXPLAIN พื้นฐาน

```sql
EXPLAIN SELECT * FROM orders WHERE customer_id = 12345;

-- Output:
--                             QUERY PLAN
-- -----------------------------------------------------------------------
-- Index Scan using idx_orders_customer on orders
--   (cost=0.43..56.78 rows=50 width=120)
--   Index Cond: (customer_id = 12345)

-- cost=0.43..56.78:
--   - 0.43 = startup cost (cost ก่อนได้ row แรก)
--   - 56.78 = total cost (cost เพื่อได้ทุก row)

-- rows=50: จำนวน rows ที่ planner ประเมิน
-- width=120: ขนาด average row ใน bytes
```

### EXPLAIN ANALYZE: ข้อมูลจริงจาก Execution

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT 
    c.name,
    COUNT(o.id) as order_count,
    SUM(o.total) as total_value
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
WHERE c.region = 'Bangkok'
GROUP BY c.id, c.name
ORDER BY total_value DESC
LIMIT 20;

-- Output ละเอียด:
-- Limit  (cost=12345.67..12345.72 rows=20 width=60)
--         (actual time=234.567..234.589 rows=20 loops=1)
--   ->  Sort  (cost=12345.67..12385.67 rows=16000 width=60)
--             (actual time=234.567..234.572 rows=20 loops=1)
--         Sort Key: (sum(o.total)) DESC
--         Sort Method: top-N heapsort  Memory: 27kB
--         ->  HashAggregate  (cost=11234.56..11394.56 rows=16000 width=60)
--                             (actual time=220.123..232.456 rows=15987 loops=1)
--               Group Key: c.id, c.name
--               Batches: 1  Memory Usage: 4097kB
--               ->  Hash Left Join  (cost=2345.00..10234.56 rows=200000 width=48)
--                                   (actual time=34.567..180.234 rows=195678 loops=1)
--                     Hash Cond: (o.customer_id = c.id)
--                     Buffers: shared hit=4567 read=1234
--                     ->  Seq Scan on orders o  (cost=0.00..6789.00 rows=200000)
--                                               (actual time=0.023..45.678 rows=200000 loops=1)
--                           Buffers: shared hit=2345 read=678
--                     ->  Hash  (cost=1234.00..1234.00 rows=16000 width=40)
--                               (actual time=12.345..12.345 rows=15987 loops=1)
--                           Buckets: 16384  Batches: 1  Memory Usage: 1025kB
--                           Buffers: shared hit=345 read=89
--                           ->  Seq Scan on customers c
--                                 Filter: (region = 'Bangkok')
--                                 Rows Removed by Filter: 84013
-- Planning Time: 2.345 ms
-- Execution Time: 234.567 ms

-- สิ่งที่ต้องดู:
-- 1. rows estimated vs actual: ถ้าต่างมาก = statistics ไม่ดี
-- 2. Buffers: read >> hit = cache miss มาก
-- 3. Sort Method: external sort = เกิน work_mem
-- 4. Batches > 1 = hash spill to disk
```

### Buffers อธิบาย

```
Buffers: shared hit=4567 read=1234 dirtied=123 written=45

shared hit: อ่านจาก shared_buffers (L2 cache ของ PostgreSQL)
read: อ่านจาก OS/disk (cache miss)
dirtied: pages ที่ถูก modify
written: pages ที่ถูกเขียนลง disk

hit/(hit+read) = buffer cache hit ratio
เป้าหมาย: > 99% สำหรับ OLTP
```

---

## Visualizing Plans: Tools ที่ใช้งาน

```sql
-- 1. EXPLAIN JSON format สำหรับ tools
EXPLAIN (ANALYZE, FORMAT JSON)
SELECT * FROM orders WHERE customer_id = 12345;

-- 2. ใช้ explain.tensor.ru (online)
-- Copy JSON output และ paste ลงเว็บ

-- 3. pgMustard (เน้น actionable insights)
-- https://www.pgmustard.com/

-- 4. PEV2 (Plan Explorer Version 2)
-- https://explain.dalibo.com/

-- Python script เพื่อส่ง EXPLAIN ไปยัง visualizer
```

```python
import psycopg2
import json
import webbrowser
import urllib.parse

def visualize_plan(query: str, conn_string: str):
    """ส่ง query plan ไปยัง explain.dalibo.com"""
    
    conn = psycopg2.connect(conn_string)
    cur = conn.cursor()
    
    # Get JSON plan
    cur.execute(f"EXPLAIN (ANALYZE, FORMAT JSON, BUFFERS) {query}")
    plan_json = cur.fetchone()[0]
    
    # Encode for URL
    plan_str = json.dumps(plan_json, indent=2)
    
    # Open in browser (PEV2)
    # หรือ save เป็น file และ upload manually
    with open('/tmp/plan.json', 'w') as f:
        json.dump(plan_json, f, indent=2)
    
    print("Plan saved to /tmp/plan.json")
    print("Upload to https://explain.dalibo.com/ for visualization")
    
    cur.close()
    conn.close()
    
    return plan_json

# การใช้งาน
plan = visualize_plan(
    "SELECT * FROM orders WHERE customer_id = 12345",
    "postgresql://localhost/mydb"
)
```

---

## Planner Hints ด้วย pg_hint_plan

```sql
-- ติดตั้ง extension
-- Ubuntu: sudo apt-get install postgresql-14-pg-hint-plan
CREATE EXTENSION pg_hint_plan;

-- การใช้งาน: hints อยู่ใน comment ด้านหน้า query
/*+ SeqScan(orders) */
SELECT * FROM orders WHERE status = 'pending';

/*+ IndexScan(orders idx_orders_status) */
SELECT * FROM orders WHERE status = 'pending';

/*+ BitmapScan(orders) */
SELECT * FROM orders WHERE status = 'pending';

-- Join hints
/*+ HashJoin(orders customers) */
SELECT * FROM orders o JOIN customers c ON o.customer_id = c.id;

/*+ NestLoop(orders customers) */
SELECT * FROM orders o JOIN customers c ON o.customer_id = c.id;

/*+ MergeJoin(orders customers) */
SELECT * FROM orders o JOIN customers c ON o.customer_id = c.id;

-- Join order hints
/*+ Leading(orders customers products) */
SELECT * 
FROM orders o
JOIN customers c ON o.customer_id = c.id
JOIN order_items oi ON o.id = oi.order_id
JOIN products p ON oi.product_id = p.id;

-- Combined hints
/*+ 
    NestLoop(o c)
    IndexScan(c customers_pkey)
    SeqScan(o)
*/
SELECT o.*, c.name
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.status = 'pending';

-- ดู hints ที่ใช้จริง
SET pg_hint_plan.debug_print = ON;
SET client_min_messages = LOG;
```

### เมื่อไหร่ควรใช้ Hints

```
ไม่ควรใช้ hints เป็นการแก้ปัญหาถาวร
ใช้เมื่อ:
1. Planner เลือก plan ผิดชัดเจน แต่ไม่มีเวลาแก้ statistics
2. Testing/debugging เพื่อเข้าใจ plan alternatives
3. Emergency hotfix ใน production

ควรแก้ปัญหาที่ต้นเหตุ:
1. อัปเดต statistics (ANALYZE)
2. สร้าง/ปรับ indexes
3. ปรับ cost parameters
4. สร้าง extended statistics
```

---

## Row Estimates: ทำไมมันผิด

### Histogram Boundaries

```sql
-- ดู histogram สำหรับ column
SELECT 
    histogram_bounds::text
FROM pg_stats
WHERE tablename = 'orders'
    AND attname = 'order_date';

-- Output: {2020-01-01, 2020-04-01, 2020-07-01, ..., 2024-01-01}
-- PostgreSQL แบ่งข้อมูลออกเป็น buckets สม่ำเสมอ

-- ปัญหา: ถ้าข้อมูลไม่กระจายสม่ำเสมอ
-- เช่น 90% ของ orders อยู่ใน 3 เดือนล่าสุด
-- Histogram จะประเมินผิดสำหรับ recent data
```

### MCV (Most Common Values)

```sql
-- ดู MCV สำหรับ column
SELECT 
    unnest(most_common_vals::text::text[]) as value,
    unnest(most_common_freqs::text::float[]) as frequency
FROM pg_stats
WHERE tablename = 'orders'
    AND attname = 'status';

-- Output:
-- value     | frequency
-- --------------------
-- completed | 0.65
-- pending   | 0.20
-- cancelled | 0.10
-- returned  | 0.05

-- PostgreSQL เก็บ MCV แยกจาก histogram
-- ถ้า value อยู่ใน MCV: ใช้ frequency โดยตรง
-- ถ้าไม่อยู่ใน MCV: ใช้ histogram estimate
```

### Correlation Statistics

```sql
-- Correlation ส่งผลต่อการเลือก Index Scan vs Seq Scan
SELECT attname, correlation
FROM pg_stats
WHERE tablename = 'orders'
ORDER BY ABS(correlation) DESC;

-- correlation = 1.0: ข้อมูลเรียงตาม physical order
-- ถ้า correlation สูง: Index Scan มีประสิทธิภาพ
-- ถ้า correlation ต่ำ: ต้องอ่านหลาย heap pages (random access)

-- หลังจาก CLUSTER: correlation จะดีขึ้น
CLUSTER orders USING idx_orders_date;
-- หลัง CLUSTER ต้อง ANALYZE ใหม่
ANALYZE orders;
```

---

## Extended Statistics: แก้ปัญหา Multi-Column Estimates

```sql
-- ปัญหา: PostgreSQL สมมติว่า columns เป็น independent
-- ตัวอย่าง: WHERE region = 'Bangkok' AND type = 'restaurant'
-- ถ้า region มี 10% Bangkok และ type มี 5% restaurant
-- PostgreSQL ประเมิน: 10% × 5% = 0.5%
-- แต่ความจริง restaurants ใน Bangkok อาจเป็น 8%

-- สร้าง Extended Statistics
CREATE STATISTICS orders_region_type_stat
    ON region, type
    FROM orders;

-- Types ของ Extended Statistics:
-- ndistinct: distinct combinations
-- dependencies: functional dependencies
-- mcv: most common value combinations

-- สร้างแบบระบุ types
CREATE STATISTICS orders_customer_status (ndistinct, dependencies)
    ON customer_id, status
    FROM orders;

-- หลังสร้าง ต้อง ANALYZE
ANALYZE orders;

-- ดู extended statistics
SELECT 
    stxname,
    stxkeys,
    stxkind,
    stxndistinct,
    stxdependencies
FROM pg_statistic_ext
WHERE stxrelid = 'orders'::regclass;

-- ดู MCV extended statistics
SELECT * FROM pg_statistic_ext_data
WHERE stxoid = 'orders_region_type_stat'::regclass;

-- ตัวอย่าง query ที่ได้ประโยชน์
EXPLAIN (ANALYZE)
SELECT * FROM orders 
WHERE region = 'Bangkok' AND type = 'restaurant';
-- Row estimate ควรแม่นยำขึ้นหลัง extended statistics
```

---

## Function Volatility: ผลต่อ Query Planning

```sql
-- IMMUTABLE: ผลลัพธ์เหมือนกันเสมอสำหรับ input เดิม
-- PostgreSQL สามารถ evaluate ณ planning time และ constant-fold
CREATE OR REPLACE FUNCTION add_vat(price NUMERIC) 
RETURNS NUMERIC AS $$
    SELECT price * 1.07
$$ LANGUAGE SQL IMMUTABLE STRICT;

-- STABLE: ผลลัพธ์เหมือนกันภายใน single query
-- ใช้ scan statistics ได้ แต่ evaluate per execution
CREATE OR REPLACE FUNCTION get_active_price(product_id INT)
RETURNS NUMERIC AS $$
    SELECT price FROM prices 
    WHERE product_id = $1 AND is_active = true
$$ LANGUAGE SQL STABLE;

-- VOLATILE: ผลลัพธ์อาจต่างทุกครั้ง (default)
-- ไม่ optimize ใดๆ ต้อง evaluate ทุก row
CREATE OR REPLACE FUNCTION random_discount()
RETURNS NUMERIC AS $$
    SELECT RANDOM() * 0.3
$$ LANGUAGE SQL VOLATILE;

-- ผลต่อ Planning:
-- IMMUTABLE ใน WHERE clause: planner สามารถ constant-fold
-- STABLE: planner ประเมิน cardinality ได้
-- VOLATILE: planner ประเมิน cardinality ไม่ได้ → อาจเลือก plan ผิด

-- ตัวอย่างผลกระทบ
EXPLAIN SELECT * FROM products WHERE price > add_vat(100);
-- IMMUTABLE: WHERE price > 107 (constant fold ณ plan time)

EXPLAIN SELECT * FROM products WHERE price > get_today_discount(product_id);
-- VOLATILE: ไม่รู้ selectivity → อาจเลือก Seq Scan เสมอ
```

---

## Parallel Query Plans

```sql
-- ดู parallel settings
SHOW max_parallel_workers;              -- 8 (สูงสุด total)
SHOW max_parallel_workers_per_gather;   -- 2 (default ต่อ query)
SHOW parallel_setup_cost;              -- 1000
SHOW parallel_tuple_cost;             -- 0.1
SHOW min_parallel_table_scan_size;    -- 8MB
SHOW min_parallel_index_scan_size;    -- 512kB

-- ตรวจสอบ parallel plan
EXPLAIN (ANALYZE)
SELECT SUM(total) FROM orders WHERE order_date > '2023-01-01';

-- Output ถ้า parallel:
-- Finalize Aggregate
--   ->  Gather
--         Workers Planned: 2
--         Workers Launched: 2
--         ->  Partial Aggregate
--               ->  Parallel Seq Scan on orders
--                     Filter: (order_date > '2023-01-01')
--                     Workers: 2

-- บังคับ parallel workers
SET max_parallel_workers_per_gather = 4;
EXPLAIN SELECT SUM(total) FROM orders;

-- ปิด parallel query
SET max_parallel_workers_per_gather = 0;

-- ปรับ per-table
ALTER TABLE orders SET (parallel_workers = 4);

-- เปิด parallel สำหรับ operations เพิ่มเติม
SET parallel_leader_participation = ON;
SET enable_parallel_hash = ON;
SET enable_parallel_append = ON;
```

### Parallel Query ตัวอย่างเต็ม

```sql
-- ตั้งค่าสำหรับ OLAP
ALTER SYSTEM SET max_parallel_workers = 8;
ALTER SYSTEM SET max_parallel_workers_per_gather = 4;
ALTER SYSTEM SET parallel_setup_cost = 100;
ALTER SYSTEM SET parallel_tuple_cost = 0.01;
SELECT pg_reload_conf();

-- ตัวอย่าง parallel join
EXPLAIN (ANALYZE)
SELECT c.region, SUM(o.total) as revenue
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.order_date >= '2024-01-01'
GROUP BY c.region;

-- จะเห็น:
-- Finalize GroupAggregate
--   ->  Gather Merge
--         Workers Planned: 4
--         ->  Partial GroupAggregate
--               ->  Parallel Hash Join
--                     ->  Parallel Seq Scan on orders
--                     ->  Parallel Hash
--                           ->  Parallel Seq Scan on customers
```

---

## Query Rewriting: Views และ Rules

```sql
-- View rewriting
CREATE VIEW recent_orders AS
SELECT * FROM orders WHERE order_date > CURRENT_DATE - INTERVAL '30 days';

-- Query ต่อ view
EXPLAIN SELECT * FROM recent_orders WHERE customer_id = 123;

-- Output: planner "inlines" view definition
-- Index Scan using idx_orders_customer_date on orders
--   Index Cond: (customer_id = 123) AND (order_date > (current_date - 30))

-- Materialized View: ไม่ inline, query ต่อ cached data
CREATE MATERIALIZED VIEW monthly_summary AS
SELECT 
    DATE_TRUNC('month', order_date) as month,
    customer_id,
    SUM(total) as total_spent
FROM orders
GROUP BY 1, 2;

CREATE UNIQUE INDEX ON monthly_summary(month, customer_id);

-- Refresh ข้อมูล
REFRESH MATERIALIZED VIEW CONCURRENTLY monthly_summary;

-- Query ต่อ Materialized View (fast)
EXPLAIN SELECT * FROM monthly_summary WHERE customer_id = 123;
-- Bitmap Heap Scan on monthly_summary
```

---

## Script ตรวจสอบ Query Performance

```sql
-- ดู top slow queries (ต้องการ pg_stat_statements)
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

SELECT 
    LEFT(query, 100) as query_snippet,
    calls,
    ROUND(mean_exec_time::numeric, 2) as avg_ms,
    ROUND(total_exec_time::numeric, 2) as total_ms,
    ROUND(stddev_exec_time::numeric, 2) as stddev_ms,
    rows,
    ROUND(100.0 * shared_blks_hit / NULLIF(shared_blks_hit + shared_blks_read, 0), 2) as hit_ratio
FROM pg_stat_statements
WHERE calls > 100
ORDER BY mean_exec_time DESC
LIMIT 20;

-- ดู queries ที่ใช้ sequential scan บน large tables
SELECT 
    schemaname,
    tablename,
    seq_scan,
    seq_tup_read,
    idx_scan,
    ROUND(100.0 * seq_tup_read / NULLIF(seq_tup_read + idx_tup_fetch, 0), 2) as seq_pct
FROM pg_stat_user_tables
WHERE seq_scan > 0
ORDER BY seq_tup_read DESC
LIMIT 20;

-- ดู missing indexes (tables ที่ใช้ seq scan มาก)
SELECT 
    schemaname,
    tablename,
    seq_scan as sequential_scans,
    n_live_tup as live_rows,
    seq_scan / NULLIF(n_live_tup, 0)::float as scans_per_row
FROM pg_stat_user_tables
WHERE seq_scan > 100
    AND n_live_tup > 10000
ORDER BY seq_scan DESC;
```

---

## Practical Tuning Workflow

```bash
#!/bin/bash
# Script สำหรับ analyze slow queries และ suggest indexes

# 1. Enable pg_stat_statements
psql -c "CREATE EXTENSION IF NOT EXISTS pg_stat_statements;"

# 2. Reset statistics (ทำก่อน profiling period)
psql -c "SELECT pg_stat_statements_reset();"

# 3. รอ 1 ชั่วโมง (หรือ period ที่ต้องการ)
echo "Waiting for profiling period..."
sleep 3600

# 4. Extract top queries
psql -c "
COPY (
    SELECT 
        query,
        calls,
        mean_exec_time,
        total_exec_time,
        rows
    FROM pg_stat_statements
    WHERE calls > 10
    ORDER BY total_exec_time DESC
    LIMIT 50
) TO '/tmp/slow_queries.csv' CSV HEADER;
"

echo "Slow queries saved to /tmp/slow_queries.csv"
```

```python
#!/usr/bin/env python3
# Python script วิเคราะห์ query plans อัตโนมัติ

import psycopg2
import json
import re
from typing import List, Dict, Any

def analyze_query_plan(conn_string: str, query: str) -> Dict[str, Any]:
    """วิเคราะห์ query plan และหา issues"""
    
    conn = psycopg2.connect(conn_string)
    cur = conn.cursor()
    
    # Get EXPLAIN JSON
    cur.execute(f"EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) {query}")
    plan = cur.fetchone()[0][0]
    
    issues = []
    suggestions = []
    
    def analyze_node(node: Dict, depth: int = 0):
        node_type = node.get('Node Type', '')
        
        # Check: row estimate accuracy
        actual_rows = node.get('Actual Rows', 0)
        plan_rows = node.get('Plan Rows', 0)
        
        if plan_rows > 0:
            estimate_ratio = actual_rows / plan_rows
            if estimate_ratio > 10 or estimate_ratio < 0.1:
                issues.append({
                    'type': 'bad_estimate',
                    'node': node_type,
                    'estimated': plan_rows,
                    'actual': actual_rows,
                    'ratio': estimate_ratio
                })
                suggestions.append(
                    f"Run ANALYZE on tables in '{node_type}' node. "
                    f"Row estimate off by {estimate_ratio:.1f}x"
                )
        
        # Check: Sequential Scan on large tables
        if node_type == 'Seq Scan':
            rows = node.get('Actual Rows', 0)
            if rows > 10000:
                issues.append({
                    'type': 'large_seq_scan',
                    'table': node.get('Relation Name', 'unknown'),
                    'rows': rows
                })
                suggestions.append(
                    f"Consider adding index on '{node.get('Relation Name')}' "
                    f"for filter: {node.get('Filter', 'unknown')}"
                )
        
        # Check: Hash batches > 1 (disk spill)
        if node_type in ['Hash', 'Hash Join']:
            batches = node.get('Hash Batches', 1)
            if batches > 1:
                issues.append({
                    'type': 'hash_spill',
                    'batches': batches,
                    'memory_used': node.get('Peak Memory Usage', 0)
                })
                suggestions.append(
                    f"Increase work_mem to avoid hash spill. "
                    f"Currently using {batches} batches."
                )
        
        # Check: Sort to disk
        if node_type == 'Sort':
            method = node.get('Sort Method', '')
            if 'external' in method.lower():
                issues.append({
                    'type': 'sort_spill',
                    'method': method
                })
                suggestions.append("Increase work_mem to avoid sort spill to disk")
        
        # Recurse into children
        for child in node.get('Plans', []):
            analyze_node(child, depth + 1)
    
    analyze_node(plan['Plan'])
    
    total_time = plan.get('Execution Time', 0)
    planning_time = plan.get('Planning Time', 0)
    
    result = {
        'query': query[:200],
        'execution_time_ms': total_time,
        'planning_time_ms': planning_time,
        'issues': issues,
        'suggestions': suggestions,
        'plan': plan
    }
    
    cur.close()
    conn.close()
    
    return result

# การใช้งาน
result = analyze_query_plan(
    "postgresql://localhost/mydb",
    "SELECT c.name, SUM(o.total) FROM customers c JOIN orders o ON c.id = o.customer_id GROUP BY c.name"
)

print(f"Execution time: {result['execution_time_ms']:.2f}ms")
print(f"\nIssues found ({len(result['issues'])}):")
for issue in result['issues']:
    print(f"  - {issue['type']}: {issue}")

print(f"\nSuggestions:")
for suggestion in result['suggestions']:
    print(f"  → {suggestion}")
```

---

## สรุป: Best Practices สำหรับ Query Planning

```
1. Statistics ต้องทันสมัยเสมอ
   - ใช้ autovacuum แต่ปรับ scale_factor สำหรับ large tables
   - Run ANALYZE หลัง bulk imports
   - เพิ่ม statistics target สำหรับ high-cardinality columns

2. Cost parameters ต้องตรงกับ hardware
   - SSD: random_page_cost = 1.1-2.0
   - HDD: random_page_cost = 4.0 (default)
   - ปรับ effective_cache_size ให้ตรงกับ OS cache จริง

3. Extended Statistics สำหรับ multi-column queries
   - สร้าง statistics สำหรับ correlated columns
   - ตรวจสอบ row estimate accuracy ด้วย EXPLAIN ANALYZE

4. Parallel Query สำหรับ OLAP
   - เพิ่ม max_parallel_workers_per_gather
   - ลด parallel_setup_cost สำหรับ workloads ที่เหมาะสม

5. ใช้ pg_stat_statements สำหรับ monitoring
   - ตรวจสอบ mean_exec_time และ total_exec_time
   - Reset statistics ระหว่าง testing periods
```

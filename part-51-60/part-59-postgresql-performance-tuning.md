# Part 59: Performance Tuning PostgreSQL

## บทนำ: ทำไม PostgreSQL ถึงช้า?

PostgreSQL ออกมาพร้อม default configuration ที่ conservative มาก เหมาะสำหรับระบบที่มี RAM น้อย เพื่อให้ทำงานได้บน hardware ทุกประเภท แต่ใน production environment ที่มี RAM มาก เราต้องปรับค่าหลายอย่างเพื่อให้ได้ประสิทธิภาพสูงสุด

```
Default PostgreSQL:
- shared_buffers = 128MB  ← น้อยมากสำหรับ server 32GB RAM
- work_mem = 4MB          ← Sort/Hash จะใช้ disk บ่อย
- max_connections = 100   ← อาจไม่พอ

Tuned PostgreSQL (32GB RAM):
- shared_buffers = 8GB    ← 25% of RAM
- work_mem = 64MB         ← Sort ใช้ RAM
- max_connections = 200   ← ใช้ PgBouncer แทน
```

---

## Memory Configuration

### shared_buffers: Buffer Cache

**shared_buffers** คือ memory ที่ PostgreSQL ใช้ cache data pages จาก disk มากขึ้น = cache มากขึ้น = disk I/O น้อยลง

```
ค่าแนะนำ: 25% ของ total RAM

Server 8GB RAM  → shared_buffers = 2GB
Server 16GB RAM → shared_buffers = 4GB  
Server 32GB RAM → shared_buffers = 8GB
Server 64GB RAM → shared_buffers = 16GB
Server 128GB RAM → shared_buffers = 32GB
```

```sql
-- ดูค่าปัจจุบัน
SHOW shared_buffers;

-- ดูประสิทธิภาพ cache
SELECT 
  sum(heap_blks_read) as heap_read,
  sum(heap_blks_hit) as heap_hit,
  round(
    sum(heap_blks_hit)::numeric / 
    nullif(sum(heap_blks_hit) + sum(heap_blks_read), 0) * 100,
    2
  ) as cache_hit_ratio
FROM pg_statio_user_tables;

-- Cache hit ratio ควร > 99% สำหรับ hot data
-- ถ้าต่ำกว่า → เพิ่ม shared_buffers
```

### effective_cache_size: Planner Hint

**effective_cache_size** ไม่ใช่ memory allocation! เป็นแค่ hint บอก query planner ว่า OS disk cache + shared_buffers ทั้งหมดเท่าไหร่ Planner จะใช้ค่านี้ตัดสินใจว่าจะใช้ index scan หรือ seq scan

```
ค่าแนะนำ: 75% ของ total RAM

Server 32GB RAM → effective_cache_size = 24GB
```

```sql
-- PostgreSQL ใช้ค่านี้เพื่อ estimate cost of index vs seq scan
-- ค่าสูง = planner ชอบ index scan มากขึ้น
SHOW effective_cache_size;

-- ตรวจสอบว่า planner ตัดสินใจถูก
EXPLAIN (ANALYZE, BUFFERS) 
SELECT * FROM orders WHERE user_id = 123;

-- ถ้า planner เลือก Seq Scan ทั้งๆ ที่ควรเป็น Index Scan
-- อาจต้องเพิ่ม effective_cache_size
```

### work_mem: Per-Sort Memory

**work_mem** คือ memory สำหรับ sort operations, hash joins, hash aggregations **ต่อ operation** (ไม่ใช่ต่อ connection)

```
⚠️ ระวัง: work_mem = 64MB และมี 100 connections
แต่ละ query อาจมี 10 sort operations
Total: 100 * 10 * 64MB = 64GB RAM !!!

สูตร: work_mem = (Available RAM - shared_buffers) / (max_connections * 3)

Server 32GB RAM, shared_buffers=8GB, max_connections=200:
work_mem = (32 - 8)GB / (200 * 3) = 40MB ≈ round down to 32MB-64MB
```

```sql
-- ดู queries ที่ spill ไป disk
EXPLAIN (ANALYZE, BUFFERS)
SELECT user_id, count(*) 
FROM orders 
GROUP BY user_id 
ORDER BY count(*) DESC;

-- ถ้าเห็น "Batches: 8" ใน Hash → work_mem ไม่พอ
-- Sort Method: external merge Disk → work_mem ไม่พอ

-- ตั้งค่า work_mem สำหรับ session เฉพาะ (ไม่กระทบ global)
SET work_mem = '256MB';
EXPLAIN (ANALYZE) SELECT ...;
RESET work_mem;

-- ตั้งค่า per user สำหรับ heavy analytics queries
ALTER USER analytics_user SET work_mem = '512MB';
```

### maintenance_work_mem: Vacuum และ Index

**maintenance_work_mem** ใช้สำหรับ VACUUM, CREATE INDEX, ALTER TABLE ADD FOREIGN KEY ค่าสูง = VACUUM เร็วขึ้น, CREATE INDEX เร็วขึ้น

```
ค่าแนะนำ: 256MB - 2GB

-- ค่าสูงช่วย:
-- CREATE INDEX CONCURRENTLY บน large table
-- VACUUM ANALYZE บน bloated table
-- CLUSTER TABLE
```

```sql
-- ตั้งค่าชั่วคราวสำหรับ maintenance tasks
SET maintenance_work_mem = '2GB';
CREATE INDEX CONCURRENTLY idx_orders_user_id ON orders(user_id);
RESET maintenance_work_mem;

-- หรือ config file
-- maintenance_work_mem = 512MB
```

### wal_buffers: WAL Buffer

**wal_buffers** คือ shared memory สำหรับ WAL (Write-Ahead Log) data ก่อนเขียนลง disk

```
ค่าแนะนำ: 16MB หรือ -1 (auto = 1/32 of shared_buffers, max 64MB)
```

---

## Checkpoint Configuration

Checkpoint คือเวลาที่ PostgreSQL เขียน dirty pages จาก shared_buffers ลง disk การ tune checkpoint ส่งผลต่อ I/O pattern และ recovery time

### checkpoint_completion_target

บอกว่า PostgreSQL ควรกระจาย write ออกไปช่วงกี่ % ของ checkpoint interval

```
ค่าแนะนำ: 0.9

checkpoint_completion_target = 0.9 หมายความว่า:
ถ้า checkpoint ทุก 5 นาที → เขียน dirty pages ใน 4.5 นาที (90%)
= กระจาย I/O ออกไป ไม่ peak หนัก
```

### max_wal_size

ขนาด WAL สูงสุดระหว่าง checkpoints การเพิ่มค่านี้ = checkpoint บ่อยน้อยลง = I/O น้อยลง แต่ recovery time นานขึ้น

```
ค่าแนะนำ: 
- Write-heavy systems: 4GB-8GB
- Read-heavy systems: 1GB (default)

# เพิ่มถ้าเห็น log:
LOG: checkpoints are occurring too frequently (XX seconds apart)
```

### checkpoint_timeout

เวลาสูงสุดระหว่าง checkpoints

```
ค่าแนะนำ: 15min (จาก default 5min)

ผลกระทบ:
- ค่าสูง → I/O น้อยลง, recovery time นานขึ้น
- ค่าต่ำ → I/O มากขึ้น, recovery time สั้นลง
```

```sql
-- ดู checkpoint statistics
SELECT * FROM pg_stat_bgwriter;

-- Checkpoint ที่ดี: checkpoints_timed เยอะกว่า checkpoints_req
-- ถ้า checkpoints_req มาก = max_wal_size น้อยเกินไป

SELECT 
  checkpoints_timed,
  checkpoints_req,
  round(100.0 * checkpoints_req / (checkpoints_timed + checkpoints_req), 2) AS req_pct,
  buffers_checkpoint,
  buffers_clean,
  buffers_backend
FROM pg_stat_bgwriter;
```

---

## Parallel Query Configuration

### เปิดใช้ Parallel Query

```sql
-- ดูการตั้งค่าปัจจุบัน
SHOW max_worker_processes;
SHOW max_parallel_workers;
SHOW max_parallel_workers_per_gather;

-- Configuration แนะนำ
-- max_worker_processes = 8 (จำนวน CPU cores)
-- max_parallel_workers = 8 (เท่ากับ max_worker_processes)  
-- max_parallel_workers_per_gather = 4 (ครึ่งหนึ่งของ cores)
```

```
max_worker_processes = 8

ความหมาย:
- จำนวน background workers ทั้งหมดในระบบ
- รวม parallel workers, logical replication workers, autovacuum workers
- ตั้งให้เท่ากับหรือเกิน CPU cores

max_parallel_workers = 8

ความหมาย:
- จำนวน workers ที่ใช้ได้สำหรับ parallel queries
- ต้อง ≤ max_worker_processes

max_parallel_workers_per_gather = 4

ความหมาย:
- จำนวน workers สำหรับ query เดียว
- Query 1 ใช้ 4 workers = 5 processes total (1 leader + 4 workers)
```

### Parallel Query ตัวอย่าง

```sql
-- เปิด parallel query สำหรับ session นี้
SET max_parallel_workers_per_gather = 4;

-- ดู parallel plan
EXPLAIN (ANALYZE, BUFFERS)
SELECT 
  date_trunc('month', created_at) AS month,
  sum(total_amount) AS revenue
FROM orders
WHERE created_at >= NOW() - INTERVAL '1 year'
GROUP BY 1
ORDER BY 1;

-- ผลลัพธ์ที่คาดหวัง:
-- Gather  (cost=...) (actual time=... rows=... loops=1)
--   Workers Planned: 4
--   Workers Launched: 4
--   ->  Parallel Seq Scan on orders  (cost=...)
--         Workers Planned: 4

-- ถ้าไม่เห็น parallel:
-- 1. ตรวจสอบ parallel_setup_cost
-- 2. ตรวจสอบ table size (เล็กเกินไปไม่ parallel)
-- 3. ตรวจสอบ max_parallel_workers_per_gather > 0
```

### Tune Parallel Cost

```sql
-- parallel_setup_cost: overhead ของการ setup parallel workers
-- ค่าต่ำ = parallel ง่ายขึ้น
-- default = 1000
SET parallel_setup_cost = 500;

-- parallel_tuple_cost: overhead ของการส่ง tuple ระหว่าง workers
-- ค่าต่ำ = parallel ง่ายขึ้น  
-- default = 0.1
SET parallel_tuple_cost = 0.1;

-- min_parallel_table_scan_size: ขนาดต่ำสุดของ table ก่อน parallel
-- default = 8MB
-- ลดลงถ้าต้องการ parallel บน tables เล็กกว่า
SET min_parallel_table_scan_size = '8MB';
```

---

## Query Planner Configuration

### random_page_cost: SSD vs HDD

**random_page_cost** บอก planner ว่าการ random read 1 page แพงแค่ไหนเทียบกับ sequential read

```
HDD: random_page_cost = 4.0 (default)
     HDD random seek ~4ms = 40x slower than sequential

SSD: random_page_cost = 1.1
     SSD random read ~0.1ms = barely slower than sequential

NVMe SSD: random_page_cost = 1.0
     NVMe random ≈ sequential speed

ผลกระทบ:
- ค่าต่ำ → planner ชอบ index scan มากขึ้น
- ค่าสูง → planner ชอบ sequential scan มากขึ้น
```

```sql
-- ดูว่า storage เป็นอะไร
SHOW random_page_cost;

-- ถ้าใช้ SSD:
ALTER SYSTEM SET random_page_cost = 1.1;
SELECT pg_reload_conf();
```

### enable_seqscan: Debug Mode

```sql
-- ปิด sequential scan เพื่อ debug (planner จะต้องเลือก index)
-- อย่าปิด globally ใน production!
SET enable_seqscan = OFF;

EXPLAIN (ANALYZE) SELECT * FROM orders WHERE user_id = 123;
-- ถ้า Seq Scan หายแล้ว = มี index แต่ planner ไม่ใช้
-- ตรวจสอบสาเหตุ: statistics เก่า, ค่าไม่ selective, etc.

SET enable_seqscan = ON;  -- เปิดกลับเสมอ!

-- สาเหตุที่ planner ไม่ใช้ index:
-- 1. Statistics เก่า → ANALYZE
-- 2. Index ไม่ selective (เช่น column ที่มีค่า TRUE/FALSE)
-- 3. random_page_cost สูงเกินไป
-- 4. n_distinct ประมาณผิด
```

### Statistics Target

```sql
-- เพิ่ม statistics สำหรับ columns ที่ใช้ filter บ่อย
-- default = 100 samples

ALTER TABLE orders ALTER COLUMN status SET STATISTICS 500;
ALTER TABLE orders ALTER COLUMN created_at SET STATISTICS 1000;

-- รัน ANALYZE หลังจากเปลี่ยน
ANALYZE orders;

-- ดู statistics ปัจจุบัน
SELECT 
  attname,
  n_distinct,
  correlation,
  most_common_vals,
  most_common_freqs
FROM pg_stats
WHERE tablename = 'orders'
ORDER BY attname;
```

---

## Autovacuum Tuning

Autovacuum มีหน้าที่สำคัญ:
1. ลบ dead tuples (ที่ MVCC สร้างขึ้น)
2. ป้องกัน transaction ID wraparound
3. อัปเดต table statistics

### ปัญหาของ Default Autovacuum

```
Table orders: 10 ล้าน rows
Default autovacuum_vacuum_scale_factor = 0.2 (20%)

Trigger vacuum เมื่อ: 10,000,000 * 0.2 + 50 = 2,000,050 dead tuples

ปัญหา: ต้อง dead tuples 2 ล้านแถว ถึงจะ vacuum!
= table bloat สูงมาก
= query ช้าเพราะต้องอ่าน dead tuples
```

### Tune Autovacuum

```sql
-- =============== Global Settings ===============
-- autovacuum_vacuum_scale_factor: % ของ table size ก่อน trigger
-- Default: 0.2 (20%) → ลดลงสำหรับ large tables
ALTER SYSTEM SET autovacuum_vacuum_scale_factor = 0.05;  -- 5%

-- autovacuum_analyze_scale_factor: % ก่อน trigger ANALYZE
ALTER SYSTEM SET autovacuum_analyze_scale_factor = 0.02;  -- 2%

-- autovacuum_vacuum_threshold: จำนวน dead tuples ขั้นต่ำ
ALTER SYSTEM SET autovacuum_vacuum_threshold = 100;  -- default 50

-- autovacuum_analyze_threshold: 
ALTER SYSTEM SET autovacuum_analyze_threshold = 50;  -- default 50

-- autovacuum_vacuum_cost_delay: ลดเพื่อให้ vacuum เร็วขึ้น
-- 0 = ไม่มี delay (aggressive vacuum)
-- 2ms = default
ALTER SYSTEM SET autovacuum_vacuum_cost_delay = 2;  -- 2ms

-- autovacuum_vacuum_cost_limit: เพิ่มเพื่อให้ vacuum ทำงานนานขึ้น
ALTER SYSTEM SET autovacuum_vacuum_cost_limit = 400;  -- default 200

-- autovacuum_max_workers: เพิ่มจำนวน workers
ALTER SYSTEM SET autovacuum_max_workers = 6;  -- default 3

SELECT pg_reload_conf();
```

### Table-Level Autovacuum Settings

```sql
-- สำหรับ large, high-traffic tables
ALTER TABLE orders SET (
  autovacuum_vacuum_scale_factor = 0.01,     -- Vacuum เมื่อ 1% dead
  autovacuum_vacuum_threshold = 1000,         -- หรือ 1000 dead tuples
  autovacuum_analyze_scale_factor = 0.005,   -- Analyze เมื่อ 0.5% changed
  autovacuum_vacuum_cost_delay = 0,           -- No throttling (aggressive)
  autovacuum_vacuum_cost_limit = 800          -- ทำงานได้มากขึ้น
);

-- สำหรับ append-only tables (logs, events)
ALTER TABLE events SET (
  autovacuum_vacuum_scale_factor = 0.5,       -- Vacuum ไม่บ่อย (insert-only)
  autovacuum_analyze_scale_factor = 0.1
);

-- ดูการตั้งค่า per-table
SELECT 
  relname,
  reloptions
FROM pg_class
WHERE reloptions IS NOT NULL
  AND relkind = 'r';
```

### Monitor Autovacuum

```sql
-- ดูว่า autovacuum ทำงานอยู่หรือไม่
SELECT 
  pid,
  query_start,
  state,
  LEFT(query, 80) AS query
FROM pg_stat_activity
WHERE query LIKE '%autovacuum%';

-- ดู vacuum history ของแต่ละ table
SELECT 
  schemaname,
  tablename,
  last_vacuum,
  last_autovacuum,
  last_analyze,
  last_autoanalyze,
  n_dead_tup,
  n_live_tup,
  round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_pct,
  autovacuum_count,
  autoanalyze_count
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 20;

-- Tables ที่ต้องการ manual vacuum
SELECT 
  schemaname || '.' || tablename AS table_name,
  n_dead_tup,
  n_live_tup,
  pg_size_pretty(pg_total_relation_size(relid)) AS total_size
FROM pg_stat_user_tables
WHERE n_dead_tup > 100000
ORDER BY n_dead_tup DESC;
```

---

## Connection Settings

### max_connections

```sql
-- max_connections = จำนวน connection สูงสุด
-- แต่ละ connection ใช้ ~5-10MB RAM (shared memory)
-- max_connections = 100 → ~500MB-1GB reserved

-- ค่าแนะนำ:
-- ใช้ PgBouncer: max_connections = 100-200
-- ไม่ใช้ PgBouncer: max_connections = ตาม actual need + 20%

-- หลักการ:
-- Application connections: N
-- Replication: 5-10
-- Superuser reserved: 3
-- Total: N + 15
ALTER SYSTEM SET max_connections = 200;  -- ต้อง restart

-- ดู connection stats
SELECT 
  count(*) AS total,
  count(*) FILTER (WHERE state = 'active') AS active,
  count(*) FILTER (WHERE state = 'idle') AS idle,
  count(*) FILTER (WHERE state = 'idle in transaction') AS idle_in_txn,
  count(*) FILTER (WHERE wait_event IS NOT NULL) AS waiting
FROM pg_stat_activity
WHERE pid != pg_backend_pid();
```

### WAL Configuration

```sql
-- wal_level: ระดับ WAL logging
-- minimal: ข้อมูลน้อยที่สุด (backup เท่านั้น)
-- replica: รองรับ streaming replication (default)
-- logical: รองรับ logical replication
ALTER SYSTEM SET wal_level = replica;

-- fsync: ถ้า off = เร็วขึ้นมาก แต่ data loss เมื่อ crash!
-- ไม่ควรปิดใน production เด็ดขาด
-- fsync = on (default, ALWAYS keep this)

-- synchronous_commit: ระดับ durability
-- off = เร็วสุด แต่ last ~0.5s transactions อาจหาย
-- local = commit เมื่อ WAL บันทึกลง primary disk
-- remote_write = รอ replica รับ WAL แต่ไม่ต้อง fsync
-- remote_apply = รอ replica apply WAL
-- on = commit เมื่อ replica acknowledge (default)

-- สำหรับ non-critical data:
SET synchronous_commit = off;

-- สำหรับ financial/critical data:
SET synchronous_commit = on;
```

---

## Table Partitioning for Performance

### List Partitioning

```sql
-- Partition orders ตาม status
CREATE TABLE orders (
  id BIGSERIAL,
  user_id BIGINT NOT NULL,
  status VARCHAR(20) NOT NULL,
  total_amount DECIMAL(10,2),
  created_at TIMESTAMPTZ DEFAULT NOW()
) PARTITION BY LIST (status);

CREATE TABLE orders_pending PARTITION OF orders
  FOR VALUES IN ('pending');

CREATE TABLE orders_processing PARTITION OF orders
  FOR VALUES IN ('processing');

CREATE TABLE orders_completed PARTITION OF orders
  FOR VALUES IN ('completed');

CREATE TABLE orders_cancelled PARTITION OF orders
  FOR VALUES IN ('cancelled');

-- Index ต้องสร้างบน each partition
CREATE INDEX idx_orders_pending_user ON orders_pending(user_id);
CREATE INDEX idx_orders_completed_user ON orders_completed(user_id, created_at);
```

### Range Partitioning by Date

```sql
-- Partition orders ตาม created_at (ทุกเดือน)
CREATE TABLE orders_history (
  id BIGINT,
  user_id BIGINT NOT NULL,
  total_amount DECIMAL(10,2),
  created_at TIMESTAMPTZ NOT NULL
) PARTITION BY RANGE (created_at);

-- สร้าง partitions
CREATE TABLE orders_2024_01 PARTITION OF orders_history
  FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
  
CREATE TABLE orders_2024_02 PARTITION OF orders_history
  FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- Automate partition creation
CREATE OR REPLACE FUNCTION create_monthly_partition(table_name TEXT, year INT, month INT)
RETURNS VOID AS $$
DECLARE
  start_date DATE := make_date(year, month, 1);
  end_date DATE := start_date + INTERVAL '1 month';
  partition_name TEXT := table_name || '_' || to_char(start_date, 'YYYY_MM');
BEGIN
  EXECUTE format(
    'CREATE TABLE IF NOT EXISTS %I PARTITION OF %I
     FOR VALUES FROM (%L) TO (%L)',
    partition_name, table_name, start_date, end_date
  );
  
  -- สร้าง index
  EXECUTE format(
    'CREATE INDEX IF NOT EXISTS %I ON %I (user_id, created_at)',
    'idx_' || partition_name || '_user',
    partition_name
  );
  
  RAISE NOTICE 'Created partition: %', partition_name;
END;
$$ LANGUAGE plpgsql;

-- สร้าง partitions สำหรับปีนี้
SELECT create_monthly_partition('orders_history', 2024, generate_series(1, 12));

-- ดู partitions
SELECT 
  parent.relname AS parent_table,
  child.relname AS partition_name,
  pg_get_expr(child.relpartbound, child.oid) AS partition_bound
FROM pg_inherits
JOIN pg_class parent ON pg_inherits.inhparent = parent.oid
JOIN pg_class child ON pg_inherits.inhrelid = child.oid
WHERE parent.relname = 'orders_history'
ORDER BY child.relname;
```

### Partition Pruning

```sql
-- Partition pruning: Postgres ข้ามพาร์ติชันที่ไม่เกี่ยวข้อง
EXPLAIN (ANALYZE)
SELECT count(*) FROM orders_history
WHERE created_at >= '2024-01-01' AND created_at < '2024-02-01';

-- ต้องเห็น: Partitions removed: N
-- ถ้าไม่เห็น: ตรวจสอบ enable_partition_pruning = on
SHOW enable_partition_pruning;
```

---

## Materialized Views

### สร้าง Materialized View

```sql
-- Sales summary report (ช้ามากถ้า query ตาม)
CREATE MATERIALIZED VIEW sales_summary AS
SELECT 
  date_trunc('day', created_at) AS sale_date,
  p.category_id,
  c.name AS category_name,
  count(DISTINCT o.id) AS order_count,
  count(oi.id) AS item_count,
  sum(oi.quantity * oi.unit_price) AS revenue,
  avg(o.total_amount) AS avg_order_value
FROM orders o
JOIN order_items oi ON o.id = oi.order_id
JOIN products p ON oi.product_id = p.id
JOIN categories c ON p.category_id = c.id
WHERE o.status = 'completed'
GROUP BY 1, 2, 3
ORDER BY 1 DESC, 4 DESC;

-- สร้าง index บน materialized view
CREATE INDEX idx_sales_summary_date ON sales_summary(sale_date);
CREATE INDEX idx_sales_summary_category ON sales_summary(category_id, sale_date);

-- Query ที่เร็วมาก (ไม่ต้อง join 4 tables)
SELECT * FROM sales_summary
WHERE sale_date >= CURRENT_DATE - INTERVAL '30 days'
ORDER BY sale_date DESC, revenue DESC;

-- Refresh (ต้อง run เอง หรือ schedule)
REFRESH MATERIALIZED VIEW sales_summary;

-- Refresh แบบ Concurrent (ไม่ lock, ต้องมี unique index)
CREATE UNIQUE INDEX idx_sales_summary_unique 
ON sales_summary(sale_date, category_id);

REFRESH MATERIALIZED VIEW CONCURRENTLY sales_summary;
```

### Automate Refresh

```sql
-- ใช้ pg_cron extension สำหรับ schedule
CREATE EXTENSION IF NOT EXISTS pg_cron;

-- Refresh ทุกชั่วโมง
SELECT cron.schedule(
  'refresh-sales-summary',
  '0 * * * *',  -- ทุกชั่วโมง
  'REFRESH MATERIALIZED VIEW CONCURRENTLY sales_summary'
);

-- ดู scheduled jobs
SELECT * FROM cron.job;

-- Refresh หลาย views ตามลำดับ
SELECT cron.schedule(
  'refresh-all-summaries',
  '30 1 * * *',  -- 01:30 ทุกวัน
  $$
    REFRESH MATERIALIZED VIEW CONCURRENTLY sales_summary;
    REFRESH MATERIALIZED VIEW CONCURRENTLY product_stats;
    REFRESH MATERIALIZED VIEW CONCURRENTLY user_metrics;
  $$
);
```

---

## Partial Indexes

### Index เฉพาะ Active Records

```sql
-- ถ้า query ส่วนใหญ่ดูเฉพาะ active orders
-- สร้าง partial index เฉพาะ active/pending
CREATE INDEX idx_orders_active ON orders(user_id, created_at)
WHERE status IN ('pending', 'processing');

-- Index จะเล็กกว่ามาก (ไม่มี completed/cancelled)
-- Query นี้ใช้ partial index ได้:
SELECT * FROM orders 
WHERE user_id = 123 AND status IN ('pending', 'processing');

-- =====================================

-- Partial index สำหรับ NULL values
CREATE INDEX idx_orders_unshipped ON orders(created_at)
WHERE shipped_at IS NULL;

-- Query ที่ใช้ partial index:
SELECT * FROM orders 
WHERE shipped_at IS NULL 
ORDER BY created_at;

-- =====================================

-- Index เฉพาะ VIP users
CREATE INDEX idx_orders_vip ON orders(user_id, created_at)
WHERE total_amount > 10000;

-- Query:
SELECT * FROM orders 
WHERE user_id = 123 AND total_amount > 10000;
```

### Expression Indexes

```sql
-- Index บน expression/function
-- Query บน case-insensitive email
CREATE INDEX idx_users_email_lower ON users(LOWER(email));

-- Query ต้องใช้ expression เดียวกัน:
SELECT * FROM users WHERE LOWER(email) = 'user@example.com';

-- =====================================

-- Index บน JSON field
CREATE INDEX idx_orders_metadata_channel 
ON orders((metadata->>'channel'));

-- Query:
SELECT * FROM orders WHERE metadata->>'channel' = 'mobile';

-- =====================================

-- Composite expression index
CREATE INDEX idx_orders_year_month 
ON orders(date_part('year', created_at), date_part('month', created_at));

-- Query:
SELECT * FROM orders 
WHERE date_part('year', created_at) = 2024 
  AND date_part('month', created_at) = 1;
```

---

## pg_prewarm: Warm Buffer Cache

```sql
-- Extension สำหรับ pre-load tables เข้า shared_buffers
CREATE EXTENSION pg_prewarm;

-- Load table ทั้งหมดเข้า buffer cache
SELECT pg_prewarm('orders');

-- Load specific index
SELECT pg_prewarm('idx_orders_user_id');

-- Load แบบ buffer (ใช้ shared_buffers, persistent)
SELECT pg_prewarm('hot_table', 'buffer');

-- Load แบบ prefetch (ใช้ OS cache, async)
SELECT pg_prewarm('cold_table', 'prefetch');

-- Script สำหรับ prewarm หลัง restart
CREATE OR REPLACE FUNCTION prewarm_critical_tables()
RETURNS void AS $$
BEGIN
  -- Tables ที่ใช้บ่อย
  PERFORM pg_prewarm('users', 'buffer');
  PERFORM pg_prewarm('products', 'buffer');
  PERFORM pg_prewarm('active_sessions', 'buffer');
  
  -- Critical indexes
  PERFORM pg_prewarm('idx_orders_user_id');
  PERFORM pg_prewarm('idx_orders_created_at');
  
  RAISE NOTICE 'Prewarm complete';
END;
$$ LANGUAGE plpgsql;

-- เรียกหลัง restart
SELECT prewarm_critical_tables();
```

---

## PgTune: Automatic Configuration

```bash
# ใช้ PgTune online: https://pgtune.leopard.in.ua/
# หรือ command line:

# ตัวอย่าง output สำหรับ Server 32GB RAM, 8 CPU, SSD

# DB Version: 16
# OS Type: linux
# DB Type: web
# Total Memory (RAM): 32 GB
# CPUs num: 8
# Connections num: 200
# Data Storage: ssd

cat > /etc/postgresql/16/main/conf.d/pgtune.conf << 'EOF'
# Generated by PgTune
max_connections = 200
shared_buffers = 8GB
effective_cache_size = 24GB
maintenance_work_mem = 2GB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 100
random_page_cost = 1.1
effective_io_concurrency = 200
work_mem = 20971kB
min_wal_size = 2GB
max_wal_size = 8GB
max_worker_processes = 8
max_parallel_workers_per_gather = 4
max_parallel_workers = 8
max_parallel_maintenance_workers = 4
EOF

# Reload config
sudo systemctl reload postgresql
```

---

## Benchmarking ด้วย pgbench

### Initialize pgbench Database

```bash
# Initialize test database (scale = 100 = 10M rows)
pgbench -i -s 100 -h localhost -U postgres testdb

# Options:
# -s 100 = scale factor 100 (= 100 * 10,000 rows = 1M rows)
# -F 90  = fillfactor 90%
# --foreign-keys = เพิ่ม foreign keys

# ตรวจสอบ tables ที่สร้าง
psql -U postgres testdb -c "\dt pgbench_*"
# pgbench_accounts  (rows = scale * 100,000)
# pgbench_branches  (rows = scale)
# pgbench_history   (rows = 0, transaction log)
# pgbench_tellers   (rows = scale * 10)
```

### Run pgbench Tests

```bash
# Basic test: 10 clients, 4 threads, 60 seconds
pgbench -c 10 -j 4 -T 60 -h localhost -U postgres testdb

# Output:
# starting vacuum...end.
# transaction type: <builtin: TPC-B (sort of)>
# scaling factor: 100
# query mode: simple
# number of clients: 10
# number of threads: 4
# duration: 60 s
# number of transactions actually processed: 74821
# latency average = 8.021 ms
# latency stddev = 8.493 ms
# tps = 1246.893 (without initial connection time)

# Test ด้วย read-only workload
pgbench -c 20 -j 4 -T 60 -S -h localhost -U postgres testdb
# -S = SELECT only (read-only)

# Test ด้วย custom SQL
cat > /tmp/test_query.sql << 'EOF'
\set aid random(1, 100000 * :scale)
SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
EOF

pgbench -c 50 -j 8 -T 120 -f /tmp/test_query.sql testdb

# Test เปรียบเทียบ config
# Before tuning:
pgbench -c 10 -j 4 -T 30 testdb 2>&1 | grep "tps ="
# tps = 1100

# After tuning:
pgbench -c 10 -j 4 -T 30 testdb 2>&1 | grep "tps ="
# tps = 3500

# Ramp-up test: เพิ่ม clients ขึ้นเรื่อยๆ
for clients in 1 5 10 20 50 100 200; do
  echo -n "Clients: $clients → "
  pgbench -c $clients -j 4 -T 30 testdb 2>&1 | grep "tps ="
done

# Progressive test output:
# Clients: 1  → tps = 350
# Clients: 5  → tps = 1500
# Clients: 10 → tps = 2800
# Clients: 20 → tps = 4200 ← peak
# Clients: 50 → tps = 3800 ← connection overhead
# Clients: 100 → tps = 2500 ← contention
# Clients: 200 → tps = 1500 ← context switching

# Latency percentiles
pgbench -c 10 -j 4 -T 60 --report-per-command testdb
# Shows p50, p95, p99 latency per command type
```

### pgbench Custom Scripts

```bash
# สร้าง test script จำลอง real workload
cat > /tmp/realistic_test.sql << 'EOF'
-- Mix of read (70%) and write (30%)
\set user_id random(1, 1000000)
\set order_id random(1, 5000000)

-- 70% chance: read order
SELECT CASE WHEN random() < 0.7 THEN
  (SELECT count(*) FROM orders WHERE user_id = :user_id)
ELSE
  0
END;

-- 20% chance: update
UPDATE orders 
SET updated_at = NOW()
WHERE id = :order_id AND random() < 0.2;

-- 10% chance: insert
INSERT INTO order_events (order_id, event_type, created_at)
SELECT :order_id, 'view', NOW()
WHERE random() < 0.1;
EOF

pgbench -c 20 -j 4 -T 60 -f /tmp/realistic_test.sql testdb
```

---

## I/O Optimization

### pg_statio_user_tables

```sql
-- ดู I/O stats ของแต่ละ table
SELECT 
  relname AS table_name,
  heap_blks_read AS disk_reads,
  heap_blks_hit AS cache_hits,
  round(
    100.0 * heap_blks_hit / NULLIF(heap_blks_hit + heap_blks_read, 0),
    2
  ) AS cache_hit_ratio,
  idx_blks_read AS index_disk_reads,
  idx_blks_hit AS index_cache_hits,
  round(
    100.0 * idx_blks_hit / NULLIF(idx_blks_hit + idx_blks_read, 0),
    2
  ) AS index_cache_hit_ratio
FROM pg_statio_user_tables
WHERE heap_blks_read + heap_blks_hit > 0
ORDER BY heap_blks_read DESC
LIMIT 20;

-- Tables ที่มี cache hit ratio ต่ำ = ควรเพิ่ม shared_buffers หรือ prewarm
```

### effective_io_concurrency

```sql
-- จำนวน disk requests concurrent ที่ PostgreSQL จะส่งพร้อมกัน
-- HDD: 2-4 (spindle หมุนเดียว)
-- SSD: 100-200
-- NVMe: 1000

ALTER SYSTEM SET effective_io_concurrency = 200;  -- SSD
SELECT pg_reload_conf();

-- ใช้สำหรับ Bitmap Index Scan prefetching
```

---

## Full Configuration สำหรับ Production

```ini
# /etc/postgresql/16/main/postgresql.conf
# ====================================
# Server: 32 CPU cores, 128GB RAM, NVMe SSD
# Workload: Mixed OLTP
# ====================================

# -------- Memory --------
shared_buffers = 32GB               # 25% of 128GB
effective_cache_size = 96GB          # 75% of 128GB
work_mem = 128MB                     # Per sort/hash
maintenance_work_mem = 4GB           # Vacuum/index
huge_pages = try                     # ใช้ huge pages ถ้าได้

# -------- WAL --------
wal_level = replica
max_wal_size = 8GB
min_wal_size = 2GB
wal_buffers = 64MB
checkpoint_completion_target = 0.9
checkpoint_timeout = 15min
archive_mode = on
archive_command = 'cp %p /var/lib/postgresql/wal_archive/%f'

# -------- Connections --------
max_connections = 300
superuser_reserved_connections = 5

# -------- Parallel --------
max_worker_processes = 32
max_parallel_workers = 32
max_parallel_workers_per_gather = 8
max_parallel_maintenance_workers = 8
parallel_setup_cost = 500
parallel_tuple_cost = 0.1

# -------- Planner --------
random_page_cost = 1.0               # NVMe
effective_io_concurrency = 1000      # NVMe
default_statistics_target = 200

# -------- Autovacuum --------
autovacuum_max_workers = 8
autovacuum_vacuum_scale_factor = 0.05
autovacuum_analyze_scale_factor = 0.02
autovacuum_vacuum_cost_delay = 2ms
autovacuum_vacuum_cost_limit = 400
autovacuum_naptime = 30s

# -------- Logging --------
log_min_duration_statement = 1000    # Log queries > 1s
log_checkpoints = on
log_connections = off
log_disconnections = off
log_lock_waits = on
log_temp_files = 0                   # Log all temp files
log_autovacuum_min_duration = 250ms  # Log slow autovacuums

# -------- Performance --------
synchronous_commit = on
fsync = on                           # NEVER disable!
full_page_writes = on                # NEVER disable!
track_activity_query_size = 4096
shared_preload_libraries = 'pg_stat_statements,auto_explain,pg_prewarm'

# auto_explain: log slow query plans
auto_explain.log_min_duration = 5000  # Plans > 5s
auto_explain.log_analyze = on
auto_explain.log_buffers = on
auto_explain.log_nested_statements = on
```

---

## สรุป: PostgreSQL Performance Tuning Checklist

```markdown
## PostgreSQL Performance Tuning Checklist

### Memory
- [ ] shared_buffers = 25% RAM
- [ ] effective_cache_size = 75% RAM
- [ ] work_mem ปรับตาม connections และ RAM
- [ ] maintenance_work_mem ปรับสำหรับ VACUUM/Index

### WAL & Checkpoints
- [ ] checkpoint_completion_target = 0.9
- [ ] max_wal_size เพิ่มถ้า checkpoint ถี่เกิน
- [ ] wal_buffers = 16-64MB

### Parallel Query
- [ ] max_parallel_workers = CPU cores
- [ ] max_parallel_workers_per_gather = CPU/2
- [ ] parallel_setup_cost ปรับตาม workload

### Planner
- [ ] random_page_cost = 1.1 (SSD), 1.0 (NVMe)
- [ ] effective_io_concurrency ตาม storage
- [ ] Statistics target เพิ่มสำหรับ important columns

### Autovacuum
- [ ] autovacuum_vacuum_scale_factor ลดสำหรับ large tables
- [ ] Per-table autovacuum สำหรับ high-traffic tables
- [ ] Monitor dead tuple ratio < 5%

### Monitoring
- [ ] pg_stat_statements enabled
- [ ] log_min_duration_statement = 1000ms
- [ ] Cache hit ratio > 99%
- [ ] Regular EXPLAIN ANALYZE บน slow queries
- [ ] pgbench สำหรับ regression testing

### Tools
- [ ] PgTune สำหรับ initial config
- [ ] pgbench สำหรับ load testing
- [ ] pg_prewarm หลัง restart
- [ ] auto_explain สำหรับ query plan logging
```

---

*Part 59 เสร็จสมบูรณ์ - ต่อไป Part 60: Performance Tuning Redis*

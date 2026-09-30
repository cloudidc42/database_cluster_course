# Part 63: Vacuum และ Autovacuum Tuning

## บทนำ: MVCC และ Dead Tuples

PostgreSQL ใช้ **MVCC (Multi-Version Concurrency Control)** เพื่อรองรับ concurrent transactions ซึ่งทำให้เกิด "dead tuples" ที่ต้องการ cleanup

### MVCC ทำงานอย่างไร

```
-- Transaction 1: อ่านข้อมูล
BEGIN;
SELECT * FROM orders WHERE id = 100;
-- เห็น version เก่า (tuple A)

-- Transaction 2: อัปเดตข้อมูล (concurrent)
BEGIN;
UPDATE orders SET status = 'completed' WHERE id = 100;
-- สร้าง tuple ใหม่ (tuple B) แต่ tuple A ยังอยู่
COMMIT;

-- Transaction 1: ยังเห็น tuple A (snapshot isolation)
SELECT * FROM orders WHERE id = 100;
-- เห็น version เดิม (tuple A) ตาม snapshot time
COMMIT;

-- หลัง Transaction 1 COMMIT:
-- tuple A = dead tuple (ไม่มีใครต้องการแล้ว)
-- VACUUM จะ reclaim space นี้
```

### Dead Tuples สะสม

```sql
-- ดูจำนวน dead tuples ในแต่ละ table
SELECT 
    schemaname,
    tablename,
    n_live_tup as live_tuples,
    n_dead_tup as dead_tuples,
    ROUND(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) as dead_pct,
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 20;

-- ดู table bloat โดยรวม
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname || '.' || tablename)) as total_size,
    pg_size_pretty(pg_relation_size(schemaname || '.' || tablename)) as table_size,
    n_live_tup,
    n_dead_tup,
    ROUND(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) as bloat_pct
FROM pg_stat_user_tables
WHERE pg_relation_size(schemaname || '.' || tablename) > 10 * 1024 * 1024  -- > 10MB
ORDER BY bloat_pct DESC;
```

---

## VACUUM: คำสั่งและ Options

### VACUUM พื้นฐาน

```sql
-- VACUUM ธรรมดา: reclaim space จาก dead tuples
-- ไม่ lock table (ยกเว้นตอน access exclusive ช่วงสั้นๆ)
VACUUM orders;

-- VACUUM ANALYZE: vacuum + update statistics
VACUUM ANALYZE orders;

-- VACUUM VERBOSE: แสดงข้อมูลละเอียด
VACUUM VERBOSE orders;

-- VACUUM ทั้ง database
VACUUM;

-- VACUUM ทั้ง database แบบ verbose
VACUUM VERBOSE;
```

### VACUUM FULL: ใช้ด้วยความระมัดระวัง

```sql
-- VACUUM FULL: เขียน table ใหม่ทั้งหมด
-- LOCK: ACCESS EXCLUSIVE (ไม่สามารถ query ระหว่างนั้น)
-- คืน disk space กลับ OS (VACUUM ธรรมดาไม่คืน)
-- ใช้เวลานานมาก

-- ตรวจสอบขนาดก่อน
SELECT pg_size_pretty(pg_relation_size('orders'));

-- Run VACUUM FULL (ระวัง: locks table!)
VACUUM FULL VERBOSE orders;

-- ตรวจสอบขนาดหลัง
SELECT pg_size_pretty(pg_relation_size('orders'));
```

### VACUUM FREEZE: ป้องกัน XID Wraparound

```sql
-- Force freeze tuples เพื่อป้องกัน wraparound
VACUUM FREEZE orders;

-- ตรวจสอบ freeze status
SELECT 
    schemaname,
    tablename,
    age(relfrozenxid) as xid_age,
    2000000000 - age(relfrozenxid) as xids_until_emergency
FROM pg_class
JOIN pg_namespace ON pg_class.relnamespace = pg_namespace.oid
WHERE relkind = 'r'
    AND schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY xid_age DESC
LIMIT 20;

-- ดู database-level freeze status
SELECT 
    datname,
    age(datfrozenxid) as xid_age,
    2000000000 - age(datfrozenxid) as xids_until_emergency
FROM pg_database
ORDER BY xid_age DESC;
```

---

## Transaction ID (XID) Wraparound

### ทำความเข้าใจ XID

```
XID: 32-bit integer = ค่าสูงสุด ~4.3 billion
PostgreSQL จัดการ modular arithmetic:
- XID ปัจจุบัน: 2,000,000,000
- XID เก่าสุดที่ "visible": 1 (ก่อนหน้า)
- XID ในอนาคต: 2,000,000,001 และต่อไป

Wraparound:
เมื่อ XID ถึง ~4 billion, วนกลับไปที่ 3
XID เก่าอาจกลายเป็น "อนาคต" → ข้อมูลหายไป!
```

```sql
-- ดู XID age ของทุก database
SELECT 
    datname,
    age(datfrozenxid) as xid_age,
    current_setting('autovacuum_freeze_max_age') as freeze_max_age,
    CASE 
        WHEN age(datfrozenxid) > 1500000000 THEN 'CRITICAL'
        WHEN age(datfrozenxid) > 1000000000 THEN 'WARNING'
        WHEN age(datfrozenxid) > 500000000  THEN 'WATCH'
        ELSE 'OK'
    END as status
FROM pg_database
ORDER BY xid_age DESC;

-- ดู tables ที่ต้องการ freeze เร่งด่วน
SELECT 
    nspname as schema,
    relname as table,
    age(relfrozenxid) as xid_age,
    pg_size_pretty(pg_total_relation_size(pg_class.oid)) as size
FROM pg_class
JOIN pg_namespace ON relnamespace = pg_namespace.oid
WHERE relkind = 'r'
    AND age(relfrozenxid) > 1800000000  -- Very close to limit
ORDER BY xid_age DESC;
```

### Emergency Vacuum: vacuum_failsafe_age

```sql
-- ดูค่า failsafe age
SHOW vacuum_failsafe_age;  -- default: 1.6 billion

-- เมื่อ age > vacuum_failsafe_age:
-- PostgreSQL จะ VACUUM แบบก้าวร้าว ignoring cost limits
-- เพื่อป้องกัน wraparound

-- ดู autovacuum_freeze_max_age
SHOW autovacuum_freeze_max_age;  -- default: 200 million

-- ปรับ freeze parameters
ALTER SYSTEM SET autovacuum_freeze_max_age = 150000000;  -- ก่อนหน้าเดิม
ALTER SYSTEM SET vacuum_freeze_min_age = 50000000;       -- Freeze sooner
ALTER SYSTEM SET vacuum_freeze_table_age = 150000000;    -- Force vacuum sooner
SELECT pg_reload_conf();

-- ตรวจสอบ
SHOW vacuum_freeze_min_age;
SHOW vacuum_freeze_table_age;
SHOW autovacuum_freeze_max_age;
```

---

## Autovacuum: Configuration ละเอียด

### Parameters สำคัญ

```sql
-- ดู autovacuum settings ทั้งหมด
SELECT name, setting, unit, short_desc
FROM pg_settings
WHERE name LIKE 'autovacuum%'
ORDER BY name;

-- ดูแยกทีละตัว
SHOW autovacuum;                          -- on/off
SHOW autovacuum_max_workers;             -- default: 3
SHOW autovacuum_naptime;                 -- default: 1min (ระยะเวลาระหว่าง runs)
SHOW autovacuum_vacuum_threshold;        -- default: 50 rows
SHOW autovacuum_vacuum_scale_factor;     -- default: 0.2 (20% ของ table)
SHOW autovacuum_analyze_threshold;       -- default: 50 rows
SHOW autovacuum_analyze_scale_factor;    -- default: 0.1 (10% ของ table)
SHOW autovacuum_vacuum_cost_delay;       -- default: 2ms
SHOW autovacuum_vacuum_cost_limit;       -- default: -1 (ใช้ vacuum_cost_limit)
```

### คำนวณ Autovacuum Trigger

```
Vacuum triggered เมื่อ:
dead_tuples > vacuum_threshold + vacuum_scale_factor × table_rows

ตัวอย่าง table ขนาดต่างๆ:

Small table (1,000 rows):
  threshold = 50 + 0.2 × 1,000 = 250 dead tuples
  
Medium table (100,000 rows):
  threshold = 50 + 0.2 × 100,000 = 20,050 dead tuples
  
Large table (10,000,000 rows):
  threshold = 50 + 0.2 × 10,000,000 = 2,000,050 dead tuples
  ← ปัญหา: ต้องรอ 2 ล้าน dead tuples ก่อน vacuum!
```

### ปรับ Autovacuum สำหรับ Large Tables

```sql
-- ปัญหา: large tables อาจมี bloat มากก่อน autovacuum ทำงาน
-- แก้ไข: ปรับ per-table settings

-- Table ที่ update บ่อย
ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor = 0.01,  -- 1% แทน 20%
    autovacuum_vacuum_threshold = 1000,      -- rows
    autovacuum_analyze_scale_factor = 0.01,
    autovacuum_analyze_threshold = 500,
    autovacuum_vacuum_cost_delay = 10,       -- ms
    autovacuum_vacuum_cost_limit = 400       -- cost units
);

-- High-throughput table (เช่น session logs)
ALTER TABLE session_logs SET (
    autovacuum_vacuum_scale_factor = 0.005, -- 0.5%
    autovacuum_vacuum_threshold = 5000,
    autovacuum_vacuum_cost_delay = 5,
    autovacuum_vacuum_cost_limit = 800,
    autovacuum_enabled = ON
);

-- Table ที่แทบไม่ update (lookup tables)
ALTER TABLE countries SET (
    autovacuum_vacuum_scale_factor = 0.5,   -- 50%
    autovacuum_analyze_scale_factor = 0.5
);

-- ดู per-table autovacuum settings
SELECT 
    relname as table,
    reloptions as autovacuum_settings
FROM pg_class
WHERE relkind = 'r'
    AND reloptions IS NOT NULL
ORDER BY relname;
```

### Autovacuum Workers

```sql
-- เพิ่ม workers สำหรับ high-load database
ALTER SYSTEM SET autovacuum_max_workers = 5;

-- ดู autovacuum workers ที่กำลังทำงาน
SELECT 
    pid,
    query,
    query_start,
    EXTRACT(EPOCH FROM (NOW() - query_start)) as running_seconds
FROM pg_stat_activity
WHERE query LIKE 'autovacuum:%'
ORDER BY query_start;

-- ดู autovacuum activity ในรายละเอียด
SELECT 
    phase,
    heap_blks_total,
    heap_blks_scanned,
    ROUND(100.0 * heap_blks_scanned / NULLIF(heap_blks_total, 0), 2) as pct_scanned,
    index_vacuum_count,
    max_dead_tuples,
    num_dead_tuples
FROM pg_stat_progress_vacuum;
```

### Autovacuum Cost-Based Throttling

```sql
-- autovacuum ใช้ cost-based throttling เพื่อไม่รบกวน production
-- vacuum_cost_page_hit: ราคา hit page จาก buffer cache (default: 1)
-- vacuum_cost_page_miss: ราคา miss (read from disk, default: 2)
-- vacuum_cost_page_dirty: ราคา dirty page (default: 20)

SHOW vacuum_cost_page_hit;   -- 1
SHOW vacuum_cost_page_miss;  -- 2
SHOW vacuum_cost_page_dirty; -- 20

-- เมื่อ accumulated cost เกิน vacuum_cost_limit:
-- autovacuum หยุดพัก autovacuum_vacuum_cost_delay ms
SHOW vacuum_cost_limit;            -- 200 (default)
SHOW autovacuum_vacuum_cost_delay; -- 2ms (default)

-- ปรับ throughput ของ autovacuum
-- เพิ่ม limit หรือลด delay = เร็วขึ้น แต่ใช้ IO มากขึ้น
ALTER SYSTEM SET autovacuum_vacuum_cost_limit = 400;  -- Double throughput
ALTER SYSTEM SET autovacuum_vacuum_cost_delay = 10;   -- Slower

-- สำหรับ maintenance window: ปิด throttling
-- (ทำชั่วคราวเท่านั้น)
SET vacuum_cost_delay = 0;
VACUUM VERBOSE orders;
RESET vacuum_cost_delay;
```

---

## Monitoring Bloat

### pgstattuple Extension

```sql
-- ติดตั้ง
CREATE EXTENSION pgstattuple;

-- ดู table bloat ละเอียด
SELECT * FROM pgstattuple('orders');
-- tuple_count: live tuples
-- tuple_len: total live tuple bytes
-- tuple_percent: % ของ total size
-- dead_tuple_count: dead tuples
-- dead_tuple_len: dead tuple bytes
-- dead_tuple_percent: % ของ total size
-- free_space: unused space in pages
-- free_percent: % ของ total size

-- ดู index bloat
SELECT * FROM pgstatindex('idx_orders_customer');
-- version: B-tree version
-- tree_level: tree depth
-- index_size: total size
-- root_block_no: root block
-- internal_pages: non-leaf pages
-- leaf_pages: leaf pages
-- empty_pages: empty pages
-- deleted_pages: deleted pages
-- avg_leaf_density: average fullness
-- leaf_fragmentation: % fragmentation

-- Script ตรวจ bloat ทุก table
DO $$
DECLARE
    r RECORD;
    stats RECORD;
BEGIN
    FOR r IN 
        SELECT schemaname, tablename 
        FROM pg_tables 
        WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
    LOOP
        SELECT dead_tuple_percent, free_percent 
        INTO stats
        FROM pgstattuple(r.schemaname || '.' || r.tablename);
        
        IF stats.dead_tuple_percent > 20 OR stats.free_percent > 30 THEN
            RAISE NOTICE 'BLOAT ALERT: %.% dead=%, free=%', 
                r.schemaname, r.tablename, 
                stats.dead_tuple_percent, stats.free_percent;
        END IF;
    END LOOP;
END $$;
```

### Bloat Query ด้วย pg_stat_user_tables

```sql
-- Query ประเมิน bloat โดยไม่ต้องใช้ extension
-- (ประมาณการ ไม่แม่นยำ 100%)
WITH table_stats AS (
    SELECT
        schemaname,
        tablename,
        pg_relation_size(schemaname || '.' || tablename) AS table_bytes,
        n_live_tup,
        n_dead_tup,
        n_tup_ins,
        n_tup_upd,
        n_tup_del,
        last_autovacuum,
        last_vacuum
    FROM pg_stat_user_tables
    WHERE n_live_tup > 0
),
bloat_estimate AS (
    SELECT
        schemaname,
        tablename,
        pg_size_pretty(table_bytes) AS table_size,
        n_live_tup,
        n_dead_tup,
        ROUND(100.0 * n_dead_tup / (n_live_tup + n_dead_tup), 2) AS dead_pct,
        COALESCE(last_autovacuum::text, 'Never') AS last_autovacuum,
        COALESCE(last_vacuum::text, 'Never') AS last_vacuum,
        -- Estimate: total modifications since last analyze
        n_tup_ins + n_tup_upd + n_tup_del AS total_modifications
    FROM table_stats
)
SELECT *
FROM bloat_estimate
WHERE dead_pct > 10
   OR total_modifications > n_live_tup * 0.3
ORDER BY dead_pct DESC
LIMIT 30;
```

### pgbloat_check: External Tool

```bash
# ติดตั้ง pgbloat_check
pip install pgbloat_check

# หรือใช้ pg_bloat_check script
wget https://github.com/keithf4/pg_bloat_check/raw/master/pg_bloat_check.py

# Run bloat check
python3 pg_bloat_check.py \
    -c "host=localhost dbname=mydb user=postgres" \
    --min_wasted_size 100 \  # MB
    --bloat_factor 2         # 2x bloat

# Output จะแสดง tables และ indexes ที่ bloated
```

---

## Table Bloat Cleanup

### VACUUM FULL: หลีกเลี่ยงใน Production

```sql
-- VACUUM FULL: เขียน table ใหม่ทั้งหมด
-- ⚠️ LOCK: ACCESS EXCLUSIVE = ไม่มีใคร query ได้
-- ✓ คืน disk space กลับ OS
-- ✓ Defragment table

-- ดู lock impact
-- ก่อน VACUUM FULL ต้องตรวจสอบ:
SELECT 
    pid, 
    usename, 
    application_name,
    state,
    query
FROM pg_stat_activity
WHERE datname = current_database()
    AND state != 'idle';

-- Run (เฉพาะ maintenance window!)
\timing
VACUUM FULL VERBOSE orders;
-- Time: อาจนานหลายนาทีถึงหลายชั่วโมง

-- หลัง VACUUM FULL ควร REINDEX ด้วย
REINDEX TABLE orders;
```

### pg_repack: Online Table Repack

```sql
-- pg_repack: reorganize tables ออนไลน์ (ไม่ต้อง lock นาน)

-- ติดตั้ง
-- Ubuntu: sudo apt-get install postgresql-14-repack
CREATE EXTENSION pg_repack;

-- Repack table ออนไลน์
-- (ใช้ trigger-based approach, lock ช่วงสั้นๆ ตอนสุดท้าย)
\! pg_repack -h localhost -d mydb -t orders

-- Repack พร้อมกำหนด order
\! pg_repack -h localhost -d mydb -t orders --order-by order_date

-- Repack schema ทั้งหมด
\! pg_repack -h localhost -d mydb -s public

-- ดู progress ระหว่าง repack
SELECT * FROM pg_stat_progress_cluster;
```

### Partitioning: ป้องกัน Bloat ใน Long-term

```sql
-- สร้าง partitioned table
CREATE TABLE orders (
    id BIGSERIAL,
    customer_id INT NOT NULL,
    order_date DATE NOT NULL,
    total DECIMAL(12,2),
    status VARCHAR(20)
) PARTITION BY RANGE (order_date);

-- สร้าง partitions รายเดือน
CREATE TABLE orders_2024_01 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
    
CREATE TABLE orders_2024_02 PARTITION OF orders
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- Automate partition creation
CREATE OR REPLACE FUNCTION create_monthly_partition(
    parent_table TEXT,
    year INT,
    month INT
) RETURNS void AS $$
DECLARE
    partition_name TEXT;
    start_date DATE;
    end_date DATE;
BEGIN
    partition_name := parent_table || '_' || year || '_' || LPAD(month::TEXT, 2, '0');
    start_date := DATE(year || '-' || month || '-01');
    end_date := start_date + INTERVAL '1 month';
    
    EXECUTE format(
        'CREATE TABLE IF NOT EXISTS %I PARTITION OF %I FOR VALUES FROM (%L) TO (%L)',
        partition_name, parent_table, start_date, end_date
    );
    
    RAISE NOTICE 'Created partition: %', partition_name;
END;
$$ LANGUAGE plpgsql;

-- สร้าง partitions สำหรับ 12 เดือน
DO $$
BEGIN
    FOR month IN 1..12 LOOP
        PERFORM create_monthly_partition('orders', 2024, month);
    END LOOP;
END $$;

-- Detach old partitions (แทน VACUUM FULL!)
-- ไม่ต้อง vacuum เพราะ detach แล้วลบทิ้ง
ALTER TABLE orders DETACH PARTITION orders_2023_01;
DROP TABLE orders_2023_01;  -- ลบ partition เก่าได้เลย ไม่ต้อง VACUUM
```

---

## Index Bloat: REINDEX CONCURRENTLY

```sql
-- ดู index bloat
SELECT 
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexname::regclass)) as index_size,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexname::regclass) DESC
LIMIT 20;

-- ดูด้วย pgstatindex (ต้องมี extension)
SELECT 
    indexname,
    pg_size_pretty(pg_relation_size(indexname::regclass)) as size,
    (SELECT leaf_fragmentation 
     FROM pgstatindex(indexname::regclass::text)) as fragmentation
FROM pg_stat_user_indexes
WHERE schemaname = 'public'
ORDER BY fragmentation DESC;

-- REINDEX: rebuild index (ล็อก table)
REINDEX INDEX idx_orders_customer;
REINDEX TABLE orders;  -- rebuild ทุก index ในตาราง

-- REINDEX CONCURRENTLY: ไม่ล็อก (PostgreSQL 12+)
REINDEX INDEX CONCURRENTLY idx_orders_customer;
REINDEX TABLE CONCURRENTLY orders;
REINDEX SCHEMA CONCURRENTLY public;
REINDEX DATABASE CONCURRENTLY mydb;

-- Monitor progress
SELECT 
    phase,
    blocks_done,
    blocks_total,
    ROUND(100.0 * blocks_done / NULLIF(blocks_total, 0), 2) as pct_done
FROM pg_stat_progress_create_index;
```

---

## Visibility Map และ Free Space Map

### Visibility Map (VM)

```sql
-- VM เก็บข้อมูล 2 bits ต่อ page:
-- Bit 1 (all-visible): ทุก tuple ใน page visible ต่อทุก transaction
-- Bit 2 (all-frozen): ทุก tuple ใน page frozen แล้ว

-- ดู visibility map statistics
SELECT 
    relname,
    relallvisible as all_visible_pages,
    relpages as total_pages,
    ROUND(100.0 * relallvisible / NULLIF(relpages, 0), 2) as vm_coverage_pct
FROM pg_class
WHERE relkind = 'r'
    AND relpages > 0
ORDER BY vm_coverage_pct ASC
LIMIT 20;

-- VM ช่วยประหยัด:
-- 1. Index Only Scan: ถ้า page all-visible, ไม่ต้องอ่าน heap
-- 2. VACUUM: ถ้า page all-visible, ข้ามได้
-- 3. FREEZE: ถ้า page all-frozen, ไม่ต้อง freeze อีก

-- VACUUM จะ set all-visible bits
-- หลัง VACUUM ควรเห็น relallvisible เพิ่มขึ้น
```

### Free Space Map (FSM)

```sql
-- FSM เก็บข้อมูล free space ใน pages เพื่อ reuse
-- ใช้สำหรับ INSERT ที่จะ reclaim deleted space

-- ดูว่า FSM ทำงานหรือไม่
-- ถ้า VACUUM ทำงานสม่ำเสมอ FSM จะอัปเดต
-- ถ้า FSM เต็ม: tables จะใช้ new pages แทน free pages

-- ดู FSM information ด้วย pageinspect
CREATE EXTENSION pageinspect;

-- ดู free space ใน specific page
SELECT * FROM pg_freespace('orders', 0);

-- ดู average free space per page
SELECT 
    avg(avail) as avg_free_bytes,
    count(*) as page_count,
    sum(avail) as total_free_bytes,
    pg_size_pretty(sum(avail)::bigint) as total_free_size
FROM pg_freespace('orders');
```

---

## pg_stat_user_tables: Monitoring Vacuum Activity

```sql
-- Query สำหรับ vacuum monitoring dashboard
SELECT 
    schemaname,
    tablename,
    n_live_tup,
    n_dead_tup,
    ROUND(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) as dead_pct,
    n_tup_ins as inserts,
    n_tup_upd as updates,
    n_tup_del as deletes,
    n_tup_hot_upd as hot_updates,
    ROUND(100.0 * n_tup_hot_upd / NULLIF(n_tup_upd, 0), 2) as hot_upd_pct,
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze,
    vacuum_count,
    autovacuum_count,
    analyze_count,
    autoanalyze_count
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;

-- ตรวจสอบ autovacuum ที่ทำงานนานเกิน
SELECT 
    pid,
    query,
    query_start,
    NOW() - query_start as running_time
FROM pg_stat_activity
WHERE query LIKE 'autovacuum:%'
ORDER BY query_start;

-- Tables ที่ไม่ถูก vacuum นาน
SELECT 
    schemaname,
    tablename,
    COALESCE(last_autovacuum, last_vacuum) as last_vacuumed,
    NOW() - COALESCE(last_autovacuum, last_vacuum) as time_since_vacuum,
    n_dead_tup,
    n_live_tup
FROM pg_stat_user_tables
WHERE COALESCE(last_autovacuum, last_vacuum) < NOW() - INTERVAL '7 days'
    OR (last_autovacuum IS NULL AND last_vacuum IS NULL)
ORDER BY time_since_vacuum DESC NULLS FIRST;
```

---

## HOT Updates: Heap-Only Tuple

```sql
-- HOT (Heap-Only Tuple) Update: optimization สำคัญ
-- เมื่อ UPDATE ไม่เปลี่ยน indexed columns:
-- - สร้าง new tuple ใน same page
-- - ไม่ต้องอัปเดต indexes
-- - VACUUM ง่ายกว่า

-- ดู HOT update ratio
SELECT 
    tablename,
    n_tup_upd as total_updates,
    n_tup_hot_upd as hot_updates,
    ROUND(100.0 * n_tup_hot_upd / NULLIF(n_tup_upd, 0), 2) as hot_pct
FROM pg_stat_user_tables
WHERE n_tup_upd > 0
ORDER BY hot_pct ASC;  -- HOT pct ต่ำ = มี index updates มาก

-- HOT ทำงานได้เมื่อ:
-- 1. Updated columns ไม่มี index
-- 2. New tuple อยู่ใน same page ได้ (fillfactor มีผล)
-- 3. ไม่มี HOT chain ยาวเกิน

-- ปรับ fillfactor เพื่อเพิ่ม HOT updates
-- ทิ้ง free space ใน page สำหรับ HOT updates
ALTER TABLE orders SET (fillfactor = 80);  -- 20% space สำหรับ updates

-- หลังเปลี่ยน fillfactor ต้อง VACUUM FULL หรือ CLUSTER
VACUUM FULL orders;

-- ดูผลลัพธ์: hot_pct ควรเพิ่มขึ้น
```

---

## Autovacuum Logging และ Monitoring

```sql
-- Enable autovacuum logging
ALTER SYSTEM SET log_autovacuum_min_duration = '1s';  -- Log vacuum ที่ใช้ > 1 วินาที
SELECT pg_reload_conf();

-- ดู log entries (ใน PostgreSQL log)
-- 2024-01-01 12:00:00 UTC [12345] LOG: automatic vacuum of table "mydb.public.orders":
--   index scans: 2
--   pages: 0 removed, 5000 remain, 100 skipped due to pins
--   tuples: 25000 removed, 975000 remain, 0 are dead but not yet removable
--   buffer usage: 6234 hits, 123 misses, 456 dirtied
--   avg read rate: 12.345 MB/s, avg write rate: 5.678 MB/s
--   system usage: CPU: user: 0.12 s, system: 0.03 s, elapsed: 2.34 s

-- Script parse autovacuum log
```

```python
#!/usr/bin/env python3
"""Parse PostgreSQL autovacuum log entries"""

import re
import subprocess
from datetime import datetime

def parse_autovacuum_log(log_file: str, hours: int = 24):
    """Parse autovacuum entries จาก PostgreSQL log"""
    
    pattern = re.compile(
        r'(\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2})'  # timestamp
        r'.*automatic vacuum of table "([^"]+)"'      # table name
    )
    
    stats_pattern = re.compile(
        r'tuples: (\d+) removed, (\d+) remain'
    )
    
    time_pattern = re.compile(
        r'elapsed: ([\d.]+) s'
    )
    
    results = []
    
    with open(log_file, 'r') as f:
        current_entry = None
        
        for line in f:
            # New autovacuum entry
            m = pattern.search(line)
            if m:
                current_entry = {
                    'timestamp': m.group(1),
                    'table': m.group(2),
                    'removed': 0,
                    'remain': 0,
                    'elapsed': 0.0
                }
                results.append(current_entry)
            
            # Stats line
            elif current_entry:
                m = stats_pattern.search(line)
                if m:
                    current_entry['removed'] = int(m.group(1))
                    current_entry['remain'] = int(m.group(2))
                
                m = time_pattern.search(line)
                if m:
                    current_entry['elapsed'] = float(m.group(1))
                    current_entry = None
    
    # Sort by elapsed time
    results.sort(key=lambda x: x['elapsed'], reverse=True)
    
    print(f"Top 10 longest autovacuum operations:")
    print(f"{'Table':<50} {'Removed':>10} {'Elapsed':>10}")
    print("-" * 75)
    
    for entry in results[:10]:
        print(f"{entry['table']:<50} {entry['removed']:>10,} {entry['elapsed']:>9.2f}s")

if __name__ == '__main__':
    parse_autovacuum_log('/var/log/postgresql/postgresql-2024-01-01.log')
```

---

## Advanced: Vacuum Parameters ทั้งหมด

```sql
-- Global vacuum parameters
SHOW vacuum_cost_delay;             -- Throttle: ms to sleep
SHOW vacuum_cost_page_hit;          -- Cost: buffer hit
SHOW vacuum_cost_page_miss;         -- Cost: disk read
SHOW vacuum_cost_page_dirty;        -- Cost: dirty page
SHOW vacuum_cost_limit;             -- Max cost before sleep
SHOW vacuum_freeze_min_age;         -- Min age before freeze
SHOW vacuum_freeze_table_age;       -- Force vacuum whole table
SHOW vacuum_multixact_freeze_min_age;
SHOW vacuum_multixact_freeze_table_age;
SHOW vacuum_failsafe_age;           -- Emergency vacuum trigger

-- Autovacuum global
SHOW autovacuum;
SHOW autovacuum_max_workers;
SHOW autovacuum_naptime;
SHOW autovacuum_vacuum_threshold;
SHOW autovacuum_vacuum_insert_threshold;
SHOW autovacuum_analyze_threshold;
SHOW autovacuum_vacuum_scale_factor;
SHOW autovacuum_vacuum_insert_scale_factor;
SHOW autovacuum_analyze_scale_factor;
SHOW autovacuum_freeze_max_age;
SHOW autovacuum_multixact_freeze_max_age;
SHOW autovacuum_vacuum_cost_delay;
SHOW autovacuum_vacuum_cost_limit;
```

### postgresql.conf สำหรับ High-Traffic System

```ini
# postgresql.conf - Vacuum tuning สำหรับ high-traffic

# ============ Autovacuum Workers ============
autovacuum_max_workers = 5           # เพิ่มจาก 3
autovacuum_naptime = 30s             # ตรวจสอบทุก 30 วินาที (เร็วขึ้น)

# ============ Vacuum Threshold ============
# ลด scale_factor สำหรับ large tables
autovacuum_vacuum_scale_factor = 0.05   # 5% แทน 20%
autovacuum_vacuum_threshold = 1000      # rows
autovacuum_analyze_scale_factor = 0.02  # 2% 
autovacuum_analyze_threshold = 500

# ============ Vacuum Performance ============
# เพิ่ม throughput ของ autovacuum
autovacuum_vacuum_cost_limit = 400      # เพิ่มจาก 200
autovacuum_vacuum_cost_delay = 10ms     # ลดความถี่ sleep

# ============ Freeze Settings ============
vacuum_freeze_min_age = 50000000        # 50M transactions
vacuum_freeze_table_age = 150000000     # 150M transactions
autovacuum_freeze_max_age = 200000000   # 200M transactions
vacuum_failsafe_age = 1600000000        # 1.6B transactions

# ============ Logging ============
log_autovacuum_min_duration = 1000      # Log vacuum > 1 second
```

---

## Vacuum Monitoring Script (Production Use)

```sql
-- Comprehensive vacuum monitoring query
WITH vacuum_stats AS (
    SELECT
        n.nspname AS schema,
        c.relname AS table,
        c.reltuples::bigint AS estimated_rows,
        pg_stat_get_live_tuples(c.oid) AS live_tuples,
        pg_stat_get_dead_tuples(c.oid) AS dead_tuples,
        pg_stat_get_vacuum_count(c.oid) AS vacuum_count,
        pg_stat_get_autovacuum_count(c.oid) AS autovacuum_count,
        pg_stat_get_last_vacuum_time(c.oid) AS last_vacuum,
        pg_stat_get_last_autovacuum_time(c.oid) AS last_autovacuum,
        age(c.relfrozenxid) AS xid_age,
        current_setting('autovacuum_freeze_max_age')::bigint AS freeze_max_age
    FROM pg_class c
    JOIN pg_namespace n ON n.oid = c.relnamespace
    WHERE c.relkind = 'r'
        AND n.nspname NOT IN ('pg_catalog', 'information_schema', 'pg_toast')
)
SELECT
    schema,
    table,
    live_tuples,
    dead_tuples,
    ROUND(100.0 * dead_tuples / NULLIF(live_tuples + dead_tuples, 0), 2) AS dead_pct,
    vacuum_count + autovacuum_count AS total_vacuums,
    COALESCE(last_autovacuum::text, last_vacuum::text, 'Never') AS last_vacuumed,
    xid_age,
    ROUND(100.0 * xid_age / freeze_max_age, 2) AS freeze_pct,
    CASE
        WHEN xid_age > 1800000000 THEN 'CRITICAL - Freeze needed NOW!'
        WHEN xid_age > freeze_max_age * 0.9 THEN 'WARNING - Freeze needed soon'
        WHEN dead_pct > 30 THEN 'WARNING - High bloat'
        WHEN dead_pct > 10 THEN 'WATCH - Moderate bloat'
        ELSE 'OK'
    END AS status
FROM vacuum_stats
WHERE live_tuples + dead_tuples > 1000  -- ไม่แสดง empty tables
ORDER BY 
    CASE WHEN xid_age > 1800000000 THEN 0
         WHEN dead_pct > 30 THEN 1
         ELSE 2
    END,
    dead_pct DESC;
```

```bash
#!/bin/bash
# autovacuum_monitor.sh - script monitoring autovacuum

DB_HOST=${DB_HOST:-localhost}
DB_PORT=${DB_PORT:-5432}
DB_NAME=${DB_NAME:-mydb}
DB_USER=${DB_USER:-postgres}
ALERT_EMAIL=${ALERT_EMAIL:-dba@example.com}

# Check for tables with high bloat
HIGH_BLOAT=$(psql -h $DB_HOST -p $DB_PORT -U $DB_USER $DB_NAME -t -c "
SELECT COUNT(*)
FROM pg_stat_user_tables
WHERE 100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0) > 30
    AND n_live_tup + n_dead_tup > 10000;
")

# Check for XID wraparound risk
HIGH_XID=$(psql -h $DB_HOST -p $DB_PORT -U $DB_USER $DB_NAME -t -c "
SELECT COUNT(*)
FROM pg_database
WHERE age(datfrozenxid) > 1500000000;
")

if [ "$HIGH_BLOAT" -gt "0" ] || [ "$HIGH_XID" -gt "0" ]; then
    echo "VACUUM ALERT: $HIGH_BLOAT tables with high bloat, $HIGH_XID DBs with high XID age" | \
        mail -s "PostgreSQL Vacuum Alert" $ALERT_EMAIL
    
    echo "Details:"
    psql -h $DB_HOST -p $DB_PORT -U $DB_USER $DB_NAME -c "
    SELECT schemaname, tablename, 
           n_dead_tup, 
           ROUND(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) as dead_pct
    FROM pg_stat_user_tables
    WHERE 100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0) > 30
        AND n_live_tup + n_dead_tup > 10000
    ORDER BY dead_pct DESC;"
fi

echo "Vacuum monitoring completed at $(date)"
```

---

## สรุป Best Practices

```
Autovacuum Tuning Guidelines:
================================
1. อย่าปิด autovacuum เด็ดขาด
2. ปรับ scale_factor สำหรับ large tables:
   - tables > 10M rows: scale_factor = 0.01-0.05
   - high-update tables: scale_factor = 0.01
   
3. Monitor XID age สม่ำเสมอ:
   - age > 200M: autovacuum จะ force vacuum
   - age > 1.5B: เริ่ม concern
   - age > 1.8B: emergency action needed
   
4. ใช้ pg_repack แทน VACUUM FULL:
   - ลด downtime
   - Table ยังใช้งานได้ระหว่าง repack
   
5. ตั้ง log_autovacuum_min_duration = 1s:
   - Monitor vacuum ที่ใช้เวลานาน
   - Tune parameters ตาม log
   
6. Per-table autovacuum settings:
   - ใช้สำหรับ tables ที่มี workload พิเศษ
   - ไม่เปลี่ยน global settings เพื่อ specific table
   
7. Partitioning ป้องกัน bloat ใน large tables:
   - Detach + Drop old partitions
   - ไม่ต้อง vacuum partition ที่ลบทิ้ง
```

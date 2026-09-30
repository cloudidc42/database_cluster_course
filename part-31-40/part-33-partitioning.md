# Part 33: Partitioning ใน PostgreSQL

## บทนำ

Table Partitioning คือการแบ่ง table ขนาดใหญ่ออกเป็น partitions (ส่วนย่อย) ที่เล็กกว่า ซึ่งจัดการได้ง่ายกว่า PostgreSQL รองรับ Declarative Partitioning ตั้งแต่ version 10 และมีการพัฒนาต่อเนื่องจนถึง version 16

---

## 1. ทำไมต้องใช้ Partitioning

### ปัญหาของ Table ขนาดใหญ่

```sql
-- Table ที่มีข้อมูล 100M+ rows
SELECT COUNT(*) FROM events;  -- 500,000,000 rows!

-- Query ช้ามาก แม้มี index
EXPLAIN ANALYZE
SELECT * FROM events 
WHERE event_date BETWEEN '2024-01-01' AND '2024-01-31';
-- Seq Scan ใช้เวลา 45 seconds
-- แม้มี B-tree index บน event_date ก็ยังช้า

-- ปัญหาอื่นๆ:
-- 1. VACUUM ช้า - ต้อง scan ทั้ง table
-- 2. Index build ช้า
-- 3. Archive ข้อมูลเก่าทำได้ยาก
-- 4. ไม่สามารถ drop data range ได้ง่าย
```

### ประโยชน์ของ Partitioning

```
1. Partition Pruning: Query ไปเฉพาะ partition ที่เกี่ยวข้อง
2. ลบข้อมูลเก่าง่าย: DROP TABLE partition แทน DELETE
3. VACUUM/ANALYZE เร็วขึ้น: ทำทีละ partition
4. Parallel Query: แต่ละ partition ทำงาน parallel
5. Archive: DETACH partition เก่าออก
6. Index ที่เล็กกว่า: index ต่อ partition แทน global index
7. I/O: partition ต่างๆ อยู่บน tablespace ต่างกันได้
```

---

## 2. Partition Types

### 2.1 Range Partitioning

เหมาะสำหรับ continuous ranges เช่น date, number

```sql
-- Range Partitioning by date
CREATE TABLE events (
    id BIGSERIAL,
    event_type TEXT NOT NULL,
    event_data JSONB,
    user_id INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (created_at);

-- สร้าง partitions
CREATE TABLE events_2024_01 PARTITION OF events
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE events_2024_02 PARTITION OF events
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

CREATE TABLE events_2024_03 PARTITION OF events
    FOR VALUES FROM ('2024-03-01') TO ('2024-04-01');

-- Range ไม่รวม upper bound
-- events_2024_01: 2024-01-01 <= x < 2024-02-01

-- Range Partitioning by number
CREATE TABLE orders (
    id BIGSERIAL,
    order_number TEXT,
    amount NUMERIC(12,2),
    created_at TIMESTAMPTZ
) PARTITION BY RANGE (id);

CREATE TABLE orders_1_1000000 PARTITION OF orders
    FOR VALUES FROM (1) TO (1000001);

CREATE TABLE orders_1000001_2000000 PARTITION OF orders
    FOR VALUES FROM (1000001) TO (2000001);
```

### 2.2 List Partitioning

เหมาะสำหรับ discrete values เช่น country, status, region

```sql
-- List Partitioning by country/region
CREATE TABLE customers (
    id SERIAL,
    name TEXT NOT NULL,
    email TEXT,
    country_code TEXT NOT NULL,
    region TEXT NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
) PARTITION BY LIST (country_code);

CREATE TABLE customers_th PARTITION OF customers
    FOR VALUES IN ('TH');

CREATE TABLE customers_sg PARTITION OF customers
    FOR VALUES IN ('SG');

CREATE TABLE customers_my PARTITION OF customers
    FOR VALUES IN ('MY');

CREATE TABLE customers_asia_other PARTITION OF customers
    FOR VALUES IN ('PH', 'ID', 'VN', 'MM', 'KH', 'LA');

CREATE TABLE customers_other PARTITION OF customers DEFAULT;

-- List Partitioning by status
CREATE TABLE support_tickets (
    id SERIAL,
    subject TEXT,
    status TEXT NOT NULL,
    priority TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
) PARTITION BY LIST (status);

CREATE TABLE tickets_open PARTITION OF support_tickets
    FOR VALUES IN ('new', 'open', 'pending');

CREATE TABLE tickets_closed PARTITION OF support_tickets
    FOR VALUES IN ('resolved', 'closed', 'cancelled');
```

### 2.3 Hash Partitioning

เหมาะสำหรับ distribute data evenly โดยไม่มี natural partition key

```sql
-- Hash Partitioning - distribute evenly
CREATE TABLE user_sessions (
    id BIGSERIAL,
    user_id INTEGER NOT NULL,
    token TEXT NOT NULL,
    data JSONB,
    expires_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW()
) PARTITION BY HASH (user_id);

-- สร้าง 8 partitions
CREATE TABLE user_sessions_0 PARTITION OF user_sessions
    FOR VALUES WITH (MODULUS 8, REMAINDER 0);

CREATE TABLE user_sessions_1 PARTITION OF user_sessions
    FOR VALUES WITH (MODULUS 8, REMAINDER 1);

CREATE TABLE user_sessions_2 PARTITION OF user_sessions
    FOR VALUES WITH (MODULUS 8, REMAINDER 2);

CREATE TABLE user_sessions_3 PARTITION OF user_sessions
    FOR VALUES WITH (MODULUS 8, REMAINDER 3);

CREATE TABLE user_sessions_4 PARTITION OF user_sessions
    FOR VALUES WITH (MODULUS 8, REMAINDER 4);

CREATE TABLE user_sessions_5 PARTITION OF user_sessions
    FOR VALUES WITH (MODULUS 8, REMAINDER 5);

CREATE TABLE user_sessions_6 PARTITION OF user_sessions
    FOR VALUES WITH (MODULUS 8, REMAINDER 6);

CREATE TABLE user_sessions_7 PARTITION OF user_sessions
    FOR VALUES WITH (MODULUS 8, REMAINDER 7);

-- Insert routing อัตโนมัติตาม hash(user_id) % 8
INSERT INTO user_sessions (user_id, token) VALUES (12345, 'abc123');
-- ไปที่ partition ที่ 12345 % 8 = partition_5
```

---

## 3. Declarative Partitioning Syntax

```sql
-- Syntax ทั่วไป
CREATE TABLE parent_table (
    ...columns...
) PARTITION BY { RANGE | LIST | HASH } (partition_key);

CREATE TABLE child_partition PARTITION OF parent_table
    FOR VALUES ...;

-- Range:
FOR VALUES FROM (lower) TO (upper)  -- upper ไม่รวม

-- List:
FOR VALUES IN (value1, value2, ...)

-- Hash:
FOR VALUES WITH (MODULUS n, REMAINDER r)

-- Default partition (รับทุก row ที่ไม่ match):
CREATE TABLE parent_default PARTITION OF parent_table DEFAULT;
```

---

## 4. Partition Pruning

PostgreSQL จะ eliminate partitions ที่ไม่เกี่ยวข้องโดยอัตโนมัติ

```sql
-- ดู execution plan
EXPLAIN (ANALYZE, BUFFERS)
SELECT COUNT(*) FROM events 
WHERE created_at BETWEEN '2024-01-01' AND '2024-01-31';

-- โดยไม่มี partition (500M rows):
-- Seq Scan on events  (cost=0.00..8500000.00 rows=8000000 width=8)
--   Filter: (created_at >= '2024-01-01' AND created_at <= '2024-01-31')

-- มี partitioning (pruning):
-- Append  (cost=0.00..120000.00 rows=1000000 width=8)
--   ->  Seq Scan on events_2024_01  (cost=0.00..120000.00 rows=1000000 width=8)
-- (scan เฉพาะ partition ที่ relevant)

-- Enable/Disable partition pruning
SET enable_partition_pruning = on;  -- default on
SHOW enable_partition_pruning;

-- Runtime pruning (query parameter)
PREPARE q AS SELECT * FROM events WHERE created_at = $1;
EXECUTE q('2024-01-15');
-- PostgreSQL จะ prune ตอน execute
```

### Partition Elimination ใน Queries

```sql
-- Query ที่ใช้ partition key ใน WHERE
SELECT * FROM events 
WHERE created_at >= '2024-03-01' AND created_at < '2024-04-01';
-- Scans: events_2024_03 only

-- Query ที่ใช้ IN list
SELECT * FROM customers WHERE country_code IN ('TH', 'SG');
-- Scans: customers_th, customers_sg only

-- Query ที่ไม่ระบุ partition key - scan ทุก partition
SELECT * FROM events WHERE user_id = 12345;
-- Scans: all partitions (no pruning)

-- ดู partitions ที่ถูก scan
EXPLAIN SELECT * FROM events WHERE created_at = '2024-01-15';
```

---

## 5. Indexes บน Partitioned Tables

```sql
-- สร้าง index บน partitioned table (สร้างอัตโนมัติบนทุก partition)
CREATE INDEX idx_events_user_id ON events (user_id);
-- สร้าง index บน events_2024_01, events_2024_02, ... ทุก partition

CREATE INDEX idx_events_type ON events (event_type) WHERE event_type != 'debug';

-- สร้าง unique index (ต้องรวม partition key)
CREATE UNIQUE INDEX idx_events_unique 
ON events (id, created_at);  -- ต้องรวม created_at (partition key)

-- Primary key บน partitioned table ต้องรวม partition key
CREATE TABLE logs (
    id BIGSERIAL,
    log_date DATE NOT NULL,
    message TEXT
) PARTITION BY RANGE (log_date);

-- Primary key ต้องรวม partition key
ALTER TABLE logs ADD PRIMARY KEY (id, log_date);
-- หรือ
CREATE UNIQUE INDEX ON logs (id, log_date);

-- ดู indexes บน partitions
SELECT 
    schemaname,
    tablename,
    indexname,
    indexdef
FROM pg_indexes
WHERE tablename LIKE 'events%'
ORDER BY tablename;
```

---

## 6. Foreign Keys และ Partitioned Tables

```sql
-- Foreign key TO partitioned table มีข้อจำกัด
-- FK จาก parent ไป partition ไม่รองรับโดยตรง

-- แนะนำ: FK ไป regular table แทน
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name TEXT
);

-- events FK ไป users ได้ปกติ
CREATE TABLE events (
    id BIGSERIAL,
    user_id INTEGER NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (created_at);

-- แต่ไม่สามารถ FK จาก regular table ไป partitioned table ได้
-- (PostgreSQL 16 ยังไม่รองรับ FK pointing to partitioned table)

-- Workaround: ใช้ trigger แทน FK
CREATE OR REPLACE FUNCTION check_user_exists()
RETURNS TRIGGER AS $$
BEGIN
    IF NOT EXISTS (SELECT 1 FROM users WHERE id = NEW.user_id) THEN
        RAISE EXCEPTION 'User % does not exist', NEW.user_id;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

---

## 7. Default Partition

```sql
-- Default partition รับ rows ที่ไม่ match partition ไหนเลย
CREATE TABLE events (
    id BIGSERIAL,
    event_type TEXT,
    created_at TIMESTAMPTZ NOT NULL
) PARTITION BY RANGE (created_at);

CREATE TABLE events_2024_01 PARTITION OF events
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

-- Default partition สำหรับ dates ที่ไม่มี specific partition
CREATE TABLE events_default PARTITION OF events DEFAULT;

-- หากไม่มี default partition และ insert ค่าที่ไม่ match → ERROR
INSERT INTO events (created_at) VALUES ('2099-01-01');
-- ERROR ถ้าไม่มี default partition

-- ดู rows ใน default partition
SELECT COUNT(*) FROM events_default;

-- ย้าย rows จาก default ไป specific partition (ต้องทำ manual)
-- 1. สร้าง partition ใหม่
CREATE TABLE events_2024_06 PARTITION OF events
    FOR VALUES FROM ('2024-06-01') TO ('2024-07-01');
-- ERROR ถ้า default partition มี data ใน range นั้น

-- ต้องทำแบบนี้:
BEGIN;
-- Lock default partition
LOCK TABLE events_default IN SHARE UPDATE EXCLUSIVE MODE;
-- ตรวจว่าไม่มี data ใน range ที่จะสร้าง
SELECT COUNT(*) FROM events_default 
WHERE created_at BETWEEN '2024-06-01' AND '2024-07-01';
-- ถ้า 0 ก็ create partition ได้เลย
CREATE TABLE events_2024_06 PARTITION OF events
    FOR VALUES FROM ('2024-06-01') TO ('2024-07-01');
COMMIT;
```

---

## 8. Attach/Detach Partitions

### DETACH Partition

```sql
-- Detach partition (ทำให้กลายเป็น regular table)
ALTER TABLE events DETACH PARTITION events_2024_01;

-- events_2024_01 ยังคงมีข้อมูล แต่ไม่ใช่ partition แล้ว
SELECT COUNT(*) FROM events_2024_01;  -- ยังได้

-- ใช้เพื่อ archiving: ย้ายไป cold storage
ALTER TABLE events_2024_01 SET TABLESPACE cold_storage;

-- หรือ export ออกไป
COPY events_2024_01 TO '/archive/events_2024_01.csv' CSV;

-- DETACH CONCURRENTLY (PostgreSQL 14+) - ไม่ block reads/writes
ALTER TABLE events DETACH PARTITION events_2024_01 CONCURRENTLY;
```

### ATTACH Partition

```sql
-- เตรียม regular table ที่จะ attach
CREATE TABLE events_2024_01_archive (LIKE events INCLUDING ALL);

-- Load data
COPY events_2024_01_archive FROM '/archive/events_2024_01.csv' CSV;

-- ตรวจสอบ constraint ก่อน attach
ALTER TABLE events_2024_01_archive 
ADD CONSTRAINT check_2024_01 
CHECK (created_at >= '2024-01-01' AND created_at < '2024-02-01');

-- Attach เป็น partition (PostgreSQL validates data)
ALTER TABLE events ATTACH PARTITION events_2024_01_archive
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

-- ATTACH CONCURRENTLY (PostgreSQL 17+)
-- ไม่ block ระหว่าง validation
```

---

## 9. Row Insert Routing

PostgreSQL route rows ไปยัง partition ที่ถูกต้องอัตโนมัติ

```sql
-- Insert routing อัตโนมัติ
INSERT INTO events (event_type, created_at) 
VALUES ('click', '2024-01-15 10:30:00');
-- PostgreSQL ส่งไป events_2024_01 อัตโนมัติ

INSERT INTO events (event_type, created_at) 
VALUES ('purchase', '2024-03-20 15:00:00');
-- PostgreSQL ส่งไป events_2024_03 อัตโนมัติ

-- COPY routing
COPY events FROM '/data/events.csv' CSV;
-- แต่ละ row ถูก route ไปยัง partition ที่ถูกต้อง

-- RETURNING ทำงานปกติ
INSERT INTO events (event_type, created_at) 
VALUES ('login', NOW())
RETURNING id, created_at;

-- ถ้าไม่มี partition match → ERROR
INSERT INTO events (event_type, created_at) 
VALUES ('test', '2099-01-01');
-- ERROR: no partition of relation "events" found for row
-- (ถ้าไม่มี default partition)
```

---

## 10. Cross-Partition Queries

```sql
-- Cross-partition query - PostgreSQL scan หลาย partitions
SELECT event_type, COUNT(*) 
FROM events 
WHERE created_at BETWEEN '2024-01-01' AND '2024-03-31'
GROUP BY event_type;

-- Execution plan แสดง partitions ที่ถูก scan
EXPLAIN ANALYZE
SELECT * FROM events 
WHERE created_at BETWEEN '2024-01-01' AND '2024-03-31';

-- Cross-partition JOIN
SELECT e.event_type, u.name
FROM events e
JOIN users u ON u.id = e.user_id
WHERE e.created_at >= '2024-01-01';

-- UPDATE/DELETE cross-partition
UPDATE events SET event_type = 'user_click'
WHERE event_type = 'click' 
AND created_at BETWEEN '2024-01-01' AND '2024-03-31';
-- Updates rows ใน 3 partitions

-- Aggregate cross-partition
SELECT 
    date_trunc('month', created_at) AS month,
    COUNT(*) AS event_count
FROM events
WHERE created_at >= '2024-01-01'
GROUP BY 1
ORDER BY 1;
```

---

## 11. Archiving: Detach Old Partition

```sql
-- Archiving workflow สำหรับ time-series data

-- 1. สร้าง archive schema
CREATE SCHEMA archive;

-- 2. Detach partition เก่า
ALTER TABLE events DETACH PARTITION events_2023_01;

-- 3. Move ไป archive schema
ALTER TABLE events_2023_01 SET SCHEMA archive;

-- 4. Optional: compress or move to cold storage tablespace
CREATE TABLESPACE cold_storage LOCATION '/mnt/cold-storage';
ALTER TABLE archive.events_2023_01 SET TABLESPACE cold_storage;

-- 5. สร้าง partition ใหม่สำหรับเดือนหน้า
CREATE TABLE events_2024_04 PARTITION OF events
    FOR VALUES FROM ('2024-04-01') TO ('2024-05-01');

-- Automated archiving function
CREATE OR REPLACE FUNCTION archive_old_events(
    months_to_keep INTEGER DEFAULT 12
)
RETURNS INTEGER AS $$
DECLARE
    v_partition_name TEXT;
    v_cutoff_date DATE;
    v_count INTEGER := 0;
BEGIN
    v_cutoff_date := DATE_TRUNC('month', NOW() - (months_to_keep || ' months')::INTERVAL);
    
    FOR v_partition_name IN
        SELECT child.relname
        FROM pg_inherits
        JOIN pg_class parent ON pg_inherits.inhparent = parent.oid
        JOIN pg_class child ON pg_inherits.inhrelid = child.oid
        WHERE parent.relname = 'events'
          AND child.relname LIKE 'events_%'
          AND child.relname < 'events_' || TO_CHAR(v_cutoff_date, 'YYYY_MM')
    LOOP
        EXECUTE format('ALTER TABLE events DETACH PARTITION %I', v_partition_name);
        EXECUTE format('ALTER TABLE %I SET SCHEMA archive', v_partition_name);
        v_count := v_count + 1;
        RAISE NOTICE 'Archived partition: %', v_partition_name;
    END LOOP;
    
    RETURN v_count;
END;
$$ LANGUAGE plpgsql;
```

---

## 12. Partition Maintenance Automation: pg_partman

```sql
-- ติดตั้ง pg_partman extension
-- sudo apt install postgresql-16-partman

CREATE EXTENSION pg_partman SCHEMA partman;

-- สร้าง parent table สำหรับ pg_partman
CREATE TABLE logs (
    id BIGSERIAL,
    log_level TEXT NOT NULL,
    message TEXT,
    context JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (created_at);

-- Configure pg_partman
SELECT partman.create_parent(
    p_parent_table := 'public.logs',
    p_control := 'created_at',
    p_type := 'native',
    p_interval := 'monthly',
    p_premake := 3,         -- สร้างล่วงหน้า 3 partitions
    p_start_partition := '2024-01-01'
);

-- ดู config
SELECT * FROM partman.part_config WHERE parent_table = 'public.logs';

-- Run maintenance (สร้าง partitions ใหม่, ลบเก่า)
SELECT partman.run_maintenance('public.logs');

-- ตั้ง retention policy
UPDATE partman.part_config
SET retention = '12 months',
    retention_keep_table = false,  -- drop partition (true = keep as regular table)
    infinite_time_partitions = true
WHERE parent_table = 'public.logs';

-- Setup automated maintenance via pg_cron
CREATE EXTENSION pg_cron;

SELECT cron.schedule(
    'partman-maintenance',
    '0 1 * * *',  -- every day at 1am
    $$SELECT partman.run_maintenance()$$
);

-- ดู partition info
SELECT 
    tablename,
    pg_size_pretty(pg_total_relation_size(tablename::regclass)) AS size
FROM pg_tables
WHERE tablename LIKE 'logs_%'
ORDER BY tablename;
```

---

## 13. Sub-Partitioning

```sql
-- Sub-partitioning: partition แล้ว partition อีกครั้ง
CREATE TABLE events_multi (
    id BIGSERIAL,
    country_code TEXT NOT NULL,
    event_type TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (created_at);  -- partition by month

-- สร้าง monthly partition
CREATE TABLE events_multi_2024_01 PARTITION OF events_multi
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01')
    PARTITION BY LIST (country_code);  -- sub-partition by country

-- สร้าง sub-partitions
CREATE TABLE events_multi_2024_01_th PARTITION OF events_multi_2024_01
    FOR VALUES IN ('TH');

CREATE TABLE events_multi_2024_01_sg PARTITION OF events_multi_2024_01
    FOR VALUES IN ('SG');

CREATE TABLE events_multi_2024_01_other PARTITION OF events_multi_2024_01
    DEFAULT;

-- Query routing ไปเฉพาะ sub-partition ที่เกี่ยวข้อง
SELECT * FROM events_multi
WHERE created_at = '2024-01-15' AND country_code = 'TH';
-- Scans เฉพาะ events_multi_2024_01_th
```

---

## 14. Partition-wise JOIN and Aggregate

```sql
-- Enable partition-wise features
SET enable_partitionwise_join = on;
SET enable_partitionwise_aggregate = on;

-- Partition-wise JOIN: join partitions กัน
-- (ทั้งสอง tables ต้อง partition บน key เดียวกัน)
EXPLAIN SELECT e.event_type, u.name
FROM events e
JOIN user_events u ON u.event_id = e.id AND u.created_at = e.created_at;

-- Partition-wise aggregate: aggregate ต่อ partition แล้วค่อย combine
EXPLAIN SELECT 
    date_trunc('month', created_at),
    COUNT(*)
FROM events
GROUP BY 1;
-- ถ้า enable: aggregate ในแต่ละ partition แบบ parallel

-- ตรวจสอบใน pg_settings
SELECT name, setting FROM pg_settings 
WHERE name LIKE '%partition%';
```

---

## 15. Full Working Example: Time-Series Data

### Schema Setup

```sql
-- ========================================
-- Time-Series Event Logging System
-- ========================================

-- Parent table
CREATE TABLE app_events (
    id BIGSERIAL NOT NULL,
    tenant_id INTEGER NOT NULL,
    session_id TEXT,
    event_type TEXT NOT NULL,
    event_name TEXT NOT NULL,
    properties JSONB DEFAULT '{}',
    user_id INTEGER,
    ip_address INET,
    user_agent TEXT,
    page_url TEXT,
    referrer_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (created_at);

-- สร้าง partitions สำหรับ 12 เดือน
DO $$
DECLARE
    start_date DATE := '2024-01-01';
    partition_date DATE;
    partition_name TEXT;
BEGIN
    FOR i IN 0..11 LOOP
        partition_date := start_date + (i || ' months')::INTERVAL;
        partition_name := 'app_events_' || TO_CHAR(partition_date, 'YYYY_MM');
        
        EXECUTE format(
            'CREATE TABLE %I PARTITION OF app_events 
             FOR VALUES FROM (%L) TO (%L)',
            partition_name,
            partition_date,
            partition_date + INTERVAL '1 month'
        );
        
        RAISE NOTICE 'Created partition: %', partition_name;
    END LOOP;
END;
$$;

-- Default partition
CREATE TABLE app_events_default PARTITION OF app_events DEFAULT;

-- Indexes (สร้างบน parent = สร้างบนทุก partition อัตโนมัติ)
CREATE INDEX idx_app_events_tenant_time 
ON app_events (tenant_id, created_at DESC);

CREATE INDEX idx_app_events_user 
ON app_events (user_id, created_at DESC) 
WHERE user_id IS NOT NULL;

CREATE INDEX idx_app_events_type 
ON app_events (event_type, created_at DESC);

CREATE INDEX idx_app_events_properties 
ON app_events USING GIN(properties);

-- Primary key (ต้องรวม partition key)
ALTER TABLE app_events ADD PRIMARY KEY (id, created_at);

-- Aggregated stats table (non-partitioned)
CREATE TABLE event_stats_daily (
    id SERIAL PRIMARY KEY,
    tenant_id INTEGER NOT NULL,
    stat_date DATE NOT NULL,
    event_type TEXT NOT NULL,
    event_count BIGINT DEFAULT 0,
    unique_users BIGINT DEFAULT 0,
    unique_sessions BIGINT DEFAULT 0,
    UNIQUE (tenant_id, stat_date, event_type)
);

-- Function: สร้าง partition อัตโนมัติ
CREATE OR REPLACE FUNCTION create_monthly_partition(
    p_table TEXT,
    p_year INTEGER,
    p_month INTEGER
)
RETURNS TEXT AS $$
DECLARE
    v_partition_name TEXT;
    v_start_date DATE;
    v_end_date DATE;
BEGIN
    v_start_date := DATE(p_year || '-' || LPAD(p_month::TEXT, 2, '0') || '-01');
    v_end_date := v_start_date + INTERVAL '1 month';
    v_partition_name := p_table || '_' || TO_CHAR(v_start_date, 'YYYY_MM');
    
    -- ตรวจว่า partition มีอยู่แล้วไหม
    IF NOT EXISTS (
        SELECT 1 FROM pg_class WHERE relname = v_partition_name
    ) THEN
        EXECUTE format(
            'CREATE TABLE %I PARTITION OF %I 
             FOR VALUES FROM (%L) TO (%L)',
            v_partition_name, p_table,
            v_start_date, v_end_date
        );
        RETURN 'Created: ' || v_partition_name;
    ELSE
        RETURN 'Already exists: ' || v_partition_name;
    END IF;
END;
$$ LANGUAGE plpgsql;

-- สร้าง partitions ล่วงหน้า 3 เดือน
SELECT create_monthly_partition('app_events', 
    EXTRACT(YEAR FROM NOW() + INTERVAL '1 month')::INTEGER,
    EXTRACT(MONTH FROM NOW() + INTERVAL '1 month')::INTEGER
);
SELECT create_monthly_partition('app_events', 
    EXTRACT(YEAR FROM NOW() + INTERVAL '2 months')::INTEGER,
    EXTRACT(MONTH FROM NOW() + INTERVAL '2 months')::INTEGER
);
SELECT create_monthly_partition('app_events', 
    EXTRACT(YEAR FROM NOW() + INTERVAL '3 months')::INTEGER,
    EXTRACT(MONTH FROM NOW() + INTERVAL '3 months')::INTEGER
);

-- Aggregate function
CREATE OR REPLACE FUNCTION aggregate_daily_stats(p_date DATE)
RETURNS VOID AS $$
BEGIN
    INSERT INTO event_stats_daily (tenant_id, stat_date, event_type, event_count, unique_users, unique_sessions)
    SELECT 
        tenant_id,
        p_date,
        event_type,
        COUNT(*),
        COUNT(DISTINCT user_id),
        COUNT(DISTINCT session_id)
    FROM app_events
    WHERE created_at::DATE = p_date
    GROUP BY tenant_id, event_type
    ON CONFLICT (tenant_id, stat_date, event_type)
    DO UPDATE SET
        event_count = EXCLUDED.event_count,
        unique_users = EXCLUDED.unique_users,
        unique_sessions = EXCLUDED.unique_sessions;
END;
$$ LANGUAGE plpgsql;

-- Archiving function
CREATE OR REPLACE FUNCTION archive_old_partitions(
    p_table TEXT DEFAULT 'app_events',
    p_months_to_keep INTEGER DEFAULT 6
)
RETURNS TABLE(archived_partition TEXT, row_count BIGINT) AS $$
DECLARE
    v_cutoff_date DATE;
    v_partition RECORD;
    v_count BIGINT;
BEGIN
    v_cutoff_date := DATE_TRUNC('month', NOW() - (p_months_to_keep || ' months')::INTERVAL);
    
    FOR v_partition IN
        SELECT 
            child.relname AS partition_name,
            pg_get_expr(child.relpartbound, child.oid) AS partition_bound
        FROM pg_inherits
        JOIN pg_class parent ON pg_inherits.inhparent = parent.oid
        JOIN pg_class child ON pg_inherits.inhrelid = child.oid
        WHERE parent.relname = p_table
          AND child.relname != p_table || '_default'
          AND child.relname < p_table || '_' || TO_CHAR(v_cutoff_date, 'YYYY_MM')
        ORDER BY child.relname
    LOOP
        -- Count rows
        EXECUTE format('SELECT COUNT(*) FROM %I', v_partition.partition_name) INTO v_count;
        
        -- Detach partition
        EXECUTE format('ALTER TABLE %I DETACH PARTITION %I', p_table, v_partition.partition_name);
        
        -- Move to archive schema (สร้างถ้ายังไม่มี)
        CREATE SCHEMA IF NOT EXISTS archive;
        EXECUTE format('ALTER TABLE %I SET SCHEMA archive', v_partition.partition_name);
        
        archived_partition := v_partition.partition_name;
        row_count := v_count;
        RETURN NEXT;
        
        RAISE NOTICE 'Archived % (% rows)', v_partition.partition_name, v_count;
    END LOOP;
END;
$$ LANGUAGE plpgsql;
```

### Analytics Queries

```sql
-- Query 1: Daily event counts (partition pruning)
SELECT 
    created_at::DATE AS day,
    event_type,
    COUNT(*) AS events
FROM app_events
WHERE 
    tenant_id = 1
    AND created_at BETWEEN '2024-01-01' AND '2024-01-31'
GROUP BY 1, 2
ORDER BY 1, 3 DESC;

-- Query 2: User funnel analysis
WITH funnel AS (
    SELECT 
        user_id,
        MAX(CASE WHEN event_name = 'page_view' THEN 1 END) AS viewed,
        MAX(CASE WHEN event_name = 'add_to_cart' THEN 1 END) AS added,
        MAX(CASE WHEN event_name = 'checkout_start' THEN 1 END) AS checkout,
        MAX(CASE WHEN event_name = 'purchase' THEN 1 END) AS purchased
    FROM app_events
    WHERE 
        tenant_id = 1
        AND created_at BETWEEN '2024-01-01' AND '2024-01-31'
        AND user_id IS NOT NULL
    GROUP BY user_id
)
SELECT
    COUNT(*) FILTER (WHERE viewed = 1) AS viewed_count,
    COUNT(*) FILTER (WHERE added = 1) AS added_count,
    COUNT(*) FILTER (WHERE checkout = 1) AS checkout_count,
    COUNT(*) FILTER (WHERE purchased = 1) AS purchased_count
FROM funnel;

-- Query 3: Real-time stats (เฉพาะ current partition)
SELECT 
    event_type,
    COUNT(*) AS count_today
FROM app_events
WHERE 
    tenant_id = 1
    AND created_at >= CURRENT_DATE
GROUP BY event_type
ORDER BY count_today DESC;

-- ดู partition sizes
SELECT 
    child.relname AS partition,
    pg_size_pretty(pg_total_relation_size(child.oid)) AS total_size,
    pg_size_pretty(pg_relation_size(child.oid)) AS table_size,
    pg_size_pretty(pg_total_relation_size(child.oid) - pg_relation_size(child.oid)) AS index_size
FROM pg_inherits
JOIN pg_class parent ON pg_inherits.inhparent = parent.oid
JOIN pg_class child ON pg_inherits.inhrelid = child.oid
WHERE parent.relname = 'app_events'
ORDER BY child.relname;
```

### Node.js Event Tracking

```typescript
// event-tracker.ts

import { Pool } from 'pg';

interface AppEvent {
  tenantId: number;
  sessionId?: string;
  eventType: string;
  eventName: string;
  properties?: Record<string, any>;
  userId?: number;
  ipAddress?: string;
  userAgent?: string;
  pageUrl?: string;
  referrerUrl?: string;
}

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 50,
});

// Batch insert สำหรับ high-throughput
class EventTracker {
  private buffer: AppEvent[] = [];
  private flushInterval: NodeJS.Timeout;
  private readonly BUFFER_SIZE = 100;
  private readonly FLUSH_INTERVAL_MS = 1000;

  constructor() {
    this.flushInterval = setInterval(() => this.flush(), this.FLUSH_INTERVAL_MS);
  }

  async track(event: AppEvent): Promise<void> {
    this.buffer.push(event);
    if (this.buffer.length >= this.BUFFER_SIZE) {
      await this.flush();
    }
  }

  async flush(): Promise<void> {
    if (this.buffer.length === 0) return;

    const events = this.buffer.splice(0, this.buffer.length);
    const values: any[] = [];
    const placeholders: string[] = [];

    events.forEach((e, i) => {
      const base = i * 9;
      placeholders.push(
        `($${base + 1}, $${base + 2}, $${base + 3}, $${base + 4}, $${base + 5}, $${base + 6}, $${base + 7}, $${base + 8}, $${base + 9})`
      );
      values.push(
        e.tenantId,
        e.sessionId || null,
        e.eventType,
        e.eventName,
        JSON.stringify(e.properties || {}),
        e.userId || null,
        e.ipAddress || null,
        e.userAgent || null,
        e.pageUrl || null
      );
    });

    try {
      await pool.query(
        `INSERT INTO app_events 
         (tenant_id, session_id, event_type, event_name, properties, user_id, ip_address, user_agent, page_url)
         VALUES ${placeholders.join(', ')}`,
        values
      );
    } catch (error) {
      console.error('Failed to insert events:', error);
      // Re-add to buffer หรือ log ไว้
    }
  }

  async getStats(tenantId: number, startDate: Date, endDate: Date) {
    const result = await pool.query(
      `SELECT 
         event_type,
         event_name,
         COUNT(*) AS total,
         COUNT(DISTINCT user_id) AS unique_users,
         COUNT(DISTINCT session_id) AS sessions
       FROM app_events
       WHERE 
         tenant_id = $1
         AND created_at BETWEEN $2 AND $3
       GROUP BY event_type, event_name
       ORDER BY total DESC
       LIMIT 50`,
      [tenantId, startDate, endDate]
    );
    return result.rows;
  }

  async getPartitionInfo(): Promise<any[]> {
    const result = await pool.query(`
      SELECT 
        child.relname AS partition_name,
        pg_size_pretty(pg_total_relation_size(child.oid)) AS size,
        pg_get_expr(child.relpartbound, child.oid) AS bounds
      FROM pg_inherits
      JOIN pg_class parent ON pg_inherits.inhparent = parent.oid
      JOIN pg_class child ON pg_inherits.inhrelid = child.oid
      WHERE parent.relname = 'app_events'
      ORDER BY child.relname DESC
    `);
    return result.rows;
  }

  destroy(): void {
    clearInterval(this.flushInterval);
  }
}

const tracker = new EventTracker();

// Express middleware
async function trackRequest(req: any, res: any, next: any) {
  await tracker.track({
    tenantId: req.tenantId,
    sessionId: req.sessionId,
    eventType: 'pageview',
    eventName: 'page_view',
    properties: {
      path: req.path,
      method: req.method,
      status: res.statusCode,
    },
    userId: req.userId,
    ipAddress: req.ip,
    userAgent: req.headers['user-agent'],
    pageUrl: req.url,
    referrerUrl: req.headers['referer'],
  });
  next();
}

export { EventTracker, trackRequest, tracker };
```

---

## 16. Monitoring Partitions

```sql
-- ดู partition statistics
SELECT 
    schemaname,
    tablename AS partition,
    n_live_tup AS live_rows,
    n_dead_tup AS dead_rows,
    last_vacuum,
    last_autovacuum,
    last_analyze
FROM pg_stat_user_tables
WHERE tablename LIKE 'app_events%'
ORDER BY tablename;

-- ดู query plans บน partitioned table
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)
SELECT COUNT(*) FROM app_events
WHERE created_at BETWEEN '2024-01-01' AND '2024-01-31'
  AND tenant_id = 1;

-- Check partition bounds
SELECT 
    c.relname AS partition,
    pg_get_expr(c.relpartbound, c.oid) AS bounds
FROM pg_class c
JOIN pg_inherits i ON i.inhrelid = c.oid
JOIN pg_class p ON p.oid = i.inhparent
WHERE p.relname = 'app_events'
ORDER BY c.relname;

-- ดู total size
SELECT 
    pg_size_pretty(SUM(pg_total_relation_size(c.oid))) AS total_size
FROM pg_class c
JOIN pg_inherits i ON i.inhrelid = c.oid
JOIN pg_class p ON p.oid = i.inhparent
WHERE p.relname = 'app_events';
```

---

## สรุป

Partitioning ใน PostgreSQL เป็น feature ที่ทรงพลังสำหรับจัดการข้อมูลขนาดใหญ่:

1. **Range Partitioning** เหมาะที่สุดสำหรับ time-series data
2. **List Partitioning** เหมาะสำหรับ discrete values (country, status)
3. **Hash Partitioning** เหมาะสำหรับ even distribution
4. **Partition Pruning** ทำให้ query เร็วขึ้นมาก - ต้องใช้ partition key ใน WHERE
5. **Indexes** บน parent table สร้างบนทุก partition อัตโนมัติ
6. **ATTACH/DETACH** ช่วย archive ข้อมูลเก่าอย่างมีประสิทธิภาพ
7. **pg_partman** ช่วย automate partition management
8. Primary key ต้องรวม partition key เสมอ
9. **Partition-wise join/aggregate** ช่วยเพิ่ม parallel processing

สำหรับ production ควร:
- สร้าง partitions ล่วงหน้า 3-6 เดือน
- ตั้ง retention policy สำหรับ old partitions
- Monitor partition sizes อย่างสม่ำเสมอ
- ใช้ default partition เพื่อป้องกัน insert errors

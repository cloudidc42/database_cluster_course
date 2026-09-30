# Part 86: Data Warehouse ด้วย PostgreSQL

## บทนำ

Data Warehouse (DW) คือระบบฐานข้อมูลที่ออกแบบมาเพื่อการวิเคราะห์ข้อมูล (OLAP - Online Analytical Processing) ต่างจาก OLTP (Online Transaction Processing) ที่ออกแบบมาสำหรับธุรกรรมประจำวัน PostgreSQL สามารถทำงานเป็น Data Warehouse ได้อย่างมีประสิทธิภาพ โดยเฉพาะสำหรับองค์กรขนาดกลางที่ข้อมูลยังไม่ถึงระดับ Petabyte

---

## 1. Data Warehouse บน PostgreSQL: เมื่อไหร่ทำได้

### ขนาดข้อมูลที่เหมาะสม

```
ขนาดที่ PostgreSQL จัดการได้ดี:
- ข้อมูล < 1TB: ดีมาก ไม่ต้องคิดอะไรมาก
- ข้อมูล 1-10TB: ดี ต้องใช้ partitioning และ indexing อย่างระมัดระวัง
- ข้อมูล 10-100TB: พอไหว ต้องใช้ Citus หรือ partitioning เชิงรุก
- ข้อมูล > 100TB: พิจารณา Snowflake, BigQuery, Redshift แทน
```

### เปรียบเทียบ PostgreSQL DW vs. Cloud DW

| Feature | PostgreSQL | Snowflake | BigQuery | Redshift |
|---------|-----------|-----------|----------|---------|
| ราคา | ถูก (self-hosted) | แพง (per-query) | แพง (per-query) | แพง (per-hour) |
| Performance < 1TB | ดีมาก | ดี | ดี | ดี |
| Performance > 10TB | พอใช้ | ดีมาก | ดีมาก | ดีมาก |
| SQL Compatibility | มาตรฐาน | มาตรฐาน | ต่าง | ต่าง |
| Extensions | มากมาย | จำกัด | จำกัด | จำกัด |
| Operational Cost | สูง (ต้องดูแลเอง) | ต่ำ | ต่ำ | กลาง |

### Use Cases ที่เหมาะสมกับ PostgreSQL DW

```
1. Business Intelligence ขนาดกลาง
   - รายงานยอดขายรายวัน/เดือน/ปี
   - Dashboard สำหรับ management
   - KPI tracking

2. Operational Analytics
   - Log analysis
   - User behavior analysis
   - Product analytics

3. Financial Reporting
   - P&L statements
   - Cost center analysis
   - Budget vs. actual

4. เมื่อต้องการ SQL features ขั้นสูง
   - Window functions
   - Recursive CTEs
   - Complex joins
   - Custom functions
```

---

## 2. Star Schema Implementation ใน PostgreSQL

### แนวคิด Star Schema

Star Schema ประกอบด้วย:
- **Fact Table**: ตารางกลางที่เก็บ measurements/metrics (ตัวเลข)
- **Dimension Tables**: ตารางรอบๆ ที่เก็บ descriptive attributes

```
          date_dim
             |
product_dim ─── sales_fact ─── customer_dim
             |
         store_dim
```

### สร้าง Database และ Schema

```sql
-- สร้าง database สำหรับ DW
CREATE DATABASE sales_dw;

-- เชื่อมต่อ
\c sales_dw

-- สร้าง schema แยกจาก public
CREATE SCHEMA warehouse;
CREATE SCHEMA staging;  -- สำหรับ ETL

-- ตั้งค่า search path
ALTER DATABASE sales_dw SET search_path TO warehouse, public;
```

### Dimension Table: date_dim

```sql
-- Date Dimension - สำคัญมากสำหรับ time-series analysis
CREATE TABLE warehouse.date_dim (
    date_key        INTEGER PRIMARY KEY,  -- YYYYMMDD format: 20240115
    full_date       DATE NOT NULL,
    day_of_week     SMALLINT NOT NULL,    -- 1=Monday, 7=Sunday
    day_name        VARCHAR(10) NOT NULL,
    day_of_month    SMALLINT NOT NULL,
    day_of_year     SMALLINT NOT NULL,
    week_of_year    SMALLINT NOT NULL,
    month_number    SMALLINT NOT NULL,
    month_name      VARCHAR(10) NOT NULL,
    quarter_number  SMALLINT NOT NULL,
    year_number     SMALLINT NOT NULL,
    is_weekend      BOOLEAN NOT NULL,
    is_holiday      BOOLEAN NOT NULL DEFAULT FALSE,
    fiscal_year     SMALLINT NOT NULL,   -- ปีงบประมาณ (อาจต่างจากปีปฏิทิน)
    fiscal_quarter  SMALLINT NOT NULL,
    fiscal_month    SMALLINT NOT NULL
);

-- Populate date dimension สำหรับ 10 ปี
INSERT INTO warehouse.date_dim
SELECT
    TO_CHAR(d, 'YYYYMMDD')::INTEGER AS date_key,
    d::DATE AS full_date,
    EXTRACT(ISODOW FROM d)::SMALLINT AS day_of_week,
    TO_CHAR(d, 'Day') AS day_name,
    EXTRACT(DAY FROM d)::SMALLINT AS day_of_month,
    EXTRACT(DOY FROM d)::SMALLINT AS day_of_year,
    EXTRACT(WEEK FROM d)::SMALLINT AS week_of_year,
    EXTRACT(MONTH FROM d)::SMALLINT AS month_number,
    TO_CHAR(d, 'Month') AS month_name,
    EXTRACT(QUARTER FROM d)::SMALLINT AS quarter_number,
    EXTRACT(YEAR FROM d)::SMALLINT AS year_number,
    CASE WHEN EXTRACT(ISODOW FROM d) IN (6, 7) THEN TRUE ELSE FALSE END AS is_weekend,
    FALSE AS is_holiday,
    -- Fiscal year: ปีงบประมาณเริ่ม April 1
    CASE 
        WHEN EXTRACT(MONTH FROM d) >= 4 
        THEN EXTRACT(YEAR FROM d)::SMALLINT
        ELSE (EXTRACT(YEAR FROM d) - 1)::SMALLINT
    END AS fiscal_year,
    CASE 
        WHEN EXTRACT(MONTH FROM d) IN (4,5,6) THEN 1
        WHEN EXTRACT(MONTH FROM d) IN (7,8,9) THEN 2
        WHEN EXTRACT(MONTH FROM d) IN (10,11,12) THEN 3
        ELSE 4
    END::SMALLINT AS fiscal_quarter,
    CASE 
        WHEN EXTRACT(MONTH FROM d) >= 4 
        THEN (EXTRACT(MONTH FROM d) - 3)::SMALLINT
        ELSE (EXTRACT(MONTH FROM d) + 9)::SMALLINT
    END AS fiscal_month
FROM GENERATE_SERIES(
    '2020-01-01'::DATE,
    '2030-12-31'::DATE,
    '1 day'::INTERVAL
) AS d;

-- Index สำหรับ query ที่ใช้บ่อย
CREATE INDEX idx_date_dim_full_date ON warehouse.date_dim(full_date);
CREATE INDEX idx_date_dim_year_month ON warehouse.date_dim(year_number, month_number);
```

### Dimension Table: product_dim

```sql
CREATE TABLE warehouse.product_dim (
    product_key     SERIAL PRIMARY KEY,
    product_id      VARCHAR(50) NOT NULL,  -- Business key จาก OLTP
    product_name    VARCHAR(200) NOT NULL,
    sku             VARCHAR(50),
    category_id     INTEGER,
    category_name   VARCHAR(100),
    subcategory_name VARCHAR(100),
    brand_name      VARCHAR(100),
    unit_cost       NUMERIC(12, 2),
    unit_price      NUMERIC(12, 2),
    weight_kg       NUMERIC(8, 3),
    is_active       BOOLEAN DEFAULT TRUE,
    -- SCD Type 2 fields
    valid_from      DATE NOT NULL DEFAULT CURRENT_DATE,
    valid_to        DATE,
    is_current      BOOLEAN DEFAULT TRUE,
    -- Metadata
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- SCD Type 2: Slowly Changing Dimension
-- เมื่อข้อมูล dimension เปลี่ยน จะ insert record ใหม่แทน update
-- ทำให้สามารถ query historical data ได้ถูกต้อง

CREATE INDEX idx_product_dim_product_id ON warehouse.product_dim(product_id);
CREATE INDEX idx_product_dim_is_current ON warehouse.product_dim(is_current) WHERE is_current = TRUE;
CREATE INDEX idx_product_dim_category ON warehouse.product_dim(category_name);
```

### Dimension Table: customer_dim

```sql
CREATE TABLE warehouse.customer_dim (
    customer_key    SERIAL PRIMARY KEY,
    customer_id     VARCHAR(50) NOT NULL,  -- Business key
    full_name       VARCHAR(200),
    email           VARCHAR(200),
    phone           VARCHAR(20),
    gender          CHAR(1),              -- M/F/O
    birth_year      SMALLINT,
    age_group       VARCHAR(20),          -- 18-24, 25-34, etc.
    registration_date DATE,
    -- Location
    city            VARCHAR(100),
    province        VARCHAR(100),
    region          VARCHAR(50),          -- North, South, etc.
    country         VARCHAR(50) DEFAULT 'Thailand',
    -- Segmentation
    customer_segment VARCHAR(50),         -- Premium, Standard, etc.
    lifetime_value  NUMERIC(15, 2),
    -- SCD Type 2
    valid_from      DATE NOT NULL DEFAULT CURRENT_DATE,
    valid_to        DATE,
    is_current      BOOLEAN DEFAULT TRUE
);

CREATE INDEX idx_customer_dim_customer_id ON warehouse.customer_dim(customer_id);
CREATE INDEX idx_customer_dim_region ON warehouse.customer_dim(region);
CREATE INDEX idx_customer_dim_segment ON warehouse.customer_dim(customer_segment);
```

### Fact Table: sales_fact

```sql
-- Sales Fact Table - ใหญ่ที่สุด append-only
-- ใช้ partitioning แบ่งตามเดือน
CREATE TABLE warehouse.sales_fact (
    sale_id         BIGSERIAL,
    -- Foreign Keys to dimensions
    date_key        INTEGER NOT NULL,
    product_key     INTEGER NOT NULL,
    customer_key    INTEGER NOT NULL,
    store_key       INTEGER,
    -- Degenerate dimensions (dimension ที่ไม่มี dimension table)
    order_id        VARCHAR(50),
    invoice_number  VARCHAR(50),
    -- Measures/Metrics
    quantity        INTEGER NOT NULL,
    unit_price      NUMERIC(12, 2) NOT NULL,
    unit_cost       NUMERIC(12, 2),
    discount_amount NUMERIC(12, 2) DEFAULT 0,
    tax_amount      NUMERIC(12, 2) DEFAULT 0,
    gross_amount    NUMERIC(15, 2) GENERATED ALWAYS AS 
                    (quantity * unit_price) STORED,
    net_amount      NUMERIC(15, 2) GENERATED ALWAYS AS 
                    (quantity * unit_price - discount_amount) STORED,
    profit_amount   NUMERIC(15, 2) GENERATED ALWAYS AS 
                    (quantity * (unit_price - COALESCE(unit_cost, 0)) - discount_amount) STORED,
    -- Timestamps
    sale_timestamp  TIMESTAMP NOT NULL,
    -- Partition key (ต้องอยู่ใน PRIMARY KEY ถ้า partition by range)
    PRIMARY KEY (sale_id, date_key)
) PARTITION BY RANGE (date_key);

-- สร้าง partitions รายเดือน
CREATE TABLE warehouse.sales_fact_2024_01 
    PARTITION OF warehouse.sales_fact
    FOR VALUES FROM (20240101) TO (20240201);

CREATE TABLE warehouse.sales_fact_2024_02 
    PARTITION OF warehouse.sales_fact
    FOR VALUES FROM (20240201) TO (20240301);

-- Auto-create partitions ด้วย script หรือ pg_partman
DO $$
DECLARE
    start_date DATE := '2024-01-01';
    end_date   DATE := '2024-12-31';
    curr_date  DATE;
    partition_name TEXT;
    start_key INTEGER;
    end_key INTEGER;
BEGIN
    curr_date := start_date;
    WHILE curr_date <= end_date LOOP
        partition_name := 'sales_fact_' || TO_CHAR(curr_date, 'YYYY_MM');
        start_key := TO_CHAR(curr_date, 'YYYYMMDD')::INTEGER;
        end_key := TO_CHAR(curr_date + INTERVAL '1 month', 'YYYYMMDD')::INTEGER;
        
        -- ตรวจสอบว่า partition มีอยู่แล้วหรือไม่
        IF NOT EXISTS (
            SELECT 1 FROM pg_class 
            WHERE relname = partition_name
        ) THEN
            EXECUTE FORMAT(
                'CREATE TABLE warehouse.%I PARTITION OF warehouse.sales_fact 
                 FOR VALUES FROM (%s) TO (%s)',
                partition_name, start_key, end_key
            );
            RAISE NOTICE 'Created partition: %', partition_name;
        END IF;
        
        curr_date := curr_date + INTERVAL '1 month';
    END LOOP;
END $$;

-- Indexes บน fact table
CREATE INDEX idx_sales_fact_date_key ON warehouse.sales_fact(date_key);
CREATE INDEX idx_sales_fact_product_key ON warehouse.sales_fact(product_key);
CREATE INDEX idx_sales_fact_customer_key ON warehouse.sales_fact(customer_key);

-- BRIN index สำหรับ sequential scan ขนาดใหญ่ (ประหยัด space มาก)
CREATE INDEX idx_sales_fact_brin ON warehouse.sales_fact 
    USING BRIN (date_key) WITH (pages_per_range = 128);
```

---

## 3. PostgreSQL Extensions สำหรับ Analytics

### ติดตั้ง Extensions

```bash
# ติดตั้ง extension packages
sudo apt-get install -y \
    postgresql-16-citus \
    postgresql-16-timescaledb-2 \
    postgresql-16-pg-partman

# หรือสำหรับ Fedora/RHEL
sudo dnf install -y \
    citus_16 \
    timescaledb-2-postgresql-16 \
    pg_partman_16
```

### Citus: Distributed PostgreSQL

```sql
-- ใน postgresql.conf
-- shared_preload_libraries = 'citus'

-- สร้าง extension
CREATE EXTENSION citus;

-- ตรวจสอบ citus nodes
SELECT * FROM citus_get_active_worker_nodes();

-- เพิ่ม worker nodes
SELECT citus_add_node('worker-1', 5432);
SELECT citus_add_node('worker-2', 5432);
SELECT citus_add_node('worker-3', 5432);

-- Distribute fact table โดย hash บน customer_key
SELECT create_distributed_table('warehouse.sales_fact', 'customer_key');

-- Distribute dimension tables เป็น reference tables (replicate ไปทุก node)
SELECT create_reference_table('warehouse.date_dim');
SELECT create_reference_table('warehouse.product_dim');
SELECT create_reference_table('warehouse.customer_dim');

-- Query ที่ distributed จะทำงานบน worker nodes ทั้งหมดพร้อมกัน
EXPLAIN SELECT 
    d.year_number,
    d.month_number,
    SUM(s.net_amount) as total_sales
FROM warehouse.sales_fact s
JOIN warehouse.date_dim d ON s.date_key = d.date_key
GROUP BY d.year_number, d.month_number
ORDER BY d.year_number, d.month_number;
```

### TimescaleDB: Time-Series

```sql
-- ใน postgresql.conf
-- shared_preload_libraries = 'timescaledb'

CREATE EXTENSION IF NOT EXISTS timescaledb;

-- สร้าง hypertable สำหรับ time-series data
CREATE TABLE warehouse.metrics (
    time        TIMESTAMPTZ NOT NULL,
    metric_name VARCHAR(100) NOT NULL,
    value       DOUBLE PRECISION,
    tags        JSONB
);

-- Convert เป็น hypertable (แบ่ง chunk ตามเวลาอัตโนมัติ)
SELECT create_hypertable('warehouse.metrics', 'time');

-- ตั้งค่า compression (ลด storage ได้ 90%+)
ALTER TABLE warehouse.metrics SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'metric_name'
);

-- Auto-compress chunks ที่เก่ากว่า 7 วัน
SELECT add_compression_policy('warehouse.metrics', INTERVAL '7 days');

-- Auto-drop old data
SELECT add_retention_policy('warehouse.metrics', INTERVAL '1 year');

-- Continuous Aggregates (เหมือน Materialized View แต่ refresh อัตโนมัติ)
CREATE MATERIALIZED VIEW warehouse.metrics_hourly
WITH (timescaledb.continuous) AS
SELECT 
    time_bucket('1 hour', time) AS bucket,
    metric_name,
    AVG(value) AS avg_value,
    MAX(value) AS max_value,
    MIN(value) AS min_value
FROM warehouse.metrics
GROUP BY bucket, metric_name;

-- Auto-refresh ทุกชั่วโมง
SELECT add_continuous_aggregate_policy('warehouse.metrics_hourly',
    start_offset => INTERVAL '3 hours',
    end_offset   => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour'
);
```

### Columnar Storage (Citus)

```sql
-- Columnar storage ดีสำหรับ analytics: compress ดี, scan เร็ว
-- แต่ไม่รองรับ UPDATE/DELETE

CREATE EXTENSION IF NOT EXISTS citus;

-- สร้าง table ด้วย columnar storage
CREATE TABLE warehouse.sales_fact_columnar (
    date_key     INTEGER,
    product_key  INTEGER,
    customer_key INTEGER,
    quantity     INTEGER,
    net_amount   NUMERIC(15, 2)
) USING columnar;

-- หรือ convert table ที่มีอยู่แล้ว
SELECT alter_table_set_access_method('existing_table', 'columnar');

-- Columnar storage options
ALTER TABLE warehouse.sales_fact_columnar SET (
    columnar.chunk_group_row_limit = 150000,
    columnar.stripe_row_limit = 15000000,
    columnar.compression = 'zstd',
    columnar.compression_level = 3
);

-- ตรวจสอบ compression ratio
SELECT 
    relname,
    pg_size_pretty(relation_size) AS heap_size,
    pg_size_pretty(compressed_size) AS compressed,
    ROUND(100 * (1 - compressed_size::numeric / relation_size), 1) AS "compression_%"
FROM columnar.relation_stats;
```

### pg_partman: Automated Partitioning

```sql
-- ติดตั้ง extension
CREATE EXTENSION pg_partman SCHEMA partman;

-- สร้าง parent table
CREATE TABLE warehouse.events (
    event_id    BIGSERIAL,
    event_time  TIMESTAMPTZ NOT NULL,
    event_type  VARCHAR(50),
    user_id     INTEGER,
    data        JSONB,
    PRIMARY KEY (event_id, event_time)
) PARTITION BY RANGE (event_time);

-- ให้ pg_partman จัดการ partitions อัตโนมัติ
SELECT partman.create_parent(
    p_parent_table => 'warehouse.events',
    p_control => 'event_time',
    p_type => 'native',
    p_interval => 'monthly',
    p_premake => 3  -- สร้าง partition ล่วงหน้า 3 เดือน
);

-- อัปเดต config
UPDATE partman.part_config 
SET 
    retention = '12 months',         -- เก็บข้อมูลแค่ 12 เดือน
    retention_keep_table = false,      -- ลบ partition เก่า
    infinite_time_partitions = true,
    automatic_maintenance = 'on'
WHERE parent_table = 'warehouse.events';

-- Run maintenance (จะถูก schedule อัตโนมัติด้วย pg_cron)
SELECT partman.run_maintenance('warehouse.events');
```

---

## 4. Materialized Views

### สร้าง Materialized Views พื้นฐาน

```sql
-- Monthly sales summary
CREATE MATERIALIZED VIEW warehouse.mv_monthly_sales AS
SELECT
    d.year_number,
    d.month_number,
    d.month_name,
    p.category_name,
    COUNT(DISTINCT s.customer_key) AS unique_customers,
    COUNT(*) AS num_transactions,
    SUM(s.quantity) AS total_units,
    SUM(s.gross_amount) AS gross_revenue,
    SUM(s.discount_amount) AS total_discounts,
    SUM(s.net_amount) AS net_revenue,
    SUM(s.profit_amount) AS gross_profit,
    AVG(s.net_amount) AS avg_order_value
FROM warehouse.sales_fact s
JOIN warehouse.date_dim d ON s.date_key = d.date_key
JOIN warehouse.product_dim p ON s.product_key = p.product_key
WHERE p.is_current = TRUE
GROUP BY d.year_number, d.month_number, d.month_name, p.category_name
WITH DATA;  -- สร้างข้อมูลทันที (ถ้า WITH NO DATA จะว่างไว้)

-- Index บน materialized view
CREATE UNIQUE INDEX idx_mv_monthly_sales 
    ON warehouse.mv_monthly_sales(year_number, month_number, category_name);
CREATE INDEX idx_mv_monthly_year 
    ON warehouse.mv_monthly_sales(year_number);

-- Customer lifetime value
CREATE MATERIALIZED VIEW warehouse.mv_customer_ltv AS
SELECT
    c.customer_key,
    c.customer_id,
    c.full_name,
    c.customer_segment,
    c.region,
    MIN(s.sale_timestamp) AS first_purchase,
    MAX(s.sale_timestamp) AS last_purchase,
    COUNT(*) AS num_orders,
    SUM(s.net_amount) AS lifetime_value,
    AVG(s.net_amount) AS avg_order_value,
    -- Days since last purchase
    EXTRACT(DAY FROM CURRENT_TIMESTAMP - MAX(s.sale_timestamp)) AS days_since_last_order,
    -- Purchase frequency (orders per month)
    COUNT(*) / GREATEST(
        EXTRACT(MONTH FROM AGE(MAX(s.sale_timestamp), MIN(s.sale_timestamp))), 
        1
    ) AS orders_per_month
FROM warehouse.customer_dim c
JOIN warehouse.sales_fact s ON c.customer_key = s.customer_key
WHERE c.is_current = TRUE
GROUP BY c.customer_key, c.customer_id, c.full_name, c.customer_segment, c.region
WITH DATA;

CREATE UNIQUE INDEX idx_mv_customer_ltv ON warehouse.mv_customer_ltv(customer_key);
```

### REFRESH MATERIALIZED VIEW CONCURRENTLY

```sql
-- Regular refresh: blocks reads ระหว่าง refresh
REFRESH MATERIALIZED VIEW warehouse.mv_monthly_sales;

-- CONCURRENTLY: ไม่ block reads แต่ต้องมี UNIQUE INDEX
-- ใช้เวลานานกว่า แต่ไม่มี downtime
REFRESH MATERIALIZED VIEW CONCURRENTLY warehouse.mv_monthly_sales;
REFRESH MATERIALIZED VIEW CONCURRENTLY warehouse.mv_customer_ltv;

-- ตรวจสอบ last refresh time
SELECT 
    schemaname,
    matviewname,
    ispopulated,
    definition
FROM pg_matviews
WHERE schemaname = 'warehouse';
```

### Auto-Refresh ด้วย pg_cron

```sql
-- ติดตั้ง pg_cron
-- ใน postgresql.conf: shared_preload_libraries = 'pg_cron'

CREATE EXTENSION pg_cron;

-- Refresh รายชั่วโมง (เวลา 00 นาทีของทุกชั่วโมง)
SELECT cron.schedule(
    'refresh-monthly-sales',
    '0 * * * *',
    $$REFRESH MATERIALIZED VIEW CONCURRENTLY warehouse.mv_monthly_sales$$
);

-- Refresh ทุก 15 นาที
SELECT cron.schedule(
    'refresh-customer-ltv',
    '*/15 * * * *',
    $$REFRESH MATERIALIZED VIEW CONCURRENTLY warehouse.mv_customer_ltv$$
);

-- Refresh ทุกคืน 02:00
SELECT cron.schedule(
    'refresh-heavy-views',
    '0 2 * * *',
    $$
    BEGIN;
    REFRESH MATERIALIZED VIEW CONCURRENTLY warehouse.mv_product_performance;
    REFRESH MATERIALIZED VIEW CONCURRENTLY warehouse.mv_cohort_analysis;
    COMMIT;
    $$
);

-- ตรวจสอบ scheduled jobs
SELECT * FROM cron.job;

-- ดู run history
SELECT * FROM cron.job_run_details 
ORDER BY start_time DESC 
LIMIT 20;
```

### Partial Refresh Strategies

```sql
-- สำหรับ view ที่ใหญ่มาก อาจทำ partial refresh ด้วย staging table

-- สร้าง incremental view
CREATE MATERIALIZED VIEW warehouse.mv_daily_sales AS
SELECT
    date_key,
    product_key,
    COUNT(*) AS num_orders,
    SUM(net_amount) AS daily_revenue
FROM warehouse.sales_fact
GROUP BY date_key, product_key
WITH DATA;

CREATE UNIQUE INDEX ON warehouse.mv_daily_sales(date_key, product_key);

-- Partial refresh: อัปเดตเฉพาะ recent data
CREATE OR REPLACE PROCEDURE warehouse.refresh_recent_sales(days_back INTEGER DEFAULT 7)
LANGUAGE plpgsql AS $$
DECLARE
    cutoff_key INTEGER;
BEGIN
    cutoff_key := TO_CHAR(CURRENT_DATE - days_back, 'YYYYMMDD')::INTEGER;
    
    -- ลบข้อมูลเก่าที่จะ refresh
    DELETE FROM warehouse.mv_daily_sales
    WHERE date_key >= cutoff_key;
    
    -- Insert ข้อมูลใหม่
    INSERT INTO warehouse.mv_daily_sales
    SELECT
        date_key,
        product_key,
        COUNT(*) AS num_orders,
        SUM(net_amount) AS daily_revenue
    FROM warehouse.sales_fact
    WHERE date_key >= cutoff_key
    GROUP BY date_key, product_key;
    
    RAISE NOTICE 'Refreshed data from date_key: %', cutoff_key;
END;
$$;

-- เรียกใช้งาน
CALL warehouse.refresh_recent_sales(7);

-- Schedule ด้วย pg_cron
SELECT cron.schedule(
    'partial-refresh-sales',
    '*/30 * * * *',
    $$CALL warehouse.refresh_recent_sales(7)$$
);
```

---

## 5. Indexing สำหรับ Analytics

### BRIN Index: สำหรับ Time-Ordered Data

```sql
-- BRIN (Block Range INdex) เหมาะกับ:
-- - ข้อมูลที่ insert ตามลำดับเวลา (time-series)
-- - ตาราง append-only ขนาดใหญ่
-- - ประหยัด storage มากเมื่อเทียบกับ B-tree

-- สร้าง BRIN index
CREATE INDEX idx_sales_brin_date ON warehouse.sales_fact 
    USING BRIN (sale_timestamp) 
    WITH (pages_per_range = 128);

-- เปรียบเทียบ size
SELECT 
    indexname,
    pg_size_pretty(pg_relation_size(indexname::regclass)) AS index_size
FROM pg_indexes
WHERE tablename = 'sales_fact'
    AND schemaname = 'warehouse';

-- BRIN vs B-tree comparison:
-- B-tree: 1GB index สำหรับ 10M rows
-- BRIN:   1MB index สำหรับ 10M rows (ถ้าข้อมูลเรียงตามลำดับ)

-- ตรวจสอบว่า BRIN ใช้งานได้
EXPLAIN (ANALYZE, BUFFERS) 
SELECT COUNT(*) FROM warehouse.sales_fact
WHERE sale_timestamp BETWEEN '2024-01-01' AND '2024-01-31';
```

### Partial Indexes: Active Rows Only

```sql
-- Partial index สำหรับ active products เท่านั้น
CREATE INDEX idx_product_active ON warehouse.product_dim(product_id)
WHERE is_active = TRUE AND is_current = TRUE;

-- Partial index สำหรับ recent orders
CREATE INDEX idx_sales_recent ON warehouse.sales_fact(customer_key, sale_timestamp)
WHERE sale_timestamp > CURRENT_DATE - INTERVAL '90 days';

-- ตัวอย่าง query ที่ใช้ partial index
EXPLAIN SELECT * FROM warehouse.product_dim
WHERE product_id = 'SKU-001' AND is_active = TRUE AND is_current = TRUE;

-- Index สำหรับ NULL values
CREATE INDEX idx_sales_no_store ON warehouse.sales_fact(date_key)
WHERE store_key IS NULL;
```

### Expression Indexes: Date Truncation

```sql
-- Expression index สำหรับ date_trunc
CREATE INDEX idx_sales_month ON warehouse.sales_fact(date_trunc('month', sale_timestamp));

-- Query ที่ใช้ expression index
EXPLAIN SELECT 
    date_trunc('month', sale_timestamp) AS month,
    SUM(net_amount)
FROM warehouse.sales_fact
GROUP BY date_trunc('month', sale_timestamp);

-- Index สำหรับ JSON fields
CREATE INDEX idx_events_event_type ON warehouse.events((data->>'event_type'));
CREATE INDEX idx_events_user_id ON warehouse.events((data->>'user_id'::INTEGER));

-- Composite expression index
CREATE INDEX idx_sales_year_month ON warehouse.sales_fact(
    EXTRACT(YEAR FROM sale_timestamp)::INTEGER,
    EXTRACT(MONTH FROM sale_timestamp)::INTEGER
);
```

---

## 6. Partitioning สำหรับ Analytics

### Monthly/Yearly Partitions

```sql
-- Parent table
CREATE TABLE warehouse.transactions (
    transaction_id  BIGSERIAL,
    transaction_date DATE NOT NULL,
    amount          NUMERIC(15, 2),
    customer_id     INTEGER,
    PRIMARY KEY (transaction_id, transaction_date)
) PARTITION BY RANGE (transaction_date);

-- Monthly partitions
CREATE TABLE warehouse.transactions_2024_01 
    PARTITION OF warehouse.transactions
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE warehouse.transactions_2024_02 
    PARTITION OF warehouse.transactions
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- Yearly partition สำหรับ archive
CREATE TABLE warehouse.transactions_2022 
    PARTITION OF warehouse.transactions
    FOR VALUES FROM ('2022-01-01') TO ('2023-01-01');

-- Default partition (จับข้อมูลที่ไม่ตรงกับ partition ไหน)
CREATE TABLE warehouse.transactions_default 
    PARTITION OF warehouse.transactions DEFAULT;
```

### Attach/Detach: Archiving

```sql
-- สร้าง archive table นอก partition
CREATE TABLE warehouse.transactions_archive_2022 (
    LIKE warehouse.transactions INCLUDING ALL
);

-- Detach partition เก่า (fast operation, no data movement)
ALTER TABLE warehouse.transactions 
    DETACH PARTITION warehouse.transactions_2022;

-- ย้ายข้อมูลไปยัง archive table
INSERT INTO warehouse.transactions_archive_2022 
    SELECT * FROM warehouse.transactions_2022;

-- Drop partition เก่า (หรือ move ไป cold storage)
DROP TABLE warehouse.transactions_2022;

-- หรือ DETACH CONCURRENTLY (PostgreSQL 14+)
-- ไม่ block reads/writes ระหว่าง detach
ALTER TABLE warehouse.transactions 
    DETACH PARTITION warehouse.transactions_old CONCURRENTLY;

-- Attach partition ใหม่
-- สร้าง table นอก partition ก่อน
CREATE TABLE warehouse.transactions_2025_01 (
    LIKE warehouse.transactions INCLUDING ALL
);

-- Load ข้อมูลเข้า (เร็วกว่า insert เข้า partitioned table โดยตรง)
COPY warehouse.transactions_2025_01 FROM '/data/transactions_2025_01.csv' CSV HEADER;

-- Attach เข้า partition (เร็วมาก, แค่เปลี่ยน metadata)
ALTER TABLE warehouse.transactions 
    ATTACH PARTITION warehouse.transactions_2025_01
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');
```

### Partition Pruning

```sql
-- ตรวจสอบว่า partition pruning ทำงาน
EXPLAIN SELECT COUNT(*) 
FROM warehouse.transactions
WHERE transaction_date BETWEEN '2024-01-01' AND '2024-03-31';

-- ผลลัพธ์ควรแสดงแค่ partition ที่เกี่ยวข้อง:
-- ->  Seq Scan on transactions_2024_01
-- ->  Seq Scan on transactions_2024_02
-- ->  Seq Scan on transactions_2024_03

-- Enable/disable partition pruning
SET enable_partition_pruning = on;  -- default: on

-- ตรวจสอบว่า pruning ทำงานใน WHERE clause
EXPLAIN SELECT * FROM warehouse.transactions
WHERE transaction_date = '2024-06-15';
-- ควรเห็นแค่ transactions_2024_06

-- Partition pruning ใน JOIN
EXPLAIN SELECT t.*, c.full_name
FROM warehouse.transactions t
JOIN warehouse.customer_dim c ON t.customer_id = c.customer_key
WHERE t.transaction_date >= '2024-01-01'
    AND t.transaction_date < '2024-02-01';
```

---

## 7. ETL สู่ PostgreSQL DW

### COPY Command: Bulk Load

```sql
-- Import จาก CSV file (เร็วที่สุด)
COPY warehouse.sales_fact (date_key, product_key, customer_key, quantity, unit_price, sale_timestamp)
FROM '/data/sales_2024_01.csv'
WITH (
    FORMAT CSV,
    HEADER TRUE,
    DELIMITER ',',
    NULL 'NULL',
    ENCODING 'UTF8'
);

-- Export เป็น CSV
COPY (
    SELECT * FROM warehouse.sales_fact
    WHERE date_key BETWEEN 20240101 AND 20240131
)
TO '/tmp/sales_jan_2024.csv'
WITH (FORMAT CSV, HEADER TRUE);

-- COPY ผ่าน network (client-side)
\COPY warehouse.sales_fact FROM 'local_file.csv' CSV HEADER;

-- COPY ด้วย transform (ใช้ staging table)
-- 1. Load เข้า staging ก่อน
COPY staging.sales_raw FROM '/data/raw_sales.csv' CSV HEADER;

-- 2. Transform และ insert ไป DW
INSERT INTO warehouse.sales_fact (
    date_key, product_key, customer_key, 
    quantity, unit_price, sale_timestamp
)
SELECT
    TO_CHAR(s.sale_date, 'YYYYMMDD')::INTEGER,
    p.product_key,
    c.customer_key,
    s.quantity,
    s.unit_price,
    s.sale_date::TIMESTAMP
FROM staging.sales_raw s
JOIN warehouse.product_dim p ON s.product_id = p.product_id AND p.is_current = TRUE
JOIN warehouse.customer_dim c ON s.customer_id = c.customer_id AND c.is_current = TRUE
WHERE NOT EXISTS (
    -- Deduplication
    SELECT 1 FROM warehouse.sales_fact sf
    WHERE sf.order_id = s.order_id
);
```

### Upsert: ON CONFLICT

```sql
-- Upsert สำหรับ dimension tables (SCD Type 1)
INSERT INTO warehouse.product_dim (
    product_id, product_name, category_name, unit_price, is_active
)
VALUES 
    ('SKU-001', 'Product A', 'Electronics', 999.00, TRUE),
    ('SKU-002', 'Product B', 'Clothing', 299.00, TRUE)
ON CONFLICT (product_id) DO UPDATE SET
    product_name  = EXCLUDED.product_name,
    category_name = EXCLUDED.category_name,
    unit_price    = EXCLUDED.unit_price,
    is_active     = EXCLUDED.is_active,
    updated_at    = CURRENT_TIMESTAMP;

-- SCD Type 2 upsert (เก็บประวัติ)
CREATE OR REPLACE PROCEDURE warehouse.upsert_product_scd2(
    p_product_id    VARCHAR(50),
    p_product_name  VARCHAR(200),
    p_unit_price    NUMERIC(12, 2)
)
LANGUAGE plpgsql AS $$
DECLARE
    old_price NUMERIC(12, 2);
BEGIN
    -- ตรวจสอบว่ามีการเปลี่ยนแปลงหรือไม่
    SELECT unit_price INTO old_price
    FROM warehouse.product_dim
    WHERE product_id = p_product_id AND is_current = TRUE;
    
    IF NOT FOUND THEN
        -- Insert new product
        INSERT INTO warehouse.product_dim (product_id, product_name, unit_price, valid_from, is_current)
        VALUES (p_product_id, p_product_name, p_unit_price, CURRENT_DATE, TRUE);
    ELSIF old_price != p_unit_price THEN
        -- Close existing record
        UPDATE warehouse.product_dim
        SET valid_to = CURRENT_DATE - 1, is_current = FALSE
        WHERE product_id = p_product_id AND is_current = TRUE;
        
        -- Insert new record
        INSERT INTO warehouse.product_dim (product_id, product_name, unit_price, valid_from, is_current)
        VALUES (p_product_id, p_product_name, p_unit_price, CURRENT_DATE, TRUE);
    END IF;
END;
$$;
```

---

## 8. Common Analytics Queries

### Cohort Retention Analysis

```sql
-- Cohort Analysis: วิเคราะห์ retention rate ของลูกค้า
WITH cohorts AS (
    -- กำหนด cohort ของแต่ละลูกค้า = เดือนที่ซื้อครั้งแรก
    SELECT
        customer_key,
        DATE_TRUNC('month', MIN(sale_timestamp)) AS cohort_month
    FROM warehouse.sales_fact
    GROUP BY customer_key
),
activity AS (
    -- ดูว่าลูกค้าซื้อในเดือนไหนบ้าง
    SELECT
        s.customer_key,
        DATE_TRUNC('month', s.sale_timestamp) AS activity_month
    FROM warehouse.sales_fact s
    GROUP BY s.customer_key, DATE_TRUNC('month', s.sale_timestamp)
),
cohort_activity AS (
    -- Join cohort กับ activity
    SELECT
        c.cohort_month,
        a.activity_month,
        COUNT(DISTINCT a.customer_key) AS active_users,
        EXTRACT(MONTH FROM AGE(a.activity_month, c.cohort_month))::INTEGER AS months_since_cohort
    FROM cohorts c
    JOIN activity a ON c.customer_key = a.customer_key
    GROUP BY c.cohort_month, a.activity_month
),
cohort_sizes AS (
    -- ขนาดของแต่ละ cohort (จำนวนลูกค้าที่เริ่มในเดือนนั้น)
    SELECT cohort_month, COUNT(*) AS cohort_size
    FROM cohorts
    GROUP BY cohort_month
)
SELECT
    ca.cohort_month,
    cs.cohort_size,
    ca.months_since_cohort,
    ca.active_users,
    ROUND(100.0 * ca.active_users / cs.cohort_size, 1) AS retention_rate
FROM cohort_activity ca
JOIN cohort_sizes cs ON ca.cohort_month = cs.cohort_month
WHERE ca.months_since_cohort <= 12
ORDER BY ca.cohort_month, ca.months_since_cohort;
```

### Moving Averages

```sql
-- 7-day, 30-day moving average ของยอดขาย
WITH daily_sales AS (
    SELECT
        d.full_date,
        SUM(s.net_amount) AS daily_revenue
    FROM warehouse.sales_fact s
    JOIN warehouse.date_dim d ON s.date_key = d.date_key
    WHERE d.year_number = 2024
    GROUP BY d.full_date
)
SELECT
    full_date,
    daily_revenue,
    -- 7-day moving average
    ROUND(AVG(daily_revenue) OVER (
        ORDER BY full_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ), 2) AS ma_7day,
    -- 30-day moving average
    ROUND(AVG(daily_revenue) OVER (
        ORDER BY full_date
        ROWS BETWEEN 29 PRECEDING AND CURRENT ROW
    ), 2) AS ma_30day,
    -- Cumulative sum (YTD)
    SUM(daily_revenue) OVER (
        ORDER BY full_date
        ROWS UNBOUNDED PRECEDING
    ) AS ytd_revenue,
    -- Day-over-day change
    daily_revenue - LAG(daily_revenue, 1) OVER (ORDER BY full_date) AS dod_change,
    -- Week-over-week change
    daily_revenue - LAG(daily_revenue, 7) OVER (ORDER BY full_date) AS wow_change
FROM daily_sales
ORDER BY full_date;
```

### Percentiles และ Statistical Analysis

```sql
-- Percentile analysis ของ order values
SELECT
    d.year_number,
    d.month_number,
    COUNT(*) AS num_orders,
    AVG(s.net_amount) AS avg_order,
    PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY s.net_amount) AS p25,
    PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY s.net_amount) AS p50_median,
    PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY s.net_amount) AS p75,
    PERCENTILE_CONT(0.90) WITHIN GROUP (ORDER BY s.net_amount) AS p90,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY s.net_amount) AS p95,
    PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY s.net_amount) AS p99,
    MAX(s.net_amount) AS max_order,
    STDDEV(s.net_amount) AS stddev_order
FROM warehouse.sales_fact s
JOIN warehouse.date_dim d ON s.date_key = d.date_key
GROUP BY d.year_number, d.month_number
ORDER BY d.year_number, d.month_number;

-- Distribution histogram
WITH order_buckets AS (
    SELECT
        WIDTH_BUCKET(net_amount, 0, 10000, 20) AS bucket,
        COUNT(*) AS num_orders,
        MIN(net_amount) AS min_in_bucket,
        MAX(net_amount) AS max_in_bucket
    FROM warehouse.sales_fact
    WHERE net_amount > 0
    GROUP BY WIDTH_BUCKET(net_amount, 0, 10000, 20)
)
SELECT
    bucket,
    ROUND(min_in_bucket, 0) || ' - ' || ROUND(max_in_bucket, 0) AS range,
    num_orders,
    REPEAT('█', (num_orders / 100)::INTEGER) AS histogram
FROM order_buckets
ORDER BY bucket;
```

### Funnel Analysis

```sql
-- E-commerce Funnel Analysis
-- Sessions → Product View → Add to Cart → Checkout → Purchase
WITH funnel_events AS (
    SELECT
        session_id,
        user_id,
        event_type,
        event_timestamp,
        product_id
    FROM warehouse.events
    WHERE event_timestamp >= CURRENT_DATE - INTERVAL '30 days'
        AND event_type IN ('session_start', 'product_view', 'add_to_cart', 'checkout', 'purchase')
),
session_funnel AS (
    SELECT
        session_id,
        user_id,
        MAX(CASE WHEN event_type = 'session_start' THEN 1 ELSE 0 END) AS reached_session,
        MAX(CASE WHEN event_type = 'product_view' THEN 1 ELSE 0 END) AS reached_product_view,
        MAX(CASE WHEN event_type = 'add_to_cart' THEN 1 ELSE 0 END) AS reached_add_to_cart,
        MAX(CASE WHEN event_type = 'checkout' THEN 1 ELSE 0 END) AS reached_checkout,
        MAX(CASE WHEN event_type = 'purchase' THEN 1 ELSE 0 END) AS reached_purchase
    FROM funnel_events
    GROUP BY session_id, user_id
)
SELECT
    'Sessions'     AS funnel_step, 1 AS step_order, SUM(reached_session) AS users,
    ROUND(100.0 * SUM(reached_session) / NULLIF(SUM(reached_session), 0), 1) AS conversion_rate
FROM session_funnel
UNION ALL
SELECT 'Product View', 2, SUM(reached_product_view),
    ROUND(100.0 * SUM(reached_product_view) / NULLIF(SUM(reached_session), 0), 1)
FROM session_funnel
UNION ALL
SELECT 'Add to Cart', 3, SUM(reached_add_to_cart),
    ROUND(100.0 * SUM(reached_add_to_cart) / NULLIF(SUM(reached_session), 0), 1)
FROM session_funnel
UNION ALL
SELECT 'Checkout', 4, SUM(reached_checkout),
    ROUND(100.0 * SUM(reached_checkout) / NULLIF(SUM(reached_session), 0), 1)
FROM session_funnel
UNION ALL
SELECT 'Purchase', 5, SUM(reached_purchase),
    ROUND(100.0 * SUM(reached_purchase) / NULLIF(SUM(reached_session), 0), 1)
FROM session_funnel
ORDER BY step_order;
```

### Session Analysis

```sql
-- Session Analysis: วิเคราะห์ user sessions
WITH raw_events AS (
    SELECT
        user_id,
        event_timestamp,
        event_type,
        -- ถ้า gap จาก event ก่อนหน้า > 30 นาที = new session
        CASE WHEN 
            event_timestamp - LAG(event_timestamp) OVER (PARTITION BY user_id ORDER BY event_timestamp) 
            > INTERVAL '30 minutes'
        OR LAG(event_timestamp) OVER (PARTITION BY user_id ORDER BY event_timestamp) IS NULL
        THEN 1 ELSE 0 END AS is_new_session
    FROM warehouse.events
    WHERE event_timestamp >= CURRENT_DATE - INTERVAL '7 days'
),
sessions AS (
    SELECT
        user_id,
        event_timestamp,
        event_type,
        -- Assign session ID
        SUM(is_new_session) OVER (
            PARTITION BY user_id 
            ORDER BY event_timestamp
        ) AS session_num
    FROM raw_events
),
session_stats AS (
    SELECT
        user_id,
        session_num,
        MIN(event_timestamp) AS session_start,
        MAX(event_timestamp) AS session_end,
        COUNT(*) AS events_in_session,
        EXTRACT(EPOCH FROM MAX(event_timestamp) - MIN(event_timestamp)) AS session_duration_seconds
    FROM sessions
    GROUP BY user_id, session_num
)
SELECT
    DATE_TRUNC('day', session_start) AS date,
    COUNT(*) AS total_sessions,
    COUNT(DISTINCT user_id) AS unique_users,
    AVG(events_in_session) AS avg_events_per_session,
    AVG(session_duration_seconds) / 60 AS avg_session_duration_minutes,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY session_duration_seconds) / 60 AS median_session_minutes
FROM session_stats
GROUP BY DATE_TRUNC('day', session_start)
ORDER BY date;
```

---

## 9. pgBadger: Query Analysis

### ติดตั้งและใช้งาน pgBadger

```bash
# ติดตั้ง pgBadger
sudo apt-get install -y pgbadger
# หรือ
pip install pgbadger

# ตั้งค่า PostgreSQL logging สำหรับ analysis
# postgresql.conf:
# log_min_duration_statement = 100    # log queries > 100ms
# log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
# log_checkpoints = on
# log_connections = on
# log_disconnections = on
# log_lock_waits = on
# log_temp_files = 0
# log_autovacuum_min_duration = 0

# Reload PostgreSQL
sudo systemctl reload postgresql

# Generate pgBadger report
pgbadger \
    --format stderr \
    --outfile /var/www/html/pgbadger/report.html \
    --last-parsed /var/log/postgresql/pgbadger_last_parsed \
    /var/log/postgresql/postgresql-*.log

# Incremental report (วันละครั้ง)
pgbadger \
    --incremental \
    --outdir /var/www/html/pgbadger/ \
    --last-parsed /var/log/postgresql/pgbadger_last_parsed \
    /var/log/postgresql/postgresql-$(date +%Y-%m-%d).log
```

### pg_stat_statements: Track Query Performance

```sql
-- Enable pg_stat_statements
-- postgresql.conf: shared_preload_libraries = 'pg_stat_statements'
-- pg_stat_statements.max = 10000
-- pg_stat_statements.track = all

CREATE EXTENSION pg_stat_statements;

-- Top 10 slowest queries (by total time)
SELECT
    ROUND(total_exec_time::NUMERIC, 2) AS total_time_ms,
    calls,
    ROUND(mean_exec_time::NUMERIC, 2) AS avg_time_ms,
    ROUND(stddev_exec_time::NUMERIC, 2) AS stddev_ms,
    rows,
    ROUND(100.0 * total_exec_time / SUM(total_exec_time) OVER(), 2) AS pct_total,
    LEFT(query, 100) AS query_snippet
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- Queries with high cache miss ratio
SELECT
    LEFT(query, 100) AS query_snippet,
    calls,
    shared_blks_hit,
    shared_blks_read,
    ROUND(100.0 * shared_blks_hit / NULLIF(shared_blks_hit + shared_blks_read, 0), 1) AS cache_hit_pct
FROM pg_stat_statements
WHERE calls > 100
ORDER BY shared_blks_read DESC
LIMIT 10;

-- Reset statistics
SELECT pg_stat_statements_reset();
```

---

## 10. Benchmarking: TPC-H สำหรับ Analytics

### TPC-H ใน PostgreSQL

```bash
# ติดตั้ง TPC-H tools
git clone https://github.com/electrum/tpch-dbgen.git
cd tpch-dbgen
make

# สร้างข้อมูล scale factor 1 = ~1GB
./dbgen -s 1

# สร้างข้อมูล scale factor 10 = ~10GB
./dbgen -s 10 -f
```

```sql
-- สร้าง TPC-H schema
CREATE SCHEMA tpch;

CREATE TABLE tpch.lineitem (
    l_orderkey      INTEGER NOT NULL,
    l_partkey       INTEGER NOT NULL,
    l_suppkey       INTEGER NOT NULL,
    l_linenumber    INTEGER NOT NULL,
    l_quantity      NUMERIC(15,2) NOT NULL,
    l_extendedprice NUMERIC(15,2) NOT NULL,
    l_discount      NUMERIC(15,2) NOT NULL,
    l_tax           NUMERIC(15,2) NOT NULL,
    l_returnflag    CHAR(1) NOT NULL,
    l_linestatus    CHAR(1) NOT NULL,
    l_shipdate      DATE NOT NULL,
    l_commitdate    DATE NOT NULL,
    l_receiptdate   DATE NOT NULL,
    l_shipinstruct  CHAR(25) NOT NULL,
    l_shipmode      CHAR(10) NOT NULL,
    l_comment       VARCHAR(44) NOT NULL
) PARTITION BY RANGE (l_shipdate);

-- สร้าง partitions
CREATE TABLE tpch.lineitem_1993 PARTITION OF tpch.lineitem
    FOR VALUES FROM ('1993-01-01') TO ('1994-01-01');
CREATE TABLE tpch.lineitem_1994 PARTITION OF tpch.lineitem
    FOR VALUES FROM ('1994-01-01') TO ('1995-01-01');
-- ... และต่อๆ ไป

-- Load data
\COPY tpch.lineitem FROM 'lineitem.tbl' DELIMITER '|' CSV;

-- TPC-H Query 1: Pricing Summary Report
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)
SELECT
    l_returnflag,
    l_linestatus,
    SUM(l_quantity) AS sum_qty,
    SUM(l_extendedprice) AS sum_base_price,
    SUM(l_extendedprice * (1 - l_discount)) AS sum_disc_price,
    SUM(l_extendedprice * (1 - l_discount) * (1 + l_tax)) AS sum_charge,
    AVG(l_quantity) AS avg_qty,
    AVG(l_extendedprice) AS avg_price,
    AVG(l_discount) AS avg_disc,
    COUNT(*) AS count_order
FROM tpch.lineitem
WHERE l_shipdate <= DATE '1998-12-01' - INTERVAL '90 days'
GROUP BY l_returnflag, l_linestatus
ORDER BY l_returnflag, l_linestatus;
```

### Benchmark Script

```bash
#!/bin/bash
# benchmark_tpch.sh

DB="sales_dw"
USER="postgres"
RESULTS_FILE="benchmark_results.csv"

echo "query,execution_time_ms" > $RESULTS_FILE

for q in 1 3 5 6 10 12; do
    echo "Running TPC-H Query $q..."
    
    TIME=$(psql -U $USER -d $DB -c "
    \timing on
    $(cat tpch_queries/q${q}.sql)
    " 2>&1 | grep "Time:" | awk '{print $2}')
    
    echo "$q,$TIME" >> $RESULTS_FILE
    echo "  Query $q: ${TIME}ms"
done

echo "Benchmark complete. Results in $RESULTS_FILE"
```

---

## 11. PostgreSQL DW Performance Tuning

### postgresql.conf สำหรับ Analytics

```ini
# postgresql.conf - Analytics workload

# Memory
shared_buffers = 16GB               # 25% of RAM
work_mem = 512MB                    # สูงสำหรับ sorting/hashing
maintenance_work_mem = 2GB          # สำหรับ VACUUM, CREATE INDEX
effective_cache_size = 48GB         # 75% of RAM
huge_pages = on

# Parallelism - สำคัญมากสำหรับ analytics
max_worker_processes = 16
max_parallel_workers = 16
max_parallel_workers_per_gather = 8
max_parallel_maintenance_workers = 4
parallel_tuple_cost = 0.05
parallel_setup_cost = 500

# Planner
effective_io_concurrency = 200      # SSD
random_page_cost = 1.1              # SSD (HDD = 4.0)
seq_page_cost = 1.0

# WAL (ลด overhead สำหรับ bulk loads)
wal_level = replica
checkpoint_completion_target = 0.9
wal_buffers = 64MB
min_wal_size = 1GB
max_wal_size = 4GB

# Statistics
default_statistics_target = 200     # Default 100, เพิ่มสำหรับ complex queries
enable_hashjoin = on
enable_mergejoin = on
enable_nestloop = on

# JIT (Just-In-Time compilation)
jit = on
jit_above_cost = 100000
jit_inline_above_cost = 500000
jit_optimize_above_cost = 500000
```

### Analyze และ Vacuum สำหรับ DW

```sql
-- ตั้งค่า autovacuum สำหรับ DW tables
ALTER TABLE warehouse.sales_fact SET (
    autovacuum_vacuum_scale_factor = 0.01,    -- Vacuum เมื่อ 1% ของ rows เปลี่ยน
    autovacuum_analyze_scale_factor = 0.005,  -- Analyze เมื่อ 0.5% เปลี่ยน
    autovacuum_vacuum_cost_delay = 2,          -- ms
    autovacuum_vacuum_cost_limit = 400
);

-- Manual VACUUM ANALYZE หลัง bulk load
VACUUM ANALYZE warehouse.sales_fact;
VACUUM ANALYZE warehouse.product_dim;
VACUUM ANALYZE warehouse.customer_dim;
VACUUM ANALYZE warehouse.date_dim;

-- ตรวจสอบ table bloat
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size,
    n_dead_tup,
    n_live_tup,
    ROUND(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 1) AS dead_pct,
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze
FROM pg_stat_user_tables
WHERE schemaname = 'warehouse'
ORDER BY n_dead_tup DESC;
```

---

## สรุป

PostgreSQL สามารถใช้เป็น Data Warehouse ได้อย่างมีประสิทธิภาพสำหรับข้อมูลขนาดกลาง โดยใช้:

1. **Star Schema** - ออกแบบ schema ให้เหมาะกับ analytics query
2. **Partitioning** - แบ่งข้อมูลตามเวลาเพื่อ performance และ maintenance
3. **Materialized Views** - Pre-compute aggregations สำหรับ fast queries
4. **Extensions** - Citus, TimescaleDB, Columnar เพิ่ม capabilities
5. **Proper Indexing** - BRIN, partial, expression indexes
6. **ETL Best Practices** - COPY, upsert, staging tables
7. **Query Optimization** - Window functions, CTEs, parallel queries

สำหรับข้อมูลที่ใหญ่กว่า 10TB หรือต้องการ auto-scaling ควรพิจารณา Cloud DW เช่น Snowflake หรือ BigQuery

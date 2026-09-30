# Part 84: OLAP vs OLTP Strategies

## บทนำ

หนึ่งในการตัดสินใจสถาปัตยกรรมที่สำคัญที่สุดสำหรับ database systems คือการเลือกว่าจะใช้ OLTP, OLAP หรือทั้งสองร่วมกัน การเข้าใจความแตกต่างและรู้จักเครื่องมือที่เหมาะสมจะช่วยให้ระบบทำงานได้เร็วกว่าหลายเท่าในต้นทุนที่ต่ำกว่า

---

## 1. OLTP: Online Transaction Processing

### 1.1 ลักษณะสำคัญ

```
OLTP = "ระบบที่ใช้ในชีวิตประจำวัน"
- E-commerce checkout
- Banking transactions
- Order management
- User authentication
- Inventory updates
```

**ลักษณะ workload:**
```
✅ Many small transactions (INSERT, UPDATE, DELETE)
✅ Low latency: < 100ms per query
✅ High concurrency: thousands of connections
✅ Row-oriented storage (เข้าถึง full row)
✅ ACID transactions
✅ Normalized schema (3NF หรือ BCNF)
❌ ไม่เหมาะสำหรับ aggregations ขนาดใหญ่
❌ ไม่เหมาะสำหรับ full table scans
```

**ตัวอย่าง OLTP queries:**
```sql
-- ดึงข้อมูล order (เร็วมาก ด้วย index)
SELECT o.*, u.email, u.phone
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.id = 'abc123';

-- อัปเดต order status
UPDATE orders 
SET status = 'SHIPPED', 
    shipped_at = NOW()
WHERE id = 'abc123' 
  AND status = 'CONFIRMED';

-- Insert order
INSERT INTO orders (id, user_id, amount, status)
VALUES ('abc124', 'user1', 1500.00, 'PENDING');

-- Transaction
BEGIN;
  UPDATE inventory SET quantity = quantity - 1 WHERE product_id = 'prod1';
  INSERT INTO order_items (order_id, product_id, quantity) VALUES ('abc124', 'prod1', 1);
COMMIT;
```

### 1.2 OLTP Tools

```
PostgreSQL:   ดีที่สุดสำหรับ complex queries + ACID
MySQL/MariaDB: รองรับ high write throughput
SQLite:       embedded, เหมาะกับ small apps
Oracle:       enterprise, ราคาแพง
SQL Server:   Windows ecosystem
```

### 1.3 PostgreSQL OLTP Optimization

```sql
-- ตัวอย่าง normalized schema (3NF) สำหรับ OLTP

-- Users table
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Products table
CREATE TABLE products (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    price NUMERIC(10, 2) NOT NULL,
    category_id UUID REFERENCES categories(id),
    inventory_count INT DEFAULT 0,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Orders table
CREATE TABLE orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    total_amount NUMERIC(10, 2) NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'pending',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Order items (normalized - ไม่เก็บซ้ำ)
CREATE TABLE order_items (
    order_id UUID REFERENCES orders(id) ON DELETE CASCADE,
    product_id UUID REFERENCES products(id),
    quantity INT NOT NULL,
    unit_price NUMERIC(10, 2) NOT NULL,
    PRIMARY KEY (order_id, product_id)
);

-- Indexes สำหรับ OLTP (หา row เร็ว)
CREATE INDEX idx_orders_user_id ON orders(user_id, created_at DESC);
CREATE INDEX idx_orders_status ON orders(status) WHERE status != 'completed';
CREATE INDEX idx_products_category ON products(category_id);
```

---

## 2. OLAP: Online Analytical Processing

### 2.1 ลักษณะสำคัญ

```
OLAP = "ระบบสำหรับ analytics และ reporting"
- Revenue reports
- User behavior analysis
- Marketing attribution
- Funnel analysis
- A/B test results
```

**ลักษณะ workload:**
```
✅ Few large queries (SELECT only)
✅ Aggregations: SUM, COUNT, AVG, percentiles
✅ Full or partial table scans
✅ Column-oriented storage (อ่านบาง columns เร็ว)
✅ Denormalized schema (fewer JOINs)
❌ ไม่รองรับ ACID transactions
❌ ไม่เหมาะกับ row-by-row operations
❌ Latency อาจเป็น seconds หรือ minutes
```

**ตัวอย่าง OLAP queries:**
```sql
-- Revenue by product category per month (scan หลาย millions rows)
SELECT
    p.category,
    DATE_TRUNC('month', o.created_at) as month,
    SUM(oi.quantity * oi.unit_price) as revenue,
    COUNT(DISTINCT o.user_id) as unique_customers,
    COUNT(*) as order_count
FROM orders o
JOIN order_items oi ON o.id = oi.order_id
JOIN products p ON oi.product_id = p.id
WHERE o.created_at >= '2024-01-01'
GROUP BY 1, 2
ORDER BY 1, 2;

-- Cohort retention (complex analytics)
WITH first_order AS (
    SELECT user_id, MIN(DATE_TRUNC('month', created_at)) as cohort_month
    FROM orders
    GROUP BY user_id
),
order_months AS (
    SELECT DISTINCT user_id, DATE_TRUNC('month', created_at) as order_month
    FROM orders
)
SELECT
    f.cohort_month,
    EXTRACT(MONTH FROM AGE(o.order_month, f.cohort_month)) as months_since_first_order,
    COUNT(DISTINCT o.user_id) as retained_users
FROM first_order f
JOIN order_months o ON f.user_id = o.user_id
GROUP BY 1, 2
ORDER BY 1, 2;
```

### 2.2 OLAP Tools Comparison

| Tool | Best For | Scale | Query Speed | Cost |
|------|----------|-------|-------------|------|
| ClickHouse | Real-time analytics, time-series | Billions of rows | Sub-second | Self-hosted |
| DuckDB | Embedded analytics, Parquet files | Millions of rows | Fast | Free |
| Redshift | Data warehouse, ETL | Petabytes | Seconds | AWS (expensive) |
| BigQuery | Serverless analytics | Petabytes | Seconds | GCP, pay-per-query |
| Snowflake | Cloud DW, separation of compute/storage | Petabytes | Seconds | Expensive |
| TimescaleDB | Time-series analytics | Billions of rows | Fast | Extension of PG |

---

## 3. PostgreSQL for Analytics

### 3.1 Materialized Views

```sql
-- Materialized view: pre-computed analytics
CREATE MATERIALIZED VIEW daily_revenue AS
SELECT
    DATE(created_at) as order_date,
    SUM(total_amount) as revenue,
    COUNT(*) as order_count,
    COUNT(DISTINCT user_id) as unique_customers
FROM orders
WHERE status = 'completed'
GROUP BY DATE(created_at)
WITH DATA;

-- Index บน materialized view
CREATE UNIQUE INDEX idx_daily_revenue_date ON daily_revenue(order_date);

-- Query (เร็วมาก เพราะ pre-computed)
SELECT * FROM daily_revenue
WHERE order_date >= CURRENT_DATE - INTERVAL '30 days'
ORDER BY order_date DESC;

-- Refresh (ต้องทำ manual หรือ schedule)
REFRESH MATERIALIZED VIEW CONCURRENTLY daily_revenue;
-- CONCURRENTLY = ไม่ lock ระหว่าง refresh

-- Incremental refresh ด้วย trigger (advanced)
CREATE FUNCTION refresh_daily_revenue() RETURNS TRIGGER AS $$
BEGIN
    REFRESH MATERIALIZED VIEW CONCURRENTLY daily_revenue;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

-- Refresh ทุกครั้งที่มี order ใหม่
-- (ในทางปฏิบัติ refresh แบบ scheduled ดีกว่า)
```

### 3.2 Partitioning สำหรับ Analytics

```sql
-- Time-series partitioning สำหรับ large tables
CREATE TABLE events (
    id BIGSERIAL,
    user_id UUID NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    properties JSONB,
    created_at TIMESTAMPTZ NOT NULL
) PARTITION BY RANGE (created_at);

-- สร้าง partitions รายเดือน
CREATE TABLE events_2024_01 PARTITION OF events
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE events_2024_02 PARTITION OF events
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

CREATE TABLE events_2024_09 PARTITION OF events
    FOR VALUES FROM ('2024-09-01') TO ('2024-10-01');

-- Default partition สำหรับ future data
CREATE TABLE events_default PARTITION OF events DEFAULT;

-- Indexes บน parent table (auto-inherited โดย partitions)
CREATE INDEX ON events(user_id, created_at);
CREATE INDEX ON events(event_type, created_at);

-- PostgreSQL จะ prune partitions อัตโนมัติ
EXPLAIN SELECT COUNT(*) FROM events WHERE created_at >= '2024-09-01';
-- Output: ใช้เฉพาะ events_2024_09 partition

-- Script: สร้าง partition รายเดือนอัตโนมัติ
CREATE OR REPLACE PROCEDURE create_monthly_partition(table_name TEXT, year INT, month INT)
LANGUAGE plpgsql AS $$
DECLARE
    partition_name TEXT;
    start_date DATE;
    end_date DATE;
BEGIN
    partition_name := format('%s_%s_%s', table_name, year, lpad(month::text, 2, '0'));
    start_date := make_date(year, month, 1);
    end_date := start_date + INTERVAL '1 month';
    
    EXECUTE format(
        'CREATE TABLE IF NOT EXISTS %I PARTITION OF %I FOR VALUES FROM (%L) TO (%L)',
        partition_name, table_name, start_date, end_date
    );
    
    RAISE NOTICE 'Created partition: %', partition_name;
END;
$$;

-- สร้าง partitions สำหรับ 12 เดือน
DO $$
BEGIN
    FOR m IN 1..12 LOOP
        CALL create_monthly_partition('events', 2024, m);
    END LOOP;
END;
$$;
```

### 3.3 Window Functions สำหรับ Analytics

```sql
-- Window functions: ทรงพลังมากสำหรับ analytics

-- Running total
SELECT
    order_date,
    daily_revenue,
    SUM(daily_revenue) OVER (ORDER BY order_date) as cumulative_revenue
FROM daily_revenue
ORDER BY order_date;

-- Percent of total
SELECT
    category,
    revenue,
    ROUND(revenue / SUM(revenue) OVER () * 100, 2) as pct_of_total
FROM (
    SELECT category, SUM(amount) as revenue
    FROM orders o
    JOIN products p ON o.product_id = p.id
    GROUP BY category
) t;

-- Month-over-month growth
SELECT
    month,
    revenue,
    LAG(revenue) OVER (ORDER BY month) as prev_month_revenue,
    ROUND(
        (revenue - LAG(revenue) OVER (ORDER BY month)) /
        NULLIF(LAG(revenue) OVER (ORDER BY month), 0) * 100,
        2
    ) as mom_growth_pct
FROM monthly_revenue
ORDER BY month;

-- Top products per category
SELECT *
FROM (
    SELECT
        category,
        product_name,
        revenue,
        RANK() OVER (PARTITION BY category ORDER BY revenue DESC) as rank
    FROM product_revenue
) t
WHERE rank <= 3;

-- Moving average (smoothing)
SELECT
    order_date,
    daily_revenue,
    ROUND(AVG(daily_revenue) OVER (
        ORDER BY order_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ), 2) as revenue_7day_avg
FROM daily_revenue
ORDER BY order_date;
```

---

## 4. ClickHouse: Real-time Analytics Engine

### 4.1 ClickHouse vs PostgreSQL

```
PostgreSQL:
- Row-oriented
- ACID transactions
- Best for: OLTP (< 1M rows per query)
- Query speed: ms

ClickHouse:
- Column-oriented
- No full ACID (eventual consistency)
- Best for: OLAP (billions of rows per query)
- Query speed: sub-second แม้ scan หลาย GB
```

### 4.2 Installation ด้วย Docker

```yaml
# docker-compose-clickhouse.yml
version: '3.8'

services:
  clickhouse:
    image: clickhouse/clickhouse-server:24.1
    ports:
      - "8123:8123"    # HTTP interface
      - "9000:9000"    # Native TCP interface
    volumes:
      - clickhouse_data:/var/lib/clickhouse
      - ./clickhouse/config.xml:/etc/clickhouse-server/config.xml
      - ./clickhouse/users.xml:/etc/clickhouse-server/users.xml
    ulimits:
      nofile:
        soft: 262144
        hard: 262144

  # ClickHouse Keeper (ZooKeeper alternative สำหรับ ClickHouse)
  clickhouse-keeper:
    image: clickhouse/clickhouse-keeper:24.1
    ports:
      - "9181:9181"
    volumes:
      - keeper_data:/var/lib/clickhouse-keeper
      - ./clickhouse/keeper_config.xml:/etc/clickhouse-keeper/keeper_config.xml

volumes:
  clickhouse_data:
  keeper_data:
```

```xml
<!-- clickhouse/users.xml -->
<clickhouse>
    <users>
        <admin>
            <password_sha256_hex><!-- SHA256 of your password --></password_sha256_hex>
            <networks>
                <ip>::/0</ip>
            </networks>
            <profile>default</profile>
            <quota>default</quota>
        </admin>
    </users>
    
    <profiles>
        <default>
            <max_memory_usage>10000000000</max_memory_usage>
            <use_uncompressed_cache>0</use_uncompressed_cache>
            <load_balancing>random</load_balancing>
        </default>
    </profiles>
</clickhouse>
```

### 4.3 MergeTree Engine

```sql
-- ClickHouse: MergeTree engine (most common)
-- ข้อมูลเก็บแบบ column-oriented, sorted by ORDER BY

CREATE TABLE orders (
    id String,
    user_id String,
    product_id String,
    quantity UInt32,
    amount Decimal(10, 2),
    status LowCardinality(String),  -- LowCardinality = dictionary encoding
    user_country LowCardinality(String),
    created_at DateTime64(3)  -- millisecond precision
) ENGINE = MergeTree()
ORDER BY (user_id, created_at)  -- Primary key สำหรับ sorting + indexing
PARTITION BY toYYYYMM(created_at)  -- Partition by month
TTL created_at + INTERVAL 2 YEAR  -- Auto-delete หลัง 2 ปี
SETTINGS index_granularity = 8192;

-- ReplicatedMergeTree สำหรับ high availability
CREATE TABLE orders_replicated (
    id String,
    user_id String,
    amount Decimal(10, 2),
    created_at DateTime64(3)
) ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/orders', '{replica}')
ORDER BY (user_id, created_at)
PARTITION BY toYYYYMM(created_at);
```

### 4.4 Real-time Ingestion

```sql
-- INSERT (fast bulk insert)
INSERT INTO orders (id, user_id, amount, status, created_at)
VALUES
    ('order1', 'user1', 1500.00, 'completed', now()),
    ('order2', 'user2', 250.00, 'pending', now());

-- INSERT from SELECT (ETL)
INSERT INTO orders
SELECT
    id,
    user_id,
    amount,
    status,
    created_at
FROM postgresql('postgres:5432', 'mydb', 'orders', 'user', 'password')
WHERE created_at >= today() - 1;

-- Buffer engine: สำหรับ high-frequency inserts
CREATE TABLE orders_buffer AS orders
ENGINE = Buffer(
    'default',    -- database
    'orders',     -- target table
    16,           -- num_layers
    10,           -- min_time (seconds)
    100,          -- max_time (seconds)
    10000,        -- min_rows
    1000000,      -- max_rows
    10000000,     -- min_bytes
    100000000     -- max_bytes
);

-- INSERT ไปที่ buffer จะ flush ไปยัง orders อัตโนมัติ
INSERT INTO orders_buffer SELECT * FROM new_orders_queue;
```

### 4.5 Performance: Billions of Rows

```sql
-- ClickHouse: query ไว แม้ data หลาย billions rows

-- Query 1: Daily revenue (scan entire table)
SELECT
    toDate(created_at) as order_date,
    sum(amount) as revenue,
    count() as order_count
FROM orders
WHERE created_at >= '2024-01-01'
GROUP BY order_date
ORDER BY order_date;
-- ⚡ < 1 second แม้มีหลาย billions rows

-- Query 2: Top users by revenue
SELECT
    user_id,
    sum(amount) as total_spent,
    count() as order_count
FROM orders
WHERE created_at >= '2024-09-01'
GROUP BY user_id
ORDER BY total_spent DESC
LIMIT 100;
-- ⚡ Sub-second

-- Query 3: ใช้ approximate functions (เร็วกว่า exact)
-- Approximate distinct count (HyperLogLog)
SELECT uniqCombined(user_id) as approx_unique_users
FROM events
WHERE event_date >= today() - 7;

-- Approximate quantiles (T-Digest)
SELECT
    quantileTDigest(0.5)(amount) as p50,
    quantileTDigest(0.95)(amount) as p95,
    quantileTDigest(0.99)(amount) as p99
FROM orders;
```

### 4.6 ClickHouse SQL Differences

```sql
-- ClickHouse มี syntax แตกต่างจาก PostgreSQL บ้าง

-- 1. ARRAY functions
SELECT arrayJoin([1, 2, 3]) as n;  -- expand array เป็น rows
SELECT groupArray(product_id) as products FROM order_items GROUP BY order_id;  -- collect values to array

-- 2. Date functions
SELECT
    today(),
    now(),
    toStartOfMonth(today()),
    toStartOfWeek(today(), 1),  -- 1 = Monday start
    dateDiff('day', '2024-01-01', today())

-- 3. String functions
SELECT
    lower('Hello'),
    upper('world'),
    trimLeft('  hello  '),
    replaceAll('hello world', ' ', '_'),
    match(email, '^[a-z]+@[a-z]+\.[a-z]+$') as valid_email

-- 4. Conditional aggregation
SELECT
    countIf(status = 'completed') as completed_count,
    sumIf(amount, status = 'completed') as completed_revenue,
    avgIf(amount, amount > 100) as avg_large_orders
FROM orders;

-- 5. SAMPLE (query ตัวอย่าง เร็วกว่า full scan)
SELECT COUNT(*) FROM orders SAMPLE 0.1;  -- 10% sample

-- 6. Materialized views ใน ClickHouse (auto-update)
CREATE MATERIALIZED VIEW orders_daily_mv
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMM(order_date)
ORDER BY (order_date, user_country)
AS SELECT
    toDate(created_at) as order_date,
    user_country,
    sum(amount) as revenue,
    count() as order_count
FROM orders
GROUP BY order_date, user_country;

-- Query materialized view (real-time updated)
SELECT order_date, sum(revenue) as total_revenue
FROM orders_daily_mv
WHERE order_date >= today() - 30
GROUP BY order_date
ORDER BY order_date;
```

### 4.7 ETL: PostgreSQL → ClickHouse

```python
# sync/pg_to_clickhouse.py
import psycopg2
import clickhouse_driver
import pandas as pd
from datetime import datetime, timedelta

class PostgresToClickHouseSync:
    def __init__(self):
        self.pg_conn = psycopg2.connect(os.environ['PG_DATABASE_URL'])
        self.ch_client = clickhouse_driver.Client(
            host='localhost',
            port=9000,
            database='analytics',
            user='admin',
            password=os.environ['CLICKHOUSE_PASSWORD']
        )
    
    def sync_orders(self, since: datetime = None):
        """Sync orders from PostgreSQL to ClickHouse"""
        if not since:
            # ดึงล่าสุดที่ sync ไว้
            result = self.ch_client.execute(
                "SELECT max(created_at) FROM orders"
            )
            since = result[0][0] or datetime(2020, 1, 1)
        
        # ดึงจาก PostgreSQL
        query = """
            SELECT 
                o.id,
                o.user_id,
                o.amount::float,
                o.status,
                u.country as user_country,
                o.created_at
            FROM orders o
            LEFT JOIN users u ON o.user_id = u.id
            WHERE o.created_at > %s
            ORDER BY o.created_at
        """
        
        df = pd.read_sql(query, self.pg_conn, params=(since,))
        
        if len(df) == 0:
            print("No new data to sync")
            return
        
        print(f"Syncing {len(df)} orders to ClickHouse...")
        
        # Insert ไปยัง ClickHouse
        self.ch_client.execute(
            "INSERT INTO orders VALUES",
            df.to_dict('records')
        )
        
        print(f"Synced {len(df)} orders successfully")
    
    def create_tables(self):
        """สร้าง tables ใน ClickHouse"""
        self.ch_client.execute("""
            CREATE TABLE IF NOT EXISTS orders (
                id String,
                user_id String,
                amount Float64,
                status LowCardinality(String),
                user_country LowCardinality(String),
                created_at DateTime64(3)
            ) ENGINE = ReplacingMergeTree(created_at)
            ORDER BY (user_id, created_at)
            PARTITION BY toYYYYMM(created_at)
        """)

# ใช้งาน
sync = PostgresToClickHouseSync()
sync.create_tables()
sync.sync_orders()
```

---

## 5. TimescaleDB สำหรับ Time-Series Analytics

```sql
-- TimescaleDB: PostgreSQL extension สำหรับ time-series

-- Enable extension
CREATE EXTENSION IF NOT EXISTS timescaledb;

-- สร้าง hypertable
CREATE TABLE metrics (
    time TIMESTAMPTZ NOT NULL,
    device_id VARCHAR(50),
    metric_name VARCHAR(100),
    value DOUBLE PRECISION
);

-- Convert เป็น hypertable (auto-partition by time)
SELECT create_hypertable('metrics', 'time',
    chunk_time_interval => INTERVAL '1 day'
);

-- Compression (save 90%+ storage)
ALTER TABLE metrics SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'device_id, metric_name',
    timescaledb.compress_orderby = 'time DESC'
);

-- Auto-compress chunks older than 7 days
SELECT add_compression_policy('metrics', INTERVAL '7 days');

-- Continuous aggregates (auto-refresh materialized views)
CREATE MATERIALIZED VIEW metrics_hourly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) as hour,
    device_id,
    metric_name,
    AVG(value) as avg_value,
    MAX(value) as max_value,
    MIN(value) as min_value
FROM metrics
GROUP BY 1, 2, 3;

-- Auto-refresh policy
SELECT add_continuous_aggregate_policy('metrics_hourly',
    start_offset => INTERVAL '3 hours',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour'
);

-- Query (ใช้ hypertable queries, fast)
SELECT
    time_bucket('5 minutes', time) as bucket,
    AVG(value) as avg_cpu
FROM metrics
WHERE device_id = 'server-1'
  AND metric_name = 'cpu_usage'
  AND time >= NOW() - INTERVAL '24 hours'
GROUP BY bucket
ORDER BY bucket;

-- TimescaleDB specific: last_point aggregate
SELECT device_id, last(value, time) as latest_value
FROM metrics
WHERE time >= NOW() - INTERVAL '1 hour'
GROUP BY device_id;
```

---

## 6. DuckDB: Embedded OLAP

### 6.1 ทำไมต้องใช้ DuckDB?

```
DuckDB คือ in-process OLAP database (เหมือน SQLite แต่สำหรับ analytics)
- ทำงานใน process เดียวกับ application
- ไม่ต้องมี server แยก
- ดี มากสำหรับ: data analysis, ETL, query Parquet/CSV
- ใช้ได้กับ Python, R, JavaScript, Java
```

### 6.2 DuckDB Basics

```python
# duckdb_demo.py
import duckdb
import pandas as pd

# Create in-memory database
conn = duckdb.connect(':memory:')

# หรือ persistent database
conn = duckdb.connect('analytics.duckdb')

# Query CSV โดยตรง (ไม่ต้อง import ก่อน!)
result = conn.execute("""
    SELECT
        strftime('%Y-%m', created_at) as month,
        SUM(amount) as revenue,
        COUNT(*) as orders
    FROM read_csv_auto('orders.csv')
    WHERE status = 'completed'
    GROUP BY month
    ORDER BY month
""").df()

# Query Parquet โดยตรง
result = conn.execute("""
    SELECT * FROM read_parquet('s3://my-bucket/data/*.parquet')
    WHERE year = '2024' AND month = '09'
""").df()

# ยังสามารถ query หลาย files ด้วย glob
result = conn.execute("""
    SELECT * FROM read_parquet([
        'data/2024/01/*.parquet',
        'data/2024/02/*.parquet',
        'data/2024/09/*.parquet'
    ])
""").df()

# Query JSON
result = conn.execute("""
    SELECT * FROM read_json('events.json')
    WHERE event_type = 'purchase'
""").df()

print(result)
```

### 6.3 DuckDB + PostgreSQL FDW

```python
# DuckDB อ่านจาก PostgreSQL โดยตรง
import duckdb

conn = duckdb.connect()

# Install postgres extension
conn.execute("INSTALL postgres; LOAD postgres;")

# Attach PostgreSQL database
conn.execute("""
    ATTACH 'dbname=mydb user=postgres host=localhost' AS pg (TYPE postgres);
""")

# Query PostgreSQL ผ่าน DuckDB (push-down execution)
result = conn.execute("""
    SELECT
        p.category,
        SUM(oi.quantity * oi.unit_price) as revenue
    FROM pg.public.order_items oi
    JOIN pg.public.products p ON oi.product_id = p.id
    JOIN pg.public.orders o ON oi.order_id = o.id
    WHERE o.created_at >= '2024-01-01'
    GROUP BY p.category
    ORDER BY revenue DESC
""").df()

print(result)

# Join PostgreSQL data กับ Parquet file!
result = conn.execute("""
    SELECT
        u.email,
        u.country,
        SUM(e.revenue) as total_revenue
    FROM read_parquet('s3://my-bucket/revenue_2024.parquet') e
    JOIN pg.public.users u ON e.user_id = u.id
    GROUP BY u.email, u.country
    ORDER BY total_revenue DESC
    LIMIT 100
""").df()
```

### 6.4 Analytics ใน Application Layer

```typescript
// Node.js: ใช้ DuckDB สำหรับ analytics
import Database from 'duckdb';
import { promisify } from 'util';

const db = new Database(':memory:');
const dbRun = promisify(db.run.bind(db));
const dbAll = promisify(db.all.bind(db));

// โหลด data จาก Parquet
async function analyzeRevenue(month: string) {
  await dbRun(`
    INSTALL parquet;
    LOAD parquet;
    INSTALL httpfs;
    LOAD httpfs;
    
    SET s3_access_key_id = '${process.env.AWS_ACCESS_KEY_ID}';
    SET s3_secret_access_key = '${process.env.AWS_SECRET_ACCESS_KEY}';
    SET s3_region = 'ap-southeast-1';
  `);
  
  const results = await dbAll(`
    SELECT
      category,
      SUM(amount) as revenue,
      COUNT(*) as orders,
      AVG(amount) as avg_order
    FROM read_parquet('s3://my-bucket/orders/month=${month}/*.parquet')
    GROUP BY category
    ORDER BY revenue DESC
  `);
  
  return results;
}

// REST API endpoint
app.get('/api/analytics/revenue/:month', async (req, res) => {
  const data = await analyzeRevenue(req.params.month);
  res.json(data);
});
```

---

## 7. Star Schema Design

### 7.1 Fact Tables

```sql
-- Fact table: เก็บ measurements/events
-- เป็นศูนย์กลางของ star schema

CREATE TABLE fact_orders (
    -- Surrogate key
    order_key BIGSERIAL PRIMARY KEY,
    
    -- Foreign keys to dimensions
    user_key INT REFERENCES dim_users(user_key),
    product_key INT REFERENCES dim_products(product_key),
    date_key INT REFERENCES dim_date(date_key),
    time_key INT REFERENCES dim_time(time_key),
    
    -- Measures (facts)
    quantity INT NOT NULL,
    unit_price NUMERIC(10, 2) NOT NULL,
    discount_pct NUMERIC(5, 2) DEFAULT 0,
    subtotal NUMERIC(12, 2) NOT NULL,
    tax_amount NUMERIC(10, 2) NOT NULL,
    total_amount NUMERIC(12, 2) NOT NULL,
    
    -- Degenerate dimensions (no separate table needed)
    order_id VARCHAR(50),
    order_status VARCHAR(50),
    
    -- Audit
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes สำหรับ analytics
CREATE INDEX idx_fact_orders_date ON fact_orders(date_key);
CREATE INDEX idx_fact_orders_user ON fact_orders(user_key);
CREATE INDEX idx_fact_orders_product ON fact_orders(product_key);
```

### 7.2 Dimension Tables

```sql
-- Dimension: Users
CREATE TABLE dim_users (
    user_key BIGSERIAL PRIMARY KEY,
    user_id VARCHAR(50) NOT NULL,  -- Natural key
    email VARCHAR(255),
    username VARCHAR(100),
    country VARCHAR(50),
    city VARCHAR(100),
    segment VARCHAR(50),  -- premium, regular, new
    registration_date DATE,
    age_group VARCHAR(20),
    gender VARCHAR(20),
    
    -- SCD Type 2 fields
    effective_date DATE NOT NULL,
    expiry_date DATE,
    is_current BOOLEAN DEFAULT TRUE,
    
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Dimension: Products
CREATE TABLE dim_products (
    product_key BIGSERIAL PRIMARY KEY,
    product_id VARCHAR(50) NOT NULL,
    product_name VARCHAR(255),
    category VARCHAR(100),
    subcategory VARCHAR(100),
    brand VARCHAR(100),
    supplier VARCHAR(100),
    cost_price NUMERIC(10, 2),
    
    -- SCD Type 2
    effective_date DATE NOT NULL,
    expiry_date DATE,
    is_current BOOLEAN DEFAULT TRUE
);

-- Dimension: Date (pre-populate สำหรับ analytics)
CREATE TABLE dim_date (
    date_key INT PRIMARY KEY,  -- YYYYMMDD format
    full_date DATE NOT NULL,
    year INT NOT NULL,
    quarter INT NOT NULL,
    month INT NOT NULL,
    month_name VARCHAR(20) NOT NULL,
    week INT NOT NULL,
    day_of_month INT NOT NULL,
    day_of_week INT NOT NULL,
    day_name VARCHAR(20) NOT NULL,
    is_weekend BOOLEAN NOT NULL,
    is_holiday BOOLEAN DEFAULT FALSE,
    fiscal_year INT,
    fiscal_quarter INT
);

-- Populate dim_date (2020-2030)
INSERT INTO dim_date
SELECT
    TO_CHAR(d, 'YYYYMMDD')::INT as date_key,
    d::DATE as full_date,
    EXTRACT(YEAR FROM d)::INT as year,
    EXTRACT(QUARTER FROM d)::INT as quarter,
    EXTRACT(MONTH FROM d)::INT as month,
    TO_CHAR(d, 'Month') as month_name,
    EXTRACT(WEEK FROM d)::INT as week,
    EXTRACT(DAY FROM d)::INT as day_of_month,
    EXTRACT(DOW FROM d)::INT as day_of_week,
    TO_CHAR(d, 'Day') as day_name,
    EXTRACT(DOW FROM d) IN (0, 6) as is_weekend,
    FALSE as is_holiday,
    EXTRACT(YEAR FROM d)::INT as fiscal_year,
    EXTRACT(QUARTER FROM d)::INT as fiscal_quarter
FROM generate_series('2020-01-01'::DATE, '2030-12-31'::DATE, '1 day') d;
```

### 7.3 Slowly Changing Dimensions (SCD)

```sql
-- SCD Type 2: เก็บ history ทั้งหมด

-- เมื่อ user เปลี่ยน segment
CREATE OR REPLACE PROCEDURE update_user_dimension(
    p_user_id VARCHAR,
    p_new_segment VARCHAR
)
LANGUAGE plpgsql AS $$
DECLARE
    v_current_key BIGINT;
BEGIN
    -- ปิด record เดิม
    UPDATE dim_users
    SET expiry_date = CURRENT_DATE - 1,
        is_current = FALSE
    WHERE user_id = p_user_id
      AND is_current = TRUE
    RETURNING user_key INTO v_current_key;
    
    -- Insert record ใหม่
    INSERT INTO dim_users (
        user_id, email, username, country, segment,
        effective_date, expiry_date, is_current
    )
    SELECT
        user_id, email, username, country,
        p_new_segment,  -- new segment
        CURRENT_DATE,
        NULL,
        TRUE
    FROM dim_users
    WHERE user_key = v_current_key;
    
    RAISE NOTICE 'Updated user % to segment %', p_user_id, p_new_segment;
END;
$$;
```

### 7.4 Star Schema Queries

```sql
-- Simple analytics query บน star schema
SELECT
    dd.year,
    dd.month_name,
    dp.category,
    du.country,
    SUM(fo.total_amount) as revenue,
    COUNT(*) as order_count,
    SUM(fo.quantity) as units_sold,
    AVG(fo.total_amount) as avg_order_value
FROM fact_orders fo
JOIN dim_date dd ON fo.date_key = dd.date_key
JOIN dim_products dp ON fo.product_key = dp.product_key
JOIN dim_users du ON fo.user_key = du.user_key
WHERE dd.year = 2024
  AND dp.is_current = TRUE
  AND du.is_current = TRUE
GROUP BY ROLLUP(dd.year, dd.month_name, dp.category, du.country)
ORDER BY dd.year, dd.month_name;

-- ROLLUP: สร้าง subtotals อัตโนมัติ
-- CUBE: สร้าง all combinations
SELECT
    COALESCE(category, 'ALL') as category,
    COALESCE(country, 'ALL') as country,
    SUM(total_amount) as revenue
FROM fact_orders fo
JOIN dim_products dp ON fo.product_key = dp.product_key
JOIN dim_users du ON fo.user_key = du.user_key
GROUP BY CUBE(dp.category, du.country)
ORDER BY 1, 2;
```

---

## 8. Hybrid HTAP

### 8.1 HTAP Architecture

```
HTAP = Hybrid Transactional/Analytical Processing
ทำ OLTP และ OLAP บน database เดียวกัน

Architecture แบบง่าย:
Primary PostgreSQL (OLTP)
     ↓ Streaming replication
Read Replica (OLAP queries)
     ↓ pg_timetable / cron
Materialized Views (pre-computed analytics)
```

```sql
-- PostgreSQL: HTAP ด้วย Read Replicas + Materialized Views

-- บน Read Replica: refresh materialized views
-- (ไม่กระทบ primary)

-- สร้าง materialized view สำหรับ analytics บน replica
CREATE MATERIALIZED VIEW analytics.revenue_summary AS
SELECT
    DATE_TRUNC('day', o.created_at) as date,
    p.category,
    u.country,
    COUNT(*) as order_count,
    SUM(o.total_amount) as revenue,
    COUNT(DISTINCT o.user_id) as unique_customers
FROM orders o
JOIN products p ON o.product_id = p.id
JOIN users u ON o.user_id = u.id
WHERE o.status = 'completed'
GROUP BY 1, 2, 3;

-- Application: route queries
class DatabaseRouter:
    def get_connection(self, query_type: str):
        if query_type == 'write':
            return primary_connection
        elif query_type == 'analytics':
            return replica_connection
        else:
            return read_replica_connection
```

---

## 9. Practical Guide: เลือก Database สำหรับ Analytics

```
📊 Decision Framework:

1. Data size < 10GB และ queries < 1s?
   → PostgreSQL ก็พอ (เพิ่ม indexes + materialized views)

2. Data size 10GB-1TB, need sub-second queries?
   → ClickHouse (self-hosted) หรือ BigQuery (managed)
   → DuckDB ถ้า queries ไม่ concurrent

3. Data size > 1TB?
   → Redshift, BigQuery, Snowflake (managed)
   → เพิ่ม Data Lake (S3) เพื่อ cold storage

4. Time-series data (metrics, IoT)?
   → TimescaleDB (PostgreSQL extension)
   → InfluxDB สำหรับ ultra high-frequency

5. Need real-time + historical analytics?
   → ClickHouse (fast inserts + fast queries)
   → Druid (Apache) สำหรับ ultra real-time

6. Analytics ใน application layer?
   → DuckDB (embedded, query Parquet/CSV)
```

---

## 10. Performance Benchmarks

```sql
-- ทดสอบ query performance ใน PostgreSQL vs ClickHouse

-- PostgreSQL: 10M rows
-- EXPLAIN ANALYZE
SELECT
    DATE(created_at) as date,
    SUM(amount) as revenue
FROM orders
WHERE created_at >= '2024-01-01'
GROUP BY 1
ORDER BY 1;
/*
Execution Time: 8432.5 ms  -- 8.4 seconds!
Rows: 10M scanned
*/

-- PostgreSQL: เพิ่ม partial index + materialized view
CREATE MATERIALIZED VIEW daily_revenue AS
SELECT DATE(created_at) as date, SUM(amount) as revenue
FROM orders GROUP BY 1;

SELECT * FROM daily_revenue WHERE date >= '2024-01-01';
/*
Execution Time: 12.3 ms  -- 12 ms! (700x faster)
*/

-- ClickHouse: query 1B rows
SELECT
    toDate(created_at) as date,
    sum(amount) as revenue
FROM orders
WHERE created_at >= '2024-01-01'
GROUP BY date
ORDER BY date;
/*
Execution Time: 0.45 seconds for 1B rows!
ใช้ CPU: 8 cores (parallel)
Data scanned: 3.2 GB (column pruning)
*/
```

---

## สรุป

OLAP vs OLTP ไม่ใช่ either/or แต่เป็น complementary:

1. **PostgreSQL**: OLTP backbone, รองรับ analytics ระดับปานกลาง
2. **ClickHouse**: สำหรับ billions of rows, sub-second queries
3. **DuckDB**: embedded analytics, perfect สำหรับ data exploration
4. **TimescaleDB**: time-series data บน PostgreSQL
5. **Star Schema**: design pattern ที่เหมาะสำหรับ analytics
6. **HTAP**: ใช้ read replicas + materialized views เพื่อ hybrid workload

Key insight: เริ่มจาก PostgreSQL + Materialized Views เสมอ แล้วค่อยเพิ่ม ClickHouse เมื่อ data > 100M rows หรือ query latency ไม่ได้ตามที่ต้องการ

# Part 78: TimescaleDB สำหรับ Time-Series Data

## Time-Series Data คืออะไร?

Time-series data คือข้อมูลที่มีการเปลี่ยนแปลงตามเวลาและถูกบันทึกพร้อมกับ timestamp:

```
ตัวอย่าง Time-Series Data:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Server Metrics:
2024-01-15 10:00:00 | server-01 | cpu=45.2% | mem=8192MB | disk_io=120MB/s
2024-01-15 10:00:10 | server-01 | cpu=48.1% | mem=8200MB | disk_io=115MB/s
2024-01-15 10:00:20 | server-01 | cpu=52.3% | mem=8250MB | disk_io=130MB/s

IoT Sensor Data:
2024-01-15 10:00:00 | sensor-A12 | temp=25.3°C | humidity=60% | pressure=1013hPa
2024-01-15 10:00:05 | sensor-A12 | temp=25.4°C | humidity=61% | pressure=1013hPa

Financial Data:
2024-01-15 09:30:00 | AAPL | price=184.25 | volume=1234567
2024-01-15 09:30:01 | AAPL | price=184.28 | volume=89034
2024-01-15 09:30:02 | AAPL | price=184.20 | volume=234512
```

### Characteristics of Time-Series Data

```
Time-Series ต่างจาก Relational Data อย่างไร:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. HIGH WRITE THROUGHPUT:
   - Metrics: ทุก 10 วินาที × 1000 servers = 100 inserts/sec
   - IoT: ทุก 5 วินาที × 10,000 sensors = 2,000 inserts/sec
   - ไม่ค่อยมี UPDATE (immutable history)

2. RECENT DATA HOTSPOT:
   - 90% of reads เป็น data ใน 7 วันล่าสุด
   - Old data rarely accessed (but must be retained for compliance)
   - Storage cost grows linearly with time

3. TIME-RANGE QUERIES:
   - "Show me CPU usage for the last 24 hours"
   - "What was the average temperature last month?"
   - Queries almost always include a time range

4. AGGREGATIONS OVER TIME:
   - min/max/avg per time bucket
   - Rate of change
   - Moving averages, percentiles

Traditional PostgreSQL Problems:
- B-tree index on timestamp → large index, slow range queries for huge tables
- No automatic data partitioning by time
- Compression not optimized for time-series patterns
- No built-in rollup/downsampling functions
```

### Challenges with Plain PostgreSQL

```sql
-- ปัญหาที่เกิดขึ้นกับ plain PostgreSQL สำหรับ time-series

-- Table ขนาดใหญ่ที่ไม่ใช้ TimescaleDB
CREATE TABLE metrics_plain (
    time TIMESTAMPTZ NOT NULL,
    device_id TEXT NOT NULL,
    metric_name TEXT NOT NULL,
    value DOUBLE PRECISION
);

CREATE INDEX ON metrics_plain (time DESC);
CREATE INDEX ON metrics_plain (device_id, time DESC);

-- หลังจาก 1 ปี: table มี ~3 billion rows
-- Query "last 24 hours" ยังเร็ว (index ดี)
-- แต่ปัญหาคือ:

-- 1. AUTOVACUUM ทำงานหนัก (ทุก row ต้องถูก vacuum)
-- 2. DELETE old data ช้ามาก (ต้อง vacuum หลัง delete)
-- 3. ไม่มี compression built-in
-- 4. No automatic partitioning

-- เปรียบเทียบ:
-- DELETE WHERE time < NOW() - INTERVAL '1 year'
-- บน 3B rows อาจใช้เวลา hours!

-- ใน TimescaleDB:
-- drop_chunks('metrics', older_than => '1 year') 
-- ใช้เวลา milliseconds! (delete entire chunk file)
```

---

## TimescaleDB: PostgreSQL Extension for Time-Series

### Installation ด้วย Docker

```bash
# Docker Compose สำหรับ TimescaleDB
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  timescaledb:
    image: timescale/timescaledb:latest-pg16
    environment:
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: metrics
      # TimescaleDB tuning
      TIMESCALEDB_TUNE_MAX_CONNS: "100"
    ports:
      - "5432:5432"
    volumes:
      - timescaledb_data:/var/lib/postgresql/data
    command: >
      postgres
        -c max_connections=200
        -c shared_buffers=256MB
        -c effective_cache_size=768MB
        -c maintenance_work_mem=64MB
        -c checkpoint_completion_target=0.9
        -c wal_buffers=16MB
        -c default_statistics_target=100
        -c random_page_cost=1.1
        -c effective_io_concurrency=200
        -c work_mem=4MB
        -c min_wal_size=1GB
        -c max_wal_size=4GB
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d metrics"]
      interval: 10s
      timeout: 5s
      retries: 5
  
  grafana:
    image: grafana/grafana:latest
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
    ports:
      - "3000:3000"
    depends_on:
      - timescaledb

volumes:
  timescaledb_data:
EOF

docker-compose up -d
```

```bash
# ติดตั้ง TimescaleDB บน Existing PostgreSQL 16
# (บน Ubuntu/Debian)

# เพิ่ม TimescaleDB repository
echo "deb https://packagecloud.io/timescale/timescaledb/ubuntu/ $(lsb_release -c -s) main" | \
    sudo tee /etc/apt/sources.list.d/timescaledb.list

wget --quiet -O - https://packagecloud.io/timescale/timescaledb/gpgkey | sudo apt-key add -

sudo apt-get update
sudo apt-get install -y timescaledb-2-postgresql-16

# Tune PostgreSQL สำหรับ TimescaleDB
sudo timescaledb-tune --quiet --yes

# Restart PostgreSQL
sudo systemctl restart postgresql

# Enable extension
psql -U postgres -d metrics -c "CREATE EXTENSION IF NOT EXISTS timescaledb;"
```

### Enable TimescaleDB Extension

```sql
-- เชื่อมต่อกับ database
\c metrics

-- ตรวจสอบ TimescaleDB version
CREATE EXTENSION IF NOT EXISTS timescaledb;

SELECT extname, extversion 
FROM pg_extension 
WHERE extname = 'timescaledb';

-- ดู TimescaleDB status
SELECT * FROM timescaledb_information.hypertables;

-- License info
SELECT * FROM timescaledb_information.license;
```

---

## Hypertables: Automatic Partitioning by Time

### สร้าง Hypertable

```sql
-- สร้าง table ปกติก่อน
CREATE TABLE server_metrics (
    time TIMESTAMPTZ NOT NULL,
    server_id TEXT NOT NULL,
    cpu_percent DOUBLE PRECISION,
    memory_mb INTEGER,
    disk_read_mb_s DOUBLE PRECISION,
    disk_write_mb_s DOUBLE PRECISION,
    network_in_mb_s DOUBLE PRECISION,
    network_out_mb_s DOUBLE PRECISION,
    load_avg_1m DOUBLE PRECISION,
    load_avg_5m DOUBLE PRECISION,
    load_avg_15m DOUBLE PRECISION
);

-- แปลงเป็น Hypertable
-- chunk_time_interval: แต่ละ chunk ครอบคลุมช่วงเวลาเท่าไหร่
SELECT create_hypertable(
    'server_metrics',
    'time',
    chunk_time_interval => INTERVAL '1 day',
    if_not_exists => TRUE
);

-- ดูข้อมูล hypertable
SELECT * FROM timescaledb_information.hypertables 
WHERE hypertable_name = 'server_metrics';

-- ดู chunks ที่สร้างแล้ว
SELECT 
    chunk_name,
    range_start,
    range_end,
    is_compressed,
    pg_size_pretty(total_bytes) AS total_size
FROM timescaledb_information.chunks
WHERE hypertable_name = 'server_metrics'
ORDER BY range_start DESC
LIMIT 20;
```

### Multi-Dimensional Partitioning (Time + Space)

```sql
-- Multi-dimensional partitioning: time + space (server_id)
-- เหมาะสำหรับ workload ที่ query specific server บ่อยๆ

CREATE TABLE iot_sensors (
    time TIMESTAMPTZ NOT NULL,
    sensor_id TEXT NOT NULL,
    location_id INTEGER NOT NULL,
    temperature DOUBLE PRECISION,
    humidity DOUBLE PRECISION,
    pressure DOUBLE PRECISION,
    battery_level DOUBLE PRECISION,
    signal_strength INTEGER
);

-- สร้าง hypertable กับ space partitioning
SELECT create_hypertable(
    'iot_sensors',
    'time',
    partitioning_column => 'location_id',
    number_partitions => 8,        -- 8 partitions ต่อ time chunk
    chunk_time_interval => INTERVAL '6 hours'
);

-- ดู partitioning dimensions
SELECT * FROM timescaledb_information.dimensions
WHERE hypertable_name = 'iot_sensors';
```

### Chunk Size Optimization

```sql
-- คำนวณ chunk_time_interval ที่เหมาะสม
-- Rule of thumb: แต่ละ chunk ควร fit ใน memory (25% of available RAM)
-- ถ้า RAM = 8GB, target chunk size = 2GB
-- ถ้า insert rate = 100 rows/sec = 8.6M rows/day
-- ถ้าแต่ละ row = 100 bytes → 860 MB/day
-- → chunk_time_interval = 2-3 days เหมาะสม

-- ตรวจสอบขนาด chunks ปัจจุบัน
SELECT
    chunk_name,
    range_start,
    range_end,
    pg_size_pretty(total_bytes) AS size,
    pg_size_pretty(heap_bytes) AS heap_size,
    pg_size_pretty(index_bytes) AS index_size
FROM timescaledb_information.chunks
WHERE hypertable_name = 'server_metrics'
ORDER BY range_start DESC;

-- ปรับ chunk_time_interval
SELECT set_chunk_time_interval('server_metrics', INTERVAL '7 days');

-- ดู adaptive chunking (TimescaleDB ทำ auto-tuning)
-- TimescaleDB 2.x: enable adaptive chunking
ALTER TABLE server_metrics SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'server_id'
);
```

---

## Compression

### Enable และ Configure Compression

```sql
-- Enable compression บน hypertable
ALTER TABLE server_metrics SET (
    timescaledb.compress,
    -- Segment by: ข้อมูลใน segment เดียวกันจะถูก compress ด้วยกัน
    -- เลือก column ที่มี low cardinality และ query บ่อยๆ
    timescaledb.compress_segmentby = 'server_id',
    -- Order by: ลำดับข้อมูลก่อน compress (ช่วย compression ratio)
    timescaledb.compress_orderby = 'time DESC'
);

-- Compress chunks manually
-- Compress chunks older than 7 days
SELECT compress_chunk(chunk)
FROM timescaledb_information.chunks
WHERE hypertable_name = 'server_metrics'
  AND range_end < NOW() - INTERVAL '7 days'
  AND is_compressed = false;

-- Compress specific chunk
SELECT compress_chunk('_timescaledb_internal._hyper_1_1_chunk');

-- Decompress chunk (เมื่อต้องการ update/delete)
SELECT decompress_chunk('_timescaledb_internal._hyper_1_1_chunk');
```

### Automatic Compression Policies

```sql
-- สร้าง compression policy: compress chunks older than 7 days
SELECT add_compression_policy('server_metrics', INTERVAL '7 days');

-- ดู compression policies
SELECT * FROM timescaledb_information.jobs
WHERE proc_name = 'policy_compression';

-- ดู compression stats
SELECT
    hypertable_name,
    total_chunks,
    number_compressed_chunks,
    ROUND(100.0 * number_compressed_chunks / total_chunks, 2) AS compression_pct,
    pg_size_pretty(before_compression_total_bytes) AS before,
    pg_size_pretty(after_compression_total_bytes) AS after,
    ROUND(
        100.0 * (1 - after_compression_total_bytes::NUMERIC / before_compression_total_bytes),
        2
    ) AS compression_ratio_pct
FROM timescaledb_information.compression_settings
JOIN (
    SELECT
        hypertable_name,
        SUM(CASE WHEN is_compressed THEN 1 ELSE 0 END) AS number_compressed_chunks,
        COUNT(*) AS total_chunks
    FROM timescaledb_information.chunks
    GROUP BY hypertable_name
) chunk_stats USING (hypertable_name);
```

### Compression Ratio Example

```sql
-- ก่อน compression:
-- server_metrics (30 days of data): 15 GB

-- หลัง compression (chunks older than 7 days):
-- Compressed: ~1.2 GB (compression ratio: 92.5%!)
-- Recent 7 days (uncompressed): ~3.5 GB
-- Total: ~4.7 GB (68.7% space savings)

-- เหตุผลที่ compress ได้ดี:
-- 1. Time-series data มี high correlation ระหว่าง adjacent rows
-- 2. cpu_percent: 45.2, 45.3, 45.1, 45.4 → delta encoding ได้ดี
-- 3. server_id (segmentby): เหมือนกันในทุก row ของ segment
-- 4. Columnar storage: ค่าเหมือนกันในแต่ละ column ถูก run-length encoded
```

---

## Continuous Aggregates

### สร้าง Continuous Aggregate

```sql
-- Continuous Aggregate: materialized view ที่ refresh อัตโนมัติ
-- เก็บผลลัพธ์ aggregation ไว้ล่วงหน้า เพื่อให้ query เร็วมาก

-- สร้าง hourly aggregate
CREATE MATERIALIZED VIEW server_metrics_hourly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS bucket,
    server_id,
    AVG(cpu_percent) AS avg_cpu,
    MAX(cpu_percent) AS max_cpu,
    MIN(cpu_percent) AS min_cpu,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY cpu_percent) AS p95_cpu,
    AVG(memory_mb) AS avg_memory_mb,
    MAX(memory_mb) AS max_memory_mb,
    AVG(disk_read_mb_s + disk_write_mb_s) AS avg_disk_io,
    COUNT(*) AS sample_count
FROM server_metrics
GROUP BY bucket, server_id
WITH NO DATA;  -- ยังไม่ populate data

-- Refresh policy: auto-refresh ทุก 30 นาที
SELECT add_continuous_aggregate_policy('server_metrics_hourly',
    start_offset => INTERVAL '3 hours',  -- refresh data starting 3 hours ago
    end_offset   => INTERVAL '10 minutes',  -- up to 10 minutes ago
    schedule_interval => INTERVAL '30 minutes'
);

-- Initial data refresh
CALL refresh_continuous_aggregate('server_metrics_hourly', NULL, NOW());

-- ดู continuous aggregates
SELECT * FROM timescaledb_information.continuous_aggregates;
```

### Daily Aggregate (สร้างจาก Hourly Aggregate)

```sql
-- Hierarchical Aggregation: daily จาก hourly
-- ประหยัดกว่าการ aggregate จาก raw data โดยตรง

CREATE MATERIALIZED VIEW server_metrics_daily
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', bucket) AS day,
    server_id,
    AVG(avg_cpu) AS avg_cpu,
    MAX(max_cpu) AS max_cpu,
    MIN(min_cpu) AS min_cpu,
    AVG(avg_memory_mb) AS avg_memory_mb,
    MAX(max_memory_mb) AS max_memory_mb,
    SUM(sample_count) AS total_samples
FROM server_metrics_hourly  -- ← จาก hourly aggregate
GROUP BY day, server_id;

SELECT add_continuous_aggregate_policy('server_metrics_daily',
    start_offset => INTERVAL '3 days',
    end_offset   => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 day'
);
```

### Real-Time Aggregates

```sql
-- Real-time aggregate: รวมข้อมูลจาก materialized data + live data
-- ทำให้ query เห็น data ล่าสุดได้โดยไม่ต้อง wait for refresh

-- เปิด real-time aggregation
ALTER MATERIALIZED VIEW server_metrics_hourly
SET (timescaledb.materialized_only = false);

-- Query นี้จะรวม:
-- 1. Materialized data (fast, from cagg)
-- 2. Real-time data ที่ยังไม่ถูก materialize
SELECT
    bucket,
    server_id,
    avg_cpu,
    max_cpu
FROM server_metrics_hourly
WHERE bucket > NOW() - INTERVAL '6 hours'
  AND server_id = 'web-server-01'
ORDER BY bucket DESC;
-- TimescaleDB จะ merge materialized + live data ให้อัตโนมัติ
```

---

## Retention Policies

### Automatic Data Retention

```sql
-- Policy: ลบ chunks older than 90 days
SELECT add_retention_policy('server_metrics', INTERVAL '90 days');

-- ดู retention policies
SELECT * FROM timescaledb_information.jobs
WHERE proc_name = 'policy_retention';

-- Manual: ลบ chunks เก่า
SELECT drop_chunks('server_metrics', older_than => INTERVAL '1 year');

-- ดูว่า chunks ไหนจะถูกลบ (dry-run)
SELECT chunk_name, range_start, range_end
FROM timescaledb_information.chunks
WHERE hypertable_name = 'server_metrics'
  AND range_end < NOW() - INTERVAL '90 days';
```

### Tiered Storage (Hot/Warm/Cold)

```sql
-- TimescaleDB Enterprise: Tiered Storage
-- Hot tier: fast SSD (recent 7 days)
-- Warm tier: cheaper SSD (7-90 days)
-- Cold tier: S3/object storage (90+ days)

-- Setup tablespace สำหรับ warm storage (slower disk)
CREATE TABLESPACE warm_storage
    LOCATION '/mnt/hdd/postgresql/warm';

-- Move old chunks to warm storage
SELECT move_chunk(
    chunk => '_timescaledb_internal._hyper_1_30_chunk',
    destination_tablespace => 'warm_storage',
    index_destination_tablespace => 'warm_storage',
    reorder_index => FALSE,
    verbose => TRUE
);

-- Automation: move chunks older than 7 days to warm storage
DO $$
DECLARE
    chunk_name TEXT;
BEGIN
    FOR chunk_name IN (
        SELECT c.chunk_name
        FROM timescaledb_information.chunks c
        WHERE c.hypertable_name = 'server_metrics'
          AND c.range_end < NOW() - INTERVAL '7 days'
          AND c.tablespace IS DISTINCT FROM 'warm_storage'
    ) LOOP
        PERFORM move_chunk(
            chunk => chunk_name::regclass,
            destination_tablespace => 'warm_storage'
        );
        RAISE NOTICE 'Moved chunk: %', chunk_name;
    END LOOP;
END $$;
```

---

## TimescaleDB Analytical Functions

### time_bucket(): Group by Time Interval

```sql
-- time_bucket: หัวใจของ TimescaleDB analytics

-- CPU usage ทุก 5 นาที
SELECT
    time_bucket('5 minutes', time) AS five_min_interval,
    server_id,
    AVG(cpu_percent) AS avg_cpu,
    MAX(cpu_percent) AS peak_cpu
FROM server_metrics
WHERE time > NOW() - INTERVAL '2 hours'
  AND server_id = 'web-server-01'
GROUP BY five_min_interval, server_id
ORDER BY five_min_interval DESC;

-- time_bucket_gapfill: เติม missing time buckets
SELECT
    time_bucket_gapfill('15 minutes', time) AS bucket,
    server_id,
    AVG(cpu_percent) AS avg_cpu
FROM server_metrics
WHERE time > NOW() - INTERVAL '6 hours'
  AND server_id = 'api-server-01'
GROUP BY bucket, server_id
ORDER BY bucket;
-- ถ้าไม่มีข้อมูลใน bucket นั้น → avg_cpu = NULL

-- ใช้ locf() เพื่อเติม NULL ด้วย last observation
SELECT
    time_bucket_gapfill('15 minutes', time) AS bucket,
    server_id,
    locf(AVG(cpu_percent)) AS avg_cpu_filled  -- Last Observation Carried Forward
FROM server_metrics
WHERE time > NOW() - INTERVAL '6 hours'
GROUP BY bucket, server_id
ORDER BY bucket;
```

### first() และ last()

```sql
-- first() และ last(): ค่าแรก/สุดท้ายใน time bucket
SELECT
    time_bucket('1 hour', time) AS hour,
    server_id,
    first(cpu_percent, time) AS cpu_at_start_of_hour,
    last(cpu_percent, time) AS cpu_at_end_of_hour,
    AVG(cpu_percent) AS avg_cpu,
    MAX(cpu_percent) - MIN(cpu_percent) AS cpu_range
FROM server_metrics
WHERE time > NOW() - INTERVAL '24 hours'
GROUP BY hour, server_id
ORDER BY hour DESC, server_id;

-- ตัวอย่าง: ราคาหุ้น OHLC (Open, High, Low, Close)
CREATE TABLE stock_prices (
    time TIMESTAMPTZ NOT NULL,
    symbol TEXT NOT NULL,
    price DOUBLE PRECISION NOT NULL,
    volume BIGINT
);

SELECT create_hypertable('stock_prices', 'time', chunk_time_interval => INTERVAL '1 day');

-- OHLC Query
SELECT
    time_bucket('1 minute', time) AS minute,
    symbol,
    first(price, time) AS open,
    MAX(price) AS high,
    MIN(price) AS low,
    last(price, time) AS close,
    SUM(volume) AS volume
FROM stock_prices
WHERE time > NOW() - INTERVAL '1 hour'
  AND symbol = 'AAPL'
GROUP BY minute, symbol
ORDER BY minute DESC;
```

### histogram(): Data Distribution

```sql
-- histogram(): ดูการกระจายของข้อมูล
SELECT
    server_id,
    histogram(cpu_percent, 0, 100, 10) AS cpu_distribution
FROM server_metrics
WHERE time > NOW() - INTERVAL '24 hours'
GROUP BY server_id;

-- Output:
-- server_id  | cpu_distribution
-- -----------+------------------------------------------
-- web-01     | {1234, 5678, 3456, 891, 234, 89, 12, 3, 1, 0}
-- หมายความว่า:
-- 1234 observations ระหว่าง 0-10%
-- 5678 observations ระหว่าง 10-20%
-- 3456 observations ระหว่าง 20-30%
-- ฯลฯ
```

### interpolate(): Fill Missing Values

```sql
-- interpolate(): linear interpolation สำหรับ missing data
SELECT
    time_bucket_gapfill('5 minutes', time) AS bucket,
    device_id,
    interpolate(AVG(temperature)) AS interpolated_temp,
    interpolate(AVG(humidity)) AS interpolated_humidity
FROM iot_sensors
WHERE time > NOW() - INTERVAL '6 hours'
  AND device_id = 'sensor-001'
GROUP BY bucket, device_id
ORDER BY bucket;
-- ถ้าไม่มีข้อมูลใน 5:30 แต่มีที่ 5:25 (25°C) และ 5:35 (27°C)
-- → interpolate ให้ 5:30 = 26°C (linear interpolation)
```

---

## Full Example: Server Metrics System (Prometheus-like)

### Database Schema

```sql
-- Complete schema สำหรับ server monitoring system

-- Servers registry
CREATE TABLE servers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    hostname TEXT UNIQUE NOT NULL,
    environment TEXT NOT NULL CHECK (environment IN ('production', 'staging', 'development')),
    datacenter TEXT NOT NULL,
    role TEXT NOT NULL,
    tags JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Main metrics hypertable
CREATE TABLE metrics (
    time TIMESTAMPTZ NOT NULL,
    server_id UUID NOT NULL REFERENCES servers(id),
    
    -- CPU
    cpu_user_pct DOUBLE PRECISION,
    cpu_system_pct DOUBLE PRECISION,
    cpu_idle_pct DOUBLE PRECISION,
    cpu_iowait_pct DOUBLE PRECISION,
    load_avg_1m DOUBLE PRECISION,
    load_avg_5m DOUBLE PRECISION,
    load_avg_15m DOUBLE PRECISION,
    
    -- Memory (MB)
    mem_total_mb INTEGER,
    mem_used_mb INTEGER,
    mem_cached_mb INTEGER,
    mem_available_mb INTEGER,
    
    -- Disk (MB/s)
    disk_read_mb_s DOUBLE PRECISION,
    disk_write_mb_s DOUBLE PRECISION,
    disk_util_pct DOUBLE PRECISION,
    
    -- Network (MB/s)
    net_rx_mb_s DOUBLE PRECISION,
    net_tx_mb_s DOUBLE PRECISION
);

-- Create hypertable
SELECT create_hypertable(
    'metrics',
    'time',
    chunk_time_interval => INTERVAL '1 day',
    if_not_exists => TRUE
);

-- Enable compression
ALTER TABLE metrics SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'server_id',
    timescaledb.compress_orderby = 'time DESC'
);

-- Auto-compress after 7 days
SELECT add_compression_policy('metrics', INTERVAL '7 days');

-- Auto-retain for 365 days
SELECT add_retention_policy('metrics', INTERVAL '365 days');

-- Indexes
CREATE INDEX ON metrics (server_id, time DESC);
CREATE INDEX ON metrics (time DESC);

-- Continuous Aggregates
CREATE MATERIALIZED VIEW metrics_1min
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 minute', time) AS bucket,
    server_id,
    AVG(cpu_user_pct + cpu_system_pct) AS avg_cpu_pct,
    MAX(cpu_user_pct + cpu_system_pct) AS max_cpu_pct,
    AVG(load_avg_1m) AS avg_load_1m,
    ROUND((AVG(mem_used_mb) / NULLIF(AVG(mem_total_mb), 0) * 100)::numeric, 2) AS avg_mem_pct,
    AVG(disk_read_mb_s + disk_write_mb_s) AS avg_disk_io_mb_s,
    AVG(net_rx_mb_s + net_tx_mb_s) AS avg_net_mb_s,
    COUNT(*) AS sample_count
FROM metrics
GROUP BY bucket, server_id
WITH NO DATA;

SELECT add_continuous_aggregate_policy('metrics_1min',
    start_offset => INTERVAL '2 minutes',
    end_offset   => INTERVAL '10 seconds',
    schedule_interval => INTERVAL '1 minute'
);

CREATE MATERIALIZED VIEW metrics_1hour
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', bucket) AS hour,
    server_id,
    AVG(avg_cpu_pct) AS avg_cpu_pct,
    MAX(max_cpu_pct) AS max_cpu_pct,
    AVG(avg_load_1m) AS avg_load_1m,
    AVG(avg_mem_pct) AS avg_mem_pct,
    MAX(avg_disk_io_mb_s) AS max_disk_io_mb_s,
    SUM(sample_count) AS total_samples
FROM metrics_1min
GROUP BY hour, server_id
WITH NO DATA;

SELECT add_continuous_aggregate_policy('metrics_1hour',
    start_offset => INTERVAL '2 hours',
    end_offset   => INTERVAL '10 minutes',
    schedule_interval => INTERVAL '30 minutes'
);
```

### Node.js Metrics Collector

```typescript
// metrics-collector.ts
import { Pool } from 'pg';
import * as os from 'os';
import * as fs from 'fs';

interface ServerMetrics {
  serverId: string;
  cpu: {
    userPct: number;
    systemPct: number;
    idlePct: number;
    iowaitPct: number;
  };
  load: {
    avg1m: number;
    avg5m: number;
    avg15m: number;
  };
  memory: {
    totalMb: number;
    usedMb: number;
    cachedMb: number;
    availableMb: number;
  };
  disk: {
    readMbs: number;
    writeMbs: number;
    utilPct: number;
  };
  network: {
    rxMbs: number;
    txMbs: number;
  };
}

class MetricsCollector {
  private pool: Pool;
  private serverId: string;
  private prevCpuStats: { idle: number; total: number } | null = null;
  private prevDiskStats: { readBytes: number; writeBytes: number; timestamp: number } | null = null;
  private prevNetStats: { rxBytes: number; txBytes: number; timestamp: number } | null = null;
  
  constructor(connectionString: string, serverId: string) {
    this.pool = new Pool({ connectionString, max: 5 });
    this.serverId = serverId;
  }
  
  private getCpuUsage(): { userPct: number; systemPct: number; idlePct: number; iowaitPct: number } {
    const cpus = os.cpus();
    let userTotal = 0, systemTotal = 0, idleTotal = 0, niceTotal = 0;
    
    for (const cpu of cpus) {
      userTotal += cpu.times.user;
      systemTotal += cpu.times.sys;
      idleTotal += cpu.times.idle;
      niceTotal += cpu.times.nice;
    }
    
    const total = userTotal + systemTotal + idleTotal + niceTotal;
    
    if (!this.prevCpuStats) {
      this.prevCpuStats = { idle: idleTotal, total };
      return { userPct: 0, systemPct: 0, idlePct: 100, iowaitPct: 0 };
    }
    
    const diffIdle = idleTotal - this.prevCpuStats.idle;
    const diffTotal = total - this.prevCpuStats.total;
    
    this.prevCpuStats = { idle: idleTotal, total };
    
    const idlePct = (diffIdle / diffTotal) * 100;
    
    return {
      userPct: (userTotal / total) * 100,
      systemPct: (systemTotal / total) * 100,
      idlePct,
      iowaitPct: 0,  // os module doesn't provide iowait directly
    };
  }
  
  private getMemoryStats(): { totalMb: number; usedMb: number; cachedMb: number; availableMb: number } {
    const totalBytes = os.totalmem();
    const freeBytes = os.freemem();
    const usedBytes = totalBytes - freeBytes;
    
    return {
      totalMb: Math.round(totalBytes / 1024 / 1024),
      usedMb: Math.round(usedBytes / 1024 / 1024),
      cachedMb: 0,  // Would need to read /proc/meminfo on Linux
      availableMb: Math.round(freeBytes / 1024 / 1024),
    };
  }
  
  private getLoadAvg(): { avg1m: number; avg5m: number; avg15m: number } {
    const [avg1m, avg5m, avg15m] = os.loadavg();
    return { avg1m, avg5m, avg15m };
  }
  
  async collectAndStore(): Promise<void> {
    const now = new Date();
    const cpu = this.getCpuUsage();
    const memory = this.getMemoryStats();
    const load = this.getLoadAvg();
    
    await this.pool.query(`
      INSERT INTO metrics (
        time, server_id,
        cpu_user_pct, cpu_system_pct, cpu_idle_pct, cpu_iowait_pct,
        load_avg_1m, load_avg_5m, load_avg_15m,
        mem_total_mb, mem_used_mb, mem_cached_mb, mem_available_mb,
        disk_read_mb_s, disk_write_mb_s, disk_util_pct,
        net_rx_mb_s, net_tx_mb_s
      ) VALUES (
        $1, $2,
        $3, $4, $5, $6,
        $7, $8, $9,
        $10, $11, $12, $13,
        $14, $15, $16,
        $17, $18
      )
    `, [
      now, this.serverId,
      cpu.userPct, cpu.systemPct, cpu.idlePct, cpu.iowaitPct,
      load.avg1m, load.avg5m, load.avg15m,
      memory.totalMb, memory.usedMb, memory.cachedMb, memory.availableMb,
      0, 0, 0,  // disk stats (simplified)
      0, 0,     // network stats (simplified)
    ]);
  }
  
  start(intervalMs: number = 10000): void {
    console.log(`Starting metrics collection every ${intervalMs}ms...`);
    
    const collect = async () => {
      try {
        await this.collectAndStore();
      } catch (error) {
        console.error('Collection error:', error);
      }
    };
    
    // Collect immediately, then on interval
    collect();
    setInterval(collect, intervalMs);
  }
}

// ตัวอย่างการใช้งาน
async function main() {
  const collector = new MetricsCollector(
    'postgresql://postgres:secret@localhost:5432/metrics',
    'server-001'
  );
  
  collector.start(10000);  // collect every 10 seconds
}

main().catch(console.error);
```

### Query API สำหรับ Dashboard

```typescript
// metrics-api.ts
import { Pool } from 'pg';
import express from 'express';

const app = express();
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// Query CPU usage for a server
app.get('/api/metrics/:serverId/cpu', async (req, res) => {
  const { serverId } = req.params;
  const { from = '1h', resolution = '1m' } = req.query;
  
  // Parse time range
  const fromInterval = from as string;
  const resolutionInterval = resolution as string;
  
  try {
    // Use continuous aggregate for longer time ranges (faster)
    const tableToUse = fromInterval.endsWith('d') || parseInt(fromInterval) > 6 
      ? 'metrics_1hour' 
      : 'metrics_1min';
    
    let query: string;
    
    if (tableToUse === 'metrics_1min') {
      query = `
        SELECT
          bucket AS time,
          ROUND(avg_cpu_pct::numeric, 2) AS avg_cpu,
          ROUND(max_cpu_pct::numeric, 2) AS max_cpu,
          ROUND(avg_load_1m::numeric, 2) AS load_avg
        FROM metrics_1min
        WHERE server_id = $1
          AND bucket > NOW() - $2::INTERVAL
        ORDER BY bucket ASC
      `;
    } else {
      query = `
        SELECT
          hour AS time,
          ROUND(avg_cpu_pct::numeric, 2) AS avg_cpu,
          ROUND(max_cpu_pct::numeric, 2) AS max_cpu,
          ROUND(avg_load_1m::numeric, 2) AS load_avg
        FROM metrics_1hour
        WHERE server_id = $1
          AND hour > NOW() - $2::INTERVAL
        ORDER BY hour ASC
      `;
    }
    
    const result = await pool.query(query, [serverId, fromInterval]);
    
    res.json({
      serverId,
      from: fromInterval,
      resolution: resolutionInterval,
      dataPoints: result.rows.length,
      data: result.rows,
    });
  } catch (error) {
    res.status(500).json({ error: String(error) });
  }
});

// Query top N servers by CPU
app.get('/api/metrics/top-cpu', async (req, res) => {
  const { limit = '10', period = '1h' } = req.query;
  
  const result = await pool.query(`
    SELECT
      m.server_id,
      s.hostname,
      s.environment,
      ROUND(AVG(m.avg_cpu_pct)::numeric, 2) AS avg_cpu,
      ROUND(MAX(m.max_cpu_pct)::numeric, 2) AS max_cpu
    FROM metrics_1min m
    JOIN servers s ON m.server_id = s.id
    WHERE m.bucket > NOW() - $1::INTERVAL
    GROUP BY m.server_id, s.hostname, s.environment
    ORDER BY avg_cpu DESC
    LIMIT $2
  `, [period, parseInt(limit as string)]);
  
  res.json({ servers: result.rows });
});

// Alert: servers with CPU > threshold
app.get('/api/alerts/high-cpu', async (req, res) => {
  const { threshold = '80', duration = '15m' } = req.query;
  
  const result = await pool.query(`
    SELECT
      m.server_id,
      s.hostname,
      s.environment,
      ROUND(AVG(m.avg_cpu_pct)::numeric, 2) AS sustained_cpu,
      MIN(m.bucket) AS started_at,
      MAX(m.bucket) AS last_seen
    FROM metrics_1min m
    JOIN servers s ON m.server_id = s.id
    WHERE m.bucket > NOW() - $1::INTERVAL
      AND m.avg_cpu_pct > $2
    GROUP BY m.server_id, s.hostname, s.environment
    HAVING COUNT(*) >= (EXTRACT(epoch FROM $1::INTERVAL) / 60)::int * 0.8
    ORDER BY sustained_cpu DESC
  `, [duration, parseFloat(threshold as string)]);
  
  res.json({ alerts: result.rows });
});

app.listen(3000, () => console.log('Metrics API running on :3000'));
```

---

## Performance Comparison: TimescaleDB vs Plain PostgreSQL

```sql
-- ทดสอบประสิทธิภาพ

-- สร้างข้อมูลทดสอบ
INSERT INTO metrics (time, server_id, cpu_user_pct, cpu_system_pct, cpu_idle_pct, 
                     load_avg_1m, mem_total_mb, mem_used_mb)
SELECT
    NOW() - (i || ' seconds')::INTERVAL,
    ('server-' || (i % 100)::text)::uuid,
    random() * 80 + 10,
    random() * 20,
    random() * 60,
    random() * 10,
    16384,
    8000 + (random() * 4000)::int
FROM generate_series(1, 10000000) AS i;

-- Query 1: Last 24 hours aggregate
\timing
SELECT
    time_bucket('1 hour', time) AS hour,
    server_id,
    AVG(cpu_user_pct) AS avg_cpu
FROM metrics  -- TimescaleDB hypertable
WHERE time > NOW() - INTERVAL '24 hours'
GROUP BY hour, server_id
ORDER BY hour DESC;
-- TimescaleDB: ~50ms (scans only 1 chunk)
-- Plain PostgreSQL: ~8500ms (scans full table!)

-- Query 2: Using continuous aggregate
SELECT avg_cpu_pct, max_cpu_pct
FROM metrics_1min  -- pre-aggregated
WHERE server_id = 'server-001'
  AND bucket > NOW() - INTERVAL '24 hours';
-- TimescaleDB cagg: ~5ms
-- Plain PostgreSQL (no cagg): ~8500ms

-- Query 3: Drop old data
SELECT drop_chunks('metrics', older_than => INTERVAL '1 year');
-- TimescaleDB: ~50ms (just removes chunk files)
-- Plain PostgreSQL DELETE: hours + VACUUM needed
```

---

## Grafana Integration

### Grafana Data Source Configuration

```json
{
  "name": "TimescaleDB",
  "type": "postgres",
  "url": "localhost:5432",
  "database": "metrics",
  "user": "grafana_reader",
  "secureJsonData": {
    "password": "reader_password"
  },
  "jsonData": {
    "sslmode": "disable",
    "maxOpenConns": 10,
    "maxIdleConns": 10,
    "connMaxLifetime": 14400,
    "postgresVersion": 1600,
    "timescaledb": true
  }
}
```

### Grafana Panel Query ตัวอย่าง

```sql
-- Panel: CPU Usage Time Series (Grafana Variables: $server, $from, $to)
SELECT
    $__timeGroupAlias(time, $__interval),
    server_id AS metric,
    AVG(cpu_user_pct + cpu_system_pct) AS "CPU Usage %"
FROM metrics
WHERE
    $__timeFilter(time)
    AND server_id = '$server'
GROUP BY 1, 2
ORDER BY 1

-- Panel: Memory Usage (เปอร์เซ็นต์)
SELECT
    $__timeGroupAlias(bucket, $__interval),
    s.hostname AS metric,
    AVG(avg_mem_pct) AS "Memory %"
FROM metrics_1min m
JOIN servers s ON m.server_id = s.id
WHERE
    $__timeFilter(bucket)
    AND s.environment = '$environment'
GROUP BY 1, 2
ORDER BY 1

-- Panel: Top CPU Servers (Table)
SELECT
    s.hostname,
    s.environment,
    ROUND(AVG(avg_cpu_pct)::numeric, 1) AS "Avg CPU %",
    ROUND(MAX(max_cpu_pct)::numeric, 1) AS "Peak CPU %"
FROM metrics_1min m
JOIN servers s ON m.server_id = s.id
WHERE bucket > NOW() - INTERVAL '1 hour'
GROUP BY s.hostname, s.environment
ORDER BY "Avg CPU %" DESC
LIMIT 20
```

---

## Use Cases และ Performance Tips

```
Use Cases ที่ TimescaleDB เหมาะมาก:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. Server/Infrastructure Monitoring
   - Metrics: CPU, memory, disk, network
   - Alerts, capacity planning, trend analysis
   - Stack: TimescaleDB + Grafana + Prometheus exporter

2. IoT Data
   - Sensor readings (temperature, pressure, etc.)
   - Device telemetry
   - Anomaly detection

3. Financial Time Series
   - Stock prices, OHLC data
   - Trading volume analysis
   - Portfolio performance tracking

4. Application Performance Monitoring (APM)
   - Request latencies, error rates
   - Throughput over time
   - SLA monitoring

5. Business Metrics
   - Revenue per hour/day
   - User signups over time
   - Conversion funnel analysis
```

```sql
-- Performance Tuning Tips:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

-- 1. ใช้ compress_segmentby wisely
-- High-cardinality (ไม่ดี): compress_segmentby = 'user_id' (millions of users)
-- Low-cardinality (ดี): compress_segmentby = 'server_id' (hundreds of servers)

-- 2. chunk_time_interval ที่เหมาะสม
-- เล็กเกินไป: metadata overhead, many chunks
-- ใหญ่เกินไป: chunk ไม่ fit ใน memory
-- Rule: target ~25% of memory per chunk (uncompressed)
SELECT show_chunks('metrics', older_than => '1 day'::INTERVAL);

-- 3. ใช้ EXPLAIN ANALYZE
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)
SELECT time_bucket('5 minutes', time), AVG(cpu_user_pct)
FROM metrics
WHERE time > NOW() - INTERVAL '24 hours'
GROUP BY 1 ORDER BY 1;
-- ดู "Chunks excluded" → chunk pruning ทำงาน

-- 4. Index ที่เหมาะสม
-- Default: TimescaleDB สร้าง index บน (time DESC) อัตโนมัติ
-- เพิ่ม: index บน columns ที่ query บ่อยๆ ใน WHERE clause
CREATE INDEX ON metrics (server_id, time DESC)
WITH (timescaledb.transaction_per_chunk);  -- create per chunk = faster
```

---

## สรุป TimescaleDB

```
TimescaleDB เหมาะกับ:
✓ Time-series data ที่มี high write throughput
✓ Recent data hotspot pattern
✓ Time-range queries เป็นหลัก
✓ ต้องการ SQL interface (ไม่ต้องเรียนรู้ query language ใหม่)
✓ ต้องการ join กับ relational data
✓ ต้องการ compression อัตโนมัติ

TimescaleDB ไม่เหมาะกับ:
✗ OLTP workload ปกติ (ไม่มีเวลา component)
✗ ต้องการ full-text search (ใช้ Elasticsearch แทน)
✗ Heavy UPDATE/DELETE patterns (time-series ควร immutable)
✗ Graph relationships (ใช้ Neo4j)

เปรียบเทียบกับทางเลือกอื่น:
InfluxDB: แข็งแกร่งสำหรับ metrics, query language ต่างกัน
ClickHouse: เร็วกว่าสำหรับ analytics queries, ไม่รองรับ ACID
Prometheus: เหมาะสำหรับ metrics + alerting, ไม่ใช่ general-purpose
TimescaleDB: ดีที่สุดเมื่อต้องการ SQL + time-series + PostgreSQL ecosystem
```

จบ Part 78 - TimescaleDB สำหรับ Time-Series Data ครอบคลุม:
- ลักษณะเฉพาะของ time-series data
- Installation, Hypertables, Partitioning
- Compression policies (10-20x ratio)
- Continuous Aggregates
- Retention policies
- Analytical functions: time_bucket, first/last, histogram, interpolate
- Full server monitoring system example
- Grafana integration

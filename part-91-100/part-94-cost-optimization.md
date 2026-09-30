# Part 94: Cost Optimization สำหรับ Database Cluster

## บทนำ

Database infrastructure เป็นหนึ่งในค่าใช้จ่ายสูงสุดของ modern applications โดยเฉพาะเมื่อ scale ขึ้น การ optimize ต้นทุนโดยไม่กระทบ performance และ reliability คือศิลปะที่ต้องใช้ทั้งความรู้ทางเทคนิคและความเข้าใจใน business

---

## 1. Database Cost Components

### 1.1 ภาพรวมต้นทุน

```markdown
## Monthly Cost Breakdown (Example: Medium SaaS Application)

### Compute
- Primary DB: db.r6g.2xlarge = ~$600/month
- Read Replica x2: db.r6g.xlarge x2 = ~$600/month
- Connection Pooler (PgBouncer): t3.small = ~$15/month

### Storage
- Primary: 500GB SSD (gp3) = ~$50/month
- Replica snapshots: ~$20/month
- Transaction logs: ~$10/month

### Network
- Data transfer out: ~$50/month
- Cross-AZ replication: ~$30/month

### Cache
- ElastiCache Redis r6g.large: ~$200/month

### Object Storage
- S3: 2TB = ~$46/month
- CloudFront: ~$40/month

### Operations
- DBA time (part-time): ~$2,000/month
- Monitoring (Datadog/CloudWatch): ~$200/month

Total: ~$3,800/month
```

### 1.2 Cost per Request Analysis

```javascript
// วิเคราะห์ cost per request
const monthlyDbCost = 1800; // USD/month
const requestsPerMonth = 50_000_000; // 50M requests

const costPerRequest = monthlyDbCost / requestsPerMonth;
// = $0.000036 per request

// ถ้า cache hit rate 80%:
// Effective requests hitting DB = 10M
// Cost per cached request ≈ 0
// Cost per DB request = $1800 / 10M = $0.00018

console.log(`Cost per DB request: $${costPerRequest.toFixed(6)}`);
console.log(`With 80% cache: $${(monthlyDbCost / (requestsPerMonth * 0.2)).toFixed(6)}`);
```

---

## 2. PostgreSQL Cost Optimization

### 2.1 Right-sizing: เริ่มเล็ก scale ขึ้นตามความต้องการ

```sql
-- ดู resource usage ปัจจุบัน
SELECT
    -- CPU utilization estimate จาก query stats
    sum(total_exec_time) / 1000 AS total_exec_seconds,
    sum(calls) AS total_calls,
    round(sum(total_exec_time / calls)::numeric, 2) AS avg_ms_per_call
FROM pg_stat_statements
WHERE dbid = (SELECT oid FROM pg_database WHERE datname = current_database());

-- Memory usage
SELECT
    pg_size_pretty(sum(pg_relation_size(oid))) AS total_table_size,
    pg_size_pretty(pg_database_size(current_database())) AS db_size
FROM pg_class
WHERE relkind = 'r';

-- Shared buffer hit rate (ควร > 99%)
SELECT
    round(
        100.0 * sum(heap_blks_hit) / NULLIF(sum(heap_blks_hit) + sum(heap_blks_read), 0),
        2
    ) AS buffer_hit_rate
FROM pg_statio_user_tables;

-- Cache hit rate per table
SELECT
    relname,
    heap_blks_hit,
    heap_blks_read,
    round(
        100.0 * heap_blks_hit / NULLIF(heap_blks_hit + heap_blks_read, 0),
        2
    ) AS hit_rate
FROM pg_statio_user_tables
WHERE heap_blks_hit + heap_blks_read > 0
ORDER BY heap_blks_read DESC
LIMIT 20;
```

### 2.2 Read Replicas: Offload Read Traffic

```sql
-- วิเคราะห์ query types
SELECT
    left(query, 50) AS query_preview,
    calls,
    total_exec_time,
    CASE
        WHEN query ILIKE 'SELECT%' THEN 'READ'
        WHEN query ILIKE 'INSERT%' OR query ILIKE 'UPDATE%' OR query ILIKE 'DELETE%' THEN 'WRITE'
        ELSE 'OTHER'
    END AS query_type,
    round((total_exec_time / total_exec_time_total * 100)::numeric, 2) AS pct_of_total_time
FROM pg_stat_statements,
    (SELECT sum(total_exec_time) AS total_exec_time_total FROM pg_stat_statements) AS totals
ORDER BY total_exec_time DESC
LIMIT 20;
```

```javascript
// connection-router.js: Route reads to replica
const { Pool } = require('pg');

const primaryPool = new Pool({ connectionString: process.env.DB_PRIMARY_URL });
const replicaPool = new Pool({ connectionString: process.env.DB_REPLICA_URL });

// Query classifier
function isReadQuery(sql) {
  const normalized = sql.trim().toUpperCase();
  return normalized.startsWith('SELECT') || normalized.startsWith('WITH');
}

async function query(sql, params, options = {}) {
  const { forceWrite = false } = options;
  
  // Use replica for reads (saves money on primary compute)
  if (isReadQuery(sql) && !forceWrite) {
    try {
      return await replicaPool.query(sql, params);
    } catch (err) {
      // Fallback to primary if replica fails
      console.warn('Replica query failed, falling back to primary:', err.message);
      return await primaryPool.query(sql, params);
    }
  }
  
  return await primaryPool.query(sql, params);
}

module.exports = { query, primaryPool, replicaPool };
```

### 2.3 Connection Pooling: ลด Connection Overhead

```ini
; pgbouncer.ini
[databases]
myapp = host=localhost port=5432 dbname=myapp

[pgbouncer]
listen_port = 6432
listen_addr = 0.0.0.0
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt

; Transaction pooling: ประหยัดที่สุด
pool_mode = transaction

; Pool sizes
default_pool_size = 20        ; connections ต่อ user/db pair
max_client_conn = 1000        ; max app connections
reserve_pool_size = 5
max_db_connections = 100      ; total connections ถึง postgres

; Timeouts
query_timeout = 30
transaction_timeout = 120
idle_transaction_timeout = 30
client_idle_timeout = 600

; Server connection lifecycle
server_lifetime = 3600
server_idle_timeout = 600
```

```
ประหยัดเท่าไหร่จาก connection pooling:

ก่อน (without pooler):
- 100 app instances x 10 connections each = 1,000 connections
- PostgreSQL overhead: ~5MB per connection
- Total: 5GB RAM เฉพาะ connections

หลัง (with PgBouncer):
- 1,000 app connections → 20 actual DB connections
- RAM สำหรับ connections: 100MB
- ประหยัด: ~4.9GB RAM
- อาจ downsize จาก r6g.2xlarge เป็น r6g.xlarge = ~$300/month savings
```

### 2.4 Efficient Indexes: ลบ Index ที่ไม่ได้ใช้

```sql
-- หา indexes ที่ไม่ได้ใช้เลย
SELECT
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    idx_scan AS times_used
FROM pg_stat_user_indexes
WHERE idx_scan = 0
AND NOT EXISTS (
    SELECT 1 FROM pg_constraint c
    WHERE c.conindid = indexrelid
)
ORDER BY pg_relation_size(indexrelid) DESC;

-- หา duplicate indexes (indexes ที่ซ้ำกัน)
SELECT
    a.indrelid::regclass AS table_name,
    a.indexrelid::regclass AS index1,
    b.indexrelid::regclass AS index2,
    pg_size_pretty(pg_relation_size(a.indexrelid)) AS size1,
    pg_size_pretty(pg_relation_size(b.indexrelid)) AS size2
FROM pg_index a
JOIN pg_index b ON a.indrelid = b.indrelid
    AND a.indexrelid < b.indexrelid
    AND a.indkey = b.indkey
ORDER BY a.indrelid;

-- ลบ unused indexes (ระวัง! ตรวจสอบก่อนลบ)
-- DROP INDEX CONCURRENTLY idx_that_is_unused;

-- สร้าง index ที่ดีกว่า
-- ✅ Partial index: ลดขนาด index
CREATE INDEX CONCURRENTLY idx_orders_pending
    ON orders(created_at)
    WHERE status = 'pending'; -- เฉพาะ pending orders

-- ✅ Covering index: หลีกเลี่ยง table lookup
CREATE INDEX CONCURRENTLY idx_users_email_role
    ON users(email) INCLUDE (id, role, tenant_id);
```

### 2.5 Partition Archiving: Cold Data ไปยัง Cheap Storage

```sql
-- Partition orders by month
CREATE TABLE orders (
    id UUID NOT NULL DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    amount DECIMAL(10,2),
    status TEXT,
    created_at TIMESTAMPTZ NOT NULL
) PARTITION BY RANGE (created_at);

-- Hot partition: current month (SSD, high IOPS)
CREATE TABLE orders_2024_11 PARTITION OF orders
    FOR VALUES FROM ('2024-11-01') TO ('2024-12-01')
    TABLESPACE fast_ssd;

CREATE TABLE orders_2024_10 PARTITION OF orders
    FOR VALUES FROM ('2024-10-01') TO ('2024-11-01')
    TABLESPACE fast_ssd;

-- Warm partition: last 6 months
CREATE TABLE orders_2024_05 PARTITION OF orders
    FOR VALUES FROM ('2024-05-01') TO ('2024-06-01')
    TABLESPACE standard_hdd;

-- Cold data: archive to cheaper storage
-- Option 1: Detach old partition and move to pg_dump/S3
-- Option 2: Use tablespace on cheaper disk

-- Tablespaces
CREATE TABLESPACE fast_ssd LOCATION '/mnt/nvme/postgres';
CREATE TABLESPACE standard_hdd LOCATION '/mnt/hdd/postgres';

-- Archive script
CREATE OR REPLACE PROCEDURE maintenance.archive_old_partitions()
LANGUAGE plpgsql AS $$
DECLARE
    v_partition TEXT;
    v_archive_before DATE := CURRENT_DATE - INTERVAL '6 months';
BEGIN
    FOR v_partition IN
        SELECT tablename
        FROM pg_tables
        WHERE tablename LIKE 'orders_%'
        AND tablename < 'orders_' || to_char(v_archive_before, 'YYYY_MM')
    LOOP
        -- Move to cheaper tablespace
        EXECUTE format(
            'ALTER TABLE %I SET TABLESPACE standard_hdd',
            v_partition
        );
        
        RAISE NOTICE 'Moved % to standard_hdd', v_partition;
    END LOOP;
END;
$$;
```

### 2.6 Table Compression: ลดขนาด Storage

```sql
-- TOAST compression (automatic)
-- ข้อมูล > 2KB จะถูก compress อัตโนมัติ

-- ดูการใช้ TOAST
SELECT
    relname AS table,
    pg_size_pretty(pg_total_relation_size(oid)) AS total_size,
    pg_size_pretty(pg_relation_size(oid)) AS table_size,
    pg_size_pretty(pg_total_relation_size(oid) - pg_relation_size(oid)) AS toast_and_index_size
FROM pg_class
WHERE relkind = 'r'
ORDER BY pg_total_relation_size(oid) DESC
LIMIT 10;

-- Storage compression settings per column
ALTER TABLE documents
    ALTER COLUMN content SET STORAGE EXTENDED;
    -- PLAIN: no compression
    -- EXTERNAL: no compression, out-of-line
    -- EXTENDED: compressed, out-of-line (default for large columns)
    -- MAIN: compressed, in-line

-- ดู compression ratio
SELECT
    tablename,
    attname AS column,
    attstorage AS storage_type,
    pg_size_pretty(avg(pg_column_size(content))::bigint) AS avg_size
FROM pg_attribute
JOIN pg_class ON pg_class.oid = pg_attribute.attrelid
JOIN pg_tables ON tablename = pg_class.relname
WHERE attname = 'content'
GROUP BY tablename, attname, attstorage;

-- สำหรับ JSON columns ที่ใหญ่
-- ลอง compress ก่อน insert
-- Application level:
-- const compressed = zlib.gzipSync(JSON.stringify(data));
-- await db.query('INSERT INTO logs (data) VALUES ($1)', [compressed]);
```

### 2.7 Vacuum Tuning: ลด Bloat

```sql
-- ดู table bloat
SELECT
    tablename,
    pg_size_pretty(pg_relation_size(tablename::regclass)) AS table_size,
    n_dead_tup,
    n_live_tup,
    round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_pct,
    last_autovacuum,
    last_autoanalyze
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC
LIMIT 20;

-- Tune autovacuum สำหรับ high-write tables
ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor = 0.01,  -- vacuum เมื่อ 1% dead
    autovacuum_analyze_scale_factor = 0.01,
    autovacuum_vacuum_cost_delay = 2        -- ลด delay (aggressive)
);

-- สำหรับ tables ที่ write เยอะมาก
ALTER TABLE events SET (
    autovacuum_vacuum_scale_factor = 0.001, -- 0.1% dead = vacuum
    autovacuum_vacuum_threshold = 100,
    autovacuum_analyze_threshold = 100
);

-- Manual vacuum ถ้า bloat มาก
VACUUM (VERBOSE, ANALYZE) orders;

-- VACUUM FULL (reclaim disk space - ระวัง! lock table)
-- ใช้เฉพาะ maintenance window
VACUUM FULL orders_2024_01;
```

### 2.8 Query Optimization: ลด CPU/IO

```sql
-- หา slow queries
SELECT
    left(query, 100) AS query_preview,
    calls,
    round(total_exec_time::numeric / calls, 2) AS avg_ms,
    round(total_exec_time::numeric, 2) AS total_ms,
    round((shared_blks_hit + shared_blks_read)::numeric / calls, 2) AS avg_blocks,
    rows / calls AS avg_rows
FROM pg_stat_statements
WHERE calls > 100
ORDER BY total_exec_time DESC
LIMIT 20;

-- Optimize specific query
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT u.*, p.*
FROM users u
JOIN profiles p ON p.user_id = u.id
WHERE u.tenant_id = '550e8400...'
AND u.is_active = true
ORDER BY u.created_at DESC
LIMIT 10;

-- ถ้าเห็น Seq Scan = ต้องการ index
-- ถ้าเห็น Hash Join แทน Index Join = อาจต้องการ covering index

-- Query rewrite ตัวอย่าง
-- ❌ ช้า: correlated subquery
SELECT id, (
    SELECT count(*) FROM orders WHERE orders.user_id = users.id
) AS order_count
FROM users;

-- ✅ เร็ว: join
SELECT u.id, count(o.id) AS order_count
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
GROUP BY u.id;
```

---

## 3. Redis Cost Optimization

### 3.1 Memory Compression: ใช้ Efficient Data Structures

```redis
# ตรวจสอบ memory usage
INFO memory
MEMORY DOCTOR

# ดู memory per key type
MEMORY USAGE mykey
```

```javascript
// ใช้ compact data structures

// ❌ ไม่ดี: เก็บ JSON string ทุก key
await redis.set('user:1234:name', 'John Doe');
await redis.set('user:1234:email', 'john@example.com');
await redis.set('user:1234:role', 'admin');

// ✅ ดี: ใช้ Hash (listpack encoding สำหรับ small hash)
await redis.hset('user:1234', {
  name: 'John Doe',
  email: 'john@example.com',
  role: 'admin',
});

// ✅ ดี: Serialize ด้วย MessagePack (compact กว่า JSON)
const msgpack = require('msgpackr');
const packed = msgpack.pack({ name: 'John', role: 'admin' });
await redis.set('user:1234', packed);

// Unpack
const data = msgpack.unpack(await redis.getBuffer('user:1234'));
```

```bash
# redis.conf: ใช้ listpack สำหรับ small structures
hash-max-listpack-entries 128
hash-max-listpack-value 64

zset-max-listpack-entries 128
zset-max-listpack-value 64

set-max-intset-entries 512
list-max-listpack-size -2  # 8kb max per node
```

### 3.2 TTL: Expire Unused Data

```javascript
// ✅ ทุก cache entry ต้องมี TTL
const CACHE_TTL = {
  userProfile: 60 * 60,        // 1 hour
  sessionData: 24 * 60 * 60,   // 1 day
  productList: 15 * 60,        // 15 minutes
  searchResults: 5 * 60,       // 5 minutes
  rateLimit: 60,               // 1 minute
};

async function cacheSet(key, data, ttlSeconds) {
  if (!ttlSeconds) throw new Error('TTL is required for all cache entries');
  return redis.setex(key, ttlSeconds, JSON.stringify(data));
}

// Monitor keys without TTL (memory leak!)
// redis-cli: SCAN 0 COUNT 100 TYPE string
// แล้วตรวจสอบ TTL ของแต่ละ key
```

```bash
# Script: หา keys ที่ไม่มี TTL
redis-cli --scan --pattern '*' | while read key; do
    ttl=$(redis-cli TTL "$key")
    if [ "$ttl" -eq -1 ]; then
        echo "No TTL: $key ($(redis-cli TYPE $key))"
    fi
done | head -50
```

### 3.3 Eviction Policy

```bash
# redis.conf
# maxmemory: กำหนด memory limit
maxmemory 2gb

# eviction policy เมื่อ memory เต็ม
# allkeys-lru: ลบ keys ที่ไม่ได้ใช้นานสุด (แนะนำสำหรับ cache)
# volatile-lru: ลบเฉพาะ keys ที่มี TTL
# allkeys-lfu: ลบ keys ที่ใช้น้อยที่สุด (LFU)
# noeviction: error เมื่อ memory เต็ม (ไม่แนะนำสำหรับ cache)
maxmemory-policy allkeys-lru

# LFU parameters
lfu-log-factor 10
lfu-decay-time 1
```

### 3.4 Data Structure Choice: ขนาดเล็กกว่า

```javascript
// ตัวอย่าง: Session storage

// ❌ String (uses more memory)
await redis.set(`session:${id}`, JSON.stringify({
  userId: 'uuid-here',
  role: 'admin',
  tenantId: 'tenant-uuid',
  permissions: ['read', 'write', 'admin'],
}));

// ✅ Hash (listpack encoding ถ้า < 128 fields, < 64 bytes per value)
await redis.hset(`session:${id}`, {
  u: 'uuid-here',     // short field names
  r: 'admin',
  t: 'tenant-uuid',
  p: 'r,w,a',         // compact permissions
});
await redis.expire(`session:${id}`, 86400);

// ✅ MessagePack + String (ดีสำหรับ large objects)
const msgpack = require('msgpackr');
const packed = msgpack.pack({
  userId: 'uuid-here',
  role: 'admin',
  tenantId: 'tenant-uuid',
});
// MessagePack ลดขนาดได้ ~40% เทียบกับ JSON
await redis.setex(`session:${id}`, 86400, packed);
```

### 3.5 Cluster Sharding: Distribute Load

```javascript
// Redis Cluster: distribute load across nodes
const Redis = require('ioredis');

const cluster = new Redis.Cluster([
  { host: 'redis-node-1', port: 6379 },
  { host: 'redis-node-2', port: 6379 },
  { host: 'redis-node-3', port: 6379 },
], {
  // Hash tags: force related keys to same node
  // {tenant_id}.users:123 → same node as {tenant_id}.orders:456
  enableReadyCheck: true,
  scaleReads: 'slave',  // Route reads to replicas
});

// Cost: 3 nodes แทน 1 ใหญ่ = ถูกกว่าและ HA ดีกว่า
// r6g.large x3 = ~$600/month
// r6g.2xlarge x1 = ~$400/month (cheaper but no HA)
```

---

## 4. S3/Object Storage Cost Optimization

### 4.1 Storage Tiers

```javascript
// S3 Storage Classes และ ราคาโดยประมาณ (ap-southeast-1):
// Standard:              $0.025/GB/month
// Standard-IA:           $0.0138/GB/month (ประหยัด 45%)
// One Zone-IA:           $0.011/GB/month (ประหยัด 56%)
// Glacier Instant:       $0.005/GB/month (ประหยัด 80%)
// Glacier Flexible:      $0.0045/GB/month
// Glacier Deep Archive:  $0.0018/GB/month (ประหยัด 93%)

// Upload with appropriate storage class
const { S3Client, PutObjectCommand } = require('@aws-sdk/client-s3');

async function uploadFile(key, data, options = {}) {
  const { storageClass = 'STANDARD', contentType } = options;
  
  return s3.send(new PutObjectCommand({
    Bucket: process.env.S3_BUCKET,
    Key: key,
    Body: data,
    ContentType: contentType,
    StorageClass: storageClass,
    // StorageClass options:
    // STANDARD, STANDARD_IA, ONEZONE_IA, GLACIER_IR,
    // GLACIER, DEEP_ARCHIVE, INTELLIGENT_TIERING
  }));
}

// Hot data: user uploads
await uploadFile('uploads/user-avatar.jpg', imageData, {
  storageClass: 'STANDARD',
  contentType: 'image/jpeg',
});

// Warm data: reports (30+ days old)
await uploadFile('reports/2024-09-report.pdf', pdfData, {
  storageClass: 'STANDARD_IA',
  contentType: 'application/pdf',
});

// Cold data: backups
await uploadFile('backups/db-2024-01.sql.gz', backupData, {
  storageClass: 'GLACIER_IR',
  contentType: 'application/gzip',
});
```

### 4.2 Lifecycle Policies

```hcl
# terraform: S3 Lifecycle Rules
resource "aws_s3_bucket_lifecycle_configuration" "main" {
  bucket = aws_s3_bucket.main.id
  
  # User uploads: transition to IA after 30 days, archive after 1 year
  rule {
    id     = "user-uploads-lifecycle"
    status = "Enabled"
    
    filter {
      prefix = "uploads/"
    }
    
    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }
    
    transition {
      days          = 365
      storage_class = "GLACIER_INSTANT_RETRIEVAL"
    }
    
    expiration {
      days = 2555  # 7 years, then delete
    }
  }
  
  # Logs: fast transition to cheap storage
  rule {
    id     = "logs-lifecycle"
    status = "Enabled"
    
    filter {
      prefix = "logs/"
    }
    
    transition {
      days          = 7
      storage_class = "STANDARD_IA"
    }
    
    transition {
      days          = 30
      storage_class = "GLACIER"
    }
    
    expiration {
      days = 365  # Delete after 1 year
    }
  }
  
  # Database backups: go straight to glacier
  rule {
    id     = "db-backups"
    status = "Enabled"
    
    filter {
      prefix = "backups/"
    }
    
    transition {
      days          = 1
      storage_class = "GLACIER_INSTANT_RETRIEVAL"
    }
    
    transition {
      days          = 90
      storage_class = "DEEP_ARCHIVE"
    }
    
    expiration {
      days = 2555  # 7 years
    }
  }
  
  # Incomplete multipart uploads: clean up
  rule {
    id     = "abort-incomplete-multipart"
    status = "Enabled"
    
    abort_incomplete_multipart_upload {
      days_after_initiation = 7
    }
  }
}
```

### 4.3 Compression ก่อน Upload

```javascript
// compress-upload.js
const zlib = require('zlib');
const { promisify } = require('util');

const gzip = promisify(zlib.gzip);
const brotliCompress = promisify(zlib.brotliCompress);

async function compressAndUpload(key, data, options = {}) {
  const { compress = 'gzip', contentType = 'application/octet-stream' } = options;
  
  let compressedData;
  let contentEncoding;
  
  switch (compress) {
    case 'gzip':
      compressedData = await gzip(data, { level: 9 });
      contentEncoding = 'gzip';
      break;
    case 'brotli':
      compressedData = await brotliCompress(data, {
        params: { [zlib.constants.BROTLI_PARAM_QUALITY]: 11 }
      });
      contentEncoding = 'br';
      break;
    default:
      compressedData = data;
  }
  
  const originalSize = Buffer.byteLength(data);
  const compressedSize = Buffer.byteLength(compressedData);
  const savings = ((1 - compressedSize / originalSize) * 100).toFixed(1);
  
  console.log(`Compressed ${originalSize} → ${compressedSize} bytes (${savings}% savings)`);
  
  return s3.send(new PutObjectCommand({
    Bucket: process.env.S3_BUCKET,
    Key: key,
    Body: compressedData,
    ContentType: contentType,
    ContentEncoding: contentEncoding,
    Metadata: {
      'original-size': originalSize.toString(),
    },
  }));
}

// JSON logs: compress saves ~90%
await compressAndUpload('logs/app-2024-11.json', logsJson, {
  compress: 'brotli',
  contentType: 'application/json',
});

// Database dumps: compress saves ~60-80%
await compressAndUpload('backups/db-2024-11-01.sql', sqlDump, {
  compress: 'gzip',
  contentType: 'application/sql',
});
```

### 4.4 CloudFront: ลด Origin Requests

```hcl
# CloudFront distribution สำหรับ S3
resource "aws_cloudfront_distribution" "assets" {
  origin {
    domain_name = aws_s3_bucket.assets.bucket_regional_domain_name
    origin_id   = "S3-${aws_s3_bucket.assets.id}"
    
    s3_origin_config {
      origin_access_identity = aws_cloudfront_origin_access_identity.main.cloudfront_access_identity_path
    }
  }
  
  enabled             = true
  default_root_object = "index.html"
  
  default_cache_behavior {
    allowed_methods  = ["GET", "HEAD"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "S3-${aws_s3_bucket.assets.id}"
    
    forwarded_values {
      query_string = false
      cookies { forward = "none" }
    }
    
    viewer_protocol_policy = "redirect-to-https"
    
    # Cache for 1 year (immutable assets)
    min_ttl     = 0
    default_ttl = 31536000
    max_ttl     = 31536000
    
    compress = true  # Gzip/Brotli automatically
  }
  
  # Cache for dynamic files (shorter TTL)
  ordered_cache_behavior {
    path_pattern     = "/uploads/*"
    allowed_methods  = ["GET", "HEAD"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "S3-${aws_s3_bucket.assets.id}"
    
    forwarded_values {
      query_string = false
      cookies { forward = "none" }
    }
    
    viewer_protocol_policy = "redirect-to-https"
    
    min_ttl     = 0
    default_ttl = 86400   # 1 day
    max_ttl     = 604800  # 1 week
    
    compress = true
  }
  
  price_class = "PriceClass_200"  # Exclude expensive regions
  
  restrictions {
    geo_restriction { restriction_type = "none" }
  }
  
  viewer_certificate {
    cloudfront_default_certificate = false
    acm_certificate_arn            = aws_acm_certificate.main.arn
    ssl_support_method             = "sni-only"
    minimum_protocol_version       = "TLSv1.2_2021"
  }
  
  tags = { Project = "myapp" }
}
```

---

## 5. Cloud Cost Optimization

### 5.1 Reserved Instances

```python
# cost_calculator.py
# เปรียบเทียบ On-Demand vs Reserved

instance_type = 'db.r6g.2xlarge'

# On-Demand pricing (ap-southeast-1 approximation)
on_demand_hourly = 0.96  # USD/hour
on_demand_monthly = on_demand_hourly * 24 * 30
on_demand_annual = on_demand_hourly * 24 * 365

# Reserved pricing
reserved_1yr_upfront = 0  # No Upfront
reserved_1yr_hourly = 0.58  # ~40% discount
reserved_1yr_monthly = reserved_1yr_hourly * 24 * 30
reserved_1yr_annual = reserved_1yr_hourly * 24 * 365

# All Upfront
reserved_3yr_upfront = 9_500  # All Upfront
reserved_3yr_monthly_effective = reserved_3yr_upfront / 36

savings_1yr = (on_demand_annual - reserved_1yr_annual) 
savings_pct_1yr = savings_1yr / on_demand_annual * 100

print(f"Instance: {instance_type}")
print(f"On-Demand: ${on_demand_monthly:.0f}/month (${on_demand_annual:.0f}/year)")
print(f"1yr Reserved: ${reserved_1yr_monthly:.0f}/month (${reserved_1yr_annual:.0f}/year)")
print(f"Savings: ${savings_1yr:.0f}/year ({savings_pct_1yr:.0f}%)")
print(f"3yr All Upfront: ${reserved_3yr_monthly_effective:.0f}/month effective")
```

```
Output:
Instance: db.r6g.2xlarge
On-Demand: $691/month ($8,294/year)
1yr Reserved: $418/month ($5,010/year)  
Savings: $3,284/year (40%)
3yr All Upfront: $264/month effective
```

### 5.2 Spot Instances สำหรับ Non-critical Workloads

```python
# use_cases_spot.py

use_cases = {
    "Development/Test DBs": {
        "spot_suitable": True,
        "reason": "สามารถ restart ได้ ไม่กระทบ production",
        "savings": "60-80%"
    },
    "Batch Processing": {
        "spot_suitable": True,
        "reason": "งาน batch ทนต่อการ interrupt ได้",
        "savings": "60-80%"
    },
    "Read Replicas": {
        "spot_suitable": True,
        "reason": "ถ้า replica ตาย traffic ไป primary ได้",
        "savings": "60-80%"
    },
    "Primary Database": {
        "spot_suitable": False,
        "reason": "ต้องการ high availability, spot อาจถูก terminate",
        "alternative": "Reserved Instance"
    },
    "Session Cache (Redis)": {
        "spot_suitable": False,
        "reason": "Users จะ logout ทันทีถ้า cache หาย",
        "alternative": "ElastiCache with Reserved Nodes"
    }
}
```

```yaml
# k8s: Run analytics job on Spot instances
apiVersion: batch/v1
kind: Job
metadata:
  name: monthly-analytics
spec:
  template:
    spec:
      # Use Spot instances
      nodeSelector:
        node.kubernetes.io/lifecycle: spot
      tolerations:
        - key: "spot"
          operator: "Equal"
          value: "true"
          effect: "NoSchedule"
      
      # Handle spot interruption
      terminationGracePeriodSeconds: 120
      
      containers:
        - name: analytics
          image: myapp/analytics:latest
          resources:
            requests:
              cpu: "2"
              memory: "4Gi"
          env:
            - name: CHECKPOINT_ENABLED
              value: "true"
            - name: S3_CHECKPOINT_BUCKET
              value: "myapp-analytics-checkpoints"
      
      restartPolicy: OnFailure
```

### 5.3 Multi-AZ vs Single-AZ Tradeoffs

```
Cost Analysis: Multi-AZ vs Single-AZ

Multi-AZ RDS:
- Benefit: Automatic failover, ~99.95% availability
- Cost: 2x instance cost (standby replica)
- Additional: Cross-AZ data transfer ~$0.01/GB

Single-AZ RDS:
- Risk: ~1-2 hour RTO on hardware failure
- Cost: Half of Multi-AZ
- Risk in $: If 1 hour downtime = $X revenue loss

Decision formula:
Multi-AZ is worth it if:
  (Annual downtime cost) > (Additional annual cost of Multi-AZ)

Example:
  Revenue: $100,000/month
  Expected downtime: 4 hours/year (without Multi-AZ)
  Downtime cost: $100,000 / (30 * 24) * 4 = $556
  Additional Multi-AZ cost: $300/month * 12 = $3,600/year
  
  → Single-AZ is cheaper if downtime < 1 hour/month
  → Multi-AZ worth it if business can't tolerate any downtime
```

### 5.4 Development vs Production Sizing

```hcl
# terraform: Different sizes for different environments

variable "environment" {
  default = "development"
}

locals {
  db_config = {
    development = {
      instance_class    = "db.t3.micro"
      allocated_storage = 20
      multi_az          = false
      deletion_protection = false
    }
    staging = {
      instance_class    = "db.t3.medium"
      allocated_storage = 100
      multi_az          = false
      deletion_protection = false
    }
    production = {
      instance_class    = "db.r6g.2xlarge"
      allocated_storage = 500
      multi_az          = true
      deletion_protection = true
    }
  }
}

resource "aws_db_instance" "main" {
  identifier     = "myapp-${var.environment}"
  instance_class = local.db_config[var.environment].instance_class
  allocated_storage = local.db_config[var.environment].allocated_storage
  multi_az      = local.db_config[var.environment].multi_az
  
  # Stop dev/staging at night (saves ~$50/month per instance)
  # ใช้ AWS Instance Scheduler หรือ Lambda
}
```

```python
# lambda: Stop dev databases at night (18:00-08:00)
import boto3
import os

def handler(event, context):
    rds = boto3.client('rds')
    
    action = event.get('action', 'stop')  # stop or start
    env_filter = ['development', 'staging']
    
    instances = rds.describe_db_instances()['DBInstances']
    
    for instance in instances:
        tags = {t['Key']: t['Value'] for t in instance.get('TagList', [])}
        
        if tags.get('Environment') in env_filter:
            db_id = instance['DBInstanceIdentifier']
            
            if action == 'stop' and instance['DBInstanceStatus'] == 'available':
                print(f"Stopping {db_id}")
                rds.stop_db_instance(DBInstanceIdentifier=db_id)
            elif action == 'start' and instance['DBInstanceStatus'] == 'stopped':
                print(f"Starting {db_id}")
                rds.start_db_instance(DBInstanceIdentifier=db_id)

# EventBridge rules:
# Stop: cron(0 11 * * ? *)  = 18:00 Bangkok time (UTC+7)
# Start: cron(0 1 * * ? *)  = 08:00 Bangkok time
```

---

## 6. Monitoring Costs

### 6.1 AWS Cost Explorer

```python
# cost_analysis.py
import boto3
from datetime import datetime, timedelta

ce = boto3.client('ce')

def get_database_costs(days=30):
    end = datetime.now().strftime('%Y-%m-%d')
    start = (datetime.now() - timedelta(days=days)).strftime('%Y-%m-%d')
    
    response = ce.get_cost_and_usage(
        TimePeriod={'Start': start, 'End': end},
        Granularity='MONTHLY',
        Filter={
            'Or': [
                {'Dimensions': {'Key': 'SERVICE', 'Values': ['Amazon RDS']}},
                {'Dimensions': {'Key': 'SERVICE', 'Values': ['Amazon ElastiCache']}},
                {'Dimensions': {'Key': 'SERVICE', 'Values': ['Amazon S3']}},
            ]
        },
        GroupBy=[
            {'Type': 'DIMENSION', 'Key': 'SERVICE'},
            {'Type': 'TAG', 'Key': 'Environment'},
        ],
        Metrics=['BlendedCost', 'UsageQuantity']
    )
    
    results = []
    for period in response['ResultsByTime']:
        for group in period['Groups']:
            service = group['Keys'][0]
            environment = group['Keys'][1] if len(group['Keys']) > 1 else 'untagged'
            cost = float(group['Metrics']['BlendedCost']['Amount'])
            
            results.append({
                'period': period['TimePeriod']['Start'],
                'service': service,
                'environment': environment,
                'cost_usd': cost,
            })
    
    return results

costs = get_database_costs(days=90)
for item in costs:
    print(f"{item['period']} | {item['service']:30s} | {item['environment']:15s} | ${item['cost_usd']:.2f}")
```

### 6.2 Budget Alerts

```hcl
# terraform: AWS Budget with alerts
resource "aws_budgets_budget" "database_monthly" {
  name         = "database-monthly-budget"
  budget_type  = "COST"
  limit_amount = "2000"
  limit_unit   = "USD"
  time_unit    = "MONTHLY"
  
  cost_filter {
    name   = "Service"
    values = ["Amazon RDS", "Amazon ElastiCache", "Amazon S3"]
  }
  
  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 80
    threshold_type             = "PERCENTAGE"
    notification_type          = "ACTUAL"
    subscriber_email_addresses = ["dba@company.com", "cto@company.com"]
  }
  
  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 100
    threshold_type             = "PERCENTAGE"
    notification_type          = "FORECASTED"
    subscriber_email_addresses = ["dba@company.com"]
  }
}
```

### 6.3 Unit Economics: Cost per User/Request

```javascript
// cost-analytics.js
class CostAnalytics {
  async calculateUnitCosts(period = 'monthly') {
    const [
      monthlyCosts,
      userCount,
      requestCount,
      revenueData,
    ] = await Promise.all([
      this.getMonthlyCosts(),
      this.getActiveUserCount(),
      this.getRequestCount(),
      this.getRevenue(),
    ]);
    
    const { dbCost, cacheCost, storageCost } = monthlyCosts;
    const totalInfraCost = dbCost + cacheCost + storageCost;
    
    return {
      period,
      costs: {
        database: dbCost,
        cache: cacheCost,
        storage: storageCost,
        total: totalInfraCost,
      },
      metrics: {
        activeUsers: userCount,
        requests: requestCount,
        revenue: revenueData.total,
      },
      unitEconomics: {
        costPerUser: totalInfraCost / userCount,
        costPerRequest: totalInfraCost / requestCount,
        costPerDollarRevenue: totalInfraCost / revenueData.total,
        infraMargin: ((revenueData.total - totalInfraCost) / revenueData.total * 100).toFixed(1),
      },
    };
  }
}

// Target unit economics for healthy SaaS:
// Cost per user: < 10% of ARPU (Average Revenue Per User)
// Infrastructure margin: > 60%
// Cost per request: < $0.0001
```

---

## 7. Open Source Alternatives

```markdown
## Commercial → Open Source Alternatives

| Commercial Service | Open Source Alternative | Monthly Savings |
|-------------------|------------------------|-----------------|
| AWS RDS PostgreSQL | Self-managed PostgreSQL on EC2 | 30-60% |
| AWS ElastiCache | Self-managed Redis on EC2 | 40-60% |
| Datadog ($500/mo) | Prometheus + Grafana (free) | $500/mo |
| PagerDuty ($200/mo) | AlertManager + OpsGenie free tier | $200/mo |
| AWS Secrets Manager | HashiCorp Vault (self-hosted) | $50-200/mo |
| AWS WAF ($100/mo) | ModSecurity/Nginx rate limiting | $100/mo |

แต่ต้องคิดถึง:
- Operations cost (DBA/DevOps time)
- Hidden costs (HA setup, monitoring)
- Risk: self-managed = responsible for availability
```

---

## 8. Full Cost Analysis Worksheet

```python
#!/usr/bin/env python3
# database_cost_calculator.py

class DatabaseCostCalculator:
    def __init__(self):
        # AWS ap-southeast-1 approximate pricing
        self.pricing = {
            # RDS PostgreSQL
            'rds_db.t3.micro': 0.025,      # $/hour
            'rds_db.t3.medium': 0.095,
            'rds_db.t3.large': 0.19,
            'rds_db.r6g.large': 0.24,
            'rds_db.r6g.xlarge': 0.48,
            'rds_db.r6g.2xlarge': 0.96,
            'rds_db.r6g.4xlarge': 1.92,
            
            # Storage
            'rds_storage_gp3': 0.115,      # $/GB/month
            'rds_storage_io1': 0.125,       # $/GB/month
            'rds_iops': 0.10,               # $/IOPS/month (io1)
            
            # ElastiCache Redis
            'redis_cache.t3.micro': 0.017,
            'redis_cache.t3.medium': 0.068,
            'redis_cache.r6g.large': 0.187,
            'redis_cache.r6g.xlarge': 0.374,
            
            # S3
            's3_standard': 0.025,           # $/GB/month
            's3_ia': 0.0138,
            's3_glacier_ir': 0.005,
            
            # Data Transfer
            'data_transfer_out': 0.09,      # $/GB (first 1TB)
            'cross_az': 0.01,               # $/GB
        }
    
    def calculate_monthly(self, config):
        """
        config = {
            'primary_instance': 'rds_db.r6g.xlarge',
            'read_replicas': 2,
            'replica_instance': 'rds_db.r6g.large',
            'storage_gb': 500,
            'storage_type': 'gp3',
            'multi_az': True,
            'redis_instance': 'redis_cache.r6g.large',
            'redis_replicas': 1,
            's3_storage_gb': 2000,
            's3_requests_per_month': 10_000_000,
            'data_transfer_gb': 500,
        }
        """
        hours_per_month = 24 * 30
        costs = {}
        
        # Primary RDS
        primary_hourly = self.pricing[f"rds_{config['primary_instance']}"]
        costs['primary_rds'] = primary_hourly * hours_per_month
        
        # Multi-AZ doubles the instance cost
        if config.get('multi_az'):
            costs['primary_rds'] *= 2
        
        # Read Replicas
        replica_hourly = self.pricing[f"rds_{config['replica_instance']}"]
        costs['read_replicas'] = (
            replica_hourly * 
            hours_per_month * 
            config.get('read_replicas', 0)
        )
        
        # Storage
        storage_cost_per_gb = self.pricing[f"rds_storage_{config['storage_type']}"]
        costs['storage'] = storage_cost_per_gb * config['storage_gb']
        
        # Redis
        if config.get('redis_instance'):
            redis_hourly = self.pricing[f"redis_{config['redis_instance']}"]
            costs['redis'] = redis_hourly * hours_per_month * (1 + config.get('redis_replicas', 0))
        
        # S3
        costs['s3'] = self.pricing['s3_standard'] * config.get('s3_storage_gb', 0)
        
        # Data Transfer
        costs['data_transfer'] = self.pricing['data_transfer_out'] * config.get('data_transfer_gb', 0)
        
        costs['total'] = sum(costs.values())
        
        return costs
    
    def print_report(self, config, name="Configuration"):
        costs = self.calculate_monthly(config)
        
        print(f"\n{'='*50}")
        print(f"Monthly Cost Report: {name}")
        print(f"{'='*50}")
        
        for component, cost in costs.items():
            if component != 'total':
                print(f"  {component:25s}: ${cost:8.2f}")
        
        print(f"  {'':25s}  {'--------'}")
        print(f"  {'TOTAL':25s}: ${costs['total']:8.2f}")
        print(f"  Annual cost         : ${costs['total'] * 12:,.2f}")

# Example calculations
calc = DatabaseCostCalculator()

# Current setup
current = {
    'primary_instance': 'db.r6g.2xlarge',
    'read_replicas': 2,
    'replica_instance': 'db.r6g.xlarge',
    'storage_gb': 500,
    'storage_type': 'gp3',
    'multi_az': True,
    'redis_instance': 'cache.r6g.large',
    'redis_replicas': 1,
    's3_storage_gb': 2000,
    'data_transfer_gb': 500,
}

# Optimized setup
optimized = {
    'primary_instance': 'db.r6g.xlarge',    # Downsize
    'read_replicas': 1,                       # 1 less replica
    'replica_instance': 'db.r6g.large',      # Smaller replicas
    'storage_gb': 300,                        # Optimized storage
    'storage_type': 'gp3',
    'multi_az': True,
    'redis_instance': 'cache.r6g.large',
    'redis_replicas': 1,
    's3_storage_gb': 2000,
    'data_transfer_gb': 300,                  # CloudFront reduces origin traffic
}

calc.print_report(current, "Current Setup")
calc.print_report(optimized, "Optimized Setup")
```

---

## 9. สรุป Cost Optimization Priorities

```markdown
## Priority Matrix

### ทำทันที (Quick Wins)
1. ลบ unused indexes → ประหยัด storage + เพิ่ม write performance
2. Enable compression สำหรับ S3 → ประหยัด 40-90% storage
3. S3 Lifecycle rules → ประหยัด 40-80% storage cost
4. TTL ใน Redis → ป้องกัน memory waste
5. Stop dev/staging instances at night → ประหยัด ~65% compute

### Medium Term (1-3 เดือน)
1. Connection pooling (PgBouncer) → อาจ downsize DB instance
2. Read replica offloading → ลด load บน primary
3. Query optimization → ลด CPU time
4. Reserved Instances → 40% savings
5. CloudFront CDN → ลด S3 requests + data transfer

### Long Term (3-12 เดือน)
1. Table partitioning + archiving
2. Audit และ remove bloat regularly
3. Evaluate multi-tenant vs single-tenant storage
4. Consider managed service vs self-hosted tradeoffs
5. Unit economics tracking + optimization

### Expected Savings
| Initiative | Monthly Savings | Effort |
|-----------|----------------|--------|
| Reserved Instances | $300-$1,000 | Low |
| Night shutdown dev/staging | $100-$500 | Low |
| S3 Lifecycle | $50-$200 | Low |
| Connection pooling | $200-$800 | Medium |
| Query optimization | $100-$500 | High |
| Index cleanup | $20-$100 | Medium |
```

---

*เนื้อหานี้เป็นส่วนหนึ่งของ Database Cluster Course - World-Class Level*
*Part 94/100: Cost Optimization สำหรับ Database Cluster*

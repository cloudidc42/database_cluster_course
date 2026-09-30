# Part 57: Metrics ด้วย Prometheus + Grafana

## บทนำ: ทำไมต้องมี Metrics?

ในขณะที่ Distributed Tracing บอกเราว่า "request นี้ใช้เวลาเท่าไหร่และผ่านอะไรบ้าง" **Metrics** บอกเราว่า "ระบบโดยรวมมีสุขภาพดีแค่ไหน" ตลอดเวลา

```
Logs   → "เกิดอะไรขึ้น?" (individual events)
Traces → "ทำไมถึงช้า?" (request path)
Metrics → "ระบบสุขภาพดีแค่ไหน?" (aggregated numbers over time)
```

### Metrics ที่ควรติดตามใน Database Cluster

```
Application Metrics:
├── Request Rate: requests/second
├── Error Rate: errors/second
├── Latency P50, P95, P99
└── Active Connections

PostgreSQL Metrics:
├── Active Connections / Max Connections
├── Query Duration (slow queries)
├── Replication Lag
├── Cache Hit Ratio
├── Dead Tuples / Autovacuum
└── Disk Space Usage

Redis Metrics:
├── Memory Usage / Max Memory
├── Hit Rate (keyspace_hits / total)
├── Operations/Second
├── Connected Clients
├── Eviction Rate
└── Replication Offset Lag

Infrastructure Metrics:
├── CPU Usage
├── Memory Usage
├── Disk I/O
└── Network I/O
```

---

## Prometheus: Pull-based Metrics Collection

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Prometheus                          │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                 Scrape Engine                       │   │
│  │  Every 15s → Pull /metrics from targets            │   │
│  └─────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Time Series Database                   │   │
│  │  [timestamp, labels, value]                        │   │
│  └─────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                 PromQL Engine                       │   │
│  │  rate(), histogram_quantile(), increase()          │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
         │ scrape              │ query
         ▼                     ▼
┌────────────────┐    ┌────────────────┐
│  /metrics      │    │    Grafana     │
│  Node.js App   │    │    Alertmgr    │
│  PG Exporter   │    └────────────────┘
│  Redis Export  │
└────────────────┘
```

### Data Model

```
# Metric ใน Prometheus มีรูปแบบ:
# metric_name{label1="value1", label2="value2"} value timestamp

# ตัวอย่าง:
http_requests_total{method="GET", status="200", route="/api/orders"} 1543
http_requests_total{method="POST", status="201", route="/api/orders"} 328
http_requests_total{method="GET", status="404", route="/api/orders"} 15

http_request_duration_seconds_bucket{le="0.1", route="/api/orders"} 1200
http_request_duration_seconds_bucket{le="0.5", route="/api/orders"} 1480
http_request_duration_seconds_bucket{le="1.0", route="/api/orders"} 1530
http_request_duration_seconds_bucket{le="+Inf", route="/api/orders"} 1543
http_request_duration_seconds_sum{route="/api/orders"} 412.5
http_request_duration_seconds_count{route="/api/orders"} 1543
```

### Metric Types

```
1. Counter: ค่าที่เพิ่มขึ้นเรื่อยๆ (reset เมื่อ restart)
   - http_requests_total
   - errors_total
   - bytes_sent_total

2. Gauge: ค่าที่ขึ้น-ลงได้
   - active_connections
   - memory_usage_bytes
   - queue_size

3. Histogram: distribution ของค่า (มี buckets)
   - request_duration_seconds
   - query_duration_seconds
   - response_size_bytes

4. Summary: เหมือน Histogram แต่ compute quantiles client-side
   - ใช้น้อยกว่า เพราะ quantiles ไม่ aggregate ได้ข้าม instances
```

---

## Node.js Metrics ด้วย prom-client

### ติดตั้ง

```bash
npm install prom-client
npm install --save-dev @types/node
```

### ตั้งค่า Metrics

```typescript
// src/metrics/index.ts
import { Registry, collectDefaultMetrics, Counter, Gauge, Histogram } from 'prom-client';

// สร้าง Registry แยก (ไม่ใช้ default global registry)
export const register = new Registry();

// เพิ่ม default labels สำหรับทุก metric
register.setDefaultLabels({
  service: process.env.SERVICE_NAME || 'my-app',
  version: process.env.SERVICE_VERSION || '1.0.0',
  environment: process.env.NODE_ENV || 'development',
});

// Collect default process metrics (CPU, Memory, EventLoop, etc.)
collectDefaultMetrics({
  register,
  prefix: 'nodejs_',
  gcDurationBuckets: [0.001, 0.01, 0.1, 1, 2, 5],
});

// ==================== HTTP Metrics ====================

// HTTP Request Counter
export const httpRequestsTotal = new Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status_code'],
  registers: [register],
});

// HTTP Request Duration Histogram
export const httpRequestDuration = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration in seconds',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10],
  registers: [register],
});

// Active HTTP Connections
export const activeConnections = new Gauge({
  name: 'http_active_connections',
  help: 'Number of active HTTP connections',
  registers: [register],
});

// HTTP Request Size
export const httpRequestSize = new Histogram({
  name: 'http_request_size_bytes',
  help: 'HTTP request size in bytes',
  labelNames: ['method', 'route'],
  buckets: [100, 1000, 5000, 10000, 50000, 100000, 500000, 1000000],
  registers: [register],
});

// HTTP Response Size
export const httpResponseSize = new Histogram({
  name: 'http_response_size_bytes',
  help: 'HTTP response size in bytes',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [100, 1000, 5000, 10000, 50000, 100000, 500000, 1000000],
  registers: [register],
});

// ==================== Cache Metrics ====================

// Cache Hit/Miss Counter
export const cacheOperations = new Counter({
  name: 'cache_operations_total',
  help: 'Total cache operations',
  labelNames: ['operation', 'result', 'key_prefix'],
  registers: [register],
});

// Cache Operation Duration
export const cacheDuration = new Histogram({
  name: 'cache_operation_duration_seconds',
  help: 'Cache operation duration in seconds',
  labelNames: ['operation'],
  buckets: [0.0001, 0.0005, 0.001, 0.005, 0.01, 0.05, 0.1, 0.5],
  registers: [register],
});

// Cache Size (number of keys)
export const cacheSize = new Gauge({
  name: 'cache_keys_total',
  help: 'Total number of keys in cache',
  labelNames: ['database'],
  registers: [register],
});

// ==================== Database Metrics ====================

// DB Query Duration Histogram
export const dbQueryDuration = new Histogram({
  name: 'db_query_duration_seconds',
  help: 'Database query duration in seconds',
  labelNames: ['operation', 'table', 'success'],
  buckets: [0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1, 2.5, 5, 10],
  registers: [register],
});

// DB Query Counter
export const dbQueriesTotal = new Counter({
  name: 'db_queries_total',
  help: 'Total number of database queries',
  labelNames: ['operation', 'table', 'success'],
  registers: [register],
});

// DB Connection Pool
export const dbPoolSize = new Gauge({
  name: 'db_pool_size',
  help: 'Database connection pool size',
  labelNames: ['state'],
  registers: [register],
});

// DB Pool Waiting Requests
export const dbPoolWaiting = new Gauge({
  name: 'db_pool_waiting_requests',
  help: 'Number of requests waiting for a DB connection',
  registers: [register],
});

// ==================== Business Metrics ====================

// Order Processing Counter
export const ordersProcessed = new Counter({
  name: 'orders_processed_total',
  help: 'Total orders processed',
  labelNames: ['status', 'payment_method'],
  registers: [register],
});

// Order Value
export const orderValue = new Histogram({
  name: 'order_value_baht',
  help: 'Order value in Thai Baht',
  buckets: [100, 500, 1000, 2500, 5000, 10000, 25000, 50000, 100000],
  registers: [register],
});

// Active Users Gauge
export const activeUsers = new Gauge({
  name: 'active_users',
  help: 'Number of currently active users',
  registers: [register],
});

// Queue Size
export const queueSize = new Gauge({
  name: 'queue_size',
  help: 'Number of items in processing queue',
  labelNames: ['queue_name'],
  registers: [register],
});

// Queue Processing Time
export const queueProcessingTime = new Histogram({
  name: 'queue_processing_duration_seconds',
  help: 'Time to process a queue item',
  labelNames: ['queue_name', 'success'],
  buckets: [0.01, 0.05, 0.1, 0.5, 1, 5, 10, 30, 60],
  registers: [register],
});
```

### Express Middleware สำหรับ HTTP Metrics

```typescript
// src/metrics/middleware.ts
import { Request, Response, NextFunction } from 'express';
import {
  httpRequestsTotal,
  httpRequestDuration,
  activeConnections,
  httpRequestSize,
  httpResponseSize,
} from './index';

// Normalize route paths (แทน /users/123 ด้วย /users/:id)
function normalizeRoute(req: Request): string {
  // Express route pattern มีอยู่ใน req.route
  if (req.route) {
    return req.baseUrl + req.route.path;
  }
  // Fallback: ใช้ pathname แต่ replace dynamic segments
  const url = new URL(req.url, 'http://localhost');
  return url.pathname
    .replace(/\/\d+/g, '/:id')
    .replace(/\/[0-9a-f-]{36}/g, '/:uuid'); // UUID pattern
}

export function metricsMiddleware(req: Request, res: Response, next: NextFunction): void {
  // เพิ่ม active connection
  activeConnections.inc();
  
  // บันทึกเวลาเริ่ม
  const startTime = process.hrtime.bigint();
  
  // วัด request size
  const requestSize = parseInt(req.headers['content-length'] || '0', 10);
  
  // Hook เมื่อ response ส่งแล้ว
  res.on('finish', () => {
    const duration = Number(process.hrtime.bigint() - startTime) / 1e9;
    const route = normalizeRoute(req);
    const method = req.method;
    const statusCode = res.statusCode.toString();
    
    // บันทึก metrics
    httpRequestsTotal.inc({ method, route, status_code: statusCode });
    httpRequestDuration.observe({ method, route, status_code: statusCode }, duration);
    
    if (requestSize > 0) {
      httpRequestSize.observe({ method, route }, requestSize);
    }
    
    const responseSize = parseInt(res.getHeader('content-length') as string || '0', 10);
    if (responseSize > 0) {
      httpResponseSize.observe({ method, route, status_code: statusCode }, responseSize);
    }
    
    // ลด active connection
    activeConnections.dec();
  });
  
  res.on('error', () => {
    activeConnections.dec();
  });
  
  next();
}

// ==================== /metrics Endpoint ====================

export function metricsEndpoint(req: Request, res: Response): void {
  res.set('Content-Type', register.contentType);
  register.metrics()
    .then((metrics) => res.end(metrics))
    .catch((err) => {
      res.status(500).end(err.message);
    });
}
```

### Database Metrics Wrapper

```typescript
// src/metrics/database.ts
import { Pool, PoolClient } from 'pg';
import { dbQueryDuration, dbQueriesTotal, dbPoolSize, dbPoolWaiting } from './index';

export class MetricsPool {
  private pool: Pool;
  
  constructor(pool: Pool) {
    this.pool = pool;
    
    // Monitor pool events
    this.pool.on('connect', () => {
      this.updatePoolMetrics();
    });
    
    this.pool.on('remove', () => {
      this.updatePoolMetrics();
    });
    
    // Update pool metrics ทุก 5 วินาที
    setInterval(() => this.updatePoolMetrics(), 5000);
  }
  
  private updatePoolMetrics(): void {
    const { totalCount, idleCount, waitingCount } = this.pool;
    dbPoolSize.set({ state: 'total' }, totalCount);
    dbPoolSize.set({ state: 'idle' }, idleCount);
    dbPoolSize.set({ state: 'active' }, totalCount - idleCount);
    dbPoolWaiting.set(waitingCount);
  }
  
  async query<T>(
    sql: string,
    params: unknown[] = [],
    table: string = 'unknown'
  ): Promise<T[]> {
    const operation = sql.trim().split(' ')[0].toUpperCase();
    const timer = dbQueryDuration.startTimer({ operation, table });
    
    try {
      const result = await this.pool.query(sql, params);
      timer({ success: 'true' });
      dbQueriesTotal.inc({ operation, table, success: 'true' });
      return result.rows;
    } catch (error) {
      timer({ success: 'false' });
      dbQueriesTotal.inc({ operation, table, success: 'false' });
      throw error;
    }
  }
}

// ==================== Cache Metrics Wrapper ====================

import Redis from 'ioredis';
import { cacheOperations, cacheDuration, cacheSize } from './index';

export class MetricsRedis {
  private redis: Redis;
  
  constructor(redis: Redis) {
    this.redis = redis;
    
    // Update cache size ทุก 30 วินาที
    setInterval(async () => {
      try {
        const info = await this.redis.info('keyspace');
        const match = info.match(/db0:keys=(\d+)/);
        if (match) {
          cacheSize.set({ database: '0' }, parseInt(match[1], 10));
        }
      } catch {}
    }, 30000);
  }
  
  async get(key: string): Promise<string | null> {
    const keyPrefix = key.split(':')[0];
    const timer = cacheDuration.startTimer({ operation: 'GET' });
    
    try {
      const value = await this.redis.get(key);
      const hit = value !== null;
      
      cacheOperations.inc({
        operation: 'GET',
        result: hit ? 'hit' : 'miss',
        key_prefix: keyPrefix,
      });
      
      timer();
      return value;
    } catch (error) {
      cacheOperations.inc({
        operation: 'GET',
        result: 'error',
        key_prefix: keyPrefix,
      });
      timer();
      throw error;
    }
  }
  
  async set(key: string, value: string, ttl?: number): Promise<void> {
    const keyPrefix = key.split(':')[0];
    const timer = cacheDuration.startTimer({ operation: 'SET' });
    
    try {
      if (ttl) {
        await this.redis.setex(key, ttl, value);
      } else {
        await this.redis.set(key, value);
      }
      cacheOperations.inc({ operation: 'SET', result: 'success', key_prefix: keyPrefix });
      timer();
    } catch (error) {
      cacheOperations.inc({ operation: 'SET', result: 'error', key_prefix: keyPrefix });
      timer();
      throw error;
    }
  }
  
  async del(key: string): Promise<void> {
    const keyPrefix = key.split(':')[0];
    const timer = cacheDuration.startTimer({ operation: 'DEL' });
    
    try {
      await this.redis.del(key);
      cacheOperations.inc({ operation: 'DEL', result: 'success', key_prefix: keyPrefix });
      timer();
    } catch (error) {
      cacheOperations.inc({ operation: 'DEL', result: 'error', key_prefix: keyPrefix });
      timer();
      throw error;
    }
  }
}
```

### Express Application พร้อม Metrics

```typescript
// src/app.ts
import express from 'express';
import { metricsMiddleware, metricsEndpoint } from './metrics/middleware';
import { register } from './metrics';

const app = express();

// Metrics middleware (ต้องอยู่ก่อน routes)
app.use(metricsMiddleware);

// JSON body parser
app.use(express.json());

// Metrics endpoint
app.get('/metrics', metricsEndpoint);

// Health check
app.get('/health', (req, res) => {
  res.json({ 
    status: 'ok',
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
  });
});

// Routes
app.use('/api', require('./routes'));

export default app;
```

---

## PostgreSQL Exporter

### Docker Setup

```yaml
# docker-compose.postgres-exporter.yml
version: '3.8'

services:
  postgres-exporter:
    image: prometheuscommunity/postgres-exporter:v0.15.0
    container_name: postgres-exporter
    ports:
      - "9187:9187"
    environment:
      DATA_SOURCE_NAME: "postgresql://postgres_exporter:exportpassword@postgres:5432/postgres?sslmode=disable"
      PG_EXPORTER_AUTO_DISCOVER_DATABASES: "true"
      PG_EXPORTER_EXTEND_QUERY_PATH: /etc/postgres_exporter/queries.yaml
    volumes:
      - ./postgres-exporter/queries.yaml:/etc/postgres_exporter/queries.yaml
    depends_on:
      - postgres
    restart: unless-stopped
```

### สร้าง User สำหรับ Exporter

```sql
-- สร้าง user สำหรับ postgres_exporter
CREATE USER postgres_exporter WITH PASSWORD 'exportpassword';
GRANT pg_monitor TO postgres_exporter;

-- สำหรับ PostgreSQL < 10
GRANT SELECT ON pg_stat_database TO postgres_exporter;
GRANT SELECT ON pg_stat_bgwriter TO postgres_exporter;
GRANT SELECT ON pg_stat_replication TO postgres_exporter;
```

### Custom Queries

```yaml
# postgres-exporter/queries.yaml
pg_replication:
  query: |
    SELECT
      CASE WHEN pg_is_in_recovery() THEN 1 ELSE 0 END AS in_recovery,
      CASE WHEN pg_is_in_recovery()
        THEN EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp()))
        ELSE 0
      END AS replication_delay_seconds,
      (SELECT count(*) FROM pg_stat_replication) AS replica_count
  metrics:
    - in_recovery:
        usage: "GAUGE"
        description: "Whether the server is in recovery mode"
    - replication_delay_seconds:
        usage: "GAUGE"
        description: "Replication delay in seconds"
    - replica_count:
        usage: "GAUGE"
        description: "Number of connected replicas"

pg_slow_queries:
  query: |
    SELECT
      count(*) AS count,
      max(EXTRACT(EPOCH FROM (now() - query_start))) AS max_duration_seconds,
      avg(EXTRACT(EPOCH FROM (now() - query_start))) AS avg_duration_seconds
    FROM pg_stat_activity
    WHERE state = 'active'
      AND query_start < now() - interval '5 seconds'
      AND query NOT LIKE '%pg_stat_activity%'
  metrics:
    - count:
        usage: "GAUGE"
        description: "Number of slow queries (>5s)"
    - max_duration_seconds:
        usage: "GAUGE"
        description: "Maximum slow query duration"
    - avg_duration_seconds:
        usage: "GAUGE"
        description: "Average slow query duration"

pg_table_bloat:
  query: |
    SELECT
      schemaname,
      tablename,
      n_dead_tup,
      n_live_tup,
      CASE WHEN n_live_tup > 0
        THEN round(100.0 * n_dead_tup / (n_live_tup + n_dead_tup), 2)
        ELSE 0
      END AS dead_tup_ratio
    FROM pg_stat_user_tables
    WHERE n_live_tup + n_dead_tup > 1000
    ORDER BY dead_tup_ratio DESC
    LIMIT 20
  metrics:
    - n_dead_tup:
        usage: "GAUGE"
        description: "Number of dead tuples"
        labels: ["schemaname", "tablename"]
    - n_live_tup:
        usage: "GAUGE"
        description: "Number of live tuples"
        labels: ["schemaname", "tablename"]
    - dead_tup_ratio:
        usage: "GAUGE"
        description: "Dead tuple ratio percentage"
        labels: ["schemaname", "tablename"]

pg_index_usage:
  query: |
    SELECT
      schemaname,
      tablename,
      indexname,
      idx_scan,
      idx_tup_read,
      idx_tup_fetch
    FROM pg_stat_user_indexes
    WHERE idx_scan = 0
      AND schemaname NOT IN ('pg_catalog', 'information_schema')
    LIMIT 30
  metrics:
    - idx_scan:
        usage: "COUNTER"
        description: "Number of index scans"
        labels: ["schemaname", "tablename", "indexname"]
    - idx_tup_read:
        usage: "COUNTER"
        description: "Tuples read via index"
        labels: ["schemaname", "tablename", "indexname"]
```

### Metrics ที่ได้จาก postgres_exporter

```
# Connection metrics
pg_stat_database_numbackends{datname="myapp"}  # Active connections
pg_settings_max_connections                      # Max allowed connections

# Replication metrics
pg_replication_in_recovery                       # Is replica?
pg_replication_replication_delay_seconds         # Replication lag

# Query performance
pg_stat_database_tup_fetched{datname="myapp"}    # Rows fetched
pg_stat_database_tup_inserted{datname="myapp"}   # Rows inserted
pg_stat_database_blks_hit{datname="myapp"}       # Buffer cache hits
pg_stat_database_blks_read{datname="myapp"}      # Disk reads

# Table stats
pg_stat_user_tables_n_dead_tup                   # Dead tuples
pg_stat_user_tables_last_autovacuum              # Last autovacuum time

# Bgwriter
pg_stat_bgwriter_checkpoints_timed_total         # Scheduled checkpoints
pg_stat_bgwriter_checkpoints_req_total           # Requested checkpoints
pg_stat_bgwriter_buffers_written_total           # Buffers written
```

---

## Redis Exporter

### Docker Setup

```yaml
# docker-compose.redis-exporter.yml
version: '3.8'

services:
  redis-exporter:
    image: oliver006/redis_exporter:v1.55.0
    container_name: redis-exporter
    ports:
      - "9121:9121"
    environment:
      REDIS_ADDR: "redis://redis:6379"
      REDIS_PASSWORD: "${REDIS_PASSWORD:-}"
      REDIS_EXPORTER_DEBUG: "false"
      REDIS_EXPORTER_CHECK_SINGLE_KEYS: "db0=orders:*,db0=sessions:*"
      REDIS_EXPORTER_SCRIPT: /scripts/redis-custom.lua
    volumes:
      - ./redis-exporter/redis-custom.lua:/scripts/redis-custom.lua
    depends_on:
      - redis
    restart: unless-stopped
```

### Metrics ที่ได้จาก redis_exporter

```
# Memory
redis_memory_used_bytes                  # Used memory
redis_memory_max_bytes                   # Max memory
redis_memory_fragmentation_ratio         # Fragmentation ratio
redis_mem_fragmentation_bytes            # Fragmentation size

# Operations
redis_commands_processed_total           # Total commands processed
redis_keyspace_hits_total                # Cache hits
redis_keyspace_misses_total              # Cache misses
redis_instantaneous_ops_per_sec         # Current ops/sec

# Clients
redis_connected_clients                  # Connected clients
redis_blocked_clients                    # Blocked clients
redis_client_recent_max_input_buffer    # Max input buffer

# Keys
redis_db_keys{db="db0"}                  # Total keys
redis_db_keys_expiring{db="db0"}         # Keys with TTL
redis_expired_keys_total                 # Total expired keys
redis_evicted_keys_total                 # Total evicted keys

# Replication
redis_master_repl_offset                 # Master replication offset
redis_connected_slaves                   # Connected replicas
redis_replication_offset{slave_id="0"}  # Replica offset (lag)

# Persistence
redis_rdb_last_save_timestamp_seconds    # Last RDB save time
redis_aof_enabled                        # AOF enabled?
redis_aof_rewrite_in_progress            # AOF rewrite running?
```

---

## Node Exporter: OS Metrics

```yaml
# docker-compose.node-exporter.yml
services:
  node-exporter:
    image: prom/node-exporter:v1.7.0
    container_name: node-exporter
    ports:
      - "9100:9100"
    command:
      - '--path.rootfs=/host'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
      - '--collector.systemd'
      - '--collector.processes'
    volumes:
      - '/:/host:ro,rslave'
    pid: "host"
    restart: unless-stopped
```

---

## Prometheus Configuration

### prometheus.yml

```yaml
# prometheus/prometheus.yml
global:
  scrape_interval: 15s          # Default scrape interval
  evaluation_interval: 15s      # Rule evaluation interval
  external_labels:
    cluster: 'production'
    datacenter: 'th-central-1'

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - 'alertmanager:9093'

# Rule files
rule_files:
  - '/etc/prometheus/rules/*.yml'

# Scrape configurations
scrape_configs:
  # Prometheus self-monitoring
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # Node.js Application
  - job_name: 'nodejs-app'
    scrape_interval: 15s
    static_configs:
      - targets: ['app:3000']
    metrics_path: '/metrics'
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
        regex: '(.*):.*'
        replacement: '${1}'

  # Multiple app instances
  - job_name: 'nodejs-app-cluster'
    scrape_interval: 15s
    static_configs:
      - targets:
          - 'app-1:3000'
          - 'app-2:3000'
          - 'app-3:3000'
    labels:
      service: 'order-api'

  # PostgreSQL Exporter
  - job_name: 'postgres'
    scrape_interval: 30s
    static_configs:
      - targets: ['postgres-exporter:9187']
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
        regex: '(.*):.*'
        replacement: 'postgres-primary'

  # Redis Exporter
  - job_name: 'redis'
    scrape_interval: 15s
    static_configs:
      - targets: ['redis-exporter:9121']

  # Node Exporter (OS metrics)
  - job_name: 'node'
    scrape_interval: 15s
    static_configs:
      - targets:
          - 'node-exporter-db1:9100'
          - 'node-exporter-db2:9100'
          - 'node-exporter-app1:9100'
    relabel_configs:
      - source_labels: [__address__]
        target_label: hostname
        regex: '(.*):.*'
        replacement: '${1}'
```

---

## Grafana: Visualization

### Docker Setup

```yaml
# docker-compose.grafana.yml
version: '3.8'

services:
  grafana:
    image: grafana/grafana:10.2.0
    container_name: grafana
    ports:
      - "3001:3000"  # Grafana UI
    environment:
      # Admin credentials
      GF_SECURITY_ADMIN_USER: admin
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASSWORD:-admin123}
      # Allow anonymous access (dev only)
      GF_AUTH_ANONYMOUS_ENABLED: "false"
      # SMTP for alerts
      GF_SMTP_ENABLED: "true"
      GF_SMTP_HOST: "smtp.gmail.com:587"
      GF_SMTP_USER: ${SMTP_USER}
      GF_SMTP_PASSWORD: ${SMTP_PASSWORD}
      GF_SMTP_FROM_ADDRESS: ${SMTP_USER}
      # Organization
      GF_ORGANIZATION_NAME: "MyApp"
      # Plugins
      GF_INSTALL_PLUGINS: "grafana-clock-panel,grafana-piechart-panel,grafana-worldmap-panel"
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
      - ./grafana/dashboards:/var/lib/grafana/dashboards
    depends_on:
      - prometheus
    restart: unless-stopped

volumes:
  grafana-data:
```

### Provisioning: Auto-Configure Data Sources

```yaml
# grafana/provisioning/datasources/prometheus.yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    jsonData:
      httpMethod: POST
      manageAlerts: true
      prometheusType: Prometheus
      prometheusVersion: 2.47.0
      cacheLevel: 'High'
      disableRecordingRules: false
      incrementalQueryOverlapWindow: 10m
    editable: true
```

### Provisioning: Auto-Load Dashboards

```yaml
# grafana/provisioning/dashboards/default.yaml
apiVersion: 1

providers:
  - name: 'Default'
    orgId: 1
    folder: 'Imported'
    type: file
    disableDeletion: false
    updateIntervalSeconds: 60
    allowUiUpdates: true
    options:
      path: /var/lib/grafana/dashboards
      foldersFromFilesStructure: true
```

---

## PromQL: Query Language

### Basic Queries

```promql
# ==================== Instant Queries ====================

# HTTP request rate (requests per second)
rate(http_requests_total[5m])

# Error rate
rate(http_requests_total{status_code=~"5.."}[5m])

# Error percentage
100 * rate(http_requests_total{status_code=~"5.."}[5m])
    / rate(http_requests_total[5m])

# P99 latency
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

# P95 latency by route
histogram_quantile(0.95, 
  sum by (route, le) (rate(http_request_duration_seconds_bucket[5m]))
)

# Cache hit ratio
rate(cache_operations_total{result="hit"}[5m])
  / (rate(cache_operations_total{result="hit"}[5m]) + rate(cache_operations_total{result="miss"}[5m]))

# Active DB connections vs max
pg_stat_database_numbackends / pg_settings_max_connections

# PostgreSQL replication lag
pg_replication_replication_delay_seconds

# Redis memory usage percentage
redis_memory_used_bytes / redis_memory_max_bytes * 100

# Redis hit rate
rate(redis_keyspace_hits_total[5m])
  / (rate(redis_keyspace_hits_total[5m]) + rate(redis_keyspace_misses_total[5m]))

# ==================== Aggregations ====================

# Total request rate across all instances
sum(rate(http_requests_total[5m]))

# Request rate by route
sum by (route) (rate(http_requests_total[5m]))

# Top 5 slowest routes
topk(5, histogram_quantile(0.95, 
  sum by (route, le) (rate(http_request_duration_seconds_bucket[5m]))
))

# DB query rate by operation
sum by (operation) (rate(db_queries_total[5m]))

# ==================== Increase / Rate ====================

# Total requests in last hour
increase(http_requests_total[1h])

# Requests per second with irate (instant rate, less smoothing)
irate(http_requests_total[5m])

# ==================== Comparison ====================

# Routes with error rate > 1%
(
  sum by (route) (rate(http_requests_total{status_code=~"5.."}[5m]))
  / sum by (route) (rate(http_requests_total[5m]))
) > 0.01

# Instances with high memory
process_resident_memory_bytes > 500 * 1024 * 1024  # > 500MB

# ==================== Joins / Math ====================

# CPU usage percentage
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Disk usage percentage
100 * (
  node_filesystem_size_bytes{mountpoint="/"}
  - node_filesystem_free_bytes{mountpoint="/"}
) / node_filesystem_size_bytes{mountpoint="/"}
```

### Recording Rules สำหรับ Expensive Queries

```yaml
# prometheus/rules/recording_rules.yml
groups:
  - name: http_metrics
    interval: 30s
    rules:
      # Pre-compute request rate
      - record: job:http_requests:rate5m
        expr: sum by (job, route, status_code) (rate(http_requests_total[5m]))
      
      # Pre-compute P99 latency
      - record: job:http_request_duration_p99:rate5m
        expr: |
          histogram_quantile(0.99,
            sum by (job, route, le) (rate(http_request_duration_seconds_bucket[5m]))
          )
      
      # Pre-compute error rate
      - record: job:http_error_rate:rate5m
        expr: |
          sum by (job, route) (rate(http_requests_total{status_code=~"5.."}[5m]))
            / sum by (job, route) (rate(http_requests_total[5m]))
      
  - name: cache_metrics
    interval: 60s
    rules:
      # Cache hit ratio
      - record: job:cache_hit_ratio:rate5m
        expr: |
          rate(cache_operations_total{result="hit"}[5m])
            / (rate(cache_operations_total{result="hit"}[5m]) 
               + rate(cache_operations_total{result="miss"}[5m]))
```

---

## Grafana Dashboard JSON (ตัวอย่าง)

```json
{
  "title": "Application Overview",
  "uid": "app-overview",
  "panels": [
    {
      "type": "stat",
      "title": "Request Rate",
      "gridPos": {"h": 4, "w": 6, "x": 0, "y": 0},
      "targets": [
        {
          "expr": "sum(rate(http_requests_total[5m]))",
          "legendFormat": "req/s"
        }
      ],
      "options": {
        "reduceOptions": {"calcs": ["lastNotNull"]},
        "orientation": "auto",
        "colorMode": "background",
        "graphMode": "area"
      },
      "fieldConfig": {
        "defaults": {
          "unit": "reqps",
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {"color": "green", "value": null},
              {"color": "yellow", "value": 1000},
              {"color": "red", "value": 5000}
            ]
          }
        }
      }
    },
    {
      "type": "stat",
      "title": "Error Rate",
      "gridPos": {"h": 4, "w": 6, "x": 6, "y": 0},
      "targets": [
        {
          "expr": "100 * sum(rate(http_requests_total{status_code=~'5..'}[5m])) / sum(rate(http_requests_total[5m]))",
          "legendFormat": "Error %"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "percent",
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {"color": "green", "value": null},
              {"color": "yellow", "value": 0.5},
              {"color": "red", "value": 1}
            ]
          }
        }
      }
    },
    {
      "type": "timeseries",
      "title": "P99 Latency by Route",
      "gridPos": {"h": 8, "w": 24, "x": 0, "y": 4},
      "targets": [
        {
          "expr": "histogram_quantile(0.99, sum by (route, le) (rate(http_request_duration_seconds_bucket[5m])))",
          "legendFormat": "{{route}}"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "s",
          "custom": {
            "lineWidth": 2,
            "fillOpacity": 10
          }
        }
      }
    }
  ],
  "time": {"from": "now-1h", "to": "now"},
  "refresh": "30s",
  "templating": {
    "list": [
      {
        "name": "service",
        "type": "query",
        "query": "label_values(http_requests_total, service)",
        "refresh": 1,
        "includeAll": true
      }
    ]
  }
}
```

---

## Grafana Alerts

### Notification Channels

```yaml
# grafana/provisioning/notifiers/slack.yaml
apiVersion: 1

notifiers:
  - name: Slack-Critical
    type: slack
    uid: slack-critical
    settings:
      url: ${SLACK_WEBHOOK_URL}
      recipient: "#incidents"
      mentionChannel: "here"
      token: ${SLACK_TOKEN}
    
  - name: PagerDuty
    type: pagerduty
    uid: pagerduty
    settings:
      integrationKey: ${PAGERDUTY_KEY}
      
  - name: Email-Team
    type: email
    uid: email-team
    settings:
      addresses: "team@example.com;oncall@example.com"
      singleEmail: false
```

### Alert Rules ใน Grafana

```json
{
  "title": "High Error Rate",
  "condition": "C",
  "data": [
    {
      "refId": "A",
      "queryType": "",
      "relativeTimeRange": {"from": 600, "to": 0},
      "datasourceUid": "prometheus",
      "model": {
        "expr": "100 * sum(rate(http_requests_total{status_code=~'5..'}[5m])) / sum(rate(http_requests_total[5m]))",
        "intervalMs": 1000,
        "maxDataPoints": 43200,
        "refId": "A"
      }
    },
    {
      "refId": "C",
      "queryType": "",
      "relativeTimeRange": {"from": 0, "to": 0},
      "datasourceUid": "__expr__",
      "model": {
        "conditions": [
          {
            "evaluator": {"params": [1], "type": "gt"},
            "operator": {"type": "and"},
            "query": {"params": ["A"]},
            "reducer": {"params": [], "type": "last"},
            "type": "query"
          }
        ],
        "refId": "C",
        "type": "classic_conditions"
      }
    }
  ],
  "noDataState": "NoData",
  "execErrState": "Error",
  "for": "5m",
  "annotations": {
    "description": "Error rate is {{ $values.A.Value | humanizePercentage }} which exceeds 1%",
    "runbook_url": "https://wiki.example.com/runbooks/high-error-rate",
    "summary": "High HTTP error rate detected"
  },
  "labels": {
    "severity": "critical",
    "team": "backend"
  }
}
```

---

## SLI/SLO: กำหนดและติดตาม

### กำหนด SLI (Service Level Indicators)

```yaml
# SLI Definitions

# Availability SLI
availability_sli: |
  1 - (
    sum(rate(http_requests_total{status_code=~"5.."}[5m]))
    / sum(rate(http_requests_total[5m]))
  )

# Latency SLI (% of requests < 500ms)
latency_sli: |
  sum(rate(http_request_duration_seconds_bucket{le="0.5"}[5m]))
  / sum(rate(http_request_duration_seconds_count[5m]))

# Throughput SLI  
throughput_sli: |
  sum(rate(http_requests_total[5m]))
```

### SLO Configuration

```yaml
# SLO Definitions
slos:
  - name: "Order API Availability"
    sli: availability_sli
    target: 0.999  # 99.9% availability
    window: 30d    # Rolling 30-day window
    alerting:
      burn_rate_alerts:
        - window: 1h
          burn_rate: 14.4  # Uses 2% error budget in 1 hour
          severity: critical
        - window: 6h
          burn_rate: 6     # Uses 5% error budget in 6 hours
          severity: warning

  - name: "Order API Latency"
    sli: latency_sli
    target: 0.95   # 95% of requests under 500ms
    window: 30d

# Error Budget Calculation:
# Target: 99.9%
# Error Budget: 0.1% = 43.8 minutes/month
# Current burn rate: 14.4x means:
#   - We'll use all budget in: 30d / 14.4 = ~2 days
#   - Alert immediately!
```

### Error Budget Dashboard

```promql
# Error Budget Remaining (%)
(
  1 - (
    sum_over_time(http_errors:rate5m[30d]) / sum_over_time(http_requests:rate5m[30d])
  ) / (1 - 0.999)
) * 100

# Time Until Budget Exhausted
(
  # Current budget remaining
  (0.001 - (sum_over_time(http_errors:rate5m[30d]) / sum_over_time(http_requests:rate5m[30d])))
  # Divided by current burn rate
  / rate(http_errors_total[1h]) * rate(http_requests_total[1h])
)
```

---

## Full Docker Compose: Monitoring Stack

```yaml
# docker-compose.monitoring.yml
version: '3.8'

services:
  # ==================== Prometheus ====================
  prometheus:
    image: prom/prometheus:v2.47.0
    container_name: prometheus
    ports:
      - "9090:9090"
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'
      - '--storage.tsdb.retention.size=50GB'
      - '--web.enable-lifecycle'
      - '--web.enable-admin-api'
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./prometheus/rules:/etc/prometheus/rules
      - prometheus-data:/prometheus
    restart: unless-stopped
    
  # ==================== Grafana ====================
  grafana:
    image: grafana/grafana:10.2.0
    container_name: grafana
    ports:
      - "3001:3000"
    environment:
      GF_SECURITY_ADMIN_USER: admin
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASSWORD:-admin123}
      GF_USERS_ALLOW_SIGN_UP: "false"
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    depends_on:
      - prometheus
    restart: unless-stopped

  # ==================== Exporters ====================
  
  postgres-exporter:
    image: prometheuscommunity/postgres-exporter:v0.15.0
    container_name: postgres-exporter
    ports:
      - "9187:9187"
    environment:
      DATA_SOURCE_NAME: "postgresql://postgres_exporter:password@postgres:5432/postgres?sslmode=disable"
      PG_EXPORTER_AUTO_DISCOVER_DATABASES: "true"
    restart: unless-stopped

  redis-exporter:
    image: oliver006/redis_exporter:v1.55.0
    container_name: redis-exporter
    ports:
      - "9121:9121"
    environment:
      REDIS_ADDR: "redis://redis:6379"
    restart: unless-stopped

  node-exporter:
    image: prom/node-exporter:v1.7.0
    container_name: node-exporter
    ports:
      - "9100:9100"
    command:
      - '--path.rootfs=/host'
    volumes:
      - '/:/host:ro,rslave'
    pid: "host"
    restart: unless-stopped

  # ==================== Alertmanager ====================
  alertmanager:
    image: prom/alertmanager:v0.26.0
    container_name: alertmanager
    ports:
      - "9093:9093"
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
      - '--storage.path=/alertmanager'
      - '--cluster.advertise-address=0.0.0.0:9093'
    volumes:
      - ./alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml
      - alertmanager-data:/alertmanager
    restart: unless-stopped

volumes:
  prometheus-data:
  grafana-data:
  alertmanager-data:
```

---

## สรุป: Metrics Checklist

```markdown
## Application Metrics Checklist

### HTTP Metrics ✓
- [ ] Request rate (req/s)
- [ ] Error rate (errors/s และ %)
- [ ] Latency P50, P95, P99
- [ ] Active connections
- [ ] Request/Response size

### Database Metrics ✓
- [ ] Query rate by operation
- [ ] Query duration P95, P99
- [ ] Connection pool usage
- [ ] Slow queries
- [ ] Replication lag (ถ้ามี)

### Cache Metrics ✓
- [ ] Hit/miss rate
- [ ] Cache operation latency
- [ ] Cache size (keys)
- [ ] Eviction rate

### Business Metrics ✓
- [ ] Orders/transactions per minute
- [ ] Revenue metrics
- [ ] User activity
- [ ] Queue sizes and processing times

### Infrastructure Metrics ✓
- [ ] CPU usage
- [ ] Memory usage
- [ ] Disk usage and I/O
- [ ] Network I/O
```

---

*Part 57 เสร็จสมบูรณ์ - ต่อไป Part 58: Alerting และ Incident Response*

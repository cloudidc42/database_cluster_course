# Part 77: Global Data Distribution

## บทนำ: ความท้าทายของ Global Data Distribution

การกระจายข้อมูลทั่วโลกเป็นหนึ่งในปัญหาที่ยากที่สุดในวิศวกรรมซอฟต์แวร์ เราต้องสร้างสมดุลระหว่าง:

```
CAP Theorem ในบริบท Global Distribution:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

C - Consistency: ทุก node เห็นข้อมูลเหมือนกัน
A - Availability: ทุก request ได้รับ response (ไม่ error)
P - Partition Tolerance: ระบบทำงานได้แม้ network ขาด

ในระบบ distributed: P เป็น given (network ขาดได้เสมอ)
เราต้องเลือกระหว่าง C หรือ A เมื่อเกิด partition

Real world tradeoff:
CP (Consistency + Partition): Strong consistency, อาจ unavailable เมื่อ network ขาด
AP (Availability + Partition): ตอบกลับเสมอ แต่อาจข้อมูลไม่ consistent
```

---

## Data Distribution Strategies

### Strategy 1: Full Replication (ข้อมูลเหมือนกันทุก Region)

```
Full Replication:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Singapore: [Users Table: 1M rows] [Products: 100K] [Orders: 10M]
Frankfurt: [Users Table: 1M rows] [Products: 100K] [Orders: 10M]
US-East:   [Users Table: 1M rows] [Products: 100K] [Orders: 10M]
Tokyo:     [Users Table: 1M rows] [Products: 100K] [Orders: 10M]

✓ Read จาก region ใดก็ได้ (low latency reads)
✓ ง่ายต่อการ query (no cross-region joins)
✗ Write ต้อง replicate ไปทุก region
✗ Storage cost × N regions
✗ Write conflicts ถ้าทำ Active-Active

เหมาะกับ: Read-heavy data (product catalog, content, config)
ไม่เหมาะกับ: High-write data (transactions, events, logs)
```

### Strategy 2: Geographic Sharding (ข้อมูลอยู่ที่ region ที่ใกล้ที่สุด)

```
Geographic Sharding:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Singapore: EU users' data NOT here
Frankfurt: [EU Users: 500K] [EU Orders: 5M]    ← เฉพาะ EU users
US-East:   [US Users: 300K] [US Orders: 3M]    ← เฉพาะ US users
Singapore: [APAC Users: 200K] [APAC Orders: 2M] ← เฉพาะ APAC users

✓ Data stays near users (compliance friendly)
✓ Storage distributed across regions
✓ Write latency ต่ำสำหรับ local users
✗ Cross-region queries ยาก (EU user buys from US seller)
✗ Data hotspots (US region อาจมี traffic มากกว่า)
✗ Rebalancing ยากเมื่อ user population เปลี่ยน

เหมาะกับ: User data, compliance-sensitive data
```

### Strategy 3: Hybrid (Hot/Cold Data Separation)

```
Hybrid Strategy:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Hot Data (replicated everywhere):
- Product catalog (read-heavy, rarely changes)
- User sessions (read-heavy, short-lived)
- Currency exchange rates (read-heavy, updated hourly)
- Configuration data

Cold/Sensitive Data (region-specific):
- User PII (email, phone, address) → stays in user's region
- Financial transactions → stays in originating region
- Audit logs → stays in region for compliance
- Old orders (>1 year) → archive in home region

Implementation:
Singapore (primary): All data
Frankfurt (replica): Hot data + EU users' cold data
US-East (replica): Hot data + US users' cold data
```

---

## PostgreSQL Partitioning + Logical Replication

### Range Partitioning by Region

```sql
-- สร้าง partitioned table สำหรับ users
CREATE TABLE users (
    id UUID NOT NULL DEFAULT gen_random_uuid(),
    email VARCHAR(255) NOT NULL,
    region VARCHAR(10) NOT NULL,
    name VARCHAR(255),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    last_login TIMESTAMPTZ,
    settings JSONB DEFAULT '{}'::jsonb
) PARTITION BY LIST (region);

-- แต่ละ partition สำหรับแต่ละ region
CREATE TABLE users_eu 
    PARTITION OF users 
    FOR VALUES IN ('eu-west-1', 'eu-central-1', 'eu-north-1');

CREATE TABLE users_apac 
    PARTITION OF users 
    FOR VALUES IN ('ap-southeast-1', 'ap-northeast-1', 'ap-south-1', 'ap-southeast-2');

CREATE TABLE users_us 
    PARTITION OF users 
    FOR VALUES IN ('us-east-1', 'us-west-2', 'ca-central-1');

CREATE TABLE users_other 
    PARTITION OF users 
    DEFAULT;

-- สร้าง index บนแต่ละ partition
CREATE INDEX ON users_eu (email);
CREATE INDEX ON users_eu (created_at DESC);
CREATE INDEX ON users_apac (email);
CREATE INDEX ON users_apac (created_at DESC);
CREATE INDEX ON users_us (email);
CREATE INDEX ON users_us (created_at DESC);

-- ตรวจสอบ partition routing
EXPLAIN SELECT * FROM users WHERE region = 'ap-southeast-1' AND email = 'user@example.com';
-- จะเห็นว่า query ไปที่ users_apac เท่านั้น (Partition pruning)
```

### Logical Replication สำหรับ Selective Replication

```sql
-- บน Primary (Singapore): Setup Publication
-- Publication 1: Hot data สำหรับทุก region
CREATE PUBLICATION hot_data_pub 
FOR TABLE products, currency_rates, config_settings, promotions;

-- Publication 2: EU-specific data
CREATE PUBLICATION eu_data_pub 
FOR TABLE users_eu, orders 
WHERE (orders.region IN ('eu-west-1', 'eu-central-1'));

-- Publication 3: APAC-specific data  
CREATE PUBLICATION apac_data_pub 
FOR TABLE users_apac, orders 
WHERE (orders.region IN ('ap-southeast-1', 'ap-northeast-1'));
```

```sql
-- บน EU Replica (Frankfurt): Subscribe to relevant publications
CREATE SUBSCRIPTION hot_data_sub
    CONNECTION 'host=sg-primary.example.com port=5432 dbname=myapp user=replication password=secret'
    PUBLICATION hot_data_pub
    WITH (copy_data = true, create_slot = true);

CREATE SUBSCRIPTION eu_data_sub
    CONNECTION 'host=sg-primary.example.com port=5432 dbname=myapp user=replication password=secret'
    PUBLICATION eu_data_pub
    WITH (copy_data = true, create_slot = true);

-- ตรวจสอบสถานะ subscription
SELECT
    subname,
    pid,
    received_lsn,
    latest_end_lsn,
    latest_end_time
FROM pg_stat_subscription;

-- ดู subscription tables
SELECT * FROM pg_subscription_rel;
```

---

## Foreign Data Wrappers (FDW)

FDW ช่วยให้ query ข้อมูลจาก remote PostgreSQL servers ได้เหมือนเป็น local tables

### Setup postgres_fdw

```sql
-- บน Singapore server: setup FDW เพื่อ query Frankfurt
CREATE EXTENSION IF NOT EXISTS postgres_fdw;

-- สร้าง foreign server สำหรับ Frankfurt
CREATE SERVER frankfurt_server
    FOREIGN DATA WRAPPER postgres_fdw
    OPTIONS (
        host 'eu-db.example.com',
        port '5432',
        dbname 'myapp',
        -- Performance options
        fetch_size '5000',      -- จำนวน rows ต่อ fetch
        use_remote_estimate 'true'  -- ใช้ remote stats สำหรับ query planning
    );

-- สร้าง User Mapping
CREATE USER MAPPING FOR CURRENT_USER
    SERVER frankfurt_server
    OPTIONS (
        user 'fdw_user',
        password 'fdw_secret'
    );

-- Import schema จาก Frankfurt (สร้าง foreign tables อัตโนมัติ)
CREATE SCHEMA frankfurt_schema;

IMPORT FOREIGN SCHEMA public
    LIMIT TO (users_eu, orders_eu, payments_eu)
    FROM SERVER frankfurt_server
    INTO frankfurt_schema;

-- ตรวจสอบ foreign tables ที่สร้าง
\d frankfurt_schema.*
```

### Distributed Queries with FDW

```sql
-- Query ที่รวมข้อมูลจากหลาย regions
-- ดู all users across all regions
SELECT u.email, u.region, COUNT(o.id) AS order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.created_at >= '2024-01-01'
GROUP BY u.email, u.region

UNION ALL

-- ดู EU users จาก Frankfurt
SELECT eu.email, eu.region, COUNT(o.id) AS order_count
FROM frankfurt_schema.users_eu eu
LEFT JOIN frankfurt_schema.orders_eu o ON eu.id = o.user_id
WHERE eu.created_at >= '2024-01-01'
GROUP BY eu.email, eu.region

ORDER BY order_count DESC
LIMIT 100;

-- Pushdown: WHERE clause จะถูกส่งไป execute ที่ remote server
EXPLAIN VERBOSE
SELECT * FROM frankfurt_schema.users_eu
WHERE email = 'specific@example.com';
-- Output จะเห็น "Remote SQL: SELECT ... WHERE email = $1"
```

```sql
-- ตัวอย่าง: Cross-region report
-- หา users ที่ซื้อสินค้าจากทั้ง 2 regions
WITH singapore_buyers AS (
    SELECT DISTINCT user_id
    FROM orders
    WHERE created_at >= NOW() - INTERVAL '30 days'
),
frankfurt_buyers AS (
    SELECT DISTINCT user_id
    FROM frankfurt_schema.orders_eu
    WHERE created_at >= NOW() - INTERVAL '30 days'
)
SELECT 
    u.email,
    u.region,
    'cross-region buyer' AS buyer_type
FROM users u
INNER JOIN singapore_buyers sb ON u.id = sb.user_id
INNER JOIN frankfurt_buyers fb ON u.id = fb.user_id;
```

### FDW Performance Considerations

```sql
-- ตรวจสอบ cost estimate สำหรับ remote query
-- FDW จะส่ง query ไปรันที่ remote และ fetch ผลลัพธ์กลับมา

-- ปัญหา: N+1 query กับ FDW
-- BAD: JOIN ระหว่าง local และ remote ใน loop
-- GOOD: ใช้ subquery หรือ CTE เพื่อ reduce round trips

-- Bad pattern (avoid this):
-- SELECT * FROM local_table l, frankfurt_schema.remote_table r 
-- WHERE l.id = r.local_id  -- PostgreSQL อาจ fetch ทุก row จาก remote!

-- Good pattern: push filter to remote
SELECT * FROM frankfurt_schema.users_eu
WHERE region = 'eu-west-1'  -- pushed down to Frankfurt
  AND created_at > '2024-01-01';  -- pushed down

-- ตรวจสอบ statistics สำหรับ FDW
ANALYZE frankfurt_schema.users_eu;
SELECT relname, reltuples, relpages 
FROM pg_class 
WHERE relname LIKE '%eu%';
```

---

## Latency Optimization

### Connection Proxying: PgBouncer ใน Each Region

```
Architecture กับ PgBouncer per Region:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

EU Users → PgBouncer (Frankfurt) → PostgreSQL Read Replica (Frankfurt)
                                 ↗ (reads, low latency)
EU Users → App Server (Frankfurt)
                                 ↘ (writes, higher latency)
                                   → PgBouncer (Singapore) → PostgreSQL Primary (Singapore)
```

```ini
# pgbouncer.ini สำหรับ Frankfurt region
[databases]
# Reads → local Frankfurt replica
myapp_read = host=localhost port=5432 dbname=myapp pool_size=50

# Writes → Singapore primary (cross-region)
myapp_write = host=sg-primary.example.com port=5432 dbname=myapp pool_size=20

[pgbouncer]
listen_port = 6432
listen_addr = *
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt

# Pool settings
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 50

# Timeouts
server_idle_timeout = 600
client_idle_timeout = 300
query_wait_timeout = 120

# For cross-region connections: higher timeout
server_connect_timeout = 15  # seconds (default 15)
```

```typescript
// connection-router.ts
interface DatabaseConfig {
  readConnectionString: string;  // local replica
  writeConnectionString: string; // primary (possibly cross-region)
}

class RegionalDatabaseRouter {
  private readPool: any;
  private writePool: any;
  
  constructor(config: DatabaseConfig) {
    const { Pool } = require('pg');
    
    // Read pool: ต่อกับ local replica ผ่าน PgBouncer
    this.readPool = new Pool({
      connectionString: config.readConnectionString,
      max: 50,
      idleTimeoutMillis: 30000,
    });
    
    // Write pool: ต่อกับ primary (cross-region)
    this.writePool = new Pool({
      connectionString: config.writeConnectionString,
      max: 20,
      idleTimeoutMillis: 30000,
      connectionTimeoutMillis: 15000,  // longer timeout for cross-region
    });
  }
  
  async query(sql: string, params?: any[], options?: { write?: boolean }): Promise<any> {
    const pool = options?.write ? this.writePool : this.readPool;
    
    const start = Date.now();
    try {
      const result = await pool.query(sql, params);
      const duration = Date.now() - start;
      
      // Monitor cross-region write latency
      if (options?.write) {
        this.recordMetric('db_write_duration_ms', duration);
        if (duration > 500) {
          console.warn(`Slow cross-region write: ${duration}ms`);
        }
      }
      
      return result;
    } catch (error) {
      throw error;
    }
  }
  
  async transaction<T>(
    fn: (query: (sql: string, params?: any[]) => Promise<any>) => Promise<T>
  ): Promise<T> {
    const client = await this.writePool.connect();
    
    try {
      await client.query('BEGIN');
      
      const result = await fn((sql, params) => client.query(sql, params));
      
      await client.query('COMMIT');
      return result;
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }
  
  private recordMetric(name: string, value: number): void {
    // Prometheus, DataDog, etc.
  }
}
```

### Cache Warming สำหรับ Popular Data

```typescript
// cache-warmer.ts
import { createClient } from 'redis';
import { Pool } from 'pg';

interface CacheWarmingStrategy {
  key: string;
  query: string;
  params?: any[];
  ttlSeconds: number;
  priority: 'high' | 'medium' | 'low';
}

class RegionalCacheWarmer {
  private redis: ReturnType<typeof createClient>;
  private pgPool: Pool;
  private region: string;
  
  constructor(redis: ReturnType<typeof createClient>, pgPool: Pool, region: string) {
    this.redis = redis;
    this.pgPool = pgPool;
    this.region = region;
  }
  
  private strategies: CacheWarmingStrategy[] = [
    {
      key: 'cache:products:featured',
      query: `SELECT id, name, price, image_url, stock_count 
              FROM products 
              WHERE featured = true AND active = true
              ORDER BY popularity_score DESC 
              LIMIT 100`,
      ttlSeconds: 300,  // 5 minutes
      priority: 'high',
    },
    {
      key: 'cache:categories:all',
      query: `SELECT id, name, slug, parent_id, image_url 
              FROM categories 
              WHERE active = true
              ORDER BY sort_order`,
      ttlSeconds: 3600,  // 1 hour
      priority: 'high',
    },
    {
      key: 'cache:promotions:active',
      query: `SELECT id, title, discount_percent, start_date, end_date
              FROM promotions 
              WHERE active = true 
                AND start_date <= NOW() 
                AND end_date >= NOW()
              ORDER BY priority DESC`,
      ttlSeconds: 60,  // 1 minute
      priority: 'high',
    },
    {
      key: 'cache:config:app',
      query: `SELECT key, value FROM app_config WHERE active = true`,
      ttlSeconds: 1800,  // 30 minutes
      priority: 'medium',
    },
  ];
  
  async warmCache(): Promise<void> {
    console.log(`[${this.region}] Starting cache warming...`);
    
    // Process high priority first
    const sortedStrategies = [...this.strategies].sort((a, b) => {
      const priority = { high: 0, medium: 1, low: 2 };
      return priority[a.priority] - priority[b.priority];
    });
    
    const results = await Promise.allSettled(
      sortedStrategies.map(s => this.warmKey(s))
    );
    
    const succeeded = results.filter(r => r.status === 'fulfilled').length;
    const failed = results.filter(r => r.status === 'rejected').length;
    
    console.log(`[${this.region}] Cache warming complete: ${succeeded} succeeded, ${failed} failed`);
  }
  
  private async warmKey(strategy: CacheWarmingStrategy): Promise<void> {
    const start = Date.now();
    
    // Check if cache is still fresh (avoid unnecessary DB load)
    const ttl = await this.redis.ttl(strategy.key);
    if (ttl > strategy.ttlSeconds * 0.5) {
      // Cache is more than 50% fresh, skip warming
      return;
    }
    
    const result = await this.pgPool.query(strategy.query, strategy.params);
    const data = JSON.stringify(result.rows);
    
    await this.redis.setEx(strategy.key, strategy.ttlSeconds, data);
    
    const duration = Date.now() - start;
    console.log(`[${this.region}] Warmed ${strategy.key} (${result.rows.length} rows, ${duration}ms)`);
  }
  
  // Schedule automatic re-warming before cache expires
  scheduleAutoWarm(): void {
    setInterval(async () => {
      try {
        await this.warmCache();
      } catch (error) {
        console.error(`[${this.region}] Auto-warm failed:`, error);
      }
    }, 60 * 1000); // Check every minute
  }
}
```

---

## Consistency Models

```
Consistency Models Spectrum:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

STRONG                                           WEAK
Consistency ←──────────────────────────────→ Performance

Linearizability → Sequential → Causal → Read-Your-Writes → Monotonic → Eventual

Strong Consistency (Linearizability):
- ทุก read เห็น latest write
- สมเหตุสมผลที่สุด แต่ latency สูง
- Example: Single-master PostgreSQL, Google Spanner
- เหมาะกับ: Financial data, inventory, tickets

Causal Consistency:
- Operations ที่ causally related เห็นในลำดับที่ถูกต้อง
- Concurrent operations อาจเห็นในลำดับต่างกัน
- Middle ground ระหว่าง Strong และ Eventual
- Example: MongoDB causal sessions

Eventual Consistency:
- ถ้าไม่มี write ใหม่ ทุก node จะ converge ไปยัง value เดียวกัน
- ไม่รับประกันว่า read จะเห็น latest write
- Latency ต่ำมาก (read จาก nearest node)
- Example: DNS, Shopping cart, social media likes
```

### Implementing Read-Your-Writes Consistency

```typescript
// read-your-writes.ts
// ปัญหา: User writes to primary, แต่ read จาก replica ที่ยังไม่ได้ sync
// Solution: Track "user's last write LSN" และ wait for replica to catch up

interface UserSession {
  userId: string;
  lastWriteLsn: string | null;
  lastWriteTime: Date | null;
}

class ReadYourWritesManager {
  private sessions: Map<string, UserSession> = new Map();
  private replicaPool: Pool;
  private primaryPool: Pool;
  
  constructor(primaryPool: Pool, replicaPool: Pool) {
    this.primaryPool = primaryPool;
    this.replicaPool = replicaPool;
  }
  
  // หลัง write: บันทึก LSN
  async afterWrite(userId: string): Promise<void> {
    const result = await this.primaryPool.query(
      'SELECT pg_current_wal_lsn()::text AS lsn'
    );
    
    const session = this.sessions.get(userId) || { userId, lastWriteLsn: null, lastWriteTime: null };
    session.lastWriteLsn = result.rows[0].lsn;
    session.lastWriteTime = new Date();
    this.sessions.set(userId, session);
  }
  
  // ก่อน read: ตรวจสอบว่า replica ได้ sync ถึง user's last write LSN หรือยัง
  async getReadPool(userId: string): Promise<Pool> {
    const session = this.sessions.get(userId);
    
    if (!session?.lastWriteLsn) {
      // User hasn't written anything, use replica safely
      return this.replicaPool;
    }
    
    // Check if replica has caught up
    const result = await this.replicaPool.query<{ replay_lsn: string; caught_up: boolean }>(`
      SELECT 
        pg_last_wal_replay_lsn()::text AS replay_lsn,
        pg_last_wal_replay_lsn() >= $1::pg_lsn AS caught_up
    `, [session.lastWriteLsn]);
    
    if (result.rows[0]?.caught_up) {
      return this.replicaPool;
    }
    
    // Replica not caught up: check if write was recent
    const writeAge = Date.now() - (session.lastWriteTime?.getTime() || 0);
    
    if (writeAge < 5000) {
      // Recent write (<5s): use primary to guarantee read-your-writes
      console.log(`User ${userId}: routing read to primary (replica lag after recent write)`);
      return this.primaryPool;
    }
    
    // Old write and replica still behind: something is wrong, use primary as fallback
    console.warn(`User ${userId}: replica significantly behind, using primary`);
    return this.primaryPool;
  }
  
  // ล้าง session หลัง interval ที่กำหนด
  cleanOldSessions(maxAgeSeconds: number = 300): void {
    const cutoff = Date.now() - (maxAgeSeconds * 1000);
    
    for (const [userId, session] of this.sessions.entries()) {
      if (session.lastWriteTime && session.lastWriteTime.getTime() < cutoff) {
        this.sessions.delete(userId);
      }
    }
  }
}
```

---

## Vector Clocks: Track Causality

```typescript
// vector-clock.ts
// Vector Clocks ใช้ track causality ระหว่าง events ใน distributed system

type VectorClock = Record<string, number>;

class VectorClockManager {
  private clock: VectorClock;
  private nodeId: string;
  
  constructor(nodeId: string, initialClock: VectorClock = {}) {
    this.nodeId = nodeId;
    this.clock = { ...initialClock };
    if (!this.clock[nodeId]) {
      this.clock[nodeId] = 0;
    }
  }
  
  // Increment local clock เมื่อมี event ใหม่
  tick(): VectorClock {
    this.clock[this.nodeId] = (this.clock[this.nodeId] || 0) + 1;
    return { ...this.clock };
  }
  
  // Merge เมื่อรับ message จาก node อื่น
  merge(remoteClock: VectorClock): VectorClock {
    // Update local clock: take max of each component
    const allNodes = new Set([...Object.keys(this.clock), ...Object.keys(remoteClock)]);
    
    for (const node of allNodes) {
      this.clock[node] = Math.max(
        this.clock[node] || 0,
        remoteClock[node] || 0
      );
    }
    
    // Increment own clock (received a message = event)
    this.clock[this.nodeId] = (this.clock[this.nodeId] || 0) + 1;
    
    return { ...this.clock };
  }
  
  // เปรียบเทียบ 2 vector clocks
  compare(vc1: VectorClock, vc2: VectorClock): 'before' | 'after' | 'concurrent' | 'equal' {
    const allNodes = new Set([...Object.keys(vc1), ...Object.keys(vc2)]);
    
    let vc1Greater = false;
    let vc2Greater = false;
    
    for (const node of allNodes) {
      const t1 = vc1[node] || 0;
      const t2 = vc2[node] || 0;
      
      if (t1 > t2) vc1Greater = true;
      if (t2 > t1) vc2Greater = true;
    }
    
    if (vc1Greater && vc2Greater) return 'concurrent'; // CONFLICT
    if (!vc1Greater && !vc2Greater) return 'equal';
    if (vc1Greater) return 'before'; // vc1 happened before vc2? ไม่, vc1 is NEWER
    return 'after';
  }
  
  getClock(): VectorClock {
    return { ...this.clock };
  }
}

// ตัวอย่างการใช้:
// Region Singapore และ Frankfurt ทำงานพร้อมกัน
const sgClock = new VectorClockManager('sg');
const euClock = new VectorClockManager('eu');

// SG ทำ event
const sgVc1 = sgClock.tick();  // { sg: 1 }

// EU ทำ event พร้อมกัน (concurrent)
const euVc1 = euClock.tick();  // { eu: 1 }

// SG ส่ง message ไป EU
const euVc2 = euClock.merge(sgVc1);  // { sg: 1, eu: 2 } (merged + incremented)

// EU ตอบกลับ
const sgVc2 = sgClock.merge(euVc2);  // { sg: 2, eu: 2 }
```

---

## CRDTs - Conflict-free Replicated Data Types

```typescript
// crdts.ts
// CRDTs รับประกันว่า concurrent operations ที่ merge กันจะได้ผลเดียวกันเสมอ

// 1. G-Counter (Grow-only Counter)
class GCounter {
  private counts: Record<string, number>;
  private nodeId: string;
  
  constructor(nodeId: string) {
    this.nodeId = nodeId;
    this.counts = { [nodeId]: 0 };
  }
  
  increment(amount: number = 1): void {
    this.counts[this.nodeId] = (this.counts[this.nodeId] || 0) + amount;
  }
  
  value(): number {
    return Object.values(this.counts).reduce((sum, v) => sum + v, 0);
  }
  
  // Merge: take max of each node's count
  merge(other: GCounter): void {
    const allNodes = new Set([
      ...Object.keys(this.counts), 
      ...Object.keys(other.counts)
    ]);
    
    for (const node of allNodes) {
      this.counts[node] = Math.max(
        this.counts[node] || 0,
        other.counts[node] || 0
      );
    }
  }
  
  toJSON() { return { ...this.counts }; }
  static fromJSON(nodeId: string, data: Record<string, number>) {
    const counter = new GCounter(nodeId);
    counter.counts = data;
    return counter;
  }
}

// 2. PN-Counter (increment AND decrement)
class PNCounter {
  private increments: GCounter;
  private decrements: GCounter;
  
  constructor(nodeId: string) {
    this.increments = new GCounter(nodeId);
    this.decrements = new GCounter(nodeId);
  }
  
  increment(amount: number = 1): void {
    this.increments.increment(amount);
  }
  
  decrement(amount: number = 1): void {
    this.decrements.increment(amount);
  }
  
  value(): number {
    return this.increments.value() - this.decrements.value();
  }
  
  merge(other: PNCounter): void {
    this.increments.merge(other.increments);
    this.decrements.merge(other.decrements);
  }
}

// 3. OR-Set (Observed-Remove Set)
// สามารถ add/remove elements ได้โดยไม่เกิด conflict
class ORSet<T> {
  // Each element has a set of unique tokens
  private elements: Map<string, Set<string>>;  // element_key → Set<token>
  private removed: Map<string, Set<string>>;   // element_key → Set<removed tokens>
  private nodeId: string;
  
  constructor(nodeId: string) {
    this.nodeId = nodeId;
    this.elements = new Map();
    this.removed = new Map();
  }
  
  add(element: T): void {
    const key = JSON.stringify(element);
    const token = `${this.nodeId}-${Date.now()}-${Math.random()}`;
    
    if (!this.elements.has(key)) {
      this.elements.set(key, new Set());
    }
    this.elements.get(key)!.add(token);
  }
  
  remove(element: T): void {
    const key = JSON.stringify(element);
    const tokens = this.elements.get(key);
    
    if (tokens) {
      // Mark all current tokens as removed
      if (!this.removed.has(key)) {
        this.removed.set(key, new Set());
      }
      for (const token of tokens) {
        this.removed.get(key)!.add(token);
      }
    }
  }
  
  has(element: T): boolean {
    const key = JSON.stringify(element);
    const tokens = this.elements.get(key);
    const removedTokens = this.removed.get(key) || new Set();
    
    if (!tokens) return false;
    
    // Element exists if it has tokens that haven't been removed
    for (const token of tokens) {
      if (!removedTokens.has(token)) return true;
    }
    return false;
  }
  
  values(): T[] {
    const result: T[] = [];
    for (const [key] of this.elements) {
      if (this.has(JSON.parse(key))) {
        result.push(JSON.parse(key));
      }
    }
    return result;
  }
  
  // Merge: union of all tokens, union of removed tokens
  merge(other: ORSet<T>): void {
    for (const [key, tokens] of other.elements) {
      if (!this.elements.has(key)) {
        this.elements.set(key, new Set());
      }
      for (const token of tokens) {
        this.elements.get(key)!.add(token);
      }
    }
    
    for (const [key, tokens] of other.removed) {
      if (!this.removed.has(key)) {
        this.removed.set(key, new Set());
      }
      for (const token of tokens) {
        this.removed.get(key)!.add(token);
      }
    }
  }
}

// ตัวอย่างใช้งาน: Shopping Cart ที่ sync ระหว่าง regions
class DistributedShoppingCart {
  private items: ORSet<{ productId: string; quantity: number }>;
  private quantityCounters: Map<string, PNCounter>;
  private nodeId: string;
  
  constructor(nodeId: string) {
    this.nodeId = nodeId;
    this.items = new ORSet<any>(nodeId);
    this.quantityCounters = new Map();
  }
  
  addItem(productId: string, quantity: number = 1): void {
    this.items.add({ productId, quantity });
    
    if (!this.quantityCounters.has(productId)) {
      this.quantityCounters.set(productId, new PNCounter(this.nodeId));
    }
    this.quantityCounters.get(productId)!.increment(quantity);
  }
  
  removeItem(productId: string): void {
    const existing = this.items.values().find(i => i.productId === productId);
    if (existing) {
      this.items.remove(existing);
    }
  }
  
  getItems(): Array<{ productId: string; quantity: number }> {
    return this.items.values().map(item => ({
      productId: item.productId,
      quantity: this.quantityCounters.get(item.productId)?.value() || 1,
    }));
  }
}
```

---

## AWS Aurora Global: Detailed Walkthrough

```typescript
// aurora-global-manager.ts
import {
  RDSClient,
  DescribeGlobalClustersCommand,
  FailoverGlobalClusterCommand,
  ModifyGlobalClusterCommand,
} from '@aws-sdk/client-rds';

interface AuroraGlobalClusterStatus {
  globalClusterId: string;
  primaryRegion: string;
  secondaryRegions: string[];
  replicationLagSeconds: number;
  status: 'available' | 'modifying' | 'failing-over';
}

class AuroraGlobalManager {
  private clients: Map<string, RDSClient>;
  
  constructor(regions: string[]) {
    this.clients = new Map();
    for (const region of regions) {
      this.clients.set(region, new RDSClient({ region }));
    }
  }
  
  async getGlobalClusterStatus(
    globalClusterId: string
  ): Promise<AuroraGlobalClusterStatus> {
    // Use us-east-1 client for global cluster info
    const client = this.clients.get('us-east-1') || 
                   [...this.clients.values()][0];
    
    const response = await client.send(new DescribeGlobalClustersCommand({
      GlobalClusterIdentifier: globalClusterId,
    }));
    
    const cluster = response.GlobalClusters?.[0];
    if (!cluster) {
      throw new Error(`Global cluster ${globalClusterId} not found`);
    }
    
    const primaryMember = cluster.GlobalClusterMembers?.find(m => m.IsWriter);
    const secondaryMembers = cluster.GlobalClusterMembers?.filter(m => !m.IsWriter) || [];
    
    const primaryRegion = primaryMember?.DBClusterArn?.split(':')[3] || '';
    const secondaryRegions = secondaryMembers.map(
      m => m.DBClusterArn?.split(':')[3] || ''
    ).filter(Boolean);
    
    return {
      globalClusterId,
      primaryRegion,
      secondaryRegions,
      replicationLagSeconds: 0, // ดูจาก CloudWatch metrics จริงๆ
      status: cluster.Status as any || 'available',
    };
  }
  
  async initiateFailover(
    globalClusterId: string,
    targetRegion: string,
    targetClusterArn: string
  ): Promise<void> {
    console.log(`Initiating failover of ${globalClusterId} to ${targetRegion}...`);
    
    // Step 1: ตรวจสอบ replication lag ก่อน failover
    const lagSeconds = await this.getReplicationLag(globalClusterId, targetRegion);
    
    if (lagSeconds > 5) {
      console.warn(`Warning: Replication lag is ${lagSeconds}s. This may cause data loss.`);
      // In real scenario: ถาม user ว่าต้องการ proceed ไหม
    }
    
    // Step 2: Execute failover
    const client = this.clients.get('us-east-1') || [...this.clients.values()][0];
    
    await client.send(new FailoverGlobalClusterCommand({
      GlobalClusterIdentifier: globalClusterId,
      TargetDbClusterIdentifier: targetClusterArn,
    }));
    
    console.log(`Failover initiated. Monitoring status...`);
    
    // Step 3: Wait for failover to complete
    await this.waitForFailoverComplete(globalClusterId, targetRegion);
    
    console.log(`Failover complete! New primary: ${targetRegion}`);
  }
  
  private async waitForFailoverComplete(
    globalClusterId: string,
    expectedPrimaryRegion: string
  ): Promise<void> {
    const maxWaitMs = 5 * 60 * 1000; // 5 minutes
    const pollIntervalMs = 10 * 1000; // 10 seconds
    const startTime = Date.now();
    
    while (Date.now() - startTime < maxWaitMs) {
      const status = await this.getGlobalClusterStatus(globalClusterId);
      
      if (status.primaryRegion === expectedPrimaryRegion && 
          status.status === 'available') {
        return;
      }
      
      console.log(`Waiting... Current primary: ${status.primaryRegion}, Status: ${status.status}`);
      await new Promise(resolve => setTimeout(resolve, pollIntervalMs));
    }
    
    throw new Error(`Failover timeout after ${maxWaitMs / 1000}s`);
  }
  
  private async getReplicationLag(
    globalClusterId: string,
    region: string
  ): Promise<number> {
    // ดึงจาก CloudWatch metric: AuroraGlobalDBReplicationLag
    // Return mock value for example
    return 1.5;
  }
  
  async modifyReplicationSettings(
    globalClusterId: string,
    options: {
      allowMajorVersionUpgrade?: boolean;
      newGlobalClusterIdentifier?: string;
    }
  ): Promise<void> {
    const client = this.clients.get('us-east-1') || [...this.clients.values()][0];
    
    await client.send(new ModifyGlobalClusterCommand({
      GlobalClusterIdentifier: globalClusterId,
      ...options,
    }));
  }
}
```

---

## Data Locality: Keep Related Data Together

```sql
-- Co-location Strategy สำหรับ CockroachDB
-- Related data ควรอยู่ใน region เดียวกันเพื่อลด cross-region queries

-- สมมติ: Thailand users → APAC region
-- ทุก data ที่เกี่ยวกับ user นั้นควรอยู่ที่ APAC

-- Table: Users (regionalized by user's home region)
CREATE TABLE users (
    id UUID PRIMARY KEY,
    region_home VARCHAR(10) NOT NULL,  -- 'APAC', 'EU', 'US'
    email VARCHAR(255) NOT NULL,
    crdb_region crdb_internal_region AS (
        CASE region_home
            WHEN 'EU' THEN 'eu-west1'
            WHEN 'US' THEN 'us-east1'
            ELSE 'ap-southeast1'  -- Default: APAC
        END
    ) STORED
) LOCALITY REGIONAL BY ROW;

-- Table: Orders (co-located with users)
CREATE TABLE orders (
    id UUID NOT NULL,
    user_id UUID NOT NULL,
    region_home VARCHAR(10) NOT NULL,
    total DECIMAL(10,2),
    status VARCHAR(20),
    created_at TIMESTAMPTZ DEFAULT now(),
    crdb_region crdb_internal_region AS (
        CASE region_home
            WHEN 'EU' THEN 'eu-west1'
            WHEN 'US' THEN 'us-east1'
            ELSE 'ap-southeast1'
        END
    ) STORED,
    PRIMARY KEY (id, crdb_region)
) LOCALITY REGIONAL BY ROW;

-- Query: User + Orders จาก APAC region (ไม่มี cross-region traffic)
SELECT u.email, o.id, o.total, o.status
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.region_home = 'APAC'
  AND o.region_home = 'APAC'
  AND u.id = $1;
-- CockroachDB routing นี้ไปที่ ap-southeast1 region โดยตรง
```

---

## Compliance: Data Must Stay in Region

### Row-Level Tenancy

```sql
-- Schema สำหรับ multi-tenant กับ data residency
CREATE TABLE tenant_data (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    data_region VARCHAR(10) NOT NULL,  -- 'EU', 'APAC', 'US'
    payload JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Row Level Security
ALTER TABLE tenant_data ENABLE ROW LEVEL SECURITY;

-- EU users เห็นเฉพาะ EU data
CREATE POLICY eu_residency ON tenant_data
    FOR ALL
    TO eu_app_user
    USING (data_region = 'EU');

-- APAC users เห็นเฉพาะ APAC data
CREATE POLICY apac_residency ON tenant_data
    FOR ALL
    TO apac_app_user
    USING (data_region = 'APAC');

-- Admin เห็นทั้งหมด
CREATE POLICY admin_full_access ON tenant_data
    FOR ALL
    TO admin_user
    USING (true);
```

### Schema-Level Separation

```sql
-- แยก schema ต่อ region
CREATE SCHEMA eu_data;
CREATE SCHEMA apac_data;
CREATE SCHEMA us_data;

-- EU-specific tables
CREATE TABLE eu_data.users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) NOT NULL,
    gdpr_consent BOOLEAN DEFAULT FALSE,
    data_processed_for TEXT[]
);

CREATE TABLE eu_data.pii_data (
    user_id UUID NOT NULL REFERENCES eu_data.users(id),
    full_name VARCHAR(255) NOT NULL,
    date_of_birth DATE,
    phone_number VARCHAR(50),
    address JSONB,
    -- Encrypted fields
    encrypted_id_number BYTEA,
    encryption_key_id UUID
);

-- ให้ permission เฉพาะ EU application เข้าถึง eu_data schema
GRANT USAGE ON SCHEMA eu_data TO eu_app_role;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA eu_data TO eu_app_role;
REVOKE ALL ON SCHEMA eu_data FROM apac_app_role;
REVOKE ALL ON SCHEMA eu_data FROM us_app_role;
```

---

## Performance Benchmarking Across Regions

```typescript
// benchmark-cross-region.ts
interface BenchmarkResult {
  operation: string;
  region: string;
  p50Latency: number;
  p95Latency: number;
  p99Latency: number;
  throughput: number;  // ops/second
  errors: number;
}

class CrossRegionBenchmark {
  private connections: Map<string, any>;  // region → pool
  
  constructor() {
    this.connections = new Map();
  }
  
  async addRegion(regionName: string, connectionString: string): Promise<void> {
    const { Pool } = await import('pg');
    this.connections.set(regionName, new Pool({ connectionString }));
  }
  
  async benchmarkRead(
    region: string,
    durationSeconds: number = 30
  ): Promise<BenchmarkResult> {
    const pool = this.connections.get(region);
    if (!pool) throw new Error(`No connection for region: ${region}`);
    
    const latencies: number[] = [];
    let errors = 0;
    const startTime = Date.now();
    const endTime = startTime + (durationSeconds * 1000);
    
    while (Date.now() < endTime) {
      const operationStart = Date.now();
      
      try {
        await pool.query('SELECT id, email FROM users ORDER BY RANDOM() LIMIT 1');
        latencies.push(Date.now() - operationStart);
      } catch (error) {
        errors++;
      }
    }
    
    latencies.sort((a, b) => a - b);
    
    return {
      operation: 'random_read',
      region,
      p50Latency: this.percentile(latencies, 50),
      p95Latency: this.percentile(latencies, 95),
      p99Latency: this.percentile(latencies, 99),
      throughput: latencies.length / durationSeconds,
      errors,
    };
  }
  
  async benchmarkWrite(
    region: string,
    durationSeconds: number = 30
  ): Promise<BenchmarkResult> {
    const pool = this.connections.get(region);
    if (!pool) throw new Error(`No connection for region: ${region}`);
    
    const latencies: number[] = [];
    let errors = 0;
    const startTime = Date.now();
    const endTime = startTime + (durationSeconds * 1000);
    
    while (Date.now() < endTime) {
      const operationStart = Date.now();
      
      try {
        await pool.query(
          'INSERT INTO benchmark_events (data, created_at) VALUES ($1, now())',
          [{ test: 'write_benchmark', ts: Date.now() }]
        );
        latencies.push(Date.now() - operationStart);
      } catch (error) {
        errors++;
      }
    }
    
    latencies.sort((a, b) => a - b);
    
    return {
      operation: 'write_insert',
      region,
      p50Latency: this.percentile(latencies, 50),
      p95Latency: this.percentile(latencies, 95),
      p99Latency: this.percentile(latencies, 99),
      throughput: latencies.length / durationSeconds,
      errors,
    };
  }
  
  async runAllBenchmarks(): Promise<BenchmarkResult[]> {
    const results: BenchmarkResult[] = [];
    
    for (const [region] of this.connections) {
      console.log(`Benchmarking ${region}...`);
      
      const readResult = await this.benchmarkRead(region, 30);
      const writeResult = await this.benchmarkWrite(region, 30);
      
      results.push(readResult, writeResult);
      
      console.log(`${region} Read: p50=${readResult.p50Latency}ms, p99=${readResult.p99Latency}ms, ${readResult.throughput.toFixed(0)} ops/s`);
      console.log(`${region} Write: p50=${writeResult.p50Latency}ms, p99=${writeResult.p99Latency}ms, ${writeResult.throughput.toFixed(0)} ops/s`);
    }
    
    return results;
  }
  
  private percentile(sorted: number[], p: number): number {
    if (sorted.length === 0) return 0;
    const index = Math.ceil((p / 100) * sorted.length) - 1;
    return sorted[Math.max(0, index)];
  }
}

// ตัวอย่าง Output ที่คาดหวัง:
// singapore Read: p50=2ms, p99=8ms, 487 ops/s
// singapore Write: p50=5ms, p99=20ms, 180 ops/s
// frankfurt Read: p50=3ms, p99=12ms, 321 ops/s (local replica)
// frankfurt Write: p50=185ms, p99=280ms, 5 ops/s (cross-region to Singapore!)
```

---

## สรุป Global Data Distribution

```
Key Takeaways:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. Distribution Strategy:
   - Read-heavy global data → Full replication
   - Compliance-sensitive data → Geographic sharding
   - Mixed workload → Hybrid (hot/cold separation)

2. Tools:
   - FDW: Query remote databases (cross-region joins)
   - Logical replication: Selective data distribution
   - PgBouncer: Connection pooling per region
   - CRDTs: Conflict-free concurrent updates

3. Consistency Model Selection:
   - Financial/inventory → Strong consistency (single write path)
   - Social/recommendation → Eventual consistency (CRDT)
   - User sessions → Causal consistency (read-your-writes)

4. Compliance:
   - Row Level Security: easy to implement
   - Schema separation: strong isolation
   - Separate databases: strongest isolation (but harder to query)

5. Performance Target:
   - Local reads: <10ms
   - Cross-region writes: 100-300ms (acceptable)
   - Cross-region reads: avoid (use local replica)
```

จบ Part 77 - Global Data Distribution ครอบคลุม:
- Distribution strategies (Full, Geographic, Hybrid)
- PostgreSQL FDW สำหรับ cross-region queries
- Logical replication สำหรับ selective distribution
- Consistency models และ implementation
- CRDTs สำหรับ conflict-free operations
- Aurora Global walkthrough
- Compliance และ data residency
- Performance benchmarking

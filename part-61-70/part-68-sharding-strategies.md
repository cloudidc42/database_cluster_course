# Part 68: Sharding Strategies

## บทนำ

Sharding คือเทคนิคการแบ่ง (partition) ข้อมูลออกไปเก็บในหลาย database servers โดยแต่ละ server เรียกว่า "shard" Sharding เป็นวิธีหลักในการ scale database แบบ horizontal เมื่อ single server ไม่สามารถรองรับข้อมูลและ traffic ได้แล้ว

---

## 1. Sharding คืออะไร และทำไมต้องใช้?

### 1.1 Horizontal vs Vertical Scaling

```
Vertical Scaling (Scale Up):          Horizontal Scaling / Sharding (Scale Out):

   ┌──────────────┐                   ┌────────┐ ┌────────┐ ┌────────┐
   │  Big Server  │                   │ Shard 1│ │ Shard 2│ │ Shard 3│
   │  64 CPU      │                   │  Small │ │  Small │ │  Small │
   │  512 GB RAM  │                   │ Server │ │ Server │ │ Server │
   │  10 TB SSD   │                   └────────┘ └────────┘ └────────┘
   └──────────────┘
   
   - มี ceiling (ใหญ่สุดแค่ไหน)     - ไม่มี ceiling (เพิ่ม shard ได้เรื่อยๆ)
   - ราคาสูงมาก                      - ราคาต่อ shard ถูกกว่า
   - Single point of failure          - Failure isolated ต่อ shard
   - Downtime เมื่อ upgrade           - Rolling upgrade ได้
```

### 1.2 เมื่อไหร่ต้องใช้ Sharding?

```
Indicators ที่บอกว่าต้องการ Sharding:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. ข้อมูลเกิน single server capacity:
   - Table ขนาด > 500 GB มีผลต่อ query performance
   - Index ไม่ fit ใน RAM → ช้ามาก

2. Write throughput เกิน limit:
   - > 50,000 inserts/second บน single PostgreSQL
   - WAL writing เป็น bottleneck

3. Query ช้าแม้ optimize แล้ว:
   - Full table scan บน table 1 billion rows
   - Even ด้วย index, disk I/O เป็น bottleneck

4. Storage cost:
   - Single server ต้องการ expensive NVMe ทั้งหมด
   - Sharding: เก็บ hot data บน fast storage, cold บน cheap storage

เมื่อยังไม่ต้อง shard (ลอง optimize ก่อน):
- Connection pooling: PgBouncer
- Read replicas: สำหรับ read-heavy
- Caching: Redis/Memcached
- Partitioning: PostgreSQL partitioned tables (บน single server)
- Index optimization
- Query optimization
```

---

## 2. Sharding Keys: การเลือกที่สำคัญที่สุด

### 2.1 Monotonic Keys (Auto-increment): Hotspot Problem

```
Auto-increment ID ปัญหา:
━━━━━━━━━━━━━━━━━━━━━━━━

Hash-based sharding ด้วย auto-increment ID:
Shard = id % num_shards

T=1s:  id=1001 → Shard 1
T=2s:  id=1002 → Shard 2
T=3s:  id=1003 → Shard 3
T=4s:  id=1004 → Shard 1  ← วนซ้ำ

ดูเหมือนดี แต่ปัญหาคือ:
- Range queries: WHERE id > 900 → ต้อง query ทุก shards
- Latest records: ORDER BY id DESC → ต้องรวมจากทุก shards
- ยังดีกว่า range-based

Range-based sharding ด้วย auto-increment ID (แย่มาก!):
Shard 1: id 1-1,000,000
Shard 2: id 1,000,001 - 2,000,000
Shard 3: id 2,000,001 - 3,000,000

ปัญหา Hotspot: ข้อมูลใหม่ทั้งหมดไปที่ Shard 3 เสมอ!
Shard 1: ไม่มีการใช้งาน (cold)
Shard 2: ไม่มีการใช้งาน (cold)
Shard 3: ร้อนมาก! รับ write ทั้งหมด

นี่คือ "hotspot problem" ที่ต้องหลีกเลี่ยง
```

### 2.2 Hash-Based Sharding: Even Distribution

```
Hash-based Sharding:
━━━━━━━━━━━━━━━━━━━━

shard_id = HASH(shard_key) % number_of_shards

ตัวอย่าง:
customer_id=1001 → hash=2547893 → 2547893 % 4 = 1 → Shard 1
customer_id=1002 → hash=8934521 → 8934521 % 4 = 2 → Shard 2
customer_id=1003 → hash=1234567 → 1234567 % 4 = 3 → Shard 3
customer_id=1004 → hash=5678901 → 5678901 % 4 = 0 → Shard 0

ข้อดี:
✅ กระจายข้อมูลสม่ำเสมอ (even distribution)
✅ ไม่มี hotspot
✅ ง่ายในการ route

ข้อเสีย:
❌ Range queries ต้อง scan ทุก shards
❌ ยากในการ rebalance (เพิ่ม shard → ต้อย reshuffle data ทั้งหมด)
   - Consistent hashing แก้ปัญหานี้ได้
```

### 2.3 Range-Based Sharding: Range Queries Efficient

```
Range-based Sharding:
━━━━━━━━━━━━━━━━━━━━━

แบ่งตาม range ของ key:

Time-based:
  Shard 1: Jan 2024 - Mar 2024
  Shard 2: Apr 2024 - Jun 2024
  Shard 3: Jul 2024 - Sep 2024
  Shard 4: Oct 2024 - Dec 2024 ← current (hotspot!)

Customer ID range (ไม่ดี ถ้า sequential):
  Shard 1: 1 - 250,000
  Shard 2: 250,001 - 500,000
  Shard 3: 500,001 - 750,000
  Shard 4: 750,001+ ← hotspot

ข้อดี:
✅ Range queries ไปยัง shard เดียวหรือสองสาม shard
✅ ง่ายต่อการเข้าใจ
✅ Time-series data ดีมาก (old data → cold storage)

ข้อเสีย:
❌ Hotspot ถ้าใช้ sequential key
❌ Uneven distribution ถ้า key distribution ไม่สม่ำเสมอ
❌ ต้องวางแผน range ล่วงหน้า
```

### 2.4 Geographic Sharding

```
Geographic Sharding:
━━━━━━━━━━━━━━━━━━━━

แบ่งตาม geography:

  Shard Asia:    customers จาก TH, SG, JP, KR, CN, IN
  Shard Europe:  customers จาก DE, FR, GB, IT, ES
  Shard Americas: customers จาก US, CA, BR, MX

ข้อดี:
✅ Data sovereignty: ข้อมูลอยู่ใน region ตามกฎหมาย (GDPR)
✅ Low latency: user อ่านข้อมูลจาก region ใกล้เคียง
✅ Regulatory compliance: บางประเทศต้องการ data residency

ข้อเสีย:
❌ Cross-region queries ช้า
❌ User ย้าย region ทำให้ต้อง migrate data
❌ เนื้อหาข้อมูลต่าง region มักต่างกัน (ยาก cross-shard)
```

### 2.5 Entity-Based Sharding

```
Entity-Based Sharding:
━━━━━━━━━━━━━━━━━━━━━━

Keep related data together บน shard เดียว

ตัวอย่าง: E-commerce
Shard key: customer_id

Customer 1001's Shard:
  - customers WHERE customer_id = 1001
  - orders WHERE customer_id = 1001
  - order_items WHERE customer_id = 1001 (denormalized)
  - addresses WHERE customer_id = 1001
  - payment_methods WHERE customer_id = 1001

ข้อดี:
✅ Queries ที่เกี่ยวกับ customer คนเดียว → single shard (ไม่ต้อง join cross-shard)
✅ Transactions ที่เกี่ยวกับ entity เดียว → single shard (ACID!)
✅ ง่ายต่อ isolation

ข้อเสีย:
❌ Cross-entity queries ยังต้อง scatter-gather
❌ ถ้าบาง customer มี data เยอะมาก → hot shard
```

---

## 3. Sharding Approaches

### 3.1 Application-Level Sharding

```typescript
// ════════════════════════════════════════════════
// Application-Level Sharding ใน Node.js
// ════════════════════════════════════════════════

import { Pool } from 'pg';
import * as crypto from 'crypto';

interface ShardConfig {
    id: number;
    host: string;
    port: number;
    database: string;
    username: string;
    password: string;
}

class ApplicationShardRouter {
    private shards: Pool[];
    private shardCount: number;
    
    constructor(shardConfigs: ShardConfig[]) {
        this.shardCount = shardConfigs.length;
        this.shards = shardConfigs.map(config => new Pool({
            host: config.host,
            port: config.port,
            database: config.database,
            user: config.username,
            password: config.password,
            max: 20,
            idleTimeoutMillis: 30000
        }));
    }
    
    // Hash-based shard routing
    getShardId(shardKey: string | number): number {
        const key = String(shardKey);
        const hash = crypto.createHash('md5').update(key).digest('hex');
        // ใช้แค่ 8 hex characters แรก เพื่อลด collision
        const hashInt = parseInt(hash.substring(0, 8), 16);
        return hashInt % this.shardCount;
    }
    
    getShard(shardKey: string | number): Pool {
        const shardId = this.getShardId(shardKey);
        return this.shards[shardId];
    }
    
    getAllShards(): Pool[] {
        return this.shards;
    }
    
    // ════════════════════════════════════════════════
    // Scatter-Gather: query ทุก shards แล้วรวมผล
    // ════════════════════════════════════════════════
    async scatterGather<T>(
        query: string, 
        params: any[],
        aggregator: (results: T[][]) => T[]
    ): Promise<T[]> {
        const promises = this.shards.map(shard => 
            shard.query(query, params).then(r => r.rows as T[])
        );
        
        const results = await Promise.all(promises);
        return aggregator(results);
    }
    
    // ดำเนิน query บน shard เดียว
    async queryOneShard<T>(shardKey: string | number, query: string, params: any[]): Promise<T[]> {
        const shard = this.getShard(shardKey);
        const result = await shard.query(query, params);
        return result.rows as T[];
    }
}

// ════════════════════════════════════════════════
// Order Service ที่ใช้ Application Sharding
// ════════════════════════════════════════════════

interface Order {
    order_id: string;
    customer_id: number;
    total_amount: number;
    status: string;
    created_at: Date;
}

class ShardedOrderService {
    constructor(private router: ApplicationShardRouter) {}
    
    async createOrder(customerId: number, items: Array<{productId: number; quantity: number; price: number}>): Promise<string> {
        const shard = this.router.getShard(customerId);
        const orderId = crypto.randomUUID();
        const totalAmount = items.reduce((sum, item) => sum + item.price * item.quantity, 0);
        
        const client = await shard.connect();
        try {
            await client.query('BEGIN');
            
            await client.query(
                `INSERT INTO orders (order_id, customer_id, total_amount, status, created_at)
                 VALUES ($1, $2, $3, 'pending', NOW())`,
                [orderId, customerId, totalAmount]
            );
            
            for (const item of items) {
                await client.query(
                    `INSERT INTO order_items (order_id, product_id, quantity, unit_price)
                     VALUES ($1, $2, $3, $4)`,
                    [orderId, item.productId, item.quantity, item.price]
                );
            }
            
            await client.query('COMMIT');
            return orderId;
            
        } catch (err) {
            await client.query('ROLLBACK');
            throw err;
        } finally {
            client.release();
        }
    }
    
    async getOrdersByCustomer(customerId: number): Promise<Order[]> {
        // Query บน single shard (รู้ว่า customer อยู่ shard ไหน)
        return this.router.queryOneShard<Order>(
            customerId,
            'SELECT * FROM orders WHERE customer_id = $1 ORDER BY created_at DESC',
            [customerId]
        );
    }
    
    async getOrderById(orderId: string, customerId: number): Promise<Order | null> {
        // ต้องรู้ customerId เพื่อ route ไป shard ที่ถูกต้อง
        const rows = await this.router.queryOneShard<Order>(
            customerId,
            'SELECT * FROM orders WHERE order_id = $1 AND customer_id = $2',
            [orderId, customerId]
        );
        return rows[0] || null;
    }
    
    async getRecentOrders(limit: number = 100): Promise<Order[]> {
        // Scatter-gather: query ทุก shards
        return this.router.scatterGather<Order>(
            'SELECT * FROM orders ORDER BY created_at DESC LIMIT $1',
            [limit],
            (results) => {
                // Merge และ sort ผลจากทุก shards
                const allOrders = results.flat();
                allOrders.sort((a, b) => 
                    new Date(b.created_at).getTime() - new Date(a.created_at).getTime()
                );
                return allOrders.slice(0, limit);
            }
        );
    }
    
    async getOrderStats(): Promise<{ totalOrders: number; totalRevenue: number }> {
        const results = await this.router.scatterGather<{count: string; revenue: string}>(
            `SELECT COUNT(*) as count, COALESCE(SUM(total_amount), 0) as revenue 
             FROM orders WHERE status = 'completed'`,
            [],
            (results) => results.flat()
        );
        
        return {
            totalOrders: results.reduce((sum, r) => sum + parseInt(r.count), 0),
            totalRevenue: results.reduce((sum, r) => sum + parseFloat(r.revenue), 0)
        };
    }
}
```

### 3.2 Middleware Sharding: Citus (PostgreSQL)

```sql
-- ════════════════════════════════════════════════
-- Citus: Middleware Sharding (Transparent to Application)
-- ════════════════════════════════════════════════

-- ติดตั้งบน Coordinator node
CREATE EXTENSION citus;

-- ลงทะเบียน Worker nodes
SELECT citus_add_node('worker-1.db.internal', 5432);
SELECT citus_add_node('worker-2.db.internal', 5432);
SELECT citus_add_node('worker-3.db.internal', 5432);
SELECT citus_add_node('worker-4.db.internal', 5432);

-- สร้าง schema
CREATE TABLE customers (
    customer_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    region VARCHAR(50),
    tier VARCHAR(20) DEFAULT 'standard',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE orders (
    order_id UUID DEFAULT gen_random_uuid(),
    customer_id BIGINT NOT NULL,
    status VARCHAR(50) DEFAULT 'pending',
    total_amount DECIMAL(12,2),
    shipping_address JSONB,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE order_items (
    item_id BIGINT GENERATED ALWAYS AS IDENTITY,
    order_id UUID NOT NULL,
    customer_id BIGINT NOT NULL,  -- denormalized สำหรับ co-location!
    product_id BIGINT NOT NULL,
    quantity INT NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL
);

-- Distribute tables
SELECT create_distributed_table('customers', 'customer_id');
SELECT create_distributed_table('orders', 'customer_id');        -- co-located กับ customers
SELECT create_distributed_table('order_items', 'customer_id');   -- co-located กับ customers

-- ตรวจสอบ shard count (default 32 shards)
SELECT COUNT(*) FROM pg_dist_shard WHERE logicalrelid = 'orders'::regclass;

-- กำหนด shard count เอง
SET citus.shard_count = 64;

-- ════════════════════════════════════════════════
-- Query Examples: Application ไม่รู้ว่ามี sharding!
-- ════════════════════════════════════════════════

-- 1. INSERT: Citus route ไป shard ที่ถูกต้องอัตโนมัติ
INSERT INTO orders (customer_id, total_amount, shipping_address) 
VALUES (12345, 1500.00, '{"street": "123 Main St", "city": "Bangkok"}'::jsonb);

-- 2. SELECT บน single customer: Citus ส่งไป single shard
SELECT o.*, oi.product_id, oi.quantity
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id AND o.customer_id = oi.customer_id
WHERE o.customer_id = 12345
ORDER BY o.created_at DESC;

-- 3. Aggregation: Citus scatter-gather แล้วรวมที่ coordinator
SELECT 
    DATE_TRUNC('month', created_at) as month,
    COUNT(*) as order_count,
    SUM(total_amount) as revenue
FROM orders
WHERE created_at >= '2024-01-01'
GROUP BY DATE_TRUNC('month', created_at)
ORDER BY month;

-- 4. Cross-table JOIN: ทำงานได้เพราะ co-located บน customer_id
SELECT 
    c.name,
    c.email,
    COUNT(DISTINCT o.order_id) as total_orders,
    SUM(o.total_amount) as total_spent
FROM customers c
JOIN orders o USING (customer_id)
WHERE o.created_at >= NOW() - INTERVAL '90 days'
GROUP BY c.customer_id, c.name, c.email
HAVING SUM(o.total_amount) > 10000
ORDER BY total_spent DESC;
```

### 3.3 Database-Native Sharding

```
Database-Native Sharding Solutions:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

CockroachDB:
- Compatible PostgreSQL protocol
- Built-in distributed SQL
- Automatic sharding และ rebalancing
- Strong consistency (SERIALIZABLE by default)
- ราคาสูง

YugabyteDB:
- PostgreSQL compatible (YSQL) + Cassandra compatible (YCQL)
- Built-in distributed transactions
- Automatic sharding
- Open source

สำหรับ MySQL:
- Vitess: sharding middleware (ใช้โดย YouTube, Slack)
```

---

## 4. Cross-Shard Queries

### 4.1 Scatter-Gather Pattern

```typescript
// ════════════════════════════════════════════════
// Scatter-Gather: Query ทุก Shards พร้อมกัน
// ════════════════════════════════════════════════

class ScatterGatherExecutor {
    constructor(private shards: Pool[]) {}
    
    async execute<T>(
        query: string,
        params: any[],
        options: {
            timeout?: number;
            failFast?: boolean;
        } = {}
    ): Promise<{ shardId: number; rows: T[]; error?: Error }[]> {
        const timeout = options.timeout || 30000;
        
        const promises = this.shards.map(async (shard, shardId) => {
            try {
                const timeoutPromise = new Promise<never>((_, reject) => {
                    setTimeout(() => reject(new Error(`Shard ${shardId} timeout`)), timeout);
                });
                
                const queryPromise = shard.query(query, params).then(r => r.rows as T[]);
                const rows = await Promise.race([queryPromise, timeoutPromise]);
                
                return { shardId, rows };
            } catch (error) {
                if (options.failFast) throw error;
                return { shardId, rows: [], error: error as Error };
            }
        });
        
        if (options.failFast) {
            const results = await Promise.all(promises);
            return results;
        } else {
            return await Promise.allSettled(promises).then(settled =>
                settled.map((result, shardId) => 
                    result.status === 'fulfilled' 
                        ? result.value 
                        : { shardId, rows: [], error: (result as PromiseRejectedResult).reason }
                )
            );
        }
    }
    
    // Fan-out: query แบบ parallel แล้ว aggregate
    async fanOut<T, R>(
        query: string,
        params: any[],
        aggregator: (allRows: T[]) => R
    ): Promise<R> {
        const results = await this.execute<T>(query, params);
        const allRows = results.flatMap(r => r.rows);
        return aggregator(allRows);
    }
    
    // Count ทุก shards
    async count(table: string, whereClause?: string, params: any[] = []): Promise<number> {
        const query = `SELECT COUNT(*) as cnt FROM ${table}${whereClause ? ' WHERE ' + whereClause : ''}`;
        return this.fanOut<{cnt: string}, number>(
            query, params,
            rows => rows.reduce((sum, r) => sum + parseInt(r.cnt), 0)
        );
    }
    
    // Top-N ทุก shards
    async topN<T>(
        query: string, // ต้องมี ORDER BY และ LIMIT
        params: any[],
        n: number,
        sortKey: keyof T,
        sortOrder: 'asc' | 'desc' = 'desc'
    ): Promise<T[]> {
        const results = await this.execute<T>(query, params);
        const allRows = results.flatMap(r => r.rows);
        
        allRows.sort((a, b) => {
            const aVal = a[sortKey] as any;
            const bVal = b[sortKey] as any;
            if (sortOrder === 'desc') return bVal > aVal ? 1 : -1;
            return aVal > bVal ? 1 : -1;
        });
        
        return allRows.slice(0, n);
    }
}
```

### 4.2 Aggregation ใน Application

```typescript
// ════════════════════════════════════════════════
// Aggregation: รวมผลจาก Shards ใน Application
// ════════════════════════════════════════════════

interface SalesStats {
    date: string;
    order_count: number;
    total_revenue: number;
    avg_order_value: number;
    unique_customers: number;
}

async function getAggregatedSalesStats(
    shards: Pool[],
    startDate: string,
    endDate: string
): Promise<SalesStats[]> {
    const executor = new ScatterGatherExecutor(shards);
    
    // Query แต่ละ shard
    const shardResults = await executor.execute<{
        date: string;
        order_count: string;
        total_revenue: string;
        unique_customers: string;
    }>(
        `SELECT 
            DATE(created_at) as date,
            COUNT(*) as order_count,
            SUM(total_amount) as total_revenue,
            COUNT(DISTINCT customer_id) as unique_customers
         FROM orders
         WHERE created_at BETWEEN $1 AND $2
           AND status = 'completed'
         GROUP BY DATE(created_at)`,
        [startDate, endDate]
    );
    
    // Aggregate ที่ application layer
    const aggregated = new Map<string, {
        order_count: number;
        total_revenue: number;
        customers: Set<string>;
    }>();
    
    for (const { rows } of shardResults) {
        for (const row of rows) {
            const existing = aggregated.get(row.date) || {
                order_count: 0,
                total_revenue: 0,
                customers: new Set()
            };
            
            existing.order_count += parseInt(row.order_count);
            existing.total_revenue += parseFloat(row.total_revenue);
            // Note: unique_customers ไม่สามารถ sum ตรงๆ ได้
            // ต้องใช้ HyperLogLog หรือ approximate counting
            
            aggregated.set(row.date, existing);
        }
    }
    
    return Array.from(aggregated.entries())
        .map(([date, stats]) => ({
            date,
            order_count: stats.order_count,
            total_revenue: stats.total_revenue,
            avg_order_value: stats.total_revenue / stats.order_count,
            unique_customers: stats.customers.size
        }))
        .sort((a, b) => a.date.localeCompare(b.date));
}
```

---

## 5. Cross-Shard Transactions

### 5.1 Two-Phase Commit (2PC)

```
2PC Protocol:
━━━━━━━━━━━━━

Phase 1 (Prepare):
  Coordinator ──► Shard 1: "Prepare transaction TX123"
  Coordinator ──► Shard 2: "Prepare transaction TX123"
  Coordinator ──► Shard 3: "Prepare transaction TX123"
  
  Shard 1 ──► Coordinator: "Ready" (lock resources, write to WAL)
  Shard 2 ──► Coordinator: "Ready"
  Shard 3 ──► Coordinator: "Ready"

Phase 2 (Commit):
  Coordinator ──► Shard 1: "Commit TX123"
  Coordinator ──► Shard 2: "Commit TX123"
  Coordinator ──► Shard 3: "Commit TX123"

ถ้า Shard ใดตอบว่า "Abort" ใน Phase 1:
  Coordinator ──► ทุก Shard: "Rollback TX123"
```

```sql
-- ════════════════════════════════════════════════
-- PostgreSQL 2PC: PREPARE TRANSACTION
-- ════════════════════════════════════════════════

-- Phase 1: Prepare บน Shard 1
BEGIN;
UPDATE inventory SET quantity = quantity - 5 WHERE product_id = 101;
PREPARE TRANSACTION 'transfer_tx_001';  -- สร้าง prepared transaction

-- Phase 1: Prepare บน Shard 2
BEGIN;
INSERT INTO orders (order_id, customer_id, product_id, quantity) 
VALUES ('ORD-001', 12345, 101, 5);
PREPARE TRANSACTION 'transfer_tx_001';

-- Phase 2: ถ้าทุก shard ready → Commit
COMMIT PREPARED 'transfer_tx_001';  -- บน Shard 1
COMMIT PREPARED 'transfer_tx_001';  -- บน Shard 2

-- Phase 2: ถ้า shard ใด fail → Rollback
ROLLBACK PREPARED 'transfer_tx_001';  -- ทุก shards

-- ดู prepared transactions ที่ค้างอยู่
SELECT * FROM pg_prepared_xacts;
```

```typescript
// ════════════════════════════════════════════════
// 2PC Implementation ใน Node.js
// ════════════════════════════════════════════════

class TwoPhaseCommitCoordinator {
    private prepared: Map<string, PoolClient[]> = new Map();
    
    async executeCrossShardTransaction(
        txId: string,
        operations: Array<{
            shard: Pool;
            queries: Array<{ sql: string; params: any[] }>;
        }>
    ): Promise<void> {
        const clients: PoolClient[] = [];
        
        try {
            // Phase 0: เริ่ม connections
            for (const op of operations) {
                const client = await op.shard.connect();
                clients.push(client);
                await client.query('BEGIN');
            }
            
            // Phase 0: Execute queries
            for (let i = 0; i < operations.length; i++) {
                const { queries } = operations[i];
                const client = clients[i];
                
                for (const { sql, params } of queries) {
                    await client.query(sql, params);
                }
            }
            
            // Phase 1: Prepare ทุก shards
            const preparePromises = clients.map(client =>
                client.query(`PREPARE TRANSACTION '${txId}'`)
            );
            await Promise.all(preparePromises);
            
            this.prepared.set(txId, clients);
            
            // Phase 2: Commit ทุก shards
            const commitPromises = clients.map(client =>
                client.query(`COMMIT PREPARED '${txId}'`)
            );
            await Promise.all(commitPromises);
            
        } catch (error) {
            // Rollback ทุก shards ที่ prepare แล้ว
            const preparedClients = this.prepared.get(txId) || [];
            for (const client of preparedClients) {
                try {
                    await client.query(`ROLLBACK PREPARED '${txId}'`);
                } catch (rollbackErr) {
                    console.error('Rollback failed:', rollbackErr);
                    // ต้อง monitor และ manual rollback
                }
            }
            
            // Rollback clients ที่ยังไม่ได้ prepare
            for (const client of clients) {
                if (!preparedClients.includes(client)) {
                    try {
                        await client.query('ROLLBACK');
                    } catch (err) {}
                }
            }
            
            throw error;
        } finally {
            for (const client of clients) {
                client.release();
            }
            this.prepared.delete(txId);
        }
    }
}
```

### 5.2 Saga Pattern: หลีกเลี่ยง 2PC

```typescript
// ════════════════════════════════════════════════
// Saga Pattern: แทน 2PC ที่ complex เกินไป
// ════════════════════════════════════════════════

// Saga: แบ่ง distributed transaction เป็น local transactions
// แต่ละ step มี compensating transaction ถ้า fail

interface SagaStep {
    execute: () => Promise<void>;
    compensate: () => Promise<void>;  // undo ถ้า step ถัดไป fail
    description: string;
}

class SagaOrchestrator {
    private completed: SagaStep[] = [];
    
    async execute(steps: SagaStep[]): Promise<void> {
        for (const step of steps) {
            try {
                console.log(`Executing: ${step.description}`);
                await step.execute();
                this.completed.push(step);
                console.log(`✅ Completed: ${step.description}`);
            } catch (error) {
                console.error(`❌ Failed: ${step.description}`, error);
                
                // Compensate ทุก steps ที่ทำไปแล้ว (reverse order)
                await this.compensate();
                throw new Error(`Saga failed at: ${step.description}`);
            }
        }
    }
    
    private async compensate(): Promise<void> {
        for (const step of [...this.completed].reverse()) {
            try {
                console.log(`Compensating: ${step.description}`);
                await step.compensate();
            } catch (error) {
                console.error(`⚠️ Compensation failed: ${step.description}`, error);
                // Log for manual intervention
            }
        }
    }
}

// ตัวอย่าง: Order Fulfillment Saga
async function processOrder(orderId: string, customerId: number, productId: number, quantity: number) {
    const orchestrator = new SagaOrchestrator();
    
    let reservationId: string;
    let paymentId: string;
    
    const saga: SagaStep[] = [
        {
            description: 'Reserve inventory',
            execute: async () => {
                // บน inventory shard
                const result = await inventoryShard.query(
                    `INSERT INTO reservations (product_id, quantity, order_id, status)
                     VALUES ($1, $2, $3, 'reserved') RETURNING reservation_id`,
                    [productId, quantity, orderId]
                );
                reservationId = result.rows[0].reservation_id;
            },
            compensate: async () => {
                // ยกเลิก reservation
                await inventoryShard.query(
                    `UPDATE reservations SET status = 'cancelled' WHERE reservation_id = $1`,
                    [reservationId]
                );
            }
        },
        {
            description: 'Process payment',
            execute: async () => {
                // บน payment shard  
                const result = await paymentShard.query(
                    `INSERT INTO payments (customer_id, order_id, amount, status)
                     VALUES ($1, $2, $3, 'completed') RETURNING payment_id`,
                    [customerId, orderId, 1500.00]
                );
                paymentId = result.rows[0].payment_id;
            },
            compensate: async () => {
                // Refund
                await paymentShard.query(
                    `UPDATE payments SET status = 'refunded' WHERE payment_id = $1`,
                    [paymentId]
                );
            }
        },
        {
            description: 'Confirm order',
            execute: async () => {
                // บน order shard (customer's shard)
                const orderShard = router.getShard(customerId);
                await orderShard.query(
                    `UPDATE orders SET status = 'confirmed', payment_id = $1 WHERE order_id = $2`,
                    [paymentId, orderId]
                );
            },
            compensate: async () => {
                const orderShard = router.getShard(customerId);
                await orderShard.query(
                    `UPDATE orders SET status = 'cancelled' WHERE order_id = $1`,
                    [orderId]
                );
            }
        }
    ];
    
    await orchestrator.execute(saga);
}
```

---

## 6. Shard Rebalancing

### 6.1 Consistent Hashing

```
Consistent Hashing:
━━━━━━━━━━━━━━━━━━━

Hash Ring (0 - 360 degrees):

              0 / 360
                 │
        270 ─────┼───── 90
                 │
                180

Nodes บน ring:
  Node A: 45°
  Node B: 135°
  Node C: 225°
  Node D: 315°

Key routing: route ไปยัง node แรกที่อยู่ถัดไปตามเข็มนาฬิกา
  key hash = 60° → Node B (135°) ← node แรกที่ > 60°
  key hash = 200° → Node C (225°)
  key hash = 300° → Node D (315°)
  key hash = 350° → Node A (45° → วนกลับ)

เพิ่ม Node E ที่ 90°:
  key hash = 60° → Node E (90°) ← เปลี่ยนจาก B เป็น E
  key hash = 200° → Node C → ไม่เปลี่ยน!
  
ข้อดี: เพิ่ม node ย้าย data แค่ fraction เดียว ไม่ต้อง reshuffle ทั้งหมด
```

```typescript
// ════════════════════════════════════════════════
// Consistent Hashing Implementation
// ════════════════════════════════════════════════

import * as crypto from 'crypto';

class ConsistentHashRing {
    private ring: Map<number, string> = new Map();
    private sortedKeys: number[] = [];
    private virtualNodes: number; // replicas per physical node
    
    constructor(virtualNodes: number = 150) {
        this.virtualNodes = virtualNodes;
    }
    
    addNode(nodeId: string): void {
        for (let i = 0; i < this.virtualNodes; i++) {
            const virtualKey = `${nodeId}:vnode-${i}`;
            const hash = this.hash(virtualKey);
            this.ring.set(hash, nodeId);
            this.sortedKeys.push(hash);
        }
        this.sortedKeys.sort((a, b) => a - b);
    }
    
    removeNode(nodeId: string): void {
        for (let i = 0; i < this.virtualNodes; i++) {
            const virtualKey = `${nodeId}:vnode-${i}`;
            const hash = this.hash(virtualKey);
            this.ring.delete(hash);
            const idx = this.sortedKeys.indexOf(hash);
            if (idx !== -1) this.sortedKeys.splice(idx, 1);
        }
    }
    
    getNode(key: string): string {
        if (this.ring.size === 0) throw new Error('No nodes in ring');
        
        const hash = this.hash(key);
        
        // หา node แรกที่ hash >= key hash (clockwise)
        for (const nodeHash of this.sortedKeys) {
            if (nodeHash >= hash) {
                return this.ring.get(nodeHash)!;
            }
        }
        
        // Wrap around: ถ้าไม่เจอ → ไปที่ node แรก
        return this.ring.get(this.sortedKeys[0])!;
    }
    
    getNodes(key: string, count: number): string[] {
        if (this.ring.size === 0) return [];
        
        const nodes: string[] = [];
        const seen = new Set<string>();
        const hash = this.hash(key);
        
        // หา starting index
        let startIdx = this.sortedKeys.findIndex(k => k >= hash);
        if (startIdx === -1) startIdx = 0;
        
        for (let i = 0; i < this.sortedKeys.length && nodes.length < count; i++) {
            const idx = (startIdx + i) % this.sortedKeys.length;
            const nodeId = this.ring.get(this.sortedKeys[idx])!;
            
            if (!seen.has(nodeId)) {
                nodes.push(nodeId);
                seen.add(nodeId);
            }
        }
        
        return nodes;
    }
    
    private hash(key: string): number {
        const buf = crypto.createHash('md5').update(key).digest();
        return buf.readUInt32BE(0);
    }
    
    // แสดง distribution ของ keys
    analyzeDistribution(sampleSize: number = 10000): Map<string, number> {
        const counts = new Map<string, number>();
        
        for (let i = 0; i < sampleSize; i++) {
            const node = this.getNode(`key-${i}`);
            counts.set(node, (counts.get(node) || 0) + 1);
        }
        
        return counts;
    }
}

// ทดสอบ consistent hashing
const ring = new ConsistentHashRing(150);
ring.addNode('shard-1');
ring.addNode('shard-2');
ring.addNode('shard-3');

console.log('Distribution before adding shard-4:');
ring.analyzeDistribution().forEach((count, node) => {
    console.log(`  ${node}: ${count} keys (${(count/100).toFixed(1)}%)`);
});

ring.addNode('shard-4');
console.log('\nDistribution after adding shard-4:');
ring.analyzeDistribution().forEach((count, node) => {
    console.log(`  ${node}: ${count} keys (${(count/100).toFixed(1)}%)`);
});
```

### 6.2 Virtual Nodes (Vnodes)

```
Virtual Nodes (Vnodes):
━━━━━━━━━━━━━━━━━━━━━━━

Without vnodes: แต่ละ node ครอบครอง 1 segment
  Node A: 0-90°
  Node B: 90-180°
  Node C: 180-270°
  Node D: 270-360°

ถ้า remove Node B: ข้อมูลทั้งหมดใน 90-180° ย้ายไป Node C
→ Node C load spike, ต้อง transfer ข้อมูลจำนวนมาก

With vnodes (150 virtual nodes per physical node):
  Node A: 10 virtual nodes กระจายทั่ว ring
  Node B: 10 virtual nodes กระจายทั่ว ring
  
ถ้า remove Node B: ข้อมูลกระจายไปทุก nodes อย่างสม่ำเสมอ
→ ไม่มี node ใด overload
```

---

## 7. Shard Lookup: Directory vs Algorithmic

### 7.1 Directory-Based Lookup

```sql
-- ════════════════════════════════════════════════
-- Directory-Based: เก็บ mapping ใน lookup table
-- ════════════════════════════════════════════════

-- Shard directory table (ใน central metadata DB)
CREATE TABLE shard_directory (
    entity_type VARCHAR(50) NOT NULL,   -- 'customer', 'order', etc.
    entity_id BIGINT NOT NULL,
    shard_id INT NOT NULL,
    migrated_at TIMESTAMP,
    PRIMARY KEY (entity_type, entity_id)
);

-- Shard config
CREATE TABLE shard_config (
    shard_id INT PRIMARY KEY,
    host VARCHAR(255) NOT NULL,
    port INT NOT NULL DEFAULT 5432,
    database VARCHAR(100) NOT NULL,
    is_active BOOLEAN DEFAULT TRUE,
    weight INT DEFAULT 100  -- สำหรับ weighted routing
);

INSERT INTO shard_config VALUES
    (0, 'shard-0.db.internal', 5432, 'app_db', true, 100),
    (1, 'shard-1.db.internal', 5432, 'app_db', true, 100),
    (2, 'shard-2.db.internal', 5432, 'app_db', true, 100),
    (3, 'shard-3.db.internal', 5432, 'app_db', true, 100);

-- Application ต้อง query directory ก่อนทุกครั้ง (overhead)
-- แต่ flexible: สามารถ migrate entity ไป shard อื่นได้ง่าย
```

```typescript
// Directory-based lookup with caching
class DirectoryShardRouter {
    private cache: Map<string, number> = new Map(); // entityKey → shardId
    private cacheTTL: number = 300000; // 5 minutes
    private cacheTime: Map<string, number> = new Map();
    
    constructor(
        private directoryDb: Pool,
        private shards: Map<number, Pool>
    ) {}
    
    async getShardForEntity(entityType: string, entityId: number): Promise<Pool> {
        const cacheKey = `${entityType}:${entityId}`;
        const cached = this.cache.get(cacheKey);
        const cacheTime = this.cacheTime.get(cacheKey) || 0;
        
        if (cached !== undefined && Date.now() - cacheTime < this.cacheTTL) {
            return this.shards.get(cached)!;
        }
        
        // Query directory
        const result = await this.directoryDb.query(
            'SELECT shard_id FROM shard_directory WHERE entity_type = $1 AND entity_id = $2',
            [entityType, entityId]
        );
        
        if (result.rows.length === 0) {
            // ไม่มีใน directory → assign shard ใหม่
            const shardId = await this.assignShard(entityType, entityId);
            return this.shards.get(shardId)!;
        }
        
        const shardId = result.rows[0].shard_id;
        this.cache.set(cacheKey, shardId);
        this.cacheTime.set(cacheKey, Date.now());
        
        return this.shards.get(shardId)!;
    }
    
    private async assignShard(entityType: string, entityId: number): Promise<number> {
        // Hash-based assignment
        const hash = parseInt(crypto.createHash('md5')
            .update(`${entityType}:${entityId}`)
            .digest('hex').substring(0, 8), 16);
        const shardId = hash % this.shards.size;
        
        await this.directoryDb.query(
            'INSERT INTO shard_directory (entity_type, entity_id, shard_id) VALUES ($1, $2, $3) ON CONFLICT DO NOTHING',
            [entityType, entityId, shardId]
        );
        
        return shardId;
    }
}
```

### 7.2 Algorithmic Lookup

```typescript
// ════════════════════════════════════════════════
// Algorithmic Lookup: คำนวณ shard ตาม formula
// ════════════════════════════════════════════════

class AlgorithmicShardRouter {
    constructor(private shards: Pool[]) {}
    
    getShardId(entityId: number): number {
        return entityId % this.shards.length;
    }
    
    getShard(entityId: number): Pool {
        return this.shards[this.getShardId(entityId)];
    }
    
    // ข้อเสีย: ถ้าเพิ่ม shard ต้องย้ายข้อมูล (reshuffle)
    // แก้ด้วย: consistent hashing
}
```

---

## 8. Hot Shard Problem

### 8.1 ตรวจสอบ Hot Shards

```sql
-- ════════════════════════════════════════════════
-- Detect Hot Shards
-- ════════════════════════════════════════════════

-- ดู shards ที่มี queries เยอะผิดปกติ (บน coordinator ใน Citus)
SELECT 
    nodename as shard_node,
    nodeport,
    COUNT(*) as query_count,
    AVG(query_duration) as avg_duration_ms
FROM pg_dist_transaction
GROUP BY nodename, nodeport
ORDER BY query_count DESC;

-- ดู shard size distribution (Citus)
SELECT 
    nodename,
    nodeport,
    pg_size_pretty(SUM(shard_size)) as total_size,
    COUNT(*) as shard_count
FROM citus_shards
GROUP BY nodename, nodeport
ORDER BY SUM(shard_size) DESC;

-- หา specific shards ที่มี data เยอะ
SELECT 
    shardid,
    nodename,
    nodeport,
    pg_size_pretty(shard_size) as size
FROM citus_shards
WHERE logicalrelid = 'orders'::regclass
ORDER BY shard_size DESC
LIMIT 10;
```

### 8.2 แก้ Hot Shard

```sql
-- ════════════════════════════════════════════════
-- Splitting Hot Shards (Citus)
-- ════════════════════════════════════════════════

-- ดู shard ที่ hot
SELECT shardid, nodename, pg_size_pretty(shard_size) 
FROM citus_shards 
WHERE logicalrelid = 'orders'::regclass
ORDER BY shard_size DESC LIMIT 5;

-- Split hot shard (Citus 11+)
SELECT citus_split_shard(102008);  -- shardid ที่ต้องการ split

-- ตรวจสอบผล
SELECT shardid, nodename, pg_size_pretty(shard_size)
FROM citus_shards
WHERE logicalrelid = 'orders'::regclass
ORDER BY shard_size DESC LIMIT 10;
```

---

## 9. สรุปและ Best Practices

### 9.1 Sharding Key Selection Guide

```
Checklist เลือก Sharding Key:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

✅ High cardinality: มีค่าหลากหลาย (ไม่ใช่แค่ 5 ค่า)
✅ Even distribution: ไม่มีค่าไหนที่มีข้อมูลเยอะผิดปกติ
✅ Query locality: queries ส่วนใหญ่ใช้ sharding key ใน WHERE clause
✅ Immutable: ค่าไม่เปลี่ยน (ถ้าเปลี่ยนต้อง migrate data)
✅ Co-location friendly: related data ใช้ key เดียวกัน

❌ หลีกเลี่ยง:
❌ Sequential (auto-increment): hotspot ถ้าใช้ range sharding
❌ Timestamp เดียว: ทุก insert ไปสุดปลาย range
❌ Low cardinality: เช่น gender (M/F), status (active/inactive)
❌ Changeable: email (ถ้า user เปลี่ยน email ต้อง migrate shard)
```

### 9.2 เมื่อ Sharding ไม่ใช่คำตอบ

```
พิจารณา Alternatives ก่อน:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. PostgreSQL Table Partitioning (บน single server):
   - Partition by date, region, etc.
   - ไม่ต้อง cross-server queries
   - Simpler operation

2. Read Replicas + Caching:
   - 80% workloads เป็น reads → replicas ช่วยได้
   - Redis cache: ลด DB load 90%+

3. Archiving + Cold Storage:
   - ย้ายข้อมูลเก่าไป archive table หรือ object storage
   - Query บน hot data เท่านั้น

4. Vertical Scaling (Scale Up):
   - Server ใหญ่กว่า, RAM มากกว่า
   - บางครั้งง่ายกว่าและถูกกว่า sharding

Sharding ควรเป็น last resort!
```

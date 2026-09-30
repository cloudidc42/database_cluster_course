# Part 67: Multi-Master Setup

## บทนำ

Multi-Master Replication หรือที่เรียกว่า Active-Active Replication คือสถาปัตยกรรมที่ทุก node ในระบบสามารถรับ write operations ได้พร้อมกัน ซึ่งแตกต่างจาก Primary-Replica ที่มีเพียง Primary เดียวที่รับ write ได้

---

## 1. Multi-Master คืออะไร?

### 1.1 ความหมายและแนวคิด

```
Multi-Master Architecture:
━━━━━━━━━━━━━━━━━━━━━━━━━

Traditional Primary-Replica:          Multi-Master:
                                       
App ──write──► Primary ◄──read── App   App ──write──► Node 1 ◄──sync──► Node 2 ◄──write── App
                  │                          │                                │
              (replicate)               (replicate)                     (replicate)
                  │                          ▼                                ▼
               Replica ◄──read── App    Node 3 ◄──── App              Node 3 ◄──── App
```

### 1.2 ทำไมต้องใช้ Multi-Master?

**กรณีที่ต้องการ Multi-Master:**
- ระบบที่ต้องการ write throughput สูงมาก (เกินกว่า single node รองรับได้)
- Geographic distribution: write ใกล้ user เพื่อ low latency
- High availability ที่ยอมให้ write ได้ทุก node
- Zero-downtime ทุก node แม้บาง node ล่ม

**ตัวอย่าง Use Cases:**
- Global e-commerce: write order ที่ data center ใกล้ลูกค้า
- Gaming: player state update ใน region ใกล้เคียง
- IoT: sensor data ส่งไป node ที่ใกล้ที่สุด

---

## 2. ความท้าทายของ Multi-Master

### 2.1 Write Conflicts

```
Write Conflict ตัวอย่าง:
━━━━━━━━━━━━━━━━━━━━━━━

Node A (Bangkok):                    Node B (Singapore):
t=1ms: UPDATE users SET balance=900  t=1ms: UPDATE users SET balance=850
       WHERE id=1 (balance was 1000)        WHERE id=1 (balance was 1000)

เวลา t=5ms Node A ส่ง change ไป B:  Node B ส่ง change ไป A:
- ค่า balance ควรเป็นเท่าไหร่?
- 900? 850? 850+900-1000=750? ไม่รู้!
- นี่คือ Write Conflict ที่ต้องแก้
```

### 2.2 Split Brain

```
Split Brain Scenario:
━━━━━━━━━━━━━━━━━━━━

             Network Partition
Node A ────────── // ────────── Node B
  │                                │
  │ (คิดว่าตัวเองเป็น master)      │ (คิดว่าตัวเองเป็น master)
  ▼                                ▼
Write operations                Write operations
(ข้อมูลแตกต่างกัน!)              (ข้อมูลแตกต่างกัน!)

หลัง partition heal: ข้อมูลไม่ตรงกัน → ต้องแก้ conflict ทั้งหมด
```

### 2.3 Consistency Models

```
CAP Theorem Triangle:
━━━━━━━━━━━━━━━━━━━━

            Consistency
                /\
               /  \
              /    \
             /      \
   CA ──── / choose  \ ──── CP
          /  2 of 3!  \
         /____________\
   Availability ──── Partition Tolerance
              AP

Multi-Master จำเป็นต้อง choose: CP (strong consistency) หรือ AP (eventual consistency)
```

---

## 3. Solutions สำหรับ Multi-Master

### 3.1 BDR (Bi-Directional Replication)

BDR พัฒนาโดย 2ndQuadrant (ปัจจุบัน EDB) เป็น extension บน pglogical:

```
BDR Cluster Architecture:
━━━━━━━━━━━━━━━━━━━━━━━━━

     Node 1 (Bangkok)
      ┌──────────┐
      │ PG + BDR │
      └────┬─────┘
   ┌───────┤ ◄──────────────────────────────────┐
   ▼       │                                    │
Node 2     │                                 Node 3
(Singapore)│                               (Tokyo)
┌──────────┘                          ┌──────────┐
│ PG + BDR │ ◄──────────────────────► │ PG + BDR │
└──────────┘                          └──────────┘

ทุก node replicate ถึงกันทั้งหมด (full mesh)
```

```sql
-- BDR Installation (EDB Postgres Advanced Server)
-- ต้องมี license จาก EDB

-- postgresql.conf
-- shared_preload_libraries = 'bdr'

-- สร้าง BDR group (บน Node 1)
SELECT bdr.create_node_group('my_cluster');

-- Join node อื่น (บน Node 2)
SELECT bdr.create_node(
    node_name := 'node2',
    local_dsn := 'host=node2 dbname=mydb user=bdruser',
    join_target_dsn := 'host=node1 dbname=mydb user=bdruser'
);

-- ดู nodes ใน cluster
SELECT * FROM bdr.node_summary;

-- Conflict handling
SELECT bdr.alter_node_set_conflict_resolver(
    'node1',
    'update_update',
    'last_update_wins'  -- Last write wins strategy
);
```

**BDR Conflict Resolution Options:**
```sql
-- ตั้งค่า conflict resolution
SELECT bdr.alter_table_conflict_detection(
    relation := 'orders'::regclass,
    method := 'column_modify_timestamp',  -- ใช้ timestamp ตัดสิน
    column_name := 'updated_at'
);

-- Origin-based: prefer changes จาก specific node
SELECT bdr.alter_node_set_conflict_resolver(
    'node_name',
    'conflict_type',
    'node_origin_wins',
    preferred_origin := 'node1'
);
```

### 3.2 Citus: Sharding + Distributed SQL

Citus เป็น extension สำหรับ PostgreSQL ที่ทำ horizontal sharding:

```
Citus Architecture:
━━━━━━━━━━━━━━━━━━

Client Application
       │
       ▼
┌─────────────────┐
│   Coordinator   │  ← รับ SQL queries ทั้งหมด
│   (PostgreSQL)  │
└────────┬────────┘
         │
    ┌────┴────┐
    ▼         ▼
┌────────┐  ┌────────┐
│Worker 1│  │Worker 2│  ← เก็บ shards จริงๆ
│ Shard  │  │ Shard  │
│  1,3,5 │  │  2,4,6 │
└────────┘  └────────┘
```

#### 3.2.1 การติดตั้ง Citus

```bash
# ════════════════════════════════════════════════
# Installation: Citus บน Ubuntu
# ════════════════════════════════════════════════

# Add Citus repository
curl https://install.citusdata.com/community/deb.sh | sudo bash

# Install Citus
sudo apt-get install postgresql-16-citus-12.1

# หรือ Docker
docker pull citusdata/citus:12.1
```

```bash
# docker-compose.yml สำหรับ Citus Cluster
cat > docker-compose.citus.yml << 'EOF'
version: '3.8'

services:
  coordinator:
    image: citusdata/citus:12.1
    environment:
      POSTGRES_PASSWORD: postgres
      POSTGRES_USER: postgres
      POSTGRES_DB: mydb
    ports:
      - "5432:5432"
    volumes:
      - coord_data:/var/lib/postgresql/data
    networks:
      - citus_net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  worker1:
    image: citusdata/citus:12.1
    environment:
      POSTGRES_PASSWORD: postgres
      POSTGRES_USER: postgres
      POSTGRES_DB: mydb
    volumes:
      - worker1_data:/var/lib/postgresql/data
    networks:
      - citus_net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  worker2:
    image: citusdata/citus:12.1
    environment:
      POSTGRES_PASSWORD: postgres
      POSTGRES_USER: postgres
      POSTGRES_DB: mydb
    volumes:
      - worker2_data:/var/lib/postgresql/data
    networks:
      - citus_net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  worker3:
    image: citusdata/citus:12.1
    environment:
      POSTGRES_PASSWORD: postgres
      POSTGRES_USER: postgres
      POSTGRES_DB: mydb
    volumes:
      - worker3_data:/var/lib/postgresql/data
    networks:
      - citus_net

networks:
  citus_net:
    driver: bridge

volumes:
  coord_data:
  worker1_data:
  worker2_data:
  worker3_data:
EOF

docker-compose -f docker-compose.citus.yml up -d
```

#### 3.2.2 Setup Citus Cluster

```sql
-- ════════════════════════════════════════════════
-- Setup บน Coordinator
-- ════════════════════════════════════════════════

-- สร้าง extension
CREATE EXTENSION citus;

-- เพิ่ม worker nodes
SELECT citus_add_node('worker1', 5432);
SELECT citus_add_node('worker2', 5432);
SELECT citus_add_node('worker3', 5432);

-- ตรวจสอบ cluster
SELECT * FROM citus_get_active_worker_nodes();

-- ดู nodes ทั้งหมด
SELECT nodeid, nodename, nodeport, isactive 
FROM pg_dist_node 
ORDER BY nodeid;
```

#### 3.2.3 Distributed Tables

```sql
-- ════════════════════════════════════════════════
-- Distributed Tables: กระจายข้อมูลข้าม workers
-- ════════════════════════════════════════════════

-- สร้าง table ปกติก่อน
CREATE TABLE orders (
    order_id BIGINT NOT NULL,
    customer_id BIGINT NOT NULL,
    product_id BIGINT NOT NULL,
    quantity INT NOT NULL,
    total_amount DECIMAL(10,2) NOT NULL,
    status VARCHAR(50) DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- กระจาย table ด้วย distribution column
-- สำคัญมาก! เลือก distribution key ให้ดี
SELECT create_distributed_table('orders', 'customer_id');
-- ข้อมูลจะถูกกระจายตาม customer_id ไปยัง workers ต่างๆ

-- สร้างตาราง customers เป็น distributed table เช่นกัน
CREATE TABLE customers (
    customer_id BIGINT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE,
    region VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

SELECT create_distributed_table('customers', 'customer_id');

-- Co-location: table ที่มี distribution key เดียวกัน
-- จะถูกเก็บไว้บน shard เดียวกัน
-- ทำให้ JOIN queries ที่ join ด้วย customer_id ทำงานได้บน single worker

-- ตรวจสอบ shard placement
SELECT 
    logicalrelid::regclass as table_name,
    shardid,
    nodename,
    nodeport,
    placementid
FROM pg_dist_shard_placement 
JOIN pg_dist_shard USING (shardid)
WHERE logicalrelid::regclass IN ('orders', 'customers')
ORDER BY table_name, shardid;
```

#### 3.2.4 Reference Tables

```sql
-- ════════════════════════════════════════════════
-- Reference Tables: replicated ไปทุก node
-- ════════════════════════════════════════════════

-- Reference tables: ข้อมูลขนาดเล็กที่ต้องการบน ทุก node
-- (lookup tables, categories, etc.)

CREATE TABLE product_categories (
    category_id INT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    parent_id INT REFERENCES product_categories(category_id)
);

-- สร้างเป็น reference table (copy ไปทุก worker)
SELECT create_reference_table('product_categories');

-- ข้อดี: JOIN กับ distributed tables ได้โดยไม่ต้อง network hop

CREATE TABLE shipping_zones (
    zone_id INT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    base_cost DECIMAL(10,2),
    estimated_days INT
);

SELECT create_reference_table('shipping_zones');

-- ตรวจสอบ reference tables
SELECT logicalrelid::regclass, partmethod 
FROM pg_dist_partition 
WHERE partmethod = 'n';  -- 'n' = reference table
```

#### 3.2.5 Distributed Queries

```sql
-- ════════════════════════════════════════════════
-- Distributed Queries: ทำงานโดย Citus อัตโนมัติ
-- ════════════════════════════════════════════════

-- Query ปกติ - Citus route ไปยัง worker ที่ถูกต้องอัตโนมัติ
SELECT * FROM orders WHERE customer_id = 12345;
-- Citus รู้ว่า customer_id=12345 อยู่บน worker ไหน → ส่งไปตรงๆ

-- Aggregation query - Citus ส่งไปทุก worker แล้ว aggregate ที่ coordinator
SELECT 
    DATE_TRUNC('day', created_at) as order_date,
    COUNT(*) as order_count,
    SUM(total_amount) as total_revenue
FROM orders
WHERE created_at >= NOW() - INTERVAL '30 days'
GROUP BY DATE_TRUNC('day', created_at)
ORDER BY order_date;

-- JOIN ระหว่าง distributed tables (co-located)
SELECT 
    c.name as customer_name,
    COUNT(o.order_id) as order_count,
    SUM(o.total_amount) as total_spent
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.name
ORDER BY total_spent DESC
LIMIT 10;

-- JOIN กับ reference table (ทำงานได้ดีเพราะ reference table อยู่ทุก node)
SELECT 
    o.order_id,
    c.name as customer_name,
    pc.name as category_name,
    o.total_amount
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN product_categories pc ON o.category_id = pc.category_id
WHERE o.status = 'completed'
  AND o.created_at >= NOW() - INTERVAL '7 days';

-- EXPLAIN DISTRIBUTED: ดูว่า query ถูก distribute อย่างไร
EXPLAIN (VERBOSE true) 
SELECT COUNT(*) FROM orders WHERE status = 'pending';
```

#### 3.2.6 Shard Rebalancing

```sql
-- ════════════════════════════════════════════════
-- Shard Rebalancing: เมื่อเพิ่ม worker ใหม่
-- ════════════════════════════════════════════════

-- เพิ่ม worker ใหม่
SELECT citus_add_node('worker4', 5432);

-- ดู shard distribution ก่อน rebalance
SELECT 
    nodename,
    COUNT(*) as shard_count
FROM pg_dist_shard_placement
GROUP BY nodename
ORDER BY nodename;

-- Rebalance shards ไปยัง workers ทั้งหมด (รวม worker ใหม่)
SELECT rebalance_table_shards('orders');
-- หรือ rebalance ทุก tables:
SELECT rebalance_table_shards();

-- ตรวจสอบ progress ระหว่าง rebalance
SELECT * FROM pg_dist_rebalance_progress;

-- ดู shard distribution หลัง rebalance
SELECT 
    nodename,
    COUNT(*) as shard_count
FROM pg_dist_shard_placement
GROUP BY nodename
ORDER BY nodename;
```

#### 3.2.7 Columnar Storage

```sql
-- ════════════════════════════════════════════════
-- Columnar Storage: สำหรับ Analytics workloads
-- ════════════════════════════════════════════════

-- สร้างตาราง columnar (ใช้ columnar storage engine)
CREATE TABLE order_analytics (
    order_id BIGINT,
    customer_id BIGINT,
    product_id BIGINT,
    category_id INT,
    total_amount DECIMAL(10,2),
    quantity INT,
    order_date DATE,
    region VARCHAR(50)
) USING columnar;

-- กระจายเป็น distributed columnar table
SELECT create_distributed_table('order_analytics', 'customer_id');

-- ข้อดี columnar: 
-- - Compression ดีมากสำหรับ analytics data
-- - Query เฉพาะ columns ที่ต้องการ (column pruning)
-- - เหมาะสำหรับ time-series และ analytics

-- เปลี่ยนตาราง row-based เป็น columnar
SELECT alter_columnar_table_set('order_analytics', 
    compression := 'zstd',
    compression_level := 3,
    chunk_group_row_limit := 100000
);
```

---

## 4. Conflict Resolution Strategies

### 4.1 Last-Write-Wins (Timestamp-Based)

```sql
-- ════════════════════════════════════════════════
-- Last-Write-Wins: ใช้ timestamp ตัดสิน
-- ════════════════════════════════════════════════

-- เพิ่ม updated_at column สำหรับ LWW
ALTER TABLE orders ADD COLUMN updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP;
ALTER TABLE orders ADD COLUMN node_id VARCHAR(50) DEFAULT current_setting('app.node_id');

-- Trigger อัพเดท timestamp อัตโนมัติ
CREATE OR REPLACE FUNCTION update_timestamp()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    NEW.node_id = current_setting('app.node_id', true);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER orders_update_timestamp
    BEFORE UPDATE ON orders
    FOR EACH ROW EXECUTE FUNCTION update_timestamp();

-- Conflict resolution function (LWW)
CREATE OR REPLACE FUNCTION resolve_conflict_lww(
    local_updated_at TIMESTAMP,
    remote_updated_at TIMESTAMP,
    remote_data JSONB
) RETURNS BOOLEAN AS $$
BEGIN
    -- ถ้า remote ใหม่กว่า apply remote change
    IF remote_updated_at > local_updated_at THEN
        RETURN TRUE;  -- apply remote
    ELSE
        RETURN FALSE; -- keep local
    END IF;
END;
$$ LANGUAGE plpgsql;
```

### 4.2 Origin-Based Resolution

```sql
-- ════════════════════════════════════════════════
-- Origin-Based: prefer specific origin/node
-- ════════════════════════════════════════════════

-- กำหนด node priority
CREATE TABLE node_priority (
    node_id VARCHAR(50) PRIMARY KEY,
    priority INT NOT NULL,  -- ต่ำกว่า = สำคัญกว่า
    region VARCHAR(50)
);

INSERT INTO node_priority VALUES
    ('node-bkk-1', 1, 'bangkok'),    -- highest priority
    ('node-sgp-1', 2, 'singapore'),
    ('node-tky-1', 3, 'tokyo');

-- Conflict resolver
CREATE OR REPLACE FUNCTION resolve_conflict_origin(
    local_node_id VARCHAR,
    remote_node_id VARCHAR,
    conflict_type TEXT
) RETURNS TEXT AS $$
DECLARE
    local_priority INT;
    remote_priority INT;
BEGIN
    SELECT priority INTO local_priority 
    FROM node_priority WHERE node_id = local_node_id;
    
    SELECT priority INTO remote_priority 
    FROM node_priority WHERE node_id = remote_node_id;
    
    -- Node ที่มี priority ต่ำกว่า (สำคัญกว่า) ชนะ
    IF remote_priority < local_priority THEN
        RETURN 'apply_remote';
    ELSE
        RETURN 'keep_local';
    END IF;
END;
$$ LANGUAGE plpgsql;
```

### 4.3 Custom Conflict Handlers

```sql
-- ════════════════════════════════════════════════
-- Custom Handlers: Logic เฉพาะสำหรับ business rules
-- ════════════════════════════════════════════════

-- ตัวอย่าง: Inventory conflict resolution
-- ถ้า 2 nodes ลด stock พร้อมกัน

CREATE OR REPLACE FUNCTION resolve_inventory_conflict(
    local_stock INT,
    remote_stock INT,
    original_stock INT
) RETURNS INT AS $$
DECLARE
    local_delta INT;
    remote_delta INT;
    combined_delta INT;
    result_stock INT;
BEGIN
    -- คำนวณ delta ของแต่ละ node
    local_delta = local_stock - original_stock;   -- เช่น 100-110 = -10
    remote_delta = remote_stock - original_stock; -- เช่น 95-110 = -15
    
    -- รวม delta ทั้งคู่ (ทั้งสองคนซื้อพร้อมกัน)
    combined_delta = local_delta + remote_delta;  -- -10 + -15 = -25
    result_stock = original_stock + combined_delta; -- 110 + (-25) = 85
    
    -- ป้องกัน negative stock
    IF result_stock < 0 THEN
        result_stock = 0;
        -- TODO: trigger out-of-stock notification
    END IF;
    
    RETURN result_stock;
END;
$$ LANGUAGE plpgsql;
```

---

## 5. Active-Active vs Active-Passive

### 5.1 เปรียบเทียบสถาปัตยกรรม

```
Active-Active (Multi-Master):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Load Balancer
     │
  ┌──┴──┐
  │     │
  ▼     ▼
DB1   DB2    ← ทั้งคู่รับ write
  ↕ sync ↕
  ▲     ▲
  │     │
  └──┬──┘
     │
(all nodes active)

✅ ข้อดี:
- Write throughput สูงกว่า (horizontal scaling)
- ไม่มี downtime ถ้า node หนึ่งล้ม
- Geographical distribution ได้
- ไม่ต้องทำ failover (ทุก node พร้อมทำงาน)

❌ ข้อเสีย:
- Conflict resolution ซับซ้อน
- Eventual consistency (อาจอ่านข้อมูลเก่า)
- Application ต้อง handle conflicts บางครั้ง
- ซับซ้อนในการ debug
```

```
Active-Passive (Primary-Replica):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Application
     │ write
     ▼
  Primary ──► Replica 1 (read-only)
              Replica 2 (read-only)
              Replica 3 (standby - failover)

✅ ข้อดี:
- Simple: ไม่มี conflict
- Strong consistency
- ง่ายต่อ debug
- Read scaling ด้วย replicas

❌ ข้อเสีย:
- Write bottleneck ที่ primary
- Failover มี downtime (ไม่กี่วินาทีถึงนาที)
- Primary ต้องรับ write ทั้งหมด
```

---

## 6. Geographic Distribution Challenges

### 6.1 ปัญหาและ Solutions

```
Geographic Multi-Master Challenges:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

                    Bangkok (Node A)
                        │
                  ~80ms latency
                        │
Singapore (Node B) ────────── Tokyo (Node C)
    ~10ms                         ~60ms
    latency                       latency

ปัญหา:
1. Network latency: replication lag ขึ้นอยู่กับ distance
2. Partition tolerance: ถ้า network ขาด → split brain
3. Consistency: data อาจไม่ consistent ทุก region ทันที

Solutions:
1. Region-aware routing: write ไปยัง node ใกล้เคียง
2. Sticky sessions: user คนเดียวไปที่ node เดิมเสมอ
3. Conflict-free data types: CRDT
4. Business-level partitioning: แต่ละ region ดูแลข้อมูลของตัวเอง
```

### 6.2 CRDT: Conflict-Free Replicated Data Types

```sql
-- ════════════════════════════════════════════════
-- CRDT: Counter ที่ไม่มี conflict
-- ════════════════════════════════════════════════

-- G-Counter (Grow-only counter)
-- แต่ละ node เก็บ increment ของตัวเอง
CREATE TABLE gcounter (
    key VARCHAR(255) PRIMARY KEY,
    node_counters JSONB DEFAULT '{}'::jsonb  -- {"node1": 5, "node2": 3}
);

-- Increment (บน node1)
CREATE OR REPLACE FUNCTION gcounter_increment(p_key VARCHAR, p_node VARCHAR, p_amount INT DEFAULT 1)
RETURNS VOID AS $$
BEGIN
    INSERT INTO gcounter (key, node_counters)
    VALUES (p_key, jsonb_build_object(p_node, p_amount))
    ON CONFLICT (key) DO UPDATE
    SET node_counters = gcounter.node_counters || 
        jsonb_build_object(p_node, 
            COALESCE((gcounter.node_counters->>p_node)::INT, 0) + p_amount
        );
END;
$$ LANGUAGE plpgsql;

-- Read (sum ทุก nodes)
CREATE OR REPLACE FUNCTION gcounter_value(p_key VARCHAR)
RETURNS BIGINT AS $$
DECLARE
    total BIGINT := 0;
    val INT;
BEGIN
    SELECT SUM(value::INT) INTO total
    FROM gcounter,
    LATERAL jsonb_each_text(node_counters)
    WHERE key = p_key;
    
    RETURN COALESCE(total, 0);
END;
$$ LANGUAGE plpgsql;

-- Merge (เมื่อ sync ระหว่าง nodes)
CREATE OR REPLACE FUNCTION gcounter_merge(p_key VARCHAR, remote_counters JSONB)
RETURNS VOID AS $$
BEGIN
    INSERT INTO gcounter (key, node_counters)
    VALUES (p_key, remote_counters)
    ON CONFLICT (key) DO UPDATE
    SET node_counters = (
        -- Take max ของแต่ละ node counter
        SELECT jsonb_object_agg(k, GREATEST(v::INT, COALESCE((gcounter.node_counters->>k)::INT, 0)))
        FROM jsonb_each_text(remote_counters) as t(k, v)
    );
END;
$$ LANGUAGE plpgsql;
```

---

## 7. Eventual Consistency Implications

### 7.1 ความหมายและผลกระทบ

```
Eventual Consistency Timeline:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

T=0: User A (Node A): UPDATE balance = 900 (was 1000)
T=0: User B (Node B): READ balance → 1000 ← ยังเก่า! (เพราะ replication ยังไม่ถึง)
T=5ms: Replication sync complete
T=5ms: User B (Node B): READ balance → 900 ← ถูกต้องแล้ว

ผลกระทบ:
- Read-your-writes: บางครั้ง user อ่านข้อมูลเก่าของตัวเอง
- Stale reads: ข้อมูลที่อ่านอาจล้าหลัง
- Lost updates: ถ้า 2 users แก้พร้อมกัน บางคนอาจสูญเสียการแก้ไข
```

### 7.2 Strategies รับมือ

```typescript
// Application-level strategies สำหรับ Eventual Consistency

// 1. Read-Your-Writes: หลัง write → อ่านจาก node เดิมชั่วคราว
class DatabaseRouter {
    private lastWriteNode: Map<string, string> = new Map(); // userId → nodeId
    private lastWriteTime: Map<string, Date> = new Map();
    
    async getReadConnection(userId: string): Promise<any> {
        const lastWrite = this.lastWriteNode.get(userId);
        const lastWriteTime = this.lastWriteTime.get(userId);
        
        // ถ้าเพิ่ง write ใน 5 วินาทีที่ผ่านมา → อ่านจาก node เดิม
        if (lastWrite && lastWriteTime && 
            Date.now() - lastWriteTime.getTime() < 5000) {
            return this.getConnection(lastWrite);
        }
        
        // ไม่งั้นก็อ่านจาก replica ปกติ
        return this.getReadReplica();
    }
    
    async write(userId: string, nodeId: string, query: string): Promise<void> {
        const conn = this.getConnection(nodeId);
        await conn.query(query);
        
        // Track last write
        this.lastWriteNode.set(userId, nodeId);
        this.lastWriteTime.set(userId, new Date());
    }
}

// 2. Monotonic Reads: รับประกันว่าไม่อ่านข้อมูลเก่ากว่าที่เคยอ่าน
class MonotonicReadSession {
    private readVersion: Map<string, number> = new Map(); // key → version
    
    async read(key: string): Promise<{ data: any; version: number }> {
        const minVersion = this.readVersion.get(key) || 0;
        
        // Request ข้อมูลที่มี version >= minVersion
        const result = await this.db.query(
            'SELECT data, version FROM data WHERE key = $1 AND version >= $2',
            [key, minVersion]
        );
        
        if (result.rows.length > 0) {
            const { data, version } = result.rows[0];
            this.readVersion.set(key, version);
            return { data, version };
        }
        
        throw new Error('Data not available at required version yet');
    }
}
```

---

## 8. Application Changes Required

### 8.1 การปรับ Application สำหรับ Multi-Master

```typescript
// ════════════════════════════════════════════════
// Application Layer: Multi-Master Aware
// ════════════════════════════════════════════════

import { Pool, PoolClient } from 'pg';

interface DatabaseCluster {
    getWriteNode(key?: string): Pool;      // สำหรับ write
    getReadNode(key?: string): Pool;       // สำหรับ read
    getAllNodes(): Pool[];                  // สำหรับ scatter-gather
}

class CitusCluster implements DatabaseCluster {
    private coordinator: Pool;
    
    constructor(coordinatorUrl: string) {
        this.coordinator = new Pool({ connectionString: coordinatorUrl });
    }
    
    getWriteNode(key?: string): Pool {
        // Citus: ส่งทุก write ไปที่ coordinator
        return this.coordinator;
    }
    
    getReadNode(key?: string): Pool {
        // Citus: อ่านผ่าน coordinator ด้วย (มัน route ไปเอง)
        return this.coordinator;
    }
    
    getAllNodes(): Pool[] {
        return [this.coordinator]; // Citus ดูแลการ distribute เอง
    }
}

// ════════════════════════════════════════════════
// Idempotent Operations: สำคัญมากใน Multi-Master
// ════════════════════════════════════════════════

class OrderService {
    constructor(private db: DatabaseCluster) {}
    
    async createOrder(orderData: {
        idempotencyKey: string;  // ← สำคัญ! ป้องกัน duplicate
        customerId: number;
        items: Array<{ productId: number; quantity: number }>;
    }): Promise<{ orderId: string; status: string }> {
        const conn = this.db.getWriteNode();
        
        return await conn.connect().then(async (client: PoolClient) => {
            try {
                await client.query('BEGIN');
                
                // ตรวจสอบ idempotency key ก่อน
                const existing = await client.query(
                    'SELECT order_id FROM orders WHERE idempotency_key = $1',
                    [orderData.idempotencyKey]
                );
                
                if (existing.rows.length > 0) {
                    // ส่ง request เดิม กลับมาแล้ว → return ผลเดิม
                    await client.query('ROLLBACK');
                    return { 
                        orderId: existing.rows[0].order_id, 
                        status: 'already_exists' 
                    };
                }
                
                // สร้าง order ใหม่
                const orderId = crypto.randomUUID();
                await client.query(
                    `INSERT INTO orders (order_id, customer_id, idempotency_key, status)
                     VALUES ($1, $2, $3, 'pending')`,
                    [orderId, orderData.customerId, orderData.idempotencyKey]
                );
                
                await client.query('COMMIT');
                return { orderId, status: 'created' };
                
            } catch (err) {
                await client.query('ROLLBACK');
                throw err;
            } finally {
                client.release();
            }
        });
    }
}
```

---

## 9. When to Use Multi-Master

### 9.1 Decision Framework

```
ควรใช้ Multi-Master เมื่อ:
━━━━━━━━━━━━━━━━━━━━━━━━━━

✅ Write throughput เกิน single node รองรับได้ (>10K TPS)
✅ Geographic distribution จำเป็น (write latency <50ms ทุก region)
✅ Zero downtime requirement แม้ node หนึ่งล้ม
✅ Business ยอมรับ eventual consistency ได้
✅ Team มีความสามารถ operate distributed system

❌ ไม่ควรใช้ Multi-Master เมื่อ:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

❌ Write throughput ยังไม่เกิน single node
❌ Strong consistency เป็น requirement (banking, financial)
❌ Team ไม่มีประสบการณ์ distributed systems
❌ ขนาด system ยังเล็ก (premature optimization)
❌ Budget ไม่พอสำหรับ complex infrastructure
```

---

## 10. Alternatives to Multi-Master

### 10.1 CQRS (Command Query Responsibility Segregation)

```
CQRS แทน Multi-Master:
━━━━━━━━━━━━━━━━━━━━━━

Write Path:                          Read Path:
Commands → Single Primary DB         Queries → Multiple Read Replicas
                                              → Elasticsearch
                                              → Redis Cache

ข้อดี:
- ง่ายกว่า Multi-Master มาก
- ไม่มี conflict
- Read scaling ที่ดี
- Write ยังคง ACID

ข้อเสีย:
- Write ยังอยู่ที่ single node (bottleneck)
- Eventual consistency บน read side
```

### 10.2 Sharding

```
Sharding แทน Multi-Master:
━━━━━━━━━━━━━━━━━━━━━━━━━━

Shard 1 (customers 1-1M):       Shard 2 (customers 1M-2M):
┌────────────────────┐           ┌────────────────────┐
│  Primary + Replica │           │  Primary + Replica │
└────────────────────┘           └────────────────────┘

ข้อดี:
- ไม่มี conflict (แต่ละ shard มี primary ของตัวเอง)
- Linear scaling
- ง่ายกว่า Multi-Master

ข้อเสีย:
- Cross-shard queries ยาก
- Rebalancing ยาก
- Hot shard ปัญหา
```

### 10.3 Read Replicas

```sql
-- ง่ายที่สุด: Primary + Read Replicas
-- เหมาะสำหรับ read-heavy workloads

-- Application routing
-- Write → Primary
-- Read → Replica (round-robin หรือ geographic)

-- PgBouncer config สำหรับ read/write split
-- [databases]
-- mydb_write = host=primary-db port=5432 dbname=mydb
-- mydb_read = host=replica-lb port=5432 dbname=mydb

-- Application code
const writePool = new Pool({ connectionString: process.env.DATABASE_WRITE_URL });
const readPool = new Pool({ connectionString: process.env.DATABASE_READ_URL });

async function getOrder(orderId: string) {
    // ใช้ read replica สำหรับ query ที่ไม่ต้องการ fresh data
    return readPool.query('SELECT * FROM orders WHERE id = $1', [orderId]);
}

async function createOrder(data: OrderData) {
    // ใช้ primary สำหรับ write
    return writePool.query('INSERT INTO orders ...', [...]);
}
```

---

## 11. สรุปและ Best Practices

### 11.1 Multi-Master Decision Matrix

```
Complexity vs Benefit:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
                    
High │    BDR/Multi-Master  │ Only when truly needed
     │    Citus (sharding)  │ 
     │                      │
Com- │    CQRS + CDC        │ Good balance
plex-│    + Read Replicas   │
ity  │                      │
     │    Primary +         │ Start here
Low  │    Read Replicas     │
     └──────────────────────┴─────────────────────
     Low ── Scale/Performance Need ── High
```

### 11.2 Checklist ก่อน Implement Multi-Master

```
ก่อนตัดสินใจใช้ Multi-Master:

□ Benchmark: single node จริงๆ ไม่พอหรือ?
□ Read/Write ratio: ถ้า read-heavy → replicas เพียงพอ
□ Business requirement: ต้องการ eventual consistency ได้ไหม?
□ Team readiness: team เข้าใจ distributed systems?
□ Monitoring: มี observability ที่ดีพอไหม?
□ Testing: มี chaos engineering plan?
□ Rollback plan: ถ้า multi-master ไม่ work จะทำอย่างไร?
□ Cost: infrastructure + operational cost สูงกว่า single node มาก
```

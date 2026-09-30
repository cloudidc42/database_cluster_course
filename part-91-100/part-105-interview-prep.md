# Part 105: Interview Preparation - Database Cluster

## บทนำ

บทนี้รวบรวมคำถาม interview ที่พบบ่อยในหัวข้อ Database Cluster พร้อมคำตอบที่ครอบคลุม เหมาะสำหรับเตรียมตัวสัมภาษณ์งานระดับ Senior/Staff Engineer และ Database Engineer

---

## สารบัญ

1. Database Fundamentals (50 คำถาม)
2. PostgreSQL-Specific (30 คำถาม)
3. Redis Questions (20 คำถาม)
4. System Design Questions
5. Architecture Decision Framework
6. Code Exercises

---

## 1. Database Fundamentals

### 1.1 ACID Properties

**Q1: ACID คืออะไร อธิบายแต่ละ property**

**คำตอบ:**

ACID เป็น properties ที่รับประกันความน่าเชื่อถือของ database transactions:

**A - Atomicity (ความเป็นหน่วยเดียวกัน)**
- Transaction ทั้งหมดต้อง succeed หรือ fail ทั้งหมด ไม่มีทำได้บางส่วน
- ตัวอย่าง: โอนเงิน 1000 บาท ต้องหักจากบัญชี A และเพิ่มในบัญชี B พร้อมกัน ถ้าหักแล้วเพิ่มไม่ได้ ต้อง rollback การหักด้วย
```sql
BEGIN;
UPDATE accounts SET balance = balance - 1000 WHERE id = 1;
UPDATE accounts SET balance = balance + 1000 WHERE id = 2;
COMMIT;  -- ทั้งคู่ succeed หรือ rollback ทั้งคู่
```

**C - Consistency (ความสอดคล้อง)**
- Database ต้องอยู่ใน valid state ก่อนและหลัง transaction
- Constraints, rules และ cascades ต้องผ่านเสมอ
- ตัวอย่าง: balance ต้องไม่ติดลบ (CHECK constraint)

**I - Isolation (การแยกกัน)**
- Transactions ที่ run concurrently ต้องไม่รบกวนกัน
- ระดับ isolation: READ UNCOMMITTED, READ COMMITTED, REPEATABLE READ, SERIALIZABLE
- PostgreSQL default: READ COMMITTED

**D - Durability (ความคงทน)**
- เมื่อ COMMIT แล้ว ข้อมูลต้องถูกเก็บถาวร แม้จะเกิด system failure
- PostgreSQL ใช้ WAL (Write-Ahead Log) รับประกัน durability

---

**Q2: CAP Theorem คืออะไร**

**คำตอบ:**

CAP Theorem (Brewer's Theorem) ระบุว่า distributed system สามารถรับประกันได้แค่ 2 ใน 3 properties:

**C - Consistency**: ทุก node เห็นข้อมูลเหมือนกันในเวลาเดียวกัน (strong consistency)

**A - Availability**: ทุก request ได้รับ response (อาจไม่ใช่ข้อมูลล่าสุด)

**P - Partition Tolerance**: ระบบยังทำงานได้เมื่อ network partition เกิดขึ้น

```
                  Consistency
                      /\
                     /  \
                    /    \
                   /  CA  \
                  /--------\
                 /  CP  AP  \
                /____________\
         Availability     Partition
                          Tolerance

CA: Traditional RDBMS (PostgreSQL standalone)
CP: MongoDB (strong consistency mode), HBase, ZooKeeper
AP: Cassandra, CouchDB, DynamoDB (eventual consistency)
```

**ในทางปฏิบัติ**: Network partitions เกิดขึ้นเสมอ ดังนั้น distributed systems ต้องเลือกระหว่าง CP หรือ AP

PostgreSQL cluster เลือก CP: ใน network partition จะ pause writes เพื่อรักษา consistency

---

**Q3: BASE คืออะไร ต่างจาก ACID อย่างไร**

**คำตอบ:**

BASE เป็น alternative model สำหรับ distributed systems:

**BA - Basically Available**: ระบบรับประกัน availability แม้จะเกิด failures บางส่วน

**S - Soft State**: State ของระบบอาจเปลี่ยนได้แม้ไม่มี input เนื่องจาก eventual consistency

**E - Eventually Consistent**: ระบบจะ consistent ในที่สุด แต่ไม่ใช่ทันที

| Aspect | ACID | BASE |
|--------|------|------|
| Consistency | Strong | Eventual |
| Availability | May sacrifice | Prioritized |
| Scalability | Harder | Easier |
| Use case | Financial, ERP | Social, Analytics |
| Example | PostgreSQL | Cassandra, DynamoDB |

---

**Q4: อธิบาย Database Normalization**

**คำตอบ:**

Normalization คือกระบวนการจัดระเบียบ database เพื่อลด redundancy และ dependency

**1NF (First Normal Form)**
- ทุก column ต้องมี atomic values (ไม่มี repeating groups)
```sql
-- BAD (NOT 1NF)
CREATE TABLE orders (
    id INT,
    items TEXT  -- "apple,banana,cherry"
);

-- GOOD (1NF)
CREATE TABLE order_items (
    order_id INT,
    item_name VARCHAR(100)
);
```

**2NF (Second Normal Form)**
- อยู่ใน 1NF และทุก non-key attribute ขึ้นกับ whole primary key
```sql
-- BAD (NOT 2NF) - customer_name ขึ้นกับแค่ customer_id ไม่ใช่ composite key ทั้งหมด
CREATE TABLE order_items (
    order_id INT,
    product_id INT,
    customer_name VARCHAR(100),  -- ขึ้นกับแค่ order_id
    quantity INT,
    PRIMARY KEY (order_id, product_id)
);

-- GOOD (2NF) - แยก customer ออกไป
CREATE TABLE orders (
    id INT PRIMARY KEY,
    customer_id INT
);
```

**3NF (Third Normal Form)**
- อยู่ใน 2NF และไม่มี transitive dependencies

**BCNF (Boyce-Codd Normal Form)**
- Strict version ของ 3NF

---

**Q5: เมื่อไหรควร Denormalize Database**

**คำตอบ:**

Denormalization เพิ่ม redundancy เพื่อ performance:

**ควร Denormalize เมื่อ:**
1. Read operations เยอะมาก และ JOIN costs สูง
2. Reporting / Analytics queries ที่ต้องการ aggregate data
3. Data ไม่ค่อยเปลี่ยน (low write frequency)
4. Response time critical

**ตัวอย่าง:**
```sql
-- Normalized (slow reads, fast writes)
SELECT
    o.id,
    c.name AS customer_name,
    c.email,
    SUM(oi.quantity * p.price) AS total
FROM orders o
JOIN customers c ON o.customer_id = c.id
JOIN order_items oi ON oi.order_id = o.id
JOIN products p ON oi.product_id = p.id
GROUP BY o.id, c.name, c.email;

-- Denormalized summary table (fast reads)
CREATE TABLE order_summaries (
    order_id INT PRIMARY KEY,
    customer_name VARCHAR(100),
    customer_email VARCHAR(255),
    total_amount DECIMAL(10,2),
    created_at TIMESTAMP
);
```

**วิธีแก้ปัญหา denormalization:**
- Materialized Views (PostgreSQL)
- Application-level caching (Redis)
- Read replicas
- CQRS pattern

---

**Q6: Index Types และเมื่อไหรควรใช้**

**คำตอบ:**

PostgreSQL มี index types หลายชนิด:

**B-tree (Default)**
```sql
CREATE INDEX idx_users_email ON users (email);
-- ใช้สำหรับ: =, <, >, <=, >=, BETWEEN, LIKE 'foo%'
-- ไม่เหมาะกับ: LIKE '%foo', full-text search
```

**Hash**
```sql
CREATE INDEX idx_sessions_token ON sessions USING HASH (token);
-- ใช้สำหรับ: = เท่านั้น
-- เร็วกว่า B-tree สำหรับ equality checks
-- ไม่รองรับ range queries
```

**GiST (Generalized Search Tree)**
```sql
CREATE INDEX idx_locations ON locations USING GIST (coordinates);
-- ใช้สำหรับ: geometric, full-text, range types
-- PostGIS: spatial queries
```

**GIN (Generalized Inverted Index)**
```sql
CREATE INDEX idx_documents_content ON documents USING GIN (to_tsvector('english', content));
CREATE INDEX idx_products_tags ON products USING GIN (tags);  -- Array
CREATE INDEX idx_data_jsonb ON data USING GIN (attributes);  -- JSONB

-- ใช้สำหรับ: full-text search, array/jsonb contains queries
-- Insert/update ช้ากว่า GiST แต่ query เร็วกว่า
```

**BRIN (Block Range Index)**
```sql
CREATE INDEX idx_logs_created_at ON logs USING BRIN (created_at);
-- ใช้สำหรับ: tables ขนาดใหญ่ที่ data เรียงตาม physical order
-- ใช้ storage น้อยมาก
-- เหมาะกับ time-series data
```

**Partial Index**
```sql
-- Index เฉพาะ active users
CREATE INDEX idx_active_users ON users (email) WHERE is_active = true;

-- Index เฉพาะ recent orders
CREATE INDEX idx_recent_orders ON orders (created_at)
WHERE created_at > NOW() - INTERVAL '30 days';
```

**Composite Index**
```sql
CREATE INDEX idx_products_category_price ON products (category_id, price);
-- Column order สำคัญมาก
-- WHERE category_id = 1 AND price > 100  -> ใช้ index ทั้งคู่
-- WHERE price > 100                       -> ไม่ใช้ index (category_id ต้องมาก่อน)
-- WHERE category_id = 1                  -> ใช้ index (leading column)
```

---

**Q7: Transaction Isolation Levels**

**คำตอบ:**

PostgreSQL รองรับ 4 isolation levels:

```sql
-- READ UNCOMMITTED (PostgreSQL treat เหมือน READ COMMITTED)
-- อาจเกิด Dirty Read: อ่านข้อมูลที่ยังไม่ COMMIT

-- READ COMMITTED (Default)
-- ป้องกัน Dirty Read แต่ยังเกิด Non-repeatable Read, Phantom Read
BEGIN;
SELECT balance FROM accounts WHERE id = 1;  -- ได้ 1000
-- Transaction อื่น update balance เป็น 500 และ COMMIT
SELECT balance FROM accounts WHERE id = 1;  -- ได้ 500 (changed!)
COMMIT;

-- REPEATABLE READ
-- ป้องกัน Dirty Read, Non-repeatable Read
-- PostgreSQL implementation ป้องกัน Phantom Read ด้วย
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
BEGIN;
SELECT balance FROM accounts WHERE id = 1;  -- ได้ 1000
-- Transaction อื่น update และ COMMIT
SELECT balance FROM accounts WHERE id = 1;  -- ยังได้ 1000 (snapshot)
COMMIT;

-- SERIALIZABLE (Strongest)
-- ป้องกันทุกอย่าง - transactions ดูเหมือน run ทีละตัว
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

| Level | Dirty Read | Non-repeatable Read | Phantom Read |
|-------|-----------|---------------------|--------------|
| READ UNCOMMITTED | ✓ (allowed) | ✓ | ✓ |
| READ COMMITTED | ✗ | ✓ | ✓ |
| REPEATABLE READ | ✗ | ✗ | ✗ (in PG) |
| SERIALIZABLE | ✗ | ✗ | ✗ |

---

**Q8: Deadlock คืออะไร และป้องกันยังไง**

**คำตอบ:**

Deadlock เกิดเมื่อ 2 transactions รอกันไม่สิ้นสุด:

```
Transaction A:               Transaction B:
LOCK table_1 (success)      LOCK table_2 (success)
LOCK table_2 (waiting...)   LOCK table_1 (waiting...)
          ↑                           ↑
          └─────────── Deadlock ──────┘
```

**การป้องกัน:**

1. **Lock ทรัพยากรตามลำดับเดียวกัน**
```sql
-- GOOD: ทุก transaction lock table_1 ก่อน table_2 เสมอ
BEGIN;
SELECT * FROM table_1 FOR UPDATE;
SELECT * FROM table_2 FOR UPDATE;
COMMIT;
```

2. **ใช้ SELECT FOR UPDATE SKIP LOCKED**
```sql
BEGIN;
SELECT id FROM jobs
WHERE status = 'pending'
ORDER BY created_at
LIMIT 1
FOR UPDATE SKIP LOCKED;  -- ข้าม rows ที่ locked แล้ว
COMMIT;
```

3. **Minimize transaction duration**
```sql
-- ทำ computation ข้างนอก transaction
-- แล้วค่อย update เร็วๆ
BEGIN;
UPDATE accounts SET balance = $new_balance WHERE id = $id;
COMMIT;
```

4. **ใช้ lock_timeout**
```sql
SET lock_timeout = '5s';  -- Fail ถ้า wait lock นานกว่า 5 วินาที
```

---

**Q9: EXPLAIN ANALYZE อธิบายยังไง**

**คำตอบ:**

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, FORMAT TEXT)
SELECT u.name, COUNT(o.id) AS order_count
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE u.created_at > NOW() - INTERVAL '30 days'
GROUP BY u.id, u.name
ORDER BY order_count DESC
LIMIT 10;
```

**ตัวอย่าง Output:**
```
Limit  (cost=1250.50..1250.53 rows=10 width=44) (actual time=45.23..45.24 rows=10 loops=1)
  ->  Sort  (cost=1250.50..1275.50 rows=10000 width=44) (actual time=45.22..45.23 rows=10 loops=1)
        Sort Key: (count(o.id)) DESC
        Sort Method: top-N heapsort  Memory: 25kB
        ->  HashAggregate  (cost=950.00..1050.00 rows=10000 width=44) (actual time=42.10..43.50 rows=9876 loops=1)
              ->  Hash Left Join  (cost=300.00..875.00 rows=10000 width=36) (actual time=5.20..35.40 rows=45000 loops=1)
                    Hash Cond: (o.user_id = u.id)
                    ->  Seq Scan on orders o  (cost=0.00..450.00 rows=45000 width=16) (actual time=0.10..12.50 rows=45000 loops=1)
                    ->  Hash  (cost=250.00..250.00 rows=4000 width=36) (actual time=4.50..4.50 rows=4000 loops=1)
                          ->  Index Scan using idx_users_created_at on users u  (cost=0.43..250.00 rows=4000 width=36) (actual time=0.05..3.20 rows=4000 loops=1)
                                Index Cond: (created_at > (now() - '30 days'::interval))
Planning Time: 1.5 ms
Execution Time: 45.5 ms
```

**วิธีอ่าน EXPLAIN:**
- **cost=X..Y**: X = startup cost, Y = total cost (arbitrary units)
- **actual time=X..Y**: X = time to first row, Y = total time (ms)
- **rows=N**: estimated vs actual rows
- **loops=N**: กี่ครั้งที่ node ทำงาน

**Red flags:**
- Seq Scan บน large table (ควรเป็น Index Scan)
- rows estimate ต่างจาก actual มาก (statistics outdated)
- Hash Join ที่ใช้ disk (work_mem ไม่พอ)
- Nested Loop ที่มี high loops count

---

**Q10: Database Partitioning vs Sharding**

**คำตอบ:**

**Partitioning** (ภายใน database เดียว):
```sql
-- Range Partitioning
CREATE TABLE orders (
    id BIGSERIAL,
    created_at TIMESTAMP NOT NULL,
    customer_id INT,
    total DECIMAL(10,2)
) PARTITION BY RANGE (created_at);

CREATE TABLE orders_2023 PARTITION OF orders
    FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');

CREATE TABLE orders_2024 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

-- List Partitioning
CREATE TABLE customers PARTITION BY LIST (country);
CREATE TABLE customers_th PARTITION OF customers FOR VALUES IN ('TH');
CREATE TABLE customers_sg PARTITION OF customers FOR VALUES IN ('SG');

-- Hash Partitioning
CREATE TABLE user_events PARTITION BY HASH (user_id);
CREATE TABLE user_events_0 PARTITION OF user_events FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE user_events_1 PARTITION OF user_events FOR VALUES WITH (MODULUS 4, REMAINDER 1);
```

**Sharding** (ข้าม database/server):
- ข้อมูลกระจายอยู่บน multiple database servers
- Application ต้องรู้ว่าข้อมูลอยู่บน shard ไหน
- ยาก: cross-shard queries, distributed transactions

| Feature | Partitioning | Sharding |
|---------|-------------|---------|
| Location | Same server | Multiple servers |
| Scalability | Vertical | Horizontal |
| Complexity | Low | High |
| Cross-partition queries | Easy | Hard |
| Use case | Large tables | Very large data |

---

### 1.2 Replication Questions

**Q11: Synchronous vs Asynchronous Replication**

**คำตอบ:**

**Synchronous Replication:**
```
Primary ---(write)---> Primary WAL ---(sync)---> Standby WAL
                                                     ↓
                       COMMIT ack ←── ack ←── Write confirmed
```
- Primary รอ standby confirm ก่อน return success
- Zero data loss
- Higher latency
- ถ้า standby ล่ม อาจหยุด primary ได้

```
-- PostgreSQL synchronous_standby_names
synchronous_standby_names = 'FIRST 1 (standby1, standby2)'
-- FIRST 1: รอ standby ตัวแรกที่ confirm
-- ANY 1: รอ standby ใดก็ได้ 1 ตัว
```

**Asynchronous Replication:**
- Primary return success ทันทีที่ commit
- Standby รับ WAL ในภายหลัง
- Lower latency
- อาจ lose data ถ้า primary crash ก่อน standby รับ

**Replication Lag Query:**
```sql
-- Primary node
SELECT
    pid,
    application_name,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    EXTRACT(EPOCH FROM (now() - write_lag)) AS write_lag_seconds,
    EXTRACT(EPOCH FROM (now() - flush_lag)) AS flush_lag_seconds,
    EXTRACT(EPOCH FROM (now() - replay_lag)) AS replay_lag_seconds
FROM pg_stat_replication;

-- Replica node
SELECT EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp())) AS lag_seconds;
```

---

**Q12: อธิบาย WAL (Write-Ahead Log)**

**คำตอบ:**

WAL เป็น mechanism ที่ PostgreSQL ใช้รับประกัน durability และ replication:

```
Transaction COMMIT flow:
1. Write changes to WAL (append-only log)
2. fsync WAL to disk
3. Return COMMIT success to client
4. Write changes to actual data files (async, background)

สาเหตุที่ write WAL ก่อน data:
- WAL เป็น sequential writes (เร็วมาก)
- Data files เป็น random writes (ช้ากว่า)
- ถ้า crash: replay WAL เพื่อ recover data
```

**WAL ใช้สำหรับ:**
1. Crash Recovery: replay WAL หลัง crash
2. Streaming Replication: ส่ง WAL ไป standby
3. Logical Replication: decode WAL เป็น SQL operations
4. Point-in-time Recovery: replay WAL ถึง specific time

**WAL Configuration:**
```
# postgresql.conf
wal_level = replica          # minimal, replica, logical
max_wal_size = 1GB           # WAL file ขนาดสูงสุด
min_wal_size = 80MB          # WAL file ขนาดต่ำสุด
wal_compression = on         # Compress WAL สำหรับ replication
archive_mode = on            # เก็บ WAL archives
archive_command = 'cp %p /archive/%f'
```

---

## 2. PostgreSQL-Specific Questions

**Q13: MVCC (Multi-Version Concurrency Control) คืออะไร**

**คำตอบ:**

MVCC เป็น mechanism ที่ PostgreSQL ใช้จัดการ concurrent transactions โดยไม่ต้องใช้ read locks:

```
ทุก row มี hidden columns:
- xmin: Transaction ID ที่ INSERT/UPDATE row นี้
- xmax: Transaction ID ที่ DELETE/UPDATE row นี้ (0 = ยังไม่ถูกลบ)
- ctid: Physical location ของ row
```

**การทำงาน:**
```sql
-- Transaction A (xid=100): Read users
SELECT * FROM users WHERE id = 1;
-- เห็น: rows ที่ xmin <= 100 และ xmax = 0 หรือ xmax > 100

-- Transaction B (xid=101): Update user
UPDATE users SET name = 'New Name' WHERE id = 1;
-- สร้าง new version ของ row: xmin=101, old version xmax=101
-- Transaction A ยังเห็น old version (xmin=99, xmax=101)
-- Transaction B เห็น new version

-- VACUUM: ล้าง dead tuples (versions ที่ไม่มี transaction เห็นอีกแล้ว)
```

**ประโยชน์:**
- Readers ไม่ block Writers
- Writers ไม่ block Readers
- High concurrency

**ข้อเสีย:**
- Table bloat: dead tuples สะสม
- VACUUM ต้องทำสม่ำเสมอ
- Transaction ID Wraparound: ต้องระวัง

---

**Q14: อธิบาย Vacuum และ Autovacuum**

**คำตอบ:**

**ทำไมต้องมี Vacuum:**
- MVCC สร้าง dead tuples (old versions ที่ไม่มีใครใช้แล้ว)
- Dead tuples กิน disk space
- Transaction ID wraparound: PostgreSQL ใช้ 32-bit XID วนซ้ำทุก ~2 billion transactions

**VACUUM (Standard):**
```sql
-- Regular VACUUM: mark dead tuples เป็น reusable (ไม่คืน disk space)
VACUUM users;

-- VACUUM FULL: compact table, คืน disk space (ต้องการ exclusive lock)
VACUUM FULL users;

-- VACUUM ANALYZE: vacuum + update statistics
VACUUM ANALYZE users;

-- ดู vacuum statistics
SELECT
    schemaname,
    tablename,
    n_dead_tup,
    n_live_tup,
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

**Autovacuum Configuration:**
```
# postgresql.conf
autovacuum = on
autovacuum_vacuum_threshold = 50           # min dead tuples
autovacuum_vacuum_scale_factor = 0.05      # 5% of table size
autovacuum_analyze_threshold = 50
autovacuum_analyze_scale_factor = 0.02     # 2% for analyze

# Per-table settings
ALTER TABLE high_churn_table SET (
    autovacuum_vacuum_scale_factor = 0.01,  # 1% for busy tables
    autovacuum_vacuum_threshold = 100,
    autovacuum_vacuum_cost_delay = 2        # ms
);
```

---

**Q15: Connection Pooling ใน PostgreSQL**

**คำตอบ:**

PostgreSQL ใช้ process-per-connection model แต่ละ connection สร้าง process ใหม่ (~5-10MB RAM) ทำให้ connection count มี limit:

**PgBouncer (Transaction Pooling):**
```ini
# pgbouncer.ini

[databases]
myapp = host=localhost port=5432 dbname=myapp

[pgbouncer]
listen_addr = *
listen_port = 6432
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt

pool_mode = transaction          # transaction-level pooling
max_client_conn = 1000           # รับได้ 1000 client connections
default_pool_size = 20           # แต่ใช้แค่ 20 backend connections

# Advanced settings
reserve_pool_size = 5
reserve_pool_timeout = 5
max_db_connections = 50
```

**Connection Pooling Modes:**
- **Session mode**: Connection ถูก assign ตลอด session (เหมือนไม่มี pool)
- **Transaction mode**: Connection return กลับ pool หลังทุก transaction (แนะนำ)
- **Statement mode**: Return หลังทุก statement (ใช้กับ multi-statement transactions ไม่ได้)

**เมื่อไหรต้องการ Connection Pool:**
```
Application connections = 100 workers × 10 threads = 1000 connections
PostgreSQL max_connections = 200
-> ต้องการ connection pool
```

---

**Q16: Full-Text Search ใน PostgreSQL**

**คำตอบ:**

```sql
-- สร้าง tsvector column
ALTER TABLE products ADD COLUMN search_vector tsvector;

-- Update search vector
UPDATE products
SET search_vector = 
    setweight(to_tsvector('english', COALESCE(name, '')), 'A') ||
    setweight(to_tsvector('english', COALESCE(description, '')), 'B') ||
    setweight(to_tsvector('english', COALESCE(array_to_string(tags, ' '), '')), 'C');

-- สร้าง GIN index
CREATE INDEX idx_products_fts ON products USING GIN (search_vector);

-- Trigger สำหรับ auto-update
CREATE FUNCTION products_search_vector_update() RETURNS trigger AS $$
BEGIN
    NEW.search_vector :=
        setweight(to_tsvector('english', COALESCE(NEW.name, '')), 'A') ||
        setweight(to_tsvector('english', COALESCE(NEW.description, '')), 'B');
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER products_search_vector_trigger
BEFORE INSERT OR UPDATE ON products
FOR EACH ROW EXECUTE FUNCTION products_search_vector_update();

-- Search query
SELECT
    id,
    name,
    ts_rank(search_vector, query) AS rank,
    ts_headline('english', description, query, 'MaxWords=50') AS highlight
FROM products,
     to_tsquery('english', 'database & cluster') AS query
WHERE search_vector @@ query
ORDER BY rank DESC
LIMIT 10;
```

---

**Q17: Row-Level Security (RLS)**

**คำตอบ:**

RLS ให้ PostgreSQL กรอง rows ตาม current user/role:

```sql
-- Enable RLS
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;

-- Policy: Users เห็นแค่ documents ของตัวเอง
CREATE POLICY user_documents ON documents
    FOR ALL
    TO authenticated_users
    USING (user_id = current_setting('app.current_user_id')::uuid);

-- Policy: Admins เห็นทุกอย่าง
CREATE POLICY admin_all ON documents
    FOR ALL
    TO admin_role
    USING (true);

-- การใช้งาน: ตั้งค่า current user ก่อน query
SET app.current_user_id = 'user-uuid-here';
SELECT * FROM documents;  -- เห็นแค่ documents ของ user นั้น

-- Multi-tenant example
CREATE POLICY tenant_isolation ON orders
    USING (tenant_id = current_setting('app.tenant_id')::int);
```

---

## 3. Redis Questions

**Q18: Redis Data Structures และ Use Cases**

**คำตอบ:**

**String:**
```bash
SET session:user123 "eyJhbGci..."  EX 3600  # Session storage
INCR page:views:home                          # Counter
SETEX cache:product:456 300 '{"name":"..."}'  # Cache
```

**List:**
```bash
LPUSH queue:emails "email1"    # Queue (push front)
RPUSH queue:emails "email2"    # Queue (push back)
BRPOP queue:emails 30          # Blocking pop (consumer)
LRANGE recent:products 0 9     # Recent 10 products
```

**Hash:**
```bash
HSET user:123 name "John" email "john@example.com" age "30"
HGET user:123 name
HGETALL user:123               # User profile
HINCRBY user:123 login_count 1 # Increment field
```

**Set:**
```bash
SADD online:users "user123"    # Online users
SREM online:users "user123"
SISMEMBER online:users "user123"
SMEMBERS online:users
SINTERSTORE active:premium premium:users online:users  # Intersection
```

**Sorted Set:**
```bash
ZADD leaderboard 1500 "player1"    # Score-based ranking
ZADD leaderboard 2000 "player2"
ZRANGE leaderboard 0 9 WITHSCORES REV  # Top 10
ZRANK leaderboard "player1"           # Get rank
ZINCRBY leaderboard 100 "player1"     # Add score
```

**Pub/Sub:**
```bash
SUBSCRIBE channel:notifications
PUBLISH channel:notifications '{"type":"message","from":"user123"}'
```

**Stream:**
```bash
XADD events * type "page_view" user "user123" page "/home"
XREAD COUNT 10 STREAMS events 0
XGROUP CREATE events processors $ MKSTREAM
XREADGROUP GROUP processors worker1 COUNT 10 STREAMS events >
```

---

**Q19: Redis Persistence: RDB vs AOF**

**คำตอบ:**

**RDB (Redis Database Snapshot):**
```conf
# redis.conf
save 900 1      # Save ถ้า 1 key เปลี่ยนภายใน 900 วินาที
save 300 10     # Save ถ้า 10 keys เปลี่ยนภายใน 300 วินาที
save 60 10000   # Save ถ้า 10000 keys เปลี่ยนภายใน 60 วินาที

dbfilename dump.rdb
dir /var/lib/redis
```

- ข้อดี: File เล็ก, restore เร็ว, เหมาะกับ backup
- ข้อเสีย: อาจ lose ข้อมูลล่าสุด ถ้า crash ก่อน save

**AOF (Append Only File):**
```conf
appendonly yes
appendfsync everysec    # Sync ทุก 1 วินาที (แนะนำ)
# appendfsync always   # ทุก write (ช้า แต่ safe สุด)
# appendfsync no       # ให้ OS จัดการ (เร็ว แต่ risky)

auto-aof-rewrite-percentage 100  # Rewrite เมื่อ AOF ใหญ่กว่า 2x
auto-aof-rewrite-min-size 64mb   # Minimum size ก่อน rewrite
```

- ข้อดี: เสีย data น้อยกว่า (สูงสุด 1 วินาที), Durable มากกว่า
- ข้อเสีย: File ใหญ่กว่า, Startup ช้ากว่า

**Hybrid (แนะนำ Production):**
```conf
appendonly yes       # AOF เป็น primary persistence
save 3600 1         # RDB backup ทุก 1 ชั่วโมง
```

---

**Q20: Redis Eviction Policies**

**คำตอบ:**

เมื่อ memory เต็ม Redis ใช้ eviction policy จัดการ:

```conf
maxmemory 2gb
maxmemory-policy allkeys-lru
```

| Policy | คำอธิบาย | Use Case |
|--------|---------|---------|
| noeviction | Return error เมื่อ memory เต็ม | ไม่แนะนำ |
| allkeys-lru | Evict keys ที่ใช้น้อยที่สุดจากทุก keys | Cache ทั่วไป |
| volatile-lru | Evict LRU keys ที่มี TTL | Mixed use |
| allkeys-lfu | Evict keys ที่ถูกใช้น้อยที่สุด (frequency) | Skewed access pattern |
| volatile-lfu | LFU เฉพาะ keys ที่มี TTL | Mixed use |
| allkeys-random | Random eviction จากทุก keys | Random access |
| volatile-random | Random eviction เฉพาะ TTL keys | - |
| volatile-ttl | Evict keys ที่ TTL น้อยที่สุด | - |

**ตัวอย่าง:**
- Pure cache (ทุก key evictable): **allkeys-lru** หรือ **allkeys-lfu**
- Cache + persistent data: **volatile-lru**
- Session store: **volatile-ttl**

---

**Q21: Redis Sentinel vs Redis Cluster**

**คำตอบ:**

**Redis Sentinel:**
```
Master ─── Replica 1
         └─── Replica 2

Sentinel 1 ─── Monitor all nodes
Sentinel 2 ─── Auto-failover
Sentinel 3 ─── Notify clients

ใช้เมื่อ:
- ต้องการ HA สำหรับ single master
- Data ไม่ต้องการ partition (ขนาดเล็ก-กลาง)
- Simple setup
```

```conf
# sentinel.conf
sentinel monitor mymaster 127.0.0.1 6379 2  # quorum = 2
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 60000
sentinel parallel-syncs mymaster 1
```

**Redis Cluster:**
```
Shard 0: Master A ─── Replica A
         Slots: 0-5460

Shard 1: Master B ─── Replica B
         Slots: 5461-10922

Shard 2: Master C ─── Replica C
         Slots: 10923-16383

ใช้เมื่อ:
- Data ขนาดใหญ่เกิน memory ของ single server
- ต้องการ horizontal scaling
- High write throughput
```

| Feature | Sentinel | Cluster |
|---------|---------|---------|
| Scaling | Vertical | Horizontal |
| Multi-key operations | Yes | Limited (same slot) |
| Setup complexity | Low | High |
| Max memory | 1 server | N servers |
| Failover | ≥3 Sentinel nodes | Automatic |

---

## 4. System Design Questions

### 4.1 Design a URL Shortener

**Q22: ออกแบบ URL Shortener เช่น bit.ly**

**Requirements:**
- Shorten URLs: longURL → short code (6-8 chars)
- Redirect: short code → longURL
- Analytics: click count, geographic data
- Scale: 100M URLs, 1B redirects/day

**คำตอบ:**

**1. Capacity Estimation:**
```
Write: 100M URLs / (365 × 24 × 3600) ≈ 3.17 URLs/sec (เล็กมาก)
Read:  1B redirects / (365 × 24 × 3600) ≈ 11,574 req/sec
Read:Write ratio = 3,650:1 (read-heavy)
Storage: 100M × 500 bytes = 50GB
```

**2. Short Code Generation:**
```
Options:
1. Random: uuid → base62 → take first 7 chars
   - ปัญหา: collision possible
   
2. Counter-based: autoincrement → base62
   - Counter: 1 → "1", 62 → "10", ...
   - ปัญหา: predictable, distributed counter ยาก
   
3. Hash-based: MD5(longURL) → take first 7 chars
   - ปัญหา: collision possible
   - แก้: เพิ่ม suffix ถ้า collision
```

**3. Database Schema:**
```sql
CREATE TABLE urls (
    id          BIGSERIAL PRIMARY KEY,
    short_code  VARCHAR(10) UNIQUE NOT NULL,
    long_url    TEXT NOT NULL,
    user_id     UUID,
    created_at  TIMESTAMP DEFAULT NOW(),
    expires_at  TIMESTAMP,
    click_count BIGINT DEFAULT 0
);

CREATE INDEX idx_urls_short_code ON urls (short_code);
CREATE INDEX idx_urls_user_id ON urls (user_id);
```

**4. Architecture:**
```
Client → CDN/CloudFront → ALB → API Servers
                                    │
                              ┌─────┴─────┐
                              │           │
                         Redis Cache  PostgreSQL
                         (hot URLs)   (all URLs)
                              │
                         Analytics DB
                         (ClickHouse/
                          TimescaleDB)
```

**5. Redirect Flow:**
```
1. Client: GET /abc123
2. API: Check Redis cache
   - HIT: Return 301/302 redirect
   - MISS: Query PostgreSQL → cache → return redirect
3. Async: Log click event to queue (Kafka/SQS)
4. Analytics worker: Process click events
```

**6. Redis Caching:**
```python
def get_url(short_code: str) -> str:
    # Check cache
    cached = redis.get(f"url:{short_code}")
    if cached:
        return cached
    
    # Database lookup
    url = db.query("SELECT long_url FROM urls WHERE short_code = $1", short_code)
    if not url:
        raise NotFoundException()
    
    # Cache for 24 hours
    redis.setex(f"url:{short_code}", 86400, url.long_url)
    return url.long_url
```

---

### 4.2 Design a Rate Limiter

**Q23: ออกแบบ Distributed Rate Limiter**

**Requirements:**
- Limit: 100 requests/minute per user/IP
- Distributed: multiple API servers
- Low latency: < 5ms overhead
- Accurate: no significant over/under counting

**Algorithms:**

**Token Bucket:**
```python
def is_allowed_token_bucket(user_id: str) -> bool:
    key = f"ratelimit:token:{user_id}"
    now = time.time()
    
    pipe = redis.pipeline()
    pipe.get(key)
    pipe.ttl(key)
    tokens, ttl = pipe.execute()
    
    tokens = float(tokens or 100)  # Start full
    last_check = now - max(0, 60 - ttl) if ttl > 0 else now
    
    # Refill tokens
    elapsed = now - last_check
    new_tokens = min(100, tokens + elapsed * (100/60))
    
    if new_tokens < 1:
        return False
    
    redis.setex(key, 60, new_tokens - 1)
    return True
```

**Sliding Window Counter (แนะนำ):**
```python
async def is_allowed_sliding_window(identifier: str) -> tuple[bool, int]:
    key = f"ratelimit:{identifier}"
    now = int(time.time() * 1000)  # milliseconds
    window = 60 * 1000  # 1 minute
    limit = 100
    
    async with redis.pipeline(transaction=True) as pipe:
        # Remove old entries
        await pipe.zremrangebyscore(key, 0, now - window)
        # Add current request
        await pipe.zadd(key, {str(now): now})
        # Count requests in window
        await pipe.zcard(key)
        # Set TTL
        await pipe.expire(key, 60)
        
        _, _, count, _ = await pipe.execute()
    
    return count <= limit, count
```

**Fixed Window (Simple แต่ burst problem):**
```python
async def is_allowed_fixed_window(identifier: str) -> bool:
    window = int(time.time() / 60)  # 1-minute windows
    key = f"ratelimit:{identifier}:{window}"
    
    count = await redis.incr(key)
    if count == 1:
        await redis.expire(key, 60)
    
    return count <= 100
```

**Architecture:**
```
Request → API Gateway/Middleware
              │
              ↓
         Rate Limiter
              │
         ┌────┴────┐
         │  Redis  │ ← Shared state across all API servers
         │ Cluster │
         └─────────┘
              │
    ┌─────────┼─────────┐
    │         │         │
  Allow    Reject    Headers:
            429      X-RateLimit-Limit: 100
                     X-RateLimit-Remaining: 45
                     X-RateLimit-Reset: 1234567890
```

---

### 4.3 Design a Distributed Cache

**Q24: ออกแบบ Distributed Cache System**

**ปัญหาที่ต้องแก้:**
- Cache invalidation
- Cache stampede (thundering herd)
- Hot keys
- Cache consistency

**Cache Stampede Solution:**

```python
import asyncio
import time
from typing import Optional

async def get_with_lock(
    redis_client,
    key: str,
    fetch_fn,
    ttl: int = 300,
) -> Any:
    """Cache-aside with distributed lock"""
    
    # Try to get from cache
    value = await redis_client.get(key)
    if value is not None:
        return json.loads(value)
    
    # Try to acquire lock
    lock_key = f"lock:{key}"
    lock_acquired = await redis_client.set(
        lock_key,
        "1",
        nx=True,   # Only set if not exists
        ex=10,     # Lock expires in 10 seconds
    )
    
    if lock_acquired:
        try:
            # Double-check after acquiring lock
            value = await redis_client.get(key)
            if value is not None:
                return json.loads(value)
            
            # Fetch from source
            result = await fetch_fn()
            
            # Store in cache
            await redis_client.setex(key, ttl, json.dumps(result))
            return result
        finally:
            await redis_client.delete(lock_key)
    else:
        # Wait for other process to populate cache
        for _ in range(20):
            await asyncio.sleep(0.5)
            value = await redis_client.get(key)
            if value is not None:
                return json.loads(value)
        
        # Fallback: fetch directly
        return await fetch_fn()
```

**Hot Key Solution:**
```python
# Local cache layer (in-memory)
from cachetools import TTLCache

local_cache = TTLCache(maxsize=1000, ttl=5)  # 5 second local cache

async def get_hot_key(key: str) -> Optional[Any]:
    # Check local cache first (no network)
    if key in local_cache:
        return local_cache[key]
    
    # Check Redis
    value = await redis.get(key)
    if value:
        local_cache[key] = json.loads(value)
        return local_cache[key]
    
    return None
```

---

## 5. Architecture Decisions

### 5.1 Decision Tree

**Q25: เมื่อไหรควรใช้ PostgreSQL vs NoSQL**

**ควรใช้ PostgreSQL เมื่อ:**
- ต้องการ ACID transactions ที่เข้มงวด
- Data มี relationships ซับซ้อน (foreign keys, JOINs)
- Schema ค่อนข้างแน่นอน
- ต้องการ complex queries (aggregations, CTEs, window functions)
- Financial/regulatory data

**ควรใช้ NoSQL เมื่อ:**

| Type | Use Case | ตัวอย่าง |
|------|---------|---------|
| Document (MongoDB) | Flexible schema, nested data | Product catalogs, CMS |
| Key-Value (Redis) | Cache, sessions, leaderboards | Cache layer |
| Wide Column (Cassandra) | Time-series, write-heavy | IoT, logs |
| Graph (Neo4j) | Complex relationships | Social networks |
| Search (Elasticsearch) | Full-text search | E-commerce search |

---

### 5.2 When to Add Read Replicas

```
Decision: Add Read Replicas?

Primary DB CPU > 70%?
    └── Yes → Check read vs write ratio
                └── Read > 60%? → Add read replica
                └── Write-heavy → Consider vertical scaling first

Query response time > 100ms?
    └── Yes → EXPLAIN ANALYZE
                └── Missing indexes? → Add indexes
                └── Table too large? → Partition
                └── Too many connections? → Connection pooling
                └── Server bottleneck? → Read replica

Reporting queries affecting production?
    └── Yes → Add read replica for reporting
```

---

## 6. Code Exercises

**Q26: เขียน Cache-Aside Pattern**

```python
# ตัวอย่าง: Product service ด้วย cache-aside

import json
import logging
from typing import Optional
from datetime import timedelta

logger = logging.getLogger(__name__)

class ProductService:
    def __init__(self, db, cache):
        self.db = db
        self.cache = cache
        self.cache_ttl = timedelta(minutes=5)
    
    async def get_product(self, product_id: str) -> Optional[dict]:
        cache_key = f"product:{product_id}"
        
        # 1. Try cache
        cached = await self.cache.get(cache_key)
        if cached:
            logger.debug(f"Cache HIT: {cache_key}")
            return json.loads(cached)
        
        logger.debug(f"Cache MISS: {cache_key}")
        
        # 2. Query database
        product = await self.db.fetchrow(
            "SELECT * FROM products WHERE id = $1 AND is_active = true",
            product_id
        )
        
        if product is None:
            # Cache negative result (shorter TTL)
            await self.cache.setex(cache_key, 30, "null")
            return None
        
        product_dict = dict(product)
        
        # 3. Write to cache
        await self.cache.setex(
            cache_key,
            int(self.cache_ttl.total_seconds()),
            json.dumps(product_dict, default=str)
        )
        
        return product_dict
    
    async def update_product(self, product_id: str, data: dict) -> dict:
        # Update in database
        product = await self.db.fetchrow(
            """
            UPDATE products
            SET name = $2, price = $3, updated_at = NOW()
            WHERE id = $1
            RETURNING *
            """,
            product_id, data["name"], data["price"]
        )
        
        # Invalidate cache
        cache_key = f"product:{product_id}"
        await self.cache.delete(cache_key)
        
        # Also invalidate list caches
        await self.cache.delete_pattern("products:list:*")
        
        return dict(product)
```

**Q27: เขียน Distributed Rate Limiter**

```python
# Sliding Window Rate Limiter

import time
import asyncio
import redis.asyncio as redis
from typing import Tuple


class DistributedRateLimiter:
    def __init__(
        self,
        redis_client: redis.Redis,
        max_requests: int,
        window_seconds: int,
    ):
        self.redis = redis_client
        self.max_requests = max_requests
        self.window_ms = window_seconds * 1000
    
    async def check(self, identifier: str) -> Tuple[bool, dict]:
        """
        Returns: (is_allowed, rate_limit_headers)
        """
        key = f"ratelimit:sw:{identifier}"
        now = int(time.time() * 1000)
        window_start = now - self.window_ms
        
        async with self.redis.pipeline(transaction=True) as pipe:
            # Remove expired entries
            pipe.zremrangebyscore(key, 0, window_start)
            # Count current requests
            pipe.zcard(key)
            # Add current request
            pipe.zadd(key, {f"{now}-{id(pipe)}": now})
            # Set TTL
            pipe.pexpire(key, self.window_ms)
            
            results = await pipe.execute()
        
        count = results[1]  # Count before adding current
        
        allowed = count < self.max_requests
        remaining = max(0, self.max_requests - count - 1)
        
        # Calculate reset time
        oldest = await self.redis.zrange(key, 0, 0, withscores=True)
        reset_time = int((oldest[0][1] + self.window_ms) / 1000) if oldest else int(time.time()) + 60
        
        headers = {
            "X-RateLimit-Limit": str(self.max_requests),
            "X-RateLimit-Remaining": str(remaining),
            "X-RateLimit-Reset": str(reset_time),
        }
        
        if not allowed:
            headers["Retry-After"] = str(reset_time - int(time.time()))
        
        return allowed, headers


# ตัวอย่างการใช้งาน
async def api_middleware(request):
    limiter = DistributedRateLimiter(
        redis_client=redis.from_url("redis://localhost"),
        max_requests=100,
        window_seconds=60,
    )
    
    identifier = request.headers.get("X-User-ID") or request.client.host
    
    allowed, headers = await limiter.check(identifier)
    
    if not allowed:
        return Response(
            status_code=429,
            headers=headers,
            content={"error": "Too Many Requests"},
        )
    
    response = await process_request(request)
    response.headers.update(headers)
    return response
```

**Q28: Connection Pool Implementation**

```go
// Simple Connection Pool in Go

package pool

import (
    "context"
    "errors"
    "sync"
    "time"
)

type Connection interface {
    Ping(ctx context.Context) error
    Close() error
    IsValid() bool
}

type Pool struct {
    mu          sync.Mutex
    connections chan Connection
    factory     func() (Connection, error)
    maxSize     int
    minSize     int
    maxIdle     time.Duration
}

func NewPool(factory func() (Connection, error), minSize, maxSize int, maxIdle time.Duration) (*Pool, error) {
    p := &Pool{
        connections: make(chan Connection, maxSize),
        factory:     factory,
        maxSize:     maxSize,
        minSize:     minSize,
        maxIdle:     maxIdle,
    }
    
    // Create minimum connections
    for i := 0; i < minSize; i++ {
        conn, err := factory()
        if err != nil {
            return nil, err
        }
        p.connections <- conn
    }
    
    return p, nil
}

func (p *Pool) Get(ctx context.Context) (Connection, error) {
    select {
    case conn := <-p.connections:
        if conn != nil && conn.IsValid() {
            return conn, nil
        }
        // Connection invalid, create new
        return p.factory()
    
    case <-ctx.Done():
        return nil, ctx.Err()
    
    default:
        // Try to create new connection
        if len(p.connections) < p.maxSize {
            return p.factory()
        }
        
        // Wait for available connection
        select {
        case conn := <-p.connections:
            return conn, nil
        case <-ctx.Done():
            return nil, ctx.Err()
        case <-time.After(5 * time.Second):
            return nil, errors.New("connection pool timeout")
        }
    }
}

func (p *Pool) Put(conn Connection) {
    if conn == nil || !conn.IsValid() {
        return
    }
    
    select {
    case p.connections <- conn:
        // Successfully returned to pool
    default:
        // Pool is full, close connection
        conn.Close()
    }
}

func (p *Pool) Close() {
    close(p.connections)
    for conn := range p.connections {
        conn.Close()
    }
}
```

---

## 7. Quick Reference

### Performance Benchmarks (เพื่อตอบ interview)

```
PostgreSQL (typical):
- Simple SELECT by PK: < 1ms
- Index scan (1M rows): 5-20ms
- Full table scan (1M rows): 100ms-1s
- Complex JOIN (multiple tables): 10-100ms

Redis:
- GET/SET: < 1ms
- PIPELINE 100 ops: 1-2ms
- CLUSTER operation: 1-5ms

Connection Pool:
- Max connections PostgreSQL: 200-500 (rule of thumb: cores × 4)
- PgBouncer: support 10,000+ clients → 100 backend
- Redis: 50-200 connections per server

Index size (rule of thumb):
- B-tree index: 10-30% of table size
- GIN index: 50-200% of column size (jsonb)
```

### เรื่องที่ควรรู้ก่อน Interview

1. **อธิบาย ACID ด้วยตัวอย่างจริงได้**
2. **EXPLAIN ANALYZE อ่านได้ หา bottleneck ได้**
3. **Index types รู้ว่าเมื่อไหรใช้อะไร**
4. **Replication lag monitoring ทำยังไง**
5. **Cache invalidation strategies**
6. **CAP theorem และ trade-offs**
7. **System design: URL shortener, rate limiter**
8. **Redis data structures และ use cases**
9. **Connection pooling เหตุผลและ configuration**
10. **Partitioning vs Sharding**

---

## สรุป

การเตรียม interview เรื่อง Database Cluster ต้องมีทั้ง:

1. **Theoretical knowledge**: ACID, CAP, MVCC, WAL
2. **Practical skills**: EXPLAIN ANALYZE, index design, query optimization
3. **System design**: scalability, reliability, maintainability
4. **Code ability**: implement common patterns (cache-aside, rate limiter, connection pool)
5. **War stories**: ประสบการณ์จริงในการ troubleshoot production issues

การฝึก hands-on กับ PostgreSQL และ Redis จริงๆ จะช่วยให้ตอบคำถามได้อย่างมั่นใจและมีความลึก

---

*เนื้อหาส่วนนี้เป็นส่วนหนึ่งของ Database Cluster Course - Advanced Content*

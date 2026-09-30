# Part 29: Query Optimization และ EXPLAIN ANALYZE

## ทำไม Query Optimization จึงสำคัญ?

Query ที่ไม่ optimal คือต้นตอของ performance ปัญหาส่วนใหญ่ใน web application:

- **Slow page loads**: Query ใช้เวลา 5 วินาที แทนที่จะเป็น 10ms
- **Database bottleneck**: CPU/IO สูง ทั้งที่มี resources เหลือ
- **Scalability**: Query ที่ช้าบน 1K records อาจหยุดทำงานบน 10M records
- **Cost**: Cloud database แพงขึ้นเพราะ query inefficient

---

## EXPLAIN: ดู Execution Plan

```sql
EXPLAIN SELECT * FROM products WHERE category_id = 'abc123';
```

**Output:**
```
QUERY PLAN
-------------------------------------------------------------------
Seq Scan on products  (cost=0.00..458.00 rows=1000 width=200)
  Filter: (category_id = 'abc123'::uuid)
```

EXPLAIN แสดง **plan** ที่ planner คิดว่าจะใช้ โดยไม่รัน query จริง

ตัวเลข `(cost=0.00..458.00 rows=1000 width=200)`:
- `0.00` = startup cost (ก่อนจะ return row แรก)
- `458.00` = total cost (estimate รวมทั้งหมด)
- `rows=1000` = จำนวน rows ที่คาดว่าจะ return
- `width=200` = average row size (bytes)

---

## EXPLAIN ANALYZE: ดู Actual Execution

```sql
EXPLAIN ANALYZE SELECT * FROM products WHERE category_id = 'abc123';
```

**Output:**
```
QUERY PLAN
-------------------------------------------------------------------
Seq Scan on products  (cost=0.00..458.00 rows=1000 width=200)
                      (actual time=0.142..12.543 rows=847 loops=1)
  Filter: (category_id = 'abc123'::uuid)
  Rows Removed by Filter: 9153
Planning Time: 0.283 ms
Execution Time: 12.701 ms
```

EXPLAIN ANALYZE **รัน query จริง** และแสดงเวลาจริง:
- `actual time=0.142..12.543` = เวลาจริง (startup..total) ms
- `rows=847` = จำนวน rows จริงที่ return
- `loops=1` = จำนวนครั้งที่ node นี้รัน

---

## EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, FORMAT JSON)
SELECT p.*, c.name as category_name
FROM products p
JOIN categories c ON p.category_id = c.id
WHERE p.status = 'active'
ORDER BY p.created_at DESC
LIMIT 10;
```

**Options:**
- `ANALYZE` = รัน query จริง + timing
- `BUFFERS` = แสดง buffer usage (cache hits/misses)
- `VERBOSE` = แสดงข้อมูลเพิ่มเติม (output columns, etc.)
- `FORMAT JSON` = output เป็น JSON (ใช้กับ tools เช่น explain.dalibo.com)

**Output (simplified):**
```json
[{
  "Plan": {
    "Node Type": "Limit",
    "Startup Cost": 125.48,
    "Total Cost": 125.71,
    "Plan Rows": 10,
    "Actual Startup Time": 2.341,
    "Actual Total Time": 2.389,
    "Plans": [{
      "Node Type": "Sort",
      "Sort Key": ["p.created_at DESC"],
      "Plans": [{
        "Node Type": "Hash Join",
        "Hash Cond": "(p.category_id = c.id)",
        "Plans": [
          {
            "Node Type": "Seq Scan",
            "Relation Name": "products",
            "Filter": "(status = 'active')",
            "Rows Removed by Filter": 1234,
            "Shared Hit Blocks": 458,
            "Shared Read Blocks": 0
          },
          {
            "Node Type": "Hash",
            "Plans": [{
              "Node Type": "Seq Scan",
              "Relation Name": "categories"
            }]
          }
        ]
      }]
    }]
  }
}]
```

---

## อ่าน Execution Plan

### Node Types

```sql
-- 1. Sequential Scan (Seq Scan)
-- อ่านทุก row ของ table
-- ดีเมื่อ: ดึงข้อมูลส่วนใหญ่ของ table
-- แย่เมื่อ: filter แค่ไม่กี่ rows

EXPLAIN SELECT * FROM products;
-- Seq Scan on products (cost=0.00..458.00 rows=10000 width=200)

-- 2. Index Scan
-- ใช้ B-Tree index เดิน traverse tree แล้วไปดึง heap
-- ดีเมื่อ: selective query (ดึงน้อย rows)

EXPLAIN SELECT * FROM products WHERE id = 'abc123';
-- Index Scan using products_pkey on products
--   (cost=0.43..8.45 rows=1 width=200)
--   Index Cond: (id = 'abc123'::uuid)

-- 3. Index Only Scan
-- ดึงข้อมูลจาก index ทั้งหมด ไม่ต้องไป heap
-- เกิดเมื่อ SELECT columns ทั้งหมดอยู่ใน covering index

EXPLAIN SELECT id, email FROM users WHERE email LIKE 'alice%';
-- Index Only Scan using users_email_idx on users
--   (cost=0.43..8.45 rows=5 width=56)
--   Index Cond: (email ~>=~ 'alice'::text)

-- 4. Bitmap Heap Scan
-- สร้าง bitmap ก่อน แล้วดึง heap pages ที่ต้องการ
-- เกิดเมื่อ ดึง rows ปานกลาง (ไม่น้อยไม่มาก)

EXPLAIN SELECT * FROM products WHERE category_id = 'cat1' AND price < 500;
-- Bitmap Heap Scan on products
--   Recheck Cond: (category_id = 'cat1')
--   Filter: (price < 500)
--   ->  Bitmap Index Scan on products_category_id_idx
--         Index Cond: (category_id = 'cat1')

-- 5. CTE Scan / Subquery Scan
EXPLAIN
WITH top_users AS (
  SELECT user_id, COUNT(*) as order_count
  FROM orders
  GROUP BY user_id
  HAVING COUNT(*) > 10
)
SELECT u.*, t.order_count
FROM users u
JOIN top_users t ON u.id = t.user_id;
```

### Join Types

```sql
-- 1. Nested Loop Join
-- Loop: สำหรับแต่ละ row ใน outer table, scan inner table
-- ดีเมื่อ: outer table เล็ก, inner table มี index

EXPLAIN
SELECT o.*, u.email
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.id = 'order-123';
-- Nested Loop
--   -> Index Scan on orders ...
--   -> Index Scan on users ...

-- 2. Hash Join
-- สร้าง hash table จาก smaller table แล้ว probe ด้วย larger table
-- ดีเมื่อ: tables ใหญ่, ไม่มี index บน join column

EXPLAIN
SELECT p.*, c.name as category_name
FROM products p
JOIN categories c ON p.category_id = c.id;
-- Hash Join
--   Hash Cond: (p.category_id = c.id)
--   -> Seq Scan on products
--   -> Hash
--        -> Seq Scan on categories

-- 3. Merge Join
-- Sort ทั้งสอง sides แล้ว merge
-- ดีเมื่อ: ทั้งสองฝั่งมี sorted data (หรือ index)

EXPLAIN
SELECT a.*, b.*
FROM table_a a
JOIN table_b b ON a.id = b.a_id
ORDER BY a.id;
-- Merge Join
--   Merge Cond: (a.id = b.a_id)
--   -> Index Scan on table_a ...
--   -> Index Scan on table_b ...
```

### Buffers: Shared Hit vs Read

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM products WHERE category_id = 'cat1';

-- Shared Hit Blocks: 458   -- อ่านจาก memory (shared_buffers)
-- Shared Read Blocks: 12   -- อ่านจาก disk (I/O)
-- Shared Written Blocks: 0
```

**Shared Hit** = ดีมาก (cache hit)
**Shared Read** = แย่ (disk I/O) — ต้องการ warm up cache หรือเพิ่ม shared_buffers

---

## N+1 Query Problem

ปัญหาที่พบบ่อยที่สุดใน ORM:

```typescript
// ❌ N+1 Problem: 1 query ดึง users + N queries ดึง orders ของแต่ละ user
const users = await prisma.user.findMany();

for (const user of users) {
  // Query แยก 1 ครั้งต่อ 1 user!
  const orders = await prisma.order.findMany({
    where: { userId: user.id },
  });
  console.log(`${user.name}: ${orders.length} orders`);
}
// ถ้ามี 100 users = 101 queries!
```

```sql
-- N+1 ใน raw SQL
-- Query 1: ดึง users
SELECT * FROM users;

-- Query 2-101: ดึง orders สำหรับแต่ละ user
SELECT * FROM orders WHERE user_id = 'user-1';
SELECT * FROM orders WHERE user_id = 'user-2';
-- ... อีก 98 queries
```

### แก้ N+1

```typescript
// ✅ Solution 1: Eager loading ด้วย include
const usersWithOrders = await prisma.user.findMany({
  include: {
    orders: true,  // 1 query JOIN
  },
});

for (const user of usersWithOrders) {
  console.log(`${user.name}: ${user.orders.length} orders`);
}
// ใช้แค่ 1 query!

// SQL ที่ Prisma generate:
// SELECT u.*, o.*
// FROM users u
// LEFT JOIN orders o ON u.id = o.user_id

// ✅ Solution 2: DataLoader pattern (batch + cache)
import DataLoader from 'dataloader';

const orderLoader = new DataLoader<string, Order[]>(async (userIds) => {
  // Batch: ดึงทุก orders ในครั้งเดียว
  const orders = await prisma.order.findMany({
    where: { userId: { in: userIds as string[] } },
  });
  
  // Group by userId
  const ordersByUserId = userIds.map(userId =>
    orders.filter(o => o.userId === userId)
  );
  
  return ordersByUserId;
});

// ใช้ DataLoader — batch อัตโนมัติ
const users = await prisma.user.findMany();
const usersWithOrders = await Promise.all(
  users.map(async user => ({
    ...user,
    orders: await orderLoader.load(user.id),
  }))
);
```

```sql
-- ✅ Solution 3: JOIN แทน separate queries
SELECT
  u.id,
  u.name,
  u.email,
  COUNT(o.id) as order_count,
  SUM(o.total_amount) as total_spent
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.name, u.email;
```

---

## JOIN vs Subquery vs CTE: Performance Comparison

```sql
-- ========== Scenario: ดึง users ที่มี orders มากกว่า 5 รายการ ==========

-- Option 1: JOIN + GROUP BY
EXPLAIN ANALYZE
SELECT DISTINCT u.*
FROM users u
JOIN orders o ON u.id = o.user_id
GROUP BY u.id
HAVING COUNT(o.id) > 5;

-- Option 2: Subquery (correlated)
EXPLAIN ANALYZE
SELECT u.*
FROM users u
WHERE (
  SELECT COUNT(*) FROM orders o WHERE o.user_id = u.id
) > 5;
-- WARNING: รัน subquery 1 ครั้งต่อ 1 user! ช้ามากถ้า users เยอะ

-- Option 3: Subquery (non-correlated) - ดีกว่า
EXPLAIN ANALYZE
SELECT u.*
FROM users u
WHERE u.id IN (
  SELECT user_id
  FROM orders
  GROUP BY user_id
  HAVING COUNT(*) > 5
);

-- Option 4: CTE
EXPLAIN ANALYZE
WITH power_users AS (
  SELECT user_id, COUNT(*) as order_count
  FROM orders
  GROUP BY user_id
  HAVING COUNT(*) > 5
)
SELECT u.*
FROM users u
JOIN power_users pu ON u.id = pu.user_id;

-- Option 5: EXISTS (มักเร็วที่สุดสำหรับ "มีอยู่ไหม")
EXPLAIN ANALYZE
SELECT u.*
FROM users u
WHERE EXISTS (
  SELECT 1
  FROM orders o
  WHERE o.user_id = u.id
  -- ไม่ได้นับ ดังนั้นไม่ใช้สำหรับ COUNT > 5
);
```

**ผลลัพธ์โดยทั่วไป:**
1. Non-correlated subquery / CTE ≈ เร็วสุด
2. JOIN + GROUP BY ≈ ดี
3. Correlated subquery = ช้ามาก (avoid!)

---

## Avoid SELECT *

```sql
-- ❌ SELECT * ดึงทุก column รวมถึง TEXT/JSONB ขนาดใหญ่
EXPLAIN ANALYZE
SELECT * FROM products WHERE category_id = 'cat1';
-- width=850 (ขนาด row ใหญ่มาก)

-- ✅ SELECT เฉพาะที่ต้องการ
EXPLAIN ANALYZE
SELECT id, name, price, status FROM products WHERE category_id = 'cat1';
-- width=80 (ขนาด row เล็กกว่า 10x!)

-- ผลกระทบ:
-- 1. Network transfer ลดลง
-- 2. Memory usage ลดลง
-- 3. Index Only Scan เป็นไปได้มากขึ้น
-- 4. Query planner estimate แม่นขึ้น
```

---

## Pagination: OFFSET vs Cursor-based

```sql
-- ❌ OFFSET pagination: ช้าขึ้นเรื่อยๆ ตามหน้า
EXPLAIN ANALYZE
SELECT * FROM products
ORDER BY created_at DESC
LIMIT 20 OFFSET 10000;
-- PostgreSQL ต้อง scan และ skip 10,000 rows ก่อน!
-- Execution Time: 850ms (page 500)

-- ✅ Cursor-based pagination: เร็วคงที่ทุกหน้า
-- หน้าแรก:
SELECT id, name, price, created_at
FROM products
ORDER BY created_at DESC, id DESC
LIMIT 20;

-- หน้าถัดไป (เอา created_at และ id ของ row สุดท้ายมาใช้):
SELECT id, name, price, created_at
FROM products
WHERE (created_at, id) < ('2024-01-15 10:30:00', 'last-page-id')
ORDER BY created_at DESC, id DESC
LIMIT 20;
-- Execution Time: 1ms (ทุกหน้า)
```

```typescript
// Cursor-based pagination ใน TypeScript
interface CursorPagination {
  limit: number;
  cursor?: {
    createdAt: Date;
    id: string;
  };
}

async function getProductsCursor(params: CursorPagination) {
  const { limit, cursor } = params;
  
  const products = await prisma.product.findMany({
    take: limit + 1,  // ดึงเพิ่ม 1 เพื่อรู้ว่ามีหน้าถัดไปไหม
    cursor: cursor
      ? { id: cursor.id }
      : undefined,
    skip: cursor ? 1 : 0,
    orderBy: [
      { createdAt: 'desc' },
      { id: 'desc' },
    ],
    select: {
      id: true,
      name: true,
      price: true,
      createdAt: true,
    },
  });

  const hasNextPage = products.length > limit;
  if (hasNextPage) products.pop();

  const nextCursor = hasNextPage
    ? {
        createdAt: products[products.length - 1].createdAt,
        id: products[products.length - 1].id,
      }
    : null;

  return {
    products,
    nextCursor,
    hasNextPage,
  };
}
```

---

## Batch Operations

```sql
-- ❌ Insert ทีละ row (N round trips)
INSERT INTO products (name, price) VALUES ('A', 100);
INSERT INTO products (name, price) VALUES ('B', 200);
INSERT INTO products (name, price) VALUES ('C', 300);
-- 3 round trips

-- ✅ Bulk insert (1 round trip)
INSERT INTO products (name, price)
VALUES ('A', 100), ('B', 200), ('C', 300);

-- ✅ INSERT ... SELECT (เร็วมากสำหรับ large datasets)
INSERT INTO product_snapshots (product_id, price, snapshot_date)
SELECT id, price, CURRENT_DATE
FROM products
WHERE status = 'active';

-- ✅ UPDATE จาก VALUES
UPDATE products AS p
SET price = v.new_price
FROM (VALUES
  ('prod-1'::uuid, 150.00),
  ('prod-2'::uuid, 250.00),
  ('prod-3'::uuid, 350.00)
) AS v(id, new_price)
WHERE p.id = v.id;

-- ✅ Upsert
INSERT INTO product_stats (product_id, view_count, last_viewed_at)
VALUES ('prod-1', 1, NOW())
ON CONFLICT (product_id)
DO UPDATE SET
  view_count = product_stats.view_count + 1,
  last_viewed_at = NOW();
```

---

## LATERAL JOIN

```sql
-- LATERAL: ให้ subquery อ้างอิง columns จาก outer query

-- ตัวอย่าง: ดึง 3 products ล่าสุดของแต่ละ category
SELECT c.name, p.name, p.price, p.created_at
FROM categories c
CROSS JOIN LATERAL (
  SELECT name, price, created_at
  FROM products
  WHERE category_id = c.id  -- อ้างอิง c.id จาก outer query
  ORDER BY created_at DESC
  LIMIT 3
) AS p;

-- เปรียบเทียบกับ approach เดิม (Window Function):
SELECT c.name, p.name, p.price, p.created_at
FROM (
  SELECT
    p.*,
    ROW_NUMBER() OVER (PARTITION BY category_id ORDER BY created_at DESC) as rn
  FROM products p
) p
JOIN categories c ON p.category_id = c.id
WHERE p.rn <= 3;

-- LATERAL มักเร็วกว่าสำหรับ top-N per group
```

---

## Materialized CTEs

```sql
-- PostgreSQL 12+: CTE ถูก inline โดย default
-- ใช้ MATERIALIZED เมื่อต้องการ materialize ผลลัพธ์ (เก็บ temp result)

-- Non-materialized (default): planner อาจ push down conditions
WITH recent_orders AS (
  SELECT * FROM orders WHERE created_at > NOW() - INTERVAL '30 days'
)
SELECT u.*, COUNT(o.id) as order_count
FROM users u
LEFT JOIN recent_orders o ON u.id = o.user_id
GROUP BY u.id;

-- Materialized: force execute CTE ก่อน แล้วค่อย join
WITH recent_orders AS MATERIALIZED (
  SELECT * FROM orders WHERE created_at > NOW() - INTERVAL '30 days'
)
SELECT u.*, COUNT(o.id) as order_count
FROM users u
LEFT JOIN recent_orders o ON u.id = o.user_id
GROUP BY u.id;

-- ใช้ MATERIALIZED เมื่อ:
-- 1. CTE ถูกใช้หลายครั้ง (ไม่ต้อง compute ซ้ำ)
-- 2. CTE มี side effects (เช่น INSERT RETURNING)
-- 3. ต้องการ fence plan optimization ของ CTE

-- ใช้ NOT MATERIALIZED เมื่อ:
-- ต้องการให้ planner optimize ได้เต็มที่
WITH active_products AS NOT MATERIALIZED (
  SELECT * FROM products WHERE status = 'active'
)
SELECT * FROM active_products WHERE price < 100;
-- Planner อาจรวมเงื่อนไขเป็น: WHERE status = 'active' AND price < 100
```

---

## pg_stat_statements: Find Slow Queries

```sql
-- Enable extension
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- ตั้งค่าใน postgresql.conf:
-- shared_preload_libraries = 'pg_stat_statements'
-- pg_stat_statements.max = 10000
-- pg_stat_statements.track = all

-- ดู top 10 slowest queries
SELECT
  left(query, 100) as query,
  calls,
  round(total_exec_time::numeric, 2) as total_ms,
  round(mean_exec_time::numeric, 2) as avg_ms,
  round(stddev_exec_time::numeric, 2) as stddev_ms,
  round((100.0 * total_exec_time / sum(total_exec_time) OVER ())::numeric, 2) as pct_total,
  rows,
  shared_blks_hit,
  shared_blks_read
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- ดู queries ที่ cache miss สูง (disk I/O)
SELECT
  left(query, 100) as query,
  calls,
  round(mean_exec_time::numeric, 2) as avg_ms,
  shared_blks_read as disk_reads,
  shared_blks_hit as cache_hits,
  round(100.0 * shared_blks_hit / NULLIF(shared_blks_hit + shared_blks_read, 0), 2) as cache_hit_pct
FROM pg_stat_statements
WHERE shared_blks_read > 0
ORDER BY shared_blks_read DESC
LIMIT 10;

-- Reset statistics
SELECT pg_stat_statements_reset();
```

---

## auto_explain Extension

```sql
-- ติดตั้ง auto_explain เพื่อ log slow queries พร้อม explain plan
-- postgresql.conf:
-- session_preload_libraries = 'auto_explain'
-- auto_explain.log_min_duration = 1000  -- log queries ที่ช้ากว่า 1 วินาที
-- auto_explain.log_analyze = on
-- auto_explain.log_buffers = on
-- auto_explain.log_format = json

-- หรือเปิดใน session:
LOAD 'auto_explain';
SET auto_explain.log_min_duration = 100;  -- 100ms
SET auto_explain.log_analyze = true;

-- ดู logs ใน postgresql log file
```

---

## Query Plan Forcing: pg_hint_plan

```sql
-- ติดตั้ง pg_hint_plan extension
-- yum install pg_hint_plan_16 หรือ compile เอง

-- Force specific scan type
/*+ SeqScan(products) */
SELECT * FROM products WHERE price < 100;

/*+ IndexScan(products products_category_id_idx) */
SELECT * FROM products WHERE category_id = 'cat1';

/*+ IndexOnlyScan(users users_email_key) */
SELECT email FROM users WHERE email LIKE 'alice%';

-- Force specific join type
/*+ HashJoin(products categories) */
SELECT p.*, c.name
FROM products p JOIN categories c ON p.category_id = c.id;

/*+ NestLoop(orders users) */
SELECT o.*, u.name
FROM orders o JOIN users u ON o.user_id = u.id
WHERE o.id = 'order-123';

-- Force join order
/*+ Leading(orders users products) */
SELECT o.*, u.name, p.name
FROM orders o
JOIN users u ON o.user_id = u.id
JOIN order_items oi ON oi.order_id = o.id
JOIN products p ON oi.product_id = p.id
WHERE o.created_at > NOW() - INTERVAL '7 days';
```

---

## Full Examples: Before/After Optimization

### Case 1: Missing Index

```sql
-- ❌ BEFORE: Seq Scan บน orders (ช้ามาก)
EXPLAIN ANALYZE
SELECT * FROM orders
WHERE user_id = 'user-123'
ORDER BY created_at DESC;

-- QUERY PLAN:
-- Sort (cost=2847.14..2852.14 rows=2000)
--   Sort Key: created_at DESC
--   -> Seq Scan on orders (cost=0.00..2745.00 rows=2000)
--        Filter: (user_id = 'user-123')
--        Rows Removed by Filter: 48000
-- Execution Time: 245ms

-- ✅ AFTER: เพิ่ม composite index
CREATE INDEX CONCURRENTLY orders_user_id_created_at_idx
  ON orders(user_id, created_at DESC);

EXPLAIN ANALYZE
SELECT * FROM orders
WHERE user_id = 'user-123'
ORDER BY created_at DESC;

-- QUERY PLAN:
-- Index Scan Backward using orders_user_id_created_at_idx
--   (cost=0.43..45.62 rows=47 width=...)
--   Index Cond: (user_id = 'user-123')
-- Execution Time: 0.8ms  ← 300x faster!
```

### Case 2: N+1 Query

```sql
-- ❌ BEFORE: N+1 approach (application code)
-- 1 query ดึง 1000 users
-- 1000 queries ดึง orders ของแต่ละ user
-- Total: 1001 queries, ~5000ms

-- ✅ AFTER: Single JOIN query
EXPLAIN ANALYZE
SELECT
  u.id,
  u.name,
  u.email,
  COUNT(o.id) as order_count,
  COALESCE(SUM(o.total_amount), 0) as total_spent,
  MAX(o.created_at) as last_order_date
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
  AND o.status != 'cancelled'
GROUP BY u.id, u.name, u.email;

-- Execution Time: 45ms (1 query แทน 1001 queries)
```

### Case 3: SELECT * ที่มี JSONB column ใหญ่

```sql
-- ❌ BEFORE
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM products
WHERE category_id = 'cat1'
LIMIT 20;

-- Execution Time: 85ms
-- Shared Read Blocks: 145 (disk I/O สูง เพราะ JSONB ใหญ่)

-- ✅ AFTER: Select เฉพาะ columns ที่ต้องการ
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, sku, name, price, stock_quantity, status, created_at
FROM products
WHERE category_id = 'cat1'
LIMIT 20;

-- Execution Time: 3ms
-- Shared Read Blocks: 8 (ลดลง 18x!)
```

### Case 4: Correlated Subquery

```sql
-- ❌ BEFORE: Correlated subquery ที่ช้ามาก
EXPLAIN ANALYZE
SELECT p.id, p.name, p.price,
  (SELECT AVG(r.rating) FROM reviews r WHERE r.product_id = p.id) as avg_rating,
  (SELECT COUNT(r.id) FROM reviews r WHERE r.product_id = p.id) as review_count
FROM products p
WHERE p.status = 'active';

-- Execution Time: 12,450ms (รัน subquery 2 ครั้ง ต่อ 1 product!)

-- ✅ AFTER: JOIN กับ aggregated subquery
EXPLAIN ANALYZE
SELECT
  p.id, p.name, p.price,
  COALESCE(r.avg_rating, 0) as avg_rating,
  COALESCE(r.review_count, 0) as review_count
FROM products p
LEFT JOIN (
  SELECT
    product_id,
    AVG(rating)::numeric(3,2) as avg_rating,
    COUNT(*) as review_count
  FROM reviews
  WHERE status = 'published'
  GROUP BY product_id
) r ON p.id = r.product_id
WHERE p.status = 'active';

-- Execution Time: 48ms  ← 260x faster!
```

### Case 5: OFFSET Pagination

```sql
-- ❌ BEFORE: OFFSET pagination บน page 500
EXPLAIN ANALYZE
SELECT id, name, price, created_at
FROM products
WHERE status = 'active'
ORDER BY created_at DESC
LIMIT 20
OFFSET 10000;

-- Execution Time: 890ms (ต้อง skip 10,000 rows)

-- ✅ AFTER: Cursor-based pagination
EXPLAIN ANALYZE
SELECT id, name, price, created_at
FROM products
WHERE status = 'active'
  AND created_at < '2024-01-15 10:30:00'::timestamptz
  AND id < 'cursor-product-id'::uuid
ORDER BY created_at DESC, id DESC
LIMIT 20;

-- Execution Time: 1.2ms (Index ใช้ได้เต็มที่)
```

### Case 6: Wildcard LIKE

```sql
-- ❌ BEFORE: Leading wildcard ใช้ index ไม่ได้
EXPLAIN ANALYZE
SELECT * FROM products WHERE name LIKE '%laptop%';

-- Seq Scan on products (ใช้ Full Text Search หรือ ilike ไม่ได้ใช้ index)
-- Execution Time: 234ms

-- ✅ AFTER: Full Text Search
-- เพิ่ม tsvector column และ GIN index
ALTER TABLE products ADD COLUMN search_vector tsvector;

UPDATE products
SET search_vector = to_tsvector('english', name || ' ' || COALESCE(description, ''));

CREATE INDEX CONCURRENTLY products_search_vector_idx ON products USING GIN(search_vector);

-- Trigger อัพเดต search_vector อัตโนมัติ
CREATE OR REPLACE FUNCTION update_product_search_vector()
RETURNS TRIGGER AS $$
BEGIN
  NEW.search_vector = to_tsvector('english', 
    NEW.name || ' ' || COALESCE(NEW.description, '')
  );
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER products_search_vector_update
  BEFORE INSERT OR UPDATE ON products
  FOR EACH ROW EXECUTE FUNCTION update_product_search_vector();

-- Query ด้วย Full Text Search
EXPLAIN ANALYZE
SELECT * FROM products
WHERE search_vector @@ to_tsquery('english', 'laptop');

-- Bitmap Heap Scan using products_search_vector_idx
-- Execution Time: 3ms ← 78x faster!

-- ✅ AFTER: สำหรับ simple prefix search ใช้ index ได้
EXPLAIN ANALYZE
SELECT * FROM products WHERE name LIKE 'laptop%';  -- prefix only, ไม่มี leading %
-- Index Scan (ถ้ามี index บน name)
```

---

## Query Optimization Checklist

```sql
-- ตรวจสอบ slow queries จาก pg_stat_statements
SELECT
  queryid,
  left(query, 150) as query_snippet,
  calls,
  round(mean_exec_time::numeric, 2) as avg_ms,
  round(max_exec_time::numeric, 2) as max_ms,
  rows / calls as avg_rows
FROM pg_stat_statements
WHERE mean_exec_time > 100  -- queries ที่ใช้เวลา > 100ms
ORDER BY mean_exec_time DESC
LIMIT 20;

-- ตรวจสอบ table stats
SELECT
  schemaname,
  tablename,
  n_live_tup as live_tuples,
  n_dead_tup as dead_tuples,
  round(n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0), 2) as dead_pct,
  last_vacuum,
  last_autovacuum,
  last_analyze,
  last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;

-- ตรวจสอบ index usage
SELECT
  schemaname,
  tablename,
  indexname,
  idx_scan as scans,
  idx_tup_read as tuples_read,
  idx_tup_fetch as tuples_fetched,
  pg_size_pretty(pg_relation_size(indexrelid)) as index_size
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC  -- unused indexes ด้านบน
LIMIT 20;

-- ตรวจสอบ cache hit ratio
SELECT
  sum(blks_hit) as cache_hits,
  sum(blks_read) as disk_reads,
  round(100.0 * sum(blks_hit) / NULLIF(sum(blks_hit) + sum(blks_read), 0), 2) as hit_ratio
FROM pg_stat_database
WHERE datname = current_database();
-- ควรมากกว่า 99% ใน production

-- ตรวจสอบ long-running queries
SELECT
  pid,
  now() - pg_stat_activity.query_start AS duration,
  query,
  state
FROM pg_stat_activity
WHERE (now() - pg_stat_activity.query_start) > INTERVAL '5 minutes'
  AND state != 'idle';
```

---

## สรุป Query Optimization Tips

1. **อ่าน EXPLAIN ANALYZE ก่อน optimize** — รู้ปัญหาก่อนแก้
2. **Index คือ solution หลัก** — แต่ index มีต้นทุน (write overhead, disk space)
3. **แก้ N+1 ด้วย JOIN หรือ DataLoader** — ลด round trips
4. **หลีก SELECT *** — ดึงเฉพาะที่ต้องการ
5. **Cursor pagination แทน OFFSET** — สำหรับ large datasets
6. **Batch operations** — ลด round trips
7. **Non-correlated subquery ดีกว่า correlated** — รัน 1 ครั้งแทน N ครั้ง
8. **Full Text Search แทน LIKE '%keyword%'** — GIN index รองรับ
9. **Monitor ด้วย pg_stat_statements** — หา hotspot queries
10. **VACUUM และ ANALYZE สม่ำเสมอ** — statistics accurate = plan ดี

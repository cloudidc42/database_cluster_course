# Part 88: Zero-Downtime Migrations

## บทนำ

Zero-downtime migration คือการเปลี่ยนแปลง database schema โดยไม่ lock production database และไม่ทำให้ application หยุดทำงาน เป็นหนึ่งในทักษะที่สำคัญที่สุดสำหรับ DBA และ Backend Engineer ในยุคปัจจุบัน

---

## 1. ปัญหาของ Naive Migrations

### ทำไม ALTER TABLE ถึงอันตราย

```sql
-- ตัวอย่าง: ตาราง orders ที่มี 50 ล้าน rows
-- DANGEROUS: blocks ALL reads and writes

-- ตรวจสอบ lock level ของแต่ละ operation
SELECT 
    command,
    lock_type,
    duration_estimate
FROM (VALUES
    ('ALTER TABLE ADD COLUMN NULL',      'ACCESS EXCLUSIVE', 'milliseconds'),
    ('ALTER TABLE ADD COLUMN NOT NULL',  'ACCESS EXCLUSIVE', 'minutes-hours (PG<11)'),
    ('ALTER TABLE ADD COLUMN DEFAULT',   'ACCESS EXCLUSIVE', 'milliseconds (PG11+)'),
    ('ALTER TABLE DROP COLUMN',          'ACCESS EXCLUSIVE', 'milliseconds'),
    ('ALTER TABLE ADD CONSTRAINT CHECK', 'ACCESS EXCLUSIVE', 'minutes (validates all rows)'),
    ('CREATE INDEX',                     'ACCESS EXCLUSIVE', 'minutes-hours'),
    ('CREATE INDEX CONCURRENTLY',        'SHARE UPDATE EXCLUSIVE', 'minutes-hours, no blocking'),
    ('ALTER TABLE ADD FK',               'ACCESS EXCLUSIVE', 'minutes (validates all rows)')
) AS t(command, lock_type, duration_estimate);
```

### Lock Types ใน PostgreSQL

```
ACCESS SHARE           - SELECT: ป้องกัน ACCESS EXCLUSIVE เท่านั้น
ROW SHARE              - SELECT FOR UPDATE: ป้องกัน EXCLUSIVE, ACCESS EXCLUSIVE
ROW EXCLUSIVE          - INSERT, UPDATE, DELETE
SHARE UPDATE EXCLUSIVE - VACUUM, CREATE INDEX CONCURRENTLY
SHARE                  - CREATE INDEX (non-concurrent)
SHARE ROW EXCLUSIVE    - CREATE TRIGGER
EXCLUSIVE              - REFRESH MATERIALIZED VIEW
ACCESS EXCLUSIVE       - ALTER TABLE, DROP TABLE, TRUNCATE
                         LOCK ชนิดนี้กัน LOCK ทุกชนิด
                         → หยุด ALL reads AND writes
```

### จำลองปัญหา

```sql
-- Session 1: Long-running transaction
BEGIN;
SELECT COUNT(*) FROM orders;  -- ถือ ACCESS SHARE lock
-- ... ไม่ commit นาน ...

-- Session 2: Tries to ALTER TABLE
ALTER TABLE orders ADD COLUMN notes TEXT;
-- นี่จะรอ SESSION 1 ปล่อย lock
-- ระหว่างนี้ SESSION 3+ จะ queue ด้วย!

-- Session 3: Simple SELECT ก็ถูก block!
SELECT * FROM orders LIMIT 1;
-- Block เพราะ ALTER TABLE รอ ACCESS SHARE lock อยู่ก่อน

-- ดู locks ที่กำลังรอ
SELECT 
    pid,
    now() - pg_stat_activity.query_start AS duration,
    query,
    state,
    wait_event_type,
    wait_event
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY duration DESC;
```

---

## 2. Safe Migration Patterns

### Pattern 1: ADD COLUMN NULL ก่อน

```sql
-- ❌ WRONG: blocks table for full rewrite
ALTER TABLE orders ADD COLUMN notes TEXT NOT NULL DEFAULT '';

-- ✅ CORRECT: Step-by-step approach

-- Step 1: ADD COLUMN NULL (fast, milliseconds)
ALTER TABLE orders ADD COLUMN notes TEXT;

-- Step 2: Backfill ด้วย batch UPDATE
DO $$
DECLARE
    batch_size INT := 10000;
    min_id BIGINT;
    max_id BIGINT;
    current_id BIGINT;
BEGIN
    SELECT MIN(id), MAX(id) INTO min_id, max_id FROM orders WHERE notes IS NULL;
    
    current_id := min_id;
    WHILE current_id <= max_id LOOP
        UPDATE orders
        SET notes = ''
        WHERE id BETWEEN current_id AND current_id + batch_size - 1
            AND notes IS NULL;
        
        RAISE NOTICE 'Backfilled up to id: %', current_id + batch_size - 1;
        
        -- Small sleep เพื่อไม่ให้ lock นานเกินไป
        PERFORM pg_sleep(0.01);
        
        current_id := current_id + batch_size;
    END LOOP;
END $$;

-- Step 3: ADD NOT NULL constraint (validate separately)
-- ก่อน PG17: ต้องใช้ NOT VALID
ALTER TABLE orders 
    ADD CONSTRAINT orders_notes_not_null 
    CHECK (notes IS NOT NULL) NOT VALID;

-- Validate (low-lock, ใช้ SHARE UPDATE EXCLUSIVE)
ALTER TABLE orders VALIDATE CONSTRAINT orders_notes_not_null;

-- หรือ PG17+ ทำได้โดยตรง
-- ALTER TABLE orders ALTER COLUMN notes SET NOT NULL;
-- (ถ้า constraint ถูก validate แล้ว จะ fast)
```

### Pattern 2: ADD DEFAULT Value

```sql
-- PostgreSQL 11+: ADD COLUMN WITH DEFAULT เป็น instant!
-- ก่อน PG11: ต้องทำ full table rewrite

-- PG11+: ✅ SAFE (instant)
ALTER TABLE orders ADD COLUMN status VARCHAR(20) DEFAULT 'pending';

-- ตรวจสอบ PostgreSQL version ก่อน
SHOW server_version;

-- สำหรับ PG < 11:
-- Step 1: Add column without default
ALTER TABLE orders ADD COLUMN status VARCHAR(20);

-- Step 2: Backfill
UPDATE orders SET status = 'pending' WHERE status IS NULL;

-- Step 3: Set default (fast, just metadata change)
ALTER TABLE orders ALTER COLUMN status SET DEFAULT 'pending';

-- Step 4: Set NOT NULL (หลัง backfill แล้ว)
ALTER TABLE orders ALTER COLUMN status SET NOT NULL;
```

### ตรวจสอบว่า Migration ปลอดภัย

```sql
-- Function ตรวจสอบว่า operation จะ lock table นานแค่ไหน
CREATE OR REPLACE FUNCTION check_migration_safety(table_name TEXT)
RETURNS TABLE (
    metric TEXT,
    value BIGINT
)
LANGUAGE sql AS $$
    SELECT 'row_count', COUNT(*)::BIGINT FROM pg_class 
    JOIN pg_namespace ON pg_class.relnamespace = pg_namespace.oid
    WHERE relname = split_part(table_name, '.', 2)
      AND nspname = split_part(table_name, '.', 1)
    UNION ALL
    SELECT 'table_size_mb', 
           pg_total_relation_size(table_name::regclass) / (1024*1024)
    UNION ALL
    SELECT 'active_connections',
           COUNT(*)::BIGINT FROM pg_stat_activity WHERE state != 'idle'
$$;

-- ใช้งาน
SELECT * FROM check_migration_safety('public.orders');
```

---

## 3. Expand-Contract Pattern

### ภาพรวม

```
Expand-Contract Pattern แก้ปัญหา: เปลี่ยน schema โดยต้อง deploy หลายครั้ง

Timeline:
Version 1.0 ─────────── Version 1.1 ─────────── Version 2.0
     │                        │                        │
     │    [Phase 1: Expand]   │  [Phase 2: Migrate]    │  [Phase 3: Contract]
     │                        │                        │
old column              old + new column           new column only
(read/write)            (write both, read old)     (read/write new)
```

### Phase 1: Expand (เพิ่ม column ใหม่)

```sql
-- สมมติ: ต้องการเปลี่ยน phone VARCHAR → phone_number JSONB
-- เพื่อรองรับ multiple phones

-- Phase 1: เพิ่ม column ใหม่ (backward compatible)
ALTER TABLE customers ADD COLUMN phone_data JSONB;

-- ตอนนี้ทั้ง phone (เก่า) และ phone_data (ใหม่) มีอยู่ด้วยกัน
-- Application version 1.1: เขียนทั้ง 2 columns, อ่านจาก phone (เก่า)
```

```typescript
// Application v1.1: Write Both, Read Old
async function updateCustomerPhone(customerId: number, phone: string) {
    await db.query(`
        UPDATE customers
        SET 
            phone = $1,  -- เขียน column เก่า
            phone_data = $2::jsonb  -- เขียน column ใหม่ด้วย
        WHERE id = $3
    `, [
        phone,
        JSON.stringify({ primary: phone, type: 'mobile' }),
        customerId
    ]);
}

async function getCustomerPhone(customerId: number): Promise<string> {
    const result = await db.query(
        'SELECT phone FROM customers WHERE id = $1',
        [customerId]
    );
    return result.rows[0].phone;  // อ่านจาก column เก่า
}
```

### Phase 2: Migrate (Backfill + Dual Write)

```sql
-- Phase 2a: Backfill ข้อมูลเก่าไปยัง column ใหม่
DO $$
DECLARE
    batch_size INT := 5000;
    last_id BIGINT := 0;
    max_id BIGINT;
    rows_updated INT;
BEGIN
    SELECT MAX(id) INTO max_id FROM customers;
    
    LOOP
        UPDATE customers
        SET phone_data = jsonb_build_object(
            'primary', phone,
            'type', 'mobile',
            'migrated', true
        )
        WHERE id > last_id
            AND id <= last_id + batch_size
            AND phone IS NOT NULL
            AND phone_data IS NULL;
        
        GET DIAGNOSTICS rows_updated = ROW_COUNT;
        
        EXIT WHEN last_id >= max_id;
        last_id := last_id + batch_size;
        
        RAISE NOTICE 'Migrated up to id: %, rows: %', last_id, rows_updated;
        PERFORM pg_sleep(0.05);  -- 50ms pause ระหว่าง batches
    END LOOP;
    
    RAISE NOTICE 'Backfill complete!';
END $$;

-- Phase 2b: ตรวจสอบว่า backfill สมบูรณ์
SELECT 
    COUNT(*) FILTER (WHERE phone IS NOT NULL AND phone_data IS NULL) AS missing_migration,
    COUNT(*) FILTER (WHERE phone_data IS NOT NULL) AS migrated,
    COUNT(*) AS total
FROM customers;
```

```typescript
// Application v1.2: Write Both, Read New (ทดสอบ column ใหม่)
async function getCustomerPhone(customerId: number): Promise<string> {
    const result = await db.query(
        'SELECT phone, phone_data FROM customers WHERE id = $1',
        [customerId]
    );
    
    const row = result.rows[0];
    
    // ลอง read จาก new column, fallback ไป old column
    if (row.phone_data?.primary) {
        return row.phone_data.primary;
    }
    return row.phone;  // fallback
}
```

### Phase 3: Contract (ลบ column เก่า)

```sql
-- หลังจาก v1.2 deploy แล้ว stable, ไม่ใช้ column เก่าแล้ว
-- Phase 3: ลบ column เก่า

-- ก่อนลบ: ตรวจสอบว่าไม่มี application อ่าน column เก่าแล้ว
-- ดู query logs สำหรับ references ถึง 'phone' column

-- ลบ default value ก่อน (optional)
ALTER TABLE customers ALTER COLUMN phone DROP DEFAULT;

-- ลบ column (fast operation)
ALTER TABLE customers DROP COLUMN phone;

-- ตรวจสอบ
\d customers
```

```typescript
// Application v2.0: ใช้แค่ phone_data
async function getCustomerPhones(customerId: number): Promise<any> {
    const result = await db.query(
        'SELECT phone_data FROM customers WHERE id = $1',
        [customerId]
    );
    return result.rows[0].phone_data;
}
```

---

## 4. CREATE INDEX CONCURRENTLY

### วิธีการทำงาน

```
Regular CREATE INDEX:
1. ถือ ACCESS EXCLUSIVE lock
2. Scan ทั้ง table
3. Build index
4. ปล่อย lock
ผล: ไม่มี reads/writes ระหว่าง build

CREATE INDEX CONCURRENTLY:
1. ถือ SHARE UPDATE EXCLUSIVE lock (ป้องกัน DDL เท่านั้น)
2. Scan 1: บันทึก snapshot, build initial index
3. Scan 2: Sync changes ที่เกิดขึ้นระหว่าง scan 1
4. Wait for all transactions ที่เริ่มก่อน scan 1 ให้เสร็จ
5. Scan 3: Final sync
6. Index พร้อมใช้งาน
ผล: reads/writes ยังทำงานได้ตลอด แต่ใช้เวลา 2-3x นานกว่า
```

### สร้าง Index แบบปลอดภัย

```sql
-- ❌ DANGEROUS: blocks writes for hours on large table
CREATE INDEX idx_orders_customer_id ON orders(customer_id);

-- ✅ SAFE: no blocking
CREATE INDEX CONCURRENTLY idx_orders_customer_id ON orders(customer_id);

-- ตรวจสอบ progress ระหว่าง build
SELECT 
    phase,
    blocks_done,
    blocks_total,
    ROUND(100.0 * blocks_done / NULLIF(blocks_total, 0), 1) AS pct_done,
    tuples_done,
    tuples_total
FROM pg_stat_progress_create_index
WHERE relid = 'orders'::regclass;

-- ตรวจสอบ index ที่กำลัง build
SELECT 
    indexname,
    indexdef,
    -- ถ้า indisvalid = false แสดงว่า build ยังไม่เสร็จ หรือ failed
    pg_index.indisvalid
FROM pg_indexes
JOIN pg_class ON pg_class.relname = pg_indexes.indexname
JOIN pg_index ON pg_index.indexrelid = pg_class.oid
WHERE tablename = 'orders';
```

### Handle Failed CONCURRENTLY Index

```sql
-- ถ้า CREATE INDEX CONCURRENTLY ถูก cancel หรือ fail
-- Index จะอยู่ในสถานะ INVALID

-- ดู invalid indexes
SELECT 
    schemaname,
    tablename,
    indexname,
    indexdef
FROM pg_indexes
JOIN pg_class ON pg_class.relname = pg_indexes.indexname
JOIN pg_index ON pg_index.indexrelid = pg_class.oid
WHERE NOT pg_index.indisvalid
    AND pg_indexes.schemaname = 'public';

-- ลบ invalid index
DROP INDEX CONCURRENTLY idx_orders_customer_id;

-- สร้างใหม่
CREATE INDEX CONCURRENTLY idx_orders_customer_id ON orders(customer_id);
```

---

## 5. ADD CONSTRAINT VALIDATE

### NOT VALID: Skip Existing Rows

```sql
-- ❌ DANGEROUS: validates ALL existing rows → long lock
ALTER TABLE orders ADD CONSTRAINT fk_orders_customer 
    FOREIGN KEY (customer_id) REFERENCES customers(id);

-- ✅ SAFE: Step 1 - Add constraint WITHOUT validating existing rows
-- ใช้ ACCESS EXCLUSIVE lock แค่ momentarily
ALTER TABLE orders ADD CONSTRAINT fk_orders_customer 
    FOREIGN KEY (customer_id) REFERENCES customers(id)
    NOT VALID;

-- NEW rows จะถูก validate ทันที
-- EXISTING rows จะ NOT be validated ยัง

-- ✅ SAFE: Step 2 - Validate existing rows (low-lock)
-- ใช้ SHARE UPDATE EXCLUSIVE lock (ไม่ block reads/writes)
ALTER TABLE orders VALIDATE CONSTRAINT fk_orders_customer;

-- ตรวจสอบ constraints ที่ยังไม่ validate
SELECT 
    conname,
    contype,
    convalidated,
    pg_get_constraintdef(oid) AS definition
FROM pg_constraint
WHERE conrelid = 'orders'::regclass
    AND NOT convalidated;
```

### Check Constraint

```sql
-- ADD CHECK CONSTRAINT
-- ❌ DANGEROUS
ALTER TABLE orders ADD CONSTRAINT chk_amount_positive 
    CHECK (amount > 0);

-- ✅ SAFE
-- Step 1: Add NOT VALID (fast)
ALTER TABLE orders ADD CONSTRAINT chk_amount_positive 
    CHECK (amount > 0) NOT VALID;

-- ตรวจสอบว่ามี rows ที่ violate ไหม
SELECT COUNT(*) FROM orders WHERE amount <= 0;

-- ถ้าไม่มี violations:
-- Step 2: Validate (low lock)
ALTER TABLE orders VALIDATE CONSTRAINT chk_amount_positive;
```

---

## 6. Renaming Columns/Tables

### Rename Column (Multi-step)

```sql
-- ❌ DANGEROUS: ทำ directly ทำให้ application break
ALTER TABLE orders RENAME COLUMN user_id TO customer_id;

-- ✅ SAFE: Multi-step rename

-- Step 1: Add new column
ALTER TABLE orders ADD COLUMN customer_id BIGINT;

-- Step 2: Create trigger ให้ sync ทั้ง 2 columns
CREATE OR REPLACE FUNCTION sync_order_customer_id()
RETURNS TRIGGER
LANGUAGE plpgsql AS $$
BEGIN
    IF TG_OP = 'INSERT' OR TG_OP = 'UPDATE' THEN
        -- Sync old → new
        IF NEW.user_id IS NOT NULL AND NEW.customer_id IS NULL THEN
            NEW.customer_id := NEW.user_id;
        END IF;
        -- Sync new → old  
        IF NEW.customer_id IS NOT NULL AND NEW.user_id IS NULL THEN
            NEW.user_id := NEW.customer_id;
        END IF;
    END IF;
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_sync_customer_id
    BEFORE INSERT OR UPDATE ON orders
    FOR EACH ROW EXECUTE FUNCTION sync_order_customer_id();

-- Step 3: Backfill new column
UPDATE orders SET customer_id = user_id WHERE customer_id IS NULL;

-- Step 4: Add index บน new column
CREATE INDEX CONCURRENTLY idx_orders_customer_id ON orders(customer_id);

-- Step 5: Deploy application ที่อ่าน/เขียนทั้ง 2 columns
-- อ่านจาก customer_id (ใหม่), เขียนทั้ง 2

-- Step 6: Deploy application version ที่ใช้แค่ customer_id
-- ไม่อ้างอิง user_id แล้ว

-- Step 7: Drop trigger และ old column
DROP TRIGGER trg_sync_customer_id ON orders;
DROP FUNCTION sync_order_customer_id();
ALTER TABLE orders DROP COLUMN user_id;
```

### Rename Table

```sql
-- ใช้ View เพื่อ backward compatibility ระหว่าง rename
-- Step 1: Rename table
ALTER TABLE users RENAME TO accounts;

-- Step 2: Create view ด้วยชื่อเก่า (backward compatible)
CREATE VIEW users AS SELECT * FROM accounts;

-- Step 3: Create INSTEAD OF triggers เพื่อรองรับ INSERT/UPDATE/DELETE ผ่าน view
CREATE OR REPLACE RULE users_insert AS
    ON INSERT TO users DO INSTEAD
    INSERT INTO accounts VALUES (NEW.*);

CREATE OR REPLACE RULE users_update AS
    ON UPDATE TO users DO INSTEAD
    UPDATE accounts SET ROW = NEW WHERE id = OLD.id;

CREATE OR REPLACE RULE users_delete AS
    ON DELETE TO users DO INSTEAD
    DELETE FROM accounts WHERE id = OLD.id;

-- Step 4: ค่อยๆ migrate application ไปใช้ accounts
-- Step 5: หลัง migration เสร็จ ลบ view เก่า
DROP VIEW users;
```

---

## 7. Large Data Backfills

### Batch Update Strategy

```python
# backfill_script.py
import psycopg2
import time
import logging
from datetime import datetime

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def run_batch_backfill(
    conn_string: str,
    table: str,
    set_clause: str,
    where_clause: str,
    batch_size: int = 10000,
    sleep_ms: int = 50,
    dry_run: bool = False
):
    """
    Run batch UPDATE เพื่อไม่ lock table นานเกินไป
    
    ตัวอย่าง:
    run_batch_backfill(
        conn_string="postgresql://localhost/mydb",
        table="orders",
        set_clause="status = 'active'",
        where_clause="status IS NULL",
        batch_size=10000
    )
    """
    
    conn = psycopg2.connect(conn_string)
    conn.autocommit = True
    cur = conn.cursor()
    
    # Count total rows to update
    cur.execute(f"SELECT COUNT(*) FROM {table} WHERE {where_clause}")
    total_rows = cur.fetchone()[0]
    logger.info(f"Total rows to backfill: {total_rows:,}")
    
    if dry_run:
        logger.info("DRY RUN - no changes made")
        return
    
    # Get ID range
    cur.execute(f"SELECT MIN(id), MAX(id) FROM {table} WHERE {where_clause}")
    min_id, max_id = cur.fetchone()
    
    if min_id is None:
        logger.info("No rows to update")
        return
    
    # Run batches
    current_id = min_id
    total_updated = 0
    start_time = datetime.now()
    
    while current_id <= max_id:
        cur.execute(f"""
            UPDATE {table}
            SET {set_clause}
            WHERE id BETWEEN %s AND %s
                AND ({where_clause})
        """, (current_id, current_id + batch_size - 1))
        
        rows_updated = cur.rowcount
        total_updated += rows_updated
        
        # Progress
        elapsed = (datetime.now() - start_time).seconds
        rate = total_updated / max(elapsed, 1)
        remaining = (total_rows - total_updated) / max(rate, 1)
        
        logger.info(
            f"Updated {total_updated:,}/{total_rows:,} rows "
            f"({100*total_updated/total_rows:.1f}%) "
            f"Rate: {rate:.0f} rows/sec "
            f"ETA: {remaining:.0f}s"
        )
        
        current_id += batch_size
        time.sleep(sleep_ms / 1000.0)
    
    logger.info(f"Backfill complete! Updated {total_updated:,} rows")
    cur.close()
    conn.close()

# ใช้งาน
if __name__ == "__main__":
    run_batch_backfill(
        conn_string="postgresql://postgres:password@localhost/production",
        table="orders",
        set_clause="notes = ''",
        where_clause="notes IS NULL",
        batch_size=5000,
        sleep_ms=20
    )
```

### Background Job สำหรับ Long-running Backfill

```python
# background_backfill.py
import asyncio
import asyncpg
import logging
from dataclasses import dataclass
from typing import Optional

@dataclass
class BackfillConfig:
    table: str
    set_clause: str
    condition: str
    batch_size: int = 10000
    sleep_ms: int = 100
    max_rows_per_run: Optional[int] = None  # None = ทำทั้งหมด

async def run_incremental_backfill(
    pool: asyncpg.Pool,
    config: BackfillConfig,
    checkpoint_table: str = "migration_checkpoints"
) -> int:
    """
    Incremental backfill ที่สามารถ resume ได้ถ้า process ตาย
    """
    
    async with pool.acquire() as conn:
        # สร้าง checkpoint table ถ้ายังไม่มี
        await conn.execute(f"""
            CREATE TABLE IF NOT EXISTS {checkpoint_table} (
                migration_name TEXT PRIMARY KEY,
                last_processed_id BIGINT,
                total_processed BIGINT DEFAULT 0,
                updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        """)
        
        migration_name = f"backfill_{config.table}_{config.set_clause[:20]}"
        
        # ดึง checkpoint
        checkpoint = await conn.fetchrow(
            f"SELECT last_processed_id, total_processed FROM {checkpoint_table} WHERE migration_name = $1",
            migration_name
        )
        
        start_id = checkpoint['last_processed_id'] if checkpoint else 0
        total_processed = checkpoint['total_processed'] if checkpoint else 0
        
        rows_this_run = 0
        current_id = start_id
        
        # Get max ID
        max_id = await conn.fetchval(
            f"SELECT MAX(id) FROM {config.table} WHERE {config.condition}"
        )
        
        if max_id is None:
            logging.info("No rows to backfill")
            return 0
        
        logging.info(f"Resuming from id: {current_id}, max: {max_id}")
        
        while current_id <= max_id:
            # Check if we should stop
            if config.max_rows_per_run and rows_this_run >= config.max_rows_per_run:
                logging.info(f"Reached max_rows_per_run: {config.max_rows_per_run}")
                break
            
            # Run batch
            updated = await conn.execute(f"""
                UPDATE {config.table}
                SET {config.set_clause}
                WHERE id > $1 AND id <= $2
                    AND ({config.condition})
            """, current_id, current_id + config.batch_size)
            
            rows_updated = int(updated.split()[-1])
            rows_this_run += rows_updated
            total_processed += rows_updated
            current_id += config.batch_size
            
            # Update checkpoint
            await conn.execute(f"""
                INSERT INTO {checkpoint_table} (migration_name, last_processed_id, total_processed)
                VALUES ($1, $2, $3)
                ON CONFLICT (migration_name) DO UPDATE SET
                    last_processed_id = EXCLUDED.last_processed_id,
                    total_processed = EXCLUDED.total_processed,
                    updated_at = CURRENT_TIMESTAMP
            """, migration_name, current_id, total_processed)
            
            logging.info(f"Processed id {current_id}: {rows_updated} rows (total: {total_processed})")
            
            await asyncio.sleep(config.sleep_ms / 1000.0)
        
        return rows_this_run

# ใช้งาน
async def main():
    pool = await asyncpg.create_pool("postgresql://localhost/production")
    
    config = BackfillConfig(
        table="orders",
        set_clause="customer_id = user_id",
        condition="customer_id IS NULL",
        batch_size=5000,
        sleep_ms=50,
        max_rows_per_run=100000  # ทำ 100k rows ต่อ run
    )
    
    rows = await run_incremental_backfill(pool, config)
    print(f"Processed {rows} rows this run")
    
    await pool.close()

asyncio.run(main())
```

---

## 8. pg_repack: Rebuild Table Without Locks

### ติดตั้ง pg_repack

```bash
# ติดตั้ง
sudo apt-get install -y postgresql-16-repack

# หรือ compile
git clone https://github.com/reorg/pg_repack.git
cd pg_repack
make
sudo make install
```

```sql
CREATE EXTENSION pg_repack;
```

### ใช้งาน pg_repack

```bash
# Repack table (เหมือน VACUUM FULL แต่ไม่ block)
pg_repack -h localhost -U postgres -d production -t orders

# Repack index เฉพาะ
pg_repack -h localhost -U postgres -d production -t orders --only-indexes

# Repack ทั้ง database
pg_repack -h localhost -U postgres -d production

# ตัวเลือก
pg_repack \
    --host=localhost \
    --port=5432 \
    --username=postgres \
    --dbname=production \
    --table=orders \
    --no-kill-backend \  # ไม่ kill long-running connections
    --wait-timeout=60    # รอ 60 วินาทีสำหรับ final lock
```

```sql
-- เมื่อไหร่ควรใช้ pg_repack:
-- 1. Table มี bloat สูง (dead tuples เยอะ)
-- 2. หลัง mass DELETE/UPDATE
-- 3. ต้องการ reorder data (CLUSTER equivalent)

-- ตรวจสอบ table bloat
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size,
    n_dead_tup,
    n_live_tup,
    ROUND(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 1) AS bloat_pct
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000000  -- มากกว่า 1M dead tuples
ORDER BY n_dead_tup DESC;
```

---

## 9. Full Migration Checklist สำหรับ Production

### Pre-Migration Checklist

```markdown
## Pre-Migration Checklist

### 1. Planning
- [ ] ระบุ migration type (ADD COLUMN, CREATE INDEX, etc.)
- [ ] ประเมิน lock duration (milliseconds vs. minutes vs. hours)
- [ ] เลือก safe migration pattern
- [ ] เขียน rollback plan
- [ ] ทดสอบบน staging environment ที่ใกล้เคียง production

### 2. Database Health Check
- [ ] VACUUM ANALYZE ก่อน migration
- [ ] ตรวจสอบ active connections
- [ ] ตรวจสอบ long-running transactions (> 1 minute)
- [ ] ตรวจสอบ replication lag
- [ ] ตรวจสอบ disk space (> 30% free)

### 3. Monitoring Setup
- [ ] Alert บน lock wait time
- [ ] Alert บน replication lag
- [ ] Alert บน error rate
- [ ] Dashboard: query latency, connections, wait events

### 4. Communication
- [ ] แจ้ง team migration window
- [ ] ตรวจสอบ on-call engineer พร้อม
- [ ] Runbook พร้อม
```

### Migration Script Template

```sql
-- migration_20240115_add_notes_column.sql
-- Author: engineer@company.com
-- Date: 2024-01-15
-- Description: Add notes column to orders table
-- Estimated Duration: < 1 second (online, no blocking)
-- Rollback: ALTER TABLE orders DROP COLUMN notes;
-- Risk: LOW

BEGIN;

-- Lock timeout: fail ถ้า lock ไม่ได้ภายใน 5 วินาที
SET lock_timeout = '5s';
-- Statement timeout: fail ถ้า query ใช้เวลา > 30 วินาที
SET statement_timeout = '30s';

-- Verify we're on the right database
DO $$
BEGIN
    IF current_database() != 'production' THEN
        RAISE EXCEPTION 'Wrong database: %', current_database();
    END IF;
END $$;

-- Check table exists
DO $$
BEGIN
    IF NOT EXISTS (
        SELECT 1 FROM information_schema.tables 
        WHERE table_name = 'orders' AND table_schema = 'public'
    ) THEN
        RAISE EXCEPTION 'Table orders does not exist';
    END IF;
END $$;

-- Check column doesn't already exist (idempotent)
DO $$
BEGIN
    IF NOT EXISTS (
        SELECT 1 FROM information_schema.columns 
        WHERE table_name = 'orders' 
            AND column_name = 'notes'
            AND table_schema = 'public'
    ) THEN
        ALTER TABLE orders ADD COLUMN notes TEXT;
        RAISE NOTICE 'Added notes column';
    ELSE
        RAISE NOTICE 'notes column already exists, skipping';
    END IF;
END $$;

-- Record migration
INSERT INTO schema_migrations (version, applied_at, description)
VALUES ('20240115001', CURRENT_TIMESTAMP, 'Add notes column to orders')
ON CONFLICT (version) DO NOTHING;

COMMIT;

-- Post-migration verification
SELECT 
    column_name,
    data_type,
    is_nullable
FROM information_schema.columns
WHERE table_name = 'orders' 
    AND column_name = 'notes';
```

### Migration Framework (TypeScript/Node.js)

```typescript
// src/migrations/migrator.ts
import { Pool } from 'pg';
import * as fs from 'fs';
import * as path from 'path';

export class Migrator {
    private pool: Pool;
    private migrationsDir: string;
    
    constructor(pool: Pool, migrationsDir: string = './migrations') {
        this.pool = pool;
        this.migrationsDir = migrationsDir;
    }
    
    async init() {
        await this.pool.query(`
            CREATE TABLE IF NOT EXISTS schema_migrations (
                version VARCHAR(20) PRIMARY KEY,
                applied_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                description TEXT
            )
        `);
    }
    
    async getAppliedMigrations(): Promise<Set<string>> {
        const result = await this.pool.query('SELECT version FROM schema_migrations');
        return new Set(result.rows.map(r => r.version));
    }
    
    async runPendingMigrations(dryRun: boolean = false): Promise<void> {
        await this.init();
        
        const applied = await this.getAppliedMigrations();
        const files = fs.readdirSync(this.migrationsDir)
            .filter(f => f.endsWith('.sql'))
            .sort();
        
        const pending = files.filter(f => {
            const version = f.split('_')[0];
            return !applied.has(version);
        });
        
        if (pending.length === 0) {
            console.log('No pending migrations');
            return;
        }
        
        console.log(`Found ${pending.length} pending migrations:`);
        pending.forEach(f => console.log(`  - ${f}`));
        
        if (dryRun) {
            console.log('\nDRY RUN - no changes applied');
            return;
        }
        
        for (const file of pending) {
            console.log(`\nApplying: ${file}`);
            
            const sql = fs.readFileSync(
                path.join(this.migrationsDir, file), 
                'utf8'
            );
            
            const client = await this.pool.connect();
            try {
                await client.query('BEGIN');
                await client.query(sql);
                await client.query('COMMIT');
                console.log(`  ✓ Applied successfully`);
            } catch (error) {
                await client.query('ROLLBACK');
                console.error(`  ✗ Failed:`, error);
                throw error;
            } finally {
                client.release();
            }
        }
        
        console.log('\nAll migrations applied successfully');
    }
}

// ใช้งาน
const pool = new Pool({ connectionString: process.env.DATABASE_URL });
const migrator = new Migrator(pool);
await migrator.runPendingMigrations();
```

---

## 10. Monitoring Migrations

### ดู Lock Waits ระหว่าง Migration

```sql
-- Script เพื่อ monitor ระหว่าง migration
-- รันใน window อื่น

-- ดู blocking locks
SELECT
    blocked.pid AS blocked_pid,
    blocked.query AS blocked_query,
    blocked.wait_event_type,
    blocked.wait_event,
    blocker.pid AS blocker_pid,
    blocker.query AS blocker_query,
    AGE(NOW(), blocked.query_start) AS blocked_for
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocker 
    ON blocker.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE blocked.wait_event_type = 'Lock'
ORDER BY blocked_for DESC;

-- Kill long-running transactions ที่ blocking migration
-- (ระวัง! ตรวจสอบก่อนเสมอ)
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'idle in transaction'
    AND query_start < NOW() - INTERVAL '10 minutes';

-- ดู migration progress (สำหรับ CREATE INDEX)
SELECT 
    phase,
    ROUND(100.0 * blocks_done / NULLIF(blocks_total, 0), 1) AS pct_done,
    blocks_done,
    blocks_total
FROM pg_stat_progress_create_index;

-- ดู cluster progress (pg_repack, CLUSTER)
SELECT 
    phase,
    ROUND(100.0 * heap_blks_scanned / NULLIF(heap_blks_total, 0), 1) AS pct_done
FROM pg_stat_progress_cluster;

-- ดู VACUUM progress
SELECT 
    phase,
    ROUND(100.0 * heap_blks_vacuumed / NULLIF(heap_blks_total, 0), 1) AS pct_done
FROM pg_stat_progress_vacuum;
```

---

## สรุป

Zero-downtime migration ต้องการ:

1. **เข้าใจ Lock Types** - แต่ละ ALTER TABLE ใช้ lock อะไร นานแค่ไหน
2. **Expand-Contract Pattern** - เปลี่ยน schema ในหลาย deploy phases
3. **CREATE INDEX CONCURRENTLY** - สร้าง index โดยไม่ block
4. **NOT VALID + VALIDATE** - เพิ่ม constraint แบบ 2 ขั้นตอน
5. **Batch Backfills** - UPDATE ทีละน้อย ไม่ lock นาน
6. **pg_repack** - Rebuild table โดยไม่ block
7. **Monitoring** - ดู lock waits ตลอดเวลา
8. **Rollback Plan** - ทุก migration ต้องมี rollback

**Golden Rule: ถ้าไม่แน่ใจ ทดสอบบน staging ก่อนเสมอ และวัด lock duration จริง**

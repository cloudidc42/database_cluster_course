# Part 06: Transaction และ ACID Properties

## สารบัญ

1. [Transaction คืออะไร](#1-transaction-คืออะไร)
2. [ACID Properties](#2-acid-properties)
3. [Isolation Levels](#3-isolation-levels)
4. [Concurrency Problems](#4-concurrency-problems)
5. [Locking Mechanisms](#5-locking-mechanisms)
6. [Deadlock](#6-deadlock)
7. [MVCC ใน PostgreSQL](#7-mvcc-ใน-postgresql)
8. [Savepoints](#8-savepoints)
9. [Transaction ใน Application Code](#9-transaction-ใน-application-code)
10. [Long-running Transactions](#10-long-running-transactions)
11. [Monitoring: pg_stat_activity และ pg_locks](#11-monitoring-pg_stat_activity-และ-pg_locks)
12. [Workshop: Bank Transfer System](#12-workshop-bank-transfer-system)

---

## 1. Transaction คืออะไร

Transaction คือชุดของ SQL operations ที่ถูกมองเป็นหน่วยเดียว (single unit of work) ซึ่งต้องทำให้สำเร็จทั้งหมดหรือล้มเหลวทั้งหมด ไม่มีสถานะกึ่งกลาง

### ตัวอย่างในชีวิตจริง

ลองนึกถึงการโอนเงินระหว่างบัญชี:
1. ตัดเงินจากบัญชี A: 1,000 บาท
2. เพิ่มเงินในบัญชี B: 1,000 บาท

หากขั้นตอนที่ 1 สำเร็จแต่ขั้นตอนที่ 2 ล้มเหลว เงินจะหายไปจากระบบ Transaction ป้องกันปัญหานี้โดยให้ทั้งสองขั้นตอนเป็นหน่วยเดียวกัน

### คำสั่งพื้นฐาน: BEGIN, COMMIT, ROLLBACK

```sql
-- เริ่มต้น Transaction
BEGIN;

-- หรือใช้ START TRANSACTION (เหมือนกัน)
START TRANSACTION;

-- ทำการ INSERT, UPDATE, DELETE ตามต้องการ
UPDATE accounts SET balance = balance - 1000 WHERE id = 1;
UPDATE accounts SET balance = balance + 1000 WHERE id = 2;

-- ยืนยันการเปลี่ยนแปลง (บันทึกถาวร)
COMMIT;

-- หรือยกเลิกการเปลี่ยนแปลงทั้งหมด
ROLLBACK;
```

### ตัวอย่าง Transaction สมบูรณ์

```sql
-- สร้างตารางทดสอบ
CREATE TABLE accounts (
    id SERIAL PRIMARY KEY,
    owner_name VARCHAR(100) NOT NULL,
    balance DECIMAL(15, 2) NOT NULL DEFAULT 0.00,
    CONSTRAINT balance_non_negative CHECK (balance >= 0)
);

-- ใส่ข้อมูลทดสอบ
INSERT INTO accounts (owner_name, balance) VALUES
    ('Alice', 5000.00),
    ('Bob', 3000.00);

-- Transaction การโอนเงิน
BEGIN;

-- ตรวจสอบยอดเงิน
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;

-- ถ้า balance >= 1000 ให้โอนได้
UPDATE accounts SET balance = balance - 1000.00 WHERE id = 1;
UPDATE accounts SET balance = balance + 1000.00 WHERE id = 2;

-- ตรวจสอบผลลัพธ์
SELECT id, owner_name, balance FROM accounts;

-- ยืนยัน
COMMIT;
```

### Auto-commit Mode

ใน PostgreSQL ค่า default คือ auto-commit mode หมายความว่าทุก statement ที่ไม่อยู่ใน explicit transaction จะถูก commit ทันที

```sql
-- นี่คือ auto-commit (commit ทันทีหลัง execute)
INSERT INTO accounts (owner_name, balance) VALUES ('Charlie', 2000.00);

-- ต้องการ Transaction ให้ใส่ BEGIN...COMMIT ล้อมรอบ
BEGIN;
INSERT INTO logs (action, timestamp) VALUES ('transfer', NOW());
UPDATE accounts SET balance = balance - 500 WHERE id = 1;
COMMIT;
```

---

## 2. ACID Properties

ACID เป็นตัวย่อของคุณสมบัติ 4 ประการที่ transaction ต้องมี เพื่อความน่าเชื่อถือของข้อมูล

### 2.1 Atomicity (ทั้งหมดหรือไม่มีเลย)

Atomicity หมายความว่า transaction ต้องทำให้สำเร็จทั้งหมด หรือไม่ก็ไม่ทำเลย ไม่มีสถานะกึ่งกลาง

```sql
-- ตัวอย่าง: Atomicity ในการโอนเงิน
BEGIN;

-- Step 1: ตัดเงิน
UPDATE accounts SET balance = balance - 2000.00 WHERE id = 1;

-- จำลองข้อผิดพลาด (เช่น constraint violation)
-- เพิ่มเงินมากเกินไปทำให้ id ไม่มีอยู่
UPDATE accounts SET balance = balance + 2000.00 WHERE id = 999; -- ไม่มี id นี้

-- แม้ว่า UPDATE แรกสำเร็จ แต่ถ้า ROLLBACK ทุกอย่างจะกลับคืน
ROLLBACK;

-- ตรวจสอบว่า balance ของ id=1 ยังเท่าเดิม
SELECT * FROM accounts WHERE id = 1;
```

```sql
-- ตัวอย่างที่แสดง Atomicity ชัดเจน
BEGIN;

INSERT INTO orders (customer_id, total_amount) VALUES (1, 500.00);

-- ถ้า INSERT สำเร็จ ดึง ID ที่เพิ่งสร้าง
-- แล้วใส่ order items
INSERT INTO order_items (order_id, product_id, quantity, price)
    VALUES (currval('orders_id_seq'), 101, 2, 250.00);

-- ถ้า step ใด step หนึ่งล้มเหลว ทั้งหมดจะถูก ROLLBACK
COMMIT;
```

### 2.2 Consistency (ข้อมูลถูกต้องเสมอ)

Consistency หมายความว่า transaction ต้องนำข้อมูลจาก state ที่ valid ไปสู่ state ที่ valid อีกอัน ไม่ละเมิด constraints, rules, หรือ triggers ใดๆ

```sql
-- สร้าง constraints เพื่อรับประกัน Consistency
CREATE TABLE accounts (
    id SERIAL PRIMARY KEY,
    owner_name VARCHAR(100) NOT NULL,
    balance DECIMAL(15, 2) NOT NULL DEFAULT 0.00,
    -- Constraint: ยอดเงินต้องไม่ติดลบ
    CONSTRAINT balance_non_negative CHECK (balance >= 0),
    -- Constraint: ชื่อเจ้าของต้องไม่ว่าง
    CONSTRAINT owner_name_not_empty CHECK (LENGTH(TRIM(owner_name)) > 0)
);

-- Transaction ที่ละเมิด Consistency จะถูก Rollback อัตโนมัติ
BEGIN;
UPDATE accounts SET balance = balance - 10000.00 WHERE id = 1;
-- ถ้า balance < 0 จะเกิด constraint violation
-- PostgreSQL จะ ROLLBACK transaction นี้
COMMIT;
```

```sql
-- Foreign Key Constraints รับประกัน Consistency ระหว่างตาราง
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER NOT NULL,
    total_amount DECIMAL(15, 2) NOT NULL,
    -- ต้องอ้างถึง customer ที่มีอยู่จริง
    FOREIGN KEY (customer_id) REFERENCES customers(id)
);

-- ใส่ order โดยไม่มี customer จะล้มเหลว
BEGIN;
INSERT INTO orders (customer_id, total_amount) VALUES (9999, 500.00);
-- ERROR: foreign key constraint violation
ROLLBACK;
```

### 2.3 Isolation (Transaction แยกจากกัน)

Isolation หมายความว่าแต่ละ transaction ไม่ควรเห็นผลลัพธ์กึ่งกลางของ transaction อื่นๆ ที่กำลังทำงานพร้อมกัน

```sql
-- Session 1 (Transaction A)
BEGIN;
UPDATE accounts SET balance = balance - 500 WHERE id = 1;
-- ยังไม่ได้ COMMIT

-- Session 2 (Transaction B) - ถ้า isolation level = READ COMMITTED
BEGIN;
SELECT balance FROM accounts WHERE id = 1;
-- จะเห็น balance เดิม (ก่อน Transaction A แก้ไข)
-- เพราะ Transaction A ยังไม่ได้ COMMIT
COMMIT;
```

### 2.4 Durability (ข้อมูลคงอยู่หลัง Commit)

Durability หมายความว่าเมื่อ transaction ถูก commit แล้ว การเปลี่ยนแปลงนั้นจะคงอยู่ถาวร แม้ระบบจะล่มหรือไฟดับ

PostgreSQL รับประกัน Durability ด้วย:
- **WAL (Write-Ahead Logging)**: บันทึก changes ลง log ก่อน apply จริง
- **fsync**: บังคับให้ flush data ไปยัง disk
- **Checkpointing**: เขียน dirty pages ลง disk เป็นระยะๆ

```sql
-- ตรวจสอบ WAL settings
SHOW wal_level;
SHOW synchronous_commit;
SHOW fsync;

-- synchronous_commit modes:
-- on (default): รอให้ WAL flush ลง disk ก่อน return
-- off: ไม่รอ (เร็วกว่าแต่ข้อมูลอาจหายถ้าระบบล่ม)
-- local: รอ local WAL เท่านั้น (สำหรับ replication)
-- remote_write: รอ standby รับ WAL
-- remote_apply: รอ standby apply แล้ว
```

---

## 3. Isolation Levels

PostgreSQL รองรับ Isolation Levels ตามมาตรฐาน SQL:

| Level | Dirty Read | Non-repeatable Read | Phantom Read |
|-------|-----------|--------------------|--------------| 
| READ UNCOMMITTED | Possible* | Possible | Possible |
| READ COMMITTED | Not Possible | Possible | Possible |
| REPEATABLE READ | Not Possible | Not Possible | Possible* |
| SERIALIZABLE | Not Possible | Not Possible | Not Possible |

*PostgreSQL ไม่มี Dirty Reads แม้ใน READ UNCOMMITTED และป้องกัน Phantom Reads ใน REPEATABLE READ ด้วย MVCC

### 3.1 READ UNCOMMITTED

```sql
-- PostgreSQL ปฏิบัติ READ UNCOMMITTED เหมือน READ COMMITTED
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
BEGIN;
SELECT * FROM accounts;
COMMIT;
```

### 3.2 READ COMMITTED (Default)

```sql
-- ค่า default ของ PostgreSQL
-- แต่ละ statement ในธุรกรรมเห็น snapshot ใหม่

-- Session 1
BEGIN;
UPDATE accounts SET balance = 9999.00 WHERE id = 1;
-- ยังไม่ COMMIT

-- Session 2 (READ COMMITTED)
BEGIN;
SELECT balance FROM accounts WHERE id = 1;
-- เห็น balance เดิม (5000.00) เพราะ Session 1 ยังไม่ COMMIT

-- Session 1 COMMIT
COMMIT;

-- Session 2 query ใหม่
SELECT balance FROM accounts WHERE id = 1;
-- ตอนนี้เห็น 9999.00 (หลัง Session 1 COMMIT)
COMMIT;
```

### 3.3 REPEATABLE READ

```sql
-- Session 1
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT balance FROM accounts WHERE id = 1;
-- เห็น 5000.00

-- Session 2 ระหว่างนั้น
BEGIN;
UPDATE accounts SET balance = 9999.00 WHERE id = 1;
COMMIT;

-- Session 1 query ซ้ำ
SELECT balance FROM accounts WHERE id = 1;
-- ยังเห็น 5000.00 เหมือนเดิม! (snapshot ณ เวลา BEGIN)
COMMIT;
```

### 3.4 SERIALIZABLE

```sql
-- Highest isolation level
-- Transaction ทั้งหมดดูเหมือนทำงานทีละอัน ไม่ซ้อนกัน

-- Session 1
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
SELECT SUM(balance) FROM accounts;
-- เห็น total = 8000.00

-- Session 2
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
UPDATE accounts SET balance = balance + 100 WHERE id = 1;
COMMIT;

-- Session 1 ทำงานต่อ
UPDATE accounts SET balance = balance - 100 WHERE id = 2;
COMMIT;
-- อาจเกิด serialization failure!
-- ERROR: could not serialize access due to read/write dependencies among transactions
```

### เปลี่ยน Isolation Level

```sql
-- สำหรับ transaction เดียว
BEGIN;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
-- ...statements...
COMMIT;

-- หรือ
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
-- ...statements...
COMMIT;

-- สำหรับ session ทั้งหมด
SET default_transaction_isolation = 'repeatable read';

-- ดู isolation level ปัจจุบัน
SHOW transaction_isolation;
```

---

## 4. Concurrency Problems

### 4.1 Dirty Read

การอ่านข้อมูลที่ transaction อื่นแก้ไขแต่ยังไม่ได้ COMMIT

```
Timeline:
T1: UPDATE balance = 9999  ← uncommitted
T2:                    READ balance = 9999  ← Dirty Read!
T1:         ROLLBACK (balance กลับเป็น 5000)
T2: ใช้ 9999 ต่อ ← ข้อมูลผิดพลาด!
```

```sql
-- PostgreSQL ป้องกัน Dirty Read แม้ใน READ UNCOMMITTED
-- ทดสอบ: เปิด 2 sessions

-- Session 1
BEGIN;
UPDATE accounts SET balance = 9999.00 WHERE id = 1;
-- ยังไม่ COMMIT

-- Session 2 (READ UNCOMMITTED - แต่ PostgreSQL จะไม่ให้อ่าน uncommitted)
BEGIN;
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SELECT balance FROM accounts WHERE id = 1;
-- ยังเห็น 5000.00 ไม่ใช่ 9999.00
COMMIT;
```

### 4.2 Non-repeatable Read

อ่านข้อมูลเดิมสองครั้ง แต่ได้ค่าต่างกัน เพราะ transaction อื่น COMMIT การเปลี่ยนแปลง

```
Timeline:
T1: READ balance = 5000
T2:              UPDATE balance = 3000, COMMIT
T1: READ balance = 3000  ← ค่าเปลี่ยน!
```

```sql
-- สาธิต Non-repeatable Read ใน READ COMMITTED

-- Session 1
BEGIN;  -- READ COMMITTED (default)
SELECT balance FROM accounts WHERE id = 1;
-- ผลลัพธ์: 5000.00

-- Session 2
BEGIN;
UPDATE accounts SET balance = 3000.00 WHERE id = 1;
COMMIT;

-- Session 1 อ่านซ้ำ
SELECT balance FROM accounts WHERE id = 1;
-- ผลลัพธ์: 3000.00 ← ค่าเปลี่ยนแปลง!
COMMIT;

-- แก้ไขด้วย REPEATABLE READ
-- Session 1
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT balance FROM accounts WHERE id = 1;
-- ผลลัพธ์: 5000.00

-- (Session 2 อัพเดทและ COMMIT)

-- Session 1 อ่านซ้ำ
SELECT balance FROM accounts WHERE id = 1;
-- ผลลัพธ์: 5000.00 ← ยังเหมือนเดิม! (REPEATABLE READ)
COMMIT;
```

### 4.3 Phantom Read

Query เดิมส่งคืน rows ต่างกัน เพราะ transaction อื่นเพิ่มหรือลบ rows

```
Timeline:
T1: SELECT WHERE balance > 1000 → rows: [A, B]
T2:              INSERT new row C (balance=2000), COMMIT
T1: SELECT WHERE balance > 1000 → rows: [A, B, C]  ← Phantom!
```

```sql
-- สาธิต Phantom Read ใน READ COMMITTED

-- Session 1
BEGIN;
SELECT COUNT(*) FROM accounts WHERE balance > 1000;
-- ผลลัพธ์: 2

-- Session 2
BEGIN;
INSERT INTO accounts (owner_name, balance) VALUES ('Dave', 2500.00);
COMMIT;

-- Session 1 query ซ้ำ
SELECT COUNT(*) FROM accounts WHERE balance > 1000;
-- ผลลัพธ์: 3 ← Phantom row!
COMMIT;
```

---

## 5. Locking Mechanisms

### 5.1 Row-level Locks

```sql
-- FOR UPDATE: Lock row เพื่อป้องกันการแก้ไขจาก transactions อื่น
BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
-- ตอนนี้ row นี้ถูก lock
-- transactions อื่นที่พยายาม UPDATE row นี้จะต้องรอ
UPDATE accounts SET balance = balance - 500 WHERE id = 1;
COMMIT;

-- FOR SHARE: Lock row แบบ shared (หลาย transactions อ่านได้ แต่ไม่มีใคร write)
BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR SHARE;
-- อ่านได้ แต่ transaction อื่นจะ lock FOR UPDATE ไม่ได้
COMMIT;

-- FOR NO KEY UPDATE: เหมือน FOR UPDATE แต่อนุญาต foreign key reference
BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR NO KEY UPDATE;
COMMIT;

-- FOR KEY SHARE: เหมือน FOR SHARE แต่สำหรับ foreign key operations
BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR KEY SHARE;
COMMIT;

-- SKIP LOCKED: ข้ามแถวที่ถูก lock แทนที่จะรอ
BEGIN;
SELECT * FROM jobs WHERE status = 'pending' LIMIT 10 FOR UPDATE SKIP LOCKED;
-- ดึง jobs ที่ยังไม่ถูก lock โดย workers อื่น
COMMIT;

-- NOWAIT: ถ้า lock ไม่ได้ให้ error ทันที แทนที่จะรอ
BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR UPDATE NOWAIT;
-- ERROR: could not obtain lock on row ถ้ามีคน lock อยู่
COMMIT;
```

### 5.2 Table-level Locks

```sql
-- Lock mode ต่างๆ (จาก least ไป most restrictive)
-- ACCESS SHARE: SELECT ปกติ
-- ROW SHARE: SELECT FOR UPDATE/SHARE
-- ROW EXCLUSIVE: INSERT, UPDATE, DELETE
-- SHARE UPDATE EXCLUSIVE: VACUUM, ANALYZE, CREATE INDEX CONCURRENTLY
-- SHARE: CREATE INDEX (non-concurrent)
-- SHARE ROW EXCLUSIVE: ไม่ค่อยใช้ตรงๆ
-- EXCLUSIVE: ไม่ค่อยใช้ตรงๆ
-- ACCESS EXCLUSIVE: ALTER TABLE, DROP TABLE, TRUNCATE

-- Lock table explicitly
BEGIN;
LOCK TABLE accounts IN EXCLUSIVE MODE;
-- ตอนนี้ไม่มีใคร read/write accounts ได้
UPDATE accounts SET balance = balance * 1.05; -- ดอกเบี้ย
COMMIT;

-- Lock หลาย tables
BEGIN;
LOCK TABLE accounts, transactions IN SHARE ROW EXCLUSIVE MODE;
-- ...
COMMIT;
```

### 5.3 Advisory Locks

```sql
-- Application-level locks ที่ PostgreSQL จัดการให้
-- ไม่ผูกกับ row หรือ table ใดๆ

-- Session-level advisory lock (ปล่อยเมื่อ session สิ้นสุด)
SELECT pg_advisory_lock(1234);        -- Lock with key 1234
SELECT pg_advisory_unlock(1234);      -- Unlock

-- Transaction-level advisory lock (ปล่อยเมื่อ transaction สิ้นสุด)
BEGIN;
SELECT pg_advisory_xact_lock(1234);
-- ...operations...
COMMIT; -- lock ถูกปล่อยอัตโนมัติ

-- Try lock (ไม่รอถ้า lock ไม่ได้)
SELECT pg_try_advisory_lock(1234);     -- คืน true ถ้า lock ได้
SELECT pg_try_advisory_xact_lock(1234);

-- Shared advisory lock
SELECT pg_advisory_lock_shared(1234);
SELECT pg_advisory_unlock_shared(1234);

-- ตัวอย่างใช้งานจริง: ป้องกัน duplicate job processing
BEGIN;
-- พยายาม lock job_id = 42
SELECT pg_try_advisory_xact_lock(42) AS got_lock;
-- ถ้าได้ lock ให้ process job นั้น
-- ถ้าไม่ได้ ให้ข้ามไป (worker อื่นกำลัง process อยู่)
UPDATE jobs SET status = 'processing', worker_id = pg_backend_pid()
WHERE id = 42;
-- ...process job...
UPDATE jobs SET status = 'done' WHERE id = 42;
COMMIT;
```

### 5.4 ดู Locks ที่มีอยู่

```sql
-- ดู locks ปัจจุบัน
SELECT 
    pl.pid,
    pl.mode,
    pl.granted,
    pl.relation::regclass AS table_name,
    pl.locktype,
    pa.query,
    pa.state,
    pa.wait_event_type,
    pa.wait_event
FROM pg_locks pl
JOIN pg_stat_activity pa ON pl.pid = pa.pid
WHERE pl.relation IS NOT NULL
ORDER BY pl.pid;
```

---

## 6. Deadlock

Deadlock เกิดขึ้นเมื่อ transactions สองตัว (หรือมากกว่า) ต่างรอให้อีกฝ่ายปล่อย lock ทำให้ไม่มีตัวใดดำเนินต่อได้

### 6.1 ตัวอย่าง Deadlock

```
Timeline:
T1: LOCK row A     ← T1 ได้ lock บน A
T2:     LOCK row B ← T2 ได้ lock บน B  
T1:         ต้องการ LOCK row B ← รอ T2
T2:     ต้องการ LOCK row A ← รอ T1
= DEADLOCK!
```

```sql
-- Session 1 (Transaction 1)
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- Lock id=1
-- รอ Session 2...
UPDATE accounts SET balance = balance + 100 WHERE id = 2;  -- ต้องการ lock id=2
COMMIT;

-- Session 2 (Transaction 2) - ทำพร้อมกัน
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 2;  -- Lock id=2
-- รอ Session 1...
UPDATE accounts SET balance = balance + 100 WHERE id = 1;  -- ต้องการ lock id=1
COMMIT;

-- PostgreSQL จะ detect deadlock และ ROLLBACK transaction หนึ่ง:
-- ERROR:  deadlock detected
-- DETAIL:  Process 12345 waits for ShareLock on transaction 67890;
--          blocked by process 11111.
```

### 6.2 วิธีป้องกัน Deadlock

#### วิธีที่ 1: Lock ตาม Order ที่สม่ำเสมอ (Consistent Ordering)

```sql
-- แทนที่จะ lock แบบไม่มีระเบียบ
-- ให้ lock ตาม primary key จากน้อยไปมากเสมอ

-- Function โอนเงินที่ป้องกัน Deadlock
CREATE OR REPLACE FUNCTION safe_transfer(
    from_account_id INTEGER,
    to_account_id INTEGER,
    amount DECIMAL
) RETURNS VOID AS $$
DECLARE
    first_id INTEGER;
    second_id INTEGER;
BEGIN
    -- เรียง ID จากน้อยไปมาก เพื่อ lock ในลำดับเดียวกันเสมอ
    IF from_account_id < to_account_id THEN
        first_id := from_account_id;
        second_id := to_account_id;
    ELSE
        first_id := to_account_id;
        second_id := from_account_id;
    END IF;
    
    -- Lock ตาม order
    PERFORM id FROM accounts WHERE id = first_id FOR UPDATE;
    PERFORM id FROM accounts WHERE id = second_id FOR UPDATE;
    
    -- ตรวจสอบและโอน
    IF (SELECT balance FROM accounts WHERE id = from_account_id) < amount THEN
        RAISE EXCEPTION 'Insufficient funds';
    END IF;
    
    UPDATE accounts SET balance = balance - amount WHERE id = from_account_id;
    UPDATE accounts SET balance = balance + amount WHERE id = to_account_id;
END;
$$ LANGUAGE plpgsql;

-- ใช้งาน
BEGIN;
SELECT safe_transfer(1, 2, 500.00);
COMMIT;
```

#### วิธีที่ 2: ใช้ NOWAIT

```sql
-- ถ้า lock ไม่ได้ทันที ให้ ROLLBACK แล้วลองใหม่
CREATE OR REPLACE FUNCTION try_transfer(
    from_id INTEGER,
    to_id INTEGER,
    amount DECIMAL
) RETURNS BOOLEAN AS $$
BEGIN
    BEGIN
        -- ลอง lock ทั้งสองพร้อมกัน
        PERFORM id FROM accounts WHERE id IN (from_id, to_id) 
        ORDER BY id FOR UPDATE NOWAIT;
        
        UPDATE accounts SET balance = balance - amount WHERE id = from_id;
        UPDATE accounts SET balance = balance + amount WHERE id = to_id;
        
        RETURN TRUE;
    EXCEPTION
        WHEN lock_not_available THEN
            RETURN FALSE;
    END;
END;
$$ LANGUAGE plpgsql;
```

#### วิธีที่ 3: ใช้ Advisory Locks

```sql
-- แทนที่ row locks ใช้ advisory locks ที่ควบคุมได้มากกว่า
CREATE OR REPLACE FUNCTION transfer_with_advisory(
    from_id INTEGER,
    to_id INTEGER,
    amount DECIMAL
) RETURNS VOID AS $$
DECLARE
    lock1 INTEGER;
    lock2 INTEGER;
BEGIN
    -- กำหนด lock keys ตาม order
    IF from_id < to_id THEN
        lock1 := from_id;
        lock2 := to_id;
    ELSE
        lock1 := to_id;
        lock2 := from_id;
    END IF;
    
    -- Lock ตาม order
    PERFORM pg_advisory_xact_lock(lock1);
    PERFORM pg_advisory_xact_lock(lock2);
    
    -- โอนเงิน
    UPDATE accounts SET balance = balance - amount WHERE id = from_id;
    UPDATE accounts SET balance = balance + amount WHERE id = to_id;
END;
$$ LANGUAGE plpgsql;
```

### 6.3 lock_timeout และ deadlock_detect_timeout

```sql
-- ตั้ง timeout สำหรับ lock wait
SET lock_timeout = '5s';       -- รอ lock ได้มากสุด 5 วินาที
SET deadlock_timeout = '1s';   -- ตรวจหา deadlock หลังจาก 1 วินาที (default 1s)

-- ใน postgresql.conf
-- deadlock_timeout = 1s       # default
-- lock_timeout = 0            # 0 = ไม่มี timeout (รอตลอด)
```

---

## 7. MVCC ใน PostgreSQL

MVCC (Multi-Version Concurrency Control) เป็นกลไกที่ PostgreSQL ใช้เพื่อให้ transactions หลายตัวทำงานพร้อมกันได้โดยไม่ต้อง block กัน

### 7.1 หลักการทำงาน

PostgreSQL เก็บ **หลาย versions** ของแต่ละ row:
- `xmin`: transaction ID ที่สร้าง row นี้
- `xmax`: transaction ID ที่ลบ/อัพเดท row นี้ (0 ถ้ายังไม่ถูกลบ)
- แต่ละ transaction เห็นเฉพาะ versions ที่ถูก commit ก่อนที่ transaction นั้นเริ่ม

```sql
-- ดู MVCC metadata ของ rows
SELECT 
    id,
    owner_name,
    balance,
    xmin,   -- transaction ที่สร้าง row นี้
    xmax,   -- transaction ที่ลบ/อัพเดท (0 = ยังอยู่)
    ctid    -- physical location (page, row) ใน heap
FROM accounts;

-- เมื่อ UPDATE PostgreSQL จะ:
-- 1. Mark row เดิมว่า deleted (set xmax)
-- 2. สร้าง row ใหม่ (row version ใหม่ที่มี xmin ใหม่)
-- ทำให้ transactions เก่าที่กำลังทำงานยังเห็น version เดิมได้

BEGIN;
UPDATE accounts SET balance = 9999.00 WHERE id = 1;
-- ดู row เดิมที่ถูก mark ว่าจะถูกลบ
SELECT xmin, xmax, id, balance FROM accounts WHERE id = 1;
ROLLBACK;
```

### 7.2 VACUUM และ Dead Tuples

```sql
-- Dead tuples คือ row versions เก่าที่ไม่มี transaction ใดต้องการแล้ว
-- VACUUM จะ reclaim space จาก dead tuples

-- ดู dead tuples
SELECT 
    schemaname,
    tablename,
    n_dead_tup,          -- จำนวน dead tuples
    n_live_tup,          -- จำนวน live tuples
    last_vacuum,         -- ครั้งล่าสุดที่ VACUUM
    last_autovacuum      -- ครั้งล่าสุดที่ auto vacuum
FROM pg_stat_user_tables
WHERE tablename = 'accounts';

-- รัน VACUUM ด้วยตนเอง
VACUUM accounts;            -- เร็ว ไม่ lock
VACUUM FULL accounts;       -- รัน full rewrite (lock table)
VACUUM ANALYZE accounts;    -- vacuum + update statistics
```

### 7.3 Transaction Snapshot

```sql
-- ดู current snapshot
SELECT txid_current();           -- current transaction ID
SELECT txid_current_snapshot();  -- current snapshot

-- Snapshot format: xmin:xmax:xip_list
-- xmin: transaction ID ต่ำสุดที่ยังทำงานอยู่
-- xmax: transaction ID ต่ำสุดถัดไปที่จะถูกสร้าง
-- xip_list: list ของ active transaction IDs
```

---

## 8. Savepoints

Savepoints ช่วยให้ ROLLBACK ไปยังจุดกลาง transaction แทนที่จะ ROLLBACK ทั้งหมด

```sql
BEGIN;

INSERT INTO accounts (owner_name, balance) VALUES ('Eve', 1000.00);

-- สร้าง savepoint
SAVEPOINT sp1;

INSERT INTO accounts (owner_name, balance) VALUES ('Frank', 2000.00);

-- สร้าง savepoint อีกจุด
SAVEPOINT sp2;

INSERT INTO accounts (owner_name, balance) VALUES ('Grace', 3000.00);

-- ตรวจสอบ
SELECT COUNT(*) FROM accounts;  -- 3 rows ใหม่

-- ROLLBACK ไปที่ sp2 (ยกเลิก Grace)
ROLLBACK TO SAVEPOINT sp2;
SELECT COUNT(*) FROM accounts;  -- 2 rows ใหม่ (Eve และ Frank)

-- ROLLBACK ไปที่ sp1 (ยกเลิก Frank ด้วย)
ROLLBACK TO SAVEPOINT sp1;
SELECT COUNT(*) FROM accounts;  -- 1 row ใหม่ (Eve เท่านั้น)

-- ลบ savepoint (ไม่จำเป็นแต่ดีถ้า savepoints เยอะ)
RELEASE SAVEPOINT sp1;

COMMIT;
-- Eve ถูก insert สำเร็จ, Frank และ Grace ไม่ถูก insert
```

### ตัวอย่างจริง: Partial Retry

```sql
BEGIN;

SAVEPOINT before_risky_operation;

BEGIN
    -- Risky operation
    INSERT INTO external_api_logs (endpoint, status) VALUES ('/payment', 'pending');
    -- ถ้าล้มเหลวให้ ROLLBACK ไป savepoint ไม่ใช่ทั้ง transaction
    
EXCEPTION WHEN OTHERS THEN
    ROLLBACK TO SAVEPOINT before_risky_operation;
    -- บันทึก error แทน
    INSERT INTO error_logs (error_msg, occurred_at) VALUES (SQLERRM, NOW());
END;

-- ส่วนที่เหลือของ transaction ยังทำงานต่อได้
UPDATE analytics SET failed_api_calls = failed_api_calls + 1;

COMMIT;
```

---

## 9. Transaction ใน Application Code

### 9.1 Node.js (node-postgres / pg)

```javascript
// ติดตั้ง: npm install pg

const { Pool } = require('pg');

const pool = new Pool({
  host: 'localhost',
  port: 5432,
  database: 'bankdb',
  user: 'postgres',
  password: 'password',
  max: 20,                  // pool size
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});

// Transaction แบบพื้นฐาน
async function transferMoney(fromId, toId, amount) {
  const client = await pool.connect();
  
  try {
    await client.query('BEGIN');
    
    // ตรวจสอบและ lock บัญชีต้นทาง
    const fromResult = await client.query(
      'SELECT balance FROM accounts WHERE id = $1 FOR UPDATE',
      [fromId]
    );
    
    if (fromResult.rows.length === 0) {
      throw new Error(`Account ${fromId} not found`);
    }
    
    const currentBalance = parseFloat(fromResult.rows[0].balance);
    
    if (currentBalance < amount) {
      throw new Error(`Insufficient funds: ${currentBalance} < ${amount}`);
    }
    
    // ตัดเงิน
    await client.query(
      'UPDATE accounts SET balance = balance - $1 WHERE id = $2',
      [amount, fromId]
    );
    
    // เพิ่มเงิน
    await client.query(
      'UPDATE accounts SET balance = balance + $1 WHERE id = $2',
      [amount, toId]
    );
    
    // บันทึก transaction history
    await client.query(
      `INSERT INTO transfer_history (from_account_id, to_account_id, amount, created_at)
       VALUES ($1, $2, $3, NOW())`,
      [fromId, toId, amount]
    );
    
    await client.query('COMMIT');
    
    console.log(`Transfer ${amount} from ${fromId} to ${toId} successful`);
    return { success: true };
    
  } catch (error) {
    await client.query('ROLLBACK');
    console.error('Transfer failed:', error.message);
    throw error;
  } finally {
    client.release(); // คืน connection กลับ pool
  }
}

// ใช้งาน
transferMoney(1, 2, 500.00)
  .then(result => console.log('Done:', result))
  .catch(err => console.error('Error:', err.message));
```

```javascript
// Transaction Helper Function

async function withTransaction(pool, fn) {
  const client = await pool.connect();
  
  try {
    await client.query('BEGIN');
    const result = await fn(client);
    await client.query('COMMIT');
    return result;
  } catch (error) {
    await client.query('ROLLBACK');
    throw error;
  } finally {
    client.release();
  }
}

// ใช้งาน
const result = await withTransaction(pool, async (client) => {
  const { rows } = await client.query(
    'SELECT * FROM accounts WHERE id = $1 FOR UPDATE',
    [accountId]
  );
  
  await client.query(
    'UPDATE accounts SET balance = balance - $1 WHERE id = $2',
    [100, accountId]
  );
  
  return rows[0];
});
```

```javascript
// Retry on Deadlock
async function transferWithRetry(fromId, toId, amount, maxRetries = 3) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await transferMoney(fromId, toId, amount);
    } catch (error) {
      // PostgreSQL error code 40P01 = deadlock detected
      if (error.code === '40P01' && attempt < maxRetries) {
        console.log(`Deadlock detected, retry ${attempt}/${maxRetries}`);
        // รอแบบ exponential backoff
        await new Promise(resolve => 
          setTimeout(resolve, Math.pow(2, attempt) * 100)
        );
        continue;
      }
      throw error;
    }
  }
}
```

### 9.2 Python (psycopg2)

```python
import psycopg2
import psycopg2.extras
from contextlib import contextmanager
from decimal import Decimal
import time

# Connection pool
from psycopg2 import pool as pgpool

connection_pool = pgpool.ThreadedConnectionPool(
    minconn=2,
    maxconn=20,
    host='localhost',
    port=5432,
    database='bankdb',
    user='postgres',
    password='password'
)

@contextmanager
def get_connection():
    """Context manager สำหรับ connection pool"""
    conn = connection_pool.getconn()
    try:
        yield conn
    finally:
        connection_pool.putconn(conn)

@contextmanager
def transaction(conn):
    """Context manager สำหรับ transaction"""
    try:
        yield conn
        conn.commit()
    except Exception:
        conn.rollback()
        raise

def transfer_money(from_id: int, to_id: int, amount: Decimal) -> dict:
    """โอนเงินระหว่างบัญชี"""
    with get_connection() as conn:
        with transaction(conn):
            with conn.cursor(cursor_factory=psycopg2.extras.RealDictCursor) as cur:
                # Lock บัญชีตาม order (ป้องกัน deadlock)
                min_id = min(from_id, to_id)
                max_id = max(from_id, to_id)
                
                cur.execute(
                    "SELECT id, balance FROM accounts WHERE id = ANY(%s) ORDER BY id FOR UPDATE",
                    ([min_id, max_id],)
                )
                accounts = {row['id']: row for row in cur.fetchall()}
                
                if from_id not in accounts:
                    raise ValueError(f"Account {from_id} not found")
                if to_id not in accounts:
                    raise ValueError(f"Account {to_id} not found")
                
                from_balance = Decimal(str(accounts[from_id]['balance']))
                
                if from_balance < amount:
                    raise ValueError(
                        f"Insufficient funds: {from_balance} < {amount}"
                    )
                
                # ตัดเงิน
                cur.execute(
                    "UPDATE accounts SET balance = balance - %s WHERE id = %s",
                    (amount, from_id)
                )
                
                # เพิ่มเงิน
                cur.execute(
                    "UPDATE accounts SET balance = balance + %s WHERE id = %s",
                    (amount, to_id)
                )
                
                # บันทึก history
                cur.execute(
                    """
                    INSERT INTO transfer_history 
                        (from_account_id, to_account_id, amount, created_at)
                    VALUES (%s, %s, %s, NOW())
                    RETURNING id
                    """,
                    (from_id, to_id, float(amount))
                )
                history_id = cur.fetchone()['id']
                
                return {
                    'success': True,
                    'history_id': history_id,
                    'transferred': float(amount)
                }

def transfer_with_retry(from_id: int, to_id: int, amount: Decimal, 
                        max_retries: int = 3) -> dict:
    """Transfer ที่ retry เมื่อเกิด deadlock"""
    for attempt in range(1, max_retries + 1):
        try:
            return transfer_money(from_id, to_id, amount)
        except psycopg2.errors.DeadlockDetected:
            if attempt < max_retries:
                wait_time = (2 ** attempt) * 0.1  # exponential backoff
                print(f"Deadlock, retry {attempt}/{max_retries} in {wait_time:.1f}s")
                time.sleep(wait_time)
            else:
                raise

# SQLAlchemy version
from sqlalchemy import create_engine, text
from sqlalchemy.orm import sessionmaker

engine = create_engine(
    'postgresql://postgres:password@localhost:5432/bankdb',
    pool_size=20,
    max_overflow=10,
    pool_pre_ping=True  # ตรวจสอบ connection ก่อนใช้
)

Session = sessionmaker(bind=engine)

def transfer_with_sqlalchemy(from_id: int, to_id: int, amount: Decimal):
    """Transaction ด้วย SQLAlchemy"""
    with Session() as session:
        with session.begin():
            # Lock rows
            from_acc = session.execute(
                text("SELECT * FROM accounts WHERE id = :id FOR UPDATE"),
                {"id": from_id}
            ).fetchone()
            
            if not from_acc:
                raise ValueError(f"Account {from_id} not found")
            
            if Decimal(str(from_acc.balance)) < amount:
                raise ValueError("Insufficient funds")
            
            session.execute(
                text("UPDATE accounts SET balance = balance - :amt WHERE id = :id"),
                {"amt": amount, "id": from_id}
            )
            
            session.execute(
                text("UPDATE accounts SET balance = balance + :amt WHERE id = :id"),
                {"amt": amount, "id": to_id}
            )
            # session.begin() context manager จะ commit หรือ rollback อัตโนมัติ
```

---

## 10. Long-running Transactions

### 10.1 ปัญหาที่เกิดจาก Long-running Transactions

Long-running transactions สร้างปัญหาหลายอย่าง:

1. **Bloat**: dead tuples ไม่สามารถ vacuum ได้
2. **Lock holding**: block operations อื่นๆ
3. **WAL accumulation**: WAL ไม่สามารถ recycle ได้
4. **Replication lag**: standby servers ต้อง apply WAL ที่สะสม

```sql
-- ดู long-running transactions
SELECT 
    pid,
    now() - pg_stat_activity.query_start AS duration,
    query,
    state,
    wait_event_type,
    wait_event,
    usename,
    application_name,
    client_addr
FROM pg_stat_activity
WHERE (now() - pg_stat_activity.query_start) > INTERVAL '5 minutes'
    AND state != 'idle'
ORDER BY duration DESC;

-- ดู transactions ที่ค้างนานที่สุด
SELECT 
    pid,
    usename,
    state,
    xact_start,
    now() - xact_start AS transaction_age,
    query_start,
    query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY transaction_age DESC
LIMIT 10;
```

### 10.2 ป้องกัน Long-running Transactions

```sql
-- ตั้ง statement timeout
SET statement_timeout = '30s';   -- ยกเลิก query ที่ใช้เวลา > 30s

-- ตั้ง transaction timeout (ใน session)
SET idle_in_transaction_session_timeout = '10min';  -- ยกเลิก idle transactions > 10 นาที

-- ใน postgresql.conf (global)
-- statement_timeout = 0                           -- ไม่มี timeout (default)
-- idle_in_transaction_session_timeout = 0         -- ไม่มี timeout (default)
-- lock_timeout = 0                                -- ไม่มี timeout (default)
```

```javascript
// Node.js: ตั้ง timeout สำหรับ transaction
async function transferWithTimeout(fromId, toId, amount) {
  const client = await pool.connect();
  
  try {
    await client.query('BEGIN');
    await client.query("SET LOCAL statement_timeout = '10s'");
    await client.query("SET LOCAL lock_timeout = '5s'");
    
    // ...transaction operations...
    
    await client.query('COMMIT');
  } catch (error) {
    await client.query('ROLLBACK');
    throw error;
  } finally {
    client.release();
  }
}
```

### 10.3 Terminate Long-running Transactions

```sql
-- Cancel query (ส่ง interrupt signal)
SELECT pg_cancel_backend(pid) FROM pg_stat_activity
WHERE (now() - query_start) > INTERVAL '10 minutes'
    AND state = 'active';

-- Terminate connection (force kill)
SELECT pg_terminate_backend(pid) FROM pg_stat_activity
WHERE (now() - xact_start) > INTERVAL '30 minutes'
    AND xact_start IS NOT NULL;

-- Terminate ทีละ pid
SELECT pg_terminate_backend(12345);
```

---

## 11. Monitoring: pg_stat_activity และ pg_locks

### 11.1 pg_stat_activity

```sql
-- ดู all active connections
SELECT 
    pid,
    usename,
    application_name,
    client_addr,
    client_port,
    backend_start,
    state,
    state_change,
    wait_event_type,
    wait_event,
    left(query, 100) AS query_snippet
FROM pg_stat_activity
ORDER BY backend_start;

-- ดู connections จัดกลุ่มตาม state
SELECT 
    state,
    COUNT(*) AS count,
    MAX(now() - state_change) AS max_duration
FROM pg_stat_activity
WHERE pid != pg_backend_pid()
GROUP BY state
ORDER BY count DESC;

-- ดู connections ที่ waiting สำหรับ lock
SELECT 
    pid,
    usename,
    wait_event_type,
    wait_event,
    state,
    query
FROM pg_stat_activity
WHERE wait_event_type = 'Lock'
ORDER BY state_change;

-- ดู queries ที่ช้า (> 1 วินาที)
SELECT 
    pid,
    usename,
    now() - query_start AS query_duration,
    state,
    wait_event,
    left(query, 200) AS query
FROM pg_stat_activity
WHERE state = 'active'
    AND query_start < NOW() - INTERVAL '1 second'
ORDER BY query_duration DESC;
```

### 11.2 pg_locks

```sql
-- ดู locks ทั้งหมด
SELECT 
    locktype,
    relation::regclass,
    mode,
    granted,
    pid,
    transactionid,
    classid,
    objid
FROM pg_locks
ORDER BY pid, locktype;

-- ดู lock conflicts (ใครรออะไร)
SELECT 
    blocked.pid AS blocked_pid,
    blocked_activity.usename AS blocked_user,
    blocking.pid AS blocking_pid,
    blocking_activity.usename AS blocking_user,
    blocked_activity.query AS blocked_query,
    blocking_activity.query AS blocking_query,
    now() - blocked_activity.query_start AS blocked_duration
FROM pg_catalog.pg_locks blocked
JOIN pg_catalog.pg_stat_activity blocked_activity 
    ON blocked.pid = blocked_activity.pid
JOIN pg_catalog.pg_locks blocking 
    ON blocked.locktype = blocking.locktype
    AND blocked.database IS NOT DISTINCT FROM blocking.database
    AND blocked.relation IS NOT DISTINCT FROM blocking.relation
    AND blocked.page IS NOT DISTINCT FROM blocking.page
    AND blocked.tuple IS NOT DISTINCT FROM blocking.tuple
    AND blocked.virtualxid IS NOT DISTINCT FROM blocking.virtualxid
    AND blocked.transactionid IS NOT DISTINCT FROM blocking.transactionid
    AND blocked.classid IS NOT DISTINCT FROM blocking.classid
    AND blocked.objid IS NOT DISTINCT FROM blocking.objid
    AND blocked.objsubid IS NOT DISTINCT FROM blocking.objsubid
    AND blocked.pid != blocking.pid
JOIN pg_catalog.pg_stat_activity blocking_activity 
    ON blocking.pid = blocking_activity.pid
WHERE NOT blocked.granted
ORDER BY blocked_duration DESC;

-- ดู lock dependency chain
WITH RECURSIVE lock_chain AS (
    -- Base case: processes ที่ blocked
    SELECT 
        blocked.pid AS pid,
        blocking.pid AS blocked_by,
        1 AS depth,
        ARRAY[blocked.pid] AS path
    FROM pg_locks blocked
    JOIN pg_locks blocking ON (
        blocked.locktype = blocking.locktype
        AND blocked.database IS NOT DISTINCT FROM blocking.database
        AND blocked.relation IS NOT DISTINCT FROM blocking.relation
        AND blocked.transactionid IS NOT DISTINCT FROM blocking.transactionid
        AND blocked.pid != blocking.pid
        AND NOT blocked.granted
        AND blocking.granted
    )
    
    UNION ALL
    
    -- Recursive: chain ต่อไป
    SELECT 
        lc.pid,
        bl.pid,
        lc.depth + 1,
        lc.path || bl.pid
    FROM lock_chain lc
    JOIN pg_locks bl ON lc.blocked_by = bl.pid
    WHERE bl.pid != ALL(lc.path)
      AND lc.depth < 10
)
SELECT DISTINCT
    pid,
    blocked_by,
    depth,
    path
FROM lock_chain
ORDER BY depth, pid;
```

### 11.3 Monitoring Dashboard View

```sql
-- สร้าง view สำหรับ monitoring
CREATE OR REPLACE VIEW active_locks_view AS
SELECT
    psa.pid,
    psa.usename,
    psa.application_name,
    psa.state,
    psa.wait_event_type,
    psa.wait_event,
    now() - psa.xact_start AS transaction_age,
    now() - psa.query_start AS query_age,
    pl.mode AS lock_mode,
    pl.granted AS lock_granted,
    pl.relation::regclass AS locked_table,
    LEFT(psa.query, 100) AS current_query
FROM pg_stat_activity psa
LEFT JOIN pg_locks pl ON psa.pid = pl.pid
    AND pl.relation IS NOT NULL
WHERE psa.pid != pg_backend_pid()
    AND psa.state != 'idle'
ORDER BY transaction_age DESC NULLS LAST;

-- ใช้งาน
SELECT * FROM active_locks_view;
```

---

## 12. Workshop: Bank Transfer System

### 12.1 Setup Database

```sql
-- สร้าง database
CREATE DATABASE bankdb;
\c bankdb

-- สร้าง tables
CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TABLE accounts (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER NOT NULL REFERENCES customers(id),
    account_number VARCHAR(20) UNIQUE NOT NULL,
    account_type VARCHAR(20) NOT NULL DEFAULT 'checking',
    balance DECIMAL(15, 2) NOT NULL DEFAULT 0.00,
    currency VARCHAR(3) NOT NULL DEFAULT 'THB',
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    CONSTRAINT balance_non_negative CHECK (balance >= 0),
    CONSTRAINT valid_account_type CHECK (account_type IN ('checking', 'savings', 'investment')),
    CONSTRAINT valid_status CHECK (status IN ('active', 'frozen', 'closed'))
);

CREATE TABLE transfers (
    id SERIAL PRIMARY KEY,
    from_account_id INTEGER NOT NULL REFERENCES accounts(id),
    to_account_id INTEGER NOT NULL REFERENCES accounts(id),
    amount DECIMAL(15, 2) NOT NULL,
    description TEXT,
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    completed_at TIMESTAMP WITH TIME ZONE,
    CONSTRAINT transfer_amount_positive CHECK (amount > 0),
    CONSTRAINT different_accounts CHECK (from_account_id != to_account_id)
);

CREATE TABLE balance_history (
    id SERIAL PRIMARY KEY,
    account_id INTEGER NOT NULL REFERENCES accounts(id),
    transfer_id INTEGER REFERENCES transfers(id),
    amount_change DECIMAL(15, 2) NOT NULL,  -- negative = debit, positive = credit
    balance_after DECIMAL(15, 2) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Indexes สำหรับ performance
CREATE INDEX idx_accounts_customer_id ON accounts(customer_id);
CREATE INDEX idx_accounts_account_number ON accounts(account_number);
CREATE INDEX idx_transfers_from_account ON transfers(from_account_id);
CREATE INDEX idx_transfers_to_account ON transfers(to_account_id);
CREATE INDEX idx_transfers_created_at ON transfers(created_at);
CREATE INDEX idx_balance_history_account_id ON balance_history(account_id);

-- Insert test data
INSERT INTO customers (name, email) VALUES
    ('Alice Johnson', 'alice@example.com'),
    ('Bob Smith', 'bob@example.com'),
    ('Charlie Brown', 'charlie@example.com');

INSERT INTO accounts (customer_id, account_number, balance) VALUES
    (1, 'ACC-001', 50000.00),
    (1, 'ACC-002', 20000.00),
    (2, 'ACC-003', 30000.00),
    (3, 'ACC-004', 15000.00);
```

### 12.2 Transfer Function (Thread-safe)

```sql
-- Stored procedure สำหรับ transfer ที่ thread-safe
CREATE OR REPLACE FUNCTION perform_transfer(
    p_from_account_number VARCHAR,
    p_to_account_number VARCHAR,
    p_amount DECIMAL,
    p_description TEXT DEFAULT NULL
) RETURNS TABLE(
    transfer_id INTEGER,
    from_balance_after DECIMAL,
    to_balance_after DECIMAL,
    status VARCHAR
) AS $$
DECLARE
    v_from_account accounts%ROWTYPE;
    v_to_account accounts%ROWTYPE;
    v_transfer_id INTEGER;
    v_from_balance_after DECIMAL;
    v_to_balance_after DECIMAL;
    v_lock_from INTEGER;
    v_lock_to INTEGER;
BEGIN
    -- ตรวจสอบ amount
    IF p_amount <= 0 THEN
        RAISE EXCEPTION 'Transfer amount must be positive: %', p_amount
            USING ERRCODE = 'check_violation';
    END IF;
    
    -- ดึงข้อมูล accounts และ lock ตาม account_number order (ป้องกัน deadlock)
    -- Lock ในลำดับเดียวกันเสมอ
    IF p_from_account_number < p_to_account_number THEN
        SELECT * INTO v_from_account FROM accounts 
            WHERE account_number = p_from_account_number FOR UPDATE;
        SELECT * INTO v_to_account FROM accounts 
            WHERE account_number = p_to_account_number FOR UPDATE;
    ELSE
        SELECT * INTO v_to_account FROM accounts 
            WHERE account_number = p_to_account_number FOR UPDATE;
        SELECT * INTO v_from_account FROM accounts 
            WHERE account_number = p_from_account_number FOR UPDATE;
    END IF;
    
    -- ตรวจสอบว่าบัญชีมีอยู่จริง
    IF v_from_account IS NULL THEN
        RAISE EXCEPTION 'Source account not found: %', p_from_account_number
            USING ERRCODE = 'no_data_found';
    END IF;
    
    IF v_to_account IS NULL THEN
        RAISE EXCEPTION 'Destination account not found: %', p_to_account_number
            USING ERRCODE = 'no_data_found';
    END IF;
    
    -- ตรวจสอบสถานะบัญชี
    IF v_from_account.status != 'active' THEN
        RAISE EXCEPTION 'Source account is not active: % (status: %)', 
            p_from_account_number, v_from_account.status
            USING ERRCODE = 'invalid_parameter_value';
    END IF;
    
    IF v_to_account.status != 'active' THEN
        RAISE EXCEPTION 'Destination account is not active: % (status: %)', 
            p_to_account_number, v_to_account.status
            USING ERRCODE = 'invalid_parameter_value';
    END IF;
    
    -- ตรวจสอบ currency ต้องตรงกัน
    IF v_from_account.currency != v_to_account.currency THEN
        RAISE EXCEPTION 'Currency mismatch: % vs %', 
            v_from_account.currency, v_to_account.currency
            USING ERRCODE = 'invalid_parameter_value';
    END IF;
    
    -- ตรวจสอบยอดเงินเพียงพอ
    IF v_from_account.balance < p_amount THEN
        RAISE EXCEPTION 'Insufficient funds: available=%, required=%', 
            v_from_account.balance, p_amount
            USING ERRCODE = 'insufficient_privilege';
    END IF;
    
    -- สร้าง transfer record
    INSERT INTO transfers (from_account_id, to_account_id, amount, description, status)
    VALUES (v_from_account.id, v_to_account.id, p_amount, p_description, 'processing')
    RETURNING id INTO v_transfer_id;
    
    -- อัพเดท balance
    UPDATE accounts 
    SET balance = balance - p_amount,
        updated_at = NOW()
    WHERE id = v_from_account.id
    RETURNING balance INTO v_from_balance_after;
    
    UPDATE accounts 
    SET balance = balance + p_amount,
        updated_at = NOW()
    WHERE id = v_to_account.id
    RETURNING balance INTO v_to_balance_after;
    
    -- บันทึก balance history
    INSERT INTO balance_history (account_id, transfer_id, amount_change, balance_after)
    VALUES 
        (v_from_account.id, v_transfer_id, -p_amount, v_from_balance_after),
        (v_to_account.id, v_transfer_id, p_amount, v_to_balance_after);
    
    -- อัพเดท transfer status
    UPDATE transfers 
    SET status = 'completed', completed_at = NOW()
    WHERE id = v_transfer_id;
    
    -- คืนผลลัพธ์
    RETURN QUERY SELECT 
        v_transfer_id,
        v_from_balance_after,
        v_to_balance_after,
        'completed'::VARCHAR;

END;
$$ LANGUAGE plpgsql;

-- ทดสอบ transfer
BEGIN;
SELECT * FROM perform_transfer('ACC-001', 'ACC-003', 5000.00, 'Payment for services');
-- ดูผลลัพธ์
SELECT account_number, balance FROM accounts WHERE account_number IN ('ACC-001', 'ACC-003');
COMMIT;
```

### 12.3 Node.js Application

```javascript
// bank-transfer-service.js
const { Pool } = require('pg');

const pool = new Pool({
  host: process.env.DB_HOST || 'localhost',
  port: process.env.DB_PORT || 5432,
  database: process.env.DB_NAME || 'bankdb',
  user: process.env.DB_USER || 'postgres',
  password: process.env.DB_PASSWORD || 'password',
  max: 20,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 5000,
});

class BankTransferService {
  
  async transfer(fromAccountNumber, toAccountNumber, amount, description = '') {
    const client = await pool.connect();
    
    try {
      await client.query('BEGIN');
      
      // ตั้ง timeouts ป้องกัน long-running transaction
      await client.query("SET LOCAL lock_timeout = '10s'");
      await client.query("SET LOCAL statement_timeout = '30s'");
      
      const result = await client.query(
        'SELECT * FROM perform_transfer($1, $2, $3, $4)',
        [fromAccountNumber, toAccountNumber, amount, description]
      );
      
      await client.query('COMMIT');
      
      const row = result.rows[0];
      return {
        success: true,
        transferId: row.transfer_id,
        fromBalanceAfter: row.from_balance_after,
        toBalanceAfter: row.to_balance_after,
        status: row.status
      };
      
    } catch (error) {
      await client.query('ROLLBACK');
      
      // แปลง PostgreSQL errors เป็น friendly messages
      if (error.code === '23514') {  // check_violation
        throw new Error('Invalid transfer amount');
      } else if (error.code === 'P0002') {  // no_data_found
        throw new Error(error.message);
      } else if (error.code === '28000') {  // insufficient_privilege  
        throw new Error('Insufficient funds');
      } else if (error.code === '40P01') {  // deadlock
        throw Object.assign(new Error('Transaction conflict, please retry'), { 
          retryable: true 
        });
      }
      
      throw error;
      
    } finally {
      client.release();
    }
  }
  
  async getAccountBalance(accountNumber) {
    const result = await pool.query(
      `SELECT a.account_number, a.balance, a.currency, a.status,
              c.name AS owner_name
       FROM accounts a
       JOIN customers c ON a.customer_id = c.id
       WHERE a.account_number = $1`,
      [accountNumber]
    );
    
    if (result.rows.length === 0) {
      throw new Error(`Account not found: ${accountNumber}`);
    }
    
    return result.rows[0];
  }
  
  async getTransferHistory(accountNumber, limit = 20) {
    const result = await pool.query(
      `SELECT 
          t.id,
          t.amount,
          t.description,
          t.status,
          t.created_at,
          fa.account_number AS from_account,
          ta.account_number AS to_account,
          bh.amount_change,
          bh.balance_after
       FROM transfers t
       JOIN accounts fa ON t.from_account_id = fa.id
       JOIN accounts ta ON t.to_account_id = ta.id
       JOIN balance_history bh ON bh.transfer_id = t.id
       JOIN accounts ba ON bh.account_id = ba.id
       WHERE (fa.account_number = $1 OR ta.account_number = $1)
           AND ba.account_number = $1
       ORDER BY t.created_at DESC
       LIMIT $2`,
      [accountNumber, limit]
    );
    
    return result.rows;
  }
  
  async bulkTransfer(transfers) {
    /**
     * โอนเงินหลายรายการใน transaction เดียว
     * @param {Array} transfers - [{from, to, amount, desc}]
     */
    const client = await pool.connect();
    
    try {
      await client.query('BEGIN');
      await client.query("SET LOCAL lock_timeout = '30s'");
      
      const results = [];
      
      for (const transfer of transfers) {
        const result = await client.query(
          'SELECT * FROM perform_transfer($1, $2, $3, $4)',
          [transfer.from, transfer.to, transfer.amount, transfer.description]
        );
        results.push(result.rows[0]);
      }
      
      await client.query('COMMIT');
      
      return {
        success: true,
        transferCount: results.length,
        transfers: results
      };
      
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }
}

// ทดสอบ
const service = new BankTransferService();

async function runTests() {
  console.log('=== Bank Transfer System Test ===\n');
  
  // Test 1: โอนเงินปกติ
  try {
    console.log('Test 1: Normal transfer');
    const result = await service.transfer('ACC-001', 'ACC-003', 1000.00, 'Test payment');
    console.log('Success:', result);
  } catch (error) {
    console.error('Failed:', error.message);
  }
  
  // Test 2: โอนเงินมากกว่ายอดที่มี
  try {
    console.log('\nTest 2: Transfer exceeding balance');
    await service.transfer('ACC-004', 'ACC-001', 999999.00);
    console.log('Should have failed!');
  } catch (error) {
    console.log('Expected error:', error.message);
  }
  
  // Test 3: ดู transfer history
  console.log('\nTest 3: Transfer history for ACC-001');
  const history = await service.getTransferHistory('ACC-001', 5);
  console.log('History:', JSON.stringify(history, null, 2));
  
  // Test 4: Concurrent transfers (stress test)
  console.log('\nTest 4: Concurrent transfers');
  const concurrentTransfers = Array.from({length: 10}, (_, i) => 
    service.transfer('ACC-001', 'ACC-003', 100.00, `Concurrent test ${i + 1}`)
      .then(r => ({ success: true, ...r }))
      .catch(e => ({ success: false, error: e.message }))
  );
  
  const results = await Promise.all(concurrentTransfers);
  const successful = results.filter(r => r.success).length;
  const failed = results.filter(r => !r.success).length;
  console.log(`Concurrent results: ${successful} succeeded, ${failed} failed`);
}

runTests().catch(console.error);

module.exports = BankTransferService;
```

### 12.4 Testing & Verification

```sql
-- ตรวจสอบว่า total balance คงที่ (conservation of money)
SELECT 
    SUM(balance) AS total_balance,
    COUNT(*) AS account_count
FROM accounts
WHERE status = 'active';

-- ตรวจสอบ balance history ตรงกัน
SELECT 
    a.account_number,
    a.balance AS current_balance,
    (SELECT bh.balance_after 
     FROM balance_history bh 
     WHERE bh.account_id = a.id 
     ORDER BY bh.created_at DESC 
     LIMIT 1) AS last_recorded_balance
FROM accounts a
WHERE (SELECT bh.balance_after 
       FROM balance_history bh 
       WHERE bh.account_id = a.id 
       ORDER BY bh.created_at DESC 
       LIMIT 1) != a.balance
   OR (SELECT bh.balance_after 
       FROM balance_history bh 
       WHERE bh.account_id = a.id 
       ORDER BY bh.created_at DESC 
       LIMIT 1) IS NULL;

-- ดู transfers ที่สำเร็จในวันนี้
SELECT 
    COUNT(*) AS total_transfers,
    SUM(amount) AS total_amount,
    AVG(amount) AS avg_amount
FROM transfers
WHERE DATE(created_at) = CURRENT_DATE
    AND status = 'completed';

-- ดู accounts ที่มี transaction เยอะที่สุด
SELECT 
    a.account_number,
    c.name,
    COUNT(DISTINCT t.id) AS transfer_count,
    SUM(CASE WHEN t.from_account_id = a.id THEN t.amount ELSE 0 END) AS total_sent,
    SUM(CASE WHEN t.to_account_id = a.id THEN t.amount ELSE 0 END) AS total_received
FROM accounts a
JOIN customers c ON a.customer_id = c.id
LEFT JOIN transfers t ON (t.from_account_id = a.id OR t.to_account_id = a.id)
    AND t.status = 'completed'
GROUP BY a.id, a.account_number, c.name
ORDER BY transfer_count DESC;
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Transaction**: BEGIN, COMMIT, ROLLBACK และความสำคัญของ atomic operations
2. **ACID Properties**: Atomicity, Consistency, Isolation, Durability
3. **Isolation Levels**: 4 ระดับตั้งแต่ READ UNCOMMITTED ถึง SERIALIZABLE
4. **Concurrency Problems**: Dirty Read, Non-repeatable Read, Phantom Read
5. **Locking**: Row-level locks, Table-level locks, Advisory locks
6. **Deadlock**: สาเหตุ ตัวอย่าง และวิธีป้องกัน
7. **MVCC**: กลไก multi-version concurrency ของ PostgreSQL
8. **Savepoints**: ควบคุม transaction อย่างละเอียด
9. **Application Code**: Node.js และ Python transaction patterns
10. **Monitoring**: pg_stat_activity และ pg_locks สำหรับ debug

บทถัดไปจะเรียนรู้เรื่อง Database Schema Design ซึ่งจะช่วยให้สามารถออกแบบ database ที่มีประสิทธิภาพและขยายได้ง่าย

---

*[Part 06 จบ — ไปต่อ Part 07: Database Schema Design]*

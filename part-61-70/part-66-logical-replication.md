# Part 66: Logical Replication ใน PostgreSQL

## บทนำ

Logical Replication เป็นหนึ่งในฟีเจอร์ที่ทรงพลังที่สุดของ PostgreSQL ซึ่งถูกเพิ่มเข้ามาอย่างเป็นทางการในเวอร์ชัน 10 และได้รับการปรับปรุงอย่างต่อเนื่องในเวอร์ชันถัดมา การเข้าใจ Logical Replication อย่างลึกซึ้งเป็นสิ่งจำเป็นสำหรับ Database Administrator และ Developer ที่ต้องการจัดการระบบฐานข้อมูลระดับ Enterprise

---

## 1. Logical Replication vs Physical (Streaming) Replication

### 1.1 Physical Replication (Streaming Replication)

Physical Replication หรือที่เรียกว่า Streaming Replication ทำงานโดยการส่ง **WAL (Write-Ahead Log) records** ทุกตัวไปยัง Standby server แบบ byte-for-byte ซึ่งหมายความว่า:

```
Primary Server                    Standby Server
┌─────────────────────┐          ┌─────────────────────┐
│  WAL Writer         │          │  WAL Receiver       │
│  ┌───────────────┐  │  WAL     │  ┌───────────────┐  │
│  │ Block changes │  │ ──────►  │  │ Apply blocks  │  │
│  └───────────────┘  │  stream  │  └───────────────┘  │
│                     │          │                     │
│  Data files:        │          │  Data files:        │
│  - Same exact       │          │  - Exact copy       │
│    binary format    │          │    of primary       │
└─────────────────────┘          └─────────────────────┘
```

**ข้อจำกัดของ Physical Replication:**
- ต้องใช้ PostgreSQL version เดียวกัน (อย่างน้อย major version)
- ต้องใช้ hardware architecture เดียวกัน (x86, ARM)
- Copy ทุกอย่างรวมถึง database ที่ไม่ต้องการ
- ไม่สามารถ replicate ไปยัง OS ที่ต่างกันได้
- Standby เป็น read-only ทั้งหมด ไม่สามารถ write ได้เลย

### 1.2 Logical Replication

Logical Replication ทำงานในระดับ **logical data changes** ไม่ใช่ระดับ block ซึ่งหมายความว่ามันส่งการเปลี่ยนแปลงในรูปแบบของ SQL operations (INSERT, UPDATE, DELETE) แทน:

```
Publisher (Source)                Subscriber (Target)
┌─────────────────────┐          ┌─────────────────────┐
│  WAL Decoder        │          │  Apply Worker       │
│  ┌───────────────┐  │  Logical │  ┌───────────────┐  │
│  │ INSERT row 1  │  │ ──────►  │  │ INSERT row 1  │  │
│  │ UPDATE row 2  │  │ changes  │  │ UPDATE row 2  │  │
│  │ DELETE row 3  │  │          │  │ DELETE row 3  │  │
│  └───────────────┘  │          │  └───────────────┘  │
└─────────────────────┘          └─────────────────────┘
```

**ความสามารถของ Logical Replication:**
- Replicate ระหว่าง PostgreSQL versions ที่ต่างกัน (เช่น PG14 → PG16)
- เลือก replicate เฉพาะบาง tables ที่ต้องการ
- Target database ยังสามารถ write ได้ (read-write replica)
- Cross-platform: Linux → Windows, x86 → ARM
- Replicate ไปหลาย subscribers ได้พร้อมกัน
- Bidirectional replication (ด้วย BDR extension)

### 1.3 เปรียบเทียบแบบตาราง

| Feature | Physical Replication | Logical Replication |
|---------|---------------------|---------------------|
| ระดับการ replicate | Block level | Row/logical level |
| PostgreSQL version | ต้องเหมือนกัน | ต่างกันได้ |
| Platform | ต้องเหมือนกัน | ต่างกันได้ |
| เลือก tables | ไม่ได้ | ได้ |
| Write บน replica | ไม่ได้ | ได้ |
| Initial setup | ง่ายกว่า | ซับซ้อนกว่า |
| Performance overhead | น้อยกว่า | มากกว่าเล็กน้อย |
| DDL replication | อัตโนมัติ | ต้องทำเอง |
| Use case หลัก | HA, DR | Migration, selective replication |

---

## 2. Use Cases ของ Logical Replication

### 2.1 Zero-Downtime Major Version Upgrade

นี่คือ use case ที่สำคัญที่สุด สำหรับระบบที่ไม่สามารถ downtime ได้ เช่น production system:

```
ขั้นตอน Zero-Downtime Upgrade: PG14 → PG16
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

เวลา T1: ติดตั้ง PostgreSQL 16 และ setup เบื้องต้น
         [PG14 Primary] ──── Logical Replication ────► [PG16 New Server]
         แอปพลิเคชันยังอ่าน-เขียนที่ PG14

เวลา T2: รอให้ PG16 ตาม PG14 ทัน (lag ≈ 0)

เวลา T3: Switch over (downtime เพียง 1-5 วินาที)
         - หยุด write ที่ PG14 (connection drain)
         - รอ PG16 apply หมด
         - เปลี่ยน connection string ใน Load Balancer
         [PG16 New Server] ← แอปพลิเคชันทั้งหมด

เวลา T4: PG14 เป็น legacy system, ปิดได้
```

### 2.2 Selective Table Replication

เหมาะสำหรับการส่งเฉพาะข้อมูลที่จำเป็น:

```sql
-- ส่งเฉพาะ tables ที่ต้องการไปยัง reporting database
CREATE PUBLICATION sales_reporting FOR TABLE 
    orders, 
    order_items, 
    customers, 
    products;

-- ไม่ส่ง: user_passwords, internal_logs, audit_trail
```

### 2.3 Cross-Platform Replication

```
Linux x86_64 (Production)          Windows Server (Analytics)
PostgreSQL 15                  →    PostgreSQL 15
                                    (สำหรับ BI tools ที่ต้องการ Windows)
```

### 2.4 Data Migration

ใช้ในการย้ายข้อมูลจากระบบเก่าไปใหม่โดยไม่ต้องหยุด:

```
Legacy System (PG 9.6)    →    New System (PG 16)
- ส่งข้อมูลไปเรื่อยๆ
- Switch traffic เมื่อพร้อม
- Rollback ง่ายถ้ามีปัญหา
```

### 2.5 Read Scaling to Different PostgreSQL Version

```
Write: PG16 Primary
  ├── Read Replica 1: PG16 (Physical Streaming)
  ├── Read Replica 2: PG16 (Physical Streaming)  
  └── Analytics DB: PG15 (Logical Replication) ← สำหรับ heavy analytics
```

---

## 3. Setup Logical Replication

### 3.1 ข้อกำหนดเบื้องต้น

**บน Publisher (Source):**
- PostgreSQL 10+
- `wal_level = logical` ใน postgresql.conf
- Network reachable จาก Subscriber

**บน Subscriber (Target):**
- Schema และ tables ต้องมีอยู่แล้ว (ต้อง create เอง)
- User มีสิทธิ์ REPLICATION หรือ SUPERUSER

### 3.2 การตั้งค่า Publisher

```bash
# postgresql.conf บน Publisher
wal_level = logical              # จำเป็น! (default คือ replica)
max_replication_slots = 10       # สำหรับ slot ของ subscribers แต่ละตัว
max_wal_senders = 10             # จำนวน connections สูงสุดสำหรับ replication
```

```bash
# pg_hba.conf บน Publisher - อนุญาต replication connections
# TYPE    DATABASE    USER           ADDRESS         METHOD
host      replication repuser        10.0.0.2/32     md5
# หรืออนุญาต logical replication ผ่าน normal connection:
host      mydb        repuser        10.0.0.2/32     md5
```

```sql
-- สร้าง replication user บน Publisher
CREATE USER repuser REPLICATION LOGIN PASSWORD 'secret123';

-- ให้ permission อ่านข้อมูลจาก tables ที่จะ replicate
GRANT SELECT ON ALL TABLES IN SCHEMA public TO repuser;
-- หรือให้ทีละ table:
GRANT SELECT ON TABLE orders, customers, products TO repuser;
```

### 3.3 Restart PostgreSQL หลังเปลี่ยน wal_level

```bash
# ตรวจสอบ wal_level ปัจจุบัน
SHOW wal_level;
-- ถ้าไม่ใช่ 'logical' ต้องรีสตาร์ท

# Restart PostgreSQL
sudo systemctl restart postgresql-16

# ตรวจสอบอีกครั้ง
psql -c "SHOW wal_level;"
-- logical
```

### 3.4 สร้าง Publication บน Publisher

```sql
-- ════════════════════════════════════════════════
-- Publication: กำหนดว่าจะ publish อะไร
-- ════════════════════════════════════════════════

-- 1. Publish เฉพาะบาง tables
CREATE PUBLICATION my_pub FOR TABLE orders, customers;

-- 2. Publish ทุก tables ใน database
CREATE PUBLICATION all_tables_pub FOR ALL TABLES;

-- 3. Publish เฉพาะ schema (PostgreSQL 15+)
CREATE PUBLICATION schema_pub FOR TABLES IN SCHEMA public, app_schema;

-- 4. Publish พร้อมระบุ operations
CREATE PUBLICATION insert_only_pub 
    FOR TABLE logs 
    WITH (publish = 'insert');  -- replicate เฉพาะ INSERT

CREATE PUBLICATION dml_pub 
    FOR TABLE orders 
    WITH (publish = 'insert, update, delete');

-- 5. Publish ด้วย row filter (PostgreSQL 15+)
CREATE PUBLICATION active_users_pub 
    FOR TABLE users 
    WHERE (status = 'active' AND region = 'asia');

-- 6. Publish เฉพาะบาง columns (PostgreSQL 16+)
CREATE PUBLICATION partial_pub 
    FOR TABLE customers (id, name, email, created_at);
    -- ไม่ส่ง: password_hash, internal_notes
```

### 3.5 สร้าง Subscription บน Subscriber

```sql
-- ════════════════════════════════════════════════
-- ก่อนสร้าง Subscription: ต้องสร้าง table ให้ตรงกัน
-- ════════════════════════════════════════════════

-- บน Subscriber database ต้องมี tables แล้ว
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    customer_id BIGINT NOT NULL,
    total_amount DECIMAL(10,2),
    status VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE customers (
    id BIGINT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ════════════════════════════════════════════════
-- สร้าง Subscription
-- ════════════════════════════════════════════════

-- Basic subscription
CREATE SUBSCRIPTION my_sub
    CONNECTION 'host=192.168.1.10 port=5432 dbname=mydb user=repuser password=secret123'
    PUBLICATION my_pub;

-- Subscription พร้อม options ทั้งหมด
CREATE SUBSCRIPTION my_sub
    CONNECTION 'host=192.168.1.10 port=5432 dbname=mydb user=repuser password=secret123 sslmode=require'
    PUBLICATION my_pub
    WITH (
        enabled = true,          -- เริ่ม replication ทันที (default: true)
        copy_data = true,        -- copy ข้อมูลปัจจุบัน (default: true)
        create_slot = true,      -- สร้าง replication slot บน publisher (default: true)
        slot_name = 'my_sub_slot', -- ชื่อ slot (default: ชื่อ subscription)
        synchronous_commit = 'off' -- performance: ไม่ต้องรอ subscriber commit
    );
```

---

## 4. Publication Types อย่างละเอียด

### 4.1 FOR TABLE: Specific Tables

```sql
-- ════════════════════════════════════════════════
-- FOR TABLE: เลือก tables ที่ต้องการ
-- ════════════════════════════════════════════════

-- Single table
CREATE PUBLICATION orders_pub FOR TABLE orders;

-- Multiple tables
CREATE PUBLICATION ecommerce_pub FOR TABLE 
    orders, 
    order_items, 
    products, 
    categories;

-- Tables จาก schemas ต่างกัน
CREATE PUBLICATION multi_schema_pub FOR TABLE 
    public.orders,
    app.customers,
    billing.invoices;

-- เพิ่ม/ลบ tables ใน publication ภายหลัง
ALTER PUBLICATION ecommerce_pub ADD TABLE reviews;
ALTER PUBLICATION ecommerce_pub DROP TABLE categories;
ALTER PUBLICATION ecommerce_pub SET TABLE orders, order_items, products;

-- ดู tables ใน publication
SELECT * FROM pg_publication_tables WHERE pubname = 'ecommerce_pub';
```

### 4.2 FOR ALL TABLES

```sql
-- ════════════════════════════════════════════════
-- FOR ALL TABLES: publish ทุก tables (รวม future tables ด้วย!)
-- ════════════════════════════════════════════════

CREATE PUBLICATION all_pub FOR ALL TABLES;

-- สำคัญ: ALL TABLES หมายความว่าทุก table ที่สร้างใหม่ในอนาคต
-- จะถูก replicate โดยอัตโนมัติ ซึ่งอาจไม่ต้องการ

-- ดู publication
SELECT * FROM pg_publication WHERE pubname = 'all_pub';
```

### 4.3 FOR TABLES IN SCHEMA (PostgreSQL 15+)

```sql
-- ════════════════════════════════════════════════
-- FOR TABLES IN SCHEMA: publish ทุก tables ใน schema
-- ════════════════════════════════════════════════

-- Schema เดียว
CREATE PUBLICATION app_pub FOR TABLES IN SCHEMA app;

-- หลาย schemas
CREATE PUBLICATION multi_pub FOR TABLES IN SCHEMA app, billing, reporting;

-- ผสม: บาง schema + บาง table
CREATE PUBLICATION mixed_pub FOR 
    TABLES IN SCHEMA app,
    TABLE billing.invoices,
    TABLE public.config;
```

### 4.4 Publish Options: insert, update, delete, truncate

```sql
-- ════════════════════════════════════════════════
-- Publish Options: ควบคุมว่า operations ไหนถูก replicate
-- ════════════════════════════════════════════════

-- Insert only (เช่น append-only log tables)
CREATE PUBLICATION log_pub 
    FOR TABLE application_logs, audit_logs
    WITH (publish = 'insert');

-- Insert และ Update แต่ไม่ Delete (ป้องกัน data loss บน subscriber)
CREATE PUBLICATION safe_pub 
    FOR TABLE critical_data
    WITH (publish = 'insert, update');

-- ทุก operations (default)
CREATE PUBLICATION full_pub 
    FOR TABLE orders
    WITH (publish = 'insert, update, delete, truncate');

-- ระวัง TRUNCATE: ถ้าไม่ต้องการให้ truncate replicate ไป
CREATE PUBLICATION no_truncate_pub 
    FOR TABLE large_table
    WITH (publish = 'insert, update, delete');
    -- ไม่มี truncate

-- แก้ไข publish options ภายหลัง
ALTER PUBLICATION my_pub SET (publish = 'insert, update, delete');
```

---

## 5. Subscription Options อย่างละเอียด

### 5.1 Connect Options

```sql
-- Connection string format
-- host=<IP/hostname> port=<port> dbname=<database> user=<user> password=<password>

-- ตัวอย่าง connection strings ต่างๆ

-- Unix socket (local)
'host=/var/run/postgresql dbname=mydb user=repuser'

-- Network connection
'host=db.example.com port=5432 dbname=mydb user=repuser password=secret'

-- SSL/TLS required
'host=db.example.com dbname=mydb user=repuser password=secret sslmode=require sslcert=/etc/ssl/client.crt sslkey=/etc/ssl/client.key'

-- Connection pooler (PgBouncer - ระวัง! ไม่รองรับ replication protocol)
-- ต้องต่อตรงกับ PostgreSQL ไม่ผ่าน PgBouncer สำหรับ logical replication

-- Multiple hosts (PostgreSQL 16+)
'host=db1.example.com,db2.example.com port=5432 dbname=mydb user=repuser password=secret'
```

### 5.2 enabled: start/stop replication

```sql
-- เริ่ม replication ทันที (default)
CREATE SUBSCRIPTION my_sub
    CONNECTION '...'
    PUBLICATION my_pub
    WITH (enabled = true);

-- สร้าง subscription แต่ยังไม่เริ่ม (setup ก่อน)
CREATE SUBSCRIPTION my_sub
    CONNECTION '...'
    PUBLICATION my_pub
    WITH (enabled = false);

-- Start/stop subscription ภายหลัง
ALTER SUBSCRIPTION my_sub ENABLE;    -- เริ่ม replication
ALTER SUBSCRIPTION my_sub DISABLE;   -- หยุด replication ชั่วคราว

-- ใช้กรณี:
-- - ต้องการ maintenance บน subscriber
-- - ต้องการ rebuild projection
-- - ป้องกัน replication ระหว่าง schema migration
```

### 5.3 copy_data: initial data copy

```sql
-- copy_data = true (default): copy ข้อมูลปัจจุบันก่อน แล้วค่อย apply changes
CREATE SUBSCRIPTION my_sub
    CONNECTION '...'
    PUBLICATION my_pub
    WITH (copy_data = true);

-- copy_data = false: ไม่ copy ข้อมูลเก่า เริ่ม replicate ตั้งแต่ตอนนี้เป็นต้นไป
-- ใช้เมื่อ:
-- - Subscriber มีข้อมูลที่ sync แล้ว (เช่น restore จาก backup)
-- - ต้องการแค่ delta changes เท่านั้น
CREATE SUBSCRIPTION my_sub
    CONNECTION '...'
    PUBLICATION my_pub
    WITH (copy_data = false);

-- ตรวจสอบ status ของ initial sync
SELECT 
    subname,
    relname,
    srsubstate,
    CASE srsubstate
        WHEN 'i' THEN 'Initialize'
        WHEN 'd' THEN 'Data is being copied'
        WHEN 'f' THEN 'Finished table copy'
        WHEN 's' THEN 'Synchronized'
        WHEN 'r' THEN 'Ready (normal replication)'
    END as state_desc
FROM pg_subscription_rel sr
JOIN pg_class c ON c.oid = sr.srrelid
JOIN pg_subscription s ON s.oid = sr.srsubid;
```

### 5.4 create_slot และ slot_name

```sql
-- create_slot = true (default): สร้าง replication slot ให้อัตโนมัติ
CREATE SUBSCRIPTION my_sub
    CONNECTION '...'
    PUBLICATION my_pub
    WITH (create_slot = true, slot_name = 'my_sub_slot');

-- create_slot = false: ใช้ slot ที่มีอยู่แล้ว
-- ต้องสร้าง slot ที่ publisher ก่อน:
-- SELECT pg_create_logical_replication_slot('my_slot', 'pgoutput');
CREATE SUBSCRIPTION my_sub
    CONNECTION '...'
    PUBLICATION my_pub
    WITH (create_slot = false, slot_name = 'my_slot');

-- ดู replication slots บน publisher
SELECT 
    slot_name,
    plugin,
    slot_type,
    database,
    active,
    confirmed_flush_lsn,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn)) as replication_lag
FROM pg_replication_slots;
```

---

## 6. Replication Identity

Replication Identity กำหนดว่า PostgreSQL จะใช้ columns ไหนในการระบุ row สำหรับ UPDATE และ DELETE operations

### 6.1 DEFAULT: Primary Key

```sql
-- Default: ใช้ Primary Key เป็น replication identity
-- เหมาะสำหรับตารางที่มี Primary Key

CREATE TABLE orders (
    id BIGINT PRIMARY KEY,  -- ← ใช้เป็น replication identity
    status VARCHAR(50),
    total_amount DECIMAL(10,2)
);

-- ตรวจสอบ replication identity
SELECT 
    relname,
    CASE relreplident
        WHEN 'd' THEN 'DEFAULT (primary key)'
        WHEN 'n' THEN 'NOTHING'
        WHEN 'f' THEN 'FULL (all columns)'
        WHEN 'i' THEN 'INDEX'
    END as replica_identity
FROM pg_class
WHERE relname IN ('orders', 'customers');
```

### 6.2 FULL: All Columns

```sql
-- FULL: ใช้ทุก columns เป็น identity
-- จำเป็นสำหรับตาราง ที่ไม่มี Primary Key หรือ Unique Index

-- ตารางที่ไม่มี PK
CREATE TABLE application_logs (
    log_time TIMESTAMP,
    level VARCHAR(10),
    message TEXT
);
-- ต้องตั้ง REPLICA IDENTITY FULL เพื่อให้ UPDATE/DELETE ทำงานได้

ALTER TABLE application_logs REPLICA IDENTITY FULL;

-- ข้อเสีย FULL: WAL ขนาดใหญ่ขึ้นเพราะต้องเก็บทุก column
-- ใช้เมื่อไม่มีทางเลือกอื่น
```

### 6.3 NOTHING

```sql
-- NOTHING: ไม่มี replication identity
-- UPDATE และ DELETE จะ fail!

ALTER TABLE temp_table REPLICA IDENTITY NOTHING;

-- ใช้เฉพาะเมื่อตารางมี INSERT เท่านั้น (append-only)
-- Publication ต้องตั้ง publish = 'insert' ด้วย
```

### 6.4 INDEX: Unique Index

```sql
-- INDEX: ใช้ Unique Index เป็น identity (ดีกว่า FULL ในบางกรณี)

CREATE TABLE users (
    id BIGINT GENERATED ALWAYS AS IDENTITY,
    email VARCHAR(255) NOT NULL,
    username VARCHAR(100) NOT NULL,
    name VARCHAR(255)
);

-- สร้าง unique index
CREATE UNIQUE INDEX users_email_idx ON users(email);

-- ตั้ง replica identity เป็น index นั้น
ALTER TABLE users REPLICA IDENTITY USING INDEX users_email_idx;

-- ข้อดีกว่า FULL: WAL เล็กกว่า แต่ยังระบุ row ได้ชัดเจน
```

### 6.5 ตัวอย่างการจัดการ Replica Identity

```sql
-- ════════════════════════════════════════════════
-- Script: ตรวจสอบและแก้ไข Replica Identity
-- ════════════════════════════════════════════════

-- ดูตารางทั้งหมดที่มีปัญหา (ไม่มี PK และไม่ได้ตั้ง FULL)
SELECT 
    n.nspname as schema_name,
    c.relname as table_name,
    CASE c.relreplident
        WHEN 'd' THEN 'DEFAULT'
        WHEN 'n' THEN 'NOTHING ⚠️'
        WHEN 'f' THEN 'FULL'
        WHEN 'i' THEN 'INDEX'
    END as replica_identity,
    (SELECT COUNT(*) FROM pg_constraint WHERE conrelid = c.oid AND contype = 'p') as has_primary_key
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE c.relkind = 'r'
  AND n.nspname NOT IN ('pg_catalog', 'information_schema')
ORDER BY n.nspname, c.relname;
```

---

## 7. Monitoring Logical Replication

### 7.1 pg_publication

```sql
-- ════════════════════════════════════════════════
-- Monitoring: Publications
-- ════════════════════════════════════════════════

-- ดู publications ทั้งหมด
SELECT 
    pubname,
    puballtables,
    pubinsert,
    pubupdate,
    pubdelete,
    pubtruncate
FROM pg_publication;

-- ดู tables ใน publication
SELECT 
    p.pubname,
    n.nspname as schema_name,
    c.relname as table_name
FROM pg_publication p
JOIN pg_publication_tables pt ON pt.pubname = p.pubname
JOIN pg_class c ON c.relname = pt.tablename
JOIN pg_namespace n ON n.nspname = pt.schemaname
ORDER BY p.pubname, n.nspname, c.relname;
```

### 7.2 pg_subscription

```sql
-- ════════════════════════════════════════════════
-- Monitoring: Subscriptions
-- ════════════════════════════════════════════════

-- ดู subscriptions ทั้งหมด
SELECT 
    subname,
    subenabled,
    subconninfo,
    subpublications,
    subslotname
FROM pg_subscription;

-- ดู subscription relations (tables)
SELECT 
    s.subname,
    c.relname,
    CASE sr.srsubstate
        WHEN 'i' THEN 'Initialize'
        WHEN 'd' THEN 'Copying data'
        WHEN 'f' THEN 'Copy finished'
        WHEN 's' THEN 'Synchronized'
        WHEN 'r' THEN 'Ready'
    END as sync_state
FROM pg_subscription s
JOIN pg_subscription_rel sr ON sr.srsubid = s.oid
JOIN pg_class c ON c.oid = sr.srrelid
ORDER BY s.subname, c.relname;
```

### 7.3 pg_stat_subscription

```sql
-- ════════════════════════════════════════════════
-- Monitoring: Subscription Stats (Replication Lag)
-- ════════════════════════════════════════════════

-- ดู replication lag และ stats
SELECT 
    subname,
    worker_type,
    pid,
    leader_pid,
    relid::regclass as table_name,
    received_lsn,
    last_msg_send_time,
    last_msg_receipt_time,
    latest_end_lsn,
    latest_end_time,
    EXTRACT(EPOCH FROM (NOW() - last_msg_receipt_time)) as seconds_since_last_msg
FROM pg_stat_subscription
ORDER BY subname;

-- Comprehensive monitoring query
WITH sub_stats AS (
    SELECT 
        subname,
        received_lsn,
        last_msg_receipt_time,
        EXTRACT(EPOCH FROM (NOW() - last_msg_receipt_time)) as lag_seconds
    FROM pg_stat_subscription
    WHERE worker_type = 'apply worker'
)
SELECT 
    ss.subname,
    CASE 
        WHEN ss.lag_seconds < 5 THEN '✅ Healthy'
        WHEN ss.lag_seconds < 30 THEN '⚠️ Slight lag'
        WHEN ss.lag_seconds < 300 THEN '🔴 High lag'
        ELSE '💀 Critical'
    END as health_status,
    ROUND(ss.lag_seconds::numeric, 2) as lag_seconds,
    ss.received_lsn
FROM sub_stats ss;
```

### 7.4 Monitoring Replication Slots บน Publisher

```sql
-- ════════════════════════════════════════════════
-- Monitoring: Replication Slots (Publisher side)
-- ════════════════════════════════════════════════

-- ตรวจสอบ replication slots และ lag
SELECT 
    slot_name,
    plugin,
    slot_type,
    database,
    active,
    active_pid,
    confirmed_flush_lsn,
    pg_current_wal_lsn() - confirmed_flush_lsn as lag_bytes,
    pg_size_pretty(pg_current_wal_lsn() - confirmed_flush_lsn) as lag_readable
FROM pg_replication_slots
WHERE slot_type = 'logical';

-- ⚠️ WARNING: Inactive logical replication slots ทำให้ WAL สะสม!
-- ถ้า subscriber ตาย และไม่ลบ slot, disk จะเต็ม

-- Alert query: slots ที่ inactive นานเกิน 1 ชั่วโมง
SELECT 
    slot_name,
    active,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn)) as wal_retained
FROM pg_replication_slots
WHERE active = false
  AND pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn) > 1073741824; -- 1 GB

-- ลบ slot ที่ไม่ใช้แล้ว (ระวัง! ทำให้ subscription ทำงานไม่ได้)
SELECT pg_drop_replication_slot('orphaned_slot_name');
```

---

## 8. การจัดการ Conflicts

### 8.1 ประเภทของ Conflicts

Conflicts ใน Logical Replication เกิดขึ้นเมื่อการเปลี่ยนแปลงจาก publisher ขัดแย้งกับข้อมูลบน subscriber:

```
Conflict Types:
1. insert_exists     - INSERT ข้อมูลที่มี PK ซ้ำอยู่แล้ว
2. update_differ     - UPDATE row ที่ถูก modified บน subscriber ด้วย
3. update_missing    - UPDATE row ที่ไม่มีอยู่บน subscriber
4. delete_missing    - DELETE row ที่ไม่มีอยู่บน subscriber
```

### 8.2 วิธีแก้ Conflict

```sql
-- ════════════════════════════════════════════════
-- เมื่อเกิด Conflict: Subscription จะหยุดทำงาน!
-- ════════════════════════════════════════════════

-- ดู error ใน subscriber logs:
-- ERROR: duplicate key value violates unique constraint "orders_pkey"
-- DETAIL: Key (id)=(12345) already exists.

-- วิธีที่ 1: Skip conflicting transaction
-- หา LSN ของ transaction ที่มีปัญหา
SELECT 
    pg_replication_origin_advance('pg_12345', '0/1234ABCD');
    -- '0/1234ABCD' คือ LSN ถัดจาก conflicting transaction

-- วิธีที่ 2: แก้ข้อมูลบน subscriber ให้ตรงกับ publisher ก่อน
-- แล้ว resume subscription
ALTER SUBSCRIPTION my_sub ENABLE;

-- วิธีที่ 3: ใช้ conflict_resolution (PostgreSQL 16+)
-- ตั้งใน postgresql.conf บน subscriber:
-- subscription_conflict_resolution = 'apply_changes'  -- apply แม้มี conflict
-- subscription_conflict_resolution = 'skip_transaction' -- skip conflicting tx

-- วิธีที่ 4: Truncate และ resync (กรณีข้อมูลเยอะมาก)
ALTER SUBSCRIPTION my_sub DISABLE;
TRUNCATE TABLE conflicting_table;  -- ลบข้อมูลทั้งหมดบน subscriber
ALTER SUBSCRIPTION my_sub REFRESH PUBLICATION;  -- sync ใหม่
ALTER SUBSCRIPTION my_sub ENABLE;
```

### 8.3 ป้องกัน Conflicts

```sql
-- ════════════════════════════════════════════════
-- Best Practices ป้องกัน Conflicts
-- ════════════════════════════════════════════════

-- 1. ไม่ write บน subscriber tables ที่ถูก replicate
--    หรือถ้า write ต้องระวัง PK collision

-- 2. ใช้ different ID ranges
--    Publisher: ID 1 - 1,000,000,000
--    Subscriber (local): ID 1,000,000,001 - 2,000,000,000

-- 3. ใช้ UUID แทน Serial ID
CREATE TABLE orders (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    -- UUID ไม่มีการชน
);

-- 4. ใช้ sequences ที่ต่างกัน
-- Publisher:
ALTER SEQUENCE orders_id_seq START WITH 1 INCREMENT BY 2;  -- odd numbers
-- Subscriber (if writing locally):
ALTER SEQUENCE orders_id_seq START WITH 2 INCREMENT BY 2;  -- even numbers
```

---

## 9. Bidirectional Replication (Experimental)

### 9.1 ข้อจำกัดของ Built-in Bidirectional

```
⚠️ PostgreSQL built-in logical replication ไม่รองรับ bidirectional
   อย่างแท้จริง มันจะเกิด infinite loop!

Node A → replicate → Node B → replicate → Node A → ... ∞

ต้องใช้ extensions เช่น:
- pglogical (2ndQuadrant/EDB)
- BDR (Bi-Directional Replication by EDB)
- Slony-I (legacy)
```

### 9.2 การป้องกัน Loop ใน Manual Bidirectional Setup

```sql
-- ════════════════════════════════════════════════
-- Replication Origin: ป้องกัน loop
-- ════════════════════════════════════════════════

-- Node A setup
CREATE PUBLICATION to_b FOR TABLE shared_table;
CREATE SUBSCRIPTION from_b
    CONNECTION 'host=nodeB dbname=mydb user=repuser'
    PUBLICATION to_a;

-- Node B setup
CREATE PUBLICATION to_a FOR TABLE shared_table;
CREATE SUBSCRIPTION from_a
    CONNECTION 'host=nodeA dbname=mydb user=repuser'
    PUBLICATION to_b;

-- PostgreSQL ใช้ replication origins เพื่อ track ที่มาของ changes
-- Changes ที่มาจาก replication จะไม่ถูก replicate กลับ (loop prevention)

-- ดู replication origins
SELECT * FROM pg_replication_origin;

-- แต่ยังมีปัญหา conflict resolution ที่ต้องจัดการเอง
```

---

## 10. pglogical Extension

pglogical เป็น extension ที่ให้ logical replication สำหรับ PostgreSQL รุ่นเก่า (9.4+) และมีฟีเจอร์เพิ่มเติม:

```bash
# ติดตั้ง pglogical
sudo apt-get install postgresql-16-pglogical

# หรือ compile from source
git clone https://github.com/2ndQuadrant/pglogical
cd pglogical
make && sudo make install
```

```sql
-- ════════════════════════════════════════════════
-- Setup pglogical บน Provider (Publisher)
-- ════════════════════════════════════════════════

-- postgresql.conf
-- shared_preload_libraries = 'pglogical'
-- wal_level = logical

-- สร้าง extension
CREATE EXTENSION pglogical;

-- สร้าง node
SELECT pglogical.create_node(
    node_name := 'provider',
    dsn := 'host=provider.example.com dbname=mydb user=pglogical password=secret'
);

-- สร้าง replication set
SELECT pglogical.create_replication_set('my_set');

-- เพิ่ม tables ใน replication set
SELECT pglogical.replication_set_add_table('my_set', 'orders');
SELECT pglogical.replication_set_add_table('my_set', 'customers');

-- เพิ่มทุก tables
SELECT pglogical.replication_set_add_all_tables('my_set', ARRAY['public']);
```

```sql
-- ════════════════════════════════════════════════
-- Setup pglogical บน Subscriber
-- ════════════════════════════════════════════════

CREATE EXTENSION pglogical;

-- สร้าง node
SELECT pglogical.create_node(
    node_name := 'subscriber',
    dsn := 'host=subscriber.example.com dbname=mydb user=pglogical password=secret'
);

-- สร้าง subscription
SELECT pglogical.create_subscription(
    subscription_name := 'my_subscription',
    provider_dsn := 'host=provider.example.com dbname=mydb user=pglogical password=secret',
    replication_sets := ARRAY['my_set'],
    synchronize_data := true,
    synchronize_structure := false  -- schema sync ต้องทำเอง
);

-- ดู status
SELECT * FROM pglogical.show_subscription_status('my_subscription');
```

---

## 11. Debezium: CDC ด้วย Logical Replication

Debezium ใช้ PostgreSQL Logical Replication ในการทำ Change Data Capture (CDC) ส่งข้อมูลไปยัง Apache Kafka:

```
Architecture: Debezium CDC

PostgreSQL ──► pgoutput/decoderbufs ──► Debezium Connector ──► Kafka Topics
(WAL)                                    (Kafka Connect)
                                              │
                                    ┌─────────┼─────────┐
                                    ▼         ▼         ▼
                               Elasticsearch  Redis   Data Lake
```

```yaml
# docker-compose.yml สำหรับ Debezium setup
version: '3.8'
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: mydb
    command: >
      postgres
      -c wal_level=logical
      -c max_replication_slots=10
      -c max_wal_senders=10
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

  zookeeper:
    image: confluentinc/cp-zookeeper:7.4.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  kafka:
    image: confluentinc/cp-kafka:7.4.0
    depends_on: [zookeeper]
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    ports:
      - "9092:9092"

  kafka-connect:
    image: debezium/connect:2.4
    depends_on: [kafka, postgres]
    environment:
      BOOTSTRAP_SERVERS: kafka:9092
      GROUP_ID: connect-cluster
      CONFIG_STORAGE_TOPIC: connect_configs
      OFFSET_STORAGE_TOPIC: connect_offsets
      STATUS_STORAGE_TOPIC: connect_statuses
    ports:
      - "8083:8083"

volumes:
  pgdata:
```

```bash
# สร้าง Debezium connector ผ่าน REST API
curl -X POST http://localhost:8083/connectors \
  -H "Content-Type: application/json" \
  -d '{
    "name": "postgres-connector",
    "config": {
      "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
      "database.hostname": "postgres",
      "database.port": "5432",
      "database.user": "postgres",
      "database.password": "postgres",
      "database.dbname": "mydb",
      "database.server.name": "mydb",
      "table.include.list": "public.orders,public.customers",
      "plugin.name": "pgoutput",
      "publication.name": "dbz_publication",
      "slot.name": "dbz_slot",
      "topic.prefix": "mydb"
    }
  }'
```

```json
// ตัวอย่าง Kafka message จาก Debezium
{
  "schema": { "type": "struct", "fields": [...] },
  "payload": {
    "before": null,
    "after": {
      "id": 12345,
      "customer_id": 678,
      "total_amount": 1500.00,
      "status": "pending",
      "created_at": 1696089600000000
    },
    "source": {
      "version": "2.4.0.Final",
      "connector": "postgresql",
      "name": "mydb",
      "ts_ms": 1696089600000,
      "snapshot": "false",
      "db": "mydb",
      "table": "orders",
      "lsn": 12345678,
      "txId": 1000
    },
    "op": "c",     // c=create, u=update, d=delete, r=read(snapshot)
    "ts_ms": 1696089600000
  }
}
```

---

## 12. Use Case: Zero-Downtime PostgreSQL Migration

### 12.1 สถานการณ์

ต้องการย้ายจาก PostgreSQL 14 ไป PostgreSQL 16 โดยไม่หยุดระบบ

```
ระบบปัจจุบัน:
- Production DB: PostgreSQL 14 บน server เก่า
- 50 tables, 10M+ rows
- ไม่สามารถ downtime เกิน 30 วินาที
```

### 12.2 ขั้นตอนการ Migrate

```bash
# ════════════════════════════════════════════════
# Phase 1: เตรียม Target Server (PG16)
# ════════════════════════════════════════════════

# ติดตั้ง PostgreSQL 16 บน new server
sudo apt-get install postgresql-16

# สร้าง database และ schema
createdb -U postgres mydb

# Copy schema จาก PG14 (ไม่ copy data)
pg_dump -h pg14-server -U postgres --schema-only mydb | \
    psql -h pg16-server -U postgres mydb

# ตรวจสอบ schema
psql -h pg16-server -U postgres -d mydb -c "\dt"
```

```sql
-- ════════════════════════════════════════════════
-- Phase 2: Setup Logical Replication
-- ════════════════════════════════════════════════

-- บน PG14 (Publisher):
-- 1. ตรวจสอบ wal_level
SHOW wal_level;  -- ต้องเป็น logical

-- 2. สร้าง replication user
CREATE USER migrator REPLICATION LOGIN PASSWORD 'migrate_secret';
GRANT SELECT ON ALL TABLES IN SCHEMA public TO migrator;

-- 3. สร้าง Publication สำหรับทุก tables
CREATE PUBLICATION migration_pub FOR ALL TABLES;
```

```sql
-- บน PG16 (Subscriber):
-- 1. สร้าง subscription พร้อม copy_data = true
CREATE SUBSCRIPTION migration_sub
    CONNECTION 'host=pg14-server port=5432 dbname=mydb user=migrator password=migrate_secret'
    PUBLICATION migration_pub
    WITH (
        copy_data = true,
        create_slot = true,
        slot_name = 'migration_slot'
    );

-- 2. ติดตาม progress การ copy data
SELECT 
    relname,
    srsubstate,
    CASE srsubstate
        WHEN 'r' THEN 'Ready ✅'
        WHEN 'd' THEN 'Copying...'
        WHEN 's' THEN 'Synced'
    END as status
FROM pg_subscription_rel sr
JOIN pg_class c ON c.oid = sr.srrelid
ORDER BY relname;
```

```bash
# ════════════════════════════════════════════════
# Phase 3: Monitor Replication Lag
# ════════════════════════════════════════════════

# Script ตรวจสอบ lag
cat > /usr/local/bin/check_replication_lag.sh << 'EOF'
#!/bin/bash
PGPASSWORD=migrate_secret psql -h pg14-server -U migrator -d mydb -c "
SELECT 
    slot_name,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn)) as lag,
    active
FROM pg_replication_slots 
WHERE slot_name = 'migration_slot';
"
EOF
chmod +x /usr/local/bin/check_replication_lag.sh

# รันทุก 10 วินาที
watch -n 10 /usr/local/bin/check_replication_lag.sh
```

```bash
# ════════════════════════════════════════════════
# Phase 4: Cutover (Switchover)
# ════════════════════════════════════════════════

# เมื่อ lag ≈ 0 bytes, ทำ cutover:

# 1. หยุด application (หรือ set read-only mode)
# systemctl stop myapp  # หรือ
psql -h pg14-server -U postgres -d mydb -c "
-- ป้องกัน new connections
ALTER DATABASE mydb CONNECTION LIMIT 0;
-- Terminate existing connections (ยกเว้น current)
SELECT pg_terminate_backend(pid) 
FROM pg_stat_activity 
WHERE datname = 'mydb' AND pid <> pg_backend_pid();
"

# 2. รอ replication ตามทัน
sleep 5

# 3. ตรวจสอบ lag = 0
psql -h pg14-server -U migrator -d mydb -c "
SELECT pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn))
FROM pg_replication_slots WHERE slot_name = 'migration_slot';
"
# ต้องได้ '0 bytes'

# 4. เปลี่ยน connection string ใน application
# เปลี่ยน DATABASE_URL จาก pg14-server ไป pg16-server

# 5. Start application ใหม่
# systemctl start myapp

# 6. ตรวจสอบว่าทุกอย่างทำงานปกติ

# 7. Cleanup (หลังมั่นใจแล้ว)
psql -h pg16-server -U postgres -d mydb -c "
DROP SUBSCRIPTION migration_sub;
"
psql -h pg14-server -U postgres -d mydb -c "
DROP PUBLICATION migration_pub;
"
```

---

## 13. Advanced Topics

### 13.1 DDL Replication

```
⚠️ Logical Replication ไม่ replicate DDL อัตโนมัติ!

ต้อง:
1. Run DDL บน Publisher ก่อน
2. Run DDL เดียวกันบน Subscriber
3. จากนั้น DML changes จะ replicate ได้

หรือใช้ pglogical ที่มี schema sync
```

```sql
-- Procedure สำหรับ apply DDL บน Publisher และ Subscriber พร้อมกัน
-- (ใช้ dblink หรือ script)

CREATE OR REPLACE PROCEDURE apply_ddl_to_both(ddl_statement TEXT)
LANGUAGE plpgsql AS $$
BEGIN
    -- Apply บน Publisher (ที่ current connection)
    EXECUTE ddl_statement;
    
    -- Apply บน Subscriber ด้วย dblink
    PERFORM dblink_exec(
        'host=subscriber.example.com dbname=mydb user=admin password=secret',
        ddl_statement
    );
    
    RAISE NOTICE 'DDL applied on both publisher and subscriber: %', ddl_statement;
END;
$$;

-- ใช้:
CALL apply_ddl_to_both('ALTER TABLE orders ADD COLUMN notes TEXT');
```

### 13.2 Performance Tuning

```sql
-- ════════════════════════════════════════════════
-- Performance: ปรับแต่ง Logical Replication
-- ════════════════════════════════════════════════

-- postgresql.conf บน Publisher
wal_sender_timeout = 60s              -- timeout สำหรับ sender
max_replication_slots = 20            -- จำนวน slots
max_slot_wal_keep_size = 10GB         -- จำกัด WAL ที่เก็บ (PG13+)

-- postgresql.conf บน Subscriber  
max_logical_replication_workers = 4   -- parallel apply workers
max_sync_workers_per_subscription = 2 -- parallel initial sync workers

-- Apply worker settings (PG15+)
-- ใช้ parallel apply เพื่อ throughput สูงขึ้น
ALTER SUBSCRIPTION my_sub SET (num_apply_workers = 4);

-- Monitoring apply lag
SELECT 
    pid,
    application_name,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    write_lag,
    flush_lag,
    replay_lag
FROM pg_stat_replication;
```

### 13.3 Logical Replication Slot Monitoring Script

```bash
#!/bin/bash
# /usr/local/bin/monitor_logical_replication.sh
# Script สำหรับ monitor logical replication health

PGHOST="${1:-localhost}"
PGPORT="${2:-5432}"
PGDATABASE="${3:-mydb}"
ALERT_LAG_GB="${4:-5}"  # Alert ถ้า lag เกิน 5 GB

echo "=== Logical Replication Monitor ==="
echo "Host: $PGHOST:$PGPORT/$PGDATABASE"
echo "Time: $(date)"
echo ""

# Check replication slots
psql -h "$PGHOST" -p "$PGPORT" -d "$PGDATABASE" -U postgres << 'EOSQL'
\echo '--- Replication Slots ---'
SELECT 
    slot_name,
    active,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn)) as lag_size,
    CASE 
        WHEN NOT active THEN '🔴 INACTIVE'
        WHEN pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn) > 5368709120 THEN '⚠️ HIGH LAG'
        ELSE '✅ OK'
    END as status
FROM pg_replication_slots
WHERE slot_type = 'logical'
ORDER BY lag_size DESC NULLS LAST;

\echo ''
\echo '--- Publications ---'
SELECT pubname, puballtables, pubinsert, pubupdate, pubdelete
FROM pg_publication;

\echo ''
\echo '--- WAL Size Used by Slots ---'
SELECT 
    pg_size_pretty(SUM(pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn))) as total_wal_retained
FROM pg_replication_slots
WHERE slot_type = 'logical';
EOSQL
```

---

## 14. สรุปและ Best Practices

### 14.1 Checklist ก่อน Setup

```
□ ตรวจสอบ wal_level = logical บน publisher
□ max_replication_slots เพียงพอ
□ max_wal_senders เพียงพอ
□ สร้าง tables บน subscriber แล้ว (schema ตรงกัน)
□ Tables มี Primary Key หรือ Replica Identity ที่เหมาะสม
□ Replication user มี permissions ที่จำเป็น
□ Network: subscriber เข้าถึง publisher ได้
□ pg_hba.conf อนุญาต connections
□ max_slot_wal_keep_size กำหนดไว้ (ป้องกัน disk เต็ม)
```

### 14.2 Do's and Don'ts

```
✅ DO:
- ใช้ UUID แทน Serial สำหรับ distributed systems
- Monitor replication lag อย่างต่อเนื่อง
- Drop replication slots เมื่อไม่ใช้แล้ว
- Test failover procedure ก่อน production
- ตั้ง max_slot_wal_keep_size เพื่อป้องกัน disk เต็ม

❌ DON'T:
- ปล่อย inactive replication slots (disk เต็ม!)
- Write ลงบน replicated tables บน subscriber โดยไม่ระวัง
- ทำ DDL บน publisher โดยไม่ทำบน subscriber ด้วย
- ลืม test restore procedure ของ subscriber
- ใช้ PgBouncer สำหรับ replication connection
```

---

## 15. Lab Exercise

### Lab 1: Setup Basic Logical Replication

```bash
# ใช้ Docker สำหรับ lab
cat > docker-compose.yml << 'EOF'
version: '3.8'
services:
  publisher:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: source_db
    command: >
      postgres
      -c wal_level=logical
      -c max_replication_slots=5
      -c max_wal_senders=5
    ports:
      - "5433:5432"
    networks:
      - pgnet

  subscriber:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: target_db
    ports:
      - "5434:5432"
    networks:
      - pgnet

networks:
  pgnet:
    driver: bridge
EOF

docker-compose up -d
sleep 5

# Setup Publisher
PGPASSWORD=postgres psql -h localhost -p 5433 -U postgres -d source_db << 'EOSQL'
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    price DECIMAL(10,2),
    stock INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO products (name, price, stock) VALUES
    ('iPhone 15', 32990.00, 100),
    ('MacBook Pro', 89990.00, 50),
    ('AirPods Pro', 9990.00, 200);

CREATE PUBLICATION products_pub FOR TABLE products;

CREATE USER repuser REPLICATION LOGIN PASSWORD 'reppass';
GRANT SELECT ON products TO repuser;
EOSQL

# Setup Subscriber Schema
PGPASSWORD=postgres psql -h localhost -p 5434 -U postgres -d target_db << 'EOSQL'
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    price DECIMAL(10,2),
    stock INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- สร้าง subscription (ใช้ hostname 'publisher' เพราะอยู่ใน Docker network เดียวกัน)
CREATE SUBSCRIPTION products_sub
    CONNECTION 'host=publisher port=5432 dbname=source_db user=repuser password=reppass'
    PUBLICATION products_pub
    WITH (copy_data = true);
EOSQL

echo "Setup complete! Testing replication..."
sleep 3

# Test replication
PGPASSWORD=postgres psql -h localhost -p 5433 -U postgres -d source_db -c "
INSERT INTO products (name, price, stock) VALUES ('iPad Pro', 39990.00, 75);"

sleep 2

echo "=== Data on Publisher ==="
PGPASSWORD=postgres psql -h localhost -p 5433 -U postgres -d source_db -c "SELECT * FROM products ORDER BY id;"

echo "=== Data on Subscriber ==="
PGPASSWORD=postgres psql -h localhost -p 5434 -U postgres -d target_db -c "SELECT * FROM products ORDER BY id;"
```

---

**สรุป Part 66: Logical Replication**

Logical Replication เป็นเครื่องมือที่ทรงพลังสำหรับ:
- **Zero-downtime upgrades**: ย้าย PostgreSQL version โดยไม่หยุดระบบ
- **Selective replication**: ส่งเฉพาะ tables ที่ต้องการ
- **Cross-platform**: ข้ามระบบปฏิบัติการและ architecture
- **Data pipelines**: CDC ด้วย Debezium + Kafka

กุญแจสำคัญในการใช้งาน:
1. ตั้ง `wal_level = logical` ก่อนเสมอ
2. ดูแล replication slots ไม่ให้ค้างอยู่โดยไม่ใช้
3. Tables ต้องมี Primary Key หรือ REPLICA IDENTITY ที่เหมาะสม
4. Monitor replication lag อย่างสม่ำเสมอ

# Part 21: PostgreSQL Replication - Primary-Replica Setup

## บทนำ: ทำไม Replication ถึงสำคัญ

ในระบบฐานข้อมูลระดับ Production การมีเซิร์ฟเวอร์เพียงเครื่องเดียวคือความเสี่ยงที่รับไม่ได้ หากเซิร์ฟเวอร์หลักล่มลง ระบบทั้งหมดจะหยุดทำงาน และข้อมูลอาจสูญหาย PostgreSQL Replication คือกลไกที่ช่วยให้เราสร้างสำเนาของฐานข้อมูลไปยังเซิร์ฟเวอร์อื่น ทำให้ระบบมีความทนทานสูง (High Availability) และสามารถกระจาย Query Load ได้

**ประโยชน์หลักของ Replication:**
- **High Availability**: หาก Primary ล่ม Replica สามารถรับ Role ต่อได้
- **Disaster Recovery**: มีสำเนาข้อมูลในสถานที่อื่น
- **Read Scaling**: กระจาย SELECT Query ไปยัง Replica หลายตัว
- **Zero Downtime Backup**: ทำ Backup บน Replica แทนที่จะทำบน Primary

---

## 1. แนวคิด Replication: Physical vs Logical

### 1.1 Physical Replication (Streaming Replication)

Physical Replication คือการคัดลอกข้อมูลในระดับ Block/Page ของ Disk โดยตรง หมายความว่า Replica จะได้รับข้อมูลเหมือนกันทุกอย่างกับ Primary ระดับ Byte-for-Byte

```
Primary Server                    Replica Server
┌─────────────────┐               ┌─────────────────┐
│  PostgreSQL     │               │  PostgreSQL      │
│  Data Files     │──WAL Stream──▶│  Data Files      │
│  (pg_data/)     │               │  (pg_data/)      │
└─────────────────┘               └─────────────────┘
```

**ลักษณะเด่น:**
- คัดลอก WAL (Write-Ahead Log) ทั้งหมด
- Replica เป็นสำเนาที่เหมือนกัน 100% กับ Primary
- ไม่สามารถ Filter หรือ Transform ข้อมูลได้
- ต้องใช้ PostgreSQL Version เดียวกัน

### 1.2 Logical Replication

Logical Replication คัดลอกในระดับ SQL Operations (INSERT, UPDATE, DELETE) ทำให้มีความยืดหยุ่นมากกว่า

```
Primary Server                    Replica Server
┌─────────────────┐               ┌─────────────────┐
│  Publication    │               │  Subscription    │
│  (ระบุ Tables)  │──Logical──────▶│  (รับ Changes)   │
│                 │               │                  │
└─────────────────┘               └─────────────────┘
```

**ลักษณะเด่น:**
- เลือก Replicate เฉพาะบาง Table ได้
- PostgreSQL Version ต่างกันได้ (ในขอบเขตหนึ่ง)
- Replica สามารถมีข้อมูลเพิ่มเติมได้
- รองรับ Multi-Master Replication

### 1.3 Streaming vs File-based Replication

**Streaming Replication:**
```
Primary ──(WAL Records)──▶ Replica
         Real-time network
```
- ส่ง WAL Records แบบ Real-time ผ่าน Network
- Lag น้อยมาก (ปกติ milliseconds)
- ต้องมี Network Connection ตลอดเวลา

**File-based (Archive) Replication:**
```
Primary ──(WAL Files)──▶ Archive ──▶ Replica
         Batch files         เมื่อพร้อม
```
- ส่ง WAL Files ไปเก็บใน Archive ก่อน
- Replica อ่านจาก Archive
- Lag มากกว่า (อาจเป็น minutes)
- ทำงานได้แม้ไม่มี Direct Connection

---

## 2. WAL (Write-Ahead Log): หัวใจของ Replication

### 2.1 WAL คืออะไร

WAL หรือ Write-Ahead Log คือ Transaction Log ที่ PostgreSQL เขียนทุก Change ก่อนที่จะเปลี่ยนข้อมูลจริงในไฟล์ Data

```
┌──────────────────────────────────────────────────────┐
│  PostgreSQL Transaction Flow                          │
│                                                        │
│  1. BEGIN TRANSACTION                                  │
│  2. เขียน Change ลงใน WAL Buffer (Memory)             │
│  3. COMMIT                                             │
│  4. WAL Buffer ถูก Flush ลงดิสก์ (pg_wal/)            │
│  5. เปลี่ยนข้อมูลใน Data Files (อาจทำทีหลัง)          │
└──────────────────────────────────────────────────────┘
```

### 2.2 WAL ทำงานอย่างไร

```
Memory:                    Disk:
┌─────────────┐           ┌─────────────┐
│ WAL Buffer  │──flush──▶ │  pg_wal/    │
│             │           │  000000010...│
│ Shared      │           │  000000020...│
│ Buffers     │           └─────────────┘
└─────────────┘
      │
      │ background writer
      ▼
┌─────────────┐
│ Data Files  │
│ base/       │
└─────────────┘
```

**WAL Segment Files:**
```bash
# ดูไฟล์ WAL ใน pg_wal directory
ls -la $PGDATA/pg_wal/
# 000000010000000000000001  (16MB ต่อไฟล์)
# 000000010000000000000002
# 000000010000000000000003
```

### 2.3 WAL Level

PostgreSQL มี WAL Level 3 ระดับ:

```
minimal   ◀──── wal_level ────▶  logical
                    │
                 replica
```

- **minimal**: เขียนน้อยที่สุด ไม่รองรับ Replication
- **replica**: รองรับ Physical Replication (Streaming/Archive)
- **logical**: รองรับทั้ง Physical และ Logical Replication

---

## 3. Primary Server Configuration

### 3.1 postgresql.conf Settings

```ini
# /etc/postgresql/15/main/postgresql.conf
# หรือ $PGDATA/postgresql.conf

#==========================================================
# REPLICATION SETTINGS
#==========================================================

# WAL Level - ต้องเป็น 'replica' หรือ 'logical' สำหรับ Replication
wal_level = replica

# จำนวน WAL Sender Process สูงสุด
# แต่ละ Replica ต้องการ 1 Sender
# เพิ่ม buffer สักหน่อยสำหรับ pg_basebackup
max_wal_senders = 10

# เก็บ WAL ไว้เท่าไหร่เพื่อให้ Replica ที่ Lag ตามได้
# PostgreSQL 13+: ใช้ wal_keep_size (หน่วย MB)
wal_keep_size = 1GB

# PostgreSQL 12 และก่อนหน้า: ใช้ wal_keep_segments
# wal_keep_segments = 64

# Replication Slot: เก็บ WAL จนกว่า Replica จะอ่านไป
# (ระวัง disk เต็มถ้า Replica ตายนาน)
max_replication_slots = 10

# Synchronous Replication: รอ Replica ยืนยันก่อน COMMIT
# ใส่ชื่อ Replica หรือ '' สำหรับ Async
synchronous_standby_names = ''

# Hot Standby: อนุญาตให้ Replica รับ READ queries
hot_standby = on

# เปิด Checksums สำหรับ Data Integrity
# (ต้อง initdb --data-checksums ตอนสร้าง cluster)
# data_checksums = on

#==========================================================
# CONNECTION SETTINGS  
#==========================================================

# ฟัง Connection จากทุก IP (ปรับตาม Security ที่ต้องการ)
listen_addresses = '*'

# Port เริ่มต้น
port = 5432

#==========================================================
# WAL ARCHIVING (Optional - สำหรับ File-based Replication)
#==========================================================

# เปิด WAL Archiving
archive_mode = on

# คำสั่งสำหรับ Archive WAL Files
archive_command = 'test ! -f /var/lib/postgresql/archive/%f && cp %p /var/lib/postgresql/archive/%f'

# ตรวจสอบ Archive Status
archive_cleanup_command = 'pg_archivecleanup /var/lib/postgresql/archive %r'
```

### 3.2 pg_hba.conf: อนุญาต Replication Connection

```
# /etc/postgresql/15/main/pg_hba.conf
# หรือ $PGDATA/pg_hba.conf

# TYPE  DATABASE    USER            ADDRESS         METHOD

# อนุญาต replication user จาก Replica IP
# local connections
local   replication  replicator                     trust

# IPv4 connections - ใส่ IP ของ Replica
host    replication  replicator  192.168.1.0/24    md5

# หรือถ้าใช้ใน Docker Network
host    replication  replicator  172.16.0.0/12     md5

# สำหรับ Production ควรใช้ scram-sha-256
host    replication  replicator  10.0.0.0/8        scram-sha-256
```

### 3.3 สร้าง Replication User

```sql
-- เชื่อมต่อ Primary ในฐานะ postgres superuser

-- สร้าง User สำหรับ Replication
CREATE USER replicator 
  WITH REPLICATION 
  LOGIN 
  PASSWORD 'StrongReplicationPassword123!';

-- ตรวจสอบ
SELECT usename, userepass, usecreatedb, usecreaterole, userepl 
FROM pg_user 
WHERE usename = 'replicator';

-- หรือดู roles
\du replicator
```

### 3.4 Reload Configuration

```bash
# Reload โดยไม่ต้อง Restart (สำหรับ postgresql.conf บางตัว)
pg_ctl reload -D $PGDATA

# หรือใช้ SQL
SELECT pg_reload_conf();

# บาง Settings ต้อง Restart เช่น wal_level, max_wal_senders
pg_ctl restart -D $PGDATA

# บน systemd
systemctl reload postgresql
systemctl restart postgresql
```

---

## 4. Replica Server Setup

### 4.1 pg_basebackup: สร้าง Base Backup

pg_basebackup คือเครื่องมือสำหรับสร้าง Base Backup ของ PostgreSQL ซึ่งใช้เป็นจุดเริ่มต้นของ Replica

```bash
# รันบน Replica Server (หรือ Primary ก็ได้)

# สร้าง Data Directory บน Replica (ต้องเป็น Owner postgres)
mkdir -p /var/lib/postgresql/15/replica
chown postgres:postgres /var/lib/postgresql/15/replica

# รัน pg_basebackup
pg_basebackup \
  --host=192.168.1.10 \        # IP ของ Primary
  --port=5432 \
  --username=replicator \
  --pgdata=/var/lib/postgresql/15/replica \   # ที่เก็บข้อมูล Replica
  --wal-method=stream \        # Stream WAL ระหว่าง Backup
  --checkpoint=fast \          # ทำ Checkpoint ทันที
  --label="Replica Backup $(date)" \
  --progress \                 # แสดง Progress
  --verbose

# ใส่ Password เมื่อถูกถาม
# Password: StrongReplicationPassword123!

# ผลลัพธ์จะเห็น:
# pg_basebackup: initiating base backup, waiting for checkpoint to complete
# pg_basebackup: checkpoint completed
# pg_basebackup: write-ahead log start point: 0/4000028 on timeline 1
# pg_basebackup: starting background WAL receiver
# pg_basebackup: base backup completed
```

**pg_basebackup Options เพิ่มเติม:**
```bash
# เก็บเป็น tar.gz แทน (ประหยัด disk ระหว่าง transfer)
pg_basebackup \
  --host=primary \
  --username=replicator \
  --pgdata=/tmp/backup \
  --format=tar \
  --compress=9 \
  --wal-method=fetch

# สร้าง standby.signal ด้วย (PostgreSQL 12+)
pg_basebackup \
  --host=primary \
  --username=replicator \
  --pgdata=/var/lib/postgresql/15/replica \
  --wal-method=stream \
  --write-recovery-conf      # สร้าง standby.signal และ postgresql.auto.conf อัตโนมัติ
```

### 4.2 Recovery Configuration (PostgreSQL 12+)

ตั้งแต่ PostgreSQL 12 เป็นต้นมา recovery.conf ถูกรวมเข้ากับ postgresql.conf

```bash
# สร้างไฟล์ standby.signal เพื่อบอกว่านี่คือ Replica
touch /var/lib/postgresql/15/replica/standby.signal
chown postgres:postgres /var/lib/postgresql/15/replica/standby.signal
```

```ini
# /var/lib/postgresql/15/replica/postgresql.conf
# (หรือสร้าง postgresql.auto.conf แยก)

#==========================================================
# STANDBY SETTINGS
#==========================================================

# Connection String ไปยัง Primary
primary_conninfo = 'host=192.168.1.10 port=5432 user=replicator password=StrongReplicationPassword123! application_name=replica1'

# ใช้ Replication Slot (ถ้าสร้างไว้บน Primary)
# primary_slot_name = 'replica1_slot'

# Recover ไปยัง Latest State
recovery_target_timeline = 'latest'

# Hot Standby: อนุญาต Read-Only Queries
hot_standby = on

# Conflict Resolution
hot_standby_feedback = on

# เปิดรับ Connection จากทุก IP (ถ้าต้องการ Read Queries)
listen_addresses = '*'
port = 5432
```

**สำหรับ PostgreSQL 11 และก่อนหน้า (recovery.conf):**
```ini
# $PGDATA/recovery.conf

standby_mode = 'on'
primary_conninfo = 'host=192.168.1.10 port=5432 user=replicator password=StrongReplicationPassword123!'
recovery_target_timeline = 'latest'
trigger_file = '/tmp/promote_to_primary'
```

### 4.3 เริ่มต้น Replica

```bash
# เริ่มต้น PostgreSQL บน Replica
pg_ctl start -D /var/lib/postgresql/15/replica

# หรือบน systemd
systemctl start postgresql

# ดู Log ว่า Replication เริ่มทำงาน
tail -f /var/log/postgresql/postgresql-15-main.log

# ควรเห็น:
# LOG:  entering standby mode
# LOG:  redo starts at 0/4000028
# LOG:  consistent recovery state reached at 0/4000100
# LOG:  database system is ready to accept read only connections
# LOG:  started streaming WAL from primary at 0/5000000 on timeline 1
```

---

## 5. ทดสอบ Replication

### 5.1 สร้าง Table บน Primary

```sql
-- เชื่อมต่อ Primary
psql -h 192.168.1.10 -U postgres

-- สร้าง Database ทดสอบ
CREATE DATABASE replication_test;
\c replication_test

-- สร้าง Table
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Insert ข้อมูล
INSERT INTO products (name, price) VALUES 
    ('MacBook Pro', 89900.00),
    ('iPhone 15', 35900.00),
    ('iPad Pro', 42900.00),
    ('AirPods Pro', 8990.00),
    ('Apple Watch', 15900.00);

-- ตรวจสอบ
SELECT * FROM products;
```

### 5.2 ตรวจสอบบน Replica

```sql
-- เชื่อมต่อ Replica
psql -h 192.168.1.20 -U postgres

\c replication_test

-- ควรเห็นข้อมูลเดียวกัน
SELECT * FROM products;

-- ลอง Write บน Replica - จะได้ Error
INSERT INTO products (name, price) VALUES ('Test', 100);
-- ERROR:  cannot execute INSERT in a read-only transaction
```

### 5.3 ทดสอบ Replication Lag

```sql
-- บน Primary: ดู Replication Status
SELECT 
    client_addr,
    application_name,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    write_lag,
    flush_lag,
    replay_lag,
    sync_state
FROM pg_stat_replication;

-- ผลลัพธ์ตัวอย่าง:
--  client_addr  | application_name | state |  sent_lsn  | write_lag | flush_lag | replay_lag | sync_state
-- --------------+------------------+-------+------------+-----------+-----------+------------+------------
--  192.168.1.20 | replica1         | streaming | 0/7000060 | 00:00:00 | 00:00:00 | 00:00:00 | async
```

---

## 6. pg_stat_replication: Monitoring

```sql
-- ข้อมูล Replication Status แบบละเอียด
SELECT 
    pid,
    usesysid,
    usename,
    application_name,
    client_addr,
    client_hostname,
    client_port,
    backend_start,
    backend_xmin,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    write_lag,
    flush_lag,
    replay_lag,
    sync_priority,
    sync_state,
    reply_time
FROM pg_stat_replication;

-- คำนวณ Replication Lag เป็น Bytes
SELECT 
    application_name,
    pg_wal_lsn_diff(sent_lsn, replay_lsn) AS replication_lag_bytes,
    pg_size_pretty(pg_wal_lsn_diff(sent_lsn, replay_lsn)) AS replication_lag_pretty
FROM pg_stat_replication;

-- บน Replica: ดู Recovery Status
SELECT 
    pg_is_in_recovery() AS is_replica,
    pg_last_wal_receive_lsn() AS received_lsn,
    pg_last_wal_replay_lsn() AS replayed_lsn,
    pg_last_xact_replay_timestamp() AS last_replay_time,
    NOW() - pg_last_xact_replay_timestamp() AS replication_lag;
```

---

## 7. Synchronous vs Asynchronous Replication

### 7.1 Asynchronous Replication (Default)

```
Primary                     Replica
   │                           │
   │  COMMIT                   │
   │──────────────────────────▶│
   │                           │
   │  OK (ทันที)               │
   │◀──────────────────────────│
   │                           │  
   │                   (replay ทีหลัง)
```

- Primary ตอบ COMMIT ทันทีโดยไม่รอ Replica
- ประสิทธิภาพสูง
- มีโอกาสสูญข้อมูลได้ถ้า Primary ล่มก่อน Replica รับ

### 7.2 Synchronous Replication

```
Primary                     Replica
   │                           │
   │  COMMIT                   │
   │──────────────────────────▶│
   │                           │  WAL Flushed
   │  Ack                      │
   │◀──────────────────────────│
   │                           │
   │  OK (หลังจาก Ack)         │
```

**การตั้งค่า Synchronous Replication:**

```ini
# postgresql.conf บน Primary

# ชื่อ Replica ที่ต้องรอ (application_name ใน primary_conninfo ของ Replica)
synchronous_standby_names = 'replica1'

# รอ 1 ใน N Replicas
synchronous_standby_names = 'ANY 1 (replica1, replica2, replica3)'

# รอทุก Replica
synchronous_standby_names = 'ALL (replica1, replica2)'

# FIRST N - รอ N อันดับแรก
synchronous_standby_names = 'FIRST 2 (replica1, replica2, replica3)'
```

**ระดับ Synchronous:**
```ini
# ระดับ synchronous_commit
synchronous_commit = on         # รอ WAL ถูก Flush บน Replica
synchronous_commit = remote_write  # รอ WAL ถูก Write บน Replica (ไม่ต้อง Flush)
synchronous_commit = remote_apply  # รอ WAL ถูก Apply บน Replica
synchronous_commit = local      # รอเฉพาะ Local (Async Replication)
synchronous_commit = off        # ไม่รอเลย (ไม่แนะนำ)
```

---

## 8. Replication Slots

### 8.1 ทำไมต้องใช้ Replication Slots

ปัญหาของ Streaming Replication ปกติ: ถ้า Replica ตายหรือ Lag มาก Primary อาจลบ WAL Files ที่ Replica ยังไม่ได้อ่าน

```
ปัญหา:
Primary ลบ WAL0001  ─────▶  Replica ยังต้องการ WAL0001
                                  (ERROR: could not find WAL)
```

Replication Slot แก้ปัญหานี้โดยบอก Primary ว่า "ยังไม่ต้องลบ WAL จนกว่า Replica จะอ่าน"

### 8.2 สร้าง Replication Slot

```sql
-- บน Primary: สร้าง Physical Replication Slot
SELECT pg_create_physical_replication_slot('replica1_slot');

-- ดู Replication Slots
SELECT 
    slot_name,
    slot_type,
    database,
    active,
    active_pid,
    xmin,
    catalog_xmin,
    restart_lsn,
    confirmed_flush_lsn,
    wal_status,
    safe_wal_size
FROM pg_replication_slots;

-- สร้าง Logical Replication Slot
SELECT pg_create_logical_replication_slot('logical_slot1', 'pgoutput');

-- ลบ Replication Slot
SELECT pg_drop_replication_slot('replica1_slot');
```

### 8.3 ตั้งค่า Replica ให้ใช้ Slot

```ini
# postgresql.conf บน Replica
primary_slot_name = 'replica1_slot'
```

**คำเตือน:** Replication Slots อาจทำให้ Disk เต็มถ้า Replica ตายนาน เพราะ Primary จะเก็บ WAL สะสม

```sql
-- Monitor ขนาด WAL ที่ค้างอยู่ใน Slot
SELECT 
    slot_name,
    pg_size_pretty(
        pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)
    ) AS retained_wal_size
FROM pg_replication_slots
WHERE active = false;

-- Alert ถ้า Slot ค้างมากกว่า 5GB
SELECT slot_name
FROM pg_replication_slots
WHERE active = false 
  AND pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) > 5 * 1024^3;
```

---

## 9. Cascading Replication (Replica of Replica)

Cascading Replication ช่วยให้เราสร้าง Replica Chain โดย Replica ตัวหนึ่งส่ง WAL ต่อให้ Replica อื่น

```
Primary ──────▶ Replica1 ──────▶ Replica2
                          ──────▶ Replica3
```

**ประโยชน์:**
- ลด Load บน Primary (ส่ง WAL ไปแค่ Replica1)
- รองรับ Replica ในหลาย Data Center

```ini
# Replica1: ทำหน้าที่ส่ง WAL ต่อ
# postgresql.conf บน Replica1
wal_level = replica
max_wal_senders = 5
hot_standby = on

# ต้องเปิดด้วย:
recovery_min_apply_delay = 0
```

```ini
# Replica2: รับจาก Replica1
# postgresql.conf บน Replica2
primary_conninfo = 'host=replica1_ip port=5432 user=replicator password=xxx'
```

---

## 10. Hot Standby: Read Queries บน Replica

Hot Standby อนุญาตให้รัน SELECT บน Replica ขณะที่กำลัง Replay WAL อยู่

```ini
# postgresql.conf บน Replica
hot_standby = on

# อนุญาต Feedback ไปยัง Primary (ป้องกัน Conflict)
hot_standby_feedback = on

# Delay ก่อน Replay (สำหรับ Delayed Replica - ป้องกัน Accidental Delete)
recovery_min_apply_delay = '30min'    # รอ 30 นาทีก่อน Apply
```

**Hot Standby Conflict:**
```sql
-- บน Replica อาจเกิด Error แบบนี้:
-- ERROR: canceling statement due to conflict with recovery
-- DETAIL: User was holding lock that conflicted with recovery

-- แก้ไขโดย:
-- 1. เพิ่ม max_standby_streaming_delay
-- postgresql.conf บน Replica:
max_standby_streaming_delay = 30s   -- รอ 30 วินาทีก่อนยกเลิก Query
max_standby_archive_delay = 60s
```

---

## 11. Failover Process: Manual Promote

### 11.1 ตรวจสอบว่า Primary ล่มจริง

```bash
# ตรวจ Connection ไปยัง Primary
pg_isready -h primary_ip -p 5432
# /tmp/.s.PGSQL.5432 - accepting connections
# หรือ
# /tmp/.s.PGSQL.5432 - no response

# ดู Replication Status บน Replica
psql -h replica_ip -U postgres -c "SELECT pg_is_in_recovery();"
# t = ยังเป็น Replica อยู่
```

### 11.2 Promote Replica เป็น Primary

```bash
# วิธีที่ 1: ใช้ pg_ctl promote
pg_ctl promote -D /var/lib/postgresql/15/main

# วิธีที่ 2: สร้าง trigger file (สำหรับ PostgreSQL 11 และก่อนหน้า)
touch /tmp/promote_to_primary

# วิธีที่ 3: ใช้ SQL Function (PostgreSQL 12+)
psql -c "SELECT pg_promote();"
```

**หลัง Promote:**
```bash
# ตรวจสอบว่า Promote สำเร็จ
psql -h new_primary_ip -U postgres -c "SELECT pg_is_in_recovery();"
# f = ไม่ได้เป็น Replica แล้ว (เป็น Primary แล้ว)

# ดู Log
tail -f /var/log/postgresql/postgresql-*.log
# LOG:  received promote request
# LOG:  redo in progress
# LOG:  selected new timeline ID: 2
# LOG:  archive recovery complete
# LOG:  database system is ready to accept connections
```

### 11.3 อัปเดต DNS / Load Balancer

```bash
# อัปเดต DNS ให้ชี้ไปยัง New Primary
# (ขึ้นอยู่กับ Infrastructure)

# หรืออัปเดต Application Config
# DB_HOST=new_primary_ip

# ถ้าใช้ HAProxy หรือ PgBouncer ให้อัปเดต Config
```

---

## 12. pg_rewind: Sync หลัง Failover

หลัง Failover Old Primary อาจมี Diverged Timeline ทำให้ไม่สามารถ Rejoin Cluster ได้โดยตรง pg_rewind ช่วยแก้ปัญหานี้

```
Timeline ก่อน Failover:
Timeline 1: A──B──C──D──E (Primary)
                         │ (Failover ที่ E)
Timeline 2:              E──F──G (New Primary)

Old Primary: A──B──C──D──E──X──Y (Diverged)
```

```bash
# หยุด Old Primary (ถ้ายังทำงานอยู่)
pg_ctl stop -D /var/lib/postgresql/15/main

# รัน pg_rewind
pg_rewind \
  --target-pgdata=/var/lib/postgresql/15/main \
  --source-server="host=new_primary_ip port=5432 user=postgres dbname=postgres" \
  --progress

# หรือ Restore จาก WAL Archive
pg_rewind \
  --target-pgdata=/var/lib/postgresql/15/main \
  --source-pgdata=/var/lib/postgresql/15/new_primary_data \
  --progress

# สร้าง standby.signal ใหม่
touch /var/lib/postgresql/15/main/standby.signal

# อัปเดต primary_conninfo
cat >> /var/lib/postgresql/15/main/postgresql.auto.conf << EOF
primary_conninfo = 'host=new_primary_ip port=5432 user=replicator password=xxx'
EOF

# เริ่ม Old Primary ในฐานะ Replica ใหม่
pg_ctl start -D /var/lib/postgresql/15/main
```

**เตรียม pg_rewind:**
```sql
-- ต้องเปิด wal_log_hints บน Primary (หรือ data_checksums ตอน initdb)
-- postgresql.conf
wal_log_hints = on

-- หรือ Full Page Writes
full_page_writes = on
```

---

## 13. Docker Compose: Primary + 2 Replicas

### 13.1 โครงสร้างไฟล์

```
postgres-replication/
├── docker-compose.yml
├── primary/
│   ├── postgresql.conf
│   ├── pg_hba.conf
│   └── init/
│       └── 01-create-replicator.sql
├── replica1/
│   └── postgresql.conf
├── replica2/
│   └── postgresql.conf
└── scripts/
    ├── setup-replicas.sh
    └── test-replication.sh
```

### 13.2 docker-compose.yml

```yaml
version: '3.8'

networks:
  pg-network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/24

volumes:
  primary-data:
  replica1-data:
  replica2-data:

services:
  # ==========================================
  # PRIMARY SERVER
  # ==========================================
  primary:
    image: postgres:15-alpine
    container_name: pg-primary
    hostname: pg-primary
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres123
      POSTGRES_REPLICATION_USER: replicator
      POSTGRES_REPLICATION_PASSWORD: replicator123
    volumes:
      - primary-data:/var/lib/postgresql/data
      - ./primary/postgresql.conf:/etc/postgresql/postgresql.conf
      - ./primary/pg_hba.conf:/etc/postgresql/pg_hba.conf
      - ./primary/init:/docker-entrypoint-initdb.d
    command: postgres -c config_file=/etc/postgresql/postgresql.conf
    networks:
      pg-network:
        ipv4_address: 172.20.0.10
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 10

  # ==========================================
  # REPLICA 1
  # ==========================================
  replica1:
    image: postgres:15-alpine
    container_name: pg-replica1
    hostname: pg-replica1
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres123
      PGUSER: replicator
      PGPASSWORD: replicator123
      PRIMARY_HOST: pg-primary
      PRIMARY_PORT: 5432
    volumes:
      - replica1-data:/var/lib/postgresql/data
      - ./scripts/setup-replicas.sh:/docker-entrypoint-initdb.d/setup-replica.sh
    networks:
      pg-network:
        ipv4_address: 172.20.0.11
    ports:
      - "5433:5432"
    depends_on:
      primary:
        condition: service_healthy
    command: >
      bash -c "
        if [ ! -f /var/lib/postgresql/data/PG_VERSION ]; then
          echo 'Setting up replica...'
          until pg_isready -h pg-primary -p 5432 -U postgres; do
            echo 'Waiting for primary...'
            sleep 2
          done
          
          PGPASSWORD=replicator123 pg_basebackup \
            -h pg-primary \
            -p 5432 \
            -U replicator \
            -D /var/lib/postgresql/data \
            -Xs -R -P -W
          
          # แก้ไข primary_conninfo
          cat >> /var/lib/postgresql/data/postgresql.auto.conf << 'EOF'
primary_conninfo = 'host=pg-primary port=5432 user=replicator password=replicator123 application_name=replica1'
hot_standby = on
EOF
          
          echo 'Replica setup complete!'
        fi
        
        exec docker-entrypoint.sh postgres
      "

  # ==========================================
  # REPLICA 2
  # ==========================================
  replica2:
    image: postgres:15-alpine
    container_name: pg-replica2
    hostname: pg-replica2
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres123
      PGUSER: replicator
      PGPASSWORD: replicator123
      PRIMARY_HOST: pg-primary
      PRIMARY_PORT: 5432
    volumes:
      - replica2-data:/var/lib/postgresql/data
    networks:
      pg-network:
        ipv4_address: 172.20.0.12
    ports:
      - "5434:5432"
    depends_on:
      primary:
        condition: service_healthy
    command: >
      bash -c "
        if [ ! -f /var/lib/postgresql/data/PG_VERSION ]; then
          until pg_isready -h pg-primary -p 5432 -U postgres; do
            sleep 2
          done
          
          PGPASSWORD=replicator123 pg_basebackup \
            -h pg-primary \
            -p 5432 \
            -U replicator \
            -D /var/lib/postgresql/data \
            -Xs -R -P -W
          
          cat >> /var/lib/postgresql/data/postgresql.auto.conf << 'EOF'
primary_conninfo = 'host=pg-primary port=5432 user=replicator password=replicator123 application_name=replica2'
hot_standby = on
EOF
        fi
        
        exec docker-entrypoint.sh postgres
      "

  # ==========================================
  # PGADMIN (Optional - Web UI)
  # ==========================================
  pgadmin:
    image: dpage/pgadmin4:latest
    container_name: pgadmin
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@example.com
      PGADMIN_DEFAULT_PASSWORD: admin123
    ports:
      - "8080:80"
    networks:
      - pg-network
    depends_on:
      - primary
```

### 13.3 primary/postgresql.conf

```ini
# Primary PostgreSQL Configuration

#------------------------------------------------------------
# FILE LOCATIONS
#------------------------------------------------------------
data_directory = '/var/lib/postgresql/data'
hba_file = '/etc/postgresql/pg_hba.conf'
ident_file = '/var/lib/postgresql/data/pg_ident.conf'

#------------------------------------------------------------
# CONNECTIONS AND AUTHENTICATION
#------------------------------------------------------------
listen_addresses = '*'
port = 5432
max_connections = 200

#------------------------------------------------------------
# REPLICATION
#------------------------------------------------------------
wal_level = replica
max_wal_senders = 10
max_replication_slots = 10
wal_keep_size = 1024           # 1 GB
hot_standby = on
synchronous_standby_names = ''   # Asynchronous

#------------------------------------------------------------
# WAL
#------------------------------------------------------------
wal_compression = on
wal_log_hints = on              # Required for pg_rewind
full_page_writes = on
checkpoint_completion_target = 0.9

#------------------------------------------------------------
# MEMORY
#------------------------------------------------------------
shared_buffers = 256MB
effective_cache_size = 1GB
work_mem = 4MB
maintenance_work_mem = 64MB

#------------------------------------------------------------
# LOGGING
#------------------------------------------------------------
log_destination = 'stderr'
logging_collector = on
log_directory = '/var/log/postgresql'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_rotation_age = 1d
log_rotation_size = 100MB
log_min_duration_statement = 1000
log_line_prefix = '%m [%p] %q%u@%d '
log_replication_commands = on
```

### 13.4 primary/pg_hba.conf

```
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     trust

# IPv4 local connections:
host    all             all             127.0.0.1/32            trust
host    all             all             172.20.0.0/24           md5

# Replication connections:
local   replication     all                                     trust
host    replication     all             127.0.0.1/32            trust
host    replication     replicator      172.20.0.0/24           md5

# IPv6 connections
host    all             all             ::1/128                 trust
```

### 13.5 primary/init/01-create-replicator.sql

```sql
-- สร้าง Replication User
CREATE USER replicator 
  WITH REPLICATION 
  LOGIN 
  PASSWORD 'replicator123';

-- ให้สิทธิ์เพิ่มเติมสำหรับ pg_rewind
GRANT EXECUTE ON FUNCTION pg_read_binary_file(text) TO replicator;
GRANT EXECUTE ON FUNCTION pg_read_binary_file(text, bigint, bigint) TO replicator;
GRANT EXECUTE ON FUNCTION pg_read_binary_file(text, bigint, bigint, boolean) TO replicator;
GRANT EXECUTE ON FUNCTION pg_ls_dir(text) TO replicator;
GRANT EXECUTE ON FUNCTION pg_ls_dir(text, boolean, boolean) TO replicator;
GRANT EXECUTE ON FUNCTION pg_stat_file(text) TO replicator;
GRANT EXECUTE ON FUNCTION pg_stat_file(text, boolean) TO replicator;

-- สร้าง Database ทดสอบ
CREATE DATABASE demo;

\c demo

-- สร้าง Table ทดสอบ
CREATE TABLE IF NOT EXISTS demo_data (
    id BIGSERIAL PRIMARY KEY,
    message TEXT NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- สร้าง Index
CREATE INDEX ON demo_data (created_at);

-- Insert ข้อมูลตัวอย่าง
INSERT INTO demo_data (message) 
SELECT 'Initial data row ' || generate_series(1, 100);

GRANT ALL ON ALL TABLES IN SCHEMA public TO postgres;
```

### 13.6 scripts/test-replication.sh

```bash
#!/bin/bash
set -e

echo "===== Testing PostgreSQL Replication ====="

# Colors
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# Wait for services
echo -e "${YELLOW}Waiting for all services to be ready...${NC}"
sleep 10

# Test Primary Connection
echo -e "\n${YELLOW}1. Testing Primary Connection...${NC}"
docker exec pg-primary psql -U postgres -c "SELECT version();" && \
    echo -e "${GREEN}Primary: OK${NC}" || echo -e "${RED}Primary: FAILED${NC}"

# Check if replicas are streaming
echo -e "\n${YELLOW}2. Checking Replication Status...${NC}"
docker exec pg-primary psql -U postgres -c "
SELECT 
    application_name,
    client_addr,
    state,
    sync_state,
    pg_size_pretty(pg_wal_lsn_diff(sent_lsn, replay_lsn)) as lag
FROM pg_stat_replication;
"

# Write to Primary
echo -e "\n${YELLOW}3. Writing to Primary...${NC}"
docker exec pg-primary psql -U postgres -d demo -c "
INSERT INTO demo_data (message) VALUES ('Test message at ' || NOW()::text)
RETURNING id, message, created_at;
"

# Wait for replication
sleep 2

# Read from Replica1
echo -e "\n${YELLOW}4. Reading from Replica1...${NC}"
docker exec pg-replica1 psql -U postgres -d demo -c "
SELECT id, message, created_at 
FROM demo_data 
ORDER BY id DESC 
LIMIT 5;
"

# Read from Replica2
echo -e "\n${YELLOW}5. Reading from Replica2...${NC}"
docker exec pg-replica2 psql -U postgres -d demo -c "
SELECT id, message, created_at 
FROM demo_data 
ORDER BY id DESC 
LIMIT 5;
"

# Check Replication Lag
echo -e "\n${YELLOW}6. Checking Replication Lag...${NC}"
docker exec pg-replica1 psql -U postgres -c "
SELECT 
    pg_is_in_recovery() as is_replica,
    pg_last_wal_receive_lsn() as received_lsn,
    pg_last_wal_replay_lsn() as replayed_lsn,
    NOW() - pg_last_xact_replay_timestamp() as lag;
"

# Test Write Rejection on Replica
echo -e "\n${YELLOW}7. Testing Write Rejection on Replica...${NC}"
docker exec pg-replica1 psql -U postgres -d demo -c "
INSERT INTO demo_data (message) VALUES ('This should fail')
" 2>&1 || echo -e "${GREEN}Write correctly rejected on Replica${NC}"

echo -e "\n${GREEN}===== Replication Test Complete =====${NC}"
```

### 13.7 รันและทดสอบ

```bash
# เริ่มต้น Cluster
docker-compose up -d

# ดู Logs
docker-compose logs -f

# รอให้ทุก Service พร้อม
sleep 30

# รัน Test Script
bash scripts/test-replication.sh

# ดู Replication Status
docker exec pg-primary psql -U postgres -c "SELECT * FROM pg_stat_replication;"

# ทดสอบ Failover
echo "=== Testing Failover ==="

# หยุด Primary
docker stop pg-primary

# Promote Replica1 เป็น Primary
docker exec pg-replica1 psql -U postgres -c "SELECT pg_promote();"

# ตรวจสอบ
docker exec pg-replica1 psql -U postgres -c "SELECT pg_is_in_recovery();"
# ควรได้ f (ไม่ใช่ Replica แล้ว)

# Write ไปยัง New Primary
docker exec pg-replica1 psql -U postgres -d demo -c "
INSERT INTO demo_data (message) VALUES ('Written to new primary');
"

# ทำความสะอาด
docker-compose down -v
```

---

## 14. Logical Replication: Publisher-Subscriber

### 14.1 ตั้งค่า Publisher (Primary)

```sql
-- postgresql.conf ต้องมี: wal_level = logical

-- สร้าง Publication
CREATE PUBLICATION my_pub FOR TABLE products, orders;

-- หรือ Publish ทุก Table
CREATE PUBLICATION all_tables_pub FOR ALL TABLES;

-- ดู Publications
SELECT * FROM pg_publication;
SELECT * FROM pg_publication_tables;
```

### 14.2 ตั้งค่า Subscriber (Replica)

```sql
-- สร้าง Table เหมือนกันบน Replica (ต้อง Schema เหมือนกัน)
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    price DECIMAL(10,2),
    created_at TIMESTAMP DEFAULT NOW()
);

-- สร้าง Subscription
CREATE SUBSCRIPTION my_sub 
  CONNECTION 'host=primary_ip dbname=myapp user=replicator password=xxx'
  PUBLICATION my_pub;

-- ดู Subscriptions
SELECT * FROM pg_subscription;
SELECT * FROM pg_stat_subscription;
```

---

## 15. Monitoring Script

```bash
#!/bin/bash
# monitor-replication.sh

PSQL="psql -U postgres -t -A"
PRIMARY_HOST="localhost"
ALERT_LAG_BYTES=104857600  # 100MB

echo "=== PostgreSQL Replication Monitor ==="
echo "Time: $(date)"
echo ""

# Primary Check
echo "--- Primary Status ---"
$PSQL -h $PRIMARY_HOST -c "
SELECT 
    'primary' as server,
    pg_is_in_recovery() as is_replica,
    pg_current_wal_lsn() as current_lsn;
"

# Replication Status
echo ""
echo "--- Connected Replicas ---"
$PSQL -h $PRIMARY_HOST -c "
SELECT 
    application_name,
    client_addr::text,
    state,
    sync_state,
    CASE 
        WHEN replay_lsn IS NULL THEN 'unknown'
        ELSE pg_size_pretty(pg_wal_lsn_diff(sent_lsn, replay_lsn))
    END as replay_lag,
    COALESCE(replay_lag::text, '0') as time_lag
FROM pg_stat_replication
ORDER BY application_name;
"

# Alert if lag > threshold
LAG=$($PSQL -h $PRIMARY_HOST -c "
SELECT COALESCE(MAX(pg_wal_lsn_diff(sent_lsn, replay_lsn)), 0) 
FROM pg_stat_replication;
" 2>/dev/null)

if [ "$LAG" -gt "$ALERT_LAG_BYTES" 2>/dev/null ]; then
    echo "ALERT: Replication lag exceeds threshold!"
    echo "Current lag: $(numfmt --to=iec $LAG)"
fi
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Physical vs Logical Replication** - ความแตกต่างและการใช้งาน
2. **WAL** - กลไกหัวใจของ PostgreSQL Replication
3. **Primary Configuration** - postgresql.conf, pg_hba.conf, replication user
4. **Replica Setup** - pg_basebackup, standby.signal, postgresql.conf
5. **Monitoring** - pg_stat_replication, lag detection
6. **Sync vs Async** - ข้อดีข้อเสียและการตั้งค่า
7. **Replication Slots** - ป้องกัน WAL ถูกลบ
8. **Cascading Replication** - Replica of Replica
9. **Hot Standby** - Read queries บน Replica
10. **Failover** - การ Promote Replica
11. **pg_rewind** - Sync หลัง Failover
12. **Docker Compose** - Primary + 2 Replicas ที่ใช้งานได้จริง

ในบทถัดไปเราจะเรียนรู้ Connection Pooling ด้วย PgBouncer เพื่อจัดการ Connection อย่างมีประสิทธิภาพ

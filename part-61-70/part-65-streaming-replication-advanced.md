# Part 65: Streaming Replication Advanced

## บทนำ: Streaming Replication Architecture

Streaming Replication คือกลไกหลักของ PostgreSQL High Availability ที่ส่ง WAL (Write-Ahead Log) จาก primary ไปยัง replica แบบ real-time

```
┌──────────────────────────────────────────────────────────┐
│                         PRIMARY                           │
│                                                           │
│  Client → Write → WAL Buffer → WAL Segments              │
│                                    ↓                      │
│                               WAL Sender Process          │
│                                    ↓                      │
└────────────────────────────────────|─────────────────────┘
                                     │ TCP/IP (WAL stream)
                              ┌──────↓──────┐
                              │  REPLICA    │
                              │             │
                              │ WAL Receiver│
                              │      ↓      │
                              │ WAL Segments│
                              │      ↓      │
                              │  Startup    │
                              │  Recovery   │
                              │  Process    │
                              └─────────────┘
```

### WAL Sender ↔ WAL Receiver Protocol

```
Primary WAL Sender:
1. Client requests replication connection
2. Authentication check
3. Send server identification
4. Negotiate start LSN
5. Continuously stream WAL records

Replica WAL Receiver:
1. Connect to primary
2. Send START_REPLICATION command
3. Receive WAL records
4. Write to local WAL files
5. Send status updates (feedback)
6. Startup Recovery Process applies WAL
```

---

## Replication Setup: Primary Configuration

### postgresql.conf ของ Primary

```ini
# ============================================
# PRIMARY CONFIGURATION
# ============================================

# WAL Settings
wal_level = replica               # minimum for streaming replication
max_wal_senders = 10              # maximum WAL sender processes
wal_keep_size = 1GB               # retain WAL for replicas (no slots)

# Replication Authentication
# ต้องเพิ่มใน pg_hba.conf:
# host replication replicator 10.0.0.0/24 scram-sha-256

# Synchronous Replication (optional)
# synchronous_standby_names = 'replica1'  # หรือ 'ANY 1 (replica1, replica2)'

# Connection Settings
wal_sender_timeout = 60s          # timeout for inactive sender

# Monitoring
track_commit_timestamp = on       # สำหรับ tracking
```

### pg_hba.conf ของ Primary

```
# TYPE  DATABASE    USER        ADDRESS         METHOD
host    replication replicator  10.0.0.0/24     scram-sha-256
host    replication replicator  10.0.1.0/24     scram-sha-256
```

### สร้าง Replication User

```sql
-- สร้าง user สำหรับ replication
CREATE ROLE replicator REPLICATION LOGIN PASSWORD 'secure_password';

-- ดู replication users
SELECT usename, userepl FROM pg_user WHERE userepl = true;
```

---

## Replication Setup: Replica Configuration

### สร้าง Base Backup

```bash
# วิธีที่ 1: pg_basebackup (แนะนำ)
pg_basebackup \
    -h primary-server \
    -U replicator \
    -D /var/lib/postgresql/data/ \
    -R \                    # สร้าง standby.signal และ primary_conninfo
    --checkpoint=fast \     # immediate checkpoint
    --wal-method=stream \   # stream WAL during backup
    -P \                    # show progress
    -v                      # verbose

# ตัวเลือก -R สร้างไฟล์เหล่านี้อัตโนมัติ:
# standby.signal (ว่า instance นี้เป็น standby)
# postgresql.auto.conf: primary_conninfo = ...

# วิธีที่ 2: manual base backup
psql -h primary-server -U replicator \
    -c "SELECT pg_start_backup('replica_backup', true, false);"

rsync -av --exclude pg_wal --exclude postmaster.pid \
    /var/lib/postgresql/data/ \
    replica:/var/lib/postgresql/data/

psql -h primary-server -U replicator \
    -c "SELECT * FROM pg_stop_backup(false, true);"
```

### postgresql.conf ของ Replica

```ini
# ============================================
# REPLICA CONFIGURATION
# ============================================

# Hot standby: allow read queries
hot_standby = on

# Connection to primary
primary_conninfo = 'host=primary-server port=5432 user=replicator password=secure_password application_name=replica1'

# Slot (optional)
# primary_slot_name = 'replica1_slot'

# Feedback to primary
hot_standby_feedback = on         # ป้องกัน query conflicts

# Delay (optional, for protection)
# recovery_min_apply_delay = '15min'

# Read replica settings
max_standby_streaming_delay = 30s  # wait before canceling conflicting queries
max_standby_archive_delay = 30s    # same for archive recovery
```

### standby.signal

```bash
# ไฟล์ standby.signal บอก PostgreSQL ว่าให้ start เป็น standby
touch /var/lib/postgresql/data/standby.signal

# Start replica
pg_ctl start -D /var/lib/postgresql/data/

# ตรวจสอบ replication status
psql -c "SELECT pg_is_in_recovery();"  -- ควรได้ t (true)
psql -c "SELECT pg_last_wal_replay_lsn();"
```

---

## Replication Modes

### Asynchronous Replication (Default)

```sql
-- Primary ไม่รอ replica acknowledge
-- Potential data loss: WAL ที่ยังไม่ส่งเมื่อ primary crash

-- ดู asynchronous state
SELECT 
    application_name,
    sync_state,            -- async
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn
FROM pg_stat_replication;

-- Lag calculation
SELECT 
    application_name,
    pg_size_pretty(pg_wal_lsn_diff(sent_lsn, replay_lsn)) as total_lag,
    replay_lag as time_lag
FROM pg_stat_replication;
```

### Synchronous Replication

```sql
-- Primary รอให้ replica ตอบกลับก่อน COMMIT return

-- ตั้งค่าบน primary
ALTER SYSTEM SET synchronous_standby_names = 'replica1';
SELECT pg_reload_conf();

-- ดู sync state
SELECT application_name, sync_state 
FROM pg_stat_replication;
-- sync_state: sync (confirmed), potential, async, quorum

-- synchronous_commit ระดับต่างๆ กับ sync replication
-- on (default): รอ flush บน replica
-- remote_apply: รอ apply บน replica (ช้ากว่า แต่ read-your-writes)
ALTER SYSTEM SET synchronous_commit = 'remote_apply';
```

### Quorum-Based Synchronous Replication

```sql
-- ANY N (replica1, replica2, ...): รอ N ใดก็ได้จาก list
-- ให้ความยืดหยุ่น: ถ้า replica หนึ่งช้า ใช้อีกตัวแทน

-- ตัวอย่าง: รอ replica ใดก็ได้ 1 ตัวจาก 3
ALTER SYSTEM SET synchronous_standby_names = 'ANY 1 (replica1, replica2, replica3)';
SELECT pg_reload_conf();

-- ตัวอย่าง: FIRST mode - เรียงตามลำดับ
-- replica1 ต้องตอบ, ถ้าไม่ available ใช้ replica2
ALTER SYSTEM SET synchronous_standby_names = 'FIRST 1 (replica1, replica2)';

-- ตัวอย่าง: รอ 2 จาก 3
ALTER SYSTEM SET synchronous_standby_names = 'ANY 2 (replica1, replica2, replica3)';

-- ดู sync states
SELECT 
    application_name,
    sync_priority,   -- ลำดับ priority ใน FIRST mode
    sync_state       -- sync, potential, async, quorum
FROM pg_stat_replication;
```

### synchronous_standby_names Options สรุป

```
1. 'replica1'
   - Single synchronous standby
   - replica1 ต้อง respond ก่อน COMMIT
   
2. 'replica1, replica2'
   - FIRST 1 (implicit)
   - replica1 เป็น primary sync, replica2 เป็น backup
   
3. 'FIRST 2 (r1, r2, r3)'
   - รอ 2 ตัวแรกจาก list ตามลำดับ
   - r1+r2 หรือ r1+r3 หรือ r2+r3 ตามที่ available
   
4. 'ANY 1 (r1, r2, r3)'
   - Quorum: รอใดก็ได้ 1 ตัว
   - ดีที่สุดสำหรับ HA

5. 'ANY 2 (r1, r2, r3)'
   - รอ 2 ตัวใดก็ได้
   - Stronger durability
   
6. '*'
   - รอ ALL replicas (ช้ามาก)
```

---

## Replication Lag Monitoring

### pg_stat_replication: Primary View

```sql
-- ดู lag แบบ bytes
SELECT 
    pid,
    usename,
    application_name,
    client_addr,
    state,
    -- WAL positions
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    -- Lag in bytes
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), sent_lsn)) as send_lag,
    pg_size_pretty(pg_wal_lsn_diff(sent_lsn, write_lsn)) as write_lag,
    pg_size_pretty(pg_wal_lsn_diff(write_lsn, flush_lsn)) as flush_lag,
    pg_size_pretty(pg_wal_lsn_diff(flush_lsn, replay_lsn)) as replay_lag,
    -- Lag in time
    write_lag as write_lag_time,
    flush_lag as flush_lag_time,
    replay_lag as replay_lag_time,
    -- Sync state
    sync_state
FROM pg_stat_replication
ORDER BY application_name;

-- ดู total lag
SELECT 
    application_name,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn)) as total_lag_bytes,
    CASE 
        WHEN replay_lag > INTERVAL '5 minutes' THEN 'CRITICAL'
        WHEN replay_lag > INTERVAL '1 minute' THEN 'WARNING'
        ELSE 'OK'
    END as lag_status,
    replay_lag
FROM pg_stat_replication;
```

### pg_wal_lsn_diff(): คำนวณ Lag

```sql
-- pg_wal_lsn_diff(lsn1, lsn2): คืนค่า bytes ระหว่าง 2 LSN

-- LSN ของ primary
SELECT pg_current_wal_lsn() as primary_lsn;

-- บน replica: ดู LSN ที่ apply แล้ว
SELECT pg_last_wal_replay_lsn() as replica_lsn;

-- คำนวณ lag จาก primary
-- (ต้องเชื่อมต่อทั้ง primary และ replica)
WITH 
    primary_lsn AS (
        SELECT pg_current_wal_lsn() as lsn
    ),
    replica_lsn AS (
        SELECT replay_lsn as lsn 
        FROM pg_stat_replication 
        WHERE application_name = 'replica1'
    )
SELECT 
    pg_size_pretty(pg_wal_lsn_diff(p.lsn, r.lsn)) as lag_bytes,
    EXTRACT(EPOCH FROM replay_lag) as lag_seconds
FROM primary_lsn p, replica_lsn r, pg_stat_replication;
```

### pg_stat_wal_receiver: Replica View

```sql
-- บน replica: ดู WAL receiver status
SELECT 
    pid,
    status,
    receive_start_lsn,
    received_lsn,
    last_msg_send_time,
    last_msg_receipt_time,
    latest_end_lsn,
    latest_end_time,
    EXTRACT(EPOCH FROM (NOW() - last_msg_receipt_time)) as seconds_since_last_msg,
    conninfo
FROM pg_stat_wal_receiver;

-- ดู replication slot ที่ใช้
SELECT 
    slot_name,
    received_lsn,
    last_msg_send_time,
    last_msg_receipt_time
FROM pg_stat_wal_receiver;
```

---

## Replication Slots: Physical และ Logical

### Physical Replication Slots

```sql
-- สร้าง physical slot บน primary
SELECT pg_create_physical_replication_slot('replica1_slot');

-- ดู slots ทั้งหมด
SELECT 
    slot_name,
    slot_type,
    datoid,
    database,
    temporary,
    active,
    active_pid,
    xmin,
    catalog_xmin,
    restart_lsn,
    confirmed_flush_lsn,
    wal_status,
    safe_wal_size,
    two_phase
FROM pg_replication_slots;

-- wal_status:
-- reserved: slot is streaming or has enough WAL
-- extended: WAL extended beyond wal_keep_size
-- unreserved: WAL may be removed soon
-- lost: WAL already removed (slot invalid)

-- Replica ใช้ slot
-- ใน postgresql.conf ของ replica:
-- primary_slot_name = 'replica1_slot'
```

### ประโยชน์และความเสี่ยงของ Slots

```
ประโยชน์:
- Primary เก็บ WAL จนกว่า slot consumer จะรับ
- Replica ไม่พลาด WAL แม้ restart
- ไม่ต้องตั้ง wal_keep_size สูงๆ

ความเสี่ยง:
- ถ้า replica หยุดทำงาน: WAL สะสมไม่หยุด
- Disk full = PostgreSQL หยุดทำงาน!

การป้องกัน:
- ตั้ง max_slot_wal_keep_size
- Monitor inactive slots
- Alert ถ้า retained WAL > threshold
```

### Script Monitor Replication Slots

```sql
-- Alert script สำหรับ replication slots
WITH slot_details AS (
    SELECT 
        slot_name,
        slot_type,
        active,
        wal_status,
        pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) as retained_bytes,
        CASE 
            WHEN NOT active AND wal_status = 'lost' THEN 'INVALID'
            WHEN NOT active AND pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) > 10 * 1024^3 THEN 'CRITICAL'
            WHEN NOT active AND pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) > 5 * 1024^3 THEN 'WARNING'
            WHEN active THEN 'OK'
            ELSE 'INACTIVE'
        END as status
    FROM pg_replication_slots
)
SELECT 
    slot_name,
    slot_type,
    active,
    wal_status,
    pg_size_pretty(retained_bytes) as retained_wal,
    status
FROM slot_details
ORDER BY retained_bytes DESC;
```

---

## Hot Standby: Queries on Replica

### Configuration

```sql
-- เปิด hot_standby บน replica
-- ใน postgresql.conf:
hot_standby = on  -- default: on (PostgreSQL 10+)

-- ตรวจสอบ
SELECT pg_is_in_recovery();  -- true = เป็น standby

-- บน replica สามารถ run read queries
SELECT * FROM orders WHERE status = 'pending' LIMIT 10;

-- ไม่สามารถทำ write operations
INSERT INTO test VALUES (1);
-- ERROR: cannot execute INSERT in a read-only transaction
```

### hot_standby_feedback: ป้องกัน Query Conflicts

```sql
-- ปัญหา: PRIMARY vacuum rows ที่ REPLICA กำลัง query
-- ผลลัพธ์: "ERROR: canceling statement due to conflict with recovery"

-- hot_standby_feedback = on:
-- Replica แจ้ง primary ถึง oldest XID ที่ใช้อยู่
-- Primary จะไม่ vacuum rows ที่ replica ต้องการ

-- ข้อเสีย: primary อาจมี bloat เพิ่มขึ้น
-- เพราะ VACUUM ไม่สามารถลบ rows บางส่วนได้

-- ตั้งค่าบน replica
ALTER SYSTEM SET hot_standby_feedback = ON;
SELECT pg_reload_conf();

-- ดูผล: primary จะเก็บ xmin ไว้สูงขึ้น
SELECT 
    application_name,
    backend_xmin,  -- oldest XID reported by replica
    backend_xmin IS NOT NULL as sends_feedback
FROM pg_stat_replication;
```

### max_standby_streaming_delay

```sql
-- เมื่อ replica ต้องการ apply WAL record ที่ conflict กับ query ที่รันอยู่:
-- รอ max_standby_streaming_delay วินาที ก่อน cancel query

ALTER SYSTEM SET max_standby_streaming_delay = '30s';  -- default: 30s
ALTER SYSTEM SET max_standby_archive_delay = '30s';   -- สำหรับ archive recovery

-- 0 = cancel ทันที (query บน replica ถูก interrupt บ่อย)
-- -1 = รอไม่จำกัด (replica อาจล้าหลังมาก)

-- ดู conflicts
SELECT 
    datname,
    confl_tablespace,
    confl_lock,
    confl_snapshot,   -- snapshot conflicts (most common)
    confl_bufferpin,
    confl_deadlock
FROM pg_stat_database_conflicts;
```

### Conflict Resolution

```sql
-- ประเภทของ conflicts บน replica:

-- 1. Snapshot Conflict:
-- Cause: VACUUM บน primary ลบ rows ที่ replica ยังต้องการ
-- Solution: hot_standby_feedback = on

-- 2. Lock Conflict:
-- Cause: DDL บน primary (ACCESS EXCLUSIVE) conflict กับ query
-- Solution: ยากจะหลีกเลี่ยง

-- 3. Buffer Pin Conflict:
-- Cause: standby กำลัง read page ที่ต้องการ update
-- Solution: rare, เพิ่ม max_standby_streaming_delay

-- ดู active conflicts ปัจจุบัน
SELECT 
    pid,
    query,
    wait_event_type,
    wait_event,
    query_start
FROM pg_stat_activity
WHERE wait_event_type = 'Lock'
    AND datname = current_database();
```

---

## Cascading Replication

### Architecture

```
Primary → Replica 1 → Replica 2 (Cascaded)
                    → Replica 3 (Cascaded)
```

### Configuration

```ini
# Replica 1 (ส่ง WAL ต่อให้ Replica 2, 3)
hot_standby = on
max_wal_senders = 5      # ต้องตั้งเพื่อรับ downstream replicas
wal_level = replica      # ต้องตั้งเพื่อส่ง WAL ต่อ

# ใน pg_hba.conf ของ Replica 1:
# host replication replicator 10.0.0.0/24 scram-sha-256
```

```ini
# Replica 2 (cascaded from Replica 1)
primary_conninfo = 'host=replica1 port=5432 user=replicator ...'
recovery_target_timeline = 'latest'  # follow timeline changes
hot_standby = on
```

### Timeline History ใน Cascading

```sql
-- บน Replica 2: ดู timeline
SELECT * FROM pg_control_checkpoint();
-- timeline_id: current timeline

-- recovery_target_timeline = 'latest':
-- ตาม timeline ล่าสุด (สำคัญมากเมื่อ primary เปลี่ยน)
-- ถ้า Replica 1 promote เป็น primary ใหม่:
-- Replica 2 ต้อง follow timeline ใหม่ของ Replica 1

-- ดู timeline history
psql -c "SELECT * FROM pg_timeline();"  -- หรือ
pg_waldump --timeline 2 $PGDATA/pg_wal/
```

---

## Delayed Replica: Protection Against Application Errors

### Configuration

```ini
# Delayed replica: apply WAL หลังจาก delay
# ป้องกัน: accidental DROP TABLE, bulk delete, etc.

recovery_min_apply_delay = '15min'  # delay 15 นาที
hot_standby = on
```

### ใช้งาน Delayed Replica

```sql
-- ตรวจสอบ current delay
SELECT 
    pg_last_wal_replay_lsn() as replay_lsn,
    pg_last_xact_replay_timestamp() as last_replay_time,
    NOW() - pg_last_xact_replay_timestamp() as current_delay,
    pg_is_in_recovery() as is_standby;

-- เมื่อ application ทำ mistake:
-- 1. หยุด delay บน delayed replica
-- 2. ปล่อยให้ apply จนถึง point ก่อน mistake

-- Promote delayed replica (ถ้าต้องการ)
-- ขั้นตอน:
-- 1. ปิด application (ป้องกัน writes ใหม่)
-- 2. หยุด primary replication
-- 3. Promote delayed replica
pg_ctl promote -D /var/lib/postgresql/data/

-- หรือ
psql -c "SELECT pg_promote();"
```

### Delayed Replica Use Cases

```
1. Accidental data deletion:
   - Developer DELETE FROM orders WHERE id IN (...)
   - แต่ลืม WHERE clause
   - มี 15 นาทีเพื่อ detect และ recover

2. Bad migrations:
   - Migration ทำให้ data corrupt
   - Delayed replica ยังมีข้อมูลที่ถูกต้อง
   
3. Ransomware protection:
   - ถ้า ransomware encrypt data
   - Delayed replica ยังไม่ได้ apply changes
```

---

## pg_rewind: Resync After Divergence

### เมื่อไหร่ต้องใช้ pg_rewind

```
Scenario:
1. Primary crash
2. Replica 1 promoted เป็น primary ใหม่
3. Old primary กลับมา online

ปัญหา:
- Old primary มี WAL ที่ไม่เคยถูก apply บน new primary
- สองฝั่ง "diverge" (timeline แตกต่าง)
- Old primary ไม่สามารถ rejoin เป็น replica ได้

Solution: pg_rewind
- Rewind old primary ให้กลับไปอยู่บน timeline ใหม่
- ไม่ต้อง pg_basebackup ใหม่ทั้งหมด (เร็วกว่า)
```

### pg_rewind Usage

```bash
# 1. ตรวจสอบ old primary หยุดทำงาน
pg_ctl status -D /var/lib/postgresql/data/

# 2. Run pg_rewind
pg_rewind \
    --target-pgdata=/var/lib/postgresql/data/ \
    --source-server="host=new-primary port=5432 user=postgres dbname=postgres" \
    -P \        # show progress
    -n          # dry run ก่อน (ดูว่าจะทำอะไร)

# 3. ถ้า dry run โอเค, run จริง
pg_rewind \
    --target-pgdata=/var/lib/postgresql/data/ \
    --source-server="host=new-primary port=5432 user=postgres dbname=postgres" \
    -P

# 4. ตั้งค่า recovery
cat >> /var/lib/postgresql/data/postgresql.auto.conf << 'EOF'
primary_conninfo = 'host=new-primary port=5432 user=replicator'
recovery_target_timeline = 'latest'
EOF

touch /var/lib/postgresql/data/standby.signal

# 5. Start เป็น standby
pg_ctl start -D /var/lib/postgresql/data/
```

### pg_rewind Prerequisites

```sql
-- Primary ต้องตั้งค่า:
SHOW wal_log_hints;     -- ต้องเป็น on (หรือ data_checksums)
SHOW full_page_writes;  -- ต้องเป็น on

-- หรือ enable data checksums ตอน initdb
initdb -D /data --data-checksums

-- ตรวจสอบ
SELECT current_setting('wal_log_hints');
SELECT current_setting('data_checksums');
```

---

## Timeline History

### PostgreSQL Timelines

```
Timeline = ประวัติ "version" ของ cluster

Timeline 1: Original primary
            WAL: 000000010000000000000001 ...
            
Failover occurred:
Timeline 2: New primary (promoted from replica)
            WAL: 000000020000000000000001 ...
            
Recovery target:
- replica ต้องรู้ว่าต้อง follow timeline ใด
```

```bash
# ดู timeline history
psql -c "SELECT * FROM pg_timeline();"

# ดู timeline files ใน pg_wal
ls $PGDATA/pg_wal/*.history

# เนื้อหา timeline history file
cat $PGDATA/pg_wal/00000002.history
# 1	0/5000000	no recovery target specified
# (เปลี่ยนจาก timeline 1 ที่ LSN 0/5000000)
```

### recovery_target_timeline

```ini
# ใน postgresql.conf ของ replica:

# latest: ตาม timeline ล่าสุดของ primary
recovery_target_timeline = 'latest'  # แนะนำ

# specific: follow timeline ที่กำหนด
recovery_target_timeline = 2

# current: ไม่เปลี่ยน timeline (ไม่แนะนำสำหรับ streaming)
recovery_target_timeline = 'current'
```

---

## Failover และ Switchover

### Manual Failover

```bash
# Primary crash scenario:
# 1. ตรวจสอบว่า primary ตาย
pg_ctl status -D /var/lib/postgresql/data/  # บน primary

# 2. ตรวจสอบ replica lag
psql -h replica -c "
    SELECT 
        pg_last_wal_replay_lsn() as replay_lsn,
        pg_last_xact_replay_timestamp() as last_replay
;"

# 3. Promote replica
pg_ctl promote -D /var/lib/postgresql/data/  # บน replica

# หรือ
psql -h replica -c "SELECT pg_promote();"

# 4. ตรวจสอบ promotion
psql -h new-primary -c "SELECT pg_is_in_recovery();"  -- ควร return false

# 5. Update application connection string
# ชี้ไปยัง new primary
```

### Graceful Switchover (Planned)

```bash
# สลับ primary/replica แบบ no-data-loss

# 1. ตรวจสอบ replica ตามทัน
psql -h primary -c "
SELECT 
    application_name,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn)) as lag
FROM pg_stat_replication;
"

# 2. ป้องกัน writes ใหม่ (optional แต่ safe)
# ปิด application หรือใช้ pg_reload_conf + pause

# 3. รอ replica ตามทัน (lag = 0)
while true; do
    lag=$(psql -h primary -t -c "
        SELECT pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn)
        FROM pg_stat_replication WHERE application_name = 'replica1';
    ")
    if [ "$lag" = " 0" ]; then
        echo "Replica caught up!"
        break
    fi
    echo "Lag: $lag bytes, waiting..."
    sleep 1
done

# 4. Promote replica
pg_ctl promote -D /var/lib/postgresql/data-replica/

# 5. Demote old primary เป็น replica
pg_rewind --target-pgdata=/var/lib/postgresql/data-primary/ \
    --source-server="host=new-primary ..."

touch /var/lib/postgresql/data-primary/standby.signal
pg_ctl start -D /var/lib/postgresql/data-primary/
```

---

## Replication Monitoring Dashboard

### Comprehensive Monitoring Query

```sql
-- Primary: สถานะ replication ทั้งหมด
WITH replication_overview AS (
    SELECT 
        application_name,
        client_addr,
        sync_state,
        state,
        -- LSN info
        sent_lsn,
        write_lsn,
        flush_lsn,
        replay_lsn,
        -- Lag bytes
        pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) as lag_bytes,
        -- Lag time
        replay_lag,
        write_lag,
        flush_lag,
        -- Slot info
        rs.slot_name,
        rs.wal_status as slot_wal_status,
        rs.safe_wal_size
    FROM pg_stat_replication r
    LEFT JOIN pg_replication_slots rs 
        ON rs.active_pid = r.pid
)
SELECT 
    application_name,
    client_addr,
    sync_state,
    state,
    pg_size_pretty(lag_bytes) as total_lag_bytes,
    replay_lag as lag_time,
    CASE 
        WHEN lag_bytes > 1 * 1024^3 THEN '🔴 CRITICAL'
        WHEN lag_bytes > 100 * 1024^2 THEN '🟡 WARNING'
        WHEN replay_lag > INTERVAL '5 minutes' THEN '🔴 TIME LAG CRITICAL'
        WHEN replay_lag > INTERVAL '1 minute' THEN '🟡 TIME LAG WARNING'
        ELSE '🟢 OK'
    END as status,
    slot_name,
    slot_wal_status
FROM replication_overview
ORDER BY lag_bytes DESC;

-- Replica: ดู receive status
SELECT 
    pg_is_in_recovery() as is_replica,
    pg_last_wal_receive_lsn() as receive_lsn,
    pg_last_wal_replay_lsn() as replay_lsn,
    pg_last_xact_replay_timestamp() as last_replay_time,
    EXTRACT(EPOCH FROM (NOW() - pg_last_xact_replay_timestamp())) as lag_seconds,
    pg_is_wal_replay_paused() as replay_paused;
```

---

## Automated Monitoring Script

```python
#!/usr/bin/env python3
"""Streaming Replication Health Monitor"""

import psycopg2
import time
import smtplib
from email.mime.text import MIMEText
from typing import Dict, List, Optional
from dataclasses import dataclass

@dataclass
class ReplicaStatus:
    application_name: str
    client_addr: str
    sync_state: str
    state: str
    lag_bytes: int
    lag_seconds: Optional[float]
    is_healthy: bool
    issues: List[str]

def get_replication_status(primary_conn_str: str) -> List[ReplicaStatus]:
    """ดึง replication status จาก primary"""
    
    conn = psycopg2.connect(primary_conn_str)
    cur = conn.cursor()
    
    cur.execute("""
        SELECT 
            application_name,
            client_addr::text,
            sync_state,
            state,
            pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) as lag_bytes,
            EXTRACT(EPOCH FROM replay_lag) as lag_seconds
        FROM pg_stat_replication
    """)
    
    replicas = []
    for row in cur.fetchall():
        app_name, client, sync_state, state, lag_bytes, lag_seconds = row
        
        issues = []
        is_healthy = True
        
        # Check lag
        if lag_bytes and lag_bytes > 1 * 1024**3:  # 1GB
            issues.append(f"CRITICAL: Replication lag {lag_bytes/(1024**3):.2f}GB")
            is_healthy = False
        elif lag_bytes and lag_bytes > 100 * 1024**2:  # 100MB
            issues.append(f"WARNING: Replication lag {lag_bytes/(1024**2):.1f}MB")
        
        # Check time lag
        if lag_seconds and lag_seconds > 300:  # 5 minutes
            issues.append(f"CRITICAL: Replay lag {lag_seconds:.0f}s (>{5*60}s)")
            is_healthy = False
        elif lag_seconds and lag_seconds > 60:
            issues.append(f"WARNING: Replay lag {lag_seconds:.0f}s (>60s)")
        
        # Check state
        if state != 'streaming':
            issues.append(f"WARNING: State is '{state}', expected 'streaming'")
        
        replicas.append(ReplicaStatus(
            application_name=app_name,
            client_addr=str(client),
            sync_state=sync_state,
            state=state,
            lag_bytes=lag_bytes or 0,
            lag_seconds=lag_seconds,
            is_healthy=is_healthy,
            issues=issues
        ))
    
    cur.close()
    conn.close()
    
    return replicas

def check_replication_slots(primary_conn_str: str) -> List[Dict]:
    """ตรวจสอบ replication slots"""
    
    conn = psycopg2.connect(primary_conn_str)
    cur = conn.cursor()
    
    cur.execute("""
        SELECT 
            slot_name,
            slot_type,
            active,
            wal_status,
            pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) as retained_bytes
        FROM pg_replication_slots
    """)
    
    slots = []
    for row in cur.fetchall():
        slot_name, slot_type, active, wal_status, retained_bytes = row
        
        issues = []
        
        if not active:
            issues.append(f"Slot '{slot_name}' is INACTIVE")
            
            if retained_bytes and retained_bytes > 5 * 1024**3:  # 5GB
                issues.append(f"Retaining {retained_bytes/(1024**3):.1f}GB of WAL")
        
        if wal_status in ('unreserved', 'lost'):
            issues.append(f"Slot WAL status: {wal_status}")
        
        slots.append({
            'name': slot_name,
            'type': slot_type,
            'active': active,
            'wal_status': wal_status,
            'retained_gb': (retained_bytes or 0) / (1024**3),
            'issues': issues
        })
    
    cur.close()
    conn.close()
    
    return slots

def send_alert(message: str, smtp_host: str, to_email: str):
    """ส่ง email alert"""
    msg = MIMEText(message)
    msg['Subject'] = 'PostgreSQL Replication Alert'
    msg['From'] = 'monitoring@example.com'
    msg['To'] = to_email
    
    try:
        with smtplib.SMTP(smtp_host) as server:
            server.sendmail(msg['From'], [to_email], msg.as_string())
    except Exception as e:
        print(f"Failed to send alert: {e}")

def monitor_replication(
    primary_conn_str: str,
    check_interval: int = 30,
    alert_email: Optional[str] = None
):
    """Main monitoring loop"""
    
    print(f"Starting replication monitor (interval: {check_interval}s)")
    
    previous_unhealthy = set()
    
    while True:
        try:
            # Check replicas
            replicas = get_replication_status(primary_conn_str)
            slots = check_replication_slots(primary_conn_str)
            
            current_unhealthy = set()
            all_issues = []
            
            # Print status
            print(f"\n{'='*60}")
            print(f"Replication Status at {time.strftime('%Y-%m-%d %H:%M:%S')}")
            print(f"{'='*60}")
            
            if not replicas:
                print("WARNING: No replicas connected!")
                all_issues.append("No replicas connected to primary")
            
            for replica in replicas:
                status_icon = "✓" if replica.is_healthy else "✗"
                lag_mb = replica.lag_bytes / (1024**2)
                
                print(f"{status_icon} {replica.application_name} ({replica.client_addr})")
                print(f"  State: {replica.state}, Sync: {replica.sync_state}")
                print(f"  Lag: {lag_mb:.1f}MB / {replica.lag_seconds or 0:.1f}s")
                
                if replica.issues:
                    for issue in replica.issues:
                        print(f"  ⚠ {issue}")
                        all_issues.append(f"{replica.application_name}: {issue}")
                    
                    if not replica.is_healthy:
                        current_unhealthy.add(replica.application_name)
            
            # Print slot status
            for slot in slots:
                if slot['issues']:
                    print(f"\nSlot '{slot['name']}': {', '.join(slot['issues'])}")
                    all_issues.extend(slot['issues'])
            
            # Send alerts for new issues
            new_unhealthy = current_unhealthy - previous_unhealthy
            if new_unhealthy and alert_email and all_issues:
                alert_msg = f"Replication issues detected:\n\n" + "\n".join(all_issues)
                send_alert(alert_msg, 'localhost', alert_email)
                print(f"Alert sent to {alert_email}")
            
            previous_unhealthy = current_unhealthy
            
        except Exception as e:
            print(f"Monitor error: {e}")
        
        time.sleep(check_interval)

# Prometheus metrics endpoint
def get_prometheus_metrics(primary_conn_str: str) -> str:
    """สร้าง metrics สำหรับ Prometheus"""
    
    replicas = get_replication_status(primary_conn_str)
    slots = check_replication_slots(primary_conn_str)
    
    metrics = []
    
    # Replication lag gauge
    metrics.append("# HELP pg_replication_lag_bytes Replication lag in bytes")
    metrics.append("# TYPE pg_replication_lag_bytes gauge")
    
    for replica in replicas:
        metrics.append(
            f'pg_replication_lag_bytes{{replica="{replica.application_name}"}} {replica.lag_bytes}'
        )
    
    # Replication connected gauge
    metrics.append("# HELP pg_replication_connected Is replica connected")
    metrics.append("# TYPE pg_replication_connected gauge")
    
    for replica in replicas:
        connected = 1 if replica.state == 'streaming' else 0
        metrics.append(
            f'pg_replication_connected{{replica="{replica.application_name}"}} {connected}'
        )
    
    # Slot WAL retained
    metrics.append("# HELP pg_replication_slot_wal_bytes WAL retained by slot")
    metrics.append("# TYPE pg_replication_slot_wal_bytes gauge")
    
    for slot in slots:
        metrics.append(
            f'pg_replication_slot_wal_bytes{{slot="{slot["name"]}"}} {slot["retained_gb"] * 1024**3:.0f}'
        )
    
    return "\n".join(metrics) + "\n"

if __name__ == '__main__':
    import os
    
    conn_str = os.environ.get(
        'PRIMARY_CONN_STR',
        'postgresql://postgres@localhost/postgres'
    )
    
    monitor_replication(
        primary_conn_str=conn_str,
        check_interval=30,
        alert_email=os.environ.get('ALERT_EMAIL')
    )
```

---

## Patroni: Automated Failover

### Configuration ตัวอย่าง

```yaml
# patroni.yml - Automated HA สำหรับ PostgreSQL

scope: postgres-cluster
namespace: /db/
name: node1

restapi:
  listen: 0.0.0.0:8008
  connect_address: 10.0.0.1:8008

etcd3:
  hosts: 10.0.0.10:2379,10.0.0.11:2379,10.0.0.12:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576  # 1MB max lag for failover
    synchronous_mode: false
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        wal_level: replica
        max_wal_senders: 10
        wal_keep_size: 1GB
        hot_standby: "on"
        wal_log_hints: "on"
        max_replication_slots: 10
        
  initdb:
    - encoding: UTF8
    - data-checksums

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 10.0.0.1:5432
  data_dir: /var/lib/postgresql/data
  bin_dir: /usr/lib/postgresql/14/bin
  pgpass: /tmp/pgpass0
  
  authentication:
    replication:
      username: replicator
      password: secure_password
    superuser:
      username: postgres
      password: admin_password
    rewind:
      username: rewind_user
      password: rewind_password
      
  parameters:
    unix_socket_directories: '/var/run/postgresql'
    
  pg_hba:
    - host replication replicator 10.0.0.0/24 md5
    - host all all 0.0.0.0/0 md5

watchdog:
  mode: automatic
  device: /dev/watchdog
  safety_margin: 5

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false
```

```bash
# ตรวจสอบ Patroni cluster status
patronictl -c patroni.yml list

# Output:
# + Cluster: postgres-cluster (12345678901234567) +----+-----------+
# | Member | Host       | Role    | State   | TL | Lag in MB |
# +--------+------------+---------+---------+----+-----------+
# | node1  | 10.0.0.1   | Leader  | running |  1 |           |
# | node2  | 10.0.0.2   | Replica | running |  1 |         0 |
# | node3  | 10.0.0.3   | Replica | running |  1 |         0 |
# +--------+------------+---------+---------+----+-----------+

# Manual failover
patronictl -c patroni.yml failover postgres-cluster

# Switchover (graceful)
patronictl -c patroni.yml switchover postgres-cluster --master node1 --candidate node2

# ดู history
patronictl -c patroni.yml history postgres-cluster
```

---

## สรุป: Streaming Replication Best Practices

```
1. ใช้ Replication Slots อย่างระมัดระวัง:
   - ตั้ง max_slot_wal_keep_size เสมอ
   - Monitor retained WAL ต่อเนื่อง
   - Drop slots ที่ไม่ใช้งาน

2. Synchronous vs Asynchronous:
   - Production critical: ANY 1 (replica1, replica2)
   - Development: async
   - Banking/Finance: synchronous_commit = remote_apply

3. Hot Standby Setup:
   - hot_standby_feedback = on เสมอ
   - ปรับ max_standby_streaming_delay ตาม workload

4. Cascading Replication:
   - ใช้ recovery_target_timeline = 'latest'
   - Monitor ทุก tier

5. Delayed Replica:
   - ตั้ง recovery_min_apply_delay = 15min หรือ 30min
   - ไม่ใช้ hot_standby_feedback บน delayed replica

6. Monitoring สำคัญมาก:
   - Lag bytes และ lag time ทั้งคู่
   - Archive status
   - Slot WAL retention
   - Checkpoint pressure

7. ใช้ Patroni หรือ tools สำหรับ automated failover:
   - Manual failover เสี่ยง human error
   - Patroni + etcd + watchdog = production-grade HA

8. ทดสอบ failover สม่ำเสมอ:
   - Scheduled drill ทุกไตรมาส
   - Document failover runbook
   - ฝึก team ก่อนเกิดเหตุจริง
```

```sql
-- Quick health check query สำหรับใช้ใน monitoring
SELECT 
    CASE 
        WHEN pg_is_in_recovery() THEN 'REPLICA'
        ELSE 'PRIMARY'
    END as role,
    (
        SELECT COUNT(*) 
        FROM pg_stat_replication 
        WHERE state = 'streaming'
    ) as streaming_replicas,
    (
        SELECT COUNT(*) 
        FROM pg_replication_slots 
        WHERE NOT active
    ) as inactive_slots,
    (
        SELECT pg_size_pretty(
            MAX(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn))
        )
        FROM pg_replication_slots
    ) as max_slot_lag,
    (
        SELECT last_archived_wal IS NOT NULL AND last_failed_time < last_archived_time
        FROM pg_stat_archiver
    ) as archive_ok;
```

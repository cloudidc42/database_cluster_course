# Part 64: WAL Configuration

## บทนำ: WAL คืออะไร

**WAL (Write-Ahead Log)** คือหัวใจของ durability ใน PostgreSQL ชื่อมาจากหลักการ "เขียน log ก่อนเสมอ" ก่อนที่จะแก้ไขข้อมูลใน data pages

### ทำไมต้องมี WAL

```
ปัญหาที่ต้องแก้: ความน่าเชื่อถือของข้อมูลเมื่อระบบล้มเหลว

ไม่มี WAL:
Transaction: UPDATE orders SET status = 'paid' WHERE id = 1
  1. อ่าน page จาก disk
  2. แก้ไข page ใน memory
  3. ระบบ crash! ← ข้อมูลสูญหาย

มี WAL:
Transaction: UPDATE orders SET status = 'paid' WHERE id = 1
  1. เขียน WAL record: "update row X to status=paid"
  2. WAL flush to disk (durable)
  3. แก้ไข page ใน memory
  4. ระบบ crash! ← ไม่เป็นไร!
  
Recovery:
  1. PostgreSQL ตรวจสอบว่า WAL มีอะไรบ้างหลัง checkpoint ล่าสุด
  2. Replay WAL records → ข้อมูลถูกต้อง
  3. Database พร้อมใช้งาน
```

### WAL Flow ละเอียด

```
Transaction COMMIT
     ↓
WAL record สร้างใน WAL buffer (ใน RAM)
     ↓
WAL flush to WAL segment file (บน disk)
[fsync: ensure durability]
     ↓
COMMIT acknowledged to client
     ↓
(Background) checkpoint process applies changes to heap pages
```

---

## WAL Segments

### ขนาดและโครงสร้าง

```bash
# WAL segment default size: 16MB
# Location: $PGDATA/pg_wal/

ls -la $PGDATA/pg_wal/
# 000000010000000000000001
# 000000010000000000000002
# 000000010000000000000003
# ...
# ชื่อไฟล์: TL/LogID/SegID (hex)

# ดู current WAL segment
psql -c "SELECT pg_walfile_name(pg_current_wal_lsn());"

# ดู current WAL LSN (Log Sequence Number)
psql -c "SELECT pg_current_wal_lsn();"
# Output: 0/3A789B40

# ดู WAL file จาก LSN
psql -c "SELECT pg_walfile_name('0/3A789B40');"
# Output: 000000010000000000000003

# ดู offset ใน WAL file
psql -c "SELECT * FROM pg_walfile_name_offset(pg_current_wal_lsn());"
# file_name      | file_offset
# ---------------+-------------
# 0000000100...  | 567234
```

### ปรับขนาด WAL Segment

```bash
# ปรับขนาด segment ตอน initdb (ไม่สามารถเปลี่ยนหลัง init)
initdb -D /var/lib/postgresql/data --wal-segsize=64  # 64MB segments

# ตรวจสอบขนาด
psql -c "SHOW wal_segment_size;"  # ขนาดใน bytes: 16777216 = 16MB

# ควรใช้ segment size ใหญ่กว่าถ้า:
# - การ archive WAL บ่อยเกินไป
# - Network overhead ของ archiving สูง
# Tradeoff: ใช้ disk space มากขึ้นก่อน segment เต็ม
```

---

## WAL Durability Settings

### fsync: ความน่าเชื่อถือสูงสุด

```sql
-- ดูค่าปัจจุบัน
SHOW fsync;  -- default: on

-- fsync = on: PostgreSQL เรียก fsync() หลังทุก WAL write
-- การันตีว่า WAL อยู่บน disk จริงๆ (ไม่ใช่แค่ OS buffer)

-- fsync = off: ไม่เรียก fsync (เร็วขึ้นมาก แต่ข้อมูลอาจหายเมื่อ crash)
-- อย่าใช้ใน production นอกจากข้อมูลสามารถ recreate ได้

-- ดูวิธี sync (OS-specific)
SHOW wal_sync_method;
-- fdatasync: เร็วกว่า fsync (ไม่ sync metadata)
-- fsync: sync both data + metadata
-- open_sync: O_SYNC flag
-- open_datasync: O_DSYNC flag

-- ใน Linux: fdatasync ดีที่สุดโดยทั่วไป
```

### synchronous_commit: ระดับ Durability

```sql
-- ดู current setting
SHOW synchronous_commit;  -- default: on

-- ======================================
-- synchronous_commit = OFF
-- ======================================
-- COMMIT return ก่อน WAL flush
-- เร็วที่สุด
-- อาจสูญเสีย transactions ล่าสุดเมื่อ crash (ไม่กี่ ms)
-- เหมาะ: logging, analytics ที่ยอมรับ data loss ได้
SET synchronous_commit = OFF;

-- ======================================
-- synchronous_commit = LOCAL
-- ======================================
-- รอให้ WAL เขียนลง local disk
-- ป้องกัน data loss บน primary
-- ไม่รอ replica
SET synchronous_commit = LOCAL;

-- ======================================
-- synchronous_commit = REMOTE_WRITE
-- ======================================
-- รอให้ replica ได้รับ WAL แต่ไม่ต้อง flush
-- ปลอดภัยจาก primary crash
-- Replica crash: อาจสูญเสียข้อมูล
SET synchronous_commit = REMOTE_WRITE;

-- ======================================
-- synchronous_commit = ON (default)
-- ======================================
-- รอให้ primary และ replica flush WAL
-- ปลอดภัยจาก single node crash
SET synchronous_commit = ON;

-- ======================================
-- synchronous_commit = REMOTE_APPLY
-- ======================================
-- รอให้ replica apply WAL (replay)
-- ป้องกัน read-your-writes lag
-- ช้าที่สุด แต่ปลอดภัยสูงสุด
SET synchronous_commit = REMOTE_APPLY;

-- ตั้งค่าสำหรับ transaction เฉพาะ
BEGIN;
SET LOCAL synchronous_commit = OFF;  -- เฉพาะ transaction นี้
INSERT INTO audit_log (event, timestamp) VALUES ('login', NOW());
COMMIT;
```

### Comparison ของ Durability Levels

```
Level           | Data Loss Risk    | Write Speed  | Use Case
----------------|-------------------|--------------|------------------
OFF             | Up to wal_writer  | Fastest      | Non-critical logs
                | _delay (default   |              |
                | 200ms)            |              |
LOCAL           | None (primary)    | Fast         | Single-node only
REMOTE_WRITE    | Replica crash     | Medium       | HA without strong
                | only              |              | guarantees
ON (default)    | None              | Slower       | Most apps
REMOTE_APPLY    | None              | Slowest      | Strong consistency
                |                   |              | requirements
```

---

## wal_level: ระดับ WAL Information

```sql
-- ดู current wal_level
SHOW wal_level;  -- replica (default ใน PostgreSQL 9.6+)

-- ======================================
-- minimal: ข้อมูลน้อยที่สุด
-- ======================================
-- ไม่รองรับ: replication, PITR, wal archiving
-- เร็วที่สุด
-- ใช้ได้เฉพาะ standalone system ที่ไม่ต้องการ replication
ALTER SYSTEM SET wal_level = minimal;

-- ======================================
-- replica (default): สำหรับ streaming replication
-- ======================================
-- รองรับ: streaming replication, PITR, base backup
-- บันทึกข้อมูลเพียงพอสำหรับ replica ทำ redo
ALTER SYSTEM SET wal_level = replica;

-- ======================================
-- logical: สำหรับ logical replication
-- ======================================
-- รองรับ: logical decoding, logical replication
-- บันทึกข้อมูลเพิ่มเติมสำหรับ logical decoding
ALTER SYSTEM SET wal_level = logical;

-- หมายเหตุ: ต้อง restart PostgreSQL หลังเปลี่ยน wal_level
-- (SIGHUP/reload ไม่พอ)
```

---

## WAL Archiving: PITR Configuration

### การตั้งค่า Archive Mode

```sql
-- ใน postgresql.conf:
-- archive_mode = on           -- เปิด archiving
-- archive_command = '<cmd>'   -- คำสั่ง archive
-- archive_library = ''        -- หรือใช้ archive library (PG15+)

-- ตรวจสอบ archive settings
SHOW archive_mode;
SHOW archive_command;
SHOW archive_status_verbose;
```

### archive_command: Copy WAL ไป S3

```bash
# ใน postgresql.conf

# Archive ไป local directory
archive_command = 'cp %p /mnt/archive/wal/%f'

# Archive ไป S3 ด้วย aws cli
archive_command = 'aws s3 cp %p s3://my-backup-bucket/wal/%f'

# Archive ไป S3 ด้วย WAL-E
archive_command = 'envdir /etc/wal-e.d/env /usr/local/bin/wal-e wal-push %p'

# Archive ไป NFS
archive_command = 'rsync -a %p backup-server:/data/wal/%f'

# Archive ด้วย compression
archive_command = 'gzip -c %p > /archive/wal/%f.gz'

# ตัวแปรใน archive_command:
# %p = full path ของ WAL file
# %f = filename ของ WAL file

# ดู archive status
SELECT 
    last_archived_wal,
    last_archived_time,
    last_failed_wal,
    last_failed_time,
    archived_count,
    failed_count
FROM pg_stat_archiver;

-- ตรวจสอบว่ามี WAL ที่ archive ไม่ได้
SELECT 
    CASE 
        WHEN last_failed_time > last_archived_time THEN 'ARCHIVE FAILING!'
        WHEN archived_count = 0 THEN 'No archives yet'
        ELSE 'OK - ' || archived_count || ' archived'
    END as archive_status
FROM pg_stat_archiver;
```

### archive_mode = always

```sql
-- archive_mode = always:
-- Archive WAL บน BOTH primary และ standby
-- ใช้สำหรับ cascaded archiving จาก standby
ALTER SYSTEM SET archive_mode = always;
-- ต้อง restart PostgreSQL
```

### PITR: Point-in-Time Recovery

```bash
# ขั้นตอน PITR:

# 1. สร้าง base backup
pg_basebackup -h localhost -U replica_user \
    -D /backup/base \
    -F tar -z \          # tar + gzip format
    -P \                 # progress
    --checkpoint=fast \  # force immediate checkpoint
    --wal-method=stream  # stream WAL during backup

# 2. กำหนด recovery target (ใน recovery.conf หรือ postgresql.conf)
cat > /backup/recovery.conf << 'EOF'
restore_command = 'aws s3 cp s3://my-backup-bucket/wal/%f %p'
recovery_target_time = '2024-01-15 14:30:00'
recovery_target_action = 'promote'  # หรือ 'pause' เพื่อตรวจสอบก่อน
EOF

# 3. Restore base backup
tar -xzf /backup/base/base.tar.gz -C /var/lib/postgresql/data/

# 4. สร้าง standby.signal
touch /var/lib/postgresql/data/standby.signal

# 5. Start PostgreSQL
pg_ctl start -D /var/lib/postgresql/data/

# PostgreSQL จะ replay WAL จนถึง recovery_target_time
# และ promote เป็น primary

# ดู recovery progress
psql -c "SELECT 
    pg_is_in_recovery(),
    pg_last_xact_replay_timestamp(),
    pg_last_wal_receive_lsn(),
    pg_last_wal_replay_lsn();"
```

### Recovery Target Options

```sql
-- กำหนด recovery target ด้วยวิธีต่างๆ

-- ตาม timestamp
recovery_target_time = '2024-01-15 14:30:00 UTC'

-- ตาม transaction ID (XID)
recovery_target_xid = '12345678'

-- ตาม LSN
recovery_target_lsn = '0/3A789B40'

-- ตาม named restore point
SELECT pg_create_restore_point('before_migration');  -- สร้าง restore point
-- recovery_target_name = 'before_migration'

-- recovery_target_inclusive: รวม transaction ที่ target หรือไม่
recovery_target_inclusive = true   -- รวม (default)
recovery_target_inclusive = false  -- ไม่รวม transaction ที่ target

-- recovery_target_timeline
recovery_target_timeline = 'latest'  -- latest timeline (default)
recovery_target_timeline = 1         -- specific timeline
```

---

## WAL Compression

```sql
-- wal_compression: บีบอัด WAL records
SHOW wal_compression;  -- default: off

-- เปิด compression (PostgreSQL 9.5+)
ALTER SYSTEM SET wal_compression = ON;

-- PostgreSQL 15+: รองรับ algorithms
ALTER SYSTEM SET wal_compression = lz4;    -- เร็วที่สุด
ALTER SYSTEM SET wal_compression = zstd;   -- compression ดีที่สุด
ALTER SYSTEM SET wal_compression = pglz;   -- default algorithm

SELECT pg_reload_conf();

-- ผลที่ได้:
-- WAL size ลดลง 30-70%
-- CPU overhead เพิ่มขึ้นเล็กน้อย
-- เหมาะสำหรับ: bandwidth-limited archiving, replication

-- วัดผล compression
SELECT 
    pg_walfile_name(pg_current_wal_lsn()) as current_wal,
    pg_size_pretty(pg_wal_lsn_diff(
        pg_current_wal_lsn(),
        '0/0'::pg_lsn
    )) as total_wal_written;
```

---

## WAL Buffers

```sql
-- wal_buffers: WAL buffer ใน shared memory
SHOW wal_buffers;  -- default: -1 (auto = 1/32 ของ shared_buffers, max 64MB)

-- ดูค่าจริง
SHOW wal_buffers;

-- ปรับ manual
ALTER SYSTEM SET wal_buffers = '64MB';  -- สำหรับ high-throughput

-- wal_writer_delay: ความถี่ที่ WAL writer flush
SHOW wal_writer_delay;  -- default: 200ms

-- wal_writer_flush_after: flush เมื่อ buffer ถึงขนาดนี้
SHOW wal_writer_flush_after;  -- default: 1MB

-- ปรับ commit_delay: รอก่อน flush เพื่อ batch commits
SHOW commit_delay;      -- default: 0 microseconds
SHOW commit_siblings;   -- default: 5 (transactions)

-- ตั้ง commit_delay เพื่อ batch flush
-- คุ้มค่าเมื่อมี concurrent transactions มาก
ALTER SYSTEM SET commit_delay = 1000;    -- 1ms
ALTER SYSTEM SET commit_siblings = 10;  -- รอถ้ามี 10 active transactions
SELECT pg_reload_conf();
```

---

## WAL Size Management

### max_wal_size และ min_wal_size

```sql
-- max_wal_size: ขนาดสูงสุดของ WAL ใน pg_wal
SHOW max_wal_size;  -- default: 1GB

-- min_wal_size: ขนาดต่ำสุด (recycle segments แทนลบ)
SHOW min_wal_size;  -- default: 80MB

-- เมื่อ WAL size เกิน max_wal_size:
-- PostgreSQL force checkpoint (อาจช้าลง)

-- ปรับสำหรับ high-throughput
ALTER SYSTEM SET max_wal_size = '4GB';   -- เพิ่มเพื่อลด checkpoint frequency
ALTER SYSTEM SET min_wal_size = '512MB'; -- ป้องกัน recycling บ่อย
SELECT pg_reload_conf();

-- ดู current WAL size
SELECT 
    pg_size_pretty(SUM(size)) as wal_size,
    COUNT(*) as segment_count
FROM (
    SELECT name, size 
    FROM pg_ls_dir('pg_wal', true, false) 
    CROSS JOIN LATERAL (
        SELECT (pg_stat_file('pg_wal/' || name)).size
    ) s
) wal_files;
```

### wal_keep_size: เก็บ WAL สำหรับ Replicas

```sql
-- wal_keep_size: เก็บ WAL ขนาดนี้สำหรับ replicas
SHOW wal_keep_size;  -- default: 0 (ไม่เก็บพิเศษ)

-- เพิ่มเพื่อให้ replicas ล้าหลังได้มากขึ้น
-- โดยไม่ต้องใช้ replication slot
ALTER SYSTEM SET wal_keep_size = '2GB';
SELECT pg_reload_conf();

-- ถ้า replica ล้าหลังเกิน wal_keep_size:
-- PostgreSQL ลบ WAL เก่า
-- Replica ต้อง resync (pg_basebackup หรือ pg_rewind)

-- ดู replication lag vs wal_keep_size
SELECT 
    application_name,
    pg_size_pretty(
        pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn)
    ) as replication_lag
FROM pg_stat_replication;
```

### wal_sender_timeout

```sql
-- wal_sender_timeout: timeout สำหรับ inactive WAL sender
SHOW wal_sender_timeout;  -- default: 60s

-- ปรับสำหรับ network ที่ไม่เสถียร
ALTER SYSTEM SET wal_sender_timeout = '120s';

-- 0 = ไม่มี timeout (อาจ hang ถ้า network มีปัญหา)
-- ค่าที่เหมาะสม: 30-120 วินาที ขึ้นอยู่กับ network quality
SELECT pg_reload_conf();
```

---

## Replication Slots: เก็บ WAL สำหรับ Consumers

### Physical Replication Slots

```sql
-- สร้าง physical replication slot
SELECT pg_create_physical_replication_slot('replica_slot_1');

-- ดู replication slots
SELECT 
    slot_name,
    slot_type,
    active,
    active_pid,
    restart_lsn,
    confirmed_flush_lsn,
    pg_size_pretty(
        pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)
    ) as retained_wal_size
FROM pg_replication_slots;

-- ลบ slot
SELECT pg_drop_replication_slot('replica_slot_1');

-- Danger: slot ที่ไม่ active จะสะสม WAL ไม่หยุด!
-- ตรวจสอบ WAL ที่ถูกเก็บโดย inactive slots
SELECT 
    slot_name,
    active,
    pg_size_pretty(
        pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)
    ) as retained_wal
FROM pg_replication_slots
WHERE NOT active
ORDER BY restart_lsn ASC;
```

### max_slot_wal_keep_size: Safety Limit

```sql
-- ป้องกัน WAL disk full จาก inactive slots
SHOW max_slot_wal_keep_size;  -- default: -1 (ไม่จำกัด)

-- ตั้ง limit
ALTER SYSTEM SET max_slot_wal_keep_size = '10GB';
SELECT pg_reload_conf();

-- เมื่อ WAL ที่ต้องเก็บเกิน limit:
-- Slot จะถูก invalidate (ไม่ใช่ drop)
-- ดู invalidated slots:
SELECT slot_name, invalidation_reason
FROM pg_replication_slots
WHERE invalidation_reason IS NOT NULL;

-- Alert สำหรับ large WAL retention
SELECT 
    slot_name,
    active,
    pg_size_pretty(
        pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)
    ) as retained_wal
FROM pg_replication_slots
WHERE pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) > 5 * 1024^3;  -- > 5GB
```

### Logical Replication Slots

```sql
-- สร้าง logical replication slot
SELECT pg_create_logical_replication_slot(
    'my_logical_slot',
    'pgoutput'    -- output plugin: pgoutput, wal2json, decoderbufs
);

-- ดู logical changes ผ่าน slot
SELECT * FROM pg_logical_slot_get_changes(
    'my_logical_slot',
    NULL,    -- upto_lsn: ถึง LSN นี้
    NULL,    -- upto_nchanges: จำนวน changes
    'include-xids', '1'  -- options
);

-- Peek โดยไม่ consume
SELECT * FROM pg_logical_slot_peek_changes(
    'my_logical_slot', NULL, NULL
);

-- ล้าง slot
SELECT pg_drop_replication_slot('my_logical_slot');
```

---

## Monitoring WAL

### pg_stat_wal: WAL Activity Statistics

```sql
-- ดู WAL statistics
SELECT 
    wal_records,
    wal_fpi,                    -- full page images
    wal_bytes,
    pg_size_pretty(wal_bytes) as wal_bytes_pretty,
    wal_buffers_full,           -- times WAL buffer was full
    wal_write,                  -- WAL writes
    wal_sync,                   -- WAL syncs (fsync calls)
    wal_write_time,             -- time spent writing
    wal_sync_time,              -- time spent syncing
    stats_reset
FROM pg_stat_wal;

-- ดู WAL generation rate
WITH wal_snapshot AS (
    SELECT wal_bytes, NOW() as ts FROM pg_stat_wal
)
SELECT 
    wal_bytes - LAG(wal_bytes) OVER (ORDER BY ts) as bytes_since_last,
    pg_size_pretty(
        (wal_bytes - LAG(wal_bytes) OVER (ORDER BY ts)) / 
        EXTRACT(EPOCH FROM (ts - LAG(ts) OVER (ORDER BY ts)))
    ) as wal_rate_per_second
FROM wal_snapshot;

-- Reset statistics
SELECT pg_stat_reset_shared('wal');
```

### pg_stat_replication: ดู WAL Senders

```sql
-- ดู replication connections
SELECT 
    pid,
    usename,
    application_name,
    client_addr,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    -- คำนวณ lag
    pg_size_pretty(pg_wal_lsn_diff(sent_lsn, write_lsn)) as write_lag_bytes,
    pg_size_pretty(pg_wal_lsn_diff(write_lsn, flush_lsn)) as flush_lag_bytes,
    pg_size_pretty(pg_wal_lsn_diff(flush_lsn, replay_lsn)) as replay_lag_bytes,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn)) as total_lag_bytes,
    write_lag,
    flush_lag,
    replay_lag,
    sync_state
FROM pg_stat_replication;
```

### pg_waldump: Debug WAL Contents

```bash
# pg_waldump: อ่านเนื้อหาของ WAL file
# ใช้สำหรับ debugging เท่านั้น (ไม่ใช้สำหรับ replication)

# ดู WAL ทั้งหมดใน segment
pg_waldump $PGDATA/pg_wal/000000010000000000000001

# ดู WAL ตาม timeline และ LSN range
pg_waldump -s 0/3A789B40 -e 0/3B000000 $PGDATA/pg_wal/

# ดู WAL specific table (relfilenode)
# หา relfilenode
psql -c "SELECT relfilenode FROM pg_class WHERE relname = 'orders';"
# 12345

# Filter WAL สำหรับ table นี้
pg_waldump -r heap $PGDATA/pg_wal/ | grep "rel 1663/16384/12345"

# ดู WAL แบบ stats
pg_waldump --stats $PGDATA/pg_wal/000000010000000000000001

# Output ตัวอย่าง:
# Type                                           N      (%)          Record size      (%)          FPW size         (%)
# ----                                           -      ---          -----------      ---          --------         ---
# XLOG/FPW                                     123     5.34%              12300     1.23%             123000    45.67%
# Heap/INSERT                                 1234    53.56%             123400    12.34%                  0     0.00%
# Heap/UPDATE                                  567    24.60%              56700     5.67%                  0     0.00%
```

---

## Checkpoint Configuration

```sql
-- Checkpoint: flush dirty pages จาก shared_buffers ลง disk
-- และสร้าง WAL "safe point"

SHOW checkpoint_completion_target;  -- default: 0.9
SHOW checkpoint_timeout;            -- default: 5min

-- checkpoint_completion_target: 
-- เป้าหมาย fraction ของ checkpoint_timeout ที่จะ spread I/O
-- 0.9 = spread I/O ตลอด 90% ของ timeout period

-- ดู checkpoint activity
SELECT 
    checkpoints_timed,      -- checkpoints triggered by timeout
    checkpoints_req,        -- checkpoints triggered by WAL full
    checkpoint_write_time,  -- ms writing dirty pages
    checkpoint_sync_time,   -- ms syncing pages
    buffers_checkpoint,     -- buffers written during checkpoints
    buffers_clean,          -- buffers written by background writer
    buffers_backend,        -- buffers written directly by backends
    buffers_backend_fsync,  -- backend fsync calls
    buffers_alloc,          -- new buffers allocated
    stats_reset
FROM pg_stat_bgwriter;

-- ปัญหา: checkpoints_req > checkpoints_timed
-- หมายความว่า WAL เต็มก่อน timeout ครบ
-- แก้: เพิ่ม max_wal_size หรือ checkpoint_timeout

-- ปรับ checkpoint settings
ALTER SYSTEM SET checkpoint_timeout = '15min';       -- เพิ่มจาก 5min
ALTER SYSTEM SET checkpoint_completion_target = 0.9; -- spread I/O
ALTER SYSTEM SET max_wal_size = '4GB';               -- ลด forced checkpoints
SELECT pg_reload_conf();
```

---

## Full Page Writes (FPW)

```sql
-- full_page_writes: เขียน entire page ครั้งแรกหลัง checkpoint
-- ป้องกัน partial page write corruption

SHOW full_page_writes;  -- default: on

-- FPW ทำให้ WAL size ใหญ่ขึ้นมาก
-- (ทุก page first write = +8KB WAL record)

-- เมื่อใช้ ZFS/hardware RAID ที่รองรับ atomic writes:
-- สามารถปิด full_page_writes (เสี่ยงน้อยลง)
-- แต่ยังคงควรเปิดไว้เสมอเพื่อความปลอดภัย

-- WAL FPI statistics
SELECT 
    wal_fpi as full_page_images,
    wal_bytes,
    ROUND(100.0 * wal_fpi * 8192 / NULLIF(wal_bytes, 0), 2) as fpi_pct_of_wal
FROM pg_stat_wal;

-- ถ้า FPI สูงมาก: checkpoint บ่อยเกินไป
-- แก้: เพิ่ม checkpoint_timeout หรือ max_wal_size
```

---

## Comprehensive WAL Configuration

### postgresql.conf สำหรับ Production

```ini
# ============================================
# WAL SETTINGS - Production Configuration
# ============================================

# Level
wal_level = replica                # สำหรับ replication

# Durability
fsync = on                         # ต้องเปิดใน production
synchronous_commit = on            # durability guarantee
full_page_writes = on              # protect against partial writes

# WAL Buffers
wal_buffers = 64MB                 # เพิ่มสำหรับ high-throughput

# WAL Writer
wal_writer_delay = 200ms           # flush interval
wal_writer_flush_after = 1MB       # flush when buffer reaches this

# Commit Batching (optional)
# commit_delay = 1000              # 1ms wait for batching
# commit_siblings = 10             # minimum concurrent transactions

# WAL Size
min_wal_size = 512MB               # prevent excessive recycling
max_wal_size = 4GB                 # prevent forced checkpoints

# WAL Compression (optional, saves disk space)
wal_compression = lz4              # requires PostgreSQL 15+

# WAL Retention for Replicas
wal_keep_size = 1GB                # retain for replicas without slots

# Replication Slot Safety
max_slot_wal_keep_size = 10GB      # prevent disk full

# WAL Sender Timeout
wal_sender_timeout = 60s

# Checkpoint
checkpoint_timeout = 15min         # เพิ่มจาก default 5min
checkpoint_completion_target = 0.9

# Archive (enable เมื่อต้องการ PITR)
# archive_mode = on
# archive_command = 'aws s3 cp %p s3://backup-bucket/wal/%f'

# Logging
log_checkpoints = on               # log checkpoint activity
```

---

## WAL Monitoring Script

```python
#!/usr/bin/env python3
"""Monitor WAL health and alert on issues"""

import psycopg2
from dataclasses import dataclass
from typing import List, Optional
import smtplib
from email.mime.text import MIMEText

@dataclass
class WALStatus:
    current_lsn: str
    wal_bytes_total: int
    wal_rate_per_sec: float
    archive_status: str
    last_archived: Optional[str]
    archive_failures: int
    replication_slots: List[dict]
    slot_wal_retained: int  # bytes
    checkpoint_req: int
    checkpoint_timed: int
    buffers_backend_fsync: int

def get_wal_status(conn_string: str) -> WALStatus:
    conn = psycopg2.connect(conn_string)
    cur = conn.cursor()
    
    # WAL basics
    cur.execute("SELECT pg_current_wal_lsn()::text")
    current_lsn = cur.fetchone()[0]
    
    # WAL stats
    cur.execute("""
        SELECT 
            wal_bytes,
            wal_records
        FROM pg_stat_wal
    """)
    wal_row = cur.fetchone()
    
    # Archive status
    cur.execute("""
        SELECT 
            COALESCE(last_archived_wal, 'None') as last_archived_wal,
            COALESCE(last_archived_time::text, 'Never') as last_archived_time,
            failed_count,
            CASE 
                WHEN last_failed_time > last_archived_time THEN 'FAILING'
                ELSE 'OK'
            END as status
        FROM pg_stat_archiver
    """)
    archive_row = cur.fetchone()
    
    # Replication slots
    cur.execute("""
        SELECT 
            slot_name,
            slot_type,
            active,
            pg_size_pretty(
                pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)
            ) as retained_wal,
            pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) as retained_bytes
        FROM pg_replication_slots
    """)
    slots = [dict(zip([d[0] for d in cur.description], row)) for row in cur.fetchall()]
    
    total_slot_bytes = sum(s.get('retained_bytes', 0) or 0 for s in slots)
    
    # Checkpoint stats
    cur.execute("""
        SELECT 
            checkpoints_req,
            checkpoints_timed,
            buffers_backend_fsync
        FROM pg_stat_bgwriter
    """)
    ckpt_row = cur.fetchone()
    
    cur.close()
    conn.close()
    
    return WALStatus(
        current_lsn=current_lsn,
        wal_bytes_total=wal_row[0],
        wal_rate_per_sec=0,  # Would need 2 measurements
        archive_status=archive_row[3] if archive_row else 'N/A',
        last_archived=archive_row[1] if archive_row else None,
        archive_failures=archive_row[2] if archive_row else 0,
        replication_slots=slots,
        slot_wal_retained=total_slot_bytes,
        checkpoint_req=ckpt_row[0] if ckpt_row else 0,
        checkpoint_timed=ckpt_row[1] if ckpt_row else 0,
        buffers_backend_fsync=ckpt_row[2] if ckpt_row else 0,
    )

def check_alerts(status: WALStatus) -> List[str]:
    alerts = []
    
    # Archive failing
    if status.archive_status == 'FAILING':
        alerts.append(f"CRITICAL: WAL archiving is FAILING! Last archive failures: {status.archive_failures}")
    
    # Large slot WAL retention
    gb = 1024 ** 3
    if status.slot_wal_retained > 5 * gb:
        alerts.append(
            f"WARNING: Replication slots retaining {status.slot_wal_retained / gb:.1f}GB of WAL"
        )
    
    # Inactive slots with large WAL
    for slot in status.replication_slots:
        if not slot['active'] and (slot.get('retained_bytes', 0) or 0) > gb:
            alerts.append(
                f"WARNING: Inactive slot '{slot['slot_name']}' retaining {slot['retained_wal']}"
            )
    
    # Checkpoint pressure
    if status.checkpoint_req > 0:
        ratio = status.checkpoint_req / max(status.checkpoint_timed, 1)
        if ratio > 0.5:
            alerts.append(
                f"WARNING: {status.checkpoint_req} forced checkpoints "
                f"({ratio:.1%} of total). Consider increasing max_wal_size"
            )
    
    # Backend fsync (performance issue)
    if status.buffers_backend_fsync > 100:
        alerts.append(
            f"WARNING: {status.buffers_backend_fsync} backend fsync calls. "
            "Checkpoint is not keeping up with writes"
        )
    
    return alerts

# Main monitoring loop
def monitor_wal(conn_string: str, interval: int = 60):
    import time
    
    print("Starting WAL monitoring...")
    
    while True:
        try:
            status = get_wal_status(conn_string)
            alerts = check_alerts(status)
            
            if alerts:
                for alert in alerts:
                    print(f"ALERT: {alert}")
            else:
                print(f"WAL OK: LSN={status.current_lsn}, "
                      f"Archive={status.archive_status}, "
                      f"Slots={len(status.replication_slots)}")
            
        except Exception as e:
            print(f"ERROR monitoring WAL: {e}")
        
        time.sleep(interval)

if __name__ == '__main__':
    monitor_wal("postgresql://localhost/mydb")
```

---

## สรุป WAL Best Practices

```
1. Production ต้องตั้งค่า:
   - fsync = on (เสมอ)
   - wal_level = replica (หรือ logical ถ้าใช้ logical replication)
   - full_page_writes = on (ยกเว้น ZFS/atomic storage)
   
2. Archive WAL สำหรับ PITR:
   - ใช้ archive_command กับ S3 หรือ offsite storage
   - ทดสอบ restore ทุก 3 เดือน
   
3. Replication Slots ต้องระวัง:
   - ตั้ง max_slot_wal_keep_size เสมอ
   - Monitor inactive slots
   - Drop slots ที่ไม่ใช้งาน
   
4. Tune checkpoint:
   - เพิ่ม checkpoint_timeout และ max_wal_size
   - checkpoints_req ควรต่ำกว่า checkpoints_timed
   
5. WAL Compression ประหยัด I/O:
   - ใช้ lz4 (fast) หรือ zstd (small)
   - เหมาะสำหรับ bandwidth-limited environments
   
6. Monitor สม่ำเสมอ:
   - pg_stat_archiver: archive health
   - pg_stat_replication: replica lag
   - pg_replication_slots: retained WAL
   - pg_stat_bgwriter: checkpoint pressure
```

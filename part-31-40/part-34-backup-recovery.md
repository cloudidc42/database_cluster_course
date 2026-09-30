# Part 34: Backup และ Recovery Strategies

## บทนำ

การสำรองข้อมูล (Backup) และการกู้คืน (Recovery) เป็นสิ่งจำเป็นอย่างยิ่งสำหรับ production database ไม่มีระบบใดที่ปลอดภัย 100% จาก hardware failure, human error, หรือ software bugs

---

## 1. Backup Types

### Full Backup

สำรองข้อมูลทั้งหมด

```
ข้อดี:  - Simple, self-contained
        - Recovery เร็ว (ไม่ต้อง chain)
ข้อเสีย: - ใช้ space มาก
         - ใช้เวลานาน
```

### Incremental Backup

สำรองเฉพาะข้อมูลที่เปลี่ยนแปลงตั้งแต่ backup ครั้งก่อน (Full หรือ Incremental)

```
ข้อดี:  - ใช้ space น้อย
        - เร็วกว่า Full backup
ข้อเสีย: - Recovery ต้องใช้ Full + chain of incrementals
         - ซับซ้อนกว่า
```

### Differential Backup

สำรองข้อมูลที่เปลี่ยนแปลงตั้งแต่ Full backup ล่าสุด

```
ข้อดี:  - Recovery ง่ายกว่า Incremental (Full + Differential)
        - เร็วกว่า Full
ข้อเสีย: - ใหญ่ขึ้นเรื่อยๆ ถ้านานมากตั้งแต่ Full
```

### Continuous Backup (WAL-based)

เก็บ WAL (Write-Ahead Log) อย่างต่อเนื่อง

```
ข้อดี:  - PITR (Point-in-Time Recovery) - กู้ได้ถึงนาทีหรือวินาที
        - RPO ต่ำมาก
ข้อเสีย: - ซับซ้อกว่า
         - ต้องมี storage สำหรับ WAL
```

---

## 2. PostgreSQL Backup Methods

### Overview

```
Method            | Type       | Consistency | Online? | Size
------------------|-----------|-------------|---------|------
pg_dump           | Logical   | Consistent  | Yes     | Compressed
pg_dumpall        | Logical   | Per-DB      | Yes     | Larger
pg_basebackup     | Physical  | Consistent  | Yes     | Full DB
WAL Archiving     | Physical  | Continuous  | Yes     | Incremental
pgBackRest        | Physical  | Consistent  | Yes     | Full+Delta
```

---

## 3. pg_dump: Logical Backup

### พื้นฐาน

```bash
# Dump single database
pg_dump mydb > mydb_backup.sql

# Custom format (แนะนำ)
pg_dump -Fc mydb > mydb_backup.dump

# Directory format (parallel dump)
pg_dump -Fd mydb -f /backup/mydb_dir/

# Tar format
pg_dump -Ft mydb > mydb_backup.tar

# Plain SQL with verbose
pg_dump -Fp -v mydb > mydb_backup.sql
```

### Options ที่สำคัญ

```bash
# Specify host, port, user
pg_dump -h localhost -p 5432 -U postgres -d mydb -Fc -f mydb.dump

# ใช้ environment variables
export PGHOST=localhost
export PGPORT=5432
export PGUSER=postgres
export PGPASSWORD=mypassword
pg_dump -d mydb -Fc -f mydb.dump

# หรือใช้ .pgpass file (ปลอดภัยกว่า PGPASSWORD)
echo "localhost:5432:mydb:postgres:mypassword" >> ~/.pgpass
chmod 600 ~/.pgpass

# Schema only (ไม่มี data)
pg_dump --schema-only mydb > schema_only.sql
pg_dump -s mydb > schema_only.sql

# Data only (ไม่มี DDL)
pg_dump --data-only mydb > data_only.sql
pg_dump -a mydb > data_only.sql

# Specific schema
pg_dump --schema=public mydb > public_schema.sql
pg_dump -n public mydb > public_schema.sql

# Specific table
pg_dump --table=users mydb > users_table.sql
pg_dump -t users mydb > users_table.sql

# Multiple tables
pg_dump -t users -t orders mydb > selected_tables.sql

# Exclude table
pg_dump --exclude-table=logs mydb > without_logs.sql
pg_dump -T logs mydb > without_logs.sql

# Exclude table data (include DDL but not data)
pg_dump --exclude-table-data=audit_logs mydb > without_audit_data.sql

# If-exists (ใช้ DROP IF EXISTS)
pg_dump --if-exists --clean mydb > clean_backup.sql

# Compression
pg_dump -Fc --compress=9 mydb > mydb.dump

# No owner, no privileges
pg_dump --no-owner --no-privileges mydb > portable_backup.sql
```

### Parallel Dump

```bash
# Parallel dump ด้วย directory format
pg_dump -Fd -j 4 mydb -f /backup/mydb_parallel/
# -j 4 = 4 parallel workers

# Monitor progress
pg_dump -Fd -j 4 --verbose mydb -f /backup/mydb_parallel/
```

---

## 4. pg_dumpall: Dump All Databases

```bash
# Dump ทุก database รวม global objects (roles, tablespaces)
pg_dumpall > full_cluster_backup.sql

# Globals only (roles, tablespaces - ไม่มี data)
pg_dumpall --globals-only > globals.sql

# Roles only
pg_dumpall --roles-only > roles.sql

# Tablespaces only
pg_dumpall --tablespaces-only > tablespaces.sql

# No roles, no tablespaces (แค่ databases)
pg_dumpall --no-role-passwords > databases_only.sql

# ดู roles ที่มี
pg_dumpall --globals-only | grep "CREATE ROLE"
```

---

## 5. pg_restore: Restore

```bash
# Restore จาก custom format
pg_restore -d mydb mydb_backup.dump

# สร้าง database ใหม่แล้ว restore
createdb mydb_restored
pg_restore -d mydb_restored mydb_backup.dump

# Restore เฉพาะ table
pg_restore -t users -d mydb mydb_backup.dump

# Restore เฉพาะ schema
pg_restore -n public -d mydb mydb_backup.dump

# Schema only
pg_restore -s -d mydb mydb_backup.dump

# Data only
pg_restore -a -d mydb mydb_backup.dump

# Drop existing objects before restore
pg_restore -c -d mydb mydb_backup.dump

# --if-exists (ใช้กับ -c)
pg_restore -c --if-exists -d mydb mydb_backup.dump

# Parallel restore
pg_restore -j 4 -d mydb /backup/mydb_parallel/

# Verbose
pg_restore -v -d mydb mydb_backup.dump

# Restore จาก plain SQL
psql -d mydb < mydb_backup.sql

# ดู table of contents
pg_restore -l mydb_backup.dump

# Restore specific items from list
pg_restore -l mydb_backup.dump > restore_list.txt
# แก้ไข restore_list.txt (comment ; ที่ต้องการ skip)
pg_restore -L restore_list.txt -d mydb mydb_backup.dump
```

---

## 6. pg_basebackup: Physical Backup

```bash
# Basic base backup
pg_basebackup -D /backup/base_backup -U replication_user -h localhost

# พร้อม WAL streaming
pg_basebackup -D /backup/base_backup \
    -U replication_user \
    -h localhost \
    --wal-method=stream \
    --checkpoint=fast \
    --progress \
    --verbose

# Tar format (compressed)
pg_basebackup -D /backup/ \
    -Ft \
    --compress=9 \
    -U replication_user \
    -h localhost

# ดู backup ที่ได้
ls -la /backup/base_backup/

# pg_basebackup options:
# -D: destination directory
# --wal-method=stream: stream WAL ระหว่าง backup
# --wal-method=fetch: fetch WAL หลัง backup
# --wal-method=none: ไม่เอา WAL
# -Fp: plain format (default)
# -Ft: tar format
# --compress: compression level 0-9
# --checkpoint=fast: force fast checkpoint
# -r rate: limit bandwidth (e.g., 10M = 10MB/s)
```

---

## 7. WAL Archiving: Continuous Archiving + PITR

### Setup

```bash
# postgresql.conf
wal_level = replica             # minimum for archiving
archive_mode = on               # enable archiving
archive_command = 'cp %p /archive/%f'  # copy WAL file
# %p = source path, %f = filename

# ตัวอย่าง archive_command ที่ดีกว่า
archive_command = 'test ! -f /archive/%f && cp %p /archive/%f'
# ตรวจก่อนว่า file ยังไม่มีใน archive

# Archive ไป S3
archive_command = 'aws s3 cp %p s3://my-bucket/wal/%f'

# Archive ด้วย compression
archive_command = 'gzip -c %p > /archive/%f.gz'

# ตรวจสอบ archive status
SELECT * FROM pg_stat_archiver;

# Archive ทันที (manual)
SELECT pg_switch_wal();
```

### Restore Configuration

```bash
# recovery.conf หรือ postgresql.conf (PG12+)
restore_command = 'cp /archive/%f %p'
# หรือจาก S3
# restore_command = 'aws s3 cp s3://my-bucket/wal/%f %p'

# Point-in-Time Recovery targets:
recovery_target_time = '2024-01-15 14:30:00'  # restore to specific time
recovery_target_lsn = '0/15D68C50'             # restore to specific LSN
recovery_target_xid = '12345'                  # restore to specific transaction
recovery_target_name = 'before_big_delete'     # restore to named checkpoint
recovery_target_inclusive = true               # include target transaction

# After reaching target:
recovery_target_action = 'promote'   # promote to primary (default)
# recovery_target_action = 'pause'  # pause for inspection
# recovery_target_action = 'shutdown' # shutdown

# ใน postgresql.conf (PG12+)
# ต้องมี recovery_target_* ตัวอย่าง:
# recovery_target_time = '2024-01-15 14:30:00+07'
```

### PITR Recovery Steps

```bash
# 1. หยุด PostgreSQL
sudo systemctl stop postgresql

# 2. Backup ข้อมูลปัจจุบัน (ป้องกัน)
mv /var/lib/postgresql/16/main /var/lib/postgresql/16/main_corrupted

# 3. Restore base backup
tar -xzf /backup/base_20240115.tar.gz -C /var/lib/postgresql/16/main

# 4. กำหนด restore config
cat > /var/lib/postgresql/16/main/postgresql.conf << EOF
# Recovery settings
restore_command = 'cp /archive/%f %p'
recovery_target_time = '2024-01-15 14:00:00+07'
recovery_target_action = 'promote'
EOF

# 5. สร้าง recovery.signal
touch /var/lib/postgresql/16/main/recovery.signal

# 6. Fix permissions
chown -R postgres:postgres /var/lib/postgresql/16/main

# 7. Start PostgreSQL
sudo systemctl start postgresql

# 8. Monitor recovery
tail -f /var/log/postgresql/postgresql-16-main.log

# 9. ตรวจสอบ recovery status
psql -c "SELECT pg_is_in_recovery();"
psql -c "SELECT pg_last_wal_replay_lsn();"

# 10. หลัง recovery สำเร็จ
# postgresql จะ promote อัตโนมัติถ้า target_action = 'promote'
# ถ้า pause: SELECT pg_promote();
```

---

## 8. Backup Compression

```bash
# gzip (standard)
pg_dump -Fc --compress=6 mydb > mydb.dump.gz

# ด้วย pipe
pg_dump mydb | gzip > mydb.sql.gz
gunzip < mydb.sql.gz | psql mydb

# lz4 (เร็วกว่า gzip แต่ compress น้อยกว่า)
pg_dump mydb | lz4 > mydb.sql.lz4
lz4 -d mydb.sql.lz4 | psql mydb

# zstd (ดีที่สุด: เร็ว + compress ดี)
pg_dump mydb | zstd > mydb.sql.zst
zstd -d mydb.sql.zst -c | psql mydb

# เปรียบเทียบ compression
time pg_dump mydb | gzip > test.gz
time pg_dump mydb | lz4 > test.lz4
time pg_dump mydb | zstd > test.zst
ls -lh test.*

# สำหรับ pg_basebackup
pg_basebackup -D /backup -Ft -z  # gzip
pg_basebackup -D /backup -Ft --compress=gzip:6
pg_basebackup -D /backup -Ft --compress=lz4    # PG15+
pg_basebackup -D /backup -Ft --compress=zstd   # PG15+
```

---

## 9. Backup Encryption

```bash
# GPG Encryption (Symmetric)
pg_dump mydb | gpg --symmetric --cipher-algo AES256 > mydb.dump.gpg
gpg --decrypt mydb.dump.gpg | psql mydb

# GPG with key (Asymmetric)
# สร้าง keypair
gpg --gen-key

# Encrypt ด้วย public key
pg_dump mydb | gpg --encrypt --recipient backup@example.com > mydb.dump.gpg

# Decrypt ด้วย private key
gpg --decrypt mydb.dump.gpg | psql mydb

# age (modern encryption tool)
# ติดตั้ง: apt install age
# สร้าง key
age-keygen -o backup_key.txt

# Encrypt
pg_dump mydb | age -r age1xxxxxxxxxx > mydb.dump.age

# Decrypt
age --decrypt -i backup_key.txt mydb.dump.age | psql mydb

# OpenSSL (AES-256-CBC)
pg_dump mydb | openssl enc -aes-256-cbc -pbkdf2 -pass pass:mypassword > mydb.enc
openssl enc -d -aes-256-cbc -pbkdf2 -pass pass:mypassword < mydb.enc | psql mydb
```

---

## 10. Backup to S3/MinIO

```bash
# Setup AWS CLI
aws configure

# Backup ไป S3
pg_dump -Fc mydb | aws s3 cp - s3://my-backup-bucket/mydb/$(date +%Y%m%d_%H%M%S).dump

# ด้วย compression + encryption + S3
pg_dump mydb | \
    gzip | \
    gpg --symmetric --cipher-algo AES256 | \
    aws s3 cp - s3://my-bucket/backups/mydb_$(date +%Y%m%d).sql.gz.gpg

# Restore จาก S3
aws s3 cp s3://my-bucket/backups/mydb_20240115.dump - | pg_restore -d mydb

# rclone (รองรับหลาย cloud providers)
# ติดตั้ง: apt install rclone
# Config: rclone config

# Backup ด้วย rclone
pg_dump -Fc mydb > /tmp/mydb_backup.dump
rclone copy /tmp/mydb_backup.dump remote:my-bucket/backups/

# MinIO setup
export MINIO_ENDPOINT=http://minio:9000
export AWS_ACCESS_KEY_ID=minioadmin
export AWS_SECRET_ACCESS_KEY=minioadmin

aws --endpoint-url $MINIO_ENDPOINT s3 cp mydb.dump s3://backups/

# pgBackRest + S3
# pgbackrest.conf:
# [global]
# repo1-type=s3
# repo1-s3-bucket=my-backup-bucket
# repo1-s3-endpoint=s3.amazonaws.com
# repo1-s3-key=AKIAXXXXXXXX
# repo1-s3-key-secret=xxxxxxxxxx
# repo1-s3-region=ap-southeast-1
```

---

## 11. RTO และ RPO

```
RPO (Recovery Point Objective):
= ข้อมูลที่ยอมสูญเสียได้สูงสุด
= ระยะเวลา backup interval

RTO (Recovery Time Objective):
= เวลาที่ต้องการในการ recover
= นาน = user impact

Strategy comparison:
Method          | RPO      | RTO
----------------|----------|--------
pg_dump daily   | 24 hours | 30-60 min
pg_dump hourly  | 1 hour   | 10-30 min
PITR (WAL)      | seconds  | 30-60 min
Streaming Rep.  | seconds  | 1-5 min (failover)
```

---

## 12. Backup Schedule

```bash
# Cron job สำหรับ backup

# Edit crontab
crontab -e

# Daily full backup at 2am
0 2 * * * /usr/local/bin/pg_backup_daily.sh >> /var/log/pg_backup.log 2>&1

# Hourly WAL archive check
0 * * * * /usr/local/bin/verify_wal_archive.sh >> /var/log/wal_verify.log 2>&1

# Weekly backup verification (restore test)
0 3 * * 0 /usr/local/bin/verify_backup.sh >> /var/log/backup_verify.log 2>&1

# pg_backup_daily.sh
#!/bin/bash
BACKUP_DIR="/backup/daily"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
DB_NAME="mydb"
RETENTION_DAYS=7

# Backup
pg_dump -Fc -U postgres -d $DB_NAME > "$BACKUP_DIR/${DB_NAME}_${TIMESTAMP}.dump"

if [ $? -eq 0 ]; then
    echo "$(date): Backup successful: ${DB_NAME}_${TIMESTAMP}.dump"
    # Remove old backups
    find "$BACKUP_DIR" -name "${DB_NAME}_*.dump" -mtime +$RETENTION_DAYS -delete
else
    echo "$(date): Backup FAILED!" >&2
    # Send alert
    echo "PostgreSQL backup failed on $(hostname)" | \
        mail -s "ALERT: Backup Failed" admin@example.com
fi
```

---

## 13. Backup Retention Policy

```bash
# retention-cleanup.sh
#!/bin/bash

BACKUP_DIR="/backup"
LOG_FILE="/var/log/backup_retention.log"

echo "$(date): Starting retention cleanup" >> $LOG_FILE

# Keep last 7 daily backups
find "$BACKUP_DIR/daily" -name "*.dump" -mtime +7 -exec ls -la {} \; >> $LOG_FILE
find "$BACKUP_DIR/daily" -name "*.dump" -mtime +7 -delete

# Keep last 4 weekly backups
find "$BACKUP_DIR/weekly" -name "*.dump" -mtime +28 -delete

# Keep last 3 monthly backups
find "$BACKUP_DIR/monthly" -name "*.dump" -mtime +90 -delete

# ลบ WAL ที่เก่าเกิน
find "/archive/wal" -name "*.gz" -mtime +14 -delete

echo "$(date): Cleanup complete" >> $LOG_FILE

# Report disk usage
df -h "$BACKUP_DIR" >> $LOG_FILE
du -sh "$BACKUP_DIR"/* >> $LOG_FILE
```

---

## 14. pgBackRest: Advanced Backup Tool

### Installation

```bash
# Ubuntu/Debian
sudo apt-get install pgbackrest

# CentOS/RHEL
sudo yum install pgbackrest

# ตรวจสอบ version
pgbackrest version
```

### Configuration

```ini
# /etc/pgbackrest/pgbackrest.conf

[global]
# Repository settings
repo1-path=/var/lib/pgbackrest
repo1-retention-full=2        # เก็บ 2 full backups
repo1-retention-diff=4        # เก็บ 4 differential backups

# Compression
repo1-bundle=y
repo1-compress-type=lz4

# Optional: S3
# repo1-type=s3
# repo1-s3-bucket=my-pgbackrest-bucket
# repo1-s3-endpoint=s3.amazonaws.com
# repo1-s3-region=ap-southeast-1

# Logging
log-level-console=info
log-level-file=detail
log-path=/var/log/pgbackrest

# Process count
process-max=4

[global:archive-push]
compress-level=3

[mydb]
pg1-path=/var/lib/postgresql/16/main
pg1-user=postgres

[mydb2]
pg1-path=/var/lib/postgresql/16/mydb2
pg1-user=postgres
```

### PostgreSQL Configuration

```bash
# postgresql.conf
archive_mode = on
archive_command = 'pgbackrest --stanza=mydb archive-push %p'
archive_timeout = 60s

wal_level = replica
max_wal_senders = 3
```

### pgBackRest Commands

```bash
# สร้าง stanza (ครั้งแรก)
sudo -u postgres pgbackrest --stanza=mydb stanza-create

# Verify configuration
sudo -u postgres pgbackrest --stanza=mydb check

# Full backup
sudo -u postgres pgbackrest --stanza=mydb --type=full backup

# Differential backup (ตาม last full)
sudo -u postgres pgbackrest --stanza=mydb --type=diff backup

# Incremental backup (ตาม last backup)
sudo -u postgres pgbackrest --stanza=mydb --type=incr backup

# ดู backup info
sudo -u postgres pgbackrest --stanza=mydb info

# Restore full
sudo -u postgres pgbackrest --stanza=mydb restore

# PITR
sudo -u postgres pgbackrest --stanza=mydb restore \
    --target="2024-01-15 14:30:00" \
    --target-action=promote

# Restore specific backup
sudo -u postgres pgbackrest --stanza=mydb restore \
    --set=20240115-020000F

# Restore to different path (for testing)
sudo -u postgres pgbackrest --stanza=mydb restore \
    --pg1-path=/var/lib/postgresql/test_restore \
    --recovery-option="port=5433"

# Archive check
sudo -u postgres pgbackrest --stanza=mydb archive-get 000000010000000000000001 /tmp/test_wal

# Expire old backups
sudo -u postgres pgbackrest --stanza=mydb expire

# Verify backup integrity
sudo -u postgres pgbackrest --stanza=mydb verify
```

---

## 15. Automated Backup Script

```bash
#!/bin/bash
# /usr/local/bin/pg_backup.sh
# Comprehensive backup script

set -e
set -o pipefail

# Configuration
DB_HOST="${PGHOST:-localhost}"
DB_PORT="${PGPORT:-5432}"
DB_USER="${PGUSER:-postgres}"
BACKUP_BASE="/backup"
S3_BUCKET="${S3_BUCKET:-s3://my-db-backups}"
RETENTION_DAYS="${RETENTION_DAYS:-7}"
LOG_FILE="/var/log/pg_backup.log"
SLACK_WEBHOOK="${SLACK_WEBHOOK:-}"

# Timestamp
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
DATE=$(date +%Y%m%d)

# Functions
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"
}

error() {
    log "ERROR: $*" >&2
    notify_failure "$*"
    exit 1
}

notify_success() {
    local message="$1"
    log "SUCCESS: $message"
    if [ -n "$SLACK_WEBHOOK" ]; then
        curl -s -X POST "$SLACK_WEBHOOK" \
            -H 'Content-type: application/json' \
            --data "{\"text\":\"✅ DB Backup Success: $message\"}" || true
    fi
}

notify_failure() {
    local message="$1"
    if [ -n "$SLACK_WEBHOOK" ]; then
        curl -s -X POST "$SLACK_WEBHOOK" \
            -H 'Content-type: application/json' \
            --data "{\"text\":\"🚨 DB Backup FAILED: $message on $(hostname)\"}" || true
    fi
    # Email alert
    echo "PostgreSQL backup failed: $message" | \
        mail -s "ALERT: DB Backup Failed - $(hostname)" \
        "${ALERT_EMAIL:-admin@example.com}" 2>/dev/null || true
}

check_disk_space() {
    local dir="$1"
    local required_gb="${2:-10}"
    local available_gb=$(df -BG "$dir" | awk 'NR==2{print $4}' | tr -d 'G')
    if [ "$available_gb" -lt "$required_gb" ]; then
        error "Insufficient disk space: ${available_gb}GB available, ${required_gb}GB required"
    fi
    log "Disk space OK: ${available_gb}GB available"
}

backup_database() {
    local db_name="$1"
    local backup_type="${2:-full}"
    local backup_dir="$BACKUP_BASE/$backup_type/$DATE"
    local backup_file="$backup_dir/${db_name}_${TIMESTAMP}.dump"
    
    mkdir -p "$backup_dir"
    
    log "Starting $backup_type backup of database: $db_name"
    
    # Execute backup
    pg_dump \
        -h "$DB_HOST" \
        -p "$DB_PORT" \
        -U "$DB_USER" \
        -Fc \
        --compress=6 \
        -d "$db_name" \
        -f "$backup_file"
    
    local backup_size=$(du -sh "$backup_file" | cut -f1)
    log "Backup complete: $backup_file ($backup_size)"
    
    # Upload to S3
    if [ -n "$S3_BUCKET" ] && command -v aws &>/dev/null; then
        log "Uploading to S3..."
        aws s3 cp "$backup_file" "$S3_BUCKET/$backup_type/$DATE/$(basename $backup_file)" \
            --storage-class STANDARD_IA
        log "Uploaded to S3: $S3_BUCKET/$backup_type/$DATE/"
    fi
    
    echo "$backup_file"
}

verify_backup() {
    local backup_file="$1"
    local test_db="backup_verify_$$"
    
    log "Verifying backup: $backup_file"
    
    # สร้าง test database
    createdb -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" "$test_db" || error "Failed to create test db"
    
    # Restore
    pg_restore \
        -h "$DB_HOST" \
        -p "$DB_PORT" \
        -U "$DB_USER" \
        -d "$test_db" \
        --no-owner \
        --no-privileges \
        "$backup_file" 2>/dev/null || true
    
    # Basic check
    local table_count=$(psql -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" \
        -d "$test_db" -t -c "SELECT COUNT(*) FROM information_schema.tables WHERE table_schema='public'" | tr -d ' ')
    
    # Cleanup
    dropdb -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" "$test_db"
    
    if [ "$table_count" -gt 0 ]; then
        log "Backup verification passed: $table_count tables restored"
        return 0
    else
        error "Backup verification failed: no tables found"
    fi
}

cleanup_old_backups() {
    local backup_dir="$1"
    local retention_days="${2:-7}"
    
    log "Cleaning up backups older than $retention_days days in $backup_dir"
    find "$backup_dir" -name "*.dump" -mtime "+$retention_days" -delete
    
    # Remove empty directories
    find "$backup_dir" -type d -empty -delete 2>/dev/null || true
}

backup_globals() {
    local backup_file="$BACKUP_BASE/globals/globals_${TIMESTAMP}.sql"
    mkdir -p "$BACKUP_BASE/globals"
    
    log "Backing up global objects (roles, tablespaces)"
    pg_dumpall \
        -h "$DB_HOST" \
        -p "$DB_PORT" \
        -U "$DB_USER" \
        --globals-only \
        -f "$backup_file"
    
    gzip "$backup_file"
    log "Globals backup: ${backup_file}.gz"
}

# Main
main() {
    log "========================================"
    log "PostgreSQL Backup Started"
    log "Host: $DB_HOST:$DB_PORT"
    log "========================================"
    
    # Pre-checks
    check_disk_space "$BACKUP_BASE" 20
    
    # Get list of databases to backup
    DATABASES=$(psql -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" \
        -t -c "SELECT datname FROM pg_database WHERE datistemplate = false AND datname NOT IN ('postgres')" \
        | tr -d ' ' | grep -v '^$')
    
    # Backup globals
    backup_globals
    
    # Backup each database
    local success_count=0
    local fail_count=0
    
    for db in $DATABASES; do
        if backup_file=$(backup_database "$db" "full"); then
            # Verify (optional, slow)
            # verify_backup "$backup_file"
            ((success_count++))
        else
            log "FAILED to backup: $db"
            ((fail_count++))
        fi
    done
    
    # Cleanup
    cleanup_old_backups "$BACKUP_BASE/full" "$RETENTION_DAYS"
    
    # Report
    log "========================================"
    log "Backup Summary:"
    log "  Successful: $success_count"
    log "  Failed: $fail_count"
    log "========================================"
    
    if [ "$fail_count" -gt 0 ]; then
        notify_failure "$fail_count database(s) failed to backup"
        exit 1
    else
        notify_success "All $success_count database(s) backed up successfully"
    fi
}

main "$@"
```

---

## 16. Backup Monitoring และ Alerting

```sql
-- Monitor backup status ใน PostgreSQL
-- ดู WAL archiver status
SELECT 
    archived_count,
    last_archived_wal,
    last_archived_time,
    failed_count,
    last_failed_wal,
    last_failed_time,
    stats_reset
FROM pg_stat_archiver;

-- ตรวจว่า archiving ล่าช้าหรือเปล่า
SELECT 
    CASE 
        WHEN last_archived_time < NOW() - INTERVAL '10 minutes' 
        THEN 'WARNING: Archiving delayed'
        ELSE 'OK'
    END AS archive_status,
    last_archived_time,
    failed_count
FROM pg_stat_archiver;

-- ดู WAL ที่ยังไม่ได้ archive
SELECT COUNT(*) AS pending_wals
FROM pg_ls_waldir()
WHERE name NOT LIKE '%.history'
AND name NOT LIKE '%.backup'
AND name NOT IN (
    SELECT last_archived_wal FROM pg_stat_archiver
);
```

```python
#!/usr/bin/env python3
# backup_monitor.py

import psycopg2
import boto3
import smtplib
from email.mime.text import MIMEText
from datetime import datetime, timedelta
import os

def check_backup_freshness(conn_string: str, max_age_hours: int = 25):
    """ตรวจสอบว่า backup ล่าสุดไม่เก่าเกิน max_age_hours"""
    
    # ตรวจ WAL archiver
    with psycopg2.connect(conn_string) as conn:
        with conn.cursor() as cur:
            cur.execute("""
                SELECT 
                    archived_count,
                    last_archived_time,
                    failed_count,
                    last_failed_time
                FROM pg_stat_archiver
            """)
            row = cur.fetchone()
            
            archived_count, last_archived, failed_count, last_failed = row
            
            issues = []
            
            if last_archived and last_archived < datetime.now().astimezone() - timedelta(hours=max_age_hours):
                issues.append(f"WAL archiving stale: last archived {last_archived}")
            
            if failed_count > 0:
                issues.append(f"Archive failures: {failed_count}, last failure: {last_failed}")
            
            return issues

def check_s3_backups(bucket: str, prefix: str, max_age_hours: int = 25):
    """ตรวจสอบ backup files ใน S3"""
    s3 = boto3.client('s3')
    cutoff = datetime.now().astimezone() - timedelta(hours=max_age_hours)
    
    response = s3.list_objects_v2(
        Bucket=bucket,
        Prefix=prefix
    )
    
    recent_backups = [
        obj for obj in response.get('Contents', [])
        if obj['LastModified'] > cutoff
    ]
    
    return len(recent_backups) > 0, recent_backups

def send_alert(subject: str, body: str):
    """ส่ง email alert"""
    msg = MIMEText(body)
    msg['Subject'] = subject
    msg['From'] = os.environ.get('ALERT_FROM', 'backup@example.com')
    msg['To'] = os.environ.get('ALERT_TO', 'admin@example.com')
    
    with smtplib.SMTP(os.environ.get('SMTP_HOST', 'localhost')) as server:
        server.send_message(msg)

def main():
    conn_string = os.environ.get('DATABASE_URL', 'postgresql://postgres@localhost/postgres')
    s3_bucket = os.environ.get('S3_BACKUP_BUCKET', 'my-backups')
    
    issues = []
    
    # ตรวจ WAL archiving
    wal_issues = check_backup_freshness(conn_string)
    issues.extend(wal_issues)
    
    # ตรวจ S3 backups
    has_recent, recent_backups = check_s3_backups(s3_bucket, 'backups/')
    if not has_recent:
        issues.append(f"No recent backups found in S3 (last 25 hours)")
    
    if issues:
        body = "Backup monitoring issues detected:\n\n"
        body += "\n".join(f"- {issue}" for issue in issues)
        body += f"\n\nHost: {os.uname().nodename}"
        body += f"\nTime: {datetime.now()}"
        
        send_alert("ALERT: Database Backup Issues", body)
        print("ALERT sent:", "\n".join(issues))
        exit(1)
    else:
        print(f"OK: {len(recent_backups)} recent backups found")

if __name__ == '__main__':
    main()
```

---

## 17. Backup Best Practices

```bash
# 1. ทดสอบ restore เป็นประจำ (อย่างน้อยสัปดาห์ละครั้ง)
# ============================================
# restore_test.sh
#!/bin/bash
DB_NAME="mydb"
BACKUP_FILE=$(ls -t /backup/daily/*.dump | head -1)
TEST_DB="${DB_NAME}_restore_test"

echo "Testing restore from: $BACKUP_FILE"

createdb $TEST_DB
pg_restore -d $TEST_DB $BACKUP_FILE

# Verify row counts
ORIGINAL_COUNT=$(psql -d $DB_NAME -t -c "SELECT reltuples::bigint FROM pg_class WHERE relname='users'")
RESTORED_COUNT=$(psql -d $TEST_DB -t -c "SELECT reltuples::bigint FROM pg_class WHERE relname='users'")

if [ "$ORIGINAL_COUNT" -eq "$RESTORED_COUNT" ]; then
    echo "PASS: Row counts match ($ORIGINAL_COUNT)"
else
    echo "FAIL: Row count mismatch (original: $ORIGINAL_COUNT, restored: $RESTORED_COUNT)"
fi

dropdb $TEST_DB

# 2. ใช้ checksums
# initdb --data-checksums
# หรือ pg_checksums --enable (offline)

# 3. ตรวจสอบ backup ด้วย pg_restore --list
pg_restore --list mydb_backup.dump | head -20

# 4. Store backup metadata
cat > /backup/metadata_${TIMESTAMP}.json << EOF
{
  "timestamp": "${TIMESTAMP}",
  "database": "${DB_NAME}",
  "host": "$(hostname)",
  "postgres_version": "$(pg_dump --version | head -1)",
  "size_bytes": $(stat -c%s "${BACKUP_FILE}"),
  "pg_lsn": "$(psql -t -c 'SELECT pg_current_wal_lsn()')",
  "tables": $(psql -t -c "SELECT COUNT(*) FROM information_schema.tables WHERE table_schema='public'" "$DB_NAME")
}
EOF
```

---

## สรุป

การ backup PostgreSQL ที่ดีต้องครอบคลุม:

1. **3-2-1 Rule**: สำรอง 3 copies, บน 2 media ต่างกัน, 1 ไว้ off-site
2. **pg_dump**: สำหรับ logical backup, portable, สามารถ restore บาง tables/schemas ได้
3. **pg_basebackup**: สำหรับ physical backup ของ cluster ทั้งหมด
4. **WAL Archiving**: สำหรับ PITR, RPO เป็นวินาที
5. **pgBackRest**: สำหรับ enterprise-grade backup management
6. **ทดสอบ restore** เป็นประจำ - backup ที่ไม่ผ่านการทดสอบไม่มีความหมาย
7. **Monitor** backup status, WAL archiver, disk space
8. **Encrypt** backups ก่อน store นอก premises
9. กำหนด **RTO/RPO** ก่อน แล้วเลือก strategy ที่เหมาะสม

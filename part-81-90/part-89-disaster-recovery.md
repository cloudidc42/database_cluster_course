# Part 89: Disaster Recovery Planning

## บทนำ

Disaster Recovery (DR) คือแผนและกระบวนการในการกู้คืนระบบหลังจากเหตุการณ์ร้ายแรง ไม่ว่าจะเป็น hardware failure, data corruption, หรือ cyber attack การวางแผน DR ที่ดีคือความแตกต่างระหว่างการสูญเสียธุรกิจและการรอดพ้น

---

## 1. DR Fundamentals

### RPO และ RTO

```
RPO (Recovery Point Objective): สูญเสียข้อมูลได้มากที่สุดแค่ไหน?
─────────────────────────────────────────────────────────
Last backup          Disaster occurs
     │                     │
     ├─────── RPO ──────────┤
     │                     │
  10:00 AM              10:30 AM
  (backup)              (failure)

ถ้า RPO = 30 minutes: ยอมรับการสูญเสียข้อมูล 30 นาที
ถ้า RPO = 0: ต้องการ zero data loss (synchronous replication)

RTO (Recovery Time Objective): ระบบต้องกลับมา online ภายในเวลาเท่าไหร่?
─────────────────────────────────────────────────────────
Disaster occurs       System restored
     │                     │
     ├────── RTO ───────────┤
     │                     │
  10:30 AM              11:30 AM
  (failure)             (recovery)

ถ้า RTO = 1 hour: ต้องกู้คืนภายใน 1 ชั่วโมง
```

### SLA Calculations

```python
# SLA Calculator
def calculate_downtime_per_year(availability_percentage: float) -> dict:
    """
    คำนวณ downtime ที่ยอมรับได้ต่อปี
    
    99.9% = 3 nines = 8.76 hours/year
    99.99% = 4 nines = 52.6 minutes/year
    99.999% = 5 nines = 5.26 minutes/year
    """
    minutes_per_year = 365 * 24 * 60  # 525,600 minutes
    
    downtime_minutes = minutes_per_year * (1 - availability_percentage / 100)
    
    return {
        'availability': f"{availability_percentage}%",
        'downtime_per_year_minutes': round(downtime_minutes, 2),
        'downtime_per_year_hours': round(downtime_minutes / 60, 2),
        'downtime_per_month_minutes': round(downtime_minutes / 12, 2),
        'downtime_per_week_minutes': round(downtime_minutes / 52, 2),
        'downtime_per_day_minutes': round(downtime_minutes / 365, 4),
    }

# ตัวอย่าง
for sla in [99.0, 99.9, 99.95, 99.99, 99.999]:
    info = calculate_downtime_per_year(sla)
    print(f"{info['availability']}: {info['downtime_per_year_hours']} hours/year")
```

```
SLA Table:
99.0%   = 87.6 hours/year (3.65 days)
99.9%   = 8.76 hours/year
99.95%  = 4.38 hours/year
99.99%  = 52.6 minutes/year
99.999% = 5.26 minutes/year
99.9999% = 31.5 seconds/year
```

### เลือก RPO/RTO ตาม Business Impact

```
ระดับ DR ตาม Business Impact:
─────────────────────────────────────────────────────────
Tier 1 (Critical):  Financial transactions, Healthcare
  RPO: Near-zero (< 1 minute)
  RTO: < 15 minutes
  Cost: สูงมาก (synchronous replication, hot standby)

Tier 2 (High):  E-commerce, Customer-facing apps
  RPO: < 15 minutes
  RTO: < 1 hour
  Cost: สูง (async replication, warm standby)

Tier 3 (Medium):  Internal tools, Analytics
  RPO: < 4 hours
  RTO: < 4 hours
  Cost: กลาง (daily backups + WAL archiving)

Tier 4 (Low):  Dev/Test, Archive
  RPO: < 24 hours
  RTO: < 24 hours
  Cost: ต่ำ (daily backups to S3)
```

---

## 2. Failure Scenarios

### ประเภทของ Disasters

```yaml
# disaster_scenarios.yml
disasters:
  hardware:
    - name: "Disk Failure"
      probability: high
      impact: medium
      recovery: "Replace disk, restore from backup/replica"
      rto_target: "30 minutes - 2 hours"
    
    - name: "Server Failure"
      probability: medium
      impact: high
      recovery: "Failover to replica/standby server"
      rto_target: "< 15 minutes (with HA setup)"
    
    - name: "Network Equipment Failure"
      probability: low
      impact: high
      recovery: "Redundant network paths"
      rto_target: "< 5 minutes"

  data_center:
    - name: "Power Outage"
      probability: low
      impact: high
      recovery: "UPS + Generator, or failover to DR site"
      rto_target: "< 30 minutes"
    
    - name: "Cooling System Failure"
      probability: very_low
      impact: critical
      recovery: "Failover to DR site"
      rto_target: "< 1 hour"

  data:
    - name: "Accidental Data Deletion"
      probability: medium
      impact: medium
      recovery: "PITR restore to point before deletion"
      rto_target: "1-4 hours"
    
    - name: "Data Corruption from Bug"
      probability: low
      impact: high
      recovery: "PITR restore + reapply valid transactions"
      rto_target: "2-8 hours"
    
    - name: "SQL Injection / Data Exfiltration"
      probability: low
      impact: critical
      recovery: "Isolate, assess damage, restore clean backup"
      rto_target: "4-24 hours"

  ransomware:
    - name: "Ransomware Attack"
      probability: low
      impact: critical
      recovery: "Air-gapped / immutable backup restore"
      rto_target: "24-72 hours"
      note: "ต้องมี offline backup ที่ไม่ถูก encrypt"
```

---

## 3. Backup Strategy for DR

### 3-2-1 Rule

```
3-2-1 Backup Rule:
─────────────────────────────────────────────────────────
3 = มีสำเนาข้อมูล 3 ชุด (1 primary + 2 backups)
2 = เก็บใน 2 media types ที่แตกต่างกัน (local disk + cloud)
1 = มีอย่างน้อย 1 ชุดที่อยู่ offsite (S3, glacier)

ตัวอย่าง:
Primary: PostgreSQL live data บน SSD
Backup 1: Local backup บน different disk/NAS
Backup 2: Remote backup บน AWS S3 (หรือ GCS, Azure Blob)

ปัจจุบัน: 3-2-1-1-0 Rule
3 copies
2 media types  
1 offsite
1 air-gapped (offline, ไม่เชื่อมต่อ network)
0 errors (test restores สม่ำเสมอ)
```

### S3 Object Lock: Immutable Backups

```python
# setup_immutable_backups.py
import boto3
from botocore.exceptions import ClientError

def setup_immutable_backup_bucket(bucket_name: str, retention_days: int = 30):
    """
    สร้าง S3 bucket สำหรับ immutable backups
    ใช้ Object Lock เพื่อป้องกัน ransomware
    """
    s3 = boto3.client('s3', region_name='ap-southeast-1')
    
    # สร้าง bucket พร้อม Object Lock
    s3.create_bucket(
        Bucket=bucket_name,
        CreateBucketConfiguration={'LocationConstraint': 'ap-southeast-1'},
        ObjectLockEnabledForBucket=True  # ต้องระบุตอนสร้าง
    )
    
    # ตั้งค่า default retention policy
    s3.put_object_lock_configuration(
        Bucket=bucket_name,
        ObjectLockConfiguration={
            'ObjectLockEnabled': 'Enabled',
            'Rule': {
                'DefaultRetention': {
                    'Mode': 'COMPLIANCE',  # COMPLIANCE = ลบไม่ได้แม้แต่ root user
                    'Days': retention_days
                }
            }
        }
    )
    
    # เปิด versioning (จำเป็นสำหรับ Object Lock)
    s3.put_bucket_versioning(
        Bucket=bucket_name,
        VersioningConfiguration={'Status': 'Enabled'}
    )
    
    # Block public access
    s3.put_public_access_block(
        Bucket=bucket_name,
        PublicAccessBlockConfiguration={
            'BlockPublicAcls': True,
            'IgnorePublicAcls': True,
            'BlockPublicPolicy': True,
            'RestrictPublicBuckets': True
        }
    )
    
    print(f"Created immutable backup bucket: {bucket_name}")
    print(f"Retention: {retention_days} days (COMPLIANCE mode)")
    print("WARNING: Files cannot be deleted during retention period!")

# ใช้งาน
setup_immutable_backup_bucket(
    bucket_name="myapp-db-backups-immutable",
    retention_days=30
)
```

---

## 4. PostgreSQL DR

### WAL Archiving ไปยัง S3

```ini
# postgresql.conf
wal_level = replica
archive_mode = on
archive_command = 'pgbackrest --stanza=main archive-push %p'
# หรือใช้ s3 directly:
archive_command = 'aws s3 cp %p s3://myapp-wal-archive/wal/%f'
archive_timeout = 300  # archive ทุก 5 นาที แม้ WAL ยังไม่เต็ม

# ตรวจสอบว่า archiving ทำงาน
restore_command = 'pgbackrest --stanza=main archive-get %f "%p"'
```

```bash
#!/bin/bash
# wal_archive_to_s3.sh - custom archive command

WAL_FILE="$1"
WAL_NAME="$2"
S3_BUCKET="s3://myapp-wal-archive"
HOSTNAME=$(hostname)

# Compress และ upload
gzip -c "${WAL_FILE}" | aws s3 cp - \
    "${S3_BUCKET}/wal/${HOSTNAME}/${WAL_NAME}.gz" \
    --storage-class STANDARD_IA \
    --metadata "hostname=${HOSTNAME},timestamp=$(date -u +%Y%m%dT%H%M%SZ)"

if [ $? -eq 0 ]; then
    echo "Archived WAL: ${WAL_NAME}"
    exit 0
else
    echo "ERROR: Failed to archive WAL: ${WAL_NAME}" >&2
    exit 1
fi
```

### PITR: Restore to Any Point in Time

```bash
#!/bin/bash
# pitr_restore.sh - Restore to specific point in time

TARGET_TIME="2024-01-15 14:30:00+07"  # เวลาก่อน incident
BACKUP_S3="s3://myapp-db-backups"
WAL_S3="s3://myapp-wal-archive"
RESTORE_DIR="/var/lib/postgresql/16/restore"

echo "=== PostgreSQL PITR Restore ==="
echo "Target time: ${TARGET_TIME}"
echo ""

# 1. หยุด PostgreSQL ปัจจุบัน
sudo systemctl stop postgresql

# 2. Backup ข้อมูลปัจจุบัน (safety net)
sudo mv /var/lib/postgresql/16/main /var/lib/postgresql/16/main.backup.$(date +%Y%m%d%H%M%S)

# 3. ดาวน์โหลด base backup ที่ใกล้เคียงที่สุด
echo "Downloading base backup..."
aws s3 sync "${BACKUP_S3}/base/latest/" "${RESTORE_DIR}/"
sudo chown -R postgres:postgres "${RESTORE_DIR}"
sudo chmod 700 "${RESTORE_DIR}"

# 4. สร้าง recovery configuration
cat > "${RESTORE_DIR}/postgresql.conf.restore" << EOF
# Recovery configuration
restore_command = 'aws s3 cp ${WAL_S3}/wal/$(hostname)/%f.gz - | gunzip > %p'
recovery_target_time = '${TARGET_TIME}'
recovery_target_action = 'promote'  # หลัง reach target time, promote เป็น primary
EOF

# 5. สร้าง recovery signal file
touch "${RESTORE_DIR}/recovery.signal"

# 6. Link configs
sudo -u postgres cp "${RESTORE_DIR}/postgresql.conf.restore" \
    "${RESTORE_DIR}/postgresql.conf"

# 7. Start PostgreSQL
sudo systemctl start postgresql

echo "PITR restore started. Monitoring recovery..."

# 8. Monitor recovery progress
while true; do
    STATE=$(sudo -u postgres psql -c "SELECT pg_is_in_recovery();" -t 2>/dev/null | tr -d ' ')
    if [ "$STATE" = "f" ]; then
        echo "Recovery complete! PostgreSQL is now primary."
        break
    fi
    
    LAST_WAL=$(sudo -u postgres psql -c "SELECT pg_last_wal_replay_lsn();" -t 2>/dev/null)
    echo "Still recovering... Last WAL: ${LAST_WAL}"
    sleep 10
done

# 9. Verify data
echo "Verifying restored data..."
sudo -u postgres psql -c "SELECT COUNT(*) FROM orders;" production
```

### pgBackRest: Enterprise Backup

```ini
# /etc/pgbackrest/pgbackrest.conf
[global]
repo1-type=s3
repo1-s3-bucket=myapp-pgbackrest
repo1-s3-endpoint=s3.ap-southeast-1.amazonaws.com
repo1-s3-region=ap-southeast-1
repo1-s3-key=AKIAIOSFODNN7EXAMPLE
repo1-s3-key-secret=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
repo1-retention-full=2      # เก็บ full backup 2 ชุด
repo1-retention-diff=7      # เก็บ diff backup 7 วัน
repo1-retention-archive=14  # เก็บ WAL archive 14 วัน
repo1-cipher-type=aes-256-cbc
repo1-cipher-pass=zWaf6XtpjIVZC5444yXB3zugNAQqZecology  # encrypt backups

compress-type=lz4
compress-level=6
process-max=4  # parallel processes

[main]
pg1-path=/var/lib/postgresql/16/main
pg1-port=5432
```

```bash
#!/bin/bash
# pgbackrest_operations.sh

# Initialize repository (ครั้งแรก)
pgbackrest --stanza=main stanza-create

# Full backup
pgbackrest --stanza=main backup --type=full

# Differential backup (เฉพาะส่วนที่เปลี่ยนจาก full)
pgbackrest --stanza=main backup --type=diff

# Incremental backup (เฉพาะส่วนที่เปลี่ยนจาก backup ล่าสุด)
pgbackrest --stanza=main backup --type=incr

# ดู backup history
pgbackrest --stanza=main info

# Restore ล่าสุด
pgbackrest --stanza=main restore

# PITR restore
pgbackrest --stanza=main restore \
    --target="2024-01-15 14:30:00.000000+07" \
    --target-action=promote

# Verify backup integrity (TEST ว่า backup ดีไหม)
pgbackrest --stanza=main check

# ตั้งค่า cron jobs
# Full backup: ทุกอาทิตย์
# 0 1 * * 0  pgbackrest --stanza=main backup --type=full

# Diff backup: ทุกวัน (ยกเว้นอาทิตย์)
# 0 1 * * 1-6  pgbackrest --stanza=main backup --type=diff
```

### Test Restore สม่ำเสมอ

```bash
#!/bin/bash
# test_restore.sh - Test backup integrity

TEST_DB_HOST="test-restore-server"
TEST_DB_PATH="/var/lib/postgresql/16/test_restore"
ALERT_EMAIL="dba@company.com"

echo "=== Backup Restore Test $(date) ==="

# 1. Restore backup ไปยัง test server
ssh postgres@${TEST_DB_HOST} "pgbackrest --stanza=main restore \
    --pg1-path=${TEST_DB_PATH} \
    --recovery-option='recovery_target_action=promote'"

# 2. Start test PostgreSQL
ssh postgres@${TEST_DB_HOST} "pg_ctl start -D ${TEST_DB_PATH}"
sleep 30  # รอให้ start

# 3. Run integrity checks
RESULT=$(ssh postgres@${TEST_DB_HOST} "psql -d production -c \"
    SELECT 
        'orders' AS table_name,
        COUNT(*) AS row_count,
        MAX(created_at) AS latest_record
    FROM orders
    UNION ALL
    SELECT 'customers', COUNT(*), MAX(created_at) FROM customers;
\" -t 2>&1")

echo "Restore Test Results:"
echo "$RESULT"

# 4. Check for errors
if echo "$RESULT" | grep -q "ERROR"; then
    echo "RESTORE TEST FAILED!"
    echo "$RESULT" | mail -s "DR Alert: Backup Restore Test FAILED" $ALERT_EMAIL
    exit 1
else
    echo "Restore test PASSED!"
    # Log result
    psql -d monitoring -c "
        INSERT INTO dr_test_results (test_date, result, details)
        VALUES (CURRENT_TIMESTAMP, 'PASS', '$RESULT')
    "
fi

# 5. Cleanup test server
ssh postgres@${TEST_DB_HOST} "
    pg_ctl stop -D ${TEST_DB_PATH}
    rm -rf ${TEST_DB_PATH}
"

echo "=== Test Complete ==="
```

---

## 5. Redis DR

### AOF Persistence Configuration

```ini
# redis.conf

# AOF (Append Only File) - เก็บทุก write operation
appendonly yes
appendfilename "appendonly.aof"

# Sync policy:
# always = sync ทุก write (slowest, safest - RPO = 0)
# everysec = sync ทุกวินาที (recommended - RPO = 1 second)  
# no = let OS decide (fastest, RPO = seconds to minutes)
appendfsync everysec

# Rewrite AOF เมื่อไฟล์ใหญ่เกิน 2x จาก size ก่อน rewrite
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

# RDB Snapshot เป็น additional safety net
save 900 1      # บันทึกถ้ามี 1 change ใน 900 วินาที
save 300 10     # บันทึกถ้ามี 10 changes ใน 300 วินาที
save 60 10000   # บันทึกถ้ามี 10000 changes ใน 60 วินาที

dbfilename dump.rdb
dir /var/lib/redis/
```

### Redis Backup to S3

```bash
#!/bin/bash
# redis_backup_s3.sh

REDIS_CLI="redis-cli -h redis -a ${REDIS_PASSWORD}"
S3_BUCKET="s3://myapp-redis-backups"
DATE=$(date +%Y%m%d_%H%M%S)

echo "=== Redis Backup to S3 ==="

# 1. Trigger RDB snapshot
echo "Triggering BGSAVE..."
$REDIS_CLI BGSAVE

# รอให้ save เสร็จ
while [ "$($REDIS_CLI LASTSAVE)" == "$LAST_SAVE" ]; do
    sleep 1
done

echo "Snapshot complete."

# 2. Copy RDB ไปยัง temp location
RDB_FILE=$($REDIS_CLI CONFIG GET dir | tail -1)/$($REDIS_CLI CONFIG GET dbfilename | tail -1)
BACKUP_FILE="/tmp/redis_backup_${DATE}.rdb.gz"

gzip -c "${RDB_FILE}" > "${BACKUP_FILE}"

# 3. Upload ไปยัง S3
aws s3 cp "${BACKUP_FILE}" \
    "${S3_BUCKET}/$(hostname)/${DATE}/redis_backup.rdb.gz" \
    --storage-class STANDARD_IA

# 4. Copy AOF
AOF_FILE=$($REDIS_CLI CONFIG GET dir | tail -1)/appendonly.aof
gzip -c "${AOF_FILE}" > "/tmp/redis_aof_${DATE}.aof.gz"
aws s3 cp "/tmp/redis_aof_${DATE}.aof.gz" \
    "${S3_BUCKET}/$(hostname)/${DATE}/appendonly.aof.gz"

# 5. Cleanup temp files
rm -f "${BACKUP_FILE}" "/tmp/redis_aof_${DATE}.aof.gz"

# 6. Delete backups เก่ากว่า 7 วัน
aws s3 ls "${S3_BUCKET}/$(hostname)/" | \
    awk '{print $2}' | \
    head -n -7 | \
    while read OLD_DIR; do
        aws s3 rm --recursive "${S3_BUCKET}/$(hostname)/${OLD_DIR}"
    done

echo "Redis backup complete: ${DATE}"
```

### Redis Sentinel for Automatic Failover

```yaml
# docker-compose-redis-sentinel.yml
version: '3.8'
services:
  redis-master:
    image: redis:7-alpine
    command: redis-server --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis-master-data:/data
    ports:
      - "6379:6379"

  redis-replica-1:
    image: redis:7-alpine
    command: >
      redis-server 
      --replicaof redis-master 6379
      --requirepass ${REDIS_PASSWORD}
      --masterauth ${REDIS_PASSWORD}
    depends_on:
      - redis-master

  redis-replica-2:
    image: redis:7-alpine
    command: >
      redis-server 
      --replicaof redis-master 6379
      --requirepass ${REDIS_PASSWORD}
      --masterauth ${REDIS_PASSWORD}
    depends_on:
      - redis-master

  sentinel-1:
    image: redis:7-alpine
    command: >
      redis-sentinel /etc/sentinel.conf
    volumes:
      - ./sentinel.conf:/etc/sentinel.conf
    depends_on:
      - redis-master
      - redis-replica-1
      - redis-replica-2

  sentinel-2:
    image: redis:7-alpine
    command: redis-sentinel /etc/sentinel.conf
    volumes:
      - ./sentinel.conf:/etc/sentinel.conf
    depends_on:
      - redis-master

  sentinel-3:
    image: redis:7-alpine
    command: redis-sentinel /etc/sentinel.conf
    volumes:
      - ./sentinel.conf:/etc/sentinel.conf
    depends_on:
      - redis-master

volumes:
  redis-master-data:
```

```ini
# sentinel.conf
sentinel monitor mymaster redis-master 6379 2
sentinel auth-pass mymaster ${REDIS_PASSWORD}
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 60000
sentinel parallel-syncs mymaster 1

# แจ้งเตือนเมื่อ failover เกิดขึ้น
sentinel notification-script mymaster /scripts/notify_failover.sh
sentinel client-reconfig-script mymaster /scripts/update_config.sh
```

---

## 6. DR Testing

### Game Days: Simulate Failures

```python
# gameday_scenarios.py
"""
Game Day Scenarios: ฝึกซ้อม DR ด้วยสถานการณ์จริง

ทำ Game Day ทุก quarter เพื่อ:
1. ตรวจสอบว่า DR plan ยังใช้ได้
2. ฝึก team ให้คุ้นเคยกับกระบวนการ
3. ค้นหา gaps ใน DR process
4. Update runbooks
"""

GAME_DAY_SCENARIOS = [
    {
        'name': 'Database Primary Failure',
        'description': 'Kill primary PostgreSQL, verify replica takes over',
        'steps': [
            '1. Monitor: start tracking metrics',
            '2. Kill primary: sudo systemctl stop postgresql',
            '3. Verify: replica promotes automatically (Patroni/repmgr)',
            '4. Verify: application reconnects',
            '5. Measure: RTO actual vs. target',
            '6. Document: issues found'
        ],
        'expected_rto': '< 60 seconds',
        'blast_radius': 'staging environment only'
    },
    {
        'name': 'Redis Failure',
        'description': 'Kill Redis master, verify Sentinel failover',
        'steps': [
            '1. Monitor: start tracking cache hit rate',
            '2. Kill master: docker kill redis-master',
            '3. Verify: Sentinel promotes replica',
            '4. Verify: application uses new master',
            '5. Verify: cache hit rate recovers',
            '6. Measure: downtime duration'
        ],
        'expected_rto': '< 30 seconds',
        'blast_radius': 'staging environment'
    },
    {
        'name': 'Accidental Table Drop',
        'description': 'DROP TABLE, restore via PITR',
        'steps': [
            '1. Record current time (T1)',
            '2. DROP TABLE staging.test_table',
            '3. Verify: app gets errors',
            '4. Execute PITR restore to T1 - 1 minute',
            '5. Verify: table restored',
            '6. Measure: total recovery time'
        ],
        'expected_rto': '< 30 minutes',
        'blast_radius': 'staging environment'
    }
]

def run_gameday_scenario(scenario: dict, dry_run: bool = True):
    """Execute game day scenario"""
    print(f"\n{'='*60}")
    print(f"GAME DAY SCENARIO: {scenario['name']}")
    print(f"Expected RTO: {scenario['expected_rto']}")
    print(f"Blast Radius: {scenario['blast_radius']}")
    print(f"{'='*60}")
    
    if dry_run:
        print("\n[DRY RUN] Steps that would be executed:")
        for step in scenario['steps']:
            print(f"  {step}")
        return
    
    import time
    start_time = time.time()
    
    print("\nExecuting scenario...")
    for step in scenario['steps']:
        print(f"\n{step}")
        input("Press Enter when step is complete...")
    
    elapsed = time.time() - start_time
    print(f"\nTotal time: {elapsed:.0f} seconds")
    print(f"Target: {scenario['expected_rto']}")
```

---

## 7. Runbook สำหรับ DR

### Declare Incident

```markdown
# Runbook: Database Disaster Recovery

## Step 1: Declare Incident

เมื่อตรวจพบ database failure:

1. **Alert ทุกคน**
   - Slack: #incidents channel
   - PagerDuty: ส่ง P1 alert
   - แจ้ง Engineering Manager

2. **เปิด Incident Channel**
   - สร้าง Slack channel: #inc-YYYYMMDD-db-failure
   - Invite: DBA, Senior Engineer, DevOps, Engineering Manager

3. **Assign Roles**
   - Incident Commander: ควบคุม incident ทั้งหมด
   - Technical Lead: execute recovery
   - Communications: อัปเดต stakeholders
   - Scribe: บันทึกทุกอย่าง

4. **Start Incident Timer**
   - บันทึกเวลาที่ detect และ declare incident
   - ตั้ง timer สำหรับ status updates ทุก 15 นาที
```

### Assess Scope

```bash
#!/bin/bash
# assess_incident.sh - รวบรวม information อย่างรวดเร็ว

echo "=== INCIDENT ASSESSMENT ==="
echo "Time: $(date)"
echo ""

# Database status
echo "--- PostgreSQL Status ---"
sudo systemctl status postgresql --no-pager
echo ""

# Replication status
echo "--- Replication ---"
psql -U postgres -c "SELECT * FROM pg_stat_replication;" 2>/dev/null || echo "Cannot connect to primary"
echo ""

# Check replicas
for REPLICA in replica-1 replica-2; do
    echo "--- Replica: ${REPLICA} ---"
    psql -h ${REPLICA} -U postgres -c "SELECT pg_is_in_recovery(), pg_last_wal_replay_lsn();" 2>/dev/null || echo "Cannot connect to ${REPLICA}"
done
echo ""

# Redis status
echo "--- Redis Status ---"
redis-cli -h redis ping 2>/dev/null || echo "Redis not responding"
echo ""

# Disk space
echo "--- Disk Space ---"
df -h /var/lib/postgresql/
echo ""

# Active connections
echo "--- Active Connections ---"
psql -U postgres -c "SELECT count(*) FROM pg_stat_activity WHERE state != 'idle';" 2>/dev/null
echo ""

# Recent errors in logs
echo "--- Recent Errors ---"
sudo tail -50 /var/log/postgresql/postgresql-$(date +%Y-%m-%d).log | grep -E "ERROR|FATAL|PANIC"

echo "=== Assessment Complete ==="
```

### Execute Recovery

```bash
#!/bin/bash
# execute_recovery.sh - DR recovery script

RECOVERY_TYPE="${1:-replica_failover}"  # replica_failover, pitr_restore, backup_restore

case "${RECOVERY_TYPE}" in
    "replica_failover")
        echo "=== Executing Replica Failover ==="
        
        # หา replica ที่ lag น้อยที่สุด
        BEST_REPLICA=$(psql -h replica-1 -U postgres -t -c "
            SELECT pg_last_wal_replay_lsn();" 2>/dev/null)
        
        # Promote replica
        if command -v patronictl &>/dev/null; then
            # ใช้ Patroni
            patronictl -c /etc/patroni/config.yml failover main \
                --master old-primary \
                --candidate replica-1 \
                --force
        elif command -v repmgr &>/dev/null; then
            # ใช้ repmgr
            repmgr standby promote -f /etc/repmgr.conf
        else
            # Manual promote
            sudo -u postgres pg_ctl promote -D /var/lib/postgresql/16/main
        fi
        
        echo "Failover complete! Verify application connectivity..."
        ;;
    
    "pitr_restore")
        echo "=== Executing PITR Restore ==="
        read -p "Enter target time (YYYY-MM-DD HH:MM:SS+TZ): " TARGET_TIME
        
        sudo systemctl stop postgresql
        
        # Restore via pgBackRest
        pgbackrest --stanza=main restore \
            --target="${TARGET_TIME}" \
            --target-action=promote
        
        sudo systemctl start postgresql
        
        echo "PITR restore initiated. Monitor recovery..."
        ;;
    
    *)
        echo "Unknown recovery type: ${RECOVERY_TYPE}"
        echo "Usage: $0 [replica_failover|pitr_restore|backup_restore]"
        exit 1
        ;;
esac
```

### Verify Data Integrity

```sql
-- verify_integrity.sql
-- รันหลังจาก recovery เสร็จ

-- 1. ตรวจสอบ table counts
SELECT 
    schemaname,
    tablename,
    n_live_tup AS row_count
FROM pg_stat_user_tables
ORDER BY n_live_tup DESC
LIMIT 20;

-- 2. ตรวจสอบ latest records
SELECT 
    'orders' AS table_name,
    MAX(created_at) AS latest_record,
    COUNT(*) AS total_rows
FROM orders
UNION ALL
SELECT 'customers', MAX(created_at), COUNT(*) FROM customers;

-- 3. ตรวจสอบ sequences
SELECT 
    sequence_name,
    last_value
FROM information_schema.sequences
WHERE sequence_schema = 'public';

-- 4. Test basic operations
BEGIN;
INSERT INTO orders (customer_id, amount, status) 
VALUES (1, 100.00, 'test_recovery') RETURNING id;
DELETE FROM orders WHERE status = 'test_recovery';
COMMIT;
SELECT 'Basic operations OK' AS status;

-- 5. ตรวจสอบ foreign key integrity
SELECT 
    conname AS constraint_name,
    conrelid::regclass AS table_name
FROM pg_constraint
WHERE contype = 'f'
    AND NOT convalidated;
```

---

## 8. Multi-Region DR

### Architecture

```yaml
# multi_region_dr.yml
regions:
  primary:
    region: ap-southeast-1  # Singapore
    components:
      - PostgreSQL primary
      - Redis master
      - Application servers
      - Load balancer
    
  dr:
    region: ap-southeast-2  # Sydney (or ap-northeast-1 Tokyo)
    components:
      - PostgreSQL replica (streaming replication)
      - Redis replica
      - Application servers (standby, scaled down)
      - Load balancer (ready)
    
  failover_trigger:
    method: Route53 Health Checks
    check_interval: 30s
    failure_threshold: 3
    recovery_threshold: 2
    
  rpo_target: 5_minutes
  rto_target: 15_minutes
```

### Route53 Health Check Failover

```python
# setup_route53_failover.py
import boto3

route53 = boto3.client('route53')

def setup_failover_dns(
    hosted_zone_id: str,
    domain: str,
    primary_ip: str,
    dr_ip: str
):
    """Setup Route53 failover routing"""
    
    # สร้าง health check สำหรับ primary
    health_check = route53.create_health_check(
        CallerReference=f"hc-primary-{domain}",
        HealthCheckConfig={
            'IPAddress': primary_ip,
            'Port': 5432,
            'Type': 'TCP',
            'RequestInterval': 30,
            'FailureThreshold': 3
        }
    )
    
    health_check_id = health_check['HealthCheck']['Id']
    
    # Primary record (FAILOVER = PRIMARY)
    route53.change_resource_record_sets(
        HostedZoneId=hosted_zone_id,
        ChangeBatch={
            'Changes': [
                {
                    'Action': 'CREATE',
                    'ResourceRecordSet': {
                        'Name': domain,
                        'Type': 'A',
                        'SetIdentifier': 'primary',
                        'Failover': 'PRIMARY',
                        'TTL': 30,
                        'ResourceRecords': [{'Value': primary_ip}],
                        'HealthCheckId': health_check_id
                    }
                },
                {
                    'Action': 'CREATE',
                    'ResourceRecordSet': {
                        'Name': domain,
                        'Type': 'A',
                        'SetIdentifier': 'secondary',
                        'Failover': 'SECONDARY',
                        'TTL': 30,
                        'ResourceRecords': [{'Value': dr_ip}]
                        # ไม่มี health check สำหรับ secondary
                    }
                }
            ]
        }
    )
    
    print(f"DNS failover configured for {domain}")
    print(f"Primary: {primary_ip}")
    print(f"DR: {dr_ip}")
    print(f"Health Check ID: {health_check_id}")

# ใช้งาน
setup_failover_dns(
    hosted_zone_id="Z3ABCDEF12345",
    domain="db.myapp.com",
    primary_ip="10.0.1.100",
    dr_ip="10.1.1.100"
)
```

---

## 9. DR Metrics

### MTTR และ MTBF

```python
# dr_metrics.py
from datetime import datetime, timedelta
from typing import list
import statistics

def calculate_mttr(incidents: list[dict]) -> float:
    """
    Mean Time To Recovery (MTTR)
    = เฉลี่ยเวลาที่ใช้กู้คืนระบบ
    
    Target: < RTO
    """
    recovery_times = []
    
    for incident in incidents:
        if incident['resolved_at'] and incident['occurred_at']:
            duration = (incident['resolved_at'] - incident['occurred_at']).total_seconds() / 60
            recovery_times.append(duration)
    
    if not recovery_times:
        return 0
    
    return statistics.mean(recovery_times)

def calculate_mtbf(incidents: list[dict]) -> float:
    """
    Mean Time Between Failures (MTBF)
    = เฉลี่ยเวลาระหว่าง incidents
    
    Target: สูงที่สุด (น้อยครั้งที่สุด)
    """
    if len(incidents) < 2:
        return float('inf')
    
    sorted_incidents = sorted(incidents, key=lambda x: x['occurred_at'])
    
    gaps = []
    for i in range(1, len(sorted_incidents)):
        gap = (sorted_incidents[i]['occurred_at'] - 
               sorted_incidents[i-1]['occurred_at']).total_seconds() / 3600
        gaps.append(gap)
    
    return statistics.mean(gaps)

def calculate_availability(incidents: list[dict], period_days: int = 30) -> float:
    """
    System Availability = (Total Time - Downtime) / Total Time
    """
    total_minutes = period_days * 24 * 60
    
    downtime_minutes = sum([
        (i['resolved_at'] - i['occurred_at']).total_seconds() / 60
        for i in incidents
        if i['resolved_at']
    ])
    
    return (total_minutes - downtime_minutes) / total_minutes * 100

# ตัวอย่าง
incidents = [
    {
        'id': 1,
        'occurred_at': datetime(2024, 1, 5, 14, 30),
        'resolved_at': datetime(2024, 1, 5, 15, 15),
        'type': 'primary_failure'
    },
    {
        'id': 2,
        'occurred_at': datetime(2024, 1, 18, 3, 0),
        'resolved_at': datetime(2024, 1, 18, 3, 45),
        'type': 'disk_full'
    }
]

print(f"MTTR: {calculate_mttr(incidents):.1f} minutes")
print(f"MTBF: {calculate_mtbf(incidents):.1f} hours")
print(f"Availability: {calculate_availability(incidents):.4f}%")
```

---

## สรุป

Disaster Recovery ที่ดีต้องมี:

1. **RPO/RTO ที่ชัดเจน** - รู้ว่าระบบทนสูญเสียข้อมูลและ downtime ได้แค่ไหน
2. **Backup ที่หลากหลาย** - 3-2-1 rule, immutable backups
3. **WAL Archiving + PITR** - กู้คืนได้ทุก point in time
4. **Replica Failover** - HA ลด downtime จาก hours เป็น minutes
5. **Test Restores** - ทดสอบ restore สม่ำเสมอ (รายเดือน)
6. **Game Days** - ฝึกซ้อม DR ด้วย real scenarios
7. **Runbooks** - ขั้นตอนที่ชัดเจน ทุกคนทำตามได้
8. **Multi-region** - ป้องกัน data center failure

**Remember: Backup ที่ไม่ได้ test คือ backup ที่ไม่มีค่า!**

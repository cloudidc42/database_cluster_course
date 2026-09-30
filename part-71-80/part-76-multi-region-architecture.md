# Part 76: Multi-Region Database Architecture

## บทนำ: ทำไมต้องใช้ Multi-Region?

ในยุคที่ผู้ใช้งานกระจายตัวอยู่ทั่วโลก การออกแบบระบบฐานข้อมูลให้รองรับหลาย Region ถือเป็นสิ่งจำเป็นสำหรับแอปพลิเคชันระดับ Enterprise มาดูเหตุผลหลักที่ทำให้ต้องใช้ Multi-Region Architecture

### 1. Latency (ความหน่วงเวลา)

```
ระยะทางระหว่าง Data Centers → ความหน่วงเวลา
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Singapore → Tokyo:       ~85ms
Singapore → London:      ~180ms
Singapore → New York:    ~230ms
Singapore → São Paulo:   ~300ms

Speed of Light in Fiber: ~200,000 km/s
Singapore to New York: ~15,000 km → ~75ms one-way (theoretical minimum)
Real-world: ~150-230ms RTT (Round Trip Time)
```

สำหรับผู้ใช้ในยุโรปที่ต้อง query ฐานข้อมูลใน Singapore ทุกครั้งที่กดปุ่ม จะได้รับประสบการณ์ที่แย่มาก การมี replica ในยุโรปสามารถลด latency จาก 180ms เหลือ ~5ms ได้

### 2. High Availability (ความพร้อมใช้งานสูง)

```
Single Region Failure Scenarios:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
- Natural disaster (earthquake, flood, fire)
- Power outage (data center generator failure)
- Network partition (ISP outage)
- Region-wide cloud provider issues

AWS us-east-1 outages (historical):
- 2011: 4-day EBS outage
- 2012: Multiple outages
- 2020: US-EAST-1 outage affecting thousands of services
- 2021: AWS us-east-1 outage (December)

ด้วย Multi-Region: traffic สามารถ failover ไปยัง region อื่นได้อัตโนมัติ
SLA: 99.99% uptime (52.6 min downtime/year) → 99.999% (5.26 min/year)
```

### 3. Compliance (การปฏิบัติตามกฎหมาย)

กฎหมายหลายฉบับกำหนดว่าข้อมูลต้องเก็บอยู่ใน region ที่เฉพาะเจาะจง:

```
GDPR (EU):
- ข้อมูลของพลเมืองยุโรปต้องอยู่ใน EU หรือประเทศที่ EU รับรอง
- ละเมิด: ปรับสูงสุด €20 ล้าน หรือ 4% ของรายได้ทั่วโลก

PDPA (Thailand):
- ข้อมูลส่วนบุคคลของคนไทยในบางกรณีต้องอยู่ในไทย
- โดยเฉพาะข้อมูลสุขภาพ, ข้อมูลทางการเงิน

Data Localization Laws by Country:
- Russia: Federal Law 242-FZ (must store locally)
- China: Cybersecurity Law (critical data must stay in China)
- India: Data Protection Bill (financial data)
- Brazil: LGPD (similar to GDPR)
```

---

## Regions vs Availability Zones

ความแตกต่างระหว่าง Region และ Availability Zone เป็นสิ่งสำคัญในการออกแบบ:

```
AWS Infrastructure:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Region: ap-southeast-1 (Singapore)
├── Availability Zone: ap-southeast-1a
│   └── Data Center: DC-A1, DC-A2
├── Availability Zone: ap-southeast-1b
│   └── Data Center: DC-B1
└── Availability Zone: ap-southeast-1c
    └── Data Center: DC-C1, DC-C2

Region: us-east-1 (N. Virginia)
├── Availability Zone: us-east-1a
├── Availability Zone: us-east-1b
├── Availability Zone: us-east-1c
├── Availability Zone: us-east-1d
├── Availability Zone: us-east-1e
└── Availability Zone: us-east-1f
```

```
ความแตกต่างหลัก:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Availability Zones:
- ระยะห่าง: 10-100 km ภายใน region เดียวกัน
- Latency: <2ms ระหว่าง AZs
- Network: เชื่อมต่อด้วย low-latency, high-bandwidth links
- Use case: HA ภายใน region (rds multi-az, patroni)

Regions:
- ระยะห่าง: ต่างทวีปหรือต่างประเทศ
- Latency: 50-300ms ระหว่าง regions
- Network: Internet backbone หรือ dedicated fiber
- Use case: Geo-distribution, DR, compliance
```

### เมื่อใดใช้ Multi-AZ vs Multi-Region?

```
Multi-AZ (แนะนำเสมอสำหรับ production):
✓ ป้องกัน data center failure
✓ Latency ต่ำ → synchronous replication ได้
✓ Automatic failover ภายในไม่กี่วินาที
✓ ราคาถูกกว่า Multi-Region

Multi-Region (จำเป็นเมื่อ):
✓ ผู้ใช้อยู่หลายทวีป (latency ต้องการ <50ms)
✓ Compliance ต้องการ data localization
✓ RTO/RPO requirement ต่ำมาก (ป้องกัน region-wide disaster)
✓ Business continuity สำหรับระบบ critical มาก
```

---

## Data Residency Requirements

### GDPR - General Data Protection Regulation

```sql
-- ตัวอย่าง: บันทึก user location สำหรับ compliance
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) NOT NULL,
    data_residency VARCHAR(10) NOT NULL, -- 'EU', 'US', 'TH', etc.
    created_at TIMESTAMPTZ DEFAULT NOW(),
    
    -- ข้อมูลที่ต้องอยู่ใน EU
    personal_data JSONB, -- encrypted
    
    -- Compliance tracking
    gdpr_consent_given BOOLEAN DEFAULT FALSE,
    gdpr_consent_date TIMESTAMPTZ,
    data_processing_purposes TEXT[],
    
    CONSTRAINT check_residency CHECK (
        data_residency IN ('EU', 'US', 'APAC', 'TH', 'AU')
    )
);

-- Row Level Security สำหรับ data isolation
ALTER TABLE users ENABLE ROW LEVEL SECURITY;

CREATE POLICY eu_data_policy ON users
    FOR ALL
    USING (
        current_setting('app.region') = 'EU' 
        AND data_residency = 'EU'
    );

CREATE POLICY us_data_policy ON users
    FOR ALL
    USING (
        current_setting('app.region') = 'US'
        AND data_residency = 'US'
    );
```

### PDPA (Thailand) - พระราชบัญญัติคุ้มครองข้อมูลส่วนบุคคล

```typescript
// pdpa-compliance.ts
interface PDPAConfig {
  dataController: string;      // บริษัทที่เป็นเจ้าของข้อมูล
  dataProcessor?: string;      // บริษัทที่ประมวลผลข้อมูล
  dataRetentionDays: number;   // ระยะเวลาเก็บข้อมูล
  requiredConsent: boolean;    // ต้องขอความยินยอมหรือไม่
  crossBorderTransfer: boolean; // มีการส่งข้อมูลข้ามชาติหรือไม่
}

interface UserDataRecord {
  userId: string;
  dataType: 'personal' | 'sensitive' | 'financial' | 'health';
  storedRegion: 'th' | 'sg' | 'us' | 'eu';
  consentGiven: boolean;
  consentDate: Date | null;
  retentionExpiry: Date;
  purposes: string[];
}

class PDPAComplianceManager {
  private config: PDPAConfig;
  
  constructor(config: PDPAConfig) {
    this.config = config;
  }
  
  async validateDataStorage(record: UserDataRecord): Promise<void> {
    // ตรวจสอบว่าข้อมูล sensitive ต้องอยู่ในไทย
    if (record.dataType === 'sensitive' || record.dataType === 'health') {
      if (record.storedRegion !== 'th') {
        throw new Error(
          `PDPA Violation: Sensitive data must be stored in Thailand region, ` +
          `but found in '${record.storedRegion}'`
        );
      }
    }
    
    // ตรวจสอบว่ามี consent สำหรับ cross-border transfer
    if (this.config.crossBorderTransfer && record.storedRegion !== 'th') {
      if (!record.consentGiven) {
        throw new Error(
          'PDPA Violation: Cross-border data transfer requires explicit consent'
        );
      }
    }
    
    // ตรวจสอบ retention period
    const now = new Date();
    if (record.retentionExpiry < now) {
      throw new Error(
        `PDPA Violation: Data retention period expired on ${record.retentionExpiry.toISOString()}`
      );
    }
  }
  
  generateDataSubjectRightsReport(userId: string): PDPAReport {
    return {
      rightToAccess: true,
      rightToRectification: true,
      rightToErasure: true,
      rightToPortability: true,
      rightToObject: true,
      rightToRestrictProcessing: true,
    };
  }
}
```

---

## Multi-Region Patterns

### Pattern 1: Active-Passive

ใน Active-Passive pattern จะมี Primary Region ที่รับ writes ทั้งหมด และ Passive Region ที่รับ reads:

```
Active-Passive Architecture:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

                    ┌─────────────────┐
                    │   Load Balancer  │
                    │   (Route53/CF)   │
                    └────────┬────────┘
                             │
              ┌──────────────┴──────────────┐
              │ Writes                       │ Reads (nearest)
              ▼                              ▼
   ┌──────────────────────┐    ┌─────────────────────────────┐
   │  PRIMARY REGION      │    │  SECONDARY REGION(S)        │
   │  (Singapore)         │    │  (Europe / US / etc.)       │
   │                      │    │                             │
   │  PostgreSQL Primary  │───▶│  PostgreSQL Read Replica    │
   │  (READ + WRITE)      │    │  (READ ONLY)                │
   │                      │    │                             │
   │  Redis Primary       │───▶│  Redis Replica              │
   │                      │    │                             │
   └──────────────────────┘    └─────────────────────────────┘
         │                              │
         │  Async Replication           │
         │  (seconds of lag)            │
         └──────────────────────────────┘
```

**ข้อดีและข้อเสีย:**
```
ข้อดี:
✓ ง่ายต่อการ implement
✓ ไม่มี write conflicts
✓ ข้อมูล consistent (single write path)
✓ Cost ต่ำกว่า Active-Active

ข้อเสีย:
✗ Write latency สูงสำหรับผู้ใช้ห่าง primary region
✗ Passive region เป็นแค่ read-only ใช้ได้ครึ่งเดียว
✗ Failover ต้องการ manual intervention (หรือ automated แต่ซับซ้อน)
✗ RPO = replication lag (อาจ seconds to minutes)
```

### Pattern 2: Active-Active

ใน Active-Active ทุก Region รับ reads และ writes:

```
Active-Active Architecture:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

      User (EU)                    User (Asia)
         │                              │
         ▼                              ▼
  ┌──────────────┐               ┌──────────────┐
  │  EU Region   │◄─────────────▶│  Asia Region │
  │              │ Bi-directional │              │
  │  Write+Read  │   Replication  │  Write+Read  │
  │              │                │              │
  └──────────────┘                └──────────────┘
         │                              │
         └──────────────────────────────┘
               Conflict Resolution
               (Last-Write-Wins / CRDT / Application Logic)
```

**ข้อดีและข้อเสีย:**
```
ข้อดี:
✓ Write latency ต่ำสำหรับทุก region
✓ ใช้ capacity ได้เต็มประสิทธิภาพ
✓ No single point of failure

ข้อเสีย:
✗ Write conflicts: 2 users แก้ไขข้อมูลเดียวกันพร้อมกัน
✗ Complex conflict resolution logic
✗ Harder to maintain consistency
✗ CAP theorem: ต้องเลือกระหว่าง Availability vs Consistency
✗ Cost สูงกว่า
```

### Pattern 3: Follow-the-Sun

Write region ตามเวลากลางวันของแต่ละ region:

```
Follow-the-Sun Pattern:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

UTC 00:00-08:00 → Primary: Asia Pacific (Singapore/Tokyo)
UTC 08:00-16:00 → Primary: Europe (Frankfurt/London)  
UTC 16:00-24:00 → Primary: Americas (US-East/US-West)

                     00:00 UTC
                    ┌─────────────────┐
                    │     APAC        │◄── ACTIVE WRITES
                    │   (Singapore)   │
                    └─────────────────┘
                           │
         ┌─────────────────┼─────────────────┐
         ▼                 ▼                 ▼
  ┌──────────┐     ┌──────────┐     ┌──────────┐
  │  Europe  │     │    US    │     │  Passive │
  │ (passive)│     │ (passive)│     │   Read   │
  └──────────┘     └──────────┘     └──────────┘
  
                     08:00 UTC
                    ┌─────────────────┐
                    │    EUROPE       │◄── ACTIVE WRITES
                    │  (Frankfurt)    │
                    └─────────────────┘
```

```typescript
// follow-the-sun-router.ts
interface RegionSchedule {
  region: string;
  endpoint: string;
  activePeriodUTCStart: number; // hour in UTC
  activePeriodUTCEnd: number;
}

class FollowTheSunRouter {
  private schedule: RegionSchedule[] = [
    {
      region: 'APAC',
      endpoint: 'postgres://sg-primary.example.com:5432/mydb',
      activePeriodUTCStart: 0,
      activePeriodUTCEnd: 8,
    },
    {
      region: 'EU',
      endpoint: 'postgres://eu-primary.example.com:5432/mydb',
      activePeriodUTCStart: 8,
      activePeriodUTCEnd: 16,
    },
    {
      region: 'US',
      endpoint: 'postgres://us-primary.example.com:5432/mydb',
      activePeriodUTCStart: 16,
      activePeriodUTCEnd: 24,
    },
  ];
  
  getCurrentWriteEndpoint(): string {
    const currentHourUTC = new Date().getUTCHours();
    
    const activeRegion = this.schedule.find(
      r => currentHourUTC >= r.activePeriodUTCStart && 
           currentHourUTC < r.activePeriodUTCEnd
    );
    
    if (!activeRegion) {
      // fallback to APAC
      return this.schedule[0].endpoint;
    }
    
    return activeRegion.endpoint;
  }
  
  async rotateWriteRegion(fromRegion: string, toRegion: string): Promise<void> {
    console.log(`Rotating write primary from ${fromRegion} to ${toRegion}`);
    
    // 1. Wait for replication to catch up
    await this.waitForReplicationSync(fromRegion, toRegion);
    
    // 2. Promote new primary
    await this.promoteRegion(toRegion);
    
    // 3. Demote old primary to replica
    await this.demoteRegion(fromRegion);
    
    // 4. Update DNS/routing
    await this.updateDNS(toRegion);
    
    console.log(`Write rotation complete. New primary: ${toRegion}`);
  }
  
  private async waitForReplicationSync(from: string, to: string): Promise<void> {
    // Implementation: check replication lag < 100ms
    const maxWait = 30000; // 30 seconds
    const startTime = Date.now();
    
    while (Date.now() - startTime < maxWait) {
      const lag = await this.getReplicationLag(from, to);
      if (lag < 100) {
        return; // Synced
      }
      await new Promise(resolve => setTimeout(resolve, 1000));
    }
    
    throw new Error(`Replication sync timeout between ${from} and ${to}`);
  }
  
  private async getReplicationLag(from: string, to: string): Promise<number> {
    // Returns lag in milliseconds
    return 0; // placeholder
  }
  
  private async promoteRegion(region: string): Promise<void> {
    // Implementation specific to your setup (Patroni, AWS RDS, etc.)
  }
  
  private async demoteRegion(region: string): Promise<void> {
    // Implementation: reconfigure as replica of new primary
  }
  
  private async updateDNS(newPrimaryRegion: string): Promise<void> {
    // Update Route53 / Cloudflare DNS records
  }
}
```

---

## Active-Passive Implementation สำหรับ PostgreSQL

### Architecture Overview

```
Active-Passive PostgreSQL Multi-Region:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Singapore (Primary Region):
├── PostgreSQL Primary (Read/Write)
│   └── Patroni cluster (3 nodes: primary + 2 standbys)
└── PgBouncer (connection pooling)

Frankfurt (Secondary Region):
├── PostgreSQL Standby (Read-only replica)
│   └── Streaming replication from Singapore
└── PgBouncer (connection pooling for reads)

Tokyo (Secondary Region):
├── PostgreSQL Standby (Read-only replica)
└── PgBouncer (connection pooling for reads)

Application Logic:
- Writes → Singapore
- Reads → Nearest region (eu users → Frankfurt, asia users → Singapore/Tokyo)
```

### PostgreSQL Streaming Replication Setup

```bash
# บน Primary Server (Singapore)
# postgresql.conf
cat > /etc/postgresql/16/main/postgresql.conf << 'EOF'
# Replication Settings
wal_level = replica
max_wal_senders = 10
wal_keep_size = 1GB
max_replication_slots = 10

# สำหรับ synchronous_commit ที่ยืดหยุ่น
synchronous_commit = on    # ค่าปกติสำหรับ local AZ

# Network
listen_addresses = '*'
wal_sender_timeout = 60s

# Logging
log_replication_commands = on
EOF

# สร้าง replication slot สำหรับ Frankfurt
psql -c "SELECT pg_create_physical_replication_slot('frankfurt_replica');"
psql -c "SELECT pg_create_physical_replication_slot('tokyo_replica');"
```

```bash
# บน Replica Server (Frankfurt)
# pg_basebackup เพื่อสร้าง replica
pg_basebackup \
  --host=sg-primary.example.com \
  --port=5432 \
  --username=replication_user \
  --pgdata=/var/lib/postgresql/16/main \
  --wal-method=stream \
  --slot=frankfurt_replica \
  --write-recovery-conf \
  --checkpoint=fast \
  --progress \
  --verbose

# postgresql.conf บน replica
cat >> /var/lib/postgresql/16/main/postgresql.conf << 'EOF'
# Hot standby settings
hot_standby = on
hot_standby_feedback = on

# Recovery
recovery_target_timeline = 'latest'

# Monitoring
wal_receiver_status_interval = 10s
wal_receiver_timeout = 60s
EOF

# postgresql.auto.conf (เขียนโดย pg_basebackup)
cat > /var/lib/postgresql/16/main/postgresql.auto.conf << 'EOF'
primary_conninfo = 'host=sg-primary.example.com port=5432 user=replication_user password=secret sslmode=require application_name=frankfurt_replica'
primary_slot_name = 'frankfurt_replica'
EOF
```

### Monitoring Replication Lag

```sql
-- บน Primary: ดู replication status
SELECT
    client_addr,
    application_name,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    -- Lag คำนวณจาก WAL position difference
    (sent_lsn - replay_lsn) AS replication_lag_bytes,
    -- Lag คำนวณจากเวลา
    extract(epoch from (now() - reply_time)) AS lag_seconds,
    sync_state
FROM pg_stat_replication
ORDER BY application_name;

-- ดู replication slots
SELECT
    slot_name,
    active,
    restart_lsn,
    -- WAL ที่ยังไม่ได้ส่ง (อาจทำให้ disk เต็ม!)
    (pg_current_wal_lsn() - restart_lsn) AS retained_bytes,
    pg_size_pretty((pg_current_wal_lsn() - restart_lsn)) AS retained_size
FROM pg_replication_slots;
```

```typescript
// replication-monitor.ts
import { Pool } from 'pg';
import { Gauge } from 'prom-client';

interface ReplicaStatus {
  applicationName: string;
  lagBytes: number;
  lagSeconds: number;
  state: string;
  syncState: string;
}

class ReplicationMonitor {
  private primaryPool: Pool;
  private replicationLagGauge: Gauge;
  
  constructor(primaryConnectionString: string) {
    this.primaryPool = new Pool({ connectionString: primaryConnectionString });
    
    this.replicationLagGauge = new Gauge({
      name: 'postgresql_replication_lag_seconds',
      help: 'Replication lag in seconds for each replica',
      labelNames: ['replica_name', 'region'],
    });
  }
  
  async checkReplicationStatus(): Promise<ReplicaStatus[]> {
    const result = await this.primaryPool.query<ReplicaStatus>(`
      SELECT
        application_name,
        state,
        sync_state,
        (sent_lsn - replay_lsn) AS lag_bytes,
        EXTRACT(epoch FROM (now() - reply_time)) AS lag_seconds
      FROM pg_stat_replication
      WHERE state = 'streaming'
      ORDER BY application_name
    `);
    
    const statuses = result.rows;
    
    // Update Prometheus metrics
    for (const status of statuses) {
      const region = this.getRegionFromAppName(status.applicationName);
      this.replicationLagGauge.set(
        { replica_name: status.applicationName, region },
        status.lagSeconds
      );
    }
    
    return statuses;
  }
  
  async alertIfLagExceedsThreshold(
    thresholdSeconds: number = 30
  ): Promise<void> {
    const statuses = await this.checkReplicationStatus();
    
    for (const status of statuses) {
      if (status.lagSeconds > thresholdSeconds) {
        await this.sendAlert({
          severity: 'warning',
          message: `Replication lag for ${status.applicationName} is ${status.lagSeconds}s (threshold: ${thresholdSeconds}s)`,
          replica: status.applicationName,
          lagSeconds: status.lagSeconds,
        });
      }
    }
  }
  
  private getRegionFromAppName(appName: string): string {
    const regionMap: Record<string, string> = {
      'frankfurt_replica': 'eu',
      'tokyo_replica': 'apac',
      'sydney_replica': 'apac',
    };
    return regionMap[appName] || 'unknown';
  }
  
  private async sendAlert(alert: {
    severity: string;
    message: string;
    replica: string;
    lagSeconds: number;
  }): Promise<void> {
    console.error(`ALERT [${alert.severity}]: ${alert.message}`);
    // ส่ง PagerDuty, Slack, หรือ email notification
  }
}
```

---

## DNS Failover: Route53 / Cloudflare

### AWS Route53 Health Checks + Failover Routing

```typescript
// route53-failover.ts
import { 
  Route53Client, 
  ChangeResourceRecordSetsCommand,
  CreateHealthCheckCommand,
  GetHealthCheckStatusCommand
} from '@aws-sdk/client-route-53';

class Route53FailoverManager {
  private client: Route53Client;
  private hostedZoneId: string;
  
  constructor(hostedZoneId: string) {
    this.client = new Route53Client({ region: 'us-east-1' });
    this.hostedZoneId = hostedZoneId;
  }
  
  async createHealthCheck(endpoint: string): Promise<string> {
    const command = new CreateHealthCheckCommand({
      CallerReference: `health-check-${Date.now()}`,
      HealthCheckConfig: {
        IPAddress: endpoint,
        Port: 5432,
        Type: 'TCP',
        RequestInterval: 10,   // seconds
        FailureThreshold: 3,   // consecutive failures before unhealthy
      },
    });
    
    const response = await this.client.send(command);
    return response.HealthCheck!.Id!;
  }
  
  async setupFailoverRouting(
    primaryIp: string,
    secondaryIp: string,
    recordName: string
  ): Promise<void> {
    const primaryHealthCheckId = await this.createHealthCheck(primaryIp);
    
    const command = new ChangeResourceRecordSetsCommand({
      HostedZoneId: this.hostedZoneId,
      ChangeBatch: {
        Changes: [
          // Primary record (active)
          {
            Action: 'CREATE',
            ResourceRecordSet: {
              Name: recordName,
              Type: 'A',
              Failover: 'PRIMARY',
              SetIdentifier: 'primary-sg',
              TTL: 60,
              ResourceRecords: [{ Value: primaryIp }],
              HealthCheckId: primaryHealthCheckId,
            },
          },
          // Secondary record (failover target)
          {
            Action: 'CREATE',
            ResourceRecordSet: {
              Name: recordName,
              Type: 'A',
              Failover: 'SECONDARY',
              SetIdentifier: 'secondary-eu',
              TTL: 60,
              ResourceRecords: [{ Value: secondaryIp }],
            },
          },
        ],
      },
    });
    
    await this.client.send(command);
    console.log(`Failover routing configured: ${primaryIp} → ${secondaryIp}`);
  }
  
  async triggerManualFailover(
    primaryRecord: string,
    secondaryIp: string
  ): Promise<void> {
    console.log(`Initiating manual failover to ${secondaryIp}...`);
    
    // Step 1: Promote secondary to primary
    await this.promotePostgresReplica(secondaryIp);
    
    // Step 2: Update DNS to point to new primary
    await this.updatePrimaryRecord(primaryRecord, secondaryIp);
    
    console.log('Failover complete!');
  }
  
  private async promotePostgresReplica(replicaIp: string): Promise<void> {
    // Execute pg_promote() on the replica
    // หรือใช้ patronictl failover <cluster-name>
    console.log(`Promoting replica at ${replicaIp}...`);
  }
  
  private async updatePrimaryRecord(
    recordName: string, 
    newIp: string
  ): Promise<void> {
    const command = new ChangeResourceRecordSetsCommand({
      HostedZoneId: this.hostedZoneId,
      ChangeBatch: {
        Changes: [{
          Action: 'UPSERT',
          ResourceRecordSet: {
            Name: recordName,
            Type: 'A',
            TTL: 30,  // ลด TTL ระหว่าง failover
            ResourceRecords: [{ Value: newIp }],
          },
        }],
      },
    });
    
    await this.client.send(command);
  }
}
```

### Cloudflare DNS Failover

```typescript
// cloudflare-failover.ts
interface CloudflareRecord {
  id: string;
  name: string;
  type: string;
  content: string;
  ttl: number;
  proxied: boolean;
}

class CloudflareFailoverManager {
  private apiToken: string;
  private zoneId: string;
  private baseUrl = 'https://api.cloudflare.com/client/v4';
  
  constructor(apiToken: string, zoneId: string) {
    this.apiToken = apiToken;
    this.zoneId = zoneId;
  }
  
  private async cfFetch(path: string, options: RequestInit = {}): Promise<any> {
    const response = await fetch(`${this.baseUrl}${path}`, {
      ...options,
      headers: {
        'Authorization': `Bearer ${this.apiToken}`,
        'Content-Type': 'application/json',
        ...options.headers,
      },
    });
    
    const data = await response.json();
    if (!data.success) {
      throw new Error(`Cloudflare API error: ${JSON.stringify(data.errors)}`);
    }
    return data;
  }
  
  async getRecord(name: string): Promise<CloudflareRecord> {
    const data = await this.cfFetch(
      `/zones/${this.zoneId}/dns_records?name=${name}&type=A`
    );
    
    if (!data.result || data.result.length === 0) {
      throw new Error(`DNS record not found: ${name}`);
    }
    
    return data.result[0];
  }
  
  async failoverTo(recordName: string, newIp: string): Promise<void> {
    const record = await this.getRecord(recordName);
    
    // อัพเดต DNS record ไปยัง IP ใหม่
    await this.cfFetch(
      `/zones/${this.zoneId}/dns_records/${record.id}`,
      {
        method: 'PUT',
        body: JSON.stringify({
          type: 'A',
          name: recordName,
          content: newIp,
          ttl: 60,    // 1 minute สำหรับ quick propagation
          proxied: false,
        }),
      }
    );
    
    console.log(`DNS record ${recordName} updated to ${newIp}`);
    console.log(`DNS propagation will complete in ~60 seconds`);
  }
}
```

---

## Active-Active Challenges

### Write Conflicts

```
Write Conflict Scenario:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

User A (in EU) updates account balance:
  T1: balance = 100 → writes 120 to EU region
  
User B (in US) updates same account:
  T1: balance = 100 → writes 90 to US region
  
After replication:
  EU sees: 90 (from US) → conflict with 120
  US sees: 120 (from EU) → conflict with 90
  
What's the real balance? 120? 90? Something else?
```

**Conflict Resolution Strategies:**

```typescript
// conflict-resolution.ts

// Strategy 1: Last-Write-Wins (LWW)
// ง่ายที่สุด แต่อาจเสียข้อมูล
interface LWWRecord {
  id: string;
  value: any;
  timestamp: number;  // milliseconds since epoch
  regionId: string;
}

function resolveConflictLWW(local: LWWRecord, remote: LWWRecord): LWWRecord {
  if (remote.timestamp > local.timestamp) {
    return remote;
  }
  if (remote.timestamp === local.timestamp) {
    // Tie-break: ใช้ region ID เพื่อ deterministic result
    return remote.regionId > local.regionId ? remote : local;
  }
  return local;
}

// Strategy 2: Vector Clocks (Causality tracking)
type VectorClock = Record<string, number>;

interface VersionedRecord {
  id: string;
  value: any;
  vectorClock: VectorClock;
}

function compareVectorClocks(vc1: VectorClock, vc2: VectorClock): 
  'happens-before' | 'happens-after' | 'concurrent' {
  
  const regions = new Set([...Object.keys(vc1), ...Object.keys(vc2)]);
  let vc1Dominates = false;
  let vc2Dominates = false;
  
  for (const region of regions) {
    const t1 = vc1[region] || 0;
    const t2 = vc2[region] || 0;
    
    if (t1 > t2) vc1Dominates = true;
    if (t2 > t1) vc2Dominates = true;
  }
  
  if (vc1Dominates && !vc2Dominates) return 'happens-before';
  if (vc2Dominates && !vc1Dominates) return 'happens-after';
  return 'concurrent'; // CONFLICT!
}

// Strategy 3: Application-Level Merge (for counters)
interface CounterRecord {
  id: string;
  // แยก increment ต่อ region
  increments: Record<string, number>;
}

function mergeCounters(local: CounterRecord, remote: CounterRecord): CounterRecord {
  const mergedIncrements: Record<string, number> = {};
  
  const regions = new Set([
    ...Object.keys(local.increments),
    ...Object.keys(remote.increments)
  ]);
  
  for (const region of regions) {
    // Max ของแต่ละ region (CRDT G-Counter)
    mergedIncrements[region] = Math.max(
      local.increments[region] || 0,
      remote.increments[region] || 0
    );
  }
  
  return {
    id: local.id,
    increments: mergedIncrements,
  };
}

function getCounterValue(counter: CounterRecord): number {
  return Object.values(counter.increments).reduce((sum, v) => sum + v, 0);
}
```

---

## Global Database Services

### AWS Aurora Global Database

```
Aurora Global Database Architecture:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Primary Region (Singapore):
├── Writer Instance (Read/Write)
├── Reader Instance (Read Only - local AZ)
└── Replication to Global Storage

Secondary Regions (Frankfurt, Tokyo, Sydney):
├── Reader Instances only (for now)
└── Can be promoted to Primary on failure

Key Features:
- Sub-second replication (typically 1 second or less)
- Up to 5 secondary regions
- RPO: < 1 second
- RTO: < 1 minute (managed failover)
- Read from secondary: typical 1-2s lag
```

```sql
-- AWS Aurora Global Database
-- สร้างผ่าน AWS Console หรือ CLI

-- ตรวจสอบ replication status
SELECT *
FROM aurora_global_db_instance_status();

-- ดู lag
SELECT
    server_id,
    durable_lsn,
    highest_lsn_rcvd,
    feedback_epoch,
    feedback_xmin
FROM aurora_global_db_status();
```

```bash
# AWS CLI commands
# สร้าง Global Database
aws rds create-global-cluster \
  --global-cluster-identifier my-global-cluster \
  --engine aurora-postgresql \
  --engine-version 15.4 \
  --storage-encrypted

# เพิ่ม Secondary Region
aws rds create-db-cluster \
  --db-cluster-identifier my-cluster-eu \
  --engine aurora-postgresql \
  --global-cluster-identifier my-global-cluster \
  --region eu-west-1

# Failover ไปยัง secondary region
aws rds failover-global-cluster \
  --global-cluster-identifier my-global-cluster \
  --target-db-cluster-identifier arn:aws:rds:eu-west-1:123456789012:cluster:my-cluster-eu
```

### Google Cloud Spanner

```
Spanner Architecture:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Spanner Instance (Multi-region)
├── Configuration: nam6 (Iowa + South Carolina + Oklahoma)
│   └── Automatically replicated across all zones
│
├── Table: users
│   └── Row-level replication: each row replicated to all nodes
│
└── TrueTime API
    ├── GPS Clock + Atomic Clock
    └── Uncertainty window: <7ms
    └── Enables external consistency (linearizability)

SQL-like but distributed:
- Strong consistency across ALL regions
- ACID transactions globally
- Scales horizontally (up to petabytes)
- Price: expensive (~$0.90/node/hour)
```

```sql
-- Google Spanner DDL
CREATE TABLE users (
  user_id   STRING(36) NOT NULL,
  email     STRING(256) NOT NULL,
  name      STRING(256),
  region    STRING(10) NOT NULL,
  created_at TIMESTAMP NOT NULL OPTIONS (allow_commit_timestamp=true),
) PRIMARY KEY (user_id);

-- Interleaved table (co-located with parent for efficiency)
CREATE TABLE user_orders (
  user_id   STRING(36) NOT NULL,
  order_id  STRING(36) NOT NULL,
  total     NUMERIC,
  status    STRING(20),
) PRIMARY KEY (user_id, order_id),
INTERLEAVE IN PARENT users ON DELETE CASCADE;

-- Index
CREATE INDEX idx_users_email ON users (email);
CREATE INDEX idx_users_region ON users (region);
```

### CockroachDB Multi-Region

```sql
-- CockroachDB Multi-Region Configuration
-- กำหนด regions ที่ใช้
ALTER DATABASE mydb ADD REGION "us-east1";
ALTER DATABASE mydb ADD REGION "eu-west1";
ALTER DATABASE mydb ADD REGION "ap-southeast1";

-- Primary region
ALTER DATABASE mydb SET PRIMARY REGION "us-east1";

-- Table with regional settings
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email STRING NOT NULL,
    region crdb_internal_region AS (
        CASE
            WHEN email LIKE '%@eu.%' THEN 'eu-west1'
            WHEN email LIKE '%@asia.%' THEN 'ap-southeast1'
            ELSE 'us-east1'
        END
    ) STORED
) LOCALITY REGIONAL BY ROW;

-- Global table (replicated everywhere, reads are fast anywhere)
CREATE TABLE currency_rates (
    currency STRING PRIMARY KEY,
    rate_usd DECIMAL(10, 6),
    updated_at TIMESTAMPTZ
) LOCALITY GLOBAL;

-- Regional table (data stays in one region)
CREATE TABLE audit_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    action STRING NOT NULL,
    created_at TIMESTAMPTZ DEFAULT now()
) LOCALITY REGIONAL IN PRIMARY REGION;
```

---

## Multi-Region Redis

### Redis Enterprise Active-Active (CRDT-based)

```
Redis Enterprise Active-Active:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Singapore Cluster ←──CRDT sync──→ Frankfurt Cluster
     │                                    │
     ├── Shard 1                         ├── Shard 1 (replica)
     ├── Shard 2                         ├── Shard 2 (replica)
     └── Shard 3                         └── Shard 3 (replica)

CRDT Data Structures:
- Counter: increment/decrement (G-Counter, PN-Counter)
- Set: add/remove members (OR-Set)
- Sorted Set: scores merged automatically
- Hash: field-level merging

Use Cases:
- Shopping cart (Add to cart from any region)
- View counters
- Session data (read from nearest region)
- Rate limiting (per-region counters merged)
```

```typescript
// redis-multiregion.ts
import { createClient } from 'redis';

interface RegionClient {
  region: string;
  client: ReturnType<typeof createClient>;
}

class MultiRegionRedis {
  private clients: RegionClient[];
  private localRegion: string;
  
  constructor(connections: Array<{ region: string; url: string }>, localRegion: string) {
    this.localRegion = localRegion;
    this.clients = connections.map(conn => ({
      region: conn.region,
      client: createClient({ url: conn.url }),
    }));
  }
  
  async connect(): Promise<void> {
    await Promise.all(this.clients.map(c => c.client.connect()));
  }
  
  // Read from local region (low latency)
  private getLocalClient(): ReturnType<typeof createClient> {
    const local = this.clients.find(c => c.region === this.localRegion);
    if (!local) throw new Error(`No client for region: ${this.localRegion}`);
    return local.client;
  }
  
  async get(key: string): Promise<string | null> {
    // Always read from local region for low latency
    return this.getLocalClient().get(key);
  }
  
  async set(
    key: string, 
    value: string, 
    options?: { expireSeconds?: number }
  ): Promise<void> {
    // Write to local region (Active-Active: each region handles its writes)
    const localClient = this.getLocalClient();
    
    if (options?.expireSeconds) {
      await localClient.setEx(key, options.expireSeconds, value);
    } else {
      await localClient.set(key, value);
    }
    
    // Redis Enterprise handles cross-region sync automatically (CRDT)
  }
  
  // Counter: safe to increment from any region (CRDT PN-Counter)
  async incrBy(key: string, amount: number): Promise<number> {
    return this.getLocalClient().incrBy(key, amount);
  }
  
  // Session management: store in local region
  async setSession(sessionId: string, data: object, ttlSeconds: number = 3600): Promise<void> {
    const key = `session:${sessionId}`;
    await this.getLocalClient().setEx(key, ttlSeconds, JSON.stringify(data));
  }
  
  async getSession(sessionId: string): Promise<object | null> {
    const key = `session:${sessionId}`;
    const data = await this.getLocalClient().get(key);
    return data ? JSON.parse(data) : null;
  }
}
```

---

## Multi-Region Object Storage (S3)

### S3 Cross-Region Replication

```typescript
// s3-multi-region.ts
import {
  S3Client,
  PutBucketReplicationCommand,
  CreateBucketCommand,
  PutBucketVersioningCommand,
} from '@aws-sdk/client-s3';

class S3MultiRegionSetup {
  async setupCrossRegionReplication(
    sourceBucket: string,
    sourceRegion: string,
    destBucket: string,
    destRegion: string,
    roleArn: string
  ): Promise<void> {
    const sourceClient = new S3Client({ region: sourceRegion });
    
    // Enable versioning (required for replication)
    await sourceClient.send(new PutBucketVersioningCommand({
      Bucket: sourceBucket,
      VersioningConfiguration: { Status: 'Enabled' },
    }));
    
    // Configure replication
    await sourceClient.send(new PutBucketReplicationCommand({
      Bucket: sourceBucket,
      ReplicationConfiguration: {
        Role: roleArn,
        Rules: [
          {
            ID: `replicate-to-${destRegion}`,
            Status: 'Enabled',
            Filter: {
              Prefix: '',  // replicate everything
            },
            Destination: {
              Bucket: `arn:aws:s3:::${destBucket}`,
              StorageClass: 'STANDARD',
              ReplicationTime: {
                Status: 'Enabled',
                Time: { Minutes: 15 },  // 99.99% in 15 minutes
              },
              Metrics: {
                Status: 'Enabled',
                EventThreshold: { Minutes: 15 },
              },
            },
            DeleteMarkerReplication: { Status: 'Enabled' },
          },
        ],
      },
    }));
    
    console.log(`S3 CRR configured: ${sourceBucket} → ${destBucket}`);
  }
}

// CloudFront Distribution สำหรับ global CDN
class CloudFrontGlobalCDN {
  async createDistribution(
    originDomain: string,
    regions: string[]
  ): Promise<string> {
    // ใช้ Origin Groups สำหรับ failover
    // Origin Group: Primary (us-east-1 bucket) + Secondary (eu-west-1 bucket)
    
    const distributionConfig = {
      Origins: {
        Quantity: 2,
        Items: [
          {
            Id: 'primary-origin',
            DomainName: `${originDomain}.s3.amazonaws.com`,
            S3OriginConfig: { OriginAccessIdentity: '' },
          },
          {
            Id: 'failover-origin',
            DomainName: `${originDomain}-eu.s3.eu-west-1.amazonaws.com`,
            S3OriginConfig: { OriginAccessIdentity: '' },
          },
        ],
      },
      OriginGroups: {
        Quantity: 1,
        Items: [{
          Id: 'origin-group-1',
          FailoverCriteria: {
            StatusCodes: {
              Quantity: 3,
              Items: [500, 502, 503],
            },
          },
          Members: {
            Quantity: 2,
            Items: [
              { OriginId: 'primary-origin' },
              { OriginId: 'failover-origin' },
            ],
          },
        }],
      },
      DefaultCacheBehavior: {
        ViewerProtocolPolicy: 'redirect-to-https',
        CachePolicyId: '658327ea-f89d-4fab-a63d-7e88639e58f6', // Managed-CachingOptimized
        TargetOriginId: 'origin-group-1',
      },
    };
    
    console.log('CloudFront distribution configured for global CDN');
    return 'E1234EXAMPLE'; // distribution ID
  }
}
```

---

## DR vs Multi-Region HA

```
Disaster Recovery vs Multi-Region HA:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Disaster Recovery (DR):
━━━━━━━━━━━━━━━━━━━━━━━━
Goal: รับมือกับ disaster (ไฟไหม้, แผ่นดินไหว, ระเบิด)
RTO: ชั่วโมง - วัน (acceptable downtime)
RPO: นาที - ชั่วโมง (acceptable data loss)
Cost: ต่ำ (passive standby, minimal compute)
Traffic: ไม่รับ traffic ปกติ (standby mode)
Test: ทดสอบ 1-2 ครั้งต่อปี (failover drill)

Strategies:
1. Backup & Restore: RTO=hours, RPO=daily
2. Pilot Light: RTO=minutes, RPO=minutes  
3. Warm Standby: RTO=minutes, RPO=seconds
4. Multi-Site Active-Active: RTO=0, RPO=0

Multi-Region HA (High Availability):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Goal: ให้บริการต่อเนื่อง, latency ต่ำ
RTO: วินาที - นาที (near-zero downtime)
RPO: วินาที หรือ 0 (no data loss)
Cost: สูง (active resources in all regions)
Traffic: รับ traffic ปกติ (active serving)
Test: ทดสอบตลอดเวลา (production traffic)

เลือกใช้อะไร:
- ถ้า SLA ต้องการ 99.9% (8.7 hr/yr downtime): Multi-AZ เพียงพอ
- ถ้า SLA ต้องการ 99.99% (52 min/yr downtime): Multi-Region HA
- ถ้า SLA ต้องการ 99.999% (5 min/yr downtime): Active-Active Multi-Region
- ถ้าแค่ต้องการ disaster protection: DR Warm Standby
```

### Cost Comparison

```
Cost Model (approximate AWS, production-grade):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Single Region (db.r6g.xlarge, 1 primary + 2 replicas):
- Compute: $0.48/hr × 3 = $1.44/hr = $1,051/month
- Storage: $0.115/GB/month × 500GB = $57.5/month
- Total: ~$1,100/month

Active-Passive (Singapore primary + Frankfurt read replica):
- Singapore: $1,100/month
- Frankfurt read replica: $700/month (smaller)
- Data transfer: ~$50/month
- Total: ~$1,850/month (+68%)

Active-Active (3 regions, full capacity each):
- Singapore: $1,100/month
- Frankfurt: $1,100/month  
- US-East: $1,100/month
- Data transfer: ~$200/month
- Total: ~$3,500/month (+218%)
```

---

## Monitoring Multi-Region Replication

### Prometheus + Grafana Dashboard

```yaml
# prometheus-rules.yaml
groups:
  - name: multi_region_replication
    interval: 30s
    rules:
      - alert: ReplicationLagHigh
        expr: |
          postgresql_replication_lag_seconds > 30
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Replication lag is high for {{ $labels.replica_name }}"
          description: "Lag is {{ $value }}s (threshold: 30s)"
      
      - alert: ReplicationLagCritical
        expr: |
          postgresql_replication_lag_seconds > 300
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Replication lag critical for {{ $labels.replica_name }}"
          description: "Lag is {{ $value }}s - data may be significantly behind"
      
      - alert: ReplicaDown
        expr: |
          absent(postgresql_replication_lag_seconds{replica_name="frankfurt_replica"})
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Frankfurt replica is down"
          description: "No replication metrics received from frankfurt_replica"
```

```typescript
// multi-region-health-check.ts
interface RegionHealth {
  region: string;
  status: 'healthy' | 'degraded' | 'down';
  latencyMs: number;
  replicationLagSeconds: number;
  lastCheckAt: Date;
}

class MultiRegionHealthMonitor {
  private regions: Array<{
    name: string;
    pgHost: string;
    redisHost: string;
  }>;
  
  constructor() {
    this.regions = [
      { name: 'singapore', pgHost: 'sg-db.example.com', redisHost: 'sg-redis.example.com' },
      { name: 'frankfurt', pgHost: 'eu-db.example.com', redisHost: 'eu-redis.example.com' },
      { name: 'us-east', pgHost: 'us-db.example.com', redisHost: 'us-redis.example.com' },
    ];
  }
  
  async checkAllRegions(): Promise<RegionHealth[]> {
    const results = await Promise.allSettled(
      this.regions.map(r => this.checkRegion(r))
    );
    
    return results.map((result, i) => {
      if (result.status === 'fulfilled') {
        return result.value;
      }
      return {
        region: this.regions[i].name,
        status: 'down' as const,
        latencyMs: -1,
        replicationLagSeconds: -1,
        lastCheckAt: new Date(),
      };
    });
  }
  
  private async checkRegion(region: {
    name: string;
    pgHost: string;
    redisHost: string;
  }): Promise<RegionHealth> {
    const start = Date.now();
    
    // Check PostgreSQL connectivity
    const { Pool } = await import('pg');
    const pool = new Pool({ host: region.pgHost, database: 'postgres', user: 'monitor' });
    
    try {
      await pool.query('SELECT 1');
      const latencyMs = Date.now() - start;
      
      // Check replication lag
      const lagResult = await pool.query(`
        SELECT EXTRACT(epoch FROM (now() - pg_last_xact_replay_timestamp())) AS lag_seconds
      `);
      
      const lagSeconds = lagResult.rows[0]?.lag_seconds || 0;
      
      const status: 'healthy' | 'degraded' | 'down' = 
        lagSeconds > 60 ? 'degraded' :
        latencyMs > 500 ? 'degraded' : 'healthy';
      
      return {
        region: region.name,
        status,
        latencyMs,
        replicationLagSeconds: lagSeconds,
        lastCheckAt: new Date(),
      };
    } finally {
      await pool.end();
    }
  }
}
```

---

## สรุป Multi-Region Architecture

```
Decision Framework:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. คุณมีผู้ใช้ในหลายทวีป?
   → ใช่: พิจารณา Multi-Region
   → ไม่: Multi-AZ เพียงพอ

2. Write latency สำคัญสำหรับทุก region?
   → ใช่: Active-Active (ซับซ้อน, แพง)
   → ไม่: Active-Passive (เรียบง่าย, ถูกกว่า)

3. มี compliance requirement?
   → GDPR/PDPA: ต้องเก็บข้อมูลในภูมิภาคที่กำหนด
   → ใช้ Row-Level Security + Data Residency routing

4. Budget?
   → Limited: Active-Passive (reads จาก nearest region)
   → Unlimited: Active-Active (CockroachDB, Spanner)

Best Practice Checklist:
✓ Measure actual latency requirements ก่อน architect
✓ Start with Active-Passive (simpler, cheaper)
✓ Use DNS failover (Route53/Cloudflare) สำหรับ automatic failover
✓ Monitor replication lag ตลอดเวลา
✓ Test failover regularly (chaos engineering)
✓ Document RTO/RPO requirements ชัดเจน
✓ Consider compliance requirements ตั้งแต่ต้น
```

---

## Workshop: ทดลองสร้าง Active-Passive Setup

```bash
#!/bin/bash
# setup-multi-region.sh
# ทดลองใช้ Docker สำหรับ simulating multi-region

# สร้าง Docker network สำหรับแต่ละ "region"
docker network create sg-region --subnet=10.1.0.0/24
docker network create eu-region --subnet=10.2.0.0/24

# Primary PostgreSQL (Singapore)
docker run -d \
  --name pg-singapore \
  --network sg-region \
  --ip 10.1.0.2 \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=myapp \
  postgres:16 \
  postgres -c wal_level=replica \
           -c max_wal_senders=10 \
           -c max_replication_slots=5

# รอให้ primary พร้อม
sleep 5

# สร้าง replication user
docker exec pg-singapore psql -U postgres -c "
  CREATE USER replication WITH REPLICATION PASSWORD 'repl_secret';
  SELECT pg_create_physical_replication_slot('eu_replica');
"

# Frankfurt Replica
docker run -d \
  --name pg-frankfurt \
  --network eu-region \
  --ip 10.2.0.2 \
  -e POSTGRES_PASSWORD=secret \
  postgres:16

# Connect Frankfurt to Singapore network (cross-region link)
docker network connect sg-region pg-frankfurt

# สร้าง base backup
docker exec pg-frankfurt bash -c "
  rm -rf /var/lib/postgresql/data/*
  pg_basebackup \
    --host=10.1.0.2 \
    --port=5432 \
    --username=replication \
    --pgdata=/var/lib/postgresql/data \
    --wal-method=stream \
    --slot=eu_replica \
    --write-recovery-conf
"

# ทดสอบ replication
docker exec pg-singapore psql -U postgres -d myapp -c "
  CREATE TABLE test (id SERIAL PRIMARY KEY, data TEXT, created_at TIMESTAMPTZ DEFAULT now());
  INSERT INTO test (data) VALUES ('Hello from Singapore!');
"

sleep 2

# ตรวจสอบว่าข้อมูลมาถึง Frankfurt หรือไม่
docker exec pg-frankfurt psql -U postgres -d myapp -c "
  SELECT * FROM test;
"
```

จบ Part 76 - Multi-Region Database Architecture ซึ่งครอบคลุม:
- เหตุผลและแรงจูงใจในการใช้ Multi-Region
- Active-Passive, Active-Active, Follow-the-Sun patterns
- Implementation รายละเอียดสำหรับ PostgreSQL
- DNS Failover ด้วย Route53 และ Cloudflare
- Global database services (Aurora Global, Spanner, CockroachDB)
- Multi-Region Redis และ S3
- Monitoring และ compliance

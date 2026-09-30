# Part 22: Connection Pooling ด้วย PgBouncer

## บทนำ: ปัญหาของ PostgreSQL Connections

PostgreSQL ใช้ Process Model ในการจัดการ Connection แต่ละ Connection ต้องการ Process แยกต่างหาก ทำให้มี Overhead สูง

```
Client 1 ──────▶ [pg Process 1]  ─ 5-10 MB RAM
Client 2 ──────▶ [pg Process 2]  ─ 5-10 MB RAM  
Client 3 ──────▶ [pg Process 3]  ─ 5-10 MB RAM
...
Client 1000 ──▶ [pg Process 1000] ─ 5-10 MB RAM
                                    ─────────────
                                    Total: ~10 GB RAM!
```

**ปัญหาที่เกิดขึ้น:**
- Memory Exhaustion
- CPU Overhead จาก Context Switching
- ช้าลงในการสร้าง Connection ใหม่ (TCP handshake + Authentication)
- PostgreSQL มีค่า max_connections จำกัด (ปกติ 100-200)

---

## 1. Connection Pooling คืออะไร

Connection Pooling คือการ "เก็บ Connection ที่เปิดอยู่แล้วไว้ใช้ซ้ำ" แทนที่จะเปิด/ปิดทุกครั้ง

```
ก่อนใช้ Pooling:
App ──(open)──▶ PostgreSQL ──(query)──▶ App ──(close)──▶ PostgreSQL
                     ↑ ใช้เวลา ~5ms ทุกครั้ง

หลังใช้ Pooling:
App ──────────▶ PgBouncer ──(reuse)──▶ PostgreSQL
                Pool (10 connections สำรองไว้)
                     ↑ ใช้เวลา ~0.1ms
```

### 1.1 ประโยชน์ของ Connection Pooling

| หัวข้อ | ไม่มี Pooling | มี Pooling |
|--------|--------------|------------|
| Connections | 1000 (ตาม App Instances) | 10-50 (จริงๆ ใน PostgreSQL) |
| Memory | ~10 GB | ~500 MB |
| Connect Time | ~5ms | ~0.1ms |
| Throughput | ต่ำ | สูง |

---

## 2. PgBouncer: Pool Modes

PgBouncer มี 3 Pool Modes ที่มีการทำงานต่างกัน:

### 2.1 Session Mode

```
Client เชื่อมต่อ ──▶ PgBouncer จัดสรร Connection ──▶ PostgreSQL
Client ยึด Connection นั้นตลอด Session
Client Disconnect ──▶ Connection คืน Pool
```

- **ปลอดภัยที่สุด**: รองรับทุก PostgreSQL Feature
- **Least Efficient**: Connection ถูกยึดแม้ขณะ Idle
- **ใช้กับ**: Application ที่ใช้ Session-level Features (Prepared Statements, SET LOCAL, Advisory Locks)

### 2.2 Transaction Mode ⭐ (แนะนำ)

```
Client ──▶ BEGIN ──▶ PgBouncer จัดสรร Connection
        (ใช้ Connection เฉพาะช่วง Transaction เท่านั้น)
Client ──▶ COMMIT ──▶ Connection คืน Pool ทันที
```

- **Efficient มาก**: Connection ถูก Hold เฉพาะช่วง Transaction
- **รองรับส่วนใหญ่**: ใช้ได้กับ App ส่วนใหญ่
- **ไม่รองรับ**: LISTEN/NOTIFY, SET (outside transaction), Prepared Statements (ต้องใช้ server-side prepared statements disabled)

### 2.3 Statement Mode

```
Client ──▶ Statement ──▶ PgBouncer จัดสรร Connection
        (Connection เปลี่ยนได้ทุก Statement)
```

- **Efficient ที่สุด**: แต่ละ Statement ใช้ Connection ที่มีว่าง
- **ไม่รองรับ**: Multi-statement Transactions
- **ใช้กับ**: Autocommit Applications เท่านั้น

---

## 3. PgBouncer Installation ด้วย Docker

### 3.1 Docker Image

```bash
# ใช้ Official Image
docker pull edoburu/pgbouncer

# หรือ bitnami
docker pull bitnami/pgbouncer
```

### 3.2 pgbouncer.ini: Full Configuration

```ini
;; pgbouncer.ini - PgBouncer Configuration File

;; =====================================================
;; DATABASES SECTION
;; =====================================================
[databases]

;; Format: database_alias = host=HOST port=PORT dbname=DBNAME
;; Application เชื่อมต่อ pgbouncer ด้วยชื่อ database_alias
;; PgBouncer จะ Forward ไปยัง host=HOST จริงๆ

;; Database หลัก
myapp = host=postgres port=5432 dbname=myapp pool_size=25

;; Read Replica
myapp_read = host=postgres-replica port=5432 dbname=myapp pool_size=50

;; ถ้าต้องการ Default (ใช้ชื่อเดียวกับที่ client ส่งมา)
;; * = host=postgres port=5432

;; Multiple Databases
;; db1 = host=pg1 port=5432 dbname=db1
;; db2 = host=pg2 port=5432 dbname=db2

;; =====================================================
;; PGBOUNCER SECTION
;; =====================================================
[pgbouncer]

;; ฟัง Connection จากทุก IP
listen_addr = *

;; Port ที่ PgBouncer ฟัง (ปกติใช้ 6432)
listen_port = 6432

;; Authentication file
auth_file = /etc/pgbouncer/userlist.txt

;; Authentication Type
;; md5, scram-sha-256, plain, trust, any, hba, cert
auth_type = md5

;; Unix Socket (ถ้าต้องการ)
;; unix_socket_dir = /var/run/postgresql

;; =====================================================
;; POOL CONFIGURATION
;; =====================================================

;; Pool Mode: session, transaction, statement
pool_mode = transaction

;; จำนวน Server Connections สูงสุดต่อ (database, user) pair
default_pool_size = 25

;; จำนวน Pool พิเศษเมื่อ Pool เต็ม (เผื่อไว้)
reserve_pool_size = 5

;; เวลาก่อน Connection จาก Reserve Pool จะถูกลบ
reserve_pool_timeout = 3

;; จำนวน Client Connections สูงสุด (รวมทุก Database)
max_client_conn = 1000

;; จำนวน Server Connections สูงสุดต่อ Database
;; max_db_connections = 50

;; จำนวน Server Connections สูงสุดต่อ User
;; max_user_connections = 50

;; =====================================================
;; TIMEOUT SETTINGS
;; =====================================================

;; เวลา Wait หาก Pool เต็ม (0 = รอตลอด)
query_wait_timeout = 120

;; ยกเลิก Query ที่นานเกิน N วินาที
query_timeout = 0

;; ยกเลิก Transaction ที่นานเกิน N วินาที
client_login_timeout = 60

;; ปิด Client ที่ Idle เกิน N วินาที
client_idle_timeout = 0

;; ปิด Server Connection ที่ Idle เกิน N วินาที
server_idle_timeout = 600

;; เวลา Connect ไปยัง PostgreSQL
server_connect_timeout = 15

;; Login ไม่สำเร็จหลาย N ครั้ง: Block
;; server_login_retry = 15

;; =====================================================
;; TLS SETTINGS (สำหรับ Secure Connection)
;; =====================================================

;; Client → PgBouncer TLS
;; client_tls_sslmode = require
;; client_tls_cert_file = /etc/pgbouncer/server.crt
;; client_tls_key_file = /etc/pgbouncer/server.key
;; client_tls_ca_file = /etc/pgbouncer/root.crt

;; PgBouncer → PostgreSQL TLS
;; server_tls_sslmode = require

;; =====================================================
;; LOGGING
;; =====================================================

;; Log Level: 0=none, 1=errors, 2=connections, 3=queries
log_connections = 0
log_disconnections = 0
log_pooler_errors = 1

;; Log Statistics ทุก N วินาที
stats_period = 60

;; =====================================================
;; ADMIN / STATS
;; =====================================================

;; Admin Users (สำหรับ SHOW commands)
admin_users = pgbouncer_admin

;; Stats Users (READ-ONLY)
stats_users = monitoring

;; =====================================================
;; CONNECTION TUNING
;; =====================================================

;; ตรวจสอบ Connection ก่อนใช้
server_check_delay = 30
server_check_query = SELECT 1

;; Reset Session State ก่อนคืน Pool
;; (สำคัญมากสำหรับ Session Mode)
server_reset_query = DISCARD ALL

;; สำหรับ Transaction Mode ใช้ SET แทน
;; server_reset_query = ;

;; =====================================================
;; MISC
;; =====================================================

;; ไม่ส่ง startup parameters ไปยัง PostgreSQL
ignore_startup_parameters = extra_float_digits

;; DNS Lookup ทุก N วินาที (สำหรับ Dynamic IP)
dns_max_ttl = 15
dns_nxdomain_ttl = 15
```

### 3.3 userlist.txt: Authentication

```
;; userlist.txt - PgBouncer User Authentication File
;; Format: "username" "password_hash"

;; MD5 password (MD5 of password + username)
;; Generate: echo -n "passwordusername" | md5sum
;; then prefix with "md5"

"app_user" "md5$(echo -n 'apppasswordapp_user' | md5sum | cut -d' ' -f1)"

;; หรือใช้ plain password (ไม่แนะนำใน Production)
;; "app_user" "apppassword"

;; Admin user
"pgbouncer_admin" "md5$(echo -n 'adminpasswordpgbouncer_admin' | md5sum | cut -d' ' -f1)"

;; Monitoring user
"monitoring" "md5$(echo -n 'monpassmonitoring' | md5sum | cut -d' ' -f1)"

;; สร้าง MD5 Hash ด้วย Python:
;; python3 -c "import hashlib; print('md5' + hashlib.md5(b'password' + b'username').hexdigest())"
```

**สร้าง MD5 Hash ง่ายๆ:**
```bash
#!/bin/bash
# generate-userlist.sh

generate_md5() {
    local username="$1"
    local password="$2"
    local hash=$(echo -n "${password}${username}" | md5sum | cut -d' ' -f1)
    echo "\"${username}\" \"md5${hash}\""
}

# สร้าง userlist.txt
cat > /etc/pgbouncer/userlist.txt << EOF
$(generate_md5 "app_user" "AppP@ssw0rd!")
$(generate_md5 "pgbouncer_admin" "AdminP@ssw0rd!")
$(generate_md5 "monitoring" "MonP@ssw0rd!")
EOF

echo "userlist.txt created successfully"
cat /etc/pgbouncer/userlist.txt
```

---

## 4. Pool Sizing Formula

### 4.1 สูตรการคำนวณ Pool Size

```
Optimal Pool Size = (CPU Cores × 2) + Disk Spindles

ตัวอย่าง:
- Server: 8 CPU Cores, SSD (1 spindle effectively)
- Optimal: (8 × 2) + 1 = 17 connections
- ปรับเป็น: 20 connections (เผื่อ headroom)
```

**เหตุผล:**
- PostgreSQL Query ใช้เวลาส่วนใหญ่รอ I/O
- ถ้า Connection มากเกินไป → CPU/Memory Overhead มากกว่า Benefit
- Database Connection ไม่ใช่ Thread ที่มากขึ้นเท่ากับเร็วขึ้น

### 4.2 ตัวอย่างการคำนวณ

```
สถานการณ์: E-commerce Platform
- PostgreSQL Server: 16 CPU cores, NVMe SSD
- Peak Traffic: 500 concurrent users
- Average Query Time: 50ms

คำนวณ:
Optimal DB Connections = (16 × 2) + 1 = 33
PgBouncer Pool Size = 33
Max Client Connections = 500

PgBouncer จัดการ:
- Client Connections: 500 users → Pool ของ 33 DB Connections
- Efficiency: 500/33 ≈ 15x multiplier
```

---

## 5. Monitoring: PgBouncer SHOW Commands

### 5.1 เชื่อมต่อ PgBouncer Admin

```bash
# เชื่อมต่อ PgBouncer Admin Console
psql -h localhost -p 6432 -U pgbouncer_admin pgbouncer

# หรือ
psql "postgresql://pgbouncer_admin:password@localhost:6432/pgbouncer"
```

### 5.2 SHOW POOLS

```sql
-- ดู Connection Pools ทั้งหมด
SHOW POOLS;

-- ผลลัพธ์:
--  database | user     | cl_active | cl_waiting | sv_active | sv_idle | sv_used | sv_tested | sv_login | maxwait | maxwait_us | pool_mode
-- ----------+----------+-----------+------------+-----------+---------+---------+-----------+----------+---------+------------+-----------
--  myapp    | app_user | 25        | 0          | 25        | 5       | 0       | 0         | 0        | 0       | 0          | transaction

-- คำอธิบาย:
-- cl_active:  Client connections ที่มี Server connection ผูกไว้
-- cl_waiting: Client connections ที่รอ Server connection
-- sv_active:  Server connections ที่ถูกใช้อยู่
-- sv_idle:    Server connections ที่ว่างใน Pool
-- sv_used:    Server connections ที่เพิ่งปล่อยออกมา ยังไม่ตรวจสอบ
-- maxwait:    เวลานานที่สุดที่ Client ต้องรอ (วินาที)
```

### 5.3 SHOW CLIENTS

```sql
-- ดู Client Connections ทั้งหมด
SHOW CLIENTS;

--  type | user     | database | state  | addr           | port  | local_addr | local_port | connect_time        | request_time
-- ------+----------+----------+--------+----------------+-------+------------+------------+---------------------+--------------------
--  C    | app_user | myapp    | active | 192.168.1.100  | 54321 | 0.0.0.0    | 6432       | 2024-01-15 10:30:00 | 2024-01-15 10:30:01

-- state: active (มี query), idle (ว่าง), used (กำลังส่ง), waiting (รอ)
```

### 5.4 SHOW SERVERS

```sql
-- ดู Server (PostgreSQL) Connections
SHOW SERVERS;

--  type | user     | database | state  | addr        | port | local_addr | local_port | connect_time        | request_time | close_needed | ptr    | link | remote_pid | tls
-- ------+----------+----------+--------+-------------+------+------------+------------+---------------------+--------------+--------------+--------+------+------------+-----
--  S    | app_user | myapp    | active | 192.168.1.10 | 5432 | 0.0.0.0    | 54000      | 2024-01-15 10:00:00 | 10:30:01     | 0            | 0x...  |      | 12345      |
```

### 5.5 SHOW STATS

```sql
-- Statistics per Database
SHOW STATS;

--  database | total_xact_count | total_query_count | total_received | total_sent | total_xact_time | total_query_time | total_wait_time | avg_xact_count | avg_query_count | avg_recv | avg_sent | avg_xact_time | avg_query_time | avg_wait_time
-- ----------+-----------------+------------------+---------------+-----------+-----------------+-----------------+----------------+---------------+----------------+----------+----------+---------------+---------------+--------------
--  myapp    | 1500000         | 5000000          | 125000000     | 45000000  | 3000000000      | 1000000000      | 50000          | 250            | 833            | 20833    | 7500     | 500           | 166            | 8

-- avg_xact_time: เวลาเฉลี่ยต่อ Transaction (microseconds)
-- avg_query_time: เวลาเฉลี่ยต่อ Query (microseconds)
-- avg_wait_time: เวลาเฉลี่ยในการรอ Pool (microseconds)
```

### 5.6 SHOW STATS_TOTALS และ SHOW STATS_AVERAGES

```sql
-- Total Counters
SHOW STATS_TOTALS;

-- Averages (per second)
SHOW STATS_AVERAGES;

-- ข้อมูล Config
SHOW CONFIG;

-- ดู PgBouncer Version
SHOW VERSION;

-- ดู Active FDs (File Descriptors)
SHOW FDS;

-- ดู Memory Usage
SHOW MEM;

-- ดู DNS Caches
SHOW DNS_HOSTS;
SHOW DNS_ZONES;
```

### 5.7 Admin Commands

```sql
-- Reload Config (ไม่ต้อง Restart)
RELOAD;

-- Pause Database (ไม่รับ Query ใหม่)
PAUSE myapp;

-- Resume Database
RESUME myapp;

-- Suspend PgBouncer (freeze all pools)
SUSPEND;

-- Resume ทั้งหมด
RESUME;

-- Shutdown
SHUTDOWN;

-- Kill Client Connection
-- (ดู ptr จาก SHOW CLIENTS)
KILL myapp;

-- Reconnect Server Connections
RECONNECT;
```

---

## 6. PgBouncer Authentication Methods

### 6.1 MD5 Authentication

```ini
;; pgbouncer.ini
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt
```

```
;; userlist.txt - MD5 Hash
;; Format: "username" "md5<hash_of_password+username>"
"app_user" "md5c4ca4238a0b923820dcc509a6f75849b"
```

**สร้าง MD5 Hash:**
```python
import hashlib

def pgbouncer_md5(username: str, password: str) -> str:
    """สร้าง PgBouncer MD5 hash"""
    combined = password + username
    md5_hash = hashlib.md5(combined.encode()).hexdigest()
    return f"md5{md5_hash}"

# ตัวอย่าง
print(pgbouncer_md5("app_user", "MyPassword123"))
# Output: md5xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

### 6.2 SCRAM-SHA-256 (แนะนำสำหรับ Production)

```ini
;; pgbouncer.ini (PgBouncer 1.17+)
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt
```

```sql
-- สร้าง SCRAM hash จาก PostgreSQL
-- ต้อง set password_encryption = scram-sha-256 ใน PostgreSQL
SET password_encryption = 'scram-sha-256';
ALTER USER app_user PASSWORD 'MyPassword123';

-- ดู SCRAM hash
SELECT usename, passwd FROM pg_shadow WHERE usename = 'app_user';
-- Output: SCRAM-SHA-256$4096:...
```

### 6.3 auth_query: ดึง Password จาก PostgreSQL โดยตรง

```ini
;; pgbouncer.ini
;; ไม่ต้องมี userlist.txt - PgBouncer ถาม PostgreSQL แทน
auth_type = md5
auth_user = pgbouncer_auth
auth_query = SELECT usename, passwd FROM pg_shadow WHERE usename=$1
```

```sql
-- สร้าง User สำหรับ auth_query
CREATE USER pgbouncer_auth WITH PASSWORD 'auth_password' LOGIN;
GRANT SELECT ON pg_shadow TO pgbouncer_auth;
```

---

## 7. Connection String Changes

### 7.1 ก่อนใช้ PgBouncer

```typescript
// ก่อน: เชื่อมต่อ PostgreSQL โดยตรง
const pool = new Pool({
  host: 'postgres-server',
  port: 5432,
  database: 'myapp',
  user: 'app_user',
  password: 'password',
  max: 20  // สร้าง 20 connections ต่อ App instance
});
```

### 7.2 หลังใช้ PgBouncer

```typescript
// หลัง: เชื่อมต่อผ่าน PgBouncer
const pool = new Pool({
  host: 'pgbouncer-server',  // ชี้ไป PgBouncer แทน
  port: 6432,                 // PgBouncer port
  database: 'myapp',          // ชื่อ Database ที่ตั้งใน pgbouncer.ini
  user: 'app_user',
  password: 'password',
  max: 5  // ลดลงได้เพราะ PgBouncer จัดการแทน

  // สำคัญ: ปิด Prepared Statements สำหรับ Transaction Mode
  // statement_timeout: ไม่ควรใช้ในระดับ Application ถ้าใช้ Transaction Mode
});

// หรือ Connection String
const connectionString = 
  'postgresql://app_user:password@pgbouncer-server:6432/myapp';
```

### 7.3 Prepared Statements กับ Transaction Mode

```typescript
// ปัญหา: Prepared Statements ไม่ทำงานกับ Transaction Mode
// ❌ จะได้ Error
const result = await pool.query({
  name: 'get-user',  // Named (Prepared) Statement
  text: 'SELECT * FROM users WHERE id = $1',
  values: [userId]
});

// ✅ ใช้ Unnamed Statement แทน
const result = await pool.query(
  'SELECT * FROM users WHERE id = $1',
  [userId]
);

// หรือปิด Prepared Statements ใน pg library
import { Pool } from 'pg';
const pool = new Pool({
  connectionString: 'postgresql://...',
  statement_timeout: undefined,
  // pg ใหม่ๆ รองรับ: 
  // keepAlive: false
});
```

---

## 8. Docker Compose: Full Stack

### 8.1 docker-compose.yml

```yaml
version: '3.8'

networks:
  app-network:
    driver: bridge

volumes:
  postgres-data:
  pgbouncer-config:

services:
  # ==========================================
  # POSTGRESQL
  # ==========================================
  postgres:
    image: postgres:15-alpine
    container_name: postgres
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres123
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./postgres/init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - app-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 10

  # ==========================================
  # PGBOUNCER
  # ==========================================
  pgbouncer:
    image: edoburu/pgbouncer:latest
    container_name: pgbouncer
    environment:
      # Configuration via Environment Variables
      DB_USER: app_user
      DB_PASSWORD: AppP@ssw0rd!
      DB_HOST: postgres
      DB_PORT: 5432
      DB_NAME: myapp
      POOL_MODE: transaction
      DEFAULT_POOL_SIZE: 25
      MAX_CLIENT_CONN: 1000
      AUTH_TYPE: md5
    volumes:
      - ./pgbouncer/pgbouncer.ini:/etc/pgbouncer/pgbouncer.ini:ro
      - ./pgbouncer/userlist.txt:/etc/pgbouncer/userlist.txt:ro
    ports:
      - "6432:6432"
    networks:
      - app-network
    depends_on:
      postgres:
        condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -h localhost -p 6432 -U app_user -d myapp"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ==========================================
  # APPLICATION (Node.js)
  # ==========================================
  app:
    build:
      context: ./app
      dockerfile: Dockerfile
    container_name: nodejs-app
    environment:
      # เชื่อมต่อผ่าน PgBouncer
      DATABASE_URL: postgresql://app_user:AppP@ssw0rd!@pgbouncer:6432/myapp
      NODE_ENV: production
      PORT: 3000
    ports:
      - "3000:3000"
    networks:
      - app-network
    depends_on:
      pgbouncer:
        condition: service_healthy

  # ==========================================
  # PGBOUNCER EXPORTER (Prometheus Metrics)
  # ==========================================
  pgbouncer-exporter:
    image: prometheuscommunity/pgbouncer-exporter:latest
    container_name: pgbouncer-exporter
    command:
      - '--pgBouncer.connectionString=postgresql://monitoring:MonP@ssw0rd!@pgbouncer:6432/pgbouncer'
    ports:
      - "9127:9127"
    networks:
      - app-network
    depends_on:
      - pgbouncer
```

### 8.2 postgres/init.sql

```sql
-- สร้าง Application User
CREATE USER app_user WITH PASSWORD 'AppP@ssw0rd!' LOGIN;

-- สร้าง Monitoring User
CREATE USER monitoring WITH PASSWORD 'MonP@ssw0rd!' LOGIN;

-- สร้าง Database
CREATE DATABASE myapp OWNER app_user;

-- ให้สิทธิ์
GRANT ALL PRIVILEGES ON DATABASE myapp TO app_user;

\c myapp app_user

-- สร้าง Tables
CREATE TABLE IF NOT EXISTS users (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(100) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id BIGINT REFERENCES users(id),
    data JSONB,
    expires_at TIMESTAMPTZ NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Insert ตัวอย่าง
INSERT INTO users (email, name) 
SELECT 
    'user' || i || '@example.com',
    'User ' || i
FROM generate_series(1, 1000) i;

CREATE INDEX ON users (email);
CREATE INDEX ON sessions (user_id);
CREATE INDEX ON sessions (expires_at);
```

### 8.3 pgbouncer/pgbouncer.ini

```ini
[databases]
myapp = host=postgres port=5432 dbname=myapp

[pgbouncer]
listen_addr = *
listen_port = 6432
auth_file = /etc/pgbouncer/userlist.txt
auth_type = md5
pool_mode = transaction
default_pool_size = 25
reserve_pool_size = 5
max_client_conn = 1000
query_wait_timeout = 30
server_idle_timeout = 600
log_connections = 0
log_disconnections = 0
log_pooler_errors = 1
stats_period = 60
admin_users = pgbouncer_admin
stats_users = monitoring
ignore_startup_parameters = extra_float_digits
server_check_query = SELECT 1
server_check_delay = 30
```

### 8.4 pgbouncer/userlist.txt

```bash
#!/bin/bash
# generate-userlist.sh - รันครั้งเดียวเพื่อสร้าง userlist.txt

python3 << 'EOF'
import hashlib

def md5_hash(password, username):
    combined = (password + username).encode()
    return "md5" + hashlib.md5(combined).hexdigest()

users = [
    ("app_user", "AppP@ssw0rd!"),
    ("pgbouncer_admin", "AdminP@ssw0rd!"),
    ("monitoring", "MonP@ssw0rd!"),
]

with open("pgbouncer/userlist.txt", "w") as f:
    for username, password in users:
        hash_val = md5_hash(password, username)
        f.write(f'"{username}" "{hash_val}"\n')
    
print("userlist.txt generated successfully")
EOF
```

---

## 9. Troubleshooting Common Issues

### 9.1 "no more connections allowed"

```
ERROR: pgbouncer: no more connections allowed
```

**สาเหตุ:** max_client_conn ถูกใช้เต็มแล้ว

**วิธีแก้:**
```ini
;; pgbouncer.ini - เพิ่ม max_client_conn
max_client_conn = 2000

;; ตรวจสอบก่อน
;; SHOW POOLS; -- ดู cl_waiting
;; ถ้า cl_waiting สูง แปลว่า Pool เต็ม ต้องเพิ่ม default_pool_size
;; แต่ระวัง max_db_connections ของ PostgreSQL
```

```bash
# Monitor real-time
watch -n 1 "psql -h localhost -p 6432 -U pgbouncer_admin pgbouncer -c 'SHOW POOLS;'"
```

### 9.2 "server closed connection unexpectedly"

```
ERROR: server closed connection unexpectedly
```

**สาเหตุ:** 
- PostgreSQL crashed หรือ Restart
- server_idle_timeout ทำให้ Connection ถูกปิด
- PostgreSQL max_connections ถูกใช้เต็ม

**วิธีแก้:**
```ini
;; pgbouncer.ini
;; ตรวจสอบ Connection ก่อนใช้
server_check_query = SELECT 1
server_check_delay = 30

;; ลด server_idle_timeout ให้น้อยกว่า PostgreSQL's tcp_keepalives_idle
server_idle_timeout = 300

;; เปิด TCP Keepalive
tcp_keepalive = 1
tcp_keepcnt = 9
tcp_keepidle = 60
tcp_keepintvl = 10
```

### 9.3 "prepared statement does not exist"

```
ERROR: prepared statement "pg_stmt_001" does not exist
```

**สาเหตุ:** ใช้ Named Prepared Statements กับ Transaction Mode

**วิธีแก้:**
```ini
;; วิธีที่ 1: เปลี่ยนเป็น Session Mode (ง่ายที่สุด)
pool_mode = session

;; วิธีที่ 2: ใช้ Statement Mode ใน pgbouncer.ini
;; (ไม่แนะนำสำหรับ Multi-statement Transaction)
pool_mode = statement

;; วิธีที่ 3: ปิด Prepared Statements ใน Application
```

```typescript
// Node.js: ปิด Prepared Statements
import { Pool } from 'pg';

// ใช้ query string แทน name
// ❌
await pool.query({ name: 'get-user', text: 'SELECT...', values: [id] });
// ✅  
await pool.query('SELECT * FROM users WHERE id = $1', [id]);
```

### 9.4 "deadlock detected" ใน PgBouncer

```
ERROR: deadlock detected
```

**สาเหตุ:** หลาย Clients รอ Connection เดียวกัน

**วิธีแก้:**
```ini
;; เพิ่ม Pool Size
default_pool_size = 50

;; เพิ่ม Reserve Pool
reserve_pool_size = 10

;; ลด Query Wait Timeout
query_wait_timeout = 10
```

---

## 10. PgBouncer + Patroni

เมื่อใช้ Patroni สำหรับ High Availability PgBouncer ต้องรู้ว่า Primary อยู่ที่ไหน

### 10.1 HAProxy + PgBouncer + Patroni

```
App ──▶ PgBouncer ──▶ HAProxy ──▶ Patroni Primary
                              ──▶ Patroni Replica (read)
```

### 10.2 pgbouncer.ini สำหรับ Patroni

```ini
[databases]
;; เชื่อมต่อผ่าน HAProxy ที่รู้ว่า Primary อยู่ที่ไหน
myapp = host=haproxy port=5000 dbname=myapp pool_size=25

;; Read-only queries ผ่าน HAProxy Read Port
myapp_ro = host=haproxy port=5001 dbname=myapp pool_size=50
```

### 10.3 Patroni pausing PgBouncer ระหว่าง Failover

```bash
#!/bin/bash
# patroni-callback.sh - เรียกใน Patroni on_role_change callback

PGBOUNCER_HOST="pgbouncer"
PGBOUNCER_PORT="6432"
PGBOUNCER_ADMIN="pgbouncer_admin"
PGBOUNCER_PASS="AdminP@ssw0rd!"

pause_pgbouncer() {
    psql "postgresql://${PGBOUNCER_ADMIN}:${PGBOUNCER_PASS}@${PGBOUNCER_HOST}:${PGBOUNCER_PORT}/pgbouncer" \
        -c "PAUSE myapp;"
    echo "PgBouncer paused"
}

resume_pgbouncer() {
    psql "postgresql://${PGBOUNCER_ADMIN}:${PGBOUNCER_PASS}@${PGBOUNCER_HOST}:${PGBOUNCER_PORT}/pgbouncer" \
        -c "RESUME myapp;"
    echo "PgBouncer resumed"
}

case "$1" in
    on_role_change)
        if [ "$2" = "primary" ]; then
            # ใหม่เพิ่งเป็น Primary - Resume PgBouncer
            sleep 2  # รอให้ Primary พร้อม
            resume_pgbouncer
        fi
        ;;
    on_start)
        pause_pgbouncer
        ;;
esac
```

---

## 11. Pgpool-II: Alternative to PgBouncer

### 11.1 Pgpool-II คืออะไร

Pgpool-II เป็น Connection Pooler และ Load Balancer ที่มี Feature มากกว่า PgBouncer

```
pgpool.conf:
```

```ini
# pgpool.conf
listen_addresses = '*'
port = 5432

# Load Balancing
load_balance_mode = on

# Backends
backend_hostname0 = 'primary'
backend_port0 = 5432
backend_weight0 = 1

backend_hostname1 = 'replica1'
backend_port1 = 5432
backend_weight1 = 2   # Replica รับ Traffic มากกว่า

# Connection Pooling
num_init_children = 32
max_pool = 4

# Health Check
health_check_period = 10
health_check_user = 'pgpool_health'
```

---

## 12. เปรียบเทียบ: PgBouncer vs Pgpool-II vs RDS Proxy

### 12.1 Comparison Table

| Feature | PgBouncer | Pgpool-II | RDS Proxy |
|---------|-----------|-----------|-----------|
| **Connection Pooling** | ✅ | ✅ | ✅ |
| **Load Balancing** | ❌ (Basic) | ✅ | ✅ |
| **Read/Write Splitting** | ❌ | ✅ | ✅ |
| **HA / Failover Detection** | ❌ | ✅ | ✅ |
| **Replication Management** | ❌ | ✅ | ❌ |
| **Parallel Query** | ❌ | ✅ | ❌ |
| **Performance** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Setup Complexity** | ง่าย | ซับซ้อน | ง่าย (AWS) |
| **Open Source** | ✅ | ✅ | ❌ (AWS Only) |
| **Overhead** | น้อยมาก | ปานกลาง | ปานกลาง |
| **Pool Modes** | session/txn/stmt | session | session/txn |

### 12.2 เมื่อไหร่ใช้อะไร

**ใช้ PgBouncer เมื่อ:**
```
- ต้องการ Connection Pooling เร็วๆ โดยไม่ต้องการ Feature พิเศษ
- Performance สำคัญมาก
- Self-hosted PostgreSQL
- ต้องการ Simplicity
```

**ใช้ Pgpool-II เมื่อ:**
```
- ต้องการ Load Balancing อัตโนมัติ
- ต้องการ Read/Write Splitting จาก Pooler (ไม่ใช่ Application)
- ต้องการ Replication Management
- ยอมรับ Complexity ที่มากกว่า
```

**ใช้ RDS Proxy เมื่อ:**
```
- ใช้ AWS RDS / Aurora
- ต้องการ IAM Authentication
- ต้องการ Serverless-friendly (Lambda)
- ยินดีจ่ายค่า AWS
```

---

## 13. Monitoring Dashboard Script

```typescript
// pgbouncer-monitor.ts
import { Client } from 'pg';

interface PoolStats {
  database: string;
  clActive: number;
  clWaiting: number;
  svActive: number;
  svIdle: number;
  maxWait: number;
  poolMode: string;
}

interface ServerStats {
  database: string;
  totalXactCount: number;
  totalQueryCount: number;
  avgXactTime: number;
  avgQueryTime: number;
  avgWaitTime: number;
}

class PgBouncerMonitor {
  private client: Client;

  constructor(connectionString: string) {
    this.client = new Client(connectionString);
  }

  async connect(): Promise<void> {
    await this.client.connect();
  }

  async disconnect(): Promise<void> {
    await this.client.end();
  }

  async getPools(): Promise<PoolStats[]> {
    const result = await this.client.query(`
      SHOW POOLS
    `);
    
    return result.rows.map(row => ({
      database: row.database,
      clActive: parseInt(row.cl_active),
      clWaiting: parseInt(row.cl_waiting),
      svActive: parseInt(row.sv_active),
      svIdle: parseInt(row.sv_idle),
      maxWait: parseInt(row.maxwait),
      poolMode: row.pool_mode,
    }));
  }

  async getStats(): Promise<ServerStats[]> {
    const result = await this.client.query(`
      SHOW STATS
    `);

    return result.rows.map(row => ({
      database: row.database,
      totalXactCount: parseInt(row.total_xact_count),
      totalQueryCount: parseInt(row.total_query_count),
      avgXactTime: parseFloat(row.avg_xact_time),
      avgQueryTime: parseFloat(row.avg_query_time),
      avgWaitTime: parseFloat(row.avg_wait_time),
    }));
  }

  async getAlerts(): Promise<string[]> {
    const alerts: string[] = [];
    const pools = await this.getPools();
    
    for (const pool of pools) {
      if (pool.clWaiting > 10) {
        alerts.push(
          `ALERT: ${pool.database} has ${pool.clWaiting} waiting clients`
        );
      }
      
      if (pool.maxWait > 5) {
        alerts.push(
          `ALERT: ${pool.database} max wait time is ${pool.maxWait}s`
        );
      }
      
      if (pool.svIdle === 0 && pool.clActive > 0) {
        alerts.push(
          `ALERT: ${pool.database} pool exhausted (no idle connections)`
        );
      }
    }

    return alerts;
  }

  async printDashboard(): Promise<void> {
    console.clear();
    console.log('=== PgBouncer Monitor ===');
    console.log(`Time: ${new Date().toISOString()}\n`);

    const pools = await this.getPools();
    const stats = await this.getStats();
    const alerts = await this.getAlerts();

    // Pools
    console.log('--- Connection Pools ---');
    console.table(pools.map(p => ({
      Database: p.database,
      'Client Active': p.clActive,
      'Client Waiting': p.clWaiting,
      'Server Active': p.svActive,
      'Server Idle': p.svIdle,
      'Max Wait (s)': p.maxWait,
      Mode: p.poolMode,
    })));

    // Stats
    console.log('\n--- Query Statistics ---');
    console.table(stats.map(s => ({
      Database: s.database,
      'Total Queries': s.totalQueryCount,
      'Avg Query (ms)': (s.avgQueryTime / 1000).toFixed(2),
      'Avg Wait (ms)': (s.avgWaitTime / 1000).toFixed(2),
    })));

    // Alerts
    if (alerts.length > 0) {
      console.log('\n--- ALERTS ---');
      alerts.forEach(a => console.log(`⚠️  ${a}`));
    } else {
      console.log('\n✅ All pools healthy');
    }
  }
}

// รัน Monitor
async function main() {
  const monitor = new PgBouncerMonitor(
    'postgresql://pgbouncer_admin:AdminP@ssw0rd!@localhost:6432/pgbouncer'
  );

  await monitor.connect();

  // Print Dashboard ทุก 5 วินาที
  setInterval(async () => {
    await monitor.printDashboard();
  }, 5000);

  await monitor.printDashboard();
}

main().catch(console.error);
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **ปัญหา PostgreSQL Connections** - Process Model ทำให้ Overhead สูง
2. **PgBouncer Pool Modes** - Session, Transaction, Statement
3. **pgbouncer.ini** - Configuration ทั้งหมดพร้อมคำอธิบาย
4. **userlist.txt** - Authentication ด้วย MD5 และ SCRAM
5. **Pool Sizing** - สูตร (CPU × 2) + Disk Spindles
6. **SHOW Commands** - Monitoring Pools, Clients, Servers, Stats
7. **Troubleshooting** - ปัญหาที่พบบ่อยและวิธีแก้
8. **Docker Compose** - Full Stack App + PgBouncer + PostgreSQL
9. **Pgpool-II** - Alternative ที่มี Feature มากกว่า
10. **Comparison** - PgBouncer vs Pgpool-II vs RDS Proxy

ในบทถัดไปเราจะเรียนรู้ Read/Write Splitting ใน Application เพื่อกระจาย Load ระหว่าง Primary และ Replica

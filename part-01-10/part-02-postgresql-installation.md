# บทที่ 2: ติดตั้ง PostgreSQL บน Linux/Mac/Docker

> **หลักสูตร:** PostgreSQL + Redis + S3/MinIO Database Cluster  
> **ระดับ:** เริ่มต้น  
> **เวลาเรียน:** 2-3 ชั่วโมง

---

## สารบัญ

1. [ติดตั้งบน Ubuntu/Debian](#1-ติดตั้งบน-ubuntudebian)
2. [ติดตั้งบน macOS](#2-ติดตั้งบน-macos)
3. [ติดตั้งด้วย Docker](#3-ติดตั้งดวย-docker)
4. [Docker Compose สำหรับ PostgreSQL](#4-docker-compose-สำหรับ-postgresql)
5. [PostgreSQL Configuration Files](#5-postgresql-configuration-files)
6. [psql CLI Commands](#6-psql-cli-commands)
7. [pgAdmin 4](#7-pgadmin-4)
8. [DBeaver](#8-dbeaver)
9. [Roles และ Permissions](#9-roles-และ-permissions)
10. [Database, User, Privileges](#10-database-user-privileges)
11. [Connection String](#11-connection-string)
12. [pg_dump และ pg_restore](#12-pg_dump-และ-pg_restore)
13. [แก้ปัญหา Common Errors](#13-แกปญหา-common-errors)

---

## 1. ติดตั้งบน Ubuntu/Debian

### 1.1 วิธีที่ 1: จาก Ubuntu Repository (ง่ายแต่อาจไม่ใช่ version ล่าสุด)

```bash
# อัปเดต package list
sudo apt update

# ติดตั้ง PostgreSQL (จะได้ version ที่ Ubuntu แนะนำ)
sudo apt install -y postgresql postgresql-contrib

# ตรวจสอบว่า service ทำงาน
sudo systemctl status postgresql

# เปิดใช้งาน auto-start เมื่อ boot
sudo systemctl enable postgresql
```

### 1.2 วิธีที่ 2: จาก Official PostgreSQL Repository (แนะนำ — ได้ version ล่าสุด)

```bash
# Step 1: ติดตั้ง dependencies
sudo apt install -y curl ca-certificates

# Step 2: สร้าง directory สำหรับ keyring
sudo install -d /usr/share/postgresql-common/pgdg

# Step 3: Download และ add PostgreSQL signing key
sudo curl -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc \
    --fail https://www.postgresql.org/media/keys/ACCC4CF8.asc

# Step 4: เพิ่ม repository
sudo sh -c 'echo "deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] \
    https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" \
    > /etc/apt/sources.list.d/pgdg.list'

# Step 5: อัปเดตและติดตั้ง
sudo apt update
sudo apt install -y postgresql-16

# ตรวจสอบ version
psql --version
# output: psql (PostgreSQL) 16.x

# ตรวจสอบ service status
sudo systemctl status postgresql@16-main
```

### 1.3 การตั้งค่าหลังติดตั้งบน Ubuntu

```bash
# PostgreSQL สร้าง user 'postgres' โดยอัตโนมัติ
# switch ไปเป็น postgres user
sudo -i -u postgres

# เข้า psql
psql

# ตั้ง password สำหรับ postgres user
\password postgres
# ป้อน password ใหม่ 2 ครั้ง

# ออกจาก psql
\q

# ออกจาก postgres user
exit

# ทดสอบ connection จาก user ปกติ
psql -U postgres -h localhost -W
# ป้อน password ที่ตั้งไว้
```

### 1.4 PostgreSQL Service Commands บน Linux

```bash
# เริ่ม service
sudo systemctl start postgresql

# หยุด service
sudo systemctl stop postgresql

# Restart service
sudo systemctl restart postgresql

# Reload config (โดยไม่ restart — สำหรับเปลี่ยน pg_hba.conf)
sudo systemctl reload postgresql

# ดู status
sudo systemctl status postgresql

# ดู logs
sudo journalctl -u postgresql -f

# หรือดู log โดยตรง
sudo tail -f /var/log/postgresql/postgresql-16-main.log
```

### 1.5 ตำแหน่งไฟล์สำคัญบน Ubuntu

```bash
# Config files
/etc/postgresql/16/main/postgresql.conf   # Main configuration
/etc/postgresql/16/main/pg_hba.conf       # Authentication config
/etc/postgresql/16/main/pg_ident.conf     # User mapping

# Data directory
/var/lib/postgresql/16/main/

# Log files
/var/log/postgresql/

# Binary files
/usr/lib/postgresql/16/bin/postgres       # Server binary
/usr/bin/psql                             # Client binary
```

---

## 2. ติดตั้งบน macOS

### 2.1 วิธีที่ 1: Homebrew (แนะนำ)

```bash
# ติดตั้ง Homebrew ก่อน (ถ้ายังไม่มี)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# ติดตั้ง PostgreSQL 16
brew install postgresql@16

# เพิ่ม PATH
echo 'export PATH="/opt/homebrew/opt/postgresql@16/bin:$PATH"' >> ~/.zshrc
# หรือสำหรับ Intel Mac:
# echo 'export PATH="/usr/local/opt/postgresql@16/bin:$PATH"' >> ~/.zshrc

# Reload shell config
source ~/.zshrc

# เริ่ม PostgreSQL service
brew services start postgresql@16

# ทดสอบ
psql --version
psql -U $(whoami) -d postgres
```

### 2.2 วิธีที่ 2: Postgres.app (GUI — ง่ายที่สุด)

```bash
# Download จาก https://postgresapp.com/
# แล้ว drag ไปใน Applications folder

# เพิ่ม command line tools
sudo mkdir -p /etc/paths.d
echo /Applications/Postgres.app/Contents/Versions/latest/bin | \
    sudo tee /etc/paths.d/postgresapp

# Reload terminal แล้วทดสอบ
psql --version
```

### 2.3 Homebrew Service Commands

```bash
# เริ่ม service
brew services start postgresql@16

# หยุด service
brew services stop postgresql@16

# Restart
brew services restart postgresql@16

# ดูทุก services
brew services list

# ดู log (macOS)
tail -f ~/Library/Logs/homebrew/postgresql@16/postgresql@16.log
# หรือ
tail -f /opt/homebrew/var/log/postgresql@16.log
```

### 2.4 ตำแหน่งไฟล์สำคัญบน macOS (Homebrew)

```bash
# Config files (Apple Silicon)
/opt/homebrew/var/postgresql@16/postgresql.conf
/opt/homebrew/var/postgresql@16/pg_hba.conf

# Config files (Intel)
/usr/local/var/postgresql@16/postgresql.conf
/usr/local/var/postgresql@16/pg_hba.conf

# Data directory
/opt/homebrew/var/postgresql@16/     # Apple Silicon
/usr/local/var/postgresql@16/        # Intel

# Log file
/opt/homebrew/var/log/postgresql@16.log
```

---

## 3. ติดตั้งด้วย Docker

### 3.1 รัน PostgreSQL Container เดี่ยว

```bash
# รัน PostgreSQL 16 container
docker run \
    --name my-postgres \
    -e POSTGRES_USER=admin \
    -e POSTGRES_PASSWORD=admin123 \
    -e POSTGRES_DB=mydb \
    -p 5432:5432 \
    -v postgres_data:/var/lib/postgresql/data \
    --restart unless-stopped \
    -d \
    postgres:16

# ตรวจสอบ container
docker ps

# ดู logs
docker logs my-postgres -f

# ทดสอบ connection
docker exec -it my-postgres psql -U admin -d mydb

# ดูตำแหน่งไฟล์ใน container
docker exec -it my-postgres ls /var/lib/postgresql/data
```

### 3.2 Environment Variables ของ PostgreSQL Docker Image

```bash
# POSTGRES_USER: Username ของ superuser (default: postgres)
# POSTGRES_PASSWORD: Password ของ superuser (จำเป็นต้องกำหนด)
# POSTGRES_DB: Database ที่จะสร้างตอน startup (default: ชื่อเดียวกับ POSTGRES_USER)
# POSTGRES_HOST_AUTH_METHOD: วิธี authentication (trust, md5, scram-sha-256)
# PGDATA: Data directory ใน container (default: /var/lib/postgresql/data)

# ตัวอย่าง: รันแบบ trust (ไม่ต้อง password — ใช้แค่ develop เท่านั้น!)
docker run \
    --name pg-dev \
    -e POSTGRES_HOST_AUTH_METHOD=trust \
    -p 5432:5432 \
    -d postgres:16

# ต่อโดยไม่ต้อง password
docker exec -it pg-dev psql -U postgres
```

### 3.3 Custom Init Scripts

```bash
# สร้าง directory สำหรับ init scripts
mkdir -p ./init-scripts

# สร้าง init script
cat > ./init-scripts/01-create-schema.sql << 'EOF'
-- สร้าง schema
CREATE SCHEMA IF NOT EXISTS app;

-- สร้าง tables เริ่มต้น
CREATE TABLE app.users (
    user_id     BIGSERIAL PRIMARY KEY,
    email       VARCHAR(255) UNIQUE NOT NULL,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- สร้าง index
CREATE INDEX idx_users_email ON app.users(email);

-- ข้อมูลทดสอบ
INSERT INTO app.users (email) VALUES 
    ('test@example.com'),
    ('admin@example.com');
EOF

# รัน container พร้อม init scripts
docker run \
    --name my-postgres \
    -e POSTGRES_USER=admin \
    -e POSTGRES_PASSWORD=admin123 \
    -e POSTGRES_DB=mydb \
    -p 5432:5432 \
    -v postgres_data:/var/lib/postgresql/data \
    -v $(pwd)/init-scripts:/docker-entrypoint-initdb.d \
    -d postgres:16

# init scripts จะรันอัตโนมัติเมื่อ container สร้างใหม่
# ลำดับการรันตาม alphabetical order ของชื่อไฟล์
```

---

## 4. Docker Compose สำหรับ PostgreSQL

### 4.1 Basic docker-compose.yml

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:16
    container_name: pg-main
    environment:
      POSTGRES_USER: ${POSTGRES_USER:-admin}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-admin123}
      POSTGRES_DB: ${POSTGRES_DB:-mydb}
      PGDATA: /var/lib/postgresql/data/pgdata
    ports:
      - "${POSTGRES_PORT:-5432}:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init-scripts:/docker-entrypoint-initdb.d:ro
      - ./postgresql.conf:/etc/postgresql/postgresql.conf:ro
    command: postgres -c config_file=/etc/postgresql/postgresql.conf
    networks:
      - app-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-admin} -d ${POSTGRES_DB:-mydb}"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

volumes:
  postgres_data:
    driver: local

networks:
  app-network:
    driver: bridge
```

### 4.2 .env file

```bash
# .env
POSTGRES_USER=admin
POSTGRES_PASSWORD=supersecretpassword
POSTGRES_DB=production_db
POSTGRES_PORT=5432
```

### 4.3 Custom postgresql.conf สำหรับ Development

```bash
# สร้าง postgresql.conf สำหรับ Docker
cat > ./postgresql.conf << 'EOF'
# Connection settings
listen_addresses = '*'
port = 5432
max_connections = 100

# Memory settings (ปรับตาม RAM ของเครื่อง)
shared_buffers = 256MB
effective_cache_size = 1GB
work_mem = 4MB
maintenance_work_mem = 64MB

# WAL settings
wal_level = replica
max_wal_senders = 3
wal_keep_size = 128MB

# Logging (สำหรับ development)
log_destination = 'stderr'
logging_collector = off
log_min_duration_statement = 1000  # log queries ที่ใช้เวลา > 1 วินาที
log_checkpoints = on
log_connections = on
log_disconnections = on
log_lock_waits = on
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '

# Performance
default_statistics_target = 100
random_page_cost = 1.1  # สำหรับ SSD

# TimeZone
timezone = 'Asia/Bangkok'
EOF
```

### 4.4 รัน Compose Stack

```bash
# รัน ใน background
docker compose up -d

# ดู logs แบบ real-time
docker compose logs -f postgres

# เข้า psql
docker compose exec postgres psql -U admin -d mydb

# รีสตาร์ท service เดียว
docker compose restart postgres

# อัปเดต config (reload config โดยไม่ restart)
docker compose exec postgres psql -U admin -c "SELECT pg_reload_conf();"

# หยุด service
docker compose stop postgres

# ลบ containers (volumes ยังอยู่)
docker compose down

# ลบทั้ง containers และ volumes
docker compose down -v
```

---

## 5. PostgreSQL Configuration Files

### 5.1 postgresql.conf — หัวใจสำคัญ

```bash
# ดู config ปัจจุบัน
docker exec -it pg-main psql -U admin -c "SHOW ALL;"

# ดู config ที่สำคัญ
docker exec -it pg-main psql -U admin -c "
SELECT name, setting, unit, context, short_desc
FROM pg_settings
WHERE name IN (
    'max_connections',
    'shared_buffers',
    'effective_cache_size',
    'work_mem',
    'maintenance_work_mem',
    'wal_level',
    'max_wal_senders',
    'listen_addresses',
    'port',
    'timezone',
    'log_min_duration_statement'
)
ORDER BY name;"
```

**คำอธิบายการตั้งค่าสำคัญ:**

```ini
# ====================================================
# CONNECTION SETTINGS
# ====================================================

# IP ที่ PostgreSQL จะ listen (ใช้ '*' สำหรับทุก IP)
listen_addresses = '*'         

# Port ที่ PostgreSQL ใช้ (default: 5432)
port = 5432                    

# จำนวน connections สูงสุด (ระวัง! แต่ละ connection ใช้ memory)
max_connections = 100          # Production: ใช้ PgBouncer แทน

# ====================================================
# MEMORY SETTINGS
# ====================================================

# ใช้ 25% ของ RAM สำหรับ shared_buffers
# เช่น 16GB RAM → shared_buffers = 4GB
shared_buffers = 256MB         

# บอก optimizer ว่า OS cache มีขนาดเท่าไหร่ (ใช้ 75% ของ RAM)
effective_cache_size = 1GB     

# Memory per sort/join operation
# เพิ่มถ้า queries ช้า แต่ max_connections * work_mem ต้องไม่เกิน RAM
work_mem = 4MB                 

# Memory สำหรับ VACUUM, CREATE INDEX, etc.
maintenance_work_mem = 64MB    

# ====================================================
# WAL (Write-Ahead Log) SETTINGS
# ====================================================

# สำหรับ streaming replication ต้องใช้ 'replica' หรือ 'logical'
wal_level = replica            

# จำนวน streaming replication connections สูงสุด
max_wal_senders = 3            

# เก็บ WAL ไว้อย่างน้อย 128MB (ป้องกัน replica ล้าหลังเกินไป)
wal_keep_size = 128MB          

# Checkpoint ทุก 5 นาที (ปรับตาม workload)
checkpoint_timeout = 5min      

# ====================================================
# LOGGING
# ====================================================

# Log queries ที่ใช้เวลานานกว่า X ms (0 = log ทุก query, -1 = ปิด)
log_min_duration_statement = 1000

# Log ทุก connection/disconnection (development เท่านั้น)
log_connections = off
log_disconnections = off

# Log lock waits (สำคัญสำหรับ debug)
log_lock_waits = on
deadlock_timeout = 1s

# Format ของ log line
log_line_prefix = '%t [%p]: user=%u,db=%d,app=%a '

# ====================================================
# PERFORMANCE
# ====================================================

# จำนวน CPU ที่ PostgreSQL ใช้สำหรับ parallel queries
max_worker_processes = 8
max_parallel_workers_per_gather = 4
max_parallel_workers = 8

# สำหรับ SSD ใช้ 1.1, สำหรับ HDD ใช้ 4.0
random_page_cost = 1.1
effective_io_concurrency = 200  # สำหรับ SSD
```

### 5.2 pg_hba.conf — Authentication Configuration

```bash
# ดู pg_hba.conf
docker exec -it pg-main cat /var/lib/postgresql/data/pgdata/pg_hba.conf
```

```conf
# pg_hba.conf: Host-Based Authentication
# Format: TYPE  DATABASE  USER  ADDRESS   METHOD  [OPTIONS]

# ========================
# TYPE:
# local   = Unix socket (same machine)
# host    = TCP/IP
# hostssl = TCP/IP with SSL
# hostnossl = TCP/IP without SSL

# METHOD:
# trust       = ไม่ต้อง password (อันตราย! ใช้แค่ dev)
# reject      = ปฏิเสธเสมอ
# md5         = MD5 password (เก่า แต่ยังใช้ได้)
# scram-sha-256 = SCRAM password (ปลอดภัยกว่า md5 — แนะนำ)
# peer        = ใช้ OS username (local เท่านั้น)
# ident       = ใช้ identd server
# ========================

# TYPE  DATABASE  USER      ADDRESS           METHOD
# Local connections
local   all       postgres                    peer
local   all       all                         md5

# IPv4 local connections
host    all       all       127.0.0.1/32      scram-sha-256

# IPv6 local connections
host    all       all       ::1/128           scram-sha-256

# Docker network (172.16.0.0/12 ครอบคลุม Docker subnets ส่วนใหญ่)
host    all       all       172.16.0.0/12     scram-sha-256

# Production: จำกัดเฉพาะ IP ที่ต้องการ
# host    mydb    myapp     10.0.1.5/32       scram-sha-256

# Development: อนุญาตทุก connection (อย่าใช้ใน production!)
# host    all     all       0.0.0.0/0         md5
```

```bash
# เปลี่ยน pg_hba.conf และ reload
docker exec -it pg-main psql -U admin -c "SELECT pg_reload_conf();"

# ตรวจสอบว่า reload สำเร็จ (ดู log)
docker logs pg-main | tail -20
```

---

## 6. psql CLI Commands

### 6.1 การเชื่อมต่อ

```bash
# รูปแบบพื้นฐาน
psql -U username -d database -h host -p port

# ตัวอย่าง
psql -U admin -d mydb -h localhost -p 5432

# ใช้ connection string
psql "postgresql://admin:admin123@localhost:5432/mydb"

# ใช้ environment variables
export PGUSER=admin
export PGPASSWORD=admin123
export PGHOST=localhost
export PGPORT=5432
export PGDATABASE=mydb
psql  # ใช้ env vars อัตโนมัติ

# เข้า psql ใน Docker
docker exec -it pg-main psql -U admin -d mydb

# Run single command แล้วออก
psql -U admin -d mydb -c "SELECT version();"

# Run จากไฟล์ SQL
psql -U admin -d mydb -f ./my-script.sql
```

### 6.2 Meta-Commands (Backslash Commands)

```sql
-- ====================================================
-- DATABASE MANAGEMENT
-- ====================================================

\l              -- list ทุก databases
\l+             -- list ทุก databases พร้อมขนาด

-- สร้าง database
CREATE DATABASE testdb;

-- เชื่อมต่อ database อื่น
\c testdb
\connect testdb admin localhost 5432

-- ====================================================
-- TABLE MANAGEMENT  
-- ====================================================

\dt             -- list ทุก tables ใน current schema
\dt *.*         -- list ทุก tables ทุก schema
\dt public.*    -- list ทุก tables ใน public schema
\dt app.*       -- list ทุก tables ใน app schema

\d tablename    -- ดู structure ของ table
\d+ tablename   -- ดู structure + additional info (ขนาด, comment)

\di             -- list ทุก indexes
\di tablename   -- indexes ของ table นั้น

\ds             -- list sequences
\dv             -- list views
\dm             -- list materialized views
\df             -- list functions
\dp             -- list privileges
\dT             -- list data types (custom)

-- ====================================================
-- SCHEMA MANAGEMENT
-- ====================================================

\dn             -- list schemas
\dn+            -- list schemas พร้อม privileges

-- เปลี่ยน search_path
SET search_path TO app, public;
SHOW search_path;

-- ====================================================
-- USER MANAGEMENT
-- ====================================================

\du             -- list roles/users
\du+            -- พร้อม member information

-- ====================================================
-- OUTPUT CONTROL
-- ====================================================

\x              -- toggle expanded display (vertical output)
\x auto         -- auto switch ตามความกว้างของ terminal

\timing on      -- show query execution time
\timing off

\pset pager off     -- ปิด pager (ให้ output ไหลผ่าน)
\pset pager on      -- เปิด pager (less)

\o filename.txt     -- redirect output ไปไฟล์
\o                  -- กลับมา stdout

\a              -- toggle aligned/unaligned output
\t              -- toggle tuples only (ซ่อน header)

-- ====================================================
-- QUERY EXECUTION
-- ====================================================

\e              -- เปิด editor (ตาม $EDITOR) เพื่อแก้ query
\ef funcname    -- edit function

-- History
\s              -- ดู command history
\s filename     -- save history ไปไฟล์

-- Run file
\i filename.sql     -- execute SQL file

-- ====================================================
-- TRANSACTION
-- ====================================================

\set AUTOCOMMIT off     -- ปิด autocommit
\set AUTOCOMMIT on      -- เปิด autocommit (default)

-- ====================================================
-- INFORMATION
-- ====================================================

\?              -- help สำหรับ meta-commands
\h              -- help สำหรับ SQL commands
\h SELECT       -- help สำหรับ SELECT command

\conninfo       -- ดู connection info ปัจจุบัน

-- ====================================================
-- EXIT
-- ====================================================

\q              -- ออกจาก psql
```

### 6.3 Tips & Tricks

```sql
-- ดู table size
SELECT 
    relname AS table_name,
    pg_size_pretty(pg_total_relation_size(relid)) AS total_size,
    pg_size_pretty(pg_relation_size(relid)) AS data_size,
    pg_size_pretty(pg_total_relation_size(relid) - pg_relation_size(relid)) AS index_size
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC;

-- ดู database size
SELECT datname, pg_size_pretty(pg_database_size(datname)) AS size
FROM pg_database
ORDER BY pg_database_size(datname) DESC;

-- ดู active connections
SELECT pid, usename, application_name, client_addr, state, query
FROM pg_stat_activity
WHERE datname = 'mydb'
ORDER BY pid;

-- ยกเลิก query ที่ค้างอยู่
SELECT pg_cancel_backend(pid)
FROM pg_stat_activity
WHERE state = 'active' AND query_start < NOW() - INTERVAL '5 minutes';

-- ดู long running queries
SELECT pid, now() - query_start AS duration, query, state
FROM pg_stat_activity
WHERE state != 'idle'
  AND query_start < NOW() - INTERVAL '1 minute'
ORDER BY duration DESC;
```

---

## 7. pgAdmin 4

### 7.1 ติดตั้ง pgAdmin 4

#### วิธีที่ 1: Docker (แนะนำสำหรับหลักสูตรนี้)

```yaml
# เพิ่มใน docker-compose.yml
pgadmin:
  image: dpage/pgadmin4:latest
  container_name: pgadmin4
  environment:
    PGADMIN_DEFAULT_EMAIL: admin@admin.com
    PGADMIN_DEFAULT_PASSWORD: admin123
    PGADMIN_CONFIG_SERVER_MODE: 'False'    # Single user mode
    PGADMIN_CONFIG_MASTER_PASSWORD_REQUIRED: 'False'
  ports:
    - "8080:80"
  volumes:
    - pgadmin_data:/var/lib/pgadmin
    - ./pgadmin-servers.json:/pgadmin4/servers.json:ro
  networks:
    - app-network
  depends_on:
    postgres:
      condition: service_healthy
```

#### pgadmin-servers.json (auto-configure connection):

```json
{
    "Servers": {
        "1": {
            "Name": "Local PostgreSQL",
            "Group": "Development",
            "Host": "postgres",
            "Port": 5432,
            "MaintenanceDB": "postgres",
            "Username": "admin",
            "SSLMode": "prefer",
            "PassFile": "/pgpass",
            "Comment": "Local development PostgreSQL server"
        }
    }
}
```

#### วิธีที่ 2: ติดตั้งบน Ubuntu

```bash
# Install pgAdmin signing key
curl -fsS https://www.pgadmin.org/static/packages_pgadmin_org.pub | \
    sudo gpg --dearmor -o /usr/share/keyrings/packages-pgadmin-org.gpg

# Add repository
sudo sh -c 'echo "deb [signed-by=/usr/share/keyrings/packages-pgadmin-org.gpg] \
    https://ftp.postgresql.org/pub/pgadmin/pgadmin4/apt/$(lsb_release -cs) pgadmin4 main" \
    > /etc/apt/sources.list.d/pgadmin4.list'

# Install
sudo apt update
sudo apt install -y pgadmin4-web

# Configure web server
sudo /usr/pgadmin4/bin/setup-web.sh

# เปิด browser: http://localhost/pgadmin4
```

### 7.2 การ Connect ไปยัง PostgreSQL ใน pgAdmin

```
1. เปิด pgAdmin: http://localhost:8080
2. Login: admin@admin.com / admin123
3. คลิกขวาที่ "Servers" → "Register" → "Server"
4. กรอกข้อมูล:
   
   Tab "General":
   - Name: My PostgreSQL
   
   Tab "Connection":
   - Host name/address: postgres (หรือ localhost)
   - Port: 5432
   - Maintenance database: postgres
   - Username: admin
   - Password: admin123
   - Save password: ✅

5. คลิก "Save"
```

### 7.3 pgAdmin Features ที่ใช้บ่อย

```
Query Tool:
- Alt+Shift+Q: เปิด Query Tool
- F5: Run query
- Ctrl+Enter: Run selected
- Shift+F5: Run explain analyze

Object Browser:
- ดู tables, views, functions ทั้งหมด
- คลิกขวา table → View/Edit Data
- คลิกขวา table → Properties → ดู/แก้ columns

Dashboard:
- ดู active connections
- ดู transaction rate
- ดู cache hit ratio

Backup/Restore:
- คลิกขวา database → Backup
- คลิกขวา database → Restore
```

---

## 8. DBeaver

### 8.1 ติดตั้ง DBeaver

```bash
# Ubuntu/Debian
wget https://dbeaver.io/files/dbeaver-ce_latest_amd64.deb
sudo dpkg -i dbeaver-ce_latest_amd64.deb
sudo apt install -f  # แก้ dependencies

# macOS
brew install --cask dbeaver-community

# หรือ Download จาก https://dbeaver.io/download/
```

### 8.2 Connection Setup ใน DBeaver

```
1. คลิก "New Database Connection" (หรือ Ctrl+N)
2. เลือก "PostgreSQL"
3. กรอกข้อมูล:
   - Host: localhost
   - Port: 5432
   - Database: mydb
   - Username: admin
   - Password: admin123
4. คลิก "Test Connection" → ต้องได้ "Connected"
5. คลิก "Finish"
```

### 8.3 DBeaver Features ที่มีประโยชน์

```sql
-- DBeaver มี features พิเศษที่ pgAdmin ไม่มี:

-- 1. ER Diagram: คลิกขวา database → View Diagram
--    แสดง relationships ระหว่าง tables อัตโนมัติ

-- 2. Data Export: คลิกขวา table → Export Data
--    Export เป็น CSV, Excel, JSON, SQL, XML

-- 3. Data Compare: เปรียบเทียบข้อมูลระหว่าง 2 databases

-- 4. SQL Editor: Advanced autocomplete, syntax highlighting

-- 5. SSH Tunnel: เชื่อมต่อผ่าน SSH โดยตรง
```

---

## 9. Roles และ Permissions

### 9.1 PostgreSQL Role System

```sql
-- ใน PostgreSQL ทุกอย่างเป็น "role" (user = role with LOGIN privilege)

-- ดู roles ทั้งหมด
\du

-- หรือ query โดยตรง
SELECT 
    rolname,
    rolsuper,
    rolinherit,
    rolcreaterole,
    rolcreatedb,
    rolcanlogin,
    rolreplication,
    rolconnlimit,
    rolvaliduntil
FROM pg_roles
ORDER BY rolname;
```

### 9.2 สร้าง Roles และ Users

```sql
-- เชื่อมต่อในฐานะ superuser ก่อน
-- psql -U postgres

-- สร้าง role แบบธรรมดา (ไม่สามารถ login ได้)
CREATE ROLE readonly_role;
CREATE ROLE app_role;
CREATE ROLE admin_role;

-- สร้าง user (role with LOGIN)
CREATE USER app_user WITH 
    PASSWORD 'app_password_123'
    CONNECTION LIMIT 20;

CREATE USER readonly_user WITH 
    PASSWORD 'readonly_pass_456'
    CONNECTION LIMIT 10;

CREATE USER report_user WITH 
    PASSWORD 'report_pass_789'
    VALID UNTIL '2025-12-31';   -- account หมดอายุ

-- สร้าง superuser (ระวัง!)
CREATE USER superadmin WITH 
    SUPERUSER 
    CREATEDB 
    CREATEROLE 
    PASSWORD 'super_secure_password';

-- ดู users ที่สร้าง
\du
```

### 9.3 Grant Privileges

```sql
-- Grant ไปยัง database
GRANT CONNECT ON DATABASE mydb TO app_user;
GRANT CONNECT ON DATABASE mydb TO readonly_user;

-- Grant ไปยัง schema
GRANT USAGE ON SCHEMA app TO app_user;
GRANT USAGE ON SCHEMA app TO readonly_user;

-- Grant ไปยัง tables (individual)
GRANT SELECT, INSERT, UPDATE, DELETE ON app.users TO app_user;
GRANT SELECT ON app.users TO readonly_user;

-- Grant ไปยังทุก tables ใน schema
GRANT SELECT ON ALL TABLES IN SCHEMA app TO readonly_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA app TO app_user;

-- Grant สำหรับ tables ที่สร้างใหม่ในอนาคต
ALTER DEFAULT PRIVILEGES IN SCHEMA app 
    GRANT SELECT ON TABLES TO readonly_user;

ALTER DEFAULT PRIVILEGES IN SCHEMA app 
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_user;

-- Grant sequences (สำหรับ SERIAL/BIGSERIAL)
GRANT USAGE ON ALL SEQUENCES IN SCHEMA app TO app_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA app
    GRANT USAGE ON SEQUENCES TO app_user;

-- Grant functions
GRANT EXECUTE ON ALL FUNCTIONS IN SCHEMA app TO app_user;

-- Revoke privileges
REVOKE INSERT, UPDATE, DELETE ON app.sensitive_table FROM app_user;

-- ดู privileges ของ table
\dp app.users
-- หรือ
SELECT grantee, privilege_type, table_name 
FROM information_schema.role_table_grants 
WHERE table_schema = 'app'
ORDER BY table_name, grantee;
```

### 9.4 Row Level Security (RLS)

```sql
-- เปิด RLS สำหรับ table
ALTER TABLE app.users ENABLE ROW LEVEL SECURITY;

-- สร้าง policy: user เห็นแค่ข้อมูลของตัวเอง
CREATE POLICY user_isolation_policy ON app.users
    USING (user_id = current_setting('app.current_user_id')::BIGINT);

-- ทดสอบ
SET app.current_user_id = '1';
SELECT * FROM app.users;  -- จะเห็นแค่ user_id = 1

-- Bypass RLS สำหรับ admin
ALTER TABLE app.users FORCE ROW LEVEL SECURITY;
CREATE POLICY admin_bypass ON app.users
    TO admin_role
    USING (TRUE);  -- admin เห็นทุก rows
```

---

## 10. Database, User, Privileges

### 10.1 สร้างและจัดการ Databases

```sql
-- สร้าง database
CREATE DATABASE shop_db
    WITH
    OWNER = admin
    ENCODING = 'UTF8'
    LOCALE = 'en_US.UTF-8'
    TEMPLATE = template0;

-- สร้าง database สำหรับแต่ละ environment
CREATE DATABASE shop_development;
CREATE DATABASE shop_test;
CREATE DATABASE shop_production;

-- ดู databases
\l
SELECT datname, pg_size_pretty(pg_database_size(datname)) AS size
FROM pg_database
ORDER BY pg_database_size(datname) DESC;

-- เปลี่ยนชื่อ database
ALTER DATABASE shop_db RENAME TO ecommerce_db;

-- ลบ database
DROP DATABASE IF EXISTS old_database;

-- Clone database (template)
CREATE DATABASE new_db TEMPLATE existing_db;
```

### 10.2 Schemas

```sql
-- สร้าง schemas สำหรับแยก concerns
CREATE SCHEMA app;      -- Application tables
CREATE SCHEMA auth;     -- Authentication tables
CREATE SCHEMA reports;  -- Reporting views
CREATE SCHEMA staging;  -- Staging/ETL tables

-- กำหนด search_path (namespace lookup order)
-- สร้างสำหรับ user
ALTER USER app_user SET search_path TO app, public;

-- ดู search_path ปัจจุบัน
SHOW search_path;

-- Set ชั่วคราว (session only)
SET search_path TO app, auth, public;
```

### 10.3 ตัวอย่าง Full Setup Script

```sql
-- setup-database.sql
-- รัน: psql -U postgres -f setup-database.sql

-- 1. สร้าง database
CREATE DATABASE ecommerce_db
    WITH OWNER = postgres
    ENCODING = 'UTF8'
    LC_COLLATE = 'en_US.UTF-8'
    LC_CTYPE = 'en_US.UTF-8'
    TEMPLATE = template0;

\connect ecommerce_db

-- 2. สร้าง schemas
CREATE SCHEMA app;
CREATE SCHEMA reports;
CREATE SCHEMA staging;

-- 3. สร้าง roles
CREATE ROLE app_readonly;
CREATE ROLE app_readwrite;
CREATE ROLE app_admin;

-- 4. สร้าง users
CREATE USER api_user WITH PASSWORD 'api_secure_pass_2024' IN ROLE app_readwrite;
CREATE USER report_user WITH PASSWORD 'report_secure_pass_2024' IN ROLE app_readonly;
CREATE USER migration_user WITH PASSWORD 'migration_secure_pass_2024' IN ROLE app_admin;

-- 5. Grants
GRANT CONNECT ON DATABASE ecommerce_db TO app_readonly, app_readwrite, app_admin;
GRANT USAGE ON SCHEMA app, reports TO app_readonly, app_readwrite;
GRANT ALL ON SCHEMA app, reports, staging TO app_admin;

-- readonly permissions
GRANT SELECT ON ALL TABLES IN SCHEMA app TO app_readonly;
ALTER DEFAULT PRIVILEGES IN SCHEMA app GRANT SELECT ON TABLES TO app_readonly;

-- readwrite permissions
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA app TO app_readwrite;
GRANT USAGE ON ALL SEQUENCES IN SCHEMA app TO app_readwrite;
ALTER DEFAULT PRIVILEGES IN SCHEMA app GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_readwrite;
ALTER DEFAULT PRIVILEGES IN SCHEMA app GRANT USAGE ON SEQUENCES TO app_readwrite;

-- admin permissions
GRANT ALL ON ALL TABLES IN SCHEMA app, staging TO app_admin;
GRANT ALL ON ALL SEQUENCES IN SCHEMA app, staging TO app_admin;
ALTER DEFAULT PRIVILEGES IN SCHEMA app GRANT ALL ON TABLES TO app_admin;
ALTER DEFAULT PRIVILEGES IN SCHEMA app GRANT ALL ON SEQUENCES TO app_admin;

-- 6. ทดสอบ
SELECT current_database(), current_user, version();
```

---

## 11. Connection String

### 11.1 Format ต่างๆ

```bash
# Standard URI format
postgresql://username:password@host:port/database

# ตัวอย่าง
postgresql://admin:admin123@localhost:5432/mydb

# พร้อม SSL
postgresql://admin:admin123@localhost:5432/mydb?sslmode=require

# พร้อม parameters อื่นๆ
postgresql://admin:admin123@localhost:5432/mydb?sslmode=require&connect_timeout=10&application_name=myapp

# Multiple hosts (สำหรับ read replicas)
postgresql://admin:admin123@primary:5432,replica1:5432,replica2:5432/mydb?target_session_attrs=read-write

# ใช้ Unix socket
postgresql:///mydb?host=/var/run/postgresql&user=admin

# Key=value format (สำหรับ psql)
"host=localhost port=5432 dbname=mydb user=admin password=admin123"
```

### 11.2 Connection Parameters สำคัญ

```bash
# sslmode options:
# disable   - ไม่ใช้ SSL
# allow     - ใช้ SSL ถ้า server support
# prefer    - ใช้ SSL ถ้าเป็นไปได้ (default)
# require   - ต้องใช้ SSL เสมอ
# verify-ca - ต้องใช้ SSL + verify CA cert
# verify-full - ต้องใช้ SSL + verify CA + hostname

# connection timeout (seconds)
connect_timeout=10

# application name (จะแสดงใน pg_stat_activity)
application_name=myapp

# target_session_attrs:
# any            - ต่อ server ไหนก็ได้
# read-write     - ต้องต่อ primary เท่านั้น
# read-only      - ต้องต่อ replica เท่านั้น
# primary        - ต้องต่อ primary
# standby        - ต้องต่อ standby
# prefer-standby - ชอบ standby แต่ใช้ primary ได้

# ตัวอย่างการใช้กับ Python psycopg2
import psycopg2
conn = psycopg2.connect(
    "postgresql://admin:admin123@localhost:5432/mydb"
    "?application_name=my_python_app&connect_timeout=10"
)

# ตัวอย่างการใช้กับ Node.js pg
const { Pool } = require('pg')
const pool = new Pool({
    connectionString: 'postgresql://admin:admin123@localhost:5432/mydb',
    ssl: process.env.NODE_ENV === 'production' ? { rejectUnauthorized: false } : false,
    max: 20,                // max connections in pool
    idleTimeoutMillis: 30000,
    connectionTimeoutMillis: 2000,
})
```

---

## 12. pg_dump และ pg_restore

### 12.1 pg_dump — Backup

```bash
# Backup database ทั้งหมด (SQL format)
pg_dump -U admin -d mydb > mydb_backup.sql

# Backup พร้อม connection info
pg_dump "postgresql://admin:admin123@localhost:5432/mydb" > backup.sql

# Backup เฉพาะ schema (ไม่รวมข้อมูล)
pg_dump -U admin -d mydb --schema-only > schema_only.sql

# Backup เฉพาะข้อมูล (ไม่รวม schema)
pg_dump -U admin -d mydb --data-only > data_only.sql

# Backup เฉพาะบาง tables
pg_dump -U admin -d mydb -t public.users -t public.orders > partial_backup.sql

# Backup เฉพาะ schema ที่ระบุ
pg_dump -U admin -d mydb --schema=app > app_schema_backup.sql

# Backup แบบ custom format (compress ได้, restore แบบ selective ได้)
pg_dump -U admin -d mydb -Fc -f mydb_backup.dump

# Backup แบบ directory format (parallel restore ได้)
pg_dump -U admin -d mydb -Fd -j 4 -f mydb_backup_dir/

# Backup พร้อม compress
pg_dump -U admin -d mydb | gzip > mydb_backup_$(date +%Y%m%d).sql.gz

# Backup ด้วย Docker
docker exec pg-main pg_dump -U admin -d mydb > backup.sql

# Backup ทุก databases
pg_dumpall -U postgres > all_databases.sql
```

### 12.2 pg_restore — Restore

```bash
# Restore จาก SQL file
psql -U admin -d mydb < mydb_backup.sql

# สร้าง database ใหม่ก่อน restore
createdb -U admin new_mydb
psql -U admin -d new_mydb < mydb_backup.sql

# Restore จาก custom format
pg_restore -U admin -d mydb mydb_backup.dump

# Restore พร้อม options
pg_restore \
    -U admin \
    -d mydb \
    --clean \          # DROP objects ก่อน restore
    --if-exists \      # ใช้ IF EXISTS ใน DROP
    --verbose \        # แสดง progress
    mydb_backup.dump

# Restore แบบ parallel (ใช้กับ directory format)
pg_restore -U admin -d mydb -j 4 mydb_backup_dir/

# Restore เฉพาะบาง tables
pg_restore -U admin -d mydb -t users -t orders mydb_backup.dump

# Restore เฉพาะ schema
pg_restore -U admin -d mydb --schema=app mydb_backup.dump

# Restore จาก gzip
gunzip -c mydb_backup_20240101.sql.gz | psql -U admin -d mydb

# Restore ด้วย Docker
docker exec -i pg-main psql -U admin -d mydb < backup.sql
```

### 12.3 Automated Backup Script

```bash
#!/bin/bash
# backup-postgres.sh

set -e

# Configuration
PG_HOST="${PG_HOST:-localhost}"
PG_PORT="${PG_PORT:-5432}"
PG_USER="${PG_USER:-admin}"
PG_PASSWORD="${PG_PASSWORD:-admin123}"
PG_DATABASE="${PG_DATABASE:-mydb}"
BACKUP_DIR="${BACKUP_DIR:-/backups/postgresql}"
RETENTION_DAYS="${RETENTION_DAYS:-7}"

# Create backup directory
mkdir -p "$BACKUP_DIR"

# Generate filename with timestamp
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/${PG_DATABASE}_${TIMESTAMP}.dump"

# Run backup
echo "Starting backup of ${PG_DATABASE} at $(date)"

PGPASSWORD="$PG_PASSWORD" pg_dump \
    -h "$PG_HOST" \
    -p "$PG_PORT" \
    -U "$PG_USER" \
    -d "$PG_DATABASE" \
    -Fc \
    -f "$BACKUP_FILE"

# Verify backup
if [ $? -eq 0 ]; then
    SIZE=$(du -sh "$BACKUP_FILE" | cut -f1)
    echo "Backup successful: $BACKUP_FILE (Size: $SIZE)"
else
    echo "ERROR: Backup failed!"
    exit 1
fi

# Remove old backups
find "$BACKUP_DIR" -name "*.dump" -mtime "+$RETENTION_DAYS" -delete
echo "Removed backups older than $RETENTION_DAYS days"

echo "Backup complete at $(date)"
```

```bash
# ตั้ง cron job สำหรับ backup อัตโนมัติ
crontab -e

# Backup ทุกวัน เวลา 2:00 AM
0 2 * * * /path/to/backup-postgres.sh >> /var/log/pg-backup.log 2>&1
```

---

## 13. แก้ปัญหา Common Errors

### 13.1 Connection Refused

```bash
# Error: could not connect to server: Connection refused
# Is the server running on host "localhost" (127.0.0.1) and accepting
# TCP/IP connections on port 5432?

# ตรวจสอบว่า PostgreSQL ทำงานอยู่
sudo systemctl status postgresql
# หรือใน Docker:
docker ps | grep postgres

# ตรวจสอบ port
netstat -tlnp | grep 5432
# หรือ
ss -tlnp | grep 5432

# ตรวจสอบ listen_addresses ใน postgresql.conf
SHOW listen_addresses;
# ถ้า = 'localhost' จะรับแค่ local connection
# เปลี่ยนเป็น '*' เพื่อรับทุก IP

# ตรวจสอบ firewall
sudo ufw status
sudo ufw allow 5432/tcp
```

### 13.2 Authentication Failed

```bash
# Error: password authentication failed for user "admin"

# ตรวจสอบ pg_hba.conf
sudo cat /etc/postgresql/16/main/pg_hba.conf

# รีเซ็ต password
sudo -u postgres psql
ALTER USER admin PASSWORD 'new_password';
\q

# ถ้า lock out จาก postgres ด้วย (แก้ pg_hba.conf ชั่วคราว)
sudo nano /etc/postgresql/16/main/pg_hba.conf
# เปลี่ยน:
# local   all   all   peer   →   local   all   all   trust
sudo systemctl reload postgresql

# Reset password
sudo -u postgres psql -c "ALTER USER postgres PASSWORD 'new_password';"

# แก้ pg_hba.conf กลับ
sudo nano /etc/postgresql/16/main/pg_hba.conf
# เปลี่ยนกลับ:
# local   all   all   trust   →   local   all   all   peer
sudo systemctl reload postgresql
```

### 13.3 Permission Denied

```sql
-- Error: permission denied for table users

-- ตรวจสอบว่า user มี permission อะไรบ้าง
SELECT grantee, privilege_type
FROM information_schema.role_table_grants
WHERE table_name = 'users' AND table_schema = 'app';

-- ตรวจสอบ schema permission
SELECT nspname, nspacl
FROM pg_namespace
WHERE nspname = 'app';

-- Grant ที่ขาดไป
GRANT USAGE ON SCHEMA app TO app_user;
GRANT SELECT ON app.users TO app_user;

-- ตรวจสอบ database permission
SELECT datname, datacl FROM pg_database WHERE datname = 'mydb';
GRANT CONNECT ON DATABASE mydb TO app_user;
```

### 13.4 Too Many Connections

```sql
-- Error: sorry, too many clients already

-- ดู connection limit
SHOW max_connections;  -- default: 100

-- ดู connections ปัจจุบัน
SELECT count(*), state
FROM pg_stat_activity
GROUP BY state;

-- ดู connections ต่อ user
SELECT usename, count(*), state
FROM pg_stat_activity
GROUP BY usename, state
ORDER BY count(*) DESC;

-- Kill idle connections (ที่ idle นานเกิน 1 ชั่วโมง)
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'idle'
  AND query_start < NOW() - INTERVAL '1 hour';

-- แก้ที่ดีกว่า: ใช้ PgBouncer (Connection Pooler)
-- แทนที่จะเพิ่ม max_connections (ใช้ memory มาก)
```

### 13.5 Disk Full

```sql
-- Error: could not extend file... No space left on device

-- ดู disk usage
-- (บน Linux)
df -h /var/lib/postgresql/

-- ดูขนาด tables ใหญ่สุด
SELECT 
    schemaname || '.' || relname AS table_name,
    pg_size_pretty(pg_total_relation_size(oid)) AS total_size
FROM pg_class
WHERE relkind = 'r'
ORDER BY pg_total_relation_size(oid) DESC
LIMIT 10;

-- ทำ VACUUM FULL เพื่อคืน space (ต้อง lock table!)
-- ใช้ VACUUM (ปกติ) แทนที่ดีกว่า
VACUUM ANALYZE;  -- คืน dead rows กลับไป free space

-- ลบ WAL เก่าๆ
-- ดู WAL size
SELECT pg_size_pretty(sum(size)) AS wal_size
FROM pg_ls_waldir();
```

### 13.6 ตรวจสอบ PostgreSQL Health

```sql
-- Health check script
-- รัน: psql -U admin -d mydb -f health-check.sql

-- 1. Version
SELECT version() AS postgresql_version;

-- 2. Database size
SELECT 
    current_database() AS database_name,
    pg_size_pretty(pg_database_size(current_database())) AS size;

-- 3. Active connections
SELECT 
    count(*) AS total_connections,
    count(*) FILTER (WHERE state = 'active') AS active,
    count(*) FILTER (WHERE state = 'idle') AS idle,
    count(*) FILTER (WHERE state = 'idle in transaction') AS idle_in_tx
FROM pg_stat_activity;

-- 4. Long running queries
SELECT pid, now() - query_start AS duration, LEFT(query, 100) AS query
FROM pg_stat_activity
WHERE state = 'active'
  AND query_start < NOW() - INTERVAL '30 seconds'
ORDER BY duration DESC;

-- 5. Cache hit ratio (ควรมากกว่า 99%)
SELECT 
    sum(heap_blks_read) AS heap_read,
    sum(heap_blks_hit) AS heap_hit,
    ROUND(sum(heap_blks_hit) / NULLIF(sum(heap_blks_hit) + sum(heap_blks_read), 0) * 100, 2) AS cache_hit_ratio
FROM pg_statio_user_tables;

-- 6. Table bloat (dead tuples)
SELECT 
    schemaname,
    relname AS table_name,
    n_dead_tup AS dead_tuples,
    n_live_tup AS live_tuples,
    last_autovacuum,
    last_autoanalyze
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC;

-- 7. Replication status (ถ้ามี replicas)
SELECT client_addr, state, sent_lsn, write_lsn, flush_lsn, replay_lsn,
    (sent_lsn - replay_lsn) AS replication_lag_bytes
FROM pg_stat_replication;
```

---

## สรุปบทที่ 2

| หัวข้อ | สิ่งที่ได้เรียน |
|--------|----------------|
| การติดตั้ง | Ubuntu, macOS (Homebrew), Docker |
| Config Files | postgresql.conf, pg_hba.conf |
| psql CLI | Meta-commands ทั้งหมด |
| GUI Tools | pgAdmin 4, DBeaver |
| Roles & Permissions | สร้าง roles, grant privileges |
| Connection String | Format, parameters |
| Backup & Restore | pg_dump, pg_restore |
| Troubleshooting | Common errors และวิธีแก้ |

**บทต่อไป:** SQL พื้นฐาน — เริ่มต้นเขียน queries จริง!

---

## แหล่งอ้างอิง

- [PostgreSQL Installation Guide](https://www.postgresql.org/docs/current/installation.html)
- [PostgreSQL Docker Image](https://hub.docker.com/_/postgres)
- [pgAdmin Documentation](https://www.pgadmin.org/docs/)
- [DBeaver Documentation](https://dbeaver.io/documentation/)
- [pg_dump Reference](https://www.postgresql.org/docs/current/app-pgdump.html)

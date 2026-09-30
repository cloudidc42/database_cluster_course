# Part 35: Docker Compose สำหรับ Database Cluster

## บทนำ

Docker Compose ช่วยให้เราสามารถ define และ run multi-container applications ได้อย่างง่ายดาย ในบทนี้เราจะสร้าง production-ready database cluster ที่ประกอบด้วย PostgreSQL Primary, PostgreSQL Replica, Redis, MinIO, และ Application server

---

## 1. โครงสร้างไฟล์

```
project/
├── docker-compose.yml           # Main compose file
├── docker-compose.override.yml  # Development overrides
├── docker-compose.prod.yml      # Production overrides
├── .env                         # Environment variables
├── .env.example                 # Template
│
├── postgres/
│   ├── primary/
│   │   ├── postgresql.conf      # Primary config
│   │   ├── pg_hba.conf         # Auth config
│   │   └── init/
│   │       ├── 01-create-users.sql
│   │       ├── 02-create-schema.sql
│   │       └── 03-seed-data.sql
│   └── replica/
│       └── setup-replica.sh     # Replica setup script
│
├── redis/
│   └── redis.conf              # Redis config
│
├── minio/
│   └── create-buckets.sh       # MinIO init script
│
├── nginx/
│   └── nginx.conf              # Load balancer config
│
└── app/
    └── Dockerfile              # Application Dockerfile
```

---

## 2. .env File

```bash
# .env - Environment Variables

# ===== PostgreSQL =====
POSTGRES_VERSION=16
POSTGRES_DB=appdb
POSTGRES_USER=appuser
POSTGRES_PASSWORD=changeme_secure_password_2024

# Replication
REPLICATION_USER=replicator
REPLICATION_PASSWORD=replication_password_2024

# Ports
POSTGRES_PRIMARY_PORT=5432
POSTGRES_REPLICA_PORT=5433

# ===== Redis =====
REDIS_VERSION=7
REDIS_PASSWORD=redis_password_2024
REDIS_PORT=6379
REDIS_MAX_MEMORY=256mb
REDIS_MAX_MEMORY_POLICY=allkeys-lru

# ===== MinIO =====
MINIO_VERSION=latest
MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD=minioadmin_password_2024
MINIO_PORT=9000
MINIO_CONSOLE_PORT=9001
MINIO_BUCKET_UPLOADS=uploads
MINIO_BUCKET_BACKUPS=backups

# ===== Application =====
APP_PORT=3000
APP_ENV=production
NODE_ENV=production
APP_SECRET=your_app_secret_key_here

# Database URLs
DATABASE_URL=postgresql://appuser:changeme_secure_password_2024@postgres-primary:5432/appdb
DATABASE_REPLICA_URL=postgresql://appuser:changeme_secure_password_2024@postgres-replica:5432/appdb
REDIS_URL=redis://:redis_password_2024@redis:6379/0

# MinIO
MINIO_ENDPOINT=http://minio:9000
AWS_ACCESS_KEY_ID=minioadmin
AWS_SECRET_ACCESS_KEY=minioadmin_password_2024

# ===== Network =====
NETWORK_SUBNET=172.20.0.0/16
```

---

## 3. Main docker-compose.yml

```yaml
# docker-compose.yml
version: '3.8'

# ===================================================
# Networks
# ===================================================
networks:
  app-network:
    driver: bridge
    ipam:
      config:
        - subnet: ${NETWORK_SUBNET:-172.20.0.0/16}
  
  db-network:
    driver: bridge
    internal: true  # ไม่มี internet access
  
  monitoring-network:
    driver: bridge

# ===================================================
# Volumes
# ===================================================
volumes:
  postgres-primary-data:
    driver: local
  postgres-replica-data:
    driver: local
  redis-data:
    driver: local
  minio-data:
    driver: local
  app-logs:
    driver: local

# ===================================================
# Services
# ===================================================
services:

  # =================================================
  # PostgreSQL Primary
  # =================================================
  postgres-primary:
    image: postgres:${POSTGRES_VERSION:-16}
    container_name: postgres-primary
    restart: unless-stopped
    
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      REPLICATION_USER: ${REPLICATION_USER}
      REPLICATION_PASSWORD: ${REPLICATION_PASSWORD}
      # Performance tuning via env
      POSTGRES_INITDB_ARGS: "--data-checksums --auth-host=scram-sha-256"
    
    volumes:
      # Data directory
      - postgres-primary-data:/var/lib/postgresql/data
      # Custom configs
      - ./postgres/primary/postgresql.conf:/etc/postgresql/postgresql.conf:ro
      - ./postgres/primary/pg_hba.conf:/etc/postgresql/pg_hba.conf:ro
      # Init scripts (run once on first start)
      - ./postgres/primary/init:/docker-entrypoint-initdb.d:ro
    
    command: >
      postgres
      -c config_file=/etc/postgresql/postgresql.conf
      -c hba_file=/etc/postgresql/pg_hba.conf
    
    ports:
      - "${POSTGRES_PRIMARY_PORT:-5432}:5432"
    
    networks:
      - app-network
      - db-network
    
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    
    shm_size: '256mb'  # Shared memory สำหรับ sort operations
    
    ulimits:
      nofile:
        soft: 65536
        hard: 65536
    
    logging:
      driver: "json-file"
      options:
        max-size: "100m"
        max-file: "5"

  # =================================================
  # PostgreSQL Replica
  # =================================================
  postgres-replica:
    image: postgres:${POSTGRES_VERSION:-16}
    container_name: postgres-replica
    restart: unless-stopped
    
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      PGPASSWORD: ${REPLICATION_PASSWORD}  # สำหรับ pg_basebackup
      PRIMARY_HOST: postgres-primary
      PRIMARY_PORT: 5432
      REPLICATION_USER: ${REPLICATION_USER}
      REPLICATION_PASSWORD: ${REPLICATION_PASSWORD}
    
    volumes:
      - postgres-replica-data:/var/lib/postgresql/data
      - ./postgres/replica/setup-replica.sh:/setup-replica.sh:ro
    
    # Override entrypoint เพื่อ setup replication
    entrypoint: ["/bin/bash", "/setup-replica.sh"]
    
    ports:
      - "${POSTGRES_REPLICA_PORT:-5433}:5432"
    
    networks:
      - app-network
      - db-network
    
    depends_on:
      postgres-primary:
        condition: service_healthy
    
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 60s
    
    shm_size: '256mb'
    
    logging:
      driver: "json-file"
      options:
        max-size: "100m"
        max-file: "5"

  # =================================================
  # Redis
  # =================================================
  redis:
    image: redis:${REDIS_VERSION:-7}-alpine
    container_name: redis
    restart: unless-stopped
    
    command: redis-server /etc/redis/redis.conf
    
    volumes:
      - redis-data:/data
      - ./redis/redis.conf:/etc/redis/redis.conf:ro
    
    ports:
      - "${REDIS_PORT:-6379}:6379"
    
    networks:
      - app-network
    
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s
    
    sysctls:
      - net.core.somaxconn=65535
    
    logging:
      driver: "json-file"
      options:
        max-size: "50m"
        max-file: "3"

  # =================================================
  # MinIO Object Storage
  # =================================================
  minio:
    image: minio/minio:${MINIO_VERSION:-latest}
    container_name: minio
    restart: unless-stopped
    
    environment:
      MINIO_ROOT_USER: ${MINIO_ROOT_USER}
      MINIO_ROOT_PASSWORD: ${MINIO_ROOT_PASSWORD}
      MINIO_BROWSER_REDIRECT_URL: http://localhost:${MINIO_CONSOLE_PORT:-9001}
    
    command: server /data --console-address ":9001"
    
    volumes:
      - minio-data:/data
    
    ports:
      - "${MINIO_PORT:-9000}:9000"
      - "${MINIO_CONSOLE_PORT:-9001}:9001"
    
    networks:
      - app-network
    
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3
      start_period: 30s
    
    logging:
      driver: "json-file"
      options:
        max-size: "50m"
        max-file: "3"

  # MinIO Setup (one-time bucket creation)
  minio-setup:
    image: minio/mc:latest
    container_name: minio-setup
    restart: "no"
    
    environment:
      MINIO_ROOT_USER: ${MINIO_ROOT_USER}
      MINIO_ROOT_PASSWORD: ${MINIO_ROOT_PASSWORD}
      MINIO_BUCKET_UPLOADS: ${MINIO_BUCKET_UPLOADS:-uploads}
      MINIO_BUCKET_BACKUPS: ${MINIO_BUCKET_BACKUPS:-backups}
    
    volumes:
      - ./minio/create-buckets.sh:/create-buckets.sh:ro
    
    entrypoint: ["/bin/sh", "/create-buckets.sh"]
    
    networks:
      - app-network
    
    depends_on:
      minio:
        condition: service_healthy

  # =================================================
  # Application
  # =================================================
  app:
    build:
      context: ./app
      dockerfile: Dockerfile
      args:
        NODE_VERSION: "20"
    
    image: myapp:latest
    container_name: app
    restart: unless-stopped
    
    environment:
      NODE_ENV: ${NODE_ENV:-production}
      APP_PORT: ${APP_PORT:-3000}
      APP_SECRET: ${APP_SECRET}
      DATABASE_URL: ${DATABASE_URL}
      DATABASE_REPLICA_URL: ${DATABASE_REPLICA_URL}
      REDIS_URL: ${REDIS_URL}
      MINIO_ENDPOINT: ${MINIO_ENDPOINT}
      AWS_ACCESS_KEY_ID: ${AWS_ACCESS_KEY_ID}
      AWS_SECRET_ACCESS_KEY: ${AWS_SECRET_ACCESS_KEY}
    
    volumes:
      - app-logs:/app/logs
    
    ports:
      - "${APP_PORT:-3000}:3000"
    
    networks:
      - app-network
    
    depends_on:
      postgres-primary:
        condition: service_healthy
      redis:
        condition: service_healthy
      minio:
        condition: service_healthy
    
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    
    logging:
      driver: "json-file"
      options:
        max-size: "100m"
        max-file: "10"

  # =================================================
  # Nginx (Load Balancer / Reverse Proxy)
  # =================================================
  nginx:
    image: nginx:alpine
    container_name: nginx
    restart: unless-stopped
    
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    
    ports:
      - "80:80"
      - "443:443"
    
    networks:
      - app-network
    
    depends_on:
      app:
        condition: service_healthy
    
    healthcheck:
      test: ["CMD", "nginx", "-t"]
      interval: 30s
      timeout: 10s
      retries: 3

  # =================================================
  # PgBouncer (Connection Pooler)
  # =================================================
  pgbouncer:
    image: bitnami/pgbouncer:latest
    container_name: pgbouncer
    restart: unless-stopped
    
    environment:
      POSTGRESQL_HOST: postgres-primary
      POSTGRESQL_PORT: 5432
      POSTGRESQL_DATABASE: ${POSTGRES_DB}
      POSTGRESQL_USERNAME: ${POSTGRES_USER}
      POSTGRESQL_PASSWORD: ${POSTGRES_PASSWORD}
      PGBOUNCER_PORT: 6432
      PGBOUNCER_DATABASE: ${POSTGRES_DB}
      PGBOUNCER_POOL_MODE: transaction
      PGBOUNCER_MAX_CLIENT_CONN: 200
      PGBOUNCER_DEFAULT_POOL_SIZE: 25
      PGBOUNCER_MIN_POOL_SIZE: 5
      PGBOUNCER_AUTH_TYPE: scram-sha-256
    
    ports:
      - "6432:6432"
    
    networks:
      - app-network
      - db-network
    
    depends_on:
      postgres-primary:
        condition: service_healthy
    
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -h localhost -p 6432 -U ${POSTGRES_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5
```

---

## 4. PostgreSQL Primary Configuration

```ini
# postgres/primary/postgresql.conf

# ===================================================
# CONNECTIONS AND AUTHENTICATION
# ===================================================
listen_addresses = '*'
port = 5432
max_connections = 200
superuser_reserved_connections = 3

# ===================================================
# RESOURCE USAGE
# ===================================================
# Memory (ปรับตาม RAM ที่มี)
shared_buffers = 256MB          # 25% of RAM
effective_cache_size = 768MB    # 75% of RAM
work_mem = 4MB                  # per sort/hash operation
maintenance_work_mem = 64MB     # for VACUUM, CREATE INDEX
huge_pages = try

# Background writer
bgwriter_delay = 200ms
bgwriter_lru_maxpages = 100
bgwriter_lru_multiplier = 2.0
bgwriter_flush_after = 512kB

# ===================================================
# WRITE AHEAD LOG
# ===================================================
wal_level = replica             # สำหรับ streaming replication
wal_compression = on
wal_buffers = 16MB
checkpoint_completion_target = 0.9
checkpoint_timeout = 10min
max_wal_size = 1GB
min_wal_size = 80MB

# WAL Archiving (uncomment ถ้าต้องการ PITR)
# archive_mode = on
# archive_command = 'test ! -f /archive/%f && cp %p /archive/%f'
# archive_cleanup_command = 'pg_archivecleanup /archive %r'

# ===================================================
# REPLICATION
# ===================================================
max_wal_senders = 5             # จำนวน replica สูงสุด
wal_keep_size = 256MB           # เก็บ WAL สำหรับ replica lag
max_replication_slots = 5
hot_standby = on
synchronous_commit = on         # off เพื่อ performance (risk!)
# synchronous_standby_names = 'replica1'  # synchronous replication

# ===================================================
# QUERY TUNING
# ===================================================
random_page_cost = 1.1          # สำหรับ SSD (default 4.0)
effective_io_concurrency = 200  # สำหรับ SSD
parallel_tuple_cost = 0.1
parallel_setup_cost = 1000
max_parallel_workers_per_gather = 2
max_parallel_workers = 4
max_parallel_maintenance_workers = 2

# Planner
enable_partitionwise_join = on
enable_partitionwise_aggregate = on
jit = on

# ===================================================
# REPORTING AND LOGGING
# ===================================================
log_destination = 'stderr'
logging_collector = on
log_directory = 'pg_log'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_rotation_age = 1d
log_rotation_size = 100MB
log_truncate_on_rotation = off

log_min_duration_statement = 1000   # log queries > 1 second
log_checkpoints = on
log_connections = off               # on ใน dev, off ใน prod
log_disconnections = off
log_lock_waits = on
log_temp_files = 0
log_autovacuum_min_duration = 250ms

log_line_prefix = '%m [%p] %q%u@%d '
log_timezone = 'Asia/Bangkok'

# ===================================================
# AUTOVACUUM
# ===================================================
autovacuum = on
autovacuum_max_workers = 3
autovacuum_naptime = 1min
autovacuum_vacuum_threshold = 50
autovacuum_vacuum_scale_factor = 0.05   # 5% of table
autovacuum_analyze_threshold = 50
autovacuum_analyze_scale_factor = 0.02  # 2% of table
autovacuum_vacuum_cost_delay = 2ms
autovacuum_vacuum_cost_limit = 200

# ===================================================
# CLIENT CONNECTION DEFAULTS
# ===================================================
datestyle = 'iso, mdy'
timezone = 'Asia/Bangkok'
lc_messages = 'en_US.UTF-8'
lc_monetary = 'th_TH.UTF-8'
lc_numeric = 'th_TH.UTF-8'
lc_time = 'th_TH.UTF-8'
default_text_search_config = 'pg_catalog.english'

# ===================================================
# LOCK MANAGEMENT
# ===================================================
deadlock_timeout = 1s
lock_timeout = 30s
statement_timeout = 0           # 0 = no timeout (ตั้งที่ application)
idle_in_transaction_session_timeout = 60s

# ===================================================
# EXTENSIONS PRELOAD
# ===================================================
shared_preload_libraries = 'pg_stat_statements, auto_explain'

# pg_stat_statements
pg_stat_statements.max = 10000
pg_stat_statements.track = all

# auto_explain (log slow query plans)
auto_explain.log_min_duration = '5s'
auto_explain.log_analyze = on
```

### pg_hba.conf

```
# postgres/primary/pg_hba.conf
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# Local connections
local   all             all                                     trust
local   all             postgres                                peer

# IPv4 local connections
host    all             all             127.0.0.1/32            scram-sha-256

# Docker network connections
host    all             all             172.20.0.0/16           scram-sha-256

# Replication connections
host    replication     replicator      172.20.0.0/16           scram-sha-256

# IPv6 local connections
host    all             all             ::1/128                 scram-sha-256
```

### Init Scripts

```sql
-- postgres/primary/init/01-create-users.sql

-- Replication user
CREATE USER replicator WITH
    REPLICATION
    ENCRYPTED PASSWORD 'replication_password_2024'
    CONNECTION LIMIT 5;

-- Read-only user for replica queries
CREATE USER readonly WITH
    ENCRYPTED PASSWORD 'readonly_password_2024'
    CONNECTION LIMIT 20;

-- Application user (already created via POSTGRES_USER)
GRANT CONNECT ON DATABASE appdb TO readonly;

-- Monitor user
CREATE USER monitor WITH
    ENCRYPTED PASSWORD 'monitor_password_2024';
GRANT pg_monitor TO monitor;
```

```sql
-- postgres/primary/init/02-create-schema.sql

-- Extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- ตัวอย่าง schema
CREATE SCHEMA IF NOT EXISTS app;

CREATE TABLE app.users (
    id SERIAL PRIMARY KEY,
    uuid UUID DEFAULT uuid_generate_v4() UNIQUE,
    email TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL,
    password_hash TEXT NOT NULL,
    role TEXT DEFAULT 'user',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_users_email ON app.users(email);
CREATE INDEX idx_users_uuid ON app.users(uuid);

-- Grant privileges
GRANT USAGE ON SCHEMA app TO appuser;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA app TO appuser;
GRANT USAGE ON ALL SEQUENCES IN SCHEMA app TO appuser;

GRANT USAGE ON SCHEMA app TO readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA app TO readonly;

-- Default privileges for future tables
ALTER DEFAULT PRIVILEGES IN SCHEMA app 
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO appuser;
ALTER DEFAULT PRIVILEGES IN SCHEMA app 
    GRANT SELECT ON TABLES TO readonly;
```

---

## 5. Replica Setup Script

```bash
#!/bin/bash
# postgres/replica/setup-replica.sh

set -e

# Configuration from environment
PRIMARY_HOST="${PRIMARY_HOST:-postgres-primary}"
PRIMARY_PORT="${PRIMARY_PORT:-5432}"
REPLICATION_USER="${REPLICATION_USER:-replicator}"
REPLICATION_PASSWORD="${REPLICATION_PASSWORD}"
PGDATA="${PGDATA:-/var/lib/postgresql/data}"

echo "==================================================="
echo "Setting up PostgreSQL Replica"
echo "Primary: ${PRIMARY_HOST}:${PRIMARY_PORT}"
echo "==================================================="

# Function: wait for primary to be ready
wait_for_primary() {
    local max_attempts=60
    local attempt=0
    
    echo "Waiting for primary to be ready..."
    
    while [ $attempt -lt $max_attempts ]; do
        if pg_isready -h "$PRIMARY_HOST" -p "$PRIMARY_PORT" -U "$REPLICATION_USER" 2>/dev/null; then
            echo "Primary is ready!"
            return 0
        fi
        
        echo "Attempt $((attempt+1))/$max_attempts: Primary not ready, waiting 2s..."
        sleep 2
        attempt=$((attempt + 1))
    done
    
    echo "ERROR: Primary did not become ready in time"
    exit 1
}

# Check ถ้า PGDATA มีข้อมูลอยู่แล้ว
if [ -f "$PGDATA/PG_VERSION" ]; then
    echo "Data directory already initialized, starting replica..."
    exec docker-entrypoint.sh postgres \
        -c hot_standby=on \
        -c primary_conninfo="host=${PRIMARY_HOST} port=${PRIMARY_PORT} user=${REPLICATION_USER} password=${REPLICATION_PASSWORD} application_name=replica1" \
        -c recovery_target_timeline=latest \
        -c max_connections=100 \
        -c hot_standby_feedback=on
fi

# Wait for primary
wait_for_primary

# สร้าง directory
mkdir -p "$PGDATA"
chmod 700 "$PGDATA"

# Create .pgpass for pg_basebackup
cat > ~/.pgpass << EOF
${PRIMARY_HOST}:${PRIMARY_PORT}:replication:${REPLICATION_USER}:${REPLICATION_PASSWORD}
EOF
chmod 600 ~/.pgpass

echo "Running pg_basebackup from ${PRIMARY_HOST}..."

# Run pg_basebackup
pg_basebackup \
    -h "$PRIMARY_HOST" \
    -p "$PRIMARY_PORT" \
    -U "$REPLICATION_USER" \
    -D "$PGDATA" \
    --wal-method=stream \
    --checkpoint=fast \
    --progress \
    --verbose \
    -R  # สร้าง standby.signal และ primary_conninfo ใน postgresql.auto.conf

echo "pg_basebackup complete!"

# สร้าง recovery configuration
cat >> "$PGDATA/postgresql.auto.conf" << EOF

# Replica Configuration
primary_conninfo = 'host=${PRIMARY_HOST} port=${PRIMARY_PORT} user=${REPLICATION_USER} password=${REPLICATION_PASSWORD} application_name=replica1'
recovery_target_timeline = 'latest'
hot_standby = on
hot_standby_feedback = on
max_standby_streaming_delay = 30s
max_standby_archive_delay = 60s
EOF

# สร้าง standby.signal (PG12+)
touch "$PGDATA/standby.signal"

echo "Starting replica PostgreSQL..."

# Start PostgreSQL as replica
exec docker-entrypoint.sh postgres \
    -c hot_standby=on \
    -c max_connections=100 \
    -c shared_buffers=256MB \
    -c effective_cache_size=768MB \
    -c random_page_cost=1.1 \
    -c effective_io_concurrency=200
```

---

## 6. Redis Configuration

```conf
# redis/redis.conf

# ===================================================
# NETWORK
# ===================================================
bind 0.0.0.0
port 6379
protected-mode yes
tcp-backlog 511
timeout 0
tcp-keepalive 300

# ===================================================
# SECURITY
# ===================================================
requirepass redis_password_2024
# rename-command FLUSHALL ""    # ปิดคำสั่งอันตราย
# rename-command FLUSHDB ""
# rename-command CONFIG ""

# ===================================================
# MEMORY
# ===================================================
maxmemory 256mb
maxmemory-policy allkeys-lru
# Policies:
# allkeys-lru: evict least recently used keys
# volatile-lru: evict lru keys with expire
# allkeys-random: evict random keys
# volatile-random: evict random keys with expire
# volatile-ttl: evict keys with shortest TTL
# noeviction: return error (สำหรับ persistent store)

maxmemory-samples 5
lazyfree-lazy-eviction no
lazyfree-lazy-expire no
lazyfree-lazy-server-del no

# ===================================================
# PERSISTENCE
# ===================================================
# RDB Snapshots
save 900 1      # save ถ้า 1 key เปลี่ยนใน 900 วินาที
save 300 10     # save ถ้า 10 keys เปลี่ยนใน 300 วินาที
save 60 10000   # save ถ้า 10000 keys เปลี่ยนใน 60 วินาที

stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
dbfilename dump.rdb
dir /data

# AOF (Append Only File) - ปิดถ้าใช้แค่ cache
appendonly no
# appendonly yes
# appendfilename "appendonly.aof"
# appendfsync everysec   # always, everysec, no
# no-appendfsync-on-rewrite no
# auto-aof-rewrite-percentage 100
# auto-aof-rewrite-min-size 64mb

# ===================================================
# SLOW LOG
# ===================================================
slowlog-log-slower-than 10000   # microseconds (10ms)
slowlog-max-len 128

# ===================================================
# ADVANCED
# ===================================================
hz 10
dynamic-hz yes
aof-rewrite-incremental-fsync yes
rdb-save-incremental-fsync yes

# Notifications
notify-keyspace-events ""
# เปิดถ้าต้องการ keyspace notifications:
# notify-keyspace-events "Ex"  # expired events

# ===================================================
# LOGGING
# ===================================================
loglevel notice
logfile ""   # "" = stdout
```

---

## 7. MinIO Create Buckets Script

```bash
#!/bin/sh
# minio/create-buckets.sh

set -e

MINIO_HOST="minio:9000"
MC_ALIAS="local"

echo "Waiting for MinIO to start..."
until curl -sf "http://${MINIO_HOST}/minio/health/live"; do
    echo "MinIO not ready, waiting..."
    sleep 2
done

echo "MinIO is ready!"

# Configure mc alias
mc alias set $MC_ALIAS \
    "http://${MINIO_HOST}" \
    "${MINIO_ROOT_USER}" \
    "${MINIO_ROOT_PASSWORD}"

# Create buckets
for BUCKET in \
    "${MINIO_BUCKET_UPLOADS:-uploads}" \
    "${MINIO_BUCKET_BACKUPS:-backups}" \
    "avatars" \
    "temp"; do
    
    if mc ls "${MC_ALIAS}/${BUCKET}" 2>/dev/null; then
        echo "Bucket ${BUCKET} already exists"
    else
        mc mb "${MC_ALIAS}/${BUCKET}"
        echo "Created bucket: ${BUCKET}"
    fi
done

# Set bucket policies
# Public read for uploads
mc anonymous set download "${MC_ALIAS}/uploads"
echo "Set public read policy for uploads"

# Private for backups and temp
mc anonymous set none "${MC_ALIAS}/backups"
mc anonymous set none "${MC_ALIAS}/temp"

# Set lifecycle rules for temp bucket (delete after 7 days)
mc ilm rule add \
    --expiry-days 7 \
    "${MC_ALIAS}/temp"

echo "MinIO setup complete!"
mc ls $MC_ALIAS
```

---

## 8. Application Dockerfile

```dockerfile
# app/Dockerfile

# ===== Build Stage =====
FROM node:20-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./
COPY tsconfig.json ./

# Install dependencies
RUN npm ci --only=production && \
    npm ci && \
    npm run build

# ===== Production Stage =====
FROM node:20-alpine AS production

# Security: non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

# Copy built files
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY package.json ./

# Create logs directory
RUN mkdir -p /app/logs && chown -R appuser:appgroup /app

USER appuser

EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
    CMD curl -f http://localhost:3000/health || exit 1

CMD ["node", "dist/index.js"]
```

---

## 9. docker-compose.override.yml (Development)

```yaml
# docker-compose.override.yml
# Development overrides - loaded automatically

version: '3.8'

services:
  postgres-primary:
    ports:
      - "5432:5432"  # expose locally
    volumes:
      - ./postgres/primary/init:/docker-entrypoint-initdb.d:ro
    environment:
      POSTGRES_PASSWORD: dev_password  # simpler password in dev
    command: >
      postgres
      -c config_file=/etc/postgresql/postgresql.conf
      -c log_min_duration_statement=0
      -c log_connections=on
      -c log_disconnections=on

  postgres-replica:
    ports:
      - "5433:5432"

  redis:
    ports:
      - "6379:6379"
    command: redis-server /etc/redis/redis.conf --loglevel verbose

  minio:
    ports:
      - "9000:9000"
      - "9001:9001"

  app:
    build:
      context: ./app
      target: builder   # Use build stage in dev
    command: npm run dev  # Hot reload
    volumes:
      - ./app/src:/app/src:ro    # Mount source for hot reload
      - app-logs:/app/logs
    environment:
      NODE_ENV: development
      LOG_LEVEL: debug
    ports:
      - "3000:3000"
      - "9229:9229"  # Node.js debug port

  # Development tools
  adminer:
    image: adminer:latest
    container_name: adminer
    restart: unless-stopped
    ports:
      - "8080:8080"
    networks:
      - app-network
    depends_on:
      postgres-primary:
        condition: service_healthy

  redis-commander:
    image: rediscommander/redis-commander:latest
    container_name: redis-commander
    environment:
      REDIS_HOSTS: "local:redis:6379:0:${REDIS_PASSWORD}"
    ports:
      - "8081:8081"
    networks:
      - app-network
    depends_on:
      redis:
        condition: service_healthy
```

---

## 10. docker-compose.prod.yml (Production)

```yaml
# docker-compose.prod.yml
# Production overrides

version: '3.8'

services:
  postgres-primary:
    ports: []  # ไม่ expose port ออกนอก (ใช้ network แทน)
    
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 4G
        reservations:
          cpus: '1'
          memory: 2G
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3

  postgres-replica:
    ports: []
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 4G

  redis:
    ports: []
    deploy:
      resources:
        limits:
          cpus: '1'
          memory: 512M

  app:
    deploy:
      replicas: 3  # 3 app instances
      resources:
        limits:
          cpus: '1'
          memory: 512M
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
      update_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
        order: start-first

  nginx:
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.prod.conf:/etc/nginx/nginx.conf:ro
      - ./certs:/etc/nginx/certs:ro  # SSL certs
```

---

## 11. Nginx Configuration

```nginx
# nginx/nginx.conf

worker_processes auto;
worker_rlimit_nofile 65535;

events {
    worker_connections 4096;
    use epoll;
    multi_accept on;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # Logging
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent"';
    
    access_log /var/log/nginx/access.log main;
    error_log /var/log/nginx/error.log warn;

    # Performance
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    gzip on;
    gzip_types text/plain application/json application/javascript text/css;

    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api:10m rate=100r/m;

    # Upstream (app instances)
    upstream app_backend {
        least_conn;
        server app:3000 max_fails=3 fail_timeout=30s;
        keepalive 32;
    }

    server {
        listen 80;
        server_name _;

        # Health check endpoint
        location /health {
            proxy_pass http://app_backend;
            access_log off;
        }

        # API rate limiting
        location /api/ {
            limit_req zone=api burst=20 nodelay;
            proxy_pass http://app_backend;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_cache_bypass $http_upgrade;
            
            # Timeouts
            proxy_connect_timeout 30s;
            proxy_send_timeout 30s;
            proxy_read_timeout 30s;
        }

        # All other requests
        location / {
            proxy_pass http://app_backend;
            proxy_http_version 1.1;
            proxy_set_header Connection '';
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }
}
```

---

## 12. Usage Commands

```bash
# ===================================================
# Development
# ===================================================

# Start all services
docker-compose up -d

# Start with build
docker-compose up -d --build

# View logs
docker-compose logs -f
docker-compose logs -f postgres-primary
docker-compose logs -f app

# Stop all
docker-compose down

# Stop and remove volumes (CAUTION: deletes data!)
docker-compose down -v

# ===================================================
# Production
# ===================================================

# Start production
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# Scale app instances
docker-compose up -d --scale app=3

# Rolling update
docker-compose up -d --no-deps --build app

# ===================================================
# Maintenance
# ===================================================

# Connect to PostgreSQL primary
docker-compose exec postgres-primary psql -U appuser -d appdb

# Connect to PostgreSQL replica
docker-compose exec postgres-replica psql -U appuser -d appdb

# Check replication status
docker-compose exec postgres-primary psql -U postgres -c "SELECT * FROM pg_stat_replication;"

# Redis CLI
docker-compose exec redis redis-cli -a redis_password_2024

# Backup
docker-compose exec postgres-primary pg_dump -U appuser -Fc appdb > backup_$(date +%Y%m%d).dump

# Restore
docker-compose exec -T postgres-primary pg_restore -U appuser -d appdb < backup.dump

# Check health
docker-compose ps
docker-compose exec postgres-primary pg_isready

# ===================================================
# Monitoring
# ===================================================

# Container stats
docker stats

# Container resource usage
docker-compose top

# Network inspection
docker network ls
docker network inspect project_app-network
```

---

## 13. Health Check Scripts

```bash
#!/bin/bash
# scripts/healthcheck.sh

echo "=== Service Health Check ==="
echo ""

# PostgreSQL Primary
echo -n "PostgreSQL Primary: "
if docker-compose exec -T postgres-primary pg_isready -q 2>/dev/null; then
    echo "✓ HEALTHY"
else
    echo "✗ UNHEALTHY"
fi

# Replication status
echo -n "Replication: "
REPLICA_COUNT=$(docker-compose exec -T postgres-primary psql -U postgres -t -c "SELECT COUNT(*) FROM pg_stat_replication WHERE state='streaming'" 2>/dev/null | tr -d ' ')
if [ "$REPLICA_COUNT" -gt 0 ]; then
    echo "✓ $REPLICA_COUNT replica(s) streaming"
else
    echo "⚠ No streaming replicas"
fi

# PostgreSQL Replica
echo -n "PostgreSQL Replica: "
if docker-compose exec -T postgres-replica pg_isready -q 2>/dev/null; then
    IS_STANDBY=$(docker-compose exec -T postgres-replica psql -U postgres -t -c "SELECT pg_is_in_recovery()" 2>/dev/null | tr -d ' ')
    if [ "$IS_STANDBY" = "t" ]; then
        echo "✓ HEALTHY (standby mode)"
    else
        echo "⚠ Running but not in standby mode"
    fi
else
    echo "✗ UNHEALTHY"
fi

# Redis
echo -n "Redis: "
if docker-compose exec -T redis redis-cli -a "${REDIS_PASSWORD}" ping 2>/dev/null | grep -q PONG; then
    echo "✓ HEALTHY"
else
    echo "✗ UNHEALTHY"
fi

# MinIO
echo -n "MinIO: "
if curl -sf "http://localhost:${MINIO_PORT:-9000}/minio/health/live" 2>/dev/null; then
    echo "✓ HEALTHY"
else
    echo "✗ UNHEALTHY"
fi

# App
echo -n "Application: "
if curl -sf "http://localhost:${APP_PORT:-3000}/health" 2>/dev/null; then
    echo "✓ HEALTHY"
else
    echo "✗ UNHEALTHY"
fi

echo ""
echo "=== Container Status ==="
docker-compose ps
```

---

## 14. Monitoring with Prometheus + Grafana (Optional)

```yaml
# docker-compose.monitoring.yml (เพิ่มเป็น optional stack)

version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus
    ports:
      - "9090:9090"
    networks:
      - app-network
      - monitoring-network

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    volumes:
      - grafana-data:/var/lib/grafana
      - ./monitoring/grafana/dashboards:/etc/grafana/provisioning/dashboards:ro
    ports:
      - "3001:3000"
    networks:
      - monitoring-network
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin_password

  postgres-exporter:
    image: prometheuscommunity/postgres-exporter:latest
    container_name: postgres-exporter
    restart: unless-stopped
    environment:
      DATA_SOURCE_NAME: "postgresql://monitor:monitor_password@postgres-primary:5432/postgres?sslmode=disable"
    ports:
      - "9187:9187"
    networks:
      - app-network
      - monitoring-network
    depends_on:
      postgres-primary:
        condition: service_healthy

  redis-exporter:
    image: oliver006/redis_exporter:latest
    container_name: redis-exporter
    restart: unless-stopped
    environment:
      REDIS_ADDR: redis:6379
      REDIS_PASSWORD: ${REDIS_PASSWORD}
    ports:
      - "9121:9121"
    networks:
      - app-network
      - monitoring-network

volumes:
  prometheus-data:
  grafana-data:
```

---

## 15. Complete Startup Guide

```bash
#!/bin/bash
# scripts/startup.sh
# Complete startup guide

set -e

echo "=========================================="
echo "Database Cluster Startup Script"
echo "=========================================="

# 1. Check requirements
echo "Checking requirements..."
command -v docker >/dev/null 2>&1 || { echo "Docker not found!"; exit 1; }
command -v docker-compose >/dev/null 2>&1 || { echo "Docker Compose not found!"; exit 1; }

# 2. Check .env file
if [ ! -f ".env" ]; then
    echo "Creating .env from .env.example..."
    cp .env.example .env
    echo "Please edit .env and set your passwords!"
    exit 1
fi

# 3. Create required directories
echo "Creating directories..."
mkdir -p postgres/primary/init
mkdir -p postgres/replica
mkdir -p redis
mkdir -p minio
mkdir -p nginx
mkdir -p app
mkdir -p backup/daily backup/weekly backup/monthly

# 4. Pull images
echo "Pulling Docker images..."
docker-compose pull --ignore-pull-failures

# 5. Start infrastructure first
echo "Starting PostgreSQL Primary..."
docker-compose up -d postgres-primary

echo "Waiting for Primary to be healthy..."
until docker-compose exec -T postgres-primary pg_isready -U postgres; do
    sleep 2
done

echo "Starting Replica..."
docker-compose up -d postgres-replica

echo "Starting Redis..."
docker-compose up -d redis

echo "Starting MinIO..."
docker-compose up -d minio

echo "Running MinIO setup..."
docker-compose up minio-setup

echo "Starting Application..."
docker-compose up -d app

echo "Starting Nginx..."
docker-compose up -d nginx

# 6. Health check
echo ""
echo "Performing health checks..."
sleep 5

docker-compose ps

echo ""
echo "=========================================="
echo "Startup Complete!"
echo ""
echo "Services:"
echo "  PostgreSQL Primary: localhost:${POSTGRES_PRIMARY_PORT:-5432}"
echo "  PostgreSQL Replica: localhost:${POSTGRES_REPLICA_PORT:-5433}"
echo "  Redis:              localhost:${REDIS_PORT:-6379}"
echo "  MinIO API:          http://localhost:${MINIO_PORT:-9000}"
echo "  MinIO Console:      http://localhost:${MINIO_CONSOLE_PORT:-9001}"
echo "  Application:        http://localhost:${APP_PORT:-3000}"
echo "=========================================="
```

---

## สรุป

Docker Compose สำหรับ Database Cluster ที่สมบูรณ์ประกอบด้วย:

1. **PostgreSQL Primary** - พร้อม streaming replication configuration
2. **PostgreSQL Replica** - auto-setup ด้วย pg_basebackup script
3. **Redis** - พร้อม password, maxmemory, persistence
4. **MinIO** - S3-compatible object storage พร้อม auto bucket creation
5. **PgBouncer** - connection pooler
6. **Nginx** - reverse proxy with load balancing
7. **Health checks** ทุก service พร้อม depends_on conditions
8. **Environment overrides** แยก dev/prod configs
9. **Monitoring** stack ด้วย Prometheus + Grafana

สำหรับ production ควร:
- ใช้ Docker Secrets แทน environment variables สำหรับ passwords
- กำหนด resource limits ให้ทุก service
- ตั้ง log rotation
- Monitor disk space ของ volumes
- Backup volumes อย่างสม่ำเสมอ
- ทดสอบ failover procedure เป็นประจำ

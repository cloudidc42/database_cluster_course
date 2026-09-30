# Part 53: PostgreSQL High Availability ด้วย Patroni

## บทนำ

Patroni เป็น High Availability solution สำหรับ PostgreSQL ที่ใช้กันอย่างแพร่หลายในระดับ production โดยมีบริษัทใหญ่ๆ อย่าง Zalando, GitLab และ Spotify ใช้งาน Patroni ทำงานร่วมกับ Distributed Configuration Store (DCS) เพื่อจัดการ leader election และ automatic failover บทนี้จะครอบคลุมการ setup และใช้งาน Patroni อย่างละเอียด

---

## 1. Patroni Architecture

### 1.1 Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        Patroni Cluster                          │
│                                                                 │
│  ┌─────────────────┐    ┌─────────────────┐    ┌────────────┐  │
│  │   PostgreSQL-1  │    │   PostgreSQL-2  │    │PostgreSQL-3│  │
│  │   (Leader)      │    │   (Replica)     │    │ (Replica)  │  │
│  │   Port: 5432    │    │   Port: 5432    │    │ Port: 5432 │  │
│  │   REST: 8008    │    │   REST: 8008    │    │ REST: 8008 │  │
│  └────────┬────────┘    └────────┬────────┘    └─────┬──────┘  │
│           │                     │                    │         │
│           └─────────────────────┴────────────────────┘         │
│                                 │                               │
│                    Streaming Replication                        │
└─────────────────────────────────┼───────────────────────────────┘
                                  │
                    ┌─────────────▼─────────────┐
                    │     DCS (etcd cluster)     │
                    │  Node1: :2379              │
                    │  Node2: :2379              │
                    │  Node3: :2379              │
                    └───────────────────────────┘
                                  │
                    ┌─────────────▼─────────────┐
                    │          HAProxy           │
                    │  :5000 → Leader (write)    │
                    │  :5001 → Replicas (read)   │
                    └───────────────────────────┘
```

### 1.2 Leader Election Process

```
1. Patroni nodes สมัคร session ใน DCS
2. DCS ตัดสินใจว่าใครเป็น Leader (distributed lock)
3. Leader: promote PostgreSQL → primary mode
4. Followers: start PostgreSQL → standby mode, connect ไปยัง Leader
5. Health checking: Patroni ตรวจสอบ PostgreSQL ทุก loop_wait วินาที
6. If Leader fails:
   - DCS session expires
   - Replicas แข่งกัน acquire lock
   - ผู้ชนะถูก promote เป็น primary ใหม่
   - ผู้แพ้ rewind/re-clone จาก primary ใหม่
```

### 1.3 DCS Options

| DCS | ข้อดี | ข้อเสีย |
|-----|-------|---------|
| **etcd** | Simple, reliable, ใช้กันมาก | ต้องดูแล cluster |
| **ZooKeeper** | Battle-tested, ใช้ใน Hadoop | Complex, Java |
| **Consul** | Service discovery built-in | ต้อง Consul agent |
| **Kubernetes** | ใช้ K8s API, ไม่ต้องตั้ง DCS แยก | ผูกกับ K8s |

---

## 2. Installation ด้วย Docker Compose

### 2.1 Directory Structure

```bash
mkdir -p patroni-cluster/{patroni1,patroni2,patroni3,etcd,haproxy}
cd patroni-cluster
```

### 2.2 etcd Cluster Configuration

```yaml
# etcd/etcd.conf.yml
name: 'etcd0'
initial-cluster-token: 'etcd-cluster-1'
initial-cluster: 'etcd0=http://etcd1:2380,etcd1=http://etcd2:2380,etcd2=http://etcd3:2380'
initial-cluster-state: 'new'
listen-peer-urls: 'http://0.0.0.0:2380'
listen-client-urls: 'http://0.0.0.0:2379'
advertise-client-urls: 'http://etcd1:2379'
initial-advertise-peer-urls: 'http://etcd1:2380'
data-dir: '/etcd-data'
```

### 2.3 Patroni Configuration Files

```yaml
# patroni1/patroni.yml
scope: postgres-cluster
namespace: /db/
name: postgresql1

restapi:
  listen: 0.0.0.0:8008
  connect_address: patroni1:8008
  authentication:
    username: patroni
    password: patroni-restapi-secret

etcd3:
  hosts:
  - etcd1:2379
  - etcd2:2379
  - etcd3:2379

bootstrap:
  # DCS configuration ที่จะ sync ไปยังทุก nodes
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576  # 1MB
    master_start_timeout: 300
    synchronous_mode: false
    synchronous_mode_strict: false
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        max_connections: 300
        shared_buffers: 512MB
        effective_cache_size: 2GB
        work_mem: 8MB
        maintenance_work_mem: 128MB
        wal_level: replica
        hot_standby: "on"
        wal_keep_size: 2GB
        max_wal_senders: 10
        max_replication_slots: 10
        wal_log_hints: "on"
        archive_mode: "off"
        log_min_duration_statement: 1000
        log_checkpoints: "on"
        log_lock_waits: "on"
        track_io_timing: "on"
        
  # Initialization script สำหรับ new cluster
  initdb:
  - encoding: UTF8
  - data-checksums
  
  # pg_hba.conf
  pg_hba:
  - local all all trust
  - host all all 127.0.0.1/32 md5
  - host all all ::1/128 md5
  - host all all 0.0.0.0/0 md5
  - host replication replicator 0.0.0.0/0 md5
  
  # สร้าง users ตอน bootstrap
  users:
    admin:
      password: admin-password
      options:
      - createrole
      - createdb
    appuser:
      password: app-password
    replicator:
      password: replicator-password
      options:
      - replication

postgresql:
  listen: 0.0.0.0:5432
  connect_address: patroni1:5432
  data_dir: /data/patroni
  bin_dir: /usr/lib/postgresql/15/bin
  
  pgpass: /tmp/pgpass0
  
  authentication:
    replication:
      username: replicator
      password: replicator-password
    superuser:
      username: postgres
      password: postgres-password
    rewind:
      username: rewind_user
      password: rewind-password
      
  parameters:
    unix_socket_directories: '/var/run/postgresql'
    
  # Callbacks: scripts ที่ run เมื่อ state เปลี่ยน
  callbacks:
    on_start: /scripts/on_start.sh
    on_stop: /scripts/on_stop.sh
    on_role_change: /scripts/on_role_change.sh

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false
```

```yaml
# patroni2/patroni.yml
scope: postgres-cluster
namespace: /db/
name: postgresql2

restapi:
  listen: 0.0.0.0:8008
  connect_address: patroni2:8008
  authentication:
    username: patroni
    password: patroni-restapi-secret

etcd3:
  hosts:
  - etcd1:2379
  - etcd2:2379
  - etcd3:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        max_connections: 300
        shared_buffers: 512MB
        wal_level: replica
        hot_standby: "on"
        max_wal_senders: 10
        max_replication_slots: 10
        wal_log_hints: "on"
        
  initdb:
  - encoding: UTF8
  - data-checksums
  
  pg_hba:
  - local all all trust
  - host all all 0.0.0.0/0 md5
  - host replication replicator 0.0.0.0/0 md5
  
  users:
    admin:
      password: admin-password
      options:
      - createrole
      - createdb

postgresql:
  listen: 0.0.0.0:5432
  connect_address: patroni2:5432
  data_dir: /data/patroni
  bin_dir: /usr/lib/postgresql/15/bin
  pgpass: /tmp/pgpass0
  
  authentication:
    replication:
      username: replicator
      password: replicator-password
    superuser:
      username: postgres
      password: postgres-password
    rewind:
      username: rewind_user
      password: rewind-password
      
  parameters:
    unix_socket_directories: '/var/run/postgresql'

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false
```

### 2.4 HAProxy Configuration

```
# haproxy/haproxy.cfg
global
    maxconn 100
    log stdout format raw local0

defaults
    log global
    mode tcp
    retries 2
    timeout client 30m
    timeout connect 4s
    timeout server 30m
    timeout check 5s

listen stats
    mode http
    bind *:7000
    stats enable
    stats uri /

# Primary (Write) endpoint - port 5000
frontend ft_postgresql_primary
    bind *:5000
    default_backend bk_postgresql_primary

backend bk_postgresql_primary
    option httpchk
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server patroni1 patroni1:5432 maxconn 50 check port 8008
    server patroni2 patroni2:5432 maxconn 50 check port 8008
    server patroni3 patroni3:5432 maxconn 50 check port 8008

# Replica (Read) endpoint - port 5001
frontend ft_postgresql_replicas
    bind *:5001
    default_backend bk_postgresql_replicas

backend bk_postgresql_replicas
    option httpchk GET /replica
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server patroni1 patroni1:5432 maxconn 100 check port 8008
    server patroni2 patroni2:5432 maxconn 100 check port 8008
    server patroni3 patroni3:5432 maxconn 100 check port 8008
```

HAProxy ใช้ Patroni REST API เพื่อตรวจสอบ:
- `GET /` หรือ `GET /master` → 200 ถ้า node เป็น primary
- `GET /replica` → 200 ถ้า node เป็น replica
- `GET /health` → 200 ถ้า node healthy

### 2.5 Docker Compose File

```yaml
# docker-compose.yml
version: '3.8'

networks:
  patroni:
    driver: bridge
    ipam:
      config:
      - subnet: 172.20.0.0/16

volumes:
  etcd1_data: {}
  etcd2_data: {}
  etcd3_data: {}
  patroni1_data: {}
  patroni2_data: {}
  patroni3_data: {}

services:
  # ─── etcd cluster ───────────────────────────────────────────
  etcd1:
    image: quay.io/coreos/etcd:v3.5.9
    hostname: etcd1
    networks:
      patroni:
        ipv4_address: 172.20.0.10
    environment:
      ETCD_NAME: etcd1
      ETCD_DATA_DIR: /etcd-data
      ETCD_LISTEN_CLIENT_URLS: http://0.0.0.0:2379
      ETCD_ADVERTISE_CLIENT_URLS: http://etcd1:2379
      ETCD_LISTEN_PEER_URLS: http://0.0.0.0:2380
      ETCD_INITIAL_ADVERTISE_PEER_URLS: http://etcd1:2380
      ETCD_INITIAL_CLUSTER: etcd1=http://etcd1:2380,etcd2=http://etcd2:2380,etcd3=http://etcd3:2380
      ETCD_INITIAL_CLUSTER_STATE: new
      ETCD_INITIAL_CLUSTER_TOKEN: etcd-cluster-1
    volumes:
    - etcd1_data:/etcd-data
    healthcheck:
      test: ["CMD", "etcdctl", "endpoint", "health"]
      interval: 10s
      timeout: 5s
      retries: 5

  etcd2:
    image: quay.io/coreos/etcd:v3.5.9
    hostname: etcd2
    networks:
      patroni:
        ipv4_address: 172.20.0.11
    environment:
      ETCD_NAME: etcd2
      ETCD_DATA_DIR: /etcd-data
      ETCD_LISTEN_CLIENT_URLS: http://0.0.0.0:2379
      ETCD_ADVERTISE_CLIENT_URLS: http://etcd2:2379
      ETCD_LISTEN_PEER_URLS: http://0.0.0.0:2380
      ETCD_INITIAL_ADVERTISE_PEER_URLS: http://etcd2:2380
      ETCD_INITIAL_CLUSTER: etcd1=http://etcd1:2380,etcd2=http://etcd2:2380,etcd3=http://etcd3:2380
      ETCD_INITIAL_CLUSTER_STATE: new
      ETCD_INITIAL_CLUSTER_TOKEN: etcd-cluster-1
    volumes:
    - etcd2_data:/etcd-data

  etcd3:
    image: quay.io/coreos/etcd:v3.5.9
    hostname: etcd3
    networks:
      patroni:
        ipv4_address: 172.20.0.12
    environment:
      ETCD_NAME: etcd3
      ETCD_DATA_DIR: /etcd-data
      ETCD_LISTEN_CLIENT_URLS: http://0.0.0.0:2379
      ETCD_ADVERTISE_CLIENT_URLS: http://etcd3:2379
      ETCD_LISTEN_PEER_URLS: http://0.0.0.0:2380
      ETCD_INITIAL_ADVERTISE_PEER_URLS: http://etcd3:2380
      ETCD_INITIAL_CLUSTER: etcd1=http://etcd1:2380,etcd2=http://etcd2:2380,etcd3=http://etcd3:2380
      ETCD_INITIAL_CLUSTER_STATE: new
      ETCD_INITIAL_CLUSTER_TOKEN: etcd-cluster-1
    volumes:
    - etcd3_data:/etcd-data

  # ─── Patroni nodes ──────────────────────────────────────────
  patroni1:
    image: patroni:latest
    build:
      context: ./docker
      dockerfile: Dockerfile.patroni
    hostname: patroni1
    networks:
      patroni:
        ipv4_address: 172.20.0.20
    ports:
    - "5432:5432"
    - "8008:8008"
    environment:
      PATRONI_NAME: postgresql1
      PATRONI_POSTGRESQL_CONNECT_ADDRESS: patroni1:5432
      PATRONI_RESTAPI_CONNECT_ADDRESS: patroni1:8008
    volumes:
    - patroni1_data:/data
    - ./patroni1/patroni.yml:/etc/patroni/patroni.yml
    - ./scripts:/scripts
    depends_on:
      etcd1:
        condition: service_healthy
      etcd2:
        condition: service_healthy
      etcd3:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "patronictl", "-c", "/etc/patroni/patroni.yml", "list"]
      interval: 10s
      timeout: 5s
      retries: 5

  patroni2:
    image: patroni:latest
    build:
      context: ./docker
      dockerfile: Dockerfile.patroni
    hostname: patroni2
    networks:
      patroni:
        ipv4_address: 172.20.0.21
    ports:
    - "5433:5432"
    - "8009:8008"
    environment:
      PATRONI_NAME: postgresql2
      PATRONI_POSTGRESQL_CONNECT_ADDRESS: patroni2:5432
      PATRONI_RESTAPI_CONNECT_ADDRESS: patroni2:8008
    volumes:
    - patroni2_data:/data
    - ./patroni2/patroni.yml:/etc/patroni/patroni.yml
    - ./scripts:/scripts
    depends_on:
      patroni1:
        condition: service_healthy

  patroni3:
    image: patroni:latest
    build:
      context: ./docker
      dockerfile: Dockerfile.patroni
    hostname: patroni3
    networks:
      patroni:
        ipv4_address: 172.20.0.22
    ports:
    - "5434:5432"
    - "8010:8008"
    environment:
      PATRONI_NAME: postgresql3
      PATRONI_POSTGRESQL_CONNECT_ADDRESS: patroni3:5432
      PATRONI_RESTAPI_CONNECT_ADDRESS: patroni3:8008
    volumes:
    - patroni3_data:/data
    - ./patroni3/patroni.yml:/etc/patroni/patroni.yml
    - ./scripts:/scripts
    depends_on:
      patroni1:
        condition: service_healthy

  # ─── HAProxy ─────────────────────────────────────────────────
  haproxy:
    image: haproxy:2.8
    hostname: haproxy
    networks:
      patroni:
        ipv4_address: 172.20.0.30
    ports:
    - "5000:5000"    # primary (write)
    - "5001:5001"    # replicas (read)
    - "7000:7000"    # stats
    volumes:
    - ./haproxy/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro
    depends_on:
    - patroni1
    - patroni2
    - patroni3
    healthcheck:
      test: ["CMD", "haproxy", "-c", "-f", "/usr/local/etc/haproxy/haproxy.cfg"]
      interval: 10s
      timeout: 5s
      retries: 3

  # ─── pgBouncer ────────────────────────────────────────────────
  pgbouncer:
    image: pgbouncer/pgbouncer:latest
    hostname: pgbouncer
    networks:
      patroni:
        ipv4_address: 172.20.0.40
    ports:
    - "6432:6432"
    environment:
      DATABASES_HOST: haproxy
      DATABASES_PORT: 5000
      DATABASES_DBNAME: appdb
      DATABASES_USER: appuser
      DATABASES_PASSWORD: app-password
      POOL_MODE: transaction
      MAX_CLIENT_CONN: 1000
      DEFAULT_POOL_SIZE: 50
      ADMIN_USERS: postgres
    depends_on:
    - haproxy
```

### 2.6 Dockerfile สำหรับ Patroni

```dockerfile
# docker/Dockerfile.patroni
FROM ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive

# ติดตั้ง dependencies
RUN apt-get update && apt-get install -y \
    postgresql-15 \
    python3 \
    python3-pip \
    python3-psycopg2 \
    python3-yaml \
    python3-requests \
    curl \
    wget \
    && rm -rf /var/lib/apt/lists/*

# ติดตั้ง Patroni
RUN pip3 install patroni[etcd3]

# สร้าง data directory
RUN mkdir -p /data/patroni && chown postgres:postgres /data/patroni
RUN mkdir -p /scripts && chmod 755 /scripts

# สร้าง script directory
COPY scripts/ /scripts/
RUN chmod +x /scripts/*.sh

USER postgres
EXPOSE 5432 8008

CMD ["/usr/local/bin/patroni", "/etc/patroni/patroni.yml"]
```

---

## 3. Patroni REST API

### 3.1 API Endpoints

```bash
# ดู cluster info
curl http://patroni1:8008/
# Output:
# {
#   "state": "running",
#   "postmaster_start_time": "2024-01-01 00:00:00",
#   "role": "master",
#   "server_version": 150000,
#   "xlog": { "location": 12345678 },
#   "timeline": 1,
#   "replication": [
#     { "application_name": "postgresql2", "state": "streaming" }
#   ]
# }

# ดู cluster status
curl http://patroni1:8008/cluster
# Output: JSON ของทุก member ใน cluster

# ดู config
curl http://patroni1:8008/config

# ดู history
curl http://patroni1:8008/history

# Health check endpoints
curl http://patroni1:8008/health          # ทุก node healthy
curl http://patroni1:8008/master          # 200 ถ้าเป็น master
curl http://patroni1:8008/primary         # เหมือน /master
curl http://patroni1:8008/replica         # 200 ถ้าเป็น replica
curl http://patroni1:8008/standby-leader  # 200 ถ้าเป็น standby leader
curl http://patroni1:8008/read-only       # 200 ถ้าเป็น read-only node
```

### 3.2 API Operations (POST)

```bash
# Force failover ไปยัง specific node
curl -s -XPOST http://patroni1:8008/failover \
  -H "Content-Type: application/json" \
  -d '{"leader": "postgresql1", "candidate": "postgresql2"}'

# Switchover (planned maintenance)
curl -s -XPOST http://patroni1:8008/switchover \
  -H "Content-Type: application/json" \
  -d '{"leader": "postgresql1"}'

# Reload configuration
curl -s -XPOST http://patroni1:8008/reload

# Restart PostgreSQL
curl -s -XPOST http://patroni1:8008/restart \
  -H "Content-Type: application/json" \
  -d '{"schedule": "2024-12-31T02:00:00"}'

# Update DCS configuration
curl -s -XPATCH http://patroni1:8008/config \
  -H "Content-Type: application/json" \
  -d '{"loop_wait": 5, "postgresql": {"parameters": {"max_connections": 400}}}'
```

---

## 4. patronictl: CLI Tool

### 4.1 patronictl list

```bash
# ดู cluster status
patronictl -c /etc/patroni/patroni.yml list

# Output:
# + Cluster: postgres-cluster (1234567890) --+----+-----------+
# | Member      | Host       | Role    | State   | TL | Lag in MB |
# +─────────────+────────────+─────────+---------+----+-----------+
# | postgresql1 | patroni1:5432 | Leader  | running | 1  |           |
# | postgresql2 | patroni2:5432 | Replica | running | 1  |        0 |
# | postgresql3 | patroni3:5432 | Replica | running | 1  |        0 |
# +─────────────+────────────+─────────+---------+----+-----------+

# ดูแบบ pretty
patronictl -c /etc/patroni/patroni.yml list --format pretty

# ดูแบบ JSON
patronictl -c /etc/patroni/patroni.yml list --format json
```

### 4.2 patronictl failover

```bash
# Failover แบบ force (emergency)
patronictl -c /etc/patroni/patroni.yml failover postgres-cluster

# Failover ไปยัง specific candidate
patronictl -c /etc/patroni/patroni.yml failover postgres-cluster \
  --master postgresql1 \
  --candidate postgresql2 \
  --force

# Failover ทันที (ไม่ถามยืนยัน)
patronictl -c /etc/patroni/patroni.yml failover postgres-cluster --force
```

### 4.3 patronictl switchover

```bash
# Switchover แบบ planned (graceful)
patronictl -c /etc/patroni/patroni.yml switchover postgres-cluster

# Switchover พร้อม schedule
patronictl -c /etc/patroni/patroni.yml switchover postgres-cluster \
  --scheduled "2024-12-31T02:00:00"

# Switchover ไปยัง specific node
patronictl -c /etc/patroni/patroni.yml switchover postgres-cluster \
  --master postgresql1 \
  --candidate postgresql2

# Cancel scheduled switchover
patronictl -c /etc/patroni/patroni.yml switchover postgres-cluster --scheduled ""
```

### 4.4 patronictl edit-config

```bash
# Edit DCS config (เปิด editor)
patronictl -c /etc/patroni/patroni.yml edit-config postgres-cluster

# Replace config จาก file
patronictl -c /etc/patroni/patroni.yml edit-config postgres-cluster \
  --replace /path/to/config.yml

# Patch specific values
patronictl -c /etc/patroni/patroni.yml edit-config postgres-cluster \
  --set max_connections=400 \
  --set shared_buffers=1GB

# ดู current config
patronictl -c /etc/patroni/patroni.yml show-config postgres-cluster
```

### 4.5 patronictl reload

```bash
# Reload config บน node ที่ต้องการ
patronictl -c /etc/patroni/patroni.yml reload postgres-cluster postgresql1

# Reload ทุก nodes
patronictl -c /etc/patroni/patroni.yml reload postgres-cluster
```

### 4.6 patronictl restart

```bash
# Restart specific node
patronictl -c /etc/patroni/patroni.yml restart postgres-cluster postgresql2

# Restart ทุก nodes (ระวัง!)
patronictl -c /etc/patroni/patroni.yml restart postgres-cluster

# Restart พร้อม schedule
patronictl -c /etc/patroni/patroni.yml restart postgres-cluster postgresql2 \
  --scheduled "2024-12-31T02:00:00"
```

### 4.7 patronictl reinit

```bash
# Reinitialize node จาก primary (ใช้เมื่อ replica corrupted)
patronictl -c /etc/patroni/patroni.yml reinit postgres-cluster postgresql2

# Force reinit
patronictl -c /etc/patroni/patroni.yml reinit postgres-cluster postgresql2 --force
```

### 4.8 patronictl pause/resume

```bash
# Pause automatic failover (สำหรับ maintenance)
patronictl -c /etc/patroni/patroni.yml pause postgres-cluster
# Patroni จะ freeze cluster state

# Resume
patronictl -c /etc/patroni/patroni.yml resume postgres-cluster
```

---

## 5. HAProxy + Patroni Integration

### 5.1 HAProxy Configuration แบบ Full

```
# haproxy/haproxy.cfg
global
    log /dev/log local0
    log /dev/log local1 notice
    maxconn 4000
    user haproxy
    group haproxy
    daemon

defaults
    mode tcp
    log global
    option tcplog
    option dontlognull
    option tcp-check
    timeout connect 5s
    timeout client 30m
    timeout server 30m
    timeout check 5s
    balance roundrobin

listen stats
    mode http
    bind *:7000
    stats enable
    stats uri /stats
    stats refresh 5s
    stats realm HAProxy Statistics
    stats auth admin:admin123
    stats show-legends
    stats show-node
    stats hide-version

# ─── Primary (Write) ─────────────────────────────────────────
frontend primary_frontend
    bind *:5000
    default_backend primary_backend

backend primary_backend
    option httpchk GET /primary
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server patroni1 patroni1:5432 maxconn 200 check port 8008
    server patroni2 patroni2:5432 maxconn 200 check port 8008
    server patroni3 patroni3:5432 maxconn 200 check port 8008

# ─── Replicas (Read-only) ────────────────────────────────────
frontend replicas_frontend
    bind *:5001
    default_backend replicas_backend

backend replicas_backend
    option httpchk GET /replica
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server patroni1 patroni1:5432 maxconn 200 check port 8008
    server patroni2 patroni2:5432 maxconn 200 check port 8008
    server patroni3 patroni3:5432 maxconn 200 check port 8008

# ─── Any (Read from any healthy node) ───────────────────────
frontend any_frontend
    bind *:5002
    default_backend any_backend

backend any_backend
    option httpchk GET /health
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server patroni1 patroni1:5432 maxconn 300 check port 8008
    server patroni2 patroni2:5432 maxconn 300 check port 8008
    server patroni3 patroni3:5432 maxconn 300 check port 8008
```

---

## 6. pgBouncer + Patroni

### 6.1 pgBouncer Configuration

```ini
# pgbouncer.ini
[databases]
appdb = host=haproxy port=5000 dbname=appdb
appdb_readonly = host=haproxy port=5001 dbname=appdb

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt

; Transaction pooling ดีกว่า session pooling สำหรับ web apps
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 50
reserve_pool_size = 10
reserve_pool_timeout = 5

; Connection limits
max_db_connections = 100
min_pool_size = 10
max_user_connections = 50

; Timeouts
server_idle_timeout = 600
client_idle_timeout = 0
query_timeout = 0
query_wait_timeout = 120
connect_timeout = 15
client_login_timeout = 60

; Logging
logfile = /var/log/pgbouncer/pgbouncer.log
pidfile = /var/run/pgbouncer/pgbouncer.pid
log_connections = 1
log_disconnections = 1
log_pooler_errors = 1
stats_period = 60

; Admin
admin_users = postgres
stats_users = stats
```

```
# userlist.txt
"appuser" "md5<hash>"
"postgres" "md5<hash>"
"stats" "md5<hash>"
```

---

## 7. Monitoring Patroni

### 7.1 Patroni Metrics ด้วย Prometheus

```bash
# patroni มี built-in metrics endpoint
curl http://patroni1:8008/metrics

# Output (Prometheus format):
# patroni_version{...} 3.2.0
# patroni_postgres_running 1
# patroni_master 1
# patroni_xlog_location 12345678
# patroni_postgres_server_version 150000
# patroni_cluster_unlocked 0
# patroni_failsafe_mode_is_active 0
# patroni_replication_lag_apply_seconds{...} 0
```

### 7.2 Prometheus Configuration

```yaml
# prometheus.yml
scrape_configs:
- job_name: patroni
  static_configs:
  - targets:
    - patroni1:8008
    - patroni2:8008
    - patroni3:8008
  metrics_path: /metrics
  
- job_name: postgresql
  static_configs:
  - targets:
    - patroni1:9187
    - patroni2:9187
    - patroni3:9187
```

### 7.3 Grafana Dashboard Queries

```promql
# ดู Leader node
patroni_master{job="patroni"} == 1

# Replication lag (seconds)
patroni_replication_lag_apply_seconds

# PostgreSQL running
patroni_postgres_running{job="patroni"}

# Timeline
patroni_master_timeline{job="patroni"}
```

---

## 8. Testing Failover

### 8.1 Manual Failover Test

```bash
# Step 1: ดู current leader
patronictl -c /etc/patroni/patroni.yml list postgres-cluster

# Step 2: ทดสอบ writes ผ่าน HAProxy port 5000
psql -h localhost -p 5000 -U appuser -d appdb -c "
  CREATE TABLE IF NOT EXISTS test_failover (
    id SERIAL PRIMARY KEY,
    message TEXT,
    created_at TIMESTAMP DEFAULT NOW()
  );
  INSERT INTO test_failover (message) VALUES ('before failover');
  SELECT * FROM test_failover;
"

# Step 3: Simulate failover
patronictl -c /etc/patroni/patroni.yml failover postgres-cluster --force

# Step 4: ตรวจสอบ leader ใหม่
patronictl -c /etc/patroni/patroni.yml list postgres-cluster

# Step 5: ทดสอบว่า write ยังทำงานได้
psql -h localhost -p 5000 -U appuser -d appdb -c "
  INSERT INTO test_failover (message) VALUES ('after failover');
  SELECT * FROM test_failover ORDER BY id;
"
```

### 8.2 Automatic Failover Test

```bash
# Step 1: หา container ของ leader
docker ps | grep patroni

# Step 2: Kill leader container
docker stop patroni-patroni1-1

# Step 3: Monitor ใน terminal อื่น
watch -n 1 'patronictl -c /etc/patroni/patroni.yml list postgres-cluster 2>/dev/null || echo "Cluster rebuilding..."'

# Step 4: ดู failover time (ปกติ 10-30 วินาที)
# จะเห็น postgresql2 หรือ postgresql3 กลายเป็น Leader

# Step 5: ทดสอบ connection
until psql -h localhost -p 5000 -U appuser -d appdb -c "SELECT NOW();" 2>/dev/null; do
  echo "Waiting for cluster to recover..."
  sleep 2
done
echo "Cluster recovered!"

# Step 6: Start leader เดิมกลับมา (จะกลายเป็น replica)
docker start patroni-patroni1-1

# Step 7: ดู cluster
patronictl -c /etc/patroni/patroni.yml list postgres-cluster
```

### 8.3 Script สำหรับ Failover Test

```bash
#!/bin/bash
# test-failover.sh

set -e

PATRONICTL="patronictl -c /etc/patroni/patroni.yml"
CLUSTER="postgres-cluster"
HAPROXY_PRIMARY="localhost:5000"
HAPROXY_REPLICA="localhost:5001"

echo "=== Patroni Failover Test ==="
echo ""

# แสดง cluster state เริ่มต้น
echo "--- Initial cluster state ---"
$PATRONICTL list $CLUSTER
echo ""

# หา current leader
LEADER=$($PATRONICTL list $CLUSTER --format json 2>/dev/null | \
  python3 -c "import json,sys; members=json.load(sys.stdin)['members']; \
  [print(m['name']) for m in members if m['role']=='Leader']")
echo "Current leader: $LEADER"

# Insert test data
echo "--- Inserting test data before failover ---"
psql -h localhost -p 5000 -U postgres -d postgres -c "
  CREATE TABLE IF NOT EXISTS failover_test (
    id SERIAL PRIMARY KEY,
    phase TEXT,
    ts TIMESTAMPTZ DEFAULT NOW()
  );
  INSERT INTO failover_test (phase) VALUES ('BEFORE_FAILOVER');
" 2>&1 || echo "Insert failed (expected if table exists)"

# Perform failover
echo "--- Performing failover ---"
START_TIME=$(date +%s%3N)
$PATRONICTL failover $CLUSTER --force 2>&1
END_TIME=$(date +%s%3N)
FAILOVER_TIME=$((END_TIME - START_TIME))
echo "Failover initiated in ${FAILOVER_TIME}ms"

# รอ cluster stabilize
echo "--- Waiting for cluster to stabilize ---"
WAIT=0
MAX_WAIT=60
while ! psql -h localhost -p 5000 -U postgres -d postgres -c "SELECT 1;" >/dev/null 2>&1; do
  sleep 1
  WAIT=$((WAIT + 1))
  if [ $WAIT -gt $MAX_WAIT ]; then
    echo "ERROR: Cluster did not recover within ${MAX_WAIT}s"
    exit 1
  fi
done
RECOVER_TIME=$((WAIT * 1000))
echo "Cluster recovered in ~${RECOVER_TIME}ms"

# แสดง cluster state หลัง failover
echo "--- Cluster state after failover ---"
$PATRONICTL list $CLUSTER

# ตรวจสอบ data integrity
echo "--- Verifying data after failover ---"
psql -h localhost -p 5000 -U postgres -d postgres -c "
  INSERT INTO failover_test (phase) VALUES ('AFTER_FAILOVER');
  SELECT * FROM failover_test ORDER BY ts;
"

NEW_LEADER=$($PATRONICTL list $CLUSTER --format json 2>/dev/null | \
  python3 -c "import json,sys; members=json.load(sys.stdin)['members']; \
  [print(m['name']) for m in members if m['role']=='Leader']")

echo ""
echo "=== Test Summary ==="
echo "Old leader: $LEADER"
echo "New leader: $NEW_LEADER"
echo "Failover time: ~${RECOVER_TIME}ms"
echo "Status: SUCCESS"
```

---

## 9. การ Monitor และ Alert

### 9.1 Alert Rules สำหรับ Patroni

```yaml
# patroni-alerts.yaml
groups:
- name: patroni.rules
  rules:
  - alert: PatroniClusterHasNoLeader
    expr: sum(patroni_master) == 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "Patroni cluster has no leader"
      
  - alert: PatroniClusterUnlocked
    expr: patroni_cluster_unlocked == 1
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "Patroni cluster is unlocked (no leader)"
      
  - alert: PatroniReplicaLagHigh
    expr: patroni_replication_lag_apply_seconds > 30
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Patroni replica lag is high: {{ $value }}s"
      
  - alert: PatroniPostgresDown
    expr: patroni_postgres_running == 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "PostgreSQL is not running on {{ $labels.instance }}"
```

---

## 10. Production Checklist

```bash
# 1. ตรวจสอบ etcd cluster health
etcdctl --endpoints=http://etcd1:2379,http://etcd2:2379,http://etcd3:2379 endpoint health

# 2. ตรวจสอบ Patroni cluster
patronictl -c /etc/patroni/patroni.yml list postgres-cluster

# 3. ตรวจสอบ replication lag
psql -h localhost -p 5000 -U postgres -c "
  SELECT 
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
"

# 4. ตรวจสอบ HAProxy stats
curl http://localhost:7000/stats

# 5. ตรวจสอบ connection pooling
# psql -h localhost -p 6432 -U pgbouncer pgbouncer -c "SHOW POOLS;"
```

---

## สรุป

Patroni เป็น HA solution ที่ครบครันสำหรับ PostgreSQL:

1. **DCS** (etcd/ZooKeeper/Consul) ทำหน้าที่เป็น distributed consensus
2. **Leader election** อัตโนมัติเมื่อ primary ล้ม
3. **Automatic failover** ภายใน 30-60 วินาที
4. **patronictl** CLI สำหรับจัดการ cluster
5. **HAProxy** ให้ single endpoint สำหรับ application
6. **pgBouncer** เพิ่ม connection pooling
7. **REST API** สำหรับ monitoring และ automation

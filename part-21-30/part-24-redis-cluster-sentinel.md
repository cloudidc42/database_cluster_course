# Part 24: Redis Cluster และ Sentinel

## บทนำ: ทำไม Redis ต้องการ HA

Redis เป็น In-Memory Database ที่ใช้กันอย่างแพร่หลายสำหรับ Caching, Session Store, Queue, และ Pub/Sub ปัญหาคือถ้า Redis Instance เดียวล่ม ระบบทั้งหมดที่พึ่งพา Redis จะพัง

```
Application ──────▶ Redis (Single) ──▶ Cache Miss ทุกครั้งถ้าล่ม!
                         │
                    ▼ Fail!
               ระบบช้าลง / พัง
```

**Redis มีสองแนวทาง HA:**
1. **Redis Sentinel** - High Availability สำหรับ Single Dataset
2. **Redis Cluster** - Horizontal Scaling + HA

---

## Part A: Redis Sentinel

## 1. Redis Sentinel คืออะไร

Redis Sentinel เป็นระบบ High Availability ที่:
- **Monitor**: ตรวจสอบ Master และ Replica อย่างต่อเนื่อง
- **Notify**: แจ้งเตือน Admin หรือ Application เมื่อมีปัญหา
- **Automatic Failover**: Promote Replica เป็น Master อัตโนมัติ
- **Configuration Provider**: บอก Client ว่า Master อยู่ที่ไหน

```
                ┌────────────┐
                │  Sentinel1 │
                └─────┬──────┘
                      │ Monitor
    ┌─────────┐    ┌───┴─────┐    ┌──────────┐
    │Sentinel2│────│  Master  │────│ Sentinel3│
    └────┬────┘    └────┬─────┘    └────┬─────┘
         │              │              │
         └──────────────┼──────────────┘
                        │ Replicate
               ┌────────┴────────┐
               │                 │
          ┌────┴─────┐    ┌──────┴────┐
          │ Replica1 │    │ Replica2  │
          └──────────┘    └───────────┘
```

### 1.1 Minimum 3 Sentinel Nodes

ต้องมี Sentinel อย่างน้อย 3 Node เพื่อ Quorum (เสียงข้างมาก):

```
Failover ต้องการ Quorum = floor(sentinels/2) + 1

3 Sentinels: Quorum = 2
เหตุผล: ป้องกัน Split-Brain (2 Sentinels เห็นต่างกัน)
```

---

## 2. sentinel.conf Configuration

```
# sentinel.conf

# Port ที่ Sentinel ฟัง
port 26379

# Daemonize
daemonize no

# Log File
logfile "/var/log/redis/sentinel.log"

# Data Directory
dir /var/lib/redis

# ===================================================
# MONITOR CONFIGURATION
# ===================================================

# ตรวจสอบ Master ชื่อ "mymaster" ที่ IP 172.20.0.10 port 6379
# Quorum = 2 (ต้องการ 2 Sentinels ยืนยันว่า Master ล่ม)
sentinel monitor mymaster 172.20.0.10 6379 2

# Password ของ Master (ถ้ามี)
sentinel auth-pass mymaster RedisPassword123!

# เวลา (ms) ที่ Master ไม่ตอบสนอง ก่อนถือว่า Subjectively Down
sentinel down-after-milliseconds mymaster 5000

# จำนวน Replica ที่ Sync พร้อมกันในระหว่าง Failover
# 1 = ทีละตัว (ลด Downtime ที่ Replica จะไม่ available ชั่วคราว)
sentinel parallel-syncs mymaster 1

# เวลา (ms) ที่ต้องรอก่อน Retry Failover
sentinel failover-timeout mymaster 30000

# ===================================================
# NOTIFICATION
# ===================================================

# Script ที่รันเมื่อ Event เกิดขึ้น
# sentinel notification-script mymaster /var/redis/notify.sh

# Script ที่รันหลัง Failover
# sentinel client-reconfig-script mymaster /var/redis/reconfig.sh

# ===================================================
# ADVANCED SETTINGS
# ===================================================

# ACL (Redis 6+)
# requirepass SentinelPassword

# Announce IP/Port ให้ Sentinel อื่นรู้ (สำหรับ Docker/NAT)
# sentinel announce-ip 1.2.3.4
# sentinel announce-port 26379

# Bind Address
# bind 0.0.0.0

# Sentinel ยอมรับ Connection จากทุกที่
# protected-mode no
```

---

## 3. redis.conf สำหรับ Master และ Replica

### 3.1 Master Configuration

```
# master/redis.conf

# Basic
port 6379
bind 0.0.0.0
protected-mode no

# Password
requirepass RedisPassword123!

# Replication
# Master ไม่ต้องตั้ง replicaof

# Persistence
save 900 1
save 300 10
save 60 10000
appendonly yes
appendfsync everysec

# Memory
maxmemory 512mb
maxmemory-policy allkeys-lru

# Logging
loglevel notice
logfile /var/log/redis/redis.log
```

### 3.2 Replica Configuration

```
# replica/redis.conf

port 6379
bind 0.0.0.0
protected-mode no

# Password
requirepass RedisPassword123!
masterauth RedisPassword123!

# ชี้ไปยัง Master
replicaof 172.20.0.10 6379

# Replica Read-Only (แนะนำ)
replica-read-only yes

# Serve Stale Data ขณะ Sync กับ Master
replica-serve-stale-data yes

# Logging
loglevel notice
logfile /var/log/redis/redis.log
```

---

## 4. Docker Compose: 1 Master + 2 Replicas + 3 Sentinels

```yaml
# docker-compose-sentinel.yml
version: '3.8'

networks:
  redis-sentinel-network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/24

services:
  # ==========================================
  # REDIS MASTER
  # ==========================================
  redis-master:
    image: redis:7-alpine
    container_name: redis-master
    hostname: redis-master
    command: >
      redis-server
      --port 6379
      --requirepass RedisPassword123!
      --appendonly yes
      --appendfsync everysec
      --maxmemory 512mb
      --maxmemory-policy allkeys-lru
      --loglevel notice
    networks:
      redis-sentinel-network:
        ipv4_address: 172.20.0.10
    ports:
      - "6379:6379"
    volumes:
      - redis-master-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "RedisPassword123!", "ping"]
      interval: 5s
      timeout: 3s
      retries: 10

  # ==========================================
  # REDIS REPLICA 1
  # ==========================================
  redis-replica1:
    image: redis:7-alpine
    container_name: redis-replica1
    hostname: redis-replica1
    command: >
      redis-server
      --port 6379
      --requirepass RedisPassword123!
      --masterauth RedisPassword123!
      --replicaof redis-master 6379
      --replica-read-only yes
      --appendonly yes
      --loglevel notice
    networks:
      redis-sentinel-network:
        ipv4_address: 172.20.0.11
    ports:
      - "6380:6379"
    volumes:
      - redis-replica1-data:/data
    depends_on:
      redis-master:
        condition: service_healthy

  # ==========================================
  # REDIS REPLICA 2
  # ==========================================
  redis-replica2:
    image: redis:7-alpine
    container_name: redis-replica2
    hostname: redis-replica2
    command: >
      redis-server
      --port 6379
      --requirepass RedisPassword123!
      --masterauth RedisPassword123!
      --replicaof redis-master 6379
      --replica-read-only yes
      --appendonly yes
      --loglevel notice
    networks:
      redis-sentinel-network:
        ipv4_address: 172.20.0.12
    ports:
      - "6381:6379"
    volumes:
      - redis-replica2-data:/data
    depends_on:
      redis-master:
        condition: service_healthy

  # ==========================================
  # SENTINEL 1
  # ==========================================
  sentinel1:
    image: redis:7-alpine
    container_name: sentinel1
    hostname: sentinel1
    command: >
      sh -c "
        cat > /tmp/sentinel.conf << EOF
        port 26379
        sentinel monitor mymaster 172.20.0.10 6379 2
        sentinel auth-pass mymaster RedisPassword123!
        sentinel down-after-milliseconds mymaster 5000
        sentinel failover-timeout mymaster 30000
        sentinel parallel-syncs mymaster 1
        sentinel announce-ip 172.20.0.20
        sentinel announce-port 26379
        EOF
        redis-sentinel /tmp/sentinel.conf
      "
    networks:
      redis-sentinel-network:
        ipv4_address: 172.20.0.20
    ports:
      - "26379:26379"
    depends_on:
      - redis-master
      - redis-replica1
      - redis-replica2

  # ==========================================
  # SENTINEL 2
  # ==========================================
  sentinel2:
    image: redis:7-alpine
    container_name: sentinel2
    hostname: sentinel2
    command: >
      sh -c "
        cat > /tmp/sentinel.conf << EOF
        port 26379
        sentinel monitor mymaster 172.20.0.10 6379 2
        sentinel auth-pass mymaster RedisPassword123!
        sentinel down-after-milliseconds mymaster 5000
        sentinel failover-timeout mymaster 30000
        sentinel parallel-syncs mymaster 1
        sentinel announce-ip 172.20.0.21
        sentinel announce-port 26379
        EOF
        redis-sentinel /tmp/sentinel.conf
      "
    networks:
      redis-sentinel-network:
        ipv4_address: 172.20.0.21
    ports:
      - "26380:26379"
    depends_on:
      - redis-master

  # ==========================================
  # SENTINEL 3
  # ==========================================
  sentinel3:
    image: redis:7-alpine
    container_name: sentinel3
    hostname: sentinel3
    command: >
      sh -c "
        cat > /tmp/sentinel.conf << EOF
        port 26379
        sentinel monitor mymaster 172.20.0.10 6379 2
        sentinel auth-pass mymaster RedisPassword123!
        sentinel down-after-milliseconds mymaster 5000
        sentinel failover-timeout mymaster 30000
        sentinel parallel-syncs mymaster 1
        sentinel announce-ip 172.20.0.22
        sentinel announce-port 26379
        EOF
        redis-sentinel /tmp/sentinel.conf
      "
    networks:
      redis-sentinel-network:
        ipv4_address: 172.20.0.22
    ports:
      - "26381:26379"
    depends_on:
      - redis-master

volumes:
  redis-master-data:
  redis-replica1-data:
  redis-replica2-data:
```

---

## 5. เชื่อมต่อ Node.js กับ Sentinel: ioredis

```typescript
// src/redis/sentinel-client.ts
import Redis from 'ioredis';

// ===================================================
// Sentinel Connection
// ===================================================
const redisSentinel = new Redis({
  sentinels: [
    { host: 'localhost', port: 26379 },
    { host: 'localhost', port: 26380 },
    { host: 'localhost', port: 26381 },
  ],
  name: 'mymaster',           // ชื่อ Master ที่ตั้งใน sentinel.conf
  password: 'RedisPassword123!',
  sentinelPassword: undefined, // ถ้า Sentinel ต้องการ Password
  
  // Connection Options
  connectTimeout: 10000,
  commandTimeout: 5000,
  maxRetriesPerRequest: 3,
  retryStrategy: (times: number) => {
    if (times > 10) return null;  // หยุด Retry หลัง 10 ครั้ง
    return Math.min(times * 200, 2000);
  },

  // Failover: ioredis จะ Reconnect อัตโนมัติ
  enableOfflineQueue: true,

  // Lazy Connect
  lazyConnect: true,
});

// Events
redisSentinel.on('connect', () => {
  console.log('Connected to Redis (via Sentinel)');
});

redisSentinel.on('ready', () => {
  console.log('Redis ready');
});

redisSentinel.on('error', (err) => {
  console.error('Redis error:', err);
});

redisSentinel.on('close', () => {
  console.log('Redis connection closed');
});

redisSentinel.on('+failover-end', () => {
  console.log('Failover completed successfully');
});

redisSentinel.on('+switch-master', (masterName, oldHost, oldPort, newHost, newPort) => {
  console.log(`Master switched: ${oldHost}:${oldPort} → ${newHost}:${newPort}`);
});

// ===================================================
// ตัวอย่าง Operations
// ===================================================
async function sentinelExample() {
  await redisSentinel.connect();

  // Basic Operations (ioredis จัดการ Failover อัตโนมัติ)
  await redisSentinel.set('user:1:name', 'Alice', 'EX', 3600);
  const name = await redisSentinel.get('user:1:name');
  console.log('Name:', name);

  // Hash
  await redisSentinel.hset('user:1', {
    name: 'Alice',
    email: 'alice@example.com',
    age: '30',
  });
  const user = await redisSentinel.hgetall('user:1');
  console.log('User:', user);

  // List
  await redisSentinel.lpush('notifications:1', 'New message', 'New follower');
  const notifications = await redisSentinel.lrange('notifications:1', 0, -1);
  console.log('Notifications:', notifications);
}

export default redisSentinel;
```

---

## Part B: Redis Cluster

## 6. Redis Cluster คืออะไร

Redis Cluster เป็น Distributed Redis ที่รองรับ:
- **Horizontal Scaling**: กระจาย Data ไปหลาย Node
- **High Availability**: มี Replica สำหรับทุก Master
- **Automatic Sharding**: ไม่ต้อง Shard เอง

```
Redis Cluster: 3 Masters + 3 Replicas

Master1 ────── Replica1    (Slots 0-5460)
Master2 ────── Replica2    (Slots 5461-10922)
Master3 ────── Replica3    (Slots 10923-16383)
```

---

## 7. Hash Slots (0-16383)

Redis Cluster แบ่งข้อมูลด้วย Hash Slot:
- มีทั้งหมด **16384 Slots** (0 ถึง 16383)
- แต่ละ Master รับผิดชอบ Slot บางส่วน
- Key ถูกกำหนด Slot ด้วย: `CRC16(key) % 16384`

```
Key → CRC16(key) → % 16384 → Slot → Master Node

ตัวอย่าง:
"user:1" → CRC16 → 9189 → Slot 9189 → Master2
"user:2" → CRC16 → 5498 → Slot 5498 → Master2
"order:1" → CRC16 → 1432 → Slot 1432 → Master1
```

**Hash Tags สำหรับ Multi-Key Operations:**
```
{user}.name   → ใช้ "user" ในการคำนวณ Hash
{user}.email  → ใช้ "user" เหมือนกัน
→ ทั้งสอง Key อยู่ใน Slot เดียวกัน!

ตัวอย่าง:
MSET {user:1}.name "Alice" {user:1}.email "alice@example.com"
→ ทำงานได้เพราะอยู่ Slot เดียวกัน
```

---

## 8. CLUSTER Commands

```bash
# เชื่อมต่อ Cluster
redis-cli -h 127.0.0.1 -p 7001 -a password -c  # -c = cluster mode

# ==========================================
# CLUSTER INFO
# ==========================================
CLUSTER INFO
# cluster_enabled:1
# cluster_state:ok
# cluster_slots_assigned:16384
# cluster_slots_ok:16384
# cluster_known_nodes:6
# cluster_size:3

# ==========================================
# CLUSTER NODES
# ==========================================
CLUSTER NODES
# <id> <ip:port> <flags> <master> <ping-sent> <pong-recv> <config-epoch> <link-state> <slot ranges>
# abc123 172.20.0.10:7001@17001 master - 0 1234567890 1 connected 0-5460
# def456 172.20.0.11:7002@17002 master - 0 1234567890 2 connected 5461-10922
# ghi789 172.20.0.12:7003@17003 master - 0 1234567890 3 connected 10923-16383
# jkl012 172.20.0.13:7004@17004 slave abc123 0 1234567890 1 connected
# mno345 172.20.0.14:7005@17005 slave def456 0 1234567890 2 connected
# pqr678 172.20.0.15:7006@17006 slave ghi789 0 1234567890 3 connected

# ==========================================
# CLUSTER SLOTS
# ==========================================
CLUSTER SLOTS
# 1) 1) (integer) 0       (Start Slot)
#    2) (integer) 5460    (End Slot)
#    3) 1) "172.20.0.10"
#       2) (integer) 7001
#       3) "abc123..."    (Master Node ID)
#    4) 1) "172.20.0.13"
#       2) (integer) 7004
#       3) "jkl012..."    (Replica Node ID)

# ==========================================
# CLUSTER KEYSLOT
# ==========================================
CLUSTER KEYSLOT user:1
# (integer) 9189

CLUSTER KEYSLOT {user:1}.name
# (integer) 5474

# ==========================================
# CLUSTER SHARDS (Redis 7+)
# ==========================================
CLUSTER SHARDS

# ==========================================
# CLUSTER STATS
# ==========================================
CLUSTER MYID   # ID ของ Node ปัจจุบัน
CLUSTER RESET  # Reset Cluster State

# ตรวจสอบ Cluster Health
redis-cli --cluster check 172.20.0.10:7001 -a password
```

---

## 9. สร้าง Redis Cluster

### 9.1 วิธีที่ 1: redis-cli --cluster create

```bash
# สร้าง Cluster จาก 6 Nodes
# --cluster-replicas 1 = 1 Replica ต่อ Master
redis-cli --cluster create \
  172.20.0.10:7001 \
  172.20.0.11:7002 \
  172.20.0.12:7003 \
  172.20.0.13:7004 \
  172.20.0.14:7005 \
  172.20.0.15:7006 \
  --cluster-replicas 1 \
  -a RedisPassword123!

# ยืนยัน: พิมพ์ 'yes'
```

### 9.2 redis.conf สำหรับ Cluster Node

```
# cluster-node.conf

port 7001
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
cluster-announce-ip 172.20.0.10
cluster-announce-port 7001
cluster-announce-bus-port 17001

requirepass RedisPassword123!
masterauth RedisPassword123!

appendonly yes
appendfsync everysec

maxmemory 256mb
maxmemory-policy allkeys-lru

loglevel notice
logfile /var/log/redis/cluster-7001.log
```

---

## 10. Docker Compose: Redis Cluster

```yaml
# docker-compose-cluster.yml
version: '3.8'

networks:
  redis-cluster-network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/24

x-redis-cluster-node: &redis-cluster-defaults
  image: redis:7-alpine
  restart: unless-stopped

services:
  # ==========================================
  # CLUSTER NODE 1 (Master 1)
  # ==========================================
  redis-node-1:
    <<: *redis-cluster-defaults
    container_name: redis-node-1
    command: >
      redis-server
      --port 7001
      --cluster-enabled yes
      --cluster-config-file nodes.conf
      --cluster-node-timeout 5000
      --cluster-announce-ip 172.20.0.10
      --cluster-announce-port 7001
      --cluster-announce-bus-port 17001
      --requirepass RedisPassword123!
      --masterauth RedisPassword123!
      --appendonly yes
      --maxmemory 256mb
      --maxmemory-policy allkeys-lru
    networks:
      redis-cluster-network:
        ipv4_address: 172.20.0.10
    ports:
      - "7001:7001"
      - "17001:17001"
    volumes:
      - redis-node-1-data:/data

  redis-node-2:
    <<: *redis-cluster-defaults
    container_name: redis-node-2
    command: >
      redis-server
      --port 7002
      --cluster-enabled yes
      --cluster-config-file nodes.conf
      --cluster-node-timeout 5000
      --cluster-announce-ip 172.20.0.11
      --cluster-announce-port 7002
      --cluster-announce-bus-port 17002
      --requirepass RedisPassword123!
      --masterauth RedisPassword123!
      --appendonly yes
      --maxmemory 256mb
    networks:
      redis-cluster-network:
        ipv4_address: 172.20.0.11
    ports:
      - "7002:7002"
      - "17002:17002"
    volumes:
      - redis-node-2-data:/data

  redis-node-3:
    <<: *redis-cluster-defaults
    container_name: redis-node-3
    command: >
      redis-server
      --port 7003
      --cluster-enabled yes
      --cluster-config-file nodes.conf
      --cluster-node-timeout 5000
      --cluster-announce-ip 172.20.0.12
      --cluster-announce-port 7003
      --cluster-announce-bus-port 17003
      --requirepass RedisPassword123!
      --masterauth RedisPassword123!
      --appendonly yes
      --maxmemory 256mb
    networks:
      redis-cluster-network:
        ipv4_address: 172.20.0.12
    ports:
      - "7003:7003"
      - "17003:17003"
    volumes:
      - redis-node-3-data:/data

  redis-node-4:
    <<: *redis-cluster-defaults
    container_name: redis-node-4
    command: >
      redis-server
      --port 7004
      --cluster-enabled yes
      --cluster-config-file nodes.conf
      --cluster-node-timeout 5000
      --cluster-announce-ip 172.20.0.13
      --cluster-announce-port 7004
      --cluster-announce-bus-port 17004
      --requirepass RedisPassword123!
      --masterauth RedisPassword123!
      --appendonly yes
      --maxmemory 256mb
    networks:
      redis-cluster-network:
        ipv4_address: 172.20.0.13
    ports:
      - "7004:7004"
      - "17004:17004"
    volumes:
      - redis-node-4-data:/data

  redis-node-5:
    <<: *redis-cluster-defaults
    container_name: redis-node-5
    command: >
      redis-server
      --port 7005
      --cluster-enabled yes
      --cluster-config-file nodes.conf
      --cluster-node-timeout 5000
      --cluster-announce-ip 172.20.0.14
      --cluster-announce-port 7005
      --cluster-announce-bus-port 17005
      --requirepass RedisPassword123!
      --masterauth RedisPassword123!
      --appendonly yes
      --maxmemory 256mb
    networks:
      redis-cluster-network:
        ipv4_address: 172.20.0.14
    ports:
      - "7005:7005"
      - "17005:17005"
    volumes:
      - redis-node-5-data:/data

  redis-node-6:
    <<: *redis-cluster-defaults
    container_name: redis-node-6
    command: >
      redis-server
      --port 7006
      --cluster-enabled yes
      --cluster-config-file nodes.conf
      --cluster-node-timeout 5000
      --cluster-announce-ip 172.20.0.15
      --cluster-announce-port 7006
      --cluster-announce-bus-port 17006
      --requirepass RedisPassword123!
      --masterauth RedisPassword123!
      --appendonly yes
      --maxmemory 256mb
    networks:
      redis-cluster-network:
        ipv4_address: 172.20.0.15
    ports:
      - "7006:7006"
      - "17006:17006"
    volumes:
      - redis-node-6-data:/data

  # ==========================================
  # CLUSTER INIT (รันครั้งเดียว)
  # ==========================================
  redis-cluster-init:
    image: redis:7-alpine
    container_name: redis-cluster-init
    command: >
      sh -c "
        echo 'Waiting for all Redis nodes...'
        sleep 10
        
        echo 'Creating Redis Cluster...'
        redis-cli --cluster create \
          172.20.0.10:7001 \
          172.20.0.11:7002 \
          172.20.0.12:7003 \
          172.20.0.13:7004 \
          172.20.0.14:7005 \
          172.20.0.15:7006 \
          --cluster-replicas 1 \
          -a RedisPassword123! \
          --cluster-yes
          
        echo 'Cluster created successfully!'
        redis-cli -h 172.20.0.10 -p 7001 -a RedisPassword123! CLUSTER INFO
      "
    networks:
      - redis-cluster-network
    depends_on:
      - redis-node-1
      - redis-node-2
      - redis-node-3
      - redis-node-4
      - redis-node-5
      - redis-node-6

volumes:
  redis-node-1-data:
  redis-node-2-data:
  redis-node-3-data:
  redis-node-4-data:
  redis-node-5-data:
  redis-node-6-data:
```

---

## 11. เชื่อมต่อ Node.js กับ Redis Cluster: ioredis

```typescript
// src/redis/cluster-client.ts
import Redis from 'ioredis';

// ===================================================
// Redis Cluster Connection
// ===================================================
const redisCluster = new Redis.Cluster(
  [
    { host: 'localhost', port: 7001 },
    { host: 'localhost', port: 7002 },
    { host: 'localhost', port: 7003 },
    { host: 'localhost', port: 7004 },
    { host: 'localhost', port: 7005 },
    { host: 'localhost', port: 7006 },
  ],
  {
    redisOptions: {
      password: 'RedisPassword123!',
      connectTimeout: 10000,
      commandTimeout: 5000,
    },
    clusterRetryStrategy: (times: number) => {
      if (times > 20) return null;
      return Math.min(100 + times * 100, 3000);
    },
    
    // อนุญาตให้ Read จาก Replica
    scaleReads: 'slave',  // 'master' | 'slave' | 'all'
    
    // Reconnect Options
    enableOfflineQueue: true,
    maxRedirections: 16,   // ตาม MOVED/ASK redirections
    retryDelayOnTryAgain: 100,
    retryDelayOnMoved: 0,
    retryDelayOnClusterDown: 300,
  }
);

redisCluster.on('connect', () => console.log('Redis Cluster connected'));
redisCluster.on('error', (err) => console.error('Redis Cluster error:', err));
redisCluster.on('+node', (node) => console.log('Node added:', node.options.host));
redisCluster.on('-node', (node) => console.log('Node removed:', node.options.host));

// ===================================================
// Basic Operations
// ===================================================
async function clusterBasicOps() {
  // Simple Key-Value
  await redisCluster.set('key1', 'value1', 'EX', 3600);
  const val = await redisCluster.get('key1');
  console.log('key1:', val);

  // Hash
  await redisCluster.hset('user:1', { name: 'Alice', age: '30' });
  const user = await redisCluster.hgetall('user:1');
  console.log('user:1:', user);

  // Check which slot/node
  const slot = await redisCluster.cluster('KEYSLOT', 'user:1');
  console.log('user:1 slot:', slot);
}

// ===================================================
// MOVED และ ASK Redirections
// ===================================================
// ioredis จัดการ MOVED และ ASK อัตโนมัติ
// แต่ถ้าต้องการ Debug:

async function redirectionExample() {
  // ioredis จะส่ง Command ไปยัง Node ที่ถูกต้องอัตโนมัติ
  // ถึงแม้ว่าจะส่งไปผิด Node ก็จะได้รับ MOVED Response
  // และ ioredis จะ Retry ที่ Node ที่ถูกต้อง

  try {
    await redisCluster.set('some-key', 'value');
  } catch (err: any) {
    if (err.message.includes('MOVED')) {
      // ไม่ควรเกิดขึ้นเพราะ ioredis จัดการอัตโนมัติ
      console.error('Unexpected MOVED error:', err);
    }
  }
}

// ===================================================
// Cross-Slot Operations
// ===================================================
async function crossSlotOperations() {
  // ❌ MSET ข้าม Slot ไม่ได้
  try {
    await redisCluster.mset('user:1', 'Alice', 'order:1', 'iPhone');
  } catch (err: any) {
    console.error('Cross-slot MSET failed:', err.message);
    // CROSSSLOT Keys in request don't hash to the same slot
  }

  // ✅ ใช้ Hash Tags
  await redisCluster.mset(
    '{user:1}.name', 'Alice',
    '{user:1}.email', 'alice@example.com'
  );

  // ✅ หรือทำ SET แยกกัน
  await Promise.all([
    redisCluster.set('user:1', 'Alice'),
    redisCluster.set('order:1', 'iPhone'),
  ]);

  // ❌ Pipeline ข้าม Slot
  const pipeline = redisCluster.pipeline();
  pipeline.set('key1', 'val1');
  pipeline.set('key2', 'val2');
  // อาจ Error ถ้า key1 และ key2 อยู่ต่าง Node

  // ✅ ใช้ Hash Tags ใน Pipeline
  const safePipeline = redisCluster.pipeline();
  safePipeline.set('{session:1}.data', '{"userId": 1}');
  safePipeline.set('{session:1}.expires', '3600');
  safePipeline.expire('{session:1}.data', 3600);
  await safePipeline.exec();
}

// ===================================================
// Cluster Rebalancing
// ===================================================
// หลัง Add/Remove Node ต้องทำ Rebalance
async function rebalanceCluster() {
  // รันจาก Command Line
  // redis-cli --cluster rebalance 172.20.0.10:7001 -a password

  // Check Cluster Health
  // redis-cli --cluster check 172.20.0.10:7001 -a password

  // Fix Cluster Issues
  // redis-cli --cluster fix 172.20.0.10:7001 -a password
}

export { redisCluster };
```

---

## 12. Redis Service Class

```typescript
// src/services/RedisService.ts
import { Cluster, Redis } from 'ioredis';
import { redisCluster } from '../redis/cluster-client';

interface CacheOptions {
  ttl?: number;      // Seconds
  nx?: boolean;      // Only Set if Not Exists
}

class RedisService {
  private redis: Cluster | Redis;

  constructor(redis: Cluster | Redis) {
    this.redis = redis;
  }

  // ==========================================
  // STRING Operations
  // ==========================================

  async set(key: string, value: unknown, options?: CacheOptions): Promise<void> {
    const serialized = typeof value === 'string' ? value : JSON.stringify(value);

    if (options?.ttl && options?.nx) {
      await this.redis.set(key, serialized, 'EX', options.ttl, 'NX');
    } else if (options?.ttl) {
      await this.redis.set(key, serialized, 'EX', options.ttl);
    } else if (options?.nx) {
      await this.redis.set(key, serialized, 'NX');
    } else {
      await this.redis.set(key, serialized);
    }
  }

  async get<T = string>(key: string): Promise<T | null> {
    const value = await this.redis.get(key);
    if (!value) return null;

    try {
      return JSON.parse(value) as T;
    } catch {
      return value as unknown as T;
    }
  }

  async del(...keys: string[]): Promise<number> {
    return this.redis.del(...keys);
  }

  async exists(...keys: string[]): Promise<number> {
    return this.redis.exists(...keys);
  }

  async expire(key: string, seconds: number): Promise<number> {
    return this.redis.expire(key, seconds);
  }

  async ttl(key: string): Promise<number> {
    return this.redis.ttl(key);
  }

  // ==========================================
  // HASH Operations
  // ==========================================

  async hset(key: string, fields: Record<string, unknown>): Promise<void> {
    const serialized: Record<string, string> = {};
    for (const [k, v] of Object.entries(fields)) {
      serialized[k] = typeof v === 'string' ? v : JSON.stringify(v);
    }
    await this.redis.hset(key, serialized);
  }

  async hget<T = string>(key: string, field: string): Promise<T | null> {
    const value = await this.redis.hget(key, field);
    if (!value) return null;
    try {
      return JSON.parse(value) as T;
    } catch {
      return value as unknown as T;
    }
  }

  async hgetall<T = Record<string, string>>(key: string): Promise<T | null> {
    const value = await this.redis.hgetall(key);
    if (!value || Object.keys(value).length === 0) return null;
    return value as unknown as T;
  }

  // ==========================================
  // SORTED SET Operations
  // ==========================================

  async zadd(key: string, members: Array<[score: number, member: string]>): Promise<number> {
    const args: (string | number)[] = [];
    for (const [score, member] of members) {
      args.push(score, member);
    }
    return (this.redis as any).zadd(key, ...args);
  }

  async zrange(key: string, start: number, stop: number, withScores = false) {
    if (withScores) {
      return (this.redis as any).zrange(key, start, stop, 'WITHSCORES');
    }
    return this.redis.zrange(key, start, stop);
  }

  // ==========================================
  // DISTRIBUTED LOCK
  // ==========================================

  async acquireLock(
    lockKey: string,
    ttl: number,
    identifier: string
  ): Promise<boolean> {
    const result = await this.redis.set(
      `lock:${lockKey}`,
      identifier,
      'PX', ttl * 1000,
      'NX'
    );
    return result === 'OK';
  }

  async releaseLock(lockKey: string, identifier: string): Promise<boolean> {
    const script = `
      if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("del", KEYS[1])
      else
        return 0
      end
    `;
    const result = await (this.redis as any).eval(
      script, 1, `lock:${lockKey}`, identifier
    );
    return result === 1;
  }

  // ==========================================
  // CACHE HELPER
  // ==========================================

  async getOrSet<T>(
    key: string,
    fetcher: () => Promise<T>,
    ttl: number = 300
  ): Promise<T> {
    const cached = await this.get<T>(key);
    if (cached !== null) return cached;

    const fresh = await fetcher();
    await this.set(key, fresh, { ttl });
    return fresh;
  }
}

export const cacheService = new RedisService(redisCluster);
```

---

## 13. เปรียบเทียบ: Sentinel vs Cluster

### 13.1 Comparison Table

| Feature | Redis Sentinel | Redis Cluster |
|---------|---------------|---------------|
| **Use Case** | HA สำหรับ Dataset เดียว | Scaling + HA |
| **Min Nodes** | 1 Master + 1+ Sentinel | 3 Masters (6 with Replicas) |
| **Data Model** | Single Dataset | Sharded |
| **Max Data** | RAM ของ 1 Node | RAM รวมทุก Node |
| **Multi-Key Ops** | ✅ (MSET, Pipeline ทำงานได้ทุกคู่) | ⚠️ (ต้อง Hash Tags) |
| **Lua Scripts** | ✅ | ⚠️ (ต้องการ Keys ใน Slot เดียว) |
| **Pub/Sub** | ✅ | ⚠️ (แต่ละ Node แยกกัน) |
| **Failover** | อัตโนมัติ (ช้ากว่า) | อัตโนมัติ (เร็วกว่า) |
| **Complexity** | ปานกลาง | สูง |
| **Client Support** | ดี | ดี (ส่วนใหญ่) |

### 13.2 เมื่อไหร่ใช้อะไร

**ใช้ Redis Sentinel เมื่อ:**
```
- Data ทั้งหมดพอดีกับ RAM ของ 1 Server
- ใช้ Multi-Key Commands หนักๆ (MSET, MGET, Pipelines)
- ใช้ Lua Scripts ซับซ้อน
- ต้องการ Simplicity
- ต้องการ HA เพียงอย่างเดียว
```

**ใช้ Redis Cluster เมื่อ:**
```
- Data มากกว่า RAM ของ Server เดียว
- ต้องการ Horizontal Scaling สำหรับ Write Throughput
- ยินดีจัดการ Cross-slot Constraints
- Team มีประสบการณ์กับ Distributed Systems
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

**Redis Sentinel:**
1. **Sentinel Role** - Monitor, Notify, Automatic Failover
2. **Quorum** - ต้องการ 3 Sentinels ขั้นต่ำ
3. **sentinel.conf** - Configuration ทั้งหมด
4. **Docker Compose** - 1 Master + 2 Replicas + 3 Sentinels
5. **ioredis Sentinel** - เชื่อมต่อจาก Node.js

**Redis Cluster:**
1. **Hash Slots** - 16384 Slots, CRC16 Hashing
2. **Cluster Topology** - 3 Masters + 3 Replicas
3. **CLUSTER Commands** - INFO, NODES, KEYSLOT
4. **Cross-Slot Operations** - ปัญหาและ Hash Tags Solution
5. **Docker Compose** - 6 Node Cluster
6. **ioredis Cluster** - MOVED/ASK Handling

ในบทถัดไปเราจะเรียนรู้ Caching Strategies ต่างๆ เช่น Cache-Aside, Write-Through, Write-Behind

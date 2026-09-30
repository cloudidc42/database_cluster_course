# Part 54: Redis High Availability แบบ Advanced

## บทนำ

Redis มีสองแนวทางหลักสำหรับ High Availability: **Redis Sentinel** สำหรับ master-replica setup ที่มีการ failover อัตโนมัติ และ **Redis Cluster** สำหรับ horizontal sharding ที่รองรับ data sets ขนาดใหญ่ บทนี้จะลงลึกทั้งสองแนวทางพร้อมตัวอย่างการใช้งานจริง

---

## 1. Redis Sentinel: Deep Dive

### 1.1 Sentinel Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Redis Sentinel Cluster                    │
│                                                             │
│  ┌──────────┐     ┌──────────┐     ┌──────────┐           │
│  │Sentinel 1│     │Sentinel 2│     │Sentinel 3│           │
│  │Port:26379│     │Port:26379│     │Port:26379│           │
│  └────┬─────┘     └────┬─────┘     └────┬─────┘           │
│       │                │                │                  │
│       └────────────────┼────────────────┘                  │
│                        │ Monitor + Gossip                   │
│       ┌────────────────┼────────────────┐                  │
│       ▼                ▼                ▼                  │
│  ┌──────────┐     ┌──────────┐     ┌──────────┐           │
│  │ Master   │────▶│ Replica 1│     │ Replica 2│           │
│  │Port:6379 │     │Port:6379 │     │Port:6379 │           │
│  └──────────┘     └──────────┘     └──────────┘           │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 sentinel.conf แบบ Full

```
# sentinel.conf สำหรับ Sentinel 1

# ─── Network ──────────────────────────────────────────────────
port 26379
bind 0.0.0.0
protected-mode no

# ─── Daemonize ────────────────────────────────────────────────
daemonize no
pidfile /var/run/redis-sentinel.pid
logfile ""
loglevel notice

# ─── Directory ────────────────────────────────────────────────
dir /tmp

# ─── Monitor Configuration ────────────────────────────────────
# sentinel monitor <master-name> <ip> <port> <quorum>
# quorum: จำนวน Sentinel ที่ต้อง agree ว่า master down
sentinel monitor mymaster redis-master 6379 2

# ─── Authentication ───────────────────────────────────────────
# Password สำหรับ connect ไปยัง master
sentinel auth-pass mymaster YourRedisPassword123

# Password สำหรับ Sentinel เอง (Redis 5.0.1+)
sentinel sentinel-pass YourSentinelPassword123

# ─── Timeout Settings ─────────────────────────────────────────
# เวลา (ms) ที่ master ไม่ตอบสนองก่อน Sentinel จะถือว่า subjectively down
sentinel down-after-milliseconds mymaster 5000

# เวลา (ms) สำหรับ failover process ทั้งหมด
sentinel failover-timeout mymaster 60000

# จำนวน replicas ที่ sync กับ new master พร้อมกัน
# 1 = sync ทีละตัว (ทำให้ replicas ยังตอบได้ระหว่าง sync)
sentinel parallel-syncs mymaster 1

# ─── Notification Scripts ─────────────────────────────────────
# Script ที่ run เมื่อเกิด warning events
sentinel notification-script mymaster /scripts/notify.sh

# Script ที่ run หลัง failover
sentinel client-reconfig-script mymaster /scripts/reconfig.sh

# ─── Announces ────────────────────────────────────────────────
# สำหรับ Docker/NAT - บอก Sentinel ว่าใช้ IP/Port อะไรในการ announce
# sentinel announce-ip 192.168.1.100
# sentinel announce-port 26379

# ─── ACL ──────────────────────────────────────────────────────
# aclfile /etc/redis/sentinel-users.acl

# ─── Resolve Hostnames ────────────────────────────────────────
sentinel resolve-hostnames yes
sentinel announce-hostnames yes
```

### 1.3 Notification Script

```bash
#!/bin/bash
# /scripts/notify.sh
# Arguments: <event-type> <event-description>

EVENT_TYPE=$1
EVENT_DESC=$2

# Log to file
echo "$(date): EVENT=$EVENT_TYPE DESC=$EVENT_DESC" >> /var/log/redis-sentinel-events.log

# ส่ง notification ไปยัง Slack
SLACK_WEBHOOK="https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
MESSAGE="Redis Sentinel Alert: *${EVENT_TYPE}*\n${EVENT_DESC}"

curl -s -X POST $SLACK_WEBHOOK \
  -H 'Content-type: application/json' \
  -d "{\"text\": \"${MESSAGE}\"}"

# ส่ง email
# echo "Redis Sentinel Alert: ${EVENT_DESC}" | \
#   mail -s "Redis Alert: ${EVENT_TYPE}" ops@example.com

exit 0
```

### 1.4 Client Reconfig Script

```bash
#!/bin/bash
# /scripts/reconfig.sh
# Arguments: <master-name> <role> <state> <from-ip> <from-port> <to-ip> <to-port>

MASTER_NAME=$1
ROLE=$2
STATE=$3
FROM_IP=$4
FROM_PORT=$5
TO_IP=$6
TO_PORT=$7

echo "$(date): Failover - Master: $MASTER_NAME, Role: $ROLE"
echo "  From: $FROM_IP:$FROM_PORT"
echo "  To: $TO_IP:$TO_PORT"

# อัปเดต config file หรือ service discovery
# ตัวอย่าง: อัปเดต consul
# consul kv put redis/master/host $TO_IP
# consul kv put redis/master/port $TO_PORT

# อัปเดต HAProxy config
sed -i "s/server redis-master.*/server redis-master $TO_IP:$TO_PORT/" /etc/haproxy/haproxy.cfg
systemctl reload haproxy

exit 0
```

### 1.5 Quorum Explanation

```
Quorum = จำนวน Sentinels ที่ต้อง agree ว่า master เป็น "subjectively down"
ก่อนที่จะเริ่ม failover

Example: 3 Sentinels, quorum = 2

Scenario 1 - Network partition:
  Sentinel 1 → ไม่เห็น Master (network issue)
  Sentinel 2 → เห็น Master OK
  Sentinel 3 → เห็น Master OK
  
  เฉพาะ Sentinel 1 คิดว่า Master down → ยังไม่ครบ quorum (1 < 2)
  → ไม่เกิด failover ✓

Scenario 2 - Real failure:
  Sentinel 1 → ไม่เห็น Master
  Sentinel 2 → ไม่เห็น Master
  Sentinel 3 → ไม่เห็น Master
  
  3 Sentinels คิดว่า Master down → ครบ quorum (3 >= 2)
  → เริ่ม failover ✓

ข้อแนะนำ:
  - Sentinels = 3, quorum = 2   (แนะนำสำหรับส่วนใหญ่)
  - Sentinels = 5, quorum = 3   (สำหรับ high availability)
```

### 1.6 Sentinel Docker Compose

```yaml
# docker-compose-sentinel.yml
version: '3.8'

networks:
  redis:
    driver: bridge

services:
  redis-master:
    image: redis:7
    hostname: redis-master
    networks:
    - redis
    ports:
    - "6379:6379"
    command: >
      redis-server
      --requirepass YourRedisPassword123
      --masterauth YourRedisPassword123
      --appendonly yes
      --appendfsync everysec
    volumes:
    - redis_master_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "YourRedisPassword123", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

  redis-replica-1:
    image: redis:7
    hostname: redis-replica-1
    networks:
    - redis
    ports:
    - "6380:6379"
    command: >
      redis-server
      --requirepass YourRedisPassword123
      --masterauth YourRedisPassword123
      --replicaof redis-master 6379
      --appendonly yes
    volumes:
    - redis_replica1_data:/data
    depends_on:
      redis-master:
        condition: service_healthy

  redis-replica-2:
    image: redis:7
    hostname: redis-replica-2
    networks:
    - redis
    ports:
    - "6381:6379"
    command: >
      redis-server
      --requirepass YourRedisPassword123
      --masterauth YourRedisPassword123
      --replicaof redis-master 6379
      --appendonly yes
    volumes:
    - redis_replica2_data:/data
    depends_on:
      redis-master:
        condition: service_healthy

  sentinel-1:
    image: redis:7
    hostname: sentinel-1
    networks:
    - redis
    ports:
    - "26379:26379"
    command: >
      sh -c "
        echo 'sentinel monitor mymaster redis-master 6379 2' > /tmp/sentinel.conf &&
        echo 'sentinel auth-pass mymaster YourRedisPassword123' >> /tmp/sentinel.conf &&
        echo 'sentinel down-after-milliseconds mymaster 5000' >> /tmp/sentinel.conf &&
        echo 'sentinel failover-timeout mymaster 60000' >> /tmp/sentinel.conf &&
        echo 'sentinel parallel-syncs mymaster 1' >> /tmp/sentinel.conf &&
        echo 'sentinel resolve-hostnames yes' >> /tmp/sentinel.conf &&
        echo 'sentinel announce-hostnames yes' >> /tmp/sentinel.conf &&
        redis-sentinel /tmp/sentinel.conf
      "
    depends_on:
    - redis-master
    - redis-replica-1
    - redis-replica-2

  sentinel-2:
    image: redis:7
    hostname: sentinel-2
    networks:
    - redis
    ports:
    - "26380:26379"
    command: >
      sh -c "
        echo 'sentinel monitor mymaster redis-master 6379 2' > /tmp/sentinel.conf &&
        echo 'sentinel auth-pass mymaster YourRedisPassword123' >> /tmp/sentinel.conf &&
        echo 'sentinel down-after-milliseconds mymaster 5000' >> /tmp/sentinel.conf &&
        echo 'sentinel failover-timeout mymaster 60000' >> /tmp/sentinel.conf &&
        echo 'sentinel parallel-syncs mymaster 1' >> /tmp/sentinel.conf &&
        echo 'sentinel resolve-hostnames yes' >> /tmp/sentinel.conf &&
        redis-sentinel /tmp/sentinel.conf
      "
    depends_on:
    - redis-master

  sentinel-3:
    image: redis:7
    hostname: sentinel-3
    networks:
    - redis
    ports:
    - "26381:26379"
    command: >
      sh -c "
        echo 'sentinel monitor mymaster redis-master 6379 2' > /tmp/sentinel.conf &&
        echo 'sentinel auth-pass mymaster YourRedisPassword123' >> /tmp/sentinel.conf &&
        echo 'sentinel down-after-milliseconds mymaster 5000' >> /tmp/sentinel.conf &&
        echo 'sentinel failover-timeout mymaster 60000' >> /tmp/sentinel.conf &&
        echo 'sentinel parallel-syncs mymaster 1' >> /tmp/sentinel.conf &&
        echo 'sentinel resolve-hostnames yes' >> /tmp/sentinel.conf &&
        redis-sentinel /tmp/sentinel.conf
      "
    depends_on:
    - redis-master

volumes:
  redis_master_data: {}
  redis_replica1_data: {}
  redis_replica2_data: {}
```

### 1.7 Application Connection ด้วย ioredis

```javascript
// app.js - Redis Sentinel ด้วย ioredis
const Redis = require('ioredis');

// ─── Sentinel Connection ────────────────────────────────────
const redis = new Redis({
  sentinels: [
    { host: 'sentinel-1', port: 26379 },
    { host: 'sentinel-2', port: 26380 },
    { host: 'sentinel-3', port: 26381 },
  ],
  name: 'mymaster',     // ชื่อ master ใน sentinel
  password: 'YourRedisPassword123',
  sentinelPassword: 'YourSentinelPassword123',
  
  // Connection options
  enableOfflineQueue: true,
  enableReadyCheck: true,
  connectTimeout: 10000,
  commandTimeout: 5000,
  
  // Retry strategy
  retryStrategy(times) {
    const delay = Math.min(times * 50, 2000);
    return delay;
  },
  
  // Error handling
  reconnectOnError(err) {
    const targetErrors = ['READONLY', 'CLUSTERDOWN'];
    return targetErrors.some(e => err.message.includes(e));
  },
});

// ─── Read-only connection (จาก replicas) ───────────────────
const redisReadOnly = new Redis({
  sentinels: [
    { host: 'sentinel-1', port: 26379 },
    { host: 'sentinel-2', port: 26380 },
    { host: 'sentinel-3', port: 26381 },
  ],
  name: 'mymaster',
  password: 'YourRedisPassword123',
  role: 'slave',  // เชื่อมต่อกับ replica เท่านั้น
});

// ─── Event handlers ────────────────────────────────────────
redis.on('connect', () => console.log('Redis connected'));
redis.on('ready', () => console.log('Redis ready'));
redis.on('error', (err) => console.error('Redis error:', err));
redis.on('close', () => console.log('Redis connection closed'));
redis.on('reconnecting', (time) => console.log(`Redis reconnecting in ${time}ms`));
redis.on('+sentinel', (o) => console.log('Sentinel added:', o));
redis.on('-sentinel', (o) => console.log('Sentinel removed:', o));

// ─── Usage ─────────────────────────────────────────────────
async function example() {
  // Write ไปยัง master
  await redis.set('user:1', JSON.stringify({ name: 'John', age: 30 }));
  await redis.expire('user:1', 3600);
  
  // Read จาก replica (read-only)
  const user = await redisReadOnly.get('user:1');
  console.log('User:', JSON.parse(user));
  
  // Pipeline
  const pipeline = redis.pipeline();
  pipeline.set('key1', 'value1');
  pipeline.set('key2', 'value2');
  pipeline.incr('counter');
  const results = await pipeline.exec();
  console.log('Pipeline results:', results);
}

example().catch(console.error);

module.exports = { redis, redisReadOnly };
```

### 1.8 Testing Sentinel

```bash
# ดู sentinel info
redis-cli -p 26379 SENTINEL masters
redis-cli -p 26379 SENTINEL replicas mymaster
redis-cli -p 26379 SENTINEL sentinels mymaster
redis-cli -p 26379 SENTINEL get-master-addr-by-name mymaster

# Test manual failover
redis-cli -p 26379 SENTINEL failover mymaster

# Monitor sentinel events
redis-cli -p 26379 SUBSCRIBE +switch-master

# ตรวจสอบ replication info
redis-cli -h redis-master -a YourRedisPassword123 INFO replication
```

---

## 2. Redis Cluster: Deep Dive

### 2.1 Hash Slots

```
Redis Cluster แบ่ง key space ออกเป็น 16,384 hash slots

Key → CRC16(key) % 16384 → slot number → node

Example:
  key "user:1"   → slot 5474  → Node A
  key "order:1"  → slot 12182 → Node B
  key "product:1"→ slot 7638  → Node C

Cluster ขนาด 3 masters:
  Node A: slots 0-5460
  Node B: slots 5461-10922
  Node C: slots 10923-16383
```

### 2.2 Redis Cluster Configuration

```
# redis-cluster.conf

# ─── Cluster Settings ─────────────────────────────────────
cluster-enabled yes
cluster-config-file /data/nodes.conf
cluster-node-timeout 5000
cluster-announce-hostname redis-node-0  # สำหรับ Docker
cluster-preferred-endpoint-type hostname  # ใช้ hostname แทน IP

# ─── Require full coverage ───────────────────────────────
# ถ้า no: cluster ยังทำงานได้แม้มี missing slots
# ถ้า yes: cluster หยุดถ้ามี slot ที่ไม่มี master
cluster-require-full-coverage yes

# ─── Replica settings ────────────────────────────────────
cluster-replica-no-failover no
cluster-allow-reads-when-down no
cluster-allow-pubsubshard-when-down yes

# ─── Node timeout settings ───────────────────────────────
cluster-link-sendbuf-limit 0
cluster-announce-ip 172.20.0.10   # สำหรับ Docker NAT
cluster-announce-port 6379
cluster-announce-bus-port 16379   # Cluster bus port = data port + 10000

# ─── Basic Settings ──────────────────────────────────────
port 6379
bind 0.0.0.0
protected-mode no
requirepass YourClusterPassword123

# ─── Persistence ─────────────────────────────────────────
appendonly yes
appendfsync everysec
appendfilename "appendonly.aof"
dir /data

# ─── Memory ──────────────────────────────────────────────
maxmemory 2gb
maxmemory-policy allkeys-lru

# ─── Replication ─────────────────────────────────────────
masterauth YourClusterPassword123
replica-serve-stale-data yes
replica-read-only yes
```

### 2.3 Redis Cluster Docker Compose (6 Nodes)

```yaml
# docker-compose-cluster.yml
version: '3.8'

networks:
  redis-cluster:
    driver: bridge
    ipam:
      config:
      - subnet: 172.25.0.0/16

volumes:
  redis_cluster_0: {}
  redis_cluster_1: {}
  redis_cluster_2: {}
  redis_cluster_3: {}
  redis_cluster_4: {}
  redis_cluster_5: {}

x-redis-cluster: &redis-cluster-defaults
  image: redis:7
  restart: unless-stopped
  networks:
  - redis-cluster

services:
  # ─── 3 Masters + 3 Replicas ─────────────────────────────
  redis-node-0:
    <<: *redis-cluster-defaults
    hostname: redis-node-0
    networks:
      redis-cluster:
        ipv4_address: 172.25.0.10
    ports:
    - "7000:6379"
    - "17000:16379"
    volumes:
    - redis_cluster_0:/data
    - ./cluster-config/redis-cluster.conf:/usr/local/etc/redis/redis.conf
    command: redis-server /usr/local/etc/redis/redis.conf
    environment:
      REDIS_NODE_ID: "0"

  redis-node-1:
    <<: *redis-cluster-defaults
    hostname: redis-node-1
    networks:
      redis-cluster:
        ipv4_address: 172.25.0.11
    ports:
    - "7001:6379"
    - "17001:16379"
    volumes:
    - redis_cluster_1:/data
    - ./cluster-config/redis-cluster.conf:/usr/local/etc/redis/redis.conf
    command: redis-server /usr/local/etc/redis/redis.conf

  redis-node-2:
    <<: *redis-cluster-defaults
    hostname: redis-node-2
    networks:
      redis-cluster:
        ipv4_address: 172.25.0.12
    ports:
    - "7002:6379"
    - "17002:16379"
    volumes:
    - redis_cluster_2:/data
    - ./cluster-config/redis-cluster.conf:/usr/local/etc/redis/redis.conf
    command: redis-server /usr/local/etc/redis/redis.conf

  redis-node-3:
    <<: *redis-cluster-defaults
    hostname: redis-node-3
    networks:
      redis-cluster:
        ipv4_address: 172.25.0.13
    ports:
    - "7003:6379"
    - "17003:16379"
    volumes:
    - redis_cluster_3:/data
    - ./cluster-config/redis-cluster.conf:/usr/local/etc/redis/redis.conf
    command: redis-server /usr/local/etc/redis/redis.conf

  redis-node-4:
    <<: *redis-cluster-defaults
    hostname: redis-node-4
    networks:
      redis-cluster:
        ipv4_address: 172.25.0.14
    ports:
    - "7004:6379"
    - "17004:16379"
    volumes:
    - redis_cluster_4:/data
    - ./cluster-config/redis-cluster.conf:/usr/local/etc/redis/redis.conf
    command: redis-server /usr/local/etc/redis/redis.conf

  redis-node-5:
    <<: *redis-cluster-defaults
    hostname: redis-node-5
    networks:
      redis-cluster:
        ipv4_address: 172.25.0.15
    ports:
    - "7005:6379"
    - "17005:16379"
    volumes:
    - redis_cluster_5:/data
    - ./cluster-config/redis-cluster.conf:/usr/local/etc/redis/redis.conf
    command: redis-server /usr/local/etc/redis/redis.conf

  # ─── Cluster Initializer ─────────────────────────────────
  cluster-init:
    image: redis:7
    networks:
    - redis-cluster
    depends_on:
    - redis-node-0
    - redis-node-1
    - redis-node-2
    - redis-node-3
    - redis-node-4
    - redis-node-5
    command:
    - sh
    - -c
    - |
      # รอให้ nodes พร้อม
      sleep 5
      
      # สร้าง cluster: 3 masters, 1 replica per master
      redis-cli \
        --cluster create \
        redis-node-0:6379 \
        redis-node-1:6379 \
        redis-node-2:6379 \
        redis-node-3:6379 \
        redis-node-4:6379 \
        redis-node-5:6379 \
        --cluster-replicas 1 \
        -a YourClusterPassword123 \
        --cluster-yes
        
      echo "Cluster created successfully!"
      redis-cli -h redis-node-0 -a YourClusterPassword123 CLUSTER INFO
    restart: "no"
```

### 2.4 สร้าง Cluster

```bash
# เริ่ม containers
docker-compose -f docker-compose-cluster.yml up -d redis-node-{0,1,2,3,4,5}

# รอให้ nodes พร้อม
sleep 10

# สร้าง cluster
redis-cli --cluster create \
  172.25.0.10:6379 \
  172.25.0.11:6379 \
  172.25.0.12:6379 \
  172.25.0.13:6379 \
  172.25.0.14:6379 \
  172.25.0.15:6379 \
  --cluster-replicas 1 \
  -a YourClusterPassword123 \
  --cluster-yes

# Output:
# >>> Performing hash slots allocation on 6 nodes...
# Master[0] -> Slots 0 - 5460
# Master[1] -> Slots 5461 - 10922
# Master[2] -> Slots 10923 - 16383
# Adding replica 172.25.0.13:6379 to 172.25.0.10:6379
# Adding replica 172.25.0.14:6379 to 172.25.0.11:6379
# Adding replica 172.25.0.15:6379 to 172.25.0.12:6379
# M: a1b2c3... 172.25.0.10:6379 slots:[0-5460] (5461 slots) master
# M: d4e5f6... 172.25.0.11:6379 slots:[5461-10922] (5462 slots) master
# M: g7h8i9... 172.25.0.12:6379 slots:[10923-16383] (5461 slots) master
# S: j1k2l3... 172.25.0.13:6379 replicates a1b2c3...
# S: m4n5o6... 172.25.0.14:6379 replicates d4e5f6...
# S: p7q8r9... 172.25.0.15:6379 replicates g7h8i9...
```

### 2.5 CLUSTER Commands

```bash
# ─── Cluster Info ────────────────────────────────────────
redis-cli -h redis-node-0 -a YourClusterPassword123 CLUSTER INFO
# Output:
# cluster_enabled:1
# cluster_state:ok
# cluster_slots_assigned:16384
# cluster_slots_ok:16384
# cluster_slots_pfail:0
# cluster_slots_fail:0
# cluster_known_nodes:6
# cluster_size:3 (masters)
# cluster_current_epoch:6

# ─── Cluster Nodes ───────────────────────────────────────
redis-cli -h redis-node-0 -a YourClusterPassword123 CLUSTER NODES
# Output: <id> <ip:port@bus-port> <flags> <master> <ping> <pong> <epoch> <link-state> <slots>

# ─── Key slot ────────────────────────────────────────────
redis-cli CLUSTER KEYSLOT "user:1"       # → 5474
redis-cli CLUSTER KEYSLOT "order:123"    # → some slot

# ─── Count keys in slot ───────────────────────────────────
redis-cli -h redis-node-0 -a YourClusterPassword123 CLUSTER COUNTKEYSINSLOT 5474

# ─── Get keys in slot ─────────────────────────────────────
redis-cli -h redis-node-0 -a YourClusterPassword123 CLUSTER GETKEYSINSLOT 5474 100

# ─── Cluster check ───────────────────────────────────────
redis-cli --cluster check redis-node-0:6379 -a YourClusterPassword123
```

### 2.6 Scale: Add Nodes

```bash
# เพิ่ม master node ใหม่
redis-cli --cluster add-node \
  NEW_NODE_IP:6379 \
  EXISTING_NODE_IP:6379 \
  -a YourClusterPassword123

# เพิ่ม replica node
redis-cli --cluster add-node \
  NEW_REPLICA_IP:6379 \
  EXISTING_NODE_IP:6379 \
  --cluster-slave \
  --cluster-master-id MASTER_NODE_ID \
  -a YourClusterPassword123
```

### 2.7 Resharding

```bash
# Reshard slots ไปยัง new node
redis-cli --cluster reshard \
  redis-node-0:6379 \
  -a YourClusterPassword123 \
  --cluster-from ALL \
  --cluster-to NEW_NODE_ID \
  --cluster-slots 1000 \
  --cluster-yes

# Rebalance cluster (กระจาย slots ใหม่)
redis-cli --cluster rebalance \
  redis-node-0:6379 \
  -a YourClusterPassword123 \
  --cluster-use-empty-masters
```

### 2.8 Remove Nodes

```bash
# Step 1: ย้าย slots ออกจาก node ก่อน (สำหรับ master)
redis-cli --cluster reshard redis-node-0:6379 \
  -a YourClusterPassword123 \
  --cluster-from NODE_TO_REMOVE_ID \
  --cluster-to ANOTHER_MASTER_ID \
  --cluster-slots 5461 \
  --cluster-yes

# Step 2: ลบ node
redis-cli --cluster del-node \
  redis-node-0:6379 \
  NODE_TO_REMOVE_ID \
  -a YourClusterPassword123
```

### 2.9 Hash Tags สำหรับ Multi-key Operations

```
ปัญหา: Redis Cluster ไม่อนุญาต multi-key commands ถ้า keys อยู่ต่าง slots

ตัวอย่างที่ผิด:
  MSET user:1 "John" user:2 "Jane"  # อาจอยู่คนละ slot
  KEYS user:*                        # ไม่ work ใน cluster

แก้ด้วย Hash Tags {}:
  Keys ที่มี {} เดียวกัน จะถูก hash โดยใช้เฉพาะส่วนใน {}
  
  MSET {user}:1 "John" {user}:2 "Jane"  # ทั้งคู่อยู่ slot เดียวกัน!
  CLUSTER KEYSLOT "{user}:1"   # → hash ของ "user"
  CLUSTER KEYSLOT "{user}:2"   # → hash ของ "user" (เหมือนกัน!)
```

```javascript
// ตัวอย่าง Hash Tags ใน Node.js
const Redis = require('ioredis');

const cluster = new Redis.Cluster([
  { host: '172.25.0.10', port: 6379 },
  { host: '172.25.0.11', port: 6379 },
  { host: '172.25.0.12', port: 6379 },
], {
  redisOptions: {
    password: 'YourClusterPassword123',
  },
  clusterRetryStrategy(times) {
    return Math.min(100 + times * 200, 2000);
  },
  enableOfflineQueue: true,
  enableReadyCheck: true,
});

// ─── Multi-key operations ด้วย Hash Tags ────────────────────
async function multiKeyExample() {
  // ใช้ {userId} เป็น hash tag เพื่อให้ keys อยู่ slot เดียวกัน
  const userId = '12345';
  
  const pipeline = cluster.pipeline();
  pipeline.set(`{user:${userId}}:profile`, JSON.stringify({ name: 'John' }));
  pipeline.set(`{user:${userId}}:settings`, JSON.stringify({ theme: 'dark' }));
  pipeline.set(`{user:${userId}}:session`, 'abc123');
  pipeline.expire(`{user:${userId}}:session`, 3600);
  await pipeline.exec();
  
  // MGET ทำงานได้เพราะ keys อยู่ slot เดียวกัน
  const [profile, settings] = await cluster.mget(
    `{user:${userId}}:profile`,
    `{user:${userId}}:settings`
  );
  
  // Atomic transaction ด้วย MULTI/EXEC
  const tx = cluster.multi();
  tx.get(`{user:${userId}}:profile`);
  tx.get(`{user:${userId}}:settings`);
  const results = await tx.exec();
  
  return results;
}

// ─── Cluster information ─────────────────────────────────────
async function clusterInfo() {
  const info = await cluster.cluster('INFO');
  console.log('Cluster Info:', info);
  
  const nodes = await cluster.cluster('NODES');
  console.log('Cluster Nodes:', nodes);
}
```

### 2.10 Redis Cluster บน Kubernetes

```yaml
# redis-cluster-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-cluster
  namespace: databases
spec:
  serviceName: redis-cluster-headless
  replicas: 6
  selector:
    matchLabels:
      app: redis-cluster
  template:
    metadata:
      labels:
        app: redis-cluster
    spec:
      initContainers:
      - name: config-init
        image: redis:7
        command:
        - sh
        - -c
        - |
          POD_INDEX=${HOSTNAME##*-}
          
          cat > /data/redis.conf << EOF
          cluster-enabled yes
          cluster-config-file /data/nodes.conf
          cluster-node-timeout 5000
          cluster-announce-hostname ${HOSTNAME}.redis-cluster-headless.databases.svc.cluster.local
          cluster-preferred-endpoint-type hostname
          cluster-require-full-coverage yes
          port 6379
          bind 0.0.0.0
          protected-mode no
          requirepass YourClusterPassword123
          masterauth YourClusterPassword123
          appendonly yes
          appendfsync everysec
          dir /data
          maxmemory 2gb
          maxmemory-policy allkeys-lru
          EOF
          
          echo "Config created for pod ${HOSTNAME}"
        volumeMounts:
        - name: data
          mountPath: /data
          
      containers:
      - name: redis
        image: redis:7
        command: ["redis-server", "/data/redis.conf"]
        ports:
        - name: redis
          containerPort: 6379
        - name: cluster-bus
          containerPort: 16379
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "4Gi"
            cpu: "1000m"
        volumeMounts:
        - name: data
          mountPath: /data
        livenessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - redis-cli -a YourClusterPassword123 ping | grep PONG
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - redis-cli -a YourClusterPassword123 ping | grep PONG
          initialDelaySeconds: 15
          periodSeconds: 5
          
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 10Gi

---
# Headless Service
apiVersion: v1
kind: Service
metadata:
  name: redis-cluster-headless
  namespace: databases
spec:
  type: ClusterIP
  clusterIP: None
  selector:
    app: redis-cluster
  ports:
  - name: redis
    port: 6379
    targetPort: 6379
  - name: cluster-bus
    port: 16379
    targetPort: 16379

---
# Job สำหรับ initialize cluster
apiVersion: batch/v1
kind: Job
metadata:
  name: redis-cluster-init
  namespace: databases
spec:
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: redis-cluster-init
        image: redis:7
        command:
        - sh
        - -c
        - |
          # รอให้ nodes พร้อม
          sleep 30
          
          NODES=""
          for i in $(seq 0 5); do
            NODES="$NODES redis-cluster-${i}.redis-cluster-headless.databases.svc.cluster.local:6379"
          done
          
          redis-cli \
            --cluster create $NODES \
            --cluster-replicas 1 \
            -a YourClusterPassword123 \
            --cluster-yes
            
          echo "Cluster initialized!"
```

---

## 3. Redis Commands Summary

### 3.1 Sentinel Commands

```bash
# ดู masters ที่ monitor
SENTINEL masters

# ดู replicas ของ master
SENTINEL replicas mymaster

# ดู sentinels อื่น
SENTINEL sentinels mymaster

# หา master address
SENTINEL get-master-addr-by-name mymaster

# Failover ด้วยตนเอง
SENTINEL failover mymaster

# Reset master state
SENTINEL reset mymaster

# ดู info
SENTINEL info mymaster
```

### 3.2 Cluster Commands

```bash
# ดู cluster state
CLUSTER INFO

# ดู nodes
CLUSTER NODES

# ดู slots
CLUSTER SLOTS
CLUSTER SHARDS  # แบบใหม่

# Key operations
CLUSTER KEYSLOT mykey
CLUSTER COUNTKEYSINSLOT 0
CLUSTER GETKEYSINSLOT 0 100

# Node operations
CLUSTER MEET ip port
CLUSTER FORGET node-id
CLUSTER REPLICATE master-id
CLUSTER FAILOVER [FORCE|TAKEOVER]
CLUSTER RESET [HARD|SOFT]

# Slot operations
CLUSTER SETSLOT slot IMPORTING node-id
CLUSTER SETSLOT slot MIGRATING node-id
CLUSTER SETSLOT slot NODE node-id
CLUSTER SETSLOT slot STABLE
```

---

## 4. Monitoring

### 4.1 Redis Exporter สำหรับ Prometheus

```yaml
# redis-exporter.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-exporter
  namespace: databases
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis-exporter
  template:
    metadata:
      labels:
        app: redis-exporter
    spec:
      containers:
      - name: redis-exporter
        image: oliver006/redis_exporter:latest
        args:
        - --redis.addr=redis-cluster-0.redis-cluster-headless:6379,redis-cluster-1.redis-cluster-headless:6379
        - --redis.password=$(REDIS_PASSWORD)
        - --check-keys=*
        - --include-system-metrics=true
        env:
        - name: REDIS_PASSWORD
          valueFrom:
            secretKeyRef:
              name: redis-secret
              key: redis-password
        ports:
        - name: metrics
          containerPort: 9121
```

### 4.2 Alert Rules

```yaml
# redis-alerts.yaml
groups:
- name: redis.rules
  rules:
  - alert: RedisClusterDown
    expr: redis_cluster_enabled == 1 AND redis_cluster_state_ok == 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "Redis cluster is down"
      
  - alert: RedisMemoryHigh
    expr: redis_memory_used_bytes / redis_memory_max_bytes > 0.85
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Redis memory usage > 85%: {{ $value | humanizePercentage }}"
      
  - alert: RedisReplicationBroken
    expr: redis_connected_slaves < 1
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "Redis has no replicas connected"
      
  - alert: RedisKeyEviction
    expr: increase(redis_evicted_keys_total[1m]) > 100
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: "Redis is evicting keys: {{ $value }} keys/min"
```

---

## สรุป

| Feature | Redis Sentinel | Redis Cluster |
|---------|---------------|---------------|
| **HA** | Automatic failover | Built-in |
| **Sharding** | ไม่รองรับ | รองรับ (16384 slots) |
| **Min nodes** | 3 sentinels + 1 master | 3 masters |
| **Multi-key ops** | รองรับทุก ops | ต้องใช้ Hash Tags |
| **Complexity** | ปานกลาง | สูง |
| **Use case** | < 1 instance RAM | > 1 instance RAM |

**แนะนำ:**
- **Redis Sentinel**: สำหรับ dataset ขนาดกลาง ที่ต้องการ HA แต่ไม่ต้องการ sharding
- **Redis Cluster**: สำหรับ dataset ขนาดใหญ่ ที่ต้องการ horizontal scaling

# Part 08: ติดตั้งและใช้งาน Redis เบื้องต้น

## สารบัญ

1. [Redis คืออะไร](#1-redis-คืออะไร)
2. [Use Cases ของ Redis](#2-use-cases-ของ-redis)
3. [ติดตั้ง Redis](#3-ติดตั้ง-redis)
4. [redis-cli: คำสั่งพื้นฐาน](#4-redis-cli-คำสั่งพื้นฐาน)
5. [Redis Configuration](#5-redis-configuration)
6. [Key Naming Conventions](#6-key-naming-conventions)
7. [TTL (Time to Live)](#7-ttl-time-to-live)
8. [Atomic Operations](#8-atomic-operations)
9. [SCAN vs KEYS](#9-scan-vs-keys)
10. [Redis Keyspace Notifications](#10-redis-keyspace-notifications)
11. [Monitoring Commands](#11-monitoring-commands)
12. [Workshop: Simple Cache Layer สำหรับ API](#12-workshop-simple-cache-layer-สำหรับ-api)

---

## 1. Redis คืออะไร

Redis (Remote Dictionary Server) คือ **in-memory data structure store** ที่ทำงานเร็วมาก สามารถใช้เป็น:
- **Database**: เก็บข้อมูลถาวร
- **Cache**: แคช responses ให้ application เร็วขึ้น  
- **Message Broker**: ส่งข้อความระหว่าง services

### ทำไม Redis ถึงเร็ว

```
1. In-memory: ข้อมูลอยู่ใน RAM (ns latency vs ms ของ disk)
2. Single-threaded: ไม่มี lock overhead (commands เข้า queue)
3. Efficient data structures: ออกแบบมาเฉพาะสำหรับ performance
4. Network I/O: ใช้ non-blocking I/O (epoll/kqueue)
5. Optional persistence: เลือกได้ว่าจะ persist ไป disk ไหม

Typical performance:
- GET/SET: 100,000+ operations/second
- Latency: < 1ms
- Throughput: 1 GB/s+
```

### Redis vs Memcached

| Feature | Redis | Memcached |
|---------|-------|-----------|
| Data structures | หลากหลาย (String, Hash, List, Set, ZSet, etc.) | String เท่านั้น |
| Persistence | มี (RDB, AOF) | ไม่มี |
| Replication | Master-Replica | ไม่มีในตัว |
| Clustering | มี | มี (แต่ต่างกัน) |
| Pub/Sub | มี | ไม่มี |
| Transactions | มี (MULTI/EXEC) | ไม่มี |
| Lua Scripting | มี | ไม่มี |
| Max value size | 512 MB | 1 MB |

---

## 2. Use Cases ของ Redis

### 2.1 Caching

```
ปัญหา: Database queries ช้า (10-100ms)
แก้ไข: Cache results ใน Redis (< 1ms)

Pattern: Cache-aside (Lazy loading)
1. ตรวจสอบ Redis ก่อน
2. ถ้ามีใน cache → return ทันที
3. ถ้าไม่มี → query database, save to cache, return
```

### 2.2 Session Storage

```
ปัญหา: Web sessions ต้องเข้าถึงเร็ว, shared ระหว่าง servers
แก้ไข: เก็บ session data ใน Redis

Key: "session:{session_id}"
Value: JSON object ของ session data
TTL: 30 นาที (auto-expire)
```

### 2.3 Real-time Leaderboards

```
ใช้ Sorted Sets (ZSet)
Key: "leaderboard:game:monthly"
Member: user_id
Score: คะแนน

ZADD leaderboard:game:monthly 9500 "user:123"
ZREVRANGE leaderboard:game:monthly 0 9  # Top 10
```

### 2.4 Rate Limiting

```
ป้องกัน API abuse
Key: "rate:limit:{user_id}:{minute}"
Value: จำนวน requests ใน minute นั้น
TTL: 60 วินาที

ถ้า value > 100 → reject request
```

### 2.5 Pub/Sub Messaging

```
Real-time features: notifications, chat, live updates
Publisher → Channel → Subscribers

PUBLISH notifications:user:123 '{"type":"message","from":"alice"}'
SUBSCRIBE notifications:user:123
```

### 2.6 Queue / Background Jobs

```
ใช้ List ทำ queue
RPUSH jobs:email '{"to":"user@example.com","template":"welcome"}'
BLPOP jobs:email 0  # Worker blocks until job arrives
```

---

## 3. ติดตั้ง Redis

### 3.1 Ubuntu / Debian

```bash
# อัพเดท packages
sudo apt update

# ติดตั้ง Redis
sudo apt install -y redis-server

# เริ่ม service
sudo systemctl start redis-server
sudo systemctl enable redis-server  # auto-start เมื่อ boot

# ตรวจสอบ status
sudo systemctl status redis-server

# ทดสอบ
redis-cli ping
# PONG

# ดู version
redis-server --version
# Redis server v=7.2.0 sha=00000000:0 malloc=jemalloc-5.3.0 bits=64

# ตำแหน่ง config file
# /etc/redis/redis.conf

# Logs
sudo journalctl -u redis-server -f
```

### 3.2 macOS (Homebrew)

```bash
# ติดตั้ง Homebrew ก่อน (ถ้ายังไม่มี)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# ติดตั้ง Redis
brew install redis

# เริ่ม Redis (background service)
brew services start redis

# หรือเริ่มแบบ foreground
redis-server

# เริ่มพร้อม config file
redis-server /usr/local/etc/redis.conf

# ทดสอบ
redis-cli ping

# ดู config file location
brew info redis
# Config: /usr/local/etc/redis.conf

# หยุด service
brew services stop redis
```

### 3.3 Docker

```bash
# Pull Redis image
docker pull redis:7-alpine

# Run Redis container พื้นฐาน
docker run -d \
  --name redis \
  -p 6379:6379 \
  redis:7-alpine

# Run พร้อม password
docker run -d \
  --name redis \
  -p 6379:6379 \
  redis:7-alpine \
  redis-server --requirepass "your_secure_password"

# Run พร้อม persistent storage
docker run -d \
  --name redis \
  -p 6379:6379 \
  -v redis_data:/data \
  redis:7-alpine \
  redis-server --appendonly yes

# เข้า redis-cli ใน container
docker exec -it redis redis-cli

# เข้าพร้อม auth
docker exec -it redis redis-cli -a your_secure_password
```

### 3.4 Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  redis:
    image: redis:7-alpine
    container_name: redis
    restart: unless-stopped
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
      - ./redis.conf:/usr/local/etc/redis/redis.conf
    command: redis-server /usr/local/etc/redis/redis.conf
    environment:
      - REDIS_REPLICATION_MODE=master
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5
    networks:
      - app-network

  redis-commander:  # Web UI สำหรับ development
    image: rediscommander/redis-commander:latest
    container_name: redis-commander
    restart: unless-stopped
    ports:
      - "8081:8081"
    environment:
      - REDIS_HOSTS=local:redis:6379
    depends_on:
      - redis
    networks:
      - app-network

volumes:
  redis_data:

networks:
  app-network:
    driver: bridge
```

```bash
# redis.conf สำหรับ development
cat > redis.conf << 'EOF'
# ฟังทุก interface (development เท่านั้น)
bind 0.0.0.0

# ไม่ใช้ password (development เท่านั้น)
# requirepass yourpassword

# Max memory
maxmemory 256mb
maxmemory-policy allkeys-lru

# Persistence: RDB
save 900 1
save 300 10
save 60 10000

# Persistence: AOF
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec

# Log level
loglevel notice

# Keyspace notifications (สำหรับ pub/sub events)
notify-keyspace-events ""
EOF

# Start
docker-compose up -d

# ดู logs
docker-compose logs -f redis
```

---

## 4. redis-cli: คำสั่งพื้นฐาน

### 4.1 เชื่อมต่อ

```bash
# เชื่อมต่อ default (localhost:6379)
redis-cli

# เชื่อมต่อ remote
redis-cli -h 192.168.1.100 -p 6379

# เชื่อมต่อพร้อม password
redis-cli -a yourpassword

# เชื่อมต่อ database เฉพาะ (default=0, มี 0-15)
redis-cli -n 1

# Single command mode
redis-cli SET mykey "hello"
redis-cli GET mykey

# Pipe mode (สำหรับ bulk operations)
cat commands.txt | redis-cli --pipe
```

### 4.2 String Commands พื้นฐาน

```bash
# SET key value
SET name "Alice"
SET age 25
SET score 98.5

# GET key
GET name         # "Alice"
GET nonexistent  # (nil)

# MSET: Set หลาย keys พร้อมกัน
MSET first "Alice" last "Johnson" city "Bangkok"

# MGET: Get หลาย keys พร้อมกัน
MGET first last city

# DEL key [key ...]
DEL name age
DEL key1 key2 key3  # ลบหลาย keys

# EXISTS key [key ...]
EXISTS name    # 1 (มีอยู่)
EXISTS notexist  # 0 (ไม่มี)
EXISTS k1 k2 k3  # จำนวน keys ที่มีอยู่

# TYPE key
TYPE name    # string
TYPE mylist  # list
TYPE myset   # set

# RENAME key newkey
RENAME oldname newname

# KEYS pattern (อย่าใช้ใน production!)
KEYS *          # ทุก key
KEYS user:*     # keys ที่ขึ้นต้นด้วย "user:"
KEYS *:profile  # keys ที่ลงท้ายด้วย ":profile"

# DBSIZE: จำนวน keys ทั้งหมด
DBSIZE

# FLUSHDB: ลบทุก key ใน current database
FLUSHDB

# FLUSHALL: ลบทุก key ทุก database
FLUSHALL

# SELECT: เปลี่ยน database (0-15)
SELECT 1
SELECT 0
```

### 4.3 Server Commands

```bash
# PING: ทดสอบ connection
PING       # PONG
PING hello # hello

# INFO: ข้อมูล server
INFO            # ทั้งหมด
INFO server     # server info
INFO clients    # client connections
INFO memory     # memory usage
INFO stats      # statistics
INFO replication # replication info
INFO keyspace   # database info

# CONFIG GET: ดู config
CONFIG GET maxmemory
CONFIG GET *          # ทุก config

# CONFIG SET: เปลี่ยน config แบบ runtime
CONFIG SET maxmemory 512mb
CONFIG SET maxmemory-policy allkeys-lru

# CONFIG REWRITE: บันทึก config ปัจจุบันลง redis.conf
CONFIG REWRITE

# SAVE: บังคับ RDB snapshot ทันที
SAVE         # synchronous (block)
BGSAVE       # asynchronous (background)

# BGREWRITEAOF: compact AOF file
BGREWRITEAOF

# LASTSAVE: เวลาที่ save ล่าสุด
LASTSAVE

# DEBUG SLEEP: ทดสอบ timeout
DEBUG SLEEP 5  # sleep 5 วินาที

# SHUTDOWN: ปิด Redis
SHUTDOWN SAVE    # save ก่อน shutdown
SHUTDOWN NOSAVE  # ปิดทันที ไม่ save
```

---

## 5. Redis Configuration

### 5.1 redis.conf หลักๆ

```bash
# /etc/redis/redis.conf หรือ /usr/local/etc/redis.conf

# ========================================
# Network
# ========================================

# bind: IP ที่รับ connections
bind 127.0.0.1        # localhost only (ปลอดภัย)
# bind 0.0.0.0        # ทุก interface (development)
# bind 127.0.0.1 ::1  # IPv4 และ IPv6

# port: port ที่ฟัง (default 6379)
port 6379

# protected-mode: เปิด/ปิด protected mode
# ถ้า bind 0.0.0.0 และไม่มี password → redis จะปฏิเสธ connections จากภายนอก
protected-mode yes

# timeout: ปิด connection ที่ idle นาน (0 = ไม่มี timeout)
timeout 300

# tcp-keepalive: keepalive interval
tcp-keepalive 300

# ========================================
# Security
# ========================================

# requirepass: password สำหรับ authentication
requirepass your_strong_password_here

# rename-command: ซ่อนหรือเปลี่ยนชื่อ dangerous commands
rename-command FLUSHDB ""           # ปิดการใช้งาน FLUSHDB
rename-command FLUSHALL ""         # ปิดการใช้งาน FLUSHALL
rename-command DEBUG ""            # ปิดการใช้งาน DEBUG
rename-command CONFIG "CONFIG_xyz"  # เปลี่ยนชื่อ CONFIG

# ========================================
# Memory Management
# ========================================

# maxmemory: จำกัด RAM ที่ใช้
maxmemory 256mb     # 256 megabytes
# maxmemory 2gb     # 2 gigabytes
# maxmemory 0       # ไม่จำกัด (ค่า default)

# maxmemory-policy: policy เมื่อ memory เต็ม
# noeviction: ปฏิเสธ write commands (default)
# allkeys-lru: evict least recently used key ใดก็ได้
# volatile-lru: evict LRU key ที่มี TTL เท่านั้น
# allkeys-random: evict random key ใดก็ได้
# volatile-random: evict random key ที่มี TTL เท่านั้น
# volatile-ttl: evict key ที่ TTL น้อยที่สุดก่อน
# allkeys-lfu: evict least frequently used key
# volatile-lfu: evict LFU key ที่มี TTL เท่านั้น
maxmemory-policy allkeys-lru

# maxmemory-samples: จำนวน samples สำหรับ LRU/LFU approximation
maxmemory-samples 5

# ========================================
# Persistence: RDB (Snapshot)
# ========================================

# save: เมื่อไหร่จะ save
# save <seconds> <changes>
save 900 1    # save ถ้ามีการเปลี่ยนแปลง 1 key ใน 900 วินาที
save 300 10   # save ถ้ามีการเปลี่ยนแปลง 10 keys ใน 300 วินาที
save 60 10000 # save ถ้ามีการเปลี่ยนแปลง 10000 keys ใน 60 วินาที

# ปิด RDB (ถ้าต้องการ AOF เท่านั้น)
# save ""

# dbfilename: ชื่อไฟล์ RDB
dbfilename dump.rdb

# dir: directory สำหรับ data files
dir /var/lib/redis

# rdbcompression: compress RDB file
rdbcompression yes

# rdbchecksum: checksum สำหรับ corruption detection
rdbchecksum yes

# ========================================
# Persistence: AOF (Append Only File)
# ========================================

# appendonly: เปิด/ปิด AOF
appendonly no  # ปิดค่า default
# appendonly yes  # เปิด

# appendfilename: ชื่อไฟล์ AOF
appendfilename "appendonly.aof"

# appendfsync: เมื่อไหร่จะ flush ลง disk
# always: flush ทุก write (ปลอดภัยที่สุด แต่ช้า)
# everysec: flush ทุกวินาที (แนะนำ - สมดุลดี)
# no: ให้ OS จัดการ (เร็วที่สุด แต่อาจสูญข้อมูลได้)
appendfsync everysec

# no-appendfsync-on-rewrite: ไม่ fsync ระหว่าง rewrite
no-appendfsync-on-rewrite no

# auto-aof-rewrite-percentage: trigger rewrite เมื่อ AOF ใหญ่กว่า base เท่านี้ %
auto-aof-rewrite-percentage 100

# auto-aof-rewrite-min-size: AOF ต้องใหญ่กว่านี้ถึงจะ rewrite
auto-aof-rewrite-min-size 64mb

# ========================================
# Logging
# ========================================

# loglevel: debug, verbose, notice, warning
loglevel notice

# logfile: path ของ log file ("" = stdout)
logfile /var/log/redis/redis-server.log

# ========================================
# Keyspace Notifications
# ========================================

# notify-keyspace-events: events ที่จะ publish
# "" = ปิด (default)
# K = Keyspace events
# E = Keyevent events
# g = Generic commands (DEL, EXPIRE, RENAME...)
# $ = String commands
# l = List commands
# s = Set commands
# h = Hash commands
# z = Sorted Set commands
# x = Expired events
# d = Module key type events
# t = Stream commands
# m = Key miss events
# A = Alias for "g$lshzxet"
notify-keyspace-events ""  # ปิด
# notify-keyspace-events "KEA"  # ทุก events
# notify-keyspace-events "Kx"   # expired events เท่านั้น

# ========================================
# Slow Log
# ========================================

# slowlog-log-slower-than: บันทึก queries ที่ช้ากว่า N microseconds
# 0 = บันทึกทุก query, -1 = ปิด
slowlog-log-slower-than 10000  # 10ms

# slowlog-max-len: จำนวน entries สูงสุดใน slow log
slowlog-max-len 128
```

### 5.2 Redis persistence: RDB vs AOF

```
RDB (Redis Database Backup):
- Snapshot ของ data ณ เวลาหนึ่ง
- ข้อดี: compact, เร็วในการ restore, เหมาะ backup
- ข้อเสีย: อาจสูญข้อมูลถ้าระบบล่มก่อน save

AOF (Append Only File):
- บันทึกทุก write command
- ข้อดี: สูญข้อมูลน้อยมาก (เสียแค่ < 1 วินาที)
- ข้อเสีย: ไฟล์ใหญ่กว่า, restore ช้ากว่า

แนะนำ: ใช้ทั้งสองอย่างพร้อมกัน
```

```bash
# ตรวจสอบ persistence status
redis-cli INFO persistence

# ผลลัพธ์:
# rdb_changes_since_last_save:0
# rdb_last_bgsave_status:ok
# rdb_last_bgsave_time_sec:0
# aof_enabled:1
# aof_current_size:1234
# aof_last_rewrite_time_sec:-1
```

---

## 6. Key Naming Conventions

### 6.1 Pattern ที่แนะนำ

```bash
# Format: "namespace:entity_type:id:field"
# ใช้ colon (:) เป็น separator

# ตัวอย่างที่ดี:
SET user:123:profile '{"name":"Alice","email":"alice@example.com"}'
SET user:123:session_token "abc123xyz"
SET product:456:inventory 42
SET order:789:status "shipped"
SET app:config:max_upload_size 10485760

# Session
SET session:abc123 '{"user_id":123,"roles":["admin"]}'

# Cache
SET cache:product:456 '{"id":456,"name":"Widget"}'
SET cache:search:query:postgresql '{"results":[...],"total":42}'

# Rate limiting
SET rate:api:user:123:2024010112 0  # user 123, hour 12 of 2024-01-01

# Counters
SET counter:page_views:homepage:2024-01-01 0
SET counter:signups:total 0

# Locks
SET lock:process:email_queue "worker_1"

# Queues
LPUSH queue:emails '{"to":"user@example.com","template":"welcome"}'
LPUSH queue:notifications '{"user_id":123,"message":"Your order shipped"}'

# Leaderboard
ZADD leaderboard:game:global 9500 "user:123"

# Feature flags
SET feature:dark_mode:enabled 1
SET feature:new_checkout:rollout_percent 25
```

### 6.2 Key Naming ที่ควรหลีกเลี่ยง

```bash
# ❌ ไม่มี namespace (ชนกันง่าย)
SET 123 "user data"
SET profile "Alice"

# ❌ ยาวเกินไป (memory expensive)
SET this:is:a:very:long:key:name:that:uses:too:much:memory:for:a:simple:value 1

# ❌ ไม่ consistent
SET user_123 "..."     # underscore
SET user:124 "..."     # colon
SET user-125 "..."     # hyphen

# ❌ Encoded data ใน key
SET user:aGVsbG8= "..."  # base64 ใน key ยากอ่าน
```

---

## 7. TTL (Time to Live)

### 7.1 คำสั่ง TTL

```bash
# SET key value EX seconds (expire ใน seconds)
SET session:abc123 "user_data" EX 3600      # expire ใน 1 ชั่วโมง

# SET key value PX milliseconds
SET temp:data "value" PX 5000               # expire ใน 5 วินาที

# SET key value EXAT unix-timestamp
SET event:countdown "2024-12-31" EXAT 1735657200

# SET key value KEEPTTL (ไม่เปลี่ยน TTL เมื่อ update ค่า)
SET session:abc123 "new_data" KEEPTTL

# EXPIRE key seconds: ตั้ง TTL ให้ key ที่มีอยู่แล้ว
EXPIRE session:abc123 3600    # expire ใน 1 ชั่วโมง

# EXPIREAT key unix-timestamp: expire ณ เวลาที่กำหนด
EXPIREAT session:abc123 1735657200

# PEXPIRE key milliseconds: TTL เป็น milliseconds
PEXPIRE temp:data 5000

# PEXPIREAT key milliseconds-timestamp
PEXPIREAT event:end 1735657200000

# TTL key: ดู TTL ที่เหลือ (seconds)
TTL session:abc123    # 3598 (วินาที)
TTL noexpire_key      # -1 (ไม่มี TTL)
TTL nonexistent       # -2 (key ไม่มีอยู่)

# PTTL key: TTL เป็น milliseconds
PTTL session:abc123   # 3598000

# PERSIST key: ลบ TTL (key จะไม่ expire)
PERSIST session:abc123

# OBJECT ENCODING key: ดู encoding ที่ใช้
OBJECT ENCODING mystring   # embstr หรือ raw
OBJECT IDLETIME key        # วินาทีที่ไม่ได้ถูก access
OBJECT FREQ key            # frequency counter (LFU)
```

### 7.2 ตัวอย่างการใช้งาน TTL

```javascript
// Node.js: Redis session management
const redis = require('ioredis');
const client = new redis({ host: 'localhost', port: 6379 });

// บันทึก session พร้อม TTL
async function saveSession(sessionId, userData, ttlSeconds = 3600) {
  const key = `session:${sessionId}`;
  await client.setex(key, ttlSeconds, JSON.stringify(userData));
  return key;
}

// อ่าน session และต่ออายุ (sliding expiration)
async function getSession(sessionId) {
  const key = `session:${sessionId}`;
  const data = await client.get(key);
  
  if (!data) return null;
  
  // ต่ออายุ session (sliding window)
  await client.expire(key, 3600);
  
  return JSON.parse(data);
}

// ตรวจสอบว่า session ยังมีอยู่ไหม
async function isSessionValid(sessionId) {
  const ttl = await client.ttl(`session:${sessionId}`);
  return ttl > 0;  // -1 = no TTL, -2 = doesn't exist
}

// ลบ session
async function destroySession(sessionId) {
  await client.del(`session:${sessionId}`);
}

// OTP (One-Time Password) with 5 minute TTL
async function saveOTP(userId, otpCode) {
  const key = `otp:${userId}`;
  await client.setex(key, 300, otpCode);  // 5 minutes
}

async function verifyOTP(userId, inputCode) {
  const key = `otp:${userId}`;
  const storedOTP = await client.get(key);
  
  if (!storedOTP) {
    return { valid: false, reason: 'OTP expired or not found' };
  }
  
  if (storedOTP !== inputCode) {
    return { valid: false, reason: 'Invalid OTP' };
  }
  
  // ลบ OTP หลังใช้งาน (one-time use)
  await client.del(key);
  return { valid: true };
}
```

---

## 8. Atomic Operations

### 8.1 Counter Operations

```bash
# INCR: เพิ่ม 1
SET page:views 0
INCR page:views    # 1
INCR page:views    # 2
INCR page:views    # 3

# DECR: ลด 1
SET inventory:item:123 100
DECR inventory:item:123    # 99

# INCRBY: เพิ่มตามจำนวนที่กำหนด
INCRBY score:user:42 10    # เพิ่ม 10
INCRBY score:user:42 25    # เพิ่ม 25 อีก

# DECRBY: ลดตามจำนวนที่กำหนด
DECRBY inventory:item:123 5   # ลด 5

# INCRBYFLOAT: เพิ่มค่า float
SET price 10.50
INCRBYFLOAT price 0.25    # 10.75
INCRBYFLOAT price -1.00   # 9.75

# GETSET: Get ค่าเดิม แล้ว Set ค่าใหม่ (atomic)
GETSET counter 0    # คืนค่าเดิม, set เป็น 0 (reset counter)

# GETDEL: Get แล้วลบ
GETDEL temp:value

# SETNX: Set ถ้า key ไม่มีอยู่ (Set if Not eXists)
SETNX lock:resource "worker_1"  # 1 ถ้า set สำเร็จ, 0 ถ้า key มีอยู่แล้ว

# SET NX EX (atomic distributed lock)
SET lock:resource "worker_1" NX EX 30    # lock expire ใน 30 วินาที
# ถ้าคืน OK = ได้ lock, ถ้า nil = มีคนอื่น lock อยู่

# SET XX: Set ถ้า key มีอยู่แล้ว
SET mykey "newvalue" XX    # OK ถ้า key มีอยู่, nil ถ้าไม่มี
```

### 8.2 Distributed Lock Pattern

```javascript
// ioredis distributed lock
class DistributedLock {
  constructor(redisClient) {
    this.redis = redisClient;
  }
  
  async acquire(resource, ttlMs = 10000) {
    const lockKey = `lock:${resource}`;
    const lockValue = `${process.pid}:${Date.now()}:${Math.random()}`;
    
    // SET key value NX PX milliseconds (atomic)
    const result = await this.redis.set(lockKey, lockValue, 'NX', 'PX', ttlMs);
    
    if (result === 'OK') {
      return lockValue;  // คืน lock token
    }
    return null;  // ไม่ได้ lock
  }
  
  async release(resource, lockValue) {
    const lockKey = `lock:${resource}`;
    
    // Lua script: check and delete atomically
    const script = `
      if redis.call("GET", KEYS[1]) == ARGV[1] then
        return redis.call("DEL", KEYS[1])
      else
        return 0
      end
    `;
    
    const result = await this.redis.eval(script, 1, lockKey, lockValue);
    return result === 1;
  }
  
  async withLock(resource, fn, ttlMs = 10000) {
    const lockValue = await this.acquire(resource, ttlMs);
    
    if (!lockValue) {
      throw new Error(`Could not acquire lock on ${resource}`);
    }
    
    try {
      return await fn();
    } finally {
      await this.release(resource, lockValue);
    }
  }
}

// ใช้งาน
const lock = new DistributedLock(redisClient);

// ป้องกัน race condition ในการหักสต็อก
await lock.withLock('inventory:product:123', async () => {
  const stock = await db.query('SELECT quantity FROM inventory WHERE product_id = 123');
  if (stock.rows[0].quantity > 0) {
    await db.query('UPDATE inventory SET quantity = quantity - 1 WHERE product_id = 123');
    return { success: true };
  }
  return { success: false, reason: 'Out of stock' };
});
```

---

## 9. SCAN vs KEYS

### 9.1 ทำไมไม่ควรใช้ KEYS ใน Production

```bash
# KEYS pattern: ค้นหา keys ทั้งหมด
# อันตราย! เพราะ:
# 1. Block Redis server ทั้งหมดระหว่างที่ scan
# 2. O(N) complexity ที่ N = จำนวน keys ทั้งหมด
# 3. ถ้ามี 10 ล้าน keys อาจ block นาน 1-2 วินาที

KEYS user:*     # ❌ อย่าใช้ใน production!
KEYS *          # ❌ อันตรายมาก!
```

### 9.2 SCAN: Safe Alternative

```bash
# SCAN cursor [MATCH pattern] [COUNT count] [TYPE type]
# cursor: เริ่มต้นที่ 0, จบเมื่อ cursor กลับมาเป็น 0
# COUNT: hint ว่าต้องการ approximately กี่ keys ต่อ iteration

# Iteration แรก (cursor=0)
SCAN 0 MATCH user:* COUNT 100
# Returns: [next_cursor, [key1, key2, ...]]

# ถ้า next_cursor ไม่ใช่ 0 ให้ scan ต่อ
SCAN 4847 MATCH user:* COUNT 100
SCAN 9823 MATCH user:* COUNT 100
# ... ทำจนกว่า cursor กลับมาเป็น 0

# HSCAN: scan Hash fields
HSCAN myhash 0 MATCH field* COUNT 50

# SSCAN: scan Set members
SSCAN myset 0 MATCH user:* COUNT 50

# ZSCAN: scan Sorted Set members
ZSCAN myzset 0 MATCH * COUNT 50
```

```javascript
// Node.js: SCAN ทุก keys ที่ match pattern
async function scanKeys(pattern) {
  const results = [];
  let cursor = '0';
  
  do {
    const [nextCursor, keys] = await redis.scan(cursor, 'MATCH', pattern, 'COUNT', 100);
    cursor = nextCursor;
    results.push(...keys);
  } while (cursor !== '0');
  
  return results;
}

// ioredis มี built-in scanStream
const stream = redis.scanStream({
  match: 'user:*',
  count: 100
});

stream.on('data', (keys) => {
  console.log('Found keys:', keys);
});

stream.on('end', () => {
  console.log('Scan complete');
});

// ลบ keys ที่ match pattern อย่างปลอดภัย
async function deleteByPattern(pattern) {
  let deleted = 0;
  let cursor = '0';
  
  do {
    const [nextCursor, keys] = await redis.scan(cursor, 'MATCH', pattern, 'COUNT', 100);
    cursor = nextCursor;
    
    if (keys.length > 0) {
      await redis.del(...keys);
      deleted += keys.length;
    }
  } while (cursor !== '0');
  
  return deleted;
}

// ใช้งาน
const count = await deleteByPattern('cache:product:*');
console.log(`Deleted ${count} cache keys`);
```

---

## 10. Redis Keyspace Notifications

### 10.1 เปิด Keyspace Notifications

```bash
# ใน redis.conf
notify-keyspace-events "KEA"
# K = Keyspace events (__keyspace@db__)
# E = Keyevent events (__keyevent@db__)
# A = Alias for "g$lshzxet" (ทุก command types)
# x = Expired events
# d = Module key type events

# หรือเปลี่ยน runtime
CONFIG SET notify-keyspace-events "KEA"

# Patterns ที่ subscribe ได้:
# __keyspace@0__:<key>     = events สำหรับ key นี้
# __keyevent@0__:<event>   = events สำหรับ event นี้

# ตัวอย่าง: subscribe expired events
SUBSCRIBE __keyevent@0__:expired
```

### 10.2 ใช้งานใน Node.js

```javascript
// Subscribe expired key events
const Redis = require('ioredis');

const subscriber = new Redis({ host: 'localhost', port: 6379 });
const publisher = new Redis({ host: 'localhost', port: 6379 });

// เปิด keyspace notifications
await publisher.config('SET', 'notify-keyspace-events', 'KEAx');

// Subscribe expired events
await subscriber.subscribe('__keyevent@0__:expired');

subscriber.on('message', (channel, message) => {
  if (channel === '__keyevent@0__:expired') {
    console.log(`Key expired: ${message}`);
    
    // ตัวอย่าง: Session expired
    if (message.startsWith('session:')) {
      const sessionId = message.replace('session:', '');
      console.log(`Session ${sessionId} expired, cleaning up...`);
      cleanupSession(sessionId);
    }
    
    // OTP expired
    if (message.startsWith('otp:')) {
      const userId = message.replace('otp:', '');
      console.log(`OTP for user ${userId} expired`);
    }
  }
});

// Subscribe ทุก events ของ key เฉพาะ
await subscriber.subscribe('__keyspace@0__:important:key');

subscriber.on('message', (channel, event) => {
  if (channel.startsWith('__keyspace@0__:')) {
    const key = channel.replace('__keyspace@0__:', '');
    console.log(`Key ${key} had event: ${event}`);
    // event = set, del, expire, expired, lpush, etc.
  }
});
```

---

## 11. Monitoring Commands

### 11.1 MONITOR Command

```bash
# MONITOR: แสดงทุก commands ที่ Redis ได้รับ real-time
# ⚠️ ใช้เพื่อ debug เท่านั้น! ลด performance 50%

redis-cli MONITOR
# 1234567890.123456 [0 127.0.0.1:54321] "SET" "key" "value"
# 1234567891.234567 [0 127.0.0.1:54322] "GET" "key"
# 1234567892.345678 [0 127.0.0.1:54321] "EXPIRE" "key" "3600"

# ออกจาก MONITOR: Ctrl+C
```

### 11.2 INFO Command

```bash
# ดูข้อมูลทั้งหมด
redis-cli INFO

# Sections:
redis-cli INFO server      # version, uptime, OS info
redis-cli INFO clients     # connected clients
redis-cli INFO memory      # memory usage
redis-cli INFO stats       # command statistics
redis-cli INFO replication # master/replica info
redis-cli INFO cpu         # CPU usage
redis-cli INFO commandstats # per-command statistics
redis-cli INFO errorstats  # error statistics
redis-cli INFO keyspace    # database info

# ตัวอย่าง memory info:
# used_memory:1234567
# used_memory_human:1.18M
# used_memory_rss:2097152
# mem_fragmentation_ratio:1.70
# mem_allocator:jemalloc-5.3.0

# ตัวอย่าง keyspace info:
# db0:keys=1234,expires=100,avg_ttl=3600000

# ดู connected clients
redis-cli CLIENT LIST
# id=5 addr=127.0.0.1:12345 fd=8 name= age=0 idle=0 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=0 qbuf-free=32768 argv-mem=10 obl=0 oll=0 omem=0 tot-mem=20512 rbs=16384 rbp=16384 resp=2 lib-name= lib-ver= events=r cmd=client|list user=default library-name= library-ver= resp=2

# Kill specific client
redis-cli CLIENT KILL ID 5
```

### 11.3 SLOWLOG Command

```bash
# ดู slow queries
redis-cli SLOWLOG GET        # ดู entries ล่าสุด (default 10)
redis-cli SLOWLOG GET 20     # ดู 20 entries ล่าสุด

# ผลลัพธ์ แต่ละ entry:
# 1) (integer) 1           # entry ID
# 2) (integer) 1609459200  # unix timestamp
# 3) (integer) 15321       # execution time (microseconds)
# 4) 1) "KEYS"             # command
#    2) "*"
# 5) "127.0.0.1:12345"     # client address
# 6) ""                    # client name

# ดูจำนวน entries ทั้งหมด
redis-cli SLOWLOG LEN

# ล้าง slow log
redis-cli SLOWLOG RESET

# ตั้งค่า slowlog threshold
redis-cli CONFIG SET slowlog-log-slower-than 10000  # 10ms
redis-cli CONFIG SET slowlog-max-len 256
```

### 11.4 LATENCY Command

```bash
# ดู latency statistics
redis-cli LATENCY LATEST     # latency ล่าสุด
redis-cli LATENCY HISTORY event  # history ของ event
redis-cli LATENCY RESET          # reset stats

# DEBUG LATENCY: สร้าง artificial latency สำหรับทดสอบ
# (ต้องเปิด latency monitor ก่อน)
redis-cli CONFIG SET latency-monitor-threshold 10  # 10ms
redis-cli CONFIG SET latency-tracking yes
```

### 11.5 Monitoring Script

```bash
#!/bin/bash
# redis-monitor.sh

REDIS_CLI="redis-cli"

echo "=== Redis Health Check ==="
echo "Timestamp: $(date)"
echo ""

echo "--- Connection ---"
$REDIS_CLI PING

echo ""
echo "--- Memory Usage ---"
$REDIS_CLI INFO memory | grep -E "used_memory_human|mem_fragmentation_ratio|maxmemory_human"

echo ""
echo "--- Connected Clients ---"
$REDIS_CLI INFO clients | grep -E "connected_clients|blocked_clients"

echo ""
echo "--- Operations Per Second ---"
$REDIS_CLI INFO stats | grep -E "instantaneous_ops_per_sec|total_commands_processed"

echo ""
echo "--- Keyspace ---"
$REDIS_CLI INFO keyspace

echo ""
echo "--- Slow Queries (last 5) ---"
$REDIS_CLI SLOWLOG GET 5

echo ""
echo "--- Replication Status ---"
$REDIS_CLI INFO replication | grep -E "role|master_host|master_port|master_link_status|connected_slaves"
```

---

## 12. Workshop: Simple Cache Layer สำหรับ API

### 12.1 Cache Service

```javascript
// cache-service.js
const Redis = require('ioredis');

class CacheService {
  constructor(options = {}) {
    this.redis = new Redis({
      host: options.host || process.env.REDIS_HOST || 'localhost',
      port: options.port || process.env.REDIS_PORT || 6379,
      password: options.password || process.env.REDIS_PASSWORD,
      db: options.db || 0,
      retryStrategy: (times) => {
        if (times > 3) return null;  // stop retry after 3 attempts
        return Math.min(times * 200, 2000);  // retry delay
      },
      reconnectOnError: (err) => {
        const targetError = 'READONLY';
        if (err.message.includes(targetError)) {
          return true;  // reconnect on READONLY error (failover)
        }
        return false;
      }
    });
    
    this.defaultTTL = options.defaultTTL || 3600;  // 1 hour
    this.keyPrefix = options.keyPrefix || 'cache:';
    
    this.redis.on('error', (err) => {
      console.error('Redis connection error:', err);
    });
    
    this.redis.on('connect', () => {
      console.log('Redis connected');
    });
  }
  
  buildKey(key) {
    return `${this.keyPrefix}${key}`;
  }
  
  async get(key) {
    try {
      const data = await this.redis.get(this.buildKey(key));
      if (!data) return null;
      return JSON.parse(data);
    } catch (error) {
      console.error('Cache GET error:', error);
      return null;  // Fail gracefully - ไม่ let cache error break application
    }
  }
  
  async set(key, value, ttlSeconds = this.defaultTTL) {
    try {
      const serialized = JSON.stringify(value);
      if (ttlSeconds > 0) {
        await this.redis.setex(this.buildKey(key), ttlSeconds, serialized);
      } else {
        await this.redis.set(this.buildKey(key), serialized);
      }
      return true;
    } catch (error) {
      console.error('Cache SET error:', error);
      return false;
    }
  }
  
  async del(key) {
    try {
      return await this.redis.del(this.buildKey(key));
    } catch (error) {
      console.error('Cache DEL error:', error);
      return 0;
    }
  }
  
  async delByPattern(pattern) {
    try {
      let deleted = 0;
      let cursor = '0';
      const fullPattern = this.buildKey(pattern);
      
      do {
        const [nextCursor, keys] = await this.redis.scan(
          cursor, 'MATCH', fullPattern, 'COUNT', 100
        );
        cursor = nextCursor;
        
        if (keys.length > 0) {
          await this.redis.del(...keys);
          deleted += keys.length;
        }
      } while (cursor !== '0');
      
      return deleted;
    } catch (error) {
      console.error('Cache DEL pattern error:', error);
      return 0;
    }
  }
  
  async getOrSet(key, fetchFn, ttlSeconds = this.defaultTTL) {
    // Cache-aside pattern
    const cached = await this.get(key);
    if (cached !== null) {
      return { data: cached, fromCache: true };
    }
    
    const data = await fetchFn();
    if (data !== null && data !== undefined) {
      await this.set(key, data, ttlSeconds);
    }
    
    return { data, fromCache: false };
  }
  
  async mget(keys) {
    try {
      const fullKeys = keys.map(k => this.buildKey(k));
      const values = await this.redis.mget(...fullKeys);
      
      return keys.reduce((result, key, index) => {
        result[key] = values[index] ? JSON.parse(values[index]) : null;
        return result;
      }, {});
    } catch (error) {
      console.error('Cache MGET error:', error);
      return {};
    }
  }
  
  async increment(key, by = 1, ttlSeconds = null) {
    try {
      const fullKey = this.buildKey(key);
      const result = await this.redis.incrby(fullKey, by);
      if (ttlSeconds && result === by) {  // ถ้า key เพิ่งสร้าง ให้ set TTL
        await this.redis.expire(fullKey, ttlSeconds);
      }
      return result;
    } catch (error) {
      console.error('Cache INCREMENT error:', error);
      return null;
    }
  }
  
  async close() {
    await this.redis.quit();
  }
}

module.exports = CacheService;
```

### 12.2 API Middleware

```javascript
// cache-middleware.js
const CacheService = require('./cache-service');

const cache = new CacheService({
  keyPrefix: 'api:',
  defaultTTL: 300  // 5 minutes
});

// Middleware สำหรับ cache API responses
function cacheMiddleware(options = {}) {
  const {
    ttl = 300,
    keyGenerator = (req) => `${req.method}:${req.originalUrl}`,
    condition = () => true,  // function ที่กำหนดว่าจะ cache ไหม
    varyBy = [],             // headers ที่ใช้ทำ cache key ต่างกัน (เช่น Accept-Language)
  } = options;
  
  return async (req, res, next) => {
    // Cache เฉพาะ GET requests
    if (req.method !== 'GET') return next();
    
    // ตรวจสอบ condition
    if (!condition(req)) return next();
    
    // สร้าง cache key
    let cacheKey = keyGenerator(req);
    for (const header of varyBy) {
      const headerValue = req.get(header);
      if (headerValue) {
        cacheKey += `:${header.toLowerCase()}:${headerValue}`;
      }
    }
    
    // ตรวจสอบ Cache-Control header
    const cacheControl = req.get('Cache-Control');
    if (cacheControl && cacheControl.includes('no-cache')) {
      return next();
    }
    
    // ลอง get จาก cache
    const cachedResponse = await cache.get(cacheKey);
    
    if (cachedResponse) {
      res.set('X-Cache', 'HIT');
      res.set('X-Cache-Key', cacheKey);
      return res.status(cachedResponse.status).json(cachedResponse.body);
    }
    
    // ไม่มีใน cache: ดักจับ response
    res.set('X-Cache', 'MISS');
    
    const originalJson = res.json.bind(res);
    res.json = async (body) => {
      // Cache เฉพาะ successful responses
      if (res.statusCode >= 200 && res.statusCode < 300) {
        await cache.set(cacheKey, {
          status: res.statusCode,
          body: body
        }, ttl);
      }
      return originalJson(body);
    };
    
    next();
  };
}

// Rate limiting middleware
function rateLimitMiddleware(options = {}) {
  const {
    windowSeconds = 60,
    maxRequests = 100,
    keyGenerator = (req) => `rate:${req.ip}`,
    onLimitReached = (req, res) => {
      res.status(429).json({
        error: 'Too Many Requests',
        retryAfter: windowSeconds
      });
    }
  } = options;
  
  return async (req, res, next) => {
    const key = keyGenerator(req);
    const count = await cache.increment(key, 1, windowSeconds);
    
    const remaining = Math.max(0, maxRequests - count);
    res.set('X-RateLimit-Limit', maxRequests);
    res.set('X-RateLimit-Remaining', remaining);
    
    if (count > maxRequests) {
      return onLimitReached(req, res);
    }
    
    next();
  };
}

module.exports = { cacheMiddleware, rateLimitMiddleware };
```

### 12.3 Express Application

```javascript
// app.js
const express = require('express');
const { cacheMiddleware, rateLimitMiddleware } = require('./cache-middleware');
const CacheService = require('./cache-service');
const { Pool } = require('pg');

const app = express();
const pool = new Pool({ /* ... */ });
const cache = new CacheService();

app.use(express.json());

// Global rate limiting: 1000 requests/minute per IP
app.use(rateLimitMiddleware({
  windowSeconds: 60,
  maxRequests: 1000
}));

// Products API
app.get('/api/products',
  cacheMiddleware({ ttl: 60 }),  // cache 1 minute
  async (req, res) => {
    const { page = 1, limit = 20, category } = req.query;
    const offset = (page - 1) * limit;
    
    let query = 'SELECT * FROM products';
    const params = [];
    
    if (category) {
      query += ' WHERE category_id = $1';
      params.push(category);
    }
    
    query += ` ORDER BY id LIMIT $${params.length + 1} OFFSET $${params.length + 2}`;
    params.push(limit, offset);
    
    const result = await pool.query(query, params);
    res.json({ products: result.rows, page, limit });
  }
);

app.get('/api/products/:id',
  cacheMiddleware({ 
    ttl: 300,
    keyGenerator: (req) => `product:${req.params.id}`
  }),
  async (req, res) => {
    const { id } = req.params;
    const result = await pool.query(
      'SELECT * FROM products WHERE id = $1', [id]
    );
    
    if (result.rows.length === 0) {
      return res.status(404).json({ error: 'Product not found' });
    }
    
    res.json(result.rows[0]);
  }
);

// Update product และ invalidate cache
app.put('/api/products/:id', async (req, res) => {
  const { id } = req.params;
  const { name, price } = req.body;
  
  const result = await pool.query(
    'UPDATE products SET name = $1, price = $2 WHERE id = $3 RETURNING *',
    [name, price, id]
  );
  
  if (result.rows.length === 0) {
    return res.status(404).json({ error: 'Product not found' });
  }
  
  // Invalidate cache
  await cache.del(`product:${id}`);
  await cache.delByPattern('GET:/api/products*');  // invalidate list caches
  
  res.json(result.rows[0]);
});

// Search endpoint with cache
app.get('/api/search',
  cacheMiddleware({ 
    ttl: 30,  // search results change faster
    keyGenerator: (req) => `search:${JSON.stringify(req.query)}`
  }),
  async (req, res) => {
    const { q, category, minPrice, maxPrice } = req.query;
    
    if (!q) return res.status(400).json({ error: 'Query required' });
    
    const result = await pool.query(`
      SELECT p.*, c.name AS category_name
      FROM products p
      JOIN categories c ON p.category_id = c.id
      WHERE p.name ILIKE $1
          AND ($2::INTEGER IS NULL OR c.id = $2)
          AND ($3::DECIMAL IS NULL OR p.price >= $3)
          AND ($4::DECIMAL IS NULL OR p.price <= $4)
      ORDER BY p.name
      LIMIT 50
    `, [`%${q}%`, category || null, minPrice || null, maxPrice || null]);
    
    res.json({ results: result.rows, query: q });
  }
);

// Cache stats endpoint
app.get('/api/admin/cache-stats', async (req, res) => {
  const info = await cache.redis.info('memory');
  const keyspace = await cache.redis.info('keyspace');
  
  res.json({
    memory: info,
    keyspace: keyspace,
    timestamp: new Date().toISOString()
  });
});

// Health check
app.get('/health', async (req, res) => {
  try {
    await cache.redis.ping();
    res.json({ 
      status: 'healthy', 
      redis: 'connected',
      timestamp: new Date().toISOString()
    });
  } catch (error) {
    res.status(503).json({ 
      status: 'degraded', 
      redis: 'disconnected',
      error: error.message 
    });
  }
});

app.listen(3000, () => console.log('Server running on port 3000'));
```

### 12.4 Testing Cache

```javascript
// test-cache.js
const CacheService = require('./cache-service');

async function runTests() {
  const cache = new CacheService({ keyPrefix: 'test:' });
  
  console.log('=== Cache Service Tests ===\n');
  
  // Test 1: Basic set/get
  console.log('Test 1: Set and Get');
  await cache.set('user:1', { name: 'Alice', age: 30 }, 60);
  const user = await cache.get('user:1');
  console.log('Got user:', user);
  console.assert(user.name === 'Alice', 'User name should be Alice');
  
  // Test 2: TTL
  console.log('\nTest 2: TTL');
  await cache.set('temp:data', 'expires soon', 2);
  const ttl = await cache.redis.ttl('test:temp:data');
  console.log('TTL:', ttl, 'seconds');
  console.assert(ttl > 0 && ttl <= 2, 'TTL should be between 0 and 2');
  
  // Wait for expiry
  await new Promise(resolve => setTimeout(resolve, 3000));
  const expired = await cache.get('temp:data');
  console.log('After expiry:', expired);
  console.assert(expired === null, 'Should be null after expiry');
  
  // Test 3: getOrSet
  console.log('\nTest 3: getOrSet (Cache-aside pattern)');
  let callCount = 0;
  
  const result1 = await cache.getOrSet('product:1', async () => {
    callCount++;
    return { id: 1, name: 'Widget', price: 99.99 };
  }, 60);
  
  const result2 = await cache.getOrSet('product:1', async () => {
    callCount++;
    return { id: 1, name: 'Widget', price: 99.99 };
  }, 60);
  
  console.log('Fetch function called:', callCount, 'times (should be 1)');
  console.log('First call from cache:', result1.fromCache);
  console.log('Second call from cache:', result2.fromCache);
  console.assert(callCount === 1, 'Fetch should be called only once');
  console.assert(!result1.fromCache, 'First call should not be from cache');
  console.assert(result2.fromCache, 'Second call should be from cache');
  
  // Test 4: Increment
  console.log('\nTest 4: Increment (Rate limiting simulation)');
  for (let i = 0; i < 5; i++) {
    const count = await cache.increment('counter:test', 1, 60);
    console.log(`Count: ${count}`);
  }
  
  // Test 5: Batch operations
  console.log('\nTest 5: Batch mget');
  await cache.set('item:1', { id: 1 });
  await cache.set('item:2', { id: 2 });
  const items = await cache.mget(['item:1', 'item:2', 'item:3']);
  console.log('Batch result:', items);
  
  // Test 6: Pattern delete
  console.log('\nTest 6: Delete by pattern');
  await cache.set('category:1', 'Electronics');
  await cache.set('category:2', 'Clothing');
  await cache.set('category:3', 'Books');
  
  const deleted = await cache.delByPattern('category:*');
  console.log(`Deleted ${deleted} keys`);
  
  const check = await cache.get('category:1');
  console.assert(check === null, 'Category should be deleted');
  
  console.log('\n=== All Tests Passed ===');
  await cache.close();
}

runTests().catch(console.error);
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Redis คืออะไร**: In-memory data store ที่เร็วมาก
2. **Use Cases**: Caching, Sessions, Leaderboards, Rate Limiting, Pub/Sub
3. **ติดตั้ง Redis**: Ubuntu, macOS, Docker และ Docker Compose
4. **redis-cli**: คำสั่งพื้นฐานทั้งหมด
5. **Configuration**: maxmemory, persistence (RDB/AOF), security
6. **Key Naming**: conventions ที่ดีสำหรับ maintainability
7. **TTL**: การตั้ง expire ด้วย EXPIRE, EXPIREAT, SET EX
8. **Atomic Operations**: INCR, INCRBY, SETNX สำหรับ race-condition-free operations
9. **SCAN vs KEYS**: ทำไม KEYS อันตรายใน production
10. **Keyspace Notifications**: รับ events เมื่อ keys เปลี่ยนแปลง
11. **Monitoring**: MONITOR, INFO, SLOWLOG
12. **Workshop**: Cache layer ที่สมบูรณ์สำหรับ Express API

บทถัดไปจะเรียนรู้ Redis Data Structures ทั้งหมด: String, Hash, List, Set, ZSet และ data structures พิเศษอื่นๆ

---

*[Part 08 จบ — ไปต่อ Part 09: Redis Data Structures]*

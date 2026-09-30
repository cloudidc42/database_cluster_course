# Part 60: Performance Tuning Redis

## บทนำ: Redis Performance Fundamentals

Redis เป็น in-memory data store ที่ออกแบบมาให้เร็วโดยธรรมชาติ แต่การตั้งค่าผิดสามารถทำให้ประสิทธิภาพลดลงอย่างมาก เช่น:

```
Redis Default Configuration Issues:
├── maxmemory ไม่ตั้งค่า → OOM killer อาจ kill process
├── appendfsync = always → Disk I/O bottleneck
├── save "900 1" → Blocking RDB save ทำให้ freeze 5-10 seconds
├── maxmemory-policy = noeviction → Errors เมื่อ memory เต็ม
└── tcp-backlog = 511 → Connection queue overflow ใน high load
```

### Redis Performance Architecture

```
Client Request
     │
     ▼
┌─────────────────────────────────────────┐
│          Redis Event Loop               │
│  (Single-threaded command processing)  │
│                                         │
│  ┌─────────┐  ┌─────────┐  ┌────────┐ │
│  │ Network │  │Commands │  │ Timer  │ │
│  │  I/O    │  │ Process │  │ Events │ │
│  └─────────┘  └─────────┘  └────────┘ │
└─────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────┐
│          Memory Store                   │
│  ┌──────────┐  ┌──────────┐            │
│  │  Hash    │  │  Skip    │            │
│  │  Tables  │  │  Lists   │            │
│  └──────────┘  └──────────┘            │
└─────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────┐
│       Persistence (Background)          │
│  RDB: fork() + bgsave                  │
│  AOF: write() to disk                  │
└─────────────────────────────────────────┘
```

---

## Memory Optimization

### maxmemory: กำหนดขอบเขต Memory

```bash
# redis.conf
# ==================== maxmemory ====================

# กำหนด maxmemory เสมอ! ไม่งั้น Redis ใช้ RAM หมด
# ค่าแนะนำ: 75-85% ของ available RAM

# Server 16GB RAM:
maxmemory 12gb

# Server 8GB RAM:
maxmemory 6gb

# คำนวณ:
# Physical RAM: 32GB
# OS + other processes: ~4GB
# Redis maxmemory: 24-26GB

# ตั้งค่า runtime (ไม่ต้อง restart)
redis-cli CONFIG SET maxmemory 12gb

# ดูค่าปัจจุบัน
redis-cli CONFIG GET maxmemory
redis-cli INFO memory | grep maxmemory
```

### maxmemory-policy: Eviction Strategies

เมื่อ memory เต็ม Redis จะใช้ policy เพื่อตัดสินใจว่าจะลบ key ไหน

```bash
# ==================== Eviction Policies ====================

# 1. noeviction (default)
# - ไม่ evict key ใด
# - Return error เมื่อ memory เต็ม (สำหรับ write commands)
# - ใช้เมื่อ: Redis ทำหน้าที่เป็น primary database (ไม่ใช่ cache)
# - ข้อเสีย: Application จะ error เมื่อ memory เต็ม
maxmemory-policy noeviction

# 2. allkeys-lru (แนะนำสำหรับ general cache)
# - Evict key ที่ใช้นานที่สุด (Least Recently Used) จาก ALL keys
# - ดีสำหรับ: General caching ที่ทุก key มีโอกาสเข้าถึงเท่าๆ กัน
# - ข้อดี: Simple, predictable, ทำงานได้ดีกับส่วนใหญ่
maxmemory-policy allkeys-lru

# 3. volatile-lru
# - Evict key ที่ใช้นานที่สุด เฉพาะ keys ที่มี TTL เท่านั้น
# - ดีสำหรับ: มี keys ทั้ง permanent และ cached
# - Permanent keys (ไม่มี TTL) จะไม่ถูก evict
# - ข้อเสีย: ถ้า keys มี TTL น้อย อาจ evict ผิด key
maxmemory-policy volatile-lru

# 4. allkeys-lfu (Redis 4.0+)
# - Evict key ที่ใช้น้อยที่สุด (Least Frequently Used) จาก ALL keys
# - ดีกว่า LRU เมื่อ access pattern ไม่สม่ำเสมอ (hot keys stay)
# - ดีสำหรับ: Hot data ที่เข้าถึงบ่อยมาก, cold data เข้าถึงน้อย
maxmemory-policy allkeys-lfu

# 5. volatile-lfu (Redis 4.0+)
# - Evict key LFU เฉพาะ keys ที่มี TTL
maxmemory-policy volatile-lfu

# 6. allkeys-random
# - Evict key แบบสุ่ม จาก ALL keys
# - ใช้เมื่อ: ไม่สนว่าจะลบอะไร (rare case)
maxmemory-policy allkeys-random

# 7. volatile-random  
# - Evict key แบบสุ่ม เฉพาะ keys ที่มี TTL
maxmemory-policy volatile-random

# 8. volatile-ttl
# - Evict key ที่จะหมดอายุเร็วที่สุด
# - ดีสำหรับ: ต้องการ priority ตาม TTL
maxmemory-policy volatile-ttl

# ==================== Decision Matrix ====================
#
# Use Case                       | Recommended Policy
# -------------------------------|-------------------
# General caching                | allkeys-lru
# Mixed cache + permanent data   | volatile-lru
# Hot/cold access pattern        | allkeys-lfu
# Cache with specific TTLs       | volatile-ttl
# Primary database (no eviction) | noeviction
# Simple cache, don't care       | allkeys-random
```

### LFU Configuration

```bash
# LFU tuning parameters (Redis 4.0+)
# lfu-log-factor: ความ sensitive ของ counter
# ค่าต่ำ = counter เพิ่มเร็ว = ต้องการ access น้อยกว่าเพื่อ increment
lfu-log-factor 10  # default, range: 0-255

# lfu-decay-time: เวลา (นาที) ก่อน counter ลดลง 1
lfu-decay-time 1   # default

# ดู LFU counter
redis-cli OBJECT FREQ mykey
# (integer) 5  ← access frequency score

# ดู key ที่ access บ่อยที่สุด
redis-cli --hotkeys --scan
```

---

## Memory Encoding Optimizations

Redis ใช้ encoding ที่แตกต่างกันตาม data size เพื่อประหยัด memory

### Hash Encoding

```bash
# Hash ที่มีขนาดเล็ก ใช้ listpack (เดิมคือ ziplist) แทน hashtable
# listpack: compact array = memory น้อยกว่า แต่ O(n) lookup
# hashtable: standard hash = memory มากกว่า แต่ O(1) lookup

# กำหนด threshold
hash-max-listpack-entries 128   # หาก hash มี <= 128 fields ใช้ listpack
hash-max-listpack-value 64      # หาก field value <= 64 bytes ใช้ listpack

# ดู encoding ของ key
redis-cli OBJECT ENCODING myhash
# "listpack" หรือ "hashtable"

# ตัวอย่าง:
redis-cli HSET small:hash f1 v1 f2 v2 f3 v3
redis-cli OBJECT ENCODING small:hash
# "listpack"  ← ประหยัด memory

redis-cli HSET large:hash $(for i in $(seq 1 200); do echo "field$i value$i"; done)
redis-cli OBJECT ENCODING large:hash
# "hashtable"  ← ต้องการ hash table

# Memory comparison:
# listpack: ~150 bytes สำหรับ 128 fields
# hashtable: ~1,500 bytes สำหรับ 128 fields
```

### List Encoding

```bash
# List ใช้ quicklist (linked list ของ listpack nodes)
list-max-listpack-size -2   # ขนาดสูงสุดของแต่ละ node
# -5: max 64kb
# -4: max 32kb
# -3: max 16kb
# -2: max 8kb (recommended)
# -1: max 4kb
# positive N: N elements per node

list-max-ziplist-size -2    # backward compat alias

# ดู encoding
redis-cli OBJECT ENCODING mylist
# "listpack" (สำหรับ list สั้นๆ) หรือ "quicklist"
```

### Set Encoding

```bash
# Set ที่มีแต่ integers ใช้ intset (sorted array)
# ประหยัด memory มากกว่า hashtable

set-max-intset-entries 512  # หาก set มี integers ≤ 512 ใช้ intset

# ตัวอย่าง:
redis-cli SADD int:set 1 2 3 4 5
redis-cli OBJECT ENCODING int:set
# "intset"  ← compact!

redis-cli SADD str:set "hello" "world"
redis-cli OBJECT ENCODING str:set
# "listpack" หรือ "hashtable" (ไม่ใช่ intset เพราะมี strings)
```

### Sorted Set Encoding

```bash
# Sorted Set (ZSet) ใช้ listpack เมื่อมีขนาดเล็ก
zset-max-listpack-entries 128   # ≤ 128 members ใช้ listpack
zset-max-listpack-value 64      # member value ≤ 64 bytes ใช้ listpack

# เมื่อเกิน threshold → skiplist + hashtable

# ดู encoding
redis-cli OBJECT ENCODING myzset
# "listpack" หรือ "skiplist"
```

### Optimize Hash-based Object Storage

```bash
# Pattern: เก็บ objects หลายๆ ตัวใน single hash key
# แทน user:1 user:2 user:3 (3 keys)
# ใช้ users:0 {field: "1:name", value: "Alice"} ใน hash

# ตัวอย่าง: เก็บ user objects
# แทน:
HMSET user:1001 name "Alice" email "alice@example.com" age 25
HMSET user:1002 name "Bob" email "bob@example.com" age 30

# ใช้ hash bucketing:
HSET users:10 "1001:name" "Alice" "1001:email" "alice@example.com" "1001:age" 25
HSET users:10 "1002:name" "Bob" "1002:email" "bob@example.com" "1002:age" 30
# Key = "users:" + (user_id / 100)

# Memory saving: 10x ประหยัดกว่า individual keys
# เพราะ hash overhead ต่อ key (metadata, etc.) ลดลงมาก
```

### Memory Fragmentation

```bash
# ดู fragmentation ratio
redis-cli INFO memory | grep -E "used_memory|fragmentation"

# used_memory_rss / used_memory = fragmentation ratio
# 1.0 = ไม่มี fragmentation
# 1.5 = 50% fragmentation (เสียพื้นที่ 50%)
# > 2.0 = สูงมาก ควรดำเนินการ

# ==================== Active Defrag ====================
# Redis 4.0+: Active defragmentation (ทำงาน background)
activedefrag yes

# เริ่ม defrag เมื่อ fragmentation > 10%
active-defrag-ignore-bytes 100mb    # min เสีย 100MB ก่อนเริ่ม defrag
active-defrag-enabled yes
active-defrag-threshold-lower 10    # เริ่มเมื่อ fragmentation > 10%
active-defrag-threshold-upper 100   # aggressive defrag เมื่อ > 100%
active-defrag-cycle-min 1           # % CPU ขั้นต่ำสำหรับ defrag
active-defrag-cycle-max 25          # % CPU สูงสุดสำหรับ defrag
active-defrag-max-scan-fields 1000  # max fields scan per cycle

# ตรวจสอบ fragmentation
redis-cli MEMORY DOCTOR
# "Sam, I detected a few problems with this Redis instance:
#  * WARNING: fragmentation ratio is 1.83 (above 1.5)"

# Force defrag (ระวัง: ใช้ CPU สูง)
redis-cli MEMORY PURGE
```

---

## Persistence Performance

### RDB Configuration

```bash
# RDB: Snapshot-based persistence
# 장점: compact, fast restore
# ข้อเสีย: อาจสูญเสียข้อมูลตั้งแต่ snapshot ล่าสุด

# กำหนด save frequency
# Format: save <seconds> <changes>
# Disable RDB: save ""

# สำหรับ cache-only (ไม่ต้องการ persistence)
save ""

# สำหรับ balance ระหว่าง safety และ performance
save 3600 1    # บันทึกถ้ามีการเปลี่ยนแปลง >= 1 ครั้งใน 1 ชั่วโมง
save 300 100   # บันทึกถ้ามีการเปลี่ยนแปลง >= 100 ครั้งใน 5 นาที
save 60 10000  # บันทึกถ้ามีการเปลี่ยนแปลง >= 10,000 ครั้งใน 1 นาที

# ==================== RDB Compression ====================
rdbcompression yes    # Compress RDB file ด้วย LZF

# RDB Checksum (เพิ่ม data safety แต่ช้าลงเล็กน้อย ~10%)
rdbchecksum yes

# RDB filename
dbfilename dump.rdb
dir /var/lib/redis

# ==================== Manual RDB Save ====================
# Background save (non-blocking):
redis-cli BGSAVE

# Synchronous save (blocking! ห้ามใช้ใน production):
redis-cli SAVE

# ดูเวลา save ล่าสุด
redis-cli LASTSAVE
# (integer) 1705312800

# ดู info เกี่ยวกับ RDB
redis-cli INFO persistence | grep rdb
```

### AOF Configuration

```bash
# AOF: Append-Only File
# ข้อดี: Durable (ข้อมูลสูญหายน้อยกว่า RDB)
# ข้อเสีย: ไฟล์ใหญ่กว่า, restart ช้ากว่า

# เปิด AOF
appendonly yes
appendfilename "appendonly.aof"

# ==================== appendfsync Options ====================

# always: fsync ทุก write command
# - ปลอดภัยที่สุด (ไม่มี data loss)
# - ช้าที่สุด (~1000 writes/sec บน HDD)
# - ไม่แนะนำ สำหรับ high-throughput
appendfsync always

# everysec (แนะนำ): fsync ทุก 1 วินาที
# - สูญเสียข้อมูลได้สูงสุด 1 วินาที
# - เร็วกว่า always มาก (~10,000-100,000 writes/sec)
# - Balance ดี ระหว่าง durability และ performance
appendfsync everysec

# no: ไม่ fsync เลย (OS จัดการ)
# - เร็วที่สุด แต่อาจสูญหายข้อมูลมาก
# - ไม่แนะนำ สำหรับ production
appendfsync no

# ==================== AOF Rewrite ====================
# AOF เติบโตเรื่อยๆ จาก command logs
# Rewrite: สร้าง AOF ใหม่จาก current state (compact)

# Rewrite อัตโนมัติเมื่อ AOF โตเป็น 2x ของ base size
auto-aof-rewrite-percentage 100   # 100% growth
auto-aof-rewrite-min-size 64mb    # ขนาดขั้นต่ำก่อน rewrite

# ตั้งค่า rewrite ให้ไม่ blocking ใน production:
# no-appendfsync-on-rewrite yes  
# ยอมสูญหายข้อมูล 1-30 วินาที ระหว่าง rewrite
# เพื่อแลกกับ performance ที่ดีขึ้น

no-appendfsync-on-rewrite no  # default: safe mode
# เปลี่ยนเป็น yes ถ้า performance สำคัญกว่า

# Manual AOF rewrite
redis-cli BGREWRITEAOF

# ดู AOF stats
redis-cli INFO persistence | grep aof
```

### AOF vs RDB: Trade-offs

```bash
# ==================== AOF vs RDB Comparison ====================

# RDB (Snapshot)
# ✓ ไฟล์เล็กกว่า (compact binary)
# ✓ Recovery เร็วกว่า (load binary vs replay commands)
# ✓ Performance ดีกว่า (background fork)
# ✗ อาจสูญหายข้อมูลตั้งแต่ snapshot ล่าสุด (นาที-ชั่วโมง)
# ✗ Fork() ใช้ memory เป็น 2x ชั่วคราว (Copy-on-Write)
# เหมาะ: Cache, Non-critical data

# AOF (Append-Only File)
# ✓ Durable กว่า (สูญหายสูงสุด 1 วินาที ด้วย everysec)
# ✓ Human-readable (text commands)
# ✓ Rewrite ไม่หยุดระบบ
# ✗ ไฟล์ใหญ่กว่า
# ✗ Recovery ช้ากว่า (replay all commands)
# ✗ Slightly slower writes
# เหมาะ: Financial data, Session data

# ใช้ทั้งสอง (Hybrid - แนะนำสำหรับ production)
# RDB + AOF = recovery ไว (RDB) + data safety (AOF)
appendonly yes
save 3600 1  # เปิด RDB ด้วย (สำหรับ full backup)

# Redis 7.0+: Multi-part AOF (MP-AOF)
# แก้ปัญหา AOF rewrite blocking
aof-use-rdb-preamble yes  # แนะนำ: ใช้ RDB ใน AOF rewrite
```

---

## Network Optimization

### TCP Configuration

```bash
# tcp-backlog: queue size สำหรับ incoming connections
# Default: 511
# ควรเพิ่มสำหรับ high-concurrency systems
# Note: Linux kernel ต้องตั้ง /proc/sys/net/core/somaxconn ให้มากกว่าด้วย
tcp-backlog 65535

# System setting:
# echo 65535 > /proc/sys/net/core/somaxconn
# echo 65535 > /proc/sys/net/ipv4/tcp_max_syn_backlog

# tcp-keepalive: ส่ง keepalive packets
# ป้องกัน dead connections ไม่ถูก clean up
tcp-keepalive 300  # 300 seconds

# bind: กำหนด IP ที่ Redis ฟัง
bind 127.0.0.1 10.0.0.1  # localhost + private IP เท่านั้น

# protected-mode: ป้องกันการ access จากภายนอก
protected-mode yes  # Keep this ON!
```

### Client Output Buffer Limits

```bash
# จำกัด output buffer ต่อ client เพื่อป้องกัน memory exhaustion

# Format: client-output-buffer-limit <class> <hard limit> <soft limit> <soft seconds>
# hard limit: disconnect ทันที
# soft limit: disconnect หลัง soft seconds ถ้า buffer > soft limit

# Normal clients
client-output-buffer-limit normal 0 0 0

# Slave/Replica (Replication backlog)
client-output-buffer-limit slave 256mb 64mb 60

# Pub/Sub clients
client-output-buffer-limit pubsub 32mb 8mb 60

# ==================== maxclients ====================
maxclients 10000  # default 10000

# ดู clients
redis-cli INFO clients
redis-cli CLIENT LIST
redis-cli CLIENT LIST ID 123  # specific client
```

---

## Pipelining: Batch Commands

### ทำไม Pipelining ถึงเร็วกว่า

```bash
# Without pipelining (Round-trip time ต่อ command):
# 1. Client → Redis: SET key1 val1
# 2. Redis → Client: +OK
# 3. Client → Redis: SET key2 val2
# 4. Redis → Client: +OK
# ... repeat 1000 times
# Total: 1000 × RTT (เช่น 1000 × 0.1ms = 100ms)

# With pipelining:
# 1. Client → Redis: SET key1 val1\r\nSET key2 val2\r\n... (1000 commands)
# 2. Redis → Client: +OK\r\n+OK\r\n... (1000 responses)
# Total: 1 × RTT + processing (≈ 1ms)
```

```python
# Python Redis Pipeline example
import redis
import time

r = redis.Redis(host='localhost', port=6379)

# Without pipeline (slow)
start = time.time()
for i in range(10000):
    r.set(f'key:{i}', f'value:{i}')
print(f"Without pipeline: {time.time() - start:.2f}s")
# Without pipeline: 2.50s

# With pipeline (fast!)
start = time.time()
pipe = r.pipeline()
for i in range(10000):
    pipe.set(f'key:{i}', f'value:{i}')
pipe.execute()
print(f"With pipeline: {time.time() - start:.2f}s")
# With pipeline: 0.05s (50x faster!)
```

```typescript
// Node.js (ioredis) Pipeline example
import Redis from 'ioredis';

const redis = new Redis();

// Pipeline
const pipeline = redis.pipeline();
for (let i = 0; i < 10000; i++) {
  pipeline.set(`key:${i}`, `value:${i}`, 'EX', 3600);
}
const results = await pipeline.exec();
// results = array of [error, result] pairs

// Multi pipeline
const multi = redis.multi();
multi.get('user:123');
multi.hgetall('user:123:profile');
multi.smembers('user:123:permissions');
const [user, profile, permissions] = await multi.exec();

// Batch with error handling
const pipe2 = redis.pipeline();
pipe2.set('key1', 'value1');
pipe2.lpush('list1', 'item1');
pipe2.hset('hash1', 'field1', 'value1');

const pipeResults = await pipe2.exec();
pipeResults?.forEach(([err, result], index) => {
  if (err) {
    console.error(`Command ${index} failed:`, err);
  }
});
```

---

## Lua Scripting: Atomic Operations

```bash
# Lua scripts run atomically (ไม่มี interruption ระหว่างกลาง)
# เหมาะสำหรับ: check-and-set, conditional operations

# ==================== Example: Atomic Counter with Limit ====================
# ปัญหา: เพิ่ม counter ถ้าไม่เกิน limit (ต้องการ atomic)

# Lua script:
local current = redis.call('GET', KEYS[1])
if current == false then
  current = 0
end
if tonumber(current) < tonumber(ARGV[1]) then
  return redis.call('INCR', KEYS[1])
else
  return -1
end

# รัน script:
redis-cli EVAL "
local current = redis.call('GET', KEYS[1])
if current == false then current = 0 end
if tonumber(current) < tonumber(ARGV[1]) then
  return redis.call('INCR', KEYS[1])
else
  return -1
end
" 1 counter:user:123 100
# KEYS[1] = counter:user:123 (key)
# ARGV[1] = 100 (limit)
```

```typescript
// ioredis Lua script example
import Redis from 'ioredis';

const redis = new Redis();

// Define reusable script
const rateLimiterScript = `
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])

local current = redis.call('GET', key)
if current == false then
  redis.call('SET', key, 1, 'EX', window)
  return 1
end

local count = tonumber(current)
if count < limit then
  redis.call('INCR', key)
  return count + 1
else
  return 0  -- 0 = rate limited
end
`;

// Load script (gets SHA hash)
const sha = await redis.script('load', rateLimiterScript);
console.log('Script SHA:', sha);

// Use script via SHA (faster, no re-send)
async function checkRateLimit(
  userId: string,
  limit: number,
  windowSeconds: number
): Promise<boolean> {
  const key = `rate:${userId}`;
  
  try {
    const result = await redis.evalsha(sha, 1, key, limit, windowSeconds) as number;
    return result > 0;  // true = allowed, false = rate limited
  } catch (err) {
    if ((err as Error).message.includes('NOSCRIPT')) {
      // Script not cached, reload
      await redis.script('load', rateLimiterScript);
      const result = await redis.eval(rateLimiterScript, 1, key, limit, windowSeconds) as number;
      return result > 0;
    }
    throw err;
  }
}

// ==================== Distributed Lock ====================
const lockScript = `
local key = KEYS[1]
local value = ARGV[1]
local ttl = ARGV[2]

local current = redis.call('SET', key, value, 'NX', 'EX', ttl)
if current == 'OK' then
  return 1
else
  return 0
end
`;

const unlockScript = `
local key = KEYS[1]
local value = ARGV[1]

local current = redis.call('GET', key)
if current == value then
  redis.call('DEL', key)
  return 1
else
  return 0
end
`;

async function acquireLock(resource: string, ttlSeconds: number): Promise<string | null> {
  const lockKey = `lock:${resource}`;
  const lockValue = `${Date.now()}-${Math.random()}`;
  
  const result = await redis.eval(lockScript, 1, lockKey, lockValue, ttlSeconds) as number;
  return result === 1 ? lockValue : null;
}

async function releaseLock(resource: string, lockValue: string): Promise<boolean> {
  const lockKey = `lock:${resource}`;
  const result = await redis.eval(unlockScript, 1, lockKey, lockValue) as number;
  return result === 1;
}
```

---

## Benchmarking ด้วย redis-benchmark

### Basic Benchmarks

```bash
# ==================== Basic Benchmark ====================
# -h: host
# -p: port
# -c: concurrent connections (default 50)
# -n: total requests (default 100000)
# -d: data size in bytes (default 3)
# -t: specific tests (SET, GET, INCR, LPUSH, etc.)
# -q: quiet mode (show only numbers)

# Basic test: 50 connections, 100,000 requests, 100 bytes data
redis-benchmark -c 50 -n 100000 -d 100 -q

# Output:
# PING_INLINE: 145348.84 requests per second, p50=0.343 msec
# PING_MBULK: 147058.83 requests per second
# SET: 121359.22 requests per second, p50=0.399 msec
# GET: 137931.03 requests per second, p50=0.351 msec
# INCR: 133333.33 requests per second
# LPUSH: 118203.31 requests per second
# RPUSH: 121654.50 requests per second
# RPOPLPUSH: 107758.62 requests per second
# LPOP: 126582.27 requests per second
# RPOP: 129870.12 requests per second
# SADD: 126103.41 requests per second
# HSET: 119047.62 requests per second
# SPOP: 140845.08 requests per second
# ZADD: 121654.50 requests per second
# ZPOPMIN: 140845.08 requests per second
# LRANGE_100: 38910.51 requests per second  ← slower!
# LRANGE_300: 15015.02 requests per second  ← even slower!

# ==================== Pipeline Benchmark ====================
# -P: pipeline depth (ส่ง N commands ก่อนรับ response)
redis-benchmark -c 50 -n 100000 -P 16 -q

# Pipeline=16:
# SET: 847457.63 requests per second (7x faster!)
# GET: 1176470.59 requests per second

# ==================== Specific Commands ====================
# ทดสอบเฉพาะ SET และ GET
redis-benchmark -c 50 -n 100000 -t set,get -d 100

# ==================== Large Data Test ====================
# ทดสอบ 1KB, 10KB, 100KB values
for size in 1000 10000 100000; do
  echo "=== Value size: ${size} bytes ==="
  redis-benchmark -c 50 -n 10000 -d ${size} -t set,get -q
done

# ==================== Benchmark คำสั่ง Expensive ====================
# Test LRANGE
redis-benchmark -c 50 -n 100000 -t lrange_100,lrange_300,lrange_500 -q

# ==================== Connection Overhead ====================
# เพิ่ม clients เพื่อหา sweet spot
for clients in 1 10 50 100 500 1000; do
  echo -n "Clients: ${clients} → "
  redis-benchmark -c ${clients} -n 100000 -t set -q 2>/dev/null | head -1
done

# ==================== Auth Benchmark ====================
redis-benchmark -a your_password -c 50 -n 100000 -t set,get -q
```

### Benchmark Analysis

```bash
# ==================== Detailed Latency Analysis ====================
# --csv: output เป็น CSV
redis-benchmark -c 50 -n 100000 -t set --csv

# ดู latency percentiles
redis-benchmark -c 50 -n 100000 -t set,get

# Sample output (detailed):
# ====== SET ======
#   100000 requests completed in 0.83 seconds
#   50 parallel clients
#   3 bytes payload
#   keep alive: 1
#   host configuration "save": ""
#   host configuration "appendonly": "no"
#   multi-thread: no
# 
# Latency by percentile distribution:
# 0.000% <= 0.199 milliseconds (cumulative count 86)
# 50.000% <= 0.367 milliseconds (cumulative count 50200)
# 75.000% <= 0.399 milliseconds (cumulative count 75000)
# 87.500% <= 0.447 milliseconds (cumulative count 87500)
# 93.750% <= 0.503 milliseconds (cumulative count 93750)
# 97.000% <= 0.647 milliseconds (cumulative count 97000)
# 99.000% <= 0.903 milliseconds (cumulative count 99000)
# 99.500% <= 1.055 milliseconds
# 99.875% <= 1.463 milliseconds
# 100.000% <= 2.407 milliseconds (the worst case)
```

---

## Slow Log Analysis

```bash
# ==================== Slow Log ====================
# บันทึก commands ที่ใช้เวลานานกว่า slowlog-log-slower-than (microseconds)

# ตั้งค่า
redis-cli CONFIG SET slowlog-log-slower-than 10000  # 10ms
redis-cli CONFIG SET slowlog-max-len 128

# ดู slow log
redis-cli SLOWLOG GET 10  # ดู 10 entries ล่าสุด

# Output:
# 1) 1) (integer) 14          ← ID
#    2) (integer) 1621400000  ← Timestamp (Unix)
#    3) (integer) 15234       ← Duration (microseconds)
#    4) 1) "SORT"             ← Command
#       2) "mylist"
#       3) "ALPHA"
#       4) "LIMIT"
#       5) "0"
#       6) "1000"
#    5) "127.0.0.1:51234"    ← Client address
#    6) "myapp"              ← Client name

# ดู slow log count
redis-cli SLOWLOG LEN

# Reset slow log
redis-cli SLOWLOG RESET

# ==================== Parse Slow Log ====================
# Script วิเคราะห์ slow commands
redis-cli SLOWLOG GET 100 | \
  awk '/microseconds/{print $NF}' | \
  sort -n | \
  tail -20

# Automatic slow log monitoring
while true; do
  COUNT=$(redis-cli SLOWLOG LEN)
  if [ "$COUNT" -gt 10 ]; then
    echo "$(date): $COUNT slow commands detected"
    redis-cli SLOWLOG GET 5
    redis-cli SLOWLOG RESET
  fi
  sleep 60
done
```

---

## Latency Monitoring

```bash
# ==================== LATENCY Commands ====================
# Monitor latency spikes

# เปิด latency monitoring
redis-cli CONFIG SET latency-monitor-threshold 100  # 100ms
redis-cli CONFIG SET latency-tracking yes

# ดู latest latency events
redis-cli LATENCY LATEST

# Output:
# 1) 1) "event"           ← event type
#    2) (integer) 1621400000  ← timestamp
#    3) (integer) 245     ← latest latency (ms)
#    4) (integer) 890     ← max latency (ms)

# ดู latency history สำหรับ event
redis-cli LATENCY HISTORY command

# Reset latency data
redis-cli LATENCY RESET

# ==================== COMMAND STATS ====================
# ดู stats ของแต่ละ command
redis-cli COMMAND STATS

# Output:
# get: calls=1000000, usec=500000, usec_per_call=0.50
# set: calls=500000, usec=300000, usec_per_call=0.60
# hgetall: calls=100, usec=50000, usec_per_call=500.00  ← SLOW!

# ==================== Monitor (Real-time) ====================
# ดู commands real-time (WARNING: ลด performance 50%!)
redis-cli MONITOR

# Filter ด้วย grep:
redis-cli MONITOR | grep -E "SET|GET" | head -20

# อย่าใช้ MONITOR ใน production นานๆ!
```

---

## Key Analysis Tools

### redis-cli Analysis Commands

```bash
# ==================== --bigkeys ====================
# หา keys ที่ใหญ่ที่สุด (ใช้ SCAN ไม่ blocking)
redis-cli --bigkeys

# Output:
# Biggest string found 'user:session:abc123' has 45678 bytes
# Biggest list found 'notifications:user:456' has 10234 items
# Biggest set found 'followers:user:789' has 5678 members
# Biggest zset found 'leaderboard:global' has 1000000 members  ← ปัญหา!
# Biggest hash found 'product:catalog:12' has 234 fields

# ==================== --memkeys ====================
# ดู memory usage ของแต่ละ key (scan based)
redis-cli --memkeys --scan

# ==================== --hotkeys ====================
# หา hot keys (ต้องการ maxmemory-policy allkeys-lfu หรือ volatile-lfu)
redis-cli --hotkeys --scan

# ==================== OBJECT ENCODING ====================
# ตรวจสอบ encoding ของ key
redis-cli OBJECT ENCODING mykey
redis-cli OBJECT ENCODING mylist
redis-cli OBJECT ENCODING myhash

# ==================== OBJECT FREQ ====================
# ดู LFU access frequency
redis-cli OBJECT FREQ mykey
# (integer) 12  ← frequency counter

# ==================== OBJECT IDLETIME ====================
# ดูว่า key ไม่ได้ใช้มานานเท่าไหร่ (seconds)
redis-cli OBJECT IDLETIME mykey
# (integer) 3600  ← ไม่ได้ใช้ 1 ชั่วโมง

# ==================== MEMORY USAGE ====================
# ดู memory ที่ key นั้นๆ ใช้
redis-cli MEMORY USAGE mykey
# (integer) 45678  ← bytes

# ดู memory แบบ samples
redis-cli MEMORY USAGE mykey SAMPLES 5
```

### Scan-based Analysis Scripts

```bash
# ==================== Find Large Keys ====================
redis-cli --scan --pattern "*" | \
  xargs -I{} redis-cli MEMORY USAGE {} 2>/dev/null | \
  sort -rn | head -20

# ==================== Find Keys Without TTL ====================
# (อย่าใช้ KEYS * ใน production!)
redis-cli --scan --count 1000 | while read key; do
  TTL=$(redis-cli TTL "$key")
  if [ "$TTL" == "-1" ]; then
    echo "$key (no TTL)"
  fi
done | head -50

# ==================== Find Expired Keys ====================
redis-cli INFO keyspace
# db0:keys=1000000,expires=500000,avg_ttl=86400000

# ==================== Pattern-based Key Count ====================
# Count keys matching pattern
redis-cli --scan --pattern "user:*" | wc -l
redis-cli --scan --pattern "session:*" | wc -l
redis-cli --scan --pattern "cache:*" | wc -l
```

---

## Redis 7.x Improvements

```bash
# ==================== Redis 7.0 Features ====================

# 1. Multi-Part AOF (MP-AOF)
# AOF แบ่งเป็น manifest + multiple segments
# ลดเวลา AOF rewrite และ memory usage

# 2. Listpack ใน all data types
# ทุก compact structure ใช้ listpack แทน ziplist
# ใช้ memory น้อยกว่า

# 3. Function (แทน EVAL scripts)
# Functions persistent ข้าม restarts!
redis-cli FUNCTION LOAD "#!lua name=mylib
  local function myfunc(keys, args)
    return redis.call('SET', keys[1], args[1])
  end
  redis.register_function('myfunc', myfunc)
"

# เรียกใช้:
redis-cli FCALL myfunc 1 mykey myvalue

# 4. ACL Log
redis-cli ACL LOG

# ==================== Redis 7.2 Features ====================

# 1. Sharded Pub/Sub
# Pub/Sub ทำงานบน specific shard ใน Cluster
redis-cli SSUBSCRIBE channel
redis-cli SPUBLISH channel message

# 2. Improved Listpack
# zset-max-listpack-entries เพิ่มเป็น 128 ด้วย default

# ==================== Redis 8.0 Features (Coming Soon) ====================
# - Vector Similarity Search ใน core
# - Better memory efficiency
# - Improved cluster management
```

---

## Full Optimization Checklist

### redis.conf ที่ Optimized

```bash
# /etc/redis/redis.conf
# ====================================
# Optimized Redis Configuration
# Server: 16 CPU, 32GB RAM, SSD
# Use case: Cache + Session storage
# ====================================

# -------- Network --------
bind 127.0.0.1 10.0.0.1
protected-mode yes
port 6379
tcp-backlog 65535
timeout 0
tcp-keepalive 300

# -------- General --------
daemonize yes
supervised systemd
pidfile /var/run/redis/redis-server.pid
loglevel notice
logfile /var/log/redis/redis-server.log
databases 16

# -------- Memory --------
maxmemory 24gb
maxmemory-policy allkeys-lfu
maxmemory-samples 10

# LFU
lfu-log-factor 10
lfu-decay-time 1

# Memory encoding
hash-max-listpack-entries 128
hash-max-listpack-value 64
list-max-listpack-size -2
set-max-intset-entries 512
zset-max-listpack-entries 128
zset-max-listpack-value 64

# Active Defrag
activedefrag yes
active-defrag-ignore-bytes 100mb
active-defrag-enabled yes
active-defrag-threshold-lower 10
active-defrag-threshold-upper 100

# -------- Persistence --------
# RDB (backup)
save 3600 1
save 300 100
save 60 10000
rdbcompression yes
rdbchecksum yes
dbfilename dump.rdb
dir /var/lib/redis

# AOF (durability)
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
aof-use-rdb-preamble yes

# -------- Performance --------
hz 15                           # Background task frequency
dynamic-hz yes                  # Adjust hz ตาม load
aof-rewrite-incremental-fsync yes
rdb-save-incremental-fsync yes
lazyfree-lazy-eviction yes      # Non-blocking eviction
lazyfree-lazy-expire yes        # Non-blocking key expiration
lazyfree-lazy-server-del yes    # Non-blocking DEL
replica-lazy-flush yes          # Non-blocking FLUSHDB on replica

# -------- Slow Log --------
slowlog-log-slower-than 10000   # 10ms
slowlog-max-len 256

# -------- Latency Monitoring --------
latency-monitor-threshold 100   # 100ms
latency-tracking yes

# -------- Client --------
maxclients 10000
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit slave 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60

# -------- Security --------
requirepass your_strong_password_here
rename-command FLUSHALL ""      # Disable dangerous commands
rename-command FLUSHDB ""
rename-command DEBUG ""
rename-command CONFIG "CONFIG_xxxxxxx"  # Rename to random string
```

### System-level Tuning

```bash
# ==================== Linux Kernel Tuning ====================
# /etc/sysctl.conf

# Network
net.core.somaxconn = 65535
net.core.netdev_max_backlog = 65535
net.ipv4.tcp_max_syn_backlog = 65535
net.ipv4.tcp_keepalive_time = 300
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 5

# Memory
vm.overcommit_memory = 1   # ให้ Redis fork() ได้โดยไม่ error
vm.swappiness = 1          # ลด swap ให้น้อยที่สุด
# vm.swappiness = 0 อาจทำให้ OOM killer kill Redis!

# Disable Transparent Huge Pages (THP)
# THP ทำให้ Redis memory ใช้สูงขึ้นและ latency spike
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag

# Apply system settings
sysctl -p

# ==================== /etc/rc.local สำหรับ THP ====================
# เพิ่มใน /etc/rc.local เพื่อให้ทำงานหลัง reboot
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
```

---

## MEMORY DOCTOR และ Diagnostics

```bash
# ==================== MEMORY DOCTOR ====================
redis-cli MEMORY DOCTOR

# Possible outputs:
# "Sam, I detected a few problems with this Redis instance:
#  * WARNING: You are using replication, and based on the total used
#    memory you could have up to 2 replicas using 7.80 GB of memory each.
#  * WARNING: Memory fragmentation is above 1.5 (it is 1.83). This means
#    there is about 2.50 GB of memory fragmentation. To reduce the memory
#    fragmentation run the command MEMORY PURGE."

# ==================== MEMORY MALLOC-STATS ====================
redis-cli MEMORY MALLOC-STATS | head -50

# ==================== DEBUG JMAP ====================
# (ต้องเปิด debug mode) - ดู memory distribution
redis-cli DEBUG JMAP

# ==================== INFO ALL ====================
# ดูทุก stats
redis-cli INFO all

# แยกเป็น sections:
redis-cli INFO server
redis-cli INFO clients
redis-cli INFO memory
redis-cli INFO stats
redis-cli INFO replication
redis-cli INFO cpu
redis-cli INFO commandstats
redis-cli INFO keyspace
redis-cli INFO latencystats
```

---

## สรุป: Redis Performance Checklist

```markdown
## Redis Performance Optimization Checklist

### Memory
- [ ] maxmemory กำหนดแล้ว (75-85% of available RAM)
- [ ] maxmemory-policy เหมาะกับ use case
  - Cache → allkeys-lfu
  - Mixed → volatile-lru
- [ ] activedefrag = yes
- [ ] Memory encoding optimized
  - hash-max-listpack-entries = 128
  - zset-max-listpack-entries = 128
  - set-max-intset-entries = 512

### Persistence
- [ ] appendfsync = everysec (ไม่ใช่ always)
- [ ] aof-use-rdb-preamble = yes
- [ ] no-appendfsync-on-rewrite ตาม use case
- [ ] RDB save frequency เหมาะสม

### Network
- [ ] tcp-backlog = 65535
- [ ] System: net.core.somaxconn = 65535
- [ ] maxclients กำหนดแล้ว
- [ ] tcp-keepalive = 300

### System
- [ ] vm.overcommit_memory = 1
- [ ] vm.swappiness = 1
- [ ] Transparent Huge Pages = never
- [ ] Swap น้อยมากหรือไม่มี

### Monitoring
- [ ] slowlog-log-slower-than = 10000 (10ms)
- [ ] latency-monitor-threshold = 100
- [ ] Regular: MEMORY DOCTOR
- [ ] Regular: SLOWLOG GET
- [ ] redis-benchmark สำหรับ performance test

### Application
- [ ] Pipeline ใช้สำหรับ bulk operations
- [ ] TTL กำหนดทุก cache key
- [ ] ไม่ใช้ KEYS * ใน production
- [ ] Connection pooling ใน application
- [ ] ไม่มี big keys (check ด้วย --bigkeys)
```

---

## Benchmarking Results: Before vs After Tuning

```bash
# ==================== Before Tuning ====================
# Default config, no optimization

redis-benchmark -c 50 -n 100000 -t set,get -q
# SET: 85,000 requests per second
# GET: 95,000 requests per second
# P99 latency SET: 2.5ms

# ==================== After Tuning ====================
# Optimized config, system tuning, pipeline

redis-benchmark -c 50 -n 100000 -t set,get -q
# SET: 145,000 requests per second (+70%)
# GET: 160,000 requests per second (+68%)
# P99 latency SET: 0.9ms (-64%)

redis-benchmark -c 50 -n 100000 -t set,get -P 16 -q
# SET: 950,000 requests per second (with pipeline)
# GET: 1,200,000 requests per second (with pipeline)
# P99 latency SET: 0.15ms (pipeline)

# Key improvements:
# 1. TCP backlog tuning: -10ms latency spike during high load
# 2. Disable THP: -30% latency variance
# 3. vm.overcommit_memory: ไม่มี BGSAVE failures
# 4. allkeys-lfu: better cache efficiency
# 5. activedefrag: stable memory usage
```

---

*Part 60 เสร็จสมบูรณ์ - จบ Part 51-60: Advanced Database Cluster Course*

---

## สรุป Part 51-60: Advanced Level

ในส่วนนี้เราได้เรียนรู้:

- **Part 56**: Distributed Tracing ด้วย OpenTelemetry, Jaeger, Zipkin
- **Part 57**: Metrics Collection ด้วย Prometheus + Grafana
- **Part 58**: Alerting, Incident Response, SLA/SLO Management
- **Part 59**: PostgreSQL Performance Tuning ครบถ้วน
- **Part 60**: Redis Performance Tuning ครบถ้วน

ทั้งหมดนี้รวมกันเป็น **Observability Stack** ที่สมบูรณ์สำหรับ Production Database Cluster

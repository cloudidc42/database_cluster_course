# Part 09: Redis Data Structures: String, Hash, List, Set, ZSet

## สารบัญ

1. [String: คำสั่งและ Use Cases](#1-string-คำสั่งและ-use-cases)
2. [Hash: คำสั่งและ Use Cases](#2-hash-คำสั่งและ-use-cases)
3. [List: คำสั่งและ Use Cases](#3-list-คำสั่งและ-use-cases)
4. [Set: คำสั่งและ Use Cases](#4-set-คำสั่งและ-use-cases)
5. [Sorted Set (ZSet): คำสั่งและ Use Cases](#5-sorted-set-zset-คำสั่งและ-use-cases)
6. [HyperLogLog: Unique Counting](#6-hyperloglog-unique-counting)
7. [Bitmap: Boolean Flags](#7-bitmap-boolean-flags)
8. [Geo: Location-based Features](#8-geo-location-based-features)
9. [Stream: Event Logging](#9-stream-event-logging)
10. [Workshop: Leaderboard + Activity Feed + Unique Visitors](#10-workshop-leaderboard--activity-feed--unique-visitors)

---

## 1. String: คำสั่งและ Use Cases

String เป็น data type พื้นฐานที่สุดใน Redis สามารถเก็บ text, integers, floats, หรือ binary data (สูงสุด 512 MB)

### 1.1 คำสั่ง String ทั้งหมด

```bash
# ========================================
# Basic GET/SET
# ========================================

# SET: กำหนดค่า
SET key "Hello Redis"
SET greeting "สวัสดี"
SET count 42
SET price 19.99

# GET: ดึงค่า
GET key            # "Hello Redis"
GET nonexistent    # (nil)

# GETSET: ดึงค่าเดิมและกำหนดค่าใหม่ (deprecated ใน Redis 6.2, ใช้ GETEX/SET แทน)
GETSET counter 0   # คืนค่าเดิม, set เป็น 0

# GETEX: GET พร้อมตั้ง expiry
GETEX key EX 3600       # GET และตั้ง TTL 1 ชั่วโมง
GETEX key PX 60000      # GET และตั้ง TTL 60 วินาที (ms)
GETEX key EXAT 1735657200  # GET และตั้ง expire time
GETEX key PERSIST          # GET และลบ TTL

# GETDEL: GET แล้วลบ key
GETDEL temp:token          # คืนค่าแล้วลบ key ทันที

# ========================================
# Multiple Keys
# ========================================

# MSET/MGET: หลาย keys พร้อมกัน
MSET user:1:name "Alice" user:1:email "alice@example.com" user:1:age "30"
MGET user:1:name user:1:email user:1:age
# 1) "Alice"
# 2) "alice@example.com"
# 3) "30"

# MSETNX: Set หลาย keys ถ้าทุก key ไม่มีอยู่ (atomic)
MSETNX session:1 "data1" session:2 "data2"
# 1 ถ้าสำเร็จทั้งหมด, 0 ถ้า key ใด key หนึ่งมีอยู่แล้ว

# ========================================
# String Manipulation
# ========================================

# APPEND: ต่อท้าย string
SET log ""
APPEND log "2024-01-01: Server started\n"
APPEND log "2024-01-01: User logged in\n"
GET log
# "2024-01-01: Server started\n2024-01-01: User logged in\n"

# STRLEN: ความยาว string
STRLEN greeting    # จำนวน bytes (ไม่ใช่ characters!)

# GETRANGE: ดึง substring
SET sentence "Hello World Redis"
GETRANGE sentence 0 4      # "Hello"
GETRANGE sentence 6 10     # "World"
GETRANGE sentence -5 -1    # "Redis" (ติดลบ = นับจากท้าย)
GETRANGE sentence 0 -1     # "Hello World Redis" (ทั้งหมด)

# SETRANGE: แก้ไข string บางส่วน
SET name "Hello World"
SETRANGE name 6 "Redis"    # "Hello Redis"
GET name                   # "Hello Redis"

# ========================================
# Numeric Operations (Atomic)
# ========================================

SET visits 0
INCR visits        # 1
INCR visits        # 2
INCRBY visits 10   # 12
DECRBY visits 3    # 9
DECR visits        # 8

SET balance 100.50
INCRBYFLOAT balance 10.25   # 110.75
INCRBYFLOAT balance -5.00   # 105.75

# ========================================
# Conditional SET
# ========================================

# SET NX: Set ถ้า key ไม่มีอยู่
SET lock:resource "worker1" NX EX 30
# OK ถ้าสำเร็จ, nil ถ้า key มีอยู่แล้ว

# SET XX: Set ถ้า key มีอยู่แล้ว
SET existing:key "newvalue" XX

# SET GET: คืนค่าเดิมพร้อม SET ค่าใหม่
SET mykey "newvalue" GET
```

### 1.2 JSON in Strings

```javascript
// ioredis: JSON in Redis Strings
const redis = require('ioredis');
const client = new redis();

// เก็บ JSON object
const user = {
  id: 123,
  name: "Alice Johnson",
  email: "alice@example.com",
  roles: ["admin", "user"],
  settings: { theme: "dark", language: "th" }
};

// Set JSON
await client.set('user:123', JSON.stringify(user), 'EX', 3600);

// Get JSON
const data = await client.get('user:123');
const parsedUser = JSON.parse(data);
console.log(parsedUser.name);  // Alice Johnson

// Update ส่วนหนึ่งของ JSON (ต้องดึง-แก้-บันทึก)
const current = JSON.parse(await client.get('user:123'));
current.settings.theme = 'light';
await client.set('user:123', JSON.stringify(current), 'EX', 3600);

// ทางเลือกที่ดีกว่า: ใช้ RedisJSON module (ถ้ามี)
// หรือใช้ Hash แทนสำหรับ object ที่มีหลาย fields
```

### 1.3 Use Cases: Simple Values, Counters, Sessions

```javascript
// Use Case 1: Simple Configuration Values
await client.set('config:max_upload_size', '10485760');  // 10 MB in bytes
await client.set('config:maintenance_mode', 'false');
await client.set('config:version', '2.1.0');

// Use Case 2: Counters
async function incrementPageView(pageId) {
  const key = `stats:page:${pageId}:views:${new Date().toISOString().split('T')[0]}`;
  const count = await client.incr(key);
  await client.expire(key, 86400 * 30);  // keep 30 days
  return count;
}

// Use Case 3: Session Storage
async function createSession(userId, sessionData) {
  const sessionId = generateUUID();
  const key = `session:${sessionId}`;
  await client.setex(key, 3600, JSON.stringify({
    userId,
    createdAt: Date.now(),
    ...sessionData
  }));
  return sessionId;
}

// Use Case 4: Temporary Tokens (Password Reset, Email Verify)
async function createPasswordResetToken(userId) {
  const token = generateSecureToken();
  const key = `reset:token:${token}`;
  await client.setex(key, 900, userId.toString());  // 15 minutes
  return token;
}

async function verifyResetToken(token) {
  const key = `reset:token:${token}`;
  const userId = await client.getdel(key);  // atomic get + delete
  return userId ? parseInt(userId) : null;
}

// Use Case 5: Feature Flags
async function isFeatureEnabled(featureName, userId) {
  // Global flag
  const globalFlag = await client.get(`feature:${featureName}:enabled`);
  if (globalFlag === '1') return true;
  if (globalFlag === '0') return false;
  
  // Per-user override
  const userFlag = await client.get(`feature:${featureName}:user:${userId}`);
  return userFlag === '1';
}
```

---

## 2. Hash: คำสั่งและ Use Cases

Hash เหมาะสำหรับเก็บ object ที่มีหลาย fields - ประหยัด memory กว่าการเก็บแต่ละ field เป็น key แยกๆ

### 2.1 คำสั่ง Hash ทั้งหมด

```bash
# ========================================
# Basic Hash Commands
# ========================================

# HSET: Set field(s)
HSET user:123 name "Alice" email "alice@example.com" age 30
HSET user:123 city "Bangkok"  # เพิ่ม/อัพเดท field เดียว

# HGET: Get field value
HGET user:123 name        # "Alice"
HGET user:123 nonexistent # (nil)

# HMSET: Set หลาย fields (deprecated ใน Redis 4.0, ใช้ HSET แทน)
HMSET user:124 name "Bob" email "bob@example.com"

# HMGET: Get หลาย fields
HMGET user:123 name email age
# 1) "Alice"
# 2) "alice@example.com"
# 3) "30"

# HGETALL: Get ทุก fields และ values
HGETALL user:123
# 1) "name"
# 2) "Alice"
# 3) "email"
# 4) "alice@example.com"
# 5) "age"
# 6) "30"
# 7) "city"
# 8) "Bangkok"

# HKEYS: ดูทุก field names
HKEYS user:123   # name, email, age, city

# HVALS: ดูทุก values
HVALS user:123   # Alice, alice@example.com, 30, Bangkok

# HLEN: จำนวน fields
HLEN user:123    # 4

# HEXISTS: ตรวจสอบว่า field มีอยู่ไหม
HEXISTS user:123 name    # 1
HEXISTS user:123 phone   # 0

# HDEL: ลบ field(s)
HDEL user:123 city
HDEL user:123 field1 field2 field3  # ลบหลาย fields

# ========================================
# Numeric Operations
# ========================================

# HINCRBY: เพิ่ม integer field
HSET stats:page page_views 0 unique_visitors 0
HINCRBY stats:page page_views 1   # 1
HINCRBY stats:page page_views 5   # 6
HINCRBY stats:page page_views -2  # 4

# HINCRBYFLOAT: เพิ่ม float field
HSET product:1 price 99.99 rating 4.0
HINCRBYFLOAT product:1 rating 0.5   # 4.5
HINCRBYFLOAT product:1 price -10.0  # 89.99

# ========================================
# Conditional SET
# ========================================

# HSETNX: Set ถ้า field ไม่มีอยู่
HSETNX user:123 created_at "2024-01-01"  # 1 ถ้าสำเร็จ, 0 ถ้ามีอยู่แล้ว

# ========================================
# HSCAN: Iterate Hash fields
# ========================================
HSCAN user:123 0
HSCAN largehash 0 MATCH field* COUNT 50
```

### 2.2 Hash vs String JSON

```javascript
// เปรียบเทียบ Hash vs String JSON

// ========================================
// String JSON approach
// ========================================
await client.set('user:json:123', JSON.stringify({
  name: 'Alice',
  email: 'alice@example.com',
  age: 30,
  city: 'Bangkok'
}));

// อัพเดท 1 field → ต้อง GET ทั้ง object, แก้ไข, SET ทั้ง object
const userData = JSON.parse(await client.get('user:json:123'));
userData.age = 31;
await client.set('user:json:123', JSON.stringify(userData));

// ข้อเสีย:
// - ต้อง deserialize ทั้ง object แม้ต้องการแค่ 1 field
// - Race condition ถ้าหลาย clients update พร้อมกัน
// - ยากที่จะ partial update

// ========================================
// Hash approach
// ========================================
await client.hset('user:hash:123', 
  'name', 'Alice',
  'email', 'alice@example.com',
  'age', '30',
  'city', 'Bangkok'
);

// อัพเดท 1 field → ตรงๆ เลย!
await client.hincrby('user:hash:123', 'age', 1);  // atomic increment

// ดึง 1 field โดยไม่ต้องดึงทั้งหมด
const name = await client.hget('user:hash:123', 'name');

// ดึงหลาย fields ที่ต้องการ
const [n, email] = await client.hmget('user:hash:123', 'name', 'email');

// ข้อดี Hash:
// + อัพเดท field เดียวโดยไม่กระทบ fields อื่น
// + ดึงเฉพาะ fields ที่ต้องการ
// + Atomic numeric increment
// + ประหยัด memory มากกว่า (small hash ใช้ ziplist encoding)

// ข้อเสีย Hash:
// - ไม่รองรับ nested objects
// - ต้องแปลง values เป็น string
```

### 2.3 Use Cases: User Profiles, Configurations

```javascript
// Use Case 1: User Profile
class UserProfile {
  constructor(redis) {
    this.redis = redis;
  }
  
  key(userId) {
    return `user:${userId}:profile`;
  }
  
  async create(userId, data) {
    await this.redis.hset(this.key(userId),
      'id', userId,
      'name', data.name,
      'email', data.email,
      'created_at', Date.now().toString(),
      'post_count', '0',
      'follower_count', '0'
    );
  }
  
  async get(userId) {
    return this.redis.hgetall(this.key(userId));
  }
  
  async getField(userId, field) {
    return this.redis.hget(this.key(userId), field);
  }
  
  async update(userId, updates) {
    const args = [];
    for (const [field, value] of Object.entries(updates)) {
      args.push(field, value.toString());
    }
    return this.redis.hset(this.key(userId), ...args);
  }
  
  async incrementStat(userId, stat) {
    return this.redis.hincrby(this.key(userId), stat, 1);
  }
  
  async getMultiple(userIds) {
    const pipeline = this.redis.pipeline();
    for (const id of userIds) {
      pipeline.hgetall(this.key(id));
    }
    const results = await pipeline.exec();
    return results.map(([err, data]) => err ? null : data);
  }
}

// Use Case 2: Shopping Cart (Hash สำหรับ cart items)
class ShoppingCart {
  constructor(redis) {
    this.redis = redis;
  }
  
  key(cartId) {
    return `cart:${cartId}`;
  }
  
  async addItem(cartId, productId, quantity) {
    // HINCRBY: เพิ่มหรือตั้งค่าเริ่มต้น
    const newQty = await this.redis.hincrby(
      this.key(cartId), `product:${productId}`, quantity
    );
    await this.redis.expire(this.key(cartId), 86400 * 7);  // 7 days TTL
    return newQty;
  }
  
  async removeItem(cartId, productId) {
    return this.redis.hdel(this.key(cartId), `product:${productId}`);
  }
  
  async getCart(cartId) {
    const items = await this.redis.hgetall(this.key(cartId));
    if (!items) return {};
    
    // แปลง "product:123": "2" เป็น { productId: 123, quantity: 2 }
    return Object.entries(items).map(([key, qty]) => ({
      productId: parseInt(key.replace('product:', '')),
      quantity: parseInt(qty)
    }));
  }
  
  async getItemCount(cartId) {
    return this.redis.hlen(this.key(cartId));
  }
  
  async clearCart(cartId) {
    return this.redis.del(this.key(cartId));
  }
}

// Use Case 3: Application Config
class AppConfig {
  constructor(redis) {
    this.redis = redis;
    this.cacheKey = 'app:config';
  }
  
  async set(key, value) {
    await this.redis.hset(this.cacheKey, key, String(value));
  }
  
  async get(key, defaultValue = null) {
    const value = await this.redis.hget(this.cacheKey, key);
    return value ?? defaultValue;
  }
  
  async getAll() {
    return this.redis.hgetall(this.cacheKey);
  }
  
  async setMany(config) {
    const args = Object.entries(config).flat().map(String);
    return this.redis.hset(this.cacheKey, ...args);
  }
}
```

---

## 3. List: คำสั่งและ Use Cases

List เป็น ordered collection ของ strings (Doubly Linked List ภายใน) เหมาะสำหรับ queues, stacks, activity feeds

### 3.1 คำสั่ง List ทั้งหมด

```bash
# ========================================
# Push Operations
# ========================================

# LPUSH: Push ทางซ้าย (head)
LPUSH mylist "c"
LPUSH mylist "b" "a"   # push a แล้ว b แล้ว c → list: [a, b, c]
# ลำดับ: จาก right ไป left ตาม arguments

# RPUSH: Push ทางขวา (tail)
RPUSH mylist "d" "e"   # list: [a, b, c, d, e]

# LPUSHX: Push ซ้ายถ้า list มีอยู่แล้ว
LPUSHX existinglist "new"   # 0 ถ้า list ไม่มี

# RPUSHX: Push ขวาถ้า list มีอยู่แล้ว
RPUSHX existinglist "new"

# ========================================
# Pop Operations
# ========================================

# LPOP: Pop ทางซ้าย (dequeue จาก head)
LPOP mylist         # "a"
LPOP mylist 2       # ["b", "c"] (Redis 6.2+ รองรับ count)

# RPOP: Pop ทางขวา (stack pop)
RPOP mylist         # "e"
RPOP mylist 2       # ["d", "c"]

# BLPOP: Blocking LPOP (รอจนมีข้อมูลหรือ timeout)
BLPOP queue:jobs 0          # รอไม่มีกำหนด
BLPOP queue:jobs queue:low 30  # รอ job จาก 2 queues, timeout 30s

# BRPOP: Blocking RPOP
BRPOP stack:tasks 10    # timeout 10 วินาที

# LMPOP: Pop จากหลาย lists (Redis 7.0+)
LMPOP 2 list1 list2 LEFT COUNT 5

# ========================================
# Read Operations
# ========================================

# LRANGE: ดู elements ในช่วง
LRANGE mylist 0 -1      # ทั้งหมด
LRANGE mylist 0 4       # 5 elements แรก
LRANGE mylist -3 -1     # 3 elements สุดท้าย
LRANGE mylist 5 10      # elements ที่ 6-11

# LINDEX: ดู element ที่ index
LINDEX mylist 0    # element แรก
LINDEX mylist -1   # element สุดท้าย

# LLEN: ความยาว list
LLEN mylist

# ========================================
# Modify Operations
# ========================================

# LSET: เปลี่ยนค่าที่ index
LSET mylist 2 "new_value"

# LINSERT: Insert ก่อน/หลัง element
LINSERT mylist BEFORE "b" "x"   # Insert "x" ก่อน "b"
LINSERT mylist AFTER "c" "y"    # Insert "y" หลัง "c"

# LREM: ลบ element ตาม value
# LREM key count value
# count > 0: ลบจากหัว count ครั้ง
# count < 0: ลบจากท้าย |count| ครั้ง
# count = 0: ลบทั้งหมด
LREM mylist 2 "duplicate"     # ลบ "duplicate" 2 ตัวแรก
LREM mylist 0 "remove_all"    # ลบ "remove_all" ทั้งหมด

# LTRIM: ตัด list เหลือแค่ช่วงที่กำหนด
LTRIM recent:activity 0 99    # เก็บแค่ 100 items ล่าสุด

# LMOVE: Move element ระหว่าง lists (Redis 6.2+)
LMOVE source destination LEFT RIGHT
LMOVE queue:pending queue:processing LEFT LEFT  # dequeue แล้ว enqueue ใน pending queue อื่น

# BRPOPLPUSH (deprecated → ใช้ BLMOVE)
# BLMOVE: Blocking LMOVE
BLMOVE source destination LEFT RIGHT 30
```

### 3.2 Use Cases: Message Queue, Activity Feed

```javascript
// Use Case 1: Task Queue (Producer-Consumer Pattern)
class TaskQueue {
  constructor(redis, queueName) {
    this.redis = redis;
    this.queueName = `queue:${queueName}`;
  }
  
  async enqueue(task) {
    const taskData = JSON.stringify({
      id: generateId(),
      data: task,
      enqueuedAt: Date.now(),
      attempts: 0
    });
    await this.redis.rpush(this.queueName, taskData);
    return taskData;
  }
  
  async dequeue(timeout = 30) {
    // Blocking pop: รอจนมี task หรือ timeout
    const result = await this.redis.blpop(this.queueName, timeout);
    if (!result) return null;
    return JSON.parse(result[1]);  // result = [queueName, value]
  }
  
  async size() {
    return this.redis.llen(this.queueName);
  }
  
  async peek(count = 5) {
    const items = await this.redis.lrange(this.queueName, 0, count - 1);
    return items.map(item => JSON.parse(item));
  }
}

// Worker
class Worker {
  constructor(queue, handler) {
    this.queue = queue;
    this.handler = handler;
    this.running = false;
  }
  
  async start() {
    this.running = true;
    console.log('Worker started');
    
    while (this.running) {
      try {
        const task = await this.queue.dequeue(5);  // timeout 5s
        if (task) {
          console.log(`Processing task ${task.id}`);
          await this.handler(task);
          console.log(`Task ${task.id} completed`);
        }
      } catch (error) {
        console.error('Worker error:', error);
        await new Promise(r => setTimeout(r, 1000));  // pause before retry
      }
    }
  }
  
  stop() {
    this.running = false;
  }
}

// Use Case 2: Activity Feed (Recent Items)
class ActivityFeed {
  constructor(redis) {
    this.redis = redis;
    this.maxItems = 100;
  }
  
  feedKey(userId) {
    return `feed:user:${userId}`;
  }
  
  async addActivity(userId, activity) {
    const key = this.feedKey(userId);
    const item = JSON.stringify({
      ...activity,
      timestamp: Date.now()
    });
    
    // Push ด้านซ้าย (ล่าสุดก่อน)
    await this.redis.lpush(key, item);
    
    // จำกัดจำนวน items
    await this.redis.ltrim(key, 0, this.maxItems - 1);
  }
  
  async getFeed(userId, page = 1, pageSize = 20) {
    const start = (page - 1) * pageSize;
    const end = start + pageSize - 1;
    
    const items = await this.redis.lrange(this.feedKey(userId), start, end);
    return items.map(item => JSON.parse(item));
  }
  
  async getFeedSize(userId) {
    return this.redis.llen(this.feedKey(userId));
  }
}

// Use Case 3: Recent Search History
class SearchHistory {
  constructor(redis) {
    this.redis = redis;
  }
  
  historyKey(userId) {
    return `search:history:${userId}`;
  }
  
  async addSearch(userId, query) {
    const key = this.historyKey(userId);
    
    // ลบ query เดิมถ้ามีอยู่แล้ว (เพื่อ move ไปด้านบน)
    await this.redis.lrem(key, 0, query);
    
    // เพิ่มที่ด้านบน
    await this.redis.lpush(key, query);
    
    // เก็บแค่ 10 items ล่าสุด
    await this.redis.ltrim(key, 0, 9);
  }
  
  async getHistory(userId) {
    return this.redis.lrange(this.historyKey(userId), 0, -1);
  }
  
  async clearHistory(userId) {
    return this.redis.del(this.historyKey(userId));
  }
}

// Use Case 4: Reliable Queue (ป้องกัน message loss)
class ReliableQueue {
  constructor(redis) {
    this.redis = redis;
    this.pendingQueue = 'queue:pending';
    this.processingQueue = 'queue:processing';
  }
  
  async enqueue(task) {
    await this.redis.rpush(this.pendingQueue, JSON.stringify(task));
  }
  
  async dequeue() {
    // Move message จาก pending ไป processing (atomic)
    const data = await this.redis.lmove(
      this.pendingQueue, this.processingQueue, 'LEFT', 'RIGHT'
    );
    return data ? JSON.parse(data) : null;
  }
  
  async acknowledge(taskData) {
    // ลบออกจาก processing queue เมื่อทำเสร็จ
    await this.redis.lrem(this.processingQueue, 1, JSON.stringify(taskData));
  }
  
  async requeueStuck(maxAgeMs = 300000) {
    // ย้าย stuck tasks กลับ pending (สำหรับ recovery)
    const items = await this.redis.lrange(this.processingQueue, 0, -1);
    
    for (const item of items) {
      const task = JSON.parse(item);
      if (Date.now() - task.startedAt > maxAgeMs) {
        await this.redis.lrem(this.processingQueue, 1, item);
        await this.redis.lpush(this.pendingQueue, item);
      }
    }
  }
}
```

---

## 4. Set: คำสั่งและ Use Cases

Set คือ unordered collection ของ unique strings ไม่มี duplicate เหมาะสำหรับ tags, unique items, relationships

### 4.1 คำสั่ง Set ทั้งหมด

```bash
# ========================================
# Add/Remove
# ========================================

# SADD: เพิ่ม member(s)
SADD tags:post:1 "redis" "database" "nosql"
SADD online:users 123 456 789

# SREM: ลบ member(s)
SREM tags:post:1 "nosql"
SREM online:users 456 789

# ========================================
# Query
# ========================================

# SMEMBERS: ดูทุก members
SMEMBERS tags:post:1        # "redis" "database"

# SISMEMBER: ตรวจสอบว่า member มีอยู่ไหม
SISMEMBER tags:post:1 "redis"       # 1
SISMEMBER tags:post:1 "python"      # 0

# SMISMEMBER: ตรวจสอบหลาย members (Redis 6.2+)
SMISMEMBER tags:post:1 "redis" "python" "database"
# 1) 1, 2) 0, 3) 1

# SCARD: จำนวน members
SCARD tags:post:1    # 2

# SRANDMEMBER: ดึง random member
SRANDMEMBER tags:post:1        # random 1 member
SRANDMEMBER tags:post:1 3      # random 3 members (ไม่ซ้ำ)
SRANDMEMBER tags:post:1 -3     # random 3 members (อาจซ้ำ)

# SPOP: Pop random member
SPOP tags:post:1        # ดึงและลบ 1 random member
SPOP tags:post:1 2      # ดึงและลบ 2 random members

# ========================================
# Set Operations
# ========================================

# SUNION: Union (รวมทุก elements จากทุก sets)
SADD set1 "a" "b" "c"
SADD set2 "b" "c" "d"
SUNION set1 set2    # "a" "b" "c" "d"

# SINTER: Intersection (เฉพาะที่อยู่ในทุก sets)
SINTER set1 set2    # "b" "c"

# SDIFF: Difference (ใน set1 แต่ไม่ใน set2)
SDIFF set1 set2     # "a"
SDIFF set2 set1     # "d"

# Store results ใน destination set
SUNIONSTORE result set1 set2    # เก็บ union ใน "result"
SINTERSTORE result set1 set2    # เก็บ intersection ใน "result"
SDIFFSTORE result set1 set2     # เก็บ difference ใน "result"

# SMOVE: Move member จาก source ไป destination
SMOVE source destination "member"

# ========================================
# SSCAN: Iterate large sets
# ========================================
SSCAN myset 0 MATCH user:* COUNT 50
```

### 4.2 Use Cases: Tags, Unique Visitors, Friends

```javascript
// Use Case 1: Tag System
class TagSystem {
  constructor(redis) {
    this.redis = redis;
  }
  
  postTagsKey(postId) {
    return `tags:post:${postId}`;
  }
  
  tagPostsKey(tagName) {
    return `tag:${tagName}:posts`;
  }
  
  async addTags(postId, tags) {
    const pipeline = this.redis.pipeline();
    
    // เพิ่ม tags ให้ post
    if (tags.length > 0) {
      pipeline.sadd(this.postTagsKey(postId), ...tags);
    }
    
    // เพิ่ม post ให้แต่ละ tag
    for (const tag of tags) {
      pipeline.sadd(this.tagPostsKey(tag), postId.toString());
    }
    
    await pipeline.exec();
  }
  
  async getPostTags(postId) {
    return this.redis.smembers(this.postTagsKey(postId));
  }
  
  async getPostsByTag(tagName) {
    const postIds = await this.redis.smembers(this.tagPostsKey(tagName));
    return postIds.map(id => parseInt(id));
  }
  
  async getPostsByMultipleTags(tags, operator = 'AND') {
    const keys = tags.map(tag => this.tagPostsKey(tag));
    
    if (operator === 'AND') {
      // Posts ที่มีทุก tags (intersection)
      const postIds = await this.redis.sinter(...keys);
      return postIds.map(id => parseInt(id));
    } else {
      // Posts ที่มีอย่างน้อย 1 tag (union)
      const postIds = await this.redis.sunion(...keys);
      return postIds.map(id => parseInt(id));
    }
  }
  
  async removeTag(postId, tagName) {
    const pipeline = this.redis.pipeline();
    pipeline.srem(this.postTagsKey(postId), tagName);
    pipeline.srem(this.tagPostsKey(tagName), postId.toString());
    await pipeline.exec();
  }
}

// Use Case 2: Online Users
class OnlineUserTracker {
  constructor(redis) {
    this.redis = redis;
    this.onlineKey = 'users:online';
  }
  
  async userOnline(userId) {
    await this.redis.sadd(this.onlineKey, userId.toString());
  }
  
  async userOffline(userId) {
    await this.redis.srem(this.onlineKey, userId.toString());
  }
  
  async isOnline(userId) {
    return !!(await this.redis.sismember(this.onlineKey, userId.toString()));
  }
  
  async getOnlineCount() {
    return this.redis.scard(this.onlineKey);
  }
  
  async getOnlineUsers() {
    const userIds = await this.redis.smembers(this.onlineKey);
    return userIds.map(id => parseInt(id));
  }
  
  async getMutualOnlineFriends(userId, friendIds) {
    // สร้าง temporary set ของ friendIds
    const tempKey = `temp:friends:${userId}`;
    const pipeline = this.redis.pipeline();
    
    if (friendIds.length > 0) {
      pipeline.sadd(tempKey, ...friendIds.map(String));
    }
    pipeline.expire(tempKey, 5);  // auto-delete after 5s
    await pipeline.exec();
    
    // Intersection กับ online users
    const mutualOnline = await this.redis.sinter(this.onlineKey, tempKey);
    return mutualOnline.map(id => parseInt(id));
  }
}

// Use Case 3: Lottery / Random Selection
class Lottery {
  constructor(redis) {
    this.redis = redis;
  }
  
  async addParticipants(eventId, userIds) {
    const key = `lottery:${eventId}`;
    await this.redis.sadd(key, ...userIds.map(String));
  }
  
  async drawWinners(eventId, count = 1) {
    const key = `lottery:${eventId}`;
    const winners = await this.redis.spop(key, count);  // random + remove
    return winners.map(id => parseInt(id));
  }
  
  async getParticipantCount(eventId) {
    return this.redis.scard(`lottery:${eventId}`);
  }
}
```

---

## 5. Sorted Set (ZSet): คำสั่งและ Use Cases

Sorted Set คล้าย Set แต่แต่ละ member มี **score** ที่ใช้ sort - เหมาะสำหรับ leaderboards, priority queues, time-series

### 5.1 คำสั่ง Sorted Set ทั้งหมด

```bash
# ========================================
# Add/Update
# ========================================

# ZADD: เพิ่ม/อัพเดท member พร้อม score
ZADD leaderboard 9500 "user:1"
ZADD leaderboard 8750 "user:2" 7200 "user:3"

# ZADD options:
# NX: เพิ่มเฉพาะ member ใหม่ (ไม่อัพเดทที่มีอยู่)
# XX: อัพเดทเฉพาะที่มีอยู่ (ไม่เพิ่มใหม่)
# GT: อัพเดทถ้า score ใหม่ > score เดิม
# LT: อัพเดทถ้า score ใหม่ < score เดิม
# CH: คืนจำนวน elements ที่เปลี่ยน (แทนที่จะคืนจำนวนที่เพิ่ม)
ZADD leaderboard NX 9500 "user:4"   # เพิ่มถ้าไม่มี
ZADD leaderboard GT 9600 "user:1"   # อัพเดทถ้า 9600 > score เดิม

# ZINCRBY: เพิ่ม score
ZINCRBY leaderboard 100 "user:1"    # เพิ่ม 100 คะแนน
ZINCRBY leaderboard -50 "user:2"    # ลด 50 คะแนน

# ========================================
# Get by Rank (Index)
# ========================================

# ZRANGE: ดู members ตาม rank (Redis 6.2+ มี REV, BYSCORE, BYLEX options)
ZRANGE leaderboard 0 -1              # ทั้งหมด ascending
ZRANGE leaderboard 0 -1 WITHSCORES  # พร้อม scores
ZRANGE leaderboard 0 -1 REV         # descending
ZRANGE leaderboard 0 9 REV WITHSCORES  # Top 10 (highest first)

# ZREVRANGE: ดูแบบ reverse (deprecated ใน Redis 6.2)
ZREVRANGE leaderboard 0 9 WITHSCORES   # Top 10

# ========================================
# Get by Score
# ========================================

# ZRANGEBYSCORE: ดู members ในช่วง score
ZRANGEBYSCORE leaderboard 5000 10000              # score ระหว่าง 5000-10000
ZRANGEBYSCORE leaderboard -inf +inf               # ทั้งหมด
ZRANGEBYSCORE leaderboard "(5000" 10000           # exclusive lower bound
ZRANGEBYSCORE leaderboard 5000 10000 WITHSCORES
ZRANGEBYSCORE leaderboard 5000 10000 LIMIT 0 10  # pagination

# ZRANGEBYLEX: ดู members ในช่วง lexicographic (score ต้องเท่ากันทั้งหมด)
ZADD words 0 "apple" 0 "banana" 0 "cherry" 0 "date"
ZRANGEBYLEX words "[b" "[d"    # "banana", "cherry"
ZRANGEBYLEX words "-" "+"      # ทั้งหมด

# ZRANGEBYSCORE (deprecated) → ใช้ ZRANGE BYSCORE แทน
ZRANGE leaderboard 5000 10000 BYSCORE WITHSCORES

# ========================================
# Rank and Score
# ========================================

# ZRANK: หา rank ของ member (ascending, 0-indexed)
ZRANK leaderboard "user:1"       # rank จาก score ต่ำ → สูง

# ZREVRANK: rank แบบ reverse (descending)
ZREVRANK leaderboard "user:1"    # rank จาก score สูง → ต่ำ (Top = 0)

# ZSCORE: ดู score
ZSCORE leaderboard "user:1"      # 9600

# ZMSCORE: ดู scores หลาย members
ZMSCORE leaderboard "user:1" "user:2" "user:3"

# ========================================
# Count
# ========================================

# ZCARD: จำนวน members ทั้งหมด
ZCARD leaderboard

# ZCOUNT: นับ members ในช่วง score
ZCOUNT leaderboard 5000 10000
ZCOUNT leaderboard -inf +inf

# ZLEXCOUNT: นับ members ในช่วง lex
ZLEXCOUNT words "[b" "[d"

# ========================================
# Remove
# ========================================

# ZREM: ลบ member(s)
ZREM leaderboard "user:5"
ZREM leaderboard "user:6" "user:7"

# ZREMRANGEBYRANK: ลบ members ในช่วง rank
ZREMRANGEBYRANK leaderboard 0 9    # ลบ 10 อันดับต่ำสุด

# ZREMRANGEBYSCORE: ลบ members ในช่วง score
ZREMRANGEBYSCORE leaderboard -inf 1000    # ลบ score < 1000

# ZPOPMIN/ZPOPMAX: Pop member ที่ score ต่ำ/สูงสุด
ZPOPMIN tasks 1      # pop 1 task ที่ priority ต่ำสุด
ZPOPMAX tasks 3      # pop 3 tasks ที่ priority สูงสุด

# BZPOPMIN/BZPOPMAX: Blocking pop
BZPOPMIN tasks 0     # รอไม่มีกำหนด
BZPOPMAX tasks 30    # timeout 30 วินาที

# ========================================
# Set Operations
# ========================================

# ZUNIONSTORE, ZINTERSTORE, ZDIFFSTORE
ZUNIONSTORE dest 2 zset1 zset2
ZINTERSTORE dest 2 zset1 zset2 WEIGHTS 2 1  # weight scores ก่อน combine
ZDIFFSTORE dest 2 zset1 zset2

# ZUNION, ZINTER, ZDIFF (ไม่ store, Redis 6.2+)
ZUNION 2 zset1 zset2 WITHSCORES
ZINTER 2 zset1 zset2 WITHSCORES
```

### 5.2 Use Cases: Leaderboards, Priority Queues, Time-series

```javascript
// Use Case 1: Game Leaderboard
class Leaderboard {
  constructor(redis, name) {
    this.redis = redis;
    this.key = `leaderboard:${name}`;
  }
  
  async addScore(userId, score) {
    return this.redis.zadd(this.key, 'GT', score, `user:${userId}`);
  }
  
  async incrementScore(userId, points) {
    const newScore = await this.redis.zincrby(this.key, points, `user:${userId}`);
    return parseFloat(newScore);
  }
  
  async getTopN(n = 10) {
    const results = await this.redis.zrange(
      this.key, 0, n - 1, 'REV', 'WITHSCORES'
    );
    
    const leaderboard = [];
    for (let i = 0; i < results.length; i += 2) {
      leaderboard.push({
        userId: parseInt(results[i].replace('user:', '')),
        score: parseFloat(results[i + 1]),
        rank: Math.floor(i / 2) + 1
      });
    }
    return leaderboard;
  }
  
  async getUserRank(userId) {
    const rank = await this.redis.zrevrank(this.key, `user:${userId}`);
    if (rank === null) return null;
    return rank + 1;  // 1-indexed
  }
  
  async getUserScore(userId) {
    const score = await this.redis.zscore(this.key, `user:${userId}`);
    return score ? parseFloat(score) : 0;
  }
  
  async getUserNeighbors(userId, range = 2) {
    const rank = await this.redis.zrevrank(this.key, `user:${userId}`);
    if (rank === null) return null;
    
    const start = Math.max(0, rank - range);
    const end = rank + range;
    
    const results = await this.redis.zrange(
      this.key, start, end, 'REV', 'WITHSCORES'
    );
    
    const neighbors = [];
    for (let i = 0; i < results.length; i += 2) {
      const memberRank = start + Math.floor(i / 2) + 1;
      neighbors.push({
        userId: parseInt(results[i].replace('user:', '')),
        score: parseFloat(results[i + 1]),
        rank: memberRank,
        isCurrentUser: results[i] === `user:${userId}`
      });
    }
    return neighbors;
  }
  
  async getScoreDistribution(buckets = 10) {
    // Histogram of scores
    const minScore = parseFloat(
      (await this.redis.zrange(this.key, 0, 0, 'WITHSCORES'))[1] || '0'
    );
    const maxScore = parseFloat(
      (await this.redis.zrange(this.key, -1, -1, 'WITHSCORES'))[1] || '0'
    );
    
    const bucketSize = (maxScore - minScore) / buckets;
    const distribution = [];
    
    for (let i = 0; i < buckets; i++) {
      const min = minScore + i * bucketSize;
      const max = i === buckets - 1 ? maxScore : min + bucketSize;
      const count = await this.redis.zcount(this.key, min, max);
      distribution.push({ min, max, count });
    }
    
    return distribution;
  }
}

// Use Case 2: Priority Queue
class PriorityQueue {
  constructor(redis, name) {
    this.redis = redis;
    this.key = `queue:priority:${name}`;
  }
  
  async enqueue(task, priority = 0) {
    const taskData = JSON.stringify({
      ...task,
      id: generateId(),
      enqueuedAt: Date.now(),
      priority
    });
    await this.redis.zadd(this.key, priority, taskData);
  }
  
  async dequeueHighest() {
    // Pop task ที่ priority สูงสุด
    const result = await this.redis.zpopmax(this.key);
    if (!result || result.length === 0) return null;
    return JSON.parse(result[0]);  // [member, score]
  }
  
  async dequeuedLowest() {
    const result = await this.redis.zpopmin(this.key);
    if (!result || result.length === 0) return null;
    return JSON.parse(result[0]);
  }
  
  async blockingDequeue(timeout = 0) {
    const result = await this.redis.bzpopmax(this.key, timeout);
    if (!result) return null;
    return JSON.parse(result[1]);  // [key, member, score]
  }
  
  async size() {
    return this.redis.zcard(this.key);
  }
  
  async peek(count = 5) {
    const results = await this.redis.zrange(
      this.key, -count, -1, 'REV', 'WITHSCORES'
    );
    const tasks = [];
    for (let i = 0; i < results.length; i += 2) {
      tasks.push({
        ...JSON.parse(results[i]),
        currentPriority: parseFloat(results[i + 1])
      });
    }
    return tasks;
  }
}

// Use Case 3: Time-series with ZSet
class TimeSeries {
  constructor(redis) {
    this.redis = redis;
  }
  
  key(metricName) {
    return `timeseries:${metricName}`;
  }
  
  async record(metricName, value, timestamp = Date.now()) {
    // score = timestamp, member = "timestamp:value"
    const member = `${timestamp}:${value}`;
    await this.redis.zadd(this.key(metricName), timestamp, member);
  }
  
  async getRange(metricName, fromMs, toMs) {
    const results = await this.redis.zrangebyscore(
      this.key(metricName), fromMs, toMs, 'WITHSCORES'
    );
    
    const series = [];
    for (let i = 0; i < results.length; i += 2) {
      const [timestamp, value] = results[i].split(':');
      series.push({
        timestamp: parseInt(timestamp),
        value: parseFloat(value)
      });
    }
    return series;
  }
  
  async cleanup(metricName, keepDurationMs) {
    const cutoff = Date.now() - keepDurationMs;
    const removed = await this.redis.zremrangebyscore(
      this.key(metricName), '-inf', cutoff
    );
    return removed;
  }
  
  async getLatest(metricName, count = 100) {
    const results = await this.redis.zrange(
      this.key(metricName), -count, -1, 'WITHSCORES'
    );
    
    const series = [];
    for (let i = 0; i < results.length; i += 2) {
      const [ts, val] = results[i].split(':');
      series.push({ timestamp: parseInt(ts), value: parseFloat(val) });
    }
    return series.reverse();
  }
}
```

---

## 6. HyperLogLog: Unique Counting

HyperLogLog ใช้ approximate counting สำหรับ unique elements โดยใช้ memory แค่ 12 KB แทนที่จะต้องเก็บทุก element

### 6.1 คำสั่ง HyperLogLog

```bash
# PFADD: เพิ่ม elements
PFADD unique:visitors:2024-01-01 "user:1" "user:2" "user:3"
PFADD unique:visitors:2024-01-01 "user:2" "user:4"  # user:2 นับแค่ครั้งเดียว

# PFCOUNT: ประมาณจำนวน unique elements (error rate ~0.81%)
PFCOUNT unique:visitors:2024-01-01    # ประมาณ 4

# หลาย keys: union count
PFADD unique:visitors:2024-01-02 "user:3" "user:5"
PFCOUNT unique:visitors:2024-01-01 unique:visitors:2024-01-02  # unique ทั้งสองวัน

# PFMERGE: รวม HyperLogLogs
PFMERGE unique:visitors:2024-01 \
    unique:visitors:2024-01-01 \
    unique:visitors:2024-01-02 \
    unique:visitors:2024-01-03
PFCOUNT unique:visitors:2024-01    # unique visitors ทั้งเดือน
```

### 6.2 Use Cases: Unique Visitor Counting

```javascript
// Unique Visitor Counter
class UniqueVisitorCounter {
  constructor(redis) {
    this.redis = redis;
  }
  
  dailyKey(pageId, date) {
    const dateStr = date.toISOString().split('T')[0];
    return `uv:page:${pageId}:${dateStr}`;
  }
  
  weeklyKey(pageId, year, week) {
    return `uv:page:${pageId}:${year}:W${week}`;
  }
  
  monthlyKey(pageId, year, month) {
    return `uv:page:${pageId}:${year}:${month}`;
  }
  
  async trackVisit(pageId, userId) {
    const now = new Date();
    const pipeline = this.redis.pipeline();
    
    const dayKey = this.dailyKey(pageId, now);
    
    // บันทึก daily, weekly, monthly พร้อมกัน
    pipeline.pfadd(dayKey, userId.toString());
    pipeline.expire(dayKey, 86400 * 90);  // keep 90 days
    
    await pipeline.exec();
  }
  
  async getDailyUniques(pageId, date = new Date()) {
    return this.redis.pfcount(this.dailyKey(pageId, date));
  }
  
  async getMonthlyUniques(pageId, year, month) {
    // รวม daily counts ทั้งเดือน
    const days = getDaysInMonth(year, month);
    const keys = days.map(day => this.dailyKey(pageId, day));
    return this.redis.pfcount(...keys);
  }
  
  async getDateRangeUniques(pageId, startDate, endDate) {
    const dates = getDateRange(startDate, endDate);
    const keys = dates.map(date => this.dailyKey(pageId, date));
    return this.redis.pfcount(...keys);
  }
}
```

---

## 7. Bitmap: Boolean Flags

Bitmap คือ String ที่ treat bits เป็น array of boolean values - ประหยัด memory มากสำหรับ per-user flags

### 7.1 คำสั่ง Bitmap

```bash
# SETBIT: ตั้งค่า bit
SETBIT daily:active:2024-01-01 123 1    # user 123 active วันนี้
SETBIT daily:active:2024-01-01 456 1    # user 456 active วันนี้
SETBIT features:enabled 0 1             # feature 0 enabled

# GETBIT: ดู bit
GETBIT daily:active:2024-01-01 123      # 1
GETBIT daily:active:2024-01-01 999      # 0

# BITCOUNT: นับ bits ที่เป็น 1
BITCOUNT daily:active:2024-01-01        # จำนวน active users วันนี้
BITCOUNT daily:active:2024-01-01 0 99   # นับใน bytes 0-99

# BITPOS: หา position ของ bit แรกที่เป็น 0 หรือ 1
BITPOS daily:active:2024-01-01 1    # user ID แรกที่ active
BITPOS daily:active:2024-01-01 0    # user ID แรกที่ไม่ active

# BITOP: Bitwise operations
BITOP AND result day1 day2    # users ที่ active ทั้งสองวัน
BITOP OR result day1 day2     # users ที่ active อย่างน้อยหนึ่งวัน
BITOP XOR result day1 day2    # users ที่ active เพียงวันเดียว
BITOP NOT result day1         # users ที่ไม่ active

# BITFIELD: Complex bit operations
BITFIELD counter INCRBY u8 0 1    # increment 8-bit unsigned field at offset 0
```

### 7.2 Use Cases: Daily Active Users, Feature Flags

```javascript
// Daily Active Users Tracking
class DailyActiveUsers {
  constructor(redis) {
    this.redis = redis;
  }
  
  bitmapKey(date) {
    return `dau:${date.toISOString().split('T')[0]}`;
  }
  
  async recordActivity(userId) {
    const key = this.bitmapKey(new Date());
    await this.redis.setbit(key, userId, 1);
    await this.redis.expire(key, 86400 * 90);  // keep 90 days
  }
  
  async isActiveToday(userId) {
    const key = this.bitmapKey(new Date());
    const bit = await this.redis.getbit(key, userId);
    return bit === 1;
  }
  
  async getDailyActiveCount(date = new Date()) {
    return this.redis.bitcount(this.bitmapKey(date));
  }
  
  async getActiveUsersForDates(dates) {
    const keys = dates.map(d => this.bitmapKey(d));
    const resultKey = `temp:dau:result:${Date.now()}`;
    
    // Users active ทุกวัน (AND)
    await this.redis.bitop('AND', resultKey, ...keys);
    const count = await this.redis.bitcount(resultKey);
    await this.redis.del(resultKey);
    
    return count;
  }
  
  async getRetentionRate(cohortDate, checkDate) {
    const cohortKey = this.bitmapKey(cohortDate);
    const checkKey = this.bitmapKey(checkDate);
    const resultKey = `temp:retention:${Date.now()}`;
    
    // Users ที่ active ใน cohort date AND check date
    await this.redis.bitop('AND', resultKey, cohortKey, checkKey);
    const retained = await this.redis.bitcount(resultKey);
    const total = await this.redis.bitcount(cohortKey);
    await this.redis.del(resultKey);
    
    return total > 0 ? (retained / total) * 100 : 0;
  }
}

// Feature Flags with User Percentage Rollout
class FeatureFlags {
  constructor(redis) {
    this.redis = redis;
  }
  
  rolloutKey(featureName) {
    return `feature:rollout:${featureName}`;
  }
  
  async enableForUser(featureName, userId) {
    await this.redis.setbit(this.rolloutKey(featureName), userId, 1);
  }
  
  async disableForUser(featureName, userId) {
    await this.redis.setbit(this.rolloutKey(featureName), userId, 0);
  }
  
  async isEnabled(featureName, userId) {
    const bit = await this.redis.getbit(this.rolloutKey(featureName), userId);
    return bit === 1;
  }
  
  async rolloutToPercent(featureName, totalUsers, percent) {
    const enabledCount = Math.floor(totalUsers * (percent / 100));
    const pipeline = this.redis.pipeline();
    
    // Enable ให้ random users
    const userIds = Array.from({length: enabledCount}, 
      (_, i) => Math.floor(Math.random() * totalUsers) + 1
    );
    
    for (const userId of userIds) {
      pipeline.setbit(this.rolloutKey(featureName), userId, 1);
    }
    
    await pipeline.exec();
    return enabledCount;
  }
  
  async getEnabledCount(featureName) {
    return this.redis.bitcount(this.rolloutKey(featureName));
  }
}
```

---

## 8. Geo: Location-based Features

Redis Geo ใช้ Sorted Set ภายในเพื่อเก็บ latitude/longitude และ query ตาม distance

### 8.1 คำสั่ง Geo

```bash
# GEOADD: เพิ่ม locations
GEOADD restaurants \
    100.5018 13.7563 "Mama Restaurant" \
    100.4930 13.7419 "Thai Bistro" \
    100.5212 13.7308 "Siam Kitchen"

# GEODIST: ระยะทางระหว่าง 2 locations
GEODIST restaurants "Mama Restaurant" "Thai Bistro"        # meters
GEODIST restaurants "Mama Restaurant" "Thai Bistro" km     # kilometers
GEODIST restaurants "Mama Restaurant" "Thai Bistro" mi     # miles

# GEOPOS: ดู lat/lng ของ members
GEOPOS restaurants "Mama Restaurant" "Thai Bistro"

# GEOSEARCH: ค้นหาใน radius (Redis 6.2+)
# GEOSEARCH key FROMLONLAT lon lat BYRADIUS radius unit [ASC|DESC] [COUNT n] [WITHCOORD] [WITHDIST]
GEOSEARCH restaurants \
    FROMLONLAT 100.5018 13.7563 \
    BYRADIUS 2 km \
    ASC \
    COUNT 5 \
    WITHCOORD WITHDIST

# GEOSEARCH by bounding box
GEOSEARCH restaurants \
    FROMLONLAT 100.5018 13.7563 \
    BYBOX 4 4 km \
    ASC \
    WITHCOORD WITHDIST

# GEOSEARCHSTORE: บันทึกผลลัพธ์ใน sorted set อื่น
GEOSEARCHSTORE dest restaurants \
    FROMLONLAT 100.5018 13.7563 \
    BYRADIUS 2 km ASC

# GEOHASH: ดู geohash string (สำหรับ indexing/comparison)
GEOHASH restaurants "Mama Restaurant"    # "w3gv4q52px0"

# Legacy commands (ยังใช้ได้ แต่ GEOSEARCH แนะนำกว่า)
GEORADIUS restaurants 100.5018 13.7563 2 km ASC WITHCOORD WITHDIST
GEORADIUSBYMEMBER restaurants "Mama Restaurant" 2 km ASC
```

### 8.2 Use Cases: Nearby Restaurants, Delivery

```javascript
// Location-based Search Service
class LocationService {
  constructor(redis) {
    this.redis = redis;
  }
  
  geoKey(category) {
    return `geo:${category}`;
  }
  
  async addLocation(category, id, lat, lng, metadata = {}) {
    // เพิ่ม location
    await this.redis.geoadd(this.geoKey(category), lng, lat, `${category}:${id}`);
    
    // เก็บ metadata แยกต่างหาก (geo ไม่เก็บ extra data)
    if (Object.keys(metadata).length > 0) {
      await this.redis.hset(
        `location:meta:${category}:${id}`,
        ...Object.entries(metadata).flat().map(String)
      );
    }
  }
  
  async findNearby(category, lat, lng, radiusKm, options = {}) {
    const { limit = 10, withDistance = true, withCoords = true } = options;
    
    const args = [
      this.geoKey(category),
      'FROMLONLAT', lng, lat,
      'BYRADIUS', radiusKm, 'km',
      'ASC',
      'COUNT', limit
    ];
    
    if (withCoords) args.push('WITHCOORD');
    if (withDistance) args.push('WITHDIST');
    
    const results = await this.redis.geosearch(...args);
    
    // แปลงผลลัพธ์
    const locations = [];
    for (const result of results) {
      let member, distance, coords;
      
      if (withDistance && withCoords) {
        [member, distance, coords] = result;
      } else if (withDistance) {
        [member, distance] = result;
      } else if (withCoords) {
        [member, coords] = result;
      } else {
        member = result;
      }
      
      const id = member.replace(`${category}:`, '');
      const meta = await this.redis.hgetall(`location:meta:${category}:${id}`);
      
      locations.push({
        id,
        member,
        distance: distance ? parseFloat(distance) : null,
        lat: coords ? parseFloat(coords[1]) : null,
        lng: coords ? parseFloat(coords[0]) : null,
        ...meta
      });
    }
    
    return locations;
  }
  
  async getDistance(category, id1, id2, unit = 'km') {
    const dist = await this.redis.geodist(
      this.geoKey(category),
      `${category}:${id1}`,
      `${category}:${id2}`,
      unit
    );
    return dist ? parseFloat(dist) : null;
  }
  
  async removeLocation(category, id) {
    await this.redis.zrem(this.geoKey(category), `${category}:${id}`);
    await this.redis.del(`location:meta:${category}:${id}`);
  }
}

// ใช้งาน
const locationService = new LocationService(redisClient);

// เพิ่ม restaurants
await locationService.addLocation('restaurant', 1, 13.7563, 100.5018, {
  name: 'Mama Restaurant',
  cuisine: 'Thai',
  rating: '4.5',
  priceRange: '$$'
});

// ค้นหา restaurants ใกล้
const nearby = await locationService.findNearby('restaurant', 13.7563, 100.5018, 2);
console.log('Nearby restaurants:', nearby);
```

---

## 9. Stream: Event Logging

Redis Stream เป็น data structure สำหรับ append-only log คล้าย Kafka แต่ built-in ใน Redis

### 9.1 คำสั่ง Stream

```bash
# XADD: เพิ่ม entry
XADD events * type "user_login" user_id "123" ip "192.168.1.1"
# * = auto-generate ID (timestamp-sequence)
# คืน ID เช่น "1704067200000-0"

XADD events 1704067200000-1 type "purchase" amount "500"  # manual ID

# XLEN: จำนวน entries
XLEN events

# XRANGE: ดู entries ในช่วง ID
XRANGE events - +              # ทั้งหมด
XRANGE events - + COUNT 10     # 10 entries แรก
XRANGE events 1704067200000-0 + # จาก ID นี้เป็นต้นไป

# XREVRANGE: reverse order
XREVRANGE events + - COUNT 10

# XREAD: อ่าน entries จากหลาย streams
XREAD COUNT 10 STREAMS events $    # $ = ตั้งแต่ตอนนี้เป็นต้นไป (new entries)
XREAD COUNT 10 STREAMS events 0    # 0 = ตั้งแต่ต้น

# Blocking XREAD
XREAD BLOCK 5000 COUNT 10 STREAMS events $   # รอ 5 วินาที

# ========================================
# Consumer Groups
# ========================================

# XGROUP CREATE: สร้าง consumer group
XGROUP CREATE events my-group $ MKSTREAM

# XREADGROUP: อ่านด้วย consumer group
XREADGROUP GROUP my-group worker-1 COUNT 10 STREAMS events >
# > = entries ที่ยังไม่ถูก deliver ให้ group นี้

# XACK: Acknowledge ว่าประมวลผลแล้ว
XACK events my-group 1704067200000-0

# XPENDING: ดู messages ที่ pending (delivered แต่ยัง ack)
XPENDING events my-group - + 10
XPENDING events my-group - + 10 worker-1   # เฉพาะ worker-1

# XCLAIM: Claim message ที่ค้าง (สำหรับ failure recovery)
XCLAIM events my-group worker-2 60000 1704067200000-0
# idle-time = 60000ms (claim ถ้า idle > 60 วินาที)

# XAUTOCLAIM: Auto claim messages ที่ idle นาน
XAUTOCLAIM events my-group worker-2 60000 0-0 COUNT 10

# XTRIM: จำกัด stream length
XTRIM events MAXLEN 10000         # เก็บแค่ 10000 entries
XTRIM events MAXLEN ~ 10000       # approximate (เร็วกว่า)

# XDEL: ลบ entry เฉพาะ
XDEL events 1704067200000-0

# XINFO: ดูข้อมูล stream/group
XINFO STREAM events
XINFO GROUPS events
XINFO CONSUMERS events my-group
```

### 9.2 Use Cases: Event Log, Message Streaming

```javascript
// Event Streaming Service
class EventStream {
  constructor(redis) {
    this.redis = redis;
    this.streamKey = 'events:app';
    this.groupName = 'processors';
  }
  
  async publish(eventType, data) {
    const id = await this.redis.xadd(
      this.streamKey,
      '*',  // auto ID
      'type', eventType,
      'data', JSON.stringify(data),
      'timestamp', Date.now().toString()
    );
    return id;
  }
  
  async createConsumerGroup(groupName = this.groupName) {
    try {
      await this.redis.xgroup('CREATE', this.streamKey, groupName, '$', 'MKSTREAM');
      console.log(`Consumer group '${groupName}' created`);
    } catch (err) {
      if (!err.message.includes('BUSYGROUP')) throw err;
      console.log(`Consumer group '${groupName}' already exists`);
    }
  }
  
  async consume(consumerName, count = 10, blockMs = 5000) {
    const results = await this.redis.xreadgroup(
      'GROUP', this.groupName,
      consumerName,
      'COUNT', count,
      'BLOCK', blockMs,
      'STREAMS', this.streamKey, '>'
    );
    
    if (!results) return [];
    
    const [, messages] = results[0];
    return messages.map(([id, fields]) => {
      const event = {};
      for (let i = 0; i < fields.length; i += 2) {
        event[fields[i]] = fields[i + 1];
      }
      return { id, ...event, data: JSON.parse(event.data) };
    });
  }
  
  async acknowledge(messageId) {
    return this.redis.xack(this.streamKey, this.groupName, messageId);
  }
  
  async getPendingMessages(consumerName = null) {
    const args = ['XPENDING', this.streamKey, this.groupName, '-', '+', 100];
    if (consumerName) args.push(consumerName);
    return this.redis.call(...args);
  }
  
  async startWorker(workerName, handler) {
    console.log(`Worker ${workerName} starting...`);
    
    while (true) {
      const messages = await this.consume(workerName, 10, 5000);
      
      for (const message of messages) {
        try {
          await handler(message);
          await this.acknowledge(message.id);
        } catch (error) {
          console.error(`Error processing ${message.id}:`, error);
          // ไม่ ack → จะยังอยู่ใน pending list
        }
      }
    }
  }
}

// ใช้งาน
const events = new EventStream(redisClient);
await events.createConsumerGroup();

// Producer
await events.publish('user_action', { 
  userId: 123, action: 'click', element: 'buy_button' 
});

// Consumer Worker
events.startWorker('worker-1', async (event) => {
  console.log(`Processing event ${event.id}:`, event.type);
  
  if (event.type === 'user_action') {
    await db.query(
      'INSERT INTO user_analytics (user_id, action, element) VALUES ($1, $2, $3)',
      [event.data.userId, event.data.action, event.data.element]
    );
  }
});
```

---

## 10. Workshop: Leaderboard + Activity Feed + Unique Visitors

```javascript
// social-gaming-platform.js - ระบบครบวงจร

const Redis = require('ioredis');
const redis = new Redis({ host: 'localhost', port: 6379 });

// ========================================
// 1. Leaderboard System
// ========================================

const leaderboard = {
  async updateScore(gameId, userId, points) {
    const key = `leaderboard:${gameId}`;
    const newScore = await redis.zincrby(key, points, `user:${userId}`);
    
    // อัพเดท user stats
    await redis.hset(`user:${userId}:stats`,
      'total_points', newScore,
      'last_active', Date.now()
    );
    
    return parseFloat(newScore);
  },
  
  async getTopPlayers(gameId, count = 10) {
    const key = `leaderboard:${gameId}`;
    const results = await redis.zrange(key, 0, count - 1, 'REV', 'WITHSCORES');
    
    const players = [];
    const pipeline = redis.pipeline();
    
    for (let i = 0; i < results.length; i += 2) {
      const userId = results[i].replace('user:', '');
      pipeline.hgetall(`user:${userId}:stats`);
      players.push({
        userId,
        score: parseFloat(results[i + 1]),
        rank: Math.floor(i / 2) + 1
      });
    }
    
    const statsResults = await pipeline.exec();
    for (let i = 0; i < players.length; i++) {
      players[i].stats = statsResults[i][1];
    }
    
    return players;
  },
  
  async getPlayerRank(gameId, userId) {
    const key = `leaderboard:${gameId}`;
    const rank = await redis.zrevrank(key, `user:${userId}`);
    const score = await redis.zscore(key, `user:${userId}`);
    return { rank: rank !== null ? rank + 1 : null, score: parseFloat(score) };
  }
};

// ========================================
// 2. Activity Feed System
// ========================================

const activityFeed = {
  async publishActivity(actorId, activityType, data) {
    const activity = {
      id: `${Date.now()}-${Math.random().toString(36).substr(2)}`,
      actorId,
      type: activityType,
      data,
      timestamp: Date.now()
    };
    
    const serialized = JSON.stringify(activity);
    
    // เพิ่มใน actor's own feed
    const actorKey = `feed:user:${actorId}`;
    await redis.lpush(actorKey, serialized);
    await redis.ltrim(actorKey, 0, 499);  // keep 500 items
    
    // Fan-out ไปยัง followers (ใน production ใช้ queue สำหรับ large-scale)
    const followers = await redis.smembers(`followers:${actorId}`);
    
    if (followers.length > 0) {
      const pipeline = redis.pipeline();
      for (const followerId of followers) {
        const followerFeedKey = `feed:user:${followerId}`;
        pipeline.lpush(followerFeedKey, serialized);
        pipeline.ltrim(followerFeedKey, 0, 199);  // keep 200 for followers
      }
      await pipeline.exec();
    }
    
    return activity;
  },
  
  async getUserFeed(userId, page = 1, pageSize = 20) {
    const start = (page - 1) * pageSize;
    const end = start + pageSize - 1;
    
    const items = await redis.lrange(`feed:user:${userId}`, start, end);
    return items.map(item => JSON.parse(item));
  },
  
  async addFollower(userId, followerId) {
    await redis.sadd(`followers:${userId}`, followerId);
    await redis.sadd(`following:${followerId}`, userId);
  },
  
  async getFollowerCount(userId) {
    return redis.scard(`followers:${userId}`);
  }
};

// ========================================
// 3. Unique Visitor Counter
// ========================================

const uniqueVisitors = {
  async trackVisit(pageId, userId) {
    const today = new Date().toISOString().split('T')[0];
    
    // HyperLogLog สำหรับ approximate count
    const hlKey = `uv:hl:${pageId}:${today}`;
    await redis.pfadd(hlKey, userId.toString());
    await redis.expire(hlKey, 86400 * 90);
    
    // Bitmap สำหรับ exact daily active users
    const bitmapKey = `uv:bm:${today}`;
    await redis.setbit(bitmapKey, userId, 1);
    await redis.expire(bitmapKey, 86400 * 90);
    
    // Sorted set สำหรับ trending pages
    const trendingKey = `trending:pages:${today}`;
    await redis.zincrby(trendingKey, 1, `page:${pageId}`);
    await redis.expire(trendingKey, 86400 * 2);
  },
  
  async getDailyUniqueVisitors(pageId, date = null) {
    const d = date || new Date().toISOString().split('T')[0];
    return redis.pfcount(`uv:hl:${pageId}:${d}`);
  },
  
  async getWeeklyUniqueVisitors(pageId) {
    const keys = [];
    for (let i = 0; i < 7; i++) {
      const d = new Date();
      d.setDate(d.getDate() - i);
      keys.push(`uv:hl:${pageId}:${d.toISOString().split('T')[0]}`);
    }
    return redis.pfcount(...keys);
  },
  
  async getDailyActiveUsers() {
    const today = new Date().toISOString().split('T')[0];
    return redis.bitcount(`uv:bm:${today}`);
  },
  
  async getTrendingPages(count = 10) {
    const today = new Date().toISOString().split('T')[0];
    const results = await redis.zrange(
      `trending:pages:${today}`, 0, count - 1, 'REV', 'WITHSCORES'
    );
    
    const pages = [];
    for (let i = 0; i < results.length; i += 2) {
      pages.push({
        pageId: results[i].replace('page:', ''),
        views: parseInt(results[i + 1])
      });
    }
    return pages;
  }
};

// ========================================
// Demo / Test
// ========================================

async function runDemo() {
  console.log('=== Social Gaming Platform Demo ===\n');
  
  // Setup: followers
  await activityFeed.addFollower(1, 2);  // user 2 follows user 1
  await activityFeed.addFollower(1, 3);  // user 3 follows user 1
  
  // Game actions
  console.log('--- Leaderboard ---');
  await leaderboard.updateScore('game:chess', 1, 1500);
  await leaderboard.updateScore('game:chess', 2, 2200);
  await leaderboard.updateScore('game:chess', 3, 1800);
  await leaderboard.updateScore('game:chess', 1, 500);  // user 1 earns more
  
  const topPlayers = await leaderboard.getTopPlayers('game:chess', 5);
  console.log('Top players:', JSON.stringify(topPlayers, null, 2));
  
  const user1Rank = await leaderboard.getPlayerRank('game:chess', 1);
  console.log('User 1 rank:', user1Rank);
  
  // Activity feed
  console.log('\n--- Activity Feed ---');
  await activityFeed.publishActivity(1, 'game_win', { 
    game: 'chess', opponent: 2, score: 2000 
  });
  await activityFeed.publishActivity(2, 'achievement', { 
    name: 'Chess Master', points: 100 
  });
  
  const user2Feed = await activityFeed.getUserFeed(2, 1, 5);
  console.log('User 2 feed (sees user 1 activity):', user2Feed.length, 'items');
  user2Feed.forEach(item => console.log(`  [${item.type}]`, item.data));
  
  // Unique visitors
  console.log('\n--- Unique Visitors ---');
  for (let i = 1; i <= 100; i++) {
    await uniqueVisitors.trackVisit('homepage', i);
  }
  // Some users visit again (shouldn't count twice)
  for (let i = 1; i <= 20; i++) {
    await uniqueVisitors.trackVisit('homepage', i);
  }
  
  const dailyUV = await uniqueVisitors.getDailyUniqueVisitors('homepage');
  const dau = await uniqueVisitors.getDailyActiveUsers();
  const trending = await uniqueVisitors.getTrendingPages(5);
  
  console.log(`Daily unique visitors (homepage): ~${dailyUV}`);
  console.log(`Daily active users: ${dau}`);
  console.log('Trending pages:', trending);
  
  console.log('\n=== Demo Complete ===');
}

runDemo()
  .then(() => redis.quit())
  .catch(console.error);
```

---

## สรุป

ในบทนี้เราได้เรียนรู้ Redis Data Structures ทั้งหมด:

1. **String**: SET, GET, INCR, APPEND - เหมาะสำหรับ simple values, counters, tokens
2. **Hash**: HSET, HGET, HINCRBY - เหมาะสำหรับ objects ที่มีหลาย fields (user profiles, carts)
3. **List**: LPUSH, RPUSH, BLPOP, LRANGE - เหมาะสำหรับ queues, feeds, recent items
4. **Set**: SADD, SISMEMBER, SINTER, SUNION - เหมาะสำหรับ unique collections, tags, relationships
5. **Sorted Set**: ZADD, ZRANGE, ZINCRBY - เหมาะสำหรับ leaderboards, priority queues, time-series
6. **HyperLogLog**: PFADD, PFCOUNT - unique counting แบบ approximate ประหยัด memory
7. **Bitmap**: SETBIT, BITCOUNT, BITOP - boolean flags per user/item (DAU, feature flags)
8. **Geo**: GEOADD, GEOSEARCH - location-based features
9. **Stream**: XADD, XREAD, XGROUP - event streaming, message queues

บทถัดไปจะเรียนรู้เรื่อง MinIO ซึ่งเป็น S3-compatible object storage สำหรับเก็บไฟล์ขนาดใหญ่

---

*[Part 09 จบ — ไปต่อ Part 10: MinIO Object Storage]*

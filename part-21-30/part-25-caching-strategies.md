# Part 25: Caching Strategies - Cache-Aside, Write-Through, Write-Behind

## บทนำ: ทำไม Caching ถึงสำคัญ

Caching คือการเก็บข้อมูลที่ใช้บ่อยไว้ในที่เข้าถึงเร็ว (เช่น Memory) เพื่อลดการเข้าถึงแหล่งข้อมูลช้า (เช่น Database)

```
ไม่มี Cache:
Request → Database (50ms) → Response
          ↑ ทุก Request ต้องไปที่ DB

มี Cache:
Request → Cache HIT  → Response (1ms)
          Cache MISS → Database (50ms) → Cache → Response
          ↑ แค่ครั้งแรกเท่านั้นที่ไป DB
```

**ประโยชน์:**
- **Latency ลดลง**: Memory เร็วกว่า Disk หลาย 1000x
- **Database Load ลดลง**: ลด Query จำนวนมาก
- **Cost ลดลง**: DB ไม่ต้อง Scale ใหญ่
- **Availability สูงขึ้น**: Cache ยังทำงานแม้ DB ช้า/ล่ม

---

## 1. Cache-Aside (Lazy Loading)

### 1.1 Flow

```
Application                Cache (Redis)           Database
     │                          │                      │
     │──── GET user:1 ─────────▶│                      │
     │                          │ MISS                 │
     │◀─── null ────────────────│                      │
     │                          │                      │
     │──── SELECT * FROM ... ──────────────────────────▶│
     │◀─── {id:1, name:"Alice"} ───────────────────────│
     │                          │                      │
     │──── SET user:1 (TTL)────▶│                      │
     │                          │                      │
     │──── Return to Client ────│                      │
     
 ครั้งต่อไป:
     │──── GET user:1 ─────────▶│                      │
     │◀─── {id:1, name:"Alice"} │ HIT                  │
     │──── Return to Client ────│                      │
```

### 1.2 Implementation

```typescript
// src/cache/CacheAside.ts
import { Redis } from 'ioredis';
import { Pool } from 'pg';

interface CacheAsideOptions {
  ttl: number;           // seconds
  keyPrefix?: string;
  compress?: boolean;
  onCacheMiss?: (key: string) => void;
  onCacheHit?: (key: string) => void;
}

class CacheAsideStrategy<T = unknown> {
  private hitCount = 0;
  private missCount = 0;

  constructor(
    private readonly redis: Redis,
    private readonly options: CacheAsideOptions
  ) {}

  // ==========================================
  // Core: Get or Fetch
  // ==========================================
  async get(
    key: string,
    fetcher: () => Promise<T | null>
  ): Promise<T | null> {
    const cacheKey = this.buildKey(key);

    // 1. ตรวจสอบ Cache ก่อน
    const cached = await this.redis.get(cacheKey);

    if (cached !== null) {
      this.hitCount++;
      this.options.onCacheHit?.(key);
      return JSON.parse(cached) as T;
    }

    // 2. Cache Miss: ดึงจาก Database
    this.missCount++;
    this.options.onCacheMiss?.(key);

    const data = await fetcher();

    // 3. เก็บใน Cache (ถ้ามีข้อมูล)
    if (data !== null && data !== undefined) {
      await this.set(key, data);
    }

    return data;
  }

  async set(key: string, value: T, ttl?: number): Promise<void> {
    const cacheKey = this.buildKey(key);
    const serialized = JSON.stringify(value);
    const effectiveTtl = ttl ?? this.options.ttl;

    await this.redis.set(cacheKey, serialized, 'EX', effectiveTtl);
  }

  async invalidate(key: string): Promise<void> {
    const cacheKey = this.buildKey(key);
    await this.redis.del(cacheKey);
  }

  async invalidatePattern(pattern: string): Promise<number> {
    // ระวัง: KEYS command ช้าใน Production (ใช้ SCAN แทน)
    const keys = await this.scanKeys(`${this.options.keyPrefix}:${pattern}`);
    if (keys.length === 0) return 0;
    return this.redis.del(...keys);
  }

  private async scanKeys(pattern: string): Promise<string[]> {
    const keys: string[] = [];
    let cursor = '0';

    do {
      const [newCursor, foundKeys] = await this.redis.scan(
        cursor,
        'MATCH', pattern,
        'COUNT', 100
      );
      cursor = newCursor;
      keys.push(...foundKeys);
    } while (cursor !== '0');

    return keys;
  }

  private buildKey(key: string): string {
    const prefix = this.options.keyPrefix ?? 'cache';
    return `${prefix}:${key}`;
  }

  getStats() {
    const total = this.hitCount + this.missCount;
    return {
      hits: this.hitCount,
      misses: this.missCount,
      total,
      hitRate: total > 0 ? (this.hitCount / total * 100).toFixed(2) + '%' : '0%',
    };
  }

  resetStats() {
    this.hitCount = 0;
    this.missCount = 0;
  }
}

// ==========================================
// ตัวอย่างการใช้งาน
// ==========================================
async function cacheAsideExample(
  redis: Redis,
  db: Pool
) {
  const userCache = new CacheAsideStrategy<{
    id: number;
    name: string;
    email: string;
  }>(redis, {
    ttl: 3600,       // 1 hour
    keyPrefix: 'user',
    onCacheMiss: (key) => console.log(`Cache miss: ${key}`),
    onCacheHit: (key) => console.log(`Cache hit: ${key}`),
  });

  // อ่านข้อมูล User (Cache-Aside)
  const user = await userCache.get('1', async () => {
    const result = await db.query(
      'SELECT id, name, email FROM users WHERE id = $1',
      [1]
    );
    return result.rows[0] ?? null;
  });

  console.log('User:', user);
  console.log('Cache Stats:', userCache.getStats());

  // Invalidate หลัง Update
  await db.query(
    'UPDATE users SET name = $1 WHERE id = $2',
    ['Bob', 1]
  );
  await userCache.invalidate('1');
}

export { CacheAsideStrategy };
```

### 1.3 ข้อดี/ข้อเสีย

```
ข้อดี:
✅ Resilient: ถ้า Cache ล่ม ยังอ่านจาก DB ได้
✅ เก็บเฉพาะ Data ที่จำเป็น (Lazy)
✅ ง่ายต่อการ Implement
✅ Cache ไม่ Stale นาน (มี TTL)

ข้อเสีย:
❌ Cold Start: เข้าถึงครั้งแรกช้า (Cache Miss)
❌ Data Staleness: ระหว่าง TTL อาจเก่า
❌ Cache Stampede: หลาย Request พร้อมกัน Miss Cache
```

---

## 2. Read-Through

### 2.1 ความแตกต่างจาก Cache-Aside

```
Cache-Aside:
Application ──▶ Cache ──(miss)──▶ Application ──▶ DB

Read-Through:
Application ──▶ Cache ──(miss)──▶ DB (Cache ดึงเอง)
                Cache ◀────────────────────────────
```

### 2.2 Implementation

```typescript
// src/cache/ReadThrough.ts

interface DataLoader<T> {
  load(key: string): Promise<T | null>;
}

class ReadThroughCache<T = unknown> {
  private readonly defaultTtl: number;

  constructor(
    private readonly redis: Redis,
    private readonly loader: DataLoader<T>,
    options: { ttl: number; keyPrefix: string }
  ) {
    this.defaultTtl = options.ttl;
  }

  async get(key: string): Promise<T | null> {
    const cacheKey = `rt:${key}`;

    const cached = await this.redis.get(cacheKey);
    if (cached) {
      return JSON.parse(cached) as T;
    }

    // Cache โหลดข้อมูลเอง (Read-Through)
    const data = await this.loader.load(key);
    if (data) {
      await this.redis.set(cacheKey, JSON.stringify(data), 'EX', this.defaultTtl);
    }

    return data;
  }
}
```

---

## 3. Write-Through

### 3.1 Flow

```
Application                Cache (Redis)           Database
     │                          │                      │
     │──── WRITE user:1 ───────▶│                      │
     │                          │──── WRITE user:1 ────▶│
     │                          │◀─── OK ──────────────│
     │◀─── OK ──────────────────│                      │
     
 Cache และ DB อัปเดตพร้อมกัน
```

### 3.2 Implementation

```typescript
// src/cache/WriteThrough.ts
import { Redis } from 'ioredis';
import { Pool, PoolClient } from 'pg';

interface WriteThroughOptions {
  ttl: number;
  keyPrefix: string;
  writeTimeout?: number;
}

class WriteThroughStrategy<T extends { id: number | string }> {
  constructor(
    private readonly redis: Redis,
    private readonly db: Pool,
    private readonly tableName: string,
    private readonly options: WriteThroughOptions
  ) {}

  // ==========================================
  // WRITE: เขียน Cache และ DB พร้อมกัน
  // ==========================================
  async write(id: number | string, data: T): Promise<T> {
    const cacheKey = `${this.options.keyPrefix}:${id}`;

    // ใช้ Promise.all เพื่อเขียนพร้อมกัน
    await Promise.all([
      // เขียน Cache
      this.redis.set(
        cacheKey,
        JSON.stringify(data),
        'EX',
        this.options.ttl
      ),

      // เขียน Database
      this.writeToDatabase(id, data),
    ]);

    return data;
  }

  private async writeToDatabase(id: number | string, data: T): Promise<void> {
    const fields = Object.keys(data).filter(k => k !== 'id');
    const values = fields.map(f => (data as any)[f]);
    const setClause = fields
      .map((f, i) => `${f} = $${i + 2}`)
      .join(', ');

    await this.db.query(
      `UPDATE ${this.tableName} SET ${setClause}, updated_at = NOW() WHERE id = $1`,
      [id, ...values]
    );
  }

  // ==========================================
  // READ: อ่านจาก Cache ก่อน
  // ==========================================
  async read(id: number | string, fetcher: () => Promise<T | null>): Promise<T | null> {
    const cacheKey = `${this.options.keyPrefix}:${id}`;

    const cached = await this.redis.get(cacheKey);
    if (cached) {
      return JSON.parse(cached) as T;
    }

    // Cache Miss: ดึงจาก DB และเก็บใน Cache
    const data = await fetcher();
    if (data) {
      await this.redis.set(
        cacheKey,
        JSON.stringify(data),
        'EX',
        this.options.ttl
      );
    }

    return data;
  }

  async delete(id: number | string): Promise<void> {
    const cacheKey = `${this.options.keyPrefix}:${id}`;

    await Promise.all([
      this.redis.del(cacheKey),
      this.db.query(`DELETE FROM ${this.tableName} WHERE id = $1`, [id]),
    ]);
  }
}

// ==========================================
// ตัวอย่าง: User Profile Cache
// ==========================================
async function writeThroughExample(redis: Redis, db: Pool) {
  interface UserProfile {
    id: number;
    name: string;
    bio: string;
    avatarUrl: string;
  }

  const profileCache = new WriteThroughStrategy<UserProfile>(
    redis,
    db,
    'user_profiles',
    { ttl: 3600, keyPrefix: 'profile' }
  );

  // Write: เขียน Cache + DB พร้อมกัน
  const updated = await profileCache.write(1, {
    id: 1,
    name: 'Alice',
    bio: 'Software Engineer',
    avatarUrl: 'https://example.com/alice.jpg',
  });

  // Read: อ่านจาก Cache
  const profile = await profileCache.read(1, async () => {
    const result = await db.query(
      'SELECT * FROM user_profiles WHERE id = $1',
      [1]
    );
    return result.rows[0] ?? null;
  });

  console.log('Profile:', profile);
}

export { WriteThroughStrategy };
```

### 3.3 ข้อดี/ข้อเสีย

```
ข้อดี:
✅ Cache และ DB Consistent เสมอ
✅ Read ไม่เคย Stale
✅ ไม่มี Cache Miss หลัง Write

ข้อเสีย:
❌ Write ช้ากว่า (ต้องรอทั้ง Cache และ DB)
❌ เขียน Data ที่อาจไม่ถูกอ่านบ่อย (เปลือง Cache Memory)
❌ ถ้า DB Write ล้มเหลว ต้องจัดการ Rollback Cache
```

---

## 4. Write-Behind (Write-Back)

### 4.1 Flow

```
Application                Cache (Redis)           Database
     │                          │                      │
     │──── WRITE user:1 ───────▶│                      │
     │◀─── OK ──────────────────│                      │
     │                          │                      │
     │                   (ทีหลัง async)                │
     │                          │──── WRITE user:1 ────▶│
     │                          │◀─── OK ──────────────│
     
 Write ไปที่ Cache ก่อน เขียน DB ทีหลัง (Async)
```

### 4.2 Implementation ด้วย Queue

```typescript
// src/cache/WriteBehind.ts
import { Redis } from 'ioredis';
import { Pool } from 'pg';
import EventEmitter from 'events';

interface PendingWrite<T> {
  key: string;
  data: T;
  timestamp: number;
  retries: number;
}

interface WriteBehindOptions {
  ttl: number;
  keyPrefix: string;
  flushIntervalMs: number;   // flush ทุกกี่ ms
  maxBatchSize: number;      // flush สูงสุดกี่ item ต่อครั้ง
  maxRetries: number;        // Retry สูงสุด
  onError?: (error: Error, key: string) => void;
  onFlush?: (count: number) => void;
}

class WriteBehindStrategy<T extends { id: number | string }> extends EventEmitter {
  private writeQueue: Map<string, PendingWrite<T>> = new Map();
  private flushTimer?: NodeJS.Timeout;
  private isFlushing = false;

  constructor(
    private readonly redis: Redis,
    private readonly db: Pool,
    private readonly tableName: string,
    private readonly options: WriteBehindOptions
  ) {
    super();
    this.startFlushTimer();
  }

  // ==========================================
  // WRITE: เขียน Cache ทันที Queue DB
  // ==========================================
  async write(id: number | string, data: T): Promise<void> {
    const cacheKey = `${this.options.keyPrefix}:${id}`;
    const queueKey = String(id);

    // 1. เขียน Cache ทันที
    await this.redis.set(
      cacheKey,
      JSON.stringify(data),
      'EX',
      this.options.ttl
    );

    // 2. Queue DB Write (Overwrite ถ้ามีอยู่แล้ว - latest wins)
    this.writeQueue.set(queueKey, {
      key: queueKey,
      data,
      timestamp: Date.now(),
      retries: 0,
    });

    this.emit('queued', queueKey);
  }

  // ==========================================
  // READ: อ่านจาก Cache
  // ==========================================
  async read(id: number | string, fetcher: () => Promise<T | null>): Promise<T | null> {
    const cacheKey = `${this.options.keyPrefix}:${id}`;

    const cached = await this.redis.get(cacheKey);
    if (cached) {
      return JSON.parse(cached) as T;
    }

    const data = await fetcher();
    if (data) {
      await this.redis.set(cacheKey, JSON.stringify(data), 'EX', this.options.ttl);
    }

    return data;
  }

  // ==========================================
  // FLUSH Queue ไปยัง Database
  // ==========================================
  private async flush(): Promise<void> {
    if (this.isFlushing || this.writeQueue.size === 0) return;

    this.isFlushing = true;

    // ดึง Batch จาก Queue
    const batch: PendingWrite<T>[] = [];
    let count = 0;

    for (const [key, write] of this.writeQueue.entries()) {
      if (count >= this.options.maxBatchSize) break;
      batch.push(write);
      this.writeQueue.delete(key);
      count++;
    }

    if (batch.length === 0) {
      this.isFlushing = false;
      return;
    }

    // เขียน DB แบบ Batch (Transaction)
    const client = await this.db.connect();

    try {
      await client.query('BEGIN');

      for (const write of batch) {
        try {
          await this.writeToDatabase(client, write.data);
        } catch (error: any) {
          // Retry Logic
          if (write.retries < this.options.maxRetries) {
            write.retries++;
            this.writeQueue.set(write.key, write);
          } else {
            this.options.onError?.(error, write.key);
            this.emit('error', { key: write.key, error });
          }
        }
      }

      await client.query('COMMIT');
      this.options.onFlush?.(batch.length);
      this.emit('flushed', batch.length);

    } catch (error: any) {
      await client.query('ROLLBACK');

      // ใส่กลับ Queue
      for (const write of batch) {
        if (write.retries < this.options.maxRetries) {
          write.retries++;
          this.writeQueue.set(write.key, write);
        }
      }

      this.options.onError?.(error, 'batch');

    } finally {
      client.release();
      this.isFlushing = false;
    }
  }

  private async writeToDatabase(
    client: import('pg').PoolClient,
    data: T
  ): Promise<void> {
    const id = (data as any).id;
    const fields = Object.keys(data).filter(k => k !== 'id');
    const values = fields.map(f => (data as any)[f]);

    const setClause = fields
      .map((f, i) => `${f} = $${i + 2}`)
      .join(', ');

    await client.query(
      `INSERT INTO ${this.tableName} (id, ${fields.join(', ')}, updated_at)
       VALUES ($1, ${fields.map((_, i) => `$${i + 2}`).join(', ')}, NOW())
       ON CONFLICT (id) DO UPDATE SET ${setClause}, updated_at = NOW()`,
      [id, ...values]
    );
  }

  private startFlushTimer(): void {
    this.flushTimer = setInterval(async () => {
      await this.flush().catch(err => {
        console.error('Flush error:', err);
      });
    }, this.options.flushIntervalMs);
  }

  // Force Flush (เช่น ตอน Shutdown)
  async forceFlush(): Promise<void> {
    while (this.writeQueue.size > 0) {
      await this.flush();
      if (this.writeQueue.size > 0) {
        await new Promise(resolve => setTimeout(resolve, 100));
      }
    }
  }

  getQueueSize(): number {
    return this.writeQueue.size;
  }

  async shutdown(): Promise<void> {
    if (this.flushTimer) {
      clearInterval(this.flushTimer);
    }
    await this.forceFlush();
  }
}

// ==========================================
// ตัวอย่าง: Game Score Cache
// ==========================================
async function writeBehindExample(redis: Redis, db: Pool) {
  interface GameScore {
    id: number;
    userId: number;
    score: number;
    level: number;
  }

  const scoreCache = new WriteBehindStrategy<GameScore>(
    redis,
    db,
    'game_scores',
    {
      ttl: 7200,
      keyPrefix: 'score',
      flushIntervalMs: 5000,   // Flush ทุก 5 วินาที
      maxBatchSize: 100,
      maxRetries: 3,
      onFlush: (count) => console.log(`Flushed ${count} scores to DB`),
      onError: (error, key) => console.error(`Failed to flush ${key}:`, error),
    }
  );

  scoreCache.on('flushed', (count) => {
    console.log(`Successfully wrote ${count} items to database`);
  });

  // เขียนคะแนน - เร็วมาก (แค่เขียน Cache)
  for (let i = 1; i <= 10; i++) {
    await scoreCache.write(i, {
      id: i,
      userId: i,
      score: Math.floor(Math.random() * 10000),
      level: Math.floor(Math.random() * 100),
    });
  }

  console.log('Queue size:', scoreCache.getQueueSize());

  // หยุดการทำงาน - Force Flush
  await scoreCache.shutdown();
}

export { WriteBehindStrategy };
```

### 4.3 ข้อดี/ข้อเสีย

```
ข้อดี:
✅ Write เร็วที่สุด (แค่เขียน Cache)
✅ Batch Writes ประหยัด DB Connections
✅ Absorb Write Spikes

ข้อเสีย:
❌ ความเสี่ยงสูงสุด: ถ้า Cache ล่มก่อน Flush → Data Loss
❌ ซับซ้อนกว่า
❌ อาจมี Inconsistency ระหว่าง Cache และ DB ชั่วคราว
```

---

## 5. Refresh-Ahead

### 5.1 Concept

```
ปกติ (Cache-Aside):
Key Expire → Miss → Slow First Request

Refresh-Ahead:
Key กำลังจะ Expire → Background Refresh → ไม่มี Miss!

Timeline:
t=0   ─── Cache SET (TTL=60s)
t=45  ─── Refresh Threshold (75% of TTL)
          → Background: ดึงข้อมูลใหม่แบบ Async
t=60  ─── Key Expire
          → แต่ Cache ใหม่พร้อมอยู่แล้ว!
```

### 5.2 Implementation

```typescript
// src/cache/RefreshAhead.ts
import { Redis } from 'ioredis';

interface RefreshAheadOptions {
  ttl: number;
  refreshThreshold: number;   // 0-1, เช่น 0.75 = refresh เมื่อเหลือ 25% TTL
  keyPrefix: string;
}

class RefreshAheadCache<T = unknown> {
  private refreshing: Set<string> = new Set();

  constructor(
    private readonly redis: Redis,
    private readonly options: RefreshAheadOptions
  ) {}

  async get(key: string, fetcher: () => Promise<T | null>): Promise<T | null> {
    const cacheKey = `${this.options.keyPrefix}:${key}`;

    const [cached, remainingTtl] = await Promise.all([
      this.redis.get(cacheKey),
      this.redis.ttl(cacheKey),
    ]);

    if (cached) {
      const refreshThresholdTtl = this.options.ttl * (1 - this.options.refreshThreshold);

      // ถ้า TTL เหลือน้อย → Refresh ใน Background
      if (remainingTtl < refreshThresholdTtl && !this.refreshing.has(key)) {
        this.refreshInBackground(key, cacheKey, fetcher);
      }

      return JSON.parse(cached) as T;
    }

    // Cache Miss: ดึงข้อมูลและเก็บ Cache
    const data = await fetcher();
    if (data) {
      await this.redis.set(
        cacheKey,
        JSON.stringify(data),
        'EX',
        this.options.ttl
      );
    }

    return data;
  }

  private refreshInBackground(
    key: string,
    cacheKey: string,
    fetcher: () => Promise<T | null>
  ): void {
    this.refreshing.add(key);

    fetcher()
      .then(async (data) => {
        if (data) {
          await this.redis.set(
            cacheKey,
            JSON.stringify(data),
            'EX',
            this.options.ttl
          );
        }
      })
      .catch((err) => {
        console.error(`Refresh-ahead failed for ${key}:`, err);
      })
      .finally(() => {
        this.refreshing.delete(key);
      });
  }
}

export { RefreshAheadCache };
```

---

## 6. Cache Key Design

### 6.1 Naming Conventions

```typescript
// Cache Key Patterns

// ==========================================
// Entity:ID Pattern
// ==========================================
const keys = {
  user: (id: number) => `user:${id}`,
  userProfile: (id: number) => `user:${id}:profile`,
  userPosts: (id: number) => `user:${id}:posts`,
  post: (id: number) => `post:${id}`,
  postComments: (id: number, page: number) => `post:${id}:comments:page:${page}`,
};

// ==========================================
// Versioning: เมื่อ Schema เปลี่ยน
// ==========================================
const versionedKeys = {
  user: (id: number) => `v2:user:${id}`,       // v2 = version
  post: (id: number) => `v3:post:${id}`,
};

// ==========================================
// Namespace Separation
// ==========================================
const namespaced = {
  // Development
  dev: {
    user: (id: number) => `dev:user:${id}`,
  },
  // Production
  prod: {
    user: (id: number) => `prod:user:${id}`,
  },
  // By Feature
  auth: {
    session: (sessionId: string) => `auth:session:${sessionId}`,
    token: (userId: number) => `auth:token:${userId}`,
  },
  catalog: {
    product: (id: number) => `catalog:product:${id}`,
    category: (id: number) => `catalog:category:${id}`,
  },
};

// ==========================================
// Complex Key with Parameters
// ==========================================
const queryKeys = {
  // List with Pagination and Sort
  users: (page: number, limit: number, sort: string) =>
    `users:list:page:${page}:limit:${limit}:sort:${sort}`,

  // Search Result
  search: (query: string, page: number) =>
    `search:${encodeURIComponent(query)}:page:${page}`,

  // Aggregate Data
  dashboardStats: (date: string) => `stats:dashboard:${date}`,
};
```

### 6.2 Key Length Best Practice

```typescript
// ✅ ดี: สั้น กระชับ
'user:1001'
'post:5023:comments'

// ❌ ไม่ดี: ยาวเกินไป
'application_user_profile_data_for_user_with_id_1001_including_all_fields'

// ✅ Hash ถ้า Key ยาวมาก
import crypto from 'crypto';
function hashKey(longKey: string): string {
  return crypto.createHash('md5').update(longKey).digest('hex').substring(0, 16);
}
// 'srch:a3f2b1c4d5e6f7a8'
```

---

## 7. TTL Strategy

### 7.1 เลือก TTL อย่างไร

```typescript
// TTL Guidelines

const TTL_CONFIG = {
  // ข้อมูลที่เปลี่ยนบ่อย
  RATE_LIMIT: 1,          // 1 second
  SESSION: 1800,          // 30 minutes
  USER_PROFILE: 3600,     // 1 hour
  
  // ข้อมูลที่เปลี่ยนปานกลาง
  PRODUCT_DETAIL: 3600,   // 1 hour
  CATEGORY_LIST: 86400,   // 1 day
  
  // ข้อมูลที่เปลี่ยนน้อย
  CONFIG: 86400,          // 1 day
  STATIC_DATA: 604800,    // 1 week
  
  // ข้อมูลที่ไม่เปลี่ยน
  ENUM_VALUES: 0,         // ไม่ Expire (ต้อง Invalidate เอง)
};
```

### 7.2 Sliding Expiration

```typescript
// Sliding Expiration: รีเซ็ต TTL ทุกครั้งที่ Access
async function getWithSlidingExpiration<T>(
  redis: Redis,
  key: string,
  ttl: number,
  fetcher: () => Promise<T>
): Promise<T> {
  const cached = await redis.get(key);

  if (cached) {
    // รีเซ็ต TTL ทุกครั้งที่ Hit
    await redis.expire(key, ttl);
    return JSON.parse(cached);
  }

  const data = await fetcher();
  await redis.set(key, JSON.stringify(data), 'EX', ttl);
  return data;
}
```

### 7.3 Jitter TTL (ป้องกัน Cache Stampede)

```typescript
// เพิ่ม Random Jitter ใน TTL เพื่อกระจาย Expiration
function jitterTtl(baseTtl: number, jitterPercent: number = 0.1): number {
  const jitter = baseTtl * jitterPercent;
  return Math.floor(baseTtl + (Math.random() * 2 - 1) * jitter);
}

// ตัวอย่าง: TTL = 3600s ± 10%
const ttl = jitterTtl(3600);  // 3240 - 3960 วินาที
```

---

## 8. Cache Stampede / Thundering Herd

### 8.1 ปัญหา

```
t=0    Cache Key Expire
t=0.1  Request 1: Miss → กำลัง Query DB...
t=0.1  Request 2: Miss → กำลัง Query DB...
t=0.1  Request 3: Miss → กำลัง Query DB...
...
t=0.1  Request 1000: Miss → กำลัง Query DB... 💥

→ DB ได้รับ 1000 requests พร้อมกัน!
```

### 8.2 Solution 1: Mutex Lock (Redis)

```typescript
// src/cache/StampedeProtection.ts
import { Redis } from 'ioredis';

class MutexCache<T = unknown> {
  private readonly lockTtl: number;

  constructor(
    private readonly redis: Redis,
    private readonly keyPrefix: string,
    options: { lockTtl?: number } = {}
  ) {
    this.lockTtl = options.lockTtl ?? 10000; // 10 seconds
  }

  async get(
    key: string,
    fetcher: () => Promise<T | null>,
    ttl: number
  ): Promise<T | null> {
    const cacheKey = `${this.keyPrefix}:${key}`;
    const lockKey = `lock:${this.keyPrefix}:${key}`;

    // ตรวจ Cache ก่อน
    const cached = await this.redis.get(cacheKey);
    if (cached) {
      return JSON.parse(cached) as T;
    }

    // ลอง Acquire Lock
    const lockId = `${Date.now()}-${Math.random()}`;
    const acquired = await this.redis.set(
      lockKey,
      lockId,
      'PX', this.lockTtl,
      'NX'
    );

    if (acquired === 'OK') {
      // ได้ Lock → เราคือคนที่ต้องดึงข้อมูล
      try {
        const data = await fetcher();
        if (data) {
          await this.redis.set(cacheKey, JSON.stringify(data), 'EX', ttl);
        }
        return data;
      } finally {
        // Release Lock
        const script = `
          if redis.call("get", KEYS[1]) == ARGV[1] then
            return redis.call("del", KEYS[1])
          end
        `;
        await (this.redis as any).eval(script, 1, lockKey, lockId);
      }
    } else {
      // ไม่ได้ Lock → รอให้คนอื่น Fetch แล้ว Retry
      return this.waitAndGet(cacheKey, 50, 20);
    }
  }

  private async waitAndGet(
    key: string,
    delayMs: number,
    maxAttempts: number
  ): Promise<T | null> {
    for (let i = 0; i < maxAttempts; i++) {
      await new Promise(resolve => setTimeout(resolve, delayMs));

      const cached = await this.redis.get(key);
      if (cached) {
        return JSON.parse(cached) as T;
      }
    }
    return null; // Timeout
  }
}
```

### 8.3 Solution 2: Probabilistic Early Expiration

```typescript
// XFetch Algorithm: เริ่ม Refresh ก่อน Expire ตาม Probability
async function xFetch<T>(
  redis: Redis,
  key: string,
  ttl: number,
  fetcher: () => Promise<T>,
  beta: number = 1.0  // Higher beta = ยิ่ง Refresh เร็ว
): Promise<T> {
  const now = Date.now() / 1000;

  // ดึง cached value และ metadata
  const metaKey = `${key}:meta`;
  const [cached, metaStr] = await Promise.all([
    redis.get(key),
    redis.get(metaKey),
  ]);

  if (cached && metaStr) {
    const meta = JSON.parse(metaStr) as { recomputeTime: number; expiresAt: number };
    const remainingTtl = meta.expiresAt - now;

    // Probabilistic Check: ยิ่ง TTL เหลือน้อย → Probability สูงขึ้น
    const shouldRefresh = now - beta * meta.recomputeTime * Math.log(Math.random()) > meta.expiresAt;

    if (!shouldRefresh) {
      return JSON.parse(cached) as T;
    }
  }

  // Recompute
  const start = Date.now() / 1000;
  const data = await fetcher();
  const recomputeTime = Date.now() / 1000 - start;

  const meta = {
    recomputeTime,
    expiresAt: now + ttl,
  };

  await Promise.all([
    redis.set(key, JSON.stringify(data), 'EX', ttl),
    redis.set(metaKey, JSON.stringify(meta), 'EX', ttl),
  ]);

  return data;
}
```

### 8.4 Solution 3: Distributed Lock ด้วย Redlock

```typescript
// npm install redlock
import Redlock from 'redlock';
import Redis from 'ioredis';

const redis = new Redis();
const redlock = new Redlock([redis], {
  driftFactor: 0.01,
  retryCount: 10,
  retryDelay: 200,
  retryJitter: 200,
});

async function getWithRedlock<T>(
  key: string,
  fetcher: () => Promise<T>,
  ttl: number
): Promise<T | null> {
  const cached = await redis.get(key);
  if (cached) return JSON.parse(cached);

  // ใช้ Redlock สำหรับ Distributed Lock
  let lock;
  try {
    lock = await redlock.acquire([`lock:${key}`], 5000);

    // Double-check หลังได้ Lock
    const cachedAfterLock = await redis.get(key);
    if (cachedAfterLock) return JSON.parse(cachedAfterLock);

    const data = await fetcher();
    if (data) {
      await redis.set(key, JSON.stringify(data), 'EX', ttl);
    }
    return data;

  } finally {
    if (lock) {
      await lock.release().catch(() => {});
    }
  }
}
```

---

## 9. Cache Warming

### 9.1 Concept

```
Cold Start Problem:
- Deploy ใหม่ → Cache ว่าง
- Traffic พุ่ง → Cache Miss ทุกอัน → DB ระเบิด

Cache Warming:
- ก่อน Deploy เสร็จ → Pre-populate Cache
- Traffic พุ่ง → Cache Hit ส่วนใหญ่ → DB ไม่ระเบิด
```

### 9.2 Implementation

```typescript
// src/cache/CacheWarmer.ts
import { Redis } from 'ioredis';
import { Pool } from 'pg';

interface WarmupConfig {
  batchSize: number;
  concurrency: number;
  onProgress?: (current: number, total: number) => void;
}

class CacheWarmer {
  constructor(
    private readonly redis: Redis,
    private readonly db: Pool
  ) {}

  async warmUsers(config: WarmupConfig): Promise<void> {
    console.log('Starting cache warm-up for users...');

    // ดึง ID ทั้งหมดที่ต้อง Warm
    const result = await this.db.query<{ id: number }>(
      'SELECT id FROM users ORDER BY updated_at DESC LIMIT $1',
      [10000]  // Top 10,000 most recently updated
    );

    const userIds = result.rows.map(r => r.id);
    const total = userIds.length;

    // Process แบบ Batch + Concurrency Control
    for (let i = 0; i < userIds.length; i += config.batchSize) {
      const batch = userIds.slice(i, i + config.batchSize);

      // Process batch with concurrency limit
      await this.processBatchWithConcurrency(
        batch,
        config.concurrency,
        async (id) => {
          const userData = await this.db.query(
            'SELECT id, name, email, created_at FROM users WHERE id = $1',
            [id]
          );

          if (userData.rows[0]) {
            await this.redis.set(
              `user:${id}`,
              JSON.stringify(userData.rows[0]),
              'EX',
              3600
            );
          }
        }
      );

      config.onProgress?.(Math.min(i + config.batchSize, total), total);
    }

    console.log(`Cache warm-up complete: ${total} users cached`);
  }

  async warmProducts(config: WarmupConfig): Promise<void> {
    const result = await this.db.query<{ id: number }>(
      `SELECT id FROM products 
       WHERE active = true 
       ORDER BY view_count DESC 
       LIMIT 5000`
    );

    const productIds = result.rows.map(r => r.id);

    await this.processBatchWithConcurrency(
      productIds,
      config.concurrency,
      async (id) => {
        const productData = await this.db.query(
          `SELECT p.*, c.name as category_name
           FROM products p
           JOIN categories c ON p.category_id = c.id
           WHERE p.id = $1`,
          [id]
        );

        if (productData.rows[0]) {
          await this.redis.set(
            `product:${id}`,
            JSON.stringify(productData.rows[0]),
            'EX',
            7200
          );
        }
      }
    );
  }

  private async processBatchWithConcurrency<T>(
    items: T[],
    concurrency: number,
    processor: (item: T) => Promise<void>
  ): Promise<void> {
    const chunks: T[][] = [];

    for (let i = 0; i < items.length; i += concurrency) {
      chunks.push(items.slice(i, i + concurrency));
    }

    for (const chunk of chunks) {
      await Promise.all(chunk.map(processor));
    }
  }

  async warmAll(config: WarmupConfig): Promise<void> {
    const startTime = Date.now();

    await Promise.all([
      this.warmUsers(config),
      this.warmProducts(config),
    ]);

    const elapsed = ((Date.now() - startTime) / 1000).toFixed(2);
    console.log(`All cache warm-up complete in ${elapsed}s`);
  }
}

// ==========================================
// ใช้ตอน Application Start
// ==========================================
async function warmupOnStart(redis: Redis, db: Pool) {
  const warmer = new CacheWarmer(redis, db);

  await warmer.warmAll({
    batchSize: 100,
    concurrency: 10,
    onProgress: (current, total) => {
      const percent = ((current / total) * 100).toFixed(1);
      process.stdout.write(`\rWarming cache: ${current}/${total} (${percent}%)`);
    },
  });

  console.log('\nServer ready!');
}

export { CacheWarmer };
```

---

## 10. Full TypeScript Implementation: All Strategies

```typescript
// src/cache/UniversalCacheManager.ts
import { Redis } from 'ioredis';
import { Pool } from 'pg';

type CacheStrategy = 'cache-aside' | 'write-through' | 'write-behind' | 'refresh-ahead';

interface UniversalCacheOptions {
  strategy: CacheStrategy;
  ttl: number;
  keyPrefix: string;
  stampede?: {
    enabled: boolean;
    lockTtl?: number;
  };
  refreshAhead?: {
    threshold: number;  // 0-1
  };
  writeBehind?: {
    flushInterval: number;
    maxBatchSize: number;
  };
}

class UniversalCacheManager<T extends { id: number | string }> {
  private writeQueue: Map<string, T> = new Map();
  private flushTimer?: NodeJS.Timeout;

  constructor(
    private readonly redis: Redis,
    private readonly db: Pool,
    private readonly tableName: string,
    private readonly options: UniversalCacheOptions
  ) {
    if (options.strategy === 'write-behind' && options.writeBehind) {
      this.startWriteBehindFlush(options.writeBehind.flushInterval);
    }
  }

  async read(id: number | string, fetcher: () => Promise<T | null>): Promise<T | null> {
    const key = `${this.options.keyPrefix}:${id}`;

    switch (this.options.strategy) {
      case 'refresh-ahead':
        return this.readWithRefreshAhead(key, fetcher);
      default:
        return this.readCacheAside(key, fetcher);
    }
  }

  async write(id: number | string, data: T): Promise<void> {
    const key = `${this.options.keyPrefix}:${id}`;

    switch (this.options.strategy) {
      case 'write-through':
        await this.writeThrough(key, data);
        break;
      case 'write-behind':
        await this.writeBehind(key, data);
        break;
      default:
        // Cache-Aside: เขียน DB ก่อน แล้ว Invalidate Cache
        await this.db.query(
          `UPDATE ${this.tableName} SET updated_at = NOW() WHERE id = $1`,
          [id]
        );
        await this.redis.del(key);
    }
  }

  private async readCacheAside(
    key: string,
    fetcher: () => Promise<T | null>
  ): Promise<T | null> {
    if (this.options.stampede?.enabled) {
      return this.readWithMutex(key, fetcher);
    }

    const cached = await this.redis.get(key);
    if (cached) return JSON.parse(cached) as T;

    const data = await fetcher();
    if (data) {
      await this.redis.set(key, JSON.stringify(data), 'EX', this.options.ttl);
    }
    return data;
  }

  private async readWithMutex(
    key: string,
    fetcher: () => Promise<T | null>
  ): Promise<T | null> {
    const cached = await this.redis.get(key);
    if (cached) return JSON.parse(cached) as T;

    const lockKey = `lock:${key}`;
    const lockId = `${Date.now()}-${Math.random()}`;
    const lockTtl = this.options.stampede?.lockTtl ?? 10000;

    const acquired = await this.redis.set(lockKey, lockId, 'PX', lockTtl, 'NX');

    if (acquired === 'OK') {
      try {
        const data = await fetcher();
        if (data) {
          await this.redis.set(key, JSON.stringify(data), 'EX', this.options.ttl);
        }
        return data;
      } finally {
        await this.redis.del(lockKey);
      }
    }

    // รอและ Retry
    await new Promise(r => setTimeout(r, 100));
    const retryValue = await this.redis.get(key);
    return retryValue ? JSON.parse(retryValue) : null;
  }

  private async readWithRefreshAhead(
    key: string,
    fetcher: () => Promise<T | null>
  ): Promise<T | null> {
    const threshold = this.options.refreshAhead?.threshold ?? 0.75;

    const [cached, remainingTtl] = await Promise.all([
      this.redis.get(key),
      this.redis.ttl(key),
    ]);

    if (cached) {
      const refreshThresholdTtl = this.options.ttl * (1 - threshold);
      if (remainingTtl < refreshThresholdTtl) {
        fetcher().then(async (data) => {
          if (data) {
            await this.redis.set(key, JSON.stringify(data), 'EX', this.options.ttl);
          }
        }).catch(console.error);
      }
      return JSON.parse(cached) as T;
    }

    const data = await fetcher();
    if (data) {
      await this.redis.set(key, JSON.stringify(data), 'EX', this.options.ttl);
    }
    return data;
  }

  private async writeThrough(key: string, data: T): Promise<void> {
    await Promise.all([
      this.redis.set(key, JSON.stringify(data), 'EX', this.options.ttl),
      this.db.query(
        `UPDATE ${this.tableName} SET data = $1, updated_at = NOW() WHERE id = $2`,
        [JSON.stringify(data), (data as any).id]
      ),
    ]);
  }

  private async writeBehind(key: string, data: T): Promise<void> {
    await this.redis.set(key, JSON.stringify(data), 'EX', this.options.ttl);
    this.writeQueue.set(key, data);
  }

  private startWriteBehindFlush(interval: number): void {
    this.flushTimer = setInterval(async () => {
      if (this.writeQueue.size === 0) return;

      const maxBatch = this.options.writeBehind?.maxBatchSize ?? 100;
      const batch = Array.from(this.writeQueue.values()).slice(0, maxBatch);
      const keys = batch.map(d => `${this.options.keyPrefix}:${(d as any).id}`);

      const client = await this.db.connect();
      try {
        await client.query('BEGIN');
        for (const data of batch) {
          await client.query(
            `UPDATE ${this.tableName} SET updated_at = NOW() WHERE id = $1`,
            [(data as any).id]
          );
        }
        await client.query('COMMIT');
        keys.forEach(k => this.writeQueue.delete(k));
      } catch (err) {
        await client.query('ROLLBACK');
        console.error('Write-behind flush error:', err);
      } finally {
        client.release();
      }
    }, interval);
  }

  async invalidate(id: number | string): Promise<void> {
    const key = `${this.options.keyPrefix}:${id}`;
    await this.redis.del(key);
  }

  async shutdown(): Promise<void> {
    if (this.flushTimer) clearInterval(this.flushTimer);
  }
}

export { UniversalCacheManager };
```

---

## สรุปเปรียบเทียบ Caching Strategies

| Strategy | Read Speed | Write Speed | Consistency | Complexity | Use Case |
|----------|-----------|------------|-------------|------------|----------|
| **Cache-Aside** | เร็ว (hit) / ช้า (miss) | ปกติ | ปานกลาง | ต่ำ | ข้อมูลทั่วไป |
| **Read-Through** | เร็ว | ปกติ | ปานกลาง | ต่ำ | เหมือน Cache-Aside แต่ Cache จัดการเอง |
| **Write-Through** | เร็ว | ช้า | สูง | ปานกลาง | ข้อมูลสำคัญ Read บ่อย |
| **Write-Behind** | เร็ว | เร็วมาก | ต่ำ | สูง | Write-Heavy, ยอมรับ Lag |
| **Refresh-Ahead** | เร็วมาก | ปกติ | ปานกลาง | ปานกลาง | Hot Data ที่ต้องเร็ว |

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Cache-Aside** - อ่าน Cache ก่อน Miss → DB → Store
2. **Read-Through** - Cache จัดการ Miss เอง
3. **Write-Through** - เขียน Cache + DB พร้อมกัน
4. **Write-Behind** - เขียน Cache ก่อน DB ทีหลัง
5. **Refresh-Ahead** - Refresh ก่อน Expire
6. **Cache Key Design** - Naming Conventions, Versioning
7. **TTL Strategy** - เลือก TTL อย่างไร, Jitter
8. **Cache Stampede** - Mutex, Probabilistic, Redlock
9. **Cache Warming** - Pre-populate หลัง Deploy
10. **UniversalCacheManager** - Implementation ครบทุก Strategy

เลือก Strategy ที่เหมาะสมกับ Use Case:
- **ส่วนใหญ่**: Cache-Aside เพราะ Simple และ Flexible
- **Consistency สำคัญ**: Write-Through
- **Write Performance**: Write-Behind
- **No Cold Start**: Refresh-Ahead

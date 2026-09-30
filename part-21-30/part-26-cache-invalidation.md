# Part 26: Cache Invalidation Patterns

## "One of the Two Hard Things in Computer Science"

Phil Karlton กล่าวไว้ว่า:

> "There are only two hard things in Computer Science: cache invalidation and naming things."

Cache invalidation คือกระบวนการทำให้ cached data หมดอายุหรือไม่ valid เมื่อข้อมูลต้นทางเปลี่ยนแปลง ปัญหาหลักคือ:

1. **Stale data**: Cache ยังเก็บข้อมูลเก่าอยู่ ทั้งที่ข้อมูลจริงเปลี่ยนแปลงไปแล้ว
2. **Race conditions**: Multiple processes invalidate cache พร้อมกัน
3. **Cascading invalidations**: การ invalidate หนึ่ง entry ควร invalidate entries อื่นที่เกี่ยวข้องด้วย
4. **Distributed caches**: ต้อง invalidate ใน cache nodes หลายตัวพร้อมกัน

---

## Cache Invalidation Strategies

### Strategy 1: TTL-Based (Time-To-Live)

วิธีที่ง่ายที่สุด — กำหนดอายุของ cache entry ให้หมดอายุอัตโนมัติ

```typescript
// TTL-based cache ใน Redis
import Redis from 'ioredis';

const redis = new Redis({
  host: 'localhost',
  port: 6379,
});

async function cacheWithTTL(key: string, data: any, ttlSeconds: number): Promise<void> {
  await redis.setex(key, ttlSeconds, JSON.stringify(data));
}

async function getFromCache(key: string): Promise<any | null> {
  const cached = await redis.get(key);
  if (!cached) return null;
  return JSON.parse(cached);
}

// ตัวอย่างการใช้งาน
async function getUserProfile(userId: string): Promise<User> {
  const cacheKey = `user:${userId}:profile`;
  
  // ลอง cache ก่อน
  const cached = await getFromCache(cacheKey);
  if (cached) {
    console.log('Cache HIT');
    return cached;
  }
  
  // ถ้าไม่มีใน cache ดึงจาก DB
  console.log('Cache MISS');
  const user = await db.query('SELECT * FROM users WHERE id = $1', [userId]);
  
  // เก็บใน cache 5 นาที
  await cacheWithTTL(cacheKey, user, 300);
  
  return user;
}
```

**ข้อดี TTL-based:**
- ง่ายมาก ไม่ต้องมี logic พิเศษ
- Cache จะ self-heal เมื่อ TTL หมด
- เหมาะกับข้อมูลที่รับ stale ได้บ้าง

**ข้อเสีย TTL-based:**
- ข้อมูลอาจ stale ตลอด TTL ที่เหลืออยู่
- TTL สั้นเกินไป = cache miss บ่อย = ดึง DB บ่อย
- TTL ยาวเกินไป = stale data นาน

```typescript
// Stale-While-Revalidate pattern
interface CacheEntry<T> {
  data: T;
  timestamp: number;
  ttl: number;       // เวลาที่ยอมรับได้ (fresh)
  staleTtl: number;  // เวลาที่ยอมรับ stale data ระหว่าง revalidate
}

class StaleWhileRevalidateCache {
  private redis: Redis;
  private revalidating = new Set<string>();

  constructor(redis: Redis) {
    this.redis = redis;
  }

  async get<T>(
    key: string,
    fetcher: () => Promise<T>,
    ttl: number,
    staleTtl: number
  ): Promise<T> {
    const raw = await this.redis.get(key);
    const now = Date.now();

    if (raw) {
      const entry: CacheEntry<T> = JSON.parse(raw);
      const age = now - entry.timestamp;
      const ttlMs = entry.ttl * 1000;
      const staleTtlMs = entry.staleTtl * 1000;

      if (age < ttlMs) {
        // ยังสด คืนค่าทันที
        return entry.data;
      }

      if (age < staleTtlMs && !this.revalidating.has(key)) {
        // Stale แต่ยังใช้ได้ คืนค่าเก่าแล้ว revalidate ใน background
        this.revalidating.add(key);
        this.revalidateInBackground(key, fetcher, ttl, staleTtl)
          .finally(() => this.revalidating.delete(key));
        return entry.data;
      }
    }

    // ไม่มีใน cache หรือ stale เกินไป ดึงใหม่
    return this.fetchAndCache(key, fetcher, ttl, staleTtl);
  }

  private async revalidateInBackground<T>(
    key: string,
    fetcher: () => Promise<T>,
    ttl: number,
    staleTtl: number
  ): Promise<void> {
    try {
      await this.fetchAndCache(key, fetcher, ttl, staleTtl);
    } catch (error) {
      console.error(`Background revalidation failed for ${key}:`, error);
    }
  }

  private async fetchAndCache<T>(
    key: string,
    fetcher: () => Promise<T>,
    ttl: number,
    staleTtl: number
  ): Promise<T> {
    const data = await fetcher();
    const entry: CacheEntry<T> = {
      data,
      timestamp: Date.now(),
      ttl,
      staleTtl,
    };
    await this.redis.setex(key, staleTtl, JSON.stringify(entry));
    return data;
  }
}
```

---

### Strategy 2: Event-Based Invalidation

Invalidate cache ทันทีเมื่อข้อมูลเปลี่ยนแปลง

#### 2.1 Application-Level Invalidation

```typescript
// Service layer invalidation
class UserService {
  private redis: Redis;
  private db: Pool;

  constructor(redis: Redis, db: Pool) {
    this.redis = redis;
    this.db = db;
  }

  async updateUser(userId: string, data: Partial<User>): Promise<User> {
    // 1. อัพเดต DB
    const updated = await this.db.query(
      'UPDATE users SET name=$1, email=$2, updated_at=NOW() WHERE id=$3 RETURNING *',
      [data.name, data.email, userId]
    );

    // 2. Invalidate cache ทันที
    await this.invalidateUserCache(userId);

    return updated.rows[0];
  }

  private async invalidateUserCache(userId: string): Promise<void> {
    const keys = [
      `user:${userId}:profile`,
      `user:${userId}:preferences`,
      `user:${userId}:stats`,
    ];
    
    // Delete ทุก keys ที่เกี่ยวข้อง
    if (keys.length > 0) {
      await this.redis.del(...keys);
    }
    
    console.log(`Invalidated cache for user ${userId}:`, keys);
  }

  async getUserProfile(userId: string): Promise<User> {
    const key = `user:${userId}:profile`;
    const cached = await this.redis.get(key);
    
    if (cached) return JSON.parse(cached);
    
    const user = await this.db.query(
      'SELECT * FROM users WHERE id = $1',
      [userId]
    );
    
    await this.redis.setex(key, 3600, JSON.stringify(user.rows[0]));
    return user.rows[0];
  }
}
```

#### 2.2 Database-Level Invalidation ด้วย pg_notify

ใช้ PostgreSQL trigger + pg_notify เพื่อส่ง event เมื่อข้อมูลเปลี่ยน

```sql
-- สร้าง function สำหรับส่ง notification
CREATE OR REPLACE FUNCTION notify_cache_invalidation()
RETURNS TRIGGER AS $$
DECLARE
  payload JSON;
BEGIN
  -- สร้าง payload พร้อมข้อมูลที่จำเป็น
  payload = json_build_object(
    'table', TG_TABLE_NAME,
    'operation', TG_OP,
    'id', CASE
      WHEN TG_OP = 'DELETE' THEN OLD.id
      ELSE NEW.id
    END,
    'timestamp', extract(epoch from now())
  );
  
  -- ส่ง notification ไปที่ channel 'cache_invalidation'
  PERFORM pg_notify('cache_invalidation', payload::text);
  
  RETURN CASE
    WHEN TG_OP = 'DELETE' THEN OLD
    ELSE NEW
  END;
END;
$$ LANGUAGE plpgsql;

-- Trigger บนตาราง users
CREATE TRIGGER users_cache_invalidation_trigger
  AFTER INSERT OR UPDATE OR DELETE ON users
  FOR EACH ROW
  EXECUTE FUNCTION notify_cache_invalidation();

-- Trigger บนตาราง products
CREATE TRIGGER products_cache_invalidation_trigger
  AFTER INSERT OR UPDATE OR DELETE ON products
  FOR EACH ROW
  EXECUTE FUNCTION notify_cache_invalidation();

-- Trigger บนตาราง orders
CREATE TRIGGER orders_cache_invalidation_trigger
  AFTER INSERT OR UPDATE OR DELETE ON orders
  FOR EACH ROW
  EXECUTE FUNCTION notify_cache_invalidation();
```

```typescript
// Node.js listener สำหรับ pg_notify
import { Pool, Client } from 'pg';
import Redis from 'ioredis';

interface CacheInvalidationPayload {
  table: string;
  operation: 'INSERT' | 'UPDATE' | 'DELETE';
  id: string;
  timestamp: number;
}

class DatabaseCacheInvalidator {
  private client: Client;
  private redis: Redis;
  private invalidationRules: Map<string, (id: string) => string[]>;

  constructor(client: Client, redis: Redis) {
    this.client = client;
    this.redis = redis;
    this.invalidationRules = new Map();
    this.setupRules();
  }

  private setupRules(): void {
    // กำหนดว่าเมื่อตารางไหนเปลี่ยน ต้อง invalidate cache keys อะไรบ้าง
    this.invalidationRules.set('users', (id: string) => [
      `user:${id}:profile`,
      `user:${id}:preferences`,
      `user:${id}:stats`,
      `user:${id}:orders`,
    ]);

    this.invalidationRules.set('products', (id: string) => [
      `product:${id}:detail`,
      `product:${id}:inventory`,
      `products:list:*`,         // wildcard - ต้องใช้ SCAN
      `category:*:products`,    // wildcard
    ]);

    this.invalidationRules.set('orders', (id: string) => [
      `order:${id}:detail`,
      `order:${id}:items`,
    ]);
  }

  async startListening(): Promise<void> {
    await this.client.connect();
    
    // Listen ที่ channel
    await this.client.query('LISTEN cache_invalidation');
    
    console.log('Listening for cache invalidation events...');
    
    this.client.on('notification', async (msg) => {
      if (msg.channel === 'cache_invalidation' && msg.payload) {
        try {
          const payload: CacheInvalidationPayload = JSON.parse(msg.payload);
          await this.handleInvalidation(payload);
        } catch (error) {
          console.error('Error handling cache invalidation:', error);
        }
      }
    });

    this.client.on('error', (error) => {
      console.error('Database client error:', error);
    });
  }

  private async handleInvalidation(payload: CacheInvalidationPayload): Promise<void> {
    const { table, operation, id } = payload;
    console.log(`Cache invalidation: ${operation} on ${table} id=${id}`);

    const getRules = this.invalidationRules.get(table);
    if (!getRules) {
      console.warn(`No invalidation rules for table: ${table}`);
      return;
    }

    const keys = getRules(id);
    const exactKeys: string[] = [];
    const patternKeys: string[] = [];

    keys.forEach(key => {
      if (key.includes('*')) {
        patternKeys.push(key);
      } else {
        exactKeys.push(key);
      }
    });

    // Delete exact keys
    if (exactKeys.length > 0) {
      await this.redis.del(...exactKeys);
      console.log(`Deleted exact keys:`, exactKeys);
    }

    // Scan และ delete pattern keys
    for (const pattern of patternKeys) {
      await this.deleteByPattern(pattern);
    }
  }

  private async deleteByPattern(pattern: string): Promise<void> {
    let cursor = '0';
    let totalDeleted = 0;
    
    do {
      const result = await this.redis.scan(cursor, 'MATCH', pattern, 'COUNT', 100);
      cursor = result[0];
      const keys = result[1];
      
      if (keys.length > 0) {
        await this.redis.del(...keys);
        totalDeleted += keys.length;
      }
    } while (cursor !== '0');
    
    if (totalDeleted > 0) {
      console.log(`Deleted ${totalDeleted} keys matching pattern: ${pattern}`);
    }
  }
}

// Main
async function main() {
  const pgClient = new Client({
    connectionString: process.env.DATABASE_URL,
  });
  
  const redis = new Redis(process.env.REDIS_URL);
  
  const invalidator = new DatabaseCacheInvalidator(pgClient, redis);
  await invalidator.startListening();
}
```

#### 2.3 Message Queue: Redis Pub/Sub สำหรับ Cache Invalidation

```typescript
// Publisher: เมื่อข้อมูลเปลี่ยน publish event
class CacheInvalidationPublisher {
  private publisher: Redis;

  constructor() {
    this.publisher = new Redis(process.env.REDIS_URL);
  }

  async publishInvalidation(event: {
    type: string;
    entityId: string;
    entityType: string;
  }): Promise<void> {
    const channel = 'cache:invalidation';
    const message = JSON.stringify({
      ...event,
      timestamp: Date.now(),
      nodeId: process.env.NODE_ID || 'node-1',
    });
    
    await this.publisher.publish(channel, message);
    console.log(`Published invalidation event:`, event);
  }
}

// Subscriber: ฟัง event และ invalidate cache
class CacheInvalidationSubscriber {
  private subscriber: Redis;
  private cacheClient: Redis;

  constructor() {
    this.subscriber = new Redis(process.env.REDIS_URL);
    this.cacheClient = new Redis(process.env.REDIS_URL);
  }

  async startListening(): Promise<void> {
    await this.subscriber.subscribe('cache:invalidation');
    
    this.subscriber.on('message', async (channel, message) => {
      if (channel === 'cache:invalidation') {
        try {
          const event = JSON.parse(message);
          
          // ไม่ invalidate ถ้า event มาจาก node เดียวกัน (optional)
          // if (event.nodeId === process.env.NODE_ID) return;
          
          await this.handleInvalidationEvent(event);
        } catch (error) {
          console.error('Error processing invalidation event:', error);
        }
      }
    });
  }

  private async handleInvalidationEvent(event: {
    type: string;
    entityId: string;
    entityType: string;
    nodeId: string;
  }): Promise<void> {
    const { entityType, entityId, type } = event;
    
    switch (entityType) {
      case 'user':
        await this.invalidateUserCache(entityId);
        break;
      case 'product':
        await this.invalidateProductCache(entityId);
        break;
      case 'order':
        await this.invalidateOrderCache(entityId);
        break;
      default:
        console.warn(`Unknown entity type: ${entityType}`);
    }
  }

  private async invalidateUserCache(userId: string): Promise<void> {
    const keys = [
      `user:${userId}:profile`,
      `user:${userId}:stats`,
    ];
    await this.cacheClient.del(...keys);
  }

  private async invalidateProductCache(productId: string): Promise<void> {
    const keys = [
      `product:${productId}:detail`,
      `product:${productId}:price`,
    ];
    await this.cacheClient.del(...keys);
    
    // Invalidate product list caches
    await this.deleteByPattern('products:list:*');
  }

  private async invalidateOrderCache(orderId: string): Promise<void> {
    const keys = [
      `order:${orderId}:detail`,
    ];
    await this.cacheClient.del(...keys);
  }

  private async deleteByPattern(pattern: string): Promise<void> {
    let cursor = '0';
    do {
      const [newCursor, keys] = await this.cacheClient.scan(
        cursor, 'MATCH', pattern, 'COUNT', 100
      );
      cursor = newCursor;
      if (keys.length > 0) {
        await this.cacheClient.del(...keys);
      }
    } while (cursor !== '0');
  }
}
```

---

### Strategy 3: Version-Based Invalidation

ฝัง version ไว้ใน cache key ทำให้ key เปลี่ยนเมื่อ version เปลี่ยน

```typescript
class VersionedCache {
  private redis: Redis;
  private VERSION_KEY_PREFIX = 'version:';

  constructor(redis: Redis) {
    this.redis = redis;
  }

  // ดึง version ของ entity
  async getVersion(entityType: string, entityId?: string): Promise<number> {
    const versionKey = entityId
      ? `${this.VERSION_KEY_PREFIX}${entityType}:${entityId}`
      : `${this.VERSION_KEY_PREFIX}${entityType}`;
    
    const version = await this.redis.get(versionKey);
    return version ? parseInt(version) : 1;
  }

  // เพิ่ม version (invalidate ทุก cache ที่ใช้ version นี้)
  async incrementVersion(entityType: string, entityId?: string): Promise<number> {
    const versionKey = entityId
      ? `${this.VERSION_KEY_PREFIX}${entityType}:${entityId}`
      : `${this.VERSION_KEY_PREFIX}${entityType}`;
    
    const newVersion = await this.redis.incr(versionKey);
    console.log(`Incremented version for ${entityType}${entityId ? ':' + entityId : ''} to ${newVersion}`);
    return newVersion;
  }

  // สร้าง cache key พร้อม version
  async buildVersionedKey(
    baseKey: string,
    entityType: string,
    entityId?: string
  ): Promise<string> {
    const version = await this.getVersion(entityType, entityId);
    return `${baseKey}:v${version}`;
  }

  // Set cache ด้วย versioned key
  async set<T>(
    baseKey: string,
    entityType: string,
    entityId: string,
    data: T,
    ttl: number
  ): Promise<void> {
    const key = await this.buildVersionedKey(baseKey, entityType, entityId);
    await this.redis.setex(key, ttl, JSON.stringify(data));
  }

  // Get cache ด้วย versioned key
  async get<T>(
    baseKey: string,
    entityType: string,
    entityId: string
  ): Promise<T | null> {
    const key = await this.buildVersionedKey(baseKey, entityType, entityId);
    const data = await this.redis.get(key);
    return data ? JSON.parse(data) : null;
  }
}

// การใช้งาน
class ProductService {
  private cache: VersionedCache;
  private db: Pool;

  constructor(cache: VersionedCache, db: Pool) {
    this.cache = cache;
    this.db = db;
  }

  async getProduct(productId: string): Promise<Product> {
    // ลอง cache ก่อน
    const cached = await this.cache.get<Product>(
      'product:detail',
      'product',
      productId
    );
    
    if (cached) {
      console.log(`Cache HIT: product ${productId}`);
      return cached;
    }

    // ดึงจาก DB
    const result = await this.db.query(
      'SELECT * FROM products WHERE id = $1',
      [productId]
    );
    const product = result.rows[0];

    // เก็บใน cache
    await this.cache.set('product:detail', 'product', productId, product, 3600);
    return product;
  }

  async updateProduct(productId: string, data: Partial<Product>): Promise<Product> {
    // อัพเดต DB
    const result = await this.db.query(
      'UPDATE products SET name=$1, price=$2 WHERE id=$3 RETURNING *',
      [data.name, data.price, productId]
    );
    const updated = result.rows[0];

    // Increment version — ทำให้ cached key เก่า miss ทันที
    await this.cache.incrementVersion('product', productId);
    
    // ถ้ามี global product list version ก็ increment ด้วย
    await this.cache.incrementVersion('products');

    return updated;
  }
}
```

---

### Strategy 4: Tag-Based Cache Invalidation

Group cache entries ด้วย tags แล้ว invalidate ทั้ง group พร้อมกัน

```typescript
class TagBasedCache {
  private redis: Redis;
  private TAG_PREFIX = 'tag:';
  private CACHE_PREFIX = 'cache:';

  constructor(redis: Redis) {
    this.redis = redis;
  }

  // เก็บ cache entry พร้อม tags
  async set<T>(
    key: string,
    data: T,
    options: {
      ttl: number;
      tags: string[];
    }
  ): Promise<void> {
    const cacheKey = `${this.CACHE_PREFIX}${key}`;
    
    // เก็บ data
    await this.redis.setex(cacheKey, options.ttl, JSON.stringify(data));
    
    // เพิ่ม key เข้าใน sets ของแต่ละ tag
    const pipeline = this.redis.pipeline();
    for (const tag of options.tags) {
      const tagKey = `${this.TAG_PREFIX}${tag}`;
      pipeline.sadd(tagKey, cacheKey);
      // Tag set ควรมีอายุยาวกว่า cache entry
      pipeline.expire(tagKey, options.ttl * 2);
    }
    await pipeline.exec();
    
    console.log(`Cached ${key} with tags: ${options.tags.join(', ')}`);
  }

  // ดึง cache
  async get<T>(key: string): Promise<T | null> {
    const cacheKey = `${this.CACHE_PREFIX}${key}`;
    const data = await this.redis.get(cacheKey);
    return data ? JSON.parse(data) : null;
  }

  // Invalidate ทุก cache entries ที่มี tag นี้
  async invalidateByTag(tag: string): Promise<number> {
    const tagKey = `${this.TAG_PREFIX}${tag}`;
    
    // ดึง keys ทั้งหมดที่อยู่ใน tag set
    const keys = await this.redis.smembers(tagKey);
    
    if (keys.length === 0) {
      console.log(`No cache entries for tag: ${tag}`);
      return 0;
    }

    // Delete ทุก cache entries
    const pipeline = this.redis.pipeline();
    pipeline.del(...keys);  // delete cache entries
    pipeline.del(tagKey);   // delete tag set
    await pipeline.exec();
    
    console.log(`Invalidated ${keys.length} entries for tag: ${tag}`);
    return keys.length;
  }

  // Invalidate หลาย tags พร้อมกัน
  async invalidateByTags(tags: string[]): Promise<number> {
    let totalInvalidated = 0;
    
    for (const tag of tags) {
      totalInvalidated += await this.invalidateByTag(tag);
    }
    
    return totalInvalidated;
  }

  // ดู keys ที่อยู่ใน tag
  async getKeysByTag(tag: string): Promise<string[]> {
    const tagKey = `${this.TAG_PREFIX}${tag}`;
    return this.redis.smembers(tagKey);
  }
}

// ตัวอย่าง: Product Catalog Cache
class ProductCatalogService {
  private cache: TagBasedCache;
  private db: Pool;

  constructor(cache: TagBasedCache, db: Pool) {
    this.cache = cache;
    this.db = db;
  }

  async getProductsByCategory(categoryId: string): Promise<Product[]> {
    const key = `products:category:${categoryId}`;
    const cached = await this.cache.get<Product[]>(key);
    
    if (cached) return cached;
    
    const result = await this.db.query(
      'SELECT * FROM products WHERE category_id = $1 AND active = true',
      [categoryId]
    );
    
    await this.cache.set(key, result.rows, {
      ttl: 1800,  // 30 นาที
      tags: [
        `category:${categoryId}`,
        'products:list',
        'catalog',
      ],
    });
    
    return result.rows;
  }

  async getProductDetail(productId: string): Promise<Product> {
    const key = `product:${productId}:detail`;
    const cached = await this.cache.get<Product>(key);
    
    if (cached) return cached;
    
    const result = await this.db.query(
      `SELECT p.*, c.name as category_name, b.name as brand_name
       FROM products p
       JOIN categories c ON p.category_id = c.id
       JOIN brands b ON p.brand_id = b.id
       WHERE p.id = $1`,
      [productId]
    );
    
    const product = result.rows[0];
    
    await this.cache.set(key, product, {
      ttl: 3600,  // 1 ชั่วโมง
      tags: [
        `product:${productId}`,
        `category:${product.category_id}`,
        `brand:${product.brand_id}`,
        'catalog',
      ],
    });
    
    return product;
  }

  async updateProduct(productId: string, data: Partial<Product>): Promise<void> {
    // อัพเดต DB
    const result = await this.db.query(
      'UPDATE products SET name=$1, price=$2 WHERE id=$3 RETURNING category_id',
      [data.name, data.price, productId]
    );
    const { category_id } = result.rows[0];

    // Invalidate ด้วย product tag
    await this.cache.invalidateByTags([
      `product:${productId}`,
      `category:${category_id}`,
    ]);
  }

  async updateCategory(categoryId: string, data: any): Promise<void> {
    await this.db.query(
      'UPDATE categories SET name=$1 WHERE id=$2',
      [data.name, categoryId]
    );

    // Invalidate ทุก products ใน category นี้
    await this.cache.invalidateByTag(`category:${categoryId}`);
  }

  async deactivateAllProductsInCategory(categoryId: string): Promise<void> {
    await this.db.query(
      'UPDATE products SET active=false WHERE category_id=$1',
      [categoryId]
    );

    // Invalidate ทุกอย่างที่เกี่ยวกับ category นี้
    await this.cache.invalidateByTags([
      `category:${categoryId}`,
      'catalog',
    ]);
  }
}
```

---

## Distributed Cache Invalidation

เมื่อมี cache nodes หลายตัว ต้อง invalidate ทุก node พร้อมกัน

### Fan-Out Pattern

```typescript
// Invalidate หลาย cache servers พร้อมกัน
class DistributedCacheInvalidator {
  private cacheNodes: Redis[];

  constructor(cacheNodeUrls: string[]) {
    this.cacheNodes = cacheNodeUrls.map(url => new Redis(url));
  }

  // Fan-out: ส่ง invalidation ไปทุก node พร้อมกัน
  async invalidateAll(keys: string[]): Promise<void> {
    if (keys.length === 0) return;

    const invalidationPromises = this.cacheNodes.map(async (node, index) => {
      try {
        await node.del(...keys);
        console.log(`Invalidated ${keys.length} keys on node ${index + 1}`);
      } catch (error) {
        console.error(`Failed to invalidate on node ${index + 1}:`, error);
        // ไม่ throw error — node อื่นยังทำงานได้
      }
    });

    await Promise.allSettled(invalidationPromises);
  }

  // Fan-out ด้วย pattern
  async invalidatePattern(pattern: string): Promise<void> {
    const invalidationPromises = this.cacheNodes.map(async (node, index) => {
      try {
        let cursor = '0';
        let totalDeleted = 0;
        
        do {
          const [newCursor, keys] = await node.scan(
            cursor, 'MATCH', pattern, 'COUNT', 100
          );
          cursor = newCursor;
          if (keys.length > 0) {
            await node.del(...keys);
            totalDeleted += keys.length;
          }
        } while (cursor !== '0');
        
        console.log(`Node ${index + 1}: deleted ${totalDeleted} keys matching ${pattern}`);
      } catch (error) {
        console.error(`Failed to invalidate pattern on node ${index + 1}:`, error);
      }
    });

    await Promise.allSettled(invalidationPromises);
  }
}

// Redis Pub/Sub สำหรับ distributed invalidation
class RedisPubSubInvalidator {
  private publisher: Redis;
  private subscribers: Redis[];
  private localCache: Map<string, any>;

  constructor(redisUrl: string) {
    this.publisher = new Redis(redisUrl);
    this.subscribers = [];
    this.localCache = new Map();
  }

  // Setup subscriber บน cache node แต่ละตัว
  async setupSubscriber(cacheNode: Redis): Promise<void> {
    const subscriber = new Redis(process.env.REDIS_URL);
    this.subscribers.push(subscriber);

    await subscriber.subscribe('cache:invalidation:broadcast');
    
    subscriber.on('message', async (channel, message) => {
      const event = JSON.parse(message);
      
      switch (event.action) {
        case 'delete_key':
          await cacheNode.del(event.key);
          this.localCache.delete(event.key);
          break;
        case 'delete_pattern':
          await this.deletePattern(cacheNode, event.pattern);
          break;
        case 'flush_tag':
          await this.flushTag(cacheNode, event.tag);
          break;
      }
    });
  }

  // Broadcast invalidation ไปทุก subscribers
  async broadcastInvalidation(event: {
    action: 'delete_key' | 'delete_pattern' | 'flush_tag';
    key?: string;
    pattern?: string;
    tag?: string;
  }): Promise<void> {
    await this.publisher.publish(
      'cache:invalidation:broadcast',
      JSON.stringify(event)
    );
  }

  private async deletePattern(node: Redis, pattern: string): Promise<void> {
    let cursor = '0';
    do {
      const [newCursor, keys] = await node.scan(cursor, 'MATCH', pattern, 'COUNT', 100);
      cursor = newCursor;
      if (keys.length > 0) {
        await node.del(...keys);
      }
    } while (cursor !== '0');
  }

  private async flushTag(node: Redis, tag: string): Promise<void> {
    const tagKey = `tag:${tag}`;
    const keys = await node.smembers(tagKey);
    if (keys.length > 0) {
      await node.del(...keys, tagKey);
    }
  }
}
```

---

## Cache Versioning: Global Version Bump

```typescript
class GlobalVersionCache {
  private redis: Redis;
  private GLOBAL_VERSION_KEY = 'cache:global:version';

  constructor(redis: Redis) {
    this.redis = redis;
  }

  async getGlobalVersion(): Promise<number> {
    const version = await this.redis.get(this.GLOBAL_VERSION_KEY);
    return version ? parseInt(version) : 1;
  }

  async bumpGlobalVersion(): Promise<number> {
    const newVersion = await this.redis.incr(this.GLOBAL_VERSION_KEY);
    console.log(`Global cache version bumped to ${newVersion}`);
    return newVersion;
  }

  async buildKey(baseKey: string): Promise<string> {
    const version = await this.getGlobalVersion();
    return `v${version}:${baseKey}`;
  }

  async set<T>(key: string, data: T, ttl: number): Promise<void> {
    const versionedKey = await this.buildKey(key);
    await this.redis.setex(versionedKey, ttl, JSON.stringify(data));
  }

  async get<T>(key: string): Promise<T | null> {
    const versionedKey = await this.buildKey(key);
    const data = await this.redis.get(versionedKey);
    return data ? JSON.parse(data) : null;
  }

  // Invalidate ทุก cache ใน global ทันที (nuclear option)
  async invalidateAll(): Promise<void> {
    await this.bumpGlobalVersion();
    console.log('All caches invalidated via global version bump');
  }
}
```

---

## Dependency Tracking

```typescript
// ติดตาม dependencies ระหว่าง cache entries
class DependencyTrackingCache {
  private redis: Redis;
  private DEP_PREFIX = 'deps:';

  constructor(redis: Redis) {
    this.redis = redis;
  }

  // เก็บ cache พร้อม dependencies
  async set<T>(
    key: string,
    data: T,
    options: {
      ttl: number;
      dependsOn?: string[];  // keys ที่ key นี้ต้องการ
    }
  ): Promise<void> {
    await this.redis.setex(key, options.ttl, JSON.stringify(data));
    
    // บันทึก reverse dependencies: ถ้า dependsOn key เปลี่ยน ต้อง invalidate key นี้
    if (options.dependsOn) {
      for (const dep of options.dependsOn) {
        const depKey = `${this.DEP_PREFIX}${dep}`;
        await this.redis.sadd(depKey, key);
        await this.redis.expire(depKey, options.ttl * 2);
      }
    }
  }

  async get<T>(key: string): Promise<T | null> {
    const data = await this.redis.get(key);
    return data ? JSON.parse(data) : null;
  }

  // Invalidate key และทุก cache ที่ depend บน key นี้ (cascading)
  async invalidate(key: string, visited = new Set<string>()): Promise<void> {
    if (visited.has(key)) return;
    visited.add(key);

    // Delete key เอง
    await this.redis.del(key);
    
    // ดึง dependent keys
    const depKey = `${this.DEP_PREFIX}${key}`;
    const dependents = await this.redis.smembers(depKey);
    
    // Invalidate dependents แบบ recursive
    for (const dependent of dependents) {
      await this.invalidate(dependent, visited);
    }
    
    // ลบ dependency tracking
    await this.redis.del(depKey);
    
    console.log(`Invalidated ${key} and ${visited.size - 1} dependents`);
  }
}

// ตัวอย่าง: Shopping Cart ที่ depend บน Product และ User
const depCache = new DependencyTrackingCache(redis);

// เก็บ product data
await depCache.set('product:123:price', { price: 299.99 }, { ttl: 3600 });

// เก็บ user cart ที่ depend บน product price
await depCache.set(
  'user:456:cart:total',
  { total: 599.98, items: 2 },
  {
    ttl: 300,
    dependsOn: ['product:123:price', 'product:456:price'],
  }
);

// เมื่อ product price เปลี่ยน → invalidate product cache และ cart cache อัตโนมัติ
await depCache.invalidate('product:123:price');
```

---

## Partial vs Full Invalidation

```typescript
class SmartCacheInvalidator {
  private redis: Redis;
  private db: Pool;

  constructor(redis: Redis, db: Pool) {
    this.redis = redis;
    this.db = db;
  }

  // Full invalidation: ลบทั้งหมดแล้ว warm up ใหม่
  async fullInvalidation(entityType: string): Promise<void> {
    console.log(`Starting full invalidation for ${entityType}...`);
    
    // 1. ลบทุก cache entries ของ entityType นี้
    let cursor = '0';
    let deletedCount = 0;
    
    do {
      const [newCursor, keys] = await this.redis.scan(
        cursor,
        'MATCH', `${entityType}:*`,
        'COUNT', 100
      );
      cursor = newCursor;
      if (keys.length > 0) {
        await this.redis.del(...keys);
        deletedCount += keys.length;
      }
    } while (cursor !== '0');
    
    console.log(`Full invalidation: deleted ${deletedCount} cache entries`);
    
    // 2. Warm up cache ด้วยข้อมูลใหม่ (optional)
    await this.warmUpCache(entityType);
  }

  // Partial invalidation: ลบเฉพาะ entries ที่เปลี่ยน
  async partialInvalidation(
    entityType: string,
    changedIds: string[]
  ): Promise<void> {
    console.log(`Partial invalidation for ${entityType}: ${changedIds.length} entities`);
    
    const keysToDelete: string[] = [];
    
    for (const id of changedIds) {
      keysToDelete.push(
        `${entityType}:${id}:detail`,
        `${entityType}:${id}:summary`,
        `${entityType}:${id}:stats`,
      );
    }
    
    if (keysToDelete.length > 0) {
      await this.redis.del(...keysToDelete);
    }
    
    // ลบ list caches ที่อาจ contain items ที่เปลี่ยน
    await this.invalidateListCaches(entityType);
    
    console.log(`Partial invalidation complete`);
  }

  // Smart invalidation: วิเคราะห์ว่า partial หรือ full ดีกว่า
  async smartInvalidation(
    entityType: string,
    changedIds: string[]
  ): Promise<void> {
    // นับจำนวน entries ที่มีใน cache
    let totalCached = 0;
    let cursor = '0';
    
    do {
      const [newCursor, keys] = await this.redis.scan(
        cursor,
        'MATCH', `${entityType}:*`,
        'COUNT', 100
      );
      cursor = newCursor;
      totalCached += keys.length;
    } while (cursor !== '0');

    // ถ้า changedIds มากกว่า 50% ของ total cached → full invalidation ดีกว่า
    const threshold = totalCached * 0.5;
    
    if (changedIds.length >= threshold) {
      console.log(`Changed ${changedIds.length}/${totalCached} entries → using full invalidation`);
      await this.fullInvalidation(entityType);
    } else {
      console.log(`Changed ${changedIds.length}/${totalCached} entries → using partial invalidation`);
      await this.partialInvalidation(entityType, changedIds);
    }
  }

  private async invalidateListCaches(entityType: string): Promise<void> {
    await this.deleteByPattern(`${entityType}:list:*`);
    await this.deleteByPattern(`${entityType}:search:*`);
  }

  private async deleteByPattern(pattern: string): Promise<void> {
    let cursor = '0';
    do {
      const [newCursor, keys] = await this.redis.scan(
        cursor, 'MATCH', pattern, 'COUNT', 100
      );
      cursor = newCursor;
      if (keys.length > 0) {
        await this.redis.del(...keys);
      }
    } while (cursor !== '0');
  }

  private async warmUpCache(entityType: string): Promise<void> {
    // Implement warm-up logic based on entity type
    console.log(`Warming up cache for ${entityType}...`);
    // ดึงข้อมูลยอดนิยม/สำคัญและเก็บใน cache ล่วงหน้า
  }
}
```

---

## Testing Cache Invalidation

```typescript
import { describe, it, expect, beforeEach, afterEach } from '@jest/globals';
import Redis from 'ioredis';
import { TagBasedCache } from './TagBasedCache';
import { ProductCatalogService } from './ProductCatalogService';

describe('Cache Invalidation Tests', () => {
  let redis: Redis;
  let cache: TagBasedCache;
  let mockDb: jest.Mocked<Pool>;

  beforeEach(async () => {
    redis = new Redis({ host: 'localhost', port: 6380 }); // test redis
    await redis.flushdb(); // clear ก่อน test
    cache = new TagBasedCache(redis);
    
    mockDb = {
      query: jest.fn(),
    } as any;
  });

  afterEach(async () => {
    await redis.flushdb();
    await redis.quit();
  });

  describe('TTL-based invalidation', () => {
    it('should return null after TTL expires', async () => {
      await cache.set('test:key', { data: 'value' }, { ttl: 1, tags: [] });
      
      // ยังมีอยู่
      let result = await cache.get('test:key');
      expect(result).toEqual({ data: 'value' });
      
      // รอ TTL หมด
      await new Promise(resolve => setTimeout(resolve, 1100));
      
      // หมดอายุแล้ว
      result = await cache.get('test:key');
      expect(result).toBeNull();
    });
  });

  describe('Tag-based invalidation', () => {
    it('should invalidate all entries with matching tag', async () => {
      // เก็บหลาย entries ด้วย tag เดียวกัน
      await cache.set('product:1:detail', { id: 1, name: 'A' }, {
        ttl: 3600,
        tags: ['category:electronics', 'product:1'],
      });
      await cache.set('product:2:detail', { id: 2, name: 'B' }, {
        ttl: 3600,
        tags: ['category:electronics', 'product:2'],
      });
      await cache.set('product:3:detail', { id: 3, name: 'C' }, {
        ttl: 3600,
        tags: ['category:clothing'],
      });

      // Invalidate category:electronics
      const invalidatedCount = await cache.invalidateByTag('category:electronics');
      
      expect(invalidatedCount).toBe(2);
      
      // Products 1 และ 2 ต้อง miss
      expect(await cache.get('product:1:detail')).toBeNull();
      expect(await cache.get('product:2:detail')).toBeNull();
      
      // Product 3 ยังอยู่
      expect(await cache.get('product:3:detail')).toEqual({ id: 3, name: 'C' });
    });

    it('should not affect entries without the tag', async () => {
      await cache.set('user:1:profile', { id: 1 }, {
        ttl: 3600,
        tags: ['user:1'],
      });

      await cache.invalidateByTag('category:electronics');
      
      // User cache ไม่ควรถูก invalidate
      expect(await cache.get('user:1:profile')).toEqual({ id: 1 });
    });
  });

  describe('Event-based invalidation', () => {
    it('should invalidate cache when entity is updated', async () => {
      const service = new ProductCatalogService(cache, mockDb);
      
      // Setup mock
      mockDb.query.mockResolvedValueOnce({
        rows: [{ id: '1', name: 'Product A', category_id: 'cat1' }],
      });

      // เรียก getProduct เพื่อ populate cache
      await service.getProductDetail('1');
      
      // ตรวจว่าอยู่ใน cache
      expect(await cache.get('product:1:detail')).not.toBeNull();
      
      // Mock การ update
      mockDb.query.mockResolvedValueOnce({
        rows: [{ category_id: 'cat1' }],
      });
      
      // อัพเดต product
      await service.updateProduct('1', { name: 'Updated Product A' });
      
      // Cache ต้องถูก invalidate
      expect(await cache.get('product:1:detail')).toBeNull();
    });
  });

  describe('Version-based invalidation', () => {
    it('should return old data after version bump when both keys exist', async () => {
      const versionCache = new VersionedCache(redis);
      
      // เก็บ version 1
      await versionCache.set('product:detail', 'product', '1', 
        { id: '1', name: 'Old Name' }, 3600
      );
      
      // ตรวจว่า get ได้
      let result = await versionCache.get('product:detail', 'product', '1');
      expect(result).toEqual({ id: '1', name: 'Old Name' });
      
      // Increment version (invalidation)
      await versionCache.incrementVersion('product', '1');
      
      // ตอนนี้ get จะ miss เพราะ version เปลี่ยน
      result = await versionCache.get('product:detail', 'product', '1');
      expect(result).toBeNull();
    });
  });

  describe('Dependency tracking', () => {
    it('should cascade invalidation to dependent entries', async () => {
      const depCache = new DependencyTrackingCache(redis);
      
      await depCache.set('product:1:price', { price: 100 }, { ttl: 3600 });
      await depCache.set('user:1:cart:total', { total: 200 }, {
        ttl: 300,
        dependsOn: ['product:1:price'],
      });

      // Invalidate product price
      await depCache.invalidate('product:1:price');
      
      // Cart total ต้องถูก invalidate ด้วย
      expect(await depCache.get('user:1:cart:total')).toBeNull();
    });
  });
});
```

---

## Full Example: Product Catalog Cache Invalidation System

```typescript
// types.ts
interface Product {
  id: string;
  name: string;
  description: string;
  price: number;
  stock: number;
  categoryId: string;
  brandId: string;
  imageUrl: string;
  active: boolean;
  createdAt: Date;
  updatedAt: Date;
}

interface Category {
  id: string;
  name: string;
  parentId?: string;
}

interface Brand {
  id: string;
  name: string;
}

// cache-manager.ts
class ProductCacheManager {
  private redis: Redis;
  private tagCache: TagBasedCache;
  private versionCache: VersionedCache;

  constructor(redis: Redis) {
    this.redis = redis;
    this.tagCache = new TagBasedCache(redis);
    this.versionCache = new VersionedCache(redis);
  }

  // === Getters ===
  
  async getProduct(productId: string): Promise<Product | null> {
    return this.tagCache.get<Product>(`product:${productId}`);
  }

  async getProductsByCategory(
    categoryId: string,
    page: number,
    limit: number
  ): Promise<{ products: Product[]; total: number } | null> {
    const key = `products:category:${categoryId}:page:${page}:limit:${limit}`;
    return this.tagCache.get(key);
  }

  async getSearchResults(query: string): Promise<Product[] | null> {
    const key = `search:${Buffer.from(query).toString('base64')}`;
    return this.tagCache.get(key);
  }

  // === Setters ===

  async setProduct(product: Product): Promise<void> {
    await this.tagCache.set(
      `product:${product.id}`,
      product,
      {
        ttl: 3600,
        tags: [
          `product:${product.id}`,
          `category:${product.categoryId}`,
          `brand:${product.brandId}`,
          'products:all',
        ],
      }
    );
  }

  async setProductsByCategory(
    categoryId: string,
    page: number,
    limit: number,
    data: { products: Product[]; total: number }
  ): Promise<void> {
    const key = `products:category:${categoryId}:page:${page}:limit:${limit}`;
    await this.tagCache.set(key, data, {
      ttl: 600,
      tags: [
        `category:${categoryId}`,
        'products:list',
      ],
    });
  }

  async setSearchResults(query: string, products: Product[]): Promise<void> {
    const key = `search:${Buffer.from(query).toString('base64')}`;
    await this.tagCache.set(key, products, {
      ttl: 300,  // 5 นาที
      tags: ['search:results'],
    });
  }

  // === Invalidators ===

  async invalidateProduct(productId: string): Promise<void> {
    console.log(`Invalidating cache for product: ${productId}`);
    await this.tagCache.invalidateByTag(`product:${productId}`);
  }

  async invalidateCategory(categoryId: string): Promise<void> {
    console.log(`Invalidating cache for category: ${categoryId}`);
    await this.tagCache.invalidateByTag(`category:${categoryId}`);
  }

  async invalidateBrand(brandId: string): Promise<void> {
    console.log(`Invalidating cache for brand: ${brandId}`);
    await this.tagCache.invalidateByTag(`brand:${brandId}`);
  }

  async invalidateAllSearchResults(): Promise<void> {
    console.log('Invalidating all search result caches');
    await this.tagCache.invalidateByTag('search:results');
  }

  async invalidateAll(): Promise<void> {
    console.log('Invalidating ALL product caches');
    await this.tagCache.invalidateByTag('products:all');
    await this.tagCache.invalidateByTag('products:list');
    await this.tagCache.invalidateByTag('search:results');
  }
}

// product-service.ts
class ProductService {
  private db: Pool;
  private cacheManager: ProductCacheManager;
  private invalidationPublisher: CacheInvalidationPublisher;

  constructor(
    db: Pool,
    cacheManager: ProductCacheManager,
    invalidationPublisher: CacheInvalidationPublisher
  ) {
    this.db = db;
    this.cacheManager = cacheManager;
    this.invalidationPublisher = invalidationPublisher;
  }

  async getProduct(productId: string): Promise<Product | null> {
    // 1. ลอง cache
    const cached = await this.cacheManager.getProduct(productId);
    if (cached) return cached;

    // 2. ดึงจาก DB
    const result = await this.db.query(
      `SELECT p.*, c.name as category_name, b.name as brand_name
       FROM products p
       JOIN categories c ON p.category_id = c.id  
       JOIN brands b ON p.brand_id = b.id
       WHERE p.id = $1 AND p.active = true`,
      [productId]
    );
    
    if (result.rows.length === 0) return null;
    
    const product = result.rows[0];
    
    // 3. เก็บใน cache
    await this.cacheManager.setProduct(product);
    
    return product;
  }

  async createProduct(data: Omit<Product, 'id' | 'createdAt' | 'updatedAt'>): Promise<Product> {
    const result = await this.db.query(
      `INSERT INTO products (name, description, price, stock, category_id, brand_id, active)
       VALUES ($1, $2, $3, $4, $5, $6, $7)
       RETURNING *`,
      [data.name, data.description, data.price, data.stock, 
       data.categoryId, data.brandId, data.active]
    );
    
    const product = result.rows[0];
    
    // Invalidate category and list caches
    await this.cacheManager.invalidateCategory(product.category_id);
    await this.cacheManager.invalidateAllSearchResults();
    
    // Broadcast ไปยัง distributed caches
    await this.invalidationPublisher.publishInvalidation({
      type: 'created',
      entityId: product.id,
      entityType: 'product',
    });
    
    return product;
  }

  async updateProduct(
    productId: string,
    data: Partial<Product>
  ): Promise<Product | null> {
    // ดึงข้อมูลเก่าก่อน
    const oldProduct = await this.getProduct(productId);
    if (!oldProduct) return null;

    const result = await this.db.query(
      `UPDATE products
       SET name=$1, description=$2, price=$3, stock=$4,
           category_id=$5, brand_id=$6, updated_at=NOW()
       WHERE id=$7
       RETURNING *`,
      [
        data.name ?? oldProduct.name,
        data.description ?? oldProduct.description,
        data.price ?? oldProduct.price,
        data.stock ?? oldProduct.stock,
        data.categoryId ?? oldProduct.categoryId,
        data.brandId ?? oldProduct.brandId,
        productId,
      ]
    );
    
    const updatedProduct = result.rows[0];
    
    // Invalidate product-specific caches
    await this.cacheManager.invalidateProduct(productId);
    
    // ถ้า category เปลี่ยน ต้อง invalidate ทั้ง 2 categories
    if (data.categoryId && data.categoryId !== oldProduct.categoryId) {
      await this.cacheManager.invalidateCategory(oldProduct.categoryId);
      await this.cacheManager.invalidateCategory(data.categoryId);
    } else {
      await this.cacheManager.invalidateCategory(oldProduct.categoryId);
    }
    
    // ถ้า brand เปลี่ยน ต้อง invalidate ทั้ง 2 brands
    if (data.brandId && data.brandId !== oldProduct.brandId) {
      await this.cacheManager.invalidateBrand(oldProduct.brandId);
      await this.cacheManager.invalidateBrand(data.brandId);
    }
    
    // Search results อาจ outdated
    await this.cacheManager.invalidateAllSearchResults();
    
    // Broadcast
    await this.invalidationPublisher.publishInvalidation({
      type: 'updated',
      entityId: productId,
      entityType: 'product',
    });
    
    return updatedProduct;
  }

  async deleteProduct(productId: string): Promise<void> {
    const product = await this.getProduct(productId);
    if (!product) return;

    await this.db.query('DELETE FROM products WHERE id = $1', [productId]);
    
    // Invalidate ทุกอย่างที่เกี่ยวกับ product นี้
    await this.cacheManager.invalidateProduct(productId);
    await this.cacheManager.invalidateCategory(product.categoryId);
    await this.cacheManager.invalidateBrand(product.brandId);
    await this.cacheManager.invalidateAllSearchResults();
    
    await this.invalidationPublisher.publishInvalidation({
      type: 'deleted',
      entityId: productId,
      entityType: 'product',
    });
  }

  async bulkUpdatePrices(
    updates: Array<{ productId: string; price: number }>
  ): Promise<void> {
    // Batch update
    const client = await this.db.connect();
    try {
      await client.query('BEGIN');
      
      for (const update of updates) {
        await client.query(
          'UPDATE products SET price=$1, updated_at=NOW() WHERE id=$2',
          [update.price, update.productId]
        );
      }
      
      await client.query('COMMIT');
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }

    // Invalidate ทีละ product
    await Promise.all(
      updates.map(({ productId }) =>
        this.cacheManager.invalidateProduct(productId)
      )
    );
    
    // ลบ search results เพราะ prices เปลี่ยนไปหลายตัว
    await this.cacheManager.invalidateAllSearchResults();
    
    console.log(`Bulk updated and invalidated ${updates.length} products`);
  }
}

// app.ts - การ wire up ทุก components
async function bootstrapApp() {
  const redis = new Redis(process.env.REDIS_URL!);
  const db = new Pool({ connectionString: process.env.DATABASE_URL });
  
  // Setup components
  const cacheManager = new ProductCacheManager(redis);
  const invalidationPublisher = new CacheInvalidationPublisher();
  const productService = new ProductService(db, cacheManager, invalidationPublisher);
  
  // Setup DB trigger listener
  const pgClient = new Client({ connectionString: process.env.DATABASE_URL });
  const dbInvalidator = new DatabaseCacheInvalidator(pgClient, redis);
  await dbInvalidator.startListening();
  
  // Setup Pub/Sub subscriber
  const pubSubInvalidator = new RedisPubSubInvalidator(process.env.REDIS_URL!);
  await pubSubInvalidator.setupSubscriber(redis);
  
  console.log('Cache invalidation system ready');
  
  return { productService, cacheManager };
}
```

---

## Cache Invalidation Checklist

เมื่อ implement cache invalidation ควรตรวจสอบ:

- [ ] กำหนด TTL ที่เหมาะสมกับข้อมูลแต่ละประเภท
- [ ] Invalidate cache ทุกที่ที่ข้อมูลเปลี่ยน (create, update, delete)
- [ ] ระวัง race condition ระหว่าง cache miss และ database write
- [ ] Test ว่า invalidation ทำงานถูกต้องใน unit tests
- [ ] Monitor cache hit rate หลัง deploy
- [ ] มี fallback กรณี Redis down (graceful degradation)
- [ ] Log cache invalidation events สำหรับ debugging
- [ ] ระวัง thundering herd เมื่อ cache cold start
- [ ] ทำ distributed invalidation ถ้ามีหลาย cache nodes

---

## สรุป

| Strategy | ความซับซ้อน | ความถูกต้อง | Performance |
|----------|------------|------------|-------------|
| TTL-based | ต่ำ | อาจ stale | ดี |
| Event-based (App) | ปานกลาง | ดีมาก | ดี |
| Event-based (DB Trigger) | สูง | ดีมาก | ดี |
| Version-based | ปานกลาง | ดีมาก | ดี |
| Tag-based | สูง | ดีมาก | ดี |
| Dependency Tracking | สูงมาก | ดีที่สุด | ปานกลาง |

ในทางปฏิบัติ ควรใช้ **Tag-based + TTL** เป็น baseline และเพิ่ม **Event-based invalidation** สำหรับข้อมูลที่ต้องการ consistency สูง

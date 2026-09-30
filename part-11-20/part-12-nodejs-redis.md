# Part 12: เชื่อมต่อ Node.js กับ Redis (ioredis)

## บทนำ

ในบทนี้เราจะเรียนรู้การใช้ `ioredis` ซึ่งเป็น Redis client ที่ดีที่สุดสำหรับ Node.js รองรับ TypeScript, Redis Cluster, Redis Sentinel และมี features ครบครัน

---

## 1. การติดตั้ง

```bash
# ติดตั้ง ioredis
npm install ioredis

# TypeScript support (built-in ใน ioredis v5+)
# ไม่ต้องติดตั้ง @types/ioredis แยก

# ทางเลือก: node-redis (Redis official client)
npm install redis
npm install --save-dev @types/redis
```

### เปรียบเทียบ ioredis vs node-redis

| Feature | ioredis | node-redis |
|---------|---------|------------|
| TypeScript | Built-in | Built-in (v4+) |
| Cluster Support | ✅ | ✅ |
| Sentinel | ✅ | ✅ |
| Pipeline | ✅ | ✅ |
| Lua Scripts | ✅ | ✅ |
| Auto-reconnect | ✅ | ✅ |
| API Style | Promise/Callback | Promise only (v4) |
| Community | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |

**แนะนำ**: ioredis สำหรับโปรเจคส่วนใหญ่เพราะ API ยืดหยุ่นกว่า, node-redis สำหรับโปรเจคใหม่ที่ต้องการ official support

---

## 2. Connection Configuration

### 2.1 Basic Connection
```typescript
// src/config/redis.ts
import Redis, { RedisOptions } from 'ioredis';
import dotenv from 'dotenv';

dotenv.config();

export const redisConfig: RedisOptions = {
  host: process.env.REDIS_HOST || 'localhost',
  port: parseInt(process.env.REDIS_PORT || '6379', 10),
  password: process.env.REDIS_PASSWORD || undefined,
  db: parseInt(process.env.REDIS_DB || '0', 10),
  
  // TLS configuration (for production)
  tls: process.env.REDIS_TLS === 'true' ? {} : undefined,
  
  // Connection timeout (ms)
  connectTimeout: 10000,
  
  // Command timeout
  commandTimeout: 5000,
  
  // Retry strategy
  retryStrategy(times: number): number | null {
    const delay = Math.min(times * 50, 2000);
    if (times > 20) {
      console.error('[Redis] Max retries reached, giving up');
      return null; // Stop retrying
    }
    return delay;
  },
  
  // Max retries per request
  maxRetriesPerRequest: 3,
  
  // Reconnect on error
  reconnectOnError(err: Error): boolean | 1 | 2 {
    const targetErrors = ['READONLY', 'ECONNRESET', 'ECONNREFUSED'];
    if (targetErrors.some(e => err.message.includes(e))) {
      return 2; // Reconnect and resend command
    }
    return false;
  },
  
  // Enable offline queue (commands queued when disconnected)
  enableOfflineQueue: true,
  
  // Lazy connect (don't connect on creation)
  lazyConnect: false,
  
  // Key prefix
  keyPrefix: process.env.REDIS_KEY_PREFIX || '',
  
  // Auto-pipeline (auto-batch commands)
  enableAutoPipelining: process.env.REDIS_AUTO_PIPELINE === 'true',
};
```

### 2.2 Environment Variables
```bash
# .env
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=your_redis_password
REDIS_DB=0
REDIS_TLS=false
REDIS_KEY_PREFIX=myapp:

# Cluster
REDIS_CLUSTER=false
REDIS_CLUSTER_NODES=localhost:7000,localhost:7001,localhost:7002

# Sentinel
REDIS_SENTINEL=false
REDIS_SENTINEL_NAME=mymaster
REDIS_SENTINEL_NODES=localhost:26379,localhost:26380

# Connection string (alternative)
REDIS_URL=redis://:password@localhost:6379/0
```

---

## 3. Redis Client Singleton

```typescript
// src/db/redis.ts
import Redis, { Cluster, RedisOptions, ClusterOptions } from 'ioredis';
import { redisConfig } from '../config/redis';

type RedisClient = Redis | Cluster;

class RedisManager {
  private client: RedisClient;
  private static instance: RedisManager;
  private isConnected: boolean = false;

  private constructor() {
    if (process.env.REDIS_CLUSTER === 'true') {
      this.client = this.createClusterClient();
    } else if (process.env.REDIS_SENTINEL === 'true') {
      this.client = this.createSentinelClient();
    } else {
      this.client = this.createStandaloneClient();
    }
    
    this.setupEventListeners();
  }

  private createStandaloneClient(): Redis {
    const redis = new Redis(redisConfig);
    return redis;
  }

  private createClusterClient(): Cluster {
    const nodes = (process.env.REDIS_CLUSTER_NODES || 'localhost:7000')
      .split(',')
      .map(node => {
        const [host, port] = node.split(':');
        return { host, port: parseInt(port, 10) };
      });

    const clusterOptions: ClusterOptions = {
      clusterRetryStrategy(times: number): number | null {
        if (times > 10) return null;
        return Math.min(times * 100, 3000);
      },
      enableOfflineQueue: true,
      enableReadyCheck: true,
      redisOptions: {
        password: process.env.REDIS_PASSWORD,
        tls: process.env.REDIS_TLS === 'true' ? {} : undefined,
        connectTimeout: 10000,
      },
      scaleReads: 'slave', // Read from replicas
    };

    return new Redis.Cluster(nodes, clusterOptions);
  }

  private createSentinelClient(): Redis {
    const sentinelNodes = (process.env.REDIS_SENTINEL_NODES || 'localhost:26379')
      .split(',')
      .map(node => {
        const [host, port] = node.split(':');
        return { host, port: parseInt(port, 10) };
      });

    return new Redis({
      sentinels: sentinelNodes,
      name: process.env.REDIS_SENTINEL_NAME || 'mymaster',
      password: process.env.REDIS_PASSWORD,
      sentinelPassword: process.env.REDIS_SENTINEL_PASSWORD,
      role: 'master',
    });
  }

  private setupEventListeners(): void {
    this.client.on('connect', () => {
      console.log('[Redis] Connecting...');
    });

    this.client.on('ready', () => {
      this.isConnected = true;
      console.log('[Redis] Ready to accept commands');
    });

    this.client.on('error', (err: Error) => {
      console.error('[Redis] Error:', err.message);
    });

    this.client.on('close', () => {
      this.isConnected = false;
      console.log('[Redis] Connection closed');
    });

    this.client.on('reconnecting', () => {
      console.log('[Redis] Reconnecting...');
    });

    this.client.on('end', () => {
      this.isConnected = false;
      console.log('[Redis] Connection ended');
    });
  }

  static getInstance(): RedisManager {
    if (!RedisManager.instance) {
      RedisManager.instance = new RedisManager();
    }
    return RedisManager.instance;
  }

  getClient(): RedisClient {
    return this.client;
  }

  isReady(): boolean {
    return this.isConnected;
  }

  async disconnect(): Promise<void> {
    await this.client.quit();
    console.log('[Redis] Disconnected gracefully');
  }

  async healthCheck(): Promise<boolean> {
    try {
      const result = await (this.client as Redis).ping();
      return result === 'PONG';
    } catch {
      return false;
    }
  }
}

export const redisManager = RedisManager.getInstance();
export const redis = redisManager.getClient() as Redis;
export default redis;
```

---

## 4. Basic Operations

```typescript
// src/db/redis-ops.ts
import { redis } from './redis';

// ============================================================
// STRING operations
// ============================================================

// SET with TTL
async function setWithTTL(key: string, value: string, ttlSeconds: number): Promise<void> {
  await redis.setex(key, ttlSeconds, value);
}

// SET with options
async function setAdvanced(
  key: string,
  value: string,
  options?: {
    ttl?: number;
    nx?: boolean;  // Set only if NOT exists
    xx?: boolean;  // Set only if EXISTS
    get?: boolean; // Return old value
  }
): Promise<string | null> {
  const args: (string | number)[] = [key, value];
  
  if (options?.ttl) {
    args.push('EX', options.ttl);
  }
  
  if (options?.nx) args.push('NX');
  if (options?.xx) args.push('XX');
  if (options?.get) args.push('GET');
  
  const result = await (redis as any).set(...args);
  return result;
}

// GET
async function get(key: string): Promise<string | null> {
  return redis.get(key);
}

// GET and DELETE
async function getAndDelete(key: string): Promise<string | null> {
  const value = await redis.get(key);
  if (value !== null) {
    await redis.del(key);
  }
  return value;
}

// INCR / DECR
async function increment(key: string, by: number = 1): Promise<number> {
  if (by === 1) return redis.incr(key);
  return redis.incrby(key, by);
}

async function decrement(key: string, by: number = 1): Promise<number> {
  if (by === 1) return redis.decr(key);
  return redis.decrby(key, by);
}

// ============================================================
// HASH operations
// ============================================================

async function hSet(key: string, field: string, value: string): Promise<void> {
  await redis.hset(key, field, value);
}

async function hSetMultiple(key: string, data: Record<string, string | number>): Promise<void> {
  await redis.hset(key, data);
}

async function hGet(key: string, field: string): Promise<string | null> {
  return redis.hget(key, field);
}

async function hGetAll(key: string): Promise<Record<string, string>> {
  return redis.hgetall(key);
}

async function hDel(key: string, ...fields: string[]): Promise<void> {
  await redis.hdel(key, ...fields);
}

// ============================================================
// LIST operations
// ============================================================

async function listPush(key: string, ...values: string[]): Promise<void> {
  await redis.rpush(key, ...values);
}

async function listPop(key: string): Promise<string | null> {
  return redis.lpop(key);
}

async function listRange(key: string, start: number, stop: number): Promise<string[]> {
  return redis.lrange(key, start, stop);
}

async function listLength(key: string): Promise<number> {
  return redis.llen(key);
}

// ============================================================
// SET operations
// ============================================================

async function setAdd(key: string, ...members: string[]): Promise<void> {
  await redis.sadd(key, ...members);
}

async function setRemove(key: string, ...members: string[]): Promise<void> {
  await redis.srem(key, ...members);
}

async function setMembers(key: string): Promise<string[]> {
  return redis.smembers(key);
}

async function setIsMember(key: string, member: string): Promise<boolean> {
  const result = await redis.sismember(key, member);
  return result === 1;
}

// ============================================================
// SORTED SET operations
// ============================================================

async function zAdd(key: string, score: number, member: string): Promise<void> {
  await redis.zadd(key, score, member);
}

async function zRange(key: string, start: number, stop: number, withScores: boolean = false): Promise<string[]> {
  if (withScores) {
    return redis.zrange(key, start, stop, 'WITHSCORES');
  }
  return redis.zrange(key, start, stop);
}

async function zRank(key: string, member: string): Promise<number | null> {
  return redis.zrank(key, member);
}

// ============================================================
// KEY operations
// ============================================================

async function exists(key: string): Promise<boolean> {
  const result = await redis.exists(key);
  return result === 1;
}

async function expire(key: string, ttlSeconds: number): Promise<void> {
  await redis.expire(key, ttlSeconds);
}

async function ttl(key: string): Promise<number> {
  return redis.ttl(key);
}

async function del(...keys: string[]): Promise<void> {
  await redis.del(...keys);
}

async function keys(pattern: string): Promise<string[]> {
  return redis.keys(pattern);
}
```

---

## 5. Pipeline: Batch Multiple Commands

```typescript
// src/db/redis-pipeline.ts
import { redis } from './redis';

// Pipeline เพิ่ม performance โดยส่ง commands หลายอัน ในครั้งเดียว
async function pipelineExample(): Promise<void> {
  const pipeline = redis.pipeline();
  
  // Queue up commands
  pipeline.set('key1', 'value1', 'EX', 60);
  pipeline.set('key2', 'value2', 'EX', 60);
  pipeline.set('key3', 'value3', 'EX', 60);
  pipeline.get('key1');
  pipeline.incr('counter');
  
  // Execute all at once
  const results = await pipeline.exec();
  
  if (results) {
    results.forEach(([error, result], index) => {
      if (error) {
        console.error(`Command ${index} failed:`, error);
      } else {
        console.log(`Command ${index} result:`, result);
      }
    });
  }
}

// Practical pipeline: bulk cache warm-up
async function warmUpCache(
  users: Array<{ id: number; data: object }>,
  ttlSeconds: number
): Promise<void> {
  const pipeline = redis.pipeline();
  
  for (const user of users) {
    pipeline.setex(
      `user:${user.id}`,
      ttlSeconds,
      JSON.stringify(user.data)
    );
  }
  
  await pipeline.exec();
  console.log(`[Cache] Warmed up ${users.length} user cache entries`);
}

// Pipeline with error handling
async function safeMultiSet(
  entries: Array<{ key: string; value: string; ttl?: number }>
): Promise<{ success: number; failed: number }> {
  const pipeline = redis.pipeline();
  
  for (const entry of entries) {
    if (entry.ttl) {
      pipeline.setex(entry.key, entry.ttl, entry.value);
    } else {
      pipeline.set(entry.key, entry.value);
    }
  }
  
  const results = await pipeline.exec();
  
  let success = 0;
  let failed = 0;
  
  if (results) {
    for (const [error] of results) {
      if (error) {
        failed++;
      } else {
        success++;
      }
    }
  }
  
  return { success, failed };
}
```

---

## 6. Transaction: MULTI/EXEC

```typescript
// src/db/redis-transactions.ts
import { redis } from './redis';

// Basic MULTI/EXEC transaction
async function atomicIncrAndSet(
  counterKey: string,
  dataKey: string,
  data: string
): Promise<void> {
  // ใช้ multi() สำหรับ atomic operations
  const result = await redis.multi()
    .incr(counterKey)
    .set(dataKey, data)
    .exec();
  
  if (!result) {
    throw new Error('Transaction aborted');
  }
  
  console.log('[Redis] Transaction completed:', result);
}

// Transaction with WATCH (optimistic locking)
async function conditionalUpdate(
  key: string,
  expectedValue: string,
  newValue: string
): Promise<boolean> {
  let attempts = 0;
  const maxAttempts = 3;
  
  while (attempts < maxAttempts) {
    // WATCH the key
    await redis.watch(key);
    
    const currentValue = await redis.get(key);
    
    if (currentValue !== expectedValue) {
      await redis.unwatch();
      return false; // Value changed, condition not met
    }
    
    // Execute transaction
    const result = await redis.multi()
      .set(key, newValue)
      .exec();
    
    if (result !== null) {
      return true; // Success
    }
    
    // result is null means WATCH detected a change
    attempts++;
    console.log(`[Redis] Transaction retry ${attempts}/${maxAttempts}`);
  }
  
  throw new Error('Transaction failed after max retries');
}

// Shopping cart with transaction
async function addToCart(
  userId: number,
  productId: number,
  quantity: number
): Promise<void> {
  const cartKey = `cart:${userId}`;
  const totalKey = `cart:${userId}:total`;
  
  // Get product price (from some source)
  const productPrice = 99.99;
  
  const multi = redis.multi();
  
  // Add/update item quantity
  multi.hset(cartKey, `product:${productId}`, quantity.toString());
  
  // Update cart expiry (24 hours)
  multi.expire(cartKey, 86400);
  
  // Recalculate total (simplified)
  multi.hincrbyfloat(totalKey, 'total', productPrice * quantity);
  multi.expire(totalKey, 86400);
  
  const results = await multi.exec();
  
  if (!results) {
    throw new Error('Cart update failed');
  }
}
```

---

## 7. Pub/Sub Implementation

```typescript
// src/db/redis-pubsub.ts
import Redis from 'ioredis';
import { redisConfig } from '../config/redis';

// ต้องใช้ connections แยกสำหรับpub และ sub
const publisher = new Redis(redisConfig);
const subscriber = new Redis(redisConfig);

export type MessageHandler = (channel: string, message: string) => void | Promise<void>;

class PubSubManager {
  private handlers: Map<string, MessageHandler[]> = new Map();
  private patternHandlers: Map<string, MessageHandler[]> = new Map();

  constructor() {
    subscriber.on('message', (channel: string, message: string) => {
      const channelHandlers = this.handlers.get(channel) || [];
      channelHandlers.forEach(handler => {
        Promise.resolve(handler(channel, message)).catch(console.error);
      });
    });

    subscriber.on('pmessage', (pattern: string, channel: string, message: string) => {
      const patternHandlerList = this.patternHandlers.get(pattern) || [];
      patternHandlerList.forEach(handler => {
        Promise.resolve(handler(channel, message)).catch(console.error);
      });
    });
  }

  async subscribe(channel: string, handler: MessageHandler): Promise<void> {
    if (!this.handlers.has(channel)) {
      this.handlers.set(channel, []);
      await subscriber.subscribe(channel);
      console.log(`[PubSub] Subscribed to: ${channel}`);
    }
    this.handlers.get(channel)!.push(handler);
  }

  async unsubscribe(channel: string): Promise<void> {
    await subscriber.unsubscribe(channel);
    this.handlers.delete(channel);
    console.log(`[PubSub] Unsubscribed from: ${channel}`);
  }

  async pSubscribe(pattern: string, handler: MessageHandler): Promise<void> {
    if (!this.patternHandlers.has(pattern)) {
      this.patternHandlers.set(pattern, []);
      await subscriber.psubscribe(pattern);
      console.log(`[PubSub] Pattern subscribed: ${pattern}`);
    }
    this.patternHandlers.get(pattern)!.push(handler);
  }

  async publish(channel: string, message: string | object): Promise<number> {
    const messageStr = typeof message === 'string'
      ? message
      : JSON.stringify(message);
    
    const receiverCount = await publisher.publish(channel, messageStr);
    return receiverCount;
  }

  async disconnect(): Promise<void> {
    await subscriber.quit();
    await publisher.quit();
  }
}

export const pubSub = new PubSubManager();

// Usage example
async function setupNotifications(): Promise<void> {
  // Subscribe to user notifications
  await pubSub.subscribe('notifications:user', async (channel, message) => {
    const notification = JSON.parse(message);
    console.log(`[Notification] User ${notification.userId}: ${notification.message}`);
    // Send to WebSocket, email, etc.
  });

  // Subscribe with pattern
  await pubSub.pSubscribe('events:*', (channel, message) => {
    console.log(`[Event] ${channel}: ${message}`);
  });

  // Publish
  await pubSub.publish('notifications:user', {
    userId: 123,
    type: 'order_shipped',
    message: 'Your order has been shipped!',
    timestamp: new Date().toISOString(),
  });
}
```

---

## 8. Lua Scripting

```typescript
// src/db/redis-lua.ts
import { redis } from './redis';

// Rate limiter using Lua script (atomic operation)
const rateLimitScript = `
  local key = KEYS[1]
  local limit = tonumber(ARGV[1])
  local window = tonumber(ARGV[2])
  local now = tonumber(ARGV[3])
  
  -- Remove old entries outside window
  redis.call('ZREMRANGEBYSCORE', key, '-inf', now - window * 1000)
  
  -- Count current requests
  local count = redis.call('ZCARD', key)
  
  if count >= limit then
    return 0  -- Rate limited
  end
  
  -- Add current request
  redis.call('ZADD', key, now, now .. '-' .. math.random())
  redis.call('EXPIRE', key, window)
  
  return limit - count - 1  -- Remaining requests
`;

export async function checkRateLimit(
  identifier: string,
  limit: number,
  windowSeconds: number
): Promise<{ allowed: boolean; remaining: number }> {
  const key = `ratelimit:${identifier}`;
  const now = Date.now();
  
  const result = await redis.eval(
    rateLimitScript,
    1,          // number of keys
    key,        // KEYS[1]
    limit,      // ARGV[1]
    windowSeconds, // ARGV[2]
    now         // ARGV[3]
  ) as number;
  
  return {
    allowed: result >= 0,
    remaining: Math.max(0, result),
  };
}

// Atomic increment with max value
const incrWithMaxScript = `
  local key = KEYS[1]
  local max = tonumber(ARGV[1])
  local current = tonumber(redis.call('GET', key) or 0)
  
  if current >= max then
    return -1  -- Already at max
  end
  
  local new_val = redis.call('INCR', key)
  return new_val
`;

export async function incrWithMax(key: string, max: number): Promise<number> {
  return redis.eval(incrWithMaxScript, 1, key, max) as Promise<number>;
}

// Cached computed value (get or compute)
const getOrComputeScript = `
  local key = KEYS[1]
  local ttl = tonumber(ARGV[1])
  
  local value = redis.call('GET', key)
  if value ~= nil and value ~= false then
    return {1, value}  -- Cache hit
  end
  
  return {0, false}  -- Cache miss
`;

export async function getOrCompute<T>(
  key: string,
  ttlSeconds: number,
  compute: () => Promise<T>
): Promise<T> {
  const result = await redis.eval(
    getOrComputeScript, 1, key, ttlSeconds
  ) as [number, string | false];

  if (result[0] === 1 && result[1]) {
    return JSON.parse(result[1]) as T;
  }

  // Cache miss - compute value
  const value = await compute();
  await redis.setex(key, ttlSeconds, JSON.stringify(value));
  return value;
}
```

---

## 9. Helper Utilities: JSON Serialize/Deserialize

```typescript
// src/utils/redis-cache.ts
import { redis } from '../db/redis';

export class RedisCache {
  private prefix: string;

  constructor(prefix: string) {
    this.prefix = prefix;
  }

  private buildKey(key: string): string {
    return `${this.prefix}:${key}`;
  }

  async get<T>(key: string): Promise<T | null> {
    const data = await redis.get(this.buildKey(key));
    if (!data) return null;
    
    try {
      return JSON.parse(data) as T;
    } catch {
      console.error(`[Cache] Failed to parse cached value for key: ${key}`);
      return null;
    }
  }

  async set<T>(
    key: string,
    value: T,
    ttlSeconds?: number
  ): Promise<void> {
    const serialized = JSON.stringify(value);
    const fullKey = this.buildKey(key);
    
    if (ttlSeconds) {
      await redis.setex(fullKey, ttlSeconds, serialized);
    } else {
      await redis.set(fullKey, serialized);
    }
  }

  async delete(key: string): Promise<void> {
    await redis.del(this.buildKey(key));
  }

  async deletePattern(pattern: string): Promise<number> {
    const keys = await redis.keys(this.buildKey(pattern));
    if (keys.length === 0) return 0;
    await redis.del(...keys);
    return keys.length;
  }

  async exists(key: string): Promise<boolean> {
    const result = await redis.exists(this.buildKey(key));
    return result === 1;
  }

  async getOrSet<T>(
    key: string,
    factory: () => Promise<T>,
    ttlSeconds?: number
  ): Promise<T> {
    const cached = await this.get<T>(key);
    if (cached !== null) return cached;

    const value = await factory();
    await this.set(key, value, ttlSeconds);
    return value;
  }

  async invalidateMany(keys: string[]): Promise<void> {
    if (keys.length === 0) return;
    const fullKeys = keys.map(k => this.buildKey(k));
    await redis.del(...fullKeys);
  }

  async mget<T>(keys: string[]): Promise<(T | null)[]> {
    if (keys.length === 0) return [];
    const fullKeys = keys.map(k => this.buildKey(k));
    const values = await redis.mget(...fullKeys);
    
    return values.map(v => {
      if (!v) return null;
      try {
        return JSON.parse(v) as T;
      } catch {
        return null;
      }
    });
  }

  async mset<T>(
    entries: Array<{ key: string; value: T }>,
    ttlSeconds?: number
  ): Promise<void> {
    if (entries.length === 0) return;

    const pipeline = redis.pipeline();
    
    for (const entry of entries) {
      const fullKey = this.buildKey(entry.key);
      const serialized = JSON.stringify(entry.value);
      
      if (ttlSeconds) {
        pipeline.setex(fullKey, ttlSeconds, serialized);
      } else {
        pipeline.set(fullKey, serialized);
      }
    }
    
    await pipeline.exec();
  }
}

// Cache instances for different domains
export const userCache = new RedisCache('user');
export const sessionCache = new RedisCache('session');
export const productCache = new RedisCache('product');
```

---

## 10. Full Working Example: Cache Layer สำหรับ User API

```typescript
// src/services/user-with-cache.service.ts
import { userRepository } from '../repositories/user.repository';
import { userCache } from '../utils/redis-cache';
import { UserPublic } from '../types/user';

const USER_CACHE_TTL = 300; // 5 minutes
const USER_LIST_CACHE_TTL = 60; // 1 minute

export class UserCacheService {
  async getUser(id: number): Promise<UserPublic | null> {
    return userCache.getOrSet(
      `id:${id}`,
      async () => {
        const user = await userRepository.findById(id);
        if (!user) return null;
        return {
          id: user.id,
          username: user.username,
          email: user.email,
          role: user.role,
          created_at: user.created_at,
        };
      },
      USER_CACHE_TTL
    );
  }

  async getUserByEmail(email: string): Promise<UserPublic | null> {
    return userCache.getOrSet(
      `email:${email}`,
      async () => {
        const user = await userRepository.findByEmail(email);
        if (!user) return null;
        return {
          id: user.id,
          username: user.username,
          email: user.email,
          role: user.role,
          created_at: user.created_at,
        };
      },
      USER_CACHE_TTL
    );
  }

  async updateUser(id: number, data: any): Promise<UserPublic | null> {
    const user = await userRepository.update(id, data);
    if (!user) return null;

    const publicUser: UserPublic = {
      id: user.id,
      username: user.username,
      email: user.email,
      role: user.role,
      created_at: user.created_at,
    };

    // Update cache
    await userCache.set(`id:${id}`, publicUser, USER_CACHE_TTL);
    // Invalidate email cache (email might have changed)
    await userCache.deletePattern(`email:*`);

    return publicUser;
  }

  async deleteUser(id: number): Promise<boolean> {
    const user = await userRepository.findById(id);
    if (!user) return false;

    const deleted = await userRepository.softDelete(id);
    if (deleted) {
      // Invalidate all related caches
      await userCache.invalidateMany([
        `id:${id}`,
        `email:${user.email}`,
      ]);
    }

    return deleted;
  }

  async warmUpCache(userIds: number[]): Promise<void> {
    // Batch fetch and cache
    const users = await Promise.all(
      userIds.map(id => userRepository.findById(id))
    );

    const entries = users
      .filter((u): u is NonNullable<typeof u> => u !== null)
      .map(user => ({
        key: `id:${user.id}`,
        value: {
          id: user.id,
          username: user.username,
          email: user.email,
          role: user.role,
          created_at: user.created_at,
        } as UserPublic,
      }));

    await userCache.mset(entries, USER_CACHE_TTL);
    console.log(`[Cache] Warmed up ${entries.length} user caches`);
  }
}

export const userCacheService = new UserCacheService();
```

---

## 11. Rate Limiter Implementation

```typescript
// src/middlewares/rate-limiter.ts
import { Request, Response, NextFunction } from 'express';
import { redis } from '../db/redis';

export interface RateLimitOptions {
  windowMs: number;      // Window in milliseconds
  maxRequests: number;   // Max requests per window
  keyPrefix?: string;    // Key prefix
  keyGenerator?: (req: Request) => string;  // Custom key generator
  message?: string;      // Error message
  skipFailedRequests?: boolean;
  headers?: boolean;     // Send rate limit headers
}

export function createRateLimiter(options: RateLimitOptions) {
  const {
    windowMs,
    maxRequests,
    keyPrefix = 'rl',
    keyGenerator = (req) => req.ip || 'unknown',
    message = 'Too many requests, please try again later',
    headers = true,
  } = options;

  const windowSeconds = Math.ceil(windowMs / 1000);

  return async (req: Request, res: Response, next: NextFunction): Promise<void> => {
    try {
      const identifier = keyGenerator(req);
      const key = `${keyPrefix}:${identifier}`;
      const now = Date.now();
      const windowStart = now - windowMs;

      // Use pipeline for atomic operations
      const pipeline = redis.pipeline();
      pipeline.zremrangebyscore(key, '-inf', windowStart);
      pipeline.zadd(key, now, `${now}-${Math.random()}`);
      pipeline.zcard(key);
      pipeline.expire(key, windowSeconds);

      const results = await pipeline.exec();
      
      if (!results) {
        next();
        return;
      }

      const count = results[2]?.[1] as number || 0;
      const remaining = Math.max(0, maxRequests - count);
      const resetTime = new Date(now + windowMs);

      if (headers) {
        res.setHeader('X-RateLimit-Limit', maxRequests);
        res.setHeader('X-RateLimit-Remaining', remaining);
        res.setHeader('X-RateLimit-Reset', resetTime.toISOString());
      }

      if (count > maxRequests) {
        res.status(429).json({
          success: false,
          error: message,
          retryAfter: Math.ceil(windowMs / 1000),
        });
        return;
      }

      next();
    } catch (error) {
      console.error('[RateLimit] Error:', error);
      next(); // Don't block on rate limiter failure
    }
  };
}

// Predefined rate limiters
export const apiRateLimiter = createRateLimiter({
  windowMs: 15 * 60 * 1000, // 15 minutes
  maxRequests: 100,
  keyPrefix: 'api-rl',
});

export const loginRateLimiter = createRateLimiter({
  windowMs: 15 * 60 * 1000,
  maxRequests: 5,
  keyPrefix: 'login-rl',
  message: 'Too many login attempts, please try again in 15 minutes',
});

export const uploadRateLimiter = createRateLimiter({
  windowMs: 60 * 60 * 1000, // 1 hour
  maxRequests: 20,
  keyPrefix: 'upload-rl',
});
```

---

## 12. Session Store Implementation

```typescript
// src/db/redis-session.ts
import { redis } from './redis';
import { v4 as uuidv4 } from 'uuid';

export interface SessionData {
  userId: number;
  username: string;
  role: string;
  createdAt: string;
  lastAccessAt: string;
  userAgent?: string;
  ipAddress?: string;
  [key: string]: unknown;
}

export class RedisSessionStore {
  private readonly TTL: number;
  private readonly prefix: string;

  constructor(ttlSeconds: number = 3600, prefix: string = 'session') {
    this.TTL = ttlSeconds;
    this.prefix = prefix;
  }

  private buildKey(sessionId: string): string {
    return `${this.prefix}:${sessionId}`;
  }

  async create(data: Omit<SessionData, 'createdAt' | 'lastAccessAt'>): Promise<string> {
    const sessionId = uuidv4();
    const sessionData: SessionData = {
      ...data,
      createdAt: new Date().toISOString(),
      lastAccessAt: new Date().toISOString(),
    };

    await redis.setex(
      this.buildKey(sessionId),
      this.TTL,
      JSON.stringify(sessionData)
    );

    // Track user sessions
    await redis.sadd(`user:${data.userId}:sessions`, sessionId);
    await redis.expire(`user:${data.userId}:sessions`, this.TTL * 2);

    return sessionId;
  }

  async get(sessionId: string): Promise<SessionData | null> {
    const data = await redis.get(this.buildKey(sessionId));
    if (!data) return null;

    try {
      return JSON.parse(data) as SessionData;
    } catch {
      await this.destroy(sessionId);
      return null;
    }
  }

  async touch(sessionId: string): Promise<boolean> {
    const session = await this.get(sessionId);
    if (!session) return false;

    session.lastAccessAt = new Date().toISOString();
    
    await redis.setex(
      this.buildKey(sessionId),
      this.TTL,
      JSON.stringify(session)
    );
    
    return true;
  }

  async update(sessionId: string, updates: Partial<SessionData>): Promise<boolean> {
    const session = await this.get(sessionId);
    if (!session) return false;

    const updated = {
      ...session,
      ...updates,
      lastAccessAt: new Date().toISOString(),
    };

    await redis.setex(
      this.buildKey(sessionId),
      this.TTL,
      JSON.stringify(updated)
    );

    return true;
  }

  async destroy(sessionId: string): Promise<void> {
    const session = await this.get(sessionId);
    
    if (session) {
      await redis.srem(`user:${session.userId}:sessions`, sessionId);
    }
    
    await redis.del(this.buildKey(sessionId));
  }

  async destroyAllUserSessions(userId: number): Promise<number> {
    const sessionIds = await redis.smembers(`user:${userId}:sessions`);
    
    if (sessionIds.length === 0) return 0;
    
    const pipeline = redis.pipeline();
    for (const sessionId of sessionIds) {
      pipeline.del(this.buildKey(sessionId));
    }
    pipeline.del(`user:${userId}:sessions`);
    
    await pipeline.exec();
    return sessionIds.length;
  }

  async getUserSessions(userId: number): Promise<SessionData[]> {
    const sessionIds = await redis.smembers(`user:${userId}:sessions`);
    
    if (sessionIds.length === 0) return [];
    
    const sessions = await Promise.all(
      sessionIds.map(id => this.get(id))
    );
    
    return sessions.filter((s): s is SessionData => s !== null);
  }
}

export const sessionStore = new RedisSessionStore(
  parseInt(process.env.SESSION_TTL || '3600', 10)
);

// Session middleware for Express
import { Request, Response, NextFunction } from 'express';

export function sessionMiddleware(req: Request, res: Response, next: NextFunction): void {
  const sessionId = req.headers['x-session-id'] as string ||
    req.cookies?.['session_id'];

  if (!sessionId) {
    next();
    return;
  }

  sessionStore.get(sessionId)
    .then(session => {
      if (session) {
        (req as any).session = session;
        (req as any).sessionId = sessionId;
        // Refresh TTL
        sessionStore.touch(sessionId).catch(console.error);
      }
      next();
    })
    .catch(err => {
      console.error('[Session] Error loading session:', err);
      next();
    });
}
```

---

## 13. Graceful Shutdown

```typescript
// src/utils/redis-shutdown.ts
import { redisManager } from '../db/redis';
import { pubSub } from '../db/redis-pubsub';

export async function disconnectRedis(): Promise<void> {
  console.log('[Redis] Starting graceful disconnect...');
  
  try {
    await pubSub.disconnect();
    await redisManager.disconnect();
    console.log('[Redis] Disconnected successfully');
  } catch (error) {
    console.error('[Redis] Error during disconnect:', error);
  }
}

// Add to main shutdown handler
process.on('SIGTERM', async () => {
  await disconnectRedis();
});

process.on('SIGINT', async () => {
  await disconnectRedis();
});
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:
1. การติดตั้งและ configure ioredis
2. การ connect แบบ Standalone, Cluster, Sentinel
3. Basic operations ทั้งหมด: string, hash, list, set, sorted set
4. Pipeline สำหรับ batch commands
5. MULTI/EXEC transactions
6. Pub/Sub messaging
7. Lua scripting สำหรับ atomic operations
8. JSON cache helper utilities
9. Full cache layer implementation
10. Rate limiter ด้วย Redis
11. Session store implementation
12. Graceful shutdown

ในบทต่อไปเราจะเรียนรู้การเชื่อมต่อกับ MinIO/S3

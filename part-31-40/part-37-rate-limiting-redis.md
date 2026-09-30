# Part 37: Rate Limiting ด้วย Redis

## บทนำ

Rate Limiting คือกลไกที่จำกัดจำนวน Request ที่ Client สามารถส่งมาได้ในช่วงเวลาหนึ่ง มันเป็นส่วนสำคัญของการ Build API ที่ปลอดภัยและ Scalable

ในบทนี้เราจะเรียนรู้:
- ทำไมต้องใช้ Rate Limiting
- อัลกอริทึม Rate Limiting ต่างๆ
- การ Implement ด้วย Redis
- Express Middleware
- Distributed Rate Limiting

---

## 1. ทำไมต้องใช้ Rate Limiting

### 1.1 เหตุผลหลัก

**ป้องกัน Abuse:**
```
ถ้าไม่มี Rate Limiting:
- Bot สามารถ brute force password ได้
- Competitor สามารถ scrape ข้อมูลทั้งหมดได้
- Bad actor สามารถ DDoS ได้
- ผู้ใช้ที่มี Bug ใน code สามารถ spam request ได้
```

**ควบคุม Costs:**
```
ถ้า API call ค่าใช้จ่าย $0.001 ต่อ request:
- ไม่มี rate limit: Bot ส่ง 1M requests/hour = $1,000/hour
- มี rate limit 100 req/min: ค่าใช้จ่ายถูกจำกัด
```

**Fairness:**
```
ผู้ใช้คนหนึ่งไม่ควรใช้ทรัพยากรจนกระทบผู้ใช้คนอื่น
```

### 1.2 Use Cases

- **API Gateway**: จำกัดแต่ละ API Key
- **Login endpoint**: ป้องกัน brute force
- **Registration**: ป้องกัน spam accounts
- **File upload**: ควบคุม bandwidth
- **Search**: ป้องกัน expensive queries

---

## 2. Rate Limiting Algorithms

### 2.1 Fixed Window Counter

**แนวคิด:**
นับจำนวน requests ในช่วงเวลาคงที่ (window) เช่น ทุก 1 นาที

```
Window:     00:00 - 00:59    01:00 - 01:59
Requests:   [1,2,3...100]    [1,2,3...]
Limit:      100 per minute
```

**ข้อดี:**
- Simple มาก
- ใช้ Memory น้อย (เก็บแค่ counter และ timestamp)
- ง่ายต่อการ Implement

**ข้อเสีย (Burst Problem):**
```
สมมติ limit = 100 requests per minute

00:59 → ส่ง 100 requests (window 1 หมด)
01:00 → ส่ง 100 requests (window 2 เริ่ม)

ผล: ส่ง 200 requests ใน 1 วินาที!
```

**Redis Implementation:**
```typescript
async function fixedWindow(
  redis: Redis,
  key: string,
  limit: number,
  windowSeconds: number
): Promise<{ allowed: boolean; remaining: number; resetAt: number }> {
  const windowKey = `ratelimit:fw:${key}:${Math.floor(Date.now() / 1000 / windowSeconds)}`;
  
  const pipeline = redis.pipeline();
  pipeline.incr(windowKey);
  pipeline.expire(windowKey, windowSeconds);
  
  const results = await pipeline.exec();
  const count = results?.[0]?.[1] as number;
  
  const resetAt = Math.ceil(Date.now() / 1000 / windowSeconds) * windowSeconds;
  
  return {
    allowed: count <= limit,
    remaining: Math.max(0, limit - count),
    resetAt,
  };
}
```

### 2.2 Sliding Window Log

**แนวคิด:**
เก็บ timestamp ของทุก request ใน sorted set แล้วนับจาก timestamp ปัจจุบันย้อนหลังไป window size

```
Requests timestamps: [100, 200, 300, 400, 500, 600, 700, 800, 900, 1000]
Window: last 10 seconds
Current time: 1010

Count requests since 1010 - 10 = 1000
Count = 1 (only timestamp 1000)
```

**ข้อดี:**
- แม่นยำมาก ไม่มี burst problem
- ไม่มี edge cases

**ข้อเสีย:**
- ใช้ Memory มาก (เก็บทุก request)
- ถ้า limit 1000 req/min และมี users 10,000 คน = เก็บ 10 ล้าน records

**Redis Implementation:**
```typescript
async function slidingWindowLog(
  redis: Redis,
  key: string,
  limit: number,
  windowSeconds: number
): Promise<{ allowed: boolean; remaining: number; resetAt: number }> {
  const now = Date.now();
  const windowStart = now - (windowSeconds * 1000);
  const windowKey = `ratelimit:swl:${key}`;
  
  const pipeline = redis.pipeline();
  
  // ลบ entries เก่ากว่า window
  pipeline.zremrangebyscore(windowKey, 0, windowStart);
  
  // นับ entries ในช่วงเวลา
  pipeline.zcard(windowKey);
  
  // เพิ่ม request ปัจจุบัน
  pipeline.zadd(windowKey, now, `${now}-${Math.random()}`);
  
  // Set expiry
  pipeline.expire(windowKey, windowSeconds + 1);
  
  const results = await pipeline.exec();
  const count = results?.[1]?.[1] as number;
  
  const allowed = count < limit;
  
  // ถ้าเกิน limit ลบ entry ที่เพิ่งเพิ่ม
  if (!allowed) {
    await redis.zpopmax(windowKey);
  }
  
  return {
    allowed,
    remaining: Math.max(0, limit - count - (allowed ? 1 : 0)),
    resetAt: Math.floor((now + windowSeconds * 1000) / 1000),
  };
}
```

### 2.3 Sliding Window Counter (Hybrid)

**แนวคิด:**
ผสม Fixed Window กับ Sliding Window โดยประมาณค่าจาก 2 windows

```
ตัวอย่าง:
Window size: 1 minute
Current time: 00:45 (45 วินาทีใน window ปัจจุบัน)

Previous window (00:00-00:59): 80 requests
Current window (01:00-01:45): 30 requests

ประมาณการ:
overlap = 45/60 = 75%
estimated = (80 * (1 - 0.75)) + 30 = 20 + 30 = 50 requests
```

**ข้อดี:**
- ใช้ Memory น้อย (เก็บแค่ 2 counters)
- แม่นยำกว่า Fixed Window
- เร็วกว่า Sliding Window Log

**Redis Implementation:**
```typescript
async function slidingWindowCounter(
  redis: Redis,
  key: string,
  limit: number,
  windowSeconds: number
): Promise<{ allowed: boolean; remaining: number; resetAt: number }> {
  const now = Date.now() / 1000;
  const currentWindow = Math.floor(now / windowSeconds);
  const previousWindow = currentWindow - 1;
  
  const currentKey = `ratelimit:swc:${key}:${currentWindow}`;
  const previousKey = `ratelimit:swc:${key}:${previousWindow}`;
  
  const pipeline = redis.pipeline();
  pipeline.get(previousKey);
  pipeline.get(currentKey);
  
  const results = await pipeline.exec();
  const prevCount = parseInt((results?.[0]?.[1] as string) || '0');
  const currCount = parseInt((results?.[1]?.[1] as string) || '0');
  
  // คำนวณว่าอยู่กี่ % ของ window ปัจจุบัน
  const windowPosition = (now % windowSeconds) / windowSeconds;
  
  // Weighted count
  const estimatedCount = Math.floor(prevCount * (1 - windowPosition)) + currCount;
  
  if (estimatedCount >= limit) {
    const resetAt = (currentWindow + 1) * windowSeconds;
    return {
      allowed: false,
      remaining: 0,
      resetAt,
    };
  }
  
  // เพิ่ม counter
  const pipeline2 = redis.pipeline();
  pipeline2.incr(currentKey);
  pipeline2.expire(currentKey, windowSeconds * 2);
  await pipeline2.exec();
  
  return {
    allowed: true,
    remaining: limit - estimatedCount - 1,
    resetAt: (currentWindow + 1) * windowSeconds,
  };
}
```

### 2.4 Token Bucket

**แนวคิด:**
มี "ถัง" (bucket) ที่มี tokens Tokens จะถูกเติมในอัตราคงที่ แต่ละ request ใช้ 1 token ถ้า bucket ว่างก็ไม่สามารถส่ง request ได้

```
Bucket capacity: 10 tokens
Refill rate: 1 token per second

t=0:  tokens=10 → request → tokens=9
t=1:  tokens=10 (refilled) → 5 requests → tokens=5
t=2:  tokens=6 → 10 requests → tokens=0 (allow 6, block 4)
t=3:  tokens=1 (refilled 1) → request allowed
```

**ข้อดี:**
- Allow bursts ได้ (ใช้ tokens ที่สะสมไว้)
- Rate ที่ smooth กว่า
- เหมาะกับ API ที่ต้องการ flexibility

**Redis Lua Script Implementation:**
```typescript
const tokenBucketScript = `
local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local refillRate = tonumber(ARGV[2])  -- tokens per second
local now = tonumber(ARGV[3])
local requested = tonumber(ARGV[4])

-- ดึง state ปัจจุบัน
local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
local tokens = tonumber(bucket[1])
local lastRefill = tonumber(bucket[2])

-- Initialize ถ้า bucket ว่าง
if tokens == nil then
  tokens = capacity
  lastRefill = now
end

-- คำนวณ tokens ที่ควรเติม
local elapsed = now - lastRefill
local newTokens = elapsed * refillRate
tokens = math.min(capacity, tokens + newTokens)
lastRefill = now

-- ตรวจสอบว่ามี tokens พอ
if tokens >= requested then
  tokens = tokens - requested
  redis.call('HMSET', key, 'tokens', tokens, 'last_refill', lastRefill)
  redis.call('EXPIRE', key, math.ceil(capacity / refillRate) + 1)
  return {1, math.floor(tokens), 0}  -- allowed, remaining, retry_after
else
  -- คำนวณว่าต้องรอกี่วินาที
  local waitTime = math.ceil((requested - tokens) / refillRate)
  redis.call('HMSET', key, 'tokens', tokens, 'last_refill', lastRefill)
  redis.call('EXPIRE', key, math.ceil(capacity / refillRate) + 1)
  return {0, math.floor(tokens), waitTime}  -- denied, remaining, retry_after
end
`;

async function tokenBucket(
  redis: Redis,
  key: string,
  capacity: number,
  refillRate: number,   // tokens per second
  requested: number = 1
): Promise<{ allowed: boolean; remaining: number; retryAfter: number }> {
  const now = Date.now() / 1000;
  const bucketKey = `ratelimit:tb:${key}`;
  
  const result = await redis.eval(
    tokenBucketScript,
    1,
    bucketKey,
    capacity.toString(),
    refillRate.toString(),
    now.toString(),
    requested.toString()
  ) as [number, number, number];
  
  return {
    allowed: result[0] === 1,
    remaining: result[1],
    retryAfter: result[2],
  };
}
```

### 2.5 Leaky Bucket

**แนวคิด:**
Request เข้ามาในอัตราใดก็ได้ แต่ถูก process ในอัตราคงที่ (เหมือนน้ำรั่วออกจากถัง)

```
Incoming: burst of 100 requests
Bucket capacity: 50 requests
Output rate: 10 req/second

t=0: bucket=50 (full), reject 50
t=1: bucket=50→40, add 10 → bucket=40
t=2: bucket=40→30, add 10 → bucket=30
...
```

**ข้อดี:**
- Output rate คงที่สม่ำเสมอ
- เหมาะกับ backend services ที่ต้องการ smooth load

**ข้อเสีย:**
- Burst จะถูก reject หรือ queue ไว้
- ซับซ้อนกว่า Token Bucket

---

## 3. Full TypeScript RateLimiter Class

```typescript
// src/ratelimit/RateLimiter.ts

import Redis from 'ioredis';

export type RateLimitAlgorithm = 'fixed-window' | 'sliding-window-log' | 'sliding-window-counter' | 'token-bucket';

export interface RateLimitOptions {
  algorithm?: RateLimitAlgorithm;
  limit: number;
  window: number;           // seconds (for window-based algorithms)
  capacity?: number;        // Token bucket: max tokens
  refillRate?: number;      // Token bucket: tokens per second
  keyPrefix?: string;
  skipFailedRequests?: boolean;
  skipSuccessfulRequests?: boolean;
  trustProxy?: boolean;
  whitelist?: string[];     // IPs to skip rate limiting
}

export interface RateLimitResult {
  allowed: boolean;
  limit: number;
  remaining: number;
  resetAt: number;          // Unix timestamp
  retryAfter?: number;      // seconds to wait
}

// Lua scripts (atomic operations)
const FIXED_WINDOW_SCRIPT = `
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local now = tonumber(ARGV[3])

local windowKey = key .. ':' .. math.floor(now / window)
local count = redis.call('INCR', windowKey)

if count == 1 then
  redis.call('EXPIRE', windowKey, window + 1)
end

local resetAt = (math.floor(now / window) + 1) * window
local remaining = math.max(0, limit - count)

return {count <= limit and 1 or 0, remaining, resetAt, count}
`;

const SLIDING_WINDOW_COUNTER_SCRIPT = `
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local now = tonumber(ARGV[3])

local currentWindow = math.floor(now / window)
local prevWindow = currentWindow - 1
local windowPosition = (now % window) / window

local currentKey = key .. ':' .. currentWindow
local prevKey = key .. ':' .. prevWindow

local prevCount = tonumber(redis.call('GET', prevKey) or 0)
local currCount = tonumber(redis.call('GET', currentKey) or 0)

local estimated = math.floor(prevCount * (1 - windowPosition)) + currCount

if estimated >= limit then
  local resetAt = (currentWindow + 1) * window
  return {0, 0, resetAt, estimated}
end

local newCount = redis.call('INCR', currentKey)
redis.call('EXPIRE', currentKey, window * 2)

local resetAt = (currentWindow + 1) * window
local remaining = math.max(0, limit - estimated - 1)

return {1, remaining, resetAt, estimated + 1}
`;

const TOKEN_BUCKET_SCRIPT = `
local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local refillRate = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
local requested = tonumber(ARGV[4])

local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
local tokens = tonumber(bucket[1])
local lastRefill = tonumber(bucket[2])

if tokens == nil then
  tokens = capacity
  lastRefill = now
end

local elapsed = now - lastRefill
local newTokens = elapsed * refillRate
tokens = math.min(capacity, tokens + newTokens)
lastRefill = now

if tokens >= requested then
  tokens = tokens - requested
  redis.call('HMSET', key, 'tokens', tokens, 'last_refill', lastRefill)
  redis.call('EXPIRE', key, math.ceil(capacity / refillRate) + 60)
  return {1, math.floor(tokens), 0}
else
  local waitTime = math.ceil((requested - tokens) / refillRate)
  redis.call('HMSET', key, 'tokens', tokens, 'last_refill', lastRefill)
  redis.call('EXPIRE', key, math.ceil(capacity / refillRate) + 60)
  return {0, math.floor(tokens), waitTime}
end
`;

export class RateLimiter {
  private redis: Redis;
  private options: Required<RateLimitOptions>;

  constructor(redis: Redis, options: RateLimitOptions) {
    this.redis = redis;
    this.options = {
      algorithm: options.algorithm ?? 'sliding-window-counter',
      limit: options.limit,
      window: options.window,
      capacity: options.capacity ?? options.limit,
      refillRate: options.refillRate ?? options.limit / options.window,
      keyPrefix: options.keyPrefix ?? 'ratelimit',
      skipFailedRequests: options.skipFailedRequests ?? false,
      skipSuccessfulRequests: options.skipSuccessfulRequests ?? false,
      trustProxy: options.trustProxy ?? true,
      whitelist: options.whitelist ?? [],
    };
  }

  /**
   * ตรวจสอบ rate limit สำหรับ identifier
   */
  async check(identifier: string): Promise<RateLimitResult> {
    // Check whitelist
    if (this.options.whitelist.includes(identifier)) {
      return {
        allowed: true,
        limit: this.options.limit,
        remaining: this.options.limit,
        resetAt: Math.floor(Date.now() / 1000) + this.options.window,
      };
    }

    const key = `${this.options.keyPrefix}:${identifier}`;
    const now = Date.now() / 1000;

    switch (this.options.algorithm) {
      case 'fixed-window':
        return this.fixedWindow(key, now);
      
      case 'sliding-window-log':
        return this.slidingWindowLog(key, now);
      
      case 'sliding-window-counter':
        return this.slidingWindowCounter(key, now);
      
      case 'token-bucket':
        return this.tokenBucket(key, now);
      
      default:
        return this.slidingWindowCounter(key, now);
    }
  }

  private async fixedWindow(key: string, now: number): Promise<RateLimitResult> {
    const result = await this.redis.eval(
      FIXED_WINDOW_SCRIPT,
      1,
      key,
      this.options.limit.toString(),
      this.options.window.toString(),
      now.toString()
    ) as [number, number, number, number];

    const allowed = result[0] === 1;
    
    return {
      allowed,
      limit: this.options.limit,
      remaining: result[1],
      resetAt: result[2],
      retryAfter: allowed ? undefined : result[2] - Math.floor(now),
    };
  }

  private async slidingWindowLog(key: string, now: number): Promise<RateLimitResult> {
    const windowStart = now - this.options.window;
    const logKey = `${key}:log`;
    
    const pipeline = this.redis.pipeline();
    pipeline.zremrangebyscore(logKey, 0, windowStart * 1000);
    pipeline.zcard(logKey);
    pipeline.expire(logKey, this.options.window + 1);
    
    const results = await pipeline.exec();
    const count = results?.[1]?.[1] as number;
    
    const allowed = count < this.options.limit;
    
    if (allowed) {
      await this.redis.zadd(logKey, now * 1000, `${now}-${Math.random().toString(36).slice(2)}`);
    }
    
    const resetAt = Math.floor(now) + this.options.window;
    
    return {
      allowed,
      limit: this.options.limit,
      remaining: Math.max(0, this.options.limit - count - (allowed ? 1 : 0)),
      resetAt,
      retryAfter: allowed ? undefined : this.options.window,
    };
  }

  private async slidingWindowCounter(key: string, now: number): Promise<RateLimitResult> {
    const result = await this.redis.eval(
      SLIDING_WINDOW_COUNTER_SCRIPT,
      1,
      key,
      this.options.limit.toString(),
      this.options.window.toString(),
      now.toString()
    ) as [number, number, number, number];

    const allowed = result[0] === 1;
    
    return {
      allowed,
      limit: this.options.limit,
      remaining: result[1],
      resetAt: result[2],
      retryAfter: allowed ? undefined : result[2] - Math.floor(now),
    };
  }

  private async tokenBucket(key: string, now: number): Promise<RateLimitResult> {
    const result = await this.redis.eval(
      TOKEN_BUCKET_SCRIPT,
      1,
      key,
      this.options.capacity.toString(),
      this.options.refillRate.toString(),
      now.toString(),
      '1'
    ) as [number, number, number];

    const allowed = result[0] === 1;
    
    return {
      allowed,
      limit: this.options.capacity,
      remaining: result[1],
      resetAt: Math.floor(now + result[1] / this.options.refillRate),
      retryAfter: allowed ? undefined : result[2],
    };
  }

  /**
   * Reset rate limit สำหรับ identifier
   */
  async reset(identifier: string): Promise<void> {
    const key = `${this.options.keyPrefix}:${identifier}`;
    const pattern = `${key}:*`;
    
    const keys = await this.redis.keys(pattern);
    if (keys.length > 0) {
      await this.redis.del(...keys);
    }
    
    // ลบ token bucket key ด้วย
    await this.redis.del(key);
  }

  /**
   * Get current usage สำหรับ identifier
   */
  async getUsage(identifier: string): Promise<{ current: number; limit: number; resetAt: number }> {
    const result = await this.check(identifier);
    return {
      current: result.limit - result.remaining,
      limit: result.limit,
      resetAt: result.resetAt,
    };
  }
}
```

---

## 4. Express Middleware

```typescript
// src/ratelimit/rateLimitMiddleware.ts

import { Request, Response, NextFunction } from 'express';
import Redis from 'ioredis';
import { RateLimiter, RateLimitOptions, RateLimitResult } from './RateLimiter';

export interface RateLimitMiddlewareOptions extends RateLimitOptions {
  keyGenerator?: (req: Request) => string;
  onRateLimited?: (req: Request, res: Response, result: RateLimitResult) => void;
  skip?: (req: Request) => boolean;
  message?: string | ((req: Request, res: Response) => object);
  standardHeaders?: boolean;   // X-RateLimit-* headers
  legacyHeaders?: boolean;     // RateLimit-* headers (new standard)
}

export function createRateLimitMiddleware(
  redis: Redis,
  options: RateLimitMiddlewareOptions
) {
  const limiter = new RateLimiter(redis, options);

  const defaultKeyGenerator = (req: Request): string => {
    // Get IP address
    let ip = req.ip;
    
    if (options.trustProxy !== false) {
      const forwardedFor = req.headers['x-forwarded-for'];
      if (forwardedFor) {
        ip = (forwardedFor as string).split(',')[0].trim();
      }
    }
    
    return ip || 'unknown';
  };

  const keyGenerator = options.keyGenerator ?? defaultKeyGenerator;

  return async (req: Request, res: Response, next: NextFunction): Promise<void> => {
    // Skip ถ้ากำหนดไว้
    if (options.skip?.(req)) {
      return next();
    }

    const identifier = keyGenerator(req);
    
    try {
      const result = await limiter.check(identifier);

      // Set rate limit headers
      if (options.standardHeaders !== false) {
        res.setHeader('X-RateLimit-Limit', result.limit);
        res.setHeader('X-RateLimit-Remaining', result.remaining);
        res.setHeader('X-RateLimit-Reset', result.resetAt);
        
        if (!result.allowed && result.retryAfter) {
          res.setHeader('Retry-After', result.retryAfter);
        }
      }

      if (options.legacyHeaders) {
        res.setHeader('RateLimit-Limit', result.limit);
        res.setHeader('RateLimit-Remaining', result.remaining);
        res.setHeader('RateLimit-Reset', result.resetAt);
      }

      if (!result.allowed) {
        // Custom handler
        if (options.onRateLimited) {
          options.onRateLimited(req, res, result);
          return;
        }

        // Default response
        const message = typeof options.message === 'function'
          ? options.message(req, res)
          : options.message ?? {
              error: 'Too Many Requests',
              message: 'Rate limit exceeded. Please slow down.',
              retryAfter: result.retryAfter,
            };

        res.status(429).json(message);
        return;
      }

      next();
    } catch (error) {
      // ถ้า Redis มีปัญหา ให้ผ่านไปก่อน (fail open)
      console.error('Rate limit check failed:', error);
      next();
    }
  };
}

/**
 * Per-IP Rate Limiter
 */
export function createIPRateLimiter(redis: Redis, limit: number, windowSeconds: number) {
  return createRateLimitMiddleware(redis, {
    limit,
    window: windowSeconds,
    keyPrefix: 'ratelimit:ip',
    keyGenerator: (req) => {
      const forwarded = req.headers['x-forwarded-for'];
      return (forwarded ? (forwarded as string).split(',')[0] : req.ip) || 'unknown';
    },
  });
}

/**
 * Per-User Rate Limiter
 */
export function createUserRateLimiter(redis: Redis, limit: number, windowSeconds: number) {
  return createRateLimitMiddleware(redis, {
    limit,
    window: windowSeconds,
    keyPrefix: 'ratelimit:user',
    keyGenerator: (req) => {
      // ดึง user ID จาก JWT หรือ session
      const userId = (req as any).user?.id;
      if (!userId) throw new Error('User not authenticated');
      return `user:${userId}`;
    },
  });
}

/**
 * Per-API-Key Rate Limiter
 */
export function createAPIKeyRateLimiter(redis: Redis, limit: number, windowSeconds: number) {
  return createRateLimitMiddleware(redis, {
    limit,
    window: windowSeconds,
    keyPrefix: 'ratelimit:apikey',
    keyGenerator: (req) => {
      const apiKey = req.headers['x-api-key'] as string;
      if (!apiKey) return req.ip || 'unknown';
      return `apikey:${apiKey}`;
    },
  });
}
```

---

## 5. Tiered Rate Limiting

```typescript
// src/ratelimit/TieredRateLimiter.ts

import { Request, Response, NextFunction } from 'express';
import Redis from 'ioredis';
import { RateLimiter } from './RateLimiter';

export type UserTier = 'free' | 'basic' | 'premium' | 'enterprise';

export interface TierConfig {
  limit: number;
  window: number;
  burstLimit?: number;   // allow short burst
}

export const DEFAULT_TIER_CONFIGS: Record<UserTier, TierConfig> = {
  free: {
    limit: 100,
    window: 3600,     // 100 req/hour
    burstLimit: 10,   // burst: 10 req/second
  },
  basic: {
    limit: 1000,
    window: 3600,     // 1000 req/hour
    burstLimit: 50,
  },
  premium: {
    limit: 10000,
    window: 3600,     // 10000 req/hour
    burstLimit: 200,
  },
  enterprise: {
    limit: 100000,
    window: 3600,     // 100000 req/hour
    burstLimit: 1000,
  },
};

export class TieredRateLimiter {
  private redis: Redis;
  private configs: Record<UserTier, TierConfig>;
  private limiters: Map<UserTier, RateLimiter> = new Map();
  private burstLimiters: Map<UserTier, RateLimiter> = new Map();

  constructor(redis: Redis, configs: Partial<Record<UserTier, TierConfig>> = {}) {
    this.redis = redis;
    this.configs = { ...DEFAULT_TIER_CONFIGS, ...configs };

    // สร้าง limiter สำหรับแต่ละ tier
    for (const [tier, config] of Object.entries(this.configs)) {
      this.limiters.set(tier as UserTier, new RateLimiter(redis, {
        limit: config.limit,
        window: config.window,
        algorithm: 'sliding-window-counter',
        keyPrefix: `ratelimit:tier:${tier}`,
      }));

      if (config.burstLimit) {
        this.burstLimiters.set(tier as UserTier, new RateLimiter(redis, {
          limit: config.burstLimit,
          window: 1,   // 1 second window for burst
          algorithm: 'token-bucket',
          capacity: config.burstLimit,
          refillRate: config.burstLimit / 2,   // refill at half rate
          keyPrefix: `ratelimit:burst:${tier}`,
        }));
      }
    }
  }

  middleware() {
    return async (req: Request, res: Response, next: NextFunction): Promise<void> => {
      const user = (req as any).user;
      const userId = user?.id || req.ip || 'unknown';
      const tier: UserTier = user?.tier ?? 'free';

      const limiter = this.limiters.get(tier);
      const burstLimiter = this.burstLimiters.get(tier);

      if (!limiter) {
        return next();
      }

      try {
        // ตรวจ burst limit ก่อน
        if (burstLimiter) {
          const burstResult = await burstLimiter.check(userId);
          if (!burstResult.allowed) {
            res.setHeader('X-RateLimit-Burst-Limit', burstResult.limit);
            res.setHeader('X-RateLimit-Burst-Remaining', burstResult.remaining);
            res.setHeader('Retry-After', burstResult.retryAfter ?? 1);
            
            res.status(429).json({
              error: 'Too Many Requests',
              message: 'Burst limit exceeded',
              tier,
              retryAfter: burstResult.retryAfter ?? 1,
            });
            return;
          }
        }

        // ตรวจ regular limit
        const result = await limiter.check(userId);
        
        const config = this.configs[tier];
        res.setHeader('X-RateLimit-Tier', tier);
        res.setHeader('X-RateLimit-Limit', result.limit);
        res.setHeader('X-RateLimit-Remaining', result.remaining);
        res.setHeader('X-RateLimit-Reset', result.resetAt);

        if (!result.allowed) {
          res.setHeader('Retry-After', result.retryAfter ?? config.window);
          
          res.status(429).json({
            error: 'Too Many Requests',
            message: `Rate limit exceeded for ${tier} tier`,
            tier,
            limit: result.limit,
            remaining: 0,
            resetAt: new Date(result.resetAt * 1000).toISOString(),
            retryAfter: result.retryAfter ?? config.window,
            upgradeMessage: tier !== 'enterprise' ? `Upgrade your plan for higher limits` : undefined,
          });
          return;
        }

        next();
      } catch (error) {
        console.error('Tiered rate limit error:', error);
        next();  // fail open
      }
    };
  }
}
```

---

## 6. Rate Limit Headers

```typescript
// Rate Limit Headers ตาม RFC 6585 และ IETF Draft

/*
ตาม IETF Draft (RateLimit header fields for HTTP):
  RateLimit-Limit: limit per window
  RateLimit-Remaining: requests remaining
  RateLimit-Reset: seconds until reset

ตาม Common Practice (X- prefix):
  X-RateLimit-Limit: limit per window
  X-RateLimit-Remaining: requests remaining
  X-RateLimit-Reset: Unix timestamp of reset
  
ตาม RFC 7231:
  Retry-After: seconds to wait (or HTTP date)
*/

export function setRateLimitHeaders(
  res: Response,
  result: RateLimitResult,
  options: { standard?: boolean; legacy?: boolean } = {}
): void {
  const { standard = true, legacy = false } = options;

  if (standard) {
    // X-RateLimit headers (most common)
    res.setHeader('X-RateLimit-Limit', result.limit);
    res.setHeader('X-RateLimit-Remaining', result.remaining);
    res.setHeader('X-RateLimit-Reset', result.resetAt);  // Unix timestamp
    res.setHeader('X-RateLimit-Reset-Date', new Date(result.resetAt * 1000).toUTCString());

    if (!result.allowed && result.retryAfter !== undefined) {
      res.setHeader('Retry-After', result.retryAfter);
    }
  }

  if (legacy) {
    // New IETF draft format
    res.setHeader('RateLimit-Limit', result.limit);
    res.setHeader('RateLimit-Remaining', result.remaining);
    res.setHeader('RateLimit-Reset', result.resetAt - Math.floor(Date.now() / 1000));  // seconds until reset
  }
}

// ตัวอย่าง Response Headers:
/*
HTTP/1.1 200 OK
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1705312800
X-RateLimit-Reset-Date: Mon, 15 Jan 2024 10:00:00 GMT
Content-Type: application/json

HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1705312800
Retry-After: 57
Content-Type: application/json

{
  "error": "Too Many Requests",
  "message": "Rate limit exceeded",
  "retryAfter": 57
}
*/
```

---

## 7. Per-Endpoint Rate Limiting

```typescript
// src/ratelimit/endpointLimits.ts

import Redis from 'ioredis';
import { Router } from 'express';
import { createRateLimitMiddleware } from './rateLimitMiddleware';

export function applyEndpointRateLimits(router: Router, redis: Redis): void {
  // Login endpoint: 5 attempts per 15 minutes per IP
  const loginLimiter = createRateLimitMiddleware(redis, {
    limit: 5,
    window: 900,  // 15 minutes
    algorithm: 'sliding-window-log',  // แม่นยำที่สุด
    keyPrefix: 'ratelimit:login',
    keyGenerator: (req) => {
      const ip = req.headers['x-forwarded-for'] as string || req.ip || 'unknown';
      const email = req.body?.email || '';
      // จำกัดทั้ง per-IP และ per-email
      return `${ip}:${email.toLowerCase()}`;
    },
    message: {
      error: 'Too Many Login Attempts',
      message: 'Too many failed login attempts. Please try again in 15 minutes.',
    },
  });

  // Registration: 3 per hour per IP
  const registrationLimiter = createRateLimitMiddleware(redis, {
    limit: 3,
    window: 3600,
    algorithm: 'fixed-window',
    keyPrefix: 'ratelimit:register',
    message: {
      error: 'Too Many Registrations',
      message: 'Too many registration attempts from this IP.',
    },
  });

  // Password reset: 3 per hour per email
  const passwordResetLimiter = createRateLimitMiddleware(redis, {
    limit: 3,
    window: 3600,
    algorithm: 'sliding-window-counter',
    keyPrefix: 'ratelimit:password-reset',
    keyGenerator: (req) => req.body?.email?.toLowerCase() || req.ip || 'unknown',
  });

  // Search endpoint: 30 per minute per user (expensive query)
  const searchLimiter = createRateLimitMiddleware(redis, {
    limit: 30,
    window: 60,
    algorithm: 'sliding-window-counter',
    keyPrefix: 'ratelimit:search',
    keyGenerator: (req) => {
      const userId = (req as any).user?.id;
      return userId ? `user:${userId}` : req.ip || 'unknown';
    },
  });

  // File upload: 10 per hour per user
  const uploadLimiter = createRateLimitMiddleware(redis, {
    limit: 10,
    window: 3600,
    algorithm: 'token-bucket',
    capacity: 10,
    refillRate: 10 / 3600,  // 10 tokens per hour
    keyPrefix: 'ratelimit:upload',
    keyGenerator: (req) => {
      const userId = (req as any).user?.id;
      return userId ? `user:${userId}` : req.ip || 'unknown';
    },
  });

  // Apply to routes
  router.post('/auth/login', loginLimiter);
  router.post('/auth/register', registrationLimiter);
  router.post('/auth/forgot-password', passwordResetLimiter);
  router.get('/search', searchLimiter);
  router.post('/files/upload', uploadLimiter);
}
```

---

## 8. Distributed Rate Limiting

```typescript
// src/ratelimit/DistributedRateLimiter.ts

/*
ในระบบที่มีหลาย servers การ Rate Limiting ต้องทำงานร่วมกัน
Redis ช่วยให้ทุก server share state เดียวกัน

Architecture:
┌─────────────────────────────────────────────┐
│                  Load Balancer               │
└─────────────┬─────────────┬─────────────────┘
              │             │
    ┌─────────▼──┐    ┌─────▼──────┐
    │  Server 1  │    │  Server 2  │
    │  RateLimit │    │  RateLimit │
    │  Middleware│    │  Middleware│
    └─────────┬──┘    └─────┬──────┘
              │             │
              └──────┬──────┘
                     │
             ┌───────▼──────┐
             │  Redis Cluster│
             │  (Shared State)│
             └───────────────┘

ทุก request ไม่ว่าจะไปเจอ server ไหน
จะถูก check กับ Redis Cluster เดียวกัน
*/

// ตัวอย่าง Redis Cluster configuration สำหรับ Rate Limiting
import { Cluster } from 'ioredis';

function createRedisCluster(): Cluster {
  return new Cluster([
    { host: 'redis-node-1', port: 6379 },
    { host: 'redis-node-2', port: 6379 },
    { host: 'redis-node-3', port: 6379 },
  ], {
    redisOptions: {
      password: process.env.REDIS_PASSWORD,
    },
    // Lua scripts ต้องการ hash tags เพื่อให้ keys อยู่ใน slot เดียวกัน
    // ใช้ {identifier} เป็น hash tag
    scaleReads: 'slave',  // อ่านจาก replica
  });
}

// เมื่อใช้ Redis Cluster ต้องใช้ hash tags
// ตัวอย่าง: key = "ratelimit:{user:123}:window:1705312800"
// {user:123} คือ hash tag ที่บังคับให้ key ทั้งหมดของ user นี้อยู่ใน slot เดียวกัน
```

---

## 9. DDoS Protection

```typescript
// src/ratelimit/DDoSProtection.ts

import Redis from 'ioredis';
import { Request, Response, NextFunction } from 'express';
import { RateLimiter } from './RateLimiter';

export interface DDoSConfig {
  // Global rate limit
  globalLimit: number;        // requests per second across all IPs
  
  // Per-IP limits
  ipLimit: number;            // requests per minute per IP
  ipBanThreshold: number;     // requests per second to trigger IP ban
  ipBanDuration: number;      // seconds to ban IP
  
  // Suspicious behavior
  errorRateThreshold: number; // % of 4xx/5xx to flag
  
  // Trusted IPs (bypass all limits)
  trustedIPs: string[];
  
  // Custom blocklist
  blockedIPs?: string[];
}

export class DDoSProtection {
  private redis: Redis;
  private config: DDoSConfig;
  private globalLimiter: RateLimiter;
  private ipLimiter: RateLimiter;
  private burstDetector: RateLimiter;

  constructor(redis: Redis, config: DDoSConfig) {
    this.redis = redis;
    this.config = config;

    this.globalLimiter = new RateLimiter(redis, {
      limit: config.globalLimit,
      window: 1,  // per second
      algorithm: 'token-bucket',
      keyPrefix: 'ddos:global',
    });

    this.ipLimiter = new RateLimiter(redis, {
      limit: config.ipLimit,
      window: 60,
      algorithm: 'sliding-window-counter',
      keyPrefix: 'ddos:ip',
    });

    this.burstDetector = new RateLimiter(redis, {
      limit: config.ipBanThreshold,
      window: 1,  // per second
      algorithm: 'fixed-window',
      keyPrefix: 'ddos:burst',
    });
  }

  middleware() {
    return async (req: Request, res: Response, next: NextFunction): Promise<void> => {
      const ip = this.getClientIP(req);

      // Check blocked list
      if (this.config.blockedIPs?.includes(ip)) {
        res.status(403).json({ error: 'Forbidden' });
        return;
      }

      // Trusted IPs bypass
      if (this.config.trustedIPs.includes(ip)) {
        return next();
      }

      // Check if IP is banned
      const isBanned = await this.redis.get(`ddos:ban:${ip}`);
      if (isBanned) {
        const ttl = await this.redis.ttl(`ddos:ban:${ip}`);
        res.setHeader('Retry-After', ttl);
        res.status(429).json({
          error: 'IP Temporarily Banned',
          message: 'Your IP has been temporarily banned due to suspicious activity',
          retryAfter: ttl,
        });
        return;
      }

      // Check burst (triggers ban)
      const burstResult = await this.burstDetector.check(ip);
      if (!burstResult.allowed) {
        // Ban the IP
        await this.redis.setex(
          `ddos:ban:${ip}`,
          this.config.ipBanDuration,
          new Date().toISOString()
        );
        
        console.warn(`[DDoS] Banned IP: ${ip} for ${this.config.ipBanDuration}s`);
        
        res.status(429).json({
          error: 'IP Banned',
          message: 'Suspicious activity detected. IP temporarily banned.',
          retryAfter: this.config.ipBanDuration,
        });
        return;
      }

      // Regular IP rate limit
      const ipResult = await this.ipLimiter.check(ip);
      if (!ipResult.allowed) {
        res.setHeader('X-RateLimit-Limit', ipResult.limit);
        res.setHeader('X-RateLimit-Remaining', 0);
        res.setHeader('Retry-After', ipResult.retryAfter ?? 60);
        
        res.status(429).json({
          error: 'Too Many Requests',
          message: 'Rate limit exceeded',
          retryAfter: ipResult.retryAfter ?? 60,
        });
        return;
      }

      next();
    };
  }

  private getClientIP(req: Request): string {
    const forwarded = req.headers['x-forwarded-for'];
    if (forwarded) {
      return (forwarded as string).split(',')[0].trim();
    }
    return req.ip || req.socket.remoteAddress || 'unknown';
  }

  /**
   * ดึงรายการ IPs ที่ถูก ban อยู่
   */
  async getBannedIPs(): Promise<{ ip: string; expiresIn: number }[]> {
    const keys = await this.redis.keys('ddos:ban:*');
    const result: { ip: string; expiresIn: number }[] = [];
    
    for (const key of keys) {
      const ip = key.replace('ddos:ban:', '');
      const ttl = await this.redis.ttl(key);
      result.push({ ip, expiresIn: ttl });
    }
    
    return result;
  }

  /**
   * Unban IP manually
   */
  async unbanIP(ip: string): Promise<void> {
    await this.redis.del(`ddos:ban:${ip}`);
    console.log(`[DDoS] Unbanned IP: ${ip}`);
  }
}
```

---

## 10. Complete Application Example

```typescript
// src/app.ts - Complete rate limiting setup

import express from 'express';
import Redis from 'ioredis';
import { createRateLimitMiddleware, createIPRateLimiter } from './ratelimit/rateLimitMiddleware';
import { TieredRateLimiter } from './ratelimit/TieredRateLimiter';
import { DDoSProtection } from './ratelimit/DDoSProtection';
import { applyEndpointRateLimits } from './ratelimit/endpointLimits';

export function createApp(redis: Redis): express.Application {
  const app = express();
  app.use(express.json());

  // ========== DDoS Protection (first line of defense) ==========
  const ddosProtection = new DDoSProtection(redis, {
    globalLimit: 10000,          // 10k req/sec globally
    ipLimit: 500,                 // 500 req/min per IP
    ipBanThreshold: 100,          // 100 req/sec = suspicious
    ipBanDuration: 3600,          // ban for 1 hour
    errorRateThreshold: 0.5,      // 50% error rate = suspicious
    trustedIPs: [
      '127.0.0.1',
      '10.0.0.0/8',              // Internal network
      process.env.MONITORING_IP || '',
    ].filter(Boolean),
  });
  app.use(ddosProtection.middleware());

  // ========== Global API Rate Limit ==========
  const globalLimiter = createIPRateLimiter(redis, 1000, 3600); // 1000 req/hour per IP
  app.use('/api/', globalLimiter);

  // ========== Tiered Rate Limiting (per user tier) ==========
  const tieredLimiter = new TieredRateLimiter(redis, {
    free: { limit: 100, window: 3600, burstLimit: 10 },
    basic: { limit: 1000, window: 3600, burstLimit: 50 },
    premium: { limit: 10000, window: 3600, burstLimit: 200 },
    enterprise: { limit: 100000, window: 3600, burstLimit: 1000 },
  });
  app.use('/api/', tieredLimiter.middleware());

  // ========== Endpoint-specific Limits ==========
  const apiRouter = express.Router();
  applyEndpointRateLimits(apiRouter, redis);
  app.use('/api/v1', apiRouter);

  // ========== Routes ==========
  app.post('/api/v1/auth/login', async (req, res) => {
    // Login logic
    res.json({ token: 'jwt-token-here' });
  });

  app.get('/api/v1/users', async (req, res) => {
    res.json({ users: [] });
  });

  // ========== Admin endpoints for rate limit management ==========
  app.get('/admin/rate-limits/banned', async (req, res) => {
    const banned = await ddosProtection.getBannedIPs();
    res.json({ banned });
  });

  app.delete('/admin/rate-limits/banned/:ip', async (req, res) => {
    await ddosProtection.unbanIP(req.params.ip);
    res.json({ message: 'IP unbanned successfully' });
  });

  // ========== Error Handler ==========
  app.use((err: Error, req: express.Request, res: express.Response, next: express.NextFunction) => {
    console.error(err);
    res.status(500).json({ error: 'Internal Server Error' });
  });

  return app;
}

// ========== Rate Limit Testing Script ==========
/*
ทดสอบ Rate Limiting:

# ทดสอบ 200 requests ติดต่อกัน
for i in {1..200}; do
  response=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/api/v1/users)
  echo "Request $i: $response"
done

# ดู rate limit headers
curl -v http://localhost:3000/api/v1/users 2>&1 | grep -i "x-ratelimit"

# ทดสอบ login brute force protection
for i in {1..10}; do
  curl -X POST http://localhost:3000/api/v1/auth/login \
    -H "Content-Type: application/json" \
    -d '{"email":"test@test.com","password":"wrong"}'
  echo "---"
done
*/
```

---

## 11. Monitoring Rate Limits

```typescript
// src/ratelimit/RateLimitMonitor.ts

import Redis from 'ioredis';

export interface RateLimitStats {
  identifier: string;
  requests: number;
  limit: number;
  remaining: number;
  percentage: number;
  resetAt: Date;
}

export class RateLimitMonitor {
  private redis: Redis;

  constructor(redis: Redis) {
    this.redis = redis;
  }

  /**
   * Get top users by request count
   */
  async getTopUsers(limit: number = 10): Promise<RateLimitStats[]> {
    const pattern = 'ratelimit:user:*';
    const keys = await this.redis.keys(pattern);
    
    const stats: RateLimitStats[] = [];
    
    for (const key of keys) {
      const count = await this.redis.get(key);
      if (count) {
        const identifier = key.replace('ratelimit:user:', '');
        stats.push({
          identifier,
          requests: parseInt(count),
          limit: 1000,  // placeholder
          remaining: Math.max(0, 1000 - parseInt(count)),
          percentage: (parseInt(count) / 1000) * 100,
          resetAt: new Date(),
        });
      }
    }
    
    return stats
      .sort((a, b) => b.requests - a.requests)
      .slice(0, limit);
  }

  /**
   * Get rate limit violations count
   */
  async getViolationsCount(windowMinutes: number = 60): Promise<number> {
    const key = `ratelimit:violations:${Math.floor(Date.now() / 1000 / 60 / windowMinutes)}`;
    const count = await this.redis.get(key);
    return parseInt(count || '0');
  }

  /**
   * Record violation
   */
  async recordViolation(identifier: string): Promise<void> {
    const windowKey = `ratelimit:violations:${Math.floor(Date.now() / 1000 / 60 / 60)}`;
    await this.redis.incr(windowKey);
    await this.redis.expire(windowKey, 3600);
    
    console.warn(`[RateLimit] Violation by: ${identifier}`);
  }
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **ทำไมต้องใช้ Rate Limiting**: ป้องกัน abuse, ควบคุม costs, fairness

2. **อัลกอริทึม 5 แบบ**:
   - **Fixed Window**: Simple แต่มี burst problem
   - **Sliding Window Log**: แม่นยำ แต่ memory intensive
   - **Sliding Window Counter**: Hybrid ที่ดีที่สุดสำหรับกรณีทั่วไป
   - **Token Bucket**: Allow bursts, refill at constant rate
   - **Leaky Bucket**: Output rate คงที่

3. **Redis Implementation**: 
   - ใช้ Lua scripts สำหรับ atomic operations
   - ป้องกัน race conditions

4. **Express Middleware**:
   - Per-IP, Per-User, Per-API-Key
   - Tiered limits (free/premium)
   - Endpoint-specific limits

5. **Rate Limit Headers**: X-RateLimit-Limit, X-RateLimit-Remaining, X-RateLimit-Reset, Retry-After

6. **DDoS Protection**: Burst detection, IP banning

Rate Limiting เป็น Layer สำคัญของ API Security และ Reliability ทำให้บริการของเราทนทานต่อการใช้งานที่ผิดปกติ

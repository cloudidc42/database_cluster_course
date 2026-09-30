# Part 20: Logging และ Monitoring เบื้องต้น

## สารบัญ
1. [Logging Levels](#1-logging-levels)
2. [Structured Logging: JSON Format](#2-structured-logging-json-format)
3. [Winston Logger Setup](#3-winston-logger-setup)
4. [Log Formats: Development vs Production](#4-log-formats-development-vs-production)
5. [Log Transports](#5-log-transports)
6. [Request Logging ด้วย Morgan](#6-request-logging-ด้วย-morgan)
7. [Correlation ID และ Request ID](#7-correlation-id-และ-request-id)
8. [Log Context: User, RequestId, Duration](#8-log-context-user-requestid-duration)
9. [Sensitive Data Masking](#9-sensitive-data-masking)
10. [Log Rotation](#10-log-rotation)
11. [Database Query Logging](#11-database-query-logging)
12. [Redis Command Logging](#12-redis-command-logging)
13. [Health Check Endpoint](#13-health-check-endpoint)
14. [Metrics Basics](#14-metrics-basics)
15. [Process Metrics](#15-process-metrics)
16. [Full Working Logging Setup](#16-full-working-logging-setup)

---

## 1. Logging Levels

### Log Levels (เรียงจากมากไปน้อยในแง่ความสำคัญ)

```
ERROR   [50] — ข้อผิดพลาดร้ายแรง ระบบทำงานได้ แต่มีส่วนที่ผิดพลาด
WARN    [40] — คำเตือน ควรตรวจสอบ แต่ยังทำงานต่อได้
INFO    [30] — ข้อมูลทั่วไปเกี่ยวกับการทำงานของระบบ
HTTP    [25] — HTTP request/response
VERBOSE [25] — ข้อมูลละเอียดมากขึ้น
DEBUG   [20] — ข้อมูลสำหรับ debugging
SILLY   [10] — ข้อมูลละเอียดที่สุด (trace)
```

### เมื่อใช้แต่ละ Level

```typescript
// ERROR: ข้อผิดพลาดที่ต้องดูแลทันที
logger.error('Database connection failed', { error: err.message, retries: 3 });
logger.error('Payment processing failed', { userId, orderId, error: err });

// WARN: สิ่งที่น่าเป็นห่วง แต่ระบบยังทำงานได้
logger.warn('High memory usage', { usedPercent: 85 });
logger.warn('Slow query detected', { query, duration: 5000 });
logger.warn('User login failed', { email, attempts: 3 });
logger.warn('Rate limit approaching', { ip, requests: 90, limit: 100 });

// INFO: event สำคัญที่ควรติดตาม
logger.info('Server started', { port: 3000, env: 'production' });
logger.info('User registered', { userId, email });
logger.info('Order created', { orderId, userId, total: 1500 });
logger.info('Payment processed', { orderId, amount: 1500, method: 'credit_card' });

// DEBUG: ข้อมูลสำหรับ development
logger.debug('Cache hit', { key, ttl });
logger.debug('Query executed', { sql, params, duration: 12 });
logger.debug('Token validated', { userId, exp });

// ไม่ควร log ใน production:
logger.debug('Processing request', { body: req.body }); // อาจมี sensitive data!
```

---

## 2. Structured Logging: JSON Format

### ทำไมต้อง Structured Logging?

```
Unstructured (ยากต่อการค้นหาและวิเคราะห์):
[2024-01-15 10:30:00] ERROR User john@example.com failed to login after 3 attempts from 192.168.1.1

Structured JSON (ค้นหาและวิเคราะห์ได้ง่าย):
{
  "level": "error",
  "timestamp": "2024-01-15T10:30:00.000Z",
  "message": "Login failed",
  "email": "john@example.com",
  "attempts": 3,
  "ip": "192.168.1.1",
  "requestId": "req_abc123",
  "service": "auth-api"
}
```

### ข้อดีของ Structured Logging

```
1. ค้นหาได้ง่าย: email="john@example.com" AND level="error"
2. Aggregate: count errors by error code
3. Alert: แจ้งเตือนเมื่อ errors > N ครั้งใน M นาที
4. Dashboard: แสดงกราฟ request rate, error rate
5. Correlation: เชื่อม logs ด้วย requestId
```

---

## 3. Winston Logger Setup

### ติดตั้ง

```bash
npm install winston winston-daily-rotate-file
npm install --save-dev @types/winston
```

### Basic Setup

```typescript
// src/utils/logger.ts
import winston from 'winston';
import DailyRotateFile from 'winston-daily-rotate-file';
import path from 'path';

const { combine, timestamp, errors, json, colorize, simple, printf, label } =
  winston.format;

// Custom log levels
const levels = {
  error: 0,
  warn: 1,
  info: 2,
  http: 3,
  debug: 4,
};

// Colors สำหรับ development
const colors = {
  error: 'red',
  warn: 'yellow',
  info: 'green',
  http: 'magenta',
  debug: 'cyan',
};

winston.addColors(colors);

// กำหนด log level ตาม environment
const logLevel = (): string => {
  const env = process.env.NODE_ENV || 'development';
  const isDevelopment = env === 'development';
  return isDevelopment ? 'debug' : 'warn';
};

export const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || logLevel(),
  levels,
  defaultMeta: {
    service: process.env.SERVICE_NAME || 'api',
    version: process.env.APP_VERSION || '1.0.0',
    environment: process.env.NODE_ENV || 'development',
  },
  transports: [], // จะ configure ด้านล่าง
  exceptionHandlers: [
    new winston.transports.File({ filename: 'logs/exceptions.log' }),
  ],
  rejectionHandlers: [
    new winston.transports.File({ filename: 'logs/rejections.log' }),
  ],
});
```

---

## 4. Log Formats: Development vs Production

```typescript
// Development format: pretty print สวยงาม
const developmentFormat = combine(
  colorize({ all: true }),
  timestamp({ format: 'HH:mm:ss' }),
  errors({ stack: true }),
  printf(({ level, message, timestamp, requestId, userId, duration, ...meta }) => {
    let log = `${timestamp} [${level}] ${message}`;

    if (requestId) log += ` | reqId=${requestId}`;
    if (userId) log += ` | user=${userId}`;
    if (duration) log += ` | ${duration}ms`;

    // แสดง metadata ถ้ามี
    const metaStr = Object.keys(meta).length
      ? '\n  ' + JSON.stringify(meta, null, 2).replace(/\n/g, '\n  ')
      : '';

    return log + metaStr;
  })
);

// Production format: JSON
const productionFormat = combine(
  timestamp({ format: 'YYYY-MM-DDTHH:mm:ss.SSSZ' }),
  errors({ stack: true }),
  json()
);

// เลือก format ตาม environment
const format =
  process.env.NODE_ENV === 'production' ? productionFormat : developmentFormat;

// Dev output:
// 10:30:15 [INFO] User registered | reqId=abc123 | user=user_456
//   { email: "john@example.com", username: "johndoe" }

// Prod output:
// {"level":"info","message":"User registered","timestamp":"2024-01-15T10:30:15.000Z",
//  "requestId":"abc123","userId":"user_456","email":"john@example.com","service":"api"}
```

---

## 5. Log Transports

```typescript
// src/utils/logger.ts (ต่อ)

// Console transport
const consoleTransport = new winston.transports.Console({
  format,
  silent: process.env.NODE_ENV === 'test', // ปิดระหว่าง test
});

// File transport สำหรับ errors
const errorFileTransport = new DailyRotateFile({
  filename: 'logs/error-%DATE%.log',
  datePattern: 'YYYY-MM-DD',
  level: 'error',
  maxSize: '20m',      // 20MB per file
  maxFiles: '30d',     // เก็บ 30 วัน
  format: combine(timestamp(), json()),
  zippedArchive: true, // compress เก่า
});

// File transport สำหรับทุก level
const combinedFileTransport = new DailyRotateFile({
  filename: 'logs/combined-%DATE%.log',
  datePattern: 'YYYY-MM-DD',
  maxSize: '50m',
  maxFiles: '14d',
  format: combine(timestamp(), json()),
  zippedArchive: true,
});

// เพิ่ม transports
logger.add(consoleTransport);

if (process.env.NODE_ENV !== 'test') {
  logger.add(errorFileTransport);
  logger.add(combinedFileTransport);
}

// Optional: ส่งไปยัง external service (Loki, Elastic, Datadog)
if (process.env.LOKI_URL) {
  const LokiTransport = require('winston-loki');
  logger.add(
    new LokiTransport({
      host: process.env.LOKI_URL,
      labels: {
        service: process.env.SERVICE_NAME,
        env: process.env.NODE_ENV,
      },
    })
  );
}
```

---

## 6. Request Logging ด้วย Morgan

```typescript
// src/middleware/requestLogger.ts
import morgan from 'morgan';
import { Request, Response } from 'express';
import { logger } from '../utils/logger';
import { IncomingMessage } from 'http';

// Custom token สำหรับ request ID
morgan.token('request-id', (req: Request) => req.id);
morgan.token('user-id', (req: Request) => req.user?.id || '-');
morgan.token('body', (req: Request) => {
  if (req.method === 'GET') return '-';
  return JSON.stringify(maskSensitiveData(req.body));
});

// Custom stream ที่ส่งไปยัง winston
const morganStream = {
  write: (message: string) => {
    logger.http(message.trim());
  },
};

// Development format: ละเอียด
const devFormat =
  ':request-id :method :url :status :response-time ms :res[content-length] - :user-id';

// Production format: เก็บ essential info
const prodFormat = JSON.stringify({
  requestId: ':request-id',
  method: ':method',
  url: ':url',
  status: ':status',
  responseTime: ':response-time',
  contentLength: ':res[content-length]',
  userAgent: ':user-agent',
  ip: ':remote-addr',
  userId: ':user-id',
});

export const requestLogger = morgan(
  process.env.NODE_ENV === 'production' ? prodFormat : devFormat,
  {
    stream: morganStream,
    skip: (req: IncomingMessage) => {
      // ข้าม health check endpoints
      return (req as Request).path === '/health' ||
             (req as Request).path === '/ready';
    },
  }
);

// Mask sensitive fields
function maskSensitiveData(data: any): any {
  if (!data || typeof data !== 'object') return data;

  const sensitiveFields = ['password', 'token', 'secret', 'credit_card', 'cvv'];
  const masked = { ...data };

  for (const field of sensitiveFields) {
    if (field in masked) {
      masked[field] = '***';
    }
  }

  return masked;
}
```

---

## 7. Correlation ID และ Request ID

```typescript
// src/middleware/requestId.ts
import { Request, Response, NextFunction } from 'express';
import { v4 as uuidv4 } from 'uuid';
import { AsyncLocalStorage } from 'async_hooks';

interface LogContext {
  requestId: string;
  userId?: string;
  startTime: number;
  method: string;
  path: string;
}

export const logContextStorage = new AsyncLocalStorage<LogContext>();

export function requestIdMiddleware(
  req: Request,
  res: Response,
  next: NextFunction
): void {
  const requestId =
    (req.headers['x-request-id'] as string) ||
    (req.headers['x-correlation-id'] as string) ||
    `req_${uuidv4().replace(/-/g, '').slice(0, 12)}`;

  req.id = requestId;
  req.startTime = Date.now();

  // ส่ง request ID กลับ
  res.set('X-Request-Id', requestId);

  // Run ใน async context
  logContextStorage.run(
    {
      requestId,
      startTime: Date.now(),
      method: req.method,
      path: req.path,
    },
    next
  );
}

// Hook ที่ set userId หลัง authenticate
export function setUserContext(userId: string): void {
  const store = logContextStorage.getStore();
  if (store) {
    store.userId = userId;
  }
}

// Logger ที่ auto-attach context
export function getContextLogger() {
  const context = logContextStorage.getStore();

  return {
    error: (message: string, meta?: Record<string, unknown>) =>
      logger.error(message, { ...context, ...meta }),
    warn: (message: string, meta?: Record<string, unknown>) =>
      logger.warn(message, { ...context, ...meta }),
    info: (message: string, meta?: Record<string, unknown>) =>
      logger.info(message, { ...context, ...meta }),
    debug: (message: string, meta?: Record<string, unknown>) =>
      logger.debug(message, { ...context, ...meta }),
  };
}
```

---

## 8. Log Context: User, RequestId, Duration

```typescript
// src/middleware/responseLogger.ts
import { Request, Response, NextFunction } from 'express';
import { logger } from '../utils/logger';

export function responseLoggerMiddleware(
  req: Request,
  res: Response,
  next: NextFunction
): void {
  // Override res.json เพื่อ log response
  const originalJson = res.json.bind(res);

  res.json = function (body: any) {
    const duration = Date.now() - req.startTime;

    // Log response info
    const logData: Record<string, unknown> = {
      requestId: req.id,
      method: req.method,
      path: req.path,
      statusCode: res.statusCode,
      duration,
      userId: req.user?.id,
    };

    if (res.statusCode >= 500) {
      logger.error('Response sent', logData);
    } else if (res.statusCode >= 400) {
      logger.warn('Response sent', logData);
    } else {
      logger.info('Response sent', { ...logData, contentLength: JSON.stringify(body).length });
    }

    return originalJson(body);
  };

  next();
}

// ใช้งานใน route handler
async function getUser(req: Request, res: Response) {
  const ctxLogger = getContextLogger();
  const start = Date.now();

  ctxLogger.debug('Fetching user from database');

  const user = await pool.query('SELECT * FROM users WHERE id = $1', [req.params.id]);

  ctxLogger.debug('User fetched', {
    userId: req.params.id,
    duration: Date.now() - start,
    found: user.rows.length > 0,
  });

  if (!user.rows[0]) {
    ctxLogger.warn('User not found', { userId: req.params.id });
    return res.status(404).json({ success: false, error: { code: 'NOT_FOUND' } });
  }

  ctxLogger.info('User retrieved', { userId: user.rows[0].id });
  res.json({ success: true, data: user.rows[0] });
}
```

---

## 9. Sensitive Data Masking

```typescript
// src/utils/dataMasking.ts

const SENSITIVE_FIELDS = new Set([
  'password',
  'passwordHash',
  'password_hash',
  'secret',
  'token',
  'accessToken',
  'refreshToken',
  'access_token',
  'refresh_token',
  'apiKey',
  'api_key',
  'privateKey',
  'private_key',
  'creditCard',
  'credit_card',
  'cardNumber',
  'card_number',
  'cvv',
  'cvc',
  'ssn',
  'passport',
]);

const PARTIAL_MASK_FIELDS = new Set([
  'email',
  'phone',
  'ip',
]);

export function maskSensitiveData(data: any, depth: number = 0): any {
  if (depth > 10) return '[MAX_DEPTH]'; // ป้องกัน circular reference
  if (data === null || data === undefined) return data;

  if (typeof data === 'string') return data;
  if (typeof data !== 'object') return data;

  if (Array.isArray(data)) {
    return data.map((item) => maskSensitiveData(item, depth + 1));
  }

  const masked: Record<string, any> = {};

  for (const [key, value] of Object.entries(data)) {
    const lowerKey = key.toLowerCase();

    if (SENSITIVE_FIELDS.has(key) || SENSITIVE_FIELDS.has(lowerKey)) {
      masked[key] = '***MASKED***';
    } else if (PARTIAL_MASK_FIELDS.has(key) || PARTIAL_MASK_FIELDS.has(lowerKey)) {
      masked[key] = partialMask(String(value), key);
    } else if (typeof value === 'object') {
      masked[key] = maskSensitiveData(value, depth + 1);
    } else {
      masked[key] = value;
    }
  }

  return masked;
}

function partialMask(value: string, fieldType: string): string {
  if (fieldType === 'email') {
    const [local, domain] = value.split('@');
    if (!domain) return '***@***';
    return `${local.slice(0, 2)}***@${domain}`;
    // john@example.com → jo***@example.com
  }

  if (fieldType === 'phone') {
    return value.replace(/(\d{3})\d{4}(\d{3,4})/, '$1****$2');
    // 0812345678 → 081****678
  }

  if (fieldType === 'ip') {
    const parts = value.split('.');
    if (parts.length === 4) {
      return `${parts[0]}.${parts[1]}.***.***`;
    }
    return '***';
  }

  return `${value.slice(0, 3)}***`;
}

// Winston format สำหรับ mask ข้อมูล
import { format } from 'winston';

export const maskFormat = format((info) => {
  if (info.body) info.body = maskSensitiveData(info.body);
  if (info.query) info.query = maskSensitiveData(info.query);
  if (info.data) info.data = maskSensitiveData(info.data);
  return info;
});
```

---

## 10. Log Rotation

```typescript
// src/utils/logger.ts — Log rotation config

const rotateConfig = {
  // เปลี่ยนไฟล์ทุกวัน
  datePattern: 'YYYY-MM-DD',

  // ขนาดสูงสุดต่อไฟล์
  maxSize: '100m', // 100 MB

  // เก็บนานแค่ไหน
  maxFiles: '30d', // 30 วัน

  // Compress ไฟล์เก่า
  zippedArchive: true,

  // ชื่อไฟล์
  filename: 'logs/%DATE%-app.log',
};

// ตั้งค่า directories
import fs from 'fs';
import path from 'path';

const logDir = path.join(process.cwd(), 'logs');
if (!fs.existsSync(logDir)) {
  fs.mkdirSync(logDir, { recursive: true });
}

// Events สำหรับ monitoring rotation
const rotateTransport = new DailyRotateFile(rotateConfig);

rotateTransport.on('rotate', (oldFilename, newFilename) => {
  logger.info('Log file rotated', { oldFilename, newFilename });
});

rotateTransport.on('archive', (zipFilename) => {
  logger.info('Log file archived', { zipFilename });
});

rotateTransport.on('deleted', (deletedFilename) => {
  logger.info('Old log file deleted', { deletedFilename });
});

rotateTransport.on('error', (error) => {
  console.error('Log rotation error:', error);
});
```

---

## 11. Database Query Logging

```typescript
// src/db/pool.ts — with query logging
import { Pool, QueryConfig } from 'pg';
import { logger } from '../utils/logger';
import { logContextStorage } from '../middleware/requestId';

const SLOW_QUERY_THRESHOLD_MS = 1000; // 1 second

class InstrumentedPool {
  private pool: Pool;

  constructor(config: any) {
    this.pool = new Pool(config);
    this.setupLogging();
  }

  private setupLogging(): void {
    this.pool.on('connect', () => {
      logger.debug('New DB connection established');
    });

    this.pool.on('error', (err) => {
      logger.error('Unexpected DB pool error', { error: err.message, stack: err.stack });
    });
  }

  async query<T = any>(
    queryText: string | QueryConfig,
    values?: any[]
  ): Promise<any> {
    const start = Date.now();
    const context = logContextStorage.getStore();

    // Sanitize query สำหรับ logging
    const queryStr = typeof queryText === 'string' ? queryText : queryText.text;
    const sanitizedQuery = queryStr.replace(/\s+/g, ' ').trim();

    try {
      const result = await this.pool.query(queryText, values);
      const duration = Date.now() - start;

      const logData = {
        query: sanitizedQuery.slice(0, 200), // ตัดยาวเกินไป
        duration,
        rows: result.rowCount,
        requestId: context?.requestId,
      };

      if (duration > SLOW_QUERY_THRESHOLD_MS) {
        logger.warn('Slow query detected', {
          ...logData,
          query: sanitizedQuery, // แสดง query เต็มถ้าช้า
          values: values?.map((v) =>
            typeof v === 'string' && v.length > 50 ? v.slice(0, 50) + '...' : v
          ),
        });
      } else if (process.env.NODE_ENV === 'development') {
        logger.debug('Query executed', logData);
      }

      return result;
    } catch (error) {
      const duration = Date.now() - start;

      logger.error('Query failed', {
        query: sanitizedQuery,
        duration,
        error: (error as Error).message,
        requestId: context?.requestId,
      });

      throw error;
    }
  }

  async getClient() {
    return this.pool.connect();
  }

  async end() {
    return this.pool.end();
  }

  // Stats
  get poolStats() {
    return {
      total: this.pool.totalCount,
      idle: this.pool.idleCount,
      waiting: this.pool.waitingCount,
    };
  }
}

export default new InstrumentedPool({
  connectionString: process.env.DATABASE_URL,
  max: 20,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});
```

---

## 12. Redis Command Logging

```typescript
// src/db/redis.ts — with command logging
import Redis from 'ioredis';
import { logger } from '../utils/logger';

const SLOW_COMMAND_THRESHOLD_MS = 100;

class InstrumentedRedis extends Redis {
  constructor(url: string, options?: any) {
    super(url, options);
    this.setupMonitoring();
  }

  private setupMonitoring(): void {
    this.on('connect', () => logger.info('Redis connected'));
    this.on('error', (err) => logger.error('Redis error', { error: err.message }));
    this.on('close', () => logger.warn('Redis connection closed'));
    this.on('reconnecting', () => logger.info('Redis reconnecting...'));

    // Monitor ทุก command
    this.monitor((err, monitor) => {
      if (err) {
        logger.error('Redis monitor error', { error: err.message });
        return;
      }

      monitor.on('monitor', (time, args, source) => {
        const command = args[0]?.toUpperCase();

        // Skip ไม่ log commands ที่ verbose เกินไป
        const skipCommands = ['PING', 'INFO', 'CLIENT'];
        if (skipCommands.includes(command)) return;

        logger.debug('Redis command', {
          command,
          key: args[1],
          source,
        });
      });
    });
  }

  // Override methods เพื่อ track performance
  async get(key: string): Promise<string | null> {
    const start = Date.now();
    const result = await super.get(key);
    const duration = Date.now() - start;

    if (duration > SLOW_COMMAND_THRESHOLD_MS) {
      logger.warn('Slow Redis GET', { key, duration });
    }

    // Track hit/miss
    logger.debug('Redis GET', { key, hit: result !== null, duration });

    return result;
  }
}

export const redis = new InstrumentedRedis(
  process.env.REDIS_URL || 'redis://localhost:6379',
  {
    maxRetriesPerRequest: 3,
    retryStrategy: (times: number) => {
      if (times > 3) {
        logger.error('Redis max retries reached');
        return null;
      }
      const delay = Math.min(times * 200, 2000);
      logger.warn('Redis reconnect attempt', { attempt: times, delayMs: delay });
      return delay;
    },
  }
);
```

---

## 13. Health Check Endpoint

```typescript
// src/routes/health.ts
import { Router, Request, Response } from 'express';
import pool from '../db/pool';
import { redis } from '../db/redis';
import { s3Client } from '../config/minio';
import { HeadBucketCommand } from '@aws-sdk/client-s3';
import os from 'os';

const router = Router();

interface HealthCheck {
  status: 'healthy' | 'degraded' | 'unhealthy';
  timestamp: string;
  uptime: number;
  version: string;
  checks: {
    [service: string]: {
      status: 'up' | 'down' | 'degraded';
      responseTime?: number;
      details?: any;
    };
  };
}

// GET /health — Basic health check
router.get('/health', (req: Request, res: Response) => {
  res.json({
    status: 'ok',
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
    version: process.env.APP_VERSION || '1.0.0',
  });
});

// GET /ready — Readiness check (ตรวจสอบ dependencies ทั้งหมด)
router.get('/ready', async (req: Request, res: Response) => {
  const checks: HealthCheck['checks'] = {};
  let overallStatus: HealthCheck['status'] = 'healthy';

  // 1. PostgreSQL check
  const dbStart = Date.now();
  try {
    await pool.query('SELECT 1');
    checks.postgresql = {
      status: 'up',
      responseTime: Date.now() - dbStart,
      details: pool.poolStats,
    };
  } catch (error) {
    checks.postgresql = {
      status: 'down',
      responseTime: Date.now() - dbStart,
      details: { error: (error as Error).message },
    };
    overallStatus = 'unhealthy';
  }

  // 2. Redis check
  const redisStart = Date.now();
  try {
    const pong = await redis.ping();
    const info = await redis.info('memory');
    const usedMemory = info.match(/used_memory_human:(.+)/)?.[1]?.trim();

    checks.redis = {
      status: pong === 'PONG' ? 'up' : 'degraded',
      responseTime: Date.now() - redisStart,
      details: { usedMemory },
    };
  } catch (error) {
    checks.redis = {
      status: 'down',
      responseTime: Date.now() - redisStart,
      details: { error: (error as Error).message },
    };
    overallStatus = overallStatus === 'unhealthy' ? 'unhealthy' : 'degraded';
  }

  // 3. MinIO check
  const minioStart = Date.now();
  try {
    await s3Client.send(new HeadBucketCommand({ Bucket: 'health-check' }));
    checks.minio = {
      status: 'up',
      responseTime: Date.now() - minioStart,
    };
  } catch (error: any) {
    // NoSuchBucket ก็ถือว่า MinIO ทำงานได้
    if (error?.name === 'NoSuchBucket' || error?.$metadata?.httpStatusCode === 404) {
      checks.minio = {
        status: 'up',
        responseTime: Date.now() - minioStart,
      };
    } else {
      checks.minio = {
        status: 'down',
        responseTime: Date.now() - minioStart,
        details: { error: error?.message },
      };
      overallStatus = overallStatus === 'unhealthy' ? 'unhealthy' : 'degraded';
    }
  }

  const response: HealthCheck = {
    status: overallStatus,
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
    version: process.env.APP_VERSION || '1.0.0',
    checks,
  };

  const statusCode =
    overallStatus === 'healthy' ? 200 :
    overallStatus === 'degraded' ? 207 : 503;

  res.status(statusCode).json(response);
});

// GET /live — Liveness check (แค่ตรวจสอบว่า process ยังทำงานอยู่)
router.get('/live', (req: Request, res: Response) => {
  // ถ้า respond ได้ = alive
  res.json({
    status: 'alive',
    timestamp: new Date().toISOString(),
  });
});

// GET /metrics — Basic metrics endpoint
router.get('/metrics', async (req: Request, res: Response) => {
  const memUsage = process.memoryUsage();
  const cpuUsage = process.cpuUsage();

  res.json({
    process: {
      uptime: process.uptime(),
      pid: process.pid,
      memory: {
        heapUsed: memUsage.heapUsed,
        heapTotal: memUsage.heapTotal,
        rss: memUsage.rss,
        external: memUsage.external,
        usedPercent: Math.round((memUsage.heapUsed / memUsage.heapTotal) * 100),
      },
      cpu: {
        user: cpuUsage.user,
        system: cpuUsage.system,
      },
    },
    os: {
      loadAvg: os.loadavg(),
      freeMemory: os.freemem(),
      totalMemory: os.totalmem(),
      cpus: os.cpus().length,
    },
    database: pool.poolStats,
    timestamp: new Date().toISOString(),
  });
});

export default router;
```

---

## 14. Metrics Basics

```typescript
// src/utils/metrics.ts — Simple in-memory metrics

interface Counter {
  value: number;
  labels: Record<string, string>;
}

interface Histogram {
  sum: number;
  count: number;
  buckets: number[];
  values: number[];
  labels: Record<string, string>;
}

class MetricsCollector {
  private counters: Map<string, Counter> = new Map();
  private histograms: Map<string, Histogram> = new Map();
  private gauges: Map<string, number> = new Map();

  // Counter: เพิ่มค่าได้อย่างเดียว
  increment(name: string, labels: Record<string, string> = {}, value: number = 1): void {
    const key = this.makeKey(name, labels);
    const counter = this.counters.get(key) || { value: 0, labels };
    counter.value += value;
    this.counters.set(key, counter);
  }

  // Gauge: ค่าปัจจุบัน เพิ่มหรือลดได้
  setGauge(name: string, value: number): void {
    this.gauges.set(name, value);
  }

  // Histogram: distribution ของค่า
  observe(name: string, value: number, labels: Record<string, string> = {}): void {
    const key = this.makeKey(name, labels);
    const hist = this.histograms.get(key) || {
      sum: 0,
      count: 0,
      buckets: [10, 50, 100, 250, 500, 1000, 2500, 5000],
      values: new Array(9).fill(0), // +1 สำหรับ +Inf
      labels,
    };

    hist.sum += value;
    hist.count++;

    // ใส่ใน buckets
    for (let i = 0; i < hist.buckets.length; i++) {
      if (value <= hist.buckets[i]) {
        hist.values[i]++;
      }
    }
    hist.values[hist.buckets.length]++; // +Inf

    this.histograms.set(key, hist);
  }

  // ดึงข้อมูลทั้งหมด
  getAll(): Record<string, any> {
    const result: Record<string, any> = {};

    for (const [key, counter] of this.counters) {
      result[key] = { type: 'counter', value: counter.value, labels: counter.labels };
    }

    for (const [key, hist] of this.histograms) {
      result[key] = {
        type: 'histogram',
        sum: hist.sum,
        count: hist.count,
        avg: hist.count > 0 ? Math.round(hist.sum / hist.count) : 0,
        labels: hist.labels,
      };
    }

    for (const [key, value] of this.gauges) {
      result[key] = { type: 'gauge', value };
    }

    return result;
  }

  private makeKey(name: string, labels: Record<string, string>): string {
    const labelStr = Object.entries(labels)
      .sort(([a], [b]) => a.localeCompare(b))
      .map(([k, v]) => `${k}="${v}"`)
      .join(',');
    return labelStr ? `${name}{${labelStr}}` : name;
  }
}

export const metrics = new MetricsCollector();

// Middleware สำหรับเก็บ metrics
export function metricsMiddleware(req: Request, res: Response, next: NextFunction): void {
  metrics.increment('http_requests_total', { method: req.method, path: req.path });

  const end = () => {
    const duration = Date.now() - req.startTime;

    metrics.observe('http_request_duration_ms', duration, {
      method: req.method,
      path: req.route?.path || req.path,
      status: String(res.statusCode),
    });

    if (res.statusCode >= 400) {
      metrics.increment('http_errors_total', {
        method: req.method,
        status: String(res.statusCode),
      });
    }
  };

  res.on('finish', end);
  next();
}
```

---

## 15. Process Metrics

```typescript
// src/utils/processMetrics.ts
import os from 'os';
import { metrics } from './metrics';

// Collect process metrics ทุก 30 วินาที
export function startProcessMetricsCollection(): NodeJS.Timeout {
  return setInterval(() => {
    const memUsage = process.memoryUsage();
    const cpuUsage = process.cpuUsage();

    // Memory
    metrics.setGauge('process_heap_used_bytes', memUsage.heapUsed);
    metrics.setGauge('process_heap_total_bytes', memUsage.heapTotal);
    metrics.setGauge('process_rss_bytes', memUsage.rss);

    // CPU
    metrics.setGauge('process_cpu_user_microseconds', cpuUsage.user);
    metrics.setGauge('process_cpu_system_microseconds', cpuUsage.system);

    // OS
    metrics.setGauge('os_free_memory_bytes', os.freemem());
    metrics.setGauge('os_load_average_1m', os.loadavg()[0]);

    // Database pool
    const poolStats = pool.poolStats;
    metrics.setGauge('db_pool_total', poolStats.total);
    metrics.setGauge('db_pool_idle', poolStats.idle);
    metrics.setGauge('db_pool_waiting', poolStats.waiting);

    // Event loop lag (simple approximation)
    const start = Date.now();
    setImmediate(() => {
      metrics.observe('event_loop_lag_ms', Date.now() - start);
    });
  }, 30000);
}

// PostgreSQL metrics
export async function collectDatabaseMetrics(): Promise<void> {
  try {
    // Connection stats
    const connStats = await pool.query(`
      SELECT
        count(*) as total,
        count(*) FILTER (WHERE state = 'active') as active,
        count(*) FILTER (WHERE state = 'idle') as idle
      FROM pg_stat_activity
      WHERE datname = current_database()
    `);

    const stats = connStats.rows[0];
    metrics.setGauge('pg_connections_total', Number(stats.total));
    metrics.setGauge('pg_connections_active', Number(stats.active));
    metrics.setGauge('pg_connections_idle', Number(stats.idle));

    // Cache hit rate
    const cacheStats = await pool.query(`
      SELECT
        sum(heap_blks_hit) as hits,
        sum(heap_blks_read) as reads
      FROM pg_statio_user_tables
    `);

    const cacheRow = cacheStats.rows[0];
    const total = Number(cacheRow.hits) + Number(cacheRow.reads);
    const hitRate = total > 0 ? (Number(cacheRow.hits) / total) * 100 : 0;

    metrics.setGauge('pg_cache_hit_rate_percent', Math.round(hitRate));
  } catch (error) {
    logger.error('Failed to collect DB metrics', { error });
  }
}

// Redis metrics
export async function collectRedisMetrics(): Promise<void> {
  try {
    const info = await redis.info();
    const lines = info.split('\r\n');

    const getValue = (key: string): number => {
      const line = lines.find((l) => l.startsWith(`${key}:`));
      return Number(line?.split(':')[1]) || 0;
    };

    metrics.setGauge('redis_used_memory_bytes', getValue('used_memory'));
    metrics.setGauge('redis_connected_clients', getValue('connected_clients'));
    metrics.setGauge('redis_total_commands_processed', getValue('total_commands_processed'));

    const hits = getValue('keyspace_hits');
    const misses = getValue('keyspace_misses');
    const total = hits + misses;
    const hitRate = total > 0 ? (hits / total) * 100 : 0;

    metrics.setGauge('redis_hit_rate_percent', Math.round(hitRate));
    metrics.setGauge('redis_keyspace_hits', hits);
    metrics.setGauge('redis_keyspace_misses', misses);
    metrics.setGauge('redis_ops_per_sec', getValue('instantaneous_ops_per_sec'));
  } catch (error) {
    logger.error('Failed to collect Redis metrics', { error });
  }
}
```

---

## 16. Full Working Logging Setup

```typescript
// src/utils/logger.ts — Complete implementation
import winston from 'winston';
import DailyRotateFile from 'winston-daily-rotate-file';
import path from 'path';
import fs from 'fs';

// สร้าง logs directory
const logDir = path.join(process.cwd(), 'logs');
if (!fs.existsSync(logDir)) {
  fs.mkdirSync(logDir, { recursive: true });
}

const isDevelopment = process.env.NODE_ENV === 'development';
const isTest = process.env.NODE_ENV === 'test';

// Custom log levels
const levels = { error: 0, warn: 1, info: 2, http: 3, debug: 4 };
const levelColors = {
  error: 'red bold',
  warn: 'yellow',
  info: 'green',
  http: 'magenta',
  debug: 'cyan',
};

winston.addColors(levelColors);

// Development format
const devFormat = winston.format.combine(
  winston.format.colorize({ all: true }),
  winston.format.timestamp({ format: 'HH:mm:ss.SSS' }),
  winston.format.errors({ stack: true }),
  winston.format.printf(({ level, message, timestamp, stack, ...meta }) => {
    let output = `${timestamp} ${level}: ${message}`;

    // แสดง stack trace
    if (stack) {
      output += `\n${stack}`;
    }

    // แสดง metadata
    const cleanMeta = { ...meta };
    delete cleanMeta.service;
    delete cleanMeta.version;
    delete cleanMeta.environment;

    if (Object.keys(cleanMeta).length > 0) {
      const metaStr = JSON.stringify(cleanMeta, null, 2);
      output += `\n${metaStr.split('\n').map((l) => '  ' + l).join('\n')}`;
    }

    return output;
  })
);

// Production format
const prodFormat = winston.format.combine(
  winston.format.timestamp(),
  winston.format.errors({ stack: true }),
  winston.format((info) => {
    // Mask sensitive data
    if (info.body) info.body = maskSensitiveData(info.body);
    if (info.headers) {
      const h = { ...info.headers } as any;
      if (h.authorization) h.authorization = 'Bearer ***';
      if (h.cookie) h.cookie = '***';
      info.headers = h;
    }
    return info;
  })(),
  winston.format.json()
);

function maskSensitiveData(data: any): any {
  if (!data || typeof data !== 'object') return data;
  const sensitive = ['password', 'token', 'secret', 'authorization'];
  return Object.fromEntries(
    Object.entries(data).map(([k, v]) =>
      sensitive.some((s) => k.toLowerCase().includes(s)) ? [k, '***'] : [k, v]
    )
  );
}

export const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || (isDevelopment ? 'debug' : 'info'),
  levels,
  defaultMeta: {
    service: process.env.SERVICE_NAME || 'api',
    version: process.env.APP_VERSION || '1.0.0',
    environment: process.env.NODE_ENV,
  },
  transports: [
    // Console
    new winston.transports.Console({
      format: isDevelopment ? devFormat : prodFormat,
      silent: isTest,
    }),

    // Error file
    ...(isTest
      ? []
      : [
          new DailyRotateFile({
            level: 'error',
            filename: path.join(logDir, 'error-%DATE%.log'),
            datePattern: 'YYYY-MM-DD',
            maxSize: '20m',
            maxFiles: '30d',
            format: winston.format.combine(winston.format.timestamp(), winston.format.json()),
            zippedArchive: true,
          }),

          // Combined file
          new DailyRotateFile({
            filename: path.join(logDir, 'combined-%DATE%.log'),
            datePattern: 'YYYY-MM-DD',
            maxSize: '100m',
            maxFiles: '14d',
            format: winston.format.combine(winston.format.timestamp(), winston.format.json()),
            zippedArchive: true,
          }),
        ]),
  ],

  exceptionHandlers: isTest
    ? []
    : [new DailyRotateFile({
        filename: path.join(logDir, 'exceptions-%DATE%.log'),
        datePattern: 'YYYY-MM-DD',
        maxFiles: '30d',
      })],

  rejectionHandlers: isTest
    ? []
    : [new DailyRotateFile({
        filename: path.join(logDir, 'rejections-%DATE%.log'),
        datePattern: 'YYYY-MM-DD',
        maxFiles: '30d',
      })],
});

// Helper functions
export const log = {
  error: (message: string, meta?: Record<string, unknown>) =>
    logger.error(message, meta),
  warn: (message: string, meta?: Record<string, unknown>) =>
    logger.warn(message, meta),
  info: (message: string, meta?: Record<string, unknown>) =>
    logger.info(message, meta),
  http: (message: string, meta?: Record<string, unknown>) =>
    logger.http(message, meta),
  debug: (message: string, meta?: Record<string, unknown>) =>
    logger.debug(message, meta),
};

export default logger;
```

### ทดสอบการ Logging

```typescript
// ทดสอบทุก level
logger.error('Database connection failed', { host: 'localhost', port: 5432 });
logger.warn('High memory usage', { percent: 85 });
logger.info('User logged in', { userId: 'user_123', ip: '192.168.1.1' });
logger.http('GET /api/users 200 45ms');
logger.debug('Cache hit', { key: 'user:123', ttl: 3600 });

// ทดสอบ sensitive data masking
logger.info('User request', {
  body: {
    email: 'john@example.com',
    password: 'secret123', // จะถูก mask
    token: 'abc.def.ghi',  // จะถูก mask
    name: 'John',          // แสดงปกติ
  },
});
// Output: { body: { email: 'john@example.com', password: '***', token: '***', name: 'John' } }
```

### Docker Health Check

```dockerfile
# Dockerfile
FROM node:20-alpine

WORKDIR /app
COPY . .
RUN npm ci --production

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1

CMD ["node", "dist/server.js"]
```

### docker-compose.yml พร้อม health check

```yaml
version: '3.8'

services:
  api:
    build: .
    ports:
      - '3000:3000'
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgresql://user:pass@postgres:5432/myapp
      - REDIS_URL=redis://redis:6379
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ['CMD', 'wget', '-qO-', 'http://localhost:3000/ready']
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  postgres:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: pass
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U user -d myapp']
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7
    healthcheck:
      test: ['CMD', 'redis-cli', 'ping']
      interval: 10s
      timeout: 5s
      retries: 5
```

### package.json scripts

```json
{
  "scripts": {
    "dev": "NODE_ENV=development LOG_LEVEL=debug ts-node-dev src/server.ts",
    "build": "tsc",
    "start": "NODE_ENV=production node dist/server.ts",
    "logs:error": "tail -f logs/error-$(date +%Y-%m-%d).log | jq",
    "logs:all": "tail -f logs/combined-$(date +%Y-%m-%d).log | jq",
    "logs:live": "tail -f logs/combined-$(date +%Y-%m-%d).log | jq 'select(.level==\"error\" or .level==\"warn\")'"
  },
  "dependencies": {
    "morgan": "^1.10.0",
    "winston": "^3.11.0",
    "winston-daily-rotate-file": "^4.7.1"
  },
  "devDependencies": {
    "@types/morgan": "^1.9.9"
  }
}
```

---

## สรุป

| หัวข้อ | Best Practice |
|--------|--------------|
| Log Levels | Error/Warn สำหรับ alerts, Info สำหรับ business events |
| Format | JSON ใน production, Pretty ใน development |
| Rotation | Daily + 30 วัน retention |
| Masking | ซ่อน password, token, PII เสมอ |
| Correlation | Request ID ทุก log |
| Health Check | /health (basic), /ready (full), /live (liveness) |
| Metrics | Request count, latency histogram, error rate |
| Process | Memory, CPU, event loop lag |
| Database | Pool stats, slow query detection |
| Redis | Hit rate, memory usage, ops/sec |

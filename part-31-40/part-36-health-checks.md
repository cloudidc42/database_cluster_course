# Part 36: Health Checks และ Readiness Probes

## บทนำ

ในระบบ Production ที่ทำงานบน Container Orchestration อย่าง Kubernetes หรือ Docker Swarm การตรวจสอบสถานะของ Service มีความสำคัญอย่างยิ่ง Health Checks ช่วยให้ระบบรู้ว่า Service ของเราพร้อมรับ Traffic หรือไม่ และยังทำงานอยู่หรือเปล่า

ในบทนี้เราจะเรียนรู้:
- ประเภทของ Health Checks ที่แตกต่างกัน
- วิธีออกแบบ Health Check Endpoints
- การตรวจสอบ Dependencies (PostgreSQL, Redis, MinIO)
- การ Implement ด้วย TypeScript
- การตั้งค่าใน Docker และ Kubernetes

---

## 1. ประเภทของ Health Checks

### 1.1 Liveness Probe

**Liveness Probe** บอกว่า Application ยังทำงานอยู่หรือไม่ หาก Liveness Probe fail Kubernetes จะ restart Container นั้น

**วัตถุประสงค์:**
- ตรวจสอบว่า Process ยังทำงานอยู่
- ตรวจสอบว่า Application ไม่ได้ Deadlock
- ตรวจสอบว่า Memory ไม่ล้น (Out of Memory)

**ตัวอย่างสถานการณ์ที่ต้องการ Restart:**
```
- Application เข้าสู่ Infinite Loop
- Memory Leak ทำให้ระบบช้ามาก
- Database Connection Pool หมด และไม่สามารถ Recover ได้
- Thread Deadlock
```

### 1.2 Readiness Probe

**Readiness Probe** บอกว่า Application พร้อมรับ Traffic หรือไม่ หาก Readiness Probe fail Kubernetes จะหยุดส่ง Traffic ไปยัง Pod นั้น แต่จะไม่ Restart

**วัตถุประสงค์:**
- ตรวจสอบว่า Dependencies พร้อมแล้ว (DB, Cache, etc.)
- ตรวจสอบว่า Application โหลด Configuration เสร็จแล้ว
- ตรวจสอบว่า Warm-up เสร็จแล้ว

**ตัวอย่างสถานการณ์ที่ต้องหยุดรับ Traffic:**
```
- PostgreSQL Connection ล้มเหลว
- Redis ไม่ตอบสนอง
- Application กำลัง Rolling Update
- Application กำลัง Load Initial Data
```

### 1.3 Startup Probe

**Startup Probe** ใช้สำหรับ Application ที่ใช้เวลา Startup นาน เช่น Application ที่ต้องโหลด Machine Learning Model หรือ Migrate Database

**วัตถุประสงค์:**
- ป้องกัน Liveness Probe kill Application ระหว่าง Startup
- ให้เวลา Application Startup เพิ่มขึ้น

```
Startup Probe → ตรวจจนผ่านครั้งแรก
     ↓
Liveness Probe + Readiness Probe เริ่มทำงาน
```

---

## 2. Health Check Endpoint Design

### 2.1 โครงสร้าง Endpoint

เราจะออกแบบ Endpoints ดังนี้:

```
GET /health        → ข้อมูลสุขภาพโดยรวม (Aggregated)
GET /health/live   → Liveness check (process running?)
GET /health/ready  → Readiness check (all deps ready?)
GET /health/startup → Startup check (initialization done?)
GET /metrics       → Prometheus metrics
```

### 2.2 Response Format

```json
{
  "status": "healthy",
  "timestamp": "2024-01-15T10:30:00.000Z",
  "uptime": 3600,
  "version": "1.0.0",
  "environment": "production",
  "checks": {
    "database": {
      "status": "healthy",
      "responseTime": 5,
      "message": "Connected to PostgreSQL"
    },
    "redis": {
      "status": "healthy",
      "responseTime": 1,
      "message": "Redis PING OK"
    },
    "storage": {
      "status": "healthy",
      "responseTime": 12,
      "message": "MinIO accessible"
    }
  }
}
```

**สถานะที่เป็นไปได้:**
- `healthy` → ทุกอย่างปกติ (HTTP 200)
- `degraded` → บาง Component มีปัญหาแต่ยังทำงานได้ (HTTP 200)
- `unhealthy` → Component สำคัญมีปัญหา (HTTP 503)

---

## 3. การ Implement HealthCheckService

### 3.1 โครงสร้างโปรเจค

```
src/
├── health/
│   ├── HealthCheckService.ts
│   ├── checks/
│   │   ├── PostgreSQLCheck.ts
│   │   ├── RedisCheck.ts
│   │   ├── MinIOCheck.ts
│   │   └── MemoryCheck.ts
│   ├── routes/
│   │   └── healthRoutes.ts
│   └── types.ts
├── middleware/
│   └── healthMiddleware.ts
└── index.ts
```

### 3.2 Types Definition

```typescript
// src/health/types.ts

export type HealthStatus = 'healthy' | 'degraded' | 'unhealthy';

export interface CheckResult {
  status: HealthStatus;
  responseTime: number;  // milliseconds
  message: string;
  details?: Record<string, unknown>;
  error?: string;
  lastChecked: Date;
}

export interface HealthCheckResult {
  status: HealthStatus;
  timestamp: Date;
  uptime: number;          // seconds
  version: string;
  environment: string;
  checks: Record<string, CheckResult>;
}

export interface HealthChecker {
  name: string;
  critical: boolean;       // ถ้า fail จะทำให้ overall status เป็น unhealthy
  check(): Promise<CheckResult>;
}

export interface HealthCheckConfig {
  cacheTTL: number;        // milliseconds, กี่ ms ถึง cache ผลไว้
  timeout: number;         // milliseconds, timeout ต่อ check
  retries: number;         // จำนวนครั้งที่ retry
}
```

### 3.3 HealthCheckService หลัก

```typescript
// src/health/HealthCheckService.ts

import { 
  HealthChecker, 
  HealthCheckResult, 
  CheckResult,
  HealthStatus,
  HealthCheckConfig 
} from './types';

export class HealthCheckService {
  private checkers: Map<string, HealthChecker> = new Map();
  private cache: Map<string, { result: CheckResult; expiresAt: number }> = new Map();
  private config: HealthCheckConfig;
  private startTime: number = Date.now();

  constructor(config: Partial<HealthCheckConfig> = {}) {
    this.config = {
      cacheTTL: config.cacheTTL ?? 5000,      // 5 seconds default
      timeout: config.timeout ?? 3000,         // 3 seconds timeout
      retries: config.retries ?? 1,
    };
  }

  /**
   * ลงทะเบียน Health Checker
   */
  register(checker: HealthChecker): this {
    this.checkers.set(checker.name, checker);
    console.log(`[HealthCheck] Registered checker: ${checker.name}`);
    return this;
  }

  /**
   * ลบ Health Checker
   */
  unregister(name: string): this {
    this.checkers.delete(name);
    this.cache.delete(name);
    return this;
  }

  /**
   * Run check พร้อม timeout และ cache
   */
  private async runCheck(checker: HealthChecker): Promise<CheckResult> {
    const cacheKey = checker.name;
    const now = Date.now();

    // ตรวจ cache ก่อน
    const cached = this.cache.get(cacheKey);
    if (cached && cached.expiresAt > now) {
      return cached.result;
    }

    const startTime = Date.now();
    let result: CheckResult;

    try {
      // Run check with timeout
      const checkPromise = checker.check();
      const timeoutPromise = new Promise<CheckResult>((_, reject) => {
        setTimeout(() => reject(new Error(`Check timeout after ${this.config.timeout}ms`)), this.config.timeout);
      });

      result = await Promise.race([checkPromise, timeoutPromise]);
      result.responseTime = Date.now() - startTime;
      result.lastChecked = new Date();
    } catch (error) {
      result = {
        status: 'unhealthy',
        responseTime: Date.now() - startTime,
        message: 'Check failed',
        error: error instanceof Error ? error.message : String(error),
        lastChecked: new Date(),
      };
    }

    // บันทึก cache
    this.cache.set(cacheKey, {
      result,
      expiresAt: now + this.config.cacheTTL,
    });

    return result;
  }

  /**
   * Run checks ทั้งหมด
   */
  async checkAll(): Promise<HealthCheckResult> {
    const checkPromises = Array.from(this.checkers.entries()).map(
      async ([name, checker]) => {
        const result = await this.runCheck(checker);
        return { name, result, critical: checker.critical };
      }
    );

    const results = await Promise.all(checkPromises);

    // สร้าง checks object
    const checks: Record<string, CheckResult> = {};
    let overallStatus: HealthStatus = 'healthy';

    for (const { name, result, critical } of results) {
      checks[name] = result;

      if (result.status === 'unhealthy' && critical) {
        overallStatus = 'unhealthy';
      } else if (result.status === 'unhealthy' && !critical && overallStatus !== 'unhealthy') {
        overallStatus = 'degraded';
      } else if (result.status === 'degraded' && overallStatus === 'healthy') {
        overallStatus = 'degraded';
      }
    }

    return {
      status: overallStatus,
      timestamp: new Date(),
      uptime: Math.floor((Date.now() - this.startTime) / 1000),
      version: process.env.APP_VERSION ?? '1.0.0',
      environment: process.env.NODE_ENV ?? 'development',
      checks,
    };
  }

  /**
   * Run เฉพาะ critical checks (สำหรับ liveness)
   */
  async checkLiveness(): Promise<HealthCheckResult> {
    const criticalCheckers = Array.from(this.checkers.entries())
      .filter(([_, checker]) => checker.critical);

    const checkPromises = criticalCheckers.map(
      async ([name, checker]) => {
        const result = await this.runCheck(checker);
        return { name, result };
      }
    );

    const results = await Promise.all(checkPromises);
    const checks: Record<string, CheckResult> = {};
    let overallStatus: HealthStatus = 'healthy';

    for (const { name, result } of results) {
      checks[name] = result;
      if (result.status === 'unhealthy') {
        overallStatus = 'unhealthy';
      }
    }

    return {
      status: overallStatus,
      timestamp: new Date(),
      uptime: Math.floor((Date.now() - this.startTime) / 1000),
      version: process.env.APP_VERSION ?? '1.0.0',
      environment: process.env.NODE_ENV ?? 'development',
      checks,
    };
  }

  /**
   * ล้าง cache ทั้งหมด
   */
  clearCache(): void {
    this.cache.clear();
  }

  /**
   * Get uptime in seconds
   */
  getUptime(): number {
    return Math.floor((Date.now() - this.startTime) / 1000);
  }
}
```

---

## 4. Individual Health Checkers

### 4.1 PostgreSQL Health Check

```typescript
// src/health/checks/PostgreSQLCheck.ts

import { Pool } from 'pg';
import { HealthChecker, CheckResult } from '../types';

export interface PostgreSQLCheckOptions {
  pool: Pool;
  name?: string;
  critical?: boolean;
  replicaLagThreshold?: number;   // seconds, alert ถ้า lag เกินนี้
  maxConnections?: number;         // alert ถ้า connections เกินนี้
}

export class PostgreSQLCheck implements HealthChecker {
  name: string;
  critical: boolean;
  private pool: Pool;
  private replicaLagThreshold: number;
  private maxConnections: number;

  constructor(options: PostgreSQLCheckOptions) {
    this.pool = options.pool;
    this.name = options.name ?? 'postgresql';
    this.critical = options.critical ?? true;
    this.replicaLagThreshold = options.replicaLagThreshold ?? 30;  // 30 seconds
    this.maxConnections = options.maxConnections ?? 90;              // 90% of max
  }

  async check(): Promise<CheckResult> {
    const client = await this.pool.connect();
    
    try {
      // Basic connectivity test
      const pingResult = await client.query('SELECT 1 as ping, NOW() as server_time');
      
      // Check replica lag (ถ้าเป็น primary)
      let replicaLag: number | null = null;
      let replicaInfo = '';
      
      try {
        const replicaQuery = await client.query(`
          SELECT 
            application_name,
            state,
            EXTRACT(EPOCH FROM (NOW() - pg_last_xact_replay_timestamp()))::INT as lag_seconds
          FROM pg_stat_replication
          LIMIT 5
        `);
        
        if (replicaQuery.rows.length > 0) {
          const maxLag = Math.max(...replicaQuery.rows.map((r: any) => r.lag_seconds || 0));
          replicaLag = maxLag;
          replicaInfo = `${replicaQuery.rows.length} replica(s), max lag: ${maxLag}s`;
        }
      } catch {
        // ถ้าเป็น replica จะ query pg_stat_replication ไม่ได้
        try {
          const isReplicaQuery = await client.query(`
            SELECT 
              pg_is_in_recovery() as is_replica,
              EXTRACT(EPOCH FROM (NOW() - pg_last_xact_replay_timestamp()))::INT as lag_seconds
          `);
          
          if (isReplicaQuery.rows[0]?.is_replica) {
            replicaLag = isReplicaQuery.rows[0]?.lag_seconds || 0;
            replicaInfo = `Replica lag: ${replicaLag}s`;
          }
        } catch {
          replicaInfo = 'Cannot determine replica status';
        }
      }

      // Check connection pool stats
      const poolStats = {
        total: this.pool.totalCount,
        idle: this.pool.idleCount,
        waiting: this.pool.waitingCount,
      };

      // Check active connections
      const connResult = await client.query(`
        SELECT 
          COUNT(*) as active,
          setting::int as max_conn
        FROM pg_stat_activity, pg_settings 
        WHERE name = 'max_connections'
        GROUP BY setting
      `);
      
      const activeConn = parseInt(connResult.rows[0]?.active || '0');
      const maxConn = parseInt(connResult.rows[0]?.max_conn || '100');
      const connUsagePercent = (activeConn / maxConn) * 100;

      // Determine status
      let status: 'healthy' | 'degraded' | 'unhealthy' = 'healthy';
      let message = 'PostgreSQL connection OK';
      
      if (replicaLag !== null && replicaLag > this.replicaLagThreshold) {
        status = 'degraded';
        message = `Replica lag too high: ${replicaLag}s`;
      }
      
      if (connUsagePercent > this.maxConnections) {
        status = 'degraded';
        message = `Connection usage high: ${connUsagePercent.toFixed(1)}%`;
      }

      return {
        status,
        responseTime: 0,  // จะถูก set โดย HealthCheckService
        message,
        lastChecked: new Date(),
        details: {
          serverTime: pingResult.rows[0]?.server_time,
          replicaInfo,
          replicaLag,
          connectionPool: poolStats,
          activeConnections: activeConn,
          maxConnections: maxConn,
          connectionUsage: `${connUsagePercent.toFixed(1)}%`,
        },
      };

    } finally {
      client.release();
    }
  }
}
```

### 4.2 Redis Health Check

```typescript
// src/health/checks/RedisCheck.ts

import Redis from 'ioredis';
import { HealthChecker, CheckResult } from '../types';

export interface RedisCheckOptions {
  client: Redis;
  name?: string;
  critical?: boolean;
  memoryThreshold?: number;   // percentage, alert ถ้า memory เกินนี้
}

export class RedisCheck implements HealthChecker {
  name: string;
  critical: boolean;
  private client: Redis;
  private memoryThreshold: number;

  constructor(options: RedisCheckOptions) {
    this.client = options.client;
    this.name = options.name ?? 'redis';
    this.critical = options.critical ?? true;
    this.memoryThreshold = options.memoryThreshold ?? 80;
  }

  async check(): Promise<CheckResult> {
    // PING test
    const pingResult = await this.client.ping();
    
    if (pingResult !== 'PONG') {
      return {
        status: 'unhealthy',
        responseTime: 0,
        message: `Redis PING failed: ${pingResult}`,
        lastChecked: new Date(),
      };
    }

    // Get Redis INFO
    const info = await this.client.info('all');
    const infoMap = this.parseRedisInfo(info);

    // Parse memory stats
    const usedMemory = parseInt(infoMap['used_memory'] || '0');
    const maxMemory = parseInt(infoMap['maxmemory'] || '0');
    let memoryUsagePercent = 0;
    
    if (maxMemory > 0) {
      memoryUsagePercent = (usedMemory / maxMemory) * 100;
    }

    // Parse connection stats
    const connectedClients = parseInt(infoMap['connected_clients'] || '0');
    const blockedClients = parseInt(infoMap['blocked_clients'] || '0');

    // Parse replication info
    const role = infoMap['role'] || 'unknown';
    const masterLinkStatus = infoMap['master_link_status'];
    const masterLastIoSeconds = parseInt(infoMap['master_last_io_seconds_ago'] || '0');

    // Parse keyspace stats
    const keyspaceInfo: Record<string, number> = {};
    for (const [key, value] of Object.entries(infoMap)) {
      if (key.startsWith('db')) {
        const match = value.match(/keys=(\d+)/);
        if (match) {
          keyspaceInfo[key] = parseInt(match[1]);
        }
      }
    }

    // Determine status
    let status: 'healthy' | 'degraded' | 'unhealthy' = 'healthy';
    let message = 'Redis PING OK';

    if (memoryUsagePercent > this.memoryThreshold) {
      status = 'degraded';
      message = `Redis memory usage high: ${memoryUsagePercent.toFixed(1)}%`;
    }

    if (role === 'slave' && masterLinkStatus === 'down') {
      status = 'degraded';
      message = 'Redis replica disconnected from master';
    }

    if (role === 'slave' && masterLinkStatus === 'up' && masterLastIoSeconds > 10) {
      status = 'degraded';
      message = `Redis replica lag: ${masterLastIoSeconds}s`;
    }

    return {
      status,
      responseTime: 0,
      message,
      lastChecked: new Date(),
      details: {
        role,
        redisVersion: infoMap['redis_version'],
        uptime: infoMap['uptime_in_seconds'],
        connectedClients,
        blockedClients,
        usedMemory: this.formatBytes(usedMemory),
        maxMemory: maxMemory > 0 ? this.formatBytes(maxMemory) : 'unlimited',
        memoryUsage: maxMemory > 0 ? `${memoryUsagePercent.toFixed(1)}%` : 'N/A',
        totalKeys: Object.values(keyspaceInfo).reduce((a, b) => a + b, 0),
        keyspaceInfo,
        replication: {
          role,
          masterLinkStatus,
          masterLastIoSeconds: role === 'slave' ? masterLastIoSeconds : undefined,
        },
      },
    };
  }

  private parseRedisInfo(info: string): Record<string, string> {
    const result: Record<string, string> = {};
    for (const line of info.split('\r\n')) {
      if (line && !line.startsWith('#')) {
        const [key, value] = line.split(':');
        if (key && value !== undefined) {
          result[key.trim()] = value.trim();
        }
      }
    }
    return result;
  }

  private formatBytes(bytes: number): string {
    if (bytes === 0) return '0 B';
    const k = 1024;
    const sizes = ['B', 'KB', 'MB', 'GB'];
    const i = Math.floor(Math.log(bytes) / Math.log(k));
    return `${parseFloat((bytes / Math.pow(k, i)).toFixed(2))} ${sizes[i]}`;
  }
}
```

### 4.3 MinIO Health Check

```typescript
// src/health/checks/MinIOCheck.ts

import { Client as MinIOClient } from 'minio';
import https from 'https';
import http from 'http';
import { HealthChecker, CheckResult } from '../types';

export interface MinIOCheckOptions {
  client: MinIOClient;
  endpoint: string;
  port?: number;
  useSSL?: boolean;
  name?: string;
  critical?: boolean;
  checkBuckets?: string[];    // optional: check specific buckets exist
}

export class MinIOCheck implements HealthChecker {
  name: string;
  critical: boolean;
  private client: MinIOClient;
  private endpoint: string;
  private port: number;
  private useSSL: boolean;
  private checkBuckets: string[];

  constructor(options: MinIOCheckOptions) {
    this.client = options.client;
    this.endpoint = options.endpoint;
    this.port = options.port ?? 9000;
    this.useSSL = options.useSSL ?? false;
    this.name = options.name ?? 'minio';
    this.critical = options.critical ?? false;   // MinIO ไม่ critical เท่า DB
    this.checkBuckets = options.checkBuckets ?? [];
  }

  async check(): Promise<CheckResult> {
    // Check MinIO health endpoint
    const healthUrl = `${this.useSSL ? 'https' : 'http'}://${this.endpoint}:${this.port}/minio/health/live`;
    
    await this.checkHttpEndpoint(healthUrl);

    // Check bucket existence ถ้ากำหนดไว้
    const bucketStatuses: Record<string, boolean> = {};
    
    if (this.checkBuckets.length > 0) {
      await Promise.all(
        this.checkBuckets.map(async (bucket) => {
          try {
            const exists = await this.client.bucketExists(bucket);
            bucketStatuses[bucket] = exists;
          } catch {
            bucketStatuses[bucket] = false;
          }
        })
      );
    }

    // ตรวจสอบว่า buckets ที่ต้องการมีอยู่ครบไหม
    const missingBuckets = Object.entries(bucketStatuses)
      .filter(([_, exists]) => !exists)
      .map(([name]) => name);

    let status: 'healthy' | 'degraded' | 'unhealthy' = 'healthy';
    let message = 'MinIO health check passed';

    if (missingBuckets.length > 0) {
      status = 'degraded';
      message = `Missing buckets: ${missingBuckets.join(', ')}`;
    }

    return {
      status,
      responseTime: 0,
      message,
      lastChecked: new Date(),
      details: {
        endpoint: `${this.endpoint}:${this.port}`,
        buckets: bucketStatuses,
        missingBuckets,
      },
    };
  }

  private checkHttpEndpoint(url: string): Promise<void> {
    return new Promise((resolve, reject) => {
      const module = url.startsWith('https') ? https : http;
      const req = module.get(url, (res) => {
        if (res.statusCode === 200) {
          resolve();
        } else {
          reject(new Error(`MinIO health endpoint returned ${res.statusCode}`));
        }
      });
      
      req.on('error', reject);
      req.setTimeout(3000, () => {
        req.destroy();
        reject(new Error('MinIO health check timeout'));
      });
    });
  }
}
```

### 4.4 Memory Health Check

```typescript
// src/health/checks/MemoryCheck.ts

import { HealthChecker, CheckResult } from '../types';

export interface MemoryCheckOptions {
  name?: string;
  critical?: boolean;
  heapUsedThreshold?: number;    // percentage
  heapTotalThreshold?: number;   // MB
  rssThreshold?: number;         // MB
}

export class MemoryCheck implements HealthChecker {
  name: string;
  critical: boolean;
  private heapUsedThreshold: number;
  private heapTotalThreshold: number;
  private rssThreshold: number;

  constructor(options: MemoryCheckOptions = {}) {
    this.name = options.name ?? 'memory';
    this.critical = options.critical ?? false;
    this.heapUsedThreshold = options.heapUsedThreshold ?? 85;  // 85%
    this.heapTotalThreshold = options.heapTotalThreshold ?? 512;  // 512 MB
    this.rssThreshold = options.rssThreshold ?? 1024;  // 1 GB
  }

  async check(): Promise<CheckResult> {
    const memUsage = process.memoryUsage();
    
    const heapUsedMB = Math.round(memUsage.heapUsed / 1024 / 1024);
    const heapTotalMB = Math.round(memUsage.heapTotal / 1024 / 1024);
    const rssMB = Math.round(memUsage.rss / 1024 / 1024);
    const externalMB = Math.round(memUsage.external / 1024 / 1024);
    
    const heapUsagePercent = (heapUsedMB / heapTotalMB) * 100;

    let status: 'healthy' | 'degraded' | 'unhealthy' = 'healthy';
    let message = 'Memory usage OK';

    if (heapUsagePercent > this.heapUsedThreshold) {
      status = 'degraded';
      message = `Heap usage high: ${heapUsagePercent.toFixed(1)}%`;
    }

    if (heapTotalMB > this.heapTotalThreshold) {
      status = 'degraded';
      message = `Heap size large: ${heapTotalMB} MB`;
    }

    if (rssMB > this.rssThreshold) {
      status = 'unhealthy';
      message = `RSS memory too high: ${rssMB} MB`;
    }

    return {
      status,
      responseTime: 0,
      message,
      lastChecked: new Date(),
      details: {
        heapUsed: `${heapUsedMB} MB`,
        heapTotal: `${heapTotalMB} MB`,
        heapUsage: `${heapUsagePercent.toFixed(1)}%`,
        rss: `${rssMB} MB`,
        external: `${externalMB} MB`,
        pid: process.pid,
        platform: process.platform,
        nodeVersion: process.version,
      },
    };
  }
}
```

---

## 5. Express Routes

```typescript
// src/health/routes/healthRoutes.ts

import { Router, Request, Response } from 'express';
import { HealthCheckService } from '../HealthCheckService';
import { HealthCheckResult } from '../types';

export function createHealthRoutes(healthService: HealthCheckService): Router {
  const router = Router();

  /**
   * GET /health
   * ข้อมูลสุขภาพโดยรวม (รวมทุก checks)
   */
  router.get('/', async (req: Request, res: Response) => {
    try {
      const result = await healthService.checkAll();
      const httpStatus = getHttpStatus(result);
      
      return res.status(httpStatus).json(result);
    } catch (error) {
      return res.status(503).json({
        status: 'unhealthy',
        timestamp: new Date(),
        error: 'Health check failed',
        message: error instanceof Error ? error.message : String(error),
      });
    }
  });

  /**
   * GET /health/live
   * Liveness probe - ใช้ใน Kubernetes livenessProbe
   * ตรวจสอบว่า Process ยังทำงานอยู่
   */
  router.get('/live', async (req: Request, res: Response) => {
    try {
      const result = await healthService.checkLiveness();
      const httpStatus = getHttpStatus(result);
      
      return res.status(httpStatus).json({
        status: result.status,
        timestamp: result.timestamp,
        uptime: result.uptime,
      });
    } catch (error) {
      return res.status(503).json({
        status: 'unhealthy',
        timestamp: new Date(),
        error: 'Liveness check failed',
      });
    }
  });

  /**
   * GET /health/ready
   * Readiness probe - ใช้ใน Kubernetes readinessProbe
   * ตรวจสอบว่า Application พร้อมรับ Traffic
   */
  router.get('/ready', async (req: Request, res: Response) => {
    try {
      const result = await healthService.checkAll();
      
      // Readiness: ready ถ้า status เป็น healthy หรือ degraded
      if (result.status === 'unhealthy') {
        return res.status(503).json({
          status: 'not_ready',
          timestamp: result.timestamp,
          checks: result.checks,
          message: 'Service is not ready to accept traffic',
        });
      }
      
      return res.status(200).json({
        status: 'ready',
        timestamp: result.timestamp,
        checks: result.checks,
      });
    } catch (error) {
      return res.status(503).json({
        status: 'not_ready',
        timestamp: new Date(),
        error: 'Readiness check failed',
      });
    }
  });

  /**
   * GET /health/startup
   * Startup probe - ใช้ใน Kubernetes startupProbe
   */
  router.get('/startup', async (req: Request, res: Response) => {
    try {
      // ตรวจสอบว่า startup สำเร็จหรือไม่
      // ในที่นี้ใช้ uptime เป็นตัวบ่งชี้ว่า startup เสร็จแล้ว
      const uptime = healthService.getUptime();
      
      if (uptime < 5) {  // ยังไม่ถึง 5 วินาที = ยังไม่พร้อม
        return res.status(503).json({
          status: 'starting',
          uptime,
          message: 'Application is still starting up',
        });
      }
      
      const result = await healthService.checkAll();
      
      if (result.status === 'unhealthy') {
        return res.status(503).json({
          status: 'startup_failed',
          timestamp: result.timestamp,
          checks: result.checks,
        });
      }
      
      return res.status(200).json({
        status: 'started',
        uptime,
        timestamp: result.timestamp,
      });
    } catch (error) {
      return res.status(503).json({
        status: 'startup_failed',
        error: error instanceof Error ? error.message : String(error),
      });
    }
  });

  /**
   * GET /health/details
   * ข้อมูลละเอียด (สำหรับ monitoring systems)
   */
  router.get('/details', async (req: Request, res: Response) => {
    try {
      const result = await healthService.checkAll();
      const httpStatus = getHttpStatus(result);
      
      return res.status(httpStatus).json({
        ...result,
        process: {
          pid: process.pid,
          nodeVersion: process.version,
          platform: process.platform,
          arch: process.arch,
          memoryUsage: process.memoryUsage(),
          cpuUsage: process.cpuUsage(),
        },
      });
    } catch (error) {
      return res.status(503).json({
        status: 'unhealthy',
        error: error instanceof Error ? error.message : String(error),
      });
    }
  });

  return router;
}

function getHttpStatus(result: HealthCheckResult): number {
  switch (result.status) {
    case 'healthy': return 200;
    case 'degraded': return 200;   // degraded ยังใช้งานได้
    case 'unhealthy': return 503;
    default: return 503;
  }
}
```

---

## 6. Health Check Middleware

```typescript
// src/middleware/healthMiddleware.ts

import { Request, Response, NextFunction } from 'express';

interface SlowRequestOptions {
  threshold: number;   // ms
  logFn?: (message: string) => void;
}

/**
 * Middleware วัดเวลา request
 */
export function requestTimingMiddleware(options: SlowRequestOptions = { threshold: 1000 }) {
  const { threshold, logFn = console.warn } = options;
  
  return (req: Request, res: Response, next: NextFunction): void => {
    const startTime = Date.now();
    
    // Override res.end เพื่อ capture response time
    const originalEnd = res.end.bind(res);
    
    (res as any).end = function(...args: any[]) {
      const duration = Date.now() - startTime;
      
      if (duration > threshold) {
        logFn(`Slow request: ${req.method} ${req.path} took ${duration}ms`);
      }
      
      res.setHeader('X-Response-Time', `${duration}ms`);
      return originalEnd(...args);
    };
    
    next();
  };
}

/**
 * Health check bypass middleware
 * ข้าม authentication สำหรับ health endpoints
 */
export function healthCheckBypassMiddleware(healthPaths: string[]) {
  return (req: Request, res: Response, next: NextFunction): void => {
    const isHealthPath = healthPaths.some(path => req.path.startsWith(path));
    
    if (isHealthPath) {
      // Skip authentication
      (req as any).skipAuth = true;
    }
    
    next();
  };
}
```

---

## 7. Prometheus Metrics Endpoint

```typescript
// src/metrics/MetricsService.ts

import { Registry, Counter, Histogram, Gauge, collectDefaultMetrics } from 'prom-client';

export class MetricsService {
  private registry: Registry;
  
  // HTTP Metrics
  httpRequestsTotal: Counter;
  httpRequestDuration: Histogram;
  
  // Database Metrics
  dbConnectionsActive: Gauge;
  dbQueryDuration: Histogram;
  dbErrors: Counter;
  
  // Redis Metrics
  redisOperationsTotal: Counter;
  redisOperationDuration: Histogram;
  redisErrors: Counter;
  
  // Application Metrics
  activeUsers: Gauge;
  jobsProcessed: Counter;
  jobsFailed: Counter;

  constructor() {
    this.registry = new Registry();
    
    // เก็บ default Node.js metrics
    collectDefaultMetrics({ register: this.registry });

    // HTTP Metrics
    this.httpRequestsTotal = new Counter({
      name: 'http_requests_total',
      help: 'Total number of HTTP requests',
      labelNames: ['method', 'route', 'status_code'],
      registers: [this.registry],
    });

    this.httpRequestDuration = new Histogram({
      name: 'http_request_duration_seconds',
      help: 'HTTP request duration in seconds',
      labelNames: ['method', 'route', 'status_code'],
      buckets: [0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1, 2, 5],
      registers: [this.registry],
    });

    // Database Metrics
    this.dbConnectionsActive = new Gauge({
      name: 'db_connections_active',
      help: 'Number of active database connections',
      registers: [this.registry],
    });

    this.dbQueryDuration = new Histogram({
      name: 'db_query_duration_seconds',
      help: 'Database query duration in seconds',
      labelNames: ['operation', 'table'],
      buckets: [0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1, 5],
      registers: [this.registry],
    });

    this.dbErrors = new Counter({
      name: 'db_errors_total',
      help: 'Total number of database errors',
      labelNames: ['operation', 'error_type'],
      registers: [this.registry],
    });

    // Redis Metrics
    this.redisOperationsTotal = new Counter({
      name: 'redis_operations_total',
      help: 'Total number of Redis operations',
      labelNames: ['operation', 'status'],
      registers: [this.registry],
    });

    this.redisOperationDuration = new Histogram({
      name: 'redis_operation_duration_seconds',
      help: 'Redis operation duration in seconds',
      labelNames: ['operation'],
      buckets: [0.0001, 0.0005, 0.001, 0.005, 0.01, 0.05, 0.1],
      registers: [this.registry],
    });

    this.redisErrors = new Counter({
      name: 'redis_errors_total',
      help: 'Total number of Redis errors',
      labelNames: ['operation'],
      registers: [this.registry],
    });

    // Application Metrics
    this.activeUsers = new Gauge({
      name: 'active_users',
      help: 'Number of active users',
      registers: [this.registry],
    });

    this.jobsProcessed = new Counter({
      name: 'jobs_processed_total',
      help: 'Total number of jobs processed',
      labelNames: ['queue', 'status'],
      registers: [this.registry],
    });

    this.jobsFailed = new Counter({
      name: 'jobs_failed_total',
      help: 'Total number of failed jobs',
      labelNames: ['queue', 'error_type'],
      registers: [this.registry],
    });
  }

  /**
   * Get metrics in Prometheus format
   */
  async getMetrics(): Promise<string> {
    return this.registry.metrics();
  }

  /**
   * Get content type for Prometheus
   */
  getContentType(): string {
    return this.registry.contentType;
  }

  /**
   * Express middleware สำหรับ track HTTP metrics
   */
  httpMetricsMiddleware() {
    return (req: any, res: any, next: any) => {
      const startTime = Date.now();
      
      res.on('finish', () => {
        const duration = (Date.now() - startTime) / 1000;
        const route = req.route?.path || req.path;
        
        this.httpRequestsTotal.inc({
          method: req.method,
          route,
          status_code: res.statusCode,
        });
        
        this.httpRequestDuration.observe(
          { method: req.method, route, status_code: res.statusCode },
          duration
        );
      });
      
      next();
    };
  }
}
```

---

## 8. Graceful Startup และ Shutdown

### 8.1 Graceful Startup

```typescript
// src/startup/GracefulStartup.ts

import { HealthCheckService } from '../health/HealthCheckService';

export interface StartupConfig {
  maxRetries: number;
  retryDelay: number;    // milliseconds
  timeout: number;       // milliseconds
}

export class GracefulStartup {
  private healthService: HealthCheckService;
  private config: StartupConfig;

  constructor(healthService: HealthCheckService, config: Partial<StartupConfig> = {}) {
    this.healthService = healthService;
    this.config = {
      maxRetries: config.maxRetries ?? 30,
      retryDelay: config.retryDelay ?? 2000,   // 2 seconds
      timeout: config.timeout ?? 60000,         // 60 seconds
    };
  }

  /**
   * รอจนกว่า dependencies จะพร้อม
   */
  async waitForDependencies(): Promise<void> {
    console.log('⏳ Waiting for dependencies to be ready...');
    
    const startTime = Date.now();
    let attempt = 0;
    
    while (attempt < this.config.maxRetries) {
      if (Date.now() - startTime > this.config.timeout) {
        throw new Error(`Startup timeout after ${this.config.timeout}ms`);
      }
      
      attempt++;
      console.log(`🔍 Health check attempt ${attempt}/${this.config.maxRetries}...`);
      
      try {
        const result = await this.healthService.checkAll();
        
        if (result.status !== 'unhealthy') {
          console.log(`✅ All dependencies ready after ${attempt} attempts`);
          return;
        }
        
        // แสดงว่า check ไหน fail
        const failedChecks = Object.entries(result.checks)
          .filter(([_, check]) => check.status === 'unhealthy')
          .map(([name, check]) => `${name}: ${check.message}`)
          .join(', ');
        
        console.log(`❌ Dependencies not ready: ${failedChecks}`);
      } catch (error) {
        console.log(`❌ Health check error: ${error}`);
      }
      
      await this.sleep(this.config.retryDelay);
    }
    
    throw new Error(`Dependencies not ready after ${this.config.maxRetries} attempts`);
  }

  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}
```

### 8.2 Graceful Shutdown

```typescript
// src/shutdown/GracefulShutdown.ts

import { Server } from 'http';
import { Pool } from 'pg';
import Redis from 'ioredis';

export interface ShutdownHandler {
  name: string;
  timeout: number;    // milliseconds
  handler: () => Promise<void>;
}

export class GracefulShutdown {
  private server: Server;
  private handlers: ShutdownHandler[] = [];
  private isShuttingDown = false;
  private shutdownTimeout: number;

  constructor(server: Server, shutdownTimeout: number = 30000) {
    this.server = server;
    this.shutdownTimeout = shutdownTimeout;
    
    // Register signal handlers
    process.on('SIGTERM', () => this.shutdown('SIGTERM'));
    process.on('SIGINT', () => this.shutdown('SIGINT'));
    
    // Handle uncaught exceptions
    process.on('uncaughtException', (error) => {
      console.error('Uncaught exception:', error);
      this.shutdown('uncaughtException').catch(console.error);
    });
    
    process.on('unhandledRejection', (reason) => {
      console.error('Unhandled rejection:', reason);
    });
  }

  /**
   * Register cleanup handler
   */
  register(handler: ShutdownHandler): this {
    this.handlers.push(handler);
    return this;
  }

  /**
   * Register PostgreSQL connection pool
   */
  registerPostgreSQL(pool: Pool, name: string = 'postgresql'): this {
    return this.register({
      name,
      timeout: 10000,
      handler: async () => {
        console.log(`🔒 Closing ${name} connection pool...`);
        await pool.end();
        console.log(`✅ ${name} connection pool closed`);
      },
    });
  }

  /**
   * Register Redis connection
   */
  registerRedis(client: Redis, name: string = 'redis'): this {
    return this.register({
      name,
      timeout: 5000,
      handler: async () => {
        console.log(`🔒 Closing ${name} connection...`);
        await client.quit();
        console.log(`✅ ${name} connection closed`);
      },
    });
  }

  /**
   * ดำเนิน Graceful Shutdown
   */
  async shutdown(signal: string): Promise<void> {
    if (this.isShuttingDown) {
      console.log('⚠️  Already shutting down, ignoring signal:', signal);
      return;
    }
    
    this.isShuttingDown = true;
    console.log(`\n🛑 Received ${signal}. Starting graceful shutdown...`);

    // Set overall timeout
    const shutdownTimer = setTimeout(() => {
      console.error('❌ Shutdown timeout, forcing exit');
      process.exit(1);
    }, this.shutdownTimeout);

    try {
      // 1. Stop accepting new connections
      console.log('📴 Stopping HTTP server (no new connections)...');
      await this.stopHttpServer();

      // 2. Run cleanup handlers in order
      for (const handler of this.handlers) {
        try {
          console.log(`🔄 Running cleanup: ${handler.name}...`);
          const timeout = new Promise<void>((_, reject) => {
            setTimeout(() => reject(new Error(`Timeout: ${handler.name}`)), handler.timeout);
          });
          
          await Promise.race([handler.handler(), timeout]);
          console.log(`✅ Cleanup done: ${handler.name}`);
        } catch (error) {
          console.error(`❌ Cleanup failed: ${handler.name}:`, error);
        }
      }

      clearTimeout(shutdownTimer);
      console.log('✅ Graceful shutdown complete');
      process.exit(0);
    } catch (error) {
      clearTimeout(shutdownTimer);
      console.error('❌ Graceful shutdown error:', error);
      process.exit(1);
    }
  }

  private stopHttpServer(): Promise<void> {
    return new Promise((resolve, reject) => {
      this.server.close((err) => {
        if (err) reject(err);
        else resolve();
      });
    });
  }
}
```

---

## 9. Docker HEALTHCHECK

```dockerfile
# Dockerfile

FROM node:20-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY dist/ ./dist/

# สร้าง healthcheck script
RUN echo '#!/bin/sh' > /healthcheck.sh && \
    echo 'wget -qO- http://localhost:3000/health/live || exit 1' >> /healthcheck.sh && \
    chmod +x /healthcheck.sh

# Docker HEALTHCHECK instruction
# --interval: ตรวจทุกกี่วินาที
# --timeout: รอได้กี่วินาทีต่อครั้ง
# --start-period: รอให้ application start ก่อน (Startup grace period)
# --retries: fail กี่ครั้งถึงจะ unhealthy
HEALTHCHECK --interval=30s \
            --timeout=10s \
            --start-period=60s \
            --retries=3 \
            CMD /healthcheck.sh

EXPOSE 3000
CMD ["node", "dist/index.js"]
```

**Alternative curl-based healthcheck:**
```dockerfile
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
  CMD curl -f http://localhost:3000/health/live || exit 1
```

---

## 10. Kubernetes Probes

```yaml
# kubernetes/deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-server
  template:
    metadata:
      labels:
        app: api-server
    spec:
      containers:
      - name: api-server
        image: myapp:latest
        ports:
        - containerPort: 3000
          name: http
        
        env:
        - name: NODE_ENV
          value: production
        - name: PORT
          value: "3000"
        
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        
        # Startup Probe: ให้เวลา startup นานพอ
        # initialDelaySeconds * failureThreshold = max startup time
        # 10s * 30 = 300s = 5 minutes maximum
        startupProbe:
          httpGet:
            path: /health/startup
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 30    # Allow 5 minutes to start
        
        # Liveness Probe: restart ถ้า process ตาย
        livenessProbe:
          httpGet:
            path: /health/live
            port: 3000
          initialDelaySeconds: 0   # Startup probe จัดการ initial delay แล้ว
          periodSeconds: 30         # ตรวจทุก 30 วินาที
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 3      # fail 3 ครั้งติดต่อกัน = restart
        
        # Readiness Probe: หยุดรับ traffic ถ้า dependencies มีปัญหา
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 3000
          initialDelaySeconds: 0
          periodSeconds: 10         # ตรวจทุก 10 วินาที
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 3      # fail 3 ครั้ง = remove from load balancer
        
        # Volume mounts
        volumeMounts:
        - name: config
          mountPath: /app/config
          readOnly: true
      
      volumes:
      - name: config
        configMap:
          name: api-config
      
      # Graceful termination
      terminationGracePeriodSeconds: 60  # ให้เวลา graceful shutdown 60 วินาที
```

---

## 11. Full Integration Example

```typescript
// src/index.ts - Complete integration

import express from 'express';
import { Pool } from 'pg';
import Redis from 'ioredis';
import { Client as MinIOClient } from 'minio';
import { createServer } from 'http';

import { HealthCheckService } from './health/HealthCheckService';
import { PostgreSQLCheck } from './health/checks/PostgreSQLCheck';
import { RedisCheck } from './health/checks/RedisCheck';
import { MinIOCheck } from './health/checks/MinIOCheck';
import { MemoryCheck } from './health/checks/MemoryCheck';
import { createHealthRoutes } from './health/routes/healthRoutes';
import { MetricsService } from './metrics/MetricsService';
import { GracefulStartup } from './startup/GracefulStartup';
import { GracefulShutdown } from './shutdown/GracefulShutdown';

async function main() {
  const app = express();
  
  // ========== Database Connections ==========
  const pgPool = new Pool({
    host: process.env.DB_HOST ?? 'localhost',
    port: parseInt(process.env.DB_PORT ?? '5432'),
    database: process.env.DB_NAME ?? 'myapp',
    user: process.env.DB_USER ?? 'postgres',
    password: process.env.DB_PASSWORD,
    max: 20,
    idleTimeoutMillis: 30000,
    connectionTimeoutMillis: 5000,
  });

  const redisClient = new Redis({
    host: process.env.REDIS_HOST ?? 'localhost',
    port: parseInt(process.env.REDIS_PORT ?? '6379'),
    password: process.env.REDIS_PASSWORD,
    lazyConnect: true,
    maxRetriesPerRequest: 3,
  });

  const minioClient = new MinIOClient({
    endPoint: process.env.MINIO_ENDPOINT ?? 'localhost',
    port: parseInt(process.env.MINIO_PORT ?? '9000'),
    useSSL: process.env.MINIO_USE_SSL === 'true',
    accessKey: process.env.MINIO_ACCESS_KEY ?? 'minioadmin',
    secretKey: process.env.MINIO_SECRET_KEY ?? 'minioadmin',
  });

  // ========== Services ==========
  const metricsService = new MetricsService();

  const healthService = new HealthCheckService({
    cacheTTL: 5000,
    timeout: 3000,
  });

  // Register health checkers
  healthService
    .register(new PostgreSQLCheck({
      pool: pgPool,
      critical: true,
      replicaLagThreshold: 30,
    }))
    .register(new RedisCheck({
      client: redisClient,
      critical: true,
      memoryThreshold: 80,
    }))
    .register(new MinIOCheck({
      client: minioClient,
      endpoint: process.env.MINIO_ENDPOINT ?? 'localhost',
      port: parseInt(process.env.MINIO_PORT ?? '9000'),
      critical: false,
      checkBuckets: ['uploads', 'thumbnails'],
    }))
    .register(new MemoryCheck({
      critical: false,
      heapUsedThreshold: 85,
      rssThreshold: 1024,
    }));

  // ========== Middleware ==========
  app.use(express.json());
  app.use(express.urlencoded({ extended: true }));
  app.use(metricsService.httpMetricsMiddleware());

  // ========== Routes ==========
  // Health routes - ไม่ต้องการ authentication
  app.use('/health', createHealthRoutes(healthService));

  // Metrics endpoint (สำหรับ Prometheus scraping)
  app.get('/metrics', async (req, res) => {
    try {
      const metrics = await metricsService.getMetrics();
      res.set('Content-Type', metricsService.getContentType());
      res.send(metrics);
    } catch (error) {
      res.status(500).json({ error: 'Failed to get metrics' });
    }
  });

  // Main API routes (ต้องการ authentication)
  app.get('/api/v1/users', async (req, res) => {
    res.json({ users: [] });
  });

  // ========== Start Server ==========
  const server = createServer(app);
  const port = parseInt(process.env.PORT ?? '3000');

  // Graceful Startup: รอ dependencies
  const startupManager = new GracefulStartup(healthService, {
    maxRetries: 30,
    retryDelay: 2000,
    timeout: 60000,
  });

  // Graceful Shutdown
  const shutdownManager = new GracefulShutdown(server, 30000);
  shutdownManager
    .registerPostgreSQL(pgPool)
    .registerRedis(redisClient)
    .register({
      name: 'minio-check',
      timeout: 5000,
      handler: async () => {
        // MinIO client ไม่มี close method ชัดเจน
        console.log('MinIO connections will close automatically');
      },
    });

  try {
    // Connect to Redis
    await redisClient.connect();
    console.log('✅ Redis connected');

    // รอ dependencies พร้อม
    await startupManager.waitForDependencies();

    // Start HTTP server
    await new Promise<void>((resolve) => {
      server.listen(port, () => {
        console.log(`✅ Server listening on port ${port}`);
        resolve();
      });
    });

    console.log(`
🚀 Application started successfully!
   Port:        ${port}
   Environment: ${process.env.NODE_ENV ?? 'development'}
   Health:      http://localhost:${port}/health
   Metrics:     http://localhost:${port}/metrics
   Ready:       http://localhost:${port}/health/ready
   Live:        http://localhost:${port}/health/live
    `);

  } catch (error) {
    console.error('❌ Startup failed:', error);
    process.exit(1);
  }
}

main().catch((error) => {
  console.error('Fatal error:', error);
  process.exit(1);
});
```

---

## 12. Health Check Dashboard

เราสามารถสร้าง Dashboard ง่ายๆ สำหรับแสดง Health Status:

```typescript
// src/health/routes/dashboardRoute.ts

import { Router, Request, Response } from 'express';
import { HealthCheckService } from '../HealthCheckService';

export function createDashboardRoute(healthService: HealthCheckService): Router {
  const router = Router();

  router.get('/dashboard', async (req: Request, res: Response) => {
    const result = await healthService.checkAll();
    
    const html = `
<!DOCTYPE html>
<html>
<head>
  <title>Health Dashboard</title>
  <meta http-equiv="refresh" content="30">
  <style>
    body { font-family: monospace; background: #1a1a1a; color: #fff; padding: 20px; }
    .status-healthy { color: #00ff00; }
    .status-degraded { color: #ffaa00; }
    .status-unhealthy { color: #ff0000; }
    .check-card { border: 1px solid #333; padding: 15px; margin: 10px 0; border-radius: 5px; }
    h1 { color: #fff; }
    .timestamp { color: #888; font-size: 0.9em; }
  </style>
</head>
<body>
  <h1>Health Dashboard</h1>
  <p class="timestamp">Last updated: ${result.timestamp.toISOString()}</p>
  <p>Overall Status: <span class="status-${result.status}">${result.status.toUpperCase()}</span></p>
  <p>Uptime: ${result.uptime}s | Version: ${result.version} | Env: ${result.environment}</p>
  
  <h2>Checks</h2>
  ${Object.entries(result.checks).map(([name, check]) => `
    <div class="check-card">
      <h3>${name}: <span class="status-${check.status}">${check.status.toUpperCase()}</span></h3>
      <p>Message: ${check.message}</p>
      <p>Response Time: ${check.responseTime}ms</p>
      <p>Last Checked: ${check.lastChecked.toISOString()}</p>
      ${check.error ? `<p style="color:#ff0000">Error: ${check.error}</p>` : ''}
      ${check.details ? `<pre>${JSON.stringify(check.details, null, 2)}</pre>` : ''}
    </div>
  `).join('')}
</body>
</html>
    `;
    
    res.setHeader('Content-Type', 'text/html');
    res.send(html);
  });

  return router;
}
```

---

## 13. Testing Health Checks

```typescript
// src/health/__tests__/HealthCheckService.test.ts

import { HealthCheckService } from '../HealthCheckService';
import { HealthChecker, CheckResult } from '../types';

describe('HealthCheckService', () => {
  let service: HealthCheckService;

  beforeEach(() => {
    service = new HealthCheckService({ cacheTTL: 0 }); // ปิด cache สำหรับ testing
  });

  it('should return healthy when all checks pass', async () => {
    const mockChecker: HealthChecker = {
      name: 'test',
      critical: true,
      check: async (): Promise<CheckResult> => ({
        status: 'healthy',
        responseTime: 5,
        message: 'OK',
        lastChecked: new Date(),
      }),
    };

    service.register(mockChecker);
    const result = await service.checkAll();

    expect(result.status).toBe('healthy');
    expect(result.checks.test.status).toBe('healthy');
  });

  it('should return unhealthy when critical check fails', async () => {
    const mockChecker: HealthChecker = {
      name: 'database',
      critical: true,
      check: async (): Promise<CheckResult> => ({
        status: 'unhealthy',
        responseTime: 100,
        message: 'Connection failed',
        lastChecked: new Date(),
      }),
    };

    service.register(mockChecker);
    const result = await service.checkAll();

    expect(result.status).toBe('unhealthy');
  });

  it('should return degraded when non-critical check fails', async () => {
    const mockChecker: HealthChecker = {
      name: 'storage',
      critical: false,
      check: async (): Promise<CheckResult> => ({
        status: 'unhealthy',
        responseTime: 100,
        message: 'Storage unavailable',
        lastChecked: new Date(),
      }),
    };

    service.register(mockChecker);
    const result = await service.checkAll();

    expect(result.status).toBe('degraded');
  });

  it('should cache results', async () => {
    let callCount = 0;
    const cachedService = new HealthCheckService({ cacheTTL: 10000 });
    
    const mockChecker: HealthChecker = {
      name: 'test',
      critical: false,
      check: async (): Promise<CheckResult> => {
        callCount++;
        return {
          status: 'healthy',
          responseTime: 5,
          message: 'OK',
          lastChecked: new Date(),
        };
      },
    };

    cachedService.register(mockChecker);
    await cachedService.checkAll();
    await cachedService.checkAll();

    expect(callCount).toBe(1);  // เรียกแค่ครั้งเดียวเพราะ cache
  });

  it('should handle check timeout', async () => {
    const slowService = new HealthCheckService({ timeout: 100 });
    
    const slowChecker: HealthChecker = {
      name: 'slow',
      critical: false,
      check: async (): Promise<CheckResult> => {
        await new Promise(resolve => setTimeout(resolve, 5000));  // 5 seconds
        return {
          status: 'healthy',
          responseTime: 5000,
          message: 'Very slow',
          lastChecked: new Date(),
        };
      },
    };

    slowService.register(slowChecker);
    const result = await slowService.checkAll();

    expect(result.checks.slow.status).toBe('unhealthy');
    expect(result.checks.slow.error).toContain('timeout');
  });
});
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **ประเภท Health Checks**: Liveness (restart ถ้า fail), Readiness (หยุด traffic ถ้า fail), Startup (ให้เวลา startup)

2. **Health Check Design**: 
   - `/health` - overall status
   - `/health/live` - liveness
   - `/health/ready` - readiness
   - `/metrics` - Prometheus metrics

3. **Implementation**:
   - `HealthCheckService` - orchestrate all checks
   - Individual checkers: PostgreSQL, Redis, MinIO, Memory
   - Caching ผลลัพธ์เพื่อไม่ให้ overload dependencies

4. **Graceful Lifecycle**:
   - Startup: รอ dependencies พร้อมก่อน accept traffic
   - Shutdown: drain requests แล้วปิด connections

5. **Container Integration**:
   - Docker HEALTHCHECK instruction
   - Kubernetes livenessProbe, readinessProbe, startupProbe

Health Checks เป็นส่วนสำคัญมากของ Production System ทำให้ Orchestration Platform รู้ว่าต้องทำอะไรกับ Service ของเรา ทำให้ระบบมี Self-healing capability

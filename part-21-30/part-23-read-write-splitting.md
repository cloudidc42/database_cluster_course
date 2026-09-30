# Part 23: Read/Write Splitting ใน Application

## บทนำ

Read/Write Splitting คือการแยก Query ประเภท READ (SELECT) ออกจาก WRITE (INSERT/UPDATE/DELETE) เพื่อกระจาย Database Load ไปยังเซิร์ฟเวอร์หลายตัว

```
Application
     │
     ├──── WRITE ──────▶ Primary DB  (100% Writes)
     │
     └──── READ ────────▶ Replica 1  (33% Reads)
                    │───▶ Replica 2  (33% Reads)
                    └───▶ Replica 3  (34% Reads)
```

**เมื่อไหร่ต้องใช้ Read/Write Splitting:**
- Read Traffic มากกว่า Write Traffic อย่างชัดเจน (80/20 rule)
- Primary มี CPU/IO สูงจาก SELECT Queries
- ต้องการ Horizontal Scaling สำหรับ Read
- มี Replica Servers ที่ว่างอยู่แล้ว

---

## 1. ทำไมต้องทำ Read/Write Splitting

### 1.1 ปัญหาของ Single Database

```
ระบบ E-commerce:
- 1 Product Page View = 10 SELECT queries
- 1 Purchase = 5 INSERT/UPDATE queries
- Traffic: 1000 Page Views/sec + 50 Purchases/sec

Load on Single DB:
- SELECTs: 10,000 queries/sec
- INSERTs: 250 queries/sec
- Total: 10,250 queries/sec  ← ระเบิด!
```

### 1.2 หลังแยก Read/Write

```
Primary DB:
- INSERTs: 250 queries/sec  ← เบาลงมาก

Replica 1 + 2 + 3:
- SELECTs: 10,000/3 ≈ 3,333 queries/sec ต่อตัว
```

---

## 2. Connection Manager สำหรับ Multiple Databases

### 2.1 โครงสร้าง DatabaseManager

```typescript
// src/database/DatabaseManager.ts

import { Pool, PoolClient, QueryResult, QueryResultRow } from 'pg';

interface DatabaseConfig {
  primary: PoolConfig;
  replicas: PoolConfig[];
  options?: {
    maxReplicaLag?: number;       // ยอมรับ Lag สูงสุดกี่ milliseconds
    healthCheckInterval?: number;  // ตรวจสอบ Health ทุกกี่ ms
    readAfterWriteTimeout?: number; // Sticky Session timeout
  };
}

interface PoolConfig {
  host: string;
  port: number;
  database: string;
  user: string;
  password: string;
  max: number;         // Max connections in pool
  idleTimeoutMillis?: number;
  connectionTimeoutMillis?: number;
}

interface ReplicaInfo {
  pool: Pool;
  host: string;
  isHealthy: boolean;
  lag: number;         // milliseconds
  lastChecked: number; // timestamp
}
```

---

## 3. DatabaseManager Class: Full Implementation

```typescript
// src/database/DatabaseManager.ts
import { Pool, PoolClient, QueryResult, QueryResultRow } from 'pg';
import { EventEmitter } from 'events';

interface PoolConfig {
  host: string;
  port?: number;
  database: string;
  user: string;
  password: string;
  max?: number;
  idleTimeoutMillis?: number;
  connectionTimeoutMillis?: number;
}

interface DatabaseConfig {
  primary: PoolConfig;
  replicas?: PoolConfig[];
  options?: {
    maxReplicaLagMs?: number;
    healthCheckIntervalMs?: number;
    readAfterWriteTimeoutMs?: number;
    enableRoundRobin?: boolean;
  };
}

interface ReplicaInfo {
  pool: Pool;
  host: string;
  port: number;
  isHealthy: boolean;
  lagMs: number;
  lastCheckedAt: number;
  connectionCount: number;
}

type TransactionCallback<T> = (client: PoolClient) => Promise<T>;

class DatabaseManager extends EventEmitter {
  private primaryPool: Pool;
  private replicas: ReplicaInfo[] = [];
  private replicaIndex = 0;
  private healthCheckTimer?: NodeJS.Timeout;
  private readonly options: Required<NonNullable<DatabaseConfig['options']>>;

  // Read-After-Write Consistency: เก็บ timestamp ของ last write ต่อ Request/User
  private recentWrites: Map<string, number> = new Map();

  constructor(config: DatabaseConfig) {
    super();

    this.options = {
      maxReplicaLagMs: config.options?.maxReplicaLagMs ?? 5000,
      healthCheckIntervalMs: config.options?.healthCheckIntervalMs ?? 10000,
      readAfterWriteTimeoutMs: config.options?.readAfterWriteTimeoutMs ?? 2000,
      enableRoundRobin: config.options?.enableRoundRobin ?? true,
    };

    // สร้าง Primary Pool
    this.primaryPool = new Pool({
      ...config.primary,
      max: config.primary.max ?? 10,
    });

    this.primaryPool.on('error', (err) => {
      console.error('Primary pool error:', err);
      this.emit('primaryError', err);
    });

    this.primaryPool.on('connect', () => {
      this.emit('primaryConnect');
    });

    // สร้าง Replica Pools
    if (config.replicas && config.replicas.length > 0) {
      for (const replicaConfig of config.replicas) {
        const pool = new Pool({
          ...replicaConfig,
          max: replicaConfig.max ?? 20,
        });

        pool.on('error', (err) => {
          console.error(`Replica ${replicaConfig.host} pool error:`, err);
          const replica = this.replicas.find(r => r.host === replicaConfig.host);
          if (replica) {
            replica.isHealthy = false;
            this.emit('replicaError', { host: replicaConfig.host, error: err });
          }
        });

        this.replicas.push({
          pool,
          host: replicaConfig.host,
          port: replicaConfig.port ?? 5432,
          isHealthy: true,
          lagMs: 0,
          lastCheckedAt: Date.now(),
          connectionCount: 0,
        });
      }
    }

    // เริ่ม Health Check
    this.startHealthCheck();
  }

  // ==========================================
  // WRITE Operations - ใช้ Primary เสมอ
  // ==========================================

  async query<T extends QueryResultRow = QueryResultRow>(
    sql: string,
    params?: unknown[],
    writeContext?: string  // Session/User ID สำหรับ Read-After-Write
  ): Promise<QueryResult<T>> {
    const isWrite = this.isWriteQuery(sql);

    if (isWrite) {
      // บันทึก Write Timestamp สำหรับ Read-After-Write Consistency
      if (writeContext) {
        this.recentWrites.set(writeContext, Date.now());
        // ลบหลัง timeout
        setTimeout(() => {
          this.recentWrites.delete(writeContext);
        }, this.options.readAfterWriteTimeoutMs);
      }

      return this.primaryPool.query<T>(sql, params);
    }

    // READ: ใช้ Replica (ถ้ามี)
    return this.queryReplica<T>(sql, params, writeContext);
  }

  private async queryReplica<T extends QueryResultRow = QueryResultRow>(
    sql: string,
    params?: unknown[],
    readContext?: string
  ): Promise<QueryResult<T>> {
    // Read-After-Write Consistency Check
    if (readContext && this.hasRecentWrite(readContext)) {
      // ใช้ Primary สำหรับ Read เพื่อป้องกัน Stale Data
      return this.primaryPool.query<T>(sql, params);
    }

    const replica = this.getHealthyReplica();
    if (!replica) {
      // Fallback ไป Primary ถ้าไม่มี Replica ที่ Healthy
      console.warn('No healthy replicas available, falling back to primary for read');
      return this.primaryPool.query<T>(sql, params);
    }

    try {
      replica.connectionCount++;
      const result = await replica.pool.query<T>(sql, params);
      replica.connectionCount--;
      return result;
    } catch (error) {
      replica.connectionCount--;
      replica.isHealthy = false;
      // Retry บน Primary
      return this.primaryPool.query<T>(sql, params);
    }
  }

  // ==========================================
  // TRANSACTION - ใช้ Primary เสมอ
  // ==========================================

  async withTransaction<T>(
    callback: TransactionCallback<T>,
    writeContext?: string
  ): Promise<T> {
    const client = await this.primaryPool.connect();

    // บันทึก Write Context
    if (writeContext) {
      this.recentWrites.set(writeContext, Date.now());
      setTimeout(() => {
        this.recentWrites.delete(writeContext);
      }, this.options.readAfterWriteTimeoutMs);
    }

    try {
      await client.query('BEGIN');
      const result = await callback(client);
      await client.query('COMMIT');
      return result;
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }

  // ==========================================
  // ROUND-ROBIN LOAD BALANCING
  // ==========================================

  private getHealthyReplica(): ReplicaInfo | null {
    const healthy = this.replicas.filter(
      r => r.isHealthy && r.lagMs <= this.options.maxReplicaLagMs
    );

    if (healthy.length === 0) return null;

    if (this.options.enableRoundRobin) {
      // Round-Robin
      const replica = healthy[this.replicaIndex % healthy.length];
      this.replicaIndex = (this.replicaIndex + 1) % healthy.length;
      return replica;
    } else {
      // Least Connections
      return healthy.reduce((min, r) =>
        r.connectionCount < min.connectionCount ? r : min
      );
    }
  }

  // ==========================================
  // READ-AFTER-WRITE CONSISTENCY
  // ==========================================

  private hasRecentWrite(context: string): boolean {
    const lastWrite = this.recentWrites.get(context);
    if (!lastWrite) return false;
    return Date.now() - lastWrite < this.options.readAfterWriteTimeoutMs;
  }

  // Mark ว่ามี Write เพิ่งเกิดขึ้น (สำหรับกรณีที่ query ไม่ผ่าน method นี้)
  markWrite(context: string): void {
    this.recentWrites.set(context, Date.now());
    setTimeout(() => {
      this.recentWrites.delete(context);
    }, this.options.readAfterWriteTimeoutMs);
  }

  // ==========================================
  // HEALTH CHECK
  // ==========================================

  private startHealthCheck(): void {
    this.healthCheckTimer = setInterval(async () => {
      await this.checkReplicaHealth();
    }, this.options.healthCheckIntervalMs);

    // ตรวจสอบทันทีครั้งแรก
    this.checkReplicaHealth().catch(console.error);
  }

  private async checkReplicaHealth(): Promise<void> {
    const checks = this.replicas.map(async (replica) => {
      try {
        const startTime = Date.now();
        const result = await replica.pool.query<{
          is_in_recovery: boolean;
          lag_ms: number | null;
        }>(`
          SELECT 
            pg_is_in_recovery() as is_in_recovery,
            EXTRACT(EPOCH FROM (NOW() - pg_last_xact_replay_timestamp())) * 1000 as lag_ms
        `);

        const row = result.rows[0];
        const queryTime = Date.now() - startTime;

        replica.isHealthy = true;
        replica.lagMs = row.lag_ms ?? 0;
        replica.lastCheckedAt = Date.now();

        if (queryTime > 1000) {
          console.warn(`Replica ${replica.host} health check took ${queryTime}ms`);
        }

        this.emit('replicaHealthy', {
          host: replica.host,
          lagMs: replica.lagMs,
        });
      } catch (error) {
        replica.isHealthy = false;
        replica.lastCheckedAt = Date.now();
        console.error(`Replica ${replica.host} health check failed:`, error);
        this.emit('replicaUnhealthy', {
          host: replica.host,
          error,
        });
      }
    });

    await Promise.allSettled(checks);
  }

  // ==========================================
  // UTILITY METHODS
  // ==========================================

  private isWriteQuery(sql: string): boolean {
    const normalizedSql = sql.trim().toUpperCase();
    const writeKeywords = [
      'INSERT', 'UPDATE', 'DELETE', 'CREATE', 'DROP',
      'ALTER', 'TRUNCATE', 'UPSERT', 'MERGE',
      'CALL', 'EXECUTE', 'DO',
    ];
    return writeKeywords.some(kw => normalizedSql.startsWith(kw));
  }

  getStats() {
    return {
      primary: {
        totalCount: this.primaryPool.totalCount,
        idleCount: this.primaryPool.idleCount,
        waitingCount: this.primaryPool.waitingCount,
      },
      replicas: this.replicas.map(r => ({
        host: r.host,
        isHealthy: r.isHealthy,
        lagMs: r.lagMs,
        connectionCount: r.connectionCount,
        lastCheckedAt: new Date(r.lastCheckedAt).toISOString(),
        poolStats: {
          totalCount: r.pool.totalCount,
          idleCount: r.pool.idleCount,
          waitingCount: r.pool.waitingCount,
        },
      })),
    };
  }

  async end(): Promise<void> {
    if (this.healthCheckTimer) {
      clearInterval(this.healthCheckTimer);
    }

    await this.primaryPool.end();
    await Promise.all(this.replicas.map(r => r.pool.end()));
  }
}

export { DatabaseManager, DatabaseConfig, PoolConfig };
```

---

## 4. Repository Pattern สำหรับ Read/Write Separation

### 4.1 Base Repository

```typescript
// src/repositories/BaseRepository.ts
import { PoolClient, QueryResult, QueryResultRow } from 'pg';
import { DatabaseManager } from '../database/DatabaseManager';

export abstract class BaseRepository {
  constructor(protected readonly db: DatabaseManager) {}

  protected async findOne<T extends QueryResultRow>(
    sql: string,
    params: unknown[],
    context?: string
  ): Promise<T | null> {
    const result = await this.db.query<T>(sql, params, context);
    return result.rows[0] ?? null;
  }

  protected async findMany<T extends QueryResultRow>(
    sql: string,
    params: unknown[],
    context?: string
  ): Promise<T[]> {
    const result = await this.db.query<T>(sql, params, context);
    return result.rows;
  }

  protected async execute(
    sql: string,
    params: unknown[],
    context?: string
  ): Promise<QueryResult> {
    return this.db.query(sql, params, context);
  }

  protected async withTransaction<T>(
    callback: (client: PoolClient) => Promise<T>,
    context?: string
  ): Promise<T> {
    return this.db.withTransaction(callback, context);
  }
}
```

### 4.2 User Repository

```typescript
// src/repositories/UserRepository.ts
import { PoolClient } from 'pg';
import { BaseRepository } from './BaseRepository';

interface User {
  id: number;
  email: string;
  name: string;
  createdAt: Date;
  updatedAt: Date;
}

interface CreateUserInput {
  email: string;
  name: string;
  passwordHash: string;
}

interface UpdateUserInput {
  name?: string;
  email?: string;
}

export class UserRepository extends BaseRepository {
  // READ - ไปที่ Replica
  async findById(id: number, sessionId?: string): Promise<User | null> {
    return this.findOne<User>(
      'SELECT id, email, name, created_at as "createdAt", updated_at as "updatedAt" FROM users WHERE id = $1',
      [id],
      sessionId  // Context สำหรับ Read-After-Write
    );
  }

  async findByEmail(email: string, sessionId?: string): Promise<User | null> {
    return this.findOne<User>(
      'SELECT id, email, name, created_at as "createdAt", updated_at as "updatedAt" FROM users WHERE email = $1',
      [email],
      sessionId
    );
  }

  async findAll(
    page: number = 1,
    limit: number = 20,
    sessionId?: string
  ): Promise<{ users: User[]; total: number }> {
    const offset = (page - 1) * limit;

    const [users, countResult] = await Promise.all([
      this.findMany<User>(
        `SELECT id, email, name, created_at as "createdAt", updated_at as "updatedAt" 
         FROM users 
         ORDER BY id 
         LIMIT $1 OFFSET $2`,
        [limit, offset],
        sessionId
      ),
      this.findOne<{ count: string }>(
        'SELECT COUNT(*) FROM users',
        [],
        sessionId
      ),
    ]);

    return {
      users,
      total: parseInt(countResult?.count ?? '0'),
    };
  }

  // WRITE - ไปที่ Primary
  async create(input: CreateUserInput, sessionId?: string): Promise<User> {
    const result = await this.execute(
      `INSERT INTO users (email, name, password_hash, created_at, updated_at)
       VALUES ($1, $2, $3, NOW(), NOW())
       RETURNING id, email, name, created_at as "createdAt", updated_at as "updatedAt"`,
      [input.email, input.name, input.passwordHash],
      sessionId
    );

    // บันทึก Write Context เพื่อ Read-After-Write Consistency
    if (sessionId) {
      this.db.markWrite(sessionId);
    }

    return result.rows[0];
  }

  async update(id: number, input: UpdateUserInput, sessionId?: string): Promise<User | null> {
    const updates: string[] = [];
    const params: unknown[] = [];
    let paramCount = 1;

    if (input.name !== undefined) {
      updates.push(`name = $${paramCount++}`);
      params.push(input.name);
    }
    if (input.email !== undefined) {
      updates.push(`email = $${paramCount++}`);
      params.push(input.email);
    }

    if (updates.length === 0) {
      return this.findById(id, sessionId);
    }

    updates.push(`updated_at = NOW()`);
    params.push(id);

    const result = await this.execute(
      `UPDATE users SET ${updates.join(', ')} 
       WHERE id = $${paramCount}
       RETURNING id, email, name, created_at as "createdAt", updated_at as "updatedAt"`,
      params,
      sessionId
    );

    if (sessionId) {
      this.db.markWrite(sessionId);
    }

    return result.rows[0] ?? null;
  }

  async delete(id: number): Promise<boolean> {
    const result = await this.execute(
      'DELETE FROM users WHERE id = $1',
      [id]
    );
    return (result.rowCount ?? 0) > 0;
  }

  // COMPLEX TRANSACTION - ใช้ Primary
  async createWithProfile(
    userInput: CreateUserInput,
    profileData: { bio: string; avatarUrl?: string },
    sessionId?: string
  ): Promise<User> {
    return this.withTransaction(async (client: PoolClient) => {
      // สร้าง User
      const userResult = await client.query(
        `INSERT INTO users (email, name, password_hash, created_at, updated_at)
         VALUES ($1, $2, $3, NOW(), NOW())
         RETURNING id, email, name, created_at as "createdAt", updated_at as "updatedAt"`,
        [userInput.email, userInput.name, userInput.passwordHash]
      );
      const user = userResult.rows[0];

      // สร้าง Profile
      await client.query(
        `INSERT INTO profiles (user_id, bio, avatar_url, created_at)
         VALUES ($1, $2, $3, NOW())`,
        [user.id, profileData.bio, profileData.avatarUrl ?? null]
      );

      // Audit Log
      await client.query(
        `INSERT INTO audit_logs (action, entity_type, entity_id, created_at)
         VALUES ('USER_CREATED', 'user', $1, NOW())`,
        [user.id]
      );

      return user;
    }, sessionId);
  }
}
```

---

## 5. Transaction Handling: ต้องใช้ Primary เสมอ

### 5.1 ทำไม Transaction ต้องใช้ Primary

```
ปัญหา: ถ้าอ่าน-เขียนต่าง Connection
Replica อาจมี Lag ทำให้เห็นข้อมูลเก่า

ตัวอย่างอันตราย:
1. BEGIN บน Replica
2. SELECT balance FROM accounts WHERE id = 1 → ได้ 1000 (อาจ Stale)
3. UPDATE accounts SET balance = balance - 500 WHERE id = 1 → ERROR (Replica is Read-Only!)

วิธีที่ถูกต้อง:
1. BEGIN บน Primary
2. SELECT balance... บน Primary (ได้ข้อมูลปัจจุบัน)
3. UPDATE balance... บน Primary
4. COMMIT
```

### 5.2 Transaction Decorator

```typescript
// src/decorators/Transaction.ts

import { DatabaseManager } from '../database/DatabaseManager';

// Simple Transaction Helper
export async function runInTransaction<T>(
  db: DatabaseManager,
  fn: (client: import('pg').PoolClient) => Promise<T>,
  sessionId?: string
): Promise<T> {
  return db.withTransaction(fn, sessionId);
}

// ตัวอย่างการใช้
async function transferMoney(
  db: DatabaseManager,
  fromAccountId: number,
  toAccountId: number,
  amount: number,
  sessionId: string
): Promise<void> {
  await runInTransaction(db, async (client) => {
    // ต้องใช้ SELECT ... FOR UPDATE เพื่อ Lock Row
    const fromAccount = await client.query(
      'SELECT balance FROM accounts WHERE id = $1 FOR UPDATE',
      [fromAccountId]
    );

    if (fromAccount.rows[0].balance < amount) {
      throw new Error('Insufficient funds');
    }

    await client.query(
      'UPDATE accounts SET balance = balance - $1 WHERE id = $2',
      [amount, fromAccountId]
    );

    await client.query(
      'UPDATE accounts SET balance = balance + $1 WHERE id = $2',
      [amount, toAccountId]
    );

    await client.query(
      `INSERT INTO transactions (from_account, to_account, amount, created_at)
       VALUES ($1, $2, $3, NOW())`,
      [fromAccountId, toAccountId, amount]
    );
  }, sessionId);
}
```

---

## 6. Read-After-Write Consistency Problem

### 6.1 ปัญหา

```
Timeline:
t=0  User POST /update-profile (name: "Alice")
     → เขียนไปที่ Primary ✅
     
t=1  User GET /profile
     → อ่านจาก Replica
     → Replica ยังไม่ sync → ได้ชื่อเก่า "Bob" ❌
     
t=2  Replication sync เสร็จ
     → Replica มีชื่อ "Alice" ✅
```

### 6.2 Solutions

**Solution 1: Sticky Session (ใช้ Primary หลัง Write)**

```typescript
// ใน DatabaseManager เราแก้ไขแล้ว
// หลัง Write ใน sessionId context จะใช้ Primary สำหรับ Read
// ภายใน readAfterWriteTimeoutMs milliseconds

const db = new DatabaseManager({
  primary: primaryConfig,
  replicas: replicaConfigs,
  options: {
    readAfterWriteTimeoutMs: 2000,  // ใช้ Primary 2 วินาทีหลัง Write
  },
});

// Middleware สำหรับ Express
app.use((req, res, next) => {
  // ใช้ Session ID เป็น Write Context
  req.dbContext = req.session?.id ?? req.headers['x-request-id'] as string;
  next();
});

// Controller
async function updateUserProfile(req: Request, res: Response) {
  const sessionId = req.dbContext;
  
  // WRITE ไปที่ Primary
  await userRepo.update(req.user.id, req.body, sessionId);
  
  // READ ทันทีหลัง Write - จะใช้ Primary (เพราะ readAfterWriteTimeoutMs)
  const updatedUser = await userRepo.findById(req.user.id, sessionId);
  
  res.json(updatedUser);
}
```

**Solution 2: Synchronous Replication**

```ini
# postgresql.conf บน Primary
synchronous_standby_names = 'ANY 1 (replica1, replica2)'
synchronous_commit = remote_apply  # รอ Replica Apply แล้วค่อย COMMIT
```

**Solution 3: Version/Timestamp Check**

```typescript
// ตรวจสอบว่า Replica ทันเพียงพอ
async function readWithLagCheck<T>(
  primaryPool: Pool,
  replicaPool: Pool,
  query: string,
  params: unknown[],
  maxLagMs: number = 1000
): Promise<T[]> {
  // ตรวจ Lag ของ Replica
  const lagResult = await replicaPool.query<{ lag_ms: number }>(`
    SELECT EXTRACT(EPOCH FROM (NOW() - pg_last_xact_replay_timestamp())) * 1000 as lag_ms
  `);

  const lagMs = lagResult.rows[0]?.lag_ms ?? Infinity;

  if (lagMs > maxLagMs) {
    console.warn(`Replica lag ${lagMs}ms exceeds ${maxLagMs}ms, using primary`);
    const result = await primaryPool.query<T>(query, params);
    return result.rows;
  }

  const result = await replicaPool.query<T>(query, params);
  return result.rows;
}
```

---

## 7. TypeScript Implementation: Complete Service Layer

```typescript
// src/services/UserService.ts
import { UserRepository } from '../repositories/UserRepository';
import { DatabaseManager } from '../database/DatabaseManager';

interface PaginatedResult<T> {
  data: T[];
  pagination: {
    page: number;
    limit: number;
    total: number;
    totalPages: number;
  };
}

export class UserService {
  private userRepo: UserRepository;

  constructor(private readonly db: DatabaseManager) {
    this.userRepo = new UserRepository(db);
  }

  async getUserById(id: number, sessionId?: string) {
    const user = await this.userRepo.findById(id, sessionId);
    if (!user) {
      throw new Error(`User ${id} not found`);
    }
    return user;
  }

  async getUsersPaginated(
    page: number,
    limit: number,
    sessionId?: string
  ): Promise<PaginatedResult<any>> {
    const { users, total } = await this.userRepo.findAll(page, limit, sessionId);
    return {
      data: users,
      pagination: {
        page,
        limit,
        total,
        totalPages: Math.ceil(total / limit),
      },
    };
  }

  async createUser(
    email: string,
    name: string,
    password: string,
    sessionId?: string
  ) {
    // Check duplicate
    const existing = await this.userRepo.findByEmail(email, sessionId);
    if (existing) {
      throw new Error(`Email ${email} already exists`);
    }

    // Hash password (simplified)
    const passwordHash = Buffer.from(password).toString('base64');

    const user = await this.userRepo.create(
      { email, name, passwordHash },
      sessionId
    );

    return user;
  }

  async updateUser(
    id: number,
    updates: { name?: string; email?: string },
    sessionId?: string
  ) {
    const user = await this.userRepo.update(id, updates, sessionId);
    if (!user) {
      throw new Error(`User ${id} not found`);
    }
    return user;
  }
}
```

### 7.1 Express.js Integration

```typescript
// src/app.ts
import express, { Request, Response, NextFunction } from 'express';
import { DatabaseManager } from './database/DatabaseManager';
import { UserService } from './services/UserService';

// สร้าง Database Manager
const db = new DatabaseManager({
  primary: {
    host: process.env.PRIMARY_DB_HOST || 'localhost',
    port: parseInt(process.env.PRIMARY_DB_PORT || '5432'),
    database: process.env.DB_NAME || 'myapp',
    user: process.env.DB_USER || 'app_user',
    password: process.env.DB_PASSWORD || 'password',
    max: 10,
  },
  replicas: [
    {
      host: process.env.REPLICA1_DB_HOST || 'replica1',
      port: 5432,
      database: process.env.DB_NAME || 'myapp',
      user: process.env.DB_USER || 'app_user',
      password: process.env.DB_PASSWORD || 'password',
      max: 20,
    },
    {
      host: process.env.REPLICA2_DB_HOST || 'replica2',
      port: 5432,
      database: process.env.DB_NAME || 'myapp',
      user: process.env.DB_USER || 'app_user',
      password: process.env.DB_PASSWORD || 'password',
      max: 20,
    },
  ],
  options: {
    maxReplicaLagMs: 5000,
    healthCheckIntervalMs: 10000,
    readAfterWriteTimeoutMs: 2000,
  },
});

// Monitor Events
db.on('replicaUnhealthy', ({ host, error }) => {
  console.error(`Replica ${host} is unhealthy:`, error.message);
  // ส่ง Alert ไปยัง Monitoring System
});

db.on('replicaHealthy', ({ host, lagMs }) => {
  if (lagMs > 1000) {
    console.warn(`Replica ${host} lag: ${lagMs}ms`);
  }
});

const userService = new UserService(db);
const app = express();
app.use(express.json());

// Middleware: ดึง Session ID จาก Request
app.use((req: Request & { dbContext?: string }, res, next) => {
  req.dbContext = req.headers['x-session-id'] as string
    || req.cookies?.sessionId
    || `anon-${req.ip}`;
  next();
});

// Routes
app.get('/users', async (req: any, res: Response) => {
  try {
    const page = parseInt(req.query.page as string) || 1;
    const limit = parseInt(req.query.limit as string) || 20;

    const result = await userService.getUsersPaginated(
      page,
      limit,
      req.dbContext
    );
    res.json(result);
  } catch (error: any) {
    res.status(500).json({ error: error.message });
  }
});

app.get('/users/:id', async (req: any, res: Response) => {
  try {
    const user = await userService.getUserById(
      parseInt(req.params.id),
      req.dbContext
    );
    res.json(user);
  } catch (error: any) {
    if (error.message.includes('not found')) {
      res.status(404).json({ error: error.message });
    } else {
      res.status(500).json({ error: error.message });
    }
  }
});

app.post('/users', async (req: any, res: Response) => {
  try {
    const { email, name, password } = req.body;
    const user = await userService.createUser(
      email, name, password, req.dbContext
    );
    res.status(201).json(user);
  } catch (error: any) {
    if (error.message.includes('already exists')) {
      res.status(409).json({ error: error.message });
    } else {
      res.status(500).json({ error: error.message });
    }
  }
});

app.patch('/users/:id', async (req: any, res: Response) => {
  try {
    const user = await userService.updateUser(
      parseInt(req.params.id),
      req.body,
      req.dbContext
    );
    res.json(user);
  } catch (error: any) {
    res.status(500).json({ error: error.message });
  }
});

// Health Check
app.get('/health', async (req, res) => {
  const stats = db.getStats();
  res.json({
    status: 'ok',
    database: stats,
  });
});

// Graceful Shutdown
process.on('SIGTERM', async () => {
  console.log('Shutting down...');
  await db.end();
  process.exit(0);
});

app.listen(3000, () => {
  console.log('Server running on port 3000');
  console.log('Primary:', process.env.PRIMARY_DB_HOST);
  console.log('Replicas:', [
    process.env.REPLICA1_DB_HOST,
    process.env.REPLICA2_DB_HOST,
  ].filter(Boolean).join(', '));
});

export default app;
```

---

## 8. Prisma Read Replicas Configuration

```typescript
// prisma/schema.prisma
// ไม่ต้องเปลี่ยน Schema

// src/database/prisma.ts
import { PrismaClient } from '@prisma/client';
import { readReplicas } from '@prisma/extension-read-replicas';

const prisma = new PrismaClient().$extends(
  readReplicas({
    url: [
      process.env.REPLICA1_DATABASE_URL!,
      process.env.REPLICA2_DATABASE_URL!,
    ],
  })
);

// การใช้งาน - Prisma จัดการ Routing อัตโนมัติ
async function example() {
  // READ: ไปที่ Replica อัตโนมัติ
  const users = await prisma.user.findMany();

  // WRITE: ไปที่ Primary อัตโนมัติ
  const newUser = await prisma.user.create({
    data: { email: 'test@example.com', name: 'Test' },
  });

  // บังคับใช้ Primary สำหรับ Read
  const criticalUser = await prisma.$primary().user.findUnique({
    where: { id: 1 },
  });

  // บังคับใช้ Replica สำหรับ Write (ไม่แนะนำ)
  // const replicaResult = await prisma.$replica().user.findMany();
}
```

---

## 9. TypeORM Read Replicas

```typescript
// src/database/typeorm.ts
import { DataSource } from 'typeorm';

export const AppDataSource = new DataSource({
  type: 'postgres',
  
  // Primary (Master)
  replication: {
    master: {
      host: process.env.PRIMARY_DB_HOST!,
      port: 5432,
      username: process.env.DB_USER!,
      password: process.env.DB_PASSWORD!,
      database: process.env.DB_NAME!,
    },
    slaves: [
      {
        host: process.env.REPLICA1_DB_HOST!,
        port: 5432,
        username: process.env.DB_USER!,
        password: process.env.DB_PASSWORD!,
        database: process.env.DB_NAME!,
      },
      {
        host: process.env.REPLICA2_DB_HOST!,
        port: 5432,
        username: process.env.DB_USER!,
        password: process.env.DB_PASSWORD!,
        database: process.env.DB_NAME!,
      },
    ],
  },

  entities: [__dirname + '/../entities/*.ts'],
  synchronize: process.env.NODE_ENV === 'development',
  logging: process.env.NODE_ENV === 'development',
});

// การใช้งาน TypeORM Read Replicas
async function typeormExample() {
  await AppDataSource.initialize();

  const userRepository = AppDataSource.getRepository('User');

  // TypeORM จัดการ Routing อัตโนมัติ:
  // find, findOne → Slave (Replica)
  // save, update, delete → Master (Primary)
  
  // READ: ไปที่ Slave อัตโนมัติ
  const users = await userRepository.find();

  // WRITE: ไปที่ Master อัตโนมัติ
  const user = userRepository.create({ email: 'test@test.com', name: 'Test' });
  await userRepository.save(user);

  // บังคับใช้ Master สำหรับ Read (Read-After-Write)
  const freshUser = await AppDataSource
    .createQueryRunner('master')
    .manager
    .findOne('User', { where: { id: user.id } });
}
```

---

## 10. Load Balancing Strategies เปรียบเทียบ

### 10.1 Round-Robin

```typescript
class RoundRobinSelector {
  private index = 0;

  select<T>(items: T[]): T {
    const item = items[this.index % items.length];
    this.index = (this.index + 1) % items.length;
    return item;
  }
}
```

### 10.2 Least Connections

```typescript
class LeastConnectionsSelector {
  select<T extends { connectionCount: number }>(items: T[]): T {
    return items.reduce((min, item) =>
      item.connectionCount < min.connectionCount ? item : min
    );
  }
}
```

### 10.3 Weighted Round-Robin

```typescript
interface WeightedReplica {
  pool: any;
  host: string;
  weight: number;  // Higher = more traffic
}

class WeightedRoundRobinSelector {
  private weightedList: WeightedReplica[] = [];

  constructor(replicas: WeightedReplica[]) {
    // สร้าง weighted list
    for (const replica of replicas) {
      for (let i = 0; i < replica.weight; i++) {
        this.weightedList.push(replica);
      }
    }
    // Shuffle
    this.weightedList.sort(() => Math.random() - 0.5);
  }

  private index = 0;

  select(): WeightedReplica {
    const item = this.weightedList[this.index % this.weightedList.length];
    this.index++;
    return item;
  }
}

// ใช้งาน:
const selector = new WeightedRoundRobinSelector([
  { pool: pool1, host: 'replica1', weight: 3 },  // รับ 60% traffic
  { pool: pool2, host: 'replica2', weight: 2 },  // รับ 40% traffic
]);
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **ทำไมต้องแยก Read/Write** - ลด Load บน Primary
2. **DatabaseManager Class** - จัดการ Primary + Replicas
3. **Round-Robin Load Balancing** - กระจาย Read Traffic
4. **Health Check** - ตรวจสอบ Replica Status และ Lag
5. **Repository Pattern** - แยก Business Logic จาก DB Logic
6. **Transaction Safety** - ต้องใช้ Primary เสมอ
7. **Read-After-Write Consistency** - แก้ปัญหา Stale Read
8. **Sticky Session** - วิธีง่ายๆ แก้ Consistency
9. **Prisma Read Replicas** - Extension สำเร็จรูป
10. **TypeORM Read Replicas** - Built-in Support

ในบทถัดไปเราจะเรียนรู้ Redis Cluster และ Sentinel สำหรับ High Availability ของ Redis

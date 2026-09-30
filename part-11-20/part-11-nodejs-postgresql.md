# Part 11: เชื่อมต่อ Node.js กับ PostgreSQL (pg module)

## บทนำ

ในบทนี้เราจะเรียนรู้การเชื่อมต่อ Node.js กับ PostgreSQL โดยใช้ `pg` module ซึ่งเป็น native PostgreSQL client สำหรับ Node.js ที่มีประสิทธิภาพสูงและใช้งานกันอย่างแพร่หลาย

---

## 1. การติดตั้ง

```bash
# ติดตั้ง pg module
npm install pg

# ติดตั้ง TypeScript types
npm install --save-dev @types/pg

# ติดตั้ง dependencies เพิ่มเติม
npm install dotenv
npm install --save-dev typescript ts-node @types/node
```

### tsconfig.json
```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

---

## 2. Connection Configuration

### 2.1 Config Object
```typescript
// src/config/database.ts
import { Pool, PoolConfig } from 'pg';
import dotenv from 'dotenv';

dotenv.config();

export interface DatabaseConfig extends PoolConfig {
  host: string;
  port: number;
  database: string;
  user: string;
  password: string;
  ssl?: boolean | { rejectUnauthorized: boolean };
}

export const dbConfig: DatabaseConfig = {
  host: process.env.DB_HOST || 'localhost',
  port: parseInt(process.env.DB_PORT || '5432', 10),
  database: process.env.DB_NAME || 'myapp',
  user: process.env.DB_USER || 'postgres',
  password: process.env.DB_PASSWORD || 'password',
  // SSL configuration
  ssl: process.env.DB_SSL === 'true'
    ? { rejectUnauthorized: process.env.DB_SSL_REJECT_UNAUTHORIZED !== 'false' }
    : false,
  // Pool configuration
  max: parseInt(process.env.DB_POOL_MAX || '20', 10),
  min: parseInt(process.env.DB_POOL_MIN || '2', 10),
  idleTimeoutMillis: parseInt(process.env.DB_IDLE_TIMEOUT || '30000', 10),
  connectionTimeoutMillis: parseInt(process.env.DB_CONNECT_TIMEOUT || '2000', 10),
  maxUses: parseInt(process.env.DB_MAX_USES || '7500', 10),
};
```

### 2.2 Connection String
```typescript
// Connection String format:
// postgresql://user:password@host:port/database?sslmode=require

const connectionString = `postgresql://${process.env.DB_USER}:${process.env.DB_PASSWORD}@${process.env.DB_HOST}:${process.env.DB_PORT}/${process.env.DB_NAME}`;

// หรือใช้ direct connection string
const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  ssl: process.env.NODE_ENV === 'production'
    ? { rejectUnauthorized: false }
    : false,
});
```

### 2.3 Environment Variables (.env)
```bash
# .env
DB_HOST=localhost
DB_PORT=5432
DB_NAME=myapp_db
DB_USER=app_user
DB_PASSWORD=secure_password_here
DB_SSL=false
DB_SSL_REJECT_UNAUTHORIZED=true

# Pool settings
DB_POOL_MAX=20
DB_POOL_MIN=2
DB_IDLE_TIMEOUT=30000
DB_CONNECT_TIMEOUT=2000
DB_MAX_USES=7500

# Full connection string (alternative)
DATABASE_URL=postgresql://app_user:secure_password_here@localhost:5432/myapp_db
```

---

## 3. Connection Pool

Connection Pool เป็นเทคนิคสำคัญในการจัดการ database connections เพื่อเพิ่มประสิทธิภาพและลด overhead

```typescript
// src/db/pool.ts
import { Pool, PoolClient, QueryResult, QueryResultRow } from 'pg';
import { dbConfig } from '../config/database';

class DatabasePool {
  private pool: Pool;
  private static instance: DatabasePool;

  private constructor() {
    this.pool = new Pool(dbConfig);

    // Event listeners
    this.pool.on('connect', (client: PoolClient) => {
      console.log(`[DB] New client connected. Pool size: ${this.pool.totalCount}`);
    });

    this.pool.on('acquire', (client: PoolClient) => {
      console.log(`[DB] Client acquired. Idle: ${this.pool.idleCount}, Total: ${this.pool.totalCount}`);
    });

    this.pool.on('remove', (client: PoolClient) => {
      console.log(`[DB] Client removed. Pool size: ${this.pool.totalCount}`);
    });

    this.pool.on('error', (err: Error, client: PoolClient) => {
      console.error('[DB] Unexpected error on idle client:', err);
      process.exit(-1);
    });
  }

  static getInstance(): DatabasePool {
    if (!DatabasePool.instance) {
      DatabasePool.instance = new DatabasePool();
    }
    return DatabasePool.instance;
  }

  // Execute query with automatic client release
  async query<T extends QueryResultRow = QueryResultRow>(
    text: string,
    params?: unknown[]
  ): Promise<QueryResult<T>> {
    const start = Date.now();
    const result = await this.pool.query<T>(text, params);
    const duration = Date.now() - start;
    
    if (process.env.LOG_QUERIES === 'true') {
      console.log('[DB] Executed query', { text, duration, rows: result.rowCount });
    }
    
    return result;
  }

  // Get client for transactions
  async getClient(): Promise<PoolClient> {
    const client = await this.pool.connect();
    const query = client.query.bind(client);
    const release = client.release.bind(client);

    // Override release to log
    const timeout = setTimeout(() => {
      console.error('[DB] Client checked out for too long!');
    }, 5000);

    client.release = (err?: Error | boolean) => {
      clearTimeout(timeout);
      client.query = query;
      client.release = release;
      return release(err as Error);
    };

    return client;
  }

  // Pool stats
  getStats() {
    return {
      total: this.pool.totalCount,
      idle: this.pool.idleCount,
      waiting: this.pool.waitingCount,
    };
  }

  // Graceful shutdown
  async end(): Promise<void> {
    console.log('[DB] Closing connection pool...');
    await this.pool.end();
    console.log('[DB] Pool closed.');
  }

  // Health check
  async healthCheck(): Promise<boolean> {
    try {
      const result = await this.pool.query('SELECT 1 as health');
      return result.rows[0].health === 1;
    } catch (error) {
      console.error('[DB] Health check failed:', error);
      return false;
    }
  }
}

export const db = DatabasePool.getInstance();
export default db;
```

---

## 4. Basic Queries

### 4.1 Simple Query
```typescript
// src/db/queries.ts
import { db } from './pool';
import { QueryResult } from 'pg';

// Simple query
async function getAllUsers(): Promise<QueryResult> {
  const result = await db.query('SELECT * FROM users ORDER BY created_at DESC');
  return result;
}

// Query with parameters (parameterized query - ALWAYS ใช้สำหรับ user input)
async function getUserById(id: number): Promise<QueryResult> {
  // $1, $2, $3 คือ placeholders
  const result = await db.query('SELECT * FROM users WHERE id = $1', [id]);
  return result;
}

// Multiple parameters
async function getUsersByFilter(
  active: boolean,
  role: string,
  limit: number,
  offset: number
): Promise<QueryResult> {
  const result = await db.query(
    `SELECT id, username, email, role, created_at
     FROM users
     WHERE is_active = $1 AND role = $2
     ORDER BY created_at DESC
     LIMIT $3 OFFSET $4`,
    [active, role, limit, offset]
  );
  return result;
}

// INSERT และ return inserted row
async function createUser(
  username: string,
  email: string,
  passwordHash: string
): Promise<QueryResult> {
  const result = await db.query(
    `INSERT INTO users (username, email, password_hash, created_at, updated_at)
     VALUES ($1, $2, $3, NOW(), NOW())
     RETURNING id, username, email, created_at`,
    [username, email, passwordHash]
  );
  return result;
}

// UPDATE
async function updateUser(
  id: number,
  updates: { username?: string; email?: string }
): Promise<QueryResult> {
  const { username, email } = updates;
  const result = await db.query(
    `UPDATE users
     SET username = COALESCE($2, username),
         email = COALESCE($3, email),
         updated_at = NOW()
     WHERE id = $1
     RETURNING id, username, email, updated_at`,
    [id, username || null, email || null]
  );
  return result;
}

// DELETE
async function deleteUser(id: number): Promise<QueryResult> {
  const result = await db.query(
    'DELETE FROM users WHERE id = $1 RETURNING id',
    [id]
  );
  return result;
}
```

### 4.2 TypeScript Row Types
```typescript
// src/types/user.ts
export interface UserRow {
  id: number;
  username: string;
  email: string;
  password_hash: string;
  role: 'admin' | 'user' | 'moderator';
  is_active: boolean;
  created_at: Date;
  updated_at: Date;
}

export interface UserPublic {
  id: number;
  username: string;
  email: string;
  role: string;
  created_at: Date;
}

// Typed query result
import { QueryResult } from 'pg';

async function getTypedUser(id: number): Promise<UserRow | null> {
  const result = await db.query<UserRow>(
    'SELECT * FROM users WHERE id = $1',
    [id]
  );
  return result.rows[0] || null;
}

async function getAllTypedUsers(): Promise<UserRow[]> {
  const result = await db.query<UserRow>(
    'SELECT * FROM users WHERE is_active = true'
  );
  return result.rows;
}
```

---

## 5. Error Handling

```typescript
// src/utils/db-errors.ts
export enum PgErrorCode {
  // Connection errors
  CONNECTION_REFUSED = 'ECONNREFUSED',
  
  // Constraint violations
  UNIQUE_VIOLATION = '23505',
  FOREIGN_KEY_VIOLATION = '23503',
  NOT_NULL_VIOLATION = '23502',
  CHECK_VIOLATION = '23514',
  
  // Transaction errors
  SERIALIZATION_FAILURE = '40001',
  DEADLOCK_DETECTED = '40P01',
  
  // Data errors
  INVALID_TEXT_REPRESENTATION = '22P02',
  NUMERIC_VALUE_OUT_OF_RANGE = '22003',
  
  // Auth errors
  INVALID_PASSWORD = '28P01',
  INVALID_CATALOG_NAME = '3D000',
}

export class DatabaseError extends Error {
  constructor(
    message: string,
    public readonly code?: string,
    public readonly detail?: string,
    public readonly constraint?: string
  ) {
    super(message);
    this.name = 'DatabaseError';
  }
}

export function handleDatabaseError(error: unknown): never {
  if (error instanceof Error) {
    const pgError = error as Error & {
      code?: string;
      detail?: string;
      constraint?: string;
      column?: string;
      table?: string;
    };

    switch (pgError.code) {
      case PgErrorCode.UNIQUE_VIOLATION:
        throw new DatabaseError(
          `Duplicate entry: ${pgError.detail || 'unique constraint violated'}`,
          pgError.code,
          pgError.detail,
          pgError.constraint
        );
      
      case PgErrorCode.FOREIGN_KEY_VIOLATION:
        throw new DatabaseError(
          `Foreign key violation: ${pgError.detail || 'referenced record not found'}`,
          pgError.code,
          pgError.detail,
          pgError.constraint
        );
      
      case PgErrorCode.NOT_NULL_VIOLATION:
        throw new DatabaseError(
          `Required field missing: ${pgError.column}`,
          pgError.code,
          pgError.detail,
          pgError.constraint
        );
      
      case PgErrorCode.SERIALIZATION_FAILURE:
      case PgErrorCode.DEADLOCK_DETECTED:
        throw new DatabaseError(
          'Transaction conflict, please retry',
          pgError.code,
          pgError.detail
        );
      
      default:
        throw new DatabaseError(
          `Database error: ${pgError.message}`,
          pgError.code,
          pgError.detail
        );
    }
  }
  
  throw new DatabaseError('Unknown database error');
}

// Usage example
async function safeCreateUser(username: string, email: string, passwordHash: string) {
  try {
    const result = await db.query<UserRow>(
      `INSERT INTO users (username, email, password_hash, created_at, updated_at)
       VALUES ($1, $2, $3, NOW(), NOW())
       RETURNING *`,
      [username, email, passwordHash]
    );
    return result.rows[0];
  } catch (error) {
    handleDatabaseError(error);
  }
}
```

---

## 6. Transactions

```typescript
// src/db/transactions.ts
import { PoolClient } from 'pg';
import { db } from './pool';

// Basic transaction
async function transferBalance(
  fromUserId: number,
  toUserId: number,
  amount: number
): Promise<void> {
  const client = await db.getClient();
  
  try {
    await client.query('BEGIN');
    
    // Deduct from sender
    const deductResult = await client.query(
      `UPDATE accounts
       SET balance = balance - $2
       WHERE user_id = $1 AND balance >= $2
       RETURNING balance`,
      [fromUserId, amount]
    );
    
    if (deductResult.rowCount === 0) {
      throw new Error('Insufficient balance');
    }
    
    // Add to receiver
    await client.query(
      `UPDATE accounts
       SET balance = balance + $2
       WHERE user_id = $1`,
      [toUserId, amount]
    );
    
    // Log transaction
    await client.query(
      `INSERT INTO transactions (from_user_id, to_user_id, amount, created_at)
       VALUES ($1, $2, $3, NOW())`,
      [fromUserId, toUserId, amount]
    );
    
    await client.query('COMMIT');
    console.log('[DB] Transaction committed successfully');
    
  } catch (error) {
    await client.query('ROLLBACK');
    console.error('[DB] Transaction rolled back:', error);
    throw error;
  } finally {
    client.release();
  }
}

// Transaction helper wrapper
async function withTransaction<T>(
  callback: (client: PoolClient) => Promise<T>
): Promise<T> {
  const client = await db.getClient();
  
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

// Transaction with savepoint
async function complexTransaction(
  operations: Array<{
    query: string;
    params: unknown[];
    savepointName?: string;
  }>
): Promise<void> {
  const client = await db.getClient();
  const savepoints: string[] = [];
  
  try {
    await client.query('BEGIN');
    
    for (const op of operations) {
      if (op.savepointName) {
        await client.query(`SAVEPOINT ${op.savepointName}`);
        savepoints.push(op.savepointName);
      }
      
      try {
        await client.query(op.query, op.params);
      } catch (error) {
        if (op.savepointName) {
          await client.query(`ROLLBACK TO SAVEPOINT ${op.savepointName}`);
          console.log(`[DB] Rolled back to savepoint: ${op.savepointName}`);
        } else {
          throw error;
        }
      }
    }
    
    await client.query('COMMIT');
  } catch (error) {
    await client.query('ROLLBACK');
    throw error;
  } finally {
    client.release();
  }
}

// Usage of withTransaction helper
async function createUserWithProfile(
  username: string,
  email: string,
  passwordHash: string,
  profileData: { bio: string; avatarUrl: string }
) {
  return withTransaction(async (client) => {
    // Create user
    const userResult = await client.query<{ id: number }>(
      `INSERT INTO users (username, email, password_hash, created_at, updated_at)
       VALUES ($1, $2, $3, NOW(), NOW())
       RETURNING id`,
      [username, email, passwordHash]
    );
    
    const userId = userResult.rows[0].id;
    
    // Create profile
    await client.query(
      `INSERT INTO user_profiles (user_id, bio, avatar_url, created_at)
       VALUES ($1, $2, $3, NOW())`,
      [userId, profileData.bio, profileData.avatarUrl]
    );
    
    return userId;
  });
}
```

---

## 7. Streaming Large Results

สำหรับ query ที่ return ข้อมูลจำนวนมาก การใช้ streaming จะช่วยลด memory usage

```typescript
// src/db/streaming.ts
import { Pool } from 'pg';
import QueryStream from 'pg-query-stream';
import { pipeline, Transform } from 'stream';
import { promisify } from 'util';

const pipelineAsync = promisify(pipeline);

// ต้องติดตั้ง pg-query-stream
// npm install pg-query-stream
// npm install --save-dev @types/pg-query-stream

// Stream large dataset
async function streamAllUsers(
  onRow: (user: UserRow) => Promise<void>
): Promise<void> {
  const pool = new Pool(dbConfig);
  const client = await pool.connect();
  
  try {
    const queryStream = new QueryStream(
      'SELECT * FROM users ORDER BY id',
      [],
      { batchSize: 100 } // process 100 rows at a time
    );
    
    const stream = client.query(queryStream);
    
    for await (const row of stream) {
      await onRow(row as UserRow);
    }
  } finally {
    client.release();
    await pool.end();
  }
}

// Transform stream - process and transform data
async function exportUsersToCSV(outputStream: NodeJS.WritableStream): Promise<void> {
  const pool = new Pool(dbConfig);
  const client = await pool.connect();
  
  try {
    const queryStream = new QueryStream(
      'SELECT id, username, email, role, created_at FROM users WHERE is_active = true',
      [],
      { batchSize: 500 }
    );
    
    const stream = client.query(queryStream);
    
    // Write CSV header
    outputStream.write('id,username,email,role,created_at\n');
    
    const transformer = new Transform({
      objectMode: true,
      transform(row: UserRow, encoding, callback) {
        const csvLine = [
          row.id,
          `"${row.username.replace(/"/g, '""')}"`,
          `"${row.email}"`,
          row.role,
          row.created_at.toISOString(),
        ].join(',') + '\n';
        
        callback(null, csvLine);
      },
    });
    
    await pipelineAsync(stream, transformer, outputStream);
  } finally {
    client.release();
    await pool.end();
  }
}
```

---

## 8. COPY Command สำหรับ Bulk Insert

```typescript
// src/db/bulk-operations.ts
import { Pool } from 'pg';
import { from as copyFrom, to as copyTo } from 'pg-copy-streams';
import { Readable } from 'stream';

// npm install pg-copy-streams
// npm install --save-dev @types/pg-copy-streams

// Bulk insert ด้วย COPY FROM
async function bulkInsertUsers(users: Array<{
  username: string;
  email: string;
  passwordHash: string;
}>): Promise<void> {
  const pool = new Pool(dbConfig);
  const client = await pool.connect();
  
  try {
    // สร้าง CSV data
    const csvData = users
      .map(u => `${u.username}\t${u.email}\t${u.passwordHash}\t\\N\t\\N`)
      .join('\n');
    
    const readable = Readable.from([csvData]);
    
    // COPY FROM stdin
    const copyStream = client.query(
      copyFrom(
        'COPY users (username, email, password_hash, created_at, updated_at) FROM STDIN WITH (FORMAT csv, DELIMITER E\'\\t\', NULL \'\\\\N\')'
      )
    );
    
    await new Promise<void>((resolve, reject) => {
      readable.pipe(copyStream)
        .on('finish', resolve)
        .on('error', reject);
    });
    
    console.log(`[DB] Bulk inserted ${users.length} users`);
  } finally {
    client.release();
    await pool.end();
  }
}

// Bulk insert ด้วย unnest (alternative approach)
async function bulkInsertWithUnnest(users: Array<{
  username: string;
  email: string;
  passwordHash: string;
}>): Promise<void> {
  if (users.length === 0) return;
  
  const usernames = users.map(u => u.username);
  const emails = users.map(u => u.email);
  const passwordHashes = users.map(u => u.passwordHash);
  
  await db.query(
    `INSERT INTO users (username, email, password_hash, created_at, updated_at)
     SELECT * FROM unnest($1::text[], $2::text[], $3::text[],
       array_fill(NOW()::timestamp, ARRAY[$4::int]),
       array_fill(NOW()::timestamp, ARRAY[$4::int]))
     ON CONFLICT (email) DO NOTHING`,
    [usernames, emails, passwordHashes, users.length]
  );
}

// Batch insert with chunks
async function batchInsert<T extends Record<string, unknown>>(
  tableName: string,
  rows: T[],
  columns: (keyof T)[],
  chunkSize: number = 1000
): Promise<number> {
  let totalInserted = 0;
  
  for (let i = 0; i < rows.length; i += chunkSize) {
    const chunk = rows.slice(i, i + chunkSize);
    
    const values: unknown[] = [];
    const placeholders = chunk.map((row, rowIndex) => {
      const rowPlaceholders = columns.map((col, colIndex) => {
        values.push(row[col]);
        return `$${rowIndex * columns.length + colIndex + 1}`;
      });
      return `(${rowPlaceholders.join(', ')})`;
    });
    
    const query = `
      INSERT INTO ${tableName} (${columns.join(', ')})
      VALUES ${placeholders.join(', ')}
      ON CONFLICT DO NOTHING
      RETURNING id
    `;
    
    const result = await db.query(query, values);
    totalInserted += result.rowCount || 0;
    
    console.log(`[DB] Inserted chunk ${Math.ceil((i + 1) / chunkSize)}: ${result.rowCount} rows`);
  }
  
  return totalInserted;
}
```

---

## 9. Connection Health Check

```typescript
// src/db/health.ts
import { db } from './pool';

export interface HealthStatus {
  status: 'healthy' | 'degraded' | 'unhealthy';
  latency: number;
  poolStats: {
    total: number;
    idle: number;
    waiting: number;
  };
  error?: string;
}

export async function checkDatabaseHealth(): Promise<HealthStatus> {
  const startTime = Date.now();
  
  try {
    // Test basic connectivity
    await db.query('SELECT 1');
    const latency = Date.now() - startTime;
    
    const poolStats = db.getStats();
    
    // Determine health based on pool stats and latency
    let status: 'healthy' | 'degraded' | 'unhealthy' = 'healthy';
    
    if (latency > 1000) {
      status = 'degraded';
    }
    
    if (poolStats.waiting > 10) {
      status = 'degraded';
    }
    
    return {
      status,
      latency,
      poolStats,
    };
  } catch (error) {
    return {
      status: 'unhealthy',
      latency: Date.now() - startTime,
      poolStats: db.getStats(),
      error: error instanceof Error ? error.message : 'Unknown error',
    };
  }
}
```

---

## 10. Graceful Shutdown

```typescript
// src/utils/shutdown.ts
import { db } from '../db/pool';

let isShuttingDown = false;

export function setupGracefulShutdown(): void {
  const shutdown = async (signal: string) => {
    if (isShuttingDown) return;
    isShuttingDown = true;
    
    console.log(`\n[App] Received ${signal}. Starting graceful shutdown...`);
    
    try {
      // Stop accepting new requests (handled by HTTP server)
      // Close database pool
      await db.end();
      console.log('[App] Database connections closed.');
      
      process.exit(0);
    } catch (error) {
      console.error('[App] Error during shutdown:', error);
      process.exit(1);
    }
  };
  
  process.on('SIGTERM', () => shutdown('SIGTERM'));
  process.on('SIGINT', () => shutdown('SIGINT'));
  
  process.on('uncaughtException', (error) => {
    console.error('[App] Uncaught Exception:', error);
    shutdown('uncaughtException');
  });
  
  process.on('unhandledRejection', (reason) => {
    console.error('[App] Unhandled Rejection:', reason);
    shutdown('unhandledRejection');
  });
}
```

---

## 11. Full Working Example: User Management API

### Database Schema
```sql
-- migrations/001_create_users.sql
CREATE TABLE IF NOT EXISTS users (
  id          SERIAL PRIMARY KEY,
  username    VARCHAR(50) UNIQUE NOT NULL,
  email       VARCHAR(255) UNIQUE NOT NULL,
  password_hash TEXT NOT NULL,
  role        VARCHAR(20) NOT NULL DEFAULT 'user'
                CHECK (role IN ('admin', 'user', 'moderator')),
  is_active   BOOLEAN NOT NULL DEFAULT true,
  created_at  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  updated_at  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_is_active ON users(is_active);

-- Auto-update updated_at
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER users_updated_at
  BEFORE UPDATE ON users
  FOR EACH ROW EXECUTE FUNCTION update_updated_at();
```

### Repository Layer
```typescript
// src/repositories/user.repository.ts
import { db } from '../db/pool';
import { withTransaction } from '../db/transactions';
import { UserRow, UserPublic } from '../types/user';
import { handleDatabaseError } from '../utils/db-errors';
import { PoolClient } from 'pg';

export interface CreateUserData {
  username: string;
  email: string;
  passwordHash: string;
  role?: 'admin' | 'user' | 'moderator';
}

export interface UpdateUserData {
  username?: string;
  email?: string;
  role?: 'admin' | 'user' | 'moderator';
  isActive?: boolean;
}

export interface UserListOptions {
  page: number;
  limit: number;
  sortBy?: 'created_at' | 'username' | 'email';
  sortOrder?: 'ASC' | 'DESC';
  role?: string;
  isActive?: boolean;
  search?: string;
}

export interface PaginatedUsers {
  users: UserPublic[];
  total: number;
  page: number;
  limit: number;
  totalPages: number;
}

export class UserRepository {
  async findById(id: number): Promise<UserRow | null> {
    try {
      const result = await db.query<UserRow>(
        'SELECT * FROM users WHERE id = $1',
        [id]
      );
      return result.rows[0] || null;
    } catch (error) {
      handleDatabaseError(error);
    }
  }

  async findByEmail(email: string): Promise<UserRow | null> {
    try {
      const result = await db.query<UserRow>(
        'SELECT * FROM users WHERE email = $1',
        [email]
      );
      return result.rows[0] || null;
    } catch (error) {
      handleDatabaseError(error);
    }
  }

  async findByUsername(username: string): Promise<UserRow | null> {
    try {
      const result = await db.query<UserRow>(
        'SELECT * FROM users WHERE username = $1',
        [username]
      );
      return result.rows[0] || null;
    } catch (error) {
      handleDatabaseError(error);
    }
  }

  async list(options: UserListOptions): Promise<PaginatedUsers> {
    const {
      page,
      limit,
      sortBy = 'created_at',
      sortOrder = 'DESC',
      role,
      isActive,
      search,
    } = options;

    const offset = (page - 1) * limit;
    const params: unknown[] = [];
    const conditions: string[] = [];

    // Build WHERE conditions
    if (role !== undefined) {
      params.push(role);
      conditions.push(`role = $${params.length}`);
    }

    if (isActive !== undefined) {
      params.push(isActive);
      conditions.push(`is_active = $${params.length}`);
    }

    if (search) {
      params.push(`%${search}%`);
      conditions.push(
        `(username ILIKE $${params.length} OR email ILIKE $${params.length})`
      );
    }

    const whereClause = conditions.length > 0
      ? `WHERE ${conditions.join(' AND ')}`
      : '';

    // Count query
    const countResult = await db.query<{ count: string }>(
      `SELECT COUNT(*) as count FROM users ${whereClause}`,
      params
    );
    const total = parseInt(countResult.rows[0].count, 10);

    // Data query
    params.push(limit, offset);
    const dataResult = await db.query<UserPublic>(
      `SELECT id, username, email, role, is_active, created_at
       FROM users
       ${whereClause}
       ORDER BY ${sortBy} ${sortOrder}
       LIMIT $${params.length - 1} OFFSET $${params.length}`,
      params
    );

    return {
      users: dataResult.rows,
      total,
      page,
      limit,
      totalPages: Math.ceil(total / limit),
    };
  }

  async create(data: CreateUserData): Promise<UserRow> {
    try {
      const result = await db.query<UserRow>(
        `INSERT INTO users (username, email, password_hash, role, created_at, updated_at)
         VALUES ($1, $2, $3, $4, NOW(), NOW())
         RETURNING *`,
        [data.username, data.email, data.passwordHash, data.role || 'user']
      );
      return result.rows[0];
    } catch (error) {
      handleDatabaseError(error);
    }
  }

  async update(id: number, data: UpdateUserData): Promise<UserRow | null> {
    try {
      const setClauses: string[] = [];
      const params: unknown[] = [id];

      if (data.username !== undefined) {
        params.push(data.username);
        setClauses.push(`username = $${params.length}`);
      }

      if (data.email !== undefined) {
        params.push(data.email);
        setClauses.push(`email = $${params.length}`);
      }

      if (data.role !== undefined) {
        params.push(data.role);
        setClauses.push(`role = $${params.length}`);
      }

      if (data.isActive !== undefined) {
        params.push(data.isActive);
        setClauses.push(`is_active = $${params.length}`);
      }

      if (setClauses.length === 0) return null;

      const result = await db.query<UserRow>(
        `UPDATE users
         SET ${setClauses.join(', ')}, updated_at = NOW()
         WHERE id = $1
         RETURNING *`,
        params
      );

      return result.rows[0] || null;
    } catch (error) {
      handleDatabaseError(error);
    }
  }

  async delete(id: number): Promise<boolean> {
    try {
      const result = await db.query(
        'DELETE FROM users WHERE id = $1 RETURNING id',
        [id]
      );
      return (result.rowCount || 0) > 0;
    } catch (error) {
      handleDatabaseError(error);
    }
  }

  async softDelete(id: number): Promise<boolean> {
    try {
      const result = await db.query(
        `UPDATE users SET is_active = false, updated_at = NOW()
         WHERE id = $1 AND is_active = true
         RETURNING id`,
        [id]
      );
      return (result.rowCount || 0) > 0;
    } catch (error) {
      handleDatabaseError(error);
    }
  }
}

export const userRepository = new UserRepository();
```

### Service Layer
```typescript
// src/services/user.service.ts
import bcrypt from 'bcryptjs';
import { userRepository, CreateUserData, UpdateUserData, UserListOptions } from '../repositories/user.repository';
import { UserRow, UserPublic } from '../types/user';

export class UserService {
  async getUser(id: number): Promise<UserPublic | null> {
    const user = await userRepository.findById(id);
    if (!user) return null;
    return this.toPublic(user);
  }

  async getUserList(options: UserListOptions) {
    return userRepository.list(options);
  }

  async createUser(data: {
    username: string;
    email: string;
    password: string;
    role?: 'admin' | 'user' | 'moderator';
  }): Promise<UserPublic> {
    // Check if email already exists
    const existingEmail = await userRepository.findByEmail(data.email);
    if (existingEmail) {
      throw new Error('Email already in use');
    }

    // Check if username already exists
    const existingUsername = await userRepository.findByUsername(data.username);
    if (existingUsername) {
      throw new Error('Username already in use');
    }

    // Hash password
    const passwordHash = await bcrypt.hash(data.password, 12);

    const user = await userRepository.create({
      username: data.username,
      email: data.email,
      passwordHash,
      role: data.role,
    });

    return this.toPublic(user);
  }

  async updateUser(id: number, data: UpdateUserData): Promise<UserPublic | null> {
    // Check email uniqueness if updating email
    if (data.email) {
      const existing = await userRepository.findByEmail(data.email);
      if (existing && existing.id !== id) {
        throw new Error('Email already in use');
      }
    }

    const user = await userRepository.update(id, data);
    if (!user) return null;
    return this.toPublic(user);
  }

  async deleteUser(id: number): Promise<boolean> {
    return userRepository.softDelete(id);
  }

  async verifyPassword(email: string, password: string): Promise<UserPublic | null> {
    const user = await userRepository.findByEmail(email);
    if (!user || !user.is_active) return null;

    const isValid = await bcrypt.compare(password, user.password_hash);
    if (!isValid) return null;

    return this.toPublic(user);
  }

  private toPublic(user: UserRow): UserPublic {
    return {
      id: user.id,
      username: user.username,
      email: user.email,
      role: user.role,
      created_at: user.created_at,
    };
  }
}

export const userService = new UserService();
```

### Controller Layer
```typescript
// src/controllers/user.controller.ts
import { Request, Response, NextFunction } from 'express';
import { userService } from '../services/user.service';

export class UserController {
  async list(req: Request, res: Response, next: NextFunction): Promise<void> {
    try {
      const page = parseInt(req.query.page as string || '1', 10);
      const limit = Math.min(parseInt(req.query.limit as string || '20', 10), 100);
      const result = await userService.getUserList({
        page,
        limit,
        sortBy: req.query.sortBy as any,
        sortOrder: req.query.sortOrder as any,
        role: req.query.role as string,
        isActive: req.query.isActive !== undefined
          ? req.query.isActive === 'true'
          : undefined,
        search: req.query.search as string,
      });

      res.json({
        success: true,
        data: result,
      });
    } catch (error) {
      next(error);
    }
  }

  async getById(req: Request, res: Response, next: NextFunction): Promise<void> {
    try {
      const id = parseInt(req.params.id, 10);
      const user = await userService.getUser(id);

      if (!user) {
        res.status(404).json({ success: false, error: 'User not found' });
        return;
      }

      res.json({ success: true, data: user });
    } catch (error) {
      next(error);
    }
  }

  async create(req: Request, res: Response, next: NextFunction): Promise<void> {
    try {
      const user = await userService.createUser(req.body);
      res.status(201).json({ success: true, data: user });
    } catch (error) {
      next(error);
    }
  }

  async update(req: Request, res: Response, next: NextFunction): Promise<void> {
    try {
      const id = parseInt(req.params.id, 10);
      const user = await userService.updateUser(id, req.body);

      if (!user) {
        res.status(404).json({ success: false, error: 'User not found' });
        return;
      }

      res.json({ success: true, data: user });
    } catch (error) {
      next(error);
    }
  }

  async delete(req: Request, res: Response, next: NextFunction): Promise<void> {
    try {
      const id = parseInt(req.params.id, 10);
      const deleted = await userService.deleteUser(id);

      if (!deleted) {
        res.status(404).json({ success: false, error: 'User not found' });
        return;
      }

      res.status(204).send();
    } catch (error) {
      next(error);
    }
  }
}

export const userController = new UserController();
```

---

## 12. Testing Database Code ด้วย Jest

```typescript
// src/__tests__/user.repository.test.ts
import { Pool } from 'pg';
import { userRepository } from '../repositories/user.repository';
import { db } from '../db/pool';

// Mock the pool
jest.mock('../db/pool', () => ({
  db: {
    query: jest.fn(),
    getClient: jest.fn(),
    getStats: jest.fn().mockReturnValue({ total: 5, idle: 3, waiting: 0 }),
    end: jest.fn(),
  },
}));

describe('UserRepository', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  describe('findById', () => {
    it('should return user when found', async () => {
      const mockUser = {
        id: 1,
        username: 'testuser',
        email: 'test@example.com',
        role: 'user',
        is_active: true,
        created_at: new Date(),
        updated_at: new Date(),
      };

      (db.query as jest.Mock).mockResolvedValueOnce({
        rows: [mockUser],
        rowCount: 1,
      });

      const user = await userRepository.findById(1);
      
      expect(user).toEqual(mockUser);
      expect(db.query).toHaveBeenCalledWith(
        'SELECT * FROM users WHERE id = $1',
        [1]
      );
    });

    it('should return null when user not found', async () => {
      (db.query as jest.Mock).mockResolvedValueOnce({
        rows: [],
        rowCount: 0,
      });

      const user = await userRepository.findById(999);
      expect(user).toBeNull();
    });
  });

  describe('create', () => {
    it('should create user successfully', async () => {
      const mockUser = {
        id: 1,
        username: 'newuser',
        email: 'new@example.com',
        role: 'user',
        is_active: true,
        created_at: new Date(),
        updated_at: new Date(),
      };

      (db.query as jest.Mock).mockResolvedValueOnce({
        rows: [mockUser],
        rowCount: 1,
      });

      const user = await userRepository.create({
        username: 'newuser',
        email: 'new@example.com',
        passwordHash: 'hashed_password',
      });

      expect(user).toEqual(mockUser);
    });

    it('should throw error on duplicate email', async () => {
      const pgError = new Error('duplicate key value') as Error & { code: string };
      pgError.code = '23505';

      (db.query as jest.Mock).mockRejectedValueOnce(pgError);

      await expect(
        userRepository.create({
          username: 'user2',
          email: 'existing@example.com',
          passwordHash: 'hash',
        })
      ).rejects.toThrow('Duplicate entry');
    });
  });
});
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:
1. การติดตั้งและ configure `pg` module
2. การสร้าง Connection Pool ที่มีประสิทธิภาพ
3. การทำ parameterized queries เพื่อป้องกัน SQL injection
4. การจัดการ TypeScript types
5. การ handle errors จาก PostgreSQL
6. การทำ transactions
7. การ stream ข้อมูลขนาดใหญ่
8. การทำ bulk insert ด้วย COPY command
9. Health check และ graceful shutdown
10. Full CRUD implementation พร้อม Repository pattern

ในบทต่อไปเราจะเรียนรู้การเชื่อมต่อกับ Redis ด้วย ioredis

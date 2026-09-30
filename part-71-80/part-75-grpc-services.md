# Part 75: gRPC Services กับ Database Cluster

## บทนำ

**gRPC** (Google Remote Procedure Call) เป็น High-Performance RPC Framework ที่พัฒนาโดย Google ใช้ **Protocol Buffers** เป็น Interface Definition Language (IDL) และ Serialization Format ทำให้ส่งข้อมูลได้เร็วกว่า JSON มาก

ในบทนี้เราจะเรียนรู้การสร้าง gRPC Services ที่เชื่อมต่อกับ PostgreSQL Database Cluster และ Redis Caching

---

## 1. gRPC vs REST vs GraphQL

### 1.1 เปรียบเทียบเชิงเทคนิค

| Feature | REST | GraphQL | gRPC |
|---------|------|---------|------|
| Protocol | HTTP/1.1 | HTTP/1.1 or 2 | HTTP/2 |
| Format | JSON/XML | JSON | Protocol Buffers (Binary) |
| Schema | OpenAPI | SDL | .proto |
| Streaming | SSE/WebSocket | Subscriptions | Native (4 types) |
| Performance | Good | Good | Excellent |
| Code Generation | Manual | Limited | Full Generate |
| Browser Support | Native | Native | Needs gRPC-Web |
| Learning Curve | Low | Medium | Medium-High |

### 1.2 เมื่อไหร่ใช้ gRPC

✅ **ใช้ gRPC เมื่อ:**
- Internal Microservice-to-Microservice Communication
- ต้องการ Performance สูงสุด (Low Latency, High Throughput)
- ต้องการ Streaming (Real-time Data)
- มี Strong Typed Contract ระหว่าง Services
- Polyglot Environment (หลายภาษา: Go, Python, Java, Node.js)

❌ **ไม่ควรใช้ gRPC เมื่อ:**
- Public API สำหรับ Third-party Developers
- Browser-facing API (ต้องการ gRPC-Web wrapper)
- Simple CRUD ที่ไม่ต้องการ Performance สูง

---

## 2. Protocol Buffers (protobuf): ทำความเข้าใจ

### 2.1 protobuf คืออะไร

Protocol Buffers เป็น Binary Serialization Format ที่:
- **Compact**: ขนาดเล็กกว่า JSON 3-10x
- **Fast**: Serialize/Deserialize เร็วกว่า JSON 5-10x
- **Strongly Typed**: Type Safety ตั้งแต่ Compile Time
- **Backward/Forward Compatible**: Fields ใหม่ไม่ทำให้ Old Code พัง

### 2.2 ตัวอย่าง: JSON vs protobuf

```json
// JSON: ~100 bytes
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "name": "John Doe",
  "email": "john@example.com",
  "age": 30
}
```

```protobuf
// protobuf definition
message User {
  string id = 1;
  string name = 2;
  string email = 3;
  int32 age = 4;
}
// Binary: ~50 bytes (เพราะ field numbers แทน field names)
```

---

## 3. Project Setup

### 3.1 Dependencies

```bash
mkdir grpc-user-service && cd grpc-user-service
npm init -y

# Core gRPC
npm install @grpc/grpc-js @grpc/proto-loader

# TypeScript Support
npm install -D typescript ts-node @types/node
npm install -D @protobuf-ts/plugin @protobuf-ts/runtime @protobuf-ts/grpc-transport

# Database
npm install pg ioredis
npm install @types/pg

# Utilities
npm install uuid zod
npm install -D @types/uuid

# Install protoc (Protocol Buffer Compiler)
# macOS: brew install protobuf
# Ubuntu: apt-get install -y protobuf-compiler
# Windows: choco install protoc
```

### 3.2 tsconfig.json

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
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

---

## 4. Proto File Definitions

### 4.1 User Service Proto

```protobuf
// proto/user.proto

syntax = "proto3";

package user.v1;

import "google/protobuf/timestamp.proto";
import "google/protobuf/empty.proto";

option java_multiple_files = true;
option java_package = "com.example.user.v1";
option go_package = "github.com/example/grpc/user/v1;userv1";

// ─── Enums ────────────────────────────────────────────────
enum UserRole {
  USER_ROLE_UNSPECIFIED = 0;
  USER_ROLE_USER = 1;
  USER_ROLE_ADMIN = 2;
  USER_ROLE_MODERATOR = 3;
}

enum UserStatus {
  USER_STATUS_UNSPECIFIED = 0;
  USER_STATUS_ACTIVE = 1;
  USER_STATUS_INACTIVE = 2;
  USER_STATUS_BANNED = 3;
}

// ─── Messages ──────────────────────────────────────────────
message User {
  string id = 1;
  string email = 2;
  string name = 3;
  UserRole role = 4;
  UserStatus status = 5;
  google.protobuf.Timestamp created_at = 6;
  google.protobuf.Timestamp updated_at = 7;
  
  // Optional fields
  optional string phone = 8;
  optional string avatar_url = 9;
  map<string, string> metadata = 10;
}

message Address {
  string street = 1;
  string city = 2;
  string state = 3;
  string country = 4;
  string postal_code = 5;
}

// ─── Request/Response ──────────────────────────────────────
message GetUserRequest {
  string id = 1;
}

message GetUserResponse {
  User user = 1;
}

message CreateUserRequest {
  string email = 1;
  string name = 2;
  string password = 3;
  optional UserRole role = 4;
}

message CreateUserResponse {
  User user = 1;
}

message UpdateUserRequest {
  string id = 1;
  optional string name = 2;
  optional string email = 3;
  optional string phone = 4;
  optional UserRole role = 5;
  optional UserStatus status = 6;
}

message UpdateUserResponse {
  User user = 1;
}

message DeleteUserRequest {
  string id = 1;
}

message DeleteUserResponse {
  bool success = 1;
}

message ListUsersRequest {
  int32 page = 1;
  int32 page_size = 2;
  optional string search = 3;
  optional UserRole role = 4;
  optional UserStatus status = 5;
  string order_by = 6;   // e.g., "created_at desc"
}

message ListUsersResponse {
  repeated User users = 1;
  int32 total_count = 2;
  int32 page = 3;
  int32 page_size = 4;
  bool has_next_page = 5;
}

message StreamUsersRequest {
  optional UserRole role = 1;
  optional UserStatus status = 2;
  int32 batch_size = 3;   // How many per stream message
}

message StreamUsersResponse {
  repeated User users = 1;
  bool is_last_batch = 2;
}

message BulkCreateUsersRequest {
  repeated CreateUserRequest users = 1;
}

message BulkCreateUsersProgress {
  int32 processed = 1;
  int32 total = 2;
  optional string last_created_id = 3;
  optional string error = 4;
}

// ─── Service Definition ────────────────────────────────────
service UserService {
  // Unary RPC: Single request, single response
  rpc GetUser (GetUserRequest) returns (GetUserResponse);
  rpc CreateUser (CreateUserRequest) returns (CreateUserResponse);
  rpc UpdateUser (UpdateUserRequest) returns (UpdateUserResponse);
  rpc DeleteUser (DeleteUserRequest) returns (DeleteUserResponse);
  rpc ListUsers (ListUsersRequest) returns (ListUsersResponse);
  
  // Server Streaming: One request, stream of responses
  rpc StreamUsers (StreamUsersRequest) returns (stream StreamUsersResponse);
  
  // Client Streaming: Stream of requests, one response
  rpc BulkCreateUsers (stream BulkCreateUsersRequest) returns (BulkCreateUsersProgress);
  
  // Bidirectional Streaming: Both sides stream
  rpc SyncUsers (stream SyncRequest) returns (stream SyncResponse);
}

message SyncRequest {
  string client_id = 1;
  int64 last_sync_timestamp = 2;
  repeated string known_user_ids = 3;
}

message SyncResponse {
  repeated User new_or_updated = 1;
  repeated string deleted_ids = 2;
  int64 sync_timestamp = 3;
}
```

### 4.2 Generate TypeScript Types

```bash
# สร้าง TypeScript code จาก .proto file
npx protoc \
  --plugin=./node_modules/.bin/protoc-gen-ts_proto \
  --ts_proto_out=./src/generated \
  --ts_proto_opt=esModuleInterop=true \
  --ts_proto_opt=outputServices=grpc-js \
  --ts_proto_opt=useOptionals=messages \
  -I ./proto \
  ./proto/user.proto

# หรือใช้ @grpc/proto-loader (Dynamic Loading - ไม่ต้องรัน protoc)
```

---

## 5. gRPC Server Implementation

### 5.1 Database Connection

```typescript
// src/database/pool.ts
import { Pool } from 'pg';
import Redis from 'ioredis';

export const pgPool = new Pool({
  host: process.env.PG_HOST || 'localhost',
  port: parseInt(process.env.PG_PORT || '5432'),
  database: process.env.PG_DATABASE || 'grpc_db',
  user: process.env.PG_USER || 'postgres',
  password: process.env.PG_PASSWORD || 'postgres123',
  max: 20,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});

export const pgReadPool = new Pool({
  host: process.env.PG_READ_HOST || process.env.PG_HOST || 'localhost',
  port: parseInt(process.env.PG_PORT || '5432'),
  database: process.env.PG_DATABASE || 'grpc_db',
  user: process.env.PG_USER || 'postgres',
  password: process.env.PG_PASSWORD || 'postgres123',
  max: 30,
});

export const redisClient = new Redis({
  host: process.env.REDIS_HOST || 'localhost',
  port: parseInt(process.env.REDIS_PORT || '6379'),
  maxRetriesPerRequest: 3,
  enableAutoPipelining: true,
});
```

### 5.2 User Service Implementation

```typescript
// src/services/user.service.ts
import {
  ServerUnaryCall,
  ServerWritableStream,
  ServerReadableStream,
  ServerDuplexStream,
  sendUnaryData,
  status,
  ServiceError,
} from '@grpc/grpc-js';
import { Pool } from 'pg';
import Redis from 'ioredis';
import { hashPassword, comparePassword } from '../utils/auth';
import { v4 as uuidv4 } from 'uuid';

// Types (from generated code or manual)
interface User {
  id: string;
  email: string;
  name: string;
  role: string;
  status: string;
  phone?: string;
  avatarUrl?: string;
  createdAt: Date;
  updatedAt: Date;
}

export class UserServiceImpl {
  constructor(
    private pgWrite: Pool,
    private pgRead: Pool,
    private redis: Redis,
  ) {}

  // ─── Unary: GetUser ────────────────────────────────
  async getUser(
    call: ServerUnaryCall<any, any>,
    callback: sendUnaryData<any>,
  ): Promise<void> {
    const { id } = call.request;

    if (!id) {
      return callback({
        code: status.INVALID_ARGUMENT,
        message: 'User ID is required',
      } as ServiceError);
    }

    try {
      // Check Redis cache first
      const cacheKey = `user:${id}`;
      const cached = await this.redis.get(cacheKey);
      if (cached) {
        return callback(null, { user: JSON.parse(cached) });
      }

      // Query from read replica
      const { rows } = await this.pgRead.query(
        `SELECT id, email, name, role, status, phone, avatar_url,
                created_at, updated_at
         FROM users WHERE id = $1`,
        [id],
      );

      if (rows.length === 0) {
        return callback({
          code: status.NOT_FOUND,
          message: `User ${id} not found`,
        } as ServiceError);
      }

      const user = this.mapRowToUser(rows[0]);

      // Cache for 5 minutes
      await this.redis.setex(cacheKey, 300, JSON.stringify(user));

      callback(null, { user });
    } catch (error: any) {
      console.error('GetUser error:', error);
      callback({
        code: status.INTERNAL,
        message: 'Internal server error',
        details: process.env.NODE_ENV === 'development' ? error.message : undefined,
      } as ServiceError);
    }
  }

  // ─── Unary: CreateUser ─────────────────────────────
  async createUser(
    call: ServerUnaryCall<any, any>,
    callback: sendUnaryData<any>,
  ): Promise<void> {
    const { email, name, password, role } = call.request;

    // Validation
    if (!email || !name || !password) {
      return callback({
        code: status.INVALID_ARGUMENT,
        message: 'email, name, and password are required',
      } as ServiceError);
    }

    if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) {
      return callback({
        code: status.INVALID_ARGUMENT,
        message: 'Invalid email format',
      } as ServiceError);
    }

    try {
      // Check duplicate email
      const { rows: existing } = await this.pgRead.query(
        'SELECT id FROM users WHERE email = $1',
        [email],
      );

      if (existing.length > 0) {
        return callback({
          code: status.ALREADY_EXISTS,
          message: `User with email ${email} already exists`,
        } as ServiceError);
      }

      const hashedPassword = await hashPassword(password);
      const userId = uuidv4();

      const { rows } = await this.pgWrite.query(
        `INSERT INTO users (id, email, name, password_hash, role, status)
         VALUES ($1, $2, $3, $4, $5, 'ACTIVE')
         RETURNING id, email, name, role, status, created_at, updated_at`,
        [userId, email, name, hashedPassword, role || 'USER'],
      );

      const user = this.mapRowToUser(rows[0]);
      callback(null, { user });

    } catch (error: any) {
      if (error.code === '23505') { // Unique violation
        return callback({
          code: status.ALREADY_EXISTS,
          message: 'Email already in use',
        } as ServiceError);
      }
      callback({
        code: status.INTERNAL,
        message: 'Failed to create user',
      } as ServiceError);
    }
  }

  // ─── Unary: UpdateUser ─────────────────────────────
  async updateUser(
    call: ServerUnaryCall<any, any>,
    callback: sendUnaryData<any>,
  ): Promise<void> {
    const { id, name, email, phone, role, status: userStatus } = call.request;

    // Build dynamic update query
    const updates: string[] = [];
    const values: any[] = [];
    let paramCount = 1;

    if (name !== undefined) {
      updates.push(`name = $${paramCount++}`);
      values.push(name);
    }
    if (email !== undefined) {
      updates.push(`email = $${paramCount++}`);
      values.push(email);
    }
    if (phone !== undefined) {
      updates.push(`phone = $${paramCount++}`);
      values.push(phone);
    }
    if (role !== undefined) {
      updates.push(`role = $${paramCount++}`);
      values.push(role);
    }
    if (userStatus !== undefined) {
      updates.push(`status = $${paramCount++}`);
      values.push(userStatus);
    }

    if (updates.length === 0) {
      return callback({
        code: status.INVALID_ARGUMENT,
        message: 'No fields to update',
      } as ServiceError);
    }

    updates.push(`updated_at = NOW()`);
    values.push(id);

    try {
      const { rows } = await this.pgWrite.query(
        `UPDATE users SET ${updates.join(', ')}
         WHERE id = $${paramCount}
         RETURNING id, email, name, role, status, phone, avatar_url, created_at, updated_at`,
        values,
      );

      if (rows.length === 0) {
        return callback({
          code: status.NOT_FOUND,
          message: `User ${id} not found`,
        } as ServiceError);
      }

      const user = this.mapRowToUser(rows[0]);

      // Invalidate cache
      await this.redis.del(`user:${id}`);

      callback(null, { user });
    } catch (error: any) {
      callback({ code: status.INTERNAL, message: 'Update failed' } as ServiceError);
    }
  }

  // ─── Unary: ListUsers ──────────────────────────────
  async listUsers(
    call: ServerUnaryCall<any, any>,
    callback: sendUnaryData<any>,
  ): Promise<void> {
    const {
      page = 1,
      pageSize = 20,
      search,
      role,
      status: userStatus,
      orderBy = 'created_at desc',
    } = call.request;

    const offset = (page - 1) * pageSize;
    const conditions: string[] = [];
    const values: any[] = [];
    let paramCount = 1;

    if (search) {
      conditions.push(`(name ILIKE $${paramCount} OR email ILIKE $${paramCount})`);
      values.push(`%${search}%`);
      paramCount++;
    }
    if (role && role !== 'USER_ROLE_UNSPECIFIED') {
      conditions.push(`role = $${paramCount++}`);
      values.push(role.replace('USER_ROLE_', ''));
    }
    if (userStatus && userStatus !== 'USER_STATUS_UNSPECIFIED') {
      conditions.push(`status = $${paramCount++}`);
      values.push(userStatus.replace('USER_STATUS_', ''));
    }

    const where = conditions.length > 0 ? `WHERE ${conditions.join(' AND ')}` : '';
    
    // Validate orderBy to prevent SQL injection
    const allowedOrderBy = ['created_at asc', 'created_at desc', 'name asc', 'name desc', 'email asc'];
    const safeOrderBy = allowedOrderBy.includes(orderBy) ? orderBy : 'created_at desc';

    try {
      const [{ rows }, { rows: countRows }] = await Promise.all([
        this.pgRead.query(
          `SELECT id, email, name, role, status, phone, avatar_url, created_at, updated_at
           FROM users ${where}
           ORDER BY ${safeOrderBy}
           LIMIT $${paramCount} OFFSET $${paramCount + 1}`,
          [...values, pageSize + 1, offset], // +1 to check hasNextPage
        ),
        this.pgRead.query(
          `SELECT COUNT(*) as count FROM users ${where}`,
          values,
        ),
      ]);

      const hasNextPage = rows.length > pageSize;
      const users = rows.slice(0, pageSize).map((row) => this.mapRowToUser(row));

      callback(null, {
        users,
        totalCount: parseInt(countRows[0].count),
        page,
        pageSize,
        hasNextPage,
      });
    } catch (error: any) {
      callback({ code: status.INTERNAL, message: 'List failed' } as ServiceError);
    }
  }

  // ─── Server Streaming: StreamUsers ─────────────────
  async streamUsers(
    call: ServerWritableStream<any, any>,
  ): Promise<void> {
    const { role, status: userStatus, batchSize = 100 } = call.request;

    const conditions: string[] = [];
    const values: any[] = [];
    let paramCount = 1;

    if (role && role !== 'USER_ROLE_UNSPECIFIED') {
      conditions.push(`role = $${paramCount++}`);
      values.push(role.replace('USER_ROLE_', ''));
    }
    if (userStatus && userStatus !== 'USER_STATUS_UNSPECIFIED') {
      conditions.push(`status = $${paramCount++}`);
      values.push(userStatus.replace('USER_STATUS_', ''));
    }

    const where = conditions.length > 0 ? `WHERE ${conditions.join(' AND ')}` : '';

    try {
      // Get total count first
      const { rows: countRows } = await this.pgRead.query(
        `SELECT COUNT(*) as count FROM users ${where}`,
        values,
      );
      const totalCount = parseInt(countRows[0].count);

      // Stream in batches
      let offset = 0;
      while (offset < totalCount) {
        if (call.cancelled) break;

        const { rows } = await this.pgRead.query(
          `SELECT id, email, name, role, status, phone, avatar_url, created_at, updated_at
           FROM users ${where}
           ORDER BY created_at ASC
           LIMIT $${paramCount} OFFSET $${paramCount + 1}`,
          [...values, batchSize, offset],
        );

        const users = rows.map((row) => this.mapRowToUser(row));
        const isLastBatch = offset + batchSize >= totalCount;

        call.write({ users, isLastBatch });

        offset += batchSize;

        // Small delay to prevent overwhelming the client
        if (!isLastBatch) {
          await new Promise((resolve) => setTimeout(resolve, 10));
        }
      }

      call.end();
    } catch (error: any) {
      call.destroy({
        code: status.INTERNAL,
        message: 'Streaming failed',
      } as any);
    }
  }

  // ─── Client Streaming: BulkCreateUsers ─────────────
  async bulkCreateUsers(
    call: ServerReadableStream<any, any>,
    callback: sendUnaryData<any>,
  ): Promise<void> {
    const allUsers: any[] = [];
    let processed = 0;
    let total = 0;

    // Collect all streamed requests
    await new Promise<void>((resolve, reject) => {
      call.on('data', (request: any) => {
        allUsers.push(...request.users);
        total = allUsers.length;
      });
      call.on('end', resolve);
      call.on('error', reject);
    });

    const client = await this.pgWrite.connect();

    try {
      await client.query('BEGIN');

      for (const userData of allUsers) {
        const hashedPassword = await hashPassword(userData.password);
        await client.query(
          `INSERT INTO users (id, email, name, password_hash, role, status)
           VALUES ($1, $2, $3, $4, $5, 'ACTIVE')
           ON CONFLICT (email) DO NOTHING`,
          [uuidv4(), userData.email, userData.name, hashedPassword, userData.role || 'USER'],
        );
        processed++;
      }

      await client.query('COMMIT');
      callback(null, { processed, total, error: '' });

    } catch (error: any) {
      await client.query('ROLLBACK');
      callback(null, {
        processed,
        total,
        error: error.message,
      });
    } finally {
      client.release();
    }
  }

  // ─── Helper ────────────────────────────────────────
  private mapRowToUser(row: any): User {
    return {
      id: row.id,
      email: row.email,
      name: row.name,
      role: `USER_ROLE_${row.role}`,
      status: `USER_STATUS_${row.status}`,
      phone: row.phone || undefined,
      avatarUrl: row.avatar_url || undefined,
      createdAt: row.created_at,
      updatedAt: row.updated_at,
    };
  }
}
```

---

## 6. Server Startup

```typescript
// src/server.ts
import * as grpc from '@grpc/grpc-js';
import * as protoLoader from '@grpc/proto-loader';
import path from 'path';
import { pgPool, pgReadPool, redisClient } from './database/pool';
import { UserServiceImpl } from './services/user.service';
import { loggingInterceptor, authInterceptor } from './interceptors';

const PROTO_PATH = path.join(__dirname, '../proto/user.proto');

// Load proto definition
const packageDefinition = protoLoader.loadSync(PROTO_PATH, {
  keepCase: true,
  longs: String,
  enums: String,
  defaults: true,
  oneofs: true,
});

const userProto = grpc.loadPackageDefinition(packageDefinition) as any;

async function main() {
  const userService = new UserServiceImpl(pgPool, pgReadPool, redisClient);

  const server = new grpc.Server({
    'grpc.max_send_message_length': 10 * 1024 * 1024,   // 10MB
    'grpc.max_receive_message_length': 10 * 1024 * 1024, // 10MB
    'grpc.keepalive_time_ms': 10000,
    'grpc.keepalive_timeout_ms': 5000,
    'grpc.keepalive_permit_without_calls': 1,
  });

  // Add service implementation
  server.addService(userProto.user.v1.UserService.service, {
    GetUser: (call: any, cb: any) => userService.getUser(call, cb),
    CreateUser: (call: any, cb: any) => userService.createUser(call, cb),
    UpdateUser: (call: any, cb: any) => userService.updateUser(call, cb),
    DeleteUser: (call: any, cb: any) => userService.deleteUser(call, cb),
    ListUsers: (call: any, cb: any) => userService.listUsers(call, cb),
    StreamUsers: (call: any) => userService.streamUsers(call),
    BulkCreateUsers: (call: any, cb: any) => userService.bulkCreateUsers(call, cb),
  });

  const port = process.env.GRPC_PORT || '50051';
  const host = process.env.GRPC_HOST || '0.0.0.0';

  return new Promise<void>((resolve, reject) => {
    server.bindAsync(
      `${host}:${port}`,
      grpc.ServerCredentials.createInsecure(), // In production: use TLS
      (err, boundPort) => {
        if (err) return reject(err);
        console.log(`gRPC User Service listening on ${host}:${boundPort}`);
        resolve();
      },
    );
  });
}

main().catch(console.error);
```

---

## 7. Interceptors (Logging, Auth, Retry)

### 7.1 Server-Side Interceptors

```typescript
// src/interceptors/logging.interceptor.ts
import * as grpc from '@grpc/grpc-js';

export function createLoggingInterceptor() {
  return function loggingInterceptor(
    methodDescriptor: grpc.MethodDefinition<any, any>,
    nextCall: Function,
  ) {
    const startTime = Date.now();
    const method = methodDescriptor.path;

    return new grpc.InterceptingCall(nextCall(methodDescriptor), {
      start: function (metadata, listener, next) {
        console.log(`gRPC ${method} - START`, {
          metadata: metadata.getMap(),
          timestamp: new Date().toISOString(),
        });

        next(metadata, {
          onReceiveMessage: function (message, next) {
            next(message);
          },
          onReceiveStatus: function (status, next) {
            const duration = Date.now() - startTime;
            console.log(`gRPC ${method} - END`, {
              code: status.code,
              details: status.details,
              durationMs: duration,
            });
            next(status);
          },
        });
      },
    });
  };
}

// src/interceptors/auth.interceptor.ts
export function createAuthInterceptor(jwtSecret: string) {
  return function authInterceptor(
    methodDescriptor: grpc.MethodDefinition<any, any>,
    nextCall: Function,
  ) {
    // Methods that don't require authentication
    const publicMethods = ['/user.v1.UserService/CreateUser'];

    return new grpc.InterceptingCall(nextCall(methodDescriptor), {
      start: function (metadata, listener, next) {
        const method = methodDescriptor.path;

        if (publicMethods.includes(method)) {
          return next(metadata, listener);
        }

        const authHeader = metadata.get('authorization')[0] as string;

        if (!authHeader || !authHeader.startsWith('Bearer ')) {
          const err = new Error('Missing authentication token');
          (err as any).code = grpc.status.UNAUTHENTICATED;
          listener.onReceiveStatus({
            code: grpc.status.UNAUTHENTICATED,
            details: 'Missing authentication token',
            metadata: new grpc.Metadata(),
          });
          return;
        }

        try {
          const token = authHeader.replace('Bearer ', '');
          const payload = verifyJWT(token, jwtSecret);
          metadata.set('x-user-id', payload.userId);
          metadata.set('x-user-role', payload.role);
          next(metadata, listener);
        } catch {
          listener.onReceiveStatus({
            code: grpc.status.UNAUTHENTICATED,
            details: 'Invalid authentication token',
            metadata: new grpc.Metadata(),
          });
        }
      },
    });
  };
}
```

---

## 8. gRPC Client

### 8.1 Client Implementation

```typescript
// src/clients/user.client.ts
import * as grpc from '@grpc/grpc-js';
import * as protoLoader from '@grpc/proto-loader';
import path from 'path';

const PROTO_PATH = path.join(__dirname, '../../proto/user.proto');

const packageDefinition = protoLoader.loadSync(PROTO_PATH, {
  keepCase: true,
  longs: String,
  enums: String,
  defaults: true,
  oneofs: true,
});

const userProto = grpc.loadPackageDefinition(packageDefinition) as any;

export class UserServiceClient {
  private client: any;

  constructor(address: string = 'localhost:50051') {
    this.client = new userProto.user.v1.UserService(
      address,
      grpc.credentials.createInsecure(),
      {
        'grpc.keepalive_time_ms': 10000,
        'grpc.keepalive_timeout_ms': 5000,
      },
    );
  }

  // ─── Unary Calls ────────────────────────────────────
  async getUser(id: string): Promise<any> {
    return new Promise((resolve, reject) => {
      const metadata = new grpc.Metadata();
      metadata.set('authorization', `Bearer ${process.env.SERVICE_TOKEN}`);

      this.client.GetUser(
        { id },
        metadata,
        (error: grpc.ServiceError | null, response: any) => {
          if (error) return reject(error);
          resolve(response.user);
        },
      );
    });
  }

  async createUser(data: {
    email: string;
    name: string;
    password: string;
    role?: string;
  }): Promise<any> {
    return new Promise((resolve, reject) => {
      this.client.CreateUser(data, (error: any, response: any) => {
        if (error) return reject(error);
        resolve(response.user);
      });
    });
  }

  async listUsers(params: {
    page?: number;
    pageSize?: number;
    search?: string;
    role?: string;
  }): Promise<any> {
    return new Promise((resolve, reject) => {
      this.client.ListUsers(
        {
          page: params.page || 1,
          page_size: params.pageSize || 20,
          search: params.search || '',
          role: params.role || 'USER_ROLE_UNSPECIFIED',
        },
        (error: any, response: any) => {
          if (error) return reject(error);
          resolve(response);
        },
      );
    });
  }

  // ─── Server Streaming ───────────────────────────────
  streamAllUsers(
    options: { role?: string } = {},
    onData: (users: any[]) => void,
    onEnd: () => void,
    onError: (error: Error) => void,
  ): void {
    const call = this.client.StreamUsers({
      role: options.role || 'USER_ROLE_UNSPECIFIED',
      batch_size: 100,
    });

    call.on('data', (response: any) => {
      onData(response.users);
    });

    call.on('end', () => {
      onEnd();
    });

    call.on('error', (error: Error) => {
      onError(error);
    });
  }

  // Helper: Stream to array
  async streamAllUsersToArray(options: { role?: string } = {}): Promise<any[]> {
    return new Promise((resolve, reject) => {
      const allUsers: any[] = [];

      this.streamAllUsers(
        options,
        (users) => allUsers.push(...users),
        () => resolve(allUsers),
        reject,
      );
    });
  }

  // ─── Client Streaming ───────────────────────────────
  async bulkCreateUsers(users: any[]): Promise<any> {
    return new Promise((resolve, reject) => {
      const call = this.client.BulkCreateUsers(
        (error: any, response: any) => {
          if (error) return reject(error);
          resolve(response);
        },
      );

      // Send users in batches of 50
      const batchSize = 50;
      for (let i = 0; i < users.length; i += batchSize) {
        call.write({ users: users.slice(i, i + batchSize) });
      }

      call.end();
    });
  }

  close(): void {
    this.client.close();
  }
}
```

---

## 9. gRPC Health Checking

```typescript
// src/health/health.service.ts
import * as grpc from '@grpc/grpc-js';

export class HealthServiceImpl {
  private serviceHealth: Map<string, string> = new Map();

  setServiceHealth(serviceName: string, status: 'SERVING' | 'NOT_SERVING' | 'UNKNOWN') {
    this.serviceHealth.set(serviceName, status);
  }

  check(call: any, callback: any): void {
    const { service } = call.request;

    const status = service
      ? (this.serviceHealth.get(service) || 'UNKNOWN')
      : 'SERVING';

    callback(null, { status });
  }

  watch(call: any): void {
    const { service } = call.request;

    // Send current status
    const status = service
      ? (this.serviceHealth.get(service) || 'UNKNOWN')
      : 'SERVING';

    call.write({ status });

    // Send updates when status changes
    const interval = setInterval(() => {
      if (call.cancelled) {
        clearInterval(interval);
        return;
      }
      const currentStatus = service
        ? (this.serviceHealth.get(service) || 'UNKNOWN')
        : 'SERVING';
      call.write({ status: currentStatus });
    }, 5000);
  }
}
```

---

## 10. Error Handling

### 10.1 gRPC Status Codes

```typescript
// src/utils/grpc-error.ts
import * as grpc from '@grpc/grpc-js';

export function createGrpcError(
  code: grpc.status,
  message: string,
  details?: any,
): grpc.ServiceError {
  const error = new Error(message) as grpc.ServiceError;
  error.code = code;
  if (details) {
    error.details = JSON.stringify(details);
  }
  return error;
}

// Common error factories
export const GrpcErrors = {
  NotFound: (resource: string, id: string) =>
    createGrpcError(grpc.status.NOT_FOUND, `${resource} ${id} not found`),

  AlreadyExists: (resource: string, field: string) =>
    createGrpcError(grpc.status.ALREADY_EXISTS, `${resource} with this ${field} already exists`),

  InvalidArgument: (message: string) =>
    createGrpcError(grpc.status.INVALID_ARGUMENT, message),

  Unauthenticated: () =>
    createGrpcError(grpc.status.UNAUTHENTICATED, 'Authentication required'),

  PermissionDenied: () =>
    createGrpcError(grpc.status.PERMISSION_DENIED, 'Insufficient permissions'),

  Internal: (message = 'Internal server error') =>
    createGrpcError(grpc.status.INTERNAL, message),

  Unavailable: () =>
    createGrpcError(grpc.status.UNAVAILABLE, 'Service temporarily unavailable'),
};
```

---

## 11. Testing ด้วย grpcurl

```bash
# ติดตั้ง grpcurl
# macOS: brew install grpcurl
# Linux: go install github.com/fullstorydev/grpcurl/cmd/grpcurl@latest

# List available services
grpcurl -plaintext localhost:50051 list

# List methods ของ UserService
grpcurl -plaintext localhost:50051 list user.v1.UserService

# GetUser
grpcurl -plaintext -d '{"id": "550e8400-e29b-41d4-a716-446655440000"}' \
  localhost:50051 user.v1.UserService/GetUser

# CreateUser
grpcurl -plaintext \
  -d '{"email": "test@example.com", "name": "Test User", "password": "password123"}' \
  localhost:50051 user.v1.UserService/CreateUser

# ListUsers
grpcurl -plaintext \
  -d '{"page": 1, "page_size": 10, "search": "john"}' \
  localhost:50051 user.v1.UserService/ListUsers

# Stream Users
grpcurl -plaintext \
  -d '{"batch_size": 50}' \
  localhost:50051 user.v1.UserService/StreamUsers

# ด้วย Authentication Header
grpcurl -plaintext \
  -H 'authorization: Bearer eyJhbGciOiJIUzI1NiJ9...' \
  -d '{"id": "user-123"}' \
  localhost:50051 user.v1.UserService/GetUser
```

---

## 12. gRPC Reflection

```typescript
// src/reflection/index.ts
import * as grpc from '@grpc/grpc-js';

// Enable reflection ให้ grpcurl ค้นหา services ได้โดยไม่ต้องมี .proto file
const reflection = require('@grpc/reflection');

export function addReflection(server: grpc.Server, filePaths: string[]): void {
  reflection.addReflection(server, filePaths);
}

// ใน server.ts:
// import { addReflection } from './reflection';
// addReflection(server, [PROTO_PATH]);
```

---

## 13. Load Balancing

### 13.1 Client-Side Load Balancing

```typescript
// src/clients/load-balanced.client.ts

// Round-robin across multiple servers
const userClient = new userProto.user.v1.UserService(
  'dns:///user-service:50051', // DNS round-robin
  grpc.credentials.createInsecure(),
  {
    'grpc.lb_policy_name': 'round_robin',
    'grpc.service_config': JSON.stringify({
      loadBalancingConfig: [{ round_robin: {} }],
    }),
  },
);

// Or use Kubernetes Service Discovery
const k8sUserClient = new userProto.user.v1.UserService(
  'kubernetes:///user-service.default.svc.cluster.local:50051',
  grpc.credentials.createInsecure(),
);
```

### 13.2 Docker Compose ครบชุด

```yaml
# docker-compose.grpc.yml
version: '3.8'

services:
  postgres-primary:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: grpc_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres123
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  postgres-replica:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: grpc_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres123
    depends_on:
      - postgres-primary
    ports:
      - "5433:5432"

  redis:
    image: redis:7-alpine
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru
    ports:
      - "6379:6379"

  user-service-1:
    build: .
    environment:
      GRPC_PORT: '50051'
      PG_HOST: postgres-primary
      PG_READ_HOST: postgres-replica
      PG_DATABASE: grpc_db
      PG_USER: postgres
      PG_PASSWORD: postgres123
      REDIS_HOST: redis
    depends_on:
      - postgres-primary
      - redis
    ports:
      - "50051:50051"

  user-service-2:
    build: .
    environment:
      GRPC_PORT: '50051'
      PG_HOST: postgres-primary
      PG_READ_HOST: postgres-replica
      PG_DATABASE: grpc_db
      PG_USER: postgres
      PG_PASSWORD: postgres123
      REDIS_HOST: redis
    depends_on:
      - postgres-primary
      - redis
    ports:
      - "50052:50051"

  envoy-proxy:
    image: envoyproxy/envoy:v1.28-latest
    volumes:
      - ./envoy.yaml:/etc/envoy/envoy.yaml
    ports:
      - "50050:50050"   # gRPC load balanced port
      - "9901:9901"     # Admin interface
    depends_on:
      - user-service-1
      - user-service-2

volumes:
  postgres_data:
```

### 13.3 Envoy Configuration

```yaml
# envoy.yaml
static_resources:
  listeners:
  - name: listener_0
    address:
      socket_address:
        address: 0.0.0.0
        port_value: 50050
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          codec_type: AUTO
          stat_prefix: ingress_http
          route_config:
            name: local_route
            virtual_hosts:
            - name: local_service
              domains: ["*"]
              routes:
              - match:
                  prefix: "/"
                route:
                  cluster: user_service
                  timeout: 10s
          http_filters:
          - name: envoy.filters.http.router
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router

  clusters:
  - name: user_service
    connect_timeout: 5s
    type: STRICT_DNS
    lb_policy: ROUND_ROBIN
    http2_protocol_options: {}
    load_assignment:
      cluster_name: user_service
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: user-service-1
                port_value: 50051
        - endpoint:
            address:
              socket_address:
                address: user-service-2
                port_value: 50051
    health_checks:
    - timeout: 1s
      interval: 10s
      unhealthy_threshold: 2
      healthy_threshold: 1
      grpc_health_check: {}
```

---

## 14. Testing ครบชุด

```typescript
// tests/user.service.spec.ts
import * as grpc from '@grpc/grpc-js';
import * as protoLoader from '@grpc/proto-loader';
import { UserServiceClient } from '../src/clients/user.client';

describe('UserService gRPC', () => {
  let client: UserServiceClient;

  beforeAll(async () => {
    client = new UserServiceClient('localhost:50051');
    // Wait for server to be ready
    await new Promise((resolve) => setTimeout(resolve, 1000));
  });

  afterAll(() => {
    client.close();
  });

  describe('CreateUser', () => {
    it('should create user successfully', async () => {
      const user = await client.createUser({
        email: `test-${Date.now()}@example.com`,
        name: 'Test User',
        password: 'password123',
      });

      expect(user.id).toBeDefined();
      expect(user.email).toContain('@example.com');
      expect(user.name).toBe('Test User');
    });

    it('should reject duplicate email', async () => {
      const email = `duplicate-${Date.now()}@example.com`;
      await client.createUser({ email, name: 'User 1', password: 'pass123' });

      await expect(
        client.createUser({ email, name: 'User 2', password: 'pass456' }),
      ).rejects.toMatchObject({ code: grpc.status.ALREADY_EXISTS });
    });

    it('should reject invalid email', async () => {
      await expect(
        client.createUser({ email: 'not-an-email', name: 'Test', password: 'pass' }),
      ).rejects.toMatchObject({ code: grpc.status.INVALID_ARGUMENT });
    });
  });

  describe('GetUser', () => {
    it('should return cached user on second call', async () => {
      const created = await client.createUser({
        email: `cache-test-${Date.now()}@example.com`,
        name: 'Cache Test',
        password: 'pass123',
      });

      const start1 = Date.now();
      await client.getUser(created.id);
      const time1 = Date.now() - start1;

      const start2 = Date.now();
      await client.getUser(created.id);
      const time2 = Date.now() - start2;

      // Second call should be faster (from cache)
      expect(time2).toBeLessThan(time1);
    });

    it('should return NOT_FOUND for nonexistent user', async () => {
      await expect(
        client.getUser('00000000-0000-0000-0000-000000000000'),
      ).rejects.toMatchObject({ code: grpc.status.NOT_FOUND });
    });
  });

  describe('StreamUsers', () => {
    it('should stream all users', async () => {
      const users = await client.streamAllUsersToArray();
      expect(Array.isArray(users)).toBe(true);
      expect(users.length).toBeGreaterThan(0);
    });
  });

  describe('BulkCreateUsers', () => {
    it('should create multiple users', async () => {
      const users = Array.from({ length: 100 }, (_, i) => ({
        email: `bulk-${Date.now()}-${i}@example.com`,
        name: `Bulk User ${i}`,
        password: 'password123',
      }));

      const result = await client.bulkCreateUsers(users);
      expect(result.processed).toBe(100);
    });
  });
});
```

---

## 15. สรุป

### 15.1 gRPC Benefits สำหรับ Database Cluster

1. **High Performance**: Binary Protocol ส่งข้อมูลเร็วกว่า REST/GraphQL
2. **Streaming**: Server Streaming สำหรับ Large Datasets ออกจาก Database
3. **Type Safety**: .proto เป็น Contract ที่ Enforce ทั้งสองฝ่าย
4. **Bidirectional**: Real-time Sync ระหว่าง Services
5. **Built-in Load Balancing**: DNS/Client-side Round Robin

### 15.2 Database Integration Best Practices

| Pattern | Implementation |
|---------|---------------|
| Read/Write Split | pgRead Pool → Query, pgWrite Pool → Mutation |
| Redis Caching | Cache GetUser responses, Invalidate on Update |
| Connection Pool | PG Pool ขนาดเหมาะสมกับ Concurrency |
| Streaming | Server Streaming สำหรับ Export/Bulk Operations |
| Error Mapping | DB Errors → Proper gRPC Status Codes |

### 15.3 When gRPC Shines

```
Microservice A ──gRPC──► Microservice B ──gRPC──► Microservice C
(Order Service)           (Inventory)               (Payment)

vs REST:
Microservice A ──HTTP──► Microservice B ──HTTP──► Microservice C

gRPC ให้ผล:
- Latency ต่ำกว่า 30-50%
- Throughput สูงกว่า 2-5x
- Type Safety ทั้ง Stack
```

gRPC เป็น Communication Layer ที่เหมาะที่สุดสำหรับ Internal Microservices ที่ต้องการ Performance สูง โดยเฉพาะเมื่อ Services ต้องการ Stream ข้อมูลจำนวนมากจาก Database Cluster

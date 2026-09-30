# Part 48: Testing - Unit, Integration, E2E

## Testing Pyramid

```
         /\
        /E2E\         ← น้อยสุด แต่ครอบคลุมกว้าง (Playwright, Cypress)
       /------\
      /  Integ \      ← ปานกลาง (Supertest + Real DB)
     /----------\
    /    Unit    \    ← มากสุด เร็วสุด (Jest, Vitest)
   /--------------\
```

### ทำไมต้อง Test ตาม Pyramid?

- **Unit Tests**: เร็ว (milliseconds), isolate, ง่ายต่อการ debug
- **Integration Tests**: ช้ากว่า (seconds), test ว่า components ทำงานร่วมกันได้
- **E2E Tests**: ช้าสุด (minutes), test user flows จริงใน browser

---

## Setup โปรเจค

```bash
npm install --save-dev jest ts-jest @types/jest
npm install --save-dev supertest @types/supertest
npm install --save-dev jest-mock-extended
npm install --save-dev @jest/globals
npm install --save-dev ioredis-mock
npm install --save-dev @playwright/test

# ถ้าใช้ Vitest (เร็วกว่า Jest สำหรับ TypeScript)
npm install --save-dev vitest @vitest/coverage-v8
```

### jest.config.ts

```typescript
import type { Config } from 'jest';

const config: Config = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  
  // Test file patterns
  testMatch: [
    '**/__tests__/**/*.test.ts',
    '**/*.spec.ts',
  ],
  
  // Coverage
  collectCoverageFrom: [
    'src/**/*.ts',
    '!src/**/*.d.ts',
    '!src/migrations/**',
    '!src/seeds/**',
  ],
  coverageThreshold: {
    global: {
      branches: 70,
      functions: 80,
      lines: 80,
      statements: 80,
    },
  },
  coverageReporters: ['text', 'lcov', 'html'],
  
  // Setup files
  setupFilesAfterFramework: ['<rootDir>/src/__tests__/setup.ts'],
  
  // Module paths
  moduleNameMapper: {
    '^@/(.*)$': '<rootDir>/src/$1',
  },
  
  // Projects: run unit and integration separately
  projects: [
    {
      displayName: 'unit',
      testMatch: ['**/__tests__/unit/**/*.test.ts'],
      testEnvironment: 'node',
    },
    {
      displayName: 'integration',
      testMatch: ['**/__tests__/integration/**/*.test.ts'],
      testEnvironment: 'node',
      globalSetup: '<rootDir>/src/__tests__/globalSetup.ts',
      globalTeardown: '<rootDir>/src/__tests__/globalTeardown.ts',
    },
  ],
};

export default config;
```

---

## Unit Testing

### การ Test Service Layer

```typescript
// src/services/__tests__/unit/user.service.test.ts
import { describe, it, expect, beforeEach, jest } from '@jest/globals';
import { UserService } from '../../user.service';
import { UserRepository } from '../../../repositories/user.repository';
import { CacheService } from '../../cache.service';
import { EmailService } from '../../email.service';

// Mock ทุก dependency
jest.mock('../../../repositories/user.repository');
jest.mock('../../cache.service');
jest.mock('../../email.service');

const MockUserRepository = UserRepository as jest.MockedClass<typeof UserRepository>;
const MockCacheService = CacheService as jest.MockedClass<typeof CacheService>;
const MockEmailService = EmailService as jest.MockedClass<typeof EmailService>;

describe('UserService', () => {
  let userService: UserService;
  let mockUserRepo: jest.Mocked<UserRepository>;
  let mockCache: jest.Mocked<CacheService>;
  let mockEmail: jest.Mocked<EmailService>;

  beforeEach(() => {
    // Clear all mocks before each test
    jest.clearAllMocks();

    mockUserRepo = new MockUserRepository() as jest.Mocked<UserRepository>;
    mockCache = new MockCacheService() as jest.Mocked<CacheService>;
    mockEmail = new MockEmailService() as jest.Mocked<EmailService>;

    userService = new UserService(mockUserRepo, mockCache, mockEmail);
  });

  // ============================================================
  // findById tests
  // ============================================================

  describe('findById', () => {
    it('should return user from cache if exists', async () => {
      const cachedUser = { id: 1, firstName: 'สมชาย', email: 'test@test.com' };
      mockCache.get.mockResolvedValue(JSON.stringify(cachedUser));

      const result = await userService.findById(1);

      expect(result).toEqual(cachedUser);
      expect(mockCache.get).toHaveBeenCalledWith('user:1');
      expect(mockUserRepo.findById).not.toHaveBeenCalled();  // ไม่ hit DB
    });

    it('should fetch from DB and cache if not in cache', async () => {
      const dbUser = { id: 1, firstName: 'สมชาย', email: 'test@test.com', createdAt: new Date() };
      mockCache.get.mockResolvedValue(null);
      mockUserRepo.findById.mockResolvedValue(dbUser);
      mockCache.set.mockResolvedValue(undefined);

      const result = await userService.findById(1);

      expect(result).toEqual(dbUser);
      expect(mockUserRepo.findById).toHaveBeenCalledWith(1);
      expect(mockCache.set).toHaveBeenCalledWith('user:1', expect.any(String), 3600);
    });

    it('should return null if user not found', async () => {
      mockCache.get.mockResolvedValue(null);
      mockUserRepo.findById.mockResolvedValue(null);

      const result = await userService.findById(999);

      expect(result).toBeNull();
    });
  });

  // ============================================================
  // create tests
  // ============================================================

  describe('create', () => {
    const createData = {
      firstName: 'สมชาย',
      lastName: 'ใจดี',
      email: 'somchai@example.com',
      password: 'Password123!',
    };

    it('should create user and send welcome email', async () => {
      const newUser = { id: 1, ...createData, password: 'hashed', createdAt: new Date() };
      mockUserRepo.findByEmail.mockResolvedValue(null);
      mockUserRepo.create.mockResolvedValue(newUser);
      mockEmail.sendWelcome.mockResolvedValue(undefined);

      const result = await userService.create(createData);

      expect(result).toEqual(newUser);
      expect(mockUserRepo.create).toHaveBeenCalledWith(
        expect.objectContaining({
          firstName: 'สมชาย',
          email: 'somchai@example.com',
          password: expect.stringMatching(/^\$2[ab]\$/),  // bcrypt hash
        })
      );
      expect(mockEmail.sendWelcome).toHaveBeenCalledWith('somchai@example.com', 'สมชาย');
    });

    it('should throw error if email already exists', async () => {
      mockUserRepo.findByEmail.mockResolvedValue({ id: 2, email: 'somchai@example.com' } as any);

      await expect(userService.create(createData)).rejects.toThrow('Email already exists');
    });

    it('should hash password before saving', async () => {
      mockUserRepo.findByEmail.mockResolvedValue(null);
      mockUserRepo.create.mockResolvedValue({ id: 1, ...createData } as any);
      mockEmail.sendWelcome.mockResolvedValue(undefined);

      await userService.create(createData);

      const createCall = mockUserRepo.create.mock.calls[0][0];
      expect(createCall.password).not.toBe(createData.password);
      expect(createCall.password).toMatch(/^\$2[ab]\$/);  // bcrypt pattern
    });
  });

  // ============================================================
  // delete tests
  // ============================================================

  describe('delete', () => {
    it('should delete user and invalidate cache', async () => {
      mockUserRepo.findById.mockResolvedValue({ id: 1, email: 'test@test.com' } as any);
      mockUserRepo.delete.mockResolvedValue(undefined);
      mockCache.del.mockResolvedValue(undefined);

      await userService.delete(1);

      expect(mockUserRepo.delete).toHaveBeenCalledWith(1);
      expect(mockCache.del).toHaveBeenCalledWith('user:1');
    });

    it('should throw NotFoundError if user does not exist', async () => {
      mockUserRepo.findById.mockResolvedValue(null);

      await expect(userService.delete(999)).rejects.toThrow('User not found');
    });
  });
});
```

### Mock Database ด้วย jest-mock-extended

```typescript
// src/__tests__/unit/post.service.test.ts
import { mockDeep, MockProxy } from 'jest-mock-extended';
import { PrismaClient } from '@prisma/client';
import { PostService } from '../../services/post.service';

describe('PostService with Prisma Mock', () => {
  let mockPrisma: MockProxy<PrismaClient>;
  let postService: PostService;

  beforeEach(() => {
    mockPrisma = mockDeep<PrismaClient>();
    postService = new PostService(mockPrisma);
  });

  it('should create a post', async () => {
    const postData = {
      title: 'Test Post',
      content: 'Content here',
      authorId: 1,
    };

    const createdPost = {
      id: 1,
      ...postData,
      published: false,
      createdAt: new Date(),
      updatedAt: new Date(),
    };

    mockPrisma.post.create.mockResolvedValue(createdPost);

    const result = await postService.create(postData);

    expect(result).toEqual(createdPost);
    expect(mockPrisma.post.create).toHaveBeenCalledWith({
      data: postData,
    });
  });

  it('should find posts with author', async () => {
    const posts = [
      { id: 1, title: 'Post 1', author: { id: 1, firstName: 'สมชาย' } },
      { id: 2, title: 'Post 2', author: { id: 1, firstName: 'สมชาย' } },
    ];

    mockPrisma.post.findMany.mockResolvedValue(posts as any);

    const result = await postService.findWithAuthor(1);

    expect(result).toHaveLength(2);
    expect(mockPrisma.post.findMany).toHaveBeenCalledWith({
      where: { authorId: 1, published: true },
      include: { author: true },
      orderBy: { createdAt: 'desc' },
    });
  });
});
```

### Mock Redis ด้วย ioredis-mock

```typescript
// src/__tests__/unit/cache.service.test.ts
import RedisMock from 'ioredis-mock';
import { CacheService } from '../../services/cache.service';

// Replace ioredis with mock
jest.mock('ioredis', () => require('ioredis-mock'));

describe('CacheService', () => {
  let redis: InstanceType<typeof RedisMock>;
  let cacheService: CacheService;

  beforeEach(() => {
    redis = new RedisMock();
    cacheService = new CacheService(redis as any);
  });

  afterEach(async () => {
    await redis.flushall();
  });

  it('should set and get value', async () => {
    await cacheService.set('key1', 'value1', 60);
    const result = await cacheService.get('key1');
    expect(result).toBe('value1');
  });

  it('should return null for missing key', async () => {
    const result = await cacheService.get('nonexistent');
    expect(result).toBeNull();
  });

  it('should delete key', async () => {
    await redis.set('key1', 'value1');
    await cacheService.del('key1');
    const result = await redis.get('key1');
    expect(result).toBeNull();
  });

  it('should expire key after TTL', async () => {
    jest.useFakeTimers();
    await cacheService.set('key1', 'value1', 1);  // 1 second TTL

    jest.advanceTimersByTime(2000);

    const ttl = await redis.ttl('key1');
    expect(ttl).toBeLessThanOrEqual(0);

    jest.useRealTimers();
  });

  it('should cache JSON objects', async () => {
    const obj = { id: 1, name: 'สมชาย', tags: ['a', 'b'] };
    await cacheService.setJson('user:1', obj, 60);
    const result = await cacheService.getJson<typeof obj>('user:1');
    expect(result).toEqual(obj);
  });
});
```

### Test Utility Functions

```typescript
// src/utils/__tests__/unit/pagination.test.ts
import { calculatePagination, buildPaginationMeta } from '../../pagination';

describe('Pagination Utils', () => {
  describe('calculatePagination', () => {
    it('should calculate correct offset', () => {
      expect(calculatePagination(1, 20)).toEqual({ offset: 0, limit: 20 });
      expect(calculatePagination(2, 20)).toEqual({ offset: 20, limit: 20 });
      expect(calculatePagination(3, 10)).toEqual({ offset: 20, limit: 10 });
    });

    it('should cap limit at 100', () => {
      expect(calculatePagination(1, 500)).toEqual({ offset: 0, limit: 100 });
    });

    it('should default page to 1 if less than 1', () => {
      expect(calculatePagination(0, 20)).toEqual({ offset: 0, limit: 20 });
      expect(calculatePagination(-1, 20)).toEqual({ offset: 0, limit: 20 });
    });
  });

  describe('buildPaginationMeta', () => {
    it('should build correct pagination meta', () => {
      const meta = buildPaginationMeta(2, 10, 45);
      expect(meta).toEqual({
        page: 2,
        limit: 10,
        total: 45,
        totalPages: 5,
        hasNext: true,
        hasPrev: true,
        nextPage: 3,
        prevPage: 1,
      });
    });

    it('should indicate no next page on last page', () => {
      const meta = buildPaginationMeta(5, 10, 45);
      expect(meta.hasNext).toBe(false);
      expect(meta.nextPage).toBeNull();
    });

    it('should indicate no prev page on first page', () => {
      const meta = buildPaginationMeta(1, 10, 45);
      expect(meta.hasPrev).toBe(false);
      expect(meta.prevPage).toBeNull();
    });
  });
});
```

---

## Integration Testing

### Setup: Test Database

```typescript
// src/__tests__/globalSetup.ts
import { exec } from 'child_process';
import { promisify } from 'util';
import pg from 'pg';

const execAsync = promisify(exec);

export default async function globalSetup() {
  console.log('Setting up test database...');

  // Create test database
  const client = new pg.Client({
    host: process.env.DB_HOST || 'localhost',
    port: parseInt(process.env.DB_PORT || '5432'),
    user: process.env.DB_USER || 'postgres',
    password: process.env.DB_PASSWORD || 'password',
    database: 'postgres',
  });

  await client.connect();
  
  // Drop and recreate test database
  await client.query('DROP DATABASE IF EXISTS test_db');
  await client.query('CREATE DATABASE test_db');
  await client.end();

  // Run migrations
  await execAsync('DATABASE_URL=postgresql://postgres:password@localhost:5432/test_db npx prisma migrate deploy');

  console.log('Test database ready');
}
```

```typescript
// src/__tests__/globalTeardown.ts
import pg from 'pg';

export default async function globalTeardown() {
  console.log('Cleaning up test database...');

  const client = new pg.Client({
    host: process.env.DB_HOST || 'localhost',
    user: process.env.DB_USER || 'postgres',
    password: process.env.DB_PASSWORD || 'password',
    database: 'postgres',
  });

  await client.connect();
  await client.query('DROP DATABASE IF EXISTS test_db');
  await client.end();

  console.log('Test database cleaned up');
}
```

```typescript
// src/__tests__/setup.ts
import { beforeAll, afterAll, beforeEach } from '@jest/globals';
import { PrismaClient } from '@prisma/client';

export const prisma = new PrismaClient({
  datasources: {
    db: {
      url: process.env.TEST_DATABASE_URL || 'postgresql://postgres:password@localhost:5432/test_db',
    },
  },
});

beforeAll(async () => {
  await prisma.$connect();
});

afterAll(async () => {
  await prisma.$disconnect();
});

// Clean database before each test (transaction rollback approach)
beforeEach(async () => {
  // Delete in reverse dependency order
  await prisma.post.deleteMany();
  await prisma.session.deleteMany();
  await prisma.user.deleteMany();
});
```

### Integration Test: API Endpoints

```typescript
// src/__tests__/integration/users.api.test.ts
import request from 'supertest';
import { app } from '../../app';
import { prisma } from '../setup';
import bcrypt from 'bcrypt';
import jwt from 'jsonwebtoken';

describe('Users API - Integration', () => {
  // Seed test user
  let testUser: any;
  let authToken: string;

  beforeEach(async () => {
    // Create test user directly in DB
    testUser = await prisma.user.create({
      data: {
        firstName: 'สมชาย',
        lastName: 'ใจดี',
        email: 'somchai@test.com',
        password: await bcrypt.hash('Password123!', 10),
        role: 'user',
        emailVerified: true,
      },
    });

    // Generate auth token
    authToken = jwt.sign(
      { sub: testUser.id.toString(), email: testUser.email, role: testUser.role },
      process.env.JWT_SECRET || 'test-secret',
      { expiresIn: '1h' },
    );
  });

  // ============================================================
  // GET /api/v2/users
  // ============================================================

  describe('GET /api/v2/users', () => {
    beforeEach(async () => {
      // Create multiple users
      await prisma.user.createMany({
        data: [
          { firstName: 'วิชัย', lastName: 'สมบูรณ์', email: 'wichai@test.com', password: 'hash', role: 'user', emailVerified: true },
          { firstName: 'สมหญิง', lastName: 'รักดี', email: 'somying@test.com', password: 'hash', role: 'admin', emailVerified: true },
        ],
      });
    });

    it('should return list of users with pagination', async () => {
      const res = await request(app)
        .get('/api/v2/users')
        .set('Authorization', `Bearer ${authToken}`)
        .query({ page: 1, limit: 2 });

      expect(res.status).toBe(200);
      expect(res.body.success).toBe(true);
      expect(res.body.data).toHaveLength(2);
      expect(res.body.meta).toMatchObject({
        page: 1,
        limit: 2,
        total: 3,
        totalPages: 2,
        hasNext: true,
        hasPrev: false,
      });
    });

    it('should return 401 without auth token', async () => {
      const res = await request(app).get('/api/v2/users');
      expect(res.status).toBe(401);
    });

    it('should filter users by search', async () => {
      const res = await request(app)
        .get('/api/v2/users')
        .set('Authorization', `Bearer ${authToken}`)
        .query({ search: 'วิชัย' });

      expect(res.status).toBe(200);
      expect(res.body.data).toHaveLength(1);
      expect(res.body.data[0].firstName).toBe('วิชัย');
    });
  });

  // ============================================================
  // GET /api/v2/users/:id
  // ============================================================

  describe('GET /api/v2/users/:id', () => {
    it('should return user by id', async () => {
      const res = await request(app)
        .get(`/api/v2/users/${testUser.id}`)
        .set('Authorization', `Bearer ${authToken}`);

      expect(res.status).toBe(200);
      expect(res.body.data).toMatchObject({
        id: testUser.id,
        firstName: 'สมชาย',
        lastName: 'ใจดี',
        email: 'somchai@test.com',
      });
      expect(res.body.data.password).toBeUndefined();  // password ไม่ควร return
    });

    it('should return 404 for non-existent user', async () => {
      const res = await request(app)
        .get('/api/v2/users/99999')
        .set('Authorization', `Bearer ${authToken}`);

      expect(res.status).toBe(404);
      expect(res.body.error.code).toBe('USER_NOT_FOUND');
    });
  });

  // ============================================================
  // POST /api/v2/users
  // ============================================================

  describe('POST /api/v2/users', () => {
    it('should create new user', async () => {
      const newUserData = {
        firstName: 'ใหม่',
        lastName: 'มาก',
        email: 'newuser@test.com',
        password: 'Password123!',
      };

      const res = await request(app)
        .post('/api/v2/users')
        .send(newUserData);

      expect(res.status).toBe(201);
      expect(res.body.data).toMatchObject({
        firstName: 'ใหม่',
        lastName: 'มาก',
        email: 'newuser@test.com',
      });

      // ตรวจสอบว่า user ถูก save ใน DB จริงๆ
      const dbUser = await prisma.user.findUnique({ where: { email: 'newuser@test.com' } });
      expect(dbUser).not.toBeNull();
      expect(dbUser!.password).not.toBe(newUserData.password);  // ควร hash แล้ว
    });

    it('should return 409 for duplicate email', async () => {
      const res = await request(app)
        .post('/api/v2/users')
        .send({
          firstName: 'ซ้ำ',
          lastName: 'อีเมล',
          email: 'somchai@test.com',  // email ซ้ำกับ testUser
          password: 'Password123!',
        });

      expect(res.status).toBe(409);
    });

    it('should return 400 for invalid email', async () => {
      const res = await request(app)
        .post('/api/v2/users')
        .send({
          firstName: 'ทดสอบ',
          lastName: 'ทดสอบ',
          email: 'not-an-email',
          password: 'Password123!',
        });

      expect(res.status).toBe(400);
      expect(res.body.error.code).toBe('VALIDATION_ERROR');
    });
  });

  // ============================================================
  // DELETE /api/v2/users/:id
  // ============================================================

  describe('DELETE /api/v2/users/:id', () => {
    it('should delete user (admin only)', async () => {
      // Create admin token
      const adminToken = jwt.sign(
        { sub: testUser.id.toString(), role: 'admin' },
        process.env.JWT_SECRET || 'test-secret',
        { expiresIn: '1h' },
      );

      const res = await request(app)
        .delete(`/api/v2/users/${testUser.id}`)
        .set('Authorization', `Bearer ${adminToken}`);

      expect(res.status).toBe(204);

      // ตรวจสอบว่า user ถูกลบใน DB
      const dbUser = await prisma.user.findUnique({ where: { id: testUser.id } });
      expect(dbUser).toBeNull();
    });

    it('should return 403 for non-admin user', async () => {
      // Create another user
      const otherUser = await prisma.user.create({
        data: {
          firstName: 'อื่น', lastName: 'คน',
          email: 'other@test.com',
          password: 'hash', role: 'user', emailVerified: true,
        },
      });

      const res = await request(app)
        .delete(`/api/v2/users/${otherUser.id}`)
        .set('Authorization', `Bearer ${authToken}`);  // regular user token

      expect(res.status).toBe(403);
    });
  });
});
```

### Transaction Rollback Pattern

```typescript
// src/__tests__/integration/helpers/db-transaction.ts
import { PrismaClient } from '@prisma/client';

// ใช้ transaction เพื่อ rollback หลังแต่ละ test
// ทำให้ test ต่างๆ independent กัน
export function withRollback(prisma: PrismaClient) {
  return async (fn: () => Promise<void>) => {
    // Start transaction
    await prisma.$executeRaw`BEGIN`;

    try {
      await fn();
    } finally {
      // Rollback เสมอ ไม่ว่า test จะ pass หรือ fail
      await prisma.$executeRaw`ROLLBACK`;
    }
  };
}

// Usage ใน test:
// it('should do something', withRollback(prisma)(async () => {
//   const user = await prisma.user.create({ data: { ... } });
//   // test logic
//   // rollback automatically after test
// }));
```

### Database Seeding

```typescript
// src/__tests__/integration/helpers/seed.ts
import { PrismaClient } from '@prisma/client';
import bcrypt from 'bcrypt';

export interface TestData {
  users: any[];
  posts: any[];
}

export async function seedDatabase(prisma: PrismaClient): Promise<TestData> {
  const hashedPassword = await bcrypt.hash('TestPassword123!', 10);

  // สร้าง users
  const users = await Promise.all([
    prisma.user.create({
      data: {
        firstName: 'แอดมิน',
        lastName: 'ระบบ',
        email: 'admin@test.com',
        password: hashedPassword,
        role: 'admin',
        emailVerified: true,
      },
    }),
    prisma.user.create({
      data: {
        firstName: 'ผู้ใช้',
        lastName: 'ทดสอบ',
        email: 'user@test.com',
        password: hashedPassword,
        role: 'user',
        emailVerified: true,
      },
    }),
  ]);

  // สร้าง posts
  const posts = await Promise.all([
    prisma.post.create({
      data: {
        title: 'โพสต์ทดสอบ 1',
        content: 'เนื้อหาทดสอบ 1',
        published: true,
        authorId: users[1].id,
      },
    }),
    prisma.post.create({
      data: {
        title: 'โพสต์ทดสอบ 2',
        content: 'เนื้อหาทดสอบ 2',
        published: false,
        authorId: users[1].id,
      },
    }),
  ]);

  return { users, posts };
}

export async function cleanDatabase(prisma: PrismaClient): Promise<void> {
  await prisma.post.deleteMany();
  await prisma.user.deleteMany();
}
```

### Auth API Integration Tests

```typescript
// src/__tests__/integration/auth.api.test.ts
import request from 'supertest';
import { app } from '../../app';
import { prisma } from '../setup';
import bcrypt from 'bcrypt';
import { createClient } from 'redis';

const redis = createClient({ url: process.env.TEST_REDIS_URL });

beforeAll(async () => {
  await redis.connect();
});

afterAll(async () => {
  await redis.disconnect();
});

beforeEach(async () => {
  await redis.flushDb();  // clear Redis before each test
});

describe('Auth API - Integration', () => {
  let testUser: any;

  beforeEach(async () => {
    testUser = await prisma.user.create({
      data: {
        firstName: 'ผู้ใช้',
        lastName: 'ทดสอบ',
        email: 'test@example.com',
        password: await bcrypt.hash('Password123!', 10),
        role: 'user',
        emailVerified: true,
      },
    });
  });

  describe('POST /api/v2/auth/login', () => {
    it('should login with correct credentials', async () => {
      const res = await request(app)
        .post('/api/v2/auth/login')
        .send({ email: 'test@example.com', password: 'Password123!' });

      expect(res.status).toBe(200);
      expect(res.body.data).toHaveProperty('accessToken');
      expect(res.body.data).toHaveProperty('refreshToken');
      expect(res.body.data.user).toMatchObject({
        id: testUser.id,
        email: 'test@example.com',
      });
    });

    it('should return 401 for wrong password', async () => {
      const res = await request(app)
        .post('/api/v2/auth/login')
        .send({ email: 'test@example.com', password: 'wrongpassword' });

      expect(res.status).toBe(401);
    });

    it('should lockout after 5 failed attempts', async () => {
      // Fail 5 times
      for (let i = 0; i < 5; i++) {
        await request(app)
          .post('/api/v2/auth/login')
          .send({ email: 'test@example.com', password: 'wrong' });
      }

      // 6th attempt should be locked
      const res = await request(app)
        .post('/api/v2/auth/login')
        .send({ email: 'test@example.com', password: 'Password123!' });

      expect(res.status).toBe(423);  // Locked
      expect(res.body.error.code).toBe('ACCOUNT_LOCKED');
    });

    it('should return 400 for missing fields', async () => {
      const res = await request(app)
        .post('/api/v2/auth/login')
        .send({ email: 'test@example.com' });  // no password

      expect(res.status).toBe(400);
    });
  });

  describe('POST /api/v2/auth/logout', () => {
    it('should logout and invalidate token', async () => {
      // Login first
      const loginRes = await request(app)
        .post('/api/v2/auth/login')
        .send({ email: 'test@example.com', password: 'Password123!' });

      const { accessToken } = loginRes.body.data;

      // Logout
      const logoutRes = await request(app)
        .post('/api/v2/auth/logout')
        .set('Authorization', `Bearer ${accessToken}`);

      expect(logoutRes.status).toBe(200);

      // Try to use token after logout
      const protectedRes = await request(app)
        .get('/api/v2/users')
        .set('Authorization', `Bearer ${accessToken}`);

      expect(protectedRes.status).toBe(401);
      expect(protectedRes.body.error.code).toBe('TOKEN_REVOKED');
    });
  });

  describe('POST /api/v2/auth/refresh', () => {
    it('should refresh access token', async () => {
      // Login
      const loginRes = await request(app)
        .post('/api/v2/auth/login')
        .send({ email: 'test@example.com', password: 'Password123!' });

      const { refreshToken } = loginRes.body.data;

      // Refresh
      const refreshRes = await request(app)
        .post('/api/v2/auth/refresh')
        .send({ refreshToken });

      expect(refreshRes.status).toBe(200);
      expect(refreshRes.body.data).toHaveProperty('accessToken');
    });
  });
});
```

---

## E2E Testing with Playwright

```typescript
// e2e/auth.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Authentication Flow', () => {
  test('should login successfully', async ({ page }) => {
    // Navigate to login page
    await page.goto('/login');

    // Fill form
    await page.fill('[data-testid="email-input"]', 'user@example.com');
    await page.fill('[data-testid="password-input"]', 'Password123!');

    // Submit
    await page.click('[data-testid="login-button"]');

    // Wait for redirect to dashboard
    await page.waitForURL('/dashboard');

    // Verify logged in state
    await expect(page.locator('[data-testid="user-menu"]')).toBeVisible();
    await expect(page.locator('[data-testid="user-email"]')).toHaveText('user@example.com');
  });

  test('should show error for invalid credentials', async ({ page }) => {
    await page.goto('/login');

    await page.fill('[data-testid="email-input"]', 'user@example.com');
    await page.fill('[data-testid="password-input"]', 'wrongpassword');
    await page.click('[data-testid="login-button"]');

    // Should stay on login page and show error
    await expect(page).toHaveURL('/login');
    await expect(page.locator('[data-testid="error-message"]')).toBeVisible();
    await expect(page.locator('[data-testid="error-message"]')).toContainText('Invalid credentials');
  });

  test('should logout successfully', async ({ page }) => {
    // Login first
    await page.goto('/login');
    await page.fill('[data-testid="email-input"]', 'user@example.com');
    await page.fill('[data-testid="password-input"]', 'Password123!');
    await page.click('[data-testid="login-button"]');
    await page.waitForURL('/dashboard');

    // Logout
    await page.click('[data-testid="user-menu"]');
    await page.click('[data-testid="logout-button"]');

    // Should redirect to login
    await page.waitForURL('/login');

    // Navigate to dashboard should redirect back
    await page.goto('/dashboard');
    await expect(page).toHaveURL('/login');
  });
});
```

### Page Object Model Pattern

```typescript
// e2e/pages/LoginPage.ts
import { Page, Locator } from '@playwright/test';

export class LoginPage {
  readonly page: Page;
  readonly emailInput: Locator;
  readonly passwordInput: Locator;
  readonly loginButton: Locator;
  readonly errorMessage: Locator;

  constructor(page: Page) {
    this.page = page;
    this.emailInput = page.locator('[data-testid="email-input"]');
    this.passwordInput = page.locator('[data-testid="password-input"]');
    this.loginButton = page.locator('[data-testid="login-button"]');
    this.errorMessage = page.locator('[data-testid="error-message"]');
  }

  async goto() {
    await this.page.goto('/login');
  }

  async login(email: string, password: string) {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.loginButton.click();
  }

  async getErrorMessage() {
    return this.errorMessage.textContent();
  }
}

// e2e/pages/DashboardPage.ts
export class DashboardPage {
  readonly page: Page;

  constructor(page: Page) {
    this.page = page;
  }

  async goto() {
    await this.page.goto('/dashboard');
  }

  async isLoggedIn() {
    return this.page.locator('[data-testid="user-menu"]').isVisible();
  }
}

// Using Page Objects in tests:
// e2e/login.spec.ts
import { test, expect } from '@playwright/test';
import { LoginPage } from './pages/LoginPage';
import { DashboardPage } from './pages/DashboardPage';

test('login flow', async ({ page }) => {
  const loginPage = new LoginPage(page);
  const dashboardPage = new DashboardPage(page);

  await loginPage.goto();
  await loginPage.login('user@example.com', 'Password123!');

  await page.waitForURL('/dashboard');
  expect(await dashboardPage.isLoggedIn()).toBe(true);
});
```

---

## Test Database Strategies

### Strategy 1: Separate Test Database

```bash
# .env.test
DATABASE_URL="postgresql://postgres:password@localhost:5432/myapp_test"
REDIS_URL="redis://localhost:6379/1"  # Database number 1 for test
```

### Strategy 2: Testcontainers

```typescript
// src/__tests__/testcontainers.test.ts
import { PostgreSqlContainer, StartedPostgreSqlContainer } from '@testcontainers/postgresql';
import { RedisContainer, StartedRedisContainer } from '@testcontainers/redis';
import { PrismaClient } from '@prisma/client';
import { exec } from 'child_process';
import { promisify } from 'util';

const execAsync = promisify(exec);

describe('With Testcontainers', () => {
  let pgContainer: StartedPostgreSqlContainer;
  let redisContainer: StartedRedisContainer;
  let prisma: PrismaClient;

  beforeAll(async () => {
    // Start PostgreSQL container
    pgContainer = await new PostgreSqlContainer('postgres:16')
      .withDatabase('testdb')
      .withUsername('testuser')
      .withPassword('testpass')
      .start();

    // Start Redis container
    redisContainer = await new RedisContainer('redis:7').start();

    // Run migrations
    const dbUrl = pgContainer.getConnectionUri();
    await execAsync(`DATABASE_URL="${dbUrl}" npx prisma migrate deploy`);

    prisma = new PrismaClient({
      datasources: { db: { url: dbUrl } },
    });
    await prisma.$connect();

    // Set environment for test
    process.env.DATABASE_URL = dbUrl;
    process.env.REDIS_URL = `redis://${redisContainer.getHost()}:${redisContainer.getPort()}`;
  }, 60_000);  // 60 second timeout for container startup

  afterAll(async () => {
    await prisma.$disconnect();
    await pgContainer.stop();
    await redisContainer.stop();
  });

  it('should connect to containerized database', async () => {
    const result = await prisma.$queryRaw`SELECT 1 as value`;
    expect(result).toBeTruthy();
  });

  it('should create and find user', async () => {
    const user = await prisma.user.create({
      data: {
        firstName: 'Container',
        lastName: 'Test',
        email: 'container@test.com',
        password: 'hash',
        role: 'user',
        emailVerified: true,
      },
    });

    const found = await prisma.user.findUnique({ where: { id: user.id } });
    expect(found?.email).toBe('container@test.com');
  });
});
```

---

## Coverage Report

```bash
# Run tests with coverage
npx jest --coverage

# Coverage output:
# ----------|---------|----------|---------|---------|
# File      | % Stmts | % Branch | % Funcs | % Lines |
# ----------|---------|----------|---------|---------|
# All files |   82.45 |    74.38 |   85.71 |   83.12 |
#  services |   90.12 |    82.35 |   91.67 |   90.48 |
#  routes   |   85.71 |    77.78 |   88.89 |   86.25 |
#  utils    |   95.65 |    93.33 |  100.00 |   95.45 |
# ----------|---------|----------|---------|---------|
```

```json
// package.json
{
  "scripts": {
    "test": "jest",
    "test:unit": "jest --testPathPattern='__tests__/unit'",
    "test:integration": "jest --testPathPattern='__tests__/integration'",
    "test:e2e": "playwright test",
    "test:coverage": "jest --coverage",
    "test:watch": "jest --watch",
    "test:ci": "jest --ci --coverage --forceExit"
  }
}
```

---

## CI Integration: GitHub Actions

```yaml
# .github/workflows/test.yml
name: Tests

on: [push, pull_request]

jobs:
  unit-tests:
    name: Unit Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run test:unit -- --coverage
      - uses: codecov/codecov-action@v3
        with:
          token: ${{ secrets.CODECOV_TOKEN }}

  integration-tests:
    name: Integration Tests
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_DB: test_db
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: password
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
      redis:
        image: redis:7
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379
    env:
      DATABASE_URL: postgresql://postgres:password@localhost:5432/test_db
      TEST_DATABASE_URL: postgresql://postgres:password@localhost:5432/test_db
      TEST_REDIS_URL: redis://localhost:6379/1
      JWT_SECRET: test-jwt-secret
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npx prisma migrate deploy
      - run: npm run test:integration
```

---

## สรุป Testing Best Practices

```
Unit Tests:
1. Test ทุก business logic path (happy path + edge cases)
2. Mock ทุก external dependency (DB, Redis, Email, HTTP)
3. ตั้งชื่อ test ให้ชัดเจน: "should [expected behavior] when [condition]"
4. แต่ละ test ควร assert เรื่องเดียว (Single Responsibility)
5. ใช้ beforeEach แทน before เพื่อ isolation

Integration Tests:
1. ใช้ database จริง (test database)
2. Reset state ระหว่าง tests (deleteMany หรือ transaction rollback)
3. Test happy paths และ error cases ผ่าน HTTP
4. ตรวจสอบ DB state หลัง operation
5. Test authentication และ authorization

E2E Tests:
1. Focus ที่ critical user flows เท่านั้น (ไม่ต้อง test ทุกอย่าง)
2. ใช้ data-testid attributes (ไม่ใช้ CSS classes)
3. Page Object Model ช่วย reuse code
4. Parallelize E2E tests ถ้าทำได้
5. Mock external services ใน E2E

Coverage:
- Aim for 80%+ coverage แต่ quality > quantity
- Coverage ≠ correctness (100% coverage ≠ bug-free)
- Focus on covering business logic branches
```

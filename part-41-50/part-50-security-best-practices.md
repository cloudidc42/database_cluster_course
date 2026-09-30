# Part 50: Security Best Practices

## OWASP Top 10 (2021)

| Rank | ชื่อ | คำอธิบาย |
|------|------|----------|
| A01 | Broken Access Control | ผู้ใช้เข้าถึง resource ที่ไม่มีสิทธิ์ |
| A02 | Cryptographic Failures | ข้อมูลสำคัญไม่ถูก encrypt |
| A03 | Injection | SQL, NoSQL, Command injection |
| A04 | Insecure Design | ออกแบบระบบโดยไม่คำนึงถึง security |
| A05 | Security Misconfiguration | Config ไม่ถูกต้อง (default passwords, verbose errors) |
| A06 | Vulnerable Components | ใช้ library เก่าที่มีช่องโหว่ |
| A07 | Auth Failures | Password ง่าย, Session hijacking |
| A08 | Software Integrity Failures | Code/Data ถูกดัดแปลง |
| A09 | Logging Failures | ไม่ log security events |
| A10 | SSRF | Server Side Request Forgery |

---

## A03: Injection Attacks

### SQL Injection

```typescript
// ❌ VULNERABLE: SQL Injection
async function getUserBad(userId: string) {
  // ถ้า userId = "1 OR 1=1" → ได้ users ทั้งหมด
  // ถ้า userId = "1; DROP TABLE users; --" → ลบ table
  const query = `SELECT * FROM users WHERE id = ${userId}`;
  return db.query(query);
}

// ✅ SAFE: Parameterized Query
async function getUserSafe(userId: number) {
  // ✅ ใช้ parameterized query เสมอ
  const result = await db.query(
    'SELECT id, first_name, email FROM users WHERE id = $1',
    [userId],
  );
  return result.rows[0];
}

// ✅ SAFE: Prisma ORM (ป้องกัน SQL injection โดยอัตโนมัติ)
async function getUserPrisma(userId: number) {
  return prisma.user.findUnique({
    where: { id: userId },
    select: { id: true, firstName: true, email: true },
  });
}

// ✅ SAFE: ถ้าต้องการ raw query ใน Prisma
async function searchUsers(searchTerm: string) {
  return prisma.$queryRaw`
    SELECT id, first_name, email 
    FROM users 
    WHERE first_name ILIKE ${'%' + searchTerm + '%'}
    LIMIT 20
  `;
}
```

### NoSQL Injection (Redis)

```typescript
// ❌ VULNERABLE: Redis key injection
async function getUserCacheBad(userId: string) {
  // ถ้า userId = "*" → ดึงทุก key
  const data = await redis.get(`user:${userId}`);
  return data;
}

// ✅ SAFE: Validate and sanitize key
async function getUserCacheSafe(userId: unknown) {
  // ตรวจสอบว่าเป็น positive integer
  const id = parseInt(String(userId));
  if (isNaN(id) || id <= 0) {
    throw new Error('Invalid user ID');
  }

  const data = await redis.get(`user:${id}`);
  return data ? JSON.parse(data) : null;
}

// ✅ SAFE: ใช้ Zod/Joi validate ก่อนใช้
import { z } from 'zod';

const userIdSchema = z.number().int().positive();

async function getUserCacheZod(userId: unknown) {
  const validId = userIdSchema.parse(userId);  // throws if invalid
  return redis.get(`user:${validId}`);
}
```

### Command Injection

```typescript
// ❌ VULNERABLE: Command injection
import { exec } from 'child_process';

async function processFileBad(filename: string) {
  // ถ้า filename = "file.txt; rm -rf /" → อันตราย!
  exec(`convert ${filename} output.png`, (error, stdout) => {
    console.log(stdout);
  });
}

// ✅ SAFE: ใช้ spawn พร้อม array arguments (ไม่ผ่าน shell)
import { spawn } from 'child_process';

async function processFileSafe(filename: string) {
  // Validate filename ก่อน
  if (!/^[a-zA-Z0-9._-]+$/.test(filename)) {
    throw new Error('Invalid filename');
  }

  return new Promise((resolve, reject) => {
    const proc = spawn('convert', [filename, 'output.png'], {
      shell: false,  // ไม่ผ่าน shell!
    });

    proc.on('close', (code) => {
      if (code === 0) resolve(true);
      else reject(new Error(`Process failed with code ${code}`));
    });
  });
}

// ✅ BETTER: หลีกเลี่ยง shell commands ใช้ library แทน
import sharp from 'sharp';  // ใช้ library โดยตรง ไม่ต้องเรียก shell
async function processImageSafe(inputPath: string) {
  return sharp(inputPath)
    .resize(800, 600)
    .toFile('output.jpg');
}
```

---

## A07: Authentication

### Password Hashing ด้วย bcrypt/argon2

```typescript
// middleware/password.ts
import bcrypt from 'bcrypt';
import argon2 from 'argon2';

const BCRYPT_ROUNDS = 12;  // 2^12 = 4096 iterations

// bcrypt (popular, battle-tested)
export async function hashPassword(password: string): Promise<string> {
  return bcrypt.hash(password, BCRYPT_ROUNDS);
}

export async function verifyPassword(password: string, hash: string): Promise<boolean> {
  return bcrypt.compare(password, hash);
}

// argon2 (winner of Password Hashing Competition 2015, แนะนำมากกว่า bcrypt)
export async function hashPasswordArgon2(password: string): Promise<string> {
  return argon2.hash(password, {
    type: argon2.argon2id,  // argon2id ป้องกัน GPU attacks ดีที่สุด
    memoryCost: 65536,       // 64MB
    timeCost: 3,             // 3 iterations
    parallelism: 4,          // 4 threads
  });
}

export async function verifyPasswordArgon2(password: string, hash: string): Promise<boolean> {
  return argon2.verify(hash, password);
}

// Password strength validator
export function validatePasswordStrength(password: string): { valid: boolean; errors: string[] } {
  const errors: string[] = [];

  if (password.length < 8) errors.push('At least 8 characters required');
  if (password.length > 128) errors.push('Maximum 128 characters');
  if (!/[a-z]/.test(password)) errors.push('Must contain lowercase letter');
  if (!/[A-Z]/.test(password)) errors.push('Must contain uppercase letter');
  if (!/\d/.test(password)) errors.push('Must contain number');
  if (!/[!@#$%^&*()_+\-=\[\]{};':"\\|,.<>\/?]/.test(password)) {
    errors.push('Must contain special character');
  }

  // Check common passwords
  const commonPasswords = ['Password123!', 'Admin123!', 'Qwerty123!'];
  if (commonPasswords.includes(password)) errors.push('Password is too common');

  return { valid: errors.length === 0, errors };
}
```

### MFA: TOTP Implementation

```typescript
// services/mfa.service.ts
import speakeasy from 'speakeasy';
import QRCode from 'qrcode';
import { createClient } from 'redis';

const redis = createClient({ url: process.env.REDIS_URL });

export class MFAService {
  // Generate TOTP secret สำหรับ user
  async setupMFA(userId: number, userEmail: string): Promise<{
    secret: string;
    qrCodeDataUrl: string;
    backupCodes: string[];
  }> {
    const secret = speakeasy.generateSecret({
      name: `MyApp (${userEmail})`,
      issuer: 'MyApp',
      length: 32,
    });

    // Generate QR code URL
    const qrCodeDataUrl = await QRCode.toDataURL(secret.otpauth_url!);

    // Generate backup codes
    const backupCodes = Array.from({ length: 10 }, () =>
      Math.random().toString(36).substring(2, 10).toUpperCase(),
    );

    // Store secret temporarily (user must verify before enabling)
    await redis.setEx(
      `mfa:pending:${userId}`,
      600,  // 10 minutes to complete setup
      JSON.stringify({ secret: secret.base32, backupCodes }),
    );

    return {
      secret: secret.base32,
      qrCodeDataUrl,
      backupCodes,
    };
  }

  // Verify TOTP code
  async verifyTOTP(token: string, secret: string): Promise<boolean> {
    return speakeasy.totp.verify({
      secret,
      encoding: 'base32',
      token,
      window: 1,  // Allow 1 step before/after for clock drift
    });
  }

  // Verify backup code
  async verifyBackupCode(userId: number, code: string): Promise<boolean> {
    const backupCodesKey = `mfa:backup:${userId}`;
    const codesJson = await redis.get(backupCodesKey);
    if (!codesJson) return false;

    const codes: string[] = JSON.parse(codesJson);
    const index = codes.indexOf(code.toUpperCase());

    if (index === -1) return false;

    // Remove used backup code
    codes.splice(index, 1);
    await redis.set(backupCodesKey, JSON.stringify(codes));

    return true;
  }
}
```

### Account Lockout (Redis)

```typescript
// services/lockout.service.ts
export class LockoutService {
  private readonly MAX_ATTEMPTS = 5;
  private readonly LOCKOUT_DURATION = 15 * 60;  // 15 minutes
  private readonly ATTEMPT_TTL = 30 * 60;        // 30 minutes

  async recordFailedAttempt(identifier: string): Promise<{
    attempts: number;
    locked: boolean;
    lockedUntil?: Date;
  }> {
    const attemptsKey = `login:attempts:${identifier}`;
    const lockedKey = `login:locked:${identifier}`;

    // ตรวจสอบ lockout
    const isLocked = await redis.exists(lockedKey);
    if (isLocked) {
      const ttl = await redis.ttl(lockedKey);
      return {
        attempts: this.MAX_ATTEMPTS,
        locked: true,
        lockedUntil: new Date(Date.now() + ttl * 1000),
      };
    }

    // นับ attempts
    const attempts = await redis.incr(attemptsKey);
    if (attempts === 1) {
      await redis.expire(attemptsKey, this.ATTEMPT_TTL);
    }

    // Lock ถ้าเกิน limit
    if (attempts >= this.MAX_ATTEMPTS) {
      await redis.setEx(lockedKey, this.LOCKOUT_DURATION, '1');
      await redis.del(attemptsKey);

      return {
        attempts,
        locked: true,
        lockedUntil: new Date(Date.now() + this.LOCKOUT_DURATION * 1000),
      };
    }

    return { attempts, locked: false };
  }

  async clearAttempts(identifier: string): Promise<void> {
    await redis.del(`login:attempts:${identifier}`);
    await redis.del(`login:locked:${identifier}`);
  }

  async isLocked(identifier: string): Promise<boolean> {
    return (await redis.exists(`login:locked:${identifier}`)) === 1;
  }
}

// Usage ใน auth service:
// const lockout = new LockoutService();
// 
// async function login(email, password) {
//   if (await lockout.isLocked(email)) {
//     throw new AppError(423, 'Account is locked', 'ACCOUNT_LOCKED');
//   }
//
//   const user = await findUserByEmail(email);
//   if (!user || !await verifyPassword(password, user.password)) {
//     const result = await lockout.recordFailedAttempt(email);
//     if (result.locked) {
//       throw new AppError(423, 'Account locked', 'ACCOUNT_LOCKED');
//     }
//     throw new AppError(401, 'Invalid credentials');
//   }
//
//   await lockout.clearAttempts(email);
//   return generateTokens(user);
// }
```

---

## A01: Authorization (RBAC & ABAC)

### RBAC: Role-Based Access Control

```typescript
// services/rbac.service.ts

// ตาราง Permissions
type Resource = 'users' | 'posts' | 'comments' | 'admin';
type Action = 'create' | 'read' | 'update' | 'delete' | 'manage';
type Permission = `${Resource}:${Action}`;

// Role definitions
const PERMISSIONS: Record<string, Permission[]> = {
  admin: [
    'users:manage', 'posts:manage', 'comments:manage', 'admin:manage',
  ],
  moderator: [
    'users:read', 'posts:read', 'posts:update', 'posts:delete',
    'comments:read', 'comments:update', 'comments:delete',
  ],
  user: [
    'users:read', 'posts:create', 'posts:read',
    'comments:create', 'comments:read',
  ],
  guest: [
    'posts:read', 'comments:read',
  ],
};

export class RBACService {
  hasPermission(role: string, permission: Permission): boolean {
    const rolePermissions = PERMISSIONS[role] || [];

    // Check direct permission
    if (rolePermissions.includes(permission)) return true;

    // Check wildcard (manage = all actions)
    const [resource] = permission.split(':');
    if (rolePermissions.includes(`${resource}:manage` as Permission)) return true;

    return false;
  }

  getPermissions(role: string): Permission[] {
    return PERMISSIONS[role] || [];
  }
}

// Middleware
export function requirePermission(permission: Permission) {
  return (req: Request, res: Response, next: NextFunction) => {
    if (!req.user) {
      return res.status(401).json({ error: 'Unauthorized' });
    }

    const rbac = new RBACService();
    if (!rbac.hasPermission(req.user.role, permission)) {
      return res.status(403).json({
        error: 'Forbidden',
        required: permission,
      });
    }

    next();
  };
}
```

### Row-Level Security ใน PostgreSQL

```sql
-- PostgreSQL Row-Level Security (RLS)

-- Enable RLS on posts table
ALTER TABLE posts ENABLE ROW LEVEL SECURITY;

-- Policy: users สามารถเห็นเฉพาะ published posts หรือ posts ของตัวเอง
CREATE POLICY posts_select_policy ON posts
  FOR SELECT
  USING (
    published = true
    OR author_id = current_setting('app.current_user_id')::integer
  );

-- Policy: users สามารถ insert เฉพาะ posts ของตัวเอง
CREATE POLICY posts_insert_policy ON posts
  FOR INSERT
  WITH CHECK (
    author_id = current_setting('app.current_user_id')::integer
  );

-- Policy: users สามารถ update เฉพาะ posts ของตัวเอง
CREATE POLICY posts_update_policy ON posts
  FOR UPDATE
  USING (
    author_id = current_setting('app.current_user_id')::integer
  );

-- Policy: users สามารถ delete เฉพาะ posts ของตัวเอง
CREATE POLICY posts_delete_policy ON posts
  FOR DELETE
  USING (
    author_id = current_setting('app.current_user_id')::integer
  );

-- Admin bypass: admin เห็นทุกอย่าง
CREATE POLICY posts_admin_policy ON posts
  USING (
    current_setting('app.current_user_role') = 'admin'
  );
```

```typescript
// Set RLS context ก่อน query
async function queryWithRLS(userId: number, role: string, queryFn: () => Promise<any>) {
  return prisma.$transaction(async (tx) => {
    // Set session variables สำหรับ RLS
    await tx.$executeRaw`SET LOCAL app.current_user_id = ${userId}`;
    await tx.$executeRaw`SET LOCAL app.current_user_role = ${role}`;

    return queryFn();
  });
}

// Usage:
// const posts = await queryWithRLS(req.user.id, req.user.role, () =>
//   prisma.post.findMany()
// );
```

---

## Input Validation แบบ Comprehensive

```typescript
// middleware/validation.ts
import { z } from 'zod';
import DOMPurify from 'isomorphic-dompurify';

// Whitelist validation patterns
const SAFE_STRING = z.string().max(1000).transform(val => val.trim());
const EMAIL = z.string().email().max(254).transform(val => val.toLowerCase());
const URL = z.string().url().max(2048).refine(
  url => /^https?:\/\//.test(url),
  'Only http/https URLs allowed',
);
const PHONE = z.string().regex(/^\+?[1-9]\d{1,14}$/, 'Invalid phone format');
const UUID = z.string().uuid();
const POSITIVE_INT = z.number().int().positive().max(2_147_483_647);
const DATE = z.string().datetime({ offset: true });

// HTML content: sanitize before saving
function sanitizeHtml(html: string): string {
  return DOMPurify.sanitize(html, {
    ALLOWED_TAGS: ['p', 'br', 'strong', 'em', 'ul', 'ol', 'li', 'a'],
    ALLOWED_ATTR: ['href', 'target'],
    ALLOWED_URI_REGEXP: /^https?:\/\//,
  });
}

const POST_SCHEMA = z.object({
  title: z.string().min(1).max(200).transform(val => val.trim()),
  content: z.string().min(1).max(50000).transform(html => sanitizeHtml(html)),
  tags: z.array(z.string().max(50)).max(10).optional(),
  publishedAt: DATE.optional(),
});

// File upload validation
const FILE_UPLOAD = z.object({
  fieldname: z.string(),
  originalname: z.string().regex(/^[a-zA-Z0-9._-]+$/, 'Invalid filename'),
  mimetype: z.enum(['image/jpeg', 'image/png', 'image/webp', 'application/pdf']),
  size: z.number().max(10 * 1024 * 1024, 'File too large (max 10MB)'),
});

// Numeric ID from URL param
function parseId(param: string): number {
  const id = parseInt(param, 10);
  if (isNaN(id) || id <= 0 || id > 2_147_483_647) {
    throw new Error('Invalid ID');
  }
  return id;
}
```

---

## XSS Prevention

```typescript
// middleware/xss-prevention.ts
import helmet from 'helmet';

// 1. Content Security Policy
export const cspMiddleware = helmet.contentSecurityPolicy({
  directives: {
    defaultSrc: ["'self'"],
    scriptSrc: [
      "'self'",
      // ไม่มี 'unsafe-inline' หรือ 'unsafe-eval'
      // ใช้ nonce สำหรับ inline scripts ถ้าจำเป็น
      (req, res) => `'nonce-${(res as any).locals.nonce}'`,
    ],
    styleSrc: ["'self'", "'unsafe-inline'"],  // styles ปลอดภัยกว่า scripts
    imgSrc: ["'self'", 'data:', 'https://trusted-cdn.com'],
    connectSrc: ["'self'", 'https://api.example.com'],
    fontSrc: ["'self'", 'https://fonts.gstatic.com'],
    frameSrc: ["'none'"],
    objectSrc: ["'none'"],
    upgradeInsecureRequests: [],
    reportUri: ['/csp-violation-report'],
  },
  reportOnly: false,  // Set true ระหว่าง testing
});

// 2. HTML Escaping (สำหรับ template engines)
export function escapeHtml(str: string): string {
  const htmlEscapes: Record<string, string> = {
    '&': '&amp;',
    '<': '&lt;',
    '>': '&gt;',
    '"': '&quot;',
    "'": '&#x27;',
    '/': '&#x2F;',
  };
  return str.replace(/[&<>"'/]/g, char => htmlEscapes[char]);
}

// 3. ใช้ DOMPurify สำหรับ HTML content ที่รับจาก user
import createDOMPurify from 'dompurify';
import { JSDOM } from 'jsdom';

const window = new JSDOM('').window;
const DOMPurify = createDOMPurify(window as any);

export function sanitizeUserHtml(html: string): string {
  return DOMPurify.sanitize(html, {
    ALLOWED_TAGS: ['p', 'br', 'strong', 'em', 'u', 'ol', 'ul', 'li', 'blockquote', 'a', 'img'],
    ALLOWED_ATTR: ['href', 'src', 'alt', 'title'],
    ALLOWED_URI_REGEXP: /^https?:\/\/|^mailto:/,
    FORBID_CONTENTS: ['script', 'style', 'iframe'],
    FORCE_BODY: true,
  });
}
```

---

## CSRF Protection

```typescript
// middleware/csrf.ts
import csrf from 'csurf';
import { CookieOptions } from 'express';

// Method 1: CSRF tokens
export const csrfProtection = csrf({
  cookie: {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'strict',
  },
});

// CSRF token endpoint
export function csrfToken(req: Request, res: Response): void {
  res.json({ csrfToken: (req as any).csrfToken() });
}

// Method 2: SameSite cookies (modern approach)
export const secureCookieOptions: CookieOptions = {
  httpOnly: true,            // ป้องกัน JavaScript อ่าน cookie
  secure: process.env.NODE_ENV === 'production',  // HTTPS only in production
  sameSite: 'strict',        // ป้องกัน CSRF (ไม่ส่ง cookie ถ้า cross-site request)
  maxAge: 24 * 60 * 60 * 1000,  // 24 hours
  path: '/',
};

// Method 3: Double Submit Cookie Pattern
import crypto from 'crypto';

export function generateCSRFToken(): string {
  return crypto.randomBytes(32).toString('hex');
}

export function validateCSRFToken(cookieToken: string, headerToken: string): boolean {
  if (!cookieToken || !headerToken) return false;
  // timing-safe comparison
  return crypto.timingSafeEqual(
    Buffer.from(cookieToken),
    Buffer.from(headerToken),
  );
}
```

---

## Secrets Management

```typescript
// config/secrets.ts
import { GetSecretValueCommand, SecretsManagerClient } from '@aws-sdk/client-secrets-manager';

const client = new SecretsManagerClient({ region: 'ap-southeast-1' });

interface AppSecrets {
  DATABASE_URL: string;
  REDIS_URL: string;
  JWT_SECRET: string;
  JWT_REFRESH_SECRET: string;
  SMTP_PASSWORD: string;
}

// Cache secrets ใน memory (ไม่ต้อง call AWS Secrets Manager ทุก request)
let cachedSecrets: AppSecrets | null = null;
let secretsLoadedAt: number = 0;
const CACHE_TTL = 5 * 60 * 1000;  // 5 minutes

export async function getSecrets(): Promise<AppSecrets> {
  // Return cached secrets ถ้ายังไม่หมดอายุ
  if (cachedSecrets && Date.now() - secretsLoadedAt < CACHE_TTL) {
    return cachedSecrets;
  }

  try {
    const command = new GetSecretValueCommand({
      SecretId: process.env.SECRET_NAME || 'myapp/production',
    });

    const response = await client.send(command);
    cachedSecrets = JSON.parse(response.SecretString!);
    secretsLoadedAt = Date.now();

    return cachedSecrets!;
  } catch (error) {
    // Fallback to environment variables (development)
    if (process.env.NODE_ENV !== 'production') {
      return {
        DATABASE_URL: process.env.DATABASE_URL!,
        REDIS_URL: process.env.REDIS_URL!,
        JWT_SECRET: process.env.JWT_SECRET!,
        JWT_REFRESH_SECRET: process.env.JWT_REFRESH_SECRET!,
        SMTP_PASSWORD: process.env.SMTP_PASSWORD!,
      };
    }
    throw error;
  }
}

// ✅ ไม่เคย log secrets
// ✅ ไม่ส่ง secrets ใน response
// ✅ ไม่ใส่ secrets ใน URL query parameters
// ✅ Rotate secrets regularly (เปลี่ยนทุก 90 วัน)
```

---

## Network Security

### HTTPS และ TLS

```typescript
// server.ts - HTTPS setup
import https from 'https';
import http from 'http';
import fs from 'fs';
import express from 'express';

const app = express();

// Redirect HTTP → HTTPS
const httpApp = express();
httpApp.use((req, res) => {
  res.redirect(301, `https://${req.headers.host}${req.url}`);
});
http.createServer(httpApp).listen(80);

// HTTPS server
const options = {
  key: fs.readFileSync('/etc/ssl/private/server.key'),
  cert: fs.readFileSync('/etc/ssl/certs/server.crt'),
  ca: fs.readFileSync('/etc/ssl/certs/ca.crt'),
  // TLS 1.3 only (ปลอดภัยที่สุด)
  minVersion: 'TLSv1.3' as const,
  // Strong cipher suites
  ciphers: [
    'TLS_AES_256_GCM_SHA384',
    'TLS_CHACHA20_POLY1305_SHA256',
    'TLS_AES_128_GCM_SHA256',
  ].join(':'),
};

https.createServer(options, app).listen(443, () => {
  console.log('HTTPS server running on port 443');
});
```

---

## PostgreSQL Security

```sql
-- PostgreSQL Security Configuration

-- 1. สร้าง role ที่มีสิทธิ์น้อยที่สุด (Least Privilege)
CREATE ROLE app_user WITH LOGIN PASSWORD 'strong_password';

-- ให้ permission เฉพาะที่จำเป็น
GRANT CONNECT ON DATABASE myapp TO app_user;
GRANT USAGE ON SCHEMA public TO app_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_user;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO app_user;

-- ไม่ให้ DDL rights (CREATE TABLE, DROP TABLE)
-- ไม่ให้ superuser access

-- 2. Readonly role สำหรับ reporting
CREATE ROLE app_readonly WITH LOGIN PASSWORD 'readonly_password';
GRANT CONNECT ON DATABASE myapp TO app_readonly;
GRANT USAGE ON SCHEMA public TO app_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_readonly;

-- 3. SSL connections บังคับ
-- ใน postgresql.conf:
-- ssl = on
-- ssl_cert_file = 'server.crt'
-- ssl_key_file = 'server.key'

-- ใน pg_hba.conf:
-- hostssl myapp app_user 0.0.0.0/0 scram-sha-256

-- 4. pg_audit สำหรับ audit logging
CREATE EXTENSION IF NOT EXISTS pgaudit;

-- ใน postgresql.conf:
-- pgaudit.log = 'write, ddl'
-- pgaudit.log_client = on
-- pgaudit.log_level = log

-- 5. ตรวจสอบ connection ที่ผิดปกติ
SELECT
  pid,
  usename,
  application_name,
  client_addr,
  state,
  query_start,
  query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY query_start;
```

```typescript
// database connection ด้วย SSL
import { Pool } from 'pg';
import fs from 'fs';

const pool = new Pool({
  host: process.env.DB_HOST,
  port: parseInt(process.env.DB_PORT || '5432'),
  database: process.env.DB_NAME,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  ssl: process.env.NODE_ENV === 'production' ? {
    rejectUnauthorized: true,
    ca: fs.readFileSync('/etc/ssl/certs/ca.crt').toString(),
    cert: fs.readFileSync('/etc/ssl/certs/client.crt').toString(),
    key: fs.readFileSync('/etc/ssl/private/client.key').toString(),
  } : false,
  max: 20,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 5000,
});
```

---

## Redis Security

```bash
# redis.conf - Security configuration

# 1. Password authentication
requirepass "your_very_strong_redis_password_here"

# 2. Bind to localhost only (ไม่ expose ออก internet)
bind 127.0.0.1 -::1

# 3. Rename dangerous commands
rename-command FLUSHALL ""           # ปิดใช้
rename-command FLUSHDB "FLUSHDB_SAFE_f7x9k2"   # เปลี่ยนชื่อ
rename-command CONFIG "CONFIG_SAFE_a3m8p1"
rename-command KEYS ""               # ปิดใช้ KEYS (ใช้ SCAN แทน)
rename-command DEBUG ""              # ปิดใช้
rename-command SHUTDOWN ""           # ปิดใช้

# 4. TLS/SSL
tls-port 6380
tls-cert-file /path/to/redis.crt
tls-key-file /path/to/redis.key
tls-ca-cert-file /path/to/ca.crt

# 5. ACL (Redis 6+) - ละเอียดกว่า requirepass
aclfile /etc/redis/users.acl
# ใน users.acl:
# user default off
# user appuser on >password ~app:* +@read +@write +SET +GET +DEL
# user readonly on >readonly_pass ~* +@read
```

```typescript
// redis client ด้วย TLS
import { createClient } from 'redis';
import fs from 'fs';
import tls from 'tls';

const redis = createClient({
  url: `rediss://redis.internal:6380`,  // rediss:// = TLS
  socket: {
    tls: true,
    cert: fs.readFileSync('/etc/ssl/redis/client.crt'),
    key: fs.readFileSync('/etc/ssl/redis/client.key'),
    ca: fs.readFileSync('/etc/ssl/redis/ca.crt'),
    rejectUnauthorized: true,
  },
  password: process.env.REDIS_PASSWORD,
  username: 'appuser',  // ACL username
});

await redis.connect();
```

---

## Dependency Scanning

```bash
# npm audit - check for vulnerabilities
npm audit

# Auto-fix vulnerabilities
npm audit fix

# แสดงเฉพาะ critical/high vulnerabilities
npm audit --audit-level=high

# ดู audit report แบบ JSON
npm audit --json

# Snyk (more detailed scanning)
npx snyk test
npx snyk monitor  # monitor continuously
```

```yaml
# .github/workflows/security-scan.yml
name: Security Scan

on:
  push:
    branches: [main]
  schedule:
    - cron: '0 6 * * 1'  # Every Monday at 6am

jobs:
  dependency-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - run: npm ci

      - name: npm audit
        run: npm audit --audit-level=high
        continue-on-error: false

      - name: Snyk scan
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high --json-file-output=snyk-results.json

      - name: Upload Snyk results
        uses: github/codeql-action/upload-sarif@v2
        if: always()
        with:
          sarif_file: snyk-results.json

  codeql-analysis:
    runs-on: ubuntu-latest
    permissions:
      security-events: write
    steps:
      - uses: actions/checkout@v4
      - name: Initialize CodeQL
        uses: github/codeql-action/init@v3
        with:
          languages: javascript
      - name: Autobuild
        uses: github/codeql-action/autobuild@v3
      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v3
```

---

## Security Checklist แบบสมบูรณ์

```typescript
// security-checklist.ts - Security audit script

const SECURITY_CHECKLIST = {
  authentication: [
    '✅ Password hashed ด้วย bcrypt/argon2 (ไม่ใช้ MD5/SHA1)',
    '✅ Password strength validation (min 8 chars, upper/lower/number/special)',
    '✅ Account lockout หลัง failed attempts (5 ครั้งใน 15 นาที)',
    '✅ MFA support (TOTP)',
    '✅ Secure password reset (token-based, single use, short expiry)',
    '✅ JWT signed ด้วย strong secret (RS256 สำหรับ production)',
    '✅ JWT short expiry (15-60 min) + refresh token',
    '✅ Refresh token rotation',
    '✅ Token revocation/blacklist ใน Redis',
    '✅ Session invalidation เมื่อ logout',
  ],

  authorization: [
    '✅ RBAC implemented',
    '✅ Resource ownership checks',
    '✅ Admin endpoints ป้องกัน',
    '✅ API rate limiting',
    '✅ Row-Level Security ใน PostgreSQL',
    '✅ Sensitive operations require re-authentication',
  ],

  inputValidation: [
    '✅ Validate ทุก input ก่อนใช้งาน',
    '✅ Parameterized queries (ไม่ string concatenation)',
    '✅ HTML sanitization สำหรับ user content',
    '✅ File upload validation (type, size, malware scan)',
    '✅ Request body size limit',
    '✅ URL/path traversal prevention',
    '✅ JSON depth limit',
  ],

  headers: [
    '✅ Helmet.js (security headers)',
    '✅ Content-Security-Policy',
    '✅ HSTS (HTTP Strict Transport Security)',
    '✅ X-Frame-Options: DENY',
    '✅ X-Content-Type-Options: nosniff',
    '✅ Referrer-Policy',
    '✅ Permissions-Policy',
    '✅ Remove X-Powered-By header',
  ],

  data: [
    '✅ HTTPS everywhere (TLS 1.3)',
    '✅ Database connections ใช้ SSL',
    '✅ Sensitive data encrypted at rest',
    '✅ PII data ไม่ถูก log',
    '✅ Secrets ใน environment variables / Secrets Manager',
    '✅ Secrets ไม่อยู่ใน git',
    '✅ Database backups encrypted',
  ],

  monitoring: [
    '✅ Security event logging (login, logout, failed auth)',
    '✅ Anomaly detection (unusual access patterns)',
    '✅ Alert on multiple failed logins',
    '✅ Alert on unusual data access',
    '✅ Dependency vulnerability scanning (weekly)',
    '✅ Penetration testing (quarterly)',
    '✅ Security headers validation (securityheaders.com)',
  ],

  infrastructure: [
    '✅ Database ไม่ expose ออก internet',
    '✅ Redis ไม่ expose ออก internet',
    '✅ Firewall rules ที่ restrictive',
    '✅ Non-root Docker containers',
    '✅ Image vulnerability scanning',
    '✅ Network segmentation',
    '✅ VPN สำหรับ production access',
  ],
};

function printChecklist() {
  for (const [category, items] of Object.entries(SECURITY_CHECKLIST)) {
    console.log(`\n=== ${category.toUpperCase()} ===`);
    items.forEach(item => console.log(item));
  }

  const totalItems = Object.values(SECURITY_CHECKLIST).flat().length;
  const completedItems = Object.values(SECURITY_CHECKLIST)
    .flat()
    .filter(item => item.startsWith('✅')).length;

  console.log(`\nScore: ${completedItems}/${totalItems} (${Math.round(completedItems / totalItems * 100)}%)`);
}

printChecklist();
```

---

## Secure Error Handling

```typescript
// ❌ อย่า expose internal errors
app.use((err: Error, req: Request, res: Response, next: NextFunction) => {
  // อย่าทำแบบนี้ใน production!
  res.status(500).json({
    error: err.message,      // ❌ อาจ expose database errors, file paths
    stack: err.stack,        // ❌ อย่าส่ง stack trace ออก
    query: (err as any).query,  // ❌ SQL query อาจมีข้อมูลสำคัญ
  });
});

// ✅ Error handling ที่ปลอดภัย
app.use((err: Error, req: Request, res: Response, next: NextFunction) => {
  // Log ข้อมูลจริงใน server (ไม่ส่งหา client)
  console.error({
    requestId: (req as any).requestId,
    error: err.message,
    stack: err.stack,
    url: req.originalUrl,
    userId: (req as any).user?.id,
  });

  // Send ข้อมูลน้อยที่สุดที่จำเป็น
  if (process.env.NODE_ENV === 'production') {
    res.status(500).json({
      error: 'Internal server error',
      requestId: (req as any).requestId,  // ให้ support ใช้ track ได้
    });
  } else {
    // Development: ให้ข้อมูลเพิ่มเติม
    res.status(500).json({
      error: err.message,
      stack: err.stack,
    });
  }
});
```

---

## สรุป: Security Mindset

```
"Security is not a feature, it's a requirement"

หลักการสำคัญ:
1. Defense in Depth: หลาย layer of security
2. Least Privilege: ให้สิทธิ์น้อยที่สุดที่จำเป็น
3. Zero Trust: ไม่ trust ใคร ต้องตรวจสอบเสมอ
4. Fail Secure: ถ้า error → deny access (ไม่ใช่ allow)
5. Security by Default: default settings ต้องปลอดภัย

Top Security Mistakes:
1. SQL Injection จาก string concatenation
2. Storing passwords ใน plaintext
3. JWT secrets ที่ weak หรือ default
4. Trusting user input โดยไม่ validate
5. Exposing stack traces ใน production
6. Hardcoded secrets ใน code
7. Using outdated libraries ที่มีช่องโหว่
8. Misconfigured CORS (allow *)
9. No rate limiting → brute force attacks
10. Missing security headers
```

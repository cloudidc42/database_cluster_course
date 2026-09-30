# Part 16: Authentication ด้วย JWT + PostgreSQL

## สารบัญ
1. [Authentication vs Authorization](#1-authentication-vs-authorization)
2. [Password Hashing ด้วย bcrypt](#2-password-hashing-ด้วย-bcrypt)
3. [JWT คืออะไร](#3-jwt-คืออะไร)
4. [Access Token vs Refresh Token](#4-access-token-vs-refresh-token)
5. [JWT Secret Key: Symmetric vs Asymmetric](#5-jwt-secret-key-symmetric-vs-asymmetric)
6. [PostgreSQL Users Table Design](#6-postgresql-users-table-design)
7. [Register Endpoint](#7-register-endpoint)
8. [Login Endpoint](#8-login-endpoint)
9. [Auth Middleware](#9-auth-middleware)
10. [Refresh Token Endpoint](#10-refresh-token-endpoint)
11. [Logout ด้วย Redis Blacklist](#11-logout-ด้วย-redis-blacklist)
12. [Password Reset Flow](#12-password-reset-flow)
13. [Role-Based Access Control (RBAC)](#13-role-based-access-control-rbac)
14. [Rate Limiting ด้วย Redis](#14-rate-limiting-ด้วย-redis)
15. [Full Implementation](#15-full-implementation)

---

## 1. Authentication vs Authorization

### Authentication (การพิสูจน์ตัวตน)
Authentication คือกระบวนการตรวจสอบว่า "คุณเป็นใคร?" — ยืนยันตัวตนของผู้ใช้ว่าเป็นคนที่อ้างว่าเป็นจริงหรือไม่

**ตัวอย่าง:**
- Login ด้วย username/password
- Login ด้วย Google OAuth
- การสแกน Face ID หรือ Fingerprint
- การใส่ OTP (One-Time Password)

### Authorization (การอนุญาต)
Authorization คือกระบวนการตรวจสอบว่า "คุณมีสิทธิ์ทำอะไร?" — ตรวจสอบว่าผู้ใช้ที่ผ่านการ authenticate แล้วมีสิทธิ์เข้าถึงทรัพยากรหรือทำงานนั้นหรือไม่

**ตัวอย่าง:**
- Admin เท่านั้นที่ลบ user ได้
- User สามารถแก้ไขเฉพาะโปรไฟล์ของตัวเองได้
- Premium user เข้าถึง feature พิเศษได้

```
Authentication Flow:
┌─────────────┐     credentials      ┌─────────────┐
│   Client    │ ──────────────────> │   Server    │
│             │ <────────────────── │             │
└─────────────┘     token/session    └─────────────┘

Authorization Flow:
┌─────────────┐  token + request    ┌─────────────┐  check permission  ┌──────────┐
│   Client    │ ──────────────────> │   Server    │ ──────────────────> │   DB/   │
│             │ <────────────────── │             │ <────────────────── │  Policy  │
└─────────────┘  allowed/denied     └─────────────┘    allow/deny       └──────────┘
```

---

## 2. Password Hashing ด้วย bcrypt

### ทำไมต้องใช้ bcrypt?

ห้าม **เก็บ password เป็น plain text** เด็ดขาด! หากฐานข้อมูลรั่ว ผู้ใช้ทุกคนจะได้รับผลกระทบ

**bcrypt** คือ password-hashing function ที่:
- ใช้ **salt** เพื่อป้องกัน rainbow table attacks
- ปรับ **cost factor** ได้เพื่อให้ช้าขึ้นตามพลังคอมพิวเตอร์ที่เพิ่มขึ้น
- เป็น one-way function (ไม่สามารถ reverse ได้)

### ติดตั้ง

```bash
npm install bcrypt
npm install --save-dev @types/bcrypt
```

### การใช้งาน bcrypt

```typescript
import bcrypt from 'bcrypt';

// กำหนด cost factor (rounds)
// แต่ละ round เพิ่มเวลา 2x
// round 10 = ~100ms, round 12 = ~400ms, round 14 = ~1.5s
const SALT_ROUNDS = 12;

// Hash password
async function hashPassword(plainPassword: string): Promise<string> {
  const hashedPassword = await bcrypt.hash(plainPassword, SALT_ROUNDS);
  return hashedPassword;
}

// Verify password
async function verifyPassword(
  plainPassword: string,
  hashedPassword: string
): Promise<boolean> {
  const isMatch = await bcrypt.compare(plainPassword, hashedPassword);
  return isMatch;
}

// ตัวอย่างการใช้งาน
async function example() {
  const password = 'mySecretPassword123!';
  
  // Hash
  const hash = await hashPassword(password);
  console.log('Hashed:', hash);
  // Output: $2b$12$LQv3c1yqBWVHxkd0LHAkCOYz6TtxMQJqhN8/lewdBenyfmZLL...
  
  // Verify correct password
  const isValid = await verifyPassword(password, hash);
  console.log('Valid:', isValid); // true
  
  // Verify wrong password
  const isInvalid = await verifyPassword('wrongPassword', hash);
  console.log('Invalid:', isInvalid); // false
}
```

### bcrypt Hash Format

```
$2b$12$LQv3c1yqBWVHxkd0LHAkCOYz6TtxMQJqhN8/lewdBenyfmZLLa6sQ
 │   │  │                    │
 │   │  └── Salt (22 chars)  └── Hash (31 chars)
 │   └── Cost factor (12)
 └── Version (2b)
```

---

## 3. JWT คืออะไร

JWT (JSON Web Token) คือ open standard (RFC 7519) สำหรับส่งข้อมูลที่ signed อย่างปลอดภัยระหว่าง parties ในรูปแบบ JSON

### โครงสร้างของ JWT

JWT ประกอบด้วย 3 ส่วน คั่นด้วย `.`

```
xxxxx.yyyyy.zzzzz
  │      │      │
Header  Payload  Signature
```

**1. Header** — ระบุ algorithm และ token type
```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

**2. Payload** — ข้อมูลที่ต้องการส่ง (Claims)
```json
{
  "sub": "user_123",
  "email": "john@example.com",
  "role": "user",
  "iat": 1700000000,
  "exp": 1700003600
}
```

**3. Signature** — ลายเซ็นเพื่อยืนยันความถูกต้อง
```
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret
)
```

### Standard Claims (Registered Claims)

| Claim | ย่อมาจาก | ความหมาย |
|-------|----------|----------|
| `iss` | Issuer | ผู้ออก token |
| `sub` | Subject | ผู้ใช้ token (user id) |
| `aud` | Audience | ผู้รับ token |
| `exp` | Expiration | วันหมดอายุ (Unix timestamp) |
| `nbf` | Not Before | ใช้ได้ตั้งแต่เมื่อไหร่ |
| `iat` | Issued At | เวลาที่สร้าง |
| `jti` | JWT ID | unique identifier |

### ติดตั้ง

```bash
npm install jsonwebtoken
npm install --save-dev @types/jsonwebtoken
```

### การใช้งาน JWT

```typescript
import jwt from 'jsonwebtoken';

const SECRET_KEY = process.env.JWT_SECRET || 'your-secret-key';

interface TokenPayload {
  userId: string;
  email: string;
  role: string;
}

// สร้าง token
function generateToken(payload: TokenPayload, expiresIn: string = '1h'): string {
  return jwt.sign(payload, SECRET_KEY, {
    expiresIn,
    issuer: 'myapp',
    audience: 'myapp-users',
  });
}

// ตรวจสอบ token
function verifyToken(token: string): TokenPayload {
  const decoded = jwt.verify(token, SECRET_KEY, {
    issuer: 'myapp',
    audience: 'myapp-users',
  }) as TokenPayload;
  return decoded;
}

// Decode โดยไม่ verify (สำหรับ debug เท่านั้น)
function decodeToken(token: string) {
  return jwt.decode(token, { complete: true });
}

// ตัวอย่าง
const token = generateToken({
  userId: 'user_123',
  email: 'john@example.com',
  role: 'user'
});

console.log('Token:', token);

try {
  const payload = verifyToken(token);
  console.log('Decoded:', payload);
} catch (error) {
  if (error instanceof jwt.TokenExpiredError) {
    console.log('Token หมดอายุแล้ว');
  } else if (error instanceof jwt.JsonWebTokenError) {
    console.log('Token ไม่ถูกต้อง');
  }
}
```

---

## 4. Access Token vs Refresh Token

### ปัญหาของ Single Token

ถ้าใช้ token เดียว:
- **อายุสั้น**: ผู้ใช้ต้อง login บ่อย → UX แย่
- **อายุยาว**: ถ้า token หลุด → ผู้ไม่ประสงค์ดีใช้ได้นาน → ไม่ปลอดภัย

### วิธีแก้: Two-Token Strategy

```
┌─────────────────────────────────────────────────────┐
│                  Token Strategy                      │
├─────────────────────┬───────────────────────────────┤
│   Access Token      │   Refresh Token               │
├─────────────────────┼───────────────────────────────┤
│ อายุสั้น (15-60 min) │ อายุยาว (7-30 days)          │
│ ส่งทุก request      │ ส่งเฉพาะตอน refresh           │
│ เก็บใน memory       │ เก็บใน httpOnly cookie         │
│ ไม่เก็บใน DB        │ เก็บใน DB (ตรวจสอบได้)        │
│ Stateless           │ Stateful (revocable)           │
└─────────────────────┴───────────────────────────────┘
```

### Refresh Token Flow

```
Client                    API Server                  Database
  │                           │                           │
  │ POST /auth/login           │                           │
  │ ──────────────────────────>│                           │
  │                           │ ตรวจสอบ credentials       │
  │                           │ ──────────────────────────>│
  │                           │ <──────────────────────────│
  │                           │ สร้าง access_token (15m)   │
  │                           │ สร้าง refresh_token (30d)  │
  │                           │ บันทึก refresh_token ใน DB │
  │                           │ ──────────────────────────>│
  │ access_token + refresh_token │                         │
  │ <──────────────────────────│                           │
  │                           │                           │
  │ (15 นาทีผ่านไป)            │                           │
  │                           │                           │
  │ POST /auth/refresh         │                           │
  │ (ส่ง refresh_token)        │                           │
  │ ──────────────────────────>│                           │
  │                           │ ตรวจสอบ refresh_token ใน DB│
  │                           │ ──────────────────────────>│
  │                           │ <──────────────────────────│
  │                           │ สร้าง access_token ใหม่   │
  │ new access_token           │                           │
  │ <──────────────────────────│                           │
```

---

## 5. JWT Secret Key: Symmetric vs Asymmetric

### Symmetric (HS256) — กุญแจเดียว

```typescript
// ใช้ secret key เดียวกันทั้ง sign และ verify
const secret = 'super-secret-key-min-32-chars-long!!';

const token = jwt.sign({ userId: '123' }, secret, { algorithm: 'HS256' });
const decoded = jwt.verify(token, secret);
```

**เหมาะสำหรับ:** Monolith หรือ services ที่เชื่อถือกัน (internal)

**ข้อเสีย:** ทุก service ต้องรู้ secret key → ถ้า service หนึ่งถูก compromise ทั้งหมดเสี่ยง

### Asymmetric (RS256) — คู่กุญแจ

```bash
# สร้าง private key
openssl genrsa -out private.key 2048

# สร้าง public key จาก private key
openssl rsa -in private.key -pubout -out public.key
```

```typescript
import fs from 'fs';

const privateKey = fs.readFileSync('./private.key');
const publicKey = fs.readFileSync('./public.key');

// Sign ด้วย private key (ทำได้เฉพาะ Auth service)
const token = jwt.sign(
  { userId: '123', email: 'user@example.com' },
  privateKey,
  { algorithm: 'RS256', expiresIn: '1h' }
);

// Verify ด้วย public key (ทุก service ทำได้)
const decoded = jwt.verify(token, publicKey, { algorithms: ['RS256'] });
```

**เหมาะสำหรับ:** Microservices — Auth service เก็บ private key, services อื่นรู้แค่ public key

---

## 6. PostgreSQL Users Table Design

### Schema Design

```sql
-- Enable UUID extension
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Roles table
CREATE TABLE roles (
  id   SERIAL PRIMARY KEY,
  name VARCHAR(50) UNIQUE NOT NULL,
  description TEXT
);

INSERT INTO roles (name, description) VALUES
  ('admin', 'System administrator'),
  ('user', 'Regular user'),
  ('moderator', 'Content moderator');

-- Users table
CREATE TABLE users (
  id            UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
  email         VARCHAR(255) UNIQUE NOT NULL,
  username      VARCHAR(100) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  role_id       INTEGER REFERENCES roles(id) DEFAULT 2,  -- default: user
  
  -- Profile
  first_name    VARCHAR(100),
  last_name     VARCHAR(100),
  avatar_url    TEXT,
  
  -- Status
  is_active     BOOLEAN DEFAULT true,
  is_verified   BOOLEAN DEFAULT false,
  
  -- Timestamps
  created_at    TIMESTAMPTZ DEFAULT NOW(),
  updated_at    TIMESTAMPTZ DEFAULT NOW(),
  last_login_at TIMESTAMPTZ
);

-- Refresh tokens table
CREATE TABLE refresh_tokens (
  id         UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
  user_id    UUID REFERENCES users(id) ON DELETE CASCADE NOT NULL,
  token_hash VARCHAR(255) UNIQUE NOT NULL,  -- เก็บ hash ไม่เก็บ token ตรงๆ
  
  -- Device info
  device_info JSONB,  -- { userAgent, ip, deviceName }
  
  -- Status
  is_revoked  BOOLEAN DEFAULT false,
  
  -- Timestamps
  created_at  TIMESTAMPTZ DEFAULT NOW(),
  expires_at  TIMESTAMPTZ NOT NULL,
  revoked_at  TIMESTAMPTZ
);

-- Email verification tokens
CREATE TABLE email_tokens (
  id         UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
  user_id    UUID REFERENCES users(id) ON DELETE CASCADE NOT NULL,
  token      VARCHAR(255) UNIQUE NOT NULL,
  type       VARCHAR(50) NOT NULL,  -- 'verification', 'password_reset'
  expires_at TIMESTAMPTZ NOT NULL,
  used_at    TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_refresh_tokens_user_id ON refresh_tokens(user_id);
CREATE INDEX idx_refresh_tokens_token_hash ON refresh_tokens(token_hash);
CREATE INDEX idx_email_tokens_token ON email_tokens(token);

-- Auto update updated_at
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER update_users_updated_at
  BEFORE UPDATE ON users
  FOR EACH ROW EXECUTE PROCEDURE update_updated_at_column();
```

---

## 7. Register Endpoint

```typescript
// src/routes/auth.ts
import { Router, Request, Response, NextFunction } from 'express';
import { z } from 'zod';
import bcrypt from 'bcrypt';
import { v4 as uuidv4 } from 'uuid';
import pool from '../db/pool';
import { generateTokens } from '../utils/jwt';

const router = Router();

// Validation schema
const registerSchema = z.object({
  email: z.string().email('Email ไม่ถูกต้อง'),
  username: z
    .string()
    .min(3, 'Username ต้องมีอย่างน้อย 3 ตัวอักษร')
    .max(50, 'Username ต้องไม่เกิน 50 ตัวอักษร')
    .regex(/^[a-zA-Z0-9_]+$/, 'Username ใช้ได้เฉพาะ a-z, A-Z, 0-9, _'),
  password: z
    .string()
    .min(8, 'Password ต้องมีอย่างน้อย 8 ตัวอักษร')
    .regex(/[A-Z]/, 'Password ต้องมีตัวพิมพ์ใหญ่อย่างน้อย 1 ตัว')
    .regex(/[0-9]/, 'Password ต้องมีตัวเลขอย่างน้อย 1 ตัว'),
  firstName: z.string().min(1).max(100).optional(),
  lastName: z.string().min(1).max(100).optional(),
});

// POST /auth/register
router.post('/register', async (req: Request, res: Response, next: NextFunction) => {
  try {
    // 1. Validate input
    const validatedData = registerSchema.parse(req.body);
    const { email, username, password, firstName, lastName } = validatedData;

    // 2. ตรวจสอบว่า email/username ซ้ำหรือเปล่า
    const existingUser = await pool.query(
      'SELECT id FROM users WHERE email = $1 OR username = $2',
      [email.toLowerCase(), username.toLowerCase()]
    );

    if (existingUser.rows.length > 0) {
      return res.status(409).json({
        success: false,
        error: {
          code: 'USER_ALREADY_EXISTS',
          message: 'Email หรือ Username นี้ถูกใช้ไปแล้ว',
        },
      });
    }

    // 3. Hash password
    const passwordHash = await bcrypt.hash(password, 12);

    // 4. สร้าง user
    const newUser = await pool.query(
      `INSERT INTO users (email, username, password_hash, first_name, last_name)
       VALUES ($1, $2, $3, $4, $5)
       RETURNING id, email, username, role_id, is_active, is_verified, created_at`,
      [email.toLowerCase(), username.toLowerCase(), passwordHash, firstName, lastName]
    );

    const user = newUser.rows[0];

    // 5. สร้าง tokens
    const { accessToken, refreshToken } = await generateTokens(user, req);

    // 6. Response
    res.status(201).json({
      success: true,
      data: {
        user: {
          id: user.id,
          email: user.email,
          username: user.username,
        },
        accessToken,
        refreshToken,
      },
    });
  } catch (error) {
    next(error);
  }
});

export default router;
```

---

## 8. Login Endpoint

```typescript
// ต่อจาก auth.ts

const loginSchema = z.object({
  emailOrUsername: z.string().min(1, 'กรุณาระบุ email หรือ username'),
  password: z.string().min(1, 'กรุณาระบุ password'),
});

// POST /auth/login
router.post('/login', async (req: Request, res: Response, next: NextFunction) => {
  try {
    // 1. Validate
    const { emailOrUsername, password } = loginSchema.parse(req.body);

    // 2. ค้นหา user (โดย email หรือ username)
    const result = await pool.query(
      `SELECT u.*, r.name as role_name
       FROM users u
       JOIN roles r ON u.role_id = r.id
       WHERE (u.email = $1 OR u.username = $1) AND u.is_active = true`,
      [emailOrUsername.toLowerCase()]
    );

    const user = result.rows[0];

    // 3. ตรวจสอบ user มีอยู่จริงหรือไม่
    // ใช้ dummy comparison เพื่อป้องกัน timing attack
    if (!user) {
      await bcrypt.compare(password, '$2b$12$invalid.hash.for.timing.attack.prevention');
      return res.status(401).json({
        success: false,
        error: {
          code: 'INVALID_CREDENTIALS',
          message: 'Email/Username หรือ Password ไม่ถูกต้อง',
        },
      });
    }

    // 4. ตรวจสอบ password
    const isPasswordValid = await bcrypt.compare(password, user.password_hash);

    if (!isPasswordValid) {
      // บันทึก failed login attempt
      await logFailedAttempt(user.id, req.ip);
      return res.status(401).json({
        success: false,
        error: {
          code: 'INVALID_CREDENTIALS',
          message: 'Email/Username หรือ Password ไม่ถูกต้อง',
        },
      });
    }

    // 5. อัพเดท last login
    await pool.query(
      'UPDATE users SET last_login_at = NOW() WHERE id = $1',
      [user.id]
    );

    // 6. สร้าง tokens
    const { accessToken, refreshToken } = await generateTokens(user, req);

    // 7. Set refresh token ใน httpOnly cookie
    res.cookie('refreshToken', refreshToken, {
      httpOnly: true,
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'strict',
      maxAge: 30 * 24 * 60 * 60 * 1000, // 30 days
    });

    res.json({
      success: true,
      data: {
        user: {
          id: user.id,
          email: user.email,
          username: user.username,
          role: user.role_name,
        },
        accessToken,
        expiresIn: 3600, // seconds
      },
    });
  } catch (error) {
    next(error);
  }
});

async function logFailedAttempt(userId: string, ip: string) {
  // บันทึกใน Redis สำหรับ rate limiting
  // (ดูเพิ่มเติมใน section rate limiting)
}
```

---

## 9. Auth Middleware

```typescript
// src/middleware/auth.ts
import { Request, Response, NextFunction } from 'express';
import jwt from 'jsonwebtoken';
import { redis } from '../db/redis';

interface JwtPayload {
  userId: string;
  email: string;
  role: string;
  jti: string;  // JWT ID สำหรับตรวจสอบ blacklist
}

// ขยาย Request type เพื่อรองรับ user
declare global {
  namespace Express {
    interface Request {
      user?: {
        id: string;
        email: string;
        role: string;
      };
    }
  }
}

export const authenticate = async (
  req: Request,
  res: Response,
  next: NextFunction
) => {
  try {
    // 1. ดึง token จาก Authorization header
    const authHeader = req.headers.authorization;

    if (!authHeader || !authHeader.startsWith('Bearer ')) {
      return res.status(401).json({
        success: false,
        error: {
          code: 'NO_TOKEN',
          message: 'กรุณา login ก่อน',
        },
      });
    }

    const token = authHeader.substring(7); // ตัด "Bearer " ออก

    // 2. Verify token
    const decoded = jwt.verify(
      token,
      process.env.JWT_SECRET!
    ) as JwtPayload;

    // 3. ตรวจสอบ blacklist ใน Redis
    const isBlacklisted = await redis.get(`blacklist:${decoded.jti}`);

    if (isBlacklisted) {
      return res.status(401).json({
        success: false,
        error: {
          code: 'TOKEN_REVOKED',
          message: 'Token ถูกยกเลิกแล้ว กรุณา login ใหม่',
        },
      });
    }

    // 4. Attach user ไว้ใน request
    req.user = {
      id: decoded.userId,
      email: decoded.email,
      role: decoded.role,
    };

    next();
  } catch (error) {
    if (error instanceof jwt.TokenExpiredError) {
      return res.status(401).json({
        success: false,
        error: {
          code: 'TOKEN_EXPIRED',
          message: 'Token หมดอายุแล้ว กรุณา refresh token',
        },
      });
    }

    if (error instanceof jwt.JsonWebTokenError) {
      return res.status(401).json({
        success: false,
        error: {
          code: 'INVALID_TOKEN',
          message: 'Token ไม่ถูกต้อง',
        },
      });
    }

    next(error);
  }
};

// Authorize by roles
export const authorize = (...roles: string[]) => {
  return (req: Request, res: Response, next: NextFunction) => {
    if (!req.user) {
      return res.status(401).json({
        success: false,
        error: { code: 'NOT_AUTHENTICATED', message: 'กรุณา login ก่อน' },
      });
    }

    if (!roles.includes(req.user.role)) {
      return res.status(403).json({
        success: false,
        error: {
          code: 'FORBIDDEN',
          message: 'คุณไม่มีสิทธิ์เข้าถึง resource นี้',
        },
      });
    }

    next();
  };
};

// ใช้งาน:
// router.get('/admin', authenticate, authorize('admin'), handler)
// router.get('/profile', authenticate, handler)
```

---

## 10. Refresh Token Endpoint

```typescript
// src/utils/jwt.ts
import jwt from 'jsonwebtoken';
import crypto from 'crypto';
import pool from '../db/pool';
import { Request } from 'express';

export interface TokenPair {
  accessToken: string;
  refreshToken: string;
}

export async function generateTokens(user: any, req: Request): Promise<TokenPair> {
  const jti = uuidv4(); // unique token ID

  // Access token (15 minutes)
  const accessToken = jwt.sign(
    {
      userId: user.id,
      email: user.email,
      role: user.role_name || user.role,
      jti,
    },
    process.env.JWT_SECRET!,
    { expiresIn: '15m', issuer: 'myapp' }
  );

  // Refresh token (30 days) — random bytes, ไม่ใช่ JWT
  const refreshToken = crypto.randomBytes(64).toString('hex');
  const refreshTokenHash = crypto
    .createHash('sha256')
    .update(refreshToken)
    .digest('hex');

  // บันทึก refresh token ใน DB
  const expiresAt = new Date(Date.now() + 30 * 24 * 60 * 60 * 1000);

  await pool.query(
    `INSERT INTO refresh_tokens (user_id, token_hash, device_info, expires_at)
     VALUES ($1, $2, $3, $4)`,
    [
      user.id,
      refreshTokenHash,
      JSON.stringify({
        userAgent: req.headers['user-agent'],
        ip: req.ip,
      }),
      expiresAt,
    ]
  );

  return { accessToken, refreshToken };
}

// POST /auth/refresh
router.post('/refresh', async (req: Request, res: Response, next: NextFunction) => {
  try {
    // รับ refresh token จาก cookie หรือ body
    const refreshToken = req.cookies.refreshToken || req.body.refreshToken;

    if (!refreshToken) {
      return res.status(401).json({
        success: false,
        error: { code: 'NO_REFRESH_TOKEN', message: 'ไม่พบ refresh token' },
      });
    }

    // Hash refresh token เพื่อค้นหาใน DB
    const tokenHash = crypto
      .createHash('sha256')
      .update(refreshToken)
      .digest('hex');

    // ค้นหาใน DB
    const result = await pool.query(
      `SELECT rt.*, u.id as user_id, u.email, u.is_active, r.name as role_name
       FROM refresh_tokens rt
       JOIN users u ON rt.user_id = u.id
       JOIN roles r ON u.role_id = r.id
       WHERE rt.token_hash = $1
         AND rt.is_revoked = false
         AND rt.expires_at > NOW()
         AND u.is_active = true`,
      [tokenHash]
    );

    if (result.rows.length === 0) {
      return res.status(401).json({
        success: false,
        error: {
          code: 'INVALID_REFRESH_TOKEN',
          message: 'Refresh token ไม่ถูกต้องหรือหมดอายุแล้ว',
        },
      });
    }

    const tokenRecord = result.rows[0];

    // Revoke old refresh token (Rotation)
    await pool.query(
      'UPDATE refresh_tokens SET is_revoked = true, revoked_at = NOW() WHERE id = $1',
      [tokenRecord.id]
    );

    // สร้าง token คู่ใหม่
    const { accessToken, refreshToken: newRefreshToken } = await generateTokens(
      tokenRecord,
      req
    );

    // Set new cookie
    res.cookie('refreshToken', newRefreshToken, {
      httpOnly: true,
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'strict',
      maxAge: 30 * 24 * 60 * 60 * 1000,
    });

    res.json({
      success: true,
      data: {
        accessToken,
        expiresIn: 900, // 15 minutes
      },
    });
  } catch (error) {
    next(error);
  }
});
```

---

## 11. Logout ด้วย Redis Blacklist

```typescript
// src/routes/auth.ts ต่อ

// POST /auth/logout
router.post('/logout', authenticate, async (req: Request, res: Response, next: NextFunction) => {
  try {
    // 1. ดึง token จาก header
    const token = req.headers.authorization!.substring(7);
    const decoded = jwt.decode(token) as any;

    if (decoded?.jti) {
      // 2. ใส่ access token ใน blacklist จนกว่าจะหมดอายุ
      const ttl = decoded.exp - Math.floor(Date.now() / 1000);
      if (ttl > 0) {
        await redis.set(`blacklist:${decoded.jti}`, '1', 'EX', ttl);
      }
    }

    // 3. Revoke refresh token
    const refreshToken = req.cookies.refreshToken;

    if (refreshToken) {
      const tokenHash = crypto
        .createHash('sha256')
        .update(refreshToken)
        .digest('hex');

      await pool.query(
        'UPDATE refresh_tokens SET is_revoked = true, revoked_at = NOW() WHERE token_hash = $1',
        [tokenHash]
      );
    }

    // 4. Clear cookie
    res.clearCookie('refreshToken');

    res.json({
      success: true,
      message: 'Logout สำเร็จ',
    });
  } catch (error) {
    next(error);
  }
});

// POST /auth/logout-all (logout จากทุก device)
router.post('/logout-all', authenticate, async (req: Request, res: Response, next: NextFunction) => {
  try {
    const userId = req.user!.id;

    // Revoke refresh tokens ทั้งหมด
    await pool.query(
      'UPDATE refresh_tokens SET is_revoked = true, revoked_at = NOW() WHERE user_id = $1 AND is_revoked = false',
      [userId]
    );

    // Blacklist access token ปัจจุบัน
    const token = req.headers.authorization!.substring(7);
    const decoded = jwt.decode(token) as any;

    if (decoded?.jti) {
      const ttl = decoded.exp - Math.floor(Date.now() / 1000);
      if (ttl > 0) {
        await redis.set(`blacklist:${decoded.jti}`, '1', 'EX', ttl);
      }
    }

    res.clearCookie('refreshToken');

    res.json({
      success: true,
      message: 'Logout จากทุก device สำเร็จ',
    });
  } catch (error) {
    next(error);
  }
});
```

---

## 12. Password Reset Flow

```typescript
import nodemailer from 'nodemailer';
import crypto from 'crypto';

// ส่ง email reset password
router.post('/forgot-password', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const { email } = z.object({ email: z.string().email() }).parse(req.body);

    const result = await pool.query(
      'SELECT id, email FROM users WHERE email = $1 AND is_active = true',
      [email.toLowerCase()]
    );

    // ส่ง response เหมือนกันไม่ว่าจะเจอ email หรือเปล่า (prevent enumeration)
    const successResponse = {
      success: true,
      message: 'ถ้า email นี้มีในระบบ คุณจะได้รับ email สำหรับ reset password',
    };

    if (result.rows.length === 0) {
      return res.json(successResponse);
    }

    const user = result.rows[0];

    // สร้าง reset token
    const resetToken = crypto.randomBytes(32).toString('hex');
    const resetTokenHash = crypto
      .createHash('sha256')
      .update(resetToken)
      .digest('hex');

    // บันทึกลง DB (หมดอายุใน 1 ชั่วโมง)
    await pool.query(
      `INSERT INTO email_tokens (user_id, token, type, expires_at)
       VALUES ($1, $2, 'password_reset', NOW() + INTERVAL '1 hour')
       ON CONFLICT DO NOTHING`,
      [user.id, resetTokenHash]
    );

    // ส่ง email
    const resetUrl = `${process.env.FRONTEND_URL}/reset-password?token=${resetToken}`;

    await sendResetEmail(user.email, resetUrl);

    res.json(successResponse);
  } catch (error) {
    next(error);
  }
});

// รีเซ็ต password
router.post('/reset-password', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const { token, newPassword } = z
      .object({
        token: z.string().min(1),
        newPassword: z
          .string()
          .min(8)
          .regex(/[A-Z]/)
          .regex(/[0-9]/),
      })
      .parse(req.body);

    const tokenHash = crypto.createHash('sha256').update(token).digest('hex');

    // ตรวจสอบ token
    const result = await pool.query(
      `SELECT et.*, u.id as user_id
       FROM email_tokens et
       JOIN users u ON et.user_id = u.id
       WHERE et.token = $1
         AND et.type = 'password_reset'
         AND et.expires_at > NOW()
         AND et.used_at IS NULL`,
      [tokenHash]
    );

    if (result.rows.length === 0) {
      return res.status(400).json({
        success: false,
        error: { code: 'INVALID_TOKEN', message: 'Token ไม่ถูกต้องหรือหมดอายุแล้ว' },
      });
    }

    const tokenRecord = result.rows[0];

    // Hash new password
    const passwordHash = await bcrypt.hash(newPassword, 12);

    // อัพเดท password
    await pool.query(
      'UPDATE users SET password_hash = $1 WHERE id = $2',
      [passwordHash, tokenRecord.user_id]
    );

    // Mark token as used
    await pool.query(
      'UPDATE email_tokens SET used_at = NOW() WHERE id = $1',
      [tokenRecord.id]
    );

    // Revoke refresh tokens ทั้งหมด (force re-login)
    await pool.query(
      'UPDATE refresh_tokens SET is_revoked = true WHERE user_id = $1',
      [tokenRecord.user_id]
    );

    res.json({
      success: true,
      message: 'Reset password สำเร็จ กรุณา login ด้วย password ใหม่',
    });
  } catch (error) {
    next(error);
  }
});

async function sendResetEmail(email: string, resetUrl: string) {
  const transporter = nodemailer.createTransport({
    host: process.env.SMTP_HOST,
    port: Number(process.env.SMTP_PORT),
    auth: {
      user: process.env.SMTP_USER,
      pass: process.env.SMTP_PASS,
    },
  });

  await transporter.sendMail({
    from: 'noreply@myapp.com',
    to: email,
    subject: 'Reset Password Request',
    html: `
      <h2>Reset Password</h2>
      <p>คลิกที่ลิงก์ด้านล่างเพื่อ reset password ของคุณ</p>
      <a href="${resetUrl}">Reset Password</a>
      <p>ลิงก์นี้จะหมดอายุใน 1 ชั่วโมง</p>
      <p>ถ้าคุณไม่ได้ขอ reset password กรุณาเพิกเฉยต่อ email นี้</p>
    `,
  });
}
```

---

## 13. Role-Based Access Control (RBAC)

```typescript
// src/middleware/rbac.ts

// Permission definitions
const permissions = {
  // User permissions
  'user:read': ['user', 'moderator', 'admin'],
  'user:update_own': ['user', 'moderator', 'admin'],
  'user:update_any': ['admin'],
  'user:delete': ['admin'],

  // Post permissions
  'post:create': ['user', 'moderator', 'admin'],
  'post:read': ['user', 'moderator', 'admin'],
  'post:update_own': ['user', 'moderator', 'admin'],
  'post:update_any': ['moderator', 'admin'],
  'post:delete_own': ['user', 'moderator', 'admin'],
  'post:delete_any': ['moderator', 'admin'],

  // Admin permissions
  'admin:dashboard': ['admin'],
  'admin:manage_users': ['admin'],
} as const;

type Permission = keyof typeof permissions;

export function requirePermission(permission: Permission) {
  return (req: Request, res: Response, next: NextFunction) => {
    const userRole = req.user?.role;

    if (!userRole) {
      return res.status(401).json({
        success: false,
        error: { code: 'NOT_AUTHENTICATED' },
      });
    }

    const allowedRoles = permissions[permission] as readonly string[];

    if (!allowedRoles.includes(userRole)) {
      return res.status(403).json({
        success: false,
        error: {
          code: 'INSUFFICIENT_PERMISSION',
          message: `ต้องการสิทธิ์ ${permission}`,
        },
      });
    }

    next();
  };
}

// Resource-level check (เจ้าของ resource หรือ admin)
export function requireOwnerOrAdmin(getResourceUserId: (req: Request) => Promise<string>) {
  return async (req: Request, res: Response, next: NextFunction) => {
    try {
      const currentUserId = req.user?.id;
      const currentRole = req.user?.role;

      if (currentRole === 'admin') {
        return next(); // Admin ทำได้ทุกอย่าง
      }

      const resourceUserId = await getResourceUserId(req);

      if (currentUserId !== resourceUserId) {
        return res.status(403).json({
          success: false,
          error: { code: 'NOT_RESOURCE_OWNER', message: 'คุณไม่มีสิทธิ์แก้ไข resource นี้' },
        });
      }

      next();
    } catch (error) {
      next(error);
    }
  };
}

// ใช้งาน:
router.delete(
  '/posts/:id',
  authenticate,
  requirePermission('post:delete_any'),
  deletePostHandler
);

router.put(
  '/posts/:id',
  authenticate,
  requireOwnerOrAdmin(async (req) => {
    const post = await pool.query('SELECT user_id FROM posts WHERE id = $1', [req.params.id]);
    return post.rows[0]?.user_id;
  }),
  updatePostHandler
);
```

---

## 14. Rate Limiting ด้วย Redis

```typescript
// src/middleware/rateLimiter.ts
import { redis } from '../db/redis';
import { Request, Response, NextFunction } from 'express';

interface RateLimitOptions {
  windowMs: number;    // ช่วงเวลา (milliseconds)
  maxRequests: number; // จำนวน request สูงสุด
  keyPrefix: string;   // prefix สำหรับ Redis key
}

export function createRateLimiter(options: RateLimitOptions) {
  const { windowMs, maxRequests, keyPrefix } = options;
  const windowSecs = Math.ceil(windowMs / 1000);

  return async (req: Request, res: Response, next: NextFunction) => {
    // ใช้ IP เป็น key (หรือ user ID ถ้า authenticated)
    const identifier = req.user?.id || req.ip;
    const key = `${keyPrefix}:${identifier}`;

    try {
      // Increment counter
      const current = await redis.incr(key);

      // Set expiry ถ้าเป็นครั้งแรก
      if (current === 1) {
        await redis.expire(key, windowSecs);
      }

      // คำนวณเวลาที่เหลือ
      const ttl = await redis.ttl(key);

      // Set headers
      res.set({
        'X-RateLimit-Limit': String(maxRequests),
        'X-RateLimit-Remaining': String(Math.max(0, maxRequests - current)),
        'X-RateLimit-Reset': String(Date.now() + ttl * 1000),
      });

      if (current > maxRequests) {
        return res.status(429).json({
          success: false,
          error: {
            code: 'RATE_LIMIT_EXCEEDED',
            message: `คุณ request มากเกินไป กรุณารอ ${ttl} วินาที`,
            retryAfter: ttl,
          },
        });
      }

      next();
    } catch (error) {
      // ถ้า Redis ล้มเหลว ให้ผ่านไปได้เลย (fail open)
      console.error('Rate limiter error:', error);
      next();
    }
  };
}

// Login rate limiter — เข้มงวดกว่า
export const loginRateLimiter = createRateLimiter({
  windowMs: 15 * 60 * 1000, // 15 นาที
  maxRequests: 5,            // 5 ครั้งต่อ 15 นาที
  keyPrefix: 'login_attempts',
});

// General API rate limiter
export const apiRateLimiter = createRateLimiter({
  windowMs: 60 * 1000,  // 1 นาที
  maxRequests: 100,
  keyPrefix: 'api',
});

// Register ใช้ limiter ที่ strict กว่า
export const registerRateLimiter = createRateLimiter({
  windowMs: 60 * 60 * 1000, // 1 ชั่วโมง
  maxRequests: 5,
  keyPrefix: 'register',
});
```

---

## 15. Full Implementation

### โครงสร้างโปรเจกต์

```
src/
├── app.ts
├── server.ts
├── config/
│   └── env.ts
├── db/
│   ├── pool.ts
│   └── redis.ts
├── middleware/
│   ├── auth.ts
│   ├── rbac.ts
│   └── rateLimiter.ts
├── routes/
│   └── auth.ts
└── utils/
    └── jwt.ts
```

### src/config/env.ts

```typescript
import { z } from 'zod';

const envSchema = z.object({
  NODE_ENV: z.enum(['development', 'production', 'test']).default('development'),
  PORT: z.string().default('3000'),

  // Database
  DATABASE_URL: z.string().url(),

  // Redis
  REDIS_URL: z.string().url(),

  // JWT
  JWT_SECRET: z.string().min(32, 'JWT_SECRET ต้องมีอย่างน้อย 32 ตัวอักษร'),

  // Email
  SMTP_HOST: z.string().optional(),
  SMTP_PORT: z.string().optional(),
  SMTP_USER: z.string().optional(),
  SMTP_PASS: z.string().optional(),

  // App
  FRONTEND_URL: z.string().url().default('http://localhost:5173'),
});

export const env = envSchema.parse(process.env);
```

### src/db/pool.ts

```typescript
import { Pool } from 'pg';
import { env } from '../config/env';

const pool = new Pool({
  connectionString: env.DATABASE_URL,
  max: 20,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});

pool.on('error', (err) => {
  console.error('Unexpected error on idle client', err);
  process.exit(-1);
});

export default pool;
```

### src/db/redis.ts

```typescript
import Redis from 'ioredis';
import { env } from '../config/env';

export const redis = new Redis(env.REDIS_URL, {
  maxRetriesPerRequest: 3,
  lazyConnect: true,
});

redis.on('error', (err) => {
  console.error('Redis error:', err);
});

redis.on('connect', () => {
  console.log('Connected to Redis');
});
```

### src/app.ts

```typescript
import express from 'express';
import cookieParser from 'cookie-parser';
import cors from 'cors';
import helmet from 'helmet';
import authRouter from './routes/auth';
import { errorHandler } from './middleware/errorHandler';
import { apiRateLimiter } from './middleware/rateLimiter';

const app = express();

// Security
app.use(helmet());
app.use(cors({
  origin: process.env.FRONTEND_URL,
  credentials: true, // สำคัญสำหรับ cookie
}));

// Parsing
app.use(express.json());
app.use(express.urlencoded({ extended: true }));
app.use(cookieParser());

// Rate limiting
app.use('/api', apiRateLimiter);

// Routes
app.use('/api/auth', authRouter);

// Health check
app.get('/health', (req, res) => {
  res.json({ status: 'ok', timestamp: new Date().toISOString() });
});

// Error handler (ต้องอยู่สุดท้าย)
app.use(errorHandler);

export default app;
```

### ทดสอบ API

```bash
# Register
curl -X POST http://localhost:3000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "john@example.com",
    "username": "johndoe",
    "password": "Password123!",
    "firstName": "John",
    "lastName": "Doe"
  }'

# Login
curl -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -c cookies.txt \
  -d '{
    "emailOrUsername": "johndoe",
    "password": "Password123!"
  }'

# Protected route
curl -X GET http://localhost:3000/api/profile \
  -H "Authorization: Bearer <access_token>"

# Refresh token
curl -X POST http://localhost:3000/api/auth/refresh \
  -b cookies.txt

# Logout
curl -X POST http://localhost:3000/api/auth/logout \
  -H "Authorization: Bearer <access_token>" \
  -b cookies.txt

# Forgot password
curl -X POST http://localhost:3000/api/auth/forgot-password \
  -H "Content-Type: application/json" \
  -d '{"email": "john@example.com"}'
```

### .env ตัวอย่าง

```env
NODE_ENV=development
PORT=3000
DATABASE_URL=postgresql://user:password@localhost:5432/myapp
REDIS_URL=redis://localhost:6379
JWT_SECRET=your-super-secret-key-must-be-at-least-32-chars-long
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your@gmail.com
SMTP_PASS=your-app-password
FRONTEND_URL=http://localhost:5173
```

### package.json dependencies

```json
{
  "dependencies": {
    "bcrypt": "^5.1.1",
    "cookie-parser": "^1.4.6",
    "cors": "^2.8.5",
    "express": "^4.18.2",
    "helmet": "^7.1.0",
    "ioredis": "^5.3.2",
    "jsonwebtoken": "^9.0.2",
    "nodemailer": "^6.9.8",
    "pg": "^8.11.3",
    "uuid": "^9.0.1",
    "zod": "^3.22.4"
  },
  "devDependencies": {
    "@types/bcrypt": "^5.0.2",
    "@types/cookie-parser": "^1.4.6",
    "@types/cors": "^2.8.17",
    "@types/express": "^4.17.21",
    "@types/jsonwebtoken": "^9.0.5",
    "@types/node": "^20.11.5",
    "@types/nodemailer": "^6.4.14",
    "@types/pg": "^8.10.9",
    "@types/uuid": "^9.0.7",
    "typescript": "^5.3.3"
  }
}
```

---

## สรุป

| หัวข้อ | สิ่งสำคัญ |
|--------|----------|
| Password | ใช้ bcrypt + salt round 12+ |
| Access Token | JWT อายุสั้น (15-60 min) |
| Refresh Token | Random bytes อายุยาว (7-30 days) เก็บใน DB |
| Blacklist | เก็บ revoked JWT JTI ใน Redis |
| Cookies | httpOnly + secure + sameSite=strict |
| Rate Limiting | Redis counter สำหรับ login attempts |
| RBAC | ตรวจสอบ role ก่อนเข้าถึง resource |

**Security Checklist:**
- [ ] ใช้ HTTPS ใน production
- [ ] เก็บ JWT_SECRET ใน environment variable
- [ ] ไม่เก็บ sensitive data ใน JWT payload
- [ ] ใส่ rate limiting ที่ login endpoint
- [ ] ใช้ httpOnly cookie สำหรับ refresh token
- [ ] Rotate refresh token ทุกครั้งที่ใช้
- [ ] Log failed login attempts

# Part 17: Session Management ด้วย Redis

## สารบัญ
1. [Session vs JWT: เปรียบเทียบ](#1-session-vs-jwt-เปรียบเทียบ)
2. [express-session ด้วย connect-redis](#2-express-session-ด้วย-connect-redis)
3. [Session Schema Design ใน Redis](#3-session-schema-design-ใน-redis)
4. [Session CRUD Operations](#4-session-crud-operations)
5. [Session Expiration และ Sliding Expiration](#5-session-expiration-และ-sliding-expiration)
6. [Secure Session Cookies](#6-secure-session-cookies)
7. [Multiple Device Sessions](#7-multiple-device-sessions)
8. [Session Invalidation](#8-session-invalidation)
9. [Session Data: User Info และ Preferences](#9-session-data-user-info-และ-preferences)
10. [Session Fixation Prevention](#10-session-fixation-prevention)
11. [CSRF Protection](#11-csrf-protection)
12. [TypeScript Implementation](#12-typescript-implementation)
13. [Full Example: Shopping Cart](#13-full-example-shopping-cart)

---

## 1. Session vs JWT: เปรียบเทียบ

### Server-Side Sessions

```
ผู้ใช้ Login
     │
     ▼
Server สร้าง Session ID (random string)
Server เก็บข้อมูล Session ใน Redis:
  session:{id} = { userId, email, role, ... }
     │
     ▼
ส่ง Session ID กลับไปใน Cookie
     │
     ▼
Request ถัดไป: ส่ง Cookie → Server ค้นหาใน Redis
```

### JWT (Stateless)

```
ผู้ใช้ Login
     │
     ▼
Server สร้าง JWT (เข้ารหัสข้อมูลไว้ใน token)
ไม่ได้เก็บอะไรใน Server
     │
     ▼
ส่ง JWT กลับไปใน Response
     │
     ▼
Request ถัดไป: ส่ง JWT ใน Header → Server verify เอง
```

### ตารางเปรียบเทียบ

| ลักษณะ | Session (Redis) | JWT |
|--------|----------------|-----|
| **ที่เก็บข้อมูล** | Server (Redis) | Client (token) |
| **Scalability** | ต้อง shared Redis | ดีกว่า (stateless) |
| **Revocation** | ง่าย (ลบ session) | ยาก (ต้อง blacklist) |
| **ข้อมูลที่เก็บ** | เก็บได้เยอะ | จำกัด (token ใหญ่) |
| **Performance** | Redis round-trip | Verify locally |
| **Complexity** | ง่ายกว่า | ซับซ้อนกว่า (refresh logic) |
| **Real-time updates** | ทันที (แก้ session) | รอ token หมดอายุ |
| **Sensitive data** | ปลอดภัย (ไม่ส่งไป client) | ระวัง (client อ่านได้) |

### เมื่อไหร่ใช้อะไร?

**ใช้ Session เมื่อ:**
- ต้องการ revoke session ทันที (banking, security-critical)
- มีข้อมูลเยอะต้องเก็บ per-user
- ต้องการ real-time permission updates
- ต้องการ server-side control เต็มที่

**ใช้ JWT เมื่อ:**
- Microservices (หลาย services ต้อง authenticate)
- Mobile apps / Third-party clients
- Stateless architecture

---

## 2. express-session ด้วย connect-redis

### ติดตั้ง packages

```bash
npm install express-session connect-redis ioredis
npm install --save-dev @types/express-session
```

### การตั้งค่าพื้นฐาน

```typescript
// src/config/session.ts
import session from 'express-session';
import { createClient } from 'redis';
import RedisStore from 'connect-redis';

// สร้าง Redis client
const redisClient = createClient({
  url: process.env.REDIS_URL || 'redis://localhost:6379',
});

redisClient.connect().catch(console.error);

redisClient.on('error', (err) => console.error('Redis Client Error', err));
redisClient.on('connect', () => console.log('Session Redis connected'));

// สร้าง Redis Store
const redisStore = new RedisStore({
  client: redisClient,
  prefix: 'sess:',       // prefix สำหรับ key ใน Redis
  ttl: 86400,            // 1 day ใน seconds
});

// Session configuration
export const sessionConfig = session({
  store: redisStore,
  secret: process.env.SESSION_SECRET || 'your-session-secret-key',
  resave: false,          // ไม่ save session ถ้าไม่มีการเปลี่ยนแปลง
  saveUninitialized: false, // ไม่ save session เปล่า
  name: 'sessionId',      // ชื่อ cookie (ไม่ใช้ default 'connect.sid')
  cookie: {
    httpOnly: true,       // JavaScript อ่านไม่ได้
    secure: process.env.NODE_ENV === 'production', // HTTPS เท่านั้น
    sameSite: 'lax',      // ป้องกัน CSRF
    maxAge: 24 * 60 * 60 * 1000, // 1 day ใน milliseconds
  },
});

export { redisClient };
```

### ใช้ใน Express App

```typescript
// src/app.ts
import express from 'express';
import { sessionConfig } from './config/session';

const app = express();

app.use(express.json());
app.use(sessionConfig);

// ขยาย Session type
declare module 'express-session' {
  interface SessionData {
    userId?: string;
    email?: string;
    role?: string;
    deviceId?: string;
    createdAt?: string;
    lastActivity?: string;
    preferences?: {
      language: string;
      theme: string;
      notifications: boolean;
    };
    cart?: CartItem[];
  }
}

interface CartItem {
  productId: string;
  name: string;
  price: number;
  quantity: number;
}
```

---

## 3. Session Schema Design ใน Redis

### Data ที่เก็บใน Session

```
Redis Key: sess:{session_id}
Redis Value: JSON string
TTL: กำหนดตาม session duration

ตัวอย่าง:
Key: sess:abc123def456
Value: {
  "cookie": {
    "originalMaxAge": 86400000,
    "expires": "2024-12-31T00:00:00.000Z",
    "secure": true,
    "httpOnly": true,
    "sameSite": "lax"
  },
  "userId": "user_uuid_here",
  "email": "john@example.com",
  "role": "user",
  "deviceId": "device_abc",
  "createdAt": "2024-01-01T00:00:00.000Z",
  "lastActivity": "2024-01-01T12:00:00.000Z",
  "preferences": {
    "language": "th",
    "theme": "dark",
    "notifications": true
  }
}
TTL: 86400 (seconds)
```

### Session Metadata (แยกเก็บ)

```
เพื่อ track multiple sessions per user:

Key: user_sessions:{user_id}
Value: Set ของ session IDs
ตัวอย่าง: ["sess_abc", "sess_def", "sess_ghi"]
```

---

## 4. Session CRUD Operations

```typescript
// src/services/sessionService.ts
import { redisClient } from '../config/session';
import { Request } from 'express';
import { v4 as uuidv4 } from 'uuid';

export class SessionService {
  private readonly SESSION_PREFIX = 'sess:';
  private readonly USER_SESSIONS_PREFIX = 'user_sessions:';

  // สร้าง session หลัง login
  async createSession(req: Request, userData: {
    userId: string;
    email: string;
    role: string;
  }): Promise<void> {
    const deviceId = uuidv4();

    // เขียนข้อมูลลง session
    req.session.userId = userData.userId;
    req.session.email = userData.email;
    req.session.role = userData.role;
    req.session.deviceId = deviceId;
    req.session.createdAt = new Date().toISOString();
    req.session.lastActivity = new Date().toISOString();

    // บันทึก session ID ไว้กับ user (สำหรับ multi-device)
    await this.addToUserSessions(userData.userId, req.session.id);
  }

  // อ่าน session
  getSessionData(req: Request) {
    if (!req.session.userId) return null;

    return {
      userId: req.session.userId,
      email: req.session.email,
      role: req.session.role,
      deviceId: req.session.deviceId,
      createdAt: req.session.createdAt,
      lastActivity: req.session.lastActivity,
    };
  }

  // อัพเดท session data
  async updateSession(req: Request, data: Partial<{
    email: string;
    role: string;
    preferences: any;
    lastActivity: string;
  }>): Promise<void> {
    Object.assign(req.session, data);
    req.session.lastActivity = new Date().toISOString();

    // บังคับ save
    await new Promise<void>((resolve, reject) => {
      req.session.save((err) => {
        if (err) reject(err);
        else resolve();
      });
    });
  }

  // ลบ session (logout)
  async destroySession(req: Request): Promise<void> {
    const userId = req.session.userId;
    const sessionId = req.session.id;

    await new Promise<void>((resolve, reject) => {
      req.session.destroy((err) => {
        if (err) reject(err);
        else resolve();
      });
    });

    // ลบออกจาก user sessions tracking
    if (userId) {
      await this.removeFromUserSessions(userId, sessionId);
    }
  }

  // ดู sessions ทั้งหมดของ user
  async getUserSessions(userId: string): Promise<string[]> {
    const key = `${this.USER_SESSIONS_PREFIX}${userId}`;
    const sessions = await redisClient.sMembers(key);
    return sessions;
  }

  // เพิ่ม session ID เข้า user's sessions
  async addToUserSessions(userId: string, sessionId: string): Promise<void> {
    const key = `${this.USER_SESSIONS_PREFIX}${userId}`;
    await redisClient.sAdd(key, sessionId);
    // Set TTL เผื่อ user ไม่ logout
    await redisClient.expire(key, 30 * 24 * 60 * 60); // 30 days
  }

  // ลบ session ID ออกจาก user's sessions
  async removeFromUserSessions(userId: string, sessionId: string): Promise<void> {
    const key = `${this.USER_SESSIONS_PREFIX}${userId}`;
    await redisClient.sRem(key, sessionId);
  }

  // ลบ sessions ทั้งหมดของ user
  async destroyAllUserSessions(userId: string): Promise<void> {
    const sessions = await this.getUserSessions(userId);

    // ลบแต่ละ session
    const deletePromises = sessions.map((sessionId) =>
      redisClient.del(`${this.SESSION_PREFIX}${sessionId}`)
    );
    await Promise.all(deletePromises);

    // ลบ tracking key
    await redisClient.del(`${this.USER_SESSIONS_PREFIX}${userId}`);
  }

  // ดูรายละเอียด session ทั้งหมดของ user
  async getUserSessionDetails(userId: string): Promise<any[]> {
    const sessionIds = await this.getUserSessions(userId);

    const sessionPromises = sessionIds.map(async (sessionId) => {
      const key = `${this.SESSION_PREFIX}${sessionId}`;
      const data = await redisClient.get(key);
      if (!data) return null;

      try {
        const session = JSON.parse(data);
        return {
          sessionId,
          deviceId: session.deviceId,
          createdAt: session.createdAt,
          lastActivity: session.lastActivity,
        };
      } catch {
        return null;
      }
    });

    const sessions = await Promise.all(sessionPromises);
    return sessions.filter(Boolean);
  }
}

export const sessionService = new SessionService();
```

---

## 5. Session Expiration และ Sliding Expiration

### Fixed Expiration

```typescript
// Session หมดอายุหลังจาก maxAge นับจากตอนสร้าง
// ไม่ขยายออกไปถึงแม้จะมีการใช้งาน
cookie: {
  maxAge: 30 * 60 * 1000, // 30 นาทีแน่นอน
}
```

### Sliding Expiration (Idle Timeout)

```typescript
// Session หมดอายุหลังจากไม่มีการใช้งาน N นาที
// ทุก request ที่ active จะ reset timer

// src/middleware/slidingSession.ts
import { Request, Response, NextFunction } from 'express';

const IDLE_TIMEOUT_MS = 30 * 60 * 1000; // 30 นาที

export function slidingSessionMiddleware(
  req: Request,
  res: Response,
  next: NextFunction
) {
  if (!req.session.userId) {
    return next(); // ไม่มี session ข้ามไป
  }

  const lastActivity = req.session.lastActivity
    ? new Date(req.session.lastActivity).getTime()
    : 0;

  const now = Date.now();
  const idleTime = now - lastActivity;

  // ถ้า idle นานเกินไป ให้ destroy session
  if (idleTime > IDLE_TIMEOUT_MS) {
    req.session.destroy((err) => {
      if (err) console.error('Session destroy error:', err);
    });

    return res.status(401).json({
      success: false,
      error: {
        code: 'SESSION_EXPIRED',
        message: 'Session หมดอายุเนื่องจากไม่มีการใช้งาน กรุณา login ใหม่',
      },
    });
  }

  // อัพเดท lastActivity และ extend cookie
  req.session.lastActivity = new Date().toISOString();

  // Extend cookie expiry
  if (req.session.cookie) {
    req.session.cookie.maxAge = IDLE_TIMEOUT_MS;
  }

  next();
}
```

### Absolute Expiration + Sliding

```typescript
// รวมทั้ง 2 แบบ: session หมดอายุตาม idle timeout
// แต่ไม่เกิน absolute max duration

const MAX_SESSION_DURATION_MS = 24 * 60 * 60 * 1000; // 1 วัน
const IDLE_TIMEOUT_MS = 30 * 60 * 1000;               // 30 นาที

export function advancedSessionMiddleware(req: Request, res: Response, next: NextFunction) {
  if (!req.session.userId) return next();

  const now = Date.now();

  // ตรวจสอบ absolute expiration
  const sessionAge = now - new Date(req.session.createdAt!).getTime();
  if (sessionAge > MAX_SESSION_DURATION_MS) {
    return req.session.destroy(() => {
      res.status(401).json({
        success: false,
        error: { code: 'SESSION_ABSOLUTE_EXPIRED', message: 'Session หมดอายุแล้ว' },
      });
    });
  }

  // ตรวจสอบ idle timeout
  const idleTime = now - new Date(req.session.lastActivity!).getTime();
  if (idleTime > IDLE_TIMEOUT_MS) {
    return req.session.destroy(() => {
      res.status(401).json({
        success: false,
        error: { code: 'SESSION_IDLE_TIMEOUT', message: 'Session หมดอายุเนื่องจาก idle' },
      });
    });
  }

  req.session.lastActivity = new Date().toISOString();
  next();
}
```

---

## 6. Secure Session Cookies

```typescript
// การตั้งค่า cookie อย่างปลอดภัย

cookie: {
  // ป้องกัน XSS: JavaScript (document.cookie) อ่านไม่ได้
  httpOnly: true,

  // ส่งผ่าน HTTPS เท่านั้น (production)
  secure: process.env.NODE_ENV === 'production',

  // SameSite policy:
  // 'strict' = ไม่ส่งเลยถ้า cross-site (ปลอดภัยสุด แต่อาจมีปัญหา UX)
  // 'lax'    = ส่งได้ถ้าเป็น top-level navigation GET (แนะนำ)
  // 'none'   = ส่งทุก request (ต้องใช้กับ secure: true)
  sameSite: 'lax',

  // อายุ cookie
  maxAge: 24 * 60 * 60 * 1000, // 1 day

  // Path ที่ cookie ใช้ได้
  path: '/',

  // Domain ที่ cookie ใช้ได้
  // domain: '.myapp.com',  // ใช้ได้กับ subdomain ทั้งหมด
}

// ชื่อ cookie ไม่ควรบอกว่าใช้ framework อะไร
name: 'sessionId',  // แทนที่จะเป็น 'connect.sid'
```

### Cookie Security Headers

```typescript
// ใช้ helmet เพื่อ set security headers
import helmet from 'helmet';

app.use(helmet({
  // Content Security Policy
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'", "'nonce-{nonce}'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      imgSrc: ["'self'", "data:", "https:"],
    },
  },
  // ป้องกัน Clickjacking
  frameguard: { action: 'deny' },
  // ป้องกัน XSS
  xssFilter: true,
  // ป้องกัน MIME sniffing
  noSniff: true,
}));
```

---

## 7. Multiple Device Sessions

```typescript
// src/routes/session.ts
import { Router, Request, Response, NextFunction } from 'express';
import { sessionService } from '../services/sessionService';
import { requireAuth } from '../middleware/auth';

const router = Router();

// ดู sessions ทั้งหมดที่ login อยู่
router.get('/sessions', requireAuth, async (req: Request, res: Response, next: NextFunction) => {
  try {
    const userId = req.session.userId!;
    const currentSessionId = req.session.id;
    const sessions = await sessionService.getUserSessionDetails(userId);

    // Mark current session
    const sessionsWithCurrent = sessions.map((s) => ({
      ...s,
      isCurrent: s.sessionId === currentSessionId,
    }));

    res.json({
      success: true,
      data: {
        activeSessions: sessionsWithCurrent.length,
        sessions: sessionsWithCurrent,
      },
    });
  } catch (error) {
    next(error);
  }
});

// ลบ session เฉพาะ device
router.delete('/sessions/:sessionId', requireAuth, async (req: Request, res: Response, next: NextFunction) => {
  try {
    const { sessionId } = req.params;
    const userId = req.session.userId!;
    const currentSessionId = req.session.id;

    if (sessionId === currentSessionId) {
      return res.status(400).json({
        success: false,
        error: { code: 'CANNOT_DELETE_CURRENT_SESSION', message: 'ไม่สามารถลบ session ปัจจุบันได้ ใช้ /logout แทน' },
      });
    }

    // ตรวจสอบว่า session นี้เป็นของ user จริงหรือเปล่า
    const userSessions = await sessionService.getUserSessions(userId);

    if (!userSessions.includes(sessionId)) {
      return res.status(404).json({
        success: false,
        error: { code: 'SESSION_NOT_FOUND', message: 'ไม่พบ session นี้' },
      });
    }

    // ลบ session จาก Redis
    await redisClient.del(`sess:${sessionId}`);
    await sessionService.removeFromUserSessions(userId, sessionId);

    res.json({
      success: true,
      message: 'ลบ session สำเร็จ',
    });
  } catch (error) {
    next(error);
  }
});

// Logout จากทุก device
router.delete('/sessions', requireAuth, async (req: Request, res: Response, next: NextFunction) => {
  try {
    const userId = req.session.userId!;

    // ลบ sessions ทั้งหมด
    await sessionService.destroyAllUserSessions(userId);

    // Clear cookie ของ current session
    res.clearCookie('sessionId');

    res.json({
      success: true,
      message: 'Logout จากทุก device สำเร็จ',
    });
  } catch (error) {
    next(error);
  }
});

export default router;
```

---

## 8. Session Invalidation

```typescript
// สถานการณ์ที่ต้อง invalidate sessions:
// 1. เปลี่ยน password
// 2. Account ถูก ban/suspend
// 3. Permission เปลี่ยน
// 4. User request logout all devices

// src/services/userService.ts
import { sessionService } from './sessionService';
import pool from '../db/pool';
import bcrypt from 'bcrypt';

export class UserService {
  // เปลี่ยน password (และ invalidate sessions ทั้งหมด)
  async changePassword(
    userId: string,
    currentPassword: string,
    newPassword: string
  ): Promise<void> {
    // ตรวจสอบ current password
    const user = await pool.query(
      'SELECT password_hash FROM users WHERE id = $1',
      [userId]
    );

    const isValid = await bcrypt.compare(currentPassword, user.rows[0].password_hash);

    if (!isValid) {
      throw new Error('Current password ไม่ถูกต้อง');
    }

    // Hash new password
    const newHash = await bcrypt.hash(newPassword, 12);

    // อัพเดทใน DB
    await pool.query(
      'UPDATE users SET password_hash = $1 WHERE id = $2',
      [newHash, userId]
    );

    // Invalidate sessions ทั้งหมด (ยกเว้น session ปัจจุบัน)
    await sessionService.destroyAllUserSessions(userId);
  }

  // Ban user
  async banUser(userId: string): Promise<void> {
    await pool.query(
      'UPDATE users SET is_active = false WHERE id = $1',
      [userId]
    );

    // Invalidate sessions ทั้งหมดทันที
    await sessionService.destroyAllUserSessions(userId);
  }

  // อัพเดท role (permission changed)
  async updateRole(userId: string, newRole: string): Promise<void> {
    await pool.query(
      `UPDATE users SET role_id = (SELECT id FROM roles WHERE name = $1)
       WHERE id = $2`,
      [newRole, userId]
    );

    // Invalidate sessions เพื่อให้ login ใหม่พร้อม role ใหม่
    await sessionService.destroyAllUserSessions(userId);
  }
}
```

---

## 9. Session Data: User Info และ Preferences

```typescript
// src/middleware/sessionAuth.ts

export const requireAuth = (req: Request, res: Response, next: NextFunction) => {
  if (!req.session.userId) {
    return res.status(401).json({
      success: false,
      error: { code: 'NOT_AUTHENTICATED', message: 'กรุณา login ก่อน' },
    });
  }
  next();
};

// Routes สำหรับ session data
router.get('/me', requireAuth, (req: Request, res: Response) => {
  res.json({
    success: true,
    data: {
      id: req.session.userId,
      email: req.session.email,
      role: req.session.role,
      preferences: req.session.preferences,
      sessionInfo: {
        createdAt: req.session.createdAt,
        lastActivity: req.session.lastActivity,
        deviceId: req.session.deviceId,
      },
    },
  });
});

// อัพเดท preferences
router.put('/preferences', requireAuth, async (req: Request, res: Response, next: NextFunction) => {
  try {
    const { language, theme, notifications } = req.body;

    req.session.preferences = {
      language: language || req.session.preferences?.language || 'th',
      theme: theme || req.session.preferences?.theme || 'light',
      notifications: notifications ?? req.session.preferences?.notifications ?? true,
    };

    // Save ลง Redis ผ่าน connect-redis
    await new Promise<void>((resolve, reject) => {
      req.session.save((err) => (err ? reject(err) : resolve()));
    });

    // บันทึกลง DB ด้วย (persistent)
    await pool.query(
      'UPDATE users SET preferences = $1 WHERE id = $2',
      [JSON.stringify(req.session.preferences), req.session.userId]
    );

    res.json({
      success: true,
      data: req.session.preferences,
    });
  } catch (error) {
    next(error);
  }
});
```

---

## 10. Session Fixation Prevention

```typescript
// Session Fixation Attack:
// ผู้โจมตีบังคับให้ victim ใช้ session ID ที่ตัวเองรู้
// วิธีป้องกัน: Regenerate session ID หลัง login สำเร็จ

router.post('/login', async (req: Request, res: Response, next: NextFunction) => {
  try {
    // ... validate credentials ...

    // สำคัญ! Regenerate session ID หลัง login สำเร็จ
    // วิธีนี้ป้องกัน session fixation attack
    await new Promise<void>((resolve, reject) => {
      req.session.regenerate((err) => {
        if (err) reject(err);
        else resolve();
      });
    });

    // เขียนข้อมูลหลัง regenerate (session ใหม่ว่างเปล่า)
    req.session.userId = user.id;
    req.session.email = user.email;
    req.session.role = user.role;
    req.session.createdAt = new Date().toISOString();
    req.session.lastActivity = new Date().toISOString();

    await sessionService.addToUserSessions(user.id, req.session.id);

    // Save session
    await new Promise<void>((resolve, reject) => {
      req.session.save((err) => (err ? reject(err) : resolve()));
    });

    res.json({
      success: true,
      data: {
        user: {
          id: user.id,
          email: user.email,
          role: user.role,
        },
      },
    });
  } catch (error) {
    next(error);
  }
});
```

---

## 11. CSRF Protection

### ทำไมต้องป้องกัน CSRF?

```
CSRF (Cross-Site Request Forgery):
ผู้โจมตีหลอกให้ browser ของ victim ส่ง request ไปยัง server
เนื่องจาก browser ส่ง cookie อัตโนมัติ session-based auth จึงเสี่ยง

ตัวอย่าง:
victim login ที่ mybank.com
เปิดเว็บ evil.com ที่มี hidden form:
<form action="https://mybank.com/transfer" method="POST">
  <input name="amount" value="10000">
  <input name="to" value="attacker_account">
</form>
<script>document.forms[0].submit()</script>
```

### วิธีป้องกัน: CSRF Token

```typescript
// src/middleware/csrf.ts
import { Request, Response, NextFunction } from 'express';
import crypto from 'crypto';

declare module 'express-session' {
  interface SessionData {
    csrfToken?: string;
  }
}

// สร้าง CSRF token
export function generateCsrfToken(req: Request): string {
  if (!req.session.csrfToken) {
    req.session.csrfToken = crypto.randomBytes(32).toString('hex');
  }
  return req.session.csrfToken;
}

// Middleware ตรวจสอบ CSRF token
export function csrfProtection(req: Request, res: Response, next: NextFunction) {
  // ข้าม CSRF check สำหรับ safe methods
  if (['GET', 'HEAD', 'OPTIONS'].includes(req.method)) {
    return next();
  }

  const sessionToken = req.session.csrfToken;
  const requestToken =
    req.headers['x-csrf-token'] ||
    req.body._csrf ||
    req.query._csrf;

  if (!sessionToken || sessionToken !== requestToken) {
    return res.status(403).json({
      success: false,
      error: {
        code: 'INVALID_CSRF_TOKEN',
        message: 'CSRF token ไม่ถูกต้อง',
      },
    });
  }

  // Rotate CSRF token หลังใช้ (ป้องกัน replay attack)
  req.session.csrfToken = crypto.randomBytes(32).toString('hex');

  next();
}

// Endpoint ให้ client ดึง CSRF token
router.get('/csrf-token', requireAuth, (req: Request, res: Response) => {
  const token = generateCsrfToken(req);
  res.json({ success: true, data: { csrfToken: token } });
});
```

### Double Submit Cookie Pattern (SameSite Alternative)

```typescript
// ถ้าใช้ SameSite=Strict หรือ SameSite=Lax
// ก็ไม่จำเป็นต้องใช้ CSRF token เพิ่มเติม
// เพราะ browser จะไม่ส่ง cookie ใน cross-site request

// แนะนำ: ใช้ทั้งสองอย่าง
cookie: {
  sameSite: 'lax',  // ลด CSRF risk อยู่แล้ว
}
// + CSRF token สำหรับ extra protection
```

---

## 12. TypeScript Implementation

### Type Definitions

```typescript
// src/types/session.ts
import 'express-session';

export interface UserPreferences {
  language: 'th' | 'en';
  theme: 'light' | 'dark' | 'system';
  notifications: boolean;
  emailDigest: 'daily' | 'weekly' | 'never';
}

export interface CartItem {
  productId: string;
  name: string;
  price: number;
  quantity: number;
  imageUrl?: string;
  addedAt: string;
}

declare module 'express-session' {
  interface SessionData {
    // User identity
    userId?: string;
    email?: string;
    role?: string;

    // Device tracking
    deviceId?: string;
    deviceName?: string;

    // Timestamps
    createdAt?: string;
    lastActivity?: string;

    // User preferences
    preferences?: UserPreferences;

    // Shopping cart
    cart?: CartItem[];

    // CSRF token
    csrfToken?: string;

    // Temporary data
    pendingAction?: {
      type: string;
      data: any;
      expiresAt: string;
    };
  }
}
```

### Session Repository

```typescript
// src/repositories/sessionRepository.ts
import { redisClient } from '../config/session';

export class SessionRepository {
  private readonly SESSION_PREFIX = 'sess:';
  private readonly USER_SESSIONS_PREFIX = 'user_sessions:';

  async getSessionData(sessionId: string): Promise<any | null> {
    const raw = await redisClient.get(`${this.SESSION_PREFIX}${sessionId}`);
    if (!raw) return null;
    return JSON.parse(raw);
  }

  async deleteSession(sessionId: string): Promise<void> {
    await redisClient.del(`${this.SESSION_PREFIX}${sessionId}`);
  }

  async getUserActiveSessions(userId: string): Promise<{
    sessionId: string;
    data: any;
  }[]> {
    const sessionIds = await redisClient.sMembers(
      `${this.USER_SESSIONS_PREFIX}${userId}`
    );

    const results = await Promise.all(
      sessionIds.map(async (id) => {
        const data = await this.getSessionData(id);
        if (!data) {
          // Session expired, clean up
          await redisClient.sRem(`${this.USER_SESSIONS_PREFIX}${userId}`, id);
          return null;
        }
        return { sessionId: id, data };
      })
    );

    return results.filter(Boolean) as { sessionId: string; data: any }[];
  }

  async countActiveSessions(userId: string): Promise<number> {
    const sessions = await this.getUserActiveSessions(userId);
    return sessions.length;
  }
}
```

---

## 13. Full Example: Shopping Cart ด้วย Sessions

```typescript
// src/routes/cart.ts
import { Router, Request, Response, NextFunction } from 'express';
import { z } from 'zod';
import pool from '../db/pool';
import { requireAuth } from '../middleware/auth';

const router = Router();

// Schema
const addToCartSchema = z.object({
  productId: z.string().uuid(),
  quantity: z.number().int().min(1).max(99),
});

// GET /cart — ดู cart
router.get('/', (req: Request, res: Response) => {
  const cart = req.session.cart || [];

  const total = cart.reduce((sum, item) => sum + item.price * item.quantity, 0);

  res.json({
    success: true,
    data: {
      items: cart,
      itemCount: cart.reduce((sum, item) => sum + item.quantity, 0),
      total: Math.round(total * 100) / 100,
    },
  });
});

// POST /cart/items — เพิ่มสินค้า
router.post('/items', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const { productId, quantity } = addToCartSchema.parse(req.body);

    // ดึงข้อมูลสินค้าจาก DB
    const product = await pool.query(
      'SELECT id, name, price, stock_quantity FROM products WHERE id = $1 AND is_active = true',
      [productId]
    );

    if (product.rows.length === 0) {
      return res.status(404).json({
        success: false,
        error: { code: 'PRODUCT_NOT_FOUND', message: 'ไม่พบสินค้านี้' },
      });
    }

    const prod = product.rows[0];

    // ตรวจสอบ stock
    if (prod.stock_quantity < quantity) {
      return res.status(400).json({
        success: false,
        error: {
          code: 'INSUFFICIENT_STOCK',
          message: `สินค้าคงเหลือเพียง ${prod.stock_quantity} ชิ้น`,
        },
      });
    }

    // อัพเดท cart ใน session
    const cart = req.session.cart || [];
    const existingIndex = cart.findIndex((item) => item.productId === productId);

    if (existingIndex >= 0) {
      // เพิ่มจำนวน
      const newQuantity = cart[existingIndex].quantity + quantity;

      if (newQuantity > prod.stock_quantity) {
        return res.status(400).json({
          success: false,
          error: {
            code: 'INSUFFICIENT_STOCK',
            message: `ไม่สามารถเพิ่มได้ stock คงเหลือ ${prod.stock_quantity} ชิ้น`,
          },
        });
      }

      cart[existingIndex].quantity = newQuantity;
    } else {
      // เพิ่มสินค้าใหม่
      cart.push({
        productId: prod.id,
        name: prod.name,
        price: parseFloat(prod.price),
        quantity,
        addedAt: new Date().toISOString(),
      });
    }

    req.session.cart = cart;

    res.json({
      success: true,
      data: { cart },
      message: 'เพิ่มสินค้าลงตะกร้าสำเร็จ',
    });
  } catch (error) {
    next(error);
  }
});

// PUT /cart/items/:productId — อัพเดทจำนวน
router.put('/items/:productId', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const { productId } = req.params;
    const { quantity } = z.object({ quantity: z.number().int().min(0).max(99) }).parse(req.body);

    const cart = req.session.cart || [];
    const index = cart.findIndex((item) => item.productId === productId);

    if (index === -1) {
      return res.status(404).json({
        success: false,
        error: { code: 'ITEM_NOT_IN_CART', message: 'ไม่พบสินค้านี้ในตะกร้า' },
      });
    }

    if (quantity === 0) {
      // ลบสินค้าออก
      cart.splice(index, 1);
    } else {
      cart[index].quantity = quantity;
    }

    req.session.cart = cart;

    res.json({
      success: true,
      data: { cart },
    });
  } catch (error) {
    next(error);
  }
});

// DELETE /cart/items/:productId — ลบสินค้า
router.delete('/items/:productId', (req: Request, res: Response) => {
  const { productId } = req.params;
  const cart = req.session.cart || [];

  req.session.cart = cart.filter((item) => item.productId !== productId);

  res.json({
    success: true,
    data: { cart: req.session.cart },
  });
});

// DELETE /cart — ล้างตะกร้า
router.delete('/', (req: Request, res: Response) => {
  req.session.cart = [];

  res.json({
    success: true,
    message: 'ล้างตะกร้าสำเร็จ',
  });
});

// POST /cart/checkout — Checkout
router.post('/checkout', requireAuth, async (req: Request, res: Response, next: NextFunction) => {
  try {
    const cart = req.session.cart;

    if (!cart || cart.length === 0) {
      return res.status(400).json({
        success: false,
        error: { code: 'EMPTY_CART', message: 'ตะกร้าสินค้าว่างเปล่า' },
      });
    }

    const client = await pool.connect();

    try {
      await client.query('BEGIN');

      // ตรวจสอบ stock ทุกรายการ
      for (const item of cart) {
        const stock = await client.query(
          'SELECT stock_quantity FROM products WHERE id = $1 FOR UPDATE',
          [item.productId]
        );

        if (stock.rows[0].stock_quantity < item.quantity) {
          await client.query('ROLLBACK');
          return res.status(400).json({
            success: false,
            error: {
              code: 'INSUFFICIENT_STOCK',
              message: `สินค้า "${item.name}" มีไม่พอ`,
            },
          });
        }
      }

      // สร้าง order
      const total = cart.reduce((sum, item) => sum + item.price * item.quantity, 0);
      const order = await client.query(
        `INSERT INTO orders (user_id, total_amount, status)
         VALUES ($1, $2, 'pending')
         RETURNING id`,
        [req.session.userId, total]
      );

      // สร้าง order items
      for (const item of cart) {
        await client.query(
          `INSERT INTO order_items (order_id, product_id, quantity, unit_price)
           VALUES ($1, $2, $3, $4)`,
          [order.rows[0].id, item.productId, item.quantity, item.price]
        );

        // ลด stock
        await client.query(
          'UPDATE products SET stock_quantity = stock_quantity - $1 WHERE id = $2',
          [item.quantity, item.productId]
        );
      }

      await client.query('COMMIT');

      // ล้างตะกร้า
      req.session.cart = [];

      res.status(201).json({
        success: true,
        data: {
          orderId: order.rows[0].id,
          total,
          itemCount: cart.length,
        },
        message: 'สร้างคำสั่งซื้อสำเร็จ',
      });
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  } catch (error) {
    next(error);
  }
});

export default router;
```

### Setup และ Run

```typescript
// src/server.ts
import app from './app';
import { redisClient } from './config/session';
import pool from './db/pool';

const PORT = process.env.PORT || 3000;

async function start() {
  try {
    // Test DB connection
    await pool.query('SELECT 1');
    console.log('PostgreSQL connected');

    // Test Redis connection
    await redisClient.ping();
    console.log('Redis connected');

    app.listen(PORT, () => {
      console.log(`Server running on http://localhost:${PORT}`);
    });
  } catch (error) {
    console.error('Failed to start server:', error);
    process.exit(1);
  }
}

start();
```

### ทดสอบ API

```bash
# Login
curl -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -c cookies.txt \
  -d '{"emailOrUsername": "john", "password": "Password123!"}'

# ดู CSRF token
curl http://localhost:3000/api/csrf-token \
  -b cookies.txt

# เพิ่มสินค้าลงตะกร้า
curl -X POST http://localhost:3000/api/cart/items \
  -H "Content-Type: application/json" \
  -H "X-CSRF-Token: <csrf_token>" \
  -b cookies.txt -c cookies.txt \
  -d '{"productId": "uuid-here", "quantity": 2}'

# ดูตะกร้า
curl http://localhost:3000/api/cart \
  -b cookies.txt

# Checkout
curl -X POST http://localhost:3000/api/cart/checkout \
  -H "X-CSRF-Token: <csrf_token>" \
  -b cookies.txt -c cookies.txt

# ดู sessions
curl http://localhost:3000/api/sessions \
  -b cookies.txt

# Logout จากทุก device
curl -X DELETE http://localhost:3000/api/sessions \
  -H "X-CSRF-Token: <csrf_token>" \
  -b cookies.txt
```

### .env

```env
NODE_ENV=development
PORT=3000
DATABASE_URL=postgresql://user:password@localhost:5432/myapp
REDIS_URL=redis://localhost:6379
SESSION_SECRET=your-session-secret-min-32-chars-long!!
FRONTEND_URL=http://localhost:5173
```

---

## สรุป

| หัวข้อ | Best Practice |
|--------|--------------|
| Session Secret | Random string อย่างน้อย 32 chars |
| Cookie | httpOnly + secure + sameSite |
| Session ID | Regenerate หลัง login |
| Expiration | Sliding + Absolute limit |
| Multiple Devices | Track user sessions ใน Redis Set |
| CSRF | Token-based + SameSite |
| Revocation | Delete session ทันทีใน Redis |
| Data | เก็บแค่ที่จำเป็น ไม่เก็บ sensitive data |

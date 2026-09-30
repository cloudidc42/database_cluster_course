# Part 47: Middleware Patterns

## Middleware คืออะไร?

Middleware คือฟังก์ชันที่อยู่ "ระหว่าง" การรับ request และการส่ง response ใน Express.js แต่ละ middleware สามารถ:

1. **ดำเนินการ** กับ request และ response objects
2. **จบ** request-response cycle
3. **ส่งต่อ** ให้ middleware ถัดไปใน chain โดยเรียก `next()`
4. **ส่ง error** ไปยัง error handler โดยเรียก `next(error)`

### Middleware Chain

```
Request → [MW1] → [MW2] → [MW3] → Route Handler → Response
              ↓         ↓         ↓
           next()    next()    next()

ถ้า MW2 ไม่เรียก next() → request จะหยุดอยู่ที่ MW2 (ต้องส่ง response เองหรือ chain จะค้าง)
```

### Middleware Signature

```typescript
// Regular middleware
type Middleware = (req: Request, res: Response, next: NextFunction) => void | Promise<void>;

// Error-handling middleware (4 parameters)
type ErrorMiddleware = (err: Error, req: Request, res: Response, next: NextFunction) => void;
```

---

## Middleware Types

### 1. Application-level Middleware

```typescript
import express from 'express';
const app = express();

// ทุก route และทุก HTTP method
app.use((req, res, next) => {
  console.log('Request received:', req.method, req.path);
  next();
});

// เฉพาะ path prefix
app.use('/api', (req, res, next) => {
  console.log('API request');
  next();
});
```

### 2. Router-level Middleware

```typescript
import { Router } from 'express';
const router = Router();

// ใช้กับทุก route ใน router นี้
router.use(authenticate);
router.use(requestLogger);

router.get('/users', getUsers);
router.post('/users', createUser);
```

### 3. Error-handling Middleware

```typescript
// ต้องมี 4 parameters เสมอ แม้จะไม่ใช้ next
app.use((err: Error, req: Request, res: Response, next: NextFunction) => {
  console.error(err);
  res.status(500).json({ error: err.message });
});
```

### 4. Built-in Middleware

```typescript
app.use(express.json());                    // parse JSON body
app.use(express.urlencoded({ extended: true }));  // parse form data
app.use(express.static('public'));          // serve static files
app.use(express.text());                    // parse text body
app.use(express.raw());                     // parse raw body (Buffer)
```

### 5. Third-party Middleware

```typescript
import cors from 'cors';
import helmet from 'helmet';
import morgan from 'morgan';
import compression from 'compression';
import rateLimit from 'express-rate-limit';

app.use(cors());
app.use(helmet());
app.use(morgan('combined'));
app.use(compression());
app.use(rateLimit({ windowMs: 15 * 60 * 1000, max: 100 }));
```

---

## 1. Request ID Middleware

```typescript
// middleware/request-id.ts
import { Request, Response, NextFunction } from 'express';
import { v4 as uuidv4 } from 'uuid';

// Extend Express Request type
declare global {
  namespace Express {
    interface Request {
      requestId: string;
      startTime: number;
    }
  }
}

export function requestId(req: Request, res: Response, next: NextFunction): void {
  // ใช้ request ID จาก header ถ้ามี (สำหรับ distributed tracing)
  const existingId = req.headers['x-request-id'] as string;
  const requestId = existingId || uuidv4();

  req.requestId = requestId;
  req.startTime = Date.now();

  // ใส่ ID ใน response header เพื่อให้ client ติดตามได้
  res.setHeader('X-Request-Id', requestId);

  next();
}

// Usage:
// app.use(requestId);
// แล้วใช้ใน route:
// console.log(req.requestId); // "a1b2c3d4-..."
```

---

## 2. Request Logging Middleware

```typescript
// middleware/logger.ts
import { Request, Response, NextFunction } from 'express';
import winston from 'winston';

const logger = winston.createLogger({
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json(),
  ),
  transports: [
    new winston.transports.Console(),
    new winston.transports.File({ filename: 'logs/access.log' }),
  ],
});

export function requestLogger(req: Request, res: Response, next: NextFunction): void {
  const startTime = Date.now();

  // Log เมื่อ response เสร็จ
  res.on('finish', () => {
    const duration = Date.now() - startTime;
    const logData = {
      requestId: req.requestId,
      method: req.method,
      url: req.originalUrl,
      path: req.path,
      query: req.query,
      statusCode: res.statusCode,
      duration: `${duration}ms`,
      userAgent: req.headers['user-agent'],
      ip: req.ip || req.socket.remoteAddress,
      userId: (req as any).user?.id,
      contentLength: res.getHeader('content-length'),
    };

    if (res.statusCode >= 500) {
      logger.error('Request failed', logData);
    } else if (res.statusCode >= 400) {
      logger.warn('Request error', logData);
    } else {
      logger.info('Request completed', logData);
    }
  });

  // Log เมื่อ connection ถูกปิดก่อนที่ response จะเสร็จ
  res.on('close', () => {
    if (!res.writableEnded) {
      logger.warn('Connection closed before response finished', {
        requestId: req.requestId,
        method: req.method,
        url: req.originalUrl,
      });
    }
  });

  next();
}

// Structured access log format
export function accessLog(req: Request, res: Response, next: NextFunction): void {
  res.on('finish', () => {
    // Combined log format (Apache/Nginx style)
    const log = [
      req.ip,
      '-',
      (req as any).user?.id || '-',
      `[${new Date().toISOString()}]`,
      `"${req.method} ${req.originalUrl} HTTP/${req.httpVersion}"`,
      res.statusCode,
      res.getHeader('content-length') || '-',
      `"${req.headers.referer || '-'}"`,
      `"${req.headers['user-agent'] || '-'}"`,
    ].join(' ');

    console.log(log);
  });
  next();
}
```

---

## 3. Authentication Middleware

```typescript
// middleware/auth.ts
import { Request, Response, NextFunction } from 'express';
import jwt, { JwtPayload } from 'jsonwebtoken';
import { createClient } from 'redis';

interface AuthUser {
  id: number;
  email: string;
  role: string;
  sessionId: string;
}

declare global {
  namespace Express {
    interface Request {
      user?: AuthUser;
    }
  }
}

const redis = createClient({ url: process.env.REDIS_URL });
redis.connect();

const JWT_SECRET = process.env.JWT_SECRET!;

// ตรวจสอบ token และ inject user ลง request
export async function authenticate(
  req: Request,
  res: Response,
  next: NextFunction,
): Promise<void> {
  try {
    // Extract token จาก header
    const authHeader = req.headers.authorization;
    if (!authHeader || !authHeader.startsWith('Bearer ')) {
      res.status(401).json({ error: 'Authorization header required' });
      return;
    }

    const token = authHeader.substring(7);

    // Verify JWT
    let payload: JwtPayload;
    try {
      payload = jwt.verify(token, JWT_SECRET) as JwtPayload;
    } catch (jwtError) {
      if (jwtError instanceof jwt.TokenExpiredError) {
        res.status(401).json({ error: 'Token expired', code: 'TOKEN_EXPIRED' });
      } else {
        res.status(401).json({ error: 'Invalid token', code: 'INVALID_TOKEN' });
      }
      return;
    }

    // Check token revocation (blacklist) ใน Redis
    const isRevoked = await redis.get(`revoked:${payload.jti}`);
    if (isRevoked) {
      res.status(401).json({ error: 'Token has been revoked', code: 'TOKEN_REVOKED' });
      return;
    }

    // Check session validity
    const sessionKey = `session:${payload.sub}:${payload.sessionId}`;
    const session = await redis.get(sessionKey);
    if (!session) {
      res.status(401).json({ error: 'Session expired', code: 'SESSION_EXPIRED' });
      return;
    }

    // Inject user into request
    req.user = {
      id: parseInt(payload.sub!),
      email: payload.email,
      role: payload.role,
      sessionId: payload.sessionId,
    };

    next();
  } catch (error) {
    next(error);
  }
}

// Optional authentication (ไม่ reject ถ้าไม่มี token)
export async function optionalAuthenticate(
  req: Request,
  res: Response,
  next: NextFunction,
): Promise<void> {
  const authHeader = req.headers.authorization;
  if (!authHeader) {
    next();
    return;
  }
  await authenticate(req, res, next);
}

// API Key authentication
export async function apiKeyAuth(
  req: Request,
  res: Response,
  next: NextFunction,
): Promise<void> {
  const apiKey = req.headers['x-api-key'] as string;
  if (!apiKey) {
    res.status(401).json({ error: 'API key required' });
    return;
  }

  // Lookup API key in Redis (หรือ database)
  const keyData = await redis.get(`apikey:${apiKey}`);
  if (!keyData) {
    res.status(401).json({ error: 'Invalid API key' });
    return;
  }

  const parsed = JSON.parse(keyData);
  req.user = { id: parsed.userId, email: parsed.email, role: parsed.role, sessionId: 'apikey' };

  // Track API key usage
  await redis.incr(`apikey:usage:${apiKey}:${new Date().toISOString().slice(0, 7)}`);

  next();
}
```

---

## 4. Authorization Middleware

```typescript
// middleware/authorization.ts
import { Request, Response, NextFunction } from 'express';

type Permission = string;
type Role = 'admin' | 'moderator' | 'user' | 'guest';

// Role hierarchy: ยิ่งสูงยิ่งมีสิทธิ์มาก
const ROLE_HIERARCHY: Record<Role, number> = {
  admin: 100,
  moderator: 50,
  user: 10,
  guest: 0,
};

// Role-based permissions
const ROLE_PERMISSIONS: Record<Role, Permission[]> = {
  admin: ['users:read', 'users:write', 'users:delete', 'posts:read', 'posts:write', 'posts:delete', 'admin:*'],
  moderator: ['users:read', 'posts:read', 'posts:write', 'posts:delete'],
  user: ['users:read:own', 'posts:read', 'posts:write:own', 'posts:delete:own'],
  guest: ['posts:read:public'],
};

// ตรวจสอบว่า user มี role สูงพอ
export function requireRole(minimumRole: Role) {
  return (req: Request, res: Response, next: NextFunction): void => {
    if (!req.user) {
      res.status(401).json({ error: 'Authentication required' });
      return;
    }

    const userLevel = ROLE_HIERARCHY[req.user.role as Role] || 0;
    const requiredLevel = ROLE_HIERARCHY[minimumRole];

    if (userLevel < requiredLevel) {
      res.status(403).json({
        error: 'Insufficient privileges',
        required: minimumRole,
        current: req.user.role,
      });
      return;
    }

    next();
  };
}

// ตรวจสอบ specific permission
export function requirePermission(...permissions: Permission[]) {
  return (req: Request, res: Response, next: NextFunction): void => {
    if (!req.user) {
      res.status(401).json({ error: 'Authentication required' });
      return;
    }

    const userPermissions = ROLE_PERMISSIONS[req.user.role as Role] || [];
    const hasPermission = permissions.every(
      perm =>
        userPermissions.includes(perm) ||
        userPermissions.includes('admin:*') ||
        userPermissions.some(up => up.endsWith(':*') && perm.startsWith(up.slice(0, -1))),
    );

    if (!hasPermission) {
      res.status(403).json({
        error: 'Permission denied',
        required: permissions,
      });
      return;
    }

    next();
  };
}

// ตรวจสอบว่าเป็นเจ้าของ resource
export function requireOwnership(getResourceUserId: (req: Request) => number | Promise<number>) {
  return async (req: Request, res: Response, next: NextFunction): Promise<void> => {
    if (!req.user) {
      res.status(401).json({ error: 'Authentication required' });
      return;
    }

    // Admin bypass
    if (req.user.role === 'admin') {
      next();
      return;
    }

    try {
      const resourceUserId = await getResourceUserId(req);

      if (req.user.id !== resourceUserId) {
        res.status(403).json({ error: 'Access denied: not the owner' });
        return;
      }

      next();
    } catch (error) {
      next(error);
    }
  };
}

// Usage examples:
// router.get('/admin/users', requireRole('admin'), getAllUsers);
// router.delete('/posts/:id', requirePermission('posts:delete'), deletePost);
// router.put('/posts/:id', requireOwnership(async (req) => {
//   const post = await Post.findByPk(req.params.id);
//   return post?.userId;
// }), updatePost);
```

---

## 5. Rate Limiting Middleware

```typescript
// middleware/rate-limit.ts
import { Request, Response, NextFunction } from 'express';
import { createClient } from 'redis';

const redis = createClient({ url: process.env.REDIS_URL });
redis.connect();

interface RateLimitOptions {
  windowMs: number;      // time window in ms
  max: number;           // max requests per window
  keyGenerator?: (req: Request) => string;
  message?: string;
  skipSuccessfulRequests?: boolean;
  skipFailedRequests?: boolean;
  onLimitReached?: (req: Request, res: Response) => void;
}

export function rateLimit(options: RateLimitOptions) {
  const {
    windowMs,
    max,
    keyGenerator = (req) => req.ip || 'unknown',
    message = 'Too many requests, please try again later',
    skipSuccessfulRequests = false,
    skipFailedRequests = false,
    onLimitReached,
  } = options;

  const windowSeconds = Math.ceil(windowMs / 1000);

  return async (req: Request, res: Response, next: NextFunction): Promise<void> => {
    const key = `ratelimit:${keyGenerator(req)}`;

    try {
      // Get current count
      const current = await redis.get(key);
      const count = current ? parseInt(current) : 0;

      // Set headers
      res.setHeader('X-RateLimit-Limit', max);
      res.setHeader('X-RateLimit-Remaining', Math.max(0, max - count - 1));
      res.setHeader('X-RateLimit-Reset', Math.ceil(Date.now() / 1000) + windowSeconds);

      if (count >= max) {
        // Rate limit exceeded
        const ttl = await redis.ttl(key);
        res.setHeader('Retry-After', ttl);

        if (onLimitReached) onLimitReached(req, res);

        res.status(429).json({
          error: message,
          retryAfter: ttl,
        });
        return;
      }

      // Increment counter
      if (count === 0) {
        await redis.setEx(key, windowSeconds, '1');
      } else {
        await redis.incr(key);
      }

      // Skip counting based on options
      res.on('finish', async () => {
        if (skipSuccessfulRequests && res.statusCode < 400) {
          await redis.decr(key);
        }
        if (skipFailedRequests && res.statusCode >= 400) {
          await redis.decr(key);
        }
      });

      next();
    } catch (error) {
      // ถ้า Redis ล้มเหลว → ไม่ block request (fail open)
      console.error('Rate limit error:', error);
      next();
    }
  };
}

// Sliding window rate limiter (more accurate)
export function slidingWindowRateLimit(options: RateLimitOptions) {
  const { windowMs, max, keyGenerator = (req) => req.ip || 'unknown' } = options;

  return async (req: Request, res: Response, next: NextFunction): Promise<void> => {
    const key = `sliding:${keyGenerator(req)}`;
    const now = Date.now();
    const windowStart = now - windowMs;

    try {
      // Remove old entries
      await redis.zRemRangeByScore(key, '-inf', windowStart.toString());

      // Count current requests
      const count = await redis.zCard(key);

      res.setHeader('X-RateLimit-Limit', max);
      res.setHeader('X-RateLimit-Remaining', Math.max(0, max - count - 1));

      if (count >= max) {
        // Get oldest request time to calculate retry-after
        const oldest = await redis.zRange(key, 0, 0, { REV: false });
        const retryAfter = oldest.length > 0
          ? Math.ceil((parseInt(oldest[0]) + windowMs - now) / 1000)
          : Math.ceil(windowMs / 1000);

        res.setHeader('Retry-After', retryAfter);
        res.status(429).json({ error: 'Too many requests', retryAfter });
        return;
      }

      // Add current request
      await redis.zAdd(key, { score: now, value: `${now}-${Math.random()}` });
      await redis.expire(key, Math.ceil(windowMs / 1000));

      next();
    } catch (error) {
      console.error('Sliding window rate limit error:', error);
      next();
    }
  };
}

// Different rate limits for different endpoints
export const apiRateLimit = rateLimit({ windowMs: 60_000, max: 100 });
export const authRateLimit = rateLimit({ windowMs: 15 * 60_000, max: 10 });
export const uploadRateLimit = rateLimit({ windowMs: 60_000, max: 5 });

// User-based rate limit (authenticated users get higher limits)
export const userRateLimit = rateLimit({
  windowMs: 60_000,
  max: 1000,
  keyGenerator: (req) => {
    if (req.user) return `user:${req.user.id}`;
    return `ip:${req.ip}`;
  },
});
```

---

## 6. Input Validation Middleware

```typescript
// middleware/validation.ts
import { Request, Response, NextFunction } from 'express';
import Joi from 'joi';
import { z } from 'zod';

// ============================================================
// Joi-based validation
// ============================================================

export function validateBody(schema: Joi.Schema) {
  return (req: Request, res: Response, next: NextFunction): void => {
    const { error, value } = schema.validate(req.body, {
      abortEarly: false,        // รวบ error ทั้งหมด
      allowUnknown: false,      // ไม่อนุญาต field ที่ไม่ได้กำหนด
      stripUnknown: true,       // ลบ field ที่ไม่ได้กำหนดออก
    });

    if (error) {
      res.status(400).json({
        error: 'Validation failed',
        details: error.details.map(d => ({
          field: d.path.join('.'),
          message: d.message,
          type: d.type,
        })),
      });
      return;
    }

    // Replace request body with validated + sanitized value
    req.body = value;
    next();
  };
}

export function validateQuery(schema: Joi.Schema) {
  return (req: Request, res: Response, next: NextFunction): void => {
    const { error, value } = schema.validate(req.query, {
      abortEarly: false,
      allowUnknown: false,
      stripUnknown: true,
      convert: true,  // auto convert string → number, etc.
    });

    if (error) {
      res.status(400).json({
        error: 'Invalid query parameters',
        details: error.details.map(d => ({
          field: d.path.join('.'),
          message: d.message,
        })),
      });
      return;
    }

    (req as any).query = value;
    next();
  };
}

export function validateParams(schema: Joi.Schema) {
  return (req: Request, res: Response, next: NextFunction): void => {
    const { error, value } = schema.validate(req.params, {
      abortEarly: false,
      convert: true,
    });

    if (error) {
      res.status(400).json({
        error: 'Invalid URL parameters',
        details: error.details.map(d => ({ field: d.path.join('.'), message: d.message })),
      });
      return;
    }

    req.params = value;
    next();
  };
}

// ============================================================
// Zod-based validation (TypeScript-first)
// ============================================================

export function validate<T>(schema: z.ZodSchema<T>) {
  return (req: Request, res: Response, next: NextFunction): void => {
    const result = schema.safeParse(req.body);

    if (!result.success) {
      res.status(400).json({
        error: 'Validation failed',
        details: result.error.errors.map(e => ({
          field: e.path.join('.'),
          message: e.message,
          code: e.code,
        })),
      });
      return;
    }

    req.body = result.data;
    next();
  };
}

// ============================================================
// Schema definitions
// ============================================================

const createUserSchema = Joi.object({
  firstName: Joi.string().min(1).max(50).required().trim(),
  lastName: Joi.string().min(1).max(50).required().trim(),
  email: Joi.string().email().required().lowercase().trim(),
  password: Joi.string()
    .min(8)
    .max(128)
    .pattern(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/)
    .required()
    .messages({
      'string.pattern.base': 'Password must contain at least one uppercase, lowercase, and number',
    }),
  role: Joi.string().valid('admin', 'user').default('user'),
  phone: Joi.string()
    .pattern(/^\+?[1-9]\d{1,14}$/)
    .optional()
    .messages({ 'string.pattern.base': 'Invalid phone number format' }),
});

const paginationSchema = Joi.object({
  page: Joi.number().integer().min(1).default(1),
  limit: Joi.number().integer().min(1).max(100).default(20),
  sort: Joi.string().valid('id', 'name', 'email', 'createdAt').default('createdAt'),
  order: Joi.string().valid('asc', 'desc').default('desc'),
  search: Joi.string().max(100).optional().trim(),
});

// Zod schema (TypeScript-first)
const CreateUserZod = z.object({
  firstName: z.string().min(1).max(50),
  lastName: z.string().min(1).max(50),
  email: z.string().email(),
  password: z
    .string()
    .min(8)
    .max(128)
    .regex(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/, 'Must contain upper, lower, number'),
  role: z.enum(['admin', 'user']).default('user'),
});

type CreateUserDto = z.infer<typeof CreateUserZod>;

// Usage:
// router.post('/users',
//   validateBody(createUserSchema),  // Joi
//   createUser
// );
// router.post('/users', validate(CreateUserZod), createUser);  // Zod
```

---

## 7. Response Compression

```typescript
// middleware/compression.ts
import compression from 'compression';
import { Request, Response } from 'express';

export const compressionMiddleware = compression({
  // Compression level: 0-9 (6 = default, higher = slower but smaller)
  level: 6,

  // Threshold: ไม่ compress ถ้า response เล็กกว่านี้
  threshold: 1024, // 1KB

  // Filter function: ตัดสินใจว่าจะ compress หรือไม่
  filter: (req: Request, res: Response) => {
    // ไม่ compress ถ้า client ไม่รองรับ
    if (req.headers['x-no-compression']) {
      return false;
    }

    // ใช้ default filter (check Accept-Encoding header)
    return compression.filter(req, res);
  },
});

// Usage: app.use(compressionMiddleware);
```

---

## 8. CORS Middleware

```typescript
// middleware/cors.ts
import cors, { CorsOptions } from 'cors';

const allowedOrigins = [
  'https://example.com',
  'https://www.example.com',
  'https://app.example.com',
  process.env.NODE_ENV === 'development' ? 'http://localhost:3000' : '',
  process.env.NODE_ENV === 'development' ? 'http://localhost:5173' : '',
].filter(Boolean);

export const corsOptions: CorsOptions = {
  origin: (origin, callback) => {
    // อนุญาต requests ที่ไม่มี origin (เช่น curl, mobile apps)
    if (!origin) {
      callback(null, true);
      return;
    }

    if (allowedOrigins.includes(origin)) {
      callback(null, true);
    } else {
      callback(new Error(`CORS: Origin ${origin} not allowed`));
    }
  },

  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS'],
  allowedHeaders: [
    'Content-Type',
    'Authorization',
    'X-Request-Id',
    'X-API-Key',
  ],
  exposedHeaders: [
    'X-Request-Id',
    'X-RateLimit-Limit',
    'X-RateLimit-Remaining',
    'X-RateLimit-Reset',
    'Deprecation',
    'Sunset',
  ],
  credentials: true,       // อนุญาต cookies/auth headers
  maxAge: 86400,          // preflight cache: 24 ชั่วโมง
  optionsSuccessStatus: 204,
};

export const corsMiddleware = cors(corsOptions);

// Dynamic CORS for multi-tenant
export function dynamicCors(getTenantOrigins: (tenantId: string) => Promise<string[]>) {
  return cors({
    origin: async (origin, callback) => {
      if (!origin) { callback(null, true); return; }

      const tenantId = extractTenantId(origin);
      if (!tenantId) { callback(new Error('Cannot determine tenant')); return; }

      const allowedOrigins = await getTenantOrigins(tenantId);
      if (allowedOrigins.includes(origin)) {
        callback(null, true);
      } else {
        callback(new Error('Origin not allowed'));
      }
    },
  });
}

function extractTenantId(origin: string): string | null {
  // e.g., https://tenant1.example.com → tenant1
  const match = origin.match(/https?:\/\/([^.]+)\.example\.com/);
  return match ? match[1] : null;
}
```

---

## 9. Security Headers Middleware (Helmet)

```typescript
// middleware/security-headers.ts
import helmet from 'helmet';
import { Request, Response, NextFunction } from 'express';

export const securityHeaders = helmet({
  // Content-Security-Policy
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'", "'unsafe-inline'", 'cdn.jsdelivr.net'],
      styleSrc: ["'self'", "'unsafe-inline'", 'fonts.googleapis.com'],
      fontSrc: ["'self'", 'fonts.gstatic.com'],
      imgSrc: ["'self'", 'data:', 'https:'],
      connectSrc: ["'self'"],
      frameSrc: ["'none'"],
      objectSrc: ["'none'"],
      baseUri: ["'self'"],
      formAction: ["'self'"],
      upgradeInsecureRequests: [],
    },
  },

  // Prevent clickjacking
  frameguard: { action: 'deny' },

  // Enable HSTS
  hsts: {
    maxAge: 31536000,     // 1 year
    includeSubDomains: true,
    preload: true,
  },

  // Prevent MIME type sniffing
  noSniff: true,

  // Referrer policy
  referrerPolicy: { policy: 'strict-origin-when-cross-origin' },

  // Remove X-Powered-By header
  hidePoweredBy: true,
});

// Custom security headers เพิ่มเติม
export function additionalSecurityHeaders(
  req: Request,
  res: Response,
  next: NextFunction,
): void {
  // Permissions Policy
  res.setHeader(
    'Permissions-Policy',
    'camera=(), microphone=(), geolocation=(), payment=()',
  );

  // Cache control for sensitive resources
  if (req.path.startsWith('/api/')) {
    res.setHeader('Cache-Control', 'no-store, no-cache, must-revalidate');
    res.setHeader('Pragma', 'no-cache');
  }

  next();
}
```

---

## 10. Request Timeout Middleware

```typescript
// middleware/timeout.ts
import { Request, Response, NextFunction } from 'express';

export function requestTimeout(timeoutMs: number = 30_000) {
  return (req: Request, res: Response, next: NextFunction): void => {
    let timedOut = false;

    const timeout = setTimeout(() => {
      timedOut = true;
      if (!res.headersSent) {
        res.status(408).json({
          error: 'Request timeout',
          timeout: timeoutMs,
        });
      }
    }, timeoutMs);

    // Clear timeout เมื่อ response ส่งแล้ว
    res.on('finish', () => clearTimeout(timeout));
    res.on('close', () => clearTimeout(timeout));

    // Inject check ว่า timed out แล้วหรือยัง
    const originalNext = next;
    const wrappedNext = (err?: any) => {
      if (timedOut) return;
      originalNext(err);
    };

    next = wrappedNext as NextFunction;
    next();
  };
}

// Usage:
// app.use(requestTimeout(30_000));    // 30 second default
// router.get('/slow-query', requestTimeout(60_000), handler);  // 1 minute for specific route
```

---

## 11. Cache Control Headers

```typescript
// middleware/cache.ts
import { Request, Response, NextFunction } from 'express';

interface CacheOptions {
  maxAge?: number;           // seconds
  private?: boolean;
  noStore?: boolean;
  mustRevalidate?: boolean;
  etag?: boolean;
}

export function cacheControl(options: CacheOptions) {
  return (req: Request, res: Response, next: NextFunction): void => {
    if (options.noStore) {
      res.setHeader('Cache-Control', 'no-store');
      next();
      return;
    }

    const directives: string[] = [];

    if (options.private) {
      directives.push('private');
    } else {
      directives.push('public');
    }

    if (options.maxAge !== undefined) {
      directives.push(`max-age=${options.maxAge}`);
    }

    if (options.mustRevalidate) {
      directives.push('must-revalidate');
    }

    res.setHeader('Cache-Control', directives.join(', '));
    next();
  };
}

// ETag middleware สำหรับ conditional requests
export function etagMiddleware(req: Request, res: Response, next: NextFunction): void {
  // Express มี etag built-in แต่นี่คือ custom implementation
  const originalJson = res.json.bind(res);

  res.json = function(body: any) {
    if (req.method === 'GET') {
      const bodyStr = JSON.stringify(body);
      const etag = `"${Buffer.from(bodyStr).toString('base64').slice(0, 27)}"`;

      res.setHeader('ETag', etag);

      // Check If-None-Match
      if (req.headers['if-none-match'] === etag) {
        res.status(304).end();
        return res;
      }
    }

    return originalJson(body);
  };

  next();
}
```

---

## 12. API Key Authentication

```typescript
// middleware/api-key.ts
import { Request, Response, NextFunction } from 'express';
import { createHash } from 'crypto';
import { createClient } from 'redis';

const redis = createClient({ url: process.env.REDIS_URL });
redis.connect();

interface ApiKeyInfo {
  name: string;
  userId: number;
  permissions: string[];
  rateLimit: number;
  createdAt: string;
  lastUsedAt?: string;
}

// Hash API key ก่อนเก็บ (ไม่เก็บ plain text)
function hashApiKey(apiKey: string): string {
  return createHash('sha256').update(apiKey).digest('hex');
}

export async function apiKeyAuthentication(
  req: Request,
  res: Response,
  next: NextFunction,
): Promise<void> {
  // Support หลาย ways ในการส่ง API key
  const apiKey =
    req.headers['x-api-key'] as string ||
    req.query.api_key as string ||
    extractBearerToken(req.headers.authorization);

  if (!apiKey) {
    res.status(401).json({ error: 'API key required' });
    return;
  }

  // Validate format (prefix:random)
  if (!/^[a-zA-Z0-9]{32,64}$/.test(apiKey)) {
    res.status(401).json({ error: 'Invalid API key format' });
    return;
  }

  try {
    const hashedKey = hashApiKey(apiKey);
    const keyData = await redis.get(`apikey:${hashedKey}`);

    if (!keyData) {
      res.status(401).json({ error: 'Invalid or expired API key' });
      return;
    }

    const keyInfo: ApiKeyInfo = JSON.parse(keyData);

    // Update last used
    keyInfo.lastUsedAt = new Date().toISOString();
    await redis.set(`apikey:${hashedKey}`, JSON.stringify(keyInfo));

    // Track usage
    const month = new Date().toISOString().slice(0, 7);
    await redis.incr(`apikey:usage:${hashedKey}:${month}`);

    req.user = {
      id: keyInfo.userId,
      email: '',
      role: 'user',
      sessionId: `apikey:${hashedKey.slice(0, 8)}`,
    };

    next();
  } catch (error) {
    next(error);
  }
}

function extractBearerToken(authorization?: string): string | null {
  if (!authorization?.startsWith('Bearer ')) return null;
  return authorization.substring(7);
}
```

---

## Middleware Composition Pattern

```typescript
// middleware/compose.ts
import { Request, Response, NextFunction } from 'express';

type Middleware = (req: Request, res: Response, next: NextFunction) => void | Promise<void>;

// Compose หลาย middleware เป็น middleware เดียว
export function compose(...middlewares: Middleware[]): Middleware {
  return async (req: Request, res: Response, next: NextFunction) => {
    let index = 0;

    const dispatch = async (i: number): Promise<void> => {
      if (i >= middlewares.length) {
        next();
        return;
      }

      const middleware = middlewares[i];
      await new Promise<void>((resolve, reject) => {
        try {
          const result = middleware(req, res, (err?: any) => {
            if (err) reject(err);
            else resolve();
          });
          if (result instanceof Promise) {
            result.catch(reject);
          }
        } catch (err) {
          reject(err);
        }
      });

      await dispatch(i + 1);
    };

    try {
      await dispatch(0);
    } catch (err) {
      next(err);
    }
  };
}

// Usage:
// const authMiddleware = compose(authenticate, authorize(['user:read']));
// router.get('/users', authMiddleware, getUsers);
```

---

## Conditional Middleware

```typescript
// middleware/conditional.ts
import { Request, Response, NextFunction } from 'express';

type Middleware = (req: Request, res: Response, next: NextFunction) => void;
type Condition = (req: Request) => boolean | Promise<boolean>;

// Apply middleware เฉพาะเมื่อ condition เป็น true
export function when(condition: Condition, middleware: Middleware): Middleware {
  return async (req: Request, res: Response, next: NextFunction) => {
    const shouldApply = await condition(req);
    if (shouldApply) {
      middleware(req, res, next);
    } else {
      next();
    }
  };
}

// Apply middleware เฉพาะเมื่อ condition เป็น false
export function unless(condition: Condition, middleware: Middleware): Middleware {
  return when(async (req) => !(await condition(req)), middleware);
}

// Examples:
// const authMiddleware = unless(
//   (req) => req.path === '/health' || req.path === '/api/auth/login',
//   authenticate
// );

// Apply rate limiting เฉพาะ production
// app.use(when(() => process.env.NODE_ENV === 'production', rateLimit({ ... })));

// Apply logging เฉพาะ non-test environment
// app.use(unless(() => process.env.NODE_ENV === 'test', requestLogger));
```

---

## Async Middleware

```typescript
// middleware/async-wrapper.ts
import { Request, Response, NextFunction, RequestHandler } from 'express';

type AsyncMiddleware = (req: Request, res: Response, next: NextFunction) => Promise<void>;

// Wrapper สำหรับ async middleware (catch errors automatically)
export function asyncHandler(fn: AsyncMiddleware): RequestHandler {
  return (req: Request, res: Response, next: NextFunction) => {
    Promise.resolve(fn(req, res, next)).catch(next);
  };
}

// Usage:
// router.get('/users', asyncHandler(async (req, res, next) => {
//   const users = await userService.findAll();
//   res.json(users);
// }));

// Better: ใช้ express-async-errors package (auto-wraps all routes)
// import 'express-async-errors';
```

---

## Error Propagation

```typescript
// middleware/error-handler.ts
import { Request, Response, NextFunction } from 'express';

export class AppError extends Error {
  constructor(
    public statusCode: number,
    public message: string,
    public code?: string,
    public details?: any,
  ) {
    super(message);
    this.name = 'AppError';
  }
}

export class ValidationError extends AppError {
  constructor(message: string, public fields: Record<string, string>) {
    super(400, message, 'VALIDATION_ERROR', fields);
    this.name = 'ValidationError';
  }
}

export class NotFoundError extends AppError {
  constructor(resource: string) {
    super(404, `${resource} not found`, 'NOT_FOUND');
    this.name = 'NotFoundError';
  }
}

export class UnauthorizedError extends AppError {
  constructor(message = 'Unauthorized') {
    super(401, message, 'UNAUTHORIZED');
    this.name = 'UnauthorizedError';
  }
}

// Global error handler (ต้องอยู่ท้ายสุด)
export function errorHandler(
  err: Error,
  req: Request,
  res: Response,
  next: NextFunction,
): void {
  // AppError: controlled errors
  if (err instanceof AppError) {
    res.status(err.statusCode).json({
      success: false,
      error: {
        code: err.code || 'ERROR',
        message: err.message,
        details: err.details,
      },
    });
    return;
  }

  // Joi Validation Error
  if (err.name === 'ValidationError') {
    res.status(400).json({
      success: false,
      error: {
        code: 'VALIDATION_ERROR',
        message: err.message,
      },
    });
    return;
  }

  // JWT Errors
  if (err.name === 'JsonWebTokenError') {
    res.status(401).json({
      success: false,
      error: { code: 'INVALID_TOKEN', message: 'Invalid token' },
    });
    return;
  }

  if (err.name === 'TokenExpiredError') {
    res.status(401).json({
      success: false,
      error: { code: 'TOKEN_EXPIRED', message: 'Token expired' },
    });
    return;
  }

  // Database errors
  if ((err as any).code === '23505') {  // PostgreSQL unique violation
    res.status(409).json({
      success: false,
      error: { code: 'CONFLICT', message: 'Resource already exists' },
    });
    return;
  }

  // Unknown errors
  console.error('Unhandled error:', err);
  res.status(500).json({
    success: false,
    error: {
      code: 'INTERNAL_ERROR',
      message: process.env.NODE_ENV === 'production'
        ? 'Internal server error'
        : err.message,
    },
  });
}
```

---

## Full Working Middleware Stack (200+ lines)

```typescript
// app.ts - Complete middleware setup

import express, { Express } from 'express';
import helmet from 'helmet';
import cors from 'cors';
import compression from 'compression';
import { v4 as uuidv4 } from 'uuid';
import rateLimit from 'express-rate-limit';
import morgan from 'morgan';
import { createClient } from 'redis';

const app: Express = express();
const redis = createClient({ url: process.env.REDIS_URL });
redis.connect();

// ============================================================
// 1. Security headers (ก่อนสุด)
// ============================================================
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      imgSrc: ["'self'", 'data:', 'https:'],
    },
  },
  hsts: { maxAge: 31536000, includeSubDomains: true, preload: true },
}));

// ============================================================
// 2. CORS
// ============================================================
app.use(cors({
  origin: process.env.ALLOWED_ORIGINS?.split(',') || '*',
  credentials: true,
  exposedHeaders: ['X-Request-Id', 'X-RateLimit-Limit', 'X-RateLimit-Remaining'],
}));

// ============================================================
// 3. Compression
// ============================================================
app.use(compression({ threshold: 1024 }));

// ============================================================
// 4. Body parsers
// ============================================================
app.use(express.json({ limit: '10mb' }));
app.use(express.urlencoded({ extended: true, limit: '10mb' }));

// ============================================================
// 5. Request ID
// ============================================================
app.use((req, res, next) => {
  const requestId = (req.headers['x-request-id'] as string) || uuidv4();
  (req as any).requestId = requestId;
  (req as any).startTime = Date.now();
  res.setHeader('X-Request-Id', requestId);
  next();
});

// ============================================================
// 6. Access logging
// ============================================================
const logFormat = process.env.NODE_ENV === 'production' ? 'combined' : 'dev';
app.use(morgan(logFormat, {
  skip: (req) => req.path === '/health',
}));

// ============================================================
// 7. Rate limiting
// ============================================================
app.use('/api', rateLimit({
  windowMs: 60_000,    // 1 minute
  max: 500,
  standardHeaders: true,
  legacyHeaders: false,
  handler: (req, res) => {
    res.status(429).json({
      error: 'Too many requests',
      retryAfter: res.getHeader('Retry-After'),
    });
  },
}));

// Strict rate limit for auth endpoints
app.use('/api/auth', rateLimit({
  windowMs: 15 * 60_000,  // 15 minutes
  max: 10,
  message: { error: 'Too many authentication attempts' },
}));

// ============================================================
// 8. Request timeout
// ============================================================
app.use((req, res, next) => {
  const timeout = setTimeout(() => {
    if (!res.headersSent) {
      res.status(408).json({ error: 'Request timeout' });
    }
  }, 30_000);

  res.on('finish', () => clearTimeout(timeout));
  res.on('close', () => clearTimeout(timeout));
  next();
});

// ============================================================
// 9. Response time header
// ============================================================
app.use((req, res, next) => {
  res.on('finish', () => {
    const duration = Date.now() - (req as any).startTime;
    res.setHeader('X-Response-Time', `${duration}ms`);
  });
  next();
});

// ============================================================
// 10. Health check (ก่อน authenticate)
// ============================================================
app.get('/health', async (req, res) => {
  try {
    await redis.ping();
    res.json({ status: 'ok', timestamp: new Date().toISOString() });
  } catch {
    res.status(503).json({ status: 'error', timestamp: new Date().toISOString() });
  }
});

// ============================================================
// 11. API Routes (ใส่ middleware ของแต่ละ route เอง)
// ============================================================

// Import routers
import usersRouter from './api/v2/routes/users.routes';
import postsRouter from './api/v2/routes/posts.routes';
import authRouter from './api/v2/routes/auth.routes';

app.use('/api/v2/auth', authRouter);
app.use('/api/v2/users', usersRouter);
app.use('/api/v2/posts', postsRouter);

// ============================================================
// 12. 404 handler
// ============================================================
app.use((req, res) => {
  res.status(404).json({
    error: {
      code: 'NOT_FOUND',
      message: `Route ${req.method} ${req.path} not found`,
    },
  });
});

// ============================================================
// 13. Global error handler (ต้องอยู่ท้ายสุด)
// ============================================================
app.use((err: Error, req: any, res: any, next: any) => {
  console.error(`[${req.requestId}] Error:`, err.message);

  if (res.headersSent) {
    next(err);
    return;
  }

  const statusCode = (err as any).statusCode || 500;
  res.status(statusCode).json({
    error: {
      code: (err as any).code || 'INTERNAL_ERROR',
      message: process.env.NODE_ENV === 'production' && statusCode === 500
        ? 'Internal server error'
        : err.message,
      requestId: req.requestId,
    },
  });
});

export default app;
```

---

## Middleware Testing

```typescript
// middleware/__tests__/auth.test.ts
import request from 'supertest';
import express from 'express';
import jwt from 'jsonwebtoken';
import { authenticate } from '../auth';

describe('Authentication Middleware', () => {
  const app = express();
  app.use(express.json());
  app.get('/protected', authenticate, (req: any, res) => {
    res.json({ userId: req.user.id });
  });

  const JWT_SECRET = 'test-secret';
  process.env.JWT_SECRET = JWT_SECRET;

  it('should reject requests without authorization header', async () => {
    const res = await request(app).get('/protected');
    expect(res.status).toBe(401);
    expect(res.body.error).toBe('Authorization header required');
  });

  it('should reject invalid tokens', async () => {
    const res = await request(app)
      .get('/protected')
      .set('Authorization', 'Bearer invalid-token');
    expect(res.status).toBe(401);
  });

  it('should accept valid tokens', async () => {
    const token = jwt.sign({ sub: '1', email: 'test@test.com', role: 'user' }, JWT_SECRET);
    const res = await request(app)
      .get('/protected')
      .set('Authorization', `Bearer ${token}`);
    expect(res.status).toBe(200);
    expect(res.body.userId).toBe(1);
  });

  it('should reject expired tokens', async () => {
    const token = jwt.sign({ sub: '1' }, JWT_SECRET, { expiresIn: '0s' });
    await new Promise(r => setTimeout(r, 100));
    const res = await request(app)
      .get('/protected')
      .set('Authorization', `Bearer ${token}`);
    expect(res.status).toBe(401);
    expect(res.body.code).toBe('TOKEN_EXPIRED');
  });
});
```

---

## TypeScript Middleware Types

```typescript
// types/express.d.ts
import { User } from '../models/user.model';

declare global {
  namespace Express {
    interface Request {
      requestId: string;
      startTime: number;
      user?: {
        id: number;
        email: string;
        role: string;
        sessionId: string;
      };
      rateLimit?: {
        limit: number;
        remaining: number;
        resetTime: Date;
      };
    }
  }
}

// Custom middleware type with context
export type AuthenticatedRequest = Request & { user: NonNullable<Request['user']> };

// Type-safe middleware factory
export type MiddlewareFactory<TOptions> = (options: TOptions) => RequestHandler;
```

---

## สรุป: Best Practices

```
1. ลำดับ middleware สำคัญมาก: Security → CORS → Parsing → Auth → Routes → Error
2. Error handler ต้องอยู่ท้ายสุดเสมอ (4 parameters)
3. Async middleware ต้อง try/catch หรือใช้ asyncHandler wrapper
4. ทุก middleware ควร call next() เสมอ ยกเว้นกรณีส่ง response
5. อย่า modify req.params/req.query โดยตรง (ใช้ Object.assign)
6. Rate limiting ควรอยู่ก่อน authentication (ป้องกัน brute force)
7. Logging ควร log ทั้ง request และ response (ใช้ res.on('finish'))
8. ตรวจสอบว่า response ส่งแล้วก่อน send ใหม่ (res.headersSent)
9. ใช้ TypeScript types ให้ครบเพื่อ type safety
10. Test ทุก middleware แยกกัน (unit test) และร่วมกัน (integration test)
```

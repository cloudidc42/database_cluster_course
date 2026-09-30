# Part 14: สร้าง REST API พื้นฐานด้วย Express.js

## บทนำ

ในบทนี้เราจะเรียนรู้การสร้าง REST API ที่มีโครงสร้างดี (Clean Architecture) ด้วย Express.js และ TypeScript พร้อม validation, error handling และ middleware ที่จำเป็น

---

## 1. การติดตั้ง

```bash
# Core dependencies
npm install express cors helmet morgan dotenv express-async-errors

# Validation
npm install zod

# TypeScript
npm install --save-dev typescript ts-node nodemon @types/node @types/express @types/cors @types/morgan

# Utilities
npm install uuid bcryptjs jsonwebtoken
npm install --save-dev @types/uuid @types/bcryptjs @types/jsonwebtoken
```

---

## 2. Project Structure (Clean Architecture)

```
src/
├── app.ts                    # Express app setup
├── server.ts                 # Server entry point
├── config/
│   ├── index.ts              # Config aggregator
│   ├── database.ts           # DB config
│   ├── redis.ts              # Redis config
│   └── storage.ts            # Storage config
├── controllers/
│   ├── post.controller.ts
│   ├── comment.controller.ts
│   └── category.controller.ts
├── services/
│   ├── post.service.ts
│   ├── comment.service.ts
│   └── category.service.ts
├── repositories/
│   ├── post.repository.ts
│   ├── comment.repository.ts
│   └── category.repository.ts
├── models/
│   ├── post.model.ts
│   ├── comment.model.ts
│   └── category.model.ts
├── middlewares/
│   ├── auth.middleware.ts
│   ├── error.middleware.ts
│   ├── validate.middleware.ts
│   ├── not-found.middleware.ts
│   └── request-logger.middleware.ts
├── routes/
│   ├── index.ts              # Route aggregator
│   ├── post.routes.ts
│   ├── comment.routes.ts
│   └── category.routes.ts
├── utils/
│   ├── response.utils.ts
│   ├── pagination.utils.ts
│   ├── errors.ts
│   └── logger.ts
└── types/
    ├── express.d.ts          # Express type extensions
    ├── post.types.ts
    └── common.types.ts
```

---

## 3. TypeScript Configuration

```json
// tsconfig.json
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
    "sourceMap": true,
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

```json
// package.json scripts
{
  "scripts": {
    "dev": "nodemon --exec ts-node src/server.ts",
    "build": "tsc",
    "start": "node dist/server.js",
    "lint": "eslint src/**/*.ts",
    "test": "jest --runInBand"
  }
}
```

---

## 4. Config

```typescript
// src/config/index.ts
import dotenv from 'dotenv';
dotenv.config();

export const config = {
  app: {
    name: process.env.APP_NAME || 'My API',
    env: process.env.NODE_ENV || 'development',
    port: parseInt(process.env.PORT || '3000', 10),
    apiVersion: process.env.API_VERSION || 'v1',
    corsOrigins: (process.env.CORS_ORIGINS || 'http://localhost:3000').split(','),
  },
  db: {
    host: process.env.DB_HOST || 'localhost',
    port: parseInt(process.env.DB_PORT || '5432', 10),
    name: process.env.DB_NAME || 'myapp',
    user: process.env.DB_USER || 'postgres',
    password: process.env.DB_PASSWORD || '',
    poolMax: parseInt(process.env.DB_POOL_MAX || '20', 10),
  },
  redis: {
    host: process.env.REDIS_HOST || 'localhost',
    port: parseInt(process.env.REDIS_PORT || '6379', 10),
    password: process.env.REDIS_PASSWORD,
  },
  jwt: {
    secret: process.env.JWT_SECRET || 'change_this_in_production',
    expiresIn: process.env.JWT_EXPIRES_IN || '7d',
    refreshExpiresIn: process.env.JWT_REFRESH_EXPIRES_IN || '30d',
  },
  storage: {
    endpoint: process.env.MINIO_ENDPOINT || 'localhost',
    port: parseInt(process.env.MINIO_PORT || '9000', 10),
    bucket: process.env.MINIO_BUCKET || 'my-app',
  },
  rateLimit: {
    windowMs: parseInt(process.env.RATE_LIMIT_WINDOW_MS || '900000', 10), // 15 min
    maxRequests: parseInt(process.env.RATE_LIMIT_MAX || '100', 10),
  },
};

export type Config = typeof config;
```

---

## 5. Models

```typescript
// src/models/post.model.ts
export interface Post {
  id: number;
  title: string;
  slug: string;
  content: string;
  excerpt?: string;
  authorId: number;
  categoryId?: number;
  status: 'draft' | 'published' | 'archived';
  tags: string[];
  viewCount: number;
  publishedAt?: Date;
  createdAt: Date;
  updatedAt: Date;
  deletedAt?: Date;
}

export interface PostWithRelations extends Post {
  author: {
    id: number;
    username: string;
  };
  category?: {
    id: number;
    name: string;
    slug: string;
  };
  commentCount: number;
}

// src/models/comment.model.ts
export interface Comment {
  id: number;
  postId: number;
  authorId: number;
  parentId?: number;
  content: string;
  isApproved: boolean;
  createdAt: Date;
  updatedAt: Date;
}

// src/models/category.model.ts
export interface Category {
  id: number;
  name: string;
  slug: string;
  description?: string;
  parentId?: number;
  postCount: number;
  createdAt: Date;
  updatedAt: Date;
}
```

---

## 6. Custom Error Classes

```typescript
// src/utils/errors.ts
export enum ErrorCode {
  // 4xx
  BAD_REQUEST = 'BAD_REQUEST',
  UNAUTHORIZED = 'UNAUTHORIZED',
  FORBIDDEN = 'FORBIDDEN',
  NOT_FOUND = 'NOT_FOUND',
  CONFLICT = 'CONFLICT',
  UNPROCESSABLE_ENTITY = 'UNPROCESSABLE_ENTITY',
  TOO_MANY_REQUESTS = 'TOO_MANY_REQUESTS',
  
  // 5xx
  INTERNAL_SERVER_ERROR = 'INTERNAL_SERVER_ERROR',
  SERVICE_UNAVAILABLE = 'SERVICE_UNAVAILABLE',
  
  // Business errors
  INSUFFICIENT_PERMISSIONS = 'INSUFFICIENT_PERMISSIONS',
  RESOURCE_LOCKED = 'RESOURCE_LOCKED',
  QUOTA_EXCEEDED = 'QUOTA_EXCEEDED',
}

export class AppError extends Error {
  public readonly code: ErrorCode;
  public readonly statusCode: number;
  public readonly isOperational: boolean;
  public readonly details?: unknown;

  constructor(
    message: string,
    code: ErrorCode,
    statusCode: number,
    details?: unknown
  ) {
    super(message);
    this.name = 'AppError';
    this.code = code;
    this.statusCode = statusCode;
    this.isOperational = true;
    this.details = details;
    
    // Capture stack trace
    Error.captureStackTrace(this, this.constructor);
  }

  static badRequest(message: string, details?: unknown): AppError {
    return new AppError(message, ErrorCode.BAD_REQUEST, 400, details);
  }

  static unauthorized(message: string = 'Unauthorized'): AppError {
    return new AppError(message, ErrorCode.UNAUTHORIZED, 401);
  }

  static forbidden(message: string = 'Forbidden'): AppError {
    return new AppError(message, ErrorCode.FORBIDDEN, 403);
  }

  static notFound(resource: string = 'Resource'): AppError {
    return new AppError(`${resource} not found`, ErrorCode.NOT_FOUND, 404);
  }

  static conflict(message: string, details?: unknown): AppError {
    return new AppError(message, ErrorCode.CONFLICT, 409, details);
  }

  static unprocessable(message: string, details?: unknown): AppError {
    return new AppError(message, ErrorCode.UNPROCESSABLE_ENTITY, 422, details);
  }

  static tooManyRequests(message: string = 'Too many requests'): AppError {
    return new AppError(message, ErrorCode.TOO_MANY_REQUESTS, 429);
  }

  static internal(message: string = 'Internal server error'): AppError {
    return new AppError(message, ErrorCode.INTERNAL_SERVER_ERROR, 500);
  }
}

export class ValidationError extends AppError {
  public readonly validationErrors: Array<{ field: string; message: string }>;

  constructor(errors: Array<{ field: string; message: string }>) {
    super('Validation failed', ErrorCode.UNPROCESSABLE_ENTITY, 422, errors);
    this.name = 'ValidationError';
    this.validationErrors = errors;
  }
}
```

---

## 7. Response Utilities

```typescript
// src/utils/response.utils.ts
import { Response } from 'express';

export interface ApiResponse<T = unknown> {
  success: boolean;
  data?: T;
  error?: {
    code: string;
    message: string;
    details?: unknown;
  };
  meta?: {
    timestamp: string;
    requestId?: string;
    [key: string]: unknown;
  };
}

export interface PaginatedResponse<T> extends ApiResponse<T[]> {
  pagination: {
    page: number;
    limit: number;
    total: number;
    totalPages: number;
    hasNext: boolean;
    hasPrev: boolean;
  };
}

export class ResponseUtils {
  static success<T>(
    res: Response,
    data: T,
    statusCode: number = 200,
    meta?: Record<string, unknown>
  ): Response {
    const response: ApiResponse<T> = {
      success: true,
      data,
      meta: {
        timestamp: new Date().toISOString(),
        ...meta,
      },
    };
    return res.status(statusCode).json(response);
  }

  static created<T>(res: Response, data: T): Response {
    return this.success(res, data, 201);
  }

  static noContent(res: Response): Response {
    return res.status(204).send();
  }

  static paginated<T>(
    res: Response,
    data: T[],
    pagination: {
      page: number;
      limit: number;
      total: number;
    }
  ): Response {
    const totalPages = Math.ceil(pagination.total / pagination.limit);
    const response: PaginatedResponse<T> = {
      success: true,
      data,
      pagination: {
        ...pagination,
        totalPages,
        hasNext: pagination.page < totalPages,
        hasPrev: pagination.page > 1,
      },
      meta: {
        timestamp: new Date().toISOString(),
      },
    };
    return res.status(200).json(response);
  }

  static error(
    res: Response,
    code: string,
    message: string,
    statusCode: number = 500,
    details?: unknown
  ): Response {
    const response: ApiResponse = {
      success: false,
      error: { code, message, details },
      meta: { timestamp: new Date().toISOString() },
    };
    return res.status(statusCode).json(response);
  }
}
```

---

## 8. Validation Middleware (Zod)

```typescript
// src/middlewares/validate.middleware.ts
import { Request, Response, NextFunction } from 'express';
import { ZodSchema, ZodError, z } from 'zod';
import { ValidationError } from '../utils/errors';

export function validateBody<T>(schema: ZodSchema<T>) {
  return (req: Request, res: Response, next: NextFunction): void => {
    const result = schema.safeParse(req.body);
    
    if (!result.success) {
      const errors = result.error.errors.map(err => ({
        field: err.path.join('.'),
        message: err.message,
      }));
      throw new ValidationError(errors);
    }
    
    req.body = result.data;
    next();
  };
}

export function validateQuery<T>(schema: ZodSchema<T>) {
  return (req: Request, res: Response, next: NextFunction): void => {
    const result = schema.safeParse(req.query);
    
    if (!result.success) {
      const errors = result.error.errors.map(err => ({
        field: err.path.join('.'),
        message: err.message,
      }));
      throw new ValidationError(errors);
    }
    
    (req as any).validatedQuery = result.data;
    next();
  };
}

export function validateParams<T>(schema: ZodSchema<T>) {
  return (req: Request, res: Response, next: NextFunction): void => {
    const result = schema.safeParse(req.params);
    
    if (!result.success) {
      const errors = result.error.errors.map(err => ({
        field: err.path.join('.'),
        message: err.message,
      }));
      throw new ValidationError(errors);
    }
    
    next();
  };
}

// Common param schemas
export const idParamSchema = z.object({
  id: z.string().regex(/^\d+$/, 'ID must be a number').transform(Number),
});

export const slugParamSchema = z.object({
  slug: z.string().min(1).max(200),
});

// Common query schemas
export const paginationSchema = z.object({
  page: z.string().regex(/^\d+$/).transform(Number).default('1'),
  limit: z.string().regex(/^\d+$/).transform(Number).default('20'),
  sortBy: z.string().optional(),
  sortOrder: z.enum(['ASC', 'DESC']).default('DESC'),
});
```

---

## 9. Error Handling Middleware

```typescript
// src/middlewares/error.middleware.ts
import { Request, Response, NextFunction } from 'express';
import { ZodError } from 'zod';
import { AppError, ValidationError, ErrorCode } from '../utils/errors';

// 404 handler
export function notFoundMiddleware(req: Request, res: Response): void {
  res.status(404).json({
    success: false,
    error: {
      code: ErrorCode.NOT_FOUND,
      message: `Route ${req.method} ${req.path} not found`,
    },
    meta: { timestamp: new Date().toISOString() },
  });
}

// Global error handler
export function errorMiddleware(
  err: Error,
  req: Request,
  res: Response,
  next: NextFunction
): void {
  // Log error
  const isDev = process.env.NODE_ENV !== 'production';
  
  if (isDev) {
    console.error('[Error]', {
      name: err.name,
      message: err.message,
      stack: err.stack,
    });
  } else if (!(err instanceof AppError) || !err.isOperational) {
    console.error('[Critical Error]', err);
  }

  // Operational errors (expected)
  if (err instanceof AppError) {
    res.status(err.statusCode).json({
      success: false,
      error: {
        code: err.code,
        message: err.message,
        details: err.details,
      },
      meta: { timestamp: new Date().toISOString() },
    });
    return;
  }

  // Validation errors
  if (err instanceof ValidationError) {
    res.status(422).json({
      success: false,
      error: {
        code: ErrorCode.UNPROCESSABLE_ENTITY,
        message: 'Validation failed',
        details: err.validationErrors,
      },
      meta: { timestamp: new Date().toISOString() },
    });
    return;
  }

  // Database errors
  const pgError = err as Error & { code?: string };
  if (pgError.code === '23505') {
    res.status(409).json({
      success: false,
      error: {
        code: ErrorCode.CONFLICT,
        message: 'Resource already exists',
      },
      meta: { timestamp: new Date().toISOString() },
    });
    return;
  }

  // Unknown errors
  const statusCode = 500;
  const response = {
    success: false,
    error: {
      code: ErrorCode.INTERNAL_SERVER_ERROR,
      message: isDev ? err.message : 'Internal server error',
      ...(isDev && { stack: err.stack }),
    },
    meta: { timestamp: new Date().toISOString() },
  };

  res.status(statusCode).json(response);
}
```

---

## 10. Auth Middleware

```typescript
// src/middlewares/auth.middleware.ts
import { Request, Response, NextFunction } from 'express';
import jwt from 'jsonwebtoken';
import { config } from '../config';
import { AppError } from '../utils/errors';

export interface JwtPayload {
  userId: number;
  username: string;
  role: string;
  iat: number;
  exp: number;
}

// Extend Request type
declare global {
  namespace Express {
    interface Request {
      user?: JwtPayload;
      sessionId?: string;
    }
  }
}

export function authenticate(req: Request, res: Response, next: NextFunction): void {
  const authHeader = req.headers.authorization;
  
  if (!authHeader?.startsWith('Bearer ')) {
    throw AppError.unauthorized('No token provided');
  }
  
  const token = authHeader.slice(7);
  
  try {
    const payload = jwt.verify(token, config.jwt.secret) as JwtPayload;
    req.user = payload;
    next();
  } catch (error) {
    if (error instanceof jwt.TokenExpiredError) {
      throw AppError.unauthorized('Token expired');
    }
    if (error instanceof jwt.JsonWebTokenError) {
      throw AppError.unauthorized('Invalid token');
    }
    throw AppError.unauthorized();
  }
}

export function authorize(...roles: string[]) {
  return (req: Request, res: Response, next: NextFunction): void => {
    if (!req.user) {
      throw AppError.unauthorized();
    }
    
    if (!roles.includes(req.user.role)) {
      throw AppError.forbidden('Insufficient permissions');
    }
    
    next();
  };
}

export function optionalAuth(req: Request, res: Response, next: NextFunction): void {
  const authHeader = req.headers.authorization;
  
  if (!authHeader?.startsWith('Bearer ')) {
    next();
    return;
  }
  
  const token = authHeader.slice(7);
  
  try {
    const payload = jwt.verify(token, config.jwt.secret) as JwtPayload;
    req.user = payload;
  } catch {
    // Ignore invalid tokens for optional auth
  }
  
  next();
}
```

---

## 11. Routes Setup

```typescript
// src/routes/post.routes.ts
import { Router } from 'express';
import { postController } from '../controllers/post.controller';
import { authenticate, authorize, optionalAuth } from '../middlewares/auth.middleware';
import { validateBody, validateQuery, validateParams, idParamSchema, paginationSchema } from '../middlewares/validate.middleware';
import { createPostSchema, updatePostSchema } from '../schemas/post.schema';

const router = Router();

// Public routes
router.get('/', validateQuery(paginationSchema), postController.list.bind(postController));
router.get('/:id', validateParams(idParamSchema), optionalAuth, postController.getById.bind(postController));
router.get('/slug/:slug', postController.getBySlug.bind(postController));

// Protected routes
router.post('/',
  authenticate,
  validateBody(createPostSchema),
  postController.create.bind(postController)
);

router.put('/:id',
  authenticate,
  validateParams(idParamSchema),
  validateBody(updatePostSchema),
  postController.update.bind(postController)
);

router.patch('/:id',
  authenticate,
  validateParams(idParamSchema),
  postController.partialUpdate.bind(postController)
);

router.delete('/:id',
  authenticate,
  validateParams(idParamSchema),
  postController.delete.bind(postController)
);

// Admin routes
router.patch('/:id/publish',
  authenticate,
  authorize('admin', 'moderator'),
  validateParams(idParamSchema),
  postController.publish.bind(postController)
);

// Nested routes: post comments
router.get('/:id/comments',
  validateParams(idParamSchema),
  postController.getComments.bind(postController)
);

export default router;
```

```typescript
// src/routes/index.ts
import { Router } from 'express';
import postRoutes from './post.routes';
import commentRoutes from './comment.routes';
import categoryRoutes from './category.routes';

const router = Router();

router.use('/posts', postRoutes);
router.use('/comments', commentRoutes);
router.use('/categories', categoryRoutes);

// Health check
router.get('/health', (req, res) => {
  res.json({
    success: true,
    data: {
      status: 'healthy',
      timestamp: new Date().toISOString(),
      uptime: process.uptime(),
    },
  });
});

export default router;
```

---

## 12. Validation Schemas

```typescript
// src/schemas/post.schema.ts
import { z } from 'zod';

export const createPostSchema = z.object({
  title: z.string()
    .min(1, 'Title is required')
    .max(200, 'Title must be less than 200 characters')
    .trim(),
  
  content: z.string()
    .min(10, 'Content must be at least 10 characters')
    .max(50000, 'Content is too long'),
  
  excerpt: z.string()
    .max(500, 'Excerpt must be less than 500 characters')
    .trim()
    .optional(),
  
  categoryId: z.number()
    .int()
    .positive()
    .optional(),
  
  status: z.enum(['draft', 'published'])
    .default('draft'),
  
  tags: z.array(z.string().trim().toLowerCase())
    .max(10, 'Too many tags')
    .default([]),
});

export const updatePostSchema = createPostSchema.partial();

export const postQuerySchema = z.object({
  page: z.string().regex(/^\d+$/).transform(Number).default('1'),
  limit: z.string().regex(/^\d+$/).transform(Number).pipe(z.number().max(100)).default('20'),
  status: z.enum(['draft', 'published', 'archived']).optional(),
  categoryId: z.string().regex(/^\d+$/).transform(Number).optional(),
  authorId: z.string().regex(/^\d+$/).transform(Number).optional(),
  search: z.string().trim().optional(),
  tags: z.string().optional(),
  sortBy: z.enum(['created_at', 'updated_at', 'view_count', 'title']).default('created_at'),
  sortOrder: z.enum(['ASC', 'DESC']).default('DESC'),
});

export type CreatePostDto = z.infer<typeof createPostSchema>;
export type UpdatePostDto = z.infer<typeof updatePostSchema>;
export type PostQueryDto = z.infer<typeof postQuerySchema>;
```

---

## 13. Express App Setup

```typescript
// src/app.ts
import express, { Application } from 'express';
import cors from 'cors';
import helmet from 'helmet';
import morgan from 'morgan';
import 'express-async-errors'; // Handle async errors automatically
import { config } from './config';
import routes from './routes';
import { errorMiddleware, notFoundMiddleware } from './middlewares/error.middleware';

export function createApp(): Application {
  const app = express();

  // ============================================================
  // Security Middlewares
  // ============================================================
  app.use(helmet({
    contentSecurityPolicy: config.app.env === 'production',
    crossOriginEmbedderPolicy: false,
  }));

  app.use(cors({
    origin: config.app.corsOrigins,
    methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS'],
    allowedHeaders: ['Content-Type', 'Authorization', 'X-Request-ID'],
    credentials: true,
    maxAge: 86400,
  }));

  // ============================================================
  // Request Parsing
  // ============================================================
  app.use(express.json({ limit: '10mb' }));
  app.use(express.urlencoded({ extended: true, limit: '10mb' }));

  // ============================================================
  // Logging
  // ============================================================
  const morganFormat = config.app.env === 'production'
    ? 'combined'
    : 'dev';
  
  app.use(morgan(morganFormat));

  // ============================================================
  // Routes
  // ============================================================
  app.use(`/api/${config.app.apiVersion}`, routes);

  // ============================================================
  // Error Handling
  // ============================================================
  app.use(notFoundMiddleware);
  app.use(errorMiddleware);

  return app;
}
```

```typescript
// src/server.ts
import { createApp } from './app';
import { config } from './config';
import { db } from './db/pool';
import { redisManager } from './db/redis';

async function startServer(): Promise<void> {
  const app = createApp();

  // Test database connection
  const dbHealthy = await db.healthCheck();
  if (!dbHealthy) {
    console.error('[Server] Database connection failed');
    process.exit(1);
  }
  console.log('[Server] Database connected');

  // Test Redis connection
  const redisHealthy = await redisManager.healthCheck();
  if (!redisHealthy) {
    console.warn('[Server] Redis not available, continuing without cache');
  } else {
    console.log('[Server] Redis connected');
  }

  const server = app.listen(config.app.port, () => {
    console.log(`[Server] Running at http://localhost:${config.app.port}`);
    console.log(`[Server] Environment: ${config.app.env}`);
    console.log(`[Server] API prefix: /api/${config.app.apiVersion}`);
  });

  // Graceful shutdown
  const shutdown = async (signal: string): Promise<void> => {
    console.log(`\n[Server] Received ${signal}. Shutting down...`);
    
    server.close(async () => {
      try {
        await db.end();
        await redisManager.disconnect();
        console.log('[Server] Shutdown complete');
        process.exit(0);
      } catch (error) {
        console.error('[Server] Error during shutdown:', error);
        process.exit(1);
      }
    });

    // Force shutdown after 30 seconds
    setTimeout(() => {
      console.error('[Server] Forced shutdown after timeout');
      process.exit(1);
    }, 30000);
  };

  process.on('SIGTERM', () => shutdown('SIGTERM'));
  process.on('SIGINT', () => shutdown('SIGINT'));
  
  process.on('uncaughtException', (error) => {
    console.error('[Server] Uncaught Exception:', error);
    shutdown('uncaughtException');
  });

  process.on('unhandledRejection', (reason, promise) => {
    console.error('[Server] Unhandled Rejection at:', promise, 'reason:', reason);
    shutdown('unhandledRejection');
  });
}

startServer().catch(console.error);
```

---

## 14. Repository Pattern - Post Repository

```typescript
// src/repositories/post.repository.ts
import { db } from '../db/pool';
import { Post, PostWithRelations } from '../models/post.model';
import { AppError } from '../utils/errors';

export interface PostFilters {
  status?: string;
  categoryId?: number;
  authorId?: number;
  search?: string;
  tags?: string[];
}

export interface PostListOptions extends PostFilters {
  page: number;
  limit: number;
  sortBy?: string;
  sortOrder?: 'ASC' | 'DESC';
}

export interface PaginatedPosts {
  posts: PostWithRelations[];
  total: number;
  page: number;
  limit: number;
}

export class PostRepository {
  async findById(id: number, includeDeleted = false): Promise<PostWithRelations | null> {
    const whereClause = includeDeleted
      ? 'p.id = $1'
      : 'p.id = $1 AND p.deleted_at IS NULL';

    const result = await db.query<PostWithRelations>(
      `SELECT
        p.*,
        u.username as author_username,
        c.name as category_name,
        c.slug as category_slug,
        COUNT(cm.id) as comment_count
       FROM posts p
       LEFT JOIN users u ON p.author_id = u.id
       LEFT JOIN categories c ON p.category_id = c.id
       LEFT JOIN comments cm ON cm.post_id = p.id AND cm.is_approved = true
       WHERE ${whereClause}
       GROUP BY p.id, u.username, c.name, c.slug`,
      [id]
    );

    return result.rows[0] || null;
  }

  async findBySlug(slug: string): Promise<PostWithRelations | null> {
    const result = await db.query<PostWithRelations>(
      `SELECT
        p.*,
        u.username as author_username,
        c.name as category_name,
        c.slug as category_slug,
        COUNT(cm.id)::int as comment_count
       FROM posts p
       LEFT JOIN users u ON p.author_id = u.id
       LEFT JOIN categories c ON p.category_id = c.id
       LEFT JOIN comments cm ON cm.post_id = p.id AND cm.is_approved = true
       WHERE p.slug = $1 AND p.deleted_at IS NULL
       GROUP BY p.id, u.username, c.name, c.slug`,
      [slug]
    );

    return result.rows[0] || null;
  }

  async list(options: PostListOptions): Promise<PaginatedPosts> {
    const { page, limit, sortBy = 'created_at', sortOrder = 'DESC', ...filters } = options;
    const offset = (page - 1) * limit;
    const params: unknown[] = [];
    const conditions: string[] = ['p.deleted_at IS NULL'];

    if (filters.status) {
      params.push(filters.status);
      conditions.push(`p.status = $${params.length}`);
    }

    if (filters.categoryId) {
      params.push(filters.categoryId);
      conditions.push(`p.category_id = $${params.length}`);
    }

    if (filters.authorId) {
      params.push(filters.authorId);
      conditions.push(`p.author_id = $${params.length}`);
    }

    if (filters.search) {
      params.push(`%${filters.search}%`);
      conditions.push(`(p.title ILIKE $${params.length} OR p.content ILIKE $${params.length})`);
    }

    if (filters.tags && filters.tags.length > 0) {
      params.push(filters.tags);
      conditions.push(`p.tags && $${params.length}::text[]`);
    }

    const whereClause = `WHERE ${conditions.join(' AND ')}`;
    
    // Valid sort columns (whitelist)
    const validSortColumns: Record<string, string> = {
      created_at: 'p.created_at',
      updated_at: 'p.updated_at',
      view_count: 'p.view_count',
      title: 'p.title',
    };
    
    const orderColumn = validSortColumns[sortBy] || 'p.created_at';
    const order = sortOrder === 'ASC' ? 'ASC' : 'DESC';

    // Count query
    const countResult = await db.query<{ count: string }>(
      `SELECT COUNT(DISTINCT p.id) as count FROM posts p ${whereClause}`,
      params
    );
    const total = parseInt(countResult.rows[0].count, 10);

    // Data query
    params.push(limit, offset);
    const dataResult = await db.query<PostWithRelations>(
      `SELECT
        p.id, p.title, p.slug, p.excerpt, p.status, p.tags,
        p.view_count, p.published_at, p.created_at, p.updated_at,
        u.id as author_id, u.username as author_username,
        c.id as category_id, c.name as category_name, c.slug as category_slug,
        COUNT(cm.id)::int as comment_count
       FROM posts p
       LEFT JOIN users u ON p.author_id = u.id
       LEFT JOIN categories c ON p.category_id = c.id
       LEFT JOIN comments cm ON cm.post_id = p.id AND cm.is_approved = true
       ${whereClause}
       GROUP BY p.id, u.id, u.username, c.id, c.name, c.slug
       ORDER BY ${orderColumn} ${order}
       LIMIT $${params.length - 1} OFFSET $${params.length}`,
      params
    );

    return {
      posts: dataResult.rows,
      total,
      page,
      limit,
    };
  }

  async create(data: Partial<Post> & { authorId: number; title: string; content: string }): Promise<Post> {
    const slug = this.generateSlug(data.title);
    
    const result = await db.query<Post>(
      `INSERT INTO posts (title, slug, content, excerpt, author_id, category_id, status, tags, created_at, updated_at)
       VALUES ($1, $2, $3, $4, $5, $6, $7, $8, NOW(), NOW())
       RETURNING *`,
      [
        data.title,
        await this.uniqueSlug(slug),
        data.content,
        data.excerpt || null,
        data.authorId,
        data.categoryId || null,
        data.status || 'draft',
        data.tags || [],
      ]
    );

    return result.rows[0];
  }

  async update(id: number, data: Partial<Post>): Promise<Post | null> {
    const setClauses: string[] = [];
    const params: unknown[] = [id];

    const updateableFields: (keyof Post)[] = [
      'title', 'content', 'excerpt', 'categoryId', 'status', 'tags'
    ];

    for (const field of updateableFields) {
      if (data[field] !== undefined) {
        params.push(data[field]);
        const dbField = this.toSnakeCase(field as string);
        setClauses.push(`${dbField} = $${params.length}`);
      }
    }

    if (setClauses.length === 0) return null;

    const result = await db.query<Post>(
      `UPDATE posts
       SET ${setClauses.join(', ')}, updated_at = NOW()
       WHERE id = $1 AND deleted_at IS NULL
       RETURNING *`,
      params
    );

    return result.rows[0] || null;
  }

  async softDelete(id: number): Promise<boolean> {
    const result = await db.query(
      `UPDATE posts
       SET deleted_at = NOW(), updated_at = NOW()
       WHERE id = $1 AND deleted_at IS NULL
       RETURNING id`,
      [id]
    );
    return (result.rowCount || 0) > 0;
  }

  async incrementViewCount(id: number): Promise<void> {
    await db.query(
      'UPDATE posts SET view_count = view_count + 1 WHERE id = $1',
      [id]
    );
  }

  private generateSlug(title: string): string {
    return title
      .toLowerCase()
      .replace(/[^a-z0-9\s-]/g, '')
      .replace(/\s+/g, '-')
      .replace(/-+/g, '-')
      .trim();
  }

  private async uniqueSlug(slug: string): Promise<string> {
    let candidate = slug;
    let counter = 0;
    
    while (true) {
      const result = await db.query(
        'SELECT id FROM posts WHERE slug = $1',
        [candidate]
      );
      
      if (result.rowCount === 0) break;
      
      counter++;
      candidate = `${slug}-${counter}`;
    }
    
    return candidate;
  }

  private toSnakeCase(str: string): string {
    return str.replace(/[A-Z]/g, letter => `_${letter.toLowerCase()}`);
  }
}

export const postRepository = new PostRepository();
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:
1. การ setup Express.js ด้วย TypeScript และ Clean Architecture
2. Project structure ที่เหมาะสม
3. Custom error classes และ error handling
4. Response utility functions
5. Zod validation middleware
6. Auth middleware ด้วย JWT
7. Routes setup ที่สมบูรณ์
8. Repository pattern
9. Graceful shutdown
10. Full Blog API structure

ในบทต่อไปเราจะเรียนรู้การทำ CRUD Operations ครบ

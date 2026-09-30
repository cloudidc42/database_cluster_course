# Part 19: Error Handling และ Validation

## สารบัญ
1. [Error Types: Operational vs Programmer](#1-error-types-operational-vs-programmer)
2. [Error Class Hierarchy](#2-error-class-hierarchy)
3. [Global Error Handler Middleware](#3-global-error-handler-middleware)
4. [Async Error Wrapper](#4-async-error-wrapper)
5. [HTTP Status Codes Guide](#5-http-status-codes-guide)
6. [Error Response Format](#6-error-response-format)
7. [Validation ด้วย Zod](#7-validation-ด้วย-zod)
8. [Request Validation Middleware](#8-request-validation-middleware)
9. [Database Error Handling](#9-database-error-handling)
10. [Redis Error Handling](#10-redis-error-handling)
11. [MinIO Error Handling](#11-minio-error-handling)
12. [Unhandled Rejections และ Uncaught Exceptions](#12-unhandled-rejections-และ-uncaught-exceptions)
13. [Error Logging](#13-error-logging)
14. [Correlation IDs](#14-correlation-ids)
15. [Full Working Implementation](#15-full-working-implementation)

---

## 1. Error Types: Operational vs Programmer

### Operational Errors

Operational errors คือ errors ที่คาดว่าจะเกิดขึ้นและสามารถจัดการได้ — เป็นส่วนหนึ่งของ normal flow ของโปรแกรม

```
ตัวอย่าง Operational Errors:
- ผู้ใช้ส่ง input ไม่ถูกต้อง (validation error)
- Resource ไม่พบ (404 Not Found)
- Authentication ล้มเหลว (401 Unauthorized)
- Rate limit เกิน (429 Too Many Requests)
- Database connection timeout
- External API ไม่ตอบสนอง
```

### Programmer Errors

Programmer errors คือ bugs ใน code — ไม่ควรเกิดขึ้นถ้า code ถูกต้อง

```
ตัวอย่าง Programmer Errors:
- TypeError: Cannot read property of undefined
- ReferenceError: variable is not defined
- Logic errors
- Unexpected null values
- Array out of bounds
```

### ความแตกต่างในการจัดการ

```typescript
// Operational error: จัดการ แสดง message ที่เหมาะสมให้ user
try {
  const user = await findUser(id);
  if (!user) {
    throw new NotFoundError('User', id); // Operational
  }
} catch (error) {
  if (error instanceof AppError) {
    // ส่ง response ที่เหมาะสม
    return res.status(error.statusCode).json({ error: error.message });
  }
  // Programmer error: log แล้ว return 500
  logger.error('Unexpected error', error);
  return res.status(500).json({ error: 'Internal server error' });
}
```

---

## 2. Error Class Hierarchy

```typescript
// src/errors/AppError.ts

// Base error class
export class AppError extends Error {
  public readonly statusCode: number;
  public readonly code: string;
  public readonly isOperational: boolean;
  public readonly details?: any;

  constructor(
    message: string,
    statusCode: number = 500,
    code: string = 'INTERNAL_ERROR',
    isOperational: boolean = true,
    details?: any
  ) {
    super(message);
    this.name = this.constructor.name;
    this.statusCode = statusCode;
    this.code = code;
    this.isOperational = isOperational;
    this.details = details;

    // Capture stack trace (Node.js specific)
    Error.captureStackTrace(this, this.constructor);
  }
}

// 400 Bad Request
export class ValidationError extends AppError {
  constructor(message: string, details?: any) {
    super(message, 400, 'VALIDATION_ERROR', true, details);
  }
}

// 401 Unauthorized
export class UnauthorizedError extends AppError {
  constructor(message: string = 'กรุณา login ก่อน', code: string = 'UNAUTHORIZED') {
    super(message, 401, code, true);
  }
}

// 403 Forbidden
export class ForbiddenError extends AppError {
  constructor(message: string = 'คุณไม่มีสิทธิ์เข้าถึง resource นี้') {
    super(message, 403, 'FORBIDDEN', true);
  }
}

// 404 Not Found
export class NotFoundError extends AppError {
  constructor(resource: string, id?: string | number) {
    const message = id
      ? `ไม่พบ ${resource} ที่มี ID: ${id}`
      : `ไม่พบ ${resource}`;
    super(message, 404, 'NOT_FOUND', true, { resource, id });
  }
}

// 409 Conflict
export class ConflictError extends AppError {
  constructor(message: string, details?: any) {
    super(message, 409, 'CONFLICT', true, details);
  }
}

// 422 Unprocessable Entity
export class UnprocessableError extends AppError {
  constructor(message: string, details?: any) {
    super(message, 422, 'UNPROCESSABLE_ENTITY', true, details);
  }
}

// 429 Too Many Requests
export class RateLimitError extends AppError {
  constructor(retryAfter?: number) {
    super('Request มากเกินไป กรุณารอสักครู่', 429, 'RATE_LIMIT_EXCEEDED', true, { retryAfter });
  }
}

// 500 Internal Server Error
export class InternalError extends AppError {
  constructor(message: string = 'เกิดข้อผิดพลาดภายใน', details?: any) {
    super(message, 500, 'INTERNAL_ERROR', false, details);
  }
}

// 503 Service Unavailable
export class ServiceUnavailableError extends AppError {
  constructor(service: string) {
    super(`${service} ไม่พร้อมใช้งาน`, 503, 'SERVICE_UNAVAILABLE', true, { service });
  }
}

// Database Error
export class DatabaseError extends AppError {
  constructor(message: string, details?: any) {
    super(message, 500, 'DATABASE_ERROR', false, details);
  }
}

// ตรวจสอบว่าเป็น AppError หรือเปล่า
export function isAppError(error: unknown): error is AppError {
  return error instanceof AppError;
}

// ตรวจสอบว่าเป็น Operational Error หรือเปล่า
export function isOperationalError(error: unknown): boolean {
  if (isAppError(error)) return error.isOperational;
  return false;
}
```

---

## 3. Global Error Handler Middleware

```typescript
// src/middleware/errorHandler.ts
import { Request, Response, NextFunction } from 'express';
import { AppError, isAppError } from '../errors/AppError';
import { logger } from '../utils/logger';
import { ZodError } from 'zod';

export function errorHandler(
  err: Error | AppError,
  req: Request,
  res: Response,
  next: NextFunction // ต้องมี 4 parameters แม้ไม่ใช้
): void {
  // รวบรวม context สำหรับ logging
  const requestContext = {
    requestId: req.id,
    method: req.method,
    path: req.path,
    userId: req.user?.id,
    ip: req.ip,
  };

  // 1. Zod Validation Error
  if (err instanceof ZodError) {
    const details = err.errors.map((e) => ({
      field: e.path.join('.'),
      message: e.message,
      code: e.code,
    }));

    logger.warn('Validation error', { ...requestContext, details });

    res.status(400).json({
      success: false,
      error: {
        code: 'VALIDATION_ERROR',
        message: 'ข้อมูลที่ส่งมาไม่ถูกต้อง',
        details,
      },
    });
    return;
  }

  // 2. AppError (Operational)
  if (isAppError(err)) {
    const logLevel = err.statusCode >= 500 ? 'error' : 'warn';

    logger[logLevel]('Application error', {
      ...requestContext,
      code: err.code,
      statusCode: err.statusCode,
      message: err.message,
      stack: err.stack,
    });

    res.status(err.statusCode).json({
      success: false,
      error: {
        code: err.code,
        message: err.message,
        ...(err.details && { details: err.details }),
        ...(process.env.NODE_ENV === 'development' && { stack: err.stack }),
      },
    });
    return;
  }

  // 3. Unknown / Programmer Error
  logger.error('Unexpected error', {
    ...requestContext,
    error: err.message,
    stack: err.stack,
  });

  // ไม่ expose details ใน production
  const message =
    process.env.NODE_ENV === 'production'
      ? 'เกิดข้อผิดพลาดภายใน กรุณาลองใหม่อีกครั้ง'
      : err.message;

  res.status(500).json({
    success: false,
    error: {
      code: 'INTERNAL_ERROR',
      message,
      ...(process.env.NODE_ENV === 'development' && { stack: err.stack }),
    },
  });
}

// Not Found Handler (ใส่ก่อน errorHandler)
export function notFoundHandler(req: Request, res: Response): void {
  res.status(404).json({
    success: false,
    error: {
      code: 'ROUTE_NOT_FOUND',
      message: `ไม่พบ endpoint: ${req.method} ${req.path}`,
    },
  });
}
```

---

## 4. Async Error Wrapper

### ปัญหา: Express ไม่รับ async errors โดยตรง

```typescript
// ❌ ถ้า async function throw Express จะ hang
router.get('/users', async (req, res) => {
  const users = await getUsers(); // ถ้า throw → ไม่มีใครรับ!
  res.json(users);
});

// ✅ ต้องใช้ try/catch ทุกที่ (verbose)
router.get('/users', async (req, res, next) => {
  try {
    const users = await getUsers();
    res.json(users);
  } catch (error) {
    next(error); // ส่งไปยัง error handler
  }
});
```

### Async Wrapper

```typescript
// src/utils/asyncHandler.ts
import { Request, Response, NextFunction, RequestHandler } from 'express';

type AsyncRequestHandler = (
  req: Request,
  res: Response,
  next: NextFunction
) => Promise<any>;

// Wrap async handler เพื่อ catch errors อัตโนมัติ
export function asyncHandler(fn: AsyncRequestHandler): RequestHandler {
  return (req: Request, res: Response, next: NextFunction) => {
    Promise.resolve(fn(req, res, next)).catch(next);
  };
}

// ใช้งาน:
router.get('/users', asyncHandler(async (req, res) => {
  const users = await getUsers();
  res.json(users);
}));

// Error จะถูกส่งไปยัง next() อัตโนมัติ

// Express 5 (ยังไม่ release): รองรับ async handlers โดยตรง
// แต่ Express 4 ต้องใช้ wrapper
```

### Route Wrapper สำหรับ Controller

```typescript
// src/utils/controllerWrapper.ts

type Controller = (req: Request, res: Response) => Promise<void>;

export function wrapController(controller: Controller): RequestHandler {
  return asyncHandler(controller);
}

// แยก handler ออกมาเป็น class
export class UserController {
  getUser = asyncHandler(async (req: Request, res: Response) => {
    const { id } = req.params;
    const user = await userService.findById(id);

    if (!user) {
      throw new NotFoundError('User', id);
    }

    res.json({ success: true, data: user });
  });

  createUser = asyncHandler(async (req: Request, res: Response) => {
    const data = createUserSchema.parse(req.body);
    const user = await userService.create(data);
    res.status(201).json({ success: true, data: user });
  });
}

const userController = new UserController();

router.get('/users/:id', authenticate, userController.getUser);
router.post('/users', authenticate, userController.createUser);
```

---

## 5. HTTP Status Codes Guide

```typescript
// src/utils/httpStatus.ts

export const HttpStatus = {
  // 2xx Success
  OK: 200,              // GET, PUT, PATCH สำเร็จ
  CREATED: 201,         // POST สร้าง resource สำเร็จ
  ACCEPTED: 202,        // Request รับแล้ว แต่ยังประมวลผลอยู่
  NO_CONTENT: 204,      // DELETE สำเร็จ ไม่มี body

  // 3xx Redirection
  MOVED_PERMANENTLY: 301,
  FOUND: 302,
  NOT_MODIFIED: 304,

  // 4xx Client Errors
  BAD_REQUEST: 400,         // Validation error, malformed request
  UNAUTHORIZED: 401,        // ยังไม่ได้ authenticate
  PAYMENT_REQUIRED: 402,    // สำหรับ payment required
  FORBIDDEN: 403,           // Authenticated แต่ไม่มีสิทธิ์
  NOT_FOUND: 404,           // Resource ไม่มี
  METHOD_NOT_ALLOWED: 405,  // HTTP method ไม่ถูกต้อง
  CONFLICT: 409,            // Resource ซ้ำกัน
  GONE: 410,                // Resource ถูกลบถาวรแล้ว
  UNPROCESSABLE: 422,       // Input ถูกต้องแต่ process ไม่ได้
  TOO_MANY_REQUESTS: 429,   // Rate limit

  // 5xx Server Errors
  INTERNAL_ERROR: 500,      // Programmer error
  NOT_IMPLEMENTED: 501,     // Feature ยังไม่ implement
  BAD_GATEWAY: 502,         // Upstream server ตอบกลับผิดพลาด
  SERVICE_UNAVAILABLE: 503, // Server ไม่พร้อม (maintenance, overload)
  GATEWAY_TIMEOUT: 504,     // Upstream timeout
} as const;

// Guide สำหรับการเลือก status code:
//
// ✅ 200 OK:          GET /users/123 → return user data
// ✅ 201 Created:     POST /users → user created
// ✅ 204 No Content:  DELETE /users/123 → user deleted
//
// ✅ 400 Bad Request: POST /users { email: "invalid" }
// ✅ 401 Unauthorized: GET /profile (ไม่ได้ส่ง token)
// ✅ 403 Forbidden:   DELETE /users/456 (ไม่ใช่ admin)
// ✅ 404 Not Found:   GET /users/999 (ไม่มี user นี้)
// ✅ 409 Conflict:    POST /users { email: "existing@email.com" }
// ✅ 422 Unprocessable: transfer amount > balance
// ✅ 429 Too Many:    5th login attempt ใน 15 นาที
//
// ✅ 500 Internal:    Unexpected exception
// ✅ 503 Unavailable: Database ไม่ตอบสนอง
```

---

## 6. Error Response Format

```typescript
// Standard error response format

interface ErrorResponse {
  success: false;
  error: {
    code: string;        // Machine-readable code
    message: string;     // Human-readable message
    details?: any;       // Validation details หรือ extra info
    requestId?: string;  // For tracing
    stack?: string;      // Development only
  };
}

interface SuccessResponse<T> {
  success: true;
  data: T;
  meta?: {
    page?: number;
    perPage?: number;
    total?: number;
    totalPages?: number;
  };
  message?: string;
}

// ตัวอย่าง Error Responses:

// Validation Error
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "ข้อมูลที่ส่งมาไม่ถูกต้อง",
    "details": [
      { "field": "email", "message": "Email ไม่ถูกต้อง" },
      { "field": "password", "message": "Password ต้องมีอย่างน้อย 8 ตัวอักษร" }
    ],
    "requestId": "req_abc123"
  }
}

// Not Found Error
{
  "success": false,
  "error": {
    "code": "NOT_FOUND",
    "message": "ไม่พบ User ที่มี ID: 123",
    "requestId": "req_def456"
  }
}

// Internal Error (Production)
{
  "success": false,
  "error": {
    "code": "INTERNAL_ERROR",
    "message": "เกิดข้อผิดพลาดภายใน กรุณาลองใหม่อีกครั้ง",
    "requestId": "req_ghi789"
  }
}

// Success Response
{
  "success": true,
  "data": { "id": "123", "email": "john@example.com" },
  "message": "สร้างบัญชีสำเร็จ"
}

// Paginated Response
{
  "success": true,
  "data": [{ "id": "1" }, { "id": "2" }],
  "meta": {
    "page": 1,
    "perPage": 20,
    "total": 150,
    "totalPages": 8
  }
}
```

---

## 7. Validation ด้วย Zod

### ทำไม Zod?

```
Zod vs alternatives:
- Joi: ไม่มี TypeScript type inference
- Yup: TypeScript support แต่ API ซับซ้อน
- Zod: TypeScript-first, type inference ดีเยี่ยม, lightweight
```

### Basic Schemas

```typescript
// src/schemas/user.ts
import { z } from 'zod';

// Primitive types
const emailSchema = z.string().email('Email ไม่ถูกต้อง').toLowerCase();
const passwordSchema = z
  .string()
  .min(8, 'Password ต้องมีอย่างน้อย 8 ตัวอักษร')
  .max(100)
  .regex(/[A-Z]/, 'ต้องมีตัวพิมพ์ใหญ่')
  .regex(/[a-z]/, 'ต้องมีตัวพิมพ์เล็ก')
  .regex(/[0-9]/, 'ต้องมีตัวเลข')
  .regex(/[^A-Za-z0-9]/, 'ต้องมีอักขระพิเศษ');

const uuidSchema = z.string().uuid('ID ไม่ถูกต้อง');
const phoneSchema = z
  .string()
  .regex(/^(\+66|0)[0-9]{9,10}$/, 'เบอร์โทรศัพท์ไม่ถูกต้อง')
  .optional();

// Object schemas
export const createUserSchema = z.object({
  email: emailSchema,
  password: passwordSchema,
  username: z
    .string()
    .min(3, 'Username ต้องมีอย่างน้อย 3 ตัวอักษร')
    .max(50)
    .regex(/^[a-zA-Z0-9_]+$/, 'Username ใช้ได้เฉพาะ a-z, A-Z, 0-9, _'),
  profile: z
    .object({
      firstName: z.string().min(1).max(100),
      lastName: z.string().min(1).max(100),
      phone: phoneSchema,
      birthdate: z.string().date('วันเกิดไม่ถูกต้อง').optional(),
      bio: z.string().max(500).optional(),
    })
    .optional(),
});

// Update schema (ทุก field optional)
export const updateUserSchema = createUserSchema.partial().omit({ password: true });

// Type inference
export type CreateUserInput = z.infer<typeof createUserSchema>;
export type UpdateUserInput = z.infer<typeof updateUserSchema>;
```

### Custom Validators

```typescript
// Custom refinement
export const transferSchema = z
  .object({
    fromAccountId: uuidSchema,
    toAccountId: uuidSchema,
    amount: z.number().positive('จำนวนเงินต้องมากกว่า 0').max(1000000),
    currency: z.enum(['THB', 'USD', 'EUR']),
    note: z.string().max(200).optional(),
  })
  .refine((data) => data.fromAccountId !== data.toAccountId, {
    message: 'ไม่สามารถโอนเงินไปยังบัญชีตัวเองได้',
    path: ['toAccountId'],
  });

// Custom transform
export const dateRangeSchema = z
  .object({
    startDate: z.string().datetime(),
    endDate: z.string().datetime(),
  })
  .transform((data) => ({
    startDate: new Date(data.startDate),
    endDate: new Date(data.endDate),
  }))
  .refine((data) => data.startDate < data.endDate, {
    message: 'startDate ต้องน้อยกว่า endDate',
  });

// Superrefine (หลาย errors)
export const signupSchema = z
  .object({
    password: z.string().min(8),
    confirmPassword: z.string(),
    agreedToTerms: z.boolean(),
  })
  .superRefine((data, ctx) => {
    if (data.password !== data.confirmPassword) {
      ctx.addIssue({
        code: z.ZodIssueCode.custom,
        message: 'Password ไม่ตรงกัน',
        path: ['confirmPassword'],
      });
    }

    if (!data.agreedToTerms) {
      ctx.addIssue({
        code: z.ZodIssueCode.custom,
        message: 'กรุณายอมรับเงื่อนไขการใช้งาน',
        path: ['agreedToTerms'],
      });
    }
  });
```

### Nested Schemas และ Arrays

```typescript
// Nested object
export const addressSchema = z.object({
  street: z.string().min(1).max(200),
  city: z.string().min(1).max(100),
  province: z.string().min(1).max(100),
  postalCode: z.string().regex(/^[0-9]{5}$/, 'รหัสไปรษณีย์ต้องมี 5 หลัก'),
  country: z.string().default('TH'),
});

export const orderSchema = z.object({
  items: z
    .array(
      z.object({
        productId: uuidSchema,
        quantity: z.number().int().min(1).max(99),
        note: z.string().max(200).optional(),
      })
    )
    .min(1, 'ต้องมีสินค้าอย่างน้อย 1 รายการ')
    .max(50, 'สั่งได้สูงสุด 50 รายการ'),
  shippingAddress: addressSchema,
  paymentMethod: z.enum(['credit_card', 'promptpay', 'cod']),
  couponCode: z.string().optional(),
});

// Union types
export const notificationPreferenceSchema = z.discriminatedUnion('type', [
  z.object({
    type: z.literal('email'),
    email: z.string().email(),
    frequency: z.enum(['immediate', 'daily', 'weekly']),
  }),
  z.object({
    type: z.literal('sms'),
    phone: phoneSchema,
  }),
  z.object({
    type: z.literal('push'),
    deviceToken: z.string().min(1),
  }),
]);

// Query params validation
export const paginationSchema = z.object({
  page: z
    .string()
    .optional()
    .transform((v) => (v ? parseInt(v, 10) : 1))
    .pipe(z.number().int().min(1)),
  perPage: z
    .string()
    .optional()
    .transform((v) => (v ? parseInt(v, 10) : 20))
    .pipe(z.number().int().min(1).max(100)),
  sortBy: z.enum(['created_at', 'updated_at', 'name']).optional(),
  order: z.enum(['asc', 'desc']).optional().default('desc'),
  search: z.string().max(100).optional(),
});
```

---

## 8. Request Validation Middleware

```typescript
// src/middleware/validate.ts
import { Request, Response, NextFunction } from 'express';
import { AnyZodObject, ZodError, z } from 'zod';

interface ValidationSchemas {
  body?: AnyZodObject;
  query?: AnyZodObject;
  params?: AnyZodObject;
  headers?: AnyZodObject;
}

export function validate(schemas: ValidationSchemas) {
  return async (req: Request, res: Response, next: NextFunction) => {
    try {
      if (schemas.body) {
        req.body = await schemas.body.parseAsync(req.body);
      }

      if (schemas.query) {
        req.query = await schemas.query.parseAsync(req.query);
      }

      if (schemas.params) {
        req.params = await schemas.params.parseAsync(req.params);
      }

      if (schemas.headers) {
        await schemas.headers.parseAsync(req.headers);
      }

      next();
    } catch (error) {
      if (error instanceof ZodError) {
        const details = error.errors.map((e) => ({
          field: e.path.join('.'),
          message: e.message,
          received: (e as any).received,
        }));

        return res.status(400).json({
          success: false,
          error: {
            code: 'VALIDATION_ERROR',
            message: 'ข้อมูลที่ส่งมาไม่ถูกต้อง',
            details,
          },
        });
      }
      next(error);
    }
  };
}

// ใช้งาน:
router.post(
  '/users',
  validate({ body: createUserSchema }),
  asyncHandler(async (req, res) => {
    // req.body มี type CreateUserInput แล้ว
    const user = await userService.create(req.body);
    res.status(201).json({ success: true, data: user });
  })
);

router.get(
  '/users',
  validate({ query: paginationSchema }),
  asyncHandler(async (req, res) => {
    const { page, perPage, sortBy, order } = req.query as z.infer<typeof paginationSchema>;
    const users = await userService.findAll({ page, perPage, sortBy, order });
    res.json({ success: true, data: users });
  })
);
```

---

## 9. Database Error Handling

```typescript
// src/utils/dbErrors.ts
import { DatabaseError as PgDatabaseError } from 'pg';
import { AppError, ConflictError, DatabaseError } from '../errors/AppError';

// PostgreSQL Error Codes
// https://www.postgresql.org/docs/current/errcodes-appendix.html
const PG_ERROR_CODES: Record<string, () => AppError> = {
  '23505': () => new ConflictError('ข้อมูลซ้ำกันในฐานข้อมูล'),     // unique_violation
  '23503': () => new AppError('ข้อมูลที่อ้างอิงไม่มีอยู่', 400, 'FOREIGN_KEY_VIOLATION'),
  '23502': () => new AppError('ข้อมูลจำเป็นไม่ครบ', 400, 'NOT_NULL_VIOLATION'),
  '23514': () => new AppError('ข้อมูลไม่ผ่านการตรวจสอบ', 400, 'CHECK_VIOLATION'),
  '42P01': () => new DatabaseError('ตาราง DB ไม่มีอยู่'),           // undefined_table
  '42703': () => new DatabaseError('Column ไม่มีอยู่'),              // undefined_column
  '08006': () => new AppError('Database ขาดการเชื่อมต่อ', 503, 'DB_CONNECTION_FAILED'),
  '08001': () => new AppError('ไม่สามารถเชื่อมต่อ Database ได้', 503, 'DB_CONNECT_ERROR'),
  '57014': () => new AppError('Query ใช้เวลานานเกินไป', 504, 'QUERY_TIMEOUT'),
  '40001': () => new AppError('Transaction conflict กรุณาลองใหม่', 409, 'SERIALIZATION_FAILURE'),
  '40P01': () => new AppError('Deadlock detected กรุณาลองใหม่', 409, 'DEADLOCK_DETECTED'),
  '53300': () => new AppError('Database connection pool เต็ม', 503, 'TOO_MANY_CONNECTIONS'),
};

export function handleDatabaseError(error: unknown): AppError {
  if (error instanceof PgDatabaseError) {
    const errorCode = error.code || '';
    const errorFactory = PG_ERROR_CODES[errorCode];

    if (errorFactory) {
      return errorFactory();
    }

    // รู้ว่าเป็น DB error แต่ไม่รู้ code
    return new DatabaseError('เกิดข้อผิดพลาดในฐานข้อมูล');
  }

  return new DatabaseError('Unexpected database error');
}

// ตัวอย่างการใช้
async function createUser(data: CreateUserInput) {
  try {
    return await pool.query(
      'INSERT INTO users (email, username) VALUES ($1, $2) RETURNING *',
      [data.email, data.username]
    );
  } catch (error) {
    const dbError = handleDatabaseError(error);

    // เพิ่ม context
    if (dbError.code === 'CONFLICT') {
      throw new ConflictError('Email หรือ Username นี้มีอยู่แล้ว');
    }

    throw dbError;
  }
}

// Transaction with retry (สำหรับ serialization failures)
async function withRetry<T>(
  fn: () => Promise<T>,
  maxRetries: number = 3
): Promise<T> {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      if (isAppError(error) && error.code === 'SERIALIZATION_FAILURE' && attempt < maxRetries) {
        // รอก่อน retry
        await new Promise((resolve) => setTimeout(resolve, attempt * 100));
        continue;
      }
      throw error;
    }
  }
  throw new DatabaseError('Transaction ล้มเหลวหลังจากพยายามหลายครั้ง');
}
```

---

## 10. Redis Error Handling

```typescript
// src/utils/redisErrors.ts
import Redis from 'ioredis';
import { ServiceUnavailableError } from '../errors/AppError';

// Wrapper ที่ handle Redis errors
export class SafeRedisClient {
  constructor(private redis: Redis) {}

  async get(key: string): Promise<string | null> {
    try {
      return await this.redis.get(key);
    } catch (error) {
      this.handleError(error, 'get', key);
      return null; // Graceful degradation
    }
  }

  async set(
    key: string,
    value: string,
    ...args: any[]
  ): Promise<'OK' | null> {
    try {
      return await (this.redis as any).set(key, value, ...args);
    } catch (error) {
      this.handleError(error, 'set', key);
      return null;
    }
  }

  async del(key: string): Promise<number> {
    try {
      return await this.redis.del(key);
    } catch (error) {
      this.handleError(error, 'del', key);
      return 0;
    }
  }

  private handleError(error: unknown, operation: string, key: string): void {
    const isNetworkError =
      error instanceof Error &&
      (error.message.includes('ECONNREFUSED') ||
        error.message.includes('ETIMEDOUT') ||
        error.message.includes('ENOTFOUND'));

    if (isNetworkError) {
      // Log แต่ไม่ throw (graceful degradation)
      logger.error('Redis connection error', { operation, key, error });
    } else {
      logger.error('Redis error', { operation, key, error });
    }
  }
}

// Circuit Breaker Pattern สำหรับ Redis
type CircuitState = 'CLOSED' | 'OPEN' | 'HALF_OPEN';

export class RedisCircuitBreaker {
  private state: CircuitState = 'CLOSED';
  private failureCount = 0;
  private lastFailureTime = 0;

  constructor(
    private redis: Redis,
    private threshold: number = 5,
    private timeout: number = 30000 // 30 seconds
  ) {}

  async execute<T>(fn: () => Promise<T>): Promise<T | null> {
    if (this.state === 'OPEN') {
      if (Date.now() - this.lastFailureTime > this.timeout) {
        this.state = 'HALF_OPEN';
      } else {
        logger.warn('Redis circuit breaker is OPEN, skipping');
        return null;
      }
    }

    try {
      const result = await fn();

      if (this.state === 'HALF_OPEN') {
        this.state = 'CLOSED';
        this.failureCount = 0;
        logger.info('Redis circuit breaker reset to CLOSED');
      }

      return result;
    } catch (error) {
      this.failureCount++;
      this.lastFailureTime = Date.now();

      if (this.failureCount >= this.threshold) {
        this.state = 'OPEN';
        logger.error('Redis circuit breaker OPEN', { failureCount: this.failureCount });
      }

      return null;
    }
  }
}
```

---

## 11. MinIO Error Handling

```typescript
// src/utils/storageErrors.ts
import { S3ServiceException } from '@aws-sdk/client-s3';
import { AppError, NotFoundError, ServiceUnavailableError } from '../errors/AppError';

export function handleStorageError(error: unknown): AppError {
  if (error instanceof S3ServiceException) {
    switch (error.name) {
      case 'NoSuchBucket':
        return new AppError('Bucket ไม่มีอยู่', 500, 'STORAGE_BUCKET_NOT_FOUND');
      case 'NoSuchKey':
        return new NotFoundError('File');
      case 'AccessDenied':
        return new AppError('ไม่มีสิทธิ์เข้าถึง storage', 500, 'STORAGE_ACCESS_DENIED');
      case 'RequestTimeout':
        return new ServiceUnavailableError('Storage service');
      case 'InternalError':
        return new ServiceUnavailableError('MinIO');
      default:
        return new AppError(`Storage error: ${error.name}`, 500, 'STORAGE_ERROR');
    }
  }

  if (error instanceof Error && error.message.includes('ECONNREFUSED')) {
    return new ServiceUnavailableError('MinIO');
  }

  return new AppError('เกิดข้อผิดพลาดใน storage', 500, 'STORAGE_ERROR');
}

// Retry wrapper สำหรับ storage operations
export async function withStorageRetry<T>(
  fn: () => Promise<T>,
  maxRetries: number = 3
): Promise<T> {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      const storageError = handleStorageError(error);

      if (
        storageError.statusCode === 503 &&
        attempt < maxRetries
      ) {
        const delay = Math.pow(2, attempt) * 1000; // Exponential backoff
        await new Promise((resolve) => setTimeout(resolve, delay));
        continue;
      }

      throw storageError;
    }
  }

  throw new ServiceUnavailableError('Storage');
}
```

---

## 12. Unhandled Rejections และ Uncaught Exceptions

```typescript
// src/server.ts
import app from './app';
import { logger } from './utils/logger';

const server = app.listen(process.env.PORT || 3000);

// Graceful shutdown
function gracefulShutdown(signal: string, error?: Error) {
  logger.info(`Received ${signal}, starting graceful shutdown`);

  if (error) {
    logger.error('Shutdown due to error', { error: error.message, stack: error.stack });
  }

  server.close(async () => {
    logger.info('HTTP server closed');

    // ปิด connections
    try {
      await pool.end();
      logger.info('Database pool closed');

      await redis.quit();
      logger.info('Redis connection closed');
    } catch (err) {
      logger.error('Error during shutdown', err);
    }

    process.exit(error ? 1 : 0);
  });

  // Force shutdown หลัง 30 วินาที
  setTimeout(() => {
    logger.error('Forced shutdown after timeout');
    process.exit(1);
  }, 30000);
}

// Unhandled Promise Rejection
process.on('unhandledRejection', (reason: unknown) => {
  logger.error('Unhandled Promise Rejection', {
    reason: reason instanceof Error ? reason.message : reason,
    stack: reason instanceof Error ? reason.stack : undefined,
  });

  // ไม่ crash ทันที แต่ log และ monitor
  // ถ้า error ร้ายแรงควร gracefulShutdown
  if (reason instanceof Error && isCriticalError(reason)) {
    gracefulShutdown('unhandledRejection', reason);
  }
});

// Uncaught Exception (synchronous)
process.on('uncaughtException', (error: Error) => {
  logger.error('Uncaught Exception', {
    error: error.message,
    stack: error.stack,
  });

  // Uncaught exceptions อาจทำให้ state เสีย ต้อง restart
  gracefulShutdown('uncaughtException', error);
});

// SIGTERM (Docker stop, Kubernetes)
process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));

// SIGINT (Ctrl+C)
process.on('SIGINT', () => gracefulShutdown('SIGINT'));

function isCriticalError(error: Error): boolean {
  return (
    error instanceof TypeError ||
    error instanceof ReferenceError ||
    error.message.includes('out of memory')
  );
}
```

---

## 13. Error Logging

```typescript
// src/utils/logger.ts (simplified - full version ใน Part 20)
import winston from 'winston';

export const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  transports: [
    new winston.transports.Console(),
    new winston.transports.File({ filename: 'logs/error.log', level: 'error' }),
  ],
});

// Log error ด้วย context
export function logError(
  error: Error,
  context?: Record<string, unknown>
): void {
  const logData: Record<string, unknown> = {
    message: error.message,
    stack: error.stack,
    ...context,
  };

  if (isAppError(error)) {
    logData.code = error.code;
    logData.statusCode = error.statusCode;
    logData.isOperational = error.isOperational;

    if (error.statusCode >= 500) {
      logger.error('Server error', logData);
    } else {
      logger.warn('Client error', logData);
    }
  } else {
    logger.error('Unexpected error', logData);
  }
}
```

---

## 14. Correlation IDs

```typescript
// src/middleware/correlationId.ts
import { Request, Response, NextFunction } from 'express';
import { v4 as uuidv4 } from 'uuid';

declare global {
  namespace Express {
    interface Request {
      id: string;
      startTime: number;
    }
  }
}

export function correlationIdMiddleware(
  req: Request,
  res: Response,
  next: NextFunction
): void {
  // ใช้ request ID จาก header (ถ้ามี) หรือสร้างใหม่
  req.id = (req.headers['x-request-id'] as string) || uuidv4();
  req.startTime = Date.now();

  // ส่ง request ID กลับไปใน response header
  res.set('X-Request-Id', req.id);

  next();
}

// Async local storage สำหรับ propagate request ID
import { AsyncLocalStorage } from 'async_hooks';

interface RequestContext {
  requestId: string;
  userId?: string;
}

export const asyncLocalStorage = new AsyncLocalStorage<RequestContext>();

export function contextMiddleware(
  req: Request,
  res: Response,
  next: NextFunction
): void {
  asyncLocalStorage.run(
    {
      requestId: req.id,
      userId: req.user?.id,
    },
    next
  );
}

// ใช้ context ใน service layer (ไม่ต้องส่ง req เป็น parameter)
export function getRequestContext(): RequestContext | undefined {
  return asyncLocalStorage.getStore();
}

// ใช้ใน logger
export function createContextLogger() {
  return {
    info: (message: string, data?: Record<string, unknown>) => {
      const context = getRequestContext();
      logger.info(message, { ...context, ...data });
    },
    error: (message: string, error?: unknown) => {
      const context = getRequestContext();
      logger.error(message, {
        ...context,
        error: error instanceof Error ? error.message : error,
      });
    },
  };
}
```

---

## 15. Full Working Implementation

```typescript
// src/app.ts — Complete setup
import express from 'express';
import { correlationIdMiddleware, contextMiddleware } from './middleware/correlationId';
import { errorHandler, notFoundHandler } from './middleware/errorHandler';
import { slidingSessionMiddleware } from './middleware/session';
import authRouter from './routes/auth';
import usersRouter from './routes/users';

const app = express();

// Request ID (ต้องอยู่ก่อน routes ทั้งหมด)
app.use(correlationIdMiddleware);
app.use(contextMiddleware);

// Parsing
app.use(express.json({ limit: '10mb' }));
app.use(express.urlencoded({ extended: true }));

// Routes
app.use('/api/auth', authRouter);
app.use('/api/users', usersRouter);

// 404 handler (ก่อน error handler)
app.use(notFoundHandler);

// Global error handler (ต้องอยู่สุดท้าย)
app.use(errorHandler);

export default app;

// src/routes/users.ts — Full example
import { Router } from 'express';
import { z } from 'zod';
import { asyncHandler } from '../utils/asyncHandler';
import { validate } from '../middleware/validate';
import { authenticate, authorize } from '../middleware/auth';
import { NotFoundError, ForbiddenError } from '../errors/AppError';
import pool from '../db/pool';
import { handleDatabaseError } from '../utils/dbErrors';

const router = Router();

const createUserSchema = z.object({
  email: z.string().email(),
  username: z.string().min(3).max(50),
  password: z.string().min(8),
  role: z.enum(['user', 'admin']).default('user'),
});

const updateUserSchema = z.object({
  email: z.string().email().optional(),
  username: z.string().min(3).max(50).optional(),
  firstName: z.string().max(100).optional(),
  lastName: z.string().max(100).optional(),
});

const userIdSchema = z.object({
  id: z.string().uuid('User ID ไม่ถูกต้อง'),
});

// GET /users
router.get(
  '/',
  authenticate,
  authorize('admin'),
  asyncHandler(async (req, res) => {
    const { page = '1', limit = '20' } = req.query;

    const offset = (Number(page) - 1) * Number(limit);

    const [users, count] = await Promise.all([
      pool.query(
        'SELECT id, email, username, role_id, created_at FROM users ORDER BY created_at DESC LIMIT $1 OFFSET $2',
        [limit, offset]
      ),
      pool.query('SELECT COUNT(*) FROM users'),
    ]);

    res.json({
      success: true,
      data: users.rows,
      meta: {
        page: Number(page),
        perPage: Number(limit),
        total: Number(count.rows[0].count),
        totalPages: Math.ceil(Number(count.rows[0].count) / Number(limit)),
      },
    });
  })
);

// GET /users/:id
router.get(
  '/:id',
  authenticate,
  validate({ params: userIdSchema }),
  asyncHandler(async (req, res) => {
    const { id } = req.params;

    // ตรวจสอบสิทธิ์: admin ดูได้ทุกคน, user ดูแค่ตัวเอง
    if (req.user!.role !== 'admin' && req.user!.id !== id) {
      throw new ForbiddenError('คุณสามารถดูเฉพาะโปรไฟล์ของตัวเองได้');
    }

    try {
      const result = await pool.query(
        `SELECT u.id, u.email, u.username, r.name as role,
                u.first_name, u.last_name, u.created_at
         FROM users u
         JOIN roles r ON u.role_id = r.id
         WHERE u.id = $1 AND u.is_active = true`,
        [id]
      );

      if (result.rows.length === 0) {
        throw new NotFoundError('User', id);
      }

      res.json({ success: true, data: result.rows[0] });
    } catch (error) {
      // Re-throw AppErrors, convert DB errors
      if (isAppError(error)) throw error;
      throw handleDatabaseError(error);
    }
  })
);

// PUT /users/:id
router.put(
  '/:id',
  authenticate,
  validate({ params: userIdSchema, body: updateUserSchema }),
  asyncHandler(async (req, res) => {
    const { id } = req.params;

    if (req.user!.role !== 'admin' && req.user!.id !== id) {
      throw new ForbiddenError();
    }

    const updates = req.body;
    const fields = Object.keys(updates);

    if (fields.length === 0) {
      return res.json({ success: true, message: 'ไม่มีข้อมูลที่ต้องอัพเดท' });
    }

    const setClause = fields
      .map((field, i) => `${field} = $${i + 2}`)
      .join(', ');

    try {
      const result = await pool.query(
        `UPDATE users SET ${setClause}, updated_at = NOW()
         WHERE id = $1 AND is_active = true
         RETURNING id, email, username`,
        [id, ...Object.values(updates)]
      );

      if (result.rowCount === 0) {
        throw new NotFoundError('User', id);
      }

      res.json({ success: true, data: result.rows[0] });
    } catch (error) {
      if (isAppError(error)) throw error;
      throw handleDatabaseError(error);
    }
  })
);

export default router;
```

---

## สรุป

| หัวข้อ | Best Practice |
|--------|--------------|
| Error Types | แยก Operational vs Programmer |
| Error Classes | ใช้ class hierarchy ที่มี statusCode |
| Error Handler | Global handler ที่ท้าย app |
| Async | ใช้ asyncHandler wrapper ทุก async route |
| Validation | Zod schema + validate middleware |
| DB Errors | Map PG error codes เป็น AppError |
| Correlation ID | ใส่ทุก request เพื่อ tracing |
| Production | ไม่ expose stack trace |
| Shutdown | Graceful shutdown + handle unhandled rejections |

# Part 46: API Versioning

## บทนำ: ทำไมต้องทำ API Versioning?

เมื่อคุณสร้าง API สาธารณะหรือ API ที่มีหลาย client ใช้งาน การเปลี่ยนแปลง API อาจทำให้ client ที่มีอยู่แล้วพัง (break) ได้ API Versioning คือกลยุทธ์ที่ช่วยให้คุณสามารถ:

1. **วิวัฒนาการ API** ได้โดยไม่ทำลาย client เดิม
2. **รองรับหลาย version** พร้อมกันชั่วคราว
3. **ค่อยๆ migrate** client จาก version เก่าไปใหม่
4. **สื่อสาร** การเปลี่ยนแปลงให้ชัดเจน

### สถานการณ์ที่ต้องการ Versioning

```
สถานการณ์จริง:
- คุณมี mobile app v1.0 ที่ client หลายแสนคนใช้งาน
- ทีมต้องการเปลี่ยน response format จาก { "name": "John" } เป็น { "firstName": "John", "lastName": "Doe" }
- ถ้าเปลี่ยนทันที → app เก่าพัง, user ไม่พอใจ
- ถ้าใช้ versioning → ทำ v2 ใหม่, v1 ยังทำงานได้, ค่อยๆ migrate
```

---

## Versioning Strategies: 4 แนวทางหลัก

### Strategy 1: URL Path Versioning (แนะนำมากที่สุด)

```
GET /api/v1/users
GET /api/v2/users
POST /api/v1/orders
POST /api/v2/orders
```

**ข้อดี:**
- เห็นชัดเจนมาก ใน URL
- Cache ง่าย (URL ต่างกัน → cache key ต่างกัน)
- Debug ง่าย ดูจาก log ก็รู้ทันที
- Browser, curl ใช้ได้ง่าย

**ข้อเสีย:**
- URL ยาวขึ้น
- ถ้ามีหลาย resource ต้องเปลี่ยนทุก URL

### Strategy 2: Query Parameter Versioning

```
GET /api/users?version=1
GET /api/users?version=2
GET /api/users?v=2
```

**ข้อดี:**
- URL base เหมือนกัน
- Optional parameter (default version ถ้าไม่ระบุ)

**ข้อเสีย:**
- Cache ซับซ้อนกว่า (ต้องรวม query param ใน cache key)
- ลืมใส่ง่าย → ได้ default version

### Strategy 3: Request Header Versioning

```http
GET /api/users HTTP/1.1
Host: api.example.com
API-Version: 2
```

หรือใช้ Accept header:

```http
GET /api/users HTTP/1.1
Host: api.example.com
Accept: application/vnd.myapp.v2+json
```

**ข้อดี:**
- URL สะอาด ไม่เปลี่ยน
- RESTful ที่สุด (URL ควร identify resource ไม่ใช่ format)

**ข้อเสีย:**
- Debug ยากกว่า (ต้องดู header)
- Browser address bar ใช้ไม่ได้โดยตรง
- Cache ต้องใช้ Vary header

### Strategy 4: Media Type / Content Negotiation

```http
GET /api/users HTTP/1.1
Accept: application/vnd.github.v3+json
Accept: application/vnd.mycompany.api+json; version=2
```

GitHub API ใช้ strategy นี้

---

## URL Versioning: Implementation แบบละเอียด

### โครงสร้างโปรเจค

```
src/
├── api/
│   ├── v1/
│   │   ├── routes/
│   │   │   ├── users.routes.ts
│   │   │   ├── posts.routes.ts
│   │   │   └── index.ts
│   │   ├── controllers/
│   │   │   ├── users.controller.ts
│   │   │   └── posts.controller.ts
│   │   └── index.ts
│   ├── v2/
│   │   ├── routes/
│   │   │   ├── users.routes.ts
│   │   │   ├── posts.routes.ts
│   │   │   └── index.ts
│   │   ├── controllers/
│   │   │   ├── users.controller.ts
│   │   │   └── posts.controller.ts
│   │   └── index.ts
│   └── router.ts
├── services/          # business logic (shared ระหว่าง versions)
│   ├── user.service.ts
│   └── post.service.ts
├── models/
│   └── user.model.ts
└── app.ts
```

### app.ts: Main Application

```typescript
import express from 'express';
import { setupRoutes } from './api/router';
import { errorHandler } from './middleware/error-handler';

const app = express();

app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// Setup all API versions
setupRoutes(app);

// Global error handler
app.use(errorHandler);

export default app;
```

### api/router.ts: Version Router

```typescript
import { Express } from 'express';
import v1Router from './v1';
import v2Router from './v2';

export function setupRoutes(app: Express): void {
  // Mount version routers
  app.use('/api/v1', v1Router);
  app.use('/api/v2', v2Router);

  // Latest version alias (optional)
  app.use('/api/latest', v2Router);

  // Default version (backward compat)
  app.use('/api', v1Router);

  // API info endpoint
  app.get('/api', (req, res) => {
    res.json({
      versions: {
        v1: {
          url: '/api/v1',
          status: 'deprecated',
          sunset: '2025-12-31',
        },
        v2: {
          url: '/api/v2',
          status: 'current',
        },
      },
      latest: '/api/v2',
      docs: 'https://docs.example.com/api',
    });
  });
}
```

### api/v1/index.ts

```typescript
import { Router } from 'express';
import usersRouter from './routes/users.routes';
import postsRouter from './routes/posts.routes';
import { deprecationMiddleware } from '../../middleware/deprecation';

const router = Router();

// Add deprecation warning for v1
router.use(deprecationMiddleware({
  version: 'v1',
  sunsetDate: '2025-12-31',
  migrationGuide: 'https://docs.example.com/migration/v1-to-v2',
}));

router.use('/users', usersRouter);
router.use('/posts', postsRouter);

export default router;
```

### api/v1/routes/users.routes.ts

```typescript
import { Router } from 'express';
import { UsersController } from '../controllers/users.controller';
import { authenticate } from '../../../middleware/auth';

const router = Router();
const controller = new UsersController();

router.get('/', authenticate, controller.getAll);
router.get('/:id', authenticate, controller.getById);
router.post('/', controller.create);
router.put('/:id', authenticate, controller.update);
router.delete('/:id', authenticate, controller.delete);

export default router;
```

### api/v1/controllers/users.controller.ts

```typescript
import { Request, Response, NextFunction } from 'express';
import { UserService } from '../../../services/user.service';

export class UsersController {
  private userService: UserService;

  constructor() {
    this.userService = new UserService();
  }

  // V1: return user with single "name" field
  getAll = async (req: Request, res: Response, next: NextFunction) => {
    try {
      const users = await this.userService.findAll();

      // V1 response format: legacy format
      const v1Users = users.map(user => ({
        id: user.id,
        name: `${user.firstName} ${user.lastName}`,  // combined name
        email: user.email,
        createdAt: user.createdAt,
      }));

      res.json({
        success: true,
        data: v1Users,
        total: v1Users.length,
      });
    } catch (error) {
      next(error);
    }
  };

  getById = async (req: Request, res: Response, next: NextFunction) => {
    try {
      const { id } = req.params;
      const user = await this.userService.findById(parseInt(id));

      if (!user) {
        return res.status(404).json({
          success: false,
          error: 'User not found',
        });
      }

      // V1 format
      res.json({
        success: true,
        data: {
          id: user.id,
          name: `${user.firstName} ${user.lastName}`,
          email: user.email,
          createdAt: user.createdAt,
        },
      });
    } catch (error) {
      next(error);
    }
  };

  create = async (req: Request, res: Response, next: NextFunction) => {
    try {
      const { name, email, password } = req.body;

      // V1: accepts single "name" field, splits it
      const [firstName, ...lastNameParts] = name.split(' ');
      const lastName = lastNameParts.join(' ');

      const user = await this.userService.create({
        firstName,
        lastName,
        email,
        password,
      });

      res.status(201).json({
        success: true,
        data: {
          id: user.id,
          name: `${user.firstName} ${user.lastName}`,
          email: user.email,
        },
      });
    } catch (error) {
      next(error);
    }
  };

  update = async (req: Request, res: Response, next: NextFunction) => {
    try {
      const { id } = req.params;
      const { name, email } = req.body;

      let updateData: any = { email };

      if (name) {
        const [firstName, ...lastNameParts] = name.split(' ');
        updateData.firstName = firstName;
        updateData.lastName = lastNameParts.join(' ');
      }

      const user = await this.userService.update(parseInt(id), updateData);

      res.json({
        success: true,
        data: {
          id: user.id,
          name: `${user.firstName} ${user.lastName}`,
          email: user.email,
        },
      });
    } catch (error) {
      next(error);
    }
  };

  delete = async (req: Request, res: Response, next: NextFunction) => {
    try {
      const { id } = req.params;
      await this.userService.delete(parseInt(id));

      res.json({
        success: true,
        message: 'User deleted successfully',
      });
    } catch (error) {
      next(error);
    }
  };
}
```

### api/v2/controllers/users.controller.ts (Breaking Changes)

```typescript
import { Request, Response, NextFunction } from 'express';
import { UserService } from '../../../services/user.service';

export class UsersController {
  private userService: UserService;

  constructor() {
    this.userService = new UserService();
  }

  // V2: return user with separate firstName/lastName + new fields
  getAll = async (req: Request, res: Response, next: NextFunction) => {
    try {
      const { page = 1, limit = 20, sort = 'createdAt', order = 'desc' } = req.query;

      const { users, total } = await this.userService.findAllPaginated({
        page: Number(page),
        limit: Number(limit),
        sort: String(sort),
        order: String(order) as 'asc' | 'desc',
      });

      // V2 response format: separate firstName/lastName + pagination
      res.json({
        success: true,
        data: users.map(user => ({
          id: user.id,
          firstName: user.firstName,   // V2: separate fields
          lastName: user.lastName,
          displayName: user.displayName || `${user.firstName} ${user.lastName}`,
          email: user.email,
          emailVerified: user.emailVerified,  // V2: new field
          role: user.role,                     // V2: new field
          avatar: user.avatar,                 // V2: new field
          createdAt: user.createdAt,
          updatedAt: user.updatedAt,           // V2: new field
        })),
        meta: {                        // V2: pagination metadata
          page: Number(page),
          limit: Number(limit),
          total,
          totalPages: Math.ceil(total / Number(limit)),
          hasNext: Number(page) < Math.ceil(total / Number(limit)),
          hasPrev: Number(page) > 1,
        },
      });
    } catch (error) {
      next(error);
    }
  };

  getById = async (req: Request, res: Response, next: NextFunction) => {
    try {
      const { id } = req.params;
      const user = await this.userService.findById(parseInt(id));

      if (!user) {
        return res.status(404).json({
          success: false,
          error: {
            code: 'USER_NOT_FOUND',
            message: 'User not found',
          },
        });
      }

      res.json({
        success: true,
        data: {
          id: user.id,
          firstName: user.firstName,
          lastName: user.lastName,
          displayName: user.displayName,
          email: user.email,
          emailVerified: user.emailVerified,
          role: user.role,
          avatar: user.avatar,
          profile: user.profile,
          createdAt: user.createdAt,
          updatedAt: user.updatedAt,
        },
      });
    } catch (error) {
      next(error);
    }
  };

  create = async (req: Request, res: Response, next: NextFunction) => {
    try {
      // V2: accepts separate firstName/lastName
      const { firstName, lastName, email, password, role = 'user' } = req.body;

      const user = await this.userService.create({
        firstName,
        lastName,
        email,
        password,
        role,
      });

      res.status(201).json({
        success: true,
        data: {
          id: user.id,
          firstName: user.firstName,
          lastName: user.lastName,
          email: user.email,
          role: user.role,
        },
      });
    } catch (error) {
      next(error);
    }
  };

  update = async (req: Request, res: Response, next: NextFunction) => {
    try {
      const { id } = req.params;
      const { firstName, lastName, email, displayName, avatar } = req.body;

      const user = await this.userService.update(parseInt(id), {
        firstName,
        lastName,
        email,
        displayName,
        avatar,
      });

      res.json({
        success: true,
        data: user,
      });
    } catch (error) {
      next(error);
    }
  };

  delete = async (req: Request, res: Response, next: NextFunction) => {
    try {
      const { id } = req.params;
      await this.userService.delete(parseInt(id));

      res.status(204).send();  // V2: 204 No Content แทน 200 + message
    } catch (error) {
      next(error);
    }
  };
}
```

---

## API Version Lifecycle

```
Alpha → Beta → GA (Generally Available) → Deprecated → Retired/Sunset
```

### Lifecycle States

| State | ความหมาย | SLA | การ Support |
|-------|-----------|-----|-------------|
| Alpha | ทดสอบ ภายใน | ไม่มี | Internal only |
| Beta | Public Preview | มีบ้าง | Best effort |
| GA | Production ready | เต็ม | Full support |
| Deprecated | ยังใช้ได้ แต่จะลบ | เต็ม | Security fixes only |
| Retired | ลบแล้ว | ไม่มี | ไม่มี |

### Version Status Headers

```typescript
// middleware/deprecation.ts
import { Request, Response, NextFunction } from 'express';

interface DeprecationOptions {
  version: string;
  sunsetDate?: string;
  migrationGuide?: string;
  status?: 'alpha' | 'beta' | 'ga' | 'deprecated';
}

export function deprecationMiddleware(options: DeprecationOptions) {
  return (req: Request, res: Response, next: NextFunction) => {
    const { version, sunsetDate, migrationGuide, status = 'deprecated' } = options;

    if (status === 'deprecated' || status === 'ga') {
      // RFC 8594 Sunset header
      if (sunsetDate) {
        res.setHeader('Sunset', new Date(sunsetDate).toUTCString());
        res.setHeader('Deprecation', 'true');
      }

      // Link to migration guide
      if (migrationGuide) {
        const existingLink = res.getHeader('Link') as string || '';
        const newLink = `<${migrationGuide}>; rel="deprecation"`;
        res.setHeader('Link', existingLink ? `${existingLink}, ${newLink}` : newLink);
      }

      // Warning header (legacy)
      res.setHeader(
        'Warning',
        `299 - "API ${version} is deprecated. ${sunsetDate ? `Will be removed on ${sunsetDate}.` : ''} ${migrationGuide ? `See ${migrationGuide} for migration guide.` : ''}"`,
      );
    }

    if (status === 'alpha' || status === 'beta') {
      res.setHeader('X-API-Version-Status', status);
      res.setHeader('Warning', `199 - "API ${version} is in ${status}. Not for production use."`);
    }

    next();
  };
}
```

---

## Breaking vs Non-Breaking Changes

### Non-Breaking Changes (Backwards Compatible)

**ทำได้ใน version เดิม ไม่ต้องขึ้น version ใหม่:**

```typescript
// ✅ เพิ่ม optional field ใหม่
// v1.1: response เดิมยังทำงานได้ client ที่ไม่รู้จัก field ใหม่จะ ignore มัน
{
  "id": 1,
  "name": "John",
  "email": "john@example.com",
  "avatar": null  // ← เพิ่มใหม่ แต่ client เก่า ignore ได้
}

// ✅ เพิ่ม endpoint ใหม่
// GET /api/v1/users/:id/profile → endpoint ใหม่, ไม่กระทบ endpoint เดิม

// ✅ เพิ่ม optional request parameter
// GET /api/v1/users?includeProfile=true → optional, ไม่ใส่ก็ได้

// ✅ เพิ่ม HTTP method สำหรับ resource เดิม
// PATCH /api/v1/users/:id → เพิ่มเติมจาก PUT ที่มีอยู่แล้ว
```

### Breaking Changes (ต้องขึ้น Version ใหม่)

```typescript
// ❌ ลบ field
// v1: { "name": "John" }
// v2: { "firstName": "John" }  ← ลบ "name" → client เก่าพัง

// ❌ เปลี่ยน type ของ field
// v1: { "id": 123 }          ← number
// v2: { "id": "user_123" }   ← string → client ที่ทำ parseInt() พัง

// ❌ เปลี่ยน required/optional
// v1: { email: optional }
// v2: { email: required } ← client เก่าที่ไม่ส่ง email จะ error

// ❌ เปลี่ยน behavior
// v1: DELETE → soft delete (status = 'deleted')
// v2: DELETE → hard delete → client ที่คิดว่า restore ได้จะพัง

// ❌ เปลี่ยน error format
// v1: { "error": "Not found" }
// v2: { "error": { "code": "NOT_FOUND", "message": "..." } }

// ❌ เปลี่ยน authentication method
// v1: API key ใน query param
// v2: Bearer token ใน header

// ❌ เปลี่ยน URL structure
// v1: /api/v1/users/:userId/posts/:postId
// v2: /api/v2/posts/:postId (ย้าย resource)
```

### Breaking Change Checklist

```typescript
// utils/api-compatibility-checker.ts
interface CompatibilityCheck {
  type: 'breaking' | 'non-breaking';
  category: string;
  description: string;
  affectedEndpoints?: string[];
}

const breakingChangePatterns: CompatibilityCheck[] = [
  {
    type: 'breaking',
    category: 'field-removal',
    description: 'Removing a previously existing response field',
  },
  {
    type: 'breaking',
    category: 'type-change',
    description: 'Changing the data type of a field',
  },
  {
    type: 'breaking',
    category: 'required-addition',
    description: 'Making an optional request field required',
  },
  {
    type: 'breaking',
    category: 'status-code-change',
    description: 'Changing the HTTP status code for a response',
  },
  {
    type: 'breaking',
    category: 'behavior-change',
    description: 'Changing what an endpoint does without changing its signature',
  },
  {
    type: 'non-breaking',
    category: 'field-addition',
    description: 'Adding a new optional response field',
  },
  {
    type: 'non-breaking',
    category: 'endpoint-addition',
    description: 'Adding a new endpoint',
  },
  {
    type: 'non-breaking',
    category: 'optional-param-addition',
    description: 'Adding a new optional request parameter',
  },
];
```

---

## Deprecation Strategy แบบละเอียด

### Step 1: ประกาศ Deprecation

```typescript
// ใน response body (optional แต่เป็น good practice)
res.json({
  success: true,
  data: users,
  _meta: {
    deprecation: {
      message: 'This endpoint is deprecated',
      sunset: '2025-12-31',
      replacement: '/api/v2/users',
      migrationGuide: 'https://docs.example.com/migration/v1-to-v2',
    },
  },
});
```

### Step 2: Deprecation Headers (RFC 8594)

```typescript
// ตาม RFC 8594: Sunset HTTP Header Field
// https://tools.ietf.org/html/rfc8594

res.setHeader('Sunset', 'Tue, 31 Dec 2025 23:59:59 GMT');
res.setHeader('Deprecation', 'Sat, 01 Jan 2025 00:00:00 GMT');  // เริ่ม deprecate วันไหน
res.setHeader(
  'Link',
  '</api/v2/users>; rel="successor-version", ' +
  '<https://docs.example.com/migration>; rel="deprecation"'
);
```

### Step 3: Track ว่าใครยังใช้ v1

```typescript
// middleware/version-tracking.ts
import { createClient } from 'redis';

const redis = createClient();

export async function versionTrackingMiddleware(
  req: Request,
  res: Response,
  next: NextFunction,
) {
  const apiVersion = req.path.match(/\/api\/(v\d+)\//)?.[1];
  
  if (apiVersion) {
    // Track ว่ามีการใช้ version นี้
    const key = `api:version:usage:${apiVersion}:${new Date().toISOString().split('T')[0]}`;
    await redis.incr(key);
    await redis.expire(key, 90 * 24 * 60 * 60);  // เก็บ 90 วัน

    // Track จาก User-Agent (รู้ว่า client ไหนยังใช้)
    const userAgent = req.headers['user-agent'];
    if (userAgent && apiVersion === 'v1') {
      const clientKey = `api:v1:clients:${Buffer.from(userAgent).toString('base64')}`;
      await redis.setEx(clientKey, 30 * 24 * 60 * 60, JSON.stringify({
        userAgent,
        lastSeen: new Date().toISOString(),
        endpoint: req.path,
      }));
    }
  }

  next();
}

// ดูว่า client ไหนยังใช้ v1
export async function getV1Clients() {
  const keys = await redis.keys('api:v1:clients:*');
  const clients = [];

  for (const key of keys) {
    const data = await redis.get(key);
    if (data) clients.push(JSON.parse(data));
  }

  return clients;
}
```

### Step 4: Migration Guide ที่ดี

```markdown
# Migration Guide: API v1 → v2

## Breaking Changes

### 1. User Response Format

**v1:**
```json
{
  "id": 1,
  "name": "John Doe",
  "email": "john@example.com"
}
```

**v2:**
```json
{
  "id": 1,
  "firstName": "John",
  "lastName": "Doe",
  "displayName": "John Doe",
  "email": "john@example.com",
  "emailVerified": true,
  "role": "user",
  "createdAt": "2024-01-01T00:00:00Z"
}
```

**การ migrate:**
```javascript
// v1 code:
const name = user.name;

// v2 code:
const name = user.displayName || `${user.firstName} ${user.lastName}`;
```
```

---

## Version Routing Middleware

```typescript
// middleware/version-router.ts
import { Request, Response, NextFunction } from 'express';

type VersionHandler = (req: Request, res: Response, next: NextFunction) => void;

interface VersionMap {
  [version: string]: VersionHandler;
}

// Route request to different handlers based on version
export function versionRoute(versionMap: VersionMap, defaultVersion = 'v1') {
  return (req: Request, res: Response, next: NextFunction) => {
    // Extract version from URL, header, or query param
    const version =
      extractVersionFromUrl(req) ||
      extractVersionFromHeader(req) ||
      extractVersionFromQuery(req) ||
      defaultVersion;

    const handler = versionMap[version] || versionMap[defaultVersion];

    if (!handler) {
      return res.status(400).json({
        error: `Unsupported API version: ${version}. Supported versions: ${Object.keys(versionMap).join(', ')}`,
      });
    }

    // Store resolved version in request
    (req as any).apiVersion = version;

    handler(req, res, next);
  };
}

function extractVersionFromUrl(req: Request): string | null {
  const match = req.path.match(/\/api\/(v\d+)\//);
  return match ? match[1] : null;
}

function extractVersionFromHeader(req: Request): string | null {
  const version = req.headers['api-version'] || req.headers['x-api-version'];
  return version ? String(version) : null;
}

function extractVersionFromQuery(req: Request): string | null {
  return req.query.version ? `v${req.query.version}` : null;
}

// Usage:
// app.get('/api/users', versionRoute({
//   v1: v1UsersController.getAll,
//   v2: v2UsersController.getAll,
// }));
```

---

## API Changelog

```markdown
# API Changelog

## v2.0.0 (2024-06-01)

### Breaking Changes
- **Users**: Removed `name` field. Use `firstName`, `lastName`, `displayName` instead
- **Users**: `DELETE /users/:id` now returns 204 No Content instead of 200
- **Auth**: JWT token format changed, old tokens invalid
- **Errors**: Error response format changed to `{ error: { code, message, details } }`

### New Features
- **Users**: Added `role`, `avatar`, `emailVerified` fields
- **Users**: Added pagination to `GET /users` (page, limit, sort, order)
- **Posts**: New endpoint `GET /users/:id/timeline`
- **Search**: New `GET /search?q=` global search endpoint

### Improvements
- Response times improved by 40% with query optimization
- Rate limits increased from 100 to 500 req/min

---

## v1.2.0 (2024-03-01)

### New Features (Non-Breaking)
- Added `GET /users/:id/avatar` endpoint
- Added optional `includeProfile=true` query param to `GET /users`

---

## v1.1.0 (2024-01-15)

### New Features (Non-Breaking)
- Added `createdAt` field to user response
- Added `PATCH /users/:id` for partial updates

---

## v1.0.0 (2024-01-01) - Initial Release
```

---

## Semantic Versioning สำหรับ APIs

```
MAJOR.MINOR.PATCH
  │      │     └─ Bug fixes (backwards compatible)
  │      └─────── New features (backwards compatible)
  └────────────── Breaking changes
```

### ควรขึ้น version เมื่อไหร่?

```typescript
interface ApiVersionDecision {
  changeType: 'patch' | 'minor' | 'major';
  examples: string[];
  urlChange: boolean;
}

const versioningRules: ApiVersionDecision[] = [
  {
    changeType: 'patch',
    examples: ['Bug fix ใน response', 'Performance improvement', 'Fix typo ใน error message'],
    urlChange: false,  // /api/v1 → /api/v1 (ไม่เปลี่ยน)
  },
  {
    changeType: 'minor',
    examples: ['เพิ่ม optional field', 'เพิ่ม endpoint ใหม่', 'เพิ่ม optional parameter'],
    urlChange: false,  // /api/v1 → /api/v1 (ไม่เปลี่ยน)
  },
  {
    changeType: 'major',
    examples: ['ลบ field', 'เปลี่ยน type', 'เปลี่ยน behavior', 'เปลี่ยน auth'],
    urlChange: true,  // /api/v1 → /api/v2 (เปลี่ยน)
  },
];
```

---

## OpenAPI/Swagger Versioning

```yaml
# swagger/v1.yaml
openapi: 3.0.0

info:
  title: My API
  version: 1.2.0
  description: |
    **⚠️ Deprecated**: This version is deprecated and will be removed on 2025-12-31.
    Please migrate to [v2](/api/v2/docs).
  contact:
    name: API Support
    email: api-support@example.com

servers:
  - url: https://api.example.com/api/v1
    description: Production (v1 - Deprecated)
  - url: https://staging-api.example.com/api/v1
    description: Staging (v1 - Deprecated)

x-api-version: "1.2.0"
x-api-status: "deprecated"
x-api-sunset: "2025-12-31"
```

```yaml
# swagger/v2.yaml
openapi: 3.0.0

info:
  title: My API v2
  version: 2.0.0
  description: Current stable version

servers:
  - url: https://api.example.com/api/v2
    description: Production (v2 - Current)

x-api-version: "2.0.0"
x-api-status: "ga"
```

### Setup Swagger UI สำหรับหลาย Versions

```typescript
// app.ts
import swaggerUi from 'swagger-ui-express';
import swaggerV1 from './swagger/v1.json';
import swaggerV2 from './swagger/v2.json';

// Serve v1 docs
app.use('/api/v1/docs', swaggerUi.serveFiles(swaggerV1), swaggerUi.setup(swaggerV1));

// Serve v2 docs
app.use('/api/v2/docs', swaggerUi.serveFiles(swaggerV2), swaggerUi.setup(swaggerV2));

// Docs index
app.get('/docs', (req, res) => {
  res.json({
    v1: { url: '/api/v1/docs', status: 'deprecated' },
    v2: { url: '/api/v2/docs', status: 'current' },
  });
});
```

---

## Full Working Example: Express.js v1 + v2 API

```typescript
// server.ts - Complete working example

import express, { Express, Request, Response, NextFunction } from 'express';
import cors from 'cors';
import helmet from 'helmet';

// ============================================================
// Types
// ============================================================

interface User {
  id: number;
  firstName: string;
  lastName: string;
  email: string;
  role: 'admin' | 'user';
  createdAt: Date;
}

// ============================================================
// In-memory "database" สำหรับ demo
// ============================================================

let users: User[] = [
  { id: 1, firstName: 'สมชาย', lastName: 'ใจดี', email: 'somchai@example.com', role: 'user', createdAt: new Date('2024-01-01') },
  { id: 2, firstName: 'สมหญิง', lastName: 'รักดี', email: 'somying@example.com', role: 'admin', createdAt: new Date('2024-01-02') },
  { id: 3, firstName: 'วิชัย', lastName: 'สมบูรณ์', email: 'wichai@example.com', role: 'user', createdAt: new Date('2024-01-03') },
];

let nextId = 4;

// ============================================================
// Middleware
// ============================================================

function deprecationWarning(sunsetDate: string, migrationUrl: string) {
  return (req: Request, res: Response, next: NextFunction) => {
    res.setHeader('Deprecation', 'true');
    res.setHeader('Sunset', new Date(sunsetDate).toUTCString());
    res.setHeader('Link', `<${migrationUrl}>; rel="deprecation"`);
    next();
  };
}

function requestLogger(req: Request, res: Response, next: NextFunction) {
  const start = Date.now();
  res.on('finish', () => {
    const duration = Date.now() - start;
    console.log(`${req.method} ${req.path} → ${res.statusCode} (${duration}ms)`);
  });
  next();
}

// ============================================================
// V1 API Routes (Deprecated)
// ============================================================

function createV1Router() {
  const router = express.Router();

  // Deprecation warning for all v1 routes
  router.use(deprecationWarning('2025-12-31', 'https://docs.example.com/migration/v1-to-v2'));

  // GET /api/v1/users
  router.get('/users', (req, res) => {
    const v1Users = users.map(u => ({
      id: u.id,
      name: `${u.firstName} ${u.lastName}`,  // V1: combined name
      email: u.email,
    }));

    res.json({ success: true, data: v1Users });
  });

  // GET /api/v1/users/:id
  router.get('/users/:id', (req, res) => {
    const user = users.find(u => u.id === parseInt(req.params.id));

    if (!user) {
      return res.status(404).json({ success: false, error: 'User not found' });
    }

    res.json({
      success: true,
      data: {
        id: user.id,
        name: `${user.firstName} ${user.lastName}`,
        email: user.email,
      },
    });
  });

  // POST /api/v1/users
  router.post('/users', (req, res) => {
    const { name, email } = req.body;

    if (!name || !email) {
      return res.status(400).json({ success: false, error: 'name and email required' });
    }

    const [firstName, ...rest] = name.split(' ');
    const lastName = rest.join(' ');

    const newUser: User = {
      id: nextId++,
      firstName,
      lastName,
      email,
      role: 'user',
      createdAt: new Date(),
    };

    users.push(newUser);

    res.status(201).json({
      success: true,
      data: { id: newUser.id, name, email },
    });
  });

  // PUT /api/v1/users/:id
  router.put('/users/:id', (req, res) => {
    const index = users.findIndex(u => u.id === parseInt(req.params.id));

    if (index === -1) {
      return res.status(404).json({ success: false, error: 'User not found' });
    }

    const { name, email } = req.body;
    if (name) {
      const [firstName, ...rest] = name.split(' ');
      users[index].firstName = firstName;
      users[index].lastName = rest.join(' ');
    }
    if (email) users[index].email = email;

    const u = users[index];
    res.json({
      success: true,
      data: { id: u.id, name: `${u.firstName} ${u.lastName}`, email: u.email },
    });
  });

  // DELETE /api/v1/users/:id
  router.delete('/users/:id', (req, res) => {
    const index = users.findIndex(u => u.id === parseInt(req.params.id));

    if (index === -1) {
      return res.status(404).json({ success: false, error: 'User not found' });
    }

    users.splice(index, 1);

    // V1: returns 200 + message
    res.json({ success: true, message: 'User deleted' });
  });

  return router;
}

// ============================================================
// V2 API Routes (Current)
// ============================================================

function createV2Router() {
  const router = express.Router();

  // GET /api/v2/users (with pagination)
  router.get('/users', (req, res) => {
    const page = parseInt(req.query.page as string) || 1;
    const limit = parseInt(req.query.limit as string) || 10;
    const sort = (req.query.sort as string) || 'id';
    const order = (req.query.order as string) || 'asc';

    // Sort
    const sorted = [...users].sort((a, b) => {
      const aVal = (a as any)[sort];
      const bVal = (b as any)[sort];
      return order === 'asc'
        ? String(aVal).localeCompare(String(bVal))
        : String(bVal).localeCompare(String(aVal));
    });

    // Paginate
    const total = sorted.length;
    const start = (page - 1) * limit;
    const paged = sorted.slice(start, start + limit);

    res.json({
      success: true,
      data: paged.map(u => ({
        id: u.id,
        firstName: u.firstName,   // V2: separate fields
        lastName: u.lastName,
        displayName: `${u.firstName} ${u.lastName}`,
        email: u.email,
        role: u.role,
        createdAt: u.createdAt,
      })),
      meta: {
        page,
        limit,
        total,
        totalPages: Math.ceil(total / limit),
        hasNext: page < Math.ceil(total / limit),
        hasPrev: page > 1,
      },
    });
  });

  // GET /api/v2/users/:id
  router.get('/users/:id', (req, res) => {
    const user = users.find(u => u.id === parseInt(req.params.id));

    if (!user) {
      // V2: structured error
      return res.status(404).json({
        success: false,
        error: {
          code: 'USER_NOT_FOUND',
          message: `User with id ${req.params.id} not found`,
        },
      });
    }

    res.json({
      success: true,
      data: {
        id: user.id,
        firstName: user.firstName,
        lastName: user.lastName,
        displayName: `${user.firstName} ${user.lastName}`,
        email: user.email,
        role: user.role,
        createdAt: user.createdAt,
      },
    });
  });

  // POST /api/v2/users
  router.post('/users', (req, res) => {
    // V2: separate firstName/lastName
    const { firstName, lastName, email, role = 'user' } = req.body;

    if (!firstName || !email) {
      return res.status(400).json({
        success: false,
        error: {
          code: 'VALIDATION_ERROR',
          message: 'firstName and email are required',
          fields: {
            firstName: !firstName ? 'required' : undefined,
            email: !email ? 'required' : undefined,
          },
        },
      });
    }

    const newUser: User = {
      id: nextId++,
      firstName,
      lastName: lastName || '',
      email,
      role,
      createdAt: new Date(),
    };

    users.push(newUser);

    res.status(201).json({
      success: true,
      data: {
        id: newUser.id,
        firstName: newUser.firstName,
        lastName: newUser.lastName,
        email: newUser.email,
        role: newUser.role,
        createdAt: newUser.createdAt,
      },
    });
  });

  // PATCH /api/v2/users/:id
  router.patch('/users/:id', (req, res) => {
    const index = users.findIndex(u => u.id === parseInt(req.params.id));

    if (index === -1) {
      return res.status(404).json({
        success: false,
        error: { code: 'USER_NOT_FOUND', message: 'User not found' },
      });
    }

    const { firstName, lastName, email, role } = req.body;
    if (firstName !== undefined) users[index].firstName = firstName;
    if (lastName !== undefined) users[index].lastName = lastName;
    if (email !== undefined) users[index].email = email;
    if (role !== undefined) users[index].role = role;

    const u = users[index];
    res.json({
      success: true,
      data: {
        id: u.id,
        firstName: u.firstName,
        lastName: u.lastName,
        email: u.email,
        role: u.role,
        createdAt: u.createdAt,
      },
    });
  });

  // DELETE /api/v2/users/:id
  router.delete('/users/:id', (req, res) => {
    const index = users.findIndex(u => u.id === parseInt(req.params.id));

    if (index === -1) {
      return res.status(404).json({
        success: false,
        error: { code: 'USER_NOT_FOUND', message: 'User not found' },
      });
    }

    users.splice(index, 1);

    // V2: 204 No Content
    res.status(204).send();
  });

  return router;
}

// ============================================================
// App Setup
// ============================================================

const app: Express = express();

app.use(helmet());
app.use(cors());
app.use(express.json());
app.use(requestLogger);

// Mount version routers
app.use('/api/v1', createV1Router());
app.use('/api/v2', createV2Router());

// API info
app.get('/api', (req, res) => {
  res.json({
    name: 'My API',
    versions: {
      v1: { status: 'deprecated', sunset: '2025-12-31', url: '/api/v1' },
      v2: { status: 'current', url: '/api/v2' },
    },
    docs: '/api/v2/docs',
  });
});

// 404 handler
app.use((req, res) => {
  res.status(404).json({
    error: { code: 'NOT_FOUND', message: `Route ${req.method} ${req.path} not found` },
  });
});

// Error handler
app.use((err: Error, req: Request, res: Response, next: NextFunction) => {
  console.error(err);
  res.status(500).json({
    error: { code: 'INTERNAL_ERROR', message: 'Internal server error' },
  });
});

// Start server
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Server running on http://localhost:${PORT}`);
  console.log(`API v1: http://localhost:${PORT}/api/v1 (deprecated)`);
  console.log(`API v2: http://localhost:${PORT}/api/v2 (current)`);
});
```

---

## ทดสอบ API

```bash
# V1 API (deprecated)
curl http://localhost:3000/api/v1/users
# Response headers จะมี:
# Deprecation: true
# Sunset: Tue, 31 Dec 2025 23:59:59 GMT

# V2 API (current) with pagination
curl "http://localhost:3000/api/v2/users?page=1&limit=2&sort=firstName&order=asc"

# Create user V1 (single name)
curl -X POST http://localhost:3000/api/v1/users \
  -H "Content-Type: application/json" \
  -d '{"name": "สมพร ใจดี", "email": "somporn@example.com"}'

# Create user V2 (separate names)
curl -X POST http://localhost:3000/api/v2/users \
  -H "Content-Type: application/json" \
  -d '{"firstName": "สมพร", "lastName": "ใจดี", "email": "somporn@example.com"}'

# Delete V1 (200 + message)
curl -X DELETE http://localhost:3000/api/v1/users/1
# Response: { "success": true, "message": "User deleted" }

# Delete V2 (204 No Content)
curl -X DELETE http://localhost:3000/api/v2/users/1
# Response: (empty, status 204)
```

---

## Best Practices Summary

```
1. เลือก URL Versioning เป็น default สำหรับ public APIs
2. ขึ้น version ใหม่เมื่อมี breaking changes เท่านั้น
3. ส่ง Deprecation และ Sunset headers ก่อน retire
4. ให้ notice อย่างน้อย 6-12 เดือน ก่อน retire version
5. สร้าง migration guide ที่ละเอียดและมีตัวอย่าง code
6. Track ว่า client ไหนยังใช้ version เก่า
7. Keep backward compatibility ให้นานที่สุดเท่าที่ทำได้
8. Document breaking changes ใน CHANGELOG
9. สร้าง API versioning policy ที่ชัดเจนและ communicate ให้ developer รู้ล่วงหน้า
10. ใช้ Semantic Versioning (MAJOR.MINOR.PATCH) ในการบริหาร version
```

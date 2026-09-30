# Part 43: Pagination Strategies

## บทนำ: ทำไมต้องมี Pagination?

ลองนึกภาพว่า API ของคุณมีข้อมูล 10 ล้าน records แล้ว client ทำ request:
```
GET /api/products
```

ถ้าไม่มี pagination จะเกิดอะไรขึ้น?

```
1. Server ต้องดึงข้อมูล 10 ล้าน records จาก DB
2. Serialize เป็น JSON (อาจใช้ RAM หลาย GB)
3. ส่งข้อมูล GB ข้ามเครือข่าย
4. Client ต้องรับและ parse JSON ขนาดมหึมา
5. ผลลัพธ์: Timeout, Out of Memory, Bad User Experience
```

Pagination คือการแบ่งข้อมูลออกเป็นหน้าๆ เพื่อ:

```
✅ ลด response size
✅ ลด server memory usage
✅ ลด database load
✅ เร็วขึ้นสำหรับ user (ดูข้อมูลได้ทันที)
✅ Enable infinite scroll, load more patterns
```

## Offset-based Pagination (วิธีแบบดั้งเดิม)

### หลักการทำงาน

```sql
-- หน้า 1: ข้อมูล 20 รายการแรก
SELECT * FROM products
ORDER BY created_at DESC
LIMIT 20 OFFSET 0;

-- หน้า 2: ข้อมูล 20 รายการถัดไป
SELECT * FROM products
ORDER BY created_at DESC
LIMIT 20 OFFSET 20;

-- หน้า N:
SELECT * FROM products
ORDER BY created_at DESC
LIMIT 20 OFFSET (N-1) * 20;
```

### API Response Format

```typescript
// Request: GET /api/products?page=2&pageSize=20
// Response:
{
  "data": [
    { "id": "21", "name": "Product 21", ... },
    { "id": "22", "name": "Product 22", ... },
    // ... 20 items
  ],
  "pagination": {
    "page": 2,
    "pageSize": 20,
    "total": 1543,
    "totalPages": 78,
    "hasNextPage": true,
    "hasPreviousPage": true
  }
}
```

### TypeScript Implementation

```typescript
// src/types/pagination.ts
export interface PaginationParams {
  page?: number;
  pageSize?: number;
}

export interface PaginatedResponse<T> {
  data: T[];
  pagination: {
    page: number;
    pageSize: number;
    total: number;
    totalPages: number;
    hasNextPage: boolean;
    hasPreviousPage: boolean;
  };
}

// src/utils/pagination.ts
export function parsePaginationParams(
  query: Record<string, string | string[] | undefined>
): { page: number; pageSize: number; offset: number } {
  const page = Math.max(1, parseInt(String(query.page || "1")));
  const pageSize = Math.min(
    100,  // max pageSize
    Math.max(1, parseInt(String(query.pageSize || query.limit || "20")))
  );
  const offset = (page - 1) * pageSize;
  
  return { page, pageSize, offset };
}

export function buildPaginationMeta(
  page: number,
  pageSize: number,
  total: number
) {
  const totalPages = Math.ceil(total / pageSize);
  
  return {
    page,
    pageSize,
    total,
    totalPages,
    hasNextPage: page < totalPages,
    hasPreviousPage: page > 1,
  };
}
```

### Database Query สำหรับ Offset Pagination

```typescript
// src/services/product.service.ts
import { db } from "../db";
import { parsePaginationParams, buildPaginationMeta, PaginatedResponse } from "../utils/pagination";

interface ProductFilters {
  category?: string;
  minPrice?: number;
  maxPrice?: number;
  search?: string;
}

async function getProducts(
  query: Record<string, string>,
  filters: ProductFilters = {}
): Promise<PaginatedResponse<Product>> {
  const { page, pageSize, offset } = parsePaginationParams(query);
  
  // Build WHERE conditions
  const where: any = { deletedAt: null };
  
  if (filters.category) {
    where.categoryId = filters.category;
  }
  if (filters.minPrice !== undefined || filters.maxPrice !== undefined) {
    where.price = {};
    if (filters.minPrice !== undefined) where.price.gte = filters.minPrice;
    if (filters.maxPrice !== undefined) where.price.lte = filters.maxPrice;
  }
  if (filters.search) {
    where.OR = [
      { name: { contains: filters.search, mode: "insensitive" } },
      { description: { contains: filters.search, mode: "insensitive" } },
    ];
  }
  
  // Fetch data + count in parallel
  const [products, total] = await Promise.all([
    db.product.findMany({
      where,
      skip: offset,
      take: pageSize,
      orderBy: { createdAt: "desc" },
      include: {
        category: { select: { id: true, name: true } },
        images: { take: 1, orderBy: { position: "asc" } },
      },
    }),
    db.product.count({ where }),
  ]);
  
  return {
    data: products,
    pagination: buildPaginationMeta(page, pageSize, total),
  };
}
```

### Express Route

```typescript
// src/routes/products.ts
import { Router } from "express";

const router = Router();

// GET /api/products?page=1&pageSize=20&category=electronics&minPrice=100&maxPrice=500&search=laptop
router.get("/", async (req, res) => {
  const { category, minPrice, maxPrice, search } = req.query;
  
  const result = await getProducts(req.query as any, {
    category: category as string,
    minPrice: minPrice ? parseFloat(minPrice as string) : undefined,
    maxPrice: maxPrice ? parseFloat(maxPrice as string) : undefined,
    search: search as string,
  });
  
  res.json(result);
});
```

### ปัญหาของ Offset Pagination

```sql
-- ปัญหา 1: Performance ที่ offset สูง
-- PostgreSQL ต้องอ่านและข้ามข้อมูลทุก row ก่อน offset
SELECT * FROM products
ORDER BY created_at DESC
LIMIT 20 OFFSET 1000000;  -- ต้องอ่าน 1,000,020 rows!

-- Query plan แสดงให้เห็น:
-- Seq Scan หรือ Index Scan: cost สูงมาก
-- actual rows=20 loops=1 (แต่ต้องผ่าน 1M rows ก่อน)

-- ทดสอบ:
EXPLAIN ANALYZE
SELECT * FROM products
ORDER BY created_at DESC
LIMIT 20 OFFSET 1000000;
-- Execution Time: ~5000ms (5 วินาที!)
```

```sql
-- ปัญหา 2: Inconsistent results (ข้อมูลเปลี่ยนระหว่าง pagination)

-- User ดูหน้า 1 (items 1-20)
-- มีคนเพิ่มข้อมูลใหม่ 5 รายการ (เพราะ sort DESC ใหม่ขึ้นมาก่อน)
-- User ดูหน้า 2 (items 21-40)
-- แต่จริงๆ item 16-20 ถูก "ดัน" ไปหน้า 2 แล้ว
-- ผลลัพธ์: User เห็น item ซ้ำ!

-- ตัวอย่าง:
-- หน้า 1: [ID: 100, 99, 98, 97, 96, 95, 94, 93, 92, 91]
-- เพิ่ม ID: 101, 102, 103
-- หน้า 2 ตอนนี้: [ID: 98, 97, 96, 95, 94, 93, 92, 91, 90, 89] ← ซ้ำ!
```

```sql
-- ปัญหา 3: Count query ช้า
SELECT COUNT(*) FROM products WHERE ...;
-- Full table scan หรือ index scan ทั้งหมด = ช้า
```

### แก้ปัญหา Count ด้วย Estimated Count

```sql
-- วิธีที่ 1: pg_class (fast, approximate)
-- เร็วมาก แต่ไม่แม่นยำ 100% (อาจ off ±10%)
SELECT reltuples::bigint AS estimate
FROM pg_class
WHERE relname = 'products';

-- วิธีที่ 2: EXPLAIN (ดีกว่า)
EXPLAIN SELECT * FROM products WHERE category_id = 5;
-- อ่าน rows estimate จาก explain plan

-- วิธีที่ 3: count cache
-- เก็บ count ใน Redis และ update ด้วย triggers
```

```typescript
// Estimated count สำหรับ large tables
async function getEstimatedCount(tableName: string, whereClause?: string): Promise<number> {
  // ใช้ EXPLAIN เพื่อ estimate
  const query = `EXPLAIN SELECT 1 FROM ${tableName}${whereClause ? ` WHERE ${whereClause}` : ""}`;
  const result = await db.$queryRaw<Array<{ "QUERY PLAN": string }>>`${query}`;
  
  const planText = result[0]["QUERY PLAN"];
  const match = planText.match(/rows=(\d+)/);
  
  return match ? parseInt(match[1]) : 0;
}
```

## Cursor-based Pagination

### หลักการ

```
แทนที่จะบอกว่า "ข้ามไป N รายการ" (OFFSET)
เราบอกว่า "ดึงข้อมูลหลังจาก item นี้" (WHERE id > cursor)

ข้อดี:
✅ O(log n) ไม่ใช่ O(n) (ใช้ index ได้เต็มที่)
✅ Consistent results (ไม่มีข้อมูลซ้ำ/หาย)
✅ เหมาะกับ infinite scroll
✅ Real-time data ที่เปลี่ยนบ่อย

ข้อเสีย:
❌ ไม่สามารถ jump ไปหน้าที่ต้องการได้
❌ ไม่รู้ total count ง่ายๆ
❌ Bidirectional ซับซ้อนกว่า
```

### Simple Cursor: Last ID

```sql
-- ดึงครั้งแรก (ไม่มี cursor)
SELECT * FROM posts
ORDER BY id DESC
LIMIT 20;
-- ได้ ID สุดท้าย: 80

-- ดึงครั้งต่อไป (cursor = 80)
SELECT * FROM posts
WHERE id < 80
ORDER BY id DESC
LIMIT 20;
-- ได้ ID สุดท้าย: 60

-- ดึงครั้งต่อไป (cursor = 60)
SELECT * FROM posts
WHERE id < 60
ORDER BY id DESC
LIMIT 20;
```

### Encoded Cursor

```typescript
// Cursor ควรเป็น opaque (ผู้ใช้ไม่รู้ว่าข้างในคืออะไร)
// เพื่อให้เปลี่ยน implementation ได้ในอนาคต

function encodeCursor(data: Record<string, unknown>): string {
  const json = JSON.stringify(data);
  return Buffer.from(json).toString("base64url"); // URL-safe base64
}

function decodeCursor<T = Record<string, unknown>>(cursor: string): T {
  try {
    const json = Buffer.from(cursor, "base64url").toString("utf8");
    return JSON.parse(json);
  } catch {
    throw new Error("Invalid cursor");
  }
}

// ตัวอย่าง:
const cursor = encodeCursor({ id: 80 });
// "eyJpZCI6ODB9" (base64url)

const decoded = decodeCursor<{ id: number }>(cursor);
// { id: 80 }
```

### Cursor Pagination Implementation

```typescript
// src/utils/cursor-pagination.ts

interface CursorPaginationParams {
  after?: string;     // cursor หลังจาก item นี้ (next page)
  before?: string;    // cursor ก่อน item นี้ (previous page)
  limit?: number;
}

interface CursorPaginatedResponse<T> {
  data: T[];
  pageInfo: {
    hasNextPage: boolean;
    hasPreviousPage: boolean;
    startCursor: string | null;
    endCursor: string | null;
  };
}

// src/services/post.service.ts (with cursor pagination)
async function getPosts(
  params: CursorPaginationParams
): Promise<CursorPaginatedResponse<Post>> {
  const { after, before, limit = 20 } = params;
  
  // Fetch 1 extra item เพื่อ detect hasNextPage/hasPreviousPage
  const take = limit + 1;
  
  const where: any = {};
  let isBackward = false;
  
  if (after) {
    const cursor = decodeCursor<{ id: number }>(after);
    where.id = { lt: cursor.id }; // ใช้ lt เพราะ sort DESC
  } else if (before) {
    const cursor = decodeCursor<{ id: number }>(before);
    where.id = { gt: cursor.id };
    isBackward = true;
  }
  
  let posts = await db.post.findMany({
    where,
    take: isBackward ? -take : take,  // negative = take from end
    orderBy: { id: "desc" },
    include: { author: { select: { id: true, name: true } } },
  });
  
  let hasNextPage = false;
  let hasPreviousPage = false;
  
  if (isBackward) {
    hasPreviousPage = posts.length > limit;
    if (hasPreviousPage) posts.shift(); // ลบ extra item
    hasNextPage = after !== undefined;
  } else {
    hasNextPage = posts.length > limit;
    if (hasNextPage) posts.pop(); // ลบ extra item
    hasPreviousPage = after !== undefined;
  }
  
  const startCursor = posts.length > 0
    ? encodeCursor({ id: posts[0].id })
    : null;
  const endCursor = posts.length > 0
    ? encodeCursor({ id: posts[posts.length - 1].id })
    : null;
  
  return {
    data: posts,
    pageInfo: {
      hasNextPage,
      hasPreviousPage,
      startCursor,
      endCursor,
    },
  };
}
```

## Keyset Pagination สำหรับ Multi-column Sort

### ปัญหา: Sort ด้วย non-unique field

```sql
-- ถ้า sort ด้วย created_at ที่อาจซ้ำกัน
-- cursor แค่ created_at ไม่พอ!

-- สมมติมีข้อมูล:
-- ID=5, created_at='2024-01-01 10:00'
-- ID=6, created_at='2024-01-01 10:00'  ← created_at เหมือนกัน!
-- ID=7, created_at='2024-01-01 10:00'

-- ถ้าใช้แค่ created_at เป็น cursor:
-- Page 1: [ID=5] cursor='2024-01-01 10:00'
-- Page 2: WHERE created_at < '2024-01-01 10:00' → ข้าม ID=6, ID=7!

-- วิธีแก้: composite cursor (created_at + id)
```

### Composite Cursor SQL

```sql
-- Sort DESC by created_at, then DESC by id (tie-breaker)
-- Page 1: ไม่มี cursor
SELECT * FROM posts
ORDER BY created_at DESC, id DESC
LIMIT 20;
-- ได้: last row = { created_at: '2024-01-01 10:00', id: 5 }
-- cursor = encode({ date: '2024-01-01T10:00:00Z', id: 5 })

-- Page 2: WHERE (created_at, id) < (cursor_date, cursor_id)
SELECT * FROM posts
WHERE (created_at, id) < ('2024-01-01 10:00', 5)
ORDER BY created_at DESC, id DESC
LIMIT 20;

-- หรือแบบ explicit (บาง DB ไม่รองรับ tuple comparison):
SELECT * FROM posts
WHERE created_at < '2024-01-01 10:00'
   OR (created_at = '2024-01-01 10:00' AND id < 5)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

### TypeScript: Generic Cursor Pagination Utility

```typescript
// src/utils/keyset-pagination.ts
import { Pool } from "pg";

interface SortField {
  column: string;
  direction: "ASC" | "DESC";
  type?: "number" | "string" | "date";
}

interface KeysetCursorOptions {
  table: string;
  fields: SortField[];
  limit: number;
  cursor?: string;
  where?: string;
  params?: unknown[];
}

interface KeysetResult<T> {
  rows: T[];
  nextCursor: string | null;
  hasMore: boolean;
}

const pool = new Pool({ connectionString: process.env.DATABASE_URL });

export async function keysetPaginate<T>(
  options: KeysetCursorOptions
): Promise<KeysetResult<T>> {
  const { table, fields, limit, cursor, where = "1=1", params = [] } = options;
  
  let cursorCondition = "";
  let cursorParams: unknown[] = [];
  
  if (cursor) {
    const cursorData = decodeCursor<Record<string, unknown>>(cursor);
    
    // สร้าง WHERE clause สำหรับ keyset
    // (col1, col2) < (val1, val2) เทียบเท่า:
    // col1 < val1 OR (col1 = val1 AND col2 < val2)
    cursorCondition = buildKeysetCondition(fields, cursorData, params.length + 1);
    cursorParams = fields.map(f => cursorData[f.column]);
  }
  
  const orderBy = fields
    .map(f => `${f.column} ${f.direction}`)
    .join(", ");
  
  const allParams = [...params, ...cursorParams];
  
  const query = `
    SELECT *
    FROM ${table}
    WHERE (${where})
      ${cursorCondition ? `AND (${cursorCondition})` : ""}
    ORDER BY ${orderBy}
    LIMIT $${allParams.length + 1}
  `;
  
  const result = await pool.query<T>(query, [...allParams, limit + 1]);
  const rows = result.rows;
  
  const hasMore = rows.length > limit;
  if (hasMore) rows.pop();
  
  const nextCursor = hasMore && rows.length > 0
    ? encodeCursor(
        Object.fromEntries(
          fields.map(f => [f.column, (rows[rows.length - 1] as any)[f.column]])
        )
      )
    : null;
  
  return { rows, nextCursor, hasMore };
}

function buildKeysetCondition(
  fields: SortField[],
  cursorData: Record<string, unknown>,
  paramOffset: number
): string {
  if (fields.length === 0) return "1=1";
  
  const conditions: string[] = [];
  let paramIndex = paramOffset;
  
  for (let i = 0; i < fields.length; i++) {
    const parts: string[] = [];
    
    // col1 = v1 AND col2 = v2 AND ... AND col_i < v_i
    for (let j = 0; j < i; j++) {
      parts.push(`${fields[j].column} = $${paramOffset + j}`);
    }
    
    const op = fields[i].direction === "DESC" ? "<" : ">";
    parts.push(`${fields[i].column} ${op} $${paramOffset + i}`);
    
    conditions.push(`(${parts.join(" AND ")})`);
  }
  
  return conditions.join(" OR ");
}
```

### ตัวอย่างการใช้งาน

```typescript
// ดึง posts เรียงตาม created_at DESC, id DESC
const result = await keysetPaginate<Post>({
  table: "posts",
  fields: [
    { column: "created_at", direction: "DESC", type: "date" },
    { column: "id", direction: "DESC", type: "number" },
  ],
  limit: 20,
  cursor: req.query.cursor as string,
  where: "deleted_at IS NULL AND user_id = $1",
  params: [userId],
});

res.json({
  data: result.rows,
  pageInfo: {
    hasNextPage: result.hasMore,
    endCursor: result.nextCursor,
  },
});
```

## Relay-style Pagination (GraphQL Standard)

### Connection Pattern

```typescript
// ตาม Relay Cursor Connections Specification
// https://relay.dev/graphql/connections.htm

interface Edge<T> {
  node: T;
  cursor: string;
}

interface PageInfo {
  hasNextPage: boolean;
  hasPreviousPage: boolean;
  startCursor: string | null;
  endCursor: string | null;
}

interface Connection<T> {
  edges: Edge<T>[];
  pageInfo: PageInfo;
  totalCount?: number;   // optional
}

// GraphQL Schema:
const typeDefs = gql`
  type PostEdge {
    node: Post!
    cursor: String!
  }
  
  type PostConnection {
    edges: [PostEdge!]!
    pageInfo: PageInfo!
    totalCount: Int
  }
  
  type PageInfo {
    hasNextPage: Boolean!
    hasPreviousPage: Boolean!
    startCursor: String
    endCursor: String
  }
  
  type Query {
    posts(
      first: Int
      after: String
      last: Int
      before: String
    ): PostConnection!
  }
`;

// Resolver:
const resolvers = {
  Query: {
    posts: async (_: any, args: { first?: number; after?: string; last?: number; before?: string }) => {
      const { first, after, last, before } = args;
      
      // Validate: ต้องมี first หรือ last
      if (!first && !last) {
        throw new Error("Must provide `first` or `last`");
      }
      
      const limit = first || last || 20;
      const result = await getPosts({ after, before, limit });
      
      return {
        edges: result.data.map(post => ({
          node: post,
          cursor: encodeCursor({ id: post.id }),
        })),
        pageInfo: result.pageInfo,
      };
    },
  },
};
```

### GraphQL Query ตัวอย่าง

```graphql
# First page
query GetPosts {
  posts(first: 10) {
    edges {
      node {
        id
        title
        createdAt
      }
      cursor
    }
    pageInfo {
      hasNextPage
      endCursor
    }
  }
}

# Next page (using endCursor)
query GetMorePosts($cursor: String!) {
  posts(first: 10, after: $cursor) {
    edges {
      node {
        id
        title
        createdAt
      }
      cursor
    }
    pageInfo {
      hasNextPage
      endCursor
    }
  }
}
```

## Counting Strategies

### Exact Count vs Estimated Count

```typescript
// src/utils/count.ts

// 1. Exact count (accurate แต่ช้า)
export async function exactCount(
  tableName: string,
  whereClause: string = "1=1",
  params: unknown[] = []
): Promise<number> {
  const result = await pool.query(
    `SELECT COUNT(*) as count FROM ${tableName} WHERE ${whereClause}`,
    params
  );
  return parseInt(result.rows[0].count);
}

// 2. Fast estimate จาก pg_class (ไม่แม่นยำ)
export async function estimatedCount(tableName: string): Promise<number> {
  const result = await pool.query(
    `SELECT reltuples::bigint AS estimate FROM pg_class WHERE relname = $1`,
    [tableName]
  );
  return parseInt(result.rows[0].estimate || "0");
}

// 3. Smart count: ถ้าน้อยกว่า threshold ใช้ exact, มากกว่าใช้ estimate
export async function smartCount(
  tableName: string,
  whereClause: string = "1=1",
  params: unknown[] = [],
  threshold: number = 100_000
): Promise<{ count: number; isExact: boolean }> {
  // ลองดูก่อนว่ามีมากกว่า threshold ไหม
  const probeResult = await pool.query(
    `SELECT COUNT(*) as count FROM ${tableName} WHERE ${whereClause} LIMIT ${threshold + 1}`,
    params
  );
  
  const probeCount = parseInt(probeResult.rows[0].count);
  
  if (probeCount <= threshold) {
    return { count: probeCount, isExact: true };
  }
  
  // ใช้ estimate
  const estimate = await estimatedCount(tableName);
  return { count: estimate, isExact: false };
}

// 4. Cached count ด้วย Redis
import { createClient } from "redis";

const redis = createClient({ url: process.env.REDIS_URL });

export async function cachedCount(
  cacheKey: string,
  countFn: () => Promise<number>,
  ttl: number = 300 // 5 นาที
): Promise<number> {
  const cached = await redis.get(cacheKey);
  if (cached) return parseInt(cached);
  
  const count = await countFn();
  await redis.setEx(cacheKey, ttl, String(count));
  
  return count;
}

// ตัวอย่าง:
const productCount = await cachedCount(
  "count:products:active",
  () => exactCount("products", "status = 'active'"),
  60 // cache 1 นาที
);
```

## Filtering + Pagination รวมกัน

```typescript
// src/services/product.service.ts

interface ProductSearchParams {
  // Pagination
  page?: number;
  pageSize?: number;
  cursor?: string;
  
  // Filters
  category?: string | string[];
  minPrice?: number;
  maxPrice?: number;
  inStock?: boolean;
  tags?: string[];
  
  // Sort
  sortBy?: "price" | "name" | "createdAt" | "rating";
  sortOrder?: "asc" | "desc";
  
  // Search
  q?: string;
}

interface SearchResult<T> {
  data: T[];
  pagination?: {           // สำหรับ offset pagination
    page: number;
    pageSize: number;
    total: number;
    totalPages: number;
    hasNextPage: boolean;
    hasPreviousPage: boolean;
  };
  pageInfo?: {             // สำหรับ cursor pagination
    hasNextPage: boolean;
    endCursor: string | null;
  };
}

async function searchProducts(params: ProductSearchParams): Promise<SearchResult<Product>> {
  const {
    page = 1,
    pageSize = 20,
    cursor,
    category,
    minPrice,
    maxPrice,
    inStock,
    tags,
    sortBy = "createdAt",
    sortOrder = "desc",
    q,
  } = params;
  
  // Build Prisma where
  const where: Prisma.ProductWhereInput = {
    deletedAt: null,
  };
  
  if (category) {
    where.category = Array.isArray(category)
      ? { slug: { in: category } }
      : { slug: category };
  }
  
  if (minPrice !== undefined || maxPrice !== undefined) {
    where.price = {};
    if (minPrice !== undefined) (where.price as any).gte = minPrice;
    if (maxPrice !== undefined) (where.price as any).lte = maxPrice;
  }
  
  if (inStock !== undefined) {
    where.stock = inStock ? { gt: 0 } : { equals: 0 };
  }
  
  if (tags?.length) {
    where.tags = { hasSome: tags };
  }
  
  if (q) {
    where.OR = [
      { name: { contains: q, mode: "insensitive" } },
      { description: { contains: q, mode: "insensitive" } },
      { sku: { equals: q, mode: "insensitive" } },
    ];
  }
  
  // Build orderBy
  const allowedSortFields = {
    price: { price: sortOrder },
    name: { name: sortOrder },
    createdAt: { createdAt: sortOrder },
    rating: { averageRating: sortOrder },
  };
  
  const orderBy = allowedSortFields[sortBy] || { createdAt: "desc" };
  
  // ถ้ามี cursor → ใช้ cursor pagination
  if (cursor) {
    const cursorData = decodeCursor<{ id: string }>(cursor);
    
    const [data, hasMore] = await Promise.all([
      db.product.findMany({
        where,
        take: pageSize + 1,
        cursor: { id: cursorData.id },
        skip: 1, // skip cursor item itself
        orderBy,
        include: { category: true },
      }),
      // No total count needed for cursor pagination
    ]);
    
    const hasNextPage = data.length > pageSize;
    if (hasNextPage) data.pop();
    
    return {
      data,
      pageInfo: {
        hasNextPage,
        endCursor: data.length > 0
          ? encodeCursor({ id: data[data.length - 1].id })
          : null,
      },
    };
  }
  
  // ไม่มี cursor → ใช้ offset pagination
  const offset = (page - 1) * pageSize;
  
  const [data, total] = await Promise.all([
    db.product.findMany({
      where,
      skip: offset,
      take: pageSize,
      orderBy,
      include: { category: true },
    }),
    db.product.count({ where }),
  ]);
  
  const totalPages = Math.ceil(total / pageSize);
  
  return {
    data,
    pagination: {
      page,
      pageSize,
      total,
      totalPages,
      hasNextPage: page < totalPages,
      hasPreviousPage: page > 1,
    },
  };
}
```

## Stable Sort สำหรับ Pagination

```sql
-- ปัญหา: Sort field ที่ไม่ unique ทำให้ pagination ไม่ stable

-- ✗ Unstable (created_at อาจซ้ำกัน)
SELECT * FROM posts ORDER BY created_at DESC LIMIT 20;

-- ✅ Stable (เพิ่ม id เป็น tiebreaker)
SELECT * FROM posts ORDER BY created_at DESC, id DESC LIMIT 20;

-- ✅ หรือ UUID tiebreaker
SELECT * FROM posts ORDER BY price ASC, id ASC LIMIT 20;
```

```typescript
// TypeScript: Safe sort utility
const ALLOWED_SORT_FIELDS = {
  price: ["price", "id"],      // primary + tiebreaker
  name: ["name", "id"],
  createdAt: ["createdAt", "id"],
  rating: ["averageRating", "id"],
} as const;

function buildSafeOrderBy(
  sortBy: keyof typeof ALLOWED_SORT_FIELDS,
  sortOrder: "asc" | "desc"
): Prisma.ProductOrderByWithRelationInput[] {
  const [primary, tiebreaker] = ALLOWED_SORT_FIELDS[sortBy];
  
  return [
    { [primary]: sortOrder },
    { [tiebreaker]: sortOrder },  // tiebreaker ใช้ direction เดียวกัน
  ];
}
```

## Frontend: React Infinite Scroll

```tsx
// hooks/useInfiniteProducts.ts
import { useState, useEffect, useRef, useCallback } from "react";

interface Product {
  id: string;
  name: string;
  price: number;
  imageUrl: string;
}

interface PageInfo {
  hasNextPage: boolean;
  endCursor: string | null;
}

export function useInfiniteProducts(filters: Record<string, string>) {
  const [products, setProducts] = useState<Product[]>([]);
  const [pageInfo, setPageInfo] = useState<PageInfo>({
    hasNextPage: true,
    endCursor: null,
  });
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<Error | null>(null);
  const loadingRef = useRef(false);
  
  const loadMore = useCallback(async () => {
    if (loadingRef.current || !pageInfo.hasNextPage) return;
    
    loadingRef.current = true;
    setLoading(true);
    
    try {
      const params = new URLSearchParams({
        ...filters,
        pageSize: "20",
        ...(pageInfo.endCursor ? { cursor: pageInfo.endCursor } : {}),
      });
      
      const response = await fetch(`/api/products?${params}`);
      if (!response.ok) throw new Error("Failed to fetch");
      
      const data = await response.json();
      
      setProducts(prev =>
        pageInfo.endCursor
          ? [...prev, ...data.data]    // append
          : data.data                  // replace (first load)
      );
      setPageInfo(data.pageInfo);
    } catch (err) {
      setError(err as Error);
    } finally {
      setLoading(false);
      loadingRef.current = false;
    }
  }, [filters, pageInfo]);
  
  // Reset เมื่อ filters เปลี่ยน
  useEffect(() => {
    setProducts([]);
    setPageInfo({ hasNextPage: true, endCursor: null });
  }, [JSON.stringify(filters)]);
  
  // Load เมื่อ pageInfo reset
  useEffect(() => {
    if (!pageInfo.endCursor && pageInfo.hasNextPage) {
      loadMore();
    }
  }, [pageInfo.endCursor]);
  
  return { products, pageInfo, loading, error, loadMore };
}

// Components/ProductList.tsx
import { useInfiniteProducts } from "../hooks/useInfiniteProducts";
import { useIntersectionObserver } from "../hooks/useIntersectionObserver";

export function ProductList({ filters }: { filters: Record<string, string> }) {
  const { products, pageInfo, loading, loadMore } = useInfiniteProducts(filters);
  const sentinelRef = useRef<HTMLDivElement>(null);
  
  // Auto-load เมื่อ scroll ถึง sentinel element
  useIntersectionObserver(sentinelRef, (isIntersecting) => {
    if (isIntersecting && pageInfo.hasNextPage && !loading) {
      loadMore();
    }
  });
  
  return (
    <div>
      <div className="product-grid">
        {products.map(product => (
          <ProductCard key={product.id} product={product} />
        ))}
      </div>
      
      {/* Sentinel element สำหรับ Intersection Observer */}
      <div ref={sentinelRef} className="sentinel">
        {loading && <Spinner />}
        {!pageInfo.hasNextPage && <p>แสดงครบทุกรายการแล้ว</p>}
      </div>
    </div>
  );
}

// hooks/useIntersectionObserver.ts
export function useIntersectionObserver(
  ref: React.RefObject<Element>,
  callback: (isIntersecting: boolean) => void
) {
  useEffect(() => {
    const element = ref.current;
    if (!element) return;
    
    const observer = new IntersectionObserver(
      ([entry]) => callback(entry.isIntersecting),
      { rootMargin: "200px" } // โหลดก่อน 200px ล่วงหน้า
    );
    
    observer.observe(element);
    return () => observer.disconnect();
  }, [ref, callback]);
}
```

## PostgreSQL Index สำหรับ Pagination

```sql
-- Offset pagination: index on sort column
CREATE INDEX idx_products_created_at ON products(created_at DESC);

-- Composite index สำหรับ keyset pagination
CREATE INDEX idx_products_sort ON products(created_at DESC, id DESC);

-- Partial index สำหรับ filtered pagination
CREATE INDEX idx_active_products_created_at
ON products(created_at DESC, id DESC)
WHERE deleted_at IS NULL AND status = 'active';

-- Index สำหรับ filter + sort
CREATE INDEX idx_products_category_price
ON products(category_id, price ASC, id ASC)
WHERE deleted_at IS NULL;

-- Covering index (ลด heap fetch)
CREATE INDEX idx_products_listing
ON products(created_at DESC, id DESC)
INCLUDE (name, price, thumbnail_url, category_id);
```

## สรุป: เลือก Pagination Strategy อย่างไร?

```
Use Case                           → Strategy
──────────────────────────────────────────────────────
Admin panel, fixed page numbers   → Offset pagination
Infinite scroll                   → Cursor pagination
Real-time feed (Twitter/Instagram) → Cursor pagination  
Search results                    → Offset (ผู้ใช้ต้องการ page numbers)
GraphQL API                       → Relay cursor (Connection spec)
Large dataset > 100K rows         → Cursor (performance)
Small dataset < 10K rows          → Offset (ง่ายกว่า)
```

```
Performance Comparison (1M rows, page 50000):
─────────────────────────────────────────────
Offset LIMIT 20 OFFSET 1000000:  ~3000ms
Cursor WHERE id < 1000001:          ~5ms
Improvement: 600x faster!
```

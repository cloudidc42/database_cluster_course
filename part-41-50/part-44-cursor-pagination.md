# Part 44: Cursor-based Pagination แบบละเอียด

## ทำไม Cursor Pagination ถึงดีกว่า OFFSET?

### ปัญหาที่แท้จริงของ OFFSET

ลองดู EXPLAIN ANALYZE จริงๆ:

```sql
-- สร้าง test table
CREATE TABLE posts (
  id         BIGSERIAL PRIMARY KEY,
  user_id    UUID NOT NULL,
  title      TEXT NOT NULL,
  content    TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Insert 1 ล้าน rows
INSERT INTO posts (user_id, title, content, created_at)
SELECT
  gen_random_uuid(),
  'Post ' || i,
  repeat('Lorem ipsum ', 50),
  NOW() - (random() * interval '365 days')
FROM generate_series(1, 1000000) AS i;

-- สร้าง index
CREATE INDEX idx_posts_created_at ON posts(created_at DESC, id DESC);

-- VACUUM เพื่อ update statistics
VACUUM ANALYZE posts;
```

```sql
-- ทดสอบ OFFSET pagination
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, title, created_at
FROM posts
ORDER BY created_at DESC, id DESC
LIMIT 20 OFFSET 0;

-- Result:
-- Limit  (cost=0.56..1.88 rows=20 width=48) (actual time=0.060..0.102 rows=20)
-- Execution Time: 0.120 ms  ✅ เร็ว

EXPLAIN (ANALYZE, BUFFERS)
SELECT id, title, created_at
FROM posts
ORDER BY created_at DESC, id DESC
LIMIT 20 OFFSET 100000;

-- Result:
-- Limit  (cost=4548.01..4549.33 rows=20 width=48) (actual time=28.143..28.157 rows=20)
-- Execution Time: 28.182 ms  ⚠️ ช้าลง

EXPLAIN (ANALYZE, BUFFERS)
SELECT id, title, created_at
FROM posts
ORDER BY created_at DESC, id DESC
LIMIT 20 OFFSET 900000;

-- Result:
-- Limit  (cost=40932.09..40933.41 rows=20 width=48) (actual time=285.671..285.685 rows=20)
-- Execution Time: 285.702 ms  ❌ ช้ามาก!

-- OFFSET 900000: ต้องอ่าน index 900,020 entries แล้วข้ามไป 900,000 รายการ
-- ยิ่ง OFFSET สูง ยิ่งช้าแบบ O(n)
```

```sql
-- ทดสอบ Cursor pagination
-- หาค่า cursor จาก row ที่ 100000
SELECT created_at, id
FROM posts
ORDER BY created_at DESC, id DESC
LIMIT 1 OFFSET 99999;
-- ได้: { created_at: '2023-05-15 14:23:11', id: 456789 }

EXPLAIN (ANALYZE, BUFFERS)
SELECT id, title, created_at
FROM posts
WHERE (created_at, id) < ('2023-05-15 14:23:11', 456789)
ORDER BY created_at DESC, id DESC
LIMIT 20;

-- Result:
-- Index Scan using idx_posts_created_at on posts
-- Index Cond: ...
-- Execution Time: 0.089 ms  ✅ เร็วเหมือนหน้าแรกทุกครั้ง!

-- Cursor pagination: O(log n) เสมอ ไม่ว่าจะหน้าที่เท่าไหร่
```

### O(n) vs O(log n): ผลกระทบในชีวิตจริง

```
1 ล้าน rows, page size = 20:

หน้า     OFFSET ms    Cursor ms    ต่างกัน
─────────────────────────────────────────
1        0.12         0.09         1.3x
1,000    1.5          0.09         17x
10,000   15           0.09         170x
50,000   85           0.09         940x
100,000  285          0.09         3,000x+
```

## Simple Cursor: Last ID

```typescript
// src/repositories/post.repository.ts
import { Pool } from "pg";

const pool = new Pool({ connectionString: process.env.DATABASE_URL });

interface Post {
  id: number;
  userId: string;
  title: string;
  content: string;
  createdAt: Date;
}

interface SimpleCursorResult {
  posts: Post[];
  nextCursor: string | null;
  hasMore: boolean;
}

// วิธีง่ายสุด: cursor = last ID
export async function getPostsById(
  afterId?: number,
  limit: number = 20
): Promise<SimpleCursorResult> {
  // Fetch limit+1 เพื่อ detect hasMore
  const result = await pool.query<Post>(
    `SELECT id, user_id, title, content, created_at
     FROM posts
     WHERE ($1::bigint IS NULL OR id < $1)
       AND deleted_at IS NULL
     ORDER BY id DESC
     LIMIT $2`,
    [afterId ?? null, limit + 1]
  );
  
  const posts = result.rows;
  const hasMore = posts.length > limit;
  
  if (hasMore) posts.pop(); // ลบ extra row
  
  const nextCursor = hasMore && posts.length > 0
    ? encodeCursor({ id: posts[posts.length - 1].id })
    : null;
  
  return { posts, nextCursor, hasMore };
}
```

### ปัญหาของ ID-only cursor

```
1. ถ้า sort field ไม่ใช่ id (เช่น sort by created_at) → cursor ต้องเปลี่ยน
2. ถ้า soft delete และ id มีช่องว่าง → ยังใช้ได้ แต่ต้องระวัง
3. ถ้า UUID primary key → ลำดับ UUID ≠ ลำดับ insert time

ดังนั้น ID-only cursor ใช้ได้ดีเฉพาะเมื่อ:
- Sort by id
- Id เป็น auto-increment (sequential)
```

## Composite Cursor สำหรับ Multi-column Sort

```sql
-- Schema สำหรับ post feed
CREATE TABLE posts (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  author_id   UUID NOT NULL,
  title       TEXT NOT NULL,
  content     TEXT,
  likes_count INTEGER NOT NULL DEFAULT 0,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  published_at TIMESTAMPTZ,
  deleted_at  TIMESTAMPTZ
);

-- Index สำหรับ sorting
CREATE INDEX idx_posts_created ON posts(created_at DESC, id DESC)
WHERE deleted_at IS NULL;

CREATE INDEX idx_posts_popular ON posts(likes_count DESC, id DESC)
WHERE deleted_at IS NULL;

CREATE INDEX idx_posts_published ON posts(published_at DESC NULLS LAST, id DESC)
WHERE deleted_at IS NULL;
```

```sql
-- Cursor สำหรับ sort by created_at (DESC) + id (DESC)

-- Page 1:
SELECT id, title, created_at, likes_count
FROM posts
WHERE deleted_at IS NULL
ORDER BY created_at DESC, id DESC
LIMIT 21;  -- +1 extra

-- สมมติ row สุดท้าย: { created_at: '2024-01-15T10:30:00Z', id: 'abc-123' }
-- cursor = encode({ c: '2024-01-15T10:30:00Z', i: 'abc-123' })

-- Page 2:
SELECT id, title, created_at, likes_count
FROM posts
WHERE deleted_at IS NULL
  AND (
    created_at < '2024-01-15T10:30:00Z'
    OR (created_at = '2024-01-15T10:30:00Z' AND id < 'abc-123')
  )
ORDER BY created_at DESC, id DESC
LIMIT 21;

-- การใช้ tuple comparison (PostgreSQL-specific):
SELECT id, title, created_at, likes_count
FROM posts
WHERE deleted_at IS NULL
  AND (created_at, id::text) < ('2024-01-15T10:30:00Z', 'abc-123')
ORDER BY created_at DESC, id DESC
LIMIT 21;
```

```typescript
// TypeScript: Composite cursor implementation

interface PostCursor {
  c: string;    // created_at (ISO string)
  i: string;    // id (UUID string)
}

interface CursorPaginationOptions {
  after?: string;     // opaque cursor (next page)
  before?: string;    // opaque cursor (previous page)
  limit?: number;
}

interface PostsConnection {
  edges: Array<{
    node: Post;
    cursor: string;
  }>;
  pageInfo: {
    hasNextPage: boolean;
    hasPreviousPage: boolean;
    startCursor: string | null;
    endCursor: string | null;
  };
}

function encodeCursor(data: object): string {
  return Buffer.from(JSON.stringify(data)).toString("base64url");
}

function decodeCursor<T>(cursor: string): T {
  try {
    return JSON.parse(Buffer.from(cursor, "base64url").toString("utf8"));
  } catch {
    throw new Error(`Invalid cursor: ${cursor}`);
  }
}

export async function getPostsWithCursor(
  options: CursorPaginationOptions
): Promise<PostsConnection> {
  const { after, before, limit = 20 } = options;
  const take = limit + 1;
  
  let whereCondition: string;
  let params: unknown[];
  let isBackward = false;
  
  if (after) {
    const cursor = decodeCursor<PostCursor>(after);
    whereCondition = `
      AND (
        p.created_at < $1::timestamptz
        OR (p.created_at = $1::timestamptz AND p.id::text < $2)
      )
    `;
    params = [cursor.c, cursor.i, take];
  } else if (before) {
    const cursor = decodeCursor<PostCursor>(before);
    whereCondition = `
      AND (
        p.created_at > $1::timestamptz
        OR (p.created_at = $1::timestamptz AND p.id::text > $2)
      )
    `;
    params = [cursor.c, cursor.i, take];
    isBackward = true;
  } else {
    whereCondition = "";
    params = [take];
  }
  
  // Adjust param index ถ้าไม่มี cursor
  const limitParam = after || before ? "$3" : "$1";
  
  const orderDirection = isBackward ? "ASC" : "DESC";
  
  const query = `
    SELECT
      p.id,
      p.author_id,
      p.title,
      p.content,
      p.likes_count,
      p.created_at,
      u.name AS author_name
    FROM posts p
    JOIN users u ON u.id = p.author_id
    WHERE p.deleted_at IS NULL
      ${whereCondition}
    ORDER BY p.created_at ${orderDirection}, p.id ${orderDirection}
    LIMIT ${limitParam}
  `;
  
  const result = await pool.query(query, params);
  let rows = result.rows;
  
  // กลับทิศทางถ้าเป็น backward pagination
  if (isBackward) {
    rows.reverse();
  }
  
  // Detect hasNextPage/hasPreviousPage
  let hasNextPage = false;
  let hasPreviousPage = false;
  
  if (isBackward) {
    hasPreviousPage = rows.length > limit;
    if (hasPreviousPage) rows.shift();
    hasNextPage = !!after;
  } else {
    hasNextPage = rows.length > limit;
    if (hasNextPage) rows.pop();
    hasPreviousPage = !!after;
  }
  
  // Build edges
  const edges = rows.map(row => ({
    node: row as Post,
    cursor: encodeCursor({
      c: row.created_at.toISOString(),
      i: row.id,
    } as PostCursor),
  }));
  
  return {
    edges,
    pageInfo: {
      hasNextPage,
      hasPreviousPage,
      startCursor: edges.length > 0 ? edges[0].cursor : null,
      endCursor: edges.length > 0 ? edges[edges.length - 1].cursor : null,
    },
  };
}
```

## Bidirectional Pagination: Next + Previous

```typescript
// การ implement bidirectional pagination ที่ถูกต้อง

// Forward (after):
// WHERE (created_at, id) < (cursor_date, cursor_id)  ← เพราะ sort DESC
// ORDER BY created_at DESC, id DESC

// Backward (before):
// WHERE (created_at, id) > (cursor_date, cursor_id)  ← reverse condition
// ORDER BY created_at ASC, id ASC                    ← reverse order
// แล้ว reverse result กลับ

// ตัวอย่าง:
// Data: [10, 9, 8, 7, 6, 5, 4, 3, 2, 1]  (sort DESC)
// Page 1 (first 3): [10, 9, 8]  endCursor=8
// Page 2 after=8: [7, 6, 5]     startCursor=7, endCursor=5
// Page 3 after=5: [4, 3, 2]     startCursor=4, endCursor=2
// Go back before=7: [10, 9, 8]  (same as page 1!)

async function getPostsBidirectional(options: {
  after?: string;
  before?: string;
  first?: number;
  last?: number;
}): Promise<PostsConnection> {
  const { after, before, first, last } = options;
  
  // Validate: ต้องมี first หรือ last อย่างใดอย่างหนึ่ง
  if (!first && !last) {
    throw new Error("Provide `first` or `last`");
  }
  if (first && last) {
    throw new Error("Cannot use both `first` and `last`");
  }
  
  const limit = first || last || 20;
  const take = limit + 1;
  
  let whereClause = "WHERE p.deleted_at IS NULL";
  let orderClause: string;
  let cursorParams: unknown[] = [];
  let isBackward = false;
  
  if (first) {
    // Forward pagination
    orderClause = "ORDER BY p.created_at DESC, p.id DESC";
    
    if (after) {
      const cursor = decodeCursor<PostCursor>(after);
      whereClause += ` AND (p.created_at < $1 OR (p.created_at = $1 AND p.id::text < $2))`;
      cursorParams = [cursor.c, cursor.i];
    }
  } else {
    // Backward pagination (last N items before cursor)
    isBackward = true;
    orderClause = "ORDER BY p.created_at ASC, p.id ASC"; // reverse!
    
    if (before) {
      const cursor = decodeCursor<PostCursor>(before);
      whereClause += ` AND (p.created_at > $1 OR (p.created_at = $1 AND p.id::text > $2))`;
      cursorParams = [cursor.c, cursor.i];
    }
  }
  
  const limitParam = `$${cursorParams.length + 1}`;
  
  const query = `
    SELECT p.id, p.title, p.created_at, u.name AS author_name
    FROM posts p
    JOIN users u ON u.id = p.author_id
    ${whereClause}
    ${orderClause}
    LIMIT ${limitParam}
  `;
  
  const result = await pool.query(query, [...cursorParams, take]);
  let rows = result.rows;
  
  // กลับทิศทางสำหรับ backward
  if (isBackward) rows.reverse();
  
  let hasNextPage = false;
  let hasPreviousPage = false;
  
  if (isBackward) {
    // last N before cursor
    hasPreviousPage = rows.length > limit;
    if (hasPreviousPage) rows.shift();
    hasNextPage = !!before;
  } else {
    // first N after cursor
    hasNextPage = rows.length > limit;
    if (hasNextPage) rows.pop();
    hasPreviousPage = !!after;
  }
  
  const edges = rows.map(row => ({
    node: row as Post,
    cursor: encodeCursor({ c: row.created_at.toISOString(), i: row.id }),
  }));
  
  return {
    edges,
    pageInfo: {
      hasNextPage,
      hasPreviousPage,
      startCursor: edges[0]?.cursor ?? null,
      endCursor: edges[edges.length - 1]?.cursor ?? null,
    },
  };
}
```

## Edge Cases ที่ต้องระวัง

### 1. First Page (ไม่มี cursor)

```typescript
// ไม่มี cursor = first page
if (!cursor) {
  // No WHERE condition for cursor
  // หน้าแรกเสมอ
}

// ต้องระวัง: อย่า throw error ถ้า cursor เป็น undefined
function decodeCursorSafe<T>(cursor: string | undefined): T | null {
  if (!cursor) return null;
  try {
    return decodeCursor<T>(cursor);
  } catch {
    throw new Error("Invalid pagination cursor");
  }
}
```

### 2. Last Page Detection

```typescript
// วิธีที่ 1: fetch limit+1 แล้วตรวจสอบ
const rows = await fetchRows(cursor, limit + 1);
const hasNextPage = rows.length > limit;
if (hasNextPage) rows.pop(); // ลบ extra

// วิธีที่ 2: COUNT (ช้ากว่า แต่แม่นยำกว่า)
const [rows, totalAfter] = await Promise.all([
  fetchRows(cursor, limit),
  countRowsAfter(cursor),
]);
const hasNextPage = totalAfter > rows.length;
```

### 3. Empty Results

```typescript
// ต้อง handle กรณี result ว่างเปล่า
const edges: Array<Edge<Post>> = [];

return {
  edges,
  pageInfo: {
    hasNextPage: false,
    hasPreviousPage: !!after, // มี after cursor = มีหน้าก่อนหน้า
    startCursor: null,        // ไม่มี edges
    endCursor: null,
  },
};
```

### 4. Deleted Items

```sql
-- ปัญหา: ถ้า item ที่ cursor ชี้อยู่ถูกลบ

-- ตัวอย่าง:
-- Page 1 ได้ items: [10, 9, 8, 7, 6]  endCursor = item 6
-- User ลบ item 6
-- Page 2 after=item6: WHERE id < 6 → [5, 4, 3, 2, 1]  ✅ ยังทำงานได้!

-- Cursor pagination ทนต่อการลบ!
-- เพราะ cursor ชี้ไปที่ position ใน sort order ไม่ใช่ row specific

-- แต่ถ้า sort by created_at และ item ถูกลบ:
-- cursor = { c: '2024-01-15', i: 'deleted-uuid' }
-- Query: WHERE created_at < '2024-01-15' OR (created_at = '2024-01-15' AND id < 'deleted-uuid')
-- → ยังทำงานได้! เพราะใช้ค่าของ cursor ไม่ใช่ row ที่มีอยู่
```

```typescript
// Handle soft delete + cursor
const whereCondition = `
  p.deleted_at IS NULL
  ${cursorData ? `
    AND (
      p.created_at < $1
      OR (p.created_at = $1 AND p.id::text < $2)
    )
  ` : ""}
`;

// Hard delete: cursor ยังใช้ได้เพราะเราใช้ค่าของ cursor ไม่ใช่ row
// Soft delete: ต้องเพิ่ม deleted_at IS NULL เสมอ
```

### 5. Stable Cursor (items ไม่ขยับเมื่อข้อมูลเพิ่ม)

```
ปัญหา:
Page 1 (first 5):    [ID=10, 9, 8, 7, 6]  cursor=6
มีคนเพิ่ม ID=11, 12
Page 2 after=6:      [ID=5, 4, 3, 2, 1]   ← ถูกต้อง! ID=11, 12 ไม่มารบกวน

Cursor pagination สร้าง "stable window" ของข้อมูล
ไม่ต้องกังวลเรื่อง data insertions ระหว่าง pagination
```

```sql
-- ตัวอย่างที่แสดงความ stable:

-- Data ตอนแรก:
-- ID=10, ID=9, ID=8, ID=7, ID=6, ID=5, ID=4, ID=3, ID=2, ID=1

-- Page 1 (limit=5): SELECT WHERE 1=1 ORDER BY id DESC LIMIT 6
-- → [10, 9, 8, 7, 6]  hasMore=true, cursor=6

-- เพิ่มข้อมูล: ID=11, ID=12

-- Page 2 (after=6): SELECT WHERE id < 6 ORDER BY id DESC LIMIT 6
-- → [5, 4, 3, 2, 1]  hasMore=false
-- ✅ ID=11, 12 ไม่โผล่มา! เพราะ sort DESC และ id < 6
```

## Connection Pattern (Relay Spec) แบบสมบูรณ์

```typescript
// src/types/relay.ts

export interface Node {
  id: string;
}

export interface Edge<T extends Node> {
  node: T;
  cursor: string;
}

export interface PageInfo {
  hasNextPage: boolean;
  hasPreviousPage: boolean;
  startCursor: string | null;
  endCursor: string | null;
}

export interface Connection<T extends Node> {
  edges: Array<Edge<T>>;
  pageInfo: PageInfo;
  totalCount?: number;
}

// src/utils/relay-pagination.ts

export function buildConnection<T extends Node>(
  items: T[],
  options: {
    hasMore: boolean;
    hasPrev: boolean;
    getCursor: (item: T) => object;
    take: number;
    isBackward: boolean;
  }
): Connection<T> {
  const { hasMore, hasPrev, getCursor, take, isBackward } = options;
  
  const edges: Array<Edge<T>> = items.map(item => ({
    node: item,
    cursor: encodeCursor(getCursor(item)),
  }));
  
  return {
    edges,
    pageInfo: {
      hasNextPage: isBackward ? hasPrev : hasMore,
      hasPreviousPage: isBackward ? hasMore : hasPrev,
      startCursor: edges[0]?.cursor ?? null,
      endCursor: edges[edges.length - 1]?.cursor ?? null,
    },
  };
}
```

## Filter + Cursor Combination

```typescript
// src/services/posts-with-filter.service.ts

interface PostFilters {
  authorId?: string;
  categoryId?: string;
  published?: boolean;
  tags?: string[];
  search?: string;
}

interface PostsQueryOptions {
  filters: PostFilters;
  after?: string;
  before?: string;
  limit?: number;
  sortBy?: "createdAt" | "popularity";
  sortOrder?: "asc" | "desc";
}

interface PostCursorData {
  c: string;   // created_at
  i: string;   // id
  v?: number;  // likes_count (สำหรับ popularity sort)
}

export async function getFilteredPostsWithCursor(
  options: PostsQueryOptions
): Promise<PostsConnection> {
  const { filters, after, before, limit = 20, sortBy = "createdAt" } = options;
  const take = limit + 1;
  const isBackward = !!before;
  
  // Build filter conditions
  const filterConditions: string[] = ["p.deleted_at IS NULL"];
  const filterParams: unknown[] = [];
  
  if (filters.authorId) {
    filterConditions.push(`p.author_id = $${filterParams.length + 1}`);
    filterParams.push(filters.authorId);
  }
  
  if (filters.categoryId) {
    filterConditions.push(`p.category_id = $${filterParams.length + 1}`);
    filterParams.push(filters.categoryId);
  }
  
  if (filters.published !== undefined) {
    filterConditions.push(
      filters.published
        ? "p.published_at IS NOT NULL AND p.published_at <= NOW()"
        : "p.published_at IS NULL"
    );
  }
  
  if (filters.tags?.length) {
    filterConditions.push(`p.tags && $${filterParams.length + 1}::text[]`);
    filterParams.push(filters.tags);
  }
  
  if (filters.search) {
    filterConditions.push(
      `to_tsvector('english', p.title || ' ' || COALESCE(p.content, '')) @@ plainto_tsquery('english', $${filterParams.length + 1})`
    );
    filterParams.push(filters.search);
  }
  
  // Build cursor conditions
  let cursorCondition = "";
  const cursorParams: unknown[] = [];
  
  const cursor = after
    ? decodeCursor<PostCursorData>(after)
    : before
    ? decodeCursor<PostCursorData>(before)
    : null;
  
  if (cursor) {
    const baseIdx = filterParams.length;
    
    if (sortBy === "createdAt") {
      const op = isBackward ? ">" : "<";
      cursorCondition = `
        AND (
          p.created_at ${op} $${baseIdx + 1}::timestamptz
          OR (p.created_at = $${baseIdx + 1}::timestamptz AND p.id::text ${op} $${baseIdx + 2})
        )
      `;
      cursorParams.push(cursor.c, cursor.i);
    } else if (sortBy === "popularity") {
      const op = isBackward ? ">" : "<";
      cursorCondition = `
        AND (
          p.likes_count ${op} $${baseIdx + 1}
          OR (p.likes_count = $${baseIdx + 1} AND p.id::text ${op} $${baseIdx + 2})
        )
      `;
      cursorParams.push(cursor.v, cursor.i);
    }
  }
  
  // Build ORDER BY
  const direction = isBackward ? "ASC" : "DESC";
  let orderBy: string;
  
  if (sortBy === "createdAt") {
    orderBy = `p.created_at ${direction}, p.id ${direction}`;
  } else {
    orderBy = `p.likes_count ${direction}, p.id ${direction}`;
  }
  
  const allParams = [...filterParams, ...cursorParams];
  const limitParam = `$${allParams.length + 1}`;
  
  const query = `
    SELECT
      p.id,
      p.author_id,
      p.title,
      p.content,
      p.likes_count,
      p.tags,
      p.created_at,
      p.published_at,
      u.name AS author_name,
      u.avatar_url AS author_avatar
    FROM posts p
    JOIN users u ON u.id = p.author_id
    WHERE ${filterConditions.join(" AND ")}
    ${cursorCondition}
    ORDER BY ${orderBy}
    LIMIT ${limitParam}
  `;
  
  const result = await pool.query(query, [...allParams, take]);
  let rows = result.rows;
  
  if (isBackward) rows.reverse();
  
  let hasNextPage = false;
  let hasPreviousPage = false;
  
  if (isBackward) {
    hasPreviousPage = rows.length > limit;
    if (hasPreviousPage) rows.shift();
    hasNextPage = true;
  } else {
    hasNextPage = rows.length > limit;
    if (hasNextPage) rows.pop();
    hasPreviousPage = !!after;
  }
  
  const edges = rows.map(row => {
    const cursorData: PostCursorData = {
      c: row.created_at.toISOString(),
      i: row.id,
      ...(sortBy === "popularity" ? { v: row.likes_count } : {}),
    };
    
    return {
      node: row as Post,
      cursor: encodeCursor(cursorData),
    };
  });
  
  return {
    edges,
    pageInfo: {
      hasNextPage,
      hasPreviousPage,
      startCursor: edges[0]?.cursor ?? null,
      endCursor: edges[edges.length - 1]?.cursor ?? null,
    },
  };
}
```

## TypeScript Generic Cursor Pagination Utility

```typescript
// src/utils/generic-cursor-pagination.ts

type SortDirection = "ASC" | "DESC";
type FieldType = "string" | "number" | "date" | "uuid";

interface SortFieldConfig {
  dbColumn: string;      // ชื่อ column ใน DB
  jsKey: string;         // ชื่อ property ใน JS
  direction: SortDirection;
  type: FieldType;
  nullable?: boolean;
}

interface CursorPaginationConfig<T> {
  table: string;
  alias?: string;
  sortFields: SortFieldConfig[];
  defaultLimit?: number;
  maxLimit?: number;
  extraJoins?: string;
  extraSelect?: string;
  filterConditions?: string;
  filterParams?: unknown[];
  mapRow?: (row: any) => T;
}

class GenericCursorPaginator<T> {
  constructor(private config: CursorPaginationConfig<T>) {}
  
  async paginate(options: {
    after?: string;
    before?: string;
    limit?: number;
    whereExtra?: string;
    paramsExtra?: unknown[];
  }): Promise<Connection<T & Node>> {
    const { after, before, limit: reqLimit } = options;
    const {
      table,
      alias = "t",
      sortFields,
      defaultLimit = 20,
      maxLimit = 100,
      extraJoins = "",
      extraSelect = "",
      filterConditions = "1=1",
      filterParams = [],
      mapRow = (r: any) => r as T,
    } = this.config;
    
    const limit = Math.min(reqLimit || defaultLimit, maxLimit);
    const take = limit + 1;
    const isBackward = !!before;
    
    const cursorValue = after
      ? decodeCursor<Record<string, unknown>>(after)
      : before
      ? decodeCursor<Record<string, unknown>>(before)
      : null;
    
    // Build cursor condition
    const { condition: cursorCondition, params: cursorParams } =
      cursorValue
        ? this.buildCursorCondition(sortFields, cursorValue, filterParams.length, isBackward)
        : { condition: "", params: [] };
    
    // Build ORDER BY
    const orderBy = sortFields
      .map(f => `${alias}.${f.dbColumn} ${isBackward ? this.reverseDir(f.direction) : f.direction}`)
      .join(", ");
    
    const allParams = [...filterParams, ...cursorParams];
    const limitIdx = allParams.length + 1;
    
    const query = `
      SELECT ${alias}.*${extraSelect ? `, ${extraSelect}` : ""}
      FROM ${table} ${alias}
      ${extraJoins}
      WHERE (${filterConditions})
        ${cursorCondition ? `AND (${cursorCondition})` : ""}
      ORDER BY ${orderBy}
      LIMIT $${limitIdx}
    `;
    
    const result = await pool.query(query, [...allParams, take]);
    let rows = result.rows;
    
    if (isBackward) rows.reverse();
    
    let hasNextPage = false;
    let hasPreviousPage = false;
    
    if (isBackward) {
      hasPreviousPage = rows.length > limit;
      if (hasPreviousPage) rows.shift();
      hasNextPage = true;
    } else {
      hasNextPage = rows.length > limit;
      if (hasNextPage) rows.pop();
      hasPreviousPage = !!after;
    }
    
    const edges = rows.map(row => {
      const cursorData = Object.fromEntries(
        sortFields.map(f => [f.jsKey, row[f.jsKey] ?? row[f.dbColumn]])
      );
      
      return {
        node: mapRow(row) as T & Node,
        cursor: encodeCursor(cursorData),
      };
    });
    
    return {
      edges,
      pageInfo: {
        hasNextPage,
        hasPreviousPage,
        startCursor: edges[0]?.cursor ?? null,
        endCursor: edges[edges.length - 1]?.cursor ?? null,
      },
    };
  }
  
  private buildCursorCondition(
    fields: SortFieldConfig[],
    cursorData: Record<string, unknown>,
    paramOffset: number,
    isBackward: boolean
  ): { condition: string; params: unknown[] } {
    const params: unknown[] = [];
    const conditions: string[] = [];
    
    for (let i = 0; i < fields.length; i++) {
      const eqParts: string[] = [];
      
      for (let j = 0; j < i; j++) {
        const f = fields[j];
        const cast = this.getCast(f.type);
        eqParts.push(`${f.dbColumn} = $${paramOffset + params.length + 1}${cast}`);
        params.push(cursorData[f.jsKey]);
      }
      
      const currentField = fields[i];
      const cast = this.getCast(currentField.type);
      const isDesc = currentField.direction === "DESC";
      const op = isBackward ? (isDesc ? ">" : "<") : (isDesc ? "<" : ">");
      
      eqParts.push(
        `${currentField.dbColumn} ${op} $${paramOffset + params.length + 1}${cast}`
      );
      params.push(cursorData[currentField.jsKey]);
      
      conditions.push(`(${eqParts.join(" AND ")})`);
    }
    
    return { condition: conditions.join(" OR "), params };
  }
  
  private getCast(type: FieldType): string {
    switch (type) {
      case "date": return "::timestamptz";
      case "number": return "::bigint";
      case "uuid": return "::uuid";
      default: return "";
    }
  }
  
  private reverseDir(dir: SortDirection): SortDirection {
    return dir === "ASC" ? "DESC" : "ASC";
  }
}

// ตัวอย่างการใช้งาน:

const postPaginator = new GenericCursorPaginator<Post>({
  table: "posts",
  alias: "p",
  sortFields: [
    { dbColumn: "created_at", jsKey: "createdAt", direction: "DESC", type: "date" },
    { dbColumn: "id", jsKey: "id", direction: "DESC", type: "uuid" },
  ],
  extraJoins: "JOIN users u ON u.id = p.author_id",
  extraSelect: "u.name AS author_name",
  filterConditions: "p.deleted_at IS NULL",
  mapRow: (row) => ({
    id: row.id,
    title: row.title,
    content: row.content,
    createdAt: row.created_at,
    authorName: row.author_name,
  }),
});

// ใช้งาน:
const result = await postPaginator.paginate({
  after: req.query.cursor as string,
  limit: 20,
});
```

## Prisma Cursor Pagination

```typescript
// Prisma รองรับ cursor pagination แบบ built-in

// src/repositories/post.prisma.repository.ts
import { PrismaClient } from "@prisma/client";

const prisma = new PrismaClient();

interface PrismaCursorOptions {
  cursor?: string;
  limit?: number;
  filters?: {
    authorId?: string;
    published?: boolean;
  };
}

export async function getPostsViaPrisma(options: PrismaCursorOptions) {
  const { cursor, limit = 20, filters = {} } = options;
  
  // Decode cursor
  const cursorData = cursor
    ? decodeCursor<{ id: string }>(cursor)
    : null;
  
  const posts = await prisma.post.findMany({
    where: {
      deletedAt: null,
      ...(filters.authorId ? { authorId: filters.authorId } : {}),
      ...(filters.published !== undefined
        ? filters.published
          ? { publishedAt: { not: null, lte: new Date() } }
          : { publishedAt: null }
        : {}),
    },
    
    // Cursor-based pagination ใน Prisma
    ...(cursorData
      ? {
          cursor: { id: cursorData.id },
          skip: 1, // skip cursor item itself
        }
      : {}),
    
    take: limit + 1,
    
    orderBy: [
      { createdAt: "desc" },
      { id: "desc" },
    ],
    
    include: {
      author: {
        select: { id: true, name: true, avatarUrl: true },
      },
      _count: {
        select: { comments: true, likes: true },
      },
    },
  });
  
  const hasNextPage = posts.length > limit;
  if (hasNextPage) posts.pop();
  
  const endCursor = posts.length > 0
    ? encodeCursor({ id: posts[posts.length - 1].id })
    : null;
  
  return {
    posts,
    pageInfo: {
      hasNextPage,
      hasPreviousPage: !!cursor,
      endCursor,
    },
  };
}

// ⚠️ ข้อจำกัดของ Prisma cursor:
// 1. Prisma cursor ใช้ primary key เสมอ (id field)
// 2. ไม่รองรับ composite cursor โดยตรง
// 3. สำหรับ composite sort (created_at + id) ต้องเขียน raw SQL
// 4. Backward pagination ต้องทำเอง

// วิธีแก้: ใช้ raw SQL สำหรับ composite cursor
const postsWithComposite = await prisma.$queryRaw<Post[]>`
  SELECT p.*, u.name AS author_name
  FROM posts p
  JOIN users u ON u.id = p.author_id
  WHERE p.deleted_at IS NULL
    ${cursorData ? Prisma.sql`
      AND (
        p.created_at < ${cursorData.c}::timestamptz
        OR (p.created_at = ${cursorData.c}::timestamptz AND p.id::text < ${cursorData.i})
      )
    ` : Prisma.empty}
  ORDER BY p.created_at DESC, p.id DESC
  LIMIT ${limit + 1}
`;
```

## Full Implementation: Posts API กับ Cursor Pagination

```typescript
// src/routes/posts.ts
import { Router } from "express";
import { z } from "zod";
import { getFilteredPostsWithCursor } from "../services/posts-with-filter.service";

const router = Router();

// Validation schema
const PostsQuerySchema = z.object({
  after: z.string().optional(),
  before: z.string().optional(),
  limit: z.coerce.number().min(1).max(100).default(20),
  authorId: z.string().uuid().optional(),
  categoryId: z.string().uuid().optional(),
  published: z.enum(["true", "false"]).transform(v => v === "true").optional(),
  tags: z.string().transform(v => v.split(",").filter(Boolean)).optional(),
  q: z.string().min(2).max(100).optional(),
  sortBy: z.enum(["createdAt", "popularity"]).default("createdAt"),
});

// GET /api/posts
// Query params: after, before, limit, authorId, categoryId, published, tags, q, sortBy
router.get("/", async (req, res) => {
  try {
    const query = PostsQuerySchema.safeParse(req.query);
    
    if (!query.success) {
      return res.status(400).json({
        error: "Invalid query parameters",
        details: query.error.flatten(),
      });
    }
    
    const { after, before, limit, authorId, categoryId, published, tags, q, sortBy } = query.data;
    
    // ไม่อนุญาต after + before พร้อมกัน
    if (after && before) {
      return res.status(400).json({
        error: "Cannot use both 'after' and 'before'",
      });
    }
    
    const result = await getFilteredPostsWithCursor({
      after,
      before,
      limit,
      sortBy,
      filters: {
        authorId,
        categoryId,
        published,
        tags,
        search: q,
      },
    });
    
    // Return response ในรูปแบบ Relay Connection
    res.json({
      edges: result.edges.map(edge => ({
        node: {
          id: edge.node.id,
          title: edge.node.title,
          excerpt: edge.node.content?.slice(0, 200),
          authorName: (edge.node as any).author_name,
          createdAt: edge.node.createdAt,
          likesCount: edge.node.likesCount,
          tags: edge.node.tags,
        },
        cursor: edge.cursor,
      })),
      pageInfo: result.pageInfo,
    });
  } catch (error: any) {
    if (error.message?.includes("Invalid cursor")) {
      return res.status(400).json({ error: "Invalid cursor" });
    }
    
    console.error("Posts query error:", error);
    res.status(500).json({ error: "Internal server error" });
  }
});

export default router;
```

### Frontend Integration

```typescript
// hooks/usePaginatedPosts.ts
import useSWRInfinite from "swr/infinite";

interface PostsPage {
  edges: Array<{
    node: Post;
    cursor: string;
  }>;
  pageInfo: {
    hasNextPage: boolean;
    hasPreviousPage: boolean;
    startCursor: string | null;
    endCursor: string | null;
  };
}

interface UsePostsOptions {
  limit?: number;
  authorId?: string;
  categoryId?: string;
  published?: boolean;
  sortBy?: "createdAt" | "popularity";
  q?: string;
}

export function usePaginatedPosts(options: UsePostsOptions = {}) {
  const { limit = 20, ...filters } = options;
  
  const getKey = (pageIndex: number, previousPage: PostsPage | null) => {
    // ถ้าหน้าก่อนหน้าบอกว่าไม่มีหน้าต่อไป
    if (previousPage && !previousPage.pageInfo.hasNextPage) return null;
    
    const params = new URLSearchParams({
      limit: String(limit),
      ...Object.fromEntries(
        Object.entries(filters)
          .filter(([, v]) => v !== undefined && v !== null && v !== "")
          .map(([k, v]) => [k, String(v)])
      ),
    });
    
    // เพิ่ม cursor ของหน้าก่อนหน้า
    if (previousPage?.pageInfo.endCursor) {
      params.set("after", previousPage.pageInfo.endCursor);
    }
    
    return `/api/posts?${params}`;
  };
  
  const { data, error, size, setSize, isValidating } = useSWRInfinite<PostsPage>(
    getKey,
    (url) => fetch(url).then(r => r.json())
  );
  
  // Flatten pages
  const posts = data?.flatMap(page => page.edges.map(e => e.node)) ?? [];
  const lastPage = data?.[data.length - 1];
  
  return {
    posts,
    error,
    isLoading: !data && !error,
    isLoadingMore: isValidating,
    hasMore: lastPage?.pageInfo.hasNextPage ?? true,
    loadMore: () => setSize(size + 1),
  };
}
```

## Performance: Index Strategy สำหรับ Cursor Pagination

```sql
-- 1. Index ตรงกับ sort fields
-- Sort: created_at DESC, id DESC
CREATE INDEX idx_posts_cursor_timeline
ON posts(created_at DESC, id DESC)
WHERE deleted_at IS NULL;

-- 2. Partial index สำหรับ filter ที่ใช้บ่อย
-- Filter: published=true, sort: created_at DESC
CREATE INDEX idx_posts_published_timeline
ON posts(created_at DESC, id DESC)
WHERE deleted_at IS NULL AND published_at IS NOT NULL AND published_at <= NOW();

-- 3. Filter + sort composite index
-- Filter: author_id, sort: created_at DESC
CREATE INDEX idx_posts_author_timeline
ON posts(author_id, created_at DESC, id DESC)
WHERE deleted_at IS NULL;

-- ตรวจสอบว่า index ถูกใช้
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM posts
WHERE deleted_at IS NULL
  AND author_id = 'some-uuid'
  AND (created_at < '2024-01-15' OR (created_at = '2024-01-15' AND id < 'some-id'))
ORDER BY created_at DESC, id DESC
LIMIT 21;

-- ควรเห็น: Index Scan using idx_posts_author_timeline
-- ไม่ใช่ Seq Scan หรือ Parallel Seq Scan
```

## สรุป: Cursor Pagination Checklist

```
Implementation:
✅ Encode cursor เป็น opaque token (base64url)
✅ Validate cursor format เมื่อ decode
✅ Fetch limit+1 เพื่อ detect hasNextPage
✅ Reverse results สำหรับ backward pagination
✅ ใช้ composite sort (primary + unique tiebreaker)
✅ สร้าง index ที่ตรงกับ sort fields

Security:
✅ Validate cursor ว่าไม่มี SQL injection
✅ ตรวจสอบ type ของ cursor values
✅ Limit max pageSize เพื่อป้องกัน DoS

Performance:
✅ Partial index สำหรับ soft-delete pattern
✅ Filter + sort composite index
✅ EXPLAIN ANALYZE เพื่อยืนยัน index usage

Edge Cases:
✅ Handle undefined/null cursor
✅ Handle empty results
✅ Handle deleted items
✅ Handle concurrent writes
```

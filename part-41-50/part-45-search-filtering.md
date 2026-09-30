# Part 45: Search และ Filtering

## บทนำ: Search ใน Database

Search เป็น feature ที่ user ต้องการมากที่สุด แต่ implement อย่างผิดวิธีทำให้ performance แย่มาก มาเรียนรู้ search strategies ทุกแบบ

```
Search Types (เรียงจากง่ายไปซับซ้อน):
1. LIKE '%term%'        → ช้า, full scan
2. LIKE 'term%'         → เร็ว, ใช้ B-Tree index
3. Full-Text Search     → ดีที่สุดสำหรับ natural language
4. Fuzzy Search         → หาคำที่ใกล้เคียง (typo tolerant)
5. External Search      → Elasticsearch, Meilisearch (เร็วสุด แต่ซับซ้อน)
```

## 1. Simple LIKE Search (ช้า แต่ง่าย)

```sql
-- ✗ LIKE '%term%': ไม่ใช้ index, full scan
-- ทุก row ต้องถูกตรวจสอบ
EXPLAIN ANALYZE
SELECT * FROM products WHERE name LIKE '%laptop%';

-- Result:
-- Seq Scan on products
-- Filter: (name ~~ '%laptop%'::text)
-- Rows Removed by Filter: 950000
-- Execution Time: 2500ms

-- ✅ LIKE 'term%': ใช้ B-Tree index (prefix match)
CREATE INDEX idx_products_name ON products(name varchar_pattern_ops);

EXPLAIN ANALYZE
SELECT * FROM products WHERE name LIKE 'laptop%';
-- Index Scan using idx_products_name
-- Execution Time: 2ms
```

### เมื่อไหร่ LIKE ยังเหมาะสม?

```sql
-- ✅ เหมาะ: prefix search บน indexed column ที่ไม่ใหญ่มาก
-- เช่น product SKU, username prefix
SELECT * FROM products WHERE sku LIKE 'PRD-2024%' LIMIT 20;

-- ✅ เหมาะ: small table < 10,000 rows
SELECT * FROM categories WHERE name ILIKE '%electronics%';

-- ❌ ไม่เหมาะ: large table, non-prefix search
SELECT * FROM products WHERE description LIKE '%gaming laptop%';
```

## 2. Full-Text Search ใน PostgreSQL

### tsvector และ tsquery

```sql
-- tsvector: เอกสารที่ถูก tokenize และ normalize
SELECT to_tsvector('english', 'The quick brown fox jumps over the lazy dog');
-- 'brown':3 'dog':9 'fox':4 'jump':5 'lazi':8 'quick':2
-- สังเกต: 'jumps' → 'jump' (stemming), 'the' ถูกตัดทิ้ง (stop words)

-- tsquery: search query ที่ถูก parse
SELECT to_tsquery('english', 'laptop & gaming');
-- 'laptop' & 'gaming'

SELECT plainto_tsquery('english', 'gaming laptop fast');
-- 'gaming' & 'laptop' & 'fast'

SELECT phraseto_tsquery('english', 'gaming laptop');
-- 'gaming' <-> 'laptop'  (phrase match)

SELECT websearch_to_tsquery('english', 'gaming laptop -slow');
-- 'gaming' & 'laptop' & !'slow'  (! = NOT)

-- Match
SELECT to_tsvector('english', 'I have a gaming laptop') @@ plainto_tsquery('english', 'gaming laptop');
-- true
```

### Setup Full-Text Search

```sql
-- 1. เพิ่ม tsvector column
ALTER TABLE products ADD COLUMN search_vector tsvector;

-- 2. อัปเดต vector จาก content
UPDATE products
SET search_vector = to_tsvector(
  'english',
  COALESCE(name, '') || ' ' ||
  COALESCE(description, '') || ' ' ||
  COALESCE(brand, '') || ' ' ||
  COALESCE(sku, '')
);

-- 3. สร้าง GIN index (เร็วกว่า GiST สำหรับ full-text)
CREATE INDEX idx_products_search ON products USING GIN(search_vector);

-- 4. Trigger เพื่ออัปเดต vector อัตโนมัติ
CREATE OR REPLACE FUNCTION update_products_search_vector()
RETURNS trigger AS $$
BEGIN
  NEW.search_vector = to_tsvector(
    'english',
    COALESCE(NEW.name, '') || ' ' ||
    COALESCE(NEW.description, '') || ' ' ||
    COALESCE(NEW.brand, '') || ' ' ||
    COALESCE(NEW.sku, '')
  );
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trig_products_search_vector
  BEFORE INSERT OR UPDATE ON products
  FOR EACH ROW
  EXECUTE FUNCTION update_products_search_vector();
```

### Full-Text Search Query

```sql
-- Basic search
SELECT id, name, brand, price,
  ts_rank(search_vector, query) AS rank
FROM products,
  plainto_tsquery('english', 'gaming laptop') query
WHERE search_vector @@ query
ORDER BY rank DESC, name
LIMIT 20;

-- Search with ranking
SELECT
  id, name, description,
  ts_rank_cd(search_vector, query, 32) AS rank,  -- cover density ranking
  ts_headline(
    'english',
    name || ' ' || COALESCE(description, ''),
    query,
    'MaxFragments=3, MaxWords=30, MinWords=15'
  ) AS highlight
FROM products,
  websearch_to_tsquery('english', $1) query
WHERE search_vector @@ query
ORDER BY rank DESC
LIMIT 20;
```

### Multi-language Full-Text Search

```sql
-- ภาษาไทยไม่มี built-in dictionary ใน PostgreSQL
-- ต้องใช้ pg_jieba, pg_bigm หรือ external tokenizer

-- วิธีที่ 1: ใช้ simple config (no stemming)
SELECT to_tsvector('simple', 'สินค้าราคาถูก คุณภาพดี');
-- 'คุณภาพดี':3 'ถูก':2 'ราคา':1 'สินค้า':0 (ไม่ stem)

-- วิธีที่ 2: pg_bigm extension (สำหรับภาษาจีน/ญี่ปุ่น/ไทย)
CREATE EXTENSION IF NOT EXISTS pg_bigm;

CREATE INDEX idx_products_name_bigm ON products USING GIN(name gin_bigm_ops);
SELECT * FROM products WHERE name LIKE '%ราคาถูก%'; -- ใช้ bigm index

-- วิธีที่ 3: ส่ง tokenized content จาก application
-- ใน Node.js: tokenize text → join with spaces → store in search_vector
```

### Full-Text Search ใน TypeScript

```typescript
// src/services/product-search.service.ts
import { Pool } from "pg";

const pool = new Pool({ connectionString: process.env.DATABASE_URL });

interface SearchOptions {
  query: string;
  category?: string;
  minPrice?: number;
  maxPrice?: number;
  page?: number;
  pageSize?: number;
  language?: string;
}

interface SearchResult {
  id: string;
  name: string;
  description: string;
  price: number;
  rank: number;
  highlight: string;
}

export async function searchProducts(options: SearchOptions) {
  const {
    query,
    category,
    minPrice,
    maxPrice,
    page = 1,
    pageSize = 20,
    language = "english",
  } = options;
  
  if (!query.trim()) {
    throw new Error("Search query cannot be empty");
  }
  
  const params: unknown[] = [query];
  const conditions: string[] = [
    "p.deleted_at IS NULL",
    `p.search_vector @@ websearch_to_tsquery($1, $2)`,
  ];
  params.push(language); // $2
  
  if (category) {
    params.push(category);
    conditions.push(`c.slug = $${params.length}`);
  }
  
  if (minPrice !== undefined) {
    params.push(minPrice);
    conditions.push(`p.price >= $${params.length}`);
  }
  
  if (maxPrice !== undefined) {
    params.push(maxPrice);
    conditions.push(`p.price <= $${params.length}`);
  }
  
  const offset = (page - 1) * pageSize;
  params.push(pageSize, offset);
  const limitIdx = params.length - 1;
  const offsetIdx = params.length;
  
  const searchSQL = `
    SELECT
      p.id,
      p.name,
      p.price,
      p.thumbnail_url,
      c.name AS category_name,
      ts_rank_cd(
        p.search_vector,
        websearch_to_tsquery($1, $2),
        32
      ) AS rank,
      ts_headline(
        $1,
        p.name || ' ' || COALESCE(p.description, ''),
        websearch_to_tsquery($1, $2),
        'MaxFragments=2, MaxWords=20, MinWords=10, StartSel=<mark>, StopSel=</mark>'
      ) AS highlight
    FROM products p
    LEFT JOIN categories c ON c.id = p.category_id
    WHERE ${conditions.join(" AND ")}
    ORDER BY rank DESC, p.name
    LIMIT $${limitIdx}
    OFFSET $${offsetIdx}
  `;
  
  const countSQL = `
    SELECT COUNT(*) AS total
    FROM products p
    LEFT JOIN categories c ON c.id = p.category_id
    WHERE ${conditions.join(" AND ")}
  `;
  
  const [searchResult, countResult] = await Promise.all([
    pool.query<SearchResult>(searchSQL, params),
    pool.query<{ total: string }>(countSQL, params.slice(0, -2)), // without limit/offset
  ]);
  
  return {
    results: searchResult.rows,
    total: parseInt(countResult.rows[0].total),
    page,
    pageSize,
    totalPages: Math.ceil(parseInt(countResult.rows[0].total) / pageSize),
  };
}
```

## 3. Fuzzy Search ด้วย pg_trgm

### pg_trgm คืออะไร?

```
Trigram: แบ่งข้อความเป็น substring ขนาด 3 ตัวอักษร
"postgresql" → "pos", "ost", "stg", "tgr", "gre", "res", "sql"

ใช้เพื่อ:
1. Fuzzy matching (typo tolerance)
2. Fast LIKE/ILIKE บน large text fields
3. Similarity scoring
```

```sql
-- Enable extension
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- ดู trigrams
SELECT show_trgm('postgresql');
-- {"  p"," po","gre","pos","res","sql","stg","tgr","ogr","osg","sgl","tgl","tgr","ure","..."}

-- Similarity score (0-1)
SELECT similarity('postgresql', 'posrgresql');
-- 0.72 (typo: "pg" ผิดเป็น "rg")

SELECT similarity('postgresql', 'mysql');
-- 0.15 (ต่างกันมาก)

-- Word similarity
SELECT word_similarity('postgresql', 'I use postgresql database');
-- 1.0 (พบ exact word)

-- Strict word similarity
SELECT strict_word_similarity('postgresql', 'I use PostgreSQL');
-- 1.0 (case-insensitive match)
```

### Fuzzy Search Index และ Query

```sql
-- สร้าง GIN index สำหรับ trigram
CREATE INDEX idx_products_name_trgm ON products USING GIN(name gin_trgm_ops);
CREATE INDEX idx_products_desc_trgm ON products USING GIN(description gin_trgm_ops);

-- Fuzzy LIKE (ใช้ trgm index)
-- % operator: similarity > pg_trgm.similarity_threshold (default 0.3)
SET pg_trgm.similarity_threshold = 0.3;

SELECT name, similarity(name, 'labtop') AS sim
FROM products
WHERE name % 'labtop'   -- fuzzy match
ORDER BY sim DESC
LIMIT 10;

-- หรือ ILIKE กับ trgm index (เร็วกว่า LIKE '%term%' มาก)
SELECT * FROM products
WHERE name ILIKE '%laptop%'
ORDER BY name
LIMIT 20;

-- Combine: ถ้า similarity สูง ให้ rank สูง
SELECT
  id,
  name,
  price,
  similarity(name, $1) AS fuzzy_score,
  CASE
    WHEN name ILIKE $2 THEN 2.0          -- exact contains
    WHEN name % $1 THEN 1.0               -- fuzzy match
    ELSE 0.5                               -- description match
  END AS boost
FROM products
WHERE (name ILIKE $2 OR name % $1 OR description % $1)
  AND deleted_at IS NULL
ORDER BY boost DESC, fuzzy_score DESC
LIMIT 20;
-- params: [$1='laptop', $2='%laptop%']
```

### Combined Search: FTS + Fuzzy

```sql
-- ดีที่สุด: ลอง FTS ก่อน ถ้าไม่เจอ ลอง fuzzy
WITH fts_results AS (
  SELECT id, name, price,
    ts_rank_cd(search_vector, query) * 2 AS score,
    'fts' AS match_type
  FROM products,
    websearch_to_tsquery('english', $1) query
  WHERE search_vector @@ query
    AND deleted_at IS NULL
),
fuzzy_results AS (
  SELECT id, name, price,
    similarity(name, $1) AS score,
    'fuzzy' AS match_type
  FROM products
  WHERE name % $1
    AND deleted_at IS NULL
    AND id NOT IN (SELECT id FROM fts_results)  -- ไม่ซ้ำกับ FTS
)
SELECT * FROM fts_results
UNION ALL
SELECT * FROM fuzzy_results
ORDER BY score DESC
LIMIT 20;
```

## 4. Filtering แบบ Type-safe

### Dynamic WHERE Builder

```typescript
// src/utils/query-builder.ts

interface FilterValue {
  eq?: unknown;           // equal
  ne?: unknown;           // not equal
  gt?: number | Date;     // greater than
  gte?: number | Date;    // greater than or equal
  lt?: number | Date;     // less than
  lte?: number | Date;    // less than or equal
  in?: unknown[];         // IN list
  nin?: unknown[];        // NOT IN list
  like?: string;          // LIKE pattern
  ilike?: string;         // ILIKE pattern (case-insensitive)
  isNull?: boolean;       // IS NULL / IS NOT NULL
  between?: [unknown, unknown]; // BETWEEN x AND y
}

type Filters = Record<string, FilterValue | unknown>;

interface QueryBuilderResult {
  where: string;
  params: unknown[];
}

class QueryBuilder {
  private conditions: string[] = [];
  private params: unknown[] = [];
  
  constructor(private baseCondition: string = "1=1") {
    if (baseCondition !== "1=1") {
      this.conditions.push(baseCondition);
    }
  }
  
  addFilter(column: string, filter: FilterValue): this {
    const addParam = (value: unknown): number => {
      this.params.push(value);
      return this.params.length;
    };
    
    if (filter.eq !== undefined) {
      this.conditions.push(`${column} = $${addParam(filter.eq)}`);
    }
    if (filter.ne !== undefined) {
      this.conditions.push(`${column} != $${addParam(filter.ne)}`);
    }
    if (filter.gt !== undefined) {
      this.conditions.push(`${column} > $${addParam(filter.gt)}`);
    }
    if (filter.gte !== undefined) {
      this.conditions.push(`${column} >= $${addParam(filter.gte)}`);
    }
    if (filter.lt !== undefined) {
      this.conditions.push(`${column} < $${addParam(filter.lt)}`);
    }
    if (filter.lte !== undefined) {
      this.conditions.push(`${column} <= $${addParam(filter.lte)}`);
    }
    if (filter.in?.length) {
      const placeholders = filter.in.map(v => `$${addParam(v)}`).join(", ");
      this.conditions.push(`${column} IN (${placeholders})`);
    }
    if (filter.nin?.length) {
      const placeholders = filter.nin.map(v => `$${addParam(v)}`).join(", ");
      this.conditions.push(`${column} NOT IN (${placeholders})`);
    }
    if (filter.like !== undefined) {
      this.conditions.push(`${column} LIKE $${addParam(filter.like)}`);
    }
    if (filter.ilike !== undefined) {
      this.conditions.push(`${column} ILIKE $${addParam(`%${filter.ilike}%`)}`);
    }
    if (filter.isNull !== undefined) {
      this.conditions.push(
        filter.isNull ? `${column} IS NULL` : `${column} IS NOT NULL`
      );
    }
    if (filter.between) {
      const [lo, hi] = filter.between;
      this.conditions.push(
        `${column} BETWEEN $${addParam(lo)} AND $${addParam(hi)}`
      );
    }
    
    return this;
  }
  
  addRaw(condition: string, ...values: unknown[]): this {
    // ใช้สำหรับ complex conditions
    // ต้องใช้ parameterized queries เสมอ ไม่ interpolate values
    const offset = this.params.length;
    const condWithParams = condition.replace(/\?/g, () => `$${++this.params.length}`);
    this.params.push(...values);
    this.conditions.push(condWithParams);
    return this;
  }
  
  build(): QueryBuilderResult {
    return {
      where: this.conditions.length > 0
        ? this.conditions.join(" AND ")
        : "1=1",
      params: this.params,
    };
  }
}

// ตัวอย่างการใช้งาน
function buildProductQuery(filters: {
  minPrice?: number;
  maxPrice?: number;
  category?: string;
  inStock?: boolean;
  tags?: string[];
  search?: string;
  brand?: string;
}): QueryBuilderResult {
  const builder = new QueryBuilder("p.deleted_at IS NULL");
  
  if (filters.minPrice !== undefined || filters.maxPrice !== undefined) {
    builder.addFilter("p.price", {
      gte: filters.minPrice,
      lte: filters.maxPrice,
    });
  }
  
  if (filters.category) {
    builder.addFilter("c.slug", { eq: filters.category });
  }
  
  if (filters.inStock !== undefined) {
    builder.addFilter("p.stock_quantity", filters.inStock ? { gt: 0 } : { eq: 0 });
  }
  
  if (filters.tags?.length) {
    // Array overlap operator
    builder.addRaw("p.tags && ?::text[]", filters.tags);
  }
  
  if (filters.brand) {
    builder.addFilter("p.brand", { ilike: filters.brand });
  }
  
  if (filters.search) {
    builder.addRaw(
      "p.search_vector @@ websearch_to_tsquery('english', ?)",
      filters.search
    );
  }
  
  return builder.build();
}
```

### Query Parameters Parsing

```typescript
// src/utils/filter-parser.ts
import { z } from "zod";

// URL Query: ?price[gte]=100&price[lte]=500&tags[in]=nodejs,typescript&stock[null]=false
// Parse เป็น structured filters

const FilterSchema = z.object({
  // Price range
  "price[gte]": z.coerce.number().optional(),
  "price[lte]": z.coerce.number().optional(),
  "price[gt]": z.coerce.number().optional(),
  "price[lt]": z.coerce.number().optional(),
  
  // Category
  "category[eq]": z.string().optional(),
  "category[in]": z.string().transform(v => v.split(",")).optional(),
  
  // Tags (array)
  "tags[in]": z.string().transform(v => v.split(",").filter(Boolean)).optional(),
  
  // Stock
  "stock[null]": z.enum(["true", "false"]).transform(v => v === "true").optional(),
  
  // Created date range
  "createdAt[gte]": z.string().datetime().optional(),
  "createdAt[lte]": z.string().datetime().optional(),
  
  // Status
  "status[eq]": z.enum(["active", "inactive", "draft"]).optional(),
  "status[in]": z.string().transform(v => v.split(",")).optional(),
  
  // Sort
  sort: z.string().optional(),                            // -createdAt,name
  
  // Search
  q: z.string().min(1).max(200).optional(),
  
  // Pagination
  page: z.coerce.number().min(1).default(1),
  pageSize: z.coerce.number().min(1).max(100).default(20),
});

type ParsedFilters = z.infer<typeof FilterSchema>;

function parseFilters(query: Record<string, string>): ParsedFilters {
  const result = FilterSchema.safeParse(query);
  
  if (!result.success) {
    throw new Error(`Invalid filters: ${JSON.stringify(result.error.flatten())}`);
  }
  
  return result.data;
}

// Sort parsing: "-createdAt,name,+price"
interface SortOrder {
  column: string;
  direction: "ASC" | "DESC";
}

const ALLOWED_SORT_COLUMNS = new Set([
  "name", "price", "createdAt", "updatedAt", "rating", "stock",
]);

function parseSort(sortParam?: string): SortOrder[] {
  if (!sortParam) return [{ column: "createdAt", direction: "DESC" }];
  
  return sortParam
    .split(",")
    .map(s => s.trim())
    .filter(Boolean)
    .map(s => {
      const direction: "ASC" | "DESC" =
        s.startsWith("-") ? "DESC" :
        s.startsWith("+") ? "ASC" : "ASC";
      
      const column = s.replace(/^[+-]/, "");
      
      // Whitelist validation (SQL injection prevention!)
      if (!ALLOWED_SORT_COLUMNS.has(column)) {
        throw new Error(`Invalid sort column: ${column}`);
      }
      
      // Map JS names to DB column names
      const columnMap: Record<string, string> = {
        createdAt: "created_at",
        updatedAt: "updated_at",
        rating: "average_rating",
        stock: "stock_quantity",
      };
      
      return {
        column: columnMap[column] || column,
        direction,
      };
    });
}

// ตัวอย่าง: parse URL query params
function buildProductsQuery(rawQuery: Record<string, string>) {
  const filters = parseFilters(rawQuery);
  const sortOrders = parseSort(filters.sort);
  
  const builder = new QueryBuilder("p.deleted_at IS NULL");
  
  // Apply filters
  if (filters["price[gte]"]) builder.addFilter("p.price", { gte: filters["price[gte]"] });
  if (filters["price[lte]"]) builder.addFilter("p.price", { lte: filters["price[lte]"] });
  if (filters["category[eq]"]) builder.addFilter("c.slug", { eq: filters["category[eq]"] });
  if (filters["category[in]"]) builder.addFilter("c.slug", { in: filters["category[in]"] });
  if (filters["tags[in]"]) builder.addRaw("p.tags && ?::text[]", filters["tags[in]"]);
  if (filters["stock[null]"] !== undefined) {
    builder.addFilter("p.stock_quantity", {
      ...(filters["stock[null]"] ? { isNull: true } : { gt: 0 }),
    });
  }
  if (filters["status[eq]"]) builder.addFilter("p.status", { eq: filters["status[eq]"] });
  if (filters.q) {
    builder.addRaw(
      "p.search_vector @@ websearch_to_tsquery('english', ?)",
      filters.q
    );
  }
  
  const { where, params } = builder.build();
  
  // Build ORDER BY (safe - whitelist validated)
  const orderByClause = sortOrders
    .map(s => `p.${s.column} ${s.direction}`)
    .join(", ");
  
  // Add tiebreaker
  const finalOrderBy = `${orderByClause}, p.id DESC`;
  
  return {
    where,
    params,
    orderBy: finalOrderBy,
    pagination: {
      limit: filters.pageSize,
      offset: (filters.page - 1) * filters.pageSize,
    },
  };
}
```

## 5. Meilisearch Integration

### ทำไม Meilisearch?

```
Meilisearch vs PostgreSQL FTS:
─────────────────────────────────────────────────────────────
Feature          PostgreSQL FTS    Meilisearch
─────────────────────────────────────────────────────────────
Speed            Fast              Ultra-fast (<50ms)
Typo tolerance   No (need trgm)    Yes (built-in)
Faceting         Manual            Built-in
Highlighting     Yes               Yes
Ranking          Basic             Customizable
Thai support     Poor              OK (unicode)
Setup            Easy              Easy
Storage          Integrated        Separate service
Data sync        N/A               Manual
─────────────────────────────────────────────────────────────

ใช้ Meilisearch เมื่อ:
- ต้องการ instant search (<100ms)
- Typo tolerance เป็นสิ่งจำเป็น
- Faceted search (filter by categories, price ranges)
- Full-featured search UI
```

### ติดตั้ง Meilisearch ด้วย Docker

```yaml
# docker-compose.yml
services:
  meilisearch:
    image: getmeili/meilisearch:v1.6
    ports:
      - "7700:7700"
    environment:
      MEILI_MASTER_KEY: "your-master-key-here"
      MEILI_ENV: "production"
      MEILI_DB_PATH: "/meili_data"
    volumes:
      - meilisearch-data:/meili_data
    restart: unless-stopped

volumes:
  meilisearch-data:
```

```bash
# ทดสอบ
curl http://localhost:7700/health
# {"status":"available"}

# สร้าง index
curl -X POST http://localhost:7700/indexes \
  -H "Authorization: Bearer your-master-key-here" \
  -H "Content-Type: application/json" \
  -d '{"uid": "products", "primaryKey": "id"}'
```

### Meilisearch Client Setup

```bash
npm install meilisearch
```

```typescript
// src/config/meilisearch.ts
import { MeiliSearch } from "meilisearch";

export const meiliClient = new MeiliSearch({
  host: process.env.MEILISEARCH_HOST || "http://localhost:7700",
  apiKey: process.env.MEILISEARCH_API_KEY,
});

// Configure product index
export async function configureMeilisearchIndex() {
  const index = meiliClient.index("products");
  
  // Searchable attributes (ลำดับ = ความสำคัญ)
  await index.updateSearchableAttributes([
    "name",           // สำคัญสุด
    "brand",
    "sku",
    "description",
    "tags",
    "categoryName",
  ]);
  
  // Filterable attributes (สำหรับ facets)
  await index.updateFilterableAttributes([
    "categoryId",
    "brand",
    "price",
    "inStock",
    "tags",
    "status",
    "createdAt",
  ]);
  
  // Sortable attributes
  await index.updateSortableAttributes([
    "price",
    "createdAt",
    "likesCount",
    "averageRating",
  ]);
  
  // Ranking rules (ลำดับ = ความสำคัญ)
  await index.updateRankingRules([
    "words",            // match มากคำกว่า
    "typo",             // typo น้อยกว่า
    "proximity",        // คำอยู่ใกล้กัน
    "attribute",        // match ใน attribute ที่สำคัญกว่า
    "sort",             // sort attribute
    "exactness",        // exact match ดีกว่า
    // Custom:
    "likesCount:desc",  // สินค้า popular ขึ้นก่อน (ถ้า relevance เท่ากัน)
  ]);
  
  // Typo tolerance settings
  await index.updateTypoTolerance({
    enabled: true,
    minWordSizeForTypos: {
      oneTypo: 5,    // คำที่มีตัวอักษร ≥5 รับ typo 1 ตัว
      twoTypos: 9,   // คำที่มีตัวอักษร ≥9 รับ typo 2 ตัว
    },
  });
  
  console.log("Meilisearch index configured");
}
```

### Sync ข้อมูลจาก PostgreSQL ไป Meilisearch

```typescript
// src/services/search-sync.service.ts
import { meiliClient } from "../config/meilisearch";
import { db } from "../db";

interface MeiliProduct {
  id: string;
  name: string;
  brand: string | null;
  sku: string;
  description: string | null;
  price: number;
  inStock: boolean;
  tags: string[];
  categoryId: string | null;
  categoryName: string | null;
  status: string;
  likesCount: number;
  averageRating: number;
  thumbnailUrl: string | null;
  createdAt: number; // Unix timestamp (Meili ชอบ number)
  updatedAt: number;
}

export class SearchSyncService {
  private readonly index = meiliClient.index<MeiliProduct>("products");
  
  // Initial full sync
  async fullSync(): Promise<void> {
    console.log("Starting full product sync to Meilisearch...");
    
    const batchSize = 1000;
    let offset = 0;
    let total = 0;
    
    while (true) {
      const products = await db.product.findMany({
        skip: offset,
        take: batchSize,
        where: { deletedAt: null },
        include: { category: { select: { id: true, name: true } } },
      });
      
      if (products.length === 0) break;
      
      const meiliDocs: MeiliProduct[] = products.map(p => ({
        id: p.id,
        name: p.name,
        brand: p.brand,
        sku: p.sku,
        description: p.description,
        price: p.price,
        inStock: p.stockQuantity > 0,
        tags: p.tags,
        categoryId: p.categoryId,
        categoryName: p.category?.name ?? null,
        status: p.status,
        likesCount: p.likesCount,
        averageRating: p.averageRating,
        thumbnailUrl: p.thumbnailUrl,
        createdAt: Math.floor(p.createdAt.getTime() / 1000),
        updatedAt: Math.floor(p.updatedAt.getTime() / 1000),
      }));
      
      // Upsert ไป Meilisearch (max 10,000 docs per batch)
      const task = await this.index.addDocuments(meiliDocs, { primaryKey: "id" });
      await this.waitForTask(task.taskUid);
      
      total += products.length;
      offset += batchSize;
      
      console.log(`Synced ${total} products...`);
    }
    
    console.log(`Full sync complete: ${total} products`);
  }
  
  // Sync single product (เรียกเมื่อ create/update)
  async syncProduct(productId: string): Promise<void> {
    const product = await db.product.findUnique({
      where: { id: productId },
      include: { category: { select: { id: true, name: true } } },
    });
    
    if (!product || product.deletedAt) {
      // ลบออกจาก Meilisearch
      await this.index.deleteDocument(productId);
      return;
    }
    
    const doc: MeiliProduct = {
      id: product.id,
      name: product.name,
      brand: product.brand,
      sku: product.sku,
      description: product.description,
      price: product.price,
      inStock: product.stockQuantity > 0,
      tags: product.tags,
      categoryId: product.categoryId,
      categoryName: product.category?.name ?? null,
      status: product.status,
      likesCount: product.likesCount,
      averageRating: product.averageRating,
      thumbnailUrl: product.thumbnailUrl,
      createdAt: Math.floor(product.createdAt.getTime() / 1000),
      updatedAt: Math.floor(product.updatedAt.getTime() / 1000),
    };
    
    await this.index.addDocuments([doc]);
  }
  
  private async waitForTask(taskUid: number): Promise<void> {
    let task = await meiliClient.getTask(taskUid);
    
    while (task.status !== "succeeded" && task.status !== "failed") {
      await new Promise(resolve => setTimeout(resolve, 500));
      task = await meiliClient.getTask(taskUid);
    }
    
    if (task.status === "failed") {
      throw new Error(`Meilisearch task failed: ${JSON.stringify(task.error)}`);
    }
  }
}

export const searchSyncService = new SearchSyncService();
```

### Meilisearch Search API

```typescript
// src/services/meili-search.service.ts
import { meiliClient } from "../config/meilisearch";

interface MeiliSearchOptions {
  q: string;
  
  // Filters (Meili filter syntax)
  category?: string;
  brand?: string | string[];
  minPrice?: number;
  maxPrice?: number;
  inStock?: boolean;
  tags?: string[];
  
  // Sorting
  sort?: string[];  // ["price:asc", "createdAt:desc"]
  
  // Facets
  facets?: string[];
  
  // Pagination
  page?: number;
  hitsPerPage?: number;
}

export async function searchWithMeilisearch(options: MeiliSearchOptions) {
  const {
    q,
    category,
    brand,
    minPrice,
    maxPrice,
    inStock,
    tags,
    sort = ["_score:desc"],
    facets = ["categoryName", "brand", "tags"],
    page = 1,
    hitsPerPage = 20,
  } = options;
  
  // Build Meilisearch filter string
  const filterParts: string[] = [];
  
  if (category) {
    filterParts.push(`categoryId = "${category}"`);
  }
  
  if (brand) {
    if (Array.isArray(brand)) {
      filterParts.push(`brand IN ["${brand.join('", "')}"]`);
    } else {
      filterParts.push(`brand = "${brand}"`);
    }
  }
  
  if (minPrice !== undefined && maxPrice !== undefined) {
    filterParts.push(`price ${minPrice} TO ${maxPrice}`);
  } else if (minPrice !== undefined) {
    filterParts.push(`price >= ${minPrice}`);
  } else if (maxPrice !== undefined) {
    filterParts.push(`price <= ${maxPrice}`);
  }
  
  if (inStock !== undefined) {
    filterParts.push(`inStock = ${inStock}`);
  }
  
  if (tags?.length) {
    // ต้องการสินค้าที่มีทุก tag ที่ระบุ
    tags.forEach(tag => filterParts.push(`tags = "${tag}"`));
  }
  
  // Always filter active products
  filterParts.push('status = "active"');
  
  const index = meiliClient.index("products");
  
  const result = await index.search(q, {
    filter: filterParts.length > 0 ? filterParts.join(" AND ") : undefined,
    sort: sort.filter(s => s !== "_score:desc"),
    facets,
    page,
    hitsPerPage,
    attributesToHighlight: ["name", "description", "brand"],
    highlightPreTag: "<mark>",
    highlightPostTag: "</mark>",
    attributesToCrop: ["description"],
    cropLength: 100,
    showRankingScore: process.env.NODE_ENV !== "production",
  });
  
  return {
    hits: result.hits,
    totalHits: result.totalHits,
    page: result.page,
    hitsPerPage: result.hitsPerPage,
    totalPages: result.totalPages,
    facetDistribution: result.facetDistribution,
    processingTimeMs: result.processingTimeMs,
  };
}
```

## Full Working Example: Product Search API

```typescript
// src/routes/search.ts
import { Router } from "express";
import { z } from "zod";
import { searchWithMeilisearch } from "../services/meili-search.service";
import { searchProducts } from "../services/product-search.service";

const router = Router();

const SearchQuerySchema = z.object({
  q: z.string().min(1).max(200),
  category: z.string().uuid().optional(),
  brand: z.union([z.string(), z.array(z.string())]).optional(),
  "price[gte]": z.coerce.number().min(0).optional(),
  "price[lte]": z.coerce.number().min(0).optional(),
  inStock: z.enum(["true", "false"]).transform(v => v === "true").optional(),
  tags: z.string().transform(v => v.split(",").filter(Boolean)).optional(),
  sort: z.string().optional(),
  page: z.coerce.number().min(1).default(1),
  pageSize: z.coerce.number().min(1).max(100).default(20),
  engine: z.enum(["postgres", "meilisearch"]).default("meilisearch"),
});

// GET /api/search?q=gaming+laptop&price[gte]=500&price[lte]=2000&inStock=true
router.get("/", async (req, res) => {
  try {
    const parsed = SearchQuerySchema.safeParse(req.query);
    
    if (!parsed.success) {
      return res.status(400).json({
        error: "Invalid search parameters",
        details: parsed.error.flatten(),
      });
    }
    
    const {
      q, category, brand, page, pageSize, engine, sort, tags, inStock,
      "price[gte]": minPrice,
      "price[lte]": maxPrice,
    } = parsed.data;
    
    let results;
    
    if (engine === "meilisearch") {
      // Parse Meili sort format: "-price,createdAt" → ["price:desc", "createdAt:asc"]
      const meiliSort = sort?.split(",").map(s => {
        const dir = s.startsWith("-") ? "desc" : "asc";
        const col = s.replace(/^[+-]/, "");
        return `${col}:${dir}`;
      });
      
      results = await searchWithMeilisearch({
        q,
        category,
        brand: brand ? (Array.isArray(brand) ? brand : brand.split(",")) : undefined,
        minPrice,
        maxPrice,
        inStock,
        tags,
        sort: meiliSort,
        page,
        hitsPerPage: pageSize,
        facets: ["categoryName", "brand", "tags"],
      });
      
      res.json({
        engine: "meilisearch",
        query: q,
        totalHits: results.totalHits,
        processingTimeMs: results.processingTimeMs,
        page: results.page,
        pageSize: results.hitsPerPage,
        totalPages: results.totalPages,
        facets: results.facetDistribution,
        results: results.hits.map(hit => ({
          id: hit.id,
          name: hit._formatted?.name || hit.name,
          description: hit._formatted?.description || hit.description?.slice(0, 150),
          price: hit.price,
          brand: hit.brand,
          thumbnailUrl: hit.thumbnailUrl,
          inStock: hit.inStock,
          tags: hit.tags,
        })),
      });
    } else {
      // PostgreSQL full-text search
      results = await searchProducts({
        query: q,
        category,
        minPrice,
        maxPrice,
        page,
        pageSize,
      });
      
      res.json({
        engine: "postgresql",
        query: q,
        total: results.total,
        page: results.page,
        pageSize: results.pageSize,
        totalPages: results.totalPages,
        results: results.results,
      });
    }
  } catch (error: any) {
    console.error("Search error:", error);
    res.status(500).json({ error: "Search failed", message: error.message });
  }
});

// GET /api/search/suggestions?q=lap (autocomplete)
router.get("/suggestions", async (req, res) => {
  const q = String(req.query.q || "").trim();
  
  if (q.length < 2) {
    return res.json({ suggestions: [] });
  }
  
  const index = meiliClient.index("products");
  
  const result = await index.search(q, {
    limit: 5,
    attributesToRetrieve: ["name", "brand", "id"],
    attributesToHighlight: ["name"],
    highlightPreTag: "<b>",
    highlightPostTag: "</b>",
  });
  
  res.json({
    suggestions: result.hits.map(hit => ({
      id: hit.id,
      name: hit._formatted?.name || hit.name,
      brand: hit.brand,
    })),
  });
});

export default router;
```

## Null Handling ใน Sorting

```sql
-- NULLS FIRST / NULLS LAST

-- Sort ascending, nulls ไปสุดท้าย
SELECT * FROM products
ORDER BY price ASC NULLS LAST;

-- Sort descending, nulls ไปสุดท้าย (default behavior ใน DESC)
SELECT * FROM products
ORDER BY published_at DESC NULLS LAST;

-- สำหรับ pagination cursor กับ nullable fields ต้องระวัง:
-- ถ้า published_at เป็น NULL ค่า cursor จะเป็นอะไร?

-- วิธีแก้: ใช้ COALESCE
SELECT * FROM products
ORDER BY COALESCE(published_at, '1970-01-01'::timestamptz) DESC, id DESC;

-- หรือ handle ใน cursor:
SELECT * FROM products
WHERE (
  CASE WHEN $cursor_date IS NULL
    THEN published_at IS NULL AND id < $cursor_id
    ELSE (
      published_at < $cursor_date
      OR (published_at = $cursor_date AND id < $cursor_id)
      OR published_at IS NULL
    )
  END
)
ORDER BY published_at DESC NULLS LAST, id DESC;
```

## Complete Search + Filter + Sort + Pagination

```typescript
// src/services/products-complete.service.ts

interface CompleteProductQueryOptions {
  // Search
  search?: string;
  
  // Filters
  filters?: {
    categories?: string[];
    brands?: string[];
    priceRange?: { min?: number; max?: number };
    inStock?: boolean;
    tags?: string[];
    status?: "active" | "inactive" | "draft";
    createdAfter?: Date;
    createdBefore?: Date;
  };
  
  // Sorting (multiple fields)
  sort?: Array<{
    field: "price" | "name" | "createdAt" | "rating" | "stock";
    direction: "asc" | "desc";
  }>;
  
  // Pagination
  pagination?: {
    type: "offset";
    page: number;
    pageSize: number;
  } | {
    type: "cursor";
    cursor?: string;
    limit: number;
  };
}

const COLUMN_MAP: Record<string, string> = {
  price: "p.price",
  name: "p.name",
  createdAt: "p.created_at",
  rating: "p.average_rating",
  stock: "p.stock_quantity",
};

export async function queryProducts(options: CompleteProductQueryOptions) {
  const { search, filters = {}, sort = [{ field: "createdAt", direction: "desc" }], pagination } = options;
  
  const params: unknown[] = [];
  const addParam = (v: unknown) => { params.push(v); return `$${params.length}`; };
  
  const conditions: string[] = ["p.deleted_at IS NULL"];
  
  // Search condition
  if (search) {
    conditions.push(
      `p.search_vector @@ websearch_to_tsquery('english', ${addParam(search)})`
    );
  }
  
  // Category filter
  if (filters.categories?.length) {
    conditions.push(`c.slug = ANY(${addParam(filters.categories)}::text[])`);
  }
  
  // Brand filter
  if (filters.brands?.length) {
    conditions.push(`p.brand = ANY(${addParam(filters.brands)}::text[])`);
  }
  
  // Price range
  if (filters.priceRange?.min !== undefined) {
    conditions.push(`p.price >= ${addParam(filters.priceRange.min)}`);
  }
  if (filters.priceRange?.max !== undefined) {
    conditions.push(`p.price <= ${addParam(filters.priceRange.max)}`);
  }
  
  // Stock
  if (filters.inStock !== undefined) {
    conditions.push(filters.inStock ? "p.stock_quantity > 0" : "p.stock_quantity = 0");
  }
  
  // Tags
  if (filters.tags?.length) {
    conditions.push(`p.tags && ${addParam(filters.tags)}::text[]`);
  }
  
  // Status
  if (filters.status) {
    conditions.push(`p.status = ${addParam(filters.status)}`);
  }
  
  // Date range
  if (filters.createdAfter) {
    conditions.push(`p.created_at >= ${addParam(filters.createdAfter)}`);
  }
  if (filters.createdBefore) {
    conditions.push(`p.created_at < ${addParam(filters.createdBefore)}`);
  }
  
  // Handle cursor condition
  if (pagination?.type === "cursor" && pagination.cursor) {
    const cursorData = decodeCursor<{ c: string; i: string }>(pagination.cursor);
    conditions.push(`
      (p.created_at < ${addParam(cursorData.c)}::timestamptz
       OR (p.created_at = ${addParam(cursorData.c)}::timestamptz AND p.id::text < ${addParam(cursorData.i)}))
    `);
  }
  
  const whereClause = conditions.join(" AND ");
  
  // Sort
  const validSort = sort
    .filter(s => COLUMN_MAP[s.field])
    .map(s => `${COLUMN_MAP[s.field]} ${s.direction.toUpperCase()}`);
  
  // Always add id as tiebreaker
  const lastSortDir = sort[sort.length - 1]?.direction === "asc" ? "ASC" : "DESC";
  validSort.push(`p.id ${lastSortDir}`);
  
  const orderByClause = validSort.join(", ");
  
  // Search ranking (ถ้ามี search)
  const rankSelect = search
    ? `, ts_rank_cd(p.search_vector, websearch_to_tsquery('english', ${addParam(search)})) AS search_rank`
    : "";
  
  const rankOrder = search ? "search_rank DESC, " : "";
  
  // Build limit/offset
  let limitClause = "";
  
  if (pagination?.type === "offset") {
    const { page, pageSize } = pagination;
    limitClause = `LIMIT ${addParam(pageSize)} OFFSET ${addParam((page - 1) * pageSize)}`;
  } else if (pagination?.type === "cursor") {
    limitClause = `LIMIT ${addParam(pagination.limit + 1)}`;
  } else {
    limitClause = `LIMIT ${addParam(50)}`; // default
  }
  
  const query = `
    SELECT
      p.id,
      p.name,
      p.sku,
      p.price,
      p.stock_quantity,
      p.tags,
      p.thumbnail_url,
      p.average_rating,
      p.status,
      p.created_at,
      c.name AS category_name,
      c.slug AS category_slug
      ${rankSelect}
    FROM products p
    LEFT JOIN categories c ON c.id = p.category_id
    WHERE ${whereClause}
    ORDER BY ${rankOrder}${orderByClause}
    ${limitClause}
  `;
  
  const result = await pool.query(query, params);
  let rows = result.rows;
  
  if (pagination?.type === "cursor") {
    const hasMore = rows.length > pagination.limit;
    if (hasMore) rows.pop();
    
    return {
      data: rows,
      pageInfo: {
        hasNextPage: hasMore,
        endCursor: rows.length > 0
          ? encodeCursor({
              c: rows[rows.length - 1].created_at.toISOString(),
              i: rows[rows.length - 1].id,
            })
          : null,
      },
    };
  }
  
  // Count สำหรับ offset pagination
  if (pagination?.type === "offset") {
    const countQuery = `
      SELECT COUNT(*) AS total
      FROM products p
      LEFT JOIN categories c ON c.id = p.category_id
      WHERE ${whereClause}
    `;
    
    const countParams = params.slice(0, -2); // ลบ limit+offset
    const countResult = await pool.query(countQuery, countParams);
    const total = parseInt(countResult.rows[0].total);
    
    return {
      data: rows,
      pagination: {
        page: pagination.page,
        pageSize: pagination.pageSize,
        total,
        totalPages: Math.ceil(total / pagination.pageSize),
        hasNextPage: pagination.page < Math.ceil(total / pagination.pageSize),
        hasPreviousPage: pagination.page > 1,
      },
    };
  }
  
  return { data: rows };
}
```

## สรุป: Search Strategy เลือกอย่างไร?

```
Scenario                          → Strategy
────────────────────────────────────────────────────────
Small table (<10K rows), simple   → LIKE '%term%' (พอ)
Prefix autocomplete               → LIKE 'term%' + B-tree index
Natural language search           → PostgreSQL Full-Text Search
Typo-tolerant search              → pg_trgm + similarity
Large catalog, fast search        → Meilisearch/Typesense
Enterprise, analytics             → Elasticsearch
E-commerce with facets            → Meilisearch (best DX)
Real-time sync needed             → Meilisearch + PostgreSQL trigger/CDC
Budget-constrained                → PostgreSQL FTS + pg_trgm
```

```sql
-- Quick win: ปรับ LIKE '%term%' ที่ช้าด้วย pg_trgm

-- ก่อน: ช้า 3 วินาที
SELECT * FROM products WHERE name LIKE '%laptop gaming%';

-- หลัง: เร็ว 50ms
CREATE EXTENSION pg_trgm;
CREATE INDEX idx_products_name_trgm ON products USING GIN(name gin_trgm_ops);
SELECT * FROM products WHERE name ILIKE '%laptop gaming%'; -- ใช้ trgm index!

-- เร็วขึ้น 60x โดยไม่ต้องเปลี่ยน application code!
```

```typescript
// Best practice: Implement search ทั้งสองแบบ
// PostgreSQL สำหรับ consistent data
// Meilisearch สำหรับ user-facing search

// Keep in sync ด้วย event-driven approach:
// 1. Product saved → emit ProductUpdated event
// 2. SearchSyncService receives event → update Meilisearch
// 3. User search → Meilisearch (fast)
// 4. Admin/reports → PostgreSQL (accurate)
```

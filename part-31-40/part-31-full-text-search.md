# Part 31: Full-Text Search ใน PostgreSQL

## บทนำ

Full-Text Search (FTS) เป็นความสามารถสำคัญของ PostgreSQL ที่ช่วยให้คุณค้นหาข้อความในเอกสารขนาดใหญ่ได้อย่างมีประสิทธิภาพ แตกต่างจากการค้นหาแบบ LIKE ที่เป็นการจับคู่รูปแบบตัวอักษร FTS ทำการวิเคราะห์ความหมายของคำ (lexical analysis) และสร้าง index ที่เหมาะสมสำหรับการค้นหา

## 1. Full-Text Search คืออะไร: แตกต่างจาก LIKE อย่างไร

### LIKE ทำงานอย่างไร

```sql
-- การค้นหาแบบ LIKE
SELECT * FROM articles WHERE content LIKE '%database%';
SELECT * FROM articles WHERE content LIKE '%postgresql%';
SELECT * FROM articles WHERE title ILIKE '%full text%';
```

ปัญหาของ LIKE:
1. **Sequential Scan**: ต้องอ่านทุก row ทุก character
2. **ไม่เข้าใจ morphology**: 'running' ≠ 'run', 'databases' ≠ 'database'
3. **ไม่มี relevance score**: ไม่รู้ว่าผลไหนเกี่ยวข้องมากกว่า
4. **ช้ามากกับข้อมูลขนาดใหญ่**: O(n * m) complexity

### Full-Text Search แก้ปัญหาอย่างไร

```sql
-- Full-Text Search
SELECT * FROM articles 
WHERE to_tsvector('english', content) @@ to_tsquery('english', 'database');

-- ค้นหา 'running' ได้แม้ text มีแค่ 'run'
SELECT to_tsvector('english', 'The runner was running fast') 
    @@ to_tsquery('english', 'run');
-- ผลลัพธ์: true

-- มี relevance score
SELECT title, ts_rank(to_tsvector('english', content), 
                       to_tsquery('english', 'database')) AS rank
FROM articles
ORDER BY rank DESC;
```

### เปรียบเทียบประสิทธิภาพ

| Feature | LIKE | Full-Text Search |
|---------|------|-----------------|
| Index support | B-tree (prefix only) | GIN/GiST |
| Stemming | ไม่รองรับ | รองรับ |
| Stop words | ไม่รองรับ | รองรับ |
| Relevance ranking | ไม่มี | มี |
| Multi-language | ไม่มี | มี |
| Performance (large data) | ช้า | เร็ว |
| Phrase search | ยาก | รองรับ |

---

## 2. tsvector: Document Representation

`tsvector` คือ data type ที่แทน document ในรูปแบบที่เหมาะสำหรับ full-text search ประกอบด้วย lexemes (รากศัพท์) พร้อม positions และ weights

### โครงสร้างของ tsvector

```sql
-- ตัวอย่าง tsvector
SELECT to_tsvector('english', 'The quick brown fox jumps over the lazy dog');
-- ผลลัพธ์:
-- 'brown':3 'dog':9 'fox':4 'jump':5 'lazi':8 'quick':2

-- สังเกต:
-- 1. Stop words ถูกลบออก: 'the', 'over'
-- 2. Stemming: 'jumps' → 'jump', 'lazy' → 'lazi'
-- 3. Position numbers: คำแต่ละคำมีตำแหน่ง

-- tsvector แบบ manual
SELECT 'cat:3 dog:5 fish:1'::tsvector;
SELECT 'fat:2A cat:3B rat:5C'::tsvector;  -- มี weight
```

### Weight Levels

tsvector รองรับ weight 4 ระดับ: A, B, C, D (A สำคัญที่สุด)

```sql
-- กำหนด weight ให้ lexeme
SELECT setweight(to_tsvector('english', 'PostgreSQL database'), 'A') ||
       setweight(to_tsvector('english', 'The best relational database system'), 'B');
-- ผลลัพธ์:
-- 'best':5B 'databas':2A,6B 'postgreql':1A 'relat':4B 'system':7B

-- tsvector concatenation
SELECT to_tsvector('english', 'hello world') || 
       to_tsvector('english', 'foo bar');
```

### การแปลง text เป็น tsvector

```sql
-- รูปแบบ: to_tsvector(config, text)
SELECT to_tsvector('english', 'Hello World');
SELECT to_tsvector('simple', 'Hello World');    -- ไม่ stem, ไม่ remove stop words
SELECT to_tsvector('thai', 'ฐานข้อมูล PostgreSQL');  -- ถ้า Thai dict ติดตั้ง

-- ใช้ default config (จาก default_text_search_config)
SELECT to_tsvector('Hello World');

-- ดู configuration ปัจจุบัน
SHOW default_text_search_config;
```

---

## 3. tsquery: Search Query

`tsquery` คือ data type ที่แทน search query ประกอบด้วย lexemes และ operators

### Operators ใน tsquery

```sql
-- AND operator (&)
SELECT to_tsquery('english', 'database & postgresql');

-- OR operator (|)
SELECT to_tsquery('english', 'database | mysql');

-- NOT operator (!)
SELECT to_tsquery('english', 'database & !mysql');

-- Phrase operator (<-> หรือ <N>)
SELECT to_tsquery('english', 'full <-> text <-> search');
SELECT to_tsquery('english', 'full <2> search');  -- ระยะห่าง 2 คำ

-- Grouping
SELECT to_tsquery('english', '(postgresql | mysql) & database');
```

### Prefix Matching

```sql
-- Prefix matching ด้วย :*
SELECT to_tsquery('english', 'data:*');  -- match: database, databases, data, etc.
SELECT to_tsquery('english', 'post:*');  -- match: postgresql, postgres, post

-- ตัวอย่างการใช้งาน
SELECT title FROM articles 
WHERE to_tsvector('english', title) @@ to_tsquery('english', 'postgre:*');
```

---

## 4. to_tsvector และ to_tsquery Functions

### to_tsvector

```sql
-- Signature
to_tsvector(config regconfig, document text) → tsvector
to_tsvector(document text) → tsvector

-- Examples
SELECT to_tsvector('english', 'PostgreSQL is a powerful database system');
-- 'databas':5 'postgreql':1 'power':4 'system':6

SELECT to_tsvector('simple', 'PostgreSQL is a powerful database system');
-- 'a':3 'database':5 'is':2 'postgresql':1 'powerful':4 'system':6

-- ดู lexemes
SELECT * FROM ts_debug('english', 'Running quickly through databases');
```

### to_tsquery

```sql
-- Signature
to_tsquery(config regconfig, querytext text) → tsquery
to_tsquery(querytext text) → tsquery

-- ต้องเป็น valid syntax ไม่งั้น error
SELECT to_tsquery('english', 'database & postgresql');
SELECT to_tsquery('english', 'data:* & post:*');

-- นำมาใช้ร่วมกัน
SELECT to_tsvector('english', 'PostgreSQL database tutorial') 
    @@ to_tsquery('english', 'database & tutorial');
-- ผลลัพธ์: true
```

---

## 5. plainto_tsquery: Simple Search

`plainto_tsquery` แปลง plain text เป็น tsquery โดยถือว่าทุกคำเป็น AND

```sql
-- Signature
plainto_tsquery(config regconfig, querytext text) → tsquery
plainto_tsquery(querytext text) → tsquery

-- ตัวอย่าง
SELECT plainto_tsquery('english', 'database postgresql tutorial');
-- ผลลัพธ์: 'databas' & 'postgreql' & 'tutori'

-- เหมาะสำหรับ user input ที่ไม่รู้ syntax
SELECT title FROM articles 
WHERE to_tsvector('english', content) @@ plainto_tsquery('english', 'database cluster setup');

-- เปรียบเทียบกับ to_tsquery
SELECT to_tsquery('english', 'database cluster setup');      -- ERROR! ต้องมี operators
SELECT plainto_tsquery('english', 'database cluster setup'); -- OK!
```

---

## 6. phraseto_tsquery: Phrase Search

`phraseto_tsquery` ค้นหา phrase ที่ต้องเรียงตามลำดับ

```sql
-- Signature
phraseto_tsquery(config regconfig, querytext text) → tsquery
phraseto_tsquery(querytext text) → tsquery

-- ตัวอย่าง
SELECT phraseto_tsquery('english', 'full text search');
-- ผลลัพธ์: 'full' <-> 'text' <-> 'search'

-- ค้นหา phrase ที่เรียงติดกัน
SELECT to_tsvector('english', 'PostgreSQL full text search is powerful') 
    @@ phraseto_tsquery('english', 'full text search');
-- ผลลัพธ์: true

SELECT to_tsvector('english', 'PostgreSQL search text full is powerful') 
    @@ phraseto_tsquery('english', 'full text search');
-- ผลลัพธ์: false (เรียงผิดลำดับ)

-- ใช้งานจริง
SELECT title, content 
FROM articles 
WHERE to_tsvector('english', content) @@ phraseto_tsquery('english', 'open source database');
```

---

## 7. websearch_to_tsquery: Google-like Syntax

`websearch_to_tsquery` รองรับ syntax คล้าย Google Search

```sql
-- Signature
websearch_to_tsquery(config regconfig, querytext text) → tsquery
websearch_to_tsquery(querytext text) → tsquery

-- Syntax ที่รองรับ:
-- word1 word2     → AND
-- "phrase here"   → phrase search
-- -word           → NOT
-- word1 OR word2  → OR

-- ตัวอย่าง
SELECT websearch_to_tsquery('english', 'postgresql database');
-- ผลลัพธ์: 'postgreql' & 'databas'

SELECT websearch_to_tsquery('english', '"full text search"');
-- ผลลัพธ์: 'full' <-> 'text' <-> 'search'

SELECT websearch_to_tsquery('english', 'postgresql -mysql');
-- ผลลัพธ์: 'postgreql' & !'mysql'

SELECT websearch_to_tsquery('english', 'postgresql OR mysql database');
-- ผลลัพธ์: ('postgreql' | 'mysql') & 'databas'

-- ตัวอย่างที่ซับซ้อน
SELECT websearch_to_tsquery('english', '"open source" database -oracle OR postgresql');

-- ใช้งานจริง - search box input
SELECT title, ts_rank(to_tsvector('english', content), query) AS rank
FROM articles, websearch_to_tsquery('english', $1) AS query
WHERE to_tsvector('english', content) @@ query
ORDER BY rank DESC
LIMIT 10;
```

---

## 8. @@ Operator: Match tsvector against tsquery

```sql
-- Syntax
tsvector @@ tsquery → boolean
tsquery @@ tsvector → boolean

-- ตัวอย่าง
SELECT to_tsvector('english', 'Hello World') @@ to_tsquery('english', 'hello');
-- true

SELECT to_tsvector('english', 'Hello World') @@ to_tsquery('english', 'goodbye');
-- false

-- ใช้ใน WHERE clause
SELECT * FROM documents
WHERE body_tsvector @@ to_tsquery('english', 'search & term');

-- ใช้กับ column ที่เก็บ tsvector
CREATE TABLE articles (
    id SERIAL PRIMARY KEY,
    title TEXT,
    content TEXT,
    search_vector TSVECTOR
);

SELECT * FROM articles
WHERE search_vector @@ websearch_to_tsquery('english', 'database tutorial');

-- @@ รองรับทั้ง 2 ทิศทาง
SELECT 'fat & cat'::tsquery @@ 'a fat cat sat on a mat'::tsvector;
```

---

## 9. ts_rank: คำนวณ Relevance Score

```sql
-- Signature
ts_rank(vector tsvector, query tsquery) → float4
ts_rank(weights float4[], vector tsvector, query tsquery) → float4
ts_rank(vector tsvector, query tsquery, normalization integer) → float4
ts_rank(weights float4[], vector tsvector, query tsquery, normalization integer) → float4

-- ตัวอย่าง
SELECT ts_rank(
    to_tsvector('english', 'PostgreSQL is a powerful open source database'),
    to_tsquery('english', 'database')
);
-- ผลลัพธ์: 0.0759906

-- Score สูงกว่าเมื่อ keyword ปรากฏบ่อยกว่า
SELECT ts_rank(
    to_tsvector('english', 'database database database'),
    to_tsquery('english', 'database')
);
-- ผลลัพธ์: 0.285964 (สูงกว่า)

-- Normalization options:
-- 0  = ไม่ normalize (default)
-- 1  = หาร rank ด้วย 1 + log(document length)
-- 2  = หาร rank ด้วย document length
-- 4  = หาร rank ด้วย mean harmonic distance
-- 8  = หาร rank ด้วย number of unique words
-- 16 = หาร rank ด้วย 1 + log(number of unique words)
-- 32 = หาร rank ด้วย rank + 1

SELECT ts_rank(
    to_tsvector('english', content),
    to_tsquery('english', 'database'),
    1  -- normalize by document length
) AS rank
FROM articles
ORDER BY rank DESC;

-- weights array: { D-weight, C-weight, B-weight, A-weight }
SELECT ts_rank(
    '{0.1, 0.2, 0.4, 1.0}',
    to_tsvector('english', content),
    to_tsquery('english', 'database')
) AS rank
FROM articles;
```

---

## 10. ts_rank_cd: Cover Density Ranking

`ts_rank_cd` คำนวณ rank โดยคำนึงถึงความหนาแน่นของ keywords ในเอกสาร

```sql
-- Signature (เหมือน ts_rank)
ts_rank_cd(vector tsvector, query tsquery) → float4
ts_rank_cd(weights float4[], vector tsvector, query tsquery) → float4

-- ตัวอย่าง
SELECT ts_rank_cd(
    to_tsvector('english', 'database postgresql database cluster database'),
    to_tsquery('english', 'database & postgresql')
);

-- เปรียบเทียบ ts_rank vs ts_rank_cd
SELECT 
    ts_rank(v, q) AS rank,
    ts_rank_cd(v, q) AS rank_cd
FROM (
    SELECT 
        to_tsvector('english', 'The quick brown fox. The database fox jumps.') AS v,
        to_tsquery('english', 'fox & database') AS q
) t;

-- ts_rank_cd ดีกว่าเมื่อ keywords อยู่ใกล้กัน
SELECT ts_rank_cd(
    to_tsvector('english', content),
    to_tsquery('english', 'database & cluster')
) AS relevance
FROM articles
ORDER BY relevance DESC
LIMIT 20;
```

---

## 11. ts_headline: Highlight Matches ใน Result

```sql
-- Signature
ts_headline(document text, query tsquery) → text
ts_headline(config regconfig, document text, query tsquery) → text
ts_headline(document text, query tsquery, options text) → text
ts_headline(config regconfig, document text, query tsquery, options text) → text

-- ตัวอย่างพื้นฐาน
SELECT ts_headline(
    'english',
    'PostgreSQL is a powerful open source relational database system',
    to_tsquery('english', 'database & system')
);
-- ผลลัพธ์: ...open source relational <b>database</b> <b>system</b>

-- Options:
-- StartSel, StopSel: เปลี่ยน highlight tag
-- MaxWords: max words ใน fragment
-- MinWords: min words ใน fragment
-- ShortWord: words ≤ N characters ถูกข้าม
-- HighlightAll: highlight ทุก occurrence
-- MaxFragments: max number of fragments
-- FragmentDelimiter: delimiter ระหว่าง fragments

SELECT ts_headline(
    'english',
    content,
    to_tsquery('english', 'database'),
    'StartSel=<mark>, StopSel=</mark>, MaxWords=20, MinWords=10'
) AS highlighted
FROM articles
WHERE to_tsvector('english', content) @@ to_tsquery('english', 'database');

-- Multiple fragments
SELECT ts_headline(
    content,
    websearch_to_tsquery('english', 'postgresql database cluster'),
    'MaxFragments=3, MaxWords=20, FragmentDelimiter=" ... "'
) AS snippet
FROM articles;

-- ใช้กับ HTML
SELECT ts_headline(
    'english',
    content,
    query,
    'StartSel=<span class="highlight">, StopSel=</span>'
) AS highlighted_content
FROM articles, websearch_to_tsquery('english', $1) AS query
WHERE to_tsvector('english', content) @@ query;
```

---

## 12. Languages: English, Simple, Thai

### English Configuration

```sql
-- ดู configurations ที่มี
SELECT cfgname, cfgparser FROM pg_ts_config;

-- English: stemming + stop words
SELECT to_tsvector('english', 'The running dogs were quickly jumping');
-- 'dog':3 'jump':6 'quick':5 'run':2

-- Debug processing
SELECT * FROM ts_debug('english', 'The running dogs were quickly jumping');
```

### Simple Configuration

```sql
-- Simple: lowercase, no stemming, no stop word removal
SELECT to_tsvector('simple', 'The running dogs were quickly jumping');
-- 'dogs':3 'jumping':6 'quickly':5 'running':2 'the':1 'were':4

-- เหมาะสำหรับ:
-- - ชื่อคน (ไม่ต้องการ stem)
-- - รหัสสินค้า
-- - Tags
-- - ภาษาที่ไม่มี dictionary
```

### Thai Language Support

```sql
-- ติดตั้ง ithaiword สำหรับ Thai text
-- (ต้องติดตั้ง package เพิ่ม)

-- ถ้า Thai dictionary พร้อม
SELECT to_tsvector('thai', 'ฐานข้อมูล PostgreSQL ทำงานได้ดี');

-- ถ้าไม่มี Thai dictionary ใช้ simple
SELECT to_tsvector('simple', 'ฐานข้อมูล PostgreSQL ทำงานได้ดี');

-- สร้าง text search configuration สำหรับ Thai + English
CREATE TEXT SEARCH CONFIGURATION thai_english (COPY = simple);
ALTER TEXT SEARCH CONFIGURATION thai_english
    ALTER MAPPING FOR asciiword WITH english_stem;

-- ดู parsers
SELECT * FROM ts_token_type('default');
```

---

## 13. GIN vs GiST Index สำหรับ Full-Text

### GIN Index (Generalized Inverted Index)

```sql
-- สร้าง GIN index
CREATE INDEX idx_articles_fts_gin 
ON articles USING GIN(to_tsvector('english', content));

-- GIN บน stored tsvector column
CREATE TABLE articles (
    id SERIAL PRIMARY KEY,
    content TEXT,
    search_vector TSVECTOR
);
CREATE INDEX idx_articles_search_gin 
ON articles USING GIN(search_vector);

-- ข้อดี GIN:
-- - เร็วกว่า GiST ในการค้นหา (query time)
-- - รองรับ phrase search
-- - เหมาะสำหรับ static/rarely-updated data

-- ข้อเสีย GIN:
-- - ช้ากว่า GiST ในการ insert/update
-- - ใช้ memory มากกว่า
-- - Build time นานกว่า
```

### GiST Index (Generalized Search Tree)

```sql
-- สร้าง GiST index
CREATE INDEX idx_articles_fts_gist 
ON articles USING GIST(to_tsvector('english', content));

-- ข้อดี GiST:
-- - Insert/Update เร็วกว่า GIN
-- - ใช้ disk space น้อยกว่า
-- - Balanced tree structure

-- ข้อเสีย GiST:
-- - Query เร็วน้อยกว่า GIN
-- - ไม่รองรับ phrase search
-- - Lossy (อาจ false positive)

-- สรุปการเลือก:
-- GIN: ข้อมูล static หรือ read-heavy, ต้องการ phrase search
-- GiST: write-heavy, ต้องการ fast inserts
```

### Concurrent Index Build

```sql
-- Build index โดยไม่ lock table
CREATE INDEX CONCURRENTLY idx_articles_fts_gin 
ON articles USING GIN(to_tsvector('english', coalesce(title, '') || ' ' || coalesce(content, '')));
```

---

## 14. tsvector Column (Stored): CREATE INDEX

```sql
-- สร้าง table พร้อม stored tsvector column
CREATE TABLE blog_posts (
    id SERIAL PRIMARY KEY,
    title TEXT NOT NULL,
    content TEXT,
    tags TEXT[],
    author TEXT,
    published_at TIMESTAMPTZ DEFAULT NOW(),
    search_vector TSVECTOR  -- stored column
);

-- สร้าง GIN index
CREATE INDEX idx_blog_posts_search 
ON blog_posts USING GIN(search_vector);

-- Generated column (PostgreSQL 12+) - ถูก update อัตโนมัติ
ALTER TABLE blog_posts 
ADD COLUMN search_vector_gen TSVECTOR 
GENERATED ALWAYS AS (
    setweight(to_tsvector('english', coalesce(title, '')), 'A') ||
    setweight(to_tsvector('english', coalesce(content, '')), 'B') ||
    setweight(to_tsvector('english', coalesce(author, '')), 'C')
) STORED;

CREATE INDEX idx_blog_posts_search_gen 
ON blog_posts USING GIN(search_vector_gen);

-- ค้นหาโดยใช้ stored index
SELECT id, title 
FROM blog_posts
WHERE search_vector_gen @@ websearch_to_tsquery('english', 'postgresql database');
```

---

## 15. Multi-Column Search: COALESCE + Weight

```sql
-- Multi-column search ที่รวม title, content, tags
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    description TEXT,
    category TEXT,
    tags TEXT[],
    brand TEXT
);

-- สร้าง tsvector รวม columns
SELECT 
    setweight(to_tsvector('english', coalesce(name, '')), 'A') ||
    setweight(to_tsvector('english', coalesce(brand, '')), 'B') ||
    setweight(to_tsvector('english', coalesce(category, '')), 'C') ||
    setweight(to_tsvector('english', coalesce(description, '')), 'D')
AS search_vector
FROM products;

-- เพิ่ม tags (array to string)
SELECT 
    setweight(to_tsvector('english', coalesce(name, '')), 'A') ||
    setweight(to_tsvector('english', coalesce(brand, '')), 'B') ||
    setweight(to_tsvector('english', array_to_string(coalesce(tags, '{}'), ' ')), 'B') ||
    setweight(to_tsvector('english', coalesce(description, '')), 'D')
AS search_vector
FROM products;

-- Update stored column
UPDATE products 
SET search_vector = 
    setweight(to_tsvector('english', coalesce(name, '')), 'A') ||
    setweight(to_tsvector('english', coalesce(brand, '')), 'B') ||
    setweight(to_tsvector('english', coalesce(description, '')), 'D');
```

---

## 16. setweight: ให้ Weight แก่ Columns

```sql
-- Weight levels: A (highest) > B > C > D (lowest)
-- Default weights: {D=0.1, C=0.2, B=0.4, A=1.0}

-- ตัวอย่าง setweight
SELECT setweight(to_tsvector('english', 'PostgreSQL'), 'A');
-- 'postgreql':1A

SELECT setweight(to_tsvector('english', 'database system'), 'B');
-- 'databas':1B 'system':2B

-- รวม weighted vectors
SELECT 
    setweight(to_tsvector('english', 'PostgreSQL Tutorial'), 'A') ||
    setweight(to_tsvector('english', 'A complete guide to PostgreSQL database'), 'D');
-- 'complet':3D 'databas':6D 'guid':4D 'postgreql':1A,5D 'tutori':2A

-- Weighted search ranking
SELECT 
    title,
    ts_rank(
        '{0.1, 0.2, 0.4, 1.0}',  -- weights: D, C, B, A
        setweight(to_tsvector('english', title), 'A') ||
        setweight(to_tsvector('english', content), 'D'),
        to_tsquery('english', 'database')
    ) AS rank
FROM articles
ORDER BY rank DESC;
```

---

## 17. Search Configuration

```sql
-- ดู configurations ทั้งหมด
\dF
SELECT cfgname FROM pg_ts_config;

-- รายละเอียด configuration
\dFd+ english
SELECT * FROM ts_token_type('default');

-- ดู stop words
SELECT word FROM pg_get_stopwords('english') LIMIT 20;

-- สร้าง custom configuration
CREATE TEXT SEARCH CONFIGURATION myconfig (COPY = english);

-- เปลี่ยน mapping
ALTER TEXT SEARCH CONFIGURATION myconfig
    ALTER MAPPING FOR hword, hword_part, word 
    WITH english_stem;

-- ทดสอบ configuration
SELECT to_tsvector('myconfig', 'The running dogs were quickly jumping over fences');

-- Set default configuration
SET default_text_search_config = 'english';
ALTER DATABASE mydb SET default_text_search_config = 'english';

-- pg_catalog.english vs english
SELECT to_tsvector('pg_catalog.english', 'Running databases');
SELECT to_tsvector('english', 'Running databases');  -- same
```

---

## 18. Custom Dictionaries

```sql
-- ดู dictionaries ที่มี
\dFd
SELECT dictname FROM pg_ts_dict;

-- สร้าง synonym dictionary
CREATE TEXT SEARCH DICTIONARY english_synonyms (
    TEMPLATE = synonym,
    SYNONYMS = english.syn  -- file path
);

-- สร้าง thesaurus dictionary
CREATE TEXT SEARCH DICTIONARY thes (
    TEMPLATE = thesaurus,
    DictFile = english,
    Dictionary = english_stem
);

-- สร้าง ispell dictionary สำหรับ spell checking
CREATE TEXT SEARCH DICTIONARY english_ispell (
    TEMPLATE = ispell,
    DictFile = english,
    AffFile = english,
    StopWords = english
);

-- ใช้ custom dictionary ใน configuration
ALTER TEXT SEARCH CONFIGURATION english
    ALTER MAPPING FOR word 
    WITH english_synonyms, english_stem;

-- Unaccent dictionary (ลบ accent marks)
CREATE EXTENSION unaccent;
CREATE TEXT SEARCH DICTIONARY unaccent_dict (
    TEMPLATE = unaccent,
    Rules = unaccent
);

ALTER TEXT SEARCH CONFIGURATION english
    ALTER MAPPING FOR hword, hword_part, word 
    WITH unaccent_dict, english_stem;

-- ทดสอบ
SELECT to_tsvector('english', 'café résumé naïve');
```

---

## 19. Real-time Search: Update tsvector Trigger

```sql
-- สร้าง trigger function สำหรับ update tsvector
CREATE OR REPLACE FUNCTION update_search_vector()
RETURNS TRIGGER AS $$
BEGIN
    NEW.search_vector := 
        setweight(to_tsvector('english', coalesce(NEW.title, '')), 'A') ||
        setweight(to_tsvector('english', coalesce(NEW.summary, '')), 'B') ||
        setweight(to_tsvector('english', coalesce(NEW.content, '')), 'C') ||
        setweight(to_tsvector('english', coalesce(NEW.author, '')), 'D');
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- สร้าง trigger
CREATE TRIGGER update_article_search_vector
    BEFORE INSERT OR UPDATE OF title, summary, content, author
    ON articles
    FOR EACH ROW
    EXECUTE FUNCTION update_search_vector();

-- ทดสอบ trigger
INSERT INTO articles (title, content, author)
VALUES ('PostgreSQL Full-Text Search', 
        'A comprehensive guide to FTS in PostgreSQL',
        'John Doe');

SELECT search_vector FROM articles WHERE title = 'PostgreSQL Full-Text Search';
-- ดูผลลัพธ์ tsvector

-- Rebuild all search vectors
UPDATE articles SET title = title;  -- trigger update ทุก row
```

---

## 20. Performance: GIN Index

```sql
-- วัด performance ก่อน/หลัง index
EXPLAIN ANALYZE
SELECT * FROM articles 
WHERE to_tsvector('english', content) @@ to_tsquery('english', 'database');

-- สร้าง GIN index
CREATE INDEX idx_articles_content_fts 
ON articles USING GIN(to_tsvector('english', content));

-- ตรวจสอบ index usage
EXPLAIN ANALYZE
SELECT * FROM articles 
WHERE to_tsvector('english', content) @@ to_tsquery('english', 'database');

-- Index บน expression ต้องตรงกับ query
-- ไม่ match: to_tsvector('simple', content)
-- Match: to_tsvector('english', content)

-- GIN parameters
CREATE INDEX idx_articles_fts_gin 
ON articles USING GIN(search_vector) WITH (fastupdate = on);
-- fastupdate: batch updates (เร็วกว่าแต่ recovery ช้ากว่า)

-- Vacuum GIN index
VACUUM ANALYZE articles;

-- Check index size
SELECT 
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexname::regclass)) AS index_size
FROM pg_indexes
WHERE tablename = 'articles';
```

---

## 21. Pagination ด้วย ts_rank

```sql
-- Cursor-based pagination ด้วย ts_rank
SELECT 
    id,
    title,
    ts_rank(search_vector, query) AS rank,
    ts_headline('english', content, query, 'MaxWords=30') AS snippet
FROM articles, websearch_to_tsquery('english', 'postgresql database') AS query
WHERE search_vector @@ query
ORDER BY rank DESC, id DESC
LIMIT 10;

-- Offset pagination (ง่ายกว่าแต่ช้ากว่าสำหรับ large offset)
SELECT 
    id, 
    title,
    ts_rank(search_vector, query) AS rank
FROM articles, websearch_to_tsquery('english', $1) AS query
WHERE search_vector @@ query
ORDER BY rank DESC, id
LIMIT $2 OFFSET $3;

-- Keyset pagination (เร็วกว่า)
-- ใช้ (rank, id) เป็น cursor
WITH ranked AS (
    SELECT 
        id,
        title,
        ts_rank(search_vector, query) AS rank
    FROM articles, websearch_to_tsquery('english', $1) AS query
    WHERE search_vector @@ query
)
SELECT * FROM ranked
WHERE (rank, id) < ($2, $3)  -- cursor from previous page
ORDER BY rank DESC, id DESC
LIMIT 10;

-- Count total results (expensive)
SELECT COUNT(*) 
FROM articles, websearch_to_tsquery('english', $1) AS query
WHERE search_vector @@ query;
```

---

## 22. Full Working Example: Blog Post Search System

### Schema

```sql
-- Complete blog post search system

-- 1. Tables
CREATE TABLE blog_categories (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL UNIQUE,
    slug TEXT NOT NULL UNIQUE
);

CREATE TABLE blog_authors (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    email TEXT UNIQUE,
    bio TEXT
);

CREATE TABLE blog_posts (
    id SERIAL PRIMARY KEY,
    title TEXT NOT NULL,
    slug TEXT NOT NULL UNIQUE,
    summary TEXT,
    content TEXT,
    author_id INTEGER REFERENCES blog_authors(id),
    category_id INTEGER REFERENCES blog_categories(id),
    tags TEXT[] DEFAULT '{}',
    status TEXT DEFAULT 'draft' CHECK (status IN ('draft', 'published', 'archived')),
    published_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    -- Full-text search columns
    search_vector TSVECTOR,
    -- Stats
    view_count INTEGER DEFAULT 0
);

-- 2. Indexes
CREATE INDEX idx_blog_posts_search ON blog_posts USING GIN(search_vector);
CREATE INDEX idx_blog_posts_status ON blog_posts(status);
CREATE INDEX idx_blog_posts_published ON blog_posts(published_at DESC) WHERE status = 'published';
CREATE INDEX idx_blog_posts_author ON blog_posts(author_id);
CREATE INDEX idx_blog_posts_category ON blog_posts(category_id);

-- 3. Trigger function
CREATE OR REPLACE FUNCTION blog_posts_search_vector_update()
RETURNS TRIGGER AS $$
DECLARE
    v_category_name TEXT;
    v_author_name TEXT;
    v_tags_text TEXT;
BEGIN
    -- Get related data
    SELECT name INTO v_category_name 
    FROM blog_categories WHERE id = NEW.category_id;
    
    SELECT name INTO v_author_name 
    FROM blog_authors WHERE id = NEW.author_id;
    
    -- Convert tags array to text
    v_tags_text := array_to_string(coalesce(NEW.tags, '{}'), ' ');
    
    -- Build weighted search vector
    NEW.search_vector := 
        setweight(to_tsvector('english', coalesce(NEW.title, '')), 'A') ||
        setweight(to_tsvector('english', coalesce(NEW.summary, '')), 'B') ||
        setweight(to_tsvector('english', coalesce(v_tags_text, '')), 'B') ||
        setweight(to_tsvector('english', coalesce(v_author_name, '')), 'C') ||
        setweight(to_tsvector('english', coalesce(v_category_name, '')), 'C') ||
        setweight(to_tsvector('english', coalesce(NEW.content, '')), 'D');
    
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- 4. Trigger
CREATE TRIGGER blog_posts_search_vector_trigger
    BEFORE INSERT OR UPDATE OF title, summary, content, tags, author_id, category_id
    ON blog_posts
    FOR EACH ROW
    EXECUTE FUNCTION blog_posts_search_vector_update();

-- 5. Updated at trigger
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER blog_posts_updated_at
    BEFORE UPDATE ON blog_posts
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at();

-- 6. Sample data
INSERT INTO blog_categories (name, slug) VALUES
    ('Database', 'database'),
    ('PostgreSQL', 'postgresql'),
    ('Performance', 'performance'),
    ('Security', 'security'),
    ('DevOps', 'devops');

INSERT INTO blog_authors (name, email, bio) VALUES
    ('John Smith', 'john@example.com', 'Database architect with 10 years experience'),
    ('Jane Doe', 'jane@example.com', 'PostgreSQL DBA and performance tuning expert'),
    ('Bob Johnson', 'bob@example.com', 'DevOps engineer specializing in database clusters');

INSERT INTO blog_posts (title, summary, content, author_id, category_id, tags, status, published_at)
VALUES 
(
    'Getting Started with PostgreSQL Full-Text Search',
    'A comprehensive guide to implementing full-text search in PostgreSQL databases',
    'PostgreSQL provides powerful full-text search capabilities that allow you to search through large amounts of text data efficiently. In this tutorial, we will explore the tsvector and tsquery data types, learn how to create appropriate indexes, and build a complete search system.',
    1, 2, 
    ARRAY['postgresql', 'full-text-search', 'tutorial'],
    'published', NOW() - INTERVAL '5 days'
),
(
    'PostgreSQL Performance Tuning: Index Strategies',
    'Learn how to optimize PostgreSQL queries using proper indexing strategies',
    'Database performance is crucial for application success. This article covers B-tree indexes, GIN indexes, GiST indexes, and when to use each type. We also discuss partial indexes and expression indexes for advanced optimization.',
    2, 3,
    ARRAY['postgresql', 'performance', 'indexes', 'optimization'],
    'published', NOW() - INTERVAL '3 days'
),
(
    'Setting Up PostgreSQL Cluster with Streaming Replication',
    'Step-by-step guide to configure PostgreSQL primary-replica cluster',
    'High availability is essential for production databases. This guide walks through setting up PostgreSQL streaming replication, configuring WAL archiving, and implementing automatic failover using Patroni.',
    3, 5,
    ARRAY['postgresql', 'replication', 'cluster', 'high-availability'],
    'published', NOW() - INTERVAL '1 day'
);

-- 7. Search function
CREATE OR REPLACE FUNCTION search_blog_posts(
    search_query TEXT,
    p_category_id INTEGER DEFAULT NULL,
    p_author_id INTEGER DEFAULT NULL,
    p_limit INTEGER DEFAULT 10,
    p_offset INTEGER DEFAULT 0
)
RETURNS TABLE (
    post_id INTEGER,
    title TEXT,
    summary TEXT,
    author_name TEXT,
    category_name TEXT,
    tags TEXT[],
    published_at TIMESTAMPTZ,
    rank FLOAT4,
    snippet TEXT,
    total_count BIGINT
) AS $$
DECLARE
    v_query TSQUERY;
BEGIN
    v_query := websearch_to_tsquery('english', search_query);
    
    RETURN QUERY
    WITH search_results AS (
        SELECT 
            p.id,
            p.title,
            p.summary,
            a.name AS author_name,
            c.name AS category_name,
            p.tags,
            p.published_at,
            ts_rank('{0.1,0.2,0.4,1.0}', p.search_vector, v_query) AS rank,
            ts_headline(
                'english', 
                p.content, 
                v_query,
                'MaxFragments=2, MaxWords=30, MinWords=15, FragmentDelimiter=" ... "'
            ) AS snippet,
            COUNT(*) OVER() AS total_count
        FROM blog_posts p
        JOIN blog_authors a ON a.id = p.author_id
        JOIN blog_categories c ON c.id = p.category_id
        WHERE 
            p.status = 'published'
            AND p.search_vector @@ v_query
            AND (p_category_id IS NULL OR p.category_id = p_category_id)
            AND (p_author_id IS NULL OR p.author_id = p_author_id)
    )
    SELECT 
        id, title, summary, author_name, category_name, tags,
        published_at, rank, snippet, total_count
    FROM search_results
    ORDER BY rank DESC, published_at DESC
    LIMIT p_limit
    OFFSET p_offset;
END;
$$ LANGUAGE plpgsql;

-- 8. Test search
SELECT * FROM search_blog_posts('postgresql database search');
SELECT * FROM search_blog_posts('performance optimization', p_category_id := 3);
SELECT * FROM search_blog_posts('"full text" search tutorial');
```

### Node.js Implementation

```typescript
// blog-search.ts - Full-Text Search API

import { Pool, QueryResult } from 'pg';

interface SearchResult {
  post_id: number;
  title: string;
  summary: string;
  author_name: string;
  category_name: string;
  tags: string[];
  published_at: Date;
  rank: number;
  snippet: string;
  total_count: number;
}

interface SearchOptions {
  query: string;
  categoryId?: number;
  authorId?: number;
  page?: number;
  pageSize?: number;
  language?: string;
}

interface SearchResponse {
  results: SearchResult[];
  pagination: {
    page: number;
    pageSize: number;
    totalCount: number;
    totalPages: number;
  };
  took: number;
}

const pool = new Pool({
  host: process.env.DB_HOST || 'localhost',
  port: parseInt(process.env.DB_PORT || '5432'),
  database: process.env.DB_NAME || 'blog_db',
  user: process.env.DB_USER || 'postgres',
  password: process.env.DB_PASSWORD,
  max: 20,
});

async function searchBlogPosts(options: SearchOptions): Promise<SearchResponse> {
  const startTime = Date.now();
  const {
    query,
    categoryId,
    authorId,
    page = 1,
    pageSize = 10,
    language = 'english',
  } = options;

  const offset = (page - 1) * pageSize;

  // Validate and sanitize input
  if (!query || query.trim().length < 2) {
    throw new Error('Search query must be at least 2 characters');
  }

  try {
    const result: QueryResult<SearchResult> = await pool.query(
      `SELECT * FROM search_blog_posts($1, $2, $3, $4, $5)`,
      [query, categoryId || null, authorId || null, pageSize, offset]
    );

    const totalCount = result.rows[0]?.total_count || 0;
    const took = Date.now() - startTime;

    return {
      results: result.rows,
      pagination: {
        page,
        pageSize,
        totalCount: Number(totalCount),
        totalPages: Math.ceil(Number(totalCount) / pageSize),
      },
      took,
    };
  } catch (error) {
    console.error('Search error:', error);
    throw error;
  }
}

async function getSuggestions(partialQuery: string, limit: number = 5): Promise<string[]> {
  // Auto-complete suggestions based on existing content
  const result = await pool.query(
    `SELECT DISTINCT 
        word 
     FROM (
        SELECT unnest(regexp_split_to_array(
            lower(title), '\\s+'
        )) AS word
        FROM blog_posts
        WHERE status = 'published'
     ) words
     WHERE word LIKE $1 AND length(word) > 2
     ORDER BY word
     LIMIT $2`,
    [`${partialQuery.toLowerCase()}%`, limit]
  );

  return result.rows.map(row => row.word);
}

async function getRelatedPosts(postId: number, limit: number = 5): Promise<any[]> {
  // Find related posts using tsvector similarity
  const result = await pool.query(
    `SELECT 
        p2.id,
        p2.title,
        p2.summary,
        ts_rank(p2.search_vector, p1.search_vector::tsquery) AS similarity
     FROM blog_posts p1
     JOIN blog_posts p2 ON p2.id != p1.id
     WHERE 
        p1.id = $1
        AND p2.status = 'published'
        AND p2.search_vector @@ (
            SELECT query FROM ts_rewrite(
                to_tsquery('english', 'placeholder'),
                'SELECT to_tsquery($1::text, word::text) FROM 
                 (SELECT unnest(tsvector_to_array(p1_inner.search_vector)) AS word 
                  FROM blog_posts p1_inner WHERE p1_inner.id = $2) sub'
            )
        )
     ORDER BY similarity DESC
     LIMIT $3`,
    [postId, postId, limit]
  );

  return result.rows;
}

// Alternative: simpler related posts
async function getRelatedPostsSimple(postId: number, limit: number = 5): Promise<any[]> {
  const result = await pool.query(
    `WITH target AS (
        SELECT search_vector, category_id, tags
        FROM blog_posts
        WHERE id = $1
     )
     SELECT 
        p.id,
        p.title,
        p.summary,
        p.published_at,
        ts_rank(p.search_vector, t.search_vector::tsquery) AS similarity
     FROM blog_posts p, target t
     WHERE 
        p.id != $1
        AND p.status = 'published'
        AND (
            p.category_id = t.category_id
            OR p.tags && t.tags  -- array overlap
        )
     ORDER BY similarity DESC, p.published_at DESC
     LIMIT $2`,
    [postId, limit]
  );

  return result.rows;
}

async function rebuildSearchVectors(): Promise<void> {
  console.log('Rebuilding search vectors...');
  const result = await pool.query(
    `UPDATE blog_posts SET title = title`  // Triggers the trigger
  );
  console.log(`Updated ${result.rowCount} posts`);
}

// Express route handler
async function searchHandler(req: any, res: any): Promise<void> {
  try {
    const {
      q: query,
      category,
      author,
      page = '1',
      size = '10',
    } = req.query;

    if (!query) {
      res.status(400).json({ error: 'Query parameter "q" is required' });
      return;
    }

    const results = await searchBlogPosts({
      query: query as string,
      categoryId: category ? parseInt(category as string) : undefined,
      authorId: author ? parseInt(author as string) : undefined,
      page: parseInt(page as string),
      pageSize: Math.min(parseInt(size as string), 50),
    });

    res.json(results);
  } catch (error) {
    console.error('Search error:', error);
    res.status(500).json({ error: 'Search failed' });
  }
}

// Main
async function main() {
  // Test search
  const results = await searchBlogPosts({
    query: 'postgresql performance',
    page: 1,
    pageSize: 5,
  });

  console.log(`Found ${results.pagination.totalCount} results in ${results.took}ms`);
  results.results.forEach(r => {
    console.log(`[${r.rank.toFixed(4)}] ${r.title}`);
    console.log(`  ${r.snippet}`);
  });

  // Suggestions
  const suggestions = await getSuggestions('post');
  console.log('Suggestions:', suggestions);

  await pool.end();
}

main().catch(console.error);
```

---

## 23. Advanced Queries

```sql
-- Spell correction (ใกล้เคียงกัน)
SELECT word, similarity(word, 'postgresq') AS sim
FROM (
    SELECT unnest(tsvector_to_array(search_vector)) AS word
    FROM blog_posts
) words
WHERE similarity(word, 'postgresq') > 0.3
ORDER BY sim DESC
LIMIT 5;

-- Boost recent posts
SELECT 
    id,
    title,
    ts_rank(search_vector, query) * 
    EXP(-0.1 * EXTRACT(days FROM NOW() - published_at)) AS boosted_rank
FROM blog_posts, websearch_to_tsquery('english', 'database') AS query
WHERE search_vector @@ query AND status = 'published'
ORDER BY boosted_rank DESC;

-- Faceted search
SELECT 
    c.name AS category,
    COUNT(*) AS post_count
FROM blog_posts p
JOIN blog_categories c ON c.id = p.category_id
WHERE p.search_vector @@ websearch_to_tsquery('english', 'database')
GROUP BY c.name
ORDER BY post_count DESC;

-- Search with tag filter
SELECT id, title
FROM blog_posts
WHERE 
    search_vector @@ websearch_to_tsquery('english', 'postgresql')
    AND tags @> ARRAY['tutorial']
ORDER BY published_at DESC;

-- Incremental search (as user types)
SELECT id, title
FROM blog_posts
WHERE 
    search_vector @@ to_tsquery('english', 'postg:*')
    AND status = 'published'
LIMIT 5;
```

---

## 24. Monitoring และ Diagnostics

```sql
-- ดู index usage statistics
SELECT 
    indexrelname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
WHERE relname = 'blog_posts';

-- ดู slow queries
SELECT 
    query,
    calls,
    total_exec_time / calls AS avg_ms,
    rows / calls AS avg_rows
FROM pg_stat_statements
WHERE query LIKE '%search_vector%'
ORDER BY avg_ms DESC
LIMIT 10;

-- ตรวจสอบ tsvector size
SELECT 
    id,
    title,
    pg_column_size(search_vector) AS vector_size_bytes,
    length(search_vector::text) AS vector_text_length
FROM blog_posts
ORDER BY vector_size_bytes DESC
LIMIT 10;

-- ดู lexemes ใน tsvector
SELECT 
    id,
    tsvector_to_array(search_vector) AS lexemes
FROM blog_posts
LIMIT 3;
```

---

## สรุป

Full-Text Search ใน PostgreSQL เป็นระบบที่ทรงพลังสำหรับการค้นหาข้อความ สิ่งสำคัญที่ควรจำ:

1. **tsvector** แทน document ในรูป lexemes + positions + weights
2. **tsquery** แทน search query พร้อม boolean operators
3. **GIN index** เร็วสุดสำหรับ FTS queries ควรใช้เสมอ
4. **setweight** ช่วย rank ผลลัพธ์ตามความสำคัญของ field
5. **ts_rank** + **ts_rank_cd** ใช้คำนวณ relevance score
6. **ts_headline** สร้าง snippet พร้อม highlighted keywords
7. **Trigger** ช่วย maintain tsvector column อัตโนมัติ
8. **websearch_to_tsquery** เหมาะที่สุดสำหรับ user input เพราะรองรับ Google-like syntax

สำหรับ production system ควร:
- เก็บ tsvector เป็น stored column (ไม่ compute ทุก query)
- ใช้ GIN index เสมอ
- กำหนด weights ให้ title > summary > content
- Implement pagination ด้วย keyset แทน offset สำหรับ large result sets

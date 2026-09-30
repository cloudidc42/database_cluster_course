# Part 79: Vector Database Integration (pgvector)

## Vector Databases ในยุค AI/ML

เราอยู่ในยุคที่ AI/ML กลายเป็นส่วนสำคัญของทุกแอปพลิเคชัน vector databases จึงกลายเป็นเทคโนโลยีที่จำเป็น

```
Vector Embedding คืออะไร?
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Text → AI Model → Vector (array of floats)
"Hello World" → [0.23, -0.45, 0.12, 0.89, ..., -0.34]  (1536 dimensions)

ข้อความที่ "คล้ายกัน" จะมี vector ที่ "ใกล้กัน"

"I love dogs" → [0.23, -0.45, 0.12, ...]
"Dogs are my favorite pets" → [0.24, -0.43, 0.11, ...]  ← ใกล้กันมาก

"The stock market crashed" → [-0.67, 0.23, -0.89, ...]  ← ห่างมาก

ใช้ประโยชน์: หาข้อความ/รูปภาพ/เสียงที่ "คล้ายกัน" โดยไม่ต้องใช้ keyword matching
```

### Use Cases หลัก

```
1. SEMANTIC SEARCH:
   - ค้นหา document ที่มีความหมายคล้ายกัน (ไม่ใช่แค่ keyword match)
   - "Find articles about machine learning" → ค้นเจอ "neural networks", "deep learning" ด้วย
   - E-commerce: ค้นหาสินค้า "ของขวัญสำหรับวันเกิด" → เจอสินค้าที่เหมาะกัน

2. RECOMMENDATION SYSTEMS:
   - User A ชอบ movie X, Y, Z
   - หา users อื่นที่มี embedding ใกล้เคียงกัน
   - แนะนำ movies ที่ users เหล่านั้นชอบ

3. IMAGE SIMILARITY:
   - ค้นหารูปภาพที่คล้ายกัน (reverse image search)
   - Duplicate detection
   - Product image search

4. RAG (Retrieval-Augmented Generation):
   - ให้ ChatGPT ตอบคำถามเกี่ยวกับ documents ของเรา
   - ค้นหา relevant documents → ส่งเป็น context ให้ LLM
   - Customer support: "How do I reset my password?" → ค้นหา FAQ

5. ANOMALY DETECTION:
   - หา transactions ที่ "ผิดปกติ" (ห่างจาก normal patterns)
   - Fraud detection
   - Intrusion detection
```

---

## pgvector: PostgreSQL Extension for Vector Similarity

### Installation

```bash
# Docker (เร็วที่สุด)
docker run -d \
  --name postgres-vector \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=vectordb \
  -p 5432:5432 \
  pgvector/pgvector:pg16

# หรือ Ubuntu/Debian
sudo apt install postgresql-16-pgvector

# หรือ build from source
cd /tmp
git clone --branch v0.7.0 https://github.com/pgvector/pgvector.git
cd pgvector
make
sudo make install
```

```sql
-- Enable extension
CREATE EXTENSION IF NOT EXISTS vector;

-- ตรวจสอบ version
SELECT extversion FROM pg_extension WHERE extname = 'vector';
-- ควรเห็น: 0.7.0 หรือสูงกว่า
```

### Vector Data Type

```sql
-- Vector type: VECTOR(dimensions)
-- OpenAI text-embedding-3-small: 1536 dimensions
-- OpenAI text-embedding-3-large: 3072 dimensions
-- Cohere embed-v3: 1024 dimensions
-- Sentence Transformers all-MiniLM-L6-v2: 384 dimensions

-- สร้าง table สำหรับเก็บ document embeddings
CREATE TABLE documents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title TEXT NOT NULL,
    content TEXT NOT NULL,
    url TEXT,
    source TEXT,                      -- 'web', 'pdf', 'database'
    metadata JSONB DEFAULT '{}',
    
    -- Embedding vectors
    embedding VECTOR(1536),            -- OpenAI text-embedding-3-small
    content_embedding VECTOR(384),     -- Smaller model for speed
    
    -- Metadata
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    indexed_at TIMESTAMPTZ
);

-- Table สำหรับ image embeddings
CREATE TABLE images (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    filename TEXT NOT NULL,
    s3_key TEXT NOT NULL,
    description TEXT,
    
    -- CLIP model: 512 dimensions
    embedding VECTOR(512),
    
    tags TEXT[],
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

---

## Vector Operations

### Distance Metrics

```sql
-- pgvector รองรับ 3 distance metrics:

-- 1. L2 Distance (Euclidean) <->
-- ดีสำหรับ: normalized embeddings, image similarity
-- เล็ก = คล้ายกันมาก

-- 2. Inner Product <#>
-- ดีสำหรับ: unnormalized embeddings, maximum inner product search
-- ใหญ่ (magnitude สูง) = คล้ายกันมาก (negative for similarity)

-- 3. Cosine Distance <=>
-- ดีสำหรับ: text embeddings (direction matters, not magnitude)
-- เล็ก = คล้ายกันมาก (0 = identical, 2 = opposite)

-- ตัวอย่าง: ค้นหา documents ที่คล้ายกับ query
-- Query vector (จาก OpenAI API)
SELECT
    id,
    title,
    content,
    -- Cosine similarity = 1 - cosine_distance
    1 - (embedding <=> '[0.12, -0.34, 0.56, ...]'::vector) AS similarity
FROM documents
ORDER BY embedding <=> '[0.12, -0.34, 0.56, ...]'::vector
LIMIT 10;

-- L2 distance
SELECT id, title, embedding <-> '[0.12, -0.34, ...]'::vector AS distance
FROM documents
ORDER BY embedding <-> '[0.12, -0.34, ...]'::vector
LIMIT 10;

-- Inner product (negative for similarity)
SELECT id, title, (embedding <#> '[0.12, -0.34, ...]'::vector) * -1 AS inner_product
FROM documents
ORDER BY inner_product DESC
LIMIT 10;
```

---

## Indexing สำหรับ Vector Search

### IVFFlat Index

```sql
-- IVFFlat: Inverted File with Flat quantization
-- ค้นหาโดยแบ่งเป็น "lists" (clusters) แล้วค้นใน lists ที่ใกล้ที่สุด

-- สร้าง IVFFlat index สำหรับ cosine distance
-- lists parameter: sqrt(total_rows) เป็น starting point
-- 1M rows → lists = 1000
-- 10M rows → lists = 3162

CREATE INDEX ON documents USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);  -- สำหรับ 10K-1M rows

-- สำหรับ L2 distance
CREATE INDEX ON documents USING ivfflat (embedding vector_l2_ops)
WITH (lists = 100);

-- สำหรับ Inner product
CREATE INDEX ON documents USING ivfflat (embedding vector_ip_ops)
WITH (lists = 100);

-- ตั้งค่า probes: จำนวน lists ที่จะค้นใน query
-- สูงขึ้น = accurate กว่า แต่ช้ากว่า
-- trade-off: probes = 1 (fastest) ↔ probes = lists (exact search)
SET ivfflat.probes = 10;  -- ค้น 10 lists (default: 1)

-- ทดสอบ performance
EXPLAIN ANALYZE
SELECT id, title, 1 - (embedding <=> $1::vector) AS similarity
FROM documents
ORDER BY embedding <=> $1::vector
LIMIT 10;
```

```sql
-- IVFFlat ต้องสร้าง index หลังจากมีข้อมูลแล้ว (ต้องการ data สำหรับ clustering)
-- ถ้าสร้าง index ก่อนมีข้อมูล → performance แย่

-- Step 1: Insert data ก่อน
INSERT INTO documents (title, content, embedding)
SELECT ...;

-- Step 2: สร้าง index หลังมีข้อมูลแล้ว
CREATE INDEX documents_embedding_idx 
ON documents USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);

-- Step 3: Analyze statistics
ANALYZE documents;
```

### HNSW Index (แนะนำ)

```sql
-- HNSW: Hierarchical Navigable Small World
-- ดีกว่า IVFFlat ในหลาย aspects:
-- - Better recall (accuracy)
-- - Faster query time
-- - No need to pre-specify clusters
-- - Works well with dynamic data inserts

-- Parameters:
-- m: connections per layer (default 16)
--    สูงขึ้น = more accurate, more memory
--    Range: 2-100 (recommend: 16-64)
-- ef_construction: build-time accuracy (default 64)
--    สูงขึ้น = slower build, better index quality
--    Range: 4-1000 (recommend: 64-200)

CREATE INDEX ON documents USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- Query-time parameter
-- ef_search: accuracy vs speed tradeoff during query
-- สูงขึ้น = more accurate, slower
SET hnsw.ef_search = 100;  -- default: 40

-- ตรวจสอบ index size
SELECT
    pg_size_pretty(pg_indexes_size('documents')) AS index_size,
    pg_size_pretty(pg_total_relation_size('documents')) AS total_size;
```

### Choosing Between IVFFlat and HNSW

```
Performance Comparison (approximate, 1M vectors, 1536 dims):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

                    IVFFlat          HNSW
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Build Time:         Fast (minutes)   Slow (hours for large datasets)
Index Size:         Small (~1.5x data) Large (~2.5x data)
Recall@10:          ~94%             ~99%
Query Time:         ~10ms            ~5ms
Dynamic Inserts:    Requires rebuild  OK (incremental)
Memory during build: Low             High

Recommendation:
- ข้อมูลน้อยกว่า 100K rows: Exact search (no index) ได้เลย
- ข้อมูล 100K-10M rows: HNSW (better recall, faster queries)
- ข้อมูล >10M rows: IVFFlat (lower memory for build)
- Frequently inserting new vectors: HNSW
- Batch insert then read: IVFFlat (better memory during build)
```

---

## Generating Embeddings

### OpenAI Embeddings

```typescript
// embeddings-openai.ts
import OpenAI from 'openai';

interface EmbeddingResult {
  text: string;
  embedding: number[];
  model: string;
  tokens: number;
}

class OpenAIEmbeddings {
  private client: OpenAI;
  
  constructor(apiKey: string) {
    this.client = new OpenAI({ apiKey });
  }
  
  async embed(text: string): Promise<EmbeddingResult> {
    // Truncate if too long (model limit: 8191 tokens for text-embedding-3-small)
    const truncated = this.truncateText(text, 8000);
    
    const response = await this.client.embeddings.create({
      model: 'text-embedding-3-small',  // 1536 dims, cheap ($0.02/1M tokens)
      input: truncated,
      encoding_format: 'float',
    });
    
    return {
      text: truncated,
      embedding: response.data[0].embedding,
      model: response.model,
      tokens: response.usage.total_tokens,
    };
  }
  
  async embedBatch(texts: string[]): Promise<EmbeddingResult[]> {
    // OpenAI API รองรับ batch สูงสุด 2048 texts ต่อ request
    const BATCH_SIZE = 100;
    const results: EmbeddingResult[] = [];
    
    for (let i = 0; i < texts.length; i += BATCH_SIZE) {
      const batch = texts.slice(i, i + BATCH_SIZE);
      const truncated = batch.map(t => this.truncateText(t, 8000));
      
      const response = await this.client.embeddings.create({
        model: 'text-embedding-3-small',
        input: truncated,
        encoding_format: 'float',
      });
      
      for (let j = 0; j < response.data.length; j++) {
        results.push({
          text: truncated[j],
          embedding: response.data[j].embedding,
          model: response.model,
          tokens: 0,  // batch doesn't give per-item token count
        });
      }
      
      // Rate limiting: pause between batches
      if (i + BATCH_SIZE < texts.length) {
        await new Promise(resolve => setTimeout(resolve, 200));
      }
    }
    
    return results;
  }
  
  private truncateText(text: string, maxTokens: number): string {
    // Approximate: 1 token ≈ 4 characters
    const maxChars = maxTokens * 4;
    if (text.length <= maxChars) return text;
    return text.substring(0, maxChars);
  }
}
```

### Local Embeddings (Ollama / HuggingFace)

```typescript
// embeddings-local.ts
// ใช้ Ollama สำหรับ local embeddings (ไม่ต้องส่ง data ไป cloud)

interface OllamaEmbeddingResponse {
  embedding: number[];
}

class OllamaEmbeddings {
  private baseUrl: string;
  private model: string;
  
  constructor(baseUrl: string = 'http://localhost:11434', model: string = 'nomic-embed-text') {
    this.baseUrl = baseUrl;
    this.model = model;
    // nomic-embed-text: 768 dimensions
    // mxbai-embed-large: 1024 dimensions
  }
  
  async embed(text: string): Promise<number[]> {
    const response = await fetch(`${this.baseUrl}/api/embeddings`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        model: this.model,
        prompt: text,
      }),
    });
    
    if (!response.ok) {
      throw new Error(`Ollama API error: ${response.status}`);
    }
    
    const data = await response.json() as OllamaEmbeddingResponse;
    return data.embedding;
  }
  
  async embedBatch(texts: string[]): Promise<number[][]> {
    // Ollama ไม่มี batch API ต้อง loop
    const results: number[][] = [];
    
    for (const text of texts) {
      results.push(await this.embed(text));
    }
    
    return results;
  }
}

// HuggingFace Inference API
class HuggingFaceEmbeddings {
  private apiKey: string;
  private modelId: string;
  
  constructor(apiKey: string, modelId: string = 'sentence-transformers/all-MiniLM-L6-v2') {
    this.apiKey = apiKey;
    this.modelId = modelId;
    // all-MiniLM-L6-v2: 384 dimensions, fast, good quality
    // all-mpnet-base-v2: 768 dimensions, slower, better quality
  }
  
  async embed(texts: string[]): Promise<number[][]> {
    const response = await fetch(
      `https://api-inference.huggingface.co/pipeline/feature-extraction/${this.modelId}`,
      {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${this.apiKey}`,
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({ inputs: texts, options: { wait_for_model: true } }),
      }
    );
    
    if (!response.ok) {
      throw new Error(`HuggingFace API error: ${response.status}`);
    }
    
    return response.json();
  }
}
```

---

## Hybrid Search: Vector + Full-Text + Filters

```sql
-- Hybrid Search คือการรวม:
-- 1. Vector similarity (semantic)
-- 2. Full-text search (keyword)
-- 3. Metadata filters (structured)

-- สร้าง full-text search index
ALTER TABLE documents ADD COLUMN ts_content TSVECTOR
    GENERATED ALWAYS AS (to_tsvector('english', content)) STORED;

CREATE INDEX ON documents USING GIN (ts_content);

-- Hybrid Search Query
WITH vector_results AS (
    SELECT
        id,
        1 - (embedding <=> $1::vector) AS vector_score,
        ROW_NUMBER() OVER (ORDER BY embedding <=> $1::vector) AS vector_rank
    FROM documents
    WHERE source = $2  -- filter by source (metadata filter)
    ORDER BY embedding <=> $1::vector
    LIMIT 50
),
text_results AS (
    SELECT
        id,
        ts_rank(ts_content, plainto_tsquery('english', $3)) AS text_score,
        ROW_NUMBER() OVER (
            ORDER BY ts_rank(ts_content, plainto_tsquery('english', $3)) DESC
        ) AS text_rank
    FROM documents
    WHERE ts_content @@ plainto_tsquery('english', $3)
      AND source = $2  -- same filter
    LIMIT 50
),
-- Reciprocal Rank Fusion (RRF) - combine rankings
combined AS (
    SELECT
        COALESCE(v.id, t.id) AS id,
        COALESCE(v.vector_score, 0) AS vector_score,
        COALESCE(t.text_score, 0) AS text_score,
        -- RRF formula: 1/(k + rank) where k=60
        COALESCE(1.0 / (60 + v.vector_rank), 0) +
        COALESCE(1.0 / (60 + t.text_rank), 0) AS rrf_score
    FROM vector_results v
    FULL OUTER JOIN text_results t ON v.id = t.id
)
SELECT
    d.id,
    d.title,
    d.content,
    c.vector_score,
    c.text_score,
    c.rrf_score
FROM combined c
JOIN documents d ON c.id = d.id
ORDER BY c.rrf_score DESC
LIMIT 10;
```

---

## RAG System Implementation

### Overview ของ RAG

```
RAG (Retrieval-Augmented Generation) Flow:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

INGESTION PHASE (offline):
Document → Chunk → Embed → Store in pgvector

QUERY PHASE (online):
User Question → Embed → Search pgvector → Top K docs → 
Prompt LLM with [Question + Context] → Answer

Benefits:
- LLM ตอบคำถามเกี่ยวกับ documents ของเรา (ที่ไม่ได้ train มา)
- ลด hallucination (มี context จริงๆ)
- Up-to-date information
- Citable sources
```

### Full Working Example: Document Search System

```typescript
// rag-system.ts
import { Pool } from 'pg';
import OpenAI from 'openai';

interface Document {
  id: string;
  title: string;
  content: string;
  url?: string;
  metadata?: Record<string, any>;
}

interface SearchResult {
  id: string;
  title: string;
  content: string;
  similarity: number;
  url?: string;
}

interface RAGResponse {
  answer: string;
  sources: SearchResult[];
  tokensUsed: number;
}

class RAGSystem {
  private pool: Pool;
  private openai: OpenAI;
  private embeddingModel = 'text-embedding-3-small';
  private completionModel = 'gpt-4o-mini';
  
  constructor(databaseUrl: string, openaiApiKey: string) {
    this.pool = new Pool({ connectionString: databaseUrl });
    this.openai = new OpenAI({ apiKey: openaiApiKey });
  }
  
  // INGESTION: Add document to the knowledge base
  async ingestDocument(doc: Document): Promise<string> {
    const chunks = this.chunkDocument(doc.content, 1000, 200);
    const insertedIds: string[] = [];
    
    for (let i = 0; i < chunks.length; i++) {
      const chunk = chunks[i];
      
      // Generate embedding for this chunk
      const embeddingResponse = await this.openai.embeddings.create({
        model: this.embeddingModel,
        input: chunk,
      });
      
      const embedding = embeddingResponse.data[0].embedding;
      
      // Store in PostgreSQL
      const result = await this.pool.query<{ id: string }>(`
        INSERT INTO documents (title, content, url, metadata, embedding)
        VALUES ($1, $2, $3, $4, $5)
        RETURNING id
      `, [
        i === 0 ? doc.title : `${doc.title} (Part ${i + 1})`,
        chunk,
        doc.url,
        { ...doc.metadata, chunkIndex: i, totalChunks: chunks.length, originalId: doc.id },
        `[${embedding.join(',')}]`,
      ]);
      
      insertedIds.push(result.rows[0].id);
    }
    
    console.log(`Ingested "${doc.title}": ${chunks.length} chunks`);
    return insertedIds[0];
  }
  
  // Chunk document into overlapping segments
  private chunkDocument(text: string, chunkSize: number = 1000, overlap: number = 200): string[] {
    const chunks: string[] = [];
    
    // Split by sentences first (better than arbitrary character split)
    const sentences = text.match(/[^.!?]+[.!?]+/g) || [text];
    
    let currentChunk = '';
    let previousChunk = '';
    
    for (const sentence of sentences) {
      const prospective = currentChunk + ' ' + sentence;
      
      if (prospective.length > chunkSize && currentChunk.length > 0) {
        chunks.push(currentChunk.trim());
        
        // Overlap: carry over last portion of current chunk
        const words = currentChunk.split(' ');
        const overlapWords = Math.floor(overlap / 6);  // Approximate words
        previousChunk = words.slice(-overlapWords).join(' ');
        currentChunk = previousChunk + ' ' + sentence;
      } else {
        currentChunk = prospective;
      }
    }
    
    if (currentChunk.trim()) {
      chunks.push(currentChunk.trim());
    }
    
    return chunks;
  }
  
  // SEARCH: Find relevant documents
  async search(
    query: string,
    options: {
      limit?: number;
      threshold?: number;
      filter?: Record<string, any>;
    } = {}
  ): Promise<SearchResult[]> {
    const { limit = 5, threshold = 0.7 } = options;
    
    // Generate query embedding
    const embeddingResponse = await this.openai.embeddings.create({
      model: this.embeddingModel,
      input: query,
    });
    
    const queryEmbedding = embeddingResponse.data[0].embedding;
    
    // Search pgvector
    const result = await this.pool.query<{
      id: string;
      title: string;
      content: string;
      url: string;
      similarity: number;
    }>(`
      SELECT
        id,
        title,
        content,
        url,
        1 - (embedding <=> $1::vector) AS similarity
      FROM documents
      WHERE 1 - (embedding <=> $1::vector) > $2
      ORDER BY embedding <=> $1::vector
      LIMIT $3
    `, [
      `[${queryEmbedding.join(',')}]`,
      threshold,
      limit,
    ]);
    
    return result.rows;
  }
  
  // RAG: Answer question using retrieved documents
  async answer(question: string): Promise<RAGResponse> {
    // Step 1: Retrieve relevant documents
    const sources = await this.search(question, { limit: 5, threshold: 0.6 });
    
    if (sources.length === 0) {
      return {
        answer: "I don't have enough information to answer this question based on the available documents.",
        sources: [],
        tokensUsed: 0,
      };
    }
    
    // Step 2: Build prompt with context
    const context = sources
      .map((s, i) => `[Document ${i + 1}: ${s.title}]\n${s.content}`)
      .join('\n\n---\n\n');
    
    const systemPrompt = `You are a helpful assistant. Answer questions based ONLY on the provided context documents. 
If the answer is not in the context, say "I don't have information about this in the provided documents."
Always cite which document(s) you used.`;
    
    const userPrompt = `Context Documents:\n${context}\n\nQuestion: ${question}`;
    
    // Step 3: Call LLM
    const completion = await this.openai.chat.completions.create({
      model: this.completionModel,
      messages: [
        { role: 'system', content: systemPrompt },
        { role: 'user', content: userPrompt },
      ],
      temperature: 0.1,
      max_tokens: 1000,
    });
    
    const answer = completion.choices[0].message.content || '';
    const tokensUsed = completion.usage?.total_tokens || 0;
    
    return { answer, sources, tokensUsed };
  }
  
  // Batch ingest from URL
  async ingestFromWebPage(url: string): Promise<void> {
    // ในการใช้งานจริง ใช้ cheerio หรือ puppeteer scrape content
    const content = `Scraped content from ${url}...`;  // placeholder
    
    await this.ingestDocument({
      id: `web-${Date.now()}`,
      title: `Page: ${url}`,
      content,
      url,
      metadata: { type: 'web', scrapedAt: new Date().toISOString() },
    });
  }
}
```

### REST API สำหรับ Document Search

```typescript
// search-api.ts
import express from 'express';
import { Pool } from 'pg';
import OpenAI from 'openai';

const app = express();
app.use(express.json());

const pool = new Pool({ connectionString: process.env.DATABASE_URL });
const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

// POST /api/search - Semantic search
app.post('/api/search', async (req, res) => {
  const { query, limit = 10, threshold = 0.7, filters = {} } = req.body;
  
  if (!query) {
    return res.status(400).json({ error: 'query is required' });
  }
  
  try {
    // Generate embedding
    const embResponse = await openai.embeddings.create({
      model: 'text-embedding-3-small',
      input: query,
    });
    
    const embedding = embResponse.data[0].embedding;
    
    // Build dynamic filter
    let filterClause = '';
    const filterParams: any[] = [];
    
    if (filters.source) {
      filterParams.push(filters.source);
      filterClause += ` AND source = $${filterParams.length + 3}`;
    }
    
    // Search
    const result = await pool.query(`
      SELECT
        id,
        title,
        LEFT(content, 200) AS excerpt,
        url,
        metadata,
        1 - (embedding <=> $1::vector) AS similarity
      FROM documents
      WHERE 1 - (embedding <=> $1::vector) > $2
        ${filterClause}
      ORDER BY embedding <=> $1::vector
      LIMIT $3
    `, [`[${embedding.join(',')}]`, threshold, limit, ...filterParams]);
    
    res.json({
      query,
      results: result.rows,
      count: result.rows.length,
      embeddingModel: 'text-embedding-3-small',
    });
  } catch (error) {
    res.status(500).json({ error: String(error) });
  }
});

// POST /api/documents - Ingest document
app.post('/api/documents', async (req, res) => {
  const { title, content, url, metadata } = req.body;
  
  if (!title || !content) {
    return res.status(400).json({ error: 'title and content are required' });
  }
  
  try {
    // Chunk content (สำหรับ long documents)
    const chunks = chunkText(content, 1000);
    const insertedIds: string[] = [];
    
    for (let i = 0; i < chunks.length; i++) {
      const embResponse = await openai.embeddings.create({
        model: 'text-embedding-3-small',
        input: chunks[i],
      });
      
      const result = await pool.query<{ id: string }>(`
        INSERT INTO documents (title, content, url, metadata, embedding)
        VALUES ($1, $2, $3, $4, $5)
        RETURNING id
      `, [
        chunks.length > 1 ? `${title} (${i + 1}/${chunks.length})` : title,
        chunks[i],
        url,
        JSON.stringify({ ...metadata, chunkIndex: i }),
        `[${embResponse.data[0].embedding.join(',')}]`,
      ]);
      
      insertedIds.push(result.rows[0].id);
    }
    
    res.json({
      message: 'Document ingested successfully',
      documentIds: insertedIds,
      chunks: chunks.length,
    });
  } catch (error) {
    res.status(500).json({ error: String(error) });
  }
});

// GET /api/similar/:id - Find similar documents
app.get('/api/similar/:id', async (req, res) => {
  const { id } = req.params;
  const { limit = 5 } = req.query;
  
  try {
    // Get embedding of the target document
    const targetResult = await pool.query(
      'SELECT embedding FROM documents WHERE id = $1',
      [id]
    );
    
    if (!targetResult.rows[0]) {
      return res.status(404).json({ error: 'Document not found' });
    }
    
    const embedding = targetResult.rows[0].embedding;
    
    // Find similar documents (exclude self)
    const result = await pool.query(`
      SELECT
        id,
        title,
        LEFT(content, 200) AS excerpt,
        1 - (embedding <=> $1::vector) AS similarity
      FROM documents
      WHERE id != $2
      ORDER BY embedding <=> $1::vector
      LIMIT $3
    `, [embedding, id, limit]);
    
    res.json({ similar: result.rows });
  } catch (error) {
    res.status(500).json({ error: String(error) });
  }
});

function chunkText(text: string, maxLength: number): string[] {
  if (text.length <= maxLength) return [text];
  
  const chunks: string[] = [];
  let start = 0;
  
  while (start < text.length) {
    let end = start + maxLength;
    
    // Try to break at sentence boundary
    if (end < text.length) {
      const lastPeriod = text.lastIndexOf('.', end);
      if (lastPeriod > start + maxLength / 2) {
        end = lastPeriod + 1;
      }
    }
    
    chunks.push(text.slice(start, end).trim());
    start = end;
  }
  
  return chunks.filter(c => c.length > 0);
}

app.listen(3000, () => console.log('Search API running on :3000'));
```

---

## Advanced: Product Recommendation System

```sql
-- Schema สำหรับ Product Recommendations
CREATE TABLE products (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    description TEXT NOT NULL,
    category TEXT NOT NULL,
    price DECIMAL(10,2),
    tags TEXT[],
    
    -- Embedding from product description + features
    description_embedding VECTOR(1536),
    
    -- Image embedding (from product photos)
    image_embedding VECTOR(512),
    
    active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX ON products USING hnsw (description_embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

CREATE INDEX ON products USING hnsw (image_embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- User interaction history
CREATE TABLE user_interactions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    product_id UUID NOT NULL REFERENCES products(id),
    interaction_type TEXT NOT NULL,  -- 'view', 'click', 'add_to_cart', 'purchase'
    weight DOUBLE PRECISION,  -- interaction strength
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- User embeddings (computed from interaction history)
CREATE TABLE user_embeddings (
    user_id UUID PRIMARY KEY,
    embedding VECTOR(1536),
    last_updated TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX ON user_embeddings USING hnsw (embedding vector_cosine_ops);
```

```sql
-- Recommendation query: similar products
CREATE OR REPLACE FUNCTION get_similar_products(
    p_product_id UUID,
    p_limit INTEGER DEFAULT 10,
    p_category_filter TEXT DEFAULT NULL
) RETURNS TABLE (
    product_id UUID,
    name TEXT,
    similarity DOUBLE PRECISION
) LANGUAGE plpgsql AS $$
DECLARE
    v_embedding VECTOR;
BEGIN
    -- Get embedding of target product
    SELECT description_embedding INTO v_embedding
    FROM products
    WHERE id = p_product_id;
    
    IF v_embedding IS NULL THEN
        RAISE EXCEPTION 'Product % not found or has no embedding', p_product_id;
    END IF;
    
    -- Find similar products
    RETURN QUERY
    SELECT
        p.id AS product_id,
        p.name,
        1 - (p.description_embedding <=> v_embedding) AS similarity
    FROM products p
    WHERE p.id != p_product_id
      AND p.active = true
      AND (p_category_filter IS NULL OR p.category = p_category_filter)
    ORDER BY p.description_embedding <=> v_embedding
    LIMIT p_limit;
END $$;

-- User-based recommendations
CREATE OR REPLACE FUNCTION get_user_recommendations(
    p_user_id UUID,
    p_limit INTEGER DEFAULT 20
) RETURNS TABLE (
    product_id UUID,
    name TEXT,
    category TEXT,
    price DECIMAL,
    score DOUBLE PRECISION,
    reason TEXT
) LANGUAGE plpgsql AS $$
DECLARE
    v_user_embedding VECTOR;
BEGIN
    -- Get user's preference embedding
    SELECT embedding INTO v_user_embedding
    FROM user_embeddings
    WHERE user_id = p_user_id;
    
    IF v_user_embedding IS NULL THEN
        -- New user: return popular products
        RETURN QUERY
        SELECT
            p.id, p.name, p.category, p.price,
            0.5::DOUBLE PRECISION AS score,
            'Popular products'::TEXT AS reason
        FROM products p
        WHERE p.active = true
        ORDER BY p.created_at DESC
        LIMIT p_limit;
        RETURN;
    END IF;
    
    -- Personalized recommendations: products similar to user's taste
    RETURN QUERY
    SELECT
        p.id, p.name, p.category, p.price,
        1 - (p.description_embedding <=> v_user_embedding) AS score,
        'Based on your browsing history'::TEXT AS reason
    FROM products p
    WHERE p.active = true
      -- Exclude already purchased products
      AND p.id NOT IN (
          SELECT product_id FROM user_interactions
          WHERE user_id = p_user_id AND interaction_type = 'purchase'
      )
    ORDER BY p.description_embedding <=> v_user_embedding
    LIMIT p_limit;
END $$;
```

---

## Monitoring และ Performance Tuning

```sql
-- ดู index usage statistics
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan AS scans,
    idx_tup_read AS tuples_read,
    idx_tup_fetch AS tuples_fetched,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE tablename = 'documents'
ORDER BY idx_scan DESC;

-- Index build progress (for large datasets)
SELECT
    phase,
    blocks_done,
    blocks_total,
    ROUND(100.0 * blocks_done / NULLIF(blocks_total, 0), 2) AS pct_done,
    tuples_done,
    current_loaders_waiting_count
FROM pg_stat_progress_create_index
WHERE relid = 'documents'::regclass;

-- Query performance analysis
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, title, 1 - (embedding <=> $1::vector) AS similarity
FROM documents
ORDER BY embedding <=> $1::vector
LIMIT 10;
-- ดูว่าใช้ Index Scan หรือ Sequential Scan
-- ถ้า Seq Scan: index อาจไม่ถูกใช้ (ต้องเพิ่ม probes)

-- Check index recall
-- สร้าง function ทดสอบ recall
CREATE OR REPLACE FUNCTION test_index_recall(
    p_sample_size INTEGER DEFAULT 100
) RETURNS DOUBLE PRECISION LANGUAGE plpgsql AS $$
DECLARE
    v_recall DOUBLE PRECISION := 0;
    v_total INTEGER := 0;
    v_doc RECORD;
    v_exact_ids UUID[];
    v_approx_ids UUID[];
    v_intersection INTEGER;
BEGIN
    FOR v_doc IN (
        SELECT id, embedding FROM documents
        ORDER BY RANDOM()
        LIMIT p_sample_size
    ) LOOP
        -- Exact search (no index)
        SET enable_indexscan = off;
        SELECT ARRAY_AGG(id ORDER BY embedding <=> v_doc.embedding)
        INTO v_exact_ids
        FROM documents
        WHERE id != v_doc.id
        LIMIT 10;
        RESET enable_indexscan;
        
        -- Approximate search (with index)
        SELECT ARRAY_AGG(id ORDER BY embedding <=> v_doc.embedding)
        INTO v_approx_ids
        FROM documents
        WHERE id != v_doc.id
        LIMIT 10;
        
        -- Calculate intersection
        SELECT COUNT(*) INTO v_intersection
        FROM unnest(v_exact_ids) AS e
        WHERE e = ANY(v_approx_ids);
        
        v_recall := v_recall + (v_intersection::DOUBLE PRECISION / 10);
        v_total := v_total + 1;
    END LOOP;
    
    RETURN v_recall / v_total;
END $$;

-- ทดสอบ recall
SELECT test_index_recall(50) AS recall_at_10;
-- ควรได้ > 0.95 (95% recall) สำหรับ HNSW ที่ default settings
```

---

## Full Schema + Migration

```sql
-- Migration: ตัวอย่างสมบูรณ์สำหรับ production

-- 001_create_vector_extension.sql
CREATE EXTENSION IF NOT EXISTS vector;
CREATE EXTENSION IF NOT EXISTS pg_trgm;  -- สำหรับ trigram text search

-- 002_create_documents_table.sql
CREATE TABLE documents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    -- Content
    title TEXT NOT NULL,
    content TEXT NOT NULL,
    content_hash TEXT GENERATED ALWAYS AS (md5(content)) STORED,
    
    -- Source info
    url TEXT,
    source TEXT NOT NULL DEFAULT 'manual',  -- 'web', 'pdf', 'api', 'manual'
    source_id TEXT,  -- ID ใน source system
    
    -- Embeddings (หลาย models สำหรับ comparison)
    embedding_small VECTOR(1536),   -- text-embedding-3-small
    embedding_large VECTOR(3072),   -- text-embedding-3-large (optional)
    
    -- Full-text search
    ts_content TSVECTOR GENERATED ALWAYS AS (
        to_tsvector('english', COALESCE(title, '') || ' ' || content)
    ) STORED,
    
    -- Metadata
    metadata JSONB DEFAULT '{}',
    tags TEXT[] DEFAULT '{}',
    language TEXT DEFAULT 'en',
    
    -- Status
    indexed BOOLEAN DEFAULT FALSE,
    active BOOLEAN DEFAULT TRUE,
    
    -- Timestamps
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    indexed_at TIMESTAMPTZ
);

-- Indexes
CREATE INDEX ON documents USING hnsw (embedding_small vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

CREATE INDEX ON documents USING GIN (ts_content);
CREATE INDEX ON documents USING GIN (tags);
CREATE INDEX ON documents USING GIN (metadata);
CREATE INDEX ON documents (source, created_at DESC);
CREATE INDEX ON documents (active, created_at DESC) WHERE active = TRUE;
CREATE UNIQUE INDEX ON documents (source, source_id) 
WHERE source_id IS NOT NULL;

-- Auto-update updated_at
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END $$;

CREATE TRIGGER documents_updated_at
    BEFORE UPDATE ON documents
    FOR EACH ROW EXECUTE FUNCTION update_updated_at();
```

---

## สรุป pgvector

```
pgvector เหมาะกับ:
✓ ต้องการ vector search + SQL ในที่เดียวกัน (ไม่ต้องใช้ฐานข้อมูลแยก)
✓ Small to medium scale (< 10M vectors สำหรับ good performance)
✓ ต้องการ join vector results กับ relational data
✓ ทีมที่คุ้นเคยกับ PostgreSQL
✓ Budget จำกัด (ไม่ต้องจ่ายค่า Pinecone/Weaviate)

pgvector ไม่เหมาะกับ:
✗ Scale > 100M vectors (ใช้ Pinecone, Weaviate, Qdrant)
✗ ต้องการ real-time indexing สำหรับ billions of vectors
✗ Multi-modal search ที่ซับซ้อนมาก

Alternatives:
- Pinecone: managed, serverless, expensive
- Weaviate: open-source, more features, separate service
- Qdrant: open-source, high performance, separate service
- Chroma: simple, embeds in Python apps
- pgvector: integrated with PostgreSQL, good for most use cases

Best Practice:
1. เริ่มด้วย HNSW index
2. Benchmark recall กับ data ของตัวเอง
3. ใช้ Hybrid search (vector + full-text) สำหรับ production
4. Cache embeddings (อย่าสร้าง embedding ซ้ำสำหรับ text เดิม)
5. Monitor index size (grows with dimensions × rows)
```

จบ Part 79 - Vector Database Integration (pgvector) ครอบคลุม:
- Vector embeddings และ use cases ใน AI/ML era
- Installation และ setup pgvector
- Distance metrics (L2, Inner Product, Cosine)
- IVFFlat และ HNSW indexes พร้อม trade-offs
- Generating embeddings (OpenAI, Ollama, HuggingFace)
- Hybrid search (vector + full-text + filters)
- Complete RAG system implementation
- Product recommendation system
- Performance monitoring และ recall testing

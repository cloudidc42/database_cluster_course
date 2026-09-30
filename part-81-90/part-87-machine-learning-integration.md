# Part 87: Machine Learning Integration กับ Database Cluster

## บทนำ

การผสาน Machine Learning เข้ากับ Database Cluster ช่วยให้สามารถทำ real-time inference, store embeddings สำหรับ semantic search, และ build intelligent applications ได้อย่างมีประสิทธิภาพ บทนี้จะครอบคลุมทุกแง่มุมตั้งแต่ pgvector ไปจนถึง LLM integration

---

## 1. ML + Database: Use Cases

### Recommendation Systems

```
Use case: แนะนำสินค้าให้ผู้ใช้
Architecture:
1. User actions (views, purchases, ratings) → PostgreSQL
2. ML model learns user preferences → generates embedding vectors
3. Store user embeddings + product embeddings ใน pgvector
4. Query: หา products ที่ใกล้เคียง user embedding
5. Return recommendations
```

### Fraud Detection

```
Use case: ตรวจจับการทำธุรกรรมผิดปกติ
Architecture:
1. Transaction arrives → extract features
2. Query feature store (Redis/PostgreSQL)
3. Call ML API for risk score
4. Score < threshold → approve
5. Score > threshold → block/review
6. Log result → retrain model periodically
```

### Churn Prediction

```
Use case: ทำนายว่าลูกค้าจะเลิกใช้บริการ
Architecture:
1. Daily batch job: compute features from PostgreSQL
2. Run ML model on all customers
3. Store churn probability ใน customer table
4. Marketing team queries high-risk customers
5. Trigger targeted campaigns
```

---

## 2. pgvector: Store และ Query Embeddings

### ติดตั้ง pgvector

```bash
# ติดตั้งจาก package manager
sudo apt-get install -y postgresql-16-pgvector

# หรือ compile จาก source
git clone https://github.com/pgvector/pgvector.git
cd pgvector
make
sudo make install

# ใน Docker
# ใช้ pgvector/pgvector image แทน postgres
```

```sql
-- Enable extension
CREATE EXTENSION IF NOT EXISTS vector;

-- ตรวจสอบ version
SELECT extversion FROM pg_extension WHERE extname = 'vector';
```

### สร้าง Table สำหรับ Product Embeddings

```sql
-- Products table with embeddings
CREATE TABLE products (
    id              SERIAL PRIMARY KEY,
    name            VARCHAR(200) NOT NULL,
    description     TEXT,
    category        VARCHAR(100),
    price           NUMERIC(10, 2),
    -- Embedding จาก text-embedding-ada-002 (1536 dimensions)
    -- หรือ sentence-transformers (384-768 dimensions)
    text_embedding  VECTOR(1536),
    image_embedding VECTOR(512),   -- จาก vision model
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Users table with preference embeddings
CREATE TABLE users (
    id              SERIAL PRIMARY KEY,
    username        VARCHAR(100),
    email           VARCHAR(200),
    -- User preference embedding (computed from interaction history)
    preference_vector VECTOR(1536),
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- IVFFlat index: approximate nearest neighbor (เร็วกว่า exact)
-- lists = sqrt(num_rows) is a good starting point
CREATE INDEX ON products USING ivfflat (text_embedding vector_cosine_ops)
    WITH (lists = 100);

-- HNSW index: better recall, more memory (PostgreSQL 16+)
CREATE INDEX ON products USING hnsw (text_embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64);
```

### Semantic Search

```python
# semantic_search.py
import psycopg2
import numpy as np
from openai import OpenAI
from sentence_transformers import SentenceTransformer

# ใช้ sentence-transformers (local, ฟรี)
model = SentenceTransformer('all-MiniLM-L6-v2')

def get_embedding(text: str) -> list[float]:
    """Generate embedding สำหรับ text"""
    embedding = model.encode(text)
    return embedding.tolist()

def semantic_search(query: str, limit: int = 10) -> list[dict]:
    """Search products ด้วย semantic similarity"""
    query_embedding = get_embedding(query)
    
    conn = psycopg2.connect("postgresql://localhost/ecommerce")
    cur = conn.cursor()
    
    cur.execute("""
        SELECT 
            id,
            name,
            description,
            category,
            price,
            -- Cosine similarity (1 = identical, 0 = unrelated, -1 = opposite)
            1 - (text_embedding <=> %s::vector) AS similarity
        FROM products
        ORDER BY text_embedding <=> %s::vector
        LIMIT %s
    """, (query_embedding, query_embedding, limit))
    
    results = []
    for row in cur.fetchall():
        results.append({
            'id': row[0],
            'name': row[1],
            'description': row[2],
            'category': row[3],
            'price': float(row[4]),
            'similarity': float(row[5])
        })
    
    cur.close()
    conn.close()
    return results

# ตัวอย่างการใช้งาน
results = semantic_search("waterproof running shoes for rainy weather")
for r in results:
    print(f"{r['name']} (similarity: {r['similarity']:.3f}) - ฿{r['price']}")
```

### Product Recommendations

```python
# recommendations.py
import psycopg2
from typing import Optional

def get_recommendations(
    user_id: int, 
    limit: int = 10,
    exclude_purchased: bool = True
) -> list[dict]:
    """แนะนำสินค้าสำหรับ user โดยใช้ vector similarity"""
    
    conn = psycopg2.connect("postgresql://localhost/ecommerce")
    cur = conn.cursor()
    
    query = """
        WITH user_vector AS (
            SELECT preference_vector 
            FROM users 
            WHERE id = %s
        ),
        purchased_items AS (
            SELECT DISTINCT product_id 
            FROM orders 
            WHERE user_id = %s
        )
        SELECT 
            p.id,
            p.name,
            p.category,
            p.price,
            1 - (p.text_embedding <=> uv.preference_vector) AS score
        FROM products p
        CROSS JOIN user_vector uv
        WHERE p.text_embedding IS NOT NULL
    """
    
    params = [user_id, user_id]
    
    if exclude_purchased:
        query += " AND p.id NOT IN (SELECT product_id FROM purchased_items)"
    
    query += " ORDER BY p.text_embedding <=> uv.preference_vector LIMIT %s"
    params.append(limit)
    
    cur.execute(query, params)
    
    results = [
        {'id': r[0], 'name': r[1], 'category': r[2], 
         'price': float(r[3]), 'score': float(r[4])}
        for r in cur.fetchall()
    ]
    
    cur.close()
    conn.close()
    return results

def update_user_preferences(user_id: int, viewed_product_ids: list[int]):
    """อัปเดต user preference vector จาก browsing history"""
    
    conn = psycopg2.connect("postgresql://localhost/ecommerce")
    cur = conn.cursor()
    
    # Compute average embedding ของ products ที่ user ดู
    cur.execute("""
        UPDATE users
        SET preference_vector = (
            SELECT AVG(text_embedding)
            FROM products
            WHERE id = ANY(%s)
        ),
        updated_at = CURRENT_TIMESTAMP
        WHERE id = %s
    """, (viewed_product_ids, user_id))
    
    conn.commit()
    cur.close()
    conn.close()
```

### User Similarity

```sql
-- หา users ที่มี preference คล้ายกัน (collaborative filtering)
SELECT 
    u2.id,
    u2.username,
    1 - (u1.preference_vector <=> u2.preference_vector) AS similarity
FROM users u1
CROSS JOIN users u2
WHERE u1.id = 42  -- user ที่เราสนใจ
    AND u2.id != 42
    AND u2.preference_vector IS NOT NULL
ORDER BY u1.preference_vector <=> u2.preference_vector
LIMIT 10;

-- แนะนำสินค้าจาก users ที่คล้ายกัน
WITH similar_users AS (
    SELECT 
        u2.id AS similar_user_id,
        1 - (u1.preference_vector <=> u2.preference_vector) AS similarity
    FROM users u1
    CROSS JOIN users u2
    WHERE u1.id = 42 AND u2.id != 42
    ORDER BY similarity DESC
    LIMIT 20
),
similar_user_purchases AS (
    SELECT 
        o.product_id,
        SUM(su.similarity) AS weighted_score
    FROM orders o
    JOIN similar_users su ON o.user_id = su.similar_user_id
    WHERE o.product_id NOT IN (
        SELECT product_id FROM orders WHERE user_id = 42
    )
    GROUP BY o.product_id
)
SELECT 
    p.id,
    p.name,
    p.category,
    p.price,
    sup.weighted_score
FROM products p
JOIN similar_user_purchases sup ON p.id = sup.product_id
ORDER BY sup.weighted_score DESC
LIMIT 10;
```

---

## 3. MADlib: ML ใน PostgreSQL

### ติดตั้ง MADlib

```bash
# ติดตั้ง MADlib
# ดาวน์โหลดจาก https://madlib.apache.org/
sudo apt-get install -y postgresql-madlib

# หรือ compile
git clone https://github.com/apache/madlib.git
cd madlib
mkdir build && cd build
cmake .. -DPOSTGRESQL_EXECUTABLE=/usr/bin/pg_config
make -j4
sudo make install

# ติดตั้ง extension
psql -c "CREATE EXTENSION madlib;"
```

### Linear Regression

```sql
-- ทำนาย sales ด้วย Linear Regression
-- Step 1: สร้าง training data
CREATE TABLE sales_training AS
SELECT
    net_amount AS sales,
    ARRAY[
        1.0,                                    -- intercept
        EXTRACT(MONTH FROM sale_timestamp),     -- month
        EXTRACT(DOW FROM sale_timestamp),       -- day of week
        quantity,
        unit_price,
        CASE WHEN discount_amount > 0 THEN 1 ELSE 0 END  -- has_discount
    ] AS features
FROM warehouse.sales_fact
WHERE sale_timestamp >= '2023-01-01'
    AND sale_timestamp < '2024-01-01';

-- Step 2: Train model
SELECT madlib.linregr_train(
    'sales_training',      -- source table
    'sales_model',         -- output table
    'sales',               -- dependent variable
    'features'             -- independent variables
);

-- Step 3: ดูผลลัพธ์
SELECT (madlib.linregr_predict(ARRAY[
    1.0, 6, 5, 3, 999.00, 1
], coef))::NUMERIC(10,2) AS predicted_sales
FROM sales_model;

-- ดู coefficients
SELECT 
    UNNEST(coef) AS coefficient,
    GENERATE_SUBSCRIPTS(coef, 1) AS feature_index
FROM sales_model;
```

### Logistic Regression สำหรับ Churn Prediction

```sql
-- สร้าง feature table สำหรับ churn prediction
CREATE TABLE churn_features AS
SELECT
    c.customer_key,
    -- Features
    EXTRACT(DAY FROM CURRENT_TIMESTAMP - MAX(s.sale_timestamp)) AS days_since_last_order,
    COUNT(*) AS total_orders,
    SUM(s.net_amount) AS lifetime_value,
    AVG(s.net_amount) AS avg_order_value,
    COUNT(*) FILTER (WHERE s.sale_timestamp >= CURRENT_TIMESTAMP - INTERVAL '90 days') AS orders_last_90_days,
    COUNT(*) FILTER (WHERE s.sale_timestamp >= CURRENT_TIMESTAMP - INTERVAL '30 days') AS orders_last_30_days,
    -- Label: churned = ไม่ซื้อใน 90 วัน
    CASE WHEN MAX(s.sale_timestamp) < CURRENT_TIMESTAMP - INTERVAL '90 days' THEN 1 ELSE 0 END AS churned
FROM warehouse.customer_dim c
LEFT JOIN warehouse.sales_fact s ON c.customer_key = s.customer_key
WHERE c.is_current = TRUE
GROUP BY c.customer_key;

-- Train Logistic Regression
SELECT madlib.logregr_train(
    'churn_features',
    'churn_model',
    'churned',
    'ARRAY[1.0, days_since_last_order, total_orders, lifetime_value, avg_order_value, orders_last_90_days]',
    NULL,           -- grouping columns
    20,             -- max iterations
    'irls'          -- optimizer
);

-- Predict churn probability
SELECT 
    customer_key,
    madlib.logregr_predict_prob(
        coef,
        ARRAY[1.0, days_since_last_order, total_orders, lifetime_value, avg_order_value, orders_last_90_days]
    ) AS churn_probability
FROM churn_features, churn_model
ORDER BY churn_probability DESC
LIMIT 100;
```

### K-Means Clustering

```sql
-- Customer segmentation ด้วย K-Means
-- สร้าง feature matrix
CREATE TABLE customer_features AS
SELECT
    customer_key,
    ARRAY[
        COALESCE(total_orders, 0)::FLOAT,
        COALESCE(lifetime_value, 0)::FLOAT,
        COALESCE(avg_order_value, 0)::FLOAT,
        COALESCE(days_since_last_order, 365)::FLOAT
    ] AS features
FROM churn_features;

-- Run K-Means (5 clusters)
SELECT madlib.kmeans(
    'customer_features',   -- source table
    'features',            -- feature column
    5,                     -- k = number of clusters
    'madlib.squared_dist_norm2',  -- distance function
    'madlib.avg',          -- centroid function
    20,                    -- max iterations
    0.001                  -- convergence threshold
);

-- Assign customers to clusters
SELECT 
    cf.customer_key,
    madlib.closest_column(
        centroids,
        cf.features
    ) AS cluster_id
FROM customer_features cf, (
    SELECT centroids FROM km_result
) AS km
LIMIT 100;
```

---

## 4. PL/Python: Python ML ใน PostgreSQL

### ติดตั้งและ Setup

```sql
-- ติดตั้ง plpython3u
-- Ubuntu: sudo apt-get install postgresql-plpython3-16

CREATE EXTENSION plpython3u;

-- ตรวจสอบ
DO $$
import sys
plpy.notice(f"Python version: {sys.version}")
$$ LANGUAGE plpython3u;
```

### Inference ด้วย scikit-learn ใน PostgreSQL

```sql
-- Function สำหรับ predict churn ด้วย scikit-learn model
CREATE OR REPLACE FUNCTION predict_churn(
    days_since_last_order FLOAT,
    total_orders FLOAT,
    lifetime_value FLOAT,
    avg_order_value FLOAT,
    orders_last_90_days FLOAT
)
RETURNS FLOAT
LANGUAGE plpython3u
AS $$
import pickle
import numpy as np

# Load model from file (หรือ cache ใน GD)
if 'churn_model' not in GD:
    with open('/models/churn_model.pkl', 'rb') as f:
        GD['churn_model'] = pickle.load(f)

model = GD['churn_model']

features = np.array([[
    days_since_last_order,
    total_orders,
    lifetime_value,
    avg_order_value,
    orders_last_90_days
]])

# Return probability of churn (class 1)
proba = model.predict_proba(features)[0][1]
return float(proba)
$$;

-- ใช้งาน function
SELECT 
    customer_key,
    days_since_last_order,
    predict_churn(
        days_since_last_order,
        total_orders,
        lifetime_value,
        avg_order_value,
        orders_last_90_days
    ) AS churn_probability
FROM churn_features
ORDER BY churn_probability DESC
LIMIT 20;
```

```python
# train_and_save_model.py
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import train_test_split
from sklearn.preprocessing import StandardScaler
from sklearn.pipeline import Pipeline
import psycopg2
import pandas as pd
import pickle

# Load data จาก PostgreSQL
conn = psycopg2.connect("postgresql://localhost/sales_dw")
df = pd.read_sql("""
    SELECT 
        days_since_last_order,
        total_orders,
        lifetime_value,
        avg_order_value,
        orders_last_90_days,
        churned
    FROM churn_features
    WHERE total_orders > 0
""", conn)
conn.close()

# Prepare features
X = df.drop('churned', axis=1)
y = df['churned']

X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

# Train model
pipeline = Pipeline([
    ('scaler', StandardScaler()),
    ('model', RandomForestClassifier(n_estimators=100, random_state=42))
])

pipeline.fit(X_train, y_train)

# Evaluate
from sklearn.metrics import classification_report, roc_auc_score
y_pred = pipeline.predict(X_test)
y_proba = pipeline.predict_proba(X_test)[:, 1]

print(classification_report(y_test, y_pred))
print(f"AUC-ROC: {roc_auc_score(y_test, y_proba):.4f}")

# Save model
with open('/models/churn_model.pkl', 'wb') as f:
    pickle.dump(pipeline, f)

print("Model saved to /models/churn_model.pkl")
```

---

## 5. External ML Service Integration

### FastAPI ML Service

```python
# ml_service/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import pickle
import numpy as np
import redis
import json
import hashlib

app = FastAPI(title="ML Scoring Service")

# Load model at startup
with open('/models/churn_model.pkl', 'rb') as f:
    model = pickle.load(f)

# Redis for prediction caching
redis_client = redis.Redis(host='redis', port=6379, decode_responses=True)

class ChurnRequest(BaseModel):
    customer_id: int
    days_since_last_order: float
    total_orders: float
    lifetime_value: float
    avg_order_value: float
    orders_last_90_days: float

class ChurnResponse(BaseModel):
    customer_id: int
    churn_probability: float
    risk_level: str
    from_cache: bool

@app.post("/predict/churn", response_model=ChurnResponse)
async def predict_churn(request: ChurnRequest):
    # Cache key
    cache_key = f"churn:{request.customer_id}"
    
    # Check cache (TTL = 1 hour)
    cached = redis_client.get(cache_key)
    if cached:
        data = json.loads(cached)
        return ChurnResponse(
            customer_id=request.customer_id,
            churn_probability=data['probability'],
            risk_level=data['risk_level'],
            from_cache=True
        )
    
    # Compute prediction
    features = np.array([[
        request.days_since_last_order,
        request.total_orders,
        request.lifetime_value,
        request.avg_order_value,
        request.orders_last_90_days
    ]])
    
    probability = float(model.predict_proba(features)[0][1])
    
    # Determine risk level
    if probability >= 0.7:
        risk_level = "HIGH"
    elif probability >= 0.4:
        risk_level = "MEDIUM"
    else:
        risk_level = "LOW"
    
    # Cache result
    redis_client.setex(
        cache_key,
        3600,  # TTL: 1 hour
        json.dumps({'probability': probability, 'risk_level': risk_level})
    )
    
    return ChurnResponse(
        customer_id=request.customer_id,
        churn_probability=probability,
        risk_level=risk_level,
        from_cache=False
    )

@app.get("/health")
async def health():
    return {"status": "ok", "model": "churn_v1"}
```

```dockerfile
# Dockerfile สำหรับ ML service
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY . .
COPY /models /models

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

```yaml
# docker-compose.yml
version: '3.8'
services:
  ml-service:
    build: ./ml_service
    ports:
      - "8000:8000"
    volumes:
      - ./models:/models
    environment:
      - REDIS_HOST=redis
    depends_on:
      - redis
  
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    command: redis-server --maxmemory 2gb --maxmemory-policy allkeys-lru
```

### Call ML Service จาก Application

```typescript
// src/services/mlService.ts
import axios from 'axios';
import { createClient } from 'redis';

const mlServiceUrl = process.env.ML_SERVICE_URL || 'http://ml-service:8000';
const redisClient = createClient({ url: process.env.REDIS_URL });

interface ChurnPrediction {
  customer_id: number;
  churn_probability: number;
  risk_level: 'LOW' | 'MEDIUM' | 'HIGH';
  from_cache: boolean;
}

export async function getChurnPrediction(
  customerId: number,
  features: {
    daysSinceLastOrder: number;
    totalOrders: number;
    lifetimeValue: number;
    avgOrderValue: number;
    ordersLast90Days: number;
  }
): Promise<ChurnPrediction> {
  try {
    const response = await axios.post<ChurnPrediction>(
      `${mlServiceUrl}/predict/churn`,
      {
        customer_id: customerId,
        days_since_last_order: features.daysSinceLastOrder,
        total_orders: features.totalOrders,
        lifetime_value: features.lifetimeValue,
        avg_order_value: features.avgOrderValue,
        orders_last_90_days: features.ordersLast90Days
      },
      {
        timeout: 500  // 500ms timeout
      }
    );
    
    return response.data;
  } catch (error) {
    // Fallback: return default LOW risk if ML service is down
    console.error('ML service error:', error);
    return {
      customer_id: customerId,
      churn_probability: 0,
      risk_level: 'LOW',
      from_cache: false
    };
  }
}
```

---

## 6. Feature Store

### Feast: Feature Store

```yaml
# feature_store.yaml
project: ecommerce_features
registry: data/registry.db
provider: local

online_store:
  type: redis
  connection_string: "redis://redis:6379"

offline_store:
  type: postgres
  host: localhost
  port: 5432
  database: feature_store
  user: postgres
  password: ${POSTGRES_PASSWORD}
```

```python
# features/customer_features.py
from feast import Entity, Feature, FeatureView, ValueType
from feast.infra.offline_stores.postgres import PostgreSQLSource
from datetime import timedelta

# Define entity
customer = Entity(
    name="customer_id",
    value_type=ValueType.INT64,
    description="Customer identifier"
)

# Define data source
customer_stats_source = PostgreSQLSource(
    query="""
        SELECT
            customer_key AS customer_id,
            days_since_last_order,
            total_orders,
            lifetime_value,
            avg_order_value,
            orders_last_90_days,
            orders_last_30_days,
            CURRENT_TIMESTAMP AS event_timestamp
        FROM churn_features
    """,
    timestamp_field="event_timestamp",
    created_timestamp_column="event_timestamp"
)

# Define Feature View
customer_stats_fv = FeatureView(
    name="customer_stats",
    entities=["customer_id"],
    ttl=timedelta(days=1),
    features=[
        Feature(name="days_since_last_order", dtype=ValueType.FLOAT),
        Feature(name="total_orders", dtype=ValueType.INT64),
        Feature(name="lifetime_value", dtype=ValueType.DOUBLE),
        Feature(name="avg_order_value", dtype=ValueType.DOUBLE),
        Feature(name="orders_last_90_days", dtype=ValueType.INT64),
        Feature(name="orders_last_30_days", dtype=ValueType.INT64),
    ],
    batch_source=customer_stats_source,
    online=True
)
```

```python
# feast_operations.py
from feast import FeatureStore
import pandas as pd

store = FeatureStore(repo_path=".")

# Materialize features to online store (Redis)
from datetime import datetime
store.materialize_incremental(end_date=datetime.utcnow())

# Get online features (real-time, low latency)
online_features = store.get_online_features(
    features=[
        "customer_stats:days_since_last_order",
        "customer_stats:total_orders",
        "customer_stats:lifetime_value",
        "customer_stats:avg_order_value",
        "customer_stats:orders_last_90_days",
    ],
    entity_rows=[
        {"customer_id": 1001},
        {"customer_id": 1002},
        {"customer_id": 1003},
    ]
)

print(online_features.to_df())

# Get historical features (for training)
entity_df = pd.DataFrame({
    "customer_id": [1001, 1002, 1003],
    "event_timestamp": pd.to_datetime(["2024-01-15", "2024-01-15", "2024-01-15"])
})

training_df = store.get_historical_features(
    entity_df=entity_df,
    features=[
        "customer_stats:days_since_last_order",
        "customer_stats:total_orders",
        "customer_stats:lifetime_value",
    ]
).to_df()
```

---

## 7. Real-time ML Pipeline

### Event-Driven Architecture

```yaml
# docker-compose-ml-pipeline.yml
version: '3.8'
services:
  kafka:
    image: confluentinc/cp-kafka:7.5.0
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
    depends_on:
      - zookeeper
  
  feature-extractor:
    build: ./feature_extractor
    environment:
      KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      REDIS_URL: redis://redis:6379
      POSTGRES_URL: postgresql://postgres:5432/ecommerce
    depends_on:
      - kafka
      - redis
      - postgres
  
  ml-scorer:
    build: ./ml_scorer
    environment:
      KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      REDIS_URL: redis://redis:6379
      MODEL_PATH: /models/churn_model.pkl
    depends_on:
      - kafka
      - redis
```

```python
# feature_extractor/main.py
from kafka import KafkaConsumer, KafkaProducer
import json
import psycopg2
import redis
import logging

consumer = KafkaConsumer(
    'user-events',
    bootstrap_servers=['kafka:9092'],
    value_deserializer=lambda m: json.loads(m.decode('utf-8'))
)

producer = KafkaProducer(
    bootstrap_servers=['kafka:9092'],
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

redis_client = redis.Redis(host='redis', port=6379, decode_responses=True)
pg_conn = psycopg2.connect("postgresql://postgres/ecommerce")

def extract_features(customer_id: int) -> dict:
    """Extract ML features สำหรับ customer"""
    
    # Check cache ก่อน
    cached = redis_client.get(f"features:{customer_id}")
    if cached:
        return json.loads(cached)
    
    # Query features จาก PostgreSQL
    cur = pg_conn.cursor()
    cur.execute("""
        SELECT
            EXTRACT(DAY FROM CURRENT_TIMESTAMP - MAX(sale_timestamp)) AS days_since_last_order,
            COUNT(*) AS total_orders,
            SUM(net_amount) AS lifetime_value,
            AVG(net_amount) AS avg_order_value,
            COUNT(*) FILTER (
                WHERE sale_timestamp >= CURRENT_TIMESTAMP - INTERVAL '90 days'
            ) AS orders_last_90_days
        FROM warehouse.sales_fact
        WHERE customer_key = %s
    """, (customer_id,))
    
    row = cur.fetchone()
    cur.close()
    
    features = {
        'customer_id': customer_id,
        'days_since_last_order': float(row[0] or 365),
        'total_orders': int(row[1] or 0),
        'lifetime_value': float(row[2] or 0),
        'avg_order_value': float(row[3] or 0),
        'orders_last_90_days': int(row[4] or 0)
    }
    
    # Cache features (TTL = 10 minutes)
    redis_client.setex(
        f"features:{customer_id}",
        600,
        json.dumps(features)
    )
    
    return features

for message in consumer:
    event = message.value
    customer_id = event.get('customer_id')
    
    if customer_id:
        features = extract_features(customer_id)
        # ส่ง features ไปยัง scoring topic
        producer.send('scoring-requests', features)
        logging.info(f"Extracted features for customer {customer_id}")
```

```python
# ml_scorer/main.py
from kafka import KafkaConsumer, KafkaProducer
import json
import pickle
import numpy as np
import redis

with open('/models/churn_model.pkl', 'rb') as f:
    model = pickle.load(f)

redis_client = redis.Redis(host='redis', port=6379, decode_responses=True)

consumer = KafkaConsumer(
    'scoring-requests',
    bootstrap_servers=['kafka:9092'],
    value_deserializer=lambda m: json.loads(m.decode('utf-8'))
)

producer = KafkaProducer(
    bootstrap_servers=['kafka:9092'],
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

for message in consumer:
    features = message.value
    customer_id = features['customer_id']
    
    # Score
    X = np.array([[
        features['days_since_last_order'],
        features['total_orders'],
        features['lifetime_value'],
        features['avg_order_value'],
        features['orders_last_90_days']
    ]])
    
    probability = float(model.predict_proba(X)[0][1])
    
    # Store score ใน Redis
    redis_client.setex(
        f"churn_score:{customer_id}",
        86400,  # TTL: 24 hours
        str(probability)
    )
    
    # Publish result
    result = {
        'customer_id': customer_id,
        'churn_probability': probability,
        'timestamp': features.get('timestamp')
    }
    
    producer.send('scoring-results', result)
    
    # Alert ถ้า high risk
    if probability > 0.7:
        producer.send('high-churn-risk', result)
```

---

## 8. MLflow: Experiment Tracking

```python
# train_with_mlflow.py
import mlflow
import mlflow.sklearn
from sklearn.ensemble import RandomForestClassifier, GradientBoostingClassifier
from sklearn.model_selection import cross_val_score
from sklearn.metrics import roc_auc_score
import psycopg2
import pandas as pd

# ตั้งค่า MLflow tracking server
mlflow.set_tracking_uri("http://mlflow:5000")
mlflow.set_experiment("churn_prediction")

# Load data
conn = psycopg2.connect("postgresql://localhost/sales_dw")
df = pd.read_sql("SELECT * FROM churn_features", conn)
conn.close()

X = df[['days_since_last_order', 'total_orders', 'lifetime_value', 
         'avg_order_value', 'orders_last_90_days']]
y = df['churned']

# Experiment 1: Random Forest
with mlflow.start_run(run_name="random_forest_v1"):
    params = {
        'n_estimators': 100,
        'max_depth': 10,
        'min_samples_split': 5,
        'random_state': 42
    }
    
    mlflow.log_params(params)
    
    model = RandomForestClassifier(**params)
    
    # Cross validation
    cv_scores = cross_val_score(model, X, y, cv=5, scoring='roc_auc')
    mlflow.log_metric("cv_auc_mean", cv_scores.mean())
    mlflow.log_metric("cv_auc_std", cv_scores.std())
    
    # Train final model
    model.fit(X, y)
    
    # Log model
    mlflow.sklearn.log_model(model, "model")
    
    # Feature importance
    for feat, imp in zip(X.columns, model.feature_importances_):
        mlflow.log_metric(f"feat_importance_{feat}", imp)
    
    print(f"RF AUC: {cv_scores.mean():.4f} ± {cv_scores.std():.4f}")

# ดู experiments
experiments = mlflow.search_runs()
best_run = experiments.sort_values('metrics.cv_auc_mean', ascending=False).iloc[0]
print(f"Best model: {best_run['run_id']}, AUC: {best_run['metrics.cv_auc_mean']:.4f}")

# Register best model
client = mlflow.tracking.MlflowClient()
result = mlflow.register_model(
    f"runs:/{best_run['run_id']}/model",
    "churn_model"
)
client.transition_model_version_stage(
    name="churn_model",
    version=result.version,
    stage="Production"
)
```

---

## 9. LLM Integration

### Store Conversation History ใน PostgreSQL

```sql
-- Conversation history table
CREATE TABLE conversations (
    id              BIGSERIAL PRIMARY KEY,
    conversation_id UUID NOT NULL DEFAULT gen_random_uuid(),
    user_id         INTEGER,
    role            VARCHAR(20) NOT NULL,   -- 'user' or 'assistant'
    content         TEXT NOT NULL,
    token_count     INTEGER,
    model           VARCHAR(50),
    -- Embedding ของ message สำหรับ semantic search
    embedding       VECTOR(1536),
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_conv_conversation_id ON conversations(conversation_id);
CREATE INDEX idx_conv_user_id ON conversations(user_id);
CREATE INDEX idx_conv_embedding ON conversations USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100);

-- Knowledge base สำหรับ RAG
CREATE TABLE knowledge_base (
    id              BIGSERIAL PRIMARY KEY,
    title           VARCHAR(500),
    content         TEXT NOT NULL,
    source          VARCHAR(200),
    chunk_index     INTEGER DEFAULT 0,
    embedding       VECTOR(1536) NOT NULL,
    metadata        JSONB,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_kb_embedding ON knowledge_base USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64);
```

### Vector Search สำหรับ RAG

```python
# rag_pipeline.py
import psycopg2
import openai
from typing import Optional
import json

client = openai.OpenAI()

def get_embedding(text: str) -> list[float]:
    """Get embedding จาก OpenAI"""
    response = client.embeddings.create(
        model="text-embedding-ada-002",
        input=text
    )
    return response.data[0].embedding

def search_knowledge_base(
    query: str,
    limit: int = 5,
    similarity_threshold: float = 0.7
) -> list[dict]:
    """ค้นหาข้อมูลที่เกี่ยวข้องจาก knowledge base"""
    
    query_embedding = get_embedding(query)
    
    conn = psycopg2.connect("postgresql://localhost/chatbot_db")
    cur = conn.cursor()
    
    cur.execute("""
        SELECT 
            id,
            title,
            content,
            source,
            1 - (embedding <=> %s::vector) AS similarity
        FROM knowledge_base
        WHERE 1 - (embedding <=> %s::vector) > %s
        ORDER BY embedding <=> %s::vector
        LIMIT %s
    """, (query_embedding, query_embedding, similarity_threshold, 
          query_embedding, limit))
    
    results = []
    for row in cur.fetchall():
        results.append({
            'id': row[0],
            'title': row[1],
            'content': row[2],
            'source': row[3],
            'similarity': float(row[4])
        })
    
    cur.close()
    conn.close()
    return results

def get_conversation_history(conversation_id: str, limit: int = 10) -> list[dict]:
    """ดึง conversation history"""
    
    conn = psycopg2.connect("postgresql://localhost/chatbot_db")
    cur = conn.cursor()
    
    cur.execute("""
        SELECT role, content
        FROM conversations
        WHERE conversation_id = %s
        ORDER BY created_at DESC
        LIMIT %s
    """, (conversation_id, limit))
    
    messages = [{'role': r[0], 'content': r[1]} for r in reversed(cur.fetchall())]
    
    cur.close()
    conn.close()
    return messages

def save_message(
    conversation_id: str,
    role: str,
    content: str,
    user_id: Optional[int] = None
):
    """บันทึก message ลงฐานข้อมูล"""
    
    embedding = get_embedding(content)
    
    conn = psycopg2.connect("postgresql://localhost/chatbot_db")
    cur = conn.cursor()
    
    cur.execute("""
        INSERT INTO conversations (conversation_id, user_id, role, content, embedding)
        VALUES (%s, %s, %s, %s, %s::vector)
    """, (conversation_id, user_id, role, content, embedding))
    
    conn.commit()
    cur.close()
    conn.close()
```

### Full ChatBot with Memory และ RAG

```python
# chatbot.py
import uuid
from typing import Optional
from openai import OpenAI

client = OpenAI()

def chat(
    message: str,
    conversation_id: Optional[str] = None,
    user_id: Optional[int] = None
) -> dict:
    """
    ChatBot ที่มี:
    1. Memory: จำ conversation ก่อนหน้า
    2. RAG: ค้นหาข้อมูลที่เกี่ยวข้อง
    """
    
    # สร้าง conversation ID ถ้ายังไม่มี
    if not conversation_id:
        conversation_id = str(uuid.uuid4())
    
    # ดึง conversation history
    history = get_conversation_history(conversation_id)
    
    # ค้นหาข้อมูลที่เกี่ยวข้องจาก knowledge base (RAG)
    relevant_docs = search_knowledge_base(message, limit=3)
    
    # สร้าง context จาก relevant docs
    context = ""
    if relevant_docs:
        context = "\n\nข้อมูลที่เกี่ยวข้อง:\n"
        for doc in relevant_docs:
            context += f"\n---\nแหล่งที่มา: {doc['source']}\n{doc['content']}\n"
    
    # สร้าง system message
    system_message = f"""คุณเป็น AI assistant ที่ช่วยตอบคำถามเกี่ยวกับผลิตภัณฑ์และบริการของเรา
    
ตอบเป็นภาษาไทย สั้นกระชับ และเป็นประโยชน์
{context}"""
    
    # Build messages array
    messages = [{"role": "system", "content": system_message}]
    messages.extend(history)
    messages.append({"role": "user", "content": message})
    
    # Call OpenAI API
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=messages,
        temperature=0.7,
        max_tokens=500
    )
    
    assistant_message = response.choices[0].message.content
    
    # บันทึก messages ลง PostgreSQL
    save_message(conversation_id, "user", message, user_id)
    save_message(conversation_id, "assistant", assistant_message, user_id)
    
    return {
        "conversation_id": conversation_id,
        "message": assistant_message,
        "sources": [doc['source'] for doc in relevant_docs],
        "tokens_used": response.usage.total_tokens
    }

# ตัวอย่างการใช้งาน
if __name__ == "__main__":
    # สนทนาครั้งแรก
    result = chat("สินค้าพวกรองเท้าวิ่ง คุณมีอะไรบ้าง?")
    conv_id = result["conversation_id"]
    print(f"Bot: {result['message']}")
    
    # ต่อสนทนา
    result = chat("ราคาเท่าไหร่?", conversation_id=conv_id)
    print(f"Bot: {result['message']}")
```

### Index Documents เข้า Knowledge Base

```python
# index_knowledge_base.py
import os
import glob
import re
import psycopg2
from openai import OpenAI

client = OpenAI()

def chunk_text(text: str, chunk_size: int = 500, overlap: int = 50) -> list[str]:
    """แบ่ง text เป็น chunks"""
    words = text.split()
    chunks = []
    
    for i in range(0, len(words), chunk_size - overlap):
        chunk = ' '.join(words[i:i + chunk_size])
        chunks.append(chunk)
    
    return chunks

def index_document(file_path: str, source: str):
    """Index document เข้า knowledge base"""
    
    with open(file_path, 'r', encoding='utf-8') as f:
        content = f.read()
    
    # Extract title จาก filename
    title = os.path.basename(file_path).replace('.md', '').replace('-', ' ').title()
    
    # แบ่งเป็น chunks
    chunks = chunk_text(content)
    
    conn = psycopg2.connect("postgresql://localhost/chatbot_db")
    cur = conn.cursor()
    
    for i, chunk in enumerate(chunks):
        # Get embedding
        response = client.embeddings.create(
            model="text-embedding-ada-002",
            input=chunk
        )
        embedding = response.data[0].embedding
        
        cur.execute("""
            INSERT INTO knowledge_base (title, content, source, chunk_index, embedding)
            VALUES (%s, %s, %s, %s, %s::vector)
        """, (title, chunk, source, i, embedding))
    
    conn.commit()
    cur.close()
    conn.close()
    
    print(f"Indexed {len(chunks)} chunks from {file_path}")

# Index ทุก markdown files
for md_file in glob.glob('./docs/**/*.md', recursive=True):
    index_document(md_file, md_file)
```

---

## สรุป

การผสาน ML เข้ากับ Database Cluster ทำให้ระบบมีความสามารถ:

1. **pgvector** - Store และค้นหา embeddings สำหรับ semantic search และ recommendations
2. **MADlib** - Run ML algorithms โดยตรงใน PostgreSQL
3. **PL/Python** - ใช้ scikit-learn/PyTorch ภายใน PostgreSQL
4. **External ML Services** - FastAPI + Redis caching สำหรับ real-time scoring
5. **Feature Store** - Feast จัดการ features สำหรับ training และ serving
6. **Real-time Pipeline** - Kafka-based event-driven ML scoring
7. **MLflow** - Track experiments และจัดการ model registry
8. **LLM Integration** - ChatBot พร้อม memory และ RAG ด้วย pgvector

Key principle: **เก็บข้อมูลและ features ใน PostgreSQL, ทำ inference ใน Python, cache ด้วย Redis**

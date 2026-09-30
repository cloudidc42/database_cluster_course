# Part 85: Analytics Pipeline

## บทนำ

Analytics Pipeline คือระบบที่นำข้อมูลดิบ (raw data) ผ่านกระบวนการหลายขั้นตอนจนกลายเป็น insights ที่มีคุณค่าสำหรับธุรกิจ บทนี้จะครอบคลุมทุก component ตั้งแต่ dbt สำหรับ data transformation, Metabase สำหรับ BI, ไปจนถึง real-time dashboards

---

## 1. Analytics Pipeline Overview

### 1.1 Pipeline Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         ANALYTICS PIPELINE                               │
│                                                                          │
│  1. SOURCES        2. INGESTION      3. STORAGE        4. TRANSFORM     │
│  ┌──────────┐     ┌────────────┐    ┌────────────┐    ┌────────────┐   │
│  │PostgreSQL│ →   │  Debezium  │ →  │  Data Lake │ →  │    dbt     │   │
│  │  MySQL   │ →   │   Kafka    │    │  (S3/MinIO)│    │  (SQL-based│   │
│  │  APIs    │ →   │  Firehose  │    │  Data WH   │    │  transform)│   │
│  │  Logs    │ →   │  Lambda    │    │ (Redshift/ │    │            │   │
│  └──────────┘     └────────────┘    │  BigQuery) │    └────────────┘   │
│                                     └────────────┘                      │
│                                                                          │
│  5. SERVING        6. VISUALIZE     7. INSIGHTS                         │
│  ┌──────────┐     ┌────────────┐    ┌────────────┐                     │
│  │Analytics │ →   │  Metabase  │ →  │  Reports   │                     │
│  │  API     │     │  Superset  │    │  Dashboards│                     │
│  │ Redshift │     │  Grafana   │    │  Alerts    │                     │
│  │ClickHouse│     │  Tableau   │    │  ML Models │                     │
│  └──────────┘     └────────────┘    └────────────┘                     │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 เลือก Pipeline แบบไหน?

```
Batch Pipeline (ปกติ):
- ข้อมูลเคลื่อนที่เป็น "batches" (ชั่วโมง/วัน)
- ง่ายกว่า, cost-effective
- Use case: daily reports, monthly analysis
- Tools: Airflow + dbt

Streaming Pipeline (real-time):
- ข้อมูลเคลื่อนที่ continuous
- ซับซ้อนกว่า, แพงกว่า
- Use case: live dashboards, fraud detection
- Tools: Kafka + Flink/Spark Streaming

Lambda Architecture (ทั้งสอง):
- Batch layer: ประมวลผลข้อมูลทั้งหมด
- Speed layer: ประมวลผล recent data real-time
- Serving layer: merge results
- ซับซ้อนมาก, maintain ยาก
```

---

## 2. dbt (data build tool)

### 2.1 ทำไมต้องใช้ dbt?

```
ปัญหาก่อนมี dbt:
- นักวิเคราะห์เขียน SQL ใน Excel/Google Sheets
- ไม่มี version control
- ไม่มี documentation
- ไม่มี testing
- ไม่รู้ว่าข้อมูลมาจากไหน

dbt แก้ปัญหาเหล่านี้:
✅ SQL-based transformations (นักวิเคราะห์ไม่ต้องรู้ Python)
✅ Version control ด้วย Git
✅ Automatic documentation
✅ Data testing
✅ Dependency graph (lineage)
✅ Incremental models (ไม่ต้อง rebuild ทั้งหมด)
```

### 2.2 Installation

```bash
# Install dbt
pip install dbt-core dbt-postgres  # สำหรับ PostgreSQL
# หรือ
pip install dbt-core dbt-redshift   # สำหรับ Redshift
pip install dbt-core dbt-bigquery   # สำหรับ BigQuery
pip install dbt-core dbt-snowflake  # สำหรับ Snowflake
pip install dbt-core dbt-duckdb     # สำหรับ DuckDB

# Initialize project
dbt init my_analytics

# Project structure
my_analytics/
├── dbt_project.yml          # Project config
├── profiles.yml             # Database connections
├── models/
│   ├── staging/             # Stage raw data (1-to-1 with source)
│   │   ├── stg_orders.sql
│   │   ├── stg_users.sql
│   │   └── schema.yml       # Tests + documentation
│   ├── intermediate/        # Complex transformations
│   │   ├── int_order_items.sql
│   │   └── int_user_sessions.sql
│   └── marts/               # Final models for BI tools
│       ├── finance/
│       │   ├── fct_orders.sql
│       │   └── schema.yml
│       └── marketing/
│           ├── fct_user_cohorts.sql
│           └── dim_users.sql
├── seeds/                   # Static CSV data
│   └── country_codes.csv
├── tests/                   # Custom tests
│   └── assert_positive_revenue.sql
├── macros/                  # Reusable SQL snippets
│   └── clean_amount.sql
└── analyses/                # Ad-hoc analyses (not materialized)
    └── user_ltv.sql
```

### 2.3 profiles.yml

```yaml
# ~/.dbt/profiles.yml
my_analytics:
  outputs:
    dev:
      type: postgres
      host: localhost
      user: postgres
      password: "{{ env_var('DBT_DB_PASSWORD') }}"
      port: 5432
      dbname: analytics_dev
      schema: dbt_dev
      threads: 4
      keepalives_idle: 0
      
    prod:
      type: postgres
      host: "{{ env_var('PROD_DB_HOST') }}"
      user: "{{ env_var('PROD_DB_USER') }}"
      password: "{{ env_var('PROD_DB_PASSWORD') }}"
      port: 5432
      dbname: analytics
      schema: public
      threads: 8
      
  target: dev  # Default to dev
```

### 2.4 dbt_project.yml

```yaml
# dbt_project.yml
name: 'my_analytics'
version: '1.0.0'
config-version: 2

profile: 'my_analytics'

model-paths: ["models"]
seed-paths: ["seeds"]
test-paths: ["tests"]
macro-paths: ["macros"]

target-path: "target"
clean-targets: ["target", "dbt_packages"]

models:
  my_analytics:
    # Staging models: views (fast, no storage)
    staging:
      +materialized: view
      +schema: staging
      
    # Intermediate models: ephemeral (ไม่ materialze ใน DB)
    intermediate:
      +materialized: ephemeral
      
    # Mart models: tables (query-ready)
    marts:
      +materialized: table
      finance:
        +materialized: table
        +schema: finance
      marketing:
        +materialized: table
        +schema: marketing

    # Incremental models สำหรับ large tables
    marts/finance/fct_orders:
      +materialized: incremental
      +unique_key: order_id
      +incremental_strategy: merge
```

### 2.5 Models: SELECT Statements

```sql
-- models/staging/stg_orders.sql
-- Staging: clean + standardize raw data

{{
    config(
        materialized='view',
        schema='staging'
    )
}}

WITH source AS (
    SELECT * FROM {{ source('raw', 'orders') }}
),

renamed AS (
    SELECT
        id AS order_id,
        user_id,
        product_id,
        quantity,
        -- Clean amount
        COALESCE(amount, 0) AS amount,
        -- Standardize status
        LOWER(TRIM(status)) AS status,
        -- Parse dates
        created_at::TIMESTAMPTZ AS created_at,
        updated_at::TIMESTAMPTZ AS updated_at,
        -- Metadata
        _loaded_at
    FROM source
    WHERE id IS NOT NULL
)

SELECT * FROM renamed
```

```sql
-- models/staging/stg_users.sql
{{
    config(materialized='view')
}}

WITH source AS (
    SELECT * FROM {{ source('raw', 'users') }}
),

cleaned AS (
    SELECT
        id AS user_id,
        LOWER(TRIM(email)) AS email,
        TRIM(first_name) AS first_name,
        TRIM(last_name) AS last_name,
        COALESCE(country, 'Unknown') AS country,
        CASE
            WHEN LOWER(status) IN ('active', 'enabled') THEN 'active'
            WHEN LOWER(status) IN ('inactive', 'disabled', 'blocked') THEN 'inactive'
            ELSE 'unknown'
        END AS status,
        created_at::DATE AS registration_date,
        _loaded_at
    FROM source
    WHERE email IS NOT NULL
      AND email LIKE '%@%'  -- Basic email validation
)

SELECT * FROM cleaned
```

```sql
-- models/marts/finance/fct_orders.sql
-- Fact table: order transactions

{{
    config(
        materialized='incremental',
        unique_key='order_id',
        incremental_strategy='merge',
        on_schema_change='sync_all_columns'
    )
}}

WITH orders AS (
    SELECT * FROM {{ ref('stg_orders') }}
    {% if is_incremental() %}
    -- Only process new/updated records
    WHERE updated_at > (SELECT MAX(updated_at) FROM {{ this }})
    {% endif %}
),

users AS (
    SELECT * FROM {{ ref('stg_users') }}
),

products AS (
    SELECT * FROM {{ ref('stg_products') }}
),

enriched AS (
    SELECT
        o.order_id,
        o.user_id,
        o.product_id,
        o.quantity,
        o.amount,
        
        -- Calculated metrics
        o.amount / NULLIF(o.quantity, 0) AS unit_price,
        
        -- User attributes
        u.email AS user_email,
        u.country AS user_country,
        u.registration_date,
        
        -- Product attributes
        p.name AS product_name,
        p.category AS product_category,
        
        -- Date dimensions
        DATE(o.created_at) AS order_date,
        DATE_TRUNC('week', o.created_at) AS order_week,
        DATE_TRUNC('month', o.created_at) AS order_month,
        EXTRACT(HOUR FROM o.created_at)::INT AS order_hour,
        EXTRACT(DOW FROM o.created_at)::INT AS day_of_week,
        
        -- User vintage (days since registration)
        DATE(o.created_at) - u.registration_date AS days_since_registration,
        
        -- Order flags
        o.status,
        o.status = 'completed' AS is_completed,
        
        o.created_at,
        o.updated_at
    FROM orders o
    LEFT JOIN users u ON o.user_id = u.user_id
    LEFT JOIN products p ON o.product_id = p.product_id
)

SELECT * FROM enriched
```

```sql
-- models/marts/marketing/fct_user_cohorts.sql
-- Cohort analysis: user retention

{{ config(materialized='table') }}

WITH first_orders AS (
    SELECT
        user_id,
        DATE_TRUNC('month', MIN(order_date)) AS cohort_month
    FROM {{ ref('fct_orders') }}
    WHERE is_completed = TRUE
    GROUP BY user_id
),

monthly_orders AS (
    SELECT DISTINCT
        user_id,
        DATE_TRUNC('month', order_date) AS order_month
    FROM {{ ref('fct_orders') }}
    WHERE is_completed = TRUE
),

cohort_data AS (
    SELECT
        f.cohort_month,
        EXTRACT(YEAR FROM AGE(o.order_month, f.cohort_month))::INT * 12 +
        EXTRACT(MONTH FROM AGE(o.order_month, f.cohort_month))::INT AS period_number,
        COUNT(DISTINCT o.user_id) AS retained_users
    FROM first_orders f
    JOIN monthly_orders o ON f.user_id = o.user_id
    GROUP BY 1, 2
),

cohort_size AS (
    SELECT cohort_month, COUNT(*) AS cohort_size
    FROM first_orders
    GROUP BY cohort_month
)

SELECT
    c.cohort_month,
    c.period_number,
    c.retained_users,
    cs.cohort_size,
    ROUND(c.retained_users::NUMERIC / cs.cohort_size * 100, 2) AS retention_rate
FROM cohort_data c
JOIN cohort_size cs ON c.cohort_month = cs.cohort_month
ORDER BY c.cohort_month, c.period_number
```

### 2.6 Tests: Data Quality

```yaml
# models/staging/schema.yml
version: 2

sources:
  - name: raw
    database: analytics
    schema: raw
    tables:
      - name: orders
        loaded_at_field: _loaded_at
        freshness:
          warn_after: {count: 12, period: hour}
          error_after: {count: 24, period: hour}
        columns:
          - name: id
            tests:
              - unique
              - not_null
          - name: amount
            tests:
              - not_null
              - dbt_utils.accepted_range:
                  min_value: 0
                  max_value: 10000000

models:
  - name: stg_orders
    description: "Cleaned orders from raw PostgreSQL table"
    columns:
      - name: order_id
        description: "Unique order identifier"
        tests:
          - unique
          - not_null
      - name: user_id
        description: "Foreign key to users"
        tests:
          - not_null
          - relationships:
              to: ref('stg_users')
              field: user_id
      - name: amount
        description: "Order amount in THB"
        tests:
          - not_null
          - dbt_utils.accepted_range:
              min_value: 0
      - name: status
        tests:
          - accepted_values:
              values: ['pending', 'confirmed', 'completed', 'cancelled']
```

```sql
-- tests/assert_revenue_matches_orders.sql
-- Custom test: revenue ต้อง match กับ total of order items

SELECT
    o.order_id,
    o.amount as order_total,
    SUM(oi.unit_price * oi.quantity) as calculated_total,
    ABS(o.amount - SUM(oi.unit_price * oi.quantity)) as difference
FROM {{ ref('fct_orders') }} o
JOIN {{ ref('fct_order_items') }} oi ON o.order_id = oi.order_id
GROUP BY o.order_id, o.amount
HAVING ABS(o.amount - SUM(oi.unit_price * oi.quantity)) > 1.0  -- tolerance 1 THB
```

### 2.7 Macros: Reusable SQL

```sql
-- macros/generate_surrogate_key.sql
{% macro generate_surrogate_key(field_list) %}
    md5(cast(coalesce(
    {% for field in field_list %}
        {{ field }}
        {% if not loop.last %} || '-' || {% endif %}
    {% endfor %}
    , '') as varchar))
{% endmacro %}

-- ใช้งาน
SELECT
    {{ generate_surrogate_key(['order_id', 'product_id']) }} as order_item_key,
    order_id,
    product_id
FROM order_items
```

```sql
-- macros/date_spine.sql
-- สร้าง date sequence สำหรับ time-series
{% macro date_spine(start_date, end_date) %}
    SELECT generate_series(
        '{{ start_date }}'::DATE,
        '{{ end_date }}'::DATE,
        '1 day'::INTERVAL
    )::DATE AS date_day
{% endmacro %}

-- ใช้ใน model
WITH date_spine AS (
    {{ date_spine('2024-01-01', 'today()') }}
),
daily_orders AS (
    SELECT order_date, SUM(amount) as revenue
    FROM fct_orders GROUP BY 1
)
-- Fill gaps ด้วย 0 สำหรับวันที่ไม่มี orders
SELECT
    d.date_day,
    COALESCE(o.revenue, 0) as revenue
FROM date_spine d
LEFT JOIN daily_orders o ON d.date_day = o.order_date
```

### 2.8 dbt Commands

```bash
# Run all models
dbt run

# Run specific model
dbt run --select stg_orders
dbt run --select marts.finance+  # model + downstream

# Run tests
dbt test
dbt test --select stg_orders

# Generate documentation
dbt docs generate
dbt docs serve  # เปิด browser ดู lineage graph

# Seed: load CSV data
dbt seed

# Snapshot: SCD Type 2
dbt snapshot

# Check freshness
dbt source freshness

# Run + test ใน production
dbt build  # run + test + seed + snapshot
```

---

## 3. Metabase: Open-Source BI Tool

### 3.1 Installation ด้วย Docker

```yaml
# docker-compose-metabase.yml
version: '3.8'

services:
  metabase:
    image: metabase/metabase:v0.48.0
    container_name: metabase
    ports:
      - "3001:3000"
    environment:
      MB_DB_TYPE: postgres
      MB_DB_DBNAME: metabase
      MB_DB_PORT: 5432
      MB_DB_USER: metabase
      MB_DB_PASS: metabase_password
      MB_DB_HOST: postgres
      
      # Site settings
      MB_SITE_URL: https://analytics.example.com
      
      # Security
      MB_ENCRYPTION_SECRET_KEY: your-secret-key-here
      
      # Email
      MB_EMAIL_SMTP_HOST: smtp.gmail.com
      MB_EMAIL_SMTP_PORT: 587
      MB_EMAIL_SMTP_USERNAME: analytics@example.com
      MB_EMAIL_SMTP_PASSWORD: smtp_password
      
    volumes:
      - metabase_data:/metabase-data
    depends_on:
      - postgres

  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: metabase
      POSTGRES_USER: metabase
      POSTGRES_PASSWORD: metabase_password
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  metabase_data:
  postgres_data:
```

### 3.2 Metabase API (Automation)

```python
# metabase_api.py
import requests
import json

class MetabaseAPI:
    def __init__(self, base_url: str, username: str, password: str):
        self.base_url = base_url
        self.session = requests.Session()
        self._authenticate(username, password)
    
    def _authenticate(self, username: str, password: str):
        """Get session token"""
        response = self.session.post(
            f"{self.base_url}/api/session",
            json={"username": username, "password": password}
        )
        response.raise_for_status()
        token = response.json()['id']
        self.session.headers.update({'X-Metabase-Session': token})
    
    def create_dashboard(self, name: str, description: str = "") -> dict:
        """สร้าง dashboard ใหม่"""
        response = self.session.post(
            f"{self.base_url}/api/dashboard",
            json={
                "name": name,
                "description": description,
                "parameters": []
            }
        )
        response.raise_for_status()
        return response.json()
    
    def add_card_to_dashboard(self, dashboard_id: int, card_id: int, 
                               col: int = 0, row: int = 0,
                               size_x: int = 12, size_y: int = 6) -> dict:
        """เพิ่ม question card เข้า dashboard"""
        response = self.session.post(
            f"{self.base_url}/api/dashboard/{dashboard_id}/cards",
            json={
                "cardId": card_id,
                "col": col,
                "row": row,
                "size_x": size_x,
                "size_y": size_y
            }
        )
        response.raise_for_status()
        return response.json()
    
    def create_question(self, name: str, database_id: int, sql: str,
                         display: str = "table") -> dict:
        """สร้าง SQL question"""
        response = self.session.post(
            f"{self.base_url}/api/card",
            json={
                "name": name,
                "dataset_query": {
                    "type": "native",
                    "database": database_id,
                    "native": {
                        "query": sql,
                        "template-tags": {}
                    }
                },
                "display": display,
                "visualization_settings": {}
            }
        )
        response.raise_for_status()
        return response.json()
    
    def get_embed_url(self, dashboard_id: int, params: dict = {}) -> str:
        """Get signed embed URL"""
        import jwt
        
        payload = {
            "resource": {"dashboard": dashboard_id},
            "params": params,
            "exp": int(time.time()) + 600  # 10 minutes
        }
        
        token = jwt.encode(
            payload,
            self.embed_secret_key,
            algorithm="HS256"
        )
        
        return f"{self.base_url}/embed/dashboard/{token}#bordered=true&titled=true"

# ใช้งาน
mb = MetabaseAPI("http://localhost:3001", "admin@example.com", "password")

# สร้าง dashboard
dashboard = mb.create_dashboard("Revenue Dashboard", "Daily revenue overview")

# สร้าง question
revenue_card = mb.create_question(
    name="Daily Revenue (Last 30 Days)",
    database_id=1,
    sql="""
        SELECT 
            order_date,
            SUM(amount) as revenue
        FROM analytics.fct_orders
        WHERE order_date >= CURRENT_DATE - 30
          AND status = 'completed'
        GROUP BY order_date
        ORDER BY order_date
    """,
    display="line"
)

# เพิ่มเข้า dashboard
mb.add_card_to_dashboard(dashboard['id'], revenue_card['id'])
```

### 3.3 Embeddable Dashboards

```typescript
// frontend/src/components/EmbeddedDashboard.tsx
import React, { useEffect, useState } from 'react';

interface Props {
  dashboardId: number;
  params?: Record<string, any>;
}

export function EmbeddedDashboard({ dashboardId, params = {} }: Props) {
  const [embedUrl, setEmbedUrl] = useState<string | null>(null);
  
  useEffect(() => {
    // Backend API สร้าง signed URL
    fetch(`/api/analytics/embed-url`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ dashboardId, params })
    })
      .then(res => res.json())
      .then(data => setEmbedUrl(data.url));
  }, [dashboardId, JSON.stringify(params)]);
  
  if (!embedUrl) return <div>Loading dashboard...</div>;
  
  return (
    <iframe
      src={embedUrl}
      width="100%"
      height="600"
      frameBorder="0"
      allowFullScreen
    />
  );
}

// Backend: สร้าง embed URL
// server/routes/analytics.ts
import jwt from 'jsonwebtoken';
import express from 'express';

const router = express.Router();

router.post('/embed-url', (req, res) => {
  const { dashboardId, params } = req.body;
  
  const token = jwt.sign(
    {
      resource: { dashboard: dashboardId },
      params: params || {},
      exp: Math.floor(Date.now() / 1000) + 600
    },
    process.env.METABASE_EMBED_KEY!,
    { algorithm: 'HS256' }
  );
  
  const url = `${process.env.METABASE_URL}/embed/dashboard/${token}#bordered=true`;
  
  res.json({ url });
});
```

---

## 4. Apache Superset: Advanced BI

### 4.1 Installation

```yaml
# docker-compose-superset.yml
version: '3.8'

services:
  superset:
    image: apache/superset:3.0.0
    ports:
      - "8088:8088"
    environment:
      SUPERSET_SECRET_KEY: your-secret-key-32chars-minimum
      DATABASE_URL: postgresql+psycopg2://superset:superset@postgres/superset
      REDIS_URL: redis://redis:6379/0
    command: >
      bash -c "
        superset db upgrade &&
        superset fab create-admin --username admin --firstname Admin --lastname User --email admin@example.com --password admin &&
        superset init &&
        gunicorn --bind 0.0.0.0:8088 --workers 4 'superset.app:create_app()'
      "
    volumes:
      - superset_home:/app/superset_home
    depends_on:
      - postgres
      - redis

  celery:
    image: apache/superset:3.0.0
    command: celery --app=superset.tasks.celery_app:app worker --concurrency=4
    environment:
      DATABASE_URL: postgresql+psycopg2://superset:superset@postgres/superset
      REDIS_URL: redis://redis:6379/0
    depends_on:
      - postgres
      - redis

  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: superset
      POSTGRES_USER: superset
      POSTGRES_PASSWORD: superset
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine

volumes:
  superset_home:
  postgres_data:
```

```python
# superset_config.py
from datetime import timedelta

# Database
SQLALCHEMY_DATABASE_URI = 'postgresql+psycopg2://superset:superset@postgres/superset'

# Redis
CACHE_CONFIG = {
    'CACHE_TYPE': 'RedisCache',
    'CACHE_DEFAULT_TIMEOUT': 300,
    'CACHE_KEY_PREFIX': 'superset_',
    'CACHE_REDIS_URL': 'redis://redis:6379/0'
}

# Feature flags
FEATURE_FLAGS = {
    'ENABLE_TEMPLATE_PROCESSING': True,
    'DASHBOARD_NATIVE_FILTERS': True,
    'DASHBOARD_CROSS_FILTERS': True,
    'EMBEDDABLE_CHARTS': True,
    'EMBEDDED_SUPERSET': True,
}

# Security
SECRET_KEY = 'your-secret-key-here'

# Row level security
ENABLE_ROW_LEVEL_SECURITY = True

# SQL Lab settings
SQLLAB_TIMEOUT = 300
SQLLAB_ASYNC_TIME_LIMIT_SEC = 600

# Thumbnails
THUMBNAIL_SELENIUM_USER = 'admin'
THUMBNAIL_CACHE_CONFIG = {
    'CACHE_TYPE': 'RedisCache',
    'CACHE_DEFAULT_TIMEOUT': 24 * 60 * 60,
    'CACHE_KEY_PREFIX': 'thumbnail_',
    'CACHE_REDIS_URL': 'redis://redis:6379/0'
}
```

### 4.2 Superset API

```python
# superset_api.py
import requests
import json

class SupersetAPI:
    def __init__(self, base_url: str, username: str, password: str):
        self.base_url = base_url
        self.session = requests.Session()
        self._authenticate(username, password)
    
    def _authenticate(self, username: str, password: str):
        # Get CSRF token
        response = self.session.get(f"{self.base_url}/api/v1/security/csrf_token/")
        csrf_token = response.json()['result']
        self.session.headers.update({'X-CSRFToken': csrf_token})
        
        # Login
        response = self.session.post(
            f"{self.base_url}/api/v1/security/login",
            json={
                "username": username,
                "password": password,
                "provider": "db"
            }
        )
        response.raise_for_status()
        token = response.json()['access_token']
        self.session.headers.update({'Authorization': f'Bearer {token}'})
    
    def run_sql(self, database_id: int, sql: str) -> dict:
        """Run SQL query"""
        response = self.session.post(
            f"{self.base_url}/api/v1/sqllab/execute/",
            json={
                "database_id": database_id,
                "sql": sql,
                "runAsync": False
            }
        )
        response.raise_for_status()
        return response.json()
    
    def create_dataset(self, database_id: int, table_name: str, schema: str) -> dict:
        """เพิ่ม dataset (table) เข้า Superset"""
        response = self.session.post(
            f"{self.base_url}/api/v1/dataset/",
            json={
                "database": database_id,
                "table_name": table_name,
                "schema": schema
            }
        )
        response.raise_for_status()
        return response.json()
    
    def get_embed_token(self, dashboard_id: str) -> str:
        """Get embed token สำหรับ embedding"""
        response = self.session.post(
            f"{self.base_url}/api/v1/security/guest_token/",
            json={
                "resources": [{
                    "id": dashboard_id,
                    "type": "dashboard"
                }],
                "rls": [],
                "user": {"username": "guest"}
            }
        )
        response.raise_for_status()
        return response.json()['token']
```

---

## 5. Grafana สำหรับ Technical Metrics

### 5.1 Setup

```yaml
# docker-compose-monitoring.yml
version: '3.8'

services:
  grafana:
    image: grafana/grafana:10.2.0
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
      GF_DATABASE_TYPE: postgres
      GF_DATABASE_HOST: postgres:5432
      GF_DATABASE_NAME: grafana
      GF_DATABASE_USER: grafana
      GF_DATABASE_PASSWORD: grafana
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
      - ./grafana/dashboards:/var/lib/grafana/dashboards

  prometheus:
    image: prom/prometheus:v2.47.0
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus

  # Postgres exporter
  postgres_exporter:
    image: prometheuscommunity/postgres-exporter:v0.14.0
    environment:
      DATA_SOURCE_NAME: "postgresql://postgres:password@postgres:5432/mydb?sslmode=disable"
    ports:
      - "9187:9187"

volumes:
  grafana_data:
  prometheus_data:
```

```yaml
# prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'postgres'
    static_configs:
      - targets: ['postgres_exporter:9187']

  - job_name: 'node'
    static_configs:
      - targets: ['node_exporter:9100']

  - job_name: 'application'
    static_configs:
      - targets: ['app:9091']

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']
```

### 5.2 Grafana Dashboard สำหรับ PostgreSQL

```json
{
  "title": "PostgreSQL Overview",
  "panels": [
    {
      "title": "Active Connections",
      "type": "stat",
      "targets": [{
        "expr": "pg_stat_database_numbackends{datname='mydb'}"
      }]
    },
    {
      "title": "Query Rate",
      "type": "graph",
      "targets": [{
        "expr": "rate(pg_stat_database_tup_fetched{datname='mydb'}[5m])"
      }]
    },
    {
      "title": "Cache Hit Rate",
      "type": "gauge",
      "targets": [{
        "expr": "pg_stat_database_blks_hit{datname='mydb'} / (pg_stat_database_blks_hit{datname='mydb'} + pg_stat_database_blks_read{datname='mydb'}) * 100"
      }],
      "thresholds": "95,99"
    }
  ]
}
```

---

## 6. Real-time Dashboard

### 6.1 Redis สำหรับ Real-time Counters

```typescript
// server/realtime/counters.ts
import { createClient } from 'redis';
import { Server } from 'socket.io';

const redis = createClient({ url: process.env.REDIS_URL });

export class RealTimeCounters {
  constructor(private io: Server) {}
  
  async incrementOrderCount(amount: number) {
    const today = new Date().toISOString().split('T')[0];
    
    await Promise.all([
      redis.incr('counters:orders:total'),
      redis.incr(`counters:orders:${today}`),
      redis.incByFloat('counters:revenue:total', amount),
      redis.incByFloat(`counters:revenue:${today}`, amount)
    ]);
    
    // Broadcast real-time update
    this.broadcastMetrics();
  }
  
  async broadcastMetrics() {
    const metrics = await this.getMetrics();
    this.io.to('dashboard').emit('metrics:update', metrics);
  }
  
  async getMetrics() {
    const today = new Date().toISOString().split('T')[0];
    
    const [
      totalOrders, todayOrders,
      totalRevenue, todayRevenue,
      activeUsers
    ] = await Promise.all([
      redis.get('counters:orders:total'),
      redis.get(`counters:orders:${today}`),
      redis.get('counters:revenue:total'),
      redis.get(`counters:revenue:${today}`),
      redis.sCard('active_users')
    ]);
    
    return {
      orders: {
        total: parseInt(totalOrders || '0'),
        today: parseInt(todayOrders || '0')
      },
      revenue: {
        total: parseFloat(totalRevenue || '0'),
        today: parseFloat(todayRevenue || '0')
      },
      activeUsers: activeUsers,
      timestamp: new Date()
    };
  }
}

// Server: เชื่อม real-time counter กับ WebSocket
import express from 'express';
import { Server } from 'socket.io';

const app = express();
const io = new Server(httpServer);
const counters = new RealTimeCounters(io);

// เมื่อ order ใหม่เข้ามา
app.post('/api/orders', async (req, res) => {
  const order = await createOrder(req.body);
  
  // อัปเดต real-time counter
  await counters.incrementOrderCount(order.amount);
  
  res.json(order);
});

// Socket: subscribe to dashboard
io.on('connection', (socket) => {
  socket.on('dashboard:subscribe', async () => {
    socket.join('dashboard');
    
    // ส่ง initial metrics
    const metrics = await counters.getMetrics();
    socket.emit('metrics:update', metrics);
  });
});
```

### 6.2 Streaming SQL ใน PostgreSQL

```sql
-- PostgreSQL LISTEN/NOTIFY สำหรับ real-time updates
-- ส่ง notification ทุกครั้งที่มี order ใหม่

CREATE OR REPLACE FUNCTION notify_new_order()
RETURNS TRIGGER AS $$
BEGIN
    PERFORM pg_notify(
        'new_order',
        json_build_object(
            'id', NEW.id,
            'amount', NEW.amount,
            'status', NEW.status,
            'created_at', NEW.created_at
        )::TEXT
    );
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER order_notify_trigger
AFTER INSERT ON orders
FOR EACH ROW
EXECUTE FUNCTION notify_new_order();
```

```typescript
// server/notifications/pgNotify.ts
import { Pool } from 'pg';

const pool = new Pool({ connectionString: process.env.DATABASE_URL });

export async function listenForNewOrders(callback: (order: any) => void) {
  // ต้องใช้ dedicated connection (ไม่ใช้ pool)
  const client = await pool.connect();
  
  client.on('notification', (msg) => {
    if (msg.channel === 'new_order') {
      const order = JSON.parse(msg.payload!);
      callback(order);
    }
  });
  
  await client.query('LISTEN new_order');
  
  console.log('Listening for new orders...');
  
  return () => {
    client.query('UNLISTEN new_order');
    client.release();
  };
}

// ใช้งานกับ WebSocket
listenForNewOrders(async (order) => {
  // Broadcast ไปยัง dashboard subscribers
  io.to('dashboard').emit('order:new', order);
  
  // อัปเดต Redis counter
  await counters.incrementOrderCount(order.amount);
});
```

---

## 7. PostgreSQL Analytics Queries

### 7.1 Window Functions

```sql
-- Funnel Analysis: ติดตาม user journey

WITH funnel_events AS (
    SELECT
        user_id,
        event_type,
        created_at,
        ROW_NUMBER() OVER (
            PARTITION BY user_id, event_type
            ORDER BY created_at
        ) as event_rank
    FROM events
    WHERE created_at >= CURRENT_DATE - INTERVAL '30 days'
      AND event_type IN ('page_view', 'add_to_cart', 'checkout', 'purchase')
),

first_events AS (
    SELECT * FROM funnel_events WHERE event_rank = 1
),

funnel AS (
    SELECT
        event_type,
        COUNT(DISTINCT user_id) as users
    FROM first_events
    GROUP BY event_type
)

SELECT
    event_type,
    users,
    users::FLOAT / FIRST_VALUE(users) OVER (ORDER BY 
        CASE event_type
            WHEN 'page_view' THEN 1
            WHEN 'add_to_cart' THEN 2
            WHEN 'checkout' THEN 3
            WHEN 'purchase' THEN 4
        END
    ) as conversion_rate
FROM funnel
ORDER BY 
    CASE event_type
        WHEN 'page_view' THEN 1
        WHEN 'add_to_cart' THEN 2
        WHEN 'checkout' THEN 3
        WHEN 'purchase' THEN 4
    END;
```

### 7.2 Cohort Analysis

```sql
-- Monthly Retention Cohort Analysis

WITH first_purchase AS (
    SELECT
        user_id,
        DATE_TRUNC('month', MIN(created_at)) AS cohort_month
    FROM orders
    WHERE status = 'completed'
    GROUP BY user_id
),

monthly_revenue AS (
    SELECT
        o.user_id,
        DATE_TRUNC('month', o.created_at) AS revenue_month,
        SUM(o.amount) AS monthly_revenue
    FROM orders o
    WHERE o.status = 'completed'
    GROUP BY o.user_id, DATE_TRUNC('month', o.created_at)
),

cohort_analysis AS (
    SELECT
        fp.cohort_month,
        mr.revenue_month,
        EXTRACT(MONTH FROM AGE(mr.revenue_month, fp.cohort_month))::INT AS months_since_first,
        COUNT(DISTINCT fp.user_id) AS total_cohort_users,
        COUNT(DISTINCT mr.user_id) AS retained_users,
        SUM(mr.monthly_revenue) AS cohort_revenue
    FROM first_purchase fp
    LEFT JOIN monthly_revenue mr ON fp.user_id = mr.user_id
        AND mr.revenue_month >= fp.cohort_month
    GROUP BY fp.cohort_month, mr.revenue_month
)

SELECT
    TO_CHAR(cohort_month, 'YYYY-MM') AS cohort,
    months_since_first AS month_number,
    total_cohort_users,
    retained_users,
    ROUND(retained_users::FLOAT / NULLIF(total_cohort_users, 0) * 100, 1) AS retention_rate,
    ROUND(cohort_revenue / NULLIF(retained_users, 0), 2) AS avg_revenue_per_user
FROM cohort_analysis
WHERE months_since_first BETWEEN 0 AND 11
ORDER BY cohort_month, months_since_first;
```

### 7.3 RFM Analysis

```sql
-- RFM (Recency, Frequency, Monetary) Analysis
-- สำหรับ customer segmentation

WITH rfm_raw AS (
    SELECT
        user_id,
        MAX(created_at)::DATE AS last_order_date,
        COUNT(*) AS frequency,
        SUM(amount) AS monetary,
        CURRENT_DATE - MAX(created_at)::DATE AS recency_days
    FROM orders
    WHERE status = 'completed'
      AND created_at >= CURRENT_DATE - INTERVAL '1 year'
    GROUP BY user_id
),

rfm_scores AS (
    SELECT
        user_id,
        recency_days,
        frequency,
        monetary,
        
        -- Score 1-5 (5 = best)
        NTILE(5) OVER (ORDER BY recency_days DESC) AS r_score,  -- ต่ำ = ดี
        NTILE(5) OVER (ORDER BY frequency ASC) AS f_score,      -- สูง = ดี
        NTILE(5) OVER (ORDER BY monetary ASC) AS m_score        -- สูง = ดี
    FROM rfm_raw
),

rfm_segments AS (
    SELECT
        *,
        CONCAT(r_score, f_score, m_score) AS rfm_cell,
        CASE
            WHEN r_score >= 4 AND f_score >= 4 AND m_score >= 4 THEN 'Champions'
            WHEN r_score >= 3 AND f_score >= 3 AND m_score >= 3 THEN 'Loyal Customers'
            WHEN r_score >= 4 AND f_score <= 2 THEN 'New Customers'
            WHEN r_score >= 3 AND f_score <= 3 AND m_score >= 3 THEN 'Potential Loyalists'
            WHEN r_score <= 2 AND f_score >= 4 AND m_score >= 4 THEN 'At Risk'
            WHEN r_score <= 2 AND f_score <= 2 AND m_score <= 2 THEN 'Hibernating'
            ELSE 'Needs Attention'
        END AS segment
    FROM rfm_scores
)

SELECT
    segment,
    COUNT(*) AS customer_count,
    ROUND(AVG(recency_days), 0) AS avg_days_since_purchase,
    ROUND(AVG(frequency), 1) AS avg_purchase_frequency,
    ROUND(AVG(monetary), 0) AS avg_lifetime_value,
    ROUND(SUM(monetary), 0) AS total_revenue
FROM rfm_segments
GROUP BY segment
ORDER BY total_revenue DESC;
```

### 7.4 A/B Test Analysis

```sql
-- A/B Test Statistical Analysis

WITH experiment_data AS (
    SELECT
        user_id,
        variant,  -- 'control' หรือ 'treatment'
        purchased,
        revenue
    FROM ab_experiments
    WHERE experiment_name = 'checkout_button_color'
      AND created_at BETWEEN '2024-09-01' AND '2024-09-30'
),

stats AS (
    SELECT
        variant,
        COUNT(*) AS total_users,
        SUM(purchased) AS conversions,
        ROUND(SUM(purchased)::FLOAT / COUNT(*) * 100, 2) AS conversion_rate,
        AVG(revenue) AS avg_revenue,
        STDDEV(revenue) AS std_revenue,
        SUM(revenue) AS total_revenue
    FROM experiment_data
    GROUP BY variant
),

-- Chi-square test สำหรับ conversion rate
control_stats AS (
    SELECT * FROM stats WHERE variant = 'control'
),
treatment_stats AS (
    SELECT * FROM stats WHERE variant = 'treatment'
)

SELECT
    t.variant,
    t.total_users,
    t.conversions,
    t.conversion_rate,
    t.avg_revenue,
    
    -- Relative lift
    ROUND((t.conversion_rate - c.conversion_rate) / c.conversion_rate * 100, 2) 
        AS relative_lift_pct,
    
    -- Statistical significance (z-score)
    ROUND(
        (t.conversion_rate/100 - c.conversion_rate/100) /
        SQRT(
            (c.conversion_rate/100 * (1 - c.conversion_rate/100) / c.total_users) +
            (t.conversion_rate/100 * (1 - t.conversion_rate/100) / t.total_users)
        ),
        3
    ) AS z_score
FROM treatment_stats t
CROSS JOIN control_stats c

UNION ALL

SELECT
    c.variant,
    c.total_users,
    c.conversions,
    c.conversion_rate,
    c.avg_revenue,
    0 AS relative_lift_pct,
    0 AS z_score
FROM control_stats c;

-- Z-score interpretation:
-- |z| > 1.645 → p < 0.1 (90% confidence)
-- |z| > 1.96  → p < 0.05 (95% confidence)  ← standard
-- |z| > 2.576 → p < 0.01 (99% confidence)
```

---

## 8. Full Analytics Stack: dbt + Metabase + PostgreSQL

### 8.1 Complete Setup Script

```bash
#!/bin/bash
# setup_analytics.sh

echo "Setting up Analytics Stack..."

# 1. Start services
docker-compose -f docker-compose-analytics.yml up -d

# 2. Wait for services
echo "Waiting for services to be ready..."
sleep 30

# 3. Initialize dbt
cd /app/dbt
dbt debug --profiles-dir . # ตรวจสอบ connection
dbt deps                    # ติดตั้ง packages

# 4. Run dbt models
dbt seed                    # Load static data
dbt run                     # Build models
dbt test                    # Run tests
dbt docs generate           # Generate docs

# 5. Create Metabase connection
python3 scripts/setup_metabase.py

echo "Analytics stack ready!"
echo "- dbt docs: http://localhost:8001"
echo "- Metabase: http://localhost:3001"
```

```yaml
# docker-compose-analytics.yml
version: '3.8'

services:
  # PostgreSQL: source database
  postgres-oltp:
    image: postgres:15
    environment:
      POSTGRES_DB: production
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app_password
    ports:
      - "5432:5432"
    volumes:
      - oltp_data:/var/lib/postgresql/data

  # PostgreSQL: analytics database (หรือ DWH)
  postgres-analytics:
    image: postgres:15
    environment:
      POSTGRES_DB: analytics
      POSTGRES_USER: analytics
      POSTGRES_PASSWORD: analytics_password
    ports:
      - "5433:5432"
    volumes:
      - analytics_data:/var/lib/postgresql/data
      - ./sql/init_analytics.sql:/docker-entrypoint-initdb.d/init.sql

  # dbt: data transformations
  dbt:
    build:
      context: ./dbt
      dockerfile: Dockerfile
    volumes:
      - ./dbt:/app
    environment:
      DBT_PROFILES_DIR: /app
      PG_OLTP_HOST: postgres-oltp
      PG_ANALYTICS_HOST: postgres-analytics
    command: ["dbt", "docs", "serve", "--port", "8001"]
    ports:
      - "8001:8001"
    depends_on:
      - postgres-oltp
      - postgres-analytics

  # Metabase: BI tool
  metabase:
    image: metabase/metabase:v0.48.0
    environment:
      MB_DB_TYPE: postgres
      MB_DB_DBNAME: metabase
      MB_DB_PORT: 5432
      MB_DB_USER: metabase
      MB_DB_PASS: metabase_password
      MB_DB_HOST: postgres-metabase
    ports:
      - "3001:3000"
    depends_on:
      - postgres-metabase
      - postgres-analytics

  # Metabase metadata DB
  postgres-metabase:
    image: postgres:15
    environment:
      POSTGRES_DB: metabase
      POSTGRES_USER: metabase
      POSTGRES_PASSWORD: metabase_password
    volumes:
      - metabase_data:/var/lib/postgresql/data

  # Redis: caching
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  oltp_data:
  analytics_data:
  metabase_data:
```

### 8.2 Automated Reporting

```python
# scripts/automated_reports.py
# ส่ง daily report ทาง email

import smtplib
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText
from email.mime.image import MIMEImage
import psycopg2
import pandas as pd
import matplotlib.pyplot as plt
import io
from datetime import datetime, timedelta

def generate_daily_report():
    """สร้าง daily analytics report"""
    conn = psycopg2.connect(os.environ['ANALYTICS_DATABASE_URL'])
    
    # ดึงข้อมูล
    yesterday = datetime.now() - timedelta(days=1)
    date_str = yesterday.strftime('%Y-%m-%d')
    
    summary_df = pd.read_sql(f"""
        SELECT
            COUNT(*) as total_orders,
            COUNT(DISTINCT user_id) as unique_customers,
            SUM(amount) as total_revenue,
            AVG(amount) as avg_order_value,
            SUM(CASE WHEN status = 'completed' THEN 1 ELSE 0 END) as completed_orders,
            SUM(CASE WHEN status = 'cancelled' THEN 1 ELSE 0 END) as cancelled_orders
        FROM analytics.fct_orders
        WHERE order_date = '{date_str}'
    """, conn)
    
    hourly_df = pd.read_sql(f"""
        SELECT order_hour, COUNT(*) as orders, SUM(amount) as revenue
        FROM analytics.fct_orders
        WHERE order_date = '{date_str}'
        GROUP BY order_hour
        ORDER BY order_hour
    """, conn)
    
    category_df = pd.read_sql(f"""
        SELECT product_category, SUM(amount) as revenue, COUNT(*) as orders
        FROM analytics.fct_orders
        WHERE order_date = '{date_str}'
        GROUP BY product_category
        ORDER BY revenue DESC
        LIMIT 10
    """, conn)
    
    conn.close()
    
    # สร้างกราฟ
    fig, axes = plt.subplots(1, 2, figsize=(14, 5))
    
    # Hourly orders
    axes[0].bar(hourly_df['order_hour'], hourly_df['orders'])
    axes[0].set_title('Orders by Hour')
    axes[0].set_xlabel('Hour of Day')
    axes[0].set_ylabel('Number of Orders')
    
    # Revenue by category
    axes[1].barh(category_df['product_category'], category_df['revenue'])
    axes[1].set_title('Revenue by Category')
    axes[1].set_xlabel('Revenue (THB)')
    
    plt.tight_layout()
    
    # Save chart ไปยัง buffer
    buf = io.BytesIO()
    plt.savefig(buf, format='png', dpi=150, bbox_inches='tight')
    buf.seek(0)
    chart_image = buf.read()
    
    # สร้าง HTML email
    html_content = f"""
    <html>
    <body>
        <h2>Daily Report: {date_str}</h2>
        
        <h3>Key Metrics</h3>
        <table border="1" cellpadding="8" style="border-collapse: collapse;">
            <tr>
                <th>Metric</th>
                <th>Value</th>
            </tr>
            <tr><td>Total Orders</td><td>{summary_df['total_orders'][0]:,.0f}</td></tr>
            <tr><td>Unique Customers</td><td>{summary_df['unique_customers'][0]:,.0f}</td></tr>
            <tr><td>Total Revenue</td><td>฿{summary_df['total_revenue'][0]:,.2f}</td></tr>
            <tr><td>Avg Order Value</td><td>฿{summary_df['avg_order_value'][0]:,.2f}</td></tr>
            <tr><td>Completed Orders</td><td>{summary_df['completed_orders'][0]:,.0f}</td></tr>
            <tr><td>Cancelled Orders</td><td>{summary_df['cancelled_orders'][0]:,.0f}</td></tr>
        </table>
        
        <br>
        <img src="cid:chart" />
        
        <p>View full dashboard: <a href="http://analytics.example.com">Click here</a></p>
    </body>
    </html>
    """
    
    return html_content, chart_image

def send_report(recipients: list):
    """ส่ง report ทาง email"""
    html_content, chart_image = generate_daily_report()
    
    msg = MIMEMultipart('related')
    msg['Subject'] = f'Daily Analytics Report - {datetime.now().strftime("%Y-%m-%d")}'
    msg['From'] = 'analytics@example.com'
    msg['To'] = ', '.join(recipients)
    
    msg.attach(MIMEText(html_content, 'html'))
    
    img = MIMEImage(chart_image)
    img.add_header('Content-ID', '<chart>')
    msg.attach(img)
    
    with smtplib.SMTP('smtp.gmail.com', 587) as server:
        server.starttls()
        server.login(os.environ['SMTP_USER'], os.environ['SMTP_PASSWORD'])
        server.send_message(msg)
    
    print("Report sent successfully!")

# Run daily at 8am
if __name__ == '__main__':
    send_report(['team@example.com', 'manager@example.com'])
```

---

## 9. Monitoring Data Quality

```python
# monitoring/data_quality.py
import psycopg2
import json
from datetime import datetime

class DataQualityMonitor:
    def __init__(self, conn_string: str):
        self.conn = psycopg2.connect(conn_string)
    
    def check_freshness(self, table: str, timestamp_col: str, max_hours: int = 24) -> dict:
        """ตรวจสอบว่า data ใหม่พอ"""
        cur = self.conn.cursor()
        cur.execute(f"""
            SELECT
                MAX({timestamp_col}) as latest_record,
                EXTRACT(EPOCH FROM (NOW() - MAX({timestamp_col}))) / 3600 as hours_since_latest
            FROM {table}
        """)
        result = cur.fetchone()
        
        return {
            'table': table,
            'latest_record': result[0],
            'hours_since_latest': result[1],
            'is_fresh': result[1] <= max_hours,
            'checked_at': datetime.now()
        }
    
    def check_completeness(self, table: str, required_columns: list) -> dict:
        """ตรวจสอบ null values ใน required columns"""
        checks = {}
        
        for col in required_columns:
            cur = self.conn.cursor()
            cur.execute(f"""
                SELECT
                    COUNT(*) as total_rows,
                    COUNT({col}) as non_null_rows,
                    ROUND(COUNT({col})::FLOAT / NULLIF(COUNT(*), 0) * 100, 2) as completeness_pct
                FROM {table}
                WHERE created_at >= NOW() - INTERVAL '24 hours'
            """)
            result = cur.fetchone()
            checks[col] = {
                'total_rows': result[0],
                'non_null_rows': result[1],
                'completeness_pct': result[2],
                'passed': result[2] >= 99.0  # 99% threshold
            }
        
        return {'table': table, 'columns': checks}
    
    def check_uniqueness(self, table: str, key_columns: list) -> dict:
        """ตรวจสอบ duplicates"""
        key_str = ', '.join(key_columns)
        cur = self.conn.cursor()
        cur.execute(f"""
            SELECT COUNT(*) as duplicates
            FROM (
                SELECT {key_str}, COUNT(*) as cnt
                FROM {table}
                GROUP BY {key_str}
                HAVING COUNT(*) > 1
            ) t
        """)
        duplicates = cur.fetchone()[0]
        
        return {
            'table': table,
            'key_columns': key_columns,
            'duplicate_count': duplicates,
            'passed': duplicates == 0
        }
    
    def run_all_checks(self) -> dict:
        """รัน checks ทั้งหมดและส่ง alert ถ้าไม่ผ่าน"""
        results = {
            'run_at': datetime.now().isoformat(),
            'checks': []
        }
        
        checks_config = [
            {
                'type': 'freshness',
                'table': 'analytics.fct_orders',
                'timestamp_col': 'created_at',
                'max_hours': 25
            },
            {
                'type': 'completeness',
                'table': 'analytics.fct_orders',
                'required_columns': ['order_id', 'user_id', 'amount']
            },
            {
                'type': 'uniqueness',
                'table': 'analytics.fct_orders',
                'key_columns': ['order_id']
            }
        ]
        
        all_passed = True
        
        for config in checks_config:
            if config['type'] == 'freshness':
                result = self.check_freshness(
                    config['table'],
                    config['timestamp_col'],
                    config.get('max_hours', 24)
                )
            elif config['type'] == 'completeness':
                result = self.check_completeness(
                    config['table'],
                    config['required_columns']
                )
            elif config['type'] == 'uniqueness':
                result = self.check_uniqueness(
                    config['table'],
                    config['key_columns']
                )
            
            results['checks'].append(result)
            
            # ตรวจสอบว่าผ่านหรือไม่
            if not result.get('is_fresh', True) or not result.get('passed', True):
                all_passed = False
        
        results['overall_passed'] = all_passed
        
        if not all_passed:
            self.send_alert(results)
        
        return results
    
    def send_alert(self, results: dict):
        """ส่ง Slack alert เมื่อ data quality check fail"""
        import requests
        
        failed_checks = [
            c for c in results['checks']
            if not c.get('is_fresh', True) or not c.get('passed', True)
        ]
        
        message = {
            "text": f"⚠️ Data Quality Alert - {len(failed_checks)} checks failed",
            "blocks": [
                {
                    "type": "section",
                    "text": {
                        "type": "mrkdwn",
                        "text": f"*Data Quality Issues Detected*\n\n" + 
                                "\n".join([f"• {c.get('table', 'unknown')}" for c in failed_checks])
                    }
                }
            ]
        }
        
        requests.post(
            os.environ['SLACK_WEBHOOK_URL'],
            json=message
        )

# Schedule: รัน checks ทุก 1 ชั่วโมง
monitor = DataQualityMonitor(os.environ['ANALYTICS_DATABASE_URL'])
results = monitor.run_all_checks()
print(json.dumps(results, indent=2, default=str))
```

---

## สรุป

Analytics Pipeline ที่สมบูรณ์ประกอบด้วย:

1. **dbt**: SQL-based transformations พร้อม testing, documentation และ lineage graph
2. **Metabase**: BI tool ที่ใช้ง่าย เหมาะสำหรับ business users
3. **Apache Superset**: BI tool ที่ powerful กว่า เหมาะสำหรับ data analysts
4. **Grafana**: สำหรับ technical metrics และ infrastructure monitoring
5. **Real-time**: PostgreSQL LISTEN/NOTIFY + Redis + WebSocket สำหรับ live dashboards
6. **Data Quality**: automated checks ที่ alert เมื่อมีปัญหา

**Recommended stack ตาม scale:**

- **Startup**: PostgreSQL + dbt + Metabase (simple, cheap, effective)
- **Growth**: + ClickHouse สำหรับ analytics + Superset สำหรับ advanced queries
- **Enterprise**: + Kafka + Spark + Data Lake + Airflow orchestration

Key principle: Start simple, measure everything, และ scale เฉพาะส่วนที่เป็น bottleneck จริงๆ

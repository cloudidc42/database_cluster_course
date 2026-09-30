# Part 83: Data Lake Architecture

## บทนำ

องค์กรยุคใหม่มีข้อมูลมหาศาลจากทุกทิศทาง ไม่ว่าจะเป็น transactions, user behaviors, logs, IoT sensors, social media การจัดการข้อมูลเหล่านี้อย่างมีประสิทธิภาพคือ competitive advantage ที่แท้จริง บทนี้จะครอบคลุม Data Lake Architecture ทั้งแบบ AWS-managed และ self-hosted

---

## 1. Data Lake vs Data Warehouse vs Database

### 1.1 เปรียบเทียบ Three Paradigms

```
┌─────────────────────────────────────────────────────────────┐
│                    TRADITIONAL DATABASE                      │
│                    (PostgreSQL, MySQL)                       │
├─────────────────────────────────────────────────────────────┤
│ ✅ Structured data, strict schema                           │
│ ✅ ACID transactions                                         │
│ ✅ Real-time queries (ms)                                   │
│ ❌ ไม่ scale ได้ถึง petabytes                              │
│ ❌ ต้องรู้ schema ก่อนจึงจะเก็บข้อมูลได้                  │
│ Use case: OLTP, application backend                         │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    DATA WAREHOUSE                            │
│             (Redshift, BigQuery, Snowflake)                  │
├─────────────────────────────────────────────────────────────┤
│ ✅ Structured, transformed, high quality                    │
│ ✅ Fast analytics queries                                   │
│ ✅ Business reporting                                       │
│ ❌ Expensive, rigid schema                                  │
│ ❌ ต้องผ่าน ETL ก่อน                                       │
│ Use case: OLAP, business intelligence                       │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    DATA LAKE                                 │
│                    (S3, HDFS, GCS)                          │
├─────────────────────────────────────────────────────────────┤
│ ✅ Any format: structured, semi-structured, unstructured    │
│ ✅ Massive scale (petabytes)                                │
│ ✅ Cheap storage                                            │
│ ✅ Store raw data → transform ทีหลัง                       │
│ ❌ Query ช้ากว่า warehouse                                  │
│ ❌ ง่ายที่จะกลายเป็น "data swamp"                          │
│ Use case: raw data storage, ML training data                │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    LAKEHOUSE                                 │
│           (Delta Lake, Apache Iceberg, Apache Hudi)         │
├─────────────────────────────────────────────────────────────┤
│ ✅ Data Lake + Data Warehouse features                      │
│ ✅ ACID on object storage                                   │
│ ✅ Schema enforcement + evolution                           │
│ ✅ Time travel (query historical data)                      │
│ ✅ Unified storage for BI + ML                             │
│ Use case: modern analytics architecture                     │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 Data Lake Zones

```
┌──────────────────────────────────────────────────────────────┐
│                        DATA LAKE                             │
│                                                              │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐    │
│  │  BRONZE/RAW  │   │ SILVER/CLEAN │   │  GOLD/CURATED│    │
│  │              │   │              │   │              │    │
│  │ Raw data     │→  │ Cleaned,     │→  │ Aggregated,  │    │
│  │ As-is        │   │ Validated,   │   │ Business-    │    │
│  │ JSON, CSV    │   │ Deduplicated │   │ ready        │    │
│  │ Immutable    │   │ Parquet      │   │ Star schema  │    │
│  └──────────────┘   └──────────────┘   └──────────────┘    │
│                                                              │
│  s3://bucket/      s3://bucket/        s3://bucket/         │
│  raw/              clean/              curated/              │
└──────────────────────────────────────────────────────────────┘
```

---

## 2. AWS Data Lake Architecture

### 2.1 Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                     AWS DATA LAKE ARCHITECTURE                       │
│                                                                      │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────────┐ │
│  │  SOURCES    │    │  INGESTION  │    │       STORAGE            │ │
│  │             │    │             │    │                          │ │
│  │ PostgreSQL  │→   │ DMS/CDC     │→   │   Amazon S3              │ │
│  │ MySQL       │    │ Kinesis     │    │   ├── raw/               │ │
│  │ APIs        │→   │ Firehose    │    │   ├── processed/         │ │
│  │ IoT         │→   │ Kafka       │    │   └── curated/           │ │
│  │ Log files   │→   │ AppFlow     │    │                          │ │
│  └─────────────┘    └─────────────┘    └─────────────────────────┘ │
│                                                 ↓                   │
│  ┌──────────────────────────┐    ┌─────────────────────────────┐   │
│  │     PROCESSING           │    │       CATALOG               │   │
│  │                          │    │                             │   │
│  │  AWS Glue (ETL)          │→   │   AWS Glue Data Catalog     │   │
│  │  AWS Lambda (serverless) │    │   (Hive Metastore compatible│   │
│  │  Amazon EMR (Spark)      │    │   Tables, Schemas)          │   │
│  └──────────────────────────┘    └─────────────────────────────┘   │
│                                                 ↓                   │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                     QUERY & ANALYTICS                         │   │
│  │                                                              │   │
│  │  Amazon Athena  │  Redshift Spectrum  │  Amazon EMR          │   │
│  │  (SQL on S3)    │  (Redshift + S3)    │  (Spark/Hive)        │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                 ↓                                   │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                     VISUALIZATION                             │   │
│  │                                                              │   │
│  │  Amazon QuickSight  │  Grafana  │  Tableau  │  PowerBI       │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 S3: Raw Storage

```bash
# S3 bucket structure
s3://company-data-lake/
├── raw/
│   ├── rds/
│   │   ├── orders/
│   │   │   ├── year=2024/month=09/day=01/orders_20240901.parquet
│   │   │   └── year=2024/month=09/day=02/orders_20240902.parquet
│   │   └── users/
│   │       └── year=2024/month=09/day=01/users_20240901.parquet
│   ├── clickstream/
│   │   └── year=2024/month=09/day=01/hour=00/events-001.json.gz
│   └── logs/
│       └── year=2024/month=09/day=01/app.log.gz
├── processed/
│   ├── orders_enriched/
│   └── user_sessions/
└── curated/
    ├── daily_revenue/
    ├── user_cohorts/
    └── product_performance/
```

```python
# Python: อัปโหลดข้อมูลไปยัง S3
import boto3
import pandas as pd
import pyarrow as pa
import pyarrow.parquet as pq
from io import BytesIO
from datetime import datetime

s3 = boto3.client('s3',
    region_name='ap-southeast-1',
    aws_access_key_id=os.environ['AWS_ACCESS_KEY_ID'],
    aws_secret_access_key=os.environ['AWS_SECRET_ACCESS_KEY']
)

def upload_to_s3_parquet(df: pd.DataFrame, bucket: str, key: str):
    """อัปโหลด DataFrame เป็น Parquet ไปยัง S3"""
    buffer = BytesIO()
    
    # Convert to Parquet
    table = pa.Table.from_pandas(df)
    pq.write_table(
        table,
        buffer,
        compression='snappy',  # หรือ 'gzip', 'zstd'
        use_dictionary=True,
        write_statistics=True
    )
    
    buffer.seek(0)
    s3.upload_fileobj(buffer, bucket, key)
    print(f"Uploaded to s3://{bucket}/{key}")

# ใช้งาน
df = pd.read_sql("SELECT * FROM orders WHERE created_at::date = '2024-09-01'", conn)

today = datetime.now()
key = f"raw/rds/orders/year={today.year}/month={today.month:02d}/day={today.day:02d}/orders.parquet"

upload_to_s3_parquet(df, 'company-data-lake', key)
```

### 2.3 AWS Glue: ETL + Data Catalog

```python
# AWS Glue ETL Job (PySpark)
# ไฟล์นี้รันบน AWS Glue

import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.job import Job
from pyspark.sql import functions as F
from pyspark.sql.types import *

args = getResolvedOptions(sys.argv, ['JOB_NAME', 'source_path', 'dest_path'])

sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args['JOB_NAME'], args)

# อ่านข้อมูล raw จาก S3
raw_orders = glueContext.create_dynamic_frame.from_options(
    format_options={"jsonPath": "$[*]", "multiline": False},
    connection_type="s3",
    format="json",
    connection_options={
        "paths": [args['source_path']],
        "recurse": True
    }
)

# Convert to Spark DataFrame
df = raw_orders.toDF()

# Transformations
df_cleaned = df \
    .filter(F.col('id').isNotNull()) \
    .filter(F.col('amount') > 0) \
    .withColumn('created_date', F.to_date(F.col('created_at'))) \
    .withColumn('amount_usd', F.col('amount') / 33.0) \
    .withColumn('order_size', 
        F.when(F.col('amount') < 500, 'small')
         .when(F.col('amount') < 2000, 'medium')
         .otherwise('large')
    ) \
    .dropDuplicates(['id'])

# Write to S3 ด้วย partition
df_cleaned.write \
    .mode('overwrite') \
    .partitionBy('created_date', 'order_size') \
    .parquet(args['dest_path'])

# Update Glue Catalog
glueContext.purge_s3_path(args['dest_path'], {"retentionPeriod": 1})

job.commit()
```

```python
# AWS Glue Crawler: auto-discover schema จาก S3

import boto3

glue = boto3.client('glue', region_name='ap-southeast-1')

# สร้าง crawler
glue.create_crawler(
    Name='orders-crawler',
    Role='arn:aws:iam::123456789:role/GlueServiceRole',
    DatabaseName='data_lake',
    Targets={
        'S3Targets': [
            {
                'Path': 's3://company-data-lake/processed/orders_enriched/',
                'Exclusions': ['**.tmp']
            }
        ]
    },
    Schedule='cron(0 2 * * ? *)',  # รันทุกวัน 2am UTC
    SchemaChangePolicy={
        'UpdateBehavior': 'UPDATE_IN_DATABASE',
        'DeleteBehavior': 'LOG'
    },
    RecrawlPolicy={'RecrawlBehavior': 'CRAWL_EVERYTHING'}
)

# Run crawler
glue.start_crawler(Name='orders-crawler')
```

### 2.4 Amazon Athena: SQL on S3

```sql
-- สร้าง database ใน Athena
CREATE DATABASE IF NOT EXISTS data_lake;

-- สร้าง external table (อ่านจาก S3)
CREATE EXTERNAL TABLE IF NOT EXISTS data_lake.orders_raw (
    id STRING,
    user_id STRING,
    product_id STRING,
    quantity INT,
    amount DOUBLE,
    status STRING,
    created_at TIMESTAMP
)
PARTITIONED BY (
    year STRING,
    month STRING,
    day STRING
)
STORED AS PARQUET
LOCATION 's3://company-data-lake/raw/rds/orders/'
TBLPROPERTIES (
    'parquet.compress' = 'SNAPPY',
    'has_encrypted_data' = 'false'
);

-- Load partitions
MSCK REPAIR TABLE data_lake.orders_raw;

-- Query ด้วย partition pruning (เร็วมาก เพราะไม่ scan ทั้งหมด)
SELECT
    year,
    month,
    COUNT(*) as order_count,
    SUM(amount) as total_revenue,
    AVG(amount) as avg_order_value
FROM data_lake.orders_raw
WHERE year = '2024' AND month = '09'
GROUP BY year, month
ORDER BY year, month;

-- Complex analytics
WITH daily_stats AS (
    SELECT
        DATE(created_at) as order_date,
        status,
        COUNT(*) as order_count,
        SUM(amount) as revenue
    FROM data_lake.orders_raw
    WHERE year = '2024'
    GROUP BY 1, 2
),
daily_total AS (
    SELECT
        order_date,
        SUM(revenue) as total_revenue,
        SUM(order_count) as total_orders
    FROM daily_stats
    GROUP BY 1
)
SELECT
    dt.order_date,
    dt.total_revenue,
    dt.total_orders,
    dt.total_revenue / NULLIF(dt.total_orders, 0) as avg_order_value,
    -- 7-day moving average
    AVG(dt.total_revenue) OVER (
        ORDER BY dt.order_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) as revenue_7day_ma
FROM daily_total dt
ORDER BY dt.order_date;

-- Cost optimization: ใช้ CTAS เพื่อสร้าง materialized view
CREATE TABLE data_lake.orders_monthly_summary
WITH (
    format = 'PARQUET',
    parquet_compression = 'SNAPPY',
    partitioned_by = ARRAY['year', 'month'],
    external_location = 's3://company-data-lake/curated/orders_monthly/'
) AS
SELECT
    SUM(amount) as total_revenue,
    COUNT(*) as order_count,
    COUNT(DISTINCT user_id) as unique_customers,
    AVG(amount) as avg_order_value,
    year,
    month
FROM data_lake.orders_raw
GROUP BY year, month;
```

### 2.5 AWS Lake Formation: Data Governance

```python
# Lake Formation: column-level security
import boto3

lf = boto3.client('lakeformation', region_name='ap-southeast-1')

# Grant permissions บน specific columns
lf.grant_permissions(
    Principal={'DataLakePrincipalIdentifier': 'arn:aws:iam::123456789:role/AnalystRole'},
    Resource={
        'TableWithColumns': {
            'DatabaseName': 'data_lake',
            'Name': 'orders_raw',
            'ColumnNames': ['id', 'amount', 'status', 'created_at'],
            # Exclude PII columns
            # 'ColumnWildcard': {'ExcludedColumnNames': ['user_email', 'phone']}
        }
    },
    Permissions=['SELECT'],
    PermissionsWithGrantOption=[]
)

# Data filters: row-level security
lf.create_data_cells_filter(
    TableData={
        'TableCatalogId': '123456789012',
        'DatabaseName': 'data_lake',
        'TableName': 'orders_raw',
        'Name': 'thailand_only',
        'RowFilter': {
            'FilterExpression': "country = 'TH'",
            'AllRowsWildcard': {}
        },
        'ColumnWildcard': {}
    }
)
```

---

## 3. Data Formats: Raw vs Optimized

### 3.1 JSON (Raw)

```json
// ข้อดี: human readable, flexible schema
// ข้อเสีย: ใหญ่, scan ทั้ง row แม้ต้องการแค่บาง columns

{"id":"abc123","userId":"user1","amount":1500.00,"status":"completed","createdAt":"2024-09-01T10:00:00Z"}
{"id":"abc124","userId":"user2","amount":250.00,"status":"pending","createdAt":"2024-09-01T10:01:00Z"}
```

### 3.2 Parquet (Columnar)

```
Row-oriented (JSON, CSV):
Row 1: [id=abc123, userId=user1, amount=1500, status=completed, ...]
Row 2: [id=abc124, userId=user2, amount=250, status=pending, ...]

Column-oriented (Parquet):
id column:     [abc123, abc124, abc125, ...]
userId column: [user1, user2, user3, ...]
amount column: [1500, 250, 3000, ...]
status column: [completed, pending, completed, ...]

Query: SELECT SUM(amount) FROM orders WHERE status = 'completed'
→ อ่านเฉพาะ 2 columns แทนที่จะอ่านทุก column
→ 10-100x faster สำหรับ analytical queries
→ 80-90% compression เพราะ column values คล้ายกัน
```

```python
# เปรียบเทียบ size และ performance
import pandas as pd
import pyarrow as pa
import pyarrow.parquet as pq
import time
import os

# สร้างข้อมูลทดสอบ
df = pd.DataFrame({
    'id': range(1000000),
    'user_id': [f'user_{i % 10000}' for i in range(1000000)],
    'amount': [float(i % 5000) for i in range(1000000)],
    'status': ['completed' if i % 3 == 0 else 'pending' for i in range(1000000)],
    'created_at': pd.date_range('2024-01-01', periods=1000000, freq='s')
})

# Save as CSV
start = time.time()
df.to_csv('/tmp/orders.csv', index=False)
csv_time = time.time() - start
csv_size = os.path.getsize('/tmp/orders.csv') / 1024 / 1024

# Save as Parquet (snappy)
start = time.time()
df.to_parquet('/tmp/orders.parquet', compression='snappy')
parquet_time = time.time() - start
parquet_size = os.path.getsize('/tmp/orders.parquet') / 1024 / 1024

print(f"CSV: {csv_size:.1f} MB, write: {csv_time:.2f}s")
print(f"Parquet: {parquet_size:.1f} MB, write: {parquet_time:.2f}s")
print(f"Size reduction: {(1 - parquet_size/csv_size)*100:.0f}%")

# Query performance
start = time.time()
csv_df = pd.read_csv('/tmp/orders.csv')
csv_query = csv_df[csv_df['status'] == 'completed']['amount'].sum()
csv_query_time = time.time() - start

start = time.time()
parquet_df = pd.read_parquet('/tmp/orders.parquet', columns=['amount', 'status'])
parquet_query = parquet_df[parquet_df['status'] == 'completed']['amount'].sum()
parquet_query_time = time.time() - start

print(f"\nQuery SUM(amount) WHERE status='completed':")
print(f"CSV: {csv_query_time:.2f}s")
print(f"Parquet: {parquet_query_time:.2f}s")
print(f"Speedup: {csv_query_time/parquet_query_time:.0f}x")
```

### 3.3 ORC และ Avro

```python
# ORC: ดีสำหรับ Hive/Hadoop ecosystem
# pyorc library
import pyorc

with open('/tmp/orders.orc', 'wb') as f:
    writer = pyorc.Writer(f, "struct<id:int,amount:double,status:string>")
    writer.write((1, 1500.0, 'completed'))
    writer.write((2, 250.0, 'pending'))

# Avro: ดีสำหรับ schema evolution กับ Kafka
import avro.schema
import avro.datafile
import avro.io

schema = avro.schema.parse(json.dumps({
    "type": "record",
    "name": "Order",
    "fields": [
        {"name": "id", "type": "string"},
        {"name": "amount", "type": "double"},
        {"name": "status", "type": "string"}
    ]
}))
```

---

## 4. Partitioning ใน S3

### 4.1 Hive-style Partitioning

```
s3://bucket/events/
├── year=2024/
│   ├── month=09/
│   │   ├── day=01/
│   │   │   ├── region=TH/
│   │   │   │   └── events-0001.parquet
│   │   │   └── region=SG/
│   │   │       └── events-0001.parquet
│   │   └── day=02/
│   └── month=10/
└── year=2025/
```

```sql
-- Athena partition pruning
-- Query นี้อ่านเฉพาะ partition year=2024/month=09/day=01/region=TH
SELECT COUNT(*) 
FROM events 
WHERE year = '2024' 
  AND month = '09' 
  AND day = '01' 
  AND region = 'TH';

-- ไม่ควรทำ (full table scan ทั้งหมด):
SELECT COUNT(*) 
FROM events 
WHERE DATE(event_time) = '2024-09-01';
```

### 4.2 Partitioning Strategy

```python
# Python: partition data ก่อนอัปโหลด S3
import pandas as pd
import boto3
from io import BytesIO
import pyarrow as pa
import pyarrow.parquet as pq

def upload_partitioned(df: pd.DataFrame, bucket: str, prefix: str):
    """อัปโหลดข้อมูล partitioned by date"""
    s3 = boto3.client('s3')
    
    # เพิ่ม partition columns
    df['year'] = df['created_at'].dt.year.astype(str)
    df['month'] = df['created_at'].dt.month.apply(lambda x: f'{x:02d}')
    df['day'] = df['created_at'].dt.day.apply(lambda x: f'{x:02d}')
    
    # Group by partition
    for (year, month, day), group in df.groupby(['year', 'month', 'day']):
        # Remove partition columns จาก data
        data = group.drop(columns=['year', 'month', 'day'])
        
        # Convert to Parquet
        buffer = BytesIO()
        table = pa.Table.from_pandas(data)
        pq.write_table(table, buffer, compression='snappy')
        buffer.seek(0)
        
        # Upload
        key = f"{prefix}/year={year}/month={month}/day={day}/data.parquet"
        s3.upload_fileobj(buffer, bucket, key)
        
        print(f"Uploaded {len(data)} rows to s3://{bucket}/{key}")

# ใช้งาน
df = pd.read_sql("SELECT * FROM orders", conn)
upload_partitioned(df, 'my-data-lake', 'raw/orders')
```

---

## 5. ETL/ELT Pipeline

### 5.1 Extract: PostgreSQL → S3

```python
# extract/postgres_to_s3.py
import psycopg2
import pandas as pd
import boto3
from datetime import datetime, timedelta
import os

class PostgresExtractor:
    def __init__(self, conn_string: str, s3_bucket: str):
        self.conn_string = conn_string
        self.s3 = boto3.client('s3')
        self.s3_bucket = s3_bucket
    
    def extract_table(self, table: str, incremental_column: str = None, 
                      last_run: datetime = None):
        """Extract ข้อมูลจาก PostgreSQL table"""
        conn = psycopg2.connect(self.conn_string)
        
        if incremental_column and last_run:
            # Incremental extract
            query = f"""
                SELECT * FROM {table}
                WHERE {incremental_column} > %s
                ORDER BY {incremental_column}
            """
            df = pd.read_sql(query, conn, params=(last_run,))
        else:
            # Full extract
            df = pd.read_sql(f"SELECT * FROM {table}", conn)
        
        conn.close()
        
        print(f"Extracted {len(df)} rows from {table}")
        return df
    
    def extract_with_cdc(self, table: str, replication_slot: str):
        """CDC extraction ด้วย logical replication"""
        conn = psycopg2.connect(self.conn_string)
        conn.autocommit = True
        cur = conn.cursor()
        
        changes = []
        cur.execute(f"""
            SELECT * FROM pg_logical_slot_get_changes(
                '{replication_slot}', NULL, NULL,
                'include-xids', '1',
                'include-timestamp', '1'
            )
        """)
        
        for row in cur.fetchall():
            changes.append(row)
        
        cur.close()
        conn.close()
        return changes
    
    def upload_to_s3(self, df: pd.DataFrame, table: str, date: datetime):
        """อัปโหลดไปยัง S3 พร้อม partitioning"""
        key = (f"raw/rds/{table}/"
               f"year={date.year}/"
               f"month={date.month:02d}/"
               f"day={date.day:02d}/"
               f"{table}_{date.strftime('%Y%m%d_%H%M%S')}.parquet")
        
        buffer = __import__('io').BytesIO()
        df.to_parquet(buffer, compression='snappy', index=False)
        buffer.seek(0)
        
        self.s3.upload_fileobj(buffer, self.s3_bucket, key)
        return f"s3://{self.s3_bucket}/{key}"

# ใช้งาน
extractor = PostgresExtractor(
    conn_string=os.environ['DATABASE_URL'],
    s3_bucket='my-data-lake'
)

# Extract orders ที่เปลี่ยนแปลงในวันนี้
df = extractor.extract_table(
    table='orders',
    incremental_column='updated_at',
    last_run=datetime.now() - timedelta(days=1)
)

s3_path = extractor.upload_to_s3(df, 'orders', datetime.now())
print(f"Data uploaded to: {s3_path}")
```

### 5.2 Transform: PySpark

```python
# transform/orders_transform.py
from pyspark.sql import SparkSession
from pyspark.sql import functions as F
from pyspark.sql.types import *
import sys

spark = SparkSession.builder \
    .appName("OrdersTransform") \
    .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension") \
    .config("spark.sql.catalog.spark_catalog", "org.apache.spark.sql.delta.catalog.DeltaCatalog") \
    .getOrCreate()

def transform_orders(input_path: str, output_path: str, users_path: str):
    # อ่านข้อมูล raw
    orders = spark.read.parquet(input_path)
    users = spark.read.parquet(users_path)
    
    # Join และ enrich
    enriched = orders \
        .join(users.select('id', 'email', 'country', 'segment'),
              orders.user_id == users.id, 'left') \
        .select(
            orders['*'],
            F.col('email').alias('user_email'),
            F.col('country').alias('user_country'),
            F.col('segment').alias('user_segment')
        )
    
    # Transformations
    result = enriched \
        .filter(F.col('deleted_at').isNull()) \
        .withColumn('amount_thb', F.col('amount').cast(DoubleType())) \
        .withColumn('amount_usd', F.round(F.col('amount') / 33.0, 2)) \
        .withColumn('order_date', F.to_date(F.col('created_at'))) \
        .withColumn('order_hour', F.hour(F.col('created_at'))) \
        .withColumn('order_dayofweek', F.dayofweek(F.col('created_at'))) \
        .withColumn('is_weekend',
            F.when(F.col('order_dayofweek').isin([1, 7]), True).otherwise(False)
        ) \
        .withColumn('order_size',
            F.when(F.col('amount') < 500, 'small')
             .when(F.col('amount') < 2000, 'medium')
             .when(F.col('amount') < 10000, 'large')
             .otherwise('enterprise')
        ) \
        .withColumn('processing_timestamp', F.current_timestamp())
    
    # Data quality checks
    invalid = result.filter(
        F.col('amount') < 0 |
        F.col('user_id').isNull() |
        F.col('id').isNull()
    )
    
    if invalid.count() > 0:
        print(f"WARNING: {invalid.count()} invalid records found")
        invalid.write.mode('append').parquet(output_path + '_invalid')
    
    valid = result.filter(
        F.col('amount') >= 0 &
        F.col('user_id').isNotNull() &
        F.col('id').isNotNull()
    )
    
    # Write partitioned Parquet
    valid.write \
        .mode('overwrite') \
        .partitionBy('order_date', 'user_country') \
        .parquet(output_path)
    
    print(f"Wrote {valid.count()} records to {output_path}")

if __name__ == '__main__':
    transform_orders(
        input_path='s3://my-data-lake/raw/rds/orders/',
        output_path='s3://my-data-lake/processed/orders_enriched/',
        users_path='s3://my-data-lake/processed/users/'
    )
```

### 5.3 Load: S3 → Redshift

```python
# load/s3_to_redshift.py
import boto3
import psycopg2

def load_to_redshift(
    s3_path: str,
    table: str,
    schema: str,
    iam_role: str,
    redshift_conn_string: str
):
    """Load จาก S3 ไปยัง Redshift"""
    conn = psycopg2.connect(redshift_conn_string)
    cur = conn.cursor()
    
    # COPY command (Redshift optimized)
    copy_sql = f"""
        COPY {schema}.{table}
        FROM '{s3_path}'
        IAM_ROLE '{iam_role}'
        FORMAT AS PARQUET
        ACCEPTINVCHARS AS '^'
        MAXERROR 100;
    """
    
    cur.execute(copy_sql)
    conn.commit()
    
    # Verify
    cur.execute(f"SELECT COUNT(*) FROM {schema}.{table}")
    count = cur.fetchone()[0]
    print(f"Loaded {count} rows into {schema}.{table}")
    
    # VACUUM and ANALYZE สำหรับ performance
    cur.execute(f"VACUUM {schema}.{table}")
    cur.execute(f"ANALYZE {schema}.{table}")
    
    cur.close()
    conn.close()
```

---

## 6. Apache Airflow: Workflow Orchestration

### 6.1 Installation ด้วย Docker

```yaml
# docker-compose-airflow.yml
version: '3.8'

x-airflow-common:
  &airflow-common
  image: apache/airflow:2.7.0
  environment:
    &airflow-common-env
    AIRFLOW__CORE__EXECUTOR: CeleryExecutor
    AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@postgres/airflow
    AIRFLOW__CELERY__RESULT_BACKEND: db+postgresql://airflow:airflow@postgres/airflow
    AIRFLOW__CELERY__BROKER_URL: redis://:@redis:6379/0
    AIRFLOW__CORE__FERNET_KEY: ''
    AIRFLOW__CORE__DAGS_ARE_PAUSED_AT_CREATION: 'true'
    AIRFLOW__CORE__LOAD_EXAMPLES: 'false'
    AIRFLOW__API__AUTH_BACKENDS: 'airflow.api.auth.backend.basic_auth'
    AWS_ACCESS_KEY_ID: ${AWS_ACCESS_KEY_ID}
    AWS_SECRET_ACCESS_KEY: ${AWS_SECRET_ACCESS_KEY}
    DATABASE_URL: ${DATABASE_URL}
  volumes:
    - ./dags:/opt/airflow/dags
    - ./logs:/opt/airflow/logs
    - ./plugins:/opt/airflow/plugins

services:
  airflow-webserver:
    <<: *airflow-common
    command: webserver
    ports:
      - 8080:8080

  airflow-scheduler:
    <<: *airflow-common
    command: scheduler

  airflow-worker:
    <<: *airflow-common
    command: celery worker
```

### 6.2 DAG: PostgreSQL → S3 → Redshift

```python
# dags/orders_pipeline_dag.py
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.providers.postgres.hooks.postgres import PostgresHook
from airflow.providers.amazon.aws.hooks.s3 import S3Hook
from airflow.providers.amazon.aws.transfers.s3_to_redshift import S3ToRedshiftOperator
from datetime import datetime, timedelta
import pandas as pd
from io import BytesIO
import pyarrow as pa
import pyarrow.parquet as pq

# DAG defaults
default_args = {
    'owner': 'data-team',
    'depends_on_past': False,
    'start_date': datetime(2024, 1, 1),
    'email_on_failure': True,
    'email_on_retry': False,
    'retries': 3,
    'retry_delay': timedelta(minutes=5),
    'execution_timeout': timedelta(hours=2)
}

# DAG definition
with DAG(
    'orders_data_pipeline',
    default_args=default_args,
    description='Extract orders from PostgreSQL, transform, load to S3 and Redshift',
    schedule_interval='0 2 * * *',  # Run daily at 2am
    catchup=False,
    tags=['orders', 'pipeline', 'daily']
) as dag:
    
    def extract_orders(**context):
        """Extract orders from PostgreSQL"""
        execution_date = context['execution_date']
        yesterday = execution_date - timedelta(days=1)
        
        pg_hook = PostgresHook(postgres_conn_id='postgres_prod')
        
        sql = """
            SELECT 
                o.id,
                o.user_id,
                o.product_id,
                o.quantity,
                o.amount,
                o.status,
                o.created_at,
                o.updated_at,
                u.email as user_email,
                u.country as user_country
            FROM orders o
            JOIN users u ON o.user_id = u.id
            WHERE o.created_at >= %s 
              AND o.created_at < %s
        """
        
        conn = pg_hook.get_conn()
        df = pd.read_sql(sql, conn, params=(
            yesterday.date(),
            execution_date.date()
        ))
        conn.close()
        
        print(f"Extracted {len(df)} orders for {yesterday.date()}")
        
        # เก็บ path ใน XCom
        date_str = yesterday.strftime('%Y/%m/%d')
        s3_path = f"raw/rds/orders/year={yesterday.year}/month={yesterday.month:02d}/day={yesterday.day:02d}/orders.parquet"
        
        # Upload to S3
        s3_hook = S3Hook(aws_conn_id='aws_default')
        
        buffer = BytesIO()
        table = pa.Table.from_pandas(df)
        pq.write_table(table, buffer, compression='snappy')
        buffer.seek(0)
        
        s3_hook.load_file_obj(
            buffer,
            key=s3_path,
            bucket_name='my-data-lake',
            replace=True
        )
        
        return s3_path
    
    def transform_orders(**context):
        """Transform orders in Spark"""
        s3_path = context['task_instance'].xcom_pull(task_ids='extract_orders')
        
        # ในการใช้จริง จะ submit Spark job ไปยัง EMR/Glue
        # สำหรับตัวอย่าง ทำ transform ด้วย pandas
        
        s3_hook = S3Hook(aws_conn_id='aws_default')
        
        # อ่านจาก S3
        s3_object = s3_hook.get_key(s3_path, bucket_name='my-data-lake')
        buffer = BytesIO(s3_object.get()['Body'].read())
        df = pd.read_parquet(buffer)
        
        # Transform
        df['amount_usd'] = df['amount'] / 33.0
        df['order_date'] = pd.to_datetime(df['created_at']).dt.date
        df['order_hour'] = pd.to_datetime(df['created_at']).dt.hour
        df['order_size'] = pd.cut(
            df['amount'],
            bins=[0, 500, 2000, 10000, float('inf')],
            labels=['small', 'medium', 'large', 'enterprise']
        ).astype(str)
        
        # Write processed
        execution_date = context['execution_date']
        yesterday = execution_date - timedelta(days=1)
        
        processed_path = f"processed/orders_enriched/order_date={yesterday.date()}/orders.parquet"
        
        buffer = BytesIO()
        table = pa.Table.from_pandas(df)
        pq.write_table(table, buffer, compression='snappy')
        buffer.seek(0)
        
        s3_hook.load_file_obj(
            buffer,
            key=processed_path,
            bucket_name='my-data-lake',
            replace=True
        )
        
        return processed_path
    
    def data_quality_check(**context):
        """ตรวจสอบ data quality"""
        s3_path = context['task_instance'].xcom_pull(task_ids='transform_orders')
        
        s3_hook = S3Hook(aws_conn_id='aws_default')
        s3_object = s3_hook.get_key(s3_path, bucket_name='my-data-lake')
        buffer = BytesIO(s3_object.get()['Body'].read())
        df = pd.read_parquet(buffer)
        
        # Checks
        assert len(df) > 0, "No data found after transformation"
        assert df['id'].is_unique, "Duplicate order IDs found"
        assert df['amount'].min() >= 0, "Negative amounts found"
        assert df['amount_usd'].notna().all(), "Null amount_usd found"
        
        null_pct = df.isnull().mean()
        for col in ['id', 'user_id', 'amount', 'status']:
            assert null_pct[col] == 0, f"Null values in required column {col}"
        
        print(f"✅ Data quality checks passed for {len(df)} records")
        return True
    
    # Task definitions
    t_extract = PythonOperator(
        task_id='extract_orders',
        python_callable=extract_orders
    )
    
    t_transform = PythonOperator(
        task_id='transform_orders',
        python_callable=transform_orders
    )
    
    t_quality = PythonOperator(
        task_id='data_quality_check',
        python_callable=data_quality_check
    )
    
    t_load = S3ToRedshiftOperator(
        task_id='load_to_redshift',
        schema='analytics',
        table='orders_enriched',
        s3_bucket='my-data-lake',
        s3_key='processed/orders_enriched/',
        redshift_conn_id='redshift_default',
        aws_conn_id='aws_default',
        copy_options=[
            'PARQUET',
            'ACCEPTINVCHARS AS \'^\''
        ],
        method='UPSERT',
        upsert_keys=['id']
    )
    
    # Task dependencies
    t_extract >> t_transform >> t_quality >> t_load
```

---

## 7. Self-Hosted Data Lake: MinIO + Spark + Trino

### 7.1 MinIO: S3-Compatible Object Storage

```yaml
# docker-compose-lakehouse.yml
version: '3.8'

services:
  # MinIO: S3-compatible object storage
  minio:
    image: minio/minio:latest
    ports:
      - "9000:9000"
      - "9001:9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    command: server /data --console-address ":9001"
    volumes:
      - minio_data:/data
    healthcheck:
      test: ["CMD", "mc", "ready", "local"]
      interval: 30s

  # Hive Metastore: catalog สำหรับ Spark/Trino
  hive-metastore:
    image: apache/hive:3.1.3
    environment:
      SERVICE_NAME: metastore
      DB_DRIVER: postgres
      HIVE_METASTORE_DB_TYPE: postgres
      SERVICE_OPTS: >-
        -Djavax.jdo.option.ConnectionDriverName=org.postgresql.Driver
        -Djavax.jdo.option.ConnectionURL=jdbc:postgresql://postgres:5432/metastore
        -Djavax.jdo.option.ConnectionUserName=hive
        -Djavax.jdo.option.ConnectionPassword=hive
    ports:
      - "9083:9083"
    depends_on:
      - postgres

  # Trino: SQL query engine
  trino:
    image: trinodb/trino:424
    ports:
      - "8080:8080"
    volumes:
      - ./trino/catalog:/etc/trino/catalog
    depends_on:
      - hive-metastore
      - minio

  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: metastore
      POSTGRES_USER: hive
      POSTGRES_PASSWORD: hive
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  minio_data:
  postgres_data:
```

```ini
# trino/catalog/hive.properties
connector.name=hive
hive.metastore.uri=thrift://hive-metastore:9083
hive.s3.endpoint=http://minio:9000
hive.s3.path-style-access=true
hive.s3.aws-access-key=minioadmin
hive.s3.aws-secret-key=minioadmin
hive.s3.ssl.enabled=false
hive.allow-drop-table=true
hive.parquet.use-column-names=true
```

### 7.2 Spark + MinIO

```python
# spark_job.py
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("DataLakeJob") \
    .config("spark.hadoop.fs.s3a.endpoint", "http://localhost:9000") \
    .config("spark.hadoop.fs.s3a.access.key", "minioadmin") \
    .config("spark.hadoop.fs.s3a.secret.key", "minioadmin") \
    .config("spark.hadoop.fs.s3a.path.style.access", "true") \
    .config("spark.hadoop.fs.s3a.impl", "org.apache.hadoop.fs.s3a.S3AFileSystem") \
    .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension") \
    .config("spark.sql.catalog.spark_catalog", 
            "org.apache.spark.sql.delta.catalog.DeltaCatalog") \
    .getOrCreate()

# อ่านจาก MinIO
df = spark.read.parquet("s3a://my-bucket/raw/orders/")

# เขียนเป็น Delta Lake (Lakehouse format)
df.write \
    .format("delta") \
    .mode("overwrite") \
    .partitionBy("order_date") \
    .save("s3a://my-bucket/delta/orders/")

# Time travel
spark.read \
    .format("delta") \
    .option("versionAsOf", 3) \
    .load("s3a://my-bucket/delta/orders/") \
    .show()
```

### 7.3 Trino SQL Query

```sql
-- Connect to Trino และ query ข้อมูล

-- สร้าง schema
CREATE SCHEMA hive.data_lake
WITH (location = 's3a://my-bucket/');

-- สร้าง external table ชี้ไป MinIO
CREATE TABLE hive.data_lake.orders (
    id VARCHAR,
    user_id VARCHAR,
    amount DOUBLE,
    status VARCHAR,
    created_at TIMESTAMP,
    order_date DATE
)
WITH (
    external_location = 's3a://my-bucket/processed/orders/',
    format = 'PARQUET',
    partitioned_by = ARRAY['order_date']
);

-- Query
SELECT
    order_date,
    status,
    COUNT(*) as order_count,
    SUM(amount) as revenue,
    AVG(amount) as avg_amount
FROM hive.data_lake.orders
WHERE order_date BETWEEN DATE '2024-09-01' AND DATE '2024-09-30'
GROUP BY 1, 2
ORDER BY 1, 2;

-- Cross-database query (Trino supports multiple catalogs)
-- Join PostgreSQL data กับ S3 data
SELECT
    o.id,
    o.amount,
    u.email,
    u.country
FROM hive.data_lake.orders o
JOIN postgresql.public.users u ON o.user_id = u.id
WHERE o.order_date = CURRENT_DATE - INTERVAL '1' DAY;
```

---

## 8. Data Governance

```python
# data_catalog/lineage.py
# Track data lineage: ข้อมูลมาจากไหน ผ่านการ transform อะไรบ้าง

class DataLineage:
    def __init__(self, db_conn):
        self.db = db_conn
    
    def record_job(self, job_name: str, inputs: list, outputs: list, 
                   row_count: int, metadata: dict = {}):
        """บันทึก data lineage"""
        self.db.execute("""
            INSERT INTO data_lineage (
                job_name, input_datasets, output_datasets,
                row_count, metadata, executed_at
            ) VALUES (%s, %s, %s, %s, %s, NOW())
        """, (job_name, inputs, outputs, row_count, metadata))
    
    def get_lineage(self, dataset: str) -> dict:
        """ดู lineage ของ dataset"""
        upstream = self.db.execute("""
            WITH RECURSIVE lineage AS (
                SELECT job_name, input_datasets, output_datasets, executed_at
                FROM data_lineage
                WHERE %s = ANY(output_datasets)
                UNION ALL
                SELECT d.job_name, d.input_datasets, d.output_datasets, d.executed_at
                FROM data_lineage d
                JOIN lineage l ON d.output_datasets && l.input_datasets
            )
            SELECT DISTINCT * FROM lineage
        """, (dataset,)).fetchall()
        
        return {'dataset': dataset, 'upstream': upstream}
```

---

## สรุป

Data Lake Architecture คือ foundation ของ modern data platform:

1. **Zone Design**: Bronze (raw) → Silver (clean) → Gold (curated) ทำให้ data flow ชัดเจน
2. **Format**: Parquet ให้ 10-100x speedup สำหรับ analytics queries
3. **Partitioning**: partition by date/region ช่วย query pruning อย่างมาก
4. **Orchestration**: Airflow จัดการ pipeline dependencies ได้ดี
5. **Self-hosted**: MinIO + Trino เป็น cost-effective alternative ของ AWS
6. **Governance**: Data lineage และ access control เป็นสิ่งสำคัญสำหรับ production

ในส่วนถัดไปจะพูดถึง OLAP vs OLTP Strategies ที่จะลงลึกถึง query engines เฉพาะทางเช่น ClickHouse

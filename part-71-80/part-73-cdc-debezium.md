# Part 73: Change Data Capture (CDC) ด้วย Debezium

## บทนำ

**Change Data Capture (CDC)** คือเทคนิคการจับการเปลี่ยนแปลงข้อมูลใน Database แบบ Real-time และส่งต่อไปยังระบบอื่น แทนที่จะ Query Database ซ้ำๆ เพื่อหาการเปลี่ยนแปลง CDC จะดักจับทุก INSERT, UPDATE, DELETE ทันทีที่เกิดขึ้น

**Debezium** เป็น Open-Source CDC Platform ที่สร้างบน Apache Kafka Connect ซึ่งเป็น Solution มาตรฐานในอุตสาหกรรมสำหรับการทำ Real-time Data Streaming

---

## 1. Use Cases ของ CDC

### 1.1 Real-time Sync to Elasticsearch

```
PostgreSQL (Source of Truth)
    │ CDC
    ▼
 Debezium → Kafka → Consumer → Elasticsearch
                               (Full-text Search Index)
```

### 1.2 Event Streaming สำหรับ Microservices

```
Order DB ─CDC─► Kafka ─► Inventory Service
                     ─► Notification Service
                     ─► Analytics Service
```

### 1.3 Audit Logs

```
ทุก INSERT/UPDATE/DELETE → Kafka → Audit Log Storage
```

### 1.4 Cache Invalidation

```
DB Update → CDC Event → Cache Service → Redis.del(key)
```

### 1.5 CQRS: Separate Read/Write Models

```
Command DB (Write) ─CDC─► Event Bus ─► Read DB (Optimize for Query)
```

---

## 2. PostgreSQL ต้องการ Configuration อะไร

### 2.1 WAL Level = Logical

PostgreSQL Write-Ahead Log (WAL) มีหลาย Level:
- **minimal**: เฉพาะ Crash recovery
- **replica**: สำหรับ Replication
- **logical**: บันทึก row-level changes (CDC ต้องการอันนี้)

```sql
-- ตรวจสอบ Current WAL Level
SHOW wal_level;

-- ตรวจสอบ max_replication_slots
SHOW max_replication_slots;

-- ตรวจสอบ max_wal_senders
SHOW max_wal_senders;
```

### 2.2 PostgreSQL Configuration

```ini
# postgresql.conf

# Enable logical replication
wal_level = logical

# Allow Debezium to create replication slots
max_replication_slots = 4        # Increase if multiple connectors

# Allow WAL senders (Debezium counts as one)
max_wal_senders = 4

# Keep WAL files until consumed
wal_keep_size = 1024             # 1GB

# Plugin for logical decoding
# pgoutput = built-in since PG10 (recommended)
# decoderbufs = older plugin (needs separate install)
```

### 2.3 pg_hba.conf สำหรับ Replication

```
# pg_hba.conf
# Allow Debezium user to create replication connections
host    replication     debezium_user    kafka-connect-host/32   md5
```

### 2.4 สร้าง Debezium User

```sql
-- สร้าง User สำหรับ Debezium
CREATE USER debezium_user WITH 
  PASSWORD 'debezium_password'
  REPLICATION      -- จำเป็น!
  LOGIN;

-- Grant access to tables
GRANT SELECT ON ALL TABLES IN SCHEMA public TO debezium_user;
GRANT USAGE ON SCHEMA public TO debezium_user;

-- สำหรับ pgoutput plugin: ต้อง Superuser หรือมี REPLICATION privilege
-- ถ้าใช้ PostgreSQL 14+: ไม่ต้องเป็น Superuser
```

---

## 3. Docker Compose: Full CDC Stack

```yaml
# docker-compose.cdc.yml
version: '3.8'

services:
  # ─── PostgreSQL ───────────────────────────────────────────
  postgres:
    image: postgres:15-alpine
    command: >
      postgres
      -c wal_level=logical
      -c max_replication_slots=4
      -c max_wal_senders=4
      -c wal_keep_size=1024
    environment:
      POSTGRES_DB: appdb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres123
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init-scripts:/docker-entrypoint-initdb.d
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d appdb"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ─── Zookeeper ────────────────────────────────────────────
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.3
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    volumes:
      - zookeeper_data:/var/lib/zookeeper/data
      - zookeeper_log:/var/lib/zookeeper/log
    healthcheck:
      test: ["CMD", "echo", "ruok", "|", "nc", "localhost", "2181"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ─── Apache Kafka ─────────────────────────────────────────
  kafka:
    image: confluentinc/cp-kafka:7.5.3
    depends_on:
      zookeeper:
        condition: service_healthy
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: 'true'
      KAFKA_LOG_RETENTION_HOURS: 168          # 7 days
      KAFKA_LOG_SEGMENT_BYTES: 1073741824     # 1GB
      KAFKA_COMPRESSION_TYPE: lz4
    volumes:
      - kafka_data:/var/lib/kafka/data
    ports:
      - "9092:9092"
    healthcheck:
      test: ["CMD", "kafka-broker-api-versions", "--bootstrap-server", "localhost:9092"]
      interval: 10s
      timeout: 5s
      retries: 10

  # ─── Schema Registry ──────────────────────────────────────
  schema-registry:
    image: confluentinc/cp-schema-registry:7.5.3
    depends_on:
      kafka:
        condition: service_healthy
    environment:
      SCHEMA_REGISTRY_HOST_NAME: schema-registry
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: kafka:29092
      SCHEMA_REGISTRY_LISTENERS: http://0.0.0.0:8081
    ports:
      - "8081:8081"

  # ─── Kafka Connect + Debezium ─────────────────────────────
  kafka-connect:
    image: debezium/connect:2.4
    depends_on:
      kafka:
        condition: service_healthy
      postgres:
        condition: service_healthy
    environment:
      BOOTSTRAP_SERVERS: kafka:29092
      GROUP_ID: debezium-connect-cluster
      CONFIG_STORAGE_TOPIC: debezium_configs
      OFFSET_STORAGE_TOPIC: debezium_offsets
      STATUS_STORAGE_TOPIC: debezium_statuses
      CONFIG_STORAGE_REPLICATION_FACTOR: 1
      OFFSET_STORAGE_REPLICATION_FACTOR: 1
      STATUS_STORAGE_REPLICATION_FACTOR: 1
      # JSON Converter (no schema)
      KEY_CONVERTER: org.apache.kafka.connect.json.JsonConverter
      VALUE_CONVERTER: org.apache.kafka.connect.json.JsonConverter
      CONNECT_KEY_CONVERTER_SCHEMAS_ENABLE: 'false'
      CONNECT_VALUE_CONVERTER_SCHEMAS_ENABLE: 'false'
      # Logging
      LOG_LEVEL: INFO
    ports:
      - "8083:8083"
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:8083/ || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 10
      start_period: 60s

  # ─── Elasticsearch ────────────────────────────────────────
  elasticsearch:
    image: elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    volumes:
      - elasticsearch_data:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:9200/_cluster/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 5

  # ─── Kibana ───────────────────────────────────────────────
  kibana:
    image: kibana:8.11.0
    depends_on:
      elasticsearch:
        condition: service_healthy
    environment:
      ELASTICSEARCH_HOSTS: http://elasticsearch:9200
    ports:
      - "5601:5601"

  # ─── Kafka UI ─────────────────────────────────────────────
  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    depends_on:
      kafka:
        condition: service_healthy
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:29092
      KAFKA_CLUSTERS_0_SCHEMAREGISTRY: http://schema-registry:8081
      KAFKA_CLUSTERS_0_KAFKACONNECT_0_NAME: debezium
      KAFKA_CLUSTERS_0_KAFKACONNECT_0_ADDRESS: http://kafka-connect:8083
    ports:
      - "8080:8080"

volumes:
  postgres_data:
  zookeeper_data:
  zookeeper_log:
  kafka_data:
  elasticsearch_data:
```

### 3.1 Init Scripts

```sql
-- init-scripts/01-init.sql

-- สร้าง Tables สำหรับ Demo
CREATE TABLE IF NOT EXISTS products (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(255) NOT NULL,
  description TEXT,
  price DECIMAL(10,2) NOT NULL,
  stock_quantity INTEGER NOT NULL DEFAULT 0,
  category VARCHAR(100),
  is_active BOOLEAN NOT NULL DEFAULT true,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS orders (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  customer_id UUID NOT NULL,
  status VARCHAR(50) NOT NULL DEFAULT 'PENDING',
  total_amount DECIMAL(10,2) NOT NULL,
  items JSONB NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS customers (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR(255) UNIQUE NOT NULL,
  full_name VARCHAR(255) NOT NULL,
  phone VARCHAR(20),
  address JSONB,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- สร้าง Debezium User
CREATE USER debezium_user WITH PASSWORD 'debezium_pass' REPLICATION LOGIN;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO debezium_user;
GRANT USAGE ON SCHEMA public TO debezium_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO debezium_user;

-- สร้าง Publication สำหรับ pgoutput plugin
CREATE PUBLICATION debezium_publication FOR ALL TABLES;
```

---

## 4. Register Debezium Connectors

### 4.1 PostgreSQL Connector สำหรับ Products

```bash
# register-connectors.sh

#!/bin/bash

CONNECT_URL="http://localhost:8083"

echo "Waiting for Kafka Connect..."
until curl -s ${CONNECT_URL}/ > /dev/null; do
  sleep 2
done
echo "Kafka Connect is ready!"

# Register PostgreSQL Connector
curl -X POST ${CONNECT_URL}/connectors \
  -H "Content-Type: application/json" \
  -d @- <<'EOF'
{
  "name": "postgres-cdc-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    
    "database.hostname": "postgres",
    "database.port": "5432",
    "database.user": "debezium_user",
    "database.password": "debezium_pass",
    "database.dbname": "appdb",
    "database.server.name": "appdb",
    
    "plugin.name": "pgoutput",
    "publication.name": "debezium_publication",
    "slot.name": "debezium_replication_slot",
    
    "table.include.list": "public.products,public.orders,public.customers",
    
    "key.converter": "org.apache.kafka.connect.json.JsonConverter",
    "key.converter.schemas.enable": "false",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter.schemas.enable": "false",
    
    "heartbeat.interval.ms": "10000",
    "heartbeat.action.query": "UPDATE heartbeat SET ts = NOW()",
    
    "decimal.handling.mode": "double",
    "time.precision.mode": "connect",
    "tombstones.on.delete": "true",
    
    "snapshot.mode": "initial",
    "snapshot.locking.mode": "minimal",
    
    "max.queue.size": "16384",
    "max.batch.size": "2048",
    "poll.interval.ms": "1000"
  }
}
EOF

echo "Connector registered!"

# ตรวจสอบสถานะ
sleep 5
curl ${CONNECT_URL}/connectors/postgres-cdc-connector/status
```

### 4.2 ตรวจสอบ Topics ที่ถูกสร้าง

```bash
# Topics ที่ Debezium สร้างอัตโนมัติ:
# appdb.public.products    ← Changes จาก products table
# appdb.public.orders      ← Changes จาก orders table
# appdb.public.customers   ← Changes จาก customers table

# ดู Messages ใน Topic
docker exec -it kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic appdb.public.products \
  --from-beginning \
  --max-messages 5
```

---

## 5. Debezium Change Event Format

### 5.1 INSERT Event

```json
{
  "before": null,
  "after": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "name": "iPhone 15 Pro",
    "description": "Apple iPhone 15 Pro 256GB",
    "price": 45900.00,
    "stock_quantity": 100,
    "category": "smartphones",
    "is_active": true,
    "created_at": "2024-01-15T10:30:00.000Z",
    "updated_at": "2024-01-15T10:30:00.000Z"
  },
  "source": {
    "version": "2.4.0.Final",
    "connector": "postgresql",
    "name": "appdb",
    "ts_ms": 1705316200000,
    "snapshot": "false",
    "db": "appdb",
    "sequence": "[\"24404296\",\"24404296\"]",
    "schema": "public",
    "table": "products",
    "txId": 492,
    "lsn": 24404296,
    "xmin": null
  },
  "op": "c",         // c=create, u=update, d=delete, r=read(snapshot)
  "ts_ms": 1705316200123,
  "transaction": null
}
```

### 5.2 UPDATE Event

```json
{
  "before": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "stock_quantity": 100,
    "price": 45900.00
  },
  "after": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "stock_quantity": 95,
    "price": 44900.00
  },
  "source": { "...": "..." },
  "op": "u",
  "ts_ms": 1705316300000
}
```

### 5.3 DELETE Event

```json
{
  "before": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "name": "iPhone 15 Pro"
  },
  "after": null,
  "source": { "...": "..." },
  "op": "d",
  "ts_ms": 1705316400000
}
```

---

## 6. Consumer: Sync Changes to Elasticsearch

### 6.1 Setup

```bash
npm install kafkajs @elastic/elasticsearch
npm install @types/node typescript ts-node
```

### 6.2 Elasticsearch Sync Consumer

```typescript
// src/consumers/elasticsearch-sync.consumer.ts
import { Kafka, Consumer, EachMessagePayload } from 'kafkajs';
import { Client as ElasticsearchClient } from '@elastic/elasticsearch';
import { Logger } from '../utils/logger';

interface DebeziumEvent {
  before: Record<string, any> | null;
  after: Record<string, any> | null;
  source: {
    schema: string;
    table: string;
    ts_ms: number;
  };
  op: 'c' | 'u' | 'd' | 'r';
  ts_ms: number;
}

export class ElasticsearchSyncConsumer {
  private kafka: Kafka;
  private consumer: Consumer;
  private esClient: ElasticsearchClient;
  private readonly logger = new Logger('ElasticsearchSync');

  private readonly tableIndexMap: Record<string, string> = {
    'products': 'products',
    'orders': 'orders',
    'customers': 'customers',
  };

  constructor() {
    this.kafka = new Kafka({
      clientId: 'elasticsearch-sync',
      brokers: process.env.KAFKA_BROKERS?.split(',') || ['localhost:9092'],
    });

    this.consumer = this.kafka.consumer({
      groupId: 'elasticsearch-sync-group',
      sessionTimeout: 30000,
      heartbeatInterval: 3000,
    });

    this.esClient = new ElasticsearchClient({
      node: process.env.ELASTICSEARCH_URL || 'http://localhost:9200',
    });
  }

  async start(): Promise<void> {
    await this.consumer.connect();
    this.logger.info('Consumer connected to Kafka');

    // Subscribe to all CDC topics
    await this.consumer.subscribe({
      topics: [
        'appdb.public.products',
        'appdb.public.orders',
        'appdb.public.customers',
      ],
      fromBeginning: false,
    });

    await this.ensureIndices();

    await this.consumer.run({
      eachBatchAutoResolve: false,
      eachBatch: async ({ batch, resolveOffset, heartbeat, isRunning, isStale }) => {
        for (const message of batch.messages) {
          if (!isRunning() || isStale()) break;

          try {
            await this.processMessage(batch.topic, message);
            resolveOffset(message.offset);
            await heartbeat();
          } catch (error) {
            this.logger.error(`Failed to process message: ${error.message}`);
            // Don't resolve offset - will retry
          }
        }
      },
    });
  }

  private async processMessage(topic: string, message: any): Promise<void> {
    if (!message.value) {
      // Tombstone message (soft delete marker)
      return;
    }

    const event: DebeziumEvent = JSON.parse(message.value.toString());
    const tableName = topic.split('.').pop()!;
    const indexName = this.tableIndexMap[tableName];

    if (!indexName) {
      this.logger.warn(`No index mapping for table: ${tableName}`);
      return;
    }

    switch (event.op) {
      case 'c':
      case 'r':
        await this.handleInsert(indexName, event.after!);
        break;

      case 'u':
        await this.handleUpdate(indexName, event.after!);
        break;

      case 'd':
        await this.handleDelete(indexName, event.before!);
        break;
    }

    this.logger.debug(`Synced ${event.op} on ${tableName} id=${event.after?.id || event.before?.id}`);
  }

  private async handleInsert(index: string, data: Record<string, any>): Promise<void> {
    await this.esClient.index({
      index,
      id: data.id,
      document: this.transformDocument(data),
      refresh: false, // Async refresh for performance
    });
  }

  private async handleUpdate(index: string, data: Record<string, any>): Promise<void> {
    await this.esClient.update({
      index,
      id: data.id,
      doc: this.transformDocument(data),
      doc_as_upsert: true, // Create if not exists
      retry_on_conflict: 3,
    });
  }

  private async handleDelete(index: string, data: Record<string, any>): Promise<void> {
    await this.esClient.delete({
      index,
      id: data.id,
      ignore_unavailable: true,
    });
  }

  private transformDocument(data: Record<string, any>): Record<string, any> {
    // Convert PostgreSQL types to Elasticsearch-friendly format
    return Object.entries(data).reduce((acc, [key, value]) => {
      if (value === null || value === undefined) {
        return acc; // Skip null values
      }
      if (typeof value === 'string' && /^\d{4}-\d{2}-\d{2}/.test(value)) {
        acc[key] = new Date(value).toISOString(); // Normalize dates
      } else {
        acc[key] = value;
      }
      return acc;
    }, {} as Record<string, any>);
  }

  private async ensureIndices(): Promise<void> {
    // Products index with optimized mappings
    await this.esClient.indices.create({
      index: 'products',
      body: {
        settings: {
          number_of_shards: 1,
          number_of_replicas: 0,
          analysis: {
            analyzer: {
              thai_analyzer: {
                type: 'custom',
                tokenizer: 'standard',
                filter: ['lowercase', 'thai_stop'],
              },
            },
          },
        },
        mappings: {
          properties: {
            id: { type: 'keyword' },
            name: {
              type: 'text',
              analyzer: 'thai_analyzer',
              fields: { keyword: { type: 'keyword' } },
            },
            description: { type: 'text', analyzer: 'thai_analyzer' },
            price: { type: 'scaled_float', scaling_factor: 100 },
            stock_quantity: { type: 'integer' },
            category: { type: 'keyword' },
            is_active: { type: 'boolean' },
            created_at: { type: 'date' },
            updated_at: { type: 'date' },
          },
        },
      },
    }).catch((err) => {
      if (err.meta?.statusCode !== 400) throw err; // 400 = already exists
    });

    this.logger.info('Elasticsearch indices ready');
  }

  async stop(): Promise<void> {
    await this.consumer.disconnect();
    await this.esClient.close();
  }
}

// Main
const consumer = new ElasticsearchSyncConsumer();

process.on('SIGTERM', async () => {
  await consumer.stop();
  process.exit(0);
});

consumer.start().catch(console.error);
```

---

## 7. Transformations (SMT): Single Message Transformations

### 7.1 ExtractNewRecordState

แทนที่จะ Process full Debezium envelope สามารถ Flatten เพื่อเอาแค่ `after` field ได้:

```json
{
  "name": "postgres-cdc-flattened",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "...": "...",
    
    "transforms": "unwrap",
    "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
    "transforms.unwrap.drop.tombstones": "false",
    "transforms.unwrap.delete.handling.mode": "rewrite",
    "transforms.unwrap.add.fields": "op,table,source.ts_ms",
    "transforms.unwrap.add.headers": "op"
  }
}
```

ผลลัพธ์หลัง SMT:
```json
{
  "id": "550e8400-...",
  "name": "iPhone 15 Pro",
  "price": 44900.00,
  "stock_quantity": 95,
  "__op": "u",
  "__table": "products",
  "__source_ts_ms": 1705316300000
}
```

### 7.2 Routing Transformation

```json
{
  "transforms": "route",
  "transforms.route.type": "org.apache.kafka.connect.transforms.ReplaceField$Value",
  "transforms.route.renames": "id:document_id"
}
```

---

## 8. Avro Serialization ด้วย Schema Registry

### 8.1 Connector Configuration ด้วย Avro

```json
{
  "name": "postgres-cdc-avro",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "...": "...",
    
    "key.converter": "io.confluent.kafka.serializers.KafkaAvroSerializer",
    "key.converter.schema.registry.url": "http://schema-registry:8081",
    "value.converter": "io.confluent.kafka.serializers.KafkaAvroSerializer",
    "value.converter.schema.registry.url": "http://schema-registry:8081"
  }
}
```

### 8.2 Avro Consumer ใน Node.js

```typescript
// src/consumers/avro-consumer.ts
import { Kafka } from 'kafkajs';
import { SchemaRegistry } from '@kafkajs/confluent-schema-registry';

const registry = new SchemaRegistry({
  host: 'http://localhost:8081',
});

const consumer = kafka.consumer({ groupId: 'avro-consumer' });

await consumer.run({
  eachMessage: async ({ message }) => {
    const decodedKey = await registry.decode(message.key);
    const decodedValue = await registry.decode(message.value);
    
    console.log('Key:', decodedKey);
    console.log('Value:', decodedValue);
  },
});
```

---

## 9. Monitoring Debezium

### 9.1 Kafka Connect REST API

```bash
# ดูรายการ Connectors
curl http://localhost:8083/connectors

# ดู Status ของ Connector
curl http://localhost:8083/connectors/postgres-cdc-connector/status

# ดู Config
curl http://localhost:8083/connectors/postgres-cdc-connector/config

# Pause Connector
curl -X PUT http://localhost:8083/connectors/postgres-cdc-connector/pause

# Resume Connector
curl -X PUT http://localhost:8083/connectors/postgres-cdc-connector/resume

# Restart Connector
curl -X POST http://localhost:8083/connectors/postgres-cdc-connector/restart
```

### 9.2 Monitor Replication Lag

```sql
-- ตรวจสอบ Replication Slots
SELECT 
  slot_name,
  plugin,
  slot_type,
  active,
  active_pid,
  pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn)) AS lag_size,
  pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn) AS lag_bytes
FROM pg_replication_slots
WHERE slot_type = 'logical';

-- ตรวจสอบ WAL Sender
SELECT 
  application_name,
  state,
  sent_lsn,
  write_lsn,
  flush_lsn,
  replay_lsn,
  pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn)) AS lag
FROM pg_stat_replication;
```

### 9.3 Prometheus Metrics ผ่าน JMX

```yaml
# kafka-connect JMX Prometheus exporter
- pattern: 'debezium.postgres<type=connector-metrics, context=snapshot, server=(.+)><>(.+)'
  name: debezium_postgres_$2
  labels:
    server: "$1"

- pattern: 'debezium.postgres<type=connector-metrics, context=streaming, server=(.+)><>(.+)'
  name: debezium_postgres_streaming_$2
  labels:
    server: "$1"
```

---

## 10. Dead Letter Queue (DLQ)

```json
{
  "name": "postgres-cdc-with-dlq",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "...": "...",
    
    "errors.tolerance": "all",
    "errors.log.enable": "true",
    "errors.log.include.messages": "true",
    "errors.deadletterqueue.topic.name": "dead-letter-cdc",
    "errors.deadletterqueue.context.headers.enable": "true",
    "errors.deadletterqueue.topic.replication.factor": "1"
  }
}
```

---

## 11. Cache Invalidation ด้วย CDC

```typescript
// src/consumers/cache-invalidation.consumer.ts
import { Kafka } from 'kafkajs';
import Redis from 'ioredis';

const redis = new Redis(process.env.REDIS_URL);

const consumer = kafka.consumer({ groupId: 'cache-invalidation' });

await consumer.subscribe({
  topics: ['appdb.public.products', 'appdb.public.orders'],
});

await consumer.run({
  eachMessage: async ({ topic, message }) => {
    if (!message.value) return;
    
    const event = JSON.parse(message.value.toString());
    const tableName = topic.split('.').pop();
    const id = event.after?.id || event.before?.id;

    if (!id) return;

    switch (tableName) {
      case 'products':
        // Invalidate product cache
        await redis.del(`product:${id}`);
        await redis.del(`products:list:*`); // Pattern delete (use carefully)
        
        // If price changed, invalidate price cache
        if (event.before?.price !== event.after?.price) {
          await redis.del(`product:price:${id}`);
        }
        break;

      case 'orders':
        await redis.del(`order:${id}`);
        if (event.after?.customer_id) {
          await redis.del(`customer:orders:${event.after.customer_id}`);
        }
        break;
    }

    console.log(`Cache invalidated for ${tableName}:${id}`);
  },
});
```

---

## 12. Advanced: Filtering Events

### 12.1 Groovy Script Filter (ใน Debezium)

```json
{
  "transforms": "filter",
  "transforms.filter.type": "io.debezium.transforms.Filter",
  "transforms.filter.language": "jsr223.groovy",
  "transforms.filter.condition": "value.op != 'r'"
}
```

### 12.2 Node.js Consumer-Side Filter

```typescript
// src/consumers/filtered-consumer.ts
await consumer.run({
  eachMessage: async ({ topic, message }) => {
    const event = JSON.parse(message.value?.toString() || '{}');
    
    // Skip snapshot events
    if (event.op === 'r') return;
    
    // Only process active products
    if (event.after && !event.after.is_active) return;
    
    // Skip non-significant updates (e.g., only updated_at changed)
    if (event.op === 'u' && event.before && event.after) {
      const changed = Object.keys(event.after).filter(
        key => key !== 'updated_at' && event.before[key] !== event.after[key]
      );
      if (changed.length === 0) return;
    }
    
    await processEvent(event);
  },
});
```

---

## 13. Full Stack Test Script

```typescript
// scripts/test-cdc.ts

import { Pool } from 'pg';
import { Client as ElasticsearchClient } from '@elastic/elasticsearch';

const pg = new Pool({ connectionString: 'postgresql://postgres:postgres123@localhost:5432/appdb' });
const es = new ElasticsearchClient({ node: 'http://localhost:9200' });

async function testCDC() {
  console.log('=== Testing CDC Pipeline ===\n');

  // 1. Insert product
  console.log('1. Inserting product into PostgreSQL...');
  const { rows: [product] } = await pg.query(
    `INSERT INTO products (name, description, price, stock_quantity, category)
     VALUES ($1, $2, $3, $4, $5) RETURNING *`,
    ['Test Product', 'CDC Test Item', 999.99, 50, 'electronics']
  );
  console.log(`   Created: ${product.id}`);

  // 2. Wait for CDC propagation
  console.log('2. Waiting for CDC propagation (5 seconds)...');
  await new Promise(resolve => setTimeout(resolve, 5000));

  // 3. Check Elasticsearch
  console.log('3. Checking Elasticsearch...');
  const esResult = await es.get({ index: 'products', id: product.id });
  console.log(`   Found in ES: ${JSON.stringify(esResult._source)}`);

  // 4. Update product
  console.log('4. Updating product price...');
  await pg.query(
    'UPDATE products SET price = $1, updated_at = NOW() WHERE id = $2',
    [899.99, product.id]
  );

  await new Promise(resolve => setTimeout(resolve, 3000));

  const updatedEs = await es.get({ index: 'products', id: product.id });
  console.log(`   Updated price in ES: ${(updatedEs._source as any).price}`);

  // 5. Delete product
  console.log('5. Deleting product...');
  await pg.query('DELETE FROM products WHERE id = $1', [product.id]);

  await new Promise(resolve => setTimeout(resolve, 3000));

  try {
    await es.get({ index: 'products', id: product.id });
    console.log('   ERROR: Product still in ES!');
  } catch (err: any) {
    if (err.meta?.statusCode === 404) {
      console.log('   Deleted from ES');
    }
  }

  console.log('\n=== CDC Test Complete ===');
  await pg.end();
  await es.close();
}

testCDC().catch(console.error);
```

---

## 14. Troubleshooting

### 14.1 Replication Slot ค้าง

```sql
-- ถ้า Debezium ถูกหยุดนาน Replication Slot จะสะสม WAL
-- ตรวจสอบ Lag
SELECT slot_name, pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn))
FROM pg_replication_slots;

-- ลบ Slot ที่ไม่ใช้งาน (ระวัง! ทำให้ Debezium ต้อง Snapshot ใหม่)
SELECT pg_drop_replication_slot('debezium_replication_slot');
```

### 14.2 Connector Error

```bash
# ดู Error ใน Connector
curl http://localhost:8083/connectors/postgres-cdc-connector/status | jq '.tasks[].trace'

# Restart failed task
curl -X POST http://localhost:8083/connectors/postgres-cdc-connector/tasks/0/restart
```

### 14.3 WAL Level ไม่ถูกต้อง

```
Error: "Replication is not allowed for a user with NOSUPERUSER role."
```

```sql
-- Fix: Grant Replication to user
ALTER USER debezium_user REPLICATION;
```

---

## 15. สรุป

### 15.1 CDC Use Case Matrix

| Use Case | CDC with Debezium | Traditional Polling |
|----------|-------------------|---------------------|
| Latency | Milliseconds | Seconds-Minutes |
| DB Load | Minimal (WAL reading) | High (SELECT queries) |
| Missed Changes | None | Possible |
| Setup Complexity | High (Kafka Stack) | Low |
| Scale | Excellent | Poor at high volume |

### 15.2 When to Use CDC

- ต้องการ Real-time Data Sync
- ต้องการ Audit Log ของทุก Change
- กำลัง Migrate จาก Monolith สู่ Microservices
- ต้องการ Cache Invalidation แบบ Real-time
- ต้องการ CQRS Architecture

### 15.3 Debezium Best Practices

1. **Monitor Replication Slot Lag**: ถ้า Lag สูง = WAL สะสม = Disk เต็ม
2. **Set WAL Retention**: ป้องกัน WAL ถูก Delete ก่อน Debezium อ่าน
3. **Use Heartbeat**: ป้องกัน Slot Lag ในช่วงไม่มี Activity
4. **DLQ for Error Handling**: อย่า Drop Events โดยไม่มี DLQ
5. **Schema Evolution**: ใช้ Schema Registry สำหรับ Avro

Debezium เป็น Foundation ของ Real-time Data Architecture ที่ทรงพลัง เหมาะสำหรับทุกระบบที่ต้องการให้ Data เดินทางจาก PostgreSQL ไปยังระบบอื่นแบบ Real-time และ Reliable

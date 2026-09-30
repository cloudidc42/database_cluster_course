# Part 82: Kafka Integration กับ Database Cluster

## บทนำ

Apache Kafka คือ distributed event streaming platform ที่ออกแบบมาสำหรับ high-throughput, fault-tolerant, publish-subscribe messaging ที่ LinkedIn สร้างขึ้นในปี 2011 ปัจจุบันกลายเป็น backbone ของ modern data architecture ในองค์กรระดับโลก บทนี้จะครอบคลุมทุกอย่างตั้งแต่ concepts พื้นฐานถึงการ integrate กับ database cluster

---

## 1. Apache Kafka: Distributed Event Streaming Platform

### 1.1 ทำไมต้องใช้ Kafka?

**ปัญหาที่ Kafka แก้ได้:**
- Microservices ต้องการสื่อสารกันโดยไม่ coupling กัน
- ต้องการเก็บ event history ย้อนหลัง
- Multiple consumers ต้องการอ่าน event เดียวกัน
- ต้องการ replay events เมื่อระบบมีปัญหา
- High throughput (millions of events/second)

**Use cases ที่ Kafka เหมาะ:**
- Event streaming (user clicks, page views)
- Log aggregation จากหลาย services
- Database change data capture (CDC)
- Real-time analytics pipeline
- Microservice communication via events
- IoT data ingestion

### 1.2 Kafka vs ระบบอื่น

| Feature | Kafka | Redis Pub/Sub | RabbitMQ | SQS |
|---------|-------|---------------|----------|-----|
| Durability | ✅ เก็บไว้ตาม retention | ❌ fire-and-forget | ✅ durable queues | ✅ |
| Replay | ✅ ย้อน offset ได้ | ❌ | ❌ | ❌ |
| Throughput | สูงมาก (millions/s) | สูง (100k+/s) | ปานกลาง | ปานกลาง |
| Ordering | ✅ per partition | ❌ | ✅ per queue | ❌ |
| Consumer Groups | ✅ | ❌ | ✅ | ✅ |
| Retention | กำหนดได้ (วัน/GB) | ไม่มี | จนกว่าจะ consume | 14 วัน |
| Complex routing | ❌ (simple) | ❌ | ✅ exchanges | ❌ |

---

## 2. Kafka Concepts

### 2.1 Core Components

```
┌─────────────────────────────────────────────────────────────────┐
│                         Kafka Cluster                           │
│                                                                 │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐          │
│  │   Broker 1  │   │   Broker 2  │   │   Broker 3  │          │
│  │             │   │             │   │             │          │
│  │ Topic: orders                                               │
│  │ ┌─────────┐ │   ┌─────────┐   │   ┌─────────┐            │
│  │ │ Part 0  │ │   │ Part 1  │   │   │ Part 2  │            │
│  │ │[L]      │ │   │[L]      │   │   │[L]      │            │
│  │ │[R] Br2  │ │   │[R] Br3  │   │   │[R] Br1  │            │
│  │ └─────────┘ │   └─────────┘   │   └─────────┘            │
│  └─────────────┘   └─────────────┘   └─────────────┘          │
│        [L] = Leader    [R] = Replica                           │
└─────────────────────────────────────────────────────────────────┘
         ↑ Producer                        ↓ Consumer Group
    ┌─────────┐                      ┌─────────────────────┐
    │ Service │                      │ Consumer 1 (Part 0) │
    │ (Order) │                      │ Consumer 2 (Part 1) │
    └─────────┘                      │ Consumer 3 (Part 2) │
                                     └─────────────────────┘
```

### 2.2 Topic และ Partition

**Topic:**
- Named stream of records (เหมือน "category" หรือ "feed")
- แบ่งเป็น partitions เพื่อ horizontal scaling

**Partition:**
- Ordered, immutable sequence of records
- แต่ละ record มี offset (monotonically increasing)
- Replication สำหรับ fault tolerance

```
Topic: orders
├── Partition 0: [offset 0: order-1][offset 1: order-3][offset 2: order-5]
├── Partition 1: [offset 0: order-2][offset 1: order-4][offset 2: order-6]
└── Partition 2: [offset 0: order-7][offset 1: order-8][offset 2: order-9]
```

**Key-based partitioning:**
```
ถ้า key = user_id, messages ของ user เดียวกัน → partition เดียวกัน
→ ordering guarantee per user
```

### 2.3 Producer, Consumer, Broker

**Producer:**
- ส่ง records ไปยัง topic
- เลือก partition ด้วย: key hash, round-robin, custom

**Consumer:**
- อ่าน records จาก topic
- Track offset ของตัวเอง
- Pull-based (consumer เป็นคน pull)

**Consumer Group:**
- หลาย consumers อ่าน topic เดียวกัน
- แต่ละ partition ถูก consumed โดย consumer เดียวใน group
- ถ้า consumers > partitions → บาง consumers idle
- Scaling: เพิ่ม partitions → เพิ่ม consumers

**Broker:**
- Server node ใน Kafka cluster
- เก็บ partitions
- Handle read/write requests

### 2.4 Offset Management

```
Consumer Group "order-processor" อ่าน Topic "orders":

Partition 0: [...][offset 5][offset 6][offset 7] ← committed offset: 6
Partition 1: [...][offset 3][offset 4][offset 5] ← committed offset: 3

เมื่อ consumer restart → กลับมาอ่านต่อจาก committed offset
```

**Auto commit vs Manual commit:**
```javascript
// Auto commit (ง่าย แต่อาจเสีย messages)
consumer.run({
  autoCommit: true,
  autoCommitInterval: 5000,
  eachMessage: async ({ message }) => {
    await processMessage(message);
    // ถ้า process fail ก่อน commit → message หาย
  }
});

// Manual commit (recommended สำหรับ production)
consumer.run({
  autoCommit: false,
  eachMessage: async ({ topic, partition, message }) => {
    await processMessage(message);
    // Commit หลังจาก process สำเร็จ
    await consumer.commitOffsets([{
      topic,
      partition,
      offset: (parseInt(message.offset) + 1).toString()
    }]);
  }
});
```

### 2.5 Replication Factor

```
replication.factor = 3 หมายความว่า:
- แต่ละ partition มี 1 leader + 2 replicas
- ทนต่อ broker failure ได้ 2 ตัว
- ค่า default สำหรับ production: 3
```

**Retention:**
```
- Time-based: log.retention.hours=168 (7 วัน)
- Size-based: log.retention.bytes=1073741824 (1 GB)
- ทั้งสอง: ใช้อันที่ถึงก่อน
```

---

## 3. KRaft: Kafka without ZooKeeper

ตั้งแต่ Kafka 3.3+, Kafka ใช้ KRaft (Kafka Raft) แทน ZooKeeper เพื่อ metadata management

**ข้อดีของ KRaft:**
- ไม่ต้องดูแล ZooKeeper cluster แยก
- Faster startup และ shutdown
- Single mode (ไม่ต้องมี ZooKeeper สำหรับ development)
- Better scalability

---

## 4. Installation ด้วย Docker Compose

### 4.1 Kafka + KRaft Mode

```yaml
# docker-compose.yml
version: '3.8'

services:
  kafka:
    image: confluentinc/cp-kafka:7.6.0
    hostname: kafka
    container_name: kafka
    ports:
      - "9092:9092"
      - "9101:9101"
    environment:
      # KRaft settings
      KAFKA_NODE_ID: 1
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: 'CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT'
      KAFKA_ADVERTISED_LISTENERS: 'PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092'
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_JMX_PORT: 9101
      KAFKA_JMX_HOSTNAME: localhost
      KAFKA_PROCESS_ROLES: 'broker,controller'
      KAFKA_CONTROLLER_QUORUM_VOTERS: '1@kafka:29093'
      KAFKA_LISTENERS: 'PLAINTEXT://kafka:29092,CONTROLLER://kafka:29093,PLAINTEXT_HOST://0.0.0.0:9092'
      KAFKA_INTER_BROKER_LISTENER_NAME: 'PLAINTEXT'
      KAFKA_CONTROLLER_LISTENER_NAMES: 'CONTROLLER'
      KAFKA_LOG_DIRS: '/tmp/kraft-combined-logs'
      CLUSTER_ID: 'MkU3OEVBNTcwNTJENDM2Qk'
      
      # Retention
      KAFKA_LOG_RETENTION_HOURS: 168
      KAFKA_LOG_RETENTION_BYTES: 1073741824
      
      # Performance
      KAFKA_NUM_PARTITIONS: 3
      KAFKA_DEFAULT_REPLICATION_FACTOR: 1
    volumes:
      - kafka_data:/var/lib/kafka/data

  # Kafka UI สำหรับ monitoring
  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    container_name: kafka-ui
    ports:
      - "8090:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:29092
    depends_on:
      - kafka

  # Schema Registry
  schema-registry:
    image: confluentinc/cp-schema-registry:7.6.0
    hostname: schema-registry
    container_name: schema-registry
    depends_on:
      - kafka
    ports:
      - "8081:8081"
    environment:
      SCHEMA_REGISTRY_HOST_NAME: schema-registry
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: 'kafka:29092'
      SCHEMA_REGISTRY_LISTENERS: http://0.0.0.0:8081

volumes:
  kafka_data:
```

### 4.2 Multi-broker Production Setup

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  kafka-1:
    image: confluentinc/cp-kafka:7.6.0
    hostname: kafka-1
    ports:
      - "9092:9092"
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: 'broker,controller'
      KAFKA_CONTROLLER_QUORUM_VOTERS: '1@kafka-1:29093,2@kafka-2:29093,3@kafka-3:29093'
      KAFKA_LISTENERS: 'PLAINTEXT://kafka-1:29092,CONTROLLER://kafka-1:29093,PLAINTEXT_HOST://0.0.0.0:9092'
      KAFKA_ADVERTISED_LISTENERS: 'PLAINTEXT://kafka-1:29092,PLAINTEXT_HOST://kafka-1:9092'
      KAFKA_DEFAULT_REPLICATION_FACTOR: 3
      KAFKA_MIN_INSYNC_REPLICAS: 2
      CLUSTER_ID: 'MkU3OEVBNTcwNTJENDM2Qk'
    volumes:
      - kafka1_data:/var/lib/kafka/data

  kafka-2:
    image: confluentinc/cp-kafka:7.6.0
    hostname: kafka-2
    ports:
      - "9093:9092"
    environment:
      KAFKA_NODE_ID: 2
      KAFKA_PROCESS_ROLES: 'broker,controller'
      KAFKA_CONTROLLER_QUORUM_VOTERS: '1@kafka-1:29093,2@kafka-2:29093,3@kafka-3:29093'
      KAFKA_LISTENERS: 'PLAINTEXT://kafka-2:29092,CONTROLLER://kafka-2:29093,PLAINTEXT_HOST://0.0.0.0:9092'
      KAFKA_ADVERTISED_LISTENERS: 'PLAINTEXT://kafka-2:29092,PLAINTEXT_HOST://kafka-2:9092'
      KAFKA_DEFAULT_REPLICATION_FACTOR: 3
      KAFKA_MIN_INSYNC_REPLICAS: 2
      CLUSTER_ID: 'MkU3OEVBNTcwNTJENDM2Qk'
    volumes:
      - kafka2_data:/var/lib/kafka/data

  kafka-3:
    image: confluentinc/cp-kafka:7.6.0
    hostname: kafka-3
    ports:
      - "9094:9092"
    environment:
      KAFKA_NODE_ID: 3
      KAFKA_PROCESS_ROLES: 'broker,controller'
      KAFKA_CONTROLLER_QUORUM_VOTERS: '1@kafka-1:29093,2@kafka-2:29093,3@kafka-3:29093'
      KAFKA_LISTENERS: 'PLAINTEXT://kafka-3:29092,CONTROLLER://kafka-3:29093,PLAINTEXT_HOST://0.0.0.0:9092'
      KAFKA_ADVERTISED_LISTENERS: 'PLAINTEXT://kafka-3:29092,PLAINTEXT_HOST://kafka-3:9092'
      KAFKA_DEFAULT_REPLICATION_FACTOR: 3
      KAFKA_MIN_INSYNC_REPLICAS: 2
      CLUSTER_ID: 'MkU3OEVBNTcwNTJENDM2Qk'
    volumes:
      - kafka3_data:/var/lib/kafka/data

volumes:
  kafka1_data:
  kafka2_data:
  kafka3_data:
```

---

## 5. Kafka Topics: Naming Convention และ Strategy

### 5.1 Naming Convention

```
Pattern: {domain}.{entity}.{event}

ตัวอย่าง:
orders.order.created
orders.order.updated
orders.order.cancelled
payments.payment.initiated
payments.payment.completed
payments.payment.failed
inventory.product.updated
inventory.stock.depleted
users.user.registered
users.user.login
notifications.email.queued
notifications.push.queued
```

### 5.2 Partitioning Strategy

```javascript
// Kafka Admin: สร้าง topics พร้อม config
const admin = kafka.admin();
await admin.createTopics({
  topics: [
    {
      topic: 'orders.order.created',
      numPartitions: 12,  // 12 partitions = scale to 12 consumers
      replicationFactor: 3,
      configEntries: [
        { name: 'retention.ms', value: String(7 * 24 * 60 * 60 * 1000) }, // 7 days
        { name: 'retention.bytes', value: String(5 * 1024 * 1024 * 1024) }, // 5 GB
        { name: 'min.insync.replicas', value: '2' },
        { name: 'compression.type', value: 'lz4' }
      ]
    },
    {
      topic: 'payments.payment.completed',
      numPartitions: 6,
      replicationFactor: 3,
      configEntries: [
        { name: 'retention.ms', value: String(30 * 24 * 60 * 60 * 1000) }, // 30 days
        { name: 'min.insync.replicas', value: '2' }
      ]
    }
  ]
});
```

**Partitioning by key:**
```javascript
// ส่ง message พร้อม key เพื่อ ordering guarantee
await producer.send({
  topic: 'orders.order.created',
  messages: [{
    key: order.userId,      // Orders ของ user เดียวกัน → partition เดียวกัน
    value: JSON.stringify(order),
    headers: {
      'correlation-id': requestId,
      'content-type': 'application/json'
    }
  }]
});
```

---

## 6. Node.js Kafka Client: KafkaJS

### 6.1 Installation

```bash
npm install kafkajs
npm install -D @types/node
```

### 6.2 Basic Setup

```typescript
// kafka/client.ts
import { Kafka, logLevel } from 'kafkajs';

export const kafka = new Kafka({
  clientId: 'order-service',
  brokers: (process.env.KAFKA_BROKERS || 'localhost:9092').split(','),
  
  // TLS (สำหรับ production)
  ssl: process.env.NODE_ENV === 'production' ? {
    rejectUnauthorized: true,
    ca: [process.env.KAFKA_CA_CERT!],
    key: process.env.KAFKA_CLIENT_KEY!,
    cert: process.env.KAFKA_CLIENT_CERT!
  } : undefined,
  
  // SASL Authentication
  sasl: process.env.KAFKA_USERNAME ? {
    mechanism: 'scram-sha-512',
    username: process.env.KAFKA_USERNAME,
    password: process.env.KAFKA_PASSWORD!
  } : undefined,
  
  // Logging
  logLevel: process.env.NODE_ENV === 'production' ? logLevel.WARN : logLevel.INFO,
  
  // Retry config
  retry: {
    initialRetryTime: 100,
    retries: 8,
    maxRetryTime: 30000,
    factor: 0.2,
    multiplier: 2
  },
  
  // Connection timeout
  connectionTimeout: 3000,
  requestTimeout: 25000
});
```

### 6.3 Producer: send, sendBatch

```typescript
// kafka/producer.ts
import { kafka } from './client';
import { Producer, RecordMetadata } from 'kafkajs';

let producer: Producer | null = null;

export async function getProducer(): Promise<Producer> {
  if (producer) return producer;
  
  producer = kafka.producer({
    // Idempotent producer (prevent duplicates)
    idempotent: true,
    
    // Acknowledgment: all replicas must confirm
    acks: -1, // -1 = all, 1 = leader only, 0 = no ack
    
    // Compression
    compression: 2, // 2 = SNAPPY
    
    // Batching
    maxInFlightRequests: 1, // required for idempotent
    transactionalId: undefined // set for transactional
  });
  
  await producer.connect();
  console.log('Kafka producer connected');
  
  return producer;
}

// ส่ง single message
export async function sendMessage(topic: string, message: {
  key?: string;
  value: object | string;
  headers?: Record<string, string>;
}): Promise<RecordMetadata[]> {
  const prod = await getProducer();
  
  const value = typeof message.value === 'string'
    ? message.value
    : JSON.stringify(message.value);
  
  return prod.send({
    topic,
    messages: [{
      key: message.key,
      value,
      headers: {
        ...message.headers,
        'timestamp': Date.now().toString(),
        'service': 'order-service'
      }
    }]
  });
}

// ส่ง batch messages (efficient สำหรับ high volume)
export async function sendBatch(messages: Array<{
  topic: string;
  key?: string;
  value: object;
  headers?: Record<string, string>;
}>): Promise<void> {
  const prod = await getProducer();
  
  // Group by topic
  const topicMap = new Map<string, any[]>();
  
  for (const msg of messages) {
    if (!topicMap.has(msg.topic)) {
      topicMap.set(msg.topic, []);
    }
    topicMap.get(msg.topic)!.push({
      key: msg.key,
      value: JSON.stringify(msg.value),
      headers: msg.headers
    });
  }
  
  const topicMessages = Array.from(topicMap.entries()).map(([topic, msgs]) => ({
    topic,
    messages: msgs
  }));
  
  await prod.sendBatch({ topicMessages });
}

// Graceful shutdown
export async function disconnectProducer() {
  if (producer) {
    await producer.disconnect();
    producer = null;
  }
}
```

### 6.4 Consumer: subscribe, eachMessage, eachBatch

```typescript
// kafka/consumer.ts
import { kafka } from './client';
import { Consumer, EachMessagePayload, EachBatchPayload } from 'kafkajs';

// Simple message consumer
export async function createConsumer(
  groupId: string,
  topics: string[],
  handler: (payload: EachMessagePayload) => Promise<void>
): Promise<Consumer> {
  const consumer = kafka.consumer({
    groupId,
    // Session timeout: ถ้า consumer ไม่ heartbeat → rebalance
    sessionTimeout: 30000,
    heartbeatInterval: 3000,
    
    // Max messages to fetch per request
    maxBytesPerPartition: 1048576, // 1 MB
    
    // Retry
    retry: {
      initialRetryTime: 300,
      retries: 5
    }
  });
  
  await consumer.connect();
  
  await consumer.subscribe({
    topics,
    fromBeginning: false // true = replay all messages
  });
  
  await consumer.run({
    autoCommit: false,
    autoCommitInterval: 5000,
    
    // Process one message at a time
    eachMessage: async (payload) => {
      const { topic, partition, message } = payload;
      
      console.log(`[${groupId}] Processing message`, {
        topic,
        partition,
        offset: message.offset,
        key: message.key?.toString()
      });
      
      try {
        await handler(payload);
        
        // Manual commit หลัง process สำเร็จ
        await consumer.commitOffsets([{
          topic,
          partition,
          offset: (parseInt(message.offset) + 1).toString()
        }]);
        
      } catch (error) {
        console.error(`[${groupId}] Message processing failed:`, error);
        // ไม่ commit → message จะถูก re-process
        // ควรมี dead-letter queue สำหรับ messages ที่ fail ซ้ำ
        throw error;
      }
    }
  });
  
  return consumer;
}

// High-throughput batch consumer
export async function createBatchConsumer(
  groupId: string,
  topics: string[],
  handler: (messages: any[]) => Promise<void>
): Promise<Consumer> {
  const consumer = kafka.consumer({ groupId });
  
  await consumer.connect();
  await consumer.subscribe({ topics, fromBeginning: false });
  
  await consumer.run({
    autoCommit: false,
    eachBatchAutoResolve: false,
    
    eachBatch: async ({ batch, resolveOffset, heartbeat, isRunning, isStale }: EachBatchPayload) => {
      const messages = batch.messages.map(msg => ({
        key: msg.key?.toString(),
        value: JSON.parse(msg.value?.toString() || '{}'),
        offset: msg.offset,
        timestamp: msg.timestamp,
        headers: msg.headers
      }));
      
      console.log(`Processing batch of ${messages.length} messages`);
      
      // Process batch
      await handler(messages);
      
      // Commit all offsets in batch
      for (const message of batch.messages) {
        if (!isRunning() || isStale()) break;
        
        resolveOffset(message.offset);
        await heartbeat(); // ป้องกัน session timeout
      }
      
      // Final commit
      await consumer.commitOffsets([{
        topic: batch.topic,
        partition: batch.partition,
        offset: (parseInt(batch.lastOffset()) + 1).toString()
      }]);
    }
  });
  
  return consumer;
}
```

### 6.5 Admin: createTopics, listTopics

```typescript
// kafka/admin.ts
import { kafka } from './client';

const admin = kafka.admin();

export async function setupTopics() {
  await admin.connect();
  
  // List existing topics
  const existingTopics = await admin.listTopics();
  console.log('Existing topics:', existingTopics);
  
  // สร้าง topics ถ้ายังไม่มี
  const topicsToCreate = [
    {
      topic: 'orders.order.created',
      numPartitions: 12,
      replicationFactor: parseInt(process.env.KAFKA_REPLICATION_FACTOR || '1'),
      configEntries: [
        { name: 'retention.ms', value: '604800000' },  // 7 days
        { name: 'compression.type', value: 'lz4' }
      ]
    },
    {
      topic: 'orders.order.completed',
      numPartitions: 6,
      replicationFactor: parseInt(process.env.KAFKA_REPLICATION_FACTOR || '1'),
      configEntries: [
        { name: 'retention.ms', value: '2592000000' }  // 30 days
      ]
    },
    {
      topic: 'payments.payment.processed',
      numPartitions: 6,
      replicationFactor: parseInt(process.env.KAFKA_REPLICATION_FACTOR || '1')
    },
    {
      topic: 'inventory.stock.updated',
      numPartitions: 3,
      replicationFactor: parseInt(process.env.KAFKA_REPLICATION_FACTOR || '1')
    },
    // Dead Letter Queue
    {
      topic: 'dlq.failed-messages',
      numPartitions: 1,
      replicationFactor: parseInt(process.env.KAFKA_REPLICATION_FACTOR || '1'),
      configEntries: [
        { name: 'retention.ms', value: '2592000000' }  // 30 days
      ]
    }
  ];
  
  // ส้รางเฉพาะที่ยังไม่มี
  const newTopics = topicsToCreate.filter(
    t => !existingTopics.includes(t.topic)
  );
  
  if (newTopics.length > 0) {
    await admin.createTopics({ topics: newTopics });
    console.log(`Created ${newTopics.length} topics`);
  }
  
  // ดู topic metadata
  const metadata = await admin.fetchTopicMetadata({
    topics: topicsToCreate.map(t => t.topic)
  });
  
  console.log('Topic metadata:', JSON.stringify(metadata, null, 2));
  
  await admin.disconnect();
}

// List consumer groups
export async function listConsumerGroups() {
  await admin.connect();
  const groups = await admin.listGroups();
  
  for (const group of groups.groups) {
    const details = await admin.describeGroups([group.groupId]);
    console.log(`Group: ${group.groupId}`, details);
  }
  
  await admin.disconnect();
}

// Reset consumer group offset (สำหรับ replay)
export async function resetOffset(groupId: string, topic: string, toEarliest = true) {
  await admin.connect();
  
  await admin.setOffsets({
    groupId,
    topic,
    partitions: [
      { partition: 0, offset: toEarliest ? '-2' : '-1' } // -2=earliest, -1=latest
    ]
  });
  
  await admin.disconnect();
}
```

---

## 7. Exactly-Once Semantics

### 7.1 Delivery Guarantees

```
1. At-most-once:
   - Producer ส่งครั้งเดียว ไม่ retry
   - ข้อมูลอาจหาย
   - เหมาะกับ: metrics, logs

2. At-least-once:
   - Producer retry เมื่อ fail
   - ข้อมูลอาจซ้ำ
   - Consumer ต้อง idempotent
   - เหมาะกับ: orders, payments (ถ้า idempotent)

3. Exactly-once:
   - ข้อมูลถูกส่งและ consume ครั้งเดียวแน่นอน
   - ต้องใช้ transactions
   - Performance overhead สูงกว่า
```

### 7.2 Idempotent Producer

```typescript
const producer = kafka.producer({
  idempotent: true,   // enable idempotent producer
  maxInFlightRequests: 1,  // required
  acks: -1,           // required: all replicas
  retries: 5
});

// Producer จะ deduplicate messages อัตโนมัติ
// ถ้า network error และ retry → broker จะรู้ว่า duplicate
```

### 7.3 Transactional Producer

```typescript
// server/kafka/transactional.ts
const producer = kafka.producer({
  idempotent: true,
  transactionalId: `order-service-${process.env.INSTANCE_ID || '1'}`,
  maxInFlightRequests: 1,
  acks: -1
});

await producer.connect();

export async function processOrderWithTransaction(order: Order) {
  const transaction = await producer.transaction();
  
  try {
    // ส่งหลาย events ใน transaction เดียว
    await transaction.send({
      topic: 'orders.order.created',
      messages: [{
        key: order.id,
        value: JSON.stringify({ type: 'ORDER_CREATED', order })
      }]
    });
    
    await transaction.send({
      topic: 'inventory.reservation.created',
      messages: [{
        key: order.productId,
        value: JSON.stringify({
          type: 'RESERVATION_CREATED',
          orderId: order.id,
          productId: order.productId,
          quantity: order.quantity
        })
      }]
    });
    
    // Commit transaction
    await transaction.commit();
    console.log('Transaction committed');
    
  } catch (error) {
    // Abort transaction
    await transaction.abort();
    console.error('Transaction aborted:', error);
    throw error;
  }
}
```

### 7.4 Idempotent Consumer

```typescript
// Consumer ที่ handle duplicate messages ได้
async function processOrderCreated(payload: EachMessagePayload) {
  const { message } = payload;
  const event = JSON.parse(message.value!.toString());
  const orderId = event.order.id;
  
  // ตรวจสอบว่าเคย process แล้วหรือยัง (idempotency check)
  const existing = await prisma.processedEvent.findUnique({
    where: {
      topic_partition_offset: {
        topic: payload.topic,
        partition: payload.partition,
        offset: parseInt(payload.message.offset)
      }
    }
  });
  
  if (existing) {
    console.log(`Event already processed: ${payload.message.offset}`);
    return; // Skip duplicate
  }
  
  // Process event
  await prisma.$transaction(async (tx) => {
    // Create order
    await tx.order.create({ data: event.order });
    
    // Mark as processed
    await tx.processedEvent.create({
      data: {
        topic: payload.topic,
        partition: payload.partition,
        offset: parseInt(payload.message.offset),
        processedAt: new Date()
      }
    });
  });
}
```

---

## 8. Schema Registry + Avro

### 8.1 ทำไมต้องใช้ Schema Registry?

```
ปัญหา: Producer เปลี่ยน schema → Consumer broke
Solution: Schema Registry ควบคุม schema evolution
```

### 8.2 Confluent Schema Registry

```bash
npm install @kafkajs/confluent-schema-registry
```

```typescript
// kafka/schemaRegistry.ts
import { SchemaRegistry, SchemaType } from '@kafkajs/confluent-schema-registry';

const registry = new SchemaRegistry({
  host: process.env.SCHEMA_REGISTRY_URL || 'http://localhost:8081',
  auth: {
    username: process.env.SCHEMA_REGISTRY_KEY || '',
    password: process.env.SCHEMA_REGISTRY_SECRET || ''
  }
});

// Avro schema สำหรับ Order
const orderSchema = {
  type: 'record',
  name: 'Order',
  namespace: 'com.example.orders',
  fields: [
    { name: 'id', type: 'string' },
    { name: 'userId', type: 'string' },
    { name: 'productId', type: 'string' },
    { name: 'quantity', type: 'int' },
    { name: 'amount', type: 'double' },
    { name: 'status', type: { type: 'enum', name: 'OrderStatus', symbols: ['PENDING', 'CONFIRMED', 'CANCELLED'] } },
    { name: 'createdAt', type: 'long', logicalType: 'timestamp-millis' },
    // Optional field (backward compatible)
    { name: 'notes', type: ['null', 'string'], default: null }
  ]
};

// Register schema
const { id: schemaId } = await registry.register({
  type: SchemaType.AVRO,
  schema: JSON.stringify(orderSchema)
}, {
  subject: 'orders.order.created-value'
});

// Encode message
export async function encodeOrder(order: Order): Promise<Buffer> {
  return registry.encode(schemaId, order);
}

// Decode message
export async function decodeOrder(encodedBuffer: Buffer): Promise<Order> {
  return registry.decode(encodedBuffer);
}

// Producer ด้วย Avro
await producer.send({
  topic: 'orders.order.created',
  messages: [{
    key: order.id,
    value: await encodeOrder(order)
  }]
});

// Consumer ด้วย Avro
consumer.run({
  eachMessage: async ({ message }) => {
    const order = await decodeOrder(message.value as Buffer);
    await processOrder(order);
  }
});
```

---

## 9. Kafka Connect: Database Integration

### 9.1 JDBC Source Connector: PostgreSQL → Kafka

```json
// connector-config/postgresql-source.json
{
  "name": "postgresql-orders-source",
  "config": {
    "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector",
    "connection.url": "jdbc:postgresql://postgres:5432/mydb",
    "connection.user": "kafka_user",
    "connection.password": "password",
    
    "mode": "timestamp+incrementing",
    "timestamp.column.name": "updated_at",
    "incrementing.column.name": "id",
    
    "table.whitelist": "orders,payments",
    "topic.prefix": "db.",
    
    "poll.interval.ms": "1000",
    "batch.max.rows": "1000",
    
    "transforms": "createKey,extractInt",
    "transforms.createKey.type": "org.apache.kafka.connect.transforms.ValueToKey",
    "transforms.createKey.fields": "id",
    "transforms.extractInt.type": "org.apache.kafka.connect.transforms.ExtractField$Key",
    "transforms.extractInt.field": "id"
  }
}
```

### 9.2 JDBC Sink Connector: Kafka → PostgreSQL

```json
// connector-config/postgresql-sink.json
{
  "name": "postgresql-analytics-sink",
  "config": {
    "connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",
    "connection.url": "jdbc:postgresql://analytics-db:5432/analytics",
    "connection.user": "kafka_sink_user",
    "connection.password": "password",
    
    "topics": "orders.order.created,orders.order.completed",
    
    "insert.mode": "upsert",
    "pk.mode": "record_key",
    "pk.fields": "id",
    
    "auto.create": true,
    "auto.evolve": true,
    
    "batch.size": "100",
    "max.retries": "3",
    "retry.backoff.ms": "3000"
  }
}
```

### 9.3 Debezium CDC Connector

```json
// Debezium: PostgreSQL → Kafka (Change Data Capture)
{
  "name": "postgres-cdc-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "plugin.name": "pgoutput",
    
    "database.hostname": "postgres",
    "database.port": "5432",
    "database.user": "replication_user",
    "database.password": "password",
    "database.dbname": "mydb",
    "database.server.name": "mydb",
    
    "table.include.list": "public.orders,public.users,public.payments",
    
    "slot.name": "debezium",
    "publication.name": "debezium_pub",
    
    "decimal.handling.mode": "double",
    "binary.handling.mode": "base64",
    
    "topic.prefix": "cdc",
    
    "heartbeat.interval.ms": "5000",
    
    "transforms": "unwrap",
    "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
    "transforms.unwrap.drop.tombstones": "false",
    "transforms.unwrap.delete.handling.mode": "rewrite"
  }
}
```

---

## 10. Full Working Example: Order Processing Pipeline

### 10.1 Architecture

```
User Service      →  users.user.registered
Order Service     →  orders.order.created
                      orders.order.confirmed
                      orders.order.cancelled
Payment Service   ←  orders.order.confirmed (consume)
                  →  payments.payment.initiated
                      payments.payment.completed
                      payments.payment.failed
Inventory Service ←  orders.order.confirmed (consume)
                  →  inventory.stock.reserved
                      inventory.stock.depleted
Notification      ←  orders.order.* (consume all)
                  ←  payments.payment.* (consume all)
                  →  sends email/push notification
Analytics Service ←  everything (consume all events)
                  →  updates dashboards
```

### 10.2 Order Service

```typescript
// services/order/src/index.ts
import express from 'express';
import { PrismaClient } from '@prisma/client';
import { kafka } from './kafka/client';
import { setupTopics } from './kafka/admin';
import { createConsumer } from './kafka/consumer';

const app = express();
const prisma = new PrismaClient();

app.use(express.json());

// Producer
const producer = kafka.producer({ idempotent: true, acks: -1 });

// === POST /orders: สร้าง order ใหม่ ===
app.post('/orders', async (req, res) => {
  try {
    const { userId, productId, quantity, amount } = req.body;
    
    // สร้าง order ใน database
    const order = await prisma.order.create({
      data: {
        userId,
        productId,
        quantity,
        amount,
        status: 'PENDING'
      }
    });
    
    // Publish event
    await producer.send({
      topic: 'orders.order.created',
      messages: [{
        key: order.id,
        value: JSON.stringify({
          type: 'ORDER_CREATED',
          orderId: order.id,
          userId: order.userId,
          productId: order.productId,
          quantity: order.quantity,
          amount: order.amount,
          timestamp: new Date().toISOString()
        })
      }]
    });
    
    res.status(201).json(order);
    
  } catch (error) {
    console.error('Create order error:', error);
    res.status(500).json({ error: 'Failed to create order' });
  }
});

// Consumer: รับ payment events
async function startConsumers() {
  const consumer = await createConsumer(
    'order-service-payment-consumer',
    ['payments.payment.completed', 'payments.payment.failed'],
    async ({ topic, message }) => {
      const event = JSON.parse(message.value!.toString());
      
      if (topic === 'payments.payment.completed') {
        await prisma.order.update({
          where: { id: event.orderId },
          data: { status: 'CONFIRMED', confirmedAt: new Date() }
        });
        
        // ส่ง event ว่า order confirmed
        await producer.send({
          topic: 'orders.order.confirmed',
          messages: [{
            key: event.orderId,
            value: JSON.stringify({
              type: 'ORDER_CONFIRMED',
              orderId: event.orderId,
              paymentId: event.paymentId,
              timestamp: new Date().toISOString()
            })
          }]
        });
        
      } else if (topic === 'payments.payment.failed') {
        await prisma.order.update({
          where: { id: event.orderId },
          data: { status: 'CANCELLED', cancelledAt: new Date() }
        });
        
        await producer.send({
          topic: 'orders.order.cancelled',
          messages: [{
            key: event.orderId,
            value: JSON.stringify({
              type: 'ORDER_CANCELLED',
              orderId: event.orderId,
              reason: event.reason,
              timestamp: new Date().toISOString()
            })
          }]
        });
      }
    }
  );
  
  return consumer;
}

// Bootstrap
async function main() {
  await setupTopics();
  await producer.connect();
  await startConsumers();
  
  app.listen(3001, () => console.log('Order service running on :3001'));
}

main().catch(console.error);
```

### 10.3 Payment Service

```typescript
// services/payment/src/index.ts
import { kafka } from './kafka/client';
import { PrismaClient } from '@prisma/client';
import Stripe from 'stripe';

const prisma = new PrismaClient();
const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);
const producer = kafka.producer({ idempotent: true });
const consumer = kafka.consumer({ groupId: 'payment-service' });

async function processPayment(event: any) {
  const { orderId, userId, amount } = event;
  
  // ตรวจสอบ idempotency
  const existing = await prisma.payment.findFirst({
    where: { orderId, status: { not: 'FAILED' } }
  });
  
  if (existing) {
    console.log(`Payment already exists for order ${orderId}`);
    return;
  }
  
  try {
    // สร้าง payment intent ใน Stripe
    const paymentIntent = await stripe.paymentIntents.create({
      amount: Math.round(amount * 100), // cents
      currency: 'thb',
      metadata: { orderId, userId }
    });
    
    // บันทึกใน database
    const payment = await prisma.payment.create({
      data: {
        orderId,
        userId,
        amount,
        status: 'INITIATED',
        stripePaymentIntentId: paymentIntent.id
      }
    });
    
    // ส่ง event
    await producer.send({
      topic: 'payments.payment.initiated',
      messages: [{
        key: orderId,
        value: JSON.stringify({
          type: 'PAYMENT_INITIATED',
          paymentId: payment.id,
          orderId,
          amount,
          stripeClientSecret: paymentIntent.client_secret
        })
      }]
    });
    
  } catch (error) {
    // ส่ง payment failed event
    await producer.send({
      topic: 'payments.payment.failed',
      messages: [{
        key: orderId,
        value: JSON.stringify({
          type: 'PAYMENT_FAILED',
          orderId,
          reason: error instanceof Error ? error.message : 'Unknown error'
        })
      }]
    });
  }
}

async function main() {
  await producer.connect();
  await consumer.connect();
  await consumer.subscribe({
    topics: ['orders.order.created'],
    fromBeginning: false
  });
  
  await consumer.run({
    autoCommit: false,
    eachMessage: async ({ topic, partition, message }) => {
      const event = JSON.parse(message.value!.toString());
      await processPayment(event);
      
      await consumer.commitOffsets([{
        topic,
        partition,
        offset: (parseInt(message.offset) + 1).toString()
      }]);
    }
  });
  
  console.log('Payment service started');
}

main().catch(console.error);
```

### 10.4 Notification Service

```typescript
// services/notification/src/index.ts
import { kafka } from './kafka/client';
import { createBatchConsumer } from './kafka/consumer';
import nodemailer from 'nodemailer';

const transporter = nodemailer.createTransport({
  host: process.env.SMTP_HOST,
  port: 587,
  auth: {
    user: process.env.SMTP_USER,
    pass: process.env.SMTP_PASS
  }
});

const notificationHandlers: Record<string, (event: any) => Promise<void>> = {
  'ORDER_CREATED': async (event) => {
    await transporter.sendMail({
      to: event.userEmail,
      subject: 'Order Received',
      html: `
        <h1>Order #${event.orderId} received!</h1>
        <p>Your order is being processed.</p>
        <p>Amount: ฿${event.amount}</p>
      `
    });
  },
  
  'ORDER_CONFIRMED': async (event) => {
    await transporter.sendMail({
      to: event.userEmail,
      subject: 'Order Confirmed',
      html: `
        <h1>Order #${event.orderId} confirmed!</h1>
        <p>Payment successful. Your order is on its way.</p>
      `
    });
  },
  
  'PAYMENT_FAILED': async (event) => {
    await transporter.sendMail({
      to: event.userEmail,
      subject: 'Payment Failed',
      html: `
        <h1>Payment for Order #${event.orderId} failed</h1>
        <p>Reason: ${event.reason}</p>
        <p>Please try again or contact support.</p>
      `
    });
  }
};

async function main() {
  const consumer = await createBatchConsumer(
    'notification-service',
    [
      'orders.order.created',
      'orders.order.confirmed',
      'orders.order.cancelled',
      'payments.payment.failed'
    ],
    async (messages) => {
      for (const msg of messages) {
        const event = msg.value;
        const handler = notificationHandlers[event.type];
        
        if (handler) {
          try {
            await handler(event);
            console.log(`Notification sent for ${event.type}`);
          } catch (error) {
            console.error(`Failed to send notification for ${event.type}:`, error);
            // Send to DLQ
            // await sendToDLQ(msg);
          }
        }
      }
    }
  );
  
  console.log('Notification service started');
}

main().catch(console.error);
```

---

## 11. Dead Letter Queue (DLQ)

```typescript
// kafka/dlq.ts
import { kafka } from './client';

const dlqProducer = kafka.producer();
await dlqProducer.connect();

export async function sendToDLQ(
  originalTopic: string,
  originalPartition: number,
  originalOffset: string,
  message: any,
  error: Error
) {
  await dlqProducer.send({
    topic: 'dlq.failed-messages',
    messages: [{
      key: `${originalTopic}-${originalPartition}-${originalOffset}`,
      value: JSON.stringify({
        originalTopic,
        originalPartition,
        originalOffset,
        originalMessage: message,
        error: {
          name: error.name,
          message: error.message,
          stack: error.stack
        },
        failedAt: new Date().toISOString()
      }),
      headers: {
        'original-topic': originalTopic,
        'error-type': error.name
      }
    }]
  });
}

// DLQ Reprocessor: process failed messages อีกครั้ง
export async function reprocessDLQ() {
  const consumer = kafka.consumer({ groupId: 'dlq-reprocessor' });
  await consumer.connect();
  await consumer.subscribe({ topics: ['dlq.failed-messages'], fromBeginning: true });
  
  await consumer.run({
    eachMessage: async ({ message }) => {
      const failed = JSON.parse(message.value!.toString());
      
      console.log('Reprocessing failed message:', {
        originalTopic: failed.originalTopic,
        error: failed.error.message,
        failedAt: failed.failedAt
      });
      
      // Retry original processing
      // TODO: implement retry logic
    }
  });
}
```

---

## 12. Monitoring และ Performance

### 12.1 Consumer Lag Monitoring

```typescript
// monitoring/consumerLag.ts
async function checkConsumerLag(groupId: string, topic: string) {
  const admin = kafka.admin();
  await admin.connect();
  
  // ดู offsets ของ consumer group
  const offsets = await admin.fetchOffsets({ groupId, topics: [topic] });
  
  // ดู latest offsets ของ topic
  const topicOffsets = await admin.fetchTopicOffsets(topic);
  
  let totalLag = 0;
  
  for (const partitionOffset of offsets[0].partitions) {
    const latest = topicOffsets.find(o => o.partition === partitionOffset.partition);
    if (latest) {
      const lag = parseInt(latest.offset) - parseInt(partitionOffset.offset);
      totalLag += lag;
      
      if (lag > 10000) {
        console.warn(`⚠️ High lag on partition ${partitionOffset.partition}: ${lag}`);
      }
    }
  }
  
  console.log(`Total consumer lag for ${groupId}: ${totalLag}`);
  
  await admin.disconnect();
  return totalLag;
}

// ตรวจสอบทุก 1 นาที
setInterval(() => {
  checkConsumerLag('order-service-payment-consumer', 'payments.payment.completed');
}, 60000);
```

### 12.2 Kafka Metrics

```yaml
# prometheus/kafka-rules.yml
groups:
  - name: kafka_consumer_lag
    rules:
      - alert: KafkaConsumerLagHigh
        expr: kafka_consumer_group_lag > 50000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Kafka consumer lag is high"
          description: "Consumer group {{ $labels.group }} has lag {{ $value }}"
      
      - alert: KafkaConsumerLagCritical
        expr: kafka_consumer_group_lag > 500000
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Kafka consumer lag is critical"
```

---

## สรุป

Kafka Integration กับ Database Cluster เป็นสถาปัตยกรรมที่ทรงพลังสำหรับ:

1. **Event-driven microservices**: services communicate ผ่าน events แทน direct API calls
2. **Data pipeline**: ส่งข้อมูลจาก OLTP database ไปยัง analytics systems
3. **CDC (Change Data Capture)**: capture database changes ใน real-time ด้วย Debezium
4. **Durability + Replay**: events เก็บไว้ตาม retention → replay ได้เมื่อต้องการ
5. **Scale**: เพิ่ม partitions และ consumers ได้ตามต้องการ

Key differences vs Redis Pub/Sub:
- **Kafka**: durable, replayable, ordered per partition, complex setup
- **Redis**: ephemeral, ultra-fast, fire-and-forget, simple setup

ใน production ใช้ทั้งสอง:
- Redis: real-time notifications, presence, typing indicators
- Kafka: business events, analytics pipeline, cross-service communication

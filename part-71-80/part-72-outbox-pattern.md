# Part 72: Outbox Pattern

## บทนำ

ใน Microservices Architecture ปัญหาที่พบบ่อยที่สุดคือ **Dual Write Problem**: เมื่อต้องการบันทึกข้อมูลลง Database และส่ง Event ไปยัง Message Broker (Kafka, RabbitMQ) พร้อมกัน — ทั้งสองการดำเนินการนี้ไม่ได้อยู่ใน Transaction เดียวกัน จึงอาจเกิดความไม่สอดคล้องได้

---

## 1. Dual Write Problem

### 1.1 สถานการณ์ปัญหา

```
// ❌ Dual Write - WRONG WAY
async function createOrder(orderData) {
  // Operation 1: Write to Database
  const order = await db.orders.insert(orderData);
  
  // *** CRASH HERE *** หรือ Kafka ล่ม
  
  // Operation 2: Publish Event
  await kafka.produce('order-created', { orderId: order.id });
  // ถ้าส่วนนี้ล้มเหลว = Order อยู่ใน DB แต่ไม่มี Event!
}
```

### 1.2 สถานการณ์ที่เป็นไปได้

**Scenario A: DB สำเร็จ, Kafka ล้มเหลว**
```
→ Order ถูกสร้างใน DB
→ ไม่มี Event ถูกส่งไป
→ Services อื่นไม่รู้ว่ามี Order ใหม่
→ Inventory ไม่ถูกจอง, Payment ไม่ถูกเรียก
= DATA INCONSISTENCY
```

**Scenario B: Kafka สำเร็จ, DB ล้มเหลว (Rollback)**
```
→ Order ไม่อยู่ใน DB
→ Event ถูกส่งไปแล้ว  
→ Inventory ถูกจองสำหรับ Order ที่ไม่มีอยู่จริง
= DATA INCONSISTENCY
```

**Scenario C: Network Timeout**
```
→ DB บันทึกสำเร็จ
→ Kafka produce timeout (แต่อาจสำเร็จหรือไม่ก็ได้)
→ Retry → อาจส่ง Event ซ้ำ (Duplicate)
= DUPLICATE EVENTS
```

---

## 2. Outbox Pattern: วิธีแก้ปัญหา

### 2.1 แนวคิดหลัก

**Outbox Pattern** แก้ปัญหาด้วยการ:
1. เขียน Business Data + Event ลง **Database เดียวกัน** ใน **Transaction เดียวกัน**
2. มี **Outbox Processor** แยกต่างหากที่อ่าน Events จาก Outbox Table แล้วส่งไป Message Broker

```
┌─────────────────────────────────────────┐
│             Application                  │
│                                          │
│  BEGIN TRANSACTION                       │
│    INSERT INTO orders (...) ✓            │
│    INSERT INTO outbox (...) ✓            │  ← Same Transaction!
│  COMMIT TRANSACTION                      │
└─────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────┐
│           Outbox Processor               │
│                                          │
│  SELECT * FROM outbox WHERE status=PENDING
│  → Publish to Kafka                      │
│  → UPDATE outbox SET status=PROCESSED   │
└─────────────────────────────────────────┘
```

### 2.2 ทำไม Outbox Processor จึงปลอดภัย

เนื่องจาก Outbox Processor เป็น **Separate Process** ที่:
- อ่าน Events จาก DB และส่งไป Kafka
- ถ้าส่งสำเร็จ → Mark as PROCESSED
- ถ้าส่งล้มเหลว → Retry ในครั้งถัดไป
- Worst case: Event อาจถูกส่งซ้ำ (At-Least-Once) แต่ไม่มีทางหาย (Not At-Most-Once)

---

## 3. Outbox Table Design

### 3.1 Schema

```sql
-- migrations/001_create_outbox_table.sql

CREATE TYPE outbox_status AS ENUM ('PENDING', 'PROCESSING', 'PROCESSED', 'FAILED', 'DEAD');

CREATE TABLE outbox_events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  
  -- Aggregate information
  aggregate_type VARCHAR(100) NOT NULL,  -- e.g., 'Order', 'User'
  aggregate_id VARCHAR(255) NOT NULL,    -- ID ของ Entity
  
  -- Event information
  event_type VARCHAR(255) NOT NULL,       -- e.g., 'OrderCreated', 'UserRegistered'
  event_version VARCHAR(20) DEFAULT '1.0',
  
  -- Payload
  payload JSONB NOT NULL,
  headers JSONB DEFAULT '{}',
  
  -- Routing
  destination_topic VARCHAR(255) NOT NULL,
  partition_key VARCHAR(255),             -- สำหรับ Kafka Partitioning
  
  -- Status tracking
  status outbox_status NOT NULL DEFAULT 'PENDING',
  retry_count INTEGER NOT NULL DEFAULT 0,
  max_retries INTEGER NOT NULL DEFAULT 5,
  last_error TEXT,
  
  -- Timestamps
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  scheduled_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),  -- เมื่อไหร่ถึง Process
  processing_started_at TIMESTAMPTZ,
  processed_at TIMESTAMPTZ,
  
  -- Idempotency
  idempotency_key VARCHAR(255) UNIQUE,   -- ป้องกัน Duplicate inserts
  
  CONSTRAINT chk_payload_is_object CHECK (jsonb_typeof(payload) = 'object')
);

-- Indexes for efficient queries
CREATE INDEX idx_outbox_status_scheduled ON outbox_events(status, scheduled_at)
  WHERE status IN ('PENDING', 'FAILED');
  
CREATE INDEX idx_outbox_aggregate ON outbox_events(aggregate_type, aggregate_id);

CREATE INDEX idx_outbox_created_at ON outbox_events(created_at DESC);

CREATE INDEX idx_outbox_idempotency ON outbox_events(idempotency_key) 
  WHERE idempotency_key IS NOT NULL;

-- Auto-cleanup old processed events (optional)
CREATE OR REPLACE FUNCTION cleanup_old_outbox_events()
RETURNS void AS $$
BEGIN
  DELETE FROM outbox_events
  WHERE status = 'PROCESSED'
    AND processed_at < NOW() - INTERVAL '7 days';
END;
$$ LANGUAGE plpgsql;
```

### 3.2 โมเดล TypeORM

```typescript
// src/entities/outbox-event.entity.ts
import {
  Entity,
  Column,
  PrimaryGeneratedColumn,
  CreateDateColumn,
  Index,
} from 'typeorm';

export enum OutboxStatus {
  PENDING = 'PENDING',
  PROCESSING = 'PROCESSING',
  PROCESSED = 'PROCESSED',
  FAILED = 'FAILED',
  DEAD = 'DEAD',
}

@Entity('outbox_events')
@Index(['status', 'scheduledAt'])
export class OutboxEvent {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ name: 'aggregate_type', length: 100 })
  aggregateType: string;

  @Column({ name: 'aggregate_id', length: 255 })
  aggregateId: string;

  @Column({ name: 'event_type', length: 255 })
  eventType: string;

  @Column({ name: 'event_version', length: 20, default: '1.0' })
  eventVersion: string;

  @Column({ type: 'jsonb' })
  payload: Record<string, any>;

  @Column({ type: 'jsonb', default: {} })
  headers: Record<string, string>;

  @Column({ name: 'destination_topic', length: 255 })
  destinationTopic: string;

  @Column({ name: 'partition_key', nullable: true })
  partitionKey?: string;

  @Column({
    type: 'enum',
    enum: OutboxStatus,
    default: OutboxStatus.PENDING,
  })
  status: OutboxStatus;

  @Column({ name: 'retry_count', default: 0 })
  retryCount: number;

  @Column({ name: 'max_retries', default: 5 })
  maxRetries: number;

  @Column({ name: 'last_error', nullable: true, type: 'text' })
  lastError?: string;

  @CreateDateColumn({ name: 'created_at' })
  createdAt: Date;

  @Column({ name: 'scheduled_at', default: () => 'NOW()' })
  scheduledAt: Date;

  @Column({ name: 'processing_started_at', nullable: true })
  processingStartedAt?: Date;

  @Column({ name: 'processed_at', nullable: true })
  processedAt?: Date;

  @Column({ name: 'idempotency_key', nullable: true, unique: true })
  idempotencyKey?: string;
}
```

---

## 4. การใช้งาน Outbox ใน Application

### 4.1 OutboxService Helper

```typescript
// src/outbox/outbox.service.ts
import { Injectable } from '@nestjs/common';
import { EntityManager } from 'typeorm';
import { OutboxEvent, OutboxStatus } from '../entities/outbox-event.entity';
import { v4 as uuidv4 } from 'uuid';
import crypto from 'crypto';

export interface PublishEventOptions {
  aggregateType: string;
  aggregateId: string;
  eventType: string;
  payload: Record<string, any>;
  destinationTopic: string;
  partitionKey?: string;
  headers?: Record<string, string>;
  idempotencyKey?: string;
  scheduledAt?: Date;
}

@Injectable()
export class OutboxService {
  /**
   * บันทึก Event ลง Outbox Table ภายใน Transaction เดียวกัน
   * ต้องเรียกภายใน dataSource.transaction() เสมอ
   */
  async publish(
    manager: EntityManager,
    options: PublishEventOptions,
  ): Promise<OutboxEvent> {
    const idempotencyKey = options.idempotencyKey 
      || this.generateIdempotencyKey(options.aggregateType, options.aggregateId, options.eventType);

    // Check for duplicate (Idempotent insert)
    const existing = await manager.findOne(OutboxEvent, {
      where: { idempotencyKey },
    });

    if (existing) {
      return existing; // Already queued, return existing
    }

    const event = manager.create(OutboxEvent, {
      id: uuidv4(),
      aggregateType: options.aggregateType,
      aggregateId: options.aggregateId,
      eventType: options.eventType,
      payload: options.payload,
      headers: options.headers || {},
      destinationTopic: options.destinationTopic,
      partitionKey: options.partitionKey || options.aggregateId,
      status: OutboxStatus.PENDING,
      idempotencyKey,
      scheduledAt: options.scheduledAt || new Date(),
    });

    return manager.save(OutboxEvent, event);
  }

  private generateIdempotencyKey(
    aggregateType: string,
    aggregateId: string,
    eventType: string,
  ): string {
    return crypto
      .createHash('sha256')
      .update(`${aggregateType}:${aggregateId}:${eventType}:${Date.now()}`)
      .digest('hex')
      .substring(0, 32);
  }
}
```

### 4.2 ใช้งานใน Order Service

```typescript
// src/orders/order.service.ts
import { Injectable } from '@nestjs/common';
import { DataSource } from 'typeorm';
import { OutboxService } from '../outbox/outbox.service';
import { Order } from './order.entity';
import { CreateOrderDto } from './dto/create-order.dto';

@Injectable()
export class OrderService {
  constructor(
    private dataSource: DataSource,
    private outboxService: OutboxService,
  ) {}

  async createOrder(dto: CreateOrderDto): Promise<Order> {
    return this.dataSource.transaction(async (manager) => {
      // 1. บันทึก Order ลง Database
      const order = manager.create(Order, {
        customerId: dto.customerId,
        items: dto.items,
        totalAmount: dto.totalAmount,
        status: 'PENDING',
      });
      const savedOrder = await manager.save(Order, order);

      // 2. บันทึก Event ลง Outbox (ใน Transaction เดียวกัน!)
      await this.outboxService.publish(manager, {
        aggregateType: 'Order',
        aggregateId: savedOrder.id,
        eventType: 'OrderCreated',
        payload: {
          orderId: savedOrder.id,
          customerId: savedOrder.customerId,
          items: savedOrder.items,
          totalAmount: savedOrder.totalAmount,
          status: savedOrder.status,
          createdAt: savedOrder.createdAt,
        },
        destinationTopic: 'orders',
        partitionKey: savedOrder.customerId, // Partition by customer
        headers: {
          'content-type': 'application/json',
          'event-version': '1.0',
        },
      });

      return savedOrder;
    });
    // ← Transaction commit: Both order AND outbox event are saved atomically
    // ← If Kafka is down, that's OK - outbox processor will retry later
  }

  async updateOrderStatus(orderId: string, status: string): Promise<Order> {
    return this.dataSource.transaction(async (manager) => {
      const order = await manager.findOneOrFail(Order, { where: { id: orderId } });
      const previousStatus = order.status;
      
      order.status = status;
      order.updatedAt = new Date();
      const updated = await manager.save(Order, order);

      await this.outboxService.publish(manager, {
        aggregateType: 'Order',
        aggregateId: orderId,
        eventType: 'OrderStatusUpdated',
        payload: {
          orderId,
          previousStatus,
          newStatus: status,
          updatedAt: updated.updatedAt,
        },
        destinationTopic: 'order-events',
        partitionKey: orderId,
      });

      return updated;
    });
  }
}
```

---

## 5. Outbox Processor

### 5.1 Poll-Based Processor

```typescript
// src/outbox/outbox-processor.service.ts
import { Injectable, Logger, OnModuleInit, OnModuleDestroy } from '@nestjs/common';
import { InjectDataSource } from '@nestjs/typeorm';
import { DataSource } from 'typeorm';
import { Kafka, Producer, Message } from 'kafkajs';
import { OutboxEvent, OutboxStatus } from '../entities/outbox-event.entity';

@Injectable()
export class OutboxProcessorService implements OnModuleInit, OnModuleDestroy {
  private readonly logger = new Logger(OutboxProcessorService.name);
  private kafkaProducer: Producer;
  private isRunning = false;
  private processingInterval: NodeJS.Timeout;

  constructor(
    @InjectDataSource()
    private dataSource: DataSource,
  ) {}

  async onModuleInit() {
    const kafka = new Kafka({
      clientId: 'outbox-processor',
      brokers: process.env.KAFKA_BROKERS.split(','),
      retry: {
        initialRetryTime: 100,
        retries: 8,
      },
    });

    this.kafkaProducer = kafka.producer({
      allowAutoTopicCreation: false,
      transactionTimeout: 30000,
      idempotent: true, // Enable Kafka Idempotent Producer
    });

    await this.kafkaProducer.connect();
    this.logger.log('Kafka producer connected');

    this.startProcessing();
  }

  async onModuleDestroy() {
    this.isRunning = false;
    if (this.processingInterval) {
      clearInterval(this.processingInterval);
    }
    await this.kafkaProducer.disconnect();
  }

  private startProcessing() {
    this.isRunning = true;
    // Process every 1 second
    this.processingInterval = setInterval(async () => {
      if (!this.isRunning) return;
      await this.processBatch();
    }, 1000);
  }

  private async processBatch(): Promise<void> {
    try {
      // Claim a batch of PENDING events
      const events = await this.claimBatch(50);
      
      if (events.length === 0) return;

      this.logger.debug(`Processing batch of ${events.length} outbox events`);

      // Group by topic for efficient Kafka produces
      const byTopic = this.groupByTopic(events);

      for (const [topic, topicEvents] of Object.entries(byTopic)) {
        await this.publishToKafka(topic, topicEvents);
      }
    } catch (error) {
      this.logger.error(`Error processing outbox batch: ${error.message}`);
    }
  }

  private async claimBatch(batchSize: number): Promise<OutboxEvent[]> {
    // Use SELECT ... FOR UPDATE SKIP LOCKED to prevent multiple processors from claiming same events
    const events = await this.dataSource.query(
      `UPDATE outbox_events
       SET status = 'PROCESSING',
           processing_started_at = NOW()
       WHERE id IN (
         SELECT id FROM outbox_events
         WHERE status IN ('PENDING', 'FAILED')
           AND scheduled_at <= NOW()
           AND retry_count < max_retries
         ORDER BY scheduled_at ASC
         LIMIT $1
         FOR UPDATE SKIP LOCKED
       )
       RETURNING *`,
      [batchSize],
    );

    return events;
  }

  private groupByTopic(events: OutboxEvent[]): Record<string, OutboxEvent[]> {
    return events.reduce((acc, event) => {
      if (!acc[event.destinationTopic]) {
        acc[event.destinationTopic] = [];
      }
      acc[event.destinationTopic].push(event);
      return acc;
    }, {} as Record<string, OutboxEvent[]>);
  }

  private async publishToKafka(topic: string, events: OutboxEvent[]): Promise<void> {
    const messages: Message[] = events.map((event) => ({
      key: event.partitionKey || event.aggregateId,
      value: JSON.stringify({
        id: event.id,
        aggregateType: event.aggregateType,
        aggregateId: event.aggregateId,
        eventType: event.eventType,
        eventVersion: event.eventVersion,
        payload: event.payload,
        createdAt: event.createdAt,
      }),
      headers: {
        ...event.headers,
        'event-id': event.id,
        'event-type': event.eventType,
        'aggregate-type': event.aggregateType,
        'aggregate-id': event.aggregateId,
      },
    }));

    try {
      await this.kafkaProducer.send({ topic, messages });

      // Mark all as PROCESSED
      const ids = events.map((e) => `'${e.id}'`).join(',');
      await this.dataSource.query(
        `UPDATE outbox_events
         SET status = 'PROCESSED',
             processed_at = NOW()
         WHERE id IN (${ids})`,
      );

      this.logger.debug(`Published ${events.length} events to topic: ${topic}`);
    } catch (error) {
      this.logger.error(`Failed to publish to Kafka topic ${topic}: ${error.message}`);

      // Mark as FAILED and schedule retry
      const ids = events.map((e) => `'${e.id}'`).join(',');
      await this.dataSource.query(
        `UPDATE outbox_events
         SET status = CASE 
               WHEN retry_count + 1 >= max_retries THEN 'DEAD'
               ELSE 'FAILED'
             END,
             retry_count = retry_count + 1,
             last_error = $1,
             -- Exponential backoff: 2^retry_count minutes
             scheduled_at = NOW() + (POWER(2, retry_count) * INTERVAL '1 minute')
         WHERE id IN (${ids})`,
        [error.message],
      );
    }
  }
}
```

### 5.2 Event-Based Processor ด้วย PostgreSQL LISTEN/NOTIFY

```typescript
// src/outbox/outbox-notify.service.ts
import { Injectable, Logger, OnModuleInit } from '@nestjs/common';
import { InjectDataSource } from '@nestjs/typeorm';
import { DataSource } from 'typeorm';
import { Client } from 'pg';

@Injectable()
export class OutboxNotifyService implements OnModuleInit {
  private readonly logger = new Logger(OutboxNotifyService.name);
  private pgClient: Client;

  constructor(
    @InjectDataSource()
    private dataSource: DataSource,
    private outboxProcessor: OutboxProcessorService,
  ) {}

  async onModuleInit() {
    // Create a dedicated PG connection for LISTEN (cannot use connection pool)
    this.pgClient = new Client({
      connectionString: process.env.DATABASE_URL,
    });

    await this.pgClient.connect();

    // Listen for outbox notifications
    await this.pgClient.query('LISTEN outbox_event_inserted');

    this.pgClient.on('notification', async (msg) => {
      if (msg.channel === 'outbox_event_inserted') {
        this.logger.debug('New outbox event notification received');
        // Trigger processing immediately
        await this.outboxProcessor.processBatch();
      }
    });

    this.logger.log('Listening for outbox notifications');
  }
}
```

```sql
-- Trigger ที่ส่ง NOTIFY เมื่อมี Outbox Event ใหม่
CREATE OR REPLACE FUNCTION notify_outbox_insert()
RETURNS TRIGGER AS $$
BEGIN
  PERFORM pg_notify('outbox_event_inserted', NEW.id::text);
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER outbox_insert_trigger
AFTER INSERT ON outbox_events
FOR EACH ROW EXECUTE FUNCTION notify_outbox_insert();
```

---

## 6. Change Data Capture (CDC) ด้วย Debezium

### 6.1 CDC แทน Outbox Processor

แทนที่จะ Poll หรือ Listen เอง เราสามารถใช้ **Debezium** อ่าน PostgreSQL WAL (Write-Ahead Log) โดยตรง:

```
PostgreSQL WAL
      │
      ▼
  Debezium Connector (Kafka Connect)
      │
      ▼
    Kafka
      │
      ▼
  Consumer Services
```

### 6.2 Docker Compose สำหรับ Debezium + Outbox

```yaml
# docker-compose.debezium.yml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    command: >
      postgres
      -c wal_level=logical
      -c max_replication_slots=4
      -c max_wal_senders=4
      -c shared_preload_libraries=pgoutput
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres123
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
    volumes:
      - zookeeper_data:/var/lib/zookeeper/data

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    depends_on:
      - zookeeper
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: 'true'
    ports:
      - "9092:9092"
    volumes:
      - kafka_data:/var/lib/kafka/data

  kafka-connect:
    image: debezium/connect:2.4
    depends_on:
      - kafka
      - postgres
    environment:
      BOOTSTRAP_SERVERS: kafka:29092
      GROUP_ID: debezium-connect-group
      CONFIG_STORAGE_TOPIC: debezium_configs
      OFFSET_STORAGE_TOPIC: debezium_offsets
      STATUS_STORAGE_TOPIC: debezium_statuses
      KEY_CONVERTER: org.apache.kafka.connect.json.JsonConverter
      VALUE_CONVERTER: org.apache.kafka.connect.json.JsonConverter
      CONNECT_KEY_CONVERTER_SCHEMAS_ENABLE: 'false'
      CONNECT_VALUE_CONVERTER_SCHEMAS_ENABLE: 'false'
    ports:
      - "8083:8083"

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    depends_on:
      - kafka
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:29092
    ports:
      - "8080:8080"

volumes:
  postgres_data:
  zookeeper_data:
  kafka_data:
```

### 6.3 Register Debezium Connector

```bash
# Register connector via Kafka Connect REST API
curl -X POST http://localhost:8083/connectors \
  -H "Content-Type: application/json" \
  -d '{
    "name": "outbox-connector",
    "config": {
      "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
      "database.hostname": "postgres",
      "database.port": "5432",
      "database.user": "postgres",
      "database.password": "postgres123",
      "database.dbname": "myapp",
      "database.server.name": "myapp",
      "plugin.name": "pgoutput",
      "slot.name": "debezium_outbox_slot",
      "publication.name": "debezium_publication",
      "table.include.list": "public.outbox_events",
      "transforms": "outbox",
      "transforms.outbox.type": "io.debezium.transforms.outbox.EventRouter",
      "transforms.outbox.table.field.event.id": "id",
      "transforms.outbox.table.field.event.key": "aggregate_id",
      "transforms.outbox.table.field.event.type": "event_type",
      "transforms.outbox.table.field.event.payload": "payload",
      "transforms.outbox.table.field.event.timestamp": "created_at",
      "transforms.outbox.route.topic.replacement": "${routedByValue}",
      "key.converter": "org.apache.kafka.connect.storage.StringConverter",
      "value.converter": "org.apache.kafka.connect.json.JsonConverter",
      "value.converter.schemas.enable": "false",
      "heartbeat.interval.ms": "10000"
    }
  }'
```

```bash
# ตรวจสอบสถานะ Connector
curl http://localhost:8083/connectors/outbox-connector/status
```

---

## 7. Inbox Pattern: Idempotent Event Consumption

### 7.1 ปัญหา At-Least-Once Delivery

Outbox Pattern รับประกัน **At-Least-Once Delivery** — Event อาจถูกส่งซ้ำ (Duplicate) ถ้า:
- Outbox Processor ส่งสำเร็จแต่ Crash ก่อน Mark PROCESSED
- Network Error ทำให้ Producer Retry

ดังนั้น Consumer ต้องจัดการ **Duplicate Events** ด้วย **Inbox Pattern**

### 7.2 Inbox Table

```sql
-- migrations/003_create_inbox_table.sql

CREATE TABLE inbox_events (
  id UUID PRIMARY KEY,        -- ใช้ Event ID เป็น Primary Key
  event_type VARCHAR(255) NOT NULL,
  aggregate_type VARCHAR(100) NOT NULL,
  aggregate_id VARCHAR(255) NOT NULL,
  payload JSONB NOT NULL,
  processed_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  
  -- ป้องกัน ID collision ข้าม event types
  CONSTRAINT unique_event UNIQUE (id, event_type)
);

-- Auto-cleanup (เก็บไว้ 30 วัน)
CREATE INDEX idx_inbox_processed_at ON inbox_events(processed_at DESC);
```

### 7.3 InboxService

```typescript
// src/inbox/inbox.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { DataSource, EntityManager } from 'typeorm';
import { InboxEvent } from '../entities/inbox-event.entity';

@Injectable()
export class InboxService {
  private readonly logger = new Logger(InboxService.name);

  constructor(
    private dataSource: DataSource,
  ) {}

  /**
   * Execute callback ถ้ายังไม่เคย Process Event นี้มาก่อน
   * ถ้าซ้ำ จะ skip อัตโนมัติ (Idempotent)
   */
  async processOnce<T>(
    eventId: string,
    eventType: string,
    aggregateType: string,
    aggregateId: string,
    payload: any,
    callback: (manager: EntityManager) => Promise<T>,
  ): Promise<{ result: T | null; isDuplicate: boolean }> {
    return this.dataSource.transaction(async (manager) => {
      // Try to insert into inbox (will fail if duplicate)
      try {
        await manager.query(
          `INSERT INTO inbox_events (id, event_type, aggregate_type, aggregate_id, payload)
           VALUES ($1, $2, $3, $4, $5)
           ON CONFLICT DO NOTHING`,
          [eventId, eventType, aggregateType, aggregateId, JSON.stringify(payload)],
        );

        // Check if actually inserted (not duplicate)
        const inserted = await manager.findOne(InboxEvent, {
          where: { id: eventId },
        });

        if (!inserted) {
          this.logger.debug(`Duplicate event detected: ${eventId} (${eventType})`);
          return { result: null, isDuplicate: true };
        }

        // Execute the actual business logic
        const result = await callback(manager);
        return { result, isDuplicate: false };

      } catch (error) {
        if (error.code === '23505') { // Unique violation
          this.logger.debug(`Duplicate event (concurrent): ${eventId}`);
          return { result: null, isDuplicate: true };
        }
        throw error;
      }
    });
  }
}
```

### 7.4 Consumer ด้วย Inbox Pattern

```typescript
// src/consumers/order-events.consumer.ts
import { Injectable, Logger } from '@nestjs/common';
import { KafkaContext, Payload, EventPattern } from '@nestjs/microservices';
import { InboxService } from '../inbox/inbox.service';
import { InventoryService } from '../inventory/inventory.service';

@Injectable()
export class OrderEventsConsumer {
  private readonly logger = new Logger(OrderEventsConsumer.name);

  constructor(
    private inboxService: InboxService,
    private inventoryService: InventoryService,
  ) {}

  @EventPattern('orders')
  async handleOrderEvent(
    @Payload() message: any,
  ): Promise<void> {
    const eventId = message.headers?.['event-id'] || message.id;
    const eventType = message.eventType;

    this.logger.log(`Received event: ${eventType} (${eventId})`);

    const { result, isDuplicate } = await this.inboxService.processOnce(
      eventId,
      eventType,
      message.aggregateType,
      message.aggregateId,
      message.payload,
      async (manager) => {
        // Business logic - only executes if not duplicate
        switch (eventType) {
          case 'OrderCreated':
            return this.inventoryService.reserveForOrder(
              manager,
              message.payload,
            );

          case 'OrderCancelled':
            return this.inventoryService.releaseForOrder(
              manager,
              message.payload,
            );

          default:
            this.logger.warn(`Unhandled event type: ${eventType}`);
        }
      },
    );

    if (isDuplicate) {
      this.logger.debug(`Skipped duplicate event: ${eventId}`);
    }
  }
}
```

---

## 8. Dead Letter Queue

### 8.1 จัดการ Events ที่ล้มเหลวซ้ำ

```typescript
// src/outbox/dead-letter.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { InjectDataSource } from '@nestjs/typeorm';
import { DataSource } from 'typeorm';
import { Cron, CronExpression } from '@nestjs/schedule';

@Injectable()
export class DeadLetterService {
  private readonly logger = new Logger(DeadLetterService.name);

  constructor(
    @InjectDataSource()
    private dataSource: DataSource,
  ) {}

  // ตรวจสอบ Dead Events ทุก 5 นาที
  @Cron('*/5 * * * *')
  async alertDeadEvents(): Promise<void> {
    const deadEvents = await this.dataSource.query(
      `SELECT 
         aggregate_type,
         event_type,
         COUNT(*) as count,
         MIN(created_at) as oldest,
         MAX(last_error) as latest_error
       FROM outbox_events
       WHERE status = 'DEAD'
         AND created_at >= NOW() - INTERVAL '24 hours'
       GROUP BY aggregate_type, event_type
       ORDER BY count DESC`,
    );

    if (deadEvents.length > 0) {
      this.logger.warn(`Dead events found: ${JSON.stringify(deadEvents)}`);
      // Send alert to Slack/PagerDuty
      await this.sendAlert(deadEvents);
    }
  }

  async retryDeadEvent(eventId: string): Promise<void> {
    await this.dataSource.query(
      `UPDATE outbox_events
       SET status = 'PENDING',
           retry_count = 0,
           scheduled_at = NOW(),
           last_error = NULL
       WHERE id = $1 AND status = 'DEAD'`,
      [eventId],
    );
    this.logger.log(`Dead event ${eventId} scheduled for retry`);
  }

  async getDeadEvents(limit = 100): Promise<any[]> {
    return this.dataSource.query(
      `SELECT 
         id, aggregate_type, aggregate_id, event_type,
         payload, last_error, retry_count, created_at
       FROM outbox_events
       WHERE status = 'DEAD'
       ORDER BY created_at DESC
       LIMIT $1`,
      [limit],
    );
  }

  private async sendAlert(deadEvents: any[]): Promise<void> {
    // Implement Slack/Email notification
  }
}
```

---

## 9. Outbox vs Saga: ใช้ร่วมกัน

### 9.1 Pattern Combination

```
┌─────────────────────────────────────────────────────────────┐
│                      Order Service                           │
│                                                              │
│  createOrder():                                              │
│    BEGIN TRANSACTION                                         │
│      INSERT INTO orders (...)          ← Business data       │
│      INSERT INTO outbox_events (...)   ← Saga start event   │
│    COMMIT                                                    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                   Outbox Processor                           │
│  Reads: outbox_events WHERE status='PENDING'                │
│  Publishes: OrderCreated → Kafka topic 'saga-events'        │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                  Saga Orchestrator                           │
│  Consumes: 'saga-events' topic                              │
│  Starts: OrderSaga                                          │
│    Step 1: Call Inventory Service                           │
│    Step 2: Call Payment Service                             │
│    Step 3: Confirm Order                                    │
│                                                             │
│  Each step uses Outbox Pattern too!                         │
└─────────────────────────────────────────────────────────────┘
```

### 9.2 Full Integration Example

```typescript
// src/saga/saga-step.service.ts
import { Injectable } from '@nestjs/common';
import { DataSource } from 'typeorm';
import { OutboxService } from '../outbox/outbox.service';

@Injectable()
export class SagaStepService {
  constructor(
    private dataSource: DataSource,
    private outboxService: OutboxService,
  ) {}

  async reserveInventory(sagaId: string, orderId: string, items: any[]): Promise<void> {
    await this.dataSource.transaction(async (manager) => {
      // Update saga step in DB
      await manager.query(
        `UPDATE saga_instances 
         SET current_step = 'RESERVE_INVENTORY', updated_at = NOW()
         WHERE id = $1`,
        [sagaId],
      );

      // Send command via Outbox (guaranteed delivery!)
      await this.outboxService.publish(manager, {
        aggregateType: 'Saga',
        aggregateId: sagaId,
        eventType: 'ReserveInventoryCommand',
        payload: { sagaId, orderId, items },
        destinationTopic: 'inventory-commands',
        partitionKey: orderId,
        idempotencyKey: `saga:${sagaId}:reserve-inventory`,
      });
    });
  }
}
```

---

## 10. Testing Outbox Pattern

### 10.1 Unit Test

```typescript
// tests/outbox.service.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { DataSource } from 'typeorm';
import { OutboxService } from '../src/outbox/outbox.service';
import { OutboxStatus } from '../src/entities/outbox-event.entity';

describe('OutboxService', () => {
  let outboxService: OutboxService;
  let mockManager: any;

  beforeEach(async () => {
    mockManager = {
      findOne: jest.fn(),
      create: jest.fn(),
      save: jest.fn(),
    };

    const module: TestingModule = await Test.createTestingModule({
      providers: [OutboxService],
    }).compile();

    outboxService = module.get<OutboxService>(OutboxService);
  });

  it('should create outbox event with correct data', async () => {
    mockManager.findOne.mockResolvedValue(null);
    mockManager.create.mockImplementation((_, data) => data);
    mockManager.save.mockImplementation((_, data) => ({ ...data, id: 'event-123' }));

    const result = await outboxService.publish(mockManager, {
      aggregateType: 'Order',
      aggregateId: 'order-001',
      eventType: 'OrderCreated',
      payload: { orderId: 'order-001', status: 'PENDING' },
      destinationTopic: 'orders',
    });

    expect(result.aggregateType).toBe('Order');
    expect(result.eventType).toBe('OrderCreated');
    expect(result.status).toBe(OutboxStatus.PENDING);
    expect(mockManager.save).toHaveBeenCalledTimes(1);
  });

  it('should return existing event for duplicate idempotency key', async () => {
    const existingEvent = { id: 'existing-123', idempotencyKey: 'test-key' };
    mockManager.findOne.mockResolvedValue(existingEvent);

    const result = await outboxService.publish(mockManager, {
      aggregateType: 'Order',
      aggregateId: 'order-001',
      eventType: 'OrderCreated',
      payload: {},
      destinationTopic: 'orders',
      idempotencyKey: 'test-key',
    });

    expect(result).toBe(existingEvent);
    expect(mockManager.save).not.toHaveBeenCalled();
  });
});
```

### 10.2 Integration Test: Simulate Kafka Failure

```typescript
// tests/outbox-processor.integration.spec.ts
describe('OutboxProcessor Integration', () => {
  let app: INestApplication;
  let dataSource: DataSource;
  let kafkaProducer: jest.Mocked<Producer>;

  beforeAll(async () => {
    // Setup test database
    await setupTestDatabase();
    
    // Mock Kafka producer
    kafkaProducer = {
      send: jest.fn(),
      connect: jest.fn().mockResolvedValue(undefined),
      disconnect: jest.fn().mockResolvedValue(undefined),
    } as any;
  });

  it('should retry failed events', async () => {
    // Insert a pending outbox event
    await dataSource.query(
      `INSERT INTO outbox_events (id, aggregate_type, aggregate_id, event_type, payload, destination_topic)
       VALUES ('test-event-1', 'Order', 'order-1', 'OrderCreated', '{}', 'orders')`
    );

    // First attempt: Kafka fails
    kafkaProducer.send.mockRejectedValueOnce(new Error('Kafka unavailable'));
    
    await outboxProcessor.processBatch();
    
    // Check event is FAILED with retry scheduled
    const event = await dataSource.query(
      `SELECT * FROM outbox_events WHERE id = 'test-event-1'`
    );
    expect(event[0].status).toBe('FAILED');
    expect(event[0].retry_count).toBe(1);
    expect(new Date(event[0].scheduled_at)).toBeGreaterThan(new Date());

    // Second attempt: Kafka succeeds
    kafkaProducer.send.mockResolvedValueOnce({});
    
    // Fast-forward scheduled_at
    await dataSource.query(
      `UPDATE outbox_events SET scheduled_at = NOW() WHERE id = 'test-event-1'`
    );
    
    await outboxProcessor.processBatch();
    
    const processedEvent = await dataSource.query(
      `SELECT * FROM outbox_events WHERE id = 'test-event-1'`
    );
    expect(processedEvent[0].status).toBe('PROCESSED');
    expect(kafkaProducer.send).toHaveBeenCalledTimes(2);
  });

  it('should move event to DEAD status after max retries', async () => {
    await dataSource.query(
      `INSERT INTO outbox_events (id, aggregate_type, aggregate_id, event_type, payload, destination_topic, retry_count, max_retries)
       VALUES ('test-event-2', 'Order', 'order-2', 'OrderCreated', '{}', 'orders', 4, 5)`
    );

    kafkaProducer.send.mockRejectedValue(new Error('Persistent Kafka error'));
    
    await outboxProcessor.processBatch();
    
    const event = await dataSource.query(
      `SELECT * FROM outbox_events WHERE id = 'test-event-2'`
    );
    expect(event[0].status).toBe('DEAD');
  });
});
```

---

## 11. Monitoring และ Metrics

```typescript
// src/outbox/outbox.metrics.ts
import { Gauge, Counter, Histogram } from 'prom-client';

export const outboxPendingEvents = new Gauge({
  name: 'outbox_pending_events',
  help: 'Number of pending outbox events',
  labelNames: ['aggregate_type'],
});

export const outboxProcessingDuration = new Histogram({
  name: 'outbox_processing_duration_seconds',
  help: 'Time to process outbox events',
  buckets: [0.01, 0.05, 0.1, 0.5, 1, 5],
});

export const outboxFailedTotal = new Counter({
  name: 'outbox_failed_total',
  help: 'Total failed outbox events',
  labelNames: ['aggregate_type', 'event_type'],
});

export const outboxDeadTotal = new Gauge({
  name: 'outbox_dead_total',
  help: 'Total dead outbox events requiring manual intervention',
});
```

```sql
-- Monitoring Query: Outbox health dashboard
SELECT
  status,
  COUNT(*) as count,
  AVG(retry_count) as avg_retries,
  MIN(created_at) as oldest_event,
  MAX(created_at) as newest_event
FROM outbox_events
WHERE created_at >= NOW() - INTERVAL '1 hour'
GROUP BY status;

-- Events older than 10 minutes that are still PENDING (possible stuck processor)
SELECT COUNT(*) as stuck_events
FROM outbox_events  
WHERE status = 'PENDING'
  AND created_at < NOW() - INTERVAL '10 minutes';
```

---

## 12. สรุป

### 12.1 Outbox Pattern Benefits

1. **Guaranteed Delivery**: Event ไม่หายแน่นอน เพราะอยู่ใน DB แล้ว
2. **Atomic**: Business Data + Event บันทึกพร้อมกันใน Transaction เดียว
3. **No Two-Phase Commit**: ไม่ต้องการ Distributed Transaction
4. **Exactly-Once Processing**: ด้วย Inbox Pattern ที่ Consumer ฝั่ง
5. **Auditability**: ดูได้ว่า Events ถูกส่งไปเมื่อไหร่

### 12.2 Best Practices

| Practice | เหตุผล |
|----------|--------|
| ใช้ Idempotency Key | ป้องกัน Duplicate inserts |
| SKIP LOCKED | ป้องกัน Multiple processors claim event เดียวกัน |
| Exponential Backoff | ลด Load บน Kafka ขณะมีปัญหา |
| Dead Letter Queue | จัดการ Unprocessable events |
| Monitor Pending Age | Alert เมื่อ Events ค้างนานผิดปกติ |
| Inbox Pattern | Idempotent consumption ที่ Consumer |

### 12.3 เมื่อไหร่ควรใช้ Outbox

- ต้องการ Guaranteed Event Delivery
- Business Operation ต้องการ Atomicity ระหว่าง DB Write และ Event Publish
- ระบบมีความสำคัญสูง เช่น Financial Transactions, Order Processing
- ต้องการ Audit Trail ของทุก Event

Outbox Pattern เป็นหนึ่งใน Patterns พื้นฐานที่สำคัญที่สุดใน Microservices Event-Driven Architecture ร่วมกับ Saga Pattern ช่วยให้ระบบมี Reliability และ Data Consistency ในระดับสูง

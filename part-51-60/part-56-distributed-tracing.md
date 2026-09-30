# Part 56: Distributed Tracing ด้วย Jaeger/Zipkin/OpenTelemetry

## บทนำ: ทำไมต้องใช้ Distributed Tracing?

ในระบบ Microservices หรือ Database Cluster ที่ซับซ้อน เมื่อ request หนึ่งเดินทางผ่านหลาย service การ debug ปัญหาด้านประสิทธิภาพกลายเป็นเรื่องยากมาก ลองนึกภาพ:

```
Client → API Gateway → Auth Service → User Service → PostgreSQL → Redis → Response
```

ถ้า latency สูงผิดปกติ เราจะรู้ได้อย่างไรว่าปัญหาอยู่ที่ไหน? Logs บอกได้เพียงบางส่วน Metrics บอก trend แต่ไม่บอก path ที่แท้จริง นี่คือที่มาของ **Distributed Tracing**

### ปัญหาที่ Distributed Tracing แก้ได้

1. **Latency Analysis**: หาว่า bottleneck อยู่ที่ service ไหน
2. **Dependency Mapping**: เข้าใจ service topology อัตโนมัติ
3. **Error Propagation**: ติดตามว่า error เกิดจากที่ไหนและกระทบอะไรบ้าง
4. **Performance Regression**: เปรียบเทียบ trace ระหว่าง version
5. **Debugging Complex Flows**: ติดตาม request ที่ผ่าน multiple services

### ตัวอย่างสถานการณ์จริง

```
# ลูกค้าร้องเรียนว่า checkout ช้า
# เราต้องหาว่าช้าที่ไหน?

UserRequest (500ms total)
├── Auth check (5ms)           ✓ เร็ว
├── Load cart items (10ms)     ✓ เร็ว  
├── Check inventory (450ms)    ✗ ช้ามาก! ← นี่คือปัญหา
│   ├── Query PostgreSQL (400ms) ← Missing index!
│   └── Update Redis cache (50ms)
└── Process payment (35ms)     ✓ เร็ว
```

Distributed Tracing ทำให้เห็น "ภาพทั้งหมด" นี้ได้ในครั้งเดียว

---

## Core Concepts: Trace, Span, Context Propagation

### 1. Trace

**Trace** คือการบันทึก path ทั้งหมดของ request หนึ่งๆ ตั้งแต่เริ่มต้นจนสิ้นสุด แต่ละ Trace มี **Trace ID** ที่ unique

```
Trace ID: abc123def456
┌─────────────────────────────────────────────────────┐
│  GET /api/orders/123                        500ms   │
│  ├── Authenticate user                       10ms   │
│  ├── Fetch order from DB                    200ms   │
│  │   └── SELECT * FROM orders WHERE id=123  200ms   │
│  ├── Fetch user details                      50ms   │
│  ├── Calculate totals                         5ms   │
│  └── Return response                          1ms   │
└─────────────────────────────────────────────────────┘
```

### 2. Span

**Span** คือหน่วยงานเล็กที่สุดใน Trace แทนการทำงานชิ้นหนึ่งๆ เช่น HTTP call, DB query, cache lookup แต่ละ Span มี:

- **Span ID**: unique identifier
- **Trace ID**: บอกว่าเป็นส่วนของ Trace ใด
- **Parent Span ID**: บอก parent span (ถ้ามี)
- **Operation Name**: ชื่อการทำงาน เช่น "SELECT orders"
- **Start Time**: เวลาเริ่ม
- **Duration**: ระยะเวลา
- **Tags/Attributes**: metadata เพิ่มเติม เช่น `db.type=postgresql`
- **Events/Logs**: เหตุการณ์ที่เกิดขึ้นระหว่าง span
- **Status**: OK, ERROR

```typescript
// ตัวอย่าง Span structure
interface Span {
  traceId: string;       // "abc123def456"
  spanId: string;        // "span001"
  parentSpanId?: string; // "root001"
  name: string;          // "PostgreSQL SELECT"
  startTime: number;     // Unix timestamp nanoseconds
  endTime: number;       // Unix timestamp nanoseconds
  attributes: Record<string, string | number | boolean>;
  events: SpanEvent[];
  status: { code: 'OK' | 'ERROR'; message?: string };
}
```

### 3. Context Propagation

Context Propagation คือกลไกส่ง Trace ID และ Span ID ไปกับ request ข้าม service boundaries เพื่อให้ Spans จาก service ต่างๆ รู้ว่าตัวเองเป็นส่วนของ Trace เดียวกัน

**W3C TraceContext Standard (RFC 7230)**:
```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             ^^ ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ ^^
             version  trace-id (16 bytes hex)    parent-id  flags

tracestate: vendor1=value1,vendor2=value2
```

ตัวอย่างการส่ง header:
```http
GET /api/inventory/check HTTP/1.1
Host: inventory-service:3001
traceparent: 00-abc123def456abc123def456abc12345-0102030405060708-01
tracestate: myapp=custom_value
```

---

## OpenTelemetry (OTel): มาตรฐานใหม่ของ Observability

**OpenTelemetry** เป็น CNCF project ที่รวม observability signals ทั้งหมด (Traces, Metrics, Logs) ไว้ในมาตรฐานเดียว แทนที่จะต้องใช้ SDK แยกสำหรับ Jaeger, Zipkin, etc.

### OTel Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Your Application                     │
│  ┌──────────────────────────────────────────────────┐  │
│  │              OpenTelemetry SDK                   │  │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────────────┐ │  │
│  │  │  Tracer  │ │  Meter   │ │  Logger Provider │ │  │
│  │  └──────────┘ └──────────┘ └──────────────────┘ │  │
│  │         │           │                │           │  │
│  │  ┌──────────────────────────────────────────┐   │  │
│  │  │         OTel Processor/Pipeline          │   │  │
│  │  └──────────────────────────────────────────┘   │  │
│  └──────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
                          │
              ┌───────────┼───────────┐
              ▼           ▼           ▼
         ┌────────┐  ┌────────┐  ┌────────┐
         │ Jaeger │  │ Zipkin │  │ OTLP   │
         │Exporter│  │Exporter│  │Exporter│
         └────────┘  └────────┘  └────────┘
              │           │           │
              ▼           ▼           ▼
         ┌────────┐  ┌────────┐  ┌─────────────┐
         │ Jaeger │  │ Zipkin │  │ OTel         │
         │ Backend│  │Backend │  │ Collector    │
         └────────┘  └────────┘  └─────────────┘
```

### ติดตั้ง OTel SDK สำหรับ Node.js

```bash
# Core packages
npm install @opentelemetry/sdk-node
npm install @opentelemetry/api

# Auto-instrumentation (ครอบคลุม libraries หลัก)
npm install @opentelemetry/auto-instrumentations-node

# Exporters
npm install @opentelemetry/exporter-trace-otlp-http
npm install @opentelemetry/exporter-jaeger
npm install @opentelemetry/exporter-zipkin

# Resources (add service metadata)
npm install @opentelemetry/resources
npm install @opentelemetry/semantic-conventions

# TypeScript types
npm install --save-dev @types/node
```

### โครงสร้าง Project

```
src/
├── tracing/
│   ├── tracer.ts        # OTel configuration
│   ├── middleware.ts    # Express tracing middleware
│   └── helpers.ts       # Helper functions
├── database/
│   ├── postgres.ts      # PostgreSQL with tracing
│   └── redis.ts         # Redis with tracing
├── routes/
│   └── orders.ts        # Example route
└── index.ts             # Application entry point
```

---

## Auto-Instrumentation: ตั้งค่าครั้งเดียว ได้ทั้งหมด

### การตั้งค่า Auto-Instrumentation

สร้างไฟล์ `src/tracing/tracer.ts`:

```typescript
import { NodeSDK } from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';
import { JaegerExporter } from '@opentelemetry/exporter-jaeger';
import { Resource } from '@opentelemetry/resources';
import { SemanticResourceAttributes } from '@opentelemetry/semantic-conventions';
import {
  BatchSpanProcessor,
  ConsoleSpanExporter,
  SimpleSpanProcessor,
} from '@opentelemetry/sdk-trace-base';
import { W3CTraceContextPropagator } from '@opentelemetry/core';
import { CompositePropagator } from '@opentelemetry/core';
import { B3Propagator } from '@opentelemetry/propagator-b3';

// ------------------- Configuration -------------------

const SERVICE_NAME = process.env.SERVICE_NAME || 'my-app';
const SERVICE_VERSION = process.env.SERVICE_VERSION || '1.0.0';
const JAEGER_ENDPOINT = process.env.JAEGER_ENDPOINT || 'http://localhost:4318/v1/traces';
const ENVIRONMENT = process.env.NODE_ENV || 'development';

// ------------------- Exporters -------------------

// OTLP Exporter (ส่งไป OTel Collector หรือ Jaeger ที่รองรับ OTLP)
const otlpExporter = new OTLPTraceExporter({
  url: JAEGER_ENDPOINT,
  headers: {
    // ถ้าต้องการ authentication
    // 'Authorization': `Bearer ${process.env.OTEL_AUTH_TOKEN}`,
  },
});

// Jaeger Exporter (โดยตรง ไม่ผ่าน Collector)
const jaegerExporter = new JaegerExporter({
  endpoint: process.env.JAEGER_HTTP_ENDPOINT || 'http://localhost:14268/api/traces',
});

// Console Exporter สำหรับ debug
const consoleExporter = new ConsoleSpanExporter();

// เลือก exporter ตาม environment
function getExporter() {
  switch (ENVIRONMENT) {
    case 'production':
      return otlpExporter;
    case 'staging':
      return jaegerExporter;
    default:
      // Development: ส่งทั้ง Jaeger และ Console
      return otlpExporter;
  }
}

// ------------------- Resource Attributes -------------------

const resource = new Resource({
  [SemanticResourceAttributes.SERVICE_NAME]: SERVICE_NAME,
  [SemanticResourceAttributes.SERVICE_VERSION]: SERVICE_VERSION,
  [SemanticResourceAttributes.DEPLOYMENT_ENVIRONMENT]: ENVIRONMENT,
  // Custom attributes
  'team.name': 'backend',
  'datacenter.region': process.env.DATACENTER_REGION || 'th-central-1',
});

// ------------------- SDK Setup -------------------

export const sdk = new NodeSDK({
  resource,
  
  // Span Processors
  spanProcessor: new BatchSpanProcessor(getExporter(), {
    // Batch configuration สำหรับ production
    maxQueueSize: 2048,      // Maximum spans in queue
    maxExportBatchSize: 512,  // Spans per batch
    scheduledDelayMillis: 5000,  // Export every 5 seconds
    exportTimeoutMillis: 30000,  // Timeout per export
  }),
  
  // Auto-instrumentation สำหรับ libraries ทั้งหมด
  instrumentations: [
    getNodeAutoInstrumentations({
      // HTTP/HTTPS instrumentation
      '@opentelemetry/instrumentation-http': {
        enabled: true,
        ignoreIncomingRequestHook: (req) => {
          // ไม่ trace health check endpoints
          return req.url === '/health' || req.url === '/metrics';
        },
        requestHook: (span, req) => {
          // เพิ่ม custom attributes
          span.setAttribute('http.client_ip', 
            (req as any).socket?.remoteAddress || 'unknown');
        },
      },
      
      // Express instrumentation
      '@opentelemetry/instrumentation-express': {
        enabled: true,
        ignoreLayers: [
          // ไม่ trace middleware เหล่านี้
          'cors',
          'helmet',
          'body-parser',
        ],
      },
      
      // PostgreSQL instrumentation (node-postgres)
      '@opentelemetry/instrumentation-pg': {
        enabled: true,
        enhancedDatabaseReporting: true, // เพิ่ม db.statement
        addSqlCommenterCommentToQueries: true, // เพิ่ม trace info ใน SQL comment
      },
      
      // ioredis instrumentation
      '@opentelemetry/instrumentation-ioredis': {
        enabled: true,
        dbStatementSerializer: (cmdName, cmdArgs) => {
          // Serialize Redis commands สำหรับ span name
          return `${cmdName} ${cmdArgs[0] || ''}`.substring(0, 100);
        },
      },
      
      // DNS instrumentation
      '@opentelemetry/instrumentation-dns': {
        enabled: true,
      },
      
      // Net instrumentation
      '@opentelemetry/instrumentation-net': {
        enabled: false, // ปิดเพราะ noise มาก
      },
      
      // gRPC instrumentation
      '@opentelemetry/instrumentation-grpc': {
        enabled: false, // เปิดถ้าใช้ gRPC
      },
    }),
  ],
  
  // Context Propagation
  textMapPropagator: new CompositePropagator({
    propagators: [
      new W3CTraceContextPropagator(), // traceparent, tracestate headers
      new B3Propagator(),               // X-B3-TraceId, X-B3-SpanId headers (Zipkin compat)
    ],
  }),
});

// ------------------- Initialize -------------------

export function initTracing(): void {
  sdk.start();
  console.log(`[OTel] Tracing initialized for service: ${SERVICE_NAME}`);
  console.log(`[OTel] Exporting to: ${JAEGER_ENDPOINT}`);
  
  // Graceful shutdown
  process.on('SIGTERM', () => {
    sdk.shutdown()
      .then(() => console.log('[OTel] Tracing terminated'))
      .catch((error) => console.error('[OTel] Error terminating tracing', error))
      .finally(() => process.exit(0));
  });
}

// Export tracer สำหรับ manual instrumentation
export { trace, context, SpanStatusCode, SpanKind } from '@opentelemetry/api';
export type { Span, Tracer, Context } from '@opentelemetry/api';
```

### สิ่งที่ Auto-Instrumentation ครอบคลุม

```typescript
// ตัวอย่าง Libraries ที่ Auto-Instrument ได้:

// 1. HTTP/HTTPS requests
import http from 'http';
const req = http.get('http://api.example.com/data', (res) => {
  // Auto-creates span: "HTTP GET"
  // Attributes: http.url, http.method, http.status_code
});

// 2. Express routes
app.get('/users/:id', (req, res) => {
  // Auto-creates span: "GET /users/:id"
  // Attributes: http.route, http.method, net.peer.ip
});

// 3. PostgreSQL queries
const client = new pg.Client({ connectionString });
await client.query('SELECT * FROM users WHERE id = $1', [userId]);
// Auto-creates span: "pg.query:SELECT"
// Attributes: db.type=postgresql, db.statement, db.name

// 4. Redis operations
const redis = new Redis({ host: 'localhost', port: 6379 });
await redis.get('user:123');
// Auto-creates span: "redis-GET"
// Attributes: db.type=redis, db.statement="GET user:123"

// 5. Axios HTTP client
import axios from 'axios';
await axios.get('https://api.external.com/data');
// Auto-creates span with HTTP details

// 6. gRPC calls (ถ้าเปิดใช้)
const client = new MyServiceClient(address, credentials);
client.myMethod(request, (err, response) => {});
// Auto-creates span: "grpc.MyService/myMethod"
```

---

## Manual Instrumentation: ควบคุมทุกรายละเอียด

### ตั้งค่า Manual Tracer

```typescript
// src/tracing/helpers.ts
import { trace, context, SpanStatusCode, SpanKind } from '@opentelemetry/api';
import type { Span, Tracer, Context, SpanOptions } from '@opentelemetry/api';

// ดึง tracer สำหรับ service
export function getTracer(name: string = 'default'): Tracer {
  return trace.getTracer(name, '1.0.0');
}

// Helper: สร้าง span และ run function ภายใน
export async function withSpan<T>(
  name: string,
  fn: (span: Span) => Promise<T>,
  options?: SpanOptions
): Promise<T> {
  const tracer = getTracer();
  return tracer.startActiveSpan(name, options || {}, async (span) => {
    try {
      const result = await fn(span);
      span.setStatus({ code: SpanStatusCode.OK });
      return result;
    } catch (error) {
      span.setStatus({
        code: SpanStatusCode.ERROR,
        message: error instanceof Error ? error.message : String(error),
      });
      span.recordException(error as Error);
      throw error;
    } finally {
      span.end();
    }
  });
}

// Helper: เพิ่ม attributes ใน current span
export function addSpanAttributes(attributes: Record<string, string | number | boolean>): void {
  const currentSpan = trace.getActiveSpan();
  if (currentSpan) {
    Object.entries(attributes).forEach(([key, value]) => {
      currentSpan.setAttribute(key, value);
    });
  }
}

// Helper: เพิ่ม event ใน current span
export function addSpanEvent(name: string, attributes?: Record<string, string | number | boolean>): void {
  const currentSpan = trace.getActiveSpan();
  if (currentSpan) {
    currentSpan.addEvent(name, attributes);
  }
}

// Helper: ดึง Trace ID จาก current context
export function getCurrentTraceId(): string | undefined {
  const currentSpan = trace.getActiveSpan();
  if (currentSpan) {
    return currentSpan.spanContext().traceId;
  }
  return undefined;
}
```

### สร้าง Spans แบบต่างๆ

```typescript
// src/routes/orders.ts
import { trace, SpanStatusCode, SpanKind, context } from '@opentelemetry/api';
import { withSpan, addSpanAttributes, addSpanEvent } from '../tracing/helpers';
import { pool } from '../database/postgres';
import { redis } from '../database/redis';

const tracer = trace.getTracer('orders-service');

// ------------------- Create Order -------------------

export async function createOrder(
  userId: string,
  items: OrderItem[]
): Promise<Order> {
  // สร้าง parent span สำหรับ operation ทั้งหมด
  return tracer.startActiveSpan('createOrder', {
    kind: SpanKind.INTERNAL,
    attributes: {
      'order.user_id': userId,
      'order.item_count': items.length,
    },
  }, async (parentSpan) => {
    try {
      // 1. Validate items
      addSpanEvent('validation.start');
      const validatedItems = await validateItems(items, parentSpan);
      addSpanEvent('validation.complete', {
        'validation.items_valid': validatedItems.length,
      });
      
      // 2. Check inventory
      const inventory = await checkInventory(validatedItems);
      
      // 3. Calculate pricing
      const pricing = await calculatePricing(userId, validatedItems);
      
      // 4. Create order in database
      const order = await saveOrderToDatabase(userId, validatedItems, pricing);
      
      // 5. Update inventory
      await updateInventory(validatedItems, order.id);
      
      // เพิ่ม result attributes
      parentSpan.setAttribute('order.id', order.id);
      parentSpan.setAttribute('order.total', order.total);
      parentSpan.setStatus({ code: SpanStatusCode.OK });
      
      return order;
    } catch (error) {
      parentSpan.setStatus({
        code: SpanStatusCode.ERROR,
        message: error instanceof Error ? error.message : 'Unknown error',
      });
      parentSpan.recordException(error as Error);
      throw error;
    } finally {
      parentSpan.end();
    }
  });
}

// ------------------- Check Inventory -------------------

async function checkInventory(items: OrderItem[]): Promise<InventoryStatus[]> {
  return withSpan('checkInventory', async (span) => {
    span.setAttribute('inventory.item_count', items.length);
    
    // ลองดึงจาก cache ก่อน
    const cacheKey = `inventory:${items.map(i => i.productId).sort().join(',')}`;
    
    const cached = await tracer.startActiveSpan('redis.get', {
      kind: SpanKind.CLIENT,
      attributes: {
        'db.type': 'redis',
        'db.operation': 'GET',
        'cache.key': cacheKey,
      },
    }, async (cacheSpan) => {
      try {
        const value = await redis.get(cacheKey);
        cacheSpan.setAttribute('cache.hit', value !== null);
        cacheSpan.setStatus({ code: SpanStatusCode.OK });
        return value ? JSON.parse(value) : null;
      } finally {
        cacheSpan.end();
      }
    });
    
    if (cached) {
      span.addEvent('cache.hit', { 'cache.key': cacheKey });
      span.setAttribute('cache.hit', true);
      return cached;
    }
    
    span.addEvent('cache.miss', { 'cache.key': cacheKey });
    span.setAttribute('cache.hit', false);
    
    // ดึงจาก Database
    const inventoryData = await tracer.startActiveSpan('postgres.query', {
      kind: SpanKind.CLIENT,
      attributes: {
        'db.type': 'postgresql',
        'db.operation': 'SELECT',
        'db.table': 'inventory',
        'db.statement': 'SELECT product_id, quantity FROM inventory WHERE product_id = ANY($1)',
      },
    }, async (dbSpan) => {
      try {
        const productIds = items.map(i => i.productId);
        const result = await pool.query(
          'SELECT product_id, quantity FROM inventory WHERE product_id = ANY($1)',
          [productIds]
        );
        
        dbSpan.setAttribute('db.rows_returned', result.rows.length);
        dbSpan.setStatus({ code: SpanStatusCode.OK });
        
        return result.rows;
      } catch (error) {
        dbSpan.setStatus({
          code: SpanStatusCode.ERROR,
          message: (error as Error).message,
        });
        dbSpan.recordException(error as Error);
        throw error;
      } finally {
        dbSpan.end();
      }
    });
    
    // บันทึกลง cache
    await redis.setex(cacheKey, 60, JSON.stringify(inventoryData));
    span.addEvent('cache.set', { 'cache.key': cacheKey, 'cache.ttl': 60 });
    
    return inventoryData;
  });
}

// ------------------- Save Order to Database (Transaction) -------------------

async function saveOrderToDatabase(
  userId: string,
  items: OrderItem[],
  pricing: PricingResult
): Promise<Order> {
  return tracer.startActiveSpan('postgres.transaction', {
    kind: SpanKind.CLIENT,
    attributes: {
      'db.type': 'postgresql',
      'db.operation': 'transaction',
      'db.items_count': items.length,
    },
  }, async (txSpan) => {
    const client = await pool.connect();
    
    try {
      await client.query('BEGIN');
      txSpan.addEvent('transaction.begin');
      
      // Insert order
      const orderResult = await client.query(
        `INSERT INTO orders (user_id, total_amount, status, created_at)
         VALUES ($1, $2, 'pending', NOW())
         RETURNING id, created_at`,
        [userId, pricing.total]
      );
      const orderId = orderResult.rows[0].id;
      txSpan.addEvent('order.inserted', { 'order.id': orderId });
      
      // Insert order items
      for (const item of items) {
        await client.query(
          `INSERT INTO order_items (order_id, product_id, quantity, unit_price)
           VALUES ($1, $2, $3, $4)`,
          [orderId, item.productId, item.quantity, item.unitPrice]
        );
      }
      txSpan.addEvent('order_items.inserted', { 'items.count': items.length });
      
      await client.query('COMMIT');
      txSpan.addEvent('transaction.commit');
      txSpan.setAttribute('order.id', orderId);
      txSpan.setStatus({ code: SpanStatusCode.OK });
      
      return { id: orderId, ...orderResult.rows[0], total: pricing.total };
    } catch (error) {
      await client.query('ROLLBACK');
      txSpan.addEvent('transaction.rollback', {
        'error.message': (error as Error).message,
      });
      txSpan.setStatus({
        code: SpanStatusCode.ERROR,
        message: (error as Error).message,
      });
      txSpan.recordException(error as Error);
      throw error;
    } finally {
      client.release();
      txSpan.end();
    }
  });
}
```

### บันทึก Exception อย่างถูกต้อง

```typescript
// ------------------- Exception Recording -------------------

async function processPayment(orderId: string, amount: number): Promise<PaymentResult> {
  return tracer.startActiveSpan('processPayment', async (span) => {
    span.setAttribute('payment.order_id', orderId);
    span.setAttribute('payment.amount', amount);
    
    try {
      // เรียก payment gateway
      const result = await paymentGateway.charge(orderId, amount);
      
      span.setAttribute('payment.transaction_id', result.transactionId);
      span.setAttribute('payment.status', 'success');
      span.setStatus({ code: SpanStatusCode.OK });
      
      return result;
    } catch (error) {
      // บันทึก exception พร้อม stack trace
      span.recordException(error as Error, {
        // Timestamp ที่ error เกิด
        [Date.now()]: true,
      });
      
      // เพิ่ม attributes เกี่ยวกับ error
      span.setAttribute('payment.status', 'failed');
      span.setAttribute('error.type', (error as Error).constructor.name);
      span.setAttribute('error.message', (error as Error).message);
      
      // ถ้าเป็น payment error เฉพาะ
      if (error instanceof PaymentGatewayError) {
        span.setAttribute('payment.error_code', error.code);
        span.setAttribute('payment.gateway_response', JSON.stringify(error.response));
      }
      
      span.setStatus({
        code: SpanStatusCode.ERROR,
        message: `Payment failed: ${(error as Error).message}`,
      });
      
      throw error;
    } finally {
      span.end();
    }
  });
}
```

### Span Events: บันทึก Timeline

```typescript
// ------------------- Span Events -------------------

async function processLargeDataset(datasetId: string): Promise<ProcessResult> {
  return tracer.startActiveSpan('processLargeDataset', async (span) => {
    span.setAttribute('dataset.id', datasetId);
    
    // Event: เริ่ม download
    span.addEvent('download.start', {
      'dataset.source': 's3://my-bucket/data',
    });
    
    const data = await downloadDataset(datasetId);
    
    // Event: download เสร็จ
    span.addEvent('download.complete', {
      'dataset.rows': data.length,
      'dataset.size_bytes': JSON.stringify(data).length,
    });
    
    // Event: เริ่ม processing
    span.addEvent('processing.start');
    let processedRows = 0;
    
    for (const batch of chunkArray(data, 1000)) {
      await processBatch(batch);
      processedRows += batch.length;
      
      // Event สำหรับแต่ละ batch
      span.addEvent('batch.processed', {
        'batch.rows_processed': processedRows,
        'batch.total_rows': data.length,
        'batch.progress_percent': Math.round((processedRows / data.length) * 100),
      });
    }
    
    // Event: เสร็จสิ้น
    span.addEvent('processing.complete', {
      'result.total_processed': processedRows,
    });
    
    span.setAttribute('result.rows_processed', processedRows);
    span.setStatus({ code: SpanStatusCode.OK });
    
    return { processedRows };
  });
}
```

---

## Context Propagation: ส่ง Trace ข้าม Services

### Propagation ใน HTTP Requests

```typescript
// ------------------- Outbound HTTP with Context Propagation -------------------
// src/http-client.ts

import { context, propagation, trace } from '@opentelemetry/api';
import axios from 'axios';

// Axios interceptor สำหรับ inject trace headers อัตโนมัติ
axios.interceptors.request.use((config) => {
  const headers: Record<string, string> = {};
  
  // Inject current context เป็น HTTP headers
  propagation.inject(context.active(), headers);
  
  config.headers = { ...config.headers, ...headers };
  
  return config;
});

// ตัวอย่างการใช้งาน
async function callUserService(userId: string): Promise<User> {
  // Context propagation ทำงานอัตโนมัติผ่าน Axios interceptor
  const response = await axios.get(`http://user-service:3001/users/${userId}`);
  return response.data;
}

// ------------------- Inbound Context Extraction -------------------
// Express middleware สำหรับ extract trace context จาก incoming requests

import { Request, Response, NextFunction } from 'express';
import { propagation, context, trace } from '@opentelemetry/api';

export function tracingMiddleware(req: Request, res: Response, next: NextFunction): void {
  // Extract context จาก incoming headers
  const extractedContext = propagation.extract(context.active(), req.headers);
  
  // Run request ภายใน extracted context
  context.with(extractedContext, () => {
    // เพิ่ม trace ID ใน response header (สำหรับ debugging)
    const currentSpan = trace.getActiveSpan();
    if (currentSpan) {
      const traceId = currentSpan.spanContext().traceId;
      res.setHeader('X-Trace-Id', traceId);
      
      // เพิ่ม trace ID ใน request object (ใช้ใน logs)
      (req as any).traceId = traceId;
    }
    
    next();
  });
}

// ------------------- Message Queue Context Propagation -------------------
// RabbitMQ / Kafka trace propagation

import amqp from 'amqplib';

// Producer: inject context ใน message headers
async function publishMessage(queue: string, message: object): Promise<void> {
  const channel = await connection.createChannel();
  const headers: Record<string, string> = {};
  
  // Inject trace context เป็น headers
  propagation.inject(context.active(), headers);
  
  channel.sendToQueue(
    queue,
    Buffer.from(JSON.stringify(message)),
    { headers }
  );
}

// Consumer: extract context จาก message headers
async function consumeMessages(queue: string): Promise<void> {
  const channel = await connection.createChannel();
  
  channel.consume(queue, (msg) => {
    if (!msg) return;
    
    // Extract trace context จาก message headers
    const headers = msg.properties.headers || {};
    const extractedContext = propagation.extract(context.active(), headers);
    
    // Process message ภายใน extracted context
    context.with(extractedContext, async () => {
      const tracer = trace.getTracer('message-consumer');
      
      await tracer.startActiveSpan(`consume.${queue}`, {
        kind: SpanKind.CONSUMER,
      }, async (span) => {
        try {
          const message = JSON.parse(msg.content.toString());
          await processMessage(message);
          channel.ack(msg);
          span.setStatus({ code: SpanStatusCode.OK });
        } catch (error) {
          channel.nack(msg);
          span.recordException(error as Error);
          span.setStatus({ code: SpanStatusCode.ERROR });
        } finally {
          span.end();
        }
      });
    });
  });
}
```

---

## Jaeger: ติดตั้งและใช้งาน

### ติดตั้ง Jaeger ด้วย Docker

```yaml
# docker-compose.jaeger.yml
version: '3.8'

services:
  # Jaeger All-in-One (development)
  jaeger:
    image: jaegertracing/all-in-one:1.51
    container_name: jaeger
    ports:
      - "5775:5775/udp"   # Zipkin Thrift compact protocol
      - "6831:6831/udp"   # Thrift compact protocol
      - "6832:6832/udp"   # Thrift binary protocol
      - "5778:5778"       # Configuration
      - "16686:16686"     # Jaeger UI
      - "14268:14268"     # Jaeger Thrift HTTP
      - "14250:14250"     # gRPC
      - "4317:4317"       # OTLP gRPC
      - "4318:4318"       # OTLP HTTP
    environment:
      - COLLECTOR_OTLP_ENABLED=true
      - COLLECTOR_ZIPKIN_HOST_PORT=:9411
      - SPAN_STORAGE_TYPE=badger  # ใช้ Badger สำหรับ development
    volumes:
      - jaeger-data:/badger
    restart: unless-stopped
    
  # Jaeger with Elasticsearch (production)
  jaeger-production:
    image: jaegertracing/all-in-one:1.51
    container_name: jaeger-prod
    environment:
      - SPAN_STORAGE_TYPE=elasticsearch
      - ES_SERVER_URLS=http://elasticsearch:9200
      - ES_INDEX_PREFIX=jaeger
      - ES_NUM_SHARDS=5
      - ES_NUM_REPLICAS=1
      - COLLECTOR_OTLP_ENABLED=true
      - SAMPLING_STRATEGIES_FILE=/etc/jaeger/sampling.json
    volumes:
      - ./jaeger/sampling.json:/etc/jaeger/sampling.json
    depends_on:
      - elasticsearch

volumes:
  jaeger-data:
```

### Sampling Configuration

```json
// jaeger/sampling.json - Sampling strategies
{
  "service_strategies": [
    {
      "service": "api-gateway",
      "type": "probabilistic",
      "param": 0.1,
      "operation_strategies": [
        {
          "operation": "GET /health",
          "type": "probabilistic",
          "param": 0.0
        },
        {
          "operation": "POST /orders",
          "type": "probabilistic",
          "param": 1.0
        }
      ]
    },
    {
      "service": "payment-service",
      "type": "ratelimiting",
      "param": 100
    }
  ],
  "default_strategy": {
    "type": "probabilistic",
    "param": 0.05
  }
}
```

### Jaeger UI: การใช้งาน

```
Jaeger UI ที่ http://localhost:16686

1. Search Traces:
   - Service: เลือก service ที่ต้องการ
   - Operation: filter ตาม operation name
   - Tags: filter ด้วย key=value เช่น order.user_id=123
   - Lookback: ช่วงเวลา เช่น Last 1 hour
   - Min/Max Duration: filter ตาม latency
   
2. Trace Timeline View:
   ├── แต่ละ row คือ 1 Span
   ├── Width แสดง relative duration
   ├── สีแดงคือ error span
   └── Click span เพื่อดู attributes, events, logs

3. Compare Traces:
   - เลือก 2 traces เพื่อเปรียบเทียบ
   - เห็น diff ของ span ที่เปลี่ยนไป

4. System Architecture:
   - แสดง service dependency graph
   - เห็น call frequency ระหว่าง services
```

---

## Zipkin: Alternative Solution

### ติดตั้ง Zipkin

```yaml
# docker-compose.zipkin.yml
version: '3.8'

services:
  zipkin:
    image: openzipkin/zipkin:latest
    container_name: zipkin
    ports:
      - "9411:9411"    # Zipkin UI + API
    environment:
      - STORAGE_TYPE=elasticsearch
      - ES_HOSTS=http://elasticsearch:9200
      - ES_INDEX=zipkin
    restart: unless-stopped
```

### Zipkin Exporter

```typescript
import { ZipkinExporter } from '@opentelemetry/exporter-zipkin';

const zipkinExporter = new ZipkinExporter({
  url: 'http://localhost:9411/api/v2/spans',
  headers: {
    'Content-Type': 'application/json',
  },
});
```

---

## Correlation IDs: เชื่อม Traces กับ Logs

### เพิ่ม Trace ID ใน Winston Logs

```typescript
// src/logger.ts
import winston from 'winston';
import { trace } from '@opentelemetry/api';

// Custom format เพิ่ม trace context
const traceContextFormat = winston.format((info) => {
  const currentSpan = trace.getActiveSpan();
  if (currentSpan) {
    const spanContext = currentSpan.spanContext();
    info.traceId = spanContext.traceId;
    info.spanId = spanContext.spanId;
    info.traceFlags = spanContext.traceFlags;
  }
  return info;
});

export const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    traceContextFormat(),
    winston.format.timestamp(),
    winston.format.json()
  ),
  transports: [
    new winston.transports.Console(),
    new winston.transports.File({
      filename: '/var/log/app/application.log',
    }),
  ],
});

// การใช้งาน
// Log output จะมี traceId เสมอ:
// {"timestamp":"2024-01-15T10:30:00Z","level":"info","message":"Order created",
//  "traceId":"abc123def456","spanId":"span001","orderId":"order-789"}
```

---

## Sampling Strategies

### 1. Always Sample (Development)

```typescript
import { AlwaysOnSampler } from '@opentelemetry/sdk-trace-base';

const sdk = new NodeSDK({
  sampler: new AlwaysOnSampler(),
  // ... other config
});
```

### 2. Probability Sampling (Production)

```typescript
import { TraceIdRatioBased } from '@opentelemetry/sdk-trace-base';

// Sample 10% of traces
const sdk = new NodeSDK({
  sampler: new TraceIdRatioBased(0.1),
  // ... other config
});
```

### 3. Parent-Based Sampling (Recommended)

```typescript
import { 
  ParentBasedSampler, 
  TraceIdRatioBased,
  AlwaysOnSampler,
  AlwaysOffSampler,
} from '@opentelemetry/sdk-trace-base';

// ถ้า parent sampled → sample ด้วย
// ถ้าไม่มี parent → sample 10%
const sdk = new NodeSDK({
  sampler: new ParentBasedSampler({
    root: new TraceIdRatioBased(0.1),
    remoteParentSampled: new AlwaysOnSampler(),
    remoteParentNotSampled: new AlwaysOffSampler(),
    localParentSampled: new AlwaysOnSampler(),
    localParentNotSampled: new AlwaysOffSampler(),
  }),
});
```

### 4. Custom Sampler

```typescript
import { Sampler, SamplingResult, SamplingDecision, Attributes } from '@opentelemetry/api';

class SmartSampler implements Sampler {
  shouldSample(
    context: Context,
    traceId: string,
    spanName: string,
    spanKind: SpanKind,
    attributes: Attributes
  ): SamplingResult {
    // ตาม sample ทุก error
    if (attributes['error'] === true) {
      return { decision: SamplingDecision.RECORD_AND_SAMPLED };
    }
    
    // ตาม sample ทุก slow request (>1s)
    const duration = attributes['http.response_time_ms'] as number;
    if (duration && duration > 1000) {
      return { decision: SamplingDecision.RECORD_AND_SAMPLED };
    }
    
    // Payment operations: sample ทุกครั้ง
    if (spanName.includes('payment')) {
      return { decision: SamplingDecision.RECORD_AND_SAMPLED };
    }
    
    // Health checks: ไม่ sample
    if (spanName === 'GET /health') {
      return { decision: SamplingDecision.NOT_RECORD };
    }
    
    // Default: sample 5%
    const hash = parseInt(traceId.substring(0, 8), 16);
    if (hash % 100 < 5) {
      return { decision: SamplingDecision.RECORD_AND_SAMPLED };
    }
    
    return { decision: SamplingDecision.NOT_RECORD };
  }
  
  toString(): string {
    return 'SmartSampler';
  }
}
```

---

## Full TypeScript Setup: Express + PostgreSQL + Redis

### Complete Application

```typescript
// src/index.ts - Entry point (ต้อง import tracing ก่อน)
import './tracing/tracer'; // MUST be first import!
import { initTracing } from './tracing/tracer';

// Initialize tracing ก่อน application code
initTracing();

import express from 'express';
import { tracingMiddleware } from './tracing/middleware';
import { orderRouter } from './routes/orders';
import { logger } from './logger';

const app = express();
const PORT = process.env.PORT || 3000;

// Middleware
app.use(express.json());
app.use(tracingMiddleware); // เพิ่ม trace context ใน request

// Routes
app.use('/api/orders', orderRouter);

// Health check (ไม่ trace)
app.get('/health', (req, res) => {
  res.json({ status: 'ok' });
});

app.listen(PORT, () => {
  logger.info(`Server started on port ${PORT}`);
});

// ------------------- Database Setup -------------------
// src/database/postgres.ts

import { Pool } from 'pg';
import { trace, SpanKind, SpanStatusCode } from '@opentelemetry/api';

export const pool = new Pool({
  host: process.env.PG_HOST || 'localhost',
  port: parseInt(process.env.PG_PORT || '5432'),
  database: process.env.PG_DATABASE || 'myapp',
  user: process.env.PG_USER || 'postgres',
  password: process.env.PG_PASSWORD || 'secret',
  max: 20,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});

// Wrapped query function พร้อม tracing
export async function tracedQuery<T extends object>(
  sql: string,
  params: unknown[] = [],
  spanName?: string
): Promise<T[]> {
  const tracer = trace.getTracer('postgres');
  
  return tracer.startActiveSpan(spanName || 'postgres.query', {
    kind: SpanKind.CLIENT,
    attributes: {
      'db.system': 'postgresql',
      'db.statement': sql.length > 500 ? sql.substring(0, 497) + '...' : sql,
      'db.operation': sql.trim().split(' ')[0].toUpperCase(),
      'db.parameters_count': params.length,
    },
  }, async (span) => {
    const startTime = Date.now();
    
    try {
      const result = await pool.query(sql, params);
      
      span.setAttribute('db.rows_affected', result.rowCount || 0);
      span.setAttribute('db.rows_returned', result.rows.length);
      span.setAttribute('db.duration_ms', Date.now() - startTime);
      span.setStatus({ code: SpanStatusCode.OK });
      
      return result.rows;
    } catch (error) {
      span.setAttribute('db.duration_ms', Date.now() - startTime);
      span.setStatus({
        code: SpanStatusCode.ERROR,
        message: (error as Error).message,
      });
      span.recordException(error as Error);
      throw error;
    } finally {
      span.end();
    }
  });
}

// ------------------- Redis Setup -------------------
// src/database/redis.ts

import Redis from 'ioredis';
import { trace, SpanKind, SpanStatusCode } from '@opentelemetry/api';

export const redis = new Redis({
  host: process.env.REDIS_HOST || 'localhost',
  port: parseInt(process.env.REDIS_PORT || '6379'),
  password: process.env.REDIS_PASSWORD,
  db: parseInt(process.env.REDIS_DB || '0'),
  maxRetriesPerRequest: 3,
  enableReadyCheck: true,
  lazyConnect: true,
});

redis.on('error', (error) => {
  console.error('[Redis] Connection error:', error);
});

// Cache helper พร้อม tracing
export class TracedCache {
  private tracer = trace.getTracer('redis');
  
  async get<T>(key: string): Promise<T | null> {
    return this.tracer.startActiveSpan('redis.get', {
      kind: SpanKind.CLIENT,
      attributes: {
        'db.system': 'redis',
        'db.operation': 'GET',
        'cache.key': key,
      },
    }, async (span) => {
      try {
        const value = await redis.get(key);
        const hit = value !== null;
        
        span.setAttribute('cache.hit', hit);
        span.setStatus({ code: SpanStatusCode.OK });
        
        return hit ? JSON.parse(value!) : null;
      } catch (error) {
        span.recordException(error as Error);
        span.setStatus({ code: SpanStatusCode.ERROR });
        return null; // Cache failure ไม่ควรหยุดระบบ
      } finally {
        span.end();
      }
    });
  }
  
  async set(key: string, value: unknown, ttlSeconds?: number): Promise<void> {
    return this.tracer.startActiveSpan('redis.set', {
      kind: SpanKind.CLIENT,
      attributes: {
        'db.system': 'redis',
        'db.operation': ttlSeconds ? 'SETEX' : 'SET',
        'cache.key': key,
        'cache.ttl': ttlSeconds || 0,
      },
    }, async (span) => {
      try {
        if (ttlSeconds) {
          await redis.setex(key, ttlSeconds, JSON.stringify(value));
        } else {
          await redis.set(key, JSON.stringify(value));
        }
        span.setStatus({ code: SpanStatusCode.OK });
      } catch (error) {
        span.recordException(error as Error);
        span.setStatus({ code: SpanStatusCode.ERROR });
      } finally {
        span.end();
      }
    });
  }
  
  async del(key: string): Promise<void> {
    return this.tracer.startActiveSpan('redis.del', {
      kind: SpanKind.CLIENT,
      attributes: {
        'db.system': 'redis',
        'db.operation': 'DEL',
        'cache.key': key,
      },
    }, async (span) => {
      try {
        await redis.del(key);
        span.setStatus({ code: SpanStatusCode.OK });
      } finally {
        span.end();
      }
    });
  }
}

export const cache = new TracedCache();
```

### Docker Compose สำหรับ Full Stack

```yaml
# docker-compose.yml
version: '3.8'

services:
  # Application
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
      - SERVICE_NAME=order-service
      - SERVICE_VERSION=1.0.0
      - JAEGER_ENDPOINT=http://jaeger:4318/v1/traces
      - PG_HOST=postgres
      - REDIS_HOST=redis
    depends_on:
      - postgres
      - redis
      - jaeger
      
  # PostgreSQL
  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: secret
    ports:
      - "5432:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data
      
  # Redis
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    command: redis-server --appendonly yes
    volumes:
      - redis-data:/data
      
  # Jaeger
  jaeger:
    image: jaegertracing/all-in-one:1.51
    ports:
      - "16686:16686"  # UI
      - "4317:4317"    # OTLP gRPC
      - "4318:4318"    # OTLP HTTP
    environment:
      - COLLECTOR_OTLP_ENABLED=true
    volumes:
      - jaeger-data:/badger

volumes:
  postgres-data:
  redis-data:
  jaeger-data:
```

---

## Performance Impact และ Best Practices

### Performance Overhead

```typescript
// Benchmark: Tracing overhead

// Without tracing: ~0.1ms per request
// With tracing (AlwaysOn): ~1-2ms per request
// With tracing (1% sampling): ~0.01-0.02ms per request (negligible)

// Recommendations:
// Development: AlwaysOn sampling
// Staging: 10-20% sampling
// Production: 1-5% sampling + error sampling

// ผลกระทบต่อ throughput:
// 10,000 RPS × 1% sampling = 100 traces/sec stored
// ไม่มีผลกระทบต่อ performance หลักของระบบ
```

### Best Practices

```typescript
// 1. ตั้งชื่อ Span อย่างสม่ำเสมอ
// ดี: "orders.create", "inventory.check", "payment.process"  
// ไม่ดี: "function1", "doStuff", "handler"

// 2. ใช้ Semantic Conventions
import { SemanticAttributes } from '@opentelemetry/semantic-conventions';

span.setAttribute(SemanticAttributes.DB_SYSTEM, 'postgresql');
span.setAttribute(SemanticAttributes.DB_NAME, 'myapp');
span.setAttribute(SemanticAttributes.DB_STATEMENT, sql);
span.setAttribute(SemanticAttributes.HTTP_METHOD, 'GET');
span.setAttribute(SemanticAttributes.HTTP_URL, url);
span.setAttribute(SemanticAttributes.HTTP_STATUS_CODE, 200);

// 3. ไม่บันทึก PII ใน attributes
// ไม่ดี: span.setAttribute('user.password', password);
// ดี: span.setAttribute('user.id', userId);

// 4. จำกัด attribute value size
const statement = sql.length > 1000 ? sql.substring(0, 997) + '...' : sql;

// 5. ใช้ Batch Processor เสมอ (ไม่ใช้ Simple)
// BatchSpanProcessor: buffer และส่ง batch → performance ดีกว่า
// SimpleSpanProcessor: ส่งทันที → ใช้เฉพาะ debug

// 6. Handle errors อย่างถูกต้อง
span.setStatus({ code: SpanStatusCode.ERROR });
span.recordException(error);
// ทั้งสองอย่างนี้จำเป็น!
```

---

## สรุป: Distributed Tracing Checklist

```markdown
## Pre-Deployment Checklist

### Setup
- [ ] OTel SDK initialized ก่อน application code
- [ ] Service name และ version ตั้งค่าแล้ว
- [ ] Exporter configured (Jaeger/Zipkin/OTLP)
- [ ] Sampling strategy เหมาะกับ environment

### Auto-Instrumentation
- [ ] HTTP/Express instrumented
- [ ] PostgreSQL instrumented  
- [ ] Redis instrumented
- [ ] Outbound HTTP clients instrumented

### Manual Instrumentation
- [ ] Business logic สำคัญมี spans
- [ ] Errors บันทึกด้วย recordException()
- [ ] Status set อย่างถูกต้อง (OK/ERROR)
- [ ] Meaningful attributes เพิ่มแล้ว

### Context Propagation
- [ ] W3C TraceContext headers ส่งใน outbound requests
- [ ] Message queue headers มี trace context
- [ ] Trace ID เพิ่มใน application logs

### Operations
- [ ] Jaeger/Zipkin accessible สำหรับ team
- [ ] Sampling rate configured ตาม load
- [ ] Alerting สำหรับ high latency traces
- [ ] Regular review ของ slow traces
```

---

*Part 56 เสร็จสมบูรณ์ - ต่อไป Part 57: Metrics ด้วย Prometheus + Grafana*

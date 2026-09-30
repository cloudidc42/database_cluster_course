# Part 80: Real-time Features ด้วย PostgreSQL LISTEN/NOTIFY

## บทนำ: Real-time ใน PostgreSQL

หลายคนคิดว่า PostgreSQL เป็นแค่ database ทั่วไป แต่จริงๆ แล้วมันมี built-in pub/sub mechanism ที่ชื่อว่า LISTEN/NOTIFY ซึ่งสามารถใช้สร้าง real-time features ได้โดยไม่ต้องพึ่ง external services เช่น Redis Pub/Sub หรือ Kafka

```
LISTEN/NOTIFY Architecture:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Application A          PostgreSQL          Application B
(Publisher)              Server             (Subscriber)
    │                      │                     │
    │  NOTIFY channel      │                     │
    │  'payload'  ──────── │                     │
    │                      │   notification      │
    │                      │ ─────────────────── │
    │                      │                     │ LISTEN channel
    │                      │                     │ (waiting for notifications)
    │                      │                     │
                           │   
         Trigger           │ pg_notify()
         (INSERT/          │  ←── Trigger function sends
          UPDATE/          │       notification automatically
          DELETE)          │
```

---

## LISTEN/NOTIFY Syntax และ Commands

### Basic Commands

```sql
-- ส่ง notification ไปยัง channel
NOTIFY channel_name;
NOTIFY channel_name, 'optional payload string';

-- Subscribe to channel (รับ notifications)
LISTEN channel_name;

-- Unsubscribe
UNLISTEN channel_name;
UNLISTEN *;  -- unsubscribe from all channels

-- pg_notify() function (ใช้ใน trigger/function)
SELECT pg_notify('channel_name', 'payload message');

-- ดู active listeners
SELECT pid, query_start, query, wait_event
FROM pg_stat_activity
WHERE wait_event = 'ClientRead'  -- กำลัง listen อยู่;
```

### Payload Format และ Limitations

```sql
-- Payload เป็น string (สูงสุด 8000 bytes!)
-- ถ้าต้องการส่ง JSON ใช้:

-- GOOD: ส่ง ID แล้วให้ subscriber ไป query เอง
SELECT pg_notify('user_updated', json_build_object(
    'id', user_id,
    'action', 'update',
    'timestamp', extract(epoch from NOW())
)::text);

-- BAD: ส่ง full object (อาจ exceed 8000 bytes!)
-- ถ้า payload > 8000 bytes: notification จะ fail ทันที!
SELECT pg_notify('user_updated', row_to_json(users.*)::text);  -- DANGEROUS

-- ตรวจสอบ payload size
SELECT length(json_build_object('id', gen_random_uuid(), 'data', repeat('x', 7000))::text);
```

---

## Triggers + NOTIFY

### สร้าง Trigger Function ที่ส่ง NOTIFY

```sql
-- สร้าง function ที่จะถูกเรียกโดย trigger
CREATE OR REPLACE FUNCTION notify_table_change()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
DECLARE
    payload JSON;
    channel_name TEXT;
BEGIN
    -- กำหนด channel name จาก table name
    channel_name := 'changes_' || TG_TABLE_NAME;
    
    -- สร้าง payload
    -- ส่งเฉพาะ minimal info เพื่อไม่ให้ exceed 8000 bytes
    IF TG_OP = 'INSERT' OR TG_OP = 'UPDATE' THEN
        payload := json_build_object(
            'operation', TG_OP,
            'table', TG_TABLE_NAME,
            'id', NEW.id,
            'timestamp', extract(epoch from NOW())::bigint
        );
    ELSIF TG_OP = 'DELETE' THEN
        payload := json_build_object(
            'operation', TG_OP,
            'table', TG_TABLE_NAME,
            'id', OLD.id,
            'timestamp', extract(epoch from NOW())::bigint
        );
    END IF;
    
    -- ส่ง notification
    PERFORM pg_notify(channel_name, payload::text);
    
    -- ส่ง notification ไปยัง generic channel ด้วย
    PERFORM pg_notify('db_changes', payload::text);
    
    RETURN COALESCE(NEW, OLD);
END;
$$;
```

### Trigger สำหรับ Orders Table

```sql
-- สร้าง orders table
CREATE TABLE orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id UUID NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending'
        CHECK (status IN ('pending', 'confirmed', 'processing', 'shipped', 'delivered', 'cancelled')),
    total_amount DECIMAL(10,2) NOT NULL,
    items JSONB NOT NULL,
    shipping_address JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Trigger สำหรับ notification เมื่อมี status change
CREATE OR REPLACE FUNCTION notify_order_change()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
DECLARE
    payload TEXT;
    notification_data JSONB;
BEGIN
    -- สร้าง notification ที่มี relevant info
    notification_data := jsonb_build_object(
        'event', CASE
            WHEN TG_OP = 'INSERT' THEN 'order.created'
            WHEN TG_OP = 'UPDATE' AND OLD.status != NEW.status THEN 'order.status_changed'
            WHEN TG_OP = 'UPDATE' THEN 'order.updated'
            WHEN TG_OP = 'DELETE' THEN 'order.deleted'
        END,
        'orderId', CASE TG_OP WHEN 'DELETE' THEN OLD.id ELSE NEW.id END,
        'customerId', CASE TG_OP WHEN 'DELETE' THEN OLD.customer_id ELSE NEW.customer_id END,
        'status', CASE TG_OP WHEN 'DELETE' THEN OLD.status ELSE NEW.status END,
        'previousStatus', CASE WHEN TG_OP = 'UPDATE' THEN OLD.status ELSE NULL END,
        'totalAmount', CASE TG_OP WHEN 'DELETE' THEN OLD.total_amount ELSE NEW.total_amount END,
        'timestamp', extract(epoch from NOW())::bigint
    );
    
    payload := notification_data::text;
    
    -- ตรวจสอบ payload size ก่อนส่ง
    IF length(payload) > 7500 THEN
        -- Truncate แล้วส่ง ID เท่านั้น
        notification_data := jsonb_build_object(
            'event', notification_data->>'event',
            'orderId', notification_data->>'orderId',
            'error', 'payload_truncated'
        );
        payload := notification_data::text;
    END IF;
    
    -- ส่งไปยัง multiple channels
    PERFORM pg_notify('orders', payload);
    
    -- Customer-specific channel (สำหรับ real-time tracking)
    IF TG_OP != 'DELETE' THEN
        PERFORM pg_notify(
            'customer_' || NEW.customer_id::text,
            payload
        );
    END IF;
    
    -- Status-specific channel (สำหรับ shipping, fulfillment teams)
    IF TG_OP = 'UPDATE' AND OLD.status != NEW.status THEN
        PERFORM pg_notify(
            'order_status_' || NEW.status,
            payload
        );
    END IF;
    
    RETURN COALESCE(NEW, OLD);
END;
$$;

-- สร้าง trigger
CREATE TRIGGER orders_change_notify
    AFTER INSERT OR UPDATE OR DELETE ON orders
    FOR EACH ROW EXECUTE FUNCTION notify_order_change();

-- ทดสอบ trigger
INSERT INTO orders (customer_id, total_amount, items)
VALUES ('550e8400-e29b-41d4-a716-446655440000', 1500.00, '[{"product": "laptop", "qty": 1}]');
-- → notification ส่งไปยัง 'orders' channel

UPDATE orders SET status = 'confirmed' WHERE id = '<order-id>';
-- → notification ส่งไปยัง 'orders', 'customer_<id>', 'order_status_confirmed'
```

### Trigger สำหรับ Cache Invalidation

```sql
-- Trigger สำหรับ invalidate Redis cache
CREATE OR REPLACE FUNCTION notify_cache_invalidation()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
DECLARE
    cache_keys TEXT[];
    key TEXT;
    payload TEXT;
BEGIN
    -- กำหนด cache keys ที่ต้อง invalidate
    CASE TG_TABLE_NAME
        WHEN 'products' THEN
            cache_keys := ARRAY[
                'product:' || COALESCE(NEW.id, OLD.id)::text,
                'products:list',
                'products:category:' || COALESCE(NEW.category, OLD.category),
                'products:featured'
            ];
        WHEN 'users' THEN
            cache_keys := ARRAY[
                'user:' || COALESCE(NEW.id, OLD.id)::text,
                'user:email:' || COALESCE(NEW.email, OLD.email)
            ];
        WHEN 'categories' THEN
            cache_keys := ARRAY[
                'categories:all',
                'category:' || COALESCE(NEW.id, OLD.id)::text
            ];
        ELSE
            cache_keys := ARRAY[]::TEXT[];
    END CASE;
    
    -- ส่ง notification สำหรับแต่ละ cache key
    FOREACH key IN ARRAY cache_keys LOOP
        payload := json_build_object(
            'action', 'invalidate',
            'key', key,
            'table', TG_TABLE_NAME,
            'operation', TG_OP
        )::text;
        
        PERFORM pg_notify('cache_invalidation', payload);
    END LOOP;
    
    RETURN COALESCE(NEW, OLD);
END;
$$;

CREATE TRIGGER products_cache_invalidate
    AFTER INSERT OR UPDATE OR DELETE ON products
    FOR EACH ROW EXECUTE FUNCTION notify_cache_invalidation();

CREATE TRIGGER users_cache_invalidate
    AFTER INSERT OR UPDATE OR DELETE ON users
    FOR EACH ROW EXECUTE FUNCTION notify_cache_invalidation();
```

---

## Node.js LISTEN Implementation

### Dedicated Connection สำหรับ LISTEN

```typescript
// pg-listener.ts
// IMPORTANT: LISTEN ต้องใช้ dedicated connection (ไม่ share กับ pool!)
// เพราะ connection ที่ LISTEN จะ block รอ notification

import { Client } from 'pg';
import { EventEmitter } from 'events';

interface NotificationPayload {
  channel: string;
  payload: any;
  processId: number;
}

class PostgresListener extends EventEmitter {
  private client: Client;
  private channels: Set<string> = new Set();
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 10;
  private reconnecting = false;
  
  constructor(private connectionString: string) {
    super();
    this.client = new Client({ connectionString });
  }
  
  async connect(): Promise<void> {
    await this.client.connect();
    
    // Handle disconnection
    this.client.on('error', async (err) => {
      console.error('PostgreSQL connection error:', err);
      await this.handleReconnect();
    });
    
    this.client.on('end', async () => {
      console.log('PostgreSQL connection ended');
      await this.handleReconnect();
    });
    
    // Handle notifications
    this.client.on('notification', (msg) => {
      let parsedPayload: any = msg.payload;
      
      try {
        parsedPayload = JSON.parse(msg.payload || '{}');
      } catch {
        // payload ไม่ใช่ JSON, ใช้ raw string
      }
      
      const notification: NotificationPayload = {
        channel: msg.channel,
        payload: parsedPayload,
        processId: msg.processId,
      };
      
      // Emit เป็น specific channel event
      this.emit(msg.channel, notification);
      
      // Emit เป็น generic 'notification' event
      this.emit('notification', notification);
    });
    
    console.log('PostgreSQL listener connected');
    this.reconnectAttempts = 0;
  }
  
  async listen(channel: string): Promise<void> {
    this.channels.add(channel);
    await this.client.query(`LISTEN ${channel}`);
    console.log(`Listening to channel: ${channel}`);
  }
  
  async unlisten(channel: string): Promise<void> {
    this.channels.delete(channel);
    await this.client.query(`UNLISTEN ${channel}`);
    console.log(`Stopped listening to channel: ${channel}`);
  }
  
  private async handleReconnect(): Promise<void> {
    if (this.reconnecting) return;
    this.reconnecting = true;
    
    while (this.reconnectAttempts < this.maxReconnectAttempts) {
      this.reconnectAttempts++;
      
      const delay = Math.min(1000 * Math.pow(2, this.reconnectAttempts), 30000);
      console.log(`Reconnect attempt ${this.reconnectAttempts}/${this.maxReconnectAttempts} in ${delay}ms`);
      
      await new Promise(resolve => setTimeout(resolve, delay));
      
      try {
        this.client = new Client({ connectionString: this.connectionString });
        await this.connect();
        
        // Re-subscribe to all channels
        for (const channel of this.channels) {
          await this.client.query(`LISTEN ${channel}`);
        }
        
        this.reconnecting = false;
        this.emit('reconnected');
        console.log('Reconnected successfully!');
        return;
      } catch (error) {
        console.error(`Reconnect attempt failed:`, error);
      }
    }
    
    this.reconnecting = false;
    this.emit('error', new Error('Max reconnect attempts reached'));
  }
  
  async disconnect(): Promise<void> {
    this.maxReconnectAttempts = 0;  // Prevent reconnection
    await this.client.end();
  }
}
```

### Full Application: Real-time Order Updates

```typescript
// realtime-orders.ts
import { PostgresListener } from './pg-listener';
import { createClient } from 'redis';
import { WebSocketServer, WebSocket } from 'ws';
import { Pool } from 'pg';

interface OrderEvent {
  event: string;
  orderId: string;
  customerId: string;
  status: string;
  previousStatus: string | null;
  totalAmount: number;
  timestamp: number;
}

interface WebSocketClient {
  ws: WebSocket;
  customerId: string | null;
  subscribedChannels: Set<string>;
}

class RealtimeOrderSystem {
  private listener: PostgresListener;
  private redis: ReturnType<typeof createClient>;
  private pool: Pool;
  private wss: WebSocketServer;
  private clients: Map<string, WebSocketClient> = new Map();
  
  constructor() {
    const DATABASE_URL = process.env.DATABASE_URL!;
    
    this.listener = new PostgresListener(DATABASE_URL);
    this.pool = new Pool({ connectionString: DATABASE_URL });
    this.redis = createClient({ url: process.env.REDIS_URL });
    this.wss = new WebSocketServer({ port: 8080 });
  }
  
  async start(): Promise<void> {
    // Connect to services
    await this.listener.connect();
    await this.redis.connect();
    
    // Setup LISTEN channels
    await this.listener.listen('orders');
    await this.listener.listen('cache_invalidation');
    
    // Handle order notifications
    this.listener.on('orders', async (notification) => {
      const event = notification.payload as OrderEvent;
      
      console.log(`Order event: ${event.event} for order ${event.orderId}`);
      
      await this.handleOrderEvent(event);
    });
    
    // Handle cache invalidation
    this.listener.on('cache_invalidation', async (notification) => {
      const { action, key } = notification.payload;
      
      if (action === 'invalidate') {
        await this.redis.del(key);
        console.log(`Cache invalidated: ${key}`);
      }
    });
    
    // Handle reconnection
    this.listener.on('reconnected', async () => {
      console.log('Listener reconnected, re-subscribing...');
      // Channels are re-subscribed automatically in handleReconnect()
    });
    
    // Setup WebSocket server
    this.setupWebSocketServer();
    
    console.log('Real-time order system started on ws://localhost:8080');
  }
  
  private async handleOrderEvent(event: OrderEvent): Promise<void> {
    // 1. Update Redis cache
    const orderCacheKey = `order:${event.orderId}`;
    
    if (event.event !== 'order.deleted') {
      // Fetch full order data from DB
      const orderResult = await this.pool.query(
        'SELECT * FROM orders WHERE id = $1',
        [event.orderId]
      );
      
      if (orderResult.rows[0]) {
        await this.redis.setEx(orderCacheKey, 3600, JSON.stringify(orderResult.rows[0]));
      }
    } else {
      await this.redis.del(orderCacheKey);
    }
    
    // 2. Send WebSocket notification to relevant clients
    await this.broadcastToCustomer(event.customerId, {
      type: 'ORDER_UPDATE',
      data: event,
    });
    
    // 3. Send to admin dashboard (broadcast to all admin connections)
    await this.broadcastToAdmins({
      type: 'ORDER_UPDATE',
      data: event,
    });
    
    // 4. Trigger downstream actions based on event
    if (event.event === 'order.status_changed') {
      await this.handleStatusChange(event);
    }
  }
  
  private async handleStatusChange(event: OrderEvent): Promise<void> {
    switch (event.status) {
      case 'confirmed':
        // Trigger inventory reservation
        await this.redis.lPush('tasks:inventory_reserve', JSON.stringify({ orderId: event.orderId }));
        break;
      
      case 'shipped':
        // Trigger email notification
        await this.redis.lPush('tasks:send_email', JSON.stringify({
          type: 'order_shipped',
          orderId: event.orderId,
          customerId: event.customerId,
        }));
        break;
      
      case 'delivered':
        // Trigger review request email after 3 days
        const deliveredAt = new Date();
        deliveredAt.setDate(deliveredAt.getDate() + 3);
        
        await this.redis.zAdd('tasks:scheduled', {
          score: deliveredAt.getTime(),
          value: JSON.stringify({
            type: 'request_review',
            orderId: event.orderId,
            customerId: event.customerId,
          }),
        });
        break;
      
      case 'cancelled':
        // Trigger inventory release
        await this.redis.lPush('tasks:inventory_release', JSON.stringify({ orderId: event.orderId }));
        break;
    }
  }
  
  private setupWebSocketServer(): void {
    this.wss.on('connection', (ws, request) => {
      const clientId = crypto.randomUUID();
      
      const client: WebSocketClient = {
        ws,
        customerId: null,
        subscribedChannels: new Set(),
      };
      
      this.clients.set(clientId, client);
      
      ws.on('message', async (data) => {
        try {
          const message = JSON.parse(data.toString());
          await this.handleClientMessage(clientId, message);
        } catch (error) {
          ws.send(JSON.stringify({ type: 'ERROR', message: 'Invalid message format' }));
        }
      });
      
      ws.on('close', () => {
        this.clients.delete(clientId);
        console.log(`WebSocket client ${clientId} disconnected`);
      });
      
      ws.on('error', (error) => {
        console.error(`WebSocket error for client ${clientId}:`, error);
        this.clients.delete(clientId);
      });
      
      // Send welcome message
      ws.send(JSON.stringify({
        type: 'CONNECTED',
        clientId,
        timestamp: Date.now(),
      }));
    });
  }
  
  private async handleClientMessage(clientId: string, message: any): Promise<void> {
    const client = this.clients.get(clientId);
    if (!client) return;
    
    switch (message.type) {
      case 'AUTHENTICATE':
        // Verify JWT token
        const customerId = await this.verifyToken(message.token);
        if (customerId) {
          client.customerId = customerId;
          client.ws.send(JSON.stringify({ type: 'AUTHENTICATED', customerId }));
          
          // Subscribe to customer-specific PostgreSQL channel
          await this.listener.listen(`customer_${customerId}`);
          this.listener.on(`customer_${customerId}`, (notification) => {
            if (client.ws.readyState === WebSocket.OPEN) {
              client.ws.send(JSON.stringify({
                type: 'ORDER_UPDATE',
                data: notification.payload,
              }));
            }
          });
        } else {
          client.ws.send(JSON.stringify({ type: 'AUTH_FAILED' }));
        }
        break;
      
      case 'SUBSCRIBE_ORDER':
        // ส่ง current order status ทันที
        const order = await this.getOrderFromCache(message.orderId);
        if (order) {
          client.ws.send(JSON.stringify({
            type: 'ORDER_CURRENT_STATE',
            data: order,
          }));
        }
        break;
      
      case 'PING':
        client.ws.send(JSON.stringify({ type: 'PONG', timestamp: Date.now() }));
        break;
    }
  }
  
  private async broadcastToCustomer(customerId: string, message: any): Promise<void> {
    for (const [_, client] of this.clients) {
      if (client.customerId === customerId && client.ws.readyState === WebSocket.OPEN) {
        client.ws.send(JSON.stringify(message));
      }
    }
  }
  
  private async broadcastToAdmins(message: any): Promise<void> {
    // ส่งไปยัง admin connections (implementation specific)
    for (const [_, client] of this.clients) {
      // Check if admin (based on authentication)
      if (client.ws.readyState === WebSocket.OPEN) {
        client.ws.send(JSON.stringify({ ...message, _broadcast: 'admin' }));
      }
    }
  }
  
  private async getOrderFromCache(orderId: string): Promise<any> {
    const cached = await this.redis.get(`order:${orderId}`);
    if (cached) return JSON.parse(cached);
    
    // Cache miss: fetch from DB
    const result = await this.pool.query(
      'SELECT * FROM orders WHERE id = $1',
      [orderId]
    );
    
    if (result.rows[0]) {
      await this.redis.setEx(`order:${orderId}`, 3600, JSON.stringify(result.rows[0]));
      return result.rows[0];
    }
    
    return null;
  }
  
  private async verifyToken(token: string): Promise<string | null> {
    // JWT verification logic here
    // Return customerId if valid, null if invalid
    return null;  // placeholder
  }
}

// Start the system
const system = new RealtimeOrderSystem();
system.start().catch(console.error);
```

---

## Pattern 1: DB Trigger → NOTIFY → App → WebSocket → Browser

```typescript
// websocket-server.ts - Complete implementation
import express from 'express';
import { createServer } from 'http';
import { WebSocketServer, WebSocket } from 'ws';
import { Client, Pool } from 'pg';

const app = express();
const server = createServer(app);
const wss = new WebSocketServer({ server });

const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// Dedicated connection for LISTEN
const listenerClient = new Client({ connectionString: process.env.DATABASE_URL });

// Track WebSocket connections
const connections: Map<string, Set<WebSocket>> = new Map();

async function setupListener() {
  await listenerClient.connect();
  
  // Listen to order updates
  await listenerClient.query('LISTEN orders');
  await listenerClient.query('LISTEN users');
  await listenerClient.query('LISTEN inventory');
  
  listenerClient.on('notification', async (msg) => {
    console.log(`Notification from ${msg.channel}:`, msg.payload);
    
    let event: any;
    try {
      event = JSON.parse(msg.payload || '{}');
    } catch {
      event = { raw: msg.payload };
    }
    
    // Route to appropriate WebSocket clients
    switch (msg.channel) {
      case 'orders':
        await broadcastOrderUpdate(event);
        break;
      
      case 'inventory':
        await broadcastInventoryUpdate(event);
        break;
    }
  });
  
  // Reconnection handling
  listenerClient.on('error', (err) => {
    console.error('Listener error:', err);
    setTimeout(setupListener, 5000);  // Reconnect after 5s
  });
}

async function broadcastOrderUpdate(event: any) {
  // ส่งไปยัง customers ที่ subscribe อยู่
  const customerChannel = `customer:${event.customerId}`;
  const subscribers = connections.get(customerChannel) || new Set();
  
  const message = JSON.stringify({
    type: 'ORDER_UPDATE',
    data: {
      orderId: event.orderId,
      status: event.status,
      previousStatus: event.previousStatus,
      timestamp: event.timestamp,
    },
  });
  
  for (const ws of subscribers) {
    if (ws.readyState === WebSocket.OPEN) {
      ws.send(message);
    }
  }
  
  // ส่งไปยัง admin dashboard
  const adminSubscribers = connections.get('admin') || new Set();
  for (const ws of adminSubscribers) {
    if (ws.readyState === WebSocket.OPEN) {
      ws.send(JSON.stringify({ type: 'ADMIN_ORDER_UPDATE', data: event }));
    }
  }
}

async function broadcastInventoryUpdate(event: any) {
  // ส่งให้ทุก client ที่ subscribe อยู่
  const productChannel = `product:${event.productId}`;
  const subscribers = connections.get(productChannel) || new Set();
  
  const message = JSON.stringify({
    type: 'INVENTORY_UPDATE',
    data: event,
  });
  
  for (const ws of subscribers) {
    if (ws.readyState === WebSocket.OPEN) {
      ws.send(message);
    }
  }
}

// WebSocket connection handling
wss.on('connection', (ws) => {
  let subscribedChannels: Set<string> = new Set();
  
  ws.on('message', async (data) => {
    const msg = JSON.parse(data.toString());
    
    if (msg.type === 'SUBSCRIBE') {
      // Client subscribes to a channel
      const channel = msg.channel;
      subscribedChannels.add(channel);
      
      if (!connections.has(channel)) {
        connections.set(channel, new Set());
      }
      connections.get(channel)!.add(ws);
      
      ws.send(JSON.stringify({ type: 'SUBSCRIBED', channel }));
    }
    
    if (msg.type === 'UNSUBSCRIBE') {
      const channel = msg.channel;
      subscribedChannels.delete(channel);
      connections.get(channel)?.delete(ws);
    }
  });
  
  ws.on('close', () => {
    // Cleanup: remove from all subscribed channels
    for (const channel of subscribedChannels) {
      connections.get(channel)?.delete(ws);
    }
  });
});

// Start
setupListener();
server.listen(3000, () => console.log('Server running on :3000'));
```

---

## Pattern 2: DB Trigger → NOTIFY → App → Invalidate Redis Cache

```typescript
// cache-invalidation-listener.ts
import { Client } from 'pg';
import { createClient } from 'redis';

interface CacheInvalidationMessage {
  action: 'invalidate' | 'refresh';
  key: string;
  table: string;
  operation: string;
}

class CacheInvalidationListener {
  private pgClient: Client;
  private redisClient: ReturnType<typeof createClient>;
  private invalidationQueue: CacheInvalidationMessage[] = [];
  private processing = false;
  
  constructor(
    private pgConnectionString: string,
    private redisConnectionString: string
  ) {
    this.pgClient = new Client({ connectionString: pgConnectionString });
    this.redisClient = createClient({ url: redisConnectionString });
  }
  
  async start(): Promise<void> {
    await this.pgClient.connect();
    await this.redisClient.connect();
    
    await this.pgClient.query('LISTEN cache_invalidation');
    
    this.pgClient.on('notification', async (msg) => {
      if (msg.channel === 'cache_invalidation') {
        try {
          const data = JSON.parse(msg.payload || '{}') as CacheInvalidationMessage;
          
          // Queue สำหรับ batch processing
          this.invalidationQueue.push(data);
          
          if (!this.processing) {
            this.processBatch();
          }
        } catch (error) {
          console.error('Failed to parse cache invalidation message:', error);
        }
      }
    });
    
    // Error handling with reconnection
    this.pgClient.on('error', async () => {
      await this.reconnect();
    });
    
    console.log('Cache invalidation listener started');
  }
  
  private async processBatch(): Promise<void> {
    this.processing = true;
    
    // Process in batches for efficiency
    while (this.invalidationQueue.length > 0) {
      const batch = this.invalidationQueue.splice(0, 100);
      
      // Deduplicate keys
      const uniqueKeys = [...new Set(batch.map(m => m.key))];
      
      // Delete all keys in parallel
      await Promise.allSettled(
        uniqueKeys.map(key => this.redisClient.del(key))
      );
      
      console.log(`Invalidated ${uniqueKeys.length} cache keys: ${uniqueKeys.slice(0, 3).join(', ')}...`);
    }
    
    this.processing = false;
  }
  
  private async reconnect(): Promise<void> {
    console.log('Reconnecting cache invalidation listener...');
    
    await new Promise(resolve => setTimeout(resolve, 2000));
    
    try {
      this.pgClient = new Client({ connectionString: this.pgConnectionString });
      await this.pgClient.connect();
      await this.pgClient.query('LISTEN cache_invalidation');
      console.log('Reconnected!');
    } catch (error) {
      console.error('Reconnection failed:', error);
      await this.reconnect();  // Try again
    }
  }
}
```

---

## Pattern 3: DB Trigger → NOTIFY → App → Send Email/Push

```typescript
// notification-dispatcher.ts
import { Client } from 'pg';

interface OrderStatusChange {
  event: string;
  orderId: string;
  customerId: string;
  status: string;
  previousStatus: string | null;
}

class NotificationDispatcher {
  private pgClient: Client;
  
  constructor(connectionString: string) {
    this.pgClient = new Client({ connectionString });
  }
  
  async start(): Promise<void> {
    await this.pgClient.connect();
    
    // Listen to relevant channels
    await this.pgClient.query('LISTEN order_status_shipped');
    await this.pgClient.query('LISTEN order_status_delivered');
    await this.pgClient.query('LISTEN order_status_cancelled');
    await this.pgClient.query('LISTEN user_registered');
    
    this.pgClient.on('notification', async (msg) => {
      const payload = JSON.parse(msg.payload || '{}');
      
      switch (msg.channel) {
        case 'order_status_shipped':
          await this.sendShippingNotification(payload);
          break;
        
        case 'order_status_delivered':
          await this.sendDeliveryConfirmation(payload);
          break;
        
        case 'order_status_cancelled':
          await this.sendCancellationNotification(payload);
          break;
        
        case 'user_registered':
          await this.sendWelcomeEmail(payload);
          break;
      }
    });
    
    console.log('Notification dispatcher started');
  }
  
  private async sendShippingNotification(data: OrderStatusChange): Promise<void> {
    console.log(`Sending shipping notification for order ${data.orderId}`);
    
    // Email
    await this.sendEmail({
      to: `customer-${data.customerId}@example.com`,
      subject: `Your order #${data.orderId.slice(0, 8)} has been shipped!`,
      body: `Your order is on its way. Track your package...`,
    });
    
    // Push notification
    await this.sendPushNotification({
      userId: data.customerId,
      title: 'Order Shipped! 📦',
      body: `Your order #${data.orderId.slice(0, 8)} is on its way!`,
      data: { orderId: data.orderId, screen: 'order_tracking' },
    });
  }
  
  private async sendDeliveryConfirmation(data: OrderStatusChange): Promise<void> {
    // Send email + schedule review request
    await this.sendEmail({
      to: `customer-${data.customerId}@example.com`,
      subject: 'Your order has been delivered!',
      body: 'Thank you for shopping with us...',
    });
  }
  
  private async sendCancellationNotification(data: OrderStatusChange): Promise<void> {
    await this.sendEmail({
      to: `customer-${data.customerId}@example.com`,
      subject: `Order #${data.orderId.slice(0, 8)} has been cancelled`,
      body: 'Your order has been cancelled. Refund will be processed...',
    });
  }
  
  private async sendWelcomeEmail(data: { userId: string; email: string }): Promise<void> {
    await this.sendEmail({
      to: data.email,
      subject: 'Welcome to our store! 🎉',
      body: 'Thanks for signing up...',
    });
  }
  
  private async sendEmail(options: { to: string; subject: string; body: string }): Promise<void> {
    // Integration with email service (SendGrid, SES, etc.)
    console.log(`Sending email to ${options.to}: ${options.subject}`);
  }
  
  private async sendPushNotification(options: {
    userId: string;
    title: string;
    body: string;
    data?: Record<string, string>;
  }): Promise<void> {
    // Integration with FCM/APNs
    console.log(`Sending push to user ${options.userId}: ${options.title}`);
  }
}
```

---

## Reliability: Notifications Lost on Reconnect

```
ปัญหา: Notifications อาจ Lost เมื่อ Connection ขาด
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Timeline:
T1: App disconnects from PostgreSQL
T2: Database: 5 orders changed status
T3: App reconnects
T4: App ไม่รู้ว่า T2 เกิดขึ้น! (notifications lost)

Solution: Outbox Pattern + CDC
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

DB Trigger → Write to outbox table → NOTIFY → App reads from outbox
                                    ↓
                               App reconnects → Poll outbox for unprocessed events
```

### Outbox Pattern Implementation

```sql
-- Outbox table: เก็บ events ที่ยังไม่ได้ process
CREATE TABLE event_outbox (
    id BIGSERIAL PRIMARY KEY,
    event_type TEXT NOT NULL,
    aggregate_id UUID NOT NULL,
    aggregate_type TEXT NOT NULL,
    payload JSONB NOT NULL,
    published BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    published_at TIMESTAMPTZ
);

CREATE INDEX ON event_outbox (published, created_at) WHERE published = FALSE;
CREATE INDEX ON event_outbox (created_at DESC);

-- Function: write to outbox AND send NOTIFY
CREATE OR REPLACE FUNCTION write_and_notify()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
DECLARE
    event_data JSONB;
    outbox_id BIGINT;
BEGIN
    -- สร้าง event payload
    event_data := jsonb_build_object(
        'operation', TG_OP,
        'table', TG_TABLE_NAME,
        'data', CASE TG_OP
            WHEN 'DELETE' THEN row_to_json(OLD)::jsonb
            ELSE row_to_json(NEW)::jsonb
        END
    );
    
    -- Write to outbox (durable)
    INSERT INTO event_outbox (event_type, aggregate_id, aggregate_type, payload)
    VALUES (
        TG_TABLE_NAME || '.' || lower(TG_OP),
        CASE TG_OP WHEN 'DELETE' THEN OLD.id ELSE NEW.id END,
        TG_TABLE_NAME,
        event_data
    )
    RETURNING id INTO outbox_id;
    
    -- Send NOTIFY with outbox ID
    -- Subscribers can read from outbox using this ID
    PERFORM pg_notify(
        'events',
        json_build_object('outboxId', outbox_id)::text
    );
    
    RETURN COALESCE(NEW, OLD);
END;
$$;

CREATE TRIGGER orders_outbox
    AFTER INSERT OR UPDATE OR DELETE ON orders
    FOR EACH ROW EXECUTE FUNCTION write_and_notify();
```

```typescript
// reliable-listener.ts
import { Client, Pool } from 'pg';

class ReliableListener {
  private listenerClient: Client;
  private pool: Pool;
  private lastProcessedId: bigint = 0n;
  
  constructor(connectionString: string) {
    this.listenerClient = new Client({ connectionString });
    this.pool = new Pool({ connectionString });
  }
  
  async start(): Promise<void> {
    // Load last processed ID from persistence (Redis/file)
    this.lastProcessedId = await this.loadLastProcessedId();
    
    await this.listenerClient.connect();
    await this.listenerClient.query('LISTEN events');
    
    this.listenerClient.on('notification', async (msg) => {
      const { outboxId } = JSON.parse(msg.payload || '{}');
      // Process events up to this outboxId
      await this.processEvents(BigInt(outboxId));
    });
    
    // On connect/reconnect: poll for missed events
    await this.catchUpMissedEvents();
    
    this.listenerClient.on('error', async () => {
      console.log('Listener disconnected, catching up after reconnect...');
      await this.reconnect();
    });
  }
  
  private async processEvents(upToId: bigint): Promise<void> {
    const result = await this.pool.query<{
      id: string;
      event_type: string;
      aggregate_id: string;
      payload: any;
    }>(`
      SELECT id, event_type, aggregate_id, payload
      FROM event_outbox
      WHERE id > $1 AND id <= $2 AND published = FALSE
      ORDER BY id ASC
    `, [this.lastProcessedId.toString(), upToId.toString()]);
    
    for (const event of result.rows) {
      try {
        await this.handleEvent(event);
        
        // Mark as published
        await this.pool.query(
          'UPDATE event_outbox SET published = TRUE, published_at = NOW() WHERE id = $1',
          [event.id]
        );
        
        this.lastProcessedId = BigInt(event.id);
        await this.saveLastProcessedId(this.lastProcessedId);
      } catch (error) {
        console.error(`Failed to process event ${event.id}:`, error);
        // Don't update lastProcessedId, will retry on next notification
        break;
      }
    }
  }
  
  private async catchUpMissedEvents(): Promise<void> {
    console.log(`Catching up from event ID ${this.lastProcessedId}...`);
    
    const result = await this.pool.query<{ max_id: string }>(`
      SELECT MAX(id) AS max_id FROM event_outbox WHERE published = FALSE
    `);
    
    const maxId = BigInt(result.rows[0]?.max_id || '0');
    
    if (maxId > this.lastProcessedId) {
      await this.processEvents(maxId);
      console.log(`Caught up to event ID ${maxId}`);
    } else {
      console.log('No missed events');
    }
  }
  
  private async handleEvent(event: {
    event_type: string;
    aggregate_id: string;
    payload: any;
  }): Promise<void> {
    console.log(`Processing event: ${event.event_type} for ${event.aggregate_id}`);
    // Route to appropriate handler based on event_type
  }
  
  private async reconnect(): Promise<void> {
    await new Promise(resolve => setTimeout(resolve, 2000));
    
    this.listenerClient = new Client({ 
      connectionString: process.env.DATABASE_URL 
    });
    
    await this.listenerClient.connect();
    await this.listenerClient.query('LISTEN events');
    
    // Catch up on missed events during downtime
    await this.catchUpMissedEvents();
  }
  
  private async loadLastProcessedId(): Promise<bigint> {
    // Load from Redis or file
    return 0n;
  }
  
  private async saveLastProcessedId(id: bigint): Promise<void> {
    // Save to Redis or file
  }
}
```

---

## React Frontend: Auto-Updates

```tsx
// useOrderStatus.tsx
import { useEffect, useState, useRef, useCallback } from 'react';

interface OrderStatus {
  orderId: string;
  status: string;
  updatedAt: Date;
}

interface WebSocketMessage {
  type: string;
  data: any;
}

function useOrderStatus(orderId: string, authToken: string) {
  const [status, setStatus] = useState<OrderStatus | null>(null);
  const [connected, setConnected] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const wsRef = useRef<WebSocket | null>(null);
  const reconnectTimeoutRef = useRef<NodeJS.Timeout | null>(null);
  
  const connect = useCallback(() => {
    const ws = new WebSocket(`ws://localhost:8080`);
    wsRef.current = ws;
    
    ws.onopen = () => {
      setConnected(true);
      setError(null);
      
      // Authenticate
      ws.send(JSON.stringify({ type: 'AUTHENTICATE', token: authToken }));
    };
    
    ws.onmessage = (event) => {
      const message: WebSocketMessage = JSON.parse(event.data);
      
      switch (message.type) {
        case 'AUTHENTICATED':
          // Subscribe to order updates
          ws.send(JSON.stringify({ 
            type: 'SUBSCRIBE', 
            channel: `customer:${message.data.customerId}` 
          }));
          
          // Get current order state
          ws.send(JSON.stringify({ 
            type: 'SUBSCRIBE_ORDER', 
            orderId 
          }));
          break;
        
        case 'ORDER_CURRENT_STATE':
          if (message.data.id === orderId) {
            setStatus({
              orderId: message.data.id,
              status: message.data.status,
              updatedAt: new Date(message.data.updated_at),
            });
          }
          break;
        
        case 'ORDER_UPDATE':
          if (message.data.orderId === orderId) {
            setStatus(prev => ({
              orderId: message.data.orderId,
              status: message.data.status,
              updatedAt: new Date(message.data.timestamp * 1000),
            }));
          }
          break;
      }
    };
    
    ws.onclose = () => {
      setConnected(false);
      
      // Auto-reconnect with exponential backoff
      reconnectTimeoutRef.current = setTimeout(connect, 3000);
    };
    
    ws.onerror = () => {
      setError('WebSocket connection failed');
      ws.close();
    };
  }, [orderId, authToken]);
  
  useEffect(() => {
    connect();
    
    return () => {
      wsRef.current?.close();
      if (reconnectTimeoutRef.current) {
        clearTimeout(reconnectTimeoutRef.current);
      }
    };
  }, [connect]);
  
  return { status, connected, error };
}

// Component ตัวอย่าง
function OrderTracking({ orderId, authToken }: { orderId: string; authToken: string }) {
  const { status, connected, error } = useOrderStatus(orderId, authToken);
  
  const statusColors: Record<string, string> = {
    pending: 'text-yellow-600',
    confirmed: 'text-blue-600',
    processing: 'text-blue-600',
    shipped: 'text-purple-600',
    delivered: 'text-green-600',
    cancelled: 'text-red-600',
  };
  
  return (
    <div className="p-4 border rounded">
      <div className="flex items-center gap-2 mb-2">
        <span className={`w-2 h-2 rounded-full ${connected ? 'bg-green-500' : 'bg-red-500'}`} />
        <span className="text-sm text-gray-500">
          {connected ? 'Live updates' : 'Connecting...'}
        </span>
      </div>
      
      {error && (
        <div className="text-red-500 text-sm mb-2">{error}</div>
      )}
      
      {status && (
        <div>
          <h3 className="font-semibold">Order #{orderId.slice(0, 8)}</h3>
          <p className={`text-lg font-bold capitalize ${statusColors[status.status] || ''}`}>
            {status.status.replace('_', ' ')}
          </p>
          <p className="text-sm text-gray-400">
            Last updated: {status.updatedAt.toLocaleString()}
          </p>
        </div>
      )}
    </div>
  );
}

export default OrderTracking;
```

---

## NOTIFY vs Logical Replication vs CDC

```
เปรียบเทียบ Real-time Solutions:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

LISTEN/NOTIFY:
✓ Built into PostgreSQL (ไม่ต้อง install อะไรเพิ่ม)
✓ Simple to implement
✓ Works with any PostgreSQL (no version restrictions)
✗ Notifications lost on disconnect (not durable)
✗ Payload limit: 8000 bytes
✗ No replay/history
✗ Not suitable for multiple consumers across processes
Use when: Simple real-time updates, cache invalidation, small payload

Logical Replication + Debezium (CDC):
✓ Durable: changes stored in WAL (replayable)
✓ High throughput (millions of changes/sec)
✓ Schema changes tracked
✓ Full row data available
✓ Multiple consumers (via Kafka)
✗ Complex setup
✗ Requires WAL level = logical
✗ Additional infrastructure (Kafka, Debezium)
Use when: Event sourcing, audit trails, microservices sync, ETL

Outbox Pattern + Polling:
✓ Reliable (events persisted in DB)
✓ At-least-once delivery
✓ Idempotent consumption
✗ Polling overhead
✗ Latency (polling interval)
Use when: Guaranteed delivery required, recovery after downtime

RECOMMENDATION:
- Start with LISTEN/NOTIFY (simple)
- Add Outbox pattern when reliability is needed
- Move to Debezium/CDC when scale becomes an issue
```

---

## Full Working System: Real-time Dashboard

### Database Setup

```sql
-- สร้าง complete schema สำหรับ real-time dashboard

-- Inventory table
CREATE TABLE inventory (
    product_id UUID PRIMARY KEY,
    quantity INTEGER NOT NULL DEFAULT 0,
    reserved INTEGER NOT NULL DEFAULT 0,
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    CONSTRAINT positive_quantity CHECK (quantity >= 0),
    CONSTRAINT positive_reserved CHECK (reserved >= 0 AND reserved <= quantity)
);

-- ฟังก์ชัน notify inventory changes
CREATE OR REPLACE FUNCTION notify_inventory_change()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    PERFORM pg_notify('inventory', json_build_object(
        'productId', NEW.product_id,
        'quantity', NEW.quantity,
        'reserved', NEW.reserved,
        'available', NEW.quantity - NEW.reserved,
        'timestamp', extract(epoch from NOW())::bigint
    )::text);
    RETURN NEW;
END $$;

CREATE TRIGGER inventory_notify
    AFTER INSERT OR UPDATE ON inventory
    FOR EACH ROW EXECUTE FUNCTION notify_inventory_change();

-- สร้าง views สำหรับ dashboard
CREATE OR REPLACE VIEW dashboard_stats AS
SELECT
    COUNT(DISTINCT o.id) FILTER (WHERE o.created_at >= NOW() - INTERVAL '24 hours') AS orders_today,
    SUM(o.total_amount) FILTER (WHERE o.created_at >= NOW() - INTERVAL '24 hours') AS revenue_today,
    COUNT(DISTINCT o.id) FILTER (WHERE o.status = 'pending') AS pending_orders,
    COUNT(DISTINCT o.id) FILTER (WHERE o.status = 'processing') AS processing_orders
FROM orders o;
```

### Node.js Backend

```typescript
// dashboard-server.ts - Complete real-time dashboard backend
import express from 'express';
import { createServer } from 'http';
import { WebSocketServer } from 'ws';
import { Client, Pool } from 'pg';

const app = express();
app.use(express.json());

const server = createServer(app);
const wss = new WebSocketServer({ server });
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// Dashboard stats cache
let dashboardCache: any = null;
let cacheUpdatedAt: Date | null = null;

// Setup PostgreSQL listener
async function setupNotifications() {
  const listener = new Client({ connectionString: process.env.DATABASE_URL });
  await listener.connect();
  
  await listener.query('LISTEN orders');
  await listener.query('LISTEN inventory');
  
  listener.on('notification', async (msg) => {
    // Broadcast to all connected dashboard clients
    const payload = JSON.parse(msg.payload || '{}');
    
    // Invalidate cache
    dashboardCache = null;
    
    // Broadcast to all WebSocket clients
    wss.clients.forEach(client => {
      if (client.readyState === 1) {  // OPEN
        client.send(JSON.stringify({
          type: msg.channel.toUpperCase() + '_UPDATE',
          data: payload,
        }));
      }
    });
  });
}

// REST API: Get dashboard stats
app.get('/api/dashboard', async (req, res) => {
  // Return from cache if fresh (< 5 seconds old)
  if (dashboardCache && cacheUpdatedAt && 
      Date.now() - cacheUpdatedAt.getTime() < 5000) {
    return res.json(dashboardCache);
  }
  
  const result = await pool.query('SELECT * FROM dashboard_stats');
  dashboardCache = result.rows[0];
  cacheUpdatedAt = new Date();
  
  res.json(dashboardCache);
});

// WebSocket: Push live updates to dashboard
wss.on('connection', async (ws) => {
  // Send current stats immediately
  const stats = await pool.query('SELECT * FROM dashboard_stats');
  ws.send(JSON.stringify({
    type: 'INITIAL_STATS',
    data: stats.rows[0],
  }));
  
  // Send recent orders
  const orders = await pool.query(`
    SELECT id, customer_id, status, total_amount, created_at
    FROM orders
    ORDER BY created_at DESC
    LIMIT 10
  `);
  
  ws.send(JSON.stringify({
    type: 'RECENT_ORDERS',
    data: orders.rows,
  }));
});

setupNotifications();
server.listen(3000, () => console.log('Dashboard server running on :3000'));
```

---

## สรุป LISTEN/NOTIFY

```
PostgreSQL LISTEN/NOTIFY สรุป:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Core Concepts:
- NOTIFY channel [, payload]: ส่ง notification
- LISTEN channel: รับ notification
- pg_notify(channel, payload): ส่งจาก SQL function/trigger
- Payload limit: 8000 bytes
- Notifications lost on disconnect (ไม่ durable)

Best Practices:
1. ใช้ dedicated connection สำหรับ LISTEN (ไม่ share กับ pool)
2. Implement reconnection logic เสมอ
3. ส่งแค่ ID ใน payload (ไม่ส่ง full object)
4. ใช้ Outbox pattern สำหรับ reliability
5. Test timeout และ reconnection scenarios

Use Cases เหมาะสมที่สุด:
✓ Cache invalidation: เร็ว, ไม่ต้องการ durability สูง
✓ Real-time dashboards: update statistic live
✓ Background job triggers: "order created → start processing"
✓ WebSocket relay: DB change → browser notification
✓ Cross-service communication (simple cases)

ไม่เหมาะสำหรับ:
✗ High-volume events (> 10,000/sec)
✗ Critical events ที่ต้องการ guaranteed delivery
✗ Events ที่ต้องการ replay
✗ Multiple consumer groups ที่ต้องการ independent offsets
  (ใช้ Kafka/CDC แทน)
```

จบ Part 80 - Real-time Features ด้วย PostgreSQL LISTEN/NOTIFY ครอบคลุม:
- LISTEN/NOTIFY mechanism และ syntax
- Trigger + NOTIFY สำหรับ automatic notifications
- Node.js listener implementation พร้อม reconnection
- Patterns: WebSocket relay, cache invalidation, email/push dispatch
- Payload size limits และ workarounds
- Reliability: Outbox pattern สำหรับ guaranteed delivery
- เปรียบเทียบกับ CDC/Logical Replication
- Full working example: real-time dashboard
- React frontend สำหรับ live order tracking

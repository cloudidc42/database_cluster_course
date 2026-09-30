# Part 39: Pub/Sub Messaging ด้วย Redis

## บทนำ

Publish/Subscribe (Pub/Sub) เป็น Messaging Pattern ที่ช่วยให้ Components ต่างๆ ใน System สื่อสารกันโดยไม่ต้องรู้จักกันโดยตรง ผู้ส่ง (Publisher) ไม่ต้องรู้ว่าใครรับ (Subscriber) และ Subscriber ไม่ต้องรู้ว่าใคร Publish

ในบทนี้เราจะเรียนรู้:
- Redis Pub/Sub ทำงานอย่างไร
- การใช้ ioredis สำหรับ Pub/Sub
- WebSocket + Redis Pub/Sub สำหรับ Real-time
- Socket.io Redis Adapter
- Real-time Chat System
- Live Notifications

---

## 1. Publish/Subscribe Pattern

### 1.1 แนวคิด

```
ไม่มี Pub/Sub (Tight Coupling):
┌──────────────┐    direct call    ┌──────────────┐
│   Service A  │──────────────────▶│   Service B  │
└──────────────┘                   └──────────────┘

ปัญหา: Service A ต้องรู้ address ของ Service B
       ถ้า Service B ล่ม Service A ก็ล้มเหลว
       ขยายได้ยาก

มี Pub/Sub (Loose Coupling):
┌──────────────┐                   ┌──────────────┐
│  Publisher   │──── channel ─────▶│  Subscriber  │
│  (Service A) │       ↑           │  (Service B) │
└──────────────┘   Message         └──────────────┘
                    Broker                  │
                  (Redis)          ┌──────────────┐
                                   │  Subscriber  │
                                   │  (Service C) │
                                   └──────────────┘

ข้อดี: ไม่ tight coupling
       Subscriber ไม่กระทบ Publisher
       เพิ่ม Subscriber ได้ง่าย
```

### 1.2 เปรียบเทียบกับ Queue

| ฟีเจอร์ | Redis Pub/Sub | Redis Queue (BullMQ) |
|---------|--------------|---------------------|
| Delivery | Fire and forget | At-least-once |
| Persistence | ❌ ไม่มี | ✅ มี |
| Multiple consumers | ✅ ทุกคนรับ | ❌ คนเดียวรับต่อ job |
| Offline subscribers | ❌ ไม่ได้รับ | ✅ ได้รับเมื่อ reconnect |
| Message ordering | ✅ FIFO | ✅ FIFO |
| Use case | Real-time events | Background jobs |

---

## 2. Redis Pub/Sub Commands

```redis
# Publisher ส่งข้อความ
PUBLISH channel message

# Subscriber รับข้อความ
SUBSCRIBE channel [channel2 ...]

# หยุด subscribe
UNSUBSCRIBE channel [channel2 ...]

# Subscribe ด้วย pattern
PSUBSCRIBE pattern [pattern2 ...]
# ตัวอย่าง: PSUBSCRIBE "events:*"  → รับทุก channel ที่ขึ้นต้นด้วย events:
# ตัวอย่าง: PSUBSCRIBE "user:*:notifications"

# หยุด pattern subscribe
PUNSUBSCRIBE pattern

# ดูจำนวน subscribers
PUBSUB NUMSUB channel
PUBSUB CHANNELS pattern
PUBSUB NUMPAT
```

### 2.1 ตัวอย่างใน Redis CLI

```bash
# Terminal 1 - Subscriber
redis-cli
> SUBSCRIBE notifications:user:123
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "notifications:user:123"
3) (integer) 1

# Terminal 2 - Publisher
redis-cli
> PUBLISH notifications:user:123 '{"type":"like","postId":"456","fromUser":"jane"}'
(integer) 1    # จำนวน subscribers ที่ได้รับ

# Terminal 1 ได้รับ:
1) "message"
2) "notifications:user:123"
3) "{\"type\":\"like\",\"postId\":\"456\",\"fromUser\":\"jane\"}"
```

---

## 3. Channel Naming Conventions

### 3.1 Pattern Design

```typescript
// =====================================================
// Channel Naming Conventions
// =====================================================

// Format: {scope}:{entity}:{action}
// ตัวอย่าง:

// User notifications
const channels = {
  // User-specific
  userNotifications: (userId: string) => `notifications:user:${userId}`,
  userPresence: (userId: string) => `presence:user:${userId}`,
  userTyping: (roomId: string, userId: string) => `typing:room:${roomId}:user:${userId}`,
  
  // Room/Chat
  roomMessages: (roomId: string) => `chat:room:${roomId}:messages`,
  roomPresence: (roomId: string) => `chat:room:${roomId}:presence`,
  
  // Product/Inventory
  productUpdates: (productId: string) => `product:${productId}:updates`,
  inventoryChanges: (warehouseId: string) => `inventory:${warehouseId}:changes`,
  
  // System events
  systemAlerts: 'system:alerts',
  deployEvents: 'system:deploy',
  
  // Application events
  orderStatus: (orderId: string) => `order:${orderId}:status`,
  paymentEvents: 'payments:events',
  
  // Patterns for PSUBSCRIBE
  allUserNotifications: 'notifications:user:*',
  allChatRooms: 'chat:room:*:messages',
  allProductUpdates: 'product:*:updates',
};
```

---

## 4. ioredis Pub/Sub Implementation

### 4.1 Basic Publisher/Subscriber

```typescript
// src/pubsub/PubSubService.ts

import Redis from 'ioredis';

export interface Message {
  type: string;
  timestamp: number;
  data: any;
  from?: string;
}

export type MessageHandler = (channel: string, message: Message) => void | Promise<void>;

export class PubSubService {
  private publisher: Redis;
  private subscriber: Redis;          // Subscriber ต้องใช้ connection แยก!
  private handlers: Map<string, Set<MessageHandler>> = new Map();
  private patternHandlers: Map<string, Set<MessageHandler>> = new Map();

  constructor(redisOptions: {
    host: string;
    port: number;
    password?: string;
  }) {
    // Publisher connection - ใช้สำหรับ PUBLISH commands
    this.publisher = new Redis({
      ...redisOptions,
      lazyConnect: true,
    });

    // Subscriber connection - ใช้สำหรับ SUBSCRIBE/PSUBSCRIBE
    // IMPORTANT: connection ที่ subscribe แล้วจะ block ไม่สามารถใช้ทำอย่างอื่นได้
    this.subscriber = new Redis({
      ...redisOptions,
      lazyConnect: true,
    });

    this.setupEventHandlers();
  }

  private setupEventHandlers(): void {
    // Handle incoming messages
    this.subscriber.on('message', async (channel: string, rawMessage: string) => {
      try {
        const message: Message = JSON.parse(rawMessage);
        const channelHandlers = this.handlers.get(channel);
        
        if (channelHandlers) {
          for (const handler of channelHandlers) {
            await handler(channel, message);
          }
        }
      } catch (error) {
        console.error(`[PubSub] Error handling message on ${channel}:`, error);
      }
    });

    // Handle pattern messages
    this.subscriber.on('pmessage', async (pattern: string, channel: string, rawMessage: string) => {
      try {
        const message: Message = JSON.parse(rawMessage);
        const patternHandlers = this.patternHandlers.get(pattern);
        
        if (patternHandlers) {
          for (const handler of patternHandlers) {
            await handler(channel, message);
          }
        }
      } catch (error) {
        console.error(`[PubSub] Error handling pmessage on ${channel}:`, error);
      }
    });

    // Connection events
    this.subscriber.on('connect', () => {
      console.log('[PubSub] Subscriber connected to Redis');
    });

    this.subscriber.on('error', (error) => {
      console.error('[PubSub] Subscriber Redis error:', error);
    });

    this.publisher.on('error', (error) => {
      console.error('[PubSub] Publisher Redis error:', error);
    });
  }

  /**
   * Connect both publisher and subscriber
   */
  async connect(): Promise<void> {
    await Promise.all([
      this.publisher.connect(),
      this.subscriber.connect(),
    ]);
    console.log('[PubSub] Connected to Redis');
  }

  /**
   * Publish ข้อความไปยัง channel
   */
  async publish(channel: string, type: string, data: any): Promise<number> {
    const message: Message = {
      type,
      timestamp: Date.now(),
      data,
    };

    const subscribers = await this.publisher.publish(channel, JSON.stringify(message));
    
    if (subscribers > 0) {
      console.log(`[PubSub] Published to ${channel}: ${type} (${subscribers} subscribers)`);
    }
    
    return subscribers;
  }

  /**
   * Subscribe ไปยัง channel
   */
  async subscribe(channel: string, handler: MessageHandler): Promise<void> {
    if (!this.handlers.has(channel)) {
      this.handlers.set(channel, new Set());
      await this.subscriber.subscribe(channel);
      console.log(`[PubSub] Subscribed to channel: ${channel}`);
    }
    
    this.handlers.get(channel)!.add(handler);
  }

  /**
   * Subscribe ไปยัง pattern
   */
  async psubscribe(pattern: string, handler: MessageHandler): Promise<void> {
    if (!this.patternHandlers.has(pattern)) {
      this.patternHandlers.set(pattern, new Set());
      await this.subscriber.psubscribe(pattern);
      console.log(`[PubSub] Subscribed to pattern: ${pattern}`);
    }
    
    this.patternHandlers.get(pattern)!.add(handler);
  }

  /**
   * Unsubscribe จาก channel
   */
  async unsubscribe(channel: string, handler?: MessageHandler): Promise<void> {
    const channelHandlers = this.handlers.get(channel);
    
    if (!channelHandlers) return;
    
    if (handler) {
      channelHandlers.delete(handler);
    } else {
      channelHandlers.clear();
    }
    
    if (channelHandlers.size === 0) {
      this.handlers.delete(channel);
      await this.subscriber.unsubscribe(channel);
      console.log(`[PubSub] Unsubscribed from: ${channel}`);
    }
  }

  /**
   * ดูจำนวน subscribers ของ channel
   */
  async getSubscriberCount(channel: string): Promise<number> {
    const result = await this.publisher.pubsub('numsub', channel) as any[];
    return parseInt(result[1] || '0');
  }

  /**
   * ดู channels ที่มี subscribers
   */
  async getActiveChannels(pattern: string = '*'): Promise<string[]> {
    return this.publisher.pubsub('channels', pattern) as Promise<string[]>;
  }

  /**
   * Close connections
   */
  async close(): Promise<void> {
    await Promise.all([
      this.publisher.quit(),
      this.subscriber.quit(),
    ]);
  }
}
```

---

## 5. Notification Service

```typescript
// src/notifications/NotificationService.ts

import { PubSubService } from '../pubsub/PubSubService';
import { Pool } from 'pg';

export type NotificationType = 
  | 'like'
  | 'comment' 
  | 'follow'
  | 'mention'
  | 'order_update'
  | 'system_alert'
  | 'achievement';

export interface Notification {
  id: string;
  userId: string;
  type: NotificationType;
  title: string;
  body: string;
  data: Record<string, any>;
  read: boolean;
  createdAt: Date;
}

export class NotificationService {
  private pubsub: PubSubService;
  private db: Pool;

  constructor(pubsub: PubSubService, db: Pool) {
    this.pubsub = pubsub;
    this.db = db;
  }

  /**
   * ส่ง notification ไปยัง user
   */
  async notify(userId: string, type: NotificationType, data: {
    title: string;
    body: string;
    metadata?: Record<string, any>;
  }): Promise<Notification> {
    // 1. บันทึกลง database
    const result = await this.db.query(`
      INSERT INTO notifications (user_id, type, title, body, data, read)
      VALUES ($1, $2, $3, $4, $5, false)
      RETURNING *
    `, [userId, type, data.title, data.body, JSON.stringify(data.metadata ?? {})]);

    const notification: Notification = result.rows[0];

    // 2. Publish real-time notification
    const channel = `notifications:user:${userId}`;
    await this.pubsub.publish(channel, 'notification', notification);

    return notification;
  }

  /**
   * Notify multiple users
   */
  async notifyBulk(userIds: string[], type: NotificationType, data: {
    title: string;
    body: string;
    metadata?: Record<string, any>;
  }): Promise<void> {
    // Insert all notifications in bulk
    const values = userIds.map((userId, i) => 
      `($${i * 5 + 1}, $${i * 5 + 2}, $${i * 5 + 3}, $${i * 5 + 4}, $${i * 5 + 5}, false)`
    ).join(', ');

    const params = userIds.flatMap(userId => [
      userId, type, data.title, data.body, JSON.stringify(data.metadata ?? {}),
    ]);

    await this.db.query(
      `INSERT INTO notifications (user_id, type, title, body, data, read) VALUES ${values}`,
      params
    );

    // Publish to each user's channel
    await Promise.all(
      userIds.map(userId =>
        this.pubsub.publish(
          `notifications:user:${userId}`,
          'notification',
          { type, title: data.title, body: data.body, metadata: data.metadata }
        )
      )
    );
  }

  /**
   * Mark notification as read
   */
  async markAsRead(notificationId: string, userId: string): Promise<void> {
    await this.db.query(
      `UPDATE notifications SET read = true WHERE id = $1 AND user_id = $2`,
      [notificationId, userId]
    );

    // Publish read event
    await this.pubsub.publish(
      `notifications:user:${userId}`,
      'notification_read',
      { notificationId }
    );
  }

  /**
   * Mark all as read
   */
  async markAllAsRead(userId: string): Promise<number> {
    const result = await this.db.query(
      `UPDATE notifications SET read = true WHERE user_id = $1 AND read = false RETURNING id`,
      [userId]
    );

    if (result.rowCount && result.rowCount > 0) {
      await this.pubsub.publish(
        `notifications:user:${userId}`,
        'all_notifications_read',
        { count: result.rowCount }
      );
    }

    return result.rowCount ?? 0;
  }

  /**
   * Get unread count
   */
  async getUnreadCount(userId: string): Promise<number> {
    const result = await this.db.query(
      `SELECT COUNT(*) FROM notifications WHERE user_id = $1 AND read = false`,
      [userId]
    );
    return parseInt(result.rows[0].count);
  }
}
```

---

## 6. Real-time Chat System

```typescript
// src/chat/ChatService.ts

import { PubSubService } from '../pubsub/PubSubService';
import { Pool } from 'pg';
import Redis from 'ioredis';

export interface ChatMessage {
  id: string;
  roomId: string;
  userId: string;
  username: string;
  content: string;
  type: 'text' | 'image' | 'file' | 'system';
  replyTo?: string;
  createdAt: Date;
}

export interface TypingEvent {
  userId: string;
  username: string;
  isTyping: boolean;
  roomId: string;
}

export interface PresenceEvent {
  userId: string;
  username: string;
  status: 'online' | 'offline' | 'away';
  lastSeen?: Date;
}

export class ChatService {
  private pubsub: PubSubService;
  private db: Pool;
  private redis: Redis;

  constructor(pubsub: PubSubService, db: Pool, redis: Redis) {
    this.pubsub = pubsub;
    this.db = db;
    this.redis = redis;
  }

  /**
   * ส่งข้อความใน room
   */
  async sendMessage(message: Omit<ChatMessage, 'id' | 'createdAt'>): Promise<ChatMessage> {
    // 1. บันทึกลง database
    const result = await this.db.query(`
      INSERT INTO chat_messages (room_id, user_id, content, type, reply_to)
      VALUES ($1, $2, $3, $4, $5)
      RETURNING *, users.username
      JOIN users ON users.id = user_id
    `, [message.roomId, message.userId, message.content, message.type, message.replyTo]);

    const chatMessage: ChatMessage = result.rows[0];

    // 2. Publish ไปยัง room channel
    const channel = `chat:room:${message.roomId}:messages`;
    await this.pubsub.publish(channel, 'new_message', chatMessage);

    // 3. Update room metadata ใน Redis
    await this.redis.hset(`chat:room:${message.roomId}`, {
      lastMessage: JSON.stringify(chatMessage),
      lastActivity: Date.now().toString(),
    });

    return chatMessage;
  }

  /**
   * ส่ง typing indicator
   */
  async sendTypingIndicator(
    roomId: string, 
    userId: string, 
    username: string, 
    isTyping: boolean
  ): Promise<void> {
    const channel = `chat:room:${roomId}:typing`;
    
    await this.pubsub.publish(channel, 'typing', {
      userId,
      username,
      isTyping,
      roomId,
    } as TypingEvent);

    // Set/remove typing indicator ใน Redis (auto-expire after 5 seconds)
    const typingKey = `chat:typing:${roomId}:${userId}`;
    if (isTyping) {
      await this.redis.setex(typingKey, 5, username);
    } else {
      await this.redis.del(typingKey);
    }
  }

  /**
   * ดู users ที่กำลัง type
   */
  async getTypingUsers(roomId: string): Promise<string[]> {
    const pattern = `chat:typing:${roomId}:*`;
    const keys = await this.redis.keys(pattern);
    
    if (keys.length === 0) return [];
    
    const usernames = await Promise.all(keys.map(key => this.redis.get(key)));
    return usernames.filter(Boolean) as string[];
  }

  /**
   * Update user presence
   */
  async updatePresence(
    userId: string, 
    username: string, 
    status: 'online' | 'offline' | 'away'
  ): Promise<void> {
    const presenceKey = `presence:user:${userId}`;
    
    if (status === 'offline') {
      await this.redis.del(presenceKey);
    } else {
      await this.redis.setex(presenceKey, 300, JSON.stringify({ userId, username, status }));  // 5 min TTL
    }

    // Publish presence update
    await this.pubsub.publish(`presence:user:${userId}`, 'presence', {
      userId,
      username,
      status,
      lastSeen: status === 'offline' ? new Date() : undefined,
    } as PresenceEvent);
  }

  /**
   * Join room
   */
  async joinRoom(roomId: string, userId: string, username: string): Promise<ChatMessage> {
    // เพิ่ม member ใน Redis Set
    await this.redis.sadd(`chat:room:${roomId}:members`, userId);
    
    // ส่ง system message
    const systemMessage = await this.sendMessage({
      roomId,
      userId,
      username: 'System',
      content: `${username} joined the room`,
      type: 'system',
    });

    return systemMessage;
  }

  /**
   * Leave room
   */
  async leaveRoom(roomId: string, userId: string, username: string): Promise<void> {
    await this.redis.srem(`chat:room:${roomId}:members`, userId);
    
    await this.sendMessage({
      roomId,
      userId,
      username: 'System',
      content: `${username} left the room`,
      type: 'system',
    });
  }

  /**
   * Get room members
   */
  async getRoomMembers(roomId: string): Promise<string[]> {
    return this.redis.smembers(`chat:room:${roomId}:members`);
  }

  /**
   * Get message history
   */
  async getMessageHistory(
    roomId: string, 
    limit: number = 50,
    before?: string   // message ID
  ): Promise<ChatMessage[]> {
    const query = before
      ? `SELECT * FROM chat_messages WHERE room_id = $1 AND id < $2 ORDER BY created_at DESC LIMIT $3`
      : `SELECT * FROM chat_messages WHERE room_id = $1 ORDER BY created_at DESC LIMIT $2`;
    
    const params = before ? [roomId, before, limit] : [roomId, limit];
    const result = await this.db.query(query, params);
    
    return result.rows.reverse();  // Return in chronological order
  }
}
```

---

## 7. Socket.io + Redis Adapter

```typescript
// src/socket/setupSocketIO.ts

import { Server as HTTPServer } from 'http';
import { Server as SocketIOServer } from 'socket.io';
import { createAdapter } from '@socket.io/redis-adapter';
import { createClient } from 'redis';  // ใช้ redis v4 สำหรับ adapter
import { ChatService } from '../chat/ChatService';
import { NotificationService } from '../notifications/NotificationService';
import jwt from 'jsonwebtoken';

export interface SocketUser {
  id: string;
  username: string;
  email: string;
}

interface ServerToClientEvents {
  'chat:message': (message: any) => void;
  'chat:typing': (event: any) => void;
  'chat:presence': (event: any) => void;
  'notification': (notification: any) => void;
  'notification:unread_count': (count: number) => void;
  error: (error: { message: string }) => void;
}

interface ClientToServerEvents {
  'chat:join': (roomId: string, callback: (error?: string) => void) => void;
  'chat:leave': (roomId: string) => void;
  'chat:message': (data: { roomId: string; content: string; type?: string }, callback: (error?: string) => void) => void;
  'chat:typing': (data: { roomId: string; isTyping: boolean }) => void;
  'notification:read': (notificationId: string) => void;
  'notification:read_all': () => void;
}

interface InterServerEvents {
  ping: () => void;
}

interface SocketData {
  user: SocketUser;
}

export async function setupSocketIO(
  httpServer: HTTPServer,
  chatService: ChatService,
  notificationService: NotificationService
): Promise<SocketIOServer> {
  // สร้าง Redis clients สำหรับ adapter
  // ต้องใช้ redis v4 (node-redis) ไม่ใช่ ioredis สำหรับ adapter
  const pubClient = createClient({
    url: `redis://${process.env.REDIS_HOST ?? 'localhost'}:${process.env.REDIS_PORT ?? '6379'}`,
    password: process.env.REDIS_PASSWORD,
  });
  
  const subClient = pubClient.duplicate();
  
  await Promise.all([
    pubClient.connect(),
    subClient.connect(),
  ]);

  const io = new SocketIOServer<
    ClientToServerEvents,
    ServerToClientEvents,
    InterServerEvents,
    SocketData
  >(httpServer, {
    // ใช้ Redis Adapter ทำให้ Socket.io ทำงานข้าม servers ได้
    adapter: createAdapter(pubClient, subClient),
    
    cors: {
      origin: process.env.FRONTEND_URL ?? 'http://localhost:3001',
      methods: ['GET', 'POST'],
      credentials: true,
    },
    
    // Ping settings
    pingTimeout: 60000,
    pingInterval: 25000,
    
    // เพิ่ม compression
    perMessageDeflate: {
      threshold: 1024,
    },
  });

  // ========== Authentication Middleware ==========
  io.use(async (socket, next) => {
    try {
      const token = socket.handshake.auth.token || 
                    socket.handshake.headers.authorization?.split(' ')[1];
      
      if (!token) {
        return next(new Error('Authentication required'));
      }

      const decoded = jwt.verify(token, process.env.JWT_SECRET!) as any;
      socket.data.user = {
        id: decoded.userId,
        username: decoded.username,
        email: decoded.email,
      };
      
      next();
    } catch (error) {
      next(new Error('Invalid token'));
    }
  });

  // ========== Connection Handler ==========
  io.on('connection', async (socket) => {
    const user = socket.data.user;
    console.log(`[Socket] User connected: ${user.username} (${socket.id})`);

    // Update presence
    await chatService.updatePresence(user.id, user.username, 'online');
    
    // Subscribe to personal notification channel
    // NOTE: ใน Socket.io + Redis Adapter ไม่ต้องใช้ Redis Pub/Sub manual
    // Socket.io จัดการเอง ผ่าน adapter
    const userRoom = `user:${user.id}`;
    socket.join(userRoom);

    // Send unread notification count on connect
    const unreadCount = await notificationService.getUnreadCount(user.id);
    socket.emit('notification:unread_count', unreadCount);

    // ========== Chat Events ==========

    socket.on('chat:join', async (roomId: string, callback) => {
      try {
        // Check permission (simplified)
        socket.join(`room:${roomId}`);
        
        await chatService.joinRoom(roomId, user.id, user.username);
        
        // Broadcast presence to room members
        socket.to(`room:${roomId}`).emit('chat:presence', {
          userId: user.id,
          username: user.username,
          status: 'online',
        });
        
        callback();  // Success
      } catch (error: any) {
        callback(error.message);
      }
    });

    socket.on('chat:leave', async (roomId: string) => {
      socket.leave(`room:${roomId}`);
      await chatService.leaveRoom(roomId, user.id, user.username);
    });

    socket.on('chat:message', async (data, callback) => {
      try {
        const { roomId, content, type = 'text' } = data;
        
        // ตรวจสอบ input
        if (!content?.trim()) {
          return callback('Message cannot be empty');
        }
        
        if (content.length > 5000) {
          return callback('Message too long');
        }

        const message = await chatService.sendMessage({
          roomId,
          userId: user.id,
          username: user.username,
          content: content.trim(),
          type: type as any,
        });

        // Broadcast to all room members (including sender)
        io.to(`room:${roomId}`).emit('chat:message', message);
        
        callback();  // Success
      } catch (error: any) {
        callback(error.message);
      }
    });

    socket.on('chat:typing', async ({ roomId, isTyping }) => {
      // Broadcast to other room members (not sender)
      socket.to(`room:${roomId}`).emit('chat:typing', {
        userId: user.id,
        username: user.username,
        isTyping,
        roomId,
      });
      
      await chatService.sendTypingIndicator(roomId, user.id, user.username, isTyping);
    });

    // ========== Notification Events ==========
    
    socket.on('notification:read', async (notificationId) => {
      await notificationService.markAsRead(notificationId, user.id);
    });

    socket.on('notification:read_all', async () => {
      await notificationService.markAllAsRead(user.id);
    });

    // ========== Disconnect ==========
    socket.on('disconnect', async (reason) => {
      console.log(`[Socket] User disconnected: ${user.username} (${reason})`);
      
      // Wait a moment to see if user reconnects
      setTimeout(async () => {
        const sockets = await io.in(userRoom).fetchSockets();
        
        if (sockets.length === 0) {
          // User truly offline
          await chatService.updatePresence(user.id, user.username, 'offline');
        }
      }, 5000);
    });

    socket.on('error', (error) => {
      console.error(`[Socket] Error for ${user.username}:`, error);
    });
  });

  console.log('[Socket] Socket.io server initialized with Redis adapter');

  return io;
}

// =====================================================
// Send notification to specific user from anywhere
// =====================================================

// Function ที่เรียกจาก NotificationService เพื่อ push to user's socket
export async function sendNotificationToUser(
  io: SocketIOServer,
  userId: string,
  notification: any
): Promise<void> {
  // ส่งไปยัง user's personal room
  // Redis Adapter จะ forward ไปยัง server ที่ user connected อยู่
  io.to(`user:${userId}`).emit('notification', notification);
}
```

---

## 8. Redis Streams vs Pub/Sub

```typescript
// =====================================================
// Redis Streams: เมื่อไหร่ใช้อะไร
// =====================================================

/*
Pub/Sub ใช้เมื่อ:
- Real-time messaging ที่ต้องการ latency ต่ำ
- ข้อความไม่จำเป็นต้อง persist
- ทุก subscriber ต้องได้รับข้อความพร้อมกัน
- Fire-and-forget events

ตัวอย่าง:
- Chat messages
- Live notifications
- Live price updates
- Real-time game events
- Typing indicators

Streams ใช้เมื่อ:
- ต้องการ message persistence
- ต้องการ replay messages
- Consumer groups (แต่ละ message ให้คนเดียวรับ)
- ต้องการ acknowledge messages
- Event sourcing
- Audit logs

ตัวอย่าง:
- Order processing pipeline
- Audit trail
- Event sourcing
- Log aggregation
*/

// ตัวอย่าง Redis Streams
import Redis from 'ioredis';

export class EventStream {
  private redis: Redis;
  private streamKey: string;

  constructor(redis: Redis, streamKey: string) {
    this.redis = redis;
    this.streamKey = streamKey;
  }

  /**
   * Add event to stream
   */
  async addEvent(type: string, data: Record<string, any>): Promise<string> {
    const id = await this.redis.xadd(
      this.streamKey,
      '*',                    // Auto-generate ID
      'type', type,
      'data', JSON.stringify(data),
      'timestamp', Date.now().toString()
    );
    
    return id as string;
  }

  /**
   * Create consumer group
   */
  async createConsumerGroup(groupName: string): Promise<void> {
    try {
      await this.redis.xgroup('CREATE', this.streamKey, groupName, '$', 'MKSTREAM');
    } catch (error: any) {
      if (!error.message.includes('BUSYGROUP')) {
        throw error;
      }
      // Group already exists, OK
    }
  }

  /**
   * Read events as consumer group member
   */
  async readEvents(
    groupName: string,
    consumerName: string,
    count: number = 10
  ): Promise<Array<{ id: string; type: string; data: any; timestamp: number }>> {
    const result = await this.redis.xreadgroup(
      'GROUP', groupName, consumerName,
      'COUNT', count,
      'BLOCK', 1000,           // Wait 1 second for new events
      'STREAMS', this.streamKey, '>'
    ) as any;

    if (!result) return [];

    const events = [];
    for (const [, messages] of result) {
      for (const [id, fields] of messages) {
        const fieldMap: Record<string, string> = {};
        for (let i = 0; i < fields.length; i += 2) {
          fieldMap[fields[i]] = fields[i + 1];
        }
        
        events.push({
          id,
          type: fieldMap.type,
          data: JSON.parse(fieldMap.data || '{}'),
          timestamp: parseInt(fieldMap.timestamp || '0'),
        });
      }
    }

    return events;
  }

  /**
   * Acknowledge processed event
   */
  async acknowledge(groupName: string, ...ids: string[]): Promise<void> {
    await this.redis.xack(this.streamKey, groupName, ...ids);
  }

  /**
   * Trim stream to max length
   */
  async trim(maxLength: number): Promise<number> {
    return this.redis.xtrim(this.streamKey, 'MAXLEN', '~', maxLength) as Promise<number>;
  }
}
```

---

## 9. Complete Real-time Application

```typescript
// src/realtime/RealtimeApp.ts

import express from 'express';
import { createServer } from 'http';
import Redis from 'ioredis';
import { Pool } from 'pg';
import { PubSubService } from '../pubsub/PubSubService';
import { ChatService } from '../chat/ChatService';
import { NotificationService } from '../notifications/NotificationService';
import { setupSocketIO, sendNotificationToUser } from '../socket/setupSocketIO';

export async function createRealtimeApp() {
  const app = express();
  const httpServer = createServer(app);
  
  app.use(express.json());

  // ========== Database connections ==========
  const db = new Pool({
    host: process.env.DB_HOST ?? 'localhost',
    database: process.env.DB_NAME ?? 'chatapp',
    user: process.env.DB_USER ?? 'postgres',
    password: process.env.DB_PASSWORD,
  });

  const redis = new Redis({
    host: process.env.REDIS_HOST ?? 'localhost',
    port: parseInt(process.env.REDIS_PORT ?? '6379'),
  });

  // ========== Services ==========
  const pubsub = new PubSubService({
    host: process.env.REDIS_HOST ?? 'localhost',
    port: parseInt(process.env.REDIS_PORT ?? '6379'),
    password: process.env.REDIS_PASSWORD,
  });
  await pubsub.connect();

  const chatService = new ChatService(pubsub, db, redis);
  const notificationService = new NotificationService(pubsub, db);

  // Setup Socket.io
  const io = await setupSocketIO(httpServer, chatService, notificationService);

  // ========== HTTP Routes ==========

  // Chat history
  app.get('/api/chat/rooms/:roomId/messages', async (req, res) => {
    try {
      const { roomId } = req.params;
      const { limit = 50, before } = req.query;
      
      const messages = await chatService.getMessageHistory(
        roomId,
        parseInt(limit as string),
        before as string
      );
      
      res.json({ messages });
    } catch (error) {
      res.status(500).json({ error: 'Failed to get messages' });
    }
  });

  // Get room members  
  app.get('/api/chat/rooms/:roomId/members', async (req, res) => {
    try {
      const members = await chatService.getRoomMembers(req.params.roomId);
      res.json({ members });
    } catch (error) {
      res.status(500).json({ error: 'Failed to get members' });
    }
  });

  // Get typing users
  app.get('/api/chat/rooms/:roomId/typing', async (req, res) => {
    try {
      const typing = await chatService.getTypingUsers(req.params.roomId);
      res.json({ typing });
    } catch (error) {
      res.status(500).json({ error: 'Failed to get typing users' });
    }
  });

  // Send notification (internal API)
  app.post('/internal/notifications', async (req, res) => {
    try {
      const { userId, type, title, body, metadata } = req.body;
      
      const notification = await notificationService.notify(userId, type, {
        title, body, metadata,
      });
      
      // Push to socket
      await sendNotificationToUser(io, userId, notification);
      
      res.json({ notification });
    } catch (error) {
      res.status(500).json({ error: 'Failed to send notification' });
    }
  });

  // Broadcast notification to multiple users
  app.post('/internal/notifications/broadcast', async (req, res) => {
    try {
      const { userIds, type, title, body } = req.body;
      
      await notificationService.notifyBulk(userIds, type, { title, body });
      
      // Push to all sockets
      await Promise.all(
        userIds.map((userId: string) =>
          sendNotificationToUser(io, userId, { type, title, body })
        )
      );
      
      res.json({ message: `Notification sent to ${userIds.length} users` });
    } catch (error) {
      res.status(500).json({ error: 'Failed to broadcast notification' });
    }
  });

  // ========== Start Server ==========
  const port = parseInt(process.env.PORT ?? '3000');
  httpServer.listen(port, () => {
    console.log(`
🚀 Real-time server started!
   HTTP: http://localhost:${port}
   WebSocket: ws://localhost:${port}
    `);
  });

  return { app, io, httpServer };
}

// ========== Frontend Integration Example ==========
/*
// Client-side JavaScript/TypeScript (Browser)

import { io } from 'socket.io-client';

const socket = io('http://localhost:3000', {
  auth: {
    token: localStorage.getItem('jwt_token')
  }
});

// Connection events
socket.on('connect', () => {
  console.log('Connected!', socket.id);
});

socket.on('disconnect', (reason) => {
  console.log('Disconnected:', reason);
});

// Join a chat room
socket.emit('chat:join', 'room-123', (error) => {
  if (error) {
    console.error('Join failed:', error);
  } else {
    console.log('Joined room!');
  }
});

// Send a message
socket.emit('chat:message', {
  roomId: 'room-123',
  content: 'Hello everyone!',
  type: 'text'
}, (error) => {
  if (error) console.error('Send failed:', error);
});

// Listen for new messages
socket.on('chat:message', (message) => {
  console.log('New message:', message);
  appendMessageToUI(message);
});

// Typing indicator
let typingTimeout: NodeJS.Timeout;

messageInput.addEventListener('input', () => {
  socket.emit('chat:typing', { roomId: 'room-123', isTyping: true });
  
  clearTimeout(typingTimeout);
  typingTimeout = setTimeout(() => {
    socket.emit('chat:typing', { roomId: 'room-123', isTyping: false });
  }, 2000);
});

socket.on('chat:typing', ({ username, isTyping }) => {
  if (isTyping) {
    showTypingIndicator(`${username} is typing...`);
  } else {
    hideTypingIndicator();
  }
});

// Real-time notifications
socket.on('notification', (notification) => {
  console.log('New notification:', notification);
  showNotificationToast(notification);
  updateNotificationBadge();
});

socket.on('notification:unread_count', (count) => {
  updateNotificationBadge(count);
});
*/
```

---

## 10. Scaling WebSocket Across Multiple Servers

```yaml
# docker-compose.yml สำหรับ scale WebSocket servers

version: '3.8'

services:
  # Load Balancer ที่รองรับ WebSocket
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - app1
      - app2
      - app3

  # Multiple app instances
  app1:
    build: .
    environment:
      - PORT=3000
      - REDIS_HOST=redis
    depends_on:
      - redis
      - postgres

  app2:
    build: .
    environment:
      - PORT=3000
      - REDIS_HOST=redis
    depends_on:
      - redis
      - postgres

  app3:
    build: .
    environment:
      - PORT=3000
      - REDIS_HOST=redis
    depends_on:
      - redis
      - postgres

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: chatapp
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  redis_data:
  postgres_data:
```

```nginx
# nginx.conf - WebSocket Load Balancer

upstream websocket_backend {
    ip_hash;  # Sticky sessions สำหรับ WebSocket

    server app1:3000;
    server app2:3000;
    server app3:3000;
}

server {
    listen 80;

    location / {
        proxy_pass http://websocket_backend;
        
        # WebSocket support
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        
        # Timeout settings for long-lived connections
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
    }
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Redis Pub/Sub**: PUBLISH, SUBSCRIBE, PSUBSCRIBE commands
   - Publisher และ Subscriber ต้องใช้ connection แยกกัน
   - ไม่มี persistence - offline subscribers ไม่ได้รับข้อความ

2. **Channel Naming**: ใช้ format `{scope}:{entity}:{action}` เช่น `notifications:user:123`

3. **PubSubService Class**: Abstraction layer ที่จัดการ connections และ handlers

4. **Real-time Applications**:
   - NotificationService: ส่ง notifications แบบ real-time
   - ChatService: Chat rooms พร้อม typing indicators และ presence

5. **Socket.io + Redis Adapter**:
   - Scale WebSocket ข้าม multiple servers
   - Redis Adapter forward events ระหว่าง servers

6. **Redis Streams vs Pub/Sub**: ใช้ Pub/Sub สำหรับ real-time fire-and-forget, ใช้ Streams เมื่อต้องการ persistence

7. **Scaling**: nginx ip_hash สำหรับ sticky sessions + Redis Adapter สำหรับ cross-server messaging

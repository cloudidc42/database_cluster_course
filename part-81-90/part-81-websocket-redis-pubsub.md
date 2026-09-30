# Part 81: WebSocket + Redis Pub/Sub สำหรับ Real-time Features

## บทนำ

ในยุคที่ผู้ใช้คาดหวังประสบการณ์แบบ real-time ไม่ว่าจะเป็น chat แจ้งเตือนทันที หรือ dashboard ที่อัปเดตอัตโนมัติ WebSocket และ Redis Pub/Sub คือเทคโนโลยีหลักที่ขาดไม่ได้ บทนี้จะลงลึกทุกแง่มุมตั้งแต่พื้นฐานจนถึงการ scale ระบบใน production

---

## 1. WebSocket: Full-Duplex Communication

### 1.1 HTTP vs WebSocket

**HTTP แบบดั้งเดิม:**
- Client ส่ง request → Server ส่ง response → Connection ปิด
- Client ต้องเป็นฝ่ายถาม (pull model)
- ไม่เหมาะกับข้อมูลที่เปลี่ยนแปลงเร็ว

**Polling แบบเก่า (ไม่ดี):**
```javascript
// Short Polling - client ถามทุก 1 วินาที (สิ้นเปลือง)
setInterval(async () => {
  const res = await fetch('/api/messages');
  const data = await res.json();
  updateUI(data);
}, 1000);
```

**Long Polling (ดีกว่า แต่ยังไม่ดีพอ):**
```javascript
// Long Polling - client รอจนมีข้อมูลใหม่
async function longPoll() {
  try {
    const res = await fetch('/api/messages/poll?lastId=' + lastMessageId);
    const data = await res.json();
    updateUI(data);
    longPoll(); // เรียกใหม่ทันที
  } catch (err) {
    setTimeout(longPoll, 1000); // retry หลัง error
  }
}
```

**WebSocket (ดีที่สุด):**
- Full-duplex: ทั้ง client และ server ส่งข้อมูลได้ตลอดเวลา
- Connection เปิดค้างไว้ตลอด session
- Overhead น้อยกว่า HTTP มาก (ไม่ต้องส่ง headers ซ้ำ)
- Protocol: `ws://` (plain) หรือ `wss://` (TLS)

### 1.2 WebSocket Handshake

WebSocket เริ่มต้นด้วย HTTP Upgrade request:

```
Client → Server:
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13

Server → Client:
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

หลังจากนั้นจะเป็น WebSocket frames แทน HTTP messages

### 1.3 Native WebSocket API (Browser)

```javascript
// ฝั่ง Browser
const ws = new WebSocket('wss://api.example.com/ws');

// เมื่อ connection เปิด
ws.onopen = () => {
  console.log('Connected!');
  ws.send(JSON.stringify({ type: 'ping' }));
};

// เมื่อได้รับข้อมูล
ws.onmessage = (event) => {
  const data = JSON.parse(event.data);
  console.log('Received:', data);
};

// เมื่อ connection ปิด
ws.onclose = (event) => {
  console.log('Disconnected:', event.code, event.reason);
  // Reconnect logic
  if (event.code !== 1000) { // 1000 = normal close
    setTimeout(() => reconnect(), 3000);
  }
};

// เมื่อเกิด error
ws.onerror = (error) => {
  console.error('WebSocket error:', error);
};

// ส่งข้อมูล
ws.send(JSON.stringify({
  type: 'message',
  content: 'Hello World',
  roomId: 'general'
}));

// ปิด connection
ws.close(1000, 'User logged out');
```

### 1.4 ws Library (Node.js)

```bash
npm install ws @types/ws
```

```typescript
// server.ts
import WebSocket, { WebSocketServer } from 'ws';
import http from 'http';

const server = http.createServer();
const wss = new WebSocketServer({ server });

// เก็บ connections ทั้งหมด
const clients = new Map<string, WebSocket>();

wss.on('connection', (ws, req) => {
  const clientId = generateId();
  clients.set(clientId, ws);
  
  console.log(`Client ${clientId} connected`);
  
  ws.on('message', (data) => {
    const message = JSON.parse(data.toString());
    
    switch (message.type) {
      case 'broadcast':
        // ส่งให้ทุก client
        wss.clients.forEach(client => {
          if (client.readyState === WebSocket.OPEN) {
            client.send(JSON.stringify({
              from: clientId,
              content: message.content
            }));
          }
        });
        break;
        
      case 'private':
        // ส่งให้ client เฉพาะคน
        const target = clients.get(message.targetId);
        if (target?.readyState === WebSocket.OPEN) {
          target.send(JSON.stringify({
            from: clientId,
            content: message.content
          }));
        }
        break;
    }
  });
  
  ws.on('close', () => {
    clients.delete(clientId);
    console.log(`Client ${clientId} disconnected`);
  });
  
  // Ping/Pong heartbeat
  ws.on('pong', () => {
    (ws as any).isAlive = true;
  });
});

// Heartbeat: ตรวจสอบ connection ที่ตายแล้ว
const heartbeat = setInterval(() => {
  wss.clients.forEach(ws => {
    if ((ws as any).isAlive === false) {
      ws.terminate();
      return;
    }
    (ws as any).isAlive = false;
    ws.ping();
  });
}, 30000);

wss.on('close', () => clearInterval(heartbeat));

server.listen(3000);
```

---

## 2. Socket.io: The Complete Solution

Socket.io เป็น library ที่ wrap WebSocket ให้ใช้งานง่ายขึ้น พร้อม features เพิ่มเติม:
- Automatic reconnection
- Fallback to polling (สำหรับ environment ที่ไม่รองรับ WebSocket)
- Rooms และ Namespaces
- Binary data support
- Acknowledgements (callback จาก server)

### 2.1 Installation

```bash
# Server
npm install socket.io

# Client (Node.js)
npm install socket.io-client

# TypeScript types
npm install -D @types/socket.io
```

### 2.2 Server Setup

```typescript
// server/index.ts
import express from 'express';
import { createServer } from 'http';
import { Server, Socket } from 'socket.io';
import cors from 'cors';

const app = express();
app.use(cors());
app.use(express.json());

const httpServer = createServer(app);

const io = new Server(httpServer, {
  cors: {
    origin: process.env.FRONTEND_URL || 'http://localhost:3000',
    methods: ['GET', 'POST'],
    credentials: true
  },
  // ตั้งค่า ping/pong
  pingTimeout: 60000,    // รอ pong กี่ ms ก่อน disconnect
  pingInterval: 25000,   // ping ทุก 25 วินาที
  
  // ขนาด payload สูงสุด
  maxHttpBufferSize: 1e6, // 1 MB
  
  // Transport options
  transports: ['websocket', 'polling'],
  
  // Upgrade จาก polling เป็น websocket อัตโนมัติ
  allowUpgrades: true
});

// Middleware: logging
io.use((socket, next) => {
  console.log(`New connection attempt from ${socket.handshake.address}`);
  next();
});

io.on('connection', (socket: Socket) => {
  console.log(`User connected: ${socket.id}`);
  
  // ส่งข้อมูลต้อนรับ
  socket.emit('welcome', {
    socketId: socket.id,
    timestamp: new Date()
  });
  
  // รับ event
  socket.on('disconnect', (reason) => {
    console.log(`User ${socket.id} disconnected: ${reason}`);
  });
});

const PORT = process.env.PORT || 8080;
httpServer.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

### 2.3 Client Connection

```typescript
// client/socket.ts
import { io, Socket } from 'socket.io-client';

// ประเภท events สำหรับ TypeScript
interface ServerToClientEvents {
  welcome: (data: { socketId: string; timestamp: Date }) => void;
  message: (data: Message) => void;
  userJoined: (data: { userId: string; username: string }) => void;
  userLeft: (data: { userId: string }) => void;
  typing: (data: { userId: string; isTyping: boolean }) => void;
  error: (data: { message: string }) => void;
}

interface ClientToServerEvents {
  joinRoom: (roomId: string, callback: (success: boolean) => void) => void;
  leaveRoom: (roomId: string) => void;
  sendMessage: (data: SendMessageData, callback: (message: Message) => void) => void;
  typing: (data: { roomId: string; isTyping: boolean }) => void;
}

const socket: Socket<ServerToClientEvents, ClientToServerEvents> = io(
  process.env.REACT_APP_WS_URL || 'http://localhost:8080',
  {
    // Authentication
    auth: {
      token: localStorage.getItem('accessToken')
    },
    
    // Connection options
    reconnection: true,
    reconnectionAttempts: 10,
    reconnectionDelay: 1000,
    reconnectionDelayMax: 5000,
    randomizationFactor: 0.5,
    
    // Timeout
    timeout: 20000,
    
    // Transport
    transports: ['websocket', 'polling']
  }
);

// Event handlers
socket.on('connect', () => {
  console.log('Connected to server:', socket.id);
});

socket.on('connect_error', (error) => {
  console.error('Connection error:', error.message);
  
  // ถ้า auth error ให้ redirect ไป login
  if (error.message === 'Unauthorized') {
    window.location.href = '/login';
  }
});

socket.on('disconnect', (reason) => {
  console.log('Disconnected:', reason);
  
  if (reason === 'io server disconnect') {
    // Server บังคับ disconnect → ต้อง reconnect เอง
    socket.connect();
  }
  // ถ้า reason อื่นๆ socket.io จะ reconnect อัตโนมัติ
});

socket.on('reconnect', (attemptNumber) => {
  console.log(`Reconnected after ${attemptNumber} attempts`);
});

export default socket;
```

### 2.4 Events: emit, on

```typescript
// server/handlers/chatHandler.ts
import { Server, Socket } from 'socket.io';
import { MessageService } from '../services/messageService';

export function registerChatHandlers(io: Server, socket: Socket, userId: string) {
  const messageService = new MessageService();
  
  // ส่งข้อความ
  socket.on('sendMessage', async (data, callback) => {
    try {
      // Validate
      if (!data.content || !data.roomId) {
        return callback({ error: 'Invalid data' });
      }
      
      // บันทึกลง database
      const message = await messageService.create({
        content: data.content,
        roomId: data.roomId,
        userId: userId
      });
      
      // ส่งให้ทุกคนในห้อง (รวม sender)
      io.to(data.roomId).emit('message', {
        id: message.id,
        content: message.content,
        userId: message.userId,
        username: message.user.username,
        roomId: message.roomId,
        createdAt: message.createdAt
      });
      
      // Callback ให้ sender รู้ว่าสำเร็จ
      callback({ success: true, messageId: message.id });
      
    } catch (error) {
      console.error('sendMessage error:', error);
      callback({ error: 'Failed to send message' });
    }
  });
  
  // Emit ด้วย acknowledgement
  socket.on('ping', (callback) => {
    callback({ pong: true, timestamp: Date.now() });
  });
  
  // Volatile emit (ไม่สนใจถ้า packet หาย - เหมาะกับ real-time data)
  socket.on('cursor', (data) => {
    socket.volatile.to(data.roomId).emit('cursorMove', {
      userId,
      x: data.x,
      y: data.y
    });
  });
}
```

### 2.5 Rooms: join, leave, emit to room

Rooms ใน Socket.io คือ namespace ย่อยที่ socket สามารถ join ได้หลาย rooms พร้อมกัน

```typescript
// server/handlers/roomHandler.ts
import { Server, Socket } from 'socket.io';
import { RoomService } from '../services/roomService';
import { UserService } from '../services/userService';

export function registerRoomHandlers(io: Server, socket: Socket, userId: string) {
  const roomService = new RoomService();
  const userService = new UserService();
  
  // Join room
  socket.on('joinRoom', async (roomId: string, callback) => {
    try {
      // ตรวจสอบสิทธิ์
      const hasAccess = await roomService.checkAccess(userId, roomId);
      if (!hasAccess) {
        return callback({ error: 'Access denied' });
      }
      
      // Join room
      socket.join(roomId);
      
      // เพิ่ม user เข้า room ใน database
      await roomService.addMember(roomId, userId);
      
      // โหลดประวัติข้อความ
      const messages = await roomService.getMessages(roomId, { limit: 50 });
      
      // โหลด online users ในห้อง
      const socketsInRoom = await io.in(roomId).fetchSockets();
      const onlineUserIds = socketsInRoom.map(s => (s as any).userId);
      
      // แจ้ง user คนอื่นในห้อง
      socket.to(roomId).emit('userJoined', {
        userId,
        username: await userService.getUsername(userId),
        timestamp: new Date()
      });
      
      callback({
        success: true,
        messages,
        onlineUsers: onlineUserIds
      });
      
      console.log(`User ${userId} joined room ${roomId}`);
      
    } catch (error) {
      callback({ error: 'Failed to join room' });
    }
  });
  
  // Leave room
  socket.on('leaveRoom', async (roomId: string) => {
    socket.leave(roomId);
    
    // แจ้ง user คนอื่น
    io.to(roomId).emit('userLeft', {
      userId,
      timestamp: new Date()
    });
    
    console.log(`User ${userId} left room ${roomId}`);
  });
  
  // Emit to specific room
  function sendToRoom(roomId: string, event: string, data: any) {
    io.to(roomId).emit(event, data);
  }
  
  // Emit to room ยกเว้น sender
  function sendToRoomExcludeSender(roomId: string, event: string, data: any) {
    socket.to(roomId).emit(event, data);
  }
  
  // Emit to multiple rooms
  function sendToMultipleRooms(roomIds: string[], event: string, data: any) {
    io.to(roomIds).emit(event, data);
  }
  
  // ดูว่า socket อยู่ใน rooms ไหนบ้าง
  socket.on('myRooms', (callback) => {
    callback(Array.from(socket.rooms));
  });
}
```

### 2.6 Namespaces: separate contexts

```typescript
// server/namespaces/index.ts
import { Server } from 'socket.io';

export function setupNamespaces(io: Server) {
  // Default namespace: /
  io.on('connection', (socket) => {
    console.log('Connected to default namespace');
  });
  
  // Chat namespace
  const chatNs = io.of('/chat');
  chatNs.use(authenticateMiddleware);
  chatNs.on('connection', (socket) => {
    console.log('Connected to /chat namespace');
    // Chat-specific logic
  });
  
  // Admin namespace
  const adminNs = io.of('/admin');
  adminNs.use(authenticateMiddleware);
  adminNs.use(requireAdminMiddleware);
  adminNs.on('connection', (socket) => {
    console.log('Admin connected to /admin namespace');
    // Admin-specific logic
  });
  
  // Notifications namespace
  const notifNs = io.of('/notifications');
  notifNs.on('connection', (socket) => {
    // Subscribe to user-specific notifications
    const userId = socket.handshake.auth.userId;
    socket.join(`user:${userId}`);
  });
  
  // Dynamic namespaces
  const dynamicNs = io.of(/^\/room-\d+$/);
  dynamicNs.on('connection', (socket) => {
    const roomId = socket.nsp.name.replace('/room-', '');
    console.log(`Connected to room ${roomId}`);
  });
}
```

### 2.7 Authentication: JWT in Handshake

```typescript
// server/middleware/auth.ts
import { Server, Socket } from 'socket.io';
import jwt from 'jsonwebtoken';

interface DecodedToken {
  userId: string;
  email: string;
  role: string;
}

// Middleware สำหรับ authenticate WebSocket connections
export function authenticateSocket(socket: Socket, next: (err?: Error) => void) {
  const token = socket.handshake.auth.token || 
                socket.handshake.headers['authorization']?.split(' ')[1];
  
  if (!token) {
    return next(new Error('Unauthorized: No token provided'));
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET!) as DecodedToken;
    
    // เก็บ user info ไว้ใน socket
    (socket as any).userId = decoded.userId;
    (socket as any).userEmail = decoded.email;
    (socket as any).userRole = decoded.role;
    
    next();
  } catch (error) {
    if (error instanceof jwt.TokenExpiredError) {
      next(new Error('Unauthorized: Token expired'));
    } else {
      next(new Error('Unauthorized: Invalid token'));
    }
  }
}

// ใช้ middleware
export function setupAuth(io: Server) {
  io.use(authenticateSocket);
  
  io.on('connection', (socket) => {
    const userId = (socket as any).userId;
    const userRole = (socket as any).userRole;
    
    // Join user-specific room
    socket.join(`user:${userId}`);
    
    // Join role-specific room
    socket.join(`role:${userRole}`);
    
    console.log(`Authenticated user ${userId} connected`);
  });
}
```

```typescript
// client: ส่ง token ใน handshake
import { io } from 'socket.io-client';

function createAuthenticatedSocket(token: string) {
  const socket = io('http://localhost:8080', {
    auth: {
      token: token
    },
    extraHeaders: {
      'Authorization': `Bearer ${token}`
    }
  });
  
  return socket;
}
```

### 2.8 Error Handling

```typescript
// server/errorHandling.ts
import { Server, Socket } from 'socket.io';

export function setupErrorHandling(io: Server) {
  // Global error handler
  io.engine.on('connection_error', (err) => {
    console.error('Connection error:', {
      code: err.code,
      message: err.message,
      context: err.context
    });
  });
  
  io.on('connection', (socket: Socket) => {
    // Per-socket error handler
    socket.on('error', (error) => {
      console.error(`Socket ${socket.id} error:`, error);
    });
    
    // Wrap event handlers ด้วย try-catch
    const safeOn = (event: string, handler: (...args: any[]) => Promise<void>) => {
      socket.on(event, async (...args) => {
        try {
          await handler(...args);
        } catch (error) {
          console.error(`Error in ${event} handler:`, error);
          
          // ส่ง error กลับไปให้ client
          const callback = args[args.length - 1];
          if (typeof callback === 'function') {
            callback({ error: error instanceof Error ? error.message : 'Unknown error' });
          } else {
            socket.emit('error', {
              event,
              message: error instanceof Error ? error.message : 'Unknown error'
            });
          }
        }
      });
    };
    
    // ใช้ safeOn แทน socket.on
    safeOn('sendMessage', async (data, callback) => {
      // ... logic
    });
  });
}
```

---

## 3. Scaling WebSocket with Redis Adapter

### 3.1 ปัญหาของ Single Server

เมื่อมี users จำนวนมาก หรือต้องการ high availability ต้องใช้หลาย server instances แต่ปัญหาคือ WebSocket connection เป็น stateful:

```
Client A → Server 1 (connected)
Client B → Server 2 (connected)

Client A ส่ง "Hello" → ถึง Server 1
Server 1 ต้องการ broadcast ไปให้ Client B
แต่ Client B อยู่กับ Server 2!
ผลลัพธ์: Client B ไม่ได้รับข้อความ
```

### 3.2 Redis Pub/Sub เป็น Solution

```
Client A → Server 1 (connected)
Client B → Server 2 (connected)

Client A ส่ง "Hello" → Server 1
Server 1 publish ไปยัง Redis channel "broadcast"
Redis notify Server 2 (subscriber)
Server 2 ส่งไปให้ Client B
ผลลัพธ์: Client B ได้รับข้อความ
```

### 3.3 @socket.io/redis-adapter

```bash
npm install @socket.io/redis-adapter ioredis
```

```typescript
// server/redis-adapter.ts
import { Server } from 'socket.io';
import { createClient } from 'redis';
import { createAdapter } from '@socket.io/redis-adapter';

export async function setupRedisAdapter(io: Server) {
  // สร้าง Redis clients แยกกัน (pub/sub ต้องใช้ connection แยก)
  const pubClient = createClient({
    url: process.env.REDIS_URL || 'redis://localhost:6379',
    socket: {
      reconnectStrategy: (retries) => {
        if (retries > 10) {
          console.error('Redis pub client: max retries reached');
          return new Error('Max retries reached');
        }
        return Math.min(retries * 100, 3000);
      }
    }
  });
  
  const subClient = pubClient.duplicate();
  
  // Error handling
  pubClient.on('error', (err) => console.error('Redis Pub Error:', err));
  subClient.on('error', (err) => console.error('Redis Sub Error:', err));
  
  // Connect
  await Promise.all([pubClient.connect(), subClient.connect()]);
  
  console.log('Redis clients connected');
  
  // ติดตั้ง adapter
  io.adapter(createAdapter(pubClient, subClient));
  
  console.log('Redis adapter installed');
  
  return { pubClient, subClient };
}
```

### 3.4 Message Serialization

Redis adapter ใช้ messagepack หรือ JSON ในการ serialize ข้อมูล:

```typescript
// Custom serializer (optional)
import { createAdapter } from '@socket.io/redis-adapter';
import { pack, unpack } from 'msgpackr';

const adapter = createAdapter(pubClient, subClient, {
  // ใช้ custom serializer
  parser: {
    encode: pack,
    decode: unpack
  },
  
  // Prefix สำหรับ Redis keys
  key: 'myapp',
  
  // Request timeout
  requestsTimeout: 5000
});

io.adapter(adapter);
```

### 3.5 Horizontal Scaling ด้วย Load Balancer

```nginx
# nginx.conf
upstream websocket_servers {
    # Sticky session โดยใช้ IP hash
    # เพื่อให้ client reconnect ไปหา server เดิม
    ip_hash;
    
    server ws-server-1:8080;
    server ws-server-2:8080;
    server ws-server-3:8080;
}

server {
    listen 443 ssl;
    server_name api.example.com;
    
    location /socket.io/ {
        proxy_pass http://websocket_servers;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        
        # Timeout settings
        proxy_read_timeout 86400s;
        proxy_send_timeout 86400s;
    }
}
```

```yaml
# docker-compose.yml สำหรับ scaled environment
version: '3.8'

services:
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes

  ws-server-1:
    build: .
    environment:
      - REDIS_URL=redis://redis:6379
      - PORT=8080
    depends_on:
      - redis

  ws-server-2:
    build: .
    environment:
      - REDIS_URL=redis://redis:6379
      - PORT=8080
    depends_on:
      - redis

  ws-server-3:
    build: .
    environment:
      - REDIS_URL=redis://redis:6379
      - PORT=8080
    depends_on:
      - redis

  nginx:
    image: nginx:alpine
    ports:
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - ws-server-1
      - ws-server-2
      - ws-server-3

volumes:
  redis_data:
```

---

## 4. Patterns สำหรับ Real-time Features

### 4.1 Chat Rooms

```typescript
// server/features/chatRoom.ts
import { Server, Socket } from 'socket.io';
import { PrismaClient } from '@prisma/client';
import { createClient } from 'redis';

const prisma = new PrismaClient();

export function setupChatRoom(io: Server, redisClient: ReturnType<typeof createClient>) {
  io.on('connection', async (socket: Socket) => {
    const userId = (socket as any).userId as string;
    
    // === JOIN ROOM ===
    socket.on('chat:join', async ({ roomId }, callback) => {
      // ตรวจสอบ membership
      const member = await prisma.roomMember.findUnique({
        where: { roomId_userId: { roomId, userId } }
      });
      
      if (!member) {
        return callback?.({ error: 'Not a member of this room' });
      }
      
      socket.join(`room:${roomId}`);
      
      // Mark user as online ใน Redis
      await redisClient.sAdd(`room:${roomId}:online`, userId);
      await redisClient.expire(`room:${roomId}:online`, 86400);
      
      // โหลดประวัติ messages
      const messages = await prisma.message.findMany({
        where: { roomId },
        include: { user: { select: { id: true, username: true, avatar: true } } },
        orderBy: { createdAt: 'desc' },
        take: 50
      });
      
      // Online users
      const onlineUsers = await redisClient.sMembers(`room:${roomId}:online`);
      
      socket.emit('chat:history', {
        messages: messages.reverse(),
        onlineUsers
      });
      
      // แจ้ง user อื่นว่ามีคนเข้ามา
      socket.to(`room:${roomId}`).emit('chat:userJoined', {
        userId,
        username: member.user?.username
      });
      
      callback?.({ success: true });
    });
    
    // === SEND MESSAGE ===
    socket.on('chat:message', async ({ roomId, content, type = 'text' }, callback) => {
      // Validate
      if (!content?.trim()) {
        return callback?.({ error: 'Message cannot be empty' });
      }
      
      if (content.length > 5000) {
        return callback?.({ error: 'Message too long' });
      }
      
      // บันทึกลง database
      const message = await prisma.message.create({
        data: {
          content: content.trim(),
          type,
          roomId,
          userId
        },
        include: {
          user: { select: { id: true, username: true, avatar: true } }
        }
      });
      
      // ส่งให้ทุกคนในห้อง
      io.to(`room:${roomId}`).emit('chat:newMessage', {
        id: message.id,
        content: message.content,
        type: message.type,
        user: message.user,
        roomId: message.roomId,
        createdAt: message.createdAt
      });
      
      callback?.({ success: true, messageId: message.id });
    });
    
    // === LEAVE ROOM ===
    socket.on('chat:leave', async ({ roomId }) => {
      socket.leave(`room:${roomId}`);
      await redisClient.sRem(`room:${roomId}:online`, userId);
      
      socket.to(`room:${roomId}`).emit('chat:userLeft', { userId });
    });
    
    // === DISCONNECT ===
    socket.on('disconnect', async () => {
      // ลบออกจาก online users ทุก rooms
      const rooms = Array.from(socket.rooms);
      for (const room of rooms) {
        if (room.startsWith('room:')) {
          const roomId = room.replace('room:', '');
          await redisClient.sRem(`room:${roomId}:online`, userId);
          socket.to(room).emit('chat:userLeft', { userId });
        }
      }
    });
  });
}
```

### 4.2 Live Notifications

```typescript
// server/features/notifications.ts
import { Server } from 'socket.io';
import { createClient } from 'redis';

export class NotificationService {
  constructor(
    private io: Server,
    private redis: ReturnType<typeof createClient>
  ) {}
  
  // ส่ง notification ให้ user คนเดียว
  async sendToUser(userId: string, notification: {
    type: string;
    title: string;
    body: string;
    data?: any;
  }) {
    const payload = {
      ...notification,
      id: generateId(),
      timestamp: new Date()
    };
    
    // ส่งผ่าน WebSocket (ถ้า online)
    this.io.to(`user:${userId}`).emit('notification', payload);
    
    // บันทึกลง Redis สำหรับ offline users
    await this.redis.lPush(
      `notifications:${userId}`,
      JSON.stringify(payload)
    );
    await this.redis.lTrim(`notifications:${userId}`, 0, 99); // เก็บ 100 ล่าสุด
    await this.redis.expire(`notifications:${userId}`, 86400 * 7); // 7 วัน
  }
  
  // ส่งให้ทุก users ใน role
  async sendToRole(role: string, notification: any) {
    this.io.to(`role:${role}`).emit('notification', notification);
  }
  
  // ส่งให้ทุกคน
  async broadcast(notification: any) {
    this.io.emit('notification', notification);
  }
  
  // โหลด unread notifications เมื่อ user กลับมา online
  async getUnreadNotifications(userId: string) {
    const raw = await this.redis.lRange(`notifications:${userId}`, 0, -1);
    return raw.map(n => JSON.parse(n));
  }
}

// ใช้งาน
const notificationService = new NotificationService(io, redisClient);

// ส่ง notification เมื่อมี order ใหม่
async function onNewOrder(order: Order) {
  await notificationService.sendToUser(order.userId, {
    type: 'order_created',
    title: 'Order Confirmed',
    body: `Your order #${order.id} has been confirmed`,
    data: { orderId: order.id }
  });
  
  // แจ้ง admins
  await notificationService.sendToRole('admin', {
    type: 'new_order',
    title: 'New Order',
    body: `Order #${order.id} received from customer`,
    data: { orderId: order.id }
  });
}
```

### 4.3 Real-time Dashboard

```typescript
// server/features/dashboard.ts
import { Server } from 'socket.io';
import { PrismaClient } from '@prisma/client';

export function setupDashboard(io: Server, prisma: PrismaClient) {
  // Broadcast metrics ทุก 5 วินาที
  setInterval(async () => {
    try {
      const metrics = await getDashboardMetrics(prisma);
      io.to('room:dashboard').emit('dashboard:update', metrics);
    } catch (error) {
      console.error('Failed to get metrics:', error);
    }
  }, 5000);
  
  io.on('connection', (socket) => {
    socket.on('dashboard:subscribe', async () => {
      socket.join('room:dashboard');
      
      // ส่ง initial data
      const metrics = await getDashboardMetrics(prisma);
      socket.emit('dashboard:update', metrics);
    });
    
    socket.on('dashboard:unsubscribe', () => {
      socket.leave('room:dashboard');
    });
  });
}

async function getDashboardMetrics(prisma: PrismaClient) {
  const [totalUsers, activeOrders, revenue, recentActivity] = await Promise.all([
    prisma.user.count(),
    prisma.order.count({ where: { status: 'active' } }),
    prisma.order.aggregate({
      _sum: { amount: true },
      where: {
        createdAt: {
          gte: new Date(Date.now() - 24 * 60 * 60 * 1000)
        }
      }
    }),
    prisma.activityLog.findMany({
      orderBy: { createdAt: 'desc' },
      take: 10
    })
  ]);
  
  return {
    totalUsers,
    activeOrders,
    dailyRevenue: revenue._sum.amount || 0,
    recentActivity,
    timestamp: new Date()
  };
}
```

### 4.4 Presence: Who's Online

```typescript
// server/features/presence.ts
import { Server, Socket } from 'socket.io';
import { createClient } from 'redis';

export class PresenceManager {
  private PRESENCE_TTL = 60; // seconds
  
  constructor(
    private io: Server,
    private redis: ReturnType<typeof createClient>
  ) {
    this.setupHeartbeat();
  }
  
  async setOnline(userId: string, metadata: {
    username: string;
    status?: 'online' | 'away' | 'busy';
  }) {
    const key = `presence:${userId}`;
    await this.redis.setEx(key, this.PRESENCE_TTL, JSON.stringify({
      ...metadata,
      userId,
      lastSeen: new Date()
    }));
    
    // แจ้ง friends ว่า online
    this.io.emit('presence:online', { userId, ...metadata });
  }
  
  async setOffline(userId: string) {
    await this.redis.del(`presence:${userId}`);
    this.io.emit('presence:offline', { userId, timestamp: new Date() });
  }
  
  async getOnlineUsers(userIds: string[]) {
    if (userIds.length === 0) return [];
    
    const pipeline = this.redis.multi();
    userIds.forEach(id => pipeline.get(`presence:${id}`));
    const results = await pipeline.exec();
    
    return results
      .map((result, i) => result ? JSON.parse(result as string) : null)
      .filter(Boolean);
  }
  
  async updateStatus(userId: string, status: 'online' | 'away' | 'busy') {
    const key = `presence:${userId}`;
    const data = await this.redis.get(key);
    
    if (data) {
      const parsed = JSON.parse(data);
      parsed.status = status;
      await this.redis.setEx(key, this.PRESENCE_TTL, JSON.stringify(parsed));
      
      this.io.emit('presence:statusChange', { userId, status });
    }
  }
  
  // Heartbeat: refresh TTL ทุก 30 วินาที
  private setupHeartbeat() {
    this.io.on('connection', (socket: Socket) => {
      const userId = (socket as any).userId;
      
      const heartbeatInterval = setInterval(async () => {
        const key = `presence:${userId}`;
        await this.redis.expire(key, this.PRESENCE_TTL);
      }, 30000);
      
      socket.on('disconnect', () => {
        clearInterval(heartbeatInterval);
        this.setOffline(userId);
      });
    });
  }
}
```

### 4.5 Typing Indicators

```typescript
// server/features/typing.ts
import { Server, Socket } from 'socket.io';
import { createClient } from 'redis';

export function setupTypingIndicators(io: Server, redis: ReturnType<typeof createClient>) {
  io.on('connection', (socket: Socket) => {
    const userId = (socket as any).userId as string;
    
    // User เริ่มพิมพ์
    socket.on('typing:start', async ({ roomId }) => {
      // เก็บใน Redis ด้วย TTL 5 วินาที
      await redis.setEx(`typing:${roomId}:${userId}`, 5, '1');
      
      // ส่งให้ user อื่นในห้อง
      socket.to(`room:${roomId}`).emit('typing:update', {
        roomId,
        userId,
        isTyping: true
      });
    });
    
    // User หยุดพิมพ์
    socket.on('typing:stop', async ({ roomId }) => {
      await redis.del(`typing:${roomId}:${userId}`);
      
      socket.to(`room:${roomId}`).emit('typing:update', {
        roomId,
        userId,
        isTyping: false
      });
    });
    
    // ดูว่าใครกำลังพิมพ์อยู่
    socket.on('typing:getStatus', async ({ roomId }, callback) => {
      const keys = await redis.keys(`typing:${roomId}:*`);
      const typingUserIds = keys.map(k => k.split(':')[2]);
      callback({ typingUsers: typingUserIds });
    });
  });
}

// Frontend component
/*
function TypingIndicator({ roomId }: { roomId: string }) {
  const [typingUsers, setTypingUsers] = useState<string[]>([]);
  
  useEffect(() => {
    socket.on('typing:update', ({ roomId: r, userId, isTyping }) => {
      if (r !== roomId) return;
      
      setTypingUsers(prev => {
        if (isTyping && !prev.includes(userId)) {
          return [...prev, userId];
        } else if (!isTyping) {
          return prev.filter(id => id !== userId);
        }
        return prev;
      });
    });
    
    return () => {
      socket.off('typing:update');
    };
  }, [roomId]);
  
  const handleInputChange = (value: string) => {
    if (value.length > 0) {
      socket.emit('typing:start', { roomId });
      resetTypingTimeout();
    } else {
      socket.emit('typing:stop', { roomId });
    }
  };
  
  // Auto-stop typing after 3 seconds of inactivity
  let typingTimeout: NodeJS.Timeout;
  const resetTypingTimeout = () => {
    clearTimeout(typingTimeout);
    typingTimeout = setTimeout(() => {
      socket.emit('typing:stop', { roomId });
    }, 3000);
  };
  
  if (typingUsers.length === 0) return null;
  
  return (
    <div className="typing-indicator">
      {typingUsers.length === 1
        ? `${typingUsers[0]} is typing...`
        : `${typingUsers.length} people are typing...`
      }
    </div>
  );
}
*/
```

### 4.6 Read Receipts

```typescript
// server/features/readReceipts.ts
import { Server, Socket } from 'socket.io';
import { PrismaClient } from '@prisma/client';

export function setupReadReceipts(io: Server, prisma: PrismaClient) {
  io.on('connection', (socket: Socket) => {
    const userId = (socket as any).userId as string;
    
    // Mark message as read
    socket.on('message:read', async ({ messageId, roomId }, callback) => {
      try {
        // บันทึก read receipt
        await prisma.messageRead.upsert({
          where: {
            messageId_userId: { messageId, userId }
          },
          create: { messageId, userId, readAt: new Date() },
          update: { readAt: new Date() }
        });
        
        // Update last read ของ user ใน room นั้น
        await prisma.roomMember.update({
          where: { roomId_userId: { roomId, userId } },
          data: { lastReadMessageId: messageId, lastReadAt: new Date() }
        });
        
        // แจ้ง users อื่นในห้อง (รวม sender ต้นฉบับ)
        io.to(`room:${roomId}`).emit('message:readUpdate', {
          messageId,
          userId,
          readAt: new Date()
        });
        
        callback?.({ success: true });
      } catch (error) {
        callback?.({ error: 'Failed to mark as read' });
      }
    });
    
    // Mark multiple messages as read (batch)
    socket.on('messages:readBatch', async ({ roomId, messageIds }) => {
      const now = new Date();
      
      await prisma.messageRead.createMany({
        data: messageIds.map((messageId: string) => ({
          messageId,
          userId,
          readAt: now
        })),
        skipDuplicates: true
      });
      
      io.to(`room:${roomId}`).emit('messages:readBatchUpdate', {
        messageIds,
        userId,
        readAt: now
      });
    });
    
    // ดู read receipts ของ message
    socket.on('message:getReads', async ({ messageId }, callback) => {
      const reads = await prisma.messageRead.findMany({
        where: { messageId },
        include: {
          user: { select: { id: true, username: true, avatar: true } }
        }
      });
      
      callback({ reads });
    });
  });
}
```

---

## 5. Room Management ใน PostgreSQL

```sql
-- Database schema สำหรับ chat rooms

CREATE TABLE rooms (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    type VARCHAR(50) NOT NULL DEFAULT 'public', -- public, private, direct
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE room_members (
    room_id UUID REFERENCES rooms(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    role VARCHAR(50) DEFAULT 'member', -- admin, moderator, member
    joined_at TIMESTAMPTZ DEFAULT NOW(),
    last_read_message_id UUID,
    last_read_at TIMESTAMPTZ,
    notifications_enabled BOOLEAN DEFAULT true,
    PRIMARY KEY (room_id, user_id)
);

CREATE TABLE messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    content TEXT NOT NULL,
    type VARCHAR(50) DEFAULT 'text', -- text, image, file, system
    room_id UUID REFERENCES rooms(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id),
    reply_to_id UUID REFERENCES messages(id),
    edited_at TIMESTAMPTZ,
    deleted_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE message_reads (
    message_id UUID REFERENCES messages(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    read_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (message_id, user_id)
);

-- Indexes
CREATE INDEX idx_messages_room_id ON messages(room_id, created_at DESC);
CREATE INDEX idx_messages_user_id ON messages(user_id);
CREATE INDEX idx_room_members_user_id ON room_members(user_id);
CREATE INDEX idx_message_reads_message_id ON message_reads(message_id);
```

```typescript
// server/services/roomService.ts
import { PrismaClient } from '@prisma/client';

export class RoomService {
  constructor(private prisma: PrismaClient) {}
  
  async createRoom(data: {
    name: string;
    description?: string;
    type: 'public' | 'private' | 'direct';
    createdBy: string;
    memberIds?: string[];
  }) {
    const room = await this.prisma.room.create({
      data: {
        name: data.name,
        description: data.description,
        type: data.type,
        createdBy: data.createdBy,
        members: {
          create: [
            // Creator เป็น admin
            { userId: data.createdBy, role: 'admin' },
            // Members อื่นๆ
            ...(data.memberIds || []).map(userId => ({
              userId,
              role: 'member' as const
            }))
          ]
        }
      },
      include: {
        members: { include: { user: true } }
      }
    });
    
    return room;
  }
  
  async getRoomsForUser(userId: string) {
    return this.prisma.room.findMany({
      where: {
        members: { some: { userId } }
      },
      include: {
        members: {
          include: {
            user: { select: { id: true, username: true, avatar: true } }
          }
        },
        messages: {
          orderBy: { createdAt: 'desc' },
          take: 1,
          include: {
            user: { select: { username: true } }
          }
        },
        _count: { select: { members: true } }
      },
      orderBy: {
        updatedAt: 'desc'
      }
    });
  }
  
  async getUnreadCount(userId: string, roomId: string) {
    const member = await this.prisma.roomMember.findUnique({
      where: { roomId_userId: { roomId, userId } }
    });
    
    if (!member?.lastReadAt) {
      return this.prisma.message.count({
        where: { roomId, userId: { not: userId } }
      });
    }
    
    return this.prisma.message.count({
      where: {
        roomId,
        userId: { not: userId },
        createdAt: { gt: member.lastReadAt }
      }
    });
  }
}
```

---

## 6. Reconnection Handling

### 6.1 Client-side Reconnection

```typescript
// client/useSocket.ts
import { useEffect, useRef, useState } from 'react';
import { Socket } from 'socket.io-client';

interface SocketState {
  connected: boolean;
  reconnecting: boolean;
  error: string | null;
  latency: number;
}

export function useSocket(socket: Socket) {
  const [state, setState] = useState<SocketState>({
    connected: socket.connected,
    reconnecting: false,
    error: null,
    latency: 0
  });
  
  // เก็บ state ที่ต้อง restore หลัง reconnect
  const pendingRooms = useRef<string[]>([]);
  const pendingSubscriptions = useRef<string[]>([]);
  
  useEffect(() => {
    const onConnect = () => {
      setState(prev => ({
        ...prev,
        connected: true,
        reconnecting: false,
        error: null
      }));
      
      // Restore state หลัง reconnect
      restoreState();
    };
    
    const onDisconnect = (reason: string) => {
      setState(prev => ({
        ...prev,
        connected: false,
        reconnecting: reason !== 'io client disconnect'
      }));
    };
    
    const onReconnect = () => {
      setState(prev => ({ ...prev, reconnecting: true }));
    };
    
    const onError = (error: Error) => {
      setState(prev => ({ ...prev, error: error.message }));
    };
    
    socket.on('connect', onConnect);
    socket.on('disconnect', onDisconnect);
    socket.on('reconnect_attempt', onReconnect);
    socket.on('connect_error', onError);
    
    // Measure latency
    const pingInterval = setInterval(() => {
      const start = Date.now();
      socket.emit('ping', () => {
        setState(prev => ({ ...prev, latency: Date.now() - start }));
      });
    }, 30000);
    
    return () => {
      socket.off('connect', onConnect);
      socket.off('disconnect', onDisconnect);
      socket.off('reconnect_attempt', onReconnect);
      socket.off('connect_error', onError);
      clearInterval(pingInterval);
    };
  }, [socket]);
  
  // Restore state: re-join rooms, re-subscribe
  async function restoreState() {
    // Re-join rooms
    for (const roomId of pendingRooms.current) {
      socket.emit('chat:join', { roomId }, () => {});
    }
    
    // Re-subscribe to channels
    for (const channel of pendingSubscriptions.current) {
      socket.emit('subscribe', { channel });
    }
    
    // โหลด missed messages
    await loadMissedMessages();
  }
  
  async function loadMissedMessages() {
    // ดึงข้อความที่พลาดไประหว่าง offline
    const lastMessageTime = localStorage.getItem('lastMessageTime');
    
    if (lastMessageTime) {
      socket.emit('getMissedMessages', {
        since: lastMessageTime
      }, (messages: any[]) => {
        // Process missed messages
        messages.forEach(msg => {
          window.dispatchEvent(new CustomEvent('missedMessage', { detail: msg }));
        });
      });
    }
  }
  
  const joinRoom = (roomId: string) => {
    if (!pendingRooms.current.includes(roomId)) {
      pendingRooms.current.push(roomId);
    }
  };
  
  const leaveRoom = (roomId: string) => {
    pendingRooms.current = pendingRooms.current.filter(r => r !== roomId);
  };
  
  return { ...state, joinRoom, leaveRoom };
}
```

### 6.2 Server-side: Missed Messages

```typescript
// server/handlers/missedMessages.ts
socket.on('getMissedMessages', async ({ since }: { since: string }, callback) => {
  try {
    const userId = (socket as any).userId as string;
    const sinceDate = new Date(since);
    
    // ดึงห้องที่ user อยู่
    const userRooms = await prisma.roomMember.findMany({
      where: { userId },
      select: { roomId: true }
    });
    
    const roomIds = userRooms.map(r => r.roomId);
    
    // ดึงข้อความที่พลาดไป
    const missedMessages = await prisma.message.findMany({
      where: {
        roomId: { in: roomIds },
        createdAt: { gt: sinceDate },
        userId: { not: userId } // ข้อความของคนอื่น
      },
      include: {
        user: { select: { id: true, username: true, avatar: true } }
      },
      orderBy: { createdAt: 'asc' },
      take: 200 // limit
    });
    
    callback(missedMessages);
  } catch (error) {
    callback([]);
  }
});
```

---

## 7. Full Working Example: Real-time Chat Application

### 7.1 Project Structure

```
chat-app/
├── server/
│   ├── src/
│   │   ├── index.ts              # Entry point
│   │   ├── socket/
│   │   │   ├── index.ts          # Socket.io setup
│   │   │   ├── middleware.ts     # Auth middleware
│   │   │   └── handlers/
│   │   │       ├── chat.ts
│   │   │       ├── presence.ts
│   │   │       └── notifications.ts
│   │   ├── services/
│   │   │   ├── roomService.ts
│   │   │   └── messageService.ts
│   │   └── redis/
│   │       └── client.ts
│   ├── prisma/
│   │   └── schema.prisma
│   └── package.json
└── client/
    ├── src/
    │   ├── App.tsx
    │   ├── socket.ts
    │   └── components/
    │       ├── ChatRoom.tsx
    │       └── MessageList.tsx
    └── package.json
```

### 7.2 Complete Server Implementation

```typescript
// server/src/index.ts
import express from 'express';
import { createServer } from 'http';
import { Server } from 'socket.io';
import { createClient } from 'redis';
import { createAdapter } from '@socket.io/redis-adapter';
import { PrismaClient } from '@prisma/client';
import cors from 'cors';
import jwt from 'jsonwebtoken';

const app = express();
const httpServer = createServer(app);
const prisma = new PrismaClient();

// Express middleware
app.use(cors({ origin: process.env.FRONTEND_URL, credentials: true }));
app.use(express.json());

// Redis clients
const pubClient = createClient({ url: process.env.REDIS_URL });
const subClient = pubClient.duplicate();

// Socket.io server
const io = new Server(httpServer, {
  cors: { origin: process.env.FRONTEND_URL, credentials: true },
  pingTimeout: 60000,
  pingInterval: 25000
});

// Health check
app.get('/health', (req, res) => res.json({ status: 'ok' }));

// REST API สำหรับ rooms
app.get('/api/rooms', async (req, res) => {
  try {
    const userId = getUserIdFromToken(req.headers.authorization);
    const rooms = await prisma.room.findMany({
      where: { members: { some: { userId } } },
      include: {
        _count: { select: { members: true } },
        messages: {
          orderBy: { createdAt: 'desc' },
          take: 1
        }
      }
    });
    res.json(rooms);
  } catch (error) {
    res.status(500).json({ error: 'Failed to fetch rooms' });
  }
});

app.post('/api/rooms', async (req, res) => {
  try {
    const userId = getUserIdFromToken(req.headers.authorization);
    const { name, description, type, memberIds } = req.body;
    
    const room = await prisma.room.create({
      data: {
        name,
        description,
        type: type || 'public',
        createdBy: userId,
        members: {
          create: [
            { userId, role: 'admin' },
            ...(memberIds || []).map((id: string) => ({ userId: id, role: 'member' }))
          ]
        }
      }
    });
    
    res.status(201).json(room);
  } catch (error) {
    res.status(500).json({ error: 'Failed to create room' });
  }
});

// Socket.io Authentication Middleware
io.use(async (socket, next) => {
  const token = socket.handshake.auth.token;
  
  if (!token) {
    return next(new Error('Authentication required'));
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET!) as {
      userId: string;
      username: string;
    };
    
    (socket as any).userId = decoded.userId;
    (socket as any).username = decoded.username;
    
    next();
  } catch {
    next(new Error('Invalid token'));
  }
});

// Connection handler
io.on('connection', async (socket) => {
  const userId = (socket as any).userId as string;
  const username = (socket as any).username as string;
  
  console.log(`✅ User connected: ${username} (${socket.id})`);
  
  // Join user's personal room
  socket.join(`user:${userId}`);
  
  // Mark as online
  await pubClient.setEx(`online:${userId}`, 300, JSON.stringify({
    userId,
    username,
    socketId: socket.id,
    connectedAt: new Date()
  }));
  
  // Broadcast online status
  io.emit('user:online', { userId, username });
  
  // === ROOM EVENTS ===
  
  // Join room
  socket.on('room:join', async ({ roomId }, callback) => {
    try {
      const member = await prisma.roomMember.findUnique({
        where: { roomId_userId: { roomId, userId } }
      });
      
      if (!member) {
        return callback?.({ error: 'Not a member' });
      }
      
      socket.join(`room:${roomId}`);
      
      // ดึงประวัติ 50 ข้อความล่าสุด
      const history = await prisma.message.findMany({
        where: { roomId, deletedAt: null },
        include: { user: { select: { id: true, username: true } } },
        orderBy: { createdAt: 'desc' },
        take: 50
      });
      
      // Online users
      const sockets = await io.in(`room:${roomId}`).fetchSockets();
      const onlineUserIds = [...new Set(sockets.map(s => (s as any).userId))];
      
      callback?.({
        success: true,
        history: history.reverse(),
        onlineUsers: onlineUserIds
      });
      
      // แจ้งคนอื่นในห้อง
      socket.to(`room:${roomId}`).emit('room:userJoined', { userId, username });
      
    } catch (error) {
      callback?.({ error: 'Failed to join room' });
    }
  });
  
  // Leave room
  socket.on('room:leave', ({ roomId }) => {
    socket.leave(`room:${roomId}`);
    socket.to(`room:${roomId}`).emit('room:userLeft', { userId, username });
  });
  
  // === MESSAGE EVENTS ===
  
  // Send message
  socket.on('message:send', async ({ roomId, content, replyToId }, callback) => {
    if (!content?.trim() || content.length > 10000) {
      return callback?.({ error: 'Invalid message content' });
    }
    
    try {
      const message = await prisma.message.create({
        data: {
          content: content.trim(),
          roomId,
          userId,
          replyToId: replyToId || null
        },
        include: {
          user: { select: { id: true, username: true } },
          replyTo: {
            include: { user: { select: { username: true } } }
          }
        }
      });
      
      // อัปเดต room updatedAt
      await prisma.room.update({
        where: { id: roomId },
        data: { updatedAt: new Date() }
      });
      
      // Broadcast ไปยังทุกคนในห้อง
      io.to(`room:${roomId}`).emit('message:new', {
        id: message.id,
        content: message.content,
        user: message.user,
        roomId: message.roomId,
        replyTo: message.replyTo,
        createdAt: message.createdAt
      });
      
      callback?.({ success: true, messageId: message.id });
      
    } catch (error) {
      callback?.({ error: 'Failed to send message' });
    }
  });
  
  // Edit message
  socket.on('message:edit', async ({ messageId, content }, callback) => {
    try {
      const message = await prisma.message.findUnique({
        where: { id: messageId }
      });
      
      if (!message || message.userId !== userId) {
        return callback?.({ error: 'Unauthorized' });
      }
      
      const updated = await prisma.message.update({
        where: { id: messageId },
        data: { content: content.trim(), editedAt: new Date() }
      });
      
      io.to(`room:${message.roomId}`).emit('message:edited', {
        messageId,
        content: updated.content,
        editedAt: updated.editedAt
      });
      
      callback?.({ success: true });
    } catch (error) {
      callback?.({ error: 'Failed to edit message' });
    }
  });
  
  // Delete message
  socket.on('message:delete', async ({ messageId }, callback) => {
    try {
      const message = await prisma.message.findUnique({
        where: { id: messageId }
      });
      
      if (!message || message.userId !== userId) {
        return callback?.({ error: 'Unauthorized' });
      }
      
      await prisma.message.update({
        where: { id: messageId },
        data: { deletedAt: new Date(), content: '[Message deleted]' }
      });
      
      io.to(`room:${message.roomId}`).emit('message:deleted', { messageId });
      
      callback?.({ success: true });
    } catch (error) {
      callback?.({ error: 'Failed to delete message' });
    }
  });
  
  // === TYPING INDICATORS ===
  
  socket.on('typing:start', ({ roomId }) => {
    socket.to(`room:${roomId}`).emit('typing:update', {
      userId, username, isTyping: true
    });
  });
  
  socket.on('typing:stop', ({ roomId }) => {
    socket.to(`room:${roomId}`).emit('typing:update', {
      userId, username, isTyping: false
    });
  });
  
  // === READ RECEIPTS ===
  
  socket.on('message:read', async ({ messageId, roomId }) => {
    try {
      await prisma.messageRead.upsert({
        where: { messageId_userId: { messageId, userId } },
        create: { messageId, userId },
        update: { readAt: new Date() }
      });
      
      io.to(`room:${roomId}`).emit('message:readBy', {
        messageId,
        userId,
        username
      });
    } catch (error) {
      console.error('Read receipt error:', error);
    }
  });
  
  // === DISCONNECT ===
  
  socket.on('disconnect', async (reason) => {
    console.log(`❌ User disconnected: ${username} (${reason})`);
    
    // Mark as offline
    await pubClient.del(`online:${userId}`);
    
    // แจ้ง rooms ที่ user อยู่
    const roomIds = Array.from(socket.rooms)
      .filter(r => r.startsWith('room:'))
      .map(r => r.replace('room:', ''));
    
    for (const roomId of roomIds) {
      io.to(`room:${roomId}`).emit('room:userLeft', { userId, username });
    }
    
    io.emit('user:offline', { userId, username });
  });
});

// Start server
async function main() {
  await Promise.all([pubClient.connect(), subClient.connect()]);
  io.adapter(createAdapter(pubClient, subClient));
  
  const PORT = process.env.PORT || 8080;
  httpServer.listen(PORT, () => {
    console.log(`🚀 Chat server running on port ${PORT}`);
  });
}

main().catch(console.error);

function getUserIdFromToken(authHeader?: string): string {
  const token = authHeader?.split(' ')[1];
  if (!token) throw new Error('No token');
  const decoded = jwt.verify(token, process.env.JWT_SECRET!) as { userId: string };
  return decoded.userId;
}
```

### 7.3 Prisma Schema

```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        String   @id @default(uuid())
  username  String   @unique
  email     String   @unique
  avatar    String?
  createdAt DateTime @default(now())
  
  rooms        RoomMember[]
  messages     Message[]
  messageReads MessageRead[]
  
  @@map("users")
}

model Room {
  id          String   @id @default(uuid())
  name        String
  description String?
  type        String   @default("public")
  createdBy   String
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  
  members  RoomMember[]
  messages Message[]
  
  @@map("rooms")
}

model RoomMember {
  roomId              String
  userId              String
  role                String   @default("member")
  joinedAt            DateTime @default(now())
  lastReadMessageId   String?
  lastReadAt          DateTime?
  notificationsEnabled Boolean @default(true)
  
  room Room @relation(fields: [roomId], references: [id], onDelete: Cascade)
  user User @relation(fields: [userId], references: [id], onDelete: Cascade)
  
  @@id([roomId, userId])
  @@map("room_members")
}

model Message {
  id        String    @id @default(uuid())
  content   String
  type      String    @default("text")
  roomId    String
  userId    String
  replyToId String?
  editedAt  DateTime?
  deletedAt DateTime?
  createdAt DateTime  @default(now())
  
  room    Room          @relation(fields: [roomId], references: [id], onDelete: Cascade)
  user    User          @relation(fields: [userId], references: [id])
  replyTo Message?      @relation("replies", fields: [replyToId], references: [id])
  replies Message[]     @relation("replies")
  reads   MessageRead[]
  
  @@index([roomId, createdAt(sort: Desc)])
  @@map("messages")
}

model MessageRead {
  messageId String
  userId    String
  readAt    DateTime @default(now())
  
  message Message @relation(fields: [messageId], references: [id], onDelete: Cascade)
  user    User    @relation(fields: [userId], references: [id], onDelete: Cascade)
  
  @@id([messageId, userId])
  @@map("message_reads")
}
```

### 7.4 Frontend Client

```typescript
// client/src/components/ChatRoom.tsx
import React, { useEffect, useState, useRef } from 'react';
import socket from '../socket';

interface Message {
  id: string;
  content: string;
  user: { id: string; username: string };
  roomId: string;
  createdAt: string;
}

interface Props {
  roomId: string;
  currentUserId: string;
}

export function ChatRoom({ roomId, currentUserId }: Props) {
  const [messages, setMessages] = useState<Message[]>([]);
  const [newMessage, setNewMessage] = useState('');
  const [onlineUsers, setOnlineUsers] = useState<string[]>([]);
  const [typingUsers, setTypingUsers] = useState<{userId: string; username: string}[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);
  
  const messagesEndRef = useRef<HTMLDivElement>(null);
  const typingTimeoutRef = useRef<NodeJS.Timeout>();
  const isTypingRef = useRef(false);
  
  useEffect(() => {
    // Join room
    socket.emit('room:join', { roomId }, (response: any) => {
      if (response.error) {
        setError(response.error);
      } else {
        setMessages(response.history);
        setOnlineUsers(response.onlineUsers);
        setLoading(false);
      }
    });
    
    // Event listeners
    socket.on('message:new', handleNewMessage);
    socket.on('message:edited', handleMessageEdited);
    socket.on('message:deleted', handleMessageDeleted);
    socket.on('room:userJoined', handleUserJoined);
    socket.on('room:userLeft', handleUserLeft);
    socket.on('typing:update', handleTypingUpdate);
    
    return () => {
      socket.emit('room:leave', { roomId });
      socket.off('message:new', handleNewMessage);
      socket.off('message:edited', handleMessageEdited);
      socket.off('message:deleted', handleMessageDeleted);
      socket.off('room:userJoined', handleUserJoined);
      socket.off('room:userLeft', handleUserLeft);
      socket.off('typing:update', handleTypingUpdate);
    };
  }, [roomId]);
  
  // Auto scroll
  useEffect(() => {
    messagesEndRef.current?.scrollIntoView({ behavior: 'smooth' });
  }, [messages]);
  
  const handleNewMessage = (message: Message) => {
    setMessages(prev => [...prev, message]);
    
    // Mark as read ถ้า room กำลัง active
    socket.emit('message:read', { messageId: message.id, roomId });
    
    // Save last message time
    localStorage.setItem('lastMessageTime', message.createdAt);
  };
  
  const handleMessageEdited = ({ messageId, content, editedAt }: any) => {
    setMessages(prev => prev.map(m =>
      m.id === messageId ? { ...m, content, editedAt } : m
    ));
  };
  
  const handleMessageDeleted = ({ messageId }: any) => {
    setMessages(prev => prev.map(m =>
      m.id === messageId ? { ...m, content: '[Message deleted]', deleted: true } : m
    ));
  };
  
  const handleUserJoined = ({ userId, username }: any) => {
    setOnlineUsers(prev => [...new Set([...prev, userId])]);
  };
  
  const handleUserLeft = ({ userId }: any) => {
    setOnlineUsers(prev => prev.filter(id => id !== userId));
  };
  
  const handleTypingUpdate = ({ userId, username, isTyping }: any) => {
    if (userId === currentUserId) return;
    
    setTypingUsers(prev => {
      if (isTyping && !prev.find(u => u.userId === userId)) {
        return [...prev, { userId, username }];
      } else if (!isTyping) {
        return prev.filter(u => u.userId !== userId);
      }
      return prev;
    });
  };
  
  const handleInputChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    setNewMessage(e.target.value);
    
    if (!isTypingRef.current) {
      isTypingRef.current = true;
      socket.emit('typing:start', { roomId });
    }
    
    clearTimeout(typingTimeoutRef.current);
    typingTimeoutRef.current = setTimeout(() => {
      isTypingRef.current = false;
      socket.emit('typing:stop', { roomId });
    }, 2000);
  };
  
  const sendMessage = async (e: React.FormEvent) => {
    e.preventDefault();
    if (!newMessage.trim()) return;
    
    // Stop typing indicator
    clearTimeout(typingTimeoutRef.current);
    isTypingRef.current = false;
    socket.emit('typing:stop', { roomId });
    
    const content = newMessage;
    setNewMessage('');
    
    socket.emit('message:send', { roomId, content }, (response: any) => {
      if (response.error) {
        setError(response.error);
        setNewMessage(content); // Restore message on error
      }
    });
  };
  
  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;
  
  return (
    <div className="chat-room">
      <div className="messages">
        {messages.map(message => (
          <div
            key={message.id}
            className={`message ${message.user.id === currentUserId ? 'own' : ''}`}
          >
            <span className="username">{message.user.username}</span>
            <span className="content">{message.content}</span>
            <span className="time">
              {new Date(message.createdAt).toLocaleTimeString()}
            </span>
          </div>
        ))}
        <div ref={messagesEndRef} />
      </div>
      
      {typingUsers.length > 0 && (
        <div className="typing-indicator">
          {typingUsers.map(u => u.username).join(', ')} is typing...
        </div>
      )}
      
      <form onSubmit={sendMessage} className="message-input">
        <input
          type="text"
          value={newMessage}
          onChange={handleInputChange}
          placeholder="Type a message..."
          maxLength={10000}
        />
        <button type="submit" disabled={!newMessage.trim()}>
          Send
        </button>
      </form>
    </div>
  );
}
```

---

## 8. Performance Optimization

### 8.1 Event Batching

```typescript
// แทนที่จะ emit ทุก event แยกกัน ให้รวมกันก่อนส่ง
class EventBatcher {
  private batch: any[] = [];
  private timer: NodeJS.Timeout | null = null;
  private readonly BATCH_INTERVAL = 50; // ms
  
  constructor(
    private socket: Socket,
    private event: string
  ) {}
  
  add(data: any) {
    this.batch.push(data);
    
    if (!this.timer) {
      this.timer = setTimeout(() => {
        if (this.batch.length > 0) {
          this.socket.emit(this.event, this.batch);
          this.batch = [];
        }
        this.timer = null;
      }, this.BATCH_INTERVAL);
    }
  }
}
```

### 8.2 Rate Limiting

```typescript
// server/middleware/rateLimit.ts
import { Socket } from 'socket.io';
import { createClient } from 'redis';

export function createRateLimiter(redis: ReturnType<typeof createClient>, options: {
  maxRequests: number;
  windowMs: number;
}) {
  return async function rateLimitMiddleware(socket: Socket, event: string, next: () => void) {
    const userId = (socket as any).userId;
    const key = `ratelimit:${userId}:${event}`;
    
    const current = await redis.incr(key);
    
    if (current === 1) {
      await redis.expire(key, Math.ceil(options.windowMs / 1000));
    }
    
    if (current > options.maxRequests) {
      socket.emit('error', {
        event,
        message: 'Rate limit exceeded. Try again later.'
      });
      return;
    }
    
    next();
  };
}
```

### 8.3 Compression

```typescript
const io = new Server(httpServer, {
  // Enable per-message compression
  perMessageDeflate: {
    zlibDeflateOptions: {
      chunkSize: 1024,
      memLevel: 7,
      level: 3
    },
    zlibInflateOptions: {
      chunkSize: 10 * 1024
    },
    clientNoContextTakeover: true,
    serverNoContextTakeover: true,
    serverMaxWindowBits: 10,
    concurrencyLimit: 10,
    threshold: 1024 // ไม่ compress ถ้า < 1kb
  }
});
```

---

## 9. Monitoring & Debugging

```typescript
// Prometheus metrics สำหรับ Socket.io
import { Counter, Gauge, Histogram } from 'prom-client';

const activeConnections = new Gauge({
  name: 'socketio_active_connections',
  help: 'Number of active WebSocket connections'
});

const messagesTotal = new Counter({
  name: 'socketio_messages_total',
  help: 'Total messages sent',
  labelNames: ['event', 'room']
});

const messageLatency = new Histogram({
  name: 'socketio_message_latency_seconds',
  help: 'Message processing latency',
  buckets: [0.001, 0.01, 0.05, 0.1, 0.5, 1]
});

io.on('connection', (socket) => {
  activeConnections.inc();
  
  socket.on('disconnect', () => {
    activeConnections.dec();
  });
  
  socket.onAny((event) => {
    messagesTotal.inc({ event, room: 'unknown' });
  });
});
```

---

## สรุป

WebSocket + Redis Pub/Sub เป็น combination ที่ทรงพลังสำหรับ real-time features:

1. **Socket.io** ให้ developer experience ที่ดีกว่า native WebSocket
2. **Redis Adapter** แก้ปัญหา horizontal scaling
3. **Patterns** ต่างๆ (chat, notifications, presence, typing) ล้วนสร้างบนพื้นฐานเดียวกัน
4. **Performance**: rate limiting, batching, compression เป็นสิ่งสำคัญใน production
5. **Reconnection** handling ต้องคิดให้รอบคอบเพื่อ UX ที่ดี

ในส่วนต่อไปจะพูดถึง Kafka Integration ที่เหมาะกับ event-driven architecture ขนาดใหญ่กว่า Redis Pub/Sub

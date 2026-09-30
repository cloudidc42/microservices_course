# Part 43: WebSockets และ Real-time Systems

## บทนำ

Real-time Systems คือระบบที่ตอบสนองต่อเหตุการณ์ทันทีทันใด ไม่ว่าจะเป็น Chat, Live Notifications, Dashboard แสดงข้อมูล Real-time หรือ Collaborative Editing ในบทนี้เราจะศึกษา Socket.io แบบ Cluster Mode ด้วย Redis Adapter, Server-Sent Events (SSE), การจัดการ Presence System, Room Management และ Horizontal Scaling

### สิ่งที่จะได้เรียนรู้

- Socket.io Cluster Mode ด้วย Redis Adapter
- Presence System (ใครออนไลน์อยู่)
- Room Management
- Server-Sent Events (SSE) สำหรับ One-Way Updates
- Horizontal Scaling Strategy
- Heartbeat และ Reconnection Logic
- Message Queue Integration (RabbitMQ)
- Rate Limiting และ Security

---

## 1. Architecture Overview

```
┌───────────────────────────────────────────────────────────────┐
│                        Load Balancer                           │
│                    (Nginx / Traefik)                           │
│            sticky sessions: ip_hash                           │
└──────────────────────────────────────────────────────────────┘
         │                    │                    │
         ▼                    ▼                    ▼
  ┌────────────┐       ┌────────────┐       ┌────────────┐
  │ WebSocket  │       │ WebSocket  │       │ WebSocket  │
  │ Server #1  │       │ Server #2  │       │ Server #3  │
  └────────────┘       └────────────┘       └────────────┘
         │                    │                    │
         └────────────────────┼────────────────────┘
                              │
                    ┌─────────▼──────────┐
                    │   Redis Adapter    │
                    │   (Pub/Sub)        │
                    │   + Presence Store │
                    └────────────────────┘
                              │
                    ┌─────────▼──────────┐
                    │     RabbitMQ       │
                    │   Message Queue    │
                    └────────────────────┘
```

---

## 2. Socket.io Server ด้วย Redis Adapter

```typescript
// src/server.ts
import express from 'express';
import http from 'http';
import { Server, Socket } from 'socket.io';
import { createAdapter } from '@socket.io/redis-adapter';
import { createClient } from 'redis';
import helmet from 'helmet';
import rateLimit from 'express-rate-limit';
import { AuthMiddleware } from './middleware/auth.middleware';
import { PresenceService } from './services/presence.service';
import { RoomService } from './services/room.service';
import { MessageService } from './services/message.service';
import { ChatHandler } from './handlers/chat.handler';
import { NotificationHandler } from './handlers/notification.handler';

async function bootstrap() {
  const app = express();

  app.use(helmet());
  app.use(express.json());

  // Rate limiting for REST endpoints
  app.use(
    '/api',
    rateLimit({
      windowMs: 15 * 60 * 1000,
      max: 100,
      standardHeaders: true,
      legacyHeaders: false,
    })
  );

  const httpServer = http.createServer(app);

  // Redis clients for adapter (pub/sub require separate connections)
  const pubClient = createClient({
    url: process.env.REDIS_URL || 'redis://localhost:6379',
    socket: {
      reconnectStrategy: (retries) => Math.min(retries * 50, 2000),
    },
  });

  const subClient = pubClient.duplicate();

  await Promise.all([pubClient.connect(), subClient.connect()]);

  // Socket.io server
  const io = new Server(httpServer, {
    adapter: createAdapter(pubClient, subClient),
    cors: {
      origin: process.env.ALLOWED_ORIGINS?.split(',') || ['http://localhost:3000'],
      credentials: true,
    },
    // Connection state recovery
    connectionStateRecovery: {
      maxDisconnectionDuration: 2 * 60 * 1000, // 2 minutes
      skipMiddlewares: true,
    },
    // Ping interval and timeout for heartbeat
    pingInterval: 25000,
    pingTimeout: 20000,
    // Transports
    transports: ['websocket', 'polling'],
    // Allow upgrades from polling to websocket
    allowUpgrades: true,
    // Per-message deflate for compression
    perMessageDeflate: {
      threshold: 1024, // Only compress messages > 1KB
    },
  });

  // Services
  const presenceService = new PresenceService(pubClient);
  const roomService = new RoomService(pubClient);
  const messageService = new MessageService();

  // Authentication middleware
  io.use(AuthMiddleware.authenticate);

  // Rate limiting middleware
  io.use(rateLimitMiddleware());

  // Connection handler
  io.on('connection', (socket: Socket) => {
    console.log(`Client connected: ${socket.id}, user: ${(socket as any).user?.id}`);

    // Initialize handlers
    new ChatHandler(io, socket, presenceService, roomService, messageService);
    new NotificationHandler(io, socket, presenceService);

    // Handle disconnection
    socket.on('disconnect', async (reason) => {
      console.log(`Client disconnected: ${socket.id}, reason: ${reason}`);
      await presenceService.setOffline((socket as any).user?.id);
    });

    // Handle errors
    socket.on('error', (error) => {
      console.error(`Socket error for ${socket.id}:`, error);
    });
  });

  // Health check endpoint
  app.get('/health', async (req, res) => {
    const redisOk = await pubClient.ping().then(() => true).catch(() => false);
    res.json({
      status: redisOk ? 'ok' : 'degraded',
      connections: io.engine.clientsCount,
      timestamp: new Date().toISOString(),
    });
  });

  // Metrics endpoint
  app.get('/metrics', async (req, res) => {
    const totalConnections = io.engine.clientsCount;
    const rooms = io.sockets.adapter.rooms;

    res.json({
      totalConnections,
      totalRooms: rooms.size,
    });
  });

  const PORT = process.env.PORT || 3000;
  httpServer.listen(PORT, () => {
    console.log(`WebSocket server running on port ${PORT}`);
  });

  // Graceful shutdown
  process.on('SIGTERM', async () => {
    console.log('Shutting down gracefully...');
    await new Promise<void>((resolve) => httpServer.close(() => resolve()));
    await pubClient.quit();
    await subClient.quit();
    process.exit(0);
  });

  return { io, httpServer };
}

function rateLimitMiddleware() {
  const connections = new Map<string, { count: number; resetAt: number }>();

  return (socket: Socket, next: (err?: Error) => void) => {
    const userId = (socket as any).user?.id || socket.handshake.address;
    const now = Date.now();
    const windowMs = 60 * 1000;
    const maxConnections = 5;

    const current = connections.get(userId);

    if (!current || current.resetAt < now) {
      connections.set(userId, { count: 1, resetAt: now + windowMs });
      return next();
    }

    if (current.count >= maxConnections) {
      return next(new Error('Too many connections'));
    }

    current.count++;
    next();
  };
}

bootstrap().catch(console.error);
```

---

## 3. Authentication Middleware

```typescript
// src/middleware/auth.middleware.ts
import { Socket } from 'socket.io';
import jwt from 'jsonwebtoken';

export class AuthMiddleware {
  static authenticate(socket: Socket, next: (err?: Error) => void) {
    const token =
      socket.handshake.auth.token ||
      socket.handshake.headers.authorization?.replace('Bearer ', '');

    if (!token) {
      return next(new Error('Authentication token required'));
    }

    try {
      const payload = jwt.verify(
        token,
        process.env.JWT_SECRET!
      ) as jwt.JwtPayload;

      // Attach user to socket
      (socket as any).user = {
        id: payload.sub,
        email: payload.email,
        role: payload.role,
        name: payload.name,
      };

      next();
    } catch (error) {
      if (error instanceof jwt.TokenExpiredError) {
        return next(new Error('Token expired'));
      }
      return next(new Error('Invalid token'));
    }
  }
}
```

---

## 4. Presence Service

```typescript
// src/services/presence.service.ts
import { RedisClientType } from 'redis';

export interface UserPresence {
  userId: string;
  status: 'online' | 'away' | 'busy' | 'offline';
  lastSeen: string;
  socketIds: string[];
  metadata?: Record<string, any>;
}

export class PresenceService {
  private readonly PREFIX = 'presence:';
  private readonly EXPIRE_SECONDS = 30; // 30 seconds TTL

  constructor(private readonly redis: RedisClientType) {}

  async setOnline(
    userId: string,
    socketId: string,
    metadata?: Record<string, any>
  ): Promise<void> {
    const key = `${this.PREFIX}${userId}`;
    const now = new Date().toISOString();

    // Get current presence
    const existing = await this.get(userId);
    const socketIds = existing?.socketIds || [];

    if (!socketIds.includes(socketId)) {
      socketIds.push(socketId);
    }

    const presence: UserPresence = {
      userId,
      status: 'online',
      lastSeen: now,
      socketIds,
      metadata,
    };

    await this.redis.setEx(key, this.EXPIRE_SECONDS, JSON.stringify(presence));

    // Publish presence change
    await this.redis.publish(
      'presence:changed',
      JSON.stringify({ userId, status: 'online', socketId })
    );
  }

  async setOffline(userId: string, socketId?: string): Promise<void> {
    const key = `${this.PREFIX}${userId}`;
    const existing = await this.get(userId);

    if (!existing) return;

    let remainingSockets = existing.socketIds;

    if (socketId) {
      remainingSockets = existing.socketIds.filter((id) => id !== socketId);
    } else {
      remainingSockets = [];
    }

    if (remainingSockets.length === 0) {
      // Truly offline - keep record for lastSeen but mark offline
      const presence: UserPresence = {
        ...existing,
        status: 'offline',
        lastSeen: new Date().toISOString(),
        socketIds: [],
      };

      // Keep for 5 minutes for lastSeen info
      await this.redis.setEx(key, 300, JSON.stringify(presence));

      await this.redis.publish(
        'presence:changed',
        JSON.stringify({ userId, status: 'offline' })
      );
    } else {
      // Still has other connections
      await this.redis.setEx(
        key,
        this.EXPIRE_SECONDS,
        JSON.stringify({ ...existing, socketIds: remainingSockets })
      );
    }
  }

  async setStatus(
    userId: string,
    status: 'online' | 'away' | 'busy'
  ): Promise<void> {
    const key = `${this.PREFIX}${userId}`;
    const existing = await this.get(userId);

    if (!existing) return;

    const updated = { ...existing, status };
    await this.redis.setEx(key, this.EXPIRE_SECONDS, JSON.stringify(updated));

    await this.redis.publish(
      'presence:changed',
      JSON.stringify({ userId, status })
    );
  }

  async get(userId: string): Promise<UserPresence | null> {
    const data = await this.redis.get(`${this.PREFIX}${userId}`);
    if (!data) return null;

    try {
      return JSON.parse(data);
    } catch {
      return null;
    }
  }

  async getMultiple(userIds: string[]): Promise<Map<string, UserPresence>> {
    if (userIds.length === 0) return new Map();

    const keys = userIds.map((id) => `${this.PREFIX}${id}`);
    const values = await this.redis.mGet(keys);

    const result = new Map<string, UserPresence>();
    userIds.forEach((userId, index) => {
      const value = values[index];
      if (value) {
        try {
          result.set(userId, JSON.parse(value));
        } catch {
          // Skip invalid entries
        }
      }
    });

    return result;
  }

  async isOnline(userId: string): Promise<boolean> {
    const presence = await this.get(userId);
    return presence?.status !== 'offline' && presence?.socketIds.length! > 0;
  }

  // Refresh TTL to keep user online (called on activity)
  async refresh(userId: string): Promise<void> {
    const key = `${this.PREFIX}${userId}`;
    await this.redis.expire(key, this.EXPIRE_SECONDS);
  }

  // Get all online users in a specific set
  async getOnlineUsersInRoom(roomId: string): Promise<string[]> {
    const members = await this.redis.sMembers(`room:${roomId}:members`);

    if (members.length === 0) return [];

    const presenceMap = await this.getMultiple(members);
    const online: string[] = [];

    presenceMap.forEach((presence, userId) => {
      if (presence.status !== 'offline' && presence.socketIds.length > 0) {
        online.push(userId);
      }
    });

    return online;
  }
}
```

---

## 5. Room Service

```typescript
// src/services/room.service.ts
import { RedisClientType } from 'redis';
import { Pool } from 'pg';

export interface Room {
  id: string;
  name: string;
  type: 'direct' | 'group' | 'channel';
  members: string[];
  createdBy: string;
  createdAt: string;
  metadata?: Record<string, any>;
}

export class RoomService {
  constructor(private readonly redis: RedisClientType) {}

  async joinRoom(userId: string, roomId: string): Promise<void> {
    const pipeline = this.redis.multi();

    // Add user to room members set
    pipeline.sAdd(`room:${roomId}:members`, userId);

    // Add room to user's rooms set
    pipeline.sAdd(`user:${userId}:rooms`, roomId);

    await pipeline.exec();

    // Publish join event
    await this.redis.publish(
      `room:${roomId}:events`,
      JSON.stringify({ type: 'user_joined', userId, roomId })
    );
  }

  async leaveRoom(userId: string, roomId: string): Promise<void> {
    const pipeline = this.redis.multi();

    pipeline.sRem(`room:${roomId}:members`, userId);
    pipeline.sRem(`user:${userId}:rooms`, roomId);

    await pipeline.exec();

    await this.redis.publish(
      `room:${roomId}:events`,
      JSON.stringify({ type: 'user_left', userId, roomId })
    );
  }

  async getRoomMembers(roomId: string): Promise<string[]> {
    return this.redis.sMembers(`room:${roomId}:members`);
  }

  async getUserRooms(userId: string): Promise<string[]> {
    return this.redis.sMembers(`user:${userId}:rooms`);
  }

  async isInRoom(userId: string, roomId: string): Promise<boolean> {
    return this.redis.sIsMember(`room:${roomId}:members`, userId);
  }

  async getRoomCount(roomId: string): Promise<number> {
    return this.redis.sCard(`room:${roomId}:members`);
  }
}
```

---

## 6. Chat Handler

```typescript
// src/handlers/chat.handler.ts
import { Server, Socket } from 'socket.io';
import { PresenceService } from '../services/presence.service';
import { RoomService } from '../services/room.service';
import { MessageService } from '../services/message.service';

interface ChatMessage {
  roomId: string;
  content: string;
  type?: 'text' | 'image' | 'file';
  replyTo?: string;
}

interface TypingEvent {
  roomId: string;
  isTyping: boolean;
}

export class ChatHandler {
  private user: { id: string; name: string; email: string };
  private typingTimeouts = new Map<string, NodeJS.Timeout>();

  constructor(
    private readonly io: Server,
    private readonly socket: Socket,
    private readonly presenceService: PresenceService,
    private readonly roomService: RoomService,
    private readonly messageService: MessageService
  ) {
    this.user = (socket as any).user;

    this.register();
    this.initializePresence();
  }

  private register(): void {
    this.socket.on('chat:join', this.handleJoinRoom.bind(this));
    this.socket.on('chat:leave', this.handleLeaveRoom.bind(this));
    this.socket.on('chat:message', this.handleMessage.bind(this));
    this.socket.on('chat:typing', this.handleTyping.bind(this));
    this.socket.on('chat:read', this.handleReadReceipt.bind(this));
    this.socket.on('chat:history', this.handleGetHistory.bind(this));
    this.socket.on('disconnect', this.handleDisconnect.bind(this));
  }

  private async initializePresence(): Promise<void> {
    await this.presenceService.setOnline(this.user.id, this.socket.id, {
      name: this.user.name,
    });

    // Rejoin previous rooms (connection state recovery)
    const userRooms = await this.roomService.getUserRooms(this.user.id);
    for (const roomId of userRooms) {
      await this.socket.join(roomId);
    }

    // Notify friends that user is online
    this.socket.broadcast.emit('presence:update', {
      userId: this.user.id,
      status: 'online',
    });
  }

  private async handleJoinRoom({ roomId }: { roomId: string }): Promise<void> {
    try {
      // Verify permission to join
      const canJoin = await this.messageService.canJoinRoom(
        this.user.id,
        roomId
      );

      if (!canJoin) {
        this.socket.emit('error', {
          event: 'chat:join',
          message: 'Not authorized to join this room',
        });
        return;
      }

      await this.socket.join(roomId);
      await this.roomService.joinRoom(this.user.id, roomId);

      const memberCount = await this.roomService.getRoomCount(roomId);

      // Notify room members
      this.io.to(roomId).emit('chat:room_joined', {
        roomId,
        user: { id: this.user.id, name: this.user.name },
        memberCount,
      });

      // Send recent history to the joiner
      const history = await this.messageService.getRecentMessages(roomId, 50);
      this.socket.emit('chat:history', { roomId, messages: history });
    } catch (error) {
      console.error('Error joining room:', error);
      this.socket.emit('error', { event: 'chat:join', message: 'Failed to join room' });
    }
  }

  private async handleLeaveRoom({ roomId }: { roomId: string }): Promise<void> {
    await this.socket.leave(roomId);
    await this.roomService.leaveRoom(this.user.id, roomId);

    const memberCount = await this.roomService.getRoomCount(roomId);

    this.io.to(roomId).emit('chat:room_left', {
      roomId,
      user: { id: this.user.id, name: this.user.name },
      memberCount,
    });
  }

  private async handleMessage(data: ChatMessage): Promise<void> {
    try {
      // Validate message
      if (!data.content?.trim() && data.type === 'text') {
        return this.socket.emit('error', {
          event: 'chat:message',
          message: 'Message content cannot be empty',
        });
      }

      if (data.content.length > 4000) {
        return this.socket.emit('error', {
          event: 'chat:message',
          message: 'Message too long (max 4000 characters)',
        });
      }

      // Check if user is in the room
      const inRoom = await this.roomService.isInRoom(this.user.id, data.roomId);
      if (!inRoom) {
        return this.socket.emit('error', {
          event: 'chat:message',
          message: 'Not in this room',
        });
      }

      // Save message to database
      const message = await this.messageService.saveMessage({
        roomId: data.roomId,
        senderId: this.user.id,
        content: data.content,
        type: data.type || 'text',
        replyTo: data.replyTo,
      });

      // Clear typing indicator
      this.clearTypingTimeout(data.roomId);

      // Broadcast to room
      this.io.to(data.roomId).emit('chat:message', {
        ...message,
        sender: {
          id: this.user.id,
          name: this.user.name,
        },
      });

      // Acknowledge to sender
      this.socket.emit('chat:message_sent', {
        messageId: message.id,
        roomId: data.roomId,
        timestamp: message.createdAt,
      });
    } catch (error) {
      console.error('Error sending message:', error);
      this.socket.emit('error', {
        event: 'chat:message',
        message: 'Failed to send message',
      });
    }
  }

  private handleTyping({ roomId, isTyping }: TypingEvent): void {
    if (isTyping) {
      // Broadcast typing indicator
      this.socket.to(roomId).emit('chat:typing', {
        roomId,
        user: { id: this.user.id, name: this.user.name },
        isTyping: true,
      });

      // Auto-clear after 3 seconds
      this.clearTypingTimeout(roomId);
      const timeout = setTimeout(() => {
        this.socket.to(roomId).emit('chat:typing', {
          roomId,
          user: { id: this.user.id, name: this.user.name },
          isTyping: false,
        });
      }, 3000);

      this.typingTimeouts.set(roomId, timeout);
    } else {
      this.clearTypingTimeout(roomId);
      this.socket.to(roomId).emit('chat:typing', {
        roomId,
        user: { id: this.user.id, name: this.user.name },
        isTyping: false,
      });
    }
  }

  private async handleReadReceipt({
    roomId,
    messageId,
  }: {
    roomId: string;
    messageId: string;
  }): Promise<void> {
    await this.messageService.markAsRead(messageId, this.user.id);

    this.socket.to(roomId).emit('chat:read', {
      roomId,
      messageId,
      userId: this.user.id,
      readAt: new Date().toISOString(),
    });
  }

  private async handleGetHistory({
    roomId,
    before,
    limit = 50,
  }: {
    roomId: string;
    before?: string;
    limit: number;
  }): Promise<void> {
    const messages = await this.messageService.getMessages(
      roomId,
      before,
      limit
    );
    this.socket.emit('chat:history', { roomId, messages });
  }

  private async handleDisconnect(): Promise<void> {
    // Clear all typing timeouts
    for (const [roomId, timeout] of this.typingTimeouts) {
      clearTimeout(timeout);
      this.socket.to(roomId).emit('chat:typing', {
        roomId,
        user: { id: this.user.id, name: this.user.name },
        isTyping: false,
      });
    }

    await this.presenceService.setOffline(this.user.id, this.socket.id);
  }

  private clearTypingTimeout(roomId: string): void {
    const existing = this.typingTimeouts.get(roomId);
    if (existing) {
      clearTimeout(existing);
      this.typingTimeouts.delete(roomId);
    }
  }
}
```

---

## 7. Message Service ด้วย PostgreSQL

```typescript
// src/services/message.service.ts
import { Pool } from 'pg';

export interface Message {
  id: string;
  roomId: string;
  senderId: string;
  content: string;
  type: 'text' | 'image' | 'file';
  replyTo?: string;
  readBy: string[];
  createdAt: string;
  updatedAt: string;
}

export class MessageService {
  constructor(private readonly pool: Pool) {}

  async saveMessage(data: {
    roomId: string;
    senderId: string;
    content: string;
    type: string;
    replyTo?: string;
  }): Promise<Message> {
    const result = await this.pool.query(
      `INSERT INTO messages (room_id, sender_id, content, type, reply_to_id)
       VALUES ($1, $2, $3, $4, $5)
       RETURNING *`,
      [data.roomId, data.senderId, data.content, data.type, data.replyTo]
    );

    return this.mapRow(result.rows[0]);
  }

  async getMessages(
    roomId: string,
    before?: string,
    limit = 50
  ): Promise<Message[]> {
    let query: string;
    let params: any[];

    if (before) {
      query = `
        SELECT m.*, array_agg(mr.user_id) FILTER (WHERE mr.user_id IS NOT NULL) as read_by
        FROM messages m
        LEFT JOIN message_reads mr ON mr.message_id = m.id
        WHERE m.room_id = $1 AND m.created_at < (
          SELECT created_at FROM messages WHERE id = $2
        )
        GROUP BY m.id
        ORDER BY m.created_at DESC
        LIMIT $3
      `;
      params = [roomId, before, limit];
    } else {
      query = `
        SELECT m.*, array_agg(mr.user_id) FILTER (WHERE mr.user_id IS NOT NULL) as read_by
        FROM messages m
        LEFT JOIN message_reads mr ON mr.message_id = m.id
        WHERE m.room_id = $1
        GROUP BY m.id
        ORDER BY m.created_at DESC
        LIMIT $2
      `;
      params = [roomId, limit];
    }

    const result = await this.pool.query(query, params);
    return result.rows.reverse().map(this.mapRow);
  }

  async getRecentMessages(roomId: string, limit: number): Promise<Message[]> {
    return this.getMessages(roomId, undefined, limit);
  }

  async markAsRead(messageId: string, userId: string): Promise<void> {
    await this.pool.query(
      `INSERT INTO message_reads (message_id, user_id, read_at)
       VALUES ($1, $2, NOW())
       ON CONFLICT (message_id, user_id) DO NOTHING`,
      [messageId, userId]
    );
  }

  async canJoinRoom(userId: string, roomId: string): Promise<boolean> {
    const result = await this.pool.query(
      `SELECT 1 FROM room_members 
       WHERE room_id = $1 AND user_id = $2 AND is_active = true`,
      [roomId, userId]
    );
    return result.rows.length > 0;
  }

  private mapRow(row: any): Message {
    return {
      id: row.id,
      roomId: row.room_id,
      senderId: row.sender_id,
      content: row.content,
      type: row.type,
      replyTo: row.reply_to_id,
      readBy: row.read_by || [],
      createdAt: row.created_at.toISOString(),
      updatedAt: row.updated_at.toISOString(),
    };
  }
}
```

---

## 8. Server-Sent Events (SSE)

```typescript
// src/sse/sse.controller.ts
import express, { Request, Response } from 'express';
import { RedisClientType } from 'redis';
import jwt from 'jsonwebtoken';

export function createSSERouter(redis: RedisClientType) {
  const router = express.Router();

  // SSE endpoint for notifications
  router.get('/notifications', async (req: Request, res: Response) => {
    const token = req.query.token as string;

    if (!token) {
      return res.status(401).json({ error: 'Token required' });
    }

    let userId: string;
    try {
      const payload = jwt.verify(token, process.env.JWT_SECRET!) as any;
      userId = payload.sub;
    } catch {
      return res.status(401).json({ error: 'Invalid token' });
    }

    // Set SSE headers
    res.setHeader('Content-Type', 'text/event-stream');
    res.setHeader('Cache-Control', 'no-cache');
    res.setHeader('Connection', 'keep-alive');
    res.setHeader('X-Accel-Buffering', 'no'); // Disable Nginx buffering
    res.flushHeaders();

    // Send initial connection event
    sendSSEEvent(res, 'connected', {
      userId,
      timestamp: new Date().toISOString(),
    });

    // Heartbeat every 30 seconds
    const heartbeatInterval = setInterval(() => {
      sendSSEEvent(res, 'heartbeat', { timestamp: new Date().toISOString() });
    }, 30000);

    // Subscribe to Redis channel for this user
    const subClient = redis.duplicate();
    await subClient.connect();

    const channel = `notifications:${userId}`;

    await subClient.subscribe(channel, (message) => {
      try {
        const data = JSON.parse(message);
        sendSSEEvent(res, data.type || 'notification', data);
      } catch {
        console.error('Failed to parse notification:', message);
      }
    });

    // Handle client disconnect
    req.on('close', async () => {
      clearInterval(heartbeatInterval);
      await subClient.unsubscribe(channel);
      await subClient.quit();
      console.log(`SSE client disconnected: ${userId}`);
    });

    // Handle errors
    res.on('error', async () => {
      clearInterval(heartbeatInterval);
      await subClient.unsubscribe(channel).catch(() => {});
      await subClient.quit().catch(() => {});
    });
  });

  // SSE for order tracking
  router.get('/orders/:orderId/track', async (req: Request, res: Response) => {
    const token = req.query.token as string;
    const { orderId } = req.params;

    if (!token) {
      return res.status(401).json({ error: 'Token required' });
    }

    let userId: string;
    try {
      const payload = jwt.verify(token, process.env.JWT_SECRET!) as any;
      userId = payload.sub;
    } catch {
      return res.status(401).json({ error: 'Invalid token' });
    }

    res.setHeader('Content-Type', 'text/event-stream');
    res.setHeader('Cache-Control', 'no-cache');
    res.setHeader('Connection', 'keep-alive');
    res.flushHeaders();

    // Send current order status immediately
    const currentStatus = await redis.get(`order:${orderId}:status`);
    if (currentStatus) {
      sendSSEEvent(res, 'order_status', JSON.parse(currentStatus));
    }

    const heartbeatInterval = setInterval(() => {
      sendSSEEvent(res, 'heartbeat', {});
    }, 15000);

    const subClient = redis.duplicate();
    await subClient.connect();

    const channel = `order:${orderId}:updates`;
    await subClient.subscribe(channel, (message) => {
      const data = JSON.parse(message);
      sendSSEEvent(res, 'order_update', data);

      // Close connection when order is delivered or cancelled
      if (
        data.status === 'DELIVERED' ||
        data.status === 'CANCELLED'
      ) {
        sendSSEEvent(res, 'stream_end', { reason: 'final_status' });
        setTimeout(() => res.end(), 1000);
      }
    });

    req.on('close', async () => {
      clearInterval(heartbeatInterval);
      await subClient.unsubscribe(channel);
      await subClient.quit();
    });
  });

  return router;
}

function sendSSEEvent(res: Response, event: string, data: any): void {
  res.write(`event: ${event}\n`);
  res.write(`data: ${JSON.stringify(data)}\n`);
  res.write(`id: ${Date.now()}\n`);
  res.write('\n');
}
```

---

## 9. Notification Publisher (ส่ง Notification ผ่าน Redis)

```typescript
// src/services/notification.service.ts
import { RedisClientType } from 'redis';
import { Pool } from 'pg';

export interface Notification {
  id: string;
  userId: string;
  type: string;
  title: string;
  body: string;
  data?: Record<string, any>;
  read: boolean;
  createdAt: string;
}

export class NotificationService {
  constructor(
    private readonly redis: RedisClientType,
    private readonly pool: Pool
  ) {}

  async send(notification: Omit<Notification, 'id' | 'read' | 'createdAt'>): Promise<void> {
    // Save to database
    const result = await this.pool.query(
      `INSERT INTO notifications (user_id, type, title, body, data)
       VALUES ($1, $2, $3, $4, $5)
       RETURNING *`,
      [
        notification.userId,
        notification.type,
        notification.title,
        notification.body,
        JSON.stringify(notification.data || {}),
      ]
    );

    const saved = result.rows[0];

    const payload: Notification = {
      id: saved.id,
      userId: saved.user_id,
      type: saved.type,
      title: saved.title,
      body: saved.body,
      data: saved.data,
      read: false,
      createdAt: saved.created_at.toISOString(),
    };

    // Publish to Redis for SSE/WebSocket delivery
    await this.redis.publish(
      `notifications:${notification.userId}`,
      JSON.stringify(payload)
    );

    // Update unread count in Redis cache
    await this.redis.incr(`notifications:${notification.userId}:unread`);
  }

  async sendBulk(
    userIds: string[],
    notification: Omit<Notification, 'id' | 'read' | 'createdAt' | 'userId'>
  ): Promise<void> {
    const pipeline = this.redis.multi();

    for (const userId of userIds) {
      const payload = { ...notification, userId, id: Date.now().toString() };
      pipeline.publish(
        `notifications:${userId}`,
        JSON.stringify(payload)
      );
    }

    await pipeline.exec();
  }

  async markAsRead(notificationId: string, userId: string): Promise<void> {
    await this.pool.query(
      `UPDATE notifications SET read = true, read_at = NOW()
       WHERE id = $1 AND user_id = $2`,
      [notificationId, userId]
    );

    // Decrement unread count
    await this.redis.decr(`notifications:${userId}:unread`);
  }

  async markAllAsRead(userId: string): Promise<void> {
    await this.pool.query(
      `UPDATE notifications SET read = true, read_at = NOW()
       WHERE user_id = $1 AND read = false`,
      [userId]
    );

    await this.redis.set(`notifications:${userId}:unread`, '0');
  }

  async getUnreadCount(userId: string): Promise<number> {
    const cached = await this.redis.get(`notifications:${userId}:unread`);
    if (cached !== null) return parseInt(cached);

    const result = await this.pool.query(
      'SELECT COUNT(*) FROM notifications WHERE user_id = $1 AND read = false',
      [userId]
    );

    const count = parseInt(result.rows[0].count);
    await this.redis.set(`notifications:${userId}:unread`, count.toString());
    return count;
  }
}
```

---

## 10. RabbitMQ Integration

```typescript
// src/queue/rabbitmq.consumer.ts
import amqp, { Connection, Channel } from 'amqplib';
import { NotificationService } from '../services/notification.service';
import { Server } from 'socket.io';

export class RabbitMQConsumer {
  private connection: Connection | null = null;
  private channel: Channel | null = null;
  private reconnectDelay = 1000;

  constructor(
    private readonly io: Server,
    private readonly notificationService: NotificationService
  ) {}

  async connect(): Promise<void> {
    try {
      this.connection = await amqp.connect(
        process.env.RABBITMQ_URL || 'amqp://localhost:5672'
      );
      this.channel = await this.connection.createChannel();

      // Set prefetch for fair dispatch
      await this.channel.prefetch(10);

      // Assert exchanges and queues
      await this.setupQueues();

      // Start consuming
      await this.startConsuming();

      // Handle connection events
      this.connection.on('error', this.handleConnectionError.bind(this));
      this.connection.on('close', this.handleConnectionClose.bind(this));

      console.log('RabbitMQ consumer connected');
      this.reconnectDelay = 1000; // Reset on successful connect
    } catch (error) {
      console.error('Failed to connect to RabbitMQ:', error);
      await this.scheduleReconnect();
    }
  }

  private async setupQueues(): Promise<void> {
    if (!this.channel) return;

    // Dead Letter Exchange
    await this.channel.assertExchange('dlx', 'fanout', { durable: true });
    await this.channel.assertQueue('dead-letters', {
      durable: true,
      arguments: { 'x-message-ttl': 7 * 24 * 60 * 60 * 1000 },
    });
    await this.channel.bindQueue('dead-letters', 'dlx', '');

    // Notifications exchange
    await this.channel.assertExchange('notifications', 'topic', { durable: true });
    await this.channel.assertQueue('realtime.notifications', {
      durable: true,
      arguments: {
        'x-dead-letter-exchange': 'dlx',
        'x-message-ttl': 30 * 60 * 1000, // 30 minutes
      },
    });
    await this.channel.bindQueue(
      'realtime.notifications',
      'notifications',
      'notification.#'
    );

    // Order events
    await this.channel.assertExchange('orders', 'topic', { durable: true });
    await this.channel.assertQueue('realtime.order-events', {
      durable: true,
      arguments: { 'x-dead-letter-exchange': 'dlx' },
    });
    await this.channel.bindQueue(
      'realtime.order-events',
      'orders',
      'order.status.changed'
    );
  }

  private async startConsuming(): Promise<void> {
    if (!this.channel) return;

    // Consume notifications
    await this.channel.consume(
      'realtime.notifications',
      async (msg) => {
        if (!msg) return;

        try {
          const data = JSON.parse(msg.content.toString());
          await this.handleNotification(data);
          this.channel!.ack(msg);
        } catch (error) {
          console.error('Failed to process notification:', error);
          // Reject and dead-letter if retried twice
          const retryCount = (msg.properties.headers?.['x-retry-count'] || 0) as number;
          if (retryCount < 2) {
            this.channel!.nack(msg, false, false); // Dead-letter it
          } else {
            this.channel!.nack(msg, false, false);
          }
        }
      },
      { noAck: false }
    );

    // Consume order events
    await this.channel.consume(
      'realtime.order-events',
      async (msg) => {
        if (!msg) return;

        try {
          const data = JSON.parse(msg.content.toString());
          await this.handleOrderStatusChange(data);
          this.channel!.ack(msg);
        } catch (error) {
          console.error('Failed to process order event:', error);
          this.channel!.nack(msg, false, true); // Requeue
        }
      },
      { noAck: false }
    );
  }

  private async handleNotification(data: any): Promise<void> {
    await this.notificationService.send({
      userId: data.userId,
      type: data.type,
      title: data.title,
      body: data.body,
      data: data.metadata,
    });

    // Send via WebSocket if user is connected
    this.io.to(`user:${data.userId}`).emit('notification', data);
  }

  private async handleOrderStatusChange(data: any): Promise<void> {
    // Broadcast order status change to all connections subscribed to this order
    this.io.to(`order:${data.orderId}`).emit('order:status_changed', {
      orderId: data.orderId,
      status: data.status,
      previousStatus: data.previousStatus,
      updatedAt: data.updatedAt,
    });

    // Also send notification to customer
    if (data.customerId) {
      await this.notificationService.send({
        userId: data.customerId,
        type: 'ORDER_STATUS_UPDATE',
        title: 'Order Update',
        body: `Your order #${data.orderId.substring(0, 8)} is now ${data.status}`,
        data: { orderId: data.orderId, status: data.status },
      });
    }
  }

  private handleConnectionError(error: Error): void {
    console.error('RabbitMQ connection error:', error);
  }

  private async handleConnectionClose(): Promise<void> {
    console.log('RabbitMQ connection closed, reconnecting...');
    await this.scheduleReconnect();
  }

  private async scheduleReconnect(): Promise<void> {
    this.reconnectDelay = Math.min(this.reconnectDelay * 2, 30000);
    console.log(`Reconnecting in ${this.reconnectDelay}ms...`);
    setTimeout(() => this.connect(), this.reconnectDelay);
  }

  async close(): Promise<void> {
    await this.channel?.close();
    await this.connection?.close();
  }
}
```

---

## 11. Client-side Implementation

```typescript
// client/src/socket-client.ts
import { io, Socket } from 'socket.io-client';

interface SocketClientConfig {
  url: string;
  token: string;
  onReconnect?: () => void;
  onDisconnect?: (reason: string) => void;
}

export class SocketClient {
  private socket: Socket | null = null;
  private token: string;
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 10;
  private eventHandlers = new Map<string, Set<Function>>();

  constructor(private readonly config: SocketClientConfig) {
    this.token = config.token;
  }

  connect(): void {
    this.socket = io(this.config.url, {
      auth: { token: this.token },
      transports: ['websocket', 'polling'],
      reconnection: true,
      reconnectionAttempts: this.maxReconnectAttempts,
      reconnectionDelay: 1000,
      reconnectionDelayMax: 30000,
      randomizationFactor: 0.5,
      timeout: 20000,
    });

    this.socket.on('connect', this.handleConnect.bind(this));
    this.socket.on('disconnect', this.handleDisconnect.bind(this));
    this.socket.on('connect_error', this.handleConnectError.bind(this));
    this.socket.on('reconnect', this.handleReconnect.bind(this));
  }

  private handleConnect(): void {
    console.log('Connected to WebSocket server');
    this.reconnectAttempts = 0;

    // Reattach event handlers after reconnect
    for (const [event, handlers] of this.eventHandlers) {
      for (const handler of handlers) {
        this.socket!.on(event, handler as any);
      }
    }
  }

  private handleDisconnect(reason: string): void {
    console.log('Disconnected:', reason);
    this.config.onDisconnect?.(reason);
  }

  private handleConnectError(error: Error): void {
    console.error('Connection error:', error.message);
    this.reconnectAttempts++;

    if (
      error.message === 'Token expired' ||
      error.message === 'Invalid token'
    ) {
      // Stop reconnection for auth errors
      this.socket?.io.opts.reconnection = false;
      this.emit('auth_error', error.message);
    }
  }

  private handleReconnect(attemptNumber: number): void {
    console.log(`Reconnected after ${attemptNumber} attempts`);
    this.config.onReconnect?.();
  }

  on(event: string, handler: Function): () => void {
    if (!this.eventHandlers.has(event)) {
      this.eventHandlers.set(event, new Set());
    }
    this.eventHandlers.get(event)!.add(handler);
    this.socket?.on(event, handler as any);

    // Return unsubscribe function
    return () => {
      this.eventHandlers.get(event)?.delete(handler);
      this.socket?.off(event, handler as any);
    };
  }

  emit(event: string, data?: any): void {
    if (!this.socket?.connected) {
      console.warn(`Cannot emit ${event}: not connected`);
      return;
    }
    this.socket.emit(event, data);
  }

  emitWithAck<T>(event: string, data: any, timeout = 5000): Promise<T> {
    return new Promise((resolve, reject) => {
      const timer = setTimeout(() => {
        reject(new Error(`Timeout waiting for ${event} acknowledgement`));
      }, timeout);

      this.socket?.emit(event, data, (response: T) => {
        clearTimeout(timer);
        resolve(response);
      });
    });
  }

  joinRoom(roomId: string): void {
    this.emit('chat:join', { roomId });
  }

  leaveRoom(roomId: string): void {
    this.emit('chat:leave', { roomId });
  }

  sendMessage(roomId: string, content: string, replyTo?: string): void {
    this.emit('chat:message', { roomId, content, type: 'text', replyTo });
  }

  setTyping(roomId: string, isTyping: boolean): void {
    this.emit('chat:typing', { roomId, isTyping });
  }

  disconnect(): void {
    this.socket?.disconnect();
    this.socket = null;
  }

  get connected(): boolean {
    return this.socket?.connected ?? false;
  }

  get id(): string | undefined {
    return this.socket?.id;
  }
}
```

---

## 12. Docker Compose สำหรับ Horizontal Scaling

```yaml
# docker-compose.yml
version: '3.9'

services:
  nginx:
    image: nginx:alpine
    ports:
      - "3000:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - websocket-1
      - websocket-2
      - websocket-3

  websocket-1:
    build:
      context: .
      dockerfile: Dockerfile
    environment:
      PORT: 3000
      INSTANCE_ID: ws-1
      REDIS_URL: redis://redis:6379
      RABBITMQ_URL: amqp://admin:admin@rabbitmq:5672
      JWT_SECRET: ${JWT_SECRET}
      DATABASE_URL: postgresql://ws_user:ws_pass@postgres:5432/chat_db
    depends_on:
      redis:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy

  websocket-2:
    build:
      context: .
      dockerfile: Dockerfile
    environment:
      PORT: 3000
      INSTANCE_ID: ws-2
      REDIS_URL: redis://redis:6379
      RABBITMQ_URL: amqp://admin:admin@rabbitmq:5672
      JWT_SECRET: ${JWT_SECRET}
      DATABASE_URL: postgresql://ws_user:ws_pass@postgres:5432/chat_db
    depends_on:
      redis:
        condition: service_healthy

  websocket-3:
    build:
      context: .
      dockerfile: Dockerfile
    environment:
      PORT: 3000
      INSTANCE_ID: ws-3
      REDIS_URL: redis://redis:6379
      RABBITMQ_URL: amqp://admin:admin@rabbitmq:5672
      JWT_SECRET: ${JWT_SECRET}
      DATABASE_URL: postgresql://ws_user:ws_pass@postgres:5432/chat_db
    depends_on:
      redis:
        condition: service_healthy

  redis:
    image: redis:7-alpine
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  rabbitmq:
    image: rabbitmq:3.13-management-alpine
    environment:
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: admin
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "ping"]
      interval: 30s
      timeout: 10s
      retries: 5

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: ws_user
      POSTGRES_PASSWORD: ws_pass
      POSTGRES_DB: chat_db
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ws_user"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  redis_data:
  rabbitmq_data:
  postgres_data:
```

```nginx
# nginx.conf
events {
  worker_connections 10240;
}

http {
  upstream websocket_servers {
    # Sticky sessions by IP — required for WebSocket
    ip_hash;
    server websocket-1:3000;
    server websocket-2:3000;
    server websocket-3:3000;
  }

  server {
    listen 80;

    location / {
      proxy_pass http://websocket_servers;
      proxy_http_version 1.1;
      
      # WebSocket upgrade headers
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection "upgrade";
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

      # Timeouts
      proxy_connect_timeout 60s;
      proxy_send_timeout 3600s;
      proxy_read_timeout 3600s;
    }

    location /health {
      proxy_pass http://websocket_servers;
      proxy_http_version 1.1;
    }
  }
}
```

---

## สรุป

| หัวข้อ | เทคนิค | ประโยชน์ |
|--------|---------|---------|
| **Socket.io + Redis Adapter** | `@socket.io/redis-adapter` | Horizontal Scaling ข้ามหลาย Server |
| **Presence System** | Redis Key + Pub/Sub + TTL | ติดตาม Online/Offline Status แบบ Real-time |
| **Room Management** | Redis Sets | จัดการ Group Chat และ Channel |
| **Server-Sent Events** | EventSource API | One-way Push Notifications แบบ Lightweight |
| **Sticky Sessions** | Nginx `ip_hash` | ให้ Client กลับมา Server เดิมเสมอ |
| **Heartbeat** | `pingInterval`/`pingTimeout` | ตรวจจับ Connection ที่ขาดหายไป |
| **Connection State Recovery** | Socket.io built-in | ส่ง Missed Events หลัง Reconnect |
| **RabbitMQ Integration** | AMQP + Dead Letter Queue | รับ Events จาก Microservices อื่น |
| **Rate Limiting** | Custom middleware | ป้องกัน Spam และ DoS |
| **DataLoader for Messages** | Batching reads | ลด DB Queries |

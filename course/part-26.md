# Part 26: Real-time Features - WebSockets & Server-Sent Events

## ภาพรวม

ใน Part นี้เราจะสร้าง Real-time features:
- **WebSocket** - Bidirectional communication
- **Socket.io** - Room-based messaging
- **Server-Sent Events (SSE)** - Server push
- **Real-time Notifications**
- **Live Order Tracking**
- **Presence System** - Online users
- **Scaling WebSockets** ด้วย Redis Adapter

---

## 1. WebSocket กับ Socket.io

### 1.1 Socket.io Server Setup

```javascript
// realtime/socket-server.js
const { createServer } = require('http');
const { Server } = require('socket.io');
const { createAdapter } = require('@socket.io/redis-adapter');
const { createClient } = require('redis');
const jwt = require('jsonwebtoken');

class RealtimeServer {
  constructor(app, options = {}) {
    this.httpServer = createServer(app);
    this.io = new Server(this.httpServer, {
      cors: {
        origin: process.env.ALLOWED_ORIGINS?.split(',') || '*',
        methods: ['GET', 'POST'],
        credentials: true
      },
      transports: ['websocket', 'polling'],
      pingTimeout: 60000,
      pingInterval: 25000
    });

    this.setupRedisAdapter();
    this.setupMiddleware();
    this.setupNamespaces();
  }

  async setupRedisAdapter() {
    const pubClient = createClient({ url: process.env.REDIS_URL });
    const subClient = pubClient.duplicate();

    await Promise.all([pubClient.connect(), subClient.connect()]);
    
    this.io.adapter(createAdapter(pubClient, subClient));
    console.log('Socket.io Redis adapter configured');
  }

  setupMiddleware() {
    // Authentication middleware
    this.io.use(async (socket, next) => {
      const token = socket.handshake.auth.token
        || socket.handshake.headers.authorization?.replace('Bearer ', '');
      
      if (!token) {
        return next(new Error('Authentication required'));
      }

      try {
        const decoded = jwt.verify(token, process.env.JWT_PUBLIC_KEY, {
          algorithms: ['RS256']
        });
        
        socket.user = {
          id: decoded.sub,
          name: decoded.name,
          role: decoded.role,
          tenantId: decoded.tenantId
        };
        
        next();
      } catch (err) {
        next(new Error('Invalid token'));
      }
    });

    // Rate limiting middleware
    this.io.use(async (socket, next) => {
      const userId = socket.user.id;
      const count = await redis.incr(`ws:connections:${userId}`);
      
      if (count === 1) {
        await redis.expire(`ws:connections:${userId}`, 3600);
      }
      
      if (count > 5) {
        return next(new Error('Too many connections'));
      }
      
      next();
    });
  }

  setupNamespaces() {
    // Main namespace
    this.setupMainNamespace();
    
    // Admin namespace
    this.setupAdminNamespace();
    
    // Support chat namespace
    this.setupSupportNamespace();
  }

  setupMainNamespace() {
    this.io.on('connection', (socket) => {
      const user = socket.user;
      console.log(`User ${user.id} connected`);

      // Join user's personal room
      socket.join(`user:${user.id}`);
      
      // Join tenant room
      if (user.tenantId) {
        socket.join(`tenant:${user.tenantId}`);
      }

      // Track presence
      this.updatePresence(user.id, 'online');

      // Handle events
      this.registerEventHandlers(socket);

      // Handle disconnect
      socket.on('disconnect', (reason) => {
        console.log(`User ${user.id} disconnected: ${reason}`);
        this.updatePresence(user.id, 'offline');
      });
    });
  }

  registerEventHandlers(socket) {
    const { user } = socket;

    // Subscribe to order updates
    socket.on('subscribe:order', async (orderId) => {
      const isOwner = await this.checkOrderOwnership(orderId, user.id);
      if (!isOwner) {
        socket.emit('error', { message: 'Access denied' });
        return;
      }
      
      socket.join(`order:${orderId}`);
      socket.emit('subscribed', { channel: `order:${orderId}` });
    });

    // Unsubscribe from order updates
    socket.on('unsubscribe:order', (orderId) => {
      socket.leave(`order:${orderId}`);
    });

    // Subscribe to inventory updates (managers only)
    socket.on('subscribe:inventory', () => {
      if (!['admin', 'manager'].includes(user.role)) {
        socket.emit('error', { message: 'Access denied' });
        return;
      }
      socket.join('inventory:updates');
    });

    // Typing indicator (for support chat)
    socket.on('typing:start', ({ roomId }) => {
      socket.to(`room:${roomId}`).emit('user:typing', {
        userId: user.id,
        name: user.name
      });
    });

    socket.on('typing:stop', ({ roomId }) => {
      socket.to(`room:${roomId}`).emit('user:stopped-typing', {
        userId: user.id
      });
    });

    // Read receipt
    socket.on('message:read', async ({ messageId, roomId }) => {
      await this.markMessageRead(messageId, user.id);
      socket.to(`room:${roomId}`).emit('message:read-by', {
        messageId,
        userId: user.id,
        readAt: new Date().toISOString()
      });
    });
  }

  // Emit to specific user (works across multiple instances via Redis)
  async emitToUser(userId, event, data) {
    this.io.to(`user:${userId}`).emit(event, data);
  }

  // Emit to all users in a tenant
  async emitToTenant(tenantId, event, data) {
    this.io.to(`tenant:${tenantId}`).emit(event, data);
  }

  // Emit to order room
  async emitOrderUpdate(orderId, data) {
    this.io.to(`order:${orderId}`).emit('order:updated', data);
  }

  async updatePresence(userId, status) {
    await redis.setex(`presence:${userId}`, 300, status);
    
    // Notify friends/contacts
    const contacts = await this.getUserContacts(userId);
    for (const contactId of contacts) {
      this.io.to(`user:${contactId}`).emit('presence:update', {
        userId,
        status,
        updatedAt: new Date().toISOString()
      });
    }
  }

  setupAdminNamespace() {
    const adminNs = this.io.of('/admin');
    
    adminNs.use((socket, next) => {
      if (socket.user.role !== 'admin') {
        return next(new Error('Admin access required'));
      }
      next();
    });

    adminNs.on('connection', (socket) => {
      socket.join('admin-room');
      
      // Real-time dashboard events
      socket.on('subscribe:metrics', () => {
        this.startMetricsStream(socket);
      });
    });
  }

  startMetricsStream(socket) {
    const interval = setInterval(async () => {
      if (!socket.connected) {
        clearInterval(interval);
        return;
      }

      const metrics = await this.collectMetrics();
      socket.emit('metrics:update', metrics);
    }, 5000);

    socket.on('disconnect', () => clearInterval(interval));
  }
}

module.exports = RealtimeServer;
```

### 1.2 Event Publisher - ส่ง events จาก Services

```javascript
// realtime/event-publisher.js

// ใน order-service: เมื่อ order อัพเดท
class OrderEventPublisher {
  constructor(realtimeServer, eventBus) {
    this.realtimeServer = realtimeServer;
    this.eventBus = eventBus;
  }

  async publishOrderStatusChange(order, previousStatus) {
    const payload = {
      orderId: order.id,
      previousStatus,
      newStatus: order.status,
      updatedAt: new Date().toISOString(),
      order: {
        id: order.id,
        status: order.status,
        items: order.items,
        totalAmount: order.totalAmount
      }
    };

    // Real-time WebSocket
    await this.realtimeServer.emitOrderUpdate(order.id, payload);
    await this.realtimeServer.emitToUser(order.userId, 'notification', {
      type: 'order_status_change',
      title: 'Order Update',
      body: `Your order is now ${order.status}`,
      data: payload,
      timestamp: new Date().toISOString()
    });

    // Also publish to message queue for other consumers
    await this.eventBus.publish({
      type: 'OrderStatusChanged',
      ...payload
    });
  }
}
```

---

## 2. Server-Sent Events (SSE)

### 2.1 SSE Handler

```javascript
// realtime/sse-handler.js

class SSEHandler {
  constructor() {
    this.clients = new Map(); // clientId -> response
  }

  // SSE endpoint
  handle() {
    return (req, res) => {
      const userId = req.user.id;
      const clientId = `${userId}:${Date.now()}`;
      
      // Setup SSE headers
      res.writeHead(200, {
        'Content-Type': 'text/event-stream',
        'Cache-Control': 'no-cache',
        'Connection': 'keep-alive',
        'Access-Control-Allow-Origin': '*',
        'X-Accel-Buffering': 'no'  // Disable nginx buffering
      });

      // Send initial connection event
      this.sendEvent(res, 'connected', { clientId });

      // Add to clients map
      this.clients.set(clientId, { res, userId });

      // Heartbeat every 30s
      const heartbeat = setInterval(() => {
        if (!res.writable) {
          clearInterval(heartbeat);
          return;
        }
        res.write(': heartbeat\n\n');
      }, 30000);

      // Clean up on disconnect
      req.on('close', () => {
        clearInterval(heartbeat);
        this.clients.delete(clientId);
        console.log(`SSE client ${clientId} disconnected`);
      });
    };
  }

  sendEvent(res, eventType, data, id = null) {
    if (!res.writable) return;
    
    let message = '';
    if (id) message += `id: ${id}\n`;
    message += `event: ${eventType}\n`;
    message += `data: ${JSON.stringify(data)}\n\n`;
    
    res.write(message);
  }

  // Send to specific user
  sendToUser(userId, event, data) {
    for (const [clientId, client] of this.clients) {
      if (client.userId === userId) {
        this.sendEvent(client.res, event, data);
      }
    }
  }

  // Broadcast to all
  broadcast(event, data) {
    for (const [, client] of this.clients) {
      this.sendEvent(client.res, event, data);
    }
  }

  getConnectedCount() {
    return this.clients.size;
  }
}

// Routes
router.get('/events', authenticate, sseHandler.handle());

// ส่ง event
app.post('/internal/notify', async (req, res) => {
  const { userId, event, data } = req.body;
  sseHandler.sendToUser(userId, event, data);
  res.json({ sent: true });
});
```

---

## 3. Live Order Tracking

```javascript
// realtime/order-tracker.js

class LiveOrderTracker {
  constructor(io, redis) {
    this.io = io;
    this.redis = redis;
  }

  async trackOrder(orderId, userId) {
    // ดึง tracking info ปัจจุบัน
    const tracking = await this.getOrderTracking(orderId);
    
    return {
      orderId,
      currentStatus: tracking.status,
      estimatedDelivery: tracking.estimatedDelivery,
      lastLocation: tracking.lastLocation,
      timeline: tracking.timeline
    };
  }

  async updateLocation(orderId, location) {
    // อัพเดท location ใน Redis
    await this.redis.setex(
      `order:location:${orderId}`,
      3600,
      JSON.stringify({
        ...location,
        timestamp: new Date().toISOString()
      })
    );

    // Push real-time update ให้ subscribers
    this.io.to(`order:${orderId}`).emit('order:location', {
      orderId,
      location,
      timestamp: new Date().toISOString()
    });
  }

  // Delivery driver app sends location updates
  async handleDriverLocationUpdate(driverId, location) {
    const activeOrders = await this.getDriverActiveOrders(driverId);
    
    for (const orderId of activeOrders) {
      await this.updateLocation(orderId, {
        lat: location.lat,
        lng: location.lng,
        accuracy: location.accuracy,
        speed: location.speed,
        heading: location.heading
      });
    }
  }

  async getOrderTracking(orderId) {
    const [tracking, location] = await Promise.all([
      this.db.query(
        `SELECT * FROM order_tracking WHERE order_id = $1`,
        [orderId]
      ),
      this.redis.get(`order:location:${orderId}`)
    ]);

    return {
      status: tracking.rows[0]?.status,
      estimatedDelivery: tracking.rows[0]?.estimated_delivery,
      lastLocation: location ? JSON.parse(location) : null,
      timeline: await this.getTimeline(orderId)
    };
  }

  async getTimeline(orderId) {
    const result = await this.db.query(
      `SELECT status, message, created_at
       FROM order_events
       WHERE order_id = $1
       ORDER BY created_at ASC`,
      [orderId]
    );
    
    return result.rows;
  }
}
```

---

## 4. Real-time Notification System

```javascript
// notifications/notification-service.js

class NotificationService {
  constructor(io, sseHandler, pushService, db) {
    this.io = io;
    this.sse = sseHandler;
    this.push = pushService;
    this.db = db;
  }

  async send(userId, notification) {
    const {
      type,
      title,
      body,
      data = {},
      channels = ['in-app', 'push']
    } = notification;

    // Save to DB
    const saved = await this.saveNotification(userId, notification);

    // Send via each channel
    await Promise.allSettled([
      channels.includes('in-app') && this.sendInApp(userId, { ...notification, id: saved.id }),
      channels.includes('push') && this.sendPush(userId, notification),
      channels.includes('email') && this.sendEmail(userId, notification),
      channels.includes('sms') && this.sendSMS(userId, notification)
    ]);

    return saved;
  }

  async sendInApp(userId, notification) {
    // WebSocket
    this.io.to(`user:${userId}`).emit('notification', notification);
    
    // SSE (for non-WebSocket clients)
    this.sse.sendToUser(userId, 'notification', notification);
  }

  async sendPush(userId, notification) {
    const tokens = await this.getUserPushTokens(userId);
    
    if (tokens.length === 0) return;

    await this.push.sendMulticast({
      tokens,
      notification: {
        title: notification.title,
        body: notification.body
      },
      data: notification.data,
      android: {
        priority: 'high',
        notification: { channelId: notification.type }
      },
      apns: {
        payload: {
          aps: {
            badge: await this.getUnreadCount(userId),
            sound: 'default'
          }
        }
      }
    });
  }

  async saveNotification(userId, notification) {
    const result = await this.db.query(
      `INSERT INTO notifications (user_id, type, title, body, data, read, created_at)
       VALUES ($1, $2, $3, $4, $5, false, NOW())
       RETURNING *`,
      [userId, notification.type, notification.title, notification.body, notification.data]
    );
    return result.rows[0];
  }

  async markRead(userId, notificationIds) {
    await this.db.query(
      `UPDATE notifications
       SET read = true, read_at = NOW()
       WHERE id = ANY($1) AND user_id = $2`,
      [notificationIds, userId]
    );

    // Update badge count
    const unreadCount = await this.getUnreadCount(userId);
    this.io.to(`user:${userId}`).emit('notification:badge', { count: unreadCount });
  }

  async getUnreadCount(userId) {
    const result = await this.db.query(
      'SELECT COUNT(*) FROM notifications WHERE user_id = $1 AND read = false',
      [userId]
    );
    return parseInt(result.rows[0].count);
  }

  async getNotifications(userId, options = {}) {
    const { cursor, limit = 20, unreadOnly = false } = options;
    
    let conditions = ['user_id = $1'];
    const params = [userId];
    
    if (unreadOnly) {
      conditions.push('read = false');
    }
    
    if (cursor) {
      params.push(cursor);
      conditions.push(`created_at < $${params.length}`);
    }

    const result = await this.db.query(
      `SELECT * FROM notifications
       WHERE ${conditions.join(' AND ')}
       ORDER BY created_at DESC
       LIMIT $${params.length + 1}`,
      [...params, limit + 1]
    );

    const rows = result.rows;
    const hasMore = rows.length > limit;
    
    return {
      notifications: hasMore ? rows.slice(0, limit) : rows,
      hasMore,
      nextCursor: hasMore ? rows[limit - 1].created_at : null
    };
  }
}
```

---

## 5. Presence System

```javascript
// realtime/presence.js

class PresenceService {
  constructor(redis, io) {
    this.redis = redis;
    this.io = io;
    this.TTL = 300; // 5 minutes
  }

  async setOnline(userId, metadata = {}) {
    await this.redis.setex(
      `presence:${userId}`,
      this.TTL,
      JSON.stringify({
        status: 'online',
        lastSeen: new Date().toISOString(),
        ...metadata
      })
    );

    await this.notifyContacts(userId, 'online');
  }

  async setAway(userId) {
    const presence = await this.getPresence(userId);
    if (!presence) return;
    
    await this.redis.setex(
      `presence:${userId}`,
      this.TTL,
      JSON.stringify({ ...presence, status: 'away' })
    );

    await this.notifyContacts(userId, 'away');
  }

  async setOffline(userId) {
    await this.redis.setex(
      `presence:${userId}`,
      86400,  // Keep for 24h to show last seen
      JSON.stringify({
        status: 'offline',
        lastSeen: new Date().toISOString()
      })
    );

    await this.notifyContacts(userId, 'offline');
  }

  async getPresence(userId) {
    const data = await this.redis.get(`presence:${userId}`);
    return data ? JSON.parse(data) : null;
  }

  async getBulkPresence(userIds) {
    const pipeline = this.redis.pipeline();
    for (const id of userIds) {
      pipeline.get(`presence:${id}`);
    }
    
    const results = await pipeline.exec();
    
    return userIds.reduce((acc, id, index) => {
      const [, data] = results[index];
      acc[id] = data ? JSON.parse(data) : { status: 'unknown' };
      return acc;
    }, {});
  }

  async notifyContacts(userId, status) {
    const contacts = await this.getOnlineContacts(userId);
    
    for (const contactId of contacts) {
      this.io.to(`user:${contactId}`).emit('presence:change', {
        userId,
        status,
        timestamp: new Date().toISOString()
      });
    }
  }

  async getOnlineContacts(userId) {
    // Get user's contact list and check their presence
    const contacts = await this.getUserContacts(userId);
    const presenceData = await this.getBulkPresence(contacts);
    
    return contacts.filter(id => presenceData[id]?.status === 'online');
  }
}
```

---

## 6. Scaling WebSockets

```javascript
// realtime/scaling.js

// Docker Compose สำหรับ multiple WebSocket servers
/*
services:
  ws-server-1:
    image: ws-server
    environment:
      - REDIS_URL=redis://redis:6379
      
  ws-server-2:
    image: ws-server
    environment:
      - REDIS_URL=redis://redis:6379
      
  nginx:
    image: nginx
    # Sticky sessions + WebSocket upgrade
*/

// nginx.conf สำหรับ WebSocket load balancing
const nginxConfig = `
upstream websocket_servers {
  # Sticky sessions (same user → same server)
  ip_hash;
  
  server ws-server-1:3000;
  server ws-server-2:3000;
  server ws-server-3:3000;
}

server {
  listen 80;
  
  location /socket.io/ {
    proxy_pass http://websocket_servers;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_read_timeout 86400;  # Keep connection alive 24h
  }
}
`;

// Redis Pub/Sub สำหรับ cross-server messaging
class CrossServerMessenger {
  constructor(pubClient, subClient, io) {
    this.pub = pubClient;
    this.sub = subClient;
    this.io = io;
    
    this.setupSubscriptions();
  }

  setupSubscriptions() {
    this.sub.subscribe('ws:emit', (message) => {
      const { room, event, data } = JSON.parse(message);
      this.io.to(room).emit(event, data);
    });
  }

  async emitToRoom(room, event, data) {
    // Publish to all ws servers via Redis
    await this.pub.publish('ws:emit', JSON.stringify({ room, event, data }));
  }
}
```

---

## Workshop: Real-time Order Dashboard

```javascript
// Client-side code (browser)
const socket = io('wss://api.company.com', {
  auth: { token: localStorage.getItem('auth_token') },
  transports: ['websocket']
});

// Connection events
socket.on('connect', () => {
  console.log('Connected:', socket.id);
  
  // Subscribe to order updates
  socket.emit('subscribe:order', orderId);
});

socket.on('disconnect', (reason) => {
  console.log('Disconnected:', reason);
  // Auto-reconnect handled by Socket.io
});

// Real-time events
socket.on('order:updated', (data) => {
  updateOrderStatusUI(data);
  showToast(`Order status: ${data.newStatus}`);
});

socket.on('order:location', (data) => {
  updateMapMarker(data.location);
  updateETA(data.estimatedDelivery);
});

socket.on('notification', (notification) => {
  showNotificationBadge(notification);
  addToNotificationList(notification);
});

socket.on('presence:change', ({ userId, status }) => {
  updateUserPresenceIcon(userId, status);
});

// SSE fallback
const eventSource = new EventSource('/api/events', {
  withCredentials: true
});

eventSource.addEventListener('notification', (e) => {
  const notification = JSON.parse(e.data);
  handleNotification(notification);
});

eventSource.addEventListener('order_update', (e) => {
  const update = JSON.parse(e.data);
  handleOrderUpdate(update);
});

eventSource.onerror = () => {
  // Fallback to polling if SSE fails
  startPolling();
};
```

---

## สรุป

| Feature | WebSocket | SSE | Polling |
|---------|-----------|-----|---------|
| Direction | Bidirectional | Server → Client | Client pull |
| Protocol | WS/WSS | HTTP | HTTP |
| Browser support | All | All (except IE) | All |
| Scaling | Redis adapter | Stateless | Easy |
| Use case | Chat, games, presence | Notifications, live data | Simple updates |

**Best Practices:**
- ใช้ Socket.io + Redis adapter สำหรับ horizontal scaling
- SSE สำหรับ notification-only use cases (simpler)
- ตรวจสอบ auth ใน middleware ก่อน socket เชื่อมต่อ
- Heartbeat ป้องกัน connection timeout
- Graceful reconnection handling ใน client

**Next:** Part 27 - Search with Elasticsearch

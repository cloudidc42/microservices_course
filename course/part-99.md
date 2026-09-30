# Part 99: Final Project - Complete Microservices System

## บทนำ

ในบทสุดท้ายของโปรเจกต์ เราจะประกอบทุกสิ่งที่เรียนมาตลอดหลักสูตรเข้าด้วยกันเป็นระบบ microservices ที่สมบูรณ์พร้อมใช้งาน production เราจะสร้าง e-commerce platform ขนาดกลางที่ประกอบด้วย 5 services หลัก พร้อมด้วย infrastructure ครบครัน

---

## 99.1 Architecture Overview

ระบบ e-commerce ประกอบด้วย 5 services หลัก:

```
┌─────────────────────────────────────────────────────────────────┐
│                         API Gateway (Kong)                        │
│                    https://api.myshop.example.com                │
└──────────┬──────────────┬──────────────┬─────────────┬──────────┘
           │              │              │             │
    ┌──────▼─────┐  ┌─────▼──────┐ ┌───▼──────┐ ┌───▼──────────┐
    │ UserService│  │ProductSvc  │ │OrderSvc  │ │PaymentService│
    │  :3001     │  │  :3002     │ │ :3003    │ │   :3004      │
    └──────┬─────┘  └─────┬──────┘ └───┬──────┘ └───┬──────────┘
           │              │            │             │
    ┌──────▼─────┐  ┌─────▼──────┐    │        ┌───▼──────────┐
    │ PostgreSQL │  │ PostgreSQL │    │        │NotificationSvc│
    │(users_db)  │  │(products_db│    │        │   :3005      │
    └──────┬─────┘  └────────────┘    │        └──────────────┘
           │                     ┌────▼───────────────────────┐
    ┌──────▼─────┐               │         Kafka              │
    │  Redis     │               │  orders.created            │
    │(cache/jwt) │               │  payments.processed        │
    └────────────┘               │  notifications.send        │
                                 └────────────────────────────┘
```

### Technology Stack

| Component | Technology |
|-----------|-----------|
| Runtime | Node.js 20 + TypeScript |
| Framework | Express.js |
| ORM | Prisma |
| Database | PostgreSQL 15 |
| Cache | Redis 7 |
| Message Broker | Apache Kafka |
| Container | Docker |
| Orchestration | Kubernetes |
| CI/CD | GitHub Actions |
| Monitoring | Prometheus + Grafana |
| Tracing | Jaeger |

---

## 99.2 TypeScript UserService

UserService จัดการ authentication และข้อมูลผู้ใช้ด้วย Express + Prisma ORM + Redis cache + JWT auth:

### Prisma Schema

```prisma
// user-service/prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id          String    @id @default(cuid())
  email       String    @unique
  password    String
  firstName   String
  lastName    String
  phone       String?
  role        UserRole  @default(CUSTOMER)
  isActive    Boolean   @default(true)
  isVerified  Boolean   @default(false)
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt
  addresses   Address[]
  sessions    Session[]

  @@index([email])
  @@map("users")
}

model Address {
  id         String   @id @default(cuid())
  userId     String
  type       String   @default("shipping")
  street     String
  city       String
  state      String
  country    String
  postalCode String
  isDefault  Boolean  @default(false)
  user       User     @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@map("addresses")
}

model Session {
  id        String   @id @default(cuid())
  userId    String
  token     String   @unique
  userAgent String?
  ipAddress String?
  expiresAt DateTime
  createdAt DateTime @default(now())
  user      User     @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([token])
  @@index([userId])
  @@map("sessions")
}

enum UserRole {
  CUSTOMER
  ADMIN
  VENDOR
}
```

### UserService Implementation

```typescript
// user-service/src/index.ts
import express from 'express';
import helmet from 'helmet';
import cors from 'cors';
import rateLimit from 'express-rate-limit';
import { PrismaClient } from '@prisma/client';
import { createClient } from 'redis';
import bcrypt from 'bcryptjs';
import jwt from 'jsonwebtoken';
import { z } from 'zod';
import { createLogger, transports, format } from 'winston';
import { register, Counter, Histogram } from 'prom-client';

const app = express();
const prisma = new PrismaClient();
const redis = createClient({ url: process.env.REDIS_URL });
const logger = createLogger({
  format: format.combine(format.timestamp(), format.json()),
  transports: [new transports.Console()],
});

// Prometheus metrics
const httpRequestsTotal = new Counter({
  name: 'http_requests_total',
  help: 'Total HTTP requests',
  labelNames: ['method', 'path', 'status'],
});
const httpRequestDuration = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration',
  labelNames: ['method', 'path'],
  buckets: [0.01, 0.05, 0.1, 0.5, 1, 5],
});

// Middleware
app.use(helmet());
app.use(cors({ origin: process.env.ALLOWED_ORIGINS?.split(',') }));
app.use(express.json({ limit: '1mb' }));
app.use(rateLimit({ windowMs: 15 * 60 * 1000, max: 100 }));

// Metrics middleware
app.use((req, res, next) => {
  const start = Date.now();
  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000;
    httpRequestsTotal.labels(req.method, req.path, String(res.statusCode)).inc();
    httpRequestDuration.labels(req.method, req.path).observe(duration);
  });
  next();
});

// Validation schemas
const RegisterSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8).max(100),
  firstName: z.string().min(1).max(50),
  lastName: z.string().min(1).max(50),
  phone: z.string().optional(),
});

const LoginSchema = z.object({
  email: z.string().email(),
  password: z.string(),
});

const UpdateUserSchema = z.object({
  firstName: z.string().min(1).max(50).optional(),
  lastName: z.string().min(1).max(50).optional(),
  phone: z.string().optional(),
});

// JWT utilities
function generateTokens(userId: string, role: string) {
  const accessToken = jwt.sign(
    { userId, role },
    process.env.JWT_SECRET!,
    { expiresIn: '15m' }
  );
  const refreshToken = jwt.sign(
    { userId, type: 'refresh' },
    process.env.JWT_REFRESH_SECRET!,
    { expiresIn: '7d' }
  );
  return { accessToken, refreshToken };
}

function verifyToken(token: string): { userId: string; role: string } {
  return jwt.verify(token, process.env.JWT_SECRET!) as { userId: string; role: string };
}

// Auth middleware
const authenticate = async (
  req: express.Request,
  res: express.Response,
  next: express.NextFunction
): Promise<void> => {
  const authHeader = req.headers.authorization;
  if (!authHeader?.startsWith('Bearer ')) {
    res.status(401).json({ error: 'Unauthorized' });
    return;
  }

  const token = authHeader.substring(7);

  // Check token blacklist in Redis
  const isBlacklisted = await redis.get(`blacklist:${token}`);
  if (isBlacklisted) {
    res.status(401).json({ error: 'Token revoked' });
    return;
  }

  try {
    const payload = verifyToken(token);
    (req as express.Request & { user: typeof payload }).user = payload;
    next();
  } catch {
    res.status(401).json({ error: 'Invalid token' });
  }
};

// Routes: Auth
app.post('/auth/register', async (req, res): Promise<void> => {
  try {
    const body = RegisterSchema.parse(req.body);

    const existing = await prisma.user.findUnique({ where: { email: body.email } });
    if (existing) {
      res.status(409).json({ error: 'Email already registered' });
      return;
    }

    const hashedPassword = await bcrypt.hash(body.password, 12);
    const user = await prisma.user.create({
      data: {
        email: body.email,
        password: hashedPassword,
        firstName: body.firstName,
        lastName: body.lastName,
        phone: body.phone,
      },
      select: { id: true, email: true, firstName: true, lastName: true, role: true },
    });

    const { accessToken, refreshToken } = generateTokens(user.id, user.role);

    // Store refresh token in Redis
    await redis.setEx(`refresh:${user.id}`, 7 * 24 * 3600, refreshToken);

    logger.info('User registered', { userId: user.id });
    res.status(201).json({ user, accessToken, refreshToken });
  } catch (error) {
    if (error instanceof z.ZodError) {
      res.status(400).json({ error: 'Validation failed', details: error.errors });
      return;
    }
    logger.error('Registration error', { error });
    res.status(500).json({ error: 'Internal server error' });
  }
});

app.post('/auth/login', async (req, res): Promise<void> => {
  try {
    const body = LoginSchema.parse(req.body);

    // Check rate limit for failed attempts
    const failKey = `login_fail:${body.email}`;
    const failures = await redis.get(failKey);
    if (failures && parseInt(failures) >= 5) {
      res.status(429).json({ error: 'Too many failed attempts. Try again in 15 minutes.' });
      return;
    }

    const user = await prisma.user.findUnique({
      where: { email: body.email },
    });

    if (!user || !(await bcrypt.compare(body.password, user.password))) {
      await redis.setEx(failKey, 900, String((parseInt(failures ?? '0') + 1)));
      res.status(401).json({ error: 'Invalid credentials' });
      return;
    }

    if (!user.isActive) {
      res.status(403).json({ error: 'Account deactivated' });
      return;
    }

    // Clear failure counter
    await redis.del(failKey);

    const { accessToken, refreshToken } = generateTokens(user.id, user.role);
    await redis.setEx(`refresh:${user.id}`, 7 * 24 * 3600, refreshToken);

    // Cache user profile
    await redis.setEx(
      `user:${user.id}`,
      300,
      JSON.stringify({ id: user.id, email: user.email, firstName: user.firstName, lastName: user.lastName, role: user.role })
    );

    logger.info('User logged in', { userId: user.id });
    res.json({
      user: { id: user.id, email: user.email, firstName: user.firstName, lastName: user.lastName, role: user.role },
      accessToken,
      refreshToken,
    });
  } catch (error) {
    if (error instanceof z.ZodError) {
      res.status(400).json({ error: 'Validation failed', details: error.errors });
      return;
    }
    logger.error('Login error', { error });
    res.status(500).json({ error: 'Internal server error' });
  }
});

app.post('/auth/refresh', async (req, res): Promise<void> => {
  const { refreshToken } = req.body;
  if (!refreshToken) {
    res.status(400).json({ error: 'Refresh token required' });
    return;
  }

  try {
    const payload = jwt.verify(refreshToken, process.env.JWT_REFRESH_SECRET!) as { userId: string };
    const stored = await redis.get(`refresh:${payload.userId}`);

    if (stored !== refreshToken) {
      res.status(401).json({ error: 'Invalid refresh token' });
      return;
    }

    const user = await prisma.user.findUnique({ where: { id: payload.userId } });
    if (!user || !user.isActive) {
      res.status(401).json({ error: 'User not found or inactive' });
      return;
    }

    const tokens = generateTokens(user.id, user.role);
    await redis.setEx(`refresh:${user.id}`, 7 * 24 * 3600, tokens.refreshToken);

    res.json(tokens);
  } catch {
    res.status(401).json({ error: 'Invalid refresh token' });
  }
});

app.post('/auth/logout', authenticate, async (req, res): Promise<void> => {
  const token = req.headers.authorization!.substring(7);
  const user = (req as express.Request & { user: { userId: string } }).user;

  // Blacklist the current access token
  await redis.setEx(`blacklist:${token}`, 900, '1'); // 15 min TTL
  await redis.del(`refresh:${user.userId}`);
  await redis.del(`user:${user.userId}`);

  res.json({ message: 'Logged out successfully' });
});

// Routes: Users CRUD
app.get('/users/me', authenticate, async (req, res): Promise<void> => {
  const user = (req as express.Request & { user: { userId: string } }).user;

  // Try cache first
  const cached = await redis.get(`user:${user.userId}`);
  if (cached) {
    res.json(JSON.parse(cached));
    return;
  }

  const userData = await prisma.user.findUnique({
    where: { id: user.userId },
    select: {
      id: true, email: true, firstName: true, lastName: true,
      phone: true, role: true, createdAt: true, addresses: true,
    },
  });

  if (!userData) {
    res.status(404).json({ error: 'User not found' });
    return;
  }

  await redis.setEx(`user:${user.userId}`, 300, JSON.stringify(userData));
  res.json(userData);
});

app.put('/users/me', authenticate, async (req, res): Promise<void> => {
  try {
    const user = (req as express.Request & { user: { userId: string } }).user;
    const body = UpdateUserSchema.parse(req.body);

    const updated = await prisma.user.update({
      where: { id: user.userId },
      data: body,
      select: { id: true, email: true, firstName: true, lastName: true, phone: true },
    });

    await redis.del(`user:${user.userId}`);
    res.json(updated);
  } catch (error) {
    if (error instanceof z.ZodError) {
      res.status(400).json({ error: 'Validation failed', details: error.errors });
      return;
    }
    res.status(500).json({ error: 'Internal server error' });
  }
});

app.get('/users/:id', authenticate, async (req, res): Promise<void> => {
  const requester = (req as express.Request & { user: { userId: string; role: string } }).user;
  if (requester.role !== 'ADMIN' && requester.userId !== req.params.id) {
    res.status(403).json({ error: 'Forbidden' });
    return;
  }

  const user = await prisma.user.findUnique({
    where: { id: req.params.id },
    select: { id: true, email: true, firstName: true, lastName: true, role: true, createdAt: true },
  });

  if (!user) {
    res.status(404).json({ error: 'User not found' });
    return;
  }

  res.json(user);
});

// Health check
app.get('/health', (_, res) => {
  res.json({ status: 'ok', service: 'user-service', timestamp: new Date().toISOString() });
});

// Metrics endpoint
app.get('/metrics', async (_, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});

async function main() {
  await redis.connect();
  logger.info('Connected to Redis');

  await prisma.$connect();
  logger.info('Connected to PostgreSQL');

  const PORT = process.env.PORT ?? 3001;
  app.listen(PORT, () => {
    logger.info(`UserService running on port ${PORT}`);
  });
}

main().catch((error) => {
  logger.error('Failed to start service', { error });
  process.exit(1);
});
```

---

## 99.3 TypeScript OrderService

OrderService จัดการ orders และ publish events ไปยัง Kafka:

```typescript
// order-service/src/index.ts
import express from 'express';
import { Pool } from 'pg';
import { Kafka, Producer, Consumer } from 'kafkajs';
import { z } from 'zod';
import axios from 'axios';
import { createLogger, transports, format } from 'winston';
import { v4 as uuidv4 } from 'uuid';

const app = express();
app.use(express.json());

const logger = createLogger({
  format: format.combine(format.timestamp(), format.json()),
  transports: [new transports.Console()],
});

// PostgreSQL connection pool
const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 20,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});

// Kafka setup
const kafka = new Kafka({
  clientId: 'order-service',
  brokers: (process.env.KAFKA_BROKERS ?? 'kafka:9092').split(','),
  retry: {
    initialRetryTime: 100,
    retries: 8,
  },
});

const producer: Producer = kafka.producer({
  idempotent: true,
  transactionalId: 'order-service-producer',
});

// Schemas
const CreateOrderSchema = z.object({
  items: z.array(
    z.object({
      productId: z.string(),
      quantity: z.number().int().positive(),
      price: z.number().positive(),
    })
  ).min(1),
  shippingAddressId: z.string(),
  couponCode: z.string().optional(),
});

interface Order {
  id: string;
  userId: string;
  status: 'pending' | 'confirmed' | 'processing' | 'shipped' | 'delivered' | 'cancelled';
  items: OrderItem[];
  subtotal: number;
  discount: number;
  total: number;
  shippingAddressId: string;
  createdAt: Date;
  updatedAt: Date;
}

interface OrderItem {
  id: string;
  orderId: string;
  productId: string;
  quantity: number;
  price: number;
  total: number;
}

// Auth middleware (validates JWT with UserService)
const authenticate = async (
  req: express.Request,
  res: express.Response,
  next: express.NextFunction
): Promise<void> => {
  const authHeader = req.headers.authorization;
  if (!authHeader?.startsWith('Bearer ')) {
    res.status(401).json({ error: 'Unauthorized' });
    return;
  }

  try {
    // Validate token with UserService
    const response = await axios.get(
      `${process.env.USER_SERVICE_URL}/users/me`,
      { headers: { Authorization: authHeader }, timeout: 2000 }
    );
    (req as express.Request & { user: { id: string; role: string } }).user = response.data;
    next();
  } catch {
    res.status(401).json({ error: 'Invalid token' });
  }
};

// Outbox pattern: save event to DB atomically with order
async function createOrderWithEvent(
  client: import('pg').PoolClient,
  userId: string,
  orderData: z.infer<typeof CreateOrderSchema>
): Promise<Order> {
  const orderId = uuidv4();
  const subtotal = orderData.items.reduce((sum, item) => sum + item.price * item.quantity, 0);
  let discount = 0;

  // Apply coupon (simplified)
  if (orderData.couponCode === 'SAVE10') {
    discount = subtotal * 0.1;
  }
  const total = subtotal - discount;

  // Create order
  await client.query(
    `INSERT INTO orders (id, user_id, status, subtotal, discount, total, shipping_address_id)
     VALUES ($1, $2, 'pending', $3, $4, $5, $6)`,
    [orderId, userId, subtotal, discount, total, orderData.shippingAddressId]
  );

  // Create order items
  for (const item of orderData.items) {
    await client.query(
      `INSERT INTO order_items (id, order_id, product_id, quantity, price, total)
       VALUES ($1, $2, $3, $4, $5, $6)`,
      [uuidv4(), orderId, item.productId, item.quantity, item.price, item.price * item.quantity]
    );
  }

  // Outbox: save event to be published
  const event = {
    orderId,
    userId,
    items: orderData.items,
    total,
    timestamp: new Date().toISOString(),
  };

  await client.query(
    `INSERT INTO outbox (id, event_type, aggregate_id, payload, status)
     VALUES ($1, 'order.created', $2, $3, 'pending')`,
    [uuidv4(), orderId, JSON.stringify(event)]
  );

  const result = await client.query<Order>(
    `SELECT o.*, json_agg(
      json_build_object(
        'id', oi.id, 'productId', oi.product_id, 'quantity', oi.quantity,
        'price', oi.price, 'total', oi.total
      )
    ) as items
    FROM orders o
    LEFT JOIN order_items oi ON o.id = oi.order_id
    WHERE o.id = $1
    GROUP BY o.id`,
    [orderId]
  );

  return result.rows[0];
}

// Routes
app.post('/orders', authenticate, async (req, res): Promise<void> => {
  const user = (req as express.Request & { user: { id: string } }).user;

  const body = CreateOrderSchema.safeParse(req.body);
  if (!body.success) {
    res.status(400).json({ error: 'Validation failed', details: body.error.errors });
    return;
  }

  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    const order = await createOrderWithEvent(client, user.id, body.data);
    await client.query('COMMIT');

    logger.info('Order created', { orderId: order.id, userId: user.id });
    res.status(201).json(order);
  } catch (error) {
    await client.query('ROLLBACK');
    logger.error('Failed to create order', { error });
    res.status(500).json({ error: 'Failed to create order' });
  } finally {
    client.release();
  }
});

app.get('/orders', authenticate, async (req, res): Promise<void> => {
  const user = (req as express.Request & { user: { id: string } }).user;
  const page = parseInt(String(req.query.page ?? '1'));
  const limit = Math.min(parseInt(String(req.query.limit ?? '10')), 50);
  const offset = (page - 1) * limit;

  const [ordersResult, countResult] = await Promise.all([
    pool.query(
      `SELECT o.*, json_agg(
        json_build_object('id', oi.id, 'productId', oi.product_id, 'quantity', oi.quantity, 'price', oi.price)
        ORDER BY oi.id
      ) as items
      FROM orders o
      LEFT JOIN order_items oi ON o.id = oi.order_id
      WHERE o.user_id = $1
      GROUP BY o.id
      ORDER BY o.created_at DESC
      LIMIT $2 OFFSET $3`,
      [user.id, limit, offset]
    ),
    pool.query('SELECT COUNT(*) FROM orders WHERE user_id = $1', [user.id]),
  ]);

  res.json({
    data: ordersResult.rows,
    pagination: {
      total: parseInt(countResult.rows[0].count),
      page,
      limit,
      pages: Math.ceil(parseInt(countResult.rows[0].count) / limit),
    },
  });
});

app.get('/orders/:id', authenticate, async (req, res): Promise<void> => {
  const user = (req as express.Request & { user: { id: string; role: string } }).user;

  const result = await pool.query(
    `SELECT o.*, json_agg(
      json_build_object('id', oi.id, 'productId', oi.product_id, 'quantity', oi.quantity, 'price', oi.price, 'total', oi.total)
      ORDER BY oi.id
    ) as items
    FROM orders o
    LEFT JOIN order_items oi ON o.id = oi.order_id
    WHERE o.id = $1 AND (o.user_id = $2 OR $3 = 'ADMIN')
    GROUP BY o.id`,
    [req.params.id, user.id, user.role]
  );

  if (!result.rows[0]) {
    res.status(404).json({ error: 'Order not found' });
    return;
  }

  res.json(result.rows[0]);
});

app.patch('/orders/:id/cancel', authenticate, async (req, res): Promise<void> => {
  const user = (req as express.Request & { user: { id: string } }).user;

  const result = await pool.query(
    `UPDATE orders SET status = 'cancelled', updated_at = NOW()
     WHERE id = $1 AND user_id = $2 AND status IN ('pending', 'confirmed')
     RETURNING *`,
    [req.params.id, user.id]
  );

  if (!result.rows[0]) {
    res.status(400).json({ error: 'Cannot cancel order' });
    return;
  }

  // Publish cancellation event
  await producer.send({
    topic: 'orders.cancelled',
    messages: [{ key: req.params.id, value: JSON.stringify(result.rows[0]) }],
  });

  res.json(result.rows[0]);
});

// Outbox processor - polls DB and publishes events
async function processOutbox() {
  const client = await pool.connect();
  try {
    const result = await client.query(
      `SELECT * FROM outbox WHERE status = 'pending' ORDER BY created_at LIMIT 100`
    );

    for (const row of result.rows) {
      try {
        await producer.send({
          topic: row.event_type.replace('.', '.'),
          messages: [{ key: row.aggregate_id, value: row.payload }],
        });

        await client.query(
          `UPDATE outbox SET status = 'published', published_at = NOW() WHERE id = $1`,
          [row.id]
        );
      } catch (err) {
        logger.error('Failed to publish outbox event', { id: row.id, error: err });
      }
    }
  } finally {
    client.release();
  }
}

app.get('/health', (_, res) => {
  res.json({ status: 'ok', service: 'order-service' });
});

async function main() {
  await producer.connect();
  logger.info('Kafka producer connected');

  // Start outbox processor
  setInterval(processOutbox, 5000);

  const PORT = process.env.PORT ?? 3003;
  app.listen(PORT, () => logger.info(`OrderService on port ${PORT}`));
}

main().catch((err) => { logger.error('Startup failed', { err }); process.exit(1); });
```

---

## 99.4 TypeScript PaymentService

PaymentService จัดการ payment processing ด้วย Stripe + idempotency key:

```typescript
// payment-service/src/index.ts
import express from 'express';
import Stripe from 'stripe';
import { Pool } from 'pg';
import { Kafka, Producer } from 'kafkajs';
import crypto from 'crypto';
import { z } from 'zod';
import { createLogger, transports, format } from 'winston';

const app = express();
const logger = createLogger({
  format: format.combine(format.timestamp(), format.json()),
  transports: [new transports.Console()],
});

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
  apiVersion: '2024-06-20',
});

const pool = new Pool({ connectionString: process.env.DATABASE_URL });

const kafka = new Kafka({
  clientId: 'payment-service',
  brokers: (process.env.KAFKA_BROKERS ?? 'kafka:9092').split(','),
});
const producer: Producer = kafka.producer();

// Use raw body for Stripe webhook signature verification
app.use('/webhooks/stripe', express.raw({ type: 'application/json' }));
app.use(express.json());

const ProcessPaymentSchema = z.object({
  orderId: z.string().uuid(),
  amount: z.number().positive(),
  currency: z.string().length(3).default('usd'),
  paymentMethodId: z.string(),
  customerId: z.string().optional(),
});

// Idempotency: Generate deterministic key from orderId
function generateIdempotencyKey(orderId: string, attempt: number = 1): string {
  return crypto
    .createHash('sha256')
    .update(`${orderId}:${attempt}`)
    .digest('hex')
    .substring(0, 36);
}

// Check if payment already processed (idempotency)
async function findExistingPayment(orderId: string) {
  const result = await pool.query(
    'SELECT * FROM payments WHERE order_id = $1 AND status != $2',
    [orderId, 'failed']
  );
  return result.rows[0] ?? null;
}

async function recordPayment(
  orderId: string,
  userId: string,
  amount: number,
  currency: string,
  stripePaymentIntentId: string,
  status: 'pending' | 'succeeded' | 'failed'
) {
  const result = await pool.query(
    `INSERT INTO payments (id, order_id, user_id, amount, currency, stripe_payment_intent_id, status)
     VALUES (gen_random_uuid(), $1, $2, $3, $4, $5, $6)
     ON CONFLICT (order_id) DO UPDATE
     SET status = EXCLUDED.status, updated_at = NOW()
     RETURNING *`,
    [orderId, userId, amount, currency, stripePaymentIntentId, status]
  );
  return result.rows[0];
}

// Routes
app.post('/payments/process', async (req, res): Promise<void> => {
  const userId = req.headers['x-user-id'] as string;
  if (!userId) {
    res.status(401).json({ error: 'Unauthorized' });
    return;
  }

  const body = ProcessPaymentSchema.safeParse(req.body);
  if (!body.success) {
    res.status(400).json({ error: 'Validation failed', details: body.error.errors });
    return;
  }

  const { orderId, amount, currency, paymentMethodId, customerId } = body.data;

  // Idempotency check
  const existing = await findExistingPayment(orderId);
  if (existing?.status === 'succeeded') {
    logger.info('Duplicate payment request - returning existing', { orderId });
    res.status(200).json({ payment: existing, idempotent: true });
    return;
  }

  const idempotencyKey = generateIdempotencyKey(orderId);

  try {
    const paymentIntent = await stripe.paymentIntents.create(
      {
        amount: Math.round(amount * 100), // Convert to cents
        currency,
        payment_method: paymentMethodId,
        customer: customerId,
        confirm: true,
        return_url: `${process.env.FRONTEND_URL}/orders/${orderId}`,
        metadata: {
          orderId,
          userId,
        },
      },
      { idempotencyKey }
    );

    const status = paymentIntent.status === 'succeeded' ? 'succeeded'
      : paymentIntent.status === 'requires_action' ? 'pending'
      : 'failed';

    const payment = await recordPayment(orderId, userId, amount, currency, paymentIntent.id, status);

    if (status === 'succeeded') {
      await producer.send({
        topic: 'payments.processed',
        messages: [{
          key: orderId,
          value: JSON.stringify({
            orderId,
            userId,
            paymentId: payment.id,
            amount,
            currency,
            status: 'succeeded',
            timestamp: new Date().toISOString(),
          }),
        }],
      });
    }

    logger.info('Payment processed', { orderId, status, paymentIntentId: paymentIntent.id });
    res.status(status === 'succeeded' ? 200 : 202).json({
      payment,
      requiresAction: paymentIntent.status === 'requires_action',
      clientSecret: paymentIntent.status === 'requires_action' ? paymentIntent.client_secret : undefined,
    });
  } catch (error) {
    if (error instanceof Stripe.errors.StripeCardError) {
      await recordPayment(orderId, userId, amount, currency, 'failed', 'failed');
      res.status(400).json({ error: error.message, code: error.code });
      return;
    }
    logger.error('Payment processing error', { error });
    res.status(500).json({ error: 'Payment processing failed' });
  }
});

app.post('/payments/:id/refund', async (req, res): Promise<void> => {
  const { id } = req.params;
  const { amount, reason } = req.body;

  const payment = await pool.query('SELECT * FROM payments WHERE id = $1', [id]);
  if (!payment.rows[0]) {
    res.status(404).json({ error: 'Payment not found' });
    return;
  }

  if (payment.rows[0].status !== 'succeeded') {
    res.status(400).json({ error: 'Can only refund succeeded payments' });
    return;
  }

  try {
    const refund = await stripe.refunds.create({
      payment_intent: payment.rows[0].stripe_payment_intent_id,
      amount: amount ? Math.round(amount * 100) : undefined,
      reason: reason ?? 'requested_by_customer',
    });

    await pool.query(
      `INSERT INTO refunds (id, payment_id, stripe_refund_id, amount, status)
       VALUES (gen_random_uuid(), $1, $2, $3, $4)`,
      [id, refund.id, amount ?? payment.rows[0].amount, refund.status]
    );

    res.json({ refundId: refund.id, status: refund.status });
  } catch (error) {
    logger.error('Refund failed', { error, paymentId: id });
    res.status(500).json({ error: 'Refund failed' });
  }
});

// Stripe webhook handler
app.post('/webhooks/stripe', async (req, res): Promise<void> => {
  const sig = req.headers['stripe-signature'] as string;

  let event: Stripe.Event;
  try {
    event = stripe.webhooks.constructEvent(req.body, sig, process.env.STRIPE_WEBHOOK_SECRET!);
  } catch {
    res.status(400).send('Webhook signature verification failed');
    return;
  }

  switch (event.type) {
    case 'payment_intent.succeeded':
      const pi = event.data.object as Stripe.PaymentIntent;
      await pool.query(
        'UPDATE payments SET status = $1 WHERE stripe_payment_intent_id = $2',
        ['succeeded', pi.id]
      );
      break;

    case 'payment_intent.payment_failed':
      const failedPi = event.data.object as Stripe.PaymentIntent;
      await pool.query(
        'UPDATE payments SET status = $1 WHERE stripe_payment_intent_id = $2',
        ['failed', failedPi.id]
      );
      break;
  }

  res.json({ received: true });
});

app.get('/health', (_, res) => res.json({ status: 'ok', service: 'payment-service' }));

async function main() {
  await producer.connect();
  const PORT = process.env.PORT ?? 3004;
  app.listen(PORT, () => logger.info(`PaymentService on port ${PORT}`));
}

main().catch((err) => { logger.error('Startup failed', { err }); process.exit(1); });
```

---

## 99.5 Docker Compose: Infrastructure Complete

```yaml
# docker-compose.yml
version: '3.8'

services:
  # ============= Databases =============
  users-db:
    image: postgres:15-alpine
    container_name: users-db
    environment:
      POSTGRES_DB: users_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-secret123}
    volumes:
      - users-db-data:/var/lib/postgresql/data
      - ./init-scripts/users:/docker-entrypoint-initdb.d
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d users_db"]
      interval: 5s
      timeout: 5s
      retries: 10
    networks:
      - microservices

  products-db:
    image: postgres:15-alpine
    container_name: products-db
    environment:
      POSTGRES_DB: products_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-secret123}
    volumes:
      - products-db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d products_db"]
      interval: 5s
      timeout: 5s
      retries: 10
    networks:
      - microservices

  orders-db:
    image: postgres:15-alpine
    container_name: orders-db
    environment:
      POSTGRES_DB: orders_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-secret123}
    volumes:
      - orders-db-data:/var/lib/postgresql/data
      - ./init-scripts/orders:/docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d orders_db"]
      interval: 5s
      timeout: 5s
      retries: 10
    networks:
      - microservices

  payments-db:
    image: postgres:15-alpine
    container_name: payments-db
    environment:
      POSTGRES_DB: payments_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-secret123}
    volumes:
      - payments-db-data:/var/lib/postgresql/data
      - ./init-scripts/payments:/docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d payments_db"]
      interval: 5s
      timeout: 5s
      retries: 10
    networks:
      - microservices

  # ============= Redis =============
  redis:
    image: redis:7-alpine
    container_name: redis
    command: redis-server --requirepass ${REDIS_PASSWORD:-redis123} --maxmemory 256mb --maxmemory-policy allkeys-lru
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD:-redis123}", "ping"]
      interval: 5s
      timeout: 3s
      retries: 10
    networks:
      - microservices

  # ============= Kafka =============
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    container_name: zookeeper
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    volumes:
      - zookeeper-data:/var/lib/zookeeper/data
      - zookeeper-logs:/var/lib/zookeeper/log
    networks:
      - microservices
    healthcheck:
      test: ["CMD", "bash", "-c", "echo ruok | nc localhost 2181"]
      interval: 10s
      timeout: 5s
      retries: 5

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    container_name: kafka
    depends_on:
      zookeeper:
        condition: service_healthy
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_LOG_RETENTION_HOURS: 168
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"
    volumes:
      - kafka-data:/var/lib/kafka/data
    networks:
      - microservices
    healthcheck:
      test: ["CMD", "kafka-topics", "--bootstrap-server", "localhost:29092", "--list"]
      interval: 30s
      timeout: 10s
      retries: 5

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
    networks:
      - microservices

  # ============= Application Services =============
  user-service:
    build:
      context: ./user-service
      dockerfile: Dockerfile
    container_name: user-service
    ports:
      - "3001:3001"
    environment:
      NODE_ENV: development
      PORT: 3001
      DATABASE_URL: postgresql://postgres:${POSTGRES_PASSWORD:-secret123}@users-db:5432/users_db
      REDIS_URL: redis://:${REDIS_PASSWORD:-redis123}@redis:6379
      JWT_SECRET: ${JWT_SECRET:-your-super-secret-key-min-32-chars}
      JWT_REFRESH_SECRET: ${JWT_REFRESH_SECRET:-your-refresh-secret-key-min-32}
      ALLOWED_ORIGINS: http://localhost:3000
    depends_on:
      users-db:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3001/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s
    networks:
      - microservices
    restart: unless-stopped

  product-service:
    build:
      context: ./product-service
      dockerfile: Dockerfile
    container_name: product-service
    ports:
      - "3002:3002"
    environment:
      NODE_ENV: development
      PORT: 3002
      DATABASE_URL: postgresql://postgres:${POSTGRES_PASSWORD:-secret123}@products-db:5432/products_db
      REDIS_URL: redis://:${REDIS_PASSWORD:-redis123}@redis:6379
      USER_SERVICE_URL: http://user-service:3001
    depends_on:
      products-db:
        condition: service_healthy
      redis:
        condition: service_healthy
      user-service:
        condition: service_healthy
    networks:
      - microservices
    restart: unless-stopped

  order-service:
    build:
      context: ./order-service
      dockerfile: Dockerfile
    container_name: order-service
    ports:
      - "3003:3003"
    environment:
      NODE_ENV: development
      PORT: 3003
      DATABASE_URL: postgresql://postgres:${POSTGRES_PASSWORD:-secret123}@orders-db:5432/orders_db
      KAFKA_BROKERS: kafka:29092
      USER_SERVICE_URL: http://user-service:3001
      PRODUCT_SERVICE_URL: http://product-service:3002
    depends_on:
      orders-db:
        condition: service_healthy
      kafka:
        condition: service_healthy
    networks:
      - microservices
    restart: unless-stopped

  payment-service:
    build:
      context: ./payment-service
      dockerfile: Dockerfile
    container_name: payment-service
    ports:
      - "3004:3004"
    environment:
      NODE_ENV: development
      PORT: 3004
      DATABASE_URL: postgresql://postgres:${POSTGRES_PASSWORD:-secret123}@payments-db:5432/payments_db
      KAFKA_BROKERS: kafka:29092
      STRIPE_SECRET_KEY: ${STRIPE_SECRET_KEY}
      STRIPE_WEBHOOK_SECRET: ${STRIPE_WEBHOOK_SECRET}
      FRONTEND_URL: http://localhost:3000
    depends_on:
      payments-db:
        condition: service_healthy
      kafka:
        condition: service_healthy
    networks:
      - microservices
    restart: unless-stopped

  notification-service:
    build:
      context: ./notification-service
      dockerfile: Dockerfile
    container_name: notification-service
    ports:
      - "3005:3005"
    environment:
      NODE_ENV: development
      PORT: 3005
      KAFKA_BROKERS: kafka:29092
      SMTP_HOST: ${SMTP_HOST:-mailhog}
      SMTP_PORT: 1025
    depends_on:
      kafka:
        condition: service_healthy
    networks:
      - microservices
    restart: unless-stopped

  # ============= Monitoring =============
  prometheus:
    image: prom/prometheus:v2.48.0
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--storage.tsdb.retention.time=30d'
    networks:
      - microservices
    restart: unless-stopped

  grafana:
    image: grafana/grafana:10.2.0
    container_name: grafana
    ports:
      - "3006:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASSWORD:-admin}
      GF_USERS_ALLOW_SIGN_UP: "false"
    volumes:
      - grafana-data:/var/lib/grafana
      - ./monitoring/grafana/dashboards:/etc/grafana/provisioning/dashboards:ro
      - ./monitoring/grafana/datasources:/etc/grafana/provisioning/datasources:ro
    depends_on:
      - prometheus
    networks:
      - microservices
    restart: unless-stopped

  jaeger:
    image: jaegertracing/all-in-one:1.51
    container_name: jaeger
    ports:
      - "16686:16686"
      - "14268:14268"
    environment:
      COLLECTOR_ZIPKIN_HOST_PORT: :9411
    networks:
      - microservices
    restart: unless-stopped

  # Development tools
  mailhog:
    image: mailhog/mailhog:v1.0.1
    container_name: mailhog
    ports:
      - "1025:1025"
      - "8025:8025"
    networks:
      - microservices

volumes:
  users-db-data:
  products-db-data:
  orders-db-data:
  payments-db-data:
  redis-data:
  kafka-data:
  zookeeper-data:
  zookeeper-logs:
  prometheus-data:
  grafana-data:

networks:
  microservices:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
```

---

## 99.6 Kubernetes Deployment YAML

```yaml
# k8s/user-service/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  namespace: production
  labels:
    app: user-service
    version: "1.0.0"
    team: platform
spec:
  replicas: 3
  selector:
    matchLabels:
      app: user-service
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: user-service
        version: "1.0.0"
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "3001"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: user-service
      terminationGracePeriodSeconds: 30
      containers:
        - name: user-service
          image: myregistry.azurecr.io/user-service:1.0.0
          ports:
            - containerPort: 3001
              name: http
          envFrom:
            - configMapRef:
                name: user-service-config
            - secretRef:
                name: user-service-secrets
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          livenessProbe:
            httpGet:
              path: /health
              port: 3001
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /health
              port: 3001
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 3
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 5"]
          securityContext:
            runAsNonRoot: true
            runAsUser: 1001
            readOnlyRootFilesystem: true
            allowPrivilegeEscalation: false
            capabilities:
              drop:
                - ALL
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - user-service
                topologyKey: kubernetes.io/hostname
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: user-service
---
apiVersion: v1
kind: Service
metadata:
  name: user-service
  namespace: production
spec:
  selector:
    app: user-service
  ports:
    - port: 80
      targetPort: 3001
      name: http
  type: ClusterIP
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: user-service-config
  namespace: production
data:
  NODE_ENV: "production"
  PORT: "3001"
  ALLOWED_ORIGINS: "https://myshop.example.com"
  LOG_LEVEL: "info"
---
apiVersion: v1
kind: Secret
metadata:
  name: user-service-secrets
  namespace: production
type: Opaque
stringData:
  DATABASE_URL: "postgresql://postgres:$(POSTGRES_PASSWORD)@users-db:5432/users_db"
  REDIS_URL: "redis://:$(REDIS_PASSWORD)@redis:6379"
  JWT_SECRET: "$(JWT_SECRET)"
  JWT_REFRESH_SECRET: "$(JWT_REFRESH_SECRET)"
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: user-service
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: user-service
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 1
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Pods
          value: 2
          periodSeconds: 60
---
# NetworkPolicy: only allow traffic from API gateway and other services
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: user-service-network-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: user-service
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: api-gateway
        - podSelector:
            matchLabels:
              app: order-service
      ports:
        - protocol: TCP
          port: 3001
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: users-db
      ports:
        - protocol: TCP
          port: 5432
    - to:
        - podSelector:
            matchLabels:
              app: redis
      ports:
        - protocol: TCP
          port: 6379
    - to:
        - namespaceSelector: {}
      ports:
        - protocol: TCP
          port: 53
        - protocol: UDP
          port: 53
```

---

## 99.7 GitHub Actions CI/CD Pipeline

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches:
      - main
      - 'release/**'
  pull_request:
    branches:
      - main

env:
  REGISTRY: myregistry.azurecr.io
  CLUSTER_NAME: production-cluster
  CLUSTER_RESOURCE_GROUP: production-rg

jobs:
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      user-service: ${{ steps.filter.outputs.user-service }}
      order-service: ${{ steps.filter.outputs.order-service }}
      payment-service: ${{ steps.filter.outputs.payment-service }}
      product-service: ${{ steps.filter.outputs.product-service }}
      notification-service: ${{ steps.filter.outputs.notification-service }}
    steps:
      - uses: actions/checkout@v4
      - uses: dorny/paths-filter@v2
        id: filter
        with:
          filters: |
            user-service:
              - 'user-service/**'
            order-service:
              - 'order-service/**'
            payment-service:
              - 'payment-service/**'
            product-service:
              - 'product-service/**'
            notification-service:
              - 'notification-service/**'

  test:
    runs-on: ubuntu-latest
    needs: detect-changes
    strategy:
      matrix:
        service:
          - name: user-service
            changed: ${{ needs.detect-changes.outputs.user-service }}
          - name: order-service
            changed: ${{ needs.detect-changes.outputs.order-service }}
          - name: payment-service
            changed: ${{ needs.detect-changes.outputs.payment-service }}
    steps:
      - uses: actions/checkout@v4
        if: matrix.service.changed == 'true'

      - name: Setup Node.js
        if: matrix.service.changed == 'true'
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
          cache-dependency-path: '${{ matrix.service.name }}/package-lock.json'

      - name: Install dependencies
        if: matrix.service.changed == 'true'
        run: npm ci
        working-directory: ${{ matrix.service.name }}

      - name: Run linter
        if: matrix.service.changed == 'true'
        run: npm run lint
        working-directory: ${{ matrix.service.name }}

      - name: Run type check
        if: matrix.service.changed == 'true'
        run: npm run typecheck
        working-directory: ${{ matrix.service.name }}

      - name: Run unit tests
        if: matrix.service.changed == 'true'
        run: npm run test:unit -- --coverage
        working-directory: ${{ matrix.service.name }}

      - name: Run integration tests
        if: matrix.service.changed == 'true'
        run: |
          docker compose -f docker-compose.test.yml up -d
          sleep 10
          npm run test:integration
          docker compose -f docker-compose.test.yml down
        working-directory: ${{ matrix.service.name }}

      - name: Upload coverage
        if: matrix.service.changed == 'true'
        uses: codecov/codecov-action@v3
        with:
          directory: ${{ matrix.service.name }}/coverage

  security-scan:
    runs-on: ubuntu-latest
    needs: detect-changes
    strategy:
      matrix:
        service: [user-service, order-service, payment-service]
    steps:
      - uses: actions/checkout@v4

      - name: Run Trivy security scan
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '${{ matrix.service }}'
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'

      - name: Upload scan results
        uses: github/codeql-action/upload-sarif@v2
        if: always()
        with:
          sarif_file: 'trivy-results.sarif'

  build-and-push:
    runs-on: ubuntu-latest
    needs: [test, security-scan]
    if: github.ref == 'refs/heads/main'
    strategy:
      matrix:
        service: [user-service, order-service, payment-service, product-service, notification-service]
    outputs:
      image-digest: ${{ steps.push.outputs.digest }}
    steps:
      - uses: actions/checkout@v4

      - name: Login to Container Registry
        uses: azure/docker-login@v1
        with:
          login-server: ${{ env.REGISTRY }}
          username: ${{ secrets.REGISTRY_USERNAME }}
          password: ${{ secrets.REGISTRY_PASSWORD }}

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ matrix.service }}
          tags: |
            type=sha,prefix=,suffix=,format=short
            type=semver,pattern={{version}}
            type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}

      - name: Build and push Docker image
        id: push
        uses: docker/build-push-action@v5
        with:
          context: ./${{ matrix.service }}
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          platforms: linux/amd64,linux/arm64
          provenance: true
          sbom: true

      - name: Sign container image
        run: |
          cosign sign --yes ${{ env.REGISTRY }}/${{ matrix.service }}@${{ steps.push.outputs.digest }}

  deploy-staging:
    runs-on: ubuntu-latest
    needs: build-and-push
    environment: staging
    steps:
      - uses: actions/checkout@v4

      - name: Setup kubectl
        uses: azure/setup-kubectl@v3

      - name: Get AKS credentials
        uses: azure/aks-set-context@v3
        with:
          resource-group: staging-rg
          cluster-name: staging-cluster
          admin: false

      - name: Deploy to staging
        run: |
          IMAGE_TAG=$(echo ${{ github.sha }} | cut -c1-7)
          for SERVICE in user-service order-service payment-service product-service notification-service; do
            kubectl set image deployment/$SERVICE \
              $SERVICE=${{ env.REGISTRY }}/$SERVICE:$IMAGE_TAG \
              -n staging
            kubectl rollout status deployment/$SERVICE -n staging --timeout=5m
          done

      - name: Run smoke tests
        run: |
          npm ci
          npm run test:smoke -- --env=staging
        working-directory: tests

      - name: Notify deployment
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "Staging deployment completed for commit ${{ github.sha }}"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

  deploy-production:
    runs-on: ubuntu-latest
    needs: deploy-staging
    environment: production
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4

      - name: Setup kubectl
        uses: azure/setup-kubectl@v3

      - name: Get AKS credentials
        uses: azure/aks-set-context@v3
        with:
          resource-group: ${{ env.CLUSTER_RESOURCE_GROUP }}
          cluster-name: ${{ env.CLUSTER_NAME }}
          admin: false

      - name: Deploy to production (canary)
        run: |
          IMAGE_TAG=$(echo ${{ github.sha }} | cut -c1-7)
          for SERVICE in user-service order-service payment-service product-service notification-service; do
            # Deploy canary (10% traffic)
            helm upgrade --install $SERVICE-canary ./helm/charts/$SERVICE \
              --namespace production \
              --set image.tag=$IMAGE_TAG \
              --set replicaCount=1 \
              --set canary.enabled=true \
              --set canary.weight=10 \
              --wait

            echo "Canary deployed for $SERVICE"
          done

      - name: Monitor canary (5 minutes)
        run: |
          sleep 300
          # Check error rates
          for SERVICE in user-service order-service payment-service; do
            ERROR_RATE=$(curl -s "https://prometheus.example.com/api/v1/query?query=rate(http_requests_total{service=\"$SERVICE\",code=~\"5..\"}[5m])/rate(http_requests_total{service=\"$SERVICE\"}[5m])" | jq '.data.result[0].value[1]' -r)
            if (( $(echo "$ERROR_RATE > 0.05" | bc -l) )); then
              echo "ERROR: $SERVICE canary error rate is $ERROR_RATE - aborting!"
              # Rollback
              helm rollback $SERVICE-canary -n production
              exit 1
            fi
            echo "$SERVICE canary OK: $ERROR_RATE error rate"
          done

      - name: Promote canary to full production
        run: |
          IMAGE_TAG=$(echo ${{ github.sha }} | cut -c1-7)
          for SERVICE in user-service order-service payment-service product-service notification-service; do
            helm upgrade $SERVICE ./helm/charts/$SERVICE \
              --namespace production \
              --set image.tag=$IMAGE_TAG \
              --set canary.enabled=false \
              --wait
          done
```

---

## 99.8 Step-by-Step Getting Started Guide

### Prerequisites

ก่อนเริ่มต้น ตรวจสอบว่ามีเครื่องมือเหล่านี้:

```bash
# ตรวจสอบ versions
node --version        # >= 20.x
npm --version         # >= 9.x
docker --version      # >= 24.x
docker compose version # >= 2.x
kubectl version       # >= 1.28.x
```

### Step 1: Clone และ Setup

```bash
git clone https://github.com/your-org/microservices-ecommerce.git
cd microservices-ecommerce

# Copy environment files
cp .env.example .env

# แก้ไขค่าใน .env
# - POSTGRES_PASSWORD
# - REDIS_PASSWORD
# - JWT_SECRET (ต้องมี >= 32 chars)
# - JWT_REFRESH_SECRET
# - STRIPE_SECRET_KEY (จาก https://dashboard.stripe.com/test/apikeys)
```

### Step 2: Start Infrastructure

```bash
# Start all infrastructure services first
docker compose up -d users-db products-db orders-db payments-db redis zookeeper kafka

# รอให้ services ready (ประมาณ 30 วินาที)
docker compose ps

# ตรวจสอบ health
docker compose exec users-db pg_isready -U postgres
docker compose exec redis redis-cli ping
```

### Step 3: Run Database Migrations

```bash
# User Service migrations
cd user-service
npm install
DATABASE_URL=postgresql://postgres:secret123@localhost:5432/users_db npx prisma migrate dev --name init
npx prisma generate

# Orders Service migrations
cd ../order-service
npm install
psql postgresql://postgres:secret123@localhost:5432/orders_db < migrations/001_init.sql

# Payments Service migrations
cd ../payment-service
npm install
psql postgresql://postgres:secret123@localhost:5432/payments_db < migrations/001_init.sql

cd ..
```

### Step 4: Start All Services

```bash
# Build all services
docker compose build

# Start everything
docker compose up -d

# ดู logs
docker compose logs -f user-service order-service payment-service

# ตรวจสอบ health ของทุก service
for service in user-service product-service order-service payment-service notification-service; do
  echo -n "$service: "
  curl -s http://localhost:$(docker compose port $service 2>/dev/null | cut -d: -f2)/health | jq '.status' 2>/dev/null || echo "not ready"
done
```

### Step 5: Test the System

```bash
# Register a user
curl -X POST http://localhost:3001/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "test@example.com",
    "password": "password123",
    "firstName": "Test",
    "lastName": "User"
  }'

# Login
TOKEN=$(curl -s -X POST http://localhost:3001/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "test@example.com", "password": "password123"}' \
  | jq -r '.accessToken')

echo "JWT Token: $TOKEN"

# Create an order
curl -X POST http://localhost:3003/orders \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "items": [{"productId": "prod-001", "quantity": 2, "price": 29.99}],
    "shippingAddressId": "addr-001"
  }'
```

### Step 6: Access Monitoring Tools

| Tool | URL | Credentials |
|------|-----|-------------|
| Grafana | http://localhost:3006 | admin / admin |
| Prometheus | http://localhost:9090 | - |
| Jaeger (Tracing) | http://localhost:16686 | - |
| Kafka UI | http://localhost:8090 | - |
| Mailhog (Email) | http://localhost:8025 | - |

### Step 7: Deploy to Kubernetes

```bash
# Create namespace
kubectl create namespace production

# Create secrets (ใช้ Sealed Secrets หรือ External Secrets ใน production จริง)
kubectl create secret generic user-service-secrets \
  --from-literal=DATABASE_URL="postgresql://postgres:secret@users-db:5432/users_db" \
  --from-literal=JWT_SECRET="your-jwt-secret-here-min-32-chars" \
  --from-literal=JWT_REFRESH_SECRET="your-refresh-secret-here" \
  --from-literal=REDIS_URL="redis://:redis123@redis:6379" \
  -n production

# Deploy with Helm
helm install user-service ./helm/charts/user-service \
  --namespace production \
  --values ./helm/values/production/user-service.yaml

# Check deployment
kubectl get pods -n production
kubectl get svc -n production
```

### Troubleshooting

```bash
# Service ไม่ start
docker compose logs <service-name>

# Database connection failed
docker compose exec <service-name> env | grep DATABASE_URL
docker compose exec users-db pg_isready -U postgres

# Kafka connection issues
docker compose exec kafka kafka-topics --bootstrap-server localhost:29092 --list

# Pod crashlooping ใน Kubernetes
kubectl describe pod <pod-name> -n production
kubectl logs <pod-name> -n production --previous

# ดู events
kubectl get events -n production --sort-by='.metadata.creationTimestamp'
```

---

## 99.9 สรุปโปรเจกต์

ระบบ e-commerce microservices ที่เราสร้างมีคุณสมบัติ:

### Reliability
- Health checks ทุก service
- Graceful shutdown
- Retry logic สำหรับ external calls
- Circuit breakers (ผ่าน Istio หรือ Resilience4j)

### Scalability
- Kubernetes HPA สำหรับ horizontal scaling
- Stateless services
- Redis สำหรับ session storage
- Kafka สำหรับ async communication

### Security
- JWT authentication
- Rate limiting
- NetworkPolicy ใน Kubernetes
- Secret management

### Observability
- Prometheus metrics
- Structured logging
- Distributed tracing (Jaeger)
- Grafana dashboards

### DevOps
- GitHub Actions CI/CD
- Canary deployments
- Docker multi-stage builds
- Kubernetes deployments with rollback

---

*ต่อไปใน Part 100: Course Summary and Next Steps - สรุปทุกสิ่งที่เรียนมาตลอดหลักสูตร 100 Parts*

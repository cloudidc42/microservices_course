# Part 99: Final Project - Complete Microservices System

## บทนำ

Final Project นี้จะรวบรวมทุกสิ่งที่เราเรียนมาตลอดคอร์สนี้ เราจะสร้าง E-Commerce Platform ที่ประกอบด้วย 6 Services: User, Product, Order, Payment, Notification และ Search โดยแต่ละ Service จะใช้ Best Practices ที่เหมาะสม

---

## 1. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     E-COMMERCE MICROSERVICES PLATFORM                        │
│                                                                               │
│  Client (Web/Mobile)                                                          │
│       │                                                                       │
│  ┌────▼─────────────────────────────────────────────────────────┐            │
│  │                    API Gateway (Kong)                          │            │
│  │  Rate Limiting │ Auth │ CORS │ Load Balance │ Metrics          │            │
│  └────┬─────────────┬───────────┬──────────────┬────────────────┘            │
│       │             │           │              │                              │
│  ┌────▼───┐  ┌──────▼───┐  ┌───▼──────┐  ┌───▼──────┐                     │
│  │  User  │  │ Product  │  │  Order   │  │ Payment  │                     │
│  │Service │  │ Service  │  │ Service  │  │ Service  │                     │
│  │:3001   │  │ :3002    │  │ :3003    │  │ :3004    │                     │
│  └────────┘  └──────────┘  └────┬─────┘  └──────────┘                     │
│                                  │                                            │
│                    ┌─────────────▼────────────────┐                          │
│                    │        Apache Kafka           │                          │
│                    │  (order.created, payment.*)   │                          │
│                    └──────┬───────────────┬────────┘                          │
│                           │               │                                   │
│                   ┌───────▼───┐   ┌───────▼───────┐                         │
│                   │Notification│   │Search Service │                         │
│                   │Service :3005│   │:3006(Elastic) │                         │
│                   └───────────┘   └───────────────┘                         │
│                                                                               │
│  Infrastructure:                                                              │
│  PostgreSQL (users, products, orders, payments)                               │
│  Redis (sessions, cache, rate limiting)                                       │
│  Kafka (async events)                                                         │
│  Elasticsearch (search index)                                                 │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Complete TypeScript UserService

```typescript
// services/user/src/index.ts
import express from 'express';
import { json } from 'express';
import { PrismaClient } from '@prisma/client';
import { createClient } from 'redis';
import { sign, verify } from 'jsonwebtoken';
import { hash, compare } from 'bcryptjs';
import { z } from 'zod';
import promClient from 'prom-client';

const app = express();
app.use(json());

const prisma = new PrismaClient();
const redis = createClient({ url: process.env.REDIS_URL });
redis.connect();

// Prometheus metrics
const register = new promClient.Registry();
promClient.collectDefaultMetrics({ register });

const httpRequestsTotal = new promClient.Counter({
  name: 'http_requests_total',
  help: 'Total HTTP requests',
  labelNames: ['method', 'route', 'status_code'],
  registers: [register],
});

const httpDuration = new promClient.Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration',
  labelNames: ['method', 'route'],
  buckets: [0.01, 0.05, 0.1, 0.3, 0.5, 1, 2, 5],
  registers: [register],
});

// Schemas
const RegisterSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
  name: z.string().min(2).max(100),
  phone: z.string().optional(),
});

const LoginSchema = z.object({
  email: z.string().email(),
  password: z.string(),
});

// Middleware
const metricsMiddleware = (req: express.Request, res: express.Response, next: express.NextFunction) => {
  const end = httpDuration.startTimer({ method: req.method, route: req.route?.path || req.path });
  res.on('finish', () => {
    httpRequestsTotal.inc({ method: req.method, route: req.route?.path || req.path, status_code: res.statusCode });
    end();
  });
  next();
};

app.use(metricsMiddleware);

const authenticateJWT = async (req: express.Request, res: express.Response, next: express.NextFunction) => {
  const token = req.headers.authorization?.replace('Bearer ', '');
  if (!token) return res.status(401).json({ error: 'Token required' });

  try {
    // Check blacklist
    const isBlacklisted = await redis.get(`blacklist:${token}`);
    if (isBlacklisted) return res.status(401).json({ error: 'Token revoked' });

    const payload = verify(token, process.env.JWT_SECRET!) as any;
    (req as any).user = payload;
    next();
  } catch {
    res.status(401).json({ error: 'Invalid token' });
  }
};

// Routes
app.post('/api/v1/auth/register', async (req, res) => {
  try {
    const body = RegisterSchema.parse(req.body);

    const existing = await prisma.user.findUnique({ where: { email: body.email } });
    if (existing) return res.status(409).json({ error: 'Email already registered' });

    const passwordHash = await hash(body.password, 12);

    const user = await prisma.user.create({
      data: {
        email: body.email,
        passwordHash,
        name: body.name,
        phone: body.phone,
        role: 'customer',
      },
      select: { id: true, email: true, name: true, role: true, createdAt: true },
    });

    const token = sign(
      { userId: user.id, email: user.email, role: user.role },
      process.env.JWT_SECRET!,
      { expiresIn: '7d', issuer: 'user-service' }
    );

    // Cache user profile
    await redis.setEx(`user:${user.id}`, 3600, JSON.stringify(user));

    res.status(201).json({ user, token });
  } catch (error) {
    if (error instanceof z.ZodError) {
      return res.status(400).json({ error: 'Validation failed', details: error.errors });
    }
    console.error('Register error:', error);
    res.status(500).json({ error: 'Registration failed' });
  }
});

app.post('/api/v1/auth/login', async (req, res) => {
  try {
    const { email, password } = LoginSchema.parse(req.body);

    // Rate limit login attempts
    const attemptKey = `login_attempts:${email}`;
    const attempts = await redis.incr(attemptKey);
    if (attempts === 1) await redis.expire(attemptKey, 900); // 15 min window
    if (attempts > 5) {
      return res.status(429).json({ error: 'Too many login attempts. Try again in 15 minutes.' });
    }

    const user = await prisma.user.findUnique({ where: { email } });
    if (!user) return res.status(401).json({ error: 'Invalid credentials' });

    const isValid = await compare(password, user.passwordHash);
    if (!isValid) return res.status(401).json({ error: 'Invalid credentials' });

    // Clear attempts on success
    await redis.del(attemptKey);

    const accessToken = sign(
      { userId: user.id, email: user.email, role: user.role },
      process.env.JWT_SECRET!,
      { expiresIn: '1h', issuer: 'user-service' }
    );

    const refreshToken = sign(
      { userId: user.id, type: 'refresh' },
      process.env.JWT_REFRESH_SECRET!,
      { expiresIn: '30d' }
    );

    // Store refresh token
    await redis.setEx(`refresh:${user.id}`, 30 * 86400, refreshToken);

    await prisma.user.update({
      where: { id: user.id },
      data: { lastLoginAt: new Date() },
    });

    res.json({
      accessToken,
      refreshToken,
      user: { id: user.id, email: user.email, name: user.name, role: user.role },
    });
  } catch (error) {
    if (error instanceof z.ZodError) {
      return res.status(400).json({ error: 'Validation failed', details: error.errors });
    }
    res.status(500).json({ error: 'Login failed' });
  }
});

app.post('/api/v1/auth/refresh', async (req, res) => {
  try {
    const { refreshToken } = req.body;
    if (!refreshToken) return res.status(400).json({ error: 'Refresh token required' });

    const payload = verify(refreshToken, process.env.JWT_REFRESH_SECRET!) as any;
    if (payload.type !== 'refresh') return res.status(401).json({ error: 'Invalid token' });

    const stored = await redis.get(`refresh:${payload.userId}`);
    if (stored !== refreshToken) return res.status(401).json({ error: 'Token expired or revoked' });

    const user = await prisma.user.findUnique({ where: { id: payload.userId } });
    if (!user) return res.status(401).json({ error: 'User not found' });

    const newAccessToken = sign(
      { userId: user.id, email: user.email, role: user.role },
      process.env.JWT_SECRET!,
      { expiresIn: '1h' }
    );

    res.json({ accessToken: newAccessToken });
  } catch {
    res.status(401).json({ error: 'Invalid refresh token' });
  }
});

app.post('/api/v1/auth/logout', authenticateJWT, async (req, res) => {
  const user = (req as any).user;
  const token = req.headers.authorization?.replace('Bearer ', '')!;

  // Blacklist token
  await redis.setEx(`blacklist:${token}`, 3600, '1');
  await redis.del(`refresh:${user.userId}`);

  res.json({ message: 'Logged out successfully' });
});

app.get('/api/v1/users/me', authenticateJWT, async (req, res) => {
  const { userId } = (req as any).user;

  // Try cache first
  const cached = await redis.get(`user:${userId}`);
  if (cached) return res.json(JSON.parse(cached));

  const user = await prisma.user.findUnique({
    where: { id: userId },
    select: { id: true, email: true, name: true, phone: true, role: true, createdAt: true },
  });

  if (!user) return res.status(404).json({ error: 'User not found' });

  await redis.setEx(`user:${userId}`, 3600, JSON.stringify(user));
  res.json(user);
});

app.patch('/api/v1/users/me', authenticateJWT, async (req, res) => {
  const { userId } = (req as any).user;
  const UpdateSchema = z.object({
    name: z.string().min(2).max(100).optional(),
    phone: z.string().optional(),
  });

  try {
    const updates = UpdateSchema.parse(req.body);
    const user = await prisma.user.update({
      where: { id: userId },
      data: updates,
      select: { id: true, email: true, name: true, phone: true, role: true },
    });

    await redis.setEx(`user:${userId}`, 3600, JSON.stringify(user));
    res.json(user);
  } catch (error) {
    if (error instanceof z.ZodError) {
      return res.status(400).json({ error: 'Validation failed', details: error.errors });
    }
    res.status(500).json({ error: 'Update failed' });
  }
});

// Internal endpoint for other services
app.get('/internal/users/:id', async (req, res) => {
  const internalKey = req.headers['x-internal-key'];
  if (internalKey !== process.env.INTERNAL_API_KEY) {
    return res.status(401).json({ error: 'Unauthorized' });
  }

  const user = await prisma.user.findUnique({
    where: { id: req.params.id },
    select: { id: true, email: true, name: true, role: true },
  });

  if (!user) return res.status(404).json({ error: 'User not found' });
  res.json(user);
});

app.get('/health', async (_req, res) => {
  try {
    await prisma.$queryRaw`SELECT 1`;
    await redis.ping();
    res.json({ status: 'healthy', service: 'user-service', timestamp: new Date() });
  } catch (error) {
    res.status(503).json({ status: 'unhealthy', error: String(error) });
  }
});

app.get('/metrics', async (_req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});

const PORT = process.env.PORT || 3001;
const server = app.listen(PORT, () => {
  console.log(`User service listening on port ${PORT}`);
});

// Graceful shutdown
process.on('SIGTERM', async () => {
  server.close(async () => {
    await prisma.$disconnect();
    await redis.quit();
    process.exit(0);
  });
});
```

---

## 3. Complete TypeScript OrderService

```typescript
// services/order/src/index.ts
import express from 'express';
import { PrismaClient } from '@prisma/client';
import { Kafka, Producer, Partitioners } from 'kafkajs';
import { createClient } from 'redis';
import { z } from 'zod';
import { v4 as uuidv4 } from 'uuid';
import axios from 'axios';

const app = express();
app.use(express.json());

const prisma = new PrismaClient();
const redis = createClient({ url: process.env.REDIS_URL });
redis.connect();

const kafka = new Kafka({
  clientId: 'order-service',
  brokers: (process.env.KAFKA_BROKERS || 'kafka:9092').split(','),
});
const producer: Producer = kafka.producer({
  createPartitioner: Partitioners.LegacyPartitioner,
  idempotent: true,
  maxInFlightRequests: 5,
  transactionTimeout: 30000,
});

producer.connect().then(() => console.log('Kafka producer connected'));

// Schemas
const CreateOrderSchema = z.object({
  items: z.array(z.object({
    productId: z.string().uuid(),
    quantity: z.number().int().min(1),
  })).min(1).max(50),
  shippingAddress: z.object({
    street: z.string(),
    city: z.string(),
    province: z.string(),
    postalCode: z.string(),
    country: z.string().default('TH'),
  }),
  paymentMethod: z.enum(['credit_card', 'promptpay', 'wallet', 'cod']),
  couponCode: z.string().optional(),
  notes: z.string().max(500).optional(),
});

// Order state machine
type OrderStatus = 'pending' | 'confirmed' | 'processing' | 'shipped' | 'delivered' | 'cancelled' | 'refunded';

const VALID_TRANSITIONS: Record<OrderStatus, OrderStatus[]> = {
  pending: ['confirmed', 'cancelled'],
  confirmed: ['processing', 'cancelled'],
  processing: ['shipped', 'cancelled'],
  shipped: ['delivered'],
  delivered: ['refunded'],
  cancelled: [],
  refunded: [],
};

const authenticateJWT = async (req: express.Request, res: express.Response, next: express.NextFunction) => {
  const token = req.headers.authorization?.replace('Bearer ', '');
  if (!token) return res.status(401).json({ error: 'Token required' });

  try {
    const response = await axios.post(
      `${process.env.USER_SERVICE_URL}/internal/validate-token`,
      { token },
      { headers: { 'x-internal-key': process.env.INTERNAL_API_KEY } }
    );
    (req as any).user = response.data;
    next();
  } catch {
    res.status(401).json({ error: 'Invalid token' });
  }
};

// Create order with Saga pattern
app.post('/api/v1/orders', authenticateJWT, async (req, res) => {
  try {
    const body = CreateOrderSchema.parse(req.body);
    const { userId } = (req as any).user;
    const orderId = uuidv4();
    const sagaId = uuidv4();

    // Step 1: Validate products and get prices
    let totalAmount = 0;
    const orderItems: Array<{
      productId: string;
      productName: string;
      quantity: number;
      unitPrice: number;
      totalPrice: number;
    }> = [];

    for (const item of body.items) {
      const productResponse = await axios.get(
        `${process.env.PRODUCT_SERVICE_URL}/internal/products/${item.productId}`,
        { headers: { 'x-internal-key': process.env.INTERNAL_API_KEY } }
      );
      const product = productResponse.data;

      if (!product.inStock || product.stockQuantity < item.quantity) {
        return res.status(400).json({
          error: `Product ${product.name} is out of stock or insufficient quantity`,
        });
      }

      const itemTotal = product.price * item.quantity;
      totalAmount += itemTotal;
      orderItems.push({
        productId: item.productId,
        productName: product.name,
        quantity: item.quantity,
        unitPrice: product.price,
        totalPrice: itemTotal,
      });
    }

    // Step 2: Apply coupon if provided
    let discountAmount = 0;
    if (body.couponCode) {
      const couponResponse = await axios.post(
        `${process.env.PRODUCT_SERVICE_URL}/internal/coupons/validate`,
        { code: body.couponCode, amount: totalAmount },
        { headers: { 'x-internal-key': process.env.INTERNAL_API_KEY } }
      ).catch(() => null);

      if (couponResponse?.data?.valid) {
        discountAmount = couponResponse.data.discountAmount;
      }
    }

    const finalAmount = totalAmount - discountAmount;

    // Step 3: Create order in DB (pending state)
    const order = await prisma.order.create({
      data: {
        id: orderId,
        userId,
        status: 'pending',
        totalAmount,
        discountAmount,
        finalAmount,
        shippingAddress: body.shippingAddress as any,
        paymentMethod: body.paymentMethod,
        couponCode: body.couponCode,
        notes: body.notes,
        sagaId,
        items: {
          create: orderItems,
        },
      },
      include: { items: true },
    });

    // Step 4: Reserve inventory (Saga step)
    await producer.send({
      topic: 'order.inventory.reserve',
      messages: [{
        key: orderId,
        value: JSON.stringify({
          sagaId,
          orderId,
          items: orderItems.map(i => ({ productId: i.productId, quantity: i.quantity })),
        }),
        headers: { event_type: 'INVENTORY_RESERVE_REQUEST', saga_id: sagaId },
      }],
    });

    // Step 5: Publish order.created event
    await producer.send({
      topic: 'order.created',
      messages: [{
        key: orderId,
        value: JSON.stringify({
          orderId: order.id,
          userId: order.userId,
          items: orderItems,
          totalAmount: order.totalAmount,
          finalAmount: order.finalAmount,
          paymentMethod: order.paymentMethod,
          shippingAddress: order.shippingAddress,
          createdAt: order.createdAt.toISOString(),
        }),
        headers: { event_type: 'ORDER_CREATED' },
      }],
    });

    res.status(201).json({
      orderId: order.id,
      status: order.status,
      totalAmount: order.totalAmount,
      finalAmount: order.finalAmount,
      items: order.items,
      estimatedDelivery: this.calculateDeliveryDate(body.shippingAddress.province),
    });
  } catch (error) {
    if (error instanceof z.ZodError) {
      return res.status(400).json({ error: 'Validation failed', details: error.errors });
    }
    console.error('Create order error:', error);
    res.status(500).json({ error: 'Failed to create order' });
  }
});

app.get('/api/v1/orders', authenticateJWT, async (req, res) => {
  const { userId } = (req as any).user;
  const page = parseInt(req.query.page as string) || 1;
  const limit = Math.min(parseInt(req.query.limit as string) || 10, 50);

  const [orders, total] = await Promise.all([
    prisma.order.findMany({
      where: { userId },
      include: { items: true },
      orderBy: { createdAt: 'desc' },
      skip: (page - 1) * limit,
      take: limit,
    }),
    prisma.order.count({ where: { userId } }),
  ]);

  res.json({
    orders,
    pagination: { page, limit, total, pages: Math.ceil(total / limit) },
  });
});

app.get('/api/v1/orders/:orderId', authenticateJWT, async (req, res) => {
  const { userId } = (req as any).user;

  const order = await prisma.order.findFirst({
    where: { id: req.params.orderId, userId },
    include: { items: true },
  });

  if (!order) return res.status(404).json({ error: 'Order not found' });
  res.json(order);
});

app.patch('/api/v1/orders/:orderId/status', authenticateJWT, async (req, res) => {
  const { userId, role } = (req as any).user;
  const { status } = req.body as { status: OrderStatus };

  const order = await prisma.order.findFirst({
    where: { id: req.params.orderId, ...(role !== 'admin' ? { userId } : {}) },
  });

  if (!order) return res.status(404).json({ error: 'Order not found' });

  const validNext = VALID_TRANSITIONS[order.status as OrderStatus];
  if (!validNext.includes(status)) {
    return res.status(400).json({
      error: `Cannot transition from ${order.status} to ${status}`,
      validTransitions: validNext,
    });
  }

  const updated = await prisma.order.update({
    where: { id: order.id },
    data: {
      status,
      ...(status === 'cancelled' ? { cancelledAt: new Date() } : {}),
      ...(status === 'delivered' ? { deliveredAt: new Date() } : {}),
    },
    include: { items: true },
  });

  // Publish status change event
  await producer.send({
    topic: 'order.status.changed',
    messages: [{
      key: order.id,
      value: JSON.stringify({
        orderId: order.id,
        userId: order.userId,
        previousStatus: order.status,
        newStatus: status,
        timestamp: new Date().toISOString(),
      }),
      headers: { event_type: 'ORDER_STATUS_CHANGED' },
    }],
  });

  res.json(updated);
});

// Kafka consumer for Saga compensation
const consumer = kafka.consumer({ groupId: 'order-saga-group' });

async function setupSagaConsumer(): Promise<void> {
  await consumer.connect();
  await consumer.subscribe({
    topics: ['order.inventory.reserved', 'order.inventory.reserve_failed', 'payment.completed', 'payment.failed'],
  });

  await consumer.run({
    eachMessage: async ({ topic, message }) => {
      const data = JSON.parse(message.value?.toString() || '{}');

      switch (topic) {
        case 'order.inventory.reserved':
          await prisma.order.update({
            where: { id: data.orderId },
            data: { status: 'confirmed' },
          });
          // Trigger payment
          await producer.send({
            topic: 'order.payment.initiate',
            messages: [{
              key: data.orderId,
              value: JSON.stringify(data),
            }],
          });
          break;

        case 'order.inventory.reserve_failed':
          // Compensate: cancel order
          await prisma.order.update({
            where: { id: data.orderId },
            data: { status: 'cancelled', cancelReason: 'Inventory unavailable' },
          });
          break;

        case 'payment.completed':
          await prisma.order.update({
            where: { id: data.orderId },
            data: { status: 'processing', paidAt: new Date() },
          });
          break;

        case 'payment.failed':
          // Compensate: release inventory + cancel order
          await producer.send({
            topic: 'order.inventory.release',
            messages: [{
              key: data.orderId,
              value: message.value!,
            }],
          });
          await prisma.order.update({
            where: { id: data.orderId },
            data: { status: 'cancelled', cancelReason: 'Payment failed' },
          });
          break;
      }
    },
  });
}

function calculateDeliveryDate(province: string): string {
  const bangkokProvinces = ['กรุงเทพมหานคร', 'นนทบุรี', 'ปทุมธานี', 'สมุทรปราการ'];
  const days = bangkokProvinces.includes(province) ? 1 : 3;
  const date = new Date();
  date.setDate(date.getDate() + days);
  return date.toISOString().split('T')[0];
}

app.get('/health', (_req, res) => {
  res.json({ status: 'healthy', service: 'order-service' });
});

const PORT = process.env.PORT || 3003;
app.listen(PORT, async () => {
  await setupSagaConsumer();
  console.log(`Order service on port ${PORT}`);
});
```

---

## 4. Complete TypeScript PaymentService

```typescript
// services/payment/src/index.ts
import express from 'express';
import { PrismaClient } from '@prisma/client';
import Stripe from 'stripe';
import { Kafka, Producer, Partitioners } from 'kafkajs';
import { createClient } from 'redis';
import { createHash } from 'crypto';
import { z } from 'zod';

const app = express();
app.use(express.json());

const prisma = new PrismaClient();
const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, { apiVersion: '2023-10-16' });
const redis = createClient({ url: process.env.REDIS_URL });
redis.connect();

const kafka = new Kafka({ clientId: 'payment-service', brokers: (process.env.KAFKA_BROKERS || 'kafka:9092').split(',') });
const producer: Producer = kafka.producer({ createPartitioner: Partitioners.LegacyPartitioner, idempotent: true });
producer.connect();

const ProcessPaymentSchema = z.object({
  orderId: z.string().uuid(),
  amount: z.number().positive(),
  currency: z.string().default('thb'),
  paymentMethod: z.enum(['credit_card', 'promptpay', 'wallet']),
  stripePaymentMethodId: z.string().optional(), // For card payments
});

// Idempotency key middleware
const idempotencyMiddleware = async (
  req: express.Request,
  res: express.Response,
  next: express.NextFunction
) => {
  const idempotencyKey = req.headers['idempotency-key'] as string;
  if (!idempotencyKey) return next();

  const cacheKey = `idempotency:${idempotencyKey}`;
  const cached = await redis.get(cacheKey);
  if (cached) {
    return res.json(JSON.parse(cached));
  }

  const originalJson = res.json.bind(res);
  res.json = (body: any) => {
    if (res.statusCode < 400) {
      redis.setEx(cacheKey, 86400, JSON.stringify(body));
    }
    return originalJson(body);
  };

  next();
};

app.post('/api/v1/payments/process', idempotencyMiddleware, async (req, res) => {
  try {
    const body = ProcessPaymentSchema.parse(req.body);
    const { orderId, amount, currency, paymentMethod } = body;

    // Check for duplicate payment
    const existing = await prisma.payment.findUnique({ where: { orderId } });
    if (existing && existing.status === 'succeeded') {
      return res.json({ payment: existing, message: 'Payment already processed' });
    }

    // Create payment intent
    let paymentIntentId: string | null = null;
    let clientSecret: string | null = null;
    let status: 'pending' | 'succeeded' | 'failed' = 'pending';

    if (paymentMethod === 'credit_card' && body.stripePaymentMethodId) {
      const intent = await stripe.paymentIntents.create({
        amount: Math.round(amount * 100), // Stripe uses smallest currency unit
        currency,
        payment_method: body.stripePaymentMethodId,
        confirm: true,
        metadata: { orderId, service: 'ecommerce-platform' },
        return_url: `${process.env.FRONTEND_URL}/orders/${orderId}/complete`,
        automatic_payment_methods: { enabled: false },
      });

      paymentIntentId = intent.id;
      clientSecret = intent.client_secret;
      status = intent.status === 'succeeded' ? 'succeeded' : 'pending';
    } else if (paymentMethod === 'promptpay') {
      const intent = await stripe.paymentIntents.create({
        amount: Math.round(amount * 100),
        currency: 'thb',
        payment_method_types: ['promptpay'],
        confirm: true,
        metadata: { orderId },
      });

      paymentIntentId = intent.id;
      clientSecret = intent.client_secret;
    }

    // Save payment record
    const payment = await prisma.payment.upsert({
      where: { orderId },
      create: {
        orderId,
        amount,
        currency,
        method: paymentMethod,
        status,
        stripePaymentIntentId: paymentIntentId,
        reference: `PAY-${Date.now()}-${orderId.substring(0, 8).toUpperCase()}`,
      },
      update: {
        stripePaymentIntentId: paymentIntentId,
        status,
      },
    });

    // Publish event
    if (status === 'succeeded') {
      await producer.send({
        topic: 'payment.completed',
        messages: [{
          key: orderId,
          value: JSON.stringify({
            orderId,
            paymentId: payment.id,
            amount,
            currency,
            method: paymentMethod,
            timestamp: new Date().toISOString(),
          }),
          headers: { event_type: 'PAYMENT_COMPLETED' },
        }],
      });
    }

    res.json({ payment, clientSecret });
  } catch (error) {
    if (error instanceof z.ZodError) {
      return res.status(400).json({ error: 'Validation failed', details: error.errors });
    }

    if (error instanceof Stripe.errors.StripeCardError) {
      // Payment failed - publish event
      await producer.send({
        topic: 'payment.failed',
        messages: [{
          key: req.body.orderId,
          value: JSON.stringify({
            orderId: req.body.orderId,
            error: error.message,
            code: error.code,
          }),
        }],
      });
      return res.status(402).json({ error: error.message, code: error.code });
    }

    console.error('Payment error:', error);
    res.status(500).json({ error: 'Payment processing failed' });
  }
});

// Stripe webhook handler
app.post('/webhooks/stripe', express.raw({ type: 'application/json' }), async (req, res) => {
  const sig = req.headers['stripe-signature'] as string;

  let event: Stripe.Event;
  try {
    event = stripe.webhooks.constructEvent(req.body, sig, process.env.STRIPE_WEBHOOK_SECRET!);
  } catch (err) {
    console.error('Stripe webhook signature verification failed:', err);
    return res.status(400).json({ error: 'Invalid signature' });
  }

  switch (event.type) {
    case 'payment_intent.succeeded': {
      const intent = event.data.object as Stripe.PaymentIntent;
      const orderId = intent.metadata.orderId;

      await prisma.payment.update({
        where: { stripePaymentIntentId: intent.id },
        data: { status: 'succeeded', paidAt: new Date() },
      });

      await producer.send({
        topic: 'payment.completed',
        messages: [{
          key: orderId,
          value: JSON.stringify({
            orderId,
            stripeIntentId: intent.id,
            amount: intent.amount / 100,
            timestamp: new Date().toISOString(),
          }),
        }],
      });
      break;
    }

    case 'payment_intent.payment_failed': {
      const intent = event.data.object as Stripe.PaymentIntent;
      const orderId = intent.metadata.orderId;

      await prisma.payment.update({
        where: { stripePaymentIntentId: intent.id },
        data: { status: 'failed' },
      });

      await producer.send({
        topic: 'payment.failed',
        messages: [{
          key: orderId,
          value: JSON.stringify({
            orderId,
            error: intent.last_payment_error?.message,
          }),
        }],
      });
      break;
    }

    case 'charge.refunded': {
      const charge = event.data.object as Stripe.Charge;
      await prisma.payment.update({
        where: { stripePaymentIntentId: charge.payment_intent as string },
        data: { status: 'refunded', refundedAt: new Date() },
      });
      break;
    }
  }

  res.json({ received: true });
});

app.get('/health', (_req, res) => res.json({ status: 'healthy', service: 'payment-service' }));

const PORT = process.env.PORT || 3004;
app.listen(PORT, () => console.log(`Payment service on port ${PORT}`));
```

---

## 5. Docker Compose: All Services

```yaml
# docker-compose.yml
version: '3.9'

services:
  kong:
    image: kong:3.4
    environment:
      KONG_DATABASE: 'off'
      KONG_DECLARATIVE_CONFIG: /kong/declarative/kong.yml
      KONG_PROXY_ACCESS_LOG: /dev/stdout
      KONG_ADMIN_ACCESS_LOG: /dev/stdout
      KONG_PROXY_ERROR_LOG: /dev/stderr
      KONG_ADMIN_ERROR_LOG: /dev/stderr
      KONG_ADMIN_LISTEN: '0.0.0.0:8001'
    volumes:
      - ./kong/kong.yml:/kong/declarative/kong.yml
    ports:
      - '8000:8000'
      - '8001:8001'
    depends_on:
      - user-service
      - product-service
      - order-service
      - payment-service

  user-service:
    build: ./services/user
    environment:
      PORT: '3001'
      DATABASE_URL: postgresql://postgres:password@postgres:5432/users
      REDIS_URL: redis://redis:6379
      JWT_SECRET: dev-jwt-secret-change-in-prod
      JWT_REFRESH_SECRET: dev-refresh-secret-change-in-prod
      INTERNAL_API_KEY: internal-dev-key
    ports:
      - '3001:3001'
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy

  product-service:
    build: ./services/product
    environment:
      PORT: '3002'
      DATABASE_URL: postgresql://postgres:password@postgres:5432/products
      REDIS_URL: redis://redis:6379
      ELASTICSEARCH_URL: http://elasticsearch:9200
      INTERNAL_API_KEY: internal-dev-key
    ports:
      - '3002:3002'

  order-service:
    build: ./services/order
    environment:
      PORT: '3003'
      DATABASE_URL: postgresql://postgres:password@postgres:5432/orders
      REDIS_URL: redis://redis:6379
      KAFKA_BROKERS: kafka:9092
      USER_SERVICE_URL: http://user-service:3001
      PRODUCT_SERVICE_URL: http://product-service:3002
      INTERNAL_API_KEY: internal-dev-key
    ports:
      - '3003:3003'
    depends_on:
      kafka:
        condition: service_healthy

  payment-service:
    build: ./services/payment
    environment:
      PORT: '3004'
      DATABASE_URL: postgresql://postgres:password@postgres:5432/payments
      REDIS_URL: redis://redis:6379
      KAFKA_BROKERS: kafka:9092
      STRIPE_SECRET_KEY: sk_test_your_stripe_key
      STRIPE_WEBHOOK_SECRET: whsec_your_webhook_secret
      FRONTEND_URL: http://localhost:3000
    ports:
      - '3004:3004'

  notification-service:
    build: ./services/notification
    environment:
      PORT: '3005'
      DATABASE_URL: postgresql://postgres:password@postgres:5432/notifications
      KAFKA_BROKERS: kafka:9092
      SMTP_HOST: mailhog
      SMTP_PORT: '1025'
      FCM_SERVER_KEY: your-fcm-key
    ports:
      - '3005:3005'

  search-service:
    build: ./services/search
    environment:
      PORT: '3006'
      ELASTICSEARCH_URL: http://elasticsearch:9200
      DATABASE_URL: postgresql://postgres:password@postgres:5432/products
      KAFKA_BROKERS: kafka:9092
    ports:
      - '3006:3006'

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: password
      POSTGRES_USER: postgres
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./db/init-databases.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - '5432:5432'
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U postgres']
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
    ports:
      - '6379:6379'
    healthcheck:
      test: ['CMD', 'redis-cli', 'ping']
      interval: 5s
      retries: 5

  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    depends_on:
      - zookeeper
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092,PLAINTEXT_HOST://localhost:29092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: 'true'
    ports:
      - '29092:29092'
    healthcheck:
      test: ['CMD', 'kafka-topics', '--bootstrap-server', 'kafka:9092', '--list']
      interval: 10s
      retries: 5

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.10.4
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - ES_JAVA_OPTS=-Xms512m -Xmx512m
    volumes:
      - elasticsearch_data:/usr/share/elasticsearch/data
    ports:
      - '9200:9200'
    healthcheck:
      test: ['CMD-SHELL', 'curl -s http://localhost:9200/_cluster/health | grep -q "green\\|yellow"']
      interval: 30s
      retries: 5

  prometheus:
    image: prom/prometheus:v2.47.2
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=15d'
    ports:
      - '9090:9090'

  grafana:
    image: grafana/grafana:10.2.0
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin123
      GF_USERS_ALLOW_SIGN_UP: 'false'
    volumes:
      - grafana_data:/var/lib/grafana
      - ./monitoring/grafana:/etc/grafana/provisioning
    ports:
      - '3000:3000'

  mailhog:
    image: mailhog/mailhog
    ports:
      - '1025:1025'
      - '8025:8025'

volumes:
  postgres_data:
  redis_data:
  elasticsearch_data:
  prometheus_data:
  grafana_data:
```

---

## 6. Kubernetes Manifests

```yaml
# k8s/base/user-service/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  namespace: ecommerce
  labels:
    app: user-service
    version: v1
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
        version: v1
      annotations:
        prometheus.io/scrape: 'true'
        prometheus.io/port: '3001'
        prometheus.io/path: '/metrics'
    spec:
      serviceAccountName: user-service
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
      containers:
        - name: user-service
          image: 123456789.dkr.ecr.ap-southeast-1.amazonaws.com/user-service:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 3001
              name: http
          env:
            - name: PORT
              value: '3001'
            - name: NODE_ENV
              value: production
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: user-service-secrets
                  key: database-url
            - name: REDIS_URL
              valueFrom:
                secretKeyRef:
                  name: shared-secrets
                  key: redis-url
            - name: JWT_SECRET
              valueFrom:
                secretKeyRef:
                  name: user-service-secrets
                  key: jwt-secret
            - name: JWT_REFRESH_SECRET
              valueFrom:
                secretKeyRef:
                  name: user-service-secrets
                  key: jwt-refresh-secret
            - name: INTERNAL_API_KEY
              valueFrom:
                secretKeyRef:
                  name: shared-secrets
                  key: internal-api-key
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
          livenessProbe:
            httpGet:
              path: /health
              port: 3001
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
            timeoutSeconds: 5
          readinessProbe:
            httpGet:
              path: /health
              port: 3001
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 3
          lifecycle:
            preStop:
              exec:
                command: ['/bin/sh', '-c', 'sleep 5']
          securityContext:
            runAsNonRoot: true
            runAsUser: 1001
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: user-service

---
# k8s/base/user-service/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: user-service
  namespace: ecommerce
  labels:
    app: user-service
spec:
  selector:
    app: user-service
  ports:
    - name: http
      port: 3001
      targetPort: 3001
      protocol: TCP
  type: ClusterIP

---
# k8s/base/user-service/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: user-service
  namespace: ecommerce
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: user-service
  minReplicas: 2
  maxReplicas: 20
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
    - type: Pods
      pods:
        metric:
          name: http_requests_total
        target:
          type: AverageValue
          averageValue: 100
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Pods
          value: 2
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 1
          periodSeconds: 120

---
# k8s/base/user-service/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: user-service-config
  namespace: ecommerce
data:
  NODE_ENV: production
  PORT: '3001'
  LOG_LEVEL: info
  RATE_LIMIT_WINDOW_MS: '900000'
  RATE_LIMIT_MAX_REQUESTS: '100'

---
# k8s/base/user-service/secret.yaml (values managed by external-secrets)
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: user-service-secrets
  namespace: ecommerce
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: user-service-secrets
    creationPolicy: Owner
  data:
    - secretKey: database-url
      remoteRef:
        key: ecommerce/production/user-service
        property: database_url
    - secretKey: jwt-secret
      remoteRef:
        key: ecommerce/production/user-service
        property: jwt_secret
    - secretKey: jwt-refresh-secret
      remoteRef:
        key: ecommerce/production/user-service
        property: jwt_refresh_secret
```

---

## 7. GitHub Actions CI/CD Pipeline

```yaml
# .github/workflows/deploy.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  AWS_REGION: ap-southeast-1
  ECR_REGISTRY: 123456789.dkr.ecr.ap-southeast-1.amazonaws.com
  EKS_CLUSTER: ecommerce-production

jobs:
  test:
    name: Test ${{ matrix.service }}
    runs-on: ubuntu-latest
    strategy:
      matrix:
        service: [user, product, order, payment, notification, search]
      fail-fast: false

    steps:
      - uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: npm
          cache-dependency-path: services/${{ matrix.service }}/package-lock.json

      - name: Install dependencies
        run: npm ci
        working-directory: services/${{ matrix.service }}

      - name: Run linting
        run: npm run lint
        working-directory: services/${{ matrix.service }}

      - name: Run type check
        run: npm run type-check
        working-directory: services/${{ matrix.service }}

      - name: Run unit tests
        run: npm run test:unit -- --coverage
        working-directory: services/${{ matrix.service }}

      - name: Run integration tests
        run: npm run test:integration
        working-directory: services/${{ matrix.service }}
        env:
          DATABASE_URL: postgresql://postgres:password@localhost:5432/test
          REDIS_URL: redis://localhost:6379

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          directory: services/${{ matrix.service }}/coverage
          flags: ${{ matrix.service }}

    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_PASSWORD: password
          POSTGRES_USER: postgres
          POSTGRES_DB: test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      redis:
        image: redis:7-alpine
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

  security-scan:
    name: Security Scan
    runs-on: ubuntu-latest
    needs: test
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'

    steps:
      - uses: actions/checkout@v4

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: fs
          scan-ref: .
          format: sarif
          output: trivy-results.sarif
          severity: CRITICAL,HIGH

      - name: Upload Trivy results
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: trivy-results.sarif

  build-and-push:
    name: Build & Push ${{ matrix.service }}
    runs-on: ubuntu-latest
    needs: [test, security-scan]
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    strategy:
      matrix:
        service: [user, product, order, payment, notification, search]

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.ECR_REGISTRY }}/${{ matrix.service }}-service
          tags: |
            type=sha,prefix=sha-
            type=raw,value=latest,enable={{is_default_branch}}

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: services/${{ matrix.service }}
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            BUILD_DATE=${{ github.event.head_commit.timestamp }}
            GIT_COMMIT=${{ github.sha }}

      - name: Scan image for vulnerabilities
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.ECR_REGISTRY }}/${{ matrix.service }}-service:${{ github.sha }}
          format: table
          exit-code: '1'
          severity: CRITICAL

  deploy:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: build-and-push
    environment:
      name: production
      url: https://api.myecommerce.com

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Update kubeconfig
        run: aws eks update-kubeconfig --name ${{ env.EKS_CLUSTER }} --region ${{ env.AWS_REGION }}

      - name: Update image tags in kustomization
        run: |
          cd k8s/overlays/production
          for service in user product order payment notification search; do
            kustomize edit set image \
              ${service}-service=${{ env.ECR_REGISTRY }}/${service}-service:sha-${{ github.sha }}
          done

      - name: Apply Kubernetes manifests
        run: |
          kubectl apply -k k8s/overlays/production
          kubectl rollout status deployment/user-service -n ecommerce --timeout=5m
          kubectl rollout status deployment/order-service -n ecommerce --timeout=5m
          kubectl rollout status deployment/payment-service -n ecommerce --timeout=5m

      - name: Verify deployment
        run: |
          for service in user-service product-service order-service payment-service notification-service search-service; do
            kubectl get deployment $service -n ecommerce -o jsonpath='{.status.readyReplicas}'
            echo " replicas ready for $service"
          done

      - name: Run smoke tests
        run: |
          curl -f https://api.myecommerce.com/health || exit 1
          curl -f https://api.myecommerce.com/api/v1/products?limit=1 || exit 1

      - name: Notify Slack on success
        if: success()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "✅ Deployment successful! SHA: ${{ github.sha }}",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "✅ *Production deployment successful!*\n*Commit:* ${{ github.sha }}\n*Author:* ${{ github.actor }}"
                  }
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

      - name: Notify Slack on failure
        if: failure()
        uses: slackapi/slack-github-action@v1
        with:
          payload: '{"text": "❌ Deployment FAILED! SHA: ${{ github.sha }}"}'
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

## 8. Prometheus + Grafana Setup

```yaml
# monitoring/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: production
    environment: production

rule_files:
  - /etc/prometheus/rules/*.yml

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']
      timeout: 10s

scrape_configs:
  - job_name: 'user-service'
    static_configs:
      - targets: ['user-service:3001']
    metrics_path: /metrics

  - job_name: 'order-service'
    static_configs:
      - targets: ['order-service:3003']

  - job_name: 'payment-service'
    static_configs:
      - targets: ['payment-service:3004']

  - job_name: 'search-service'
    static_configs:
      - targets: ['search-service:3006']

  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: 'true'
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: kubernetes_pod_name

---
# monitoring/alertmanager.yml
global:
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'alerts@myecommerce.com'
  slack_api_url: 'https://hooks.slack.com/services/YOUR/WEBHOOK/URL'

route:
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: 'slack-general'
  routes:
    - match:
        severity: critical
      receiver: 'pagerduty-critical'
      continue: true
    - match:
        severity: critical
      receiver: 'slack-critical'

receivers:
  - name: 'slack-general'
    slack_configs:
      - channel: '#platform-alerts'
        title: '{{ .CommonAnnotations.summary }}'
        text: '{{ .CommonAnnotations.description }}'
        send_resolved: true

  - name: 'slack-critical'
    slack_configs:
      - channel: '#platform-critical'
        title: '🚨 CRITICAL: {{ .CommonAnnotations.summary }}'
        text: '{{ .CommonAnnotations.description }}'
        color: '#FF0000'
        send_resolved: true

  - name: 'pagerduty-critical'
    pagerduty_configs:
      - routing_key: 'YOUR_PAGERDUTY_KEY'
        description: '{{ .CommonAnnotations.summary }}'
```

---

## 9. Step-by-Step Local Development Guide

```bash
#!/bin/bash
# scripts/setup-local.sh

echo "=== Setting up local development environment ==="

# 1. Check prerequisites
check_prerequisites() {
  echo "Checking prerequisites..."
  commands=("docker" "docker-compose" "node" "npm" "kubectl" "helm")
  for cmd in "${commands[@]}"; do
    if ! command -v "$cmd" &> /dev/null; then
      echo "❌ $cmd not found. Please install it."
      exit 1
    fi
    echo "✅ $cmd found"
  done
}

# 2. Start infrastructure
start_infrastructure() {
  echo "Starting infrastructure..."
  docker-compose up -d postgres redis kafka zookeeper elasticsearch
  
  echo "Waiting for postgres..."
  until docker-compose exec postgres pg_isready -U postgres; do sleep 2; done
  
  echo "Waiting for elasticsearch..."
  until curl -s http://localhost:9200/_cluster/health | grep -q '"status":"green"\|"status":"yellow"'; do
    sleep 5
  done
  
  echo "✅ Infrastructure ready"
}

# 3. Run database migrations
run_migrations() {
  echo "Running database migrations..."
  
  # Create databases
  docker-compose exec postgres psql -U postgres -c "
    CREATE DATABASE IF NOT EXISTS users;
    CREATE DATABASE IF NOT EXISTS products;
    CREATE DATABASE IF NOT EXISTS orders;
    CREATE DATABASE IF NOT EXISTS payments;
    CREATE DATABASE IF NOT EXISTS notifications;
  " 2>/dev/null || true
  
  # Run Prisma migrations
  for service in user product order payment notification; do
    echo "Migrating $service service..."
    cd services/$service
    DATABASE_URL="postgresql://postgres:password@localhost:5432/${service}s" \
      npx prisma migrate dev --name init 2>/dev/null || true
    cd ../..
  done
  
  echo "✅ Migrations complete"
}

# 4. Install dependencies
install_dependencies() {
  echo "Installing dependencies..."
  for service in user product order payment notification search; do
    echo "Installing $service service dependencies..."
    cd services/$service && npm install && cd ../..
  done
  echo "✅ Dependencies installed"
}

# 5. Start services in dev mode
start_services() {
  echo "Starting services..."
  docker-compose up -d user-service product-service order-service payment-service notification-service search-service
  echo "✅ All services started"
  echo ""
  echo "Services:"
  echo "  API Gateway:    http://localhost:8000"
  echo "  User Service:   http://localhost:3001"
  echo "  Product Service: http://localhost:3002"
  echo "  Order Service:  http://localhost:3003"
  echo "  Payment Service: http://localhost:3004"
  echo "  Prometheus:     http://localhost:9090"
  echo "  Grafana:        http://localhost:3000 (admin/admin123)"
  echo "  MailHog:        http://localhost:8025"
}

check_prerequisites
start_infrastructure
install_dependencies
run_migrations
start_services
echo "=== Local development environment ready! ==="
```

---

## 10. Production Deployment Guide

```bash
#!/bin/bash
# scripts/deploy-production.sh

echo "=== Production Deployment Guide ==="

# Step 1: Pre-deployment checks
pre_deployment_checks() {
  echo "Running pre-deployment checks..."
  
  # Check cluster access
  kubectl cluster-info || { echo "❌ Cannot connect to cluster"; exit 1; }
  
  # Check namespaces
  kubectl get namespace ecommerce || kubectl create namespace ecommerce
  kubectl get namespace monitoring || kubectl create namespace monitoring
  
  # Check secrets exist
  kubectl get secret user-service-secrets -n ecommerce || {
    echo "❌ Secrets not configured. Run: ./scripts/setup-secrets.sh"
    exit 1
  }
  
  echo "✅ Pre-deployment checks passed"
}

# Step 2: Deploy infrastructure
deploy_infrastructure() {
  echo "Deploying infrastructure..."
  
  # Deploy PostgreSQL (using Helm)
  helm upgrade --install postgres bitnami/postgresql \
    --namespace ecommerce \
    --values helm/postgres-values.yaml \
    --wait
  
  # Deploy Redis Cluster
  helm upgrade --install redis bitnami/redis \
    --namespace ecommerce \
    --values helm/redis-values.yaml \
    --wait
  
  # Deploy Kafka
  helm upgrade --install kafka bitnami/kafka \
    --namespace ecommerce \
    --values helm/kafka-values.yaml \
    --wait
  
  # Deploy Elasticsearch
  helm upgrade --install elasticsearch elastic/elasticsearch \
    --namespace ecommerce \
    --values helm/elasticsearch-values.yaml \
    --wait
  
  echo "✅ Infrastructure deployed"
}

# Step 3: Deploy services
deploy_services() {
  echo "Deploying microservices..."
  kubectl apply -k k8s/overlays/production
  
  # Wait for rollouts
  for service in user product order payment notification search; do
    echo "Waiting for ${service}-service..."
    kubectl rollout status deployment/${service}-service -n ecommerce --timeout=5m
  done
  
  echo "✅ Services deployed"
}

# Step 4: Verify deployment
verify_deployment() {
  echo "Verifying deployment..."
  
  # Check pods are running
  kubectl get pods -n ecommerce
  
  # Run health checks
  GATEWAY_URL=$(kubectl get svc kong -n ecommerce -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
  
  for endpoint in "/health" "/api/v1/products?limit=1"; do
    response=$(curl -s -o /dev/null -w "%{http_code}" "http://${GATEWAY_URL}${endpoint}")
    if [ "$response" != "200" ]; then
      echo "❌ Health check failed for ${endpoint}: HTTP ${response}"
      exit 1
    fi
    echo "✅ ${endpoint} - OK"
  done
  
  echo "✅ Deployment verified"
}

pre_deployment_checks
deploy_infrastructure
deploy_services
verify_deployment

echo ""
echo "=== 🎉 Production deployment complete! ==="
echo "API Gateway: https://api.myecommerce.com"
echo "Grafana: https://grafana.myecommerce.com"
echo "Prometheus: https://prometheus.myecommerce.com"
```

---

## สรุป

| Component | Technology | Purpose |
|-----------|-----------|---------|
| UserService | Express + Prisma + Redis + JWT | Authentication & User Management |
| OrderService | Express + Prisma + Kafka Saga | Order Processing with Distributed Transaction |
| PaymentService | Express + Stripe + Kafka | Secure Payment Processing |
| ProductService | Express + Prisma + Elasticsearch | Product Catalog & Inventory |
| NotificationService | Kafka Consumer + NodeMailer + FCM | Multi-channel Notifications |
| SearchService | Express + Elasticsearch | Product Search & Autocomplete |
| API Gateway | Kong | Rate Limiting, Auth, Routing |
| Database | PostgreSQL (Prisma) | Persistent Data Storage |
| Cache | Redis | Sessions, Rate Limiting, Cache |
| Events | Apache Kafka | Async Communication, Saga |
| Search | Elasticsearch | Full-text Search |
| Monitoring | Prometheus + Grafana | Metrics & Alerting |
| CI/CD | GitHub Actions | Automated Testing & Deployment |
| Container | Docker + Kubernetes | Containerization & Orchestration |

> "A complete microservices system is not just code — it's testing, monitoring, deployment, security, and team practices working together"

---

*ถัดไป: Part 100 - Course Summary and Next Steps*

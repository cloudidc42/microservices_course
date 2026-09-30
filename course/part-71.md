# Part 71: Testing Strategy for Microservices

## บทนำ

การทดสอบ Microservices มีความซับซ้อนกว่า Monolith เพราะต้องครอบคลุมทั้ง unit, integration, contract, และ end-to-end ระหว่าง services ต่างๆ บทนี้จะครอบคลุม Testing Pyramid สมบูรณ์พร้อม tools ทันสมัย

---

## 1. Testing Pyramid for Microservices

```
        /\
       /E2E\          ← น้อย: Playwright, Cypress
      /------\
     /Contract\       ← ปานกลาง: Pact
    /----------\
   /Integration \     ← ปานกลาง: Testcontainers
  /--------------\
 /   Unit Tests   \   ← มาก: Jest + Mocking
/------------------\
```

### โครงสร้าง Testing

```typescript
// test-strategy-overview.ts
// แสดง testing layers ของ microservices

interface TestingLayer {
  name: string;
  scope: string;
  speed: 'fast' | 'medium' | 'slow';
  tools: string[];
  coverage: string;
}

const testingLayers: TestingLayer[] = [
  {
    name: 'Unit Tests',
    scope: 'Single function/class',
    speed: 'fast',
    tools: ['Jest', 'Vitest', 'Sinon'],
    coverage: '70-80% of codebase',
  },
  {
    name: 'Integration Tests',
    scope: 'Service + dependencies',
    speed: 'medium',
    tools: ['Testcontainers', 'Supertest'],
    coverage: '15-20% key flows',
  },
  {
    name: 'Contract Tests',
    scope: 'Service interfaces',
    speed: 'medium',
    tools: ['Pact', 'Spring Contract'],
    coverage: 'All service boundaries',
  },
  {
    name: 'Component Tests',
    scope: 'Service in isolation',
    speed: 'medium',
    tools: ['Supertest', 'MSW'],
    coverage: 'Business logic flows',
  },
  {
    name: 'E2E Tests',
    scope: 'Full system',
    speed: 'slow',
    tools: ['Playwright', 'Cypress'],
    coverage: 'Critical user journeys',
  },
];
```

---

## 2. Unit Tests with Jest and Mocking

### Setup Jest สำหรับ TypeScript Microservice

```json
// jest.config.ts (jest configuration)
{
  "jest": {
    "preset": "ts-jest",
    "testEnvironment": "node",
    "roots": ["<rootDir>/src", "<rootDir>/tests"],
    "testMatch": ["**/*.test.ts", "**/*.spec.ts"],
    "collectCoverageFrom": [
      "src/**/*.ts",
      "!src/**/*.d.ts",
      "!src/index.ts"
    ],
    "coverageThreshold": {
      "global": {
        "branches": 80,
        "functions": 85,
        "lines": 85,
        "statements": 85
      }
    },
    "moduleNameMapper": {
      "@/(.*)": "<rootDir>/src/$1"
    },
    "setupFilesAfterFramework": ["<rootDir>/tests/setup.ts"]
  }
}
```

```typescript
// src/services/order.service.ts
import { Pool } from 'pg';
import { Redis } from 'ioredis';
import { EventEmitter } from 'events';

export interface Order {
  id: string;
  userId: string;
  items: OrderItem[];
  totalAmount: number;
  status: OrderStatus;
  createdAt: Date;
  updatedAt: Date;
}

export interface OrderItem {
  productId: string;
  quantity: number;
  price: number;
}

export type OrderStatus = 'PENDING' | 'CONFIRMED' | 'SHIPPED' | 'DELIVERED' | 'CANCELLED';

export class OrderService {
  constructor(
    private readonly db: Pool,
    private readonly redis: Redis,
    private readonly eventEmitter: EventEmitter,
    private readonly inventoryClient: InventoryClient,
    private readonly paymentClient: PaymentClient
  ) {}

  async createOrder(userId: string, items: OrderItem[]): Promise<Order> {
    // Validate items
    if (!items || items.length === 0) {
      throw new Error('Order must have at least one item');
    }

    // Check inventory
    for (const item of items) {
      const available = await this.inventoryClient.checkAvailability(
        item.productId,
        item.quantity
      );
      if (!available) {
        throw new Error(`Product ${item.productId} is out of stock`);
      }
    }

    const totalAmount = items.reduce((sum, item) => sum + item.price * item.quantity, 0);

    // Create order in DB
    const result = await this.db.query<Order>(
      `INSERT INTO orders (user_id, items, total_amount, status, created_at, updated_at)
       VALUES ($1, $2, $3, 'PENDING', NOW(), NOW())
       RETURNING *`,
      [userId, JSON.stringify(items), totalAmount]
    );

    const order = result.rows[0];

    // Cache order
    await this.redis.setex(
      `order:${order.id}`,
      3600,
      JSON.stringify(order)
    );

    // Emit event
    this.eventEmitter.emit('order.created', { orderId: order.id, userId, totalAmount });

    return order;
  }

  async getOrder(orderId: string): Promise<Order | null> {
    // Check cache first
    const cached = await this.redis.get(`order:${orderId}`);
    if (cached) {
      return JSON.parse(cached) as Order;
    }

    const result = await this.db.query<Order>(
      'SELECT * FROM orders WHERE id = $1',
      [orderId]
    );

    return result.rows[0] || null;
  }

  async cancelOrder(orderId: string, userId: string): Promise<Order> {
    const order = await this.getOrder(orderId);
    
    if (!order) {
      throw new Error('Order not found');
    }

    if (order.userId !== userId) {
      throw new Error('Unauthorized to cancel this order');
    }

    if (!['PENDING', 'CONFIRMED'].includes(order.status)) {
      throw new Error(`Cannot cancel order in ${order.status} status`);
    }

    const result = await this.db.query<Order>(
      `UPDATE orders SET status='CANCELLED', updated_at=NOW() WHERE id=$1 RETURNING *`,
      [orderId]
    );

    const updatedOrder = result.rows[0];

    // Invalidate cache
    await this.redis.del(`order:${orderId}`);

    // Emit event
    this.eventEmitter.emit('order.cancelled', { orderId, userId });

    return updatedOrder;
  }
}

export interface InventoryClient {
  checkAvailability(productId: string, quantity: number): Promise<boolean>;
  reserveItems(orderId: string, items: OrderItem[]): Promise<void>;
  releaseItems(orderId: string): Promise<void>;
}

export interface PaymentClient {
  processPayment(orderId: string, amount: number, userId: string): Promise<string>;
  refund(paymentId: string): Promise<void>;
}
```

```typescript
// tests/unit/order.service.test.ts
import { describe, test, expect, jest, beforeEach } from '@jest/globals';
import { OrderService, InventoryClient, PaymentClient, Order } from '../../src/services/order.service';
import { EventEmitter } from 'events';

// Mock dependencies
const mockDb = {
  query: jest.fn(),
} as any;

const mockRedis = {
  get: jest.fn(),
  set: jest.fn(),
  setex: jest.fn(),
  del: jest.fn(),
} as any;

const mockInventoryClient: InventoryClient = {
  checkAvailability: jest.fn(),
  reserveItems: jest.fn(),
  releaseItems: jest.fn(),
};

const mockPaymentClient: PaymentClient = {
  processPayment: jest.fn(),
  refund: jest.fn(),
};

describe('OrderService', () => {
  let orderService: OrderService;
  let eventEmitter: EventEmitter;

  beforeEach(() => {
    jest.clearAllMocks();
    eventEmitter = new EventEmitter();
    orderService = new OrderService(
      mockDb,
      mockRedis,
      eventEmitter,
      mockInventoryClient,
      mockPaymentClient
    );
  });

  describe('createOrder', () => {
    const validItems = [
      { productId: 'prod-1', quantity: 2, price: 100 },
      { productId: 'prod-2', quantity: 1, price: 50 },
    ];

    test('should create order successfully', async () => {
      // Arrange
      (mockInventoryClient.checkAvailability as jest.Mock)
        .mockResolvedValue(true);

      const expectedOrder: Partial<Order> = {
        id: 'order-123',
        userId: 'user-1',
        items: validItems,
        totalAmount: 250,
        status: 'PENDING',
      };

      mockDb.query.mockResolvedValueOnce({ rows: [expectedOrder] });
      mockRedis.setex.mockResolvedValue('OK');

      // Act
      const order = await orderService.createOrder('user-1', validItems);

      // Assert
      expect(order).toEqual(expectedOrder);
      expect(mockInventoryClient.checkAvailability).toHaveBeenCalledTimes(2);
      expect(mockDb.query).toHaveBeenCalledWith(
        expect.stringContaining('INSERT INTO orders'),
        expect.arrayContaining(['user-1', expect.any(String), 250])
      );
      expect(mockRedis.setex).toHaveBeenCalledWith(
        'order:order-123',
        3600,
        expect.any(String)
      );
    });

    test('should throw error when items array is empty', async () => {
      // Act & Assert
      await expect(orderService.createOrder('user-1', [])).rejects.toThrow(
        'Order must have at least one item'
      );
      expect(mockDb.query).not.toHaveBeenCalled();
    });

    test('should throw error when product is out of stock', async () => {
      // Arrange
      (mockInventoryClient.checkAvailability as jest.Mock)
        .mockResolvedValueOnce(true)
        .mockResolvedValueOnce(false);

      // Act & Assert
      await expect(
        orderService.createOrder('user-1', validItems)
      ).rejects.toThrow('Product prod-2 is out of stock');
    });

    test('should emit order.created event on success', async () => {
      // Arrange
      (mockInventoryClient.checkAvailability as jest.Mock).mockResolvedValue(true);
      mockDb.query.mockResolvedValueOnce({
        rows: [{ id: 'order-123', userId: 'user-1', totalAmount: 250 }]
      });
      mockRedis.setex.mockResolvedValue('OK');

      const eventSpy = jest.fn();
      eventEmitter.on('order.created', eventSpy);

      // Act
      await orderService.createOrder('user-1', validItems);

      // Assert
      expect(eventSpy).toHaveBeenCalledWith({
        orderId: 'order-123',
        userId: 'user-1',
        totalAmount: 250,
      });
    });
  });

  describe('getOrder', () => {
    test('should return cached order when available', async () => {
      // Arrange
      const cachedOrder = { id: 'order-123', status: 'PENDING' };
      mockRedis.get.mockResolvedValueOnce(JSON.stringify(cachedOrder));

      // Act
      const order = await orderService.getOrder('order-123');

      // Assert
      expect(order).toEqual(cachedOrder);
      expect(mockDb.query).not.toHaveBeenCalled();
    });

    test('should fetch from DB when cache miss', async () => {
      // Arrange
      mockRedis.get.mockResolvedValueOnce(null);
      const dbOrder = { id: 'order-123', status: 'CONFIRMED' };
      mockDb.query.mockResolvedValueOnce({ rows: [dbOrder] });

      // Act
      const order = await orderService.getOrder('order-123');

      // Assert
      expect(order).toEqual(dbOrder);
      expect(mockDb.query).toHaveBeenCalledWith(
        expect.stringContaining('SELECT * FROM orders'),
        ['order-123']
      );
    });

    test('should return null when order not found', async () => {
      // Arrange
      mockRedis.get.mockResolvedValueOnce(null);
      mockDb.query.mockResolvedValueOnce({ rows: [] });

      // Act
      const order = await orderService.getOrder('non-existent');

      // Assert
      expect(order).toBeNull();
    });
  });

  describe('cancelOrder', () => {
    test('should cancel order successfully', async () => {
      // Arrange
      const existingOrder = {
        id: 'order-123',
        userId: 'user-1',
        status: 'PENDING',
      };
      mockRedis.get.mockResolvedValueOnce(JSON.stringify(existingOrder));
      
      const cancelledOrder = { ...existingOrder, status: 'CANCELLED' };
      mockDb.query.mockResolvedValueOnce({ rows: [cancelledOrder] });
      mockRedis.del.mockResolvedValue(1);

      // Act
      const result = await orderService.cancelOrder('order-123', 'user-1');

      // Assert
      expect(result.status).toBe('CANCELLED');
      expect(mockRedis.del).toHaveBeenCalledWith('order:order-123');
    });

    test('should throw error for unauthorized cancellation', async () => {
      // Arrange
      const existingOrder = {
        id: 'order-123',
        userId: 'user-1',
        status: 'PENDING',
      };
      mockRedis.get.mockResolvedValueOnce(JSON.stringify(existingOrder));

      // Act & Assert
      await expect(
        orderService.cancelOrder('order-123', 'different-user')
      ).rejects.toThrow('Unauthorized to cancel this order');
    });

    test('should throw error when cancelling shipped order', async () => {
      // Arrange
      const existingOrder = {
        id: 'order-123',
        userId: 'user-1',
        status: 'SHIPPED',
      };
      mockRedis.get.mockResolvedValueOnce(JSON.stringify(existingOrder));

      // Act & Assert
      await expect(
        orderService.cancelOrder('order-123', 'user-1')
      ).rejects.toThrow('Cannot cancel order in SHIPPED status');
    });
  });
});

// Advanced Mocking Patterns
describe('Advanced Mocking', () => {
  test('should handle partial mock with spy', () => {
    const realService = {
      compute: (x: number) => x * 2,
      process: (x: number) => realService.compute(x) + 1,
    };

    const spy = jest.spyOn(realService, 'compute').mockReturnValue(10);
    
    const result = realService.process(5);
    
    expect(result).toBe(11);
    expect(spy).toHaveBeenCalledWith(5);
  });

  test('should mock timers', async () => {
    jest.useFakeTimers();
    
    const callback = jest.fn();
    setTimeout(callback, 5000);
    
    jest.advanceTimersByTime(5000);
    
    expect(callback).toHaveBeenCalledTimes(1);
    
    jest.useRealTimers();
  });

  test('should mock modules', async () => {
    jest.mock('ioredis', () => ({
      default: jest.fn().mockImplementation(() => ({
        get: jest.fn().mockResolvedValue(null),
        set: jest.fn().mockResolvedValue('OK'),
      })),
    }));
  });
});
```

---

## 3. Integration Tests with Testcontainers

```typescript
// tests/integration/order.integration.test.ts
import { describe, test, expect, beforeAll, afterAll, beforeEach } from '@jest/globals';
import { GenericContainer, StartedTestContainer } from 'testcontainers';
import { Pool } from 'pg';
import Redis from 'ioredis';
import { OrderService } from '../../src/services/order.service';
import { EventEmitter } from 'events';

describe('OrderService Integration Tests', () => {
  let postgresContainer: StartedTestContainer;
  let redisContainer: StartedTestContainer;
  let db: Pool;
  let redis: Redis;
  let orderService: OrderService;

  beforeAll(async () => {
    // Start PostgreSQL container
    postgresContainer = await new GenericContainer('postgres:15')
      .withEnvironment({
        POSTGRES_DB: 'testdb',
        POSTGRES_USER: 'testuser',
        POSTGRES_PASSWORD: 'testpass',
      })
      .withExposedPorts(5432)
      .withHealthCheck({
        test: ['CMD-SHELL', 'pg_isready -U testuser -d testdb'],
        interval: 1000,
        timeout: 5000,
        retries: 10,
      })
      .start();

    // Start Redis container
    redisContainer = await new GenericContainer('redis:7')
      .withExposedPorts(6379)
      .withHealthCheck({
        test: ['CMD', 'redis-cli', 'ping'],
        interval: 1000,
        timeout: 5000,
        retries: 10,
      })
      .start();

    // Connect to containers
    db = new Pool({
      host: postgresContainer.getHost(),
      port: postgresContainer.getMappedPort(5432),
      database: 'testdb',
      user: 'testuser',
      password: 'testpass',
    });

    redis = new Redis({
      host: redisContainer.getHost(),
      port: redisContainer.getMappedPort(6379),
    });

    // Run migrations
    await db.query(`
      CREATE TABLE IF NOT EXISTS orders (
        id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
        user_id TEXT NOT NULL,
        items JSONB NOT NULL,
        total_amount DECIMAL NOT NULL,
        status TEXT NOT NULL DEFAULT 'PENDING',
        created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
        updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
      )
    `);

  }, 60000); // Extended timeout for container startup

  afterAll(async () => {
    await db.end();
    await redis.quit();
    await postgresContainer.stop();
    await redisContainer.stop();
  });

  beforeEach(async () => {
    // Clean test data
    await db.query('DELETE FROM orders');
    await redis.flushall();
  });

  test('should create and retrieve order from real database', async () => {
    const mockInventory = {
      checkAvailability: async () => true,
      reserveItems: async () => {},
      releaseItems: async () => {},
    };
    
    const mockPayment = {
      processPayment: async () => 'payment-123',
      refund: async () => {},
    };

    orderService = new OrderService(
      db,
      redis,
      new EventEmitter(),
      mockInventory,
      mockPayment
    );

    const items = [{ productId: 'prod-1', quantity: 2, price: 100 }];
    
    // Create order
    const created = await orderService.createOrder('user-1', items);
    
    expect(created.id).toBeDefined();
    expect(created.status).toBe('PENDING');
    expect(created.totalAmount).toBe(200);

    // Verify in database directly
    const dbResult = await db.query('SELECT * FROM orders WHERE id=$1', [created.id]);
    expect(dbResult.rows).toHaveLength(1);
    expect(dbResult.rows[0].status).toBe('PENDING');

    // Verify cache
    const cached = await redis.get(`order:${created.id}`);
    expect(cached).not.toBeNull();
    const parsedCached = JSON.parse(cached!);
    expect(parsedCached.id).toBe(created.id);
  });

  test('should handle concurrent order creation', async () => {
    const mockInventory = {
      checkAvailability: async () => true,
      reserveItems: async () => {},
      releaseItems: async () => {},
    };

    const mockPayment = {
      processPayment: async () => 'payment-123',
      refund: async () => {},
    };

    orderService = new OrderService(
      db,
      redis,
      new EventEmitter(),
      mockInventory,
      mockPayment
    );

    const items = [{ productId: 'prod-1', quantity: 1, price: 50 }];

    // Create multiple orders concurrently
    const orderPromises = Array.from({ length: 10 }, (_, i) =>
      orderService.createOrder(`user-${i}`, items)
    );

    const orders = await Promise.all(orderPromises);

    // All should succeed
    expect(orders).toHaveLength(10);
    expect(new Set(orders.map(o => o.id)).size).toBe(10); // All unique IDs

    // Verify in DB
    const dbCount = await db.query('SELECT COUNT(*) FROM orders');
    expect(parseInt(dbCount.rows[0].count)).toBe(10);
  });
}, 120000);

// Testcontainers with Docker Compose
import { DockerComposeEnvironment } from 'testcontainers';
import * as path from 'path';

describe('Full Stack Integration Tests', () => {
  let environment: any;

  beforeAll(async () => {
    const composeFilePath = path.join(__dirname, '../docker-compose.test.yml');
    
    environment = await new DockerComposeEnvironment(
      path.dirname(composeFilePath),
      path.basename(composeFilePath)
    )
      .withStartupTimeout(120000)
      .up();
  }, 180000);

  afterAll(async () => {
    if (environment) {
      await environment.down();
    }
  });

  test('should communicate between services', async () => {
    const orderServicePort = environment
      .getContainer('order-service_1')
      .getMappedPort(3000);

    const response = await fetch(`http://localhost:${orderServicePort}/health`);
    expect(response.ok).toBe(true);
  });
});
```

```yaml
# docker-compose.test.yml
version: '3.8'
services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: testdb
      POSTGRES_USER: testuser
      POSTGRES_PASSWORD: testpass
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U testuser -d testdb"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 5s
      retries: 5

  order-service:
    build:
      context: .
      dockerfile: Dockerfile
    environment:
      DATABASE_URL: postgres://testuser:testpass@postgres:5432/testdb
      REDIS_URL: redis://redis:6379
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    ports:
      - "3000"
```

---

## 4. Contract Tests with Pact

```typescript
// tests/contract/order-consumer.pact.test.ts
// Consumer-side contract test

import { PactV3, MatchersV3 } from '@pact-foundation/pact';
import { OrderApiClient } from '../../src/clients/order-api.client';
import path from 'path';

const { like, eachLike, string, integer, datetime } = MatchersV3;

describe('OrderApiClient - Pact Consumer Tests', () => {
  const provider = new PactV3({
    consumer: 'PaymentService',
    provider: 'OrderService',
    dir: path.resolve(process.cwd(), 'pacts'),
    logLevel: 'INFO',
  });

  describe('GET /orders/:id', () => {
    test('should return order details', async () => {
      // Define the expected interaction
      await provider
        .given('order with id order-123 exists')
        .uponReceiving('a request to get order order-123')
        .withRequest({
          method: 'GET',
          path: '/orders/order-123',
          headers: {
            Accept: 'application/json',
            Authorization: like('Bearer token123'),
          },
        })
        .willRespondWith({
          status: 200,
          headers: {
            'Content-Type': 'application/json',
          },
          body: {
            id: like('order-123'),
            userId: like('user-1'),
            items: eachLike({
              productId: string('prod-1'),
              quantity: integer(2),
              price: like(100),
            }),
            totalAmount: like(250),
            status: like('PENDING'),
            createdAt: datetime("yyyy-MM-dd'T'HH:mm:ss.SSSZ", '2024-01-01T00:00:00.000Z'),
          },
        })
        .executeTest(async (mockserver) => {
          const client = new OrderApiClient(mockserver.url);
          const order = await client.getOrder('order-123', 'Bearer token123');
          
          expect(order.id).toBe('order-123');
          expect(order.status).toBe('PENDING');
        });
    });

    test('should return 404 when order not found', async () => {
      await provider
        .given('order with id non-existent does not exist')
        .uponReceiving('a request to get non-existent order')
        .withRequest({
          method: 'GET',
          path: '/orders/non-existent',
          headers: {
            Accept: 'application/json',
          },
        })
        .willRespondWith({
          status: 404,
          body: {
            error: like('Order not found'),
          },
        })
        .executeTest(async (mockserver) => {
          const client = new OrderApiClient(mockserver.url);
          await expect(client.getOrder('non-existent', '')).rejects.toThrow('Order not found');
        });
    });
  });

  describe('POST /orders', () => {
    test('should create new order', async () => {
      await provider
        .given('user user-1 exists and products are available')
        .uponReceiving('a request to create an order')
        .withRequest({
          method: 'POST',
          path: '/orders',
          headers: {
            'Content-Type': 'application/json',
            Authorization: like('Bearer token123'),
          },
          body: {
            userId: like('user-1'),
            items: eachLike({
              productId: string('prod-1'),
              quantity: integer(2),
              price: like(100),
            }),
          },
        })
        .willRespondWith({
          status: 201,
          headers: {
            'Content-Type': 'application/json',
          },
          body: {
            id: like('new-order-id'),
            status: like('PENDING'),
            totalAmount: like(200),
          },
        })
        .executeTest(async (mockserver) => {
          const client = new OrderApiClient(mockserver.url);
          const order = await client.createOrder(
            'user-1',
            [{ productId: 'prod-1', quantity: 2, price: 100 }],
            'Bearer token123'
          );
          
          expect(order.status).toBe('PENDING');
          expect(order.id).toBeDefined();
        });
    });
  });
});
```

```typescript
// tests/contract/order-provider.pact.test.ts
// Provider-side contract verification

import { PactV3, Verifier } from '@pact-foundation/pact';
import { app } from '../../src/app';
import path from 'path';
import http from 'http';

describe('OrderService - Pact Provider Tests', () => {
  let server: http.Server;
  let port: number;

  beforeAll(async () => {
    server = http.createServer(app);
    await new Promise<void>(resolve => {
      server.listen(0, () => {
        port = (server.address() as any).port;
        resolve();
      });
    });
  });

  afterAll(async () => {
    await new Promise<void>(resolve => server.close(() => resolve()));
  });

  test('should verify pact with PaymentService', async () => {
    await new Verifier({
      provider: 'OrderService',
      providerBaseUrl: `http://localhost:${port}`,
      pactUrls: [path.resolve(process.cwd(), 'pacts/PaymentService-OrderService.json')],
      stateHandlers: {
        'order with id order-123 exists': async () => {
          // Seed test data
          await seedOrder('order-123', 'user-1');
        },
        'order with id non-existent does not exist': async () => {
          // Ensure order doesn't exist
          await deleteOrder('non-existent');
        },
        'user user-1 exists and products are available': async () => {
          // Setup state for order creation
          await seedUser('user-1');
          await seedProducts(['prod-1']);
        },
      },
      requestFilter: (req, res, next) => {
        // Add auth bypass for testing
        req.headers['x-test-bypass'] = 'true';
        next();
      },
    }).verifyProvider();
  });
});

async function seedOrder(orderId: string, userId: string): Promise<void> {
  // Implementation to seed test data
}

async function deleteOrder(orderId: string): Promise<void> {
  // Implementation to delete test data
}

async function seedUser(userId: string): Promise<void> {
  // Implementation to seed user
}

async function seedProducts(productIds: string[]): Promise<void> {
  // Implementation to seed products
}
```

---

## 5. Component Tests

```typescript
// tests/component/order.component.test.ts
// Test service in isolation with mocked external dependencies

import request from 'supertest';
import { app } from '../../src/app';
import nock from 'nock';

describe('Order Service Component Tests', () => {
  beforeEach(() => {
    nock.cleanAll();
  });

  afterAll(() => {
    nock.restore();
  });

  describe('POST /orders', () => {
    test('should create order successfully', async () => {
      // Mock inventory service
      nock('http://inventory-service:3001')
        .post('/check-availability')
        .reply(200, { available: true });

      // Mock payment service
      nock('http://payment-service:3002')
        .post('/process')
        .reply(200, { paymentId: 'pay-123', status: 'SUCCESS' });

      const response = await request(app)
        .post('/orders')
        .set('Authorization', 'Bearer valid-token')
        .send({
          userId: 'user-1',
          items: [{ productId: 'prod-1', quantity: 2, price: 100 }],
        })
        .expect(201);

      expect(response.body.status).toBe('PENDING');
      expect(response.body.id).toBeDefined();
    });

    test('should return 400 when inventory unavailable', async () => {
      nock('http://inventory-service:3001')
        .post('/check-availability')
        .reply(200, { available: false, reason: 'Out of stock' });

      const response = await request(app)
        .post('/orders')
        .set('Authorization', 'Bearer valid-token')
        .send({
          userId: 'user-1',
          items: [{ productId: 'prod-1', quantity: 100, price: 100 }],
        })
        .expect(400);

      expect(response.body.error).toContain('out of stock');
    });

    test('should handle inventory service timeout', async () => {
      nock('http://inventory-service:3001')
        .post('/check-availability')
        .delayConnection(5000) // Simulate timeout
        .reply(200, { available: true });

      const response = await request(app)
        .post('/orders')
        .set('Authorization', 'Bearer valid-token')
        .send({
          userId: 'user-1',
          items: [{ productId: 'prod-1', quantity: 2, price: 100 }],
        })
        .expect(503);

      expect(response.body.error).toContain('timeout');
    });
  });

  describe('GET /orders/:id', () => {
    test('should return order with correct format', async () => {
      const response = await request(app)
        .get('/orders/existing-order-id')
        .set('Authorization', 'Bearer valid-token')
        .expect(200);

      expect(response.body).toMatchObject({
        id: expect.any(String),
        userId: expect.any(String),
        status: expect.stringMatching(/^(PENDING|CONFIRMED|SHIPPED|DELIVERED|CANCELLED)$/),
        totalAmount: expect.any(Number),
        items: expect.arrayContaining([
          expect.objectContaining({
            productId: expect.any(String),
            quantity: expect.any(Number),
          }),
        ]),
      });
    });
  });
});
```

---

## 6. End-to-End Tests with Playwright

```typescript
// tests/e2e/checkout.e2e.test.ts
import { test, expect, Page } from '@playwright/test';

test.describe('Checkout Flow', () => {
  test.beforeEach(async ({ page }) => {
    // Login before each test
    await page.goto('/login');
    await page.fill('[data-testid="email"]', 'test@example.com');
    await page.fill('[data-testid="password"]', 'password123');
    await page.click('[data-testid="login-btn"]');
    await page.waitForURL('/dashboard');
  });

  test('should complete full checkout flow', async ({ page }) => {
    // Navigate to product
    await page.goto('/products/prod-1');
    await expect(page.locator('h1')).toContainText('Product Name');

    // Add to cart
    await page.click('[data-testid="add-to-cart"]');
    await expect(page.locator('[data-testid="cart-count"]')).toHaveText('1');

    // Go to cart
    await page.click('[data-testid="cart-icon"]');
    await expect(page.url()).toContain('/cart');

    // Verify cart contents
    await expect(page.locator('[data-testid="cart-item"]')).toHaveCount(1);
    await expect(page.locator('[data-testid="cart-total"]')).toBeVisible();

    // Proceed to checkout
    await page.click('[data-testid="checkout-btn"]');
    await expect(page.url()).toContain('/checkout');

    // Fill shipping info
    await page.fill('[data-testid="shipping-name"]', 'Test User');
    await page.fill('[data-testid="shipping-address"]', '123 Test St');
    await page.fill('[data-testid="shipping-city"]', 'Bangkok');

    // Fill payment info
    await page.fill('[data-testid="card-number"]', '4242424242424242');
    await page.fill('[data-testid="card-expiry"]', '12/25');
    await page.fill('[data-testid="card-cvv"]', '123');

    // Submit order
    await page.click('[data-testid="place-order-btn"]');

    // Wait for confirmation
    await page.waitForURL('/order-confirmation/**');
    await expect(page.locator('[data-testid="order-id"]')).toBeVisible();
    await expect(page.locator('[data-testid="success-message"]')).toContainText(
      'Your order has been placed'
    );
  });

  test('should handle payment failure gracefully', async ({ page }) => {
    await page.goto('/checkout');
    
    // Fill with declined card
    await page.fill('[data-testid="card-number"]', '4000000000000002'); // Stripe test declined card
    await page.fill('[data-testid="card-expiry"]', '12/25');
    await page.fill('[data-testid="card-cvv"]', '123');

    await page.click('[data-testid="place-order-btn"]');

    await expect(page.locator('[data-testid="payment-error"]')).toBeVisible();
    await expect(page.locator('[data-testid="payment-error"]')).toContainText(
      'Payment declined'
    );
  });

  test('should handle concurrent users placing orders', async ({ browser }) => {
    // Create multiple contexts to simulate concurrent users
    const contexts = await Promise.all(
      Array.from({ length: 3 }, () => browser.newContext())
    );
    
    const pages = await Promise.all(contexts.map(ctx => ctx.newPage()));
    
    // All users navigate to same product
    await Promise.all(pages.map(p => p.goto('/products/limited-stock-product')));
    
    // All click add to cart simultaneously
    await Promise.all(pages.map(p => p.click('[data-testid="add-to-cart"]')));
    
    // Verify stock handling
    const results = await Promise.all(
      pages.map(async (p) => {
        try {
          await p.click('[data-testid="checkout-btn"]');
          return 'success';
        } catch {
          return 'failed';
        }
      })
    );
    
    // At most the available stock should succeed
    const successes = results.filter(r => r === 'success');
    expect(successes.length).toBeGreaterThanOrEqual(1);
    
    await Promise.all(contexts.map(ctx => ctx.close()));
  });
});
```

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests/e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: [
    ['html', { outputFolder: 'playwright-report' }],
    ['junit', { outputFile: 'test-results/e2e-results.xml' }],
  ],
  use: {
    baseURL: process.env.BASE_URL || 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] },
    },
    {
      name: 'Mobile Chrome',
      use: { ...devices['Pixel 5'] },
    },
  ],
  webServer: {
    command: 'npm run start:test',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
});
```

---

## 7. Chaos Testing with Chaos Monkey

```typescript
// tests/chaos/chaos-monkey.ts
import Chance from 'chance';

interface ChaosConfig {
  networkLatencyMs?: number;
  networkErrorRate?: number;
  serviceFailureRate?: number;
  memoryPressure?: boolean;
  cpuPressure?: boolean;
}

class ChaosMonkey {
  private config: ChaosConfig;
  private chance: Chance.Chance;

  constructor(config: ChaosConfig = {}) {
    this.config = config;
    this.chance = new Chance();
  }

  // Middleware สำหรับ Express
  expressMiddleware() {
    return (req: any, res: any, next: any) => {
      // Random network latency
      if (this.config.networkLatencyMs && this.chance.bool({ likelihood: 30 })) {
        const delay = this.chance.integer({
          min: 0,
          max: this.config.networkLatencyMs,
        });
        setTimeout(next, delay);
        return;
      }

      // Random errors
      if (
        this.config.networkErrorRate &&
        this.chance.bool({ likelihood: this.config.networkErrorRate * 100 })
      ) {
        res.status(503).json({ error: 'Chaos Monkey: Service temporarily unavailable' });
        return;
      }

      next();
    };
  }

  // Wrap function with chaos
  wrapWithChaos<T>(fn: () => Promise<T>): () => Promise<T> {
    return async () => {
      // Random failure
      if (
        this.config.serviceFailureRate &&
        this.chance.bool({ likelihood: this.config.serviceFailureRate * 100 })
      ) {
        throw new Error('Chaos Monkey: Injected failure');
      }

      // Random latency
      if (this.config.networkLatencyMs) {
        const delay = this.chance.integer({
          min: 0,
          max: this.config.networkLatencyMs,
        });
        await new Promise(resolve => setTimeout(resolve, delay));
      }

      return fn();
    };
  }
}

// Chaos Test for Circuit Breaker
describe('Circuit Breaker with Chaos', () => {
  let chaosMonkey: ChaosMonkey;

  beforeEach(() => {
    chaosMonkey = new ChaosMonkey({
      serviceFailureRate: 0.5, // 50% failure rate
      networkLatencyMs: 2000,
    });
  });

  test('circuit breaker should open after repeated failures', async () => {
    const circuitBreaker = new CircuitBreaker({
      failureThreshold: 5,
      timeout: 10000,
      retryTime: 5000,
    });

    let failures = 0;
    let circuitOpened = false;

    for (let i = 0; i < 20; i++) {
      try {
        await circuitBreaker.execute(
          chaosMonkey.wrapWithChaos(async () => {
            return { status: 'OK' };
          })
        );
      } catch (error: any) {
        if (error.message.includes('Circuit breaker is OPEN')) {
          circuitOpened = true;
          break;
        }
        failures++;
      }
    }

    expect(circuitOpened).toBe(true);
    expect(failures).toBeGreaterThanOrEqual(5);
  });
});

class CircuitBreaker {
  private state: 'CLOSED' | 'OPEN' | 'HALF_OPEN' = 'CLOSED';
  private failureCount = 0;
  private lastFailureTime?: number;
  private config: {
    failureThreshold: number;
    timeout: number;
    retryTime: number;
  };

  constructor(config: { failureThreshold: number; timeout: number; retryTime: number }) {
    this.config = config;
  }

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === 'OPEN') {
      if (Date.now() - (this.lastFailureTime || 0) > this.config.retryTime) {
        this.state = 'HALF_OPEN';
      } else {
        throw new Error('Circuit breaker is OPEN');
      }
    }

    try {
      const result = await Promise.race([
        fn(),
        new Promise<never>((_, reject) =>
          setTimeout(() => reject(new Error('Timeout')), this.config.timeout)
        ),
      ]);

      if (this.state === 'HALF_OPEN') {
        this.state = 'CLOSED';
        this.failureCount = 0;
      }

      return result;
    } catch (error) {
      this.failureCount++;
      this.lastFailureTime = Date.now();

      if (this.failureCount >= this.config.failureThreshold) {
        this.state = 'OPEN';
      }

      throw error;
    }
  }
}
```

---

## 8. Performance Testing with k6

```javascript
// tests/performance/load-test.js
import http from 'k6/http';
import { check, sleep, group } from 'k6';
import { Rate, Trend, Counter } from 'k6/metrics';

// Custom metrics
const errorRate = new Rate('error_rate');
const orderCreationTime = new Trend('order_creation_time');
const orderCreations = new Counter('order_creations');

export const options = {
  stages: [
    { duration: '1m', target: 10 },   // Ramp up to 10 users
    { duration: '3m', target: 50 },   // Ramp up to 50 users
    { duration: '5m', target: 100 },  // Ramp up to 100 users
    { duration: '2m', target: 100 },  // Stay at 100 users
    { duration: '1m', target: 0 },    // Ramp down to 0 users
  ],
  thresholds: {
    http_req_duration: ['p(95)<500', 'p(99)<1000'], // 95% < 500ms
    error_rate: ['rate<0.01'],                        // Error rate < 1%
    order_creation_time: ['p(95)<1000'],              // 95% of orders < 1s
  },
};

const BASE_URL = __ENV.BASE_URL || 'http://localhost:3000';

export default function () {
  group('User Journey', () => {
    // Get product list
    group('Browse Products', () => {
      const productsRes = http.get(`${BASE_URL}/products`, {
        headers: { Accept: 'application/json' },
      });
      
      check(productsRes, {
        'products status 200': (r) => r.status === 200,
        'products has data': (r) => JSON.parse(r.body).length > 0,
      });
      
      errorRate.add(productsRes.status !== 200);
    });

    sleep(1);

    // Create order
    group('Create Order', () => {
      const startTime = Date.now();
      
      const orderRes = http.post(
        `${BASE_URL}/orders`,
        JSON.stringify({
          userId: `user-${__VU}`,
          items: [
            { productId: 'prod-1', quantity: 1, price: 100 },
          ],
        }),
        {
          headers: {
            'Content-Type': 'application/json',
            Authorization: 'Bearer test-token',
          },
        }
      );

      const duration = Date.now() - startTime;
      orderCreationTime.add(duration);

      const isSuccess = check(orderRes, {
        'order created successfully': (r) => r.status === 201,
        'order has id': (r) => JSON.parse(r.body).id !== undefined,
      });

      if (isSuccess) {
        orderCreations.add(1);
      }

      errorRate.add(orderRes.status !== 201);

      // Get the order
      if (orderRes.status === 201) {
        const orderId = JSON.parse(orderRes.body).id;
        
        const getOrderRes = http.get(`${BASE_URL}/orders/${orderId}`, {
          headers: { Authorization: 'Bearer test-token' },
        });

        check(getOrderRes, {
          'get order status 200': (r) => r.status === 200,
          'order id matches': (r) => JSON.parse(r.body).id === orderId,
        });
      }
    });

    sleep(2);
  });
}

export function handleSummary(data) {
  return {
    'tests/performance/results/load-test-summary.json': JSON.stringify(data, null, 2),
    stdout: textSummary(data, { indent: '  ', enableColors: true }),
  };
}

function textSummary(data, opts) {
  return `
Load Test Summary
=================
Total requests: ${data.metrics.http_reqs.values.count}
Error rate: ${(data.metrics.error_rate.values.rate * 100).toFixed(2)}%
Avg response time: ${data.metrics.http_req_duration.values.avg.toFixed(2)}ms
P95 response time: ${data.metrics.http_req_duration.values['p(95)'].toFixed(2)}ms
P99 response time: ${data.metrics.http_req_duration.values['p(99)'].toFixed(2)}ms
Orders created: ${data.metrics.order_creations ? data.metrics.order_creations.values.count : 0}
  `.trim();
}
```

```yaml
# k6 deployment in Kubernetes
apiVersion: batch/v1
kind: Job
metadata:
  name: k6-load-test
  namespace: testing
spec:
  template:
    spec:
      containers:
        - name: k6
          image: grafana/k6:latest
          command: ["k6", "run", "--out", "influxdb=http://influxdb:8086/k6", "/scripts/load-test.js"]
          env:
            - name: BASE_URL
              value: "http://order-service.microservices.svc.cluster.local:3000"
          volumeMounts:
            - name: scripts
              mountPath: /scripts
      volumes:
        - name: scripts
          configMap:
            name: k6-scripts
      restartPolicy: Never
```

---

## 9. Mutation Testing with Stryker

```typescript
// stryker.config.ts
import type { Config } from '@stryker-mutator/api/config';

const config: Config = {
  testRunner: 'jest',
  jest: {
    projectType: 'custom',
    configFile: 'jest.config.ts',
  },
  reporters: ['html', 'clear-text', 'progress', 'dashboard'],
  coverageAnalysis: 'perTest',
  mutate: ['src/**/*.ts', '!src/**/*.d.ts', '!src/index.ts'],
  thresholds: {
    high: 80,
    low: 60,
    break: 50,
  },
  // Mutation operators
  plugins: ['@stryker-mutator/jest-runner'],
  timeoutMS: 60000,
  timeoutFactor: 1.5,
  // ไม่ mutate test files
  ignorePatterns: ['node_modules', 'tests', 'dist'],
};

export default config;
```

```bash
# รัน mutation testing
npx stryker run

# รัน สำหรับ specific file
npx stryker run --mutate "src/services/order.service.ts"
```

---

## 10. Test Data Management

```typescript
// tests/helpers/test-data-factory.ts
import { faker } from '@faker-js/faker';
import { Pool } from 'pg';
import { Order, OrderItem, OrderStatus } from '../../src/services/order.service';

// Factory pattern สำหรับ test data
class OrderFactory {
  static create(overrides: Partial<Order> = {}): Order {
    const items = overrides.items || OrderFactory.createItems(2);
    const totalAmount = items.reduce(
      (sum, item) => sum + item.price * item.quantity,
      0
    );

    return {
      id: faker.string.uuid(),
      userId: faker.string.uuid(),
      items,
      totalAmount,
      status: 'PENDING',
      createdAt: faker.date.recent(),
      updatedAt: faker.date.recent(),
      ...overrides,
    };
  }

  static createItems(count: number = 1): OrderItem[] {
    return Array.from({ length: count }, () => ({
      productId: faker.string.uuid(),
      quantity: faker.number.int({ min: 1, max: 10 }),
      price: faker.number.float({ min: 10, max: 1000, precision: 0.01 }),
    }));
  }

  static createWithStatus(status: OrderStatus): Order {
    return OrderFactory.create({ status });
  }

  static createMany(count: number): Order[] {
    return Array.from({ length: count }, () => OrderFactory.create());
  }
}

// Database Seeder
class TestDataSeeder {
  constructor(private db: Pool) {}

  async seedOrders(count: number = 10): Promise<Order[]> {
    const orders = OrderFactory.createMany(count);
    
    const insertQueries = orders.map(order =>
      this.db.query(
        `INSERT INTO orders (id, user_id, items, total_amount, status, created_at, updated_at)
         VALUES ($1, $2, $3, $4, $5, $6, $7)`,
        [
          order.id,
          order.userId,
          JSON.stringify(order.items),
          order.totalAmount,
          order.status,
          order.createdAt,
          order.updatedAt,
        ]
      )
    );

    await Promise.all(insertQueries);
    return orders;
  }

  async cleanAll(): Promise<void> {
    await this.db.query('DELETE FROM orders');
    await this.db.query('DELETE FROM users');
    await this.db.query('DELETE FROM products');
  }
}

// Snapshot Testing
describe('Order Serialization', () => {
  test('should serialize order to expected format', () => {
    const order = OrderFactory.create({
      id: 'fixed-id',
      userId: 'fixed-user',
      createdAt: new Date('2024-01-01T00:00:00.000Z'),
      updatedAt: new Date('2024-01-01T00:00:00.000Z'),
    });

    expect(order).toMatchSnapshot();
  });

  test('should match API response snapshot', async () => {
    const response = await request(app)
      .get('/orders/fixed-id')
      .set('Authorization', 'Bearer test-token');

    expect(response.body).toMatchSnapshot();
  });
});

// Test Fixtures
export const fixtures = {
  orders: {
    pending: OrderFactory.createWithStatus('PENDING'),
    confirmed: OrderFactory.createWithStatus('CONFIRMED'),
    shipped: OrderFactory.createWithStatus('SHIPPED'),
    delivered: OrderFactory.createWithStatus('DELIVERED'),
    cancelled: OrderFactory.createWithStatus('CANCELLED'),
  },
  users: {
    regular: { id: 'user-1', email: 'user@example.com', role: 'USER' },
    admin: { id: 'admin-1', email: 'admin@example.com', role: 'ADMIN' },
  },
};
```

---

## CI/CD Pipeline สำหรับ Testing

```yaml
# .github/workflows/test.yml
name: Test Suite

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  unit-tests:
    name: Unit Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run test:unit -- --coverage
      - uses: codecov/codecov-action@v3
        with:
          file: ./coverage/lcov.info

  integration-tests:
    name: Integration Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run test:integration
        env:
          TESTCONTAINERS_RYUK_DISABLED: true

  contract-tests:
    name: Contract Tests
    runs-on: ubuntu-latest
    needs: [unit-tests]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run test:contract
      - name: Publish pacts
        run: npm run pact:publish
        env:
          PACT_BROKER_URL: ${{ secrets.PACT_BROKER_URL }}
          PACT_BROKER_TOKEN: ${{ secrets.PACT_BROKER_TOKEN }}

  e2e-tests:
    name: E2E Tests
    runs-on: ubuntu-latest
    needs: [integration-tests]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npx playwright install --with-deps
      - name: Start services
        run: docker-compose -f docker-compose.test.yml up -d
      - name: Wait for services
        run: ./scripts/wait-for-services.sh
      - run: npm run test:e2e
      - uses: actions/upload-artifact@v3
        if: failure()
        with:
          name: playwright-report
          path: playwright-report/
```

---

## สรุปตาราง Testing Strategy

| Test Type | Tool | Scope | Speed | Cost | Run When |
|-----------|------|-------|-------|------|----------|
| Unit | Jest | Function/Class | Fast (<1s) | ต่ำ | ทุก commit |
| Integration | Testcontainers | Service+DB | Medium (30s) | ปานกลาง | ทุก PR |
| Contract | Pact | Service boundary | Medium | ปานกลาง | ทุก PR |
| Component | Supertest+Nock | Service isolated | Medium | ปานกลาง | ทุก PR |
| E2E | Playwright | Full system | Slow (mins) | สูง | Pre-release |
| Chaos | Custom/Chaos Toolkit | Resilience | Slow | สูง | Weekly |
| Performance | k6 | Load | Slow | สูง | Weekly/Release |
| Mutation | Stryker | Code quality | Very slow | สูง | Weekly |

| Metric | Target | Tool |
|--------|--------|------|
| Unit test coverage | >85% | Jest coverage |
| Contract test coverage | 100% APIs | Pact |
| E2E critical paths | 100% | Playwright |
| P95 latency | <500ms | k6 |
| Error rate | <1% | k6 |
| Mutation score | >80% | Stryker |

---

## สรุป

Testing Strategy สำหรับ Microservices ต้องเป็น multi-layered:

1. **Unit Tests** เป็น foundation - เร็ว, ครอบคลุม logic
2. **Integration Tests** ด้วย Testcontainers ทดสอบกับ real dependencies
3. **Contract Tests** ด้วย Pact รับประกัน service compatibility
4. **Component Tests** ทดสอบ service behavior แบบ isolated
5. **E2E Tests** ด้วย Playwright ครอบคลุม user journeys
6. **Chaos Testing** ทดสอบ resilience
7. **Performance Testing** ด้วย k6 ตรวจสอบ SLOs
8. **Mutation Testing** ด้วย Stryker วัดคุณภาพของ tests

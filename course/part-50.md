# Part 50: Microservices Testing ขั้นสูง — TestContainers, Contract Testing, Performance, และ Chaos Engineering

ในบทนี้เราจะเรียนรู้การทดสอบ Microservices อย่างครบวงจร ตั้งแต่ Integration Testing ด้วย TestContainers, Consumer-Driven Contract Testing ด้วย Pact.js, Performance Testing ด้วย k6, ไปจนถึง Chaos Engineering

---

## 1. TestContainers สำหรับ Integration Tests

TestContainers ช่วยให้เราสามารถ spin up จริงๆ ของ dependencies เช่น PostgreSQL, Redis, Kafka ใน Docker containers ระหว่าง test โดยไม่ต้องใช้ mock

### 1.1 Setup TestContainers

```bash
npm install testcontainers \
  @testcontainers/postgresql \
  @testcontainers/redis \
  @testcontainers/kafka \
  vitest \
  @vitest/coverage-v8
```

### 1.2 PostgreSQL Integration Tests

```typescript
// src/tests/integration/user-repository.test.ts
import { describe, it, expect, beforeAll, afterAll, beforeEach } from 'vitest';
import { PostgreSqlContainer, StartedPostgreSqlContainer } from '@testcontainers/postgresql';
import { GenericContainer, StartedTestContainer, Wait } from 'testcontainers';
import knex, { Knex } from 'knex';
import { UserRepository } from '../../repositories/user.repository';
import { runMigrations } from '../../database/migrations';

describe('UserRepository Integration Tests', () => {
  let container: StartedPostgreSqlContainer;
  let db: Knex;
  let userRepo: UserRepository;
  
  beforeAll(async () => {
    // Start PostgreSQL container
    container = await new PostgreSqlContainer('postgres:15-alpine')
      .withDatabase('testdb')
      .withUsername('testuser')
      .withPassword('testpass')
      .withExposedPorts(5432)
      .withStartupTimeout(60000)
      .start();
    
    // Connect to container
    db = knex({
      client: 'postgresql',
      connection: {
        host: container.getHost(),
        port: container.getMappedPort(5432),
        database: container.getDatabase(),
        user: container.getUsername(),
        password: container.getPassword(),
      },
      pool: { min: 2, max: 10 },
    });
    
    // Run migrations
    await runMigrations(db);
    
    userRepo = new UserRepository(db);
  }, 120000); // 2 minute timeout for container startup
  
  afterAll(async () => {
    await db.destroy();
    await container.stop();
  });
  
  beforeEach(async () => {
    // Clean state before each test
    await db.raw('TRUNCATE TABLE users, user_profiles CASCADE');
  });
  
  describe('create', () => {
    it('should create user with hashed password', async () => {
      const user = await userRepo.create({
        email: 'test@example.com',
        password: 'SecurePass123!',
        firstName: 'John',
        lastName: 'Doe',
      });
      
      expect(user).toMatchObject({
        id: expect.any(String),
        email: 'test@example.com',
        firstName: 'John',
        lastName: 'Doe',
        createdAt: expect.any(Date),
      });
      
      // Password should be hashed
      expect(user.passwordHash).not.toBe('SecurePass123!');
      expect(user).not.toHaveProperty('password');
    });
    
    it('should throw on duplicate email', async () => {
      await userRepo.create({
        email: 'duplicate@example.com',
        password: 'Pass123!',
        firstName: 'User',
        lastName: 'One',
      });
      
      await expect(
        userRepo.create({
          email: 'duplicate@example.com',
          password: 'Pass456!',
          firstName: 'User',
          lastName: 'Two',
        })
      ).rejects.toThrow(/duplicate key/i);
    });
    
    it('should handle concurrent creates correctly', async () => {
      const users = await Promise.all(
        Array.from({ length: 10 }, (_, i) =>
          userRepo.create({
            email: `user${i}@example.com`,
            password: 'Pass123!',
            firstName: `User${i}`,
            lastName: 'Test',
          })
        )
      );
      
      expect(users).toHaveLength(10);
      const ids = new Set(users.map(u => u.id));
      expect(ids.size).toBe(10); // All unique IDs
    });
  });
  
  describe('search', () => {
    beforeEach(async () => {
      // Seed test data
      await Promise.all([
        userRepo.create({ email: 'alice@example.com', password: 'P', firstName: 'Alice', lastName: 'Smith' }),
        userRepo.create({ email: 'bob@example.com', password: 'P', firstName: 'Bob', lastName: 'Jones' }),
        userRepo.create({ email: 'charlie@example.com', password: 'P', firstName: 'Charlie', lastName: 'Smith' }),
      ]);
    });
    
    it('should search by last name', async () => {
      const results = await userRepo.search({ lastName: 'Smith' });
      
      expect(results.users).toHaveLength(2);
      expect(results.users.map(u => u.firstName)).toEqual(
        expect.arrayContaining(['Alice', 'Charlie'])
      );
    });
    
    it('should paginate results', async () => {
      const page1 = await userRepo.search({ limit: 2, offset: 0 });
      const page2 = await userRepo.search({ limit: 2, offset: 2 });
      
      expect(page1.users).toHaveLength(2);
      expect(page2.users).toHaveLength(1);
      expect(page1.total).toBe(3);
      
      // No overlap between pages
      const ids1 = new Set(page1.users.map(u => u.id));
      const ids2 = new Set(page2.users.map(u => u.id));
      expect([...ids1].filter(id => ids2.has(id))).toHaveLength(0);
    });
  });
});
```

### 1.3 Redis Integration Tests

```typescript
// src/tests/integration/cache.test.ts
import { RedisContainer, StartedRedisContainer } from '@testcontainers/redis';
import { describe, it, expect, beforeAll, afterAll } from 'vitest';
import { Redis } from 'ioredis';
import { CacheService } from '../../services/cache.service';

describe('CacheService Integration Tests', () => {
  let container: StartedRedisContainer;
  let redis: Redis;
  let cacheService: CacheService;
  
  beforeAll(async () => {
    container = await new RedisContainer('redis:7-alpine')
      .withExposedPorts(6379)
      .withStartupTimeout(30000)
      .start();
    
    redis = new Redis({
      host: container.getHost(),
      port: container.getMappedPort(6379),
    });
    
    cacheService = new CacheService(redis);
  }, 60000);
  
  afterAll(async () => {
    await redis.quit();
    await container.stop();
  });
  
  it('should set and get values', async () => {
    await cacheService.set('test-key', { data: 'value' }, 60);
    const result = await cacheService.get<{ data: string }>('test-key');
    
    expect(result).toEqual({ data: 'value' });
  });
  
  it('should respect TTL', async () => {
    await cacheService.set('ttl-key', 'expire-me', 1);
    
    await new Promise(r => setTimeout(r, 1500));
    
    const result = await cacheService.get('ttl-key');
    expect(result).toBeNull();
  });
  
  it('should handle cache stampede with mutex', async () => {
    let computeCount = 0;
    
    const expensiveCompute = async () => {
      computeCount++;
      await new Promise(r => setTimeout(r, 100));
      return { value: 'expensive-result' };
    };
    
    // Simulate concurrent requests
    const results = await Promise.all(
      Array.from({ length: 10 }, () =>
        cacheService.getOrSet('mutex-key', expensiveCompute, 60)
      )
    );
    
    // All should get same result
    expect(results.every(r => r.value === 'expensive-result')).toBe(true);
    
    // Should only compute once (or very few times with race conditions)
    expect(computeCount).toBeLessThanOrEqual(3);
  });
});
```

### 1.4 Kafka Integration Tests

```typescript
// src/tests/integration/event-bus.test.ts
import {
  KafkaContainer,
  StartedKafkaContainer,
} from '@testcontainers/kafka';
import { describe, it, expect, beforeAll, afterAll } from 'vitest';
import { Kafka, Partitioners } from 'kafkajs';
import { EventBus } from '../../messaging/event-bus';

describe('EventBus Kafka Integration', () => {
  let container: StartedKafkaContainer;
  let kafka: Kafka;
  let eventBus: EventBus;
  
  beforeAll(async () => {
    container = await new KafkaContainer('confluentinc/cp-kafka:7.5.0')
      .withExposedPorts(9093)
      .withStartupTimeout(120000)
      .start();
    
    kafka = new Kafka({
      brokers: [container.getBootstrapAddress()],
      clientId: 'test-client',
    });
    
    eventBus = new EventBus(kafka);
    await eventBus.connect();
  }, 150000);
  
  afterAll(async () => {
    await eventBus.disconnect();
    await container.stop();
  });
  
  it('should publish and consume events', async () => {
    const received: any[] = [];
    
    await eventBus.subscribe(
      'test-topic',
      'test-group',
      async (event) => {
        received.push(event);
      }
    );
    
    await eventBus.publish('test-topic', {
      type: 'TEST_EVENT',
      payload: { message: 'hello' },
      timestamp: new Date().toISOString(),
    });
    
    // Wait for consumption
    await new Promise(r => setTimeout(r, 3000));
    
    expect(received).toHaveLength(1);
    expect(received[0].type).toBe('TEST_EVENT');
    expect(received[0].payload.message).toBe('hello');
  }, 30000);
  
  it('should handle message ordering within a partition', async () => {
    const received: number[] = [];
    const PARTITION_KEY = 'order-123'; // Same key = same partition = ordered
    
    await eventBus.subscribe(
      'ordering-test',
      'ordering-group',
      async (event) => {
        received.push(event.sequence);
      }
    );
    
    // Publish 10 ordered messages to same partition
    for (let i = 0; i < 10; i++) {
      await eventBus.publish('ordering-test', {
        sequence: i,
        partitionKey: PARTITION_KEY,
      });
    }
    
    await new Promise(r => setTimeout(r, 5000));
    
    // Verify order preserved
    expect(received).toEqual([0, 1, 2, 3, 4, 5, 6, 7, 8, 9]);
  }, 30000);
});
```

---

## 2. Pact.js Consumer-Driven Contract Testing

Contract Testing ช่วยให้ Consumer และ Provider ตกลงกัน API contract โดยไม่ต้องรอให้ Provider พร้อมก่อน

### 2.1 Consumer Side

```typescript
// src/tests/contracts/consumer/order-service-consumer.pact.test.ts
import { PactV3, MatchersV3 } from '@pact-foundation/pact';
import { describe, it, expect, beforeAll, afterAll } from 'vitest';
import path from 'path';
import { ProductClient } from '../../../clients/product.client';

const { like, eachLike, string, integer, decimal, datetime, regex } = MatchersV3;

const provider = new PactV3({
  consumer: 'OrderService',
  provider: 'ProductService',
  dir: path.resolve(__dirname, '../../../../pacts'),
  port: 8080,
  logLevel: 'warn',
});

describe('OrderService → ProductService Contract', () => {
  let productClient: ProductClient;
  
  beforeAll(() => {
    productClient = new ProductClient(`http://localhost:8080`);
  });
  
  describe('GET /products/:id', () => {
    it('returns product details for valid product ID', async () => {
      await provider.addInteraction({
        states: [{ description: 'product 123 exists' }],
        uponReceiving: 'a request for product details',
        withRequest: {
          method: 'GET',
          path: '/products/123',
          headers: {
            Accept: 'application/json',
            Authorization: regex('Bearer [A-Za-z0-9._-]+', 'Bearer valid-token'),
          },
        },
        willRespondWith: {
          status: 200,
          headers: { 'Content-Type': 'application/json' },
          body: like({
            id: string('123'),
            name: string('Premium Widget'),
            price: decimal(29.99),
            currency: string('USD'),
            stock: integer(100),
            sku: regex('[A-Z]{3}-[0-9]{6}', 'WID-001234'),
            category: like({
              id: string('cat-456'),
              name: string('Electronics'),
            }),
            createdAt: datetime("yyyy-MM-dd'T'HH:mm:ss.SSSSSSXXX", '2024-01-15T10:30:00.000000+00:00'),
          }),
        },
      });
      
      await provider.executeTest(async () => {
        const product = await productClient.getProduct('123');
        
        expect(product.id).toBe('123');
        expect(product.name).toBeTruthy();
        expect(typeof product.price).toBe('number');
        expect(product.stock).toBeGreaterThanOrEqual(0);
      });
    });
    
    it('returns 404 for non-existent product', async () => {
      await provider.addInteraction({
        states: [{ description: 'product 999 does not exist' }],
        uponReceiving: 'a request for a non-existent product',
        withRequest: {
          method: 'GET',
          path: '/products/999',
          headers: { Accept: 'application/json' },
        },
        willRespondWith: {
          status: 404,
          headers: { 'Content-Type': 'application/json' },
          body: like({
            error: string('not_found'),
            message: string('Product not found'),
          }),
        },
      });
      
      await provider.executeTest(async () => {
        await expect(productClient.getProduct('999'))
          .rejects.toThrow(/not found/i);
      });
    });
  });
  
  describe('POST /products/batch', () => {
    it('returns multiple products for batch request', async () => {
      await provider.addInteraction({
        states: [{ description: 'products 123 and 456 exist' }],
        uponReceiving: 'a batch products request',
        withRequest: {
          method: 'POST',
          path: '/products/batch',
          headers: {
            'Content-Type': 'application/json',
            Authorization: regex('Bearer [A-Za-z0-9._-]+', 'Bearer valid-token'),
          },
          body: {
            ids: ['123', '456'],
          },
        },
        willRespondWith: {
          status: 200,
          headers: { 'Content-Type': 'application/json' },
          body: {
            products: eachLike({
              id: string('123'),
              name: string('Product Name'),
              price: decimal(9.99),
              currency: string('USD'),
              stock: integer(10),
            }),
          },
        },
      });
      
      await provider.executeTest(async () => {
        const result = await productClient.getBatchProducts(['123', '456']);
        
        expect(result.products).toHaveLength(2);
        expect(result.products[0]).toHaveProperty('id');
        expect(result.products[0]).toHaveProperty('price');
      });
    });
  });
  
  describe('POST /inventory/reserve', () => {
    it('successfully reserves inventory for order', async () => {
      await provider.addInteraction({
        states: [
          { description: 'product 123 has sufficient stock (100 units)' },
        ],
        uponReceiving: 'a request to reserve inventory for an order',
        withRequest: {
          method: 'POST',
          path: '/inventory/reserve',
          headers: { 'Content-Type': 'application/json' },
          body: {
            orderId: string('order-789'),
            items: eachLike({
              productId: string('123'),
              quantity: integer(5),
            }),
          },
        },
        willRespondWith: {
          status: 200,
          headers: { 'Content-Type': 'application/json' },
          body: like({
            reservationId: string('res-abc-123'),
            orderId: string('order-789'),
            status: string('reserved'),
            expiresAt: datetime("yyyy-MM-dd'T'HH:mm:ss.SSSSSSXXX"),
          }),
        },
      });
      
      await provider.executeTest(async () => {
        const reservation = await productClient.reserveInventory({
          orderId: 'order-789',
          items: [{ productId: '123', quantity: 5 }],
        });
        
        expect(reservation.status).toBe('reserved');
        expect(reservation.reservationId).toBeTruthy();
      });
    });
    
    it('fails when insufficient stock', async () => {
      await provider.addInteraction({
        states: [
          { description: 'product 123 has only 2 units in stock' },
        ],
        uponReceiving: 'a request to reserve more inventory than available',
        withRequest: {
          method: 'POST',
          path: '/inventory/reserve',
          headers: { 'Content-Type': 'application/json' },
          body: {
            orderId: string('order-999'),
            items: eachLike({
              productId: string('123'),
              quantity: integer(100),
            }),
          },
        },
        willRespondWith: {
          status: 409,
          headers: { 'Content-Type': 'application/json' },
          body: like({
            error: string('insufficient_stock'),
            details: eachLike({
              productId: string('123'),
              available: integer(2),
              requested: integer(100),
            }),
          }),
        },
      });
      
      await provider.executeTest(async () => {
        await expect(
          productClient.reserveInventory({
            orderId: 'order-999',
            items: [{ productId: '123', quantity: 100 }],
          })
        ).rejects.toThrow(/insufficient/i);
      });
    });
  });
});
```

### 2.2 Provider Side Verification

```typescript
// src/tests/contracts/provider/product-service-provider.pact.test.ts
import { Verifier } from '@pact-foundation/pact';
import { describe, it } from 'vitest';
import path from 'path';
import { createApp } from '../../../app';
import { db } from '../../../database';

describe('ProductService Contract Verification', () => {
  it('verifies consumer contracts', async () => {
    const app = createApp();
    const server = app.listen(9001);
    
    try {
      const verifier = new Verifier({
        providerBaseUrl: 'http://localhost:9001',
        provider: 'ProductService',
        
        // Read pacts from file system (CI: from Pact Broker)
        pactUrls: [
          path.resolve(__dirname, '../../../../pacts/OrderService-ProductService.json'),
        ],
        
        // Or from Pact Broker
        // pactBrokerUrl: process.env.PACT_BROKER_URL,
        // pactBrokerToken: process.env.PACT_BROKER_TOKEN,
        // consumerVersionSelectors: [
        //   { mainBranch: true },
        //   { deployedOrReleased: true },
        // ],
        
        // State handlers to set up test data
        stateHandlers: {
          'product 123 exists': async () => {
            await db('products').insert({
              id: '123',
              name: 'Premium Widget',
              price: 29.99,
              currency: 'USD',
              stock: 100,
              sku: 'WID-001234',
              category_id: 'cat-456',
            }).onConflict('id').merge();
            
            await db('categories').insert({
              id: 'cat-456',
              name: 'Electronics',
            }).onConflict('id').merge();
          },
          
          'product 999 does not exist': async () => {
            await db('products').where({ id: '999' }).delete();
          },
          
          'products 123 and 456 exist': async () => {
            await db('products').insert([
              { id: '123', name: 'Product A', price: 9.99, currency: 'USD', stock: 50 },
              { id: '456', name: 'Product B', price: 19.99, currency: 'USD', stock: 25 },
            ]).onConflict('id').merge();
          },
          
          'product 123 has sufficient stock (100 units)': async () => {
            await db('products')
              .where({ id: '123' })
              .update({ stock: 100 });
          },
          
          'product 123 has only 2 units in stock': async () => {
            await db('products')
              .where({ id: '123' })
              .update({ stock: 2 });
          },
        },
        
        publishVerificationResult: process.env.CI === 'true',
        providerVersion: process.env.GIT_COMMIT || '0.0.0',
        providerVersionBranch: process.env.GIT_BRANCH || 'local',
      });
      
      await verifier.verifyProvider();
    } finally {
      server.close();
    }
  }, 60000);
});
```

---

## 3. k6 Performance Test Scenarios

```javascript
// tests/performance/order-service.k6.js
import http from 'k6/http';
import { check, sleep, group } from 'k6';
import { Rate, Trend, Counter } from 'k6/metrics';
import { SharedArray } from 'k6/data';

// Custom metrics
const errorRate = new Rate('errors');
const orderDuration = new Trend('order_creation_duration');
const successfulOrders = new Counter('successful_orders');

// Load test users from file
const users = new SharedArray('users', function() {
  return JSON.parse(open('./test-data/users.json'));
});

const products = new SharedArray('products', function() {
  return JSON.parse(open('./test-data/products.json'));
});

// Test configuration
export const options = {
  scenarios: {
    // Smoke test: basic functionality check
    smoke: {
      executor: 'constant-vus',
      vus: 1,
      duration: '1m',
      tags: { scenario: 'smoke' },
      env: { SCENARIO: 'smoke' },
    },
    
    // Load test: normal expected load
    load: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '2m', target: 50 },   // Ramp up
        { duration: '5m', target: 50 },   // Hold
        { duration: '2m', target: 100 },  // Scale up
        { duration: '5m', target: 100 },  // Hold
        { duration: '2m', target: 0 },    // Ramp down
      ],
      tags: { scenario: 'load' },
    },
    
    // Stress test: find breaking point
    stress: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '2m', target: 100 },
        { duration: '5m', target: 200 },
        { duration: '2m', target: 300 },
        { duration: '5m', target: 300 },
        { duration: '2m', target: 400 },
        { duration: '5m', target: 400 },
        { duration: '5m', target: 0 },
      ],
      tags: { scenario: 'stress' },
    },
    
    // Spike test: sudden traffic surge
    spike: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '30s', target: 10 },
        { duration: '30s', target: 200 },  // Spike!
        { duration: '30s', target: 10 },
        { duration: '1m', target: 0 },
      ],
      tags: { scenario: 'spike' },
    },
    
    // Soak test: extended period at moderate load
    soak: {
      executor: 'constant-vus',
      vus: 50,
      duration: '4h',
      tags: { scenario: 'soak' },
    },
  },
  
  thresholds: {
    // 95th percentile response time under 500ms
    'http_req_duration': ['p(95)<500', 'p(99)<1000'],
    // Less than 1% errors
    'errors': ['rate<0.01'],
    // Order creation specific
    'order_creation_duration': ['p(95)<800'],
    // Minimum throughput
    'http_reqs': ['rate>100'],
  },
};

const BASE_URL = __ENV.BASE_URL || 'http://localhost:3000';

// Authenticate and get token
function authenticate() {
  const user = users[Math.floor(Math.random() * users.length)];
  
  const response = http.post(
    `${BASE_URL}/oauth/token`,
    JSON.stringify({
      grant_type: 'password',
      username: user.email,
      password: user.password,
      client_id: 'test-client',
    }),
    { headers: { 'Content-Type': 'application/json' } }
  );
  
  check(response, { 'auth successful': r => r.status === 200 });
  
  return response.json('access_token');
}

export default function() {
  const token = authenticate();
  const headers = {
    Authorization: `Bearer ${token}`,
    'Content-Type': 'application/json',
    'X-Request-ID': `test-${Date.now()}-${Math.random().toString(36).slice(2)}`,
  };
  
  group('Browse Products', () => {
    // List products
    const listRes = http.get(
      `${BASE_URL}/api/products?limit=20&page=1`,
      { headers }
    );
    
    check(listRes, {
      'products list 200': r => r.status === 200,
      'products list has data': r => r.json('products').length > 0,
    });
    
    errorRate.add(listRes.status !== 200);
    
    // Get product detail
    const product = products[Math.floor(Math.random() * products.length)];
    const detailRes = http.get(
      `${BASE_URL}/api/products/${product.id}`,
      { headers }
    );
    
    check(detailRes, {
      'product detail 200': r => r.status === 200,
      'product detail has price': r => r.json('price') > 0,
    });
    
    sleep(0.5);
  });
  
  group('Create Order', () => {
    const product = products[Math.floor(Math.random() * products.length)];
    const orderPayload = {
      items: [
        {
          productId: product.id,
          quantity: Math.floor(Math.random() * 5) + 1,
        },
      ],
      shippingAddress: {
        street: '123 Test St',
        city: 'Bangkok',
        country: 'TH',
        postalCode: '10110',
      },
      paymentMethod: 'card',
    };
    
    const start = Date.now();
    const orderRes = http.post(
      `${BASE_URL}/api/orders`,
      JSON.stringify(orderPayload),
      { headers }
    );
    const duration = Date.now() - start;
    
    orderDuration.add(duration);
    
    const orderOk = check(orderRes, {
      'order created 201': r => r.status === 201,
      'order has id': r => r.json('id') !== undefined,
      'order status pending': r => r.json('status') === 'pending',
    });
    
    if (orderOk) {
      successfulOrders.add(1);
      
      // Get order detail
      const orderId = orderRes.json('id');
      const getOrderRes = http.get(
        `${BASE_URL}/api/orders/${orderId}`,
        { headers }
      );
      
      check(getOrderRes, {
        'get order 200': r => r.status === 200,
      });
    }
    
    errorRate.add(orderRes.status >= 400);
    
    sleep(1);
  });
}

// Lifecycle hooks
export function setup() {
  console.log(`Starting load test against ${BASE_URL}`);
  
  // Verify service is healthy
  const healthRes = http.get(`${BASE_URL}/health`);
  if (healthRes.status !== 200) {
    throw new Error(`Service not healthy: ${healthRes.status}`);
  }
  
  return { startTime: Date.now() };
}

export function teardown(data) {
  const duration = (Date.now() - data.startTime) / 1000;
  console.log(`Test completed in ${duration}s`);
}
```

---

## 4. Chaos Engineering with Chaos Toolkit

### 4.1 Chaos Experiments Definition

```json
// chaos/experiments/order-service-resilience.json
{
  "version": "1.0.0",
  "title": "Order Service survives product service failure",
  "description": "Test that order service gracefully handles product service unavailability",
  
  "configuration": {
    "base_url": {
      "type": "env",
      "key": "ORDER_SERVICE_URL",
      "default": "http://localhost:3000"
    },
    "namespace": {
      "type": "env",
      "key": "K8S_NAMESPACE",
      "default": "production"
    }
  },
  
  "steady-state-hypothesis": {
    "title": "Order service is healthy and responsive",
    "probes": [
      {
        "type": "probe",
        "name": "order-service-responds",
        "tolerance": 200,
        "provider": {
          "type": "http",
          "url": "${base_url}/health",
          "timeout": 5
        }
      },
      {
        "type": "probe",
        "name": "can-create-order",
        "tolerance": 201,
        "provider": {
          "type": "http",
          "method": "POST",
          "url": "${base_url}/api/orders",
          "headers": {
            "Content-Type": "application/json",
            "Authorization": "Bearer ${TEST_TOKEN}"
          },
          "arguments": {
            "body": {
              "items": [{"productId": "test-product-1", "quantity": 1}]
            }
          },
          "timeout": 10
        }
      }
    ]
  },
  
  "method": [
    {
      "type": "action",
      "name": "terminate-product-service-pods",
      "provider": {
        "type": "python",
        "module": "chaosk8s.pod.actions",
        "func": "terminate_pods",
        "arguments": {
          "label_selector": "app=product-service",
          "ns": "${namespace}",
          "rand": true,
          "count": 1
        }
      },
      "pauses": {
        "after": 5
      }
    }
  ],
  
  "rollbacks": [
    {
      "type": "action",
      "name": "wait-for-product-service-recovery",
      "provider": {
        "type": "python",
        "module": "chaosk8s.deployment.probes",
        "func": "deployment_available_and_healthy",
        "arguments": {
          "name": "product-service",
          "ns": "${namespace}",
          "timeout": 120
        }
      }
    }
  ]
}
```

### 4.2 Network Chaos TypeScript Tests

```typescript
// src/tests/chaos/resilience.test.ts
import { describe, it, expect, beforeAll, afterAll } from 'vitest';
import { GenericContainer, StartedTestContainer, Network } from 'testcontainers';
import axios from 'axios';

describe('Order Service Resilience Tests', () => {
  let network: any;
  let orderService: StartedTestContainer;
  let productService: StartedTestContainer;
  let toxiproxy: StartedTestContainer;
  
  beforeAll(async () => {
    network = await new Network().start();
    
    // Start Toxiproxy for network fault injection
    toxiproxy = await new GenericContainer('shopify/toxiproxy:2.7.0')
      .withNetwork(network)
      .withNetworkAliases('toxiproxy')
      .withExposedPorts(8474, 8475)
      .start();
    
    // Start product service
    productService = await new GenericContainer('product-service:test')
      .withNetwork(network)
      .withNetworkAliases('product-service')
      .withExposedPorts(3001)
      .start();
    
    // Create toxiproxy proxy for product service
    const toxiproxyClient = axios.create({
      baseURL: `http://${toxiproxy.getHost()}:${toxiproxy.getMappedPort(8474)}`,
    });
    
    await toxiproxyClient.post('/api/proxies', {
      name: 'product-service',
      listen: '0.0.0.0:8475',
      upstream: 'product-service:3001',
      enabled: true,
    });
    
    // Start order service configured to use toxiproxy
    orderService = await new GenericContainer('order-service:test')
      .withNetwork(network)
      .withExposedPorts(3000)
      .withEnvironment({
        PRODUCT_SERVICE_URL: 'http://toxiproxy:8475',
        CIRCUIT_BREAKER_THRESHOLD: '3',
        CIRCUIT_BREAKER_TIMEOUT: '5000',
      })
      .start();
  }, 120000);
  
  afterAll(async () => {
    await orderService?.stop();
    await productService?.stop();
    await toxiproxy?.stop();
  });
  
  it('should serve cached product data when product service is slow', async () => {
    const orderServiceUrl = `http://${orderService.getHost()}:${orderService.getMappedPort(3000)}`;
    const toxiproxyApiUrl = `http://${toxiproxy.getHost()}:${toxiproxy.getMappedPort(8474)}`;
    
    // First call to populate cache
    const initialResponse = await axios.get(
      `${orderServiceUrl}/api/products/123`,
      { timeout: 5000 }
    );
    expect(initialResponse.status).toBe(200);
    
    // Inject latency via toxiproxy
    await axios.post(
      `${toxiproxyApiUrl}/api/proxies/product-service/toxics`,
      {
        name: 'slow-network',
        type: 'latency',
        stream: 'downstream',
        attributes: { latency: 3000, jitter: 500 },
      }
    );
    
    try {
      // Should still respond quickly using cache
      const cachedResponse = await axios.get(
        `${orderServiceUrl}/api/products/123`,
        { timeout: 1000 }
      );
      
      expect(cachedResponse.status).toBe(200);
      expect(cachedResponse.headers['x-cache']).toBe('HIT');
    } finally {
      // Remove toxic
      await axios.delete(
        `${toxiproxyApiUrl}/api/proxies/product-service/toxics/slow-network`
      );
    }
  }, 30000);
  
  it('should open circuit breaker after repeated failures', async () => {
    const orderServiceUrl = `http://${orderService.getHost()}:${orderService.getMappedPort(3000)}`;
    const toxiproxyApiUrl = `http://${toxiproxy.getHost()}:${toxiproxy.getMappedPort(8474)}`;
    
    // Inject connection reset
    await axios.post(
      `${toxiproxyApiUrl}/api/proxies/product-service/toxics`,
      {
        name: 'connection-reset',
        type: 'reset_peer',
        stream: 'upstream',
        attributes: { timeout: 100 },
      }
    );
    
    try {
      const responses = [];
      
      // Make multiple requests to trigger circuit breaker
      for (let i = 0; i < 10; i++) {
        try {
          const res = await axios.post(
            `${orderServiceUrl}/api/orders`,
            { items: [{ productId: `prod-${i}`, quantity: 1 }] },
            { timeout: 2000, validateStatus: () => true }
          );
          responses.push(res.status);
        } catch {
          responses.push(503);
        }
        
        await new Promise(r => setTimeout(r, 200));
      }
      
      // After circuit opens, should get fast failure (circuit breaker response)
      const lastResponses = responses.slice(-3);
      expect(lastResponses.every(s => s === 503 || s === 201)).toBe(true);
      
      // Verify circuit breaker metrics
      const metrics = await axios.get(`${orderServiceUrl}/metrics`);
      expect(metrics.data).toContain('circuit_breaker_state{state="open"}');
    } finally {
      await axios.delete(
        `${toxiproxyApiUrl}/api/proxies/product-service/toxics/connection-reset`
      );
    }
  }, 60000);
});
```

---

## 5. Test Data Factories

```typescript
// src/tests/factories/index.ts
import { faker } from '@faker-js/faker';
import bcrypt from 'bcrypt';

type DeepPartial<T> = {
  [P in keyof T]?: T[P] extends object ? DeepPartial<T[P]> : T[P];
};

// Base factory class
abstract class Factory<T> {
  abstract build(overrides?: DeepPartial<T>): T;
  
  buildList(count: number, overrides?: DeepPartial<T>): T[] {
    return Array.from({ length: count }, () => this.build(overrides));
  }
  
  async create(overrides?: DeepPartial<T>): Promise<T> {
    const data = this.build(overrides);
    return this.persist(data);
  }
  
  async createList(count: number, overrides?: DeepPartial<T>): Promise<T[]> {
    return Promise.all(
      Array.from({ length: count }, () => this.create(overrides))
    );
  }
  
  protected abstract persist(data: T): Promise<T>;
}

// User factory
export class UserFactory extends Factory<User> {
  build(overrides: DeepPartial<User> = {}): User {
    const firstName = faker.person.firstName();
    const lastName = faker.person.lastName();
    
    return {
      id: faker.string.uuid(),
      email: faker.internet.email({ firstName, lastName }).toLowerCase(),
      passwordHash: '$2b$10$mockhashedpassword',
      firstName,
      lastName,
      role: 'user',
      emailVerified: true,
      createdAt: faker.date.past(),
      updatedAt: faker.date.recent(),
      ...overrides,
    };
  }
  
  protected async persist(data: User): Promise<User> {
    const [user] = await db('users').insert(data).returning('*');
    return user;
  }
  
  // Convenience builder methods
  asAdmin(overrides?: DeepPartial<User>): User {
    return this.build({ ...overrides, role: 'admin' });
  }
  
  unverified(overrides?: DeepPartial<User>): User {
    return this.build({ ...overrides, emailVerified: false });
  }
  
  async withPassword(password: string, overrides?: DeepPartial<User>): Promise<User> {
    const hash = await bcrypt.hash(password, 10);
    return this.create({ ...overrides, passwordHash: hash });
  }
}

// Product factory
export class ProductFactory extends Factory<Product> {
  build(overrides: DeepPartial<Product> = {}): Product {
    return {
      id: faker.string.uuid(),
      name: faker.commerce.productName(),
      description: faker.commerce.productDescription(),
      price: parseFloat(faker.commerce.price({ min: 1, max: 1000 })),
      currency: 'USD',
      stock: faker.number.int({ min: 0, max: 1000 }),
      sku: `${faker.string.alpha(3).toUpperCase()}-${faker.string.numeric(6)}`,
      categoryId: faker.string.uuid(),
      isActive: true,
      createdAt: faker.date.past(),
      updatedAt: faker.date.recent(),
      ...overrides,
    };
  }
  
  protected async persist(data: Product): Promise<Product> {
    const [product] = await db('products').insert(data).returning('*');
    return product;
  }
  
  outOfStock(overrides?: DeepPartial<Product>): Product {
    return this.build({ ...overrides, stock: 0 });
  }
  
  onSale(discountPercent = 20, overrides?: DeepPartial<Product>): Product {
    const base = this.build(overrides);
    return {
      ...base,
      price: parseFloat((base.price * (1 - discountPercent / 100)).toFixed(2)),
    };
  }
}

// Order factory with relationships
export class OrderFactory extends Factory<Order> {
  constructor(
    private readonly userFactory = new UserFactory(),
    private readonly productFactory = new ProductFactory()
  ) {
    super();
  }
  
  build(overrides: DeepPartial<Order> = {}): Order {
    return {
      id: faker.string.uuid(),
      userId: faker.string.uuid(),
      status: 'pending',
      items: [
        {
          id: faker.string.uuid(),
          productId: faker.string.uuid(),
          productName: faker.commerce.productName(),
          quantity: faker.number.int({ min: 1, max: 10 }),
          unitPrice: parseFloat(faker.commerce.price()),
          totalPrice: 0, // Computed
        },
      ],
      totalAmount: parseFloat(faker.commerce.price()),
      currency: 'USD',
      shippingAddress: {
        street: faker.location.streetAddress(),
        city: faker.location.city(),
        country: faker.location.countryCode(),
        postalCode: faker.location.zipCode(),
      },
      createdAt: faker.date.past(),
      updatedAt: faker.date.recent(),
      ...overrides,
    };
  }
  
  protected async persist(data: Order): Promise<Order> {
    const [order] = await db('orders').insert(data).returning('*');
    return order;
  }
  
  // Create order with real user and products
  async withRealRelationships(): Promise<Order & { user: User; products: Product[] }> {
    const user = await this.userFactory.create();
    const product = await this.productFactory.create();
    
    const order = await this.create({
      userId: user.id,
      items: [{
        productId: product.id,
        productName: product.name,
        quantity: 2,
        unitPrice: product.price,
        totalPrice: product.price * 2,
      }],
      totalAmount: product.price * 2,
    });
    
    return { ...order, user, products: [product] };
  }
}

// Singleton exports
export const userFactory = new UserFactory();
export const productFactory = new ProductFactory();
export const orderFactory = new OrderFactory();
```

---

## 6. Test Isolation Strategies

```typescript
// src/tests/helpers/test-isolation.ts
import { db } from '../../database';
import { Redis } from 'ioredis';

// Database transaction isolation
export class DatabaseIsolation {
  private trx: Knex.Transaction | null = null;
  
  async setup() {
    this.trx = await db.transaction();
    // Monkey-patch db to use transaction
    (db as any)._transactionContext = this.trx;
  }
  
  async teardown() {
    if (this.trx) {
      await this.trx.rollback();
      this.trx = null;
    }
  }
}

// Redis namespace isolation
export class RedisIsolation {
  private readonly prefix: string;
  
  constructor() {
    this.prefix = `test:${Date.now()}:${Math.random().toString(36).slice(2)}:`;
  }
  
  createIsolatedClient(redis: Redis): Redis {
    // Proxy Redis client with namespace prefix
    return new Proxy(redis, {
      get(target, prop) {
        const value = (target as any)[prop];
        
        if (typeof value === 'function' && ['get', 'set', 'del', 'setex', 'exists', 'hget', 'hset'].includes(prop as string)) {
          return function(key: string, ...args: any[]) {
            return value.call(target, `${this.prefix}${key}`, ...args);
          }.bind({ prefix: `test:${Date.now()}:` });
        }
        
        return value;
      }
    });
  }
  
  async cleanup(redis: Redis) {
    const keys = await redis.keys(`${this.prefix}*`);
    if (keys.length > 0) {
      await redis.del(...keys);
    }
  }
}

// Vitest setup/teardown helpers
export function withDatabaseIsolation() {
  const isolation = new DatabaseIsolation();
  
  beforeEach(async () => {
    await isolation.setup();
  });
  
  afterEach(async () => {
    await isolation.teardown();
  });
  
  return isolation;
}

// Global test setup
// vitest.config.ts
export default {
  test: {
    setupFiles: ['./src/tests/setup.ts'],
    globalSetup: ['./src/tests/global-setup.ts'],
    testTimeout: 30000,
    hookTimeout: 30000,
    coverage: {
      provider: 'v8',
      reporter: ['text', 'lcov', 'html'],
      exclude: [
        'node_modules/**',
        'src/tests/**',
        'src/**/*.d.ts',
        'src/migrations/**',
      ],
      thresholds: {
        global: {
          branches: 80,
          functions: 85,
          lines: 85,
          statements: 85,
        },
      },
    },
  },
};
```

---

## สรุป

ในบทนี้เราได้เรียนรู้การทดสอบ Microservices อย่างครบวงจร:

1. **TestContainers** — Integration tests กับ PostgreSQL, Redis, Kafka จริงๆ ใน Docker containers ให้ test ที่ reliable กว่า mocks

2. **Pact.js Contract Testing** — Consumer-side interaction definitions, Provider-side verification, state handlers สำหรับ test data setup

3. **k6 Performance Tests** — หลาย scenario (smoke/load/stress/spike/soak), custom metrics, thresholds, และ lifecycle hooks

4. **Chaos Engineering** — Toxiproxy สำหรับ network fault injection, circuit breaker testing, และ Chaos Toolkit experiments JSON

5. **Test Data Factories** — Factory pattern ด้วย Faker.js สำหรับ type-safe, composable test data generation

6. **Test Isolation** — Database transaction rollback และ Redis namespace isolation เพื่อ independent tests

Key takeaways:
- ใช้ TestContainers แทน mock สำหรับ infrastructure dependencies ใน integration tests
- Contract tests ช่วยให้ services สามารถ deploy independently โดยมั่นใจว่า API ไม่ break
- k6 scenarios ครบ lifecycle: smoke → load → stress → spike → soak
- Chaos tests ควรรัน regularly ใน staging environment
- Factory pattern ช่วยลด boilerplate และทำให้ test data consistent

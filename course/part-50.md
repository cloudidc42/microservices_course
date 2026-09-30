# Part 50: Microservices Testing ขั้นสูง

## บทนำ

การทดสอบ Microservices ที่ครอบคลุมต้องการหลายระดับ ตั้งแต่ Unit Tests, Integration Tests ด้วย TestContainers, Consumer-Driven Contract Testing ด้วย Pact.js, Performance Testing ด้วย k6, ไปจนถึง Chaos Engineering เพื่อให้ระบบมีความ resilient อย่างแท้จริง

---

## 1. โครงสร้างโปรเจกต์

```
microservices-testing/
├── services/
│   ├── user-service/
│   │   ├── src/
│   │   ├── test/
│   │   │   ├── unit/
│   │   │   ├── integration/
│   │   │   ├── contract/
│   │   │   │   ├── consumer/
│   │   │   │   └── provider/
│   │   │   └── e2e/
│   │   └── package.json
│   └── order-service/
│       ├── src/
│       └── test/
│           ├── unit/
│           ├── integration/
│           └── contract/
├── performance/
│   ├── k6/
│   │   ├── scenarios/
│   │   │   ├── load-test.js
│   │   │   ├── stress-test.js
│   │   │   ├── spike-test.js
│   │   │   └── soak-test.js
│   │   ├── helpers/
│   │   └── dashboards/
├── chaos/
│   ├── litmus/
│   │   └── experiments/
│   └── k8s-chaos/
├── test-data/
│   ├── factories/
│   ├── seeds/
│   └── fixtures/
└── docker-compose.test.yml
```

---

## 2. TestContainers สำหรับ Integration Tests

### 2.1 Setup TestContainers

```typescript
// services/user-service/test/integration/setup.ts
import {
  PostgreSqlContainer,
  StartedPostgreSqlContainer,
} from '@testcontainers/postgresql';
import { RedisContainer, StartedRedisContainer } from '@testcontainers/redis';
import { Network, StartedNetwork } from 'testcontainers';
import { DataSource } from 'typeorm';
import { createClient, RedisClientType } from 'redis';

export interface TestEnvironment {
  postgres: StartedPostgreSqlContainer;
  redis: StartedRedisContainer;
  network: StartedNetwork;
  dataSource: DataSource;
  redisClient: RedisClientType;
}

let testEnv: TestEnvironment | null = null;

export async function setupTestEnvironment(): Promise<TestEnvironment> {
  if (testEnv) return testEnv;

  console.log('Starting test containers...');

  // สร้าง Docker network สำหรับ test
  const network = await new Network().start();

  // Start PostgreSQL container
  const postgres = await new PostgreSqlContainer('postgres:16-alpine')
    .withNetwork(network)
    .withNetworkAliases('postgres-test')
    .withDatabase('test_db')
    .withUsername('test_user')
    .withPassword('test_password')
    .withCommand([
      'postgres',
      '-c', 'max_connections=200',
      '-c', 'shared_buffers=128MB',
      '-c', 'log_min_duration_statement=500',
    ])
    .withHealthCheck({
      test: ['CMD-SHELL', 'pg_isready -U test_user -d test_db'],
      interval: 10000,
      timeout: 5000,
      retries: 5,
    })
    .withStartupTimeout(120000)
    .start();

  // Start Redis container
  const redis = await new RedisContainer('redis:7-alpine')
    .withNetwork(network)
    .withNetworkAliases('redis-test')
    .withStartupTimeout(60000)
    .start();

  // สร้าง TypeORM DataSource
  const dataSource = new DataSource({
    type: 'postgres',
    host: postgres.getHost(),
    port: postgres.getMappedPort(5432),
    database: 'test_db',
    username: 'test_user',
    password: 'test_password',
    entities: [`${__dirname}/../../src/**/*.entity.ts`],
    migrations: [`${__dirname}/../../src/migrations/*.ts`],
    synchronize: false,
    logging: process.env.DB_LOGGING === 'true',
  });

  await dataSource.initialize();
  await dataSource.runMigrations();

  // สร้าง Redis client
  const redisClient = createClient({
    socket: {
      host: redis.getHost(),
      port: redis.getMappedPort(6379),
    },
  }) as RedisClientType;

  await redisClient.connect();

  testEnv = { postgres, redis, network, dataSource, redisClient };

  console.log('Test containers started successfully');
  return testEnv;
}

export async function teardownTestEnvironment(): Promise<void> {
  if (!testEnv) return;

  await testEnv.redisClient.quit();
  await testEnv.dataSource.destroy();
  await testEnv.postgres.stop();
  await testEnv.redis.stop();
  await testEnv.network.stop();

  testEnv = null;
  console.log('Test containers stopped');
}
```

### 2.2 Integration Test ตัวอย่าง

```typescript
// services/user-service/test/integration/user.service.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { TypeOrmModule } from '@nestjs/typeorm';
import { UserService } from '../../src/users/user.service';
import { User } from '../../src/users/entities/user.entity';
import { setupTestEnvironment, teardownTestEnvironment, TestEnvironment } from './setup';
import { UserFactory } from '../factories/user.factory';
import { ConflictException, NotFoundException } from '@nestjs/common';

describe('UserService Integration Tests', () => {
  let env: TestEnvironment;
  let userService: UserService;
  let module: TestingModule;
  let userFactory: UserFactory;

  beforeAll(async () => {
    env = await setupTestEnvironment();

    module = await Test.createTestingModule({
      imports: [
        TypeOrmModule.forRoot({
          type: 'postgres',
          host: env.postgres.getHost(),
          port: env.postgres.getMappedPort(5432),
          database: 'test_db',
          username: 'test_user',
          password: 'test_password',
          entities: [User],
          synchronize: false,
        }),
        TypeOrmModule.forFeature([User]),
      ],
      providers: [UserService],
    }).compile();

    userService = module.get<UserService>(UserService);
    userFactory = new UserFactory(env.dataSource);
  });

  afterAll(async () => {
    await module.close();
    await teardownTestEnvironment();
  });

  beforeEach(async () => {
    // ล้างข้อมูลก่อนแต่ละ test
    await env.dataSource.query('TRUNCATE TABLE users RESTART IDENTITY CASCADE');
  });

  describe('createUser', () => {
    it('should create a new user successfully', async () => {
      const dto = {
        email: 'test@example.com',
        username: 'testuser',
        password: 'SecurePass123!',
        firstName: 'Test',
        lastName: 'User',
      };

      const user = await userService.create(dto);

      expect(user).toBeDefined();
      expect(user.email).toBe(dto.email);
      expect(user.username).toBe(dto.username);
      expect(user.password).not.toBe(dto.password); // ต้อง hash แล้ว
      expect(user.id).toBeDefined();
      expect(user.createdAt).toBeInstanceOf(Date);
    });

    it('should throw ConflictException for duplicate email', async () => {
      await userFactory.create({ email: 'duplicate@example.com' });

      await expect(
        userService.create({
          email: 'duplicate@example.com',
          username: 'newuser',
          password: 'Password123!',
          firstName: 'New',
          lastName: 'User',
        }),
      ).rejects.toThrow(ConflictException);
    });

    it('should handle concurrent user creation (race condition)', async () => {
      const createUser = (suffix: string) =>
        userService.create({
          email: `user${suffix}@example.com`,
          username: `user${suffix}`,
          password: 'Password123!',
          firstName: 'User',
          lastName: suffix,
        });

      // สร้าง users พร้อมกัน
      const results = await Promise.allSettled(
        Array.from({ length: 10 }, (_, i) => createUser(String(i))),
      );

      const succeeded = results.filter((r) => r.status === 'fulfilled').length;
      expect(succeeded).toBe(10); // ทั้งหมดต้องสำเร็จ
    });
  });

  describe('findById', () => {
    it('should find existing user by ID', async () => {
      const created = await userFactory.create();

      const found = await userService.findById(created.id);

      expect(found.id).toBe(created.id);
      expect(found.email).toBe(created.email);
    });

    it('should throw NotFoundException for non-existing user', async () => {
      await expect(
        userService.findById('00000000-0000-0000-0000-000000000000'),
      ).rejects.toThrow(NotFoundException);
    });
  });

  describe('updateUser', () => {
    it('should update user profile', async () => {
      const user = await userFactory.create({ firstName: 'Original' });

      const updated = await userService.update(user.id, { firstName: 'Updated' });

      expect(updated.firstName).toBe('Updated');
      expect(updated.updatedAt.getTime()).toBeGreaterThan(user.updatedAt.getTime());
    });

    it('should not update password directly', async () => {
      const user = await userFactory.create();
      const originalPasswordHash = user.password;

      // ไม่ควรอนุญาตให้ update password โดยตรงผ่าน update method
      await userService.update(user.id, { firstName: 'New Name' });
      
      const updated = await userService.findById(user.id);
      expect(updated.password).toBe(originalPasswordHash);
    });
  });

  describe('searchUsers', () => {
    it('should search users by name with proper pagination', async () => {
      // สร้าง test data
      await userFactory.createMany(25, { firstName: 'SearchTest' });
      await userFactory.createMany(5, { firstName: 'OtherName' });

      const result = await userService.search({
        query: 'SearchTest',
        page: 1,
        limit: 10,
      });

      expect(result.data).toHaveLength(10);
      expect(result.total).toBe(25);
      expect(result.page).toBe(1);

      // ตรวจสอบว่า results ทั้งหมด match criteria
      result.data.forEach((user) => {
        expect(user.firstName).toBe('SearchTest');
      });
    });

    it('should prevent SQL injection in search', async () => {
      const maliciousQuery = "'; DROP TABLE users; --";
      
      // ไม่ควร throw และไม่ควรลบข้อมูล
      const result = await userService.search({ query: maliciousQuery, page: 1, limit: 10 });
      expect(result.data).toHaveLength(0);
      
      // ตรวจสอบว่า users table ยังอยู่
      const count = await env.dataSource.query('SELECT COUNT(*) FROM users');
      expect(parseInt(count[0].count)).toBeGreaterThanOrEqual(0);
    });
  });
});
```

---

## 3. Test Data Factories

```typescript
// test/factories/user.factory.ts
import { DataSource } from 'typeorm';
import { User } from '../../src/users/entities/user.entity';
import * as bcrypt from 'bcrypt';
import { faker } from '@faker-js/faker';

type PartialUser = Partial<Omit<User, 'id' | 'createdAt' | 'updatedAt'>>;

export class UserFactory {
  constructor(private readonly dataSource: DataSource) {}

  async create(overrides: PartialUser = {}): Promise<User> {
    const repo = this.dataSource.getRepository(User);

    const defaultData: PartialUser = {
      email: faker.internet.email().toLowerCase(),
      username: faker.internet.userName().toLowerCase().replace(/[^a-z0-9_-]/g, '_'),
      password: await bcrypt.hash('TestPassword123!', 10),
      firstName: faker.person.firstName(),
      lastName: faker.person.lastName(),
      status: 'active',
      role: 'user',
    };

    const user = repo.create({ ...defaultData, ...overrides });
    return repo.save(user);
  }

  async createMany(count: number, overrides: PartialUser = {}): Promise<User[]> {
    return Promise.all(
      Array.from({ length: count }, () => this.create(overrides)),
    );
  }

  async createWithRole(role: string): Promise<User> {
    return this.create({ role });
  }

  async createAdmin(): Promise<User> {
    return this.create({ role: 'admin', status: 'active' });
  }

  async createInactive(): Promise<User> {
    return this.create({ status: 'inactive' });
  }

  // สร้าง user ที่มี predictable data สำหรับ snapshot tests
  async createPredictable(seed: number): Promise<User> {
    faker.seed(seed);
    return this.create({
      email: `user${seed}@predictable.com`,
      username: `predictable_user_${seed}`,
    });
  }
}

// Order Factory
export class OrderFactory {
  constructor(private readonly dataSource: DataSource) {}

  async create(
    userId: string,
    overrides: Record<string, unknown> = {},
  ): Promise<any> {
    const defaultData = {
      userId,
      status: 'pending',
      total: faker.number.float({ min: 100, max: 10000, fractionDigits: 2 }),
      currency: 'THB',
      items: [
        {
          productId: faker.string.uuid(),
          name: faker.commerce.productName(),
          quantity: faker.number.int({ min: 1, max: 5 }),
          price: faker.number.float({ min: 100, max: 2000, fractionDigits: 2 }),
        },
      ],
      shippingAddress: {
        street: faker.location.streetAddress(),
        city: 'Bangkok',
        province: 'Bangkok',
        postalCode: '10110',
        country: 'TH',
      },
      ...overrides,
    };

    const repo = this.dataSource.getRepository('Order');
    const order = repo.create(defaultData);
    return repo.save(order);
  }
}
```

---

## 4. Consumer-Driven Contract Testing ด้วย Pact.js

### 4.1 Consumer Test (Order Service)

```typescript
// services/order-service/test/contract/consumer/user-service.consumer.spec.ts
import { Pact, MatchersV3 } from '@pact-foundation/pact';
import { resolve } from 'path';
import axios from 'axios';

const { like, eachLike, string, uuid, boolean, timestamp } = MatchersV3;

// สร้าง Pact สำหรับ Order Service (consumer) กับ User Service (provider)
const provider = new Pact({
  consumer: 'OrderService',
  provider: 'UserService',
  port: 1234,
  log: resolve(__dirname, '../logs', 'pact.log'),
  dir: resolve(__dirname, '../pacts'),
  logLevel: 'warn',
});

describe('Order Service -> User Service Contract', () => {
  beforeAll(() => provider.setup());
  afterAll(() => provider.finalize());
  afterEach(() => provider.verify());

  describe('GET /api/users/:id', () => {
    it('should get user by ID for order processing', async () => {
      // กำหนด interaction ที่คาดหวัง
      await provider.addInteraction({
        state: 'a user with ID user-123 exists',
        uponReceiving: 'a request to get user by ID for order',
        withRequest: {
          method: 'GET',
          path: '/api/users/user-123',
          headers: {
            Accept: 'application/json',
            Authorization: like('Bearer valid-token'),
          },
        },
        willRespondWith: {
          status: 200,
          headers: {
            'Content-Type': 'application/json',
          },
          body: {
            id: like('user-123'),
            email: like('user@example.com'),
            username: like('johndoe'),
            firstName: like('John'),
            lastName: like('Doe'),
            status: like('active'),
            shippingAddresses: eachLike({
              id: like('addr-123'),
              street: like('123 Main St'),
              city: like('Bangkok'),
              postalCode: like('10110'),
              country: like('TH'),
              isDefault: boolean(true),
            }),
          },
        },
      });

      // ทำ request จริง
      const response = await axios.get(
        `${provider.mockService.baseUrl}/api/users/user-123`,
        {
          headers: {
            Accept: 'application/json',
            Authorization: 'Bearer valid-token',
          },
        },
      );

      expect(response.status).toBe(200);
      expect(response.data.id).toBeDefined();
      expect(response.data.email).toBeDefined();
      expect(response.data.status).toBe('active');
    });

    it('should handle user not found', async () => {
      await provider.addInteraction({
        state: 'no user with ID nonexistent-user exists',
        uponReceiving: 'a request to get non-existing user',
        withRequest: {
          method: 'GET',
          path: '/api/users/nonexistent-user',
          headers: {
            Accept: 'application/json',
            Authorization: like('Bearer valid-token'),
          },
        },
        willRespondWith: {
          status: 404,
          headers: {
            'Content-Type': 'application/json',
          },
          body: {
            statusCode: like(404),
            message: like('User not found'),
          },
        },
      });

      try {
        await axios.get(
          `${provider.mockService.baseUrl}/api/users/nonexistent-user`,
          {
            headers: {
              Accept: 'application/json',
              Authorization: 'Bearer valid-token',
            },
          },
        );
        fail('Should have thrown 404 error');
      } catch (error: any) {
        expect(error.response.status).toBe(404);
      }
    });
  });

  describe('POST /api/users/:id/loyalty-points', () => {
    it('should add loyalty points after order completion', async () => {
      await provider.addInteraction({
        state: 'a user with ID user-123 exists with loyalty program',
        uponReceiving: 'a request to add loyalty points',
        withRequest: {
          method: 'POST',
          path: '/api/users/user-123/loyalty-points',
          headers: {
            'Content-Type': 'application/json',
            Authorization: like('Bearer valid-token'),
          },
          body: {
            points: like(100),
            reason: like('order_completed'),
            referenceId: like('order-456'),
          },
        },
        willRespondWith: {
          status: 200,
          body: {
            userId: like('user-123'),
            pointsAdded: like(100),
            totalPoints: like(500),
            tier: like('silver'),
          },
        },
      });

      const response = await axios.post(
        `${provider.mockService.baseUrl}/api/users/user-123/loyalty-points`,
        {
          points: 100,
          reason: 'order_completed',
          referenceId: 'order-456',
        },
        {
          headers: {
            'Content-Type': 'application/json',
            Authorization: 'Bearer valid-token',
          },
        },
      );

      expect(response.status).toBe(200);
      expect(response.data.pointsAdded).toBe(100);
    });
  });
});
```

### 4.2 Provider Test (User Service)

```typescript
// services/user-service/test/contract/provider/order-service.provider.spec.ts
import { Verifier, VerifierOptions } from '@pact-foundation/pact';
import { resolve } from 'path';
import { Test, TestingModule } from '@nestjs/testing';
import { INestApplication } from '@nestjs/common';
import { AppModule } from '../../../src/app.module';
import { setupTestEnvironment, teardownTestEnvironment } from '../../integration/setup';
import { UserFactory } from '../../factories/user.factory';

describe('User Service Provider Contract Tests', () => {
  let app: INestApplication;
  let testEnv: Awaited<ReturnType<typeof setupTestEnvironment>>;
  let userFactory: UserFactory;

  beforeAll(async () => {
    testEnv = await setupTestEnvironment();
    userFactory = new UserFactory(testEnv.dataSource);

    const moduleFixture: TestingModule = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = moduleFixture.createNestApplication();
    await app.listen(3001);
  });

  afterAll(async () => {
    await app.close();
    await teardownTestEnvironment();
  });

  it('should verify consumer contracts', async () => {
    const options: VerifierOptions = {
      providerBaseUrl: 'http://localhost:3001',
      
      // ดึง pacts จาก Pact Broker
      pactBrokerUrl: process.env.PACT_BROKER_URL || 'http://pact-broker:9292',
      pactBrokerToken: process.env.PACT_BROKER_TOKEN,
      
      // หรือ load จาก local file
      pactUrls: [
        resolve(__dirname, '../pacts/OrderService-UserService.json'),
      ],
      
      provider: 'UserService',
      providerVersion: process.env.SERVICE_VERSION || '1.0.0',
      publishVerificationResult: process.env.CI === 'true',
      
      // Provider states setup
      stateHandlers: {
        'a user with ID user-123 exists': async () => {
          // สร้าง user ที่มี ID user-123
          await userFactory.create({ id: 'user-123', email: 'user@example.com' });
        },
        
        'no user with ID nonexistent-user exists': async () => {
          // ลบ user ถ้ามีอยู่
          await testEnv.dataSource.query(
            "DELETE FROM users WHERE id = 'nonexistent-user'",
          );
        },
        
        'a user with ID user-123 exists with loyalty program': async () => {
          await userFactory.create({
            id: 'user-123',
            email: 'user@example.com',
            loyaltyPoints: 400,
            loyaltyTier: 'silver',
          });
        },
      },

      // Setup request verification
      requestFilter: (req, res, next) => {
        // เพิ่ม test token
        if (!req.headers.authorization) {
          req.headers.authorization = 'Bearer test-token';
        }
        next();
      },
    };

    const verifier = new Verifier(options);
    await verifier.verifyProvider();
  });
});
```

---

## 5. Performance Testing ด้วย k6

### 5.1 Load Test

```javascript
// performance/k6/scenarios/load-test.js
import http from 'k6/http';
import { check, sleep, group } from 'k6';
import { Rate, Counter, Trend, Gauge } from 'k6/metrics';
import { randomString, randomIntBetween } from 'https://jslib.k6.io/k6-utils/1.4.0/index.js';

// Custom metrics
const errorRate = new Rate('error_rate');
const orderCreationDuration = new Trend('order_creation_duration', true);
const activeOrders = new Gauge('active_orders');
const orderRevenue = new Counter('order_revenue');

const BASE_URL = __ENV.BASE_URL || 'https://staging.myapp.com';

// Test configuration
export const options = {
  scenarios: {
    // Scenario 1: Steady load
    steady_load: {
      executor: 'constant-arrival-rate',
      rate: 100,
      timeUnit: '1s',
      duration: '5m',
      preAllocatedVUs: 50,
      maxVUs: 200,
    },
    
    // Scenario 2: Ramp up
    ramp_up: {
      executor: 'ramping-arrival-rate',
      startRate: 10,
      timeUnit: '1s',
      preAllocatedVUs: 50,
      maxVUs: 500,
      stages: [
        { target: 50, duration: '2m' },
        { target: 100, duration: '3m' },
        { target: 200, duration: '5m' },
        { target: 100, duration: '2m' },
        { target: 0, duration: '1m' },
      ],
    },
  },
  
  thresholds: {
    'http_req_duration': ['p(95)<500', 'p(99)<1000'],
    'http_req_failed': ['rate<0.01'],
    'error_rate': ['rate<0.05'],
    'order_creation_duration': ['p(95)<2000'],
    'checks': ['rate>0.99'],
  },
};

// Setup: สร้าง test users ก่อน
export function setup() {
  const users = [];
  
  for (let i = 0; i < 100; i++) {
    const res = http.post(
      `${BASE_URL}/api/auth/register`,
      JSON.stringify({
        email: `loadtest_${randomString(8)}@test.com`,
        username: `loadtest_${randomString(8)}`,
        password: 'LoadTest123!',
        firstName: 'Load',
        lastName: 'Test',
        acceptTerms: true,
      }),
      { headers: { 'Content-Type': 'application/json' } },
    );
    
    if (res.status === 201) {
      const authRes = http.post(
        `${BASE_URL}/api/auth/login`,
        JSON.stringify({
          email: res.json('email'),
          password: 'LoadTest123!',
        }),
        { headers: { 'Content-Type': 'application/json' } },
      );
      
      if (authRes.status === 200) {
        users.push({
          email: res.json('email'),
          token: authRes.json('access_token'),
          userId: res.json('id'),
        });
      }
    }
  }
  
  return { users };
}

export default function (data) {
  const user = data.users[Math.floor(Math.random() * data.users.length)];
  const headers = {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${user.token}`,
  };

  group('Browse Products', () => {
    const res = http.get(`${BASE_URL}/api/products?page=1&limit=20`, { headers });
    
    check(res, {
      'products list status 200': (r) => r.status === 200,
      'products list has items': (r) => r.json('data') && r.json('data').length > 0,
    }) || errorRate.add(1);
    
    sleep(randomIntBetween(1, 3));
  });

  group('Create Order', () => {
    const orderStart = Date.now();
    
    const orderRes = http.post(
      `${BASE_URL}/api/orders`,
      JSON.stringify({
        items: [
          {
            productId: `product-${randomIntBetween(1, 100)}`,
            quantity: randomIntBetween(1, 3),
            price: randomIntBetween(100, 1000),
          },
        ],
        shippingAddress: {
          street: '123 Test Street',
          city: 'Bangkok',
          province: 'Bangkok',
          postalCode: '10110',
          country: 'TH',
        },
        paymentMethod: 'credit_card',
      }),
      { headers },
    );
    
    const orderDuration = Date.now() - orderStart;
    orderCreationDuration.add(orderDuration);
    
    const success = check(orderRes, {
      'order created status 201': (r) => r.status === 201,
      'order has ID': (r) => r.json('id') !== undefined,
      'order status pending': (r) => r.json('status') === 'pending',
    });
    
    if (!success) {
      errorRate.add(1);
    } else {
      const orderAmount = orderRes.json('total');
      orderRevenue.add(orderAmount);
      activeOrders.add(1);
    }
    
    sleep(randomIntBetween(2, 5));
  });

  group('Check Order Status', () => {
    const ordersRes = http.get(
      `${BASE_URL}/api/orders?userId=${user.userId}&page=1&limit=5`,
      { headers },
    );
    
    check(ordersRes, {
      'orders list status 200': (r) => r.status === 200,
      'response time < 500ms': (r) => r.timings.duration < 500,
    }) || errorRate.add(1);
    
    sleep(1);
  });
}

// Teardown: ล้างข้อมูล test
export function teardown(data) {
  // ลบ test users (optional)
  for (const user of data.users) {
    http.del(
      `${BASE_URL}/api/admin/test-users/${user.userId}`,
      null,
      {
        headers: {
          'Authorization': `Bearer ${__ENV.ADMIN_TOKEN}`,
        },
      },
    );
  }
}
```

### 5.2 Spike Test

```javascript
// performance/k6/scenarios/spike-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

const errorRate = new Rate('error_rate');
const BASE_URL = __ENV.BASE_URL || 'https://staging.myapp.com';

export const options = {
  stages: [
    { duration: '2m', target: 10 },    // Warm up
    { duration: '10s', target: 1000 }, // Spike!
    { duration: '5m', target: 1000 },  // Sustained spike
    { duration: '10s', target: 10 },   // Back to normal
    { duration: '2m', target: 0 },     // Cool down
  ],
  
  thresholds: {
    'http_req_duration': ['p(95)<3000'],
    'error_rate': ['rate<0.10'], // ยอมรับ error rate สูงขึ้นในช่วง spike
    'http_req_failed': ['rate<0.10'],
  },
};

export default function () {
  const res = http.get(`${BASE_URL}/api/products`, {
    headers: { Accept: 'application/json' },
  });
  
  const success = check(res, {
    'status is 200 or 503': (r) => r.status === 200 || r.status === 503,
    'response time < 3s': (r) => r.timings.duration < 3000,
  });
  
  if (!success || res.status >= 500) {
    errorRate.add(1);
  }
  
  sleep(0.1);
}
```

### 5.3 Soak Test

```javascript
// performance/k6/scenarios/soak-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Trend } from 'k6/metrics';

const memoryLeak = new Trend('response_size_bytes', true);
const BASE_URL = __ENV.BASE_URL || 'https://staging.myapp.com';

export const options = {
  stages: [
    { duration: '5m', target: 50 },   // Ramp up
    { duration: '8h', target: 50 },   // Soak (8 hours)
    { duration: '5m', target: 0 },    // Ramp down
  ],
  
  thresholds: {
    'http_req_duration': ['p(95)<1000'],
    'http_req_failed': ['rate<0.01'],
    // ตรวจสอบว่า response size ไม่เพิ่มขึ้นเรื่อยๆ (memory leak)
    'response_size_bytes': ['p(90)<10000'],
  },
};

export default function () {
  const res = http.get(`${BASE_URL}/api/health`, {
    headers: { Accept: 'application/json' },
  });
  
  check(res, {
    'health check passed': (r) => r.status === 200,
    'no memory growth': (r) => {
      const bodySize = r.body?.length || 0;
      memoryLeak.add(bodySize);
      return bodySize < 10000;
    },
  });
  
  sleep(1);
}
```

---

## 6. Chaos Engineering ด้วย Litmus

### 6.1 Network Latency Experiment

```yaml
# chaos/litmus/experiments/network-latency.yaml
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: network-latency-experiment
  namespace: microservices-staging
spec:
  appinfo:
    appns: microservices-staging
    applabel: app=user-service
    appkind: deployment
  
  annotationCheck: 'false'
  
  chaosServiceAccount: litmus-admin
  
  experiments:
    - name: pod-network-latency
      spec:
        components:
          env:
            # เพิ่ม latency 2 seconds
            - name: NETWORK_LATENCY
              value: '2000'
            # ระยะเวลาทดสอบ (5 minutes)
            - name: TOTAL_CHAOS_DURATION
              value: '300'
            # เป้าหมาย containers
            - name: TARGET_CONTAINER
              value: user-service
            # ทดสอบกับ 50% ของ pods
            - name: PODS_AFFECTED_PERC
              value: '50'
            # Network interface
            - name: NETWORK_INTERFACE
              value: eth0
            # Jitter (±500ms)
            - name: JITTER
              value: '500'
  
  # Steady state hypothesis
  steadyStateHypothesis:
    title: "Service should respond within 10s even with latency"
    probes:
      - name: "check-api-response"
        type: "httpProbe"
        mode: "Continuous"
        runProperties:
          probeTimeout: 10
          retry: 3
          interval: 5
        httpProbe/inputs:
          url: "https://staging.myapp.com/api/health"
          insecureSkipVerify: false
          responseTimeout: 10000
          method:
            get:
              criteria: ==
              responseCode: "200"
```

### 6.2 Pod Kill Experiment

```yaml
# chaos/litmus/experiments/pod-kill.yaml
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: pod-kill-experiment
  namespace: microservices-staging
spec:
  appinfo:
    appns: microservices-staging
    applabel: app=order-service
    appkind: deployment
  
  chaosServiceAccount: litmus-admin
  
  experiments:
    - name: pod-delete
      spec:
        components:
          env:
            - name: TOTAL_CHAOS_DURATION
              value: '300'
            # ลบ pods ทุก 30 วินาที
            - name: CHAOS_INTERVAL
              value: '30'
            # ลบทีละ 1 pod
            - name: PODS_AFFECTED_PERC
              value: '33'
            # รอให้ pod เป็น running ก่อน chaos
            - name: FORCE
              value: 'false'
  
  steadyStateHypothesis:
    title: "Service maintains at least 2 replicas and responds"
    probes:
      - name: "check-min-replicas"
        type: "k8sProbe"
        mode: "Continuous"
        k8sProbe/inputs:
          group: "apps"
          version: "v1"
          resource: "deployments"
          namespace: "microservices-staging"
          fieldSelector: "metadata.name=order-service"
          labelSelector: "app=order-service"
          operation: "present"
        runProperties:
          probeTimeout: 5
          retry: 3
          interval: 10
      
      - name: "check-order-api"
        type: "httpProbe"
        mode: "Continuous"
        httpProbe/inputs:
          url: "https://staging.myapp.com/api/orders/health"
          method:
            get:
              criteria: ==
              responseCode: "200"
        runProperties:
          probeTimeout: 5
          retry: 3
          interval: 10
```

### 6.3 CPU Stress Experiment

```yaml
# chaos/litmus/experiments/cpu-stress.yaml
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: cpu-stress-experiment
  namespace: microservices-staging
spec:
  appinfo:
    appns: microservices-staging
    applabel: app=payment-service
    appkind: deployment
  
  chaosServiceAccount: litmus-admin
  
  experiments:
    - name: pod-cpu-hog
      spec:
        components:
          env:
            - name: TOTAL_CHAOS_DURATION
              value: '300'
            - name: CPU_CORES
              value: '2'
            # ใช้ CPU 80%
            - name: CPU_LOAD
              value: '80'
            - name: PODS_AFFECTED_PERC
              value: '50'
  
  steadyStateHypothesis:
    title: "Payment service handles CPU stress gracefully"
    probes:
      - name: "check-payment-health"
        type: "httpProbe"
        mode: "Continuous"
        httpProbe/inputs:
          url: "https://staging.myapp.com/api/payments/health"
          responseTimeout: 5000
          method:
            get:
              criteria: ==
              responseCode: "200"
        runProperties:
          probeTimeout: 5
          retry: 3
          interval: 15
```

---

## 7. API Contract Testing

```typescript
// services/user-service/test/contract/api-schema.spec.ts
import Ajv from 'ajv';
import addFormats from 'ajv-formats';
import * as request from 'supertest';
import { Test, TestingModule } from '@nestjs/testing';
import { INestApplication } from '@nestjs/common';
import { AppModule } from '../../src/app.module';

const ajv = new Ajv({ allErrors: true, strict: false });
addFormats(ajv);

// OpenAPI Schema สำหรับ User
const userSchema = {
  type: 'object',
  required: ['id', 'email', 'username', 'firstName', 'lastName', 'status', 'createdAt'],
  properties: {
    id: { type: 'string', format: 'uuid' },
    email: { type: 'string', format: 'email' },
    username: { type: 'string', minLength: 3, maxLength: 50 },
    firstName: { type: 'string', minLength: 1, maxLength: 100 },
    lastName: { type: 'string', minLength: 1, maxLength: 100 },
    status: { type: 'string', enum: ['active', 'inactive', 'suspended'] },
    role: { type: 'string', enum: ['user', 'admin', 'moderator'] },
    createdAt: { type: 'string', format: 'date-time' },
    updatedAt: { type: 'string', format: 'date-time' },
  },
  additionalProperties: false,
};

const paginatedUsersSchema = {
  type: 'object',
  required: ['data', 'total', 'page', 'pageSize'],
  properties: {
    data: {
      type: 'array',
      items: userSchema,
    },
    total: { type: 'integer', minimum: 0 },
    page: { type: 'integer', minimum: 1 },
    pageSize: { type: 'integer', minimum: 1 },
    hasNextPage: { type: 'boolean' },
    hasPreviousPage: { type: 'boolean' },
  },
};

describe('User API Contract Tests', () => {
  let app: INestApplication;
  let authToken: string;

  beforeAll(async () => {
    const module: TestingModule = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = module.createNestApplication();
    await app.init();

    // ล็อกอินเพื่อรับ token
    const loginRes = await request(app.getHttpServer())
      .post('/api/auth/login')
      .send({ email: 'admin@test.com', password: 'AdminPass123!' });

    authToken = loginRes.body.access_token;
  });

  afterAll(async () => {
    await app.close();
  });

  it('GET /api/users should return paginated users matching schema', async () => {
    const response = await request(app.getHttpServer())
      .get('/api/users')
      .set('Authorization', `Bearer ${authToken}`)
      .expect(200);

    const validate = ajv.compile(paginatedUsersSchema);
    const valid = validate(response.body);

    if (!valid) {
      console.error('Schema validation errors:', validate.errors);
    }

    expect(valid).toBe(true);
  });

  it('GET /api/users/:id should return user matching schema', async () => {
    // สร้าง user ก่อน
    const createRes = await request(app.getHttpServer())
      .post('/api/users')
      .set('Authorization', `Bearer ${authToken}`)
      .send({
        email: 'schema-test@example.com',
        username: 'schematest',
        password: 'Test123!',
        firstName: 'Schema',
        lastName: 'Test',
        acceptTerms: true,
      })
      .expect(201);

    const userId = createRes.body.id;

    const response = await request(app.getHttpServer())
      .get(`/api/users/${userId}`)
      .set('Authorization', `Bearer ${authToken}`)
      .expect(200);

    const validate = ajv.compile(userSchema);
    const valid = validate(response.body);

    if (!valid) {
      console.error('Schema validation errors:', validate.errors);
    }

    expect(valid).toBe(true);
  });

  it('should maintain backward compatibility', async () => {
    const response = await request(app.getHttpServer())
      .get('/api/users')
      .set('Authorization', `Bearer ${authToken}`)
      .expect(200);

    // ตรวจสอบว่า required fields ยังมีอยู่
    const user = response.body.data[0];
    if (user) {
      expect(user).toHaveProperty('id');
      expect(user).toHaveProperty('email');
      expect(user).toHaveProperty('username');
      expect(user).toHaveProperty('createdAt');
    }
  });
});
```

---

## 8. Test Data Management

```typescript
// test-data/seeds/production-like-seed.ts
import { DataSource } from 'typeorm';
import { faker } from '@faker-js/faker';

export async function seedProductionLikeData(dataSource: DataSource): Promise<void> {
  console.log('Seeding production-like data...');

  await dataSource.transaction(async (manager) => {
    // สร้าง 1000 users
    const users = Array.from({ length: 1000 }, () => ({
      id: faker.string.uuid(),
      email: faker.internet.email().toLowerCase(),
      username: faker.internet.userName().toLowerCase().slice(0, 50),
      password: '$2b$10$hashedpassword', // pre-hashed
      firstName: faker.person.firstName(),
      lastName: faker.person.lastName(),
      status: faker.helpers.arrayElement(['active', 'active', 'active', 'inactive']),
      role: faker.helpers.arrayElement(['user', 'user', 'user', 'moderator', 'admin']),
      createdAt: faker.date.between({
        from: new Date('2023-01-01'),
        to: new Date(),
      }),
    }));

    await manager.query(
      `INSERT INTO users (id, email, username, password, first_name, last_name, status, role, created_at)
       VALUES ${users.map(() => '(?, ?, ?, ?, ?, ?, ?, ?, ?)').join(', ')}
       ON CONFLICT (email) DO NOTHING`,
      users.flatMap((u) => [
        u.id, u.email, u.username, u.password,
        u.firstName, u.lastName, u.status, u.role, u.createdAt,
      ]),
    );

    // สร้าง orders
    const orders = users.slice(0, 800).flatMap((user) =>
      Array.from(
        { length: faker.number.int({ min: 0, max: 20 }) },
        () => ({
          id: faker.string.uuid(),
          userId: user.id,
          status: faker.helpers.arrayElement([
            'pending', 'processing', 'completed', 'cancelled',
          ]),
          total: faker.number.float({ min: 100, max: 50000, fractionDigits: 2 }),
          currency: 'THB',
          createdAt: faker.date.between({
            from: new Date('2023-01-01'),
            to: new Date(),
          }),
        }),
      ),
    );

    // Batch insert
    const BATCH_SIZE = 500;
    for (let i = 0; i < orders.length; i += BATCH_SIZE) {
      const batch = orders.slice(i, i + BATCH_SIZE);
      await manager.query(
        `INSERT INTO orders (id, user_id, status, total, currency, created_at)
         VALUES ${batch.map(() => '(?, ?, ?, ?, ?, ?)').join(', ')}`,
        batch.flatMap((o) => [
          o.id, o.userId, o.status, o.total, o.currency, o.createdAt,
        ]),
      );
    }
  });

  console.log('Seeding completed');
}

// Snapshot test utility
export async function captureDataSnapshot(
  dataSource: DataSource,
  tables: string[],
): Promise<Record<string, unknown[]>> {
  const snapshot: Record<string, unknown[]> = {};

  for (const table of tables) {
    const rows = await dataSource.query(
      `SELECT * FROM ${table} ORDER BY id LIMIT 100`,
    );
    snapshot[table] = rows;
  }

  return snapshot;
}

export async function assertDataUnchanged(
  dataSource: DataSource,
  snapshot: Record<string, unknown[]>,
): Promise<void> {
  for (const [table, expectedRows] of Object.entries(snapshot)) {
    const currentRows = await dataSource.query(
      `SELECT * FROM ${table} ORDER BY id LIMIT 100`,
    );

    expect(currentRows).toEqual(expectedRows);
  }
}
```

---

## 9. CI/CD Test Pipeline

```yaml
# .github/workflows/testing.yml
name: Full Test Suite

on:
  pull_request:
    branches: [main, develop]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        service: [user-service, order-service, payment-service]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
          cache-dependency-path: services/${{ matrix.service }}/package-lock.json
      - run: npm ci
        working-directory: services/${{ matrix.service }}
      - run: npm run test:unit -- --coverage
        working-directory: services/${{ matrix.service }}
      - uses: codecov/codecov-action@v4
        with:
          file: services/${{ matrix.service }}/coverage/lcov.info
          flags: ${{ matrix.service }}-unit

  integration-tests:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        service: [user-service, order-service, payment-service]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
          cache-dependency-path: services/${{ matrix.service }}/package-lock.json
      - run: npm ci
        working-directory: services/${{ matrix.service }}
      - name: Run integration tests with TestContainers
        run: npm run test:integration
        working-directory: services/${{ matrix.service }}
        env:
          TESTCONTAINERS_RYUK_DISABLED: true
          DOCKER_HOST: unix:///var/run/docker.sock

  contract-tests:
    runs-on: ubuntu-latest
    needs: [unit-tests, integration-tests]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - name: Run consumer contract tests
        run: |
          cd services/order-service
          npm ci
          npm run test:contract:consumer
      - name: Publish pacts to broker
        run: |
          npx pact-broker publish \
            ./pacts \
            --broker-base-url ${{ secrets.PACT_BROKER_URL }} \
            --broker-token ${{ secrets.PACT_BROKER_TOKEN }} \
            --consumer-app-version ${{ github.sha }} \
            --branch ${{ github.ref_name }}
      - name: Verify provider contracts
        run: |
          cd services/user-service
          npm ci
          npm run test:contract:provider
        env:
          PACT_BROKER_URL: ${{ secrets.PACT_BROKER_URL }}
          PACT_BROKER_TOKEN: ${{ secrets.PACT_BROKER_TOKEN }}

  performance-tests:
    runs-on: ubuntu-latest
    needs: [contract-tests]
    if: github.ref == 'refs/heads/main' || github.base_ref == 'main'
    steps:
      - uses: actions/checkout@v4
      - name: Run k6 load test
        uses: grafana/k6-action@v0.3.1
        with:
          filename: performance/k6/scenarios/load-test.js
          flags: --out json=k6-results.json
        env:
          BASE_URL: ${{ secrets.STAGING_URL }}
          K6_CLOUD_TOKEN: ${{ secrets.K6_CLOUD_TOKEN }}
      - name: Check performance thresholds
        run: |
          # ตรวจสอบว่า p95 < 500ms
          P95=$(cat k6-results.json | jq '[.metrics.http_req_duration.values.p(95)] | add')
          if (( $(echo "$P95 > 500" | bc -l) )); then
            echo "❌ P95 latency exceeded threshold: ${P95}ms > 500ms"
            exit 1
          fi
          echo "✅ P95 latency within threshold: ${P95}ms"
```

---

## สรุป

| หัวข้อ | เทคโนโลยี | วัตถุประสงค์ |
|--------|-----------|-------------|
| TestContainers | @testcontainers/postgresql, redis | Integration tests ที่ใช้ real databases ไม่ใช่ mocks |
| Test Factories | @faker-js/faker + TypeORM | สร้าง test data ที่ realistic และ reproducible |
| Contract Testing | Pact.js (consumer + provider) | ตรวจสอบ API contracts ระหว่าง services |
| API Schema Testing | AJV + OpenAPI schema | ตรวจสอบ response format ถูกต้องตาม spec |
| Load Testing | k6 scenarios | ทดสอบ performance ภายใต้ load ปกติ |
| Spike Testing | k6 ramping | ทดสอบการรับมือกับ traffic ที่พุ่งสูงกะทันหัน |
| Soak Testing | k6 long duration | ตรวจหา memory leaks และ degradation เมื่อเวลาผ่านไป |
| Chaos Engineering | Litmus experiments | ทดสอบ resilience เมื่อ components fail |
| Test Data Management | Snapshots + Seeds | จัดการ test data อย่างเป็นระบบ |
| CI/CD Integration | GitHub Actions pipeline | รัน tests อัตโนมัติใน pipeline |

> **Best Practice**: ใช้ TestContainers แทน H2/SQLite สำหรับ integration tests เพื่อให้ behavior ตรงกับ production database ให้มากที่สุด และรัน contract tests ก่อน deploy ทุกครั้ง

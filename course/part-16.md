# Part 16: Testing Strategies for Microservices

## บทนำ

การ test ระบบ Microservices มีความซับซ้อนมากกว่า Monolith เพราะต้องทดสอบทั้ง individual services และ interactions ระหว่าง services

---

## 16.1 Testing Pyramid for Microservices

```
                        /\
                       /  \
                      / E2E \         ← น้อย, ช้า, แต่ครอบคลุม
                     /--------\
                    / Contract \      ← ตรวจสอบ API contracts
                   /------------\
                  / Integration  \    ← ทดสอบ service + dependencies
                 /----------------\
                /   Unit Tests     \  ← เยอะ, เร็ว, isolate
               /--------------------\
```

### Strategy ที่แนะนำ

| Layer | เปอร์เซ็นต์ | เหตุผล |
|-------|------------|--------|
| Unit Tests | 60-70% | เร็ว, isolate, pinpoints bugs |
| Integration Tests | 20-30% | ทดสอบ real dependencies |
| Contract Tests | 5-10% | verify service interfaces |
| E2E Tests | 1-5% | critical paths only |

---

## 16.2 Unit Testing

### User Service - Unit Tests

```javascript
// user-service/tests/unit/services/authService.test.js

const bcrypt = require('bcryptjs');
const AuthService = require('../../../src/services/authService');
const tokenService = require('../../../src/services/tokenService');
const { ConflictError, UnauthorizedError } = require('../../../src/errors/AppError');

// Mock dependencies
jest.mock('../../../src/services/tokenService');
jest.mock('bcryptjs');

describe('AuthService', () => {
  let authService;
  let mockUserRepository;
  let mockEmailService;

  beforeEach(() => {
    jest.clearAllMocks();

    mockUserRepository = {
      findByEmail: jest.fn(),
      findById: jest.fn(),
      create: jest.fn(),
      updateLastLogin: jest.fn(),
      incrementLoginAttempts: jest.fn(),
    };

    mockEmailService = {
      sendWelcomeEmail: jest.fn().mockResolvedValue(),
      sendVerification: jest.fn().mockResolvedValue(),
    };

    authService = new AuthService({
      userRepository: mockUserRepository,
      emailService: mockEmailService,
    });

    // Default mock implementations
    tokenService.generateAccessToken.mockReturnValue('mock-access-token');
    tokenService.generateRefreshToken.mockReturnValue('mock-refresh-token');
    tokenService.saveRefreshToken.mockResolvedValue('token-hash');
  });

  describe('register', () => {
    const validUserData = {
      email: 'new@example.com',
      password: 'SecureP@ss123',
      firstName: 'John',
      lastName: 'Doe',
    };

    it('should create a new user successfully', async () => {
      mockUserRepository.findByEmail.mockResolvedValue(null);
      mockUserRepository.create.mockResolvedValue({
        id: 'user-123',
        email: 'new@example.com',
        firstName: 'John',
        lastName: 'Doe',
        passwordHash: 'hashed-password',
        status: 'pending_verification',
      });
      bcrypt.hash.mockResolvedValue('hashed-password');

      const result = await authService.register(validUserData);

      expect(mockUserRepository.findByEmail).toHaveBeenCalledWith('new@example.com');
      expect(bcrypt.hash).toHaveBeenCalledWith('SecureP@ss123', 12);
      expect(mockUserRepository.create).toHaveBeenCalledWith(
        expect.objectContaining({
          email: 'new@example.com',
          passwordHash: 'hashed-password',
          firstName: 'John',
          lastName: 'Doe',
        })
      );
      expect(result.user.passwordHash).toBeUndefined();
    });

    it('should throw ConflictError when email already exists', async () => {
      mockUserRepository.findByEmail.mockResolvedValue({ id: 'existing-user' });

      await expect(authService.register(validUserData))
        .rejects
        .toThrow(ConflictError);

      expect(mockUserRepository.create).not.toHaveBeenCalled();
    });

    it('should send verification email after registration', async () => {
      mockUserRepository.findByEmail.mockResolvedValue(null);
      mockUserRepository.create.mockResolvedValue({
        id: 'user-123',
        email: 'new@example.com',
        passwordHash: 'hash',
      });
      bcrypt.hash.mockResolvedValue('hash');

      await authService.register(validUserData);

      expect(mockEmailService.sendVerification).toHaveBeenCalledWith(
        'new@example.com',
        expect.any(String)
      );
    });
  });

  describe('login', () => {
    const activeUser = {
      id: 'user-123',
      email: 'user@example.com',
      passwordHash: 'correct-hash',
      role: 'user',
      status: 'active',
    };

    it('should return tokens on successful login', async () => {
      mockUserRepository.findByEmail.mockResolvedValue(activeUser);
      bcrypt.compare.mockResolvedValue(true);

      const result = await authService.login({
        email: 'user@example.com',
        password: 'correct-password',
      });

      expect(result.accessToken).toBe('mock-access-token');
      expect(result.refreshToken).toBe('mock-refresh-token');
      expect(result.user.passwordHash).toBeUndefined();
      expect(mockUserRepository.updateLastLogin).toHaveBeenCalledWith('user-123');
    });

    it('should throw UnauthorizedError for wrong password', async () => {
      mockUserRepository.findByEmail.mockResolvedValue(activeUser);
      bcrypt.compare.mockResolvedValue(false);

      await expect(
        authService.login({ email: 'user@example.com', password: 'wrong' })
      ).rejects.toThrow();

      expect(mockUserRepository.incrementLoginAttempts).toHaveBeenCalledWith('user-123');
    });

    it('should throw for non-existent user (same error to prevent enumeration)', async () => {
      mockUserRepository.findByEmail.mockResolvedValue(null);

      await expect(
        authService.login({ email: 'nonexistent@example.com', password: 'any' })
      ).rejects.toThrow();
    });

    it('should throw for suspended account', async () => {
      mockUserRepository.findByEmail.mockResolvedValue({
        ...activeUser,
        status: 'suspended',
      });

      const error = await authService.login({
        email: 'user@example.com',
        password: 'password',
      }).catch(e => e);

      expect(error.code).toBe('ACCOUNT_SUSPENDED');
    });
  });
});
```

### Order Repository - Unit Tests

```javascript
// order-service/tests/unit/repositories/orderRepository.test.js

const OrderRepository = require('../../../src/repositories/orderRepository');
const { NotFoundError } = require('../../../src/errors/AppError');

// Mock database pool
const mockPool = {
  query: jest.fn(),
  connect: jest.fn(),
};

jest.mock('../../../src/database', () => ({
  pool: mockPool,
}));

describe('OrderRepository', () => {
  let repo;

  beforeEach(() => {
    jest.clearAllMocks();
    repo = new OrderRepository();
  });

  describe('findById', () => {
    it('should return order when found', async () => {
      const mockOrder = {
        id: 'order-1',
        user_id: 'user-1',
        status: 'pending',
        total: 200.00,
        items: [{ productId: 'p1', quantity: 2, price: 100 }],
      };

      mockPool.query.mockResolvedValue({ rows: [mockOrder] });

      const result = await repo.findById('order-1');

      expect(result).toEqual(mockOrder);
      expect(mockPool.query).toHaveBeenCalledWith(
        expect.stringContaining('WHERE id = $1'),
        ['order-1']
      );
    });

    it('should return null when not found', async () => {
      mockPool.query.mockResolvedValue({ rows: [] });

      const result = await repo.findById('non-existent');
      expect(result).toBeNull();
    });
  });

  describe('create', () => {
    it('should create order and return it', async () => {
      const orderData = {
        userId: 'user-1',
        items: [{ productId: 'p1', quantity: 2, price: 100 }],
        total: 200,
        shippingAddress: { street: '123 Main St', city: 'Bangkok' },
      };

      const createdOrder = { id: 'order-new', ...orderData, status: 'pending' };
      mockPool.query.mockResolvedValue({ rows: [createdOrder] });

      const result = await repo.create(orderData);

      expect(result).toEqual(createdOrder);
      expect(mockPool.query).toHaveBeenCalledWith(
        expect.stringContaining('INSERT INTO orders'),
        expect.arrayContaining(['user-1', 200])
      );
    });
  });

  describe('updateStatus', () => {
    it('should update order status', async () => {
      const updatedOrder = { id: 'order-1', status: 'confirmed' };
      mockPool.query.mockResolvedValue({ rows: [updatedOrder] });

      const result = await repo.updateStatus('order-1', 'confirmed');
      expect(result.status).toBe('confirmed');
    });

    it('should return null when order not found', async () => {
      mockPool.query.mockResolvedValue({ rows: [] });

      const result = await repo.updateStatus('non-existent', 'confirmed');
      expect(result).toBeNull();
    });
  });
});
```

---

## 16.3 Integration Testing

```javascript
// user-service/tests/integration/userService.test.js

const { Pool } = require('pg');
const request = require('supertest');
const app = require('../../src/app');

// Use test database
const pool = new Pool({
  connectionString: process.env.TEST_DATABASE_URL || 'postgresql://test:test@localhost:5433/test_users',
});

describe('User Service Integration Tests', () => {
  // Setup: clean database before each test
  beforeEach(async () => {
    await pool.query(`
      DELETE FROM users WHERE email LIKE 'test-%@example.com';
      DELETE FROM refresh_tokens WHERE user_id IN (
        SELECT id FROM users WHERE email LIKE 'test-%@example.com'
      );
    `);
  });

  afterAll(async () => {
    await pool.end();
  });

  describe('User CRUD', () => {
    it('should create, read, update, delete a user', async () => {
      // Create
      const createRes = await request(app)
        .post('/api/v1/auth/register')
        .send({
          email: 'test-crud@example.com',
          password: 'SecureP@ss123',
          firstName: 'Test',
          lastName: 'User',
        })
        .expect(201);

      const userId = createRes.body.data.user.id;
      expect(userId).toBeDefined();

      // Activate user (simulate email verification)
      await pool.query(
        'UPDATE users SET status = $1, email_verified_at = NOW() WHERE id = $2',
        ['active', userId]
      );

      // Login
      const loginRes = await request(app)
        .post('/api/v1/auth/login')
        .send({ email: 'test-crud@example.com', password: 'SecureP@ss123' })
        .expect(200);

      const token = loginRes.body.data.accessToken;

      // Read own profile
      const profileRes = await request(app)
        .get('/api/v1/users/me')
        .set('Authorization', `Bearer ${token}`)
        .expect(200);

      expect(profileRes.body.data.email).toBe('test-crud@example.com');

      // Update profile
      await request(app)
        .put('/api/v1/users/me')
        .set('Authorization', `Bearer ${token}`)
        .send({ firstName: 'Updated', lastName: 'Name' })
        .expect(200);

      // Verify update
      const { rows } = await pool.query(
        'SELECT first_name FROM users WHERE id = $1',
        [userId]
      );
      expect(rows[0].first_name).toBe('Updated');
    });
  });

  describe('Token Management', () => {
    let userId, accessToken, refreshToken;

    beforeEach(async () => {
      // Setup: create and activate user
      const res = await request(app)
        .post('/api/v1/auth/register')
        .send({
          email: 'test-token@example.com',
          password: 'SecureP@ss123',
          firstName: 'Token',
          lastName: 'Test',
        });

      userId = res.body.data.user.id;
      await pool.query(
        'UPDATE users SET status = $1 WHERE id = $2',
        ['active', userId]
      );

      const loginRes = await request(app)
        .post('/api/v1/auth/login')
        .send({ email: 'test-token@example.com', password: 'SecureP@ss123' });

      accessToken = loginRes.body.data.accessToken;
      refreshToken = loginRes.headers['set-cookie']
        ?.find(c => c.startsWith('refreshToken='))
        ?.split(';')[0]
        ?.replace('refreshToken=', '');
    });

    it('should refresh token successfully', async () => {
      const res = await request(app)
        .post('/api/v1/auth/refresh')
        .set('Cookie', `refreshToken=${refreshToken}`)
        .expect(200);

      expect(res.body.data.accessToken).toBeDefined();
      expect(res.body.data.accessToken).not.toBe(accessToken); // New token
    });

    it('should logout and invalidate refresh token', async () => {
      await request(app)
        .post('/api/v1/auth/logout')
        .set('Authorization', `Bearer ${accessToken}`)
        .set('Cookie', `refreshToken=${refreshToken}`)
        .expect(200);

      // Try to use the refresh token again
      const res = await request(app)
        .post('/api/v1/auth/refresh')
        .set('Cookie', `refreshToken=${refreshToken}`)
        .expect(401);

      expect(res.body.error.code).toBe('INVALID_REFRESH_TOKEN');
    });
  });
});
```

---

## 16.4 Contract Testing (Consumer-Driven)

Consumer-Driven Contract Testing ช่วยให้ services ไม่ break ซึ่งกันและกันโดยไม่รู้ตัว

### Using Pact

```javascript
// npm install @pact-foundation/pact

// order-service/tests/contract/userService.consumer.test.js
// ORDER SERVICE เป็น CONSUMER ของ USER SERVICE

const { PactV3, MatchersV3 } = require('@pact-foundation/pact');
const { string, integer, like } = MatchersV3;
const UserServiceClient = require('../../src/clients/userServiceClient');

const provider = new PactV3({
  consumer: 'order-service',
  provider: 'user-service',
  pactBranchAndVersion: {
    branch: process.env.GIT_BRANCH || 'main',
    version: process.env.APP_VERSION || '1.0.0',
  },
  dir: './pacts',
});

describe('Order Service (Consumer) → User Service (Provider)', () => {
  let client;

  describe('GET /api/v1/users/:id', () => {
    it('should get user by ID', async () => {
      // Define the expected interaction
      await provider
        .given('user with id user-123 exists')
        .uponReceiving('a request for user user-123')
        .withRequest({
          method: 'GET',
          path: '/api/v1/users/user-123',
          headers: {
            'x-user-id': string('user-123'),
            'x-request-id': string('req-1'),
          },
        })
        .willRespondWith({
          status: 200,
          headers: { 'Content-Type': 'application/json' },
          body: {
            success: true,
            data: {
              id: string('user-123'),
              email: string('user@example.com'),
              firstName: string('John'),
              lastName: string('Doe'),
              role: string('user'),
              status: string('active'),
            },
          },
        })
        .executeTest(async (mockServer) => {
          client = new UserServiceClient(mockServer.url);
          const user = await client.getUser('user-123');

          expect(user.id).toBe('user-123');
          expect(user.email).toBe('user@example.com');
        });
    });

    it('should handle user not found', async () => {
      await provider
        .given('user with id unknown-user does not exist')
        .uponReceiving('a request for non-existent user')
        .withRequest({
          method: 'GET',
          path: '/api/v1/users/unknown-user',
          headers: {
            'x-request-id': string('req-2'),
          },
        })
        .willRespondWith({
          status: 404,
          body: {
            success: false,
            error: {
              code: 'NOT_FOUND',
              message: string('User not found'),
            },
          },
        })
        .executeTest(async (mockServer) => {
          client = new UserServiceClient(mockServer.url);
          await expect(client.getUser('unknown-user')).rejects.toThrow();
        });
    });
  });
});
```

### Provider Verification (User Service)

```javascript
// user-service/tests/contract/provider.test.js
// USER SERVICE ตรวจสอบว่า fulfill contracts ที่ consumers คาดหวัง

const { PactV3 } = require('@pact-foundation/pact');
const app = require('../../src/app');

describe('User Service (Provider) Contract Verification', () => {
  it('should satisfy all consumer contracts', async () => {
    const verifier = new PactV3({
      provider: 'user-service',
      providerBaseUrl: `http://localhost:${process.env.PORT || 3001}`,
      pactUrls: [
        // Local pact files from consumers
        './pacts/order-service-user-service.json',
        './pacts/notification-service-user-service.json',
      ],
      // Or fetch from Pact Broker
      // pactBrokerUrl: process.env.PACT_BROKER_URL,
      // pactBrokerToken: process.env.PACT_BROKER_TOKEN,
    });

    // State handlers - setup test data
    verifier.addStateHandler('user with id user-123 exists', async () => {
      await testDb.query(`
        INSERT INTO users (id, email, first_name, last_name, role, status)
        VALUES ('user-123', 'user@example.com', 'John', 'Doe', 'user', 'active')
        ON CONFLICT (id) DO NOTHING
      `);
    });

    verifier.addStateHandler('user with id unknown-user does not exist', async () => {
      await testDb.query("DELETE FROM users WHERE id = 'unknown-user'");
    });

    await verifier.verifyProvider();
  });
});
```

---

## 16.5 End-to-End Testing

```javascript
// e2e/tests/orderFlow.test.js

const request = require('supertest');
const { v4: uuidv4 } = require('uuid');

// Use the actual running services
const API_GATEWAY = process.env.API_GATEWAY_URL || 'http://localhost:3000';

describe('Complete Order Flow E2E', () => {
  let userToken;
  let userId;
  let productId;
  let orderId;

  // Helper
  const api = (path) => `${API_GATEWAY}${path}`;

  describe('1. User Registration & Login', () => {
    const email = `e2e-${uuidv4()}@example.com`;

    it('should register a new user', async () => {
      const res = await request(API_GATEWAY)
        .post('/api/v1/auth/register')
        .send({
          email,
          password: 'E2eTest@123',
          firstName: 'E2E',
          lastName: 'Test',
        })
        .expect(201);

      userId = res.body.data.user.id;
      expect(userId).toBeDefined();
    });

    it('should login with credentials', async () => {
      // In real E2E, you'd verify email first
      // For testing, we can seed the database or use test-only endpoint
      const res = await request(API_GATEWAY)
        .post('/api/v1/auth/login')
        .send({ email, password: 'E2eTest@123' })
        .expect(200);

      userToken = res.body.data.accessToken;
      expect(userToken).toBeDefined();
    });
  });

  describe('2. Browse Products', () => {
    it('should list products', async () => {
      const res = await request(API_GATEWAY)
        .get('/api/v1/products')
        .expect(200);

      expect(res.body.data).toBeInstanceOf(Array);
      expect(res.body.data.length).toBeGreaterThan(0);
      productId = res.body.data[0].id;
    });

    it('should get product details', async () => {
      const res = await request(API_GATEWAY)
        .get(`/api/v1/products/${productId}`)
        .expect(200);

      expect(res.body.data.id).toBe(productId);
      expect(res.body.data.pricing).toBeDefined();
    });
  });

  describe('3. Create Order', () => {
    it('should create an order', async () => {
      const res = await request(API_GATEWAY)
        .post('/api/v1/orders')
        .set('Authorization', `Bearer ${userToken}`)
        .send({
          items: [{ productId, quantity: 1 }],
          shippingAddress: {
            street: '123 Test St',
            city: 'Bangkok',
            postalCode: '10100',
            country: 'TH',
          },
          paymentMethod: 'credit_card',
        })
        .expect(201);

      orderId = res.body.data.id;
      expect(orderId).toBeDefined();
      expect(res.body.data.status).toBe('pending');
    });

    it('should find the order in order history', async () => {
      const res = await request(API_GATEWAY)
        .get('/api/v1/orders')
        .set('Authorization', `Bearer ${userToken}`)
        .expect(200);

      const order = res.body.data.find(o => o.id === orderId);
      expect(order).toBeDefined();
    });
  });

  describe('4. Payment', () => {
    it('should process payment for order', async () => {
      const res = await request(API_GATEWAY)
        .post(`/api/v1/orders/${orderId}/payments`)
        .set('Authorization', `Bearer ${userToken}`)
        .send({
          paymentMethod: 'credit_card',
          cardNumber: '4111111111111111', // Test card
          expiryMonth: '12',
          expiryYear: '2025',
          cvv: '123',
        })
        .expect(201);

      expect(res.body.data.status).toBe('completed');
    });

    it('should update order status to confirmed after payment', async () => {
      // Wait a bit for async processing
      await new Promise(resolve => setTimeout(resolve, 1000));

      const res = await request(API_GATEWAY)
        .get(`/api/v1/orders/${orderId}`)
        .set('Authorization', `Bearer ${userToken}`)
        .expect(200);

      expect(res.body.data.status).toBe('confirmed');
    });
  });
});
```

---

## 16.6 Load Testing

```javascript
// load-tests/order-creation.js - k6 script

import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Counter, Trend } from 'k6/metrics';

// Custom metrics
const errorRate = new Rate('errors');
const orderCreatedCount = new Counter('orders_created');
const orderCreationDuration = new Trend('order_creation_duration', true);

// Test configuration
export const options = {
  stages: [
    { duration: '30s', target: 10 },   // Ramp up to 10 users
    { duration: '1m', target: 10 },    // Stay at 10 users
    { duration: '30s', target: 50 },   // Ramp up to 50 users
    { duration: '2m', target: 50 },    // Stay at 50 users
    { duration: '30s', target: 100 },  // Peak load
    { duration: '1m', target: 100 },
    { duration: '30s', target: 0 },    // Ramp down
  ],
  thresholds: {
    'http_req_duration': ['p(95)<2000'],  // 95% under 2 seconds
    'errors': ['rate<0.01'],              // Error rate under 1%
    'http_req_failed': ['rate<0.01'],
  },
};

const BASE_URL = __ENV.BASE_URL || 'http://localhost:3000';

// Setup: Get auth tokens
export function setup() {
  const loginRes = http.post(`${BASE_URL}/api/v1/auth/login`, JSON.stringify({
    email: 'loadtest@example.com',
    password: 'LoadTest@123',
  }), { headers: { 'Content-Type': 'application/json' } });

  return { token: JSON.parse(loginRes.body).data.accessToken };
}

export default function(data) {
  const headers = {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${data.token}`,
  };

  // Create order
  const start = Date.now();
  const orderRes = http.post(
    `${BASE_URL}/api/v1/orders`,
    JSON.stringify({
      items: [{ productId: 'test-product-id', quantity: 1 }],
      shippingAddress: {
        street: '123 Load Test St',
        city: 'Bangkok',
        postalCode: '10100',
        country: 'TH',
      },
    }),
    { headers }
  );

  const duration = Date.now() - start;
  orderCreationDuration.add(duration);

  const success = check(orderRes, {
    'status is 201': (r) => r.status === 201,
    'has order ID': (r) => JSON.parse(r.body).data?.id !== undefined,
    'response time < 2s': () => duration < 2000,
  });

  if (!success) {
    errorRate.add(1);
  } else {
    orderCreatedCount.add(1);
  }

  sleep(1);
}
```

---

## 16.7 Test Doubles (Mocks, Stubs, Fakes)

```javascript
// tests/helpers/testDoubles.js

// FAKE - Simple in-memory implementation for testing
class InMemoryUserRepository {
  constructor() {
    this.users = new Map();
    this.nextId = 1;
  }

  async findByEmail(email) {
    return Array.from(this.users.values()).find(u => u.email === email) || null;
  }

  async findById(id) {
    return this.users.get(id) || null;
  }

  async create(data) {
    const user = {
      id: `user-${this.nextId++}`,
      ...data,
      createdAt: new Date(),
      updatedAt: new Date(),
    };
    this.users.set(user.id, user);
    return user;
  }

  async update(id, data) {
    const user = this.users.get(id);
    if (!user) return null;
    const updated = { ...user, ...data, updatedAt: new Date() };
    this.users.set(id, updated);
    return updated;
  }

  async delete(id) {
    const user = this.users.get(id);
    this.users.delete(id);
    return user;
  }

  clear() {
    this.users.clear();
    this.nextId = 1;
  }
}

// STUB - Returns predefined responses
class EmailServiceStub {
  constructor() {
    this.sentEmails = [];
  }

  async sendVerification(email, token) {
    this.sentEmails.push({ type: 'verification', email, token });
  }

  async sendPasswordReset(email, token) {
    this.sentEmails.push({ type: 'reset', email, token });
  }

  async sendWelcomeEmail(email, name) {
    this.sentEmails.push({ type: 'welcome', email, name });
  }

  getEmailsSentTo(email) {
    return this.sentEmails.filter(e => e.email === email);
  }

  clear() {
    this.sentEmails = [];
  }
}

// SPY - Records calls (wraps real implementation)
class SpyEventBus {
  constructor(realEventBus) {
    this.real = realEventBus;
    this.publishedEvents = [];
    this.subscribers = new Map();
  }

  async publish(eventType, payload) {
    this.publishedEvents.push({ eventType, payload, publishedAt: new Date() });
    if (this.real) {
      return this.real.publish(eventType, payload);
    }
  }

  async subscribe(serviceName, pattern, handler) {
    this.subscribers.set(`${serviceName}:${pattern}`, handler);
    if (this.real) {
      return this.real.subscribe(serviceName, pattern, handler);
    }
  }

  getPublishedEvents(type) {
    return this.publishedEvents.filter(e => e.eventType === type);
  }

  async triggerEvent(eventType, payload) {
    const handler = this.subscribers.get(eventType);
    if (handler) await handler(payload, { type: eventType });
  }

  clear() {
    this.publishedEvents = [];
  }
}

module.exports = { InMemoryUserRepository, EmailServiceStub, SpyEventBus };
```

---

## 16.8 Test Infrastructure (Docker Compose)

```yaml
# docker-compose.test.yml
version: '3.8'

services:
  # Test databases - isolated from development
  postgres-test:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: test_users
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test
    ports:
      - "5433:5432"
    tmpfs:
      - /var/lib/postgresql/data  # In-memory for speed

  mongodb-test:
    image: mongo:7
    environment:
      MONGO_INITDB_DATABASE: test_products
    ports:
      - "27018:27017"
    tmpfs:
      - /data/db

  redis-test:
    image: redis:7-alpine
    ports:
      - "6380:6379"

  rabbitmq-test:
    image: rabbitmq:3.12-alpine
    ports:
      - "5673:5672"
```

### Jest Configuration

```javascript
// jest.config.js
module.exports = {
  testEnvironment: 'node',
  testMatch: ['**/*.test.js'],
  testPathIgnorePatterns: ['/node_modules/', '/e2e/'],
  collectCoverage: true,
  coverageDirectory: 'coverage',
  coverageReporters: ['text', 'lcov', 'html'],
  coverageThreshold: {
    global: {
      branches: 70,
      functions: 80,
      lines: 80,
      statements: 80,
    },
  },
  setupFilesAfterFramework: ['./tests/setup.js'],
  testTimeout: 10000,
  // Environment variables for tests
  testEnvironment: 'node',
  globalSetup: './tests/globalSetup.js',
  globalTeardown: './tests/globalTeardown.js',
};
```

### Test Setup

```javascript
// tests/setup.js
const dotenv = require('dotenv');
dotenv.config({ path: '.env.test' });

// Increase timeout for integration tests
jest.setTimeout(30000);

// Mock external services by default
jest.mock('../src/utils/emailService', () => ({
  sendVerification: jest.fn().mockResolvedValue(),
  sendWelcomeEmail: jest.fn().mockResolvedValue(),
  sendPasswordReset: jest.fn().mockResolvedValue(),
}));
```

```javascript
// tests/globalSetup.js
const { execSync } = require('child_process');

module.exports = async () => {
  console.log('Setting up test environment...');

  // Start test databases
  execSync('docker-compose -f docker-compose.test.yml up -d', {
    stdio: 'inherit',
  });

  // Wait for databases to be ready
  await waitForPostgres();
  await waitForMongo();

  // Run migrations
  process.env.DATABASE_URL = 'postgresql://test:test@localhost:5433/test_users';
  const { runMigrations } = require('../src/database/migrate');
  await runMigrations();

  console.log('Test environment ready');
};

async function waitForPostgres(maxAttempts = 30) {
  const { Pool } = require('pg');
  const pool = new Pool({ connectionString: 'postgresql://test:test@localhost:5433/test_users' });

  for (let i = 0; i < maxAttempts; i++) {
    try {
      await pool.query('SELECT 1');
      await pool.end();
      return;
    } catch {
      await new Promise(r => setTimeout(r, 1000));
    }
  }
  throw new Error('PostgreSQL not ready after 30 seconds');
}

module.exports.waitForPostgres = waitForPostgres;
```

---

## 16.9 Continuous Testing in CI/CD

```yaml
# .github/workflows/test.yml
name: Run Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        service: [user-service, product-service, order-service, payment-service]

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
          cache-dependency-path: ${{ matrix.service }}/package-lock.json

      - name: Install dependencies
        working-directory: ${{ matrix.service }}
        run: npm ci

      - name: Run unit tests
        working-directory: ${{ matrix.service }}
        run: npm run test:unit -- --coverage

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          directory: ${{ matrix.service }}/coverage
          flags: ${{ matrix.service }}-unit

  integration-tests:
    runs-on: ubuntu-latest
    needs: unit-tests

    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_DB: test_db
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      mongodb:
        image: mongo:7
        ports:
          - 27017:27017

      redis:
        image: redis:7
        ports:
          - 6379:6379

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Run integration tests
        env:
          TEST_DATABASE_URL: postgresql://test:test@localhost:5432/test_db
          REDIS_URL: redis://localhost:6379
        run: npm run test:integration

  contract-tests:
    runs-on: ubuntu-latest
    needs: unit-tests

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4

      - name: Run consumer contract tests
        run: npm run test:contract:consumer

      - name: Publish pacts to Pact Broker
        run: |
          npm run pact:publish
        env:
          PACT_BROKER_URL: ${{ secrets.PACT_BROKER_URL }}
          PACT_BROKER_TOKEN: ${{ secrets.PACT_BROKER_TOKEN }}

  e2e-tests:
    runs-on: ubuntu-latest
    needs: [unit-tests, integration-tests, contract-tests]

    steps:
      - uses: actions/checkout@v4

      - name: Start all services
        run: docker-compose up -d

      - name: Wait for services
        run: |
          ./scripts/wait-for-services.sh

      - name: Run E2E tests
        run: npm run test:e2e
        env:
          API_GATEWAY_URL: http://localhost:3000
```

---

## สรุป Part 16

ในบทนี้เราได้เรียนรู้:
- **Testing Pyramid** - strategy สำหรับ Microservices
- **Unit Tests** - isolate services ด้วย mocks
- **Integration Tests** - ทดสอบกับ real databases
- **Contract Testing** - Pact สำหรับ Consumer-Driven Contracts
- **E2E Tests** - ทดสอบ complete user flows
- **Load Testing** - k6 สำหรับ performance testing
- **Test Doubles** - Fake, Stub, Spy, Mock
- **Test Infrastructure** - Docker Compose สำหรับ test databases
- **CI/CD** - automated testing pipeline

**Next:** Part 17 - Kubernetes Introduction & Deployment

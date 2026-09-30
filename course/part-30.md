# Part 30: Advanced Testing - Contract Testing & Chaos Engineering

## บทนำ

ใน Microservices ที่มีหลาย service ทำงานร่วมกัน การทดสอบแบบดั้งเดิมยังไม่เพียงพอ เราต้องการ Contract Testing เพื่อรับประกันว่า services communicate กันถูกต้อง และ Chaos Engineering เพื่อทดสอบว่าระบบทนต่อความล้มเหลวได้

## 1. Consumer-Driven Contract Testing with Pact.js

### 1.1 Consumer Test (Client Side)

```javascript
// order-service/tests/contracts/consumer/orderService.contract.test.js
const { PactV3, MatchersV3 } = require('@pact-foundation/pact');
const { like, eachLike, integer, string, iso8601DateTimeWithMillis } = MatchersV3;
const path = require('path');
const axios = require('axios');

const pact = new PactV3({
  consumer: 'OrderService',
  provider: 'ProductService',
  dir: path.resolve(process.cwd(), 'pacts'),
  logLevel: 'info',
});

describe('OrderService → ProductService Contract', () => {
  describe('GET /products/:id', () => {
    test('returns product details when product exists', async () => {
      await pact
        .given('product with id prod-001 exists')
        .uponReceiving('a request for product prod-001')
        .withRequest({
          method: 'GET',
          path: '/api/v1/products/prod-001',
          headers: {
            Accept: 'application/json',
            Authorization: like('Bearer valid-token'),
          },
        })
        .willRespondWith({
          status: 200,
          headers: { 'Content-Type': 'application/json' },
          body: {
            data: {
              id: string('prod-001'),
              name: string('MacBook Pro 14"'),
              price: like(59900.00),
              currency: string('THB'),
              inventory: {
                available: integer(10),
              },
              isActive: like(true),
            }
          },
        })
        .executeTest(async (mockServer) => {
          const response = await axios.get(
            `${mockServer.url}/api/v1/products/prod-001`,
            {
              headers: {
                Accept: 'application/json',
                Authorization: 'Bearer valid-token',
              }
            }
          );
          
          expect(response.status).toBe(200);
          expect(response.data.data.id).toBe('prod-001');
          expect(response.data.data.price).toBeGreaterThan(0);
          expect(response.data.data.inventory.available).toBeGreaterThanOrEqual(0);
        });
    });
    
    test('returns 404 when product does not exist', async () => {
      await pact
        .given('product with id prod-999 does not exist')
        .uponReceiving('a request for non-existent product')
        .withRequest({
          method: 'GET',
          path: '/api/v1/products/prod-999',
          headers: { Accept: 'application/json' },
        })
        .willRespondWith({
          status: 404,
          headers: { 'Content-Type': 'application/json' },
          body: {
            error: {
              code: string('PRODUCT_NOT_FOUND'),
              message: like('Product not found'),
            }
          },
        })
        .executeTest(async (mockServer) => {
          try {
            await axios.get(`${mockServer.url}/api/v1/products/prod-999`, {
              headers: { Accept: 'application/json' },
            });
            fail('Should have thrown');
          } catch (error) {
            expect(error.response.status).toBe(404);
            expect(error.response.data.error.code).toBe('PRODUCT_NOT_FOUND');
          }
        });
    });
  });
  
  describe('POST /products/:id/reserve', () => {
    test('successfully reserves inventory', async () => {
      await pact
        .given('product prod-001 has 10 units available')
        .uponReceiving('a request to reserve 2 units of prod-001')
        .withRequest({
          method: 'POST',
          path: '/api/v1/products/prod-001/reserve',
          headers: {
            'Content-Type': 'application/json',
            Authorization: like('Bearer valid-token'),
          },
          body: {
            quantity: integer(2),
            orderId: string('order-abc'),
          },
        })
        .willRespondWith({
          status: 200,
          body: {
            data: {
              reservationId: string('res-123'),
              productId: string('prod-001'),
              quantity: integer(2),
              expiresAt: like('2024-01-01T00:00:00.000Z'),
            }
          },
        })
        .executeTest(async (mockServer) => {
          const response = await axios.post(
            `${mockServer.url}/api/v1/products/prod-001/reserve`,
            { quantity: 2, orderId: 'order-abc' },
            { headers: { Authorization: 'Bearer valid-token' } }
          );
          
          expect(response.status).toBe(200);
          expect(response.data.data.reservationId).toBeDefined();
          expect(response.data.data.quantity).toBe(2);
        });
    });
    
    test('returns 409 when insufficient stock', async () => {
      await pact
        .given('product prod-001 has only 1 unit available')
        .uponReceiving('a request to reserve 5 units when only 1 available')
        .withRequest({
          method: 'POST',
          path: '/api/v1/products/prod-001/reserve',
          headers: { 'Content-Type': 'application/json' },
          body: {
            quantity: integer(5),
            orderId: string('order-xyz'),
          },
        })
        .willRespondWith({
          status: 409,
          body: {
            error: {
              code: string('INSUFFICIENT_STOCK'),
              available: integer(1),
              requested: integer(5),
            }
          },
        })
        .executeTest(async (mockServer) => {
          try {
            await axios.post(
              `${mockServer.url}/api/v1/products/prod-001/reserve`,
              { quantity: 5, orderId: 'order-xyz' }
            );
          } catch (error) {
            expect(error.response.status).toBe(409);
            expect(error.response.data.error.code).toBe('INSUFFICIENT_STOCK');
          }
        });
    });
  });
});
```

### 1.2 Provider Verification Test

```javascript
// product-service/tests/contracts/provider/productService.provider.test.js
const { PactV3 } = require('@pact-foundation/pact');
const path = require('path');
const app = require('../../../src/app');

const provider = new PactV3({
  consumer: 'OrderService',
  provider: 'ProductService',
  pactUrls: [
    path.resolve(process.cwd(), '../order-service/pacts/OrderService-ProductService.json'),
  ],
  // Or use Pact Broker
  // pactBrokerUrl: process.env.PACT_BROKER_URL,
  // providerVersion: process.env.GIT_COMMIT,
  logLevel: 'info',
});

describe('ProductService Provider Verification', () => {
  let server;
  
  beforeAll(async () => {
    server = app.listen(0);
  });
  
  afterAll(() => server.close());
  
  test('verifies contract with OrderService', async () => {
    await provider
      .addStateHandler('product with id prod-001 exists', async () => {
        // Set up test data - insert product into test DB
        await testDb.query(`
          INSERT INTO products (id, name, price, currency, is_active)
          VALUES ('prod-001', 'MacBook Pro 14"', 59900.00, 'THB', true)
          ON CONFLICT (id) DO UPDATE SET is_active = true
        `);
        
        await testDb.query(`
          INSERT INTO inventory (product_id, available_quantity, reserved_quantity)
          VALUES ('prod-001', 10, 0)
          ON CONFLICT (product_id) DO UPDATE SET available_quantity = 10
        `);
      })
      .addStateHandler('product with id prod-999 does not exist', async () => {
        // Ensure product doesn't exist
        await testDb.query("DELETE FROM products WHERE id = 'prod-999'");
      })
      .addStateHandler('product prod-001 has 10 units available', async () => {
        await testDb.query(`
          UPDATE inventory SET available_quantity = 10 
          WHERE product_id = 'prod-001'
        `);
      })
      .addStateHandler('product prod-001 has only 1 unit available', async () => {
        await testDb.query(`
          UPDATE inventory SET available_quantity = 1 
          WHERE product_id = 'prod-001'
        `);
      })
      .verifyProvider({
        providerBaseUrl: `http://localhost:${server.address().port}`,
        requestFilter: (req, res, next) => {
          // Handle auth token
          if (req.headers.authorization?.startsWith('Bearer ')) {
            req.headers.authorization = `Bearer ${generateTestToken()}`;
          }
          next();
        },
      });
  });
});
```

### 1.3 Pact Broker Configuration

```yaml
# docker-compose.pact.yml
version: '3.8'
services:
  pact-broker:
    image: pactfoundation/pact-broker:2.107.1
    ports:
      - "9292:9292"
    environment:
      PACT_BROKER_DATABASE_URL: "postgres://pact:pact@pact-db/pact"
      PACT_BROKER_BASIC_AUTH_USERNAME: admin
      PACT_BROKER_BASIC_AUTH_PASSWORD: admin
      PACT_BROKER_ALLOW_PUBLIC_READ: 'true'
      PACT_BROKER_LOG_LEVEL: INFO
    depends_on:
      pact-db:
        condition: service_healthy
  
  pact-db:
    image: postgres:15
    environment:
      POSTGRES_DB: pact
      POSTGRES_USER: pact
      POSTGRES_PASSWORD: pact
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U pact"]
      interval: 10s
      timeout: 5s
      retries: 5
```

### 1.4 CI/CD Pipeline for Contract Tests

```yaml
# .github/workflows/contract-tests.yml
name: Contract Tests

on:
  push:
    branches: [main, develop]
  pull_request:

env:
  PACT_BROKER_URL: ${{ secrets.PACT_BROKER_URL }}
  PACT_BROKER_USERNAME: ${{ secrets.PACT_BROKER_USERNAME }}
  PACT_BROKER_PASSWORD: ${{ secrets.PACT_BROKER_PASSWORD }}

jobs:
  consumer-tests:
    name: Consumer Contract Tests
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
        working-directory: order-service
      
      - name: Run consumer contract tests
        run: npm run test:contracts:consumer
        working-directory: order-service
        env:
          PACT_BROKER_URL: ${{ env.PACT_BROKER_URL }}
          GIT_COMMIT: ${{ github.sha }}
          GIT_BRANCH: ${{ github.ref_name }}
      
      - name: Publish pacts to broker
        run: |
          npx pact-broker publish pacts \
            --broker-base-url $PACT_BROKER_URL \
            --broker-username $PACT_BROKER_USERNAME \
            --broker-password $PACT_BROKER_PASSWORD \
            --consumer-app-version ${{ github.sha }} \
            --branch ${{ github.ref_name }} \
            --tag ${{ github.ref_name }}
        working-directory: order-service
  
  provider-tests:
    name: Provider Contract Verification
    runs-on: ubuntu-latest
    needs: consumer-tests
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - name: Install dependencies
        run: npm ci
        working-directory: product-service
      
      - name: Run provider verification
        run: npm run test:contracts:provider
        working-directory: product-service
        env:
          PACT_BROKER_URL: ${{ env.PACT_BROKER_URL }}
          PACT_BROKER_USERNAME: ${{ env.PACT_BROKER_USERNAME }}
          PACT_BROKER_PASSWORD: ${{ env.PACT_BROKER_PASSWORD }}
          GIT_COMMIT: ${{ github.sha }}
          GIT_BRANCH: ${{ github.ref_name }}
          PACT_PROVIDER_VERSION: ${{ github.sha }}
      
      - name: Can I deploy?
        run: |
          npx pact-broker can-i-deploy \
            --pacticipant ProductService \
            --version ${{ github.sha }} \
            --to-environment production \
            --broker-base-url $PACT_BROKER_URL \
            --broker-username $PACT_BROKER_USERNAME \
            --broker-password $PACT_BROKER_PASSWORD
        working-directory: product-service
```

## 2. Mutation Testing with Stryker

### 2.1 Stryker Configuration

```json
// stryker.config.json
{
  "$schema": "./node_modules/@stryker-mutator/core/schema/stryker-schema.json",
  "mutate": [
    "src/**/*.js",
    "!src/**/*.test.js",
    "!src/migrations/**"
  ],
  "testRunner": "jest",
  "jest": {
    "projectType": "custom",
    "config": {
      "testMatch": ["**/*.test.js"],
      "testPathIgnorePatterns": ["contracts", "e2e", "integration"]
    }
  },
  "reporters": ["progress", "html", "json"],
  "thresholds": {
    "high": 80,
    "low": 60,
    "break": 50
  },
  "timeoutMS": 60000,
  "concurrency": 4,
  "htmlReporter": {
    "fileName": "reports/mutation/index.html"
  }
}
```

### 2.2 Mutation Test Example

```javascript
// src/services/pricingService.js
class PricingService {
  calculateDiscount(originalPrice, coupon) {
    if (!coupon || !coupon.isActive) {
      return 0;
    }
    
    if (new Date() > new Date(coupon.expiresAt)) {
      return 0;
    }
    
    if (originalPrice < coupon.minOrderAmount) {
      return 0;
    }
    
    let discount;
    if (coupon.type === 'PERCENTAGE') {
      discount = originalPrice * (coupon.value / 100);
    } else {
      discount = coupon.value;
    }
    
    // Cap at max discount
    if (coupon.maxDiscount && discount > coupon.maxDiscount) {
      discount = coupon.maxDiscount;
    }
    
    return Math.min(discount, originalPrice); // Can't discount more than price
  }
}

// src/services/pricingService.test.js - Tests that catch mutations
describe('PricingService', () => {
  let service;
  
  beforeEach(() => {
    service = new PricingService();
  });
  
  describe('calculateDiscount', () => {
    const validCoupon = {
      isActive: true,
      type: 'PERCENTAGE',
      value: 20,
      minOrderAmount: 100,
      maxDiscount: 500,
      expiresAt: new Date(Date.now() + 86400000).toISOString(),
    };
    
    test('returns 0 for inactive coupon', () => {
      const discount = service.calculateDiscount(1000, { ...validCoupon, isActive: false });
      expect(discount).toBe(0); // Catches mutation: isActive → !isActive
    });
    
    test('returns 0 for expired coupon', () => {
      const expiredCoupon = { ...validCoupon, expiresAt: new Date(Date.now() - 1000).toISOString() };
      const discount = service.calculateDiscount(1000, expiredCoupon);
      expect(discount).toBe(0); // Catches mutation: > → <
    });
    
    test('returns 0 when order below minimum', () => {
      const discount = service.calculateDiscount(50, validCoupon); // 50 < 100 min
      expect(discount).toBe(0); // Catches mutation: < → <=
    });
    
    test('calculates percentage discount correctly', () => {
      const discount = service.calculateDiscount(1000, validCoupon);
      expect(discount).toBe(200); // 1000 * 20/100 = 200
      // Catches mutation: * → /, value/100 → value*100
    });
    
    test('calculates fixed amount discount correctly', () => {
      const fixedCoupon = { ...validCoupon, type: 'FIXED', value: 150 };
      const discount = service.calculateDiscount(1000, fixedCoupon);
      expect(discount).toBe(150);
    });
    
    test('caps discount at maxDiscount', () => {
      const largeCoupon = { ...validCoupon, value: 60, maxDiscount: 300 };
      const discount = service.calculateDiscount(1000, largeCoupon); // 1000 * 60% = 600, capped at 300
      expect(discount).toBe(300); // Catches mutation: > → >=
    });
    
    test('caps discount at original price', () => {
      const hugeCoupon = { ...validCoupon, value: 200 }; // 200% discount
      const discount = service.calculateDiscount(100, { ...hugeCoupon, maxDiscount: 99999 });
      expect(discount).toBe(100); // Cannot exceed original price
      // Catches mutation: Math.min(discount, originalPrice) → Math.max
    });
    
    test('returns 0 for null coupon', () => {
      expect(service.calculateDiscount(1000, null)).toBe(0);
    });
    
    test('returns 0 for undefined coupon', () => {
      expect(service.calculateDiscount(1000, undefined)).toBe(0);
    });
  });
});
```

## 3. Architecture Testing

### 3.1 Dependency Rules Testing

```javascript
// tests/architecture/dependency.test.js
const madge = require('madge');
const path = require('path');

describe('Architecture Dependency Rules', () => {
  let dependencyGraph;
  
  beforeAll(async () => {
    dependencyGraph = await madge(path.resolve(__dirname, '../../src'), {
      fileExtensions: ['js', 'ts'],
      excludeRegExp: [/\.test\.(js|ts)$/, /node_modules/],
    });
  });
  
  test('no circular dependencies', async () => {
    const circular = dependencyGraph.circular();
    
    if (circular.length > 0) {
      const cycles = circular.map(cycle => cycle.join(' → ')).join('\n');
      throw new Error(`Circular dependencies found:\n${cycles}`);
    }
    
    expect(circular).toHaveLength(0);
  });
  
  test('controllers do not import from other controllers', () => {
    const deps = dependencyGraph.obj();
    
    const violations = [];
    Object.entries(deps).forEach(([file, imports]) => {
      if (!file.includes('/controllers/')) return;
      
      imports.forEach(dep => {
        if (dep.includes('/controllers/') && dep !== file) {
          violations.push(`${file} imports ${dep}`);
        }
      });
    });
    
    if (violations.length > 0) {
      throw new Error(`Layer violation - controllers importing controllers:\n${violations.join('\n')}`);
    }
  });
  
  test('repositories do not import from services', () => {
    const deps = dependencyGraph.obj();
    
    const violations = [];
    Object.entries(deps).forEach(([file, imports]) => {
      if (!file.includes('/repositories/')) return;
      
      imports.forEach(dep => {
        if (dep.includes('/services/')) {
          violations.push(`${file} imports ${dep}`);
        }
      });
    });
    
    expect(violations).toHaveLength(0);
  });
  
  test('services do not import from controllers', () => {
    const deps = dependencyGraph.obj();
    
    const violations = [];
    Object.entries(deps).forEach(([file, imports]) => {
      if (!file.includes('/services/')) return;
      
      imports.forEach(dep => {
        if (dep.includes('/controllers/')) {
          violations.push(`${file} imports ${dep}`);
        }
      });
    });
    
    expect(violations).toHaveLength(0);
  });
  
  test('maximum dependency depth from entry point is less than 10', () => {
    const depth = dependencyGraph.depends('src/app.js');
    // Check all transitive dependencies
    expect(depth.length).toBeLessThan(50); // Reasonable limit
  });
});
```

### 3.2 Service Interface Testing

```javascript
// tests/architecture/interface.test.js
const fs = require('fs');
const path = require('path');

describe('Service Interface Contracts', () => {
  const servicesDir = path.resolve(__dirname, '../../src/services');
  const serviceFiles = fs.readdirSync(servicesDir)
    .filter(f => f.endsWith('.js') && !f.includes('.test.'));
  
  test('all services export a class', () => {
    serviceFiles.forEach(file => {
      const ServiceClass = require(path.join(servicesDir, file));
      expect(typeof ServiceClass).toBe('function');
    });
  });
  
  test('all service constructors accept config/dependencies', () => {
    serviceFiles.forEach(file => {
      const ServiceClass = require(path.join(servicesDir, file));
      // Constructor should accept at least one parameter (dependencies)
      expect(ServiceClass.length).toBeGreaterThanOrEqual(0);
    });
  });
  
  test('event emitter services emit correctly typed events', () => {
    // Add specific service interface tests
    const OrderService = require('../../src/services/orderService');
    const service = new OrderService({ db: mockDb, eventBus: mockEventBus });
    
    expect(service).toHaveProperty('create');
    expect(service).toHaveProperty('findById');
    expect(service).toHaveProperty('update');
    expect(service).toHaveProperty('cancel');
    
    expect(typeof service.create).toBe('function');
    expect(typeof service.findById).toBe('function');
  });
});
```

## 4. Chaos Engineering

### 4.1 Chaos Test Framework

```javascript
// chaos/chaosEngine.js
const axios = require('axios');

class ChaosEngine {
  constructor(config) {
    this.baseUrl = config.baseUrl;
    this.scenarios = [];
    this.results = [];
  }
  
  addScenario(scenario) {
    this.scenarios.push(scenario);
    return this;
  }
  
  async runAll(options = {}) {
    const { parallel = false, dryRun = false } = options;
    
    console.log(`Running ${this.scenarios.length} chaos scenarios...`);
    
    if (parallel) {
      this.results = await Promise.allSettled(
        this.scenarios.map(s => this.runScenario(s, dryRun))
      );
    } else {
      for (const scenario of this.scenarios) {
        const result = await this.runScenario(scenario, dryRun);
        this.results.push(result);
      }
    }
    
    return this.generateReport();
  }
  
  async runScenario(scenario, dryRun = false) {
    const startTime = Date.now();
    console.log(`\n🔥 Running chaos scenario: ${scenario.name}`);
    console.log(`   Description: ${scenario.description}`);
    
    if (dryRun) {
      console.log('   [DRY RUN] Skipping execution');
      return { scenario: scenario.name, skipped: true };
    }
    
    try {
      // Apply chaos
      await scenario.inject(this.baseUrl);
      console.log('   ✓ Chaos injected');
      
      // Wait for chaos to take effect
      await this.sleep(scenario.stabilizationDelay || 2000);
      
      // Verify behavior under chaos
      const verificationResults = await scenario.verify(this.baseUrl);
      console.log('   ✓ System behavior verified');
      
      // Cleanup
      await scenario.cleanup(this.baseUrl);
      console.log('   ✓ Cleanup completed');
      
      const passed = verificationResults.every(r => r.passed);
      
      return {
        scenario: scenario.name,
        passed,
        duration: Date.now() - startTime,
        verifications: verificationResults,
      };
      
    } catch (error) {
      // Always cleanup even on failure
      try {
        await scenario.cleanup(this.baseUrl);
      } catch (cleanupError) {
        console.error('   ✗ Cleanup failed:', cleanupError.message);
      }
      
      return {
        scenario: scenario.name,
        passed: false,
        error: error.message,
        duration: Date.now() - startTime,
      };
    }
  }
  
  generateReport() {
    const passed = this.results.filter(r => r.passed).length;
    const failed = this.results.filter(r => !r.passed && !r.skipped).length;
    
    console.log('\n📊 Chaos Engineering Report');
    console.log('=' .repeat(50));
    console.log(`Total scenarios: ${this.results.length}`);
    console.log(`Passed: ${passed}`);
    console.log(`Failed: ${failed}`);
    console.log('');
    
    this.results.forEach(result => {
      const icon = result.passed ? '✅' : result.skipped ? '⏭️' : '❌';
      console.log(`${icon} ${result.scenario}`);
      if (result.error) {
        console.log(`   Error: ${result.error}`);
      }
    });
    
    return {
      summary: { total: this.results.length, passed, failed },
      results: this.results,
    };
  }
  
  sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

module.exports = ChaosEngine;
```

### 4.2 Chaos Scenarios

```javascript
// chaos/scenarios/serviceUnavailable.js
const axios = require('axios');

// Scenario 1: Downstream service unavailable
const inventoryServiceDown = {
  name: 'inventory-service-down',
  description: 'Order service should gracefully handle inventory service being down',
  stabilizationDelay: 3000,
  
  async inject(baseUrl) {
    // Use Toxiproxy to simulate service failure
    await axios.post('http://localhost:8474/proxies/inventory-service/toxics', {
      name: 'connection_reset',
      type: 'reset_peer',
      stream: 'downstream',
      toxicity: 1.0,
    });
  },
  
  async verify(baseUrl) {
    const results = [];
    
    // Test 1: Create order should succeed (graceful degradation with fallback)
    try {
      const response = await axios.post(`${baseUrl}/api/v1/orders`, {
        customerId: 'test-customer',
        items: [{ productId: 'prod-001', quantity: 1 }],
      }, { timeout: 5000 });
      
      results.push({
        test: 'order creation with degraded inventory check',
        passed: response.status === 201 || response.status === 202,
        detail: `Status: ${response.status}`,
      });
    } catch (error) {
      // If timeout/circuit open, should be graceful
      results.push({
        test: 'order creation with degraded inventory check',
        passed: error.response?.status === 503, // Service unavailable is acceptable
        detail: error.message,
      });
    }
    
    // Test 2: Health endpoint should report degraded state (not fail)
    const healthResponse = await axios.get(`${baseUrl}/health/ready`);
    results.push({
      test: 'health check shows degraded state',
      passed: [200, 503].includes(healthResponse.status) && 
               healthResponse.data.checks?.inventoryService !== undefined,
      detail: JSON.stringify(healthResponse.data),
    });
    
    return results;
  },
  
  async cleanup(baseUrl) {
    // Remove toxic to restore service
    await axios.delete('http://localhost:8474/proxies/inventory-service/toxics/connection_reset');
  },
};

// Scenario 2: Network latency
const highLatency = {
  name: 'high-network-latency',
  description: 'System should handle 3-second network latency without cascading failures',
  stabilizationDelay: 1000,
  
  async inject(baseUrl) {
    await axios.post('http://localhost:8474/proxies/database/toxics', {
      name: 'latency',
      type: 'latency',
      stream: 'downstream',
      toxicity: 1.0,
      attributes: { latency: 3000, jitter: 500 },
    });
  },
  
  async verify(baseUrl) {
    const results = [];
    const start = Date.now();
    
    try {
      const response = await axios.get(`${baseUrl}/api/v1/products?limit=5`, {
        timeout: 10000,
      });
      
      const duration = Date.now() - start;
      results.push({
        test: 'API responds within timeout under high latency',
        passed: response.status === 200,
        detail: `Response time: ${duration}ms`,
      });
      
      // Should still use cache for repeated requests
      const start2 = Date.now();
      await axios.get(`${baseUrl}/api/v1/products?limit=5`);
      const cachedDuration = Date.now() - start2;
      
      results.push({
        test: 'Cached responses are fast under high latency',
        passed: cachedDuration < 1000, // Should be cached
        detail: `Cached response time: ${cachedDuration}ms`,
      });
      
    } catch (error) {
      results.push({
        test: 'API responds under high latency',
        passed: false,
        detail: error.message,
      });
    }
    
    return results;
  },
  
  async cleanup() {
    await axios.delete('http://localhost:8474/proxies/database/toxics/latency');
  },
};

// Scenario 3: Memory pressure
const memoryPressure = {
  name: 'memory-pressure',
  description: 'Service should not crash under memory pressure',
  stabilizationDelay: 5000,
  
  async inject(baseUrl) {
    // Trigger memory-intensive operations
    const promises = Array.from({ length: 100 }, () =>
      axios.get(`${baseUrl}/api/v1/products?limit=100`, { timeout: 30000 })
        .catch(() => {}) // Ignore individual failures
    );
    
    // Don't await - let them run in background
    Promise.allSettled(promises);
  },
  
  async verify(baseUrl) {
    const results = [];
    
    // Wait for load spike to peak
    await new Promise(resolve => setTimeout(resolve, 2000));
    
    // Service should still respond
    try {
      const response = await axios.get(`${baseUrl}/health/live`, { timeout: 5000 });
      results.push({
        test: 'Service stays alive under memory pressure',
        passed: response.status === 200,
        detail: `Status: ${response.status}`,
      });
    } catch (error) {
      results.push({
        test: 'Service stays alive under memory pressure',
        passed: false,
        detail: error.message,
      });
    }
    
    // Check for memory leak indicators
    const metricsResponse = await axios.get(`${baseUrl}/metrics`);
    const memoryMatch = metricsResponse.data.match(/nodejs_heap_size_used_bytes (\d+)/);
    
    if (memoryMatch) {
      const heapUsed = parseInt(memoryMatch[1]);
      const heapUsedMB = heapUsed / 1024 / 1024;
      
      results.push({
        test: 'Heap memory stays under 512MB under pressure',
        passed: heapUsedMB < 512,
        detail: `Heap used: ${heapUsedMB.toFixed(1)}MB`,
      });
    }
    
    return results;
  },
  
  async cleanup() {
    // Just wait for requests to complete
    await new Promise(resolve => setTimeout(resolve, 3000));
  },
};

module.exports = { inventoryServiceDown, highLatency, memoryPressure };
```

### 4.3 Running Chaos Tests

```javascript
// chaos/runChaos.js
const ChaosEngine = require('./chaosEngine');
const { inventoryServiceDown, highLatency, memoryPressure } = require('./scenarios/serviceUnavailable');

async function runChaosTests() {
  const engine = new ChaosEngine({
    baseUrl: process.env.TARGET_URL || 'http://localhost:3000',
  });
  
  engine
    .addScenario(inventoryServiceDown)
    .addScenario(highLatency)
    .addScenario(memoryPressure);
  
  const report = await engine.runAll({
    parallel: false, // Run sequentially to avoid interference
    dryRun: process.env.DRY_RUN === 'true',
  });
  
  // Exit with error code if any scenario failed
  if (report.summary.failed > 0) {
    console.error('\n❌ Chaos engineering tests FAILED');
    process.exit(1);
  } else {
    console.log('\n✅ All chaos engineering tests PASSED');
    process.exit(0);
  }
}

runChaosTests().catch(error => {
  console.error('Chaos test runner failed:', error);
  process.exit(1);
});
```

## 5. Load Testing with k6

### 5.1 Load Test Script

```javascript
// load-tests/scenarios/endToEnd.js
import http from 'k6/http';
import { check, group, sleep } from 'k6';
import { Rate, Trend, Counter } from 'k6/metrics';
import { htmlReport } from 'https://raw.githubusercontent.com/benc-uk/k6-reporter/main/dist/bundle.js';

// Custom metrics
const errorRate = new Rate('error_rate');
const orderCreateTrend = new Trend('order_create_duration');
const checkoutSuccessCounter = new Counter('checkout_success');

export const options = {
  scenarios: {
    // Smoke test: minimal load
    smoke: {
      executor: 'constant-vus',
      vus: 1,
      duration: '1m',
      tags: { scenario: 'smoke' },
    },
    
    // Load test: normal traffic
    load: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '2m', target: 50 },  // Ramp up
        { duration: '5m', target: 50 },  // Sustain
        { duration: '2m', target: 0 },   // Ramp down
      ],
      tags: { scenario: 'load' },
      startTime: '2m',
    },
    
    // Stress test: find breaking point
    stress: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '2m', target: 100 },
        { duration: '5m', target: 100 },
        { duration: '2m', target: 200 },
        { duration: '5m', target: 200 },
        { duration: '2m', target: 300 },
        { duration: '5m', target: 300 },
        { duration: '5m', target: 0 },
      ],
      tags: { scenario: 'stress' },
      startTime: '12m',
    },
    
    // Spike test: sudden traffic burst
    spike: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '30s', target: 500 },  // Sudden spike
        { duration: '1m', target: 500 },   // Sustain spike
        { duration: '30s', target: 0 },    // Drop immediately
      ],
      tags: { scenario: 'spike' },
      startTime: '42m',
    },
  },
  
  thresholds: {
    http_req_duration: ['p(95)<500', 'p(99)<1000'],
    http_req_failed: ['rate<0.01'],  // <1% error rate
    error_rate: ['rate<0.01'],
    order_create_duration: ['p(95)<2000'],
  },
};

const BASE_URL = __ENV.BASE_URL || 'http://localhost:3000';
const TOKEN_POOL = JSON.parse(__ENV.TOKENS || '["token1", "token2", "token3"]');

function getToken() {
  return TOKEN_POOL[Math.floor(Math.random() * TOKEN_POOL.length)];
}

export default function () {
  const token = getToken();
  const headers = {
    'Content-Type': 'application/json',
    Authorization: `Bearer ${token}`,
  };
  
  group('Browse Products', () => {
    // List products
    const listRes = http.get(`${BASE_URL}/api/v1/products?limit=20`, { headers });
    check(listRes, {
      'products list status 200': r => r.status === 200,
      'products list has data': r => JSON.parse(r.body).data?.length > 0,
    }) || errorRate.add(1);
    
    sleep(1);
    
    // Get single product
    const products = JSON.parse(listRes.body).data || [];
    if (products.length > 0) {
      const productId = products[0].id;
      const productRes = http.get(`${BASE_URL}/api/v1/products/${productId}`, { headers });
      check(productRes, {
        'product detail status 200': r => r.status === 200,
      }) || errorRate.add(1);
    }
  });
  
  sleep(2);
  
  group('Create Order', () => {
    const orderPayload = JSON.stringify({
      items: [
        { productId: 'prod-001', quantity: 1 },
        { productId: 'prod-002', quantity: 2 },
      ],
      shippingAddress: {
        street: '123 Test St',
        city: 'Bangkok',
        postalCode: '10100',
      },
    });
    
    const startTime = Date.now();
    const createRes = http.post(`${BASE_URL}/api/v1/orders`, orderPayload, { headers });
    orderCreateTrend.add(Date.now() - startTime);
    
    const success = check(createRes, {
      'order created status 201': r => r.status === 201,
      'order has id': r => JSON.parse(r.body).data?.id !== undefined,
    });
    
    if (success) {
      checkoutSuccessCounter.add(1);
    } else {
      errorRate.add(1);
    }
  });
  
  sleep(1);
}

export function handleSummary(data) {
  return {
    'reports/load-test/summary.html': htmlReport(data),
    'reports/load-test/summary.json': JSON.stringify(data, null, 2),
  };
}
```

### 5.2 k6 Docker Integration

```yaml
# docker-compose.load-test.yml
version: '3.8'
services:
  k6:
    image: grafana/k6:0.47.0
    volumes:
      - ./load-tests:/scripts
      - ./reports:/reports
    environment:
      BASE_URL: http://app:3000
      TOKENS: '["jwt-token-1", "jwt-token-2"]'
    command: run /scripts/scenarios/endToEnd.js
    depends_on:
      - app
    
  influxdb:
    image: influxdb:1.8
    environment:
      INFLUXDB_DB: k6
    ports:
      - "8086:8086"
  
  grafana:
    image: grafana/grafana:10.0.0
    ports:
      - "3001:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
    volumes:
      - ./monitoring/grafana/provisioning:/etc/grafana/provisioning
```

## 6. Test Coverage Analysis

```javascript
// scripts/coverage-report.js
const fs = require('fs');
const path = require('path');

function analyzeCoverage(coverageFile) {
  const coverage = JSON.parse(fs.readFileSync(coverageFile, 'utf8'));
  
  const analysis = {
    totalFiles: 0,
    coveredFiles: 0,
    uncoveredLines: [],
    criticalUncovered: [],
  };
  
  Object.entries(coverage).forEach(([file, data]) => {
    analysis.totalFiles++;
    
    const statements = Object.values(data.s);
    const coveredStatements = statements.filter(count => count > 0).length;
    const coveragePercent = statements.length > 0 
      ? (coveredStatements / statements.length) * 100 
      : 100;
    
    if (coveragePercent < 100) {
      const uncoveredLines = Object.entries(data.s)
        .filter(([, count]) => count === 0)
        .map(([key]) => data.statementMap[key].start.line);
      
      analysis.uncoveredLines.push({
        file: path.relative(process.cwd(), file),
        coverage: coveragePercent.toFixed(1),
        uncoveredLines: [...new Set(uncoveredLines)].sort((a, b) => a - b),
      });
      
      // Flag critical files (services, controllers)
      if (file.includes('/services/') || file.includes('/controllers/')) {
        if (coveragePercent < 80) {
          analysis.criticalUncovered.push({
            file: path.relative(process.cwd(), file),
            coverage: coveragePercent.toFixed(1),
          });
        }
      }
    } else {
      analysis.coveredFiles++;
    }
  });
  
  // Sort by coverage ascending (worst coverage first)
  analysis.uncoveredLines.sort((a, b) => parseFloat(a.coverage) - parseFloat(b.coverage));
  
  console.log('\n📊 Test Coverage Analysis');
  console.log('='.repeat(50));
  console.log(`Files: ${analysis.coveredFiles}/${analysis.totalFiles} fully covered`);
  
  if (analysis.criticalUncovered.length > 0) {
    console.log('\n⚠️  Critical files with low coverage (< 80%):');
    analysis.criticalUncovered.forEach(({ file, coverage }) => {
      console.log(`  ❌ ${file}: ${coverage}%`);
    });
  }
  
  console.log('\n📉 Files with lowest coverage:');
  analysis.uncoveredLines.slice(0, 10).forEach(({ file, coverage, uncoveredLines }) => {
    console.log(`  ${file}: ${coverage}% (uncovered lines: ${uncoveredLines.slice(0, 5).join(', ')}${uncoveredLines.length > 5 ? '...' : ''})`);
  });
  
  return analysis;
}

// Run if called directly
if (require.main === module) {
  const coverageFile = process.argv[2] || 'coverage/coverage-final.json';
  if (!fs.existsSync(coverageFile)) {
    console.error(`Coverage file not found: ${coverageFile}`);
    process.exit(1);
  }
  
  const analysis = analyzeCoverage(coverageFile);
  
  if (analysis.criticalUncovered.length > 0) {
    console.error('\n❌ Critical files have insufficient coverage');
    process.exit(1);
  }
  
  console.log('\n✅ Coverage analysis complete');
}
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Contract Testing with Pact** - Consumer-driven contracts ที่รับประกัน API compatibility ระหว่าง services
2. **Provider State Management** - Setup test data ก่อน provider verification
3. **Pact Broker** - Centralized contract storage และ "Can I Deploy?" checks
4. **Mutation Testing** - Verify test quality ด้วย Stryker (kill rate > 80% = good tests)
5. **Architecture Testing** - Automated detection ของ circular dependencies และ layer violations
6. **Chaos Engineering** - Test resilience ด้วย failure injection (service down, high latency, memory pressure)
7. **Load Testing with k6** - Scenarios, thresholds, custom metrics และ HTML reports
8. **Coverage Analysis** - Identify critical uncovered paths

## แบบฝึกหัด

1. เขียน Consumer contract test ระหว่าง `PaymentService` และ `OrderService`
2. Setup Pact Broker และ integrate กับ CI/CD pipeline
3. Run Stryker บน `pricingService.js` และ improve tests เพื่อ kill rate > 85%
4. Implement chaos scenario สำหรับ database connection pool exhaustion
5. สร้าง k6 load test ที่ simulate real user journey (browse → add to cart → checkout)

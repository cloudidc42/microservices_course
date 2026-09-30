# Part 09: Service-to-Service Communication
## การสื่อสารระหว่าง Microservices - Synchronous และ Asynchronous

> **ระดับ:** ⭐⭐ พื้นฐาน | **เวลาเรียน:** 5-6 ชั่วโมง | **Prerequisites:** Parts 01-08

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Synchronous vs Asynchronous Communication
- HTTP/REST Communication Patterns
- Service Discovery
- Retry Logic, Timeout, Circuit Breaker
- Event-Driven Communication พื้นฐาน
- Workshop: Resilient Service Communication

---

## 1. Communication Patterns

### 1.1 Synchronous vs Asynchronous

```
Synchronous (Request-Response):
Service A ─────────────────────► Service B
         ◄─────────────────────
         (รอจนได้ response)

✅ ใช้เมื่อ:
  - ต้องการ response ทันที
  - Read operations
  - ต้องการ confirmation ก่อนดำเนินการต่อ

❌ ปัญหา:
  - ถ้า Service B ช้า = Service A ช้าด้วย
  - ถ้า Service B down = Service A ล้มเหลว
  - Cascading failures

Asynchronous (Event-Driven):
Service A ─────────────────────► Message Queue
                                      │
                                      ▼
                                  Service B
         (ไม่รอ response)

✅ ใช้เมื่อ:
  - ไม่ต้องการ response ทันที
  - Long-running operations
  - Fire-and-forget

✅ ข้อดี:
  - Service A ไม่ต้องรอ Service B
  - Service B down ≠ Service A fail
  - รองรับ Load spikes ได้ดีกว่า
```

### 1.2 Communication Styles

```
Style 1: Direct HTTP (REST/gRPC)
  Service A ──HTTP──► Service B

Style 2: API Gateway
  Client ──► API Gateway ──► Service A
                         ──► Service B

Style 3: Message Broker
  Service A ──► RabbitMQ/Kafka ──► Service B

Style 4: Service Mesh
  Service A ──Sidecar──Network──Sidecar──► Service B
  (Istio/Envoy handles: retries, tracing, mTLS)
```

---

## 2. HTTP Communication ระหว่าง Services

### 2.1 Basic Service Client

```javascript
// src/clients/base.client.js
const axios = require('axios');
const logger = require('../utils/logger');

class BaseServiceClient {
  constructor(serviceName, baseUrl, options = {}) {
    this.serviceName = serviceName;
    
    this.client = axios.create({
      baseURL: baseUrl,
      timeout: options.timeout || 10000,  // 10 seconds default
      headers: {
        'Content-Type': 'application/json',
        'X-Service-Name': process.env.SERVICE_NAME || 'unknown',
      },
    });
    
    // Request interceptor
    this.client.interceptors.request.use(
      (config) => {
        // Add request ID for tracing
        config.headers['X-Request-ID'] = require('uuid').v4();
        
        logger.debug(`→ ${serviceName}`, {
          method: config.method?.toUpperCase(),
          url: config.url,
          requestId: config.headers['X-Request-ID'],
        });
        
        config.metadata = { startTime: Date.now() };
        return config;
      }
    );
    
    // Response interceptor
    this.client.interceptors.response.use(
      (response) => {
        const duration = Date.now() - response.config.metadata.startTime;
        
        logger.debug(`← ${serviceName}`, {
          status: response.status,
          duration,
          url: response.config.url,
        });
        
        return response;
      },
      (error) => {
        const duration = Date.now() - (error.config?.metadata?.startTime || Date.now());
        
        logger.warn(`✗ ${serviceName} error`, {
          status: error.response?.status,
          message: error.message,
          url: error.config?.url,
          duration,
        });
        
        return Promise.reject(error);
      }
    );
  }
  
  async get(path, params = {}) {
    const response = await this.client.get(path, { params });
    return response.data;
  }
  
  async post(path, data) {
    const response = await this.client.post(path, data);
    return response.data;
  }
  
  async put(path, data) {
    const response = await this.client.put(path, data);
    return response.data;
  }
  
  async patch(path, data) {
    const response = await this.client.patch(path, data);
    return response.data;
  }
  
  async delete(path) {
    const response = await this.client.delete(path);
    return response.data;
  }
}

module.exports = BaseServiceClient;
```

### 2.2 Retry Logic

```javascript
// src/clients/retry.client.js
const BaseServiceClient = require('./base.client');
const logger = require('../utils/logger');

class RetryClient extends BaseServiceClient {
  constructor(serviceName, baseUrl, options = {}) {
    super(serviceName, baseUrl, options);
    
    this.maxRetries = options.maxRetries || 3;
    this.retryDelay = options.retryDelay || 1000;  // 1 second
    this.retryMultiplier = options.retryMultiplier || 2;  // Exponential backoff
    
    // Retryable HTTP status codes
    this.retryableStatuses = options.retryableStatuses || [429, 500, 502, 503, 504];
  }
  
  async withRetry(operation, context = '') {
    let lastError;
    
    for (let attempt = 1; attempt <= this.maxRetries; attempt++) {
      try {
        return await operation();
        
      } catch (error) {
        lastError = error;
        
        // ไม่ retry ถ้าเป็น Client Error (4xx) ยกเว้น 429
        const status = error.response?.status;
        if (status && status >= 400 && status < 500 && status !== 429) {
          throw error;
        }
        
        // ไม่ retry ถ้าเป็น attempt สุดท้าย
        if (attempt === this.maxRetries) {
          logger.error(`${context} failed after ${this.maxRetries} attempts`, {
            error: error.message,
          });
          break;
        }
        
        // Calculate delay with exponential backoff + jitter
        const delay = this.retryDelay * Math.pow(this.retryMultiplier, attempt - 1);
        const jitter = Math.random() * 200;  // Add jitter ป้องกัน thundering herd
        const totalDelay = delay + jitter;
        
        logger.warn(`${context} attempt ${attempt} failed, retrying in ${totalDelay.toFixed(0)}ms`, {
          error: error.message,
          attempt,
          nextRetryIn: totalDelay,
        });
        
        await new Promise(resolve => setTimeout(resolve, totalDelay));
      }
    }
    
    throw lastError;
  }
  
  async get(path, params) {
    return this.withRetry(
      () => super.get(path, params),
      `GET ${path}`
    );
  }
  
  async post(path, data) {
    return this.withRetry(
      () => super.post(path, data),
      `POST ${path}`
    );
  }
}

module.exports = RetryClient;
```

### 2.3 Circuit Breaker Pattern

```javascript
// src/clients/circuit-breaker.client.js
const RetryClient = require('./retry.client');
const logger = require('../utils/logger');

const CircuitState = {
  CLOSED: 'CLOSED',    // ปกติ - requests ผ่าน
  OPEN: 'OPEN',        // Circuit เปิด - reject requests ทันที
  HALF_OPEN: 'HALF_OPEN', // ทดสอบ - allow probe requests
};

class CircuitBreaker {
  constructor(name, options = {}) {
    this.name = name;
    this.state = CircuitState.CLOSED;
    
    this.failureThreshold = options.failureThreshold || 5;   // ล้มเหลว 5 ครั้ง
    this.successThreshold = options.successThreshold || 2;    // สำเร็จ 2 ครั้งเพื่อ close
    this.timeout = options.timeout || 60000;                  // 60 วินาทีก่อน retry
    
    this.failureCount = 0;
    this.successCount = 0;
    this.lastFailureTime = null;
    this.nextAttemptTime = null;
  }
  
  async execute(operation) {
    // State: OPEN - ไม่ allow requests
    if (this.state === CircuitState.OPEN) {
      if (Date.now() < this.nextAttemptTime) {
        const waitTime = ((this.nextAttemptTime - Date.now()) / 1000).toFixed(0);
        
        logger.warn(`Circuit OPEN for ${this.name}, retry in ${waitTime}s`);
        
        throw new Error(
          `Circuit breaker OPEN for ${this.name}. Service unavailable.`
        );
      }
      
      // Transition to HALF_OPEN after timeout
      this.state = CircuitState.HALF_OPEN;
      logger.info(`Circuit HALF_OPEN for ${this.name}, testing...`);
    }
    
    try {
      const result = await operation();
      this.onSuccess();
      return result;
      
    } catch (error) {
      this.onFailure(error);
      throw error;
    }
  }
  
  onSuccess() {
    this.failureCount = 0;
    
    if (this.state === CircuitState.HALF_OPEN) {
      this.successCount++;
      
      if (this.successCount >= this.successThreshold) {
        this.state = CircuitState.CLOSED;
        this.successCount = 0;
        logger.info(`Circuit CLOSED for ${this.name} - service recovered`);
      }
    }
  }
  
  onFailure(error) {
    this.lastFailureTime = Date.now();
    this.failureCount++;
    this.successCount = 0;
    
    if (
      this.state === CircuitState.HALF_OPEN ||
      this.failureCount >= this.failureThreshold
    ) {
      this.state = CircuitState.OPEN;
      this.nextAttemptTime = Date.now() + this.timeout;
      
      logger.error(`Circuit OPENED for ${this.name}`, {
        failureCount: this.failureCount,
        error: error.message,
        nextAttempt: new Date(this.nextAttemptTime).toISOString(),
      });
    }
  }
  
  getStats() {
    return {
      name: this.name,
      state: this.state,
      failureCount: this.failureCount,
      successCount: this.successCount,
      lastFailureTime: this.lastFailureTime 
        ? new Date(this.lastFailureTime).toISOString() 
        : null,
    };
  }
}

// Service Client with Circuit Breaker
class ResilientServiceClient extends RetryClient {
  constructor(serviceName, baseUrl, options = {}) {
    super(serviceName, baseUrl, options);
    
    this.circuitBreaker = new CircuitBreaker(serviceName, {
      failureThreshold: options.failureThreshold || 5,
      successThreshold: options.successThreshold || 2,
      timeout: options.circuitTimeout || 60000,
    });
    
    this.fallback = options.fallback || null;
  }
  
  async execute(operation, fallbackValue = null) {
    try {
      return await this.circuitBreaker.execute(operation);
    } catch (error) {
      // Circuit is open or all retries failed
      if (this.fallback) {
        logger.warn(`Using fallback for ${this.serviceName}`, {
          error: error.message,
        });
        return this.fallback(error);
      }
      
      if (fallbackValue !== null) {
        return fallbackValue;
      }
      
      throw error;
    }
  }
  
  async get(path, params, fallback = null) {
    return this.execute(
      () => this.withRetry(() => super.get(path, params), `GET ${path}`),
      fallback
    );
  }
  
  async post(path, data) {
    return this.execute(
      () => this.withRetry(() => super.post(path, data), `POST ${path}`)
    );
  }
  
  getCircuitStats() {
    return this.circuitBreaker.getStats();
  }
}

module.exports = { ResilientServiceClient, CircuitBreaker };
```

### 2.4 Service Clients

```javascript
// src/clients/user-service.client.js
const { ResilientServiceClient } = require('./circuit-breaker.client');
const config = require('../config');
const logger = require('../utils/logger');

class UserServiceClient extends ResilientServiceClient {
  constructor() {
    super('user-service', config.services.userServiceUrl, {
      timeout: 5000,
      maxRetries: 3,
      failureThreshold: 5,
      circuitTimeout: 30000,
      
      // Fallback function - return degraded response
      fallback: (error) => {
        logger.warn('User service unavailable, using fallback');
        return null;  // caller handles null gracefully
      },
    });
  }
  
  async getUser(userId) {
    try {
      const response = await this.get(`/users/${userId}`);
      return response?.data;
    } catch (error) {
      if (error.response?.status === 404) return null;
      throw error;
    }
  }
  
  async validateUser(userId) {
    const user = await this.getUser(userId);
    return user !== null;
  }
  
  async getUsers(params = {}) {
    const response = await this.get('/users', params, { data: [], pagination: {} });
    return response;
  }
}

// Singleton
module.exports = new UserServiceClient();
```

---

## 3. Service Discovery

### 3.1 Static Discovery (ง่ายสุด - Development)

```javascript
// config/services.js
module.exports = {
  userServiceUrl: process.env.USER_SERVICE_URL || 'http://user-service:3001',
  orderServiceUrl: process.env.ORDER_SERVICE_URL || 'http://order-service:3002',
  productServiceUrl: process.env.PRODUCT_SERVICE_URL || 'http://product-service:3003',
  paymentServiceUrl: process.env.PAYMENT_SERVICE_URL || 'http://payment-service:3004',
};
```

### 3.2 DNS-based Discovery (Docker/Kubernetes)

```yaml
# In Docker Compose, services resolve by name
services:
  order-service:
    environment:
      # Docker DNS resolves "user-service" to container IP
      - USER_SERVICE_URL=http://user-service:3001

# In Kubernetes, services resolve by:
# <service-name>.<namespace>.svc.cluster.local
# Example: user-service.default.svc.cluster.local
```

### 3.3 Health-aware Service Discovery

```javascript
// src/discovery/service-registry.js
const axios = require('axios');
const logger = require('../utils/logger');

class ServiceRegistry {
  constructor() {
    this.services = new Map();
    this.healthCheckInterval = 30000; // 30 seconds
  }
  
  register(name, instances) {
    this.services.set(name, instances.map(url => ({
      url,
      healthy: true,
      lastCheck: null,
      consecutiveFailures: 0,
    })));
    
    // Start health checking
    this.startHealthChecks(name);
  }
  
  getHealthyInstance(name) {
    const instances = this.services.get(name);
    if (!instances) throw new Error(`Service ${name} not registered`);
    
    const healthy = instances.filter(i => i.healthy);
    if (healthy.length === 0) {
      throw new Error(`No healthy instances for ${name}`);
    }
    
    // Simple round-robin
    const index = Math.floor(Math.random() * healthy.length);
    return healthy[index].url;
  }
  
  async startHealthChecks(serviceName) {
    setInterval(async () => {
      const instances = this.services.get(serviceName);
      
      for (const instance of instances) {
        try {
          await axios.get(`${instance.url}/health`, { timeout: 3000 });
          
          if (!instance.healthy) {
            logger.info(`${serviceName} ${instance.url} recovered`);
          }
          
          instance.healthy = true;
          instance.consecutiveFailures = 0;
          instance.lastCheck = new Date();
          
        } catch (error) {
          instance.consecutiveFailures++;
          
          if (instance.consecutiveFailures >= 3) {
            if (instance.healthy) {
              logger.warn(`${serviceName} ${instance.url} marked unhealthy`);
            }
            instance.healthy = false;
          }
          
          instance.lastCheck = new Date();
        }
      }
    }, this.healthCheckInterval);
  }
  
  getServiceStats() {
    const stats = {};
    
    for (const [name, instances] of this.services) {
      stats[name] = instances.map(i => ({
        url: i.url,
        healthy: i.healthy,
        lastCheck: i.lastCheck,
        consecutiveFailures: i.consecutiveFailures,
      }));
    }
    
    return stats;
  }
}

const registry = new ServiceRegistry();

// Register services
registry.register('user-service', [
  process.env.USER_SERVICE_URL_1 || 'http://user-service-1:3001',
  process.env.USER_SERVICE_URL_2 || 'http://user-service-2:3001',
]);

module.exports = registry;
```

---

## 4. Timeout Management

```javascript
// src/utils/timeout.js

// Wraps any promise with a timeout
function withTimeout(promise, timeoutMs, operation = 'operation') {
  const timeoutPromise = new Promise((_, reject) => {
    setTimeout(() => {
      reject(new Error(`${operation} timed out after ${timeoutMs}ms`));
    }, timeoutMs);
  });
  
  return Promise.race([promise, timeoutPromise]);
}

// Example usage
async function createOrderWithTimeout(orderData) {
  const [user, product] = await Promise.all([
    withTimeout(
      userClient.getUser(orderData.userId), 
      3000,  // 3 second timeout
      'getUser'
    ),
    withTimeout(
      productClient.getProduct(orderData.productId),
      3000,
      'getProduct'
    ),
  ]);
  
  if (!user) throw new Error('User not found');
  if (!product) throw new Error('Product not found');
  
  return createOrder({ user, product, ...orderData });
}
```

---

## 5. Bulkhead Pattern

```javascript
// src/utils/bulkhead.js
// แยก resources สำหรับแต่ละ external service
// ป้องกัน one slow service จากการ block ทุก requests

class Bulkhead {
  constructor(name, options = {}) {
    this.name = name;
    this.maxConcurrent = options.maxConcurrent || 10;
    this.maxQueue = options.maxQueue || 20;
    
    this.activeRequests = 0;
    this.queue = [];
  }
  
  async execute(operation) {
    // ถ้า Active requests เต็ม
    if (this.activeRequests >= this.maxConcurrent) {
      // ถ้า Queue เต็มด้วย
      if (this.queue.length >= this.maxQueue) {
        throw new Error(`Bulkhead ${this.name} queue full`);
      }
      
      // เพิ่มเข้า Queue
      return new Promise((resolve, reject) => {
        this.queue.push({ operation, resolve, reject });
      });
    }
    
    return this.run(operation);
  }
  
  async run(operation) {
    this.activeRequests++;
    
    try {
      return await operation();
    } finally {
      this.activeRequests--;
      
      // Process next in queue
      if (this.queue.length > 0) {
        const { operation, resolve, reject } = this.queue.shift();
        this.run(operation).then(resolve).catch(reject);
      }
    }
  }
  
  getStats() {
    return {
      name: this.name,
      activeRequests: this.activeRequests,
      queueLength: this.queue.length,
      maxConcurrent: this.maxConcurrent,
      maxQueue: this.maxQueue,
    };
  }
}

// ใช้งาน
const userServiceBulkhead = new Bulkhead('user-service', { maxConcurrent: 10 });
const paymentServiceBulkhead = new Bulkhead('payment-service', { maxConcurrent: 5 });

async function createOrder(data) {
  // ใช้ bulkhead สำหรับ payment (จำกัด concurrent calls)
  const payment = await paymentServiceBulkhead.execute(
    () => paymentClient.processPayment(data)
  );
  
  return payment;
}
```

---

## 6. Event-Driven Communication พื้นฐาน

### 6.1 Event Bus แบบง่าย (In-Process)

```javascript
// src/events/event-bus.js
const EventEmitter = require('events');
const logger = require('../utils/logger');

class EventBus extends EventEmitter {
  constructor() {
    super();
    this.setMaxListeners(50);
  }
  
  publish(eventName, data) {
    logger.debug(`Event published: ${eventName}`, { data });
    this.emit(eventName, {
      eventName,
      data,
      timestamp: new Date().toISOString(),
      source: process.env.SERVICE_NAME,
    });
  }
  
  subscribe(eventName, handler) {
    this.on(eventName, async (event) => {
      try {
        await handler(event);
      } catch (error) {
        logger.error(`Error handling event ${eventName}`, { error: error.message });
      }
    });
    
    logger.debug(`Subscribed to event: ${eventName}`);
  }
}

const eventBus = new EventBus();

// Events
const Events = {
  ORDER_CREATED: 'order.created',
  ORDER_CONFIRMED: 'order.confirmed',
  ORDER_CANCELLED: 'order.cancelled',
  PAYMENT_COMPLETED: 'payment.completed',
  PAYMENT_FAILED: 'payment.failed',
  USER_REGISTERED: 'user.registered',
};

module.exports = { eventBus, Events };
```

### 6.2 Event-Driven Order Flow

```javascript
// services/order.service.js
const { eventBus, Events } = require('../events/event-bus');

class OrderService {
  async createOrder(data) {
    const order = await ordersRepository.create(data);
    
    // Publish event - ไม่รอ response
    eventBus.publish(Events.ORDER_CREATED, {
      orderId: order.id,
      userId: order.userId,
      totalAmount: order.totalAmount,
    });
    
    return order;
  }
  
  async confirmOrder(orderId) {
    const order = await ordersRepository.updateStatus(orderId, 'confirmed');
    
    eventBus.publish(Events.ORDER_CONFIRMED, {
      orderId: order.id,
      userId: order.userId,
    });
    
    return order;
  }
}

// services/notification.service.js
const { eventBus, Events } = require('../events/event-bus');

// Subscribe to events
eventBus.subscribe(Events.ORDER_CREATED, async (event) => {
  const { orderId, userId } = event.data;
  
  const user = await userClient.getUser(userId);
  await emailService.sendOrderConfirmation(user.email, orderId);
  
  logger.info('Order confirmation email sent', { orderId, userId });
});

eventBus.subscribe(Events.PAYMENT_FAILED, async (event) => {
  const { orderId, userId, reason } = event.data;
  
  const user = await userClient.getUser(userId);
  await emailService.sendPaymentFailed(user.email, { orderId, reason });
});
```

---

## 7. Complete Workshop

### 7.1 Resilient Order Service

```javascript
// services/order.service.js - Production Grade
const userServiceClient = require('../clients/user-service.client');
const productServiceClient = require('../clients/product-service.client');
const paymentServiceClient = require('../clients/payment-service.client');
const ordersRepository = require('../repositories/orders.repository');
const { eventBus, Events } = require('../events/event-bus');
const { NotFoundError, AppError } = require('../utils/AppError');
const logger = require('../utils/logger');

class OrderService {
  async createOrder({ userId, items }) {
    // Step 1: Validate user (with circuit breaker + retry)
    const user = await userServiceClient.getUser(userId);
    
    if (!user) {
      throw new NotFoundError('User');
    }
    
    // Step 2: Validate and price items (parallel with timeout)
    const productPromises = items.map(item => 
      productServiceClient.getProduct(item.productId)
        .then(product => {
          if (!product) throw new NotFoundError(`Product ${item.productId}`);
          if (product.stock < item.quantity) {
            throw new AppError(
              `Insufficient stock for product ${item.productId}`,
              400,
              'INSUFFICIENT_STOCK'
            );
          }
          return { ...item, product, subtotal: product.price * item.quantity };
        })
    );
    
    const enrichedItems = await Promise.all(productPromises);
    const totalAmount = enrichedItems.reduce((sum, item) => sum + item.subtotal, 0);
    
    // Step 3: Create order (in DB)
    const order = await ordersRepository.create({
      userId,
      items: enrichedItems.map(item => ({
        productId: item.productId,
        productName: item.product.name,
        quantity: item.quantity,
        price: item.product.price,
        subtotal: item.subtotal,
      })),
      totalAmount,
      status: 'pending',
    });
    
    logger.info('Order created', { orderId: order.id, userId, totalAmount });
    
    // Step 4: Publish event (async - ไม่ block)
    eventBus.publish(Events.ORDER_CREATED, {
      orderId: order.id,
      userId,
      items: order.items,
      totalAmount,
    });
    
    return order;
  }
  
  async processPayment(orderId, paymentData) {
    const order = await this.getOrderById(orderId);
    
    if (order.status !== 'pending') {
      throw new AppError('Order is not in pending state', 400, 'INVALID_STATE');
    }
    
    // Process payment via payment service
    let payment;
    try {
      payment = await paymentServiceClient.processPayment({
        orderId,
        userId: order.userId,
        amount: order.totalAmount,
        ...paymentData,
      });
      
    } catch (error) {
      // Payment failed - update order
      await ordersRepository.updateStatus(orderId, 'payment_failed');
      
      eventBus.publish(Events.PAYMENT_FAILED, {
        orderId,
        userId: order.userId,
        reason: error.message,
      });
      
      throw error;
    }
    
    // Payment successful
    await ordersRepository.updateStatus(orderId, 'confirmed', {
      paymentId: payment.id,
    });
    
    // Reserve inventory
    const reservations = order.items.map(item =>
      productServiceClient.reserveStock(item.productId, item.quantity)
        .catch(err => logger.warn('Stock reservation failed', { 
          productId: item.productId, error: err.message 
        }))
    );
    
    await Promise.allSettled(reservations);  // Don't fail if reservation fails
    
    eventBus.publish(Events.PAYMENT_COMPLETED, {
      orderId,
      userId: order.userId,
      paymentId: payment.id,
      amount: order.totalAmount,
    });
    
    return ordersRepository.findById(orderId);
  }
}

module.exports = new OrderService();
```

### 7.2 Health Status with Dependencies

```javascript
// routes/health.routes.js
const router = require('express').Router();
const userServiceClient = require('../clients/user-service.client');
const productServiceClient = require('../clients/product-service.client');
const { CircuitBreaker } = require('../clients/circuit-breaker.client');

router.get('/health', async (req, res) => {
  const health = {
    service: 'order-service',
    status: 'healthy',
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
    checks: {},
    circuitBreakers: {},
  };
  
  // Check database
  try {
    const start = Date.now();
    await require('../utils/database').query('SELECT 1');
    health.checks.database = { status: 'healthy', latency: `${Date.now() - start}ms` };
  } catch (error) {
    health.checks.database = { status: 'unhealthy', error: error.message };
  }
  
  // Check circuit breakers
  health.circuitBreakers = {
    userService: userServiceClient.getCircuitStats(),
    productService: productServiceClient.getCircuitStats(),
  };
  
  const isHealthy = Object.values(health.checks).every(c => c.status === 'healthy');
  health.status = isHealthy ? 'healthy' : 'degraded';
  
  res.status(isHealthy ? 200 : 503).json(health);
});

module.exports = router;
```

---

## 8. Testing Service Communication

```javascript
// tests/integration/service-communication.test.js
const nock = require('nock');  // HTTP interceptor
const orderService = require('../../src/services/order.service');

describe('Order Service - External Calls', () => {
  afterEach(() => {
    nock.cleanAll();
  });
  
  describe('createOrder', () => {
    it('should create order when all services available', async () => {
      // Mock User Service
      nock('http://user-service:3001')
        .get('/users/1')
        .reply(200, { data: { id: 1, name: 'Test User' } });
      
      // Mock Product Service
      nock('http://product-service:3003')
        .get('/products/1')
        .reply(200, { data: { id: 1, name: 'Test Product', price: 100, stock: 10 } });
      
      const order = await orderService.createOrder({
        userId: 1,
        items: [{ productId: 1, quantity: 1 }],
      });
      
      expect(order.status).toBe('pending');
      expect(order.totalAmount).toBe(100);
    });
    
    it('should throw when user service is down (after retries)', async () => {
      // Mock User Service failure
      nock('http://user-service:3001')
        .get('/users/1')
        .times(3)  // Fails 3 times (matching retry count)
        .reply(503);
      
      await expect(orderService.createOrder({
        userId: 1,
        items: [{ productId: 1, quantity: 1 }],
      })).rejects.toThrow();
    });
    
    it('should use fallback when user service circuit is open', async () => {
      // Simulate circuit open
      const client = require('../../src/clients/user-service.client');
      jest.spyOn(client, 'getUser').mockResolvedValue(null);  // fallback returns null
      
      await expect(orderService.createOrder({
        userId: 1,
        items: [{ productId: 1, quantity: 1 }],
      })).rejects.toThrow('User not found');
    });
  });
});
```

---

## 9. สรุป

### Communication Pattern Checklist

```
Synchronous HTTP:
✅ Timeout สำหรับทุก HTTP call
✅ Retry ด้วย exponential backoff
✅ Circuit Breaker สำหรับ dependencies
✅ Bulkhead เพื่อ isolate resource pools
✅ Health checks ที่ตรวจ dependencies

Event-Driven:
✅ Events เป็น past tense (OrderCreated, PaymentCompleted)
✅ Events มี schema ชัดเจน
✅ Idempotent event handlers
✅ Dead Letter Queue สำหรับ failed events

General:
✅ Structured logging ทุก service call
✅ Distributed tracing (X-Request-ID)
✅ Graceful degradation (fallbacks)
✅ Monitor circuit breaker states
```

---

**ต่อไป:** [Part 10 - API Gateway Pattern →](part-10.md)

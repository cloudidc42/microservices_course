# Part 10: API Gateway Pattern

## บทนำ

API Gateway คือ "ประตูหน้า" ของระบบ Microservices ทั้งหมด ทุก request จาก client จะผ่าน Gateway ก่อนที่จะไปถึง service ใด ๆ

---

## 10.1 ทำไมต้องมี API Gateway?

### ปัญหาที่เกิดขึ้นหากไม่มี Gateway

```
Client App
    ├── GET /users    → User Service:3001
    ├── GET /products → Product Service:3002
    ├── POST /orders  → Order Service:3003
    ├── GET /payments → Payment Service:3004
    └── GET /reviews  → Review Service:3005
```

**ปัญหา:**
- Client ต้องรู้ address ของทุก service
- Authentication ต้องทำซ้ำในทุก service
- Rate limiting ต้องทำซ้ำในทุก service
- CORS, SSL termination ต้องทำซ้ำ
- Cross-cutting concerns กระจัดกระจาย

### API Gateway แก้ปัญหาอะไร?

```
Client App
    └── All requests → API Gateway:3000
                            ├── Auth/Authorization
                            ├── Rate Limiting
                            ├── SSL Termination
                            ├── Load Balancing
                            ├── Request Routing
                            ├── Response Aggregation
                            ├── Caching
                            ├── Logging/Monitoring
                            └── Circuit Breaking
                                    ├── User Service:3001
                                    ├── Product Service:3002
                                    ├── Order Service:3003
                                    ├── Payment Service:3004
                                    └── Review Service:3005
```

---

## 10.2 API Gateway Patterns

### Pattern 1: Simple Reverse Proxy

ส่ง request ต่อไปยัง service เดียว โดยไม่ทำอะไรเพิ่มเติม

```
GET /api/users → User Service
POST /api/orders → Order Service
```

### Pattern 2: Request Aggregation (BFF - Backend for Frontend)

รวม response จากหลาย service เป็น response เดียว

```
GET /api/dashboard
  → User Service (get user profile)
  → Order Service (get recent orders)
  → Product Service (get recommended products)
  → ผสมรวมและ return เป็น response เดียว
```

### Pattern 3: Protocol Translation

แปลง protocol ระหว่าง client กับ backend

```
REST (HTTP/1.1) → gRPC (HTTP/2)
WebSocket → Server-Sent Events
GraphQL → REST
```

### Pattern 4: BFF (Backend for Frontend)

สร้าง Gateway แยกต่างหากสำหรับแต่ละ client type

```
Mobile App → Mobile BFF → Services
Web App    → Web BFF    → Services
3rd Party  → Public API → Services
```

---

## 10.3 สร้าง API Gateway ด้วย Node.js

### Project Structure

```
api-gateway/
├── src/
│   ├── config/
│   │   └── index.js
│   ├── middleware/
│   │   ├── auth.js
│   │   ├── rateLimit.js
│   │   ├── logging.js
│   │   ├── errorHandler.js
│   │   └── circuitBreaker.js
│   ├── routes/
│   │   ├── index.js
│   │   ├── users.js
│   │   ├── products.js
│   │   ├── orders.js
│   │   └── aggregator.js
│   ├── services/
│   │   ├── proxyService.js
│   │   └── aggregatorService.js
│   ├── utils/
│   │   ├── logger.js
│   │   └── circuitBreaker.js
│   └── app.js
├── package.json
└── Dockerfile
```

### package.json

```json
{
  "name": "api-gateway",
  "version": "1.0.0",
  "description": "API Gateway for Microservices",
  "main": "src/app.js",
  "scripts": {
    "start": "node src/app.js",
    "dev": "nodemon src/app.js",
    "test": "jest --coverage"
  },
  "dependencies": {
    "express": "^4.18.2",
    "http-proxy-middleware": "^2.0.6",
    "axios": "^1.6.0",
    "express-rate-limit": "^7.1.5",
    "rate-limit-redis": "^4.2.0",
    "ioredis": "^5.3.2",
    "jsonwebtoken": "^9.0.2",
    "morgan": "^1.10.0",
    "winston": "^3.11.0",
    "helmet": "^7.1.0",
    "cors": "^2.8.5",
    "compression": "^1.7.4",
    "express-async-errors": "^3.1.1",
    "uuid": "^9.0.1",
    "opossum": "^7.1.0"
  },
  "devDependencies": {
    "nodemon": "^3.0.2",
    "jest": "^29.7.0",
    "supertest": "^6.3.3",
    "nock": "^13.4.0"
  }
}
```

### src/config/index.js

```javascript
require('dotenv').config();

module.exports = {
  port: process.env.PORT || 3000,
  env: process.env.NODE_ENV || 'development',

  // JWT Config
  jwt: {
    secret: process.env.JWT_SECRET || 'your-super-secret-key',
    expiresIn: process.env.JWT_EXPIRES_IN || '24h',
  },

  // Service URLs
  services: {
    user: process.env.USER_SERVICE_URL || 'http://user-service:3001',
    product: process.env.PRODUCT_SERVICE_URL || 'http://product-service:3002',
    order: process.env.ORDER_SERVICE_URL || 'http://order-service:3003',
    payment: process.env.PAYMENT_SERVICE_URL || 'http://payment-service:3004',
    notification: process.env.NOTIFICATION_SERVICE_URL || 'http://notification-service:3005',
  },

  // Redis Config
  redis: {
    host: process.env.REDIS_HOST || 'redis',
    port: parseInt(process.env.REDIS_PORT) || 6379,
    password: process.env.REDIS_PASSWORD || '',
  },

  // Rate Limiting
  rateLimit: {
    windowMs: parseInt(process.env.RATE_LIMIT_WINDOW_MS) || 15 * 60 * 1000, // 15 minutes
    max: parseInt(process.env.RATE_LIMIT_MAX) || 100, // max requests per window
    skipSuccessfulRequests: false,
  },

  // Circuit Breaker
  circuitBreaker: {
    timeout: parseInt(process.env.CB_TIMEOUT) || 3000,
    errorThresholdPercentage: parseInt(process.env.CB_ERROR_THRESHOLD) || 50,
    resetTimeout: parseInt(process.env.CB_RESET_TIMEOUT) || 30000,
    volumeThreshold: parseInt(process.env.CB_VOLUME_THRESHOLD) || 10,
  },

  // CORS
  cors: {
    origins: (process.env.CORS_ORIGINS || 'http://localhost:3000').split(','),
    methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS'],
    allowedHeaders: ['Content-Type', 'Authorization', 'X-Request-ID', 'X-User-ID'],
  },

  // Timeouts
  timeout: {
    request: parseInt(process.env.REQUEST_TIMEOUT) || 30000,
    upstream: parseInt(process.env.UPSTREAM_TIMEOUT) || 10000,
  },
};
```

### src/utils/logger.js

```javascript
const winston = require('winston');
const config = require('../config');

const logger = winston.createLogger({
  level: config.env === 'production' ? 'info' : 'debug',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    config.env === 'production'
      ? winston.format.json()
      : winston.format.combine(
          winston.format.colorize(),
          winston.format.printf(({ timestamp, level, message, ...meta }) => {
            const metaStr = Object.keys(meta).length ? JSON.stringify(meta, null, 2) : '';
            return `${timestamp} [${level}] ${message} ${metaStr}`;
          })
        )
  ),
  transports: [
    new winston.transports.Console(),
    ...(config.env === 'production' ? [
      new winston.transports.File({ filename: 'logs/error.log', level: 'error' }),
      new winston.transports.File({ filename: 'logs/combined.log' }),
    ] : []),
  ],
});

module.exports = logger;
```

### src/middleware/logging.js

```javascript
const { v4: uuidv4 } = require('uuid');
const logger = require('../utils/logger');

module.exports = function loggingMiddleware(req, res, next) {
  // Assign request ID
  req.requestId = req.headers['x-request-id'] || uuidv4();
  res.setHeader('X-Request-ID', req.requestId);

  const startTime = Date.now();

  // Log request
  logger.info('Incoming request', {
    requestId: req.requestId,
    method: req.method,
    url: req.url,
    ip: req.ip,
    userAgent: req.headers['user-agent'],
    userId: req.user?.id,
  });

  // Log response
  res.on('finish', () => {
    const duration = Date.now() - startTime;
    const logLevel = res.statusCode >= 400 ? 'warn' : 'info';

    logger[logLevel]('Request completed', {
      requestId: req.requestId,
      method: req.method,
      url: req.url,
      statusCode: res.statusCode,
      duration: `${duration}ms`,
      userId: req.user?.id,
    });
  });

  next();
};
```

### src/middleware/auth.js

```javascript
const jwt = require('jsonwebtoken');
const config = require('../config');
const logger = require('../utils/logger');

// Public paths ที่ไม่ต้องการ authentication
const PUBLIC_PATHS = [
  { method: 'POST', path: '/api/v1/auth/login' },
  { method: 'POST', path: '/api/v1/auth/register' },
  { method: 'POST', path: '/api/v1/auth/refresh' },
  { method: 'GET', path: '/api/v1/products' },
  { method: 'GET', path: /^\/api\/v1\/products\/[^/]+$/ },
  { method: 'GET', path: '/health' },
  { method: 'GET', path: '/metrics' },
];

function isPublicPath(method, path) {
  return PUBLIC_PATHS.some(({ method: m, path: p }) => {
    if (m !== method && m !== '*') return false;
    if (typeof p === 'string') return p === path;
    if (p instanceof RegExp) return p.test(path);
    return false;
  });
}

function extractToken(req) {
  const authHeader = req.headers.authorization;
  if (authHeader && authHeader.startsWith('Bearer ')) {
    return authHeader.slice(7);
  }
  return req.cookies?.token || req.query?.token;
}

module.exports = {
  authenticate: function(req, res, next) {
    // Skip auth for public paths
    if (isPublicPath(req.method, req.path)) {
      return next();
    }

    const token = extractToken(req);
    if (!token) {
      return res.status(401).json({
        success: false,
        error: {
          code: 'UNAUTHORIZED',
          message: 'Authentication token is required',
        },
      });
    }

    try {
      const decoded = jwt.verify(token, config.jwt.secret);
      req.user = decoded;

      // Forward user info to downstream services
      req.headers['x-user-id'] = decoded.id;
      req.headers['x-user-role'] = decoded.role;
      req.headers['x-user-email'] = decoded.email;

      next();
    } catch (error) {
      logger.warn('JWT verification failed', {
        requestId: req.requestId,
        error: error.message,
      });

      if (error.name === 'TokenExpiredError') {
        return res.status(401).json({
          success: false,
          error: {
            code: 'TOKEN_EXPIRED',
            message: 'Token has expired',
          },
        });
      }

      return res.status(401).json({
        success: false,
        error: {
          code: 'INVALID_TOKEN',
          message: 'Invalid authentication token',
        },
      });
    }
  },

  authorize: function(...allowedRoles) {
    return function(req, res, next) {
      if (!req.user) {
        return res.status(401).json({
          success: false,
          error: { code: 'UNAUTHORIZED', message: 'Authentication required' },
        });
      }

      if (!allowedRoles.includes(req.user.role)) {
        logger.warn('Authorization failed', {
          requestId: req.requestId,
          userId: req.user.id,
          userRole: req.user.role,
          requiredRoles: allowedRoles,
        });

        return res.status(403).json({
          success: false,
          error: {
            code: 'FORBIDDEN',
            message: 'Insufficient permissions',
          },
        });
      }

      next();
    };
  },
};
```

### src/middleware/rateLimit.js

```javascript
const rateLimit = require('express-rate-limit');
const RedisStore = require('rate-limit-redis').default;
const Redis = require('ioredis');
const config = require('../config');
const logger = require('../utils/logger');

// Create Redis client
const redisClient = new Redis({
  host: config.redis.host,
  port: config.redis.port,
  password: config.redis.password || undefined,
  enableOfflineQueue: false,
});

redisClient.on('error', (err) => {
  logger.error('Redis rate limit error', { error: err.message });
});

// General rate limiter
const generalLimiter = rateLimit({
  windowMs: config.rateLimit.windowMs,
  max: config.rateLimit.max,
  standardHeaders: true,
  legacyHeaders: false,
  store: new RedisStore({
    sendCommand: (...args) => redisClient.call(...args),
    prefix: 'rl:general:',
  }),
  keyGenerator: (req) => {
    return req.user?.id || req.ip;
  },
  handler: (req, res) => {
    logger.warn('Rate limit exceeded', {
      requestId: req.requestId,
      ip: req.ip,
      userId: req.user?.id,
    });
    res.status(429).json({
      success: false,
      error: {
        code: 'RATE_LIMIT_EXCEEDED',
        message: 'Too many requests, please try again later',
        retryAfter: Math.ceil(config.rateLimit.windowMs / 1000),
      },
    });
  },
});

// Strict limiter for auth endpoints
const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 10, // 10 attempts per 15 minutes
  standardHeaders: true,
  legacyHeaders: false,
  store: new RedisStore({
    sendCommand: (...args) => redisClient.call(...args),
    prefix: 'rl:auth:',
  }),
  keyGenerator: (req) => req.ip,
  skipSuccessfulRequests: true,
  handler: (req, res) => {
    res.status(429).json({
      success: false,
      error: {
        code: 'AUTH_RATE_LIMIT_EXCEEDED',
        message: 'Too many authentication attempts. Please try again after 15 minutes.',
      },
    });
  },
});

// Upload/heavy endpoints
const heavyLimiter = rateLimit({
  windowMs: 60 * 60 * 1000, // 1 hour
  max: 50,
  standardHeaders: true,
  store: new RedisStore({
    sendCommand: (...args) => redisClient.call(...args),
    prefix: 'rl:heavy:',
  }),
  keyGenerator: (req) => req.user?.id || req.ip,
});

module.exports = {
  general: generalLimiter,
  auth: authLimiter,
  heavy: heavyLimiter,
};
```

### src/middleware/circuitBreaker.js

```javascript
const CircuitBreaker = require('opossum');
const logger = require('../utils/logger');
const config = require('../config');

const breakers = new Map();

function getBreaker(serviceName, fn) {
  if (!breakers.has(serviceName)) {
    const breaker = new CircuitBreaker(fn, {
      timeout: config.circuitBreaker.timeout,
      errorThresholdPercentage: config.circuitBreaker.errorThresholdPercentage,
      resetTimeout: config.circuitBreaker.resetTimeout,
      volumeThreshold: config.circuitBreaker.volumeThreshold,
      name: serviceName,
    });

    breaker.on('open', () => {
      logger.error(`Circuit OPEN for ${serviceName}`);
    });

    breaker.on('halfOpen', () => {
      logger.warn(`Circuit HALF-OPEN for ${serviceName}`);
    });

    breaker.on('close', () => {
      logger.info(`Circuit CLOSED for ${serviceName}`);
    });

    breaker.on('fallback', (result) => {
      logger.warn(`Circuit fallback triggered for ${serviceName}`, { result });
    });

    breakers.set(serviceName, breaker);
  }

  return breakers.get(serviceName);
}

function getBreakerStats() {
  const stats = {};
  for (const [name, breaker] of breakers.entries()) {
    stats[name] = {
      state: breaker.opened ? 'OPEN' : breaker.halfOpen ? 'HALF_OPEN' : 'CLOSED',
      stats: breaker.stats,
    };
  }
  return stats;
}

module.exports = { getBreaker, getBreakerStats };
```

### src/services/proxyService.js

```javascript
const axios = require('axios');
const config = require('../config');
const logger = require('../utils/logger');
const { getBreaker } = require('../middleware/circuitBreaker');

class ProxyService {
  constructor() {
    this.client = axios.create({
      timeout: config.timeout.upstream,
    });

    // Request interceptor - add request ID
    this.client.interceptors.request.use((axiosConfig) => {
      if (!axiosConfig.headers['x-request-id']) {
        axiosConfig.headers['x-request-id'] = axiosConfig.requestId || 'unknown';
      }
      return axiosConfig;
    });

    // Response interceptor - log timing
    this.client.interceptors.response.use(
      (response) => {
        logger.debug('Upstream response', {
          service: response.config.url,
          status: response.status,
        });
        return response;
      },
      (error) => {
        logger.error('Upstream error', {
          url: error.config?.url,
          status: error.response?.status,
          message: error.message,
        });
        return Promise.reject(error);
      }
    );
  }

  async forward(req, res, targetUrl) {
    const serviceName = new URL(targetUrl).hostname;

    // Create circuit breaker wrapped function
    const makeRequest = async (options) => {
      const response = await this.client({
        method: options.method,
        url: options.url,
        data: options.data,
        headers: options.headers,
        params: options.params,
      });
      return response;
    };

    const breaker = getBreaker(serviceName, makeRequest);

    // Fallback response when circuit is open
    breaker.fallback(() => ({
      data: {
        success: false,
        error: {
          code: 'SERVICE_UNAVAILABLE',
          message: `${serviceName} service is temporarily unavailable`,
        },
      },
      status: 503,
    }));

    try {
      const response = await breaker.fire({
        method: req.method,
        url: targetUrl,
        data: req.body,
        headers: this.buildForwardHeaders(req),
        params: req.query,
      });

      // Remove internal headers before sending to client
      const headers = { ...response.headers };
      delete headers['x-powered-by'];
      delete headers['server'];

      // Set response headers
      Object.entries(headers).forEach(([key, value]) => {
        if (!['content-encoding', 'transfer-encoding', 'connection'].includes(key)) {
          res.setHeader(key, value);
        }
      });

      res.status(response.status).json(response.data);
    } catch (error) {
      const status = error.response?.status || 502;
      const data = error.response?.data || {
        success: false,
        error: {
          code: 'BAD_GATEWAY',
          message: 'Failed to communicate with upstream service',
        },
      };

      res.status(status).json(data);
    }
  }

  buildForwardHeaders(req) {
    return {
      'content-type': req.headers['content-type'] || 'application/json',
      'x-request-id': req.requestId,
      'x-forwarded-for': req.ip,
      'x-forwarded-host': req.hostname,
      'x-forwarded-proto': req.protocol,
      'x-user-id': req.headers['x-user-id'] || '',
      'x-user-role': req.headers['x-user-role'] || '',
      'x-user-email': req.headers['x-user-email'] || '',
      'accept': req.headers['accept'] || 'application/json',
    };
  }

  async get(url, headers = {}) {
    const response = await this.client.get(url, { headers });
    return response.data;
  }

  async post(url, data, headers = {}) {
    const response = await this.client.post(url, data, { headers });
    return response.data;
  }
}

module.exports = new ProxyService();
```

### src/services/aggregatorService.js

```javascript
const proxyService = require('./proxyService');
const config = require('../config');
const logger = require('../utils/logger');

class AggregatorService {
  // Dashboard: aggregate user profile + recent orders + recommendations
  async getDashboard(userId, requestId) {
    const headers = {
      'x-user-id': userId,
      'x-request-id': requestId,
    };

    // Execute all requests in parallel
    const [userResult, ordersResult, recommendationsResult] = await Promise.allSettled([
      proxyService.get(`${config.services.user}/api/v1/users/${userId}`, headers),
      proxyService.get(`${config.services.order}/api/v1/orders?userId=${userId}&limit=5&sort=-createdAt`, headers),
      proxyService.get(`${config.services.product}/api/v1/products/recommendations?userId=${userId}&limit=6`, headers),
    ]);

    return {
      user: userResult.status === 'fulfilled' ? userResult.value?.data : null,
      recentOrders: ordersResult.status === 'fulfilled' ? ordersResult.value?.data : [],
      recommendations: recommendationsResult.status === 'fulfilled' ? recommendationsResult.value?.data : [],
      _meta: {
        userFetched: userResult.status === 'fulfilled',
        ordersFetched: ordersResult.status === 'fulfilled',
        recommendationsFetched: recommendationsResult.status === 'fulfilled',
      },
    };
  }

  // Product detail: product info + reviews + stock
  async getProductDetail(productId, userId, requestId) {
    const headers = {
      'x-user-id': userId || '',
      'x-request-id': requestId,
    };

    const [productResult, reviewsResult, stockResult] = await Promise.allSettled([
      proxyService.get(`${config.services.product}/api/v1/products/${productId}`, headers),
      proxyService.get(`${config.services.product}/api/v1/products/${productId}/reviews?limit=10`, headers),
      proxyService.get(`${config.services.product}/api/v1/inventory/${productId}`, headers),
    ]);

    if (productResult.status === 'rejected') {
      throw new Error(`Product ${productId} not found`);
    }

    return {
      ...productResult.value?.data,
      reviews: reviewsResult.status === 'fulfilled' ? reviewsResult.value?.data : [],
      stock: stockResult.status === 'fulfilled' ? stockResult.value?.data : null,
    };
  }

  // Order with full details: order + user + products
  async getOrderDetail(orderId, userId, requestId) {
    const headers = {
      'x-user-id': userId,
      'x-request-id': requestId,
    };

    // Get order first
    const orderData = await proxyService.get(
      `${config.services.order}/api/v1/orders/${orderId}`,
      headers
    );

    const order = orderData?.data;
    if (!order) throw new Error('Order not found');

    // Then fetch related data in parallel
    const productIds = order.items?.map(item => item.productId) || [];
    const [userResult, productsResult] = await Promise.allSettled([
      proxyService.get(`${config.services.user}/api/v1/users/${order.userId}`, headers),
      ...productIds.map(id =>
        proxyService.get(`${config.services.product}/api/v1/products/${id}`, headers)
      ),
    ]);

    const productMap = {};
    productsResult.forEach((result, index) => {
      if (result.status === 'fulfilled') {
        productMap[productIds[index]] = result.value?.data;
      }
    });

    return {
      ...order,
      customer: userResult.status === 'fulfilled' ? userResult.value?.data : null,
      items: order.items?.map(item => ({
        ...item,
        product: productMap[item.productId] || null,
      })),
    };
  }
}

module.exports = new AggregatorService();
```

### src/routes/users.js

```javascript
const express = require('express');
const router = express.Router();
const proxyService = require('../services/proxyService');
const { authenticate, authorize } = require('../middleware/auth');
const { auth: authLimiter } = require('../middleware/rateLimit');
const config = require('../config');

const USER_SERVICE = config.services.user;

// Auth routes (public)
router.post('/auth/register', authLimiter, (req, res) => {
  proxyService.forward(req, res, `${USER_SERVICE}/api/v1/auth/register`);
});

router.post('/auth/login', authLimiter, (req, res) => {
  proxyService.forward(req, res, `${USER_SERVICE}/api/v1/auth/login`);
});

router.post('/auth/refresh', authLimiter, (req, res) => {
  proxyService.forward(req, res, `${USER_SERVICE}/api/v1/auth/refresh`);
});

router.post('/auth/logout', authenticate, (req, res) => {
  proxyService.forward(req, res, `${USER_SERVICE}/api/v1/auth/logout`);
});

router.post('/auth/forgot-password', authLimiter, (req, res) => {
  proxyService.forward(req, res, `${USER_SERVICE}/api/v1/auth/forgot-password`);
});

// User profile routes (authenticated)
router.get('/users/me', authenticate, (req, res) => {
  proxyService.forward(req, res, `${USER_SERVICE}/api/v1/users/me`);
});

router.put('/users/me', authenticate, (req, res) => {
  proxyService.forward(req, res, `${USER_SERVICE}/api/v1/users/me`);
});

router.patch('/users/me/password', authenticate, (req, res) => {
  proxyService.forward(req, res, `${USER_SERVICE}/api/v1/users/me/password`);
});

// Admin routes
router.get('/users', authenticate, authorize('admin'), (req, res) => {
  proxyService.forward(req, res, `${USER_SERVICE}/api/v1/users`);
});

router.get('/users/:id', authenticate, (req, res) => {
  proxyService.forward(req, res, `${USER_SERVICE}/api/v1/users/${req.params.id}`);
});

router.put('/users/:id', authenticate, authorize('admin'), (req, res) => {
  proxyService.forward(req, res, `${USER_SERVICE}/api/v1/users/${req.params.id}`);
});

router.delete('/users/:id', authenticate, authorize('admin'), (req, res) => {
  proxyService.forward(req, res, `${USER_SERVICE}/api/v1/users/${req.params.id}`);
});

module.exports = router;
```

### src/routes/products.js

```javascript
const express = require('express');
const router = express.Router();
const proxyService = require('../services/proxyService');
const { authenticate, authorize } = require('../middleware/auth');
const { heavy: heavyLimiter } = require('../middleware/rateLimit');
const config = require('../config');

const PRODUCT_SERVICE = config.services.product;

// Public product routes
router.get('/products', (req, res) => {
  proxyService.forward(req, res, `${PRODUCT_SERVICE}/api/v1/products`);
});

router.get('/products/search', (req, res) => {
  proxyService.forward(req, res, `${PRODUCT_SERVICE}/api/v1/products/search`);
});

router.get('/products/recommendations', (req, res) => {
  proxyService.forward(req, res, `${PRODUCT_SERVICE}/api/v1/products/recommendations`);
});

router.get('/products/:id', (req, res) => {
  proxyService.forward(req, res, `${PRODUCT_SERVICE}/api/v1/products/${req.params.id}`);
});

router.get('/products/:id/reviews', (req, res) => {
  proxyService.forward(req, res, `${PRODUCT_SERVICE}/api/v1/products/${req.params.id}/reviews`);
});

// Category routes
router.get('/categories', (req, res) => {
  proxyService.forward(req, res, `${PRODUCT_SERVICE}/api/v1/categories`);
});

router.get('/categories/:id/products', (req, res) => {
  proxyService.forward(req, res, `${PRODUCT_SERVICE}/api/v1/categories/${req.params.id}/products`);
});

// Authenticated routes
router.post('/products/:id/reviews', authenticate, (req, res) => {
  proxyService.forward(req, res, `${PRODUCT_SERVICE}/api/v1/products/${req.params.id}/reviews`);
});

// Admin routes
router.post('/products', authenticate, authorize('admin', 'seller'), heavyLimiter, (req, res) => {
  proxyService.forward(req, res, `${PRODUCT_SERVICE}/api/v1/products`);
});

router.put('/products/:id', authenticate, authorize('admin', 'seller'), (req, res) => {
  proxyService.forward(req, res, `${PRODUCT_SERVICE}/api/v1/products/${req.params.id}`);
});

router.patch('/products/:id', authenticate, authorize('admin', 'seller'), (req, res) => {
  proxyService.forward(req, res, `${PRODUCT_SERVICE}/api/v1/products/${req.params.id}`);
});

router.delete('/products/:id', authenticate, authorize('admin'), (req, res) => {
  proxyService.forward(req, res, `${PRODUCT_SERVICE}/api/v1/products/${req.params.id}`);
});

module.exports = router;
```

### src/routes/orders.js

```javascript
const express = require('express');
const router = express.Router();
const proxyService = require('../services/proxyService');
const { authenticate, authorize } = require('../middleware/auth');
const config = require('../config');

const ORDER_SERVICE = config.services.order;

// All order routes require authentication
router.use(authenticate);

// User order routes
router.post('/orders', (req, res) => {
  proxyService.forward(req, res, `${ORDER_SERVICE}/api/v1/orders`);
});

router.get('/orders', (req, res) => {
  proxyService.forward(req, res, `${ORDER_SERVICE}/api/v1/orders`);
});

router.get('/orders/:id', (req, res) => {
  proxyService.forward(req, res, `${ORDER_SERVICE}/api/v1/orders/${req.params.id}`);
});

router.post('/orders/:id/cancel', (req, res) => {
  proxyService.forward(req, res, `${ORDER_SERVICE}/api/v1/orders/${req.params.id}/cancel`);
});

router.post('/orders/:id/confirm', (req, res) => {
  proxyService.forward(req, res, `${ORDER_SERVICE}/api/v1/orders/${req.params.id}/confirm`);
});

// Payment routes (sub-resource of orders)
router.post('/orders/:id/payments', (req, res) => {
  proxyService.forward(req, res, `${config.services.payment}/api/v1/orders/${req.params.id}/payments`);
});

router.get('/orders/:id/payments', (req, res) => {
  proxyService.forward(req, res, `${config.services.payment}/api/v1/orders/${req.params.id}/payments`);
});

// Admin routes
router.get('/admin/orders', authorize('admin'), (req, res) => {
  proxyService.forward(req, res, `${ORDER_SERVICE}/api/v1/admin/orders`);
});

router.patch('/admin/orders/:id/status', authorize('admin'), (req, res) => {
  proxyService.forward(req, res, `${ORDER_SERVICE}/api/v1/admin/orders/${req.params.id}/status`);
});

module.exports = router;
```

### src/routes/aggregator.js

```javascript
const express = require('express');
const router = express.Router();
const aggregatorService = require('../services/aggregatorService');
const { authenticate } = require('../middleware/auth');
const logger = require('../utils/logger');

// Dashboard - requires authentication
router.get('/dashboard', authenticate, async (req, res) => {
  try {
    const dashboard = await aggregatorService.getDashboard(req.user.id, req.requestId);

    res.json({
      success: true,
      data: dashboard,
    });
  } catch (error) {
    logger.error('Dashboard aggregation failed', {
      requestId: req.requestId,
      userId: req.user.id,
      error: error.message,
    });

    res.status(502).json({
      success: false,
      error: {
        code: 'AGGREGATION_FAILED',
        message: 'Failed to load dashboard data',
      },
    });
  }
});

// Product detail with reviews and stock
router.get('/products/:id/detail', async (req, res) => {
  try {
    const detail = await aggregatorService.getProductDetail(
      req.params.id,
      req.user?.id,
      req.requestId
    );

    res.json({
      success: true,
      data: detail,
    });
  } catch (error) {
    logger.error('Product detail aggregation failed', {
      requestId: req.requestId,
      productId: req.params.id,
      error: error.message,
    });

    const status = error.message.includes('not found') ? 404 : 502;
    res.status(status).json({
      success: false,
      error: {
        code: status === 404 ? 'NOT_FOUND' : 'AGGREGATION_FAILED',
        message: error.message,
      },
    });
  }
});

// Order detail with customer and product info
router.get('/orders/:id/detail', authenticate, async (req, res) => {
  try {
    const detail = await aggregatorService.getOrderDetail(
      req.params.id,
      req.user.id,
      req.requestId
    );

    res.json({
      success: true,
      data: detail,
    });
  } catch (error) {
    logger.error('Order detail aggregation failed', {
      requestId: req.requestId,
      orderId: req.params.id,
      error: error.message,
    });

    res.status(502).json({
      success: false,
      error: {
        code: 'AGGREGATION_FAILED',
        message: 'Failed to load order detail',
      },
    });
  }
});

module.exports = router;
```

### src/routes/index.js

```javascript
const express = require('express');
const router = express.Router();
const { getBreakerStats } = require('../middleware/circuitBreaker');
const config = require('../config');

// Mount routes
router.use('/api/v1', require('./users'));
router.use('/api/v1', require('./products'));
router.use('/api/v1', require('./orders'));
router.use('/api/v1/aggregate', require('./aggregator'));

// Health check
router.get('/health', (req, res) => {
  res.json({
    success: true,
    status: 'healthy',
    service: 'api-gateway',
    version: '1.0.0',
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
  });
});

// Circuit breaker status
router.get('/health/circuit-breakers', (req, res) => {
  res.json({
    success: true,
    data: getBreakerStats(),
  });
});

// Gateway info (for debugging in dev)
if (config.env !== 'production') {
  router.get('/debug/config', (req, res) => {
    res.json({
      services: Object.entries(config.services).reduce((acc, [key, url]) => {
        acc[key] = url;
        return acc;
      }, {}),
    });
  });
}

module.exports = router;
```

### src/middleware/errorHandler.js

```javascript
const logger = require('../utils/logger');

module.exports = function errorHandler(err, req, res, next) {
  logger.error('Unhandled error', {
    requestId: req.requestId,
    method: req.method,
    url: req.url,
    error: err.message,
    stack: err.stack,
  });

  if (res.headersSent) {
    return next(err);
  }

  const status = err.status || err.statusCode || 500;

  res.status(status).json({
    success: false,
    error: {
      code: err.code || 'INTERNAL_SERVER_ERROR',
      message: process.env.NODE_ENV === 'production'
        ? 'An unexpected error occurred'
        : err.message,
    },
    requestId: req.requestId,
  });
};
```

### src/app.js

```javascript
require('express-async-errors');
const express = require('express');
const helmet = require('helmet');
const cors = require('cors');
const compression = require('compression');
const morgan = require('morgan');
const config = require('./config');
const logger = require('./utils/logger');
const loggingMiddleware = require('./middleware/logging');
const { general: generalLimiter } = require('./middleware/rateLimit');
const errorHandler = require('./middleware/errorHandler');
const routes = require('./routes');

const app = express();

// Security middleware
app.use(helmet({
  contentSecurityPolicy: false, // Configure as needed
  crossOriginEmbedderPolicy: false,
}));

// CORS
app.use(cors({
  origin: (origin, callback) => {
    if (!origin || config.cors.origins.includes(origin) || config.cors.origins.includes('*')) {
      callback(null, true);
    } else {
      callback(new Error(`CORS not allowed from origin: ${origin}`));
    }
  },
  methods: config.cors.methods,
  allowedHeaders: config.cors.allowedHeaders,
  credentials: true,
}));

// Compression
app.use(compression());

// Body parsing
app.use(express.json({ limit: '10mb' }));
app.use(express.urlencoded({ extended: true, limit: '10mb' }));

// Request logging (dev)
if (config.env !== 'production') {
  app.use(morgan('dev'));
}

// Custom logging middleware
app.use(loggingMiddleware);

// Rate limiting
app.use(generalLimiter);

// Trust proxy (important for rate limiting by IP behind load balancer)
app.set('trust proxy', 1);

// Routes
app.use(routes);

// 404 handler
app.use('*', (req, res) => {
  res.status(404).json({
    success: false,
    error: {
      code: 'NOT_FOUND',
      message: `Route ${req.method} ${req.originalUrl} not found`,
    },
    requestId: req.requestId,
  });
});

// Error handler (must be last)
app.use(errorHandler);

// Graceful shutdown
const server = app.listen(config.port, () => {
  logger.info(`API Gateway running`, {
    port: config.port,
    env: config.env,
  });
});

process.on('SIGTERM', () => {
  logger.info('SIGTERM received, starting graceful shutdown');
  server.close(() => {
    logger.info('HTTP server closed');
    process.exit(0);
  });
});

process.on('SIGINT', () => {
  logger.info('SIGINT received, starting graceful shutdown');
  server.close(() => {
    logger.info('HTTP server closed');
    process.exit(0);
  });
});

module.exports = app;
```

---

## 10.4 Nginx เป็น API Gateway

สำหรับ production ระดับสูง nginx เป็นตัวเลือกที่นิยมมาก

### nginx.conf - Advanced

```nginx
# /etc/nginx/nginx.conf
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 4096;
    multi_accept on;
    use epoll;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # Logging format
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for" '
                    'rt=$request_time uct=$upstream_connect_time '
                    'uht=$upstream_header_time urt=$upstream_response_time '
                    'rid=$request_id';

    access_log /var/log/nginx/access.log main;

    # Performance
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;

    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_min_length 10240;
    gzip_proxied expired no-cache no-store private auth;
    gzip_types text/plain text/css text/xml text/javascript
               application/json application/javascript application/xml+rss;
    gzip_disable "MSIE [1-6]\.";

    # Rate limiting zones
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;
    limit_req_zone $binary_remote_addr zone=auth_limit:5m rate=1r/s;
    limit_conn_zone $binary_remote_addr zone=conn_limit:10m;

    # Upstream servers (with health checks)
    upstream user_service {
        least_conn;
        keepalive 32;
        server user-service:3001 max_fails=3 fail_timeout=30s;
        # server user-service-2:3001 max_fails=3 fail_timeout=30s; # additional instance
    }

    upstream product_service {
        least_conn;
        keepalive 32;
        server product-service:3002 max_fails=3 fail_timeout=30s;
    }

    upstream order_service {
        least_conn;
        keepalive 32;
        server order-service:3003 max_fails=3 fail_timeout=30s;
    }

    upstream payment_service {
        server payment-service:3004 max_fails=3 fail_timeout=30s;
    }

    # Common proxy settings
    proxy_http_version 1.1;
    proxy_set_header Connection "";
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Request-ID $request_id;
    proxy_connect_timeout 5s;
    proxy_read_timeout 30s;
    proxy_send_timeout 30s;

    # Main server block
    server {
        listen 80;
        listen [::]:80;
        server_name api.example.com;

        # Redirect HTTP to HTTPS
        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl http2;
        listen [::]:443 ssl http2;
        server_name api.example.com;

        # SSL Configuration
        ssl_certificate /etc/ssl/certs/api.example.com.crt;
        ssl_certificate_key /etc/ssl/private/api.example.com.key;
        ssl_session_timeout 1d;
        ssl_session_cache shared:MozSSL:10m;
        ssl_session_tickets off;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
        ssl_prefer_server_ciphers off;
        add_header Strict-Transport-Security "max-age=63072000" always;

        # Security headers
        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header X-XSS-Protection "1; mode=block" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header Referrer-Policy "strict-origin-when-cross-origin" always;
        add_header Permissions-Policy "accelerometer=(), camera=(), geolocation=(), microphone=()" always;

        # Remove server version info
        server_tokens off;

        # Connection limits
        limit_conn conn_limit 20;

        # Health check (no rate limiting)
        location /health {
            access_log off;
            add_header Content-Type application/json;
            return 200 '{"status":"healthy","service":"nginx-gateway"}';
        }

        # Auth routes - strict rate limiting
        location ~ ^/api/v1/auth/(login|register|forgot-password) {
            limit_req zone=auth_limit burst=5 nodelay;
            limit_req_status 429;

            proxy_pass http://user_service;
        }

        # User service routes
        location /api/v1/auth/ {
            limit_req zone=api_limit burst=20 nodelay;
            proxy_pass http://user_service;
        }

        location /api/v1/users/ {
            limit_req zone=api_limit burst=20 nodelay;
            proxy_pass http://user_service;
        }

        # Product service routes (public, less strict)
        location ~ ^/api/v1/products {
            limit_req zone=api_limit burst=30 nodelay;
            proxy_pass http://product_service;

            # Cache for public product listings
            proxy_cache_valid 200 1m;
        }

        location ~ ^/api/v1/categories {
            limit_req zone=api_limit burst=30 nodelay;
            proxy_pass http://product_service;
        }

        # Order service routes
        location /api/v1/orders/ {
            limit_req zone=api_limit burst=10 nodelay;
            proxy_pass http://order_service;
        }

        # Payment routes - extra strict
        location /api/v1/payments/ {
            limit_req zone=auth_limit burst=3 nodelay;

            # Extra security headers for payments
            add_header Cache-Control "no-store, no-cache, must-revalidate";
            add_header Pragma "no-cache";

            proxy_pass http://payment_service;
        }

        # Default - 404
        location / {
            return 404 '{"success":false,"error":{"code":"NOT_FOUND","message":"Endpoint not found"}}';
            add_header Content-Type application/json;
        }

        # Error pages
        error_page 429 = @rate_limit_error;
        error_page 502 503 504 = @upstream_error;

        location @rate_limit_error {
            add_header Content-Type application/json;
            return 429 '{"success":false,"error":{"code":"RATE_LIMIT_EXCEEDED","message":"Too many requests"}}';
        }

        location @upstream_error {
            add_header Content-Type application/json;
            return 503 '{"success":false,"error":{"code":"SERVICE_UNAVAILABLE","message":"Service temporarily unavailable"}}';
        }
    }
}
```

---

## 10.5 Docker Compose with API Gateway

```yaml
# docker-compose.yml
version: '3.8'

services:
  api-gateway:
    build:
      context: ./api-gateway
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - PORT=3000
      - JWT_SECRET=${JWT_SECRET}
      - USER_SERVICE_URL=http://user-service:3001
      - PRODUCT_SERVICE_URL=http://product-service:3002
      - ORDER_SERVICE_URL=http://order-service:3003
      - PAYMENT_SERVICE_URL=http://payment-service:3004
      - REDIS_HOST=redis
      - REDIS_PORT=6379
    depends_on:
      redis:
        condition: service_healthy
    networks:
      - frontend
      - backend
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  nginx-gateway:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/ssl:ro
    depends_on:
      - user-service
      - product-service
      - order-service
    networks:
      - frontend
    restart: unless-stopped

  user-service:
    build: ./user-service
    environment:
      - NODE_ENV=production
      - PORT=3001
      - DATABASE_URL=postgresql://user:pass@postgres-user:5432/users
      - JWT_SECRET=${JWT_SECRET}
    depends_on:
      postgres-user:
        condition: service_healthy
    networks:
      - backend
    restart: unless-stopped

  product-service:
    build: ./product-service
    environment:
      - NODE_ENV=production
      - PORT=3002
      - DATABASE_URL=mongodb://mongo-product:27017/products
    depends_on:
      mongo-product:
        condition: service_healthy
    networks:
      - backend
    restart: unless-stopped

  order-service:
    build: ./order-service
    environment:
      - NODE_ENV=production
      - PORT=3003
      - DATABASE_URL=postgresql://order:pass@postgres-order:5432/orders
    depends_on:
      postgres-order:
        condition: service_healthy
    networks:
      - backend
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    command: redis-server --save 60 1 --loglevel warning
    volumes:
      - redis-data:/data
    networks:
      - backend
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  postgres-user:
    image: postgres:15-alpine
    environment:
      - POSTGRES_DB=users
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
    volumes:
      - postgres-user-data:/var/lib/postgresql/data
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d users"]
      interval: 10s
      timeout: 5s
      retries: 5

  postgres-order:
    image: postgres:15-alpine
    environment:
      - POSTGRES_DB=orders
      - POSTGRES_USER=order
      - POSTGRES_PASSWORD=pass
    volumes:
      - postgres-order-data:/var/lib/postgresql/data
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U order -d orders"]
      interval: 10s
      timeout: 5s
      retries: 5

  mongo-product:
    image: mongo:7
    environment:
      - MONGO_INITDB_DATABASE=products
    volumes:
      - mongo-product-data:/data/db
    networks:
      - backend
    healthcheck:
      test: echo 'db.runCommand("ping").ok' | mongosh localhost:27017/test --quiet
      interval: 10s
      timeout: 5s
      retries: 5

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true

volumes:
  redis-data:
  postgres-user-data:
  postgres-order-data:
  mongo-product-data:
```

---

## 10.6 Testing API Gateway

### Integration Tests

```javascript
// tests/gateway.test.js
const request = require('supertest');
const nock = require('nock');
const app = require('../src/app');
const jwt = require('jsonwebtoken');
const config = require('../src/config');

describe('API Gateway', () => {
  let userToken;
  let adminToken;

  beforeEach(() => {
    userToken = jwt.sign(
      { id: 'user-1', email: 'user@test.com', role: 'user' },
      config.jwt.secret,
      { expiresIn: '1h' }
    );

    adminToken = jwt.sign(
      { id: 'admin-1', email: 'admin@test.com', role: 'admin' },
      config.jwt.secret,
      { expiresIn: '1h' }
    );
  });

  afterEach(() => {
    nock.cleanAll();
  });

  describe('Authentication', () => {
    it('should reject requests to protected routes without token', async () => {
      const response = await request(app)
        .get('/api/v1/users/me')
        .expect(401);

      expect(response.body.success).toBe(false);
      expect(response.body.error.code).toBe('UNAUTHORIZED');
    });

    it('should reject expired tokens', async () => {
      const expiredToken = jwt.sign(
        { id: 'user-1', role: 'user' },
        config.jwt.secret,
        { expiresIn: '-1s' }
      );

      const response = await request(app)
        .get('/api/v1/users/me')
        .set('Authorization', `Bearer ${expiredToken}`)
        .expect(401);

      expect(response.body.error.code).toBe('TOKEN_EXPIRED');
    });

    it('should allow public routes without token', async () => {
      nock(config.services.product)
        .get('/api/v1/products')
        .reply(200, { success: true, data: [] });

      await request(app)
        .get('/api/v1/products')
        .expect(200);
    });
  });

  describe('Request Proxying', () => {
    it('should forward request to user service with user context headers', async () => {
      nock(config.services.user)
        .get('/api/v1/users/me')
        .reply(function() {
          expect(this.req.headers['x-user-id']).toBe('user-1');
          expect(this.req.headers['x-user-role']).toBe('user');
          return [200, { success: true, data: { id: 'user-1' } }];
        });

      await request(app)
        .get('/api/v1/users/me')
        .set('Authorization', `Bearer ${userToken}`)
        .expect(200);
    });

    it('should add X-Request-ID header to forwarded requests', async () => {
      nock(config.services.product)
        .get('/api/v1/products')
        .reply(function() {
          expect(this.req.headers['x-request-id']).toBeDefined();
          return [200, { success: true, data: [] }];
        });

      await request(app)
        .get('/api/v1/products')
        .expect(200);
    });
  });

  describe('Authorization', () => {
    it('should allow admin to access admin routes', async () => {
      nock(config.services.user)
        .get('/api/v1/users')
        .reply(200, { success: true, data: [] });

      await request(app)
        .get('/api/v1/users')
        .set('Authorization', `Bearer ${adminToken}`)
        .expect(200);
    });

    it('should reject regular user from admin routes', async () => {
      await request(app)
        .get('/api/v1/users')
        .set('Authorization', `Bearer ${userToken}`)
        .expect(403);
    });
  });

  describe('Health Check', () => {
    it('should return healthy status', async () => {
      const response = await request(app)
        .get('/health')
        .expect(200);

      expect(response.body.status).toBe('healthy');
      expect(response.body.service).toBe('api-gateway');
    });
  });

  describe('Circuit Breaker', () => {
    it('should return 503 when circuit is open', async () => {
      // Simulate multiple failures
      nock(config.services.product)
        .get('/api/v1/products')
        .times(15)
        .replyWithError('Connection refused');

      // Make multiple failing requests
      for (let i = 0; i < 15; i++) {
        await request(app).get('/api/v1/products');
      }

      // Circuit should now be open
      const response = await request(app)
        .get('/api/v1/products');

      expect(response.status).toBeGreaterThanOrEqual(500);
    });
  });
});
```

---

## 10.7 API Gateway Best Practices

### Do's ✅

1. **Single Responsibility** - Gateway จัดการ cross-cutting concerns ไม่ใช่ business logic
2. **Stateless** - ไม่เก็บ session state ใน Gateway
3. **Health Checks** - ตรวจสอบ upstream services ก่อน route
4. **Idempotency Keys** - รองรับ safe retry สำหรับ POST requests
5. **Request Tracing** - Request ID ต้องผ่านไปทุก service
6. **Graceful Degradation** - Fallback responses เมื่อ service ล้มเหลว
7. **Cache** - Cache responses ที่เหมาะสมใน Gateway
8. **Monitor** - ติดตาม latency, error rate ของทุก upstream

### Don'ts ❌

1. **No Business Logic** - ไม่ใส่ business rules ใน Gateway
2. **No Database Access** - Gateway ไม่ควร query database โดยตรง
3. **No Tight Coupling** - ไม่ couple กับ internal data models ของ services
4. **No Session Storage** - ไม่เก็บ user sessions ใน Gateway
5. **No Complex Aggregation** - Aggregation ที่ซับซ้อนควรอยู่ใน BFF layer แยก
6. **No Long Computation** - งานหนักควร delegate ไป service

---

## Workshop: สร้าง Full API Gateway

### ขั้นตอนที่ 1: สร้าง Project

```bash
mkdir api-gateway && cd api-gateway
npm init -y
npm install express axios jsonwebtoken express-rate-limit helmet cors compression winston uuid opossum express-async-errors
npm install -D nodemon jest supertest nock
```

### ขั้นตอนที่ 2: สร้าง Directory Structure

```bash
mkdir -p src/{config,middleware,routes,services,utils}
touch src/{app.js,config/index.js}
touch src/middleware/{auth.js,rateLimit.js,logging.js,errorHandler.js,circuitBreaker.js}
touch src/routes/{index.js,users.js,products.js,orders.js,aggregator.js}
touch src/services/{proxyService.js,aggregatorService.js}
touch src/utils/{logger.js}
```

### ขั้นตอนที่ 3: สร้าง Dockerfile

```dockerfile
# Multi-stage build
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:20-alpine AS production
RUN addgroup -g 1001 -S nodejs && adduser -S nodejs -u 1001
WORKDIR /app

COPY --from=deps /app/node_modules ./node_modules
COPY --chown=nodejs:nodejs src/ ./src/

USER nodejs
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD wget -q --spider http://localhost:3000/health || exit 1

CMD ["node", "src/app.js"]
```

### ขั้นตอนที่ 4: Test the Gateway

```bash
# Start with docker-compose
docker-compose up -d

# Test health
curl http://localhost:3000/health

# Test public route
curl http://localhost:3000/api/v1/products

# Test auth flow
TOKEN=$(curl -s -X POST http://localhost:3000/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"user@test.com","password":"password123"}' \
  | jq -r '.data.token')

# Test protected route
curl http://localhost:3000/api/v1/users/me \
  -H "Authorization: Bearer $TOKEN"

# Test circuit breaker status
curl http://localhost:3000/health/circuit-breakers
```

---

## สรุป Part 10

ในบทนี้เราได้เรียนรู้:
- **API Gateway Pattern** และประโยชน์ในระบบ Microservices
- **Node.js API Gateway** - สร้าง production-ready gateway ด้วย Express
- **Request Proxying** - forward requests พร้อม context headers
- **Authentication/Authorization** ที่ Gateway layer
- **Rate Limiting** ด้วย Redis
- **Circuit Breaker** ด้วย opossum library
- **Request Aggregation** - รวม response จากหลาย services
- **Nginx Gateway** - สำหรับ high-performance production
- **Testing** - Integration tests ด้วย nock

**Next:** Part 11 - Database per Service Pattern

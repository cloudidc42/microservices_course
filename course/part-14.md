# Part 14: Error Handling & Resilience Patterns

## บทนำ

ในระบบ Microservices ความล้มเหลวไม่ใช่คำถามว่า "จะเกิดขึ้นไหม?" แต่คือ "จะเกิดขึ้นเมื่อไหร่?" การออกแบบระบบให้ "resilient" หมายถึงระบบยังทำงานต่อได้แม้บางส่วนล้มเหลว

---

## 14.1 Error Hierarchy

```javascript
// src/errors/AppError.js - Error hierarchy สำหรับทุก service

class AppError extends Error {
  constructor(message, options = {}) {
    super(message);
    this.name = this.constructor.name;
    this.statusCode = options.statusCode || 500;
    this.code = options.code || 'INTERNAL_SERVER_ERROR';
    this.isOperational = options.isOperational !== false; // false = programming error
    this.details = options.details || null;
    this.requestId = options.requestId || null;

    if (Error.captureStackTrace) {
      Error.captureStackTrace(this, this.constructor);
    }
  }

  toJSON() {
    return {
      code: this.code,
      message: this.message,
      ...(this.details && { details: this.details }),
      ...(process.env.NODE_ENV !== 'production' && { stack: this.stack }),
    };
  }
}

class ValidationError extends AppError {
  constructor(message, details = []) {
    super(message, {
      statusCode: 400,
      code: 'VALIDATION_ERROR',
      details,
    });
  }
}

class NotFoundError extends AppError {
  constructor(resource = 'Resource', id) {
    super(`${resource}${id ? ` with id '${id}'` : ''} not found`, {
      statusCode: 404,
      code: 'NOT_FOUND',
    });
  }
}

class ConflictError extends AppError {
  constructor(message) {
    super(message, { statusCode: 409, code: 'CONFLICT' });
  }
}

class UnauthorizedError extends AppError {
  constructor(message = 'Authentication required') {
    super(message, { statusCode: 401, code: 'UNAUTHORIZED' });
  }
}

class ForbiddenError extends AppError {
  constructor(message = 'Insufficient permissions') {
    super(message, { statusCode: 403, code: 'FORBIDDEN' });
  }
}

class ServiceUnavailableError extends AppError {
  constructor(serviceName) {
    super(`${serviceName} service is temporarily unavailable`, {
      statusCode: 503,
      code: 'SERVICE_UNAVAILABLE',
    });
  }
}

class RateLimitError extends AppError {
  constructor(retryAfter) {
    super('Rate limit exceeded', {
      statusCode: 429,
      code: 'RATE_LIMIT_EXCEEDED',
      details: { retryAfter },
    });
  }
}

class ExternalServiceError extends AppError {
  constructor(serviceName, message) {
    super(`External service error: ${serviceName} - ${message}`, {
      statusCode: 502,
      code: 'EXTERNAL_SERVICE_ERROR',
    });
  }
}

class DatabaseError extends AppError {
  constructor(message, originalError) {
    super(message, {
      statusCode: 500,
      code: 'DATABASE_ERROR',
      isOperational: false,
    });
    this.originalError = originalError;
  }
}

module.exports = {
  AppError,
  ValidationError,
  NotFoundError,
  ConflictError,
  UnauthorizedError,
  ForbiddenError,
  ServiceUnavailableError,
  RateLimitError,
  ExternalServiceError,
  DatabaseError,
};
```

---

## 14.2 Global Error Handler

```javascript
// src/middleware/errorHandler.js
const { AppError } = require('../errors/AppError');
const logger = require('../utils/logger');

// Normalize different error types to AppError
function normalizeError(err) {
  // Already an AppError
  if (err instanceof AppError) return err;

  // Joi validation error
  if (err.isJoi || err.name === 'ValidationError' && err.details) {
    const { ValidationError } = require('../errors/AppError');
    return new ValidationError(
      'Validation failed',
      err.details.map(d => ({ field: d.path.join('.'), message: d.message }))
    );
  }

  // PostgreSQL unique violation
  if (err.code === '23505') {
    const { ConflictError } = require('../errors/AppError');
    const field = err.constraint?.split('_idx_')[1] || 'field';
    return new ConflictError(`${field} already exists`);
  }

  // PostgreSQL not null violation
  if (err.code === '23502') {
    const { ValidationError } = require('../errors/AppError');
    return new ValidationError(`${err.column} is required`);
  }

  // JWT errors
  if (err.name === 'JsonWebTokenError') {
    const { UnauthorizedError } = require('../errors/AppError');
    return new UnauthorizedError('Invalid token');
  }

  if (err.name === 'TokenExpiredError') {
    const { UnauthorizedError } = require('../errors/AppError');
    return new UnauthorizedError('Token expired');
  }

  // MongoDB duplicate key
  if (err.code === 11000) {
    const { ConflictError } = require('../errors/AppError');
    const field = Object.keys(err.keyPattern || {})[0] || 'field';
    return new ConflictError(`${field} already exists`);
  }

  // Axios/HTTP client error (upstream service)
  if (err.response) {
    const { ExternalServiceError } = require('../errors/AppError');
    return new ExternalServiceError(
      err.config?.baseURL || 'upstream',
      err.response.data?.error?.message || err.message
    );
  }

  // Unknown error - treat as internal server error
  const { AppError: BaseError } = require('../errors/AppError');
  return new BaseError(
    process.env.NODE_ENV === 'production' ? 'An unexpected error occurred' : err.message,
    { isOperational: false }
  );
}

module.exports = function errorHandler(err, req, res, next) {
  const normalizedError = normalizeError(err);

  // Log based on severity
  if (!normalizedError.isOperational || normalizedError.statusCode >= 500) {
    logger.error('Unhandled error', {
      requestId: req.requestId,
      method: req.method,
      url: req.url,
      userId: req.user?.id,
      error: {
        name: err.name,
        message: err.message,
        code: normalizedError.code,
        stack: err.stack,
      },
    });
  } else {
    logger.warn('Operational error', {
      requestId: req.requestId,
      code: normalizedError.code,
      message: normalizedError.message,
    });
  }

  if (res.headersSent) return next(err);

  res.status(normalizedError.statusCode).json({
    success: false,
    error: normalizedError.toJSON(),
    requestId: req.requestId,
    timestamp: new Date().toISOString(),
  });
};
```

---

## 14.3 Circuit Breaker Pattern (Deep Dive)

```javascript
// src/resilience/CircuitBreaker.js

const EventEmitter = require('events');
const logger = require('../utils/logger');

const STATES = {
  CLOSED: 'CLOSED',   // Normal operation
  OPEN: 'OPEN',       // Failing, reject requests
  HALF_OPEN: 'HALF_OPEN', // Testing if service recovered
};

class CircuitBreaker extends EventEmitter {
  constructor(name, fn, options = {}) {
    super();
    this.name = name;
    this.fn = fn;

    // Configuration
    this.failureThreshold = options.failureThreshold || 5;     // Failures before OPEN
    this.successThreshold = options.successThreshold || 2;     // Successes to close
    this.timeout = options.timeout || 5000;                    // Request timeout (ms)
    this.resetTimeout = options.resetTimeout || 30000;          // Time before HALF_OPEN
    this.volumeThreshold = options.volumeThreshold || 10;      // Min calls before tripping

    // State
    this.state = STATES.CLOSED;
    this.failureCount = 0;
    this.successCount = 0;
    this.totalCount = 0;
    this.lastFailureTime = null;
    this.halfOpenTimer = null;

    // Sliding window for tracking
    this.window = [];
    this.windowSize = options.windowSize || 60000; // 1 minute window
  }

  async fire(...args) {
    this.totalCount++;
    this.cleanWindow();

    if (this.state === STATES.OPEN) {
      if (this.shouldAttemptReset()) {
        return this.tryHalfOpen(...args);
      }

      this.emit('rejected');
      throw Object.assign(
        new Error(`Circuit breaker OPEN for ${this.name}`),
        { code: 'CIRCUIT_OPEN' }
      );
    }

    return this.call(...args);
  }

  async call(...args) {
    const startTime = Date.now();

    try {
      // Add timeout wrapper
      const result = await this.withTimeout(this.fn(...args));
      this.onSuccess(Date.now() - startTime);
      return result;
    } catch (error) {
      this.onFailure(error, Date.now() - startTime);
      throw error;
    }
  }

  withTimeout(promise) {
    if (!this.timeout) return promise;

    return new Promise((resolve, reject) => {
      const timer = setTimeout(() => {
        reject(Object.assign(new Error(`Circuit breaker timeout for ${this.name}`), {
          code: 'CIRCUIT_TIMEOUT',
        }));
      }, this.timeout);

      promise.then(
        (value) => { clearTimeout(timer); resolve(value); },
        (err) => { clearTimeout(timer); reject(err); }
      );
    });
  }

  onSuccess(duration) {
    this.window.push({ success: true, duration, timestamp: Date.now() });

    if (this.state === STATES.HALF_OPEN) {
      this.successCount++;
      if (this.successCount >= this.successThreshold) {
        this.close();
      }
    } else {
      this.failureCount = 0; // Reset on success
    }

    this.emit('success', { duration });
  }

  onFailure(error, duration) {
    this.window.push({ success: false, duration, error: error.message, timestamp: Date.now() });
    this.lastFailureTime = Date.now();

    if (this.state === STATES.HALF_OPEN) {
      this.open(error);
      return;
    }

    this.failureCount++;
    const recentErrors = this.window.filter(e => !e.success).length;
    const errorRate = this.window.length >= this.volumeThreshold
      ? recentErrors / this.window.length
      : 0;

    if (this.failureCount >= this.failureThreshold || errorRate > 0.5) {
      this.open(error);
    }

    this.emit('failure', { error: error.message, failureCount: this.failureCount });
  }

  open(error) {
    this.state = STATES.OPEN;
    this.successCount = 0;
    logger.error(`Circuit OPENED for ${this.name}`, {
      failureCount: this.failureCount,
      error: error?.message,
    });
    this.emit('open', { name: this.name });
  }

  close() {
    this.state = STATES.CLOSED;
    this.failureCount = 0;
    this.successCount = 0;
    logger.info(`Circuit CLOSED for ${this.name}`);
    this.emit('close', { name: this.name });
  }

  async tryHalfOpen(...args) {
    this.state = STATES.HALF_OPEN;
    this.successCount = 0;
    logger.info(`Circuit HALF-OPEN for ${this.name}, testing...`);
    this.emit('halfOpen', { name: this.name });

    try {
      const result = await this.call(...args);
      return result;
    } catch (error) {
      throw error;
    }
  }

  shouldAttemptReset() {
    return this.lastFailureTime &&
      (Date.now() - this.lastFailureTime) >= this.resetTimeout;
  }

  cleanWindow() {
    const cutoff = Date.now() - this.windowSize;
    this.window = this.window.filter(e => e.timestamp > cutoff);
  }

  getStats() {
    this.cleanWindow();
    const recent = this.window;
    const successes = recent.filter(e => e.success).length;
    const failures = recent.filter(e => !e.success).length;
    const avgDuration = recent.length > 0
      ? recent.reduce((sum, e) => sum + e.duration, 0) / recent.length
      : 0;

    return {
      name: this.name,
      state: this.state,
      failureCount: this.failureCount,
      recentTotal: recent.length,
      recentSuccesses: successes,
      recentFailures: failures,
      errorRate: recent.length > 0 ? Math.round((failures / recent.length) * 100) : 0,
      avgDuration: Math.round(avgDuration),
      lastFailureTime: this.lastFailureTime
        ? new Date(this.lastFailureTime).toISOString()
        : null,
    };
  }
}

module.exports = { CircuitBreaker, STATES };
```

---

## 14.4 Retry Pattern

```javascript
// src/resilience/retry.js

const logger = require('../utils/logger');

// Default retry conditions
const DEFAULT_RETRYABLE_ERRORS = new Set([
  'ECONNRESET',
  'ECONNREFUSED',
  'ETIMEDOUT',
  'ENOTFOUND',
  'NETWORK_ERROR',
]);

const DEFAULT_RETRYABLE_STATUS = new Set([408, 429, 500, 502, 503, 504]);

function sleep(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

function calculateDelay(attempt, options) {
  const { baseDelay = 100, maxDelay = 30000, strategy = 'exponential', jitter = true } = options;

  let delay;
  switch (strategy) {
    case 'linear':
      delay = baseDelay * attempt;
      break;
    case 'constant':
      delay = baseDelay;
      break;
    case 'exponential':
    default:
      delay = baseDelay * Math.pow(2, attempt - 1);
  }

  // Add jitter to prevent thundering herd
  if (jitter) {
    delay = delay * (0.5 + Math.random() * 0.5); // ±50% jitter
  }

  return Math.min(delay, maxDelay);
}

async function withRetry(fn, options = {}) {
  const {
    maxAttempts = 3,
    baseDelay = 100,
    maxDelay = 30000,
    strategy = 'exponential',
    jitter = true,
    shouldRetry,
    onRetry,
    name = 'operation',
  } = options;

  let lastError;

  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      const result = await fn(attempt);
      if (attempt > 1) {
        logger.info(`${name} succeeded after ${attempt} attempts`);
      }
      return result;
    } catch (error) {
      lastError = error;

      const isRetryable = shouldRetry
        ? shouldRetry(error, attempt)
        : isRetryableError(error);

      if (!isRetryable || attempt === maxAttempts) {
        logger.error(`${name} failed after ${attempt} attempts`, { error: error.message });
        throw error;
      }

      const delay = calculateDelay(attempt, { baseDelay, maxDelay, strategy, jitter });

      logger.warn(`${name} attempt ${attempt} failed, retrying in ${delay}ms`, {
        error: error.message,
        attempt,
        maxAttempts,
      });

      if (onRetry) {
        await onRetry(error, attempt, delay);
      }

      await sleep(delay);
    }
  }

  throw lastError;
}

function isRetryableError(error) {
  if (DEFAULT_RETRYABLE_ERRORS.has(error.code)) return true;
  if (error.response && DEFAULT_RETRYABLE_STATUS.has(error.response.status)) return true;
  return false;
}

// Decorator version
function retryable(options = {}) {
  return function(target, propertyKey, descriptor) {
    const originalMethod = descriptor.value;
    descriptor.value = function(...args) {
      return withRetry(() => originalMethod.apply(this, args), {
        name: `${target.constructor.name}.${propertyKey}`,
        ...options,
      });
    };
    return descriptor;
  };
}

module.exports = { withRetry, isRetryableError, retryable };
```

---

## 14.5 Bulkhead Pattern

```javascript
// src/resilience/bulkhead.js

const logger = require('../utils/logger');

class Bulkhead {
  constructor(name, options = {}) {
    this.name = name;
    this.maxConcurrent = options.maxConcurrent || 10;
    this.maxQueue = options.maxQueue || 50;
    this.timeout = options.timeout || 30000; // Queue wait timeout

    this.activeCount = 0;
    this.queue = [];
  }

  execute(fn) {
    return new Promise((resolve, reject) => {
      // Check queue capacity
      if (this.queue.length >= this.maxQueue) {
        reject(Object.assign(
          new Error(`Bulkhead ${this.name} queue full`),
          { code: 'BULKHEAD_FULL' }
        ));
        return;
      }

      const task = { fn, resolve, reject };

      if (this.activeCount < this.maxConcurrent) {
        this.runTask(task);
      } else {
        // Add timeout to queued task
        task.timer = setTimeout(() => {
          const idx = this.queue.indexOf(task);
          if (idx !== -1) {
            this.queue.splice(idx, 1);
            reject(Object.assign(
              new Error(`Bulkhead ${this.name} queue timeout`),
              { code: 'BULKHEAD_TIMEOUT' }
            ));
          }
        }, this.timeout);

        this.queue.push(task);
        logger.debug(`Bulkhead ${this.name} queued`, {
          active: this.activeCount,
          queued: this.queue.length,
        });
      }
    });
  }

  async runTask(task) {
    this.activeCount++;

    try {
      const result = await task.fn();
      task.resolve(result);
    } catch (error) {
      task.reject(error);
    } finally {
      this.activeCount--;
      clearTimeout(task.timer);

      // Process next in queue
      if (this.queue.length > 0) {
        const next = this.queue.shift();
        this.runTask(next);
      }
    }
  }

  getStats() {
    return {
      name: this.name,
      maxConcurrent: this.maxConcurrent,
      maxQueue: this.maxQueue,
      activeCount: this.activeCount,
      queuedCount: this.queue.length,
      utilization: Math.round((this.activeCount / this.maxConcurrent) * 100),
    };
  }
}

// Create bulkheads for different services
const bulkheads = {
  userService: new Bulkhead('user-service', { maxConcurrent: 20, maxQueue: 100 }),
  productService: new Bulkhead('product-service', { maxConcurrent: 50, maxQueue: 200 }),
  orderService: new Bulkhead('order-service', { maxConcurrent: 30, maxQueue: 150 }),
  paymentService: new Bulkhead('payment-service', { maxConcurrent: 10, maxQueue: 50 }),
};

module.exports = { Bulkhead, bulkheads };
```

---

## 14.6 Timeout Pattern

```javascript
// src/resilience/timeout.js

function withTimeout(promise, ms, message) {
  let timeoutId;

  const timeoutPromise = new Promise((_, reject) => {
    timeoutId = setTimeout(() => {
      reject(Object.assign(
        new Error(message || `Operation timed out after ${ms}ms`),
        { code: 'TIMEOUT' }
      ));
    }, ms);
  });

  return Promise.race([
    promise.finally(() => clearTimeout(timeoutId)),
    timeoutPromise,
  ]);
}

// Express timeout middleware
function requestTimeout(ms) {
  return (req, res, next) => {
    const timer = setTimeout(() => {
      if (!res.headersSent) {
        res.status(503).json({
          success: false,
          error: {
            code: 'REQUEST_TIMEOUT',
            message: 'Request timed out',
          },
          requestId: req.requestId,
        });
      }
    }, ms);

    res.on('finish', () => clearTimeout(timer));
    res.on('close', () => clearTimeout(timer));
    next();
  };
}

module.exports = { withTimeout, requestTimeout };
```

---

## 14.7 Resilient HTTP Client

```javascript
// src/http/resilientClient.js

const axios = require('axios');
const { CircuitBreaker } = require('../resilience/CircuitBreaker');
const { withRetry } = require('../resilience/retry');
const { Bulkhead } = require('../resilience/bulkhead');
const { withTimeout } = require('../resilience/timeout');
const logger = require('../utils/logger');

class ResilientHttpClient {
  constructor(baseURL, options = {}) {
    this.baseURL = baseURL;
    this.serviceName = options.name || new URL(baseURL).hostname;

    // Create axios instance
    this.axios = axios.create({
      baseURL,
      timeout: options.timeout || 5000,
      headers: {
        'Content-Type': 'application/json',
        ...options.headers,
      },
    });

    // Setup resilience patterns
    this.circuitBreaker = new CircuitBreaker(
      this.serviceName,
      (config) => this.axios(config),
      {
        failureThreshold: options.failureThreshold || 5,
        resetTimeout: options.resetTimeout || 30000,
        timeout: options.requestTimeout || 5000,
      }
    );

    this.bulkhead = new Bulkhead(this.serviceName, {
      maxConcurrent: options.maxConcurrent || 20,
      maxQueue: options.maxQueue || 100,
    });

    this.retryOptions = {
      maxAttempts: options.maxRetries || 3,
      baseDelay: options.retryDelay || 200,
      name: this.serviceName,
    };

    // Add request interceptor for logging
    this.axios.interceptors.request.use((config) => {
      config.metadata = { startTime: Date.now() };
      return config;
    });

    this.axios.interceptors.response.use(
      (response) => {
        const duration = Date.now() - response.config.metadata?.startTime;
        logger.debug(`HTTP response from ${this.serviceName}`, {
          method: response.config.method,
          url: response.config.url,
          status: response.status,
          duration: `${duration}ms`,
        });
        return response;
      },
      (error) => {
        const duration = Date.now() - error.config?.metadata?.startTime;
        logger.warn(`HTTP error from ${this.serviceName}`, {
          method: error.config?.method,
          url: error.config?.url,
          status: error.response?.status,
          duration: `${duration}ms`,
          error: error.message,
        });
        return Promise.reject(error);
      }
    );
  }

  async request(config, options = {}) {
    const makeRequest = async () => {
      return this.bulkhead.execute(async () => {
        return this.circuitBreaker.fire(config);
      });
    };

    if (options.retry !== false) {
      return withRetry(makeRequest, {
        ...this.retryOptions,
        shouldRetry: options.shouldRetry,
      });
    }

    return makeRequest();
  }

  async get(url, params, options) {
    const response = await this.request({ method: 'GET', url, params }, options);
    return response.data;
  }

  async post(url, data, options) {
    const response = await this.request({ method: 'POST', url, data }, options);
    return response.data;
  }

  async put(url, data, options) {
    const response = await this.request({ method: 'PUT', url, data }, options);
    return response.data;
  }

  async patch(url, data, options) {
    const response = await this.request({ method: 'PATCH', url, data }, options);
    return response.data;
  }

  async delete(url, options) {
    const response = await this.request({ method: 'DELETE', url }, options);
    return response.data;
  }

  getHealth() {
    return {
      circuitBreaker: this.circuitBreaker.getStats(),
      bulkhead: this.bulkhead.getStats(),
    };
  }
}

module.exports = ResilientHttpClient;
```

---

## 14.8 Health Check Pattern

```javascript
// src/health/healthChecker.js

const logger = require('../utils/logger');

class HealthChecker {
  constructor(name) {
    this.name = name;
    this.checks = new Map();
    this.status = 'initializing';
  }

  register(name, checkFn, options = {}) {
    this.checks.set(name, {
      fn: checkFn,
      critical: options.critical !== false, // Default: critical
      timeout: options.timeout || 3000,
      interval: options.interval || 30000,
      lastResult: null,
      lastCheckedAt: null,
    });

    // Start periodic check
    if (options.interval !== 0) {
      setInterval(() => this.runCheck(name), options.interval || 30000);
    }

    return this;
  }

  async runCheck(name) {
    const check = this.checks.get(name);
    if (!check) return;

    const startTime = Date.now();

    try {
      const result = await withTimeout(check.fn(), check.timeout);
      check.lastResult = {
        status: 'healthy',
        duration: Date.now() - startTime,
        details: result,
      };
    } catch (error) {
      check.lastResult = {
        status: 'unhealthy',
        duration: Date.now() - startTime,
        error: error.message,
      };
    }

    check.lastCheckedAt = new Date().toISOString();
    return check.lastResult;
  }

  async runAll() {
    const results = {};

    await Promise.all(
      Array.from(this.checks.entries()).map(async ([name]) => {
        results[name] = await this.runCheck(name);
      })
    );

    return results;
  }

  async getHealth() {
    const checkResults = await this.runAll();

    const allHealthy = Object.entries(checkResults).every(([name, result]) => {
      const check = this.checks.get(name);
      return !check.critical || result?.status === 'healthy';
    });

    this.status = allHealthy ? 'healthy' : 'unhealthy';

    return {
      status: this.status,
      service: this.name,
      version: process.env.APP_VERSION || '1.0.0',
      timestamp: new Date().toISOString(),
      uptime: process.uptime(),
      memory: {
        used: Math.round(process.memoryUsage().heapUsed / 1024 / 1024),
        total: Math.round(process.memoryUsage().heapTotal / 1024 / 1024),
        unit: 'MB',
      },
      checks: Object.entries(checkResults).reduce((acc, [name, result]) => {
        acc[name] = {
          status: result?.status || 'unknown',
          duration: result?.duration,
          checkedAt: this.checks.get(name)?.lastCheckedAt,
          ...(result?.error && { error: result.error }),
          ...(result?.details && { details: result.details }),
        };
        return acc;
      }, {}),
    };
  }

  middleware() {
    return async (req, res) => {
      const health = await this.getHealth();
      const statusCode = health.status === 'healthy' ? 200 : 503;
      res.status(statusCode).json(health);
    };
  }

  liveness() {
    return (req, res) => {
      res.json({
        status: 'alive',
        service: this.name,
        timestamp: new Date().toISOString(),
      });
    };
  }

  readiness() {
    return async (req, res) => {
      const health = await this.getHealth();
      const isReady = health.status === 'healthy';
      res.status(isReady ? 200 : 503).json({
        status: isReady ? 'ready' : 'not ready',
        checks: health.checks,
      });
    };
  }
}

function withTimeout(promise, ms) {
  return new Promise((resolve, reject) => {
    const timer = setTimeout(() => reject(new Error(`Check timed out after ${ms}ms`)), ms);
    promise.then(
      v => { clearTimeout(timer); resolve(v); },
      e => { clearTimeout(timer); reject(e); }
    );
  });
}

module.exports = HealthChecker;
```

### Setting Up Health Checks in a Service

```javascript
// src/health/index.js
const HealthChecker = require('./healthChecker');
const { pool } = require('../database');
const redis = require('../cache/redis');
const rabbitmq = require('../messaging/rabbitmq');
const config = require('../config');

const healthChecker = new HealthChecker(config.serviceName);

// Database check
healthChecker.register('database', async () => {
  const result = await pool.query('SELECT 1');
  return { connected: true, responseTime: result.duration };
}, { critical: true });

// Redis check
healthChecker.register('redis', async () => {
  await redis.ping();
  return { connected: true };
}, { critical: false }); // Redis is not critical - cache can miss

// RabbitMQ check
healthChecker.register('rabbitmq', async () => {
  const connected = rabbitmq.isConnected();
  if (!connected) throw new Error('RabbitMQ not connected');
  return { connected: true };
}, { critical: false });

// Disk space check
healthChecker.register('disk', async () => {
  const { checkDiskSpace } = require('check-disk-space');
  const { free, size } = await checkDiskSpace('/');
  const freePercent = Math.round((free / size) * 100);

  if (freePercent < 10) {
    throw new Error(`Low disk space: ${freePercent}% free`);
  }

  return { freePercent, freeGB: Math.round(free / 1e9) };
}, { critical: false, interval: 60000 });

module.exports = healthChecker;
```

### Routes

```javascript
// src/routes/health.routes.js
const express = require('express');
const router = express.Router();
const healthChecker = require('../health');

// Kubernetes liveness probe
router.get('/health/live', healthChecker.liveness());

// Kubernetes readiness probe
router.get('/health/ready', healthChecker.readiness());

// Full health check
router.get('/health', healthChecker.middleware());

module.exports = router;
```

---

## 14.9 Graceful Shutdown

```javascript
// src/utils/gracefulShutdown.js

const logger = require('./logger');

class GracefulShutdown {
  constructor(server, options = {}) {
    this.server = server;
    this.timeout = options.timeout || 30000; // 30 seconds
    this.handlers = [];
    this.isShuttingDown = false;

    this.setup();
  }

  addHandler(name, fn) {
    this.handlers.push({ name, fn });
    return this;
  }

  setup() {
    const shutdown = this.shutdown.bind(this);

    process.on('SIGTERM', () => {
      logger.info('SIGTERM received');
      shutdown('SIGTERM');
    });

    process.on('SIGINT', () => {
      logger.info('SIGINT received');
      shutdown('SIGINT');
    });

    process.on('uncaughtException', (error) => {
      logger.error('Uncaught Exception', { error: error.message, stack: error.stack });
      shutdown('uncaughtException');
    });

    process.on('unhandledRejection', (reason, promise) => {
      logger.error('Unhandled Rejection', {
        reason: reason?.message || reason,
        stack: reason?.stack,
      });
      // Don't shutdown for unhandled rejections - just log them
    });
  }

  async shutdown(signal) {
    if (this.isShuttingDown) return;
    this.isShuttingDown = true;

    logger.info(`Starting graceful shutdown (signal: ${signal})`);

    // Stop accepting new connections
    this.server.close(async () => {
      logger.info('HTTP server closed (no new connections)');
    });

    // Timeout force kill
    const forceTimeout = setTimeout(() => {
      logger.error('Graceful shutdown timeout, forcing exit');
      process.exit(1);
    }, this.timeout);

    try {
      // Run cleanup handlers in reverse order
      for (const { name, fn } of [...this.handlers].reverse()) {
        try {
          logger.info(`Running shutdown handler: ${name}`);
          await fn();
          logger.info(`Shutdown handler completed: ${name}`);
        } catch (error) {
          logger.error(`Shutdown handler failed: ${name}`, { error: error.message });
        }
      }

      clearTimeout(forceTimeout);
      logger.info('Graceful shutdown complete');
      process.exit(0);
    } catch (error) {
      logger.error('Shutdown error', { error: error.message });
      clearTimeout(forceTimeout);
      process.exit(1);
    }
  }
}

module.exports = GracefulShutdown;

// Usage in app.js
/*
const server = app.listen(config.port);
const gracefulShutdown = new GracefulShutdown(server);

gracefulShutdown
  .addHandler('database', () => pool.end())
  .addHandler('redis', () => redis.quit())
  .addHandler('rabbitmq', () => rabbitmq.close());
*/
```

---

## 14.10 Testing Error Handling

```javascript
// tests/errorHandling.test.js
const request = require('supertest');
const app = require('../src/app');
const { CircuitBreaker } = require('../src/resilience/CircuitBreaker');

describe('Error Handling', () => {
  describe('Global Error Handler', () => {
    it('should handle 404 errors', async () => {
      const response = await request(app)
        .get('/api/non-existent-route')
        .expect(404);

      expect(response.body.success).toBe(false);
      expect(response.body.error.code).toBe('NOT_FOUND');
    });

    it('should handle validation errors', async () => {
      const response = await request(app)
        .post('/api/v1/orders')
        .send({ invalid: 'data' })
        .expect(400);

      expect(response.body.success).toBe(false);
      expect(response.body.error.code).toBe('VALIDATION_ERROR');
    });

    it('should not expose stack traces in production', async () => {
      process.env.NODE_ENV = 'production';

      const response = await request(app)
        .get('/api/trigger-error')
        .expect(500);

      expect(response.body.error.stack).toBeUndefined();

      process.env.NODE_ENV = 'test';
    });
  });

  describe('Circuit Breaker', () => {
    let callCount = 0;
    let breaker;

    beforeEach(() => {
      callCount = 0;
      breaker = new CircuitBreaker('test-service', async () => {
        callCount++;
        throw new Error('Service down');
      }, {
        failureThreshold: 3,
        resetTimeout: 100,
        timeout: 1000,
      });
    });

    it('should open after failure threshold', async () => {
      expect(breaker.state).toBe('CLOSED');

      for (let i = 0; i < 3; i++) {
        try { await breaker.fire(); } catch {}
      }

      expect(breaker.state).toBe('OPEN');
    });

    it('should reject requests when OPEN', async () => {
      // Open the circuit
      for (let i = 0; i < 3; i++) {
        try { await breaker.fire(); } catch {}
      }

      const callsBefore = callCount;

      try {
        await breaker.fire();
      } catch (error) {
        expect(error.code).toBe('CIRCUIT_OPEN');
      }

      // Should not have called the function
      expect(callCount).toBe(callsBefore);
    });

    it('should transition to HALF_OPEN after resetTimeout', async () => {
      // Open the circuit
      for (let i = 0; i < 3; i++) {
        try { await breaker.fire(); } catch {}
      }

      expect(breaker.state).toBe('OPEN');

      // Wait for reset timeout
      await new Promise(resolve => setTimeout(resolve, 150));

      // Next request should try (HALF_OPEN)
      try { await breaker.fire(); } catch {}

      expect(breaker.state).not.toBe('CLOSED'); // Still unhealthy
    });
  });

  describe('Retry Pattern', () => {
    const { withRetry } = require('../src/resilience/retry');

    it('should retry on transient failures', async () => {
      let attempts = 0;
      const result = await withRetry(async () => {
        attempts++;
        if (attempts < 3) throw Object.assign(new Error('Transient'), { code: 'ECONNRESET' });
        return 'success';
      }, { maxAttempts: 3, baseDelay: 10 });

      expect(result).toBe('success');
      expect(attempts).toBe(3);
    });

    it('should not retry on non-retryable errors', async () => {
      let attempts = 0;

      await expect(withRetry(async () => {
        attempts++;
        throw Object.assign(new Error('Not retryable'), { code: 'INVALID_INPUT' });
      }, {
        maxAttempts: 3,
        baseDelay: 10,
        shouldRetry: (err) => err.code === 'ECONNRESET',
      })).rejects.toThrow('Not retryable');

      expect(attempts).toBe(1);
    });
  });

  describe('Health Check', () => {
    it('should return 200 when all critical checks pass', async () => {
      const response = await request(app)
        .get('/health')
        .expect(200);

      expect(response.body.status).toBe('healthy');
    });

    it('should return 503 when critical check fails', async () => {
      // Mock DB failure
      jest.spyOn(require('../src/database').pool, 'query')
        .mockRejectedValueOnce(new Error('DB connection failed'));

      const response = await request(app)
        .get('/health')
        .expect(503);

      expect(response.body.status).toBe('unhealthy');
    });
  });
});
```

---

## 14.11 Observability - Metrics

```javascript
// src/metrics/prometheus.js
const client = require('prom-client');

// Register default metrics (CPU, memory, event loop lag)
client.collectDefaultMetrics({ prefix: 'node_' });

// Custom metrics
const httpRequestDuration = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration in seconds',
  labelNames: ['method', 'route', 'status_code', 'service'],
  buckets: [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5],
});

const httpRequestsTotal = new client.Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status_code', 'service'],
});

const circuitBreakerState = new client.Gauge({
  name: 'circuit_breaker_state',
  help: 'Circuit breaker state (0=closed, 1=open, 2=half-open)',
  labelNames: ['service'],
});

const activeConnections = new client.Gauge({
  name: 'active_connections',
  help: 'Number of active connections',
  labelNames: ['service'],
});

const messageQueueSize = new client.Gauge({
  name: 'message_queue_size',
  help: 'Number of messages in queue',
  labelNames: ['queue'],
});

// Middleware to track request metrics
function metricsMiddleware(serviceName) {
  return (req, res, next) => {
    const start = Date.now();

    res.on('finish', () => {
      const duration = (Date.now() - start) / 1000;
      const route = req.route?.path || req.path;
      const labels = {
        method: req.method,
        route,
        status_code: res.statusCode,
        service: serviceName,
      };

      httpRequestDuration.observe(labels, duration);
      httpRequestsTotal.inc(labels);
    });

    next();
  };
}

function metricsRoute() {
  return async (req, res) => {
    res.set('Content-Type', client.register.contentType);
    res.end(await client.register.metrics());
  };
}

module.exports = {
  metricsMiddleware,
  metricsRoute,
  metrics: {
    httpRequestDuration,
    httpRequestsTotal,
    circuitBreakerState,
    activeConnections,
    messageQueueSize,
  },
  register: client.register,
};
```

---

## สรุป Part 14

ในบทนี้เราได้เรียนรู้:
- **Error Hierarchy** - สร้าง custom errors แบบ typed
- **Global Error Handler** - normalize errors, log, respond
- **Circuit Breaker** - ป้องกัน cascading failures
- **Retry Pattern** - exponential backoff with jitter
- **Bulkhead Pattern** - isolate concurrent connections
- **Timeout Pattern** - ป้องกัน hanging requests
- **Resilient HTTP Client** - รวม patterns ทั้งหมด
- **Health Checks** - liveness, readiness, full health
- **Graceful Shutdown** - clean shutdown process
- **Prometheus Metrics** - observability

**Next:** Part 15 - Logging & Distributed Tracing

# Part 68: Microservices Resilience Patterns

## บทนำ

Resilience patterns ช่วยให้ระบบ microservices ทนทานต่อ failures และกู้คืนได้อัตโนมัติ ใน Part นี้เราจะเรียนรู้ patterns ที่ใช้งานจริงในระดับ production พร้อมโค้ดตัวอย่างครบถ้วน

---

## 1. Circuit Breaker ด้วย Opossum

Circuit Breaker ป้องกันการเรียก service ที่ล้มเหลวซ้ำๆ ซึ่งทำให้เกิด cascade failure

```bash
npm install opossum
npm install --save-dev @types/opossum
```

```typescript
// resilience/circuit-breaker.ts
import CircuitBreaker from "opossum";
import { logger } from "../utils/logger";
import { metrics } from "../utils/metrics";

interface CircuitBreakerConfig {
  timeout?: number;           // ms ก่อน request timeout
  errorThresholdPercentage?: number; // % error ที่จะ open circuit
  resetTimeout?: number;      // ms ก่อนลอง half-open
  rollingCountTimeout?: number;      // ช่วงเวลา rolling stats
  rollingCountBuckets?: number;      // จำนวน bucket ใน rolling window
  volumeThreshold?: number;         // minimum requests ก่อนเปิด circuit
}

export function createCircuitBreaker<T extends (...args: any[]) => Promise<any>>(
  fn: T,
  name: string,
  options: CircuitBreakerConfig = {}
): CircuitBreaker {
  const breaker = new CircuitBreaker(fn, {
    timeout: options.timeout ?? 3000,
    errorThresholdPercentage: options.errorThresholdPercentage ?? 50,
    resetTimeout: options.resetTimeout ?? 30000,
    rollingCountTimeout: options.rollingCountTimeout ?? 10000,
    rollingCountBuckets: options.rollingCountBuckets ?? 10,
    volumeThreshold: options.volumeThreshold ?? 5,
    name,
  });

  // Event handlers
  breaker.on("open", () => {
    logger.warn(`Circuit breaker OPENED: ${name}`);
    metrics.increment(`circuit_breaker.${name}.opened`);
  });

  breaker.on("halfOpen", () => {
    logger.info(`Circuit breaker HALF-OPEN: ${name}`);
    metrics.increment(`circuit_breaker.${name}.half_opened`);
  });

  breaker.on("close", () => {
    logger.info(`Circuit breaker CLOSED: ${name}`);
    metrics.increment(`circuit_breaker.${name}.closed`);
  });

  breaker.on("fallback", (result) => {
    logger.info(`Circuit breaker fallback used: ${name}`, { result });
    metrics.increment(`circuit_breaker.${name}.fallback`);
  });

  breaker.on("success", (result, latency) => {
    metrics.histogram(`circuit_breaker.${name}.latency`, latency);
  });

  breaker.on("failure", (error) => {
    logger.error(`Circuit breaker failure: ${name}`, error);
    metrics.increment(`circuit_breaker.${name}.failures`);
  });

  breaker.on("timeout", () => {
    logger.warn(`Circuit breaker timeout: ${name}`);
    metrics.increment(`circuit_breaker.${name}.timeouts`);
  });

  return breaker;
}

// services/payment.client.ts
import axios, { AxiosInstance } from "axios";

class PaymentServiceClient {
  private readonly httpClient: AxiosInstance;
  private readonly circuitBreaker: CircuitBreaker;

  constructor(private readonly baseUrl: string) {
    this.httpClient = axios.create({
      baseURL: baseUrl,
      timeout: 5000,
    });

    // Create circuit breaker for the charge function
    this.circuitBreaker = createCircuitBreaker(
      this.chargeInternal.bind(this),
      "payment-service",
      {
        timeout: 5000,
        errorThresholdPercentage: 50,
        resetTimeout: 30000,
        volumeThreshold: 3,
      }
    );

    // Set fallback
    this.circuitBreaker.fallback(async (params) => {
      logger.warn("Payment service unavailable, queuing charge", params);
      await paymentQueue.enqueue(params);
      return { status: "queued", id: `queue-${Date.now()}` };
    });
  }

  async charge(params: ChargeParams): Promise<ChargeResult> {
    return this.circuitBreaker.fire(params) as Promise<ChargeResult>;
  }

  private async chargeInternal(params: ChargeParams): Promise<ChargeResult> {
    const response = await this.httpClient.post<ChargeResult>("/charges", params);
    return response.data;
  }

  getState(): string {
    return this.circuitBreaker.opened
      ? "open"
      : this.circuitBreaker.halfOpen
      ? "half-open"
      : "closed";
  }

  getStats() {
    const stats = this.circuitBreaker.stats;
    return {
      state: this.getState(),
      requests: stats.fires,
      successes: stats.successes,
      failures: stats.failures,
      timeouts: stats.timeouts,
      fallbacks: stats.fallbacks,
      rejects: stats.rejects,
    };
  }
}
```

---

## 2. Retry with Exponential Backoff

```typescript
// resilience/retry.ts
interface RetryOptions {
  maxAttempts: number;
  initialDelayMs: number;
  maxDelayMs: number;
  backoffMultiplier: number;
  jitter: boolean;
  retryableErrors?: (error: Error) => boolean;
  onRetry?: (error: Error, attempt: number, delay: number) => void;
}

const defaultOptions: RetryOptions = {
  maxAttempts: 3,
  initialDelayMs: 100,
  maxDelayMs: 10000,
  backoffMultiplier: 2,
  jitter: true,
};

export async function retry<T>(
  fn: () => Promise<T>,
  options: Partial<RetryOptions> = {}
): Promise<T> {
  const opts = { ...defaultOptions, ...options };
  let lastError: Error;

  for (let attempt = 1; attempt <= opts.maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error as Error;

      // Check if error is retryable
      if (opts.retryableErrors && !opts.retryableErrors(lastError)) {
        throw lastError;
      }

      if (attempt === opts.maxAttempts) break;

      // Calculate delay with exponential backoff
      const exponentialDelay =
        opts.initialDelayMs * Math.pow(opts.backoffMultiplier, attempt - 1);
      const boundedDelay = Math.min(exponentialDelay, opts.maxDelayMs);
      const delay = opts.jitter
        ? boundedDelay * (0.5 + Math.random() * 0.5)
        : boundedDelay;

      opts.onRetry?.(lastError, attempt, delay);
      await sleep(delay);
    }
  }

  throw lastError!;
}

// Retry สำหรับ HTTP errors เฉพาะ
function isRetryableHttpError(error: Error): boolean {
  if (axios.isAxiosError(error)) {
    const status = error.response?.status;
    // Retry on 5xx and 429 (rate limit)
    return !status || status >= 500 || status === 429;
  }
  // Retry on network errors
  return error.message.includes("ECONNREFUSED") ||
    error.message.includes("ETIMEDOUT") ||
    error.message.includes("ENOTFOUND");
}

// ตัวอย่างการใช้งาน
const result = await retry(
  () => paymentService.charge({ amount: 100, currency: "THB" }),
  {
    maxAttempts: 5,
    initialDelayMs: 200,
    maxDelayMs: 8000,
    backoffMultiplier: 2,
    jitter: true,
    retryableErrors: isRetryableHttpError,
    onRetry: (error, attempt, delay) => {
      logger.warn("Retrying payment charge", {
        attempt,
        delay: `${delay.toFixed(0)}ms`,
        error: error.message,
      });
    },
  }
);

// Decorator version
export function Retry(options: Partial<RetryOptions> = {}) {
  return function (
    _target: unknown,
    _propertyKey: string,
    descriptor: PropertyDescriptor
  ) {
    const originalMethod = descriptor.value;
    descriptor.value = function (...args: unknown[]) {
      return retry(() => originalMethod.apply(this, args), options);
    };
    return descriptor;
  };
}
```

---

## 3. Bulkhead Pattern

Bulkhead แยก resources ออกจากกัน เพื่อให้ failure ของ component หนึ่งไม่กระทบ component อื่น

```typescript
// resilience/bulkhead.ts
import { Semaphore } from "async-mutex";

interface BulkheadConfig {
  maxConcurrentCalls: number;
  maxQueueSize: number;
  queueTimeoutMs: number;
}

export class Bulkhead {
  private readonly semaphore: Semaphore;
  private currentQueue: number = 0;

  constructor(
    private readonly name: string,
    private readonly config: BulkheadConfig
  ) {
    this.semaphore = new Semaphore(config.maxConcurrentCalls);
  }

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    if (this.currentQueue >= this.config.maxQueueSize) {
      metrics.increment(`bulkhead.${this.name}.rejected`);
      throw new BulkheadFullError(this.name);
    }

    this.currentQueue++;
    metrics.gauge(`bulkhead.${this.name}.queue_size`, this.currentQueue);

    try {
      // Wait for a slot with timeout
      const acquired = await this.semaphore.acquire();

      this.currentQueue--;
      metrics.gauge(`bulkhead.${this.name}.queue_size`, this.currentQueue);
      metrics.gauge(
        `bulkhead.${this.name}.active`,
        this.config.maxConcurrentCalls - this.semaphore.getValue()
      );

      try {
        return await Promise.race([
          fn(),
          this.createTimeout(this.config.queueTimeoutMs),
        ]);
      } finally {
        acquired[1](); // release semaphore
        metrics.gauge(
          `bulkhead.${this.name}.active`,
          this.config.maxConcurrentCalls - this.semaphore.getValue()
        );
      }
    } catch (error) {
      if (error instanceof TimeoutError) {
        this.currentQueue--;
        metrics.increment(`bulkhead.${this.name}.timeouts`);
      }
      throw error;
    }
  }

  private createTimeout(ms: number): Promise<never> {
    return new Promise((_, reject) =>
      setTimeout(() => reject(new TimeoutError(`Bulkhead ${this.name} timeout`)), ms)
    );
  }

  get stats() {
    return {
      concurrentCalls: this.config.maxConcurrentCalls - this.semaphore.getValue(),
      maxConcurrentCalls: this.config.maxConcurrentCalls,
      queueSize: this.currentQueue,
      maxQueueSize: this.config.maxQueueSize,
    };
  }
}

// Thread pool bulkhead สำหรับ CPU-intensive tasks
class ThreadPoolBulkhead {
  private readonly workers: Worker[] = [];
  private readonly queue: Array<{
    work: () => Promise<unknown>;
    resolve: (v: unknown) => void;
    reject: (e: Error) => void;
  }> = [];
  private activeCount = 0;

  constructor(private readonly maxThreads: number) {}

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    return new Promise<T>((resolve, reject) => {
      this.queue.push({ work: fn as any, resolve: resolve as any, reject });
      this.processQueue();
    });
  }

  private async processQueue(): Promise<void> {
    if (this.activeCount >= this.maxThreads || this.queue.length === 0) {
      return;
    }

    const task = this.queue.shift()!;
    this.activeCount++;

    try {
      const result = await task.work();
      task.resolve(result);
    } catch (error) {
      task.reject(error as Error);
    } finally {
      this.activeCount--;
      this.processQueue();
    }
  }
}

// Separate bulkheads per service
const bulkheads = {
  payment: new Bulkhead("payment", {
    maxConcurrentCalls: 10,
    maxQueueSize: 20,
    queueTimeoutMs: 5000,
  }),
  inventory: new Bulkhead("inventory", {
    maxConcurrentCalls: 20,
    maxQueueSize: 40,
    queueTimeoutMs: 3000,
  }),
  notification: new Bulkhead("notification", {
    maxConcurrentCalls: 50,
    maxQueueSize: 100,
    queueTimeoutMs: 10000,
  }),
};

// Usage
async function checkout(order: Order): Promise<void> {
  // These run with separate bulkheads — inventory failure won't block payments
  const [inventory, payment] = await Promise.all([
    bulkheads.inventory.execute(() => inventoryService.reserve(order.items)),
    bulkheads.payment.execute(() => paymentService.charge(order.amount)),
  ]);
}
```

---

## 4. Timeout Patterns

```typescript
// resilience/timeout.ts
export function withTimeout<T>(
  promise: Promise<T>,
  timeoutMs: number,
  message?: string
): Promise<T> {
  const timeoutPromise = new Promise<never>((_, reject) =>
    setTimeout(
      () => reject(new TimeoutError(message ?? `Operation timed out after ${timeoutMs}ms`)),
      timeoutMs
    )
  );

  return Promise.race([promise, timeoutPromise]);
}

// AbortController timeout
export async function fetchWithTimeout(
  url: string,
  options: RequestInit & { timeoutMs: number }
): Promise<Response> {
  const { timeoutMs, ...fetchOptions } = options;
  const controller = new AbortController();

  const timeoutId = setTimeout(() => controller.abort(), timeoutMs);

  try {
    const response = await fetch(url, {
      ...fetchOptions,
      signal: controller.signal,
    });
    clearTimeout(timeoutId);
    return response;
  } catch (error) {
    clearTimeout(timeoutId);
    if ((error as Error).name === "AbortError") {
      throw new TimeoutError(`Request to ${url} timed out after ${timeoutMs}ms`);
    }
    throw error;
  }
}

// Hierarchical timeouts
class RequestContext {
  private readonly deadline: number;

  constructor(timeoutMs: number) {
    this.deadline = Date.now() + timeoutMs;
  }

  get remainingMs(): number {
    return Math.max(0, this.deadline - Date.now());
  }

  get isExpired(): boolean {
    return this.remainingMs === 0;
  }

  withTimeout<T>(fn: () => Promise<T>): Promise<T> {
    if (this.isExpired) {
      return Promise.reject(new TimeoutError("Request deadline exceeded"));
    }
    return withTimeout(fn(), this.remainingMs);
  }

  childContext(fraction: number): RequestContext {
    const ctx = new RequestContext(0);
    (ctx as any).deadline = this.deadline - (this.remainingMs * (1 - fraction));
    return ctx;
  }
}

// Usage with cascading timeouts
async function handleCheckout(orderId: string): Promise<void> {
  const ctx = new RequestContext(10000); // 10 second total budget

  // Inventory check gets 20% of budget
  const inventory = await ctx.withTimeout(
    () => inventoryService.check(orderId)
  );

  // Payment gets 60% of remaining budget
  const payment = await ctx.withTimeout(
    () => paymentService.charge(orderId)
  );

  // Notification gets remaining budget
  await ctx.withTimeout(
    () => notificationService.send(orderId)
  );
}
```

---

## 5. Fallback Strategies

```typescript
// resilience/fallback.ts
type FallbackStrategy<T> =
  | { type: "static"; value: T }
  | { type: "cached"; getCached: () => Promise<T | null>; maxAge?: number }
  | { type: "degraded"; getDegraded: () => Promise<T> }
  | { type: "queue"; enqueue: (args: unknown[]) => Promise<string> };

class ServiceWithFallback<T> {
  constructor(
    private readonly primaryFn: (...args: unknown[]) => Promise<T>,
    private readonly fallback: FallbackStrategy<T>,
    private readonly cache?: CacheService
  ) {}

  async execute(...args: unknown[]): Promise<T> {
    try {
      const result = await this.primaryFn(...args);

      // Cache successful results
      if (this.cache) {
        await this.cache.set(JSON.stringify(args), result, 300);
      }

      return result;
    } catch (error) {
      logger.warn("Primary failed, using fallback", { error: (error as Error).message });
      return this.executeFallback(args);
    }
  }

  private async executeFallback(args: unknown[]): Promise<T> {
    switch (this.fallback.type) {
      case "static":
        return this.fallback.value;

      case "cached": {
        const cached = await this.fallback.getCached();
        if (cached !== null) {
          metrics.increment("fallback.cache_hit");
          return cached;
        }
        throw new Error("No cached data available");
      }

      case "degraded":
        metrics.increment("fallback.degraded");
        return this.fallback.getDegraded();

      case "queue": {
        const jobId = await this.fallback.enqueue(args);
        metrics.increment("fallback.queued");
        throw new QueuedOperationError(jobId);
      }
    }
  }
}

// Product recommendations with fallback
const recommendationService = new ServiceWithFallback(
  (userId: string) => mlService.getRecommendations(userId),
  {
    type: "degraded",
    getDegraded: async () => {
      // Return popular products as fallback
      return productRepo.findTopSelling(10);
    },
  },
  cacheService
);

// Payment with queue fallback
const paymentService = new ServiceWithFallback(
  (chargeParams: ChargeParams) => stripeClient.charge(chargeParams),
  {
    type: "queue",
    enqueue: (args) => paymentQueue.add("charge", args[0]),
  }
);
```

---

## 6. Health Checks และ Self-healing

```typescript
// health/health-check.ts
interface HealthCheck {
  name: string;
  check: () => Promise<HealthStatus>;
  critical: boolean;
  timeout?: number;
}

interface HealthStatus {
  status: "healthy" | "degraded" | "unhealthy";
  message?: string;
  details?: Record<string, unknown>;
}

interface OverallHealth {
  status: "healthy" | "degraded" | "unhealthy";
  timestamp: string;
  checks: Record<string, HealthStatus & { duration: number }>;
}

export class HealthCheckManager {
  private readonly checks: HealthCheck[] = [];

  register(check: HealthCheck): void {
    this.checks.push(check);
  }

  async check(): Promise<OverallHealth> {
    const results: Record<string, HealthStatus & { duration: number }> = {};

    await Promise.all(
      this.checks.map(async (check) => {
        const start = Date.now();
        try {
          const status = await withTimeout(
            check.check(),
            check.timeout ?? 5000
          );
          results[check.name] = { ...status, duration: Date.now() - start };
        } catch (error) {
          results[check.name] = {
            status: "unhealthy",
            message: (error as Error).message,
            duration: Date.now() - start,
          };
        }
      })
    );

    const criticalUnhealthy = this.checks
      .filter((c) => c.critical)
      .some((c) => results[c.name]?.status === "unhealthy");

    const anyDegraded = Object.values(results).some(
      (r) => r.status === "degraded"
    );

    const overall = criticalUnhealthy
      ? "unhealthy"
      : anyDegraded
      ? "degraded"
      : "healthy";

    return {
      status: overall,
      timestamp: new Date().toISOString(),
      checks: results,
    };
  }
}

// Health check implementations
const dbHealthCheck: HealthCheck = {
  name: "database",
  critical: true,
  timeout: 3000,
  async check() {
    try {
      await pool.query("SELECT 1");
      const stats = pool as any;
      return {
        status: "healthy",
        details: {
          totalConnections: stats.totalCount,
          idleConnections: stats.idleCount,
          waitingClients: stats.waitingCount,
        },
      };
    } catch (error) {
      return {
        status: "unhealthy",
        message: (error as Error).message,
      };
    }
  },
};

const redisHealthCheck: HealthCheck = {
  name: "redis",
  critical: true,
  timeout: 2000,
  async check() {
    try {
      const start = Date.now();
      await redis.ping();
      const latency = Date.now() - start;

      return {
        status: latency > 100 ? "degraded" : "healthy",
        details: { latency: `${latency}ms` },
      };
    } catch (error) {
      return {
        status: "unhealthy",
        message: (error as Error).message,
      };
    }
  },
};

const externalServiceHealthCheck = (name: string, url: string): HealthCheck => ({
  name,
  critical: false,
  timeout: 5000,
  async check() {
    try {
      const start = Date.now();
      await axios.get(`${url}/health`, { timeout: 4000 });
      return {
        status: "healthy",
        details: { latency: `${Date.now() - start}ms` },
      };
    } catch {
      return { status: "degraded", message: "Service unavailable" };
    }
  },
});

// Express health check endpoint
function setupHealthEndpoints(
  app: Express,
  manager: HealthCheckManager
): void {
  // Kubernetes liveness probe — is the process alive?
  app.get("/healthz/live", (_req, res) => {
    res.status(200).json({ status: "alive" });
  });

  // Kubernetes readiness probe — can handle traffic?
  app.get("/healthz/ready", async (_req, res) => {
    const health = await manager.check();
    const statusCode = health.status === "unhealthy" ? 503 : 200;
    res.status(statusCode).json(health);
  });

  // Detailed health for monitoring
  app.get("/health", async (_req, res) => {
    const health = await manager.check();
    res.status(200).json(health);
  });
}

// Self-healing: auto-restart on repeated failures
class SelfHealingService {
  private failureCount = 0;
  private lastFailure?: Date;
  private readonly failureThreshold = 5;
  private readonly failureWindowMs = 60000;

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    try {
      const result = await fn();
      this.failureCount = 0;
      return result;
    } catch (error) {
      this.recordFailure();

      if (this.shouldRestart()) {
        logger.error("Too many failures, triggering restart");
        this.triggerRestart();
      }

      throw error;
    }
  }

  private recordFailure(): void {
    const now = new Date();
    if (
      this.lastFailure &&
      now.getTime() - this.lastFailure.getTime() > this.failureWindowMs
    ) {
      this.failureCount = 0;
    }
    this.failureCount++;
    this.lastFailure = now;
  }

  private shouldRestart(): boolean {
    return this.failureCount >= this.failureThreshold;
  }

  private triggerRestart(): void {
    setTimeout(() => process.exit(1), 1000); // Let Kubernetes restart the pod
  }
}
```

---

## 7. Chaos Engineering

```typescript
// chaos/chaos-middleware.ts
// ⚠️ ใช้เฉพาะใน non-production environments!

interface ChaosConfig {
  enabled: boolean;
  latencyMs?: { min: number; max: number };
  errorRate?: number;  // 0-1
  targetEndpoints?: string[];
}

export class ChaosMiddleware {
  constructor(private readonly config: ChaosConfig) {}

  middleware() {
    return async (req: Request, res: Response, next: NextFunction) => {
      if (!this.config.enabled) {
        return next();
      }

      // Check if this endpoint should be affected
      if (
        this.config.targetEndpoints &&
        !this.config.targetEndpoints.some((ep) => req.path.startsWith(ep))
      ) {
        return next();
      }

      // Inject latency
      if (this.config.latencyMs) {
        const { min, max } = this.config.latencyMs;
        const delay = Math.random() * (max - min) + min;
        await sleep(delay);
      }

      // Inject errors
      if (this.config.errorRate && Math.random() < this.config.errorRate) {
        logger.warn("Chaos: injecting error", { path: req.path });
        res.status(500).json({ error: "Chaos-injected error" });
        return;
      }

      next();
    };
  }
}

// Load testing script (k6)
// k6-load-test.js
/*
import http from "k6/http";
import { check, sleep } from "k6";
import { Rate } from "k6/metrics";

const errorRate = new Rate("errors");

export const options = {
  stages: [
    { duration: "30s", target: 20 },   // Ramp up
    { duration: "2m", target: 100 },    // Stay at 100 users
    { duration: "30s", target: 200 },   // Spike
    { duration: "1m", target: 100 },    // Settle
    { duration: "30s", target: 0 },     // Ramp down
  ],
  thresholds: {
    http_req_duration: ["p(95)<500"],   // 95% of requests < 500ms
    errors: ["rate<0.01"],              // Error rate < 1%
  },
};

export default function () {
  const response = http.get("http://localhost:3000/api/products");

  const result = check(response, {
    "status is 200": (r) => r.status === 200,
    "response time < 500ms": (r) => r.timings.duration < 500,
  });

  errorRate.add(!result);
  sleep(1);
}
*/
```

---

## 8. Docker Compose สำหรับ Testing Resilience

```yaml
# docker-compose.resilience-test.yml
version: "3.8"

services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=test
      - DATABASE_URL=postgresql://postgres:password@db:5432/testdb
      - REDIS_URL=redis://redis:6379
      - CHAOS_ENABLED=true
      - CHAOS_LATENCY_MIN=50
      - CHAOS_LATENCY_MAX=200
      - CHAOS_ERROR_RATE=0.05
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: on-failure:5
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/healthz/ready"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 30s

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_PASSWORD: password
      POSTGRES_DB: testdb
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 5
    tmpfs:
      - /var/lib/postgresql/data  # Use tmpfs for speed in tests

  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

  # Toxiproxy for simulating network failures
  toxiproxy:
    image: ghcr.io/shopify/toxiproxy:2.7.0
    ports:
      - "8474:8474"  # API port
      - "5432:5432"  # Proxy for postgres

  # Prometheus + Grafana for observing resilience behavior
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3001:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
```

---

## 9. Kubernetes Resilience Configuration

```yaml
# k8s/resilience/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: production
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0  # Zero downtime deployment
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      # Pod disruption budget
      terminationGracePeriodSeconds: 30

      # Anti-affinity: spread pods across nodes
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - order-service
              topologyKey: kubernetes.io/hostname

      containers:
        - name: order-service
          image: myregistry/order-service:1.0.0
          ports:
            - containerPort: 3000

          # Resource limits prevent one pod from starving others
          resources:
            requests:
              memory: "256Mi"
              cpu: "100m"
            limits:
              memory: "512Mi"
              cpu: "500m"

          # Liveness: restart if stuck
          livenessProbe:
            httpGet:
              path: /healthz/live
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3

          # Readiness: remove from load balancer if not ready
          readinessProbe:
            httpGet:
              path: /healthz/ready
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 2

          # Startup: give time to initialize
          startupProbe:
            httpGet:
              path: /healthz/live
              port: 3000
            failureThreshold: 30
            periodSeconds: 10

          env:
            - name: NODE_ENV
              value: "production"
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: order-service-secrets
                  key: database-url

          # Graceful shutdown
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 5"]

---
# PodDisruptionBudget: ป้องกันการ evict พร้อมกันหลาย pods
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: order-service-pdb
  namespace: production
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: order-service

---
# HorizontalPodAutoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 3
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
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5 min before scaling down
      policies:
        - type: Pods
          value: 1
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Percent
          value: 100
          periodSeconds: 15
```

---

## 10. Monitoring Resilience Metrics

```typescript
// monitoring/resilience-metrics.ts
import { register, Counter, Gauge, Histogram } from "prom-client";

export class ResilienceMetrics {
  private readonly circuitBreakerState: Gauge;
  private readonly retryAttempts: Counter;
  private readonly bulkheadRejections: Counter;
  private readonly fallbackUsed: Counter;
  private readonly timeoutErrors: Counter;
  private readonly requestDuration: Histogram;

  constructor() {
    this.circuitBreakerState = new Gauge({
      name: "circuit_breaker_state",
      help: "Circuit breaker state (0=closed, 1=open, 2=half-open)",
      labelNames: ["service"],
    });

    this.retryAttempts = new Counter({
      name: "retry_attempts_total",
      help: "Total retry attempts",
      labelNames: ["service", "result"],
    });

    this.bulkheadRejections = new Counter({
      name: "bulkhead_rejections_total",
      help: "Requests rejected by bulkhead",
      labelNames: ["bulkhead"],
    });

    this.fallbackUsed = new Counter({
      name: "fallback_used_total",
      help: "Times fallback was used",
      labelNames: ["service", "strategy"],
    });

    this.timeoutErrors = new Counter({
      name: "timeout_errors_total",
      help: "Total timeout errors",
      labelNames: ["operation"],
    });

    this.requestDuration = new Histogram({
      name: "request_duration_seconds",
      help: "Request duration",
      labelNames: ["method", "path", "status_code"],
      buckets: [0.01, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10],
    });
  }

  recordCircuitBreakerState(service: string, state: "closed" | "open" | "half-open"): void {
    const stateValue = { closed: 0, open: 1, "half-open": 2 }[state];
    this.circuitBreakerState.labels(service).set(stateValue);
  }

  recordRetry(service: string, result: "success" | "failure"): void {
    this.retryAttempts.labels(service, result).inc();
  }

  recordBulkheadRejection(bulkhead: string): void {
    this.bulkheadRejections.labels(bulkhead).inc();
  }

  recordFallback(service: string, strategy: string): void {
    this.fallbackUsed.labels(service, strategy).inc();
  }

  recordTimeout(operation: string): void {
    this.timeoutErrors.labels(operation).inc();
  }

  observeRequestDuration(
    method: string,
    path: string,
    statusCode: number,
    durationSeconds: number
  ): void {
    this.requestDuration
      .labels(method, path, String(statusCode))
      .observe(durationSeconds);
  }
}
```

---

## สรุป

ใน Part นี้เราได้เรียนรู้ Microservices Resilience Patterns ที่ครอบคลุม:

1. **Circuit Breaker** — ใช้ opossum ป้องกัน cascade failures โดย open circuit เมื่อ error rate สูง
2. **Retry with Exponential Backoff** — retry อัตโนมัติพร้อม jitter ป้องกัน thundering herd
3. **Bulkhead** — แยก resource pools เพื่อกัน failure isolation
4. **Timeout** — กำหนด deadline ให้ทุก operation ป้องกัน resource leak
5. **Fallback** — มี degraded mode เมื่อ service ไม่พร้อม
6. **Health Checks** — Liveness และ Readiness probes สำหรับ Kubernetes
7. **Self-healing** — auto-restart เมื่อ failure threshold ถูกเกิน
8. **Chaos Engineering** — ทดสอบ resilience โดยจำลอง failures จริง

Pattern เหล่านี้ทำงานร่วมกัน: Circuit Breaker + Retry + Fallback + Timeout = Complete resilience stack ที่พร้อม production

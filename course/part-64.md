# Part 64: Cloud-Native Patterns

## บทนำ

Cloud-Native Application ถูกออกแบบมาโดยเฉพาะเพื่อทำงานบน Cloud Infrastructure โดยใช้ประโยชน์จาก Scalability, Resilience และ Observability ของ Cloud บทนี้ครอบคลุม 12-Factor App, Sidecar Pattern, Ambassador Pattern, Adapter Pattern, Health Endpoint, Bulkhead Isolation และ Retry with Exponential Backoff

## 1. 12-Factor App Methodology

### 1.1 12-Factor Application Template

```typescript
// src/config/twelve-factor-config.ts
import { z } from 'zod';

// Factor 3: Config - Store config in environment
const configSchema = z.object({
  // Factor 1: Codebase
  GIT_COMMIT: z.string().default('unknown'),
  BUILD_DATE: z.string().default(new Date().toISOString()),
  
  // Factor 3: Config
  NODE_ENV: z.enum(['development', 'staging', 'production']).default('development'),
  PORT: z.string().default('3000').transform(Number),
  
  // Database (Factor 4: Backing Services)
  DATABASE_URL: z.string(),
  REDIS_URL: z.string(),
  KAFKA_BROKERS: z.string().transform(v => v.split(',')),
  
  // External Services
  PAYMENT_GATEWAY_URL: z.string().url(),
  PAYMENT_GATEWAY_API_KEY: z.string(),
  LINE_CHANNEL_ACCESS_TOKEN: z.string().optional(),
  SMS_API_KEY: z.string().optional(),
  
  // Factor 7: Port Binding
  HTTP_PORT: z.string().default('3000').transform(Number),
  METRICS_PORT: z.string().default('9090').transform(Number),
  
  // Observability
  LOG_LEVEL: z.enum(['debug', 'info', 'warn', 'error']).default('info'),
  JAEGER_ENDPOINT: z.string().url().optional(),
  
  // Feature Flags
  FEATURE_NEW_CHECKOUT: z.string().default('false').transform(v => v === 'true'),
  FEATURE_AI_RECOMMENDATION: z.string().default('false').transform(v => v === 'true'),
});

export type AppConfig = z.infer<typeof configSchema>;

export function loadConfig(): AppConfig {
  try {
    return configSchema.parse(process.env);
  } catch (error) {
    if (error instanceof z.ZodError) {
      console.error('Configuration validation failed:');
      error.errors.forEach(e => {
        console.error(`  ${e.path.join('.')}: ${e.message}`);
      });
      process.exit(1);
    }
    throw error;
  }
}

// Factor 6: Processes - Stateless Process
export class StatelessApplication {
  // ไม่เก็บ State ใน Process Memory ที่ต้องการระหว่าง Requests
  // ใช้ External Storage แทน (Redis, Database)
  
  // ✅ ถูกต้อง: เก็บ Session ใน Redis
  private sessionStore: any; // Redis-backed
  
  // ❌ ผิด: เก็บ State ใน Memory
  // private sessions = new Map(); // อย่าใช้วิธีนี้
  
  async processRequest(requestId: string, data: any) {
    // ประมวลผล Request โดยไม่มี Shared State ใน Memory
    const result = await this.computeResult(data);
    return result;
  }
  
  private async computeResult(data: any): Promise<any> {
    // Pure function - ผลลัพธ์ขึ้นอยู่กับ Input เท่านั้น
    return { processed: true, data };
  }
}
```

### 1.2 Docker ตามหลัก 12-Factor

```dockerfile
# Dockerfile - 12-Factor Compliant
FROM node:20-alpine AS base
WORKDIR /app

# Factor 2: Dependencies - Explicitly declare all dependencies
COPY package*.json ./
RUN npm ci --only=production

FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Factor 5: Build, release, run - Separate stages
FROM node:20-alpine AS production
WORKDIR /app

# Security: Non-root user
RUN addgroup -g 1001 -S app && adduser -S app -u 1001 -G app

COPY --from=builder --chown=app:app /app/dist ./dist
COPY --from=base --chown=app:app /app/node_modules ./node_modules

USER app

# Factor 3: Config - Read from environment (not baked in)
# No ENV statements for application config here
# Factor 7: Port Binding
EXPOSE 3000

# Factor 9: Disposability - Fast startup, graceful shutdown
STOPSIGNAL SIGTERM

CMD ["node", "dist/server.js"]
```

## 2. Sidecar Pattern

Sidecar Pattern แนบ Container เสริมเข้ากับ Main Application Container เพื่อเพิ่ม Functionality โดยไม่ต้องแก้ไข Code หลัก

### 2.1 Envoy Sidecar Configuration

```yaml
# kubernetes/sidecars/envoy-sidecar.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: envoy-config
  namespace: production
data:
  envoy.yaml: |
    static_resources:
      listeners:
      - name: listener_0
        address:
          socket_address:
            protocol: TCP
            address: 0.0.0.0
            port_value: 9901
        filter_chains:
        - filters:
          - name: envoy.filters.network.http_connection_manager
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
              stat_prefix: ingress_http
              access_log:
              - name: envoy.access_loggers.stdout
                typed_config:
                  "@type": type.googleapis.com/envoy.extensions.access_loggers.stream.v3.StdoutAccessLog
              http_filters:
              # Rate Limiting
              - name: envoy.filters.http.local_ratelimit
                typed_config:
                  "@type": type.googleapis.com/envoy.extensions.filters.http.local_ratelimit.v3.LocalRateLimit
                  stat_prefix: http_local_rate_limiter
                  token_bucket:
                    max_tokens: 1000
                    tokens_per_fill: 1000
                    fill_interval: 60s
                  filter_enabled:
                    default_value:
                      numerator: 100
                      denominator: HUNDRED
              # Retry Policy
              - name: envoy.filters.http.router
                typed_config:
                  "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
              route_config:
                name: local_route
                virtual_hosts:
                - name: local_service
                  domains: ["*"]
                  routes:
                  - match:
                      prefix: "/"
                    route:
                      cluster: local_service
                      retry_policy:
                        retry_on: "5xx,connect-failure,retriable-4xx"
                        num_retries: 3
                        per_try_timeout: 10s
                        retry_back_off:
                          base_interval: 0.25s
                          max_interval: 10s
      clusters:
      - name: local_service
        type: STATIC
        connect_timeout: 5s
        load_assignment:
          cluster_name: local_service
          endpoints:
          - lb_endpoints:
            - endpoint:
                address:
                  socket_address:
                    address: 127.0.0.1
                    port_value: 3000
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: production
spec:
  template:
    spec:
      containers:
      # Main Application Container
      - name: order-service
        image: registry.company.th/order-service:v1.0.0
        ports:
        - containerPort: 3000
          name: http
        env:
        - name: PORT
          value: "3000"
      
      # Sidecar: Envoy Proxy
      - name: envoy
        image: envoyproxy/envoy:v1.28.0
        ports:
        - containerPort: 9901
          name: envoy-http
        - containerPort: 9902
          name: envoy-admin
        args:
        - "--config-path"
        - "/etc/envoy/envoy.yaml"
        volumeMounts:
        - name: envoy-config
          mountPath: /etc/envoy
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
      
      # Sidecar: Log Forwarder (Fluent Bit)
      - name: fluent-bit
        image: fluent/fluent-bit:2.2
        volumeMounts:
        - name: logs
          mountPath: /var/log/app
        - name: fluent-bit-config
          mountPath: /fluent-bit/etc
        resources:
          requests:
            memory: "32Mi"
            cpu: "25m"
          limits:
            memory: "64Mi"
            cpu: "50m"
      
      volumes:
      - name: envoy-config
        configMap:
          name: envoy-config
      - name: logs
        emptyDir: {}
      - name: fluent-bit-config
        configMap:
          name: fluent-bit-config
```

## 3. Ambassador Pattern

Ambassador Pattern ช่วย Route Traffic ออกไปยัง External Services โดยมี Proxy Layer

### 3.1 Ambassador Proxy สำหรับ External Payment Gateway

```typescript
// src/patterns/ambassador.ts
import axios, { AxiosInstance, AxiosRequestConfig, AxiosResponse } from 'axios';
import { logger } from '../utils/logger';
import { MetricsCollector } from '../monitoring/metrics';
import { CircuitBreaker } from './circuit-breaker';

interface AmbassadorConfig {
  targetUrl: string;
  timeout: number;
  retries: number;
  circuitBreakerThreshold: number;
  circuitBreakerTimeout: number;
}

export class PaymentGatewayAmbassador {
  private httpClient: AxiosInstance;
  private circuitBreaker: CircuitBreaker;
  private metrics: MetricsCollector;

  constructor(private config: AmbassadorConfig, metrics: MetricsCollector) {
    this.metrics = metrics;
    
    this.httpClient = axios.create({
      baseURL: config.targetUrl,
      timeout: config.timeout,
    });

    this.circuitBreaker = new CircuitBreaker({
      threshold: config.circuitBreakerThreshold,
      timeout: config.circuitBreakerTimeout,
    });

    this.setupInterceptors();
  }

  private setupInterceptors(): void {
    // Request interceptor: เพิ่ม Authentication, Logging
    this.httpClient.interceptors.request.use((request) => {
      request.headers['X-Request-Id'] = crypto.randomUUID();
      request.headers['X-Client-Id'] = process.env.PAYMENT_CLIENT_ID;
      request.headers['Authorization'] = `Bearer ${process.env.PAYMENT_API_KEY}`;
      request.headers['X-Timestamp'] = Date.now().toString();
      
      logger.debug('Ambassador outbound request', {
        url: request.url,
        method: request.method,
      });
      
      return request;
    });

    // Response interceptor: Logging, Error Handling
    this.httpClient.interceptors.response.use(
      (response) => {
        this.metrics.histogram('ambassador.response_time', 
          response.config.metadata?.duration ?? 0,
          { target: 'payment_gateway' }
        );
        return response;
      },
      (error) => {
        this.metrics.increment('ambassador.error', {
          target: 'payment_gateway',
          status: error.response?.status?.toString() ?? 'network_error',
        });
        return Promise.reject(error);
      }
    );
  }

  async chargeCard(
    cardToken: string,
    amount: number,
    currency: string,
    orderId: string
  ): Promise<any> {
    return this.circuitBreaker.execute(async () => {
      const response = await this.withRetry(
        () => this.httpClient.post('/v1/charges', {
          source: cardToken,
          amount: Math.round(amount * 100), // สตางค์
          currency,
          metadata: { orderId },
        })
      );
      
      return response.data;
    });
  }

  async createRefund(chargeId: string, amount?: number): Promise<any> {
    return this.circuitBreaker.execute(async () => {
      const response = await this.withRetry(
        () => this.httpClient.post(`/v1/charges/${chargeId}/refund`, {
          amount: amount ? Math.round(amount * 100) : undefined,
        })
      );
      
      return response.data;
    });
  }

  private async withRetry<T>(
    fn: () => Promise<AxiosResponse<T>>,
    attempt = 0
  ): Promise<AxiosResponse<T>> {
    try {
      return await fn();
    } catch (error: any) {
      const isRetryable = 
        error.response?.status >= 500 ||
        error.code === 'ECONNABORTED' ||
        error.code === 'ETIMEDOUT';
      
      if (isRetryable && attempt < this.config.retries) {
        const delay = Math.pow(2, attempt) * 1000 + Math.random() * 500;
        
        logger.warn('Ambassador retrying request', {
          attempt: attempt + 1,
          maxRetries: this.config.retries,
          delayMs: delay,
        });
        
        await this.sleep(delay);
        return this.withRetry(fn, attempt + 1);
      }
      
      throw error;
    }
  }

  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}
```

## 4. Adapter Pattern

Adapter Pattern แปลง Interface ที่ไม่เข้ากันให้ทำงานร่วมกันได้

### 4.1 Payment Provider Adapter

```typescript
// src/patterns/adapter.ts

// Target Interface ที่ Application ต้องการ
interface UnifiedPaymentProvider {
  charge(params: ChargeParams): Promise<ChargeResult>;
  refund(transactionId: string, amount?: number): Promise<RefundResult>;
  getStatus(transactionId: string): Promise<TransactionStatus>;
}

interface ChargeParams {
  amount: number;  // บาท
  currency: string;
  cardToken?: string;
  promptPayId?: string;
  description: string;
  reference: string;
}

interface ChargeResult {
  transactionId: string;
  status: 'success' | 'pending' | 'failed';
  authCode?: string;
}

interface RefundResult {
  refundId: string;
  status: 'success' | 'pending';
}

interface TransactionStatus {
  status: 'success' | 'pending' | 'failed';
  amount: number;
}

// Adapter สำหรับ Omise (Payment Provider ไทย)
class OmiseAdapter implements UnifiedPaymentProvider {
  private omise: any; // Omise SDK

  constructor(secretKey: string) {
    // Initialize Omise
    this.omise = require('omise')({ secretKey });
  }

  async charge(params: ChargeParams): Promise<ChargeResult> {
    // Omise ใช้ สตางค์ (ไม่ใช่บาท)
    const omiseCharge = await this.omise.charges.create({
      amount: Math.round(params.amount * 100),
      currency: params.currency.toLowerCase(),
      card: params.cardToken,
      description: params.description,
      metadata: { reference: params.reference },
    });

    return {
      transactionId: omiseCharge.id,
      status: omiseCharge.status === 'successful' ? 'success' 
        : omiseCharge.status === 'pending' ? 'pending' : 'failed',
      authCode: omiseCharge.authorize_uri,
    };
  }

  async refund(transactionId: string, amount?: number): Promise<RefundResult> {
    const refund = await this.omise.charges.createRefund(transactionId, {
      amount: amount ? Math.round(amount * 100) : undefined,
    });

    return {
      refundId: refund.id,
      status: refund.status === 'closed' ? 'success' : 'pending',
    };
  }

  async getStatus(transactionId: string): Promise<TransactionStatus> {
    const charge = await this.omise.charges.retrieve(transactionId);
    
    return {
      status: charge.status === 'successful' ? 'success'
        : charge.status === 'pending' ? 'pending' : 'failed',
      amount: charge.amount / 100,
    };
  }
}

// Adapter สำหรับ 2C2P (Payment Provider ไทยอีกเจ้า)
class TwoCTwoPAdapter implements UnifiedPaymentProvider {
  constructor(
    private merchantId: string,
    private secretKey: string,
    private apiUrl: string
  ) {}

  async charge(params: ChargeParams): Promise<ChargeResult> {
    const payload = {
      merchantID: this.merchantId,
      invoiceNo: params.reference,
      description: params.description,
      amount: params.amount.toFixed(2),
      currencyCode: params.currency,
      // 2C2P format...
    };

    const response = await fetch(`${this.apiUrl}/PaymentAction`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(payload),
    });

    const data = await response.json();

    return {
      transactionId: data.tranRef,
      status: data.respCode === '0000' ? 'success' : 'failed',
    };
  }

  async refund(transactionId: string, amount?: number): Promise<RefundResult> {
    const response = await fetch(`${this.apiUrl}/Maintenance/refund`, {
      method: 'POST',
      body: JSON.stringify({
        merchantID: this.merchantId,
        invoiceNo: transactionId,
        actionAmount: amount?.toFixed(2),
      }),
    });

    const data = await response.json();
    return {
      refundId: data.tranRef,
      status: 'success',
    };
  }

  async getStatus(transactionId: string): Promise<TransactionStatus> {
    const response = await fetch(`${this.apiUrl}/Inquiry`, {
      method: 'POST',
      body: JSON.stringify({
        merchantID: this.merchantId,
        invoiceNo: transactionId,
      }),
    });

    const data = await response.json();
    
    return {
      status: data.respCode === '0000' ? 'success' : 'pending',
      amount: parseFloat(data.amount),
    };
  }
}

// Factory เพื่อสร้าง Adapter ตาม Provider
class PaymentProviderFactory {
  static create(provider: string): UnifiedPaymentProvider {
    switch (provider) {
      case 'omise':
        return new OmiseAdapter(process.env.OMISE_SECRET_KEY!);
      case '2c2p':
        return new TwoCTwoPAdapter(
          process.env.TWOC2P_MERCHANT_ID!,
          process.env.TWOC2P_SECRET_KEY!,
          process.env.TWOC2P_API_URL!
        );
      default:
        throw new Error(`Unknown payment provider: ${provider}`);
    }
  }
}
```

## 5. Health Endpoint Pattern

```typescript
// src/patterns/health-endpoint.ts
import { Router, Request, Response } from 'express';
import { Pool } from 'pg';
import Redis from 'ioredis';

interface DependencyCheck {
  name: string;
  check: () => Promise<{ status: 'up' | 'down' | 'degraded'; details?: any }>;
  critical: boolean;
}

export class HealthEndpointPattern {
  private checks: DependencyCheck[] = [];

  addCheck(check: DependencyCheck): void {
    this.checks.push(check);
  }

  createRouter(): Router {
    const router = Router();

    router.get('/health', async (req: Request, res: Response) => {
      const results = await this.runAllChecks();
      const overallStatus = this.determineOverallStatus(results);
      
      res.status(overallStatus === 'unhealthy' ? 503 : 200).json({
        status: overallStatus,
        timestamp: new Date().toISOString(),
        version: process.env.APP_VERSION ?? 'unknown',
        environment: process.env.NODE_ENV ?? 'unknown',
        dependencies: results,
      });
    });

    router.get('/health/live', (_req: Request, res: Response) => {
      res.json({ status: 'alive' });
    });

    router.get('/health/ready', async (_req: Request, res: Response) => {
      const criticalChecks = this.checks.filter(c => c.critical);
      const results = await Promise.allSettled(
        criticalChecks.map(async c => ({
          name: c.name,
          result: await c.check(),
        }))
      );

      const allReady = results.every(r => 
        r.status === 'fulfilled' && r.value.result.status === 'up'
      );

      res.status(allReady ? 200 : 503).json({
        status: allReady ? 'ready' : 'not_ready',
        checks: results.map(r => 
          r.status === 'fulfilled' 
            ? { name: r.value.name, status: r.value.result.status }
            : { name: 'unknown', status: 'down' }
        ),
      });
    });

    return router;
  }

  private async runAllChecks(): Promise<Record<string, any>> {
    const results: Record<string, any> = {};
    
    await Promise.allSettled(
      this.checks.map(async (check) => {
        const start = Date.now();
        try {
          const result = await check.check();
          results[check.name] = {
            ...result,
            latencyMs: Date.now() - start,
          };
        } catch (error) {
          results[check.name] = {
            status: 'down',
            error: (error as Error).message,
            latencyMs: Date.now() - start,
          };
        }
      })
    );
    
    return results;
  }

  private determineOverallStatus(
    results: Record<string, any>
  ): 'healthy' | 'degraded' | 'unhealthy' {
    const criticalNames = this.checks
      .filter(c => c.critical)
      .map(c => c.name);
    
    for (const name of criticalNames) {
      if (results[name]?.status === 'down') {
        return 'unhealthy';
      }
    }
    
    const anyDegraded = Object.values(results).some(r => r.status !== 'up');
    return anyDegraded ? 'degraded' : 'healthy';
  }
}

// ตัวอย่างการ Setup สำหรับ Payment Service
function setupHealthChecks(
  db: Pool,
  redis: Redis,
  healthService: HealthEndpointPattern
): void {
  healthService.addCheck({
    name: 'database',
    critical: true,
    check: async () => {
      await db.query('SELECT 1');
      const stats = await db.query(
        'SELECT count(*) as connections FROM pg_stat_activity'
      );
      return {
        status: 'up',
        details: {
          activeConnections: parseInt(stats.rows[0].connections),
          poolSize: db.totalCount,
        },
      };
    },
  });

  healthService.addCheck({
    name: 'redis',
    critical: true,
    check: async () => {
      await redis.ping();
      const info = await redis.info('server');
      const versionMatch = info.match(/redis_version:([^\r\n]+)/);
      return {
        status: 'up',
        details: {
          version: versionMatch?.[1]?.trim(),
        },
      };
    },
  });

  healthService.addCheck({
    name: 'payment_gateway',
    critical: false,
    check: async () => {
      try {
        const response = await fetch(
          `${process.env.PAYMENT_GATEWAY_URL}/health`,
          { signal: AbortSignal.timeout(3000) }
        );
        return {
          status: response.ok ? 'up' : 'degraded',
          details: { statusCode: response.status },
        };
      } catch {
        return { status: 'degraded' as const };
      }
    },
  });
}
```

## 6. Bulkhead Isolation Pattern

Bulkhead แยก Resources เพื่อป้องกันไม่ให้ Failure ของ Service หนึ่งกระทบ Services อื่น

### 6.1 Resource Pool Bulkhead

```typescript
// src/patterns/bulkhead.ts
import { Semaphore } from '../locks/async-mutex';
import { logger } from '../utils/logger';
import { MetricsCollector } from '../monitoring/metrics';

interface BulkheadConfig {
  maxConcurrent: number;
  maxQueue: number;
  queueTimeoutMs: number;
}

export class Bulkhead {
  private semaphore: Semaphore;
  private queueLength = 0;

  constructor(
    private name: string,
    private config: BulkheadConfig,
    private metrics: MetricsCollector
  ) {
    this.semaphore = new Semaphore(config.maxConcurrent);
  }

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    // ตรวจสอบ Queue ไม่เต็ม
    if (this.queueLength >= this.config.maxQueue) {
      this.metrics.increment('bulkhead.rejected', { name: this.name });
      throw new Error(
        `Bulkhead ${this.name} is full (${this.config.maxConcurrent} concurrent, ${this.config.maxQueue} queued)`
      );
    }

    this.queueLength++;
    
    try {
      // รอ Semaphore พร้อม Timeout
      const acquired = await Promise.race([
        this.semaphore.acquire().then(() => true),
        new Promise<false>(resolve => 
          setTimeout(() => resolve(false), this.config.queueTimeoutMs)
        ),
      ]);

      if (!acquired) {
        this.metrics.increment('bulkhead.timeout', { name: this.name });
        throw new Error(`Bulkhead ${this.name} queue timeout after ${this.config.queueTimeoutMs}ms`);
      }

      this.metrics.increment('bulkhead.executing', { name: this.name });
      
      try {
        return await fn();
      } finally {
        this.semaphore.release();
        this.metrics.decrement('bulkhead.executing', { name: this.name });
      }
    } finally {
      this.queueLength--;
    }
  }
}

// Bulkhead Manager สำหรับ Multiple Services
export class BulkheadManager {
  private bulkheads = new Map<string, Bulkhead>();

  constructor(private metrics: MetricsCollector) {
    // สร้าง Bulkhead สำหรับแต่ละ External Dependency
    this.register('payment_gateway', {
      maxConcurrent: 10,  // สูงสุด 10 concurrent calls
      maxQueue: 20,
      queueTimeoutMs: 5000,
    });

    this.register('database', {
      maxConcurrent: 20,
      maxQueue: 50,
      queueTimeoutMs: 3000,
    });

    this.register('sms_service', {
      maxConcurrent: 5,
      maxQueue: 10,
      queueTimeoutMs: 10000,
    });

    this.register('line_api', {
      maxConcurrent: 5,
      maxQueue: 10,
      queueTimeoutMs: 10000,
    });
  }

  register(name: string, config: BulkheadConfig): void {
    this.bulkheads.set(name, new Bulkhead(name, config, this.metrics));
  }

  get(name: string): Bulkhead {
    const bulkhead = this.bulkheads.get(name);
    if (!bulkhead) {
      throw new Error(`Bulkhead not found: ${name}`);
    }
    return bulkhead;
  }
}
```

## 7. Retry with Exponential Backoff and Jitter

### 7.1 Robust Retry Implementation

```typescript
// src/patterns/retry.ts
import { logger } from '../utils/logger';
import { MetricsCollector } from '../monitoring/metrics';

type RetryableError = (error: Error) => boolean;

interface RetryConfig {
  maxAttempts: number;
  baseDelayMs: number;
  maxDelayMs: number;
  jitterMs: number;
  backoffMultiplier: number;
  isRetryable?: RetryableError;
  onRetry?: (attempt: number, error: Error, nextDelayMs: number) => void;
}

export class RetryWithBackoff {
  constructor(
    private config: RetryConfig,
    private metrics?: MetricsCollector
  ) {}

  async execute<T>(
    fn: () => Promise<T>,
    operationName = 'operation'
  ): Promise<T> {
    let lastError: Error = new Error('No attempts made');
    
    for (let attempt = 1; attempt <= this.config.maxAttempts; attempt++) {
      try {
        const result = await fn();
        
        if (attempt > 1) {
          logger.info(`${operationName} succeeded after ${attempt} attempts`);
          this.metrics?.increment('retry.success', {
            operation: operationName,
            attempts: attempt.toString(),
          });
        }
        
        return result;
        
      } catch (error) {
        lastError = error as Error;
        
        if (attempt === this.config.maxAttempts) {
          break;
        }

        // ตรวจสอบว่า Error สามารถ Retry ได้หรือไม่
        if (this.config.isRetryable && !this.config.isRetryable(lastError)) {
          logger.warn(`${operationName} failed with non-retryable error`, {
            error: lastError.message,
          });
          break;
        }

        const delay = this.calculateDelay(attempt);
        
        logger.warn(`${operationName} attempt ${attempt} failed, retrying in ${delay}ms`, {
          error: lastError.message,
          attempt,
          nextDelay: delay,
        });

        this.config.onRetry?.(attempt, lastError, delay);
        this.metrics?.increment('retry.attempt', {
          operation: operationName,
          attempt: attempt.toString(),
        });

        await this.sleep(delay);
      }
    }

    this.metrics?.increment('retry.exhausted', {
      operation: operationName,
      attempts: this.config.maxAttempts.toString(),
    });

    throw lastError;
  }

  private calculateDelay(attempt: number): number {
    // Exponential Backoff: baseDelay * multiplier^(attempt-1)
    const exponentialDelay = 
      this.config.baseDelayMs * Math.pow(this.config.backoffMultiplier, attempt - 1);
    
    // ไม่เกิน Max Delay
    const cappedDelay = Math.min(exponentialDelay, this.config.maxDelayMs);
    
    // เพิ่ม Full Jitter เพื่อป้องกัน Thundering Herd Problem
    const jitter = Math.random() * this.config.jitterMs;
    
    return Math.floor(cappedDelay + jitter);
  }

  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

// Default Configurations สำหรับ Use Cases ต่างๆ
export const retryConfigs = {
  // สำหรับ External Payment API (Fast retry)
  paymentApi: {
    maxAttempts: 3,
    baseDelayMs: 200,
    maxDelayMs: 5000,
    jitterMs: 100,
    backoffMultiplier: 2,
    isRetryable: (error: Error) => {
      // อย่า Retry ถ้าเป็น Business Error (400, 422)
      const status = (error as any).response?.status;
      return !status || status >= 500 || status === 429;
    },
  },
  
  // สำหรับ Database Operations
  database: {
    maxAttempts: 5,
    baseDelayMs: 100,
    maxDelayMs: 10000,
    jitterMs: 200,
    backoffMultiplier: 2.5,
    isRetryable: (error: Error) => {
      // Retry เฉพาะ Connection Errors
      return error.message.includes('connection') || 
             error.message.includes('timeout') ||
             (error as any).code === 'ECONNRESET';
    },
  },

  // สำหรับ Message Queue Publishing
  messageQueue: {
    maxAttempts: 10,
    baseDelayMs: 500,
    maxDelayMs: 30000,
    jitterMs: 500,
    backoffMultiplier: 2,
    isRetryable: () => true, // Retry เสมอสำหรับ Message Queue
  },
};

// ตัวอย่าง: ใช้ Retry ในการชำระเงิน
class PaymentService {
  private retrier: RetryWithBackoff;

  constructor(private paymentGateway: any, metrics: MetricsCollector) {
    this.retrier = new RetryWithBackoff(retryConfigs.paymentApi, metrics);
  }

  async processPayment(params: any): Promise<any> {
    return this.retrier.execute(
      async () => {
        const result = await this.paymentGateway.charge(params);
        
        if (result.status === 'failed') {
          // ไม่ Retry ถ้า Gateway บอกว่า Failed (เช่น บัตรหมดอายุ)
          const err = new Error(`Payment failed: ${result.errorMessage}`);
          (err as any).isBusinessError = true;
          throw err;
        }
        
        return result;
      },
      'process_payment'
    );
  }
}
```

## 8. Circuit Breaker Pattern

```typescript
// src/patterns/circuit-breaker.ts

type CircuitState = 'closed' | 'open' | 'half_open';

interface CircuitBreakerConfig {
  threshold: number;         // จำนวน Failure ก่อน Open
  timeout: number;           // ms ก่อน Half-Open
  halfOpenRequests: number;  // จำนวน Requests ใน Half-Open state
  successThreshold: number;  // จำนวน Success ก่อน Close
}

export class CircuitBreaker {
  private state: CircuitState = 'closed';
  private failureCount = 0;
  private successCount = 0;
  private lastFailureTime = 0;
  private halfOpenRequestCount = 0;

  constructor(private config: CircuitBreakerConfig) {}

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === 'open') {
      // ตรวจสอบว่า Timeout ผ่านไปหรือยัง
      if (Date.now() - this.lastFailureTime >= this.config.timeout) {
        this.transitionTo('half_open');
      } else {
        throw new Error('Circuit breaker is OPEN - requests are rejected');
      }
    }

    if (this.state === 'half_open') {
      if (this.halfOpenRequestCount >= this.config.halfOpenRequests) {
        throw new Error('Circuit breaker is HALF_OPEN - max probe requests reached');
      }
      this.halfOpenRequestCount++;
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  private onSuccess(): void {
    if (this.state === 'half_open') {
      this.successCount++;
      if (this.successCount >= this.config.successThreshold) {
        this.transitionTo('closed');
      }
    } else {
      this.failureCount = 0;
    }
  }

  private onFailure(): void {
    this.failureCount++;
    this.lastFailureTime = Date.now();

    if (
      this.state === 'half_open' ||
      this.failureCount >= this.config.threshold
    ) {
      this.transitionTo('open');
    }
  }

  private transitionTo(state: CircuitState): void {
    logger.warn(`Circuit breaker state transition: ${this.state} -> ${state}`);
    this.state = state;
    
    if (state === 'closed') {
      this.failureCount = 0;
      this.successCount = 0;
    }
    
    if (state === 'half_open') {
      this.halfOpenRequestCount = 0;
      this.successCount = 0;
    }
  }

  getState(): CircuitState {
    return this.state;
  }
}
```

## สรุป

| Pattern | หลักการ | ประโยชน์ | เหมาะกับ |
|---------|---------|---------|---------|
| 12-Factor App | Methodology | Portable, Scalable | ทุก Cloud-Native App |
| Sidecar | แนบ Container เสริม | ไม่แก้ Application Code | Logging, Proxy, Config |
| Ambassador | Proxy ออก External | Centralized outbound control | External API calls |
| Adapter | แปลง Interface | Interoperability | Legacy integration |
| Health Endpoint | Expose Health Status | Kubernetes Integration | ทุก Microservice |
| Bulkhead | แยก Resource Pools | Fault Isolation | Critical Services |
| Retry + Backoff | ลอง Request ซ้ำ | Transient fault tolerance | Network calls |
| Circuit Breaker | ป้องกัน Cascade Failure | System Stability | External Dependencies |

Cloud-Native Patterns ช่วยให้ระบบมี Resilience สูงขึ้น สำหรับระบบ E-commerce และ FinTech ไทยที่มีการใช้งานสูงในช่วง Flash Sale หรือวันหยุด การใช้ Bulkhead และ Circuit Breaker จะช่วยป้องกันไม่ให้ระบบ Payment ล้มเหลวแม้ระบบอื่นจะมีปัญหา

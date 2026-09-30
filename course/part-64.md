# Part 64: Cloud-Native Patterns

## บทนำ

ใน Part นี้เราจะเรียนรู้ Cloud-Native Patterns ที่สำคัญสำหรับการพัฒนา Microservices บน Kubernetes ตั้งแต่หลักการ Twelve-Factor App ไปจนถึง Sidecar, Ambassador, Adapter Patterns, Init Containers, Pod Lifecycle Hooks, Graceful Shutdown, Health Checks, Resource Management และ Container Image Best Practices

---

## 1. Twelve-Factor App Principles กับ Node.js

Twelve-Factor App คือชุดหลักการสำหรับสร้าง Software-as-a-Service (SaaS) ที่ทำงานได้ดีบน Cloud ใน 12 ข้อ

### Factor I-III: Codebase, Dependencies, Config

```typescript
// src/config/environment.ts
// Factor III: Store config in the environment — ทุก config ต้องมาจาก env vars
import * as dotenv from 'dotenv';
dotenv.config();

interface AppEnvironment {
  PORT: number;
  HOST: string;
  NODE_ENV: string;
  DATABASE_URL: string;
  DATABASE_POOL_MIN: number;
  DATABASE_POOL_MAX: number;
  REDIS_URL: string;
  REDIS_TTL: number;
  USER_SERVICE_URL: string;
  PAYMENT_SERVICE_URL: string;
  JWT_SECRET: string;
  ENCRYPTION_KEY: string;
}

function getEnv(key: string, defaultValue?: string): string {
  const value = process.env[key] ?? defaultValue;
  if (value === undefined) {
    throw new Error(`Missing required environment variable: ${key}`);
  }
  return value;
}

function getEnvNumber(key: string, defaultValue?: number): number {
  const value = process.env[key];
  if (value === undefined) {
    if (defaultValue !== undefined) return defaultValue;
    throw new Error(`Missing required environment variable: ${key}`);
  }
  const num = parseInt(value, 10);
  if (isNaN(num)) {
    throw new Error(`Environment variable ${key} must be a number, got: ${value}`);
  }
  return num;
}

export const env: AppEnvironment = {
  PORT: getEnvNumber('PORT', 3000),
  HOST: getEnv('HOST', '0.0.0.0'),
  NODE_ENV: getEnv('NODE_ENV', 'development'),
  DATABASE_URL: getEnv('DATABASE_URL'),
  DATABASE_POOL_MIN: getEnvNumber('DATABASE_POOL_MIN', 2),
  DATABASE_POOL_MAX: getEnvNumber('DATABASE_POOL_MAX', 10),
  REDIS_URL: getEnv('REDIS_URL', 'redis://localhost:6379'),
  REDIS_TTL: getEnvNumber('REDIS_TTL', 3600),
  USER_SERVICE_URL: getEnv('USER_SERVICE_URL', 'http://user-service:3001'),
  PAYMENT_SERVICE_URL: getEnv('PAYMENT_SERVICE_URL', 'http://payment-service:3002'),
  JWT_SECRET: getEnv('JWT_SECRET'),
  ENCRYPTION_KEY: getEnv('ENCRYPTION_KEY'),
};
```

```json
// package.json — Factor II: Declare and isolate dependencies
{
  "name": "order-service",
  "version": "1.0.0",
  "engines": {
    "node": ">=20.0.0",
    "npm": ">=10.0.0"
  },
  "dependencies": {
    "express": "^4.18.2",
    "pg": "^8.11.3",
    "redis": "^4.6.10",
    "dotenv": "^16.3.1",
    "prom-client": "^15.1.0"
  },
  "devDependencies": {
    "typescript": "^5.3.2",
    "@types/express": "^4.17.21",
    "@types/node": "^20.10.0",
    "jest": "^29.7.0"
  },
  "scripts": {
    "build": "tsc",
    "start": "node dist/server.js",
    "dev": "ts-node src/server.ts",
    "test": "jest"
  }
}
```

### Factor IV-VI: Backing Services, Build/Release/Run, Processes

```typescript
// src/infrastructure/database.ts
// Factor IV: Treat backing services as attached resources
import { Pool } from 'pg';
import { createClient } from 'redis';
import { env } from '../config/environment';

export const db = new Pool({
  connectionString: env.DATABASE_URL,
  min: env.DATABASE_POOL_MIN,
  max: env.DATABASE_POOL_MAX,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});

export const redis = createClient({ url: env.REDIS_URL });

redis.on('error', (err) => console.error('Redis Client Error:', err));

export async function connectBackingServices(): Promise<void> {
  await redis.connect();
  const client = await db.connect();
  client.release();
  console.log('All backing services connected');
}
```

```typescript
// src/services/session.service.ts
// Factor VI: Execute the app as stateless processes
// State ทั้งหมดเก็บใน Redis ไม่เก็บใน memory ของ process
import { redis } from '../infrastructure/database';

export class SessionService {
  private readonly SESSION_PREFIX = 'session:';
  private readonly TTL = 3600;

  async createSession(userId: string, data: Record<string, unknown>): Promise<string> {
    const sessionId = crypto.randomUUID();
    const key = `${this.SESSION_PREFIX}${sessionId}`;
    await redis.setEx(key, this.TTL, JSON.stringify({
      userId,
      data,
      createdAt: new Date().toISOString(),
    }));
    return sessionId;
  }

  async getSession(sessionId: string): Promise<Record<string, unknown> | null> {
    const key = `${this.SESSION_PREFIX}${sessionId}`;
    const value = await redis.get(key);
    return value ? JSON.parse(value) : null;
  }

  async destroySession(sessionId: string): Promise<void> {
    await redis.del(`${this.SESSION_PREFIX}${sessionId}`);
  }
}
```

### Factor VII-XI: Port Binding, Concurrency, Disposability, Dev/Prod Parity, Logs

```typescript
// src/server.ts
// Factor VII: Export services via port binding
// Factor XI: Logs as event streams — เขียนไปที่ stdout เสมอ
import express from 'express';
import { env } from './config/environment';

const app = express();
app.use(express.json());

// Factor XI: Structured logging to stdout — ไม่เขียนลงไฟล์
function log(level: string, message: string, meta?: Record<string, unknown>): void {
  const entry = {
    timestamp: new Date().toISOString(),
    level,
    message,
    service: 'order-service',
    pid: process.pid,
    ...meta,
  };
  process.stdout.write(JSON.stringify(entry) + '\n');
}

app.get('/health/live', (req, res) => {
  res.json({ status: 'ok', timestamp: new Date().toISOString() });
});

// Factor VII: Port Binding — listen on configured port
const PORT = env.PORT;
const server = app.listen(PORT, env.HOST, () => {
  log('info', 'Server started', { port: PORT, host: env.HOST });
});

export { app, server, log };
```

```dockerfile
# Dockerfile — Factor V: Strictly separate build and run stages
# Build stage
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Production deps stage
FROM node:20-alpine AS prod-deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev

# Runtime stage
FROM gcr.io/distroless/nodejs20-debian12 AS runtime
WORKDIR /app
USER nonroot:nonroot
COPY --from=prod-deps --chown=nonroot:nonroot /app/node_modules ./node_modules
COPY --from=builder --chown=nonroot:nonroot /app/dist ./dist
COPY --chown=nonroot:nonroot package.json ./
EXPOSE 3000
CMD ["dist/server.js"]
```

---

## 2. Sidecar Pattern

Sidecar Pattern คือการแนบ container เสริมเข้ากับ main container ใน Pod เดียวกัน เพื่อให้บริการเสริม เช่น logging, monitoring, proxying โดยไม่ต้องแก้ main app

### 2.1 Envoy Proxy Sidecar

```yaml
# kubernetes/sidecar-envoy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9901"
        prometheus.io/path: "/stats/prometheus"
    spec:
      containers:
        # Main Application
        - name: order-service
          image: order-service:1.0.0
          ports:
            - containerPort: 3000
              name: http
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"

        # Envoy Proxy Sidecar
        - name: envoy-proxy
          image: envoyproxy/envoy:v1.28-latest
          args: ["--config-path", "/etc/envoy/envoy.yaml"]
          ports:
            - containerPort: 8080
              name: envoy-http
            - containerPort: 9901
              name: envoy-admin
          volumeMounts:
            - name: envoy-config
              mountPath: /etc/envoy
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"
            limits:
              cpu: "200m"
              memory: "256Mi"

      volumes:
        - name: envoy-config
          configMap:
            name: envoy-config
```

```yaml
# kubernetes/envoy-configmap.yaml
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
              address: 0.0.0.0
              port_value: 8080
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
                    route_config:
                      name: local_route
                      virtual_hosts:
                        - name: backend
                          domains: ["*"]
                          routes:
                            - match:
                                prefix: "/"
                              route:
                                cluster: order_service_cluster
                                timeout: 30s
                                retry_policy:
                                  retry_on: "5xx,gateway-error,connect-failure"
                                  num_retries: 3
                                  per_try_timeout: 10s
                    http_filters:
                      - name: envoy.filters.http.router
                        typed_config:
                          "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router

      clusters:
        - name: order_service_cluster
          connect_timeout: 5s
          type: STATIC
          load_assignment:
            cluster_name: order_service_cluster
            endpoints:
              - lb_endpoints:
                  - endpoint:
                      address:
                        socket_address:
                          address: 127.0.0.1
                          port_value: 3000
          health_checks:
            - timeout: 1s
              interval: 5s
              unhealthy_threshold: 3
              healthy_threshold: 2
              http_health_check:
                path: "/health/live"

    admin:
      address:
        socket_address:
          address: 0.0.0.0
          port_value: 9901
```

### 2.2 Logging Sidecar ด้วย Fluentd

```yaml
# kubernetes/sidecar-fluentd.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
  namespace: production
spec:
  replicas: 2
  selector:
    matchLabels:
      app: payment-service
  template:
    spec:
      containers:
        - name: payment-service
          image: payment-service:1.0.0
          env:
            - name: LOG_FILE
              value: "/var/log/app/payment.log"
          volumeMounts:
            - name: app-logs
              mountPath: /var/log/app

        # Fluentd sidecar อ่าน logs จาก shared volume
        - name: fluentd
          image: fluent/fluentd-kubernetes-daemonset:v1.16-debian-elasticsearch8-1
          env:
            - name: FLUENT_ELASTICSEARCH_HOST
              value: "elasticsearch.logging.svc.cluster.local"
            - name: FLUENT_ELASTICSEARCH_PORT
              value: "9200"
            - name: FLUENT_ELASTICSEARCH_SCHEME
              value: "http"
            - name: FLUENTD_SYSTEMD_CONF
              value: "disable"
          volumeMounts:
            - name: app-logs
              mountPath: /var/log/app
              readOnly: true
            - name: fluentd-config
              mountPath: /fluentd/etc
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"
            limits:
              cpu: "200m"
              memory: "256Mi"

      volumes:
        - name: app-logs
          emptyDir: {}
        - name: fluentd-config
          configMap:
            name: fluentd-config
```

```yaml
# kubernetes/fluentd-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
  namespace: production
data:
  fluent.conf: |
    <source>
      @type tail
      path /var/log/app/*.log
      pos_file /var/log/fluentd/app.log.pos
      tag app.logs
      read_from_head true
      <parse>
        @type json
        time_key timestamp
        time_format %Y-%m-%dT%H:%M:%S.%NZ
      </parse>
    </source>

    <filter app.logs>
      @type record_transformer
      <record>
        kubernetes_pod "#{ENV['MY_POD_NAME']}"
        kubernetes_namespace "#{ENV['MY_POD_NAMESPACE']}"
      </record>
    </filter>

    <match app.logs>
      @type elasticsearch
      host "#{ENV['FLUENT_ELASTICSEARCH_HOST']}"
      port "#{ENV['FLUENT_ELASTICSEARCH_PORT']}"
      scheme "#{ENV['FLUENT_ELASTICSEARCH_SCHEME']}"
      index_name app-logs
      type_name _doc
      include_timestamp true
      <buffer>
        @type file
        path /var/log/fluentd/buffer
        flush_interval 5s
        retry_max_times 5
      </buffer>
    </match>
```

### 2.3 Metrics Sidecar ด้วย Prometheus Exporter

```typescript
// sidecars/metrics-exporter/src/index.ts
import express from 'express';
import { register, Counter, Gauge, Histogram, collectDefaultMetrics } from 'prom-client';

const app = express();

// เก็บ default Node.js metrics
collectDefaultMetrics({ prefix: 'nodejs_' });

export const httpRequestsTotal = new Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status_code'],
});

export const httpRequestDuration = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration in seconds',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1, 5],
});

export const activeConnections = new Gauge({
  name: 'active_connections',
  help: 'Number of active connections',
});

export const queueDepth = new Gauge({
  name: 'queue_depth',
  help: 'Current queue depth',
  labelNames: ['queue_name'],
});

app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});

app.get('/health', (req, res) => res.json({ status: 'ok' }));

const METRICS_PORT = parseInt(process.env.METRICS_PORT ?? '9090', 10);
app.listen(METRICS_PORT, () => {
  console.log(`Metrics server listening on port ${METRICS_PORT}`);
});
```

---

## 3. Ambassador Pattern

Ambassador Pattern คือ Proxy Container ที่ทำงานแทน main application ในการสื่อสารกับ external services ช่วยให้ main app ไม่ต้องรู้รายละเอียดของ external service เช่น retry, circuit breaker, service discovery

### 3.1 TypeScript Ambassador Proxy Class

```typescript
// src/ambassador/ambassador.proxy.ts
import http from 'http';
import https from 'https';
import { URL } from 'url';

interface RetryConfig {
  maxAttempts: number;
  initialDelayMs: number;
  maxDelayMs: number;
  backoffMultiplier: number;
}

interface CircuitBreakerConfig {
  failureThreshold: number;
  successThreshold: number;
  timeoutMs: number;
}

enum CircuitState {
  CLOSED = 'CLOSED',
  OPEN = 'OPEN',
  HALF_OPEN = 'HALF_OPEN',
}

interface AmbassadorOptions {
  targetUrl: string;
  retry: RetryConfig;
  circuitBreaker: CircuitBreakerConfig;
  timeout: number;
}

export class AmbassadorProxy {
  private readonly targetUrl: URL;
  private readonly retryConfig: RetryConfig;
  private readonly cbConfig: CircuitBreakerConfig;
  private readonly timeout: number;

  private cbState: CircuitState = CircuitState.CLOSED;
  private failureCount = 0;
  private successCount = 0;
  private lastFailureTime = 0;

  constructor(options: AmbassadorOptions) {
    this.targetUrl = new URL(options.targetUrl);
    this.retryConfig = options.retry;
    this.cbConfig = options.circuitBreaker;
    this.timeout = options.timeout;
  }

  private checkCircuitBreaker(): void {
    if (this.cbState === CircuitState.OPEN) {
      const now = Date.now();
      if (now - this.lastFailureTime >= this.cbConfig.timeoutMs) {
        this.cbState = CircuitState.HALF_OPEN;
        this.successCount = 0;
        console.log('[Ambassador] Circuit Breaker: OPEN -> HALF_OPEN');
      } else {
        throw new Error('Circuit Breaker is OPEN — request rejected');
      }
    }
  }

  private recordSuccess(): void {
    this.failureCount = 0;
    if (this.cbState === CircuitState.HALF_OPEN) {
      this.successCount++;
      if (this.successCount >= this.cbConfig.successThreshold) {
        this.cbState = CircuitState.CLOSED;
        console.log('[Ambassador] Circuit Breaker: HALF_OPEN -> CLOSED');
      }
    }
  }

  private recordFailure(): void {
    this.failureCount++;
    this.lastFailureTime = Date.now();
    if (this.cbState === CircuitState.HALF_OPEN ||
        this.failureCount >= this.cbConfig.failureThreshold) {
      this.cbState = CircuitState.OPEN;
      console.log(`[Ambassador] Circuit Breaker: -> OPEN (failures: ${this.failureCount})`);
    }
  }

  private async makeRequest(
    path: string,
    method: string,
    headers: Record<string, string>,
    body?: string,
  ): Promise<{ statusCode: number; body: string; headers: Record<string, string> }> {
    return new Promise((resolve, reject) => {
      const url = new URL(path, this.targetUrl);
      const isHttps = url.protocol === 'https:';
      const transport = isHttps ? https : http;

      const options: http.RequestOptions = {
        hostname: url.hostname,
        port: url.port || (isHttps ? 443 : 80),
        path: url.pathname + url.search,
        method,
        headers: {
          ...headers,
          'Content-Type': 'application/json',
          'User-Agent': 'AmbassadorProxy/1.0',
        },
        timeout: this.timeout,
      };

      const req = transport.request(options, (res) => {
        let data = '';
        res.on('data', (chunk) => (data += chunk));
        res.on('end', () => {
          resolve({
            statusCode: res.statusCode ?? 0,
            body: data,
            headers: res.headers as Record<string, string>,
          });
        });
      });

      req.on('timeout', () => {
        req.destroy();
        reject(new Error(`Request timeout after ${this.timeout}ms`));
      });

      req.on('error', reject);
      if (body) req.write(body);
      req.end();
    });
  }

  async request(
    path: string,
    method = 'GET',
    headers: Record<string, string> = {},
    body?: unknown,
  ): Promise<{ statusCode: number; body: unknown }> {
    this.checkCircuitBreaker();

    const serializedBody = body ? JSON.stringify(body) : undefined;
    let lastError: Error | null = null;
    let delay = this.retryConfig.initialDelayMs;

    for (let attempt = 1; attempt <= this.retryConfig.maxAttempts; attempt++) {
      try {
        const response = await this.makeRequest(path, method, headers, serializedBody);
        if (response.statusCode >= 500) {
          throw new Error(`Server error: ${response.statusCode}`);
        }
        this.recordSuccess();
        return { statusCode: response.statusCode, body: JSON.parse(response.body) };
      } catch (error) {
        lastError = error as Error;
        console.warn(`[Ambassador] Attempt ${attempt} failed: ${lastError.message}`);
        if (attempt < this.retryConfig.maxAttempts) {
          await new Promise((resolve) => setTimeout(resolve, delay));
          delay = Math.min(delay * this.retryConfig.backoffMultiplier, this.retryConfig.maxDelayMs);
        }
      }
    }

    this.recordFailure();
    throw lastError ?? new Error('Request failed after all retries');
  }

  getCircuitState(): CircuitState { return this.cbState; }
}

export function createUserServiceAmbassador(): AmbassadorProxy {
  return new AmbassadorProxy({
    targetUrl: process.env.USER_SERVICE_URL ?? 'http://user-service:3001',
    timeout: 5000,
    retry: {
      maxAttempts: 3,
      initialDelayMs: 100,
      maxDelayMs: 2000,
      backoffMultiplier: 2,
    },
    circuitBreaker: {
      failureThreshold: 5,
      successThreshold: 2,
      timeoutMs: 30000,
    },
  });
}
```

```yaml
# kubernetes/ambassador-sidecar.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-service
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-service
  template:
    spec:
      containers:
        - name: api-service
          image: api-service:1.0.0
          env:
            # ชี้ไปที่ ambassador sidecar บน localhost
            - name: DATABASE_HOST
              value: "127.0.0.1"
            - name: DATABASE_PORT
              value: "5432"

        # Ambassador: PgBouncer connection pooler
        - name: pgbouncer-ambassador
          image: pgbouncer/pgbouncer:1.21.0
          ports:
            - containerPort: 5432
          env:
            - name: DATABASES_HOST
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: host
            - name: DATABASES_PORT
              value: "5432"
            - name: DATABASES_DBNAME
              value: "orders"
          volumeMounts:
            - name: pgbouncer-config
              mountPath: /etc/pgbouncer
      volumes:
        - name: pgbouncer-config
          configMap:
            name: pgbouncer-config
```

---

## 4. Adapter Pattern สำหรับ Legacy Systems

Adapter Pattern ช่วยให้ Legacy System ที่มี interface แบบเก่าสามารถทำงานร่วมกับ Modern System ได้ โดยไม่ต้องแก้ legacy code

```typescript
// src/adapters/legacy-payment.adapter.ts
interface LegacyPaymentRequest {
  TransactionId: string;
  CardNumber: string;
  ExpiryDate: string;  // Format: MMYY
  Amount: number;      // ใน satang (100 satang = 1 baht)
  MerchantId: string;
}

interface LegacyPaymentResponse {
  ResponseCode: string;
  ResponseMessage: string;
  AuthorizationCode?: string;
  TransactionDateTime: string;  // Format: YYYYMMDDHHMMSS
}

interface ModernPaymentRequest {
  orderId: string;
  cardToken: string;
  expiryMonth: number;
  expiryYear: number;
  amountTHB: number;
  merchantId: string;
}

interface ModernPaymentResponse {
  success: boolean;
  transactionId: string;
  authorizationCode?: string;
  timestamp: Date;
  errorMessage?: string;
}

class LegacyPaymentClient {
  constructor(private readonly endpoint: string) {}

  async processPayment(request: LegacyPaymentRequest): Promise<LegacyPaymentResponse> {
    // จำลอง SOAP call ไปยัง legacy system
    const soapBody = `
      <soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
        <soap:Body>
          <ProcessPayment>
            <TransactionId>${request.TransactionId}</TransactionId>
            <Amount>${request.Amount}</Amount>
            <MerchantId>${request.MerchantId}</MerchantId>
          </ProcessPayment>
        </soap:Body>
      </soap:Envelope>
    `;
    console.log(`[LegacyClient] Sending SOAP to ${this.endpoint}`);
    // Mock response
    return {
      ResponseCode: '00',
      ResponseMessage: 'Approved',
      AuthorizationCode: 'AUTH' + Math.random().toString(36).substring(2, 8).toUpperCase(),
      TransactionDateTime: new Date().toISOString().replace(/[-:T.Z]/g, '').substring(0, 14),
    };
  }
}

export class LegacyPaymentAdapter {
  private readonly legacyClient: LegacyPaymentClient;

  constructor(legacyEndpoint: string) {
    this.legacyClient = new LegacyPaymentClient(legacyEndpoint);
  }

  async processPayment(request: ModernPaymentRequest): Promise<ModernPaymentResponse> {
    const legacyRequest = this.toLegacyFormat(request);
    try {
      const legacyResponse = await this.legacyClient.processPayment(legacyRequest);
      return this.toModernFormat(legacyResponse, request.orderId);
    } catch (error) {
      return {
        success: false,
        transactionId: request.orderId,
        timestamp: new Date(),
        errorMessage: `Legacy system error: ${(error as Error).message}`,
      };
    }
  }

  private toLegacyFormat(modern: ModernPaymentRequest): LegacyPaymentRequest {
    return {
      TransactionId: modern.orderId,
      CardNumber: modern.cardToken,
      ExpiryDate: `${String(modern.expiryMonth).padStart(2, '0')}${String(modern.expiryYear).slice(-2)}`,
      Amount: Math.round(modern.amountTHB * 100),
      MerchantId: modern.merchantId,
    };
  }

  private toModernFormat(legacy: LegacyPaymentResponse, orderId: string): ModernPaymentResponse {
    const success = legacy.ResponseCode === '00';
    const dt = legacy.TransactionDateTime;
    const timestamp = new Date(
      `${dt.substring(0, 4)}-${dt.substring(4, 6)}-${dt.substring(6, 8)}T` +
      `${dt.substring(8, 10)}:${dt.substring(10, 12)}:${dt.substring(12, 14)}Z`,
    );
    return {
      success,
      transactionId: orderId,
      authorizationCode: success ? legacy.AuthorizationCode : undefined,
      timestamp,
      errorMessage: success ? undefined : legacy.ResponseMessage,
    };
  }
}
```

```yaml
# kubernetes/adapter-sidecar.yaml
# Adapter sidecar แปลง protocol จาก gRPC เป็น REST สำหรับ legacy clients
apiVersion: apps/v1
kind: Deployment
metadata:
  name: legacy-adapter-service
  namespace: production
spec:
  replicas: 2
  selector:
    matchLabels:
      app: legacy-adapter-service
  template:
    spec:
      containers:
        # Modern gRPC service
        - name: grpc-service
          image: grpc-order-service:1.0.0
          ports:
            - containerPort: 50051
              name: grpc

        # Adapter sidecar: REST -> gRPC transcoding
        - name: grpc-rest-adapter
          image: envoyproxy/envoy:v1.28-latest
          args: ["--config-path", "/etc/envoy/envoy.yaml"]
          ports:
            - containerPort: 8080
              name: rest
          volumeMounts:
            - name: envoy-grpc-config
              mountPath: /etc/envoy
      volumes:
        - name: envoy-grpc-config
          configMap:
            name: envoy-grpc-transcoding-config
```

---

## 5. Init Containers

Init Containers คือ containers พิเศษที่รันก่อน main containers เสมอ ใช้สำหรับ setup งานที่ต้องทำก่อน main app เริ่ม เช่น database migration, certificate fetch, dependency check

### 5.1 Database Migration Init Container

```typescript
// init-containers/db-migrate/src/migrate.ts
import { Pool } from 'pg';
import * as fs from 'fs';
import * as path from 'path';
import * as crypto from 'crypto';

interface MigrationRecord {
  id: number;
  filename: string;
  executed_at: Date;
  checksum: string;
}

async function runMigrations(): Promise<void> {
  const pool = new Pool({
    connectionString: process.env.DATABASE_URL,
    max: 1,
    connectionTimeoutMillis: 30000,
  });

  const client = await pool.connect();

  try {
    await client.query(`
      CREATE TABLE IF NOT EXISTS schema_migrations (
        id SERIAL PRIMARY KEY,
        filename VARCHAR(255) NOT NULL UNIQUE,
        executed_at TIMESTAMP NOT NULL DEFAULT NOW(),
        checksum VARCHAR(64) NOT NULL
      )
    `);

    const migrationsDir = path.join(__dirname, '../migrations');
    const files = fs.readdirSync(migrationsDir)
      .filter(f => f.endsWith('.sql'))
      .sort();

    const { rows } = await client.query<MigrationRecord>(
      'SELECT filename, checksum FROM schema_migrations ORDER BY id',
    );
    const executedMigrations = new Map(rows.map(r => [r.filename, r.checksum]));

    let migrationsRun = 0;

    for (const file of files) {
      const filePath = path.join(migrationsDir, file);
      const content = fs.readFileSync(filePath, 'utf-8');
      const checksum = crypto.createHash('sha256').update(content).digest('hex');

      if (executedMigrations.has(file)) {
        if (executedMigrations.get(file) !== checksum) {
          throw new Error(
            `Migration ${file} has been modified after execution! ` +
            `Create a new migration instead.`,
          );
        }
        console.log(`[Migrate] Skipping ${file} (already executed)`);
        continue;
      }

      console.log(`[Migrate] Running ${file}...`);
      await client.query('BEGIN');
      try {
        await client.query(content);
        await client.query(
          'INSERT INTO schema_migrations (filename, checksum) VALUES ($1, $2)',
          [file, checksum],
        );
        await client.query('COMMIT');
        migrationsRun++;
        console.log(`[Migrate] Done: ${file}`);
      } catch (error) {
        await client.query('ROLLBACK');
        throw new Error(`Migration ${file} failed: ${(error as Error).message}`);
      }
    }

    console.log(`[Migrate] Complete! ${migrationsRun} migration(s) run.`);
  } finally {
    client.release();
    await pool.end();
  }
}

runMigrations().catch((error) => {
  console.error('[Migrate] Fatal:', error.message);
  process.exit(1);
});
```

```yaml
# kubernetes/init-container-migrate.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    spec:
      initContainers:
        # 1. Wait for PostgreSQL
        - name: wait-for-db
          image: postgres:16-alpine
          command:
            - sh
            - -c
            - |
              echo "Waiting for PostgreSQL..."
              until pg_isready -h $DB_HOST -p $DB_PORT -U $DB_USER; do
                echo "PostgreSQL not ready, waiting 2s..."
                sleep 2
              done
              echo "PostgreSQL is ready!"
          env:
            - name: DB_HOST
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: host
            - name: DB_PORT
              value: "5432"
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: username

        # 2. Run database migrations
        - name: db-migrate
          image: order-service-migrate:1.0.0
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: url
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"
            limits:
              cpu: "200m"
              memory: "256Mi"

        # 3. Wait for Redis
        - name: wait-for-redis
          image: redis:7-alpine
          command:
            - sh
            - -c
            - |
              until redis-cli -h $REDIS_HOST ping | grep -q PONG; do
                echo "Redis not ready, waiting 2s..."
                sleep 2
              done
              echo "Redis is ready!"
          env:
            - name: REDIS_HOST
              value: "redis.production.svc.cluster.local"

      containers:
        - name: order-service
          image: order-service:1.0.0
          ports:
            - containerPort: 3000
```

### 5.2 Certificate Download Init Container

```yaml
# kubernetes/init-container-certs.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-service
  namespace: production
spec:
  replicas: 2
  selector:
    matchLabels:
      app: secure-service
  template:
    spec:
      serviceAccountName: vault-auth-sa
      initContainers:
        - name: vault-cert-fetcher
          image: hashicorp/vault:1.15
          command:
            - sh
            - -c
            - |
              set -e
              echo "Authenticating with Vault..."

              VAULT_TOKEN=$(vault write -field=token auth/kubernetes/login \
                role=secure-service \
                jwt=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token))

              export VAULT_TOKEN

              echo "Fetching TLS certificate from Vault PKI..."
              vault write -format=json pki/issue/secure-service \
                common_name="secure-service.production.svc.cluster.local" \
                ttl=24h > /certs/cert-response.json

              cat /certs/cert-response.json | jq -r '.data.certificate' > /certs/tls.crt
              cat /certs/cert-response.json | jq -r '.data.private_key' > /certs/tls.key
              cat /certs/cert-response.json | jq -r '.data.ca_chain[]' > /certs/ca.crt

              chmod 600 /certs/tls.key
              chmod 644 /certs/tls.crt /certs/ca.crt

              echo "Certificates fetched successfully"
          env:
            - name: VAULT_ADDR
              value: "https://vault.vault.svc.cluster.local:8200"
            - name: VAULT_CACERT
              value: "/etc/vault-ca/ca.crt"
          volumeMounts:
            - name: certs
              mountPath: /certs
            - name: vault-ca
              mountPath: /etc/vault-ca
              readOnly: true

      containers:
        - name: secure-service
          image: secure-service:1.0.0
          ports:
            - containerPort: 8443
              name: https
          volumeMounts:
            - name: certs
              mountPath: /etc/certs
              readOnly: true

      volumes:
        - name: certs
          emptyDir:
            medium: Memory  # เก็บใน memory ไม่เขียน disk
        - name: vault-ca
          configMap:
            name: vault-ca-cert
```

---

## 6. Pod Lifecycle Hooks

Kubernetes มี lifecycle hooks 2 ชนิด: `postStart` (รันหลัง container เริ่ม) และ `preStop` (รันก่อน container หยุด)

### 6.1 preStop: Graceful Shutdown Handler (Express)

```typescript
// src/hooks/prestop.handler.ts
import { Request, Response, NextFunction } from 'express';
import * as http from 'http';

export class GracefulShutdownHandler {
  private isShuttingDown = false;
  private activeRequests = 0;
  private server: http.Server | null = null;

  middleware() {
    return (req: Request, res: Response, next: NextFunction): void => {
      if (this.isShuttingDown) {
        res.set('Connection', 'close');
        res.status(503).json({
          error: 'Service is shutting down',
          retryAfter: 5,
        });
        return;
      }
      this.activeRequests++;
      res.on('finish', () => { this.activeRequests--; });
      next();
    };
  }

  setServer(server: http.Server): void {
    this.server = server;
  }

  async shutdown(timeoutMs = 30000): Promise<void> {
    console.log('[Shutdown] Starting graceful shutdown...');
    this.isShuttingDown = true;

    if (this.server) {
      await new Promise<void>((resolve) => {
        this.server!.close(() => {
          console.log('[Shutdown] HTTP server closed');
          resolve();
        });
      });
    }

    const deadline = Date.now() + timeoutMs;
    while (this.activeRequests > 0 && Date.now() < deadline) {
      console.log(`[Shutdown] Waiting for ${this.activeRequests} active requests...`);
      await new Promise((resolve) => setTimeout(resolve, 500));
    }

    if (this.activeRequests > 0) {
      console.warn(`[Shutdown] Timeout! Force-closing ${this.activeRequests} requests`);
    } else {
      console.log('[Shutdown] All requests completed');
    }
  }
}
```

```yaml
# kubernetes/lifecycle-hooks.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    spec:
      terminationGracePeriodSeconds: 60
      containers:
        - name: order-service
          image: order-service:1.0.0
          ports:
            - containerPort: 3000
          lifecycle:
            # postStart: รันหลัง container เริ่ม
            postStart:
              exec:
                command:
                  - /bin/sh
                  - -c
                  - |
                    echo "Container started at $(date)" >> /var/log/lifecycle.log
                    timeout 30 sh -c 'until curl -sf http://localhost:3000/health/live; do sleep 1; done'
                    echo "App is ready" >> /var/log/lifecycle.log

            # preStop: รันก่อน SIGTERM ส่ง
            preStop:
              exec:
                command:
                  - /bin/sh
                  - -c
                  - |
                    echo "PreStop hook executing..."
                    sleep 5
                    kill -SIGTERM 1
                    sleep 25
```

```yaml
# kubernetes/poststart-http.yaml
# ใช้ httpGet สำหรับ postStart
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cache-warmer-service
spec:
  template:
    spec:
      containers:
        - name: cache-warmer-service
          image: cache-warmer-service:1.0.0
          lifecycle:
            postStart:
              httpGet:
                path: /admin/warm-cache
                port: 3000
                httpHeaders:
                  - name: X-Lifecycle-Hook
                    value: "postStart"
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 5
            failureThreshold: 10
```

---

## 7. Graceful Shutdown ใน TypeScript

```typescript
// src/server.ts — Complete Graceful Shutdown Implementation
import express, { Request, Response } from 'express';
import * as http from 'http';
import { Pool } from 'pg';
import { createClient } from 'redis';

const app = express();
const pool = new Pool({ connectionString: process.env.DATABASE_URL });
const redisClient = createClient({ url: process.env.REDIS_URL });

let isShuttingDown = false;
let activeConnections = 0;
let server: http.Server;

// Middleware: track active requests
app.use((req: Request, res: Response, next) => {
  if (isShuttingDown) {
    res.set('Connection', 'close');
    res.status(503).json({ error: 'Service Unavailable', message: 'Server is shutting down' });
    return;
  }
  activeConnections++;
  res.on('finish', () => { activeConnections--; });
  next();
});

app.get('/health/live', (req, res) => res.json({ status: 'alive' }));

app.get('/health/ready', (req, res) => {
  if (isShuttingDown) {
    res.status(503).json({ status: 'shutting_down' });
    return;
  }
  res.json({ status: 'ready' });
});

async function gracefulShutdown(signal: string): Promise<void> {
  console.log(`[Shutdown] Received ${signal}, starting graceful shutdown...`);
  if (isShuttingDown) return;
  isShuttingDown = true;

  const shutdownTimeout = parseInt(process.env.SHUTDOWN_TIMEOUT_MS ?? '25000', 10);

  try {
    // 1. หยุดรับ TCP connections ใหม่
    await new Promise<void>((resolve, reject) => {
      server.close((err) => {
        if (err) reject(err);
        else {
          console.log('[Shutdown] HTTP server stopped accepting connections');
          resolve();
        }
      });
    });

    // 2. รอ active requests จบ
    const drainStart = Date.now();
    while (activeConnections > 0) {
      if (Date.now() - drainStart > shutdownTimeout) {
        console.warn(`[Shutdown] Drain timeout! ${activeConnections} connections still active`);
        break;
      }
      console.log(`[Shutdown] Draining ${activeConnections} active connections...`);
      await new Promise((resolve) => setTimeout(resolve, 500));
    }

    // 3. ปิด Database pool
    await pool.end();
    console.log('[Shutdown] Database pool closed');

    // 4. ปิด Redis
    await redisClient.quit();
    console.log('[Shutdown] Redis connection closed');

    console.log('[Shutdown] Graceful shutdown complete');
    process.exit(0);
  } catch (error) {
    console.error('[Shutdown] Error during graceful shutdown:', error);
    process.exit(1);
  }
}

// Signal Handlers
process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));
process.on('SIGINT', () => gracefulShutdown('SIGINT'));
process.on('SIGUSR2', () => gracefulShutdown('SIGUSR2'));

process.on('uncaughtException', (error) => {
  console.error('[Fatal] Uncaught exception:', error);
  gracefulShutdown('uncaughtException');
});

process.on('unhandledRejection', (reason) => {
  console.error('[Fatal] Unhandled promise rejection:', reason);
  gracefulShutdown('unhandledRejection');
});

async function startServer(): Promise<void> {
  await redisClient.connect();
  const PORT = parseInt(process.env.PORT ?? '3000', 10);
  server = app.listen(PORT, () => {
    console.log(`[Server] Listening on port ${PORT}`);
  });
  // Keep-alive timeout สั้นกว่า load balancer timeout
  server.keepAliveTimeout = 65000;
  server.headersTimeout = 66000;
}

startServer().catch((error) => {
  console.error('[Fatal] Failed to start server:', error);
  process.exit(1);
});
```

---

## 8. Health Check Patterns

Kubernetes มี 3 ชนิด Probes: Liveness (container ยังมีชีวิต), Readiness (พร้อมรับ traffic), Startup (เริ่มต้นสำเร็จ)

```typescript
// src/health/health.controller.ts
import { Router, Request, Response } from 'express';
import { Pool } from 'pg';
import { RedisClientType } from 'redis';

interface HealthStatus {
  status: 'ok' | 'degraded' | 'error';
  timestamp: string;
  uptime: number;
  checks?: Record<string, CheckResult>;
}

interface CheckResult {
  status: 'ok' | 'error';
  latencyMs?: number;
  message?: string;
}

export class HealthController {
  private readonly router: Router;
  private readonly startTime = Date.now();
  private startupComplete = false;

  constructor(
    private readonly db: Pool,
    private readonly redis: RedisClientType,
  ) {
    this.router = Router();
    this.setupRoutes();
  }

  private setupRoutes(): void {
    this.router.get('/live', this.livenessCheck.bind(this));
    this.router.get('/ready', this.readinessCheck.bind(this));
    this.router.get('/start', this.startupCheck.bind(this));
  }

  private livenessCheck(req: Request, res: Response): void {
    res.status(200).json({
      status: 'ok',
      timestamp: new Date().toISOString(),
      uptime: Date.now() - this.startTime,
    } satisfies HealthStatus);
  }

  private async readinessCheck(req: Request, res: Response): Promise<void> {
    const checks: Record<string, CheckResult> = {};
    let overallOk = true;

    // ตรวจ Database
    const dbStart = Date.now();
    try {
      const client = await this.db.connect();
      await client.query('SELECT 1');
      client.release();
      checks.database = { status: 'ok', latencyMs: Date.now() - dbStart };
    } catch (error) {
      checks.database = {
        status: 'error',
        latencyMs: Date.now() - dbStart,
        message: (error as Error).message,
      };
      overallOk = false;
    }

    // ตรวจ Redis
    const redisStart = Date.now();
    try {
      await this.redis.ping();
      checks.redis = { status: 'ok', latencyMs: Date.now() - redisStart };
    } catch (error) {
      checks.redis = {
        status: 'error',
        latencyMs: Date.now() - redisStart,
        message: (error as Error).message,
      };
      overallOk = false;
    }

    // ตรวจ memory usage
    const memUsage = process.memoryUsage();
    const heapUsedPercent = (memUsage.heapUsed / memUsage.heapTotal) * 100;
    if (heapUsedPercent > 90) {
      checks.memory = { status: 'error', message: `Heap usage critical: ${heapUsedPercent.toFixed(1)}%` };
      overallOk = false;
    } else {
      checks.memory = { status: 'ok', message: `Heap: ${heapUsedPercent.toFixed(1)}%` };
    }

    res.status(overallOk ? 200 : 503).json({
      status: overallOk ? 'ok' : 'error',
      timestamp: new Date().toISOString(),
      uptime: Date.now() - this.startTime,
      checks,
    } satisfies HealthStatus);
  }

  private startupCheck(req: Request, res: Response): void {
    if (!this.startupComplete) {
      res.status(503).json({
        status: 'starting',
        timestamp: new Date().toISOString(),
        message: 'Application is still initializing...',
      });
      return;
    }
    res.status(200).json({
      status: 'ok',
      timestamp: new Date().toISOString(),
      message: 'Application started successfully',
    });
  }

  markStartupComplete(): void { this.startupComplete = true; }
  getRouter(): Router { return this.router; }
}
```

```yaml
# kubernetes/health-probes.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    spec:
      containers:
        - name: order-service
          image: order-service:1.0.0
          ports:
            - containerPort: 3000

          # Startup Probe — ให้เวลา app เริ่มต้นได้นานสุด 3 นาที
          startupProbe:
            httpGet:
              path: /health/start
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 36
            successThreshold: 1
            timeoutSeconds: 3

          # Liveness Probe — ตรวจว่า app ยังทำงานอยู่
          livenessProbe:
            httpGet:
              path: /health/live
              port: 3000
            initialDelaySeconds: 0
            periodSeconds: 10
            failureThreshold: 3
            successThreshold: 1
            timeoutSeconds: 3

          # Readiness Probe — ตรวจว่าพร้อมรับ traffic
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 3000
            initialDelaySeconds: 0
            periodSeconds: 5
            failureThreshold: 3
            successThreshold: 1
            timeoutSeconds: 3
```

---

## 9. Resource Limits and Requests

การกำหนด Resource Limits และ Requests ที่เหมาะสมเป็นสิ่งสำคัญสำหรับ cluster stability และ cost efficiency

### 9.1 VPA (Vertical Pod Autoscaler) Recommendations

```yaml
# kubernetes/vpa.yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: order-service-vpa
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  updatePolicy:
    # "Off" = แค่ recommend ไม่ auto-update
    # "Initial" = set ตอน pod สร้างใหม่
    # "Auto" = auto-update (อาจ restart)
    updateMode: "Off"
  resourcePolicy:
    containerPolicies:
      - containerName: order-service
        minAllowed:
          cpu: "50m"
          memory: "64Mi"
        maxAllowed:
          cpu: "2000m"
          memory: "2Gi"
        controlledResources: ["cpu", "memory"]
        controlledValues: RequestsAndLimits
```

### 9.2 TypeScript CPU/Memory Profiling

```typescript
// src/monitoring/profiler.ts
import v8 from 'v8';
import os from 'os';

interface ResourceMetrics {
  timestamp: string;
  process: {
    pid: number;
    uptime: number;
    cpuUsage: NodeJS.CpuUsage;
    memoryUsage: NodeJS.MemoryUsage;
  };
  heap: {
    totalHeapSize: number;
    usedHeapSize: number;
    heapSizeLimit: number;
    heapUsedPercent: number;
  };
  system: {
    totalMemory: number;
    freeMemory: number;
    cpuCount: number;
    loadAvg: number[];
  };
}

export class ResourceProfiler {
  private previousCpuUsage: NodeJS.CpuUsage | null = null;
  private previousTimestamp: number = Date.now();

  getMetrics(): ResourceMetrics {
    const memUsage = process.memoryUsage();
    const heapStats = v8.getHeapStatistics();
    const cpuUsage = process.cpuUsage(this.previousCpuUsage ?? undefined);

    this.previousCpuUsage = process.cpuUsage();
    this.previousTimestamp = Date.now();

    return {
      timestamp: new Date().toISOString(),
      process: {
        pid: process.pid,
        uptime: process.uptime(),
        cpuUsage,
        memoryUsage: memUsage,
      },
      heap: {
        totalHeapSize: heapStats.total_heap_size,
        usedHeapSize: heapStats.used_heap_size,
        heapSizeLimit: heapStats.heap_size_limit,
        heapUsedPercent: (heapStats.used_heap_size / heapStats.heap_size_limit) * 100,
      },
      system: {
        totalMemory: os.totalmem(),
        freeMemory: os.freemem(),
        cpuCount: os.cpus().length,
        loadAvg: os.loadavg(),
      },
    };
  }

  getResourceRecommendations(): { cpu: string; memory: string } {
    const metrics = this.getMetrics();
    const memMiB = Math.ceil(metrics.process.memoryUsage.rss / (1024 * 1024));
    const cpuMillicores = Math.ceil(
      (metrics.process.cpuUsage.user + metrics.process.cpuUsage.system) / 1000 / 100,
    );
    return {
      cpu: `${cpuMillicores * 2}m`,
      memory: `${memMiB * 2}Mi`,
    };
  }
}

// ส่ง metrics ทุก 30 วินาที
const profiler = new ResourceProfiler();
setInterval(() => {
  const metrics = profiler.getMetrics();
  process.stdout.write(JSON.stringify({ level: 'info', message: 'resource_metrics', ...metrics }) + '\n');
}, 30000);
```

```yaml
# kubernetes/resource-limits.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    spec:
      containers:
        - name: order-service
          image: order-service:1.0.0
          resources:
            # requests: ใช้สำหรับ scheduling
            requests:
              cpu: "100m"
              memory: "128Mi"
            # limits: maximum ที่ใช้ได้
            # CPU เกิน = throttled; Memory เกิน = OOMKilled
            limits:
              cpu: "500m"
              memory: "512Mi"
          env:
            - name: NODE_OPTIONS
              value: "--max-old-space-size=384"  # ต่ำกว่า memory limit

---
# LimitRange: default resource ทั้ง namespace
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
    - type: Container
      default:
        cpu: "200m"
        memory: "256Mi"
      defaultRequest:
        cpu: "50m"
        memory: "64Mi"
      max:
        cpu: "2000m"
        memory: "2Gi"
      min:
        cpu: "10m"
        memory: "32Mi"
```

---

## 10. Container Image Best Practices

### 10.1 Multi-Stage Dockerfile (Builder → Runtime)

```dockerfile
# Dockerfile.optimized
# ===== Stage 1: Dependencies =====
FROM node:20-alpine AS deps
WORKDIR /app
# คัดลอก package files ก่อน เพื่อใช้ Docker layer cache
COPY package.json package-lock.json ./
RUN npm ci

# ===== Stage 2: Builder =====
FROM node:20-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

# ===== Stage 3: Production Dependencies =====
FROM node:20-alpine AS prod-deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev

# ===== Stage 4: Runtime (Distroless) =====
# distroless ไม่มี shell, package manager — attack surface น้อย
FROM gcr.io/distroless/nodejs20-debian12 AS runtime

USER nonroot:nonroot
WORKDIR /app

COPY --from=prod-deps --chown=nonroot:nonroot /app/node_modules ./node_modules
COPY --from=builder --chown=nonroot:nonroot /app/dist ./dist
COPY --chown=nonroot:nonroot package.json ./

LABEL org.opencontainers.image.title="Order Service"
LABEL org.opencontainers.image.version="1.0.0"
LABEL org.opencontainers.image.source="https://github.com/company/order-service"

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD ["/nodejs/bin/node", "-e", \
    "require('http').get('http://localhost:3000/health/live', r => process.exit(r.statusCode === 200 ? 0 : 1))"]

# distroless ไม่มี sh ต้องใช้ exec form
CMD ["dist/server.js"]
```

### 10.2 Security Scanning ด้วย Trivy

```yaml
# .github/workflows/security-scan.yaml
name: Container Security Scan

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  build-and-scan:
    runs-on: ubuntu-latest
    permissions:
      security-events: write
      contents: read

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Build Docker image
        run: |
          docker build -t order-service:${{ github.sha }} \
            --file Dockerfile.optimized .

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: "order-service:${{ github.sha }}"
          format: "sarif"
          output: "trivy-results.sarif"
          severity: "CRITICAL,HIGH"
          exit-code: "1"

      - name: Upload Trivy results to GitHub Security tab
        uses: github/codeql-action/upload-sarif@v2
        if: always()
        with:
          sarif_file: "trivy-results.sarif"

      - name: Run Hadolint Dockerfile linter
        uses: hadolint/hadolint-action@v3.1.0
        with:
          dockerfile: Dockerfile.optimized
          failure-threshold: warning
```

```typescript
// scripts/check-image-size.ts
import { execSync } from 'child_process';

function getImageSize(imageTag: string): number {
  const output = execSync(
    `docker image inspect ${imageTag} --format '{{.Size}}'`,
  ).toString().trim();
  return parseInt(output, 10);
}

function formatBytes(bytes: number): string {
  return `${(bytes / (1024 * 1024)).toFixed(1)} MB`;
}

const SIZE_THRESHOLDS = {
  api: 150 * 1024 * 1024,
  worker: 100 * 1024 * 1024,
  init: 50 * 1024 * 1024,
};

const imageTag = process.argv[2];
const serviceType = (process.argv[3] as keyof typeof SIZE_THRESHOLDS) ?? 'api';

if (!imageTag) {
  console.error('Usage: ts-node check-image-size.ts <image-tag> [service-type]');
  process.exit(1);
}

const sizeBytes = getImageSize(imageTag);
const threshold = SIZE_THRESHOLDS[serviceType];

console.log(`Image: ${imageTag}`);
console.log(`Size: ${formatBytes(sizeBytes)}`);
console.log(`Threshold: ${formatBytes(threshold)}`);

if (sizeBytes > threshold) {
  console.error(`Image size exceeds threshold for '${serviceType}' service!`);
  process.exit(1);
} else {
  console.log(`Image size is within threshold`);
  process.exit(0);
}
```

---

## 11. Complete Kubernetes Deployment

นำทุก pattern มารวมกัน — complete deployment ที่ใช้ทุก pattern ในบทนี้

```yaml
# kubernetes/complete-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: production
  labels:
    app: order-service
    version: "1.0.0"
    team: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: order-service
        version: "1.0.0"
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: order-service-sa
      terminationGracePeriodSeconds: 60

      securityContext:
        runAsNonRoot: true
        runAsUser: 65532
        runAsGroup: 65532
        fsGroup: 65532
        seccompProfile:
          type: RuntimeDefault

      initContainers:
        - name: wait-for-db
          image: postgres:16-alpine
          command: ["sh", "-c", "until pg_isready -h $(DB_HOST) -p 5432; do sleep 1; done"]
          env:
            - name: DB_HOST
              valueFrom:
                secretKeyRef:
                  name: order-db-credentials
                  key: host

        - name: db-migrate
          image: order-service-migrate:1.0.0
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: order-db-credentials
                  key: url

      containers:
        # Main Application
        - name: order-service
          image: order-service:1.0.0
          imagePullPolicy: Always

          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]

          ports:
            - containerPort: 3000
              name: http

          env:
            - name: PORT
              value: "3000"
            - name: NODE_ENV
              value: "production"
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: order-db-credentials
                  key: url
            - name: REDIS_URL
              valueFrom:
                secretKeyRef:
                  name: redis-credentials
                  key: url
            - name: NODE_OPTIONS
              value: "--max-old-space-size=384"

          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"

          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 5"]

          startupProbe:
            httpGet:
              path: /health/start
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 36

          livenessProbe:
            httpGet:
              path: /health/live
              port: 3000
            periodSeconds: 10
            failureThreshold: 3
            timeoutSeconds: 3

          readinessProbe:
            httpGet:
              path: /health/ready
              port: 3000
            periodSeconds: 5
            failureThreshold: 3
            timeoutSeconds: 3

          volumeMounts:
            - name: tmp
              mountPath: /tmp
            - name: app-logs
              mountPath: /var/log/app

        # Sidecar: Metrics Exporter
        - name: metrics-exporter
          image: order-service-metrics:1.0.0
          ports:
            - containerPort: 9090
              name: metrics
          resources:
            requests:
              cpu: "10m"
              memory: "32Mi"
            limits:
              cpu: "50m"
              memory: "64Mi"

        # Sidecar: Fluentd Log Shipper
        - name: fluentd
          image: fluent/fluentd-kubernetes-daemonset:v1.16-debian-elasticsearch8-1
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"
            limits:
              cpu: "200m"
              memory: "256Mi"
          volumeMounts:
            - name: app-logs
              mountPath: /var/log/app
              readOnly: true
            - name: fluentd-config
              mountPath: /fluentd/etc

      volumes:
        - name: tmp
          emptyDir: {}
        - name: app-logs
          emptyDir: {}
        - name: fluentd-config
          configMap:
            name: fluentd-config

      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values: [order-service]
                topologyKey: kubernetes.io/hostname

      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: order-service
```

```yaml
# kubernetes/pod-disruption-budget.yaml
# PDB ป้องกันไม่ให้ Kubernetes ลด replicas มากเกินไประหว่าง maintenance
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: order-service-pdb
  namespace: production
spec:
  minAvailable: 2  # ต้องมี pod อย่างน้อย 2 ตัวเสมอ
  selector:
    matchLabels:
      app: order-service
```

---

## สรุป Part 64

| Pattern | จุดประสงค์ | เครื่องมือ |
|---------|------------|-----------|
| Twelve-Factor App | Cloud-native best practices | Node.js, dotenv, structured logging |
| Sidecar | เพิ่ม capability โดยไม่แก้ main app | Envoy, Fluentd, Prometheus |
| Ambassador | Abstract external service complexity | TypeScript Proxy class |
| Adapter | รองรับ Legacy Systems | TypeScript Adapter class |
| Init Container | Setup ก่อน main app เริ่ม | Migration scripts, Vault cert fetchers |
| Lifecycle Hooks | preStop/postStart | Shell scripts, HTTP GET |
| Graceful Shutdown | จบ requests ก่อน terminate | SIGTERM/SIGINT handlers |
| Health Checks | Live/Ready/Startup probes | Express HTTP endpoints |
| Resource Management | Stability และ cost | VPA, LimitRange, profiler |
| Distroless Images | Security, ขนาดเล็ก | Multi-stage Dockerfile |
| PDB | HA during maintenance | PodDisruptionBudget |

Cloud-Native Patterns เหล่านี้ช่วยให้ Microservices ทำงานได้อย่างน่าเชื่อถือบน Kubernetes โดยใช้ทรัพยากรอย่างมีประสิทธิภาพ และง่ายต่อการดูแลรักษาใน production environment

# Part 64: Cloud-Native Patterns

## บทนำ

ใน Part นี้เราจะเรียนรู้ Cloud-Native Patterns ที่สำคัญสำหรับการพัฒนา Microservices บน Kubernetes ตั้งแต่หลักการ Twelve-Factor App ไปจนถึง Sidecar, Ambassador, Adapter Patterns, Init Containers, Pod Lifecycle Hooks, Graceful Shutdown, Health Checks, Resource Management และ Container Image Best Practices

---

## 1. Twelve-Factor App Principles กับ Node.js

Twelve-Factor App คือชุดหลักการสำหรับสร้าง Software-as-a-Service (SaaS) ที่ทำงานได้ดีบน Cloud โดยมี 12 ข้อดังนี้

### Factor I: Codebase — เก็บโค้ดใน Version Control

```typescript
// src/config/app.config.ts
// Factor I: One codebase tracked in revision control, many deploys
export const appConfig = {
  name: process.env.APP_NAME || 'microservice',
  version: process.env.APP_VERSION || '1.0.0',
  environment: process.env.NODE_ENV || 'development',
};
```

### Factor II: Dependencies — ประกาศ Dependencies อย่างชัดเจน

```json
// package.json
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
    "dotenv": "^16.3.1"
  },
  "devDependencies": {
    "typescript": "^5.3.2",
    "@types/express": "^4.17.21",
    "@types/node": "^20.10.0"
  }
}
```

### Factor III: Config — เก็บ Config ใน Environment Variables

```typescript
// src/config/environment.ts
// Factor III: Store config in the environment
import * as dotenv from 'dotenv';

dotenv.config();

interface AppEnvironment {
  // Server
  PORT: number;
  HOST: string;
  NODE_ENV: string;

  // Database
  DATABASE_URL: string;
  DATABASE_POOL_MIN: number;
  DATABASE_POOL_MAX: number;

  // Redis
  REDIS_URL: string;
  REDIS_TTL: number;

  // External Services
  USER_SERVICE_URL: string;
  PAYMENT_SERVICE_URL: string;

  // Secrets
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

### Factor IV: Backing Services — ปฏิบัติกับ Backing Services เป็น Attached Resources

```typescript
// src/infrastructure/database.ts
// Factor IV: Treat backing services as attached resources
import { Pool } from 'pg';
import { createClient } from 'redis';
import { env } from '../config/environment';

// Database Connection — ใช้ URL สามารถเปลี่ยน host ได้โดยไม่แก้โค้ด
export const db = new Pool({
  connectionString: env.DATABASE_URL,
  min: env.DATABASE_POOL_MIN,
  max: env.DATABASE_POOL_MAX,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});

// Redis Connection — เช่นกัน
export const redis = createClient({
  url: env.REDIS_URL,
});

redis.on('error', (err) => {
  console.error('Redis Client Error:', err);
});

export async function connectBackingServices(): Promise<void> {
  await redis.connect();
  // Test DB connection
  const client = await db.connect();
  client.release();
  console.log('All backing services connected');
}
```

### Factor V: Build, Release, Run — แยก Build, Release, Run Stages

```dockerfile
# Dockerfile — แสดง Build stage ที่แยกจาก Run stage
# Factor V: Strictly separate build and run stages

# ===== Build Stage =====
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production=false
COPY . .
RUN npm run build

# ===== Release Stage (ด้วย version tagging) =====
FROM node:20-alpine AS release
WORKDIR /app
# Copy only production dependencies
COPY package*.json ./
RUN npm ci --only=production
COPY --from=builder /app/dist ./dist

# ===== Run Stage =====
FROM gcr.io/distroless/nodejs20-debian12 AS runtime
WORKDIR /app
COPY --from=release /app/node_modules ./node_modules
COPY --from=release /app/dist ./dist
EXPOSE 3000
CMD ["dist/server.js"]
```

### Factor VI: Processes — รัน App เป็น Stateless Processes

```typescript
// src/services/session.service.ts
// Factor VI: Execute the app as one or more stateless processes
// State ทั้งหมดเก็บใน Redis ไม่เก็บใน memory ของ process
import { redis } from '../infrastructure/database';

export class SessionService {
  private readonly SESSION_PREFIX = 'session:';
  private readonly TTL = 3600; // 1 hour

  async createSession(userId: string, data: Record<string, unknown>): Promise<string> {
    const sessionId = crypto.randomUUID();
    const key = `${this.SESSION_PREFIX}${sessionId}`;

    // เก็บ state ใน Redis (shared backing service) ไม่ใช่ in-memory
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
    const key = `${this.SESSION_PREFIX}${sessionId}`;
    await redis.del(key);
  }
}
```

### Factors VII-XII: Port Binding, Concurrency, Disposability, Dev/Prod Parity, Logs, Admin Processes

```typescript
// src/server.ts
// Factor VII: Export services via port binding
// Factor IX: Maximize robustness with fast startup and graceful shutdown
// Factor XI: Treat logs as event streams
import express from 'express';
import { env } from './config/environment';

const app = express();

// Factor XI: Logs as event streams — เขียนไปที่ stdout เสมอ
// ไม่เปิด/ปิด file, ไม่จัดการ log rotation
function log(level: string, message: string, meta?: Record<string, unknown>): void {
  const entry = {
    timestamp: new Date().toISOString(),
    level,
    message,
    service: env.appConfig?.name ?? 'app',
    ...meta,
  };
  // Factor XI: Write to stdout, let the platform collect it
  process.stdout.write(JSON.stringify(entry) + '\n');
}

// Factor VII: Port Binding
const server = app.listen(env.PORT, env.HOST, () => {
  log('info', `Server started`, { port: env.PORT, host: env.HOST });
});

// Factor IX: Fast startup
server.on('listening', () => {
  log('info', 'Ready to accept connections');
});

export { app, server, log };
```

---

## 2. Sidecar Pattern

Sidecar Pattern คือการแนบ container เสริม (sidecar) เข้ากับ main application container ในสกัด Pod เดียวกัน โดย sidecar ให้บริการเสริมเช่น logging, monitoring, proxying

### 2.1 Envoy Proxy Sidecar

```yaml
# kubernetes/sidecar-envoy.yaml
# Envoy ทำหน้าที่เป็น proxy สำหรับ network traffic
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
        # Prometheus scrape annotations สำหรับ Envoy metrics
        prometheus.io/scrape: "true"
        prometheus.io/port: "9901"
        prometheus.io/path: "/stats/prometheus"
    spec:
      containers:
        # ===== Main Application Container =====
        - name: order-service
          image: order-service:1.0.0
          ports:
            - containerPort: 3000
              name: http
          env:
            - name: PORT
              value: "3000"
            # ชี้ไปที่ localhost เพราะ Envoy รันใน Pod เดียวกัน
            - name: UPSTREAM_HOST
              value: "127.0.0.1"
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"

        # ===== Envoy Proxy Sidecar =====
        - name: envoy-proxy
          image: envoyproxy/envoy:v1.28-latest
          args:
            - "--config-path /etc/envoy/envoy.yaml"
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
# Fluentd sidecar อ่าน logs จาก shared volume แล้วส่งไปยัง Elasticsearch
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
    metadata:
      labels:
        app: payment-service
    spec:
      containers:
        # ===== Main Container — เขียน logs ลงไฟล์ใน shared volume =====
        - name: payment-service
          image: payment-service:1.0.0
          env:
            - name: LOG_FILE
              value: "/var/log/app/payment.log"
          volumeMounts:
            - name: app-logs
              mountPath: /var/log/app
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"

        # ===== Fluentd Sidecar =====
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
        kubernetes_node "#{ENV['MY_NODE_NAME']}"
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
// Custom metrics sidecar ที่อ่าน metrics จาก main app และ expose ให้ Prometheus
import express from 'express';
import { register, Counter, Gauge, Histogram, collectDefaultMetrics } from 'prom-client';

const app = express();

// เก็บ default Node.js metrics (CPU, memory, etc.)
collectDefaultMetrics({ prefix: 'nodejs_' });

// Custom metrics
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

// Metrics endpoint สำหรับ Prometheus scrape
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});

app.get('/health', (req, res) => {
  res.json({ status: 'ok' });
});

const METRICS_PORT = parseInt(process.env.METRICS_PORT || '9090', 10);
app.listen(METRICS_PORT, () => {
  console.log(`Metrics server listening on port ${METRICS_PORT}`);
});
```

---

## 3. Ambassador Pattern

Ambassador Pattern คือ Proxy Container ที่ทำงานแทน main application ในการสื่อสารกับ external services โดย main app ไม่ต้องรู้รายละเอียดของ external service

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
  CLOSED = 'CLOSED',     // ปกติ — ส่ง request ได้
  OPEN = 'OPEN',         // เปิด — reject request ทันที
  HALF_OPEN = 'HALF_OPEN', // ทดสอบ — ส่ง request จำนวนจำกัด
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

  // Circuit Breaker State
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

  // ตรวจสอบ Circuit Breaker state
  private checkCircuitBreaker(): void {
    if (this.cbState === CircuitState.OPEN) {
      const now = Date.now();
      // ถ้าผ่าน timeout แล้ว ให้เปลี่ยนเป็น HALF_OPEN
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

      if (body) {
        req.write(body);
      }
      req.end();
    });
  }

  // Retry with exponential backoff
  async request(
    path: string,
    method = 'GET',
    headers: Record<string, string> = {},
    body?: unknown,
  ): Promise<{ statusCode: number; body: unknown }> {
    // ตรวจสอบ Circuit Breaker ก่อน
    this.checkCircuitBreaker();

    const serializedBody = body ? JSON.stringify(body) : undefined;
    let lastError: Error | null = null;
    let delay = this.retryConfig.initialDelayMs;

    for (let attempt = 1; attempt <= this.retryConfig.maxAttempts; attempt++) {
      try {
        const response = await this.makeRequest(path, method, headers, serializedBody);

        // 5xx errors ถือเป็น failure สำหรับ retry
        if (response.statusCode >= 500) {
          throw new Error(`Server error: ${response.statusCode}`);
        }

        this.recordSuccess();
        return {
          statusCode: response.statusCode,
          body: JSON.parse(response.body),
        };
      } catch (error) {
        lastError = error as Error;
        console.warn(`[Ambassador] Attempt ${attempt} failed: ${lastError.message}`);

        if (attempt < this.retryConfig.maxAttempts) {
          await new Promise((resolve) => setTimeout(resolve, delay));
          delay = Math.min(
            delay * this.retryConfig.backoffMultiplier,
            this.retryConfig.maxDelayMs,
          );
        }
      }
    }

    this.recordFailure();
    throw lastError ?? new Error('Request failed after all retries');
  }

  getCircuitState(): CircuitState {
    return this.cbState;
  }
}

// ตัวอย่างการใช้งาน Ambassador
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
# Ambassador สำหรับ Database connection pooling + credential rotation
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

Adapter Pattern ช่วยให้ Legacy System ที่มี interface แบบเก่าสามารถทำงานร่วมกับ Modern System ได้

```typescript
// src/adapters/legacy-payment.adapter.ts
// ระบบ Legacy ใช้ XML-based SOAP API
// Modern System ต้องการ JSON REST API

interface LegacyPaymentRequest {
  // รูปแบบ XML field จาก legacy system
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

// Modern interface ที่ microservice ต้องการ
interface ModernPaymentRequest {
  orderId: string;
  cardToken: string;  // tokenized card number
  expiryMonth: number;
  expiryYear: number;
  amountTHB: number;  // ใน บาท
  merchantId: string;
}

interface ModernPaymentResponse {
  success: boolean;
  transactionId: string;
  authorizationCode?: string;
  timestamp: Date;
  errorMessage?: string;
}

// Legacy payment client (เลียนแบบ SOAP client)
class LegacyPaymentClient {
  constructor(private readonly endpoint: string) {}

  async processPayment(request: LegacyPaymentRequest): Promise<LegacyPaymentResponse> {
    // จำลอง SOAP call ไปยัง legacy system
    const soapBody = `
      <soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
        <soap:Body>
          <ProcessPayment>
            <TransactionId>${request.TransactionId}</TransactionId>
            <CardNumber>${request.CardNumber}</CardNumber>
            <ExpiryDate>${request.ExpiryDate}</ExpiryDate>
            <Amount>${request.Amount}</Amount>
            <MerchantId>${request.MerchantId}</MerchantId>
          </ProcessPayment>
        </soap:Body>
      </soap:Envelope>
    `;

    console.log(`[LegacyClient] Sending SOAP request to ${this.endpoint}`);
    // ในการใช้งานจริงจะส่ง HTTP POST ไปที่ SOAP endpoint
    // จำลอง response
    return {
      ResponseCode: '00',
      ResponseMessage: 'Approved',
      AuthorizationCode: 'AUTH' + Math.random().toString(36).substring(2, 8).toUpperCase(),
      TransactionDateTime: new Date().toISOString().replace(/[-:T.Z]/g, '').substring(0, 14),
    };
  }
}

// Adapter: แปลง Modern interface ให้ทำงานกับ Legacy client
export class LegacyPaymentAdapter {
  private readonly legacyClient: LegacyPaymentClient;

  constructor(legacyEndpoint: string) {
    this.legacyClient = new LegacyPaymentClient(legacyEndpoint);
  }

  async processPayment(request: ModernPaymentRequest): Promise<ModernPaymentResponse> {
    // แปลง Modern request เป็น Legacy format
    const legacyRequest = this.tolegacyFormat(request);

    try {
      const legacyResponse = await this.legacyClient.processPayment(legacyRequest);
      // แปลง Legacy response เป็น Modern format
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

  private tolegacyFormat(modern: ModernPaymentRequest): LegacyPaymentRequest {
    return {
      TransactionId: modern.orderId,
      CardNumber: modern.cardToken,
      // แปลง month/year เป็น MMYY format
      ExpiryDate: `${String(modern.expiryMonth).padStart(2, '0')}${String(modern.expiryYear).slice(-2)}`,
      // แปลง THB เป็น satang (คูณ 100)
      Amount: Math.round(modern.amountTHB * 100),
      MerchantId: modern.merchantId,
    };
  }

  private toModernFormat(legacy: LegacyPaymentResponse, orderId: string): ModernPaymentResponse {
    const success = legacy.ResponseCode === '00';
    // แปลง YYYYMMDDHHMMSS เป็น Date object
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
# Adapter sidecar แปลง protocol จาก gRPC เป็น REST สำหรับ legacy client
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

        # Adapter sidecar: แปลง REST -> gRPC
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

Init Containers คือ containers พิเศษที่รันก่อน main containers เสมอ ใช้สำหรับ setup งานที่ต้องทำก่อน main app เริ่ม

### 5.1 Database Migration Init Container

```typescript
// init-containers/db-migrate/src/migrate.ts
import { Pool } from 'pg';
import * as fs from 'fs';
import * as path from 'path';

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
    // สร้าง migrations table ถ้ายังไม่มี
    await client.query(`
      CREATE TABLE IF NOT EXISTS schema_migrations (
        id SERIAL PRIMARY KEY,
        filename VARCHAR(255) NOT NULL UNIQUE,
        executed_at TIMESTAMP NOT NULL DEFAULT NOW(),
        checksum VARCHAR(64) NOT NULL
      )
    `);

    // หา migration files ทั้งหมด
    const migrationsDir = path.join(__dirname, '../migrations');
    const files = fs.readdirSync(migrationsDir)
      .filter(f => f.endsWith('.sql'))
      .sort(); // เรียงตาม filename (001_xxx, 002_xxx, ...)

    // ดึง migrations ที่รันแล้ว
    const { rows } = await client.query<MigrationRecord>(
      'SELECT filename, checksum FROM schema_migrations ORDER BY id',
    );
    const executedMigrations = new Map(rows.map(r => [r.filename, r.checksum]));

    let migrationsRun = 0;

    for (const file of files) {
      const filePath = path.join(migrationsDir, file);
      const content = fs.readFileSync(filePath, 'utf-8');
      const checksum = require('crypto')
        .createHash('sha256')
        .update(content)
        .digest('hex');

      if (executedMigrations.has(file)) {
        // ตรวจสอบ checksum — ห้ามแก้ migration ที่รันแล้ว
        if (executedMigrations.get(file) !== checksum) {
          throw new Error(
            `Migration ${file} has been modified after execution! ` +
            `This is not allowed. Create a new migration instead.`,
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
        console.log(`[Migrate] ✓ ${file} completed`);
      } catch (error) {
        await client.query('ROLLBACK');
        throw new Error(`Migration ${file} failed: ${(error as Error).message}`);
      }
    }

    console.log(`[Migrate] Done! ${migrationsRun} migration(s) executed.`);
  } finally {
    client.release();
    await pool.end();
  }
}

runMigrations().catch((error) => {
  console.error('[Migrate] Fatal error:', error.message);
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
      # Init containers รันตามลำดับก่อน main containers
      initContainers:
        # 1. รอให้ Database พร้อม
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

        # 2. รัน Database Migrations
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

        # 3. รอให้ Redis พร้อม
        - name: wait-for-redis
          image: redis:7-alpine
          command:
            - sh
            - -c
            - |
              echo "Waiting for Redis..."
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
# Init container สำหรับดาวน์โหลด TLS certificates จาก Vault
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

              # Login ด้วย Kubernetes service account
              VAULT_TOKEN=$(vault write -field=token auth/kubernetes/login \
                role=secure-service \
                jwt=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token))

              export VAULT_TOKEN

              echo "Fetching TLS certificate from Vault PKI..."
              vault write -format=json pki/issue/secure-service \
                common_name="secure-service.production.svc.cluster.local" \
                ttl=24h > /certs/cert-response.json

              # Extract certificate components
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

### 6.1 preStop: Graceful Shutdown Hook

```typescript
// src/hooks/prestop.handler.ts
// Express middleware สำหรับจัดการ preStop hook
import { Request, Response, NextFunction } from 'express';
import * as http from 'http';

export class GracefulShutdownHandler {
  private isShuttingDown = false;
  private activeRequests = 0;
  private server: http.Server | null = null;

  // Middleware ที่ตรวจสอบว่า server กำลัง shutdown หรือไม่
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
      res.on('finish', () => {
        this.activeRequests--;
      });
      next();
    };
  }

  setServer(server: http.Server): void {
    this.server = server;
  }

  // เรียกตอน preStop hook
  async shutdown(timeoutMs = 30000): Promise<void> {
    console.log('[Shutdown] Starting graceful shutdown...');
    this.isShuttingDown = true;

    // หยุดรับ connections ใหม่
    if (this.server) {
      await new Promise<void>((resolve) => {
        this.server!.close(() => {
          console.log('[Shutdown] HTTP server closed');
          resolve();
        });
      });
    }

    // รอให้ active requests จบ
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
      # terminationGracePeriodSeconds ต้องมากกว่า preStop timeout
      terminationGracePeriodSeconds: 60
      containers:
        - name: order-service
          image: order-service:1.0.0
          ports:
            - containerPort: 3000
          lifecycle:
            # postStart: รันหลัง container เริ่ม (ไม่ guarantee ว่าจะก่อน ENTRYPOINT เสร็จ)
            postStart:
              exec:
                command:
                  - /bin/sh
                  - -c
                  - |
                    echo "Container started at $(date)" >> /var/log/lifecycle.log
                    # รอให้ app พร้อมก่อน
                    timeout 30 sh -c 'until curl -sf http://localhost:3000/health/live; do sleep 1; done'
                    echo "App is ready" >> /var/log/lifecycle.log

            # preStop: รันก่อน SIGTERM ส่ง — ใช้ delay เพื่อให้ load balancer remove endpoint ก่อน
            preStop:
              exec:
                command:
                  - /bin/sh
                  - -c
                  - |
                    echo "PreStop hook executing..."
                    # รอ 5 วินาที ให้ load balancer หยุดส่ง traffic มา
                    sleep 5
                    # ส่ง signal ให้ app เริ่ม graceful shutdown
                    kill -SIGTERM 1
                    # รอให้ app shutdown เอง
                    sleep 25
```

### 6.2 postStart Initialization Hook ด้วย HTTP

```yaml
# kubernetes/poststart-http.yaml
# ใช้ httpGet สำหรับ postStart — เรียก initialization endpoint
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
            # รอนานขึ้นเพราะต้อง warm cache ก่อน
            initialDelaySeconds: 30
            periodSeconds: 5
            failureThreshold: 10
```

---

## 7. Graceful Shutdown ใน TypeScript

การ shutdown อย่าง graceful คือการให้ service จบ active requests ที่มีอยู่ก่อน แล้วค่อย shutdown

```typescript
// src/server.ts — Complete Graceful Shutdown Implementation
import express, { Request, Response } from 'express';
import * as http from 'http';
import { Pool } from 'pg';
import { createClient } from 'redis';

const app = express();
const pool = new Pool({ connectionString: process.env.DATABASE_URL });
const redisClient = createClient({ url: process.env.REDIS_URL });

// Track state
let isShuttingDown = false;
let activeConnections = 0;
let server: http.Server;

// Middleware: track active requests
app.use((req: Request, res: Response, next) => {
  if (isShuttingDown) {
    res.set('Connection', 'close');
    res.status(503).json({ 
      error: 'Service Unavailable',
      message: 'Server is shutting down',
    });
    return;
  }
  activeConnections++;
  res.on('finish', () => { activeConnections--; });
  next();
});

app.get('/health/live', (req, res) => {
  res.status(200).json({ status: 'alive' });
});

app.get('/health/ready', (req, res) => {
  if (isShuttingDown) {
    res.status(503).json({ status: 'shutting_down' });
    return;
  }
  res.status(200).json({ status: 'ready' });
});

// ===== Graceful Shutdown Function =====
async function gracefulShutdown(signal: string): Promise<void> {
  console.log(`[Shutdown] Received ${signal}, starting graceful shutdown...`);

  if (isShuttingDown) {
    console.log('[Shutdown] Already shutting down, ignoring signal');
    return;
  }

  isShuttingDown = true;
  const shutdownTimeout = parseInt(process.env.SHUTDOWN_TIMEOUT_MS ?? '25000', 10);

  try {
    // 1. หยุดรับ TCP connections ใหม่
    await new Promise<void>((resolve, reject) => {
      server.close((err) => {
        if (err) {
          console.error('[Shutdown] Error closing HTTP server:', err);
          reject(err);
        } else {
          console.log('[Shutdown] HTTP server stopped accepting new connections');
          resolve();
        }
      });
    });

    // 2. รอ active requests จบ หรือ timeout
    const drainStart = Date.now();
    while (activeConnections > 0) {
      if (Date.now() - drainStart > shutdownTimeout) {
        console.warn(`[Shutdown] Drain timeout! ${activeConnections} connections still active`);
        break;
      }
      console.log(`[Shutdown] Draining ${activeConnections} active connections...`);
      await new Promise((resolve) => setTimeout(resolve, 500));
    }

    // 3. ปิด Database connection pool
    console.log('[Shutdown] Closing database pool...');
    await pool.end();
    console.log('[Shutdown] Database pool closed');

    // 4. ปิด Redis connection
    console.log('[Shutdown] Closing Redis connection...');
    await redisClient.quit();
    console.log('[Shutdown] Redis connection closed');

    console.log('[Shutdown] Graceful shutdown complete');
    process.exit(0);
  } catch (error) {
    console.error('[Shutdown] Error during graceful shutdown:', error);
    process.exit(1);
  }
}

// ===== Signal Handlers =====
// SIGTERM — Kubernetes ส่งเมื่อต้องการหยุด pod
process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));

// SIGINT — Ctrl+C ระหว่าง development
process.on('SIGINT', () => gracefulShutdown('SIGINT'));

// SIGUSR2 — nodemon restart
process.on('SIGUSR2', () => gracefulShutdown('SIGUSR2'));

// Uncaught exceptions — ไม่ควร crash โดยไม่ cleanup
process.on('uncaughtException', (error) => {
  console.error('[Fatal] Uncaught exception:', error);
  gracefulShutdown('uncaughtException');
});

process.on('unhandledRejection', (reason) => {
  console.error('[Fatal] Unhandled promise rejection:', reason);
  gracefulShutdown('unhandledRejection');
});

// ===== Start Server =====
async function startServer(): Promise<void> {
  await redisClient.connect();

  const PORT = parseInt(process.env.PORT ?? '3000', 10);
  server = app.listen(PORT, () => {
    console.log(`[Server] Listening on port ${PORT}`);
  });

  // Keep-alive connections ควรมี timeout เพื่อให้ shutdown ได้เร็ว
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

Kubernetes รองรับ 3 ชนิด Probes:
- **Liveness Probe**: ตรวจว่า container ยังมีชีวิตอยู่ (ถ้า fail จะ restart)
- **Readiness Probe**: ตรวจว่า container พร้อมรับ traffic (ถ้า fail จะเอาออกจาก load balancer)
- **Startup Probe**: ตรวจว่า container เริ่มต้นสำเร็จ (ใช้สำหรับ app ที่ start ช้า)

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
  private isReady = false;
  private startupComplete = false;

  constructor(
    private readonly db: Pool,
    private readonly redis: RedisClientType,
  ) {
    this.router = Router();
    this.setupRoutes();
  }

  private setupRoutes(): void {
    // Liveness: ตรวจว่า process ยังมีชีวิตอยู่
    // ควร fail เฉพาะตอนที่ app ค้าง (deadlock, OOM, ฯลฯ)
    this.router.get('/live', this.livenessCheck.bind(this));

    // Readiness: ตรวจว่าพร้อมรับ traffic
    // ควร fail เมื่อ dependencies ไม่พร้อม
    this.router.get('/ready', this.readinessCheck.bind(this));

    // Startup: ตรวจว่าเริ่มต้นสำเร็จ
    // ใช้แทน readiness สำหรับ slow-starting apps
    this.router.get('/start', this.startupCheck.bind(this));
  }

  // Liveness: เบาที่สุด — แค่ตอบว่ายังมีชีวิตอยู่
  private livenessCheck(req: Request, res: Response): void {
    const uptime = Date.now() - this.startTime;
    const status: HealthStatus = {
      status: 'ok',
      timestamp: new Date().toISOString(),
      uptime,
    };
    res.status(200).json(status);
  }

  // Readiness: ตรวจ dependencies ทั้งหมด
  private async readinessCheck(req: Request, res: Response): Promise<void> {
    const checks: Record<string, CheckResult> = {};
    let overallOk = true;

    // ตรวจ Database
    const dbStart = Date.now();
    try {
      const client = await this.db.connect();
      await client.query('SELECT 1');
      client.release();
      checks.database = {
        status: 'ok',
        latencyMs: Date.now() - dbStart,
      };
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
      checks.redis = {
        status: 'ok',
        latencyMs: Date.now() - redisStart,
      };
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
      checks.memory = {
        status: 'error',
        message: `Heap usage critical: ${heapUsedPercent.toFixed(1)}%`,
      };
      overallOk = false;
    } else {
      checks.memory = {
        status: 'ok',
        message: `Heap: ${heapUsedPercent.toFixed(1)}%`,
      };
    }

    const statusCode = overallOk ? 200 : 503;
    res.status(statusCode).json({
      status: overallOk ? 'ok' : 'error',
      timestamp: new Date().toISOString(),
      uptime: Date.now() - this.startTime,
      checks,
    } satisfies HealthStatus);
  }

  // Startup: ตรวจว่า initialization เสร็จแล้ว
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

  markReady(): void {
    this.isReady = true;
  }

  markStartupComplete(): void {
    this.startupComplete = true;
  }

  getRouter(): Router {
    return this.router;
  }
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

          # Startup Probe — ตรวจว่า app เริ่มต้นสำเร็จ
          # ให้เวลา app เริ่มได้นานสุด: failureThreshold * periodSeconds = 3 นาที
          startupProbe:
            httpGet:
              path: /health/start
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 36     # 36 * 5 = 180 วินาที = 3 นาที
            successThreshold: 1
            timeoutSeconds: 3

          # Liveness Probe — ตรวจว่า app ยังทำงานอยู่
          # เริ่มตรวจหลัง startup probe ผ่านแล้ว
          livenessProbe:
            httpGet:
              path: /health/live
              port: 3000
            initialDelaySeconds: 0
            periodSeconds: 10
            failureThreshold: 3      # restart หลัง fail 3 ครั้ง
            successThreshold: 1
            timeoutSeconds: 3

          # Readiness Probe — ตรวจว่าพร้อมรับ traffic
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 3000
            initialDelaySeconds: 0
            periodSeconds: 5
            failureThreshold: 3      # เอาออกจาก load balancer หลัง fail 3 ครั้ง
            successThreshold: 1
            timeoutSeconds: 3
```

---

## 9. Resource Limits and Requests

การกำหนด Resource Limits และ Requests ที่เหมาะสมเป็นสิ่งสำคัญสำหรับ cluster stability และ cost efficiency

### 9.1 VPA (Vertical Pod Autoscaler) Recommendations

```yaml
# kubernetes/vpa.yaml
# VPA จะ recommend/auto-set resource requests ตาม actual usage
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
    # "Auto" = auto-update (อาจทำให้ restart)
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
// ติดตาม resource usage ของ Node.js process
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
    mallocedMemory: number;
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

    const now = Date.now();
    const elapsed = now - this.previousTimestamp;

    this.previousCpuUsage = process.cpuUsage();
    this.previousTimestamp = now;

    // คำนวณ CPU percentage (microseconds / milliseconds * 100)
    const cpuPercent = ((cpuUsage.user + cpuUsage.system) / 1000 / elapsed) * 100;

    return {
      timestamp: new Date().toISOString(),
      process: {
        pid: process.pid,
        uptime: process.uptime(),
        cpuUsage: {
          user: cpuUsage.user,
          system: cpuUsage.system,
        },
        memoryUsage: memUsage,
      },
      heap: {
        totalHeapSize: heapStats.total_heap_size,
        usedHeapSize: heapStats.used_heap_size,
        heapSizeLimit: heapStats.heap_size_limit,
        heapUsedPercent: (heapStats.used_heap_size / heapStats.heap_size_limit) * 100,
        mallocedMemory: heapStats.malloced_memory,
      },
      system: {
        totalMemory: os.totalmem(),
        freeMemory: os.freemem(),
        cpuCount: os.cpus().length,
        loadAvg: os.loadavg(),
      },
    };
  }

  // แนะนำ resource limits จาก observed usage
  getResourceRecommendations(): { cpu: string; memory: string } {
    const metrics = this.getMetrics();
    const memMiB = Math.ceil(metrics.process.memoryUsage.rss / (1024 * 1024));
    const cpuMillicores = Math.ceil(
      (metrics.process.cpuUsage.user + metrics.process.cpuUsage.system) / 1000 / 100,
    );

    // แนะนำ 2x actual usage เป็น limit
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
  process.stdout.write(JSON.stringify({
    level: 'info',
    message: 'resource_metrics',
    ...metrics,
  }) + '\n');
}, 30000);
```

```yaml
# kubernetes/resource-limits.yaml
# Best practices สำหรับ resource configuration
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
            # requests: ใช้สำหรับ scheduling — Kubernetes จอง resource นี้
            requests:
              cpu: "100m"       # 0.1 CPU core
              memory: "128Mi"   # 128 MiB RAM

            # limits: maximum ที่ container ใช้ได้
            # ถ้า CPU เกิน: throttled (ไม่ kill)
            # ถ้า memory เกิน: OOMKilled
            limits:
              cpu: "500m"       # 0.5 CPU core
              memory: "512Mi"   # 512 MiB RAM

          # Environment variables สำหรับ Node.js memory
          env:
            # กำหนด Node.js heap size ให้น้อยกว่า container memory limit
            - name: NODE_OPTIONS
              value: "--max-old-space-size=384"  # 384 MB < 512 MB limit

---
# LimitRange: กำหนด default และ max สำหรับทั้ง namespace
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
    - type: Container
      default:         # ใช้ถ้าไม่ได้ระบุ limits
        cpu: "200m"
        memory: "256Mi"
      defaultRequest:  # ใช้ถ้าไม่ได้ระบุ requests
        cpu: "50m"
        memory: "64Mi"
      max:             # ห้ามเกินนี้
        cpu: "2000m"
        memory: "2Gi"
      min:             # ต้องอย่างน้อยนี้
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

# ติดตั้ง dependencies ทั้งหมด (รวม devDependencies สำหรับ build)
RUN npm ci

# ===== Stage 2: Builder =====
FROM node:20-alpine AS builder
WORKDIR /app

# รับ dependencies จาก deps stage
COPY --from=deps /app/node_modules ./node_modules
COPY . .

# Build TypeScript
RUN npm run build

# ===== Stage 3: Production Dependencies =====
FROM node:20-alpine AS prod-deps
WORKDIR /app
COPY package.json package-lock.json ./
# ติดตั้งเฉพาะ production dependencies
RUN npm ci --omit=dev

# ===== Stage 4: Runtime (Distroless) =====
# ใช้ distroless เพราะไม่มี shell, package manager, ทำให้ attack surface น้อยลง
FROM gcr.io/distroless/nodejs20-debian12 AS runtime

# กำหนด non-root user
USER nonroot:nonroot

WORKDIR /app

# คัดลอกเฉพาะสิ่งที่จำเป็น
COPY --from=prod-deps --chown=nonroot:nonroot /app/node_modules ./node_modules
COPY --from=builder --chown=nonroot:nonroot /app/dist ./dist
COPY --chown=nonroot:nonroot package.json ./

# Metadata labels
LABEL org.opencontainers.image.title="Order Service"
LABEL org.opencontainers.image.version="1.0.0"
LABEL org.opencontainers.image.source="https://github.com/company/order-service"

# Port documentation (ไม่ได้เปิด port จริง)
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD ["/nodejs/bin/node", "-e", "require('http').get('http://localhost:3000/health/live', r => process.exit(r.statusCode === 200 ? 0 : 1))"]

# Run as non-root, distroless ไม่มี sh จึงใช้ exec form เสมอ
CMD ["dist/server.js"]
```

### 10.2 Security Scanning ด้วย Trivy

```yaml
# .github/workflows/security-scan.yaml
# CI pipeline สำหรับ scan container image
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
          exit-code: "1"  # fail CI ถ้าพบ CRITICAL/HIGH

      - name: Upload Trivy scan results to GitHub Security tab
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
// ตรวจสอบ image size ไม่ให้เกิน threshold
import { execSync } from 'child_process';

interface ImageInfo {
  repository: string;
  tag: string;
  size: string;
  sizeBytes: number;
}

function getImageSize(imageTag: string): number {
  const output = execSync(
    `docker image inspect ${imageTag} --format '{{.Size}}'`,
  ).toString().trim();
  return parseInt(output, 10);
}

function formatBytes(bytes: number): string {
  const mb = bytes / (1024 * 1024);
  return `${mb.toFixed(1)} MB`;
}

// Threshold สำหรับแต่ละ service type
const SIZE_THRESHOLDS = {
  api: 150 * 1024 * 1024,     // 150 MB
  worker: 100 * 1024 * 1024,  // 100 MB
  init: 50 * 1024 * 1024,     // 50 MB
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
  console.error(`❌ Image size exceeds threshold for '${serviceType}' service!`);
  process.exit(1);
} else {
  console.log(`✓ Image size is within threshold`);
  process.exit(0);
}
```

---

## 11. ตัวอย่าง Complete Deployment

นำทุกอย่างมารวมกัน — complete deployment ที่ใช้ทุก pattern ในบทนี้

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
  # Rolling update strategy
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0  # ไม่ลด capacity ระหว่าง deploy
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

      # Security Context ระดับ Pod
      securityContext:
        runAsNonRoot: true
        runAsUser: 65532  # nonroot user ใน distroless
        runAsGroup: 65532
        fsGroup: 65532
        seccompProfile:
          type: RuntimeDefault

      # Init Containers
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

          # Security Context ระดับ Container
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]

          ports:
            - containerPort: 3000
              name: http
              protocol: TCP

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

          # Lifecycle Hooks
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 5"]

          # Probes
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

      # Anti-affinity: กระจาย pods ไปคนละ node
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

      # Topology Spread: กระจายระหว่าง zones
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: order-service
```

---

## สรุป Part 64

| Pattern | จุดประสงค์ | เครื่องมือ |
|---------|------------|-----------|
| Twelve-Factor App | Cloud-native best practices | Node.js, dotenv |
| Sidecar | เพิ่ม capability โดยไม่แก้ main app | Envoy, Fluentd |
| Ambassador | Abstract external service complexity | Custom TypeScript Proxy |
| Adapter | รองรับ Legacy Systems | TypeScript Adapter class |
| Init Container | Setup ก่อน main app เริ่ม | Migration scripts, cert fetchers |
| Lifecycle Hooks | preStop/postStart actions | Shell scripts, HTTP |
| Graceful Shutdown | จบ requests ก่อน terminate | SIGTERM/SIGINT handlers |
| Health Checks | Live/Ready/Startup probes | Express endpoints |
| Resource Management | Stability และ cost efficiency | VPA, LimitRange |
| Distroless Images | Security, small size | Multi-stage Dockerfile |

Cloud-Native Patterns เหล่านี้ช่วยให้ Microservices ทำงานได้อย่างน่าเชื่อถือบน Kubernetes โดยใช้ทรัพยากรอย่างมีประสิทธิภาพ และง่ายต่อการดูแลรักษาใน production environment

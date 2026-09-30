# Part 64: Cloud-Native Patterns

## บทนำ

Cloud-Native Patterns คือชุดของ Design Patterns ที่ออกแบบมาเพื่อ maximize
ประสิทธิภาพของ Applications ที่รันบน Cloud Infrastructure โดยเฉพาะ Kubernetes
บทนี้จะครอบคลุม Patterns ที่สำคัญตั้งแต่ Twelve-Factor App ไปจนถึง Container best practices

---

## 1. Twelve-Factor App Principles

### Factor I: Codebase

```
One codebase tracked in version control, many deploys
- ใช้ Git Repository เดียวต่อ service
- ต่างกันแค่ environment variables
- ไม่ share code ระหว่าง services ผ่าน copy-paste
```

### Factor III: Config (การจัดการ Configuration)

```typescript
// src/config/twelve-factor.config.ts
// Config ต้องมาจาก Environment Variables เท่านั้น
// ไม่ hardcode values ใน code

export function validateRequiredEnvVars(): void {
  const required = [
    'DATABASE_URL',
    'REDIS_URL',
    'JWT_SECRET',
    'PORT',
  ];

  const missing = required.filter(key => !process.env[key]);

  if (missing.length > 0) {
    throw new Error(`Missing required environment variables: ${missing.join(', ')}`);
  }
}

// เรียกใน main.ts ก่อน bootstrap
validateRequiredEnvVars();
```

### Factor XI: Logs (Treat logs as event streams)

```typescript
// src/logger/structured-logger.ts
import { createLogger, format, transports } from 'winston';

export const logger = createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: format.combine(
    format.timestamp(),
    format.errors({ stack: true }),
    format.json()
  ),
  // ส่ง logs ไปยัง stdout เท่านั้น (ตาม 12-factor)
  transports: [
    new transports.Console({
      format: format.combine(
        format.colorize(),
        format.simple()
      ),
    }),
  ],
  // ไม่ไปยุ่งกับ log aggregation ใน application code
  // ให้ platform จัดการ (Fluentd, Logstash ฯลฯ)
});
```

---

## 2. Sidecar Pattern

### Architecture

```
Pod:
├── Main Container (application)
└── Sidecar Container (supporting function)
    ├── Logging sidecar (Fluentd)
    ├── Proxy sidecar (Envoy/Istio)
    ├── Config sync sidecar
    └── Metrics sidecar
```

```yaml
# sidecar-pattern.yaml
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
    spec:
      volumes:
        - name: shared-logs
          emptyDir: {}
        - name: config-sync
          emptyDir: {}

      containers:
        # Main Application Container
        - name: order-service
          image: registry.company.com/order-service:1.0.0
          ports:
            - containerPort: 3000
          volumeMounts:
            - name: shared-logs
              mountPath: /var/log/app
          env:
            - name: LOG_DIR
              value: /var/log/app

        # Sidecar 1: Log Shipper (Fluentd)
        - name: log-shipper
          image: fluent/fluentd-kubernetes-daemonset:v1.16
          volumeMounts:
            - name: shared-logs
              mountPath: /var/log/app
              readOnly: true
          resources:
            requests:
              memory: "64Mi"
              cpu: "50m"
            limits:
              memory: "128Mi"
              cpu: "100m"

        # Sidecar 2: Config Sync (Git sync)
        - name: config-sync
          image: k8s.gcr.io/git-sync/git-sync:v3.6.1
          args:
            - --repo=https://github.com/company/configs
            - --branch=main
            - --root=/config
            - --dest=current
            - --period=60s
          volumeMounts:
            - name: config-sync
              mountPath: /config
          resources:
            requests:
              memory: "32Mi"
              cpu: "50m"
```

### Custom Sidecar ด้วย TypeScript

```typescript
// sidecar/metrics-proxy/src/main.ts
import express from 'express';
import httpProxy from 'http-proxy-middleware';
import { register, Counter, Histogram } from 'prom-client';

const app = express();
const MAIN_APP_PORT = process.env.MAIN_APP_PORT || '3000';
const SIDECAR_PORT = process.env.SIDECAR_PORT || '3001';

// Metrics
const requestCounter = new Counter({
  name: 'http_requests_total',
  help: 'Total HTTP requests',
  labelNames: ['method', 'path', 'status_code'],
});

const requestDuration = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration',
  labelNames: ['method', 'path'],
  buckets: [0.01, 0.05, 0.1, 0.5, 1, 5],
});

// Metrics endpoint
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});

// Intercept requests and collect metrics
app.use((req, res, next) => {
  const startTime = Date.now();
  const method = req.method;
  const path = req.path;

  res.on('finish', () => {
    const duration = (Date.now() - startTime) / 1000;
    requestCounter.inc({ method, path, status_code: res.statusCode.toString() });
    requestDuration.observe({ method, path }, duration);
  });

  next();
});

// Proxy to main app
app.use('/', httpProxy.createProxyMiddleware({
  target: `http://localhost:${MAIN_APP_PORT}`,
  changeOrigin: true,
}));

app.listen(SIDECAR_PORT, () => {
  console.log(`Metrics sidecar listening on port ${SIDECAR_PORT}`);
});
```

---

## 3. Ambassador Pattern

```yaml
# ambassador-pattern.yaml
# Ambassador: proxy ระหว่าง application กับ external service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
spec:
  template:
    spec:
      containers:
        # Main Application
        - name: payment-service
          image: registry.company.com/payment-service:1.0.0
          env:
            # ชี้ไปที่ ambassador แทน external service โดยตรง
            - name: STRIPE_API_ENDPOINT
              value: "http://localhost:8080"

        # Ambassador Container: handles retry, circuit breaking, TLS
        - name: stripe-ambassador
          image: envoyproxy/envoy:v1.28-latest
          ports:
            - containerPort: 8080
          volumeMounts:
            - name: envoy-config
              mountPath: /etc/envoy
          command: ["/usr/local/bin/envoy", "-c", "/etc/envoy/envoy.yaml"]
          resources:
            requests:
              memory: "64Mi"
              cpu: "100m"
            limits:
              memory: "128Mi"
              cpu: "200m"

      volumes:
        - name: envoy-config
          configMap:
            name: stripe-envoy-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: stripe-envoy-config
data:
  envoy.yaml: |
    static_resources:
      listeners:
        - name: local_listener
          address:
            socket_address:
              address: 0.0.0.0
              port_value: 8080
          filter_chains:
            - filters:
                - name: envoy.filters.network.http_connection_manager
                  typed_config:
                    "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
                    codec_type: AUTO
                    stat_prefix: ingress_http
                    route_config:
                      name: local_route
                      virtual_hosts:
                        - name: stripe
                          domains: ["*"]
                          routes:
                            - match:
                                prefix: "/"
                              route:
                                cluster: stripe_api
                                retry_policy:
                                  retry_on: "5xx,reset,connect-failure"
                                  num_retries: 3
                                  per_try_timeout: 10s
                    http_filters:
                      - name: envoy.filters.http.router
      clusters:
        - name: stripe_api
          connect_timeout: 5s
          type: LOGICAL_DNS
          dns_lookup_family: V4_ONLY
          load_assignment:
            cluster_name: stripe_api
            endpoints:
              - lb_endpoints:
                  - endpoint:
                      address:
                        socket_address:
                          address: api.stripe.com
                          port_value: 443
          transport_socket:
            name: envoy.transport_sockets.tls
```

---

## 4. Init Containers

```yaml
# init-containers.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: production
spec:
  template:
    spec:
      initContainers:
        # 1. รอ Database พร้อม
        - name: wait-for-database
          image: postgres:15-alpine
          command: ['sh', '-c']
          args:
            - |
              until pg_isready -h $DB_HOST -p $DB_PORT -U $DB_USER; do
                echo "$(date) - waiting for database..."
                sleep 2
              done
              echo "Database is ready!"
          env:
            - name: DB_HOST
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: DATABASE_HOST
            - name: DB_PORT
              value: "5432"
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: DATABASE_USERNAME

        # 2. รัน Database Migrations
        - name: run-migrations
          image: registry.company.com/order-service:1.0.0
          command: ['node', 'dist/migrate.js']
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: DATABASE_URL
          resources:
            limits:
              memory: "256Mi"
              cpu: "500m"

        # 3. Download configuration files
        - name: download-config
          image: curlimages/curl:latest
          command: ['sh', '-c']
          args:
            - |
              curl -o /config/app-config.json \
                "http://config-server:8888/order-service/production"
              echo "Config downloaded successfully"
          volumeMounts:
            - name: config-volume
              mountPath: /config

      containers:
        - name: order-service
          image: registry.company.com/order-service:1.0.0
          volumeMounts:
            - name: config-volume
              mountPath: /app/config
              readOnly: true

      volumes:
        - name: config-volume
          emptyDir: {}
```

---

## 5. Graceful Shutdown

```typescript
// src/lifecycle/graceful-shutdown.ts
import { Injectable, Logger, OnApplicationShutdown } from '@nestjs/common';
import { HttpServer } from '@nestjs/common';

@Injectable()
export class GracefulShutdownService implements OnApplicationShutdown {
  private readonly logger = new Logger(GracefulShutdownService.name);
  private isShuttingDown = false;
  private readonly SHUTDOWN_TIMEOUT_MS = 30000;

  async onApplicationShutdown(signal?: string): Promise<void> {
    this.logger.log(`Received shutdown signal: ${signal}`);
    this.isShuttingDown = true;

    const shutdownTimeout = setTimeout(() => {
      this.logger.error('Graceful shutdown timeout, forcing exit');
      process.exit(1);
    }, this.SHUTDOWN_TIMEOUT_MS);

    try {
      await this.performGracefulShutdown();
      clearTimeout(shutdownTimeout);
      this.logger.log('Graceful shutdown completed');
    } catch (error) {
      this.logger.error('Error during graceful shutdown:', error);
      clearTimeout(shutdownTimeout);
      process.exit(1);
    }
  }

  private async performGracefulShutdown(): Promise<void> {
    // 1. หยุดรับ traffic ใหม่ (liveness probe จะ return 503)
    this.logger.log('Step 1: Stopping new connections');

    // 2. รอให้ in-flight requests เสร็จ (max 20 seconds)
    await this.waitForInflightRequests(20000);

    // 3. Close database connections
    this.logger.log('Step 3: Closing database connections');
    await this.closeDatabaseConnections();

    // 4. Close message queue connections
    this.logger.log('Step 4: Closing message queue connections');
    await this.closeMessageQueueConnections();

    // 5. Flush logs
    this.logger.log('Step 5: Flushing logs');
    await this.flushLogs();

    this.logger.log('All cleanup tasks completed');
  }

  private async waitForInflightRequests(maxWaitMs: number): Promise<void> {
    const startTime = Date.now();
    
    while (Date.now() - startTime < maxWaitMs) {
      const inflightCount = await this.getInflightRequestCount();
      
      if (inflightCount === 0) {
        this.logger.log('All in-flight requests completed');
        return;
      }
      
      this.logger.log(`Waiting for ${inflightCount} in-flight requests...`);
      await new Promise(resolve => setTimeout(resolve, 500));
    }
    
    this.logger.warn(`Timeout waiting for in-flight requests`);
  }

  private async getInflightRequestCount(): Promise<number> {
    return 0; // Implement with actual tracking
  }

  private async closeDatabaseConnections(): Promise<void> {
    // Close DB pool
    this.logger.log('Database connections closed');
  }

  private async closeMessageQueueConnections(): Promise<void> {
    // Close MQ connections
    this.logger.log('Message queue connections closed');
  }

  private async flushLogs(): Promise<void> {
    await new Promise(resolve => setTimeout(resolve, 100));
  }

  isShutdownInProgress(): boolean {
    return this.isShuttingDown;
  }
}
```

```yaml
# pod-lifecycle-hooks.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  template:
    spec:
      terminationGracePeriodSeconds: 60  # ให้เวลา 60 วินาที
      containers:
        - name: order-service
          image: registry.company.com/order-service:1.0.0

          lifecycle:
            # preStop: รันก่อนที่ container จะถูกหยุด
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 5"]
                # sleep 5 วินาทีเพื่อให้ load balancer หยุดส่ง traffic
            # postStart: รันหลังจาก container เริ่มทำงาน
            postStart:
              exec:
                command: ["/bin/sh", "-c", "echo 'Container started' >> /var/log/lifecycle.log"]
```

---

## 6. Health Check Patterns

```typescript
// src/health/health.controller.ts
import { Controller, Get } from '@nestjs/common';
import { HealthCheck, HealthCheckService, TypeOrmHealthIndicator, MemoryHealthIndicator, DiskHealthIndicator } from '@nestjs/terminus';
import { InjectRepository } from '@nestjs/typeorm';
import { DataSource } from 'typeorm';

@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private db: TypeOrmHealthIndicator,
    private memory: MemoryHealthIndicator,
    private disk: DiskHealthIndicator,
    private dataSource: DataSource
  ) {}

  // Liveness Probe: ตรวจสอบว่า application ทำงานอยู่ (ถ้า fail → restart)
  @Get('liveness')
  @HealthCheck()
  async liveness() {
    return this.health.check([
      // ตรวจสอบแค่ว่า process ยังทำงานได้
      () => this.memory.checkHeap('memory_heap', 250 * 1024 * 1024),
      // ตรวจสอบว่าไม่อยู่ใน graceful shutdown
      async () => {
        const isShuttingDown = false; // Check shutdown flag
        if (isShuttingDown) {
          return {
            shutdown: {
              status: 'down',
              message: 'Service is shutting down',
            },
          };
        }
        return { shutdown: { status: 'up' } };
      },
    ]);
  }

  // Readiness Probe: ตรวจสอบว่าพร้อมรับ traffic (ถ้า fail → หยุดส่ง traffic)
  @Get('readiness')
  @HealthCheck()
  async readiness() {
    return this.health.check([
      // ตรวจสอบ database connection
      () => this.db.pingCheck('database'),
      // ตรวจสอบ disk space
      () => this.disk.checkStorage('disk', { path: '/', threshold: 0.9 }),
    ]);
  }

  // Startup Probe: ตรวจสอบว่า application เริ่มต้นเสร็จแล้ว
  @Get('startup')
  @HealthCheck()
  async startup() {
    return this.health.check([
      async () => {
        // ตรวจสอบว่า migrations รันเสร็จแล้ว
        try {
          await this.dataSource.query('SELECT 1');
          return { migrations: { status: 'up' } };
        } catch {
          return { migrations: { status: 'down', message: 'Database not ready' } };
        }
      },
    ]);
  }
}
```

```yaml
# health-check-probes.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  template:
    spec:
      containers:
        - name: order-service
          image: registry.company.com/order-service:1.0.0

          # Startup Probe: ให้เวลา 60 วินาทีสำหรับ startup
          startupProbe:
            httpGet:
              path: /health/startup
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 12  # 12 * 5s = 60 seconds
            successThreshold: 1
            timeoutSeconds: 3

          # Liveness Probe: ตรวจสอบทุก 30 วินาที
          livenessProbe:
            httpGet:
              path: /health/liveness
              port: 3000
            initialDelaySeconds: 0
            periodSeconds: 30
            failureThreshold: 3  # Restart หลัง 3 ครั้ง fail
            successThreshold: 1
            timeoutSeconds: 5

          # Readiness Probe: ตรวจสอบทุก 10 วินาที
          readinessProbe:
            httpGet:
              path: /health/readiness
              port: 3000
            initialDelaySeconds: 0
            periodSeconds: 10
            failureThreshold: 3
            successThreshold: 2  # ต้อง pass 2 ครั้งติดกัน
            timeoutSeconds: 3
```

---

## 7. Resource Limits and Requests

```yaml
# resource-limits.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  template:
    spec:
      containers:
        - name: order-service
          image: registry.company.com/order-service:1.0.0

          resources:
            requests:
              # ทรัพยากรขั้นต่ำที่ต้องการ (สำหรับ scheduling)
              memory: "256Mi"
              cpu: "250m"     # 0.25 CPU cores
            limits:
              # ทรัพยากรสูงสุด (เกินกว่านี้จะถูก throttle/OOM killed)
              memory: "512Mi"
              cpu: "500m"     # 0.5 CPU cores

---
# Vertical Pod Autoscaler (VPA)
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: order-service-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
      - containerName: order-service
        minAllowed:
          cpu: "100m"
          memory: "128Mi"
        maxAllowed:
          cpu: "2"
          memory: "2Gi"
        controlledResources: ["cpu", "memory"]

---
# LimitRange สำหรับ namespace
apiVersion: v1
kind: LimitRange
metadata:
  name: order-service-limits
  namespace: production
spec:
  limits:
    - type: Container
      default:
        memory: "256Mi"
        cpu: "250m"
      defaultRequest:
        memory: "128Mi"
        cpu: "100m"
      max:
        memory: "2Gi"
        cpu: "2"
      min:
        memory: "64Mi"
        cpu: "50m"
    - type: Pod
      max:
        memory: "4Gi"
        cpu: "4"
```

---

## 8. Multi-Stage Docker Build (Distroless)

```dockerfile
# Dockerfile - Multi-Stage Build with Distroless
# Stage 1: Dependencies
FROM node:20-alpine AS deps
WORKDIR /app

# ติดตั้งเฉพาะ production dependencies
COPY package*.json ./
RUN npm ci --only=production && \
    npm cache clean --force

# Stage 2: Build
FROM node:20-alpine AS builder
WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build && \
    npm prune --production

# Stage 3: Security scan (optional)
FROM aquasec/trivy:latest AS scanner
COPY --from=builder /app /scan-target
RUN trivy filesystem --exit-code 1 --severity HIGH,CRITICAL /scan-target || true

# Stage 4: Final - Distroless image
FROM gcr.io/distroless/nodejs20-debian12 AS production

# ไม่มี shell, package manager, หรือ utilities ใดๆ
# ลด attack surface อย่างมาก

WORKDIR /app

# Copy built artifacts
COPY --from=builder --chown=nonroot:nonroot /app/dist ./dist
COPY --from=builder --chown=nonroot:nonroot /app/node_modules ./node_modules
COPY --from=builder --chown=nonroot:nonroot /app/package.json ./

# Non-root user (UID 65532 ใน distroless)
USER nonroot

EXPOSE 3000

# ใช้ exec form เพื่อ process ได้รับ signals โดยตรง
CMD ["dist/main.js"]
```

```dockerfile
# Dockerfile.development - สำหรับ development (hot-reload)
FROM node:20-alpine

WORKDIR /app

# ติดตั้ง development tools
RUN apk add --no-cache \
    bash \
    curl \
    git

COPY package*.json ./
RUN npm install

COPY . .

EXPOSE 3000 9229

# Debug port 9229
CMD ["node", "--inspect=0.0.0.0:9229", "-r", "ts-node/register", "src/main.ts"]
```

---

## 9. Adapter Pattern ใน Kubernetes

```yaml
# adapter-pattern.yaml
# Adapter: แปลง format ของ legacy app ให้เข้ากับ monitoring system
apiVersion: apps/v1
kind: Deployment
metadata:
  name: legacy-app
spec:
  template:
    spec:
      containers:
        # Legacy Application (ส่ง metrics ใน custom format)
        - name: legacy-app
          image: registry.company.com/legacy-app:1.0.0
          ports:
            - containerPort: 8080
              name: http
            - containerPort: 9090
              name: custom-metrics

        # Adapter Sidecar: แปลง custom metrics เป็น Prometheus format
        - name: metrics-adapter
          image: registry.company.com/metrics-adapter:1.0.0
          ports:
            - containerPort: 9091
              name: prometheus
          env:
            - name: LEGACY_METRICS_URL
              value: "http://localhost:9090/metrics"
            - name: LISTEN_PORT
              value: "9091"
          resources:
            requests:
              memory: "32Mi"
              cpu: "50m"
            limits:
              memory: "64Mi"
              cpu: "100m"
```

```typescript
// adapter/metrics-adapter/src/main.ts
// Adapter: แปลง legacy metrics format เป็น Prometheus format
import express from 'express';
import axios from 'axios';
import { register, Gauge } from 'prom-client';

const app = express();
const LEGACY_URL = process.env.LEGACY_METRICS_URL!;

// สร้าง Prometheus metrics จาก legacy format
const legacyMetrics = new Map<string, Gauge>();

async function fetchAndConvertMetrics(): Promise<void> {
  const response = await axios.get<Record<string, number>>(LEGACY_URL);
  const legacyData = response.data;

  // แปลง legacy format เป็น Prometheus format
  for (const [key, value] of Object.entries(legacyData)) {
    const metricName = `legacy_${key.replace(/[^a-zA-Z0-9_]/g, '_')}`;

    if (!legacyMetrics.has(metricName)) {
      legacyMetrics.set(metricName, new Gauge({
        name: metricName,
        help: `Legacy metric: ${key}`,
      }));
    }

    legacyMetrics.get(metricName)!.set(value);
  }
}

app.get('/metrics', async (req, res) => {
  await fetchAndConvertMetrics();
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});

app.listen(process.env.LISTEN_PORT || 9091);
```

---

## 10. PodDisruptionBudget

```yaml
# pod-disruption-budget.yaml
# กำหนดว่า pod จะ unavailable ได้กี่ตัวในช่วง voluntary disruption
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: order-service-pdb
  namespace: production
spec:
  selector:
    matchLabels:
      app: order-service
  minAvailable: 2  # ต้องมี pods ทำงานอยู่อย่างน้อย 2 ตัว
  # หรือ
  # maxUnavailable: 1  # unavailable ได้สูงสุด 1 ตัว
```

---

## 11. Horizontal Pod Autoscaler

```yaml
# hpa.yaml
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
          averageUtilization: 70  # Scale up เมื่อ CPU > 70%
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
    # Custom metric: scale based on queue depth
    - type: External
      external:
        metric:
          name: rabbitmq_queue_messages
          selector:
            matchLabels:
              queue: orders.processing
        target:
          type: AverageValue
          averageValue: "100"  # Scale ถ้า queue > 100 messages per pod
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Percent
          value: 100
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300  # รอ 5 นาทีก่อน scale down
      policies:
        - type: Percent
          value: 50
          periodSeconds: 60
```

---

## สรุป

| Pattern | ประโยชน์ | ตัวอย่าง Use Case |
|---------|---------|-----------------|
| Twelve-Factor | Best practices สำหรับ Cloud-Native | ทุก service |
| Sidecar | เพิ่ม functionality โดยไม่แก้ main app | Logging, Metrics, Config sync |
| Ambassador | Proxy ไปยัง external service | API calls, Retry logic |
| Adapter | แปลง interface ที่ไม่ compatible | Legacy metrics, Old APIs |
| Init Containers | Pre-condition checks | DB migration, Config download |
| Graceful Shutdown | ป้องกัน request drop | Rolling updates |
| Health Checks | Kubernetes lifecycle management | Auto-healing, Traffic routing |
| Resource Limits | ป้องกัน resource starvation | Production stability |
| Multi-Stage Build | ลดขนาด image, เพิ่ม security | Production images |
| PDB | HA during maintenance | Cluster upgrades |

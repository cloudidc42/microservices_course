# Part 85: Microservices Debugging

## บทนำ

การ debug ใน distributed system นั้นยากกว่า monolith มาก เพราะ error อาจเกิดจาก service ใดก็ได้ในหลาย services บทนี้จะให้ทั้ง tools, techniques, และ methodologies สำหรับ debug microservices อย่างมีประสิทธิภาพ

---

## Distributed Tracing Deep Dive

### การตั้งค่า OpenTelemetry

```typescript
// services/order/src/tracing.ts
// ต้อง import ก่อน modules อื่นทั้งหมด (top of main.ts)

import { NodeSDK } from '@opentelemetry/sdk-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';
import { Resource } from '@opentelemetry/resources';
import { SemanticResourceAttributes } from '@opentelemetry/semantic-conventions';
import {
  SimpleSpanProcessor,
  BatchSpanProcessor,
} from '@opentelemetry/sdk-trace-base';
import { HttpInstrumentation } from '@opentelemetry/instrumentation-http';
import { ExpressInstrumentation } from '@opentelemetry/instrumentation-express';
import { NestInstrumentation } from '@opentelemetry/instrumentation-nestjs-core';
import { PgInstrumentation } from '@opentelemetry/instrumentation-pg';
import { RedisInstrumentation } from '@opentelemetry/instrumentation-redis';

const sdk = new NodeSDK({
  resource: new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]: 'order-service',
    [SemanticResourceAttributes.SERVICE_VERSION]: process.env.APP_VERSION || '1.0.0',
    [SemanticResourceAttributes.DEPLOYMENT_ENVIRONMENT]: process.env.NODE_ENV || 'development',
    'service.team': 'order-team',
  }),

  spanProcessor: new BatchSpanProcessor(
    new OTLPTraceExporter({
      url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT || 'http://jaeger:4318/v1/traces',
      headers: {
        'X-Service-Token': process.env.OTEL_AUTH_TOKEN || '',
      },
    }),
    {
      maxQueueSize: 2048,
      maxExportBatchSize: 512,
      scheduledDelayMillis: 5000,
    }
  ),

  instrumentations: [
    new HttpInstrumentation({
      // Filter health checks from traces
      ignoreIncomingRequestHook: (request) => {
        return ['/health', '/metrics', '/ready'].includes(request.url || '');
      },
      // Add custom attributes to all spans
      requestHook: (span, request) => {
        span.setAttribute('http.request_id', 
          request.headers['x-request-id'] as string || ''
        );
      },
    }),
    new ExpressInstrumentation(),
    new NestInstrumentation(),
    new PgInstrumentation({
      enhancedDatabaseReporting: true, // เก็บ query parameters (ระวัง PII)
    }),
    new RedisInstrumentation(),
  ],
});

sdk.start();

// Graceful shutdown
process.on('SIGTERM', () => {
  sdk.shutdown().then(() => process.exit(0));
});
```

### Custom Span สำหรับ Business Logic

```typescript
// services/order/src/order.service.ts

import { Injectable } from '@nestjs/common';
import { trace, context, SpanStatusCode, SpanKind } from '@opentelemetry/api';

@Injectable()
export class OrderService {
  private readonly tracer = trace.getTracer('order-service');

  async createOrder(
    userId: string,
    dto: CreateOrderDto,
  ): Promise<Order> {
    return this.tracer.startActiveSpan(
      'order.create',
      {
        kind: SpanKind.INTERNAL,
        attributes: {
          'order.user_id': userId,
          'order.item_count': dto.items.length,
          'order.total_amount': dto.items.reduce(
            (sum, item) => sum + item.price * item.quantity, 0
          ),
        },
      },
      async (span) => {
        try {
          // Validate inventory
          const inventoryResult = await this.tracer.startActiveSpan(
            'order.validate_inventory',
            async (inventorySpan) => {
              try {
                const result = await this.inventoryService.validate(dto.items);
                inventorySpan.setAttribute('inventory.available', result.available);
                inventorySpan.setStatus({ code: SpanStatusCode.OK });
                return result;
              } catch (error) {
                inventorySpan.recordException(error as Error);
                inventorySpan.setStatus({ 
                  code: SpanStatusCode.ERROR,
                  message: error.message,
                });
                throw error;
              } finally {
                inventorySpan.end();
              }
            }
          );

          if (!inventoryResult.available) {
            span.setAttribute('order.failure_reason', 'INSUFFICIENT_INVENTORY');
            throw new BadRequestException('Insufficient inventory');
          }

          // Create order in DB
          const order = await this.tracer.startActiveSpan(
            'order.persist',
            async (dbSpan) => {
              try {
                const savedOrder = await this.orderRepo.save({
                  userId,
                  items: dto.items,
                  status: 'PENDING',
                });
                dbSpan.setAttribute('db.rows_affected', 1);
                dbSpan.setAttribute('order.id', savedOrder.id);
                return savedOrder;
              } finally {
                dbSpan.end();
              }
            }
          );

          span.setAttribute('order.id', order.id);
          span.setStatus({ code: SpanStatusCode.OK });
          return order;

        } catch (error) {
          span.recordException(error as Error);
          span.setStatus({ 
            code: SpanStatusCode.ERROR,
            message: (error as Error).message,
          });
          throw error;
        } finally {
          span.end();
        }
      }
    );
  }
}
```

---

## Log Correlation

### Structured Logging ที่ติด Trace Context

```typescript
// shared/src/logging/logger.service.ts

import { Injectable, LoggerService } from '@nestjs/common';
import { context, trace } from '@opentelemetry/api';
import * as winston from 'winston';

@Injectable()
export class CorrelatedLogger implements LoggerService {
  private readonly winston: winston.Logger;

  constructor() {
    this.winston = winston.createLogger({
      level: process.env.LOG_LEVEL || 'info',
      format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.errors({ stack: true }),
        winston.format.json(),
        // เพิ่ม trace context โดยอัตโนมัติ
        winston.format((info) => {
          const span = trace.getActiveSpan();
          if (span) {
            const spanContext = span.spanContext();
            info.traceId = spanContext.traceId;
            info.spanId = spanContext.spanId;
            info.traceFlags = spanContext.traceFlags;
          }
          return info;
        })(),
      ),
      defaultMeta: {
        service: process.env.SERVICE_NAME || 'unknown-service',
        version: process.env.APP_VERSION || '1.0.0',
        environment: process.env.NODE_ENV || 'development',
      },
      transports: [
        new winston.transports.Console({
          format: process.env.NODE_ENV === 'development'
            ? winston.format.combine(
                winston.format.colorize(),
                winston.format.simple()
              )
            : winston.format.json(),
        }),
      ],
    });
  }

  log(message: string, context?: string, metadata?: object): void {
    this.winston.info(message, { context, ...metadata });
  }

  error(message: string, trace?: string, context?: string): void {
    this.winston.error(message, { trace, context });
  }

  warn(message: string, context?: string): void {
    this.winston.warn(message, { context });
  }

  debug(message: string, context?: string): void {
    this.winston.debug(message, { context });
  }
}
```

```json
// ตัวอย่าง correlated log output:
{
  "timestamp": "2024-01-15T10:23:45.123Z",
  "level": "error",
  "message": "Failed to process payment",
  "service": "order-service",
  "version": "1.2.3",
  "environment": "production",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "spanId": "00f067aa0ba902b7",
  "traceFlags": 1,
  "context": "OrderService",
  "orderId": "ord-12345",
  "userId": "usr-67890",
  "error": {
    "message": "Payment gateway timeout",
    "stack": "Error: Payment gateway timeout\n    at PaymentService..."
  }
}
```

---

## kubectl Debugging Commands

### Essential Debugging Commands

```bash
#!/bin/bash
# scripts/debug-toolkit.sh
# รวม debugging commands ที่ใช้บ่อย

SERVICE_NAME="order-service"
NAMESPACE="production"

echo "=== Pod Status ==="
kubectl get pods -n $NAMESPACE -l app=$SERVICE_NAME -o wide

echo "=== Recent Events ==="
kubectl get events -n $NAMESPACE --field-selector involvedObject.name=$SERVICE_NAME \
  --sort-by='.lastTimestamp' | tail -20

echo "=== Resource Usage ==="
kubectl top pods -n $NAMESPACE -l app=$SERVICE_NAME

echo "=== Pod Logs (last 100 lines) ==="
kubectl logs -n $NAMESPACE -l app=$SERVICE_NAME \
  --tail=100 \
  --prefix \
  --timestamps

echo "=== Previous Container Logs (if crashed) ==="
kubectl logs -n $NAMESPACE -l app=$SERVICE_NAME \
  --previous \
  --tail=50

# Stream logs with filtering
echo "=== Stream ERROR logs ==="
kubectl logs -n $NAMESPACE -l app=$SERVICE_NAME \
  --follow \
  --timestamps \
  | grep -i "error\|exception\|fatal"
```

```bash
# Debug specific pod ที่มีปัญหา

# 1. Describe pod เพื่อดู events และ container status
kubectl describe pod -n production order-service-7d8f9c-xxxxx

# 2. Exec เข้าไปใน container เพื่อ inspect
kubectl exec -n production -it order-service-7d8f9c-xxxxx -- /bin/sh

# ภายใน container:
# Check environment variables
env | grep -E "DATABASE|REDIS|API_KEY"

# Check DNS resolution
nslookup user-service.production.svc.cluster.local

# Check connectivity
wget -qO- http://user-service:3000/health
curl -v http://product-service:3001/health

# Check file system
df -h
ls -la /app

# Check processes
ps aux

# 3. Port-forward สำหรับ local debugging
kubectl port-forward -n production service/order-service 3000:3000

# แล้วเรียก API จาก local machine:
curl http://localhost:3000/health
curl http://localhost:3000/api/v1/orders

# 4. Copy files จาก pod
kubectl cp production/order-service-7d8f9c-xxxxx:/app/logs/error.log ./error.log

# 5. Check ConfigMap และ Secrets
kubectl get configmap -n production order-service-config -o yaml
kubectl get secret -n production order-service-secrets -o jsonpath='{.data}' | base64 -d

# 6. Debug node affinity / scheduling issues
kubectl describe node <node-name> | grep -A 20 "Conditions:"
kubectl get pod -n production order-service-7d8f9c-xxxxx -o yaml | grep -A 10 "affinity"
```

### Debug Network Issues

```bash
# Deploy temporary debug pod
kubectl run -n production debug-pod \
  --image=nicolaka/netshoot \
  --rm \
  --restart=Never \
  -it \
  -- /bin/bash

# ภายใน debug-pod:

# Test DNS
dig order-service.production.svc.cluster.local

# Test TCP connectivity
nc -zv order-service 3000
nc -zv user-service 3000

# Check network policies
# (ต้องทำจาก outside pod)
kubectl get networkpolicies -n production -o yaml

# Trace network path
traceroute order-service

# Check if service endpoints are correct
kubectl get endpoints -n production order-service

# Check iptables rules (บน node)
kubectl debug node/<node-name> -it --image=ubuntu -- chroot /host bash
iptables -L -n -v | grep order-service
```

---

## Ephemeral Containers

```bash
# Kubernetes 1.23+ รองรับ Ephemeral Containers
# ใช้สำหรับ debug production containers ที่ไม่มี shell

# Add debug container ไปยัง running pod
kubectl debug -n production \
  -it order-service-7d8f9c-xxxxx \
  --image=busybox:latest \
  --target=order-service \
  -- sh

# ใช้ image ที่มี debugging tools
kubectl debug -n production \
  -it order-service-7d8f9c-xxxxx \
  --image=nicolaka/netshoot \
  --target=order-service \
  -- bash

# ภายใน ephemeral container:
# Share process namespace กับ main container
ls /proc/1/root/app  # เข้าถึง filesystem ของ main container
cat /proc/1/environ  # environment variables ของ main container

# Network debugging
ss -tlnp
netstat -tulpn

# Process inspection
cat /proc/1/cmdline | tr '\0' ' '
```

```yaml
# Distroless image debugging (containers ที่ไม่มี shell)
# services/order/kubernetes/debug-pod.yaml

apiVersion: v1
kind: Pod
metadata:
  name: order-service-debug
  namespace: production
  labels:
    app: order-service-debug
spec:
  # Shared process namespace
  shareProcessNamespace: true
  
  containers:
  - name: order-service
    image: company/order-service:latest
    # ... regular config ...
  
  # Ephemeral debug container (via kubectl debug is better,
  # but this shows the concept)
  - name: debugger
    image: nicolaka/netshoot
    command: ['sleep', '3600']
    securityContext:
      capabilities:
        add: ['SYS_PTRACE']
```

---

## Telepresence สำหรับ Local Development

### Setup Telepresence

```bash
# Install Telepresence
curl -fL https://app.getambassador.io/download/tel2/linux/amd64/latest/telepresence -o telepresence
chmod +x telepresence
sudo mv telepresence /usr/local/bin/

# Connect Telepresence ไปยัง Kubernetes cluster
telepresence connect

# ตอนนี้ local machine สามารถ access kubernetes services ได้โดยตรง
curl http://user-service.production:3000/health

# Intercept order-service traffic ไปยัง local instance
# (requests ที่ไปยัง order-service ใน k8s จะ route มา local แทน)
telepresence intercept order-service \
  --namespace production \
  --port 3000 \
  --env-file .env.k8s  # export environment variables จาก pod

# ตอนนี้รัน local service ที่ใช้ environment จาก production
npm run start:dev

# Intercept แค่ requests ที่มี specific header
telepresence intercept order-service \
  --namespace production \
  --port 3000 \
  --http-header "x-dev-user=john"

# ดู intercepts ที่ active
telepresence list

# หยุด intercept
telepresence leave order-service
```

### Local Development Workflow

```typescript
// .env.k8s จะถูก export โดย Telepresence อัตโนมัติ
// services/order/.env.k8s

/*
DATABASE_URL=postgresql://order_svc:pass@order-db.production:5432/orders
REDIS_URL=redis://redis.production:6379
USER_SERVICE_URL=http://user-service.production:3000
PRODUCT_SERVICE_URL=http://product-service.production:3001
*/

// services/order/src/main.ts
async function bootstrap() {
  // local service ใช้ environment จาก Kubernetes
  // แต่ code รันใน local machine → breakpoints ทำงาน!
  const app = await NestFactory.create(AppModule);
  
  // VSCode debugger สามารถ attach ได้
  const port = process.env.PORT || 3000;
  await app.listen(port);
  
  console.log(`Service running on port ${port}`);
  console.log(`Connected to K8s services via Telepresence`);
}
```

```json
// .vscode/launch.json - Debug configuration
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "launch",
      "name": "Debug Order Service (Telepresence)",
      "program": "${workspaceFolder}/dist/main.js",
      "preLaunchTask": "build",
      "envFile": "${workspaceFolder}/.env.k8s",
      "sourceMaps": true,
      "smartStep": true,
      "outFiles": ["${workspaceFolder}/dist/**/*.js"],
      "resolveSourceMapLocations": ["${workspaceFolder}/**", "!**/node_modules/**"]
    }
  ]
}
```

---

## Common Issues Checklist

```bash
#!/bin/bash
# scripts/debug-checklist.sh
# Run ก่อน escalate ทุกครั้ง

SERVICE=$1
NAMESPACE=${2:-production}

echo "🔍 Debugging: $SERVICE in $NAMESPACE"
echo "=================================================="

echo ""
echo "1️⃣  Pod Status"
kubectl get pods -n $NAMESPACE -l app=$SERVICE

echo ""
echo "2️⃣  Recent Events (last 1 hour)"
kubectl get events -n $NAMESPACE \
  --field-selector involvedObject.name=$SERVICE \
  --sort-by='.lastTimestamp' | \
  awk -v d="$(date -d '1 hour ago' +%s)" \
  'NR==1{print} NR>1 && $1 != "0s"{print}'

echo ""
echo "3️⃣  Resource Usage"
kubectl top pods -n $NAMESPACE -l app=$SERVICE 2>/dev/null || \
  echo "metrics-server not available"

echo ""
echo "4️⃣  Service Endpoints"
kubectl get endpoints -n $NAMESPACE $SERVICE

echo ""
echo "5️⃣  Recent Errors in Logs"
kubectl logs -n $NAMESPACE -l app=$SERVICE \
  --tail=200 --timestamps \
  | grep -iE "error|exception|fatal|panic" \
  | tail -20

echo ""
echo "6️⃣  HPA Status"
kubectl get hpa -n $NAMESPACE | grep $SERVICE

echo ""
echo "7️⃣  Resource Limits"
kubectl get pod -n $NAMESPACE -l app=$SERVICE \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{range .spec.containers[*]}  {.name}: CPU={.resources.limits.cpu}, Memory={.resources.limits.memory}{"\n"}{end}{end}'

echo ""
echo "8️⃣  ConfigMap Versions"
kubectl get configmap -n $NAMESPACE | grep $SERVICE

echo ""
echo "9️⃣  Secret Versions"
kubectl get secrets -n $NAMESPACE | grep $SERVICE

echo ""
echo "🔟  Network Policy"
kubectl get networkpolicies -n $NAMESPACE | grep $SERVICE
```

---

## Remote Debugging ใน Kubernetes

```yaml
# kubernetes/debug/order-service-debug.yaml
# Deploy debug version ที่เปิด debug port

apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service-debug
  namespace: staging
spec:
  replicas: 1
  selector:
    matchLabels:
      app: order-service-debug
  template:
    metadata:
      labels:
        app: order-service-debug
    spec:
      containers:
      - name: order-service
        image: company/order-service:debug
        command: ["node", "--inspect=0.0.0.0:9229", "dist/main.js"]
        ports:
        - containerPort: 3000
          name: http
        - containerPort: 9229
          name: debug
        env:
        - name: NODE_ENV
          value: "staging"
        securityContext:
          # ต้องการสำหรับ node inspect
          capabilities:
            add: ["SYS_PTRACE"]
---
apiVersion: v1
kind: Service
metadata:
  name: order-service-debug
  namespace: staging
spec:
  selector:
    app: order-service-debug
  ports:
  - name: http
    port: 3000
  - name: debug
    port: 9229
```

```bash
# Port-forward debug port
kubectl port-forward -n staging \
  service/order-service-debug \
  9229:9229 \
  3000:3000 &

# ตอนนี้เปิด Chrome DevTools:
# chrome://inspect
# หรือ VSCode debugger:
```

```json
// .vscode/launch.json สำหรับ remote K8s debugging
{
  "configurations": [
    {
      "type": "node",
      "request": "attach",
      "name": "Attach to K8s Pod",
      "port": 9229,
      "localRoot": "${workspaceFolder}",
      "remoteRoot": "/app",
      "sourceMaps": true,
      "skipFiles": ["<node_internals>/**"]
    }
  ]
}
```

---

## Debugging Performance Issues

```bash
# Clinic.js สำหรับ Node.js performance profiling

# Profile service ใน staging
kubectl exec -n staging -it order-service-xxx -- sh

# ภายใน container:
npm install -g clinic
clinic doctor -- node dist/main.js
# สร้าง HTML report

# หรือใช้ --prof flag:
node --prof dist/main.js
# แล้ว analyze:
node --prof-process isolate-*.log > profile.txt
```

```typescript
// services/order/src/debug/performance-monitor.ts

import { Injectable } from '@nestjs/common';

@Injectable()
export class PerformanceMonitor {
  // ตรวจจับ slow database queries
  async analyzeSlowQueries(): Promise<SlowQueryReport[]> {
    const result = await this.dataSource.query(`
      SELECT
        query,
        calls,
        total_time,
        mean_time,
        max_time,
        stddev_time,
        rows
      FROM pg_stat_statements
      WHERE mean_time > 100  -- queries ที่ใช้เวลา > 100ms
      ORDER BY mean_time DESC
      LIMIT 20
    `);

    return result.map(row => ({
      query: row.query,
      avgMs: parseFloat(row.mean_time),
      maxMs: parseFloat(row.max_time),
      callCount: parseInt(row.calls),
    }));
  }

  // Memory leak detection
  monitorMemory(): void {
    const threshold = 0.8; // 80% of heap limit
    
    setInterval(() => {
      const usage = process.memoryUsage();
      const heapUsedPercent = usage.heapUsed / usage.heapTotal;
      
      if (heapUsedPercent > threshold) {
        console.error('MEMORY_WARNING', {
          heapUsed: `${Math.round(usage.heapUsed / 1024 / 1024)}MB`,
          heapTotal: `${Math.round(usage.heapTotal / 1024 / 1024)}MB`,
          percentage: `${Math.round(heapUsedPercent * 100)}%`,
          rss: `${Math.round(usage.rss / 1024 / 1024)}MB`,
        });
        
        // Force GC (ถ้า run with --expose-gc flag)
        if (global.gc) {
          global.gc();
          console.log('Forced garbage collection');
        }
      }
    }, 30000); // check ทุก 30 วินาที
  }
}
```

---

## Distributed Debugging Workflow

```
การ Debug Incident ใน Production:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Step 1: Identify Scope
┌─────────────────────────────────────────────────┐
│ Alert fires: order creation error rate > 1%     │
│                                                  │
│ Check: ✓ Error started when?                    │
│ Check: ✓ Which service is erroring?             │
│ Check: ✓ Is it all users or subset?             │
└─────────────────────────────────────────────────┘
            │
            ▼
Step 2: Get Trace ID
┌─────────────────────────────────────────────────┐
│ From error log:                                 │
│ {"traceId":"4bf92f3577b34da6a3ce929d0e0e4736"}  │
│                                                  │
│ Open Jaeger: /search?traceID=4bf92f...          │
└─────────────────────────────────────────────────┘
            │
            ▼
Step 3: Follow the Trace
┌─────────────────────────────────────────────────┐
│ order-service (45ms) ──OK──►                   │
│   user-service (5ms)  ──OK──►                  │
│   product-service (8ms) ──OK──►                │
│   payment-service (30ms) ──ERROR──► ← HERE!    │
└─────────────────────────────────────────────────┘
            │
            ▼
Step 4: Analyze Error Span
┌─────────────────────────────────────────────────┐
│ Span: payment-service.charge                    │
│ Error: "Connection timeout to payment gateway"  │
│ Duration: 30000ms (timeout!)                    │
│ Tags:                                           │
│   payment.gateway: "stripe"                     │
│   payment.gateway.region: "us-east-1"          │
└─────────────────────────────────────────────────┘
            │
            ▼
Step 5: Check Service Logs
kubectl logs -n production -l app=payment-service |
  grep "4bf92f3577b34da6a3ce929d0e0e4736"
```

---

## สรุป

Debugging microservices ต้องการ systematic approach:

| เครื่องมือ | ใช้เมื่อ |
|-----------|---------|
| Distributed Tracing (Jaeger) | ติดตาม request ข้าม services |
| Structured Logs + correlation | หาต้นเหตุของ error |
| kubectl debug | inspect pods ที่ distroless |
| Ephemeral containers | debug without restart |
| Telepresence | local dev กับ K8s services |
| Port-forward | test service โดยตรง |
| Performance profiler | CPU/memory issues |

Golden rules สำหรับ debug microservices:
1. **Trace ID เป็น key** — ทุก request ต้องมี trace ID
2. **Logs ต้อง structured** — JSON + correlation IDs
3. **Never debug blind** — มี metrics, traces, logs ก่อนจะ debug
4. **Reproduce locally ก่อน** — ใช้ Telepresence intercept
5. **Fix root cause** — ไม่ใช่แค่ symptom

# Part 72: Production Readiness Checklist

## บทนำ

Production readiness คือการรับประกันว่า service ของเราพร้อมรับ load จริง มีความเสถียร สามารถ monitor และ debug ได้ และมี process สำหรับจัดการ incidents อย่างเป็นระบบ ใน Part นี้เราจะ walkthrough checklist ที่ครอบคลุมทุกด้าน

---

## 1. Pre-Launch Checklist

```typescript
// scripts/production-readiness-check.ts
interface CheckResult {
  category: string;
  check: string;
  status: "pass" | "fail" | "warn";
  message?: string;
}

class ProductionReadinessChecker {
  private results: CheckResult[] = [];

  async runAll(): Promise<void> {
    await Promise.all([
      this.checkConfiguration(),
      this.checkSecurity(),
      this.checkDatabase(),
      this.checkObservability(),
      this.checkPerformance(),
      this.checkResilience(),
      this.checkDocumentation(),
    ]);

    this.printReport();
  }

  private async checkConfiguration(): Promise<void> {
    // Required environment variables
    const requiredEnvVars = [
      "DATABASE_URL",
      "REDIS_URL",
      "JWT_SECRET",
      "LOG_LEVEL",
      "SERVICE_NAME",
      "PORT",
    ];

    for (const envVar of requiredEnvVars) {
      this.results.push({
        category: "Configuration",
        check: `Environment variable ${envVar}`,
        status: process.env[envVar] ? "pass" : "fail",
        message: process.env[envVar] ? undefined : `${envVar} is not set`,
      });
    }

    // JWT secret strength
    const jwtSecret = process.env.JWT_SECRET;
    if (jwtSecret) {
      this.results.push({
        category: "Configuration",
        check: "JWT secret length",
        status: jwtSecret.length >= 32 ? "pass" : "fail",
        message: jwtSecret.length < 32
          ? `JWT secret too short: ${jwtSecret.length} chars (need >= 32)`
          : undefined,
      });
    }

    // NODE_ENV
    this.results.push({
      category: "Configuration",
      check: "NODE_ENV is production",
      status: process.env.NODE_ENV === "production" ? "pass" : "warn",
      message: process.env.NODE_ENV !== "production"
        ? `NODE_ENV is '${process.env.NODE_ENV}', expected 'production'`
        : undefined,
    });
  }

  private async checkSecurity(): Promise<void> {
    // Check for common security misconfigurations
    const checks = [
      {
        check: "HTTPS enforced",
        condition: process.env.FORCE_HTTPS === "true",
        severity: "fail" as const,
      },
      {
        check: "CORS configured",
        condition: process.env.CORS_ORIGINS !== undefined,
        severity: "warn" as const,
      },
      {
        check: "Rate limiting enabled",
        condition: process.env.RATE_LIMIT_ENABLED !== "false",
        severity: "warn" as const,
      },
      {
        check: "Request size limit configured",
        condition: process.env.MAX_REQUEST_SIZE !== undefined,
        severity: "warn" as const,
      },
    ];

    for (const { check, condition, severity } of checks) {
      this.results.push({
        category: "Security",
        check,
        status: condition ? "pass" : severity,
      });
    }
  }

  private async checkDatabase(): Promise<void> {
    try {
      const { Pool } = await import("pg");
      const pool = new Pool({ connectionString: process.env.DATABASE_URL });

      // Connectivity
      await pool.query("SELECT 1");
      this.results.push({
        category: "Database",
        check: "Database connectivity",
        status: "pass",
      });

      // Connection pool
      const maxConnections = parseInt(
        process.env.DATABASE_MAX_CONNECTIONS ?? "10"
      );
      this.results.push({
        category: "Database",
        check: "Connection pool size",
        status: maxConnections >= 5 ? "pass" : "warn",
        message: maxConnections < 5
          ? `Pool size ${maxConnections} may be too small`
          : undefined,
      });

      // Check migrations
      const migrationResult = await pool.query(
        "SELECT COUNT(*) FROM schema_migrations"
      );
      this.results.push({
        category: "Database",
        check: "Migrations applied",
        status: "pass",
        message: `${migrationResult.rows[0].count} migrations applied`,
      });

      await pool.end();
    } catch (error) {
      this.results.push({
        category: "Database",
        check: "Database connectivity",
        status: "fail",
        message: (error as Error).message,
      });
    }
  }

  private async checkObservability(): Promise<void> {
    const checks = [
      { check: "Structured logging configured", condition: process.env.LOG_FORMAT === "json" },
      { check: "Metrics endpoint available", condition: process.env.METRICS_ENABLED !== "false" },
      { check: "Trace sampling rate configured", condition: process.env.TRACE_SAMPLE_RATE !== undefined },
      { check: "Health check endpoints configured", condition: process.env.HEALTH_CHECK_ENABLED !== "false" },
    ];

    for (const { check, condition } of checks) {
      this.results.push({
        category: "Observability",
        check,
        status: condition ? "pass" : "warn",
      });
    }
  }

  private async checkPerformance(): Promise<void> {
    this.results.push({
      category: "Performance",
      check: "Compression enabled",
      status: process.env.COMPRESSION_ENABLED !== "false" ? "pass" : "warn",
    });

    this.results.push({
      category: "Performance",
      check: "Caching configured",
      status: process.env.REDIS_URL ? "pass" : "warn",
      message: !process.env.REDIS_URL ? "Redis not configured — caching disabled" : undefined,
    });
  }

  private async checkResilience(): Promise<void> {
    const checks = [
      { check: "Circuit breaker configured", condition: process.env.CIRCUIT_BREAKER_ENABLED !== "false" },
      { check: "Retry configured", condition: process.env.RETRY_ENABLED !== "false" },
      { check: "Timeout configured", condition: process.env.REQUEST_TIMEOUT_MS !== undefined },
    ];

    for (const { check, condition } of checks) {
      this.results.push({
        category: "Resilience",
        check,
        status: condition ? "pass" : "warn",
      });
    }
  }

  private async checkDocumentation(): Promise<void> {
    const { existsSync } = await import("fs");

    const docs = [
      { file: "README.md", required: true },
      { file: "RUNBOOK.md", required: true },
      { file: "API.md", required: false },
      { file: ".env.example", required: true },
    ];

    for (const { file, required } of docs) {
      this.results.push({
        category: "Documentation",
        check: `${file} exists`,
        status: existsSync(file)
          ? "pass"
          : required
          ? "fail"
          : "warn",
      });
    }
  }

  private printReport(): void {
    const passed = this.results.filter((r) => r.status === "pass").length;
    const failed = this.results.filter((r) => r.status === "fail").length;
    const warned = this.results.filter((r) => r.status === "warn").length;

    console.log("\n========== Production Readiness Report ==========\n");

    const categories = [...new Set(this.results.map((r) => r.category))];
    for (const category of categories) {
      console.log(`\n${category}:`);
      const categoryResults = this.results.filter((r) => r.category === category);
      for (const result of categoryResults) {
        const icon = result.status === "pass" ? "✅" : result.status === "fail" ? "❌" : "⚠️";
        console.log(`  ${icon} ${result.check}${result.message ? ` — ${result.message}` : ""}`);
      }
    }

    console.log(`\n========== Summary ==========`);
    console.log(`Passed: ${passed} | Failed: ${failed} | Warnings: ${warned}`);

    if (failed > 0) {
      console.log("\n❌ NOT READY FOR PRODUCTION — fix failing checks first");
      process.exit(1);
    } else if (warned > 0) {
      console.log("\n⚠️  Ready with warnings — review warnings before deploying");
    } else {
      console.log("\n✅ READY FOR PRODUCTION");
    }
  }
}

// Run
const checker = new ProductionReadinessChecker();
checker.runAll().catch(console.error);
```

---

## 2. Observability Setup

### Structured Logging

```typescript
// observability/logger.ts
import winston from "winston";
import { v4 as uuidv4 } from "uuid";

const { combine, timestamp, json, errors, metadata } = winston.format;

export const logger = winston.createLogger({
  level: process.env.LOG_LEVEL ?? "info",
  format: combine(
    errors({ stack: true }),
    metadata({ fillExcept: ["message", "level", "timestamp", "service"] }),
    timestamp({ format: "YYYY-MM-DDTHH:mm:ss.SSSZ" }),
    json()
  ),
  defaultMeta: {
    service: process.env.SERVICE_NAME ?? "unknown",
    version: process.env.APP_VERSION ?? "unknown",
    environment: process.env.NODE_ENV ?? "development",
  },
  transports: [
    new winston.transports.Console({
      silent: process.env.NODE_ENV === "test",
    }),
  ],
});

// Correlation ID middleware
export function correlationIdMiddleware() {
  return (req: Request, res: Response, next: NextFunction) => {
    const correlationId =
      (req.headers["x-correlation-id"] as string) ?? uuidv4();

    req.correlationId = correlationId;
    res.setHeader("x-correlation-id", correlationId);

    // Create child logger with correlation ID
    req.logger = logger.child({ correlationId });

    next();
  };
}

// Request logging middleware
export function requestLoggingMiddleware() {
  return (req: Request, res: Response, next: NextFunction) => {
    const start = Date.now();

    req.logger?.info("Request received", {
      method: req.method,
      path: req.path,
      query: req.query,
      userAgent: req.headers["user-agent"],
      ip: req.ip,
    });

    res.on("finish", () => {
      const duration = Date.now() - start;
      const logLevel = res.statusCode >= 500 ? "error" : res.statusCode >= 400 ? "warn" : "info";

      req.logger?.[logLevel]("Request completed", {
        method: req.method,
        path: req.path,
        statusCode: res.statusCode,
        duration: `${duration}ms`,
        contentLength: res.get("content-length"),
      });
    });

    next();
  };
}
```

### Metrics Setup

```typescript
// observability/metrics.ts
import {
  register,
  collectDefaultMetrics,
  Counter,
  Histogram,
  Gauge,
  Summary,
} from "prom-client";

// Collect default Node.js metrics
collectDefaultMetrics({
  prefix: `${process.env.SERVICE_NAME}_`,
  labels: {
    service: process.env.SERVICE_NAME ?? "unknown",
    version: process.env.APP_VERSION ?? "unknown",
  },
});

// Business metrics
export const metrics = {
  // HTTP metrics
  httpRequestsTotal: new Counter({
    name: "http_requests_total",
    help: "Total HTTP requests",
    labelNames: ["method", "path", "status_code"],
  }),

  httpRequestDuration: new Histogram({
    name: "http_request_duration_seconds",
    help: "HTTP request duration",
    labelNames: ["method", "path", "status_code"],
    buckets: [0.01, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10],
  }),

  // Business metrics
  ordersCreated: new Counter({
    name: "orders_created_total",
    help: "Total orders created",
    labelNames: ["status"],
  }),

  orderProcessingDuration: new Histogram({
    name: "order_processing_duration_seconds",
    help: "Order processing duration",
    buckets: [0.1, 0.5, 1, 5, 10, 30, 60],
  }),

  activeOrders: new Gauge({
    name: "active_orders_current",
    help: "Currently active orders",
    labelNames: ["status"],
  }),

  // Database metrics
  dbQueryDuration: new Histogram({
    name: "db_query_duration_seconds",
    help: "Database query duration",
    labelNames: ["operation", "table"],
    buckets: [0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1],
  }),

  dbPoolSize: new Gauge({
    name: "db_pool_size",
    help: "Database connection pool size",
    labelNames: ["state"],
  }),

  // Cache metrics
  cacheHits: new Counter({
    name: "cache_hits_total",
    help: "Total cache hits",
    labelNames: ["cache_name"],
  }),

  cacheMisses: new Counter({
    name: "cache_misses_total",
    help: "Total cache misses",
    labelNames: ["cache_name"],
  }),

  // External service metrics
  externalServiceRequests: new Counter({
    name: "external_service_requests_total",
    help: "Total external service requests",
    labelNames: ["service", "operation", "status"],
  }),

  externalServiceDuration: new Histogram({
    name: "external_service_duration_seconds",
    help: "External service request duration",
    labelNames: ["service", "operation"],
    buckets: [0.05, 0.1, 0.25, 0.5, 1, 2.5, 5],
  }),
};

// Prometheus metrics endpoint
export function setupMetricsEndpoint(app: Express): void {
  app.get("/metrics", async (_req, res) => {
    try {
      res.set("Content-Type", register.contentType);
      res.end(await register.metrics());
    } catch (error) {
      res.status(500).end((error as Error).message);
    }
  });
}
```

### Distributed Tracing

```typescript
// observability/tracing.ts
import { NodeSDK } from "@opentelemetry/sdk-node";
import { getNodeAutoInstrumentations } from "@opentelemetry/auto-instrumentations-node";
import { OTLPTraceExporter } from "@opentelemetry/exporter-trace-otlp-http";
import { Resource } from "@opentelemetry/resources";
import { SemanticResourceAttributes } from "@opentelemetry/semantic-conventions";
import { BatchSpanProcessor } from "@opentelemetry/sdk-trace-base";
import { trace, context, SpanStatusCode } from "@opentelemetry/api";

export function initTracing(): NodeSDK {
  const exporter = new OTLPTraceExporter({
    url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT ?? "http://otel-collector:4318/v1/traces",
  });

  const sdk = new NodeSDK({
    resource: new Resource({
      [SemanticResourceAttributes.SERVICE_NAME]:
        process.env.SERVICE_NAME ?? "unknown",
      [SemanticResourceAttributes.SERVICE_VERSION]:
        process.env.APP_VERSION ?? "unknown",
      [SemanticResourceAttributes.DEPLOYMENT_ENVIRONMENT]:
        process.env.NODE_ENV ?? "development",
    }),
    spanProcessor: new BatchSpanProcessor(exporter),
    instrumentations: [
      getNodeAutoInstrumentations({
        "@opentelemetry/instrumentation-fs": { enabled: false }, // Too noisy
      }),
    ],
  });

  sdk.start();
  console.log("OpenTelemetry tracing initialized");
  return sdk;
}

// Custom tracing helper
export function withSpan<T>(
  name: string,
  fn: (span: ReturnType<typeof trace.getTracer>["startSpan"] extends (name: string) => infer S ? S : never) => Promise<T>,
  attributes?: Record<string, string | number | boolean>
): Promise<T> {
  const tracer = trace.getTracer(process.env.SERVICE_NAME ?? "unknown");

  return tracer.startActiveSpan(name, async (span) => {
    if (attributes) {
      span.setAttributes(attributes);
    }

    try {
      const result = await fn(span);
      span.setStatus({ code: SpanStatusCode.OK });
      return result;
    } catch (error) {
      span.setStatus({
        code: SpanStatusCode.ERROR,
        message: (error as Error).message,
      });
      span.recordException(error as Error);
      throw error;
    } finally {
      span.end();
    }
  });
}

// Usage
async function processOrder(orderId: string): Promise<void> {
  await withSpan(
    "order.process",
    async (span) => {
      span.setAttribute("order.id", orderId);

      const order = await withSpan("order.fetch", () =>
        orderRepo.findById(orderId)
      );

      await withSpan("payment.charge", () =>
        paymentService.charge(order.totalAmount)
      );
    },
    { "order.id": orderId }
  );
}
```

---

## 3. Alerting Configuration

```yaml
# monitoring/alert-rules.yaml
groups:
  - name: service.critical
    interval: 30s
    rules:
      # High error rate
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status_code=~"5.."}[5m]))
          /
          sum(rate(http_requests_total[5m])) > 0.05
        for: 2m
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "High error rate: {{ $value | humanizePercentage }}"
          description: |
            Service {{ $labels.service }} has error rate above 5% for 2 minutes.
            Current rate: {{ $value | humanizePercentage }}
          runbook_url: "https://runbooks.example.com/high-error-rate"
          dashboard_url: "https://grafana.example.com/d/service-overview"

      # Service down
      - alert: ServiceDown
        expr: up{job="microservices"} == 0
        for: 1m
        labels:
          severity: critical
          pagerduty: "true"
        annotations:
          summary: "Service {{ $labels.instance }} is down"
          runbook_url: "https://runbooks.example.com/service-down"

      # High latency
      - alert: HighLatency
        expr: |
          histogram_quantile(0.99,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
          ) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High p99 latency: {{ $value }}s"
          description: "Service {{ $labels.service }} p99 latency is {{ $value }}s"

  - name: database.alerts
    rules:
      # Database connection pool exhaustion
      - alert: DBPoolExhausted
        expr: |
          db_pool_size{state="waiting"} > 5
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Database connection pool has {{ $value }} waiting clients"

      # Slow queries
      - alert: SlowQueryDetected
        expr: |
          histogram_quantile(0.95,
            sum(rate(db_query_duration_seconds_bucket[5m])) by (le, operation)
          ) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Slow DB queries detected for {{ $labels.operation }}"

  - name: business.alerts
    rules:
      # Order processing failure rate
      - alert: OrderProcessingFailures
        expr: |
          rate(orders_created_total{status="failed"}[10m])
          /
          rate(orders_created_total[10m]) > 0.1
        for: 5m
        labels:
          severity: critical
          team: business
        annotations:
          summary: "High order failure rate: {{ $value | humanizePercentage }}"

      # Payment failures
      - alert: PaymentServiceDown
        expr: |
          sum(rate(external_service_requests_total{service="payment", status="error"}[5m]))
          /
          sum(rate(external_service_requests_total{service="payment"}[5m])) > 0.2
        for: 2m
        labels:
          severity: critical
          pagerduty: "true"
        annotations:
          summary: "Payment service failure rate: {{ $value | humanizePercentage }}"

  - name: infrastructure.alerts
    rules:
      # High memory usage
      - alert: HighMemoryUsage
        expr: |
          process_resident_memory_bytes
          /
          container_spec_memory_limit_bytes > 0.85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage in {{ $labels.pod }}"

      # High CPU usage
      - alert: HighCPUUsage
        expr: |
          rate(process_cpu_seconds_total[5m]) > 0.8
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage in {{ $labels.service }}"
```

---

## 4. Runbook

```markdown
# Order Service Runbook

## Service Overview

**Service:** order-service
**Team:** Backend Platform
**Tier:** Critical (Tier 1)
**On-call:** PagerDuty Escalation Policy: backend-oncall
**Slack:** #incidents, #backend-oncall

## Architecture

order-service → PostgreSQL (primary)
             → Redis (cache)
             → payment-service (HTTP)
             → inventory-service (HTTP)
             → kafka (events)

## Common Incidents

### 1. High Error Rate (5xx)

**Alert:** `HighErrorRate`
**Severity:** Critical

#### Diagnosis Steps

1. Check current error rate in Grafana:
   `https://grafana.example.com/d/order-service`

2. Check logs for patterns:
   ```
   kubectl logs -n production -l app=order-service --since=10m | jq 'select(.level=="error")'
   ```

3. Check dependencies:
   ```
   curl https://order-service.example.com/health
   ```

4. Check recent deployments:
   ```
   kubectl rollout history deployment/order-service -n production
   ```

#### Resolution

**If caused by bad deployment:**
```
kubectl rollout undo deployment/order-service -n production
```

**If caused by database:**
- Check DB connection pool: `kubectl describe hpa order-service -n production`
- Restart pods gradually: `kubectl rollout restart deployment/order-service -n production`

**If caused by downstream service:**
- Circuit breaker should activate automatically within 30 seconds
- Monitor circuit breaker state in Grafana

---

### 2. Service Down (No healthy pods)

**Alert:** `ServiceDown`
**Severity:** Critical — Page on-call immediately

#### Diagnosis

1. Check pod status:
   ```
   kubectl get pods -n production -l app=order-service
   kubectl describe pod -n production -l app=order-service
   ```

2. Check recent events:
   ```
   kubectl get events -n production --sort-by='.lastTimestamp' | tail -20
   ```

3. Check node health:
   ```
   kubectl get nodes
   kubectl describe node <node-name>
   ```

#### Resolution

1. Scale up if needed:
   ```
   kubectl scale deployment order-service -n production --replicas=5
   ```

2. Check for OOMKill:
   ```
   kubectl get pod -o json | jq '.status.containerStatuses[].lastState.terminated'
   ```

3. If node issue, drain and replace:
   ```
   kubectl cordon <node-name>
   kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
   ```

---

### 3. Database Connection Pool Exhausted

**Alert:** `DBPoolExhausted`
**Severity:** Warning → Critical if not resolved in 10 min

#### Diagnosis

1. Check current connections:
   ```sql
   SELECT count(*), state FROM pg_stat_activity
   WHERE application_name = 'order-service'
   GROUP BY state;
   ```

2. Check for long-running queries:
   ```sql
   SELECT pid, now() - pg_stat_activity.query_start AS duration, query, state
   FROM pg_stat_activity
   WHERE (now() - pg_stat_activity.query_start) > interval '5 minutes';
   ```

3. Kill long-running queries if needed:
   ```sql
   SELECT pg_terminate_backend(<pid>);
   ```

#### Resolution

1. Increase pool size temporarily (requires restart):
   ```
   kubectl set env deployment/order-service -n production DATABASE_MAX_CONNECTIONS=20
   ```

2. Scale up service instances to distribute connections:
   ```
   kubectl scale deployment order-service -n production --replicas=5
   ```
```

---

## 5. Incident Response Procedures

```typescript
// incident/incident-manager.ts

interface Incident {
  id: string;
  title: string;
  severity: "P1" | "P2" | "P3" | "P4";
  status: "detected" | "acknowledged" | "investigating" | "mitigating" | "resolved";
  startTime: Date;
  acknowledgedAt?: Date;
  resolvedAt?: Date;
  commander?: string;
  affectedServices: string[];
  timeline: IncidentEvent[];
  actions: string[];
}

interface IncidentEvent {
  timestamp: Date;
  author: string;
  message: string;
  type: "detection" | "acknowledgment" | "investigation" | "mitigation" | "resolution" | "update";
}

// SLO definitions
interface SLO {
  name: string;
  target: number;  // 0-1 (e.g., 0.999 = 99.9%)
  window: string;  // e.g., "30d"
  metric: string;
}

const serviceSLOs: SLO[] = [
  {
    name: "Availability",
    target: 0.999,   // 99.9% = 43.8 minutes downtime/month
    window: "30d",
    metric: 'sum(rate(http_requests_total{status_code!~"5.."}[30d])) / sum(rate(http_requests_total[30d]))',
  },
  {
    name: "Latency p99 < 500ms",
    target: 0.99,    // 99% of requests
    window: "30d",
    metric: 'histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[30d])) < 0.5',
  },
  {
    name: "Error Rate < 1%",
    target: 0.99,
    window: "30d",
    metric: '1 - (sum(rate(http_requests_total{status_code=~"5.."}[30d])) / sum(rate(http_requests_total[30d])))',
  },
];

// Error budget calculation
function calculateErrorBudget(slo: SLO, currentValue: number): {
  budget: number;
  consumed: number;
  remaining: number;
  burnRate: number;
} {
  const budget = 1 - slo.target;
  const consumed = 1 - currentValue;
  const remaining = budget - consumed;
  const burnRate = consumed / budget;

  return {
    budget,
    consumed,
    remaining,
    burnRate,
  };
}

// Incident severity classification
function classifyIncident(impact: {
  errorRate?: number;
  latencyP99?: number;
  affectedUsersPercent?: number;
  revenueImpactPerHour?: number;
}): "P1" | "P2" | "P3" | "P4" {
  if (
    (impact.errorRate ?? 0) > 0.5 ||
    (impact.affectedUsersPercent ?? 0) > 50 ||
    (impact.revenueImpactPerHour ?? 0) > 100000
  ) {
    return "P1"; // Service down, revenue impact > 100k/hr
  }

  if (
    (impact.errorRate ?? 0) > 0.1 ||
    (impact.latencyP99 ?? 0) > 5 ||
    (impact.affectedUsersPercent ?? 0) > 10
  ) {
    return "P2"; // Significant degradation
  }

  if (
    (impact.errorRate ?? 0) > 0.01 ||
    (impact.latencyP99 ?? 0) > 2 ||
    (impact.affectedUsersPercent ?? 0) > 1
  ) {
    return "P3"; // Minor degradation
  }

  return "P4"; // No user impact
}

// Response time SLAs
const incidentResponseSLAs = {
  P1: { acknowledge: 5, resolve: 60 },    // minutes
  P2: { acknowledge: 15, resolve: 240 },
  P3: { acknowledge: 60, resolve: 1440 }, // 24 hours
  P4: { acknowledge: 240, resolve: 4320 }, // 72 hours
};
```

---

## 6. On-call Rotation

```yaml
# pagerduty/on-call-rotation.yaml
# This is a documentation template — actual config goes in PagerDuty

schedule:
  name: "Backend On-Call"
  timezone: "Asia/Bangkok"

  layers:
    - name: "Primary"
      rotation_type: weekly
      start_day: Monday
      handoff_time: "09:00"
      users:
        - alice@example.com
        - bob@example.com
        - charlie@example.com
        - diana@example.com

    - name: "Secondary"
      rotation_type: weekly
      start_day: Monday
      handoff_time: "09:00"
      start_offset: 3  # Start 3 weeks offset from primary
      users:
        - eve@example.com
        - frank@example.com

escalation_policy:
  name: "Backend Escalation"
  rules:
    - delay_in_minutes: 0
      targets:
        - type: schedule
          name: "Backend On-Call (Primary)"

    - delay_in_minutes: 10  # If no ack in 10 min
      targets:
        - type: schedule
          name: "Backend On-Call (Secondary)"

    - delay_in_minutes: 20  # If still no ack
      targets:
        - type: user
          email: manager@example.com
```

---

## 7. Post-Mortem Process

```markdown
# Post-Mortem Template

## Incident Summary

| Field | Value |
|-------|-------|
| Date | 2024-01-15 |
| Duration | 47 minutes |
| Severity | P1 |
| Incident Commander | Alice |
| Services Affected | order-service, payment-service |
| Users Affected | ~15,000 users unable to checkout |
| Revenue Impact | ~฿450,000 |

## Timeline

| Time (UTC) | Event |
|------------|-------|
| 14:32 | Deployment of order-service v2.3.1 started |
| 14:35 | CPU usage began increasing on order-service pods |
| 14:38 | Alert: `HighErrorRate` fired (5% threshold) |
| 14:40 | Alice acknowledged alert, started investigation |
| 14:42 | Identified cause: N+1 query in new feature |
| 14:45 | Decision: rollback deployment |
| 14:47 | Rollback initiated |
| 14:52 | Error rate returning to normal |
| 15:19 | All pods healthy, incident resolved |

## Root Cause Analysis

### What Happened

Deploy v2.3.1 included a new feature that displayed order history with related product details. The implementation made one database query per order item instead of using a JOIN, resulting in N+1 queries that saturated the database connection pool.

### Contributing Factors

1. **Missing code review**: The PR was approved without load testing
2. **No performance test**: Integration tests didn't catch N+1 patterns
3. **No query monitoring**: No alert for slow query detection
4. **Rushed deployment**: Deployed during peak traffic (14:00-16:00)

## What Went Well

- Alert fired quickly (6 minutes after deploy)
- Incident commander acknowledged within 2 minutes
- Root cause identified in 4 minutes
- Rollback completed in 7 minutes
- Clear runbook reduced MTTR

## What Could Be Improved

- Code review process should include performance review checklist
- Add integration test for N+1 query detection
- Add slow query alerting in staging
- Enforce deployment freeze during peak hours (14:00-16:00 BKK time)

## Action Items

| Action | Owner | Due Date | Priority |
|--------|-------|----------|----------|
| Add N+1 query detection to CI | Alice | 2024-01-22 | High |
| Add slow query alert | Bob | 2024-01-19 | High |
| Update code review checklist | Alice | 2024-01-19 | Medium |
| Deploy freeze policy for peak hours | Charlie | 2024-01-25 | Medium |
| Load test suite for new features | Team | 2024-02-01 | Medium |

## Lessons Learned

1. Performance testing must be part of the PR review process
2. Deploy windows should avoid peak traffic hours
3. Database query patterns need monitoring in staging before production
```

---

## 8. SLO Definition และ Error Budget

```typescript
// slo/error-budget-calculator.ts
interface SLOConfig {
  name: string;
  description: string;
  target: number;    // e.g., 0.999 for 99.9%
  windowDays: number;
}

class ErrorBudgetManager {
  constructor(private readonly configs: SLOConfig[]) {}

  getErrorBudgetMinutes(slo: SLOConfig): number {
    const windowMinutes = slo.windowDays * 24 * 60;
    return windowMinutes * (1 - slo.target);
  }

  formatBudget(slo: SLOConfig): string {
    const minutes = this.getErrorBudgetMinutes(slo);
    if (minutes < 60) {
      return `${minutes.toFixed(1)} minutes`;
    }
    if (minutes < 1440) {
      return `${(minutes / 60).toFixed(1)} hours`;
    }
    return `${(minutes / 1440).toFixed(1)} days`;
  }

  calculateBurnRate(
    slo: SLOConfig,
    currentAvailability: number
  ): {
    burnRate: number;
    hoursUntilExhausted: number;
    status: "ok" | "warning" | "critical";
  } {
    const errorBudgetTotal = 1 - slo.target;
    const currentErrorRate = 1 - currentAvailability;
    const burnRate = currentErrorRate / errorBudgetTotal;

    const windowHours = slo.windowDays * 24;
    const hoursUntilExhausted =
      burnRate > 0 ? windowHours / burnRate : Infinity;

    const status =
      burnRate > 14.4  // Burning 1hr budget in 5 min
        ? "critical"
        : burnRate > 1
        ? "warning"
        : "ok";

    return { burnRate, hoursUntilExhausted, status };
  }

  printReport(measurements: Map<string, number>): void {
    console.log("\n======= Error Budget Report =======\n");

    for (const slo of this.configs) {
      const currentValue = measurements.get(slo.name) ?? slo.target;
      const { burnRate, hoursUntilExhausted, status } = this.calculateBurnRate(
        slo,
        currentValue
      );

      const icon =
        status === "ok" ? "✅" : status === "warning" ? "⚠️" : "🚨";

      console.log(`${icon} ${slo.name}`);
      console.log(`   Target: ${(slo.target * 100).toFixed(2)}%`);
      console.log(`   Current: ${(currentValue * 100).toFixed(3)}%`);
      console.log(`   Error Budget: ${this.formatBudget(slo)} per ${slo.windowDays}d`);
      console.log(`   Burn Rate: ${burnRate.toFixed(1)}x`);
      if (hoursUntilExhausted < Infinity) {
        console.log(
          `   Budget Exhausted In: ${hoursUntilExhausted.toFixed(1)} hours`
        );
      }
      console.log();
    }
  }
}

const sloManager = new ErrorBudgetManager([
  {
    name: "Availability",
    description: "HTTP success rate",
    target: 0.999,    // 99.9%
    windowDays: 30,   // Error budget: 43.8 minutes/month
  },
  {
    name: "Latency",
    description: "p99 < 500ms",
    target: 0.99,     // 99%
    windowDays: 30,   // Error budget: 7.2 hours/month
  },
]);
```

---

## 9. Kubernetes Production Deployment

```yaml
# k8s/production/order-service.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: production
  annotations:
    deployment.kubernetes.io/revision: "1"
    change-cause: "v2.3.2: fix N+1 query in order history"
spec:
  replicas: 5
  revisionHistoryLimit: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2
      maxUnavailable: 0
  selector:
    matchLabels:
      app: order-service
      version: v2.3.2
  template:
    metadata:
      labels:
        app: order-service
        version: v2.3.2
        team: backend
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "3000"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: order-service
      automountServiceAccountToken: false

      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: order-service
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: order-service

      containers:
        - name: order-service
          image: myregistry.example.com/order-service:v2.3.2
          imagePullPolicy: Always
          ports:
            - name: http
              containerPort: 3000
            - name: metrics
              containerPort: 9090

          env:
            - name: NODE_ENV
              value: "production"
            - name: SERVICE_NAME
              value: "order-service"
            - name: APP_VERSION
              value: "v2.3.2"
            - name: LOG_FORMAT
              value: "json"
            - name: LOG_LEVEL
              value: "info"
            - name: PORT
              value: "3000"
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: order-service-db
                  key: url
            - name: REDIS_URL
              valueFrom:
                secretKeyRef:
                  name: order-service-redis
                  key: url
            - name: JWT_SECRET
              valueFrom:
                secretKeyRef:
                  name: order-service-jwt
                  key: secret

          resources:
            requests:
              cpu: "200m"
              memory: "256Mi"
            limits:
              cpu: "1000m"
              memory: "512Mi"

          livenessProbe:
            httpGet:
              path: /healthz/live
              port: http
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3

          readinessProbe:
            httpGet:
              path: /healthz/ready
              port: http
            initialDelaySeconds: 10
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 2

          startupProbe:
            httpGet:
              path: /healthz/live
              port: http
            failureThreshold: 30
            periodSeconds: 10

          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 10"]

          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            runAsNonRoot: true
            runAsUser: 1000
            capabilities:
              drop: ["ALL"]

          volumeMounts:
            - name: tmp
              mountPath: /tmp

      volumes:
        - name: tmp
          emptyDir: {}

      terminationGracePeriodSeconds: 60

---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: order-service-pdb
  namespace: production
spec:
  minAvailable: 3
  selector:
    matchLabels:
      app: order-service

---
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
  minReplicas: 5
  maxReplicas: 30
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 70
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "100"
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
        - type: Percent
          value: 50
          periodSeconds: 30
        - type: Pods
          value: 3
          periodSeconds: 30
      selectPolicy: Max
```

---

## สรุป

ใน Part นี้เราได้เรียนรู้ Production Readiness ที่ครอบคลุมทุกด้าน:

1. **Pre-Launch Checklist** — ตรวจสอบ configuration, security, database, observability, และ documentation ก่อน deploy
2. **Observability Setup** — Structured logging, Prometheus metrics, Distributed tracing ด้วย OpenTelemetry
3. **Alerting Configuration** — Alert rules สำหรับ critical, warning, และ business metrics พร้อม runbook links
4. **Runbook** — เอกสาร step-by-step สำหรับจัดการ incidents แต่ละประเภท
5. **Incident Response** — กระบวนการ detect, acknowledge, investigate, mitigate, และ resolve incidents
6. **On-call Rotation** — ตารางเวร on-call และ escalation policy
7. **Post-mortem Process** — เรียนรู้จาก incidents เพื่อป้องกันการเกิดซ้ำ
8. **SLO และ Error Budget** — กำหนด targets ที่ชัดเจนและ monitor error budget consumption
9. **Production Kubernetes Config** — Deployment ที่ครบถ้วนพร้อม HPA, PDB, security contexts

Production readiness ไม่ใช่แค่ code ที่ทำงานได้ — แต่คือการมีระบบที่ monitor, alert, และ recover ได้อัตโนมัติ พร้อม process ที่ชัดเจนสำหรับทีมเมื่อเกิดปัญหา

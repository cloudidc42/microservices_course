# Part 54: Microservices Monitoring ขั้นสูง

## บทนำ

Monitoring ใน Microservices ต้องครอบคลุมหลายมิติ: Infrastructure Metrics, Application Metrics, Business Metrics, Distributed Tracing, และ Log Aggregation การมีระบบ Monitoring ที่ดีช่วยให้ทีมตรวจพบ Incident เร็ว, ลด MTTR (Mean Time to Recovery), และสร้าง Reliability เชิงรุกผ่าน SLI/SLO

---

## 1. Prometheus Custom Metrics

### 1.1 ติดตั้ง prom-client

```bash
npm install prom-client
npm install --save-dev @types/node
```

### 1.2 Metrics Registry

```typescript
// packages/shared/src/metrics/registry.ts

import {
  Counter,
  Gauge,
  Histogram,
  Summary,
  Registry,
  collectDefaultMetrics,
} from 'prom-client';

// สร้าง Registry แยกต่างหากจาก global register
export const metricsRegistry = new Registry();

// Collect default Node.js metrics (CPU, Memory, Event Loop, etc.)
collectDefaultMetrics({ register: metricsRegistry });

// ─── HTTP Metrics ──────────────────────────────────────────────────────────

export const httpRequestsTotal = new Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status_code', 'service'],
  registers: [metricsRegistry],
});

export const httpRequestDuration = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status_code', 'service'],
  buckets: [0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10],
  registers: [metricsRegistry],
});

export const httpRequestSize = new Histogram({
  name: 'http_request_size_bytes',
  help: 'Size of HTTP requests in bytes',
  labelNames: ['method', 'route', 'service'],
  buckets: [100, 1000, 10000, 100000, 1000000],
  registers: [metricsRegistry],
});

export const httpResponseSize = new Histogram({
  name: 'http_response_size_bytes',
  help: 'Size of HTTP responses in bytes',
  labelNames: ['method', 'route', 'service'],
  buckets: [100, 1000, 10000, 100000, 1000000],
  registers: [metricsRegistry],
});

export const httpActiveRequests = new Gauge({
  name: 'http_active_requests',
  help: 'Number of active HTTP requests',
  labelNames: ['service'],
  registers: [metricsRegistry],
});

// ─── Business Metrics ──────────────────────────────────────────────────────

export const ordersCreatedTotal = new Counter({
  name: 'orders_created_total',
  help: 'Total number of orders created',
  labelNames: ['status', 'channel'],
  registers: [metricsRegistry],
});

export const orderValueHistogram = new Histogram({
  name: 'order_value_baht',
  help: 'Distribution of order values in Thai Baht',
  labelNames: ['category'],
  buckets: [100, 500, 1000, 5000, 10000, 50000, 100000],
  registers: [metricsRegistry],
});

export const activeOrdersGauge = new Gauge({
  name: 'active_orders_total',
  help: 'Current number of active orders',
  labelNames: ['status'],
  registers: [metricsRegistry],
});

export const paymentProcessingDuration = new Summary({
  name: 'payment_processing_duration_seconds',
  help: 'Time to process payment',
  labelNames: ['payment_method', 'provider'],
  percentiles: [0.5, 0.9, 0.95, 0.99],
  maxAgeSeconds: 600,
  ageBuckets: 5,
  registers: [metricsRegistry],
});

// ─── Database Metrics ──────────────────────────────────────────────────────

export const dbConnectionsActive = new Gauge({
  name: 'db_connections_active',
  help: 'Number of active database connections',
  labelNames: ['database', 'host'],
  registers: [metricsRegistry],
});

export const dbQueryDuration = new Histogram({
  name: 'db_query_duration_seconds',
  help: 'Duration of database queries',
  labelNames: ['operation', 'table', 'database'],
  buckets: [0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1, 5],
  registers: [metricsRegistry],
});

export const dbQueryErrors = new Counter({
  name: 'db_query_errors_total',
  help: 'Total number of database query errors',
  labelNames: ['operation', 'table', 'error_type'],
  registers: [metricsRegistry],
});

// ─── Message Queue Metrics ─────────────────────────────────────────────────

export const mqMessagesPublished = new Counter({
  name: 'mq_messages_published_total',
  help: 'Total messages published to message queue',
  labelNames: ['exchange', 'routing_key'],
  registers: [metricsRegistry],
});

export const mqMessagesConsumed = new Counter({
  name: 'mq_messages_consumed_total',
  help: 'Total messages consumed from message queue',
  labelNames: ['queue', 'status'],
  registers: [metricsRegistry],
});

export const mqMessageProcessingDuration = new Histogram({
  name: 'mq_message_processing_duration_seconds',
  help: 'Time to process a message',
  labelNames: ['queue'],
  buckets: [0.01, 0.05, 0.1, 0.5, 1, 5, 10],
  registers: [metricsRegistry],
});

export const mqQueueDepth = new Gauge({
  name: 'mq_queue_depth',
  help: 'Current depth of message queue',
  labelNames: ['queue'],
  registers: [metricsRegistry],
});

// ─── Saga Metrics ──────────────────────────────────────────────────────────

export const sagaExecutionsTotal = new Counter({
  name: 'saga_executions_total',
  help: 'Total saga executions',
  labelNames: ['saga_type', 'status'],
  registers: [metricsRegistry],
});

export const sagaDuration = new Histogram({
  name: 'saga_duration_seconds',
  help: 'Duration of saga executions',
  labelNames: ['saga_type', 'status'],
  buckets: [0.1, 0.5, 1, 5, 10, 30, 60, 300],
  registers: [metricsRegistry],
});

export const sagaCompensationsTotal = new Counter({
  name: 'saga_compensations_total',
  help: 'Total saga compensations triggered',
  labelNames: ['saga_type', 'failed_step'],
  registers: [metricsRegistry],
});
```

---

## 2. HTTP Metrics Middleware

```typescript
// packages/shared/src/metrics/http.middleware.ts

import { Request, Response, NextFunction } from 'express';
import {
  httpRequestsTotal,
  httpRequestDuration,
  httpActiveRequests,
  httpRequestSize,
  httpResponseSize,
} from './registry';

export function createMetricsMiddleware(serviceName: string) {
  return (req: Request, res: Response, next: NextFunction): void => {
    const startTime = process.hrtime.bigint();

    httpActiveRequests.inc({ service: serviceName });

    // Capture request size
    const requestSize = parseInt(req.headers['content-length'] ?? '0', 10);

    res.on('finish', () => {
      const duration =
        Number(process.hrtime.bigint() - startTime) / 1_000_000_000;

      // Normalize route (replace dynamic segments)
      const route = req.route?.path ?? req.path ?? 'unknown';
      const normalizedRoute = normalizeRoute(route);

      const labels = {
        method: req.method,
        route: normalizedRoute,
        status_code: String(res.statusCode),
        service: serviceName,
      };

      httpRequestsTotal.inc(labels);
      httpRequestDuration.observe(labels, duration);

      if (requestSize > 0) {
        httpRequestSize.observe(
          { method: req.method, route: normalizedRoute, service: serviceName },
          requestSize
        );
      }

      const responseSize = parseInt(
        res.getHeader('content-length')?.toString() ?? '0',
        10
      );
      if (responseSize > 0) {
        httpResponseSize.observe(
          { method: req.method, route: normalizedRoute, service: serviceName },
          responseSize
        );
      }

      httpActiveRequests.dec({ service: serviceName });
    });

    res.on('error', () => {
      httpActiveRequests.dec({ service: serviceName });
    });

    next();
  };
}

// Normalize URL params เช่น /orders/123 → /orders/:id
function normalizeRoute(route: string): string {
  return route
    .replace(/\/[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}/gi, '/:uuid')
    .replace(/\/\d+/g, '/:id')
    .replace(/\/[a-z0-9]{24,}/gi, '/:hash');
}

// Metrics endpoint
import { metricsRegistry } from './registry';

export function metricsHandler() {
  return async (_req: Request, res: Response): Promise<void> => {
    res.set('Content-Type', metricsRegistry.contentType);
    res.end(await metricsRegistry.metrics());
  };
}
```

---

## 3. Database Instrumentation

```typescript
// packages/shared/src/metrics/db.instrumentation.ts

import { Pool, QueryConfig } from 'pg';
import {
  dbConnectionsActive,
  dbQueryDuration,
  dbQueryErrors,
} from './registry';

export function instrumentPool(pool: Pool, dbName: string, host: string): Pool {
  // Track connection count
  pool.on('connect', () => {
    dbConnectionsActive.inc({ database: dbName, host });
  });

  pool.on('remove', () => {
    dbConnectionsActive.dec({ database: dbName, host });
  });

  // Patch query method
  const originalQuery = pool.query.bind(pool);

  // @ts-expect-error — override query
  pool.query = async (
    queryOrConfig: string | QueryConfig,
    valuesOrCallback?: unknown[]
  ) => {
    const queryText =
      typeof queryOrConfig === 'string'
        ? queryOrConfig
        : queryOrConfig.text;

    const operation = detectOperation(queryText);
    const table = detectTable(queryText);
    const startTime = Date.now();

    try {
      const result = valuesOrCallback
        ? await originalQuery(queryOrConfig as string, valuesOrCallback as unknown[])
        : await originalQuery(queryOrConfig as QueryConfig);

      const durationSeconds = (Date.now() - startTime) / 1000;
      dbQueryDuration.observe(
        { operation, table, database: dbName },
        durationSeconds
      );

      return result;
    } catch (error) {
      const durationSeconds = (Date.now() - startTime) / 1000;
      dbQueryDuration.observe(
        { operation, table, database: dbName },
        durationSeconds
      );

      const errorType = classifyDbError(error as Error);
      dbQueryErrors.inc({ operation, table, error_type: errorType });

      throw error;
    }
  };

  return pool;
}

function detectOperation(query: string): string {
  const normalized = query.trim().toUpperCase();
  if (normalized.startsWith('SELECT')) return 'SELECT';
  if (normalized.startsWith('INSERT')) return 'INSERT';
  if (normalized.startsWith('UPDATE')) return 'UPDATE';
  if (normalized.startsWith('DELETE')) return 'DELETE';
  if (normalized.startsWith('BEGIN')) return 'BEGIN';
  if (normalized.startsWith('COMMIT')) return 'COMMIT';
  if (normalized.startsWith('ROLLBACK')) return 'ROLLBACK';
  return 'OTHER';
}

function detectTable(query: string): string {
  const match =
    query.match(/(?:FROM|INTO|UPDATE|TABLE)\s+["']?(\w+)["']?/i);
  return match?.[1] ?? 'unknown';
}

function classifyDbError(error: Error): string {
  const msg = error.message.toLowerCase();
  if (msg.includes('connection')) return 'connection_error';
  if (msg.includes('timeout')) return 'timeout';
  if (msg.includes('duplicate') || msg.includes('unique')) return 'constraint_violation';
  if (msg.includes('deadlock')) return 'deadlock';
  if (msg.includes('permission') || msg.includes('denied')) return 'permission_denied';
  return 'unknown';
}
```

---

## 4. SLI/SLO Tracking

```typescript
// packages/shared/src/slo/slo.tracker.ts

import { Counter, Gauge, Registry } from 'prom-client';
import { metricsRegistry } from '../metrics/registry';

export interface SLODefinition {
  name: string;
  description: string;
  target: number; // เช่น 0.999 = 99.9%
  window: '1h' | '6h' | '24h' | '7d' | '30d';
}

// SLI Metrics สำหรับคำนวณ Error Budget
export const sliGoodRequests = new Counter({
  name: 'sli_good_requests_total',
  help: 'Total good (successful) requests for SLI calculation',
  labelNames: ['service', 'slo_name'],
  registers: [metricsRegistry],
});

export const sliTotalRequests = new Counter({
  name: 'sli_total_requests_total',
  help: 'Total requests for SLI calculation',
  labelNames: ['service', 'slo_name'],
  registers: [metricsRegistry],
});

export const sloErrorBudgetRemaining = new Gauge({
  name: 'slo_error_budget_remaining_ratio',
  help: 'Remaining error budget as a ratio (0-1)',
  labelNames: ['service', 'slo_name', 'window'],
  registers: [metricsRegistry],
});

export const sloBurnRate = new Gauge({
  name: 'slo_burn_rate',
  help: 'Current burn rate of error budget',
  labelNames: ['service', 'slo_name', 'window'],
  registers: [metricsRegistry],
});

export class SLOTracker {
  private readonly slos: Map<string, SLODefinition> = new Map();

  constructor(
    private readonly serviceName: string,
    private readonly registry: Registry = metricsRegistry
  ) {}

  registerSLO(slo: SLODefinition): void {
    this.slos.set(slo.name, slo);
    console.log(`[SLO] Registered: ${slo.name} (target: ${slo.target * 100}%)`);
  }

  // เรียกเมื่อ Request สำเร็จ
  recordGoodRequest(sloName: string): void {
    sliGoodRequests.inc({ service: this.serviceName, slo_name: sloName });
    sliTotalRequests.inc({ service: this.serviceName, slo_name: sloName });
  }

  // เรียกเมื่อ Request ล้มเหลว
  recordBadRequest(sloName: string): void {
    sliTotalRequests.inc({ service: this.serviceName, slo_name: sloName });
  }

  // เรียกเพื่ออัปเดต error budget (ควรเรียกจาก Prometheus recording rule แทน)
  updateErrorBudget(sloName: string, window: string, remaining: number): void {
    sloErrorBudgetRemaining.set(
      { service: this.serviceName, slo_name: sloName, window },
      remaining
    );
  }
}
```

---

## 5. Prometheus Alert Rules

```yaml
# monitoring/prometheus/alerts/microservices.yml
groups:
  - name: availability
    interval: 30s
    rules:
      # ─── SLO: 99.9% Availability (30-day window) ─────────────────────────
      - alert: SLOAvailabilityBudgetBurn
        expr: |
          (
            sum(rate(sli_good_requests_total[1h])) by (service, slo_name)
            /
            sum(rate(sli_total_requests_total[1h])) by (service, slo_name)
          ) < 0.999
        for: 5m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "SLO availability below target for {{ $labels.service }}"
          description: |
            Service {{ $labels.service }} SLO {{ $labels.slo_name }} availability
            is {{ $value | humanizePercentage }} which is below the 99.9% target.
          runbook_url: "https://runbooks.internal/slo-availability"

      # ─── Burn Rate Alerts (Multi-window, Multi-burn-rate) ─────────────────
      # Fast burn: 14.4x burn rate over 1h window
      - alert: ErrorBudgetFastBurn
        expr: |
          (
            sum(rate(http_requests_total{status_code=~"5.."}[1h])) by (service)
            /
            sum(rate(http_requests_total[1h])) by (service)
          ) > (14.4 * 0.001)
        for: 2m
        labels:
          severity: critical
          page: "true"
        annotations:
          summary: "Fast error budget burn for {{ $labels.service }}"
          description: |
            Service {{ $labels.service }} is burning error budget at 14.4x normal rate.
            At this rate, the monthly error budget will be exhausted in ~2 hours.

      # Slow burn: 6x burn rate over 6h window
      - alert: ErrorBudgetSlowBurn
        expr: |
          (
            sum(rate(http_requests_total{status_code=~"5.."}[6h])) by (service)
            /
            sum(rate(http_requests_total[6h])) by (service)
          ) > (6 * 0.001)
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "Slow error budget burn for {{ $labels.service }}"
          description: |
            Service {{ $labels.service }} is burning error budget at 6x normal rate.
            At this rate, the monthly budget will be exhausted in ~5 days.

  - name: latency
    rules:
      - alert: HighLatencyP99
        expr: |
          histogram_quantile(
            0.99,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service, route)
          ) > 1.0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High P99 latency for {{ $labels.service }}:{{ $labels.route }}"
          description: |
            P99 latency is {{ $value | humanizeDuration }}
            for {{ $labels.service }}{{ $labels.route }}

      - alert: HighLatencyP50
        expr: |
          histogram_quantile(
            0.5,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service, route)
          ) > 0.5
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High median latency for {{ $labels.service }}"

  - name: infrastructure
    rules:
      - alert: ServiceDown
        expr: up{job="microservices"} == 0
        for: 1m
        labels:
          severity: critical
          page: "true"
        annotations:
          summary: "Service {{ $labels.instance }} is down"
          description: "{{ $labels.instance }} has been down for more than 1 minute."

      - alert: HighMemoryUsage
        expr: |
          (
            process_resident_memory_bytes
            / node_memory_MemTotal_bytes * on(instance) group_left()
            node_memory_MemTotal_bytes
          ) > 0.85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage on {{ $labels.instance }}"

      - alert: DatabaseConnectionPoolExhausted
        expr: |
          db_connections_active / db_connections_max > 0.9
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "DB connection pool nearly exhausted for {{ $labels.database }}"

  - name: business
    rules:
      - alert: OrderCreationAnomalous
        expr: |
          abs(
            rate(orders_created_total[5m])
            - avg_over_time(rate(orders_created_total[5m])[1h:5m])
          ) / stddev_over_time(rate(orders_created_total[5m])[1h:5m]) > 3
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Anomalous order creation rate detected"

      - alert: PaymentFailureRateHigh
        expr: |
          sum(rate(orders_created_total{status="payment_failed"}[5m]))
          /
          sum(rate(orders_created_total[5m])) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Payment failure rate above 5%"
          description: "Payment failure rate is {{ $value | humanizePercentage }}"
```

---

## 6. Recording Rules สำหรับ Performance

```yaml
# monitoring/prometheus/rules/recording.yml
groups:
  - name: http_aggregated
    interval: 30s
    rules:
      # Pre-compute error rate per service
      - record: job:http_error_rate:ratio_rate5m
        expr: |
          sum(rate(http_requests_total{status_code=~"5.."}[5m])) by (service)
          /
          sum(rate(http_requests_total[5m])) by (service)

      # Pre-compute request rate per service
      - record: job:http_request_rate:rate5m
        expr: |
          sum(rate(http_requests_total[5m])) by (service)

      # Pre-compute P95 latency per service
      - record: job:http_p95_latency:histogram_quantile
        expr: |
          histogram_quantile(
            0.95,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
          )

      # Pre-compute P99 latency per service
      - record: job:http_p99_latency:histogram_quantile
        expr: |
          histogram_quantile(
            0.99,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
          )

      # Error budget consumed (30d rolling)
      - record: job:slo_error_budget_consumed:ratio30d
        expr: |
          1 - (
            sum(increase(sli_good_requests_total[30d])) by (service, slo_name)
            /
            sum(increase(sli_total_requests_total[30d])) by (service, slo_name)
          )
```

---

## 7. Grafana Dashboard JSON

```json
{
  "title": "Microservices Overview",
  "uid": "microservices-overview",
  "tags": ["microservices", "slo"],
  "time": { "from": "now-6h", "to": "now" },
  "refresh": "30s",
  "panels": [
    {
      "id": 1,
      "title": "Request Rate (RPS)",
      "type": "timeseries",
      "gridPos": { "h": 8, "w": 12, "x": 0, "y": 0 },
      "targets": [
        {
          "expr": "sum(rate(http_requests_total[5m])) by (service)",
          "legendFormat": "{{service}}"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "reqps",
          "color": { "mode": "palette-classic" }
        }
      }
    },
    {
      "id": 2,
      "title": "Error Rate (%)",
      "type": "timeseries",
      "gridPos": { "h": 8, "w": 12, "x": 12, "y": 0 },
      "targets": [
        {
          "expr": "sum(rate(http_requests_total{status_code=~'5..'}[5m])) by (service) / sum(rate(http_requests_total[5m])) by (service) * 100",
          "legendFormat": "{{service}}"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "percent",
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "green", "value": null },
              { "color": "yellow", "value": 1 },
              { "color": "red", "value": 5 }
            ]
          }
        }
      }
    },
    {
      "id": 3,
      "title": "P99 Latency (ms)",
      "type": "timeseries",
      "gridPos": { "h": 8, "w": 12, "x": 0, "y": 8 },
      "targets": [
        {
          "expr": "histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)) * 1000",
          "legendFormat": "{{service}} p99"
        },
        {
          "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)) * 1000",
          "legendFormat": "{{service}} p95"
        }
      ],
      "fieldConfig": {
        "defaults": { "unit": "ms" }
      }
    },
    {
      "id": 4,
      "title": "Error Budget Remaining (30d)",
      "type": "bargauge",
      "gridPos": { "h": 8, "w": 12, "x": 12, "y": 8 },
      "targets": [
        {
          "expr": "1 - (sum(increase(sli_total_requests_total[30d])) by (service, slo_name) - sum(increase(sli_good_requests_total[30d])) by (service, slo_name)) / (sum(increase(sli_total_requests_total[30d])) by (service, slo_name) * 0.001)",
          "legendFormat": "{{service}} - {{slo_name}}"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "percentunit",
          "min": 0,
          "max": 1,
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "red", "value": null },
              { "color": "yellow", "value": 0.25 },
              { "color": "green", "value": 0.5 }
            ]
          }
        }
      }
    },
    {
      "id": 5,
      "title": "Active Orders by Status",
      "type": "stat",
      "gridPos": { "h": 4, "w": 8, "x": 0, "y": 16 },
      "targets": [
        {
          "expr": "sum(active_orders_total) by (status)",
          "legendFormat": "{{status}}"
        }
      ]
    },
    {
      "id": 6,
      "title": "DB Query Duration (P95)",
      "type": "timeseries",
      "gridPos": { "h": 8, "w": 12, "x": 0, "y": 20 },
      "targets": [
        {
          "expr": "histogram_quantile(0.95, sum(rate(db_query_duration_seconds_bucket[5m])) by (le, database, operation)) * 1000",
          "legendFormat": "{{database}}/{{operation}}"
        }
      ],
      "fieldConfig": {
        "defaults": { "unit": "ms" }
      }
    },
    {
      "id": 7,
      "title": "Burn Rate (1h vs 6h)",
      "type": "timeseries",
      "gridPos": { "h": 8, "w": 12, "x": 12, "y": 20 },
      "targets": [
        {
          "expr": "sum(rate(http_requests_total{status_code=~'5..'}[1h])) by (service) / sum(rate(http_requests_total[1h])) by (service) / 0.001",
          "legendFormat": "{{service}} 1h burn rate"
        },
        {
          "expr": "sum(rate(http_requests_total{status_code=~'5..'}[6h])) by (service) / sum(rate(http_requests_total[6h])) by (service) / 0.001",
          "legendFormat": "{{service}} 6h burn rate"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "custom": {
            "thresholdsStyle": { "mode": "line" }
          },
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "green", "value": null },
              { "color": "yellow", "value": 6 },
              { "color": "red", "value": 14.4 }
            ]
          }
        }
      }
    }
  ],
  "templating": {
    "list": [
      {
        "name": "service",
        "type": "query",
        "query": "label_values(http_requests_total, service)",
        "multi": true,
        "includeAll": true,
        "allValue": ".*"
      },
      {
        "name": "interval",
        "type": "interval",
        "options": ["1m", "5m", "10m", "30m", "1h"],
        "current": "5m"
      }
    ]
  }
}
```

---

## 8. PagerDuty Integration

```typescript
// packages/alertmanager/src/pagerduty.integration.ts

import axios from 'axios';

interface PagerDutyPayload {
  routing_key: string;
  event_action: 'trigger' | 'acknowledge' | 'resolve';
  dedup_key?: string;
  payload?: {
    summary: string;
    severity: 'critical' | 'error' | 'warning' | 'info';
    source: string;
    timestamp?: string;
    component?: string;
    group?: string;
    class?: string;
    custom_details?: Record<string, unknown>;
  };
  links?: Array<{ href: string; text: string }>;
  images?: Array<{ src: string; href?: string; alt?: string }>;
}

export class PagerDutyClient {
  private readonly baseUrl = 'https://events.pagerduty.com/v2/enqueue';

  constructor(private readonly integrationKey: string) {}

  async trigger(params: {
    dedupKey: string;
    summary: string;
    severity: 'critical' | 'error' | 'warning' | 'info';
    source: string;
    component?: string;
    group?: string;
    details?: Record<string, unknown>;
    runbookUrl?: string;
    dashboardUrl?: string;
  }): Promise<void> {
    const payload: PagerDutyPayload = {
      routing_key: this.integrationKey,
      event_action: 'trigger',
      dedup_key: params.dedupKey,
      payload: {
        summary: params.summary,
        severity: params.severity,
        source: params.source,
        timestamp: new Date().toISOString(),
        component: params.component,
        group: params.group,
        custom_details: params.details,
      },
      links: [
        ...(params.runbookUrl
          ? [{ href: params.runbookUrl, text: 'Runbook' }]
          : []),
        ...(params.dashboardUrl
          ? [{ href: params.dashboardUrl, text: 'Dashboard' }]
          : []),
      ],
    };

    await this.send(payload);
  }

  async resolve(dedupKey: string): Promise<void> {
    await this.send({
      routing_key: this.integrationKey,
      event_action: 'resolve',
      dedup_key: dedupKey,
    });
  }

  async acknowledge(dedupKey: string): Promise<void> {
    await this.send({
      routing_key: this.integrationKey,
      event_action: 'acknowledge',
      dedup_key: dedupKey,
    });
  }

  private async send(payload: PagerDutyPayload): Promise<void> {
    try {
      await axios.post(this.baseUrl, payload, {
        headers: { 'Content-Type': 'application/json' },
        timeout: 10000,
      });
    } catch (error) {
      console.error('[PagerDuty] Failed to send event:', error);
      throw error;
    }
  }
}

// Alertmanager Webhook Receiver
import express from 'express';

interface AlertmanagerAlert {
  status: 'firing' | 'resolved';
  labels: Record<string, string>;
  annotations: Record<string, string>;
  startsAt: string;
  endsAt: string;
  fingerprint: string;
}

interface AlertmanagerWebhook {
  version: string;
  groupKey: string;
  status: 'firing' | 'resolved';
  receiver: string;
  alerts: AlertmanagerAlert[];
  commonLabels: Record<string, string>;
  commonAnnotations: Record<string, string>;
}

export function createAlertmanagerWebhook(pagerDuty: PagerDutyClient) {
  const app = express();
  app.use(express.json());

  app.post('/webhook/pagerduty', async (req, res) => {
    const webhook: AlertmanagerWebhook = req.body;

    for (const alert of webhook.alerts) {
      const dedupKey = `${alert.labels.alertname}-${alert.labels.service ?? 'unknown'}-${alert.fingerprint}`;

      if (alert.status === 'firing') {
        const severity = mapSeverity(alert.labels.severity);

        await pagerDuty.trigger({
          dedupKey,
          summary: alert.annotations.summary ?? alert.labels.alertname,
          severity,
          source: alert.labels.service ?? 'unknown',
          component: alert.labels.route,
          group: alert.labels.namespace,
          details: {
            labels: alert.labels,
            annotations: alert.annotations,
            startsAt: alert.startsAt,
          },
          runbookUrl: alert.annotations.runbook_url,
          dashboardUrl: alert.annotations.dashboard_url,
        });

        console.log(`[Alertmanager] Triggered PD alert: ${dedupKey}`);
      } else {
        await pagerDuty.resolve(dedupKey);
        console.log(`[Alertmanager] Resolved PD alert: ${dedupKey}`);
      }
    }

    res.status(200).json({ status: 'ok' });
  });

  return app;
}

function mapSeverity(
  severity?: string
): 'critical' | 'error' | 'warning' | 'info' {
  switch (severity?.toLowerCase()) {
    case 'critical': return 'critical';
    case 'error': return 'error';
    case 'warning': return 'warning';
    default: return 'info';
  }
}
```

---

## 9. Runbook Automation

```typescript
// packages/runbook/src/runbook.executor.ts
// Automated runbook actions triggered by alerts

import { PagerDutyClient } from './pagerduty.integration';

interface RunbookAction {
  name: string;
  description: string;
  execute: (context: RunbookContext) => Promise<RunbookResult>;
}

interface RunbookContext {
  alertName: string;
  service: string;
  severity: string;
  labels: Record<string, string>;
  annotations: Record<string, string>;
}

interface RunbookResult {
  success: boolean;
  message: string;
  details?: Record<string, unknown>;
}

export class RunbookExecutor {
  private readonly runbooks: Map<string, RunbookAction[]> = new Map();

  constructor(private readonly pagerDuty: PagerDutyClient) {}

  registerRunbook(alertName: string, actions: RunbookAction[]): void {
    this.runbooks.set(alertName, actions);
  }

  async execute(context: RunbookContext): Promise<void> {
    const actions = this.runbooks.get(context.alertName);
    if (!actions?.length) {
      console.log(`[Runbook] No automated actions for: ${context.alertName}`);
      return;
    }

    console.log(
      `[Runbook] Executing ${actions.length} actions for: ${context.alertName}`
    );

    const results: Array<{ action: string; result: RunbookResult }> = [];

    for (const action of actions) {
      try {
        console.log(`[Runbook] Running: ${action.name}`);
        const result = await action.execute(context);
        results.push({ action: action.name, result });

        if (!result.success) {
          console.warn(`[Runbook] Action failed: ${action.name} — ${result.message}`);
        }
      } catch (error) {
        const result: RunbookResult = {
          success: false,
          message: (error as Error).message,
        };
        results.push({ action: action.name, result });
      }
    }

    // Update PagerDuty incident with runbook results
    const dedupKey = `${context.alertName}-${context.labels.service}-auto`;
    await this.pagerDuty.acknowledge(dedupKey);
  }
}

// Example: Auto-scaling runbook for high CPU
import { execSync } from 'child_process';

export function createKubernetesRunbook(
  kubeContext: string
): RunbookAction[] {
  return [
    {
      name: 'ScaleUpDeployment',
      description: 'Increase replica count when CPU is high',
      async execute(ctx) {
        try {
          const currentReplicas = parseInt(
            execSync(
              `kubectl get deployment ${ctx.service} -n microservices -o jsonpath='{.spec.replicas}' --context=${kubeContext}`
            ).toString().trim()
          );

          const newReplicas = Math.min(currentReplicas + 2, 10);

          execSync(
            `kubectl scale deployment ${ctx.service} --replicas=${newReplicas} -n microservices --context=${kubeContext}`
          );

          return {
            success: true,
            message: `Scaled ${ctx.service} from ${currentReplicas} to ${newReplicas} replicas`,
            details: { oldReplicas: currentReplicas, newReplicas },
          };
        } catch (error) {
          return {
            success: false,
            message: `Failed to scale: ${(error as Error).message}`,
          };
        }
      },
    },
    {
      name: 'RestartUnhealthyPods',
      description: 'Restart pods that are in CrashLoopBackOff',
      async execute(ctx) {
        try {
          const output = execSync(
            `kubectl get pods -n microservices -l app=${ctx.service} --field-selector=status.phase!=Running --context=${kubeContext} -o name`
          ).toString().trim();

          if (!output) {
            return { success: true, message: 'No unhealthy pods found' };
          }

          const pods = output.split('\n');
          for (const pod of pods) {
            execSync(
              `kubectl delete ${pod} -n microservices --context=${kubeContext}`
            );
          }

          return {
            success: true,
            message: `Restarted ${pods.length} unhealthy pods`,
            details: { pods },
          };
        } catch (error) {
          return {
            success: false,
            message: `Failed to restart pods: ${(error as Error).message}`,
          };
        }
      },
    },
  ];
}
```

---

## 10. Prometheus Configuration

```yaml
# monitoring/prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: production
    region: ap-southeast-1

rule_files:
  - /etc/prometheus/rules/*.yml
  - /etc/prometheus/alerts/*.yml

alerting:
  alertmanagers:
    - static_configs:
        - targets: ["alertmanager:9093"]
      timeout: 10s

scrape_configs:
  # Scrape Prometheus itself
  - job_name: prometheus
    static_configs:
      - targets: ["localhost:9090"]

  # Scrape all microservices via Consul SD
  - job_name: microservices
    consul_sd_configs:
      - server: consul:8500
        services: []
    relabel_configs:
      - source_labels: [__meta_consul_service]
        target_label: service
      - source_labels: [__meta_consul_tags]
        regex: .*,version=([^,]+),.*
        target_label: version
      - source_labels: [__meta_consul_service_address]
        target_label: __address__
        replacement: "${1}:${__meta_consul_service_port}"
      - source_labels: [__meta_consul_health]
        regex: passing
        action: keep

  # Kubernetes pod scraping
  - job_name: kubernetes-pods
    kubernetes_sd_configs:
      - role: pod
        namespaces:
          names: ["microservices"]
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: "true"
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        target_label: __address__
        regex: (\d+)
        replacement: "${__meta_kubernetes_pod_ip}:${1}"
      - source_labels: [__meta_kubernetes_pod_label_app]
        target_label: service
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
```

---

## 11. Docker Compose สำหรับ Monitoring Stack

```yaml
# docker-compose.monitoring.yml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:v2.52.0
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.path=/prometheus
      - --storage.tsdb.retention.time=30d
      - --web.enable-lifecycle
      - --web.enable-admin-api
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus:/etc/prometheus
      - prometheus_data:/prometheus
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:9090/-/healthy"]
      interval: 15s
      timeout: 10s
      retries: 3

  alertmanager:
    image: prom/alertmanager:v0.27.0
    ports:
      - "9093:9093"
    volumes:
      - ./monitoring/alertmanager:/etc/alertmanager
    command:
      - --config.file=/etc/alertmanager/alertmanager.yml
      - --storage.path=/alertmanager
      - --web.external-url=http://alertmanager:9093

  grafana:
    image: grafana/grafana:10.4.0
    environment:
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASSWORD:-admin123}
      GF_USERS_ALLOW_SIGN_UP: "false"
      GF_FEATURE_TOGGLES_ENABLE: traceqlEditor
      GF_SMTP_ENABLED: "true"
      GF_SMTP_HOST: smtp.gmail.com:587
      GF_SMTP_USER: ${SMTP_USER}
      GF_SMTP_PASSWORD: ${SMTP_PASSWORD}
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
      - ./monitoring/grafana/provisioning:/etc/grafana/provisioning
      - ./monitoring/grafana/dashboards:/var/lib/grafana/dashboards
    depends_on:
      - prometheus

  # Loki สำหรับ Log Aggregation
  loki:
    image: grafana/loki:2.9.0
    command: -config.file=/etc/loki/loki-config.yml
    ports:
      - "3100:3100"
    volumes:
      - ./monitoring/loki:/etc/loki
      - loki_data:/loki

  # Tempo สำหรับ Distributed Tracing
  tempo:
    image: grafana/tempo:2.4.0
    command: -config.file=/etc/tempo/tempo-config.yml
    ports:
      - "3200:3200"
      - "4317:4317"  # OTLP gRPC
      - "4318:4318"  # OTLP HTTP
    volumes:
      - ./monitoring/tempo:/etc/tempo
      - tempo_data:/var/tempo

  # Promtail สำหรับ Log Shipping
  promtail:
    image: grafana/promtail:2.9.0
    command: -config.file=/etc/promtail/promtail-config.yml
    volumes:
      - /var/log:/var/log
      - /var/lib/docker/containers:/var/lib/docker/containers
      - ./monitoring/promtail:/etc/promtail

volumes:
  prometheus_data:
  grafana_data:
  loki_data:
  tempo_data:
```

---

## 12. Error Budget Calculation Script

```typescript
// scripts/error-budget-report.ts
// Script สำหรับ generate Error Budget Report

import axios from 'axios';

const PROMETHEUS_URL = process.env.PROMETHEUS_URL ?? 'http://localhost:9090';

interface ErrorBudgetReport {
  service: string;
  sloName: string;
  target: number;
  window: string;
  totalRequests: number;
  goodRequests: number;
  actualAvailability: number;
  errorBudgetTotal: number;
  errorBudgetConsumed: number;
  errorBudgetRemaining: number;
  burnRate: number;
  projectedExhaustionDays: number | null;
}

async function queryPrometheus(query: string): Promise<number> {
  const response = await axios.get(`${PROMETHEUS_URL}/api/v1/query`, {
    params: { query },
  });

  const result = response.data.data.result;
  if (!result.length) return 0;
  return parseFloat(result[0].value[1]);
}

async function generateErrorBudgetReport(
  services: string[],
  sloTarget = 0.999,
  windowDays = 30
): Promise<ErrorBudgetReport[]> {
  const reports: ErrorBudgetReport[] = [];

  for (const service of services) {
    const window = `${windowDays}d`;

    const totalRequests = await queryPrometheus(
      `sum(increase(sli_total_requests_total{service="${service}"}[${window}]))`
    );

    const goodRequests = await queryPrometheus(
      `sum(increase(sli_good_requests_total{service="${service}"}[${window}]))`
    );

    if (totalRequests === 0) continue;

    const actualAvailability = goodRequests / totalRequests;
    const errorBudgetTotal = (1 - sloTarget) * totalRequests;
    const errorBudgetConsumed = totalRequests - goodRequests;
    const errorBudgetRemaining = errorBudgetTotal - errorBudgetConsumed;

    // Current burn rate (1h)
    const currentErrorRate = await queryPrometheus(
      `sum(rate(sli_total_requests_total{service="${service}"}[1h])) - sum(rate(sli_good_requests_total{service="${service}"}[1h]))`
    );

    const allowedErrorRate =
      (1 - sloTarget) *
      (await queryPrometheus(
        `sum(rate(sli_total_requests_total{service="${service}"}[1h]))`
      ));

    const burnRate =
      allowedErrorRate > 0 ? currentErrorRate / allowedErrorRate : 0;

    // Projected exhaustion
    const requestsPerHour = await queryPrometheus(
      `sum(rate(sli_total_requests_total{service="${service}"}[1h])) * 3600`
    );

    const projectedExhaustionDays =
      errorBudgetRemaining > 0 && requestsPerHour > 0
        ? errorBudgetRemaining / (currentErrorRate * 3600 * 24)
        : null;

    reports.push({
      service,
      sloName: 'availability',
      target: sloTarget,
      window,
      totalRequests: Math.round(totalRequests),
      goodRequests: Math.round(goodRequests),
      actualAvailability,
      errorBudgetTotal: Math.round(errorBudgetTotal),
      errorBudgetConsumed: Math.round(errorBudgetConsumed),
      errorBudgetRemaining: Math.round(errorBudgetRemaining),
      burnRate,
      projectedExhaustionDays,
    });
  }

  return reports;
}

async function main() {
  const services = [
    'order-service',
    'inventory-service',
    'payment-service',
    'fulfillment-service',
  ];

  const reports = await generateErrorBudgetReport(services);

  console.log('\n=== Error Budget Report (30-day rolling) ===\n');

  for (const report of reports) {
    const budgetPct = (report.errorBudgetRemaining / report.errorBudgetTotal) * 100;
    const availPct = (report.actualAvailability * 100).toFixed(4);

    console.log(`Service: ${report.service}`);
    console.log(`  SLO Target:          ${(report.target * 100).toFixed(1)}%`);
    console.log(`  Actual Availability: ${availPct}%`);
    console.log(`  Error Budget:`);
    console.log(`    Total:     ${report.errorBudgetTotal.toLocaleString()} requests`);
    console.log(`    Consumed:  ${report.errorBudgetConsumed.toLocaleString()} requests (${(100 - budgetPct).toFixed(1)}%)`);
    console.log(`    Remaining: ${Math.max(0, report.errorBudgetRemaining).toLocaleString()} requests (${Math.max(0, budgetPct).toFixed(1)}%)`);
    console.log(`  Burn Rate:   ${report.burnRate.toFixed(2)}x`);

    if (report.projectedExhaustionDays !== null) {
      if (report.projectedExhaustionDays < 7) {
        console.log(`  ⚠️  Budget exhaustion in: ${report.projectedExhaustionDays.toFixed(1)} days`);
      } else {
        console.log(`  Budget exhaustion in: ${report.projectedExhaustionDays.toFixed(1)} days`);
      }
    }

    console.log('');
  }
}

main().catch(console.error);
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Prometheus Custom Metrics** — สร้าง Counter/Gauge/Histogram/Summary สำหรับ HTTP, Database, Message Queue, และ Business Metrics

2. **SLI/SLO Tracking** — นิยาม SLI ที่วัดได้จริง และคำนวณ Error Budget จาก Prometheus metrics

3. **Multi-window Burn Rate Alerts** — ใช้ Fast Burn (14.4x/1h) และ Slow Burn (6x/6h) เพื่อ detect ปัญหาทั้ง immediate และ gradual

4. **Grafana Dashboard** — แสดง RPS, Error Rate, Latency Percentiles, และ Error Budget ในหน้าเดียว

5. **PagerDuty Integration** — Routing alerts ไปยัง on-call engineer พร้อม runbook links

6. **Runbook Automation** — Auto-scale หรือ restart pods เมื่อ alert trigger

**Key Principle:** SLO ที่ดีต้องสะท้อน User Experience จริง — วัด Availability จาก Successful User Actions ไม่ใช่แค่ HTTP 200

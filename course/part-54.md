# Part 54: Microservices Monitoring ขั้นสูง — Prometheus, Grafana, SLO, และ Multi-window Burn Rate Alerts

ในบทนี้เราจะเรียนรู้การสร้าง Observability stack ระดับ Production ครอบคลุม Custom Prometheus Metrics, Grafana Dashboards, AlertManager Rules, SLI/SLO Definition, Error Budget Calculation, และ PagerDuty Integration

---

## 1. Prometheus Custom Metrics ด้วย prom-client

```typescript
// src/monitoring/metrics.ts
import {
  register,
  Counter,
  Histogram,
  Gauge,
  Summary,
  collectDefaultMetrics,
  Registry,
} from 'prom-client';

// Create dedicated registry for this service
const serviceRegistry = new Registry();

// Collect Node.js default metrics (CPU, memory, event loop, GC)
collectDefaultMetrics({
  register: serviceRegistry,
  labels: {
    service: process.env.SERVICE_NAME || 'unknown',
    version: process.env.SERVICE_VERSION || '0.0.0',
    environment: process.env.NODE_ENV || 'development',
  },
  prefix: 'nodejs_',
  gcDurationBuckets: [0.001, 0.01, 0.1, 1, 2, 5],
  eventLoopMonitoringPrecision: 10,
});

// HTTP metrics
export const httpRequestDurationSeconds = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status_code', 'service'],
  buckets: [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5],
  registers: [serviceRegistry],
});

export const httpRequestsTotal = new Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status_code', 'service'],
  registers: [serviceRegistry],
});

export const httpRequestSizeBytes = new Histogram({
  name: 'http_request_size_bytes',
  help: 'Size of HTTP request bodies in bytes',
  labelNames: ['method', 'route'],
  buckets: [100, 1000, 10000, 100000, 1000000],
  registers: [serviceRegistry],
});

export const httpResponseSizeBytes = new Histogram({
  name: 'http_response_size_bytes',
  help: 'Size of HTTP response bodies in bytes',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [100, 1000, 10000, 100000, 1000000],
  registers: [serviceRegistry],
});

// Business metrics
export const ordersTotal = new Counter({
  name: 'orders_total',
  help: 'Total number of orders by status',
  labelNames: ['status', 'payment_method', 'currency'],
  registers: [serviceRegistry],
});

export const orderValueDollars = new Histogram({
  name: 'order_value_dollars',
  help: 'Order value distribution in dollars',
  labelNames: ['payment_method', 'currency'],
  buckets: [10, 25, 50, 100, 250, 500, 1000, 5000, 10000],
  registers: [serviceRegistry],
});

export const activeUsersGauge = new Gauge({
  name: 'active_users_total',
  help: 'Number of currently active users',
  labelNames: ['plan_type'],
  registers: [serviceRegistry],
});

// Database metrics
export const dbQueryDurationSeconds = new Histogram({
  name: 'db_query_duration_seconds',
  help: 'Duration of database queries',
  labelNames: ['operation', 'table', 'success'],
  buckets: [0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1, 5],
  registers: [serviceRegistry],
});

export const dbConnectionPoolSize = new Gauge({
  name: 'db_connection_pool_size',
  help: 'Database connection pool metrics',
  labelNames: ['state'], // 'idle', 'active', 'waiting'
  registers: [serviceRegistry],
});

// Cache metrics
export const cacheOperationsTotal = new Counter({
  name: 'cache_operations_total',
  help: 'Total cache operations',
  labelNames: ['operation', 'result', 'cache_name'],
  registers: [serviceRegistry],
});

export const cacheHitRatio = new Gauge({
  name: 'cache_hit_ratio',
  help: 'Cache hit ratio over last 5 minutes',
  labelNames: ['cache_name'],
  registers: [serviceRegistry],
});

// Message queue metrics
export const messageQueueDepth = new Gauge({
  name: 'message_queue_depth',
  help: 'Current message queue depth',
  labelNames: ['queue_name', 'priority'],
  registers: [serviceRegistry],
});

export const messageProcessingDuration = new Histogram({
  name: 'message_processing_duration_seconds',
  help: 'Time to process a message',
  labelNames: ['queue_name', 'message_type', 'success'],
  buckets: [0.01, 0.05, 0.1, 0.5, 1, 5, 30],
  registers: [serviceRegistry],
});

export const messageProcessingErrors = new Counter({
  name: 'message_processing_errors_total',
  help: 'Total message processing errors',
  labelNames: ['queue_name', 'message_type', 'error_type'],
  registers: [serviceRegistry],
});

// External dependency metrics
export const externalCallDuration = new Histogram({
  name: 'external_call_duration_seconds',
  help: 'Duration of calls to external services',
  labelNames: ['service', 'operation', 'status'],
  buckets: [0.01, 0.05, 0.1, 0.5, 1, 5],
  registers: [serviceRegistry],
});

export const circuitBreakerState = new Gauge({
  name: 'circuit_breaker_state',
  help: 'Circuit breaker state (0=closed, 1=open, 0.5=half-open)',
  labelNames: ['service', 'operation'],
  registers: [serviceRegistry],
});

// HTTP metrics middleware
export function metricsMiddleware() {
  return (req: express.Request, res: express.Response, next: express.NextFunction) => {
    const startTime = Date.now();
    const route = getRouteTemplate(req);
    
    // Track request size
    const requestSize = parseInt(req.headers['content-length'] || '0');
    if (requestSize > 0) {
      httpRequestSizeBytes.labels(req.method, route).observe(requestSize);
    }
    
    res.on('finish', () => {
      const duration = (Date.now() - startTime) / 1000;
      const statusCode = res.statusCode.toString();
      
      const labels = {
        method: req.method,
        route,
        status_code: statusCode,
        service: process.env.SERVICE_NAME || 'unknown',
      };
      
      httpRequestDurationSeconds.labels(labels).observe(duration);
      httpRequestsTotal.labels(labels).inc();
      
      // Track response size
      const responseSize = parseInt(res.getHeader('content-length') as string || '0');
      if (responseSize > 0) {
        httpResponseSizeBytes.labels(req.method, route, statusCode).observe(responseSize);
      }
    });
    
    next();
  };
}

function getRouteTemplate(req: express.Request): string {
  // Get express route template (e.g., /users/:id instead of /users/123)
  return req.route?.path || req.path.replace(/\/[0-9a-f-]{36}/g, '/:id') || 'unknown';
}

// Metrics endpoint
export function createMetricsRouter() {
  const router = express.Router();
  
  router.get('/metrics', async (req, res) => {
    // Basic auth for metrics endpoint
    const authHeader = req.headers.authorization;
    if (process.env.METRICS_AUTH_TOKEN && authHeader !== `Bearer ${process.env.METRICS_AUTH_TOKEN}`) {
      return res.status(401).json({ error: 'unauthorized' });
    }
    
    res.set('Content-Type', serviceRegistry.contentType);
    res.send(await serviceRegistry.metrics());
  });
  
  router.get('/health', (req, res) => {
    res.json({
      status: 'healthy',
      version: process.env.SERVICE_VERSION,
      timestamp: new Date().toISOString(),
    });
  });
  
  return router;
}

export { serviceRegistry };
```

---

## 2. Grafana Dashboard JSON

```json
{
  "title": "Order Service - Production Dashboard",
  "uid": "order-service-prod",
  "tags": ["microservices", "order-service", "production"],
  "refresh": "30s",
  "time": { "from": "now-3h", "to": "now" },
  "panels": [
    {
      "id": 1,
      "title": "Request Rate (RPS)",
      "type": "stat",
      "gridPos": { "x": 0, "y": 0, "w": 4, "h": 4 },
      "targets": [
        {
          "expr": "sum(rate(http_requests_total{service='order-service'}[5m]))",
          "legendFormat": "RPS"
        }
      ],
      "options": {
        "colorMode": "background",
        "graphMode": "area",
        "justifyMode": "center"
      },
      "fieldConfig": {
        "defaults": {
          "color": { "mode": "thresholds" },
          "thresholds": {
            "steps": [
              { "color": "green", "value": null },
              { "color": "yellow", "value": 100 },
              { "color": "red", "value": 500 }
            ]
          },
          "unit": "reqps",
          "mappings": []
        }
      }
    },
    {
      "id": 2,
      "title": "Error Rate",
      "type": "stat",
      "gridPos": { "x": 4, "y": 0, "w": 4, "h": 4 },
      "targets": [
        {
          "expr": "sum(rate(http_requests_total{service='order-service',status_code=~'5..'}[5m])) / sum(rate(http_requests_total{service='order-service'}[5m])) * 100",
          "legendFormat": "Error Rate %"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "color": { "mode": "thresholds" },
          "thresholds": {
            "steps": [
              { "color": "green", "value": null },
              { "color": "yellow", "value": 0.1 },
              { "color": "red", "value": 1 }
            ]
          },
          "unit": "percent",
          "max": 100,
          "min": 0
        }
      }
    },
    {
      "id": 3,
      "title": "P99 Latency",
      "type": "stat",
      "gridPos": { "x": 8, "y": 0, "w": 4, "h": 4 },
      "targets": [
        {
          "expr": "histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket{service='order-service'}[5m])) by (le)) * 1000",
          "legendFormat": "P99 Latency"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "color": { "mode": "thresholds" },
          "thresholds": {
            "steps": [
              { "color": "green", "value": null },
              { "color": "yellow", "value": 200 },
              { "color": "red", "value": 1000 }
            ]
          },
          "unit": "ms"
        }
      }
    },
    {
      "id": 4,
      "title": "SLO Compliance (30d)",
      "type": "stat",
      "gridPos": { "x": 12, "y": 0, "w": 4, "h": 4 },
      "targets": [
        {
          "expr": "(1 - (sum(increase(http_requests_total{service='order-service',status_code=~'5..'}[30d])) / sum(increase(http_requests_total{service='order-service'}[30d])))) * 100",
          "legendFormat": "Availability %"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "thresholds": {
            "steps": [
              { "color": "red", "value": null },
              { "color": "yellow", "value": 99 },
              { "color": "green", "value": 99.9 }
            ]
          },
          "unit": "percent",
          "min": 99,
          "max": 100
        }
      }
    },
    {
      "id": 10,
      "title": "HTTP Request Rate by Route",
      "type": "timeseries",
      "gridPos": { "x": 0, "y": 4, "w": 12, "h": 8 },
      "targets": [
        {
          "expr": "sum(rate(http_requests_total{service='order-service'}[5m])) by (route, method)",
          "legendFormat": "{{method}} {{route}}"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "reqps",
          "custom": {
            "drawStyle": "line",
            "lineInterpolation": "smooth",
            "fillOpacity": 10
          }
        }
      }
    },
    {
      "id": 11,
      "title": "Latency Percentiles",
      "type": "timeseries",
      "gridPos": { "x": 12, "y": 4, "w": 12, "h": 8 },
      "targets": [
        {
          "expr": "histogram_quantile(0.50, sum(rate(http_request_duration_seconds_bucket{service='order-service'}[5m])) by (le)) * 1000",
          "legendFormat": "P50"
        },
        {
          "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket{service='order-service'}[5m])) by (le)) * 1000",
          "legendFormat": "P95"
        },
        {
          "expr": "histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket{service='order-service'}[5m])) by (le)) * 1000",
          "legendFormat": "P99"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "ms",
          "custom": { "drawStyle": "line" }
        }
      }
    },
    {
      "id": 20,
      "title": "Orders Created per Minute",
      "type": "timeseries",
      "gridPos": { "x": 0, "y": 12, "w": 8, "h": 6 },
      "targets": [
        {
          "expr": "sum(rate(orders_total{status='created'}[1m])) * 60",
          "legendFormat": "Orders/min"
        }
      ]
    },
    {
      "id": 21,
      "title": "Order Value Distribution",
      "type": "histogram",
      "gridPos": { "x": 8, "y": 12, "w": 8, "h": 6 },
      "targets": [
        {
          "expr": "sum(rate(order_value_dollars_bucket[5m])) by (le)",
          "legendFormat": "{{le}}"
        }
      ]
    },
    {
      "id": 30,
      "title": "DB Connection Pool",
      "type": "timeseries",
      "gridPos": { "x": 0, "y": 18, "w": 8, "h": 6 },
      "targets": [
        {
          "expr": "db_connection_pool_size{state='active'}",
          "legendFormat": "Active"
        },
        {
          "expr": "db_connection_pool_size{state='idle'}",
          "legendFormat": "Idle"
        },
        {
          "expr": "db_connection_pool_size{state='waiting'}",
          "legendFormat": "Waiting"
        }
      ]
    },
    {
      "id": 31,
      "title": "Cache Hit Rate",
      "type": "timeseries",
      "gridPos": { "x": 8, "y": 18, "w": 8, "h": 6 },
      "targets": [
        {
          "expr": "sum(rate(cache_operations_total{result='hit'}[5m])) / sum(rate(cache_operations_total[5m])) * 100",
          "legendFormat": "Hit Rate %"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "percent",
          "min": 0,
          "max": 100
        }
      }
    }
  ]
}
```

---

## 3. AlertManager Rules

```yaml
# kubernetes/prometheus/alert-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: order-service-alerts
  namespace: monitoring
  labels:
    prometheus: kube-prometheus
    role: alert-rules
spec:
  groups:
    - name: order-service.availability
      interval: 30s
      rules:
        # Service down
        - alert: OrderServiceDown
          expr: |
            absent(up{job="order-service"} == 1)
          for: 1m
          labels:
            severity: critical
            team: backend
            service: order-service
          annotations:
            summary: "Order Service is down"
            description: "Order Service has been down for more than 1 minute"
            runbook_url: "https://wiki.example.com/runbooks/order-service-down"
            dashboard_url: "https://grafana.example.com/d/order-service-prod"
        
        # High error rate
        - alert: OrderServiceHighErrorRate
          expr: |
            (
              sum(rate(http_requests_total{service="order-service",status_code=~"5.."}[5m]))
              /
              sum(rate(http_requests_total{service="order-service"}[5m]))
            ) > 0.01
          for: 5m
          labels:
            severity: warning
            team: backend
          annotations:
            summary: "High error rate on Order Service"
            description: "Error rate is {{ $value | humanizePercentage }} over the last 5 minutes"
        
        - alert: OrderServiceCriticalErrorRate
          expr: |
            (
              sum(rate(http_requests_total{service="order-service",status_code=~"5.."}[5m]))
              /
              sum(rate(http_requests_total{service="order-service"}[5m]))
            ) > 0.05
          for: 2m
          labels:
            severity: critical
            team: backend
          annotations:
            summary: "Critical error rate on Order Service"
            description: "Error rate is {{ $value | humanizePercentage }}, SLO breach imminent"
    
    - name: order-service.latency
      rules:
        - alert: OrderServiceHighLatency
          expr: |
            histogram_quantile(0.95,
              sum(rate(http_request_duration_seconds_bucket{service="order-service"}[5m])) by (le, route)
            ) > 0.5
          for: 5m
          labels:
            severity: warning
            team: backend
          annotations:
            summary: "High P95 latency on {{ $labels.route }}"
            description: "P95 latency is {{ $value | humanizeDuration }} for route {{ $labels.route }}"
        
        - alert: OrderServiceP99LatencyBreach
          expr: |
            histogram_quantile(0.99,
              sum(rate(http_request_duration_seconds_bucket{service="order-service"}[10m])) by (le)
            ) > 2
          for: 10m
          labels:
            severity: critical
            team: backend
          annotations:
            summary: "P99 latency SLO breach"
            description: "P99 latency {{ $value | humanizeDuration }} exceeds 2s SLO"
    
    - name: order-service.slo
      # Multi-window burn rate alerts (Google SRE book pattern)
      rules:
        # Fast burn: consumes 2% error budget in 1 hour
        - alert: OrderServiceSLOFastBurn
          expr: |
            (
              sum(rate(http_requests_total{service="order-service",status_code=~"5.."}[1h]))
              /
              sum(rate(http_requests_total{service="order-service"}[1h]))
            ) > (14.4 * 0.001)
            and
            (
              sum(rate(http_requests_total{service="order-service",status_code=~"5.."}[5m]))
              /
              sum(rate(http_requests_total{service="order-service"}[5m]))
            ) > (14.4 * 0.001)
          labels:
            severity: critical
            team: backend
            slo_type: fast_burn
          annotations:
            summary: "Order Service SLO fast burn detected"
            description: |
              Error budget is burning at {{ $value | humanizePercentage }} x14.4 the normal rate.
              At this rate the monthly error budget will be exhausted in 1 hour.
        
        # Slow burn: consumes 5% error budget in 6 hours  
        - alert: OrderServiceSLOSlowBurn
          expr: |
            (
              sum(rate(http_requests_total{service="order-service",status_code=~"5.."}[6h]))
              /
              sum(rate(http_requests_total{service="order-service"}[6h]))
            ) > (6 * 0.001)
            and
            (
              sum(rate(http_requests_total{service="order-service",status_code=~"5.."}[30m]))
              /
              sum(rate(http_requests_total{service="order-service"}[30m]))
            ) > (6 * 0.001)
          labels:
            severity: warning
            team: backend
            slo_type: slow_burn
          annotations:
            summary: "Order Service SLO slow burn detected"
            description: |
              Error budget is burning at 6x the normal rate.
              5% of monthly error budget consumed in 6 hours.
    
    - name: order-service.infrastructure
      rules:
        - alert: OrderServiceHighMemory
          expr: |
            (
              process_resident_memory_bytes{job="order-service"}
              /
              container_spec_memory_limit_bytes{container="order-service"}
            ) > 0.9
          for: 5m
          labels:
            severity: warning
            team: platform
          annotations:
            summary: "High memory usage on Order Service pod {{ $labels.pod }}"
            description: "Memory usage is {{ $value | humanizePercentage }} of limit"
        
        - alert: DatabaseConnectionPoolExhausted
          expr: |
            db_connection_pool_size{state="waiting"} > 5
          for: 2m
          labels:
            severity: warning
            team: backend
          annotations:
            summary: "Database connection pool has waiting connections"
            description: "{{ $value }} connections waiting for a pool slot"
        
        - alert: MessageQueueDepthHigh
          expr: |
            message_queue_depth > 1000
          for: 5m
          labels:
            severity: warning
            team: backend
          annotations:
            summary: "Message queue depth high: {{ $labels.queue_name }}"
            description: "Queue {{ $labels.queue_name }} has {{ $value }} messages"
```

---

## 4. SLI/SLO Definition และ Error Budget Calculation

```typescript
// src/monitoring/slo.ts
import { Gauge, Counter } from 'prom-client';
import { serviceRegistry } from './metrics';

interface SLODefinition {
  name: string;
  description: string;
  target: number; // 0-1 (e.g., 0.999 for 99.9%)
  window: string; // e.g., '30d', '7d'
  sli: {
    goodEvents: string; // PromQL for good events
    totalEvents: string; // PromQL for total events
  };
}

const SLO_DEFINITIONS: SLODefinition[] = [
  {
    name: 'order_service_availability',
    description: 'Order Service HTTP availability (non-5xx responses)',
    target: 0.999, // 99.9% = 43.8 min/month downtime budget
    window: '30d',
    sli: {
      goodEvents: 'sum(increase(http_requests_total{service="order-service",status_code!~"5.."}[30d]))',
      totalEvents: 'sum(increase(http_requests_total{service="order-service"}[30d]))',
    },
  },
  {
    name: 'order_service_latency',
    description: 'Order API P95 latency under 500ms',
    target: 0.95, // 95% of requests under 500ms
    window: '30d',
    sli: {
      goodEvents: 'sum(increase(http_request_duration_seconds_bucket{service="order-service",le="0.5"}[30d]))',
      totalEvents: 'sum(increase(http_request_duration_seconds_count{service="order-service"}[30d]))',
    },
  },
  {
    name: 'order_creation_success_rate',
    description: 'Order creation succeeds 99.5% of time',
    target: 0.995,
    window: '30d',
    sli: {
      goodEvents: 'sum(increase(orders_total{status="created"}[30d]))',
      totalEvents: 'sum(increase(orders_total[30d]))',
    },
  },
];

// SLO metrics for tracking
const errorBudgetRemaining = new Gauge({
  name: 'slo_error_budget_remaining_ratio',
  help: 'Remaining error budget as a ratio (0-1)',
  labelNames: ['slo_name', 'window'],
  registers: [serviceRegistry],
});

const sloComplianceRate = new Gauge({
  name: 'slo_compliance_rate',
  help: 'Current SLO compliance rate (0-1)',
  labelNames: ['slo_name', 'window'],
  registers: [serviceRegistry],
});

// Error budget calculator
export class ErrorBudgetCalculator {
  calculateErrorBudget(
    sloTarget: number,
    windowHours: number
  ): {
    totalMinutesInWindow: number;
    allowedDowntimeMinutes: number;
    allowedErrorRate: number;
  } {
    const totalMinutesInWindow = windowHours * 60;
    const allowedDowntimeMinutes = totalMinutesInWindow * (1 - sloTarget);
    const allowedErrorRate = 1 - sloTarget;
    
    return {
      totalMinutesInWindow,
      allowedDowntimeMinutes,
      allowedErrorRate,
    };
  }
  
  calculateBurnRate(
    actualErrorRate: number,
    allowedErrorRate: number
  ): number {
    return actualErrorRate / allowedErrorRate;
  }
  
  calculateTimeToExhaustBudget(
    burnRate: number,
    remainingBudgetRatio: number,
    windowHours: number
  ): number {
    if (burnRate <= 1) return Infinity;
    return (remainingBudgetRatio * windowHours) / burnRate;
  }
  
  // Multi-window burn rate thresholds (from Google SRE workbook)
  getAlertThresholds(sloTarget: number): {
    fastBurn: { window: string; burnRateThreshold: number };
    slowBurn: { window: string; burnRateThreshold: number };
  } {
    const allowedErrorRate = 1 - sloTarget;
    
    return {
      // 2% of budget consumed in 1 hour → burn rate 14.4x
      fastBurn: {
        window: '1h',
        burnRateThreshold: 14.4,
      },
      // 5% of budget consumed in 6 hours → burn rate 6x
      slowBurn: {
        window: '6h',
        burnRateThreshold: 6,
      },
    };
  }
}

// SLO PromQL expressions
export const SLO_QUERIES = {
  // Error budget remaining (30-day window)
  errorBudgetRemaining: (sloTarget: number) => `
    1 - (
      (
        1 - (
          sum(increase(http_requests_total{service="order-service",status_code!~"5.."}[30d]))
          /
          sum(increase(http_requests_total{service="order-service"}[30d]))
        )
      ) / (1 - ${sloTarget})
    )
  `.trim(),
  
  // Multi-window burn rate for alerting
  burnRateShort: (errorRateThreshold: number) => `
    sum(rate(http_requests_total{service="order-service",status_code=~"5.."}[1h]))
    /
    sum(rate(http_requests_total{service="order-service"}[1h]))
    > ${errorRateThreshold}
  `.trim(),
  
  burnRateLong: (errorRateThreshold: number) => `
    sum(rate(http_requests_total{service="order-service",status_code=~"5.."}[6h]))
    /
    sum(rate(http_requests_total{service="order-service"}[6h]))
    > ${errorRateThreshold}
  `.trim(),
};
```

---

## 5. PagerDuty Integration

```yaml
# kubernetes/alertmanager/alertmanager-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: alertmanager-config
  namespace: monitoring
data:
  alertmanager.yml: |
    global:
      resolve_timeout: 5m
      pagerduty_url: https://events.pagerduty.com/v2/enqueue
      slack_api_url_file: /etc/alertmanager/secrets/slack-webhook-url
    
    # Alert routing tree
    route:
      receiver: default-receiver
      group_by: ['alertname', 'cluster', 'service']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 4h
      
      routes:
        # Critical alerts → PagerDuty (immediate page)
        - match:
            severity: critical
          receiver: pagerduty-critical
          group_wait: 0s
          repeat_interval: 1h
          routes:
            # SLO fast burn → very urgent
            - match:
                slo_type: fast_burn
              receiver: pagerduty-critical
              group_wait: 0s
              repeat_interval: 30m
        
        # Warning alerts → Slack (no page)
        - match:
            severity: warning
          receiver: slack-warnings
          group_wait: 5m
          repeat_interval: 6h
        
        # Info alerts → Slack only
        - match:
            severity: info
          receiver: slack-info
          group_wait: 10m
          repeat_interval: 24h
    
    receivers:
      - name: default-receiver
        slack_configs:
          - channel: '#alerts-default'
            send_resolved: true
            title: '{{ template "slack.default.title" . }}'
            text: '{{ template "slack.default.text" . }}'
      
      - name: pagerduty-critical
        pagerduty_configs:
          - routing_key: ${PAGERDUTY_SERVICE_KEY}
            description: '{{ template "pagerduty.default.description" . }}'
            client: 'AlertManager'
            client_url: '{{ template "pagerduty.default.clientURL" . }}'
            details:
              firing: '{{ template "pagerduty.default.instances" .Alerts.Firing }}'
              resolved: '{{ template "pagerduty.default.instances" .Alerts.Resolved }}'
              num_firing: '{{ .Alerts.Firing | len }}'
              num_resolved: '{{ .Alerts.Resolved | len }}'
              summary: '{{ .CommonAnnotations.summary }}'
              runbook: '{{ .CommonAnnotations.runbook_url }}'
              dashboard: '{{ .CommonAnnotations.dashboard_url }}'
            severity: '{{ if eq .CommonLabels.severity "critical" }}critical{{ else }}warning{{ end }}'
            links:
              - href: '{{ .CommonAnnotations.dashboard_url }}'
                text: 'Grafana Dashboard'
              - href: '{{ .CommonAnnotations.runbook_url }}'
                text: 'Runbook'
        slack_configs:
          - channel: '#alerts-critical'
            send_resolved: true
            color: '{{ if eq .Status "firing" }}danger{{ else }}good{{ end }}'
            title: '🚨 CRITICAL: {{ .CommonAnnotations.summary }}'
            text: |
              *Alert:* {{ .CommonAnnotations.description }}
              *Severity:* {{ .CommonLabels.severity }}
              *Service:* {{ .CommonLabels.service }}
              *Dashboard:* <{{ .CommonAnnotations.dashboard_url }}|Open Grafana>
              *Runbook:* <{{ .CommonAnnotations.runbook_url }}|View Runbook>
      
      - name: slack-warnings
        slack_configs:
          - channel: '#alerts-warnings'
            send_resolved: true
            color: 'warning'
            title: '⚠️ Warning: {{ .CommonAnnotations.summary }}'
            text: '{{ .CommonAnnotations.description }}'
      
      - name: slack-info
        slack_configs:
          - channel: '#alerts-info'
            send_resolved: false
            title: 'ℹ️ {{ .CommonAnnotations.summary }}'
            text: '{{ .CommonAnnotations.description }}'
    
    # Inhibition rules to suppress noisy alerts
    inhibit_rules:
      # If service is down, suppress high latency and error rate alerts
      - source_match:
          alertname: OrderServiceDown
        target_match_re:
          alertname: OrderService.*
        equal: ['service']
      
      # If critical alert firing, suppress warning for same service
      - source_match:
          severity: critical
        target_match:
          severity: warning
        equal: ['alertname', 'service']
```

### 5.1 PagerDuty Alert Integration TypeScript

```typescript
// src/monitoring/pagerduty.ts
import axios from 'axios';
import { log } from '../telemetry/logger';

interface PagerDutyEvent {
  routingKey: string;
  eventAction: 'trigger' | 'acknowledge' | 'resolve';
  dedupKey?: string;
  payload: {
    summary: string;
    severity: 'critical' | 'error' | 'warning' | 'info';
    source: string;
    timestamp?: string;
    component?: string;
    group?: string;
    class?: string;
    customDetails?: Record<string, unknown>;
  };
  links?: Array<{ href: string; text: string }>;
  images?: Array<{ src: string; href?: string; alt?: string }>;
}

export class PagerDutyClient {
  private readonly client = axios.create({
    baseURL: 'https://events.pagerduty.com/v2',
    timeout: 10000,
    headers: { 'Content-Type': 'application/json' },
  });
  
  async sendAlert(event: PagerDutyEvent): Promise<string> {
    try {
      const response = await this.client.post('/enqueue', event);
      
      log.info('PagerDuty alert sent', {
        dedupKey: event.dedupKey,
        action: event.eventAction,
        status: response.data.status,
      });
      
      return response.data.dedup_key;
    } catch (error) {
      log.error('Failed to send PagerDuty alert', error as Error, {
        summary: event.payload.summary,
      });
      throw error;
    }
  }
  
  async triggerAlert(options: {
    summary: string;
    severity: PagerDutyEvent['payload']['severity'];
    source?: string;
    component?: string;
    details?: Record<string, unknown>;
    runbookUrl?: string;
    dashboardUrl?: string;
  }): Promise<string> {
    const dedupKey = `${process.env.SERVICE_NAME}-${options.component || 'general'}-${Date.now()}`;
    
    return this.sendAlert({
      routingKey: process.env.PAGERDUTY_ROUTING_KEY!,
      eventAction: 'trigger',
      dedupKey,
      payload: {
        summary: options.summary,
        severity: options.severity,
        source: options.source || process.env.SERVICE_NAME || 'unknown',
        component: options.component,
        timestamp: new Date().toISOString(),
        customDetails: {
          environment: process.env.NODE_ENV,
          version: process.env.SERVICE_VERSION,
          ...options.details,
        },
      },
      links: [
        ...(options.runbookUrl ? [{ href: options.runbookUrl, text: 'Runbook' }] : []),
        ...(options.dashboardUrl ? [{ href: options.dashboardUrl, text: 'Dashboard' }] : []),
      ],
    });
  }
  
  async resolveAlert(dedupKey: string): Promise<void> {
    await this.sendAlert({
      routingKey: process.env.PAGERDUTY_ROUTING_KEY!,
      eventAction: 'resolve',
      dedupKey,
      payload: {
        summary: 'Alert resolved',
        severity: 'info',
        source: process.env.SERVICE_NAME || 'unknown',
      },
    });
  }
}

// Error budget tracker that pages when budget is low
export class ErrorBudgetMonitor {
  private readonly pd = new PagerDutyClient();
  private alertDedupKey: string | null = null;
  
  async checkAndAlert(
    currentErrorRate: number,
    sloTarget: number,
    windowDays: number
  ) {
    const allowedErrorRate = 1 - sloTarget;
    const burnRate = currentErrorRate / allowedErrorRate;
    const remainingBudgetPercent = Math.max(0, (1 - burnRate) * 100);
    
    // Update gauge
    errorBudgetRemainingGauge
      .labels(process.env.SERVICE_NAME!, `${windowDays}d`)
      .set(remainingBudgetPercent / 100);
    
    // Alert if budget is low
    if (remainingBudgetPercent < 10 && !this.alertDedupKey) {
      this.alertDedupKey = await this.pd.triggerAlert({
        summary: `Error budget is ${remainingBudgetPercent.toFixed(1)}% remaining for ${process.env.SERVICE_NAME}`,
        severity: remainingBudgetPercent < 5 ? 'critical' : 'warning',
        component: 'slo',
        details: {
          slo_target: `${(sloTarget * 100).toFixed(2)}%`,
          current_error_rate: `${(currentErrorRate * 100).toFixed(4)}%`,
          burn_rate: burnRate.toFixed(2),
          remaining_budget_percent: remainingBudgetPercent.toFixed(2),
        },
        runbookUrl: `https://wiki.example.com/runbooks/${process.env.SERVICE_NAME}-slo`,
        dashboardUrl: `https://grafana.example.com/d/order-service-prod?var-window=${windowDays}d`,
      });
    } else if (remainingBudgetPercent > 15 && this.alertDedupKey) {
      // Resolve alert if budget recovers
      await this.pd.resolveAlert(this.alertDedupKey);
      this.alertDedupKey = null;
    }
  }
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้ Microservices Monitoring ขั้นสูงครอบคลุม:

1. **prom-client Custom Metrics** — HTTP, business, database, cache, queue metrics พร้อม proper labels และ histogram buckets

2. **Grafana Dashboard JSON** — Complete dashboard ด้วย stat panels, time series, และ SLO tracking

3. **AlertManager Rules** — Multi-severity routing, inhibition rules, และ notification channel configuration

4. **SLI/SLO Definition** — Availability, latency, และ success rate SLOs พร้อม PromQL expressions

5. **Error Budget** — การคำนวณ burn rate, multi-window alerts (fast burn 1h/slow burn 6h)

6. **PagerDuty Integration** — Events API, severity routing, และ Error Budget Monitor ที่ pages เมื่อ budget ใกล้หมด

Key takeaways:
- ใช้ histogram แทน gauge สำหรับ latency เพื่อให้ calculate percentiles ได้
- Multi-window burn rate alerts ช่วยให้ alert เร็วขึ้นโดยไม่ false positive
- Error budget เป็น shared goal ระหว่าง dev และ ops ไม่ใช่แค่ ops metric
- AlertManager inhibition rules ป้องกัน alert storm เมื่อ root cause เดียวทำให้หลาย alerts fire

# Part 15: Logging & Distributed Tracing

## บทนำ

ใน Microservices หนึ่ง request อาจผ่าน 5-10 services ก่อนจะ complete ถ้าเกิด error เราต้องติดตามได้ว่า request นั้นผ่านอะไรไปบ้าง นั่นคือที่มาของ **Distributed Tracing**

---

## 15.1 The Three Pillars of Observability

```
Observability
├── Logs     - "What happened?" - detailed text records
├── Metrics  - "How is the system?" - numbers over time
└── Traces   - "How did it flow?" - request journey across services
```

---

## 15.2 Structured Logging

```javascript
// src/utils/logger.js - Production-grade logger

const winston = require('winston');
const { ElasticsearchTransport } = require('winston-elasticsearch');
const config = require('../config');

// Custom format สำหรับ structured logging
const structuredFormat = winston.format.combine(
  winston.format.timestamp({ format: 'YYYY-MM-DDTHH:mm:ss.SSSZ' }),
  winston.format.errors({ stack: true }),
  winston.format.metadata({
    fillExcept: ['message', 'level', 'timestamp', 'service'],
  }),
  winston.format.json()
);

// Development format - อ่านง่าย
const devFormat = winston.format.combine(
  winston.format.colorize(),
  winston.format.timestamp({ format: 'HH:mm:ss.SSS' }),
  winston.format.printf(({ timestamp, level, message, service, ...meta }) => {
    const metaStr = Object.keys(meta).length
      ? '\n' + JSON.stringify(meta, null, 2)
      : '';
    return `${timestamp} [${service}] ${level}: ${message}${metaStr}`;
  })
);

const transports = [
  new winston.transports.Console({
    format: config.env === 'production' ? structuredFormat : devFormat,
  }),
];

// Add file transport in production
if (config.env === 'production') {
  transports.push(
    new winston.transports.File({
      filename: 'logs/error.log',
      level: 'error',
      maxsize: 50 * 1024 * 1024, // 50MB
      maxFiles: 5,
      tailable: true,
    }),
    new winston.transports.File({
      filename: 'logs/combined.log',
      maxsize: 100 * 1024 * 1024, // 100MB
      maxFiles: 10,
      tailable: true,
    })
  );

  // Add Elasticsearch transport if configured
  if (config.elasticsearch?.node) {
    transports.push(
      new ElasticsearchTransport({
        level: 'info',
        clientOpts: {
          node: config.elasticsearch.node,
          auth: config.elasticsearch.auth,
        },
        indexPrefix: `logs-${config.serviceName}`,
        indexSuffixPattern: 'YYYY.MM.DD',
      })
    );
  }
}

const logger = winston.createLogger({
  level: config.logLevel || (config.env === 'production' ? 'info' : 'debug'),
  defaultMeta: {
    service: config.serviceName || 'unknown-service',
    version: process.env.APP_VERSION || '1.0.0',
    environment: config.env,
  },
  transports,
  // Don't exit on handled exceptions
  exitOnError: false,
});

// Add child logger with request context
logger.withRequest = function(req) {
  return this.child({
    requestId: req.requestId,
    method: req.method,
    path: req.path,
    userId: req.user?.id,
    traceId: req.headers['x-trace-id'],
    spanId: req.headers['x-span-id'],
  });
};

module.exports = logger;
```

---

## 15.3 Request Logging Middleware

```javascript
// src/middleware/requestLogger.js

const { v4: uuidv4 } = require('uuid');
const logger = require('../utils/logger');

module.exports = function requestLogger(req, res, next) {
  // Generate or pass through request ID
  req.requestId = req.headers['x-request-id'] || uuidv4();
  req.startTime = Date.now();

  // Create child logger with request context
  req.logger = logger.child({
    requestId: req.requestId,
    traceId: req.headers['x-b3-traceid'] || req.headers['x-trace-id'],
    spanId: req.headers['x-b3-spanid'],
    userId: null, // Will be set after auth
  });

  // Set response headers
  res.setHeader('X-Request-ID', req.requestId);

  // Log incoming request
  req.logger.info('Request received', {
    method: req.method,
    url: req.originalUrl,
    ip: req.ip,
    userAgent: req.get('user-agent'),
    contentType: req.get('content-type'),
    contentLength: req.get('content-length'),
    referer: req.get('referer'),
  });

  // Log outgoing response
  const logResponse = () => {
    const duration = Date.now() - req.startTime;

    // Update logger with user info (set by auth middleware)
    req.logger = req.logger.child({ userId: req.user?.id });

    const logLevel = res.statusCode >= 500 ? 'error'
      : res.statusCode >= 400 ? 'warn'
      : 'info';

    req.logger[logLevel]('Request completed', {
      method: req.method,
      url: req.originalUrl,
      statusCode: res.statusCode,
      duration: `${duration}ms`,
      responseSize: res.get('content-length'),
    });
  };

  res.on('finish', logResponse);
  res.on('close', logResponse);

  next();
};
```

---

## 15.4 Distributed Tracing with OpenTelemetry

OpenTelemetry เป็น standard สำหรับ distributed tracing ที่ใช้ได้กับทุก language

### Setup OpenTelemetry

```javascript
// src/tracing/index.js

const { NodeSDK } = require('@opentelemetry/sdk-node');
const { getNodeAutoInstrumentations } = require('@opentelemetry/auto-instrumentations-node');
const { JaegerExporter } = require('@opentelemetry/exporter-jaeger');
const { OTLPTraceExporter } = require('@opentelemetry/exporter-trace-otlp-http');
const { Resource } = require('@opentelemetry/resources');
const { SemanticResourceAttributes } = require('@opentelemetry/semantic-conventions');

function setupTracing(serviceName, options = {}) {
  const exporter = options.useJaeger
    ? new JaegerExporter({
        endpoint: process.env.JAEGER_ENDPOINT || 'http://jaeger:14268/api/traces',
      })
    : new OTLPTraceExporter({
        url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT || 'http://otel-collector:4318/v1/traces',
      });

  const sdk = new NodeSDK({
    resource: new Resource({
      [SemanticResourceAttributes.SERVICE_NAME]: serviceName,
      [SemanticResourceAttributes.SERVICE_VERSION]: process.env.APP_VERSION || '1.0.0',
      [SemanticResourceAttributes.DEPLOYMENT_ENVIRONMENT]: process.env.NODE_ENV || 'development',
    }),
    traceExporter: exporter,
    instrumentations: [
      getNodeAutoInstrumentations({
        // Auto-instrument HTTP, Express, PostgreSQL, MongoDB, Redis
        '@opentelemetry/instrumentation-http': {
          ignoreIncomingPaths: ['/health', '/metrics'],
          requestHook: (span, info) => {
            span.setAttributes({
              'http.request.body': JSON.stringify(info.request.body)?.slice(0, 200),
            });
          },
        },
        '@opentelemetry/instrumentation-express': {},
        '@opentelemetry/instrumentation-pg': {},
        '@opentelemetry/instrumentation-mongoose': {},
        '@opentelemetry/instrumentation-ioredis': {},
      }),
    ],
  });

  sdk.start();
  console.log(`OpenTelemetry SDK started for service: ${serviceName}`);

  process.on('SIGTERM', () => {
    sdk.shutdown()
      .then(() => console.log('Tracing terminated'))
      .catch(console.error);
  });

  return sdk;
}

module.exports = { setupTracing };
```

**สำคัญมาก:** ต้อง import tracing ก่อนทุกอย่างใน entry point

```javascript
// src/server.js - Entry point
// MUST be first import for instrumentation to work
require('./tracing').setupTracing('order-service');

// Then import everything else
const app = require('./app');
const config = require('./config');
```

### Manual Tracing (Custom Spans)

```javascript
// src/services/orderService.js
const opentelemetry = require('@opentelemetry/api');
const tracer = opentelemetry.trace.getTracer('order-service');

class OrderService {
  async createOrder(orderData) {
    // Create custom span
    const span = tracer.startSpan('order.create', {
      attributes: {
        'order.user_id': orderData.userId,
        'order.items_count': orderData.items.length,
        'order.total': orderData.total,
      },
    });

    try {
      // Create a child span for DB operation
      const dbSpan = tracer.startSpan('db.insert_order', {
        parent: opentelemetry.context.active(),
      });

      let order;
      try {
        order = await orderRepository.create(orderData);
        dbSpan.setAttributes({ 'db.rows_affected': 1 });
        dbSpan.setStatus({ code: opentelemetry.SpanStatusCode.OK });
      } catch (dbError) {
        dbSpan.recordException(dbError);
        dbSpan.setStatus({
          code: opentelemetry.SpanStatusCode.ERROR,
          message: dbError.message,
        });
        throw dbError;
      } finally {
        dbSpan.end();
      }

      // Add order ID to span after creation
      span.setAttribute('order.id', order.id);
      span.setStatus({ code: opentelemetry.SpanStatusCode.OK });

      return order;
    } catch (error) {
      span.recordException(error);
      span.setStatus({
        code: opentelemetry.SpanStatusCode.ERROR,
        message: error.message,
      });
      throw error;
    } finally {
      span.end();
    }
  }
}
```

### Propagating Trace Context

```javascript
// src/http/tracingClient.js

const opentelemetry = require('@opentelemetry/api');
const { W3CTraceContextPropagator } = require('@opentelemetry/core');
const axios = require('axios');

const propagator = new W3CTraceContextPropagator();

class TracingHttpClient {
  constructor(baseURL) {
    this.client = axios.create({ baseURL });

    // Inject trace context into outgoing requests
    this.client.interceptors.request.use((config) => {
      const headers = {};
      propagator.inject(
        opentelemetry.context.active(),
        headers,
        opentelemetry.defaultTextMapSetter
      );
      config.headers = { ...config.headers, ...headers };
      return config;
    });
  }

  async get(url, params) {
    const response = await this.client.get(url, { params });
    return response.data;
  }

  async post(url, data) {
    const response = await this.client.post(url, data);
    return response.data;
  }
}

module.exports = TracingHttpClient;
```

---

## 15.5 Correlation ID Pattern

```javascript
// src/utils/correlationId.js

const { AsyncLocalStorage } = require('async_hooks');
const { v4: uuidv4 } = require('uuid');

const asyncLocalStorage = new AsyncLocalStorage();

class CorrelationContext {
  static run(requestId, fn) {
    const store = {
      requestId: requestId || uuidv4(),
      startTime: Date.now(),
    };
    return asyncLocalStorage.run(store, fn);
  }

  static get() {
    return asyncLocalStorage.getStore();
  }

  static getRequestId() {
    return this.get()?.requestId;
  }
}

// Middleware
function correlationMiddleware(req, res, next) {
  const requestId = req.headers['x-request-id'] || uuidv4();
  req.requestId = requestId;
  res.setHeader('X-Request-ID', requestId);

  CorrelationContext.run(requestId, next);
}

// Logger that automatically includes correlation ID
const logger = require('./logger');

const correlationLogger = {
  info: (message, meta = {}) => logger.info(message, {
    ...meta,
    requestId: CorrelationContext.getRequestId(),
  }),
  warn: (message, meta = {}) => logger.warn(message, {
    ...meta,
    requestId: CorrelationContext.getRequestId(),
  }),
  error: (message, meta = {}) => logger.error(message, {
    ...meta,
    requestId: CorrelationContext.getRequestId(),
  }),
  debug: (message, meta = {}) => logger.debug(message, {
    ...meta,
    requestId: CorrelationContext.getRequestId(),
  }),
};

module.exports = { CorrelationContext, correlationMiddleware, correlationLogger };
```

---

## 15.6 ELK Stack Setup (Elasticsearch, Logstash, Kibana)

```yaml
# docker-compose.elk.yml
version: '3.8'

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - ES_JAVA_OPTS=-Xms1g -Xmx1g
      - xpack.security.enabled=false
      - xpack.security.enrollment.enabled=false
    ports:
      - "9200:9200"
    volumes:
      - elasticsearch-data:/usr/share/elasticsearch/data
    networks:
      - monitoring
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:9200 || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 10

  logstash:
    image: docker.elastic.co/logstash/logstash:8.11.0
    container_name: logstash
    depends_on:
      elasticsearch:
        condition: service_healthy
    ports:
      - "5044:5044"    # Beats input
      - "5000:5000/tcp" # Syslog input
      - "5000:5000/udp"
      - "9600:9600"    # Logstash API
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline:ro
      - ./logstash/config/logstash.yml:/usr/share/logstash/config/logstash.yml:ro
    networks:
      - monitoring

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    container_name: kibana
    depends_on:
      elasticsearch:
        condition: service_healthy
    ports:
      - "5601:5601"
    environment:
      ELASTICSEARCH_HOSTS: '["http://elasticsearch:9200"]'
    networks:
      - monitoring
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:5601/api/status || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 10

  # Filebeat - ships logs from containers
  filebeat:
    image: docker.elastic.co/beats/filebeat:8.11.0
    container_name: filebeat
    user: root
    depends_on:
      elasticsearch:
        condition: service_healthy
    volumes:
      - ./filebeat/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - monitoring
    command: filebeat -e -strict.perms=false

networks:
  monitoring:
    driver: bridge

volumes:
  elasticsearch-data:
```

### Logstash Pipeline

```
# logstash/pipeline/logstash.conf

input {
  beats {
    port => 5044
  }
  
  # Direct from applications
  tcp {
    port => 5000
    codec => json_lines
  }
}

filter {
  # Parse JSON logs
  if [message] =~ /^\{/ {
    json {
      source => "message"
      target => "parsed"
    }
    
    mutate {
      rename => { "[parsed][level]" => "level" }
      rename => { "[parsed][service]" => "service" }
      rename => { "[parsed][requestId]" => "requestId" }
      rename => { "[parsed][traceId]" => "traceId" }
      rename => { "[parsed][userId]" => "userId" }
      rename => { "[parsed][message]" => "log_message" }
    }
  }
  
  # Add geo-location for IP addresses
  if [ip] {
    geoip {
      source => "ip"
      target => "geoip"
    }
  }
  
  # Parse duration (remove "ms" suffix)
  if [duration] {
    mutate {
      gsub => ["duration", "ms", ""]
    }
    mutate {
      convert => { "duration" => "integer" }
    }
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "microservices-logs-%{+YYYY.MM.dd}"
  }
  
  # Stdout for debugging
  stdout { codec => rubydebug }
}
```

### Filebeat Configuration

```yaml
# filebeat/filebeat.yml

filebeat.inputs:
  - type: container
    paths:
      - '/var/lib/docker/containers/*/*.log'
    processors:
      - add_docker_metadata:
          host: "unix:///var/run/docker.sock"
      - decode_json_fields:
          fields: ["message"]
          target: ""
          overwrite_keys: true

output.logstash:
  hosts: ["logstash:5044"]

processors:
  - add_cloud_metadata: ~
  - add_host_metadata: ~
```

---

## 15.7 Jaeger for Distributed Tracing

```yaml
# docker-compose.jaeger.yml
version: '3.8'

services:
  jaeger:
    image: jaegertracing/all-in-one:1.52
    container_name: jaeger
    environment:
      - COLLECTOR_ZIPKIN_HOST_PORT=:9411
      - COLLECTOR_OTLP_ENABLED=true
    ports:
      - "16686:16686"  # Jaeger UI
      - "14268:14268"  # Jaeger collector HTTP
      - "4317:4317"    # OTLP gRPC
      - "4318:4318"    # OTLP HTTP
      - "9411:9411"    # Zipkin compatible
    networks:
      - monitoring

  # OpenTelemetry Collector (optional - aggregates before Jaeger)
  otel-collector:
    image: otel/opentelemetry-collector-contrib:0.91.0
    container_name: otel-collector
    command: ["--config=/etc/otel-collector-config.yaml"]
    volumes:
      - ./otel/config.yaml:/etc/otel-collector-config.yaml:ro
    ports:
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP
      - "8888:8888"   # Prometheus metrics
    depends_on:
      - jaeger
    networks:
      - monitoring

networks:
  monitoring:
    driver: bridge
```

### OpenTelemetry Collector Config

```yaml
# otel/config.yaml

receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 1s
    send_batch_size: 1024
  
  memory_limiter:
    limit_mib: 512
    spike_limit_mib: 128
    check_interval: 5s
  
  resource:
    attributes:
      - key: environment
        value: ${NODE_ENV}
        action: upsert

exporters:
  jaeger:
    endpoint: jaeger:14250
    tls:
      insecure: true
  
  prometheus:
    endpoint: "0.0.0.0:8888"
  
  logging:
    loglevel: debug

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch, resource]
      exporters: [jaeger, logging]
    
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [prometheus]
```

---

## 15.8 Prometheus & Grafana

```yaml
# docker-compose.monitoring.yml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:v2.48.0
    container_name: prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/usr/share/prometheus/console_libraries'
      - '--web.console.templates=/usr/share/prometheus/consoles'
      - '--storage.tsdb.retention.time=30d'
      - '--web.enable-lifecycle'
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./prometheus/rules:/etc/prometheus/rules:ro
      - prometheus-data:/prometheus
    networks:
      - monitoring

  grafana:
    image: grafana/grafana:10.2.2
    container_name: grafana
    depends_on:
      - prometheus
    ports:
      - "3001:3000"
    environment:
      GF_SECURITY_ADMIN_USER: admin
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASSWORD:-admin123}
      GF_USERS_ALLOW_SIGN_UP: "false"
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/dashboards:/var/lib/grafana/dashboards:ro
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
    networks:
      - monitoring

  alertmanager:
    image: prom/alertmanager:v0.26.0
    container_name: alertmanager
    command:
      - '--config.file=/etc/alertmanager/config.yml'
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager/config.yml:/etc/alertmanager/config.yml:ro
      - alertmanager-data:/alertmanager
    networks:
      - monitoring

  # Node exporter - system metrics
  node-exporter:
    image: prom/node-exporter:v1.7.0
    container_name: node-exporter
    command:
      - '--path.procfs=/host/proc'
      - '--path.rootfs=/rootfs'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    ports:
      - "9100:9100"
    networks:
      - monitoring

networks:
  monitoring:
    driver: bridge

volumes:
  prometheus-data:
  grafana-data:
  alertmanager-data:
```

### Prometheus Configuration

```yaml
# prometheus/prometheus.yml

global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: 'microservices-dev'
    environment: 'development'

rule_files:
  - /etc/prometheus/rules/*.yml

scrape_configs:
  # API Gateway
  - job_name: 'api-gateway'
    static_configs:
      - targets: ['api-gateway:3000']
    metrics_path: '/metrics'

  # User Service
  - job_name: 'user-service'
    static_configs:
      - targets: ['user-service:3001']
    metrics_path: '/metrics'

  # Product Service
  - job_name: 'product-service'
    static_configs:
      - targets: ['product-service:3002']
    metrics_path: '/metrics'

  # Order Service
  - job_name: 'order-service'
    static_configs:
      - targets: ['order-service:3003']
    metrics_path: '/metrics'

  # Node Exporter (system metrics)
  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']

  # PostgreSQL Exporter
  - job_name: 'postgresql'
    static_configs:
      - targets: ['postgres-exporter:9187']

  # Redis Exporter
  - job_name: 'redis'
    static_configs:
      - targets: ['redis-exporter:9121']

  # RabbitMQ
  - job_name: 'rabbitmq'
    static_configs:
      - targets: ['rabbitmq:15692']

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']
```

### Prometheus Alerting Rules

```yaml
# prometheus/rules/microservices.yml

groups:
  - name: microservices
    rules:
      # High error rate
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status_code=~"5.."}[5m])) by (service)
          /
          sum(rate(http_requests_total[5m])) by (service)
          > 0.05
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "High error rate for {{ $labels.service }}"
          description: "Error rate is {{ $value | humanizePercentage }} for {{ $labels.service }}"

      # High response time
      - alert: HighResponseTime
        expr: |
          histogram_quantile(0.95,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (service, le)
          ) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High response time for {{ $labels.service }}"
          description: "P95 response time is {{ $value }}s for {{ $labels.service }}"

      # Service down
      - alert: ServiceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Service {{ $labels.job }} is down"
          description: "{{ $labels.job }} has been down for more than 1 minute"

      # High memory usage
      - alert: HighMemoryUsage
        expr: |
          process_resident_memory_bytes / 1024 / 1024 > 500
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage for {{ $labels.job }}"
          description: "Memory usage is {{ $value | humanize }}MB for {{ $labels.job }}"

      # Circuit breaker open
      - alert: CircuitBreakerOpen
        expr: circuit_breaker_state > 0
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Circuit breaker open for {{ $labels.service }}"
          description: "Circuit breaker for {{ $labels.service }} has been open"
```

---

## 15.9 Log Aggregation Pattern

```javascript
// src/utils/auditLogger.js
// Audit log for security and compliance

const logger = require('./logger');
const { pool } = require('../database');

class AuditLogger {
  async log(action, options = {}) {
    const auditEntry = {
      timestamp: new Date().toISOString(),
      action,
      userId: options.userId,
      targetType: options.targetType,
      targetId: options.targetId,
      requestId: options.requestId,
      ip: options.ip,
      userAgent: options.userAgent,
      details: options.details,
      result: options.result || 'success',
    };

    // Log to structured logs
    logger.info('Audit event', auditEntry);

    // Also persist to database for compliance
    try {
      await pool.query(
        `INSERT INTO audit_logs
         (action, user_id, target_type, target_id, request_id, ip_address, user_agent, details, result)
         VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9)`,
        [
          action,
          options.userId,
          options.targetType,
          options.targetId,
          options.requestId,
          options.ip,
          options.userAgent,
          JSON.stringify(options.details),
          options.result || 'success',
        ]
      );
    } catch (error) {
      logger.error('Failed to persist audit log', { error: error.message, auditEntry });
    }
  }

  middleware(action) {
    return (req, res, next) => {
      const originalJson = res.json.bind(res);

      res.json = (data) => {
        this.log(action, {
          userId: req.user?.id,
          requestId: req.requestId,
          ip: req.ip,
          userAgent: req.get('user-agent'),
          targetId: req.params?.id,
          details: {
            method: req.method,
            url: req.originalUrl,
            statusCode: res.statusCode,
          },
          result: res.statusCode < 400 ? 'success' : 'failure',
        });

        return originalJson(data);
      };

      next();
    };
  }
}

module.exports = new AuditLogger();

// Usage:
// router.delete('/users/:id',
//   authenticate,
//   authorize('admin'),
//   auditLogger.middleware('USER_DELETED'),
//   userController.delete
// );
```

---

## 15.10 Grafana Dashboard สำหรับ Microservices

```json
// grafana/dashboards/microservices-overview.json (simplified)
{
  "title": "Microservices Overview",
  "panels": [
    {
      "title": "Request Rate (RPS)",
      "type": "graph",
      "targets": [
        {
          "expr": "sum(rate(http_requests_total[1m])) by (service)",
          "legendFormat": "{{ service }}"
        }
      ]
    },
    {
      "title": "Error Rate (%)",
      "type": "graph",
      "targets": [
        {
          "expr": "sum(rate(http_requests_total{status_code=~\"5..\"}[1m])) by (service) / sum(rate(http_requests_total[1m])) by (service) * 100",
          "legendFormat": "{{ service }}"
        }
      ]
    },
    {
      "title": "P95 Response Time",
      "type": "graph",
      "targets": [
        {
          "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (service, le)) * 1000",
          "legendFormat": "{{ service }} P95"
        }
      ]
    },
    {
      "title": "Active Connections",
      "type": "stat",
      "targets": [
        {
          "expr": "sum(active_connections) by (service)"
        }
      ]
    }
  ]
}
```

---

## สรุป Part 15

ในบทนี้เราได้เรียนรู้:
- **Structured Logging** - JSON logs, child loggers, multiple transports
- **Request Logging** - correlation IDs, timing, context
- **OpenTelemetry** - distributed tracing standard
- **Trace Propagation** - W3C trace context headers ระหว่าง services
- **ELK Stack** - Elasticsearch, Logstash, Kibana, Filebeat
- **Jaeger** - distributed tracing UI
- **Prometheus** - metrics collection, alerting rules
- **Grafana** - dashboards and visualization
- **Audit Logging** - security and compliance
- **AsyncLocalStorage** - implicit context propagation

**Next:** Part 16 - Testing Strategies for Microservices

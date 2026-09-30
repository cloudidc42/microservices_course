# Part 49: Distributed Tracing และ Observability

## บทนำ

Observability คือความสามารถในการเข้าใจสถานะภายในของระบบจากข้อมูลภายนอก ประกอบด้วย 3 เสาหลัก: Traces, Metrics, และ Logs ในบทนี้จะเรียนรู้การ implement OpenTelemetry SDK, Jaeger, Trace Correlation IDs, Custom Spans, Sampling Strategies, Performance Profiling และ Continuous Profiling ด้วย Pyroscope

---

## 1. โครงสร้างโปรเจกต์

```
observability/
├── src/
│   ├── telemetry/
│   │   ├── tracer.ts
│   │   ├── metrics.ts
│   │   ├── logger.ts
│   │   ├── correlation.middleware.ts
│   │   └── sampling.ts
│   ├── instrumentation/
│   │   ├── http.instrumentation.ts
│   │   ├── database.instrumentation.ts
│   │   └── redis.instrumentation.ts
│   └── profiling/
│       └── pyroscope.ts
├── docker-compose.yml
├── otel-collector-config.yaml
├── jaeger/
│   └── jaeger-config.yaml
└── k8s/
    ├── jaeger.yaml
    ├── otel-collector.yaml
    └── pyroscope.yaml
```

---

## 2. OpenTelemetry SDK Setup

### 2.1 Tracer Configuration

```typescript
// src/telemetry/tracer.ts
import { NodeSDK } from '@opentelemetry/sdk-node';
import { Resource } from '@opentelemetry/resources';
import { SemanticResourceAttributes } from '@opentelemetry/semantic-conventions';
import { BatchSpanProcessor, ConsoleSpanExporter } from '@opentelemetry/sdk-trace-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';
import { OTLPMetricExporter } from '@opentelemetry/exporter-metrics-otlp-http';
import { PeriodicExportingMetricReader } from '@opentelemetry/sdk-metrics';
import { OTLPLogExporter } from '@opentelemetry/exporter-logs-otlp-http';
import { BatchLogRecordProcessor } from '@opentelemetry/sdk-logs';
import { HttpInstrumentation } from '@opentelemetry/instrumentation-http';
import { ExpressInstrumentation } from '@opentelemetry/instrumentation-express';
import { PgInstrumentation } from '@opentelemetry/instrumentation-pg';
import { RedisInstrumentation } from '@opentelemetry/instrumentation-redis-4';
import { NestInstrumentation } from '@opentelemetry/instrumentation-nestjs-core';
import { WinstonInstrumentation } from '@opentelemetry/instrumentation-winston';
import {
  ParentBasedSampler,
  TraceIdRatioBasedSampler,
  AlwaysOnSampler,
} from '@opentelemetry/sdk-trace-node';

const SERVICE_NAME = process.env.SERVICE_NAME || 'microservice';
const SERVICE_VERSION = process.env.SERVICE_VERSION || '1.0.0';
const ENVIRONMENT = process.env.NODE_ENV || 'development';
const OTEL_ENDPOINT = process.env.OTEL_EXPORTER_OTLP_ENDPOINT || 'http://otel-collector:4318';

// Custom sampler ที่ sample ตาม business rules
class BusinessRuleSampler {
  private readonly defaultSampler: TraceIdRatioBasedSampler;

  constructor(private readonly sampleRate: number) {
    this.defaultSampler = new TraceIdRatioBasedSampler(sampleRate);
  }

  shouldSample(
    context: any,
    traceId: string,
    spanName: string,
    spanKind: any,
    attributes: any,
    links: any,
  ): any {
    // เพิ่ม sampling rate สำหรับ error traces
    if (attributes?.['http.status_code'] >= 400) {
      return new AlwaysOnSampler().shouldSample(
        context, traceId, spanName, spanKind, attributes, links,
      );
    }

    // เพิ่ม sampling rate สำหรับ slow requests (> 1 second)
    if (attributes?.['http.response_time'] > 1000) {
      return new AlwaysOnSampler().shouldSample(
        context, traceId, spanName, spanKind, attributes, links,
      );
    }

    // Health check endpoints ไม่ต้อง trace
    if (spanName.includes('/health') || spanName.includes('/metrics')) {
      return { decision: 0, attributes: {}, traceState: undefined }; // RECORD_AND_SAMPLED = 0 off
    }

    return this.defaultSampler.shouldSample(
      context, traceId, spanName, spanKind, attributes, links,
    );
  }

  toString(): string {
    return `BusinessRuleSampler(${this.sampleRate})`;
  }
}

export function initializeOpenTelemetry(): NodeSDK {
  const resource = Resource.default().merge(
    new Resource({
      [SemanticResourceAttributes.SERVICE_NAME]: SERVICE_NAME,
      [SemanticResourceAttributes.SERVICE_VERSION]: SERVICE_VERSION,
      [SemanticResourceAttributes.DEPLOYMENT_ENVIRONMENT]: ENVIRONMENT,
      [SemanticResourceAttributes.SERVICE_NAMESPACE]: 'microservices',
      'service.instance.id': process.env.HOSTNAME || 'unknown',
    }),
  );

  // Trace Exporter
  const traceExporter = new OTLPTraceExporter({
    url: `${OTEL_ENDPOINT}/v1/traces`,
    headers: {
      'X-API-Key': process.env.OTEL_API_KEY || '',
    },
  });

  // Metric Exporter
  const metricExporter = new OTLPMetricExporter({
    url: `${OTEL_ENDPOINT}/v1/metrics`,
  });

  // Log Exporter
  const logExporter = new OTLPLogExporter({
    url: `${OTEL_ENDPOINT}/v1/logs`,
  });

  // Sampler: 100% ใน dev, 10% ใน prod (แต่ sample errors ทั้งหมดเสมอ)
  const sampleRate = ENVIRONMENT === 'production' ? 0.1 : 1.0;

  const sdk = new NodeSDK({
    resource,
    
    // Span processor
    spanProcessor: new BatchSpanProcessor(traceExporter, {
      maxQueueSize: 2048,
      maxExportBatchSize: 512,
      scheduledDelayMillis: 5000,
      exportTimeoutMillis: 30000,
    }),

    // Metric reader
    metricReader: new PeriodicExportingMetricReader({
      exporter: metricExporter,
      exportIntervalMillis: 30000,
      exportTimeoutMillis: 10000,
    }),

    // Log record processor
    logRecordProcessor: new BatchLogRecordProcessor(logExporter),

    // Sampler
    sampler: new ParentBasedSampler({
      root: new BusinessRuleSampler(sampleRate) as any,
    }),

    // Auto-instrumentation
    instrumentations: [
      new HttpInstrumentation({
        ignoreIncomingRequestHook: (req) => {
          // Skip health check endpoints
          return ['/health', '/metrics', '/favicon.ico'].some((path) =>
            req.url?.includes(path),
          );
        },
        requestHook: (span, req) => {
          span.setAttribute('http.request_id', req.headers['x-request-id'] || '');
          span.setAttribute('http.correlation_id', req.headers['x-correlation-id'] || '');
        },
        responseHook: (span, res) => {
          span.setAttribute('http.response_content_length', 
            parseInt(res.getHeader('content-length') as string) || 0);
        },
      }),
      
      new ExpressInstrumentation({
        requestHook: (span, info) => {
          span.setAttribute('express.route', info.layerPath || '');
        },
      }),
      
      new NestInstrumentation(),
      
      new PgInstrumentation({
        addSqlCommenterCommentToQueries: true,
        enhancedDatabaseReporting: true,
        dbStatementSerializer: (operation, payload) => {
          // Truncate long queries
          const query = payload.text || '';
          return query.length > 500 ? query.substring(0, 500) + '...' : query;
        },
      }),
      
      new RedisInstrumentation({
        dbStatementSerializer: (cmdName, cmdArgs) => {
          // ซ่อน sensitive data เช่น passwords
          if (['AUTH', 'SET', 'SETEX'].includes(cmdName.toUpperCase())) {
            return `${cmdName} [REDACTED]`;
          }
          return `${cmdName} ${cmdArgs.join(' ')}`;
        },
      }),
      
      new WinstonInstrumentation({
        enabled: true,
        logHook: (span, record) => {
          // เพิ่ม trace context ลงใน log record
          record['trace_id'] = span.spanContext().traceId;
          record['span_id'] = span.spanContext().spanId;
        },
      }),
    ],
  });

  sdk.start();

  // Graceful shutdown
  process.on('SIGTERM', () => {
    sdk.shutdown()
      .then(() => console.log('OpenTelemetry SDK shut down successfully'))
      .catch((error) => console.error('Error shutting down OpenTelemetry SDK:', error));
  });

  return sdk;
}
```

### 2.2 Custom Spans และ Attributes

```typescript
// src/telemetry/custom-spans.ts
import {
  trace,
  context,
  SpanStatusCode,
  SpanKind,
  Attributes,
} from '@opentelemetry/api';

const tracer = trace.getTracer('microservice-tracer', '1.0.0');

// Decorator สำหรับ auto-instrument methods
export function Traced(operationName?: string, attributes?: Attributes) {
  return function (
    target: any,
    propertyKey: string,
    descriptor: PropertyDescriptor,
  ) {
    const originalMethod = descriptor.value;
    const spanName = operationName || `${target.constructor.name}.${propertyKey}`;

    descriptor.value = async function (...args: unknown[]) {
      const span = tracer.startSpan(spanName, {
        kind: SpanKind.INTERNAL,
        attributes: {
          'code.function': propertyKey,
          'code.namespace': target.constructor.name,
          ...attributes,
        },
      });

      return context.with(trace.setSpan(context.active(), span), async () => {
        try {
          const result = await originalMethod.apply(this, args);
          span.setStatus({ code: SpanStatusCode.OK });
          return result;
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
      });
    };

    return descriptor;
  };
}

// Service สำหรับสร้าง custom spans
export class SpanService {
  // สร้าง span สำหรับ business operation
  async traceBusinessOperation<T>(
    operationName: string,
    operation: (span: ReturnType<typeof tracer.startSpan>) => Promise<T>,
    attributes?: Attributes,
  ): Promise<T> {
    const span = tracer.startSpan(operationName, {
      kind: SpanKind.INTERNAL,
      attributes: {
        'operation.name': operationName,
        ...attributes,
      },
    });

    return context.with(trace.setSpan(context.active(), span), async () => {
      try {
        const result = await operation(span);
        span.setStatus({ code: SpanStatusCode.OK });
        return result;
      } catch (error) {
        const err = error as Error;
        span.recordException(err);
        span.setStatus({
          code: SpanStatusCode.ERROR,
          message: err.message,
        });
        span.setAttribute('error.type', err.constructor.name);
        span.setAttribute('error.message', err.message);
        throw error;
      } finally {
        span.end();
      }
    });
  }

  // สร้าง span สำหรับ external API call
  async traceExternalCall<T>(
    serviceName: string,
    url: string,
    method: string,
    operation: () => Promise<T>,
  ): Promise<T> {
    const span = tracer.startSpan(`${method} ${new URL(url).pathname}`, {
      kind: SpanKind.CLIENT,
      attributes: {
        'http.method': method,
        'http.url': url,
        'peer.service': serviceName,
      },
    });

    const startTime = Date.now();

    return context.with(trace.setSpan(context.active(), span), async () => {
      try {
        const result = await operation();
        span.setAttribute('http.status_code', 200);
        span.setAttribute('http.duration_ms', Date.now() - startTime);
        span.setStatus({ code: SpanStatusCode.OK });
        return result;
      } catch (error) {
        const err = error as any;
        span.setAttribute('http.status_code', err.response?.status || 0);
        span.setAttribute('http.duration_ms', Date.now() - startTime);
        span.recordException(err);
        span.setStatus({ code: SpanStatusCode.ERROR, message: err.message });
        throw error;
      } finally {
        span.end();
      }
    });
  }

  // สร้าง span สำหรับ database query
  async traceDatabaseQuery<T>(
    dbSystem: string,
    dbOperation: string,
    dbTable: string,
    query: () => Promise<T>,
  ): Promise<T> {
    const span = tracer.startSpan(`${dbOperation} ${dbTable}`, {
      kind: SpanKind.CLIENT,
      attributes: {
        'db.system': dbSystem,
        'db.operation': dbOperation,
        'db.sql.table': dbTable,
      },
    });

    const startTime = Date.now();

    return context.with(trace.setSpan(context.active(), span), async () => {
      try {
        const result = await query();
        const duration = Date.now() - startTime;
        span.setAttribute('db.query_duration_ms', duration);
        
        // Flag slow queries
        if (duration > 1000) {
          span.setAttribute('db.slow_query', true);
          span.setAttribute('db.query_threshold_ms', 1000);
        }
        
        span.setStatus({ code: SpanStatusCode.OK });
        return result;
      } catch (error) {
        span.recordException(error as Error);
        span.setStatus({ code: SpanStatusCode.ERROR });
        throw error;
      } finally {
        span.end();
      }
    });
  }
}
```

---

## 3. Metrics Collection

```typescript
// src/telemetry/metrics.ts
import {
  metrics,
  Counter,
  Histogram,
  ObservableGauge,
  Meter,
  UpDownCounter,
} from '@opentelemetry/api';

const meter: Meter = metrics.getMeter('microservice-metrics', '1.0.0');

// HTTP Metrics
export const httpRequestsTotal: Counter = meter.createCounter(
  'http_requests_total',
  {
    description: 'Total number of HTTP requests',
    unit: '{requests}',
  },
);

export const httpRequestDuration: Histogram = meter.createHistogram(
  'http_request_duration_seconds',
  {
    description: 'HTTP request duration in seconds',
    unit: 's',
    advice: {
      explicitBucketBoundaries: [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10],
    },
  },
);

export const httpRequestSize: Histogram = meter.createHistogram(
  'http_request_size_bytes',
  {
    description: 'HTTP request size in bytes',
    unit: 'By',
    advice: {
      explicitBucketBoundaries: [100, 1000, 10000, 100000, 1000000],
    },
  },
);

// Business Metrics
export const ordersCreatedTotal: Counter = meter.createCounter(
  'orders_created_total',
  {
    description: 'Total number of orders created',
    unit: '{orders}',
  },
);

export const orderAmountHistogram: Histogram = meter.createHistogram(
  'order_amount_baht',
  {
    description: 'Order amount distribution in Baht',
    unit: 'THB',
    advice: {
      explicitBucketBoundaries: [100, 500, 1000, 5000, 10000, 50000, 100000],
    },
  },
);

export const activeUsers: UpDownCounter = meter.createUpDownCounter(
  'active_users_current',
  {
    description: 'Current number of active users',
    unit: '{users}',
  },
);

// Database Metrics
export const dbConnectionPoolSize: ObservableGauge = meter.createObservableGauge(
  'db_connection_pool_size',
  {
    description: 'Database connection pool size',
    unit: '{connections}',
  },
);

export const dbQueryDuration: Histogram = meter.createHistogram(
  'db_query_duration_seconds',
  {
    description: 'Database query duration',
    unit: 's',
    advice: {
      explicitBucketBoundaries: [0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1, 5],
    },
  },
);

// Cache Metrics
export const cacheHitsTotal: Counter = meter.createCounter(
  'cache_hits_total',
  {
    description: 'Total cache hits',
  },
);

export const cacheMissesTotal: Counter = meter.createCounter(
  'cache_misses_total',
  {
    description: 'Total cache misses',
  },
);

// Message Queue Metrics
export const messagesPublishedTotal: Counter = meter.createCounter(
  'messages_published_total',
  {
    description: 'Total messages published to queue',
  },
);

export const messagesConsumedTotal: Counter = meter.createCounter(
  'messages_consumed_total',
  {
    description: 'Total messages consumed from queue',
  },
);

export const messageProcessingDuration: Histogram = meter.createHistogram(
  'message_processing_duration_seconds',
  {
    description: 'Message processing duration',
    unit: 's',
  },
);

// Metrics Middleware
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';

@Injectable()
export class MetricsMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction): void {
    const startTime = Date.now();

    res.on('finish', () => {
      const duration = (Date.now() - startTime) / 1000;
      const route = req.route?.path || req.path;
      const method = req.method;
      const statusCode = res.statusCode.toString();

      httpRequestsTotal.add(1, {
        method,
        route,
        status_code: statusCode,
        service: process.env.SERVICE_NAME || 'unknown',
      });

      httpRequestDuration.record(duration, {
        method,
        route,
        status_code: statusCode,
      });

      const contentLength = parseInt(req.headers['content-length'] || '0');
      if (contentLength > 0) {
        httpRequestSize.record(contentLength, {
          method,
          route,
        });
      }
    });

    next();
  }
}
```

---

## 4. Structured Logging

```typescript
// src/telemetry/logger.ts
import * as winston from 'winston';
import { trace, context } from '@opentelemetry/api';
import { LoggerService } from '@nestjs/common';

interface LogContext {
  service: string;
  version: string;
  environment: string;
  traceId?: string;
  spanId?: string;
  userId?: string;
  requestId?: string;
  [key: string]: unknown;
}

// Custom format สำหรับเพิ่ม trace context
const traceContextFormat = winston.format((info) => {
  const span = trace.getActiveSpan();
  
  if (span) {
    const spanContext = span.spanContext();
    info.trace_id = spanContext.traceId;
    info.span_id = spanContext.spanId;
    info.trace_flags = spanContext.traceFlags;
  }
  
  return info;
});

// Custom format สำหรับ ECS (Elastic Common Schema)
const ecsFormat = winston.format((info) => {
  const { level, message, timestamp, ...rest } = info;
  
  return {
    '@timestamp': timestamp,
    'log.level': level,
    message,
    ecs: { version: '8.0.0' },
    service: {
      name: process.env.SERVICE_NAME,
      version: process.env.SERVICE_VERSION,
      environment: process.env.NODE_ENV,
    },
    ...rest,
  };
});

export const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    traceContextFormat(),
    ecsFormat(),
    winston.format.json(),
  ),
  defaultMeta: {
    service: process.env.SERVICE_NAME || 'microservice',
    version: process.env.SERVICE_VERSION || '1.0.0',
    environment: process.env.NODE_ENV || 'development',
  },
  transports: [
    new winston.transports.Console({
      silent: process.env.NODE_ENV === 'test',
    }),
  ],
});

// เพิ่ม file transport ใน production
if (process.env.NODE_ENV === 'production') {
  logger.add(
    new winston.transports.File({
      filename: '/var/log/app/error.log',
      level: 'error',
      maxsize: 100 * 1024 * 1024, // 100MB
      maxFiles: 10,
      tailable: true,
    }),
  );
}

// NestJS Logger Adapter
export class AppLogger implements LoggerService {
  private context?: string;

  constructor(context?: string) {
    this.context = context;
  }

  log(message: unknown, context?: string): void {
    logger.info(this.formatMessage(message), {
      context: context || this.context,
    });
  }

  error(message: unknown, trace?: string, context?: string): void {
    logger.error(this.formatMessage(message), {
      context: context || this.context,
      stack_trace: trace,
    });
  }

  warn(message: unknown, context?: string): void {
    logger.warn(this.formatMessage(message), {
      context: context || this.context,
    });
  }

  debug(message: unknown, context?: string): void {
    logger.debug(this.formatMessage(message), {
      context: context || this.context,
    });
  }

  verbose(message: unknown, context?: string): void {
    logger.verbose(this.formatMessage(message), {
      context: context || this.context,
    });
  }

  private formatMessage(message: unknown): string {
    if (typeof message === 'string') return message;
    if (message instanceof Error) return message.message;
    return JSON.stringify(message);
  }

  // Log method สำหรับ business events
  logBusinessEvent(
    event: string,
    data: Record<string, unknown>,
    userId?: string,
  ): void {
    logger.info(event, {
      event_type: 'business_event',
      event_name: event,
      user_id: userId,
      ...data,
    });
  }

  // Log สำหรับ security events
  logSecurityEvent(
    event: string,
    severity: 'low' | 'medium' | 'high' | 'critical',
    data: Record<string, unknown>,
  ): void {
    const level = severity === 'critical' || severity === 'high' ? 'error' : 'warn';
    
    logger.log(level, event, {
      event_type: 'security_event',
      event_name: event,
      severity,
      ...data,
    });
  }
}
```

---

## 5. Correlation ID Middleware

```typescript
// src/telemetry/correlation.middleware.ts
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';
import { v4 as uuidv4 } from 'uuid';
import { context, trace, propagation } from '@opentelemetry/api';
import { W3CTraceContextPropagator } from '@opentelemetry/core';

declare global {
  namespace Express {
    interface Request {
      requestId: string;
      correlationId: string;
      userId?: string;
    }
  }
}

@Injectable()
export class CorrelationMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction): void {
    // ดึง Request ID จาก header หรือสร้างใหม่
    const requestId = (req.headers['x-request-id'] as string) || uuidv4();
    
    // ดึง Correlation ID (สำหรับ tracking ข้าม services)
    const correlationId =
      (req.headers['x-correlation-id'] as string) ||
      requestId;

    req.requestId = requestId;
    req.correlationId = correlationId;

    // Set headers ใน response
    res.setHeader('X-Request-ID', requestId);
    res.setHeader('X-Correlation-ID', correlationId);

    // Extract trace context จาก W3C headers (traceparent, tracestate)
    const carrier: Record<string, string> = {};
    for (const [key, value] of Object.entries(req.headers)) {
      if (typeof value === 'string') {
        carrier[key] = value;
      }
    }

    const propagator = new W3CTraceContextPropagator();
    const extractedContext = propagator.extract(
      context.active(),
      carrier,
      {
        get: (carrier, key) => carrier[key],
        keys: (carrier) => Object.keys(carrier),
      },
    );

    // เพิ่ม request/correlation IDs เข้า active span
    const span = trace.getSpan(extractedContext);
    if (span) {
      span.setAttribute('request.id', requestId);
      span.setAttribute('correlation.id', correlationId);
    }

    // เก็บ context ใน async local storage
    context.with(extractedContext, () => {
      next();
    });
  }
}
```

---

## 6. Jaeger Deployment

```yaml
# docker-compose.yml
version: '3.9'

services:
  # Jaeger All-in-One สำหรับ development
  jaeger:
    image: jaegertracing/all-in-one:1.52
    environment:
      COLLECTOR_OTLP_ENABLED: "true"
      SPAN_STORAGE_TYPE: elasticsearch
      ES_SERVER_URLS: http://elasticsearch:9200
      ES_INDEX_PREFIX: jaeger
    ports:
      - "16686:16686"   # Jaeger UI
      - "14268:14268"   # HTTP Thrift collector
      - "14250:14250"   # gRPC model collector  
      - "6831:6831/udp" # UDP agent (Jaeger Thrift compact)
      - "4317:4317"     # OTLP gRPC
      - "4318:4318"     # OTLP HTTP
    networks:
      - observability
    depends_on:
      elasticsearch:
        condition: service_healthy

  # OpenTelemetry Collector
  otel-collector:
    image: otel/opentelemetry-collector-contrib:0.91.0
    command: ["--config=/etc/otel-collector-config.yaml"]
    volumes:
      - ./otel-collector-config.yaml:/etc/otel-collector-config.yaml:ro
    ports:
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP
      - "8888:8888"   # Prometheus metrics (self)
      - "8889:8889"   # Prometheus exporter
      - "13133:13133" # Health check
    networks:
      - observability
    depends_on:
      - jaeger

  # Elasticsearch สำหรับเก็บ traces
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    environment:
      discovery.type: single-node
      xpack.security.enabled: "false"
      ES_JAVA_OPTS: "-Xms512m -Xmx512m"
    volumes:
      - elasticsearch_data:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"
    networks:
      - observability
    healthcheck:
      test: ["CMD-SHELL", "curl -sf http://localhost:9200/_cluster/health | jq -r '.status' | grep -v red"]
      interval: 30s
      timeout: 10s
      retries: 5

  # Prometheus สำหรับ metrics
  prometheus:
    image: prom/prometheus:v2.48.0
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'
      - '--web.enable-lifecycle'
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    ports:
      - "9090:9090"
    networks:
      - observability

  # Grafana สำหรับ visualization
  grafana:
    image: grafana/grafana:10.2.0
    environment:
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASSWORD:-admin}
      GF_USERS_ALLOW_SIGN_UP: "false"
      GF_INSTALL_PLUGINS: grafana-clock-panel,grafana-piechart-panel
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/dashboards:/etc/grafana/provisioning/dashboards:ro
      - ./grafana/datasources:/etc/grafana/provisioning/datasources:ro
    ports:
      - "3000:3000"
    networks:
      - observability
    depends_on:
      - prometheus

  # Pyroscope สำหรับ continuous profiling
  pyroscope:
    image: grafana/pyroscope:1.3.0
    ports:
      - "4040:4040"
    volumes:
      - pyroscope_data:/data
    networks:
      - observability

networks:
  observability:
    driver: bridge

volumes:
  elasticsearch_data:
  prometheus_data:
  grafana_data:
  pyroscope_data:
```

---

## 7. OpenTelemetry Collector Config

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318
        cors:
          allowed_origins:
            - "http://localhost:*"
            - "https://*.myapp.com"

  # Prometheus metrics receiver (scrape own services)
  prometheus:
    config:
      scrape_configs:
        - job_name: 'microservices'
          scrape_interval: 15s
          static_configs:
            - targets:
                - 'user-service:3001'
                - 'order-service:3002'
                - 'payment-service:3003'
          metrics_path: '/metrics'

processors:
  # กรอง attributes ที่ sensitive
  attributes:
    actions:
      - key: db.statement
        action: hash
      - key: http.request.header.authorization
        action: delete
      - key: http.request.header.x-api-key
        action: delete

  # Batch processor
  batch:
    timeout: 5s
    send_batch_size: 1000
    send_batch_max_size: 2000

  # Memory limiter
  memory_limiter:
    check_interval: 5s
    limit_mib: 512
    spike_limit_mib: 128

  # Resource detection
  resourcedetection:
    detectors: [env, system, docker]
    timeout: 5s

  # Filter out health check traces
  filter:
    traces:
      span:
        - 'attributes["http.route"] == "/health"'
        - 'attributes["http.route"] == "/health/live"'
        - 'attributes["http.route"] == "/health/ready"'
        - 'attributes["http.route"] == "/metrics"'

  # Tail-based sampling (sample ตาม criteria)
  tail_sampling:
    decision_wait: 10s
    num_traces: 100
    expected_new_traces_per_sec: 10
    policies:
      # เก็บ error traces ทั้งหมด
      - name: errors-policy
        type: status_code
        status_code:
          status_codes: [ERROR]
      
      # เก็บ slow traces (> 2 seconds)
      - name: slow-traces-policy
        type: latency
        latency:
          threshold_ms: 2000
      
      # Sample 10% ของ traces ปกติ
      - name: probabilistic-policy
        type: probabilistic
        probabilistic:
          sampling_percentage: 10

exporters:
  # ส่งไปยัง Jaeger
  jaeger:
    endpoint: jaeger:14250
    tls:
      insecure: true

  # ส่งไปยัง Prometheus
  prometheus:
    endpoint: "0.0.0.0:8889"
    namespace: otel

  # ส่งไปยัง Elasticsearch (logs)
  elasticsearch:
    endpoint: http://elasticsearch:9200
    index: otel-logs
    mapping:
      mode: ecs

  # Debug (สำหรับ development)
  debug:
    verbosity: basic

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, resourcedetection, attributes, filter, tail_sampling, batch]
      exporters: [jaeger]
    
    metrics:
      receivers: [otlp, prometheus]
      processors: [memory_limiter, batch]
      exporters: [prometheus]
    
    logs:
      receivers: [otlp]
      processors: [memory_limiter, attributes, batch]
      exporters: [elasticsearch]

  telemetry:
    logs:
      level: warn
    metrics:
      address: 0.0.0.0:8888
```

---

## 8. Continuous Profiling ด้วย Pyroscope

```typescript
// src/profiling/pyroscope.ts
import Pyroscope from '@pyroscope/nodejs';

export function initializePyroscope(): void {
  const enabled = process.env.PYROSCOPE_ENABLED === 'true';
  
  if (!enabled) {
    console.log('Pyroscope profiling disabled');
    return;
  }

  Pyroscope.init({
    serverAddress: process.env.PYROSCOPE_SERVER_ADDRESS || 'http://pyroscope:4040',
    appName: process.env.SERVICE_NAME || 'microservice',
    
    tags: {
      environment: process.env.NODE_ENV || 'development',
      version: process.env.SERVICE_VERSION || '1.0.0',
      region: process.env.REGION || 'ap-southeast-1',
      pod: process.env.HOSTNAME || 'unknown',
    },
    
    // Configuration
    basicAuthUser: process.env.PYROSCOPE_AUTH_USER,
    basicAuthPassword: process.env.PYROSCOPE_AUTH_PASSWORD,
    
    // Profiling configuration
    wall: {
      collectCpuTime: true,
    },
  });

  Pyroscope.start();
  
  console.log('Pyroscope profiling started');

  process.on('SIGTERM', () => {
    Pyroscope.stop();
  });
}

// Custom labeling สำหรับ profiling ตาม request context
export function withProfilingLabels<T>(
  labels: Record<string, string>,
  fn: () => T,
): T {
  return Pyroscope.wrapWithLabels(labels, fn);
}

// Middleware สำหรับ label profiling ตาม route
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';

@Injectable()
export class ProfilingMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction): void {
    const route = req.route?.path || req.path;
    const method = req.method;

    withProfilingLabels(
      {
        route: route.replace(/[/:.]/g, '_'),
        method: method.toLowerCase(),
      },
      () => next(),
    );
  }
}
```

---

## 9. Performance Profiling

```typescript
// src/profiling/performance-monitor.ts
import { performance, PerformanceObserver } from 'perf_hooks';
import { trace, SpanStatusCode } from '@opentelemetry/api';
import { dbQueryDuration } from '../telemetry/metrics';

export class PerformanceMonitor {
  private readonly slowQueryThreshold: number;
  private readonly observer: PerformanceObserver;

  constructor(slowQueryThresholdMs: number = 1000) {
    this.slowQueryThreshold = slowQueryThresholdMs;
    this.observer = this.setupObserver();
  }

  private setupObserver(): PerformanceObserver {
    const observer = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        this.processEntry(entry);
      }
    });

    observer.observe({ entryTypes: ['measure', 'mark'] });
    return observer;
  }

  private processEntry(entry: PerformanceEntry): void {
    const tracer = trace.getActiveSpan();
    
    if (entry.entryType === 'measure') {
      // Record metric
      dbQueryDuration.record(entry.duration / 1000, {
        operation: entry.name,
      });

      // Log slow operations
      if (entry.duration > this.slowQueryThreshold) {
        console.warn(`Slow operation detected: ${entry.name} took ${entry.duration}ms`);
        
        tracer?.setAttribute('performance.slow_operation', true);
        tracer?.setAttribute('performance.operation_name', entry.name);
        tracer?.setAttribute('performance.duration_ms', entry.duration);
      }
    }
  }

  // Measure function execution time
  async measure<T>(name: string, fn: () => Promise<T>): Promise<T> {
    const markStart = `${name}-start`;
    const markEnd = `${name}-end`;
    
    performance.mark(markStart);
    
    try {
      const result = await fn();
      performance.mark(markEnd);
      performance.measure(name, markStart, markEnd);
      return result;
    } catch (error) {
      performance.mark(markEnd);
      performance.measure(`${name}-error`, markStart, markEnd);
      throw error;
    } finally {
      // Cleanup marks
      performance.clearMarks(markStart);
      performance.clearMarks(markEnd);
    }
  }

  // Generate flame graph data
  async captureHeapSnapshot(): Promise<string> {
    const v8 = require('v8');
    const stream = v8.writeHeapSnapshot();
    return stream;
  }

  // Memory usage report
  getMemoryUsage(): {
    heapUsed: number;
    heapTotal: number;
    external: number;
    rss: number;
    heapUsedPercent: number;
  } {
    const memUsage = process.memoryUsage();
    
    return {
      heapUsed: Math.round(memUsage.heapUsed / 1024 / 1024),
      heapTotal: Math.round(memUsage.heapTotal / 1024 / 1024),
      external: Math.round(memUsage.external / 1024 / 1024),
      rss: Math.round(memUsage.rss / 1024 / 1024),
      heapUsedPercent: Math.round(
        (memUsage.heapUsed / memUsage.heapTotal) * 100,
      ),
    };
  }

  destroy(): void {
    this.observer.disconnect();
  }
}
```

---

## 10. Kubernetes Observability Stack

```yaml
# k8s/otel-collector.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: otel-collector
  namespace: monitoring
spec:
  replicas: 2
  selector:
    matchLabels:
      app: otel-collector
  template:
    metadata:
      labels:
        app: otel-collector
    spec:
      containers:
        - name: otel-collector
          image: otel/opentelemetry-collector-contrib:0.91.0
          args:
            - "--config=/conf/otel-collector-config.yaml"
          ports:
            - containerPort: 4317
              name: otlp-grpc
            - containerPort: 4318
              name: otlp-http
            - containerPort: 8888
              name: metrics
            - containerPort: 13133
              name: health
          volumeMounts:
            - name: otel-collector-config
              mountPath: /conf
          livenessProbe:
            httpGet:
              path: /
              port: 13133
            initialDelaySeconds: 5
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /
              port: 13133
            initialDelaySeconds: 5
            periodSeconds: 10
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "512Mi"
              cpu: "500m"
      volumes:
        - name: otel-collector-config
          configMap:
            name: otel-collector-config

---
apiVersion: v1
kind: Service
metadata:
  name: otel-collector
  namespace: monitoring
spec:
  selector:
    app: otel-collector
  ports:
    - name: otlp-grpc
      port: 4317
      targetPort: 4317
    - name: otlp-http
      port: 4318
      targetPort: 4318
    - name: metrics
      port: 8888
      targetPort: 8888

---
# Jaeger deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jaeger
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jaeger
  template:
    metadata:
      labels:
        app: jaeger
    spec:
      containers:
        - name: jaeger
          image: jaegertracing/all-in-one:1.52
          env:
            - name: SPAN_STORAGE_TYPE
              value: elasticsearch
            - name: ES_SERVER_URLS
              value: http://elasticsearch:9200
            - name: COLLECTOR_OTLP_ENABLED
              value: "true"
          ports:
            - containerPort: 16686
              name: ui
            - containerPort: 4317
              name: otlp-grpc
            - containerPort: 4318
              name: otlp-http
          resources:
            requests:
              memory: "256Mi"
              cpu: "200m"
            limits:
              memory: "1Gi"
              cpu: "1000m"
```

---

## 11. Grafana Dashboard Configuration

```json
// grafana/dashboards/microservices-overview.json
{
  "title": "Microservices Overview",
  "panels": [
    {
      "title": "Request Rate",
      "type": "stat",
      "targets": [
        {
          "expr": "sum(rate(http_requests_total[5m])) by (service)",
          "legendFormat": "{{service}}"
        }
      ]
    },
    {
      "title": "Error Rate",
      "type": "timeseries",
      "targets": [
        {
          "expr": "sum(rate(http_requests_total{status_code=~'5..'}[5m])) by (service) / sum(rate(http_requests_total[5m])) by (service) * 100",
          "legendFormat": "{{service}} error %"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "percent",
          "thresholds": {
            "steps": [
              {"value": 0, "color": "green"},
              {"value": 1, "color": "yellow"},
              {"value": 5, "color": "red"}
            ]
          }
        }
      }
    },
    {
      "title": "P99 Latency",
      "type": "timeseries",
      "targets": [
        {
          "expr": "histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (service, le))",
          "legendFormat": "{{service}} p99"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "s"
        }
      }
    },
    {
      "title": "Active Database Connections",
      "type": "gauge",
      "targets": [
        {
          "expr": "db_connection_pool_size",
          "legendFormat": "{{service}}"
        }
      ]
    }
  ]
}
```

---

## สรุป

| หัวข้อ | เทคโนโลยี | วัตถุประสงค์ |
|--------|-----------|-------------|
| Distributed Tracing | OpenTelemetry + Jaeger | ติดตาม request ข้าม services |
| Auto-Instrumentation | OTel instrumentations | Trace HTTP, DB, Redis โดยอัตโนมัติ |
| Custom Spans | OTel Tracer API | เพิ่ม business context ใน traces |
| Metrics | OTel Metrics + Prometheus | วัด performance และ business KPIs |
| Structured Logs | Winston + ECS format | Logs ที่ searchable และ correlatable |
| Correlation IDs | W3C TraceContext propagation | เชื่อม traces, metrics, logs เข้าด้วยกัน |
| Sampling | Tail-based + Rate-based | ลด cost โดยยังเก็บ important traces |
| Continuous Profiling | Pyroscope | หา CPU/memory hotspots แบบ real-time |
| OTel Collector | otel-collector-contrib | Pipeline สำหรับ process และ route telemetry |
| Visualization | Grafana + Jaeger UI | Dashboard สำหรับ SRE team |

> **Best Practice**: ใช้ Correlation ID เชื่อม Traces, Metrics, Logs เข้าด้วยกัน และตั้ง Alert บน SLO ไม่ใช่ SLI แต่ละตัว

# Part 49: Distributed Tracing ขั้นสูง — OpenTelemetry, Jaeger, และ Continuous Profiling

ในบทนี้เราจะเรียนรู้การ implement Distributed Tracing อย่างครบวงจรโดยใช้ OpenTelemetry SDK, การ deploy Jaeger บน Kubernetes, การสร้าง Custom Spans, Context Propagation ข้าม services, Sampling Strategies, และการทำ Continuous Profiling ด้วย Pyroscope

---

## 1. OpenTelemetry SDK Setup สำหรับ Node.js

OpenTelemetry เป็น CNCF standard สำหรับ observability ที่รวม traces, metrics, และ logs เข้าด้วยกัน

### 1.1 การติดตั้ง Dependencies

```bash
npm install @opentelemetry/sdk-node \
  @opentelemetry/auto-instrumentations-node \
  @opentelemetry/exporter-trace-otlp-http \
  @opentelemetry/exporter-metrics-otlp-http \
  @opentelemetry/exporter-logs-otlp-http \
  @opentelemetry/resources \
  @opentelemetry/semantic-conventions \
  @opentelemetry/sdk-metrics \
  @opentelemetry/sdk-logs \
  @opentelemetry/api \
  @opentelemetry/api-logs
```

### 1.2 Instrumentation Setup (ต้องโหลดก่อน code อื่นทุกชนิด)

```typescript
// src/instrumentation.ts
import { NodeSDK } from '@opentelemetry/sdk-node';
import { Resource } from '@opentelemetry/resources';
import {
  SEMRESATTRS_SERVICE_NAME,
  SEMRESATTRS_SERVICE_VERSION,
  SEMRESATTRS_DEPLOYMENT_ENVIRONMENT,
  SEMRESATTRS_SERVICE_NAMESPACE,
  SEMRESATTRS_HOST_NAME,
  SEMRESATTRS_CONTAINER_ID,
} from '@opentelemetry/semantic-conventions';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';
import { OTLPMetricExporter } from '@opentelemetry/exporter-metrics-otlp-http';
import { OTLPLogExporter } from '@opentelemetry/exporter-logs-otlp-http';
import { PeriodicExportingMetricReader } from '@opentelemetry/sdk-metrics';
import { BatchLogRecordProcessor, LoggerProvider } from '@opentelemetry/sdk-logs';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import {
  BatchSpanProcessor,
  ParentBasedSampler,
  TraceIdRatioBasedSampler,
} from '@opentelemetry/sdk-trace-base';
import os from 'os';
import fs from 'fs';

// Read container ID for Kubernetes pod tracking
function getContainerId(): string {
  try {
    const cgroup = fs.readFileSync('/proc/self/cgroup', 'utf8');
    const match = cgroup.match(/\/([a-f0-9]{64})/);
    return match?.[1] ?? '';
  } catch {
    return '';
  }
}

// Create resource with service metadata
const resource = Resource.default().merge(
  new Resource({
    [SEMRESATTRS_SERVICE_NAME]: process.env.SERVICE_NAME || 'unknown-service',
    [SEMRESATTRS_SERVICE_VERSION]: process.env.SERVICE_VERSION || '0.0.0',
    [SEMRESATTRS_DEPLOYMENT_ENVIRONMENT]: process.env.NODE_ENV || 'development',
    [SEMRESATTRS_SERVICE_NAMESPACE]: process.env.SERVICE_NAMESPACE || 'default',
    [SEMRESATTRS_HOST_NAME]: os.hostname(),
    [SEMRESATTRS_CONTAINER_ID]: getContainerId(),
    'k8s.pod.name': process.env.POD_NAME || '',
    'k8s.namespace.name': process.env.POD_NAMESPACE || '',
    'k8s.node.name': process.env.NODE_NAME || '',
  })
);

// Configure OTLP exporters
const traceExporter = new OTLPTraceExporter({
  url: `${process.env.OTEL_COLLECTOR_URL || 'http://localhost:4318'}/v1/traces`,
  headers: {
    'x-honeycomb-team': process.env.HONEYCOMB_API_KEY || '',
  },
});

const metricExporter = new OTLPMetricExporter({
  url: `${process.env.OTEL_COLLECTOR_URL || 'http://localhost:4318'}/v1/metrics`,
});

const logExporter = new OTLPLogExporter({
  url: `${process.env.OTEL_COLLECTOR_URL || 'http://localhost:4318'}/v1/logs`,
});

// Configure sampling strategy
// In production: sample 10% of requests, but keep all errors
const sampler = new ParentBasedSampler({
  root: new TraceIdRatioBasedSampler(
    parseFloat(process.env.TRACE_SAMPLE_RATE || '0.1')
  ),
});

// Initialize SDK
const sdk = new NodeSDK({
  resource,
  sampler,
  spanProcessors: [
    new BatchSpanProcessor(traceExporter, {
      maxQueueSize: 2048,
      maxExportBatchSize: 512,
      scheduledDelayMillis: 5000,
      exportTimeoutMillis: 30000,
    }),
  ],
  metricReader: new PeriodicExportingMetricReader({
    exporter: metricExporter,
    exportIntervalMillis: 10000,
    exportTimeoutMillis: 30000,
  }),
  logRecordProcessors: [
    new BatchLogRecordProcessor(logExporter),
  ],
  instrumentations: [
    getNodeAutoInstrumentations({
      '@opentelemetry/instrumentation-http': {
        ignoreIncomingRequestHook: (req) => {
          // Don't trace health checks and metrics endpoints
          return ['/health', '/metrics', '/ready'].includes(req.url || '');
        },
        requestHook: (span, req) => {
          span.setAttribute('http.request.body.size',
            req.headers['content-length'] || 0
          );
        },
        responseHook: (span, res) => {
          span.setAttribute('http.response.body.size',
            res.getHeader('content-length') || 0
          );
        },
      },
      '@opentelemetry/instrumentation-express': {
        ignoreLayers: ['expressInit', 'query'],
      },
      '@opentelemetry/instrumentation-pg': {
        enhancedDatabaseReporting: true,
      },
      '@opentelemetry/instrumentation-redis': {
        dbStatementSerializer: (cmdName, cmdArgs) => {
          // Don't log sensitive data
          if (['AUTH', 'SET', 'SETEX'].includes(cmdName)) {
            return `${cmdName} [REDACTED]`;
          }
          return `${cmdName} ${cmdArgs.join(' ')}`;
        },
      },
    }),
  ],
});

// Start SDK
sdk.start();
console.log('OpenTelemetry SDK started');

// Graceful shutdown
process.on('SIGTERM', () => {
  sdk.shutdown()
    .then(() => console.log('OpenTelemetry SDK shut down'))
    .catch(err => console.error('Error shutting down OpenTelemetry SDK', err))
    .finally(() => process.exit(0));
});

export { sdk };
```

### 1.3 Custom Metrics Setup

```typescript
// src/telemetry/metrics.ts
import {
  metrics,
  Counter,
  Histogram,
  ObservableGauge,
  ValueType,
} from '@opentelemetry/api';

const meter = metrics.getMeter('order-service', '1.0.0');

// HTTP request metrics
export const httpRequestDuration = meter.createHistogram(
  'http_request_duration_ms',
  {
    description: 'Duration of HTTP requests in milliseconds',
    unit: 'ms',
    valueType: ValueType.DOUBLE,
    advice: {
      explicitBucketBoundaries: [5, 10, 25, 50, 75, 100, 250, 500, 750, 1000, 2500, 5000],
    },
  }
);

export const httpRequestsTotal = meter.createCounter(
  'http_requests_total',
  {
    description: 'Total number of HTTP requests',
    valueType: ValueType.INT,
  }
);

// Business metrics
export const ordersCreated = meter.createCounter(
  'orders_created_total',
  {
    description: 'Total number of orders created',
    valueType: ValueType.INT,
  }
);

export const orderAmount = meter.createHistogram(
  'order_amount_usd',
  {
    description: 'Order amount in USD',
    unit: 'USD',
    valueType: ValueType.DOUBLE,
    advice: {
      explicitBucketBoundaries: [10, 25, 50, 100, 250, 500, 1000, 5000],
    },
  }
);

// Queue depth gauge
let currentQueueDepth = 0;
const queueDepthGauge = meter.createObservableGauge(
  'message_queue_depth',
  {
    description: 'Current message queue depth',
    valueType: ValueType.INT,
  }
);

queueDepthGauge.addCallback((result) => {
  result.observe(currentQueueDepth, {
    'queue.name': 'order-processing',
  });
});

export function updateQueueDepth(depth: number) {
  currentQueueDepth = depth;
}

// Database connection pool metrics
export const dbPoolActive = meter.createObservableGauge(
  'db_connection_pool_active',
  { description: 'Active database connections' }
);

export const dbPoolIdle = meter.createObservableGauge(
  'db_connection_pool_idle',
  { description: 'Idle database connections' }
);

// Cache metrics
export const cacheHits = meter.createCounter(
  'cache_hits_total',
  { description: 'Total cache hits' }
);

export const cacheMisses = meter.createCounter(
  'cache_misses_total',
  { description: 'Total cache misses' }
);
```

---

## 2. Jaeger Deployment on Kubernetes

### 2.1 Jaeger Operator Installation

```yaml
# kubernetes/jaeger/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: observability
  labels:
    app.kubernetes.io/managed-by: helm

---
# kubernetes/jaeger/operator-install.yaml
# First install the Jaeger Operator via Helm
# helm repo add jaegertracing https://jaegertracing.github.io/helm-charts
# helm install jaeger-operator jaegertracing/jaeger-operator -n observability
```

### 2.2 Jaeger Production Instance

```yaml
# kubernetes/jaeger/jaeger-production.yaml
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: jaeger-production
  namespace: observability
spec:
  strategy: production
  
  collector:
    maxReplicas: 5
    resources:
      limits:
        cpu: 500m
        memory: 512Mi
      requests:
        cpu: 100m
        memory: 128Mi
    autoscale: true
    options:
      collector:
        queue-size: 2000
        num-workers: 50
  
  query:
    replicas: 2
    resources:
      limits:
        cpu: 500m
        memory: 512Mi
      requests:
        cpu: 100m
        memory: 128Mi
    options:
      query:
        base-path: /jaeger
  
  storage:
    type: elasticsearch
    elasticsearch:
      nodeCount: 3
      resources:
        limits:
          cpu: "1"
          memory: 2Gi
        requests:
          cpu: 200m
          memory: 1Gi
      storage:
        size: 50Gi
        storageClassName: fast-ssd
      redundancyPolicy: SingleRedundancy
    options:
      es:
        server-urls: https://elasticsearch.observability:9200
        index-prefix: jaeger-prod
        num-shards: 5
        num-replicas: 1
        version: 7
  
  ingress:
    enabled: true
    annotations:
      kubernetes.io/ingress.class: nginx
      cert-manager.io/cluster-issuer: letsencrypt-prod
    hosts:
      - jaeger.internal.example.com
    tls:
      - hosts:
          - jaeger.internal.example.com
        secretName: jaeger-tls

---
# OpenTelemetry Collector for receiving traces
apiVersion: apps/v1
kind: Deployment
metadata:
  name: otel-collector
  namespace: observability
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
          image: otel/opentelemetry-collector-contrib:0.90.0
          args: ['--config=/etc/otel/config.yaml']
          ports:
            - containerPort: 4317  # OTLP gRPC
            - containerPort: 4318  # OTLP HTTP
            - containerPort: 8888  # Prometheus metrics
          resources:
            limits:
              cpu: "1"
              memory: 1Gi
            requests:
              cpu: 200m
              memory: 256Mi
          volumeMounts:
            - name: config
              mountPath: /etc/otel
      volumes:
        - name: config
          configMap:
            name: otel-collector-config

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-config
  namespace: observability
data:
  config.yaml: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
      
      # Collect collector's own metrics
      prometheus:
        config:
          scrape_configs:
            - job_name: otel-collector
              scrape_interval: 10s
              static_configs:
                - targets: [0.0.0.0:8888]
    
    processors:
      # Batch traces before sending
      batch:
        timeout: 5s
        send_batch_size: 512
        send_batch_max_size: 1024
      
      # Add k8s metadata
      k8sattributes:
        auth_type: serviceAccount
        passthrough: false
        filter:
          node_from_env_var: NODE_NAME
        extract:
          metadata:
            - k8s.pod.name
            - k8s.pod.uid
            - k8s.deployment.name
            - k8s.namespace.name
            - k8s.node.name
            - k8s.container.name
          labels:
            - tag_name: app.version
              key: app.kubernetes.io/version
              from: pod
        pod_association:
          - sources:
              - from: resource_attribute
                name: k8s.pod.ip
      
      # Memory limiter to prevent OOM
      memory_limiter:
        check_interval: 5s
        limit_mib: 800
        spike_limit_mib: 200
      
      # Tail sampling for smart sampling decisions
      tail_sampling:
        decision_wait: 10s
        num_traces: 50000
        policies:
          - name: always-sample-errors
            type: status_code
            status_code:
              status_codes: [ERROR]
          - name: always-sample-slow-requests
            type: latency
            latency:
              threshold_ms: 2000
          - name: probabilistic-10-percent
            type: probabilistic
            probabilistic:
              sampling_percentage: 10
    
    exporters:
      jaeger:
        endpoint: jaeger-production-collector.observability:14250
        tls:
          insecure: true
      
      prometheus:
        endpoint: 0.0.0.0:8889
      
      logging:
        verbosity: normal
    
    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [memory_limiter, k8sattributes, batch, tail_sampling]
          exporters: [jaeger]
        metrics:
          receivers: [otlp, prometheus]
          processors: [memory_limiter, batch]
          exporters: [prometheus]
        logs:
          receivers: [otlp]
          processors: [memory_limiter, batch]
          exporters: [logging]
```

---

## 3. Custom Spans with Attributes

```typescript
// src/telemetry/tracing.ts
import {
  trace,
  context,
  SpanStatusCode,
  SpanKind,
  Attributes,
  Context,
} from '@opentelemetry/api';

const tracer = trace.getTracer('order-service', '1.0.0');

// Decorator for tracing class methods
export function Traced(spanName?: string, attributes?: Attributes) {
  return function (
    target: any,
    propertyKey: string,
    descriptor: PropertyDescriptor
  ) {
    const originalMethod = descriptor.value;
    const name = spanName || `${target.constructor.name}.${propertyKey}`;
    
    descriptor.value = async function (...args: any[]) {
      return tracer.startActiveSpan(name, { attributes }, async (span) => {
        try {
          const result = await originalMethod.apply(this, args);
          span.setStatus({ code: SpanStatusCode.OK });
          return result;
        } catch (error) {
          span.setStatus({
            code: SpanStatusCode.ERROR,
            message: error instanceof Error ? error.message : String(error),
          });
          span.recordException(error as Error);
          throw error;
        } finally {
          span.end();
        }
      });
    };
    
    return descriptor;
  };
}

// Functional wrapper for tracing
export async function withSpan<T>(
  spanName: string,
  fn: (span: ReturnType<typeof tracer.startSpan>) => Promise<T>,
  options?: {
    attributes?: Attributes;
    kind?: SpanKind;
    parentContext?: Context;
  }
): Promise<T> {
  const ctx = options?.parentContext ?? context.active();
  
  return tracer.startActiveSpan(
    spanName,
    {
      kind: options?.kind ?? SpanKind.INTERNAL,
      attributes: options?.attributes,
    },
    ctx,
    async (span) => {
      try {
        const result = await fn(span);
        span.setStatus({ code: SpanStatusCode.OK });
        return result;
      } catch (error) {
        const err = error instanceof Error ? error : new Error(String(error));
        span.recordException(err);
        span.setStatus({
          code: SpanStatusCode.ERROR,
          message: err.message,
        });
        throw error;
      } finally {
        span.end();
      }
    }
  );
}

// Order Service with detailed tracing
export class OrderService {
  @Traced('OrderService.createOrder', { 'order.source': 'api' })
  async createOrder(orderData: CreateOrderDto, userId: string): Promise<Order> {
    const span = trace.getActiveSpan();
    
    // Add business context to span
    span?.setAttributes({
      'order.user_id': userId,
      'order.items_count': orderData.items.length,
      'order.total_amount': orderData.items.reduce(
        (sum, item) => sum + item.price * item.quantity, 0
      ),
      'order.currency': orderData.currency,
    });
    
    // Step 1: Validate inventory
    const inventoryResult = await withSpan(
      'validate-inventory',
      async (span) => {
        span.setAttributes({
          'inventory.item_ids': orderData.items.map(i => i.productId).join(','),
        });
        return this.inventoryService.checkAvailability(orderData.items);
      },
      { kind: SpanKind.CLIENT }
    );
    
    span?.addEvent('inventory-validated', {
      'available': inventoryResult.allAvailable,
    });
    
    if (!inventoryResult.allAvailable) {
      span?.setAttributes({
        'order.failed_reason': 'insufficient_inventory',
        'order.unavailable_items': inventoryResult.unavailableItems.join(','),
      });
      throw new InsufficientInventoryError(inventoryResult.unavailableItems);
    }
    
    // Step 2: Calculate pricing
    const pricing = await withSpan(
      'calculate-pricing',
      async (span) => {
        span.setAttribute('pricing.discount_code', orderData.discountCode || 'none');
        return this.pricingService.calculate(orderData);
      }
    );
    
    span?.addEvent('pricing-calculated', {
      'subtotal': pricing.subtotal,
      'discount': pricing.discount,
      'total': pricing.total,
    });
    
    // Step 3: Create order in database
    const order = await withSpan(
      'db.insert-order',
      async (span) => {
        span.setAttributes({
          'db.system': 'postgresql',
          'db.operation': 'INSERT',
          'db.table': 'orders',
        });
        return this.orderRepository.create({
          ...orderData,
          userId,
          totalAmount: pricing.total,
          status: 'pending',
        });
      },
      { kind: SpanKind.CLIENT }
    );
    
    span?.setAttributes({
      'order.id': order.id,
      'order.status': order.status,
    });
    
    // Step 4: Publish domain event
    await withSpan(
      'messaging.publish-order-created',
      async (span) => {
        span.setAttributes({
          'messaging.system': 'rabbitmq',
          'messaging.destination': 'orders.created',
          'messaging.message_id': order.id,
        });
        return this.eventBus.publish('orders.created', {
          orderId: order.id,
          userId,
          totalAmount: pricing.total,
        });
      },
      { kind: SpanKind.PRODUCER }
    );
    
    return order;
  }
}
```

---

## 4. Context Propagation ข้าม Services

```typescript
// src/telemetry/propagation.ts
import {
  context,
  propagation,
  trace,
  ROOT_CONTEXT,
  Context,
} from '@opentelemetry/api';
import { W3CTraceContextPropagator } from '@opentelemetry/core';
import { Request, Response, NextFunction } from 'express';
import amqplib from 'amqplib';

// HTTP Context Extraction Middleware
export function extractTraceContext() {
  return (req: Request, res: Response, next: NextFunction) => {
    const carrier = req.headers;
    const parentContext = propagation.extract(ROOT_CONTEXT, carrier);
    
    context.with(parentContext, () => {
      next();
    });
  };
}

// HTTP Context Injection for outgoing requests
export class TracedHttpClient {
  async get(url: string, options: RequestInit = {}): Promise<Response> {
    const carrier: Record<string, string> = {};
    propagation.inject(context.active(), carrier);
    
    return fetch(url, {
      ...options,
      headers: {
        ...options.headers,
        ...carrier, // Inject traceparent, tracestate headers
      },
    });
  }
  
  async post(url: string, body: unknown, options: RequestInit = {}): Promise<Response> {
    const carrier: Record<string, string> = {};
    propagation.inject(context.active(), carrier);
    
    return fetch(url, {
      method: 'POST',
      ...options,
      headers: {
        'Content-Type': 'application/json',
        ...options.headers,
        ...carrier,
      },
      body: JSON.stringify(body),
    });
  }
}

// RabbitMQ Context Propagation
export function injectTraceContextToMessage(
  message: Record<string, unknown>
): Record<string, unknown> {
  const carrier: Record<string, string> = {};
  propagation.inject(context.active(), carrier);
  
  return {
    ...message,
    _tracing: carrier,
  };
}

export function extractTraceContextFromMessage(
  message: Record<string, unknown>
): Context {
  const carrier = (message._tracing as Record<string, string>) || {};
  return propagation.extract(ROOT_CONTEXT, carrier);
}

// Kafka Context Propagation
export function injectTraceToKafkaHeaders(): Record<string, Buffer> {
  const carrier: Record<string, string> = {};
  propagation.inject(context.active(), carrier);
  
  const headers: Record<string, Buffer> = {};
  for (const [key, value] of Object.entries(carrier)) {
    headers[key] = Buffer.from(value);
  }
  return headers;
}

export function extractTraceFromKafkaHeaders(
  headers: Record<string, Buffer | string | undefined>
): Context {
  const carrier: Record<string, string> = {};
  for (const [key, value] of Object.entries(headers)) {
    if (value) {
      carrier[key] = typeof value === 'string' ? value : value.toString();
    }
  }
  return propagation.extract(ROOT_CONTEXT, carrier);
}

// Message consumer with context propagation
export class TracedMessageConsumer {
  async processMessage(
    channel: amqplib.Channel,
    message: amqplib.ConsumeMessage
  ) {
    const payload = JSON.parse(message.content.toString());
    const parentContext = extractTraceContextFromMessage(payload);
    
    const tracer = trace.getTracer('order-consumer');
    
    // Create a new span as child of the publisher's span
    return context.with(parentContext, () => {
      return tracer.startActiveSpan(
        'process-order-message',
        {
          kind: SpanKind.CONSUMER,
          attributes: {
            'messaging.system': 'rabbitmq',
            'messaging.source': message.fields.routingKey,
            'messaging.message_id': message.properties.messageId,
          },
        },
        async (span) => {
          try {
            await this.handlePayload(payload);
            channel.ack(message);
            span.setStatus({ code: SpanStatusCode.OK });
          } catch (error) {
            channel.nack(message, false, false); // Dead letter
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
    });
  }
  
  private async handlePayload(payload: any) {
    // Business logic
  }
}
```

---

## 5. Sampling Strategies

### 5.1 Head-based Sampling

```typescript
// src/telemetry/sampling.ts
import {
  Sampler,
  SamplingDecision,
  SamplingResult,
  SpanKind,
  Attributes,
  Link,
  Context,
  TraceFlags,
} from '@opentelemetry/api';

// Custom sampler: always sample errors and slow requests
export class SmartSampler implements Sampler {
  constructor(
    private readonly baseRate: number = 0.1,
    private readonly alwaysSamplePaths: string[] = ['/api/payments', '/api/checkout'],
    private readonly neverSamplePaths: string[] = ['/health', '/metrics', '/ready']
  ) {}
  
  shouldSample(
    context: Context,
    traceId: string,
    spanName: string,
    spanKind: SpanKind,
    attributes: Attributes,
    links: Link[]
  ): SamplingResult {
    const httpTarget = attributes['http.target'] as string || '';
    const httpMethod = attributes['http.method'] as string || '';
    
    // Never trace health checks
    if (this.neverSamplePaths.some(path => httpTarget.startsWith(path))) {
      return { decision: SamplingDecision.NOT_RECORD };
    }
    
    // Always trace specific paths
    if (this.alwaysSamplePaths.some(path => httpTarget.startsWith(path))) {
      return {
        decision: SamplingDecision.RECORD_AND_SAMPLED,
        attributes: { 'sampling.reason': 'priority_path' },
      };
    }
    
    // Always trace non-GET requests to API
    if (httpMethod !== 'GET' && httpTarget.startsWith('/api/')) {
      return {
        decision: SamplingDecision.RECORD_AND_SAMPLED,
        attributes: { 'sampling.reason': 'mutation_operation' },
      };
    }
    
    // Probabilistic sampling for the rest
    const traceIdAsHex = traceId.replace(/-/g, '');
    const traceIdAsFloat = parseInt(traceIdAsHex.slice(0, 8), 16) / 0xffffffff;
    
    if (traceIdAsFloat < this.baseRate) {
      return {
        decision: SamplingDecision.RECORD_AND_SAMPLED,
        attributes: { 'sampling.reason': 'probabilistic' },
      };
    }
    
    return { decision: SamplingDecision.NOT_RECORD };
  }
  
  toString(): string {
    return `SmartSampler(baseRate=${this.baseRate})`;
  }
}

// Tail-based sampling via OTel Collector (config shown in Kubernetes section)
// For application-level tail sampling, use a custom SpanProcessor:
export class ErrorRateSamplingProcessor {
  private readonly windowSize = 60000; // 1 minute
  private errorCount = 0;
  private totalCount = 0;
  private windowStart = Date.now();
  private isSamplingAll = false;
  
  onStart(span: any) {
    // Check if we should enable full sampling due to high error rate
    const now = Date.now();
    if (now - this.windowStart > this.windowSize) {
      const errorRate = this.totalCount > 0 ? this.errorCount / this.totalCount : 0;
      this.isSamplingAll = errorRate > 0.05; // Enable full sampling if >5% error rate
      this.errorCount = 0;
      this.totalCount = 0;
      this.windowStart = now;
    }
    
    if (this.isSamplingAll) {
      span.setAttribute('sampling.override', 'high_error_rate');
    }
  }
  
  onEnd(span: any) {
    this.totalCount++;
    if (span.status?.code === SpanStatusCode.ERROR) {
      this.errorCount++;
    }
  }
  
  shutdown() { return Promise.resolve(); }
  forceFlush() { return Promise.resolve(); }
}
```

---

## 6. Performance Profiling with Pyroscope

### 6.1 Pyroscope Kubernetes Deployment

```yaml
# kubernetes/pyroscope/pyroscope.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: pyroscope

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pyroscope
  namespace: pyroscope
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pyroscope
  template:
    metadata:
      labels:
        app: pyroscope
    spec:
      containers:
        - name: pyroscope
          image: grafana/pyroscope:1.2.0
          ports:
            - containerPort: 4040
          env:
            - name: PYROSCOPE_STORAGE_BACKEND
              value: s3
            - name: PYROSCOPE_STORAGE_S3_BUCKET_NAME
              value: pyroscope-profiles
            - name: PYROSCOPE_STORAGE_S3_REGION
              value: us-east-1
          resources:
            limits:
              cpu: "2"
              memory: 4Gi
            requests:
              cpu: 500m
              memory: 1Gi
          volumeMounts:
            - name: config
              mountPath: /etc/pyroscope
      volumes:
        - name: config
          configMap:
            name: pyroscope-config

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: pyroscope-config
  namespace: pyroscope
data:
  config.yaml: |
    server:
      http_listen_port: 4040
    
    storage:
      backend: s3
      s3:
        bucket_name: pyroscope-profiles
        region: us-east-1
    
    ingester:
      max_block_duration: 30m
    
    compactor:
      working_directory: /tmp/pyroscope
    
    limits:
      max_query_lookback: 720h  # 30 days
      max_series_per_user: 10000
```

### 6.2 Node.js Profiling Integration

```typescript
// src/profiling/pyroscope.ts
import Pyroscope from '@pyroscope/nodejs';

export function initProfiling() {
  if (process.env.NODE_ENV !== 'production' && process.env.ENABLE_PROFILING !== 'true') {
    return;
  }
  
  Pyroscope.init({
    serverAddress: process.env.PYROSCOPE_URL || 'http://pyroscope.pyroscope:4040',
    appName: process.env.SERVICE_NAME || 'unknown-service',
    tags: {
      version: process.env.SERVICE_VERSION || '0.0.0',
      environment: process.env.NODE_ENV || 'production',
      region: process.env.AWS_REGION || 'us-east-1',
      'k8s.pod': process.env.POD_NAME || '',
    },
    wall: {
      samplingDurationMs: 10000, // Collect 10s profiles
      samplingIntervalMs: 10,    // Sample every 10ms
    },
    heap: {
      collectAllocations: true,
    },
  });
  
  Pyroscope.start();
  
  console.log('Pyroscope profiling started');
  
  process.on('SIGTERM', () => {
    Pyroscope.stop();
  });
}

// Tag specific code sections for profiling
export async function profiledOperation<T>(
  label: string,
  fn: () => Promise<T>,
  tags?: Record<string, string>
): Promise<T> {
  const start = performance.now();
  
  try {
    return await fn();
  } finally {
    const duration = performance.now() - start;
    
    // If operation took more than 100ms, label it for easier profiling
    if (duration > 100) {
      Pyroscope.wrapWithLabels(
        { operation: label, slow: 'true', ...tags },
        () => {} // Labels applied retroactively via flush
      );
    }
  }
}
```

---

## 7. Trace-based Testing

```typescript
// src/tests/trace-testing.ts
import { InMemorySpanExporter, SimpleSpanProcessor } from '@opentelemetry/sdk-trace-base';
import { NodeTracerProvider } from '@opentelemetry/sdk-trace-node';
import { trace, context } from '@opentelemetry/api';
import { describe, it, expect, beforeEach, afterEach } from 'vitest';

// Setup in-memory span collection for tests
export class TraceTestHelper {
  private exporter = new InMemorySpanExporter();
  private provider: NodeTracerProvider;
  
  setup() {
    this.provider = new NodeTracerProvider();
    this.provider.addSpanProcessor(new SimpleSpanProcessor(this.exporter));
    this.provider.register();
  }
  
  teardown() {
    this.exporter.reset();
    this.provider.shutdown();
  }
  
  getSpans() {
    return this.exporter.getFinishedSpans();
  }
  
  getSpanByName(name: string) {
    return this.getSpans().find(span => span.name === name);
  }
  
  getSpansByName(name: string) {
    return this.getSpans().filter(span => span.name === name);
  }
  
  getChildSpans(parentSpanId: string) {
    return this.getSpans().filter(
      span => span.parentSpanId === parentSpanId
    );
  }
  
  assertSpanExists(name: string): void {
    const span = this.getSpanByName(name);
    if (!span) {
      throw new Error(`Expected span "${name}" to exist. Found spans: ${
        this.getSpans().map(s => s.name).join(', ')
      }`);
    }
  }
  
  assertSpanAttribute(spanName: string, key: string, value: unknown): void {
    const span = this.getSpanByName(spanName);
    if (!span) {
      throw new Error(`Span "${spanName}" not found`);
    }
    
    const actual = span.attributes[key];
    if (actual !== value) {
      throw new Error(
        `Expected span "${spanName}" to have attribute "${key}=${value}", got "${actual}"`
      );
    }
  }
  
  assertNoErrors(): void {
    const errorSpans = this.getSpans().filter(
      span => span.status.code === SpanStatusCode.ERROR
    );
    
    if (errorSpans.length > 0) {
      throw new Error(
        `Found ${errorSpans.length} spans with errors: ${
          errorSpans.map(s => `${s.name}: ${s.status.message}`).join(', ')
        }`
      );
    }
  }
}

// Usage in tests
const traceHelper = new TraceTestHelper();

describe('OrderService tracing', () => {
  beforeEach(() => {
    traceHelper.setup();
  });
  
  afterEach(() => {
    traceHelper.teardown();
  });
  
  it('should create spans for order creation flow', async () => {
    const orderService = createOrderService();
    
    await orderService.createOrder(
      { items: [{ productId: 'prod-1', quantity: 2, price: 29.99 }], currency: 'USD' },
      'user-123'
    );
    
    // Verify root span
    traceHelper.assertSpanExists('OrderService.createOrder');
    traceHelper.assertSpanAttribute('OrderService.createOrder', 'order.user_id', 'user-123');
    traceHelper.assertSpanAttribute('OrderService.createOrder', 'order.items_count', 1);
    
    // Verify child spans
    traceHelper.assertSpanExists('validate-inventory');
    traceHelper.assertSpanExists('calculate-pricing');
    traceHelper.assertSpanExists('db.insert-order');
    traceHelper.assertSpanExists('messaging.publish-order-created');
    
    // Verify no errors
    traceHelper.assertNoErrors();
    
    // Verify span hierarchy
    const rootSpan = traceHelper.getSpanByName('OrderService.createOrder')!;
    const childSpans = traceHelper.getChildSpans(rootSpan.spanContext().spanId);
    expect(childSpans).toHaveLength(4);
  });
  
  it('should record error span on inventory failure', async () => {
    const orderService = createOrderServiceWithFailingInventory();
    
    await expect(
      orderService.createOrder(
        { items: [{ productId: 'out-of-stock-prod', quantity: 100, price: 9.99 }], currency: 'USD' },
        'user-456'
      )
    ).rejects.toThrow('Insufficient inventory');
    
    const rootSpan = traceHelper.getSpanByName('OrderService.createOrder')!;
    expect(rootSpan.status.code).toBe(SpanStatusCode.ERROR);
    expect(rootSpan.events).toContainEqual(
      expect.objectContaining({ name: 'exception' })
    );
  });
});
```

---

## 8. Structured Logging ที่เชื่อมกับ Traces

```typescript
// src/telemetry/logger.ts
import { trace, context, SpanStatusCode } from '@opentelemetry/api';
import winston from 'winston';
import { LoggerProvider, logs, SeverityNumber } from '@opentelemetry/api-logs';

const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json(),
    winston.format((info) => {
      // Inject trace context into every log
      const span = trace.getActiveSpan();
      if (span) {
        const spanContext = span.spanContext();
        info['trace.id'] = spanContext.traceId;
        info['span.id'] = spanContext.spanId;
        info['trace.flags'] = spanContext.traceFlags;
        
        // For Datadog/Grafana Loki correlation
        info['dd.trace_id'] = BigInt(`0x${spanContext.traceId.slice(16)}`).toString();
        info['dd.span_id'] = BigInt(`0x${spanContext.spanId}`).toString();
      }
      return info;
    })()
  ),
  transports: [
    new winston.transports.Console(),
    // Production: send to log aggregator via OTLP
  ],
});

// High-level API that correlates logs with spans
export const log = {
  info: (message: string, data?: object) => {
    logger.info(message, data);
  },
  
  error: (message: string, error?: Error, data?: object) => {
    // Record exception in active span
    const span = trace.getActiveSpan();
    if (error && span) {
      span.recordException(error);
      span.setStatus({
        code: SpanStatusCode.ERROR,
        message: error.message,
      });
    }
    
    logger.error(message, { error: error?.stack, ...data });
  },
  
  audit: (action: string, data: object) => {
    const span = trace.getActiveSpan();
    span?.addEvent('audit', {
      'audit.action': action,
      ...Object.fromEntries(
        Object.entries(data).map(([k, v]) => [`audit.${k}`, String(v)])
      ),
    });
    
    logger.info(action, { type: 'audit', ...data });
  },
};
```

---

## สรุป

ในบทนี้เราได้เรียนรู้ Distributed Tracing ขั้นสูงครอบคลุม:

1. **OpenTelemetry SDK Setup** — การตั้งค่า complete pipeline สำหรับ traces, metrics, logs พร้อม auto-instrumentation และ graceful shutdown

2. **Jaeger on Kubernetes** — Production deployment พร้อม Elasticsearch backend, OTel Collector สำหรับ tail-based sampling, และ k8s metadata enrichment

3. **Custom Spans** — Decorator pattern, functional wrapper, และ business context attributes ที่ช่วยให้ debug ได้ง่าย

4. **Context Propagation** — W3C TraceContext across HTTP, RabbitMQ, Kafka ให้ end-to-end trace ครบ

5. **Sampling Strategies** — Head-based (SmartSampler) และ tail-based (OTel Collector policy) รองรับทั้ง always-sample errors และ probabilistic sampling

6. **Pyroscope Profiling** — Continuous profiling ที่ correlate กับ traces เพื่อ identify performance bottlenecks

7. **Trace-based Testing** — InMemorySpanExporter สำหรับ unit/integration tests ที่ verify instrumentation ครบ

8. **Structured Logging** — Log records ที่มี trace/span IDs เพื่อ correlate logs กับ distributed traces

Key takeaways:
- โหลด instrumentation.ts เป็น module แรกสุดผ่าน `--require` flag
- ใช้ tail-based sampling ที่ OTel Collector level สำหรับ smarter sampling decisions
- เพิ่ม business context ลงใน spans เสมอ ไม่ใช่แค่ technical attributes
- ทดสอบ instrumentation ด้วย InMemorySpanExporter ใน test suite

# Part 85: Debugging Microservices

## บทนำ

การ debug ระบบ Microservices มีความซับซ้อนมากกว่า Monolith เนื่องจาก request หนึ่งอาจผ่านหลาย service การหา root cause จึงต้องใช้เครื่องมือหลายอย่างประกอบกัน บทนี้จะแนะนำวิธีการ debug ตั้งแต่ correlation IDs ไปจนถึง chaos engineering

---

## 1. Distributed Debugging with Correlation IDs

### 1.1 Correlation ID Middleware

```typescript
// src/middleware/correlation.ts
import { Request, Response, NextFunction } from 'express';
import { v4 as uuidv4 } from 'uuid';
import { AsyncLocalStorage } from 'async_hooks';

interface RequestContext {
  requestId: string;
  correlationId: string;
  userId?: string;
  sessionId?: string;
  startTime: number;
}

// เก็บ context ใน async context (ไม่ต้อง pass ตลอด)
export const asyncLocalStorage = new AsyncLocalStorage<RequestContext>();

export const correlationMiddleware = (req: Request, res: Response, next: NextFunction) => {
  // รับ correlation ID จาก upstream หรือสร้างใหม่
  const correlationId = 
    req.headers['x-correlation-id'] as string ||
    req.headers['x-trace-id'] as string ||
    uuidv4();
  
  const requestId = uuidv4();
  
  const context: RequestContext = {
    requestId,
    correlationId,
    userId: req.headers['x-user-id'] as string,
    sessionId: req.headers['x-session-id'] as string,
    startTime: Date.now(),
  };
  
  // ตั้ง response headers
  res.setHeader('X-Request-ID', requestId);
  res.setHeader('X-Correlation-ID', correlationId);
  
  // เก็บ context ใน AsyncLocalStorage
  asyncLocalStorage.run(context, () => {
    next();
  });
};

// Helper function เพื่อดึง context
export function getRequestContext(): RequestContext | undefined {
  return asyncLocalStorage.getStore();
}

// ใช้ใน logger
export function getCorrelationId(): string {
  return getRequestContext()?.correlationId || 'unknown';
}
```

### 1.2 Propagation ไปยัง Downstream Services

```typescript
// src/utils/http-client.ts
import axios, { AxiosInstance, AxiosRequestConfig } from 'axios';
import { getRequestContext } from './correlation';

export function createTracedHttpClient(baseURL: string): AxiosInstance {
  const client = axios.create({ baseURL });
  
  // Interceptor: เพิ่ม correlation headers ทุก outbound request
  client.interceptors.request.use((config) => {
    const ctx = getRequestContext();
    
    if (ctx) {
      config.headers = config.headers || {};
      config.headers['X-Correlation-ID'] = ctx.correlationId;
      config.headers['X-Request-ID'] = ctx.requestId;
      config.headers['X-Parent-Request-ID'] = ctx.requestId;
      
      if (ctx.userId) {
        config.headers['X-User-ID'] = ctx.userId;
      }
    }
    
    return config;
  });
  
  // Interceptor: log outbound requests
  client.interceptors.request.use((config) => {
    const ctx = getRequestContext();
    console.log({
      level: 'info',
      message: 'Outbound HTTP request',
      correlationId: ctx?.correlationId,
      method: config.method?.toUpperCase(),
      url: `${config.baseURL}${config.url}`,
      timestamp: new Date().toISOString(),
    });
    return config;
  });
  
  return client;
}
```

---

## 2. Remote Debugging with Node.js Inspector

### 2.1 Debug Configuration

```yaml
# docker-compose.debug.yml
version: '3.8'
services:
  user-service:
    build:
      context: ./user-service
      dockerfile: Dockerfile.debug
    command: node --inspect=0.0.0.0:9229 --inspect-brk dist/main.js
    ports:
      - "3001:3000"
      - "9229:9229"  # Debug port
    environment:
      NODE_ENV: development
    volumes:
      - ./user-service/src:/app/src:ro
```

```dockerfile
# Dockerfile.debug
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Install development tools
RUN npm install -g @nestjs/cli

EXPOSE 3000 9229
CMD ["node", "--inspect=0.0.0.0:9229", "dist/main.js"]
```

### 2.2 VS Code Debug Configuration

```json
// .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug User Service (Docker)",
      "type": "node",
      "request": "attach",
      "port": 9229,
      "address": "localhost",
      "localRoot": "${workspaceFolder}/user-service",
      "remoteRoot": "/app",
      "sourceMaps": true,
      "skipFiles": ["<node_internals>/**"],
      "restart": true,
      "timeout": 10000
    },
    {
      "name": "Debug All Services",
      "type": "node",
      "request": "attach",
      "port": 9229,
      "address": "localhost",
      "sourceMaps": true,
      "restart": true
    },
    {
      "name": "Debug Tests",
      "type": "node",
      "request": "launch",
      "runtimeExecutable": "node",
      "runtimeArgs": [
        "--inspect-brk",
        "${workspaceFolder}/node_modules/.bin/jest",
        "--runInBand",
        "--no-coverage"
      ],
      "cwd": "${workspaceFolder}",
      "console": "integratedTerminal"
    }
  ]
}
```

### 2.3 Production Debugging (Safe)

```typescript
// src/utils/conditional-debug.ts
import inspector from 'inspector';

export function enableDebugIfRequested(req: any) {
  // เปิด debug เฉพาะเมื่อมี debug token ที่ถูกต้อง
  const debugToken = req.headers['x-debug-token'];
  const expectedToken = process.env.DEBUG_TOKEN;
  
  if (!debugToken || !expectedToken || debugToken !== expectedToken) {
    return;
  }
  
  // ตรวจสอบ rate limiting สำหรับ debug sessions
  const session = new inspector.Session();
  session.connect();
  
  // เปิด profiling เป็นเวลา 10 วินาที
  session.post('Profiler.enable', () => {
    session.post('Profiler.start', () => {
      setTimeout(() => {
        session.post('Profiler.stop', (err, { profile }) => {
          // Save profile
          console.log('Profile captured:', JSON.stringify(profile).length, 'bytes');
          session.disconnect();
        });
      }, 10000);
    });
  });
}
```

---

## 3. Log Aggregation with ELK Stack

### 3.1 Elasticsearch + Logstash + Kibana Setup

```yaml
# docker-compose.elk.yml
version: '3.8'

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ports:
      - "9200:9200"
    volumes:
      - elasticsearch_data:/usr/share/elasticsearch/data
    healthcheck:
      test: curl -s http://localhost:9200/_cluster/health
      interval: 10s
      timeout: 5s
      retries: 10

  logstash:
    image: docker.elastic.co/logstash/logstash:8.11.0
    volumes:
      - ./config/logstash/pipeline:/usr/share/logstash/pipeline:ro
      - ./config/logstash/logstash.yml:/usr/share/logstash/config/logstash.yml:ro
    ports:
      - "5044:5044"   # Beats input
      - "5000:5000"   # TCP input
      - "9600:9600"   # Monitoring
    depends_on:
      elasticsearch:
        condition: service_healthy

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    environment:
      ELASTICSEARCH_HOSTS: '["http://elasticsearch:9200"]'
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch

  filebeat:
    image: docker.elastic.co/beats/filebeat:8.11.0
    user: root
    volumes:
      - ./config/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    depends_on:
      - logstash

volumes:
  elasticsearch_data:
```

```yaml
# config/logstash/pipeline/microservices.conf
input {
  beats {
    port => 5044
  }
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
    
    if "_jsonparsefailure" not in [tags] {
      mutate {
        replace => {
          "level" => "%{[parsed][level]}"
          "correlationId" => "%{[parsed][correlationId]}"
          "service" => "%{[parsed][service]}"
          "message" => "%{[parsed][message]}"
        }
      }
    }
  }
  
  # Extract service name from Docker container
  if [docker][container][labels][com_docker_compose_service] {
    mutate {
      add_field => {
        "service" => "%{[docker][container][labels][com_docker_compose_service]}"
      }
    }
  }
  
  # Parse timestamp
  date {
    match => ["[parsed][timestamp]", "ISO8601"]
    target => "@timestamp"
  }
  
  # Remove raw message after parsing
  mutate {
    remove_field => ["parsed", "message"]
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "microservices-logs-%{+YYYY.MM.dd}"
    document_id => "%{correlationId}-%{[@timestamp]}"
  }
  
  # Debug output
  if [@metadata][pipeline] == "debug" {
    stdout { codec => rubydebug }
  }
}
```

### 3.2 Structured Logging

```typescript
// src/utils/logger.ts
import winston from 'winston';
import { ElasticsearchTransport } from 'winston-elasticsearch';
import { getRequestContext } from './correlation';

const { combine, timestamp, errors, json } = winston.format;

// Custom format เพิ่ม correlation info
const correlationFormat = winston.format((info) => {
  const ctx = getRequestContext();
  if (ctx) {
    info.correlationId = ctx.correlationId;
    info.requestId = ctx.requestId;
    info.userId = ctx.userId;
  }
  return info;
})();

export const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: combine(
    timestamp({ format: 'YYYY-MM-DDTHH:mm:ss.SSSZ' }),
    errors({ stack: true }),
    correlationFormat,
    json()
  ),
  defaultMeta: {
    service: process.env.SERVICE_NAME || 'unknown',
    version: process.env.APP_VERSION || '0.0.0',
    environment: process.env.NODE_ENV || 'development',
  },
  transports: [
    // Console output
    new winston.transports.Console({
      format: process.env.NODE_ENV === 'development'
        ? winston.format.combine(
            winston.format.colorize(),
            winston.format.simple()
          )
        : json(),
    }),
    
    // File transport
    new winston.transports.File({
      filename: '/var/log/app/error.log',
      level: 'error',
      maxsize: 10 * 1024 * 1024, // 10MB
      maxFiles: 5,
    }),
    
    // Elasticsearch transport (production)
    ...(process.env.ELASTICSEARCH_URL ? [
      new ElasticsearchTransport({
        level: 'info',
        clientOpts: {
          node: process.env.ELASTICSEARCH_URL,
        },
        index: `microservices-${process.env.SERVICE_NAME}`,
      })
    ] : []),
  ],
});

// Convenience methods with structured data
export const log = {
  info: (message: string, data?: Record<string, any>) => 
    logger.info(message, data),
  
  error: (message: string, error: Error, data?: Record<string, any>) =>
    logger.error(message, {
      ...data,
      error: {
        message: error.message,
        name: error.name,
        stack: error.stack,
      },
    }),
  
  warn: (message: string, data?: Record<string, any>) =>
    logger.warn(message, data),
  
  debug: (message: string, data?: Record<string, any>) =>
    logger.debug(message, data),
    
  // HTTP request logging
  http: (req: any, res: any, duration: number) =>
    logger.info('HTTP Request', {
      method: req.method,
      url: req.url,
      statusCode: res.statusCode,
      duration,
      userAgent: req.headers['user-agent'],
      ip: req.ip,
    }),
};
```

---

## 4. Distributed Tracing Analysis (Jaeger/Zipkin)

### 4.1 OpenTelemetry Setup

```typescript
// src/tracing.ts - ต้อง import ก่อน everything else
import { NodeSDK } from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';
import { Resource } from '@opentelemetry/resources';
import { SemanticResourceAttributes } from '@opentelemetry/semantic-conventions';
import { BatchSpanProcessor } from '@opentelemetry/sdk-trace-base';
import { trace, context, SpanStatusCode } from '@opentelemetry/api';

const resource = Resource.default().merge(
  new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]: process.env.SERVICE_NAME || 'user-service',
    [SemanticResourceAttributes.SERVICE_VERSION]: process.env.APP_VERSION || '1.0.0',
    [SemanticResourceAttributes.DEPLOYMENT_ENVIRONMENT]: process.env.NODE_ENV || 'development',
  })
);

const traceExporter = new OTLPTraceExporter({
  url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT || 'http://jaeger:4318/v1/traces',
});

const sdk = new NodeSDK({
  resource,
  spanProcessor: new BatchSpanProcessor(traceExporter),
  instrumentations: [
    getNodeAutoInstrumentations({
      '@opentelemetry/instrumentation-fs': { enabled: false },
      '@opentelemetry/instrumentation-http': {
        requestHook: (span, request) => {
          // เพิ่ม correlation ID ใน span attributes
          const correlationId = (request as any).headers?.['x-correlation-id'];
          if (correlationId) {
            span.setAttribute('correlation.id', correlationId);
          }
        },
      },
    }),
  ],
});

sdk.start();

// Manual tracing สำหรับ business logic
export const tracer = trace.getTracer(process.env.SERVICE_NAME || 'user-service');

export async function traceOperation<T>(
  operationName: string,
  operation: () => Promise<T>,
  attributes?: Record<string, string | number | boolean>
): Promise<T> {
  const span = tracer.startSpan(operationName, { attributes });
  
  try {
    const result = await context.with(
      trace.setSpan(context.active(), span),
      operation
    );
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
}
```

### 4.2 Jaeger Setup

```yaml
# docker-compose.tracing.yml
version: '3.8'
services:
  jaeger:
    image: jaegertracing/all-in-one:1.52
    environment:
      COLLECTOR_OTLP_ENABLED: "true"
      SPAN_STORAGE_TYPE: elasticsearch
      ES_SERVER_URLS: http://elasticsearch:9200
    ports:
      - "5775:5775/udp"   # UDP compact
      - "6831:6831/udp"   # UDP binary
      - "6832:6832/udp"   # UDP binary compact
      - "5778:5778"       # HTTP configs
      - "16686:16686"     # Jaeger UI
      - "14268:14268"     # HTTP collector
      - "4317:4317"       # OTLP gRPC
      - "4318:4318"       # OTLP HTTP
```

---

## 5. Memory Leak Detection

### 5.1 Memory Monitoring

```typescript
// src/utils/memory-monitor.ts
import v8 from 'v8';
import { performance } from 'perf_hooks';

interface MemorySnapshot {
  timestamp: string;
  heapUsed: number;
  heapTotal: number;
  external: number;
  rss: number;
  heapUsedMB: number;
  heapTotalMB: number;
  rssMB: number;
}

export class MemoryMonitor {
  private snapshots: MemorySnapshot[] = [];
  private interval?: NodeJS.Timeout;
  private readonly maxSnapshots = 100;
  private readonly alertThresholdMB = 500;

  start(intervalMs: number = 10000): void {
    this.interval = setInterval(() => {
      this.takeSnapshot();
    }, intervalMs);
  }

  stop(): void {
    if (this.interval) {
      clearInterval(this.interval);
    }
  }

  takeSnapshot(): MemorySnapshot {
    const mem = process.memoryUsage();
    
    const snapshot: MemorySnapshot = {
      timestamp: new Date().toISOString(),
      heapUsed: mem.heapUsed,
      heapTotal: mem.heapTotal,
      external: mem.external,
      rss: mem.rss,
      heapUsedMB: Math.round(mem.heapUsed / 1024 / 1024 * 100) / 100,
      heapTotalMB: Math.round(mem.heapTotal / 1024 / 1024 * 100) / 100,
      rssMB: Math.round(mem.rss / 1024 / 1024 * 100) / 100,
    };
    
    this.snapshots.push(snapshot);
    
    if (this.snapshots.length > this.maxSnapshots) {
      this.snapshots.shift();
    }
    
    // Alert if memory exceeds threshold
    if (snapshot.heapUsedMB > this.alertThresholdMB) {
      console.error({
        level: 'error',
        message: 'HIGH_MEMORY_USAGE',
        snapshot,
        alert: `Heap used ${snapshot.heapUsedMB}MB exceeds threshold ${this.alertThresholdMB}MB`,
      });
    }
    
    return snapshot;
  }

  // ตรวจหา memory leak โดยดูแนวโน้ม
  detectLeak(): { isLeaking: boolean; growthRateMB: number; message: string } {
    if (this.snapshots.length < 10) {
      return { isLeaking: false, growthRateMB: 0, message: 'Insufficient data' };
    }
    
    const recent = this.snapshots.slice(-10);
    const first = recent[0].heapUsedMB;
    const last = recent[recent.length - 1].heapUsedMB;
    const growthMB = last - first;
    const growthRate = growthMB / first;
    
    const isLeaking = growthRate > 0.2; // เพิ่มมากกว่า 20% ใน 10 snapshots
    
    return {
      isLeaking,
      growthRateMB: Math.round(growthMB * 100) / 100,
      message: isLeaking 
        ? `Potential memory leak: +${growthMB.toFixed(2)}MB (${(growthRate * 100).toFixed(1)}%) in last ${this.snapshots.length} samples`
        : 'Memory usage appears stable',
    };
  }

  // Force GC (ต้องรัน Node.js ด้วย --expose-gc)
  forceGC(): void {
    if (global.gc) {
      global.gc();
    }
  }

  // Heap snapshot สำหรับ analysis ใน Chrome DevTools
  writeHeapSnapshot(path: string): string {
    return v8.writeHeapSnapshot(path);
  }

  getStats(): {
    current: MemorySnapshot;
    trend: ReturnType<MemoryMonitor['detectLeak']>;
    v8Stats: v8.HeapStatistics;
  } {
    return {
      current: this.takeSnapshot(),
      trend: this.detectLeak(),
      v8Stats: v8.getHeapStatistics(),
    };
  }
}

// Memory leak ตัวอย่าง (และวิธีแก้)
class LeakyEventEmitter {
  private emitter = new EventEmitter();
  private handlers: Function[] = [];
  
  // ❌ Bad: สะสม listeners ไม่ได้ลบ
  addListenerBad(event: string, handler: Function) {
    this.emitter.on(event as any, handler as any);
    this.handlers.push(handler);
  }
  
  // ✅ Good: ลบ listener เมื่อไม่ใช้
  addListenerGood(event: string, handler: Function): () => void {
    this.emitter.on(event as any, handler as any);
    
    // Return cleanup function
    return () => {
      this.emitter.off(event as any, handler as any);
    };
  }
}
```

---

## 6. CPU Profiling in Production

```typescript
// src/utils/cpu-profiler.ts
import v8Profiler from 'v8-profiler-next';
import fs from 'fs';
import path from 'path';

export class CPUProfiler {
  private isProfilering = false;
  
  async profileEndpoint(
    handler: () => Promise<void>,
    options: {
      duration: number;     // milliseconds
      outputPath?: string;
      title?: string;
    }
  ): Promise<string> {
    if (this.isProfilering) {
      throw new Error('Profiling already in progress');
    }
    
    this.isProfilering = true;
    const title = options.title || `profile-${Date.now()}`;
    
    v8Profiler.startProfiling(title, true);
    
    try {
      // รัน handler ที่ต้องการ profile
      await handler();
      
      // รอจนครบเวลา
      if (options.duration > 0) {
        await new Promise(resolve => setTimeout(resolve, options.duration));
      }
    } finally {
      const profile = v8Profiler.stopProfiling(title);
      this.isProfilering = false;
      
      return new Promise((resolve, reject) => {
        profile.export((error: Error | null, result: string | Buffer) => {
          profile.delete();
          
          if (error) {
            reject(error);
            return;
          }
          
          const outputPath = options.outputPath || 
            path.join('/tmp', `${title}.cpuprofile`);
          
          fs.writeFileSync(outputPath, result as string);
          resolve(outputPath);
        });
      });
    }
  }
  
  // Profile endpoint สำหรับ production (ต้องมี auth)
  createProfileEndpoint() {
    return async (req: any, res: any) => {
      const duration = parseInt(req.query.duration) || 5000;
      const maxDuration = 30000; // สูงสุด 30 วินาที
      
      if (duration > maxDuration) {
        return res.status(400).json({ error: 'Duration too long' });
      }
      
      const outputPath = await this.profileEndpoint(
        async () => {},
        { duration: Math.min(duration, maxDuration) }
      );
      
      res.download(outputPath, 'profile.cpuprofile', () => {
        fs.unlinkSync(outputPath);
      });
    };
  }
}
```

---

## 7. Database Query Analysis

```typescript
// src/utils/query-analyzer.ts
import { DataSource } from 'typeorm';

interface SlowQuery {
  query: string;
  duration: number;
  timestamp: Date;
  correlationId?: string;
  parameters?: any[];
}

export class QueryAnalyzer {
  private slowQueries: SlowQuery[] = [];
  private readonly slowQueryThresholdMs = 1000;

  setupLogging(dataSource: DataSource): void {
    // TypeORM query logging
    (dataSource.driver as any).afterQuery = (
      query: string,
      parameters: any[],
      queryRunner: any
    ) => {
      const duration = queryRunner.data?.queryTime;
      
      if (duration > this.slowQueryThresholdMs) {
        const slowQuery: SlowQuery = {
          query,
          duration,
          timestamp: new Date(),
          correlationId: getCorrelationId(),
          parameters,
        };
        
        this.slowQueries.push(slowQuery);
        
        console.warn({
          level: 'warn',
          message: 'SLOW_QUERY',
          query: query.substring(0, 500),
          duration,
          correlationId: slowQuery.correlationId,
        });
        
        // Alert if too slow
        if (duration > 5000) {
          this.alertSlowQuery(slowQuery);
        }
      }
    };
  }

  async analyzeQuery(dataSource: DataSource, query: string): Promise<{
    executionPlan: any;
    estimatedCost: number;
    recommendations: string[];
  }> {
    // PostgreSQL EXPLAIN ANALYZE
    const plan = await dataSource.query(`EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) ${query}`);
    
    const executionPlan = plan[0]['QUERY PLAN'][0];
    const estimatedCost = executionPlan['Plan']['Total Cost'];
    
    const recommendations = this.generateRecommendations(executionPlan);
    
    return { executionPlan, estimatedCost, recommendations };
  }

  private generateRecommendations(plan: any): string[] {
    const recommendations: string[] = [];
    
    const checkNode = (node: any) => {
      // Sequential scan บน large tables
      if (node['Node Type'] === 'Seq Scan' && node['Actual Rows'] > 1000) {
        recommendations.push(
          `Consider adding index on table "${node['Relation Name']}" ` +
          `(seq scan returned ${node['Actual Rows']} rows)`
        );
      }
      
      // Hash join ที่ใช้ memory มาก
      if (node['Node Type'] === 'Hash' && node['Peak Memory Usage'] > 10000) {
        recommendations.push(
          `Hash join using ${node['Peak Memory Usage']}KB memory, consider nested loop`
        );
      }
      
      if (node['Plans']) {
        node['Plans'].forEach(checkNode);
      }
    };
    
    checkNode(plan['Plan']);
    return recommendations;
  }

  private async alertSlowQuery(query: SlowQuery): Promise<void> {
    // ส่ง alert ไปยัง monitoring system
    console.error({
      level: 'error',
      message: 'CRITICAL_SLOW_QUERY',
      ...query,
    });
  }

  getSlowQueryReport(): { queries: SlowQuery[]; stats: any } {
    const avgDuration = this.slowQueries.reduce((sum, q) => sum + q.duration, 0) /
                       (this.slowQueries.length || 1);
    
    return {
      queries: this.slowQueries.slice(-50),
      stats: {
        total: this.slowQueries.length,
        avgDuration,
        maxDuration: Math.max(...this.slowQueries.map(q => q.duration), 0),
      },
    };
  }
}
```

---

## 8. Network Debugging Tools

```typescript
// src/utils/network-debugger.ts
import { exec } from 'child_process';
import { promisify } from 'util';
import net from 'net';
import dns from 'dns';

const execAsync = promisify(exec);
const dnsResolve = promisify(dns.resolve);

export class NetworkDebugger {
  // ตรวจสอบ connectivity ไปยัง service
  async checkConnectivity(host: string, port: number, timeout: number = 5000): Promise<{
    connected: boolean;
    latencyMs?: number;
    error?: string;
  }> {
    return new Promise((resolve) => {
      const start = Date.now();
      const socket = new net.Socket();
      
      socket.setTimeout(timeout);
      
      socket.connect(port, host, () => {
        const latencyMs = Date.now() - start;
        socket.destroy();
        resolve({ connected: true, latencyMs });
      });
      
      socket.on('timeout', () => {
        socket.destroy();
        resolve({ connected: false, error: `Connection timed out after ${timeout}ms` });
      });
      
      socket.on('error', (err) => {
        resolve({ connected: false, error: err.message });
      });
    });
  }

  // DNS lookup
  async resolveDNS(hostname: string): Promise<{
    addresses: string[];
    latencyMs: number;
  }> {
    const start = Date.now();
    const addresses = await dnsResolve(hostname);
    return {
      addresses,
      latencyMs: Date.now() - start,
    };
  }

  // Health check endpoints
  async checkServiceHealth(services: Record<string, string>): Promise<
    Record<string, { healthy: boolean; latencyMs?: number; error?: string }>
  > {
    const results: Record<string, any> = {};
    
    await Promise.all(
      Object.entries(services).map(async ([name, url]) => {
        const start = Date.now();
        try {
          const response = await fetch(`${url}/health`, {
            signal: AbortSignal.timeout(5000),
          });
          results[name] = {
            healthy: response.ok,
            latencyMs: Date.now() - start,
            statusCode: response.status,
          };
        } catch (error) {
          results[name] = {
            healthy: false,
            latencyMs: Date.now() - start,
            error: (error as Error).message,
          };
        }
      })
    );
    
    return results;
  }
}
```

---

## 9. Chaos Engineering for Debugging

```typescript
// src/chaos/chaos-monkey.ts
import { Injectable } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';

interface ChaosConfig {
  enabled: boolean;
  probability: number;    // 0-1, โอกาสที่จะเกิด chaos
  type: ChaosType;
  targetPaths?: string[]; // paths ที่ affect
  excludePaths?: string[];
}

type ChaosType = 
  | 'latency'             // เพิ่ม latency
  | 'error'               // ส่ง error response
  | 'timeout'             // ทำให้ timeout
  | 'cpu_spike'           // spike CPU
  | 'memory_pressure';    // pressure memory

@Injectable()
export class ChaosMonkey {
  private config: ChaosConfig = {
    enabled: process.env.CHAOS_ENABLED === 'true',
    probability: parseFloat(process.env.CHAOS_PROBABILITY || '0.01'),
    type: 'latency',
  };

  // Middleware สำหรับ inject chaos
  middleware() {
    return async (req: Request, res: Response, next: NextFunction) => {
      if (!this.config.enabled) return next();
      if (!this.shouldApplyChaos(req.path)) return next();
      if (Math.random() > this.config.probability) return next();
      
      console.warn({
        level: 'warn',
        message: 'CHAOS_INJECTED',
        type: this.config.type,
        path: req.path,
        correlationId: req.headers['x-correlation-id'],
      });
      
      switch (this.config.type) {
        case 'latency':
          const delay = Math.random() * 5000;
          await new Promise(resolve => setTimeout(resolve, delay));
          break;
          
        case 'error':
          return res.status(500).json({
            error: { code: 'CHAOS_ERROR', message: 'Chaos monkey struck!' }
          });
          
        case 'timeout':
          await new Promise(resolve => setTimeout(resolve, 60000));
          return;
          
        case 'cpu_spike':
          this.spikeCPU(1000);
          break;
          
        case 'memory_pressure':
          this.pressureMemory(50);
          break;
      }
      
      next();
    };
  }

  private shouldApplyChaos(path: string): boolean {
    if (this.config.excludePaths?.some(p => path.startsWith(p))) {
      return false;
    }
    if (this.config.targetPaths?.length) {
      return this.config.targetPaths.some(p => path.startsWith(p));
    }
    return true;
  }

  private spikeCPU(durationMs: number): void {
    const end = Date.now() + durationMs;
    while (Date.now() < end) {
      // Busy loop
      Math.random() * Math.random();
    }
  }

  private pressureMemory(sizeMB: number): void {
    const arr = new Array(sizeMB * 1024 * 256).fill(0);
    setTimeout(() => arr.length = 0, 5000); // Release after 5s
  }

  updateConfig(newConfig: Partial<ChaosConfig>): void {
    this.config = { ...this.config, ...newConfig };
  }
}
```

---

## 10. Debugging Checklist และ Runbook

```typescript
// src/debugging/runbook.ts

/**
 * RUNBOOK: Debug High Latency Issue
 * 
 * 1. ตรวจสอบ error rate: Kibana -> Metrics -> Error Rate
 * 2. ตรวจสอบ trace: Jaeger -> Search -> Service = affected-service
 * 3. ดู slow spans ใน trace
 * 4. ตรวจสอบ database queries ใน trace
 * 5. ตรวจสอบ resource usage: Kubernetes -> pod metrics
 */

export class DebuggingRunbook {
  async diagnoseHighLatency(serviceName: string, correlationId?: string) {
    const steps: Array<{ step: string; action: string; result?: any }> = [];
    
    // Step 1: Check recent errors
    const errorRate = await this.metricsService.getErrorRate(serviceName, '5m');
    steps.push({
      step: '1. Error Rate Check',
      action: `Query error rate for ${serviceName} in last 5 minutes`,
      result: `${errorRate.toFixed(2)}% error rate`,
    });
    
    // Step 2: Check P99 latency
    const p99Latency = await this.metricsService.getP99Latency(serviceName, '5m');
    steps.push({
      step: '2. Latency Check',
      action: `Query P99 latency for ${serviceName}`,
      result: `${p99Latency}ms P99 latency`,
    });
    
    // Step 3: Find slow traces
    const slowTraces = await this.tracingService.findSlowTraces(serviceName, {
      minDuration: 5000,
      limit: 10,
    });
    steps.push({
      step: '3. Slow Traces',
      action: 'Find traces slower than 5 seconds',
      result: `Found ${slowTraces.length} slow traces`,
    });
    
    // Step 4: Check database
    const slowQueries = await this.dbAnalyzer.getSlowQueryReport();
    steps.push({
      step: '4. Database Analysis',
      action: 'Check slow queries',
      result: slowQueries.stats,
    });
    
    return steps;
  }
}
```

---

## สรุปท้ายบท

| เครื่องมือ | ใช้สำหรับ | Setup Complexity |
|-----------|---------|-----------------|
| Correlation IDs | ติดตาม request ข้าม services | ต่ำ |
| Node.js Inspector | Debug code ตรงๆ | ต่ำ |
| ELK Stack | Log aggregation & search | กลาง |
| OpenTelemetry + Jaeger | Distributed tracing | กลาง |
| Memory Monitor | Detect memory leaks | ต่ำ |
| CPU Profiler | Find performance bottlenecks | ต่ำ |
| Query Analyzer | Database optimization | กลาง |
| Network Debugger | Service connectivity | ต่ำ |
| Chaos Monkey | Test resilience | กลาง |

### Debug Workflow

1. **Error Reported** -> ดู Correlation ID จาก response
2. **Search Logs** -> Kibana: `correlationId:"xxx"`
3. **View Trace** -> Jaeger: Search by traceId
4. **Identify Bottleneck** -> Slow span ใน trace
5. **Deep Dive** -> Remote debug หรือ heap snapshot
6. **Fix & Verify** -> Deploy + monitor metrics

---

## 11. Advanced Logging Patterns

### 11.1 Structured Logging Best Practices

```typescript
// src/utils/structured-logger.ts
import winston from 'winston';

// Log levels ตาม severity
const LOG_LEVELS = {
  error: 0,   // System errors ที่ต้องการ immediate action
  warn: 1,    // สถานการณ์ที่อาจเป็นปัญหา
  info: 2,    // Business events ที่สำคัญ
  http: 3,    // HTTP requests
  debug: 4,   // Debug information
};

// สิ่งที่ควร log เสมอ
interface StandardLogFields {
  // Identity
  service: string;
  version: string;
  environment: string;
  hostname: string;

  // Request context
  correlationId: string;
  requestId?: string;
  userId?: string;
  sessionId?: string;

  // Timing
  timestamp: string;
  duration?: number;

  // Business context
  action?: string;
  resource?: string;
  resourceId?: string;

  // Error
  error?: {
    name: string;
    message: string;
    stack?: string;
    code?: string;
  };
}

// ตัวอย่าง structured log entries
const EXAMPLE_LOGS = {
  httpRequest: {
    level: 'http',
    message: 'HTTP Request',
    method: 'GET',
    path: '/api/v1/users/123',
    statusCode: 200,
    duration: 45,
    userAgent: 'Mozilla/5.0',
    correlationId: 'abc-123',
    service: 'user-service',
  },

  businessEvent: {
    level: 'info',
    message: 'User created',
    action: 'USER_CREATED',
    userId: '550e8400-e29b-41d4-a716-446655440000',
    email: 'j***@example.com',  // Masked PII
    correlationId: 'abc-123',
    service: 'user-service',
  },

  securityEvent: {
    level: 'warn',
    message: 'Failed login attempt',
    action: 'LOGIN_FAILED',
    email: 'j***@example.com',
    ipAddress: '192.168.1.xxx',   // Partially masked
    attemptCount: 3,
    correlationId: 'def-456',
    service: 'auth-service',
  },

  error: {
    level: 'error',
    message: 'Database connection failed',
    error: {
      name: 'ConnectionError',
      message: 'ECONNREFUSED 127.0.0.1:5432',
      code: 'DB_CONNECTION_FAILED',
    },
    correlationId: 'ghi-789',
    service: 'user-service',
  },
};

// Log sampling สำหรับ high-traffic endpoints
export class LogSampler {
  private readonly samplingRates: Record<string, number> = {
    'GET /api/v1/health': 0.01,    // Log เพียง 1%
    'GET /api/v1/metrics': 0.05,   // Log 5%
    'POST /api/v1/orders': 1.0,    // Log ทั้งหมด
    'default': 0.1,                // Default: 10%
  };

  shouldLog(path: string, level: string): boolean {
    // Error ต้อง log เสมอ
    if (level === 'error' || level === 'warn') return true;

    const rate = this.samplingRates[path] || this.samplingRates['default'];
    return Math.random() < rate;
  }
}
```

### 11.2 Log-Based Alerting

```typescript
// src/monitoring/log-alerter.ts
// ใช้ Elasticsearch Watcher หรือ Kibana Alerting

const KIBANA_ALERT_CONFIG = `
# config/kibana-alerts/error-rate.json
{
  "name": "High Error Rate Alert",
  "schedule": { "interval": "1m" },
  "consumer": "alerts",
  "rule_type_id": "metrics.alert.threshold",
  "params": {
    "criteria": [
      {
        "aggType": "count",
        "comparator": ">",
        "threshold": [100],
        "timeSize": 5,
        "timeUnit": "m",
        "metric": "error",
        "filterQuery": "level:error"
      }
    ],
    "sourceId": "default",
    "alertOnNoData": false
  },
  "actions": [
    {
      "id": "slack-connector",
      "group": "threshold met",
      "params": {
        "message": "High error rate detected: {{context.reason}}"
      }
    }
  ]
}
`;

// Elasticsearch query สำหรับ error analysis
export const ERROR_ANALYSIS_QUERIES = {
  // Top errors ใน 1 ชั่วโมงที่ผ่านมา
  topErrors: {
    query: {
      bool: {
        filter: [
          { term: { level: 'error' } },
          { range: { '@timestamp': { gte: 'now-1h' } } }
        ]
      }
    },
    aggs: {
      byError: {
        terms: {
          field: 'error.code.keyword',
          size: 10
        },
        aggs: {
          byService: {
            terms: { field: 'service.keyword' }
          }
        }
      }
    }
  },

  // Error rate per service
  errorRateByService: {
    query: {
      range: { '@timestamp': { gte: 'now-1h' } }
    },
    aggs: {
      byService: {
        terms: { field: 'service.keyword' },
        aggs: {
          totalRequests: { value_count: { field: 'correlationId.keyword' } },
          errors: {
            filter: { term: { level: 'error' } },
            aggs: {
              count: { value_count: { field: 'correlationId.keyword' } }
            }
          }
        }
      }
    }
  }
};
```

---

## 12. Production Debugging Tips

### 12.1 Safe Production Debug Techniques

```typescript
// src/debugging/production-debug.ts

/**
 * เทคนิค debug ที่ปลอดภัยใน production:
 * 1. Dynamic log level - เพิ่ม log level ชั่วคราว
 * 2. Sampling - เพิ่ม sampling rate ชั่วคราว
 * 3. Feature flags - เปิด debug mode สำหรับ user ที่เฉพาะ
 * 4. Request tagging - เพิ่ม detail สำหรับ specific requests
 */

import { Injectable } from '@nestjs/common';
import Redis from 'ioredis';

interface DebugConfig {
  userId?: string;
  correlationIdPattern?: string;
  servicePattern?: string;
  logLevel: 'debug' | 'info' | 'warn' | 'error';
  expiresAt: Date;
  enabledBy: string;
}

@Injectable()
export class ProductionDebugService {
  constructor(private redis: Redis) {}

  // เปิด debug mode สำหรับ user ที่เฉพาะ
  async enableUserDebug(
    userId: string,
    durationMinutes: number = 30,
    enabledBy: string
  ): Promise<void> {
    const config: DebugConfig = {
      userId,
      logLevel: 'debug',
      expiresAt: new Date(Date.now() + durationMinutes * 60 * 1000),
      enabledBy,
    };

    await this.redis.setex(
      `debug:user:${userId}`,
      durationMinutes * 60,
      JSON.stringify(config)
    );

    console.warn({
      message: 'Debug mode enabled for user',
      userId,
      durationMinutes,
      enabledBy,
    });
  }

  // ตรวจสอบว่า request นี้ควร debug หรือไม่
  async shouldDebug(userId?: string, correlationId?: string): Promise<boolean> {
    if (userId) {
      const config = await this.redis.get(`debug:user:${userId}`);
      if (config) return true;
    }

    if (correlationId) {
      const config = await this.redis.get(`debug:correlation:${correlationId}`);
      if (config) return true;
    }

    return false;
  }

  // Debug middleware
  debugMiddleware() {
    return async (req: any, res: any, next: any) => {
      const userId = req.user?.id;
      const correlationId = req.headers['x-correlation-id'];

      if (await this.shouldDebug(userId, correlationId)) {
        req.debugMode = true;
        req.debugStartTime = Date.now();

        // เพิ่ม detailed logging
        console.debug({
          message: 'DEBUG REQUEST',
          method: req.method,
          path: req.path,
          headers: req.headers,
          body: req.body,
          userId,
          correlationId,
        });
      }

      next();
    };
  }
}

// k6 debug script
export const K6_DEBUG_SCRIPT = `
import http from 'k6/http';
import { check } from 'k6';

export const options = {
  vus: 1,
  duration: '30s',
};

export default function() {
  const correlationId = 'debug-k6-' + __ITER;
  
  const res = http.get('http://localhost:3000/api/v1/users', {
    headers: {
      'X-Correlation-ID': correlationId,
      'X-Debug-Token': __ENV.DEBUG_TOKEN,
    },
  });
  
  check(res, {
    'status is 200': (r) => r.status === 200,
    'has correlation id': (r) => r.headers['X-Correlation-ID'] !== undefined,
  });
  
  console.log('Correlation ID:', correlationId);
  console.log('Response Time:', res.timings.duration, 'ms');
  console.log('Response:', res.body.substring(0, 200));
}
`;
```

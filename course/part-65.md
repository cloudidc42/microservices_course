# Part 65: Microservices Governance

## บทนำ

Microservices Governance คือชุดของ Policies, Processes และ Standards ที่ช่วยให้
องค์กรสามารถพัฒนา Microservices ได้อย่างมีระเบียบและสม่ำเสมอ บทนี้ครอบคลุมตั้งแต่
Contract Testing ไปจนถึง OPA Policy Automation

---

## 1. Consumer-Driven Contract Testing with Pact

### หลักการทำงาน

```
Consumer (Order Service) กำหนด contract ที่ต้องการจาก Provider (User Service)
Provider ต้องทำให้ contract นั้น pass

Flow:
1. Consumer เขียน test ที่ระบุ expectations
2. Pact สร้าง contract file (JSON)
3. Contract ถูก publish ไปยัง Pact Broker
4. Provider ดึง contract และ verify ว่า implementation ตรงตาม contract
```

### Consumer Test

```typescript
// tests/pacts/order-user.consumer.pact.spec.ts
import { Pact, Matchers } from '@pact-foundation/pact';
import path from 'path';
import axios from 'axios';

const { like, term, eachLike } = Matchers;

describe('Order Service → User Service Contract', () => {
  const provider = new Pact({
    consumer: 'OrderService',
    provider: 'UserService',
    port: 8080,
    log: path.resolve(__dirname, '../../logs', 'pact.log'),
    dir: path.resolve(__dirname, '../../pacts'),
    logLevel: 'warn',
  });

  beforeAll(() => provider.setup());
  afterAll(() => provider.finalize());
  afterEach(() => provider.verify());

  describe('GET /users/:id', () => {
    it('should return user details for a valid user ID', async () => {
      const userId = '550e8400-e29b-41d4-a716-446655440000';

      await provider.addInteraction({
        state: `user ${userId} exists`,
        uponReceiving: 'a request to get user by ID',
        withRequest: {
          method: 'GET',
          path: `/users/${userId}`,
          headers: {
            Accept: 'application/json',
            Authorization: term({
              generate: 'Bearer valid-token',
              matcher: 'Bearer .+',
            }),
          },
        },
        willRespondWith: {
          status: 200,
          headers: {
            'Content-Type': 'application/json; charset=utf-8',
          },
          body: {
            id: like(userId),
            email: term({
              generate: 'user@example.com',
              matcher: '^[^@]+@[^@]+\\.[^@]+$',
            }),
            name: like('John Doe'),
            tier: term({
              generate: 'standard',
              matcher: '^(standard|premium|vip)$',
            }),
            createdAt: like('2024-01-01T00:00:00.000Z'),
          },
        },
      });

      // Actual service call
      const response = await axios.get(
        `http://localhost:8080/users/${userId}`,
        {
          headers: {
            Accept: 'application/json',
            Authorization: 'Bearer valid-token',
          },
        }
      );

      expect(response.status).toBe(200);
      expect(response.data.id).toBe(userId);
      expect(response.data.email).toMatch(/^[^@]+@[^@]+\.[^@]+$/);
    });

    it('should return 404 for non-existent user', async () => {
      const nonExistentId = '00000000-0000-0000-0000-000000000000';

      await provider.addInteraction({
        state: `user ${nonExistentId} does not exist`,
        uponReceiving: 'a request to get a non-existent user',
        withRequest: {
          method: 'GET',
          path: `/users/${nonExistentId}`,
          headers: {
            Accept: 'application/json',
            Authorization: term({
              generate: 'Bearer valid-token',
              matcher: 'Bearer .+',
            }),
          },
        },
        willRespondWith: {
          status: 404,
          body: {
            error: like('User not found'),
            code: like('USER_NOT_FOUND'),
          },
        },
      });

      try {
        await axios.get(
          `http://localhost:8080/users/${nonExistentId}`,
          { headers: { Authorization: 'Bearer valid-token' } }
        );
        fail('Should have thrown');
      } catch (error: unknown) {
        const axiosError = error as { response?: { status?: number } };
        expect(axiosError.response?.status).toBe(404);
      }
    });
  });

  describe('POST /users/:id/orders', () => {
    it('should add order to user history', async () => {
      const userId = '550e8400-e29b-41d4-a716-446655440000';

      await provider.addInteraction({
        state: `user ${userId} exists`,
        uponReceiving: 'a request to add order to user history',
        withRequest: {
          method: 'POST',
          path: `/users/${userId}/orders`,
          headers: {
            'Content-Type': 'application/json',
            Authorization: term({
              generate: 'Bearer valid-token',
              matcher: 'Bearer .+',
            }),
          },
          body: {
            orderId: like('order-123'),
            amount: like(100.50),
            currency: like('THB'),
          },
        },
        willRespondWith: {
          status: 201,
          body: {
            success: like(true),
            orderHistoryCount: like(5),
          },
        },
      });

      const response = await axios.post(
        `http://localhost:8080/users/${userId}/orders`,
        { orderId: 'order-123', amount: 100.50, currency: 'THB' },
        { headers: { Authorization: 'Bearer valid-token' } }
      );

      expect(response.status).toBe(201);
      expect(response.data.success).toBe(true);
    });
  });
});
```

### Provider Verification

```typescript
// tests/pacts/user.provider.pact.spec.ts
import { Verifier } from '@pact-foundation/pact';
import path from 'path';
import { NestFactory } from '@nestjs/core';
import { AppModule } from '../../src/app.module';

describe('User Service Provider Verification', () => {
  let app: ReturnType<typeof NestFactory.create> extends Promise<infer T> ? T : never;
  let server: unknown;

  beforeAll(async () => {
    const application = await NestFactory.create(AppModule, { logger: false });
    await application.listen(8081);
    app = application as unknown as typeof app;
    server = application.getHttpServer();
  });

  afterAll(async () => {
    if (app) await (app as { close: () => Promise<void> }).close();
  });

  it('should validate contract with OrderService consumer', async () => {
    const verifier = new Verifier({
      provider: 'UserService',
      providerBaseUrl: 'http://localhost:8081',

      // ดึง pacts จาก Pact Broker
      pactBrokerUrl: process.env.PACT_BROKER_URL || 'http://pact-broker:9292',
      pactBrokerToken: process.env.PACT_BROKER_TOKEN,

      // หรือ ดึงจาก local file
      pactUrls: process.env.PACT_BROKER_URL 
        ? undefined 
        : [path.resolve(__dirname, '../../pacts/OrderService-UserService.json')],

      // State handlers
      stateHandlers: {
        [`user 550e8400-e29b-41d4-a716-446655440000 exists`]: async () => {
          // Setup test data
          await setupUser('550e8400-e29b-41d4-a716-446655440000');
        },
        [`user 00000000-0000-0000-0000-000000000000 does not exist`]: async () => {
          // Ensure user doesn't exist
          await deleteUser('00000000-0000-0000-0000-000000000000');
        },
      },

      publishVerificationResult: process.env.CI === 'true',
      providerVersion: process.env.APP_VERSION || '1.0.0',
      providerVersionTags: [process.env.GIT_BRANCH || 'main'],
    });

    await verifier.verifyProvider();
  });
});

async function setupUser(userId: string): Promise<void> {
  // Insert test user into database
}

async function deleteUser(userId: string): Promise<void> {
  // Delete test user from database
}
```

---

## 2. API Versioning Strategies

```typescript
// src/versioning/api-versioning.controller.ts

// Strategy 1: Path Versioning (/v1/orders, /v2/orders)
import { Controller, Get, Version } from '@nestjs/common';

@Controller('orders')
export class OrdersControllerV1 {
  @Version('1')
  @Get()
  findAll() {
    return { version: 'v1', data: [] };
  }
}

@Controller('orders')
export class OrdersControllerV2 {
  @Version('2')
  @Get()
  findAll() {
    // V2 returns different format
    return { 
      version: 'v2', 
      items: [],
      metadata: { total: 0, page: 1 }
    };
  }
}
```

```typescript
// src/main.ts - Setup versioning
import { NestFactory } from '@nestjs/core';
import { VersioningType } from '@nestjs/common';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Strategy 1: URI versioning
  app.enableVersioning({
    type: VersioningType.URI,
    prefix: 'v',
    defaultVersion: '1',
  });

  // Strategy 2: Header versioning
  // app.enableVersioning({
  //   type: VersioningType.HEADER,
  //   header: 'X-API-Version',
  //   defaultVersion: '1',
  // });

  // Strategy 3: Media Type versioning
  // app.enableVersioning({
  //   type: VersioningType.MEDIA_TYPE,
  //   key: 'v=',
  //   defaultVersion: '1',
  // });

  await app.listen(3000);
}

bootstrap();
```

### Version Deprecation Middleware

```typescript
// src/versioning/deprecation.middleware.ts
import { Injectable, NestMiddleware, Logger } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';

interface VersionInfo {
  deprecated: boolean;
  sunsetDate?: string;
  migrationGuide?: string;
  successorVersion?: string;
}

@Injectable()
export class DeprecationMiddleware implements NestMiddleware {
  private readonly logger = new Logger(DeprecationMiddleware.name);

  private readonly versionInfo: Record<string, VersionInfo> = {
    'v1': {
      deprecated: true,
      sunsetDate: '2025-01-01',
      successorVersion: 'v2',
      migrationGuide: 'https://docs.company.com/api/migration/v1-to-v2',
    },
    'v2': {
      deprecated: false,
    },
    'v3': {
      deprecated: false,
    },
  };

  use(req: Request, res: Response, next: NextFunction): void {
    const version = this.extractVersion(req.path);

    if (version && this.versionInfo[version]) {
      const info = this.versionInfo[version];

      if (info.deprecated) {
        res.setHeader('Deprecation', 'true');
        res.setHeader('Sunset', info.sunsetDate || 'TBD');

        if (info.migrationGuide) {
          res.setHeader('Link', `<${info.migrationGuide}>; rel="deprecation"`);
        }

        if (info.successorVersion) {
          const successorUrl = req.path.replace(version, info.successorVersion);
          res.setHeader('Link', `<${successorUrl}>; rel="successor-version"`);
        }

        this.logger.warn(
          `Deprecated API version ${version} called from ${req.ip}: ${req.method} ${req.path}`
        );
      }
    }

    next();
  }

  private extractVersion(path: string): string | null {
    const match = path.match(/^\/?(v\d+)\//);
    return match ? match[1] : null;
  }
}
```

---

## 3. Architecture Decision Records (ADR)

```markdown
<!-- docs/adr/ADR-001-use-rabbitmq-for-messaging.md -->
# ADR-001: Use RabbitMQ for Asynchronous Messaging

## Status
Accepted

## Date
2024-01-15

## Context
เราต้องการ Message Broker สำหรับ async communication ระหว่าง microservices
Options ที่พิจารณา:
- Apache Kafka
- RabbitMQ
- AWS SQS
- Redis Streams

## Decision
เลือก RabbitMQ เนื่องจาก:
1. Team มีประสบการณ์กับ AMQP protocol
2. Supports complex routing patterns (topic, headers exchanges)
3. Management UI ใช้งานง่าย
4. Community support ดี

## Consequences
### Positive
- ง่ายต่อการ setup และ maintain
- Flexible routing
- Good monitoring ผ่าน Prometheus

### Negative
- ไม่ suitable สำหรับ event sourcing หรือ log aggregation
- ต่ำกว่า Kafka ในแง่ throughput
- ไม่มี built-in log retention

## Alternatives Considered
- Kafka: throughput สูงกว่า แต่ complex เกินไปสำหรับ current scale
- AWS SQS: ง่ายกว่า แต่ vendor lock-in
```

---

## 4. Service Level Objectives (SLO)

```typescript
// src/slo/slo-monitor.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { Gauge, Counter, Histogram } from 'prom-client';

interface SLO {
  name: string;
  target: number;    // เช่น 0.999 = 99.9%
  window: string;    // เช่น '30d'
}

@Injectable()
export class SLOMonitorService {
  private readonly logger = new Logger(SLOMonitorService.name);

  private readonly sloCompliance = new Gauge({
    name: 'slo_compliance_ratio',
    help: 'Current SLO compliance ratio',
    labelNames: ['slo_name', 'service'],
  });

  private readonly errorBudgetRemaining = new Gauge({
    name: 'slo_error_budget_remaining_seconds',
    help: 'Remaining error budget in seconds',
    labelNames: ['slo_name', 'service'],
  });

  private readonly slos: SLO[] = [
    { name: 'availability', target: 0.999, window: '30d' },  // 99.9% uptime
    { name: 'latency_p99', target: 0.99, window: '24h' },   // 99% requests < 500ms
    { name: 'error_rate', target: 0.999, window: '24h' },   // < 0.1% errors
  ];

  async calculateSLOCompliance(
    service: string,
    sloName: string,
    metrics: { good: number; total: number }
  ): Promise<number> {
    const compliance = metrics.total > 0 ? metrics.good / metrics.total : 1;
    
    this.sloCompliance.set({ slo_name: sloName, service }, compliance);
    
    const slo = this.slos.find(s => s.name === sloName);
    if (slo) {
      const windowSeconds = this.parseWindow(slo.window);
      const errorBudget = (1 - slo.target) * windowSeconds;
      const consumedBudget = (1 - compliance) * windowSeconds;
      const remaining = Math.max(0, errorBudget - consumedBudget);
      
      this.errorBudgetRemaining.set({ slo_name: sloName, service }, remaining);
      
      if (remaining < errorBudget * 0.1) {
        this.logger.warn(
          `Low error budget for ${service} ${sloName}: ${remaining.toFixed(0)}s remaining`
        );
      }
    }
    
    return compliance;
  }

  private parseWindow(window: string): number {
    const match = window.match(/^(\d+)([hd])$/);
    if (!match) return 86400;
    
    const value = parseInt(match[1]);
    const unit = match[2];
    
    return unit === 'h' ? value * 3600 : value * 86400;
  }
}
```

---

## 5. OPA Policy Automation

### OPA Installation และ Setup

```yaml
# opa-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: opa
  namespace: opa-system
spec:
  replicas: 2
  selector:
    matchLabels:
      app: opa
  template:
    metadata:
      labels:
        app: opa
    spec:
      containers:
        - name: opa
          image: openpolicyagent/opa:0.58.0
          args:
            - "run"
            - "--server"
            - "--addr=:8181"
            - "--bundle"
            - "/bundles"
          ports:
            - containerPort: 8181
          volumeMounts:
            - name: policy-bundles
              mountPath: /bundles
      volumes:
        - name: policy-bundles
          configMap:
            name: opa-policies
---
apiVersion: v1
kind: Service
metadata:
  name: opa
  namespace: opa-system
spec:
  ports:
    - port: 8181
      targetPort: 8181
  selector:
    app: opa
```

### OPA Policies (Rego)

```rego
# policies/api-governance.rego
package api.governance

import future.keywords.if
import future.keywords.in

# ตรวจสอบว่า API มี versioning หรือไม่
deny[msg] if {
  input.path[0] != "v1"
  input.path[0] != "v2"
  input.path[0] != "v3"
  not startswith(input.path[0], "health")
  not startswith(input.path[0], "metrics")
  
  msg := sprintf("API path '%v' must include version prefix (v1, v2, v3)", [concat("/", input.path)])
}

# ตรวจสอบว่า response มี required headers
warn[msg] if {
  not input.response.headers["X-Request-ID"]
  msg := "Response should include X-Request-ID header"
}

# ตรวจสอบว่า endpoint ที่ต้องการ auth มี Authorization header
deny[msg] if {
  input.path[1] in {"orders", "payments", "users"}
  input.method in {"POST", "PUT", "DELETE", "PATCH"}
  not input.headers["Authorization"]
  
  msg := sprintf("Protected endpoint %v %v requires Authorization header", 
    [input.method, concat("/", input.path)])
}

# Rate limit policy
deny[msg] if {
  input.headers["X-RateLimit-Remaining"]
  to_number(input.headers["X-RateLimit-Remaining"]) < 0
  
  msg := "Rate limit exceeded"
}
```

```rego
# policies/microservice-standards.rego
package microservice.standards

# ตรวจสอบ Kubernetes deployment standards
deny[msg] if {
  input.kind == "Deployment"
  not input.spec.template.spec.containers[_].resources.requests.memory
  
  msg := sprintf("Deployment '%v' must define memory requests", [input.metadata.name])
}

deny[msg] if {
  input.kind == "Deployment"
  not input.spec.template.spec.containers[_].resources.limits.memory
  
  msg := sprintf("Deployment '%v' must define memory limits", [input.metadata.name])
}

deny[msg] if {
  input.kind == "Deployment"
  not input.spec.template.spec.containers[_].livenessProbe
  
  msg := sprintf("Deployment '%v' must define liveness probe", [input.metadata.name])
}

deny[msg] if {
  input.kind == "Deployment"
  not input.spec.template.spec.containers[_].readinessProbe
  
  msg := sprintf("Deployment '%v' must define readiness probe", [input.metadata.name])
}

# ตรวจสอบว่าไม่ใช้ latest tag
deny[msg] if {
  input.kind == "Deployment"
  container := input.spec.template.spec.containers[_]
  endswith(container.image, ":latest")
  
  msg := sprintf("Container '%v' must not use ':latest' tag", [container.name])
}

# ตรวจสอบ Secret management - ไม่ควร hardcode secrets ใน env vars
deny[msg] if {
  input.kind == "Deployment"
  container := input.spec.template.spec.containers[_]
  env := container.env[_]
  upper(env.name) in {"PASSWORD", "SECRET", "KEY", "TOKEN"}
  env.value  # มี value โดยตรง (ไม่ใช้ secretKeyRef)
  
  msg := sprintf("Container '%v' has hardcoded sensitive env var '%v'", 
    [container.name, env.name])
}
```

### OPA Client ใน TypeScript

```typescript
// src/governance/opa.client.ts
import { Injectable, Logger } from '@nestjs/common';
import axios, { AxiosInstance } from 'axios';

interface OPAResult<T = unknown> {
  result: T;
}

interface PolicyViolation {
  message: string;
  severity: 'deny' | 'warn';
}

@Injectable()
export class OPAClient {
  private readonly logger = new Logger(OPAClient.name);
  private readonly client: AxiosInstance;

  constructor() {
    this.client = axios.create({
      baseURL: process.env.OPA_URL || 'http://opa.opa-system:8181',
      timeout: 5000,
      headers: {
        'Content-Type': 'application/json',
      },
    });
  }

  async evaluate<T = unknown>(
    policy: string,
    input: unknown
  ): Promise<OPAResult<T>> {
    const response = await this.client.post<OPAResult<T>>(
      `/v1/data/${policy.replace(/\./g, '/')}`,
      { input }
    );

    return response.data;
  }

  async checkAPIGovernance(request: {
    method: string;
    path: string[];
    headers: Record<string, string>;
  }): Promise<PolicyViolation[]> {
    const violations: PolicyViolation[] = [];

    try {
      const denyResult = await this.evaluate<string[]>(
        'api.governance.deny',
        request
      );

      for (const msg of (denyResult.result || [])) {
        violations.push({ message: msg, severity: 'deny' });
      }

      const warnResult = await this.evaluate<string[]>(
        'api.governance.warn',
        request
      );

      for (const msg of (warnResult.result || [])) {
        violations.push({ message: msg, severity: 'warn' });
        this.logger.warn(`Governance warning: ${msg}`);
      }
    } catch (error) {
      this.logger.error('OPA evaluation failed:', error);
    }

    return violations;
  }

  async validateKubernetesManifest(manifest: unknown): Promise<{
    valid: boolean;
    violations: PolicyViolation[];
  }> {
    const violations: PolicyViolation[] = [];

    try {
      const result = await this.evaluate<string[]>(
        'microservice.standards.deny',
        manifest
      );

      for (const msg of (result.result || [])) {
        violations.push({ message: msg, severity: 'deny' });
      }
    } catch (error) {
      this.logger.error('Failed to validate manifest:', error);
    }

    return {
      valid: violations.filter(v => v.severity === 'deny').length === 0,
      violations,
    };
  }
}
```

---

## 6. Breaking Change Detection

```typescript
// src/governance/breaking-change-detector.ts
import { Injectable, Logger } from '@nestjs/common';

interface APISchema {
  version: string;
  paths: Record<string, PathDefinition>;
  components: {
    schemas: Record<string, SchemaDefinition>;
  };
}

interface PathDefinition {
  [method: string]: {
    parameters?: Parameter[];
    requestBody?: { required: boolean; content: Record<string, unknown> };
    responses: Record<string, unknown>;
  };
}

interface Parameter {
  name: string;
  in: string;
  required?: boolean;
  schema: SchemaDefinition;
}

interface SchemaDefinition {
  type?: string;
  properties?: Record<string, SchemaDefinition>;
  required?: string[];
  enum?: unknown[];
}

interface BreakingChange {
  type: string;
  path: string;
  description: string;
  severity: 'breaking' | 'warning';
}

@Injectable()
export class BreakingChangeDetector {
  private readonly logger = new Logger(BreakingChangeDetector.name);

  detectBreakingChanges(
    oldSchema: APISchema,
    newSchema: APISchema
  ): BreakingChange[] {
    const changes: BreakingChange[] = [];

    // ตรวจสอบ endpoints ที่ถูกลบ
    for (const path of Object.keys(oldSchema.paths)) {
      if (!newSchema.paths[path]) {
        changes.push({
          type: 'endpoint_removed',
          path,
          description: `Endpoint ${path} was removed`,
          severity: 'breaking',
        });
        continue;
      }

      for (const method of Object.keys(oldSchema.paths[path])) {
        if (!newSchema.paths[path][method]) {
          changes.push({
            type: 'method_removed',
            path: `${method.toUpperCase()} ${path}`,
            description: `HTTP method ${method} removed from ${path}`,
            severity: 'breaking',
          });
        }
      }
    }

    // ตรวจสอบ required fields ที่เพิ่มใหม่
    for (const [path, pathDef] of Object.entries(newSchema.paths)) {
      if (!oldSchema.paths[path]) continue;

      for (const [method, methodDef] of Object.entries(pathDef)) {
        const oldMethodDef = oldSchema.paths[path]?.[method];
        if (!oldMethodDef) continue;

        if (methodDef.parameters) {
          for (const param of methodDef.parameters) {
            if (param.required) {
              const oldParam = oldMethodDef.parameters?.find(
                p => p.name === param.name
              );
              if (!oldParam) {
                changes.push({
                  type: 'required_param_added',
                  path: `${method.toUpperCase()} ${path}`,
                  description: `Required parameter '${param.name}' added to ${method.toUpperCase()} ${path}`,
                  severity: 'breaking',
                });
              }
            }
          }
        }
      }
    }

    return changes;
  }

  async generateChangeReport(
    oldVersion: string,
    newVersion: string,
    changes: BreakingChange[]
  ): Promise<string> {
    const breaking = changes.filter(c => c.severity === 'breaking');
    const warnings = changes.filter(c => c.severity === 'warning');

    let report = `# API Change Report: ${oldVersion} → ${newVersion}\n\n`;

    if (breaking.length > 0) {
      report += `## Breaking Changes (${breaking.length})\n\n`;
      for (const change of breaking) {
        report += `### ${change.type}\n`;
        report += `- **Path:** ${change.path}\n`;
        report += `- **Description:** ${change.description}\n\n`;
      }
    } else {
      report += `## ✓ No Breaking Changes\n\n`;
    }

    if (warnings.length > 0) {
      report += `## Warnings (${warnings.length})\n\n`;
      for (const change of warnings) {
        report += `- ${change.description}\n`;
      }
    }

    return report;
  }
}
```

---

## 7. Service Catalog

```typescript
// src/governance/service-catalog.service.ts
import { Injectable, Logger } from '@nestjs/common';

export interface ServiceEntry {
  name: string;
  version: string;
  team: string;
  description: string;
  repository: string;
  documentation: string;
  endpoints: EndpointEntry[];
  dependencies: DependencyEntry[];
  slos: SLOEntry[];
  contacts: ContactEntry[];
  tags: string[];
  maturityLevel: 1 | 2 | 3 | 4 | 5;
}

export interface EndpointEntry {
  method: string;
  path: string;
  description: string;
  version: string;
  deprecated?: boolean;
}

export interface DependencyEntry {
  service: string;
  version: string;
  type: 'runtime' | 'buildtime';
  critical: boolean;
}

export interface SLOEntry {
  name: string;
  target: number;
  window: string;
}

export interface ContactEntry {
  name: string;
  email: string;
  role: string;
  oncall?: boolean;
}

@Injectable()
export class ServiceCatalogService {
  private readonly logger = new Logger(ServiceCatalogService.name);
  private readonly catalog = new Map<string, ServiceEntry>();

  async registerService(entry: ServiceEntry): Promise<void> {
    this.catalog.set(entry.name, entry);
    this.logger.log(`Service registered: ${entry.name} v${entry.version}`);
  }

  async getService(name: string): Promise<ServiceEntry | undefined> {
    return this.catalog.get(name);
  }

  async getAllServices(): Promise<ServiceEntry[]> {
    return Array.from(this.catalog.values());
  }

  async getServicesByTeam(team: string): Promise<ServiceEntry[]> {
    return Array.from(this.catalog.values())
      .filter(s => s.team === team);
  }

  async getDependencyGraph(): Promise<Map<string, string[]>> {
    const graph = new Map<string, string[]>();

    for (const [name, entry] of this.catalog) {
      const deps = entry.dependencies
        .filter(d => d.type === 'runtime')
        .map(d => d.service);
      graph.set(name, deps);
    }

    return graph;
  }

  async detectCircularDependencies(): Promise<string[][]> {
    const graph = await this.getDependencyGraph();
    const cycles: string[][] = [];

    const visited = new Set<string>();
    const recursionStack = new Set<string>();

    const dfs = (service: string, path: string[]): void => {
      visited.add(service);
      recursionStack.add(service);

      const deps = graph.get(service) || [];
      for (const dep of deps) {
        if (!visited.has(dep)) {
          dfs(dep, [...path, dep]);
        } else if (recursionStack.has(dep)) {
          const cycleStart = path.indexOf(dep);
          if (cycleStart !== -1) {
            cycles.push([...path.slice(cycleStart), dep]);
          }
        }
      }

      recursionStack.delete(service);
    };

    for (const service of graph.keys()) {
      if (!visited.has(service)) {
        dfs(service, [service]);
      }
    }

    return cycles;
  }
}
```

---

## 8. Governance CI/CD Integration

```yaml
# .github/workflows/governance-checks.yml
name: Governance Checks

on:
  pull_request:
    branches: [main, develop]

jobs:
  api-contract-tests:
    name: API Contract Tests (Pact)
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          
      - name: Install dependencies
        run: npm ci
        
      - name: Run consumer contract tests
        run: npm run test:pact:consumer
        
      - name: Publish pacts to broker
        env:
          PACT_BROKER_URL: ${{ secrets.PACT_BROKER_URL }}
          PACT_BROKER_TOKEN: ${{ secrets.PACT_BROKER_TOKEN }}
        run: |
          npx pact-broker publish ./pacts \
            --broker-base-url=$PACT_BROKER_URL \
            --broker-token=$PACT_BROKER_TOKEN \
            --consumer-app-version=${{ github.sha }} \
            --tag=${{ github.head_ref }}

  breaking-change-detection:
    name: Breaking Change Detection
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
          
      - name: Install oasdiff
        run: |
          curl -fsSL https://raw.githubusercontent.com/Tufin/oasdiff/main/install.sh | sh
          
      - name: Check for breaking changes
        run: |
          git show HEAD~1:docs/openapi.yaml > /tmp/old-spec.yaml || \
            echo '{}' > /tmp/old-spec.yaml
          oasdiff breaking /tmp/old-spec.yaml docs/openapi.yaml \
            --format markdown > /tmp/breaking-changes.md
          
      - name: Comment on PR
        if: always()
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const report = fs.readFileSync('/tmp/breaking-changes.md', 'utf8');
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## API Breaking Change Analysis\n\n${report}`
            });

  kubernetes-policy-validation:
    name: Kubernetes Policy Validation (OPA)
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Install conftest
        run: |
          curl -L https://github.com/open-policy-agent/conftest/releases/download/v0.46.0/conftest_0.46.0_linux_x86_64.tar.gz \
            | tar xz
          sudo mv conftest /usr/local/bin/
          
      - name: Validate Kubernetes manifests
        run: |
          conftest test k8s/ \
            --policy policies/microservice-standards.rego \
            --output json > /tmp/policy-results.json
          
      - name: Check for violations
        run: |
          FAILURES=$(cat /tmp/policy-results.json | jq '[.[] | select(.failures | length > 0)] | length')
          if [ "$FAILURES" -gt "0" ]; then
            echo "Policy violations found!"
            cat /tmp/policy-results.json | jq '.[] | select(.failures | length > 0)'
            exit 1
          fi
```

---

## สรุป

| หัวข้อ | เครื่องมือ | ประโยชน์ |
|--------|-----------|---------|
| Contract Testing | Pact | ป้องกัน breaking changes ระหว่าง services |
| API Versioning | NestJS Versioning | รองรับ clients หลาย versions |
| Deprecation Management | Middleware + Headers | แจ้ง clients เกี่ยวกับ deprecation |
| ADR | Markdown + Git | บันทึก Architecture decisions |
| SLO Monitoring | Prometheus + Grafana | วัด service reliability |
| OPA Policies | Rego + conftest | Enforce standards อัตโนมัติ |
| Breaking Change Detection | oasdiff + GitHub Actions | CI/CD gate |
| Service Catalog | Custom Service | Service discovery และ documentation |
| Dependency Graph | Custom Analysis | ตรวจสอบ circular dependencies |
| Governance CI/CD | GitHub Actions | Automate governance checks |

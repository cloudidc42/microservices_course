# Part 65: Microservices Governance

## บทนำ

ใน Part นี้เราจะเรียนรู้ Microservices Governance ซึ่งครอบคลุมทุกแง่มุมของการบริหารจัดการ ecosystem ของ microservices ในองค์กร ตั้งแต่ API Contract Testing, Service Catalog, Architecture Decision Records, Maturity Assessment, Breaking Change Detection, API Versioning, SLO Management, Governance Automation ไปจนถึง Team API Ownership

---

## 1. API Contract Testing ด้วย Pact

Pact เป็น Consumer-Driven Contract Testing framework ที่ช่วยให้ consumer และ provider ตกลงกันเรื่อง API contract โดยไม่ต้องรันทั้งระบบพร้อมกัน

### 1.1 Consumer Test (Order Service)

```typescript
// order-service/src/__tests__/pact/user-service.pact.spec.ts
import { PactV3, MatchersV3 } from '@pact-foundation/pact';
import path from 'path';
import { UserServiceClient } from '../../clients/user-service.client';

const { like, string, integer, eachLike } = MatchersV3;

// กำหนด Pact provider
const provider = new PactV3({
  consumer: 'order-service',
  provider: 'user-service',
  // บันทึก pact file ไปที่ pacts/ directory
  dir: path.resolve(process.cwd(), 'pacts'),
  logLevel: 'warn',
});

describe('Order Service → User Service Contract', () => {
  // ===== Test Case 1: ดึงข้อมูล user ที่มีอยู่ =====
  describe('GET /users/:id', () => {
    it('should return user data for existing user', async () => {
      // กำหนด interaction ที่คาดหวัง
      provider
        .given('user with id "user-123" exists')
        .uponReceiving('a request to get user "user-123"')
        .withRequest({
          method: 'GET',
          path: '/users/user-123',
          headers: {
            Accept: 'application/json',
            Authorization: like('Bearer eyJhbGciOi...'),
          },
        })
        .willRespondWith({
          status: 200,
          headers: {
            'Content-Type': 'application/json',
          },
          body: {
            id: string('user-123'),
            email: string('user@example.com'),
            name: string('John Doe'),
            tier: string('premium'),
            createdAt: string('2024-01-01T00:00:00Z'),
            // order-service ต้องการ fields เหล่านี้เท่านั้น
          },
        });

      await provider.executeTest(async (mockserver) => {
        const client = new UserServiceClient(mockserver.url);
        const user = await client.getUserById('user-123', 'Bearer token');

        expect(user.id).toBe('user-123');
        expect(user.email).toBeDefined();
        expect(user.name).toBeDefined();
        expect(user.tier).toBeDefined();
      });
    });

    it('should return 404 for non-existent user', async () => {
      provider
        .given('user with id "unknown-user" does not exist')
        .uponReceiving('a request to get non-existent user')
        .withRequest({
          method: 'GET',
          path: '/users/unknown-user',
          headers: {
            Accept: 'application/json',
            Authorization: like('Bearer token'),
          },
        })
        .willRespondWith({
          status: 404,
          headers: { 'Content-Type': 'application/json' },
          body: {
            error: string('User not found'),
            code: string('USER_NOT_FOUND'),
          },
        });

      await provider.executeTest(async (mockserver) => {
        const client = new UserServiceClient(mockserver.url);
        await expect(client.getUserById('unknown-user', 'Bearer token'))
          .rejects.toThrow('User not found');
      });
    });
  });

  // ===== Test Case 2: ดึง user พร้อม address =====
  describe('GET /users/:id?include=address', () => {
    it('should return user with shipping address', async () => {
      provider
        .given('user "user-456" has saved address')
        .uponReceiving('a request to get user with address')
        .withRequest({
          method: 'GET',
          path: '/users/user-456',
          query: { include: 'address' },
          headers: { Accept: 'application/json' },
        })
        .willRespondWith({
          status: 200,
          body: {
            id: string('user-456'),
            email: string('buyer@example.com'),
            name: string('Jane Smith'),
            tier: string('standard'),
            address: like({
              street: string('123 Main St'),
              city: string('Bangkok'),
              province: string('Bangkok'),
              postalCode: string('10100'),
              country: string('TH'),
            }),
          },
        });

      await provider.executeTest(async (mockserver) => {
        const client = new UserServiceClient(mockserver.url);
        const user = await client.getUserWithAddress('user-456');
        expect(user.address).toBeDefined();
        expect(user.address?.country).toBe('TH');
      });
    });
  });

  // ===== Test Case 3: ตรวจสอบ user credit limit =====
  describe('GET /users/:id/credit', () => {
    it('should return credit information', async () => {
      provider
        .given('user "user-789" has credit limit')
        .uponReceiving('a request to check credit limit')
        .withRequest({
          method: 'GET',
          path: '/users/user-789/credit',
        })
        .willRespondWith({
          status: 200,
          body: {
            userId: string('user-789'),
            creditLimit: integer(50000),
            availableCredit: integer(35000),
            currency: string('THB'),
          },
        });

      await provider.executeTest(async (mockserver) => {
        const client = new UserServiceClient(mockserver.url);
        const credit = await client.getCreditLimit('user-789');
        expect(credit.creditLimit).toBeGreaterThan(0);
        expect(credit.currency).toBe('THB');
      });
    });
  });
});
```

### 1.2 UserServiceClient ที่ Order Service ใช้

```typescript
// order-service/src/clients/user-service.client.ts
import axios, { AxiosInstance } from 'axios';

interface User {
  id: string;
  email: string;
  name: string;
  tier: string;
  createdAt: string;
  address?: UserAddress;
}

interface UserAddress {
  street: string;
  city: string;
  province: string;
  postalCode: string;
  country: string;
}

interface CreditInfo {
  userId: string;
  creditLimit: number;
  availableCredit: number;
  currency: string;
}

export class UserServiceClient {
  private readonly http: AxiosInstance;

  constructor(baseURL: string) {
    this.http = axios.create({
      baseURL,
      timeout: 5000,
      headers: { Accept: 'application/json' },
    });
  }

  async getUserById(userId: string, authToken: string): Promise<User> {
    const response = await this.http.get<User>(`/users/${userId}`, {
      headers: { Authorization: authToken },
    });
    return response.data;
  }

  async getUserWithAddress(userId: string): Promise<User> {
    const response = await this.http.get<User>(`/users/${userId}`, {
      params: { include: 'address' },
    });
    return response.data;
  }

  async getCreditLimit(userId: string): Promise<CreditInfo> {
    const response = await this.http.get<CreditInfo>(`/users/${userId}/credit`);
    return response.data;
  }
}
```

### 1.3 Provider Verification (User Service)

```typescript
// user-service/src/__tests__/pact/provider.pact.spec.ts
import { Verifier, VerifierOptions } from '@pact-foundation/pact';
import path from 'path';
import { app } from '../../app';
import * as http from 'http';

describe('User Service — Pact Provider Verification', () => {
  let server: http.Server;
  let port: number;

  beforeAll((done) => {
    server = app.listen(0, () => {
      const addr = server.address();
      port = typeof addr === 'object' ? addr!.port : 0;
      console.log(`Provider test server running on port ${port}`);
      done();
    });
  });

  afterAll((done) => {
    server.close(done);
  });

  it('should verify all consumer contracts', async () => {
    const options: VerifierOptions = {
      provider: 'user-service',
      providerBaseUrl: `http://localhost:${port}`,

      // โหลด pact files จาก order-service
      pactUrls: [
        path.resolve(__dirname, '../../../../order-service/pacts/order-service-user-service.json'),
      ],

      // หรือใช้ Pact Broker
      // pactBrokerUrl: process.env.PACT_BROKER_URL,
      // publishVerificationResult: true,
      // providerVersion: process.env.APP_VERSION,

      // State handlers — setup test data ตาม "given" states
      stateHandlers: {
        'user with id "user-123" exists': async () => {
          // seed test data
          await seedUser({
            id: 'user-123',
            email: 'user@example.com',
            name: 'John Doe',
            tier: 'premium',
          });
        },
        'user with id "unknown-user" does not exist': async () => {
          // ลบ user นี้ถ้ามี
          await deleteUser('unknown-user');
        },
        'user "user-456" has saved address': async () => {
          await seedUserWithAddress('user-456');
        },
        'user "user-789" has credit limit': async () => {
          await seedUserWithCredit('user-789', 50000, 35000);
        },
      },

      // Request filter — inject auth headers
      requestFilter: (req, res, next) => {
        // bypass auth verification ใน test
        req.headers['x-test-bypass-auth'] = 'true';
        next();
      },

      logLevel: 'warn',
    };

    await new Verifier(options).verifyProvider();
  }, 60000);
});

// Test data helpers
async function seedUser(data: Partial<{ id: string; email: string; name: string; tier: string }>): Promise<void> {
  // ใส่ข้อมูลลง test database
  console.log(`[Pact State] Seeding user ${data.id}`);
}

async function deleteUser(userId: string): Promise<void> {
  console.log(`[Pact State] Deleting user ${userId}`);
}

async function seedUserWithAddress(userId: string): Promise<void> {
  console.log(`[Pact State] Seeding user ${userId} with address`);
}

async function seedUserWithCredit(userId: string, limit: number, available: number): Promise<void> {
  console.log(`[Pact State] Seeding user ${userId} with credit ${limit}/${available}`);
}
```

---

## 2. Service Catalog Server (Backstage-like)

Service Catalog คือ registry ที่รวบรวมข้อมูล microservices ทั้งหมดในองค์กร ช่วยให้ developer หาข้อมูล services ได้ง่าย

```typescript
// service-catalog/src/registry/service.registry.ts
import express, { Router, Request, Response } from 'express';

interface ServiceEndpoint {
  name: string;
  url: string;
  description: string;
}

interface ServiceDependency {
  serviceId: string;
  type: 'hard' | 'soft';  // hard = critical, soft = optional
  description: string;
}

interface ServiceOwnership {
  team: string;
  teamEmail: string;
  slackChannel: string;
  oncallRotation?: string;
}

interface ServiceSLO {
  availability: number;      // percentage, e.g. 99.9
  latencyP99Ms: number;      // milliseconds
  errorRatePercent: number;  // max error rate
}

interface ServiceEntry {
  id: string;
  name: string;
  description: string;
  version: string;
  status: 'active' | 'deprecated' | 'experimental';
  type: 'api' | 'worker' | 'cron' | 'gateway';
  language: string;
  framework: string;
  ownership: ServiceOwnership;
  endpoints: ServiceEndpoint[];
  dependencies: ServiceDependency[];
  slo: ServiceSLO;
  tags: string[];
  documentation: string;    // URL to docs
  repository: string;       // URL to git repo
  dashboardUrl?: string;    // Grafana dashboard
  alertsUrl?: string;       // AlertManager rules
  createdAt: string;
  updatedAt: string;
}

interface ServiceFilter {
  team?: string;
  status?: string;
  type?: string;
  tag?: string;
}

export class ServiceRegistry {
  private readonly services: Map<string, ServiceEntry> = new Map();

  registerService(entry: ServiceEntry): void {
    const existing = this.services.get(entry.id);
    if (existing && existing.version === entry.version) {
      throw new Error(`Service ${entry.id} version ${entry.version} is already registered`);
    }

    this.services.set(entry.id, {
      ...entry,
      updatedAt: new Date().toISOString(),
      createdAt: existing?.createdAt ?? new Date().toISOString(),
    });

    console.log(`[Catalog] Registered service: ${entry.id} v${entry.version}`);
  }

  getService(serviceId: string): ServiceEntry | undefined {
    return this.services.get(serviceId);
  }

  listServices(filter?: ServiceFilter): ServiceEntry[] {
    let services = Array.from(this.services.values());

    if (filter?.team) {
      services = services.filter(s => s.ownership.team === filter.team);
    }
    if (filter?.status) {
      services = services.filter(s => s.status === filter.status);
    }
    if (filter?.type) {
      services = services.filter(s => s.type === filter.type);
    }
    if (filter?.tag) {
      services = services.filter(s => s.tags.includes(filter.tag!));
    }

    return services.sort((a, b) => a.name.localeCompare(b.name));
  }

  // ค้นหา services ที่ depend บน serviceId ที่ระบุ
  getDependents(serviceId: string): ServiceEntry[] {
    return Array.from(this.services.values()).filter(service =>
      service.dependencies.some(dep => dep.serviceId === serviceId),
    );
  }

  // ตรวจสอบ dependency graph หาวงกลม (circular dependency)
  detectCircularDependencies(): string[][] {
    const cycles: string[][] = [];

    const visited = new Set<string>();
    const recursionStack = new Set<string>();

    const dfs = (serviceId: string, path: string[]): void => {
      visited.add(serviceId);
      recursionStack.add(serviceId);

      const service = this.services.get(serviceId);
      if (!service) return;

      for (const dep of service.dependencies) {
        if (!visited.has(dep.serviceId)) {
          dfs(dep.serviceId, [...path, dep.serviceId]);
        } else if (recursionStack.has(dep.serviceId)) {
          // พบ cycle
          const cycleStart = path.indexOf(dep.serviceId);
          cycles.push(path.slice(cycleStart));
        }
      }

      recursionStack.delete(serviceId);
    };

    for (const serviceId of this.services.keys()) {
      if (!visited.has(serviceId)) {
        dfs(serviceId, [serviceId]);
      }
    }

    return cycles;
  }

  updateServiceStatus(serviceId: string, status: ServiceEntry['status']): void {
    const service = this.services.get(serviceId);
    if (!service) {
      throw new Error(`Service ${serviceId} not found`);
    }
    this.services.set(serviceId, {
      ...service,
      status,
      updatedAt: new Date().toISOString(),
    });
  }
}

// ===== Express Router สำหรับ Service Catalog API =====
export function createServiceCatalogRouter(registry: ServiceRegistry): Router {
  const router = Router();

  // GET /services — list all services
  router.get('/services', (req: Request, res: Response) => {
    const filter: ServiceFilter = {};
    if (req.query.team) filter.team = req.query.team as string;
    if (req.query.status) filter.status = req.query.status as string;
    if (req.query.type) filter.type = req.query.type as string;
    if (req.query.tag) filter.tag = req.query.tag as string;

    const services = registry.listServices(filter);
    res.json({
      total: services.length,
      services,
    });
  });

  // GET /services/:id — get specific service
  router.get('/services/:id', (req: Request, res: Response) => {
    const service = registry.getService(req.params.id);
    if (!service) {
      res.status(404).json({ error: 'Service not found', id: req.params.id });
      return;
    }
    res.json(service);
  });

  // POST /services — register new service
  router.post('/services', (req: Request, res: Response) => {
    try {
      registry.registerService(req.body as ServiceEntry);
      res.status(201).json({ message: 'Service registered', id: req.body.id });
    } catch (error) {
      res.status(400).json({ error: (error as Error).message });
    }
  });

  // GET /services/:id/dependents — services that depend on this service
  router.get('/services/:id/dependents', (req: Request, res: Response) => {
    const dependents = registry.getDependents(req.params.id);
    res.json({ total: dependents.length, dependents });
  });

  // GET /graph/cycles — ตรวจสอบ circular dependencies
  router.get('/graph/cycles', (req: Request, res: Response) => {
    const cycles = registry.detectCircularDependencies();
    res.json({
      hasCycles: cycles.length > 0,
      cycles,
    });
  });

  // PATCH /services/:id/status — update service status
  router.patch('/services/:id/status', (req: Request, res: Response) => {
    try {
      registry.updateServiceStatus(req.params.id, req.body.status);
      res.json({ message: 'Status updated' });
    } catch (error) {
      res.status(404).json({ error: (error as Error).message });
    }
  });

  return router;
}

// ===== ตัวอย่างการ register services =====
export function seedExampleServices(registry: ServiceRegistry): void {
  registry.registerService({
    id: 'order-service',
    name: 'Order Service',
    description: 'Manages order lifecycle from creation to fulfillment',
    version: '2.3.1',
    status: 'active',
    type: 'api',
    language: 'TypeScript',
    framework: 'Express',
    ownership: {
      team: 'commerce-team',
      teamEmail: 'commerce-team@company.com',
      slackChannel: '#commerce-eng',
      oncallRotation: 'order-service-oncall',
    },
    endpoints: [
      { name: 'Create Order', url: '/orders', description: 'POST — create new order' },
      { name: 'Get Order', url: '/orders/:id', description: 'GET — get order by ID' },
      { name: 'List Orders', url: '/orders', description: 'GET — list user orders' },
    ],
    dependencies: [
      { serviceId: 'user-service', type: 'hard', description: 'Get user information' },
      { serviceId: 'inventory-service', type: 'hard', description: 'Check stock' },
      { serviceId: 'payment-service', type: 'hard', description: 'Process payment' },
      { serviceId: 'notification-service', type: 'soft', description: 'Send order confirmations' },
    ],
    slo: {
      availability: 99.9,
      latencyP99Ms: 500,
      errorRatePercent: 0.1,
    },
    tags: ['ecommerce', 'critical-path', 'payment'],
    documentation: 'https://docs.company.com/order-service',
    repository: 'https://github.com/company/order-service',
    dashboardUrl: 'https://grafana.company.com/d/order-service',
    createdAt: '2023-01-15T00:00:00Z',
    updatedAt: new Date().toISOString(),
  });
}
```

---

## 3. Architecture Decision Records (ADR)

ADR คือเอกสารบันทึก Architecture Decisions ที่สำคัญพร้อมเหตุผล ช่วยให้ทีมเข้าใจว่าทำไมถึงตัดสินใจแบบนั้น

### 3.1 ADR Template

```markdown
# ADR-{number}: {Title}

**Date:** {YYYY-MM-DD}
**Status:** {Proposed | Accepted | Deprecated | Superseded by ADR-XXX}
**Deciders:** {list of people involved}
**Tags:** {api-design, security, infrastructure, ...}

## Context

{อธิบาย background และ problem ที่ต้องตัดสินใจ}

## Decision

{อธิบาย decision ที่เลือก}

## Rationale

{อธิบายเหตุผลที่เลือก option นี้}

## Considered Options

| Option | Pros | Cons |
|--------|------|------|
| Option A | ... | ... |
| Option B | ... | ... |

## Consequences

### Positive
- {positive outcome 1}

### Negative
- {negative outcome 1}

### Risks
- {risk 1}

## Implementation Notes

{รายละเอียดการ implement}
```

### 3.2 ADR-001: API Versioning Strategy

```markdown
# ADR-001: API Versioning Strategy

**Date:** 2024-01-10
**Status:** Accepted
**Deciders:** Backend Chapter, Platform Team, API Working Group
**Tags:** api-design, versioning

## Context

เรามี 15 microservices ที่ expose REST APIs โดย consumer ทั้ง internal (mobile apps, frontend) 
และ external (partners, third-party integrations) ต้องการ stability ของ API
เมื่อต้องการเปลี่ยน API เราต้องการ strategy ที่:
1. ไม่ทำให้ existing clients พัง
2. ง่ายต่อการ test และ debug
3. ง่ายต่อการ deprecate versions เก่า

## Decision

ใช้ **URL Path Versioning** เป็น primary strategy: `/v1/`, `/v2/`
และ **Accept Header Versioning** เป็น optional สำหรับ minor versions

## Rationale

URL Path Versioning เลือกเพราะ:
- มองเห็นได้ชัดใน URL — ง่ายต่อการ test ด้วย curl
- Browser และ CDN cache ทำงานได้ดี
- ทีมทุกคนเข้าใจโดยไม่ต้องฝึก
- Logging และ monitoring อ่านง่าย

## Considered Options

| Option | Pros | Cons |
|--------|------|------|
| URL Path (/v1/) | ชัดเจน, cacheable, ง่าย debug | URL ยาวขึ้น, ต้องดูแล multiple routes |
| Accept Header | Clean URL | ซับซ้อน, browser cache ยาก, ยาก test |
| Query Param (?v=1) | ง่าย implement | ไม่ standard, CDN cache ยาก |
| Subdomain (v1.api.) | Domain isolation | Cert management ซับซ้อน |

## Consequences

### Positive
- Clients เลือก version ที่ต้องการได้ชัดเจน
- สามารถ deploy หลาย versions พร้อมกันได้
- Easy A/B testing ระหว่าง versions

### Negative
- Code duplication ระหว่าง versions (แก้ด้วย shared business logic layer)
- ต้องตัดสินใจว่าจะ deprecate version เก่าเมื่อไหร่

### Risks
- Teams อาจไม่ deprecate version เก่าทำให้ต้องดูแลหลาย versions นานเกินไป
  - Mitigation: กำหนด policy ว่า version ต้องถูก deprecated ภายใน 12 เดือนหลัง v.ใหม่ออก

## Implementation Notes

```typescript
// router setup สำหรับ versioned API
app.use('/v1', v1Router);
app.use('/v2', v2Router);

// Deprecation header
app.use('/v1', (req, res, next) => {
  res.set('Deprecation', 'true');
  res.set('Sunset', 'Sat, 01 Jan 2026 00:00:00 GMT');
  next();
});
```
```

### 3.3 ADR-002: Event Broker Choice

```markdown
# ADR-002: Event Broker for Async Communication

**Date:** 2024-02-15
**Status:** Accepted
**Deciders:** Platform Team, Backend Chapter Leads
**Tags:** infrastructure, messaging, event-driven

## Context

เราต้องการ event broker สำหรับ async communication ระหว่าง microservices
Requirements:
- Throughput: 100,000 events/second peak
- Message ordering ต่อ partition
- Message retention อย่างน้อย 7 วัน สำหรับ replay
- At-least-once delivery
- Consumer groups สำหรับ scaling consumers

## Decision

ใช้ **Apache Kafka** เป็น event broker หลัก

## Rationale

Kafka ตอบโจทย์ requirements ทั้งหมด และทีม platform มี expertise อยู่แล้ว

## Considered Options

| Option | Pros | Cons |
|--------|------|------|
| Kafka | High throughput, durable, replay | Complex ops, learning curve |
| RabbitMQ | Familiar, flexible routing | ไม่ log-based, replay ยาก |
| AWS SQS/SNS | Managed, simple | Vendor lock-in, limited retention |
| NATS JetStream | Fast, lightweight | Ecosystem เล็กกว่า |

## Consequences

### Positive
- Event replay สำหรับ debugging และ new consumer bootstrapping
- High throughput รองรับ peak traffic
- Schema Registry (Confluent) สำหรับ schema evolution

### Negative  
- ต้องการ ZooKeeper/KRaft cluster — ops overhead สูงกว่า managed services
- Learning curve สำหรับทีมที่ไม่คุ้นเคย

## Implementation Notes

```yaml
# Kafka topic naming convention
# {domain}.{entity}.{event-type}
# เช่น: commerce.order.created, commerce.order.paid
```
```

### 3.4 ADR-003: Authentication Strategy

```markdown
# ADR-003: Authentication and Authorization Strategy

**Date:** 2024-03-01
**Status:** Accepted
**Deciders:** Security Team, Backend Chapter, Frontend Team
**Tags:** security, authentication, jwt

## Context

ต้องการ auth strategy สำหรับ:
- External APIs (mobile apps, web)
- Internal service-to-service communication
- Admin APIs

## Decision

- **External**: JWT (RS256) ออกโดย Auth Service, verify ที่ API Gateway
- **Internal**: mTLS ระหว่าง services ใน Kubernetes mesh
- **Admin**: JWT + OTP (TOTP)

## Considered Options

| Option | Pros | Cons |
|--------|------|------|
| JWT (stateless) | No DB lookup, scalable | Token revocation ยาก |
| Session (stateful) | Easy revocation | DB lookup ทุก request |
| API Keys | Simple | ยาก rotate, ไม่มี expiry standard |

## Consequences

### Positive
- API Gateway verify JWT โดยไม่ต้องคุย Auth Service ทุก request
- mTLS ให้ service identity ใน mesh

### Negative
- JWT revocation ต้อง short TTL (15 นาที) + refresh token pattern
- mTLS certificate rotation ต้องมี automation

## Implementation Notes

```typescript
// JWT verification middleware
import jwt from 'jsonwebtoken';
import jwksClient from 'jwks-rsa';

const client = jwksClient({
  jwksUri: process.env.JWKS_URI!, // e.g. https://auth.company.com/.well-known/jwks.json
  cache: true,
  cacheMaxAge: 600000, // 10 นาที
});

export async function verifyJWT(token: string): Promise<jwt.JwtPayload> {
  const decoded = jwt.decode(token, { complete: true });
  const kid = decoded?.header?.kid;
  const key = await client.getSigningKey(kid);
  return jwt.verify(token, key.getPublicKey(), { algorithms: ['RS256'] }) as jwt.JwtPayload;
}
```
```

---

## 4. Microservices Maturity Assessment

```typescript
// scripts/maturity-assessment.ts
// ประเมินความ mature ของ microservice ตาม criteria ต่างๆ
interface MaturityCriteria {
  category: string;
  criteria: string;
  weight: number;
  maxScore: number;
}

interface ServiceAssessment {
  serviceName: string;
  assessmentDate: string;
  assessor: string;
  scores: Record<string, number>;
}

interface MaturityResult {
  serviceName: string;
  totalScore: number;
  maxPossibleScore: number;
  percentage: number;
  level: string;
  categoryBreakdown: Record<string, { score: number; max: number; percentage: number }>;
  recommendations: string[];
}

const MATURITY_CRITERIA: MaturityCriteria[] = [
  // === 1. Observability (25 points) ===
  { category: 'Observability', criteria: 'structured_logging', weight: 1, maxScore: 5 },
  { category: 'Observability', criteria: 'distributed_tracing', weight: 1, maxScore: 5 },
  { category: 'Observability', criteria: 'metrics_exposed', weight: 1, maxScore: 5 },
  { category: 'Observability', criteria: 'alerting_configured', weight: 1, maxScore: 5 },
  { category: 'Observability', criteria: 'dashboard_exists', weight: 1, maxScore: 5 },

  // === 2. Reliability (25 points) ===
  { category: 'Reliability', criteria: 'health_checks', weight: 1, maxScore: 5 },
  { category: 'Reliability', criteria: 'graceful_shutdown', weight: 1, maxScore: 5 },
  { category: 'Reliability', criteria: 'circuit_breaker', weight: 1, maxScore: 5 },
  { category: 'Reliability', criteria: 'retry_with_backoff', weight: 1, maxScore: 5 },
  { category: 'Reliability', criteria: 'slo_defined', weight: 1, maxScore: 5 },

  // === 3. Security (20 points) ===
  { category: 'Security', criteria: 'authentication', weight: 1, maxScore: 5 },
  { category: 'Security', criteria: 'secrets_management', weight: 1, maxScore: 5 },
  { category: 'Security', criteria: 'dependency_scanning', weight: 1, maxScore: 5 },
  { category: 'Security', criteria: 'image_scanning', weight: 1, maxScore: 5 },

  // === 4. CI/CD (15 points) ===
  { category: 'CICD', criteria: 'automated_tests', weight: 1, maxScore: 5 },
  { category: 'CICD', criteria: 'contract_tests', weight: 1, maxScore: 5 },
  { category: 'CICD', criteria: 'automated_deployment', weight: 1, maxScore: 5 },

  // === 5. Documentation (15 points) ===
  { category: 'Documentation', criteria: 'openapi_spec', weight: 1, maxScore: 5 },
  { category: 'Documentation', criteria: 'runbook_exists', weight: 1, maxScore: 5 },
  { category: 'Documentation', criteria: 'adr_documented', weight: 1, maxScore: 5 },
];

const SCORE_DESCRIPTIONS: Record<number, string> = {
  0: 'ไม่มีเลย',
  1: 'เริ่มต้น / ยังไม่สมบูรณ์',
  2: 'พื้นฐาน / บาง cases',
  3: 'ส่วนใหญ่ครอบคลุม',
  4: 'ครบถ้วน / มีเอกสาร',
  5: 'ยอดเยี่ยม / best practice',
};

const MATURITY_LEVELS = [
  { level: 'Level 1: Initial', minPercent: 0, maxPercent: 40 },
  { level: 'Level 2: Developing', minPercent: 40, maxPercent: 60 },
  { level: 'Level 3: Defined', minPercent: 60, maxPercent: 75 },
  { level: 'Level 4: Managed', minPercent: 75, maxPercent: 90 },
  { level: 'Level 5: Optimizing', minPercent: 90, maxPercent: 101 },
];

export class MaturityAssessor {
  assess(assessment: ServiceAssessment): MaturityResult {
    let totalScore = 0;
    let maxPossibleScore = 0;
    const categoryScores: Record<string, { score: number; max: number }> = {};
    const recommendations: string[] = [];

    for (const criterion of MATURITY_CRITERIA) {
      const score = assessment.scores[criterion.criteria] ?? 0;
      const validScore = Math.max(0, Math.min(score, criterion.maxScore));

      totalScore += validScore;
      maxPossibleScore += criterion.maxScore;

      if (!categoryScores[criterion.category]) {
        categoryScores[criterion.category] = { score: 0, max: 0 };
      }
      categoryScores[criterion.category].score += validScore;
      categoryScores[criterion.category].max += criterion.maxScore;

      // สร้าง recommendations สำหรับ criteria ที่คะแนนต่ำ
      if (validScore < 3) {
        recommendations.push(
          `[${criterion.category}] ปรับปรุง "${criterion.criteria}": ` +
          `คะแนนปัจจุบัน ${validScore}/5`,
        );
      }
    }

    const percentage = (totalScore / maxPossibleScore) * 100;
    const levelInfo = MATURITY_LEVELS.find(
      l => percentage >= l.minPercent && percentage < l.maxPercent,
    ) ?? MATURITY_LEVELS[0];

    const categoryBreakdown: MaturityResult['categoryBreakdown'] = {};
    for (const [cat, scores] of Object.entries(categoryScores)) {
      categoryBreakdown[cat] = {
        score: scores.score,
        max: scores.max,
        percentage: (scores.score / scores.max) * 100,
      };
    }

    return {
      serviceName: assessment.serviceName,
      totalScore,
      maxPossibleScore,
      percentage: Math.round(percentage * 10) / 10,
      level: levelInfo.level,
      categoryBreakdown,
      recommendations: recommendations.slice(0, 5), // top 5 recommendations
    };
  }

  printReport(result: MaturityResult): void {
    console.log('\n' + '='.repeat(60));
    console.log(`Maturity Assessment: ${result.serviceName}`);
    console.log('='.repeat(60));
    console.log(`Overall Score: ${result.totalScore}/${result.maxPossibleScore} (${result.percentage}%)`);
    console.log(`Maturity Level: ${result.level}`);
    console.log('\nCategory Breakdown:');

    for (const [category, scores] of Object.entries(result.categoryBreakdown)) {
      const bar = '█'.repeat(Math.round(scores.percentage / 10)) +
                  '░'.repeat(10 - Math.round(scores.percentage / 10));
      console.log(`  ${category.padEnd(15)} ${bar} ${scores.percentage.toFixed(0)}%`);
    }

    if (result.recommendations.length > 0) {
      console.log('\nTop Recommendations:');
      result.recommendations.forEach((rec, i) => {
        console.log(`  ${i + 1}. ${rec}`);
      });
    }
    console.log('='.repeat(60) + '\n');
  }
}

// ===== ตัวอย่างการใช้งาน =====
const assessor = new MaturityAssessor();

const orderServiceAssessment: ServiceAssessment = {
  serviceName: 'order-service',
  assessmentDate: new Date().toISOString(),
  assessor: 'Platform Team',
  scores: {
    structured_logging: 5,
    distributed_tracing: 4,
    metrics_exposed: 5,
    alerting_configured: 3,
    dashboard_exists: 4,
    health_checks: 5,
    graceful_shutdown: 5,
    circuit_breaker: 4,
    retry_with_backoff: 4,
    slo_defined: 3,
    authentication: 5,
    secrets_management: 4,
    dependency_scanning: 3,
    image_scanning: 4,
    automated_tests: 4,
    contract_tests: 3,
    automated_deployment: 5,
    openapi_spec: 5,
    runbook_exists: 2,
    adr_documented: 3,
  },
};

const result = assessor.assess(orderServiceAssessment);
assessor.printReport(result);
```

---

## 5. Breaking Change Detection

```typescript
// scripts/breaking-change-detector.ts
// ตรวจหา breaking changes ระหว่าง 2 versions ของ OpenAPI spec
import * as fs from 'fs';
import * as path from 'path';

interface OpenAPISpec {
  openapi: string;
  info: { title: string; version: string };
  paths: Record<string, PathItem>;
  components?: {
    schemas?: Record<string, SchemaObject>;
  };
}

interface PathItem {
  get?: OperationObject;
  post?: OperationObject;
  put?: OperationObject;
  patch?: OperationObject;
  delete?: OperationObject;
}

interface OperationObject {
  operationId?: string;
  parameters?: Parameter[];
  requestBody?: RequestBody;
  responses: Record<string, Response>;
}

interface Parameter {
  name: string;
  in: string;
  required?: boolean;
  schema?: SchemaObject;
}

interface RequestBody {
  required?: boolean;
  content: Record<string, { schema?: SchemaObject }>;
}

interface Response {
  description: string;
  content?: Record<string, { schema?: SchemaObject }>;
}

interface SchemaObject {
  type?: string;
  required?: string[];
  properties?: Record<string, SchemaObject>;
  items?: SchemaObject;
  enum?: unknown[];
  nullable?: boolean;
}

type BreakingChangeSeverity = 'BREAKING' | 'POTENTIALLY_BREAKING' | 'NON_BREAKING';

interface ChangeReport {
  severity: BreakingChangeSeverity;
  path: string;
  description: string;
  before?: string;
  after?: string;
}

export class BreakingChangeDetector {
  detect(oldSpec: OpenAPISpec, newSpec: OpenAPISpec): ChangeReport[] {
    const changes: ChangeReport[] = [];

    // 1. ตรวจ removed endpoints
    for (const [path, oldPathItem] of Object.entries(oldSpec.paths)) {
      if (!newSpec.paths[path]) {
        changes.push({
          severity: 'BREAKING',
          path: `paths.${path}`,
          description: `Endpoint removed: ${path}`,
        });
        continue;
      }

      // 2. ตรวจ removed methods
      const newPathItem = newSpec.paths[path];
      for (const method of ['get', 'post', 'put', 'patch', 'delete'] as const) {
        const oldOp = oldPathItem[method];
        const newOp = newPathItem[method];

        if (oldOp && !newOp) {
          changes.push({
            severity: 'BREAKING',
            path: `paths.${path}.${method}`,
            description: `HTTP method removed: ${method.toUpperCase()} ${path}`,
          });
          continue;
        }

        if (oldOp && newOp) {
          changes.push(...this.compareOperations(path, method, oldOp, newOp));
        }
      }
    }

    // 3. ตรวจ schema changes
    const oldSchemas = oldSpec.components?.schemas ?? {};
    const newSchemas = newSpec.components?.schemas ?? {};

    for (const [schemaName, oldSchema] of Object.entries(oldSchemas)) {
      if (!newSchemas[schemaName]) {
        changes.push({
          severity: 'POTENTIALLY_BREAKING',
          path: `components.schemas.${schemaName}`,
          description: `Schema removed: ${schemaName}`,
        });
        continue;
      }
      changes.push(...this.compareSchemas(
        `components.schemas.${schemaName}`,
        oldSchema,
        newSchemas[schemaName],
      ));
    }

    return changes;
  }

  private compareOperations(
    path: string,
    method: string,
    oldOp: OperationObject,
    newOp: OperationObject,
  ): ChangeReport[] {
    const changes: ChangeReport[] = [];
    const prefix = `paths.${path}.${method}`;

    // ตรวจ required parameters เพิ่มขึ้น
    const oldParams = oldOp.parameters ?? [];
    const newParams = newOp.parameters ?? [];

    for (const newParam of newParams) {
      const oldParam = oldParams.find(p => p.name === newParam.name && p.in === newParam.in);
      if (!oldParam && newParam.required) {
        changes.push({
          severity: 'BREAKING',
          path: `${prefix}.parameters.${newParam.name}`,
          description: `Required parameter added: ${newParam.name} (in: ${newParam.in})`,
        });
      }
    }

    // ตรวจ parameters ถูกลบ
    for (const oldParam of oldParams) {
      const newParam = newParams.find(p => p.name === oldParam.name && p.in === oldParam.in);
      if (!newParam) {
        changes.push({
          severity: oldParam.required ? 'BREAKING' : 'NON_BREAKING',
          path: `${prefix}.parameters.${oldParam.name}`,
          description: `Parameter removed: ${oldParam.name} (in: ${oldParam.in})`,
        });
      }
    }

    // ตรวจ response status codes ถูกลบ
    for (const statusCode of Object.keys(oldOp.responses)) {
      if (!newOp.responses[statusCode] && statusCode !== 'default') {
        changes.push({
          severity: 'POTENTIALLY_BREAKING',
          path: `${prefix}.responses.${statusCode}`,
          description: `Response status code removed: ${statusCode}`,
        });
      }
    }

    return changes;
  }

  private compareSchemas(
    path: string,
    oldSchema: SchemaObject,
    newSchema: SchemaObject,
  ): ChangeReport[] {
    const changes: ChangeReport[] = [];

    // ตรวจ type เปลี่ยน
    if (oldSchema.type !== newSchema.type) {
      changes.push({
        severity: 'BREAKING',
        path,
        description: `Schema type changed`,
        before: oldSchema.type,
        after: newSchema.type,
      });
    }

    // ตรวจ required fields เพิ่ม
    const oldRequired = new Set(oldSchema.required ?? []);
    const newRequired = new Set(newSchema.required ?? []);

    for (const field of newRequired) {
      if (!oldRequired.has(field)) {
        changes.push({
          severity: 'BREAKING',
          path: `${path}.required`,
          description: `Required field added: ${field}`,
        });
      }
    }

    // ตรวจ properties ถูกลบ
    const oldProps = oldSchema.properties ?? {};
    const newProps = newSchema.properties ?? {};

    for (const propName of Object.keys(oldProps)) {
      if (!newProps[propName]) {
        changes.push({
          severity: 'BREAKING',
          path: `${path}.properties.${propName}`,
          description: `Property removed: ${propName}`,
        });
      }
    }

    return changes;
  }

  formatReport(changes: ChangeReport[]): string {
    if (changes.length === 0) {
      return 'No breaking changes detected.';
    }

    const breaking = changes.filter(c => c.severity === 'BREAKING');
    const potentiallyBreaking = changes.filter(c => c.severity === 'POTENTIALLY_BREAKING');
    const nonBreaking = changes.filter(c => c.severity === 'NON_BREAKING');

    let report = `Breaking Change Report\n${'='.repeat(40)}\n`;
    report += `Total changes: ${changes.length}\n`;
    report += `BREAKING: ${breaking.length}\n`;
    report += `POTENTIALLY_BREAKING: ${potentiallyBreaking.length}\n`;
    report += `NON_BREAKING: ${nonBreaking.length}\n\n`;

    for (const severity of ['BREAKING', 'POTENTIALLY_BREAKING', 'NON_BREAKING'] as const) {
      const group = changes.filter(c => c.severity === severity);
      if (group.length > 0) {
        report += `\n${severity}:\n`;
        for (const change of group) {
          report += `  - ${change.description}\n`;
          report += `    Path: ${change.path}\n`;
          if (change.before) report += `    Before: ${change.before}\n`;
          if (change.after) report += `    After: ${change.after}\n`;
        }
      }
    }

    return report;
  }
}

// CLI usage
const oldSpecPath = process.argv[2];
const newSpecPath = process.argv[3];

if (!oldSpecPath || !newSpecPath) {
  console.error('Usage: ts-node breaking-change-detector.ts <old-spec.json> <new-spec.json>');
  process.exit(1);
}

const oldSpec = JSON.parse(fs.readFileSync(path.resolve(oldSpecPath), 'utf-8')) as OpenAPISpec;
const newSpec = JSON.parse(fs.readFileSync(path.resolve(newSpecPath), 'utf-8')) as OpenAPISpec;

const detector = new BreakingChangeDetector();
const changes = detector.detect(oldSpec, newSpec);
console.log(detector.formatReport(changes));

// Exit code 1 ถ้ามี BREAKING changes (ใช้ใน CI)
const hasBreakingChanges = changes.some(c => c.severity === 'BREAKING');
process.exit(hasBreakingChanges ? 1 : 0);
```

---

## 6. API Versioning Strategies

### 6.1 URL Path Versioning

```typescript
// src/api/versioning/url-path.ts
import express, { Router, Request, Response } from 'express';

// V1 Router — original API
const v1Router = Router();

v1Router.get('/orders', (req: Request, res: Response) => {
  res.json({
    // V1 format: returns array directly
    orders: [
      { id: '1', status: 'pending', amount: 100 },
    ],
  });
});

v1Router.get('/orders/:id', (req: Request, res: Response) => {
  res.json({
    id: req.params.id,
    status: 'pending',
    amount: 100,
    // V1: ไม่มี metadata
  });
});

// V2 Router — improved API with pagination + metadata
const v2Router = Router();

v2Router.get('/orders', (req: Request, res: Response) => {
  const page = parseInt(req.query.page as string ?? '1', 10);
  const limit = parseInt(req.query.limit as string ?? '20', 10);

  res.json({
    // V2 format: ห่อด้วย pagination metadata
    data: [
      { id: '1', status: 'pending', totalAmount: { value: 100, currency: 'THB' } },
    ],
    pagination: {
      page,
      limit,
      total: 1,
      totalPages: 1,
    },
    // V2: เพิ่ม metadata
    meta: {
      apiVersion: 'v2',
      requestId: req.headers['x-request-id'] ?? crypto.randomUUID(),
    },
  });
});

// Main app
const app = express();
app.use('/v1', v1Router);
app.use('/v2', v2Router);

// Deprecation warning middleware สำหรับ V1
app.use('/v1', (req: Request, res: Response, next) => {
  res.set('Deprecation', 'true');
  res.set('Sunset', 'Sat, 31 Dec 2025 23:59:59 GMT');
  res.set('Link', '</v2/orders>; rel="successor-version"');
  next();
});
```

### 6.2 Accept Header Versioning

```typescript
// src/api/versioning/accept-header.ts
import { Request, Response, NextFunction, Router } from 'express';

type ApiVersion = 'v1' | 'v2' | 'v3';

interface VersionedRouter {
  version: ApiVersion;
  router: Router;
}

// Middleware: parse version จาก Accept header
function versionMiddleware(req: Request, res: Response, next: NextFunction): void {
  // Accept: application/vnd.company.api+json; version=2
  // หรือ Accept: application/vnd.company.api.v2+json
  const accept = req.headers.accept ?? '';

  let version: ApiVersion = 'v1'; // default

  const versionMatch = accept.match(/version=(\d+)/) ??
                       accept.match(/vnd\.company\.api\.v(\d+)/);
  if (versionMatch) {
    const v = parseInt(versionMatch[1], 10);
    if (v === 1) version = 'v1';
    else if (v === 2) version = 'v2';
    else if (v === 3) version = 'v3';
  }

  // เก็บ version ใน request object
  (req as Request & { apiVersion: ApiVersion }).apiVersion = version;

  // Set response header บอก version ที่ใช้
  res.set('Content-Type', `application/vnd.company.api.${version}+json`);
  next();
}

// Factory function สำหรับสร้าง versioned handler
function createVersionedHandler(handlers: Partial<Record<ApiVersion, (req: Request, res: Response) => void>>) {
  return (req: Request, res: Response) => {
    const version = (req as Request & { apiVersion: ApiVersion }).apiVersion;
    const handler = handlers[version] ?? handlers['v1'];
    if (handler) {
      handler(req, res);
    } else {
      res.status(406).json({ error: 'API version not supported' });
    }
  };
}

const router = Router();
router.use(versionMiddleware);

router.get('/products/:id', createVersionedHandler({
  v1: (req, res) => {
    res.json({ id: req.params.id, name: 'Product Name', price: 99.99 });
  },
  v2: (req, res) => {
    res.json({
      id: req.params.id,
      name: 'Product Name',
      pricing: { amount: 99.99, currency: 'THB', vat: 7 },
      metadata: { createdAt: new Date().toISOString() },
    });
  },
}));
```

### 6.3 Query Parameter Versioning

```typescript
// src/api/versioning/query-param.ts
import { Request, Response, NextFunction } from 'express';

function queryVersionMiddleware(req: Request, res: Response, next: NextFunction): void {
  const version = (req.query.api_version as string) ?? 
                  (req.query.v as string) ?? 
                  '1';
  
  (req as Request & { apiVersion: string }).apiVersion = `v${version}`;
  next();
}

// ตัวอย่าง: GET /orders?api_version=2
// หรือ: GET /orders?v=2
```

---

## 7. SLO Definition และ Tracking

### 7.1 SLO YAML Specification

```yaml
# slo/order-service-slos.yaml
# Service Level Objectives สำหรับ order-service
apiVersion: slo/v1
kind: ServiceLevelObjective
metadata:
  name: order-service-slos
  namespace: production
  labels:
    service: order-service
    team: commerce-team

spec:
  service: order-service
  description: "SLOs for Order Service critical user journeys"

  slos:
    # === 1. Availability SLO ===
    - name: availability
      description: "Order service must be available for users"
      target: 99.9   # 99.9% = สูงสุด 43.8 นาที downtime/เดือน
      window: 30d
      indicator:
        type: ratio
        goodEvents:
          # คำนวณจาก: requests ที่ไม่ใช่ 5xx
          promql: |
            sum(rate(http_requests_total{service="order-service",
              status_code!~"5.."}[5m]))
        totalEvents:
          promql: |
            sum(rate(http_requests_total{service="order-service"}[5m]))

    # === 2. Latency SLO ===
    - name: latency-p99
      description: "99% of order creation requests complete within 500ms"
      target: 99.0
      window: 7d
      indicator:
        type: ratio
        goodEvents:
          promql: |
            sum(rate(http_request_duration_seconds_bucket{
              service="order-service",
              route="/orders",
              method="POST",
              le="0.5"
            }[5m]))
        totalEvents:
          promql: |
            sum(rate(http_request_duration_seconds_count{
              service="order-service",
              route="/orders",
              method="POST"
            }[5m]))

    # === 3. Error Rate SLO ===
    - name: error-rate
      description: "Error rate must stay below 0.1%"
      target: 99.9  # 1 - error_rate = 99.9%
      window: 30d
      indicator:
        type: ratio
        goodEvents:
          promql: |
            sum(rate(http_requests_total{
              service="order-service",
              status_code!~"5.."
            }[5m]))
        totalEvents:
          promql: |
            sum(rate(http_requests_total{service="order-service"}[5m]))

    # === 4. Order Creation Success Rate ===
    - name: order-creation-success
      description: "95% of order creation attempts succeed"
      target: 95.0
      window: 7d
      indicator:
        type: ratio
        goodEvents:
          promql: |
            sum(rate(business_events_total{
              service="order-service",
              event="order.created"
            }[5m]))
        totalEvents:
          promql: |
            sum(rate(business_events_total{
              service="order-service",
              event="order.creation.attempted"
            }[5m]))

  alerting:
    # Alert เมื่อ burn rate สูงเกิน threshold
    burnRateAlerts:
      - name: "High Burn Rate - Critical"
        burnRateThreshold: 14.4  # consume 100% error budget ใน 1 ชั่วโมง
        forMinutes: 2
        severity: critical
        annotations:
          summary: "SLO burn rate critical for {{ $labels.slo }}"

      - name: "High Burn Rate - Warning"
        burnRateThreshold: 6.0   # consume 100% error budget ใน ~2.5 ชั่วโมง
        forMinutes: 15
        severity: warning
```

### 7.2 TypeScript SLOTracker Class

```typescript
// src/slo/slo-tracker.ts
// คำนวณ SLO metrics และ error budget

interface SLOConfig {
  name: string;
  target: number;        // เปอร์เซ็นต์ (เช่น 99.9)
  windowDays: number;    // window ในหน่วยวัน
}

interface SLOMetrics {
  sloName: string;
  targetPercent: number;
  currentPercent: number;
  errorBudgetPercent: number;        // % ของ error budget ที่เหลือ
  errorBudgetConsumedPercent: number; // % ที่ใช้ไปแล้ว
  remainingMinutes: number;          // นาที downtime ที่เหลือใน window
  status: 'healthy' | 'at_risk' | 'violated';
  burnRate: number;                  // burn rate ปัจจุบัน
  projectedExhaustionHours?: number; // คาดว่า budget จะหมดใน กี่ชั่วโมง
}

export class SLOTracker {
  private readonly config: SLOConfig;

  constructor(config: SLOConfig) {
    this.config = config;
  }

  // คำนวณ SLO metrics จากข้อมูล events
  calculate(goodEvents: number, totalEvents: number): SLOMetrics {
    const { name, target, windowDays } = this.config;

    if (totalEvents === 0) {
      throw new Error('Cannot calculate SLO with zero total events');
    }

    // คำนวณ current SLO percentage
    const currentPercent = (goodEvents / totalEvents) * 100;

    // Error budget = 1 - target = portion ที่เหลือไว้สำหรับ errors
    // เช่น target 99.9% → error budget = 0.1%
    const errorBudgetPercent = 100 - target;

    // Error budget ที่เหลือ = error budget - errors ที่เกิดขึ้นจริง
    const actualErrorPercent = 100 - currentPercent;
    const errorBudgetConsumedPercent = (actualErrorPercent / errorBudgetPercent) * 100;
    const errorBudgetRemainingPercent = Math.max(0, 100 - errorBudgetConsumedPercent);

    // คำนวณ remaining minutes ใน window
    const totalMinutesInWindow = windowDays * 24 * 60;
    const allowedDowntimeMinutes = totalMinutesInWindow * (errorBudgetPercent / 100);
    const remainingMinutes = allowedDowntimeMinutes * (errorBudgetRemainingPercent / 100);

    // Burn rate = rate ที่ใช้ error budget เทียบกับ ideal rate
    // burn rate = 1 หมายถึงใช้ error budget พอดีกับที่ตั้งไว้
    // burn rate = 2 หมายถึงใช้เร็วกว่า 2x
    const burnRate = errorBudgetConsumedPercent > 0
      ? errorBudgetConsumedPercent / 100
      : 0;

    // คาดว่า budget จะหมดใน กี่ชั่วโมง
    let projectedExhaustionHours: number | undefined;
    if (burnRate > 0 && errorBudgetRemainingPercent > 0) {
      const remainingBudgetHours = (errorBudgetRemainingPercent / 100) * totalMinutesInWindow / 60;
      projectedExhaustionHours = remainingBudgetHours / burnRate;
    }

    // กำหนด status
    let status: SLOMetrics['status'];
    if (errorBudgetRemainingPercent > 50) {
      status = 'healthy';
    } else if (errorBudgetRemainingPercent > 0) {
      status = 'at_risk';
    } else {
      status = 'violated';
    }

    return {
      sloName: name,
      targetPercent: target,
      currentPercent: Math.round(currentPercent * 1000) / 1000,
      errorBudgetPercent,
      errorBudgetConsumedPercent: Math.round(errorBudgetConsumedPercent * 10) / 10,
      remainingMinutes: Math.round(remainingMinutes),
      status,
      burnRate: Math.round(burnRate * 100) / 100,
      projectedExhaustionHours: projectedExhaustionHours
        ? Math.round(projectedExhaustionHours * 10) / 10
        : undefined,
    };
  }

  // คำนวณ burn rate สำหรับ alerting
  calculateBurnRate(
    recentGoodEvents: number,
    recentTotalEvents: number,
    windowMinutes: number,
  ): number {
    if (recentTotalEvents === 0) return 0;

    const recentErrorRate = 1 - (recentGoodEvents / recentTotalEvents);
    const allowedErrorRate = (100 - this.config.target) / 100;
    const totalWindowMinutes = this.config.windowDays * 24 * 60;

    // burn rate = (recent error rate / allowed error rate) * (window / recent window)
    return (recentErrorRate / allowedErrorRate) * (totalWindowMinutes / windowMinutes);
  }
}

// ===== ตัวอย่างการใช้งาน =====
const availabilitySLO = new SLOTracker({
  name: 'availability',
  target: 99.9,
  windowDays: 30,
});

// สมมติใน 30 วัน มี requests 1,000,000 ครั้ง error 200 ครั้ง
const metrics = availabilitySLO.calculate(
  999800,  // good events
  1000000, // total events
);

console.log('SLO Metrics:', metrics);
// Output:
// {
//   sloName: 'availability',
//   targetPercent: 99.9,
//   currentPercent: 99.98,
//   errorBudgetPercent: 0.1,
//   errorBudgetConsumedPercent: 20,    // ใช้ไป 20% ของ budget
//   remainingMinutes: 34,              // เหลืออีก ~34 นาที
//   status: 'healthy',
//   burnRate: 0.2,
// }
```

---

## 8. Governance Automation ด้วย OPA

Open Policy Agent (OPA) เป็น policy engine ที่ช่วยบังคับใช้ governance policies อัตโนมัติ

### 8.1 OPA Policy สำหรับ Kubernetes

```rego
# policies/kubernetes/container-security.rego
package kubernetes.admission

import future.keywords.if
import future.keywords.in

# ===== Policy 1: ห้าม run as root =====
deny[msg] {
  input.request.kind.kind == "Pod"
  container := input.request.object.spec.containers[_]
  not container.securityContext.runAsNonRoot
  msg := sprintf("Container '%v' must not run as root. Set securityContext.runAsNonRoot: true", [container.name])
}

# ===== Policy 2: ต้องมี resource limits =====
deny[msg] {
  input.request.kind.kind == "Pod"
  container := input.request.object.spec.containers[_]
  not container.resources.limits.memory
  msg := sprintf("Container '%v' must have memory limit set", [container.name])
}

deny[msg] {
  input.request.kind.kind == "Pod"
  container := input.request.object.spec.containers[_]
  not container.resources.limits.cpu
  msg := sprintf("Container '%v' must have CPU limit set", [container.name])
}

# ===== Policy 3: ห้ามใช้ latest tag =====
deny[msg] {
  input.request.kind.kind == "Pod"
  container := input.request.object.spec.containers[_]
  endswith(container.image, ":latest")
  msg := sprintf("Container '%v' must not use ':latest' image tag", [container.name])
}

deny[msg] {
  input.request.kind.kind == "Pod"
  container := input.request.object.spec.containers[_]
  not contains(container.image, ":")
  msg := sprintf("Container '%v' must specify image tag (not just image name)", [container.name])
}

# ===== Policy 4: ต้องมี readiness probe =====
warn[msg] {
  input.request.kind.kind == "Pod"
  container := input.request.object.spec.containers[_]
  not container.readinessProbe
  msg := sprintf("Container '%v' should have readinessProbe defined", [container.name])
}

# ===== Policy 5: ห้ามใช้ privileged mode =====
deny[msg] {
  input.request.kind.kind == "Pod"
  container := input.request.object.spec.containers[_]
  container.securityContext.privileged == true
  msg := sprintf("Container '%v' must not run in privileged mode", [container.name])
}
```

### 8.2 TypeScript OPA Policy Checker

```typescript
// src/governance/opa-checker.ts
// ตรวจสอบ policies ด้วย OPA REST API
import axios from 'axios';

interface OPAInput {
  request: {
    kind: { kind: string; apiVersion: string };
    object: unknown;
    namespace?: string;
    userInfo?: { username: string; groups: string[] };
  };
}

interface OPAPolicyResult {
  allow: boolean;
  deny: string[];
  warn: string[];
}

interface AdmissionReviewResult {
  allowed: boolean;
  denialReasons: string[];
  warnings: string[];
  policyName: string;
}

export class OPAPolicyChecker {
  private readonly opaUrl: string;

  constructor(opaUrl = 'http://opa.opa-system.svc.cluster.local:8181') {
    this.opaUrl = opaUrl;
  }

  async checkPolicy(
    policyPath: string,
    input: OPAInput,
  ): Promise<AdmissionReviewResult> {
    const url = `${this.opaUrl}/v1/data/${policyPath.replace(/\./g, '/')}`;

    try {
      const response = await axios.post<{ result: OPAPolicyResult }>(url, { input });
      const result = response.data.result;

      return {
        allowed: !result.deny || result.deny.length === 0,
        denialReasons: result.deny ?? [],
        warnings: result.warn ?? [],
        policyName: policyPath,
      };
    } catch (error) {
      // Fail open หรือ fail closed ขึ้นอยู่กับ policy
      console.error(`[OPA] Error checking policy ${policyPath}:`, error);
      throw new Error(`OPA policy check failed: ${(error as Error).message}`);
    }
  }

  // ตรวจสอบ Kubernetes resource ก่อน apply
  async checkKubernetesResource(
    resourceKind: string,
    resourceObject: unknown,
    namespace = 'default',
  ): Promise<AdmissionReviewResult[]> {
    const input: OPAInput = {
      request: {
        kind: { kind: resourceKind, apiVersion: 'v1' },
        object: resourceObject,
        namespace,
        userInfo: { username: 'ci-system', groups: ['system:serviceaccounts'] },
      },
    };

    const policies = [
      'kubernetes.admission',
    ];

    const results = await Promise.all(
      policies.map(policy => this.checkPolicy(policy, input)),
    );

    return results;
  }

  // Helper: รวม results จากหลาย policies
  mergeResults(results: AdmissionReviewResult[]): {
    allowed: boolean;
    allDenials: string[];
    allWarnings: string[];
  } {
    return {
      allowed: results.every(r => r.allowed),
      allDenials: results.flatMap(r => r.denialReasons),
      allWarnings: results.flatMap(r => r.warnings),
    };
  }
}

// ===== ตัวอย่าง: ตรวจสอบ Deployment ก่อน deploy =====
async function validateDeployment(deploymentYaml: unknown): Promise<void> {
  const checker = new OPAPolicyChecker();

  const results = await checker.checkKubernetesResource(
    'Deployment',
    deploymentYaml,
    'production',
  );

  const merged = checker.mergeResults(results);

  if (merged.allWarnings.length > 0) {
    console.warn('Governance Warnings:');
    merged.allWarnings.forEach(w => console.warn(`  ⚠️  ${w}`));
  }

  if (!merged.allowed) {
    console.error('Governance Policy VIOLATIONS:');
    merged.allDenials.forEach(d => console.error(`  ❌  ${d}`));
    throw new Error('Deployment violates governance policies');
  }

  console.log('✅ All governance policies passed');
}
```

---

## 9. Dependency Version Compatibility Checker

```typescript
// scripts/dependency-checker.ts
// ตรวจสอบ compatibility ของ dependencies ระหว่าง services
import * as fs from 'fs';
import * as path from 'path';
import * as semver from 'semver';

interface PackageJson {
  name: string;
  version: string;
  dependencies: Record<string, string>;
  devDependencies?: Record<string, string>;
  peerDependencies?: Record<string, string>;
}

interface DependencyConflict {
  packageName: string;
  services: Array<{
    serviceName: string;
    requiredVersion: string;
    resolvedVersion?: string;
  }>;
  severity: 'conflict' | 'warning' | 'info';
  recommendation: string;
}

interface CompatibilityReport {
  scannedServices: number;
  totalDependencies: number;
  conflicts: DependencyConflict[];
  outdatedPackages: Array<{ package: string; current: string; latest: string }>;
}

export class DependencyCompatibilityChecker {
  // โหลด package.json จาก services directory
  loadServicePackages(servicesDir: string): Map<string, PackageJson> {
    const services = new Map<string, PackageJson>();

    const entries = fs.readdirSync(servicesDir, { withFileTypes: true });
    for (const entry of entries) {
      if (!entry.isDirectory()) continue;

      const pkgPath = path.join(servicesDir, entry.name, 'package.json');
      if (!fs.existsSync(pkgPath)) continue;

      try {
        const pkg = JSON.parse(fs.readFileSync(pkgPath, 'utf-8')) as PackageJson;
        services.set(entry.name, pkg);
      } catch (error) {
        console.warn(`[Checker] Could not parse ${pkgPath}: ${(error as Error).message}`);
      }
    }

    return services;
  }

  // ตรวจหา version conflicts ระหว่าง services
  detectConflicts(services: Map<string, PackageJson>): DependencyConflict[] {
    const conflicts: DependencyConflict[] = [];

    // รวบรวม versions ของแต่ละ package จากทุก services
    const packageVersions = new Map<string, Array<{ service: string; version: string }>>();

    for (const [serviceName, pkg] of services) {
      const allDeps = {
        ...pkg.dependencies,
        ...(pkg.devDependencies ?? {}),
      };

      for (const [pkgName, version] of Object.entries(allDeps)) {
        if (!packageVersions.has(pkgName)) {
          packageVersions.set(pkgName, []);
        }
        packageVersions.get(pkgName)!.push({ service: serviceName, version });
      }
    }

    // ตรวจว่า versions เข้ากันได้หรือไม่
    for (const [pkgName, versionEntries] of packageVersions) {
      if (versionEntries.length < 2) continue;

      // ตรวจว่า major versions ต่างกัน (breaking change)
      const majors = new Set(
        versionEntries
          .map(e => {
            const v = semver.minVersion(e.version);
            return v ? semver.major(v) : null;
          })
          .filter(Boolean),
      );

      if (majors.size > 1) {
        conflicts.push({
          packageName: pkgName,
          services: versionEntries.map(e => ({
            serviceName: e.service,
            requiredVersion: e.version,
          })),
          severity: 'conflict',
          recommendation: `Align all services to use the same major version of ${pkgName}`,
        });
      }
    }

    return conflicts;
  }

  // ตรวจ packages ที่มี known security vulnerabilities
  async checkSecurityAdvisories(
    packages: Map<string, string>,
  ): Promise<Array<{ package: string; version: string; advisory: string }>> {
    // ในการใช้งานจริง จะ query npm advisory database หรือ snyk API
    // นี่เป็น mock implementation
    const mockAdvisories = [
      { package: 'lodash', affectedVersions: '<4.17.21', advisory: 'CVE-2021-23337' },
      { package: 'axios', affectedVersions: '<1.3.2', advisory: 'CVE-2023-45857' },
    ];

    const vulnerabilities: Array<{ package: string; version: string; advisory: string }> = [];

    for (const [pkgName, version] of packages) {
      const advisory = mockAdvisories.find(
        a => a.package === pkgName && semver.satisfies(
          semver.minVersion(version)?.version ?? '0.0.0',
          a.affectedVersions,
        ),
      );

      if (advisory) {
        vulnerabilities.push({
          package: pkgName,
          version,
          advisory: advisory.advisory,
        });
      }
    }

    return vulnerabilities;
  }

  formatReport(report: CompatibilityReport): string {
    let output = '=== Dependency Compatibility Report ===\n\n';
    output += `Scanned Services: ${report.scannedServices}\n`;
    output += `Total Dependencies: ${report.totalDependencies}\n`;
    output += `Conflicts Found: ${report.conflicts.length}\n\n`;

    if (report.conflicts.length > 0) {
      output += '--- Version Conflicts ---\n';
      for (const conflict of report.conflicts) {
        output += `\n[${conflict.severity.toUpperCase()}] ${conflict.packageName}\n`;
        for (const svc of conflict.services) {
          output += `  ${svc.serviceName}: ${svc.requiredVersion}\n`;
        }
        output += `  Recommendation: ${conflict.recommendation}\n`;
      }
    }

    return output;
  }
}
```

---

## 10. Team API Ownership

### 10.1 CODEOWNERS Format

```
# CODEOWNERS
# format: <pattern> <owner(s)>
# เมื่อ PR เปลี่ยนไฟล์ที่ match pattern จะ auto-assign reviewers

# === Global Rules ===
* @company/platform-team

# === Service Ownership ===
/services/order-service/       @company/commerce-team
/services/payment-service/     @company/payments-team
/services/user-service/        @company/identity-team
/services/inventory-service/   @company/catalog-team
/services/notification-service/ @company/platform-team

# === Infrastructure ===
/kubernetes/                   @company/platform-team @company/sre-team
/terraform/                    @company/platform-team @company/sre-team
/.github/workflows/            @company/platform-team

# === API Contracts (require both teams to review) ===
/services/order-service/api/   @company/commerce-team @company/api-team
/services/payment-service/api/ @company/payments-team @company/api-team

# === Shared Libraries ===
/packages/shared-types/        @company/platform-team @company/api-team
/packages/common-utils/        @company/platform-team

# === ADRs require Architecture approval ===
/docs/adr/                     @company/architecture-committee
```

### 10.2 Service Ownership Registry

```typescript
// src/governance/ownership-registry.ts
// Registry สำหรับ service ownership และ escalation paths
interface TeamContact {
  name: string;
  email: string;
  slackChannel: string;
  slackHandle: string;
  pagerDutySchedule?: string;
  oncallRotationUrl?: string;
}

interface ServiceOwnership {
  serviceId: string;
  primaryTeam: string;
  secondaryTeam?: string;
  contacts: TeamContact[];
  escalationPath: string[];
  slaResponseTime: {
    critical: number;  // minutes
    high: number;
    medium: number;
    low: number;
  };
  reviewRequirements: {
    minApprovals: number;
    requiresSecurityReview: boolean;
    requiresArchitectureReview: boolean;
    requiresProductApproval: boolean;
  };
  maintenanceWindows?: Array<{
    dayOfWeek: string;
    startTime: string;
    endTime: string;
    timezone: string;
  }>;
}

const SERVICE_OWNERSHIP_REGISTRY: ServiceOwnership[] = [
  {
    serviceId: 'order-service',
    primaryTeam: 'commerce-team',
    contacts: [
      {
        name: 'Commerce Team',
        email: 'commerce-team@company.com',
        slackChannel: '#commerce-eng',
        slackHandle: '@commerce-oncall',
        pagerDutySchedule: 'P1234567',
      },
    ],
    escalationPath: [
      'commerce-team',
      'backend-chapter-lead',
      'vp-engineering',
    ],
    slaResponseTime: {
      critical: 15,
      high: 60,
      medium: 240,
      low: 1440,
    },
    reviewRequirements: {
      minApprovals: 2,
      requiresSecurityReview: true,
      requiresArchitectureReview: false,
      requiresProductApproval: false,
    },
    maintenanceWindows: [
      {
        dayOfWeek: 'Sunday',
        startTime: '02:00',
        endTime: '06:00',
        timezone: 'Asia/Bangkok',
      },
    ],
  },
  {
    serviceId: 'payment-service',
    primaryTeam: 'payments-team',
    secondaryTeam: 'security-team',
    contacts: [
      {
        name: 'Payments Team',
        email: 'payments@company.com',
        slackChannel: '#payments-eng',
        slackHandle: '@payments-oncall',
        pagerDutySchedule: 'P7654321',
      },
    ],
    escalationPath: [
      'payments-team',
      'security-team',
      'cto',
    ],
    slaResponseTime: {
      critical: 5,    // payments critical = 5 นาที
      high: 30,
      medium: 120,
      low: 480,
    },
    reviewRequirements: {
      minApprovals: 3,
      requiresSecurityReview: true,
      requiresArchitectureReview: true,
      requiresProductApproval: true,   // payment changes ต้องได้ product approval
    },
  },
];

export class ServiceOwnershipRegistry {
  private readonly registry: Map<string, ServiceOwnership>;

  constructor(entries: ServiceOwnership[]) {
    this.registry = new Map(entries.map(e => [e.serviceId, e]));
  }

  getOwnership(serviceId: string): ServiceOwnership | undefined {
    return this.registry.get(serviceId);
  }

  getTeamServices(teamName: string): ServiceOwnership[] {
    return Array.from(this.registry.values()).filter(
      o => o.primaryTeam === teamName || o.secondaryTeam === teamName,
    );
  }

  getEscalationContact(serviceId: string, level = 0): string | undefined {
    const ownership = this.registry.get(serviceId);
    return ownership?.escalationPath[level];
  }

  getSLAResponseTime(
    serviceId: string,
    severity: keyof ServiceOwnership['slaResponseTime'],
  ): number | undefined {
    return this.registry.get(serviceId)?.slaResponseTime[severity];
  }
}

export const ownershipRegistry = new ServiceOwnershipRegistry(SERVICE_OWNERSHIP_REGISTRY);
```

---

## 11. Docker Compose สำหรับ Local Testing

```yaml
# docker-compose.yml
# Local development environment สำหรับ governance tools
version: '3.8'

services:
  # ===== Pact Broker =====
  pact-broker:
    image: pactfoundation/pact-broker:latest
    ports:
      - "9292:9292"
    environment:
      PACT_BROKER_DATABASE_URL: "postgres://pact:pact@pact-db/pact"
      PACT_BROKER_BASIC_AUTH_USERNAME: admin
      PACT_BROKER_BASIC_AUTH_PASSWORD: secret
      PACT_BROKER_ALLOW_PUBLIC_READ: "true"
    depends_on:
      - pact-db
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:9292/diagnostic/status/heartbeat"]
      interval: 10s
      timeout: 3s
      retries: 5

  pact-db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: pact
      POSTGRES_PASSWORD: pact
      POSTGRES_DB: pact
    volumes:
      - pact-db-data:/var/lib/postgresql/data

  # ===== OPA (Open Policy Agent) =====
  opa:
    image: openpolicyagent/opa:latest-rootless
    ports:
      - "8181:8181"
    command:
      - "run"
      - "--server"
      - "--log-level=info"
      - "--log-format=json-pretty"
      - "/policies"
    volumes:
      - ./policies:/policies:ro
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:8181/health"]
      interval: 10s
      timeout: 3s
      retries: 3

  # ===== Service Catalog =====
  service-catalog:
    build:
      context: ./service-catalog
      dockerfile: Dockerfile
    ports:
      - "3010:3010"
    environment:
      PORT: "3010"
      NODE_ENV: development
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:3010/health"]
      interval: 15s
      timeout: 5s
      retries: 3

  # ===== SLO Dashboard (Grafana) =====
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_USER: admin
      GF_SECURITY_ADMIN_PASSWORD: admin
      GF_INSTALL_PLUGINS: grafana-piechart-panel
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/dashboards:/etc/grafana/provisioning/dashboards:ro
      - ./grafana/datasources:/etc/grafana/provisioning/datasources:ro

  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=30d'

volumes:
  pact-db-data:
  grafana-data:
  prometheus-data:
```

---

## 12. Jest Test Examples

```typescript
// tests/governance.test.ts
import { ServiceRegistry, createServiceCatalogRouter } from '../src/registry/service.registry';
import { MaturityAssessor } from '../src/governance/maturity-assessor';
import { SLOTracker } from '../src/slo/slo-tracker';
import { BreakingChangeDetector } from '../src/governance/breaking-change-detector';
import express from 'express';
import request from 'supertest';

describe('ServiceRegistry', () => {
  let registry: ServiceRegistry;

  beforeEach(() => {
    registry = new ServiceRegistry();
  });

  it('should register a service successfully', () => {
    registry.registerService({
      id: 'test-service',
      name: 'Test Service',
      description: 'A test service',
      version: '1.0.0',
      status: 'active',
      type: 'api',
      language: 'TypeScript',
      framework: 'Express',
      ownership: {
        team: 'test-team',
        teamEmail: 'test@test.com',
        slackChannel: '#test',
      },
      endpoints: [],
      dependencies: [],
      slo: { availability: 99.9, latencyP99Ms: 500, errorRatePercent: 0.1 },
      tags: ['test'],
      documentation: 'https://docs.test.com',
      repository: 'https://github.com/test/test',
      createdAt: new Date().toISOString(),
      updatedAt: new Date().toISOString(),
    });

    const service = registry.getService('test-service');
    expect(service).toBeDefined();
    expect(service?.name).toBe('Test Service');
  });

  it('should list services filtered by team', () => {
    // Register services for different teams
    ['service-a', 'service-b'].forEach(id => {
      registry.registerService({
        id,
        name: id,
        description: '',
        version: '1.0.0',
        status: 'active',
        type: 'api',
        language: 'TypeScript',
        framework: 'Express',
        ownership: { team: 'team-alpha', teamEmail: '', slackChannel: '' },
        endpoints: [],
        dependencies: [],
        slo: { availability: 99.9, latencyP99Ms: 500, errorRatePercent: 0.1 },
        tags: [],
        documentation: '',
        repository: '',
        createdAt: new Date().toISOString(),
        updatedAt: new Date().toISOString(),
      });
    });

    registry.registerService({
      id: 'service-c',
      name: 'service-c',
      description: '',
      version: '1.0.0',
      status: 'active',
      type: 'api',
      language: 'TypeScript',
      framework: 'Express',
      ownership: { team: 'team-beta', teamEmail: '', slackChannel: '' },
      endpoints: [],
      dependencies: [],
      slo: { availability: 99.9, latencyP99Ms: 500, errorRatePercent: 0.1 },
      tags: [],
      documentation: '',
      repository: '',
      createdAt: new Date().toISOString(),
      updatedAt: new Date().toISOString(),
    });

    const alphaServices = registry.listServices({ team: 'team-alpha' });
    expect(alphaServices).toHaveLength(2);
    expect(alphaServices.every(s => s.ownership.team === 'team-alpha')).toBe(true);
  });

  it('should detect circular dependencies', () => {
    // Create services with circular dependency: A → B → C → A
    ['service-a', 'service-b', 'service-c'].forEach((id, index) => {
      const nextId = ['service-b', 'service-c', 'service-a'][index];
      registry.registerService({
        id,
        name: id,
        description: '',
        version: '1.0.0',
        status: 'active',
        type: 'api',
        language: 'TypeScript',
        framework: 'Express',
        ownership: { team: 'test-team', teamEmail: '', slackChannel: '' },
        endpoints: [],
        dependencies: [{ serviceId: nextId, type: 'hard', description: 'test' }],
        slo: { availability: 99.9, latencyP99Ms: 500, errorRatePercent: 0.1 },
        tags: [],
        documentation: '',
        repository: '',
        createdAt: new Date().toISOString(),
        updatedAt: new Date().toISOString(),
      });
    });

    const cycles = registry.detectCircularDependencies();
    expect(cycles.length).toBeGreaterThan(0);
  });
});

describe('SLOTracker', () => {
  it('should calculate SLO metrics correctly', () => {
    const tracker = new SLOTracker({
      name: 'test-availability',
      target: 99.9,
      windowDays: 30,
    });

    // 99.95% good events
    const metrics = tracker.calculate(999500, 1000000);

    expect(metrics.currentPercent).toBeCloseTo(99.95, 2);
    expect(metrics.status).toBe('healthy');
    expect(metrics.errorBudgetConsumedPercent).toBeCloseTo(50, 1);
  });

  it('should mark SLO as violated when error budget exhausted', () => {
    const tracker = new SLOTracker({
      name: 'test-slo',
      target: 99.9,
      windowDays: 30,
    });

    // Only 99% good — error rate 10x budget
    const metrics = tracker.calculate(990000, 1000000);

    expect(metrics.status).toBe('violated');
    expect(metrics.errorBudgetConsumedPercent).toBeGreaterThan(100);
  });
});

describe('BreakingChangeDetector', () => {
  let detector: BreakingChangeDetector;

  beforeEach(() => {
    detector = new BreakingChangeDetector();
  });

  it('should detect removed endpoint as BREAKING', () => {
    const oldSpec = {
      openapi: '3.0.0',
      info: { title: 'API', version: '1.0.0' },
      paths: {
        '/users': { get: { responses: { '200': { description: 'OK' } } } },
        '/users/{id}': { get: { responses: { '200': { description: 'OK' } } } },
      },
    };

    const newSpec = {
      openapi: '3.0.0',
      info: { title: 'API', version: '2.0.0' },
      paths: {
        '/users': { get: { responses: { '200': { description: 'OK' } } } },
        // '/users/{id}' ถูกลบออก
      },
    };

    const changes = detector.detect(oldSpec as any, newSpec as any);
    const breakingChanges = changes.filter(c => c.severity === 'BREAKING');
    expect(breakingChanges.length).toBeGreaterThan(0);
    expect(breakingChanges[0].description).toContain('/users/{id}');
  });

  it('should not flag new optional endpoints as breaking', () => {
    const oldSpec = {
      openapi: '3.0.0',
      info: { title: 'API', version: '1.0.0' },
      paths: {
        '/users': { get: { responses: { '200': { description: 'OK' } } } },
      },
    };

    const newSpec = {
      openapi: '3.0.0',
      info: { title: 'API', version: '1.1.0' },
      paths: {
        '/users': { get: { responses: { '200': { description: 'OK' } } } },
        '/users/{id}/profile': { get: { responses: { '200': { description: 'OK' } } } },
      },
    };

    const changes = detector.detect(oldSpec as any, newSpec as any);
    const breakingChanges = changes.filter(c => c.severity === 'BREAKING');
    expect(breakingChanges).toHaveLength(0);
  });
});

describe('MaturityAssessor', () => {
  it('should score correctly and recommend improvements', () => {
    const assessor = new MaturityAssessor();
    const result = assessor.assess({
      serviceName: 'test-service',
      assessmentDate: new Date().toISOString(),
      assessor: 'test',
      scores: {
        structured_logging: 5,
        distributed_tracing: 2,
        metrics_exposed: 5,
        alerting_configured: 1,
        dashboard_exists: 3,
        health_checks: 5,
        graceful_shutdown: 5,
        circuit_breaker: 2,
        retry_with_backoff: 2,
        slo_defined: 1,
        authentication: 5,
        secrets_management: 5,
        dependency_scanning: 3,
        image_scanning: 4,
        automated_tests: 4,
        contract_tests: 1,
        automated_deployment: 5,
        openapi_spec: 5,
        runbook_exists: 1,
        adr_documented: 2,
      },
    });

    expect(result.totalScore).toBeGreaterThan(0);
    expect(result.totalScore).toBeLessThanOrEqual(result.maxPossibleScore);
    expect(result.recommendations.length).toBeGreaterThan(0);
    expect(result.level).toBeTruthy();
  });
});
```

---

## สรุป Part 65

| หัวข้อ | เครื่องมือ/Approach | ประโยชน์ |
|--------|---------------------|---------|
| API Contract Testing | Pact (Consumer-Driven) | ป้องกัน API breaking changes โดยไม่ต้อง integration test ทั้งระบบ |
| Service Catalog | TypeScript ServiceRegistry | ค้นหา service, dependency graph, ownership ได้ง่าย |
| ADR | Markdown template + 3 examples | บันทึกเหตุผลการตัดสินใจ ลด "ทำไมถึงทำแบบนี้?" |
| Maturity Assessment | TypeScript Scoring Script | วัดระดับ maturity ของ services หา gaps ที่ต้องพัฒนา |
| Breaking Change Detection | OpenAPI Diff Script | ตรวจ breaking changes อัตโนมัติใน CI/CD |
| API Versioning | URL Path, Header, Query Param | รองรับ clients เก่าขณะพัฒนา API ใหม่ |
| SLO Tracking | YAML Spec + TypeScript SLOTracker | วัด error budget, burn rate, alert เมื่อ risk สูง |
| Governance Automation | OPA + TypeScript Checker | บังคับใช้ policies อัตโนมัติ ลด manual review |
| Dependency Checking | TypeScript Checker | ตรวจ version conflicts, security advisories |
| Team Ownership | CODEOWNERS + Registry | ชัดเจนว่าใครรับผิดชอบอะไร, escalation path |

Microservices Governance ที่ดีช่วยให้ ecosystem ของ services เติบโตอย่างมีระเบียบ ลด technical debt และทำให้ทีมสามารถ deliver features ได้เร็วขึ้นด้วยความมั่นใจมากขึ้น

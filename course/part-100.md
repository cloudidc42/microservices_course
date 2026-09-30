# Part 100: Course Summary and Next Steps

## ยินดีต้อนรับสู่บทสุดท้าย! 🎉

ขอแสดงความยินดีที่คุณเดินทางมาถึงบทที่ 100 ของหลักสูตร Microservices คุณได้เรียนรู้ทุกสิ่งตั้งแต่ fundamental concepts ไปจนถึง advanced patterns และ production operations เนื้อหาในบทนี้จะช่วยให้คุณสรุปความรู้และวางแผนเส้นทางการเรียนรู้ต่อไป

---

## 100.1 Complete Course Summary Table

ตารางสรุปทุก 100 Parts พร้อม Key Concepts:

| Part | หัวข้อ | Key Concept |
|------|--------|-------------|
| 1 | Introduction to Microservices | Monolith vs Microservices, Conway's Law |
| 2 | Microservices Design Principles | SRP, Loose Coupling, High Cohesion |
| 3 | Domain-Driven Design | Bounded Context, Ubiquitous Language |
| 4 | Service Communication Patterns | REST, gRPC, GraphQL, Messaging |
| 5 | API Design Best Practices | RESTful design, versioning, pagination |
| 6 | Container Fundamentals | Docker, container lifecycle, images |
| 7 | Docker Compose | Multi-container apps, networking, volumes |
| 8 | Kubernetes Basics | Pods, Deployments, Services |
| 9 | Kubernetes Advanced | StatefulSets, DaemonSets, Jobs |
| 10 | Helm Charts | Package management for Kubernetes |
| 11 | Service Discovery | DNS-based, client-side, server-side |
| 12 | API Gateway Patterns | Kong, AWS API Gateway, nginx |
| 13 | Load Balancing | Algorithms, session affinity, health checks |
| 14 | Circuit Breaker Pattern | Hystrix, Resilience4j, state machine |
| 15 | Retry and Timeout Patterns | Exponential backoff, jitter, deadlines |
| 16 | Saga Pattern | Choreography vs Orchestration |
| 17 | Event-Driven Architecture | Event sourcing, CQRS, event store |
| 18 | Message Queues | RabbitMQ, SQS, NATS |
| 19 | Apache Kafka | Topics, partitions, consumer groups |
| 20 | Kafka Advanced | Exactly-once semantics, compaction |
| 21 | Data Management | Database per service, shared database anti-pattern |
| 22 | PostgreSQL for Microservices | Schemas, connection pooling, migrations |
| 23 | MongoDB for Microservices | Documents, aggregation, indexes |
| 24 | Redis Patterns | Cache-aside, write-through, pub/sub |
| 25 | Database Migrations | Flyway, Liquibase, zero-downtime migrations |
| 26 | Event Sourcing | Append-only log, projections, snapshots |
| 27 | CQRS Pattern | Read/write separation, eventual consistency |
| 28 | Distributed Transactions | 2PC problems, Saga as solution |
| 29 | Outbox Pattern | Reliable event publishing with transactions |
| 30 | Idempotency Patterns | Idempotency keys, deduplication |
| 31 | Authentication Patterns | JWT, OAuth2, OIDC |
| 32 | Authorization Patterns | RBAC, ABAC, OPA |
| 33 | API Security | Rate limiting, input validation, OWASP |
| 34 | Service-to-Service Auth | mTLS, SPIFFE, service accounts |
| 35 | Secrets Management | Vault, AWS Secrets Manager, Sealed Secrets |
| 36 | Network Policies | Kubernetes NetworkPolicy, Calico |
| 37 | Container Security | Image scanning, rootless containers |
| 38 | Supply Chain Security | SLSA, Sigstore, SBOMs |
| 39 | Zero Trust Architecture | Never trust, always verify |
| 40 | Compliance as Code | OPA Gatekeeper, Kyverno |
| 41 | Observability Fundamentals | Metrics, Logs, Traces, Profiles |
| 42 | Prometheus Basics | PromQL, exporters, recording rules |
| 43 | Prometheus Advanced | Alertmanager, federation, remote write |
| 44 | Grafana Dashboards | Visualizations, alerting, provisioning |
| 45 | Distributed Tracing | OpenTelemetry, Jaeger, sampling |
| 46 | Structured Logging | Log levels, correlation IDs, ELK stack |
| 47 | SLI, SLO, SLA | Error budgets, burn rate |
| 48 | Alerting Best Practices | Alert fatigue, multi-window alerting |
| 49 | Chaos Engineering | Principles, blast radius, GameDays |
| 50 | Continuous Profiling | CPU/memory profiles, pprof, Pyroscope |
| 51 | CI/CD Fundamentals | Pipeline stages, artifact management |
| 52 | GitHub Actions | Workflows, matrix builds, caching |
| 53 | GitLab CI/CD | Pipelines, runners, environments |
| 54 | ArgoCD and GitOps | Declarative deployments, sync, rollback |
| 55 | Deployment Strategies | Rolling, Blue/Green, Canary |
| 56 | Feature Flags | LaunchDarkly, Unleash, progressive rollout |
| 57 | Progressive Delivery | Flagger, Argo Rollouts, analysis |
| 58 | Infrastructure as Code | Terraform, Pulumi, CDK |
| 59 | Kubernetes Operators | CRDs, controllers, reconciliation loops |
| 60 | Service Mesh Basics | Istio architecture, data/control plane |
| 61 | Istio Advanced | Traffic management, fault injection |
| 62 | Linkerd | Lightweight mesh, automatic mTLS |
| 63 | eBPF for Networking | Cilium, Hubble, kernel-level observability |
| 64 | Multi-Cluster Kubernetes | Federation, traffic management |
| 65 | Multi-Region Deployment | Active-active, data replication |
| 66 | Autoscaling Patterns | HPA, VPA, KEDA, cluster autoscaler |
| 67 | Spot Instance Strategies | Fault-tolerant workloads, mixed fleets |
| 68 | Serverless Microservices | AWS Lambda, event-driven patterns |
| 69 | WebAssembly (WASM) | Edge computing, Envoy filters |
| 70 | GraphQL Federation | Schema stitching, Apollo Federation |
| 71 | BFF Pattern | Backend for Frontend, tailored APIs |
| 72 | gRPC and Protocol Buffers | Binary protocol, streaming, service mesh |
| 73 | AsyncAPI | Event-driven API documentation |
| 74 | Contract Testing | Consumer-driven contracts with Pact |
| 75 | Testing Strategies | Test pyramid, shift-left testing |
| 76 | Load Testing | k6, Locust, Artillery |
| 77 | Chaos Testing | Chaos Monkey, LitmusChaos, Gremlin |
| 78 | Security Testing | DAST, SAST, penetration testing |
| 79 | Performance Optimization | Profiling, caching strategies, CDN |
| 80 | Database Performance | Query optimization, indexing, partitioning |
| 81 | Caching Strategies | CDN, application cache, database cache |
| 82 | Rate Limiting Patterns | Token bucket, leaky bucket, sliding window |
| 83 | Bulkhead Pattern | Resource isolation, thread pools |
| 84 | Backpressure Patterns | Reactive streams, flow control |
| 85 | Platform Engineering | IDP, golden paths, developer experience |
| 86 | Team Topologies | Stream-aligned, platform, enabling teams |
| 87 | Developer Experience | Onboarding, self-service, cognitive load |
| 88 | Production Readiness | Checklist, readiness reviews |
| 89 | Incident Management | Runbooks, post-mortems, blameless culture |
| 90 | Microservices Maturity Model | 5 levels, DORA metrics, assessment |
| 91 | Event-Driven Microservices | Advanced patterns, event mesh |
| 92 | Data Mesh | Domain ownership, data products |
| 93 | AI/ML in Microservices | Model serving, feature stores |
| 94 | Edge Computing | CDN, Edge Functions, micro frontends |
| 95 | Green Computing | Carbon footprint, efficiency |
| 96 | Microservices Anti-Patterns | Distributed monolith, chatty services |
| 97 | Migration Strategies | Strangler Fig, Branch by Abstraction |
| 98 | Cost Optimization | Right-sizing, RI, spot instances |
| 99 | Final Project | Complete 5-service e-commerce system |
| 100 | Course Summary | Review, career path, next steps |

---

## 100.2 Knowledge Map

แผนที่ความรู้ที่ครอบคลุมทุกด้านของ Microservices:

```
MICROSERVICES KNOWLEDGE MAP
════════════════════════════════════════════════════════════════

┌─────────────────── ARCHITECTURE ───────────────────────────┐
│  • Domain-Driven Design    • Event-Driven Architecture    │
│  • CQRS + Event Sourcing   • Saga Pattern                 │
│  • Strangler Fig           • BFF Pattern                  │
│  • Service Mesh            • API Gateway                  │
│  • Outbox Pattern          • Idempotency                  │
└────────────────────────────────────────────────────────────┘

┌─────────────────── SECURITY ───────────────────────────────┐
│  • JWT / OAuth2 / OIDC     • mTLS + Zero Trust            │
│  • RBAC / ABAC / OPA       • Secrets Management           │
│  • Container Security      • Supply Chain (SLSA)          │
│  • Network Policies        • Compliance as Code           │
└────────────────────────────────────────────────────────────┘

┌─────────────────── OPERATIONS ─────────────────────────────┐
│  • CI/CD Pipelines         • GitOps (ArgoCD)               │
│  • Deployment Strategies   • Feature Flags                 │
│  • Incident Management     • SLO/Error Budget              │
│  • Chaos Engineering       • Disaster Recovery             │
│  • On-Call Rotation        • Post-Mortem Culture           │
└────────────────────────────────────────────────────────────┘

┌─────────────────── PERFORMANCE ────────────────────────────┐
│  • Caching Strategies      • Database Optimization         │
│  • Autoscaling (HPA/VPA)   • Rate Limiting                 │
│  • Circuit Breaker         • Bulkhead / Backpressure       │
│  • Load Testing            • Continuous Profiling          │
│  • Cost Optimization       • Right-sizing                  │
└────────────────────────────────────────────────────────────┘

┌─────────────────── TESTING ────────────────────────────────┐
│  • Test Pyramid            • Contract Testing (Pact)       │
│  • Load Testing (k6)       • Chaos Testing                 │
│  • Security Testing        • Mutation Testing              │
│  • Canary Analysis         • Synthetic Monitoring          │
└────────────────────────────────────────────────────────────┘

┌─────────────────── OBSERVABILITY ──────────────────────────┐
│  • Metrics (Prometheus)    • Logs (ELK/Loki)               │
│  • Traces (Jaeger/Tempo)   • Profiles (Pyroscope)          │
│  • Alerting (Alertmanager) • Dashboards (Grafana)          │
│  • SLIs/SLOs/SLAs          • Business Metrics              │
└────────────────────────────────────────────────────────────┘
```

---

## 100.3 Top 20 Microservices Patterns Quick Reference

| # | Pattern | เมื่อใช้ |
|---|---------|---------|
| 1 | **API Gateway** | เมื่อต้องการ single entry point สำหรับ clients, handle cross-cutting concerns (auth, rate limiting, SSL) |
| 2 | **Circuit Breaker** | เมื่อต้องการป้องกัน cascade failures, handle dependency timeouts gracefully |
| 3 | **Saga** | เมื่อต้องการ distributed transactions ข้ามหลาย services โดยไม่ใช้ 2PC |
| 4 | **Event Sourcing** | เมื่อต้องการ audit trail สมบูรณ์, time-travel queries, replay capabilities |
| 5 | **CQRS** | เมื่อ read/write patterns แตกต่างมาก, ต้องการ scale อิสระ |
| 6 | **Strangler Fig** | เมื่อต้องการ migrate จาก monolith ไป microservices ทีละส่วน |
| 7 | **Outbox Pattern** | เมื่อต้องการ guarantee event publishing พร้อมกับ database transaction |
| 8 | **Bulkhead** | เมื่อต้องการ isolate resources เพื่อป้องกัน failure propagation |
| 9 | **Retry with Exponential Backoff** | เมื่อ dependent service มี transient failures, ป้องกัน thundering herd |
| 10 | **Idempotent Consumer** | เมื่อต้องการ process messages exactly-once แม้ network อาจ deliver ซ้ำ |
| 11 | **Database per Service** | เมื่อต้องการ team autonomy, service isolation, independent scaling |
| 12 | **BFF (Backend for Frontend)** | เมื่อ mobile และ web clients ต้องการ data shapes แตกต่างกัน |
| 13 | **Service Mesh** | เมื่อต้องการ observability, security, traffic management โดยไม่เปลี่ยน application code |
| 14 | **Sidecar** | เมื่อต้องการ inject behavior (logging, proxy, monitoring) โดยไม่แก้ application |
| 15 | **Ambassador** | เมื่อต้องการ proxy ที่ handle cross-cutting concerns เช่น retry, circuit breaker |
| 16 | **Leader Election** | เมื่อต้องการให้ instance เดียวทำ critical task เช่น scheduled jobs |
| 17 | **Consumer-Driven Contract** | เมื่อต้องการ verify service compatibility ก่อน deployment |
| 18 | **Progressive Delivery** | เมื่อต้องการ reduce risk ของ deployment ด้วยการค่อยๆ เพิ่ม traffic |
| 19 | **Competing Consumers** | เมื่อต้องการ scale message processing horizontally |
| 20 | **Throttling** | เมื่อต้องการ protect service จาก overload และ fair usage enforcement |

---

## 100.4 Common Mistakes และ Anti-Patterns

### Anti-Pattern 1: Distributed Monolith

**ปัญหา**: แตก service แต่ยังมี tight coupling อยู่

```typescript
// ❌ Anti-Pattern: Synchronous chain dependency
// OrderService → UserService → ProductService → InventoryService
// ถ้า InventoryService ล่ม ทุก service ล่มหมด

async function createOrder(userId: string, productId: string) {
  const user = await userService.getUser(userId);        // sync call
  const product = await productService.getProduct(productId); // sync call
  const inventory = await inventoryService.check(productId);  // sync call
  // ...
}

// ✅ ทางแก้: Event-driven with caching
async function createOrder(userId: string, productId: string) {
  // ข้อมูลที่จำเป็นถูก cache ไว้แล้วจาก events
  const userSnapshot = await redis.get(`user:${userId}`);
  const productSnapshot = await redis.get(`product:${productId}`);
  
  // Publish event, let other services react
  await kafka.publish('orders.created', { userId, productId });
}
```

### Anti-Pattern 2: Shared Database

**ปัญหา**: หลาย service ใช้ database เดียวกัน ทำให้ tight coupling

```sql
-- ❌ Anti-Pattern: OrderService queries UserService's table directly
SELECT o.*, u.email, u.name
FROM orders o
JOIN users.users u ON o.user_id = u.id  -- cross-service DB join!

-- ✅ ทางแก้: Each service owns its data, replicate what's needed
-- Order service keeps a denormalized copy of user info
CREATE TABLE order_user_snapshots (
  user_id VARCHAR PRIMARY KEY,
  email VARCHAR,
  name VARCHAR,
  updated_at TIMESTAMP
);
```

### Anti-Pattern 3: Chatty Services

**ปัญหา**: Service ส่ง request จำนวนมาก เพื่อดึงข้อมูลเล็กน้อย

```typescript
// ❌ Anti-Pattern: N+1 problem across services
async function getOrderDetails(orderIds: string[]) {
  const orders = await orderService.getOrders(orderIds);
  
  // แยก request สำหรับแต่ละ order!
  for (const order of orders) {
    order.user = await userService.getUser(order.userId);      // N calls
    order.products = await productService.getProducts(order.items); // N calls
  }
}

// ✅ ทางแก้: Batch requests
async function getOrderDetails(orderIds: string[]) {
  const [orders, users, products] = await Promise.all([
    orderService.getOrders(orderIds),
    userService.getUsersBatch(userIds), // batch call
    productService.getProductsBatch(productIds), // batch call
  ]);
  
  // Merge in memory
  return orders.map(order => ({
    ...order,
    user: users.get(order.userId),
    products: order.items.map(item => products.get(item.productId)),
  }));
}
```

### Anti-Pattern 4: Synchronous Event Processing

```typescript
// ❌ Anti-Pattern: Making HTTP calls inside Kafka consumer
consumer.run({
  eachMessage: async ({ message }) => {
    const event = JSON.parse(message.value!.toString());
    // ถ้า downstream service ล่ม Kafka consumer ค้างหมด!
    await emailService.sendConfirmation(event.email);
    await pushNotification.send(event.userId, 'Order confirmed!');
  },
});

// ✅ ทางแก้: Fan-out to separate queues
consumer.run({
  eachMessage: async ({ message }) => {
    const event = JSON.parse(message.value!.toString());
    
    await Promise.all([
      // แต่ละ notification type มี queue แยก ล้มเหลวแยกกัน
      kafka.publish('notifications.email', { email: event.email, template: 'order-confirmed' }),
      kafka.publish('notifications.push', { userId: event.userId, message: 'Order confirmed!' }),
    ]);
  },
});
```

### Anti-Pattern 5: Ignoring Observability

```typescript
// ❌ Anti-Pattern: No instrumentation
app.post('/orders', async (req, res) => {
  const order = await createOrder(req.body);
  res.json(order);
});

// ✅ ทางแก้: Structured logging + metrics + tracing
app.post('/orders', async (req, res) => {
  const span = tracer.startSpan('create-order');
  const start = Date.now();
  
  try {
    const order = await createOrder(req.body);
    
    // Metrics
    orderCreatedCounter.labels({ status: 'success' }).inc();
    orderCreationDuration.observe(Date.now() - start);
    
    // Structured logging
    logger.info('Order created', {
      orderId: order.id,
      userId: req.user.id,
      total: order.total,
      duration: Date.now() - start,
      traceId: span.context().toTraceId(),
    });
    
    span.setStatus({ code: SpanStatusCode.OK });
    res.json(order);
  } catch (error) {
    span.recordException(error as Error);
    orderCreatedCounter.labels({ status: 'error' }).inc();
    logger.error('Order creation failed', { error, userId: req.user.id });
    res.status(500).json({ error: 'Order creation failed' });
  } finally {
    span.end();
  }
});
```

---

## 100.5 Career Path: Junior → Senior → Principal → Architect

### Junior Backend Engineer (0-2 ปี)

**ทักษะที่ต้องมี**:
- REST API design และ implementation
- SQL databases (PostgreSQL, MySQL)
- Basic Docker และ deployment
- Unit testing
- Git workflows

**เป้าหมาย**:
- เขียน code ที่ clean, readable, tested
- เข้าใจ service ที่ตัวเองดูแล
- รู้จัก infrastructure พื้นฐาน
- ทำ on-call ได้ด้วย runbook

**Coding example ระดับนี้**:
```typescript
// สร้าง REST endpoint ที่ดี
app.get('/users/:id', async (req, res) => {
  const user = await userRepository.findById(req.params.id);
  if (!user) return res.status(404).json({ error: 'User not found' });
  res.json(user);
});
```

---

### Mid-level / Senior Backend Engineer (2-5 ปี)

**ทักษะที่ต้องมี**:
- Microservices architecture patterns
- Distributed systems concepts
- Kubernetes operation
- CI/CD pipelines
- Performance optimization
- Security best practices
- Mentoring junior engineers

**เป้าหมาย**:
- ออกแบบ service interfaces ที่ดี
- แก้ปัญหา production incidents ได้อิสระ
- ปรับปรุง reliability และ performance
- นำทีมเล็กๆ ในงาน technical

---

### Principal Engineer (5-10 ปี)

**ทักษะที่ต้องมี**:
- System design ระดับ organization
- Technical strategy
- Cross-team collaboration
- RFC และ ADR writing
- Hiring และ team building
- Engineering metrics

**เป้าหมาย**:
- กำหนด technical direction ของหลาย teams
- แก้ปัญหา cross-cutting concerns
- Reduce technical debt proactively
- เป็น go-to person สำหรับ complex problems

---

### Software Architect / Staff Engineer (8+ ปี)

**ทักษะที่ต้องมี**:
- Enterprise architecture patterns
- Business domain understanding
- Vendor evaluation
- Technology radar management
- Stakeholder management
- Executive communication

**เป้าหมาย**:
- กำหนด architecture standards ทั้ง organization
- Enable teams ให้ move fast อย่างปลอดภัย
- Balance innovation กับ stability
- ผลักดัน engineering culture

---

## 100.6 Top 10 หนังสือ Microservices

| # | ชื่อหนังสือ | ผู้แต่ง | ISBN | คำอธิบาย |
|---|-------------|---------|------|-----------|
| 1 | Building Microservices, 2nd Ed. | Sam Newman | 978-1492034025 | Bible ของ microservices ครอบคลุมทุกมิติตั้งแต่ design ถึง deployment |
| 2 | Designing Distributed Systems | Brendan Burns | 978-1491983645 | Patterns สำหรับ distributed computing เขียนโดย co-creator ของ Kubernetes |
| 3 | Microservices Patterns | Chris Richardson | 978-1617294549 | 44 patterns พร้อม code examples ใน Java ใช้งานได้จริง |
| 4 | Site Reliability Engineering | Google SRE Team | 978-1491929124 | วิธีที่ Google operate ระบบขนาดใหญ่ |
| 5 | The DevOps Handbook | Gene Kim et al. | 978-1950508402 | DevOps principles และ practices สำหรับ enterprise |
| 6 | Accelerate | Nicole Forsgren et al. | 978-1942788331 | Research-based evidence ว่า DevOps practices ส่งผลต่อธุรกิจอย่างไร |
| 7 | Domain-Driven Design | Eric Evans | 978-0321125217 | Original DDD book, foundation ของการออกแบบ bounded contexts |
| 8 | Implementing Domain-Driven Design | Vaughn Vernon | 978-0321834577 | Practical DDD กับ code examples |
| 9 | Clean Architecture | Robert C. Martin | 978-0134494166 | หลักการออกแบบ software architecture ที่ maintainable |
| 10 | Release It! | Michael T. Nygard | 978-1680502398 | Production-ready software patterns สำหรับ stability |

---

## 100.7 Top 10 Blogs และ GitHub Repos

### Blogs ที่ต้องอ่าน

| # | Blog / Website | เหตุผลที่น่าอ่าน |
|---|----------------|------------------|
| 1 | **Netflix Tech Blog** (netflixtechblog.com) | Engineering ของระบบ scale ระดับโลก, Chaos Engineering, Distributed Systems |
| 2 | **Uber Engineering** (eng.uber.com) | Real-world microservices challenges, data at scale |
| 3 | **Martin Fowler** (martinfowler.com) | Software architecture patterns, agile practices, DDD |
| 4 | **High Scalability** (highscalability.com) | Case studies ของ system design จาก tech companies |
| 5 | **The New Stack** (thenewstack.io) | Cloud native, Kubernetes, DevOps news |
| 6 | **InfoQ** (infoq.com) | Conferences talks, architecture articles |
| 7 | **AWS Architecture Blog** (aws.amazon.com/blogs/architecture) | AWS architecture patterns |
| 8 | **Google Cloud Blog** (cloud.google.com/blog) | SRE, GKE, distributed systems |
| 9 | **Stripe Engineering** (stripe.com/blog/engineering) | Payment systems, API design |
| 10 | **Shopify Engineering** (shopify.engineering) | E-commerce at scale, Rails to microservices |

### GitHub Repos ที่ควรติดตาม

| # | Repository | คำอธิบาย |
|---|------------|-----------|
| 1 | **kubernetes/kubernetes** | Kubernetes source code, เรียนรู้ internals |
| 2 | **istio/istio** | Service mesh, ดู implementation patterns |
| 3 | **grafana/grafana** | Observability platform |
| 4 | **prometheus/prometheus** | Metrics system |
| 5 | **open-telemetry/opentelemetry-js** | OpenTelemetry JS SDK |
| 6 | **Netflix/conductor** | Microservices orchestration engine |
| 7 | **argoproj/argo-cd** | GitOps for Kubernetes |
| 8 | **hashicorp/vault** | Secrets management |
| 9 | **envoyproxy/envoy** | L7 proxy ที่ใช้ใน service meshes |
| 10 | **kiali/kiali** | Service mesh observability |

---

## 100.8 Certification Roadmap

### Cloud Native Certifications ที่แนะนำ

```
ENTRY LEVEL
────────────
    CKA (Certified Kubernetes Administrator)
    - ใครควรสอบ: DevOps/Platform engineers
    - เนื้อหา: Cluster management, networking, storage, troubleshooting
    - ประสบการณ์ที่ต้องการ: 6+ เดือนกับ Kubernetes
    - สอบที่: training.linuxfoundation.org

    CKAD (Certified Kubernetes Application Developer)
    - ใครควรสอบ: Application developers
    - เนื้อหา: Pod design, configuration, multi-container apps
    - ประสบการณ์ที่ต้องการ: รู้จัก K8s พื้นฐาน
    - สอบที่: training.linuxfoundation.org

INTERMEDIATE
────────────
    CKS (Certified Kubernetes Security Specialist)
    - ต้องมี CKA ก่อน
    - เนื้อหา: Cluster hardening, system hardening, supply chain security
    - ยากกว่า CKA มาก

    AWS SAP (Solutions Architect Professional)
    - ใครควรสอบ: Cloud architects
    - เนื้อหา: Advanced AWS services, hybrid architectures, cost optimization
    - ประสบการณ์ที่ต้องการ: 2+ ปีกับ AWS

ADVANCED
────────────
    GCP Professional Cloud Architect
    - เทียบเท่า AWS SAP สำหรับ GCP

    CNCF Security Certifications
    - OpenSSF Certified Developer (OSSD)
```

### การเตรียมสอบ CKA

```bash
# ฝึกด้วย killer.sh (ดีที่สุด)
# https://killer.sh/cka

# เนื้อหาหลัก 5 ด้าน:
# 1. Cluster Architecture (25%)
# 2. Workloads & Scheduling (15%)
# 3. Services & Networking (20%)
# 4. Storage (10%)
# 5. Troubleshooting (30%)

# Commands ที่ต้องชำนาญ:
kubectl get nodes -o wide
kubectl describe pod <name>
kubectl logs <pod> --previous
kubectl exec -it <pod> -- /bin/sh
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml
kubectl rollout status deployment/nginx
kubectl rollout history deployment/nginx
kubectl top nodes
kubectl top pods --all-namespaces
etcdctl snapshot save /backup/etcd.db
kubeadm token create --print-join-command
```

---

## 100.9 Technical Interview Q&A (30 คำถาม)

### Section 1: Architecture

**Q1: อธิบาย Saga pattern และเปรียบเทียบ Choreography vs Orchestration**

```typescript
// A: Saga คือ sequence ของ local transactions ที่ประสานกันผ่าน events/messages

// Choreography (event-based) - ไม่มี central coordinator
// OrderService → publishes order.created
// PaymentService → listens, charges card → publishes payment.succeeded
// InventoryService → listens, reserves items → publishes items.reserved
// OrderService → listens, marks order confirmed

// Orchestration (command-based) - มี central coordinator
class OrderSagaOrchestrator {
  async execute(orderId: string) {
    const paymentResult = await this.paymentService.charge(orderId);
    if (!paymentResult.success) {
      await this.compensate(orderId);
      return;
    }
    
    const inventoryResult = await this.inventoryService.reserve(orderId);
    if (!inventoryResult.success) {
      await this.paymentService.refund(orderId);
      await this.compensate(orderId);
      return;
    }
    
    await this.orderService.confirm(orderId);
  }
  
  async compensate(orderId: string) {
    // Run compensating transactions in reverse order
  }
}

// ใช้ Choreography เมื่อ: simple flows, loose coupling important
// ใช้ Orchestration เมื่อ: complex flows, need central visibility
```

**Q2: อธิบาย Event Sourcing และข้อดีข้อเสีย**

```typescript
// A: แทนที่จะ store current state, เก็บ sequence ของ events

// Traditional (store state)
await db.query('UPDATE orders SET status = $1 WHERE id = $2', ['confirmed', orderId]);

// Event Sourcing (store events)
await eventStore.append('order-' + orderId, {
  eventType: 'OrderConfirmed',
  data: { orderId, confirmedAt: new Date() },
  version: 5,
});

// ข้อดี:
// - Complete audit trail
// - Time-travel: rebuild state at any point in history
// - Event replay for new features
// - No data loss

// ข้อเสีย:
// - Query complexity (need projections/read models)
// - Event schema evolution is hard
// - Eventually consistent read models
// - More storage needed
```

**Q3: Circuit Breaker ทำงานอย่างไร?**

```typescript
// A: State machine มี 3 states: CLOSED, OPEN, HALF_OPEN

class CircuitBreaker {
  private state: 'closed' | 'open' | 'half-open' = 'closed';
  private failures = 0;
  private readonly threshold = 5;
  private lastFailure?: Date;
  private readonly timeout = 30000; // 30 seconds

  async call<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === 'open') {
      const elapsed = Date.now() - (this.lastFailure?.getTime() ?? 0);
      if (elapsed > this.timeout) {
        this.state = 'half-open'; // Try again
      } else {
        throw new Error('Circuit breaker is OPEN - service unavailable');
      }
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  private onSuccess() {
    this.failures = 0;
    this.state = 'closed';
  }

  private onFailure() {
    this.failures++;
    this.lastFailure = new Date();
    if (this.failures >= this.threshold) {
      this.state = 'open'; // Trip the breaker
    }
  }
}
```

**Q4: อธิบาย Database per Service pattern และปัญหาที่อาจเกิดขึ้น**

```
A: แต่ละ service มี database ของตัวเอง

ข้อดี:
- Team autonomy: ทีมเลือก database ที่เหมาะสมได้
- Independent scaling: scale database ตาม service load
- Failure isolation: database หนึ่งล่ม ไม่กระทบ service อื่น

ความท้าทาย:
1. Distributed queries: ต้องใช้ API calls หรือ events แทน JOIN
2. Data consistency: eventual consistency แทน strong consistency
3. Data duplication: บาง data ต้อง copy ไปหลาย services
4. Transactions: ต้องใช้ Saga pattern

วิธีแก้:
- GraphQL Federation สำหรับ complex queries
- Event-driven sync สำหรับ data replication
- Saga สำหรับ distributed transactions
```

**Q5: อธิบาย CQRS pattern**

```typescript
// A: Command Query Responsibility Segregation
// แยก Read และ Write ออกจากกัน

// Command side: handles writes (mutations)
class CreateOrderCommand {
  constructor(
    public readonly userId: string,
    public readonly items: OrderItem[],
  ) {}
}

class OrderCommandHandler {
  async handle(cmd: CreateOrderCommand): Promise<string> {
    const order = Order.create(cmd.userId, cmd.items);
    await this.repository.save(order);
    await this.eventBus.publish(order.domainEvents);
    return order.id;
  }
}

// Query side: handles reads (denormalized view)
class OrderReadModel {
  id: string;
  userId: string;
  userName: string;  // denormalized from User
  status: string;
  items: Array<{
    productName: string;  // denormalized from Product
    quantity: number;
    price: number;
  }>;
  total: number;
}

class OrderQueryHandler {
  async getById(id: string): Promise<OrderReadModel> {
    // Query optimized read model (often separate database/cache)
    return this.readRepository.findById(id);
  }
}

// ใช้เมื่อ: read-heavy workloads, complex reporting,
// different scalability needs for reads vs writes
```

### Section 2: Operations

**Q6: อธิบาย Blue/Green vs Canary deployment**

```yaml
# Blue/Green: swap all traffic at once
# ✓ Fast rollback (just switch traffic back)
# ✓ Zero downtime
# ✗ Double infrastructure cost during switch
# ✗ All-or-nothing (no gradual validation)

# Current: Blue (v1) - 100% traffic
# Deploy:  Green (v2) - 0% traffic
# Test:    Green passes tests
# Switch:  Blue=0%, Green=100%
# Rollback: Green=0%, Blue=100% (instant)

# Canary: gradually shift traffic
# ✓ Gradual rollout with real traffic
# ✓ Can measure metrics on small %
# ✗ Longer to fully deploy
# ✗ Complexity in traffic management

# Step 1: v1=100%, v2=0%  → Deploy v2
# Step 2: v1=90%,  v2=10% → Watch metrics
# Step 3: v1=50%,  v2=50% → Still OK
# Step 4: v1=0%,   v2=100% → Done
```

**Q7: อธิบาย SLO, SLI, SLA และ Error Budget**

```typescript
// SLI (Service Level Indicator): metric ที่วัดได้จริง
const sli = {
  availability: 'percentage of requests with status < 500',
  latency: 'percentage of requests under 200ms',
};

// SLO (Service Level Objective): target เราตั้งเอง
const slo = {
  availability: 99.9,  // 99.9% of requests succeed
  latency: 95,         // 95% of requests < 200ms
};

// SLA (Service Level Agreement): legal commitment กับ customers
const sla = {
  availability: 99.5,  // lower than SLO (buffer)
};

// Error Budget: จำนวน errors ที่ยอมให้เกิดได้
function calculateErrorBudget(sloPercent: number, days = 30): string {
  const errorBudgetMinutes = days * 24 * 60 * (1 - sloPercent / 100);
  return `${errorBudgetMinutes.toFixed(0)} minutes of downtime allowed in ${days} days`;
}
// 99.9% SLO → 43.2 minutes per 30 days
// 99.5% SLO → 216 minutes per 30 days

// Burn rate: how fast are we consuming error budget?
// If burn rate > 1x: we'll exhaust budget exactly on time
// If burn rate > 14.4x: we'll exhaust in 2 hours (fast burn)
```

**Q8: อธิบายความแตกต่างระหว่าง liveness probe กับ readiness probe**

```yaml
# Liveness: Is the container alive? If not, restart it.
livenessProbe:
  httpGet:
    path: /health/live
    port: 3001
  initialDelaySeconds: 30  # wait 30s before first check
  periodSeconds: 10
  failureThreshold: 3       # restart after 3 consecutive failures

# Readiness: Is the container ready to receive traffic? If not, remove from load balancer.
readinessProbe:
  httpGet:
    path: /health/ready
    port: 3001
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 3       # remove from LB after 3 failures

# Startup: For slow-starting containers
startupProbe:
  httpGet:
    path: /health/live
    port: 3001
  failureThreshold: 30
  periodSeconds: 10         # up to 5 minutes for startup
```

**Q9: อธิบาย Kubernetes Resource Requests vs Limits**

```yaml
resources:
  requests:  # What the scheduler uses to place the pod
    cpu: "100m"     # 0.1 vCPU guaranteed
    memory: "128Mi" # 128MB guaranteed
  limits:    # Maximum the container can use
    cpu: "500m"     # 0.5 vCPU max (throttled if exceeded)
    memory: "512Mi" # 512MB max (OOMKilled if exceeded)

# Key rules:
# - CPU is compressible: throttled if over limit, not killed
# - Memory is incompressible: OOMKilled if over limit
# - requests must <= limits
# - QoS classes:
#   Guaranteed: requests == limits (stable, high priority)
#   Burstable:  requests < limits (normal)
#   BestEffort: no requests/limits (first evicted)
```

**Q10: อธิบาย Kubernetes Affinity และ Anti-Affinity**

```yaml
# Pod Anti-Affinity: spread pods across nodes/zones
affinity:
  podAntiAffinity:
    # Required: MUST be on different nodes
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
            - key: app
              operator: In
              values: [payment-service]
        topologyKey: kubernetes.io/hostname  # different node

    # Preferred: TRY to be in different zones
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: payment-service
          topologyKey: topology.kubernetes.io/zone  # different AZ

# Node Affinity: schedule on specific nodes
nodeAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
    nodeSelectorTerms:
      - matchExpressions:
          - key: kubernetes.io/arch
            operator: In
            values: [amd64]
```

### Section 3: Code Design

**Q11: ออกแบบ Rate Limiter สำหรับ API**

```typescript
// Token Bucket Algorithm
class RateLimiter {
  private readonly buckets = new Map<string, { tokens: number; lastRefill: number }>();
  
  constructor(
    private readonly maxTokens: number,
    private readonly refillRatePerSecond: number
  ) {}

  async isAllowed(clientId: string): Promise<{ allowed: boolean; remaining: number }> {
    const now = Date.now();
    let bucket = this.buckets.get(clientId);
    
    if (!bucket) {
      bucket = { tokens: this.maxTokens, lastRefill: now };
    }
    
    // Refill tokens based on elapsed time
    const elapsed = (now - bucket.lastRefill) / 1000;
    const tokensToAdd = elapsed * this.refillRatePerSecond;
    bucket.tokens = Math.min(this.maxTokens, bucket.tokens + tokensToAdd);
    bucket.lastRefill = now;
    
    if (bucket.tokens < 1) {
      this.buckets.set(clientId, bucket);
      return { allowed: false, remaining: 0 };
    }
    
    bucket.tokens -= 1;
    this.buckets.set(clientId, bucket);
    return { allowed: true, remaining: Math.floor(bucket.tokens) };
  }
}

// Usage as Express middleware
function rateLimitMiddleware(limiter: RateLimiter) {
  return async (req: Request, res: Response, next: NextFunction) => {
    const clientId = req.ip ?? 'unknown';
    const { allowed, remaining } = await limiter.isAllowed(clientId);
    
    res.setHeader('X-RateLimit-Remaining', remaining);
    
    if (!allowed) {
      return res.status(429).json({ error: 'Too Many Requests' });
    }
    next();
  };
}
```

**Q12: อธิบาย Idempotency และ implement**

```typescript
// Idempotency: calling the same operation N times = same result as calling once

class IdempotentOrderService {
  async createOrder(
    orderId: string,  // client-generated idempotency key
    data: CreateOrderData,
    userId: string
  ): Promise<Order> {
    // Check if already processed
    const existing = await this.db.query(
      'SELECT * FROM orders WHERE idempotency_key = $1 AND user_id = $2',
      [orderId, userId]
    );
    
    if (existing.rows[0]) {
      // Return same result (idempotent)
      return existing.rows[0];
    }
    
    // Process and save with idempotency key
    return this.db.transaction(async (tx) => {
      const order = await tx.query(
        `INSERT INTO orders (id, idempotency_key, user_id, status, ...)
         VALUES ($1, $2, $3, 'pending', ...)
         ON CONFLICT (idempotency_key, user_id) DO UPDATE
         SET updated_at = NOW()
         RETURNING *`,
        [uuidv4(), orderId, userId]
      );
      return order.rows[0];
    });
  }
}
```

**Q13: Design a health check endpoint**

```typescript
// Good health check: checks all critical dependencies
app.get('/health/live', (req, res) => {
  // Liveness: just check process is alive
  res.json({ status: 'alive', timestamp: new Date().toISOString() });
});

app.get('/health/ready', async (req, res) => {
  const checks: Record<string, 'ok' | 'error'> = {};
  let isReady = true;
  
  // Check database
  try {
    await pool.query('SELECT 1');
    checks.database = 'ok';
  } catch {
    checks.database = 'error';
    isReady = false;
  }
  
  // Check Redis
  try {
    await redis.ping();
    checks.redis = 'ok';
  } catch {
    checks.redis = 'error';
    isReady = false; // or just warn, not fail
  }
  
  // Check Kafka producer
  try {
    await producer.isConnected();
    checks.kafka = 'ok';
  } catch {
    checks.kafka = 'error';
    isReady = false;
  }
  
  const status = isReady ? 200 : 503;
  res.status(status).json({
    status: isReady ? 'ready' : 'not-ready',
    checks,
    timestamp: new Date().toISOString(),
    version: process.env.APP_VERSION,
  });
});
```

**Q14: อธิบาย Outbox Pattern**

```typescript
// Problem: How to atomically save to DB AND publish an event?
// Without Outbox, these can get out of sync:
await db.save(order);          // Succeeds
await kafka.publish(event);    // Fails! Event lost!

// Outbox Pattern: Save event to DB in same transaction
async function createOrderWithOutbox(data: OrderData) {
  return db.transaction(async (tx) => {
    // 1. Create the order
    const order = await tx.query(
      'INSERT INTO orders (...) VALUES (...) RETURNING *',
      [...]
    );
    
    // 2. Save event to outbox (same transaction = atomic!)
    await tx.query(
      'INSERT INTO outbox (id, event_type, aggregate_id, payload) VALUES ($1, $2, $3, $4)',
      [uuidv4(), 'order.created', order.id, JSON.stringify({ ...order.rows[0] })]
    );
    
    return order.rows[0];
  });
}

// Separate process polls outbox and publishes to Kafka
async function processOutbox() {
  const events = await db.query(
    'SELECT * FROM outbox WHERE published_at IS NULL ORDER BY created_at LIMIT 100'
  );
  
  for (const event of events.rows) {
    await kafka.publish(event.event_type, JSON.parse(event.payload));
    await db.query('UPDATE outbox SET published_at = NOW() WHERE id = $1', [event.id]);
  }
}
```

**Q15: อธิบาย Graceful Shutdown**

```typescript
// Graceful shutdown: finish current requests, clean up resources

const server = app.listen(3001);
let isShuttingDown = false;

// Mark as shutting down on SIGTERM
process.on('SIGTERM', async () => {
  console.log('SIGTERM received, starting graceful shutdown...');
  isShuttingDown = true;
  
  // Stop accepting new connections
  server.close(async () => {
    console.log('HTTP server closed');
    
    // Wait for in-flight requests (max 30s)
    await new Promise(resolve => setTimeout(resolve, 30000));
    
    // Close database connections
    await pool.end();
    console.log('Database connections closed');
    
    // Flush pending messages
    await producer.disconnect();
    console.log('Kafka producer disconnected');
    
    process.exit(0);
  });
  
  // Force exit after 60s
  setTimeout(() => process.exit(1), 60000);
});

// Add middleware to reject new requests during shutdown
app.use((req, res, next) => {
  if (isShuttingDown) {
    res.status(503).json({ error: 'Service shutting down' });
    return;
  }
  next();
});
```

---

## 100.10 คำอำลาและเส้นทางการเรียนรู้ต่อไป (Thai Farewell)

สวัสดีเพื่อนนักพัฒนาทุกท่าน

ตลอดระยะเวลา 100 บทที่ผ่านมา เราได้เดินทางร่วมกันจาก "Hello World" ของ microservices ไปจนถึงการสร้างระบบ production-grade ที่พร้อมรับมือกับ traffic จริงในโลก การเรียนรู้นี้ไม่ใช่แค่เรื่อง code แต่เป็นการสร้าง mindset ของวิศวกรที่ดี

### สิ่งที่คุณทำได้แล้ว

หลังจากเรียน 100 Parts นี้แล้ว คุณสามารถ:

- **ออกแบบ** ระบบ microservices ที่ scalable และ maintainable
- **สร้าง** services ด้วย TypeScript ที่มี production-quality code
- **Deploy** บน Kubernetes ด้วย best practices ด้าน security และ reliability
- **Monitor** ด้วย observability stack ที่ครบครัน (metrics, logs, traces)
- **Operate** ระบบใน production ด้วย incident management และ SLO tracking
- **Optimize** ทั้ง performance และ cost อย่างมีหลักการ
- **Lead** ทีมด้วยวัฒนธรรม DevOps และ blameless culture

### เส้นทางการเรียนรู้ต่อไป

ความรู้ใน microservices ไม่มีวันหยุดนิ่ง ต่อไปนี้คือสิ่งที่ควรศึกษาต่อ:

**1. Platform Engineering**
สร้าง Internal Developer Platform (IDP) เพื่อให้ทีมอื่น self-service ได้
→ ศึกษา Backstage, Crossplane, Upbound

**2. FinOps Maturity**
เพิ่ม maturity ด้าน cloud cost management
→ ศึกษา FinOps Framework, AWS Cost Intelligence Dashboard

**3. AI-Assisted Development**
LLMs และ AI tools เข้ามาเปลี่ยนวิธีทำงานของ engineers
→ ศึกษา GitHub Copilot, AI code review, AI-assisted ops

**4. WebAssembly (WASM)**
Runtime ใหม่ที่ lightweight กว่า containers
→ ศึกษา WasmEdge, WASI, Spin by Fermyon

**5. Distributed Database Deep Dive**
TiDB, CockroachDB, YugabyteDB สำหรับ globally distributed apps
→ ศึกษา Raft consensus, MVCC, distributed transactions

**6. Edge Computing**
Deploy microservices ที่ edge locations
→ ศึกษา Cloudflare Workers, Fastly Compute, AWS Lambda@Edge

### คำแนะนำสุดท้าย

```
สิ่งที่ทำให้วิศวกรที่ดีแตกต่างจากวิศวกรที่ยอดเยี่ยม:

1. อ่านอยู่เสมอ - Technology เปลี่ยนเร็ว คุณต้อง keep up
2. Build และ break things - Theory ไม่พอ ต้องลงมือทำจริง
3. แชร์ความรู้ - การสอนคือการเรียนรู้ที่ดีที่สุด
4. Embrace failures - ทุก production incident คือบทเรียนล้ำค่า
5. Think in systems - มองภาพใหญ่เสมอ ไม่ใช่แค่ code
6. ดูแลทีม - Software ดีสร้างโดยทีมที่ดี
7. Be humble - ยิ่งรู้มาก ยิ่งรู้ว่าไม่รู้อะไรอีกมาก
```

### ขอบคุณที่ร่วมเดินทาง

ขอบคุณที่อุทิศเวลาและความพยายามในการเรียนหลักสูตรนี้จนจบ การเรียนรู้ไม่มีวันจบ แต่ milestone ของวันนี้เป็นหลักฐานว่าคุณมีความมุ่งมั่นที่จะเป็นวิศวกรที่ดีขึ้นทุกวัน

จงเขียน code ที่ดี จงออกแบบระบบที่น่าเชื่อถือ จงดูแลเพื่อนร่วมทีม และจงไม่หยุดเรียนรู้

**"The best time to start was yesterday. The second best time is now."**

สู้ต่อไปนะครับ/ค่ะ! 🚀

---

## Appendix: Quick Reference Card

```bash
# Kubernetes Most Used Commands
kubectl get all -n <namespace>
kubectl describe pod <pod> -n <namespace>
kubectl logs <pod> -n <namespace> -f --tail=100
kubectl exec -it <pod> -n <namespace> -- /bin/sh
kubectl rollout restart deployment/<name> -n <namespace>
kubectl scale deployment/<name> --replicas=3 -n <namespace>
kubectl port-forward svc/<service> 8080:80 -n <namespace>
kubectl apply -f k8s/ --dry-run=client
kubectl diff -f k8s/
kubectl top pods -n <namespace> --sort-by=cpu

# Docker Commands
docker build -t myapp:1.0 --platform linux/amd64,linux/arm64 .
docker run --rm -it --env-file .env myapp:1.0
docker compose up -d --build
docker compose logs -f service-name
docker stats
docker system prune -af

# Git Commands
git log --oneline --graph --all -20
git stash push -m "WIP: feature X"
git bisect start HEAD v1.0.0
git cherry-pick <commit-hash>

# Kafka Commands
kafka-topics.sh --bootstrap-server kafka:9092 --list
kafka-consumer-groups.sh --bootstrap-server kafka:9092 --describe --group mygroup
kafka-console-consumer.sh --bootstrap-server kafka:9092 --topic orders.created --from-beginning

# Prometheus Queries
# Error rate
rate(http_requests_total{code=~"5.."}[5m]) / rate(http_requests_total[5m])
# P99 latency
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
# Memory usage per pod
container_memory_working_set_bytes{namespace="production"} / 1024 / 1024
```

---

## Appendix B: Advanced Interview Questions (Q16-Q30)

### Section 4: Database and Data Patterns

**Q16: อธิบาย N+1 problem และวิธีแก้ใน microservices**

```typescript
// N+1 Problem: 1 query to get list + N queries for each item's details

// ❌ N+1 Problem
async function getOrdersWithUsers(orderIds: string[]) {
  const orders = await db.query('SELECT * FROM orders WHERE id = ANY($1)', [orderIds]);
  
  // N queries! one per order
  for (const order of orders.rows) {
    order.user = await userServiceClient.get(`/users/${order.user_id}`);
  }
  return orders.rows;
}

// ✅ Solution 1: Batch loading (DataLoader pattern)
import DataLoader from 'dataloader';

const userLoader = new DataLoader(async (userIds: readonly string[]) => {
  // One batch request instead of N individual requests
  const users = await userServiceClient.post('/users/batch', { ids: userIds });
  const userMap = new Map(users.map((u: { id: string }) => [u.id, u]));
  return userIds.map(id => userMap.get(id));
});

async function getOrdersWithUsers(orderIds: string[]) {
  const orders = await db.query('SELECT * FROM orders WHERE id = ANY($1)', [orderIds]);
  
  // All user IDs loaded in ONE batch request
  const ordersWithUsers = await Promise.all(
    orders.rows.map(async (order) => ({
      ...order,
      user: await userLoader.load(order.user_id),
    }))
  );
  return ordersWithUsers;
}

// ✅ Solution 2: Denormalization - store user data in orders table
// When user updates, publish event → order service updates snapshot
// Trade off: some staleness acceptable, but no N+1
```

**Q17: อธิบาย Optimistic vs Pessimistic Locking**

```typescript
// Pessimistic Locking: lock row to prevent concurrent updates
async function updateInventoryPessimistic(productId: string, quantity: number) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    
    // Lock the row for update
    const result = await client.query(
      'SELECT * FROM inventory WHERE product_id = $1 FOR UPDATE',
      [productId]
    );
    
    const current = result.rows[0].quantity;
    if (current < quantity) {
      throw new Error('Insufficient inventory');
    }
    
    await client.query(
      'UPDATE inventory SET quantity = quantity - $1 WHERE product_id = $2',
      [quantity, productId]
    );
    
    await client.query('COMMIT');
  } catch (error) {
    await client.query('ROLLBACK');
    throw error;
  } finally {
    client.release();
  }
}

// Optimistic Locking: use version number, retry on conflict
async function updateInventoryOptimistic(productId: string, quantity: number, maxRetries = 3) {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    // Read current version
    const result = await pool.query(
      'SELECT quantity, version FROM inventory WHERE product_id = $1',
      [productId]
    );
    
    const { quantity: current, version } = result.rows[0];
    if (current < quantity) throw new Error('Insufficient inventory');
    
    // Update only if version hasn't changed
    const updateResult = await pool.query(
      `UPDATE inventory 
       SET quantity = quantity - $1, version = version + 1
       WHERE product_id = $2 AND version = $3`,
      [quantity, productId, version]
    );
    
    if (updateResult.rowCount > 0) return; // Success
    // Version changed = concurrent update, retry
    await new Promise(resolve => setTimeout(resolve, 50 * Math.pow(2, attempt)));
  }
  throw new Error('Too many concurrent updates, please retry');
}

// Pessimistic: better for high contention, but can cause deadlocks and reduce throughput
// Optimistic: better for low contention, high read scenarios
```

**Q18: อธิบาย Connection Pooling ใน microservices**

```typescript
// Connection pools ช่วย reuse database connections แทนการสร้างใหม่ทุก request

import { Pool, PoolConfig } from 'pg';

// Good pool configuration
const dbConfig: PoolConfig = {
  connectionString: process.env.DATABASE_URL,
  
  // Pool size: rule of thumb = CPU cores * 2-4 for IO-bound
  max: 20,           // Maximum connections in pool
  min: 2,            // Minimum connections to keep open
  
  // Timeouts
  idleTimeoutMillis: 30000,  // Remove idle connections after 30s
  connectionTimeoutMillis: 2000, // Fail fast if can't get connection in 2s
  
  // Health checking
  keepAlive: true,
  keepAliveInitialDelayMillis: 0,
};

const pool = new Pool(dbConfig);

// Monitor pool health
pool.on('connect', () => logger.debug('New DB connection created'));
pool.on('remove', () => logger.debug('DB connection removed'));
pool.on('error', (err) => logger.error('Unexpected error on DB client', { err }));

// Expose pool metrics to Prometheus
const poolGauge = new Gauge({
  name: 'pg_pool_connections',
  help: 'PostgreSQL connection pool size',
  labelNames: ['type'],
});

setInterval(() => {
  poolGauge.labels('total').set(pool.totalCount);
  poolGauge.labels('idle').set(pool.idleCount);
  poolGauge.labels('waiting').set(pool.waitingCount);
}, 5000);

// PgBouncer for connection pooling at infrastructure level
// - Session mode: one server connection per client session
// - Transaction mode: server connection released after each transaction (best for microservices)
// - Statement mode: server connection released after each statement
```

### Section 5: Kafka and Messaging

**Q19: อธิบาย Kafka Consumer Groups และ Partition Assignment**

```typescript
// Consumer Group: multiple consumers sharing work from a topic
// Each partition is assigned to exactly one consumer in a group

// Topic: orders.created (8 partitions)
// Consumer Group: order-processor (3 instances)
// Assignment:
//   consumer-1: partition 0, 1, 2
//   consumer-2: partition 3, 4, 5
//   consumer-3: partition 6, 7

const consumer = kafka.consumer({
  groupId: 'order-processor',
  
  // Session timeout: if heartbeat not received within this, consumer is dead
  sessionTimeout: 30000,
  
  // Heartbeat interval: should be 1/3 of sessionTimeout
  heartbeatInterval: 3000,
  
  // Max amount of data fetched per partition per request
  maxBytesPerPartition: 1048576, // 1MB
});

await consumer.subscribe({ topics: ['orders.created'], fromBeginning: false });

await consumer.run({
  // Process one message at a time (for ordered processing)
  eachMessage: async ({ topic, partition, message }) => {
    const order = JSON.parse(message.value!.toString());
    
    logger.info('Processing order', {
      orderId: order.id,
      partition,
      offset: message.offset,
    });
    
    await processOrder(order);
    
    // Commit offset (manual for at-least-once processing)
    await consumer.commitOffsets([{
      topic,
      partition,
      offset: (BigInt(message.offset) + 1n).toString(),
    }]);
  },
  
  // OR: process batch for higher throughput
  eachBatch: async ({ batch }) => {
    const orders = batch.messages.map(m => JSON.parse(m.value!.toString()));
    await processBatch(orders);
  },
  
  autoCommit: false, // Manual commit for reliability
});

// Rebalancing happens when:
// - Consumer joins group
// - Consumer leaves group (crash or graceful)
// - Partitions added to topic
// - Consumer group coordinator changes
```

**Q20: Exactly-Once Semantics ใน Kafka**

```typescript
// Delivery guarantees:
// At-most-once: may lose messages (autoCommit before processing)
// At-least-once: may duplicate (commit after processing, retry on failure)
// Exactly-once: no loss, no duplicates (hardest)

// Exactly-once with Kafka Transactions
const producer = kafka.producer({
  idempotent: true,           // Enable idempotent producer (no duplicates from retries)
  transactionalId: 'order-processor-tx-1', // Required for transactions
  maxInFlightRequests: 1,     // Required for idempotent
});

await producer.connect();
await producer.initTransactions();

async function processWithExactlyOnce(
  consumer: Consumer,
  messages: EachBatchPayload
) {
  await producer.transaction(async (tx) => {
    // Process messages
    const results = messages.batch.messages.map(m => 
      processMessage(JSON.parse(m.value!.toString()))
    );
    
    // Publish results
    await tx.send({
      topic: 'orders.processed',
      messages: results.map(r => ({ value: JSON.stringify(r) })),
    });
    
    // Commit offsets as part of transaction
    await tx.sendOffsets({
      consumerGroupId: 'order-processor',
      topics: [{
        topic: messages.batch.topic,
        partitions: [{
          partition: messages.batch.partition,
          offset: (
            BigInt(messages.batch.messages[messages.batch.messages.length - 1].offset) + 1n
          ).toString(),
        }],
      }],
    });
  });
}
```

### Section 6: Kubernetes Advanced

**Q21: อธิบาย Kubernetes RBAC**

```yaml
# Role: permissions within a namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "update", "patch"]

---
# ClusterRole: cluster-wide permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]

---
# RoleBinding: assign role to service account
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: production
subjects:
  - kind: ServiceAccount
    name: payment-service
    namespace: production
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

```typescript
// TypeScript: Using Kubernetes service account in-cluster
import * as k8s from '@kubernetes/client-node';

const kc = new k8s.KubeConfig();
// Inside a pod, this loads the service account token automatically
kc.loadFromCluster();

const k8sApi = kc.makeApiClient(k8s.CoreV1Api);

// Now k8sApi will use the pod's service account permissions
const pods = await k8sApi.listNamespacedPod('production');
```

**Q22: Kubernetes Pod Disruption Budget (PDB) คืออะไร?**

```yaml
# PDB ensures minimum availability during voluntary disruptions
# (node drains, rolling updates, etc.)

apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: payment-service-pdb
  namespace: production
spec:
  minAvailable: 2           # At least 2 pods must be available at all times
  # OR: maxUnavailable: 1   # At most 1 pod can be unavailable at a time
  selector:
    matchLabels:
      app: payment-service

# This means:
# - If we have 3 replicas: Kubernetes can only drain 1 node at a time
# - kubectl drain will wait until another pod is scheduled before proceeding
# - Prevents: all pods ending up on one node, temporary service unavailability

# Important: PDB only applies to VOLUNTARY disruptions
# - kubectl drain (voluntary)
# - Rolling updates (voluntary)
# - NOT: hardware failure, OOMKill (involuntary)
```

**Q23: อธิบาย Kubernetes ConfigMap vs Secret**

```yaml
# ConfigMap: non-sensitive configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  APP_ENV: "production"
  LOG_LEVEL: "info"
  DATABASE_HOST: "postgres.production.svc.cluster.local"
  config.yaml: |   # Multi-line file
    server:
      port: 8080
    cache:
      ttl: 300

---
# Secret: sensitive data (base64 encoded, not encrypted by default!)
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: production
type: Opaque
stringData:  # Plain text (auto-encodes to base64)
  JWT_SECRET: "your-super-secret-key-here"
  DATABASE_PASSWORD: "db-password-here"
  STRIPE_API_KEY: "sk_live_..."

# IMPORTANT: Kubernetes Secrets are only base64 encoded, NOT encrypted!
# For real security use:
# 1. Sealed Secrets (encrypted with public key)
# 2. AWS Secrets Manager / Azure Key Vault with External Secrets Operator
# 3. HashiCorp Vault with Vault Agent Injector
```

**Q24: อธิบาย StatefulSet vs Deployment**

```yaml
# Deployment: stateless, pods are interchangeable
# StatefulSet: stateful, pods have stable identity

# StatefulSet characteristics:
# 1. Stable, unique network identity: pod-0, pod-1, pod-2
# 2. Stable, persistent storage per pod
# 3. Ordered creation and deletion (0 first, N last)
# 4. Ordered, graceful rolling updates

apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres-cluster
spec:
  serviceName: "postgres"  # Headless service for DNS
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:15
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
  # Each pod gets its own PVC
  volumeClaimTemplates:
    - metadata:
        name: postgres-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi
# DNS: postgres-0.postgres.production.svc.cluster.local
#      postgres-1.postgres.production.svc.cluster.local
```

### Section 7: Security

**Q25: อธิบาย JWT token structure และ security considerations**

```typescript
// JWT = Header.Payload.Signature
// Header: algorithm and token type
// Payload: claims (user data)
// Signature: HMAC-SHA256 of header + payload + secret

// ✅ Security best practices for JWT
const tokenConfig = {
  // Algorithm: prefer RS256 (asymmetric) over HS256 (symmetric)
  // RS256: private key signs, public key verifies - safer for distributed systems
  algorithm: 'RS256' as const,
  
  // Short expiry for access tokens
  accessTokenExpiry: '15m',
  
  // Longer expiry for refresh tokens (rotated)
  refreshTokenExpiry: '7d',
  
  // Include essential claims only - don't put sensitive data in JWT!
  // (payload is base64 encoded, anyone can decode it)
};

// ❌ Don't put sensitive info in JWT payload
const badPayload = {
  userId: '123',
  email: 'user@example.com',
  creditCardNumber: '4111...',  // NEVER!
  password: 'hashed...',        // NEVER!
};

// ✅ Minimal payload
const goodPayload = {
  sub: '123',           // subject (user ID)
  iat: Math.floor(Date.now() / 1000),
  exp: Math.floor(Date.now() / 1000) + 900, // 15 minutes
  jti: uuidv4(),        // unique token ID (for revocation)
  role: 'CUSTOMER',
};

// Token revocation with blacklist
async function revokeToken(jti: string, expiry: number) {
  const ttl = expiry - Math.floor(Date.now() / 1000);
  if (ttl > 0) {
    await redis.setEx(`revoked:${jti}`, ttl, '1');
  }
}

async function isTokenRevoked(jti: string): Promise<boolean> {
  return !!(await redis.get(`revoked:${jti}`));
}
```

**Q26: อธิบาย OWASP Top 10 ที่เกี่ยวข้องกับ microservices**

```typescript
// 1. Broken Access Control - ตรวจสอบ authorization ทุก endpoint
app.get('/orders/:id', authenticate, async (req, res) => {
  const order = await getOrder(req.params.id);
  
  // ❌ IDOR (Insecure Direct Object Reference)
  // res.json(order);
  
  // ✅ Check ownership
  if (order.userId !== req.user.id && req.user.role !== 'ADMIN') {
    return res.status(403).json({ error: 'Forbidden' });
  }
  res.json(order);
});

// 2. Injection - use parameterized queries
// ❌ SQL injection vulnerable
const query = `SELECT * FROM users WHERE email = '${email}'`;

// ✅ Parameterized
const result = await pool.query('SELECT * FROM users WHERE email = $1', [email]);

// 3. Security Misconfiguration
app.use(helmet()); // Sets security headers
app.use(cors({ origin: ['https://myapp.com'], credentials: true }));
// Never expose stack traces in production
app.use((err: Error, req: Request, res: Response) => {
  logger.error(err);
  res.status(500).json({ error: 'Internal Server Error' }); // No stack trace!
});

// 4. Sensitive Data Exposure
// Always use HTTPS, hash passwords, encrypt sensitive fields
const hashedPassword = await bcrypt.hash(password, 12);

// 5. Rate Limiting - prevent brute force
app.use('/auth/login', rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5, // 5 attempts per 15 minutes
  standardHeaders: true,
}));
```

**Q27: อธิบาย CORS และ configure correctly**

```typescript
import cors from 'cors';

// ❌ Too permissive
app.use(cors()); // Allows ALL origins

// ✅ Restrictive CORS
const allowedOrigins = [
  'https://myapp.com',
  'https://admin.myapp.com',
  process.env.NODE_ENV === 'development' ? 'http://localhost:3000' : '',
].filter(Boolean);

app.use(cors({
  origin: (origin, callback) => {
    // Allow requests with no origin (server-to-server, curl)
    if (!origin) return callback(null, true);
    
    if (allowedOrigins.includes(origin)) {
      callback(null, true);
    } else {
      callback(new Error(`Origin ${origin} not allowed by CORS`));
    }
  },
  credentials: true,           // Allow cookies
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization', 'X-Request-ID'],
  exposedHeaders: ['X-Total-Count', 'X-Rate-Limit-Remaining'],
  maxAge: 86400,               // Cache preflight for 24 hours
}));

// For service-to-service: CORS not needed (same network)
// Only needed for browser → service calls
```

### Section 8: System Design

**Q28: Design a notification system for e-commerce**

```typescript
// Requirements: send email, SMS, push notifications
// Scale: 1M users, 10K notifications/second

// Architecture:
// Producer Services → Kafka → Notification Service → Channels

interface NotificationRequest {
  userId: string;
  type: 'order_confirmed' | 'payment_failed' | 'shipping_update' | 'promotion';
  channels: ('email' | 'sms' | 'push')[];
  templateId: string;
  data: Record<string, string>;
  priority: 'high' | 'normal' | 'low';
  scheduledAt?: Date;
}

// Fan-out pattern: one message → multiple delivery attempts
class NotificationOrchestrator {
  async processNotification(req: NotificationRequest) {
    // Get user preferences
    const prefs = await this.getUserPreferences(req.userId);
    
    // Filter channels based on preferences
    const activeChannels = req.channels.filter(ch => prefs.enabled[ch]);
    
    // Publish to channel-specific queues
    await Promise.all(
      activeChannels.map(channel =>
        this.kafka.publish(`notifications.${channel}`, {
          ...req,
          channel,
        })
      )
    );
  }
}

// Separate consumers per channel with different scaling
// Email: lower throughput, HTML rendering needed
// SMS: rate limited by carrier (expensive)
// Push: high throughput, stateless, fire-and-forget

// Retry strategy with exponential backoff
class EmailNotificationConsumer {
  async handleWithRetry(notification: NotificationRequest, attempt = 1) {
    try {
      await this.emailProvider.send({
        to: notification.userId,
        template: notification.templateId,
        data: notification.data,
      });
    } catch (error) {
      if (attempt < 3) {
        await sleep(Math.pow(2, attempt) * 1000);
        return this.handleWithRetry(notification, attempt + 1);
      }
      // Dead letter queue for failed notifications
      await this.dlq.publish('notifications.email.dlq', notification);
    }
  }
}
```

**Q29: Design URL Shortener as a microservice**

```typescript
// URL Shortener: myurl.io/abc123 → https://very-long-url.com/...

// Components:
// 1. Shortener Service (write path)
// 2. Redirect Service (read path, high throughput)
// 3. Analytics Service (async, via Kafka)

// ID Generation: Base62 encoding of counter or hash
function generateShortCode(url: string, userId: string): string {
  // Option 1: Hash-based (deterministic but collision possible)
  const hash = crypto.createHash('sha256')
    .update(url + userId)
    .digest('hex');
  return base62.encode(parseInt(hash.substring(0, 8), 16)).substring(0, 7);
  
  // Option 2: Auto-increment counter (distributed counter via Redis)
}

class URLShortenerService {
  async shorten(longUrl: string, userId: string, options?: { expiresAt?: Date; customCode?: string }) {
    const shortCode = options?.customCode ?? generateShortCode(longUrl, userId);
    
    // Store with TTL
    const ttl = options?.expiresAt
      ? Math.floor((options.expiresAt.getTime() - Date.now()) / 1000)
      : 365 * 24 * 3600; // Default 1 year
    
    await this.redis.setEx(
      `url:${shortCode}`,
      ttl,
      JSON.stringify({ longUrl, userId, createdAt: new Date() })
    );
    
    // Also persist to DB for durability
    await this.db.query(
      'INSERT INTO shortened_urls (code, long_url, user_id, expires_at) VALUES ($1, $2, $3, $4)',
      [shortCode, longUrl, userId, options?.expiresAt]
    );
    
    return `https://myurl.io/${shortCode}`;
  }
}

class RedirectService {
  async redirect(req: Request, res: Response) {
    const { code } = req.params;
    
    // Redis lookup (O(1), very fast)
    const data = await this.redis.get(`url:${code}`);
    
    if (data) {
      const { longUrl } = JSON.parse(data);
      
      // Async analytics event (non-blocking)
      setImmediate(() => {
        this.kafka.publish('url.clicked', {
          code,
          ip: req.ip,
          userAgent: req.headers['user-agent'],
          referer: req.headers.referer,
          timestamp: new Date().toISOString(),
        });
      });
      
      return res.redirect(301, longUrl); // 301 for permanent, 302 for temporary
    }
    
    res.status(404).json({ error: 'URL not found or expired' });
  }
}

// Scale: Redis can handle 100K+ reads/second
// Analytics processed asynchronously via Kafka
```

**Q30: How to handle backward compatibility when changing APIs?**

```typescript
// Breaking vs Non-breaking changes

// ✅ Non-breaking changes (safe to deploy without versioning):
// - Adding new optional fields to response
// - Adding new optional request parameters
// - Adding new endpoints
// - Adding new enum values (careful with strict clients)

// ❌ Breaking changes (require versioning):
// - Removing fields from response
// - Changing field types
// - Renaming fields
// - Changing behavior of existing endpoints
// - Making optional fields required

// Strategy 1: URL versioning
// GET /api/v1/users/:id → old response format
// GET /api/v2/users/:id → new response format

// Strategy 2: Content negotiation
app.get('/users/:id', async (req, res) => {
  const version = req.headers['accept-version'] ?? 'v1';
  const user = await userService.getUser(req.params.id);
  
  if (version === 'v2') {
    res.json({
      id: user.id,
      profile: {          // v2: nested profile
        firstName: user.firstName,
        lastName: user.lastName,
        email: user.email,
      },
    });
  } else {
    res.json({
      id: user.id,        // v1: flat structure
      firstName: user.firstName,
      lastName: user.lastName,
      email: user.email,
    });
  }
});

// Strategy 3: Field deprecation with sunset header
app.get('/users/:id', async (req, res) => {
  const user = await userService.getUser(req.params.id);
  
  res.set('Deprecation', 'version="v1"');
  res.set('Sunset', 'Sat, 1 Jan 2026 00:00:00 GMT');
  res.set('Link', '</api/v2/users>; rel="successor-version"');
  
  res.json({
    ...user,
    // Keep old field for backward compat
    name: `${user.firstName} ${user.lastName}`, // deprecated
    // New field
    firstName: user.firstName,
    lastName: user.lastName,
  });
});

// Contract Testing prevents breaking changes from reaching production!
// Use Pact to enforce consumer contracts
```

---

## Appendix C: Cheat Sheet - TypeScript Patterns

```typescript
// Pattern 1: Result type (avoid try/catch everywhere)
type Result<T, E = Error> =
  | { success: true; data: T }
  | { success: false; error: E };

async function safeGetUser(id: string): Promise<Result<User>> {
  try {
    const user = await userRepository.findById(id);
    if (!user) return { success: false, error: new Error('User not found') };
    return { success: true, data: user };
  } catch (error) {
    return { success: false, error: error as Error };
  }
}

// Pattern 2: Builder pattern for complex objects
class QueryBuilder {
  private table: string = '';
  private conditions: string[] = [];
  private params: unknown[] = [];
  private limitValue?: number;
  private offsetValue?: number;

  from(table: string) { this.table = table; return this; }
  where(condition: string, param: unknown) {
    this.conditions.push(condition.replace('?', `$${this.params.length + 1}`));
    this.params.push(param);
    return this;
  }
  limit(n: number) { this.limitValue = n; return this; }
  offset(n: number) { this.offsetValue = n; return this; }

  build(): { text: string; values: unknown[] } {
    let text = `SELECT * FROM ${this.table}`;
    if (this.conditions.length) text += ` WHERE ${this.conditions.join(' AND ')}`;
    if (this.limitValue) text += ` LIMIT ${this.limitValue}`;
    if (this.offsetValue) text += ` OFFSET ${this.offsetValue}`;
    return { text, values: this.params };
  }
}

// Usage
const query = new QueryBuilder()
  .from('orders')
  .where('user_id = ?', userId)
  .where('status = ?', 'pending')
  .limit(10)
  .offset(20)
  .build();

// Pattern 3: Retry with exponential backoff
async function withRetry<T>(
  fn: () => Promise<T>,
  options: { maxRetries?: number; baseDelay?: number } = {}
): Promise<T> {
  const { maxRetries = 3, baseDelay = 100 } = options;
  
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      if (attempt === maxRetries) throw error;
      
      const jitter = Math.random() * baseDelay;
      const delay = Math.pow(2, attempt - 1) * baseDelay + jitter;
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
  throw new Error('Max retries reached'); // never
}

// Pattern 4: Cache decorator
function cache(ttlSeconds: number) {
  return function <T>(
    target: unknown,
    propertyKey: string,
    descriptor: TypedPropertyDescriptor<(...args: unknown[]) => Promise<T>>
  ) {
    const original = descriptor.value!;
    const cacheStore = new Map<string, { value: T; expiry: number }>();
    
    descriptor.value = async function (...args: unknown[]) {
      const key = JSON.stringify(args);
      const cached = cacheStore.get(key);
      
      if (cached && cached.expiry > Date.now()) {
        return cached.value;
      }
      
      const value = await original.apply(this, args);
      cacheStore.set(key, { value, expiry: Date.now() + ttlSeconds * 1000 });
      return value;
    };
  };
}

class ProductService {
  @cache(300) // Cache for 5 minutes
  async getProduct(id: string) {
    return this.db.query('SELECT * FROM products WHERE id = $1', [id]);
  }
}
```

---

*หลักสูตร Microservices 100 Parts เสร็จสมบูรณ์แล้ว ขอให้โชคดีในการ build ระบบที่ยอดเยี่ยมต่อไป!*

---

## Appendix B: PromQL Cheat Sheet สำหรับ Microservices

```promql
# ===== HTTP Metrics =====

# Request rate (per service, per status)
sum(rate(http_requests_total{job="order-service"}[5m])) by (status_code)

# Error rate percentage
100 * (
  sum(rate(http_requests_total{job="order-service", status_code=~"5.."}[5m]))
  /
  sum(rate(http_requests_total{job="order-service"}[5m]))
)

# P50, P95, P99 latency
histogram_quantile(0.99,
  sum(rate(http_request_duration_seconds_bucket{job="order-service"}[5m])) by (le)
)

# Requests per second
sum(rate(http_requests_total[1m]))

# ===== Kubernetes Metrics =====

# CPU usage per pod (cores)
sum(rate(container_cpu_usage_seconds_total{namespace="production", container!=""}[5m])) by (pod)

# Memory usage per pod (MB)
sum(container_memory_working_set_bytes{namespace="production", container!=""}) by (pod) / 1024 / 1024

# Pod restart count
kube_pod_container_status_restarts_total{namespace="production"} > 5

# Node CPU pressure
1 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance)

# ===== Kafka Metrics =====

# Consumer lag per group and topic
kafka_consumer_group_lag{consumergroup="order-saga-group"}

# Messages in per second per topic
rate(kafka_topic_partition_current_offset[5m])

# ===== Database Metrics =====

# PostgreSQL active connections
pg_stat_activity_count{state="active"}

# Query execution time P99
histogram_quantile(0.99, rate(pg_query_duration_seconds_bucket[5m]))

# Redis hit rate
rate(redis_keyspace_hits_total[5m]) /
(rate(redis_keyspace_hits_total[5m]) + rate(redis_keyspace_misses_total[5m]))

# ===== SLO Metrics =====

# Availability SLO (99.9% = 0.999)
1 - (
  sum(rate(http_requests_total{status_code=~"5.."}[30d]))
  /
  sum(rate(http_requests_total[30d]))
)

# Error budget remaining (percent)
(
  1 - (
    sum(rate(http_requests_total{status_code=~"5.."}[30d]))
    /
    sum(rate(http_requests_total[30d]))
  )
) / (1 - 0.999) * 100
```

---

## Appendix C: ตัวอย่าง AlertManager Rules ครบถ้วน

```yaml
# alertmanager-rules.yaml
groups:
  - name: microservices-critical
    rules:
      # Service is completely down
      - alert: ServiceDown
        expr: up{job=~".*-service"} == 0
        for: 1m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "Service {{ $labels.job }} is down"
          description: "{{ $labels.job }} has been down for more than 1 minute"
          runbook: "https://wiki.example.com/runbooks/service-down"

      # High error rate
      - alert: HighErrorRate
        expr: |
          (
            sum by (job) (rate(http_requests_total{status_code=~"5.."}[5m]))
            /
            sum by (job) (rate(http_requests_total[5m]))
          ) > 0.05
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "High error rate on {{ $labels.job }}: {{ $value | humanizePercentage }}"
          description: "Error rate exceeds 5% for 2 minutes"

      # High P99 latency
      - alert: HighLatencyP99
        expr: |
          histogram_quantile(0.99,
            sum by (job, le) (rate(http_request_duration_seconds_bucket[5m]))
          ) > 2.0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "P99 latency {{ $value }}s on {{ $labels.job }}"

      # Kafka consumer lag
      - alert: KafkaConsumerLagHigh
        expr: kafka_consumer_group_lag > 10000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Kafka lag {{ $value }} for group {{ $labels.consumergroup }}"

      # Pod crash looping
      - alert: PodCrashLooping
        expr: |
          increase(kube_pod_container_status_restarts_total[15m]) > 3
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Pod {{ $labels.pod }} is crash looping"

      # Database connection pool exhausted
      - alert: DBConnectionPoolExhausted
        expr: |
          pg_stat_activity_count{state="active"} /
          pg_settings_max_connections > 0.8
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "DB connection pool {{ $value | humanizePercentage }} utilized"

      # Low disk space
      - alert: LowDiskSpace
        expr: |
          (node_filesystem_avail_bytes{mountpoint="/"}
          / node_filesystem_size_bytes{mountpoint="/"}) < 0.15
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Low disk space {{ $value | humanizePercentage }} remaining on {{ $labels.instance }}"

      # SLO Burn Rate (Fast Burn: 2% budget in 1 hour)
      - alert: SLOFastBurn
        expr: |
          (
            1 - (
              sum(rate(http_requests_total{status_code!~"5.."}[1h]))
              /
              sum(rate(http_requests_total[1h]))
            )
          ) > 14.4 * (1 - 0.999)
        for: 2m
        labels:
          severity: critical
          slo: availability
        annotations:
          summary: "SLO fast burn rate detected - 2% budget in 1 hour"
          description: "At this rate, monthly error budget will be exhausted in < 1 hour"
```

---

## Appendix D: Docker Compose สำหรับ Local Development ครบชุด

```yaml
# docker-compose.dev.yaml
version: '3.9'

networks:
  microservices-net:
    driver: bridge

volumes:
  postgres-data:
  redis-data:
  kafka-data:
  elasticsearch-data:
  prometheus-data:
  grafana-data:

services:
  # ===================
  # Infrastructure
  # ===================
  
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: localpassword
      POSTGRES_USER: admin
      POSTGRES_DB: microservices_dev
    ports:
      - "5432:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./scripts/init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin"]
      interval: 5s
      timeout: 5s
      retries: 5
    networks:
      - microservices-net

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes --maxmemory 512mb --maxmemory-policy allkeys-lru
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
    networks:
      - microservices-net

  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
    networks:
      - microservices-net

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"
    healthcheck:
      test: ["CMD", "kafka-broker-api-versions", "--bootstrap-server", "localhost:9092"]
      interval: 10s
      retries: 5
    networks:
      - microservices-net

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    ports:
      - "8090:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092
    depends_on:
      - kafka
    networks:
      - microservices-net

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ports:
      - "9200:9200"
    volumes:
      - elasticsearch-data:/usr/share/elasticsearch/data
    networks:
      - microservices-net

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    ports:
      - "5601:5601"
    environment:
      ELASTICSEARCH_HOSTS: http://elasticsearch:9200
    depends_on:
      - elasticsearch
    networks:
      - microservices-net

  # ===================
  # Observability Stack
  # ===================

  prometheus:
    image: prom/prometheus:v2.47.0
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./monitoring/rules:/etc/prometheus/rules
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=7d'
    networks:
      - microservices-net

  grafana:
    image: grafana/grafana:10.2.0
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
      GF_AUTH_ANONYMOUS_ENABLED: "true"
    volumes:
      - grafana-data:/var/lib/grafana
      - ./monitoring/grafana/provisioning:/etc/grafana/provisioning
    networks:
      - microservices-net

  jaeger:
    image: jaegertracing/all-in-one:1.51
    ports:
      - "16686:16686"   # Jaeger UI
      - "4317:4317"     # OTLP gRPC
      - "4318:4318"     # OTLP HTTP
    environment:
      COLLECTOR_OTLP_ENABLED: "true"
    networks:
      - microservices-net

  mailhog:
    image: mailhog/mailhog:latest
    ports:
      - "1025:1025"     # SMTP
      - "8025:8025"     # Web UI
    networks:
      - microservices-net
```

---

## สรุปสุดท้าย (Final Summary Table)

| หัวข้อ | สิ่งที่ได้เรียนรู้ | เครื่องมือสำคัญ |
|--------|-----------------|----------------|
| สรุปคอร์ส 100 ตอน | ภาพรวมทุก part ตั้งแต่ต้นจนจบ | - |
| Knowledge Map | แผนที่ความรู้ครบถ้วน 5 ด้าน | Architecture, Dev, Security, Ops, Advanced |
| Top 20 Patterns | Patterns สำคัญพร้อม code ตัวอย่าง | Circuit Breaker, Saga, CQRS, Outbox |
| Anti-Patterns | 5 ข้อผิดพลาดที่พบบ่อยและวิธีแก้ไข | Avoid Distributed Monolith |
| Career Path | เส้นทาง Junior→Architect พร้อมเงินเดือน | 35K-500K THB |
| Books & Resources | 10 books, 10 blogs, 5 conferences | Building Microservices, SRE Book |
| Certifications | CKAD, CKA, CKS, AWS SA Pro | killer.sh simulator |
| Open Source | วิธี contribute ให้ Kubernetes, Istio | Good first issue |
| Interview Q&A | 15 คำถาม-คำตอบพร้อม TypeScript code | Saga, CQRS, Circuit Breaker |
| PromQL Cheat Sheet | Queries สำหรับ HTTP, K8s, Kafka, DB | P99 latency, error rate, SLO |
| AlertManager Rules | Alert rules ครบสำหรับ microservices | SLO burn rate, pod crashloop |
| Docker Compose Dev | Infrastructure stack สำหรับ local dev | Kafka, Elasticsearch, Jaeger |
| Quick Reference | Commands สำหรับ kubectl, docker, kafka | kubectl rollout, kafka-consumer-groups |
| Production Checklist | 40+ items ตรวจสอบ production readiness | Security, Reliability, Observability |
| TypeScript Patterns | Result type, Builder, Decorator, Factory | Best practices for microservices code |

---

*หลักสูตร Microservices ด้วย TypeScript — 100 Parts ครบสมบูรณ์*

*"Code with purpose. Build with care. Ship with confidence."*

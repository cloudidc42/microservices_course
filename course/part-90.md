# Part 90: Microservices Maturity Model

## บทนำ

Microservices Maturity Model เป็นกรอบการประเมินความสามารถขององค์กรในการพัฒนาและดำเนินงาน microservices อย่างมีประสิทธิภาพ การรู้ว่าองค์กรอยู่ที่ระดับใด จะช่วยให้วางแผนการพัฒนาได้ตรงจุดและมีประสิทธิผลมากขึ้น

---

## 90.1 The 5 Maturity Levels

### Level 1: Developer (ระดับนักพัฒนา)

**คำอธิบาย**: ทีมเพิ่งเริ่มต้นกับ microservices มี service แยกกันบ้างแล้ว แต่ยังขาด infrastructure support ที่ดี

**ลักษณะเฉพาะ**:
- Deploy แบบ manual บน VMs หรือ bare metal
- ไม่มี centralized logging หรือ monitoring
- Communication ระหว่าง service แบบ REST แต่ไม่มี circuit breaker
- ไม่มี service discovery - ใช้ hardcoded IP addresses
- Testing แบบ manual หรือมี unit tests บางส่วนเท่านั้น
- ไม่มี CI/CD pipeline ที่สมบูรณ์
- Team ownership ไม่ชัดเจน - ทีมเดียวดูแลทุก service

**ตัวชี้วัด**:
- Deployment frequency: รายสัปดาห์หรือนานกว่านั้น
- MTTR: > 4 ชั่วโมง
- Change failure rate: > 30%
- Lead time for changes: > 1 สัปดาห์

**ความเสี่ยง**: ระบบ fragile มาก การ deploy แต่ละครั้งกลัวว่าจะพัง

---

### Level 2: Operational (ระดับการดำเนินงาน)

**คำอธิบาย**: มี basic infrastructure พร้อมกว่า Level 1 มีกระบวนการ deployment ที่เป็นระบบมากขึ้น

**ลักษณะเฉพาะ**:
- Container-based deployment (Docker) แต่อาจยังไม่ใช้ orchestration เต็มรูปแบบ
- มี basic CI/CD pipeline (build → test → deploy)
- มี centralized logging (ELK stack หรือ similar)
- มี basic health checks และ alerting
- มี service discovery (Consul หรือ Kubernetes DNS)
- มี API gateway สำหรับ external traffic
- Team ownership เริ่มชัดขึ้น แต่ยังมี shared services บางส่วน

**ตัวชี้วัด**:
- Deployment frequency: รายวัน
- MTTR: 1-4 ชั่วโมง
- Change failure rate: 15-30%
- Lead time for changes: 1-7 วัน

---

### Level 3: Scaled (ระดับขยายตัว)

**คำอธิบาย**: ระบบ production-ready พร้อม infrastructure ที่แข็งแกร่ง สามารถ scale ได้ตามความต้องการ

**ลักษณะเฉพาะ**:
- Kubernetes orchestration เต็มรูปแบบ
- Service mesh (Istio/Linkerd) สำหรับ observability และ security
- Distributed tracing (Jaeger/Zipkin)
- Event-driven architecture บางส่วน (Kafka/RabbitMQ)
- Automated scaling (HPA, KEDA)
- Blue/green หรือ canary deployments
- Feature flags
- Chaos engineering เบื้องต้น

**ตัวชี้วัด**:
- Deployment frequency: หลายครั้งต่อวัน
- MTTR: 30-60 นาที
- Change failure rate: 5-15%
- Lead time for changes: < 1 วัน

---

### Level 4: Optimized (ระดับเพิ่มประสิทธิภาพ)

**คำอธิบาย**: ระบบมีประสิทธิภาพสูง ทีม operate ได้อย่างอิสระ มีการวัดผลและปรับปรุงอย่างต่อเนื่อง

**ลักษณะเฉพาะ**:
- Full automated testing pyramid (unit, integration, contract, e2e)
- Advanced observability: SLOs, error budgets
- Self-healing systems (auto rollback, auto scaling)
- GitOps สำหรับ infrastructure management
- Security as Code (policy enforcement, automated scanning)
- FinOps - cost monitoring และ optimization
- Team topologies aligned กับ Conway's Law
- Internal developer platform (IDP)

**ตัวชี้วัด**:
- Deployment frequency: On-demand (multiple per day)
- MTTR: < 30 นาที
- Change failure rate: < 5%
- Lead time for changes: < 1 ชั่วโมง

---

### Level 5: Autonomous (ระดับอัตโนมัติ)

**คำอธิบาย**: ระบบสามารถ self-optimize และ self-heal ได้ AI/ML ถูกนำมาใช้ในการดำเนินงาน

**ลักษณะเฉพาะ**:
- AIOps - AI-assisted incident detection and remediation
- ML-powered capacity planning และ autoscaling
- Continuous chaos engineering
- Self-service infrastructure - developer สามารถ provision ทุกอย่างได้เอง
- Zero-trust security architecture
- Progressive delivery ที่ controlled โดย AI
- Business metrics-driven deployment decisions
- Platform engineering ที่สมบูรณ์

**ตัวชี้วัด**:
- Deployment frequency: Continuous
- MTTR: < 5 นาที (automated remediation)
- Change failure rate: < 1%
- Lead time for changes: < 15 นาที

---

## 90.2 TypeScript MaturityAssessment

การประเมิน maturity ต้องครอบคลุมหลายมิติ เรามี 40+ คำถามที่ครอบคลุมทุกด้าน:

```typescript
// maturity-assessment.ts
export type MaturityLevel = 1 | 2 | 3 | 4 | 5;

export interface AssessmentQuestion {
  id: string;
  category: AssessmentCategory;
  question: string;
  options: AssessmentOption[];
  weight: number; // 1-3, higher = more important
}

export type AssessmentCategory =
  | 'cicd'
  | 'testing'
  | 'monitoring'
  | 'security'
  | 'scaling'
  | 'team'
  | 'architecture'
  | 'operations';

export interface AssessmentOption {
  text: string;
  score: number; // 0-4 maps to levels 1-5
  level: MaturityLevel;
}

export interface AssessmentResponse {
  questionId: string;
  selectedOptionScore: number;
}

export interface AssessmentResult {
  overall: MaturityLevel;
  overallScore: number;
  categoryScores: Record<AssessmentCategory, { score: number; level: MaturityLevel }>;
  strengths: string[];
  gaps: string[];
  topRecommendations: string[];
  roadmap: RoadmapItem[];
}

export interface RoadmapItem {
  priority: 'immediate' | 'short-term' | 'medium-term' | 'long-term';
  category: AssessmentCategory;
  action: string;
  estimatedEffort: string;
  impact: 'high' | 'medium' | 'low';
}

const ASSESSMENT_QUESTIONS: AssessmentQuestion[] = [
  // CI/CD (8 questions)
  {
    id: 'cicd-01',
    category: 'cicd',
    question: 'How are deployments performed?',
    weight: 3,
    options: [
      { text: 'Manual deployments via SSH or scripts', score: 0, level: 1 },
      { text: 'Basic CI pipeline: build and test, manual deploy', score: 1, level: 2 },
      { text: 'Full CI/CD: automated build, test, and deploy to staging', score: 2, level: 3 },
      { text: 'CD to production with approval gates and automated rollback', score: 3, level: 4 },
      { text: 'Continuous deployment with progressive delivery and AI-assisted rollout', score: 4, level: 5 },
    ],
  },
  {
    id: 'cicd-02',
    category: 'cicd',
    question: 'What is your deployment frequency?',
    weight: 3,
    options: [
      { text: 'Monthly or less', score: 0, level: 1 },
      { text: 'Weekly', score: 1, level: 2 },
      { text: 'Daily', score: 2, level: 3 },
      { text: 'Multiple times per day', score: 3, level: 4 },
      { text: 'On-demand, any time', score: 4, level: 5 },
    ],
  },
  {
    id: 'cicd-03',
    category: 'cicd',
    question: 'How do you handle deployment failures?',
    weight: 2,
    options: [
      { text: 'Manual detection and rollback (often slow)', score: 0, level: 1 },
      { text: 'Basic health checks with manual rollback', score: 1, level: 2 },
      { text: 'Automated smoke tests with manual rollback trigger', score: 2, level: 3 },
      { text: 'Automated rollback based on error rate and latency thresholds', score: 3, level: 4 },
      { text: 'AI-assisted deployment with predictive rollback', score: 4, level: 5 },
    ],
  },
  {
    id: 'cicd-04',
    category: 'cicd',
    question: 'What deployment strategies do you use?',
    weight: 2,
    options: [
      { text: 'Big bang / recreate deployment', score: 0, level: 1 },
      { text: 'Rolling deployment', score: 1, level: 2 },
      { text: 'Blue/green deployment', score: 2, level: 3 },
      { text: 'Canary with automated traffic shifting', score: 3, level: 4 },
      { text: 'Progressive delivery with ML-driven traffic analysis', score: 4, level: 5 },
    ],
  },
  {
    id: 'cicd-05',
    category: 'cicd',
    question: 'How do you manage infrastructure changes?',
    weight: 2,
    options: [
      { text: 'Manual changes via console or SSH', score: 0, level: 1 },
      { text: 'Basic scripts, not version controlled', score: 1, level: 2 },
      { text: 'Infrastructure as Code (Terraform/CloudFormation)', score: 2, level: 3 },
      { text: 'GitOps with automated drift detection', score: 3, level: 4 },
      { text: 'Self-service IDP with full governance and policy enforcement', score: 4, level: 5 },
    ],
  },
  {
    id: 'cicd-06',
    category: 'cicd',
    question: 'How do you handle secrets and configuration?',
    weight: 2,
    options: [
      { text: 'Hardcoded in source code or config files', score: 0, level: 1 },
      { text: 'Environment variables, not encrypted', score: 1, level: 2 },
      { text: 'Secret management tool (Vault, AWS Secrets Manager)', score: 2, level: 3 },
      { text: 'Dynamic secrets with rotation and audit logging', score: 3, level: 4 },
      { text: 'Zero-trust secrets with automated rotation and compliance reporting', score: 4, level: 5 },
    ],
  },
  {
    id: 'cicd-07',
    category: 'cicd',
    question: 'What is your lead time from code commit to production?',
    weight: 2,
    options: [
      { text: 'More than 1 month', score: 0, level: 1 },
      { text: '1 week to 1 month', score: 1, level: 2 },
      { text: '1 day to 1 week', score: 2, level: 3 },
      { text: 'Less than 1 day', score: 3, level: 4 },
      { text: 'Less than 1 hour', score: 4, level: 5 },
    ],
  },
  {
    id: 'cicd-08',
    category: 'cicd',
    question: 'Do you use feature flags for deployments?',
    weight: 1,
    options: [
      { text: 'No feature flags', score: 0, level: 1 },
      { text: 'Manual code-based feature toggles', score: 1, level: 2 },
      { text: 'Feature flag service for some features', score: 2, level: 3 },
      { text: 'Feature flags for all new features with kill switches', score: 3, level: 4 },
      { text: 'Progressive rollout with automated A/B testing and metrics analysis', score: 4, level: 5 },
    ],
  },

  // Testing (6 questions)
  {
    id: 'testing-01',
    category: 'testing',
    question: 'What types of automated tests do you have?',
    weight: 3,
    options: [
      { text: 'No automated tests', score: 0, level: 1 },
      { text: 'Some unit tests (< 50% coverage)', score: 1, level: 2 },
      { text: 'Unit + integration tests (> 80% coverage)', score: 2, level: 3 },
      { text: 'Unit + integration + contract + e2e tests', score: 3, level: 4 },
      { text: 'Full testing pyramid + performance + chaos + mutation testing', score: 4, level: 5 },
    ],
  },
  {
    id: 'testing-02',
    category: 'testing',
    question: 'How do you test service contracts (APIs)?',
    weight: 2,
    options: [
      { text: 'No contract testing', score: 0, level: 1 },
      { text: 'Manual API testing', score: 1, level: 2 },
      { text: 'OpenAPI spec validation', score: 2, level: 3 },
      { text: 'Consumer-driven contract testing (Pact)', score: 3, level: 4 },
      { text: 'Bi-directional contract testing with schema registry', score: 4, level: 5 },
    ],
  },
  {
    id: 'testing-03',
    category: 'testing',
    question: 'Do you perform load/performance testing?',
    weight: 2,
    options: [
      { text: 'No load testing', score: 0, level: 1 },
      { text: 'Occasional manual load tests before major releases', score: 1, level: 2 },
      { text: 'Load tests in CI/CD for major changes', score: 2, level: 3 },
      { text: 'Continuous performance testing with regression alerts', score: 3, level: 4 },
      { text: 'Continuous chaos engineering + load testing in production', score: 4, level: 5 },
    ],
  },
  {
    id: 'testing-04',
    category: 'testing',
    question: 'How do you test in production?',
    weight: 1,
    options: [
      { text: 'No production testing', score: 0, level: 1 },
      { text: 'Manual smoke tests after deploy', score: 1, level: 2 },
      { text: 'Automated smoke tests after deploy', score: 2, level: 3 },
      { text: 'Canary analysis + synthetic monitoring', score: 3, level: 4 },
      { text: 'Continuous validation + chaos experiments in production', score: 4, level: 5 },
    ],
  },
  {
    id: 'testing-05',
    category: 'testing',
    question: 'How do you handle test data?',
    weight: 1,
    options: [
      { text: 'No standard approach - hardcoded test data', score: 0, level: 1 },
      { text: 'Static test data files', score: 1, level: 2 },
      { text: 'Test data factories and seeding', score: 2, level: 3 },
      { text: 'Test data service with versioned datasets', score: 3, level: 4 },
      { text: 'Automated test data generation and sanitization from production', score: 4, level: 5 },
    ],
  },
  {
    id: 'testing-06',
    category: 'testing',
    question: 'What is your change failure rate?',
    weight: 3,
    options: [
      { text: 'More than 30%', score: 0, level: 1 },
      { text: '15-30%', score: 1, level: 2 },
      { text: '5-15%', score: 2, level: 3 },
      { text: '1-5%', score: 3, level: 4 },
      { text: 'Less than 1%', score: 4, level: 5 },
    ],
  },

  // Monitoring (6 questions)
  {
    id: 'monitoring-01',
    category: 'monitoring',
    question: 'What observability pillars have you implemented?',
    weight: 3,
    options: [
      { text: 'No monitoring beyond server health', score: 0, level: 1 },
      { text: 'Basic metrics + centralized logs', score: 1, level: 2 },
      { text: 'Metrics + logs + basic distributed tracing', score: 2, level: 3 },
      { text: 'Full observability (metrics + logs + traces) with correlations', score: 3, level: 4 },
      { text: 'Full observability + eBPF + continuous profiling + AIOps', score: 4, level: 5 },
    ],
  },
  {
    id: 'monitoring-02',
    category: 'monitoring',
    question: 'How do you manage SLOs?',
    weight: 2,
    options: [
      { text: 'No SLOs defined', score: 0, level: 1 },
      { text: 'Basic uptime SLAs with static thresholds', score: 1, level: 2 },
      { text: 'SLOs defined with error budgets', score: 2, level: 3 },
      { text: 'Multi-window burn rate alerts, error budget policies', score: 3, level: 4 },
      { text: 'Business-aligned SLOs with automated budget management', score: 4, level: 5 },
    ],
  },
  {
    id: 'monitoring-03',
    category: 'monitoring',
    question: 'What is your MTTR (Mean Time to Recovery)?',
    weight: 3,
    options: [
      { text: 'More than 4 hours', score: 0, level: 1 },
      { text: '1-4 hours', score: 1, level: 2 },
      { text: '30-60 minutes', score: 2, level: 3 },
      { text: '5-30 minutes', score: 3, level: 4 },
      { text: 'Less than 5 minutes (automated remediation)', score: 4, level: 5 },
    ],
  },
  {
    id: 'monitoring-04',
    category: 'monitoring',
    question: 'How are alerts managed?',
    weight: 2,
    options: [
      { text: 'Email alerts only, often ignored', score: 0, level: 1 },
      { text: 'PagerDuty/OpsGenie with basic on-call rotation', score: 1, level: 2 },
      { text: 'Alert routing by service with runbooks', score: 2, level: 3 },
      { text: 'Alert deduplication, SLO-based alerting, on-call management', score: 3, level: 4 },
      { text: 'AIOps alert correlation + automated incident creation and remediation', score: 4, level: 5 },
    ],
  },
  {
    id: 'monitoring-05',
    category: 'monitoring',
    question: 'Can you trace a request across all services?',
    weight: 2,
    options: [
      { text: 'No distributed tracing', score: 0, level: 1 },
      { text: 'Request IDs in logs only', score: 1, level: 2 },
      { text: 'Distributed tracing for critical paths', score: 2, level: 3 },
      { text: 'Full distributed tracing for all services with sampling', score: 3, level: 4 },
      { text: 'Full tracing + baggage propagation + continuous profiling', score: 4, level: 5 },
    ],
  },
  {
    id: 'monitoring-06',
    category: 'monitoring',
    question: 'Do you have business metrics monitoring?',
    weight: 2,
    options: [
      { text: 'No business metrics', score: 0, level: 1 },
      { text: 'Manual dashboard with business KPIs', score: 1, level: 2 },
      { text: 'Automated business metrics with alerts', score: 2, level: 3 },
      { text: 'Business metrics correlated with technical metrics', score: 3, level: 4 },
      { text: 'Real-time business impact measurement driving deployment decisions', score: 4, level: 5 },
    ],
  },

  // Security (6 questions)
  {
    id: 'security-01',
    category: 'security',
    question: 'How do you handle authentication between services?',
    weight: 3,
    options: [
      { text: 'No authentication between internal services', score: 0, level: 1 },
      { text: 'Shared API keys or basic auth', score: 1, level: 2 },
      { text: 'JWT tokens or OAuth2 for service-to-service', score: 2, level: 3 },
      { text: 'mTLS with automated certificate rotation', score: 3, level: 4 },
      { text: 'Zero-trust architecture with continuous verification', score: 4, level: 5 },
    ],
  },
  {
    id: 'security-02',
    category: 'security',
    question: 'How do you scan for vulnerabilities?',
    weight: 2,
    options: [
      { text: 'No vulnerability scanning', score: 0, level: 1 },
      { text: 'Occasional manual security reviews', score: 1, level: 2 },
      { text: 'SAST + container scanning in CI', score: 2, level: 3 },
      { text: 'SAST + DAST + SCA + container scanning in CI with policy gates', score: 3, level: 4 },
      { text: 'Continuous runtime security + supply chain security (SLSA)', score: 4, level: 5 },
    ],
  },
  {
    id: 'security-03',
    category: 'security',
    question: 'How do you manage network policies?',
    weight: 2,
    options: [
      { text: 'All services can communicate with all others', score: 0, level: 1 },
      { text: 'Basic firewall rules', score: 1, level: 2 },
      { text: 'Kubernetes NetworkPolicies restricting traffic', score: 2, level: 3 },
      { text: 'Service mesh with fine-grained traffic policies', score: 3, level: 4 },
      { text: 'Zero-trust network with microsegmentation and continuous audit', score: 4, level: 5 },
    ],
  },
  {
    id: 'security-04',
    category: 'security',
    question: 'How do you handle compliance and audit?',
    weight: 2,
    options: [
      { text: 'No compliance automation', score: 0, level: 1 },
      { text: 'Manual compliance checks periodically', score: 1, level: 2 },
      { text: 'Basic audit logging in place', score: 2, level: 3 },
      { text: 'Policy as Code with automated compliance reporting', score: 3, level: 4 },
      { text: 'Continuous compliance with real-time risk scoring', score: 4, level: 5 },
    ],
  },
  {
    id: 'security-05',
    category: 'security',
    question: 'How do you manage container image security?',
    weight: 2,
    options: [
      { text: 'Using latest/unverified base images', score: 0, level: 1 },
      { text: 'Using official base images', score: 1, level: 2 },
      { text: 'Distroless/minimal images + vulnerability scanning', score: 2, level: 3 },
      { text: 'Private registry + image signing + admission control', score: 3, level: 4 },
      { text: 'SBOMs + continuous CVE monitoring + automated remediation', score: 4, level: 5 },
    ],
  },
  {
    id: 'security-06',
    category: 'security',
    question: 'Do you run penetration testing?',
    weight: 1,
    options: [
      { text: 'Never', score: 0, level: 1 },
      { text: 'Annually', score: 1, level: 2 },
      { text: 'Quarterly with third-party', score: 2, level: 3 },
      { text: 'Continuous automated pen testing + bug bounty', score: 3, level: 4 },
      { text: 'Red team exercises + automated fuzzing + bug bounty program', score: 4, level: 5 },
    ],
  },

  // Scaling (5 questions)
  {
    id: 'scaling-01',
    category: 'scaling',
    question: 'How do you handle increased load?',
    weight: 3,
    options: [
      { text: 'Manual vertical scaling (bigger server)', score: 0, level: 1 },
      { text: 'Manual horizontal scaling', score: 1, level: 2 },
      { text: 'Kubernetes HPA based on CPU/memory', score: 2, level: 3 },
      { text: 'Custom metrics scaling (KEDA) + cluster autoscaler', score: 3, level: 4 },
      { text: 'Predictive scaling with ML + global load balancing', score: 4, level: 5 },
    ],
  },
  {
    id: 'scaling-02',
    category: 'scaling',
    question: 'How do you handle service failures?',
    weight: 2,
    options: [
      { text: 'Services crash and require manual restart', score: 0, level: 1 },
      { text: 'Basic health checks with manual intervention', score: 1, level: 2 },
      { text: 'Circuit breakers + retry with exponential backoff', score: 2, level: 3 },
      { text: 'Full resilience patterns: bulkhead, timeout, fallback, circuit breaker', score: 3, level: 4 },
      { text: 'Self-healing with chaos engineering validation', score: 4, level: 5 },
    ],
  },
  {
    id: 'scaling-03',
    category: 'scaling',
    question: 'How do you manage database scaling?',
    weight: 2,
    options: [
      { text: 'Single database instance', score: 0, level: 1 },
      { text: 'Primary-replica setup', score: 1, level: 2 },
      { text: 'Read replicas + connection pooling + caching', score: 2, level: 3 },
      { text: 'Sharding + multi-region + automated failover', score: 3, level: 4 },
      { text: 'Globally distributed database with automatic sharding', score: 4, level: 5 },
    ],
  },
  {
    id: 'scaling-04',
    category: 'scaling',
    question: 'Do you practice chaos engineering?',
    weight: 1,
    options: [
      { text: 'No chaos engineering', score: 0, level: 1 },
      { text: 'Occasional manual failure injection testing', score: 1, level: 2 },
      { text: 'Chaos experiments in staging environment', score: 2, level: 3 },
      { text: 'Regular chaos experiments in production with blast radius control', score: 3, level: 4 },
      { text: 'Continuous chaos engineering as part of deployment pipeline', score: 4, level: 5 },
    ],
  },
  {
    id: 'scaling-05',
    category: 'scaling',
    question: 'How do you manage multi-region deployments?',
    weight: 2,
    options: [
      { text: 'Single region only', score: 0, level: 1 },
      { text: 'Single region with backup region (hot standby)', score: 1, level: 2 },
      { text: 'Active-passive multi-region', score: 2, level: 3 },
      { text: 'Active-active multi-region with data replication', score: 3, level: 4 },
      { text: 'Global active-active with intelligent traffic routing', score: 4, level: 5 },
    ],
  },

  // Team (5 questions)
  {
    id: 'team-01',
    category: 'team',
    question: 'How are teams organized around services?',
    weight: 3,
    options: [
      { text: 'One team owns all services', score: 0, level: 1 },
      { text: 'Functional teams (frontend, backend, ops) sharing services', score: 1, level: 2 },
      { text: 'Product teams own end-to-end vertical slices', score: 2, level: 3 },
      { text: 'Full team autonomy: you build it, you run it', score: 3, level: 4 },
      { text: 'Platform + stream-aligned + enabling teams per Team Topologies', score: 4, level: 5 },
    ],
  },
  {
    id: 'team-02',
    category: 'team',
    question: 'How do teams handle incidents?',
    weight: 2,
    options: [
      { text: 'No formal on-call rotation', score: 0, level: 1 },
      { text: 'Basic on-call rotation with manual escalation', score: 1, level: 2 },
      { text: 'On-call rotation with runbooks and escalation policy', score: 2, level: 3 },
      { text: 'Full incident management: commander, runbook automation, post-mortems', score: 3, level: 4 },
      { text: 'AI-assisted incident management with automated remediation', score: 4, level: 5 },
    ],
  },
  {
    id: 'team-03',
    category: 'team',
    question: 'How do you share knowledge across teams?',
    weight: 2,
    options: [
      { text: 'No formal knowledge sharing', score: 0, level: 1 },
      { text: 'Informal documentation and ad-hoc meetings', score: 1, level: 2 },
      { text: 'Wiki, regular tech talks, post-mortem sharing', score: 2, level: 3 },
      { text: 'Internal developer portal with service catalog', score: 3, level: 4 },
      { text: 'Self-service IDP + communities of practice + learning paths', score: 4, level: 5 },
    ],
  },
  {
    id: 'team-04',
    category: 'team',
    question: 'Can a team deploy independently without coordinating with other teams?',
    weight: 3,
    options: [
      { text: 'Coordination required for all deployments', score: 0, level: 1 },
      { text: 'Some dependencies but can usually deploy independently', score: 1, level: 2 },
      { text: 'Independent deployments for most services', score: 2, level: 3 },
      { text: 'Fully independent deployments with API versioning and contract tests', score: 3, level: 4 },
      { text: 'Complete autonomy with platform providing all required capabilities', score: 4, level: 5 },
    ],
  },
  {
    id: 'team-05',
    category: 'team',
    question: 'How do teams handle code reviews?',
    weight: 1,
    options: [
      { text: 'No code reviews', score: 0, level: 1 },
      { text: 'Informal code reviews', score: 1, level: 2 },
      { text: 'Required PRs with at least 1 reviewer', score: 2, level: 3 },
      { text: 'Required PRs with automated checks + 2 reviewers', score: 3, level: 4 },
      { text: 'Automated code analysis + architecture compliance + security review', score: 4, level: 5 },
    ],
  },

  // Architecture (5 questions)
  {
    id: 'arch-01',
    category: 'architecture',
    question: 'How well are service boundaries defined?',
    weight: 3,
    options: [
      { text: 'No clear domain boundaries - shared databases and tightly coupled', score: 0, level: 1 },
      { text: 'Some separation but shared databases still exist', score: 1, level: 2 },
      { text: 'Clear bounded contexts with separate databases', score: 2, level: 3 },
      { text: 'DDD-aligned microservices with published language and context maps', score: 3, level: 4 },
      { text: 'Cell-based architecture with complete domain autonomy', score: 4, level: 5 },
    ],
  },
  {
    id: 'arch-02',
    category: 'architecture',
    question: 'How do services communicate?',
    weight: 2,
    options: [
      { text: 'Synchronous REST only, point-to-point', score: 0, level: 1 },
      { text: 'REST with API gateway', score: 1, level: 2 },
      { text: 'REST + async messaging for some workflows', score: 2, level: 3 },
      { text: 'Event-driven architecture + CQRS for read-heavy services', score: 3, level: 4 },
      { text: 'Event sourcing + CQRS + saga patterns across all services', score: 4, level: 5 },
    ],
  },
  {
    id: 'arch-03',
    category: 'architecture',
    question: 'How do you handle data consistency across services?',
    weight: 2,
    options: [
      { text: 'Distributed transactions (2PC) or no consistency strategy', score: 0, level: 1 },
      { text: 'Eventual consistency with manual compensation', score: 1, level: 2 },
      { text: 'Outbox pattern or saga for distributed transactions', score: 2, level: 3 },
      { text: 'Event sourcing with domain events for all state changes', score: 3, level: 4 },
      { text: 'CRDT-based conflict resolution + globally consistent event log', score: 4, level: 5 },
    ],
  },
  {
    id: 'arch-04',
    category: 'architecture',
    question: 'Do you have an API versioning strategy?',
    weight: 2,
    options: [
      { text: 'No versioning - breaking changes directly', score: 0, level: 1 },
      { text: 'Version in URL (/v1/, /v2/) but no deprecation process', score: 1, level: 2 },
      { text: 'URL versioning with documented deprecation timeline', score: 2, level: 3 },
      { text: 'Semantic versioning with backward compatibility enforcement', score: 3, level: 4 },
      { text: 'Schema registry + automated compatibility checks + deprecation enforcement', score: 4, level: 5 },
    ],
  },
  {
    id: 'arch-05',
    category: 'architecture',
    question: 'How do you manage service dependencies?',
    weight: 2,
    options: [
      { text: 'No dependency tracking', score: 0, level: 1 },
      { text: 'Basic service catalog or README documentation', score: 1, level: 2 },
      { text: 'Service mesh with dependency visualization', score: 2, level: 3 },
      { text: 'Service catalog with SLO tracking and dependency health', score: 3, level: 4 },
      { text: 'Dynamic service graph with automated impact analysis', score: 4, level: 5 },
    ],
  },

  // Operations (5 questions)
  {
    id: 'ops-01',
    category: 'operations',
    question: 'How do you manage Kubernetes resources?',
    weight: 2,
    options: [
      { text: 'Not using Kubernetes', score: 0, level: 1 },
      { text: 'Kubernetes with manual resource requests set', score: 1, level: 2 },
      { text: 'Resource requests/limits + LimitRanges + ResourceQuotas', score: 2, level: 3 },
      { text: 'VPA for automatic resource tuning + capacity planning', score: 3, level: 4 },
      { text: 'ML-driven resource optimization + cost allocation per team', score: 4, level: 5 },
    ],
  },
  {
    id: 'ops-02',
    category: 'operations',
    question: 'How do you handle log management?',
    weight: 2,
    options: [
      { text: 'SSH to servers to read logs', score: 0, level: 1 },
      { text: 'Centralized log aggregation (ELK or similar)', score: 1, level: 2 },
      { text: 'Structured logging with correlation IDs and retention policies', score: 2, level: 3 },
      { text: 'Log analytics with anomaly detection and automated alerting', score: 3, level: 4 },
      { text: 'Unified observability platform with ML-based log analysis', score: 4, level: 5 },
    ],
  },
  {
    id: 'ops-03',
    category: 'operations',
    question: 'How do you perform capacity planning?',
    weight: 2,
    options: [
      { text: 'Reactive: scale when things break', score: 0, level: 1 },
      { text: 'Manual capacity reviews before known events', score: 1, level: 2 },
      { text: 'Usage trend analysis and quarterly planning', score: 2, level: 3 },
      { text: 'Automated capacity alerts with monthly planning process', score: 3, level: 4 },
      { text: 'ML-driven predictive scaling and cost optimization', score: 4, level: 5 },
    ],
  },
  {
    id: 'ops-04',
    category: 'operations',
    question: 'How do you handle configuration drift?',
    weight: 2,
    options: [
      { text: 'No drift detection', score: 0, level: 1 },
      { text: 'Manual audits occasionally', score: 1, level: 2 },
      { text: 'Configuration management with periodic drift checks', score: 2, level: 3 },
      { text: 'Continuous drift detection with automated remediation', score: 3, level: 4 },
      { text: 'Immutable infrastructure - drift is impossible by design', score: 4, level: 5 },
    ],
  },
  {
    id: 'ops-05',
    category: 'operations',
    question: 'Do you have a disaster recovery plan?',
    weight: 3,
    options: [
      { text: 'No DR plan', score: 0, level: 1 },
      { text: 'Basic backup and manual recovery documented', score: 1, level: 2 },
      { text: 'DR plan with defined RPO/RTO and tested annually', score: 2, level: 3 },
      { text: 'Automated DR with quarterly failover drills and SLA guarantees', score: 3, level: 4 },
      { text: 'Continuous DR validation + multi-region active-active with RPO < 1 min', score: 4, level: 5 },
    ],
  },
];

export class MaturityAssessment {
  private questions: AssessmentQuestion[];

  constructor() {
    this.questions = ASSESSMENT_QUESTIONS;
  }

  getQuestions(category?: AssessmentCategory): AssessmentQuestion[] {
    if (category) {
      return this.questions.filter((q) => q.category === category);
    }
    return this.questions;
  }

  calculateResult(responses: AssessmentResponse[]): AssessmentResult {
    const responseMap = new Map(
      responses.map((r) => [r.questionId, r.selectedOptionScore])
    );

    // Calculate category scores
    const categories: AssessmentCategory[] = [
      'cicd', 'testing', 'monitoring', 'security', 'scaling', 'team', 'architecture', 'operations',
    ];

    const categoryScores: Record<AssessmentCategory, { score: number; level: MaturityLevel }> =
      {} as Record<AssessmentCategory, { score: number; level: MaturityLevel }>;

    let totalWeightedScore = 0;
    let totalWeight = 0;

    for (const category of categories) {
      const categoryQuestions = this.questions.filter((q) => q.category === category);
      let categoryWeightedScore = 0;
      let categoryWeight = 0;

      for (const question of categoryQuestions) {
        const response = responseMap.get(question.id);
        if (response !== undefined) {
          categoryWeightedScore += response * question.weight;
          categoryWeight += question.weight * 4; // max score is 4
        }
      }

      const normalizedScore = categoryWeight > 0
        ? (categoryWeightedScore / categoryWeight) * 100
        : 0;

      const level = this.scoreToLevel(normalizedScore);
      categoryScores[category] = { score: Math.round(normalizedScore), level };

      totalWeightedScore += categoryWeightedScore;
      totalWeight += categoryWeight;
    }

    const overallScore = totalWeight > 0
      ? (totalWeightedScore / totalWeight) * 100
      : 0;

    const overall = this.scoreToLevel(overallScore);

    // Identify strengths and gaps
    const sortedCategories = Object.entries(categoryScores).sort(
      ([, a], [, b]) => b.score - a.score
    );

    const strengths = sortedCategories
      .slice(0, 3)
      .map(([cat, data]) => `${cat}: ${data.score}% (Level ${data.level})`);

    const gaps = sortedCategories
      .slice(-3)
      .reverse()
      .map(([cat, data]) => `${cat}: ${data.score}% (Level ${data.level})`);

    const recommendations = this.generateRecommendations(categoryScores, overall);
    const roadmap = this.generateRoadmap(categoryScores);

    return {
      overall,
      overallScore: Math.round(overallScore),
      categoryScores,
      strengths,
      gaps,
      topRecommendations: recommendations,
      roadmap,
    };
  }

  private scoreToLevel(score: number): MaturityLevel {
    if (score < 20) return 1;
    if (score < 40) return 2;
    if (score < 60) return 3;
    if (score < 80) return 4;
    return 5;
  }

  private generateRecommendations(
    categoryScores: Record<AssessmentCategory, { score: number; level: MaturityLevel }>,
    overallLevel: MaturityLevel
  ): string[] {
    const recs: string[] = [];

    const lowCategories = Object.entries(categoryScores)
      .filter(([, data]) => data.level <= overallLevel - 1)
      .sort(([, a], [, b]) => a.score - b.score);

    const recommendations: Record<string, string[]> = {
      cicd: [
        'Implement automated CI/CD pipeline with staged deployments',
        'Add automated rollback based on SLO burn rate',
        'Implement GitOps for infrastructure management',
      ],
      testing: [
        'Establish testing pyramid with unit, integration, and contract tests',
        'Add consumer-driven contract testing with Pact',
        'Implement continuous performance testing in CI',
      ],
      monitoring: [
        'Define SLOs for all critical services',
        'Implement distributed tracing across all services',
        'Set up multi-window burn rate alerting',
      ],
      security: [
        'Implement mTLS for service-to-service communication',
        'Add SAST and container scanning to CI pipeline',
        'Establish secrets management with rotation',
      ],
      scaling: [
        'Implement circuit breakers for all service dependencies',
        'Set up Kubernetes HPA and cluster autoscaler',
        'Conduct regular chaos engineering experiments',
      ],
      team: [
        'Restructure teams around business domains (Team Topologies)',
        'Implement formal on-call rotation with runbooks',
        'Create internal developer portal with service catalog',
      ],
      architecture: [
        'Define clear bounded contexts and domain ownership',
        'Implement event-driven communication for async workflows',
        'Establish API versioning strategy with deprecation policy',
      ],
      operations: [
        'Implement comprehensive disaster recovery with regular drills',
        'Set up continuous configuration drift detection',
        'Establish capacity planning process with predictive scaling',
      ],
    };

    lowCategories.slice(0, 3).forEach(([category]) => {
      const catRecs = recommendations[category] ?? [];
      recs.push(...catRecs.slice(0, 2));
    });

    return recs.slice(0, 6);
  }

  private generateRoadmap(
    categoryScores: Record<AssessmentCategory, { score: number; level: MaturityLevel }>
  ): RoadmapItem[] {
    const roadmap: RoadmapItem[] = [];

    const items: RoadmapItem[] = [
      {
        priority: 'immediate',
        category: 'monitoring',
        action: 'Define and instrument SLOs for top 3 critical services',
        estimatedEffort: '2 weeks',
        impact: 'high',
      },
      {
        priority: 'immediate',
        category: 'cicd',
        action: 'Implement automated rollback on deployment failure',
        estimatedEffort: '1 week',
        impact: 'high',
      },
      {
        priority: 'short-term',
        category: 'testing',
        action: 'Add contract testing with Pact for all service-to-service APIs',
        estimatedEffort: '4 weeks',
        impact: 'high',
      },
      {
        priority: 'short-term',
        category: 'security',
        action: 'Implement secrets rotation and dynamic secrets with Vault',
        estimatedEffort: '3 weeks',
        impact: 'high',
      },
      {
        priority: 'medium-term',
        category: 'team',
        action: 'Reorganize teams into stream-aligned teams per Team Topologies',
        estimatedEffort: '3 months',
        impact: 'high',
      },
      {
        priority: 'medium-term',
        category: 'scaling',
        action: 'Implement chaos engineering program',
        estimatedEffort: '6 weeks',
        impact: 'medium',
      },
      {
        priority: 'long-term',
        category: 'architecture',
        action: 'Migrate to event-driven architecture for async workflows',
        estimatedEffort: '6 months',
        impact: 'high',
      },
      {
        priority: 'long-term',
        category: 'operations',
        action: 'Build Internal Developer Platform for self-service provisioning',
        estimatedEffort: '12 months',
        impact: 'high',
      },
    ];

    // Filter based on which categories are weakest
    return items.filter((item) => {
      const catScore = categoryScores[item.category];
      return catScore && catScore.level <= 3;
    });
  }
}
```

---

## 90.3 Strangler Fig Pattern TypeScript Proxy

Strangler Fig เป็น pattern ในการ migrate จาก legacy system ไปยัง microservices ทีละส่วน:

```typescript
// strangler-fig-proxy.ts
import express, { Request, Response, NextFunction } from 'express';
import httpProxy from 'http-proxy';
import crypto from 'crypto';

export interface RouteConfig {
  path: string;
  method: string;
  legacyUrl: string;
  newUrl: string;
  trafficPercentage: number; // 0-100, percentage routed to new service
  enabled: boolean;
  shadowMode?: boolean; // send to both but return legacy response
  headers?: Record<string, string>;
}

export interface ProxyMetrics {
  totalRequests: number;
  newServiceRequests: number;
  legacyServiceRequests: number;
  shadowRequests: number;
  newServiceErrors: number;
  legacyServiceErrors: number;
}

export class StranglerFigProxy {
  private app: express.Application;
  private proxy: httpProxy;
  private routes: RouteConfig[];
  private metrics: ProxyMetrics;
  private metricsStore: Map<string, ProxyMetrics>;

  constructor() {
    this.app = express();
    this.proxy = httpProxy.createProxyServer({ changeOrigin: true });
    this.routes = [];
    this.metrics = {
      totalRequests: 0,
      newServiceRequests: 0,
      legacyServiceRequests: 0,
      shadowRequests: 0,
      newServiceErrors: 0,
      legacyServiceErrors: 0,
    };
    this.metricsStore = new Map();

    this.setupErrorHandling();
    this.setupMiddleware();
  }

  addRoute(config: RouteConfig): void {
    const existing = this.routes.findIndex(
      (r) => r.path === config.path && r.method === config.method
    );

    if (existing >= 0) {
      this.routes[existing] = config;
    } else {
      this.routes.push(config);
    }

    this.metricsStore.set(`${config.method}:${config.path}`, {
      totalRequests: 0,
      newServiceRequests: 0,
      legacyServiceRequests: 0,
      shadowRequests: 0,
      newServiceErrors: 0,
      legacyServiceErrors: 0,
    });
  }

  updateTrafficSplit(path: string, method: string, newPercentage: number): boolean {
    const route = this.routes.find(
      (r) => r.path === path && r.method === method
    );

    if (!route) return false;
    if (newPercentage < 0 || newPercentage > 100) return false;

    route.trafficPercentage = newPercentage;
    return true;
  }

  private setupMiddleware(): void {
    this.app.use(express.json());
    this.app.use(this.proxyMiddleware.bind(this));
  }

  private proxyMiddleware(req: Request, res: Response, next: NextFunction): void {
    const route = this.findRoute(req.path, req.method);

    if (!route || !route.enabled) {
      next();
      return;
    }

    const routeMetrics = this.metricsStore.get(`${route.method}:${route.path}`);
    if (routeMetrics) routeMetrics.totalRequests++;
    this.metrics.totalRequests++;

    const requestId = crypto.randomUUID();
    req.headers['x-request-id'] = requestId;

    if (route.shadowMode) {
      this.handleShadowMode(req, res, route, routeMetrics);
      return;
    }

    // Determine routing based on percentage
    const routeToNew = this.shouldRouteToNew(route, req);

    if (routeToNew) {
      this.proxyToNew(req, res, route, routeMetrics);
    } else {
      this.proxyToLegacy(req, res, route, routeMetrics);
    }
  }

  private shouldRouteToNew(route: RouteConfig, req: Request): boolean {
    // Use sticky routing based on user ID if available
    const userId = req.headers['x-user-id'] as string;
    if (userId) {
      const hash = parseInt(
        crypto.createHash('md5').update(userId + route.path).digest('hex').slice(0, 8),
        16
      );
      return (hash % 100) < route.trafficPercentage;
    }

    // Otherwise random
    return Math.random() * 100 < route.trafficPercentage;
  }

  private proxyToNew(
    req: Request,
    res: Response,
    route: RouteConfig,
    metrics?: ProxyMetrics
  ): void {
    if (metrics) metrics.newServiceRequests++;
    this.metrics.newServiceRequests++;

    req.headers['x-routed-to'] = 'new';
    if (route.headers) {
      Object.assign(req.headers, route.headers);
    }

    this.proxy.web(req, res, {
      target: route.newUrl,
      selfHandleResponse: false,
    });
  }

  private proxyToLegacy(
    req: Request,
    res: Response,
    route: RouteConfig,
    metrics?: ProxyMetrics
  ): void {
    if (metrics) metrics.legacyServiceRequests++;
    this.metrics.legacyServiceRequests++;

    req.headers['x-routed-to'] = 'legacy';

    this.proxy.web(req, res, {
      target: route.legacyUrl,
      selfHandleResponse: false,
    });
  }

  private handleShadowMode(
    req: Request,
    res: Response,
    route: RouteConfig,
    metrics?: ProxyMetrics
  ): void {
    if (metrics) {
      metrics.legacyServiceRequests++;
      metrics.shadowRequests++;
    }
    this.metrics.legacyServiceRequests++;
    this.metrics.shadowRequests++;

    // Send to legacy and return its response
    this.proxy.web(req, res, { target: route.legacyUrl });

    // Also shadow to new service (fire and forget)
    const shadowReq = { ...req, headers: { ...req.headers, 'x-shadow': 'true' } };
    setTimeout(() => {
      const http = require('http');
      const newUrl = new URL(route.newUrl);
      const options = {
        hostname: newUrl.hostname,
        port: newUrl.port || 80,
        path: req.path,
        method: req.method,
        headers: shadowReq.headers,
      };
      const shadowRequest = http.request(options, () => {});
      shadowRequest.on('error', () => {
        if (metrics) metrics.newServiceErrors++;
      });
      shadowRequest.end();
    }, 0);
  }

  private findRoute(path: string, method: string): RouteConfig | undefined {
    return this.routes.find((r) => {
      const methodMatch = r.method === method || r.method === '*';
      const pathMatch = this.matchPath(path, r.path);
      return methodMatch && pathMatch;
    });
  }

  private matchPath(requestPath: string, routePath: string): boolean {
    const routeParts = routePath.split('/');
    const requestParts = requestPath.split('/');

    if (routeParts.length !== requestParts.length) return false;

    return routeParts.every((part, i) => {
      return part.startsWith(':') || part === requestParts[i];
    });
  }

  private setupErrorHandling(): void {
    this.proxy.on('error', (err, req, res) => {
      console.error('Proxy error:', err);
      if (!res.headersSent) {
        (res as Response).status(502).json({
          error: 'Bad Gateway',
          message: err.message,
        });
      }
    });
  }

  getMetrics(): { global: ProxyMetrics; perRoute: Record<string, ProxyMetrics> } {
    const perRoute: Record<string, ProxyMetrics> = {};
    this.metricsStore.forEach((metrics, key) => {
      perRoute[key] = { ...metrics };
    });

    return {
      global: { ...this.metrics },
      perRoute,
    };
  }

  getApp(): express.Application {
    return this.app;
  }
}

// Usage example
const proxy = new StranglerFigProxy();

proxy.addRoute({
  path: '/api/products/:id',
  method: 'GET',
  legacyUrl: 'http://legacy-monolith:3000',
  newUrl: 'http://product-service:8080',
  trafficPercentage: 10, // Start with 10% to new service
  enabled: true,
  shadowMode: false,
});

proxy.addRoute({
  path: '/api/users/:id',
  method: 'GET',
  legacyUrl: 'http://legacy-monolith:3000',
  newUrl: 'http://user-service:8081',
  trafficPercentage: 0, // Shadow mode first
  enabled: true,
  shadowMode: true,
});

const server = proxy.getApp().listen(3001, () => {
  console.log('Strangler Fig proxy running on port 3001');
});

// Gradually increase traffic to new service
const migration = [
  { percentage: 10, delay: 0 },
  { percentage: 25, delay: 24 * 60 * 60 * 1000 }, // 1 day
  { percentage: 50, delay: 48 * 60 * 60 * 1000 }, // 2 days
  { percentage: 75, delay: 72 * 60 * 60 * 1000 }, // 3 days
  { percentage: 100, delay: 96 * 60 * 60 * 1000 }, // 4 days
];
```

---

## 90.4 TypeScript TeamAutonomyScorer

```typescript
// team-autonomy-scorer.ts
export interface TeamInfo {
  name: string;
  services: string[];
  hasOwnCICD: boolean;
  hasOwnMonitoring: boolean;
  hasOwnDatabase: boolean;
  canDeployIndependently: boolean;
  deploymentCoordination: 'none' | 'light' | 'heavy';
  sharedComponents: number;
  teamSize: number;
  onCallRotation: boolean;
  ownsDomain: boolean;
  hasServiceOwnership: boolean;
  crossTeamDependencies: number;
  decisionMakingSpeed: 'fast' | 'medium' | 'slow';
}

export interface TeamAutonomyScore {
  team: string;
  totalScore: number;
  maxScore: number;
  percentage: number;
  level: 'low' | 'medium' | 'high' | 'fully-autonomous';
  dimensions: Record<string, { score: number; maxScore: number; notes: string }>;
  improvements: string[];
}

export class TeamAutonomyScorer {
  scoreTeam(team: TeamInfo): TeamAutonomyScore {
    const dimensions: Record<string, { score: number; maxScore: number; notes: string }> = {};

    // Deployment independence (20 points)
    const deployScore = this.scoreDeploymentIndependence(team);
    dimensions['deployment_independence'] = deployScore;

    // Infrastructure ownership (15 points)
    const infraScore = this.scoreInfrastructureOwnership(team);
    dimensions['infrastructure_ownership'] = infraScore;

    // Domain ownership (20 points)
    const domainScore = this.scoreDomainOwnership(team);
    dimensions['domain_ownership'] = domainScore;

    // Decision making (15 points)
    const decisionScore = this.scoreDecisionMaking(team);
    dimensions['decision_making'] = decisionScore;

    // Operational capability (15 points)
    const opsScore = this.scoreOperationalCapability(team);
    dimensions['operational_capability'] = opsScore;

    // Team structure (15 points)
    const structureScore = this.scoreTeamStructure(team);
    dimensions['team_structure'] = structureScore;

    const totalScore = Object.values(dimensions).reduce((sum, d) => sum + d.score, 0);
    const maxScore = Object.values(dimensions).reduce((sum, d) => sum + d.maxScore, 0);
    const percentage = Math.round((totalScore / maxScore) * 100);

    const level = this.scoreToLevel(percentage);
    const improvements = this.generateImprovements(dimensions, team);

    return {
      team: team.name,
      totalScore,
      maxScore,
      percentage,
      level,
      dimensions,
      improvements,
    };
  }

  private scoreDeploymentIndependence(team: TeamInfo) {
    let score = 0;

    if (team.hasOwnCICD) score += 10;
    if (team.canDeployIndependently) score += 5;
    if (team.deploymentCoordination === 'none') score += 5;
    else if (team.deploymentCoordination === 'light') score += 2;

    return {
      score,
      maxScore: 20,
      notes: `CI/CD: ${team.hasOwnCICD ? '✓' : '✗'}, Independent: ${team.canDeployIndependently ? '✓' : '✗'}, Coordination: ${team.deploymentCoordination}`,
    };
  }

  private scoreInfrastructureOwnership(team: TeamInfo) {
    let score = 0;

    if (team.hasOwnMonitoring) score += 7;
    if (team.hasOwnDatabase) score += 8;
    score -= Math.min(team.sharedComponents * 1, 5);

    return {
      score: Math.max(score, 0),
      maxScore: 15,
      notes: `Monitoring: ${team.hasOwnMonitoring ? '✓' : '✗'}, Own DB: ${team.hasOwnDatabase ? '✓' : '✗'}, Shared components: ${team.sharedComponents}`,
    };
  }

  private scoreDomainOwnership(team: TeamInfo) {
    let score = 0;

    if (team.ownsDomain) score += 10;
    if (team.hasServiceOwnership) score += 5;
    score -= Math.min(team.crossTeamDependencies * 2, 10);

    return {
      score: Math.max(score, 0),
      maxScore: 20,
      notes: `Domain owner: ${team.ownsDomain ? '✓' : '✗'}, Service ownership: ${team.hasServiceOwnership ? '✓' : '✗'}, Cross-team deps: ${team.crossTeamDependencies}`,
    };
  }

  private scoreDecisionMaking(team: TeamInfo) {
    const speedScore = team.decisionMakingSpeed === 'fast' ? 15
      : team.decisionMakingSpeed === 'medium' ? 8
      : 3;

    return {
      score: speedScore,
      maxScore: 15,
      notes: `Decision speed: ${team.decisionMakingSpeed}`,
    };
  }

  private scoreOperationalCapability(team: TeamInfo) {
    let score = 0;

    if (team.onCallRotation) score += 15;
    else score += 5;

    return {
      score,
      maxScore: 15,
      notes: `On-call rotation: ${team.onCallRotation ? '✓' : '✗'}`,
    };
  }

  private scoreTeamStructure(team: TeamInfo) {
    let score = 0;

    const idealSize = team.teamSize >= 5 && team.teamSize <= 9;
    if (idealSize) score += 15;
    else if (team.teamSize >= 3 && team.teamSize <= 12) score += 8;
    else score += 3;

    return {
      score,
      maxScore: 15,
      notes: `Team size: ${team.teamSize} (ideal: 5-9)`,
    };
  }

  private scoreToLevel(percentage: number): TeamAutonomyScore['level'] {
    if (percentage >= 85) return 'fully-autonomous';
    if (percentage >= 65) return 'high';
    if (percentage >= 40) return 'medium';
    return 'low';
  }

  private generateImprovements(
    dimensions: Record<string, { score: number; maxScore: number }>,
    team: TeamInfo
  ): string[] {
    const improvements: string[] = [];

    Object.entries(dimensions).forEach(([key, value]) => {
      const percentage = (value.score / value.maxScore) * 100;
      if (percentage < 50) {
        switch (key) {
          case 'deployment_independence':
            improvements.push('Create team-owned CI/CD pipeline to eliminate deployment dependencies');
            break;
          case 'infrastructure_ownership':
            improvements.push('Move to team-owned databases and eliminate shared infrastructure');
            break;
          case 'domain_ownership':
            improvements.push('Clarify domain boundaries and reduce cross-team dependencies');
            break;
          case 'decision_making':
            improvements.push('Empower team to make technical decisions without approval chains');
            break;
          case 'operational_capability':
            improvements.push('Establish team on-call rotation with runbooks');
            break;
          case 'team_structure':
            improvements.push(`Adjust team size towards 5-9 people (current: ${team.teamSize})`);
            break;
        }
      }
    });

    return improvements;
  }

  compareTeams(teams: TeamInfo[]): Array<TeamAutonomyScore & { rank: number }> {
    const scores = teams.map((t) => this.scoreTeam(t));
    return scores
      .sort((a, b) => b.percentage - a.percentage)
      .map((score, index) => ({ ...score, rank: index + 1 }));
  }
}
```

---

## 90.5 TypeScript OpsExcellenceChecker

```typescript
// ops-excellence-checker.ts
import axios from 'axios';

export interface DORAMetrics {
  deploymentFrequency: number; // deployments per day
  leadTimeForChanges: number; // hours from commit to production
  changeFailureRate: number; // percentage 0-100
  mttr: number; // minutes
}

export interface OpsExcellenceResult {
  overallScore: number; // 0-100
  level: 'low' | 'medium' | 'high' | 'elite';
  doraMetrics: DORAMetrics;
  doraLevel: 'low' | 'medium' | 'high' | 'elite';
  details: {
    deploymentFrequency: { value: number; score: number; label: string };
    leadTime: { value: number; score: number; label: string };
    changeFailureRate: { value: number; score: number; label: string };
    mttr: { value: number; score: number; label: string };
  };
  recommendations: string[];
}

export class OpsExcellenceChecker {
  private prometheusUrl: string;

  constructor(prometheusUrl: string) {
    this.prometheusUrl = prometheusUrl;
  }

  async checkExcellence(
    serviceName: string,
    timeRangeDays = 30
  ): Promise<OpsExcellenceResult> {
    const metrics = await this.fetchDORAMetrics(serviceName, timeRangeDays);
    return this.calculateExcellenceScore(metrics);
  }

  private async fetchDORAMetrics(
    serviceName: string,
    days: number
  ): Promise<DORAMetrics> {
    const endTime = Math.floor(Date.now() / 1000);
    const startTime = endTime - days * 24 * 3600;

    // Fetch deployment frequency
    const deploymentQuery = `sum(increase(deployments_total{service="${serviceName}"}[${days}d])) / ${days}`;
    const deployFreq = await this.queryPrometheus(deploymentQuery);

    // Fetch lead time (assuming we track this as a gauge)
    const leadTimeQuery = `avg_over_time(deployment_lead_time_hours{service="${serviceName}"}[${days}d])`;
    const leadTime = await this.queryPrometheus(leadTimeQuery);

    // Fetch change failure rate
    const failedDeployQuery = `sum(increase(deployments_failed_total{service="${serviceName}"}[${days}d]))`;
    const totalDeployQuery = `sum(increase(deployments_total{service="${serviceName}"}[${days}d]))`;

    const [failedDeploys, totalDeploys] = await Promise.all([
      this.queryPrometheus(failedDeployQuery),
      this.queryPrometheus(totalDeployQuery),
    ]);

    const changeFailureRate = totalDeploys > 0
      ? (failedDeploys / totalDeploys) * 100
      : 0;

    // Fetch MTTR
    const mttrQuery = `avg_over_time(incident_recovery_time_minutes{service="${serviceName}"}[${days}d])`;
    const mttr = await this.queryPrometheus(mttrQuery);

    return {
      deploymentFrequency: deployFreq || 0.1, // fallback
      leadTimeForChanges: leadTime || 48, // fallback: 2 days
      changeFailureRate: changeFailureRate || 10,
      mttr: mttr || 240, // fallback: 4 hours
    };
  }

  private async queryPrometheus(query: string): Promise<number> {
    try {
      const response = await axios.get(`${this.prometheusUrl}/api/v1/query`, {
        params: { query },
        timeout: 5000,
      });

      const result = response.data?.data?.result?.[0]?.value?.[1];
      return result ? parseFloat(result) : 0;
    } catch {
      return 0;
    }
  }

  calculateExcellenceScore(metrics: DORAMetrics): OpsExcellenceResult {
    // DORA Research benchmarks:
    // Elite: deploys on demand, < 1 hour lead time, < 5% CFR, < 1 hour MTTR
    // High: 1/week to 1/day, 1 day to 1 week, 5-10% CFR, < 1 day MTTR
    // Medium: 1/week to 1/month, 1 week to 1 month, 10-15% CFR, < 1 week MTTR
    // Low: < 1/month, > 1 month, > 15% CFR, > 1 week MTTR

    const dfScore = this.scoreDeploymentFrequency(metrics.deploymentFrequency);
    const ltScore = this.scoreLeadTime(metrics.leadTimeForChanges);
    const cfrScore = this.scoreChangeFailureRate(metrics.changeFailureRate);
    const mttrScore = this.scoreMTTR(metrics.mttr);

    const details = {
      deploymentFrequency: {
        value: metrics.deploymentFrequency,
        score: dfScore.score,
        label: dfScore.label,
      },
      leadTime: {
        value: metrics.leadTimeForChanges,
        score: ltScore.score,
        label: ltScore.label,
      },
      changeFailureRate: {
        value: metrics.changeFailureRate,
        score: cfrScore.score,
        label: cfrScore.label,
      },
      mttr: {
        value: metrics.mttr,
        score: mttrScore.score,
        label: mttrScore.label,
      },
    };

    const avgScore = (dfScore.score + ltScore.score + cfrScore.score + mttrScore.score) / 4;
    const level = this.levelFromScore(avgScore);

    // DORA level is the minimum across all dimensions
    const minLevel = Math.min(dfScore.level, ltScore.level, cfrScore.level, mttrScore.level);
    const doraLevel = this.doraLevelName(minLevel);

    return {
      overallScore: Math.round(avgScore),
      level: level as OpsExcellenceResult['level'],
      doraMetrics: metrics,
      doraLevel,
      details,
      recommendations: this.generateRecommendations(details),
    };
  }

  private scoreDeploymentFrequency(deploysPerDay: number): { score: number; label: string; level: number } {
    if (deploysPerDay >= 1) return { score: 100, label: 'Elite: Daily+', level: 4 };
    if (deploysPerDay >= 1 / 7) return { score: 75, label: 'High: Weekly', level: 3 };
    if (deploysPerDay >= 1 / 30) return { score: 50, label: 'Medium: Monthly', level: 2 };
    return { score: 25, label: 'Low: Less than monthly', level: 1 };
  }

  private scoreLeadTime(hours: number): { score: number; label: string; level: number } {
    if (hours < 1) return { score: 100, label: 'Elite: < 1 hour', level: 4 };
    if (hours < 24) return { score: 75, label: 'High: < 1 day', level: 3 };
    if (hours < 168) return { score: 50, label: 'Medium: < 1 week', level: 2 };
    return { score: 25, label: 'Low: > 1 week', level: 1 };
  }

  private scoreChangeFailureRate(percentage: number): { score: number; label: string; level: number } {
    if (percentage < 5) return { score: 100, label: 'Elite: < 5%', level: 4 };
    if (percentage < 10) return { score: 75, label: 'High: 5-10%', level: 3 };
    if (percentage < 15) return { score: 50, label: 'Medium: 10-15%', level: 2 };
    return { score: 25, label: 'Low: > 15%', level: 1 };
  }

  private scoreMTTR(minutes: number): { score: number; label: string; level: number } {
    if (minutes < 60) return { score: 100, label: 'Elite: < 1 hour', level: 4 };
    if (minutes < 24 * 60) return { score: 75, label: 'High: < 1 day', level: 3 };
    if (minutes < 7 * 24 * 60) return { score: 50, label: 'Medium: < 1 week', level: 2 };
    return { score: 25, label: 'Low: > 1 week', level: 1 };
  }

  private levelFromScore(score: number): string {
    if (score >= 87.5) return 'elite';
    if (score >= 62.5) return 'high';
    if (score >= 37.5) return 'medium';
    return 'low';
  }

  private doraLevelName(level: number): OpsExcellenceResult['doraLevel'] {
    if (level >= 4) return 'elite';
    if (level >= 3) return 'high';
    if (level >= 2) return 'medium';
    return 'low';
  }

  private generateRecommendations(details: OpsExcellenceResult['details']): string[] {
    const recs: string[] = [];

    if (details.deploymentFrequency.score < 75) {
      recs.push('Invest in CI/CD automation to increase deployment frequency');
    }
    if (details.leadTime.score < 75) {
      recs.push('Reduce pipeline duration through parallel testing and faster builds');
    }
    if (details.changeFailureRate.score < 75) {
      recs.push('Improve testing coverage and add canary deployments to reduce failures');
    }
    if (details.mttr.score < 75) {
      recs.push('Create runbooks and implement automated incident response to reduce MTTR');
    }

    return recs;
  }
}
```

---

## 90.6 Migration Roadmap Generator TypeScript

```typescript
// migration-roadmap-generator.ts
export interface MigrationPhase {
  phase: number;
  name: string;
  duration: string;
  goals: string[];
  deliverables: string[];
  successCriteria: string[];
  risks: string[];
  dependencies: string[];
}

export function generateMigrationRoadmap(
  currentLevel: MaturityLevel,
  targetLevel: MaturityLevel,
  teamSize: number,
  timelineMonths: number
): MigrationPhase[] {
  const phases: MigrationPhase[] = [];

  if (currentLevel === 1 && targetLevel >= 2) {
    phases.push({
      phase: 1,
      name: 'Foundation Building',
      duration: '1-2 months',
      goals: [
        'Establish containerization with Docker',
        'Set up basic CI/CD pipeline',
        'Implement centralized logging',
        'Define team ownership boundaries',
      ],
      deliverables: [
        'All services containerized',
        'Jenkins/GitHub Actions pipeline per service',
        'ELK stack or Loki for log aggregation',
        'Service ownership RACI matrix',
        'Basic API documentation (OpenAPI)',
      ],
      successCriteria: [
        'All services deploy via pipeline (no manual deploys)',
        'Can query logs from central UI',
        'Each service has a designated team',
      ],
      risks: [
        'Existing services may have complex startup dependencies',
        'Team resistance to new tools',
      ],
      dependencies: [],
    });
  }

  if (currentLevel <= 2 && targetLevel >= 3) {
    phases.push({
      phase: 2,
      name: 'Kubernetes Adoption',
      duration: '2-3 months',
      goals: [
        'Migrate to Kubernetes',
        'Implement service discovery',
        'Set up distributed tracing',
        'Define SLOs for critical services',
      ],
      deliverables: [
        'All services running on Kubernetes',
        'Helm charts for all services',
        'Jaeger or Zipkin distributed tracing',
        'Prometheus + Grafana monitoring',
        'First set of SLO definitions',
        'Circuit breakers implemented',
      ],
      successCriteria: [
        'Zero downtime deployments (rolling)',
        'P99 latency tracked per service',
        'Error budget dashboards visible to all teams',
      ],
      risks: [
        'Stateful services complex to migrate',
        'Learning curve for Kubernetes',
        'Potential performance changes',
      ],
      dependencies: ['Phase 1 complete'],
    });
  }

  if (currentLevel <= 3 && targetLevel >= 4) {
    phases.push({
      phase: 3,
      name: 'Advanced Operations',
      duration: '3-4 months',
      goals: [
        'Implement GitOps',
        'Full testing pyramid',
        'Service mesh deployment',
        'Security hardening',
        'Team autonomy enablement',
      ],
      deliverables: [
        'ArgoCD for GitOps deployments',
        'Contract testing with Pact',
        'Istio or Linkerd service mesh',
        'mTLS between all services',
        'Feature flag service',
        'Internal developer platform (basic)',
        'Canary deployment capability',
      ],
      successCriteria: [
        'Teams can deploy independently without coordination',
        'mTLS enforced for all service-to-service communication',
        'Error budget policy in place',
        'Canary deployments for all new features',
      ],
      risks: [
        'Service mesh complexity',
        'mTLS certificate management',
        'Team topology changes require organizational buy-in',
      ],
      dependencies: ['Phase 2 complete'],
    });
  }

  if (currentLevel <= 4 && targetLevel >= 5) {
    phases.push({
      phase: 4,
      name: 'Autonomous Operations',
      duration: '6-12 months',
      goals: [
        'AIOps implementation',
        'Complete self-service IDP',
        'Chaos engineering program',
        'ML-driven scaling',
        'Business metrics integration',
      ],
      deliverables: [
        'AI-powered incident detection and auto-remediation',
        'Full self-service developer platform',
        'Continuous chaos engineering in production',
        'ML-based predictive autoscaling',
        'Business metrics-driven deployment decisions',
        'SLSA Level 3 supply chain security',
      ],
      successCriteria: [
        'MTTR < 5 minutes for common incidents',
        'Developers self-service all infrastructure',
        'Chaos experiments run weekly in production',
        'Deployment decisions driven by business impact metrics',
      ],
      risks: [
        'Cultural resistance to autonomous systems',
        'AI/ML reliability requirements',
        'Significant investment required',
      ],
      dependencies: ['Phase 3 complete', 'Executive sponsorship'],
    });
  }

  return phases;
}
```

---

## 90.7 Maturity Radar Chart (Mermaid)

Mermaid diagram สามารถสร้าง radar chart เพื่อแสดง maturity แต่ละมิติได้:

```mermaid
%%{init: {'theme': 'default'}}%%
radar
  title Microservices Maturity Assessment
  "CI/CD" : [20, 60, 85, 90]
  "Testing" : [15, 45, 75, 88]
  "Monitoring" : [30, 70, 80, 92]
  "Security" : [10, 40, 65, 85]
  "Scaling" : [25, 55, 78, 90]
  "Team Autonomy" : [20, 50, 70, 86]
  "Architecture" : [15, 45, 72, 88]
  "Operations" : [25, 60, 82, 91]
```

```typescript
// maturity-report-generator.ts
export function generateMaturityReport(result: AssessmentResult): string {
  const bars: Record<string, string> = {};

  Object.entries(result.categoryScores).forEach(([category, data]) => {
    const filled = Math.round(data.score / 5);
    const empty = 20 - filled;
    bars[category] = `[${'█'.repeat(filled)}${'░'.repeat(empty)}] ${data.score}% (Level ${data.level})`;
  });

  return `
# Microservices Maturity Assessment Report

## Overall Result
**Overall Level**: ${result.overall} / 5
**Overall Score**: ${result.overallScore}%

## Category Scores

| Category | Score | Level | Progress |
|----------|-------|-------|----------|
${Object.entries(result.categoryScores)
  .map(([cat, data]) =>
    `| ${cat.toUpperCase()} | ${data.score}% | Level ${data.level} | ${bars[cat]} |`
  )
  .join('\n')}

## Strengths
${result.strengths.map((s) => `✅ ${s}`).join('\n')}

## Areas for Improvement
${result.gaps.map((g) => `🔴 ${g}`).join('\n')}

## Top Recommendations
${result.topRecommendations.map((r, i) => `${i + 1}. ${r}`).join('\n')}

## Migration Roadmap

${result.roadmap
  .map(
    (item) =>
      `### ${item.priority.toUpperCase()}: ${item.action}
- **Category**: ${item.category}
- **Effort**: ${item.estimatedEffort}
- **Impact**: ${item.impact.toUpperCase()}`
  )
  .join('\n\n')}
`;
}
```

---

## 90.8 สรุป

Microservices Maturity Model เป็นเครื่องมือสำคัญสำหรับ:

1. **ทำความเข้าใจจุดยืน**: รู้ว่าองค์กรอยู่ที่ระดับใดใน 5 ระดับ
2. **วางแผนการพัฒนา**: มี roadmap ที่ชัดเจนสำหรับการก้าวไปสู่ระดับถัดไป
3. **วัดความก้าวหน้า**: ใช้ DORA metrics เป็น objective measurements
4. **สื่อสารกับ stakeholders**: แสดงให้เห็นว่าการลงทุนใน infrastructure จะช่วยธุรกิจได้อย่างไร

สิ่งสำคัญคือการก้าวไปสู่ระดับถัดไปต้องทำทีละขั้น ไม่ใช่กระโดดข้ามขั้นตอน การรีบเร่งโดยขาด foundation ที่แข็งแกร่งมักนำไปสู่ปัญหาในระยะยาว

---

## แบบฝึกหัด

1. ใช้ MaturityAssessment class ประเมินองค์กรหรือทีมของคุณ และบันทึกผลลัพธ์

2. สร้าง migration roadmap จากผล assessment โดยกำหนด timeline และ owner สำหรับแต่ละ action item

3. Deploy StranglerFigProxy ในสภาพแวดล้อม development และทดลองค่อย ๆ เพิ่ม traffic percentage ไปยัง new service

4. ใช้ OpsExcellenceChecker วัด DORA metrics ของ service ที่คุณดูแล และเปรียบเทียบกับ benchmark

5. วาด Team Autonomy radar chart สำหรับทีมของคุณ และระบุ 3 สิ่งที่ต้องปรับปรุงมากที่สุด

---

*ต่อไปใน Part 98: Cost Optimization Strategies - วิธีลดค่าใช้จ่ายใน microservices infrastructure โดยไม่ลดคุณภาพ*

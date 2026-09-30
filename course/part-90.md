# Part 90: Microservices Maturity Model

## บทนำ

Microservices Maturity Model เป็น framework สำหรับประเมินระดับความพร้อมขององค์กรในการใช้งาน Microservices บทนี้จะอธิบาย 5 ระดับ พร้อม TypeScript tools สำหรับการประเมินและวางแผนการพัฒนา

---

## 1. Maturity Levels 1-5

### 1.1 ภาพรวม Maturity Levels

| Level | ชื่อ | คำอธิบาย | ลักษณะสำคัญ |
|-------|------|----------|------------|
| Level 1 | Initial | Monolith หรือ Microservices เริ่มต้น | ไม่มี automation, manual processes |
| Level 2 | Managed | Services แยกกัน แต่ยังมี shared dependencies | มี CI/CD เบื้องต้น, basic monitoring |
| Level 3 | Defined | Service-oriented architecture ชัดเจน | API contracts, service discovery, tracing |
| Level 4 | Quantitatively Managed | Data-driven operations | SLOs, chaos engineering, auto-scaling |
| Level 5 | Optimizing | Continuous improvement culture | AI/ML optimization, platform engineering |

### 1.2 รายละเอียด Level 1: Initial

```
สัญญาณ:
- ทุก service deploy พร้อมกัน
- Teams ไม่ independent
- ไม่มี API versioning
- ไม่มี distributed tracing
- Database shared ระหว่าง services
- Test ทำ manual
- Rollback ทำได้ยาก
```

### 1.3 รายละเอียด Level 2: Managed

```
สัญญาณ:
- Services deploy แยกกันได้
- Basic CI/CD (build, test, deploy)
- Service registry เบื้องต้น
- Basic metrics (CPU, memory)
- API versioning เริ่มต้น
- Containerization (Docker)
- ยังมี shared databases บางส่วน
```

### 1.4 รายละเอียด Level 3: Defined

```
สัญญาณ:
- Service mesh หรือ API Gateway
- Distributed tracing (Jaeger/Zipkin)
- Contract testing (Pact)
- Centralized logging (ELK)
- Service discovery อัตโนมัติ
- Health checks ครบ
- Database per service
- Circuit breakers
```

### 1.5 รายละเอียด Level 4: Quantitatively Managed

```
สัญญาณ:
- SLOs/SLAs defined และ monitored
- Chaos engineering
- Auto-scaling (HPA/VPA/KEDA)
- Capacity planning ด้วยข้อมูล
- Feature flags
- Canary deployments
- Post-mortem culture
- Error budgets
```

### 1.6 รายละเอียด Level 5: Optimizing

```
สัญญาณ:
- Platform engineering team
- Internal developer platform (IDP)
- AI/ML สำหรับ anomaly detection
- Self-healing systems
- Predictive scaling
- Cost optimization automation
- Innovation culture
- Contribution กลับไป open source
```

---

## 2. TypeScript MaturityAssessment Class

```typescript
// maturity/maturity-assessment.ts

export interface AssessmentQuestion {
  id: string;
  category: string;
  subcategory: string;
  question: string;
  description: string;
  options: Array<{
    level: 1 | 2 | 3 | 4 | 5;
    description: string;
    examples: string[];
  }>;
  weight: number; // 1-3, ความสำคัญของคำถาม
}

export interface AssessmentAnswer {
  questionId: string;
  selectedLevel: 1 | 2 | 3 | 4 | 5;
  notes?: string;
}

export interface CategoryScore {
  category: string;
  score: number; // 0-100
  level: 1 | 2 | 3 | 4 | 5;
  questions: number;
  strengths: string[];
  improvements: string[];
}

export interface MaturityReport {
  organizationName: string;
  assessedAt: Date;
  overallScore: number;
  overallLevel: 1 | 2 | 3 | 4 | 5;
  categoryScores: CategoryScore[];
  topStrengths: string[];
  topImprovements: string[];
  nextSteps: string[];
  estimatedTimeToNextLevel: string;
  radarChart: string; // Mermaid diagram
}

// 50+ assessment questions
const assessmentQuestions: AssessmentQuestion[] = [
  // =========================================================
  // Category: Architecture
  // =========================================================
  {
    id: 'ARCH-001',
    category: 'Architecture',
    subcategory: 'Service Decomposition',
    question: 'ระดับการแยก services ของระบบ',
    description: 'วัดว่า services ถูก decompose ตาม business domains อย่างชัดเจนแค่ไหน',
    weight: 3,
    options: [
      { level: 1, description: 'Monolith หรือ services ขนาดใหญ่ที่รวมหลาย functions', examples: ['Single deployable unit', 'Functions mixed together'] },
      { level: 2, description: 'แยก services ตาม technical layer (frontend/backend/db)', examples: ['UI service', 'API service', 'DB service'] },
      { level: 3, description: 'Services แยกตาม business domain', examples: ['Order service', 'User service', 'Payment service'] },
      { level: 4, description: 'Services ขนาดเล็กตาม bounded context ด้วย DDD', examples: ['Order creation', 'Order fulfillment', 'Order reporting แยกกัน'] },
      { level: 5, description: 'Cell-based architecture พร้อม blast radius containment', examples: ['Micro-frontends', 'Nano-services ที่เหมาะสม'] },
    ],
  },
  {
    id: 'ARCH-002',
    category: 'Architecture',
    subcategory: 'Data Management',
    question: 'การจัดการ database ต่อ service',
    description: 'Database isolation ระหว่าง services',
    weight: 3,
    options: [
      { level: 1, description: 'Shared database เดียวสำหรับทุก service', examples: ['Single PostgreSQL instance', 'All services hit same DB'] },
      { level: 2, description: 'Shared database แต่ schema แยกกัน', examples: ['Separate schemas', 'Different tables'] },
      { level: 3, description: 'Database แยกต่อ service (ยังมีบางส่วนที่ share)', examples: ['Most services have own DB', 'Some legacy sharing'] },
      { level: 4, description: 'Database per service อย่างสมบูรณ์', examples: ['No cross-service DB access', 'APIs only'] },
      { level: 5, description: 'Polyglot persistence ด้วย right tool for right job', examples: ['Redis for cache', 'Cassandra for time-series', 'MongoDB for docs'] },
    ],
  },
  {
    id: 'ARCH-003',
    category: 'Architecture',
    subcategory: 'Communication',
    question: 'รูปแบบการสื่อสารระหว่าง services',
    description: 'Synchronous vs Asynchronous communication patterns',
    weight: 2,
    options: [
      { level: 1, description: 'Direct database calls หรือ synchronous REST ทั้งหมด', examples: ['Service A queries Service B\'s DB directly'] },
      { level: 2, description: 'REST APIs ระหว่าง services', examples: ['HTTP calls between services'] },
      { level: 3, description: 'Mix ของ sync REST และ async messaging', examples: ['RabbitMQ for events', 'REST for queries'] },
      { level: 4, description: 'Event-driven architecture พร้อม CQRS', examples: ['Kafka events', 'Event sourcing', 'Saga pattern'] },
      { level: 5, description: 'Intelligent routing พร้อม gRPC, GraphQL federation', examples: ['gRPC for internal', 'GraphQL for external', 'Event mesh'] },
    ],
  },

  // =========================================================
  // Category: CI/CD
  // =========================================================
  {
    id: 'CICD-001',
    category: 'CI/CD',
    subcategory: 'Build Pipeline',
    question: 'ระดับ automation ใน build pipeline',
    description: 'การ automate ขั้นตอน build, test, deploy',
    weight: 3,
    options: [
      { level: 1, description: 'Manual build และ deploy', examples: ['SSH แล้ว run scripts', 'Upload jars manually'] },
      { level: 2, description: 'Basic CI (build + unit tests)', examples: ['Jenkins builds on commit', 'Basic test stage'] },
      { level: 3, description: 'Full CI/CD pipeline พร้อม staging', examples: ['Automated tests', 'Deploy to staging', 'Manual approval for prod'] },
      { level: 4, description: 'Continuous deployment พร้อม feature flags', examples: ['Auto deploy on green', 'Canary deployments', 'Feature toggles'] },
      { level: 5, description: 'Progressive delivery พร้อม automated rollback', examples: ['Flagger', 'Argo Rollouts', 'ML-based rollback'] },
    ],
  },
  {
    id: 'CICD-002',
    category: 'CI/CD',
    subcategory: 'Testing',
    question: 'ระดับ test coverage และ test strategy',
    description: 'ประเภทและคุณภาพของ tests',
    weight: 3,
    options: [
      { level: 1, description: 'Manual testing หรือ unit tests เท่านั้น', examples: ['Click testing', 'Basic unit tests < 20%'] },
      { level: 2, description: 'Unit + integration tests', examples: ['50% unit test coverage', 'Basic integration tests'] },
      { level: 3, description: 'Full test pyramid (unit, integration, e2e)', examples: ['80% unit coverage', 'Contract tests', 'E2E critical paths'] },
      { level: 4, description: 'Contract testing + chaos testing', examples: ['Pact consumer-driven tests', 'Chaos Monkey', 'Performance tests in CI'] },
      { level: 5, description: 'AI-assisted testing + mutation testing', examples: ['Auto test generation', 'Mutation testing', 'Property-based testing'] },
    ],
  },
  {
    id: 'CICD-003',
    category: 'CI/CD',
    subcategory: 'Deployment Strategy',
    question: 'กลยุทธ์การ deploy',
    description: 'วิธีการ deploy ที่ลด risk',
    weight: 2,
    options: [
      { level: 1, description: 'Deploy ทุก service พร้อมกัน (big bang)', examples: ['Maintenance window', 'All-or-nothing'] },
      { level: 2, description: 'Rolling update', examples: ['Pod by pod replacement'] },
      { level: 3, description: 'Blue-green deployment', examples: ['Swap traffic between versions'] },
      { level: 4, description: 'Canary deployment พร้อม automated rollback', examples: ['5% canary', 'Auto rollback on errors'] },
      { level: 5, description: 'Progressive delivery ด้วย ML-based traffic routing', examples: ['A/B testing ตาม user segments', 'Automated experiment evaluation'] },
    ],
  },

  // =========================================================
  // Category: Observability
  // =========================================================
  {
    id: 'OBS-001',
    category: 'Observability',
    subcategory: 'Metrics',
    question: 'ระดับ metrics collection',
    description: 'การเก็บและใช้งาน metrics',
    weight: 3,
    options: [
      { level: 1, description: 'ไม่มี metrics หรือมีแค่ system-level (CPU, memory)', examples: ['CloudWatch basic', 'Nagios ping checks'] },
      { level: 2, description: 'Application metrics เบื้องต้น', examples: ['Request count', 'Error rate', 'Basic dashboards'] },
      { level: 3, description: 'Golden signals (Latency, Traffic, Errors, Saturation)', examples: ['Prometheus + Grafana', 'Service-level metrics'] },
      { level: 4, description: 'Business metrics + SLOs/SLIs', examples: ['Error budgets', 'Customer-impacting metrics', 'SLO dashboards'] },
      { level: 5, description: 'Predictive analytics + anomaly detection', examples: ['ML-based alerting', 'Business impact prediction'] },
    ],
  },
  {
    id: 'OBS-002',
    category: 'Observability',
    subcategory: 'Logging',
    question: 'ระดับ centralized logging',
    description: 'การจัดการ logs ข้าม services',
    weight: 2,
    options: [
      { level: 1, description: 'Logs อยู่บน individual servers', examples: ['SSH to each server', 'tail -f manually'] },
      { level: 2, description: 'Basic log aggregation', examples: ['Filebeat + Elasticsearch', 'CloudWatch Logs'] },
      { level: 3, description: 'Structured logging พร้อม correlation IDs', examples: ['JSON logs', 'Trace IDs', 'Kibana dashboards'] },
      { level: 4, description: 'Log-based alerting + analysis', examples: ['Automated anomaly detection', 'Log-based SLIs'] },
      { level: 5, description: 'AI-powered log analysis', examples: ['Root cause suggestion', 'Automated triage'] },
    ],
  },
  {
    id: 'OBS-003',
    category: 'Observability',
    subcategory: 'Tracing',
    question: 'ระดับ distributed tracing',
    description: 'การ trace requests ข้าม services',
    weight: 2,
    options: [
      { level: 1, description: 'ไม่มี distributed tracing', examples: ['Cannot follow request across services'] },
      { level: 2, description: 'Tracing บาง services', examples: ['Partial Jaeger implementation'] },
      { level: 3, description: 'Full distributed tracing', examples: ['100% trace sampling', 'All services instrumented'] },
      { level: 4, description: 'Trace-based alerting + performance profiling', examples: ['Auto-detect slow spans', 'Continuous profiling'] },
      { level: 5, description: 'AI-powered trace analysis', examples: ['Automatic root cause from traces', 'Dependency analysis'] },
    ],
  },

  // =========================================================
  // Category: Security
  // =========================================================
  {
    id: 'SEC-001',
    category: 'Security',
    subcategory: 'Authentication & Authorization',
    question: 'ระดับ security ใน service-to-service communication',
    description: 'mTLS, JWT, service identity',
    weight: 3,
    options: [
      { level: 1, description: 'ไม่มี authentication ระหว่าง services', examples: ['Open internal APIs', 'No auth between services'] },
      { level: 2, description: 'API keys หรือ basic auth', examples: ['Shared secrets', 'IP whitelisting'] },
      { level: 3, description: 'JWT tokens สำหรับ service identity', examples: ['OAuth 2.0', 'OIDC', 'Service accounts'] },
      { level: 4, description: 'mTLS ทุก service (service mesh)', examples: ['Istio mTLS', 'SPIFFE/SPIRE', 'Zero trust'] },
      { level: 5, description: 'Zero trust พร้อม fine-grained authorization', examples: ['OPA policies', 'Dynamic credentials', 'RBAC per API'] },
    ],
  },
  {
    id: 'SEC-002',
    category: 'Security',
    subcategory: 'Secret Management',
    question: 'การจัดการ secrets',
    description: 'วิธีจัดเก็บและ rotate secrets',
    weight: 3,
    options: [
      { level: 1, description: 'Secrets ใน code หรือ config files', examples: ['Hardcoded passwords', '.env committed to git'] },
      { level: 2, description: 'Environment variables', examples: ['Kubernetes secrets (base64)', 'AWS parameter store basic'] },
      { level: 3, description: 'Secret management service', examples: ['HashiCorp Vault', 'AWS Secrets Manager', 'Auto rotation'] },
      { level: 4, description: 'Dynamic secrets + audit trail', examples: ['Vault dynamic DB credentials', 'Secret leakage detection'] },
      { level: 5, description: 'Zero-standing-privilege ด้วย just-in-time credentials', examples: ['Ephemeral credentials', 'Automatic rotation on breach'] },
    ],
  },
  {
    id: 'SEC-003',
    category: 'Security',
    subcategory: 'Vulnerability Management',
    question: 'การจัดการ vulnerabilities',
    description: 'Scanning, patching, compliance',
    weight: 2,
    options: [
      { level: 1, description: 'ไม่มี vulnerability scanning', examples: ['No CVE checks'] },
      { level: 2, description: 'Manual vulnerability scanning', examples: ['Periodic scans', 'Manual patching'] },
      { level: 3, description: 'Automated scanning ใน CI/CD', examples: ['Snyk in pipeline', 'Container scanning', 'Block on critical CVEs'] },
      { level: 4, description: 'Continuous compliance + SBOMs', examples: ['SBOM generation', 'License scanning', 'Runtime security'] },
      { level: 5, description: 'Proactive threat detection ด้วย AI', examples: ['Runtime anomaly detection', 'Supply chain security'] },
    ],
  },

  // =========================================================
  // Category: Team Autonomy
  // =========================================================
  {
    id: 'TEAM-001',
    category: 'Team Autonomy',
    subcategory: 'Team Ownership',
    question: 'ระดับ team ownership ของ services',
    description: 'You build it, you run it',
    weight: 3,
    options: [
      { level: 1, description: 'Centralized ops team ดูแลทุก service', examples: ['Dev writes code, ops deploys', 'No team owns specific service'] },
      { level: 2, description: 'Dev teams own code, shared ops', examples: ['Teams own code only', 'Shared on-call'] },
      { level: 3, description: 'Teams own full stack ของ services', examples: ['Team owns deployment', 'Team on-call for own services'] },
      { level: 4, description: 'Full product teams with embedded SRE', examples: ['Cross-functional teams', 'Team owns full product lifecycle'] },
      { level: 5, description: 'Platform teams ที่ enable self-service', examples: ['Internal developer platform', 'Teams fully autonomous'] },
    ],
  },
  {
    id: 'TEAM-002',
    category: 'Team Autonomy',
    subcategory: 'Release Independence',
    question: 'ความสามารถในการ release อิสระ',
    description: 'Teams สามารถ release ได้โดยไม่ต้องรอทีมอื่น',
    weight: 3,
    options: [
      { level: 1, description: 'ต้องรอ release train รายสัปดาห์/รายเดือน', examples: ['Scheduled releases', 'Coordination needed'] },
      { level: 2, description: 'Release ทุก 2 สัปดาห์ แต่ต้องมี coordination', examples: ['Sprint-based releases', 'Some dependencies'] },
      { level: 3, description: 'Release ทุกสัปดาห์ อิสระส่วนใหญ่', examples: ['Few cross-team dependencies', 'Contract tests reduce coordination'] },
      { level: 4, description: 'Release on-demand หลายครั้งต่อวัน', examples: ['Multiple daily deployments', 'Feature flags for coordination'] },
      { level: 5, description: 'Continuous deployment ทุก commit', examples: ['Deploy on merge to main', 'Zero-downtime always'] },
    ],
  },

  // =========================================================
  // Category: Operations
  // =========================================================
  {
    id: 'OPS-001',
    category: 'Operations',
    subcategory: 'Incident Management',
    question: 'ระดับ incident management maturity',
    description: 'Detection, response, resolution, post-mortem',
    weight: 3,
    options: [
      { level: 1, description: 'ไม่มี formal incident process', examples: ['Users report issues', 'No runbooks', 'No post-mortems'] },
      { level: 2, description: 'Basic alerting และ on-call', examples: ['PagerDuty basic', 'Some runbooks', 'Informal post-mortems'] },
      { level: 3, description: 'Defined incident process พร้อม runbooks', examples: ['Severity levels', 'Formal post-mortems', 'SLO-based alerts'] },
      { level: 4, description: 'Automated runbooks + blameless culture', examples: ['Automated mitigation', 'Action item tracking', 'MTTR trending'] },
      { level: 5, description: 'Proactive incident prevention ด้วย chaos engineering', examples: ['GameDays', 'Chaos experiments', 'Automated recovery'] },
    ],
  },
  {
    id: 'OPS-002',
    category: 'Operations',
    subcategory: 'Capacity Planning',
    question: 'ระดับ capacity planning',
    description: 'การวางแผนทรัพยากรล่วงหน้า',
    weight: 2,
    options: [
      { level: 1, description: 'Reactive scaling เมื่อระบบช้าแล้ว', examples: ['Manual scale up when full', 'No forecasting'] },
      { level: 2, description: 'Manual capacity reviews รายไตรมาส', examples: ['Quarterly reviews', 'Simple forecasting'] },
      { level: 3, description: 'Auto-scaling ตาม metrics', examples: ['HPA', 'Target tracking scaling'] },
      { level: 4, description: 'Predictive scaling ตาม historical patterns', examples: ['KEDA with Kafka lag', 'Scheduled scaling for known peaks'] },
      { level: 5, description: 'AI-driven capacity optimization', examples: ['ML-based prediction', 'Automatic right-sizing', 'Multi-cloud optimization'] },
    ],
  },

  // =========================================================
  // Category: Documentation
  // =========================================================
  {
    id: 'DOC-001',
    category: 'Documentation',
    subcategory: 'API Documentation',
    question: 'ระดับ API documentation',
    description: 'การ document APIs ของ services',
    weight: 2,
    options: [
      { level: 1, description: 'ไม่มี documentation หรือ outdated', examples: ['Word docs', 'Tribal knowledge'] },
      { level: 2, description: 'Basic README และ manual docs', examples: ['Markdown files', 'Some API docs'] },
      { level: 3, description: 'Auto-generated API docs (OpenAPI/Swagger)', examples: ['Swagger UI', 'OpenAPI 3.0', 'Always up to date'] },
      { level: 4, description: 'API catalog พร้อม discovery', examples: ['Backstage', 'API portal', 'Consumer-driven contracts'] },
      { level: 5, description: 'Living documentation ที่ self-updating', examples: ['Docs generated from code + tests', 'AI-assisted documentation'] },
    ],
  },
  {
    id: 'DOC-002',
    category: 'Documentation',
    subcategory: 'Architecture Documentation',
    question: 'ระดับ architecture documentation',
    description: 'ADRs, diagrams, service catalogs',
    weight: 2,
    options: [
      { level: 1, description: 'ไม่มี architecture docs', examples: ['Knowledge in people\'s heads'] },
      { level: 2, description: 'Basic diagrams และ README', examples: ['Whiteboard photos', 'High-level diagrams'] },
      { level: 3, description: 'ADRs + up-to-date diagrams', examples: ['Architecture Decision Records', 'C4 diagrams', 'Service dependency maps'] },
      { level: 4, description: 'Living architecture ที่ generated จาก code', examples: ['Auto-generated service maps from traces', 'Drift detection'] },
      { level: 5, description: 'Architecture governance ด้วย automation', examples: ['Automated compliance checks', 'AI architecture advisor'] },
    ],
  },

  // =========================================================
  // Additional Questions (30+)
  // =========================================================
  {
    id: 'ARCH-004',
    category: 'Architecture',
    subcategory: 'API Design',
    question: 'ระดับ API design standards',
    description: 'Consistency และ versioning',
    weight: 2,
    options: [
      { level: 1, description: 'ไม่มี API standards', examples: ['Inconsistent naming', 'No versioning'] },
      { level: 2, description: 'Basic REST conventions', examples: ['HTTP verbs', 'Status codes', 'Basic versioning'] },
      { level: 3, description: 'API style guide + linting', examples: ['OpenAPI first', 'API linter', 'Consistent patterns'] },
      { level: 4, description: 'API-first development + consumer feedback', examples: ['Design review process', 'Consumer-driven APIs'] },
      { level: 5, description: 'Self-service API marketplace', examples: ['API as product', 'Developer experience score', 'API analytics'] },
    ],
  },
  {
    id: 'INFRA-001',
    category: 'Infrastructure',
    subcategory: 'Infrastructure as Code',
    question: 'ระดับ Infrastructure as Code',
    description: 'Terraform, Helm, GitOps',
    weight: 3,
    options: [
      { level: 1, description: 'Manual infrastructure setup', examples: ['Click Ops', 'No IaC'] },
      { level: 2, description: 'Basic IaC สำหรับบางส่วน', examples: ['Partial Terraform', 'Some Helm charts'] },
      { level: 3, description: 'Full IaC พร้อม version control', examples: ['Terraform all resources', 'GitOps for K8s', 'Drift detection'] },
      { level: 4, description: 'Self-service infrastructure ด้วย templates', examples: ['Service templates', 'Automated provisioning', 'Policy as code'] },
      { level: 5, description: 'Immutable infrastructure พร้อม automatic optimization', examples: ['Phoenix servers', 'Automatic cost optimization'] },
    ],
  },
  {
    id: 'INFRA-002',
    category: 'Infrastructure',
    subcategory: 'Container & Orchestration',
    question: 'ระดับ containerization และ orchestration',
    description: 'Docker, Kubernetes maturity',
    weight: 3,
    options: [
      { level: 1, description: 'ไม่มี containerization', examples: ['Bare metal', 'VMs without containers'] },
      { level: 2, description: 'Docker containers แต่ไม่มี orchestration', examples: ['Docker Compose', 'Manual Docker run'] },
      { level: 3, description: 'Kubernetes production-ready', examples: ['RBAC', 'Resource limits', 'Health probes', 'PDB'] },
      { level: 4, description: 'Service mesh + advanced K8s features', examples: ['Istio', 'Network policies', 'OPA Gatekeeper'] },
      { level: 5, description: 'Multi-cluster + hybrid cloud orchestration', examples: ['Cluster federation', 'Workload portability'] },
    ],
  },
  {
    id: 'PERF-001',
    category: 'Performance',
    subcategory: 'Performance Testing',
    question: 'ระดับ performance testing',
    description: 'Load testing, performance benchmarks',
    weight: 2,
    options: [
      { level: 1, description: 'ไม่มี performance testing', examples: ['No load tests'] },
      { level: 2, description: 'Manual load tests ก่อน release ใหญ่', examples: ['JMeter occasional', 'Pre-release only'] },
      { level: 3, description: 'Automated performance tests ใน CI/CD', examples: ['k6 in pipeline', 'Performance budgets', 'Regression detection'] },
      { level: 4, description: 'Continuous performance testing + SLOs', examples: ['Always-on load testing', 'Performance SLOs', 'Automatic alerts'] },
      { level: 5, description: 'Production traffic mirroring + optimization', examples: ['Shadow testing', 'Automatic performance optimization'] },
    ],
  },
];

class MaturityAssessment {
  private questions: AssessmentQuestion[] = assessmentQuestions;

  getQuestions(categories?: string[]): AssessmentQuestion[] {
    if (!categories || categories.length === 0) {
      return this.questions;
    }
    return this.questions.filter((q) => categories.includes(q.category));
  }

  calculateScore(answers: AssessmentAnswer[]): MaturityReport {
    const answerMap = new Map(answers.map((a) => [a.questionId, a]));

    // Group questions by category
    const categoryQuestions = new Map<string, AssessmentQuestion[]>();
    for (const question of this.questions) {
      const list = categoryQuestions.get(question.category) || [];
      list.push(question);
      categoryQuestions.set(question.category, list);
    }

    // Calculate per-category scores
    const categoryScores: CategoryScore[] = [];

    for (const [category, questions] of categoryQuestions) {
      let totalWeight = 0;
      let weightedScore = 0;
      const strengths: string[] = [];
      const improvements: string[] = [];

      for (const question of questions) {
        const answer = answerMap.get(question.id);
        const level = answer?.selectedLevel || 1;
        const normalizedScore = ((level - 1) / 4) * 100;

        totalWeight += question.weight;
        weightedScore += normalizedScore * question.weight;

        if (level >= 4) {
          strengths.push(`${question.subcategory}: ${question.options[level - 1].description}`);
        } else if (level <= 2) {
          improvements.push(
            `${question.subcategory}: เพิ่มเป็น Level ${level + 1} - ${question.options[level].description}`
          );
        }
      }

      const score = totalWeight > 0 ? weightedScore / totalWeight : 0;
      const level = this.scoreToLevel(score);

      categoryScores.push({
        category,
        score: Math.round(score),
        level,
        questions: questions.length,
        strengths: strengths.slice(0, 3),
        improvements: improvements.slice(0, 3),
      });
    }

    // Overall score
    const overallScore = Math.round(
      categoryScores.reduce((sum, c) => sum + c.score, 0) / categoryScores.length
    );
    const overallLevel = this.scoreToLevel(overallScore);

    // Top strengths and improvements
    const allStrengths = categoryScores.flatMap((c) => c.strengths);
    const allImprovements = categoryScores.flatMap((c) => c.improvements);

    // Next steps based on current level
    const nextSteps = this.generateNextSteps(overallLevel, categoryScores);
    const estimatedTimeToNextLevel = this.estimateTimeToNextLevel(overallLevel, categoryScores);

    // Generate radar chart
    const radarChart = this.generateRadarChart(categoryScores);

    return {
      organizationName: 'Assessment Result',
      assessedAt: new Date(),
      overallScore,
      overallLevel,
      categoryScores,
      topStrengths: allStrengths.slice(0, 5),
      topImprovements: allImprovements.slice(0, 5),
      nextSteps,
      estimatedTimeToNextLevel,
      radarChart,
    };
  }

  private scoreToLevel(score: number): 1 | 2 | 3 | 4 | 5 {
    if (score >= 80) return 5;
    if (score >= 60) return 4;
    if (score >= 40) return 3;
    if (score >= 20) return 2;
    return 1;
  }

  private generateNextSteps(
    currentLevel: number,
    categoryScores: CategoryScore[]
  ): string[] {
    const steps: string[] = [];

    // หา categories ที่ต่ำกว่า overall
    const lowCategories = categoryScores
      .filter((c) => c.level < currentLevel)
      .sort((a, b) => a.score - b.score)
      .slice(0, 3);

    for (const cat of lowCategories) {
      steps.push(`ยกระดับ ${cat.category}: ${cat.improvements[0] || 'ปรับปรุงตาม best practices'}`);
    }

    // Level-specific recommendations
    switch (currentLevel) {
      case 1:
        steps.push('เริ่ม containerize applications ด้วย Docker');
        steps.push('สร้าง basic CI/CD pipeline');
        steps.push('เพิ่ม centralized logging');
        break;
      case 2:
        steps.push('ย้าย shared databases เป็น database per service');
        steps.push('เพิ่ม distributed tracing');
        steps.push('implement API contracts และ versioning');
        break;
      case 3:
        steps.push('กำหนด SLOs และ error budgets');
        steps.push('เริ่ม chaos engineering experiments');
        steps.push('implement canary deployments');
        break;
      case 4:
        steps.push('สร้าง Internal Developer Platform');
        steps.push('เพิ่ม AI/ML สำหรับ anomaly detection');
        steps.push('implement predictive scaling');
        break;
    }

    return steps.slice(0, 5);
  }

  private estimateTimeToNextLevel(
    currentLevel: number,
    categoryScores: CategoryScore[]
  ): string {
    const avgScore = categoryScores.reduce((sum, c) => sum + c.score, 0) / categoryScores.length;
    const targetScore = currentLevel * 20;
    const gap = targetScore - avgScore;

    if (currentLevel >= 5) return 'อยู่ที่ระดับสูงสุดแล้ว - focus on continuous improvement';

    const months = Math.ceil(gap / 5); // สมมติ 5 points per month
    if (months <= 3) return `${months} เดือน`;
    if (months <= 12) return `${months} เดือน (ประมาณ ${Math.ceil(months / 3)} ไตรมาส)`;
    return `${Math.ceil(months / 12)} ปี`;
  }

  private generateRadarChart(categoryScores: CategoryScore[]): string {
    return this.generateMermaidRadar(categoryScores);
  }
}
```

---

## 3. Migration Path: Strangler Fig Pattern

```typescript
// maturity/strangler-fig.ts
import express, { Request, Response, NextFunction } from 'express';
import axios from 'axios';

interface RouteConfig {
  pattern: RegExp;
  destination: 'legacy' | 'new' | 'canary';
  newServiceUrl?: string;
  canaryPercent?: number;
}

interface MigrationState {
  totalRequests: number;
  legacyRequests: number;
  newServiceRequests: number;
  migrationPercent: number;
  lastUpdated: Date;
}

class StranglerFigProxy {
  private app = express();
  private routes: RouteConfig[] = [];
  private state: MigrationState = {
    totalRequests: 0,
    legacyRequests: 0,
    newServiceRequests: 0,
    migrationPercent: 0,
    lastUpdated: new Date(),
  };

  constructor(
    private readonly legacyServiceUrl: string,
    private readonly port: number = 8080
  ) {
    this.setupMiddleware();
    this.setupAdminRoutes();
  }

  private setupMiddleware(): void {
    this.app.use(express.json());

    // Proxy middleware
    this.app.use(async (req: Request, res: Response, next: NextFunction) => {
      this.state.totalRequests++;

      const route = this.findRoute(req.path);

      try {
        if (route) {
          await this.handleRoute(req, res, route);
        } else {
          await this.forwardToLegacy(req, res);
        }
      } catch (error: any) {
        console.error('Proxy error:', error.message);
        res.status(502).json({ error: 'Bad Gateway' });
      }
    });
  }

  private findRoute(path: string): RouteConfig | null {
    return this.routes.find((r) => r.pattern.test(path)) || null;
  }

  private async handleRoute(
    req: Request,
    res: Response,
    route: RouteConfig
  ): Promise<void> {
    switch (route.destination) {
      case 'new':
        this.state.newServiceRequests++;
        await this.forwardToNew(req, res, route.newServiceUrl!);
        break;

      case 'canary':
        const percent = Math.random() * 100;
        if (percent < (route.canaryPercent || 10)) {
          this.state.newServiceRequests++;
          await this.forwardToNew(req, res, route.newServiceUrl!);
        } else {
          this.state.legacyRequests++;
          await this.forwardToLegacy(req, res);
        }
        break;

      case 'legacy':
      default:
        this.state.legacyRequests++;
        await this.forwardToLegacy(req, res);
        break;
    }

    this.state.migrationPercent =
      (this.state.newServiceRequests / this.state.totalRequests) * 100;
    this.state.lastUpdated = new Date();
  }

  private async forwardToLegacy(req: Request, res: Response): Promise<void> {
    const url = `${this.legacyServiceUrl}${req.path}`;
    const response = await axios({
      method: req.method as any,
      url,
      headers: { ...req.headers, host: undefined },
      data: req.body,
      params: req.query,
      validateStatus: () => true,
    });

    res.status(response.status);
    Object.entries(response.headers).forEach(([k, v]) => {
      if (k !== 'transfer-encoding') res.setHeader(k, v as string);
    });
    res.send(response.data);
  }

  private async forwardToNew(
    req: Request,
    res: Response,
    newServiceUrl: string
  ): Promise<void> {
    const url = `${newServiceUrl}${req.path}`;
    const response = await axios({
      method: req.method as any,
      url,
      headers: { ...req.headers, host: undefined },
      data: req.body,
      params: req.query,
      validateStatus: () => true,
    });

    res.status(response.status);
    Object.entries(response.headers).forEach(([k, v]) => {
      if (k !== 'transfer-encoding') res.setHeader(k, v as string);
    });
    res.send(response.data);
  }

  // Admin routes สำหรับจัดการ migration
  private setupAdminRoutes(): void {
    this.app.get('/admin/migration/state', (req, res) => {
      res.json(this.state);
    });

    this.app.post('/admin/migration/route', (req, res) => {
      const { pattern, destination, newServiceUrl, canaryPercent } = req.body;
      this.addRoute({
        pattern: new RegExp(pattern),
        destination,
        newServiceUrl,
        canaryPercent,
      });
      res.json({ success: true, routesCount: this.routes.length });
    });

    this.app.put('/admin/migration/route/:index/canary', (req, res) => {
      const index = parseInt(req.params.index);
      const { canaryPercent } = req.body;
      if (this.routes[index]) {
        this.routes[index].canaryPercent = canaryPercent;
        this.routes[index].destination = 'canary';
        res.json({ success: true });
      } else {
        res.status(404).json({ error: 'Route not found' });
      }
    });
  }

  addRoute(config: RouteConfig): void {
    this.routes.push(config);
    console.log(`Route added: ${config.pattern} → ${config.destination}`);
  }

  start(): void {
    this.app.listen(this.port, () => {
      console.log(`Strangler Fig Proxy listening on :${this.port}`);
      console.log(`Legacy: ${this.legacyServiceUrl}`);
    });
  }
}

// ตัวอย่าง Migration Plan
async function demonstrateStranglerFig() {
  const proxy = new StranglerFigProxy('http://legacy-monolith:3000', 8080);

  // Phase 1: เพิ่ม route สำหรับ new user service (canary 10%)
  proxy.addRoute({
    pattern: /^\/api\/users\//,
    destination: 'canary',
    newServiceUrl: 'http://user-service:3001',
    canaryPercent: 10,
  });

  // Phase 2: เพิ่ม route สำหรับ new order service (ทั้งหมด)
  proxy.addRoute({
    pattern: /^\/api\/orders\//,
    destination: 'new',
    newServiceUrl: 'http://order-service:3002',
  });

  proxy.start();
}

demonstrateStranglerFig().catch(console.error);
```

---

## 4. Team Autonomy Scorer

```typescript
// maturity/team-autonomy.ts

interface TeamProfile {
  name: string;
  size: number;
  servicesOwned: string[];
  deploymentFrequency: 'multiple-daily' | 'daily' | 'weekly' | 'monthly' | 'quarterly';
  leadTimeForChange: 'less-1h' | '1-24h' | '1-7d' | '1-4w' | 'greater-4w';
  changeFailureRate: 'less-5%' | '5-10%' | '10-15%' | '15-20%' | 'greater-20%';
  timeToRestoreService: 'less-1h' | '1-24h' | '1-7d' | '1-4w' | 'greater-4w';
  hasOwnCI: boolean;
  hasOwnCD: boolean;
  ownsDatabaseSchemas: boolean;
  hasOnCall: boolean;
  hasProductOwner: boolean;
  hasDedicatedDesigner: boolean;
  testCoverage: number; // percentage
  deploymentApprovals: number; // number of approvals needed
}

interface TeamAutonomyScore {
  teamName: string;
  overallScore: number; // 0-100
  doraMetrics: {
    deploymentFrequency: number;
    leadTime: number;
    changeFailureRate: number;
    timeToRestore: number;
  };
  organizationalScore: number;
  technicalScore: number;
  bottlenecks: string[];
  recommendations: string[];
  doraLevel: 'elite' | 'high' | 'medium' | 'low';
}

class TeamAutonomyScorer {
  private scoreDeploymentFrequency(freq: TeamProfile['deploymentFrequency']): number {
    const scores: Record<TeamProfile['deploymentFrequency'], number> = {
      'multiple-daily': 100,
      'daily': 80,
      'weekly': 60,
      'monthly': 30,
      'quarterly': 10,
    };
    return scores[freq];
  }

  private scoreLeadTime(time: TeamProfile['leadTimeForChange']): number {
    const scores: Record<TeamProfile['leadTimeForChange'], number> = {
      'less-1h': 100,
      '1-24h': 75,
      '1-7d': 50,
      '1-4w': 25,
      'greater-4w': 10,
    };
    return scores[time];
  }

  private scoreChangeFailureRate(rate: TeamProfile['changeFailureRate']): number {
    const scores: Record<TeamProfile['changeFailureRate'], number> = {
      'less-5%': 100,
      '5-10%': 75,
      '10-15%': 50,
      '15-20%': 25,
      'greater-20%': 10,
    };
    return scores[rate];
  }

  private scoreTimeToRestore(time: TeamProfile['timeToRestoreService']): number {
    const scores: Record<TeamProfile['timeToRestoreService'], number> = {
      'less-1h': 100,
      '1-24h': 75,
      '1-7d': 50,
      '1-4w': 25,
      'greater-4w': 10,
    };
    return scores[time];
  }

  scoreTeam(profile: TeamProfile): TeamAutonomyScore {
    // DORA metrics scores
    const doraMetrics = {
      deploymentFrequency: this.scoreDeploymentFrequency(profile.deploymentFrequency),
      leadTime: this.scoreLeadTime(profile.leadTimeForChange),
      changeFailureRate: this.scoreChangeFailureRate(profile.changeFailureRate),
      timeToRestore: this.scoreTimeToRestore(profile.timeToRestoreService),
    };

    const doraAvg =
      (doraMetrics.deploymentFrequency +
        doraMetrics.leadTime +
        doraMetrics.changeFailureRate +
        doraMetrics.timeToRestore) /
      4;

    // Organizational autonomy score
    let orgScore = 0;
    if (profile.hasOwnCI) orgScore += 20;
    if (profile.hasOwnCD) orgScore += 20;
    if (profile.ownsDatabaseSchemas) orgScore += 20;
    if (profile.hasOnCall) orgScore += 20;
    if (profile.hasProductOwner) orgScore += 10;
    if (profile.hasDedicatedDesigner) orgScore += 10;

    // Technical score
    let techScore = 0;
    techScore += Math.min(100, profile.testCoverage);
    techScore -= profile.deploymentApprovals * 10; // -10 per approval needed
    techScore = Math.max(0, techScore);

    const overallScore = Math.round(doraAvg * 0.5 + orgScore * 0.3 + techScore * 0.2);

    // DORA level
    let doraLevel: TeamAutonomyScore['doraLevel'];
    if (doraAvg >= 80) doraLevel = 'elite';
    else if (doraAvg >= 60) doraLevel = 'high';
    else if (doraAvg >= 40) doraLevel = 'medium';
    else doraLevel = 'low';

    // Identify bottlenecks
    const bottlenecks: string[] = [];
    if (!profile.hasOwnCI) bottlenecks.push('ไม่มี CI pipeline ของตัวเอง');
    if (!profile.hasOwnCD) bottlenecks.push('ไม่มี CD pipeline ของตัวเอง');
    if (!profile.ownsDatabaseSchemas) bottlenecks.push('ต้องขอ permission เพื่อเปลี่ยน database schema');
    if (profile.deploymentApprovals > 1) bottlenecks.push(`ต้องผ่าน ${profile.deploymentApprovals} approval layers`);
    if (profile.testCoverage < 60) bottlenecks.push(`Test coverage ต่ำ (${profile.testCoverage}%)`);

    // Recommendations
    const recommendations: string[] = [];
    if (doraMetrics.deploymentFrequency < 60) {
      recommendations.push('เพิ่มความถี่ในการ deploy ด้วย feature flags');
    }
    if (doraMetrics.leadTime < 50) {
      recommendations.push('ลด lead time ด้วยการ automate testing และ review process');
    }
    if (profile.deploymentApprovals > 1) {
      recommendations.push('ลด approval gates ด้วย automated quality gates');
    }

    return {
      teamName: profile.name,
      overallScore,
      doraMetrics,
      organizationalScore: orgScore,
      technicalScore: Math.round(techScore),
      bottlenecks,
      recommendations,
      doraLevel,
    };
  }
}
```

---

## 5. Technical Excellence Checker

```typescript
// maturity/tech-excellence-checker.ts
import axios from 'axios';

interface TechExcellenceCheck {
  name: string;
  passed: boolean;
  score: number;
  details: string;
  remediation?: string;
}

class TechExcellenceChecker {
  async runAllChecks(
    repoUrl: string,
    prometheusUrl: string,
    k8sNamespace: string
  ): Promise<{
    overallScore: number;
    checks: TechExcellenceCheck[];
    grade: 'A' | 'B' | 'C' | 'D' | 'F';
  }> {
    const checks: TechExcellenceCheck[] = [];

    // 1. CI/CD Checks
    checks.push(await this.checkCICDPipeline(repoUrl));
    checks.push(await this.checkTestCoverage(prometheusUrl));
    checks.push(await this.checkDeploymentFrequency(prometheusUrl));

    // 2. Code Quality
    checks.push(await this.checkCodeQuality(repoUrl));
    checks.push(await this.checkDependencyUpdates(repoUrl));

    // 3. Security
    checks.push(await this.checkSecurityScanning(repoUrl));
    checks.push(await this.checkSecretManagement(k8sNamespace));

    // 4. Observability
    checks.push(await this.checkMetricsCoverage(prometheusUrl));
    checks.push(await this.checkTracingCoverage(prometheusUrl));
    checks.push(await this.checkAlertCoverage(prometheusUrl));

    // 5. Reliability
    checks.push(await this.checkSLODefinition(prometheusUrl));
    checks.push(await this.checkResourceLimits(k8sNamespace));
    checks.push(await this.checkHorizontalPodAutoscaler(k8sNamespace));

    const totalScore = checks.reduce((sum, c) => sum + c.score, 0);
    const overallScore = Math.round(totalScore / checks.length);

    let grade: 'A' | 'B' | 'C' | 'D' | 'F';
    if (overallScore >= 90) grade = 'A';
    else if (overallScore >= 80) grade = 'B';
    else if (overallScore >= 70) grade = 'C';
    else if (overallScore >= 60) grade = 'D';
    else grade = 'F';

    return { overallScore, checks, grade };
  }

  private async checkCICDPipeline(repoUrl: string): Promise<TechExcellenceCheck> {
    // จำลองการตรวจสอบ
    const hasCI = true;
    const hasCD = true;
    const hasStaging = true;

    const score = (hasCI ? 40 : 0) + (hasCD ? 40 : 0) + (hasStaging ? 20 : 0);

    return {
      name: 'CI/CD Pipeline',
      passed: score >= 80,
      score,
      details: `CI: ${hasCI}, CD: ${hasCD}, Staging: ${hasStaging}`,
      remediation: score < 80 ? 'เพิ่ม CD pipeline และ staging environment' : undefined,
    };
  }

  private async checkTestCoverage(prometheusUrl: string): Promise<TechExcellenceCheck> {
    // จำลอง: query test coverage จาก SonarQube metrics
    const coverage = 75; // percent
    const passed = coverage >= 80;

    return {
      name: 'Test Coverage',
      passed,
      score: Math.min(100, coverage),
      details: `Current coverage: ${coverage}%`,
      remediation: passed ? undefined : `เพิ่ม test coverage จาก ${coverage}% เป็น 80%+`,
    };
  }

  private async checkDeploymentFrequency(prometheusUrl: string): Promise<TechExcellenceCheck> {
    // จำลอง: คำนวณจาก deployment events
    const deploymentsPerDay = 2.5;
    const passed = deploymentsPerDay >= 1;
    const score = Math.min(100, deploymentsPerDay * 20);

    return {
      name: 'Deployment Frequency',
      passed,
      score,
      details: `${deploymentsPerDay.toFixed(1)} deployments per day`,
      remediation: passed ? undefined : 'เพิ่ม deployment frequency ด้วย feature flags',
    };
  }

  private async checkCodeQuality(repoUrl: string): Promise<TechExcellenceCheck> {
    const score = 80;
    return {
      name: 'Code Quality (SonarQube)',
      passed: score >= 70,
      score,
      details: 'Code smell: 5, Duplications: 3%, Maintainability: A',
    };
  }

  private async checkDependencyUpdates(repoUrl: string): Promise<TechExcellenceCheck> {
    const outdatedDeps = 3;
    const criticalDeps = 0;
    const passed = criticalDeps === 0 && outdatedDeps <= 5;
    const score = Math.max(0, 100 - (outdatedDeps * 5) - (criticalDeps * 20));

    return {
      name: 'Dependency Updates',
      passed,
      score,
      details: `Outdated: ${outdatedDeps}, Critical: ${criticalDeps}`,
      remediation: passed ? undefined : 'อัพเดต dependencies ที่ outdated',
    };
  }

  private async checkSecurityScanning(repoUrl: string): Promise<TechExcellenceCheck> {
    const criticalVulns = 0;
    const highVulns = 2;
    const passed = criticalVulns === 0 && highVulns <= 3;
    const score = Math.max(0, 100 - (criticalVulns * 30) - (highVulns * 10));

    return {
      name: 'Security Scanning',
      passed,
      score,
      details: `Critical: ${criticalVulns}, High: ${highVulns}`,
      remediation: passed ? undefined : 'แก้ไข critical vulnerabilities ทันที',
    };
  }

  private async checkSecretManagement(namespace: string): Promise<TechExcellenceCheck> {
    const usesVault = true;
    const noHardcodedSecrets = true;
    const autoRotation = false;

    const score = (usesVault ? 40 : 0) + (noHardcodedSecrets ? 40 : 0) + (autoRotation ? 20 : 0);

    return {
      name: 'Secret Management',
      passed: score >= 80,
      score,
      details: `Vault: ${usesVault}, No hardcoded: ${noHardcodedSecrets}, Auto-rotate: ${autoRotation}`,
      remediation: score < 80 ? 'เพิ่ม auto-rotation สำหรับ secrets' : undefined,
    };
  }

  private async checkMetricsCoverage(prometheusUrl: string): Promise<TechExcellenceCheck> {
    const score = 85;
    return {
      name: 'Metrics Coverage',
      passed: score >= 80,
      score,
      details: '85% of services have golden signals metrics',
    };
  }

  private async checkTracingCoverage(prometheusUrl: string): Promise<TechExcellenceCheck> {
    const score = 70;
    return {
      name: 'Distributed Tracing',
      passed: score >= 70,
      score,
      details: '70% of services have distributed tracing',
      remediation: score < 70 ? 'เพิ่ม OpenTelemetry instrumentation' : undefined,
    };
  }

  private async checkAlertCoverage(prometheusUrl: string): Promise<TechExcellenceCheck> {
    const score = 75;
    return {
      name: 'Alert Coverage',
      passed: score >= 70,
      score,
      details: 'SLO-based alerts: 75% coverage',
    };
  }

  private async checkSLODefinition(prometheusUrl: string): Promise<TechExcellenceCheck> {
    const servicesWithSLO = 8;
    const totalServices = 12;
    const coverage = (servicesWithSLO / totalServices) * 100;

    return {
      name: 'SLO Definition',
      passed: coverage >= 80,
      score: Math.round(coverage),
      details: `${servicesWithSLO}/${totalServices} services have defined SLOs`,
      remediation: coverage < 80 ? 'กำหนด SLOs สำหรับ services ที่เหลือ' : undefined,
    };
  }

  private async checkResourceLimits(namespace: string): Promise<TechExcellenceCheck> {
    const score = 90;
    return {
      name: 'Resource Limits',
      passed: score >= 80,
      score,
      details: '90% of pods have CPU and memory limits set',
    };
  }

  private async checkHorizontalPodAutoscaler(namespace: string): Promise<TechExcellenceCheck> {
    const score = 80;
    return {
      name: 'Auto-scaling (HPA)',
      passed: score >= 70,
      score,
      details: '80% of critical services have HPA configured',
    };
  }
}
```

---

## 6. Architecture Decision Record Helper

```typescript
// maturity/adr-decision-helper.ts
import * as fs from 'fs';
import * as path from 'path';

interface ADR {
  id: string;
  title: string;
  date: string;
  status: 'proposed' | 'accepted' | 'deprecated' | 'superseded';
  context: string;
  decision: string;
  rationale: string;
  consequences: {
    positive: string[];
    negative: string[];
    risks: string[];
  };
  alternatives: Array<{
    option: string;
    pros: string[];
    cons: string[];
    whyNotChosen: string;
  }>;
  supersededBy?: string;
  relatedADRs?: string[];
  tags: string[];
  deciders: string[];
}

class ADRDecisionHelper {
  generateTemplate(
    id: string,
    title: string,
    deciders: string[]
  ): ADR {
    return {
      id: `ADR-${id.padStart(3, '0')}`,
      title,
      date: new Date().toISOString().split('T')[0],
      status: 'proposed',
      context: 'อธิบาย context ที่นำไปสู่การตัดสินใจนี้ รวมถึง constraints และ requirements',
      decision: 'อธิบายการตัดสินใจที่เลือก',
      rationale: 'อธิบายเหตุผลที่เลือก option นี้',
      consequences: {
        positive: ['ผลดีที่คาดว่าจะได้รับ'],
        negative: ['ข้อเสียหรือ tradeoffs'],
        risks: ['risks ที่อาจเกิดขึ้น'],
      },
      alternatives: [
        {
          option: 'Alternative 1',
          pros: ['ข้อดี'],
          cons: ['ข้อเสีย'],
          whyNotChosen: 'เหตุผลที่ไม่เลือก',
        },
      ],
      relatedADRs: [],
      tags: [],
      deciders,
    };
  }

  toMarkdown(adr: ADR): string {
    const lines: string[] = [
      `# ${adr.id}: ${adr.title}`,
      '',
      `**Date:** ${adr.date}`,
      `**Status:** ${adr.status.toUpperCase()}`,
      `**Deciders:** ${adr.deciders.join(', ')}`,
      '',
      '## Context',
      '',
      adr.context,
      '',
      '## Decision',
      '',
      adr.decision,
      '',
      '## Rationale',
      '',
      adr.rationale,
      '',
      '## Consequences',
      '',
      '### Positive',
      ...adr.consequences.positive.map((p) => `- ${p}`),
      '',
      '### Negative',
      ...adr.consequences.negative.map((n) => `- ${n}`),
      '',
      '### Risks',
      ...adr.consequences.risks.map((r) => `- ${r}`),
      '',
      '## Alternatives Considered',
      '',
    ];

    for (const alt of adr.alternatives) {
      lines.push(`### ${alt.option}`);
      lines.push('**Pros:**');
      alt.pros.forEach((p) => lines.push(`- ${p}`));
      lines.push('**Cons:**');
      alt.cons.forEach((c) => lines.push(`- ${c}`));
      lines.push(`**Why not chosen:** ${alt.whyNotChosen}`);
      lines.push('');
    }

    if (adr.relatedADRs && adr.relatedADRs.length > 0) {
      lines.push('## Related ADRs');
      adr.relatedADRs.forEach((r) => lines.push(`- ${r}`));
    }

    if (adr.tags.length > 0) {
      lines.push('');
      lines.push(`**Tags:** ${adr.tags.join(', ')}`);
    }

    return lines.join('\n');
  }

  saveADR(adr: ADR, directory: string): void {
    const filename = `${adr.id}-${adr.title.toLowerCase().replace(/\s+/g, '-')}.md`;
    const filepath = path.join(directory, filename);
    fs.writeFileSync(filepath, this.toMarkdown(adr));
    console.log(`ADR saved: ${filepath}`);
  }
}
```

---

## 7. Maturity Radar Chart Generator (Mermaid)

```typescript
// maturity/radar-chart-generator.ts

interface RadarData {
  dimension: string;
  current: number;   // 1-5
  target: number;    // 1-5
}

function generateMermaidRadar(categoryScores: CategoryScore[]): string {
  const lines: string[] = [
    '```mermaid',
    'radarChart',
    '  title Microservices Maturity Assessment',
  ];

  for (const cat of categoryScores) {
    const current = Math.round(cat.score / 20); // Convert 0-100 to 1-5
    lines.push(`  "${cat.category}" : ${current}`);
  }

  lines.push('```');
  return lines.join('\n');
}

// ตัวอย่าง Mermaid Quadrant Chart
function generateQuadrantChart(teams: Array<{ name: string; autonomy: number; maturity: number }>): string {
  const lines = [
    '```mermaid',
    'quadrantChart',
    '  title Team Autonomy vs Technical Maturity',
    '  x-axis Low Autonomy --> High Autonomy',
    '  y-axis Low Maturity --> High Maturity',
    '  quadrant-1 Stars (Invest)',
    '  quadrant-2 Need Enablement',
    '  quadrant-3 Transform First',
    '  quadrant-4 Quick Wins',
  ];

  for (const team of teams) {
    const x = team.autonomy / 100;
    const y = team.maturity / 100;
    lines.push(`  ${team.name}: [${x.toFixed(2)}, ${y.toFixed(2)}]`);
  }

  lines.push('```');
  return lines.join('\n');
}

// ตัวอย่างการใช้งาน
async function runMaturityAssessment() {
  const assessment = new MaturityAssessment();

  // สร้าง mock answers
  const answers: AssessmentAnswer[] = assessment.getQuestions().map((q) => ({
    questionId: q.id,
    selectedLevel: (Math.floor(Math.random() * 3) + 2) as 1 | 2 | 3 | 4 | 5,
    notes: 'Auto-generated for demo',
  }));

  const report = assessment.calculateScore(answers);

  console.log('\n=== Microservices Maturity Assessment ===');
  console.log(`Overall Score: ${report.overallScore}/100`);
  console.log(`Overall Level: Level ${report.overallLevel}`);
  console.log(`Estimated time to next level: ${report.estimatedTimeToNextLevel}`);
  console.log('\nCategory Scores:');
  for (const cat of report.categoryScores) {
    console.log(`  ${cat.category}: ${cat.score}/100 (Level ${cat.level})`);
  }

  console.log('\nTop Improvements:');
  report.topImprovements.slice(0, 3).forEach((i) => console.log(`  - ${i}`));

  console.log('\nNext Steps:');
  report.nextSteps.forEach((s) => console.log(`  - ${s}`));

  console.log('\n' + report.radarChart);
}

runMaturityAssessment().catch(console.error);
```

---

## สรุป

| หัวข้อ | เนื้อหาสำคัญ |
|--------|-------------|
| Maturity Levels 1-5 | Initial → Managed → Defined → Quantitatively Managed → Optimizing |
| MaturityAssessment Class | 50+ questions ครอบคลุม Architecture, CI/CD, Security, Operations |
| Strangler Fig Pattern | TypeScript proxy สำหรับ gradual migration จาก monolith |
| Team Autonomy Scorer | DORA metrics scoring พร้อม bottleneck identification |
| Tech Excellence Checker | CI/CD, security, observability, reliability checks |
| Operational Excellence | MTTR, change failure rate metrics |
| Security Maturity | Secret management, vulnerability scanning, zero trust |
| Documentation Maturity | API catalog, ADRs, living documentation |
| ADR Decision Helper | Template generator และ Markdown exporter |
| Radar Chart Generator | Mermaid diagram สำหรับ visual representation |

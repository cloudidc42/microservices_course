# Part 90: Microservices Maturity Model

## บทนำ

Microservices Maturity Model เป็น framework สำหรับประเมินและพัฒนาความสามารถขององค์กรในการสร้างและดำเนินงาน Microservices บทนี้เป็นบทสรุปของ course ทั้งหมด โดยเชื่อมโยงทุกหัวข้อที่เรียนมาเข้าด้วยกันผ่าน maturity model ที่สามารถนำไปใช้ประเมินองค์กรได้จริง

---

## 1. Microservices Maturity Levels

### 1.1 Overview ของ 5 ระดับ

```typescript
// src/maturity/levels.ts

export enum MaturityLevel {
  LEVEL_1 = 1, // Initial - เพิ่งเริ่มต้น
  LEVEL_2 = 2, // Managed - มีกระบวนการ
  LEVEL_3 = 3, // Defined - มีมาตรฐาน
  LEVEL_4 = 4, // Quantitatively Managed - วัดผลได้
  LEVEL_5 = 5, // Optimizing - ปรับปรุงต่อเนื่อง
}

export const MATURITY_LEVEL_DESCRIPTIONS = {
  [MaturityLevel.LEVEL_1]: {
    name: 'Initial',
    description: 'เพิ่งเริ่ม migrate จาก Monolith หรือสร้าง Microservices แรก',
    characteristics: [
      'Services แยกออกจาก monolith แบบ ad-hoc',
      'ไม่มีมาตรฐาน API design',
      'Deploy ด้วย manual processes',
      'Monitoring พื้นฐานหรือไม่มีเลย',
      'ทีมเดียวดูแลทุก service',
      'ไม่มี CI/CD pipeline ที่สมบูรณ์',
    ],
    challenges: [
      'ขาดความรู้ distributed systems',
      'Network latency ที่ไม่เคยเจอ',
      'Data consistency ยาก',
      'Testing ซับซ้อนขึ้น',
    ],
    timeline: '0-6 months',
  },
  [MaturityLevel.LEVEL_2]: {
    name: 'Managed',
    description: 'มีกระบวนการพื้นฐาน แต่ยังไม่ consistent ทุก service',
    characteristics: [
      'CI/CD pipeline สำหรับบาง services',
      'มี basic monitoring (metrics, logs)',
      'API design เริ่มมีมาตรฐาน',
      'Docker containers บน Kubernetes',
      'Service discovery พื้นฐาน',
      'การจัดการ secrets ดีขึ้น',
    ],
    challenges: [
      'Inconsistent practices ระหว่าง teams',
      'Manual deployment ยังมีอยู่บ้าง',
      'Distributed tracing ยังไม่มี',
      'Service mesh ยังไม่ได้ใช้',
    ],
    timeline: '6-18 months',
  },
  [MaturityLevel.LEVEL_3]: {
    name: 'Defined',
    description: 'มีมาตรฐานที่ชัดเจนและ apply กับทุก service',
    characteristics: [
      'Golden path สำหรับ service creation',
      'Distributed tracing (Jaeger/Zipkin)',
      'Service mesh (Istio/Linkerd)',
      'Centralized configuration',
      'Automated testing pyramid',
      'Feature flags',
      'Blue/Green deployments',
    ],
    challenges: [
      'Service mesh complexity',
      'Too many services to manage',
      'Data consistency at scale',
    ],
    timeline: '18-36 months',
  },
  [MaturityLevel.LEVEL_4]: {
    name: 'Quantitatively Managed',
    description: 'วัดผลได้ทุกอย่าง มี SLOs สำหรับทุก service',
    characteristics: [
      'SLOs defined สำหรับทุก service',
      'Error budget management',
      'Automated capacity planning',
      'Chaos engineering regular practice',
      'Self-service infrastructure',
      'Developer productivity metrics',
      'Cost attribution per service',
    ],
    challenges: [
      'Data-driven decision fatigue',
      'Alert fatigue ถ้า threshold ผิด',
    ],
    timeline: '3-5 years',
  },
  [MaturityLevel.LEVEL_5]: {
    name: 'Optimizing',
    description: 'ปรับปรุงต่อเนื่อง มี culture ที่แข็งแกร่ง',
    characteristics: [
      'Continuous experimentation (A/B testing at infra level)',
      'ML-based anomaly detection',
      'Auto-remediation of common issues',
      'Progressive delivery',
      'Internal developer platform (IDP)',
      'Blameless culture deeply embedded',
      'Contributing back to open source',
    ],
    challenges: [
      'Maintaining culture at scale',
      'Preventing over-engineering',
      'Keeping simplicity despite scale',
    ],
    timeline: '5+ years',
  },
};
```

---

## 2. Assessment Framework

### 2.1 Maturity Assessment Tool

```typescript
// src/maturity/assessment.ts

interface AssessmentDimension {
  name: string;
  description: string;
  questions: AssessmentQuestion[];
  weight: number;  // 1-5 importance weight
}

interface AssessmentQuestion {
  id: string;
  question: string;
  options: Array<{
    score: number;  // 1-5
    label: string;
    description: string;
  }>;
}

interface AssessmentResult {
  overallLevel: MaturityLevel;
  overallScore: number;        // 0-100
  dimensionScores: Record<string, number>;
  strengths: string[];
  gaps: string[];
  topPriorities: string[];
  roadmap: RoadmapItem[];
}

export const ASSESSMENT_DIMENSIONS: AssessmentDimension[] = [
  {
    name: 'Service Design',
    description: 'How well are services designed and organized',
    weight: 5,
    questions: [
      {
        id: 'sd-1',
        question: 'How are service boundaries defined?',
        options: [
          { score: 1, label: 'Ad-hoc', description: 'No clear strategy, split by technical concerns' },
          { score: 2, label: 'By team', description: 'Split by team ownership' },
          { score: 3, label: 'By domain', description: 'DDD with bounded contexts' },
          { score: 4, label: 'Optimized', description: 'Regularly reviewed and refined' },
          { score: 5, label: 'Autonomous', description: 'Services are independently deployable with clear contracts' },
        ],
      },
      {
        id: 'sd-2',
        question: 'How are APIs designed and documented?',
        options: [
          { score: 1, label: 'No standard', description: 'Each service has different patterns' },
          { score: 2, label: 'Basic REST', description: 'HTTP endpoints, some documentation' },
          { score: 3, label: 'OpenAPI', description: 'OpenAPI 3.0 spec, versioning' },
          { score: 4, label: 'Contract-first', description: 'API design before implementation, consumer-driven contracts' },
          { score: 5, label: 'Self-service', description: 'API portal, automatic SDK generation, deprecation management' },
        ],
      },
    ],
  },
  {
    name: 'Deployment & Operations',
    description: 'CI/CD, infrastructure, and deployment practices',
    weight: 5,
    questions: [
      {
        id: 'do-1',
        question: 'What is your deployment process?',
        options: [
          { score: 1, label: 'Manual', description: 'Manual SSH and copy files' },
          { score: 2, label: 'Scripts', description: 'Shell scripts, some automation' },
          { score: 3, label: 'CI/CD Pipeline', description: 'Automated builds, tests, deployments' },
          { score: 4, label: 'GitOps', description: 'Git-based, automated rollbacks' },
          { score: 5, label: 'Progressive Delivery', description: 'Canary, feature flags, automatic rollback on SLO breach' },
        ],
      },
      {
        id: 'do-2',
        question: 'How is infrastructure managed?',
        options: [
          { score: 1, label: 'Manual', description: 'Click-ops in cloud console' },
          { score: 2, label: 'Scripts', description: 'Bash scripts, some automation' },
          { score: 3, label: 'IaC', description: 'Terraform, Helm charts' },
          { score: 4, label: 'Self-service', description: 'Developers provision infra via templates' },
          { score: 5, label: 'Platform', description: 'Internal Developer Platform, golden paths' },
        ],
      },
    ],
  },
  {
    name: 'Observability',
    description: 'Monitoring, logging, tracing, and alerting',
    weight: 4,
    questions: [
      {
        id: 'obs-1',
        question: 'What observability capabilities do you have?',
        options: [
          { score: 1, label: 'Basic logs', description: 'Server logs only' },
          { score: 2, label: 'Metrics + Logs', description: 'Prometheus, centralized logging' },
          { score: 3, label: 'Full observability', description: 'Metrics, logs, traces (three pillars)' },
          { score: 4, label: 'Correlated', description: 'Correlation IDs, unified view' },
          { score: 5, label: 'Proactive', description: 'ML-based anomaly detection, predictive alerts' },
        ],
      },
      {
        id: 'obs-2',
        question: 'How are SLOs/SLAs managed?',
        options: [
          { score: 1, label: 'None', description: 'No formal SLOs' },
          { score: 2, label: 'Uptime only', description: 'Basic uptime monitoring' },
          { score: 3, label: 'SLOs defined', description: 'SLOs with error budget' },
          { score: 4, label: 'SLO-driven', description: 'Alerts based on error budget burn rate' },
          { score: 5, label: 'Full lifecycle', description: 'SLOs inform product decisions, automated budget management' },
        ],
      },
    ],
  },
  {
    name: 'Security',
    description: 'Security practices and compliance',
    weight: 4,
    questions: [
      {
        id: 'sec-1',
        question: 'How is service-to-service authentication handled?',
        options: [
          { score: 1, label: 'None', description: 'No authentication between services' },
          { score: 2, label: 'API keys', description: 'Shared API keys' },
          { score: 3, label: 'JWT/mTLS', description: 'JWT tokens or mutual TLS' },
          { score: 4, label: 'Zero trust', description: 'Zero-trust network, SPIFFE/SPIRE' },
          { score: 5, label: 'Policy-driven', description: 'OPA policies, automated cert rotation' },
        ],
      },
    ],
  },
  {
    name: 'Team & Culture',
    description: 'Team structure, autonomy, and engineering culture',
    weight: 3,
    questions: [
      {
        id: 'team-1',
        question: 'How are teams structured?',
        options: [
          { score: 1, label: 'Functional', description: 'Dev team + Ops team, siloed' },
          { score: 2, label: 'DevOps', description: 'Mixed teams, basic DevOps' },
          { score: 3, label: 'Product teams', description: 'Cross-functional, own their services' },
          { score: 4, label: 'Platform + Stream', description: 'Platform team + stream-aligned teams' },
          { score: 5, label: 'Full autonomy', description: 'Teams own full lifecycle, minimal coordination needed' },
        ],
      },
    ],
  },
];

export class MaturityAssessor {
  assess(responses: Record<string, number>): AssessmentResult {
    const dimensionScores: Record<string, number> = {};
    
    for (const dimension of ASSESSMENT_DIMENSIONS) {
      const questionScores = dimension.questions
        .map(q => responses[q.id] || 1);
      
      const avgScore = questionScores.reduce((a, b) => a + b, 0) / questionScores.length;
      dimensionScores[dimension.name] = avgScore;
    }
    
    // Calculate weighted overall score
    const totalWeight = ASSESSMENT_DIMENSIONS.reduce((sum, d) => sum + d.weight, 0);
    const weightedScore = ASSESSMENT_DIMENSIONS.reduce((sum, d) => {
      return sum + (dimensionScores[d.name] * d.weight);
    }, 0) / totalWeight;
    
    const overallScore = Math.round((weightedScore / 5) * 100);
    
    // Determine level
    const overallLevel = weightedScore <= 1.5 ? MaturityLevel.LEVEL_1
      : weightedScore <= 2.5 ? MaturityLevel.LEVEL_2
      : weightedScore <= 3.5 ? MaturityLevel.LEVEL_3
      : weightedScore <= 4.5 ? MaturityLevel.LEVEL_4
      : MaturityLevel.LEVEL_5;
    
    // Find strengths and gaps
    const sortedDimensions = Object.entries(dimensionScores)
      .sort(([,a], [,b]) => b - a);
    
    const strengths = sortedDimensions.slice(0, 2).map(([name]) => name);
    const gaps = sortedDimensions.slice(-2).map(([name]) => name);
    
    return {
      overallLevel,
      overallScore,
      dimensionScores,
      strengths,
      gaps,
      topPriorities: this.generatePriorities(dimensionScores, overallLevel),
      roadmap: this.generateRoadmap(overallLevel, dimensionScores),
    };
  }

  private generatePriorities(
    scores: Record<string, number>,
    level: MaturityLevel
  ): string[] {
    const priorities: string[] = [];
    
    for (const [dimension, score] of Object.entries(scores)) {
      if (score < level) {
        priorities.push(`Improve ${dimension} from level ${Math.round(score)} to ${level}`);
      }
    }
    
    return priorities.slice(0, 5);
  }

  private generateRoadmap(
    currentLevel: MaturityLevel,
    scores: Record<string, number>
  ): RoadmapItem[] {
    const roadmap: RoadmapItem[] = [];
    
    if (currentLevel < MaturityLevel.LEVEL_3) {
      roadmap.push({
        phase: 'Phase 1: Foundation (0-3 months)',
        items: [
          'Implement CI/CD pipeline for all services',
          'Set up centralized logging (ELK Stack)',
          'Define API design standards',
          'Implement health checks and basic metrics',
          'Document service boundaries and ownership',
        ],
      });
    }
    
    if (currentLevel < MaturityLevel.LEVEL_4) {
      roadmap.push({
        phase: 'Phase 2: Standardization (3-12 months)',
        items: [
          'Implement distributed tracing',
          'Set up service mesh',
          'Define and track SLOs',
          'Implement feature flags',
          'Adopt GitOps',
          'Implement security scanning in pipeline',
        ],
      });
    }
    
    if (currentLevel < MaturityLevel.LEVEL_5) {
      roadmap.push({
        phase: 'Phase 3: Optimization (12-24 months)',
        items: [
          'Build Internal Developer Platform',
          'Implement chaos engineering',
          'Set up cost attribution',
          'Implement progressive delivery',
          'Establish blameless post-mortem culture',
          'ML-based anomaly detection',
        ],
      });
    }
    
    return roadmap;
  }
}

interface RoadmapItem {
  phase: string;
  items: string[];
}
```

---

## 3. Migration Path from Monolith

### 3.1 Strangler Fig Pattern Implementation

```typescript
// src/migration/strangler-fig.ts
import express, { Request, Response, NextFunction } from 'express';
import httpProxy from 'http-proxy';

interface RouteMigration {
  path: string;
  method: string;
  percentage: number;    // % ของ traffic ที่ไป new service
  newService: string;
  phase: 'planning' | 'testing' | 'migrating' | 'complete';
}

export class StranglerFigProxy {
  private proxy = httpProxy.createProxy();
  private migrations: RouteMigration[] = [];
  
  constructor(
    private legacyUrl: string,
    private newServicesMap: Record<string, string>
  ) {}

  registerMigration(migration: RouteMigration): void {
    this.migrations.push(migration);
    console.log(`Registered migration: ${migration.method} ${migration.path} -> ${migration.newService} (${migration.percentage}%)`);
  }

  // Middleware ที่ route traffic ระหว่าง legacy และ new service
  middleware() {
    return (req: Request, res: Response, next: NextFunction) => {
      const migration = this.findMigration(req.method, req.path);
      
      if (!migration) {
        // ไม่มี migration: ส่งไป legacy
        this.proxyToLegacy(req, res);
        return;
      }
      
      // Shadow mode: ส่งไปทั้งคู่ แต่ return จาก legacy
      if (migration.phase === 'testing') {
        this.shadowProxy(req, res, migration);
        return;
      }
      
      // Canary: route ตาม percentage
      const useNew = Math.random() * 100 < migration.percentage;
      
      if (useNew || migration.phase === 'complete') {
        this.proxyToNewService(req, res, migration.newService);
      } else {
        this.proxyToLegacy(req, res);
      }
    };
  }

  private findMigration(method: string, path: string): RouteMigration | null {
    return this.migrations.find(m => {
      const pathPattern = m.path.replace(/:[^/]+/g, '[^/]+');
      return m.method === method && new RegExp(`^${pathPattern}`).test(path);
    }) || null;
  }

  private proxyToLegacy(req: Request, res: Response): void {
    this.proxy.web(req, res, { target: this.legacyUrl });
  }

  private proxyToNewService(req: Request, res: Response, service: string): void {
    const target = this.newServicesMap[service];
    this.proxy.web(req, res, { target });
  }

  private async shadowProxy(
    req: Request, 
    res: Response, 
    migration: RouteMigration
  ): Promise<void> {
    // ส่งไปยัง new service แบบ async (shadow)
    this.sendShadowRequest(req, migration.newService)
      .then(shadowResponse => {
        this.compareResponses(req.path, null, shadowResponse);
      })
      .catch(err => {
        console.error('Shadow request failed:', err);
      });
    
    // Return จาก legacy
    this.proxyToLegacy(req, res);
  }

  private async sendShadowRequest(req: Request, service: string): Promise<any> {
    const target = this.newServicesMap[service];
    const response = await fetch(`${target}${req.path}`, {
      method: req.method,
      headers: req.headers as any,
      body: req.method !== 'GET' ? JSON.stringify(req.body) : undefined,
    });
    return response.json();
  }

  private compareResponses(path: string, legacy: any, newService: any): void {
    const diffs = this.diff(legacy, newService);
    if (diffs.length > 0) {
      console.warn({
        message: 'Shadow response diff detected',
        path,
        diffs,
      });
    }
  }

  private diff(a: any, b: any): string[] {
    // Simple diff implementation
    return [];
  }
}

// Migration phases
export const MIGRATION_PHASES = {
  1: {
    name: 'Identify',
    steps: [
      'Map all modules in monolith',
      'Identify bounded contexts using DDD',
      'Define service boundaries',
      'Map data ownership',
      'Identify shared databases',
    ],
  },
  2: {
    name: 'Prepare',
    steps: [
      'Set up API gateway/proxy (strangler fig)',
      'Add feature flags',
      'Set up CI/CD for new services',
      'Create monitoring baseline',
      'Train team on microservices patterns',
    ],
  },
  3: {
    name: 'Extract',
    steps: [
      'Start with least-coupled, highest-value service',
      'Implement anti-corruption layer',
      'Run shadow mode first (dual write)',
      'Validate data consistency',
      'Gradual traffic shifting (10% -> 50% -> 100%)',
    ],
  },
  4: {
    name: 'Decommission',
    steps: [
      'Route 100% traffic to new service',
      'Monitor for 2 weeks',
      'Remove legacy code',
      'Clean up database tables',
      'Update documentation',
    ],
  },
};
```

---

## 4. Team Autonomy Model

### 4.1 Team Topologies Implementation

```typescript
// src/teams/team-topology.ts

export type TeamType = 
  | 'stream_aligned'    // Product team ที่ own a stream of work
  | 'platform'          // Platform team ที่สร้าง tools สำหรับ stream teams
  | 'enabling'          // ช่วย stream teams adopt new practices
  | 'complicated_subsystem'; // ดูแล system ที่ซับซ้อนเป็นพิเศษ

export type InteractionMode = 
  | 'collaboration'     // ทำงานร่วมกัน (temporary)
  | 'x_as_a_service'   // ให้บริการ (arms-length)
  | 'facilitating';     // coaching/mentoring

interface TeamDefinition {
  id: string;
  name: string;
  type: TeamType;
  missionStatement: string;
  services: string[];
  ownedCapabilities: string[];
  dependencies: Array<{
    team: string;
    mode: InteractionMode;
    description: string;
  }>;
  metrics: TeamMetrics;
}

interface TeamMetrics {
  deploymentFrequency: string;   // 'multiple/day', 'daily', 'weekly'
  changeLeadTime: string;        // 'hours', 'days', 'weeks'
  changeFailureRate: string;     // '<5%', '5-10%', '>10%'
  mttrMinutes: number;
  cognitiveLoad: 'low' | 'medium' | 'high';
}

const TEAM_DEFINITIONS: TeamDefinition[] = [
  {
    id: 'team-platform',
    name: 'Platform Team',
    type: 'platform',
    missionStatement: 'Enable stream teams to deliver value quickly and safely',
    services: ['kubernetes', 'ci-cd', 'monitoring', 'security'],
    ownedCapabilities: [
      'Kubernetes cluster management',
      'CI/CD pipelines as a service',
      'Observability platform',
      'Security scanning',
      'Internal developer portal',
    ],
    dependencies: [],
    metrics: {
      deploymentFrequency: 'multiple/day',
      changeLeadTime: 'hours',
      changeFailureRate: '<5%',
      mttrMinutes: 30,
      cognitiveLoad: 'high',
    },
  },
  {
    id: 'team-user',
    name: 'User Team',
    type: 'stream_aligned',
    missionStatement: 'Own the user registration, authentication, and profile experience',
    services: ['user-service', 'auth-service'],
    ownedCapabilities: [
      'User registration and onboarding',
      'Authentication and authorization',
      'User profile management',
      'User preferences',
    ],
    dependencies: [
      {
        team: 'team-platform',
        mode: 'x_as_a_service',
        description: 'Use CI/CD, monitoring, k8s',
      },
    ],
    metrics: {
      deploymentFrequency: 'multiple/day',
      changeLeadTime: 'hours',
      changeFailureRate: '<5%',
      mttrMinutes: 60,
      cognitiveLoad: 'low',
    },
  },
  {
    id: 'team-payment',
    name: 'Payment Team',
    type: 'complicated_subsystem',
    missionStatement: 'Provide secure, reliable payment processing',
    services: ['payment-service', 'fraud-detection-service'],
    ownedCapabilities: [
      'Payment processing',
      'Fraud detection',
      'PCI-DSS compliance',
      'Refund processing',
    ],
    dependencies: [
      {
        team: 'team-platform',
        mode: 'x_as_a_service',
        description: 'Use CI/CD, monitoring',
      },
    ],
    metrics: {
      deploymentFrequency: 'weekly',
      changeLeadTime: 'days',
      changeFailureRate: '<1%',
      mttrMinutes: 15,
      cognitiveLoad: 'high',
    },
  },
];
```

---

## 5. Technical, Operational, Security, and Documentation Maturity

### 5.1 Comprehensive Maturity Scorecard

```typescript
// src/maturity/scorecard.ts

interface MaturityScorecard {
  technical: TechnicalMaturity;
  operational: OperationalMaturity;
  security: SecurityMaturity;
  documentation: DocumentationMaturity;
  testing: TestingMaturity;
  overallScore: number;
  level: MaturityLevel;
}

interface TechnicalMaturity {
  score: number;
  indicators: {
    serviceDesign: number;        // 1-5
    apiDesign: number;
    dataManagement: number;
    messaging: number;
    resilience: number;
    performance: number;
  };
  strengths: string[];
  improvements: string[];
}

interface OperationalMaturity {
  score: number;
  indicators: {
    ciCd: number;
    monitoring: number;
    logging: number;
    tracing: number;
    alerting: number;
    capacityPlanning: number;
    incidentManagement: number;
  };
}

interface SecurityMaturity {
  score: number;
  indicators: {
    authentication: number;
    authorization: number;
    dataEncryption: number;
    secretManagement: number;
    vulnerabilityScanning: number;
    complianceGDPR: number;
  };
}

interface DocumentationMaturity {
  score: number;
  indicators: {
    apiDocumentation: number;
    runbooks: number;
    architecture: number;
    onboarding: number;
    postMortems: number;
  };
}

interface TestingMaturity {
  score: number;
  indicators: {
    unitTests: number;
    integrationTests: number;
    contractTests: number;
    e2eTests: number;
    performanceTests: number;
    chaosEngineering: number;
  };
}

export class MaturityScorecardGenerator {
  generate(answers: Record<string, number>): MaturityScorecard {
    const technical = this.scoreTechnical(answers);
    const operational = this.scoreOperational(answers);
    const security = this.scoreSecurity(answers);
    const documentation = this.scoreDocumentation(answers);
    const testing = this.scoreTesting(answers);
    
    const overallScore = (
      technical.score * 0.30 +
      operational.score * 0.25 +
      security.score * 0.20 +
      documentation.score * 0.10 +
      testing.score * 0.15
    );
    
    const level = this.determineLevel(overallScore);
    
    return {
      technical,
      operational,
      security,
      documentation,
      testing,
      overallScore: Math.round(overallScore * 100) / 100,
      level,
    };
  }

  private scoreTechnical(answers: Record<string, number>): TechnicalMaturity {
    const indicators = {
      serviceDesign: answers['tech_service_design'] || 1,
      apiDesign: answers['tech_api_design'] || 1,
      dataManagement: answers['tech_data_mgmt'] || 1,
      messaging: answers['tech_messaging'] || 1,
      resilience: answers['tech_resilience'] || 1,
      performance: answers['tech_performance'] || 1,
    };
    
    const score = Object.values(indicators).reduce((a, b) => a + b) / 
                 Object.values(indicators).length;
    
    const strengths: string[] = [];
    const improvements: string[] = [];
    
    for (const [key, value] of Object.entries(indicators)) {
      if (value >= 4) strengths.push(this.getIndicatorName(key));
      if (value <= 2) improvements.push(this.getIndicatorName(key));
    }
    
    return { score, indicators, strengths, improvements };
  }

  private scoreOperational(answers: Record<string, number>): OperationalMaturity {
    const indicators = {
      ciCd: answers['ops_cicd'] || 1,
      monitoring: answers['ops_monitoring'] || 1,
      logging: answers['ops_logging'] || 1,
      tracing: answers['ops_tracing'] || 1,
      alerting: answers['ops_alerting'] || 1,
      capacityPlanning: answers['ops_capacity'] || 1,
      incidentManagement: answers['ops_incident'] || 1,
    };
    
    const score = Object.values(indicators).reduce((a, b) => a + b) / 
                 Object.values(indicators).length;
    
    return { score, indicators };
  }

  private scoreSecurity(answers: Record<string, number>): SecurityMaturity {
    const indicators = {
      authentication: answers['sec_authn'] || 1,
      authorization: answers['sec_authz'] || 1,
      dataEncryption: answers['sec_encryption'] || 1,
      secretManagement: answers['sec_secrets'] || 1,
      vulnerabilityScanning: answers['sec_scanning'] || 1,
      complianceGDPR: answers['sec_gdpr'] || 1,
    };
    
    const score = Object.values(indicators).reduce((a, b) => a + b) / 
                 Object.values(indicators).length;
    
    return { score, indicators };
  }

  private scoreDocumentation(answers: Record<string, number>): DocumentationMaturity {
    const indicators = {
      apiDocumentation: answers['doc_api'] || 1,
      runbooks: answers['doc_runbooks'] || 1,
      architecture: answers['doc_architecture'] || 1,
      onboarding: answers['doc_onboarding'] || 1,
      postMortems: answers['doc_postmortems'] || 1,
    };
    
    const score = Object.values(indicators).reduce((a, b) => a + b) / 
                 Object.values(indicators).length;
    
    return { score, indicators };
  }

  private scoreTesting(answers: Record<string, number>): TestingMaturity {
    const indicators = {
      unitTests: answers['test_unit'] || 1,
      integrationTests: answers['test_integration'] || 1,
      contractTests: answers['test_contract'] || 1,
      e2eTests: answers['test_e2e'] || 1,
      performanceTests: answers['test_performance'] || 1,
      chaosEngineering: answers['test_chaos'] || 1,
    };
    
    const score = Object.values(indicators).reduce((a, b) => a + b) / 
                 Object.values(indicators).length;
    
    return { score, indicators };
  }

  private determineLevel(score: number): MaturityLevel {
    if (score <= 1.5) return MaturityLevel.LEVEL_1;
    if (score <= 2.5) return MaturityLevel.LEVEL_2;
    if (score <= 3.5) return MaturityLevel.LEVEL_3;
    if (score <= 4.5) return MaturityLevel.LEVEL_4;
    return MaturityLevel.LEVEL_5;
  }

  private getIndicatorName(key: string): string {
    const names: Record<string, string> = {
      serviceDesign: 'Service Design',
      apiDesign: 'API Design',
      dataManagement: 'Data Management',
      messaging: 'Messaging',
      resilience: 'Resilience Patterns',
      performance: 'Performance Optimization',
    };
    return names[key] || key;
  }
}
```

---

## 6. Architecture Decision Framework

### 6.1 Architecture Decision Record (ADR)

```typescript
// src/architecture/adr.ts

interface ArchitectureDecisionRecord {
  id: string;                   // ADR-001, ADR-002, ...
  title: string;
  status: 'proposed' | 'accepted' | 'deprecated' | 'superseded';
  date: Date;
  deciders: string[];
  
  context: string;              // สถานการณ์ที่นำมาสู่การตัดสินใจ
  decision: string;             // การตัดสินใจที่ทำ
  rationale: string;            // เหตุผลของการตัดสินใจ
  
  alternatives: Array<{
    option: string;
    pros: string[];
    cons: string[];
    rejected: boolean;
    rejectionReason?: string;
  }>;
  
  consequences: {
    positive: string[];
    negative: string[];
    risks: string[];
  };
  
  relatedADRs?: string[];
  supersededBy?: string;
}

// ตัวอย่าง ADR
export const EXAMPLE_ADRS: ArchitectureDecisionRecord[] = [
  {
    id: 'ADR-001',
    title: 'Use Event-Driven Architecture for Inter-Service Communication',
    status: 'accepted',
    date: new Date('2024-01-15'),
    deciders: ['Tech Lead', 'Architect', 'Engineering Manager'],
    
    context: `
      เราต้องการสื่อสารระหว่าง services โดยมี requirements:
      - Loose coupling ระหว่าง services
      - ทนต่อ service failures
      - รองรับ scale ได้
      - Audit trail สำหรับทุก business event
    `,
    
    decision: 'ใช้ Apache Kafka เป็น message broker หลักสำหรับ async communication',
    
    rationale: `
      Kafka ให้ทั้ง durability, replay capability, และ high throughput
      ที่จำเป็นสำหรับ event-driven architecture ของเรา
      Consumer groups ช่วยให้ scale ได้โดยไม่ต้อง coordinate กัน
    `,
    
    alternatives: [
      {
        option: 'RabbitMQ',
        pros: ['ง่ายกว่า Kafka', 'routing patterns หลากหลาย'],
        cons: ['ไม่รองรับ message replay', 'throughput ต่ำกว่า'],
        rejected: true,
        rejectionReason: 'ต้องการ message replay สำหรับ audit',
      },
      {
        option: 'Synchronous REST calls',
        pros: ['ง่ายต่อการ debug', 'immediate consistency'],
        cons: ['coupling สูง', 'cascade failures', 'ไม่ scale ได้ดี'],
        rejected: true,
        rejectionReason: 'ทำให้เกิด tight coupling และ cascade failures',
      },
    ],
    
    consequences: {
      positive: [
        'Services decoupled จากกัน',
        'Replay events สำหรับ audit และ debugging',
        'High throughput',
        'Resilient ต่อ downstream failures',
      ],
      negative: [
        'Eventual consistency ยาก debug',
        'Operational complexity เพิ่มขึ้น',
        'ต้องดูแล Kafka cluster',
      ],
      risks: [
        'Message ordering จะเป็นประเด็นถ้า partition key ไม่ถูกต้อง',
        'Schema evolution ต้องระวังเรื่อง backward compatibility',
      ],
    },
  },
  
  {
    id: 'ADR-002',
    title: 'Adopt Domain-Driven Design for Service Boundaries',
    status: 'accepted',
    date: new Date('2024-02-01'),
    deciders: ['Tech Lead', 'Architect', 'Domain Experts'],
    
    context: `
      เราต้องการ define service boundaries ที่ชัดเจนและ align กับ business
      หลังจากประสบปัญหา:
      - Services ที่ tight coupled กัน
      - ไม่ชัดเจนว่า service ไหน own data อะไร
      - Changes บ่อยทำให้ต้อง update หลาย services พร้อมกัน
    `,
    
    decision: 'ใช้ Domain-Driven Design (DDD) โดยเฉพาะ Bounded Contexts เพื่อ define service boundaries',
    
    rationale: 'Bounded Contexts ช่วยให้ services มี clear ownership และ align กับ business domains',
    
    alternatives: [],
    
    consequences: {
      positive: [
        'Clear ownership ของแต่ละ domain',
        'Independent deployability สูงขึ้น',
        'Teams align กับ business domains',
      ],
      negative: [
        'ต้องลงทุนเวลาเพื่อเรียนรู้ DDD',
        'อาจมี initial overhead',
      ],
      risks: [
        'Over-decomposition ถ้าไม่ระวัง',
        'Anemic domain model ถ้าใช้ DDD ไม่ถูกต้อง',
      ],
    },
  },
];
```

---

## สรุปท้ายบท - ภาพรวม Course ทั้งหมด

| Part | หัวข้อ | Maturity Level | ความสำคัญ |
|------|--------|----------------|-----------|
| 1-10 | Microservices Foundations | Level 1-2 | สูงมาก |
| 11-20 | Docker & Kubernetes | Level 2 | สูงมาก |
| 21-30 | Service Communication | Level 2-3 | สูง |
| 31-40 | Data Management | Level 3 | สูง |
| 41-50 | Security | Level 3-4 | สูงมาก |
| 51-60 | Observability | Level 3 | สูงมาก |
| 61-70 | Advanced Patterns | Level 3-4 | กลาง |
| 71-80 | Platform Engineering | Level 4 | กลาง |
| 81-90 | Mastery & Optimization | Level 4-5 | กลาง |
| Part 83 | API Design Best Practices | Level 2-3 | สูง |
| Part 84 | Data Privacy & GDPR | Level 3 | สูงมาก |
| Part 85 | Debugging Microservices | Level 2-3 | สูง |
| Part 86 | Message Brokers | Level 3 | สูง |
| Part 87 | gRPC | Level 3 | กลาง |
| Part 88 | Capacity Planning | Level 4 | กลาง |
| Part 89 | Incident Management | Level 3-4 | สูง |
| Part 90 | Maturity Model | Level 4-5 | สูง |

### Microservices Maturity Levels Summary

| Level | ชื่อ | Timeline | Key Capabilities |
|-------|------|----------|-----------------|
| 1 | Initial | 0-6 เดือน | Services แยกออกจาก monolith, basic CI/CD |
| 2 | Managed | 6-18 เดือน | Container, basic monitoring, API standards |
| 3 | Defined | 18-36 เดือน | Distributed tracing, service mesh, SLOs |
| 4 | Quantitatively Managed | 3-5 ปี | Error budgets, chaos engineering, cost attribution |
| 5 | Optimizing | 5+ ปี | ML-based detection, auto-remediation, IDP |

### คำแนะนำสำหรับ Journey

1. **อย่า rush** - การก้าวข้าม level ที่เร็วเกินไปทำให้เกิดปัญหา technical debt
2. **People first** - Technical tools ไม่ work ถ้าคนไม่พร้อม
3. **Measure everything** - ถ้าวัดไม่ได้ ปรับปรุงไม่ได้
4. **Blameless culture** - สำคัญที่สุดในการ scale organization
5. **Platform thinking** - ลงทุน developer experience เพื่อ velocity
6. **Start small** - Prove value ก่อน scale
7. **Document decisions** - ADR ช่วยให้ team เข้าใจ context
8. **Celebrate progress** - การ migrate เป็น marathon ไม่ใช่ sprint

### Continuous Improvement Framework

```
Plan -> Do -> Check -> Act (PDCA)

สำหรับ Microservices:
1. PLAN: กำหนด target maturity level และ roadmap
2. DO: Implement ตาม roadmap
3. CHECK: Assess maturity ทุกไตรมาส
4. ACT: Adjust roadmap ตาม learnings
```

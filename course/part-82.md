# Part 82: Team Organization สำหรับ Microservices

## บทนำ

"Organizations which design systems are constrained to produce designs which are copies of the communication structures of those organizations." — Melvin Conway, 1967

Conway's Law ไม่ใช่แค่ทฤษฎี แต่เป็นความจริงที่ทุก engineering organization เผชิญ บทนี้จะสำรวจวิธีจัด team structure ให้สอดคล้องกับ microservices architecture และสร้าง ownership model ที่ชัดเจน

---

## Conway's Law ในทางปฏิบัติ

### ตัวอย่างจริง: Team Structure กำหนด Architecture

```
❌ BAD: Functional Team Structure → Distributed Monolith

Organization:
├── Frontend Team (React)
├── Backend Team (Node.js)
├── Database Team (MySQL)
└── DevOps Team (Kubernetes)

ผลลัพธ์ที่ได้:
┌──────────────────────────────────────────────────────────────────────┐
│                                                                      │
│  Frontend ──── Backend API ──── Database                             │
│                    │                                                  │
│                DevOps CI/CD                                          │
│                                                                      │
│  = Layered Architecture (ไม่ใช่ Microservices)                       │
└──────────────────────────────────────────────────────────────────────┘

✅ GOOD: Cross-Functional Team Structure → True Microservices

Organization:
├── Order Team (Frontend + Backend + DB + Infra ownership)
├── User Team (Frontend + Backend + DB + Infra ownership)
├── Payment Team (Frontend + Backend + DB + Infra ownership)
└── Platform Team (shared infrastructure, tools)

ผลลัพธ์ที่ได้:
┌────────────────────────────────────────────────────────────────────┐
│                                                                    │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                         │
│  │  Order   │  │  User    │  │ Payment  │                         │
│  │  Service │  │  Service │  │  Service │                         │
│  │  (owned) │  │  (owned) │  │  (owned) │                         │
│  └──────────┘  └──────────┘  └──────────┘                         │
│                                                                    │
│         ┌──────────────────────────────────────┐                  │
│         │      Platform Team (shared infra)    │                  │
│         └──────────────────────────────────────┘                  │
└────────────────────────────────────────────────────────────────────┘
```

---

## Team Topologies Framework

Matthew Skelton และ Manuel Pais ได้กำหนด 4 ประเภท Team ที่เหมาะกับ Modern Software Organizations:

### 1. Stream-aligned Team

```
Stream-aligned Team คือ team หลักที่ deliver value ให้ business

ลักษณะ:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
- Owns ทั้ง service lifecycle: design → develop → deploy → operate
- Cross-functional: Frontend + Backend + QA + Product
- ขนาด: 5-8 คน (2-pizza rule)
- มี autonomy ในการตัดสินใจ technical ใน domain ของตัว
- Responsible for on-call ของ service ที่ตัวเอง own

ตัวอย่าง Team Charter:
```

```yaml
# team-charters/order-team.yaml

team:
  name: Order Tribe
  mission: "Own the complete order lifecycle from cart to delivery confirmation"
  
  members:
    - role: Engineering Lead
      count: 1
      responsibilities:
        - Technical direction and architecture decisions
        - Code review standards
        - Technical debt management
    
    - role: Senior Backend Engineer
      count: 2
      responsibilities:
        - Core service development
        - API design and contracts
        - Database design
    
    - role: Backend Engineer
      count: 2
      responsibilities:
        - Feature implementation
        - Bug fixes
        - Unit and integration tests
    
    - role: Frontend Engineer
      count: 1
      responsibilities:
        - Order UI components
        - API integration
    
    - role: QA Engineer
      count: 1
      responsibilities:
        - Test strategy
        - E2E test automation
        - Performance testing

  owned_services:
    - name: order-service
      repo: github.com/company/order-service
      on_call: true
      sla: "99.9% uptime, p99 < 200ms"
    
    - name: order-notification-service
      repo: github.com/company/order-notification-service
      on_call: true
    
    - name: order-analytics-pipeline
      repo: github.com/company/order-analytics-pipeline
      on_call: false  # batch job, not customer-facing

  communication:
    slack_channel: "#team-order"
    on_call_rotation: PagerDuty (weekly rotation)
    sprint_length: 2 weeks
    
  dependencies:
    upstream: []  # services ที่ order team consume
    - service: user-service
      type: synchronous
      contact: "@user-team"
    - service: product-service
      type: event
      topic: "product.*"
    
    downstream: []  # services ที่ consume order team's APIs
    - service: delivery-service
      type: event
      topic: "order.confirmed"
    - service: analytics-service
      type: event
      topic: "order.*"
```

### 2. Platform Team

```
Platform Team สร้าง Internal Developer Platform (IDP) 
ให้ Stream-aligned Teams ทำงานได้เร็วขึ้น

ความรับผิดชอบหลัก:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
- Kubernetes cluster management
- CI/CD pipeline templates
- Observability stack (Prometheus, Grafana, Jaeger)
- Service mesh (Istio/Linkerd)
- Secret management (Vault)
- Developer portal (Backstage)
- Shared libraries

Platform Team ต้องมี Product mindset:
- Stream-aligned teams คือ "customers"
- วัดผลด้วย Developer Experience (DX) metrics
- มี SLA สำหรับ platform services
```

```typescript
// platform/developer-portal/src/service-catalog.ts
// Internal Developer Portal using Backstage

// catalog-info.yaml - ทุก service ต้อง register ตัวเอง
```

```yaml
# services/order-service/catalog-info.yaml

apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: order-service
  description: Manages the complete order lifecycle
  annotations:
    github.com/project-slug: company/order-service
    pagerduty.com/integration-key: ${PAGERDUTY_KEY}
    grafana/dashboard-selector: "order-service"
    backstage.io/techdocs-ref: dir:.
  tags:
    - microservice
    - nodejs
    - critical
  links:
    - url: https://grafana.company.com/order-service
      title: Grafana Dashboard
    - url: https://jaeger.company.com/search?service=order-service
      title: Distributed Traces
    - url: https://runbooks.company.com/order-service
      title: Runbooks

spec:
  type: service
  lifecycle: production
  owner: order-team
  system: commerce-platform
  
  providesApis:
    - order-api-v1
    - order-api-v2
  
  consumesApis:
    - user-api-v2
    - product-api-v1
  
  dependsOn:
    - resource:order-db
    - resource:order-redis
    - component:kafka
```

```typescript
// platform/src/golden-path/service-template.ts
// Golden Path Template สำหรับสร้าง service ใหม่

export const serviceTemplate: Template = {
  name: 'microservice-template',
  title: 'NestJS Microservice',
  description: 'Create a new NestJS microservice with best practices',
  
  parameters: [
    {
      title: 'Service Details',
      properties: {
        name: {
          title: 'Service Name',
          type: 'string',
          pattern: '^[a-z][a-z0-9-]*-service$',
        },
        owner: {
          title: 'Team Owner',
          type: 'string',
          enum: ['order-team', 'user-team', 'payment-team'],
        },
        database: {
          title: 'Database',
          type: 'string',
          enum: ['postgresql', 'mysql', 'mongodb', 'none'],
        },
      },
    },
  ],
  
  steps: [
    {
      id: 'fetch-base',
      name: 'Fetch Base Template',
      action: 'fetch:template',
      input: {
        url: './skeleton',
        values: {
          name: '${{ parameters.name }}',
          owner: '${{ parameters.owner }}',
        },
      },
    },
    {
      id: 'publish',
      name: 'Publish to GitHub',
      action: 'publish:github',
      input: {
        repoUrl: 'github.com/company/${{ parameters.name }}',
        defaultBranch: 'main',
      },
    },
    {
      id: 'register',
      name: 'Register in Catalog',
      action: 'catalog:register',
      input: {
        repoContentsUrl: '${{ steps.publish.output.repoContentsUrl }}',
      },
    },
  ],
};
```

### 3. Enabling Team

```
Enabling Team ช่วย Stream-aligned Teams 
เรียนรู้ทักษะใหม่ที่จำเป็น (temporary role)

ตัวอย่าง:
- Security champions team ที่ help teams implement security practices
- SRE advisory team ที่ช่วย teams ทำ SLO/SLA
- Architecture guild ที่ review และ guide technical decisions

Enabling Team ≠ Dependency
ทำงานกับ stream-aligned team สักระยะ แล้วถ่ายทอด skills
และออกไป ไม่ใช่ team ที่ต้องขอ approval ทุกอย่าง
```

### 4. Complicated Subsystem Team

```
Complicated Subsystem Team manage components ที่ต้องการ
deep specialist knowledge

ตัวอย่าง:
- Machine Learning platform team
- Real-time payment processing engine
- Video encoding pipeline
- Geospatial calculation engine

ข้อสังเกต: ทีมประเภทนี้ควรมีน้อย
ถ้ามีมากเกินไป อาจเป็นสัญญาณว่า architecture ซับซ้อนเกินไป
```

---

## Ownership Model

### RACI Matrix สำหรับ Microservices

```
RACI = Responsible, Accountable, Consulted, Informed

                        │Order Team│Platform│Security│CTO Office│
────────────────────────┼──────────┼────────┼────────┼──────────┤
Service Development     │    R/A   │   C    │   C    │    I     │
Production Deployment   │    R/A   │   C    │   I    │    I     │
On-call Response        │    R/A   │   C    │   I    │    I     │
Database Schema Change  │    R/A   │   I    │   C    │    I     │
External API Contract   │    R     │   I    │   C    │    A     │
Security Vulnerability  │    R     │   C    │   A    │    I     │
Cost Optimization       │    R     │   C    │   I    │    A     │
Architecture Decision   │    R/A   │   C    │   C    │    C     │
```

### Service Ownership Document

```markdown
<!-- services/order-service/OWNERSHIP.md -->

# Order Service Ownership

## Primary Owner
Team: Order Team
Lead: Engineering Lead (Slack: @order-lead)
On-call: PagerDuty Schedule "order-team-oncall"

## Service SLA
- Availability: 99.9% (8.7 hours downtime/year)
- Error Rate: < 0.1% of requests
- P50 Latency: < 50ms
- P99 Latency: < 200ms
- P999 Latency: < 500ms

## Escalation Path
1. On-call Engineer (immediate)
2. Engineering Lead (after 15 min)
3. Engineering Manager (after 30 min)
4. VP Engineering (after 1 hour - Sev1 only)

## Runbook Links
- [Service Overview](runbooks/overview.md)
- [Common Incidents](runbooks/incidents.md)
- [Database Issues](runbooks/database.md)
- [Scaling Issues](runbooks/scaling.md)

## Dependencies
### We depend on:
| Service | Type | Contact | SLA |
|---------|------|---------|-----|
| user-service | Sync REST | @user-team | 99.9% |
| product-service | Async Event | @product-team | N/A |
| payment-service | Sync REST | @payment-team | 99.95% |

### Services that depend on us:
| Service | Type | Contact |
|---------|------|---------|
| delivery-service | Async Event | @delivery-team |
| analytics-pipeline | Async Event | @data-team |
```

---

## On-Call Rotation

### PagerDuty Configuration

```yaml
# pagerduty/schedules/order-team.yaml
# Managed via Terraform

resource "pagerduty_schedule" "order_team" {
  name      = "Order Team On-Call"
  time_zone = "Asia/Bangkok"

  layer {
    name                         = "Primary On-Call"
    start                        = "2024-01-01T00:00:00+07:00"
    rotation_virtual_start       = "2024-01-01T09:00:00+07:00"
    rotation_turn_length_seconds = 604800  # 1 week

    users = [
      pagerduty_user.engineer1.id,
      pagerduty_user.engineer2.id,
      pagerduty_user.engineer3.id,
      pagerduty_user.engineer4.id,
    ]

    restriction {
      type              = "weekly_restriction"
      start_time_of_day = "09:00:00"
      start_day_of_week = 1  # Monday
      duration_seconds  = 432000  # 5 days (Mon-Fri business hours)
    }
  }

  layer {
    name                         = "After Hours"
    start                        = "2024-01-01T00:00:00+07:00"
    rotation_virtual_start       = "2024-01-01T18:00:00+07:00"
    rotation_turn_length_seconds = 604800

    users = [
      pagerduty_user.senior_engineer1.id,
      pagerduty_user.senior_engineer2.id,
    ]

    restriction {
      type              = "weekly_restriction"
      start_time_of_day = "18:00:00"
      start_day_of_week = 1
      duration_seconds  = 57600  # 16 hours (6PM - 10AM)
    }
  }
}
```

### On-Call Runbook Template

```markdown
<!-- runbooks/on-call-guide.md -->

# On-Call Guide: Order Service

## Before You Start Your Rotation
- [ ] Verify PagerDuty app installed on phone
- [ ] Check Slack notifications are ON for #alerts-order
- [ ] Review recent incidents in postmortem doc
- [ ] Check current deployment status
- [ ] Ensure VPN access works

## Response Time SLA
| Severity | Response Time | Resolution Target |
|----------|---------------|-------------------|
| SEV1 | 5 minutes | 1 hour |
| SEV2 | 15 minutes | 4 hours |
| SEV3 | 30 minutes | 24 hours |
| SEV4 | Next business day | 1 week |

## Triage Steps
1. Check [Grafana Dashboard](https://grafana.company.com/order-service)
2. Check error rate: > 1% = SEV2, > 5% = SEV1
3. Check [Jaeger traces](https://jaeger.company.com) for slow requests
4. Check pod logs: `kubectl logs -n production -l app=order-service --tail=100`
5. Check downstream services (user, product, payment)

## Common Incidents

### High Error Rate (5xx)
```bash
# Check recent errors
kubectl logs -n production -l app=order-service --tail=500 | grep ERROR

# Check database connections
kubectl exec -n production deploy/order-service -- \
  node -e "require('./health').checkDb()"

# Check if rollback needed
kubectl rollout history deployment/order-service -n production
kubectl rollout undo deployment/order-service -n production  # if needed
```

### High Latency
```bash
# Check which endpoint is slow
kubectl top pods -n production -l app=order-service

# Check database query performance
kubectl exec -n production -it <db-pod> -- \
  psql -U order_svc -d orders -c "SELECT * FROM pg_stat_activity WHERE state='active';"

# Scale up if needed
kubectl scale deployment/order-service --replicas=10 -n production
```
```

---

## Incident Management

### Incident Severity Levels

```typescript
// shared/src/incident/severity.ts

export enum IncidentSeverity {
  SEV1 = 'SEV1', // Critical: Revenue loss, data loss, complete outage
  SEV2 = 'SEV2', // Major: Significant user impact, degraded service
  SEV3 = 'SEV3', // Minor: Limited user impact, workaround available
  SEV4 = 'SEV4', // Low: Cosmetic issues, minimal impact
}

export const SEVERITY_DEFINITIONS = {
  [IncidentSeverity.SEV1]: {
    description: 'Complete service outage or critical data loss',
    examples: [
      'Payment service down - cannot process orders',
      'Database corruption - data loss',
      'Security breach',
    ],
    responseTimeMinutes: 5,
    stakeholderNotification: ['Engineering', 'Product', 'CEO'],
    warRoomRequired: true,
    postmortemRequired: true,
  },
  [IncidentSeverity.SEV2]: {
    description: 'Partial outage or significantly degraded service',
    examples: [
      'Order creation failing for 20% of users',
      'Search returning wrong results',
      'Checkout flow broken',
    ],
    responseTimeMinutes: 15,
    stakeholderNotification: ['Engineering', 'Product'],
    warRoomRequired: true,
    postmortemRequired: true,
  },
  [IncidentSeverity.SEV3]: {
    description: 'Minor issues with limited user impact',
    examples: [
      'Order history loading slowly',
      'Email notifications delayed',
    ],
    responseTimeMinutes: 30,
    stakeholderNotification: ['Engineering'],
    warRoomRequired: false,
    postmortemRequired: false, // optional
  },
  [IncidentSeverity.SEV4]: {
    description: 'Cosmetic or trivial issues',
    examples: [
      'Wrong currency symbol displayed',
      'Typo in error message',
    ],
    responseTimeMinutes: 480, // next business day
    stakeholderNotification: [],
    warRoomRequired: false,
    postmortemRequired: false,
  },
};
```

### Incident Response Automation

```typescript
// platform/src/incident/incident-manager.ts

import { Injectable, Logger } from '@nestjs/common';
import { SlackService } from '../slack/slack.service';
import { PagerDutyService } from '../pagerduty/pagerduty.service';
import { JiraService } from '../jira/jira.service';

@Injectable()
export class IncidentManager {
  private readonly logger = new Logger(IncidentManager.name);

  constructor(
    private readonly slack: SlackService,
    private readonly pagerDuty: PagerDutyService,
    private readonly jira: JiraService,
  ) {}

  async declareIncident(params: DeclareIncidentParams): Promise<Incident> {
    const incident: Incident = {
      id: `INC-${Date.now()}`,
      severity: params.severity,
      title: params.title,
      service: params.service,
      declaredAt: new Date(),
      declaredBy: params.declaredBy,
      status: 'INVESTIGATING',
    };

    // 1. สร้าง Slack channel สำหรับ incident
    const channelId = await this.slack.createChannel(
      `inc-${incident.id.toLowerCase()}`
    );

    await this.slack.sendMessage(channelId, {
      text: `🚨 *${incident.severity} Incident Declared*`,
      blocks: [
        {
          type: 'section',
          text: {
            type: 'mrkdwn',
            text: [
              `*Incident ID:* ${incident.id}`,
              `*Severity:* ${incident.severity}`,
              `*Service:* ${incident.service}`,
              `*Title:* ${incident.title}`,
              `*Declared by:* ${incident.declaredBy}`,
              `*Time:* ${incident.declaredAt.toISOString()}`,
            ].join('\n'),
          },
        },
        {
          type: 'actions',
          elements: [
            {
              type: 'button',
              text: { type: 'plain_text', text: '📊 Dashboard' },
              url: `https://grafana.company.com/service/${incident.service}`,
            },
            {
              type: 'button',
              text: { type: 'plain_text', text: '🔍 Traces' },
              url: `https://jaeger.company.com/search?service=${incident.service}`,
            },
            {
              type: 'button',
              text: { type: 'plain_text', text: '📝 Runbook' },
              url: `https://runbooks.company.com/${incident.service}`,
            },
          ],
        },
      ],
    });

    // 2. Trigger PagerDuty ถ้า SEV1 หรือ SEV2
    if (['SEV1', 'SEV2'].includes(incident.severity)) {
      await this.pagerDuty.createIncident({
        title: `[${incident.severity}] ${incident.title}`,
        serviceId: await this.pagerDuty.getServiceId(incident.service),
        urgency: incident.severity === 'SEV1' ? 'high' : 'low',
        body: {
          type: 'incident_body',
          details: `Incident declared via automation. Channel: #inc-${incident.id.toLowerCase()}`,
        },
      });
    }

    // 3. สร้าง Jira ticket สำหรับ tracking
    const jiraTicket = await this.jira.createIssue({
      project: 'INC',
      issuetype: { name: 'Incident' },
      summary: `[${incident.severity}] ${incident.title}`,
      priority: { name: incident.severity === 'SEV1' ? 'Critical' : 'High' },
      labels: [incident.service, incident.severity.toLowerCase()],
    });

    // 4. Notify stakeholders ตาม severity
    const definition = SEVERITY_DEFINITIONS[incident.severity];
    for (const stakeholder of definition.stakeholderNotification) {
      await this.slack.notifyGroup(
        stakeholder,
        `⚠️ ${incident.severity} Incident: ${incident.title} - Join #inc-${incident.id.toLowerCase()}`
      );
    }

    return incident;
  }

  async resolveIncident(incidentId: string, resolution: string): Promise<void> {
    const channelId = `inc-${incidentId.toLowerCase()}`;
    
    await this.slack.sendMessage(channelId, {
      text: `✅ *Incident ${incidentId} RESOLVED*\n\n*Resolution:* ${resolution}`,
    });

    // Remind team to write postmortem
    await this.slack.sendMessage(channelId, {
      text: `📝 Please complete the postmortem within 48 hours:\nhttps://postmortems.company.com/new?incident=${incidentId}`,
    });
  }
}
```

---

## Postmortem Template

```markdown
<!-- postmortems/template.md -->

# Incident Postmortem: [INCIDENT-ID]

**Date:** YYYY-MM-DD  
**Severity:** SEV[1-4]  
**Duration:** X hours Y minutes  
**Author(s):** @engineer1, @engineer2  
**Review Status:** Draft / In Review / Final  

---

## Summary
[2-3 sentences describing what happened and impact]

## Timeline (UTC+7)
| Time | Event |
|------|-------|
| HH:MM | Monitoring alert triggered |
| HH:MM | On-call engineer acknowledged |
| HH:MM | Root cause identified |
| HH:MM | Fix deployed |
| HH:MM | Service restored to normal |
| HH:MM | Incident declared resolved |

## Root Cause Analysis

### What happened?
[Technical description of the failure]

### Why did it happen? (5 Whys)
1. Why? → [Answer]
2. Why? → [Answer]
3. Why? → [Answer]
4. Why? → [Answer]
5. Why? → [Root cause]

## Impact
- **Users affected:** X,XXX users
- **Revenue impact:** ฿X,XXX,XXX (estimated)
- **Duration:** X hours
- **Services affected:** order-service, delivery-service

## What Went Well
- [Thing 1 that worked]
- [Thing 2 that worked]

## What Went Wrong
- [Thing 1 that didn't work]
- [Thing 2 that didn't work]

## Action Items
| Action | Owner | Due Date | Priority |
|--------|-------|----------|----------|
| Add monitoring for X | @engineer | 2 weeks | High |
| Improve circuit breaker config | @engineer | 1 week | Critical |
| Update runbook for Y scenario | @engineer | 1 week | Medium |

## Lessons Learned
[Key takeaways for the team and organization]

---
*This is a blameless postmortem. We focus on systems and processes, not individuals.*
```

---

## Engineering Metrics for Team Health

```typescript
// platform/src/metrics/team-health.ts

interface TeamHealthMetrics {
  // DORA Metrics
  deploymentFrequency: number;     // deploys per day
  leadTimeForChanges: number;      // hours from commit to production
  changeFailureRate: number;       // % of deployments causing incidents
  meanTimeToRestore: number;       // hours to restore service

  // Team Satisfaction
  onCallBurden: number;            // incidents per engineer per month
  technicalDebtRatio: number;      // % of sprint on tech debt

  // Service Health
  errorBudgetRemaining: number;    // % of error budget left
  p99LatencyCompliance: number;    // % of time meeting p99 SLA
}

// Grafana dashboard config สำหรับ team health
export const teamHealthDashboard = {
  panels: [
    {
      title: 'Deployment Frequency',
      query: 'rate(deployments_total[7d]) * 86400',
      thresholds: {
        green: { value: 1, label: 'Elite: > 1/day' },
        yellow: { value: 0.14, label: 'High: > 1/week' },
        red: { value: 0, label: 'Medium/Low: < 1/week' },
      },
    },
    {
      title: 'Lead Time for Changes',
      query: 'histogram_quantile(0.5, deployment_lead_time_hours)',
      thresholds: {
        green: { value: 0, max: 1, label: 'Elite: < 1 hour' },
        yellow: { value: 1, max: 24, label: 'High: 1-24 hours' },
        red: { value: 24, label: 'Medium/Low: > 1 day' },
      },
    },
    {
      title: 'On-call Incidents per Engineer',
      query: 'sum(incidents_total[30d]) / team_size',
      thresholds: {
        green: { value: 0, max: 2, label: 'Healthy: < 2/month' },
        yellow: { value: 2, max: 5, label: 'Moderate: 2-5/month' },
        red: { value: 5, label: 'Concerning: > 5/month' },
      },
    },
  ],
};
```

---

## Interaction Modes ระหว่าง Teams

```
Team Interaction Modes (Team Topologies):

1. Collaboration (short-term, discovery)
   ┌──────────┐     ┌──────────┐
   │ Team A   │◄───►│ Team B   │
   └──────────┘     └──────────┘
   ใช้เมื่อ: ต้องการ innovate ร่วมกัน, ช่วงแรกของ new service

2. X-as-a-Service (long-term, stable)
   ┌──────────┐  API  ┌──────────┐
   │ Team A   │──────►│ Platform │
   │(consumer)│       │  Team    │
   └──────────┘       └──────────┘
   ใช้เมื่อ: Platform team deliver stable services (CI/CD, Observability)

3. Facilitating (temporary, enabling)
   ┌──────────┐  ~~~  ┌──────────┐
   │ Team A   │      ◄│ Enabling │
   │          │       │  Team    │
   └──────────┘       └──────────┘
   ใช้เมื่อ: Enabling team ช่วย stream team เรียนรู้ทักษะใหม่
```

---

## Practical: Team Setup Script

```bash
#!/bin/bash
# scripts/setup-new-team.sh
# Script สำหรับ onboard team ใหม่

TEAM_NAME="$1"
TEAM_LEAD="$2"

if [ -z "$TEAM_NAME" ] || [ -z "$TEAM_LEAD" ]; then
  echo "Usage: ./setup-new-team.sh <team-name> <team-lead-email>"
  exit 1
fi

echo "Setting up team: $TEAM_NAME"

# 1. Create GitHub team
gh api \
  --method POST \
  -H "Accept: application/vnd.github+json" \
  /orgs/company/teams \
  -f name="$TEAM_NAME" \
  -f description="Stream-aligned team for ${TEAM_NAME} domain" \
  -f privacy=closed

# 2. Create Slack channels
curl -X POST https://slack.com/api/conversations.create \
  -H "Authorization: Bearer $SLACK_TOKEN" \
  -d "name=team-${TEAM_NAME}&is_private=false"

curl -X POST https://slack.com/api/conversations.create \
  -H "Authorization: Bearer $SLACK_TOKEN" \
  -d "name=alerts-${TEAM_NAME}&is_private=false"

# 3. Create PagerDuty schedule
cat > /tmp/pd-schedule.json << EOF
{
  "schedule": {
    "name": "${TEAM_NAME^} Team On-Call",
    "time_zone": "Asia/Bangkok",
    "schedule_layers": [{
      "start": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
      "rotation_virtual_start": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
      "rotation_turn_length_seconds": 604800,
      "users": [{"user": {"type": "user_reference", "email": "$TEAM_LEAD"}}]
    }]
  }
}
EOF

curl -X POST https://api.pagerduty.com/schedules \
  -H "Authorization: Token token=$PAGERDUTY_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d @/tmp/pd-schedule.json

echo "Team setup complete for: $TEAM_NAME"
echo "Next steps:"
echo "  1. Add team members to GitHub team"
echo "  2. Create service repositories using Backstage template"
echo "  3. Set up Grafana dashboards"
echo "  4. Complete team charter document"
```

---

## สรุป

การจัด team organization ที่เหมาะสมสำหรับ Microservices ต้องคำนึงถึง:

| หัวข้อ | Best Practice |
|--------|---------------|
| Team Structure | Cross-functional, stream-aligned teams |
| Team Size | 5-8 คน (2-pizza rule) |
| Ownership | Clear ownership ของแต่ละ service |
| On-call | ทีมที่ build service ต้อง run service |
| Conway's Law | ออกแบบ team structure ก่อน, architecture ตามมา |
| Platform Team | สร้าง IDP เพื่อเพิ่ม developer velocity |
| Incidents | Response playbook, postmortem culture |

กุญแจสำคัญคือ **"you build it, you run it"** — ทีมที่สร้าง service ต้องรับผิดชอบ operation ด้วย นี่คือแรงผลักดันสำคัญที่ทำให้ทีมสร้างระบบที่ดีและ reliable

# Part 89: Incident Management

## บทนำ

Incident Management ที่ดีไม่ใช่แค่การแก้ปัญหาเร็ว แต่คือระบบที่ทำให้องค์กรเรียนรู้และแข็งแกร่งขึ้นจากทุก incident การสร้าง blameless culture และ runbook ที่ครบถ้วนทำให้ทีมสามารถ respond ได้อย่างมั่นใจแม้ในยามวิกฤต

---

## Incident Response Lifecycle

```
Incident Response Lifecycle:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  DETECT → ALERT → ACKNOWLEDGE → INVESTIGATE → MITIGATE         │
│                                                                 │
│  ↓                                                              │
│  RESOLVE → VERIFY → COMMUNICATE → CLOSE → POSTMORTEM           │
│                                                                 │
│  ↓                                                              │
│  IMPROVE (Preventive Actions)                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Incident Response Playbook

### Automated Incident Declaration

```typescript
// platform/src/incident/incident-lifecycle.service.ts

import { Injectable, Logger } from '@nestjs/common';
import { SlackService } from '../slack/slack.service';
import { PagerDutyService } from '../pagerduty/pagerduty.service';
import { JiraService } from '../jira/jira.service';
import { StatusPageService } from '../statuspage/statuspage.service';

export enum IncidentStatus {
  OPEN = 'OPEN',
  INVESTIGATING = 'INVESTIGATING',
  IDENTIFIED = 'IDENTIFIED',
  MONITORING = 'MONITORING',
  RESOLVED = 'RESOLVED',
}

export enum IncidentSeverity {
  SEV1 = 'SEV1',  // Critical
  SEV2 = 'SEV2',  // Major
  SEV3 = 'SEV3',  // Minor
  SEV4 = 'SEV4',  // Low
}

interface IncidentContext {
  id: string;
  title: string;
  severity: IncidentSeverity;
  affectedServices: string[];
  slackChannelId: string;
  pagerDutyIncidentId?: string;
  jiraTicketId?: string;
  startTime: Date;
  commanderUserId?: string;
}

@Injectable()
export class IncidentLifecycleService {
  private readonly logger = new Logger(IncidentLifecycleService.name);
  private activeIncidents: Map<string, IncidentContext> = new Map();

  constructor(
    private readonly slack: SlackService,
    private readonly pagerDuty: PagerDutyService,
    private readonly jira: JiraService,
    private readonly statusPage: StatusPageService,
  ) {}

  async declareIncident(params: {
    title: string;
    severity: IncidentSeverity;
    affectedServices: string[];
    declaredBy: string;
    description?: string;
  }): Promise<IncidentContext> {
    const incidentId = `INC-${new Date().toISOString().slice(0, 10).replace(/-/g, '')}-${
      Math.random().toString(36).substring(2, 7).toUpperCase()
    }`;

    this.logger.warn(`Incident declared: ${incidentId} - ${params.title}`);

    // 1. สร้าง Slack incident channel
    const channelName = `inc-${incidentId.toLowerCase()}`;
    const channelId = await this.slack.createChannel(channelName);

    // 2. Invite on-call engineers
    await this.slack.inviteToChannel(channelId, [
      await this.pagerDuty.getCurrentOnCallUser(params.affectedServices[0]),
    ]);

    // 3. Post initial incident message
    await this.slack.sendMessage(channelId, this.buildIncidentCard(
      incidentId,
      params,
    ));

    // 4. Pin runbook ไว้ใน channel
    for (const service of params.affectedServices) {
      await this.slack.sendMessage(channelId, {
        text: `📖 Runbook: https://runbooks.company.com/${service}`,
      });
    }

    // 5. Trigger PagerDuty สำหรับ SEV1/SEV2
    let pdIncidentId: string | undefined;
    if ([IncidentSeverity.SEV1, IncidentSeverity.SEV2].includes(params.severity)) {
      pdIncidentId = await this.pagerDuty.triggerIncident({
        title: `[${params.severity}] ${params.title}`,
        services: params.affectedServices,
        slackChannelUrl: `https://company.slack.com/archives/${channelId}`,
      });
    }

    // 6. สร้าง Jira ticket
    const jiraTicketId = await this.jira.createIncidentTicket({
      summary: `[${params.severity}] ${params.title}`,
      severity: params.severity,
      services: params.affectedServices,
    });

    // 7. Update status page สำหรับ SEV1/SEV2
    if ([IncidentSeverity.SEV1, IncidentSeverity.SEV2].includes(params.severity)) {
      await this.statusPage.createIncident({
        title: params.title,
        status: 'investigating',
        components: params.affectedServices.map(s => ({ name: s, status: 'degraded_performance' })),
      });
    }

    const incident: IncidentContext = {
      id: incidentId,
      title: params.title,
      severity: params.severity,
      affectedServices: params.affectedServices,
      slackChannelId: channelId,
      pagerDutyIncidentId: pdIncidentId,
      jiraTicketId,
      startTime: new Date(),
    };

    this.activeIncidents.set(incidentId, incident);
    return incident;
  }

  async updateIncident(incidentId: string, update: {
    status?: IncidentStatus;
    message: string;
    updatedBy: string;
  }): Promise<void> {
    const incident = this.activeIncidents.get(incidentId);
    if (!incident) {
      throw new Error(`Incident ${incidentId} not found`);
    }

    await this.slack.sendMessage(incident.slackChannelId, {
      text: `📋 *Status Update* (${update.updatedBy})\n${update.message}`,
    });

    if (update.status === IncidentStatus.RESOLVED) {
      await this.resolveIncident(incidentId, update.message, update.updatedBy);
    }
  }

  async resolveIncident(
    incidentId: string,
    resolution: string,
    resolvedBy: string,
  ): Promise<void> {
    const incident = this.activeIncidents.get(incidentId);
    if (!incident) return;

    const duration = Math.round(
      (Date.now() - incident.startTime.getTime()) / 1000 / 60
    );

    // 1. Post resolution message
    await this.slack.sendMessage(incident.slackChannelId, {
      text: [
        `✅ *Incident ${incidentId} RESOLVED*`,
        `Duration: ${duration} minutes`,
        `Resolved by: ${resolvedBy}`,
        `Resolution: ${resolution}`,
        '',
        `📝 Please complete the postmortem within 48 hours:`,
        `https://postmortems.company.com/new?incident=${incidentId}`,
      ].join('\n'),
    });

    // 2. Resolve PagerDuty
    if (incident.pagerDutyIncidentId) {
      await this.pagerDuty.resolveIncident(incident.pagerDutyIncidentId);
    }

    // 3. Update status page
    await this.statusPage.resolveIncident(incident.id);

    // 4. Schedule postmortem reminder (48 hours later)
    await this.schedulePostmortemReminder(incident, 48);

    this.activeIncidents.delete(incidentId);
  }

  private buildIncidentCard(id: string, params: any): SlackMessage {
    return {
      text: `🚨 *${params.severity} Incident Declared: ${params.title}*`,
      blocks: [
        {
          type: 'header',
          text: {
            type: 'plain_text',
            text: `🚨 ${params.severity} - ${params.title}`,
          },
        },
        {
          type: 'section',
          fields: [
            { type: 'mrkdwn', text: `*Incident ID:*\n${id}` },
            { type: 'mrkdwn', text: `*Severity:*\n${params.severity}` },
            { type: 'mrkdwn', text: `*Affected Services:*\n${params.affectedServices.join(', ')}` },
            { type: 'mrkdwn', text: `*Declared by:*\n${params.declaredBy}` },
          ],
        },
        {
          type: 'section',
          text: {
            type: 'mrkdwn',
            text: `*Expected Response Times:*\n• SEV1: 5 min ack, 1hr resolve\n• SEV2: 15 min ack, 4hr resolve`,
          },
        },
        {
          type: 'actions',
          elements: [
            {
              type: 'button',
              text: { type: 'plain_text', text: '📊 Grafana' },
              url: `https://grafana.company.com/d/service-dashboard?service=${params.affectedServices[0]}`,
              style: 'danger',
            },
            {
              type: 'button',
              text: { type: 'plain_text', text: '🔍 Jaeger' },
              url: `https://jaeger.company.com/search?service=${params.affectedServices[0]}&lookback=1h`,
            },
            {
              type: 'button',
              text: { type: 'plain_text', text: '📖 Runbook' },
              url: `https://runbooks.company.com/${params.affectedServices[0]}`,
            },
          ],
        },
      ],
    };
  }
}
```

---

## On-Call Runbooks

### Service-Specific Runbook Template

```markdown
<!-- runbooks/order-service/high-error-rate.md -->

# Runbook: Order Service High Error Rate

## Alert: OrderServiceHighErrorRate
**Trigger:** Error rate > 1% for 5 minutes  
**Severity:** SEV2 (escalate to SEV1 if > 5%)  
**Team:** Order Team  

---

## Triage (do these in order)

### Step 1: Assess Severity (2 minutes)
```bash
# Check current error rate
curl -s "http://prometheus.monitoring.svc/api/v1/query?query=\
  sum(rate(http_requests_total{service='order-service',status=~'5..'}[5m])) /\
  sum(rate(http_requests_total{service='order-service'}[5m]))" | jq .

# Quick pod health check
kubectl get pods -n production -l app=order-service
```

ถ้า error rate > 5% → escalate to SEV1 immediately

### Step 2: Check Recent Deployments (2 minutes)
```bash
# ตรวจสอบว่ามี deployment เมื่อกี้ไหม
kubectl rollout history deployment/order-service -n production

# ถ้า deployment เมื่อกี้ → ลอง rollback ก่อน
kubectl rollout undo deployment/order-service -n production
kubectl rollout status deployment/order-service -n production --timeout=120s
```

### Step 3: Identify Error Type (5 minutes)
```bash
# ดู error ล่าสุด
kubectl logs -n production -l app=order-service \
  --tail=500 --timestamps \
  | grep -iE '"level":"error"' \
  | tail -20

# ดู error distribution
kubectl logs -n production -l app=order-service \
  --tail=1000 \
  | python3 -c "
import json, sys, collections
errors = []
for line in sys.stdin:
    try:
        data = json.loads(line)
        if data.get('level') == 'error':
            errors.append(data.get('message', 'unknown'))
    except: pass
counter = collections.Counter(errors)
for msg, count in counter.most_common(10):
    print(f'{count}: {msg}')
"
```

### Step 4: Check Dependencies
```bash
# Check ถ้า dependency services down
for svc in user-service product-service payment-service; do
  echo -n "$svc: "
  kubectl exec -n production deploy/order-service -- \
    wget -qO- --timeout=5 http://$svc:3000/health 2>/dev/null \
    && echo "OK" || echo "FAILED"
done
```

## Common Error Scenarios

### Scenario A: Database Connection Issues
```
Symptoms: "Error: getaddrinfo ENOTFOUND order-db"
          "Error: connection pool exhausted"
```

```bash
# Check DB pod status
kubectl get pods -n production -l app=order-db

# Check connection pool
kubectl exec -n production -it deploy/order-service -- \
  node -e "const { DataSource } = require('typeorm');
    const ds = global.appDataSource;
    console.log('Pool:', ds?.driver?.pool?.totalCount, 'total,',
      ds?.driver?.pool?.idleCount, 'idle');"

# ถ้า pool exhausted → restart pod (temporary fix)
kubectl rollout restart deployment/order-service -n production

# หาต้นเหตุ: long-running queries
kubectl exec -n production -it order-db-0 -- \
  psql -U order_svc -d orders -c "
    SELECT pid, now() - pg_stat_activity.query_start AS duration, query
    FROM pg_stat_activity
    WHERE (now() - pg_stat_activity.query_start) > interval '30 seconds';"
```

### Scenario B: Payment Service Unavailable
```
Symptoms: "Error: connect ECONNREFUSED payment-service"
          Circuit breaker state: OPEN
```

```bash
# Check payment service
kubectl get pods -n production -l app=payment-service
kubectl logs -n production -l app=payment-service --tail=50

# ถ้า payment service down → orders ยังสร้างได้ แต่ payment pending
# Check circuit breaker state
kubectl exec -n production deploy/order-service -- \
  wget -qO- http://localhost:3000/internal/circuit-breakers

# Escalate to payment team: @payment-team
```

### Scenario C: OOM Kill
```bash
# Check สำหรับ OOM events
kubectl get events -n production --sort-by='.lastTimestamp' \
  | grep -i "oom\|killed"

# Check memory usage
kubectl top pods -n production -l app=order-service

# Scale ขึ้นเพื่อ reduce load per pod
kubectl scale deployment/order-service --replicas=20 -n production
```

## Post-Resolution Checklist
- [ ] Error rate กลับมา < 0.1%
- [ ] ตรวจสอบ data integrity (orders ที่ fail ต้องการ manual processing?)
- [ ] Update incident timeline
- [ ] Complete runbook ถ้ามีขั้นตอนใหม่
- [ ] File postmortem (ถ้า SEV1/SEV2)
```

---

## SLO Breach Procedures

```typescript
// platform/src/slo/slo-breach-handler.ts

interface SLOConfig {
  service: string;
  name: string;
  target: number;          // e.g., 0.999 = 99.9%
  errorBudgetMinutes: number; // monthly error budget in minutes
  windowDays: number;      // rolling window
}

interface SLOStatus {
  currentSLO: number;
  errorBudgetRemainingPercent: number;
  burnRate: number;         // rate at which error budget is consumed
  projectedExhaustion: Date | null;
}

@Injectable()
export class SLOBreachHandler {
  private readonly SLO_CONFIGS: SLOConfig[] = [
    {
      service: 'order-service',
      name: 'Order API Availability',
      target: 0.999,
      errorBudgetMinutes: 43.8, // 43.8 minutes per month (0.1%)
      windowDays: 30,
    },
    {
      service: 'order-service',
      name: 'Order API Latency p99',
      target: 0.99,
      errorBudgetMinutes: 432, // ~7.2 hours per month
      windowDays: 30,
    },
  ];

  async checkSLOStatus(service: string): Promise<SLOStatus> {
    const config = this.SLO_CONFIGS.find(c => c.service === service);
    if (!config) throw new Error(`No SLO config for service: ${service}`);

    const windowSeconds = config.windowDays * 24 * 3600;
    
    // Query actual SLO
    const goodEvents = await this.prometheus.query(
      `sum(rate(http_requests_total{service="${service}",status!~"5.."}[${windowSeconds}s]))`
    );
    const totalEvents = await this.prometheus.query(
      `sum(rate(http_requests_total{service="${service}"}[${windowSeconds}s]))`
    );

    const currentSLO = parseFloat(goodEvents.value) / parseFloat(totalEvents.value);
    const errorBudgetConsumedPercent = 
      ((config.target - currentSLO) / (1 - config.target)) * 100;
    const errorBudgetRemainingPercent = 100 - errorBudgetConsumedPercent;

    // Burn rate: how fast we're consuming error budget
    const burnRate = errorBudgetConsumedPercent / 
      (Date.now() / 1000 / (config.windowDays * 24 * 3600) * 100);

    // Project exhaustion
    let projectedExhaustion: Date | null = null;
    if (burnRate > 1) {
      const remainingDays = (errorBudgetRemainingPercent / 100) / 
        (burnRate / (config.windowDays * 24));
      projectedExhaustion = new Date(Date.now() + remainingDays * 24 * 3600 * 1000);
    }

    return {
      currentSLO,
      errorBudgetRemainingPercent,
      burnRate,
      projectedExhaustion,
    };
  }

  // Multi-window burn rate alerts (Google SRE Book approach)
  async checkBurnRateAlerts(service: string): Promise<BurnRateAlert[]> {
    const alerts: BurnRateAlert[] = [];

    // Fast burn: 14.4x burn rate over 1 hour (exhausts 1hr from 100%)
    const fastBurnRate = await this.calculateBurnRate(service, '1h');
    if (fastBurnRate >= 14.4) {
      alerts.push({
        severity: IncidentSeverity.SEV1,
        window: '1h',
        burnRate: fastBurnRate,
        message: `Fast burn detected: error budget exhausted in ~3 days at this rate`,
      });
    }

    // Slow burn: 6x burn rate over 6 hours
    const slowBurnRate = await this.calculateBurnRate(service, '6h');
    if (slowBurnRate >= 6 && slowBurnRate < 14.4) {
      alerts.push({
        severity: IncidentSeverity.SEV2,
        window: '6h',
        burnRate: slowBurnRate,
        message: `Slow burn detected: error budget exhausted in ~7 days at this rate`,
      });
    }

    return alerts;
  }

  private async calculateBurnRate(service: string, window: string): Promise<number> {
    const errorRate = await this.prometheus.query(
      `sum(rate(http_requests_total{service="${service}",status=~"5.."}[${window}])) /
       sum(rate(http_requests_total{service="${service}"}[${window}]))`
    );
    const sloTarget = this.SLO_CONFIGS.find(c => c.service === service)?.target || 0.999;
    const errorBudget = 1 - sloTarget;
    return parseFloat(errorRate.value) / errorBudget;
  }
}
```

---

## Postmortem Templates

```markdown
<!-- templates/postmortem.md -->

# Postmortem: {{ incident_title }}

**Incident ID:** {{ incident_id }}  
**Date:** {{ date }}  
**Duration:** {{ duration }} ({{ start_time }} - {{ end_time }} UTC+7)  
**Severity:** {{ severity }}  
**Status:** {{ Draft | Under Review | Final }}  

**Authors:** {{ authors }}  
**Reviewers:** {{ reviewers }}  

---

## Impact Summary

| Metric | Value |
|--------|-------|
| Users Affected | {{ user_count }} |
| Orders Failed | {{ failed_orders }} |
| Revenue Impact (estimated) | ฿{{ revenue_impact }} |
| Availability During Incident | {{ availability }}% |

---

## Timeline (UTC+7)

| Time | Who | Event |
|------|-----|-------|
| {{ time }} | Monitoring | Alert fired: {{ alert_name }} |
| {{ time }} | {{ oncall }} | On-call acknowledged alert |
| {{ time }} | {{ oncall }} | Began investigation |
| {{ time }} | {{ engineer }} | Root cause identified: {{ root_cause_brief }} |
| {{ time }} | {{ engineer }} | Mitigation applied: {{ mitigation_brief }} |
| {{ time }} | {{ engineer }} | Service restored to normal |
| {{ time }} | {{ manager }} | Incident declared resolved |

**Time to Acknowledge:** {{ tta }} minutes  
**Time to Mitigate:** {{ ttm }} minutes  
**Time to Resolution:** {{ ttr }} minutes  

---

## What Happened (Technical Description)

[อธิบายรายละเอียดทางเทคนิคว่าเกิดอะไรขึ้น เป็น factual ไม่ใช่ blame]

---

## Root Cause Analysis

### Why did this happen? (5 Whys)

**Why 1:** ทำไม users ถึงเห็น 503 errors?  
→ เพราะ order-service pods ทุกตัว crash  

**Why 2:** ทำไม pods ถึง crash?  
→ เพราะ out of memory (OOM) kill  

**Why 3:** ทำไมถึง out of memory?  
→ เพราะ memory leak ใน connection pool  

**Why 4:** ทำไมถึงมี memory leak?  
→ เพราะ connection ไม่ถูก release เมื่อ query timeout  

**Why 5:** ทำไมถึงไม่มี timeout?  
→ เพราะ config ผิดพลาดใน deployment ใหม่ (config drift)  

**Root Cause:** Config drift ระหว่าง staging และ production ทำให้ database query timeout ไม่ถูก set  

---

## Contributing Factors

1. ไม่มี automated config validation ระหว่าง environments
2. Memory usage alerting threshold สูงเกินไป (alert เมื่อ 90%, ควรเป็น 80%)
3. Pod restart policy ไม่ถูก configure → ไม่มี automatic recovery

---

## What Went Well

- On-call response time ดีมาก (3 นาที หลัง alert)
- Runbook ช่วยให้ triage ได้เร็ว
- Rollback สำเร็จภายใน 5 นาที
- Communication ใน incident channel ชัดเจน

---

## What Could Be Improved

- Memory leak ถูก detect ช้า → ควรมี memory growth rate alert
- Config ระหว่าง staging/production ต่างกัน โดยไม่มีใครรู้
- ไม่มี automatic circuit breaker เพื่อป้องกัน cascade

---

## Action Items

| # | Action | Owner | Due | Priority |
|---|--------|-------|-----|----------|
| 1 | Add memory growth rate alert (> 50MB/min) | @devops | 1 week | HIGH |
| 2 | Implement config drift detection | @platform | 2 weeks | HIGH |
| 3 | Set database query timeout in all environments | @order-team | 3 days | CRITICAL |
| 4 | Add OOM protection with graceful degradation | @order-team | 1 week | HIGH |
| 5 | Update memory alert threshold from 90% to 75% | @devops | 3 days | MEDIUM |
| 6 | Add integration test for connection pool cleanup | @order-team | 2 weeks | MEDIUM |

---

## Lessons Learned

1. **Config Management:** ต้องมี single source of truth สำหรับ config ทุก environment
2. **Progressive alerting:** Alert สำหรับ memory growth rate ไม่ใช่แค่ absolute threshold
3. **Chaos Engineering:** การ test OOM scenario ใน staging ป้องกันได้

---

*Note: This is a blameless postmortem. We focus on improving systems, not assigning blame to individuals.*
```

---

## Blameless Culture Implementation

```typescript
// platform/src/culture/blameless-review.service.ts

@Injectable()
export class BlamelessReviewService {
  // ตรวจสอบ postmortem draft สำหรับ blame language
  async reviewPostmortem(draft: string): Promise<ReviewResult> {
    const blamePatterns = [
      /\b(fault|blame|mistake|error|failure)\s+(of|by|from)\s+([A-Z][a-z]+)/g,
      /\b([A-Z][a-z]+)\s+(forgot|failed|didn't|should have)/g,
      /should\s+have\s+(known|checked|tested)/g,
    ];

    const warnings: string[] = [];
    
    for (const pattern of blamePatterns) {
      const matches = draft.match(pattern);
      if (matches) {
        warnings.push(
          `Possible blame language found: "${matches.join('", "')}"` +
          '. Consider rephrasing to focus on system conditions.'
        );
      }
    }

    const systemFocusTerms = [
      'system', 'process', 'configuration', 'monitoring',
      'alert', 'runbook', 'automation', 'workflow',
    ];

    const systemFocusScore = systemFocusTerms.reduce((count, term) => {
      return count + (draft.toLowerCase().match(new RegExp(term, 'g'))?.length || 0);
    }, 0);

    return {
      hasBlameLanguage: warnings.length > 0,
      warnings,
      systemFocusScore,
      suggestion: warnings.length > 0
        ? 'Review highlighted sections and reframe to focus on systems, not individuals'
        : 'Good use of blameless language',
    };
  }
}
```

---

## Chaos Engineering as Incident Prevention

```typescript
// tools/chaos/src/chaos-experiments.ts

import { Injectable, Logger } from '@nestjs/common';

interface ChaosExperiment {
  name: string;
  hypothesis: string;
  actions: ChaosAction[];
  steadyStateMetrics: SteadyStateMetric[];
  rollbackActions: ChaosAction[];
}

@Injectable()
export class ChaosEngineeringService {
  private readonly logger = new Logger(ChaosEngineeringService.name);

  // Experiment 1: Database connection failure
  async runDatabaseFailureExperiment(): Promise<ExperimentResult> {
    const experiment: ChaosExperiment = {
      name: 'Database Connection Failure',
      hypothesis: 'Order service ยังทำงานได้เมื่อ database ไม่ available (graceful degradation)',
      
      steadyStateMetrics: [
        {
          name: 'error_rate',
          probe: () => this.getErrorRate('order-service'),
          tolerance: { max: 0.01 }, // < 1% error rate
        },
        {
          name: 'health_check',
          probe: () => this.checkHealth('order-service'),
          tolerance: { equals: true },
        },
      ],
      
      actions: [
        {
          type: 'network',
          target: 'order-db',
          action: 'add-latency',
          params: { latency: 5000, jitter: 1000, correlation: 100 },
          duration: 120, // 2 minutes
        },
      ],
      
      rollbackActions: [
        {
          type: 'network',
          target: 'order-db',
          action: 'remove-latency',
        },
      ],
    };

    return this.runExperiment(experiment);
  }

  // Experiment 2: Pod failure
  async runPodFailureExperiment(): Promise<ExperimentResult> {
    const experiment: ChaosExperiment = {
      name: 'Pod Failure',
      hypothesis: 'System maintains availability when 50% of order-service pods fail',
      
      steadyStateMetrics: [
        {
          name: 'availability',
          probe: () => this.getAvailability('order-service', '5m'),
          tolerance: { min: 0.99 },
        },
      ],
      
      actions: [
        {
          type: 'pod',
          target: 'order-service',
          action: 'kill',
          params: { percentage: 50 },
          duration: 300, // 5 minutes
        },
      ],
      
      rollbackActions: [],  // K8s will restart pods automatically
    };

    return this.runExperiment(experiment);
  }

  // Experiment 3: Memory pressure
  async runMemoryPressureExperiment(): Promise<ExperimentResult> {
    const experiment: ChaosExperiment = {
      name: 'Memory Pressure',
      hypothesis: 'Service degrades gracefully under memory pressure without OOM kill',
      
      steadyStateMetrics: [
        {
          name: 'oom_kills',
          probe: () => this.getOOMKills('order-service', '5m'),
          tolerance: { equals: 0 },
        },
      ],
      
      actions: [
        {
          type: 'resource',
          target: 'order-service',
          action: 'memory-hog',
          params: { percentage: 80 },
          duration: 180,
        },
      ],
      
      rollbackActions: [],
    };

    return this.runExperiment(experiment);
  }

  private async runExperiment(experiment: ChaosExperiment): Promise<ExperimentResult> {
    this.logger.log(`Starting chaos experiment: ${experiment.name}`);
    
    // 1. Verify steady state before
    const beforeState = await this.verifySteadyState(experiment.steadyStateMetrics);
    if (!beforeState.passed) {
      return {
        name: experiment.name,
        passed: false,
        reason: 'Steady state not met before experiment',
        beforeState,
      };
    }

    // 2. Run chaos actions
    for (const action of experiment.actions) {
      await this.executeAction(action);
    }

    // 3. Wait for experiment duration
    await new Promise(resolve => setTimeout(resolve, 60000)); // 1 min

    // 4. Verify steady state during
    const duringState = await this.verifySteadyState(experiment.steadyStateMetrics);

    // 5. Rollback
    for (const action of experiment.rollbackActions) {
      await this.executeAction(action);
    }

    // 6. Wait for recovery
    await new Promise(resolve => setTimeout(resolve, 30000));

    // 7. Verify steady state after
    const afterState = await this.verifySteadyState(experiment.steadyStateMetrics);

    const passed = duringState.passed && afterState.passed;
    
    this.logger.log(
      `Chaos experiment ${experiment.name}: ${passed ? 'PASSED' : 'FAILED'}`
    );

    return {
      name: experiment.name,
      hypothesis: experiment.hypothesis,
      passed,
      beforeState,
      duringState,
      afterState,
      insights: passed
        ? [`System behaved as expected under ${experiment.name} conditions`]
        : [`System did NOT maintain steady state during ${experiment.name}`],
    };
  }
}
```

```yaml
# kubernetes/chaos/chaos-schedule.yaml
# Chaos Monkey สำหรับ Kubernetes (ใช้ Chaos Toolkit / Litmus)

apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: order-service-chaos
  namespace: production
spec:
  appinfo:
    appns: production
    applabel: "app=order-service"
    appkind: deployment
  
  # จะ notify ผ่าน slack ก่อน run
  annotationCheck: 'false'
  
  experiments:
  - name: pod-delete
    spec:
      components:
        env:
        - name: TOTAL_CHAOS_DURATION
          value: "60"       # 60 seconds
        - name: CHAOS_INTERVAL
          value: "30"       # ลบ pod ทุก 30 วินาที
        - name: FORCE
          value: "false"
        - name: PERCENTAGE_PODS_TO_DELETE
          value: "33"       # ลบ 1 ใน 3 pods
  
  jobCleanUpPolicy: delete
  
  # Schedule: ทุกวันอาทิตย์ 2 AM (traffic ต่ำ)
```

---

## สรุป

Incident Management ที่ดีประกอบด้วย 3 ส่วนหลัก:

**1. Response (ตอบสนองเร็ว)**
- Runbooks พร้อมสำหรับ common scenarios
- Automated incident declaration
- Clear escalation paths
- War room (Slack channel) ทันที

**2. Learn (เรียนรู้จากทุก incident)**
- Blameless postmortems ทุก SEV1/SEV2
- 5 Whys root cause analysis
- Action items ที่ติดตามผลได้
- Lessons learned ที่กระจายทั่วองค์กร

**3. Prevent (ป้องกันล่วงหน้า)**
- Chaos engineering ทดสอบ resilience
- SLO burn rate alerts (ไม่ใช่แค่ threshold)
- Pre-production runbooks
- Game days (disaster drill)

| เครื่องมือ | ใช้สำหรับ |
|-----------|----------|
| PagerDuty | On-call rotation, alerting |
| Slack | War room, communication |
| Jira | Ticket tracking, action items |
| Grafana | Metrics during incident |
| Jaeger | Distributed tracing |
| StatusPage | Customer communication |
| Litmus/Chaos Toolkit | Chaos experiments |

จำไว้ว่า **MTTR (Mean Time to Recovery)** สำคัญกว่า MTBF (Mean Time Between Failures) ในระบบที่ซับซ้อน เพราะ failures จะเกิดขึ้นเสมอ สิ่งที่ต่างกันคือเราตอบสนองและ recover ได้เร็วแค่ไหน

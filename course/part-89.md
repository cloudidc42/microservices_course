# Part 89: Incident Management

## บทนำ

Incident Management คือกระบวนการตรวจจับ ตอบสนอง และแก้ไขเหตุการณ์ที่ส่งผลกระทบต่อระบบในช่วงเวลา production บทนี้จะครอบคลุมตั้งแต่ lifecycle ของ incident ไปจนถึง blameless culture ที่จำเป็นสำหรับทีมที่มีประสิทธิภาพ

---

## 1. Incident Response Lifecycle

### 1.1 Incident Classification

```typescript
// src/incident/types.ts

export enum IncidentSeverity {
  SEV1 = 1,  // Critical: Production down, all users affected
  SEV2 = 2,  // High: Major feature broken, many users affected
  SEV3 = 3,  // Medium: Feature degraded, some users affected
  SEV4 = 4,  // Low: Minor issue, few users affected, has workaround
  SEV5 = 5,  // Informational: Potential issue, monitoring required
}

export const SEVERITY_DEFINITIONS = {
  [IncidentSeverity.SEV1]: {
    name: 'Critical',
    description: 'Complete service outage or critical data loss',
    responseTime: '< 5 minutes',
    resolution: '< 1 hour',
    notifications: ['CEO', 'CTO', 'Engineering Head', 'All on-call'],
    color: '#FF0000',
    examples: [
      'Payment processing completely down',
      'All users cannot login',
      'Data corruption in production database',
    ],
  },
  [IncidentSeverity.SEV2]: {
    name: 'High',
    description: 'Major feature unavailable or significant performance degradation',
    responseTime: '< 15 minutes',
    resolution: '< 4 hours',
    notifications: ['Engineering Head', 'On-call engineer', 'Product Manager'],
    color: '#FF6600',
    examples: [
      'Checkout flow broken for 50% of users',
      'API response time > 5 seconds',
      'Failed deployments affecting users',
    ],
  },
  [IncidentSeverity.SEV3]: {
    name: 'Medium',
    description: 'Feature degraded but core functionality works',
    responseTime: '< 1 hour',
    resolution: '< 24 hours',
    notifications: ['On-call engineer', 'Team lead'],
    color: '#FFCC00',
    examples: [
      'Search returning incorrect results',
      'Email notifications delayed',
      'Dashboard showing stale data',
    ],
  },
  [IncidentSeverity.SEV4]: {
    name: 'Low',
    description: 'Minor issue with workaround available',
    responseTime: '< 4 hours',
    resolution: '< 1 week',
    notifications: ['On-call engineer'],
    color: '#0099FF',
    examples: [
      'Minor UI glitches',
      'Non-critical API endpoint slow',
      'Logging errors in development',
    ],
  },
};

export interface Incident {
  id: string;
  title: string;
  severity: IncidentSeverity;
  status: IncidentStatus;
  services: string[];
  affectedUsers: number;
  
  // Timestamps
  detectedAt: Date;
  acknowledgedAt?: Date;
  mitigatedAt?: Date;
  resolvedAt?: Date;
  
  // People
  detectedBy: string;
  commanderUserId?: string;
  responders: string[];
  
  // Communication
  statusPageUrl?: string;
  slackChannelId?: string;
  zoomMeetingUrl?: string;
  
  // Timeline
  timeline: IncidentEvent[];
  
  // Post-mortem
  postMortemUrl?: string;
  rootCauses?: string[];
  actionItems?: ActionItem[];
}

export type IncidentStatus = 
  | 'detected'
  | 'investigating'
  | 'identified'
  | 'mitigating'
  | 'monitoring'
  | 'resolved'
  | 'postmortem';

export interface IncidentEvent {
  timestamp: Date;
  type: 'update' | 'action' | 'escalation' | 'resolution';
  userId: string;
  message: string;
  metadata?: Record<string, any>;
}

export interface ActionItem {
  id: string;
  title: string;
  description: string;
  priority: 'critical' | 'high' | 'medium' | 'low';
  assignee?: string;
  dueDate?: Date;
  status: 'open' | 'in_progress' | 'done';
  linkedIssueUrl?: string;
}
```

### 1.2 Incident Manager Service

```typescript
// src/incident/incident-manager.ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { EventEmitter2 } from '@nestjs/event-emitter';

@Injectable()
export class IncidentManagerService {
  constructor(
    @InjectRepository(Incident)
    private incidentRepo: Repository<Incident>,
    private eventEmitter: EventEmitter2,
    private slackService: SlackService,
    private pagerdutyService: PagerDutyService,
    private statusPageService: StatusPageService,
  ) {}

  async declareIncident(
    title: string,
    severity: IncidentSeverity,
    services: string[],
    declaredBy: string
  ): Promise<Incident> {
    const incident: Incident = {
      id: `INC-${Date.now()}`,
      title,
      severity,
      status: 'detected',
      services,
      affectedUsers: 0,
      detectedAt: new Date(),
      detectedBy: declaredBy,
      responders: [declaredBy],
      timeline: [{
        timestamp: new Date(),
        type: 'update',
        userId: declaredBy,
        message: `Incident declared: ${title}`,
      }],
    };
    
    await this.incidentRepo.save(incident);
    
    // Trigger notifications based on severity
    await this.notifyResponders(incident);
    
    // Create Slack war room channel
    const channel = await this.slackService.createIncidentChannel(incident);
    incident.slackChannelId = channel.id;
    
    // Update status page
    if (severity <= IncidentSeverity.SEV2) {
      await this.statusPageService.createIncident({
        name: title,
        status: 'investigating',
        impact: severity === IncidentSeverity.SEV1 ? 'critical' : 'major',
        body: `We are currently investigating an issue with ${services.join(', ')}.`,
      });
    }
    
    // Trigger PagerDuty alert
    await this.pagerdutyService.triggerAlert({
      summary: `[${IncidentSeverity[severity]}] ${title}`,
      severity: this.mapSeverityToPD(severity),
      source: 'incident-management',
      incidentKey: incident.id,
    });
    
    await this.incidentRepo.save(incident);
    this.eventEmitter.emit('incident.declared', incident);
    
    return incident;
  }

  async updateIncident(
    incidentId: string,
    status: IncidentStatus,
    message: string,
    updatedBy: string
  ): Promise<Incident> {
    const incident = await this.incidentRepo.findOneOrFail({
      where: { id: incidentId }
    });
    
    const previousStatus = incident.status;
    incident.status = status;
    
    // Set timestamps based on status transitions
    if (status === 'mitigating' && !incident.mitigatedAt) {
      incident.mitigatedAt = new Date();
    }
    if (status === 'resolved' && !incident.resolvedAt) {
      incident.resolvedAt = new Date();
    }
    
    incident.timeline.push({
      timestamp: new Date(),
      type: 'update',
      userId: updatedBy,
      message,
      metadata: {
        previousStatus,
        newStatus: status,
      },
    });
    
    await this.incidentRepo.save(incident);
    
    // Update status page
    await this.statusPageService.updateIncident(incidentId, {
      status: this.mapStatusToStatusPage(status),
      body: message,
    });
    
    // Notify Slack
    await this.slackService.postToChannel(incident.slackChannelId!, {
      text: `*Incident Update* [${status.toUpperCase()}]\n${message}`,
      color: status === 'resolved' ? 'good' : 'warning',
    });
    
    if (status === 'resolved') {
      await this.startPostMortemProcess(incident);
    }
    
    return incident;
  }

  private async notifyResponders(incident: Incident): Promise<void> {
    const severityDef = SEVERITY_DEFINITIONS[incident.severity];
    
    // Post to main alert channel
    await this.slackService.postToChannel('#incidents', {
      blocks: [
        {
          type: 'header',
          text: {
            type: 'plain_text',
            text: `🚨 [${IncidentSeverity[incident.severity]}] ${incident.title}`,
          },
        },
        {
          type: 'section',
          fields: [
            { type: 'mrkdwn', text: `*Incident ID:*\n${incident.id}` },
            { type: 'mrkdwn', text: `*Severity:*\n${IncidentSeverity[incident.severity]}` },
            { type: 'mrkdwn', text: `*Affected Services:*\n${incident.services.join(', ')}` },
            { type: 'mrkdwn', text: `*Detected By:*\n${incident.detectedBy}` },
          ],
        },
        {
          type: 'actions',
          elements: [
            {
              type: 'button',
              text: { type: 'plain_text', text: 'Acknowledge' },
              style: 'primary',
              action_id: `ack_${incident.id}`,
            },
          ],
        },
      ],
    });
  }

  private async startPostMortemProcess(incident: Incident): Promise<void> {
    // Create post-mortem document automatically
    const doc = await this.confluenceService.createPage({
      title: `Post-mortem: ${incident.title} (${incident.id})`,
      content: this.generatePostMortemTemplate(incident),
      space: 'POSTMORTEMS',
    });
    
    incident.postMortemUrl = doc.url;
    await this.incidentRepo.save(incident);
    
    // Schedule post-mortem meeting
    await this.calendarService.createMeeting({
      title: `Post-mortem: ${incident.id}`,
      attendees: incident.responders,
      startTime: this.getPostMortemTime(incident),
      duration: 60,
      description: `Post-mortem for incident ${incident.id}: ${incident.title}\n\nDoc: ${doc.url}`,
    });
  }

  private mapSeverityToPD(severity: IncidentSeverity): string {
    const mapping: Record<IncidentSeverity, string> = {
      [IncidentSeverity.SEV1]: 'critical',
      [IncidentSeverity.SEV2]: 'error',
      [IncidentSeverity.SEV3]: 'warning',
      [IncidentSeverity.SEV4]: 'info',
      [IncidentSeverity.SEV5]: 'info',
    };
    return mapping[severity];
  }

  private mapStatusToStatusPage(status: IncidentStatus): string {
    const mapping: Record<IncidentStatus, string> = {
      'detected': 'investigating',
      'investigating': 'investigating',
      'identified': 'identified',
      'mitigating': 'monitoring',
      'monitoring': 'monitoring',
      'resolved': 'resolved',
      'postmortem': 'resolved',
    };
    return mapping[status];
  }

  private getPostMortemTime(incident: Incident): Date {
    const meetingTime = new Date();
    // SEV1/SEV2: ทำ post-mortem ภายใน 24 ชั่วโมง
    // SEV3: ภายใน 1 อาทิตย์
    const daysDelay = incident.severity <= IncidentSeverity.SEV2 ? 1 : 5;
    meetingTime.setDate(meetingTime.getDate() + daysDelay);
    meetingTime.setHours(14, 0, 0, 0); // 2 PM
    return meetingTime;
  }
}
```

---

## 2. Runbook Automation

### 2.1 Automated Runbook Framework

```typescript
// src/runbook/runbook-executor.ts
import { Injectable } from '@nestjs/common';

interface RunbookStep {
  id: string;
  name: string;
  description: string;
  automated: boolean;
  command?: string;
  handler?: () => Promise<RunbookStepResult>;
  rollback?: () => Promise<void>;
  approvalRequired?: boolean;
  timeout?: number;
}

interface RunbookStepResult {
  success: boolean;
  output?: string;
  error?: string;
  metrics?: Record<string, any>;
}

interface Runbook {
  id: string;
  name: string;
  description: string;
  severity: IncidentSeverity[];
  services: string[];
  steps: RunbookStep[];
  estimatedDuration: number; // minutes
}

// ตัวอย่าง Runbooks
export const RUNBOOKS: Runbook[] = [
  {
    id: 'high-memory-pod',
    name: 'High Memory Usage - Pod OOMKill',
    description: 'Handle pod killed due to out of memory',
    severity: [IncidentSeverity.SEV2, IncidentSeverity.SEV3],
    services: ['*'],
    estimatedDuration: 15,
    steps: [
      {
        id: '1',
        name: 'Check pod status',
        description: 'Verify pod is OOMKilled',
        automated: true,
        command: 'kubectl get pods -n {namespace} -l app={service} --field-selector=status.phase!=Running',
      },
      {
        id: '2',
        name: 'Check memory usage history',
        description: 'Review memory trends in Grafana',
        automated: false,
        description: 'Go to Grafana -> K8s Memory Dashboard -> Filter by {service}',
      },
      {
        id: '3',
        name: 'Restart affected pods',
        description: 'Rolling restart to recover service',
        automated: true,
        command: 'kubectl rollout restart deployment/{service} -n {namespace}',
        approvalRequired: true,
        rollback: async () => {
          // If restart makes things worse, rollback
        },
      },
      {
        id: '4',
        name: 'Increase memory limits temporarily',
        description: 'Increase pod memory limits by 50%',
        automated: true,
        approvalRequired: true,
        handler: async () => {
          return { success: true, output: 'Memory limits updated' };
        },
      },
      {
        id: '5',
        name: 'Create JIRA ticket',
        description: 'Track root cause investigation',
        automated: true,
        handler: async () => {
          return { success: true, output: 'Ticket created' };
        },
      },
    ],
  },
  {
    id: 'database-connection-exhaustion',
    name: 'Database Connection Pool Exhaustion',
    description: 'Handle database connection pool errors',
    severity: [IncidentSeverity.SEV1, IncidentSeverity.SEV2],
    services: ['user-service', 'order-service', 'payment-service'],
    estimatedDuration: 30,
    steps: [
      {
        id: '1',
        name: 'Check active connections',
        automated: true,
        command: 'psql -c "SELECT count(*), state FROM pg_stat_activity GROUP BY state;"',
      },
      {
        id: '2',
        name: 'Identify long-running queries',
        automated: true,
        command: `psql -c "SELECT pid, now() - pg_stat_activity.query_start AS duration, query, state FROM pg_stat_activity WHERE (now() - pg_stat_activity.query_start) > interval '5 minutes';"`,
      },
      {
        id: '3',
        name: 'Kill idle connections',
        automated: true,
        approvalRequired: true,
        command: `psql -c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE state = 'idle' AND state_change < NOW() - INTERVAL '10 minutes';"`,
      },
      {
        id: '4',
        name: 'Increase PgBouncer pool size',
        automated: true,
        approvalRequired: true,
        command: 'kubectl patch configmap pgbouncer-config -p \'{"data":{"pool_size":"50"}}\'',
      },
    ],
  },
];

@Injectable()
export class RunbookExecutor {
  async executeRunbook(
    runbookId: string,
    incidentId: string,
    variables: Record<string, string>
  ): Promise<{
    success: boolean;
    completedSteps: string[];
    failedStep?: string;
    results: Record<string, RunbookStepResult>;
  }> {
    const runbook = RUNBOOKS.find(r => r.id === runbookId);
    if (!runbook) throw new Error(`Runbook ${runbookId} not found`);
    
    const completedSteps: string[] = [];
    const results: Record<string, RunbookStepResult> = {};
    
    for (const step of runbook.steps) {
      console.log(`Executing step: ${step.name}`);
      
      if (step.approvalRequired) {
        const approved = await this.requestApproval(incidentId, step);
        if (!approved) {
          console.log(`Step ${step.id} skipped (not approved)`);
          continue;
        }
      }
      
      try {
        let result: RunbookStepResult;
        
        if (step.handler) {
          result = await this.executeWithTimeout(step.handler, step.timeout || 60000);
        } else if (step.command) {
          const resolvedCommand = this.resolveVariables(step.command, variables);
          result = await this.executeCommand(resolvedCommand);
        } else {
          // Manual step - just log and continue
          result = { success: true, output: 'Manual step - awaiting human action' };
        }
        
        results[step.id] = result;
        
        if (!result.success) {
          return {
            success: false,
            completedSteps,
            failedStep: step.id,
            results,
          };
        }
        
        completedSteps.push(step.id);
        
        // Log to incident timeline
        await this.logToIncident(incidentId, `Runbook step '${step.name}' completed: ${result.output}`);
        
      } catch (error) {
        console.error(`Step ${step.id} failed:`, error);
        
        // Attempt rollback
        if (step.rollback) {
          await step.rollback().catch(e => console.error('Rollback failed:', e));
        }
        
        return {
          success: false,
          completedSteps,
          failedStep: step.id,
          results,
        };
      }
    }
    
    return { success: true, completedSteps, results };
  }

  private resolveVariables(template: string, variables: Record<string, string>): string {
    return template.replace(/\{(\w+)\}/g, (_, key) => variables[key] || `{${key}}`);
  }

  private async executeWithTimeout<T>(fn: () => Promise<T>, timeout: number): Promise<T> {
    return Promise.race([
      fn(),
      new Promise<never>((_, reject) => 
        setTimeout(() => reject(new Error('Step timed out')), timeout)
      ),
    ]);
  }

  private async executeCommand(command: string): Promise<RunbookStepResult> {
    const { exec } = require('child_process');
    const { promisify } = require('util');
    const execAsync = promisify(exec);
    
    try {
      const { stdout, stderr } = await execAsync(command, { timeout: 30000 });
      return { success: true, output: stdout || stderr };
    } catch (error) {
      return { success: false, error: (error as Error).message };
    }
  }

  private async requestApproval(incidentId: string, step: RunbookStep): Promise<boolean> {
    // ส่ง approval request ไปยัง Slack
    const response = await this.slackService.sendApprovalRequest({
      channel: `#inc-${incidentId}`,
      text: `Approval required for runbook step: *${step.name}*\n${step.description}`,
      timeout: 300000, // 5 minutes
    });
    
    return response.approved;
  }

  private async logToIncident(incidentId: string, message: string): Promise<void> {
    // Log to incident timeline
  }
}
```

---

## 3. Post-mortem Process and Templates

### 3.1 Post-mortem Generator

```typescript
// src/postmortem/generator.ts

export function generatePostMortemTemplate(incident: Incident): string {
  const ttd = incident.acknowledgedAt 
    ? Math.round((incident.acknowledgedAt.getTime() - incident.detectedAt.getTime()) / 60000)
    : 'N/A';
  
  const ttr = incident.resolvedAt
    ? Math.round((incident.resolvedAt.getTime() - incident.detectedAt.getTime()) / 60000)
    : 'N/A';
    
  return `# Post-mortem: ${incident.title}

**Incident ID:** ${incident.id}  
**Date:** ${incident.detectedAt.toISOString().split('T')[0]}  
**Severity:** ${IncidentSeverity[incident.severity]}  
**Status:** DRAFT  

**Authors:** ${incident.responders.join(', ')}  
**Last Updated:** ${new Date().toISOString()}

---

## Summary

*1-2 ย่อหน้าอธิบายสิ่งที่เกิดขึ้น ผลกระทบ และสิ่งที่ทำเพื่อแก้ปัญหา*

${incident.title} occurred on ${incident.detectedAt.toISOString()}.
This incident affected ${incident.services.join(', ')} and impacted approximately ${incident.affectedUsers} users.

## Impact

- **Duration:** ${ttr} minutes
- **Services Affected:** ${incident.services.join(', ')}
- **Users Impacted:** ${incident.affectedUsers}
- **Revenue Impact:** *ประเมินผลกระทบทางการเงิน*
- **SLO Impact:** *SLO burned/error budget consumed*

## Timeline (UTC)

| Time | Event |
|------|-------|
${incident.timeline.map(e => `| ${e.timestamp.toISOString()} | ${e.message} |`).join('\n')}

## Detection

- **Detected:** ${incident.detectedAt.toISOString()}
- **Detected By:** ${incident.detectedBy}
- **Time to Detect:** ${ttd} minutes
- **Detection Method:** *Alert? Customer report? Internal testing?*

### What helped with detection?
*อะไรที่ทำให้ detect ได้เร็ว*

### What made detection harder?
*อะไรที่ทำให้ detect ช้า*

## Root Cause Analysis

### The 5 Whys

1. **Why did the service fail?**
   - *ตอบ*

2. **Why did [answer 1] happen?**
   - *ตอบ*

3. **Why did [answer 2] happen?**
   - *ตอบ*

4. **Why did [answer 3] happen?**
   - *ตอบ*

5. **Why did [answer 4] happen?**
   - *Root cause*

### Root Cause Summary

*อธิบาย root cause โดยไม่โทษคน*

## Contributing Factors

- *Factor 1: ...*
- *Factor 2: ...*

## Resolution

### What we did to resolve it

1. *Step 1*
2. *Step 2*

### Time to Resolve: ${ttr} minutes

## Lessons Learned

### What went well
- *หัวข้อที่ทำได้ดี*

### What went wrong  
- *หัวข้อที่ควรปรับปรุง*

### What was lucky
- *สิ่งที่โชคดีที่ไม่ทำให้แย่กว่านี้*

## Action Items

| ID | Title | Priority | Owner | Due Date | Status |
|----|-------|----------|-------|----------|--------|
| 1 | *ชื่อ action* | Critical | @owner | ${new Date(Date.now() + 7*24*60*60*1000).toISOString().split('T')[0]} | Open |

## Metrics

### SLO Impact
- Error budget burned: *X%*
- SLO compliance this week: *X%*

### Performance During Incident
- Peak error rate: *X%*
- Max latency: *Xms*
- Affected request count: *N*

---

*เขียนโดย: [ชื่อ] | Review โดย: [ชื่อ] | วันที่ publish: [วันที่]*
`;
}
```

---

## 4. SLO-Based Alerting

```typescript
// src/slo/slo-alerting.ts
interface SLO {
  name: string;
  service: string;
  indicator: 'availability' | 'latency' | 'error_rate';
  target: number;        // 0-100 (%)
  window: string;        // '30d', '7d'
  errorBudget: number;   // minutes/hour allowed to fail
}

interface ErrorBudgetStatus {
  sloName: string;
  remainingBudgetPercent: number;
  burnRate: number;           // current rate of consumption
  projectedExhaustion?: Date; // when budget will be exhausted
  shouldAlert: boolean;
  severity: IncidentSeverity;
}

export const SLOS: SLO[] = [
  {
    name: 'user-service-availability',
    service: 'user-service',
    indicator: 'availability',
    target: 99.9,
    window: '30d',
    errorBudget: 43.8, // 99.9% of 30 days = 43.8 minutes budget
  },
  {
    name: 'payment-service-availability',
    service: 'payment-service',
    indicator: 'availability',
    target: 99.99,
    window: '30d',
    errorBudget: 4.38, // 99.99% = 4.38 minutes budget
  },
  {
    name: 'api-gateway-latency-p99',
    service: 'api-gateway',
    indicator: 'latency',
    target: 99,  // 99% of requests < 500ms
    window: '7d',
    errorBudget: 100.8, // 1% of 7 days requests
  },
];

export class SLOAlertingService {
  async checkErrorBudgets(): Promise<ErrorBudgetStatus[]> {
    const statuses: ErrorBudgetStatus[] = [];
    
    for (const slo of SLOS) {
      const status = await this.calculateErrorBudgetStatus(slo);
      statuses.push(status);
      
      if (status.shouldAlert) {
        await this.triggerAlert(slo, status);
      }
    }
    
    return statuses;
  }

  private async calculateErrorBudgetStatus(slo: SLO): Promise<ErrorBudgetStatus> {
    // Query Prometheus for current error rate
    const currentErrorRate = await this.queryPrometheus(
      `sum(rate(http_requests_total{service="${slo.service}", status=~"5.."}[1h])) / 
       sum(rate(http_requests_total{service="${slo.service}"}[1h]))`
    );
    
    // Calculate burn rate (1x = consuming at exactly SLO pace)
    const allowedErrorRate = 1 - (slo.target / 100);
    const burnRate = currentErrorRate / allowedErrorRate;
    
    // Estimate remaining budget
    const consumedPercent = await this.getConsumedBudgetPercent(slo);
    const remainingPercent = 100 - consumedPercent;
    
    // Determine if we should alert based on burn rate and remaining budget
    // Google SRE recommends alerting at 2x burn rate with 5% budget remaining
    const shouldAlert = burnRate > 2 && remainingPercent < 50 ||
                       burnRate > 5 ||
                       remainingPercent < 10;
    
    // Calculate when budget will be exhausted
    let projectedExhaustion: Date | undefined;
    if (burnRate > 0 && remainingPercent > 0) {
      const minutesRemaining = (slo.errorBudget * remainingPercent) / 100;
      const minutesUntilExhaustion = minutesRemaining / burnRate;
      projectedExhaustion = new Date(Date.now() + minutesUntilExhaustion * 60 * 1000);
    }
    
    // Determine severity
    let severity: IncidentSeverity;
    if (remainingPercent < 10 || burnRate > 10) {
      severity = IncidentSeverity.SEV1;
    } else if (remainingPercent < 25 || burnRate > 5) {
      severity = IncidentSeverity.SEV2;
    } else {
      severity = IncidentSeverity.SEV3;
    }
    
    return {
      sloName: slo.name,
      remainingBudgetPercent: remainingPercent,
      burnRate,
      projectedExhaustion,
      shouldAlert,
      severity,
    };
  }

  private async triggerAlert(slo: SLO, status: ErrorBudgetStatus): Promise<void> {
    const message = `
🔥 *SLO Alert: ${slo.name}*

• Error Budget Remaining: *${status.remainingBudgetPercent.toFixed(1)}%*
• Current Burn Rate: *${status.burnRate.toFixed(1)}x*
${status.projectedExhaustion ? 
  `• Budget Exhaustion: *${status.projectedExhaustion.toISOString()}*` : ''}

*SLO Target:* ${slo.target}% (${slo.window} window)
*Service:* ${slo.service}

Action: Investigate immediately to prevent SLO violation.
Dashboard: https://grafana.example.com/d/slo-dashboard
    `;
    
    await this.slackService.postToChannel('#slo-alerts', { text: message });
    
    if (status.remainingBudgetPercent < 10) {
      await this.pagerdutyService.triggerAlert({
        summary: `SLO Error Budget Critical: ${slo.name}`,
        severity: 'critical',
        source: 'slo-monitoring',
        details: status,
      });
    }
  }

  private async queryPrometheus(query: string): Promise<number> {
    // In reality, use prometheus-query library
    return 0.001; // 0.1% error rate
  }

  private async getConsumedBudgetPercent(slo: SLO): Promise<number> {
    return 25; // 25% consumed
  }
}
```

---

## 5. Root Cause Analysis - 5 Whys and Fishbone

```typescript
// src/postmortem/rca.ts

interface FishboneCategory {
  category: 'People' | 'Process' | 'Technology' | 'Environment' | 'Measurement';
  causes: string[];
}

export class RootCauseAnalyzer {
  // 5 Whys Analysis
  performFiveWhys(
    problem: string,
    initialCause: string
  ): {
    problem: string;
    whys: Array<{ why: string; because: string }>;
    rootCause: string;
    recommendations: string[];
  } {
    // สร้าง template สำหรับ 5 Whys
    return {
      problem,
      whys: [
        { why: `Why did "${problem}" occur?`, because: initialCause },
        { why: `Why did "${initialCause}" happen?`, because: '...' },
        { why: 'Why did that happen?', because: '...' },
        { why: 'Why did that happen?', because: '...' },
        { why: 'Why did that happen?', because: '(Root Cause)' },
      ],
      rootCause: 'Root cause to be determined through investigation',
      recommendations: [
        'Immediate: Fix symptoms',
        'Short-term: Address contributing factors',
        'Long-term: Fix root cause',
      ],
    };
  }

  // Fishbone (Ishikawa) Diagram
  buildFishboneDiagram(problem: string): {
    problem: string;
    categories: FishboneCategory[];
  } {
    return {
      problem,
      categories: [
        {
          category: 'Technology',
          causes: [
            'Software bug',
            'Infrastructure failure',
            'Dependency failure',
            'Configuration error',
          ],
        },
        {
          category: 'Process',
          causes: [
            'Inadequate testing',
            'Missing code review',
            'No deployment checklist',
            'Insufficient monitoring',
          ],
        },
        {
          category: 'People',
          causes: [
            'Insufficient training',
            'Communication failure',
            'Missing documentation',
            'Unclear ownership',
          ],
        },
        {
          category: 'Environment',
          causes: [
            'High load / traffic spike',
            'Third-party service outage',
            'Cloud provider issues',
            'Network instability',
          ],
        },
        {
          category: 'Measurement',
          causes: [
            'Missing metrics',
            'Alert thresholds too high',
            'No baseline established',
            'Incorrect SLO definition',
          ],
        },
      ],
    };
  }
}
```

---

## 6. War Room Coordination

```typescript
// src/incident/war-room.ts
import { Injectable } from '@nestjs/common';

interface WarRoomConfig {
  incidentId: string;
  severity: IncidentSeverity;
  zoomMeetingId: string;
  slackChannelId: string;
  roles: WarRoomRole[];
}

interface WarRoomRole {
  role: 'incident_commander' | 'tech_lead' | 'communications' | 'scribe';
  userId: string;
  responsibilities: string[];
}

@Injectable()
export class WarRoomCoordinator {
  async setupWarRoom(incident: Incident): Promise<WarRoomConfig> {
    // Create Zoom meeting
    const zoomMeeting = await this.zoomService.createMeeting({
      topic: `🚨 War Room: ${incident.id} - ${incident.title}`,
      type: 1, // Instant meeting
      settings: {
        join_before_host: true,
        mute_upon_entry: false,
        auto_recording: 'cloud',
      },
    });
    
    // Create Slack channel
    const slackChannel = await this.slackService.createChannel(
      `inc-${incident.id.toLowerCase()}`,
      {
        is_private: false,
        description: `War room for incident ${incident.id}: ${incident.title}`,
      }
    );
    
    // Post initial message in channel
    await this.slackService.postToChannel(slackChannel.id, {
      blocks: this.buildWarRoomStartMessage(incident, zoomMeeting.join_url),
    });
    
    // Assign roles
    const roles = await this.assignRoles(incident);
    
    // Invite responders
    await this.slackService.inviteUsersToChannel(
      slackChannel.id,
      incident.responders
    );
    
    return {
      incidentId: incident.id,
      severity: incident.severity,
      zoomMeetingId: zoomMeeting.id,
      slackChannelId: slackChannel.id,
      roles,
    };
  }

  private buildWarRoomStartMessage(incident: Incident, zoomUrl: string): any[] {
    return [
      {
        type: 'header',
        text: {
          type: 'plain_text',
          text: `🚨 War Room: ${incident.id}`,
        },
      },
      {
        type: 'section',
        text: {
          type: 'mrkdwn',
          text: `*${incident.title}*\nSeverity: *${IncidentSeverity[incident.severity]}*`,
        },
      },
      {
        type: 'section',
        fields: [
          { type: 'mrkdwn', text: `*Join Zoom:*\n${zoomUrl}` },
          { type: 'mrkdwn', text: `*Affected Services:*\n${incident.services.join(', ')}` },
        ],
      },
      {
        type: 'section',
        text: {
          type: 'mrkdwn',
          text: `*War Room Checklist:*\n• [ ] Incident Commander assigned\n• [ ] Impact assessment done\n• [ ] Stakeholders notified\n• [ ] Status page updated\n• [ ] Resolution timeline estimated`,
        },
      },
    ];
  }

  private async assignRoles(incident: Incident): Promise<WarRoomRole[]> {
    return [
      {
        role: 'incident_commander',
        userId: incident.commanderUserId || incident.responders[0],
        responsibilities: [
          'Coordinate response effort',
          'Make key decisions',
          'Communicate with stakeholders',
          'Drive to resolution',
        ],
      },
      {
        role: 'tech_lead',
        userId: incident.responders[1] || incident.responders[0],
        responsibilities: [
          'Investigate root cause',
          'Implement fixes',
          'Guide technical decisions',
        ],
      },
      {
        role: 'communications',
        userId: incident.responders[2] || incident.responders[0],
        responsibilities: [
          'Update status page',
          'Communicate with customers',
          'Draft external communications',
          'Update Slack with status',
        ],
      },
      {
        role: 'scribe',
        userId: incident.responders[3] || incident.responders[0],
        responsibilities: [
          'Document timeline',
          'Record decisions made',
          'Note action items',
          'Prepare for post-mortem',
        ],
      },
    ];
  }
}
```

---

## 7. Blameless Culture

```typescript
// src/postmortem/blameless.ts

/**
 * หลักการ Blameless Post-mortem:
 * 
 * 1. ระบบล้มเหลว ไม่ใช่คนล้มเหลว
 * 2. ทุกคนทำสิ่งที่ดีที่สุดด้วยข้อมูลที่มีในขณะนั้น
 * 3. มุ่งหาสาเหตุ ไม่ใช่โทษผู้กระทำ
 * 4. เรียนรู้เพื่อปรับปรุงระบบ
 */

export class BlamelessCultureGuide {
  // Language guidelines สำหรับ post-mortem
  readonly languageGuidelines = {
    avoid: [
      'คุณทำผิด...',
      'ทำไมถึงไม่...',
      'ควรจะรู้ว่า...',
      'ถ้า X ไม่ทำ Y แล้ว...',
      'เพราะ X ไม่ระวัง...',
    ],
    prefer: [
      'ระบบขาด safeguard สำหรับ...',
      'Process ไม่ได้ include ขั้นตอน...',
      'ข้อมูลที่มีในขณะนั้นทำให้ตัดสินใจ...',
      'ระบบ monitoring ไม่ได้แจ้งเตือนเมื่อ...',
      'Documentation ไม่ได้อธิบาย edge case นี้...',
    ],
  };

  // Checklist สำหรับ blameless post-mortem
  readonly facilitatorChecklist = [
    'ตั้งกฎ ground rules ก่อนเริ่ม meeting',
    'เริ่มด้วย "ขอขอบคุณทุกคนที่ช่วยแก้ incident"',
    'ถ้าใครพูดโทษคน ให้ redirect กลับมาที่ระบบ',
    'focus ที่ timeline ไม่ใช่ individual actions',
    'ถามว่า "เราเรียนรู้อะไร" ไม่ใช่ "ใครผิด"',
    'สรุป action items ที่ improve ระบบ ไม่ใช่ลงโทษคน',
  ];

  generatePsychologicalSafetyStatement(): string {
    return `
## สัญญาของ Team ในการทำ Post-mortem นี้

เราเชื่อว่า:
- ทุกคนที่เกี่ยวข้องทำสิ่งที่ดีที่สุดด้วยข้อมูลที่มีในขณะนั้น
- เหตุการณ์เกิดจากความซับซ้อนของระบบ ไม่ใช่ความประมาทของบุคคล
- การเรียนรู้จากความผิดพลาดสำคัญกว่าการหาผู้รับผิดชอบ
- ทุกคนในห้องนี้ปลอดภัยที่จะพูดความจริง

เราตกลงที่จะ:
- พูดถึงระบบ ไม่ใช่บุคคล
- ฟังด้วยความเข้าใจ ไม่ใช่การพิพากษา  
- มุ่งหา action items ที่ปรับปรุงระบบ
- Celebrate ความกล้าที่จะแชร์ข้อมูล
`;
  }
}
```

---

## สรุปท้ายบท

| หัวข้อ | เครื่องมือ / Framework | Key Metric |
|--------|----------------------|------------|
| Incident Detection | PagerDuty, OpsGenie | MTTD (Mean Time to Detect) |
| Incident Response | Slack, Zoom, Statuspage | MTTR (Mean Time to Resolve) |
| Runbook Automation | Ansible, Scripts | Automation Rate |
| Post-mortem | Confluence, Notion | Action Item Completion Rate |
| SLO Alerting | Prometheus + Alertmanager | Error Budget Burn Rate |
| RCA | 5 Whys, Fishbone | Root Cause Found Rate |
| War Room | Zoom, Slack | Time to Assemble Team |
| Blameless Culture | Training, Process | Psychological Safety Score |

### Incident KPIs

- **MTTD** (Mean Time to Detect): เป้าหมาย < 5 นาที
- **MTTA** (Mean Time to Acknowledge): เป้าหมาย < 15 นาที  
- **MTTR** (Mean Time to Resolve): SEV1 < 1hr, SEV2 < 4hr
- **Post-mortem Completion Rate**: > 90% ภายใน 5 วันทำการ
- **Action Item Closure Rate**: > 80% ภายในกำหนด

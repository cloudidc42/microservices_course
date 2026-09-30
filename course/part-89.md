# Part 89: Incident Management

## บทนำ

Incident Management คือกระบวนการจัดการเหตุการณ์ที่ไม่คาดฝันในระบบ ตั้งแต่การตรวจจับ การแก้ไข จนถึงการเรียนรู้จากเหตุการณ์นั้น บทนี้ครอบคลุม lifecycle ของ incident การทำ runbook automation และ post-mortem process

---

## 1. Incident Lifecycle

### 1.1 ขั้นตอนหลักของ Incident Lifecycle

```
Detection → Triage → Mitigation → Resolution → Post-mortem

1. Detection    : ระบบตรวจพบ incident (Alert, User report)
2. Triage       : ประเมินความร้ายแรงและผลกระทบ
3. Mitigation   : ลดผลกระทบเบื้องต้น (Rollback, Scale up, Circuit break)
4. Resolution   : แก้ไข root cause จริง
5. Post-mortem  : เรียนรู้และป้องกันการเกิดซ้ำ
```

### 1.2 Severity Levels

| Severity | คำอธิบาย | Response Time | Example |
|----------|----------|---------------|---------|
| SEV1 (Critical) | ระบบ down หรือ data loss | < 5 นาที | Production database down |
| SEV2 (Major) | Feature หลักใช้ไม่ได้ | < 15 นาที | Payment service fails |
| SEV3 (Minor) | Performance degradation | < 1 ชั่วโมง | Slow response times |
| SEV4 (Low) | Minor issues | < 24 ชั่วโมง | UI bug, non-critical feature |

---

## 2. Runbook Automation

### 2.1 TypeScript RunbookExecutor

```typescript
// incident/runbook-executor.ts

interface RunbookStep {
  id: string;
  name: string;
  description: string;
  type: 'automated' | 'manual' | 'decision';
  automated?: () => Promise<StepResult>;
  decision?: {
    question: string;
    options: Array<{
      label: string;
      nextStepId: string;
    }>;
  };
  timeout?: number; // milliseconds
  retries?: number;
  rollback?: () => Promise<void>;
  skipCondition?: () => Promise<boolean>;
}

interface StepResult {
  success: boolean;
  output?: string;
  error?: string;
  data?: Record<string, unknown>;
}

interface RunbookExecution {
  runbookId: string;
  incidentId: string;
  startedAt: Date;
  completedAt?: Date;
  status: 'running' | 'completed' | 'failed' | 'paused' | 'cancelled';
  completedSteps: Array<{
    stepId: string;
    status: 'success' | 'failed' | 'skipped';
    result?: StepResult;
    duration: number;
  }>;
  currentStepId?: string;
  errors: string[];
}

interface Runbook {
  id: string;
  name: string;
  description: string;
  severity: string[];
  tags: string[];
  steps: RunbookStep[];
  firstStepId: string;
}

class RunbookExecutor {
  private executions: Map<string, RunbookExecution> = new Map();

  async execute(
    runbook: Runbook,
    incidentId: string,
    context: Record<string, unknown> = {},
    decisionHandler?: (question: string, options: Array<{ label: string; nextStepId: string }>) => Promise<string>
  ): Promise<RunbookExecution> {
    const execution: RunbookExecution = {
      runbookId: runbook.id,
      incidentId,
      startedAt: new Date(),
      status: 'running',
      completedSteps: [],
      errors: [],
    };

    this.executions.set(`${runbook.id}-${incidentId}`, execution);

    console.log(`[Runbook] Starting: ${runbook.name} for incident ${incidentId}`);

    const stepsMap = new Map(runbook.steps.map((s) => [s.id, s]));
    let currentStepId = runbook.firstStepId;

    while (currentStepId) {
      const step = stepsMap.get(currentStepId);
      if (!step) break;

      execution.currentStepId = currentStepId;
      const stepStart = Date.now();

      console.log(`[Runbook] Step: ${step.name} (${step.type})`);

      // ตรวจสอบ skip condition
      if (step.skipCondition) {
        const shouldSkip = await step.skipCondition().catch(() => false);
        if (shouldSkip) {
          execution.completedSteps.push({
            stepId: step.id,
            status: 'skipped',
            duration: Date.now() - stepStart,
          });
          console.log(`[Runbook] Skipped: ${step.name}`);
          currentStepId = this.getNextStepId(runbook, currentStepId);
          continue;
        }
      }

      let result: StepResult = { success: false };
      let nextStepId: string | undefined;

      try {
        switch (step.type) {
          case 'automated':
            result = await this.executeAutomatedStep(step, context);
            nextStepId = this.getNextStepId(runbook, currentStepId);
            break;

          case 'manual':
            execution.status = 'paused';
            console.log(`[Runbook] MANUAL ACTION REQUIRED: ${step.description}`);
            // รอ external trigger เพื่อ resume
            result = { success: true, output: 'Manual step completed' };
            nextStepId = this.getNextStepId(runbook, currentStepId);
            execution.status = 'running';
            break;

          case 'decision':
            if (step.decision && decisionHandler) {
              const chosenOptionLabel = await decisionHandler(
                step.decision.question,
                step.decision.options
              );
              const chosenOption = step.decision.options.find(
                (o) => o.label === chosenOptionLabel
              );
              nextStepId = chosenOption?.nextStepId;
              result = {
                success: true,
                output: `Decision: ${chosenOptionLabel}`,
              };
            }
            break;
        }

        execution.completedSteps.push({
          stepId: step.id,
          status: result.success ? 'success' : 'failed',
          result,
          duration: Date.now() - stepStart,
        });

        if (!result.success) {
          execution.errors.push(
            `Step ${step.name} failed: ${result.error || 'Unknown error'}`
          );

          // ลอง rollback ถ้ามี
          if (step.rollback) {
            console.log(`[Runbook] Rolling back: ${step.name}`);
            await step.rollback().catch((err) => {
              console.error(`Rollback failed for ${step.name}:`, err);
            });
          }

          execution.status = 'failed';
          break;
        }

        currentStepId = nextStepId || '';
      } catch (error: any) {
        execution.completedSteps.push({
          stepId: step.id,
          status: 'failed',
          result: { success: false, error: error.message },
          duration: Date.now() - stepStart,
        });
        execution.errors.push(`Step ${step.name} threw: ${error.message}`);
        execution.status = 'failed';
        break;
      }
    }

    if (execution.status === 'running') {
      execution.status = 'completed';
    }

    execution.completedAt = new Date();
    execution.currentStepId = undefined;

    console.log(`[Runbook] ${execution.status.toUpperCase()}: ${runbook.name}`);
    return execution;
  }

  private async executeAutomatedStep(
    step: RunbookStep,
    context: Record<string, unknown>
  ): Promise<StepResult> {
    if (!step.automated) {
      return { success: false, error: 'No automated function defined' };
    }

    const maxRetries = step.retries || 1;
    let lastError: Error | null = null;

    for (let attempt = 1; attempt <= maxRetries; attempt++) {
      try {
        const result = await Promise.race([
          step.automated(),
          new Promise<StepResult>((_, reject) =>
            setTimeout(
              () => reject(new Error('Step timeout')),
              step.timeout || 30000
            )
          ),
        ]);
        return result;
      } catch (error: any) {
        lastError = error;
        if (attempt < maxRetries) {
          console.log(`[Runbook] Retry ${attempt}/${maxRetries} for ${step.name}`);
          await new Promise((resolve) => setTimeout(resolve, 1000 * attempt));
        }
      }
    }

    return {
      success: false,
      error: lastError?.message || 'Step failed after all retries',
    };
  }

  private getNextStepId(runbook: Runbook, currentStepId: string): string {
    const currentIndex = runbook.steps.findIndex((s) => s.id === currentStepId);
    if (currentIndex < 0 || currentIndex >= runbook.steps.length - 1) return '';
    return runbook.steps[currentIndex + 1].id;
  }

  getExecution(runbookId: string, incidentId: string): RunbookExecution | undefined {
    return this.executions.get(`${runbookId}-${incidentId}`);
  }
}

// =========================================================
// ตัวอย่าง Runbook: Database Connection Exhaustion
// =========================================================

function createDatabaseRunbook(): Runbook {
  return {
    id: 'db-connection-exhaustion',
    name: 'Database Connection Exhaustion Runbook',
    description: 'Handle database connection pool exhaustion',
    severity: ['SEV1', 'SEV2'],
    tags: ['database', 'postgres', 'connections'],
    firstStepId: 'check-connections',
    steps: [
      {
        id: 'check-connections',
        name: 'ตรวจสอบจำนวน connections ปัจจุบัน',
        description: 'Query Prometheus สำหรับจำนวน DB connections',
        type: 'automated',
        automated: async () => {
          // จำลองการ query Prometheus
          const currentConnections = 95; // จำลอง: 95% ของ max
          console.log(`Current DB connections: ${currentConnections}%`);
          return {
            success: true,
            output: `DB connections at ${currentConnections}%`,
            data: { connectionPercent: currentConnections },
          };
        },
        timeout: 10000,
      },
      {
        id: 'identify-top-consumers',
        name: 'ระบุ services ที่ใช้ connections มากที่สุด',
        description: 'Query pg_stat_activity เพื่อหา top consumers',
        type: 'automated',
        automated: async () => {
          // จำลองการ query
          const topConsumers = [
            { service: 'order-service', connections: 45 },
            { service: 'user-service', connections: 30 },
            { service: 'report-service', connections: 20 },
          ];
          console.log('Top DB consumers:', topConsumers);
          return {
            success: true,
            output: JSON.stringify(topConsumers),
            data: { topConsumers },
          };
        },
        timeout: 15000,
      },
      {
        id: 'kill-idle-connections',
        name: 'Kill idle connections (> 10 นาที)',
        description: 'Terminate connections ที่ idle นานเกินกำหนด',
        type: 'automated',
        automated: async () => {
          // จำลองการ kill connections
          const killedCount = 15;
          console.log(`Killed ${killedCount} idle connections`);
          return {
            success: true,
            output: `Killed ${killedCount} idle connections`,
            data: { killedCount },
          };
        },
        rollback: async () => {
          console.log('Rollback: ไม่สามารถ rollback การ kill connections ได้');
        },
        timeout: 30000,
        retries: 2,
      },
      {
        id: 'check-pgbouncer',
        name: 'ตรวจสอบ PgBouncer configuration',
        description: 'ตรวจสอบว่า PgBouncer pool size เหมาะสมหรือไม่',
        type: 'decision',
        description: 'PgBouncer ถูก configure ไว้หรือไม่?',
        decision: {
          question: 'PgBouncer ถูก deploy ไว้ในระบบหรือไม่?',
          options: [
            { label: 'ใช่ - มี PgBouncer', nextStepId: 'update-pgbouncer-pool' },
            { label: 'ไม่มี - ไม่ได้ deploy PgBouncer', nextStepId: 'scale-down-consumers' },
          ],
        },
      },
      {
        id: 'update-pgbouncer-pool',
        name: 'Update PgBouncer pool size',
        description: 'เพิ่ม pool_size ใน PgBouncer config',
        type: 'automated',
        automated: async () => {
          // จำลองการ update config
          console.log('Updating PgBouncer pool_size to 50...');
          return {
            success: true,
            output: 'PgBouncer pool_size updated to 50',
          };
        },
        timeout: 30000,
      },
      {
        id: 'scale-down-consumers',
        name: 'Scale down report-service',
        description: 'ลด replicas ของ report-service (non-critical)',
        type: 'automated',
        automated: async () => {
          console.log('Scaling down report-service to 1 replica...');
          return {
            success: true,
            output: 'report-service scaled to 1 replica',
          };
        },
        rollback: async () => {
          console.log('Rolling back: scaling report-service back to 3 replicas');
        },
        timeout: 60000,
      },
      {
        id: 'verify-resolution',
        name: 'ตรวจสอบว่า connections กลับมาปกติ',
        description: 'รอ 2 นาทีแล้วตรวจสอบ connection count อีกครั้ง',
        type: 'automated',
        automated: async () => {
          await new Promise((resolve) => setTimeout(resolve, 2000)); // จำลองการรอ
          const currentConnections = 65; // จำลอง: ลดลงแล้ว
          const isResolved = currentConnections < 80;
          console.log(`Connection level after mitigation: ${currentConnections}%`);
          return {
            success: isResolved,
            output: `Connections: ${currentConnections}% (${isResolved ? 'RESOLVED' : 'STILL HIGH'})`,
            data: { connectionPercent: currentConnections },
          };
        },
        timeout: 120000,
      },
      {
        id: 'notify-team',
        name: 'แจ้งทีมผ่าน Slack',
        description: 'ส่ง notification ไปยัง #incidents channel',
        type: 'automated',
        automated: async () => {
          console.log('Sending Slack notification...');
          return {
            success: true,
            output: 'Slack notification sent',
          };
        },
        timeout: 10000,
      },
    ],
  };
}

// ตัวอย่างการใช้งาน
async function main() {
  const executor = new RunbookExecutor();
  const runbook = createDatabaseRunbook();

  const execution = await executor.execute(
    runbook,
    'INC-2024-001',
    { environment: 'production', cluster: 'k8s-prod' },
    async (question, options) => {
      console.log(`\nDecision needed: ${question}`);
      options.forEach((o, i) => console.log(`  ${i + 1}. ${o.label}`));
      // จำลองการตัดสินใจ
      return options[0].label;
    }
  );

  console.log('\n=== Execution Summary ===');
  console.log(`Status: ${execution.status}`);
  console.log(`Steps completed: ${execution.completedSteps.length}`);
  console.log(`Duration: ${execution.completedAt
    ? Math.round((execution.completedAt.getTime() - execution.startedAt.getTime()) / 1000)
    : 'N/A'} seconds`);

  if (execution.errors.length > 0) {
    console.log('Errors:', execution.errors);
  }
}

main().catch(console.error);
```

---

## 3. Post-mortem Template (YAML)

```yaml
# incident/post-mortem-template.yaml
post_mortem:
  incident_id: "INC-2024-001"
  title: "Database Connection Exhaustion - Production Outage"
  severity: "SEV1"
  status: "completed"

  dates:
    incident_start: "2024-01-15T14:30:00Z"
    incident_detected: "2024-01-15T14:32:00Z"
    mitigation_start: "2024-01-15T14:45:00Z"
    resolution: "2024-01-15T15:30:00Z"
    post_mortem_meeting: "2024-01-17T10:00:00Z"
    post_mortem_deadline: "2024-01-22T10:00:00Z"

  impact:
    duration_minutes: 60
    affected_services:
      - order-service
      - user-service
      - api-gateway
    affected_users_count: 15000
    affected_regions:
      - ap-southeast-1
    error_rate_peak_percent: 85
    revenue_impact_usd: 25000
    description: >
      ผู้ใช้ไม่สามารถ login และสร้าง order ได้เป็นเวลา 60 นาที
      Error rate พุ่งสูงถึง 85% ส่งผลให้เสียรายได้ประมาณ $25,000

  timeline:
    - time: "2024-01-15T14:30:00Z"
      event: "Database connections เริ่มพุ่งสูง"
      type: "trigger"
      actor: "automated"
    - time: "2024-01-15T14:32:00Z"
      event: "PagerDuty alert fired: DB connections > 90%"
      type: "detection"
      actor: "automated"
    - time: "2024-01-15T14:35:00Z"
      event: "On-call engineer รับ alert"
      type: "response"
      actor: "john.doe"
    - time: "2024-01-15T14:40:00Z"
      event: "War room opened ใน Slack #incident-2024-001"
      type: "response"
      actor: "john.doe"
    - time: "2024-01-15T14:45:00Z"
      event: "ระบุสาเหตุ: report-service เปิด connection ไม่ปิด"
      type: "diagnosis"
      actor: "jane.smith"
    - time: "2024-01-15T14:50:00Z"
      event: "Kill idle connections และ scale down report-service"
      type: "mitigation"
      actor: "john.doe"
    - time: "2024-01-15T15:10:00Z"
      event: "Error rate ลดลงเหลือ 5%"
      type: "progress"
      actor: "automated"
    - time: "2024-01-15T15:30:00Z"
      event: "ระบบกลับมาปกติ ปิด incident"
      type: "resolution"
      actor: "john.doe"

  root_cause:
    summary: "Connection pool leak ใน report-service"
    detailed: >
      report-service version 2.3.1 มี bug ที่ทำให้ database connection
      ไม่ถูกปิดอย่างถูกต้องหลังจาก query สิ้นสุด ส่งผลให้ connections
      สะสมจนเต็ม pool (100 connections) ในเวลาประมาณ 30 นาที
    contributing_factors:
      - "ขาด connection leak detection ใน monitoring"
      - "ไม่มี connection timeout ที่ database level"
      - "ขาด load test สำหรับ long-running reports"
    five_whys:
      - why: "ทำไม database connections หมด?"
        answer: "เพราะ connections ไม่ถูกปิดหลัง query"
      - why: "ทำไม connections ไม่ถูกปิด?"
        answer: "เพราะ report-service ไม่ได้ handle connection cleanup ใน error path"
      - why: "ทำไมไม่ handle error path?"
        answer: "เพราะ developer ลืม await promise ก่อน function return"
      - why: "ทำไมไม่ตรวจพบก่อน deploy?"
        answer: "เพราะไม่มี integration test สำหรับ long-running operations"
      - why: "ทำไมไม่มี integration test?"
        answer: "เพราะขาด test coverage standards สำหรับ database operations"

  what_went_well:
    - "Alert triggered ภายใน 2 นาทีหลัง incident เริ่ม"
    - "On-call engineer respond ภายใน 3 นาที"
    - "Runbook ช่วยให้ mitigation ทำได้เร็วขึ้น"
    - "Communication ใน war room ชัดเจน"

  what_went_poorly:
    - "ใช้เวลานานถึง 15 นาทีกว่าจะระบุ root cause"
    - "ไม่มี canary deployment ทำให้ bad version ไปถึง production ทั้งหมด"
    - "Customer communication ล่าช้า 20 นาที"

  action_items:
    - id: "AI-001"
      title: "เพิ่ม connection leak detection ใน monitoring"
      description: "สร้าง Prometheus alert สำหรับ connection count ต่อ service"
      priority: "P0"
      assignee: "jane.smith"
      due_date: "2024-01-22"
      status: "in-progress"

    - id: "AI-002"
      title: "เพิ่ม connection timeout ที่ database"
      description: "ตั้ง idle connection timeout = 5 นาที ใน PostgreSQL"
      priority: "P0"
      assignee: "john.doe"
      due_date: "2024-01-19"
      status: "completed"

    - id: "AI-003"
      title: "Fix connection leak ใน report-service"
      description: "เพิ่ม proper connection cleanup ใน all code paths"
      priority: "P0"
      assignee: "alice.wang"
      due_date: "2024-01-18"
      status: "completed"

    - id: "AI-004"
      title: "เพิ่ม integration tests สำหรับ database operations"
      description: "Test connection cleanup ใน happy path และ error path"
      priority: "P1"
      assignee: "alice.wang"
      due_date: "2024-01-31"
      status: "todo"

    - id: "AI-005"
      title: "Implement canary deployment"
      description: "Deploy ทีละ 5% แล้วค่อย gradually rollout"
      priority: "P1"
      assignee: "devops-team"
      due_date: "2024-02-15"
      status: "todo"

    - id: "AI-006"
      title: "สร้าง customer communication template"
      description: "Template สำหรับ status page และ email เมื่อเกิด incident"
      priority: "P2"
      assignee: "product-team"
      due_date: "2024-02-01"
      status: "todo"

  metrics:
    time_to_detect_minutes: 2
    time_to_respond_minutes: 3
    time_to_mitigate_minutes: 20
    time_to_resolve_minutes: 60
    mttr_minutes: 60

  attendees:
    - john.doe@company.com
    - jane.smith@company.com
    - alice.wang@company.com
    - bob.manager@company.com
```

---

## 4. SLO-based Alerting

```yaml
# monitoring/slo-alerts.yaml
# Multi-window multi-burn-rate alerts (Google SRE Book approach)

groups:
  - name: slo-alerts
    rules:
      # ======================================================
      # Order Service: Availability SLO = 99.9%
      # Error budget: 43.2 min/month (0.1%)
      # ======================================================

      # Fast burn: 14.4x burn rate in 1h AND 6h windows
      - alert: SLOErrorBudgetFastBurn
        expr: |
          (
            sum(rate(http_requests_total{service="order-service", status=~"5.."}[1h]))
            / sum(rate(http_requests_total{service="order-service"}[1h]))
          ) > (14.4 * 0.001)
          and
          (
            sum(rate(http_requests_total{service="order-service", status=~"5.."}[6h]))
            / sum(rate(http_requests_total{service="order-service"}[6h]))
          ) > (14.4 * 0.001)
        for: 1m
        labels:
          severity: critical
          slo: order-service-availability
        annotations:
          summary: "Error budget burning too fast (Critical)"
          description: |
            Order service is burning error budget at 14.4x rate.
            Current 1h error rate: {{ printf "%.2f" (mul $value 100) }}%
            SLO target: 99.9%
            At this rate, monthly error budget will be exhausted in ~2 hours.

      # Slow burn: 6x burn rate in 6h AND 3d windows
      - alert: SLOErrorBudgetSlowBurn
        expr: |
          (
            sum(rate(http_requests_total{service="order-service", status=~"5.."}[6h]))
            / sum(rate(http_requests_total{service="order-service"}[6h]))
          ) > (6 * 0.001)
          and
          (
            sum(rate(http_requests_total{service="order-service", status=~"5.."}[3d]))
            / sum(rate(http_requests_total{service="order-service"}[3d]))
          ) > (6 * 0.001)
        for: 1m
        labels:
          severity: warning
          slo: order-service-availability
        annotations:
          summary: "Error budget burning fast (Warning)"
          description: |
            Order service is burning error budget at 6x rate.
            At this rate, monthly error budget will be exhausted in ~5 days.

      # ======================================================
      # Order Service: Latency SLO = 95% of requests < 200ms
      # ======================================================

      - alert: SLOLatencyFastBurn
        expr: |
          (
            1 - (
              sum(rate(http_request_duration_seconds_bucket{service="order-service", le="0.2"}[1h]))
              / sum(rate(http_request_duration_seconds_count{service="order-service"}[1h]))
            )
          ) > (14.4 * 0.05)
        for: 1m
        labels:
          severity: critical
          slo: order-service-latency
        annotations:
          summary: "Latency SLO error budget burning too fast"
          description: |
            {{ printf "%.1f" (mul $value 100) }}% of requests are taking > 200ms (target: 5%)

      # ======================================================
      # Payment Service: Availability SLO = 99.99%
      # Error budget: 4.32 min/month (0.01%)
      # ======================================================

      - alert: PaymentSLOCritical
        expr: |
          (
            sum(rate(http_requests_total{service="payment-service", status=~"5.."}[5m]))
            / sum(rate(http_requests_total{service="payment-service"}[5m]))
          ) > (36 * 0.0001)
        for: 2m
        labels:
          severity: critical
          slo: payment-service-availability
          page: "true"
        annotations:
          summary: "Payment service SLO critical breach"
          description: |
            Payment service error rate: {{ printf "%.4f" (mul $value 100) }}%
            SLO target: 99.99%
            Immediate action required!

      # ======================================================
      # Error Budget Remaining Alert
      # ======================================================

      - alert: ErrorBudgetLow
        expr: |
          (
            1 - (
              sum(increase(http_requests_total{service="order-service", status=~"5.."}[30d]))
              / sum(increase(http_requests_total{service="order-service"}[30d]))
            ) / 0.001
          ) < 0.1
        labels:
          severity: warning
          slo: order-service-availability
        annotations:
          summary: "Less than 10% error budget remaining"
          description: |
            Order service has {{ printf "%.1f" (mul $value 100) }}% of monthly error budget remaining.
            Consider freezing deployments.

      # ======================================================
      # Alerting on SLO Recovery
      # ======================================================
      - alert: ErrorBudgetRecovered
        expr: |
          (
            sum(rate(http_requests_total{service="order-service", status=~"5.."}[1h]))
            / sum(rate(http_requests_total{service="order-service"}[1h]))
          ) < (0.001 / 2)
        for: 10m
        labels:
          severity: info
          slo: order-service-availability
        annotations:
          summary: "SLO error rate back to normal"
          description: "Order service error rate is below 0.05% for 10 minutes"
```

---

## 5. PagerDuty Integration

```typescript
// incident/pagerduty-client.ts
import axios, { AxiosInstance } from 'axios';

interface PagerDutyIncident {
  id?: string;
  title: string;
  service: { id: string; type: 'service_reference' };
  urgency: 'high' | 'low';
  body?: {
    type: 'incident_body';
    details: string;
  };
  escalation_policy?: { id: string; type: 'escalation_policy_reference' };
}

interface EscalateOptions {
  incidentId: string;
  escalationLevel: number;
  note?: string;
}

interface IncidentNote {
  content: string;
}

class PagerDutyClient {
  private readonly client: AxiosInstance;
  private readonly fromEmail: string;

  constructor(
    private readonly apiToken: string,
    fromEmail: string = 'oncall@company.com'
  ) {
    this.fromEmail = fromEmail;
    this.client = axios.create({
      baseURL: 'https://api.pagerduty.com',
      headers: {
        Authorization: `Token token=${apiToken}`,
        Accept: 'application/vnd.pagerduty+json;version=2',
        'Content-Type': 'application/json',
        From: fromEmail,
      },
      timeout: 10000,
    });
  }

  // สร้าง incident ใหม่
  async createIncident(incident: PagerDutyIncident): Promise<{
    id: string;
    htmlUrl: string;
    status: string;
    incidentNumber: number;
  }> {
    try {
      const response = await this.client.post('/incidents', {
        incident: {
          type: 'incident',
          title: incident.title,
          service: incident.service,
          urgency: incident.urgency,
          body: incident.body,
          escalation_policy: incident.escalation_policy,
        },
      });

      const data = response.data.incident;
      console.log(`PagerDuty incident created: ${data.id}`);

      return {
        id: data.id,
        htmlUrl: data.html_url,
        status: data.status,
        incidentNumber: data.incident_number,
      };
    } catch (error: any) {
      throw new Error(`Failed to create PagerDuty incident: ${error.response?.data?.error || error.message}`);
    }
  }

  // Escalate incident
  async escalateIncident(options: EscalateOptions): Promise<void> {
    try {
      await this.client.put(`/incidents/${options.incidentId}`, {
        incident: {
          type: 'incident_reference',
          escalation_level: options.escalationLevel,
          ...(options.note && {
            body: { type: 'incident_body', details: options.note },
          }),
        },
      });

      // เพิ่ม note
      if (options.note) {
        await this.addNote(options.incidentId, {
          content: `Escalated to level ${options.escalationLevel}: ${options.note}`,
        });
      }

      console.log(`Incident ${options.incidentId} escalated to level ${options.escalationLevel}`);
    } catch (error: any) {
      throw new Error(`Failed to escalate incident: ${error.response?.data?.error || error.message}`);
    }
  }

  // Resolve incident
  async resolveIncident(
    incidentId: string,
    resolution: string
  ): Promise<void> {
    try {
      await this.client.put(`/incidents/${incidentId}`, {
        incident: {
          type: 'incident_reference',
          status: 'resolved',
          resolution: resolution,
        },
      });

      await this.addNote(incidentId, {
        content: `Incident resolved: ${resolution}`,
      });

      console.log(`Incident ${incidentId} resolved`);
    } catch (error: any) {
      throw new Error(`Failed to resolve incident: ${error.response?.data?.error || error.message}`);
    }
  }

  // เพิ่ม note ใน incident
  async addNote(incidentId: string, note: IncidentNote): Promise<void> {
    try {
      await this.client.post(`/incidents/${incidentId}/notes`, {
        note: { content: note.content },
      });
    } catch (error: any) {
      console.error(`Failed to add note: ${error.message}`);
    }
  }

  // Get incident details
  async getIncident(incidentId: string): Promise<any> {
    const response = await this.client.get(`/incidents/${incidentId}`);
    return response.data.incident;
  }

  // List on-call users
  async getOnCallUsers(scheduleId: string): Promise<Array<{
    id: string;
    name: string;
    email: string;
  }>> {
    const response = await this.client.get('/oncalls', {
      params: {
        schedule_ids: [scheduleId],
        'include[]': ['users'],
      },
    });

    return response.data.oncalls.map((oncall: any) => ({
      id: oncall.user.id,
      name: oncall.user.name,
      email: oncall.user.email,
    }));
  }

  // Acknowledge incident
  async acknowledgeIncident(incidentId: string): Promise<void> {
    await this.client.put(`/incidents/${incidentId}`, {
      incident: {
        type: 'incident_reference',
        status: 'acknowledged',
      },
    });
    console.log(`Incident ${incidentId} acknowledged`);
  }
}

// ตัวอย่างการใช้งาน
async function demonstratePagerDuty() {
  const pd = new PagerDutyClient(
    process.env.PAGERDUTY_API_TOKEN || 'test-token',
    'oncall@company.com'
  );

  try {
    // สร้าง incident
    const incident = await pd.createIncident({
      title: '[SEV1] Database connection exhaustion - Production',
      service: { id: 'SERVICE_ID_HERE', type: 'service_reference' },
      urgency: 'high',
      body: {
        type: 'incident_body',
        details: 'DB connections at 95%. Order service returning 500 errors.',
      },
    });

    console.log(`Created incident: #${incident.incidentNumber}`);
    console.log(`View at: ${incident.htmlUrl}`);

    // Acknowledge
    await pd.acknowledgeIncident(incident.id);

    // Escalate ถ้าไม่มีการตอบสนองภายใน 5 นาที
    setTimeout(async () => {
      await pd.escalateIncident({
        incidentId: incident.id,
        escalationLevel: 2,
        note: 'No response after 5 minutes, escalating to L2',
      });
    }, 5 * 60 * 1000);

    // Resolve
    await pd.resolveIncident(
      incident.id,
      'Killed idle connections and scaled down report-service. DB connections back to 65%.'
    );
  } catch (error) {
    console.error('PagerDuty error:', error);
  }
}
```

---

## 6. Incident Communication: Slack Incident Bot

```typescript
// incident/slack-incident-bot.ts
import axios from 'axios';

interface IncidentState {
  id: string;
  title: string;
  severity: string;
  status: 'open' | 'investigating' | 'mitigating' | 'resolved';
  commander: string;
  channelId?: string;
  startTime: Date;
  lastUpdate: Date;
  updates: Array<{
    timestamp: Date;
    author: string;
    message: string;
  }>;
}

class SlackIncidentBot {
  private incidents: Map<string, IncidentState> = new Map();

  constructor(
    private readonly webhookUrl: string,
    private readonly botToken: string,
    private readonly defaultChannel: string = '#incidents'
  ) {}

  private async postMessage(
    channel: string,
    blocks: unknown[],
    text?: string
  ): Promise<{ ok: boolean; ts?: string; channel?: string }> {
    try {
      const response = await axios.post(
        'https://slack.com/api/chat.postMessage',
        {
          channel,
          blocks,
          text: text || 'Incident Update',
          unfurl_links: false,
        },
        {
          headers: {
            Authorization: `Bearer ${this.botToken}`,
            'Content-Type': 'application/json',
          },
        }
      );
      return response.data;
    } catch (error: any) {
      console.error('Slack post failed:', error.message);
      return { ok: false };
    }
  }

  // เปิด war room สำหรับ incident
  async openWarRoom(incident: Omit<IncidentState, 'channelId' | 'lastUpdate' | 'updates'>): Promise<string> {
    const state: IncidentState = {
      ...incident,
      channelId: this.defaultChannel,
      lastUpdate: new Date(),
      updates: [],
    };

    this.incidents.set(incident.id, state);

    const blocks = [
      {
        type: 'header',
        text: {
          type: 'plain_text',
          text: `🚨 ${incident.severity} Incident: ${incident.id}`,
        },
      },
      {
        type: 'section',
        fields: [
          { type: 'mrkdwn', text: `*Title:*\n${incident.title}` },
          { type: 'mrkdwn', text: `*Severity:*\n${incident.severity}` },
          { type: 'mrkdwn', text: `*Status:*\n${incident.status.toUpperCase()}` },
          { type: 'mrkdwn', text: `*Commander:*\n<@${incident.commander}>` },
          { type: 'mrkdwn', text: `*Started:*\n${incident.startTime.toLocaleString('th-TH')}` },
        ],
      },
      {
        type: 'divider',
      },
      {
        type: 'section',
        text: {
          type: 'mrkdwn',
          text: `*📋 Runbook:* <https://runbooks.company.com/${incident.id}|View Runbook>`,
        },
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
          {
            type: 'button',
            text: { type: 'plain_text', text: 'View Dashboard' },
            url: `https://grafana.company.com/incident/${incident.id}`,
          },
        ],
      },
    ];

    const result = await this.postMessage(
      this.defaultChannel,
      blocks,
      `New ${incident.severity} incident: ${incident.title}`
    );

    if (result.ok) {
      console.log(`War room opened for incident ${incident.id}`);
    }

    return incident.id;
  }

  // ส่ง status update
  async sendStatusUpdate(
    incidentId: string,
    update: string,
    author: string,
    newStatus?: IncidentState['status']
  ): Promise<void> {
    const incident = this.incidents.get(incidentId);
    if (!incident) throw new Error(`Incident ${incidentId} not found`);

    incident.updates.push({
      timestamp: new Date(),
      author,
      message: update,
    });

    if (newStatus) {
      incident.status = newStatus;
    }

    incident.lastUpdate = new Date();

    const statusEmoji: Record<IncidentState['status'], string> = {
      open: '🔴',
      investigating: '🟡',
      mitigating: '🔵',
      resolved: '🟢',
    };

    const blocks = [
      {
        type: 'section',
        text: {
          type: 'mrkdwn',
          text: `${statusEmoji[incident.status]} *[${incidentId}] Status Update*`,
        },
      },
      {
        type: 'section',
        fields: [
          { type: 'mrkdwn', text: `*Status:*\n${incident.status.toUpperCase()}` },
          { type: 'mrkdwn', text: `*Updated by:*\n<@${author}>` },
          { type: 'mrkdwn', text: `*Time:*\n${new Date().toLocaleString('th-TH')}` },
          {
            type: 'mrkdwn',
            text: `*Duration:*\n${Math.round((Date.now() - incident.startTime.getTime()) / 60000)} min`,
          },
        ],
      },
      {
        type: 'section',
        text: {
          type: 'mrkdwn',
          text: `*Update:*\n${update}`,
        },
      },
    ];

    await this.postMessage(incident.channelId || this.defaultChannel, blocks);
    console.log(`Status update sent for incident ${incidentId}`);
  }

  // ปิด incident
  async closeIncident(
    incidentId: string,
    resolution: string,
    author: string
  ): Promise<void> {
    const incident = this.incidents.get(incidentId);
    if (!incident) throw new Error(`Incident ${incidentId} not found`);

    incident.status = 'resolved';
    incident.lastUpdate = new Date();

    const duration = Math.round(
      (Date.now() - incident.startTime.getTime()) / 60000
    );

    const blocks = [
      {
        type: 'header',
        text: {
          type: 'plain_text',
          text: `✅ Incident Resolved: ${incidentId}`,
        },
      },
      {
        type: 'section',
        fields: [
          { type: 'mrkdwn', text: `*Title:*\n${incident.title}` },
          { type: 'mrkdwn', text: `*Total Duration:*\n${duration} minutes` },
          { type: 'mrkdwn', text: `*Resolved by:*\n<@${author}>` },
          { type: 'mrkdwn', text: `*Updates sent:*\n${incident.updates.length}` },
        ],
      },
      {
        type: 'section',
        text: {
          type: 'mrkdwn',
          text: `*Resolution:*\n${resolution}`,
        },
      },
      {
        type: 'section',
        text: {
          type: 'mrkdwn',
          text: `📝 Post-mortem deadline: ${new Date(Date.now() + 5 * 24 * 60 * 60 * 1000).toLocaleDateString('th-TH')}`,
        },
      },
    ];

    await this.postMessage(incident.channelId || this.defaultChannel, blocks);
    console.log(`Incident ${incidentId} closed after ${duration} minutes`);
  }
}
```

---

## 7. Root Cause Analysis: 5 Whys

```typescript
// incident/rca-analyzer.ts

interface WhyAnswer {
  why: string;
  answer: string;
  evidence: string[];
  level: number;
}

interface RCAResult {
  incidentId: string;
  immediateSymptom: string;
  rootCauses: string[];
  whyChain: WhyAnswer[];
  contributingFactors: string[];
  systemicIssues: string[];
  recommendations: string[];
  confidence: 'high' | 'medium' | 'low';
}

class RCAAnalyzer {
  async performFiveWhys(
    incidentId: string,
    immediateSymptom: string,
    answers: Omit<WhyAnswer, 'level'>[]
  ): Promise<RCAResult> {
    const whyChain: WhyAnswer[] = answers.map((a, index) => ({
      ...a,
      level: index + 1,
    }));

    // ระบุ root causes (สาเหตุสุดท้าย)
    const rootCauses: string[] = [];
    const contributingFactors: string[] = [];
    const systemicIssues: string[] = [];

    // วิเคราะห์ pattern ของ root causes
    for (const why of whyChain) {
      const answer = why.answer.toLowerCase();

      // Process/People issues
      if (
        answer.includes('no test') ||
        answer.includes('ไม่มี test') ||
        answer.includes('no review') ||
        answer.includes('ไม่มี review')
      ) {
        systemicIssues.push(`Process gap: ${why.answer}`);
      }

      // Technical debt
      if (
        answer.includes('legacy') ||
        answer.includes('technical debt') ||
        answer.includes('หนี้เทคนิค')
      ) {
        systemicIssues.push(`Technical debt: ${why.answer}`);
      }

      // Missing monitoring
      if (
        answer.includes('no alert') ||
        answer.includes('no monitoring') ||
        answer.includes('ไม่มี alert')
      ) {
        contributingFactors.push(`Missing observability: ${why.answer}`);
      }
    }

    // Root cause = สาเหตุสุดท้ายใน why chain
    if (whyChain.length > 0) {
      rootCauses.push(whyChain[whyChain.length - 1].answer);
    }

    // สร้าง recommendations
    const recommendations = this.generateRecommendations(
      rootCauses,
      contributingFactors,
      systemicIssues
    );

    // ประเมิน confidence
    const confidence = whyChain.every((w) => w.evidence.length > 0) ? 'high' :
      whyChain.some((w) => w.evidence.length > 0) ? 'medium' : 'low';

    return {
      incidentId,
      immediateSymptom,
      rootCauses,
      whyChain,
      contributingFactors,
      systemicIssues,
      recommendations,
      confidence,
    };
  }

  private generateRecommendations(
    rootCauses: string[],
    contributingFactors: string[],
    systemicIssues: string[]
  ): string[] {
    const recommendations: string[] = [];

    // Root cause recommendations
    for (const cause of rootCauses) {
      if (cause.includes('connection') || cause.includes('pool')) {
        recommendations.push('Implement connection pool monitoring และ alerting');
        recommendations.push('เพิ่ม connection timeout ที่ database และ application level');
      }
      if (cause.includes('test') || cause.includes('ทดสอบ')) {
        recommendations.push('เพิ่ม test coverage requirement ใน code review checklist');
        recommendations.push('Implement integration tests สำหรับ critical paths');
      }
    }

    // Contributing factors recommendations
    for (const factor of contributingFactors) {
      if (factor.includes('monitoring') || factor.includes('alert')) {
        recommendations.push('ตรวจสอบและเพิ่ม monitoring coverage สำหรับ critical resources');
      }
    }

    // Systemic recommendations
    if (systemicIssues.length > 0) {
      recommendations.push('จัดทำ architecture review สำหรับ systemic issues ที่พบ');
      recommendations.push('เพิ่ม runbook สำหรับ scenarios ที่พบบ่อย');
    }

    return [...new Set(recommendations)]; // Remove duplicates
  }

  generateReport(rca: RCAResult): string {
    const lines: string[] = [
      `# Root Cause Analysis Report`,
      `**Incident:** ${rca.incidentId}`,
      `**Confidence:** ${rca.confidence.toUpperCase()}`,
      '',
      '## Immediate Symptom',
      rca.immediateSymptom,
      '',
      '## 5 Whys Chain',
    ];

    for (const why of rca.whyChain) {
      lines.push(`\n**Why ${why.level}:** ${why.why}`);
      lines.push(`**Answer:** ${why.answer}`);
      if (why.evidence.length > 0) {
        lines.push(`**Evidence:**`);
        why.evidence.forEach((e) => lines.push(`- ${e}`));
      }
    }

    lines.push('', '## Root Causes');
    rca.rootCauses.forEach((rc) => lines.push(`- ${rc}`));

    if (rca.contributingFactors.length > 0) {
      lines.push('', '## Contributing Factors');
      rca.contributingFactors.forEach((cf) => lines.push(`- ${cf}`));
    }

    if (rca.systemicIssues.length > 0) {
      lines.push('', '## Systemic Issues');
      rca.systemicIssues.forEach((si) => lines.push(`- ${si}`));
    }

    lines.push('', '## Recommendations');
    rca.recommendations.forEach((rec) => lines.push(`- ${rec}`));

    return lines.join('\n');
  }
}
```

---

## 8. Chaos Engineering for Preparedness

```typescript
// incident/chaos-runner.ts
import axios from 'axios';

interface ChaosExperiment {
  id: string;
  name: string;
  description: string;
  targetService: string;
  type: 'network-latency' | 'service-kill' | 'cpu-stress' | 'memory-stress' | 'disk-full' | 'packet-loss';
  parameters: Record<string, unknown>;
  duration: number; // seconds
  hypothesis: string;
  expectedBehavior: string;
}

interface ChaosResult {
  experimentId: string;
  success: boolean;
  hypothesis: string;
  actualBehavior: string;
  metrics: {
    errorRateBefore: number;
    errorRateDuring: number;
    errorRateAfter: number;
    latencyBefore: number;
    latencyDuring: number;
    latencyAfter: number;
    recoveryTimeSeconds: number;
  };
  issues: string[];
  recommendations: string[];
}

class ChaosRunner {
  private readonly chaosMonkeyUrl: string;

  constructor(chaosMonkeyUrl: string = 'http://chaos-monkey:8080') {
    this.chaosMonkeyUrl = chaosMonkeyUrl;
  }

  async runExperiment(experiment: ChaosExperiment): Promise<ChaosResult> {
    console.log(`\n[Chaos] Starting experiment: ${experiment.name}`);
    console.log(`[Chaos] Hypothesis: ${experiment.hypothesis}`);

    // Collect baseline metrics
    const baselineMetrics = await this.collectMetrics(experiment.targetService);

    // Run experiment
    const experimentId = await this.injectFault(experiment);

    // Wait for duration
    await new Promise((resolve) => setTimeout(resolve, experiment.duration * 1000));

    // Collect during-experiment metrics
    const duringMetrics = await this.collectMetrics(experiment.targetService);

    // Stop fault injection
    await this.stopFault(experimentId);

    // Wait for recovery
    const recoveryStart = Date.now();
    await this.waitForRecovery(experiment.targetService, baselineMetrics);
    const recoveryTimeSeconds = (Date.now() - recoveryStart) / 1000;

    // Collect post-experiment metrics
    const afterMetrics = await this.collectMetrics(experiment.targetService);

    // Analyze results
    const issues: string[] = [];
    const recommendations: string[] = [];

    if (duringMetrics.errorRate > 5) {
      issues.push(`Error rate สูงมาก (${duringMetrics.errorRate.toFixed(1)}%) ระหว่าง experiment`);
    }

    if (recoveryTimeSeconds > 60) {
      issues.push(`Recovery time นานเกินไป: ${recoveryTimeSeconds.toFixed(0)} seconds`);
      recommendations.push('ปรับ health check intervals และ circuit breaker thresholds');
    }

    if (afterMetrics.errorRate > baselineMetrics.errorRate * 1.1) {
      issues.push('ระบบยังไม่ fully recovered');
      recommendations.push('ตรวจสอบ state cleanup ใน service');
    }

    const success = issues.length === 0;

    return {
      experimentId,
      success,
      hypothesis: experiment.hypothesis,
      actualBehavior: success
        ? 'ระบบทำงานได้ตาม hypothesis'
        : `พบ issues: ${issues.join('; ')}`,
      metrics: {
        errorRateBefore: baselineMetrics.errorRate,
        errorRateDuring: duringMetrics.errorRate,
        errorRateAfter: afterMetrics.errorRate,
        latencyBefore: baselineMetrics.latency,
        latencyDuring: duringMetrics.latency,
        latencyAfter: afterMetrics.latency,
        recoveryTimeSeconds,
      },
      issues,
      recommendations,
    };
  }

  private async injectFault(experiment: ChaosExperiment): Promise<string> {
    const response = await axios.post(`${this.chaosMonkeyUrl}/faults`, {
      type: experiment.type,
      target: experiment.targetService,
      parameters: experiment.parameters,
      duration: experiment.duration,
    });
    return response.data.faultId;
  }

  private async stopFault(faultId: string): Promise<void> {
    await axios.delete(`${this.chaosMonkeyUrl}/faults/${faultId}`);
  }

  private async collectMetrics(
    serviceName: string
  ): Promise<{ errorRate: number; latency: number }> {
    // จำลองการเก็บ metrics จาก Prometheus
    return {
      errorRate: Math.random() * 2, // 0-2% baseline
      latency: 50 + Math.random() * 50, // 50-100ms baseline
    };
  }

  private async waitForRecovery(
    serviceName: string,
    baseline: { errorRate: number; latency: number }
  ): Promise<void> {
    const maxWaitMs = 300000; // 5 minutes
    const checkIntervalMs = 5000;
    const startTime = Date.now();

    while (Date.now() - startTime < maxWaitMs) {
      const current = await this.collectMetrics(serviceName);

      if (
        current.errorRate <= baseline.errorRate * 1.05 &&
        current.latency <= baseline.latency * 1.1
      ) {
        console.log(`[Chaos] Service recovered after ${(Date.now() - startTime) / 1000}s`);
        return;
      }

      await new Promise((resolve) => setTimeout(resolve, checkIntervalMs));
    }

    console.warn(`[Chaos] Service did not fully recover within ${maxWaitMs / 1000}s`);
  }
}
```

---

## 9. On-call Rotation Scheduler

```typescript
// incident/oncall-scheduler.ts

interface Engineer {
  id: string;
  name: string;
  email: string;
  timezone: string;
  skills: string[];
  maxConsecutiveWeeks: number;
}

interface OncallShift {
  engineerId: string;
  startDate: Date;
  endDate: Date;
  isBackup: boolean;
  week: number;
}

interface OncallSchedule {
  startDate: Date;
  endDate: Date;
  shifts: OncallShift[];
  metadata: {
    totalEngineers: number;
    weeksPerEngineer: number;
    fairnessScore: number;
  };
}

class OncallScheduler {
  // สร้าง schedule ด้วย fairness algorithm
  createSchedule(
    engineers: Engineer[],
    startDate: Date,
    weeks: number
  ): OncallSchedule {
    const shifts: OncallShift[] = [];
    const weekCount = new Map<string, number>();
    const recentOncall = new Set<string>(); // engineers ที่เพิ่งออก on-call

    engineers.forEach((e) => weekCount.set(e.id, 0));

    for (let week = 0; week < weeks; week++) {
      const shiftStart = new Date(startDate);
      shiftStart.setDate(shiftStart.getDate() + week * 7);
      const shiftEnd = new Date(shiftStart);
      shiftEnd.setDate(shiftEnd.getDate() + 7);

      // เลือก primary on-call
      const primaryEngineer = this.selectNextEngineer(
        engineers,
        weekCount,
        recentOncall,
        shifts
      );

      // เลือก backup on-call
      const remainingEngineers = engineers.filter((e) => e.id !== primaryEngineer.id);
      const backupEngineer = this.selectNextEngineer(
        remainingEngineers,
        weekCount,
        recentOncall,
        shifts
      );

      shifts.push({
        engineerId: primaryEngineer.id,
        startDate: new Date(shiftStart),
        endDate: new Date(shiftEnd),
        isBackup: false,
        week,
      });

      shifts.push({
        engineerId: backupEngineer.id,
        startDate: new Date(shiftStart),
        endDate: new Date(shiftEnd),
        isBackup: true,
        week,
      });

      // อัพเดต counts
      weekCount.set(primaryEngineer.id, (weekCount.get(primaryEngineer.id) || 0) + 1);
      weekCount.set(backupEngineer.id, (weekCount.get(backupEngineer.id) || 0) + 0.5);

      // Track recent on-call (ป้องกัน back-to-back)
      recentOncall.clear();
      if (week > 0) {
        recentOncall.add(primaryEngineer.id);
      }
    }

    // คำนวณ fairness score
    const counts = Array.from(weekCount.values()).filter((v) => v > 0);
    const avg = counts.reduce((a, b) => a + b, 0) / counts.length;
    const variance = counts.reduce((sum, v) => sum + Math.pow(v - avg, 2), 0) / counts.length;
    const fairnessScore = Math.max(0, 1 - Math.sqrt(variance) / avg);

    const endDate = new Date(startDate);
    endDate.setDate(endDate.getDate() + weeks * 7);

    return {
      startDate,
      endDate,
      shifts,
      metadata: {
        totalEngineers: engineers.length,
        weeksPerEngineer: Math.round((weeks / engineers.length) * 10) / 10,
        fairnessScore: Math.round(fairnessScore * 100) / 100,
      },
    };
  }

  private selectNextEngineer(
    engineers: Engineer[],
    weekCount: Map<string, number>,
    recentOncall: Set<string>,
    existingShifts: OncallShift[]
  ): Engineer {
    // Priority: engineer ที่ไม่เพิ่งออก on-call และมีจำนวน weeks น้อยที่สุด
    const available = engineers.filter((e) => !recentOncall.has(e.id));
    const pool = available.length > 0 ? available : engineers;

    return pool.reduce((least, current) => {
      const leastCount = weekCount.get(least.id) || 0;
      const currentCount = weekCount.get(current.id) || 0;
      return currentCount < leastCount ? current : least;
    });
  }

  generateReport(schedule: OncallSchedule, engineers: Engineer[]): string {
    const engineerMap = new Map(engineers.map((e) => [e.id, e]));
    const lines: string[] = [
      '# On-Call Schedule',
      `Period: ${schedule.startDate.toLocaleDateString('th-TH')} - ${schedule.endDate.toLocaleDateString('th-TH')}`,
      `Fairness Score: ${(schedule.metadata.fairnessScore * 100).toFixed(1)}%`,
      '',
      '| Week | Primary | Backup | Start | End |',
      '|------|---------|--------|-------|-----|',
    ];

    const primaryShifts = schedule.shifts
      .filter((s) => !s.isBackup)
      .sort((a, b) => a.week - b.week);

    for (const shift of primaryShifts) {
      const backup = schedule.shifts.find(
        (s) => s.week === shift.week && s.isBackup
      );
      const primary = engineerMap.get(shift.engineerId);
      const backupEngineer = backup ? engineerMap.get(backup.engineerId) : null;

      lines.push(
        `| ${shift.week + 1} | ${primary?.name || 'Unknown'} | ${backupEngineer?.name || 'Unknown'} | ` +
          `${shift.startDate.toLocaleDateString('th-TH')} | ${shift.endDate.toLocaleDateString('th-TH')} |`
      );
    }

    return lines.join('\n');
  }
}
```

---

## 10. Blameless Post-mortem Guidelines

```typescript
// incident/blameless-guidelines.ts

interface BlamelessChecker {
  text: string;
  issues: string[];
  suggestions: string[];
  score: number; // 0-100
}

class BlamelessPostmortemChecker {
  // Patterns ที่บ่งบอกถึง blame culture
  private readonly blamePhrases = [
    { pattern: /\b(john|jane|bob|alice)\b.*(forgot|failed|mistake|wrong)/gi, issue: 'กล่าวโทษบุคคลโดยตรง' },
    { pattern: /should have known/gi, issue: 'สมมติว่าคนควรรู้อยู่แล้ว' },
    { pattern: /\bcareless(ly)?\b/gi, issue: 'ใช้คำว่า "ไม่ระวัง"' },
    { pattern: /\bnegligent\b/gi, issue: 'ใช้คำว่า "ละเลย"' },
    { pattern: /\bstupid(ly)?\b/gi, issue: 'ใช้คำดูถูก' },
    { pattern: /\bfailed to\b/gi, issue: 'ใช้ภาษาที่บ่งโทษ' },
    { pattern: /\bwas supposed to\b/gi, issue: 'บ่งบอกความผิดพลาดของบุคคล' },
  ];

  // System-focused phrases ที่ดี
  private readonly systemFocusPhrases = [
    'the system',
    'the process',
    'the configuration',
    'the alert',
    'the runbook',
    'the monitoring',
    'the deployment process',
    'the testing strategy',
  ];

  analyze(postmortemText: string): BlamelessChecker {
    const issues: string[] = [];
    const suggestions: string[] = [];
    let deductions = 0;

    // ตรวจสอบ blame phrases
    for (const { pattern, issue } of this.blamePhrases) {
      const matches = postmortemText.match(pattern);
      if (matches) {
        issues.push(`${issue}: "${matches[0]}"`);
        suggestions.push(
          `แทนที่ "${matches[0]}" ด้วยภาษาที่เน้น system/process`
        );
        deductions += 10;
      }
    }

    // ตรวจสอบว่ามี action items ที่ system-focused
    const actionItems = postmortemText.match(/action item.*/gi) || [];
    const personActions = actionItems.filter((ai) =>
      ai.match(/\b(john|jane|bob|alice)\s+(should|must|will|needs)\b/i)
    );

    if (personActions.length > actionItems.length * 0.5) {
      issues.push('Action items เน้นที่บุคคลมากกว่า system/process');
      suggestions.push(
        'เปลี่ยน action items เป็น "เพิ่ม X ใน system" แทน "John ต้องทำ X"'
      );
      deductions += 15;
    }

    // ตรวจสอบว่ามี contributing factors (บ่งบอกถึง systemic thinking)
    if (!postmortemText.includes('contributing factor')) {
      issues.push('ไม่มี contributing factors ที่ระบุชัดเจน');
      suggestions.push('เพิ่ม section "Contributing Factors" เพื่อแสดง systemic issues');
      deductions += 5;
    }

    // ตรวจสอบว่ามี "What went well"
    if (!postmortemText.match(/what went well/i)) {
      issues.push('ไม่มี "What went well" section');
      suggestions.push('เพิ่ม section สำหรับสิ่งที่ทำงานได้ดีเพื่อ balance perspective');
      deductions += 5;
    }

    const score = Math.max(0, 100 - deductions);

    return {
      text: postmortemText,
      issues,
      suggestions,
      score,
    };
  }

  generateGuidelines(): string {
    return `
# Blameless Post-mortem Guidelines

## หลักการ Blameless Culture

1. **เน้น System ไม่ใช่ People**
   - ❌ "John forgot to update the config"
   - ✅ "The deployment process lacked validation for config changes"

2. **ถือว่าทุกคนทำดีที่สุดแล้วด้วยข้อมูลที่มี**
   - People มักทำ decision ที่สมเหตุสมผลตาม context ในขณะนั้น
   - Focus ที่ why the system allowed the decision, ไม่ใช่ who made it

3. **ระบุ Contributing Factors ให้ครบ**
   - Technical debt, missing tooling, unclear documentation
   - High cognitive load, time pressure, unclear ownership

4. **Action Items ต้องเป็น Systemic**
   - ❌ "Bob ต้องระวังมากขึ้น"
   - ✅ "เพิ่ม automated validation ใน CI/CD pipeline"

5. **ฉลองสิ่งที่ทำงานได้ดี**
   - ยกย่องการ detect ที่รวดเร็ว, การ communicate ที่ดี
   - สร้าง psychological safety สำหรับการ report ในอนาคต

## Template Phrases ที่ดี

- "The system did not prevent..."
- "The process lacked..."
- "There was no automated check for..."
- "The runbook was unclear about..."
- "Monitoring did not cover..."
- "The alert threshold was set too high..."
`;
  }
}
```

---

## สรุป

| หัวข้อ | เนื้อหาสำคัญ |
|--------|-------------|
| Incident Lifecycle | 5 ขั้นตอน: Detection, Triage, Mitigation, Resolution, Post-mortem |
| Runbook Executor | Automated step execution พร้อม retry, rollback, และ decision points |
| Post-mortem Template | YAML template ครบถ้วนพร้อม timeline, root cause, และ action items |
| SLO-based Alerting | Multi-window multi-burn-rate alerts ด้วย Prometheus |
| PagerDuty Integration | TypeScript client สำหรับ create, escalate, resolve incidents |
| Slack Incident Bot | War room, status updates, และ resolution notifications |
| Root Cause Analysis | 5 Whys implementation พร้อม systemic issue identification |
| Chaos Engineering | Controlled fault injection สำหรับ preparedness testing |
| On-call Scheduler | Fairness algorithm สำหรับ rotation scheduling |
| Blameless Guidelines | Tools สำหรับตรวจสอบและส่งเสริม blameless culture |

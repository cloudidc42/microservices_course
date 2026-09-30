# Part 72: Production Readiness Checklist

## บทนำ

ก่อนที่จะ deploy microservice ขึ้น production ต้องตรวจสอบรายการสำคัญหลายด้าน ทั้ง Security, Performance, Reliability, Observability และ Documentation บทนี้ครอบคลุม Production Readiness Checklist ที่สมบูรณ์

---

## 1. Production Readiness Checklist

### Security Checklist

```typescript
// security-checklist.ts
// TypeScript interface สำหรับ security checks

interface SecurityCheck {
  category: string;
  item: string;
  status: 'PASS' | 'FAIL' | 'WARN' | 'NA';
  priority: 'CRITICAL' | 'HIGH' | 'MEDIUM' | 'LOW';
  notes?: string;
}

const securityChecklist: SecurityCheck[] = [
  // Authentication & Authorization
  {
    category: 'Authentication',
    item: 'JWT tokens expire within 15 minutes',
    status: 'PASS',
    priority: 'CRITICAL',
  },
  {
    category: 'Authentication',
    item: 'Refresh tokens are rotated on use',
    status: 'PASS',
    priority: 'CRITICAL',
  },
  {
    category: 'Authorization',
    item: 'All endpoints require authentication',
    status: 'PASS',
    priority: 'CRITICAL',
  },
  {
    category: 'Authorization',
    item: 'RBAC/ABAC implemented correctly',
    status: 'PASS',
    priority: 'HIGH',
  },
  // Data Security
  {
    category: 'Data Security',
    item: 'PII data encrypted at rest',
    status: 'PASS',
    priority: 'CRITICAL',
  },
  {
    category: 'Data Security',
    item: 'TLS 1.2+ for data in transit',
    status: 'PASS',
    priority: 'CRITICAL',
  },
  {
    category: 'Data Security',
    item: 'Secrets managed via vault (not env vars)',
    status: 'PASS',
    priority: 'CRITICAL',
  },
  // API Security
  {
    category: 'API Security',
    item: 'Rate limiting implemented',
    status: 'PASS',
    priority: 'HIGH',
  },
  {
    category: 'API Security',
    item: 'Input validation on all endpoints',
    status: 'PASS',
    priority: 'HIGH',
  },
  {
    category: 'API Security',
    item: 'SQL injection prevention',
    status: 'PASS',
    priority: 'CRITICAL',
  },
];

function generateSecurityReport(checks: SecurityCheck[]): void {
  const failed = checks.filter(c => c.status === 'FAIL');
  const critical = failed.filter(c => c.priority === 'CRITICAL');
  
  console.log('=== Security Report ===');
  console.log(`Total checks: ${checks.length}`);
  console.log(`Passed: ${checks.filter(c => c.status === 'PASS').length}`);
  console.log(`Failed: ${failed.length}`);
  console.log(`Critical failures: ${critical.length}`);
  
  if (critical.length > 0) {
    console.error('CRITICAL FAILURES - Cannot deploy to production!');
    critical.forEach(c => console.error(`  - ${c.category}: ${c.item}`));
  }
}
```

### Performance Checklist

```typescript
// performance-checklist.ts

interface PerformanceMetric {
  name: string;
  target: string;
  current?: string;
  status: 'PASS' | 'FAIL' | 'UNKNOWN';
}

const performanceChecklist: PerformanceMetric[] = [
  { name: 'API P95 latency', target: '<200ms', current: '150ms', status: 'PASS' },
  { name: 'API P99 latency', target: '<500ms', current: '450ms', status: 'PASS' },
  { name: 'Throughput', target: '>1000 RPS', current: '1500 RPS', status: 'PASS' },
  { name: 'Error rate', target: '<0.1%', current: '0.05%', status: 'PASS' },
  { name: 'DB query P95', target: '<100ms', current: '80ms', status: 'PASS' },
  { name: 'Memory usage', target: '<80%', current: '65%', status: 'PASS' },
  { name: 'CPU usage', target: '<70%', current: '45%', status: 'PASS' },
  { name: 'Connection pool utilization', target: '<80%', current: '60%', status: 'PASS' },
  { name: 'Cache hit rate', target: '>90%', current: '94%', status: 'PASS' },
  { name: 'GC pause time', target: '<50ms', current: '30ms', status: 'PASS' },
];
```

---

## 2. Runbook Creation

```typescript
// runbook-generator.ts
// สร้าง Runbook สำหรับ operational procedures

interface RunbookSection {
  title: string;
  content: string;
  steps?: string[];
  commands?: string[];
}

interface Runbook {
  serviceName: string;
  version: string;
  lastUpdated: Date;
  oncallContact: string;
  escalationPath: string[];
  sections: RunbookSection[];
}

const orderServiceRunbook: Runbook = {
  serviceName: 'Order Service',
  version: '2.1.0',
  lastUpdated: new Date('2024-01-01'),
  oncallContact: 'order-team@company.com',
  escalationPath: [
    'L1: On-call engineer (PagerDuty)',
    'L2: Order Team Lead',
    'L3: Engineering Manager',
    'L4: CTO',
  ],
  sections: [
    {
      title: 'Service Overview',
      content: `
        Order Service handles all order management operations including:
        - Order creation, update, and cancellation
        - Integration with Inventory and Payment services
        - Order status tracking and notifications
        
        Dependencies:
        - PostgreSQL (primary database)
        - Redis (caching and sessions)
        - Kafka (event streaming)
        - Inventory Service (HTTP)
        - Payment Service (HTTP)
      `,
    },
    {
      title: 'High Availability',
      content: 'Service runs with 3 replicas minimum in production',
      steps: [
        'Check current replica count: kubectl get deployment order-service -n production',
        'Scale if needed: kubectl scale deployment order-service --replicas=3 -n production',
        'Verify pods are running: kubectl get pods -l app=order-service -n production',
      ],
    },
    {
      title: 'Database Issues',
      content: 'Procedures for handling database connectivity issues',
      steps: [
        'Check DB connection: kubectl exec -it <pod> -- node -e "require(\'./db\').ping()"',
        'Check DB metrics in Grafana dashboard',
        'Check connection pool exhaustion in logs',
        'If primary is down: failover to replica using Route53 update',
      ],
      commands: [
        'kubectl logs -l app=order-service -n production --tail=100 | grep -i "db\\|database\\|connection"',
        'kubectl exec -it postgres-primary -- psql -U postgres -c "SELECT count(*) FROM pg_stat_activity"',
      ],
    },
    {
      title: 'High Memory Usage',
      content: 'Procedures when memory usage exceeds 85%',
      steps: [
        'Identify memory-heavy pods: kubectl top pods -l app=order-service -n production',
        'Check for memory leaks in logs',
        'Force GC if needed: kubectl exec -it <pod> -- kill -USR2 1',
        'Rolling restart if issue persists',
      ],
      commands: [
        'kubectl top pods -l app=order-service -n production',
        'kubectl rollout restart deployment/order-service -n production',
      ],
    },
    {
      title: 'Service Degradation',
      content: 'When error rate exceeds 1% or latency exceeds 500ms',
      steps: [
        'Check Datadog/Grafana for error patterns',
        'Check downstream service health',
        'Enable circuit breaker manually if needed',
        'Consider traffic reduction via load balancer',
        'Notify stakeholders if degradation > 5 minutes',
      ],
    },
  ],
};

function exportRunbookAsMarkdown(runbook: Runbook): string {
  let md = `# Runbook: ${runbook.serviceName}\n\n`;
  md += `**Version:** ${runbook.version}\n`;
  md += `**Last Updated:** ${runbook.lastUpdated.toISOString().split('T')[0]}\n`;
  md += `**On-call Contact:** ${runbook.oncallContact}\n\n`;
  
  md += `## Escalation Path\n`;
  runbook.escalationPath.forEach(path => {
    md += `- ${path}\n`;
  });
  md += '\n';

  for (const section of runbook.sections) {
    md += `## ${section.title}\n\n`;
    md += `${section.content.trim()}\n\n`;
    
    if (section.steps) {
      md += `### Steps\n`;
      section.steps.forEach((step, i) => {
        md += `${i + 1}. ${step}\n`;
      });
      md += '\n';
    }
    
    if (section.commands) {
      md += `### Commands\n\`\`\`bash\n`;
      section.commands.forEach(cmd => {
        md += `${cmd}\n`;
      });
      md += `\`\`\`\n\n`;
    }
  }
  
  return md;
}
```

---

## 3. On-call Rotation Setup

```yaml
# pagerduty-schedule.yaml
# Configuration for PagerDuty on-call schedule

schedules:
  - name: "Order Team On-call"
    description: "Primary on-call rotation for Order Service"
    timezone: "Asia/Bangkok"
    layers:
      - name: "Primary On-call"
        rotation_type: "weekly"
        start: "2024-01-01T09:00:00+07:00"
        users:
          - name: "Alice"
            email: "alice@company.com"
          - name: "Bob"
            email: "bob@company.com"
          - name: "Charlie"
            email: "charlie@company.com"
        restrictions:
          - type: "weekly_restriction"
            start_day_of_week: 1  # Monday
            start_time_of_day: "09:00:00"
            duration_seconds: 432000  # 5 days (weekdays only)

      - name: "Weekend On-call"
        rotation_type: "weekly"
        start: "2024-01-01T09:00:00+07:00"
        users:
          - name: "Alice"
            email: "alice@company.com"
          - name: "Bob"
            email: "bob@company.com"
        restrictions:
          - type: "weekly_restriction"
            start_day_of_week: 6  # Saturday
            start_time_of_day: "00:00:00"
            duration_seconds: 172800  # 2 days

escalation_policies:
  - name: "Order Service Escalation"
    rules:
      - escalation_delay_in_minutes: 5
        targets:
          - type: "schedule"
            name: "Order Team On-call"
      - escalation_delay_in_minutes: 15
        targets:
          - type: "user"
            name: "Engineering Manager"
      - escalation_delay_in_minutes: 30
        targets:
          - type: "user"
            name: "CTO"
```

```typescript
// oncall-notification.ts
// ระบบแจ้งเตือน on-call

interface Incident {
  id: string;
  title: string;
  severity: 'P1' | 'P2' | 'P3' | 'P4';
  service: string;
  description: string;
  detectedAt: Date;
  assignedTo?: string;
}

interface AlertRule {
  name: string;
  condition: string;
  severity: 'P1' | 'P2' | 'P3' | 'P4';
  channels: string[];
  autoResolveAfterMinutes?: number;
}

const alertRules: AlertRule[] = [
  {
    name: 'High Error Rate',
    condition: 'error_rate > 5%',
    severity: 'P1',
    channels: ['pagerduty', 'slack-critical', 'email'],
    autoResolveAfterMinutes: 60,
  },
  {
    name: 'Service Down',
    condition: 'health_check_failed for 2 minutes',
    severity: 'P1',
    channels: ['pagerduty', 'slack-critical', 'phone'],
  },
  {
    name: 'High Latency',
    condition: 'p95_latency > 1000ms for 5 minutes',
    severity: 'P2',
    channels: ['pagerduty', 'slack-warnings'],
    autoResolveAfterMinutes: 120,
  },
  {
    name: 'Database Connection Pool',
    condition: 'db_connection_pool_usage > 90%',
    severity: 'P2',
    channels: ['slack-warnings', 'email'],
  },
  {
    name: 'Memory Warning',
    condition: 'memory_usage > 85%',
    severity: 'P3',
    channels: ['slack-info'],
  },
];

class IncidentManager {
  private incidents: Map<string, Incident> = new Map();

  async createIncident(title: string, severity: Incident['severity'], service: string, description: string): Promise<Incident> {
    const incident: Incident = {
      id: crypto.randomUUID(),
      title,
      severity,
      service,
      description,
      detectedAt: new Date(),
    };

    this.incidents.set(incident.id, incident);

    // Notify based on severity
    await this.notifyTeam(incident);

    return incident;
  }

  private async notifyTeam(incident: Incident): Promise<void> {
    switch (incident.severity) {
      case 'P1':
        await this.callOnCall(incident);
        await this.sendSlackAlert(incident, '#incidents-critical');
        break;
      case 'P2':
        await this.pageOnCall(incident);
        await this.sendSlackAlert(incident, '#incidents-major');
        break;
      case 'P3':
        await this.sendSlackAlert(incident, '#incidents-minor');
        break;
      case 'P4':
        await this.sendSlackAlert(incident, '#incidents-info');
        break;
    }
  }

  private async callOnCall(incident: Incident): Promise<void> {
    console.log(`PHONE CALL: P1 Incident - ${incident.title}`);
    // Integrate with PagerDuty API
  }

  private async pageOnCall(incident: Incident): Promise<void> {
    console.log(`PAGER: P2 Incident - ${incident.title}`);
  }

  private async sendSlackAlert(incident: Incident, channel: string): Promise<void> {
    const message = {
      channel,
      text: `🚨 ${incident.severity} Incident: ${incident.title}`,
      blocks: [
        {
          type: 'header',
          text: { type: 'plain_text', text: `${incident.severity} Incident` },
        },
        {
          type: 'section',
          fields: [
            { type: 'mrkdwn', text: `*Service:* ${incident.service}` },
            { type: 'mrkdwn', text: `*Detected:* ${incident.detectedAt.toISOString()}` },
            { type: 'mrkdwn', text: `*Description:* ${incident.description}` },
          ],
        },
      ],
    };

    console.log('Slack alert:', JSON.stringify(message, null, 2));
  }
}
```

---

## 4. Alerting and Incident Response

```yaml
# prometheus-alerts.yaml
# Prometheus alerting rules สำหรับ production

groups:
  - name: order-service-alerts
    rules:
      # P1: Service completely down
      - alert: OrderServiceDown
        expr: up{job="order-service"} == 0
        for: 2m
        labels:
          severity: critical
          team: order-team
          priority: P1
        annotations:
          summary: "Order Service is down"
          description: "Order Service has been unavailable for more than 2 minutes"
          runbook: "https://wiki.company.com/runbooks/order-service"
          dashboard: "https://grafana.company.com/d/order-service"

      # P1: High error rate
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{job="order-service",status=~"5.."}[5m]))
          /
          sum(rate(http_requests_total{job="order-service"}[5m]))
          > 0.05
        for: 5m
        labels:
          severity: critical
          priority: P1
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value | humanizePercentage }} (threshold: 5%)"

      # P2: High latency
      - alert: HighLatency
        expr: |
          histogram_quantile(0.95,
            sum(rate(http_request_duration_seconds_bucket{job="order-service"}[5m])) by (le)
          ) > 0.5
        for: 5m
        labels:
          severity: warning
          priority: P2
        annotations:
          summary: "High API latency"
          description: "P95 latency is {{ $value | humanizeDuration }} (threshold: 500ms)"

      # P2: Database connection pool exhaustion
      - alert: DBConnectionPoolHigh
        expr: pg_stat_database_numbackends / pg_settings_max_connections > 0.9
        for: 3m
        labels:
          severity: warning
          priority: P2
        annotations:
          summary: "Database connection pool near exhaustion"
          description: "Connection pool is {{ $value | humanizePercentage }} utilized"

      # P3: Memory usage high
      - alert: HighMemoryUsage
        expr: container_memory_usage_bytes{pod=~"order-service.*"} / container_spec_memory_limit_bytes > 0.85
        for: 10m
        labels:
          severity: warning
          priority: P3
        annotations:
          summary: "High memory usage"
          description: "Memory usage is {{ $value | humanizePercentage }}"

      # P3: Disk space low
      - alert: LowDiskSpace
        expr: kubelet_volume_stats_available_bytes / kubelet_volume_stats_capacity_bytes < 0.1
        for: 5m
        labels:
          severity: warning
          priority: P3
        annotations:
          summary: "Low disk space"
          description: "Only {{ $value | humanizePercentage }} disk space remaining"

  - name: slo-alerts
    rules:
      # SLO burn rate alert
      - alert: SLOErrorBudgetBurnRate
        expr: |
          (
            sum(rate(http_requests_total{job="order-service",status=~"5.."}[1h]))
            /
            sum(rate(http_requests_total{job="order-service"}[1h]))
          ) > 14 * (1 - 0.999)
        for: 5m
        labels:
          severity: critical
          priority: P1
        annotations:
          summary: "SLO error budget burning too fast"
          description: "Error budget is burning at 14x the expected rate"
```

---

## 5. SLO/SLA Definition and Tracking

```typescript
// slo-tracker.ts

interface SLO {
  name: string;
  description: string;
  target: number; // percentage (e.g., 99.9)
  window: '30d' | '7d' | '1d';
  errorBudget: number; // minutes of allowed downtime
  indicators: SLI[];
}

interface SLI {
  name: string;
  query: string; // Prometheus query
  goodEventQuery: string;
  totalEventQuery: string;
}

const orderServiceSLOs: SLO[] = [
  {
    name: 'Availability SLO',
    description: 'Percentage of time Order Service is available',
    target: 99.9,
    window: '30d',
    errorBudget: 43.2, // 43.2 minutes per month
    indicators: [
      {
        name: 'Service Health',
        query: 'avg(up{job="order-service"})',
        goodEventQuery: 'sum(rate(http_requests_total{job="order-service",status!~"5.."}[5m]))',
        totalEventQuery: 'sum(rate(http_requests_total{job="order-service"}[5m]))',
      },
    ],
  },
  {
    name: 'Latency SLO',
    description: '95% of requests complete within 200ms',
    target: 95.0,
    window: '30d',
    errorBudget: 1440, // 24 hours per month of slow requests
    indicators: [
      {
        name: 'Request Latency',
        query: 'histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket{job="order-service"}[5m])) by (le))',
        goodEventQuery: 'sum(rate(http_request_duration_seconds_bucket{job="order-service",le="0.2"}[5m]))',
        totalEventQuery: 'sum(rate(http_request_duration_seconds_count{job="order-service"}[5m]))',
      },
    ],
  },
  {
    name: 'Order Success Rate SLO',
    description: '99.5% of order creation attempts succeed',
    target: 99.5,
    window: '30d',
    errorBudget: 216, // 3.6 hours per month
    indicators: [
      {
        name: 'Order Creation Success',
        query: 'sum(rate(order_creation_total{status="success"}[5m])) / sum(rate(order_creation_total[5m]))',
        goodEventQuery: 'sum(rate(order_creation_total{status="success"}[5m]))',
        totalEventQuery: 'sum(rate(order_creation_total[5m]))',
      },
    ],
  },
];

class SLOTracker {
  async calculateCurrentSLO(slo: SLO): Promise<{
    slo: SLO;
    current: number;
    errorBudgetRemaining: number;
    burnRate: number;
    status: 'HEALTHY' | 'AT_RISK' | 'BREACHED';
  }> {
    // Fetch metrics from Prometheus
    const current = await this.fetchMetric(slo);
    const errorBudgetConsumed = Math.max(0, slo.target - current);
    const errorBudgetRemaining = slo.errorBudget - (errorBudgetConsumed * slo.errorBudget / (100 - slo.target));
    const burnRate = errorBudgetConsumed / (100 - slo.target);

    let status: 'HEALTHY' | 'AT_RISK' | 'BREACHED';
    if (current >= slo.target) {
      status = 'HEALTHY';
    } else if (current >= slo.target - 0.5) {
      status = 'AT_RISK';
    } else {
      status = 'BREACHED';
    }

    return { slo, current, errorBudgetRemaining, burnRate, status };
  }

  private async fetchMetric(slo: SLO): Promise<number> {
    // Fetch from Prometheus API
    const response = await fetch(
      `http://prometheus:9090/api/v1/query?query=${encodeURIComponent(slo.indicators[0].query)}`
    );
    const data = await response.json() as { data: { result: [{ value: [number, string] }] } };
    return parseFloat(data.data.result[0]?.value[1] || '0') * 100;
  }

  async generateSLOReport(): Promise<string> {
    const results = await Promise.all(
      orderServiceSLOs.map(slo => this.calculateCurrentSLO(slo))
    );

    let report = '# SLO Report\n\n';
    
    for (const result of results) {
      const statusEmoji = result.status === 'HEALTHY' ? '✅' :
        result.status === 'AT_RISK' ? '⚠️' : '🔴';
      
      report += `## ${statusEmoji} ${result.slo.name}\n`;
      report += `- **Target:** ${result.slo.target}%\n`;
      report += `- **Current:** ${result.current.toFixed(3)}%\n`;
      report += `- **Error Budget Remaining:** ${result.errorBudgetRemaining.toFixed(1)} minutes\n`;
      report += `- **Burn Rate:** ${result.burnRate.toFixed(2)}x\n\n`;
    }

    return report;
  }
}
```

---

## 6. Capacity Planning

```typescript
// capacity-planning.ts

interface ServiceMetrics {
  name: string;
  currentRPS: number;
  p95LatencyMs: number;
  cpuUsagePercent: number;
  memoryUsageMB: number;
  dbConnectionsUsed: number;
  replicaCount: number;
}

interface GrowthProjection {
  currentMonth: number;
  month3: number;
  month6: number;
  month12: number;
}

class CapacityPlanner {
  calculateCapacityNeeds(
    metrics: ServiceMetrics,
    growthRatePerMonth: number
  ): {
    currentUtilization: Record<string, number>;
    projections: GrowthProjection;
    recommendations: string[];
  } {
    const currentUtilization = {
      cpu: metrics.cpuUsagePercent,
      memory: (metrics.memoryUsageMB / 1024) * 100, // assuming 1GB limit
      rps: (metrics.currentRPS / (metrics.replicaCount * 1000)) * 100,
      dbConnections: (metrics.dbConnectionsUsed / 100) * 100,
    };

    // Project growth
    const projections: GrowthProjection = {
      currentMonth: metrics.currentRPS,
      month3: metrics.currentRPS * Math.pow(1 + growthRatePerMonth, 3),
      month6: metrics.currentRPS * Math.pow(1 + growthRatePerMonth, 6),
      month12: metrics.currentRPS * Math.pow(1 + growthRatePerMonth, 12),
    };

    const recommendations: string[] = [];

    // CPU recommendations
    if (currentUtilization.cpu > 70) {
      const neededReplicas = Math.ceil(metrics.replicaCount * (currentUtilization.cpu / 70));
      recommendations.push(
        `Scale from ${metrics.replicaCount} to ${neededReplicas} replicas to reduce CPU pressure`
      );
    }

    // Memory recommendations
    if (currentUtilization.memory > 80) {
      recommendations.push(
        `Increase memory limit from current value, current usage is too high`
      );
    }

    // RPS scaling
    const rpsAt12Months = projections.month12;
    const currentCapacity = metrics.replicaCount * 1000; // 1000 RPS per replica
    if (rpsAt12Months > currentCapacity * 0.8) {
      const neededReplicas = Math.ceil(rpsAt12Months / (1000 * 0.8));
      recommendations.push(
        `Plan to scale to ${neededReplicas} replicas within 12 months to handle projected ${Math.round(rpsAt12Months)} RPS`
      );
    }

    // Database connections
    if (currentUtilization.dbConnections > 80) {
      recommendations.push(
        'Consider adding read replicas or connection pooler (PgBouncer)'
      );
    }

    return { currentUtilization, projections, recommendations };
  }

  generateCapacityReport(
    services: ServiceMetrics[],
    growthRatePerMonth: number = 0.1
  ): void {
    console.log('=== Capacity Planning Report ===\n');
    
    for (const service of services) {
      const analysis = this.calculateCapacityNeeds(service, growthRatePerMonth);
      
      console.log(`Service: ${service.name}`);
      console.log(`Current RPS: ${service.currentRPS}`);
      console.log('Projections:');
      console.log(`  3 months: ${Math.round(analysis.projections.month3)} RPS`);
      console.log(`  6 months: ${Math.round(analysis.projections.month6)} RPS`);
      console.log(`  12 months: ${Math.round(analysis.projections.month12)} RPS`);
      
      if (analysis.recommendations.length > 0) {
        console.log('Recommendations:');
        analysis.recommendations.forEach(r => console.log(`  - ${r}`));
      }
      
      console.log('');
    }
  }
}
```

---

## 7. Disaster Recovery Plan

```typescript
// disaster-recovery.ts

interface RecoveryObjective {
  rto: number; // Recovery Time Objective (minutes)
  rpo: number; // Recovery Point Objective (minutes)
  mttr: number; // Mean Time To Recovery (minutes)
}

interface DRPlan {
  service: string;
  tier: 'CRITICAL' | 'HIGH' | 'MEDIUM' | 'LOW';
  objectives: RecoveryObjective;
  scenarios: DRScenario[];
  testSchedule: string;
  lastTested: Date;
}

interface DRScenario {
  name: string;
  trigger: string;
  procedure: string[];
  estimatedTime: number; // minutes
  automatable: boolean;
}

const orderServiceDRPlan: DRPlan = {
  service: 'Order Service',
  tier: 'CRITICAL',
  objectives: {
    rto: 15,  // 15 minutes to restore
    rpo: 5,   // Max 5 minutes of data loss
    mttr: 30, // Mean time to recover
  },
  scenarios: [
    {
      name: 'Primary Database Failure',
      trigger: 'PostgreSQL primary becomes unavailable',
      procedure: [
        '1. Alert triggers - PagerDuty notifies on-call',
        '2. Verify failure: check AWS RDS console or pg_isready',
        '3. Initiate automatic failover if not already started',
        '4. Update DNS/Route53 to point to replica',
        '5. Verify order service reconnects (max 60 seconds)',
        '6. Test order creation to verify recovery',
        '7. Update status page',
        '8. Begin post-mortem investigation',
      ],
      estimatedTime: 10,
      automatable: true,
    },
    {
      name: 'Region Failure',
      trigger: 'AWS region ap-southeast-1 becomes unavailable',
      procedure: [
        '1. Verify region failure via AWS Health Dashboard',
        '2. Activate DR region (ap-northeast-1)',
        '3. Update global Route53 health checks to failover',
        '4. Verify data sync status from last cross-region backup',
        '5. Scale up DR region resources',
        '6. Update service configurations for new region',
        '7. Run smoke tests',
        '8. Communicate to stakeholders',
      ],
      estimatedTime: 30,
      automatable: false,
    },
    {
      name: 'Data Corruption',
      trigger: 'Critical data corruption detected',
      procedure: [
        '1. Immediately stop writes to affected service',
        '2. Assess scope of corruption',
        '3. Identify last known good backup',
        '4. Restore from backup to DR environment first',
        '5. Validate restored data',
        '6. Execute point-in-time recovery if needed',
        '7. Replay any events after backup from Kafka',
        '8. Gradual traffic restoration with monitoring',
      ],
      estimatedTime: 60,
      automatable: false,
    },
  ],
  testSchedule: 'Quarterly',
  lastTested: new Date('2024-01-15'),
};

// Automated DR Testing
class DRTester {
  async runDRTest(scenario: string): Promise<{
    passed: boolean;
    actualRTO: number;
    notes: string[];
  }> {
    const start = Date.now();
    const notes: string[] = [];
    let passed = true;

    try {
      notes.push(`Starting DR test: ${scenario}`);
      
      // Simulate failure
      await this.simulateFailure(scenario);
      notes.push('Failure simulated');

      // Start recovery
      await this.executeRecovery(scenario);
      notes.push('Recovery procedure executed');

      // Verify recovery
      const verified = await this.verifyRecovery();
      if (!verified) {
        passed = false;
        notes.push('FAILED: Recovery verification failed');
      } else {
        notes.push('Recovery verified successfully');
      }

    } catch (error) {
      passed = false;
      notes.push(`ERROR: ${error}`);
    }

    const actualRTO = Math.round((Date.now() - start) / 60000);

    return { passed, actualRTO, notes };
  }

  private async simulateFailure(scenario: string): Promise<void> {
    console.log(`Simulating: ${scenario}`);
    await new Promise(resolve => setTimeout(resolve, 1000));
  }

  private async executeRecovery(scenario: string): Promise<void> {
    console.log(`Executing recovery for: ${scenario}`);
    await new Promise(resolve => setTimeout(resolve, 2000));
  }

  private async verifyRecovery(): Promise<boolean> {
    const response = await fetch('http://order-service/health');
    return response.ok;
  }
}
```

---

## 8. Backup and Restore Procedures

```typescript
// backup-restore.ts

interface BackupConfig {
  schedule: string;
  retention: {
    daily: number;
    weekly: number;
    monthly: number;
  };
  encryption: boolean;
  compression: boolean;
  destination: {
    primary: string;
    secondary: string;
  };
}

const backupConfigs: Record<string, BackupConfig> = {
  postgresql: {
    schedule: '0 */6 * * *', // Every 6 hours
    retention: {
      daily: 7,
      weekly: 4,
      monthly: 12,
    },
    encryption: true,
    compression: true,
    destination: {
      primary: 's3://backups/postgres/primary/',
      secondary: 's3://backups-dr/postgres/secondary/',
    },
  },
  redis: {
    schedule: '0 */1 * * *', // Every hour
    retention: {
      daily: 3,
      weekly: 1,
      monthly: 3,
    },
    encryption: true,
    compression: true,
    destination: {
      primary: 's3://backups/redis/primary/',
      secondary: 's3://backups-dr/redis/secondary/',
    },
  },
};

class BackupManager {
  async createBackup(service: string, config: BackupConfig): Promise<string> {
    const timestamp = new Date().toISOString().replace(/[:.]/g, '-');
    const backupId = `${service}-${timestamp}`;
    
    console.log(`Creating backup: ${backupId}`);
    
    // Execute backup based on service type
    if (service === 'postgresql') {
      await this.backupPostgreSQL(backupId, config);
    } else if (service === 'redis') {
      await this.backupRedis(backupId, config);
    }
    
    // Verify backup integrity
    await this.verifyBackup(backupId, config.destination.primary);
    
    // Copy to secondary destination
    await this.copyToSecondary(backupId, config);
    
    console.log(`Backup completed: ${backupId}`);
    return backupId;
  }

  private async backupPostgreSQL(backupId: string, config: BackupConfig): Promise<void> {
    const dumpCommand = [
      'pg_dump',
      '--format=custom',
      '--compress=9',
      '--no-acl',
      '--no-owner',
      `--file=/tmp/${backupId}.dump`,
      process.env.DATABASE_URL || '',
    ].join(' ');

    console.log(`Running: ${dumpCommand}`);
    // Execute command
  }

  private async backupRedis(backupId: string, config: BackupConfig): Promise<void> {
    // Trigger Redis BGSAVE
    console.log('Triggering Redis BGSAVE...');
    // Connect to Redis and trigger backup
  }

  private async verifyBackup(backupId: string, destination: string): Promise<void> {
    console.log(`Verifying backup integrity: ${backupId}`);
    // Verify file exists and is not corrupted
  }

  private async copyToSecondary(backupId: string, config: BackupConfig): Promise<void> {
    console.log(`Copying ${backupId} to secondary: ${config.destination.secondary}`);
    // Copy to secondary storage
  }

  async restoreFromBackup(backupId: string, targetEnvironment: string): Promise<void> {
    console.log(`Restoring ${backupId} to ${targetEnvironment}`);
    
    // Download backup
    const localPath = await this.downloadBackup(backupId);
    
    // Verify integrity before restore
    await this.verifyBackupIntegrity(localPath);
    
    // Execute restore
    await this.executeRestore(localPath, targetEnvironment);
    
    // Verify restore
    await this.verifyRestore(targetEnvironment);
    
    console.log(`Restore completed successfully`);
  }

  private async downloadBackup(backupId: string): Promise<string> {
    return `/tmp/restore-${backupId}.dump`;
  }

  private async verifyBackupIntegrity(path: string): Promise<void> {
    console.log(`Verifying integrity: ${path}`);
  }

  private async executeRestore(localPath: string, targetEnvironment: string): Promise<void> {
    const restoreCommand = [
      'pg_restore',
      '--clean',
      '--if-exists',
      `--dbname=${targetEnvironment}`,
      localPath,
    ].join(' ');

    console.log(`Running: ${restoreCommand}`);
  }

  private async verifyRestore(targetEnvironment: string): Promise<void> {
    console.log(`Verifying restore at: ${targetEnvironment}`);
  }
}
```

---

## 9. Security Hardening Checklist

```yaml
# kubernetes-security-hardening.yaml
# Security hardening สำหรับ Kubernetes Pods

apiVersion: v1
kind: Pod
metadata:
  name: order-service-hardened
  namespace: production
  labels:
    app: order-service
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  
  containers:
    - name: order-service
      image: order-service:latest
      
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop:
            - ALL
          add:
            - NET_BIND_SERVICE
      
      env:
        - name: NODE_ENV
          value: production
        # ไม่ใส่ secrets ตรงๆ - ใช้ Secret references
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: order-service-secrets
              key: database-url
        - name: REDIS_URL
          valueFrom:
            secretKeyRef:
              name: order-service-secrets
              key: redis-url
      
      resources:
        requests:
          memory: "128Mi"
          cpu: "100m"
        limits:
          memory: "512Mi"
          cpu: "500m"
      
      # Read-only filesystem mounts
      volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /app/.cache
      
      livenessProbe:
        httpGet:
          path: /health
          port: 3000
        initialDelaySeconds: 30
        periodSeconds: 10
        failureThreshold: 3
      
      readinessProbe:
        httpGet:
          path: /ready
          port: 3000
        initialDelaySeconds: 5
        periodSeconds: 5
  
  volumes:
    - name: tmp
      emptyDir: {}
    - name: cache
      emptyDir: {}
  
  # Service Account with minimal permissions
  serviceAccountName: order-service-sa
  automountServiceAccountToken: false
  
  # Image pull policy
  imagePullPolicy: Always
---
# NetworkPolicy - restrict traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: order-service-netpol
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: order-service
  policyTypes:
    - Ingress
    - Egress
  
  ingress:
    # ยอมรับ traffic จาก API Gateway เท่านั้น
    - from:
        - podSelector:
            matchLabels:
              app: api-gateway
      ports:
        - protocol: TCP
          port: 3000
    # ยอมรับ Prometheus scraping
    - from:
        - namespaceSelector:
            matchLabels:
              name: monitoring
      ports:
        - protocol: TCP
          port: 9090
  
  egress:
    # ออกไปยัง PostgreSQL
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432
    # ออกไปยัง Redis
    - to:
        - podSelector:
            matchLabels:
              app: redis
      ports:
        - protocol: TCP
          port: 6379
    # DNS
    - ports:
        - protocol: UDP
          port: 53
---
# RBAC - minimal permissions
apiVersion: v1
kind: ServiceAccount
metadata:
  name: order-service-sa
  namespace: production
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: order-service-role
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["order-service-config"]
    verbs: ["get", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: order-service-rolebinding
  namespace: production
subjects:
  - kind: ServiceAccount
    name: order-service-sa
    namespace: production
roleRef:
  kind: Role
  apiGroup: rbac.authorization.k8s.io
  name: order-service-role
```

---

## 10. Documentation Requirements

```typescript
// api-documentation.ts
// OpenAPI/Swagger specification

const openApiSpec = {
  openapi: '3.0.3',
  info: {
    title: 'Order Service API',
    description: `
      Order Service handles order lifecycle management.
      
      ## Authentication
      All endpoints require Bearer token authentication.
      
      ## Rate Limiting
      - Standard: 100 requests/minute
      - Burst: 200 requests/minute
      
      ## Versioning
      API version is included in the URL path (/v1, /v2)
    `,
    version: '2.1.0',
    contact: {
      name: 'Order Team',
      email: 'order-team@company.com',
      url: 'https://wiki.company.com/order-service',
    },
    license: {
      name: 'Internal',
    },
  },
  servers: [
    { url: 'https://api.company.com/v1', description: 'Production' },
    { url: 'https://api-staging.company.com/v1', description: 'Staging' },
  ],
  paths: {
    '/orders': {
      post: {
        summary: 'Create a new order',
        operationId: 'createOrder',
        tags: ['Orders'],
        security: [{ BearerAuth: [] }],
        requestBody: {
          required: true,
          content: {
            'application/json': {
              schema: {
                type: 'object',
                required: ['userId', 'items'],
                properties: {
                  userId: { type: 'string', format: 'uuid' },
                  items: {
                    type: 'array',
                    minItems: 1,
                    items: {
                      type: 'object',
                      required: ['productId', 'quantity'],
                      properties: {
                        productId: { type: 'string' },
                        quantity: { type: 'integer', minimum: 1 },
                        price: { type: 'number', minimum: 0 },
                      },
                    },
                  },
                },
              },
            },
          },
        },
        responses: {
          '201': {
            description: 'Order created successfully',
            content: {
              'application/json': {
                schema: { $ref: '#/components/schemas/Order' },
              },
            },
          },
          '400': { $ref: '#/components/responses/BadRequest' },
          '401': { $ref: '#/components/responses/Unauthorized' },
          '422': { $ref: '#/components/responses/ValidationError' },
          '503': { $ref: '#/components/responses/ServiceUnavailable' },
        },
      },
    },
  },
  components: {
    schemas: {
      Order: {
        type: 'object',
        properties: {
          id: { type: 'string', format: 'uuid' },
          userId: { type: 'string', format: 'uuid' },
          items: {
            type: 'array',
            items: { $ref: '#/components/schemas/OrderItem' },
          },
          totalAmount: { type: 'number' },
          status: {
            type: 'string',
            enum: ['PENDING', 'CONFIRMED', 'SHIPPED', 'DELIVERED', 'CANCELLED'],
          },
          createdAt: { type: 'string', format: 'date-time' },
          updatedAt: { type: 'string', format: 'date-time' },
        },
      },
      OrderItem: {
        type: 'object',
        properties: {
          productId: { type: 'string' },
          quantity: { type: 'integer' },
          price: { type: 'number' },
        },
      },
    },
    responses: {
      BadRequest: {
        description: 'Bad Request',
        content: {
          'application/json': {
            schema: {
              type: 'object',
              properties: {
                error: { type: 'string' },
                details: { type: 'array', items: { type: 'string' } },
              },
            },
          },
        },
      },
      Unauthorized: {
        description: 'Unauthorized',
        content: {
          'application/json': {
            schema: {
              type: 'object',
              properties: {
                error: { type: 'string', example: 'Invalid or expired token' },
              },
            },
          },
        },
      },
      ValidationError: {
        description: 'Validation Error',
      },
      ServiceUnavailable: {
        description: 'Service Unavailable',
      },
    },
    securitySchemes: {
      BearerAuth: {
        type: 'http',
        scheme: 'bearer',
        bearerFormat: 'JWT',
      },
    },
  },
};
```

---

## Production Readiness Score Card

```typescript
// readiness-scorecard.ts

interface ReadinessCategory {
  name: string;
  weight: number;
  checks: ReadinessCheck[];
}

interface ReadinessCheck {
  name: string;
  passed: boolean;
  critical: boolean;
  evidence?: string;
}

function calculateReadinessScore(categories: ReadinessCategory[]): {
  score: number;
  ready: boolean;
  criticalFailures: string[];
} {
  const criticalFailures: string[] = [];
  let totalScore = 0;
  let totalWeight = 0;

  for (const category of categories) {
    const passed = category.checks.filter(c => c.passed).length;
    const total = category.checks.length;
    const categoryScore = (passed / total) * category.weight;
    
    totalScore += categoryScore;
    totalWeight += category.weight;

    // Collect critical failures
    category.checks
      .filter(c => c.critical && !c.passed)
      .forEach(c => criticalFailures.push(`${category.name}: ${c.name}`));
  }

  const score = (totalScore / totalWeight) * 100;
  const ready = criticalFailures.length === 0 && score >= 85;

  return { score, ready, criticalFailures };
}

const productionReadiness: ReadinessCategory[] = [
  {
    name: 'Security',
    weight: 25,
    checks: [
      { name: 'Authentication implemented', passed: true, critical: true },
      { name: 'Authorization implemented', passed: true, critical: true },
      { name: 'Secrets in vault', passed: true, critical: true },
      { name: 'HTTPS/TLS configured', passed: true, critical: true },
      { name: 'Rate limiting enabled', passed: true, critical: false },
      { name: 'Security scan passed', passed: true, critical: false },
    ],
  },
  {
    name: 'Reliability',
    weight: 25,
    checks: [
      { name: 'Health checks configured', passed: true, critical: true },
      { name: 'Circuit breakers implemented', passed: true, critical: false },
      { name: 'Retry logic implemented', passed: true, critical: false },
      { name: 'Graceful shutdown', passed: true, critical: false },
      { name: 'PodDisruptionBudget set', passed: true, critical: false },
    ],
  },
  {
    name: 'Observability',
    weight: 20,
    checks: [
      { name: 'Structured logging', passed: true, critical: true },
      { name: 'Metrics exposed', passed: true, critical: true },
      { name: 'Distributed tracing', passed: true, critical: false },
      { name: 'Alerts configured', passed: true, critical: false },
      { name: 'Dashboards created', passed: true, critical: false },
    ],
  },
  {
    name: 'Scalability',
    weight: 15,
    checks: [
      { name: 'Horizontal scaling tested', passed: true, critical: false },
      { name: 'Resource limits set', passed: true, critical: true },
      { name: 'HPA configured', passed: true, critical: false },
      { name: 'Load tested', passed: true, critical: false },
    ],
  },
  {
    name: 'Documentation',
    weight: 15,
    checks: [
      { name: 'API documented (OpenAPI)', passed: true, critical: false },
      { name: 'Runbook created', passed: true, critical: true },
      { name: 'Architecture diagram', passed: true, critical: false },
      { name: 'Dependency map', passed: true, critical: false },
      { name: 'On-call rotation setup', passed: true, critical: false },
    ],
  },
];

const result = calculateReadinessScore(productionReadiness);
console.log(`Production Readiness Score: ${result.score.toFixed(1)}%`);
console.log(`Ready for Production: ${result.ready}`);
if (result.criticalFailures.length > 0) {
  console.log('Critical Failures:', result.criticalFailures);
}
```

---

## สรุปตาราง Production Readiness

| หมวดหมู่ | รายการ | ความสำคัญ | เครื่องมือ |
|---------|--------|-----------|-----------|
| Security | Authentication/Authorization | Critical | JWT, OAuth2, RBAC |
| Security | Secrets Management | Critical | HashiCorp Vault, AWS Secrets Manager |
| Security | TLS/HTTPS | Critical | cert-manager, Let's Encrypt |
| Security | Rate Limiting | High | nginx, Kong, Envoy |
| Reliability | Health Checks | Critical | Kubernetes probes |
| Reliability | Circuit Breakers | High | Resilience4j, Hystrix |
| Reliability | Graceful Shutdown | High | SIGTERM handler |
| Observability | Structured Logging | Critical | Winston, Pino |
| Observability | Metrics | Critical | Prometheus, Datadog |
| Observability | Tracing | High | Jaeger, Zipkin |
| Observability | Alerting | Critical | Alertmanager, PagerDuty |
| Scalability | Resource Limits | Critical | Kubernetes resources |
| Scalability | HPA | High | Kubernetes HPA |
| Scalability | Load Testing | High | k6, Gatling |
| Documentation | API Docs | High | OpenAPI/Swagger |
| Documentation | Runbook | Critical | Confluence, Wiki |
| DR | Backup/Restore | Critical | pg_dump, AWS RDS |
| DR | RTO/RPO Defined | Critical | DR Plan |
| On-call | Escalation Policy | Critical | PagerDuty |
| On-call | Rotation Setup | High | PagerDuty/OpsGenie |

| SLO Level | Availability | Error Budget/Month |
|-----------|-------------|-------------------|
| 99.9% (three nines) | ~43.8 min downtime | 43.8 min |
| 99.95% | ~21.9 min | 21.9 min |
| 99.99% (four nines) | ~4.4 min | 4.4 min |

---

## สรุป

Production Readiness ต้องครอบคลุมทุกด้าน:

1. **Security**: Authentication, Authorization, Secrets, TLS
2. **Reliability**: Health checks, Circuit breakers, Graceful shutdown
3. **Observability**: Logs, Metrics, Traces, Alerts
4. **Scalability**: Resource limits, HPA, Load testing
5. **Documentation**: API docs, Runbooks, Architecture diagrams
6. **DR**: Backup/restore, RTO/RPO
7. **On-call**: Rotation, escalation, incident response
8. **SLOs**: Define, track, alert on burn rate

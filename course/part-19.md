# Part 19: Advanced Observability - SLOs, SLAs, and Production Monitoring

## ภาพรวม

ใน Part นี้เราจะเรียนรู้ระบบ Observability ระดับ Production:
- **SLI/SLO/SLA** - วัดความน่าเชื่อถือของระบบ
- **Error Budget** - บริหารจัดการ Reliability
- **Alerting ที่ถูกต้อง** - ลด Alert Fatigue
- **Dashboards** - Grafana สำหรับ Production
- **Chaos Engineering** - ทดสอบ Resilience
- **Runbooks** - แก้ปัญหาอย่างมีระบบ

---

## 1. SLI, SLO, SLA - ภาษาของ Reliability

### 1.1 Definitions

```
SLI (Service Level Indicator)
= การวัดพฤติกรรมของระบบ
= เช่น "99.5% ของ requests ตอบกลับภายใน 200ms"

SLO (Service Level Objective)
= เป้าหมายที่เราต้องการ
= เช่น "ระบบต้องมี availability 99.9%"

SLA (Service Level Agreement)
= สัญญากับลูกค้า
= เช่น "ถ้า uptime < 99.5% ลูกค้าได้รับเงินคืน"

Error Budget
= เวลาที่ระบบอนุญาตให้ล้มเหลวได้
= 99.9% SLO = 0.1% error budget = 43.8 นาที/เดือน
```

### 1.2 เลือก SLIs ที่ดี

```javascript
// sli-definitions.js

const SLI_TYPES = {
  // Availability SLI
  AVAILABILITY: {
    description: 'Fraction of requests served successfully',
    formula: 'successful_requests / total_requests',
    example: 'HTTP 2xx responses / total responses'
  },

  // Latency SLI
  LATENCY: {
    description: 'Fraction of requests under threshold',
    formula: 'requests_under_threshold / total_requests',
    example: 'requests < 200ms / total requests'
  },

  // Error Rate SLI
  ERROR_RATE: {
    description: 'Fraction of requests that are errors',
    formula: 'error_requests / total_requests',
    example: 'HTTP 5xx / total responses'
  },

  // Throughput SLI
  THROUGHPUT: {
    description: 'Rate of requests per time unit',
    formula: 'requests_per_second',
    example: 'RPS averaged over 5 minutes'
  },

  // Freshness SLI (for data pipelines)
  FRESHNESS: {
    description: 'Age of data',
    formula: 'now - last_update_time',
    example: 'Cache updated within 60 seconds'
  }
};
```

### 1.3 Prometheus Rules สำหรับ SLIs

```yaml
# prometheus/slo-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: slo-rules
  namespace: monitoring
  labels:
    prometheus: kube-prometheus
spec:
  groups:
    # ===== User Service SLOs =====
    - name: user-service.slos
      interval: 30s
      rules:
        # SLI: Availability (% of successful requests)
        - record: sli:user_service:availability_ratio
          expr: |
            sum(rate(http_requests_total{
              service="user-service",
              status_code!~"5.."
            }[5m]))
            /
            sum(rate(http_requests_total{
              service="user-service"
            }[5m]))

        # SLI: Latency (% of requests < 200ms)
        - record: sli:user_service:latency_ratio
          expr: |
            sum(rate(http_request_duration_seconds_bucket{
              service="user-service",
              le="0.2"
            }[5m]))
            /
            sum(rate(http_request_duration_seconds_count{
              service="user-service"
            }[5m]))

        # SLO: Availability = 99.9%
        - alert: UserServiceAvailabilitySLOViolation
          expr: |
            (
              1 - sli:user_service:availability_ratio
            ) > (1 - 0.999)
          for: 5m
          labels:
            severity: critical
            slo: availability
            service: user-service
          annotations:
            summary: "User Service Availability SLO violated"
            description: |
              Availability is {{ $value | humanizePercentage }} 
              (SLO target: 99.9%)
            runbook: "https://runbooks.company.com/user-service/availability"

        # Error Budget Burn Rate Alert (fast burn)
        - alert: UserServiceErrorBudgetFastBurn
          expr: |
            (
              1 - sli:user_service:availability_ratio
            ) > 14.4 * (1 - 0.999)
          for: 1m
          labels:
            severity: critical
            slo: availability
          annotations:
            summary: "Error budget burning too fast (14.4x rate)"
            description: "At this rate, error budget exhausted in 1 hour"

        # Error Budget Burn Rate Alert (slow burn)
        - alert: UserServiceErrorBudgetSlowBurn
          expr: |
            (
              1 - sli:user_service:availability_ratio
            ) > 6 * (1 - 0.999)
          for: 30m
          labels:
            severity: warning
            slo: availability
          annotations:
            summary: "Error budget burning faster than expected (6x rate)"
            description: "At this rate, error budget exhausted in 6 hours"
```

### 1.4 Error Budget Tracking

```javascript
// error-budget-tracker.js
const prometheus = require('prom-client');

class ErrorBudgetTracker {
  constructor(prometheusClient) {
    this.client = prometheusClient;

    this.errorBudgetRemaining = new prometheus.Gauge({
      name: 'slo_error_budget_remaining_ratio',
      help: 'Remaining error budget as a ratio',
      labelNames: ['service', 'slo_name']
    });

    this.errorBudgetBurnRate = new prometheus.Gauge({
      name: 'slo_error_budget_burn_rate',
      help: 'Error budget burn rate (1.0 = consuming at exactly expected rate)',
      labelNames: ['service', 'slo_name', 'window']
    });
  }

  async calculateErrorBudget(service, sloConfig) {
    const {
      targetPercentage,   // e.g., 99.9
      windowDays,         // e.g., 30 days
      currentAvailability // measured value
    } = sloConfig;

    const errorBudgetTotal = (100 - targetPercentage) / 100;
    const errorRate = 1 - (currentAvailability / 100);
    const consumed = errorRate / errorBudgetTotal;
    const remaining = Math.max(0, 1 - consumed);

    this.errorBudgetRemaining.set(
      { service, slo_name: 'availability' },
      remaining
    );

    return {
      totalMinutes: windowDays * 24 * 60 * errorBudgetTotal,
      consumedMinutes: windowDays * 24 * 60 * errorRate,
      remainingMinutes: windowDays * 24 * 60 * remaining * errorBudgetTotal,
      remainingPercentage: remaining * 100,
      status: remaining > 0.5 ? 'healthy' : remaining > 0.25 ? 'warning' : 'critical'
    };
  }
}

// SLO Dashboard API
async function getSLODashboard(req, res) {
  const slos = [
    {
      service: 'user-service',
      name: 'availability',
      target: 99.9,
      current: await measureAvailability('user-service', '30d'),
      window: '30 days'
    },
    {
      service: 'user-service',
      name: 'latency-p99',
      target: 99.0,  // 99% requests < 500ms
      current: await measureLatencyCompliance('user-service', 0.5, '30d'),
      window: '30 days'
    }
  ];

  const tracker = new ErrorBudgetTracker(prometheus);
  const results = await Promise.all(
    slos.map(async slo => ({
      ...slo,
      errorBudget: await tracker.calculateErrorBudget(slo.service, {
        targetPercentage: slo.target,
        windowDays: 30,
        currentAvailability: slo.current
      })
    }))
  );

  res.json({ slos: results });
}
```

---

## 2. Grafana Dashboards สำหรับ Production

### 2.1 Grafana Dashboard as Code

```json
{
  "dashboard": {
    "id": null,
    "title": "Microservices Overview",
    "tags": ["microservices", "production"],
    "timezone": "browser",
    "refresh": "30s",
    "time": {
      "from": "now-1h",
      "to": "now"
    },
    "panels": [
      {
        "id": 1,
        "title": "Request Rate",
        "type": "stat",
        "gridPos": { "h": 4, "w": 6, "x": 0, "y": 0 },
        "targets": [
          {
            "expr": "sum(rate(http_requests_total[5m]))",
            "legendFormat": "RPS"
          }
        ],
        "options": {
          "reduceOptions": { "calcs": ["lastNotNull"] },
          "colorMode": "background",
          "graphMode": "area",
          "justifyMode": "center"
        }
      },
      {
        "id": 2,
        "title": "Error Rate",
        "type": "stat",
        "gridPos": { "h": 4, "w": 6, "x": 6, "y": 0 },
        "targets": [
          {
            "expr": "sum(rate(http_requests_total{status_code=~'5..'}[5m])) / sum(rate(http_requests_total[5m])) * 100",
            "legendFormat": "Error %"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "thresholds": {
              "steps": [
                { "color": "green", "value": null },
                { "color": "yellow", "value": 0.1 },
                { "color": "red", "value": 1 }
              ]
            },
            "unit": "percent"
          }
        }
      },
      {
        "id": 3,
        "title": "P99 Latency by Service",
        "type": "timeseries",
        "gridPos": { "h": 8, "w": 12, "x": 0, "y": 4 },
        "targets": [
          {
            "expr": "histogram_quantile(0.99, sum by (service, le) (rate(http_request_duration_seconds_bucket[5m])))",
            "legendFormat": "{{service}} p99"
          },
          {
            "expr": "histogram_quantile(0.95, sum by (service, le) (rate(http_request_duration_seconds_bucket[5m])))",
            "legendFormat": "{{service}} p95"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "s",
            "custom": {
              "thresholdsStyle": { "mode": "line" }
            },
            "thresholds": {
              "steps": [
                { "color": "green", "value": null },
                { "color": "yellow", "value": 0.2 },
                { "color": "red", "value": 0.5 }
              ]
            }
          }
        }
      }
    ]
  }
}
```

### 2.2 Grafana Provisioning

```yaml
# grafana/provisioning/datasources/prometheus.yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    url: http://prometheus:9090
    access: proxy
    isDefault: true
    jsonData:
      timeInterval: "15s"
      queryTimeout: "60s"
      httpMethod: POST

  - name: Loki
    type: loki
    url: http://loki:3100
    access: proxy

  - name: Jaeger
    type: jaeger
    url: http://jaeger:16686
    access: proxy
    jsonData:
      tracesToLogs:
        datasourceUid: loki
        tags: ['service', 'trace_id']
```

```yaml
# grafana/provisioning/dashboards/dashboards.yaml
apiVersion: 1
providers:
  - name: 'default'
    orgId: 1
    folder: 'Microservices'
    type: file
    disableDeletion: false
    updateIntervalSeconds: 30
    options:
      path: /var/lib/grafana/dashboards
      foldersFromFilesStructure: true
```

---

## 3. Advanced Alerting Strategy

### 3.1 Alert Tiers

```yaml
# ระดับ Alert
# P1: Critical - ตื่นกลางดึก
# P2: High - แจ้งเตือนทันที แต่ไม่ต้องตื่นกลางดึก
# P3: Medium - ดูในเวลาทำงาน
# P4: Low - ดูสัปดาห์นี้

# prometheus/alert-rules.yaml
groups:
  - name: p1.critical
    rules:
      # ระบบล่ม
      - alert: ServiceDown
        expr: up{job=~".*-service"} == 0
        for: 1m
        labels:
          severity: critical
          pagerduty_escalation: p1
        annotations:
          summary: "{{ $labels.job }} is DOWN"
          description: "Service {{ $labels.job }} has been down for more than 1 minute"
          runbook: "https://runbooks.company.com/service-down"

      # Error rate สูงมาก
      - alert: CriticalErrorRate
        expr: |
          sum(rate(http_requests_total{status_code=~"5.."}[5m])) by (service)
          /
          sum(rate(http_requests_total[5m])) by (service)
          > 0.05
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Critical error rate on {{ $labels.service }}"
          description: "Error rate is {{ $value | humanizePercentage }}"

  - name: p2.high
    rules:
      # Latency สูง
      - alert: HighLatency
        expr: |
          histogram_quantile(0.99,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (service, le)
          ) > 1
        for: 5m
        labels:
          severity: high
        annotations:
          summary: "High P99 latency on {{ $labels.service }}"
          description: "P99 latency is {{ $value }}s"

      # Error Budget burning fast
      - alert: ErrorBudgetBurnFast
        expr: |
          (1 - sli:user_service:availability_ratio) > 14.4 * 0.001
        for: 1m
        labels:
          severity: high
        annotations:
          summary: "Error budget burning at 14.4x rate"
          description: "Will exhaust error budget within 1 hour"

  - name: p3.medium
    rules:
      # Memory ใกล้เต็ม
      - alert: HighMemoryUsage
        expr: |
          container_memory_usage_bytes{container!=""}
          /
          container_spec_memory_limit_bytes{container!=""}
          > 0.85
        for: 10m
        labels:
          severity: medium
        annotations:
          summary: "High memory usage in {{ $labels.pod }}"

      # CPU สูง
      - alert: HighCPUUsage
        expr: |
          sum(rate(container_cpu_usage_seconds_total{container!=""}[5m])) by (pod)
          /
          sum(container_spec_cpu_quota{container!=""}/container_spec_cpu_period{container!=""}) by (pod)
          > 0.8
        for: 15m
        labels:
          severity: medium
```

### 3.2 Alertmanager Configuration

```yaml
# alertmanager.yaml
global:
  resolve_timeout: 5m
  slack_api_url: 'https://hooks.slack.com/services/...'
  pagerduty_url: 'https://events.pagerduty.com/v2/enqueue'

templates:
  - '/etc/alertmanager/templates/*.tmpl'

route:
  group_by: ['alertname', 'service', 'severity']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: 'slack-default'

  routes:
    # P1: Critical -> PagerDuty (wake up!)
    - match:
        severity: critical
      receiver: pagerduty-critical
      group_wait: 0s
      repeat_interval: 1h
      continue: true

    # P1: Critical -> Slack
    - match:
        severity: critical
      receiver: slack-critical
      continue: false

    # P2: High -> Slack #alerts channel
    - match:
        severity: high
      receiver: slack-high
      repeat_interval: 2h

    # P3-P4: Medium/Low -> Slack #monitoring
    - match_re:
        severity: medium|low
      receiver: slack-default
      repeat_interval: 8h

receivers:
  - name: pagerduty-critical
    pagerduty_configs:
      - routing_key: '{{ .ExternalURL }}'
        description: '{{ template "pagerduty.description" . }}'
        severity: '{{ if eq .CommonLabels.severity "critical" }}critical{{ else }}warning{{ end }}'
        details:
          summary: '{{ .CommonAnnotations.summary }}'
          runbook: '{{ .CommonAnnotations.runbook }}'

  - name: slack-critical
    slack_configs:
      - channel: '#alerts-critical'
        send_resolved: true
        username: 'Prometheus'
        icon_emoji: ':fire:'
        color: '{{ if eq .Status "firing" }}danger{{ else }}good{{ end }}'
        title: '{{ template "slack.title" . }}'
        text: '{{ template "slack.text" . }}'
        actions:
          - type: button
            text: 'Runbook'
            url: '{{ .CommonAnnotations.runbook }}'
          - type: button
            text: 'Dashboard'
            url: '{{ .CommonAnnotations.dashboard }}'

  - name: slack-high
    slack_configs:
      - channel: '#alerts'
        send_resolved: true
        color: '{{ if eq .Status "firing" }}warning{{ else }}good{{ end }}'
        title: '{{ template "slack.title" . }}'
        text: '{{ template "slack.text" . }}'

  - name: slack-default
    slack_configs:
      - channel: '#monitoring'
        send_resolved: false

inhibit_rules:
  # ถ้า service down แล้ว ไม่ต้องแจ้ง latency/error alerts ของ service นั้น
  - source_match:
      alertname: ServiceDown
    target_match_re:
      alertname: HighLatency|CriticalErrorRate
    equal: ['service']
```

### 3.3 Alert Templates

```
# alertmanager/templates/slack.tmpl
{{ define "slack.title" }}
[{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}]
{{ .CommonLabels.alertname }} - {{ .CommonLabels.service }}
{{ end }}

{{ define "slack.text" }}
{{ range .Alerts }}
*Alert:* {{ .Annotations.summary }}
*Description:* {{ .Annotations.description }}
*Severity:* {{ .Labels.severity }}
*Service:* {{ .Labels.service }}
{{ if .Annotations.runbook }}*Runbook:* {{ .Annotations.runbook }}{{ end }}
*Started:* {{ .StartsAt | since }}
{{ end }}
{{ end }}
```

---

## 4. Chaos Engineering

### 4.1 Chaos Engineering Principles

```
Chaos Engineering Workflow:
1. Define steady state (normal behavior)
2. Hypothesize: "ระบบยังทำงานถ้า X ล้มเหลว"
3. Introduce chaos in production (หรือ staging)
4. Observe actual vs expected behavior
5. Fix weaknesses found
6. Repeat
```

### 4.2 Chaos Experiments ด้วย Istio Fault Injection

```yaml
# chaos/latency-injection.yaml
# เพิ่ม latency 5 วินาที สำหรับ 10% ของ requests
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: user-service-chaos
spec:
  hosts:
    - user-service
  http:
    - fault:
        delay:
          percentage:
            value: 10
          fixedDelay: 5s
      route:
        - destination:
            host: user-service

---
# chaos/error-injection.yaml
# Return HTTP 500 สำหรับ 5% ของ requests
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: user-service-chaos
spec:
  hosts:
    - user-service
  http:
    - fault:
        abort:
          percentage:
            value: 5
          httpStatus: 500
      route:
        - destination:
            host: user-service
```

### 4.3 Chaos Monkey ด้วย Kubernetes

```javascript
// chaos/pod-killer.js
const k8s = require('@kubernetes/client-node');
const cron = require('node-cron');

class ChaosPodKiller {
  constructor(config) {
    const kc = new k8s.KubeConfig();
    kc.loadFromDefault();
    this.k8sApi = kc.makeApiClient(k8s.CoreV1Api);
    this.config = config;
  }

  async getTargetPods(namespace, selector) {
    const response = await this.k8sApi.listNamespacedPod(
      namespace,
      undefined, undefined, undefined, undefined,
      selector
    );
    return response.body.items;
  }

  async killRandomPod(namespace, selector) {
    const pods = await this.getTargetPods(namespace, selector);
    
    if (pods.length === 0) {
      console.log('No pods found');
      return;
    }

    // เลือก pod แบบ random
    const targetPod = pods[Math.floor(Math.random() * pods.length)];
    const podName = targetPod.metadata.name;

    console.log(`🔥 Killing pod: ${podName}`);

    await this.k8sApi.deleteNamespacedPod(podName, namespace);
    
    console.log(`✓ Pod ${podName} killed`);
    return podName;
  }

  schedule(cronExpression) {
    cron.schedule(cronExpression, async () => {
      try {
        // ทดสอบทุก service
        for (const target of this.config.targets) {
          if (Math.random() < target.probability) {
            await this.killRandomPod(
              target.namespace,
              target.selector
            );
          }
        }
      } catch (error) {
        console.error('Chaos experiment failed:', error);
      }
    });
  }
}

// ใช้งาน: kill pod ทุก 30 นาที ใน staging
const chaos = new ChaosPodKiller({
  targets: [
    {
      namespace: 'staging',
      selector: 'app=user-service',
      probability: 0.5  // 50% chance ใน staging
    },
    {
      namespace: 'production',
      selector: 'app=user-service',
      probability: 0.1  // 10% chance ใน production
    }
  ]
});

// Run ทุก 30 นาที ในเวลาทำงาน
chaos.schedule('*/30 9-17 * * 1-5');
```

### 4.4 Chaos Toolkit Experiment

```json
{
  "version": "1.0.0",
  "title": "Kill user-service pod and check recovery",
  "description": "Verify that the system recovers when a user-service pod is killed",

  "steady-state-hypothesis": {
    "title": "System is healthy",
    "probes": [
      {
        "type": "probe",
        "name": "user-service-responds",
        "tolerance": 200,
        "provider": {
          "type": "http",
          "url": "https://api.company.com/users/health",
          "timeout": 3
        }
      },
      {
        "type": "probe",
        "name": "error-rate-low",
        "tolerance": {
          "type": "range",
          "range": [0, 0.01]
        },
        "provider": {
          "type": "python",
          "module": "chaosprometheus.probes",
          "func": "query_within_range",
          "arguments": {
            "query": "sum(rate(http_requests_total{status_code=~'5..'}[5m])) / sum(rate(http_requests_total[5m]))",
            "when": "now"
          }
        }
      }
    ]
  },

  "method": [
    {
      "type": "action",
      "name": "kill-user-service-pod",
      "provider": {
        "type": "python",
        "module": "chaosk8s.pod.actions",
        "func": "terminate_pods",
        "arguments": {
          "label_selector": "app=user-service",
          "ns": "production",
          "qty": 1
        }
      },
      "pauses": {
        "after": 30
      }
    }
  ],

  "rollbacks": [
    {
      "type": "action",
      "name": "restore-original-replicas",
      "provider": {
        "type": "python",
        "module": "chaosk8s.deployment.actions",
        "func": "scale_deployment",
        "arguments": {
          "name": "user-service",
          "replicas": 3,
          "ns": "production"
        }
      }
    }
  ]
}
```

---

## 5. Production Runbooks

### 5.1 Runbook Template

```markdown
# Runbook: User Service High Error Rate

## Alert
**Alert Name:** UserServiceHighErrorRate  
**Severity:** P2 (High)  
**SLO Impact:** Availability SLO

## Symptoms
- Error rate > 1% for user-service
- HTTP 5xx responses increasing
- Possible impact on downstream services

## Impact Assessment
- [ ] Check how many users affected
- [ ] Check if it's complete outage or partial
- [ ] Check if downstream services are impacted

## Diagnosis Steps

### Step 1: Verify the alert is real
```bash
# ดู error rate ใน Prometheus
curl -s "http://prometheus:9090/api/v1/query?query=sum(rate(http_requests_total{service='user-service',status_code=~'5..'}[5m]))/sum(rate(http_requests_total{service='user-service'}[5m]))" | jq '.data.result'
```

### Step 2: Check pod status
```bash
kubectl get pods -n production -l app=user-service
kubectl describe pod <pod-name> -n production
```

### Step 3: Check recent logs
```bash
kubectl logs -n production -l app=user-service --tail=100 | grep -i error
```

### Step 4: Check recent deployments
```bash
kubectl rollout history deployment/user-service -n production
```

## Common Causes & Fixes

### Cause 1: Bad deployment
```bash
# Rollback
kubectl rollout undo deployment/user-service -n production
kubectl rollout status deployment/user-service -n production
```

### Cause 2: Database connection issues
```bash
# Check DB connectivity
kubectl exec -it <pod-name> -n production -- nc -zv postgresql 5432

# Check connection pool
kubectl logs <pod-name> -n production | grep -i "connection\|pool\|postgres"
```

### Cause 3: Memory issues
```bash
# Check memory
kubectl top pods -n production -l app=user-service

# Restart pods (rolling)
kubectl rollout restart deployment/user-service -n production
```

## Escalation
- After 15 min: Escalate to P1
- Contact: Platform Team
- Slack: #platform-oncall
```

### 5.2 Runbook Automation

```javascript
// runbooks/auto-remediation.js
const k8s = require('@kubernetes/client-node');

class AutoRemediation {
  constructor() {
    const kc = new k8s.KubeConfig();
    kc.loadFromDefault();
    this.appsApi = kc.makeApiClient(k8s.AppsV1Api);
    this.coreApi = kc.makeApiClient(k8s.CoreV1Api);
  }

  async handleHighMemoryAlert(service, namespace) {
    const pods = await this.getPodsWithHighMemory(service, namespace);
    
    for (const pod of pods) {
      console.log(`Restarting pod ${pod.name} due to high memory`);
      
      // Delete pod (Deployment will recreate it)
      await this.coreApi.deleteNamespacedPod(pod.name, namespace);
      
      // Log to audit
      await this.logRemediation({
        action: 'pod_restart',
        reason: 'high_memory',
        pod: pod.name,
        service,
        namespace
      });
    }
  }

  async handleBadDeployment(service, namespace) {
    const history = await this.getRolloutHistory(service, namespace);
    
    if (history.length >= 2) {
      // Rollback to previous version
      await this.rollback(service, namespace);
      
      console.log(`Rolled back ${service} to previous version`);
      
      await this.logRemediation({
        action: 'rollback',
        reason: 'high_error_rate',
        service,
        namespace
      });
    }
  }

  async rollback(service, namespace) {
    // kubectl rollout undo
    const patch = {
      spec: {
        rollbackTo: {
          revision: 0  // ย้อนไป revision ก่อนหน้า
        }
      }
    };
    
    await this.appsApi.patchNamespacedDeployment(
      service, namespace, patch,
      undefined, undefined, undefined, undefined,
      { headers: { 'Content-Type': 'application/strategic-merge-patch+json' } }
    );
  }

  async logRemediation(event) {
    // Log to audit system
    console.log('[AUTO-REMEDIATION]', JSON.stringify({
      ...event,
      timestamp: new Date().toISOString(),
      automated: true
    }));
  }
}
```

---

## 6. Capacity Planning

### 6.1 Prometheus Queries สำหรับ Capacity Planning

```promql
# CPU Utilization Trend (last 7 days)
# ใช้ predict_linear เพื่อ forecast
predict_linear(
  sum(rate(container_cpu_usage_seconds_total{namespace="production"}[5m]))[7d:1h],
  86400 * 30  # predict 30 days
)

# Memory Growth Rate
deriv(
  sum(container_memory_usage_bytes{namespace="production"})[7d:1h]
)

# Request Rate Trend
predict_linear(
  sum(rate(http_requests_total{namespace="production"}[5m]))[7d:1h],
  86400 * 30
)

# Storage growth
predict_linear(
  kubelet_volume_stats_used_bytes{namespace="production"}[7d:1h],
  86400 * 90  # 90 days
)
```

### 6.2 Capacity Planning Report

```javascript
// capacity-planning/report.js
const axios = require('axios');

async function generateCapacityReport() {
  const prometheus = axios.create({
    baseURL: process.env.PROMETHEUS_URL || 'http://prometheus:9090'
  });

  const query = async (expr, time) => {
    const response = await prometheus.get('/api/v1/query', {
      params: { query: expr, time }
    });
    return response.data.data.result;
  };

  const rangeQuery = async (expr, start, end, step) => {
    const response = await prometheus.get('/api/v1/query_range', {
      params: { query: expr, start, end, step }
    });
    return response.data.data.result;
  };

  // Current usage
  const now = Date.now() / 1000;
  const weekAgo = now - 7 * 24 * 3600;

  const [cpuUsage, memUsage, requestRate] = await Promise.all([
    query('sum(rate(container_cpu_usage_seconds_total{namespace="production"}[5m])) by (service)', now),
    query('sum(container_memory_usage_bytes{namespace="production"}) by (service)', now),
    query('sum(rate(http_requests_total{namespace="production"}[5m])) by (service)', now)
  ]);

  // Predict next 30 days
  const [cpuForecast, memForecast, requestForecast] = await Promise.all([
    query(
      'predict_linear(sum(rate(container_cpu_usage_seconds_total{namespace="production"}[5m]))[7d:1h], 86400*30)',
      now
    ),
    query(
      'predict_linear(sum(container_memory_usage_bytes{namespace="production"})[7d:1h], 86400*30)',
      now
    ),
    query(
      'predict_linear(sum(rate(http_requests_total{namespace="production"}[5m]))[7d:1h], 86400*30)',
      now
    )
  ]);

  return {
    reportDate: new Date().toISOString(),
    current: {
      cpu: formatCPU(cpuUsage),
      memory: formatMemory(memUsage),
      requestRate: formatRPS(requestRate)
    },
    forecast30Days: {
      cpu: formatCPU(cpuForecast),
      memory: formatMemory(memForecast),
      requestRate: formatRPS(requestForecast)
    },
    recommendations: generateRecommendations(cpuUsage, memUsage, cpuForecast, memForecast)
  };
}

function generateRecommendations(currentCPU, currentMem, forecastCPU, forecastMem) {
  const recommendations = [];

  // Check if we need more replicas
  const cpuPercentage = (currentCPU / totalCPUCapacity) * 100;
  if (cpuPercentage > 70) {
    recommendations.push({
      type: 'scale-up',
      resource: 'cpu',
      message: `CPU usage at ${cpuPercentage.toFixed(1)}%. Consider scaling up.`,
      priority: 'high'
    });
  }

  return recommendations;
}
```

---

## Workshop: สร้าง SLO Dashboard

```bash
# 1. Deploy Prometheus + Grafana stack
kubectl create namespace monitoring

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set grafana.adminPassword=admin123 \
  --set prometheus.prometheusSpec.retention=30d

# 2. Apply SLO rules
kubectl apply -f prometheus/slo-rules.yaml

# 3. Access Grafana
kubectl port-forward svc/kube-prometheus-stack-grafana 3000:80 -n monitoring
# http://localhost:3000 (admin/admin123)

# 4. Import SLO dashboard
# Dashboard ID: 14348 (SLO dashboard from Grafana.com)

# 5. Configure Alertmanager
kubectl create secret generic alertmanager-kube-prometheus-stack-alertmanager \
  --from-file=alertmanager.yaml=alertmanager.yaml \
  -n monitoring
```

---

## สรุป

| เรื่อง | สิ่งสำคัญ |
|-------|----------|
| **SLI** | วัดให้ตรง user journey จริงๆ |
| **SLO** | ตั้งเป้าที่ทำได้จริง ไม่ใช่ 100% |
| **Error Budget** | เครื่องมือตัดสินใจ deploy หรือ reliability work |
| **Alerting** | Alert ต้องมี action ที่ชัดเจน ไม่งั้นคือ noise |
| **Chaos Engineering** | ทดสอบ resilience ก่อนที่จะเกิดจริง |
| **Runbooks** | เตรียมการแก้ปัญหาไว้ล่วงหน้า |

**Next:** Part 20 - Performance Optimization & Caching Strategies

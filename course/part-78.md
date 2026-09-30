# Part 78: Service Mesh Advanced

## บทนำ

Service Mesh เป็นโครงสร้างพื้นฐานสำหรับการสื่อสารระหว่าง microservices ที่ให้ฟีเจอร์ต่างๆ เช่น mutual TLS, traffic management, observability และ policy enforcement โดยไม่ต้องแก้ไข application code

ในบทนี้เราจะเรียนรู้เกี่ยวกับ Service Mesh ขั้นสูง ครอบคลุม Istio, Linkerd, Consul Connect การตั้งค่า multi-cluster และการ debug

---

## 78.1 การเปรียบเทียบ Istio vs Linkerd vs Consul Connect

### ตารางเปรียบเทียบฟีเจอร์

| ฟีเจอร์ | Istio | Linkerd | Consul Connect |
|---------|-------|---------|----------------|
| **Data Plane** | Envoy Proxy | linkerd2-proxy (Rust) | Envoy Proxy |
| **Control Plane** | istiod | linkerd-controller | Consul Server |
| **mTLS** | ✅ Auto | ✅ Auto | ✅ Auto |
| **Traffic Management** | ✅ Full | ✅ Basic | ✅ Medium |
| **Circuit Breaker** | ✅ Envoy-native | ✅ Built-in | ✅ Envoy-native |
| **Retry Logic** | ✅ Advanced | ✅ Basic | ✅ Advanced |
| **Observability** | ✅ Full (Prometheus, Grafana, Jaeger) | ✅ Built-in Dashboard | ✅ Prometheus |
| **Multi-cluster** | ✅ Primary-Remote | ✅ Multi-cluster | ✅ WAN Federation |
| **VM Support** | ✅ WorkloadEntry | ❌ Limited | ✅ Full |
| **WebAssembly** | ✅ Envoy WASM | ❌ | ✅ Envoy WASM |
| **Memory Usage** | ~200-500MB control plane | ~50MB/pod | ~100-200MB |
| **CPU Overhead** | 5-10% | 2-4% | 4-8% |
| **Latency Added** | ~1-3ms | ~0.5-1ms | ~1-2ms |
| **Complexity** | High | Low-Medium | Medium |
| **Learning Curve** | Steep | Gentle | Medium |
| **Community** | Large (CNCF Graduated) | Large (CNCF Graduated) | Large (HashiCorp) |
| **License** | Apache 2.0 | Apache 2.0 | MPL 2.0 / BSL |

### ตารางเปรียบเทียบ Performance

| Metric | Istio (Envoy) | Linkerd (Rust Proxy) | Direct (No Mesh) |
|--------|---------------|---------------------|-----------------|
| P50 Latency | 1.2ms | 0.6ms | 0.3ms |
| P99 Latency | 8ms | 2ms | 1ms |
| Throughput | -8% | -3% | 0% |
| Memory per pod | ~60MB | ~20MB | 0MB |
| CPU per pod | ~0.05 core | ~0.02 core | 0 core |

### เมื่อไหรควรเลือกอะไร

- **Istio**: ต้องการฟีเจอร์ครบถ้วน, enterprise, traffic management ซับซ้อน
- **Linkerd**: ต้องการ simplicity, performance, ทีมเล็ก
- **Consul Connect**: ใช้ HashiCorp stack อยู่แล้ว, hybrid cloud, VM support

---

## 78.2 Linkerd Installation และ TypeScript Health Check Client

### ติดตั้ง Linkerd

```bash
# Install Linkerd CLI
curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/install | sh
export PATH=$HOME/.linkerd2/bin:$PATH

# Verify installation
linkerd version

# Pre-flight checks
linkerd check --pre

# Install Linkerd CRDs
linkerd install --crds | kubectl apply -f -

# Install Linkerd control plane
linkerd install | kubectl apply -f -

# Wait for installation
linkerd check

# Install Linkerd viz extension (observability)
linkerd viz install | kubectl apply -f -
linkerd viz check

# Install Linkerd multicluster extension
linkerd multicluster install | kubectl apply -f -
```

### Annotate Namespace สำหรับ Linkerd Injection

```yaml
# namespace-injection.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  annotations:
    linkerd.io/inject: enabled
    config.linkerd.io/proxy-cpu-request: "100m"
    config.linkerd.io/proxy-memory-request: "20Mi"
    config.linkerd.io/proxy-cpu-limit: "1000m"
    config.linkerd.io/proxy-memory-limit: "250Mi"
```

### TypeScript Health Check Client สำหรับ Linkerd

```typescript
// linkerd-health-client.ts
import * as http from 'http';
import * as https from 'https';

interface ServiceHealth {
  serviceName: string;
  namespace: string;
  healthy: boolean;
  successRate: number;
  p99Latency: number;
  rps: number;
  lastChecked: Date;
  meshEnabled: boolean;
  mtlsEnabled: boolean;
}

interface LinkerdMetrics {
  successRate: number;
  requestsPerSecond: number;
  p50LatencyMs: number;
  p95LatencyMs: number;
  p99LatencyMs: number;
  tcpConnections: number;
}

interface PrometheusQueryResult {
  status: string;
  data: {
    resultType: string;
    result: Array<{
      metric: Record<string, string>;
      value: [number, string];
    }>;
  };
}

class LinkerdHealthClient {
  private prometheusUrl: string;
  private linkerdVizUrl: string;
  private healthThresholds = {
    minSuccessRate: 0.99,
    maxP99LatencyMs: 1000,
    minRps: 0,
  };

  constructor(
    prometheusUrl: string = 'http://prometheus.linkerd-viz.svc.cluster.local:9090',
    linkerdVizUrl: string = 'http://web.linkerd-viz.svc.cluster.local:8084'
  ) {
    this.prometheusUrl = prometheusUrl;
    this.linkerdVizUrl = linkerdVizUrl;
  }

  private async queryPrometheus(query: string): Promise<PrometheusQueryResult> {
    const url = new URL(`${this.prometheusUrl}/api/v1/query`);
    url.searchParams.set('query', query);

    return new Promise((resolve, reject) => {
      const client = url.protocol === 'https:' ? https : http;
      
      const req = client.get(url.toString(), (res) => {
        let data = '';
        res.on('data', (chunk) => { data += chunk; });
        res.on('end', () => {
          try {
            resolve(JSON.parse(data));
          } catch (err) {
            reject(new Error(`Failed to parse Prometheus response: ${err}`));
          }
        });
      });

      req.on('error', reject);
      req.setTimeout(10000, () => {
        req.destroy();
        reject(new Error('Prometheus query timeout'));
      });
    });
  }

  async getServiceMetrics(
    serviceName: string,
    namespace: string,
    windowSeconds: number = 60
  ): Promise<LinkerdMetrics> {
    const window = `${windowSeconds}s`;
    const labelSelector = `service="${serviceName}",namespace="${namespace}"`;

    const [successRateResult, rpsResult, p99Result, p50Result, p95Result] = await Promise.all([
      // Success rate query
      this.queryPrometheus(
        `sum(rate(response_total{${labelSelector},classification="success"}[${window}])) / ` +
        `sum(rate(response_total{${labelSelector}}[${window}]))`
      ),
      // Requests per second
      this.queryPrometheus(
        `sum(rate(response_total{${labelSelector}}[${window}]))`
      ),
      // P99 latency in ms
      this.queryPrometheus(
        `histogram_quantile(0.99, sum(rate(response_latency_ms_bucket{${labelSelector}}[${window}])) by (le))`
      ),
      // P50 latency
      this.queryPrometheus(
        `histogram_quantile(0.50, sum(rate(response_latency_ms_bucket{${labelSelector}}[${window}])) by (le))`
      ),
      // P95 latency
      this.queryPrometheus(
        `histogram_quantile(0.95, sum(rate(response_latency_ms_bucket{${labelSelector}}[${window}])) by (le))`
      ),
    ]);

    const getValue = (result: PrometheusQueryResult): number => {
      const val = result.data?.result?.[0]?.value?.[1];
      return val ? parseFloat(val) : 0;
    };

    return {
      successRate: getValue(successRateResult),
      requestsPerSecond: getValue(rpsResult),
      p50LatencyMs: getValue(p50Result),
      p95LatencyMs: getValue(p95Result),
      p99LatencyMs: getValue(p99Result),
      tcpConnections: 0, // จะ query แยก
    };
  }

  async checkServiceHealth(
    serviceName: string,
    namespace: string
  ): Promise<ServiceHealth> {
    const metrics = await this.getServiceMetrics(serviceName, namespace);
    
    const healthy = 
      metrics.successRate >= this.healthThresholds.minSuccessRate &&
      metrics.p99LatencyMs <= this.healthThresholds.maxP99LatencyMs;

    return {
      serviceName,
      namespace,
      healthy,
      successRate: metrics.successRate,
      p99Latency: metrics.p99LatencyMs,
      rps: metrics.requestsPerSecond,
      lastChecked: new Date(),
      meshEnabled: true,
      mtlsEnabled: await this.checkMtlsEnabled(serviceName, namespace),
    };
  }

  private async checkMtlsEnabled(
    serviceName: string,
    namespace: string
  ): Promise<boolean> {
    try {
      const result = await this.queryPrometheus(
        `sum(rate(response_total{` +
        `service="${serviceName}",namespace="${namespace}",` +
        `tls="true"}[60s]))`
      );
      
      const totalResult = await this.queryPrometheus(
        `sum(rate(response_total{` +
        `service="${serviceName}",namespace="${namespace}"}[60s]))`
      );

      const tlsCount = parseFloat(result.data?.result?.[0]?.value?.[1] ?? '0');
      const totalCount = parseFloat(totalResult.data?.result?.[0]?.value?.[1] ?? '1');
      
      return tlsCount / totalCount > 0.95; // 95% ขึ้นไปถือว่า mTLS enabled
    } catch {
      return false;
    }
  }

  async checkMultipleServices(
    services: Array<{ name: string; namespace: string }>
  ): Promise<ServiceHealth[]> {
    const checks = services.map(svc => 
      this.checkServiceHealth(svc.name, svc.namespace).catch(err => ({
        serviceName: svc.name,
        namespace: svc.namespace,
        healthy: false,
        successRate: 0,
        p99Latency: 0,
        rps: 0,
        lastChecked: new Date(),
        meshEnabled: false,
        mtlsEnabled: false,
        error: err.message,
      } as ServiceHealth))
    );

    return Promise.all(checks);
  }

  printHealthReport(healthResults: ServiceHealth[]): void {
    console.log('\n=== Linkerd Service Health Report ===\n');
    console.log(new Date().toISOString());
    console.log('');

    const healthy = healthResults.filter(r => r.healthy);
    const unhealthy = healthResults.filter(r => !r.healthy);

    console.log(`Overall: ${healthy.length}/${healthResults.length} services healthy\n`);

    if (unhealthy.length > 0) {
      console.log('UNHEALTHY SERVICES:');
      unhealthy.forEach(svc => {
        console.log(`  [FAIL] ${svc.namespace}/${svc.serviceName}`);
        console.log(`    Success Rate: ${(svc.successRate * 100).toFixed(2)}%`);
        console.log(`    P99 Latency: ${svc.p99Latency.toFixed(1)}ms`);
        console.log(`    RPS: ${svc.rps.toFixed(1)}`);
        console.log(`    mTLS: ${svc.mtlsEnabled ? 'enabled' : 'DISABLED'}`);
      });
      console.log('');
    }

    console.log('HEALTHY SERVICES:');
    healthy.forEach(svc => {
      console.log(`  [OK] ${svc.namespace}/${svc.serviceName}`);
      console.log(`    Success Rate: ${(svc.successRate * 100).toFixed(2)}%`);
      console.log(`    P99 Latency: ${svc.p99Latency.toFixed(1)}ms`);
      console.log(`    RPS: ${svc.rps.toFixed(1)}`);
      console.log(`    mTLS: ${svc.mtlsEnabled ? 'enabled' : 'disabled'}`);
    });
  }
}

// การใช้งาน
async function main() {
  const client = new LinkerdHealthClient(
    process.env.PROMETHEUS_URL,
    process.env.LINKERD_VIZ_URL
  );

  const services = [
    { name: 'api-gateway', namespace: 'production' },
    { name: 'user-service', namespace: 'production' },
    { name: 'order-service', namespace: 'production' },
    { name: 'payment-service', namespace: 'production' },
    { name: 'notification-service', namespace: 'production' },
  ];

  const healthResults = await client.checkMultipleServices(services);
  client.printHealthReport(healthResults);

  // Exit with error code if any service is unhealthy
  const hasUnhealthy = healthResults.some(r => !r.healthy);
  process.exit(hasUnhealthy ? 1 : 0);
}

main().catch(console.error);
```

---

## 78.3 Traffic Splitting YAML (90/10 Canary) และ TypeScript Canary Controller

### Linkerd TrafficSplit YAML

```yaml
# traffic-split-canary.yaml
apiVersion: split.smi-spec.io/v1alpha2
kind: TrafficSplit
metadata:
  name: order-service-canary
  namespace: production
spec:
  service: order-service
  backends:
    - service: order-service-stable
      weight: "900m"   # 90% traffic
    - service: order-service-canary
      weight: "100m"   # 10% traffic
---
# order-service-stable deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service-stable
  namespace: production
  labels:
    app: order-service
    version: stable
    track: stable
spec:
  replicas: 9
  selector:
    matchLabels:
      app: order-service
      track: stable
  template:
    metadata:
      labels:
        app: order-service
        version: stable
        track: stable
      annotations:
        linkerd.io/inject: enabled
    spec:
      containers:
        - name: order-service
          image: myregistry/order-service:v1.2.3
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
---
# order-service-canary deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service-canary
  namespace: production
  labels:
    app: order-service
    version: canary
    track: canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: order-service
      track: canary
  template:
    metadata:
      labels:
        app: order-service
        version: canary
        track: canary
      annotations:
        linkerd.io/inject: enabled
    spec:
      containers:
        - name: order-service
          image: myregistry/order-service:v1.3.0-rc1
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
---
# Services
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: production
spec:
  selector:
    app: order-service
  ports:
    - port: 80
      targetPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: order-service-stable
  namespace: production
spec:
  selector:
    app: order-service
    track: stable
  ports:
    - port: 80
      targetPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: order-service-canary
  namespace: production
spec:
  selector:
    app: order-service
    track: canary
  ports:
    - port: 80
      targetPort: 8080
```

### TypeScript Canary Controller

```typescript
// canary-controller.ts
import * as k8s from '@kubernetes/client-node';

interface CanaryConfig {
  serviceName: string;
  namespace: string;
  stableVersion: string;
  canaryVersion: string;
  initialCanaryWeight: number;    // เริ่มที่ 10%
  incrementStep: number;          // เพิ่มทีละ 10%
  intervalMinutes: number;        // interval ระหว่าง step
  successRateThreshold: number;   // minimum success rate ก่อน promote
  maxP99LatencyMs: number;        // max p99 latency ก่อน rollback
  prometheusUrl: string;
}

interface CanaryMetrics {
  canarySuccessRate: number;
  stableSuccessRate: number;
  canaryP99LatencyMs: number;
  stableP99LatencyMs: number;
  canaryRps: number;
}

type CanaryAction = 'promote' | 'rollback' | 'continue' | 'pause';

class CanaryController {
  private k8sCustomApi: k8s.CustomObjectsApi;
  private prometheusUrl: string;

  constructor(prometheusUrl: string) {
    const kc = new k8s.KubeConfig();
    kc.loadFromDefault();
    this.k8sCustomApi = kc.makeApiClient(k8s.CustomObjectsApi);
    this.prometheusUrl = prometheusUrl;
  }

  private async getTrafficSplit(
    name: string,
    namespace: string
  ): Promise<any> {
    const response = await this.k8sCustomApi.getNamespacedCustomObject(
      'split.smi-spec.io',
      'v1alpha2',
      namespace,
      'trafficsplits',
      name
    );
    return response.body;
  }

  private async updateTrafficSplit(
    name: string,
    namespace: string,
    stableWeight: number,
    canaryWeight: number
  ): Promise<void> {
    const patch = {
      spec: {
        backends: [
          {
            service: `${name}-stable`,
            weight: `${stableWeight * 10}m`,
          },
          {
            service: `${name}-canary`,
            weight: `${canaryWeight * 10}m`,
          },
        ],
      },
    };

    await this.k8sCustomApi.patchNamespacedCustomObject(
      'split.smi-spec.io',
      'v1alpha2',
      namespace,
      'trafficsplits',
      name,
      patch,
      undefined,
      undefined,
      undefined,
      {
        headers: { 'Content-Type': 'application/merge-patch+json' },
      }
    );

    console.log(
      `Updated traffic split: stable=${stableWeight}%, canary=${canaryWeight}%`
    );
  }

  private async queryMetric(query: string): Promise<number> {
    const url = new URL(`${this.prometheusUrl}/api/v1/query`);
    url.searchParams.set('query', query);

    const response = await fetch(url.toString());
    const data = await response.json();
    
    const value = data.data?.result?.[0]?.value?.[1];
    return value ? parseFloat(value) : 0;
  }

  async getCanaryMetrics(
    config: CanaryConfig,
    windowSeconds: number = 120
  ): Promise<CanaryMetrics> {
    const window = `${windowSeconds}s`;
    const ns = config.namespace;

    const [
      canarySuccessRate,
      stableSuccessRate,
      canaryP99,
      stableP99,
      canaryRps,
    ] = await Promise.all([
      this.queryMetric(
        `sum(rate(response_total{namespace="${ns}",service="${config.serviceName}-canary",classification="success"}[${window}])) / ` +
        `sum(rate(response_total{namespace="${ns}",service="${config.serviceName}-canary"}[${window}]))`
      ),
      this.queryMetric(
        `sum(rate(response_total{namespace="${ns}",service="${config.serviceName}-stable",classification="success"}[${window}])) / ` +
        `sum(rate(response_total{namespace="${ns}",service="${config.serviceName}-stable"}[${window}]))`
      ),
      this.queryMetric(
        `histogram_quantile(0.99, sum(rate(response_latency_ms_bucket{namespace="${ns}",service="${config.serviceName}-canary"}[${window}])) by (le))`
      ),
      this.queryMetric(
        `histogram_quantile(0.99, sum(rate(response_latency_ms_bucket{namespace="${ns}",service="${config.serviceName}-stable"}[${window}])) by (le))`
      ),
      this.queryMetric(
        `sum(rate(response_total{namespace="${ns}",service="${config.serviceName}-canary"}[${window}]))`
      ),
    ]);

    return {
      canarySuccessRate,
      stableSuccessRate,
      canaryP99LatencyMs: canaryP99,
      stableP99LatencyMs: stableP99,
      canaryRps,
    };
  }

  analyzeMetrics(
    metrics: CanaryMetrics,
    config: CanaryConfig,
    currentCanaryWeight: number
  ): CanaryAction {
    // ถ้า success rate ต่ำกว่า threshold ให้ rollback
    if (metrics.canarySuccessRate < config.successRateThreshold) {
      console.log(
        `[ROLLBACK] Canary success rate ${(metrics.canarySuccessRate * 100).toFixed(2)}% < ` +
        `threshold ${(config.successRateThreshold * 100)}%`
      );
      return 'rollback';
    }

    // ถ้า p99 latency สูงเกิน threshold ให้ rollback
    if (metrics.canaryP99LatencyMs > config.maxP99LatencyMs) {
      console.log(
        `[ROLLBACK] Canary P99 latency ${metrics.canaryP99LatencyMs.toFixed(1)}ms > ` +
        `threshold ${config.maxP99LatencyMs}ms`
      );
      return 'rollback';
    }

    // ถ้า canary weight ถึง 100% แล้วให้ promote
    if (currentCanaryWeight >= 100) {
      console.log('[PROMOTE] Canary has reached 100% traffic, promoting to stable');
      return 'promote';
    }

    // ถ้าทุกอย่างดีให้ continue เพิ่ม traffic
    console.log(
      `[CONTINUE] Canary metrics OK: ` +
      `success_rate=${(metrics.canarySuccessRate * 100).toFixed(2)}%, ` +
      `p99=${metrics.canaryP99LatencyMs.toFixed(1)}ms`
    );
    return 'continue';
  }

  async runCanaryDeployment(config: CanaryConfig): Promise<void> {
    console.log(`Starting canary deployment for ${config.serviceName}`);
    console.log(`Version: ${config.stableVersion} -> ${config.canaryVersion}`);
    console.log(`Strategy: ${config.initialCanaryWeight}% initial, +${config.incrementStep}% every ${config.intervalMinutes}min\n`);

    let currentCanaryWeight = config.initialCanaryWeight;
    
    // ตั้ง initial traffic split
    await this.updateTrafficSplit(
      config.serviceName,
      config.namespace,
      100 - currentCanaryWeight,
      currentCanaryWeight
    );

    while (currentCanaryWeight <= 100) {
      // รอ metrics ให้ stable
      console.log(`\nWaiting ${config.intervalMinutes} minutes before analysis...`);
      await this.sleep(config.intervalMinutes * 60 * 1000);

      // ดึง metrics
      const metrics = await this.getCanaryMetrics(config);
      this.logMetrics(metrics, currentCanaryWeight);

      // วิเคราะห์ว่าจะทำอะไรต่อ
      const action = this.analyzeMetrics(metrics, config, currentCanaryWeight);

      switch (action) {
        case 'rollback':
          await this.rollback(config);
          return;

        case 'promote':
          await this.promote(config);
          return;

        case 'continue':
          currentCanaryWeight = Math.min(
            currentCanaryWeight + config.incrementStep,
            100
          );
          await this.updateTrafficSplit(
            config.serviceName,
            config.namespace,
            100 - currentCanaryWeight,
            currentCanaryWeight
          );
          break;

        case 'pause':
          console.log('Canary deployment paused. Manual intervention required.');
          return;
      }
    }
  }

  private async rollback(config: CanaryConfig): Promise<void> {
    console.log(`\n[ROLLBACK] Rolling back ${config.serviceName} to stable version`);
    await this.updateTrafficSplit(
      config.serviceName,
      config.namespace,
      100,
      0
    );
    console.log('Rollback complete. 100% traffic restored to stable.');
  }

  private async promote(config: CanaryConfig): Promise<void> {
    console.log(`\n[PROMOTE] Promoting canary version ${config.canaryVersion} to stable`);
    // ใน real scenario จะต้อง update stable deployment image และลบ canary
    await this.updateTrafficSplit(
      config.serviceName,
      config.namespace,
      100,
      0
    );
    console.log(`Promotion complete. ${config.canaryVersion} is now stable.`);
  }

  private logMetrics(metrics: CanaryMetrics, canaryWeight: number): void {
    console.log(`\n--- Metrics Analysis (canary=${canaryWeight}%) ---`);
    console.log(`Canary:  success=${(metrics.canarySuccessRate * 100).toFixed(2)}%, p99=${metrics.canaryP99LatencyMs.toFixed(1)}ms, rps=${metrics.canaryRps.toFixed(1)}`);
    console.log(`Stable:  success=${(metrics.stableSuccessRate * 100).toFixed(2)}%, p99=${metrics.stableP99LatencyMs.toFixed(1)}ms`);
  }

  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

// ตัวอย่างการใช้งาน
async function main() {
  const controller = new CanaryController(
    process.env.PROMETHEUS_URL || 'http://prometheus.linkerd-viz.svc:9090'
  );

  const config: CanaryConfig = {
    serviceName: 'order-service',
    namespace: 'production',
    stableVersion: 'v1.2.3',
    canaryVersion: 'v1.3.0',
    initialCanaryWeight: 10,
    incrementStep: 10,
    intervalMinutes: 10,
    successRateThreshold: 0.99,
    maxP99LatencyMs: 500,
    prometheusUrl: process.env.PROMETHEUS_URL || 'http://prometheus.linkerd-viz.svc:9090',
  };

  await controller.runCanaryDeployment(config);
}

main().catch(console.error);
```

---

## 78.4 Service Mesh Observability: Golden Signals Dashboard

### Golden Signals คืออะไร

ตาม Google SRE Book, Golden Signals ประกอบด้วย 4 อย่าง:
1. **Latency** - เวลาที่ใช้ในการตอบ request
2. **Traffic** - จำนวน requests ต่อวินาที
3. **Errors** - อัตราส่วนของ requests ที่ fail
4. **Saturation** - ความเต็มของ resource

### Prometheus Recording Rules

```yaml
# linkerd-recording-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: linkerd-golden-signals
  namespace: monitoring
  labels:
    app: kube-prometheus-stack
    release: prometheus
spec:
  groups:
    - name: linkerd.golden_signals
      interval: 30s
      rules:
        # Success Rate (1 - Error Rate)
        - record: namespace_service:success_rate:ratio_rate5m
          expr: |
            sum by (namespace, service) (
              rate(response_total{classification="success"}[5m])
            )
            /
            sum by (namespace, service) (
              rate(response_total{}[5m])
            )

        # Error Rate
        - record: namespace_service:error_rate:ratio_rate5m
          expr: |
            1 - namespace_service:success_rate:ratio_rate5m

        # Request Rate (Traffic)
        - record: namespace_service:request_rate:sum_rate5m
          expr: |
            sum by (namespace, service) (
              rate(response_total{}[5m])
            )

        # P50 Latency
        - record: namespace_service:latency_p50:histogram_quantile5m
          expr: |
            histogram_quantile(0.50,
              sum by (namespace, service, le) (
                rate(response_latency_ms_bucket{}[5m])
              )
            )

        # P95 Latency
        - record: namespace_service:latency_p95:histogram_quantile5m
          expr: |
            histogram_quantile(0.95,
              sum by (namespace, service, le) (
                rate(response_latency_ms_bucket{}[5m])
              )
            )

        # P99 Latency
        - record: namespace_service:latency_p99:histogram_quantile5m
          expr: |
            histogram_quantile(0.99,
              sum by (namespace, service, le) (
                rate(response_latency_ms_bucket{}[5m])
              )
            )

        # Saturation: TCP connection utilization
        - record: namespace_service:tcp_saturation:ratio
          expr: |
            sum by (namespace, service) (
              tcp_open_total{}
            )
            /
            sum by (namespace, service) (
              tcp_open_total{} + tcp_open_total{} * 0.2
            )
---
# Alerting Rules
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: linkerd-alerts
  namespace: monitoring
spec:
  groups:
    - name: linkerd.alerts
      rules:
        - alert: ServiceHighErrorRate
          expr: |
            namespace_service:error_rate:ratio_rate5m > 0.05
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "High error rate for {{ $labels.service }}"
            description: "Service {{ $labels.namespace }}/{{ $labels.service }} has error rate {{ $value | humanizePercentage }} for 5 minutes"

        - alert: ServiceVeryHighErrorRate
          expr: |
            namespace_service:error_rate:ratio_rate5m > 0.20
          for: 2m
          labels:
            severity: critical
          annotations:
            summary: "Critical error rate for {{ $labels.service }}"
            description: "Service {{ $labels.namespace }}/{{ $labels.service }} has error rate {{ $value | humanizePercentage }}"

        - alert: ServiceHighLatency
          expr: |
            namespace_service:latency_p99:histogram_quantile5m > 1000
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "High P99 latency for {{ $labels.service }}"
            description: "P99 latency is {{ $value | humanizeDuration }}"
```

### TypeScript Golden Signals Dashboard

```typescript
// golden-signals-dashboard.ts
interface GoldenSignals {
  service: string;
  namespace: string;
  timestamp: Date;
  latency: {
    p50Ms: number;
    p95Ms: number;
    p99Ms: number;
  };
  traffic: {
    requestsPerSecond: number;
    bytesPerSecond?: number;
  };
  errors: {
    rate: number;           // 0.0 - 1.0
    count5m: number;
    types: Record<string, number>;
  };
  saturation: {
    cpuUtilization: number; // 0.0 - 1.0
    memoryUtilization: number;
    tcpConnections: number;
  };
}

interface SLOConfig {
  errorRateTarget: number;      // e.g., 0.01 = 1% error budget
  latencyP99TargetMs: number;   // e.g., 500ms
  availabilityTarget: number;   // e.g., 0.999 = 99.9%
}

interface SLOStatus {
  service: string;
  namespace: string;
  withinSLO: boolean;
  errorBudgetRemaining: number; // percentage
  violations: string[];
}

class GoldenSignalsDashboard {
  private prometheusUrl: string;
  private sloConfigs: Map<string, SLOConfig>;

  constructor(prometheusUrl: string) {
    this.prometheusUrl = prometheusUrl;
    this.sloConfigs = new Map();
  }

  registerSLO(service: string, namespace: string, config: SLOConfig): void {
    this.sloConfigs.set(`${namespace}/${service}`, config);
  }

  private async query(promql: string): Promise<number> {
    const url = `${this.prometheusUrl}/api/v1/query?query=${encodeURIComponent(promql)}`;
    const response = await fetch(url);
    const data = await response.json();
    return parseFloat(data.data?.result?.[0]?.value?.[1] ?? '0');
  }

  async collectGoldenSignals(
    service: string,
    namespace: string
  ): Promise<GoldenSignals> {
    const label = `service="${service}",namespace="${namespace}"`;
    const window = '5m';

    const [p50, p95, p99, rps, errorRate, tcpConns] = await Promise.all([
      this.query(`histogram_quantile(0.50, sum(rate(response_latency_ms_bucket{${label}}[${window}])) by (le))`),
      this.query(`histogram_quantile(0.95, sum(rate(response_latency_ms_bucket{${label}}[${window}])) by (le))`),
      this.query(`histogram_quantile(0.99, sum(rate(response_latency_ms_bucket{${label}}[${window}])) by (le))`),
      this.query(`sum(rate(response_total{${label}}[${window}]))`),
      this.query(
        `1 - (sum(rate(response_total{${label},classification="success"}[${window}])) / ` +
        `sum(rate(response_total{${label}}[${window}])))`
      ),
      this.query(`sum(tcp_open_total{${label}})`),
    ]);

    return {
      service,
      namespace,
      timestamp: new Date(),
      latency: { p50Ms: p50, p95Ms: p95, p99Ms: p99 },
      traffic: { requestsPerSecond: rps },
      errors: {
        rate: errorRate,
        count5m: 0,
        types: {},
      },
      saturation: {
        cpuUtilization: 0,
        memoryUtilization: 0,
        tcpConnections: tcpConns,
      },
    };
  }

  checkSLO(signals: GoldenSignals): SLOStatus {
    const key = `${signals.namespace}/${signals.service}`;
    const sloConfig = this.sloConfigs.get(key);

    if (!sloConfig) {
      return {
        service: signals.service,
        namespace: signals.namespace,
        withinSLO: true,
        errorBudgetRemaining: 100,
        violations: [],
      };
    }

    const violations: string[] = [];

    if (signals.errors.rate > sloConfig.errorRateTarget) {
      violations.push(
        `Error rate ${(signals.errors.rate * 100).toFixed(3)}% exceeds target ${(sloConfig.errorRateTarget * 100)}%`
      );
    }

    if (signals.latency.p99Ms > sloConfig.latencyP99TargetMs) {
      violations.push(
        `P99 latency ${signals.latency.p99Ms.toFixed(1)}ms exceeds target ${sloConfig.latencyP99TargetMs}ms`
      );
    }

    const errorBudgetUsed = signals.errors.rate / sloConfig.errorRateTarget;
    const errorBudgetRemaining = Math.max(0, (1 - errorBudgetUsed) * 100);

    return {
      service: signals.service,
      namespace: signals.namespace,
      withinSLO: violations.length === 0,
      errorBudgetRemaining,
      violations,
    };
  }

  renderDashboard(
    signalsArray: GoldenSignals[],
    sloStatuses: SLOStatus[]
  ): string {
    const lines: string[] = [];
    
    lines.push('╔══════════════════════════════════════════════════════════════╗');
    lines.push('║           SERVICE MESH GOLDEN SIGNALS DASHBOARD             ║');
    lines.push(`║  Updated: ${new Date().toISOString()}           ║`);
    lines.push('╚══════════════════════════════════════════════════════════════╝');
    lines.push('');

    for (const signals of signalsArray) {
      const slo = sloStatuses.find(
        s => s.service === signals.service && s.namespace === signals.namespace
      );
      
      const statusIcon = slo?.withinSLO ? '✅' : '❌';
      const budgetBar = this.renderProgressBar(slo?.errorBudgetRemaining ?? 100, 20);

      lines.push(`${statusIcon} ${signals.namespace}/${signals.service}`);
      lines.push(`  LATENCY:    P50=${signals.latency.p50Ms.toFixed(1)}ms  P95=${signals.latency.p95Ms.toFixed(1)}ms  P99=${signals.latency.p99Ms.toFixed(1)}ms`);
      lines.push(`  TRAFFIC:    ${signals.traffic.requestsPerSecond.toFixed(1)} req/s`);
      lines.push(`  ERRORS:     ${(signals.errors.rate * 100).toFixed(3)}%`);
      lines.push(`  SAT TCP:    ${signals.saturation.tcpConnections} connections`);
      lines.push(`  ERR BUDGET: [${budgetBar}] ${slo?.errorBudgetRemaining.toFixed(1)}% remaining`);
      
      if (slo?.violations && slo.violations.length > 0) {
        lines.push('  VIOLATIONS:');
        slo.violations.forEach(v => lines.push(`    ⚠️  ${v}`));
      }
      lines.push('');
    }

    return lines.join('\n');
  }

  private renderProgressBar(percentage: number, width: number): string {
    const filled = Math.round((percentage / 100) * width);
    const empty = width - filled;
    const color = percentage > 50 ? '█' : percentage > 20 ? '▓' : '░';
    return color.repeat(filled) + '░'.repeat(empty);
  }
}

// ตัวอย่างการใช้งาน
async function runDashboard() {
  const dashboard = new GoldenSignalsDashboard(
    process.env.PROMETHEUS_URL || 'http://prometheus.monitoring.svc:9090'
  );

  // ลงทะเบียน SLO targets
  const services = ['api-gateway', 'user-service', 'order-service', 'payment-service'];
  const namespace = 'production';

  services.forEach(service => {
    dashboard.registerSLO(service, namespace, {
      errorRateTarget: 0.01,
      latencyP99TargetMs: 500,
      availabilityTarget: 0.999,
    });
  });

  // Collect metrics
  const signalsArray = await Promise.all(
    services.map(service => dashboard.collectGoldenSignals(service, namespace))
  );

  const sloStatuses = signalsArray.map(s => dashboard.checkSLO(s));

  // Render dashboard
  console.clear();
  console.log(dashboard.renderDashboard(signalsArray, sloStatuses));

  // Refresh every 30 seconds
  setInterval(async () => {
    const freshSignals = await Promise.all(
      services.map(service => dashboard.collectGoldenSignals(service, namespace))
    );
    const freshSLOs = freshSignals.map(s => dashboard.checkSLO(s));
    console.clear();
    console.log(dashboard.renderDashboard(freshSignals, freshSLOs));
  }, 30000);
}

runDashboard().catch(console.error);
```

---

## 78.5 mTLS Certificate Rotation Automation

### ทำไมต้อง Rotate mTLS Certificates?

- ลดความเสี่ยงจาก certificate ที่ถูก compromise
- Compliance requirements (PCI DSS, SOC2)
- Best practice สำหรับ zero-trust security

### TypeScript mTLS Certificate Rotation Script

```typescript
// mtls-cert-rotation.ts
import * as k8s from '@kubernetes/client-node';
import * as forge from 'node-forge';
import { execSync } from 'child_process';

interface CertificateInfo {
  subject: string;
  issuer: string;
  notBefore: Date;
  notAfter: Date;
  daysUntilExpiry: number;
  fingerprint: string;
  isExpired: boolean;
  willExpireSoon: boolean;
}

interface RotationResult {
  service: string;
  namespace: string;
  success: boolean;
  oldCertFingerprint: string;
  newCertFingerprint: string;
  error?: string;
}

interface MtlsRotationConfig {
  expiryWarningDays: number;       // เตือนก่อน N วัน
  forcedRotationDays: number;      // บังคับ rotate ก่อน expiry N วัน
  certValidityDays: number;        // certificate ใหม่ valid กี่วัน
  rootCASecret: string;            // K8s secret ที่เก็บ root CA
  rootCANamespace: string;
}

class MtlsCertRotationManager {
  private k8sApi: k8s.CoreV1Api;
  private config: MtlsRotationConfig;

  constructor(config: MtlsRotationConfig) {
    const kc = new k8s.KubeConfig();
    kc.loadFromDefault();
    this.k8sApi = kc.makeApiClient(k8s.CoreV1Api);
    this.config = config;
  }

  parseCertificate(certPem: string): CertificateInfo {
    const cert = forge.pki.certificateFromPem(certPem);
    
    const notAfter = cert.validity.notAfter;
    const now = new Date();
    const daysUntilExpiry = Math.floor(
      (notAfter.getTime() - now.getTime()) / (1000 * 60 * 60 * 24)
    );

    const md = forge.md.sha256.create();
    md.update(forge.asn1.toDer(forge.pki.certificateToAsn1(cert)).getBytes());

    return {
      subject: cert.subject.attributes.map(a => `${a.shortName}=${a.value}`).join(','),
      issuer: cert.issuer.attributes.map(a => `${a.shortName}=${a.value}`).join(','),
      notBefore: cert.validity.notBefore,
      notAfter,
      daysUntilExpiry,
      fingerprint: md.digest().toHex(),
      isExpired: daysUntilExpiry < 0,
      willExpireSoon: daysUntilExpiry < this.config.expiryWarningDays,
    };
  }

  async generateNewCertificate(
    serviceName: string,
    namespace: string,
    rootCACert: string,
    rootCAKey: string
  ): Promise<{ cert: string; key: string }> {
    const keys = forge.pki.rsa.generateKeyPair(2048);
    const cert = forge.pki.createCertificate();

    cert.publicKey = keys.publicKey;
    cert.serialNumber = forge.util.bytesToHex(forge.random.getBytesSync(16));
    
    const now = new Date();
    cert.validity.notBefore = now;
    cert.validity.notAfter = new Date(
      now.getTime() + this.config.certValidityDays * 24 * 60 * 60 * 1000
    );

    const attrs = [
      { name: 'commonName', value: `${serviceName}.${namespace}.svc.cluster.local` },
      { name: 'organizationName', value: 'cluster.local' },
    ];
    cert.setSubject(attrs);

    // Set issuer from root CA
    const caCert = forge.pki.certificateFromPem(rootCACert);
    cert.setIssuer(caCert.subject.attributes);

    // Extensions
    cert.setExtensions([
      { name: 'basicConstraints', cA: false },
      { name: 'keyUsage', digitalSignature: true, keyEncipherment: true },
      { name: 'extKeyUsage', serverAuth: true, clientAuth: true },
      {
        name: 'subjectAltName',
        altNames: [
          { type: 2, value: `${serviceName}.${namespace}.svc.cluster.local` },
          { type: 2, value: `${serviceName}.${namespace}.svc` },
          { type: 2, value: `${serviceName}.${namespace}` },
          { type: 2, value: serviceName },
        ],
      },
    ]);

    // Sign with root CA key
    const caKey = forge.pki.privateKeyFromPem(rootCAKey);
    cert.sign(caKey, forge.md.sha256.create());

    return {
      cert: forge.pki.certificateToPem(cert),
      key: forge.pki.privateKeyToPem(keys.privateKey),
    };
  }

  async getRootCA(): Promise<{ cert: string; key: string }> {
    const secret = await this.k8sApi.readNamespacedSecret(
      this.config.rootCASecret,
      this.config.rootCANamespace
    );

    const data = secret.body.data;
    if (!data?.['tls.crt'] || !data?.['tls.key']) {
      throw new Error('Root CA secret missing tls.crt or tls.key');
    }

    return {
      cert: Buffer.from(data['tls.crt'], 'base64').toString('utf8'),
      key: Buffer.from(data['tls.key'], 'base64').toString('utf8'),
    };
  }

  async updateCertificateSecret(
    secretName: string,
    namespace: string,
    certPem: string,
    keyPem: string
  ): Promise<void> {
    const secretData = {
      'tls.crt': Buffer.from(certPem).toString('base64'),
      'tls.key': Buffer.from(keyPem).toString('base64'),
    };

    try {
      // Try to update existing secret
      await this.k8sApi.patchNamespacedSecret(
        secretName,
        namespace,
        { data: secretData },
        undefined,
        undefined,
        undefined,
        undefined,
        undefined,
        { headers: { 'Content-Type': 'application/merge-patch+json' } }
      );
    } catch (err: any) {
      if (err.statusCode === 404) {
        // Create new secret
        await this.k8sApi.createNamespacedSecret(namespace, {
          metadata: { name: secretName, namespace },
          type: 'kubernetes.io/tls',
          data: secretData,
        });
      } else {
        throw err;
      }
    }
  }

  async checkAndRotateCertificate(
    serviceName: string,
    namespace: string,
    secretName: string
  ): Promise<RotationResult> {
    console.log(`Checking certificate for ${namespace}/${serviceName}...`);

    let needsRotation = false;
    let oldFingerprint = '';

    try {
      const secret = await this.k8sApi.readNamespacedSecret(secretName, namespace);
      const certData = secret.body.data?.['tls.crt'];
      
      if (certData) {
        const certPem = Buffer.from(certData, 'base64').toString('utf8');
        const certInfo = this.parseCertificate(certPem);
        oldFingerprint = certInfo.fingerprint;
        
        console.log(`  Current cert expires in ${certInfo.daysUntilExpiry} days`);
        
        if (certInfo.isExpired || certInfo.daysUntilExpiry < this.config.forcedRotationDays) {
          console.log(`  Rotation required: ${certInfo.isExpired ? 'EXPIRED' : `expires in ${certInfo.daysUntilExpiry} days`}`);
          needsRotation = true;
        }
      } else {
        needsRotation = true;
      }
    } catch (err: any) {
      if (err.statusCode === 404) {
        needsRotation = true;
      } else {
        throw err;
      }
    }

    if (!needsRotation) {
      console.log(`  Certificate OK, no rotation needed`);
      return {
        service: serviceName,
        namespace,
        success: true,
        oldCertFingerprint: oldFingerprint,
        newCertFingerprint: oldFingerprint,
      };
    }

    // Rotate certificate
    try {
      const rootCA = await this.getRootCA();
      const newCert = await this.generateNewCertificate(
        serviceName,
        namespace,
        rootCA.cert,
        rootCA.key
      );

      await this.updateCertificateSecret(secretName, namespace, newCert.cert, newCert.key);
      
      const newCertInfo = this.parseCertificate(newCert.cert);
      
      console.log(`  Certificate rotated successfully. New cert expires in ${newCertInfo.daysUntilExpiry} days`);

      return {
        service: serviceName,
        namespace,
        success: true,
        oldCertFingerprint: oldFingerprint,
        newCertFingerprint: newCertInfo.fingerprint,
      };
    } catch (err: any) {
      console.error(`  Rotation FAILED: ${err.message}`);
      return {
        service: serviceName,
        namespace,
        success: false,
        oldCertFingerprint: oldFingerprint,
        newCertFingerprint: '',
        error: err.message,
      };
    }
  }

  async rotateAll(
    services: Array<{ name: string; namespace: string; secretName: string }>
  ): Promise<RotationResult[]> {
    console.log(`\n=== mTLS Certificate Rotation ===`);
    console.log(`Time: ${new Date().toISOString()}\n`);

    const results = await Promise.all(
      services.map(svc => 
        this.checkAndRotateCertificate(svc.name, svc.namespace, svc.secretName)
      )
    );

    const succeeded = results.filter(r => r.success).length;
    const failed = results.filter(r => !r.success).length;

    console.log(`\n=== Summary ===`);
    console.log(`Total: ${results.length}, Success: ${succeeded}, Failed: ${failed}`);
    
    if (failed > 0) {
      console.log('\nFailed rotations:');
      results.filter(r => !r.success).forEach(r => {
        console.log(`  ${r.namespace}/${r.service}: ${r.error}`);
      });
    }

    return results;
  }
}

// การใช้งาน - ทำงานเป็น CronJob ทุกวัน
async function main() {
  const manager = new MtlsCertRotationManager({
    expiryWarningDays: 30,
    forcedRotationDays: 7,
    certValidityDays: 90,
    rootCASecret: 'linkerd-identity-issuer',
    rootCANamespace: 'linkerd',
  });

  const services = [
    { name: 'api-gateway', namespace: 'production', secretName: 'api-gateway-mtls' },
    { name: 'user-service', namespace: 'production', secretName: 'user-service-mtls' },
    { name: 'order-service', namespace: 'production', secretName: 'order-service-mtls' },
    { name: 'payment-service', namespace: 'production', secretName: 'payment-service-mtls' },
  ];

  const results = await manager.rotateAll(services);
  
  // Exit with error if any rotation failed
  process.exit(results.some(r => !r.success) ? 1 : 0);
}

main().catch(console.error);
```

---

## 78.6 Performance Overhead Benchmarking

### TypeScript Benchmark: Direct vs Mesh Calls

```typescript
// mesh-benchmark.ts
import * as http from 'http';
import * as https from 'https';

interface BenchmarkConfig {
  targetUrl: string;
  concurrency: number;
  totalRequests: number;
  warmupRequests: number;
  requestTimeoutMs: number;
}

interface RequestResult {
  durationMs: number;
  statusCode: number;
  success: boolean;
  error?: string;
}

interface BenchmarkStats {
  totalRequests: number;
  successRequests: number;
  failedRequests: number;
  successRate: number;
  throughputRps: number;
  latency: {
    min: number;
    max: number;
    mean: number;
    median: number;
    p75: number;
    p95: number;
    p99: number;
    p999: number;
  };
  totalDurationMs: number;
}

class ServiceMeshBenchmark {
  async makeRequest(url: string, timeoutMs: number): Promise<RequestResult> {
    const startTime = process.hrtime.bigint();

    return new Promise((resolve) => {
      const parsedUrl = new URL(url);
      const client = parsedUrl.protocol === 'https:' ? https : http;

      const options: http.RequestOptions = {
        hostname: parsedUrl.hostname,
        port: parsedUrl.port,
        path: parsedUrl.pathname + parsedUrl.search,
        method: 'GET',
        timeout: timeoutMs,
        headers: {
          'Connection': 'keep-alive',
          'Accept': 'application/json',
        },
      };

      const req = client.request(options, (res) => {
        let data = '';
        res.on('data', (chunk) => { data += chunk; });
        res.on('end', () => {
          const endTime = process.hrtime.bigint();
          const durationMs = Number(endTime - startTime) / 1_000_000;
          resolve({
            durationMs,
            statusCode: res.statusCode ?? 0,
            success: (res.statusCode ?? 0) >= 200 && (res.statusCode ?? 0) < 400,
          });
        });
      });

      req.on('error', (err) => {
        const endTime = process.hrtime.bigint();
        const durationMs = Number(endTime - startTime) / 1_000_000;
        resolve({
          durationMs,
          statusCode: 0,
          success: false,
          error: err.message,
        });
      });

      req.on('timeout', () => {
        req.destroy();
        resolve({
          durationMs: timeoutMs,
          statusCode: 0,
          success: false,
          error: 'Request timeout',
        });
      });

      req.end();
    });
  }

  async runBatch(
    url: string,
    count: number,
    concurrency: number,
    timeoutMs: number
  ): Promise<RequestResult[]> {
    const results: RequestResult[] = [];
    let completed = 0;

    // ใช้ semaphore สำหรับ concurrency control
    const semaphore = new Semaphore(concurrency);

    const makeRequest = async () => {
      await semaphore.acquire();
      try {
        const result = await this.makeRequest(url, timeoutMs);
        results.push(result);
        completed++;
        
        if (completed % 100 === 0) {
          process.stdout.write(`\r  Progress: ${completed}/${count}`);
        }
      } finally {
        semaphore.release();
      }
    };

    const promises: Promise<void>[] = [];
    for (let i = 0; i < count; i++) {
      promises.push(makeRequest());
    }

    await Promise.all(promises);
    process.stdout.write(`\r  Progress: ${completed}/${count}\n`);

    return results;
  }

  calculateStats(results: RequestResult[], totalDurationMs: number): BenchmarkStats {
    const successful = results.filter(r => r.success);
    const durations = successful.map(r => r.durationMs).sort((a, b) => a - b);

    const percentile = (p: number): number => {
      if (durations.length === 0) return 0;
      const index = Math.ceil((p / 100) * durations.length) - 1;
      return durations[Math.max(0, index)];
    };

    const mean = durations.length > 0
      ? durations.reduce((a, b) => a + b, 0) / durations.length
      : 0;

    return {
      totalRequests: results.length,
      successRequests: successful.length,
      failedRequests: results.length - successful.length,
      successRate: successful.length / results.length,
      throughputRps: (results.length / totalDurationMs) * 1000,
      latency: {
        min: durations[0] ?? 0,
        max: durations[durations.length - 1] ?? 0,
        mean,
        median: percentile(50),
        p75: percentile(75),
        p95: percentile(95),
        p99: percentile(99),
        p999: percentile(99.9),
      },
      totalDurationMs,
    };
  }

  async runBenchmark(
    name: string,
    config: BenchmarkConfig
  ): Promise<BenchmarkStats> {
    console.log(`\n--- Benchmark: ${name} ---`);
    console.log(`URL: ${config.targetUrl}`);
    console.log(`Requests: ${config.totalRequests}, Concurrency: ${config.concurrency}`);

    // Warmup
    if (config.warmupRequests > 0) {
      console.log(`\nWarming up (${config.warmupRequests} requests)...`);
      await this.runBatch(
        config.targetUrl,
        config.warmupRequests,
        config.concurrency,
        config.requestTimeoutMs
      );
    }

    // Benchmark
    console.log(`\nRunning benchmark...`);
    const startTime = Date.now();
    const results = await this.runBatch(
      config.targetUrl,
      config.totalRequests,
      config.concurrency,
      config.requestTimeoutMs
    );
    const totalDurationMs = Date.now() - startTime;

    return this.calculateStats(results, totalDurationMs);
  }

  compareResults(
    directStats: BenchmarkStats,
    meshStats: BenchmarkStats
  ): void {
    console.log('\n╔══════════════════════════════════════════════════════════╗');
    console.log('║          BENCHMARK COMPARISON: Direct vs Mesh           ║');
    console.log('╚══════════════════════════════════════════════════════════╝\n');

    const formatMs = (ms: number) => `${ms.toFixed(2)}ms`;
    const formatOverhead = (direct: number, mesh: number) => {
      const overhead = ((mesh - direct) / direct * 100);
      const sign = overhead >= 0 ? '+' : '';
      return `${sign}${overhead.toFixed(1)}%`;
    };

    console.log('THROUGHPUT:');
    console.log(`  Direct:  ${directStats.throughputRps.toFixed(1)} req/s`);
    console.log(`  Mesh:    ${meshStats.throughputRps.toFixed(1)} req/s`);
    console.log(`  Overhead: ${formatOverhead(directStats.throughputRps, meshStats.throughputRps)} (negative = worse)`);

    console.log('\nLATENCY:');
    const metrics: Array<[string, keyof BenchmarkStats['latency']]> = [
      ['Min', 'min'],
      ['Mean', 'mean'],
      ['Median (P50)', 'median'],
      ['P75', 'p75'],
      ['P95', 'p95'],
      ['P99', 'p99'],
      ['P99.9', 'p999'],
      ['Max', 'max'],
    ];

    metrics.forEach(([label, key]) => {
      const direct = directStats.latency[key];
      const mesh = meshStats.latency[key];
      console.log(
        `  ${label.padEnd(15)} Direct=${formatMs(direct).padEnd(10)} Mesh=${formatMs(mesh).padEnd(10)} Overhead=${formatOverhead(direct, mesh)}`
      );
    });

    console.log('\nSUCCESS RATE:');
    console.log(`  Direct:  ${(directStats.successRate * 100).toFixed(3)}%`);
    console.log(`  Mesh:    ${(meshStats.successRate * 100).toFixed(3)}%`);
  }
}

class Semaphore {
  private permits: number;
  private waiting: Array<() => void> = [];

  constructor(permits: number) {
    this.permits = permits;
  }

  async acquire(): Promise<void> {
    if (this.permits > 0) {
      this.permits--;
      return;
    }
    await new Promise<void>(resolve => this.waiting.push(resolve));
  }

  release(): void {
    if (this.waiting.length > 0) {
      const next = this.waiting.shift()!;
      next();
    } else {
      this.permits++;
    }
  }
}

// การใช้งาน
async function main() {
  const benchmark = new ServiceMeshBenchmark();

  const commonConfig = {
    concurrency: 50,
    totalRequests: 10000,
    warmupRequests: 1000,
    requestTimeoutMs: 5000,
  };

  // Benchmark direct access (no mesh proxy)
  const directStats = await benchmark.runBenchmark('Direct (No Mesh)', {
    ...commonConfig,
    targetUrl: process.env.DIRECT_URL || 'http://order-service-direct:8080/api/orders',
  });

  // Benchmark through service mesh
  const meshStats = await benchmark.runBenchmark('Through Linkerd Mesh', {
    ...commonConfig,
    targetUrl: process.env.MESH_URL || 'http://order-service:8080/api/orders',
  });

  // Print comparison
  benchmark.compareResults(directStats, meshStats);
}

main().catch(console.error);
```

---

## 78.7 Multi-cluster Istio Setup

### ServiceEntry และ Cross-cluster VirtualService

```yaml
# multi-cluster-istio.yaml
# ====================================================
# Cluster 1 (Primary) - us-east-1
# ====================================================

# ServiceEntry สำหรับ remote service ใน cluster 2
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: order-service-cluster2
  namespace: production
spec:
  hosts:
    - order-service.cluster2.global
  location: MESH_INTERNAL
  ports:
    - number: 80
      name: http
      protocol: HTTP
    - number: 443
      name: https
      protocol: HTTPS
  resolution: DNS
  addresses:
    - 240.0.0.2    # Virtual IP สำหรับ cross-cluster routing
  endpoints:
    - address: east-west-gateway.cluster2.example.com
      ports:
        http: 15443  # Istio east-west gateway port
      labels:
        topology.istio.io/cluster: cluster2
        topology.istio.io/region: ap-southeast-1
---
# VirtualService สำหรับ cross-cluster traffic management
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: order-service-global
  namespace: production
spec:
  hosts:
    - order-service
    - order-service.production.svc.cluster.local
  http:
    - name: "canary-cluster2"
      match:
        - headers:
            x-region:
              exact: "ap-southeast-1"
      route:
        - destination:
            host: order-service.cluster2.global
            port:
              number: 80
          weight: 100
    - name: "locality-aware-routing"
      route:
        - destination:
            host: order-service.production.svc.cluster.local
            port:
              number: 80
          weight: 80
          headers:
            request:
              set:
                x-forwarded-cluster: cluster1
        - destination:
            host: order-service.cluster2.global
            port:
              number: 80
          weight: 20
          headers:
            request:
              set:
                x-forwarded-cluster: cluster2
      retries:
        attempts: 3
        perTryTimeout: 5s
        retryOn: "gateway-error,connect-failure,retriable-4xx"
      timeout: 30s
---
# DestinationRule สำหรับ multi-cluster
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: order-service-global-dr
  namespace: production
spec:
  host: order-service
  trafficPolicy:
    loadBalancer:
      localityLbSetting:
        enabled: true
        distribute:
          - from: "us-east-1/*"
            to:
              "us-east-1/*": 80
              "ap-southeast-1/*": 20
          - from: "ap-southeast-1/*"
            to:
              "ap-southeast-1/*": 80
              "us-east-1/*": 20
        failover:
          - from: us-east-1
            to: ap-southeast-1
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http2MaxRequests: 1000
        maxRequestsPerConnection: 100
    outlierDetection:
      consecutiveGatewayErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
  subsets:
    - name: cluster1
      labels:
        topology.istio.io/cluster: cluster1
    - name: cluster2
      labels:
        topology.istio.io/cluster: cluster2
---
# East-West Gateway สำหรับ cross-cluster traffic
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: east-west-gateway
  namespace: istio-system
  labels:
    topology.istio.io/network: network1
spec:
  selector:
    istio: eastwestgateway
    app: istio-eastwestgateway
  servers:
    - port:
        number: 15443
        name: tls
        protocol: TLS
      tls:
        mode: AUTO_PASSTHROUGH
      hosts:
        - "*.local"
---
# IstioOperator สำหรับ multi-cluster east-west gateway
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: eastwest
  namespace: istio-system
spec:
  revision: ""
  profile: empty
  components:
    ingressGateways:
      - name: istio-eastwestgateway
        label:
          istio: eastwestgateway
          app: istio-eastwestgateway
          topology.istio.io/network: network1
        enabled: true
        k8s:
          env:
            - name: ISTIO_META_REQUESTED_NETWORK_VIEW
              value: network1
          service:
            ports:
              - name: status-port
                port: 15021
                targetPort: 15021
              - name: tls
                port: 15443
                targetPort: 15443
              - name: tls-istiod
                port: 15012
                targetPort: 15012
              - name: tls-webhook
                port: 15017
                targetPort: 15017
  values:
    gateways:
      istio-ingressgateway:
        injectionTemplate: gateway
    global:
      network: network1
```

---

## 78.8 Service Mesh Migration Strategy

### Gradual Opt-in with Namespace Injection

```yaml
# migration-strategy.yaml

# Phase 1: Enable injection per namespace (opt-in)
# เริ่มจาก non-critical namespace ก่อน
apiVersion: v1
kind: Namespace
metadata:
  name: staging
  labels:
    istio-injection: enabled       # Enable Istio injection
    # linkerd.io/inject: enabled   # หรือ Linkerd injection
  annotations:
    mesh.phase: "1"
    mesh.migrated-at: "2024-01-15"
---
# Phase 2: Production namespace injection
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    istio-injection: enabled
  annotations:
    mesh.phase: "2"
    mesh.migrated-at: "2024-02-01"
---
# PeerAuthentication: เริ่มจาก PERMISSIVE ก่อน
# ยอมรับทั้ง mTLS และ plaintext ระหว่าง migration
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: PERMISSIVE   # Phase 1: PERMISSIVE
    # mode: STRICT     # Phase 2: STRICT (หลัง migration เสร็จ)
---
# AuthorizationPolicy: เริ่มจาก ALLOW_ALL ก่อน
# ค่อยๆ เพิ่ม restrictive policy
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-all-during-migration
  namespace: production
spec:
  {} # Empty spec = allow all
  # หลัง migration เสร็จให้เปลี่ยนเป็น restrictive policy:
  # rules:
  # - from:
  #   - source:
  #       principals: ["cluster.local/ns/production/sa/*"]
---
# DestinationRule สำหรับ gradual mTLS
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: order-service-migration
  namespace: production
spec:
  host: order-service
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL     # Use Istio-managed mTLS
  # subsets แยก migrated vs non-migrated pods
  subsets:
    - name: v1-with-mesh
      labels:
        mesh: enabled
      trafficPolicy:
        tls:
          mode: ISTIO_MUTUAL
    - name: v1-no-mesh
      labels:
        mesh: disabled
      trafficPolicy:
        tls:
          mode: DISABLE
```

### TypeScript Migration Tracker

```typescript
// migration-tracker.ts
import * as k8s from '@kubernetes/client-node';

interface MigrationStatus {
  namespace: string;
  phase: number;
  injectionEnabled: boolean;
  mtlsMode: 'PERMISSIVE' | 'STRICT' | 'DISABLE' | 'UNKNOWN';
  totalPods: number;
  meshedPods: number;
  meshPercentage: number;
  readyForNextPhase: boolean;
}

class MeshMigrationTracker {
  private k8sCoreApi: k8s.CoreV1Api;
  private k8sCustomApi: k8s.CustomObjectsApi;

  constructor() {
    const kc = new k8s.KubeConfig();
    kc.loadFromDefault();
    this.k8sCoreApi = kc.makeApiClient(k8s.CoreV1Api);
    this.k8sCustomApi = kc.makeApiClient(k8s.CustomObjectsApi);
  }

  async getNamespaceMigrationStatus(namespace: string): Promise<MigrationStatus> {
    const [nsInfo, pods, peerAuth] = await Promise.all([
      this.k8sCoreApi.readNamespace(namespace),
      this.k8sCoreApi.listNamespacedPod(namespace),
      this.getPeerAuthentication(namespace),
    ]);

    const labels = nsInfo.body.metadata?.labels ?? {};
    const annotations = nsInfo.body.metadata?.annotations ?? {};

    const injectionEnabled = labels['istio-injection'] === 'enabled' ||
      labels['linkerd.io/inject'] === 'enabled';

    const totalPods = pods.body.items.length;
    const meshedPods = pods.body.items.filter(pod => {
      const podAnnotations = pod.metadata?.annotations ?? {};
      // Istio สร้าง container 'istio-proxy'; Linkerd สร้าง 'linkerd-proxy'
      const containers = pod.spec?.initContainers?.map(c => c.name) ?? [];
      return containers.includes('istio-init') || containers.includes('linkerd-init');
    }).length;

    const meshPercentage = totalPods > 0 ? (meshedPods / totalPods) * 100 : 0;

    return {
      namespace,
      phase: parseInt(annotations['mesh.phase'] ?? '0'),
      injectionEnabled,
      mtlsMode: peerAuth ?? 'UNKNOWN',
      totalPods,
      meshedPods,
      meshPercentage,
      readyForNextPhase: meshPercentage >= 95,
    };
  }

  private async getPeerAuthentication(
    namespace: string
  ): Promise<'PERMISSIVE' | 'STRICT' | 'DISABLE' | null> {
    try {
      const result = await this.k8sCustomApi.getNamespacedCustomObject(
        'security.istio.io',
        'v1beta1',
        namespace,
        'peerauthentications',
        'default'
      ) as any;

      return result.body?.spec?.mtls?.mode ?? null;
    } catch {
      return null;
    }
  }

  async reportMigrationStatus(namespaces: string[]): Promise<void> {
    console.log('\n=== Service Mesh Migration Status ===\n');

    const statuses = await Promise.all(
      namespaces.map(ns => this.getNamespaceMigrationStatus(ns))
    );

    statuses.forEach(status => {
      const pct = status.meshPercentage.toFixed(0);
      const bar = '█'.repeat(Math.floor(status.meshPercentage / 5)).padEnd(20, '░');
      const ready = status.readyForNextPhase ? '✅' : '⏳';
      
      console.log(`Namespace: ${status.namespace} (Phase ${status.phase}) ${ready}`);
      console.log(`  Injection: ${status.injectionEnabled ? 'enabled' : 'disabled'}`);
      console.log(`  mTLS Mode: ${status.mtlsMode}`);
      console.log(`  Meshed Pods: ${status.meshedPods}/${status.totalPods}`);
      console.log(`  Progress: [${bar}] ${pct}%`);
      console.log('');
    });
  }
}

async function main() {
  const tracker = new MeshMigrationTracker();
  await tracker.reportMigrationStatus([
    'development',
    'staging',
    'production',
  ]);
}

main().catch(console.error);
```

---

## 78.9 Hybrid Mesh: VM WorkloadEntry

### WorkloadEntry สำหรับ VM Registration

```yaml
# vm-workload.yaml
# สำหรับ VM ที่อยู่นอก Kubernetes cluster

# WorkloadEntry ลงทะเบียน VM ใน mesh
apiVersion: networking.istio.io/v1beta1
kind: WorkloadEntry
metadata:
  name: legacy-db-server
  namespace: production
  annotations:
    # VM's IP address
    proxy.istio.io/config: |
      holdApplicationUntilProxyStarts: true
spec:
  address: 10.0.0.50          # VM's private IP
  labels:
    app: legacy-db
    version: v1
    instance: vm-us-east-1a
    topology.istio.io/network: vm-network
  serviceAccount: legacy-db-sa
  network: vm-network
  locality: us-east-1a
  ports:
    mysql:
      number: 3306
      protocol: TCP
    redis:
      number: 6379
      protocol: TCP
---
# ServiceEntry สร้าง mesh representation สำหรับ VM
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: legacy-db-service
  namespace: production
spec:
  hosts:
    - legacy-db.production.svc.cluster.local
  ports:
    - number: 3306
      name: mysql
      protocol: TCP
    - number: 6379
      name: redis
      protocol: TCP
  location: MESH_INTERNAL
  resolution: STATIC
  workloadSelector:
    labels:
      app: legacy-db
---
# Sidecar resource สำหรับ VM
# กำหนด traffic scope สำหรับ VM proxy
apiVersion: networking.istio.io/v1beta1
kind: Sidecar
metadata:
  name: legacy-db-sidecar
  namespace: production
spec:
  workloadSelector:
    labels:
      app: legacy-db
  ingress:
    - port:
        number: 3306
        protocol: TCP
        name: mysql
      defaultEndpoint: "127.0.0.1:3306"
    - port:
        number: 6379
        protocol: TCP
        name: redis
      defaultEndpoint: "127.0.0.1:6379"
  egress:
    - hosts:
        - "production/*"
        - "istio-system/*"
---
# WorkloadGroup ใช้สำหรับ auto-registration ของ VM fleet
apiVersion: networking.istio.io/v1beta1
kind: WorkloadGroup
metadata:
  name: legacy-db-group
  namespace: production
spec:
  metadata:
    labels:
      app: legacy-db
      version: v1
    annotations:
      proxy.istio.io/config: |
        concurrency: 2
  template:
    ports:
      mysql:
        number: 3306
        protocol: TCP
    serviceAccount: legacy-db-sa
    network: vm-network
  probe:
    initialDelaySeconds: 5
    timeoutSeconds: 3
    periodSeconds: 30
    successThreshold: 1
    failureThreshold: 3
    tcpSocket:
      port: 3306
```

### Script ติดตั้ง Istio Proxy บน VM

```bash
#!/bin/bash
# install-vm-proxy.sh
# รันบน VM ที่ต้องการเข้า mesh

set -euo pipefail

ISTIO_VERSION="1.20.0"
CLUSTER_NAME="production"
CLUSTER_NETWORK="vm-network"
ISTIO_NAMESPACE="istio-system"
SERVICE_NAMESPACE="production"
SERVICE_ACCOUNT="legacy-db-sa"

echo "=== Installing Istio Proxy on VM ==="

# 1. ดาวน์โหลด Istio
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=${ISTIO_VERSION} sh -
export PATH=$PWD/istio-${ISTIO_VERSION}/bin:$PATH

# 2. สร้าง token สำหรับ VM authentication
kubectl create token ${SERVICE_ACCOUNT} \
  --namespace ${SERVICE_NAMESPACE} \
  --duration=8760h \
  > /tmp/vm-token.txt

# 3. ดึง root certificate
kubectl -n ${ISTIO_NAMESPACE} get configmap istio-ca-root-cert \
  -o jsonpath='{.data.root-cert\.pem}' > /tmp/root-cert.pem

# 4. สร้าง cluster env file
cat > /tmp/cluster.env << EOF
ISTIO_SERVICE_NAMESPACE=${SERVICE_NAMESPACE}
ISTIO_SERVICE=legacy-db
ISTIO_PILOT_AGENT_OPTS="--domain ${SERVICE_NAMESPACE}.svc.cluster.local"
CLUSTER_ENV_CLUSTER=${CLUSTER_NAME}
EOF

# 5. ติดตั้ง proxy
sudo mkdir -p /etc/istio/proxy /etc/certs /var/lib/istio/envoy

# Copy files
sudo cp /tmp/root-cert.pem /etc/certs/root-cert.pem
sudo cp /tmp/vm-token.txt /var/lib/istio/envoy/sds-grpc.token
sudo cp /tmp/cluster.env /etc/istio/proxy/cluster.env

# 6. ติดตั้ง Debian package
curl -LO https://storage.googleapis.com/istio-release/releases/${ISTIO_VERSION}/deb/istio-sidecar.deb
sudo dpkg -i istio-sidecar.deb

# 7. เริ่ม istio.service
sudo systemctl enable istio
sudo systemctl start istio

echo "VM proxy installed and started successfully"
sudo systemctl status istio
```

---

## 78.10 Debugging with istioctl

### คำสั่ง Debug ที่สำคัญ

```bash
# ============================================
# istioctl analyze - ตรวจสอบ mesh configuration
# ============================================

# วิเคราะห์ namespace เดียว
istioctl analyze --namespace production

# วิเคราะห์ทั้ง cluster
istioctl analyze --all-namespaces

# วิเคราะห์ไฟล์ YAML ก่อน apply
istioctl analyze -f my-virtualservice.yaml

# Output แบบ JSON
istioctl analyze --namespace production -o json

# ============================================
# proxy-status - ดู sync status ของ proxy
# ============================================

# ดู sync status ทั้งหมด
istioctl proxy-status

# ดู specific pod
istioctl proxy-status <pod-name>.<namespace>

# ============================================
# proxy-config - ดู Envoy configuration
# ============================================

# ดู listeners ของ pod
istioctl proxy-config listeners <pod-name> -n <namespace>

# ดู routes
istioctl proxy-config routes <pod-name> -n <namespace>

# ดู clusters
istioctl proxy-config clusters <pod-name> -n <namespace>

# ดู endpoints
istioctl proxy-config endpoints <pod-name> -n <namespace>

# ดู secrets (mTLS certs)
istioctl proxy-config secret <pod-name> -n <namespace>

# ดู bootstrap config
istioctl proxy-config bootstrap <pod-name> -n <namespace>

# ============================================
# xtoproxy-status - ดู xDS sync
# ============================================
istioctl experimental proxy-status

# ============================================
# debug - ดู debug info
# ============================================
istioctl debug <pod-name> -n <namespace> --port 15000

# ============================================
# Envoy admin API โดยตรง
# ============================================

# Port-forward Envoy admin
kubectl port-forward <pod-name> 15000:15000 -n <namespace>

# ดู stats
curl http://localhost:15000/stats

# ดู config_dump
curl http://localhost:15000/config_dump | jq .

# ดู clusters
curl http://localhost:15000/clusters

# ดู server info
curl http://localhost:15000/server_info

# Reset stats counter
curl -X POST http://localhost:15000/reset_counters
```

### TypeScript Istio Debug Client

```typescript
// istio-debug-client.ts
import { execSync, ExecSyncOptionsWithStringEncoding } from 'child_process';

interface ProxyStatus {
  name: string;
  namespace: string;
  clustersStatus: string;
  listenersStatus: string;
  routesStatus: string;
  endpointsStatus: string;
  version: string;
}

interface AnalysisMessage {
  code: string;
  level: 'Error' | 'Warning' | 'Info';
  message: string;
  origin?: string;
  reference?: string;
}

interface AnalysisResult {
  messages: AnalysisMessage[];
  errors: AnalysisMessage[];
  warnings: AnalysisMessage[];
}

class IstioctlDebugClient {
  private execOptions: ExecSyncOptionsWithStringEncoding = {
    encoding: 'utf8',
    timeout: 30000,
  };

  private exec(command: string): string {
    try {
      return execSync(command, this.execOptions).trim();
    } catch (err: any) {
      throw new Error(`Command failed: ${command}\n${err.stderr ?? err.message}`);
    }
  }

  analyzeNamespace(namespace: string): AnalysisResult {
    const output = this.exec(
      `istioctl analyze --namespace ${namespace} -o json 2>/dev/null`
    );
    
    let messages: AnalysisMessage[] = [];
    try {
      const parsed = JSON.parse(output);
      messages = (parsed.messages ?? []).map((m: any) => ({
        code: m.code,
        level: m.level,
        message: m.message,
        origin: m.origin,
        reference: m.reference,
      }));
    } catch {
      // Parse failed, try line by line
    }

    return {
      messages,
      errors: messages.filter(m => m.level === 'Error'),
      warnings: messages.filter(m => m.level === 'Warning'),
    };
  }

  getProxyStatus(podName?: string, namespace?: string): ProxyStatus[] {
    const cmd = podName
      ? `istioctl proxy-status ${podName}.${namespace ?? 'default'} -o json`
      : `istioctl proxy-status -o json`;

    const output = this.exec(cmd);
    
    try {
      const data = JSON.parse(output);
      return (Array.isArray(data) ? data : [data]).map((item: any) => ({
        name: item.name ?? item.podName ?? '',
        namespace: item.namespace ?? '',
        clustersStatus: item.clusters ?? 'UNKNOWN',
        listenersStatus: item.listeners ?? 'UNKNOWN',
        routesStatus: item.routes ?? 'UNKNOWN',
        endpointsStatus: item.endpoints ?? 'UNKNOWN',
        version: item.version ?? '',
      }));
    } catch {
      return [];
    }
  }

  getProxyListeners(podName: string, namespace: string): any[] {
    const output = this.exec(
      `istioctl proxy-config listeners ${podName}.${namespace} -o json`
    );
    
    try {
      return JSON.parse(output);
    } catch {
      return [];
    }
  }

  getProxyClusters(podName: string, namespace: string): any[] {
    const output = this.exec(
      `istioctl proxy-config clusters ${podName}.${namespace} -o json`
    );
    
    try {
      return JSON.parse(output);
    } catch {
      return [];
    }
  }

  getProxyEndpoints(podName: string, namespace: string): any[] {
    const output = this.exec(
      `istioctl proxy-config endpoints ${podName}.${namespace} -o json`
    );
    
    try {
      return JSON.parse(output);
    } catch {
      return [];
    }
  }

  checkMtlsStatus(
    sourcePod: string,
    sourceNs: string,
    destinationService: string,
    destinationNs: string
  ): string {
    return this.exec(
      `istioctl authn tls-check ${sourcePod}.${sourceNs} ` +
      `${destinationService}.${destinationNs}.svc.cluster.local`
    );
  }

  runDiagnostics(namespace: string): void {
    console.log(`\n=== Istio Diagnostics for namespace: ${namespace} ===\n`);

    // 1. Analyze
    console.log('1. Configuration Analysis:');
    try {
      const analysis = this.analyzeNamespace(namespace);
      if (analysis.errors.length === 0 && analysis.warnings.length === 0) {
        console.log('   No issues found');
      } else {
        analysis.errors.forEach(e => 
          console.log(`   [ERROR] ${e.code}: ${e.message}`)
        );
        analysis.warnings.forEach(w => 
          console.log(`   [WARN] ${w.code}: ${w.message}`)
        );
      }
    } catch (err) {
      console.log(`   Failed: ${err}`);
    }

    // 2. Proxy sync status
    console.log('\n2. Proxy Sync Status:');
    try {
      const statuses = this.getProxyStatus();
      const nsStatuses = statuses.filter(s => s.namespace === namespace);
      
      if (nsStatuses.length === 0) {
        console.log('   No proxies found');
      } else {
        nsStatuses.forEach(s => {
          const allSynced = [
            s.clustersStatus,
            s.listenersStatus,
            s.routesStatus,
            s.endpointsStatus,
          ].every(status => status === 'SYNCED');
          
          const icon = allSynced ? '✅' : '❌';
          console.log(`   ${icon} ${s.name}`);
          if (!allSynced) {
            console.log(`      Clusters: ${s.clustersStatus}`);
            console.log(`      Listeners: ${s.listenersStatus}`);
            console.log(`      Routes: ${s.routesStatus}`);
            console.log(`      Endpoints: ${s.endpointsStatus}`);
          }
        });
      }
    } catch (err) {
      console.log(`   Failed: ${err}`);
    }

    console.log('\n=== Diagnostics Complete ===');
  }
}

// การใช้งาน
async function main() {
  const client = new IstioctlDebugClient();
  
  const namespace = process.argv[2] || 'production';
  client.runDiagnostics(namespace);
}

main().catch(console.error);
```

---

## สรุปบทที่ 78

| หัวข้อ | สิ่งที่เรียนรู้ |
|--------|----------------|
| Service Mesh Comparison | Istio vs Linkerd vs Consul Connect - feature, performance, complexity |
| Linkerd Setup | Installation, namespace injection, TypeScript health check client |
| Canary Deployment | TrafficSplit YAML 90/10, TypeScript canary controller |
| Golden Signals | Error rate, latency, traffic, saturation dashboard |
| mTLS Rotation | Certificate lifecycle, auto-rotation TypeScript script |
| Performance Benchmark | Direct vs mesh overhead measurement |
| Multi-cluster Istio | ServiceEntry, cross-cluster VirtualService, east-west gateway |
| Migration Strategy | Gradual opt-in, PERMISSIVE → STRICT mTLS |
| VM Integration | WorkloadEntry, WorkloadGroup, VM proxy install |
| Debugging | istioctl analyze, proxy-status, proxy-config |

### Key Takeaways

1. **เลือก Service Mesh ให้เหมาะกับทีม**: Linkerd สำหรับ simplicity, Istio สำหรับ enterprise features
2. **ทำ Canary Deployment ด้วย TrafficSplit**: ลด risk ของ deployment ใหม่
3. **Monitor Golden Signals เสมอ**: Latency, Traffic, Errors, Saturation
4. **Rotate mTLS Certificates อัตโนมัติ**: ลด security risk
5. **Migration ควรทำแบบ Gradual**: PERMISSIVE ก่อน STRICT
6. **Debug ด้วย istioctl**: analyze, proxy-status, proxy-config เป็นเครื่องมือหลัก

---

*Part 78 จบแล้ว - ต่อไปบทที่ 79: Scalability Patterns*

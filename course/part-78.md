# Part 78: Service Mesh Advanced

## บทนำ

Service Mesh เป็น infrastructure layer ที่จัดการ service-to-service communication ใน Microservices บทนี้ครอบคลุมการเปรียบเทียบ Istio vs Linkerd vs Consul Connect, configuration, traffic management, observability, mTLS และ advanced patterns

---

## 1. Istio vs Linkerd vs Consul Connect Comparison

```typescript
// service-mesh-comparison.ts

interface ServiceMeshFeature {
  feature: string;
  istio: string;
  linkerd: string;
  consulConnect: string;
  importance: 'HIGH' | 'MEDIUM' | 'LOW';
}

const comparison: ServiceMeshFeature[] = [
  {
    feature: 'Control Plane',
    istio: 'istiod (monolithic)',
    linkerd: 'linkerd-control-plane',
    consulConnect: 'Consul server',
    importance: 'HIGH',
  },
  {
    feature: 'Data Plane',
    istio: 'Envoy proxy',
    linkerd: 'linkerd2-proxy (Rust)',
    consulConnect: 'Envoy proxy',
    importance: 'HIGH',
  },
  {
    feature: 'Resource Overhead',
    istio: 'High (~50MB per sidecar)',
    linkerd: 'Low (~10MB per sidecar)',
    consulConnect: 'Medium (~30MB)',
    importance: 'HIGH',
  },
  {
    feature: 'mTLS',
    istio: 'Auto (with SPIFFE)',
    linkerd: 'Auto (zero-config)',
    consulConnect: 'Auto (with Connect)',
    importance: 'HIGH',
  },
  {
    feature: 'Traffic Management',
    istio: 'Advanced (VirtualService, DestinationRule)',
    linkerd: 'Basic-Medium (HTTPRoute)',
    consulConnect: 'Medium (service router)',
    importance: 'HIGH',
  },
  {
    feature: 'Observability',
    istio: 'Advanced (metrics, traces, logs)',
    linkerd: 'Good (metrics, traces)',
    consulConnect: 'Good (metrics)',
    importance: 'HIGH',
  },
  {
    feature: 'Learning Curve',
    istio: 'Steep',
    linkerd: 'Gentle',
    consulConnect: 'Medium',
    importance: 'MEDIUM',
  },
  {
    feature: 'Multi-cluster',
    istio: 'Yes (advanced)',
    linkerd: 'Yes (multicluster)',
    consulConnect: 'Yes (WAN federation)',
    importance: 'MEDIUM',
  },
  {
    feature: 'VM Support',
    istio: 'Yes',
    linkerd: 'Limited',
    consulConnect: 'Yes (native)',
    importance: 'MEDIUM',
  },
  {
    feature: 'External Authorization',
    istio: 'Yes (OPA, ext_authz)',
    linkerd: 'Limited',
    consulConnect: 'Yes (Intentions)',
    importance: 'MEDIUM',
  },
];

// Recommendation engine
function recommendServiceMesh(requirements: {
  teamExpertise: 'beginner' | 'intermediate' | 'expert';
  performanceSensitive: boolean;
  vmWorkloads: boolean;
  advancedTrafficManagement: boolean;
  multiCluster: boolean;
}): string {
  let score = { istio: 0, linkerd: 0, consulConnect: 0 };

  if (requirements.teamExpertise === 'beginner') {
    score.linkerd += 3;
    score.consulConnect += 1;
  } else if (requirements.teamExpertise === 'expert') {
    score.istio += 2;
  }

  if (requirements.performanceSensitive) {
    score.linkerd += 3;
    score.istio -= 1;
  }

  if (requirements.vmWorkloads) {
    score.consulConnect += 3;
    score.istio += 1;
  }

  if (requirements.advancedTrafficManagement) {
    score.istio += 3;
  }

  if (requirements.multiCluster) {
    score.istio += 2;
    score.consulConnect += 2;
    score.linkerd += 1;
  }

  const winner = Object.entries(score).reduce(
    (max, [name, s]) => (s > max.score ? { name, score: s } : max),
    { name: '', score: -1 }
  );

  return winner.name;
}
```

---

## 2. Linkerd Setup and Configuration

```bash
# ติดตั้ง Linkerd CLI
curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/install | sh
export PATH=$PATH:$HOME/.linkerd2/bin

# ตรวจสอบ cluster compatibility
linkerd check --pre

# ติดตั้ง Linkerd control plane
linkerd install --crds | kubectl apply -f -
linkerd install | kubectl apply -f -

# ตรวจสอบการติดตั้ง
linkerd check

# ติดตั้ง Linkerd viz extension
linkerd viz install | kubectl apply -f -
linkerd viz check

# ติดตั้ง Linkerd multicluster
linkerd multicluster install | kubectl apply -f -
```

```yaml
# linkerd-namespace-annotation.yaml
# Enable Linkerd injection สำหรับ namespace
apiVersion: v1
kind: Namespace
metadata:
  name: microservices
  annotations:
    linkerd.io/inject: enabled
---
# ServiceProfile สำหรับ Linkerd
apiVersion: linkerd.io/v1alpha2
kind: ServiceProfile
metadata:
  name: order-service.microservices.svc.cluster.local
  namespace: microservices
spec:
  routes:
    - name: POST /orders
      condition:
        method: POST
        pathRegex: /orders
      responseClasses:
        - condition:
            status:
              min: 500
              max: 599
          isFailure: true
      timeout: 5s
      retryBudget:
        retryRatio: 0.2
        minRetriesPerSecond: 10
        ttl: 10s
    
    - name: GET /orders/{id}
      condition:
        method: GET
        pathRegex: /orders/[^/]*$
      isRetryable: true
      timeout: 2s
    
    - name: GET /health
      condition:
        method: GET
        pathRegex: /health
      isRetryable: true
      timeout: 1s
---
# TrafficSplit สำหรับ Canary Deployment
apiVersion: split.smi-spec.io/v1alpha1
kind: TrafficSplit
metadata:
  name: order-service-canary
  namespace: microservices
spec:
  service: order-service
  backends:
    - service: order-service-stable
      weight: 90
    - service: order-service-canary
      weight: 10
```

```typescript
// linkerd-service-profile.ts
// TypeScript definition สำหรับ Linkerd ServiceProfile

interface LinkerdServiceProfile {
  apiVersion: 'linkerd.io/v1alpha2';
  kind: 'ServiceProfile';
  metadata: {
    name: string; // service.namespace.svc.cluster.local
    namespace: string;
  };
  spec: {
    routes: Route[];
    retryBudget?: RetryBudget;
    opaquePorts?: number[];
  };
}

interface Route {
  name: string;
  condition: RouteCondition;
  timeout?: string;
  isRetryable?: boolean;
  responseClasses?: ResponseClass[];
  retryBudget?: RetryBudget;
}

interface RouteCondition {
  method?: string;
  pathRegex?: string;
  any?: RouteCondition[];
  all?: RouteCondition[];
}

interface ResponseClass {
  condition: {
    status?: { min: number; max: number };
    header?: { name: string; value?: string };
  };
  isFailure: boolean;
}

interface RetryBudget {
  retryRatio: number;
  minRetriesPerSecond: number;
  ttl: string;
}

// Generator สำหรับ ServiceProfile
function generateServiceProfile(
  serviceName: string,
  namespace: string,
  routes: { path: string; method: string; timeout: string; retryable: boolean }[]
): LinkerdServiceProfile {
  return {
    apiVersion: 'linkerd.io/v1alpha2',
    kind: 'ServiceProfile',
    metadata: {
      name: `${serviceName}.${namespace}.svc.cluster.local`,
      namespace,
    },
    spec: {
      routes: routes.map(route => ({
        name: `${route.method} ${route.path}`,
        condition: {
          method: route.method,
          pathRegex: route.path.replace(/\{[^}]+\}/g, '[^/]*'),
        },
        timeout: route.timeout,
        isRetryable: route.retryable,
        responseClasses: [
          {
            condition: { status: { min: 500, max: 599 } },
            isFailure: true,
          },
        ],
      })),
      retryBudget: {
        retryRatio: 0.2,
        minRetriesPerSecond: 10,
        ttl: '10s',
      },
    },
  };
}
```

---

## 3. Traffic Splitting and Canary Deployment

```typescript
// traffic-splitting.ts
// Progressive delivery with service mesh

import { KubernetesObject } from '@kubernetes/client-node';

interface CanaryConfig {
  serviceName: string;
  namespace: string;
  stableVersion: string;
  canaryVersion: string;
  initialWeight: number; // % traffic to canary
  maxWeight: number;
  increment: number;
  incrementInterval: number; // minutes
  metrics: CanaryMetric[];
}

interface CanaryMetric {
  name: string;
  threshold: number;
  interval: string;
  query: string;
}

class CanaryDeploymentManager {
  async progressiveRollout(config: CanaryConfig): Promise<void> {
    let currentWeight = config.initialWeight;
    
    console.log(`Starting canary rollout: ${config.canaryVersion}`);
    console.log(`Initial traffic split: ${currentWeight}% to canary`);

    // Initial traffic split
    await this.updateTrafficSplit(config.serviceName, config.namespace, {
      stable: 100 - currentWeight,
      canary: currentWeight,
    });

    while (currentWeight < config.maxWeight) {
      // Wait for analysis interval
      await this.wait(config.incrementInterval * 60000);

      // Analyze metrics
      const healthy = await this.analyzeMetrics(config.metrics);

      if (!healthy) {
        console.log('Canary unhealthy - rolling back!');
        await this.rollback(config);
        return;
      }

      // Increment traffic
      currentWeight = Math.min(currentWeight + config.increment, config.maxWeight);
      
      console.log(`Incrementing canary traffic to ${currentWeight}%`);
      
      await this.updateTrafficSplit(config.serviceName, config.namespace, {
        stable: 100 - currentWeight,
        canary: currentWeight,
      });
    }

    // Full rollout
    if (currentWeight >= config.maxWeight) {
      console.log('Canary deployment successful - promoting to stable');
      await this.promoteCanary(config);
    }
  }

  private async updateTrafficSplit(
    serviceName: string,
    namespace: string,
    weights: { stable: number; canary: number }
  ): Promise<void> {
    // Update Kubernetes TrafficSplit resource
    const trafficSplit = {
      apiVersion: 'split.smi-spec.io/v1alpha1',
      kind: 'TrafficSplit',
      metadata: {
        name: `${serviceName}-split`,
        namespace,
      },
      spec: {
        service: serviceName,
        backends: [
          {
            service: `${serviceName}-stable`,
            weight: weights.stable,
          },
          {
            service: `${serviceName}-canary`,
            weight: weights.canary,
          },
        ],
      },
    };

    // Apply using kubectl or Kubernetes API
    console.log('Applying traffic split:', JSON.stringify(trafficSplit.spec.backends));
  }

  private async analyzeMetrics(metrics: CanaryMetric[]): Promise<boolean> {
    for (const metric of metrics) {
      const value = await this.queryPrometheus(metric.query);
      
      if (metric.name.includes('error_rate') && value > metric.threshold) {
        console.error(`Metric ${metric.name} exceeded threshold: ${value} > ${metric.threshold}`);
        return false;
      }
      
      if (metric.name.includes('latency') && value > metric.threshold) {
        console.error(`Metric ${metric.name} exceeded threshold: ${value}ms > ${metric.threshold}ms`);
        return false;
      }
    }
    
    return true;
  }

  private async queryPrometheus(query: string): Promise<number> {
    const response = await fetch(
      `http://prometheus:9090/api/v1/query?query=${encodeURIComponent(query)}`
    );
    const data = await response.json() as {
      data: { result: [{ value: [number, string] }] }
    };
    return parseFloat(data.data.result[0]?.value[1] || '0');
  }

  private async rollback(config: CanaryConfig): Promise<void> {
    await this.updateTrafficSplit(config.serviceName, config.namespace, {
      stable: 100,
      canary: 0,
    });
    console.log('Rolled back to stable version');
  }

  private async promoteCanary(config: CanaryConfig): Promise<void> {
    // Route all traffic to canary (which becomes new stable)
    await this.updateTrafficSplit(config.serviceName, config.namespace, {
      stable: 0,
      canary: 100,
    });
    console.log('Canary promoted to stable');
  }

  private wait(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

// Flagger-style canary configuration
const flaggerCanary = `
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: order-service
  namespace: microservices
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  
  service:
    port: 3000
    targetPort: 3000
    gateways:
      - public-gateway.istio-system.svc.cluster.local
    hosts:
      - order-service.company.com
  
  analysis:
    interval: 2m
    threshold: 5
    maxWeight: 50
    stepWeight: 10
    metrics:
      - name: request-success-rate
        thresholdRange:
          min: 99
        interval: 1m
      - name: request-duration
        thresholdRange:
          max: 500
        interval: 30s
    
    webhooks:
      - name: acceptance-test
        type: pre-rollout
        url: http://flagger-loadtester.test/
        metadata:
          type: bash
          cmd: "curl -sd 'test' http://order-service-canary:3000/health | grep ok"
      
      - name: load-test
        url: http://flagger-loadtester.test/
        metadata:
          type: cmd
          cmd: "hey -z 2m -q 10 -c 2 http://order-service-canary:3000/health"
`;
```

---

## 4. Service Mesh Observability

```typescript
// mesh-observability.ts
// Observability ใน Service Mesh

interface MeshMetrics {
  successRate: number;     // Percentage
  requestRate: number;     // RPS
  p50Latency: number;      // ms
  p95Latency: number;      // ms
  p99Latency: number;      // ms
  tcpOpenConnections: number;
}

class ServiceMeshObservability {
  private prometheusUrl: string;

  constructor(prometheusUrl: string) {
    this.prometheusUrl = prometheusUrl;
  }

  async getServiceMetrics(
    service: string,
    namespace: string,
    window: string = '5m'
  ): Promise<MeshMetrics> {
    const queries = {
      successRate: `
        sum(rate(response_total{
          namespace="${namespace}",
          dst_service="${service}",
          classification="success"
        }[${window}]))
        /
        sum(rate(response_total{
          namespace="${namespace}",
          dst_service="${service}"
        }[${window}]))
        * 100
      `,
      requestRate: `
        sum(rate(response_total{
          namespace="${namespace}",
          dst_service="${service}"
        }[${window}]))
      `,
      p50Latency: `
        histogram_quantile(0.50,
          sum(rate(response_latency_ms_bucket{
            namespace="${namespace}",
            dst_service="${service}"
          }[${window}])) by (le)
        )
      `,
      p95Latency: `
        histogram_quantile(0.95,
          sum(rate(response_latency_ms_bucket{
            namespace="${namespace}",
            dst_service="${service}"
          }[${window}])) by (le)
        )
      `,
      p99Latency: `
        histogram_quantile(0.99,
          sum(rate(response_latency_ms_bucket{
            namespace="${namespace}",
            dst_service="${service}"
          }[${window}])) by (le)
        )
      `,
    };

    const results = await Promise.all(
      Object.entries(queries).map(async ([metric, query]) => {
        const value = await this.queryPrometheus(query.trim());
        return [metric, value] as [string, number];
      })
    );

    return Object.fromEntries(results) as unknown as MeshMetrics;
  }

  private async queryPrometheus(query: string): Promise<number> {
    const url = `${this.prometheusUrl}/api/v1/query?query=${encodeURIComponent(query)}`;
    const response = await fetch(url);
    const data = await response.json() as {
      data: { result: [{ value: [number, string] }] }
    };
    return parseFloat(data.data.result[0]?.value[1] || '0');
  }

  async getTopologyGraph(namespace: string): Promise<{
    nodes: Array<{ id: string; name: string; namespace: string }>;
    edges: Array<{ source: string; target: string; rps: number; errorRate: number }>;
  }> {
    // Get traffic flow data
    const response = await fetch(
      `${this.prometheusUrl}/api/v1/query?query=` +
      encodeURIComponent(
        `sum(rate(response_total{namespace="${namespace}"}[5m])) by (src_service, dst_service)`
      )
    );
    
    const data = await response.json() as {
      data: {
        result: Array<{
          metric: { src_service: string; dst_service: string };
          value: [number, string];
        }>
      }
    };

    const nodes = new Map<string, { id: string; name: string; namespace: string }>();
    const edges: Array<{ source: string; target: string; rps: number; errorRate: number }> = [];

    for (const item of data.data.result) {
      const src = item.metric.src_service;
      const dst = item.metric.dst_service;
      
      if (src && !nodes.has(src)) {
        nodes.set(src, { id: src, name: src, namespace });
      }
      if (dst && !nodes.has(dst)) {
        nodes.set(dst, { id: dst, name: dst, namespace });
      }

      edges.push({
        source: src,
        target: dst,
        rps: parseFloat(item.value[1]),
        errorRate: 0, // Fetch separately
      });
    }

    return {
      nodes: [...nodes.values()],
      edges,
    };
  }
}
```

---

## 5. mTLS Certificate Management

```yaml
# mtls-config.yaml
# Istio mTLS configuration

# Enable strict mTLS for entire mesh
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
---
# Allow specific services to use PERMISSIVE mode during migration
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: legacy-service-permissive
  namespace: microservices
spec:
  selector:
    matchLabels:
      app: legacy-service
  mtls:
    mode: PERMISSIVE  # Allow both mTLS and plain text
---
# DestinationRule for mTLS
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: order-service-mtls
  namespace: microservices
spec:
  host: order-service
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL  # Auto mTLS using Istio certificates
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
```

```typescript
// cert-rotation.ts
// Certificate rotation สำหรับ service mesh

interface CertificateInfo {
  commonName: string;
  issuer: string;
  validFrom: Date;
  validTo: Date;
  daysUntilExpiry: number;
  thumbprint: string;
}

class CertificateManager {
  async getCertificateInfo(podName: string, namespace: string): Promise<CertificateInfo> {
    // ดึง certificate จาก istio-proxy
    const certPem = await this.execInSidecar(
      podName,
      namespace,
      'istio-proxy',
      ['cat', '/var/run/secrets/istio/root-cert.pem']
    );

    return this.parseCertificate(certPem);
  }

  private parseCertificate(pem: string): CertificateInfo {
    // Parse PEM certificate
    // In practice, use x509 library
    const now = new Date();
    const validTo = new Date(now.getTime() + 90 * 24 * 60 * 60 * 1000); // 90 days
    
    return {
      commonName: 'order-service',
      issuer: 'istiod.istio-system.svc',
      validFrom: now,
      validTo,
      daysUntilExpiry: 90,
      thumbprint: 'sha256:...',
    };
  }

  async monitorCertExpiry(namespace: string, warningDays: number = 30): Promise<void> {
    const pods = await this.listPods(namespace);
    
    for (const pod of pods) {
      const certInfo = await this.getCertificateInfo(pod.name, namespace);
      
      if (certInfo.daysUntilExpiry <= warningDays) {
        console.warn(
          `Certificate for ${pod.name} expires in ${certInfo.daysUntilExpiry} days!`
        );
        // Trigger rotation
        await this.triggerCertRotation(pod.name, namespace);
      }
    }
  }

  private async triggerCertRotation(podName: string, namespace: string): Promise<void> {
    // Istio auto-rotates certificates, but we can force it
    console.log(`Triggering cert rotation for ${podName} in ${namespace}`);
    // kubectl rollout restart deployment/podName -n namespace
  }

  private async execInSidecar(
    podName: string,
    namespace: string,
    container: string,
    command: string[]
  ): Promise<string> {
    return ''; // Implementation via kubectl exec
  }

  private async listPods(namespace: string): Promise<Array<{ name: string }>> {
    return []; // Implementation via Kubernetes API
  }
}
```

---

## 6. Service Mesh Performance Overhead

```typescript
// mesh-performance.ts
// วัดและ optimize performance overhead ของ service mesh

interface PerformanceBenchmark {
  withoutMesh: {
    latencyP50: number;
    latencyP95: number;
    latencyP99: number;
    throughput: number;
    cpuOverhead: number;
    memoryOverhead: number;
  };
  withMesh: {
    latencyP50: number;
    latencyP95: number;
    latencyP99: number;
    throughput: number;
    cpuOverhead: number;
    memoryOverhead: number;
  };
}

function calculateOverhead(benchmark: PerformanceBenchmark): {
  latencyOverheadMs: { p50: number; p95: number; p99: number };
  throughputImpact: number; // percentage
  resourceOverhead: { cpu: string; memory: string };
} {
  return {
    latencyOverheadMs: {
      p50: benchmark.withMesh.latencyP50 - benchmark.withoutMesh.latencyP50,
      p95: benchmark.withMesh.latencyP95 - benchmark.withoutMesh.latencyP95,
      p99: benchmark.withMesh.latencyP99 - benchmark.withoutMesh.latencyP99,
    },
    throughputImpact:
      ((benchmark.withoutMesh.throughput - benchmark.withMesh.throughput) /
        benchmark.withoutMesh.throughput) * 100,
    resourceOverhead: {
      cpu: `${benchmark.withMesh.cpuOverhead - benchmark.withoutMesh.cpuOverhead}m`,
      memory: `${benchmark.withMesh.memoryOverhead - benchmark.withoutMesh.memoryOverhead}Mi`,
    },
  };
}

// Typical overhead numbers (for reference)
const typicalBenchmark: PerformanceBenchmark = {
  withoutMesh: {
    latencyP50: 2,
    latencyP95: 5,
    latencyP99: 10,
    throughput: 10000,
    cpuOverhead: 0,
    memoryOverhead: 0,
  },
  withMesh: {
    // Linkerd overhead (lighter)
    latencyP50: 3,    // +1ms
    latencyP95: 7,    // +2ms
    latencyP99: 14,   // +4ms
    throughput: 9500, // -5%
    cpuOverhead: 10,  // 10m per sidecar
    memoryOverhead: 20, // 20Mi per sidecar
  },
};

// Performance tuning configurations
const istioPerformanceTuning = `
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: istio-performance
spec:
  meshConfig:
    # Reduce Envoy resource usage
    defaultConfig:
      concurrency: 2  # Number of worker threads
      proxyStatsMatcher:
        inclusionRegexps:
          - ".*circuit_breakers.*"
          - ".*_rq_.*"
          - ".*_cx_.*"
  
  components:
    pilot:
      k8s:
        resources:
          requests:
            cpu: 500m
            memory: 2Gi
          limits:
            cpu: 1000m
            memory: 4Gi
    
    ingressGateways:
      - name: istio-ingressgateway
        k8s:
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 2000m
              memory: 1Gi
          autoscaleMin: 1
          autoscaleMax: 5
          autoscaleEnabled: true
`;
```

---

## 7. Multi-cluster Service Mesh

```yaml
# multi-cluster-istio.yaml
# Multi-cluster Istio configuration

# Primary cluster (us-east-1)
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: istio-primary
spec:
  meshConfig:
    defaultConfig:
      proxyMetadata:
        ISTIO_META_DNS_CAPTURE: "true"
        ISTIO_META_DNS_AUTO_ALLOCATE: "true"
  
  values:
    pilot:
      env:
        EXTERNAL_ISTIOD: false
    
    global:
      meshID: mesh1
      multiCluster:
        clusterName: us-east-cluster
      network: us-east-network
---
# Remote cluster (ap-southeast-1)
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: istio-remote
spec:
  profile: remote
  
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: ap-southeast-cluster
      network: ap-southeast-network
      remotePilotAddress: "34.xxx.xxx.xxx"  # Primary cluster istiod address
---
# ServiceEntry to expose remote service
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: order-service-remote
  namespace: microservices
spec:
  hosts:
    - order-service.microservices.global
  ports:
    - number: 3000
      name: http
      protocol: HTTP
  resolution: STATIC
  endpoints:
    - address: 10.100.0.1
      network: ap-southeast-network
      labels:
        version: v1
        cluster: ap-southeast-cluster
      ports:
        http: 3000
```

```typescript
// multi-cluster-routing.ts
// Route traffic across clusters

const multiClusterVirtualService = {
  apiVersion: 'networking.istio.io/v1beta1',
  kind: 'VirtualService',
  metadata: {
    name: 'order-service-global',
    namespace: 'microservices',
  },
  spec: {
    hosts: ['order-service'],
    http: [
      {
        // Route AP users to AP cluster
        match: [
          {
            headers: {
              'x-user-region': { exact: 'ap-southeast' },
            },
          },
        ],
        route: [
          {
            destination: {
              host: 'order-service.microservices.global',
              subset: 'ap-southeast',
            },
            weight: 100,
          },
        ],
      },
      {
        // Default route to US cluster
        route: [
          {
            destination: {
              host: 'order-service',
              subset: 'v1',
            },
            weight: 100,
          },
        ],
      },
    ],
  },
};

// Locality-based load balancing
const destinationRuleWithLocality = {
  apiVersion: 'networking.istio.io/v1beta1',
  kind: 'DestinationRule',
  metadata: {
    name: 'order-service-locality',
    namespace: 'microservices',
  },
  spec: {
    host: 'order-service',
    trafficPolicy: {
      loadBalancer: {
        localityLbSetting: {
          enabled: true,
          distribute: [
            {
              from: 'us-east1/*',
              to: {
                'us-east1/*': 80,
                'us-central1/*': 20,
              },
            },
            {
              from: 'ap-southeast1/*',
              to: {
                'ap-southeast1/*': 90,
                'ap-northeast1/*': 10,
              },
            },
          ],
          failover: [
            { from: 'us-east1', to: 'us-central1' },
            { from: 'ap-southeast1', to: 'ap-northeast1' },
          ],
        },
      },
    },
  },
};
```

---

## 8. Service Mesh Migration Strategies

```typescript
// mesh-migration.ts
// กลยุทธ์การ migrate ไปยัง service mesh

interface MigrationPhase {
  phase: number;
  name: string;
  services: string[];
  tasks: string[];
  validationChecks: string[];
  rollbackPlan: string;
}

const migrationPlan: MigrationPhase[] = [
  {
    phase: 1,
    name: 'Pilot - Non-critical Services',
    services: ['notification-service', 'analytics-service'],
    tasks: [
      'Install Linkerd control plane',
      'Annotate namespace with injection',
      'Deploy services with sidecar',
      'Enable mTLS in PERMISSIVE mode',
      'Monitor for 1 week',
    ],
    validationChecks: [
      'Service health checks pass',
      'No latency regression > 10ms',
      'mTLS working between services',
      'Metrics visible in Grafana',
    ],
    rollbackPlan: 'Remove namespace annotation, restart pods',
  },
  {
    phase: 2,
    name: 'Core Services',
    services: ['order-service', 'inventory-service', 'payment-service'],
    tasks: [
      'Verify pilot phase learnings',
      'Enable strict mTLS for pilot services',
      'Inject core services',
      'Configure circuit breakers',
      'Set up retry policies',
    ],
    validationChecks: [
      'All service communications use mTLS',
      'Circuit breakers functioning',
      'Error rates stable',
      'P95 latency within SLO',
    ],
    rollbackPlan: 'Switch back to PERMISSIVE mode, disable injection',
  },
  {
    phase: 3,
    name: 'Full Mesh + Traffic Management',
    services: ['*'],
    tasks: [
      'Inject remaining services',
      'Configure traffic splitting for canary deployments',
      'Enable distributed tracing',
      'Set up ServiceProfiles',
      'Configure timeout policies',
      'Enable strict mTLS mesh-wide',
    ],
    validationChecks: [
      'Topology map shows all services',
      'Traffic splitting working',
      'Distributed traces visible',
      'All SLOs met',
    ],
    rollbackPlan: 'Phase rollback by service group',
  },
];

// Migration validation
async function validateMigrationPhase(phase: MigrationPhase): Promise<{
  passed: boolean;
  failures: string[];
}> {
  const failures: string[] = [];

  for (const check of phase.validationChecks) {
    const result = await runValidationCheck(check);
    if (!result.passed) {
      failures.push(`${check}: ${result.reason}`);
    }
  }

  return { passed: failures.length === 0, failures };
}

async function runValidationCheck(check: string): Promise<{ passed: boolean; reason: string }> {
  // Implement each check
  return { passed: true, reason: '' };
}
```

---

## 9. Hybrid Mesh (VM + Kubernetes)

```yaml
# hybrid-mesh-vm.yaml
# เพิ่ม VM เข้า Istio Mesh

# WorkloadEntry สำหรับ VM
apiVersion: networking.istio.io/v1beta1
kind: WorkloadEntry
metadata:
  name: legacy-db-service
  namespace: microservices
spec:
  address: "192.168.1.100"  # VM IP
  ports:
    http: 8080
    grpc: 9090
  serviceAccount: legacy-service-account
  labels:
    app: legacy-db-service
    version: v1
    instance-id: vm-instance-001
---
# ServiceEntry for VM workloads
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: legacy-db-service-se
  namespace: microservices
spec:
  hosts:
    - legacy-db-service.microservices.svc.cluster.local
  location: MESH_INTERNAL
  ports:
    - number: 8080
      name: http
      protocol: HTTP
  resolution: STATIC
  workloadSelector:
    labels:
      app: legacy-db-service
```

```bash
# Install Istio on VM
# ดึง token จาก Kubernetes cluster
kubectl -n microservices create token legacy-service-account \
  --duration=24h > /tmp/istio-token

# บน VM
curl -LO https://storage.googleapis.com/istio-release/releases/1.20.0/istioctl
chmod +x istioctl

# Generate VM config
./istioctl x workload entry configure \
  --file workload.yaml \
  --output /tmp/vm-config/ \
  --clusterID "Kubernetes" \
  --ingressIP "34.xxx.xxx.xxx"

# ติดตั้ง Istio agent บน VM
# (copy files from /tmp/vm-config/ to VM)
sudo cp /tmp/vm-config/hosts /etc/hosts
sudo cp /tmp/vm-config/istio-token /var/run/secrets/tokens/
sudo cp /tmp/vm-config/mesh.yaml /etc/istio/config/

# Start Istio agent
sudo systemctl start istio
sudo systemctl enable istio
```

---

## 10. Service Mesh Debugging Tools

```typescript
// mesh-debugging.ts
// เครื่องมือ debug service mesh

class MeshDebugger {
  async diagnoseServiceConnectivity(
    sourceService: string,
    targetService: string,
    namespace: string
  ): Promise<{
    healthy: boolean;
    issues: string[];
    recommendations: string[];
  }> {
    const issues: string[] = [];
    const recommendations: string[] = [];

    // Check mTLS configuration
    const mtlsStatus = await this.checkMTLSConfig(sourceService, targetService, namespace);
    if (!mtlsStatus.configured) {
      issues.push('mTLS not properly configured');
      recommendations.push('Apply PeerAuthentication and DestinationRule');
    }

    // Check authorization policies
    const authzStatus = await this.checkAuthorizationPolicies(sourceService, targetService, namespace);
    if (!authzStatus.allowed) {
      issues.push(`Service ${sourceService} not authorized to call ${targetService}`);
      recommendations.push(`Add AuthorizationPolicy allowing ${sourceService} service account`);
    }

    // Check VirtualService configuration
    const vsStatus = await this.checkVirtualService(targetService, namespace);
    if (!vsStatus.valid) {
      issues.push('VirtualService configuration issues');
      recommendations.push(vsStatus.recommendation || 'Review VirtualService configuration');
    }

    return {
      healthy: issues.length === 0,
      issues,
      recommendations,
    };
  }

  private async checkMTLSConfig(
    source: string,
    target: string,
    namespace: string
  ): Promise<{ configured: boolean; mode: string }> {
    // Check using istioctl proxy-config
    console.log(`Checking mTLS between ${source} -> ${target}`);
    return { configured: true, mode: 'STRICT' };
  }

  private async checkAuthorizationPolicies(
    source: string,
    target: string,
    namespace: string
  ): Promise<{ allowed: boolean; reason?: string }> {
    console.log(`Checking AuthorizationPolicies for ${source} -> ${target}`);
    return { allowed: true };
  }

  private async checkVirtualService(
    service: string,
    namespace: string
  ): Promise<{ valid: boolean; recommendation?: string }> {
    console.log(`Checking VirtualService for ${service}`);
    return { valid: true };
  }

  async getEnvoyConfig(podName: string, namespace: string): Promise<void> {
    // istioctl proxy-config all <pod>.<namespace>
    console.log(`Fetching Envoy config for ${podName}.${namespace}`);
    // Execute: kubectl exec -n <namespace> <pod> -c istio-proxy -- pilot-agent request GET /config_dump
  }

  async traceRequest(
    sourceService: string,
    targetService: string,
    namespace: string
  ): Promise<void> {
    // Send test request and trace
    console.log(`Tracing request: ${sourceService} -> ${targetService}`);
    // Use Jaeger or Zipkin to trace
  }
}

// Istio debugging commands
const debugCommands = {
  checkProxy: 'istioctl proxy-status',
  validateConfig: 'istioctl analyze -n microservices',
  checkMTLS: 'istioctl x check-inject -n microservices',
  proxyConfig: 'istioctl proxy-config cluster <pod>.<namespace>',
  authorizationCheck: 'istioctl x authz check <pod>.<namespace>',
  getMetrics: 'kubectl exec -n microservices <pod> -c istio-proxy -- curl localhost:15090/metrics',
};
```

---

## Advanced Istio Traffic Management

```yaml
# istio-advanced-traffic.yaml
# Advanced traffic management features

# Circuit Breaker via DestinationRule
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: order-service-circuit-breaker
  namespace: microservices
spec:
  host: order-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
        connectTimeout: 30s
      http:
        http2MaxRequests: 1000
        maxRequestsPerConnection: 10
        maxRetries: 3
        retryOn: "5xx,reset,connect-failure,retriable-4xx"
    
    outlierDetection:
      consecutiveGatewayErrors: 5
      consecutiveLocalOriginFailures: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
      minHealthPercent: 30
---
# Fault Injection สำหรับ Chaos Testing
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: order-service-chaos
  namespace: microservices
spec:
  hosts:
    - order-service
  http:
    # Inject 5% error rate
    - fault:
        abort:
          percentage:
            value: 5
          httpStatus: 503
      match:
        - headers:
            x-chaos-test:
              exact: "enabled"
      route:
        - destination:
            host: order-service
    
    # Inject 2 second delay for 10% of requests
    - fault:
        delay:
          percentage:
            value: 10
          fixedDelay: 2s
      match:
        - headers:
            x-chaos-test:
              exact: "latency"
      route:
        - destination:
            host: order-service
    
    # Normal routing
    - route:
        - destination:
            host: order-service
            subset: stable
          weight: 90
        - destination:
            host: order-service
            subset: canary
          weight: 10
```

---

## สรุปตาราง Service Mesh

| Feature | Istio | Linkerd | Consul Connect |
|---------|-------|---------|----------------|
| Data Plane | Envoy | linkerd2-proxy (Rust) | Envoy |
| Memory/sidecar | ~50MB | ~10MB | ~30MB |
| CPU overhead | Medium | Low | Medium |
| mTLS | Auto/SPIFFE | Auto | Auto/Intentions |
| Traffic Management | Advanced | Basic-Medium | Medium |
| Learning Curve | Steep | Gentle | Medium |
| Multi-cluster | Yes | Yes | Yes |
| VM Support | Yes | Limited | Yes (native) |
| Dashboard | Kiali | Linkerd Viz | Consul UI |
| Commercial Support | Yes (Tetrate) | Yes (Buoyant) | Yes (HashiCorp) |

| Traffic Pattern | Istio Config | Use Case |
|----------------|-------------|---------|
| Canary Deployment | VirtualService weight | New version testing |
| A/B Testing | Header-based routing | Feature testing |
| Circuit Breaker | DestinationRule outlierDetection | Failure isolation |
| Retry | VirtualService retries | Transient failures |
| Fault Injection | VirtualService fault | Chaos testing |
| Traffic Mirror | VirtualService mirror | Shadow testing |
| Rate Limiting | EnvoyFilter/Ratelimit | DDoS protection |

---

## 78.11 Istio Circuit Breaker และ Outlier Detection

### DestinationRule สำหรับ Circuit Breaker

```yaml
# circuit-breaker-dr.yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: payment-service-circuit-breaker
  namespace: production
spec:
  host: payment-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
        connectTimeout: 5s
        tcpKeepalive:
          time: 7200s
          interval: 75s
      http:
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
        maxRequestsPerConnection: 10
        maxRetries: 3
        idleTimeout: 90s
        h2UpgradePolicy: UPGRADE
    outlierDetection:
      # Circuit breaker triggers
      consecutiveGatewayErrors: 5
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
      minHealthPercent: 50
      # Panic threshold - ถ้า healthy hosts ต่ำกว่า 50% จะ load balance ทุก host
      splitExternalLocalOriginErrors: true
      consecutiveLocalOriginFailures: 5
```

### TypeScript Circuit Breaker Monitor

```typescript
// circuit-breaker-monitor.ts
interface CircuitBreakerState {
  service: string;
  namespace: string;
  ejectedHosts: string[];
  totalHosts: number;
  healthyHosts: number;
  state: 'closed' | 'open' | 'half-open';
  lastStateChange: Date;
  consecutiveErrors: number;
}

class CircuitBreakerMonitor {
  private prometheusUrl: string;

  constructor(prometheusUrl: string) {
    this.prometheusUrl = prometheusUrl;
  }

  async getCircuitBreakerStates(namespace: string): Promise<CircuitBreakerState[]> {
    const url = `${this.prometheusUrl}/api/v1/query?query=` +
      encodeURIComponent(
        `envoy_cluster_outlier_detection_ejections_active{namespace="${namespace}"}`
      );

    const response = await fetch(url);
    const data = await response.json();

    return (data.data?.result ?? []).map((result: any) => {
      const ejectedCount = parseInt(result.value[1]);
      const service = result.metric.cluster_name?.replace('outbound|80||', '').split('.')[0] ?? '';
      
      return {
        service,
        namespace,
        ejectedHosts: [],
        totalHosts: 0,
        healthyHosts: 0,
        state: ejectedCount > 0 ? 'open' : 'closed',
        lastStateChange: new Date(),
        consecutiveErrors: ejectedCount,
      } as CircuitBreakerState;
    });
  }

  async watchAndAlert(namespace: string, intervalMs: number = 30000): Promise<void> {
    console.log(`Monitoring circuit breakers in namespace: ${namespace}`);
    
    const check = async () => {
      const states = await this.getCircuitBreakerStates(namespace);
      
      states.filter(s => s.state !== 'closed').forEach(state => {
        console.log(
          `[CIRCUIT BREAKER OPEN] ${state.namespace}/${state.service}: ` +
          `${state.consecutiveErrors} consecutive errors`
        );
      });
    };

    await check();
    setInterval(check, intervalMs);
  }
}

export { CircuitBreakerMonitor };
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
| Circuit Breaker | DestinationRule outlierDetection, TypeScript monitor |

| Feature | Istio | Linkerd | Consul Connect |
|---------|-------|---------|----------------|
| Data Plane | Envoy | linkerd2-proxy (Rust) | Envoy |
| Memory/sidecar | ~50MB | ~10MB | ~30MB |
| CPU overhead | Medium | Low | Medium |
| mTLS | Auto/SPIFFE | Auto | Auto/Intentions |
| Traffic Management | Advanced | Basic-Medium | Medium |
| Learning Curve | Steep | Gentle | Medium |
| Multi-cluster | Yes | Yes | Yes |
| VM Support | Yes | Limited | Yes (native) |
| Dashboard | Kiali | Linkerd Viz | Consul UI |
| Commercial Support | Yes (Tetrate) | Yes (Buoyant) | Yes (HashiCorp) |

| Traffic Pattern | Istio Config | Use Case |
|----------------|-------------|---------|
| Canary Deployment | VirtualService weight | New version testing |
| A/B Testing | Header-based routing | Feature testing |
| Circuit Breaker | DestinationRule outlierDetection | Failure isolation |
| Retry | VirtualService retries | Transient failures |
| Fault Injection | VirtualService fault | Chaos testing |
| Traffic Mirror | VirtualService mirror | Shadow testing |
| Rate Limiting | EnvoyFilter/Ratelimit | DDoS protection |

---

## สรุป

Service Mesh Advanced Patterns:

1. **Linkerd** เหมาะสำหรับ teams ที่ต้องการ simple setup และ low overhead
2. **Istio** เหมาะสำหรับ advanced traffic management และ enterprise features
3. **Canary Deployment** ด้วย TrafficSplit รับประกัน zero-downtime releases
4. **mTLS** auto-configure ทุก service-to-service communication
5. **Multi-cluster** ช่วย global deployment และ disaster recovery
6. **Performance Tuning** ลด overhead ด้วย proper sizing และ concurrency settings
7. **VM Integration** รองรับ hybrid environments
8. **Debugging** ด้วย istioctl/linkerd CLI และ distributed tracing

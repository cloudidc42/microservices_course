# Part 44: Service Mesh ขั้นสูงด้วย Istio

## บทนำ

Service Mesh คือ infrastructure layer ที่จัดการ service-to-service communication ใน microservices architecture โดยไม่ต้องแก้ไข application code Istio เป็น service mesh ที่ได้รับความนิยมสูงสุด ใช้ Envoy proxy เป็น data plane และ control plane ที่มีฟีเจอร์ครบครัน

ในบทนี้เราจะเรียนรู้:
- Traffic splitting สำหรับ canary deployments
- Fault injection testing เพื่อทดสอบ resilience
- Retry/timeout policies ใน VirtualService
- Authorization policies สำหรับ zero-trust security
- Envoy filter configuration
- Distributed tracing ด้วย Jaeger/Zipkin
- DestinationRule load balancing
- mTLS peer authentication
- Traffic mirroring (shadowing)
- Ingress gateway TLS termination

## 1. การติดตั้ง Istio

### 1.1 ติดตั้ง Istio ด้วย IstioOperator

```yaml
# istio-operator.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: production-istio
  namespace: istio-system
spec:
  profile: production
  meshConfig:
    defaultConfig:
      proxyStatsMatcher:
        inclusionRegexps:
          - ".*outlier_detection.*"
          - ".*upstream_rq_retry.*"
          - ".*upstream_cx_active.*"
    enableAutoMtls: true
    enablePrometheusMerge: true
    defaultProviders:
      tracing:
        - name: "jaeger"
    extensionProviders:
      - name: jaeger
        zipkin:
          service: "jaeger-collector.monitoring.svc.cluster.local"
          port: 9411
          maxTagLength: 256
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
        hpaSpec:
          minReplicas: 2
          maxReplicas: 5
          metrics:
            - type: Resource
              resource:
                name: cpu
                target:
                  type: Utilization
                  averageUtilization: 80
    ingressGateways:
      - name: istio-ingressgateway
        enabled: true
        k8s:
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 2000m
              memory: 1024Mi
          hpaSpec:
            minReplicas: 2
            maxReplicas: 10
          service:
            type: LoadBalancer
            ports:
              - port: 80
                targetPort: 8080
                name: http2
              - port: 443
                targetPort: 8443
                name: https
              - port: 15443
                targetPort: 15443
                name: tls
  values:
    global:
      proxy:
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 2000m
            memory: 1024Mi
      tracer:
        zipkin:
          address: jaeger-collector.monitoring:9411
    pilot:
      traceSampling: 1.0
```

```bash
# ติดตั้ง Istio
istioctl install -f istio-operator.yaml --verify

# Enable sidecar injection สำหรับ namespace
kubectl label namespace production istio-injection=enabled

# ตรวจสอบ installation
istioctl verify-install
istioctl analyze --all-namespaces
```

### 1.2 Namespace และ Application Setup

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    istio-injection: enabled
    environment: production
---
apiVersion: v1
kind: Namespace
metadata:
  name: staging
  labels:
    istio-injection: enabled
    environment: staging
```

## 2. Traffic Splitting สำหรับ Canary Deployments

### 2.1 Deployment สำหรับ Stable และ Canary Versions

```yaml
# product-service-stable.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-service-v1
  namespace: production
  labels:
    app: product-service
    version: v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: product-service
      version: v1
  template:
    metadata:
      labels:
        app: product-service
        version: v1
    spec:
      containers:
        - name: product-service
          image: myregistry/product-service:1.0.0
          ports:
            - containerPort: 3000
          env:
            - name: VERSION
              value: "v1"
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: url
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health/live
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 10
---
# product-service-canary.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-service-v2
  namespace: production
  labels:
    app: product-service
    version: v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: product-service
      version: v2
  template:
    metadata:
      labels:
        app: product-service
        version: v2
    spec:
      containers:
        - name: product-service
          image: myregistry/product-service:2.0.0
          ports:
            - containerPort: 3000
          env:
            - name: VERSION
              value: "v2"
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: url
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi
---
# kubernetes-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: product-service
  namespace: production
  labels:
    app: product-service
spec:
  selector:
    app: product-service
  ports:
    - name: http
      port: 80
      targetPort: 3000
      protocol: TCP
```

### 2.2 DestinationRule และ VirtualService สำหรับ Traffic Splitting

```yaml
# destination-rule.yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: product-service-dr
  namespace: production
spec:
  host: product-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
        connectTimeout: 30ms
        tcpKeepalive:
          time: 7200s
          interval: 75s
      http:
        http1MaxPendingRequests: 1000
        http2MaxRequests: 10000
        maxRequestsPerConnection: 100
        h2UpgradePolicy: UPGRADE
    loadBalancer:
      simple: LEAST_CONN
    outlierDetection:
      consecutiveGatewayErrors: 5
      consecutive5xxErrors: 5
      interval: 10s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
      minHealthPercent: 30
    retryPolicy:
      attempts: 3
      perTryTimeout: 5s
      retryOn: "5xx,reset,connect-failure,retriable-4xx"
  subsets:
    - name: v1
      labels:
        version: v1
      trafficPolicy:
        loadBalancer:
          simple: ROUND_ROBIN
    - name: v2
      labels:
        version: v2
      trafficPolicy:
        loadBalancer:
          simple: RANDOM
---
# virtual-service-canary.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: product-service-vs
  namespace: production
spec:
  hosts:
    - product-service
    - product.example.com
  gateways:
    - production/main-gateway
    - mesh
  http:
    # Canary rule: route beta users to v2
    - match:
        - headers:
            x-user-group:
              exact: beta
          uri:
            prefix: /api/v1/products
      route:
        - destination:
            host: product-service
            subset: v2
            port:
              number: 80
      timeout: 10s
      retries:
        attempts: 3
        perTryTimeout: 3s
        retryOn: "5xx,reset,connect-failure"
    # Weight-based canary: 90% stable, 10% canary
    - match:
        - uri:
            prefix: /api/v1/products
      route:
        - destination:
            host: product-service
            subset: v1
            port:
              number: 80
          weight: 90
        - destination:
            host: product-service
            subset: v2
            port:
              number: 80
          weight: 10
      timeout: 10s
      retries:
        attempts: 3
        perTryTimeout: 3s
        retryOn: "5xx,reset,connect-failure"
    # Default route
    - route:
        - destination:
            host: product-service
            subset: v1
            port:
              number: 80
```

### 2.3 TypeScript Canary Deployment Controller

```typescript
// canary-controller.ts
import * as k8s from "@kubernetes/client-node";
import { CustomObjectsApi } from "@kubernetes/client-node";

interface CanaryConfig {
  namespace: string;
  serviceName: string;
  stableVersion: string;
  canaryVersion: string;
  initialWeight: number;
  incrementStep: number;
  maxWeight: number;
  analysisInterval: number; // minutes
  successThreshold: number; // error rate threshold %
}

interface CanaryMetrics {
  errorRate: number;
  latencyP99: number;
  requestsPerSecond: number;
}

async function getCanaryMetrics(
  prometheusUrl: string,
  serviceName: string,
  version: string,
  duration: string = "5m"
): Promise<CanaryMetrics> {
  const errorRateQuery = `
    sum(rate(istio_requests_total{
      destination_service_name="${serviceName}",
      destination_version="${version}",
      response_code=~"5.*"
    }[${duration}])) /
    sum(rate(istio_requests_total{
      destination_service_name="${serviceName}",
      destination_version="${version}"
    }[${duration}]))
  `;

  const latencyQuery = `
    histogram_quantile(0.99,
      sum(rate(istio_request_duration_milliseconds_bucket{
        destination_service_name="${serviceName}",
        destination_version="${version}"
      }[${duration}])) by (le)
    )
  `;

  const rpsQuery = `
    sum(rate(istio_requests_total{
      destination_service_name="${serviceName}",
      destination_version="${version}"
    }[${duration}]))
  `;

  const [errorResp, latencyResp, rpsResp] = await Promise.all([
    fetch(
      `${prometheusUrl}/api/v1/query?query=${encodeURIComponent(errorRateQuery)}`
    ),
    fetch(
      `${prometheusUrl}/api/v1/query?query=${encodeURIComponent(latencyQuery)}`
    ),
    fetch(
      `${prometheusUrl}/api/v1/query?query=${encodeURIComponent(rpsQuery)}`
    ),
  ]);

  const [errorData, latencyData, rpsData] = await Promise.all([
    errorResp.json(),
    latencyResp.json(),
    rpsResp.json(),
  ]);

  return {
    errorRate:
      parseFloat(errorData.data?.result?.[0]?.value?.[1] ?? "0") * 100,
    latencyP99: parseFloat(latencyData.data?.result?.[0]?.value?.[1] ?? "0"),
    requestsPerSecond: parseFloat(
      rpsData.data?.result?.[0]?.value?.[1] ?? "0"
    ),
  };
}

async function updateVirtualServiceWeight(
  customApi: CustomObjectsApi,
  config: CanaryConfig,
  canaryWeight: number
): Promise<void> {
  const stableWeight = 100 - canaryWeight;

  const patch = [
    {
      op: "replace",
      path: "/spec/http/1/route/0/weight",
      value: stableWeight,
    },
    {
      op: "replace",
      path: "/spec/http/1/route/1/weight",
      value: canaryWeight,
    },
  ];

  await customApi.patchNamespacedCustomObject(
    "networking.istio.io",
    "v1beta1",
    config.namespace,
    "virtualservices",
    `${config.serviceName}-vs`,
    patch,
    undefined,
    undefined,
    undefined,
    { headers: { "Content-Type": "application/json-patch+json" } }
  );

  console.log(
    `Updated weights: stable=${stableWeight}%, canary=${canaryWeight}%`
  );
}

async function promoteCanary(
  customApi: CustomObjectsApi,
  config: CanaryConfig
): Promise<void> {
  console.log(
    `Promoting canary ${config.canaryVersion} to stable in ${config.namespace}/${config.serviceName}`
  );

  // Route 100% to canary (now stable)
  await updateVirtualServiceWeight(customApi, config, 100);

  // Update stable deployment image
  const appsApi = new k8s.AppsV1Api();
  await appsApi.patchNamespacedDeployment(
    `${config.serviceName}-v1`,
    config.namespace,
    [
      {
        op: "replace",
        path: "/spec/template/spec/containers/0/image",
        value: `myregistry/${config.serviceName}:${config.canaryVersion}`,
      },
    ],
    undefined,
    undefined,
    undefined,
    undefined,
    undefined,
    { headers: { "Content-Type": "application/json-patch+json" } }
  );

  // Route back to stable (v1 now has canary image)
  await updateVirtualServiceWeight(customApi, config, 0);

  console.log("Canary promotion complete");
}

async function rollbackCanary(
  customApi: CustomObjectsApi,
  config: CanaryConfig,
  reason: string
): Promise<void> {
  console.error(`Rolling back canary: ${reason}`);
  await updateVirtualServiceWeight(customApi, config, 0);
  console.log("Canary rolled back to 0%");
}

export async function runCanaryDeployment(
  config: CanaryConfig,
  prometheusUrl: string
): Promise<void> {
  const kc = new k8s.KubeConfig();
  kc.loadFromDefault();
  const customApi = kc.makeApiClient(CustomObjectsApi);

  let currentWeight = config.initialWeight;

  console.log(
    `Starting canary deployment for ${config.serviceName} ${config.canaryVersion}`
  );
  console.log(
    `Initial weight: ${currentWeight}%, target: ${config.maxWeight}%`
  );

  await updateVirtualServiceWeight(customApi, config, currentWeight);

  while (currentWeight < config.maxWeight) {
    // Wait for analysis interval
    await new Promise((resolve) =>
      setTimeout(resolve, config.analysisInterval * 60 * 1000)
    );

    // Get metrics
    const metrics = await getCanaryMetrics(
      prometheusUrl,
      config.serviceName,
      config.canaryVersion
    );

    console.log(`Canary metrics at ${currentWeight}% weight:`, metrics);

    // Validate metrics
    if (metrics.errorRate > config.successThreshold) {
      await rollbackCanary(
        customApi,
        config,
        `Error rate ${metrics.errorRate.toFixed(2)}% exceeds threshold ${config.successThreshold}%`
      );
      return;
    }

    if (metrics.latencyP99 > 2000) {
      // 2 seconds
      await rollbackCanary(
        customApi,
        config,
        `P99 latency ${metrics.latencyP99}ms exceeds 2000ms threshold`
      );
      return;
    }

    // Increase weight
    currentWeight = Math.min(
      currentWeight + config.incrementStep,
      config.maxWeight
    );
    await updateVirtualServiceWeight(customApi, config, currentWeight);

    if (currentWeight >= config.maxWeight) {
      await promoteCanary(customApi, config);
      console.log("Canary deployment successful!");
    }
  }
}

// Usage example
const canaryConfig: CanaryConfig = {
  namespace: "production",
  serviceName: "product-service",
  stableVersion: "1.0.0",
  canaryVersion: "2.0.0",
  initialWeight: 5,
  incrementStep: 10,
  maxWeight: 100,
  analysisInterval: 5,
  successThreshold: 1.0,
};

runCanaryDeployment(canaryConfig, "http://prometheus:9090").catch(
  console.error
);
```

## 3. Fault Injection Testing

### 3.1 VirtualService สำหรับ Fault Injection

```yaml
# fault-injection-delay.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: product-service-fault-test
  namespace: staging
spec:
  hosts:
    - product-service
  http:
    # Inject 500ms delay for 30% of requests from test clients
    - match:
        - headers:
            x-test-scenario:
              exact: latency-test
      fault:
        delay:
          percentage:
            value: 30
          fixedDelay: 500ms
      route:
        - destination:
            host: product-service
            port:
              number: 80
    # Inject 503 errors for 10% of requests
    - match:
        - headers:
            x-test-scenario:
              exact: error-test
      fault:
        abort:
          percentage:
            value: 10
          httpStatus: 503
      route:
        - destination:
            host: product-service
            port:
              number: 80
    # Combination: delay + abort
    - match:
        - headers:
            x-test-scenario:
              exact: chaos-test
      fault:
        delay:
          percentage:
            value: 50
          fixedDelay: 1s
        abort:
          percentage:
            value: 20
          httpStatus: 500
      route:
        - destination:
            host: product-service
            port:
              number: 80
    # Default route
    - route:
        - destination:
            host: product-service
            port:
              number: 80
```

### 3.2 TypeScript Chaos Testing Framework

```typescript
// chaos-test.ts
import axios, { AxiosInstance } from "axios";
import * as k8s from "@kubernetes/client-node";
import { CustomObjectsApi } from "@kubernetes/client-node";

interface ChaosScenario {
  name: string;
  description: string;
  duration: number; // seconds
  delayPercentage?: number;
  delayMs?: number;
  abortPercentage?: number;
  abortStatus?: number;
  targetService: string;
  targetVersion?: string;
}

interface TestResult {
  scenario: string;
  totalRequests: number;
  successRequests: number;
  failedRequests: number;
  timeoutRequests: number;
  averageLatency: number;
  p99Latency: number;
  errorRate: number;
  passed: boolean;
}

class ChaosTestRunner {
  private customApi: CustomObjectsApi;
  private namespace: string;
  private httpClient: AxiosInstance;

  constructor(namespace: string, serviceUrl: string) {
    const kc = new k8s.KubeConfig();
    kc.loadFromDefault();
    this.customApi = kc.makeApiClient(CustomObjectsApi);
    this.namespace = namespace;
    this.httpClient = axios.create({
      baseURL: serviceUrl,
      timeout: 10000,
    });
  }

  async applyChaosVirtualService(scenario: ChaosScenario): Promise<void> {
    const faultSpec: Record<string, unknown> = {};

    if (scenario.delayPercentage && scenario.delayMs) {
      faultSpec.delay = {
        percentage: { value: scenario.delayPercentage },
        fixedDelay: `${scenario.delayMs}ms`,
      };
    }

    if (scenario.abortPercentage && scenario.abortStatus) {
      faultSpec.abort = {
        percentage: { value: scenario.abortPercentage },
        httpStatus: scenario.abortStatus,
      };
    }

    const vsSpec = {
      apiVersion: "networking.istio.io/v1beta1",
      kind: "VirtualService",
      metadata: {
        name: `${scenario.targetService}-chaos`,
        namespace: this.namespace,
        labels: { "chaos-test": "true" },
      },
      spec: {
        hosts: [scenario.targetService],
        http: [
          {
            match: [{ headers: { "x-chaos-test": { exact: "true" } } }],
            fault: faultSpec,
            route: [
              {
                destination: {
                  host: scenario.targetService,
                  port: { number: 80 },
                },
              },
            ],
          },
          {
            route: [
              {
                destination: {
                  host: scenario.targetService,
                  port: { number: 80 },
                },
              },
            ],
          },
        ],
      },
    };

    try {
      await this.customApi.createNamespacedCustomObject(
        "networking.istio.io",
        "v1beta1",
        this.namespace,
        "virtualservices",
        vsSpec
      );
    } catch (err: unknown) {
      const error = err as { response?: { status?: number } };
      if (error.response?.status === 409) {
        // Already exists, replace
        await this.customApi.replaceNamespacedCustomObject(
          "networking.istio.io",
          "v1beta1",
          this.namespace,
          "virtualservices",
          `${scenario.targetService}-chaos`,
          vsSpec
        );
      } else {
        throw err;
      }
    }

    console.log(`Applied chaos VirtualService for scenario: ${scenario.name}`);
  }

  async cleanupChaosVirtualService(serviceName: string): Promise<void> {
    try {
      await this.customApi.deleteNamespacedCustomObject(
        "networking.istio.io",
        "v1beta1",
        this.namespace,
        "virtualservices",
        `${serviceName}-chaos`
      );
      console.log(`Cleaned up chaos VirtualService for ${serviceName}`);
    } catch (err) {
      console.warn(`Failed to cleanup chaos VS: ${err}`);
    }
  }

  async runLoadTest(
    scenario: ChaosScenario,
    concurrency: number = 10,
    requestsPerSecond: number = 50
  ): Promise<TestResult> {
    const latencies: number[] = [];
    let totalRequests = 0;
    let successRequests = 0;
    let failedRequests = 0;
    let timeoutRequests = 0;

    const endTime = Date.now() + scenario.duration * 1000;

    const sendRequest = async (): Promise<void> => {
      const start = Date.now();
      try {
        await this.httpClient.get("/api/v1/products", {
          headers: {
            "x-chaos-test": "true",
            "x-request-id": `chaos-${Date.now()}-${Math.random()}`,
          },
        });
        const latency = Date.now() - start;
        latencies.push(latency);
        successRequests++;
      } catch (err: unknown) {
        const error = err as { code?: string; response?: { status?: number } };
        if (error.code === "ECONNABORTED" || error.code === "ETIMEDOUT") {
          timeoutRequests++;
        } else {
          failedRequests++;
        }
      } finally {
        totalRequests++;
      }
    };

    // Run load test
    while (Date.now() < endTime) {
      const batch = Array(concurrency)
        .fill(null)
        .map(() => sendRequest());
      await Promise.allSettled(batch);
      await new Promise((resolve) =>
        setTimeout(resolve, (1000 / requestsPerSecond) * concurrency)
      );
    }

    // Calculate statistics
    latencies.sort((a, b) => a - b);
    const averageLatency =
      latencies.length > 0
        ? latencies.reduce((sum, l) => sum + l, 0) / latencies.length
        : 0;
    const p99Index = Math.floor(latencies.length * 0.99);
    const p99Latency = latencies[p99Index] ?? 0;
    const errorRate =
      totalRequests > 0
        ? ((failedRequests + timeoutRequests) / totalRequests) * 100
        : 0;

    return {
      scenario: scenario.name,
      totalRequests,
      successRequests,
      failedRequests,
      timeoutRequests,
      averageLatency,
      p99Latency,
      errorRate,
      passed: errorRate < 5 && p99Latency < 3000,
    };
  }

  async runScenario(
    scenario: ChaosScenario,
    acceptableErrorRate: number = 5
  ): Promise<TestResult> {
    console.log(`\nRunning chaos scenario: ${scenario.name}`);
    console.log(`Description: ${scenario.description}`);

    try {
      // Apply chaos
      await this.applyChaosVirtualService(scenario);

      // Wait for VS to take effect
      await new Promise((resolve) => setTimeout(resolve, 5000));

      // Run test
      const result = await this.runLoadTest(scenario);

      console.log(`Results for ${scenario.name}:`);
      console.log(`  Total requests: ${result.totalRequests}`);
      console.log(`  Success: ${result.successRequests}`);
      console.log(`  Failed: ${result.failedRequests}`);
      console.log(`  Timeouts: ${result.timeoutRequests}`);
      console.log(`  Error rate: ${result.errorRate.toFixed(2)}%`);
      console.log(`  Avg latency: ${result.averageLatency.toFixed(0)}ms`);
      console.log(`  P99 latency: ${result.p99Latency}ms`);
      console.log(`  Passed: ${result.passed}`);

      return result;
    } finally {
      await this.cleanupChaosVirtualService(scenario.targetService);
    }
  }
}

// Production chaos test suite
async function runChaosTestSuite(): Promise<void> {
  const runner = new ChaosTestRunner(
    "staging",
    "http://product-service.staging.svc.cluster.local"
  );

  const scenarios: ChaosScenario[] = [
    {
      name: "network-delay-30pct",
      description: "30% of requests have 500ms delay",
      duration: 60,
      delayPercentage: 30,
      delayMs: 500,
      targetService: "product-service",
    },
    {
      name: "service-error-10pct",
      description: "10% of requests return 503",
      duration: 60,
      abortPercentage: 10,
      abortStatus: 503,
      targetService: "product-service",
    },
    {
      name: "high-latency-complete",
      description: "100% requests have 2s delay",
      duration: 30,
      delayPercentage: 100,
      delayMs: 2000,
      targetService: "product-service",
    },
  ];

  const results: TestResult[] = [];

  for (const scenario of scenarios) {
    const result = await runner.runScenario(scenario);
    results.push(result);
    await new Promise((resolve) => setTimeout(resolve, 10000)); // Cool down between tests
  }

  console.log("\n=== CHAOS TEST SUITE RESULTS ===");
  const passed = results.filter((r) => r.passed).length;
  const failed = results.filter((r) => !r.passed).length;
  console.log(`Passed: ${passed}/${results.length}`);
  console.log(`Failed: ${failed}/${results.length}`);

  if (failed > 0) {
    process.exit(1);
  }
}

runChaosTestSuite().catch(console.error);
```

## 4. Retry/Timeout Policies ใน VirtualService

### 4.1 Advanced Retry และ Timeout Configuration

```yaml
# retry-timeout-policies.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: order-service-vs
  namespace: production
spec:
  hosts:
    - order-service
  http:
    # Critical path: payment processing - strict timeout, no retry
    - match:
        - uri:
            prefix: /api/v1/payments
      timeout: 30s
      retries:
        attempts: 1
        perTryTimeout: 25s
        retryOn: "connect-failure,refused-stream"
        retryRemoteLocalities: false
      route:
        - destination:
            host: order-service
            port:
              number: 80
      headers:
        request:
          add:
            x-timeout-type: "payment-critical"

    # Read operations: aggressive retry
    - match:
        - uri:
            prefix: /api/v1/orders
          method:
            exact: GET
      timeout: 10s
      retries:
        attempts: 5
        perTryTimeout: 2s
        retryOn: "5xx,reset,connect-failure,retriable-4xx,cancelled,resource-exhausted,unavailable"
        retryRemoteLocalities: true
      route:
        - destination:
            host: order-service
            port:
              number: 80

    # Write operations: limited retry
    - match:
        - uri:
            prefix: /api/v1/orders
          method:
            regex: "POST|PUT|PATCH"
      timeout: 15s
      retries:
        attempts: 2
        perTryTimeout: 6s
        retryOn: "connect-failure,refused-stream,reset"
      route:
        - destination:
            host: order-service
            port:
              number: 80

    # Default with circuit breaker
    - route:
        - destination:
            host: order-service
            port:
              number: 80
      timeout: 5s
      retries:
        attempts: 3
        perTryTimeout: 1s
        retryOn: "5xx,reset,connect-failure"
---
# Destination Rule with Circuit Breaker
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: order-service-dr
  namespace: production
spec:
  host: order-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 50
        connectTimeout: 5s
      http:
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
        maxRequestsPerConnection: 10
        idleTimeout: 60s
    outlierDetection:
      consecutiveGatewayErrors: 3
      consecutive5xxErrors: 3
      interval: 5s
      baseEjectionTime: 15s
      maxEjectionPercent: 100
      minHealthPercent: 0
      splitExternalLocalOriginErrors: true
      consecutiveLocalOriginFailures: 3
```

## 5. Authorization Policies สำหรับ Zero-Trust

### 5.1 PeerAuthentication และ AuthorizationPolicy

```yaml
# peer-authentication-strict.yaml
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
# Namespace-level PeerAuthentication
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: production-mtls
  namespace: production
spec:
  mtls:
    mode: STRICT
---
# Per-workload PeerAuthentication with port exemption
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: product-service-pa
  namespace: production
spec:
  selector:
    matchLabels:
      app: product-service
  mtls:
    mode: STRICT
  portLevelMtls:
    9090: # Prometheus metrics port - allow plain text
      mode: PERMISSIVE
---
# Authorization Policy: Deny all by default
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: production
spec:
  {}  # Empty spec = deny all
---
# Authorization Policy: Allow specific services
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: product-service-authz
  namespace: production
spec:
  selector:
    matchLabels:
      app: product-service
  action: ALLOW
  rules:
    # Allow API gateway to access all endpoints
    - from:
        - source:
            principals:
              - "cluster.local/ns/production/sa/api-gateway"
            namespaces:
              - production
      to:
        - operation:
            methods: ["GET", "POST", "PUT", "DELETE"]
            paths: ["/api/v1/products*"]
    # Allow order-service read access
    - from:
        - source:
            principals:
              - "cluster.local/ns/production/sa/order-service"
      to:
        - operation:
            methods: ["GET"]
            paths: ["/api/v1/products/*", "/api/v1/products/*/inventory"]
    # Allow monitoring scraping
    - from:
        - source:
            namespaces: ["monitoring"]
      to:
        - operation:
            methods: ["GET"]
            paths: ["/metrics"]
            ports: ["9090"]
    # Allow health checks from any namespace
    - from:
        - source:
            namespaces: ["*"]
      to:
        - operation:
            methods: ["GET"]
            paths: ["/health/*"]
---
# JWT Authentication for external requests
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: jwt-auth
  namespace: production
spec:
  selector:
    matchLabels:
      app: product-service
  jwtRules:
    - issuer: "https://auth.example.com"
      jwksUri: "https://auth.example.com/.well-known/jwks.json"
      audiences:
        - "product-service"
      forwardOriginalToken: true
      outputPayloadToHeader: x-jwt-payload
---
# Authorization Policy using JWT claims
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: product-service-jwt-authz
  namespace: production
spec:
  selector:
    matchLabels:
      app: product-service
  action: ALLOW
  rules:
    # Admin can do everything
    - from:
        - source:
            requestPrincipals: ["https://auth.example.com/*"]
      to:
        - operation:
            methods: ["*"]
      when:
        - key: request.auth.claims[role]
          values: ["admin"]
    # Regular users - read only
    - from:
        - source:
            requestPrincipals: ["https://auth.example.com/*"]
      to:
        - operation:
            methods: ["GET"]
      when:
        - key: request.auth.claims[role]
          values: ["user", "viewer"]
```

### 5.2 ServiceAccount RBAC Setup

```yaml
# service-accounts.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: product-service
  namespace: production
  annotations:
    iam.gke.io/google-service-account: product-svc@project.iam.gserviceaccount.com
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: order-service
  namespace: production
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: api-gateway
  namespace: production
---
# RBAC for service accounts
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["db-secret", "redis-secret"]
    verbs: ["get", "watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: product-service-secret-reader
  namespace: production
subjects:
  - kind: ServiceAccount
    name: product-service
    namespace: production
roleRef:
  kind: Role
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

## 6. Envoy Filter Configuration

### 6.1 Custom Envoy Filters

```yaml
# envoy-filter-rate-limit.yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: rate-limit-filter
  namespace: production
spec:
  workloadSelector:
    labels:
      app: product-service
  configPatches:
    - applyTo: HTTP_FILTER
      match:
        context: SIDECAR_INBOUND
        listener:
          filterChain:
            filter:
              name: envoy.filters.network.http_connection_manager
              subFilter:
                name: envoy.filters.http.router
      patch:
        operation: INSERT_BEFORE
        value:
          name: envoy.filters.http.local_ratelimit
          typed_config:
            "@type": type.googleapis.com/udpa.type.v1.TypedStruct
            type_url: type.googleapis.com/envoy.extensions.filters.http.local_ratelimit.v3.LocalRateLimit
            value:
              stat_prefix: http_local_rate_limiter
              token_bucket:
                max_tokens: 1000
                tokens_per_fill: 100
                fill_interval: 1s
              filter_enabled:
                runtime_key: local_rate_limit_enabled
                default_value:
                  numerator: 100
                  denominator: HUNDRED
              filter_enforced:
                runtime_key: local_rate_limit_enforced
                default_value:
                  numerator: 100
                  denominator: HUNDRED
              response_headers_to_add:
                - append: false
                  header:
                    key: x-local-rate-limit
                    value: "true"
---
# EnvoyFilter for custom request headers
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: custom-headers-filter
  namespace: production
spec:
  workloadSelector:
    labels:
      app: api-gateway
  configPatches:
    - applyTo: HTTP_FILTER
      match:
        context: SIDECAR_OUTBOUND
        listener:
          filterChain:
            filter:
              name: envoy.filters.network.http_connection_manager
              subFilter:
                name: envoy.filters.http.router
      patch:
        operation: INSERT_BEFORE
        value:
          name: envoy.filters.http.lua
          typed_config:
            "@type": type.googleapis.com/envoy.extensions.filters.http.lua.v3.LuaPerRoute
            source_code:
              inline_string: |
                function envoy_on_request(request_handle)
                  -- Add request ID if not present
                  local request_id = request_handle:headers():get("x-request-id")
                  if not request_id then
                    request_handle:headers():add("x-request-id", 
                      tostring(math.random(1000000)))
                  end
                  
                  -- Add timestamp
                  request_handle:headers():add("x-request-time", 
                    tostring(os.time()))
                end
                
                function envoy_on_response(response_handle)
                  -- Add response headers
                  response_handle:headers():add("x-served-by", "istio-mesh")
                end
---
# EnvoyFilter for WASM plugin
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: wasm-auth-filter
  namespace: production
spec:
  workloadSelector:
    labels:
      app: product-service
  configPatches:
    - applyTo: HTTP_FILTER
      match:
        context: SIDECAR_INBOUND
        listener:
          filterChain:
            filter:
              name: envoy.filters.network.http_connection_manager
              subFilter:
                name: envoy.filters.http.router
      patch:
        operation: INSERT_BEFORE
        value:
          name: envoy.filters.http.wasm
          typed_config:
            "@type": type.googleapis.com/udpa.type.v1.TypedStruct
            type_url: type.googleapis.com/envoy.extensions.filters.http.wasm.v3.Wasm
            value:
              config:
                name: custom-auth
                root_id: custom-auth
                vm_config:
                  runtime: envoy.wasm.runtime.v8
                  code:
                    local:
                      filename: /etc/istio/extensions/custom-auth.wasm
                configuration:
                  "@type": type.googleapis.com/google.protobuf.StringValue
                  value: '{"allowed_prefixes":["/api/v1/products","/health"]}'
```

## 7. Distributed Tracing ด้วย Jaeger/Zipkin

### 7.1 Jaeger Installation และ Configuration

```yaml
# jaeger-all-in-one.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jaeger
  namespace: monitoring
  labels:
    app: jaeger
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jaeger
  template:
    metadata:
      labels:
        app: jaeger
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "14269"
    spec:
      containers:
        - name: jaeger
          image: jaegertracing/all-in-one:1.50
          ports:
            - containerPort: 5775
              name: zk-compact-trft
              protocol: UDP
            - containerPort: 5778
              name: config-rest
            - containerPort: 6831
              name: jg-compact-trft
              protocol: UDP
            - containerPort: 6832
              name: jg-binary-trft
              protocol: UDP
            - containerPort: 9411
              name: zipkin
            - containerPort: 14268
              name: jaeger-collector
            - containerPort: 14269
              name: admin-http
            - containerPort: 16686
              name: query
            - containerPort: 4317
              name: grpc-otlp
          env:
            - name: SPAN_STORAGE_TYPE
              value: elasticsearch
            - name: ES_SERVER_URLS
              value: http://elasticsearch:9200
            - name: ES_NUM_SHARDS
              value: "2"
            - name: ES_NUM_REPLICAS
              value: "1"
            - name: COLLECTOR_ZIPKIN_HOST_PORT
              value: ":9411"
            - name: COLLECTOR_OTLP_ENABLED
              value: "true"
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: 1000m
              memory: 1Gi
          readinessProbe:
            httpGet:
              path: /
              port: 14269
            initialDelaySeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: jaeger-collector
  namespace: monitoring
spec:
  selector:
    app: jaeger
  ports:
    - name: zipkin
      port: 9411
      targetPort: 9411
    - name: grpc-otlp
      port: 4317
      targetPort: 4317
    - name: http-otlp
      port: 4318
      targetPort: 4318
    - name: jaeger-collector-http
      port: 14268
      targetPort: 14268
---
apiVersion: v1
kind: Service
metadata:
  name: jaeger-query
  namespace: monitoring
spec:
  selector:
    app: jaeger
  ports:
    - name: query-http
      port: 16686
      targetPort: 16686
  type: ClusterIP
```

### 7.2 TypeScript OpenTelemetry Tracing Setup

```typescript
// tracing.ts
import { NodeSDK } from "@opentelemetry/sdk-node";
import { OTLPTraceExporter } from "@opentelemetry/exporter-trace-otlp-grpc";
import { OTLPMetricExporter } from "@opentelemetry/exporter-metrics-otlp-grpc";
import {
  PeriodicExportingMetricReader,
  ConsoleMetricExporter,
} from "@opentelemetry/sdk-metrics";
import { Resource } from "@opentelemetry/resources";
import {
  SemanticResourceAttributes,
} from "@opentelemetry/semantic-conventions";
import { BatchSpanProcessor } from "@opentelemetry/sdk-trace-base";
import { B3Propagator } from "@opentelemetry/propagator-b3";
import { W3CTraceContextPropagator } from "@opentelemetry/core";
import { CompositePropagator } from "@opentelemetry/core";
import {
  trace,
  context,
  SpanStatusCode,
  Span,
  SpanKind,
} from "@opentelemetry/api";
import { HttpInstrumentation } from "@opentelemetry/instrumentation-http";
import { ExpressInstrumentation } from "@opentelemetry/instrumentation-express";
import { PgInstrumentation } from "@opentelemetry/instrumentation-pg";
import { RedisInstrumentation } from "@opentelemetry/instrumentation-redis-4";

const SERVICE_NAME =
  process.env.SERVICE_NAME || "product-service";
const SERVICE_VERSION = process.env.SERVICE_VERSION || "1.0.0";
const OTLP_ENDPOINT =
  process.env.OTLP_ENDPOINT ||
  "http://jaeger-collector.monitoring:4317";

export function initTracing(): NodeSDK {
  const resource = new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]: SERVICE_NAME,
    [SemanticResourceAttributes.SERVICE_VERSION]: SERVICE_VERSION,
    [SemanticResourceAttributes.DEPLOYMENT_ENVIRONMENT]:
      process.env.NODE_ENV || "production",
    "service.namespace": process.env.NAMESPACE || "production",
    "k8s.pod.name": process.env.POD_NAME || "unknown",
    "k8s.node.name": process.env.NODE_NAME || "unknown",
  });

  const traceExporter = new OTLPTraceExporter({
    url: OTLP_ENDPOINT,
    headers: {},
  });

  const metricExporter = new OTLPMetricExporter({
    url: OTLP_ENDPOINT,
  });

  const sdk = new NodeSDK({
    resource,
    spanProcessor: new BatchSpanProcessor(traceExporter, {
      maxQueueSize: 2048,
      maxExportBatchSize: 512,
      scheduledDelayMillis: 500,
      exportTimeoutMillis: 30000,
    }),
    metricReader: new PeriodicExportingMetricReader({
      exporter: metricExporter,
      exportIntervalMillis: 10000,
    }),
    textMapPropagator: new CompositePropagator({
      propagators: [
        new W3CTraceContextPropagator(),
        new B3Propagator(),
      ],
    }),
    instrumentations: [
      new HttpInstrumentation({
        ignoreIncomingRequestHook: (req) => {
          // Skip health check and metrics endpoints
          return (
            req.url?.startsWith("/health") || req.url?.startsWith("/metrics")
          ) ?? false;
        },
        requestHook: (span, req) => {
          // Add custom attributes
          span.setAttribute("http.request.id", req.headers["x-request-id"] as string || "");
        },
        responseHook: (span, res) => {
          span.setAttribute("http.response.content_type", res.getHeader("content-type") as string || "");
        },
      }),
      new ExpressInstrumentation({
        requestHook: (span, info) => {
          span.setAttribute("http.route", info.route);
        },
      }),
      new PgInstrumentation({
        enhancedDatabaseReporting: true,
      }),
      new RedisInstrumentation(),
    ],
  });

  sdk.start();

  process.on("SIGTERM", () => {
    sdk.shutdown().then(() => console.log("Tracing terminated"));
  });

  return sdk;
}

// Custom tracer utilities
export class TracingHelper {
  private static tracer = trace.getTracer(SERVICE_NAME, SERVICE_VERSION);

  static async withSpan<T>(
    name: string,
    fn: (span: Span) => Promise<T>,
    options?: {
      kind?: SpanKind;
      attributes?: Record<string, string | number | boolean>;
    }
  ): Promise<T> {
    const span = this.tracer.startSpan(name, {
      kind: options?.kind ?? SpanKind.INTERNAL,
      attributes: options?.attributes,
    });

    const ctx = trace.setSpan(context.active(), span);

    try {
      const result = await context.with(ctx, () => fn(span));
      span.setStatus({ code: SpanStatusCode.OK });
      return result;
    } catch (error) {
      span.setStatus({
        code: SpanStatusCode.ERROR,
        message: error instanceof Error ? error.message : String(error),
      });
      span.recordException(error instanceof Error ? error : new Error(String(error)));
      throw error;
    } finally {
      span.end();
    }
  }

  static addEvent(
    name: string,
    attributes?: Record<string, string | number | boolean>
  ): void {
    const span = trace.getActiveSpan();
    if (span) {
      span.addEvent(name, attributes);
    }
  }

  static setAttributes(
    attributes: Record<string, string | number | boolean>
  ): void {
    const span = trace.getActiveSpan();
    if (span) {
      Object.entries(attributes).forEach(([key, value]) => {
        span.setAttribute(key, value);
      });
    }
  }
}

// Example usage in service
async function getProduct(productId: string): Promise<unknown> {
  return TracingHelper.withSpan(
    "product.get",
    async (span) => {
      span.setAttribute("product.id", productId);

      TracingHelper.addEvent("fetching-from-cache");
      const cached = await getFromCache(productId);

      if (cached) {
        TracingHelper.addEvent("cache-hit");
        span.setAttribute("cache.hit", true);
        return cached;
      }

      TracingHelper.addEvent("cache-miss-fetching-from-db");
      span.setAttribute("cache.hit", false);

      const product = await TracingHelper.withSpan(
        "product.db.query",
        async (dbSpan) => {
          dbSpan.setAttribute("db.operation", "SELECT");
          dbSpan.setAttribute("db.table", "products");
          return fetchFromDatabase(productId);
        },
        { kind: SpanKind.CLIENT }
      );

      await cacheProduct(productId, product);
      return product;
    },
    {
      kind: SpanKind.SERVER,
      attributes: {
        "service.operation": "getProduct",
      },
    }
  );
}

async function getFromCache(_id: string): Promise<unknown> { return null; }
async function fetchFromDatabase(_id: string): Promise<unknown> { return {}; }
async function cacheProduct(_id: string, _data: unknown): Promise<void> {}
```

## 8. DestinationRule Load Balancing

### 8.1 Advanced Load Balancing Configurations

```yaml
# destination-rule-lb.yaml
# Least connections load balancing
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: product-service-lb-least-conn
  namespace: production
spec:
  host: product-service
  trafficPolicy:
    loadBalancer:
      simple: LEAST_CONN
    connectionPool:
      http:
        http2MaxRequests: 10000
        maxRequestsPerConnection: 100
---
# Consistent hash load balancing (session stickiness)
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: session-service-lb-hash
  namespace: production
spec:
  host: session-service
  trafficPolicy:
    loadBalancer:
      consistentHash:
        httpCookie:
          name: session-id
          path: /
          ttl: 3600s
        minimumRingSize: 1024
---
# Header-based hash (route by user ID)
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: user-service-lb-header
  namespace: production
spec:
  host: user-service
  trafficPolicy:
    loadBalancer:
      consistentHash:
        httpHeaderName: x-user-id
        minimumRingSize: 512
---
# Locality-based load balancing
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: product-service-locality-lb
  namespace: production
spec:
  host: product-service
  trafficPolicy:
    loadBalancer:
      simple: ROUND_ROBIN
      localityLbSetting:
        enabled: true
        failover:
          - from: us-east1
            to: us-central1
          - from: us-central1
            to: us-west1
        failoverPriority:
          - "topology.kubernetes.io/region"
          - "topology.kubernetes.io/zone"
          - "kubernetes.io/hostname"
    outlierDetection:
      consecutiveGatewayErrors: 5
      interval: 10s
      baseEjectionTime: 30s
```

## 9. mTLS Peer Authentication

### 9.1 mTLS Configuration และ Certificates

```yaml
# mtls-strict.yaml
# Strict mTLS for entire namespace
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: strict-mtls
  namespace: production
spec:
  mtls:
    mode: STRICT
---
# Custom certificate rotation
apiVersion: v1
kind: ConfigMap
metadata:
  name: istio-ca-config
  namespace: istio-system
data:
  # Certificate signing config
  cert-chain.pem: |
    # Issued by private CA
  workload-cert-ttl: "24h"
  self-signed-ca-cert-ttl: "87600h"
---
# External CA configuration
apiVersion: v1
kind: Secret
metadata:
  name: cacerts
  namespace: istio-system
type: Opaque
data:
  # base64 encoded certificates from your CA
  ca-cert.pem: <base64-encoded-cert>
  ca-key.pem: <base64-encoded-key>
  root-cert.pem: <base64-encoded-root-cert>
  cert-chain.pem: <base64-encoded-cert-chain>
```

### 9.2 mTLS Verification Script

```typescript
// mtls-verifier.ts
import * as tls from "tls";
import * as https from "https";
import * as fs from "fs";
import { execSync } from "child_process";

interface mTLSConfig {
  cert: string;
  key: string;
  ca: string;
  host: string;
  port: number;
}

interface mTLSVerificationResult {
  connected: boolean;
  peerCertificate?: tls.PeerCertificate;
  mutuallyAuthenticated: boolean;
  error?: string;
  cipherSuite?: string;
  protocol?: string;
}

export async function verifyMTLS(
  config: mTLSConfig
): Promise<mTLSVerificationResult> {
  return new Promise((resolve) => {
    const options: tls.ConnectionOptions = {
      host: config.host,
      port: config.port,
      cert: fs.readFileSync(config.cert),
      key: fs.readFileSync(config.key),
      ca: fs.readFileSync(config.ca),
      rejectUnauthorized: true,
      checkServerIdentity: (hostname, cert) => {
        // Custom verification: check SPIFFE URI
        const san = cert.subjectaltname || "";
        const spiffeUri = san
          .split(", ")
          .find((s) => s.startsWith("URI:spiffe://"));

        if (!spiffeUri) {
          return new Error("No SPIFFE URI in certificate");
        }

        console.log(`SPIFFE identity: ${spiffeUri}`);
        return undefined;
      },
    };

    const socket = tls.connect(options, () => {
      const peerCert = socket.getPeerCertificate(true);
      const authorized = socket.authorized;
      const cipher = socket.getCipher();

      console.log(`Connected to ${config.host}:${config.port}`);
      console.log(`Authorized: ${authorized}`);
      console.log(`Cipher: ${cipher.name}`);
      console.log(`Protocol: ${socket.getProtocol()}`);

      if (peerCert.subjectaltname) {
        console.log(`SANs: ${peerCert.subjectaltname}`);
      }

      resolve({
        connected: true,
        peerCertificate: peerCert,
        mutuallyAuthenticated: authorized,
        cipherSuite: cipher.name,
        protocol: socket.getProtocol() || undefined,
      });

      socket.destroy();
    });

    socket.on("error", (err) => {
      resolve({
        connected: false,
        mutuallyAuthenticated: false,
        error: err.message,
      });
    });

    socket.setTimeout(5000, () => {
      socket.destroy();
      resolve({
        connected: false,
        mutuallyAuthenticated: false,
        error: "Connection timed out",
      });
    });
  });
}

export async function checkIstioCertificates(
  namespace: string,
  podName: string
): Promise<void> {
  try {
    // Get certificate info from Envoy
    const certOutput = execSync(
      `kubectl exec -n ${namespace} ${podName} -c istio-proxy -- ` +
        `openssl x509 -in /var/run/secrets/workload-spiffe-credentials/tls-cert.pem -noout -text`
    ).toString();

    console.log("Certificate Info:");
    console.log(certOutput);

    // Check certificate expiry
    const expiryOutput = execSync(
      `kubectl exec -n ${namespace} ${podName} -c istio-proxy -- ` +
        `openssl x509 -in /var/run/secrets/workload-spiffe-credentials/tls-cert.pem -noout -dates`
    ).toString();

    console.log("Certificate Dates:");
    console.log(expiryOutput);
  } catch (error) {
    console.error("Failed to check certificates:", error);
  }
}
```

## 10. Traffic Mirroring (Shadowing)

### 10.1 VirtualService สำหรับ Traffic Mirroring

```yaml
# traffic-mirroring.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: product-service-mirror
  namespace: production
spec:
  hosts:
    - product-service
  http:
    # Mirror 20% of production traffic to v2 for testing
    - match:
        - uri:
            prefix: /api/v1/products
          method:
            exact: GET
      route:
        - destination:
            host: product-service
            subset: v1
            port:
              number: 80
          weight: 100
      mirror:
        host: product-service
        subset: v2
        port:
          number: 80
      mirrorPercentage:
        value: 20
    # Shadow all writes to shadow cluster for validation
    - match:
        - uri:
            prefix: /api/v1/products
          method:
            regex: "POST|PUT"
      route:
        - destination:
            host: product-service
            subset: v1
            port:
              number: 80
          weight: 100
      mirror:
        host: product-service-shadow.staging.svc.cluster.local
        port:
          number: 80
      mirrorPercentage:
        value: 100
---
# Shadow service deployment (receives mirrored traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-service-shadow
  namespace: staging
  labels:
    app: product-service-shadow
spec:
  replicas: 1
  selector:
    matchLabels:
      app: product-service-shadow
  template:
    metadata:
      labels:
        app: product-service-shadow
    spec:
      containers:
        - name: product-service
          image: myregistry/product-service:2.0.0-rc1
          ports:
            - containerPort: 3000
          env:
            - name: SHADOW_MODE
              value: "true"
            - name: LOG_SHADOW_REQUESTS
              value: "true"
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 256Mi
```

### 10.2 Shadow Traffic Analyzer

```typescript
// shadow-analyzer.ts
import express, { Request, Response, NextFunction } from "express";
import { promisify } from "util";
import Redis from "ioredis";

interface ShadowComparison {
  requestId: string;
  timestamp: number;
  path: string;
  method: string;
  stableResponse: {
    statusCode: number;
    body: unknown;
    latencyMs: number;
  };
  shadowResponse: {
    statusCode: number;
    body: unknown;
    latencyMs: number;
  };
  diverged: boolean;
  divergenceReason?: string;
}

class ShadowTrafficAnalyzer {
  private redis: Redis;
  private comparisons: Map<string, Partial<ShadowComparison>>;

  constructor(redisUrl: string) {
    this.redis = new Redis(redisUrl);
    this.comparisons = new Map();
  }

  expressMiddleware() {
    return async (req: Request, res: Response, next: NextFunction) => {
      const isShadow = req.headers["x-envoy-is-shadow-request"] === "1";
      const requestId = (req.headers["x-request-id"] as string) || "";

      if (isShadow) {
        const start = Date.now();
        const originalSend = res.send.bind(res);

        let responseBody: unknown;
        res.send = (body: unknown) => {
          responseBody = body;
          return originalSend(body);
        };

        res.on("finish", async () => {
          const comparison = this.comparisons.get(requestId);
          if (comparison) {
            comparison.shadowResponse = {
              statusCode: res.statusCode,
              body: responseBody,
              latencyMs: Date.now() - start,
            };

            // Analyze divergence
            this.analyzeDivergence(comparison as ShadowComparison);

            // Store for analysis
            await this.redis.setex(
              `shadow:${requestId}`,
              3600,
              JSON.stringify(comparison)
            );

            this.comparisons.delete(requestId);
          }
        });
      }

      next();
    };
  }

  private analyzeDivergence(comparison: ShadowComparison): void {
    const stable = comparison.stableResponse;
    const shadow = comparison.shadowResponse;

    if (!stable || !shadow) return;

    if (stable.statusCode !== shadow.statusCode) {
      comparison.diverged = true;
      comparison.divergenceReason = `Status code divergence: ${stable.statusCode} vs ${shadow.statusCode}`;
      return;
    }

    // Deep compare response bodies
    try {
      const stableBody = JSON.stringify(stable.body);
      const shadowBody = JSON.stringify(shadow.body);

      if (stableBody !== shadowBody) {
        comparison.diverged = true;
        comparison.divergenceReason = "Response body divergence";
      }
    } catch {
      // Non-JSON responses
    }

    // Check latency regression
    if (shadow.latencyMs > stable.latencyMs * 2) {
      comparison.divergenceReason = comparison.divergenceReason
        ? `${comparison.divergenceReason}; Latency regression: ${shadow.latencyMs}ms vs ${stable.latencyMs}ms`
        : `Latency regression: ${shadow.latencyMs}ms vs ${stable.latencyMs}ms`;
    }
  }

  async getDivergenceReport(
    hours: number = 24
  ): Promise<{ total: number; diverged: number; divergenceRate: number }> {
    const keys = await this.redis.keys("shadow:*");
    let total = 0;
    let diverged = 0;

    const cutoff = Date.now() - hours * 3600 * 1000;

    for (const key of keys) {
      const data = await this.redis.get(key);
      if (!data) continue;

      const comparison: ShadowComparison = JSON.parse(data);
      if (comparison.timestamp < cutoff) continue;

      total++;
      if (comparison.diverged) diverged++;
    }

    return {
      total,
      diverged,
      divergenceRate: total > 0 ? (diverged / total) * 100 : 0,
    };
  }
}

export { ShadowTrafficAnalyzer };
```

## 11. Ingress Gateway TLS Termination

### 11.1 TLS Certificate Setup

```bash
# สร้าง self-signed certificate สำหรับ testing
openssl req -x509 -newkey rsa:4096 \
  -keyout tls.key \
  -out tls.crt \
  -days 365 \
  -nodes \
  -subj "/CN=api.example.com" \
  -addext "subjectAltName=DNS:api.example.com,DNS:*.example.com"

# สร้าง TLS secret ใน Kubernetes
kubectl create secret tls api-tls-secret \
  -n istio-system \
  --cert=tls.crt \
  --key=tls.key

# สำหรับ production ใช้ cert-manager
```

### 11.2 cert-manager Certificate

```yaml
# cert-manager-certificate.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: ops@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
      - http01:
          ingress:
            class: istio
      - dns01:
          cloudflare:
            email: ops@example.com
            apiTokenSecretRef:
              name: cloudflare-api-token
              key: api-token
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: api-example-com
  namespace: istio-system
spec:
  secretName: api-tls-secret
  duration: 2160h # 90 days
  renewBefore: 360h # 15 days
  subject:
    organizations:
      - Example Corp
  isCA: false
  privateKey:
    algorithm: RSA
    encoding: PKCS1
    size: 2048
  usages:
    - server auth
    - client auth
  dnsNames:
    - api.example.com
    - "*.api.example.com"
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
```

### 11.3 Gateway และ VirtualService สำหรับ TLS

```yaml
# ingress-gateway-tls.yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: main-gateway
  namespace: production
spec:
  selector:
    istio: ingressgateway
  servers:
    # HTTP redirect to HTTPS
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - "api.example.com"
        - "*.example.com"
      tls:
        httpsRedirect: true
    # HTTPS with TLS termination
    - port:
        number: 443
        name: https
        protocol: HTTPS
      hosts:
        - "api.example.com"
        - "*.example.com"
      tls:
        mode: SIMPLE
        credentialName: api-tls-secret
        minProtocolVersion: TLSV1_3
        cipherSuites:
          - ECDHE-ECDSA-AES256-GCM-SHA384
          - ECDHE-RSA-AES256-GCM-SHA384
    # mTLS for internal services
    - port:
        number: 15443
        name: tls-internal
        protocol: TLS
      hosts:
        - "*.internal.example.com"
      tls:
        mode: MUTUAL
        credentialName: internal-tls-secret
        minProtocolVersion: TLSV1_3
---
# VirtualService เชื่อมกับ Gateway
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: api-gateway-vs
  namespace: production
spec:
  hosts:
    - "api.example.com"
  gateways:
    - production/main-gateway
  http:
    - match:
        - uri:
            prefix: /api/v1/products
      route:
        - destination:
            host: product-service
            port:
              number: 80
      timeout: 30s
      corsPolicy:
        allowOrigins:
          - exact: "https://app.example.com"
          - regex: "https://.*\\.example\\.com"
        allowMethods:
          - GET
          - POST
          - PUT
          - DELETE
          - OPTIONS
        allowHeaders:
          - Authorization
          - Content-Type
          - X-Request-ID
        exposeHeaders:
          - X-Request-ID
          - X-Rate-Limit-Remaining
        maxAge: "24h"
        allowCredentials: true
    - match:
        - uri:
            prefix: /api/v1/orders
      route:
        - destination:
            host: order-service
            port:
              number: 80
      timeout: 30s
    # Health check passthrough
    - match:
        - uri:
            exact: /health
      directResponse:
        status: 200
        body:
          string: '{"status":"healthy","gateway":"istio"}'
```

### 11.4 Monitoring Istio ด้วย Prometheus

```yaml
# prometheus-istio-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: istio-alerts
  namespace: monitoring
  labels:
    prometheus: kube-prometheus
    role: alert-rules
spec:
  groups:
    - name: istio.rules
      interval: 30s
      rules:
        - alert: IstioHighRequestLatency
          expr: |
            histogram_quantile(0.99,
              sum(rate(istio_request_duration_milliseconds_bucket{
                reporter="destination"
              }[5m])) by (le, destination_service_name)
            ) > 2000
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "High P99 request latency for {{ $labels.destination_service_name }}"
            description: "P99 latency is {{ $value }}ms for {{ $labels.destination_service_name }}"

        - alert: IstioHighErrorRate
          expr: |
            sum(rate(istio_requests_total{
              reporter="destination",
              response_code=~"5.*"
            }[5m])) by (destination_service_name) /
            sum(rate(istio_requests_total{
              reporter="destination"
            }[5m])) by (destination_service_name) > 0.05
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "High error rate for {{ $labels.destination_service_name }}"
            description: "Error rate is {{ $value | humanizePercentage }} for {{ $labels.destination_service_name }}"

        - alert: IstioMTLSNotEnabled
          expr: |
            sum by (namespace, source_app, destination_app) (
              rate(istio_requests_total{
                connection_security_policy="none"
              }[5m])
            ) > 0
          for: 1m
          labels:
            severity: critical
          annotations:
            summary: "mTLS not enabled between {{ $labels.source_app }} and {{ $labels.destination_app }}"

        - alert: IstioCertificateExpiringSoon
          expr: |
            (istio_agent_pilot_conflict_outbound_listener_tcp_over_current_tcp > 0)
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Certificate conflict detected"
```

## 12. Istio Observability Dashboard

```yaml
# grafana-dashboard-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: istio-service-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "1"
data:
  istio-service.json: |
    {
      "title": "Istio Service Dashboard",
      "uid": "istio-service",
      "panels": [
        {
          "title": "Request Volume",
          "type": "graph",
          "targets": [
            {
              "expr": "sum(rate(istio_requests_total{reporter=\"destination\"}[5m])) by (destination_service_name)",
              "legendFormat": "{{destination_service_name}}"
            }
          ]
        },
        {
          "title": "Error Rate",
          "type": "graph",
          "targets": [
            {
              "expr": "sum(rate(istio_requests_total{reporter=\"destination\",response_code=~\"5.*\"}[5m])) by (destination_service_name) / sum(rate(istio_requests_total{reporter=\"destination\"}[5m])) by (destination_service_name)",
              "legendFormat": "{{destination_service_name}} error rate"
            }
          ]
        },
        {
          "title": "P99 Latency",
          "type": "graph",
          "targets": [
            {
              "expr": "histogram_quantile(0.99, sum(rate(istio_request_duration_milliseconds_bucket{reporter=\"destination\"}[5m])) by (le, destination_service_name))",
              "legendFormat": "{{destination_service_name}} p99"
            }
          ]
        }
      ]
    }
```

## สรุป

| Feature | Configuration | Use Case | Complexity |
|---------|--------------|----------|------------|
| **Traffic Splitting** | VirtualService weights | Canary/Blue-Green deployments | Medium |
| **Fault Injection** | VirtualService fault spec | Chaos engineering, resilience testing | Low |
| **Retry Policy** | VirtualService retries | Handle transient failures | Low |
| **Timeout Policy** | VirtualService timeout | Prevent cascade failures | Low |
| **Authorization Policy** | AuthorizationPolicy CRD | Zero-trust security | High |
| **mTLS** | PeerAuthentication STRICT | Service-to-service encryption | Medium |
| **Envoy Filter** | EnvoyFilter CRD | Custom proxy behavior | Very High |
| **Distributed Tracing** | Jaeger + OpenTelemetry | Debugging, performance analysis | Medium |
| **Load Balancing** | DestinationRule trafficPolicy | Traffic distribution | Medium |
| **Traffic Mirroring** | VirtualService mirror | Shadow testing new versions | Medium |
| **TLS Termination** | Gateway + cert-manager | HTTPS ingress | Medium |
| **Circuit Breaking** | DestinationRule outlierDetection | Failure isolation | Medium |

### Key Takeaways

1. **Zero-Trust Security**: ใช้ `PeerAuthentication STRICT` + `AuthorizationPolicy` เพื่อ deny-by-default และเปิดเฉพาะ service ที่จำเป็น

2. **Progressive Delivery**: เริ่ม canary ที่ 5% → เพิ่มทีละ 10% โดยดู error rate และ latency ก่อน promote

3. **Observability**: ทำ Tracing ทุก request ที่สำคัญ ใช้ Jaeger ดู distributed traces เมื่อมี issue

4. **Fault Tolerance**: กำหนด retry/timeout ให้เหมาะกับ operation type (read vs write, idempotent vs non-idempotent)

5. **Testing**: ใช้ fault injection ใน staging ก่อน production เพื่อทดสอบว่า service ทนต่อ failure ได้

6. **Certificate Management**: ใช้ cert-manager สำหรับ automatic certificate rotation ลด operational overhead

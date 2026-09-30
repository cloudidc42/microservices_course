# Part 32: Service Mesh Advanced - Traffic Management & Progressive Delivery

## บทนำ

Istio Service Mesh ช่วยให้เราจัดการ traffic ระหว่าง microservices ได้อย่างละเอียด โดยไม่ต้องแก้ไข application code ในบทนี้เราจะเรียนรู้เทคนิคขั้นสูงสำหรับ Progressive Delivery, Traffic Splitting, Fault Injection Testing, และ mTLS mutual authentication

## 1. Canary Deployment with Traffic Splitting

### 1.1 VirtualService for Canary Releases

```yaml
# k8s/istio/virtualservice-canary.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: product-service
  namespace: production
spec:
  hosts:
    - product-service
    - product.example.com
  gateways:
    - ingressgateway
    - mesh  # Internal service-to-service
  http:
    # Route 5% of traffic to v2 (canary)
    - name: canary-split
      match:
        # Internal calls use header-based routing
        - headers:
            x-canary-user:
              exact: "true"
      route:
        - destination:
            host: product-service
            subset: v2
          weight: 100
    
    - name: main-traffic
      route:
        - destination:
            host: product-service
            subset: v1
          weight: 95
        - destination:
            host: product-service
            subset: v2
          weight: 5
      timeout: 30s
      retries:
        attempts: 3
        perTryTimeout: 10s
        retryOn: gateway-error,connect-failure,retriable-4xx
```

### 1.2 DestinationRule for Subsets

```yaml
# k8s/istio/destinationrule-canary.yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: product-service
  namespace: production
spec:
  host: product-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 1000
      http:
        http1MaxPendingRequests: 500
        http2MaxRequests: 1000
        maxRetries: 3
    loadBalancer:
      simple: LEAST_CONN
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 10s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
      minHealthPercent: 30
  subsets:
    - name: v1
      labels:
        version: v1
      trafficPolicy:
        connectionPool:
          http:
            http2MaxRequests: 800
    
    - name: v2
      labels:
        version: v2
      trafficPolicy:
        connectionPool:
          http:
            http2MaxRequests: 200  # Limited for canary
```

### 1.3 Automated Canary Analysis with Argo Rollouts

```yaml
# k8s/argo-rollouts/product-service-rollout.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: product-service
  namespace: production
spec:
  replicas: 10
  selector:
    matchLabels:
      app: product-service
  template:
    metadata:
      labels:
        app: product-service
    spec:
      containers:
        - name: product-service
          image: product-service:latest
          ports:
            - containerPort: 3000
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi
  
  strategy:
    canary:
      maxSurge: "20%"
      maxUnavailable: 0
      
      analysis:
        templates:
          - templateName: success-rate
          - templateName: latency
        startingStep: 2
        args:
          - name: service-name
            value: product-service
      
      steps:
        - setWeight: 5       # 5% canary
        - pause: { duration: 5m }
        - analysis:          # Check metrics
            templates:
              - templateName: success-rate
        - setWeight: 20      # 20% if analysis passes
        - pause: { duration: 5m }
        - setWeight: 50
        - pause: { duration: 10m }
        - setWeight: 80
        - pause: { duration: 5m }
        # Full rollout after final approval
      
      trafficRouting:
        istio:
          virtualService:
            name: product-service
            routes:
              - primary
          destinationRule:
            name: product-service
            canarySubsetName: canary
            stableSubsetName: stable
```

### 1.4 Rollout Analysis Template

```yaml
# k8s/argo-rollouts/analysis-template.yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
  namespace: production
spec:
  args:
    - name: service-name
  metrics:
    - name: success-rate
      interval: 1m
      successCondition: result[0] >= 0.99  # 99% success rate
      failureCondition: result[0] < 0.95   # Fail if below 95%
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            sum(
              rate(
                istio_requests_total{
                  reporter="destination",
                  destination_service_name="{{args.service-name}}",
                  response_code!~"5.*"
                }[5m]
              )
            ) /
            sum(
              rate(
                istio_requests_total{
                  reporter="destination",
                  destination_service_name="{{args.service-name}}"
                }[5m]
              )
            )
    
    - name: latency-p99
      interval: 1m
      successCondition: result[0] <= 0.5   # p99 <= 500ms
      failureCondition: result[0] > 1.0    # p99 > 1s
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            histogram_quantile(0.99,
              sum(
                rate(
                  istio_request_duration_milliseconds_bucket{
                    reporter="destination",
                    destination_service_name="{{args.service-name}}"
                  }[5m]
                )
              ) by (le)
            ) / 1000

---
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: latency
  namespace: production
spec:
  metrics:
    - name: latency-p95
      interval: 2m
      successCondition: result[0] <= 0.3
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            histogram_quantile(0.95,
              sum(rate(istio_request_duration_milliseconds_bucket[5m])) by (le)
            ) / 1000
```

## 2. A/B Testing with Header-Based Routing

```yaml
# k8s/istio/ab-testing.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: checkout-service
  namespace: production
spec:
  hosts:
    - checkout-service
  http:
    # Segment A: High-value customers see new checkout
    - name: segment-premium
      match:
        - headers:
            x-user-tier:
              exact: "premium"
      route:
        - destination:
            host: checkout-service
            subset: v2-new-ux
    
    # Segment B: Mobile users
    - name: segment-mobile
      match:
        - headers:
            user-agent:
              regex: ".*Mobile.*"
        - headers:
            x-experiment:
              exact: "checkout-v2"
      route:
        - destination:
            host: checkout-service
            subset: v2-new-ux
    
    # Default: old checkout
    - name: default
      route:
        - destination:
            host: checkout-service
            subset: v1-stable
```

## 3. Circuit Breaking

```yaml
# k8s/istio/circuit-breaker.yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: payment-service
  namespace: production
spec:
  host: payment-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
        connectTimeout: 10s
        tcpKeepalive:
          time: 7200s
          interval: 75s
      http:
        http1MaxPendingRequests: 50
        http2MaxRequests: 100
        maxRequestsPerConnection: 10
        maxRetries: 2
        idleTimeout: 90s
        h2UpgradePolicy: UPGRADE
    
    outlierDetection:
      # Consecutive gateway errors before ejection
      consecutiveGatewayErrors: 5
      # Consecutive 5xx errors before ejection
      consecutive5xxErrors: 10
      # Check interval
      interval: 30s
      # How long to eject unhealthy host
      baseEjectionTime: 60s
      # Max percentage of hosts to eject
      maxEjectionPercent: 50
      # Minimum healthy hosts
      minHealthPercent: 30
      # Split external and local origin errors
      splitExternalLocalOriginErrors: true
      consecutiveLocalOriginFailures: 5
```

## 4. Fault Injection Testing

```yaml
# k8s/istio/fault-injection-test.yaml
# For testing resilience in staging environment
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: inventory-service-fault
  namespace: staging
spec:
  hosts:
    - inventory-service
  http:
    - name: inject-delay
      match:
        - headers:
            x-chaos-delay:
              exact: "true"
      fault:
        delay:
          percentage:
            value: 100
          fixedDelay: 5s
      route:
        - destination:
            host: inventory-service
    
    - name: inject-abort
      match:
        - headers:
            x-chaos-abort:
              exact: "true"
      fault:
        abort:
          percentage:
            value: 100
          httpStatus: 503
      route:
        - destination:
            host: inventory-service
    
    - name: mixed-chaos
      match:
        - headers:
            x-chaos-mixed:
              exact: "true"
      fault:
        delay:
          percentage:
            value: 30  # 30% of requests get delay
          fixedDelay: 2s
        abort:
          percentage:
            value: 10  # 10% of requests get 503
          httpStatus: 503
      route:
        - destination:
            host: inventory-service
    
    # Normal traffic
    - route:
        - destination:
            host: inventory-service
```

### 4.1 Chaos Test Script using Fault Injection

```javascript
// scripts/chaos-istio-test.js
const axios = require('axios');
const { KubeConfig, CustomObjectsApi } = require('@kubernetes/client-node');

class IstioFaultInjector {
  constructor() {
    const kc = new KubeConfig();
    kc.loadFromDefault();
    this.k8s = kc.makeApiClient(CustomObjectsApi);
  }
  
  async injectDelay(namespace, service, delayMs, percentage = 100) {
    const vs = await this.getVirtualService(namespace, service);
    
    // Add fault injection to first HTTP route
    vs.spec.http[0].fault = {
      delay: {
        percentage: { value: percentage },
        fixedDelay: `${delayMs}ms`,
      }
    };
    
    await this.updateVirtualService(namespace, service, vs);
    console.log(`Injected ${delayMs}ms delay (${percentage}%) into ${service}`);
  }
  
  async injectAbort(namespace, service, httpStatus = 503, percentage = 100) {
    const vs = await this.getVirtualService(namespace, service);
    
    vs.spec.http[0].fault = {
      abort: {
        percentage: { value: percentage },
        httpStatus,
      }
    };
    
    await this.updateVirtualService(namespace, service, vs);
    console.log(`Injected ${httpStatus} abort (${percentage}%) into ${service}`);
  }
  
  async removeFaultInjection(namespace, service) {
    const vs = await this.getVirtualService(namespace, service);
    
    if (vs.spec.http[0].fault) {
      delete vs.spec.http[0].fault;
      await this.updateVirtualService(namespace, service, vs);
      console.log(`Removed fault injection from ${service}`);
    }
  }
  
  async getVirtualService(namespace, name) {
    const { body } = await this.k8s.getNamespacedCustomObject(
      'networking.istio.io', 'v1beta1',
      namespace, 'virtualservices', name
    );
    return body;
  }
  
  async updateVirtualService(namespace, name, vs) {
    await this.k8s.replaceNamespacedCustomObject(
      'networking.istio.io', 'v1beta1',
      namespace, 'virtualservices', name,
      vs
    );
  }
}

// Chaos test execution
async function runChaosTest() {
  const injector = new IstioFaultInjector();
  const baseUrl = process.env.TEST_URL || 'http://localhost:3000';
  
  console.log('Starting Istio Chaos Test...\n');
  
  // Baseline measurement
  console.log('1. Measuring baseline...');
  const baseline = await measureSuccessRate(baseUrl, 30);
  console.log(`   Baseline success rate: ${baseline.successRate}%`);
  console.log(`   Baseline p95 latency: ${baseline.p95}ms\n`);
  
  // Test 1: 2s delay on inventory
  console.log('2. Injecting 2s delay on inventory service...');
  await injector.injectDelay('production', 'inventory-service', 2000, 50);
  
  await sleep(5000); // Let it stabilize
  
  const withDelay = await measureSuccessRate(baseUrl, 30);
  console.log(`   Success rate with delay: ${withDelay.successRate}%`);
  console.log(`   p95 latency with delay: ${withDelay.p95}ms`);
  
  const resilient = withDelay.successRate >= 99 && withDelay.p95 < 5000;
  console.log(`   Resilience check: ${resilient ? '✅ PASS' : '❌ FAIL'}\n`);
  
  await injector.removeFaultInjection('production', 'inventory-service');
  await sleep(3000);
  
  // Test 2: 503 errors on payment
  console.log('3. Injecting 20% 503 errors on payment service...');
  await injector.injectAbort('production', 'payment-service', 503, 20);
  
  await sleep(5000);
  
  const withErrors = await measureSuccessRate(baseUrl, 30);
  console.log(`   Success rate with errors: ${withErrors.successRate}%`);
  
  // Should handle gracefully - errors should not cascade
  const handlesFaults = withErrors.successRate >= 75;
  console.log(`   Fault tolerance check: ${handlesFaults ? '✅ PASS' : '❌ FAIL'}\n`);
  
  await injector.removeFaultInjection('production', 'payment-service');
  
  console.log('Chaos test completed!');
}

async function measureSuccessRate(url, durationSeconds) {
  const results = [];
  const endTime = Date.now() + durationSeconds * 1000;
  
  while (Date.now() < endTime) {
    const start = Date.now();
    try {
      await axios.get(`${url}/api/v1/products?limit=1`, { timeout: 10000 });
      results.push({ success: true, latency: Date.now() - start });
    } catch {
      results.push({ success: false, latency: Date.now() - start });
    }
    await sleep(500);
  }
  
  const successful = results.filter(r => r.success).length;
  const latencies = results.filter(r => r.success).map(r => r.latency).sort((a, b) => a - b);
  const p95Index = Math.floor(latencies.length * 0.95);
  
  return {
    successRate: ((successful / results.length) * 100).toFixed(1),
    p95: latencies[p95Index] || 0,
    total: results.length,
  };
}

function sleep(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

runChaosTest().catch(console.error);
```

## 5. mTLS (Mutual TLS) Configuration

```yaml
# k8s/istio/mtls-strict.yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT  # All traffic must use mTLS

---
# Allow specific external services to bypass mTLS
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: monitoring-permissive
  namespace: production
spec:
  selector:
    matchLabels:
      app: prometheus-node-exporter
  mtls:
    mode: PERMISSIVE  # Allow both mTLS and plaintext

---
# DestinationRule for mTLS origination
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: enable-mtls-all
  namespace: production
spec:
  host: "*.production.svc.cluster.local"
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL  # Use Istio-managed certificates
```

## 6. AuthorizationPolicy (Zero-Trust)

```yaml
# k8s/istio/authorization-policy.yaml
# Deny all by default
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: production
spec:
  {}  # No action = deny all

---
# Allow specific service-to-service communication
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: product-service-policy
  namespace: production
spec:
  selector:
    matchLabels:
      app: product-service
  action: ALLOW
  rules:
    # Allow order-service to access product-service
    - from:
        - source:
            principals:
              - cluster.local/ns/production/sa/order-service
    # Allow API gateway
    - from:
        - source:
            principals:
              - cluster.local/ns/istio-system/sa/istio-ingressgateway-service-account
      to:
        - operation:
            methods: ["GET"]
            paths: ["/api/v1/products*"]
    # Allow monitoring
    - from:
        - source:
            namespaces: ["monitoring"]
      to:
        - operation:
            paths: ["/metrics"]

---
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: order-service-policy
  namespace: production
spec:
  selector:
    matchLabels:
      app: order-service
  action: ALLOW
  rules:
    - from:
        - source:
            principals:
              - cluster.local/ns/production/sa/api-gateway
    - from:
        - source:
            principals:
              - cluster.local/ns/production/sa/payment-service
      to:
        - operation:
            methods: ["PATCH"]
            paths: ["/internal/orders/*/payment-status"]
```

## 7. Traffic Mirroring (Shadow Traffic)

```yaml
# k8s/istio/traffic-mirror.yaml
# Shadow 10% of production traffic to new service version for testing
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: order-service-mirror
  namespace: production
spec:
  hosts:
    - order-service
  http:
    - name: mirror-to-v2
      route:
        - destination:
            host: order-service
            subset: v1
          weight: 100
      mirror:
        host: order-service
        subset: v2
      mirrorPercentage:
        value: 10.0  # Shadow 10% to v2
```

## 8. Ingress Gateway with TLS Termination

```yaml
# k8s/istio/gateway-tls.yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: api-gateway
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway
  servers:
    - port:
        number: 443
        name: https
        protocol: HTTPS
      tls:
        mode: SIMPLE
        credentialName: api-example-com-cert  # Kubernetes secret with TLS cert
      hosts:
        - api.example.com
    
    - port:
        number: 80
        name: http-redirect
        protocol: HTTP
      tls:
        httpsRedirect: true  # Redirect HTTP to HTTPS
      hosts:
        - api.example.com

---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: api-gateway-routing
  namespace: production
spec:
  hosts:
    - api.example.com
  gateways:
    - istio-system/api-gateway
  http:
    - name: products
      match:
        - uri:
            prefix: /api/v1/products
      route:
        - destination:
            host: product-service
            port:
              number: 3000
    
    - name: orders
      match:
        - uri:
            prefix: /api/v1/orders
      headers:
        request:
          add:
            x-forwarded-proto: https
      route:
        - destination:
            host: order-service
            port:
              number: 3000
      timeout: 30s
    
    - name: auth
      match:
        - uri:
            prefix: /api/v1/auth
      route:
        - destination:
            host: auth-service
            port:
              number: 3000
      corsPolicy:
        allowOrigins:
          - exact: https://app.example.com
          - regex: https://.*\.example\.com
        allowMethods:
          - GET
          - POST
          - PUT
          - PATCH
          - DELETE
        allowHeaders:
          - Authorization
          - Content-Type
          - X-Request-ID
        exposeHeaders:
          - X-Request-ID
        maxAge: 24h
        allowCredentials: true
```

## 9. Service Entry (External Services)

```yaml
# k8s/istio/service-entry.yaml
# Allow access to external Stripe API
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: stripe-api
  namespace: production
spec:
  hosts:
    - api.stripe.com
  location: MESH_EXTERNAL
  ports:
    - number: 443
      name: https
      protocol: HTTPS
  resolution: DNS

---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: stripe-api
  namespace: production
spec:
  host: api.stripe.com
  trafficPolicy:
    tls:
      mode: SIMPLE
      sni: api.stripe.com
    connectionPool:
      tcp:
        maxConnections: 10  # Limit connections to external service
    outlierDetection:
      consecutive5xxErrors: 3
      interval: 30s
      baseEjectionTime: 60s
```

## 10. Observability with Istio

### 10.1 Kiali Service Graph Query

```bash
# Query Kiali API for service dependencies
curl -s "http://kiali:20001/api/namespaces/production/graph" \
  -H "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
  | jq '.elements.nodes[] | {id: .data.id, label: .data.nodeType}'
```

### 10.2 Istio Metrics Dashboard

```yaml
# k8s/monitoring/istio-dashboard.yaml
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
      "title": "Istio Service Metrics",
      "panels": [
        {
          "title": "Request Rate by Service",
          "type": "graph",
          "targets": [
            {
              "expr": "sum(rate(istio_requests_total[5m])) by (destination_service_name)"
            }
          ]
        },
        {
          "title": "Error Rate by Service",
          "type": "graph",
          "targets": [
            {
              "expr": "sum(rate(istio_requests_total{response_code=~'5..'}[5m])) by (destination_service_name) / sum(rate(istio_requests_total[5m])) by (destination_service_name)"
            }
          ]
        },
        {
          "title": "P99 Latency by Service",
          "type": "graph",
          "targets": [
            {
              "expr": "histogram_quantile(0.99, sum(rate(istio_request_duration_milliseconds_bucket[5m])) by (le, destination_service_name))"
            }
          ]
        }
      ]
    }
```

## 11. Progressive Delivery Automation Script

```javascript
// scripts/progressiveDelivery.js
const axios = require('axios');
const { execSync } = require('child_process');

class ProgressiveDeliveryManager {
  constructor(config) {
    this.prometheusUrl = config.prometheusUrl;
    this.namespace = config.namespace;
    this.serviceName = config.serviceName;
    this.stages = config.stages || [5, 10, 25, 50, 75, 100];
    this.stabilizationPeriod = config.stabilizationPeriod || 300; // 5 minutes
  }
  
  async getCurrentWeight() {
    const vs = JSON.parse(
      execSync(`kubectl get vs ${this.serviceName} -n ${this.namespace} -o json`).toString()
    );
    
    const canaryRoute = vs.spec.http[0].route.find(r => r.destination.subset === 'canary');
    return canaryRoute?.weight || 0;
  }
  
  async setCanaryWeight(weight) {
    const vs = JSON.parse(
      execSync(`kubectl get vs ${this.serviceName} -n ${this.namespace} -o json`).toString()
    );
    
    const stableWeight = 100 - weight;
    vs.spec.http[0].route = [
      {
        destination: { host: this.serviceName, subset: 'stable' },
        weight: stableWeight,
      },
      {
        destination: { host: this.serviceName, subset: 'canary' },
        weight,
      },
    ];
    
    require('fs').writeFileSync('/tmp/vs-patch.json', JSON.stringify(vs));
    execSync(`kubectl apply -f /tmp/vs-patch.json`);
    
    console.log(`Set canary weight to ${weight}%`);
  }
  
  async checkMetrics() {
    const [successRate, p99Latency] = await Promise.all([
      this.queryPrometheus(
        `sum(rate(istio_requests_total{reporter="destination",destination_service_name="${this.serviceName}",response_code!~"5.*"}[5m])) / sum(rate(istio_requests_total{reporter="destination",destination_service_name="${this.serviceName}"}[5m]))`
      ),
      this.queryPrometheus(
        `histogram_quantile(0.99, sum(rate(istio_request_duration_milliseconds_bucket{reporter="destination",destination_service_name="${this.serviceName}"}[5m])) by (le)) / 1000`
      ),
    ]);
    
    return {
      successRate: parseFloat(successRate) * 100,
      p99LatencySeconds: parseFloat(p99Latency),
      healthy: parseFloat(successRate) >= 0.99 && parseFloat(p99Latency) < 1.0,
    };
  }
  
  async queryPrometheus(query) {
    const response = await axios.get(`${this.prometheusUrl}/api/v1/query`, {
      params: { query },
    });
    return response.data.data.result[0]?.value[1] || '0';
  }
  
  async rollout(newVersion) {
    console.log(`Starting progressive rollout of ${this.serviceName}:${newVersion}`);
    
    // Deploy canary
    execSync(`kubectl set image deployment/${this.serviceName}-canary app=${newVersion} -n ${this.namespace}`);
    
    // Wait for canary pods to be ready
    execSync(`kubectl rollout status deployment/${this.serviceName}-canary -n ${this.namespace} --timeout=5m`);
    
    for (const targetWeight of this.stages) {
      console.log(`\nProgress to ${targetWeight}%...`);
      await this.setCanaryWeight(targetWeight);
      
      // Stabilization period
      console.log(`Stabilizing for ${this.stabilizationPeriod}s...`);
      await this.sleep(this.stabilizationPeriod * 1000);
      
      // Check metrics
      const metrics = await this.checkMetrics();
      console.log(`Metrics: success=${metrics.successRate.toFixed(2)}%, p99=${metrics.p99LatencySeconds.toFixed(3)}s`);
      
      if (!metrics.healthy) {
        console.error(`❌ Metrics degraded! Rolling back...`);
        await this.rollback();
        return false;
      }
      
      console.log(`✅ Stage ${targetWeight}% healthy`);
    }
    
    console.log('\n✅ Progressive rollout completed successfully!');
    await this.promoteCanary();
    return true;
  }
  
  async rollback() {
    await this.setCanaryWeight(0);
    execSync(`kubectl rollout undo deployment/${this.serviceName}-canary -n ${this.namespace}`);
    console.log('Rollback completed');
  }
  
  async promoteCanary() {
    // Update stable deployment to canary image
    const canaryImage = JSON.parse(
      execSync(`kubectl get deployment ${this.serviceName}-canary -n ${this.namespace} -o json`).toString()
    ).spec.template.spec.containers[0].image;
    
    execSync(`kubectl set image deployment/${this.serviceName} app=${canaryImage} -n ${this.namespace}`);
    await this.setCanaryWeight(0);
    console.log('Canary promoted to stable');
  }
  
  sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

// Usage
const manager = new ProgressiveDeliveryManager({
  prometheusUrl: 'http://prometheus:9090',
  namespace: 'production',
  serviceName: 'product-service',
  stages: [5, 20, 50, 100],
  stabilizationPeriod: 300,
});

manager.rollout(`product-service:${process.env.NEW_VERSION}`)
  .then(success => process.exit(success ? 0 : 1));
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Canary Deployments** - VirtualService traffic splitting + DestinationRule subsets + Argo Rollouts
2. **A/B Testing** - Header/cookie based routing สำหรับ user segments
3. **Circuit Breaking** - outlierDetection + connection pool limits ใน DestinationRule
4. **Fault Injection** - Test resilience ด้วย Istio delay/abort injection
5. **mTLS** - PeerAuthentication STRICT mode + DestinationRule ISTIO_MUTUAL
6. **Zero-Trust** - AuthorizationPolicy แบบ deny-all แล้ว whitelist
7. **Traffic Mirroring** - Shadow production traffic ไปยัง new version
8. **Ingress Gateway** - TLS termination, HTTP redirect, CORS policy
9. **Service Entry** - Allow external service access อย่างปลอดภัย
10. **Progressive Delivery** - Automated rollout ด้วย Prometheus metrics validation

## แบบฝึกหัด

1. Configure canary deployment ด้วย Argo Rollouts + Prometheus analysis
2. Implement A/B test ระหว่าง checkout v1 และ v2 โดยใช้ user tier routing
3. Setup mTLS strict mode และทดสอบว่า services communicate ได้ผ่าน mTLS เท่านั้น
4. สร้าง fault injection test suite ครอบคลุม delay, abort, mixed scenarios
5. Implement circuit breaker สำหรับ external payment API ด้วย outlierDetection

# Part 34: Cost Optimization & FinOps for Microservices

## บทนำ

เมื่อ Microservices architecture scale ขึ้น ค่าใช้จ่าย Cloud ก็เพิ่มตามอย่างรวดเร็ว FinOps (Financial Operations) คือ practice ที่ช่วยให้ทีม engineering เข้าใจและควบคุม cloud costs โดยไม่ลด performance ในบทนี้เราจะเรียนรู้เทคนิค Cost Optimization ระดับ World-Class

## 1. Cost Monitoring & Visibility

### 1.1 Cost Allocation with Kubernetes Labels

```yaml
# k8s/namespace.yaml - All namespaces with cost labels
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    cost-center: "engineering"
    team: "platform"
    environment: "production"
    product: "ecommerce"
    billing-code: "ENG-001"

---
# Deploy with cost labels for granular tracking
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-service
  namespace: production
  labels:
    app: product-service
    cost-center: "engineering"
    team: "catalog"
    feature: "product-management"
    # Used by cost allocation tools
    app.kubernetes.io/name: product-service
    app.kubernetes.io/component: api
    app.kubernetes.io/part-of: ecommerce
```

### 1.2 Cost Dashboard with Custom Metrics

```javascript
// monitoring/costMetrics.js
const prometheus = require('prom-client');
const axios = require('axios');

class CostMetricsCollector {
  constructor() {
    this.register = new prometheus.Registry();
    
    // Track estimated cost per service
    this.estimatedCostGauge = new prometheus.Gauge({
      name: 'service_estimated_cost_usd_hour',
      help: 'Estimated hourly cost in USD for this service instance',
      labelNames: ['service', 'namespace', 'node_type'],
      registers: [this.register],
    });
    
    // CPU cost
    this.cpuCostGauge = new prometheus.Gauge({
      name: 'service_cpu_cost_usd_hour',
      help: 'Estimated CPU cost per hour',
      labelNames: ['service', 'namespace'],
      registers: [this.register],
    });
    
    // Memory cost
    this.memoryCostGauge = new prometheus.Gauge({
      name: 'service_memory_cost_usd_hour',
      help: 'Estimated memory cost per hour',
      labelNames: ['service', 'namespace'],
      registers: [this.register],
    });
    
    // Request cost efficiency
    this.costPerRequestGauge = new prometheus.Gauge({
      name: 'service_cost_per_request_usd',
      help: 'Estimated cost per request in USD',
      labelNames: ['service'],
      registers: [this.register],
    });
  }
  
  async collect() {
    // AWS/GCP pricing (example for t3.medium: $0.0416/hr)
    const NODE_COSTS = {
      't3.medium': { cpu: 0.0104, memoryGB: 0.00702, total: 0.0416 },
      't3.large': { cpu: 0.0208, memoryGB: 0.00702, total: 0.0832 },
      'm5.xlarge': { cpu: 0.0480, memoryGB: 0.00702, total: 0.192 },
    };
    
    const services = await this.getServiceMetrics();
    
    for (const service of services) {
      const nodeCost = NODE_COSTS[service.nodeType] || NODE_COSTS['t3.medium'];
      
      // Estimate based on requested resources
      const cpuCost = (service.cpuRequested / 2) * nodeCost.cpu; // 2 vCPU per t3.medium
      const memoryCost = (service.memoryRequestedGB / 4) * nodeCost.memoryGB * 4; // 4GB per t3.medium
      const totalCost = cpuCost + memoryCost;
      
      this.estimatedCostGauge.set(
        { service: service.name, namespace: service.namespace, node_type: service.nodeType },
        totalCost
      );
      
      this.cpuCostGauge.set({ service: service.name, namespace: service.namespace }, cpuCost);
      this.memoryCostGauge.set({ service: service.name, namespace: service.namespace }, memoryCost);
      
      if (service.requestsPerHour > 0) {
        const costPerRequest = totalCost / service.requestsPerHour;
        this.costPerRequestGauge.set({ service: service.name }, costPerRequest * 1000); // per 1000 requests
      }
    }
  }
  
  async getServiceMetrics() {
    // Get from Prometheus
    const cpuQuery = 'sum by (pod, namespace) (container_cpu_request_cores{container!="POD"})';
    const memQuery = 'sum by (pod, namespace) (container_memory_request_bytes{container!="POD"}) / 1024 / 1024 / 1024';
    
    // Simplified - in reality query Prometheus API
    return [];
  }
}

module.exports = new CostMetricsCollector();
```

## 2. Right-Sizing Resources

### 2.1 Resource Recommendation Engine

```javascript
// tools/resourceRecommender.js
const axios = require('axios');

class ResourceRecommender {
  constructor(prometheusUrl) {
    this.prometheusUrl = prometheusUrl;
    this.lookbackDays = 7;
  }
  
  async analyzeService(namespace, serviceName) {
    const [cpuData, memoryData, replicaData] = await Promise.all([
      this.getCPUUsage(namespace, serviceName),
      this.getMemoryUsage(namespace, serviceName),
      this.getReplicaCount(namespace, serviceName),
    ]);
    
    const recommendations = {
      service: serviceName,
      namespace,
      current: {
        cpuRequest: cpuData.requested,
        cpuLimit: cpuData.limit,
        memoryRequest: memoryData.requested,
        memoryLimit: memoryData.limit,
        replicas: replicaData.current,
      },
      recommended: {
        cpuRequest: this.recommendCPURequest(cpuData),
        cpuLimit: this.recommendCPULimit(cpuData),
        memoryRequest: this.recommendMemoryRequest(memoryData),
        memoryLimit: this.recommendMemoryLimit(memoryData),
        minReplicas: this.recommendMinReplicas(replicaData, cpuData),
        maxReplicas: this.recommendMaxReplicas(replicaData, cpuData),
      },
      estimatedSavings: 0,
    };
    
    recommendations.estimatedSavings = this.calculateSavings(
      recommendations.current,
      recommendations.recommended
    );
    
    return recommendations;
  }
  
  recommendCPURequest(cpuData) {
    // Set request to p90 usage + 20% buffer
    const p90 = cpuData.p90Usage;
    return Math.ceil(p90 * 1.2 * 1000) / 1000; // Round to 3 decimal places
  }
  
  recommendCPULimit(cpuData) {
    // Set limit to p99 usage + 30% buffer (2x request minimum)
    const p99 = cpuData.p99Usage;
    const fromUsage = Math.ceil(p99 * 1.3 * 1000) / 1000;
    return Math.max(fromUsage, this.recommendCPURequest(cpuData) * 2);
  }
  
  recommendMemoryRequest(memData) {
    // Memory: use max observed + 20% buffer (no burst like CPU)
    return Math.ceil(memData.maxUsage * 1.2);
  }
  
  recommendMemoryLimit(memData) {
    // Memory limit: max observed + 50% buffer for GC spikes
    return Math.ceil(memData.maxUsage * 1.5);
  }
  
  recommendMinReplicas(replicaData, cpuData) {
    // Min replicas: ensure we can handle average load at 50% CPU utilization
    const avgLoadFactor = cpuData.avgUsage / this.recommendCPURequest(cpuData);
    return Math.max(2, Math.ceil(avgLoadFactor * 2)); // At least 2 for HA
  }
  
  recommendMaxReplicas(replicaData, cpuData) {
    // Max replicas: can scale to handle p99 load at 70% CPU
    const maxLoadFactor = cpuData.p99Usage / this.recommendCPURequest(cpuData);
    return Math.max(this.recommendMinReplicas(replicaData, cpuData) * 2, 
                    Math.ceil(maxLoadFactor * 1.5));
  }
  
  calculateSavings(current, recommended) {
    // CPU cost per vCPU-hour (t3 pricing)
    const CPU_COST_PER_HOUR = 0.0208; // $/vCPU/hr
    const MEMORY_COST_PER_GB_HOUR = 0.00702; // $/GB/hr
    
    const currentCost = (
      current.cpuRequest * current.replicas * CPU_COST_PER_HOUR +
      (current.memoryRequest / 1024) * current.replicas * MEMORY_COST_PER_GB_HOUR
    ) * 24 * 30; // Monthly
    
    const recommendedCost = (
      recommended.cpuRequest * recommended.minReplicas * CPU_COST_PER_HOUR +
      (recommended.memoryRequest / 1024) * recommended.minReplicas * MEMORY_COST_PER_GB_HOUR
    ) * 24 * 30;
    
    return {
      currentMonthly: Math.round(currentCost * 100) / 100,
      recommendedMonthly: Math.round(recommendedCost * 100) / 100,
      savingsMonthly: Math.round((currentCost - recommendedCost) * 100) / 100,
      savingsPercent: Math.round(((currentCost - recommendedCost) / currentCost) * 100),
    };
  }
  
  async getCPUUsage(namespace, service) {
    const queries = {
      p90: `quantile_over_time(0.9, sum(rate(container_cpu_usage_seconds_total{namespace="${namespace}",pod=~"${service}-.*"}[5m]))[${this.lookbackDays}d:5m])`,
      p99: `quantile_over_time(0.99, sum(rate(container_cpu_usage_seconds_total{namespace="${namespace}",pod=~"${service}-.*"}[5m]))[${this.lookbackDays}d:5m])`,
      avg: `avg_over_time(sum(rate(container_cpu_usage_seconds_total{namespace="${namespace}",pod=~"${service}-.*"}[5m]))[${this.lookbackDays}d:5m])`,
    };
    
    const [p90, p99, avg] = await Promise.all(
      Object.values(queries).map(q => this.query(q))
    );
    
    return {
      p90Usage: parseFloat(p90),
      p99Usage: parseFloat(p99),
      avgUsage: parseFloat(avg),
      requested: 0.5, // From deployment spec (would query k8s API in reality)
      limit: 1.0,
    };
  }
  
  async getMemoryUsage(namespace, service) {
    const maxQuery = `max_over_time(sum(container_memory_working_set_bytes{namespace="${namespace}",pod=~"${service}-.*"})[${this.lookbackDays}d:5m]) / 1024 / 1024`;
    const max = await this.query(maxQuery);
    
    return {
      maxUsage: parseFloat(max),
      requested: 256,  // MiB from deployment
      limit: 512,
    };
  }
  
  async getReplicaCount(namespace, service) {
    const minQuery = `min_over_time(kube_deployment_status_replicas_ready{namespace="${namespace}",deployment="${service}"}[${this.lookbackDays}d:1h])`;
    const maxQuery = `max_over_time(kube_deployment_status_replicas_ready{namespace="${namespace}",deployment="${service}"}[${this.lookbackDays}d:1h])`;
    
    const [min, max] = await Promise.all([
      this.query(minQuery),
      this.query(maxQuery),
    ]);
    
    return { min: parseInt(min), max: parseInt(max), current: parseInt(max) };
  }
  
  async query(promQL) {
    const response = await axios.get(`${this.prometheusUrl}/api/v1/query`, {
      params: { query: promQL },
    });
    return response.data.data.result[0]?.value[1] || '0';
  }
  
  async generateReport(namespace) {
    const { execSync } = require('child_process');
    const deployments = JSON.parse(
      execSync(`kubectl get deployments -n ${namespace} -o json`).toString()
    );
    
    const services = deployments.items.map(d => d.metadata.name);
    const analyses = await Promise.all(
      services.map(s => this.analyzeService(namespace, s))
    );
    
    const totalCurrentCost = analyses.reduce((sum, a) => sum + a.estimatedSavings.currentMonthly, 0);
    const totalRecommendedCost = analyses.reduce((sum, a) => sum + a.estimatedSavings.recommendedMonthly, 0);
    const totalSavings = totalCurrentCost - totalRecommendedCost;
    
    console.log('\n💰 Resource Right-Sizing Report');
    console.log('='.repeat(60));
    console.log(`Namespace: ${namespace}`);
    console.log(`Total Current Monthly Cost: $${totalCurrentCost.toFixed(2)}`);
    console.log(`Total Recommended Monthly Cost: $${totalRecommendedCost.toFixed(2)}`);
    console.log(`Potential Monthly Savings: $${totalSavings.toFixed(2)} (${Math.round((totalSavings/totalCurrentCost)*100)}%)\n`);
    
    analyses
      .sort((a, b) => b.estimatedSavings.savingsMonthly - a.estimatedSavings.savingsMonthly)
      .forEach(analysis => {
        if (analysis.estimatedSavings.savingsMonthly > 0) {
          console.log(`📦 ${analysis.service}`);
          console.log(`   CPU: ${analysis.current.cpuRequest} → ${analysis.recommended.cpuRequest} cores`);
          console.log(`   Memory: ${analysis.current.memoryRequest}Mi → ${analysis.recommended.memoryRequest}Mi`);
          console.log(`   Replicas: ${analysis.current.replicas} → ${analysis.recommended.minReplicas}-${analysis.recommended.maxReplicas}`);
          console.log(`   Savings: $${analysis.estimatedSavings.savingsMonthly}/month (${analysis.estimatedSavings.savingsPercent}%)\n`);
        }
      });
    
    return analyses;
  }
}

module.exports = ResourceRecommender;
```

## 3. Spot/Preemptible Instances

### 3.1 Mixed Instance Node Groups

```yaml
# k8s/node-groups/mixed-instances.yaml
# EKS Managed Node Group with mixed instances (Spot + On-Demand)
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: production-cluster
  region: ap-southeast-1

managedNodeGroups:
  # On-demand: System/stateful workloads
  - name: system-ondemand
    instanceType: m5.large
    minSize: 3
    maxSize: 10
    labels:
      workload-type: system
      capacity-type: on-demand
    taints:
      - key: workload
        value: system
        effect: NoSchedule
  
  # Spot: Stateless applications (huge cost savings ~70%)
  - name: app-spot
    instanceTypes:
      - t3.large
      - t3a.large
      - m5.large
      - m5a.large
      - m4.large
    minSize: 5
    maxSize: 100
    spot: true
    labels:
      workload-type: application
      capacity-type: spot
    taints:
      - key: capacity-type
        value: spot
        effect: NoSchedule
```

### 3.2 Workload Spot Toleration

```yaml
# k8s/deployments/product-service.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-service
spec:
  replicas: 5
  template:
    spec:
      # Tolerate spot instances
      tolerations:
        - key: capacity-type
          operator: Equal
          value: spot
          effect: NoSchedule
      
      # Prefer spot but fallback to on-demand
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 80
              preference:
                matchExpressions:
                  - key: capacity-type
                    operator: In
                    values: [spot]
            - weight: 20
              preference:
                matchExpressions:
                  - key: capacity-type
                    operator: In
                    values: [on-demand]
      
      # Spread across availability zones
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: product-service
      
      # Graceful termination for spot interrupts
      terminationGracePeriodSeconds: 120
      
      containers:
        - name: product-service
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 10; /app/drain.sh"]
```

### 3.3 Spot Interruption Handler

```javascript
// handlers/spotInterruptionHandler.js
const http = require('http');
const { execSync } = require('child_process');

class SpotInterruptionHandler {
  constructor(app) {
    this.app = app;
    this.isTerminating = false;
    this.server = null;
  }
  
  start() {
    // Poll EC2 instance metadata for termination notice
    this.pollMetadata();
    
    // Handle SIGTERM (Kubernetes sends this before termination)
    process.on('SIGTERM', () => {
      console.log('SIGTERM received, starting graceful shutdown...');
      this.gracefulShutdown();
    });
    
    // Handle SIGINT (Ctrl+C in development)
    process.on('SIGINT', () => {
      console.log('SIGINT received, starting graceful shutdown...');
      this.gracefulShutdown();
    });
  }
  
  async pollMetadata() {
    const METADATA_URL = 'http://169.254.169.254/latest/meta-data/spot/termination-time';
    
    setInterval(async () => {
      if (this.isTerminating) return;
      
      try {
        const response = await fetch(METADATA_URL, { timeout: 2000 });
        if (response.status === 200) {
          const terminationTime = await response.text();
          console.log(`Spot termination notice received! Termination at: ${terminationTime}`);
          this.gracefulShutdown();
        }
      } catch {
        // 404 means no termination notice (normal operation)
      }
    }, 5000); // Check every 5 seconds (termination notice appears ~2 min before)
  }
  
  async gracefulShutdown() {
    if (this.isTerminating) return;
    this.isTerminating = true;
    
    console.log('Starting graceful shutdown...');
    
    // 1. Stop accepting new requests
    if (this.server) {
      this.server.close(() => console.log('HTTP server closed'));
    }
    
    // 2. Wait for in-flight requests to complete (max 30s)
    await this.waitForInflightRequests(30000);
    
    // 3. Flush any buffered events/logs
    await this.flushPendingData();
    
    // 4. Close database connections
    await this.closeConnections();
    
    console.log('Graceful shutdown complete');
    process.exit(0);
  }
  
  async waitForInflightRequests(maxWaitMs) {
    return new Promise(resolve => {
      const start = Date.now();
      const check = () => {
        const activeConnections = this.app.getActiveConnections?.() || 0;
        if (activeConnections === 0 || Date.now() - start > maxWaitMs) {
          resolve();
        } else {
          console.log(`Waiting for ${activeConnections} in-flight requests...`);
          setTimeout(check, 1000);
        }
      };
      check();
    });
  }
  
  async flushPendingData() {
    // Flush metrics, logs, etc.
    console.log('Flushing pending data...');
  }
  
  async closeConnections() {
    console.log('Closing database connections...');
    // Close DB pools, Redis, etc.
  }
}

module.exports = SpotInterruptionHandler;
```

## 4. Autoscaling Cost Optimization

### 4.1 KEDA (Kubernetes Event-Driven Autoscaling)

```yaml
# k8s/keda/order-processor-scaler.yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-processor
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-processor
  
  minReplicaCount: 0   # Scale to zero when idle (huge cost savings!)
  maxReplicaCount: 50
  
  pollingInterval: 30
  cooldownPeriod: 300  # Wait 5 minutes before scaling down
  
  triggers:
    - type: kafka
      metadata:
        bootstrapServers: kafka:9092
        consumerGroup: order-processor
        topic: orders.created
        lagThreshold: "5"  # Scale up when 5+ messages per replica
        offsetResetPolicy: latest
    
    - type: prometheus
      metadata:
        serverAddress: http://prometheus:9090
        metricName: order_queue_depth
        query: sum(order_queue_depth{service="order-processor"})
        threshold: "10"  # Scale up when 10+ items queued

---
# Scale to zero completely during off-hours
apiVersion: keda.sh/v1alpha1
kind: ScaledJob
metadata:
  name: batch-report-generator
  namespace: production
spec:
  jobTargetRef:
    template:
      spec:
        containers:
          - name: report-generator
            image: report-generator:latest
            command: ["node", "generateReport.js"]
        restartPolicy: Never
  
  pollingInterval: 60
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 5
  
  maxReplicaCount: 5
  
  triggers:
    - type: cron
      metadata:
        timezone: Asia/Bangkok
        start: "0 6 * * *"   # Run at 6 AM daily
        end: "0 8 * * *"     # End at 8 AM
        desiredReplicas: "3"  # Only run during these hours
```

### 4.2 Vertical Pod Autoscaler

```yaml
# k8s/vpa/product-service-vpa.yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: product-service
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: product-service
  
  updatePolicy:
    updateMode: Auto  # Automatically update pod resources
  
  resourcePolicy:
    containerPolicies:
      - containerName: product-service
        minAllowed:
          cpu: 50m
          memory: 64Mi
        maxAllowed:
          cpu: 2
          memory: 2Gi
        controlledResources:
          - cpu
          - memory
        controlledValues: RequestsAndLimits
```

## 5. Cost-Aware Caching Strategy

```javascript
// services/tieredCache.js
const Redis = require('ioredis');
const LRU = require('lru-cache');

class CostAwareCacheService {
  constructor(config) {
    // L1: In-memory (free, ultra-fast, small capacity)
    this.l1Cache = new LRU({
      max: 1000,           // Max 1000 items
      maxSize: 100 * 1024 * 1024, // 100MB
      sizeCalculation: (v) => Buffer.byteLength(JSON.stringify(v)),
      ttl: 60 * 1000,     // 1 minute TTL
    });
    
    // L2: Redis (costs money, fast, larger capacity)
    this.l2Cache = new Redis(config.redis);
    
    // Track cache economics
    this.stats = {
      l1Hits: 0,
      l2Hits: 0,
      misses: 0,
      computeCalls: 0,
    };
  }
  
  async get(key, fetchFn, options = {}) {
    const {
      l1TTL = 60,          // 1 minute in L1
      l2TTL = 3600,        // 1 hour in L2
      expensive = false,    // Is the fetchFn expensive (DB query, API call)?
    } = options;
    
    // Check L1 first (free)
    const l1Value = this.l1Cache.get(key);
    if (l1Value !== undefined) {
      this.stats.l1Hits++;
      return l1Value;
    }
    
    // Check L2 (cheap)
    const l2Value = await this.l2Cache.get(key);
    if (l2Value !== null) {
      this.stats.l2Hits++;
      const parsed = JSON.parse(l2Value);
      
      // Warm L1 cache
      this.l1Cache.set(key, parsed, { ttl: l1TTL * 1000 });
      
      return parsed;
    }
    
    // Cache miss - call the actual function
    this.stats.misses++;
    this.stats.computeCalls++;
    
    const value = await fetchFn();
    
    // Store in both caches
    this.l1Cache.set(key, value, { ttl: l1TTL * 1000 });
    await this.l2Cache.setex(key, l2TTL, JSON.stringify(value));
    
    return value;
  }
  
  async invalidate(pattern) {
    // Clear L1
    if (pattern.includes('*')) {
      // Can't do pattern matching in LRU, clear all matching keys
      for (const key of this.l1Cache.keys()) {
        if (this.matchesPattern(key, pattern)) {
          this.l1Cache.delete(key);
        }
      }
    } else {
      this.l1Cache.delete(pattern);
    }
    
    // Clear L2
    const keys = await this.l2Cache.keys(pattern);
    if (keys.length > 0) {
      await this.l2Cache.del(...keys);
    }
  }
  
  getEfficiencyReport() {
    const total = this.stats.l1Hits + this.stats.l2Hits + this.stats.misses;
    
    return {
      l1HitRate: total > 0 ? (this.stats.l1Hits / total * 100).toFixed(1) + '%' : '0%',
      l2HitRate: total > 0 ? (this.stats.l2Hits / total * 100).toFixed(1) + '%' : '0%',
      missRate: total > 0 ? (this.stats.misses / total * 100).toFixed(1) + '%' : '0%',
      
      // Cost estimation (L1=free, L2=Redis cost, Compute=DB/API cost)
      redisRequests: this.stats.l2Hits + this.stats.misses,
      computeRequests: this.stats.misses,
      
      // If 1000 RPM with above stats, cost estimate
      estimatedRedisCallsPerHour: (this.stats.l2Hits + this.stats.misses) / total * 1000 * 60,
      estimatedComputeCallsPerHour: this.stats.misses / total * 1000 * 60,
    };
  }
  
  matchesPattern(key, pattern) {
    const regex = new RegExp('^' + pattern.replace(/\*/g, '.*') + '$');
    return regex.test(key);
  }
}

module.exports = CostAwareCacheService;
```

## 6. Cost Optimization CI/CD Gates

```yaml
# .github/workflows/cost-check.yml
name: Cost Impact Analysis

on:
  pull_request:
    paths:
      - 'k8s/**'
      - 'helm/**'

jobs:
  cost-analysis:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0
      
      - name: Install infracost
        uses: infracost/actions/setup@v2
        with:
          api-key: ${{ secrets.INFRACOST_API_KEY }}
      
      - name: Checkout base branch
        uses: actions/checkout@v3
        with:
          ref: ${{ github.event.pull_request.base.sha }}
          path: base
      
      - name: Generate base cost
        run: |
          infracost breakdown \
            --path base/k8s \
            --format json \
            --out-file /tmp/infracost-base.json
      
      - name: Generate PR cost
        run: |
          infracost breakdown \
            --path k8s \
            --format json \
            --out-file /tmp/infracost-pr.json
      
      - name: Compare costs
        id: cost-compare
        run: |
          DIFF=$(infracost diff \
            --path /tmp/infracost-pr.json \
            --compare-to /tmp/infracost-base.json \
            --format json)
          
          MONTHLY_COST=$(echo $DIFF | jq '.diffTotalMonthlyCost')
          echo "monthly_cost=$MONTHLY_COST" >> $GITHUB_OUTPUT
          echo "diff_json=$DIFF" >> $GITHUB_OUTPUT
      
      - name: Post cost comment
        uses: infracost/actions/comment@v1
        with:
          path: /tmp/infracost-pr.json
          compare-to: /tmp/infracost-base.json
          behavior: update
      
      - name: Fail if cost increase > $100/month
        if: steps.cost-compare.outputs.monthly_cost > 100
        run: |
          echo "❌ Cost increase of ${{ steps.cost-compare.outputs.monthly_cost }}/month exceeds $100 limit"
          echo "Please review resource allocations or get approval from FinOps team"
          exit 1
```

## 7. FinOps Tagging Strategy

```javascript
// scripts/enforceTagging.js
const { KubeConfig, AppsV1Api, CoreV1Api } = require('@kubernetes/client-node');

const REQUIRED_TAGS = ['cost-center', 'team', 'environment', 'product'];

class TaggingEnforcer {
  constructor() {
    const kc = new KubeConfig();
    kc.loadFromDefault();
    this.appsApi = kc.makeApiClient(AppsV1Api);
    this.coreApi = kc.makeApiClient(CoreV1Api);
  }
  
  async auditNamespace(namespace) {
    const violations = [];
    
    // Check deployments
    const { body: deployments } = await this.appsApi.listNamespacedDeployment(namespace);
    
    for (const deployment of deployments.items) {
      const missingTags = REQUIRED_TAGS.filter(
        tag => !deployment.metadata.labels?.[tag]
      );
      
      if (missingTags.length > 0) {
        violations.push({
          type: 'Deployment',
          name: deployment.metadata.name,
          namespace,
          missingTags,
        });
      }
    }
    
    // Check services
    const { body: services } = await this.coreApi.listNamespacedService(namespace);
    
    for (const service of services.items) {
      if (service.metadata.name === 'kubernetes') continue;
      
      const missingTags = REQUIRED_TAGS.filter(
        tag => !service.metadata.labels?.[tag]
      );
      
      if (missingTags.length > 0) {
        violations.push({
          type: 'Service',
          name: service.metadata.name,
          namespace,
          missingTags,
        });
      }
    }
    
    return violations;
  }
  
  async generateReport(namespaces) {
    console.log('\n🏷️  FinOps Tagging Compliance Report');
    console.log('='.repeat(60));
    
    let totalViolations = 0;
    
    for (const namespace of namespaces) {
      const violations = await this.auditNamespace(namespace);
      
      if (violations.length > 0) {
        console.log(`\n⚠️  Namespace: ${namespace} (${violations.length} violations)`);
        violations.forEach(v => {
          console.log(`   ${v.type}/${v.name}: missing [${v.missingTags.join(', ')}]`);
        });
        totalViolations += violations.length;
      } else {
        console.log(`\n✅ Namespace: ${namespace} - fully compliant`);
      }
    }
    
    console.log(`\nTotal violations: ${totalViolations}`);
    
    if (totalViolations > 0) {
      process.exit(1); // Fail CI/CD if violations exist
    }
  }
}

// Run
const enforcer = new TaggingEnforcer();
enforcer.generateReport(['production', 'staging']).catch(console.error);
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Cost Visibility** - Label strategy สำหรับ cost allocation per service/team/product
2. **Right-Sizing** - Automated resource recommendations จาก Prometheus metrics
3. **Spot Instances** - 60-70% cost savings ด้วย mixed instance node groups
4. **Scale to Zero** - KEDA scaling ตาม Kafka lag + Prometheus metrics
5. **VPA** - Vertical Pod Autoscaler ปรับ resources อัตโนมัติ
6. **Tiered Caching** - L1 (free) + L2 (Redis) เพื่อลด compute costs
7. **Cost Gates in CI/CD** - Infracost ป้องกัน unexpected cost increases
8. **FinOps Tagging** - Enforce required labels เพื่อ cost attribution

## แบบฝึกหัด

1. Implement resource right-sizing สำหรับ services ทั้งหมดใน staging namespace
2. Configure mixed spot/on-demand node groups ใน EKS/GKE
3. Setup KEDA ScaledObject สำหรับ background job workers ที่ scale to zero
4. สร้าง cost optimization CI/CD gate ด้วย Infracost
5. Implement tagging compliance check ที่ fail PR เมื่อ missing required labels

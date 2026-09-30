# Part 88: Microservices Capacity Planning

## บทนำ

Capacity Planning คือการวางแผนล่วงหน้าว่าระบบต้องการ resources เท่าไหร่เพื่อรองรับ load ที่คาดการณ์ไว้ การทำ Capacity Planning ที่ดีช่วยประหยัดค่าใช้จ่าย ป้องกัน outage และทำให้ทีมมั่นใจในการ scale

---

## Load Forecasting

### การเก็บข้อมูล Baseline

```typescript
// services/metrics/src/capacity/traffic-analyzer.ts

import { Injectable } from '@nestjs/common';
import { PrometheusService } from './prometheus.service';

@Injectable()
export class TrafficAnalyzer {
  constructor(private readonly prometheus: PrometheusService) {}

  async analyzeTrafficPatterns(): Promise<TrafficPattern> {
    // ดึง 30 วันย้อนหลัง
    const thirtyDaysAgo = Math.floor(Date.now() / 1000) - 30 * 24 * 3600;
    const now = Math.floor(Date.now() / 1000);

    // Request rate per hour
    const hourlyRates = await this.prometheus.queryRange(
      'sum(rate(http_requests_total[5m]))',
      thirtyDaysAgo,
      now,
      '1h'
    );

    // Peak analysis
    const peakRPS = Math.max(...hourlyRates.map(r => r.value));
    const avgRPS = hourlyRates.reduce((sum, r) => sum + r.value, 0) / hourlyRates.length;
    const medianRPS = this.median(hourlyRates.map(r => r.value));

    // Day-of-week pattern
    const dowPattern = this.analyzeDayOfWeekPattern(hourlyRates);

    // Peak hours analysis
    const peakHours = this.analyzePeakHours(hourlyRates);

    return {
      peakRPS,
      avgRPS,
      medianRPS,
      peakToAvgRatio: peakRPS / avgRPS,
      dowPattern,
      peakHours,
      
      // Thai market specific patterns
      patterns: {
        campaignEffect: this.detectCampaignEffect(hourlyRates),
        lunchTimePeak: this.detectLunchTimePeak(hourlyRates),
        paydayEffect: this.detectPaydayEffect(hourlyRates),
      },
    };
  }

  // คำนวณ growth rate จาก historical data
  async calculateGrowthRate(weeks = 12): Promise<GrowthAnalysis> {
    const data = await this.getWeeklyPeaks(weeks);
    
    // Linear regression สำหรับ trend
    const regression = this.linearRegression(
      data.map((_, i) => i),
      data.map(d => d.peakRPS)
    );

    const weeklyGrowthRate = regression.slope / data[0].peakRPS;
    const monthlyGrowthRate = weeklyGrowthRate * 4;
    const yearlyGrowthRate = weeklyGrowthRate * 52;

    return {
      weeklyGrowthPercent: weeklyGrowthRate * 100,
      monthlyGrowthPercent: monthlyGrowthRate * 100,
      yearlyGrowthPercent: yearlyGrowthRate * 100,
      currentPeakRPS: data[data.length - 1].peakRPS,
      projectedPeakRPS: {
        in3months: data[data.length - 1].peakRPS * (1 + monthlyGrowthRate * 3),
        in6months: data[data.length - 1].peakRPS * (1 + monthlyGrowthRate * 6),
        in12months: data[data.length - 1].peakRPS * (1 + yearlyGrowthRate),
      },
    };
  }

  private linearRegression(xs: number[], ys: number[]): { slope: number; intercept: number } {
    const n = xs.length;
    const sumX = xs.reduce((a, b) => a + b, 0);
    const sumY = ys.reduce((a, b) => a + b, 0);
    const sumXY = xs.reduce((sum, x, i) => sum + x * ys[i], 0);
    const sumX2 = xs.reduce((sum, x) => sum + x * x, 0);
    
    const slope = (n * sumXY - sumX * sumY) / (n * sumX2 - sumX * sumX);
    const intercept = (sumY - slope * sumX) / n;
    
    return { slope, intercept };
  }

  private median(values: number[]): number {
    const sorted = [...values].sort((a, b) => a - b);
    const mid = Math.floor(sorted.length / 2);
    return sorted.length % 2 !== 0
      ? sorted[mid]
      : (sorted[mid - 1] + sorted[mid]) / 2;
  }
}
```

---

## Capacity Modeling

### Resource Estimation Calculator

```typescript
// tools/capacity/src/capacity-calculator.ts

interface ServiceCapacityConfig {
  name: string;
  
  // Current measurements
  currentRPS: number;           // requests per second
  avgResponseTimeMs: number;    // average response time
  p99ResponseTimeMs: number;    // p99 response time
  cpuUsagePercentAtCurrentLoad: number;  // CPU % at current load
  memoryUsageMBAtCurrentLoad: number;   // Memory MB at current load
  
  // Resource per pod
  podCPULimit: string;     // e.g., "1000m"
  podMemoryLimit: string;  // e.g., "512Mi"
  
  // Current scale
  currentReplicas: number;
}

interface CapacityProjection {
  targetRPS: number;
  requiredReplicas: number;
  requiredCPU: string;
  requiredMemory: string;
  estimatedCostPerMonth: number;
  safetyMarginPercent: number;
}

export class CapacityCalculator {
  // Little's Law: N = λ × W
  // N = concurrent users, λ = arrival rate, W = time in system
  calculateConcurrentUsers(
    rps: number,
    avgResponseTimeMs: number
  ): number {
    return rps * (avgResponseTimeMs / 1000);
  }

  // คำนวณจำนวน replicas ที่ต้องการ
  calculateRequiredReplicas(
    config: ServiceCapacityConfig,
    targetRPS: number,
    safetyMarginPercent = 30,
  ): CapacityProjection {
    const scaleFactor = targetRPS / config.currentRPS;
    const safetyMultiplier = 1 + safetyMarginPercent / 100;

    // CPU-based calculation
    const cpuUtilizationRatio = config.cpuUsagePercentAtCurrentLoad / 100;
    const replicasForCPU = Math.ceil(
      config.currentReplicas * scaleFactor * safetyMultiplier * cpuUtilizationRatio * (1 / 0.7) // target 70% utilization
    );

    // Memory-based calculation
    const memoryPerRequestMB = config.memoryUsageMBAtCurrentLoad / config.currentRPS;
    const targetMemoryMB = memoryPerRequestMB * targetRPS * safetyMultiplier;
    const podMemoryMB = this.parseMemory(config.podMemoryLimit);
    const replicasForMemory = Math.ceil(targetMemoryMB / (podMemoryMB * 0.8));

    const requiredReplicas = Math.max(replicasForCPU, replicasForMemory, 2); // minimum 2 for HA

    // Cost estimation (AWS EKS on-demand, ap-southeast-1)
    const podCPU = this.parseCPU(config.podCPULimit);
    const totalCPU = podCPU * requiredReplicas;
    const totalMemoryGB = (podMemoryMB / 1024) * requiredReplicas;
    
    // Rough cost: EC2 m5.xlarge (4 vCPU, 16GB) = ~$0.23/hour in Singapore
    const estimatedCostPerMonth = 
      (totalCPU / 4 + totalMemoryGB / 16) * 0.23 * 24 * 30;

    return {
      targetRPS,
      requiredReplicas,
      requiredCPU: `${Math.ceil(podCPU * requiredReplicas)}m`,
      requiredMemory: `${Math.ceil(podMemoryMB * requiredReplicas)}Mi`,
      estimatedCostPerMonth,
      safetyMarginPercent,
    };
  }

  // Generate capacity plan
  generateCapacityPlan(
    services: ServiceCapacityConfig[],
    scenarios: { name: string; multiplier: number }[],
  ): CapacityPlan {
    const plan: CapacityPlan = {
      generatedAt: new Date(),
      scenarios: [],
    };

    for (const scenario of scenarios) {
      const serviceProjections = services.map(service => ({
        serviceName: service.name,
        projection: this.calculateRequiredReplicas(
          service,
          service.currentRPS * scenario.multiplier,
        ),
      }));

      const totalMonthlyCost = serviceProjections.reduce(
        (sum, s) => sum + s.projection.estimatedCostPerMonth,
        0
      );

      plan.scenarios.push({
        name: scenario.name,
        loadMultiplier: scenario.multiplier,
        services: serviceProjections,
        totalMonthlyCostUSD: totalMonthlyCost,
      });
    }

    return plan;
  }

  private parseCPU(cpu: string): number {
    if (cpu.endsWith('m')) return parseInt(cpu) / 1000;
    return parseFloat(cpu);
  }

  private parseMemory(memory: string): number {
    if (memory.endsWith('Mi')) return parseInt(memory);
    if (memory.endsWith('Gi')) return parseInt(memory) * 1024;
    return parseInt(memory);
  }
}
```

---

## Stress Testing

### k6 Load Testing Scripts

```javascript
// tests/load/order-service.k6.js
// Full load test suite สำหรับ Order Service

import http from 'k6/http';
import { check, group, sleep } from 'k6';
import { Rate, Trend, Counter } from 'k6/metrics';
import { SharedArray } from 'k6/data';

// Custom metrics
const errorRate = new Rate('error_rate');
const orderCreationDuration = new Trend('order_creation_duration_ms');
const successfulOrders = new Counter('successful_orders');

// Test configuration
export const options = {
  scenarios: {
    // Smoke test - ตรวจสอบว่าระบบ work
    smoke: {
      executor: 'constant-vus',
      vus: 1,
      duration: '1m',
      tags: { test_type: 'smoke' },
    },
    
    // Load test - normal expected load
    load: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '5m', target: 50 },   // ramp up
        { duration: '20m', target: 50 },  // stay at 50 VUs
        { duration: '5m', target: 0 },    // ramp down
      ],
      tags: { test_type: 'load' },
    },
    
    // Stress test - beyond normal load
    stress: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '5m', target: 100 },
        { duration: '10m', target: 100 },
        { duration: '5m', target: 200 },
        { duration: '10m', target: 200 },
        { duration: '5m', target: 300 },
        { duration: '10m', target: 300 },
        { duration: '5m', target: 0 },
      ],
      tags: { test_type: 'stress' },
    },
    
    // Spike test - sudden traffic spike
    spike: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '30s', target: 10 },
        { duration: '1m', target: 500 }, // spike!
        { duration: '30s', target: 10 },
      ],
      tags: { test_type: 'spike' },
    },
    
    // Soak test - extended normal load
    soak: {
      executor: 'constant-vus',
      vus: 50,
      duration: '2h',
      tags: { test_type: 'soak' },
    },
  },
  
  thresholds: {
    'http_req_duration': ['p(95)<500', 'p(99)<1000'],
    'http_req_failed': ['rate<0.01'],         // < 1% error rate
    'order_creation_duration_ms': ['p(95)<300'],
    'error_rate': ['rate<0.01'],
  },
};

const BASE_URL = __ENV.BASE_URL || 'https://api-staging.company.com';
const AUTH_TOKEN = __ENV.AUTH_TOKEN;

// Preload test data
const testUsers = new SharedArray('users', () => {
  return JSON.parse(open('./data/test-users.json'));
});

const testProducts = new SharedArray('products', () => {
  return JSON.parse(open('./data/test-products.json'));
});

export default function() {
  const user = testUsers[Math.floor(Math.random() * testUsers.length)];
  
  group('Order Creation Flow', () => {
    const headers = {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${AUTH_TOKEN || user.token}`,
      'X-Request-ID': `k6-${__VU}-${__ITER}-${Date.now()}`,
      'Idempotency-Key': `k6-order-${__VU}-${__ITER}`,
    };

    // Step 1: Create order
    const orderPayload = JSON.stringify({
      items: [
        {
          productId: testProducts[Math.floor(Math.random() * testProducts.length)].id,
          quantity: Math.floor(Math.random() * 3) + 1,
        }
      ],
      shippingAddress: {
        street: '123 Sukhumvit Road',
        city: 'Bangkok',
        province: 'Bangkok',
        postalCode: '10110',
        countryCode: 'TH',
      },
    });

    const createStart = Date.now();
    const createResponse = http.post(
      `${BASE_URL}/api/v1/orders`,
      orderPayload,
      { headers, timeout: '10s' }
    );
    orderCreationDuration.add(Date.now() - createStart);

    const createSuccess = check(createResponse, {
      'create order: status 201': (r) => r.status === 201,
      'create order: has orderId': (r) => {
        const body = JSON.parse(r.body);
        return body.id !== undefined;
      },
      'create order: response time < 500ms': (r) => r.timings.duration < 500,
    });

    if (!createSuccess) {
      errorRate.add(1);
      return;
    }

    errorRate.add(0);
    successfulOrders.add(1);

    const order = JSON.parse(createResponse.body);

    // Step 2: Get order
    const getResponse = http.get(
      `${BASE_URL}/api/v1/orders/${order.id}`,
      { headers }
    );

    check(getResponse, {
      'get order: status 200': (r) => r.status === 200,
      'get order: correct ID': (r) => JSON.parse(r.body).id === order.id,
    });

    sleep(Math.random() * 2 + 1); // Think time 1-3 seconds
  });
}

export function handleSummary(data) {
  return {
    'results/load-test-summary.json': JSON.stringify(data, null, 2),
    'results/load-test-summary.html': htmlReport(data),
  };
}
```

```bash
#!/bin/bash
# scripts/run-load-test.sh

# Run specific scenario
k6 run \
  --env BASE_URL=https://api-staging.company.com \
  --env AUTH_TOKEN=$(cat .test-token) \
  --out influxdb=http://influxdb:8086/k6 \
  --tag environment=staging \
  --scenario load \
  tests/load/order-service.k6.js

# Run all scenarios (parallel)
k6 run \
  --env BASE_URL=https://api-staging.company.com \
  tests/load/order-service.k6.js \
  2>&1 | tee results/$(date +%Y%m%d_%H%M%S)-load-test.log
```

---

## Bottleneck Analysis

### Resource Profiling ใน Production

```typescript
// services/order/src/capacity/bottleneck-detector.ts

import { Injectable, Logger } from '@nestjs/common';

interface BottleneckAnalysis {
  bottleneck: 'CPU' | 'MEMORY' | 'DATABASE' | 'NETWORK' | 'NONE';
  severity: 'LOW' | 'MEDIUM' | 'HIGH' | 'CRITICAL';
  details: string;
  recommendations: string[];
}

@Injectable()
export class BottleneckDetector {
  private readonly logger = new Logger(BottleneckDetector.name);

  async analyze(): Promise<BottleneckAnalysis[]> {
    const bottlenecks: BottleneckAnalysis[] = [];

    // Check CPU bottleneck
    const cpuUsage = await this.getCpuUsage();
    if (cpuUsage > 0.8) {
      bottlenecks.push({
        bottleneck: 'CPU',
        severity: cpuUsage > 0.95 ? 'CRITICAL' : 'HIGH',
        details: `CPU usage at ${(cpuUsage * 100).toFixed(1)}%`,
        recommendations: [
          'Scale horizontally (increase replica count)',
          'Increase CPU limits',
          'Profile CPU-intensive operations',
          'Check for blocking event loop operations',
        ],
      });
    }

    // Check memory bottleneck
    const memUsage = await this.getMemoryUsage();
    if (memUsage > 0.85) {
      bottlenecks.push({
        bottleneck: 'MEMORY',
        severity: memUsage > 0.95 ? 'CRITICAL' : 'HIGH',
        details: `Memory usage at ${(memUsage * 100).toFixed(1)}%`,
        recommendations: [
          'Check for memory leaks',
          'Increase memory limits',
          'Review caching strategies',
          'Enable GC profiling',
        ],
      });
    }

    // Check database connection pool
    const dbPoolUtilization = await this.getDBPoolUtilization();
    if (dbPoolUtilization > 0.9) {
      bottlenecks.push({
        bottleneck: 'DATABASE',
        severity: dbPoolUtilization > 0.95 ? 'CRITICAL' : 'HIGH',
        details: `DB connection pool ${(dbPoolUtilization * 100).toFixed(1)}% utilized`,
        recommendations: [
          'Increase connection pool size',
          'Add read replicas for read-heavy workloads',
          'Implement connection pooling (PgBouncer)',
          'Optimize slow queries',
          'Consider caching frequently read data',
        ],
      });
    }

    return bottlenecks;
  }

  private async getCpuUsage(): Promise<number> {
    // Query Prometheus
    const result = await this.prometheus.query(
      'sum(rate(process_cpu_seconds_total{service="order-service"}[5m]))'
    );
    return parseFloat(result.value);
  }

  private async getMemoryUsage(): Promise<number> {
    const used = await this.prometheus.query(
      'process_resident_memory_bytes{service="order-service"}'
    );
    const limit = await this.prometheus.query(
      'container_spec_memory_limit_bytes{container="order-service"}'
    );
    return parseFloat(used.value) / parseFloat(limit.value);
  }

  private async getDBPoolUtilization(): Promise<number> {
    const result = await this.prometheus.query(
      `
      sum(pg_stat_activity_count{datname="orders", state="active"}) /
      sum(pg_settings_max_connections{datname="orders"})
      `
    );
    return parseFloat(result.value);
  }
}
```

---

## Auto-scaling Configuration

### Kubernetes HPA + KEDA

```yaml
# kubernetes/autoscaling/order-service-hpa.yaml

apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  
  minReplicas: 3
  maxReplicas: 50
  
  metrics:
  # CPU-based scaling
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70  # Scale up ถ้า CPU > 70%
  
  # Memory-based scaling
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  
  # Custom metric: Request rate (ต้องการ custom metrics adapter)
  - type: External
    external:
      metric:
        name: requests_per_second
        selector:
          matchLabels:
            service: order-service
      target:
        type: AverageValue
        averageValue: "100"  # Target 100 RPS per pod
  
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60   # ไม่ scale up บ่อยกว่า 1 นาที
      policies:
      - type: Pods
        value: 5           # เพิ่มได้ครั้งละไม่เกิน 5 pods
        periodSeconds: 60
      - type: Percent
        value: 100         # หรือ double ใน 60 วินาที
        periodSeconds: 60
      selectPolicy: Max    # ใช้ policy ที่ได้ผลมากกว่า
    
    scaleDown:
      stabilizationWindowSeconds: 300  # รอ 5 นาทีก่อน scale down
      policies:
      - type: Pods
        value: 2           # ลดครั้งละไม่เกิน 2 pods
        periodSeconds: 120
```

```yaml
# kubernetes/autoscaling/order-service-keda.yaml
# KEDA (Kubernetes Event-Driven Autoscaling) - scale based on Kafka lag

apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-service-scaledobject
  namespace: production
spec:
  scaleTargetRef:
    name: order-service
  
  minReplicaCount: 2
  maxReplicaCount: 50
  
  triggers:
  # Scale based on Kafka consumer lag
  - type: kafka
    metadata:
      bootstrapServers: kafka.production:9092
      consumerGroup: order-service-consumer-group
      topic: order.events.incoming
      lagThreshold: "100"       # Scale up ถ้า lag > 100 messages per partition
      activationLagThreshold: "10"
  
  # Scale based on Prometheus metrics
  - type: prometheus
    metadata:
      serverAddress: http://prometheus.monitoring:9090
      metricName: http_requests_in_flight
      threshold: "50"
      query: sum(http_requests_in_flight{service="order-service"})
  
  advanced:
    horizontalPodAutoscalerConfig:
      behavior:
        scaleDown:
          stabilizationWindowSeconds: 300
```

---

## Cost Estimation

### AWS Cost Calculator

```typescript
// tools/capacity/src/cost-estimator.ts

interface AWSRegionPricing {
  ec2PerVCPUHour: number;    // USD per vCPU per hour
  ec2PerGBMemHour: number;   // USD per GB RAM per hour
  eksPer10kNodes: number;    // EKS control plane cost per 10 node hours
  rdsStoragePerGBMonth: number;
  rdsPer2vcpuHour: number;
  transferPerGB: number;     // Data transfer out cost
}

// Asia Pacific (Singapore) - ap-southeast-1 pricing (approximate)
const AP_SE_1_PRICING: AWSRegionPricing = {
  ec2PerVCPUHour: 0.048,    // Fargate, 2 vCPU = ~$0.096/hr
  ec2PerGBMemHour: 0.0052,  // Fargate, 1GB = ~$0.0052/hr
  eksPer10kNodes: 0.10,     // EKS cluster fee + node time
  rdsStoragePerGBMonth: 0.138,
  rdsPer2vcpuHour: 0.109,   // db.t3.medium
  transferPerGB: 0.08,
};

export class CostEstimator {
  estimateMonthlyInfrastructureCost(
    config: InfrastructureConfig,
    pricing: AWSRegionPricing = AP_SE_1_PRICING,
  ): CostBreakdown {
    const HOURS_PER_MONTH = 730;

    // EKS Compute (Fargate)
    const computeCost = config.services.reduce((total, service) => {
      const cpuCost = service.cpuVCPU * service.replicas * 
        pricing.ec2PerVCPUHour * HOURS_PER_MONTH;
      const memoryCost = service.memoryGB * service.replicas * 
        pricing.ec2PerGBMemHour * HOURS_PER_MONTH;
      return total + cpuCost + memoryCost;
    }, 0);

    // RDS (Aurora PostgreSQL)
    const dbCost = config.databases.reduce((total, db) => {
      const instanceCost = (db.vCPU / 2) * pricing.rdsPer2vcpuHour * HOURS_PER_MONTH;
      const storageCost = db.storageGB * pricing.rdsStoragePerGBMonth;
      const replicaCost = instanceCost * db.readReplicas * 0.8; // replicas slightly cheaper
      return total + instanceCost + storageCost + replicaCost;
    }, 0);

    // Data Transfer
    const transferCost = config.estimatedMonthlyDataTransferGB * pricing.transferPerGB;

    // Load Balancer (ALB)
    const lbCost = config.loadBalancers * 22.27; // ~$22.27/month per ALB

    const totalCost = computeCost + dbCost + transferCost + lbCost;

    return {
      compute: computeCost,
      database: dbCost,
      dataTransfer: transferCost,
      loadBalancer: lbCost,
      total: totalCost,
      breakdown: {
        computePercent: (computeCost / totalCost) * 100,
        databasePercent: (dbCost / totalCost) * 100,
        transferPercent: (transferCost / totalCost) * 100,
      },
    };
  }
}
```

### Cost Optimization Recommendations

```
Cost Optimization Checklist:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. Right-sizing (ลดต้นทุน 20-40%)
   [ ] วิเคราะห์ CPU/Memory utilization จาก metrics
   [ ] ลด resource limits ที่ over-provisioned
   [ ] ใช้ Vertical Pod Autoscaler (VPA) แนะนำขนาดที่เหมาะสม

2. Spot Instances (ลดต้นทุน 60-90%)
   [ ] Stateless services ที่รับ interruption ได้ → Spot/Preemptible
   [ ] ใช้ spot instances สำหรับ batch workloads
   [ ] Configure proper interruption handling

3. Reserved Capacity (ลดต้นทุน 30-60%)
   [ ] Production workloads ที่ stable → Reserved instances 1-3 ปี
   [ ] Database instances → Reserved DB instances

4. Auto-scaling (ลดต้นทุน 20-30%)
   [ ] Cluster Autoscaler ลด nodes ช่วงกลางคืน
   [ ] HPA ลด pods เมื่อ traffic ต่ำ
   [ ] Schedule-based scaling (scale down นอกเวลาทำการ)

5. Storage Optimization (ลดต้นทุน 15-25%)
   [ ] S3 Intelligent-Tiering สำหรับ infrequent access data
   [ ] EBS gp3 แทน gp2 (ราคาถูกกว่า 20%)
   [ ] Archive logs ไปยัง Glacier หลัง 90 วัน

6. Network Optimization (ลดต้นทุน 10-20%)
   [ ] VPC Endpoints แทน Internet Gateway สำหรับ AWS services
   [ ] CloudFront สำหรับ static content
   [ ] ลด cross-AZ data transfer
```

---

## Capacity Planning Dashboard

```yaml
# grafana/dashboards/capacity-planning.json (abbreviated)

panels:
  - title: "Current vs. Projected Load"
    type: timeseries
    targets:
      - expr: "sum(rate(http_requests_total[5m])) * on() group_left vector(1)"
        legendFormat: "Current RPS"
      - expr: "forecast_request_rate_next_30d"
        legendFormat: "30-day Forecast"
    
  - title: "Resource Headroom"
    type: gauge
    targets:
      - expr: |
          (
            sum(kube_pod_container_resource_limits{resource="cpu",
                container="order-service"}) -
            sum(rate(container_cpu_usage_seconds_total{
                container="order-service"}[5m]))
          ) / 
          sum(kube_pod_container_resource_limits{resource="cpu",
              container="order-service"}) * 100
    thresholds:
      - color: red
        value: 0
      - color: yellow
        value: 20
      - color: green
        value: 40

  - title: "Auto-scaling Events (24h)"
    type: timeseries
    targets:
      - expr: "changes(kube_deployment_spec_replicas{deployment='order-service'}[1h])"
        legendFormat: "Scaling Events"

  - title: "Error Budget Burn Rate"
    type: stat
    targets:
      - expr: |
          (
            sum(rate(http_requests_total{status=~"5..",service="order-service"}[1h])) /
            sum(rate(http_requests_total{service="order-service"}[1h]))
          ) / (1 - 0.999) * 100
        legendFormat: "Error Budget Burn Rate %"
```

---

## Capacity Review Process

```
Monthly Capacity Review Meeting Agenda:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Attendees: Engineering Lead, SRE, Product Manager, Finance

1. Traffic Review (15 min)
   - Current traffic vs. last month
   - Growth rate calculation
   - Upcoming events that will impact load
     (campaigns, holidays, special events)

2. Resource Utilization Review (15 min)
   - Services with > 70% CPU/Memory utilization
   - Database query performance trends
   - Network bandwidth usage

3. Incident Review Impact (10 min)
   - Incidents caused by capacity issues
   - Near-miss situations

4. Forecast (15 min)
   - 30-day projection
   - 90-day projection
   - Upcoming product launches impact

5. Budget Review (10 min)
   - Current cloud spend vs. budget
   - Cost optimization opportunities
   - Projected spend for next quarter

6. Action Items (15 min)
   - Services needing right-sizing
   - Services needing scale-up
   - Cost optimization initiatives
```

---

## สรุป

Capacity Planning ต้องทำอย่างสม่ำเสมอและ data-driven:

| กระบวนการ | ความถี่ | เครื่องมือ |
|-----------|---------|-----------|
| Traffic monitoring | Real-time | Grafana |
| Growth analysis | Weekly | Custom analytics |
| Load forecasting | Monthly | Historical data + regression |
| Stress testing | Per release | k6 |
| Resource review | Monthly | Kubernetes metrics |
| Cost optimization | Quarterly | AWS Cost Explorer |

Formula ที่จำเป็น:
- **Required Replicas** = (Target RPS / Current RPS) × Current Replicas × Safety Margin
- **Concurrent Users** (Little's Law) = Arrival Rate × Response Time
- **Error Budget** = 1 - SLO (e.g., 99.9% SLO = 0.1% error budget)

กุญแจสำคัญ: Plan for 2x current capacity เสมอ และ test กับ load 3x ก่อน production

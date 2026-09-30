# Part 88: Capacity Planning

## บทนำ

Capacity Planning (การวางแผนความสามารถ) คือกระบวนการกำหนดทรัพยากรที่จำเป็นสำหรับระบบในอนาคต บทนี้จะแนะนำวิธีการสร้าง model การคาดการณ์ load การวางแผน database, network, cache และ cost optimization สำหรับระบบ Microservices

---

## 1. Load Modeling and Forecasting

### 1.1 Traffic Pattern Analysis

```typescript
// src/capacity/traffic-analyzer.ts
import { DataSource } from 'typeorm';

interface TrafficMetrics {
  timestamp: Date;
  requestsPerSecond: number;
  p50LatencyMs: number;
  p95LatencyMs: number;
  p99LatencyMs: number;
  errorRate: number;
  activeConnections: number;
  cpuUsagePercent: number;
  memoryUsageMB: number;
}

interface TrafficForecast {
  date: Date;
  forecastedRPS: number;
  confidenceInterval: {
    lower: number;
    upper: number;
  };
  trend: 'increasing' | 'decreasing' | 'stable';
  seasonality: {
    hourOfDay: number;
    dayOfWeek: number;
    monthOfYear: number;
  };
}

export class TrafficAnalyzer {
  constructor(private dataSource: DataSource) {}

  async analyzeTrafficPatterns(
    serviceName: string,
    days: number = 30
  ): Promise<{
    hourlyAverage: number[];    // RPS per hour (0-23)
    weeklyPattern: number[];    // RPS per day (0-6, Mon-Sun)
    peakMultiplier: number;     // Peak vs Average ratio
    growthRate: number;         // Month-over-month growth %
  }> {
    const fromDate = new Date();
    fromDate.setDate(fromDate.getDate() - days);
    
    // Query Prometheus/metrics database
    const metrics = await this.dataSource.query(`
      SELECT 
        date_trunc('hour', timestamp) as hour,
        avg(requests_per_second) as avg_rps,
        max(requests_per_second) as max_rps,
        extract(hour from timestamp) as hour_of_day,
        extract(dow from timestamp) as day_of_week
      FROM service_metrics
      WHERE service_name = $1
        AND timestamp >= $2
      GROUP BY 1, 4, 5
      ORDER BY 1
    `, [serviceName, fromDate]);
    
    // Calculate hourly averages (0-23)
    const hourlyAverage = new Array(24).fill(0);
    const hourlyCounts = new Array(24).fill(0);
    
    for (const row of metrics) {
      const hour = parseInt(row.hour_of_day);
      hourlyAverage[hour] += parseFloat(row.avg_rps);
      hourlyCounts[hour]++;
    }
    
    for (let i = 0; i < 24; i++) {
      if (hourlyCounts[i] > 0) {
        hourlyAverage[i] /= hourlyCounts[i];
      }
    }
    
    // Weekly pattern (0=Sun, 1=Mon, ..., 6=Sat)
    const weeklyPattern = new Array(7).fill(0);
    const weeklyCounts = new Array(7).fill(0);
    
    for (const row of metrics) {
      const dow = parseInt(row.day_of_week);
      weeklyPattern[dow] += parseFloat(row.avg_rps);
      weeklyCounts[dow]++;
    }
    
    for (let i = 0; i < 7; i++) {
      if (weeklyCounts[i] > 0) {
        weeklyPattern[i] /= weeklyCounts[i];
      }
    }
    
    const avgRPS = hourlyAverage.reduce((a, b) => a + b) / 24;
    const maxRPS = Math.max(...metrics.map((m: any) => parseFloat(m.max_rps)));
    const peakMultiplier = maxRPS / avgRPS;
    
    // Calculate growth rate (compare first week vs last week)
    const growthRate = this.calculateGrowthRate(metrics);
    
    return { hourlyAverage, weeklyPattern, peakMultiplier, growthRate };
  }

  async forecastTraffic(
    currentRPS: number,
    growthRateMonthly: number,
    months: number
  ): Promise<TrafficForecast[]> {
    const forecasts: TrafficForecast[] = [];
    
    for (let m = 1; m <= months; m++) {
      const forecastedRPS = currentRPS * Math.pow(1 + growthRateMonthly / 100, m);
      
      // 95% confidence interval (using simplified model)
      const uncertainty = 0.15 * m; // uncertainty grows with time
      
      const date = new Date();
      date.setMonth(date.getMonth() + m);
      
      forecasts.push({
        date,
        forecastedRPS,
        confidenceInterval: {
          lower: forecastedRPS * (1 - uncertainty),
          upper: forecastedRPS * (1 + uncertainty),
        },
        trend: growthRateMonthly > 2 ? 'increasing' : 
               growthRateMonthly < -2 ? 'decreasing' : 'stable',
        seasonality: {
          hourOfDay: this.getPeakHour(),
          dayOfWeek: 1, // Monday typically highest
          monthOfYear: 11, // December peak for e-commerce
        },
      });
    }
    
    return forecasts;
  }

  private calculateGrowthRate(metrics: any[]): number {
    if (metrics.length < 14) return 0;
    
    const firstWeek = metrics.slice(0, 7);
    const lastWeek = metrics.slice(-7);
    
    const firstAvg = firstWeek.reduce((sum: number, m: any) => 
      sum + parseFloat(m.avg_rps), 0) / firstWeek.length;
    const lastAvg = lastWeek.reduce((sum: number, m: any) => 
      sum + parseFloat(m.avg_rps), 0) / lastWeek.length;
    
    return ((lastAvg - firstAvg) / firstAvg) * 100;
  }

  private getPeakHour(): number {
    return 14; // 2 PM typically
  }
}
```

### 1.2 Load Testing สำหรับ Capacity Estimation

```typescript
// src/capacity/load-test.ts
// ใช้ k6 script

// k6 load test script
export default function k6Script() {
  return `
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Trend, Counter } from 'k6/metrics';

const errorRate = new Rate('error_rate');
const responseTime = new Trend('response_time_ms');
const requestCount = new Counter('request_count');

export const options = {
  stages: [
    { duration: '2m', target: 100 },    // Ramp up to 100 users
    { duration: '5m', target: 100 },    // Steady state
    { duration: '2m', target: 500 },    // Ramp up to 500 users
    { duration: '5m', target: 500 },    // Steady state
    { duration: '2m', target: 1000 },   // Ramp up to 1000 users
    { duration: '5m', target: 1000 },   // Steady state at peak
    { duration: '2m', target: 0 },      // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(99)<500'],   // 99% ต้องเร็วกว่า 500ms
    http_req_failed: ['rate<0.01'],     // Error rate < 1%
  },
};

const BASE_URL = __ENV.BASE_URL || 'http://localhost:3000';

export default function () {
  const res = http.get(\`\${BASE_URL}/api/v1/users\`, {
    headers: {
      'Authorization': 'Bearer test-token',
      'X-Correlation-ID': \`k6-\${__VU}-\${__ITER}\`,
    },
    timeout: '10s',
  });
  
  check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
  });
  
  errorRate.add(res.status !== 200);
  responseTime.add(res.timings.duration);
  requestCount.add(1);
  
  sleep(0.1); // 100ms between requests
}
  `;
}
```

---

## 2. Resource Utilization Baselines

```typescript
// src/capacity/resource-baseline.ts
interface ResourceBaseline {
  service: string;
  environment: string;
  measuredAt: Date;
  cpu: {
    idle: number;
    p50LoadRPS: number;    // CPU% ที่ 50% of peak RPS
    p100LoadRPS: number;   // CPU% ที่ 100% of peak RPS
    ceiling: number;       // Maximum RPS ก่อน CPU > 80%
  };
  memory: {
    baselineMB: number;    // ใช้ตอน idle
    perRequestMB: number;  // Memory เพิ่มต่อ request
    maxMB: number;         // Pod memory limit
    recommendedMB: number; // 70% ของ max
  };
  connections: {
    dbPoolSize: number;
    redisPoolSize: number;
    maxConcurrent: number;
  };
  network: {
    avgRequestBytes: number;
    avgResponseBytes: number;
    peakMbps: number;
  };
}

export class ResourceBaselineCalculator {
  async calculateBaseline(
    serviceName: string,
    peakRPS: number
  ): Promise<ResourceBaseline> {
    // Query metrics from Prometheus
    const cpuAt50 = await this.getMetric(serviceName, 'cpu', peakRPS * 0.5);
    const cpuAt100 = await this.getMetric(serviceName, 'cpu', peakRPS);
    
    // Calculate linear regression for CPU vs RPS
    const cpuPerRPS = (cpuAt100 - cpuAt50) / (peakRPS * 0.5);
    const cpuCeiling = (80 - cpuAt50) / cpuPerRPS + peakRPS * 0.5;
    
    return {
      service: serviceName,
      environment: process.env.NODE_ENV || 'production',
      measuredAt: new Date(),
      cpu: {
        idle: 2.5,          // 2.5% baseline CPU
        p50LoadRPS: cpuAt50,
        p100LoadRPS: cpuAt100,
        ceiling: cpuCeiling,
      },
      memory: {
        baselineMB: 256,    // 256MB baseline
        perRequestMB: 0.05, // 50KB per request
        maxMB: 1024,        // 1GB limit
        recommendedMB: 716, // 70% of max
      },
      connections: {
        dbPoolSize: Math.ceil(peakRPS / 10), // 1 DB conn per 10 RPS
        redisPoolSize: Math.ceil(peakRPS / 50),
        maxConcurrent: peakRPS * 2, // Allow 2x burst
      },
      network: {
        avgRequestBytes: 500,
        avgResponseBytes: 2000,
        peakMbps: (peakRPS * (500 + 2000) * 8) / 1e6,
      },
    };
  }

  calculateRequiredInstances(
    baseline: ResourceBaseline,
    forecastedRPS: number,
    safetyFactor: number = 1.3
  ): {
    minInstances: number;
    recommendedInstances: number;
    maxInstances: number;
    reasoning: string;
  } {
    const effectiveRPS = forecastedRPS * safetyFactor;
    
    // Based on CPU ceiling
    const instancesByCPU = Math.ceil(effectiveRPS / baseline.cpu.ceiling);
    
    // Based on memory (assume 1 request in flight per ms = RPS/1000 concurrent)
    const concurrentRequests = forecastedRPS / 1000;
    const memoryNeeded = baseline.memory.baselineMB + 
                        concurrentRequests * baseline.memory.perRequestMB;
    const instancesByMemory = Math.ceil(memoryNeeded / baseline.memory.maxMB);
    
    const recommended = Math.max(instancesByCPU, instancesByMemory, 2);
    
    return {
      minInstances: recommended,
      recommendedInstances: recommended + 1, // n+1 for zero-downtime deploys
      maxInstances: recommended * 3, // Auto-scale ceiling
      reasoning: `CPU ceiling: ${Math.round(baseline.cpu.ceiling)} RPS/instance, ` +
                `needs ${instancesByCPU} instances. ` +
                `Memory: ${Math.round(memoryNeeded)}MB needed, ` +
                `needs ${instancesByMemory} instances.`,
    };
  }

  private async getMetric(service: string, metric: string, rps: number): Promise<number> {
    // Mock: ในความเป็นจริงจะ query Prometheus
    return metric === 'cpu' ? rps * 0.08 : rps * 50 / 1024;
  }
}
```

---

## 3. Database Capacity Planning

```typescript
// src/capacity/database-planner.ts
interface DatabaseCapacityPlan {
  service: string;
  currentMetrics: {
    sizeGB: number;
    rowCount: number;
    queryRPS: number;
    avgQueryMs: number;
    connectionPoolSize: number;
    activeConnections: number;
  };
  projections: {
    months3: DatabaseProjection;
    months6: DatabaseProjection;
    months12: DatabaseProjection;
  };
  recommendations: string[];
}

interface DatabaseProjection {
  sizeGB: number;
  rowCount: number;
  queryRPS: number;
  requiredConnections: number;
  storageAlertDate: Date | null;
  actionRequired: boolean;
}

export class DatabaseCapacityPlanner {
  async planCapacity(
    dbName: string,
    growthRateMonthly: number = 15  // 15% per month
  ): Promise<DatabaseCapacityPlan> {
    const current = await this.getCurrentMetrics(dbName);
    
    const project = (months: number): DatabaseProjection => {
      const growthFactor = Math.pow(1 + growthRateMonthly / 100, months);
      const projectedSize = current.sizeGB * growthFactor;
      const projectedRows = current.rowCount * growthFactor;
      const projectedRPS = current.queryRPS * growthFactor;
      const projectedConnections = Math.ceil(projectedRPS / 10);
      
      // คำนวณว่าจะเต็ม disk เมื่อไหร่
      const diskCapacityGB = 500; // Assume 500GB disk
      const monthsUntilFull = Math.log(diskCapacityGB / current.sizeGB) / 
                              Math.log(1 + growthRateMonthly / 100);
      
      const storageAlertDate = new Date();
      storageAlertDate.setMonth(
        storageAlertDate.getMonth() + monthsUntilFull
      );
      
      return {
        sizeGB: Math.round(projectedSize * 100) / 100,
        rowCount: Math.round(projectedRows),
        queryRPS: Math.round(projectedRPS),
        requiredConnections: projectedConnections,
        storageAlertDate: monthsUntilFull <= 18 ? storageAlertDate : null,
        actionRequired: projectedConnections > 100 || projectedSize > 200,
      };
    };
    
    const recommendations = this.generateRecommendations(current, growthRateMonthly);
    
    return {
      service: dbName,
      currentMetrics: current,
      projections: {
        months3: project(3),
        months6: project(6),
        months12: project(12),
      },
      recommendations,
    };
  }

  private generateRecommendations(
    current: DatabaseCapacityPlan['currentMetrics'],
    growthRate: number
  ): string[] {
    const recommendations: string[] = [];
    
    // Connection pool
    if (current.activeConnections > current.connectionPoolSize * 0.8) {
      recommendations.push(
        `Connection pool at ${Math.round(current.activeConnections / current.connectionPoolSize * 100)}% capacity. ` +
        `Consider increasing pool size or adding read replicas.`
      );
    }
    
    // Query performance
    if (current.avgQueryMs > 100) {
      recommendations.push(
        `Average query time ${current.avgQueryMs}ms exceeds 100ms SLA. ` +
        `Review query plans and add indexes.`
      );
    }
    
    // Growth rate
    if (growthRate > 20) {
      recommendations.push(
        `High growth rate (${growthRate}%/month). Consider:
        1. Partitioning large tables by date
        2. Archiving data older than 6 months
        3. Upgrading to larger instance class in 3-6 months`
      );
    }
    
    // Size
    if (current.sizeGB > 100) {
      recommendations.push(
        `Database size ${current.sizeGB}GB. Consider:
        1. Enable table partitioning
        2. Review and implement data archival strategy
        3. Use Read Replicas for read-heavy workloads`
      );
    }
    
    return recommendations;
  }

  private async getCurrentMetrics(dbName: string): Promise<DatabaseCapacityPlan['currentMetrics']> {
    // Query from monitoring system
    return {
      sizeGB: 45,
      rowCount: 10000000,
      queryRPS: 500,
      avgQueryMs: 35,
      connectionPoolSize: 100,
      activeConnections: 75,
    };
  }
}
```

---

## 4. Kubernetes Resource Right-Sizing

```typescript
// src/capacity/k8s-rightsizer.ts
interface K8sResourceSpec {
  requests: { cpu: string; memory: string };
  limits: { cpu: string; memory: string };
}

interface RightSizingRecommendation {
  service: string;
  current: K8sResourceSpec;
  recommended: K8sResourceSpec;
  potentialSavings: {
    cpu: string;
    memory: string;
    estimatedCostSavingPercent: number;
  };
  reasoning: string;
}

export class KubernetesRightSizer {
  async analyzeAndRecommend(
    namespace: string,
    deploymentName: string,
    lookbackDays: number = 14
  ): Promise<RightSizingRecommendation> {
    // Query VPA recommendations from Kubernetes
    const vpaRecommendation = await this.getVPARecommendation(
      namespace, 
      deploymentName
    );
    
    // Query actual usage from Prometheus
    const actualUsage = await this.getActualUsage(
      namespace, 
      deploymentName, 
      lookbackDays
    );
    
    // Calculate recommendations with headroom
    const HEADROOM_FACTOR = 1.2; // 20% headroom
    
    const recommendedCpuMilli = Math.ceil(
      actualUsage.p99CpuMilli * HEADROOM_FACTOR
    );
    const recommendedMemoryMB = Math.ceil(
      actualUsage.p99MemoryMB * HEADROOM_FACTOR
    );
    
    const current = await this.getCurrentSpec(namespace, deploymentName);
    
    return {
      service: deploymentName,
      current,
      recommended: {
        requests: {
          cpu: `${Math.ceil(recommendedCpuMilli * 0.5)}m`, // Request = 50% of limit
          memory: `${Math.ceil(recommendedMemoryMB * 0.7)}Mi`,
        },
        limits: {
          cpu: `${recommendedCpuMilli}m`,
          memory: `${recommendedMemoryMB}Mi`,
        },
      },
      potentialSavings: this.calculateSavings(current, {
        requests: {
          cpu: `${Math.ceil(recommendedCpuMilli * 0.5)}m`,
          memory: `${Math.ceil(recommendedMemoryMB * 0.7)}Mi`,
        },
        limits: {
          cpu: `${recommendedCpuMilli}m`,
          memory: `${recommendedMemoryMB}Mi`,
        },
      }),
      reasoning: `Based on ${lookbackDays} days: P99 CPU=${actualUsage.p99CpuMilli}m, P99 Memory=${actualUsage.p99MemoryMB}MB`,
    };
  }

  private async getActualUsage(
    namespace: string,
    deployment: string,
    days: number
  ): Promise<{ p99CpuMilli: number; p99MemoryMB: number }> {
    // Prometheus queries
    const cpuQuery = `
      quantile_over_time(0.99, 
        rate(container_cpu_usage_seconds_total{
          namespace="${namespace}",
          pod=~"${deployment}-.*"
        }[5m])[${days}d:5m]
      ) * 1000
    `;
    
    const memQuery = `
      quantile_over_time(0.99,
        container_memory_working_set_bytes{
          namespace="${namespace}",
          pod=~"${deployment}-.*"
        }[${days}d:5m]
      ) / (1024 * 1024)
    `;
    
    // In reality, query Prometheus API
    return { p99CpuMilli: 250, p99MemoryMB: 384 };
  }

  private calculateSavings(
    current: K8sResourceSpec,
    recommended: K8sResourceSpec
  ): RightSizingRecommendation['potentialSavings'] {
    const currentCpuMilli = parseInt(current.requests.cpu);
    const recommendedCpuMilli = parseInt(recommended.requests.cpu);
    const savingPercent = ((currentCpuMilli - recommendedCpuMilli) / currentCpuMilli) * 100;
    
    return {
      cpu: `${currentCpuMilli - recommendedCpuMilli}m`,
      memory: `${parseInt(current.requests.memory) - parseInt(recommended.requests.memory)}Mi`,
      estimatedCostSavingPercent: Math.max(0, savingPercent),
    };
  }

  private async getCurrentSpec(namespace: string, deployment: string): Promise<K8sResourceSpec> {
    return {
      requests: { cpu: '500m', memory: '512Mi' },
      limits: { cpu: '1000m', memory: '1Gi' },
    };
  }

  private async getVPARecommendation(namespace: string, deployment: string): Promise<any> {
    return null;
  }
}
```

---

## 5. Cache Sizing Strategies

```typescript
// src/capacity/cache-sizer.ts
interface CacheCapacityPlan {
  service: string;
  currentStats: {
    hitRate: number;
    memoryUsedMB: number;
    keyCount: number;
    avgValueSizeBytes: number;
    evictionRate: number;
  };
  recommendations: {
    targetHitRate: number;
    recommendedMemoryMB: number;
    ttlSettings: Record<string, number>;
    evictionPolicy: string;
  };
}

export class CacheSizer {
  async planCacheCapacity(
    serviceName: string,
    rps: number
  ): Promise<CacheCapacityPlan> {
    const currentStats = await this.getCurrentStats(serviceName);
    
    // Target hit rate: 80%+
    const targetHitRate = 0.85;
    
    // Calculate cache size needed for target hit rate
    // ใช้ working set estimation
    const uniqueKeysPerHour = rps * 3600 * (1 - targetHitRate);
    const avgTTLHours = 1;
    const workingSetKeys = uniqueKeysPerHour * avgTTLHours;
    const recommendedMemoryMB = Math.ceil(
      (workingSetKeys * currentStats.avgValueSizeBytes) / 1024 / 1024 * 1.5
    );
    
    return {
      service: serviceName,
      currentStats,
      recommendations: {
        targetHitRate,
        recommendedMemoryMB,
        ttlSettings: {
          user_profiles: 3600,       // 1 hour
          product_catalog: 86400,    // 24 hours
          session_data: 1800,        // 30 minutes
          config: 3600,              // 1 hour
          search_results: 300,       // 5 minutes
        },
        evictionPolicy: currentStats.hitRate < 0.7 ? 
          'allkeys-lru' : 'volatile-lru',
      },
    };
  }

  private async getCurrentStats(serviceName: string): Promise<CacheCapacityPlan['currentStats']> {
    return {
      hitRate: 0.72,
      memoryUsedMB: 256,
      keyCount: 500000,
      avgValueSizeBytes: 512,
      evictionRate: 0.02,
    };
  }
}
```

---

## 6. Cost Optimization Strategies

```typescript
// src/capacity/cost-optimizer.ts
interface CostModel {
  onDemand: number;      // $/hour
  reserved1Year: number; // $/hour (equivalent)
  reserved3Year: number; // $/hour (equivalent)
  spot: number;          // $/hour (average)
  savingsFor1Year: number;  // % savings vs on-demand
  savingsFor3Year: number;
  spotSavings: number;
}

const EC2_PRICING: Record<string, CostModel> = {
  't3.small': {
    onDemand: 0.0208,
    reserved1Year: 0.0124,
    reserved3Year: 0.0083,
    spot: 0.0062,
    savingsFor1Year: 40,
    savingsFor3Year: 60,
    spotSavings: 70,
  },
  't3.medium': {
    onDemand: 0.0416,
    reserved1Year: 0.0249,
    reserved3Year: 0.0166,
    spot: 0.0125,
    savingsFor1Year: 40,
    savingsFor3Year: 60,
    spotSavings: 70,
  },
  'c5.xlarge': {
    onDemand: 0.17,
    reserved1Year: 0.102,
    reserved3Year: 0.068,
    spot: 0.051,
    savingsFor1Year: 40,
    savingsFor3Year: 60,
    spotSavings: 70,
  },
};

export class CostOptimizer {
  analyzePurchaseOptions(
    instanceType: string,
    hoursPerMonth: number,
    baselineInstances: number
  ): {
    currentCost: number;
    optimizedCost: number;
    savings: number;
    strategy: string;
    recommendations: string[];
  } {
    const pricing = EC2_PRICING[instanceType];
    if (!pricing) throw new Error(`Unknown instance type: ${instanceType}`);
    
    const currentCost = pricing.onDemand * hoursPerMonth * baselineInstances;
    
    // Strategy: Mixed purchase for cost optimization
    // - Baseline (80%): Reserved 1-year for predictable load
    // - Burst (20%): Spot instances for scale-out
    const reservedInstances = Math.ceil(baselineInstances * 0.8);
    const spotInstances = baselineInstances - reservedInstances;
    
    const optimizedCost = 
      (pricing.reserved1Year * hoursPerMonth * reservedInstances) +
      (pricing.spot * hoursPerMonth * spotInstances);
    
    const savings = currentCost - optimizedCost;
    const savingsPercent = (savings / currentCost) * 100;
    
    return {
      currentCost: Math.round(currentCost * 100) / 100,
      optimizedCost: Math.round(optimizedCost * 100) / 100,
      savings: Math.round(savings * 100) / 100,
      strategy: `${reservedInstances}x Reserved 1-year + ${spotInstances}x Spot`,
      recommendations: [
        `Save $${Math.round(savings)}/month (${Math.round(savingsPercent)}%) by mixing Reserved and Spot instances`,
        `Use Reserved Instances for ${reservedInstances} baseline nodes`,
        `Use Spot Instances with Karpenter for burst capacity`,
        `Enable Savings Plans for additional flexibility`,
        `Review instance sizing monthly using AWS Cost Anomaly Detection`,
      ],
    };
  }

  // Kubernetes cost optimization
  generateKarpenterNodePool(): string {
    return `
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64", "arm64"]
        - key: kubernetes.io/os
          operator: In
          values: ["linux"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: ["c", "m", "r"]
        - key: karpenter.k8s.aws/instance-generation
          operator: Gt
          values: ["2"]
      nodeClassRef:
        name: default
      # Prefer spot instances
      expireAfter: 720h
  limits:
    cpu: "1000"
    memory: 1000Gi
  disruption:
    consolidationPolicy: WhenUnderutilized
    consolidateAfter: 30s
---
apiVersion: karpenter.k8s.aws/v1beta1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiFamily: AL2
  role: "KarpenterNodeRole"
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: "my-cluster"
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: "my-cluster"
  # Cost optimization: use ARM (Graviton) instances
  instanceStorePolicy: RAID0
`;
  }
}
```

---

## 7. Multi-Region Capacity Planning

```typescript
// src/capacity/multi-region.ts
interface RegionCapacityPlan {
  region: string;
  userBase: number;          // Users in this region
  trafficPercent: number;    // % of total traffic
  latencyRequirement: number; // ms SLA
  currentInstances: number;
  recommendedInstances: number;
  failoverCapacity: number;   // Extra capacity for regional failover
  costPerMonth: number;
}

export class MultiRegionCapacityPlanner {
  planMultiRegion(
    totalUsers: number,
    totalRPS: number,
    regions: Array<{
      name: string;
      userPercent: number;
      latencySLA: number;
    }>
  ): RegionCapacityPlan[] {
    const plans: RegionCapacityPlan[] = [];
    
    for (const region of regions) {
      const regionUsers = Math.ceil(totalUsers * region.userPercent / 100);
      const regionRPS = Math.ceil(totalRPS * region.userPercent / 100);
      
      // Add 50% extra capacity for regional failover
      const failoverCapacity = Math.ceil(regionRPS * 0.5);
      
      // Calculate instances needed (assume 100 RPS per instance)
      const instancesForNormal = Math.ceil(regionRPS / 100);
      const instancesWithFailover = Math.ceil((regionRPS + failoverCapacity) / 100);
      
      // Calculate cost (using c5.large at $0.085/hour)
      const costPerMonth = instancesWithFailover * 0.085 * 24 * 30;
      
      plans.push({
        region: region.name,
        userBase: regionUsers,
        trafficPercent: region.userPercent,
        latencyRequirement: region.latencySLA,
        currentInstances: instancesForNormal,
        recommendedInstances: instancesWithFailover,
        failoverCapacity: instancesWithFailover - instancesForNormal,
        costPerMonth: Math.round(costPerMonth * 100) / 100,
      });
    }
    
    return plans;
  }

  generateTerraformConfig(plans: RegionCapacityPlan[]): string {
    return plans.map(plan => `
# Region: ${plan.region}
module "eks_${plan.region.replace('-', '_')}" {
  source = "./modules/eks"
  
  region             = "${plan.region}"
  cluster_name       = "microservices-${plan.region}"
  
  node_groups = {
    application = {
      desired_size = ${plan.recommendedInstances}
      min_size     = ${Math.max(2, Math.ceil(plan.currentInstances * 0.5))}
      max_size     = ${plan.recommendedInstances * 3}
      
      instance_types = ["c5.large", "c5a.large", "c5n.large"]
      capacity_type  = "SPOT"
    }
    
    system = {
      desired_size = 2
      min_size     = 2
      max_size     = 4
      instance_types = ["t3.medium"]
      capacity_type  = "ON_DEMAND"
    }
  }
}
`).join('\n');
  }
}
```

---

## สรุปท้ายบท

| หัวข้อ | วิธีการ | เครื่องมือ |
|--------|---------|-----------|
| Traffic Forecasting | Linear regression + seasonality | Prometheus, Grafana |
| Load Testing | Ramp testing + soak testing | k6, Artillery |
| Resource Baseline | P95/P99 CPU & Memory | VPA, Prometheus |
| DB Capacity | Growth rate projection | pg_stat_statements |
| K8s Right-Sizing | VPA recommendations | Kubernetes VPA |
| Cache Sizing | Hit rate optimization | Redis INFO |
| Cost Optimization | Reserved + Spot mix | AWS Cost Explorer |
| Multi-Region | Traffic distribution | Route53, Global Accelerator |

### Capacity Planning Calendar

- **Weekly**: ตรวจสอบ resource utilization trends
- **Monthly**: Review growth forecasts, update capacity plans
- **Quarterly**: Review cost optimization, Reserved Instance renewals
- **Annually**: Major capacity review, infrastructure refresh

### Rules of Thumb

1. **CPU**: อย่าให้เกิน 70% sustained, 80% peak
2. **Memory**: อย่าให้เกิน 80% (ป้องกัน OOMKilled)
3. **DB Connections**: อย่าให้เกิน 80% ของ pool size
4. **Cache Hit Rate**: ควรอยู่ที่ 80%+ ถ้าต่ำกว่าให้ขยาย cache
5. **Reserved Instances**: 70-80% ของ baseline เป็น Reserved, ที่เหลือ Spot

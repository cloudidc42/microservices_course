# Part 88: Capacity Planning

## บทนำ

Capacity Planning คือกระบวนการประเมินและวางแผนทรัพยากรที่จำเป็นสำหรับระบบเพื่อให้รองรับ load ในปัจจุบันและอนาคต บทนี้จะครอบคลุมเครื่องมือและ TypeScript classes สำหรับ capacity planning ใน Microservices

---

## 1. Load Modeling

### 1.1 TypeScript LoadModel Class

```typescript
// capacity/load-model.ts
interface ServiceMetrics {
  requestsPerSecond: number;
  avgResponseTimeMs: number;
  p99ResponseTimeMs: number;
  errorRate: number;       // เปอร์เซ็นต์
  cpuUsagePercent: number;
  memoryUsageMB: number;
  networkInMBps: number;
  networkOutMBps: number;
}

interface ResourceRequirements {
  cpuCores: number;
  memoryGB: number;
  storageGB: number;
  networkMBps: number;
  replicaCount: number;
}

interface LoadScenario {
  name: string;
  requestsPerSecond: number;
  concurrentUsers: number;
  avgSessionDurationSeconds: number;
  peakMultiplier: number;
}

class LoadModel {
  private baselineMetrics: ServiceMetrics;
  private safetyFactor: number;

  constructor(
    baselineMetrics: ServiceMetrics,
    safetyFactor: number = 1.3 // 30% buffer
  ) {
    this.baselineMetrics = baselineMetrics;
    this.safetyFactor = safetyFactor;
  }

  // คำนวณ resource requirements จาก RPS ที่คาดการณ์
  calculateResourcesForRPS(targetRPS: number): ResourceRequirements {
    const scaleFactor = targetRPS / this.baselineMetrics.requestsPerSecond;

    // คำนวณ CPU
    const rawCpuCores = (this.baselineMetrics.cpuUsagePercent / 100) * scaleFactor;
    const cpuCores = Math.ceil(rawCpuCores * this.safetyFactor * 100) / 100;

    // คำนวณ Memory
    const rawMemoryGB = (this.baselineMetrics.memoryUsageMB / 1024) * scaleFactor;
    const memoryGB = Math.ceil(rawMemoryGB * this.safetyFactor * 10) / 10;

    // คำนวณ Network
    const networkInMBps = this.baselineMetrics.networkInMBps * scaleFactor * this.safetyFactor;
    const networkOutMBps = this.baselineMetrics.networkOutMBps * scaleFactor * this.safetyFactor;
    const networkMBps = Math.ceil(networkInMBps + networkOutMBps);

    // คำนวณจำนวน replicas ตาม CPU (สมมติว่าแต่ละ instance ใช้ได้ 2 cores)
    const maxCoresPerInstance = 2;
    const replicaCount = Math.max(
      2, // minimum 2 replicas สำหรับ HA
      Math.ceil(cpuCores / maxCoresPerInstance)
    );

    // Storage estimate (ขึ้นอยู่กับ request size และ logging)
    const avgRequestSizeKB = 10;
    const logRetentionDays = 30;
    const storageGB = Math.ceil(
      (targetRPS * avgRequestSizeKB * 86400 * logRetentionDays) / (1024 * 1024) * this.safetyFactor
    );

    return {
      cpuCores: cpuCores / replicaCount,
      memoryGB: memoryGB / replicaCount,
      storageGB,
      networkMBps,
      replicaCount,
    };
  }

  // คำนวณ RPS จากจำนวน concurrent users
  calculateRPSFromUsers(scenario: LoadScenario): number {
    // Little's Law: L = λ × W
    // L = concurrent users, λ = arrival rate (RPS), W = avg session duration
    const thinkTimeFactor = 0.7; // สมมติว่า 70% ของเวลาเป็น active requests
    const rps =
      (scenario.concurrentUsers * thinkTimeFactor) /
      (this.baselineMetrics.avgResponseTimeMs / 1000);

    return rps * scenario.peakMultiplier;
  }

  // สร้าง capacity plan สำหรับหลาย scenarios
  generateCapacityPlan(scenarios: LoadScenario[]): Array<{
    scenario: LoadScenario;
    estimatedRPS: number;
    resources: ResourceRequirements;
    monthlyAWSCost: number;
  }> {
    return scenarios.map((scenario) => {
      const estimatedRPS = this.calculateRPSFromUsers(scenario);
      const resources = this.calculateResourcesForRPS(estimatedRPS);
      const monthlyAWSCost = this.estimateAWSCost(resources);

      return {
        scenario,
        estimatedRPS: Math.round(estimatedRPS),
        resources,
        monthlyAWSCost,
      };
    });
  }

  private estimateAWSCost(resources: ResourceRequirements): number {
    // EC2 t3.medium: $0.0416/hour = ~$30/month
    // อ้างอิง: https://aws.amazon.com/ec2/pricing/
    const ec2HourlyRate = 0.0416;
    const hoursPerMonth = 730;

    // EBS: $0.1/GB/month
    const ebsRate = 0.1;

    // Data transfer: $0.09/GB
    const dataTransferRate = 0.09;

    const ec2Cost =
      resources.replicaCount *
      Math.ceil(resources.cpuCores / 2) * // จำนวน instance ต่อ replica
      ec2HourlyRate *
      hoursPerMonth;

    const storageCost = resources.storageGB * ebsRate;

    const networkCost =
      resources.networkMBps * 3600 * hoursPerMonth / 1024 * dataTransferRate;

    return Math.round(ec2Cost + storageCost + networkCost);
  }
}

// ตัวอย่างการใช้งาน
async function demonstrateLoadModel() {
  const orderServiceBaseline: ServiceMetrics = {
    requestsPerSecond: 100,
    avgResponseTimeMs: 50,
    p99ResponseTimeMs: 200,
    errorRate: 0.1,
    cpuUsagePercent: 30,
    memoryUsageMB: 512,
    networkInMBps: 5,
    networkOutMBps: 10,
  };

  const model = new LoadModel(orderServiceBaseline, 1.3);

  const scenarios: LoadScenario[] = [
    {
      name: 'Normal Traffic',
      requestsPerSecond: 100,
      concurrentUsers: 500,
      avgSessionDurationSeconds: 300,
      peakMultiplier: 1.0,
    },
    {
      name: 'Flash Sale (10x)',
      requestsPerSecond: 1000,
      concurrentUsers: 5000,
      avgSessionDurationSeconds: 180,
      peakMultiplier: 10.0,
    },
    {
      name: 'Holiday Peak (5x)',
      requestsPerSecond: 500,
      concurrentUsers: 2500,
      avgSessionDurationSeconds: 240,
      peakMultiplier: 5.0,
    },
  ];

  const plan = model.generateCapacityPlan(scenarios);

  console.log('\n=== Capacity Planning Report ===\n');
  for (const item of plan) {
    console.log(`Scenario: ${item.scenario.name}`);
    console.log(`  Estimated RPS: ${item.estimatedRPS}`);
    console.log(`  CPU per replica: ${item.resources.cpuCores} cores`);
    console.log(`  Memory per replica: ${item.resources.memoryGB} GB`);
    console.log(`  Replicas needed: ${item.resources.replicaCount}`);
    console.log(`  Storage: ${item.resources.storageGB} GB`);
    console.log(`  Monthly AWS cost: $${item.monthlyAWSCost}`);
    console.log('');
  }
}

demonstrateLoadModel().catch(console.error);
```

---

## 2. Resource Utilization Baseline Collector

```typescript
// capacity/baseline-collector.ts
import axios from 'axios';

interface PrometheusQueryResult {
  metric: Record<string, string>;
  value: [number, string]; // [timestamp, value]
}

interface ServiceBaseline {
  serviceName: string;
  collectionPeriodDays: number;
  metrics: {
    avgCPUPercent: number;
    maxCPUPercent: number;
    p99CPUPercent: number;
    avgMemoryMB: number;
    maxMemoryMB: number;
    avgRPS: number;
    maxRPS: number;
    p99LatencyMs: number;
    avgLatencyMs: number;
    errorRate: number;
    networkInMBps: number;
    networkOutMBps: number;
  };
  collectedAt: Date;
}

class BaselineCollector {
  constructor(
    private readonly prometheusUrl: string = 'http://prometheus:9090',
    private readonly defaultRange: string = '7d' // 7 days
  ) {}

  private async queryPrometheus(
    query: string,
    range?: string
  ): Promise<PrometheusQueryResult[]> {
    const url = `${this.prometheusUrl}/api/v1/query`;
    const response = await axios.get(url, {
      params: { query, time: Date.now() / 1000 },
      timeout: 30000,
    });

    if (response.data.status !== 'success') {
      throw new Error(`Prometheus query failed: ${response.data.error}`);
    }

    return response.data.data.result;
  }

  private async queryRange(
    query: string,
    start: number,
    end: number,
    step: string = '5m'
  ): Promise<Array<{ time: number; value: number }>> {
    const url = `${this.prometheusUrl}/api/v1/query_range`;
    const response = await axios.get(url, {
      params: { query, start, end, step },
      timeout: 60000,
    });

    if (response.data.status !== 'success') {
      throw new Error(`Prometheus range query failed: ${response.data.error}`);
    }

    const results = response.data.data.result;
    if (results.length === 0) return [];

    return results[0].values.map(([time, value]: [number, string]) => ({
      time: time * 1000,
      value: parseFloat(value),
    }));
  }

  async collectBaseline(
    serviceName: string,
    namespace: string = 'default',
    days: number = 7
  ): Promise<ServiceBaseline> {
    const endTime = Math.floor(Date.now() / 1000);
    const startTime = endTime - days * 86400;

    console.log(`Collecting baseline for ${serviceName} over ${days} days...`);

    // CPU metrics
    const cpuData = await this.queryRange(
      `rate(container_cpu_usage_seconds_total{namespace="${namespace}", container="${serviceName}"}[5m]) * 100`,
      startTime,
      endTime
    );

    const cpuValues = cpuData.map((d) => d.value);
    const avgCPU = cpuValues.reduce((a, b) => a + b, 0) / cpuValues.length;
    const maxCPU = Math.max(...cpuValues);
    const sortedCPU = [...cpuValues].sort((a, b) => a - b);
    const p99CPU = sortedCPU[Math.floor(sortedCPU.length * 0.99)] || 0;

    // Memory metrics
    const memData = await this.queryRange(
      `container_memory_usage_bytes{namespace="${namespace}", container="${serviceName}"} / 1024 / 1024`,
      startTime,
      endTime
    );
    const memValues = memData.map((d) => d.value);
    const avgMemory = memValues.reduce((a, b) => a + b, 0) / memValues.length;
    const maxMemory = Math.max(...memValues);

    // RPS metrics
    const rpsData = await this.queryRange(
      `rate(http_requests_total{service="${serviceName}"}[5m])`,
      startTime,
      endTime
    );
    const rpsValues = rpsData.map((d) => d.value);
    const avgRPS = rpsValues.reduce((a, b) => a + b, 0) / rpsValues.length;
    const maxRPS = Math.max(...rpsValues);

    // Latency metrics
    const latencyP99Result = await this.queryPrometheus(
      `histogram_quantile(0.99, rate(http_request_duration_seconds_bucket{service="${serviceName}"}[${days}d])) * 1000`
    );
    const p99Latency = latencyP99Result.length > 0
      ? parseFloat(latencyP99Result[0].value[1])
      : 0;

    const latencyAvgResult = await this.queryPrometheus(
      `rate(http_request_duration_seconds_sum{service="${serviceName}"}[${days}d]) / rate(http_request_duration_seconds_count{service="${serviceName}"}[${days}d]) * 1000`
    );
    const avgLatency = latencyAvgResult.length > 0
      ? parseFloat(latencyAvgResult[0].value[1])
      : 0;

    // Error rate
    const errorRateResult = await this.queryPrometheus(
      `rate(http_requests_total{service="${serviceName}", status=~"5.."}[${days}d]) / rate(http_requests_total{service="${serviceName}"}[${days}d]) * 100`
    );
    const errorRate = errorRateResult.length > 0
      ? parseFloat(errorRateResult[0].value[1])
      : 0;

    // Network metrics
    const networkInResult = await this.queryPrometheus(
      `rate(container_network_receive_bytes_total{namespace="${namespace}", pod=~"${serviceName}.*"}[${days}d]) / 1024 / 1024`
    );
    const networkIn = networkInResult.length > 0
      ? parseFloat(networkInResult[0].value[1])
      : 0;

    const networkOutResult = await this.queryPrometheus(
      `rate(container_network_transmit_bytes_total{namespace="${namespace}", pod=~"${serviceName}.*"}[${days}d]) / 1024 / 1024`
    );
    const networkOut = networkOutResult.length > 0
      ? parseFloat(networkOutResult[0].value[1])
      : 0;

    return {
      serviceName,
      collectionPeriodDays: days,
      metrics: {
        avgCPUPercent: Math.round(avgCPU * 100) / 100,
        maxCPUPercent: Math.round(maxCPU * 100) / 100,
        p99CPUPercent: Math.round(p99CPU * 100) / 100,
        avgMemoryMB: Math.round(avgMemory),
        maxMemoryMB: Math.round(maxMemory),
        avgRPS: Math.round(avgRPS * 100) / 100,
        maxRPS: Math.round(maxRPS * 100) / 100,
        p99LatencyMs: Math.round(p99Latency),
        avgLatencyMs: Math.round(avgLatency),
        errorRate: Math.round(errorRate * 100) / 100,
        networkInMBps: Math.round(networkIn * 100) / 100,
        networkOutMBps: Math.round(networkOut * 100) / 100,
      },
      collectedAt: new Date(),
    };
  }

  async collectAllServicesBaseline(
    services: string[],
    namespace: string = 'default'
  ): Promise<ServiceBaseline[]> {
    const baselines: ServiceBaseline[] = [];

    for (const service of services) {
      try {
        const baseline = await this.collectBaseline(service, namespace);
        baselines.push(baseline);
        console.log(`✓ Collected baseline for ${service}`);
      } catch (error) {
        console.error(`✗ Failed to collect baseline for ${service}:`, error);
      }
    }

    return baselines;
  }
}
```

---

## 3. Growth Forecasting

### 3.1 Linear Growth Predictor

```typescript
// capacity/growth-predictor.ts

interface DataPoint {
  timestamp: Date;
  value: number;
}

interface ForecastResult {
  timestamp: Date;
  predicted: number;
  lowerBound: number;
  upperBound: number;
  confidenceLevel: number;
}

class LinearGrowthPredictor {
  private slope: number = 0;
  private intercept: number = 0;
  private rSquared: number = 0;

  // Fit linear regression model
  fit(dataPoints: DataPoint[]): void {
    if (dataPoints.length < 2) {
      throw new Error('Need at least 2 data points for linear regression');
    }

    const n = dataPoints.length;
    const baseTime = dataPoints[0].timestamp.getTime();

    // แปลง timestamps เป็น x values (days จาก base)
    const xValues = dataPoints.map(
      (d) => (d.timestamp.getTime() - baseTime) / (1000 * 60 * 60 * 24)
    );
    const yValues = dataPoints.map((d) => d.value);

    // คำนวณ slope และ intercept
    const xMean = xValues.reduce((a, b) => a + b, 0) / n;
    const yMean = yValues.reduce((a, b) => a + b, 0) / n;

    let numerator = 0;
    let denominator = 0;

    for (let i = 0; i < n; i++) {
      numerator += (xValues[i] - xMean) * (yValues[i] - yMean);
      denominator += Math.pow(xValues[i] - xMean, 2);
    }

    this.slope = denominator !== 0 ? numerator / denominator : 0;
    this.intercept = yMean - this.slope * xMean;

    // คำนวณ R-squared
    const yPredicted = xValues.map((x) => this.slope * x + this.intercept);
    const ssTot = yValues.reduce((sum, y) => sum + Math.pow(y - yMean, 2), 0);
    const ssRes = yValues.reduce(
      (sum, y, i) => sum + Math.pow(y - yPredicted[i], 2),
      0
    );
    this.rSquared = ssTot !== 0 ? 1 - ssRes / ssTot : 0;

    console.log(`Linear model fitted: slope=${this.slope.toFixed(4)}, R²=${this.rSquared.toFixed(4)}`);
  }

  predict(
    baseDate: Date,
    futureDates: Date[],
    confidencePercent: number = 95
  ): ForecastResult[] {
    const baseTime = baseDate.getTime();
    const zScore = confidencePercent === 95 ? 1.96 : 2.576; // 95% หรือ 99%

    return futureDates.map((date) => {
      const x = (date.getTime() - baseTime) / (1000 * 60 * 60 * 24);
      const predicted = this.slope * x + this.intercept;
      const standardError = Math.abs(predicted) * (1 - this.rSquared) * 0.1;
      const margin = zScore * standardError;

      return {
        timestamp: date,
        predicted: Math.max(0, Math.round(predicted * 100) / 100),
        lowerBound: Math.max(0, Math.round((predicted - margin) * 100) / 100),
        upperBound: Math.max(0, Math.round((predicted + margin) * 100) / 100),
        confidenceLevel: confidencePercent,
      };
    });
  }

  getGrowthRate(): number {
    // Growth rate เป็น % ต่อเดือน
    return (this.slope * 30 / this.intercept) * 100;
  }

  getModelQuality(): 'excellent' | 'good' | 'fair' | 'poor' {
    if (this.rSquared >= 0.9) return 'excellent';
    if (this.rSquared >= 0.7) return 'good';
    if (this.rSquared >= 0.5) return 'fair';
    return 'poor';
  }
}

// =========================================================
// Seasonal Predictor (Holt-Winters)
// =========================================================

class SeasonalPredictor {
  private alpha: number; // level smoothing
  private beta: number;  // trend smoothing
  private gamma: number; // seasonal smoothing
  private level: number = 0;
  private trend: number = 0;
  private seasonals: number[] = [];
  private seasonalLength: number;

  constructor(
    seasonalLength: number = 7, // 7 = weekly seasonality
    alpha: number = 0.3,
    beta: number = 0.1,
    gamma: number = 0.3
  ) {
    this.seasonalLength = seasonalLength;
    this.alpha = alpha;
    this.beta = beta;
    this.gamma = gamma;
  }

  fit(dataPoints: DataPoint[]): void {
    const values = dataPoints.map((d) => d.value);
    const n = values.length;

    if (n < 2 * this.seasonalLength) {
      throw new Error(`Need at least ${2 * this.seasonalLength} data points`);
    }

    // Initialize level, trend, seasonals
    this.level = values.slice(0, this.seasonalLength).reduce((a, b) => a + b, 0) / this.seasonalLength;
    this.trend =
      (values.slice(this.seasonalLength, 2 * this.seasonalLength).reduce((a, b) => a + b, 0) / this.seasonalLength -
        this.level) /
      this.seasonalLength;

    // Initialize seasonals
    this.seasonals = [];
    for (let i = 0; i < this.seasonalLength; i++) {
      const seasonal =
        values.filter((_, idx) => idx % this.seasonalLength === i).reduce((a, b) => a + b, 0) /
        Math.floor(n / this.seasonalLength);
      this.seasonals.push(seasonal / this.level);
    }

    // Apply Holt-Winters smoothing
    for (let i = 0; i < n; i++) {
      const y = values[i];
      const s = this.seasonals[i % this.seasonalLength];

      const prevLevel = this.level;
      const prevTrend = this.trend;

      this.level = this.alpha * (y / s) + (1 - this.alpha) * (prevLevel + prevTrend);
      this.trend = this.beta * (this.level - prevLevel) + (1 - this.beta) * prevTrend;
      this.seasonals[i % this.seasonalLength] =
        this.gamma * (y / this.level) + (1 - this.gamma) * s;
    }

    console.log(`Seasonal model fitted: level=${this.level.toFixed(2)}, trend=${this.trend.toFixed(4)}`);
  }

  predict(steps: number): ForecastResult[] {
    const forecasts: ForecastResult[] = [];
    let level = this.level;
    let trend = this.trend;

    for (let h = 1; h <= steps; h++) {
      const seasonal = this.seasonals[h % this.seasonalLength];
      const predicted = (level + h * trend) * seasonal;
      const uncertainty = predicted * 0.1 * Math.sqrt(h); // Uncertainty grows with horizon

      forecasts.push({
        timestamp: new Date(Date.now() + h * 24 * 60 * 60 * 1000),
        predicted: Math.max(0, Math.round(predicted * 100) / 100),
        lowerBound: Math.max(0, Math.round((predicted - uncertainty) * 100) / 100),
        upperBound: Math.round((predicted + uncertainty) * 100) / 100,
        confidenceLevel: 95,
      });
    }

    return forecasts;
  }
}
```

---

## 4. Database Capacity Calculator

```typescript
// capacity/database-capacity.ts

interface DatabaseMetrics {
  currentConnections: number;
  maxConnections: number;
  iopsRead: number;
  iopsWrite: number;
  storageSizeGB: number;
  storageGrowthGBPerMonth: number;
  avgQueryTimeMs: number;
  slowQueryCount: number;
  cacheHitRate: number;
}

interface DatabaseCapacityPlan {
  currentLoad: DatabaseMetrics;
  projectedLoad: DatabaseMetrics;
  recommendations: string[];
  upgradeRequired: boolean;
  estimatedUpgradeCost: number;
}

class DatabaseCapacityCalculator {
  // คำนวณจำนวน connections ที่ต้องการ
  calculateConnectionPoolSize(
    numberOfServices: number,
    avgConnectionsPerService: number,
    peakMultiplier: number = 1.5,
    bufferPercent: number = 20
  ): {
    recommended: number;
    max: number;
    pgBouncerSize: number;
  } {
    const base = numberOfServices * avgConnectionsPerService;
    const peak = Math.ceil(base * peakMultiplier);
    const withBuffer = Math.ceil(peak * (1 + bufferPercent / 100));

    return {
      recommended: peak,
      max: withBuffer,
      // PgBouncer pool size = connections ที่เปิดจริงกับ database
      pgBouncerSize: Math.ceil(withBuffer / 10),
    };
  }

  // คำนวณ IOPS ที่ต้องการ
  calculateIOPS(
    requestsPerSecond: number,
    dbQueriesPerRequest: number = 5,
    cacheHitRate: number = 0.8,
    readWriteRatio: number = 0.8 // 80% reads
  ): {
    totalIOPS: number;
    readIOPS: number;
    writeIOPS: number;
    recommendedDiskType: string;
  } {
    const dbRequestsPerSecond = requestsPerSecond * dbQueriesPerRequest;
    const uncachedRequests = dbRequestsPerSecond * (1 - cacheHitRate);
    const totalIOPS = Math.ceil(uncachedRequests);
    const readIOPS = Math.ceil(totalIOPS * readWriteRatio);
    const writeIOPS = Math.ceil(totalIOPS * (1 - readWriteRatio));

    let recommendedDiskType = 'gp2 (SSD)';
    if (totalIOPS > 16000) recommendedDiskType = 'io1 (Provisioned IOPS)';
    else if (totalIOPS > 3000) recommendedDiskType = 'gp3 (SSD)';

    return { totalIOPS, readIOPS, writeIOPS, recommendedDiskType };
  }

  // คำนวณ storage growth
  calculateStorageGrowth(
    currentStorageGB: number,
    dailyGrowthGB: number,
    retentionDays: number,
    compressionRatio: number = 0.3
  ): {
    storageAfter30Days: number;
    storageAfter90Days: number;
    storageAfter365Days: number;
    recommendedStorageGB: number;
    monthsUntilFull: number;
  } {
    const effectiveDailyGrowth = dailyGrowthGB * (1 - compressionRatio);
    const retentionCap = retentionDays * effectiveDailyGrowth;

    const storageAfter30Days = Math.min(
      currentStorageGB + effectiveDailyGrowth * 30,
      retentionCap
    );
    const storageAfter90Days = Math.min(
      currentStorageGB + effectiveDailyGrowth * 90,
      retentionCap
    );
    const storageAfter365Days = Math.min(
      currentStorageGB + effectiveDailyGrowth * 365,
      retentionCap
    );

    // แนะนำขนาด storage พร้อม 20% buffer
    const recommendedStorageGB = Math.ceil(storageAfter365Days * 1.2 / 100) * 100;

    // เดือนจนกว่าจะเต็ม (ถ้าปัจจุบัน 70% เต็ม)
    const capacityLimit = recommendedStorageGB * 0.7;
    const monthsUntilFull =
      currentStorageGB >= capacityLimit
        ? 0
        : (capacityLimit - currentStorageGB) / (effectiveDailyGrowth * 30);

    return {
      storageAfter30Days: Math.round(storageAfter30Days),
      storageAfter90Days: Math.round(storageAfter90Days),
      storageAfter365Days: Math.round(storageAfter365Days),
      recommendedStorageGB,
      monthsUntilFull: Math.round(monthsUntilFull * 10) / 10,
    };
  }

  generateCapacityReport(
    serviceName: string,
    currentMetrics: DatabaseMetrics,
    targetRPS: number,
    growthMonths: number = 12
  ): DatabaseCapacityPlan {
    const scaleFactor = targetRPS / 100; // สมมติ baseline 100 RPS

    // Project metrics
    const projectedMetrics: DatabaseMetrics = {
      currentConnections: Math.ceil(currentMetrics.currentConnections * scaleFactor),
      maxConnections: Math.ceil(currentMetrics.maxConnections * scaleFactor),
      iopsRead: Math.ceil(currentMetrics.iopsRead * scaleFactor),
      iopsWrite: Math.ceil(currentMetrics.iopsWrite * scaleFactor),
      storageSizeGB:
        currentMetrics.storageSizeGB +
        currentMetrics.storageGrowthGBPerMonth * growthMonths,
      storageGrowthGBPerMonth: currentMetrics.storageGrowthGBPerMonth * scaleFactor,
      avgQueryTimeMs: currentMetrics.avgQueryTimeMs,
      slowQueryCount: Math.ceil(currentMetrics.slowQueryCount * scaleFactor),
      cacheHitRate: currentMetrics.cacheHitRate,
    };

    const recommendations: string[] = [];
    let upgradeRequired = false;

    // ตรวจสอบ connections
    if (projectedMetrics.currentConnections > projectedMetrics.maxConnections * 0.8) {
      recommendations.push('เพิ่ม max_connections หรือใช้ PgBouncer connection pooler');
      upgradeRequired = true;
    }

    // ตรวจสอบ IOPS
    if (projectedMetrics.iopsRead + projectedMetrics.iopsWrite > 16000) {
      recommendations.push('Upgrade to Provisioned IOPS (io1/io2) storage');
      upgradeRequired = true;
    }

    // ตรวจสอบ storage
    if (projectedMetrics.storageSizeGB > currentMetrics.storageSizeGB * 2) {
      recommendations.push(`เพิ่ม storage เป็น ${Math.ceil(projectedMetrics.storageSizeGB * 1.3)} GB`);
    }

    // ตรวจสอบ cache hit rate
    if (projectedMetrics.cacheHitRate < 0.9) {
      recommendations.push('เพิ่ม cache size หรือปรับ query optimization');
    }

    // ตรวจสอบ slow queries
    if (projectedMetrics.slowQueryCount > 100) {
      recommendations.push('ทบทวน และ optimize slow queries');
    }

    // คำนวณ estimated upgrade cost (AWS RDS)
    const estimatedUpgradeCost = upgradeRequired ? 500 : 0; // simplified

    return {
      currentLoad: currentMetrics,
      projectedLoad: projectedMetrics,
      recommendations,
      upgradeRequired,
      estimatedUpgradeCost,
    };
  }
}
```

---

## 5. Network Bandwidth Calculator

```typescript
// capacity/bandwidth-calculator.ts

interface ServiceNetworkProfile {
  serviceName: string;
  avgRequestSizeKB: number;
  avgResponseSizeKB: number;
  requestsPerSecond: number;
  externalAPICallsPerRequest: number;
  avgExternalResponseSizeKB: number;
  messageBrokerMBps: number;
  databaseQueryMBps: number;
  cacheQueryMBps: number;
}

interface NetworkCapacityResult {
  ingressMBps: number;
  egressMBps: number;
  totalMBps: number;
  monthlyGBTransferred: number;
  estimatedCostPerMonth: number;
  recommendations: string[];
}

class BandwidthCalculator {
  private readonly mbpsToGBPerMonth = (3600 * 24 * 30) / 1024; // MB/s to GB/month

  calculateServiceBandwidth(
    profile: ServiceNetworkProfile,
    safetyFactor: number = 1.3
  ): NetworkCapacityResult {
    // Ingress (incoming traffic)
    const userRequestMBps =
      (profile.requestsPerSecond * profile.avgRequestSizeKB) / 1024;
    const externalAPIIngressMBps =
      (profile.requestsPerSecond *
        profile.externalAPICallsPerRequest *
        profile.avgExternalResponseSizeKB) /
      1024;
    const messageBrokerIngressMBps = profile.messageBrokerMBps * 0.5;
    const dbIngressMBps = profile.databaseQueryMBps * 0.5;
    const cacheIngressMBps = profile.cacheQueryMBps * 0.5;

    const totalIngressMBps =
      userRequestMBps +
      externalAPIIngressMBps +
      messageBrokerIngressMBps +
      dbIngressMBps +
      cacheIngressMBps;

    // Egress (outgoing traffic)
    const userResponseMBps =
      (profile.requestsPerSecond * profile.avgResponseSizeKB) / 1024;
    const externalAPIEgressMBps =
      (profile.requestsPerSecond *
        profile.externalAPICallsPerRequest *
        (profile.avgRequestSizeKB / 4)) /
      1024; // Assume request to external is 1/4 of response
    const messageBrokerEgressMBps = profile.messageBrokerMBps * 0.5;
    const dbEgressMBps = profile.databaseQueryMBps * 0.5;
    const cacheEgressMBps = profile.cacheQueryMBps * 0.5;

    const totalEgressMBps =
      userResponseMBps +
      externalAPIEgressMBps +
      messageBrokerEgressMBps +
      dbEgressMBps +
      cacheEgressMBps;

    const totalMBps = (totalIngressMBps + totalEgressMBps) * safetyFactor;

    // Monthly data transfer
    const monthlyGBTransferred = totalMBps * this.mbpsToGBPerMonth;

    // Cost estimation (AWS: $0.09/GB after first 10TB)
    const dataTransferCost = monthlyGBTransferred * 0.09;

    const recommendations: string[] = [];

    if (totalMBps > 1000) {
      recommendations.push('พิจารณาใช้ AWS Direct Connect หรือ VPN สำหรับ traffic สูง');
    }

    if (profile.externalAPICallsPerRequest > 5) {
      recommendations.push('ใช้ API Gateway caching เพื่อลด external API calls');
    }

    if (profile.messageBrokerMBps > 100) {
      recommendations.push('เปิด message compression ใน message broker');
    }

    return {
      ingressMBps: Math.round(totalIngressMBps * safetyFactor * 100) / 100,
      egressMBps: Math.round(totalEgressMBps * safetyFactor * 100) / 100,
      totalMBps: Math.round(totalMBps * 100) / 100,
      monthlyGBTransferred: Math.round(monthlyGBTransferred),
      estimatedCostPerMonth: Math.round(dataTransferCost),
      recommendations,
    };
  }

  calculateClusterBandwidth(
    services: ServiceNetworkProfile[]
  ): {
    totalMBps: number;
    perServiceBreakdown: Array<{ serviceName: string; totalMBps: number }>;
    recommendedNetworkType: string;
    estimatedTotalCost: number;
  } {
    const breakdowns = services.map((service) => {
      const result = this.calculateServiceBandwidth(service);
      return {
        serviceName: service.serviceName,
        totalMBps: result.totalMBps,
      };
    });

    const totalMBps = breakdowns.reduce((sum, b) => sum + b.totalMBps, 0);

    let recommendedNetworkType = '1 Gbps (1000 Mbps)';
    if (totalMBps > 800) recommendedNetworkType = '10 Gbps';
    if (totalMBps > 8000) recommendedNetworkType = '25 Gbps or multiple NICs';

    const estimatedTotalCost = services.reduce((sum, service) => {
      const result = this.calculateServiceBandwidth(service);
      return sum + result.estimatedCostPerMonth;
    }, 0);

    return {
      totalMBps: Math.round(totalMBps * 100) / 100,
      perServiceBreakdown: breakdowns,
      recommendedNetworkType,
      estimatedTotalCost,
    };
  }
}
```

---

## 6. AWS Cost Optimizer

```typescript
// capacity/aws-cost-optimizer.ts

interface EC2Instance {
  instanceType: string;
  vCPUs: number;
  memoryGB: number;
  onDemandPricePerHour: number;
  reservedPricePerHour1Year: number;
  reservedPricePerHour3Year: number;
  spotPricePerHour: number;
}

interface WorkloadProfile {
  serviceName: string;
  cpuCores: number;
  memoryGB: number;
  hoursPerMonth: number;
  workloadType: 'steady' | 'variable' | 'batch';
}

class AWSCostOptimizer {
  // AWS EC2 pricing (approximate, USD/hour)
  private readonly instances: EC2Instance[] = [
    {
      instanceType: 't3.small',
      vCPUs: 2,
      memoryGB: 2,
      onDemandPricePerHour: 0.0208,
      reservedPricePerHour1Year: 0.0124,
      reservedPricePerHour3Year: 0.0089,
      spotPricePerHour: 0.0062,
    },
    {
      instanceType: 't3.medium',
      vCPUs: 2,
      memoryGB: 4,
      onDemandPricePerHour: 0.0416,
      reservedPricePerHour1Year: 0.0252,
      reservedPricePerHour3Year: 0.0181,
      spotPricePerHour: 0.0125,
    },
    {
      instanceType: 't3.large',
      vCPUs: 2,
      memoryGB: 8,
      onDemandPricePerHour: 0.0832,
      reservedPricePerHour1Year: 0.0504,
      reservedPricePerHour3Year: 0.0361,
      spotPricePerHour: 0.025,
    },
    {
      instanceType: 'm5.xlarge',
      vCPUs: 4,
      memoryGB: 16,
      onDemandPricePerHour: 0.192,
      reservedPricePerHour1Year: 0.1152,
      reservedPricePerHour3Year: 0.0825,
      spotPricePerHour: 0.0577,
    },
    {
      instanceType: 'm5.2xlarge',
      vCPUs: 8,
      memoryGB: 32,
      onDemandPricePerHour: 0.384,
      reservedPricePerHour1Year: 0.2304,
      reservedPricePerHour3Year: 0.165,
      spotPricePerHour: 0.1152,
    },
    {
      instanceType: 'c5.xlarge',
      vCPUs: 4,
      memoryGB: 8,
      onDemandPricePerHour: 0.17,
      reservedPricePerHour1Year: 0.102,
      reservedPricePerHour3Year: 0.073,
      spotPricePerHour: 0.051,
    },
    {
      instanceType: 'r5.xlarge',
      vCPUs: 4,
      memoryGB: 32,
      onDemandPricePerHour: 0.252,
      reservedPricePerHour1Year: 0.1512,
      reservedPricePerHour3Year: 0.108,
      spotPricePerHour: 0.0756,
    },
  ];

  // หา instance ที่เหมาะสมที่สุด
  findOptimalInstance(
    requiredCPUs: number,
    requiredMemoryGB: number
  ): EC2Instance | null {
    const suitable = this.instances
      .filter(
        (i) => i.vCPUs >= requiredCPUs && i.memoryGB >= requiredMemoryGB
      )
      .sort((a, b) => a.onDemandPricePerHour - b.onDemandPricePerHour);

    return suitable[0] || null;
  }

  // วิเคราะห์ Reserved vs On-demand
  analyzeReservedVsOnDemand(
    profile: WorkloadProfile
  ): {
    recommendation: 'on-demand' | 'reserved-1year' | 'reserved-3year' | 'spot';
    selectedInstance: EC2Instance | null;
    monthlyCosts: {
      onDemand: number;
      reserved1Year: number;
      reserved3Year: number;
      spot: number;
    };
    annualSavings: {
      vs1YearReserved: number;
      vs3YearReserved: number;
    };
    breakEvenMonths: {
      reserved1Year: number;
      reserved3Year: number;
    };
  } {
    const instance = this.findOptimalInstance(profile.cpuCores, profile.memoryGB);

    if (!instance) {
      return {
        recommendation: 'on-demand',
        selectedInstance: null,
        monthlyCosts: { onDemand: 0, reserved1Year: 0, reserved3Year: 0, spot: 0 },
        annualSavings: { vs1YearReserved: 0, vs3YearReserved: 0 },
        breakEvenMonths: { reserved1Year: 0, reserved3Year: 0 },
      };
    }

    const monthlyOnDemand = instance.onDemandPricePerHour * profile.hoursPerMonth;
    const monthlyReserved1Year = instance.reservedPricePerHour1Year * profile.hoursPerMonth;
    const monthlyReserved3Year = instance.reservedPricePerHour3Year * profile.hoursPerMonth;
    const monthlySpot = instance.spotPricePerHour * profile.hoursPerMonth;

    const annualSavings1Year = (monthlyOnDemand - monthlyReserved1Year) * 12;
    const annualSavings3Year = (monthlyOnDemand - monthlyReserved3Year) * 12;

    // Break-even: เดือนที่คุ้มกับการจ่าย upfront reserved
    const upfront1Year = instance.reservedPricePerHour1Year * 8760 * 0.3; // 30% upfront
    const upfront3Year = instance.reservedPricePerHour3Year * 8760 * 3 * 0.3;

    const breakEven1Year = Math.ceil(upfront1Year / (monthlyOnDemand - monthlyReserved1Year));
    const breakEven3Year = Math.ceil(upfront3Year / (monthlyOnDemand - monthlyReserved3Year));

    // เลือก recommendation
    let recommendation: 'on-demand' | 'reserved-1year' | 'reserved-3year' | 'spot';

    if (profile.workloadType === 'batch') {
      recommendation = 'spot';
    } else if (profile.workloadType === 'steady' && profile.hoursPerMonth > 500) {
      recommendation = 'reserved-3year';
    } else if (profile.hoursPerMonth > 200) {
      recommendation = 'reserved-1year';
    } else {
      recommendation = 'on-demand';
    }

    return {
      recommendation,
      selectedInstance: instance,
      monthlyCosts: {
        onDemand: Math.round(monthlyOnDemand * 100) / 100,
        reserved1Year: Math.round(monthlyReserved1Year * 100) / 100,
        reserved3Year: Math.round(monthlyReserved3Year * 100) / 100,
        spot: Math.round(monthlySpot * 100) / 100,
      },
      annualSavings: {
        vs1YearReserved: Math.round(annualSavings1Year),
        vs3YearReserved: Math.round(annualSavings3Year),
      },
      breakEvenMonths: {
        reserved1Year: isFinite(breakEven1Year) ? breakEven1Year : -1,
        reserved3Year: isFinite(breakEven3Year) ? breakEven3Year : -1,
      },
    };
  }
}
```

---

## 7. Kubernetes Right-sizing: VPA Recommendation Analyzer

```typescript
// capacity/vpa-analyzer.ts

interface VPARecommendation {
  containerName: string;
  lowerBound: { cpu: string; memory: string };
  target: { cpu: string; memory: string };
  upperBound: { cpu: string; memory: string };
  uncappedTarget: { cpu: string; memory: string };
}

interface CurrentResources {
  requests: { cpu: string; memory: string };
  limits: { cpu: string; memory: string };
}

class VPARecommendationAnalyzer {
  private parseCPU(cpu: string): number {
    // แปลง CPU string เป็น millicores
    if (cpu.endsWith('m')) return parseInt(cpu);
    return parseFloat(cpu) * 1000;
  }

  private parseMemory(memory: string): number {
    // แปลง memory string เป็น MB
    const units: Record<string, number> = {
      Ki: 1 / 1024,
      Mi: 1,
      Gi: 1024,
      K: 1 / 1024,
      M: 1,
      G: 1024,
    };

    for (const [unit, multiplier] of Object.entries(units)) {
      if (memory.endsWith(unit)) {
        return parseFloat(memory) * multiplier;
      }
    }
    return parseFloat(memory) / (1024 * 1024); // bytes to MB
  }

  analyzeRecommendation(
    podName: string,
    recommendation: VPARecommendation,
    current: CurrentResources
  ): {
    cpuChangePercent: number;
    memoryChangePercent: number;
    action: 'scale-up' | 'scale-down' | 'no-change';
    savings: { cpuMillicores: number; memoryMB: number };
    urgency: 'high' | 'medium' | 'low';
    details: string[];
  } {
    const currentCPURequests = this.parseCPU(current.requests.cpu);
    const targetCPU = this.parseCPU(recommendation.target.cpu);
    const currentMemoryRequests = this.parseMemory(current.requests.memory);
    const targetMemory = this.parseMemory(recommendation.target.memory);

    const cpuChangePercent =
      ((targetCPU - currentCPURequests) / currentCPURequests) * 100;
    const memoryChangePercent =
      ((targetMemory - currentMemoryRequests) / currentMemoryRequests) * 100;

    const cpuSavings = currentCPURequests - targetCPU;
    const memorySavings = currentMemoryRequests - targetMemory;

    let action: 'scale-up' | 'scale-down' | 'no-change';
    if (Math.abs(cpuChangePercent) < 10 && Math.abs(memoryChangePercent) < 10) {
      action = 'no-change';
    } else if (cpuChangePercent > 0 || memoryChangePercent > 0) {
      action = 'scale-up';
    } else {
      action = 'scale-down';
    }

    let urgency: 'high' | 'medium' | 'low';
    if (Math.abs(cpuChangePercent) > 50 || Math.abs(memoryChangePercent) > 50) {
      urgency = 'high';
    } else if (Math.abs(cpuChangePercent) > 25 || Math.abs(memoryChangePercent) > 25) {
      urgency = 'medium';
    } else {
      urgency = 'low';
    }

    const details: string[] = [];

    if (cpuChangePercent > 20) {
      details.push(`CPU: เพิ่มจาก ${current.requests.cpu} เป็น ${recommendation.target.cpu} (+${cpuChangePercent.toFixed(1)}%)`);
    } else if (cpuChangePercent < -20) {
      details.push(`CPU: ลดจาก ${current.requests.cpu} เป็น ${recommendation.target.cpu} (${cpuChangePercent.toFixed(1)}% savings)`);
    }

    if (memoryChangePercent > 20) {
      details.push(`Memory: เพิ่มจาก ${current.requests.memory} เป็น ${recommendation.target.memory} (+${memoryChangePercent.toFixed(1)}%)`);
    } else if (memoryChangePercent < -20) {
      details.push(`Memory: ลดจาก ${current.requests.memory} เป็น ${recommendation.target.memory} (${memoryChangePercent.toFixed(1)}% savings)`);
    }

    return {
      cpuChangePercent: Math.round(cpuChangePercent * 10) / 10,
      memoryChangePercent: Math.round(memoryChangePercent * 10) / 10,
      action,
      savings: {
        cpuMillicores: Math.round(cpuSavings),
        memoryMB: Math.round(memorySavings),
      },
      urgency,
      details,
    };
  }
}
```

---

## 8. Cache Sizing Calculator

```typescript
// capacity/cache-sizing.ts

interface CacheWorkload {
  totalObjects: number;
  avgObjectSizeKB: number;
  requestsPerSecond: number;
  targetHitRate: number; // 0.0 - 1.0
  objectAccessPattern: 'uniform' | 'zipf' | 'hotspot';
  ttlSeconds: number;
}

class CacheSizeCalculator {
  // Zipf distribution: top X% objects receive Y% of traffic
  private zipfCoverage: Array<{ topPercent: number; trafficPercent: number }> = [
    { topPercent: 1, trafficPercent: 0.5 },    // Top 1% → 50% traffic
    { topPercent: 5, trafficPercent: 0.7 },    // Top 5% → 70% traffic
    { topPercent: 10, trafficPercent: 0.8 },   // Top 10% → 80% traffic
    { topPercent: 20, trafficPercent: 0.9 },   // Top 20% → 90% traffic
    { topPercent: 50, trafficPercent: 0.95 },  // Top 50% → 95% traffic
  ];

  calculateCacheSize(workload: CacheWorkload): {
    recommendedSizeGB: number;
    objectsToCache: number;
    achievableHitRate: number;
    memoryCost: Record<string, { sizeGB: number; hitRate: number; monthlyCostUSD: number }>;
    recommendation: string;
  } {
    let objectsToCache: number;

    switch (workload.objectAccessPattern) {
      case 'uniform':
        // Uniform: ต้อง cache ทุกอย่างเพื่อให้ได้ hit rate ที่ต้องการ
        objectsToCache = Math.ceil(workload.totalObjects * workload.targetHitRate);
        break;

      case 'zipf':
        // Zipf: หา minimum % ที่ cover target hit rate
        const coverage = this.zipfCoverage.find(
          (c) => c.trafficPercent >= workload.targetHitRate
        );
        const percentToCover = coverage ? coverage.topPercent / 100 : 0.5;
        objectsToCache = Math.ceil(workload.totalObjects * percentToCover);
        break;

      case 'hotspot':
        // 80/20 rule: 20% of objects = 80% of traffic
        if (workload.targetHitRate <= 0.8) {
          objectsToCache = Math.ceil(workload.totalObjects * 0.2);
        } else {
          objectsToCache = Math.ceil(workload.totalObjects * 0.5);
        }
        break;

      default:
        objectsToCache = Math.ceil(workload.totalObjects * workload.targetHitRate);
    }

    const baselineSizeGB =
      (objectsToCache * workload.avgObjectSizeKB) / (1024 * 1024);
    const overhead = 1.2; // 20% overhead สำหรับ Redis metadata
    const recommendedSizeGB = Math.ceil(baselineSizeGB * overhead * 10) / 10;

    // คำนวณ achievable hit rate
    const achievableHitRate =
      workload.objectAccessPattern === 'zipf'
        ? Math.min(0.95, workload.targetHitRate * 1.05)
        : workload.targetHitRate;

    // Memory options และ cost
    const memoryCost: Record<string, { sizeGB: number; hitRate: number; monthlyCostUSD: number }> = {};
    const memorySizes = [1, 2, 4, 8, 16, 32, 64];

    for (const sizeGB of memorySizes) {
      if (sizeGB < recommendedSizeGB * 0.5) continue;

      const objectsFit = Math.floor(
        (sizeGB * 1024 * 1024) / (workload.avgObjectSizeKB * overhead)
      );
      const coverageRatio = Math.min(1, objectsFit / workload.totalObjects);

      let hitRate: number;
      switch (workload.objectAccessPattern) {
        case 'zipf':
          const zipfCoverage = this.zipfCoverage.find(
            (c) => c.topPercent / 100 >= coverageRatio
          );
          hitRate = zipfCoverage ? zipfCoverage.trafficPercent : 0.99;
          break;
        case 'hotspot':
          hitRate = coverageRatio >= 0.2 ? 0.8 : coverageRatio * 4 * 0.8;
          break;
        default:
          hitRate = coverageRatio;
      }

      // AWS ElastiCache Redis cost (~$0.034/hour per GB for cache.r6g.large)
      const monthlyCost = Math.ceil(sizeGB * 0.034 * 730);

      memoryCost[`${sizeGB}GB`] = {
        sizeGB,
        hitRate: Math.round(hitRate * 1000) / 1000,
        monthlyCostUSD: monthlyCost,
      };
    }

    const recommendation =
      achievableHitRate >= 0.95
        ? `Cache ${recommendedSizeGB}GB ให้ hit rate ${(achievableHitRate * 100).toFixed(1)}% (ดีมาก)`
        : achievableHitRate >= 0.80
        ? `Cache ${recommendedSizeGB}GB ให้ hit rate ${(achievableHitRate * 100).toFixed(1)}% (ดี)`
        : `Hit rate ต่ำ - พิจารณา prefetch หรือปรับ TTL`;

    return {
      recommendedSizeGB,
      objectsToCache,
      achievableHitRate,
      memoryCost,
      recommendation,
    };
  }
}
```

---

## 9. Kafka Capacity Calculator

```typescript
// capacity/kafka-capacity.ts

interface KafkaWorkload {
  topicsCount: number;
  messagesPerSecond: number;
  avgMessageSizeKB: number;
  replicationFactor: number;
  retentionDays: number;
  consumerGroups: number;
  targetThroughputMBps: number;
}

class KafkaCapacityCalculator {
  calculatePartitionCount(workload: KafkaWorkload): {
    recommendedPartitions: number;
    throughputPerPartition: number;
    reasoning: string[];
  } {
    const reasoning: string[] = [];

    // กฎทั่วไป: partition ต้องรองรับ throughput ที่ต้องการ
    const partitionThroughputMBps = 10; // MB/s per partition (typical Kafka limit)
    const partitionsForThroughput = Math.ceil(
      workload.targetThroughputMBps / partitionThroughputMBps
    );
    reasoning.push(
      `Throughput requirement: ${workload.targetThroughputMBps} MB/s ÷ ${partitionThroughputMBps} MB/s/partition = ${partitionsForThroughput} partitions`
    );

    // คำนึงถึง consumer parallelism
    const partitionsForConsumers = workload.consumerGroups * 3; // 3 consumers per group
    reasoning.push(
      `Consumer parallelism: ${workload.consumerGroups} groups × 3 = ${partitionsForConsumers} partitions`
    );

    // เลือกค่าที่มากกว่า
    const recommended = Math.max(
      partitionsForThroughput,
      partitionsForConsumers,
      3 // minimum 3 partitions
    );

    reasoning.push(
      `Recommended: max(${partitionsForThroughput}, ${partitionsForConsumers}, 3) = ${recommended}`
    );

    return {
      recommendedPartitions: recommended,
      throughputPerPartition:
        Math.round((workload.targetThroughputMBps / recommended) * 100) / 100,
      reasoning,
    };
  }

  calculateBrokerSizing(workload: KafkaWorkload): {
    recommendedBrokers: number;
    storagePerBrokerGB: number;
    cpuPerBroker: number;
    memoryPerBrokerGB: number;
    networkPerBroker: string;
    totalClusterCost: number;
  } {
    // Storage calculation
    const totalDataPerDayGB =
      (workload.messagesPerSecond * workload.avgMessageSizeKB * 86400) / (1024 * 1024);
    const totalRetentionGB = totalDataPerDayGB * workload.retentionDays;
    const totalWithReplication = totalRetentionGB * workload.replicationFactor;

    // Minimum 3 brokers for HA
    const minBrokers = Math.max(3, workload.replicationFactor);
    const storagePerBroker = Math.ceil(totalWithReplication / minBrokers * 1.2); // 20% buffer

    // CPU: 4 cores ต่อทุก 100 MB/s throughput
    const totalThroughputMBps =
      workload.targetThroughputMBps * workload.replicationFactor;
    const totalCPUCores = Math.ceil((totalThroughputMBps / 100) * 4);
    const cpuPerBroker = Math.max(4, Math.ceil(totalCPUCores / minBrokers));

    // Memory: 6GB + 1GB per partition
    const { recommendedPartitions } = this.calculatePartitionCount(workload);
    const memoryPerBroker = Math.max(6, 6 + Math.ceil(recommendedPartitions / 10));

    // Network
    const networkMBps = totalThroughputMBps / minBrokers;
    const networkType =
      networkMBps > 800
        ? '10 Gbps'
        : networkMBps > 200
        ? '1 Gbps'
        : '1 Gbps (sufficient)';

    // Cost estimation (i3.xlarge: $0.312/hour)
    const instanceCostPerHour = 0.312;
    const storageCostPerGB = 0.025; // SSD
    const totalClusterCost = Math.round(
      minBrokers * instanceCostPerHour * 730 +
        storagePerBroker * minBrokers * storageCostPerGB
    );

    return {
      recommendedBrokers: minBrokers,
      storagePerBrokerGB: storagePerBroker,
      cpuPerBroker,
      memoryPerBrokerGB: memoryPerBroker,
      networkPerBroker: networkType,
      totalClusterCost,
    };
  }
}
```

---

## 10. Multi-Region Capacity Report Generator

```typescript
// capacity/multi-region-report.ts

interface RegionCapacity {
  region: string;
  services: string[];
  estimatedUsers: number;
  peakRPS: number;
  currentReplicaCount: number;
  recommendedReplicaCount: number;
  currentCostMonthly: number;
  projectedCostMonthly: number;
}

class MultiRegionCapacityReporter {
  private readonly loadModel: LoadModel;
  private readonly costOptimizer: AWSCostOptimizer;

  constructor() {
    const baselineMetrics = {
      requestsPerSecond: 100,
      avgResponseTimeMs: 50,
      p99ResponseTimeMs: 200,
      errorRate: 0.1,
      cpuUsagePercent: 30,
      memoryUsageMB: 512,
      networkInMBps: 5,
      networkOutMBps: 10,
    };
    this.loadModel = new LoadModel(baselineMetrics, 1.3);
    this.costOptimizer = new AWSCostOptimizer();
  }

  async generateReport(regions: RegionCapacity[]): Promise<string> {
    const lines: string[] = [
      '# Multi-Region Capacity Planning Report',
      `Generated: ${new Date().toISOString()}`,
      '',
      '## Executive Summary',
      '',
    ];

    let totalCurrentCost = 0;
    let totalProjectedCost = 0;
    const regionSummaries: string[] = [];

    for (const region of regions) {
      totalCurrentCost += region.currentCostMonthly;
      totalProjectedCost += region.projectedCostMonthly;

      const costDiff = region.projectedCostMonthly - region.currentCostMonthly;
      const costChangePercent = ((costDiff / region.currentCostMonthly) * 100).toFixed(1);

      regionSummaries.push(
        `| ${region.region} | ${region.estimatedUsers.toLocaleString()} | ${region.peakRPS} | ` +
          `${region.currentReplicaCount} | ${region.recommendedReplicaCount} | ` +
          `$${region.currentCostMonthly} | $${region.projectedCostMonthly} | ` +
          `${costDiff > 0 ? '+' : ''}${costChangePercent}% |`
      );
    }

    // Summary table
    lines.push('| Region | Users | Peak RPS | Current Replicas | Recommended | Current Cost | Projected Cost | Change |');
    lines.push('|--------|-------|----------|-----------------|-------------|-------------|----------------|--------|');
    lines.push(...regionSummaries);
    lines.push('');
    lines.push(`**Total Current Monthly Cost:** $${totalCurrentCost.toLocaleString()}`);
    lines.push(`**Total Projected Monthly Cost:** $${totalProjectedCost.toLocaleString()}`);
    lines.push(`**Total Change:** $${(totalProjectedCost - totalCurrentCost).toLocaleString()}`);
    lines.push('');

    // Detailed per-region analysis
    for (const region of regions) {
      lines.push(`## ${region.region}`);
      lines.push('');
      lines.push(`- **Estimated Users:** ${region.estimatedUsers.toLocaleString()}`);
      lines.push(`- **Peak RPS:** ${region.peakRPS}`);
      lines.push(`- **Services:** ${region.services.join(', ')}`);
      lines.push('');

      // Resource requirements
      const resources = this.loadModel.calculateResourcesForRPS(region.peakRPS);
      lines.push('### Resource Requirements');
      lines.push(`- CPU per replica: ${resources.cpuCores} cores`);
      lines.push(`- Memory per replica: ${resources.memoryGB} GB`);
      lines.push(`- Recommended replicas: ${resources.replicaCount}`);
      lines.push('');

      // Cost analysis
      const costAnalysis = this.costOptimizer.analyzeReservedVsOnDemand({
        serviceName: region.region,
        cpuCores: resources.cpuCores * resources.replicaCount,
        memoryGB: resources.memoryGB * resources.replicaCount,
        hoursPerMonth: 730,
        workloadType: 'steady',
      });

      if (costAnalysis.selectedInstance) {
        lines.push('### Cost Analysis');
        lines.push(`- Recommended instance: ${costAnalysis.selectedInstance.instanceType}`);
        lines.push(`- On-demand monthly: $${costAnalysis.monthlyCosts.onDemand}`);
        lines.push(`- Reserved 1Y monthly: $${costAnalysis.monthlyCosts.reserved1Year}`);
        lines.push(`- Reserved 3Y monthly: $${costAnalysis.monthlyCosts.reserved3Year}`);
        lines.push(`- **Recommendation: ${costAnalysis.recommendation}**`);
        lines.push('');
      }
    }

    return lines.join('\n');
  }
}

// ตัวอย่างการใช้งาน
async function generateMultiRegionReport() {
  const reporter = new MultiRegionCapacityReporter();

  const regions: RegionCapacity[] = [
    {
      region: 'ap-southeast-1 (Singapore)',
      services: ['api-gateway', 'order-service', 'user-service'],
      estimatedUsers: 50000,
      peakRPS: 500,
      currentReplicaCount: 3,
      recommendedReplicaCount: 5,
      currentCostMonthly: 2500,
      projectedCostMonthly: 3800,
    },
    {
      region: 'us-east-1 (N. Virginia)',
      services: ['api-gateway', 'order-service', 'user-service', 'analytics-service'],
      estimatedUsers: 200000,
      peakRPS: 2000,
      currentReplicaCount: 10,
      recommendedReplicaCount: 15,
      currentCostMonthly: 8000,
      projectedCostMonthly: 12000,
    },
    {
      region: 'eu-west-1 (Ireland)',
      services: ['api-gateway', 'order-service', 'user-service'],
      estimatedUsers: 75000,
      peakRPS: 750,
      currentReplicaCount: 4,
      recommendedReplicaCount: 7,
      currentCostMonthly: 3500,
      projectedCostMonthly: 5500,
    },
  ];

  const report = await reporter.generateReport(regions);
  console.log(report);
}

generateMultiRegionReport().catch(console.error);
```

---

## สรุป

| หัวข้อ | เนื้อหาสำคัญ |
|--------|-------------|
| Load Model | คำนวณ resources จาก RPS, concurrent users, และ peak multiplier |
| Baseline Collector | เก็บ baseline metrics จาก Prometheus สำหรับ 7-30 วัน |
| Linear Growth Predictor | Regression model สำหรับ predict growth ตาม trend |
| Seasonal Predictor | Holt-Winters model สำหรับ pattern ตามฤดูกาล |
| Database Capacity | คำนวณ connections, IOPS, storage growth |
| Network Bandwidth | คำนวณ ingress/egress ต่อ service และ cluster |
| AWS Cost Optimizer | วิเคราะห์ Reserved vs On-demand vs Spot pricing |
| VPA Analyzer | วิเคราะห์ Kubernetes VPA recommendations |
| Cache Sizing | คำนวณ cache size ตาม hit rate target และ access pattern |
| Kafka Capacity | คำนวณ partition count และ broker sizing |
| Multi-Region Report | สร้าง capacity report สำหรับหลาย regions |

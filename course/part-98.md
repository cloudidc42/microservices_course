# Part 98: Cost Optimization Strategies

## บทนำ

FinOps (Financial Operations) คือการนำ Financial Accountability มาใช้กับ Cloud Infrastructure Microservices สามารถ Scale ได้ง่าย แต่ถ้าไม่มีการ Monitor Cost อย่างระมัดระวัง ค่าใช้จ่ายอาจพุ่งสูงโดยไม่รู้ตัว บทนี้จะสอนการ Track, Analyze และ Optimize Cost ของ Microservices บน AWS

---

## 1. TypeScript CostDashboard: AWS Cost Explorer

```typescript
// services/finops/src/cost-dashboard.service.ts
import {
  CostExplorerClient,
  GetCostAndUsageCommand,
  GetCostForecastCommand,
  GetDimensionValuesCommand,
  Dimension,
  Granularity,
  GetRightsizingRecommendationCommand,
  RightsizingType,
} from '@aws-sdk/client-cost-explorer';
import { Redis } from 'ioredis';

interface CostBreakdown {
  service: string;
  amount: number;
  currency: string;
  unit: string;
  changePercent: number;
}

interface MonthlyCostReport {
  month: string;
  totalCost: number;
  currency: string;
  byService: CostBreakdown[];
  byTag: Record<string, number>;
  forecast: {
    estimatedMonthEnd: number;
    lowerBound: number;
    upperBound: number;
  };
}

interface ServiceCostTrend {
  dates: string[];
  costs: number[];
  avgDailyCost: number;
  peakDay: string;
  peakCost: number;
}

export class CostDashboardService {
  private costExplorer: CostExplorerClient;
  private redis: Redis;
  private readonly CACHE_TTL = 3600; // 1 hour

  constructor() {
    this.costExplorer = new CostExplorerClient({
      region: 'us-east-1', // Cost Explorer is global, us-east-1
    });
    this.redis = new Redis({ host: process.env.REDIS_HOST });
  }

  async getMonthlyCostReport(month?: string): Promise<MonthlyCostReport> {
    const targetMonth = month || this.getCurrentMonth();
    const cacheKey = `cost:monthly:${targetMonth}`;
    const cached = await this.redis.get(cacheKey);
    if (cached) return JSON.parse(cached);

    const [startDate, endDate] = this.getMonthRange(targetMonth);

    // Get cost by service
    const costByServiceCmd = new GetCostAndUsageCommand({
      TimePeriod: { Start: startDate, End: endDate },
      Granularity: 'MONTHLY' as Granularity,
      Metrics: ['BlendedCost', 'UnblendedCost', 'UsageQuantity'],
      GroupBy: [{ Type: 'DIMENSION', Key: 'SERVICE' as Dimension }],
    });

    const serviceResponse = await this.costExplorer.send(costByServiceCmd);
    const resultByTime = serviceResponse.ResultsByTime?.[0];

    const byService: CostBreakdown[] = (resultByTime?.Groups || []).map(group => ({
      service: group.Keys?.[0] || 'Unknown',
      amount: parseFloat(group.Metrics?.BlendedCost?.Amount || '0'),
      currency: group.Metrics?.BlendedCost?.Unit || 'USD',
      unit: group.Metrics?.UsageQuantity?.Unit || '',
      changePercent: 0, // Calculate separately
    }));

    const totalCost = byService.reduce((sum, s) => sum + s.amount, 0);

    // Get cost by tag (service name tag)
    const costByTagCmd = new GetCostAndUsageCommand({
      TimePeriod: { Start: startDate, End: endDate },
      Granularity: 'MONTHLY' as Granularity,
      Metrics: ['BlendedCost'],
      GroupBy: [{ Type: 'TAG', Key: 'service' }],
      Filter: {
        Tags: {
          Key: 'environment',
          Values: ['production'],
        },
      },
    });

    const tagResponse = await this.costExplorer.send(costByTagCmd);
    const byTag: Record<string, number> = {};
    for (const group of tagResponse.ResultsByTime?.[0]?.Groups || []) {
      const tagValue = group.Keys?.[0]?.split('$')[1] || 'untagged';
      byTag[tagValue] = parseFloat(group.Metrics?.BlendedCost?.Amount || '0');
    }

    // Get forecast
    const forecastCmd = new GetCostForecastCommand({
      TimePeriod: {
        Start: new Date().toISOString().split('T')[0],
        End: this.getMonthEnd(targetMonth),
      },
      Metric: 'BLENDED_COST',
      Granularity: 'MONTHLY' as Granularity,
    });

    let forecast = { estimatedMonthEnd: totalCost, lowerBound: totalCost * 0.9, upperBound: totalCost * 1.1 };
    try {
      const forecastResponse = await this.costExplorer.send(forecastCmd);
      const total = forecastResponse.Total;
      forecast = {
        estimatedMonthEnd: parseFloat(total?.Amount || '0'),
        lowerBound: parseFloat(forecastResponse.ForecastResultsByTime?.[0]?.PredictionIntervalLowerBound || '0'),
        upperBound: parseFloat(forecastResponse.ForecastResultsByTime?.[0]?.PredictionIntervalUpperBound || '0'),
      };
    } catch (err) {
      console.warn('Could not get forecast:', err);
    }

    const report: MonthlyCostReport = {
      month: targetMonth,
      totalCost: Math.round(totalCost * 100) / 100,
      currency: 'USD',
      byService: byService.sort((a, b) => b.amount - a.amount).slice(0, 20),
      byTag,
      forecast,
    };

    await this.redis.setex(cacheKey, this.CACHE_TTL, JSON.stringify(report));
    return report;
  }

  async getServiceCostTrend(
    serviceName: string,
    days: number = 30
  ): Promise<ServiceCostTrend> {
    const endDate = new Date().toISOString().split('T')[0];
    const startDate = new Date(Date.now() - days * 86400000).toISOString().split('T')[0];

    const command = new GetCostAndUsageCommand({
      TimePeriod: { Start: startDate, End: endDate },
      Granularity: 'DAILY' as Granularity,
      Metrics: ['BlendedCost'],
      Filter: {
        Tags: {
          Key: 'service',
          Values: [serviceName],
        },
      },
    });

    const response = await this.costExplorer.send(command);
    const results = response.ResultsByTime || [];

    const dates = results.map(r => r.TimePeriod?.Start || '');
    const costs = results.map(r =>
      parseFloat(r.Total?.BlendedCost?.Amount || '0')
    );

    const avgDailyCost = costs.reduce((a, b) => a + b, 0) / costs.length;
    const peakIndex = costs.indexOf(Math.max(...costs));

    return {
      dates,
      costs,
      avgDailyCost: Math.round(avgDailyCost * 100) / 100,
      peakDay: dates[peakIndex] || '',
      peakCost: costs[peakIndex] || 0,
    };
  }

  async getRightsizingRecommendations(): Promise<Array<{
    instanceId: string;
    currentType: string;
    recommendedType: string;
    estimatedMonthlySavings: number;
    cpuUtilization: number;
    memoryUtilization: number;
  }>> {
    const command = new GetRightsizingRecommendationCommand({
      Service: 'AmazonEC2',
      Configuration: {
        BenefitsConsidered: true,
        RecommendationTarget: 'CROSS_INSTANCE_FAMILY' as RightsizingType,
      },
      PageSize: 100,
    });

    const response = await this.costExplorer.send(command);

    return (response.RightsizingRecommendations || []).map(rec => ({
      instanceId: rec.CurrentInstance?.ResourceId || '',
      currentType: rec.CurrentInstance?.ResourceDetails?.EC2ResourceDetails?.InstanceType || '',
      recommendedType: rec.ModifyRecommendationDetail?.TargetInstances?.[0]?.ResourceDetails?.EC2ResourceDetails?.InstanceType || '',
      estimatedMonthlySavings: parseFloat(
        rec.ModifyRecommendationDetail?.TargetInstances?.[0]?.EstimatedMonthlySavings?.Value || '0'
      ),
      cpuUtilization: parseFloat(
        rec.CurrentInstance?.ResourceUtilization?.EC2ResourceUtilization?.MaxCpuUtilizationPercentage || '0'
      ),
      memoryUtilization: parseFloat(
        rec.CurrentInstance?.ResourceUtilization?.EC2ResourceUtilization?.MaxMemoryUtilizationPercentage || '0'
      ),
    }));
  }

  private getCurrentMonth(): string {
    return new Date().toISOString().substring(0, 7);
  }

  private getMonthRange(month: string): [string, string] {
    const date = new Date(month + '-01');
    const startDate = date.toISOString().split('T')[0];
    const endDate = new Date(date.getFullYear(), date.getMonth() + 1, 1)
      .toISOString()
      .split('T')[0];
    return [startDate, endDate];
  }

  private getMonthEnd(month: string): string {
    const date = new Date(month + '-01');
    return new Date(date.getFullYear(), date.getMonth() + 1, 1)
      .toISOString()
      .split('T')[0];
  }
}
```

---

## 2. Container Right-Sizing Analyzer

```typescript
// services/finops/src/container-rightsizing.ts
import {
  CloudWatchClient,
  GetMetricStatisticsCommand,
  Statistic,
} from '@aws-sdk/client-cloudwatch';
import { KubeConfig, CoreV1Api } from '@kubernetes/client-node';

interface ContainerMetrics {
  podName: string;
  containerName: string;
  namespace: string;
  requests: { cpu: string; memory: string };
  limits: { cpu: string; memory: string };
  actual: {
    avgCpuPercent: number;
    maxCpuPercent: number;
    avgMemoryMB: number;
    maxMemoryMB: number;
  };
  recommendations: {
    cpuRequest: string;
    memoryRequest: string;
    wastedCpu: number;
    wastedMemoryMB: number;
    estimatedMonthlySavings: number;
  };
}

export class ContainerRightsizingAnalyzer {
  private k8sApi: CoreV1Api;
  private cloudWatch: CloudWatchClient;

  // Cost per vCPU-hour and GB-hour (EKS Fargate pricing)
  private readonly CPU_COST_PER_VCPU_HOUR = 0.04048;
  private readonly MEMORY_COST_PER_GB_HOUR = 0.004445;

  constructor() {
    const kc = new KubeConfig();
    kc.loadFromDefault();
    this.k8sApi = kc.makeApiClient(CoreV1Api);
    this.cloudWatch = new CloudWatchClient({ region: process.env.AWS_REGION });
  }

  async analyzeNamespace(namespace: string): Promise<ContainerMetrics[]> {
    const podsResponse = await this.k8sApi.listNamespacedPod(namespace);
    const metrics: ContainerMetrics[] = [];

    for (const pod of podsResponse.body.items) {
      if (!pod.metadata?.name || !pod.spec?.containers) continue;

      for (const container of pod.spec.containers) {
        const containerMetrics = await this.analyzeContainer(
          pod.metadata.name,
          container.name,
          namespace,
          container.resources
        );
        if (containerMetrics) metrics.push(containerMetrics);
      }
    }

    return metrics.sort((a, b) =>
      b.recommendations.estimatedMonthlySavings - a.recommendations.estimatedMonthlySavings
    );
  }

  private async analyzeContainer(
    podName: string,
    containerName: string,
    namespace: string,
    resources: any
  ): Promise<ContainerMetrics | null> {
    const requests = {
      cpu: resources?.requests?.cpu || '100m',
      memory: resources?.requests?.memory || '128Mi',
    };
    const limits = {
      cpu: resources?.limits?.cpu || requests.cpu,
      memory: resources?.limits?.memory || requests.memory,
    };

    // Get actual metrics from CloudWatch Container Insights
    const endTime = new Date();
    const startTime = new Date(endTime.getTime() - 7 * 24 * 60 * 60 * 1000); // 7 days

    const [cpuMetrics, memoryMetrics] = await Promise.all([
      this.getContainerMetric(namespace, podName, containerName, 'pod_cpu_utilized', startTime, endTime),
      this.getContainerMetric(namespace, podName, containerName, 'pod_memory_utilized', startTime, endTime),
    ]);

    if (!cpuMetrics.length && !memoryMetrics.length) return null;

    const requestedCpuMillis = this.parseCPU(requests.cpu);
    const requestedMemoryMB = this.parseMemory(requests.memory);

    const avgCpuMillis = cpuMetrics.reduce((a, b) => a + b, 0) / (cpuMetrics.length || 1);
    const maxCpuMillis = Math.max(...cpuMetrics, 0);
    const avgMemoryMB = memoryMetrics.reduce((a, b) => a + b, 0) / (memoryMetrics.length || 1);
    const maxMemoryMB = Math.max(...memoryMetrics, 0);

    // Recommend: 20% headroom above max usage
    const recommendedCpuMillis = Math.ceil(maxCpuMillis * 1.2);
    const recommendedMemoryMB = Math.ceil(maxMemoryMB * 1.2);

    // Calculate wasted resources
    const wastedCpuMillis = Math.max(0, requestedCpuMillis - recommendedCpuMillis);
    const wastedMemoryMB = Math.max(0, requestedMemoryMB - recommendedMemoryMB);

    // Estimate monthly savings (720 hours/month)
    const hoursPerMonth = 720;
    const savedCpuCost = (wastedCpuMillis / 1000) * this.CPU_COST_PER_VCPU_HOUR * hoursPerMonth;
    const savedMemoryCost = (wastedMemoryMB / 1024) * this.MEMORY_COST_PER_GB_HOUR * hoursPerMonth;
    const estimatedMonthlySavings = savedCpuCost + savedMemoryCost;

    return {
      podName,
      containerName,
      namespace,
      requests,
      limits,
      actual: {
        avgCpuPercent: Math.round((avgCpuMillis / requestedCpuMillis) * 100),
        maxCpuPercent: Math.round((maxCpuMillis / requestedCpuMillis) * 100),
        avgMemoryMB: Math.round(avgMemoryMB),
        maxMemoryMB: Math.round(maxMemoryMB),
      },
      recommendations: {
        cpuRequest: `${recommendedCpuMillis}m`,
        memoryRequest: `${recommendedMemoryMB}Mi`,
        wastedCpu: wastedCpuMillis,
        wastedMemoryMB,
        estimatedMonthlySavings: Math.round(estimatedMonthlySavings * 100) / 100,
      },
    };
  }

  private async getContainerMetric(
    namespace: string,
    podName: string,
    containerName: string,
    metricName: string,
    startTime: Date,
    endTime: Date
  ): Promise<number[]> {
    try {
      const response = await this.cloudWatch.send(
        new GetMetricStatisticsCommand({
          Namespace: 'ContainerInsights',
          MetricName: metricName,
          Dimensions: [
            { Name: 'Namespace', Value: namespace },
            { Name: 'PodName', Value: podName },
            { Name: 'ContainerName', Value: containerName },
            { Name: 'ClusterName', Value: process.env.CLUSTER_NAME || 'my-cluster' },
          ],
          StartTime: startTime,
          EndTime: endTime,
          Period: 3600, // 1 hour intervals
          Statistics: ['Average', 'Maximum'] as Statistic[],
        })
      );

      return (response.Datapoints || []).map(dp => dp.Average || 0);
    } catch {
      return [];
    }
  }

  private parseCPU(cpu: string): number {
    if (cpu.endsWith('m')) return parseInt(cpu);
    return parseFloat(cpu) * 1000;
  }

  private parseMemory(memory: string): number {
    if (memory.endsWith('Mi')) return parseInt(memory);
    if (memory.endsWith('Gi')) return parseInt(memory) * 1024;
    if (memory.endsWith('Ki')) return parseInt(memory) / 1024;
    return parseInt(memory) / (1024 * 1024);
  }

  generatePatchManifest(metrics: ContainerMetrics[]): string {
    const patches = metrics
      .filter(m => m.recommendations.estimatedMonthlySavings > 1)
      .map(m => ({
        op: 'replace',
        path: `/spec/containers/${m.containerName}/resources/requests`,
        value: {
          cpu: m.recommendations.cpuRequest,
          memory: m.recommendations.memoryRequest,
        },
      }));

    return JSON.stringify(patches, null, 2);
  }
}
```

---

## 3. Spot Instance Handler

```typescript
// services/infra/src/spot-termination-handler.ts
import axios from 'axios';
import { createServer } from 'http';

interface TerminationNotice {
  action: string;
  time: string;
}

type ShutdownCallback = () => Promise<void>;

export class SpotTerminationHandler {
  private callbacks: ShutdownCallback[] = [];
  private isTerminating = false;
  private readonly METADATA_URL = 'http://169.254.169.254/latest/meta-data';
  private readonly CHECK_INTERVAL_MS = 5000;
  private checkTimer: NodeJS.Timeout | null = null;

  register(callback: ShutdownCallback): void {
    this.callbacks.push(callback);
  }

  start(): void {
    console.log('Spot termination handler started');

    // Poll EC2 instance metadata for termination notice
    this.checkTimer = setInterval(
      () => this.checkForTerminationNotice(),
      this.CHECK_INTERVAL_MS
    );

    // Handle Unix signals
    process.on('SIGTERM', () => this.gracefulShutdown('SIGTERM'));
    process.on('SIGINT', () => this.gracefulShutdown('SIGINT'));

    // Handle uncaught exceptions
    process.on('uncaughtException', async (err) => {
      console.error('Uncaught exception:', err);
      await this.gracefulShutdown('uncaughtException');
    });

    process.on('unhandledRejection', async (reason) => {
      console.error('Unhandled rejection:', reason);
      // Don't shut down for unhandled rejections unless critical
    });
  }

  private async checkForTerminationNotice(): Promise<void> {
    if (this.isTerminating) return;

    try {
      const response = await axios.get(
        `${this.METADATA_URL}/spot/termination-time`,
        { timeout: 1000 }
      );

      if (response.status === 200) {
        console.warn('Spot instance termination notice received!', response.data);
        await this.gracefulShutdown('spot-termination');
      }
    } catch (err: any) {
      // 404 means no termination notice (normal)
      if (err.response?.status !== 404) {
        // Silently ignore connection errors (metadata not available in non-AWS envs)
      }
    }
  }

  private async gracefulShutdown(reason: string): Promise<void> {
    if (this.isTerminating) return;
    this.isTerminating = true;

    console.log(`Graceful shutdown initiated. Reason: ${reason}`);

    if (this.checkTimer) {
      clearInterval(this.checkTimer);
      this.checkTimer = null;
    }

    // Execute all registered shutdown callbacks in sequence
    for (const callback of this.callbacks) {
      try {
        console.log('Executing shutdown callback...');
        await Promise.race([
          callback(),
          new Promise((_, reject) =>
            setTimeout(() => reject(new Error('Callback timeout')), 25000)
          ),
        ]);
      } catch (err) {
        console.error('Shutdown callback error:', err);
      }
    }

    console.log('Graceful shutdown complete');
    process.exit(0);
  }

  stop(): void {
    if (this.checkTimer) {
      clearInterval(this.checkTimer);
    }
  }
}

// Example usage in a service
export function setupGracefulShutdown(
  server: ReturnType<typeof createServer>,
  kafkaConsumer?: { disconnect: () => Promise<void> },
  dbClient?: { $disconnect: () => Promise<void> }
): SpotTerminationHandler {
  const handler = new SpotTerminationHandler();

  // Stop accepting new connections
  handler.register(async () => {
    console.log('Stopping HTTP server...');
    await new Promise<void>((resolve, reject) => {
      server.close(err => err ? reject(err) : resolve());
    });
    console.log('HTTP server stopped');
  });

  // Flush Kafka messages
  if (kafkaConsumer) {
    handler.register(async () => {
      console.log('Disconnecting Kafka consumer...');
      await kafkaConsumer.disconnect();
      console.log('Kafka consumer disconnected');
    });
  }

  // Close database connections
  if (dbClient) {
    handler.register(async () => {
      console.log('Closing database connections...');
      await dbClient.$disconnect();
      console.log('Database connections closed');
    });
  }

  handler.start();
  return handler;
}
```

---

## 4. Reserved Capacity Calculator

```typescript
// services/finops/src/reserved-capacity-calculator.ts

interface InstanceOption {
  instanceType: string;
  region: string;
  odPriceHourly: number;
  ri1YearNoUpfrontHourly: number;
  ri3YearNoUpfrontHourly: number;
  ri1YearAllUpfront: number;
  ri3YearAllUpfront: number;
}

interface ROICalculation {
  instanceType: string;
  currentMonthlyCost: number;
  ri1YearMonthlyCost: number;
  ri3YearMonthlyCost: number;
  savings1Year: { monthly: number; annual: number; percent: number };
  savings3Year: { monthly: number; total: number; percent: number };
  breakeven1Year: { months: number; date: string };
  recommendation: 'on-demand' | 'ri-1-year' | 'ri-3-year';
  paybackPeriod: number;
}

// Real AWS pricing for ap-southeast-1 (Singapore)
const EC2_PRICING: Record<string, InstanceOption> = {
  't3.micro': {
    instanceType: 't3.micro',
    region: 'ap-southeast-1',
    odPriceHourly: 0.0116,
    ri1YearNoUpfrontHourly: 0.007,
    ri3YearNoUpfrontHourly: 0.0044,
    ri1YearAllUpfront: 51,
    ri3YearAllUpfront: 116,
  },
  't3.medium': {
    instanceType: 't3.medium',
    region: 'ap-southeast-1',
    odPriceHourly: 0.0464,
    ri1YearNoUpfrontHourly: 0.028,
    ri3YearNoUpfrontHourly: 0.0176,
    ri1YearAllUpfront: 204,
    ri3YearAllUpfront: 463,
  },
  'm5.large': {
    instanceType: 'm5.large',
    region: 'ap-southeast-1',
    odPriceHourly: 0.107,
    ri1YearNoUpfrontHourly: 0.0642,
    ri3YearNoUpfrontHourly: 0.0404,
    ri1YearAllUpfront: 473,
    ri3YearAllUpfront: 1064,
  },
  'm5.xlarge': {
    instanceType: 'm5.xlarge',
    region: 'ap-southeast-1',
    odPriceHourly: 0.214,
    ri1YearNoUpfrontHourly: 0.1284,
    ri3YearNoUpfrontHourly: 0.0808,
    ri1YearAllUpfront: 945,
    ri3YearAllUpfront: 2127,
  },
  'r5.large': {
    instanceType: 'r5.large',
    region: 'ap-southeast-1',
    odPriceHourly: 0.138,
    ri1YearNoUpfrontHourly: 0.0828,
    ri3YearNoUpfrontHourly: 0.0522,
    ri1YearAllUpfront: 610,
    ri3YearAllUpfront: 1371,
  },
};

export class ReservedCapacityCalculator {
  calculateROI(
    instanceType: string,
    instanceCount: number = 1,
    utilizationPercent: number = 100
  ): ROICalculation {
    const pricing = EC2_PRICING[instanceType];
    if (!pricing) throw new Error(`Unknown instance type: ${instanceType}`);

    const hoursPerMonth = 720;
    const adjustedHoursPerMonth = hoursPerMonth * (utilizationPercent / 100);

    // Current on-demand cost
    const odMonthlyCost = pricing.odPriceHourly * adjustedHoursPerMonth * instanceCount;

    // RI costs
    const ri1YearMonthlyCost = pricing.ri1YearNoUpfrontHourly * adjustedHoursPerMonth * instanceCount;
    const ri3YearMonthlyCost = pricing.ri3YearNoUpfrontHourly * adjustedHoursPerMonth * instanceCount;

    // Savings
    const savings1Year = {
      monthly: odMonthlyCost - ri1YearMonthlyCost,
      annual: (odMonthlyCost - ri1YearMonthlyCost) * 12,
      percent: ((odMonthlyCost - ri1YearMonthlyCost) / odMonthlyCost) * 100,
    };

    const savings3Year = {
      monthly: odMonthlyCost - ri3YearMonthlyCost,
      total: (odMonthlyCost - ri3YearMonthlyCost) * 36,
      percent: ((odMonthlyCost - ri3YearMonthlyCost) / odMonthlyCost) * 100,
    };

    // Breakeven for all-upfront 1-year RI
    const upfrontCost1Year = pricing.ri1YearAllUpfront * instanceCount;
    const monthlyODCost = pricing.odPriceHourly * hoursPerMonth * instanceCount;
    const monthlyRI1Year = (pricing.ri1YearAllUpfront / 12) * instanceCount;
    const monthlySavingsWithUpfront = monthlyODCost - monthlyRI1Year;
    const breakevenMonths = upfrontCost1Year / monthlySavingsWithUpfront;

    const breakevenDate = new Date();
    breakevenDate.setMonth(breakevenDate.getMonth() + Math.ceil(breakevenMonths));

    // Recommendation
    let recommendation: ROICalculation['recommendation'] = 'on-demand';
    if (utilizationPercent >= 60) {
      if (savings3Year.percent >= 40) {
        recommendation = 'ri-3-year';
      } else {
        recommendation = 'ri-1-year';
      }
    }

    return {
      instanceType,
      currentMonthlyCost: Math.round(odMonthlyCost * 100) / 100,
      ri1YearMonthlyCost: Math.round(ri1YearMonthlyCost * 100) / 100,
      ri3YearMonthlyCost: Math.round(ri3YearMonthlyCost * 100) / 100,
      savings1Year: {
        monthly: Math.round(savings1Year.monthly * 100) / 100,
        annual: Math.round(savings1Year.annual * 100) / 100,
        percent: Math.round(savings1Year.percent * 10) / 10,
      },
      savings3Year: {
        monthly: Math.round(savings3Year.monthly * 100) / 100,
        total: Math.round(savings3Year.total * 100) / 100,
        percent: Math.round(savings3Year.percent * 10) / 10,
      },
      breakeven1Year: {
        months: Math.ceil(breakevenMonths),
        date: breakevenDate.toISOString().split('T')[0],
      },
      recommendation,
      paybackPeriod: Math.ceil(breakevenMonths),
    };
  }

  analyzeFleet(fleet: Array<{ instanceType: string; count: number; utilization: number }>): {
    totalCurrentCost: number;
    totalOptimizedCost: number;
    totalSavings: number;
    recommendations: ROICalculation[];
  } {
    const recommendations = fleet.map(({ instanceType, count, utilization }) =>
      this.calculateROI(instanceType, count, utilization)
    );

    const totalCurrentCost = recommendations.reduce((sum, r) => sum + r.currentMonthlyCost, 0);
    const totalOptimizedCost = recommendations.reduce((sum, r) => {
      if (r.recommendation === 'ri-3-year') return sum + r.ri3YearMonthlyCost;
      if (r.recommendation === 'ri-1-year') return sum + r.ri1YearMonthlyCost;
      return sum + r.currentMonthlyCost;
    }, 0);

    return {
      totalCurrentCost: Math.round(totalCurrentCost * 100) / 100,
      totalOptimizedCost: Math.round(totalOptimizedCost * 100) / 100,
      totalSavings: Math.round((totalCurrentCost - totalOptimizedCost) * 100) / 100,
      recommendations,
    };
  }
}
```

---

## 5. Cost Attribution: Kubernetes Label-Based Cost Allocator

```typescript
// services/finops/src/cost-allocator.ts
import { KubeConfig, CoreV1Api, AppsV1Api } from '@kubernetes/client-node';
import { CostExplorerClient, GetCostAndUsageCommand } from '@aws-sdk/client-cost-explorer';

interface ServiceCostAllocation {
  team: string;
  service: string;
  environment: string;
  namespace: string;
  allocatedCostUSD: number;
  breakdown: {
    compute: number;
    memory: number;
    storage: number;
    network: number;
  };
  podCount: number;
  cpuRequested: number;
  memoryRequestedGB: number;
}

export class KubernetesCostAllocator {
  private k8sCore: CoreV1Api;
  private k8sApps: AppsV1Api;
  private costExplorer: CostExplorerClient;

  // Cost per resource unit per hour (EKS Fargate)
  private readonly COST_RATES = {
    vcpuPerHour: 0.04048,
    gbMemoryPerHour: 0.004445,
    gbStoragePerMonth: 0.10,
    gbNetworkPerMonth: 0.114,
  };

  constructor() {
    const kc = new KubeConfig();
    kc.loadFromDefault();
    this.k8sCore = kc.makeApiClient(CoreV1Api);
    this.k8sApps = kc.makeApiClient(AppsV1Api);
    this.costExplorer = new CostExplorerClient({ region: 'us-east-1' });
  }

  async allocateCosts(namespace?: string): Promise<ServiceCostAllocation[]> {
    // Get all pods
    const podsResponse = namespace
      ? await this.k8sCore.listNamespacedPod(namespace)
      : await this.k8sCore.listPodForAllNamespaces();

    const pods = podsResponse.body.items;
    const allocations = new Map<string, ServiceCostAllocation>();

    for (const pod of pods) {
      if (pod.status?.phase !== 'Running') continue;

      const labels = pod.metadata?.labels || {};
      const team = labels['team'] || labels['app.kubernetes.io/part-of'] || 'unknown';
      const service = labels['app'] || labels['app.kubernetes.io/name'] || 'unknown';
      const environment = labels['environment'] || labels['env'] || 'unknown';
      const podNamespace = pod.metadata?.namespace || 'default';

      const allocationKey = `${team}:${service}:${environment}`;

      if (!allocations.has(allocationKey)) {
        allocations.set(allocationKey, {
          team,
          service,
          environment,
          namespace: podNamespace,
          allocatedCostUSD: 0,
          breakdown: { compute: 0, memory: 0, storage: 0, network: 0 },
          podCount: 0,
          cpuRequested: 0,
          memoryRequestedGB: 0,
        });
      }

      const allocation = allocations.get(allocationKey)!;
      allocation.podCount++;

      // Sum up resource requests
      for (const container of pod.spec?.containers || []) {
        const cpuRequest = this.parseCPU(container.resources?.requests?.cpu || '100m');
        const memRequest = this.parseMemoryGB(container.resources?.requests?.memory || '128Mi');

        allocation.cpuRequested += cpuRequest;
        allocation.memoryRequestedGB += memRequest;

        // Calculate hourly cost
        const hourlyComputeCost = (cpuRequest / 1000) * this.COST_RATES.vcpuPerHour;
        const hourlyMemoryCost = memRequest * this.COST_RATES.gbMemoryPerHour;

        // Monthly estimate (720 hours)
        allocation.breakdown.compute += hourlyComputeCost * 720;
        allocation.breakdown.memory += hourlyMemoryCost * 720;
      }
    }

    // Calculate totals and add storage/network estimates
    return Array.from(allocations.values()).map(allocation => {
      // Add estimated storage (20GB per service average)
      allocation.breakdown.storage = 20 * this.COST_RATES.gbStoragePerMonth;

      // Add estimated network (10GB outbound per service average)
      allocation.breakdown.network = 10 * this.COST_RATES.gbNetworkPerMonth;

      allocation.allocatedCostUSD = Object.values(allocation.breakdown)
        .reduce((a, b) => a + b, 0);

      // Round values
      allocation.allocatedCostUSD = Math.round(allocation.allocatedCostUSD * 100) / 100;
      allocation.cpuRequested = Math.round(allocation.cpuRequested);
      allocation.memoryRequestedGB = Math.round(allocation.memoryRequestedGB * 100) / 100;

      return allocation;
    }).sort((a, b) => b.allocatedCostUSD - a.allocatedCostUSD);
  }

  generateCostReport(allocations: ServiceCostAllocation[]): string {
    const totalCost = allocations.reduce((sum, a) => sum + a.allocatedCostUSD, 0);
    const byTeam = allocations.reduce((acc, a) => {
      acc[a.team] = (acc[a.team] || 0) + a.allocatedCostUSD;
      return acc;
    }, {} as Record<string, number>);

    let report = `# Cost Allocation Report\n`;
    report += `Generated: ${new Date().toISOString()}\n\n`;
    report += `## Total Monthly Cost: $${totalCost.toFixed(2)}\n\n`;
    report += `## By Team\n`;

    Object.entries(byTeam)
      .sort((a, b) => b[1] - a[1])
      .forEach(([team, cost]) => {
        const percent = ((cost / totalCost) * 100).toFixed(1);
        report += `- **${team}**: $${cost.toFixed(2)} (${percent}%)\n`;
      });

    report += `\n## By Service\n`;
    report += `| Team | Service | Environment | Monthly Cost | Pods | CPU (cores) | Memory (GB) |\n`;
    report += `|------|---------|-------------|-------------|------|-------------|-------------|\n`;

    allocations.forEach(a => {
      report += `| ${a.team} | ${a.service} | ${a.environment} | $${a.allocatedCostUSD} | ${a.podCount} | ${(a.cpuRequested / 1000).toFixed(2)} | ${a.memoryRequestedGB} |\n`;
    });

    return report;
  }

  private parseCPU(cpu: string): number {
    if (cpu.endsWith('m')) return parseInt(cpu);
    return parseFloat(cpu) * 1000;
  }

  private parseMemoryGB(memory: string): number {
    if (memory.endsWith('Gi')) return parseFloat(memory);
    if (memory.endsWith('Mi')) return parseFloat(memory) / 1024;
    if (memory.endsWith('Ki')) return parseFloat(memory) / (1024 * 1024);
    return parseFloat(memory) / (1024 * 1024 * 1024);
  }
}
```

---

## 6. Database Cost Analyzer

```typescript
// services/finops/src/database-cost-analyzer.ts

interface DatabaseOption {
  name: string;
  type: 'rds' | 'aurora-serverless-v2';
  instanceClass?: string;
  minACU?: number;
  maxACU?: number;
  hourlyCost: number;
  storageGBMonth: number;
  ioPerMillion?: number;
}

interface CostAnalysis {
  scenario: string;
  monthlyCost: number;
  workload: string;
  recommendation: string;
  breakevenRPU: number; // Requests Per Unit (Aurora Serverless v2)
}

// ap-southeast-1 pricing
const DB_OPTIONS: DatabaseOption[] = [
  {
    name: 'RDS db.t3.medium',
    type: 'rds',
    instanceClass: 'db.t3.medium',
    hourlyCost: 0.068,
    storageGBMonth: 0.138,
  },
  {
    name: 'RDS db.t3.large',
    type: 'rds',
    instanceClass: 'db.t3.large',
    hourlyCost: 0.136,
    storageGBMonth: 0.138,
  },
  {
    name: 'RDS db.m5.large',
    type: 'rds',
    instanceClass: 'db.m5.large',
    hourlyCost: 0.214,
    storageGBMonth: 0.138,
  },
  {
    name: 'Aurora Serverless v2 (0.5-2 ACU)',
    type: 'aurora-serverless-v2',
    minACU: 0.5,
    maxACU: 2,
    hourlyCost: 0.12, // per ACU
    storageGBMonth: 0.12,
    ioPerMillion: 0.25,
  },
  {
    name: 'Aurora Serverless v2 (0.5-8 ACU)',
    type: 'aurora-serverless-v2',
    minACU: 0.5,
    maxACU: 8,
    hourlyCost: 0.12,
    storageGBMonth: 0.12,
    ioPerMillion: 0.25,
  },
];

export class DatabaseCostAnalyzer {
  analyzeBreakeven(
    workload: {
      peakHoursPerDay: number;
      avgConnectionsPerHour: number;
      storageGB: number;
      ioRequestsPerDay: number;
    }
  ): CostAnalysis[] {
    const analyses: CostAnalysis[] = [];

    for (const dbOption of DB_OPTIONS) {
      let monthlyCost = 0;

      if (dbOption.type === 'rds') {
        // RDS: fixed cost
        monthlyCost = dbOption.hourlyCost * 720;
        monthlyCost += workload.storageGB * dbOption.storageGBMonth;
      } else if (dbOption.type === 'aurora-serverless-v2') {
        // Aurora Serverless v2: cost based on actual ACU usage
        const peakHours = workload.peakHoursPerDay * 30;
        const offPeakHours = 720 - peakHours;

        // During peak: use maxACU
        const peakCost = peakHours * (dbOption.maxACU || 2) * dbOption.hourlyCost;

        // During off-peak: use minACU (0.5 ACU minimum)
        const offPeakCost = offPeakHours * (dbOption.minACU || 0.5) * dbOption.hourlyCost;

        // Storage + IO
        const storageCost = workload.storageGB * dbOption.storageGBMonth;
        const ioCost = (workload.ioRequestsPerDay * 30 / 1_000_000) * (dbOption.ioPerMillion || 0);

        monthlyCost = peakCost + offPeakCost + storageCost + ioCost;
      }

      const isHighUtilization = workload.peakHoursPerDay >= 18;
      let recommendation = '';

      if (dbOption.type === 'rds') {
        recommendation = isHighUtilization
          ? 'Good for predictable high-traffic workloads'
          : 'Consider Serverless for variable traffic';
      } else {
        recommendation = isHighUtilization
          ? 'More expensive than RDS at sustained high load'
          : 'Cost-effective for variable traffic patterns';
      }

      analyses.push({
        scenario: dbOption.name,
        monthlyCost: Math.round(monthlyCost * 100) / 100,
        workload: `${workload.peakHoursPerDay}h peak/day, ${workload.storageGB}GB storage`,
        recommendation,
        breakevenRPU: this.calculateBreakevenRPU(dbOption, workload),
      });
    }

    return analyses.sort((a, b) => a.monthlyCost - b.monthlyCost);
  }

  private calculateBreakevenRPU(
    option: DatabaseOption,
    workload: { peakHoursPerDay: number; storageGB: number }
  ): number {
    // Hours per month where Aurora Serverless == RDS cost
    if (option.type !== 'aurora-serverless-v2') return 0;

    const rdsBaseline = DB_OPTIONS.find(o => o.name === 'RDS db.t3.medium');
    if (!rdsBaseline) return 0;

    const rdsMonthlyCost = rdsBaseline.hourlyCost * 720;
    const auroraFixedCost = workload.storageGB * option.storageGBMonth;
    const costPerACUHour = option.hourlyCost;

    // Aurora breaks even with RDS when: auroraFixed + ACU * hours = rdsCost
    const breakEvenACUHours = (rdsMonthlyCost - auroraFixedCost) / costPerACUHour;
    return Math.round(breakEvenACUHours / 720 * 100) / 100; // avg ACU
  }
}
```

---

## 7. Storage Tiering: S3 Lifecycle Policy Generator

```typescript
// services/finops/src/s3-lifecycle-generator.ts
import {
  S3Client,
  PutBucketLifecycleConfigurationCommand,
  GetBucketLifecycleConfigurationCommand,
  LifecycleRule,
  TransitionStorageClass,
} from '@aws-sdk/client-s3';

interface StorageTierConfig {
  bucketName: string;
  prefix?: string;
  rules: Array<{
    id: string;
    description: string;
    daysToInfrequentAccess?: number;
    daysToGlacierInstantRetrieval?: number;
    daysToGlacierFlexible?: number;
    daysToDeepArchive?: number;
    daysToExpire?: number;
    applyToNonCurrentVersions?: boolean;
    abortIncompleteMultipartUploadDays?: number;
  }>;
}

export class S3LifecyclePolicyGenerator {
  private s3Client: S3Client;

  // Storage class pricing (ap-southeast-1, per GB/month)
  private readonly PRICING = {
    STANDARD: 0.025,
    STANDARD_IA: 0.0138,
    INTELLIGENT_TIERING: 0.025,
    ONEZONE_IA: 0.011,
    GLACIER_IR: 0.005,
    GLACIER: 0.004,
    DEEP_ARCHIVE: 0.00099,
  };

  constructor() {
    this.s3Client = new S3Client({ region: process.env.AWS_REGION });
  }

  generateVideoStoragePolicy(): StorageTierConfig {
    return {
      bucketName: process.env.S3_OUTPUT_BUCKET!,
      rules: [
        {
          id: 'hls-popular-content',
          description: 'Keep popular HLS content in Standard for 30 days, then IA',
          daysToInfrequentAccess: 30,
          daysToGlacierInstantRetrieval: 90,
          daysToExpire: 365,
          abortIncompleteMultipartUploadDays: 7,
        },
        {
          id: 'raw-video-archive',
          description: 'Archive raw uploaded videos quickly',
          daysToInfrequentAccess: 7,
          daysToGlacierFlexible: 30,
          daysToDeepArchive: 180,
          applyToNonCurrentVersions: true,
        },
        {
          id: 'thumbnails-tier',
          description: 'Move thumbnails to IA after 60 days',
          daysToInfrequentAccess: 60,
          daysToGlacierInstantRetrieval: 180,
        },
      ],
    };
  }

  async applyLifecyclePolicy(config: StorageTierConfig): Promise<void> {
    const rules: LifecycleRule[] = config.rules.map(rule => {
      const lifecycleRule: LifecycleRule = {
        ID: rule.id,
        Status: 'Enabled',
        Filter: config.prefix ? { Prefix: config.prefix } : {},
        Transitions: [],
        NoncurrentVersionTransitions: [],
      };

      // Add transitions
      if (rule.daysToInfrequentAccess) {
        lifecycleRule.Transitions!.push({
          Days: rule.daysToInfrequentAccess,
          StorageClass: 'STANDARD_IA' as TransitionStorageClass,
        });
      }
      if (rule.daysToGlacierInstantRetrieval) {
        lifecycleRule.Transitions!.push({
          Days: rule.daysToGlacierInstantRetrieval,
          StorageClass: 'GLACIER_IR' as TransitionStorageClass,
        });
      }
      if (rule.daysToGlacierFlexible) {
        lifecycleRule.Transitions!.push({
          Days: rule.daysToGlacierFlexible,
          StorageClass: 'GLACIER' as TransitionStorageClass,
        });
      }
      if (rule.daysToDeepArchive) {
        lifecycleRule.Transitions!.push({
          Days: rule.daysToDeepArchive,
          StorageClass: 'DEEP_ARCHIVE' as TransitionStorageClass,
        });
      }
      if (rule.daysToExpire) {
        lifecycleRule.Expiration = { Days: rule.daysToExpire };
      }
      if (rule.abortIncompleteMultipartUploadDays) {
        lifecycleRule.AbortIncompleteMultipartUpload = {
          DaysAfterInitiation: rule.abortIncompleteMultipartUploadDays,
        };
      }
      if (rule.applyToNonCurrentVersions) {
        lifecycleRule.NoncurrentVersionTransitions = [
          { NoncurrentDays: 1, StorageClass: 'GLACIER' as TransitionStorageClass },
        ];
        lifecycleRule.NoncurrentVersionExpiration = { NoncurrentDays: 90 };
      }

      return lifecycleRule;
    });

    await this.s3Client.send(
      new PutBucketLifecycleConfigurationCommand({
        Bucket: config.bucketName,
        LifecycleConfiguration: { Rules: rules },
      })
    );

    console.log(`Applied lifecycle policy to ${config.bucketName}: ${rules.length} rules`);
  }

  estimateMonthlySavings(
    storageGB: number,
    accessPattern: {
      hotPercent: number;   // % accessed in first 30 days
      warmPercent: number;  // % accessed 30-90 days
      coldPercent: number;  // % accessed 90+ days
    }
  ): { currentCost: number; optimizedCost: number; savings: number; savingsPercent: number } {
    const hotGB = storageGB * (accessPattern.hotPercent / 100);
    const warmGB = storageGB * (accessPattern.warmPercent / 100);
    const coldGB = storageGB * (accessPattern.coldPercent / 100);

    // Current: all in Standard
    const currentCost = storageGB * this.PRICING.STANDARD;

    // Optimized: tiered storage
    const optimizedCost =
      hotGB * this.PRICING.STANDARD +
      warmGB * this.PRICING.STANDARD_IA +
      coldGB * this.PRICING.GLACIER;

    const savings = currentCost - optimizedCost;
    const savingsPercent = (savings / currentCost) * 100;

    return {
      currentCost: Math.round(currentCost * 100) / 100,
      optimizedCost: Math.round(optimizedCost * 100) / 100,
      savings: Math.round(savings * 100) / 100,
      savingsPercent: Math.round(savingsPercent * 10) / 10,
    };
  }
}
```

---

## 8. Kubecost Integration: TypeScript API Client

```typescript
// services/finops/src/kubecost.client.ts
import axios, { AxiosInstance } from 'axios';

interface KubecostAllocation {
  name: string;
  properties: {
    cluster: string;
    node: string;
    namespace: string;
    controller: string;
    controllerKind: string;
    pod: string;
    container: string;
    providerID: string;
    labels: Record<string, string>;
  };
  window: { start: string; end: string };
  start: string;
  end: string;
  minutes: number;
  cpuCores: number;
  cpuCoreRequestAverage: number;
  cpuCoreUsageAverage: number;
  cpuCost: number;
  gpuCount: number;
  gpuCost: number;
  networkIngressBytes: number;
  networkEgressBytes: number;
  networkCost: number;
  loadBalancerCost: number;
  pvBytes: number;
  pvCost: number;
  ramBytes: number;
  ramByteRequestAverage: number;
  ramByteUsageAverage: number;
  ramCost: number;
  sharedCost: number;
  externalCost: number;
  totalCost: number;
  totalEfficiency: number;
}

export class KubecostClient {
  private http: AxiosInstance;

  constructor(kubecostUrl: string = 'http://kubecost-cost-analyzer:9090') {
    this.http = axios.create({
      baseURL: kubecostUrl,
      timeout: 30000,
    });
  }

  async getAllocations(params: {
    window: string; // e.g., '7d', '1M', 'lastweek'
    aggregate?: string; // 'namespace', 'label:team', 'controller'
    idle?: boolean;
    external?: boolean;
    shareNamespaces?: string;
    shareTenancyCosts?: boolean;
  }): Promise<KubecostAllocation[]> {
    const response = await this.http.get('/model/allocation', {
      params: {
        window: params.window,
        aggregate: params.aggregate || 'namespace',
        idle: params.idle ?? false,
        external: params.external ?? false,
        shareNamespaces: params.shareNamespaces,
        shareTenancyCosts: params.shareTenancyCosts ?? true,
        includeSharedCostBreakdown: true,
        reconcile: true,
        format: 'json',
      },
    });

    const data = response.data?.data?.[0] || {};
    return Object.values(data) as KubecostAllocation[];
  }

  async getNamespaceCosts(window: string = '30d'): Promise<Array<{
    namespace: string;
    totalCost: number;
    cpuCost: number;
    ramCost: number;
    storagesCost: number;
    efficiency: number;
  }>> {
    const allocations = await this.getAllocations({ window, aggregate: 'namespace' });

    return allocations.map(a => ({
      namespace: a.name,
      totalCost: Math.round(a.totalCost * 100) / 100,
      cpuCost: Math.round(a.cpuCost * 100) / 100,
      ramCost: Math.round(a.ramCost * 100) / 100,
      storagesCost: Math.round(a.pvCost * 100) / 100,
      efficiency: Math.round(a.totalEfficiency * 1000) / 10,
    })).sort((a, b) => b.totalCost - a.totalCost);
  }

  async getLabelCosts(
    label: string,
    window: string = '30d'
  ): Promise<Array<{ labelValue: string; totalCost: number }>> {
    const allocations = await this.getAllocations({
      window,
      aggregate: `label:${label}`,
    });

    return allocations
      .map(a => ({
        labelValue: a.name,
        totalCost: Math.round(a.totalCost * 100) / 100,
      }))
      .sort((a, b) => b.totalCost - a.totalCost);
  }

  async getCostEfficiencyReport(window: string = '7d'): Promise<{
    overprovisioned: Array<{ namespace: string; wastedCost: number; efficiency: number }>;
    underprovisioned: Array<{ namespace: string; namespace2: string }>;
    totalWastedCost: number;
  }> {
    const allocations = await this.getAllocations({ window, aggregate: 'namespace' });

    const overprovisioned = allocations
      .filter(a => a.totalEfficiency < 0.5 && a.totalCost > 10)
      .map(a => ({
        namespace: a.name,
        wastedCost: Math.round((a.totalCost * (1 - a.totalEfficiency)) * 100) / 100,
        efficiency: Math.round(a.totalEfficiency * 1000) / 10,
      }))
      .sort((a, b) => b.wastedCost - a.wastedCost);

    const totalWastedCost = overprovisioned.reduce((sum, a) => sum + a.wastedCost, 0);

    return {
      overprovisioned,
      underprovisioned: [],
      totalWastedCost: Math.round(totalWastedCost * 100) / 100,
    };
  }
}
```

---

## 9. Monthly Cost Report Generator

```typescript
// services/finops/src/monthly-report-generator.ts
import { CostDashboardService } from './cost-dashboard.service';
import { KubecostClient } from './kubecost.client';
import { ContainerRightsizingAnalyzer } from './container-rightsizing';
import { S3LifecyclePolicyGenerator } from './s3-lifecycle-generator';
import { ReservedCapacityCalculator } from './reserved-capacity-calculator';

interface MonthlyReport {
  period: string;
  generatedAt: string;
  totalMonthlyCost: number;
  topServices: Array<{ name: string; cost: number; percentOfTotal: number }>;
  recommendations: Array<{
    category: string;
    description: string;
    estimatedSavings: number;
    effort: 'low' | 'medium' | 'high';
    priority: number;
  }>;
  costTrend: {
    previousMonth: number;
    currentMonth: number;
    changePercent: number;
  };
  teamBreakdown: Array<{ team: string; cost: number; services: number }>;
  kpiMetrics: {
    costPerRequest: number;
    costPerUser: number;
    costPerGB: number;
  };
}

export class MonthlyReportGenerator {
  private costDashboard: CostDashboardService;
  private kubecost: KubecostClient;
  private rightsizing: ContainerRightsizingAnalyzer;
  private s3Lifecycle: S3LifecyclePolicyGenerator;
  private riCalculator: ReservedCapacityCalculator;

  constructor() {
    this.costDashboard = new CostDashboardService();
    this.kubecost = new KubecostClient();
    this.rightsizing = new ContainerRightsizingAnalyzer();
    this.s3Lifecycle = new S3LifecyclePolicyGenerator();
    this.riCalculator = new ReservedCapacityCalculator();
  }

  async generateReport(month?: string): Promise<MonthlyReport> {
    const targetMonth = month || new Date().toISOString().substring(0, 7);
    const prevMonth = this.getPreviousMonth(targetMonth);

    console.log(`Generating cost report for ${targetMonth}...`);

    const [currentCost, previousCost, namespaceCosts] = await Promise.all([
      this.costDashboard.getMonthlyCostReport(targetMonth),
      this.costDashboard.getMonthlyCostReport(prevMonth),
      this.kubecost.getNamespaceCosts('30d').catch(() => []),
    ]);

    const changePercent = previousCost.totalCost > 0
      ? ((currentCost.totalCost - previousCost.totalCost) / previousCost.totalCost) * 100
      : 0;

    // Generate recommendations
    const recommendations = this.generateRecommendations(currentCost, namespaceCosts);

    // Calculate KPIs (using placeholder values - integrate with metrics)
    const kpiMetrics = {
      costPerRequest: Math.round((currentCost.totalCost / 1000000) * 100) / 100, // placeholder
      costPerUser: Math.round((currentCost.totalCost / 10000) * 100) / 100, // placeholder
      costPerGB: Math.round((currentCost.totalCost / 500) * 100) / 100, // placeholder
    };

    const topServices = currentCost.byService
      .slice(0, 10)
      .map(s => ({
        name: s.service,
        cost: s.amount,
        percentOfTotal: Math.round((s.amount / currentCost.totalCost) * 1000) / 10,
      }));

    // Team breakdown from tags
    const teamBreakdown = Object.entries(currentCost.byTag)
      .map(([team, cost]) => ({
        team,
        cost: Math.round(cost * 100) / 100,
        services: namespaceCosts.filter(n => n.namespace.includes(team)).length,
      }))
      .sort((a, b) => b.cost - a.cost);

    const report: MonthlyReport = {
      period: targetMonth,
      generatedAt: new Date().toISOString(),
      totalMonthlyCost: currentCost.totalCost,
      topServices,
      recommendations,
      costTrend: {
        previousMonth: previousCost.totalCost,
        currentMonth: currentCost.totalCost,
        changePercent: Math.round(changePercent * 10) / 10,
      },
      teamBreakdown,
      kpiMetrics,
    };

    return report;
  }

  private generateRecommendations(
    costReport: any,
    namespaceCosts: any[]
  ): MonthlyReport['recommendations'] {
    const recommendations: MonthlyReport['recommendations'] = [];

    // 1. Reserved Instances
    const ec2Cost = costReport.byService.find((s: any) => s.service === 'Amazon EC2');
    if (ec2Cost && ec2Cost.amount > 500) {
      recommendations.push({
        category: 'Reserved Instances',
        description: 'Purchase 1-year Reserved Instances for predictable EC2 workloads. Estimated 30-40% savings.',
        estimatedSavings: Math.round(ec2Cost.amount * 0.35 * 100) / 100,
        effort: 'low',
        priority: 1,
      });
    }

    // 2. S3 Lifecycle
    const s3Cost = costReport.byService.find((s: any) => s.service === 'Amazon S3');
    if (s3Cost && s3Cost.amount > 200) {
      const storageSavings = this.s3Lifecycle.estimateMonthlySavings(
        10000, // 10TB estimated
        { hotPercent: 20, warmPercent: 30, coldPercent: 50 }
      );
      recommendations.push({
        category: 'Storage Tiering',
        description: `Implement S3 lifecycle policies to move infrequent data to cheaper storage classes. ${storageSavings.savingsPercent}% estimated savings.`,
        estimatedSavings: storageSavings.savings,
        effort: 'low',
        priority: 2,
      });
    }

    // 3. Container right-sizing
    const inefficientNamespaces = namespaceCosts.filter(n => n.efficiency < 50);
    if (inefficientNamespaces.length > 0) {
      const wastedCost = inefficientNamespaces.reduce(
        (sum, n) => sum + n.totalCost * (1 - n.efficiency / 100), 0
      );
      recommendations.push({
        category: 'Container Right-sizing',
        description: `${inefficientNamespaces.length} namespaces have <50% efficiency. Right-size container requests/limits.`,
        estimatedSavings: Math.round(wastedCost * 0.5 * 100) / 100,
        effort: 'medium',
        priority: 3,
      });
    }

    // 4. Spot instances
    recommendations.push({
      category: 'Spot Instances',
      description: 'Move stateless services to Spot instances with graceful termination. 60-80% savings on compute.',
      estimatedSavings: Math.round((costReport.totalCost * 0.3) * 100) / 100,
      effort: 'high',
      priority: 4,
    });

    // 5. Database optimization
    const rdsCost = costReport.byService.find((s: any) => s.service?.includes('RDS'));
    if (rdsCost && rdsCost.amount > 300) {
      recommendations.push({
        category: 'Database Optimization',
        description: 'Evaluate Aurora Serverless v2 for variable workloads. Enable storage auto-scaling.',
        estimatedSavings: Math.round(rdsCost.amount * 0.2 * 100) / 100,
        effort: 'medium',
        priority: 5,
      });
    }

    return recommendations.sort((a, b) => a.priority - b.priority);
  }

  formatReportAsMarkdown(report: MonthlyReport): string {
    const totalSavings = report.recommendations
      .reduce((sum, r) => sum + r.estimatedSavings, 0);

    let md = `# Monthly Cost Report - ${report.period}\n\n`;
    md += `**Generated:** ${new Date(report.generatedAt).toLocaleString('th-TH')}\n\n`;

    md += `## สรุปภาพรวม\n\n`;
    md += `| Metric | Value |\n|--------|-------|\n`;
    md += `| Total Monthly Cost | $${report.totalMonthlyCost.toFixed(2)} |\n`;
    md += `| vs Previous Month | ${report.costTrend.changePercent > 0 ? '+' : ''}${report.costTrend.changePercent}% |\n`;
    md += `| Potential Savings | $${totalSavings.toFixed(2)}/month |\n\n`;

    md += `## Top Services by Cost\n\n`;
    md += `| Service | Cost | % of Total |\n|---------|------|------------|\n`;
    report.topServices.forEach(s => {
      md += `| ${s.name} | $${s.cost.toFixed(2)} | ${s.percentOfTotal}% |\n`;
    });

    md += `\n## Recommendations\n\n`;
    report.recommendations.forEach((rec, i) => {
      md += `### ${i + 1}. ${rec.category} (Effort: ${rec.effort})\n`;
      md += `${rec.description}\n\n`;
      md += `**Estimated Savings:** $${rec.estimatedSavings.toFixed(2)}/month\n\n`;
    });

    return md;
  }

  private getPreviousMonth(month: string): string {
    const date = new Date(month + '-01');
    date.setMonth(date.getMonth() - 1);
    return date.toISOString().substring(0, 7);
  }
}
```

---

## สรุป

| หัวข้อ | Tool/Technique | Potential Savings |
|--------|---------------|------------------|
| Cost Dashboard | AWS Cost Explorer API | ความเข้าใจ Cost Drivers |
| Container Right-sizing | CloudWatch + K8s Metrics | 20-40% compute cost |
| Spot Instances | SIGTERM Handler + Draining | 60-80% EC2 cost |
| Reserved Instances | 1-year RI Calculator | 30-40% on predictable workloads |
| Cost Attribution | K8s Label-based Allocation | Accountability per team |
| Database Optimization | Aurora Serverless v2 Analysis | 20-50% for variable workloads |
| Storage Tiering | S3 Lifecycle Policies | 40-70% storage cost |
| FinOps Culture | Monthly Reports + KPIs | 30%+ overall reduction |

> "FinOps is a team sport — engineering, finance, and product must work together to optimize cloud spending without sacrificing velocity"

---

*ถัดไป: Part 99 - Final Project: Complete Microservices System*

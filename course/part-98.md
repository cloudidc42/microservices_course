# Part 98: Cost Optimization Strategies สำหรับ Microservices

## บทนำ

ค่าใช้จ่าย cloud infrastructure เป็นหนึ่งในความกังวลหลักของทีม engineering ที่ดูแลระบบ microservices ในระดับ production การ optimize cost โดยไม่กระทบ reliability และ performance เป็นทั้งศาสตร์และศิลป์ ในบทนี้เราจะเรียนรู้เครื่องมือและ techniques ที่ใช้งานได้จริงสำหรับการลดค่าใช้จ่ายของ microservices infrastructure

---

## 98.1 ทำความเข้าใจ Cloud Cost Structure

ค่าใช้จ่าย cloud ของ microservices แบ่งออกเป็นหมวดหลัก:

| หมวด | ส่วนประกอบ | % ต้นทุน (ทั่วไป) |
|------|-----------|-----------------|
| Compute | EC2, EKS nodes, Lambda | 40-60% |
| Storage | S3, EBS, RDS storage | 15-25% |
| Database | RDS, Aurora, DynamoDB | 15-25% |
| Network | Data transfer, NAT, Load Balancers | 10-20% |
| Other | CloudWatch, API Gateway, etc. | 5-10% |

### หลักการ Cost Optimization

1. **Right-sizing**: ใช้ทรัพยากรให้พอดี ไม่ over-provision
2. **Reserved capacity**: จอง capacity ล่วงหน้าสำหรับ predictable workloads
3. **Spot/Preemptible**: ใช้ spot instances สำหรับ fault-tolerant workloads
4. **Waste elimination**: หยุด resources ที่ไม่ได้ใช้
5. **Architecture optimization**: ออกแบบให้ใช้ทรัพยากรน้อยลง

---

## 98.2 TypeScript CostDashboard พร้อม AWS Cost Explorer SDK

```typescript
// cost-dashboard.ts
import {
  CostExplorerClient,
  GetCostAndUsageCommand,
  GetCostForecastCommand,
  GetDimensionValuesCommand,
  GetTagsCommand,
  GetCostAndUsageCommandInput,
  Expression,
} from '@aws-sdk/client-cost-explorer';

export interface DailyCost {
  date: string;
  amount: number;
  currency: string;
  services: Record<string, number>;
}

export interface ServiceCost {
  service: string;
  currentMonthCost: number;
  previousMonthCost: number;
  changePercent: number;
  forecast30Days: number;
}

export interface CostReport {
  period: { start: string; end: string };
  totalCost: number;
  dailyCosts: DailyCost[];
  topServices: ServiceCost[];
  byTag: Record<string, number>;
  forecastEndOfMonth: number;
  savings: {
    potentialSavings: number;
    rightsizingOpportunities: number;
    unusedResources: number;
  };
}

export class CostDashboard {
  private client: CostExplorerClient;

  constructor(region: string = 'us-east-1') {
    this.client = new CostExplorerClient({ region });
  }

  async getMonthlyReport(year: number, month: number): Promise<CostReport> {
    const startDate = `${year}-${String(month).padStart(2, '0')}-01`;
    const endDate = this.getEndOfMonth(year, month);

    const [dailyCosts, serviceBreakdown, tagBreakdown, forecast] = await Promise.all([
      this.getDailyCosts(startDate, endDate),
      this.getCostByService(startDate, endDate),
      this.getCostByTag('Team', startDate, endDate),
      this.getForecastEndOfMonth(year, month),
    ]);

    const totalCost = dailyCosts.reduce((sum, day) => sum + day.amount, 0);
    const topServices = await this.enrichWithPreviousMonth(serviceBreakdown, year, month);

    return {
      period: { start: startDate, end: endDate },
      totalCost,
      dailyCosts,
      topServices,
      byTag: tagBreakdown,
      forecastEndOfMonth: forecast,
      savings: await this.estimateSavings(),
    };
  }

  async getDailyCosts(startDate: string, endDate: string): Promise<DailyCost[]> {
    const params: GetCostAndUsageCommandInput = {
      TimePeriod: { Start: startDate, End: endDate },
      Granularity: 'DAILY',
      Metrics: ['UnblendedCost'],
      GroupBy: [
        {
          Type: 'DIMENSION',
          Key: 'SERVICE',
        },
      ],
    };

    const command = new GetCostAndUsageCommand(params);
    const response = await this.client.send(command);

    return (response.ResultsByTime ?? []).map((result) => {
      const services: Record<string, number> = {};
      let totalAmount = 0;

      (result.Groups ?? []).forEach((group) => {
        const serviceName = group.Keys?.[0] ?? 'Unknown';
        const amount = parseFloat(group.Metrics?.UnblendedCost?.Amount ?? '0');
        services[serviceName] = amount;
        totalAmount += amount;
      });

      return {
        date: result.TimePeriod?.Start ?? '',
        amount: totalAmount,
        currency: 'USD',
        services,
      };
    });
  }

  async getCostByService(startDate: string, endDate: string): Promise<Array<{
    service: string;
    cost: number;
  }>> {
    const command = new GetCostAndUsageCommand({
      TimePeriod: { Start: startDate, End: endDate },
      Granularity: 'MONTHLY',
      Metrics: ['UnblendedCost'],
      GroupBy: [{ Type: 'DIMENSION', Key: 'SERVICE' }],
    });

    const response = await this.client.send(command);
    const results: Array<{ service: string; cost: number }> = [];

    (response.ResultsByTime?.[0]?.Groups ?? []).forEach((group) => {
      const service = group.Keys?.[0] ?? 'Unknown';
      const cost = parseFloat(group.Metrics?.UnblendedCost?.Amount ?? '0');
      if (cost > 0) {
        results.push({ service, cost });
      }
    });

    return results.sort((a, b) => b.cost - a.cost);
  }

  async getCostByTag(
    tagKey: string,
    startDate: string,
    endDate: string
  ): Promise<Record<string, number>> {
    const command = new GetCostAndUsageCommand({
      TimePeriod: { Start: startDate, End: endDate },
      Granularity: 'MONTHLY',
      Metrics: ['UnblendedCost'],
      GroupBy: [{ Type: 'TAG', Key: tagKey }],
    });

    const response = await this.client.send(command);
    const result: Record<string, number> = {};

    (response.ResultsByTime?.[0]?.Groups ?? []).forEach((group) => {
      const tagValue = group.Keys?.[0]?.replace(`${tagKey}$`, '') ?? 'untagged';
      const cost = parseFloat(group.Metrics?.UnblendedCost?.Amount ?? '0');
      result[tagValue] = cost;
    });

    return result;
  }

  async getForecastEndOfMonth(year: number, month: number): Promise<number> {
    const today = new Date().toISOString().split('T')[0];
    const endDate = this.getEndOfMonth(year, month);

    if (today >= endDate) return 0; // Month already ended

    const command = new GetCostForecastCommand({
      TimePeriod: { Start: today, End: endDate },
      Metric: 'UNBLENDED_COST',
      Granularity: 'MONTHLY',
    });

    try {
      const response = await this.client.send(command);
      return parseFloat(response.Total?.Amount ?? '0');
    } catch {
      return 0;
    }
  }

  private async enrichWithPreviousMonth(
    currentServices: Array<{ service: string; cost: number }>,
    year: number,
    month: number
  ): Promise<ServiceCost[]> {
    const prevMonth = month === 1 ? 12 : month - 1;
    const prevYear = month === 1 ? year - 1 : year;
    const prevStart = `${prevYear}-${String(prevMonth).padStart(2, '0')}-01`;
    const prevEnd = this.getEndOfMonth(prevYear, prevMonth);

    const prevServices = await this.getCostByService(prevStart, prevEnd);
    const prevMap = new Map(prevServices.map((s) => [s.service, s.cost]));

    return currentServices.slice(0, 10).map((s) => {
      const prevCost = prevMap.get(s.service) ?? 0;
      const changePercent = prevCost > 0 ? ((s.cost - prevCost) / prevCost) * 100 : 0;

      return {
        service: s.service,
        currentMonthCost: s.cost,
        previousMonthCost: prevCost,
        changePercent,
        forecast30Days: s.cost * (30 / new Date().getDate()),
      };
    });
  }

  private async estimateSavings(): Promise<CostReport['savings']> {
    // In a real implementation, this would query AWS Cost Optimization Hub
    // or AWS Trusted Advisor
    return {
      potentialSavings: 0,
      rightsizingOpportunities: 0,
      unusedResources: 0,
    };
  }

  private getEndOfMonth(year: number, month: number): string {
    const endDate = new Date(year, month, 0); // Day 0 = last day of previous month
    return endDate.toISOString().split('T')[0];
  }

  generateHTMLReport(report: CostReport): string {
    const serviceRows = report.topServices
      .map(
        (s) =>
          `<tr>
            <td>${s.service}</td>
            <td>$${s.currentMonthCost.toFixed(2)}</td>
            <td>$${s.previousMonthCost.toFixed(2)}</td>
            <td style="color:${s.changePercent > 10 ? 'red' : s.changePercent < -5 ? 'green' : 'black'}">
              ${s.changePercent > 0 ? '+' : ''}${s.changePercent.toFixed(1)}%
            </td>
          </tr>`
      )
      .join('');

    return `<!DOCTYPE html>
<html>
<head><title>AWS Cost Report ${report.period.start}</title></head>
<body>
  <h1>AWS Cost Report</h1>
  <p>Period: ${report.period.start} to ${report.period.end}</p>
  <h2>Total Cost: $${report.totalCost.toFixed(2)}</h2>
  <h2>Forecast (End of Month): $${report.forecastEndOfMonth.toFixed(2)}</h2>
  <h3>Top Services</h3>
  <table border="1">
    <tr><th>Service</th><th>This Month</th><th>Last Month</th><th>Change</th></tr>
    ${serviceRows}
  </table>
  <h3>Cost by Team</h3>
  ${Object.entries(report.byTag)
    .map(([team, cost]) => `<p>${team}: $${cost.toFixed(2)}</p>`)
    .join('')}
</body>
</html>`;
  }
}
```

---

## 98.3 Container Right-Sizing Analyzer

การ right-size container เป็นหนึ่งในวิธีที่มีประสิทธิภาพสูงสุดในการลดค่าใช้จ่าย:

```typescript
// container-rightsizing-analyzer.ts
import * as k8s from '@kubernetes/client-node';
import axios from 'axios';

export interface VPARecommendation {
  containerName: string;
  targetCPU: string;
  targetMemory: string;
  lowerBoundCPU: string;
  lowerBoundMemory: string;
  upperBoundCPU: string;
  upperBoundMemory: string;
}

export interface ContainerActualUsage {
  containerName: string;
  avgCPU: number; // millicores
  p95CPU: number;
  p99CPU: number;
  maxCPU: number;
  avgMemory: number; // MiB
  p95Memory: number;
  p99Memory: number;
  maxMemory: number;
}

export interface ContainerCurrentConfig {
  containerName: string;
  requestCPU: number; // millicores
  requestMemory: number; // MiB
  limitCPU: number;
  limitMemory: number;
}

export interface RightsizingRecommendation {
  namespace: string;
  deploymentName: string;
  containerName: string;
  currentConfig: ContainerCurrentConfig;
  actualUsage: ContainerActualUsage;
  vpaRecommendation?: VPARecommendation;
  recommendedRequests: { cpu: string; memory: string };
  recommendedLimits: { cpu: string; memory: string };
  estimatedSavingsPercent: number;
  estimatedMonthlySavingsUSD: number;
  risk: 'low' | 'medium' | 'high';
  reasoning: string;
}

export class ContainerRightsizingAnalyzer {
  private k8sAppsApi: k8s.AppsV1Api;
  private k8sCoreApi: k8s.CoreV1Api;
  private prometheusUrl: string;

  // AWS pricing approximations
  private readonly CPU_COST_PER_MCORE_PER_MONTH = 0.0000012; // ~$0.04/vCPU/hour
  private readonly MEMORY_COST_PER_MIB_PER_MONTH = 0.00000065; // ~$0.005/GB/hour

  constructor(prometheusUrl: string) {
    const kc = new k8s.KubeConfig();
    kc.loadFromDefault();
    this.k8sAppsApi = kc.makeApiClient(k8s.AppsV1Api);
    this.k8sCoreApi = kc.makeApiClient(k8s.CoreV1Api);
    this.prometheusUrl = prometheusUrl;
  }

  async analyzeNamespace(namespace: string): Promise<RightsizingRecommendation[]> {
    const deployments = await this.k8sAppsApi.listNamespacedDeployment(namespace);
    const recommendations: RightsizingRecommendation[] = [];

    for (const deployment of deployments.body.items) {
      const name = deployment.metadata?.name ?? '';
      const containers = deployment.spec?.template?.spec?.containers ?? [];

      for (const container of containers) {
        const containerName = container.name;
        const currentConfig = this.extractCurrentConfig(container);
        const actualUsage = await this.getActualUsage(namespace, name, containerName);
        const vpaRec = await this.getVPARecommendation(namespace, name, containerName);

        const rec = this.generateRecommendation(
          namespace,
          name,
          containerName,
          currentConfig,
          actualUsage,
          vpaRec
        );

        if (rec.estimatedSavingsPercent > 10) {
          recommendations.push(rec);
        }
      }
    }

    return recommendations.sort((a, b) => b.estimatedMonthlySavingsUSD - a.estimatedMonthlySavingsUSD);
  }

  private extractCurrentConfig(container: k8s.V1Container): ContainerCurrentConfig {
    const parseCPU = (value?: string): number => {
      if (!value) return 1000; // default 1 vCPU
      if (value.endsWith('m')) return parseInt(value);
      return parseFloat(value) * 1000;
    };

    const parseMemory = (value?: string): number => {
      if (!value) return 512; // default 512 MiB
      if (value.endsWith('Mi')) return parseInt(value);
      if (value.endsWith('Gi')) return parseFloat(value) * 1024;
      if (value.endsWith('Ki')) return parseInt(value) / 1024;
      return parseInt(value) / (1024 * 1024);
    };

    return {
      containerName: container.name,
      requestCPU: parseCPU(container.resources?.requests?.['cpu']),
      requestMemory: parseMemory(container.resources?.requests?.['memory']),
      limitCPU: parseCPU(container.resources?.limits?.['cpu']),
      limitMemory: parseMemory(container.resources?.limits?.['memory']),
    };
  }

  private async getActualUsage(
    namespace: string,
    deployment: string,
    container: string,
    days = 14
  ): Promise<ContainerActualUsage> {
    const range = `${days}d`;
    const selector = `namespace="${namespace}",pod=~"${deployment}-.*",container="${container}"`;

    const queries = {
      avgCPU: `avg(avg_over_time(container_cpu_usage_seconds_total:rate5m{${selector}}[${range}])) * 1000`,
      p95CPU: `avg(quantile_over_time(0.95, container_cpu_usage_seconds_total:rate5m{${selector}}[${range}])) * 1000`,
      p99CPU: `avg(quantile_over_time(0.99, container_cpu_usage_seconds_total:rate5m{${selector}}[${range}])) * 1000`,
      maxCPU: `max(max_over_time(container_cpu_usage_seconds_total:rate5m{${selector}}[${range}])) * 1000`,
      avgMemory: `avg(avg_over_time(container_memory_working_set_bytes{${selector}}[${range}])) / 1024 / 1024`,
      p95Memory: `avg(quantile_over_time(0.95, container_memory_working_set_bytes{${selector}}[${range}])) / 1024 / 1024`,
      p99Memory: `avg(quantile_over_time(0.99, container_memory_working_set_bytes{${selector}}[${range}])) / 1024 / 1024`,
      maxMemory: `max(max_over_time(container_memory_working_set_bytes{${selector}}[${range}])) / 1024 / 1024`,
    };

    const results = await Promise.all(
      Object.entries(queries).map(async ([key, query]) => {
        try {
          const response = await axios.get(`${this.prometheusUrl}/api/v1/query`, {
            params: { query },
          });
          const value = parseFloat(response.data?.data?.result?.[0]?.value?.[1] ?? '0');
          return [key, value];
        } catch {
          return [key, 0];
        }
      })
    );

    return Object.fromEntries([['containerName', container], ...results]) as ContainerActualUsage;
  }

  private async getVPARecommendation(
    namespace: string,
    deployment: string,
    container: string
  ): Promise<VPARecommendation | undefined> {
    try {
      const kc = new k8s.KubeConfig();
      kc.loadFromDefault();
      const customApi = kc.makeApiClient(k8s.CustomObjectsApi);

      const vpas = await customApi.listNamespacedCustomObject(
        'autoscaling.k8s.io',
        'v1',
        namespace,
        'verticalpodautoscalers',
      ) as { body: { items: Array<Record<string, unknown>> } };

      const vpa = vpas.body.items.find((v) => {
        const targetRef = (v as Record<string, Record<string, Record<string, string>>>).spec?.targetRef;
        return targetRef?.name === deployment && targetRef?.kind === 'Deployment';
      });

      if (!vpa) return undefined;

      const status = (vpa as Record<string, Record<string, { containerRecommendations: Array<Record<string, unknown>> }>>).status;
      const containerRec = status?.recommendation?.containerRecommendations?.find(
        (c) => (c as Record<string, string>).containerName === container
      );

      if (!containerRec) return undefined;

      const target = containerRec.target as Record<string, string> | undefined;
      const lower = containerRec.lowerBound as Record<string, string> | undefined;
      const upper = containerRec.upperBound as Record<string, string> | undefined;

      return {
        containerName: container,
        targetCPU: target?.['cpu'] ?? '0',
        targetMemory: target?.['memory'] ?? '0',
        lowerBoundCPU: lower?.['cpu'] ?? '0',
        lowerBoundMemory: lower?.['memory'] ?? '0',
        upperBoundCPU: upper?.['cpu'] ?? '0',
        upperBoundMemory: upper?.['memory'] ?? '0',
      };
    } catch {
      return undefined;
    }
  }

  private generateRecommendation(
    namespace: string,
    deployment: string,
    container: string,
    current: ContainerCurrentConfig,
    actual: ContainerActualUsage,
    vpa?: VPARecommendation
  ): RightsizingRecommendation {
    // Use 20% headroom above p95 for requests
    const recCPURequest = Math.round(actual.p95CPU * 1.2);
    const recMemoryRequest = Math.round(actual.p95Memory * 1.2);

    // Use p99 * 1.5 for limits, or 2x requests
    const recCPULimit = Math.round(Math.max(actual.p99CPU * 1.5, recCPURequest * 2));
    const recMemoryLimit = Math.round(Math.max(actual.p99Memory * 1.5, recMemoryRequest * 1.5));

    // Calculate savings
    const currentMonthlyCPUCost = current.requestCPU * this.CPU_COST_PER_MCORE_PER_MONTH * 730;
    const currentMonthlyMemoryCost = current.requestMemory * this.MEMORY_COST_PER_MIB_PER_MONTH * 730;
    const currentMonthlyCost = currentMonthlyCPUCost + currentMonthlyMemoryCost;

    const recMonthlyCPUCost = recCPURequest * this.CPU_COST_PER_MCORE_PER_MONTH * 730;
    const recMonthlyMemoryCost = recMemoryRequest * this.MEMORY_COST_PER_MIB_PER_MONTH * 730;
    const recMonthlyCost = recMonthlyCPUCost + recMonthlyMemoryCost;

    const savingsPercent = currentMonthlyCost > 0
      ? ((currentMonthlyCost - recMonthlyCost) / currentMonthlyCost) * 100
      : 0;

    const risk = savingsPercent > 50 ? 'high' : savingsPercent > 25 ? 'medium' : 'low';

    return {
      namespace,
      deploymentName: deployment,
      containerName: container,
      currentConfig: current,
      actualUsage: actual,
      vpaRecommendation: vpa,
      recommendedRequests: {
        cpu: `${recCPURequest}m`,
        memory: `${recMemoryRequest}Mi`,
      },
      recommendedLimits: {
        cpu: `${recCPULimit}m`,
        memory: `${recMemoryLimit}Mi`,
      },
      estimatedSavingsPercent: Math.round(savingsPercent),
      estimatedMonthlySavingsUSD: Math.round((currentMonthlyCost - recMonthlyCost) * 100) / 100,
      risk,
      reasoning: `CPU: using ${actual.avgCPU.toFixed(0)}m avg (${actual.p95CPU.toFixed(0)}m p95) vs ${current.requestCPU}m requested. Memory: using ${actual.avgMemory.toFixed(0)}Mi avg vs ${current.requestMemory}Mi requested.`,
    };
  }
}
```

---

## 98.4 Spot Instance Handler พร้อม Graceful Termination

```typescript
// spot-instance-handler.ts
import { createClient } from 'redis';
import axios from 'axios';
import { EventEmitter } from 'events';

export interface CheckpointData {
  workerId: string;
  jobId: string;
  progress: number; // 0-100
  lastProcessedItem: string;
  state: Record<string, unknown>;
  timestamp: number;
}

export interface SpotTerminationEvent {
  instance_id: string;
  instance_action: 'terminate' | 'stop' | 'hibernate';
  time: string;
}

export class SpotInstanceHandler extends EventEmitter {
  private redis: ReturnType<typeof createClient>;
  private isTerminating: boolean = false;
  private readonly workerId: string;
  private checkpointKey: string;
  private pollingInterval?: ReturnType<typeof setInterval>;

  constructor(redisUrl: string, workerId: string) {
    super();
    this.workerId = workerId;
    this.checkpointKey = `checkpoint:${workerId}`;
    this.redis = createClient({ url: redisUrl });
    this.setupHandlers();
  }

  async initialize(): Promise<void> {
    await this.redis.connect();
    this.startTerminationPolling();
    console.log(`[SpotHandler ${this.workerId}] Initialized`);
  }

  private setupHandlers(): void {
    // Handle SIGTERM from Kubernetes or spot interruption
    process.on('SIGTERM', async () => {
      console.log(`[SpotHandler ${this.workerId}] SIGTERM received - beginning graceful shutdown`);
      await this.handleTermination('SIGTERM');
    });

    // Handle SIGINT for local testing
    process.on('SIGINT', async () => {
      console.log(`[SpotHandler ${this.workerId}] SIGINT received`);
      await this.handleTermination('SIGINT');
    });
  }

  private startTerminationPolling(): void {
    // AWS provides a 2-minute warning via instance metadata
    this.pollingInterval = setInterval(async () => {
      const terminationNotice = await this.checkSpotTerminationNotice();
      if (terminationNotice) {
        console.log(
          `[SpotHandler ${this.workerId}] Spot termination notice received:`,
          terminationNotice
        );
        this.emit('termination-notice', terminationNotice);
        await this.handleTermination('spot-notice');
      }
    }, 5000); // Check every 5 seconds
  }

  private async checkSpotTerminationNotice(): Promise<SpotTerminationEvent | null> {
    try {
      // First get the token for IMDSv2
      const tokenResponse = await axios.put(
        'http://169.254.169.254/latest/api/token',
        null,
        {
          headers: { 'X-aws-ec2-metadata-token-ttl-seconds': '21600' },
          timeout: 1000,
        }
      );

      const token = tokenResponse.data;
      const response = await axios.get(
        'http://169.254.169.254/latest/meta-data/spot/instance-action',
        {
          headers: { 'X-aws-ec2-metadata-token': token },
          timeout: 1000,
        }
      );

      return response.data as SpotTerminationEvent;
    } catch {
      // 404 means no termination notice - that's normal
      return null;
    }
  }

  async saveCheckpoint(jobId: string, data: Omit<CheckpointData, 'workerId' | 'timestamp'>): Promise<void> {
    const checkpoint: CheckpointData = {
      ...data,
      workerId: this.workerId,
      timestamp: Date.now(),
    };

    await this.redis.setEx(
      `${this.checkpointKey}:${jobId}`,
      86400, // 24 hour TTL
      JSON.stringify(checkpoint)
    );
  }

  async loadCheckpoint(jobId: string): Promise<CheckpointData | null> {
    const data = await this.redis.get(`${this.checkpointKey}:${jobId}`);
    if (!data) return null;
    return JSON.parse(data) as CheckpointData;
  }

  async clearCheckpoint(jobId: string): Promise<void> {
    await this.redis.del(`${this.checkpointKey}:${jobId}`);
  }

  private async handleTermination(reason: string): Promise<void> {
    if (this.isTerminating) return;
    this.isTerminating = true;

    console.log(`[SpotHandler ${this.workerId}] Starting graceful shutdown (reason: ${reason})`);
    this.emit('shutdown-start', { reason });

    // Stop polling
    if (this.pollingInterval) {
      clearInterval(this.pollingInterval);
    }

    // Give the application 90 seconds to finish (before the 2-min spot deadline)
    const shutdownDeadline = setTimeout(async () => {
      console.log(`[SpotHandler ${this.workerId}] Shutdown deadline reached - forcing exit`);
      await this.redis.disconnect();
      process.exit(1);
    }, 90000);

    // Wait for graceful shutdown signal
    await new Promise<void>((resolve) => {
      this.once('shutdown-complete', () => {
        clearTimeout(shutdownDeadline);
        resolve();
      });
    });

    await this.redis.disconnect();
    console.log(`[SpotHandler ${this.workerId}] Graceful shutdown complete`);
    process.exit(0);
  }

  isShuttingDown(): boolean {
    return this.isTerminating;
  }

  async signalShutdownComplete(): Promise<void> {
    this.emit('shutdown-complete');
  }
}

// Example: Batch job processor with spot instance support
export class BatchJobProcessor {
  private handler: SpotInstanceHandler;
  private currentJobId: string | null = null;
  private currentProgress = 0;

  constructor(handler: SpotInstanceHandler) {
    this.handler = handler;

    // Listen for shutdown
    handler.on('shutdown-start', async () => {
      await this.onShutdown();
    });
  }

  async processJob(
    jobId: string,
    items: string[],
    processor: (item: string) => Promise<void>
  ): Promise<void> {
    this.currentJobId = jobId;

    // Check for existing checkpoint
    const checkpoint = await this.handler.loadCheckpoint(jobId);
    let startIndex = 0;
    let lastProcessedItem = '';

    if (checkpoint) {
      console.log(`[BatchJob ${jobId}] Resuming from checkpoint: ${checkpoint.progress}% complete`);
      startIndex = items.findIndex((item) => item === checkpoint.lastProcessedItem) + 1;
    }

    for (let i = startIndex; i < items.length; i++) {
      if (this.handler.isShuttingDown()) {
        console.log(`[BatchJob ${jobId}] Interrupted at item ${i}/${items.length}`);
        break;
      }

      const item = items[i];
      await processor(item);
      lastProcessedItem = item;

      this.currentProgress = Math.round(((i + 1) / items.length) * 100);

      // Save checkpoint every 10 items
      if (i % 10 === 0 || i === items.length - 1) {
        await this.handler.saveCheckpoint(jobId, {
          jobId,
          progress: this.currentProgress,
          lastProcessedItem,
          state: { processedCount: i + 1 },
        });
      }
    }

    if (this.currentProgress === 100) {
      await this.handler.clearCheckpoint(jobId);
      console.log(`[BatchJob ${jobId}] Completed successfully`);
    }
  }

  private async onShutdown(): Promise<void> {
    if (this.currentJobId) {
      console.log(
        `[BatchJob ${this.currentJobId}] Saving final checkpoint at ${this.currentProgress}%`
      );
      // Final checkpoint is saved by the main loop above
    }
    await this.handler.signalShutdownComplete();
  }
}
```

---

## 98.5 Reserved Instance ROI Calculator

```typescript
// reserved-instance-calculator.ts
export type RI_Term = '1year' | '3year';
export type RI_PaymentOption = 'no_upfront' | 'partial_upfront' | 'all_upfront';

export interface EC2InstancePricing {
  instanceType: string;
  onDemandHourly: number;
  reservedMonthly: {
    '1year': {
      no_upfront: { monthly: number; upfront: number };
      partial_upfront: { monthly: number; upfront: number };
      all_upfront: { monthly: number; upfront: number };
    };
    '3year': {
      no_upfront: { monthly: number; upfront: number };
      partial_upfront: { monthly: number; upfront: number };
      all_upfront: { monthly: number; upfront: number };
    };
  };
}

export interface ROIAnalysis {
  instanceType: string;
  term: RI_Term;
  paymentOption: RI_PaymentOption;
  quantity: number;
  utilizationPercent: number;
  onDemandMonthly: number;
  riMonthly: number;
  upfrontCost: number;
  breakEvenMonths: number;
  totalSavings: number;
  savingsPercent: number;
  roi: number; // Return on investment %
  recommendation: string;
}

export class ReservedInstanceROICalculator {
  // Sample pricing data (in production, fetch from AWS Pricing API)
  private pricingData: Map<string, EC2InstancePricing> = new Map([
    ['m5.xlarge', {
      instanceType: 'm5.xlarge',
      onDemandHourly: 0.192,
      reservedMonthly: {
        '1year': {
          no_upfront: { monthly: 124, upfront: 0 },
          partial_upfront: { monthly: 60, upfront: 768 },
          all_upfront: { monthly: 0, upfront: 1404 },
        },
        '3year': {
          no_upfront: { monthly: 83, upfront: 0 },
          partial_upfront: { monthly: 40, upfront: 1548 },
          all_upfront: { monthly: 0, upfront: 2628 },
        },
      },
    }],
    ['m5.2xlarge', {
      instanceType: 'm5.2xlarge',
      onDemandHourly: 0.384,
      reservedMonthly: {
        '1year': {
          no_upfront: { monthly: 248, upfront: 0 },
          partial_upfront: { monthly: 120, upfront: 1536 },
          all_upfront: { monthly: 0, upfront: 2808 },
        },
        '3year': {
          no_upfront: { monthly: 166, upfront: 0 },
          partial_upfront: { monthly: 80, upfront: 3096 },
          all_upfront: { monthly: 0, upfront: 5256 },
        },
      },
    }],
    ['c5.xlarge', {
      instanceType: 'c5.xlarge',
      onDemandHourly: 0.170,
      reservedMonthly: {
        '1year': {
          no_upfront: { monthly: 110, upfront: 0 },
          partial_upfront: { monthly: 53, upfront: 682 },
          all_upfront: { monthly: 0, upfront: 1245 },
        },
        '3year': {
          no_upfront: { monthly: 73, upfront: 0 },
          partial_upfront: { monthly: 35, upfront: 1367 },
          all_upfront: { monthly: 0, upfront: 2330 },
        },
      },
    }],
  ]);

  calculateROI(
    instanceType: string,
    term: RI_Term,
    paymentOption: RI_PaymentOption,
    quantity: number,
    utilizationPercent: number = 100
  ): ROIAnalysis {
    const pricing = this.pricingData.get(instanceType);
    if (!pricing) throw new Error(`Unknown instance type: ${instanceType}`);

    const termMonths = term === '1year' ? 12 : 36;
    const riPricing = pricing.reservedMonthly[term][paymentOption];

    // On-demand cost (adjusted for utilization)
    const hoursPerMonth = 730;
    const onDemandMonthly = pricing.onDemandHourly * hoursPerMonth * (utilizationPercent / 100) * quantity;

    // RI cost
    const riMonthly = riPricing.monthly * quantity;
    const upfrontCost = riPricing.upfront * quantity;

    // Total costs over the term
    const totalOnDemand = onDemandMonthly * termMonths;
    const totalRI = riMonthly * termMonths + upfrontCost;
    const totalSavings = totalOnDemand - totalRI;
    const savingsPercent = (totalSavings / totalOnDemand) * 100;

    // Break-even analysis
    const monthlySavings = onDemandMonthly - riMonthly;
    const breakEvenMonths = monthlySavings > 0
      ? Math.ceil(upfrontCost / monthlySavings)
      : 0;

    // ROI = (Net Profit / Cost) * 100
    const roi = upfrontCost > 0
      ? ((totalSavings) / (upfrontCost + riMonthly * termMonths)) * 100
      : savingsPercent;

    let recommendation: string;
    if (savingsPercent > 40 && breakEvenMonths < 6) {
      recommendation = '✅ STRONGLY RECOMMENDED - Excellent ROI with fast break-even';
    } else if (savingsPercent > 25 && breakEvenMonths < 12) {
      recommendation = '✅ RECOMMENDED - Good savings with acceptable break-even';
    } else if (savingsPercent > 15) {
      recommendation = '⚠️ CONSIDER - Moderate savings, evaluate cash flow impact';
    } else {
      recommendation = '❌ NOT RECOMMENDED - Low savings, consider Savings Plans instead';
    }

    return {
      instanceType,
      term,
      paymentOption,
      quantity,
      utilizationPercent,
      onDemandMonthly,
      riMonthly,
      upfrontCost,
      breakEvenMonths,
      totalSavings,
      savingsPercent,
      roi,
      recommendation,
    };
  }

  compareOptions(
    instanceType: string,
    quantity: number,
    utilizationPercent: number = 100
  ): ROIAnalysis[] {
    const analyses: ROIAnalysis[] = [];

    const terms: RI_Term[] = ['1year', '3year'];
    const paymentOptions: RI_PaymentOption[] = ['no_upfront', 'partial_upfront', 'all_upfront'];

    for (const term of terms) {
      for (const paymentOption of paymentOptions) {
        analyses.push(
          this.calculateROI(instanceType, term, paymentOption, quantity, utilizationPercent)
        );
      }
    }

    return analyses.sort((a, b) => b.totalSavings - a.totalSavings);
  }

  generateReport(analyses: ROIAnalysis[]): string {
    const lines = [
      '# Reserved Instance ROI Analysis',
      '',
      `| Term | Payment | Upfront | Monthly RI | Monthly OD | Savings | Break-even | Recommendation |`,
      `|------|---------|---------|------------|------------|---------|------------|----------------|`,
    ];

    analyses.forEach((a) => {
      lines.push(
        `| ${a.term} | ${a.paymentOption} | $${a.upfrontCost.toFixed(0)} | $${a.riMonthly.toFixed(0)} | $${a.onDemandMonthly.toFixed(0)} | ${a.savingsPercent.toFixed(1)}% | ${a.breakEvenMonths} months | ${a.recommendation} |`
      );
    });

    return lines.join('\n');
  }
}
```

---

## 98.6 Cost Attribution Middleware

```typescript
// cost-attribution-middleware.ts
import { Request, Response, NextFunction } from 'express';
import { Counter, Histogram, register } from 'prom-client';

export interface CostAttributionConfig {
  serviceName: string;
  teamName: string;
  environment: string;
  costCenter?: string;
  enableDetailedTracking?: boolean;
}

export interface RequestCostMetadata {
  service: string;
  team: string;
  environment: string;
  endpoint: string;
  method: string;
  costCenter: string;
}

// Prometheus metrics for cost attribution
const requestCounter = new Counter({
  name: 'http_requests_total_with_cost_attribution',
  help: 'Total HTTP requests with cost attribution labels',
  labelNames: ['service', 'team', 'environment', 'endpoint', 'method', 'cost_center', 'status'],
});

const requestDurationHistogram = new Histogram({
  name: 'http_request_duration_ms_with_attribution',
  help: 'HTTP request duration in milliseconds with cost attribution',
  labelNames: ['service', 'team', 'environment', 'endpoint', 'method', 'cost_center'],
  buckets: [10, 50, 100, 250, 500, 1000, 2500, 5000],
});

const responseBodySizeHistogram = new Histogram({
  name: 'http_response_body_bytes_with_attribution',
  help: 'HTTP response body size in bytes',
  labelNames: ['service', 'team', 'environment', 'endpoint', 'cost_center'],
  buckets: [100, 1000, 10000, 100000, 1000000],
});

export function createCostAttributionMiddleware(
  config: CostAttributionConfig
): (req: Request, res: Response, next: NextFunction) => void {
  return (req: Request, res: Response, next: NextFunction): void => {
    const startTime = Date.now();

    // Add cost attribution headers to request
    req.headers['x-service-name'] = config.serviceName;
    req.headers['x-team-name'] = config.teamName;
    req.headers['x-cost-center'] = config.costCenter ?? config.teamName;
    req.headers['x-environment'] = config.environment;

    // Normalize endpoint path (remove IDs for better cardinality)
    const normalizedPath = normalizeEndpointPath(req.path);

    const metadata: RequestCostMetadata = {
      service: config.serviceName,
      team: config.teamName,
      environment: config.environment,
      endpoint: normalizedPath,
      method: req.method,
      costCenter: config.costCenter ?? config.teamName,
    };

    // Capture response
    const originalEnd = res.end.bind(res);
    let responseBodySize = 0;

    res.end = function (
      chunk?: unknown,
      ...args: unknown[]
    ): Response {
      if (chunk) {
        responseBodySize = Buffer.isBuffer(chunk)
          ? chunk.length
          : Buffer.byteLength(String(chunk));
      }

      const duration = Date.now() - startTime;

      // Record metrics
      requestCounter.labels(
        metadata.service,
        metadata.team,
        metadata.environment,
        metadata.endpoint,
        metadata.method,
        metadata.costCenter,
        String(res.statusCode)
      ).inc();

      requestDurationHistogram.labels(
        metadata.service,
        metadata.team,
        metadata.environment,
        metadata.endpoint,
        metadata.method,
        metadata.costCenter
      ).observe(duration);

      if (responseBodySize > 0) {
        responseBodySizeHistogram.labels(
          metadata.service,
          metadata.team,
          metadata.environment,
          metadata.endpoint,
          metadata.costCenter
        ).observe(responseBodySize);
      }

      return originalEnd.call(res, chunk, ...args as Parameters<typeof originalEnd>);
    };

    next();
  };
}

function normalizeEndpointPath(path: string): string {
  // Replace UUIDs and numeric IDs with placeholders
  return path
    .replace(/[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}/gi, ':uuid')
    .replace(/\/\d+/g, '/:id')
    .replace(/\?.*$/, ''); // Remove query string
}

// Kubernetes resource tagging for cost attribution
export function generateKubernetesLabels(team: string, service: string, costCenter: string): Record<string, string> {
  return {
    'app.kubernetes.io/name': service,
    'app.kubernetes.io/component': 'microservice',
    'team': team.toLowerCase().replace(/\s+/g, '-'),
    'cost-center': costCenter,
    'managed-by': 'platform-team',
  };
}
```

---

## 98.7 Database Cost Comparison: RDS vs Aurora Serverless

```typescript
// database-cost-calculator.ts
export interface RDSConfig {
  instanceClass: string;
  multiAZ: boolean;
  storageGB: number;
  iops?: number;
  backupRetentionDays: number;
  dataTransferGB: number;
}

export interface AuroraServerlessConfig {
  minACU: number;
  maxACU: number;
  storageGB: number;
  backupRetentionDays: number;
  avgACU: number; // average ACU during operation
  operatingHoursPerDay: number;
}

export interface DatabaseCostComparison {
  rds: {
    monthlyCompute: number;
    monthlyStorage: number;
    monthlyIOPS: number;
    monthlyBackup: number;
    monthlyDataTransfer: number;
    monthlyTotal: number;
  };
  auroraServerless: {
    monthlyACU: number;
    monthlyStorage: number;
    monthlyBackup: number;
    monthlyTotal: number;
    pauseEnabled: boolean;
    pausedHoursPerMonth: number;
  };
  breakEvenHoursPerDay: number;
  recommendation: string;
  chart: Array<{ hoursPerDay: number; rdsMonthly: number; auroraMonthly: number }>;
}

export class DatabaseCostCalculator {
  // AWS pricing (us-east-1, approximate)
  private readonly RDS_PRICING: Record<string, number> = {
    'db.t3.micro': 0.017,
    'db.t3.small': 0.034,
    'db.t3.medium': 0.068,
    'db.r5.large': 0.24,
    'db.r5.xlarge': 0.48,
    'db.r5.2xlarge': 0.96,
  };

  private readonly AURORA_ACU_HOURLY = 0.06; // per ACU-hour
  private readonly AURORA_STORAGE_PER_GB = 0.10; // per GB-month
  private readonly AURORA_IOPS_PER_MILLION = 0.20;
  private readonly RDS_STORAGE_PER_GB = 0.115; // gp2
  private readonly RDS_STORAGE_IOPS = 0.10; // per provisioned IOPS
  private readonly BACKUP_PER_GB = 0.095; // per GB-month
  private readonly DATA_TRANSFER_PER_GB = 0.09;

  compareRDSvsAuroraServerless(
    rds: RDSConfig,
    aurora: AuroraServerlessConfig
  ): DatabaseCostComparison {
    // RDS Costs
    const rdsHourlyRate = this.RDS_PRICING[rds.instanceClass] ?? 0.068;
    const rdsMultiAZMultiplier = rds.multiAZ ? 2 : 1;
    const rdsMonthlyCompute = rdsHourlyRate * 730 * rdsMultiAZMultiplier;
    const rdsMonthlyStorage = rds.storageGB * this.RDS_STORAGE_PER_GB;
    const rdsMonthlyIOPS = rds.iops ? rds.iops * this.RDS_STORAGE_IOPS : 0;
    const rdsMonthlyBackup = rds.storageGB * this.BACKUP_PER_GB * (rds.backupRetentionDays / 30);
    const rdsMonthlyDataTransfer = rds.dataTransferGB * this.DATA_TRANSFER_PER_GB;
    const rdsMonthlyTotal = rdsMonthlyCompute + rdsMonthlyStorage + rdsMonthlyIOPS + rdsMonthlyBackup + rdsMonthlyDataTransfer;

    // Aurora Serverless Costs
    const pausedHoursPerMonth = (24 - aurora.operatingHoursPerDay) * 30;
    const activeHoursPerMonth = aurora.operatingHoursPerDay * 30;
    const auroraMonthlyACU = aurora.avgACU * this.AURORA_ACU_HOURLY * activeHoursPerMonth;
    const auroraMonthlyStorage = aurora.storageGB * this.AURORA_STORAGE_PER_GB;
    const auroraMonthlyBackup = aurora.storageGB * this.BACKUP_PER_GB * (aurora.backupRetentionDays / 30);
    const auroraMonthlyTotal = auroraMonthlyACU + auroraMonthlyStorage + auroraMonthlyBackup;

    // Break-even calculation
    const rdsHourlyCost = rdsMonthlyCompute / 730;
    const auroraHourlyCost = aurora.avgACU * this.AURORA_ACU_HOURLY;
    const breakEvenHoursPerDay = rdsHourlyCost > auroraHourlyCost ? 24
      : (rdsMonthlyStorage + rdsMonthlyIOPS) / ((auroraHourlyCost - rdsHourlyCost) * 30);

    // Generate comparison chart data
    const chart = Array.from({ length: 25 }, (_, i) => {
      const hours = i;
      const rdsM = rdsMonthlyTotal; // RDS is constant regardless of usage hours
      const auroraM = (aurora.avgACU * this.AURORA_ACU_HOURLY * hours * 30) + auroraMonthlyStorage + auroraMonthlyBackup;
      return { hoursPerDay: hours, rdsMonthly: Math.round(rdsM), auroraMonthly: Math.round(auroraM) };
    });

    let recommendation: string;
    if (aurora.operatingHoursPerDay < 12 && auroraMonthlyTotal < rdsMonthlyTotal) {
      recommendation = `✅ Aurora Serverless recommended - saves $${(rdsMonthlyTotal - auroraMonthlyTotal).toFixed(0)}/month for low-usage workloads`;
    } else if (aurora.operatingHoursPerDay >= 18 && rdsMonthlyTotal < auroraMonthlyTotal) {
      recommendation = `✅ RDS recommended - saves $${(auroraMonthlyTotal - rdsMonthlyTotal).toFixed(0)}/month for consistently high-usage workloads`;
    } else {
      recommendation = `⚖️ Consider Aurora Serverless if usage is unpredictable, RDS for consistent load`;
    }

    return {
      rds: {
        monthlyCompute: rdsMonthlyCompute,
        monthlyStorage: rdsMonthlyStorage,
        monthlyIOPS: rdsMonthlyIOPS,
        monthlyBackup: rdsMonthlyBackup,
        monthlyDataTransfer: rdsMonthlyDataTransfer,
        monthlyTotal: rdsMonthlyTotal,
      },
      auroraServerless: {
        monthlyACU: auroraMonthlyACU,
        monthlyStorage: auroraMonthlyStorage,
        monthlyBackup: auroraMonthlyBackup,
        monthlyTotal: auroraMonthlyTotal,
        pauseEnabled: aurora.operatingHoursPerDay < 24,
        pausedHoursPerMonth,
      },
      breakEvenHoursPerDay: Math.round(breakEvenHoursPerDay * 10) / 10,
      recommendation,
      chart,
    };
  }
}
```

---

## 98.8 Network Cross-AZ Cost Estimator

```typescript
// network-cost-estimator.ts
export interface ServiceDependency {
  fromService: string;
  fromAZ: string;
  toService: string;
  toAZ: string;
  dailyTrafficGB: number;
  requestsPerSecond: number;
  avgResponseSizeKB: number;
}

export interface NetworkCostEstimate {
  crossAZTrafficGB: number;
  sameAZTrafficGB: number;
  crossAZCostMonthly: number;
  crossAZCostAnnual: number;
  savingsWithAZLocality: number;
  recommendations: string[];
  topExpensivePairs: Array<{
    from: string;
    to: string;
    monthlyGB: number;
    monthlyCost: number;
  }>;
}

export class NetworkCostEstimator {
  private readonly CROSS_AZ_TRANSFER_PRICE = 0.01; // per GB

  estimateCosts(dependencies: ServiceDependency[]): NetworkCostEstimate {
    const crossAZDeps = dependencies.filter((d) => d.fromAZ !== d.toAZ);
    const sameAZDeps = dependencies.filter((d) => d.fromAZ === d.toAZ);

    const crossAZTrafficGB = crossAZDeps.reduce((sum, d) => sum + d.dailyTrafficGB * 30, 0);
    const sameAZTrafficGB = sameAZDeps.reduce((sum, d) => sum + d.dailyTrafficGB * 30, 0);

    const crossAZCostMonthly = crossAZTrafficGB * this.CROSS_AZ_TRANSFER_PRICE;

    // Top expensive pairs
    const expensivePairs = crossAZDeps
      .map((d) => ({
        from: `${d.fromService} (${d.fromAZ})`,
        to: `${d.toService} (${d.toAZ})`,
        monthlyGB: d.dailyTrafficGB * 30,
        monthlyCost: d.dailyTrafficGB * 30 * this.CROSS_AZ_TRANSFER_PRICE,
      }))
      .sort((a, b) => b.monthlyCost - a.monthlyCost)
      .slice(0, 5);

    // Estimate savings if we use AZ locality
    const savingsWithAZLocality = crossAZCostMonthly * 0.7; // 70% reduction with topology-aware routing

    const recommendations: string[] = [];
    if (crossAZCostMonthly > 100) {
      recommendations.push(
        `Enable topology-aware routing in Kubernetes to reduce cross-AZ traffic`
      );
      recommendations.push(
        `Consider deploying read replicas in same AZ as consumers`
      );
    }

    if (expensivePairs.length > 0) {
      recommendations.push(
        `Co-locate ${expensivePairs[0].from} with ${expensivePairs[0].to} for maximum savings`
      );
    }

    if (crossAZTrafficGB > sameAZTrafficGB) {
      recommendations.push(
        `Review service placement - majority of traffic is cross-AZ`
      );
    }

    return {
      crossAZTrafficGB,
      sameAZTrafficGB,
      crossAZCostMonthly,
      crossAZCostAnnual: crossAZCostMonthly * 12,
      savingsWithAZLocality,
      recommendations,
      topExpensivePairs: expensivePairs,
    };
  }
}
```

---

## 98.9 Kubecost TypeScript API Client

```typescript
// kubecost-client.ts
import axios, { AxiosInstance } from 'axios';

export interface KubecostAllocation {
  name: string;
  window: string;
  start: string;
  end: string;
  cpuCoreRequestAverage: number;
  cpuCoreUsageAverage: number;
  cpuCost: number;
  gpuCost: number;
  networkCost: number;
  loadBalancerCost: number;
  pvCost: number;
  ramByteRequestAverage: number;
  ramByteUsageAverage: number;
  ramCost: number;
  sharedCost: number;
  externalCost: number;
  totalCost: number;
  totalEfficiency: number;
}

export interface KubecostSummary {
  namespace?: string;
  deployment?: string;
  totalCost: number;
  cpuCost: number;
  ramCost: number;
  networkCost: number;
  pvCost: number;
  efficiency: number;
  period: string;
}

export class KubecostClient {
  private client: AxiosInstance;

  constructor(kubecostUrl: string) {
    this.client = axios.create({
      baseURL: kubecostUrl,
      timeout: 30000,
    });
  }

  async getAllocations(
    window: string = '7d',
    aggregate: string = 'namespace',
    namespace?: string
  ): Promise<KubecostAllocation[]> {
    const params: Record<string, string> = {
      window,
      aggregate,
      step: '1d',
      accumulate: 'true',
    };

    if (namespace) {
      params.filter = `namespace:"${namespace}"`;
    }

    const response = await this.client.get('/model/allocation', { params });
    const allocations: KubecostAllocation[] = [];

    // Kubecost returns an array of time windows
    const data = response.data?.data?.[0] ?? {};
    Object.entries(data).forEach(([name, alloc]) => {
      allocations.push({
        name,
        ...(alloc as Omit<KubecostAllocation, 'name'>),
      });
    });

    return allocations.sort((a, b) => b.totalCost - a.totalCost);
  }

  async getNamespaceCosts(
    window: string = '30d'
  ): Promise<Array<KubecostSummary & { namespace: string }>> {
    const allocations = await this.getAllocations(window, 'namespace');

    return allocations.map((a) => ({
      namespace: a.name,
      totalCost: a.totalCost,
      cpuCost: a.cpuCost,
      ramCost: a.ramCost,
      networkCost: a.networkCost,
      pvCost: a.pvCost,
      efficiency: a.totalEfficiency,
      period: window,
    }));
  }

  async getDeploymentCosts(
    namespace: string,
    window: string = '30d'
  ): Promise<Array<KubecostSummary & { deployment: string }>> {
    const allocations = await this.getAllocations(
      window,
      'deployment',
      namespace
    );

    return allocations.map((a) => ({
      deployment: a.name,
      namespace,
      totalCost: a.totalCost,
      cpuCost: a.cpuCost,
      ramCost: a.ramCost,
      networkCost: a.networkCost,
      pvCost: a.pvCost,
      efficiency: a.totalEfficiency,
      period: window,
    }));
  }

  async getTopExpensiveDeployments(
    topN: number = 10,
    window: string = '30d'
  ): Promise<Array<KubecostSummary & { deployment: string; namespace: string }>> {
    const allocations = await this.getAllocations(window, 'deployment,namespace');

    return allocations
      .slice(0, topN)
      .map((a) => {
        const [deployment, namespace] = a.name.split('/');
        return {
          deployment,
          namespace,
          totalCost: a.totalCost,
          cpuCost: a.cpuCost,
          ramCost: a.ramCost,
          networkCost: a.networkCost,
          pvCost: a.pvCost,
          efficiency: a.totalEfficiency,
          period: window,
        };
      });
  }

  async getClusterTotalCost(window: string = '30d'): Promise<number> {
    const allocations = await this.getAllocations(window, 'cluster');
    return allocations.reduce((sum, a) => sum + a.totalCost, 0);
  }
}
```

---

## 98.10 Monthly Cost Report Generator

```typescript
// monthly-cost-report-generator.ts
import { CostDashboard } from './cost-dashboard';
import { KubecostClient } from './kubecost-client';
import { ContainerRightsizingAnalyzer } from './container-rightsizing-analyzer';

export interface MonthlyReport {
  generatedAt: Date;
  period: string;
  executive_summary: {
    totalCloudCost: number;
    totalK8sCost: number;
    monthOverMonthChange: number;
    topSavingsOpportunity: string;
    estimatedMonthlySavings: number;
  };
  aws_costs: {
    total: number;
    byService: Array<{ service: string; cost: number; change: number }>;
    byTeam: Array<{ team: string; cost: number }>;
    forecast: number;
  };
  kubernetes_costs: {
    total: number;
    byNamespace: Array<{ namespace: string; cost: number; efficiency: number }>;
    topExpensiveDeployments: Array<{ name: string; namespace: string; cost: number }>;
  };
  optimization_opportunities: {
    rightsizing: Array<{
      service: string;
      currentCost: number;
      potentialSaving: number;
      risk: string;
    }>;
    unusedResources: string[];
    reservedInstanceOpportunities: string[];
  };
  action_items: Array<{
    priority: 'high' | 'medium' | 'low';
    action: string;
    estimatedSaving: number;
    owner: string;
    dueDate: string;
  }>;
}

export class MonthlyReportGenerator {
  private costDashboard: CostDashboard;
  private kubecostClient: KubecostClient;
  private rightsizingAnalyzer: ContainerRightsizingAnalyzer;

  constructor(
    awsRegion: string,
    kubecostUrl: string,
    prometheusUrl: string
  ) {
    this.costDashboard = new CostDashboard(awsRegion);
    this.kubecostClient = new KubecostClient(kubecostUrl);
    this.rightsizingAnalyzer = new ContainerRightsizingAnalyzer(prometheusUrl);
  }

  async generateReport(year: number, month: number): Promise<MonthlyReport> {
    const [awsReport, k8sCosts, k8sNamespaces] = await Promise.all([
      this.costDashboard.getMonthlyReport(year, month),
      this.kubecostClient.getClusterTotalCost('30d'),
      this.kubecostClient.getNamespaceCosts('30d'),
    ]);

    const topDeployments = await this.kubecostClient.getTopExpensiveDeployments(10);
    const rightsizingRecs = await this.rightsizingAnalyzer.analyzeNamespace('production');

    const prevMonthChange = awsReport.topServices[0]?.changePercent ?? 0;
    const totalSavingsOpportunity = rightsizingRecs.reduce(
      (sum, r) => sum + r.estimatedMonthlySavingsUSD,
      0
    );

    const report: MonthlyReport = {
      generatedAt: new Date(),
      period: `${year}-${String(month).padStart(2, '0')}`,
      executive_summary: {
        totalCloudCost: awsReport.totalCost,
        totalK8sCost: k8sCosts,
        monthOverMonthChange: prevMonthChange,
        topSavingsOpportunity: rightsizingRecs[0]
          ? `Right-size ${rightsizingRecs[0].deploymentName}: save $${rightsizingRecs[0].estimatedMonthlySavingsUSD}/month`
          : 'Review reserved instances',
        estimatedMonthlySavings: totalSavingsOpportunity,
      },
      aws_costs: {
        total: awsReport.totalCost,
        byService: awsReport.topServices.map((s) => ({
          service: s.service,
          cost: s.currentMonthCost,
          change: s.changePercent,
        })),
        byTeam: Object.entries(awsReport.byTag).map(([team, cost]) => ({ team, cost })),
        forecast: awsReport.forecastEndOfMonth,
      },
      kubernetes_costs: {
        total: k8sCosts,
        byNamespace: k8sNamespaces.map((n) => ({
          namespace: n.namespace,
          cost: n.totalCost,
          efficiency: n.efficiency,
        })),
        topExpensiveDeployments: topDeployments.map((d) => ({
          name: d.deployment,
          namespace: d.namespace,
          cost: d.totalCost,
        })),
      },
      optimization_opportunities: {
        rightsizing: rightsizingRecs.slice(0, 5).map((r) => ({
          service: `${r.namespace}/${r.deploymentName}`,
          currentCost: r.currentConfig.requestCPU * 0.0001 + r.currentConfig.requestMemory * 0.00005,
          potentialSaving: r.estimatedMonthlySavingsUSD,
          risk: r.risk,
        })),
        unusedResources: [
          'Review unattached EBS volumes',
          'Check idle load balancers',
          'Review unused reserved instances',
        ],
        reservedInstanceOpportunities: [
          'Consider 1-year RI for stable production workloads',
          'Evaluate Compute Savings Plans for Kubernetes nodes',
        ],
      },
      action_items: this.generateActionItems(rightsizingRecs, awsReport),
    };

    return report;
  }

  private generateActionItems(
    rightsizingRecs: Awaited<ReturnType<ContainerRightsizingAnalyzer['analyzeNamespace']>>,
    awsReport: Awaited<ReturnType<CostDashboard['getMonthlyReport']>>
  ): MonthlyReport['action_items'] {
    const items: MonthlyReport['action_items'] = [];

    rightsizingRecs.slice(0, 3).forEach((rec) => {
      items.push({
        priority: rec.estimatedSavingsPercent > 40 ? 'high' : 'medium',
        action: `Right-size ${rec.deploymentName}: reduce CPU from ${rec.currentConfig.requestCPU}m to ${rec.recommendedRequests.cpu}`,
        estimatedSaving: rec.estimatedMonthlySavingsUSD,
        owner: 'Platform Team',
        dueDate: new Date(Date.now() + 14 * 24 * 3600 * 1000).toISOString().split('T')[0],
      });
    });

    // Add high-change service item
    const highChangeService = awsReport.topServices.find((s) => s.changePercent > 20);
    if (highChangeService) {
      items.push({
        priority: 'high',
        action: `Investigate ${highChangeService.service} cost increase of ${highChangeService.changePercent.toFixed(0)}% MoM`,
        estimatedSaving: highChangeService.currentMonthCost * 0.15,
        owner: 'FinOps Team',
        dueDate: new Date(Date.now() + 7 * 24 * 3600 * 1000).toISOString().split('T')[0],
      });
    }

    return items.sort((a, b) => {
      const priority = { high: 0, medium: 1, low: 2 };
      return priority[a.priority] - priority[b.priority];
    });
  }

  formatReportAsMarkdown(report: MonthlyReport): string {
    return `# Monthly Cloud Cost Report - ${report.period}
Generated: ${report.generatedAt.toISOString()}

## Executive Summary

| Metric | Value |
|--------|-------|
| Total AWS Cost | $${report.executive_summary.totalCloudCost.toFixed(2)} |
| Total K8s Cost | $${report.executive_summary.totalK8sCost.toFixed(2)} |
| MoM Change | ${report.executive_summary.monthOverMonthChange > 0 ? '+' : ''}${report.executive_summary.monthOverMonthChange.toFixed(1)}% |
| Savings Opportunity | $${report.executive_summary.estimatedMonthlySavings.toFixed(0)}/month |

## Top Services by Cost
${report.aws_costs.byService
  .slice(0, 5)
  .map((s) => `- ${s.service}: $${s.cost.toFixed(2)} (${s.change > 0 ? '+' : ''}${s.change.toFixed(1)}% MoM)`)
  .join('\n')}

## Action Items
${report.action_items
  .map((item) => `### [${item.priority.toUpperCase()}] ${item.action}
- **Estimated Saving**: $${item.estimatedSaving.toFixed(0)}/month
- **Owner**: ${item.owner}
- **Due**: ${item.dueDate}`)
  .join('\n\n')}
`;
  }
}
```

---

## 98.11 สรุป

Cost Optimization ที่ดีต้องมีความสมดุลระหว่างการลดค่าใช้จ่ายกับการรักษา reliability และ performance:

1. **วัดก่อนแก้**: ใช้ CostDashboard และ Kubecost เพื่อเข้าใจ current state
2. **Right-sizing**: ปรับ container resources ให้พอดีกับ actual usage
3. **Spot instances**: ประหยัดได้ 60-90% สำหรับ batch workloads พร้อม graceful termination
4. **Reserved capacity**: ประหยัดได้ 30-60% สำหรับ predictable workloads
5. **Cost attribution**: ติด tag ทุก resource เพื่อให้ทีมเห็นค่าใช้จ่ายของตัวเอง
6. **Regular reporting**: Monthly report พร้อม action items ที่ชัดเจน

---

*ต่อไปใน Part 99: Final Project - Complete Microservices System - ประกอบทุกสิ่งที่เรียนมาเป็น production-ready system*

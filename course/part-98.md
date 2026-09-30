# Part 98: Cost Optimization Strategies

## กลยุทธ์การลดต้นทุน Cloud สำหรับ Microservices

ในบทนี้เราจะเรียนรู้วิธีลดต้นทุน Cloud ในระบบ Microservices อย่างมีประสิทธิภาพ

---

## 1. ภาพรวมต้นทุน Cloud สำหรับ Microservices

```
Cloud Cost Breakdown (ตัวอย่างระบบขนาดกลาง)
═══════════════════════════════════════════

Compute (Kubernetes Nodes): 45%
├── On-demand Instances: 60%
├── Spot/Preemptible: 30%
└── Reserved Instances: 10%

Storage: 20%
├── Block Storage (PV): 40%
├── Object Storage (S3): 35%
└── Database Storage: 25%

Database: 15%
├── RDS/Cloud SQL: 60%
├── ElastiCache/Memorystore: 30%
└── Other: 10%

Network: 12%
├── Egress: 70%
├── Load Balancer: 20%
└── VPN/CDN: 10%

Other Services: 8%
```

---

## 2. Right-sizing Containers

การ Right-sizing คือการกำหนด Resource Requests/Limits ให้เหมาะสม

### VPA (Vertical Pod Autoscaler)

```yaml
# vpa/payment-service-vpa.yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: payment-service-vpa
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: payment-service
  updatePolicy:
    updateMode: "Off"  # แนะนำค่า แต่ไม่ Update อัตโนมัติ
  resourcePolicy:
    containerPolicies:
      - containerName: payment-service
        minAllowed:
          cpu: 50m
          memory: 128Mi
        maxAllowed:
          cpu: 2000m
          memory: 2Gi
        controlledResources:
          - cpu
          - memory
```

```typescript
// cost-optimization/src/rightsizing.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { PrometheusService } from './prometheus.service';

export interface ResourceRecommendation {
  serviceName: string;
  namespace: string;
  current: {
    cpuRequest: string;
    memoryRequest: string;
    cpuLimit: string;
    memoryLimit: string;
  };
  recommended: {
    cpuRequest: string;
    memoryRequest: string;
    cpuLimit: string;
    memoryLimit: string;
  };
  potentialSavings: {
    cpu: number;
    memory: number;
    estimatedMonthlySaving: number;
  };
}

@Injectable()
export class RightSizingService {
  private readonly logger = new Logger(RightSizingService.name);
  
  // Cost per unit (USD/month)
  private readonly CPU_COST_PER_CORE = 30;  // $30/core/month
  private readonly MEMORY_COST_PER_GB = 5;  // $5/GB/month

  constructor(private readonly prometheusService: PrometheusService) {}

  async analyzeService(
    serviceName: string,
    namespace: string,
    days: number = 14,
  ): Promise<ResourceRecommendation> {
    // ดึงข้อมูลการใช้งานจาก Prometheus
    const [cpuUsage, memoryUsage, currentResources] = await Promise.all([
      this.getCPUUsagePercentiles(serviceName, namespace, days),
      this.getMemoryUsagePercentiles(serviceName, namespace, days),
      this.getCurrentResources(serviceName, namespace),
    ]);

    // คำนวณ Recommendations
    const recommendedCPURequest = this.calculateCPURequest(cpuUsage);
    const recommendedCPULimit = this.calculateCPULimit(cpuUsage);
    const recommendedMemRequest = this.calculateMemoryRequest(memoryUsage);
    const recommendedMemLimit = this.calculateMemoryLimit(memoryUsage);

    // คำนวณการประหยัด
    const currentCPU = this.parseCPU(currentResources.cpuRequest);
    const recommendedCPU = this.parseCPU(recommendedCPURequest);
    const cpuSaving = Math.max(0, currentCPU - recommendedCPU);

    const currentMem = this.parseMemory(currentResources.memoryRequest);
    const recommendedMem = this.parseMemory(recommendedMemRequest);
    const memSaving = Math.max(0, currentMem - recommendedMem);

    const estimatedMonthlySaving = 
      (cpuSaving * this.CPU_COST_PER_CORE) +
      (memSaving / 1024 * this.MEMORY_COST_PER_GB);

    return {
      serviceName,
      namespace,
      current: currentResources,
      recommended: {
        cpuRequest: recommendedCPURequest,
        memoryRequest: recommendedMemRequest,
        cpuLimit: recommendedCPULimit,
        memoryLimit: recommendedMemLimit,
      },
      potentialSavings: {
        cpu: cpuSaving,
        memory: memSaving,
        estimatedMonthlySaving,
      },
    };
  }

  private async getCPUUsagePercentiles(
    serviceName: string,
    namespace: string,
    days: number,
  ): Promise<{ p50: number; p90: number; p99: number; max: number }> {
    const range = `${days * 24}h`;
    
    const queries = {
      p50: `quantile_over_time(0.50, rate(container_cpu_usage_seconds_total{namespace="${namespace}", pod=~"${serviceName}-.*"}[5m])[${range}:5m])`,
      p90: `quantile_over_time(0.90, rate(container_cpu_usage_seconds_total{namespace="${namespace}", pod=~"${serviceName}-.*"}[5m])[${range}:5m])`,
      p99: `quantile_over_time(0.99, rate(container_cpu_usage_seconds_total{namespace="${namespace}", pod=~"${serviceName}-.*"}[5m])[${range}:5m])`,
      max: `max_over_time(rate(container_cpu_usage_seconds_total{namespace="${namespace}", pod=~"${serviceName}-.*"}[5m])[${range}:5m])`,
    };

    const results = await Promise.all(
      Object.entries(queries).map(([key, query]) =>
        this.prometheusService.query(query).then(r => [key, r])
      )
    );

    return Object.fromEntries(results) as any;
  }

  private async getMemoryUsagePercentiles(
    serviceName: string,
    namespace: string,
    days: number,
  ): Promise<{ p50: number; p90: number; p99: number; max: number }> {
    const range = `${days * 24}h`;
    
    const query = `quantile_over_time(0.99, container_memory_working_set_bytes{namespace="${namespace}", pod=~"${serviceName}-.*"}[${range}:5m])`;
    const p99 = await this.prometheusService.query(query);
    
    return { p50: p99 * 0.7, p90: p99 * 0.9, p99, max: p99 * 1.1 };
  }

  private calculateCPURequest(usage: { p50: number; p90: number }): string {
    // Request = P90 + 20% buffer
    const recommended = usage.p90 * 1.2;
    return `${Math.ceil(recommended * 1000)}m`;
  }

  private calculateCPULimit(usage: { max: number }): string {
    // Limit = Max * 2
    const recommended = usage.max * 2;
    return `${Math.ceil(recommended * 1000)}m`;
  }

  private calculateMemoryRequest(usage: { p90: number }): string {
    const recommended = usage.p90 * 1.25;
    return `${Math.ceil(recommended / (1024 * 1024))}Mi`;
  }

  private calculateMemoryLimit(usage: { max: number }): string {
    const recommended = usage.max * 1.5;
    return `${Math.ceil(recommended / (1024 * 1024))}Mi`;
  }

  private async getCurrentResources(serviceName: string, namespace: string): Promise<any> {
    // ดึงจาก Kubernetes API
    return {
      cpuRequest: '500m',
      memoryRequest: '512Mi',
      cpuLimit: '1000m',
      memoryLimit: '1Gi',
    };
  }

  private parseCPU(cpu: string): number {
    if (cpu.endsWith('m')) return parseInt(cpu) / 1000;
    return parseFloat(cpu);
  }

  private parseMemory(mem: string): number {
    if (mem.endsWith('Gi')) return parseFloat(mem) * 1024;
    if (mem.endsWith('Mi')) return parseFloat(mem);
    return parseFloat(mem) / (1024 * 1024);
  }
}
```

---

## 3. Spot/Preemptible Instance Strategies

```yaml
# k8s/spot-node-pool.yaml
# AWS EKS Managed Node Group ด้วย Spot Instances
apiVersion: eks.aws.crossplane.io/v1beta1
kind: NodeGroup
metadata:
  name: spot-workers
spec:
  forProvider:
    region: ap-southeast-1
    clusterName: production-cluster
    nodeGroupName: spot-workers
    scalingConfig:
      minSize: 3
      desiredSize: 10
      maxSize: 50
    instanceTypes:
      - m5.xlarge
      - m5.2xlarge
      - m4.xlarge
      - m4.2xlarge
      - r5.xlarge
    capacityType: SPOT  # ใช้ Spot Instances
    labels:
      node-type: spot
      workload: batch
    taints:
      - key: spot
        value: "true"
        effect: NoSchedule
---
# workload-tolerations.yaml
# Workloads ที่รองรับ Spot Instances
apiVersion: apps/v1
kind: Deployment
metadata:
  name: analytics-worker
  namespace: production
spec:
  replicas: 5
  selector:
    matchLabels:
      app: analytics-worker
  template:
    spec:
      # รองรับ Spot Instance
      tolerations:
        - key: spot
          operator: Equal
          value: "true"
          effect: NoSchedule
      nodeSelector:
        node-type: spot
      
      # Graceful Termination สำหรับ Spot Interruption
      terminationGracePeriodSeconds: 120
      
      containers:
        - name: analytics-worker
          image: your-registry/analytics-worker:1.0.0
          resources:
            requests:
              cpu: 500m
              memory: 1Gi
          # Checkpoint สำหรับ Resume หลัง Interruption
          env:
            - name: CHECKPOINT_ENABLED
              value: "true"
            - name: CHECKPOINT_INTERVAL
              value: "30"
---
# spot-interruption-handler.yaml
# ตรวจจับและ Handle Spot Interruption
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: spot-interruption-handler
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: spot-interruption-handler
  template:
    spec:
      tolerations:
        - key: node.kubernetes.io/unschedulable
          effect: NoSchedule
      containers:
        - name: handler
          image: pmoricz/aws-spot-termination-handler:latest
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
```

```typescript
// spot-handler/src/spot-termination.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { HttpService } from '@nestjs/axios';
import { firstValueFrom } from 'rxjs';

@Injectable()
export class SpotTerminationHandlerService {
  private readonly logger = new Logger(SpotTerminationHandlerService.name);
  private readonly IMDS_ENDPOINT = 'http://169.254.169.254/latest';
  private isTerminating = false;

  async checkForTermination(): Promise<boolean> {
    try {
      const response = await firstValueFrom(
        this.httpService.get(
          `${this.IMDS_ENDPOINT}/meta-data/spot/termination-time`,
        ),
      );
      
      if (response.status === 200) {
        this.logger.warn(`Spot termination scheduled at: ${response.data}`);
        return true;
      }
      
      return false;
    } catch {
      return false;
    }
  }

  async startMonitoring(): Promise<void> {
    this.logger.log('Starting Spot termination monitoring');
    
    setInterval(async () => {
      if (this.isTerminating) return;
      
      const isTerminating = await this.checkForTermination();
      
      if (isTerminating) {
        this.isTerminating = true;
        await this.handleTermination();
      }
    }, 5000); // ตรวจสอบทุก 5 วินาที
  }

  private async handleTermination(): Promise<void> {
    this.logger.warn('Spot instance termination detected! Starting graceful shutdown...');
    
    // 1. หยุดรับ Request ใหม่
    process.env.ACCEPTING_REQUESTS = 'false';
    
    // 2. รอ In-flight Requests ให้เสร็จ (สูงสุด 90 วินาที)
    await this.waitForInflightRequests(90000);
    
    // 3. Save State/Checkpoint
    await this.saveCheckpoint();
    
    // 4. ส่ง Signal ให้ Kubernetes Drain Node
    process.kill(process.pid, 'SIGTERM');
    
    this.logger.log('Graceful shutdown completed');
  }

  private async waitForInflightRequests(timeoutMs: number): Promise<void> {
    const startTime = Date.now();
    
    while (Date.now() - startTime < timeoutMs) {
      const inflightCount = parseInt(process.env.INFLIGHT_REQUESTS || '0');
      if (inflightCount === 0) {
        this.logger.log('All in-flight requests completed');
        return;
      }
      
      this.logger.log(`Waiting for ${inflightCount} in-flight requests...`);
      await new Promise(resolve => setTimeout(resolve, 1000));
    }
    
    this.logger.warn('Timeout waiting for in-flight requests');
  }

  private async saveCheckpoint(): Promise<void> {
    // บันทึก State ไปยัง Redis หรือ S3
    this.logger.log('Saving checkpoint...');
  }
}
```

---

## 4. Reserved Capacity Planning

```typescript
// capacity-planning/src/capacity.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { Cron } from '@nestjs/schedule';

export interface CapacityRecommendation {
  service: string;
  currentCost: number;
  onDemandCost: number;
  reservedCost: number;
  spotCost: number;
  recommendedMix: {
    onDemandPercent: number;
    reservedPercent: number;
    spotPercent: number;
  };
  estimatedSavings: number;
}

@Injectable()
export class CapacityPlanningService {
  private readonly logger = new Logger(CapacityPlanningService.name);

  // ราคา AWS ap-southeast-1 (USD/hour)
  private readonly PRICES = {
    'm5.xlarge': {
      onDemand: 0.214,
      reserved1yr: 0.136,  // Saving ~36%
      reserved3yr: 0.086,  // Saving ~60%
      spot: 0.064,          // Saving ~70%
    },
    'm5.2xlarge': {
      onDemand: 0.428,
      reserved1yr: 0.272,
      reserved3yr: 0.172,
      spot: 0.128,
    },
  };

  @Cron('0 9 1 * *') // วันที่ 1 ของทุกเดือน 09:00
  async generateCapacityReport(): Promise<void> {
    this.logger.log('Generating monthly capacity planning report...');
    
    const recommendations = await this.analyzeWorkloads();
    await this.sendReport(recommendations);
  }

  async analyzeWorkloads(): Promise<CapacityRecommendation[]> {
    const workloads = await this.getWorkloadProfiles();
    const recommendations: CapacityRecommendation[] = [];

    for (const workload of workloads) {
      const recommendation = this.calculateOptimalMix(workload);
      recommendations.push(recommendation);
    }

    return recommendations;
  }

  private calculateOptimalMix(workload: any): CapacityRecommendation {
    const baselineInstances = workload.averageInstances;
    const peakInstances = workload.peakInstances;
    const instanceType = workload.instanceType;
    const prices = this.PRICES[instanceType as keyof typeof this.PRICES];

    // Strategy:
    // - Baseline (สม่ำเสมอ): Reserved Instances
    // - Peak (เป็นครั้งคราว): On-demand
    // - Batch/Dev: Spot Instances

    const reservedCount = Math.floor(baselineInstances * 0.7);
    const onDemandCount = peakInstances - reservedCount;
    const spotCount = Math.floor(workload.batchInstances * 0.8);

    const monthlyHours = 730;

    const currentCost = workload.totalInstances * prices.onDemand * monthlyHours;
    const optimizedCost =
      (reservedCount * prices.reserved1yr * monthlyHours) +
      (onDemandCount * prices.onDemand * monthlyHours) +
      (spotCount * prices.spot * monthlyHours);

    const savings = currentCost - optimizedCost;
    const savingsPercent = (savings / currentCost) * 100;

    return {
      service: workload.name,
      currentCost,
      onDemandCost: currentCost,
      reservedCost: reservedCount * prices.reserved1yr * monthlyHours,
      spotCost: spotCount * prices.spot * monthlyHours,
      recommendedMix: {
        onDemandPercent: Math.round((onDemandCount / workload.totalInstances) * 100),
        reservedPercent: Math.round((reservedCount / workload.totalInstances) * 100),
        spotPercent: Math.round((spotCount / workload.totalInstances) * 100),
      },
      estimatedSavings: savings,
    };
  }

  private async getWorkloadProfiles(): Promise<any[]> {
    return [
      {
        name: 'payment-service',
        instanceType: 'm5.xlarge',
        averageInstances: 5,
        peakInstances: 10,
        batchInstances: 0,
        totalInstances: 10,
      },
      {
        name: 'analytics-worker',
        instanceType: 'm5.2xlarge',
        averageInstances: 2,
        peakInstances: 4,
        batchInstances: 6,
        totalInstances: 8,
      },
    ];
  }

  private async sendReport(recommendations: CapacityRecommendation[]): Promise<void> {
    const totalCurrentCost = recommendations.reduce((sum, r) => sum + r.currentCost, 0);
    const totalSavings = recommendations.reduce((sum, r) => sum + r.estimatedSavings, 0);
    
    this.logger.log(`
Capacity Planning Report
========================
Total Current Monthly Cost: $${totalCurrentCost.toFixed(2)}
Total Potential Savings: $${totalSavings.toFixed(2)} (${((totalSavings/totalCurrentCost)*100).toFixed(1)}%)

Recommendations:
${recommendations.map(r => `
  ${r.service}:
    - Current: $${r.currentCost.toFixed(2)}/month
    - Savings: $${r.estimatedSavings.toFixed(2)}/month
    - Mix: ${r.recommendedMix.reservedPercent}% Reserved, ${r.recommendedMix.onDemandPercent}% On-demand, ${r.recommendedMix.spotPercent}% Spot
`).join('')}
`);
  }
}
```

---

## 5. Cost Attribution and Chargeback

```yaml
# cost-attribution/namespace-labels.yaml
# Label Namespaces เพื่อ Track ต้นทุนตาม Team

apiVersion: v1
kind: Namespace
metadata:
  name: team-payments
  labels:
    team: payments
    cost-center: CC-001
    environment: production
    project: core-platform
---
apiVersion: v1
kind: Namespace
metadata:
  name: team-logistics
  labels:
    team: logistics
    cost-center: CC-002
    environment: production
    project: delivery-platform
```

```typescript
// cost-attribution/src/kubecost.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { HttpService } from '@nestjs/axios';
import { firstValueFrom } from 'rxjs';
import { Cron } from '@nestjs/schedule';

export interface TeamCostReport {
  team: string;
  costCenter: string;
  period: string;
  costs: {
    compute: number;
    memory: number;
    storage: number;
    network: number;
    total: number;
  };
  efficiency: {
    cpuUtilization: number;
    memoryUtilization: number;
    idleCost: number;
  };
  recommendations: string[];
}

@Injectable()
export class KubecostService {
  private readonly logger = new Logger(KubecostService.name);
  private readonly kubecostUrl = process.env.KUBECOST_URL || 'http://kubecost:9090';

  constructor(private readonly httpService: HttpService) {}

  async getTeamCosts(team: string, window: string = '7d'): Promise<TeamCostReport> {
    // เรียก Kubecost API
    const response = await firstValueFrom(
      this.httpService.get(`${this.kubecostUrl}/model/allocation`, {
        params: {
          window,
          aggregate: 'label:team',
          filter: `label[team]="${team}"`,
          includeIdle: true,
        },
      }),
    );

    const data = response.data.data[0]?.[team];
    if (!data) {
      throw new Error(`No cost data found for team ${team}`);
    }

    const efficiency = await this.getEfficiencyMetrics(team, window);

    return {
      team,
      costCenter: this.getCostCenter(team),
      period: window,
      costs: {
        compute: data.cpuCost + data.ramCost,
        memory: data.ramCost,
        storage: data.pvCost + data.sharedCost,
        network: data.networkCost,
        total: data.totalCost,
      },
      efficiency: {
        cpuUtilization: efficiency.cpuUtilization,
        memoryUtilization: efficiency.memoryUtilization,
        idleCost: data.idleCost || 0,
      },
      recommendations: this.generateRecommendations(data, efficiency),
    };
  }

  @Cron('0 8 * * 1') // ทุกวันจันทร์ 08:00
  async generateWeeklyChargebackReport(): Promise<void> {
    const teams = ['payments', 'logistics', 'analytics', 'platform'];
    const reports: TeamCostReport[] = [];

    for (const team of teams) {
      const report = await this.getTeamCosts(team, '7d');
      reports.push(report);
      await this.sendChargebackEmail(report);
    }

    // บันทึกรายงาน
    await this.saveReport(reports);

    const totalCost = reports.reduce((sum, r) => sum + r.costs.total, 0);
    this.logger.log(`Weekly chargeback complete. Total: $${totalCost.toFixed(2)}`);
  }

  private async getEfficiencyMetrics(team: string, window: string): Promise<{
    cpuUtilization: number;
    memoryUtilization: number;
  }> {
    const response = await firstValueFrom(
      this.httpService.get(`${this.kubecostUrl}/model/savings/requestSizingV2`, {
        params: {
          filter: `label[team]="${team}"`,
          window,
        },
      }),
    );

    const data = response.data;
    return {
      cpuUtilization: data?.recommendations?.[0]?.cpuUtilization || 0,
      memoryUtilization: data?.recommendations?.[0]?.memoryUtilization || 0,
    };
  }

  private generateRecommendations(data: any, efficiency: any): string[] {
    const recommendations: string[] = [];

    if (efficiency.cpuUtilization < 30) {
      recommendations.push(
        `CPU utilization is only ${efficiency.cpuUtilization.toFixed(1)}%. ` +
        `Consider reducing CPU requests to save $${(data.cpuCost * 0.4).toFixed(2)}/week.`
      );
    }

    if (efficiency.memoryUtilization < 40) {
      recommendations.push(
        `Memory utilization is only ${efficiency.memoryUtilization.toFixed(1)}%. ` +
        `Reduce memory requests to save $${(data.ramCost * 0.3).toFixed(2)}/week.`
      );
    }

    if (data.idleCost > data.totalCost * 0.2) {
      recommendations.push(
        `High idle cost: $${data.idleCost.toFixed(2)}/week. ` +
        `Consider scaling down during off-peak hours.`
      );
    }

    return recommendations;
  }

  private getCostCenter(team: string): string {
    const costCenters: Record<string, string> = {
      payments: 'CC-001',
      logistics: 'CC-002',
      analytics: 'CC-003',
      platform: 'CC-004',
    };
    return costCenters[team] || 'CC-000';
  }

  private async sendChargebackEmail(report: TeamCostReport): Promise<void> {
    this.logger.log(`Sending chargeback report to team ${report.team}: $${report.costs.total.toFixed(2)}`);
  }

  private async saveReport(reports: TeamCostReport[]): Promise<void> {
    this.logger.log(`Saving ${reports.length} team cost reports`);
  }
}
```

---

## 6. Database Cost Optimization

```typescript
// db-optimization/src/database-cost.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { InjectDataSource } from '@nestjs/typeorm';
import { DataSource } from 'typeorm';

export interface DatabaseOptimizationReport {
  unusedIndexes: string[];
  largeTablesWithLowUsage: { table: string; sizeGb: number; readCount: number }[];
  connectionPoolRecommendations: string[];
  archiveSuggestions: { table: string; rowsOlderThan: string; estimatedSizeGb: number }[];
}

@Injectable()
export class DatabaseCostOptimizationService {
  private readonly logger = new Logger(DatabaseCostOptimizationService.name);

  constructor(
    @InjectDataSource() private readonly dataSource: DataSource,
  ) {}

  async analyzeDatabase(): Promise<DatabaseOptimizationReport> {
    const [unusedIndexes, largeTables, archiveSuggestions] = await Promise.all([
      this.findUnusedIndexes(),
      this.findLargeUnderusedTables(),
      this.findArchiveCandidates(),
    ]);

    return {
      unusedIndexes,
      largeTablesWithLowUsage: largeTables,
      connectionPoolRecommendations: await this.analyzeConnectionPool(),
      archiveSuggestions,
    };
  }

  private async findUnusedIndexes(): Promise<string[]> {
    const result = await this.dataSource.query(`
      SELECT
        schemaname || '.' || tablename || '.' || indexname AS index_name,
        pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
      FROM pg_stat_user_indexes
      WHERE idx_scan = 0
        AND indexrelname NOT LIKE 'pk_%'
        AND indexrelname NOT LIKE '%_pkey'
      ORDER BY pg_relation_size(indexrelid) DESC
      LIMIT 20;
    `);

    return result.map((r: any) => `${r.index_name} (${r.index_size})`);
  }

  private async findLargeUnderusedTables(): Promise<
    { table: string; sizeGb: number; readCount: number }[]
  > {
    const result = await this.dataSource.query(`
      SELECT
        schemaname || '.' || tablename AS table_name,
        pg_size_pretty(pg_total_relation_size(relid)) AS total_size,
        ROUND(pg_total_relation_size(relid) / 1073741824.0, 2) AS size_gb,
        seq_scan + idx_scan AS total_reads
      FROM pg_stat_user_tables
      WHERE pg_total_relation_size(relid) > 1073741824  -- > 1GB
        AND seq_scan + idx_scan < 1000  -- น้อยกว่า 1000 reads
      ORDER BY pg_total_relation_size(relid) DESC;
    `);

    return result.map((r: any) => ({
      table: r.table_name,
      sizeGb: r.size_gb,
      readCount: r.total_reads,
    }));
  }

  private async findArchiveCandidates(): Promise<
    { table: string; rowsOlderThan: string; estimatedSizeGb: number }[]
  > {
    // ตรวจสอบตารางที่มีข้อมูลเก่าๆ ที่สามารถ Archive ได้
    const tables = ['transactions', 'audit_logs', 'notifications', 'analytics_events'];
    const candidates = [];

    for (const table of tables) {
      try {
        const result = await this.dataSource.query(`
          SELECT 
            COUNT(*) as old_rows,
            pg_size_pretty(COUNT(*) * (SELECT avg_width FROM pg_stats WHERE tablename = $1 LIMIT 1)) as estimated_size
          FROM ${table}
          WHERE created_at < NOW() - INTERVAL '1 year'
        `, [table]);

        if (parseInt(result[0]?.old_rows || '0') > 100000) {
          candidates.push({
            table,
            rowsOlderThan: '1 year',
            estimatedSizeGb: 0,
          });
        }
      } catch {
        // ตารางอาจไม่มี created_at column
      }
    }

    return candidates;
  }

  private async analyzeConnectionPool(): Promise<string[]> {
    const result = await this.dataSource.query(`
      SELECT
        max_conn,
        used,
        res_for_super,
        max_conn - used - res_for_super AS free
      FROM
        (SELECT COUNT(*) used FROM pg_stat_activity) t1,
        (SELECT setting::int res_for_super FROM pg_settings WHERE name=$$superuser_reserved_connections$$) t2,
        (SELECT setting::int max_conn FROM pg_settings WHERE name=$$max_connections$$) t3;
    `);

    const recommendations = [];
    const { max_conn, used, free } = result[0];
    const usagePercent = (used / max_conn) * 100;

    if (usagePercent > 80) {
      recommendations.push(`High connection usage: ${usagePercent.toFixed(1)}%. Consider PgBouncer for connection pooling.`);
    }

    if (max_conn > 200 && usagePercent < 30) {
      recommendations.push(`max_connections=${max_conn} but only ${usagePercent.toFixed(1)}% used. Reduce max_connections to lower memory usage.`);
    }

    return recommendations;
  }

  // Data Archiving Strategy
  async archiveOldData(
    table: string,
    olderThanDays: number,
    archiveBucket: string,
  ): Promise<{ archivedRows: number; freedSpaceGb: number }> {
    this.logger.log(`Starting archive of ${table} (older than ${olderThanDays} days)`);

    const batchSize = 10000;
    let totalArchived = 0;
    let hasMore = true;

    while (hasMore) {
      // ดึงข้อมูลเก่า
      const rows = await this.dataSource.query(`
        SELECT * FROM ${table}
        WHERE created_at < NOW() - INTERVAL '${olderThanDays} days'
        ORDER BY created_at
        LIMIT ${batchSize}
      `);

      if (rows.length === 0) {
        hasMore = false;
        break;
      }

      // บันทึกไปยัง S3
      await this.uploadToS3(archiveBucket, `${table}/archive-${Date.now()}.json`, rows);

      // ลบออกจาก Database
      const ids = rows.map((r: any) => r.id);
      await this.dataSource.query(
        `DELETE FROM ${table} WHERE id = ANY($1)`,
        [ids],
      );

      totalArchived += rows.length;
      this.logger.log(`Archived ${totalArchived} rows from ${table}`);

      if (rows.length < batchSize) {
        hasMore = false;
      }
    }

    // VACUUM เพื่อคืนพื้นที่
    await this.dataSource.query(`VACUUM ANALYZE ${table}`);

    this.logger.log(`Archive complete: ${totalArchived} rows archived from ${table}`);

    return {
      archivedRows: totalArchived,
      freedSpaceGb: (totalArchived * 0.001) / 1024, // Estimate
    };
  }

  private async uploadToS3(bucket: string, key: string, data: any[]): Promise<void> {
    // อัปโหลดไปยัง S3 สำหรับ Cold Storage
    this.logger.debug(`Uploading ${data.length} records to s3://${bucket}/${key}`);
  }
}
```

---

## 7. Network Cost Reduction

```typescript
// network-optimization/src/network-cost.service.ts
import { Injectable, Logger } from '@nestjs/common';

export interface NetworkCostReport {
  egressByRegion: { region: string; gbTransferred: number; cost: number }[];
  intraClusterTraffic: number;
  crossAZTraffic: number;
  cdnSavings: number;
  recommendations: string[];
}

@Injectable()
export class NetworkCostOptimizationService {
  private readonly logger = new Logger(NetworkCostOptimizationService.name);

  // AWS Egress Pricing (USD/GB)
  private readonly EGRESS_PRICING = {
    intraRegion: 0.02,    // ภายใน Region เดียวกัน
    crossRegion: 0.09,    // ข้าม Region
    internet: 0.085,      // ออก Internet
  };

  async analyzeNetworkCosts(): Promise<NetworkCostReport> {
    const recommendations: string[] = [];

    // 1. ตรวจสอบ Cross-AZ Traffic
    const crossAZReduction = this.analyzeCrossAZTraffic();
    if (crossAZReduction > 0) {
      recommendations.push(
        `Topology-aware routing could save $${crossAZReduction.toFixed(2)}/month in cross-AZ traffic`
      );
    }

    // 2. ตรวจสอบ Unnecessary External Calls
    recommendations.push(
      'Enable Topology Aware Routing to prefer same-AZ endpoints'
    );

    recommendations.push(
      'Use VPC Endpoints for AWS services to avoid internet egress'
    );

    recommendations.push(
      'Enable CloudFront for static assets to reduce origin traffic'
    );

    return {
      egressByRegion: [],
      intraClusterTraffic: 0,
      crossAZTraffic: 0,
      cdnSavings: 0,
      recommendations,
    };
  }

  private analyzeCrossAZTraffic(): number {
    return 100; // Mock: $100/month potential savings
  }
}
```

```yaml
# k8s/topology-aware-routing.yaml
# Topology Aware Routing - ลด Cross-AZ Traffic
apiVersion: v1
kind: Service
metadata:
  name: payment-service
  namespace: production
  annotations:
    service.kubernetes.io/topology-mode: Auto  # Kubernetes 1.27+
spec:
  selector:
    app: payment-service
  ports:
    - port: 80
      targetPort: 3000
---
# สำหรับ Kubernetes รุ่นเก่า
apiVersion: v1
kind: Service
metadata:
  name: payment-service-legacy
  namespace: production
  annotations:
    service.kubernetes.io/topology-aware-hints: Auto
```

---

## 8. Storage Tier Optimization

```yaml
# storage/storage-classes.yaml
# Tiered Storage Classes
---
# Fast SSD สำหรับ Database
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs
parameters:
  type: io2
  iopsPerGB: "50"
  fsType: ext4
  encrypted: "true"
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
---
# Standard SSD สำหรับ General Use
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard-ssd
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  throughput: "125"
  iops: "3000"
  fsType: ext4
  encrypted: "true"
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
---
# HDD สำหรับ Log Storage (ราคาถูก)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: cheap-hdd
provisioner: kubernetes.io/aws-ebs
parameters:
  type: st1  # Throughput Optimized HDD
  fsType: ext4
  encrypted: "true"
reclaimPolicy: Delete
```

```typescript
// storage-lifecycle/src/s3-lifecycle.service.ts
import { Injectable, Logger } from '@nestjs/common';
import {
  S3Client,
  PutBucketLifecycleConfigurationCommand,
} from '@aws-sdk/client-s3';

@Injectable()
export class S3LifecycleService {
  private readonly logger = new Logger(S3LifecycleService.name);
  private readonly s3Client: S3Client;

  constructor() {
    this.s3Client = new S3Client({ region: process.env.AWS_REGION });
  }

  async setLifecyclePolicy(bucket: string): Promise<void> {
    const command = new PutBucketLifecycleConfigurationCommand({
      Bucket: bucket,
      LifecycleConfiguration: {
        Rules: [
          {
            ID: 'transition-to-ia',
            Status: 'Enabled',
            Filter: { Prefix: 'logs/' },
            Transitions: [
              {
                Days: 30,
                StorageClass: 'STANDARD_IA', // ลด 40% ค่าใช้จ่าย
              },
              {
                Days: 90,
                StorageClass: 'GLACIER_IR', // ลด 60% ค่าใช้จ่าย
              },
              {
                Days: 365,
                StorageClass: 'DEEP_ARCHIVE', // ลด 90% ค่าใช้จ่าย
              },
            ],
            Expiration: {
              Days: 2555, // ลบหลัง 7 ปี
            },
          },
          {
            ID: 'delete-incomplete-uploads',
            Status: 'Enabled',
            Filter: { Prefix: '' },
            AbortIncompleteMultipartUpload: {
              DaysAfterInitiation: 7,
            },
          },
          {
            ID: 'delete-old-versions',
            Status: 'Enabled',
            Filter: { Prefix: '' },
            NoncurrentVersionExpiration: {
              NoncurrentDays: 30,
            },
          },
        ],
      },
    });

    await this.s3Client.send(command);
    this.logger.log(`S3 lifecycle policy set for bucket: ${bucket}`);
  }
}
```

---

## 9. FinOps Dashboard

```typescript
// finops-dashboard/src/dashboard.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { Cron } from '@nestjs/schedule';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';

export interface DailyCloudCost {
  date: Date;
  team: string;
  service: string;
  compute: number;
  storage: number;
  network: number;
  total: number;
  budget: number;
  budgetUtilization: number;
}

@Injectable()
export class FinOpsDashboardService {
  private readonly logger = new Logger(FinOpsDashboardService.name);

  // Monthly Budgets per Team
  private readonly TEAM_BUDGETS: Record<string, number> = {
    payments: 5000,
    logistics: 3000,
    analytics: 2000,
    platform: 4000,
  };

  @Cron('0 7 * * *') // ทุกวัน 07:00
  async generateDailyCostReport(): Promise<void> {
    const yesterday = new Date();
    yesterday.setDate(yesterday.getDate() - 1);

    const costs = await this.getDailyCosts(yesterday);
    
    for (const cost of costs) {
      // แจ้งเตือนถ้า Budget เกิน 80%
      if (cost.budgetUtilization > 80) {
        await this.sendBudgetAlert(cost);
      }
    }

    await this.updateDashboard(costs);
    this.logger.log(`Daily cost report generated for ${yesterday.toISOString().split('T')[0]}`);
  }

  async getDailyCosts(date: Date): Promise<DailyCloudCost[]> {
    // ดึงจาก AWS Cost Explorer หรือ Kubecost
    const costs: DailyCloudCost[] = [];
    
    for (const [team, budget] of Object.entries(this.TEAM_BUDGETS)) {
      // Mock data - ในการใช้งานจริงดึงจาก API
      const dailyCost = budget / 30 * (0.8 + Math.random() * 0.4);
      const monthlyUsed = dailyCost * new Date().getDate();
      
      costs.push({
        date,
        team,
        service: `${team}-services`,
        compute: dailyCost * 0.6,
        storage: dailyCost * 0.25,
        network: dailyCost * 0.15,
        total: dailyCost,
        budget,
        budgetUtilization: (monthlyUsed / budget) * 100,
      });
    }
    
    return costs;
  }

  async getCostTrend(team: string, days: number = 30): Promise<{
    dates: string[];
    costs: number[];
    trend: 'INCREASING' | 'DECREASING' | 'STABLE';
    changePercent: number;
  }> {
    const dates: string[] = [];
    const costs: number[] = [];
    
    for (let i = days - 1; i >= 0; i--) {
      const date = new Date();
      date.setDate(date.getDate() - i);
      dates.push(date.toISOString().split('T')[0]);
      
      // Mock: ดึงจาก Database จริง
      costs.push(Math.random() * 200 + 100);
    }
    
    const firstHalf = costs.slice(0, Math.floor(days / 2));
    const secondHalf = costs.slice(Math.floor(days / 2));
    
    const firstAvg = firstHalf.reduce((a, b) => a + b, 0) / firstHalf.length;
    const secondAvg = secondHalf.reduce((a, b) => a + b, 0) / secondHalf.length;
    
    const changePercent = ((secondAvg - firstAvg) / firstAvg) * 100;
    
    let trend: 'INCREASING' | 'DECREASING' | 'STABLE';
    if (changePercent > 5) trend = 'INCREASING';
    else if (changePercent < -5) trend = 'DECREASING';
    else trend = 'STABLE';

    return { dates, costs, trend, changePercent };
  }

  private async sendBudgetAlert(cost: DailyCloudCost): Promise<void> {
    this.logger.warn(
      `Budget alert for team ${cost.team}: ${cost.budgetUtilization.toFixed(1)}% of monthly budget used`
    );
    // ส่ง Slack/Email notification
  }

  private async updateDashboard(costs: DailyCloudCost[]): Promise<void> {
    // อัปเดต Grafana Dashboard หรือ Custom Dashboard
    this.logger.log(`Dashboard updated with ${costs.length} team cost records`);
  }
}
```

---

## 10. Cost Monitoring with Kubecost

```yaml
# kubecost/values.yaml
kubecostToken: "YOUR_KUBECOST_TOKEN"

global:
  prometheus:
    enabled: false  # ใช้ Prometheus ที่มีอยู่แล้ว
    fqdn: http://prometheus-server.monitoring.svc.cluster.local

  grafana:
    enabled: false  # ใช้ Grafana ที่มีอยู่แล้ว
    proxy: false
    
kubecostProductConfigs:
  clusterName: production-cluster
  currencyCode: USD
  defaultIdle: true
  grafanaURL: http://grafana.monitoring.svc.cluster.local
  
  # AWS Spot Pricing Integration
  spotLabel: node-lifecycle
  spotLabelValue: spot
  
  # Custom Discount (ถ้ามี Reserved Instances)
  customPricesEnabled: true
  defaultModelPricing:
    enabled: true
    CPU: 0.03  # $/CPU-hour
    spotCPU: 0.009
    RAM: 0.01  # $/GB-hour
    spotRAM: 0.003
    storage: 0.00004  # $/GB-hour
    zoneNetworkEgress: 0.01  # $/GB

networkCosts:
  enabled: true
  podMonitor:
    enabled: true
  config:
    destinations:
      crossRegionDestinationCosts:
        - region: ap-southeast-1
          zoneName: ap-southeast-1a
          bandwidth: 0.02
      directClassificationServices:
        - "s3"
        - "dynamodb"

alerts:
  enabled: true
  slack:
    enabled: true
    webhook: ${SLACK_WEBHOOK_URL}
  email:
    enabled: true
    smtpAddress: smtp.example.com
    
  # Alert เมื่อ Budget เกิน
  budgetAlerts:
    - name: payments-team-budget
      namespace: team-payments
      monthlyBudget: 5000
      threshold: 80  # เปอร์เซ็นต์
```

---

## 11. FinOps Practices for Engineering Teams

```typescript
// finops-practices/src/engineering-finops.md
/**
 * FinOps Practices สำหรับ Engineering Teams
 * 
 * 1. Cost Awareness Culture
 *    - แสดงต้นทุนใน CI/CD Pipeline
 *    - Cost Review ใน Sprint Retrospective
 *    - ประมาณต้นทุนก่อน Deploy Feature ใหม่
 * 
 * 2. Engineering Practices
 *    - ตั้งค่า Resource Requests/Limits ทุก Pod
 *    - ใช้ HPA เพื่อ Scale ตามความต้องการ
 *    - ปิด Dev/Staging environments หลัง 18:00
 *    - ใช้ Spot/Preemptible สำหรับ Batch Jobs
 * 
 * 3. Architecture Decisions
 *    - เลือก Database ให้เหมาะกับ Workload
 *    - ใช้ Cache ลด Database Reads
 *    - Compress API Responses
 *    - ใช้ CDN สำหรับ Static Assets
 * 
 * 4. Monitoring & Optimization
 *    - ดู Cost Dashboard ทุกสัปดาห์
 *    - ตรวจสอบ Unused Resources ทุกเดือน
 *    - ทำ Right-sizing ทุก Quarter
 */

// ตัวอย่าง: Cost Estimation ใน CI/CD
interface DeploymentCostEstimate {
  serviceName: string;
  replicaCount: number;
  cpuRequest: string;
  memoryRequest: string;
  estimatedMonthlyCost: number;
  comparisonToPrevious?: {
    previousCost: number;
    change: number;
    changePercent: number;
  };
}

function estimateDeploymentCost(config: any): DeploymentCostEstimate {
  const CPU_COST_PER_CORE_MONTH = 30;
  const MEMORY_COST_PER_GB_MONTH = 5;
  
  const cpuCores = parseFloat(config.cpuRequest.replace('m', '')) / 1000;
  const memoryGb = parseFloat(config.memoryRequest.replace('Mi', '')) / 1024;
  
  const monthlyCost = 
    (cpuCores * CPU_COST_PER_CORE_MONTH * config.replicas) +
    (memoryGb * MEMORY_COST_PER_GB_MONTH * config.replicas);
  
  return {
    serviceName: config.name,
    replicaCount: config.replicas,
    cpuRequest: config.cpuRequest,
    memoryRequest: config.memoryRequest,
    estimatedMonthlyCost: monthlyCost,
  };
}
```

---

## สรุปบทที่ 98

| กลยุทธ์ | การประหยัดโดยประมาณ | ความซับซ้อน |
|---------|------------------|------------|
| **Right-sizing** | 20-40% ของ Compute | ต่ำ |
| **Spot Instances** | 60-80% ของ Batch Workloads | กลาง |
| **Reserved Capacity** | 30-60% ของ Baseline | กลาง |
| **Database Archiving** | 30-50% ของ Storage | กลาง |
| **Topology-aware Routing** | 10-20% ของ Network | ต่ำ |
| **Storage Lifecycle** | 50-80% ของ Log Storage | ต่ำ |
| **CDN Integration** | 30-50% ของ Egress | กลาง |
| **Cost Chargeback** | 10-20% ผ่าน Accountability | สูง |
| **FinOps Culture** | 15-25% ผ่าน Awareness | สูง |
| **รวมทั้งหมด** | **40-60%** ของต้นทุนรวม | - |

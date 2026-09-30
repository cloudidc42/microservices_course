# Part 59: Microservices Deployment Strategies

## บทนำ

ในโลกของ Microservices การ Deploy แอปพลิเคชันให้มีประสิทธิภาพ ปลอดภัย และลด Downtime เป็นศูนย์นั้นเป็นสิ่งสำคัญมาก บทนี้จะครอบคลุม Deployment Strategies ที่ใช้กันจริงในองค์กรระดับ Production รวมถึง Blue-Green Deployment, Canary Deployment, Rolling Updates, Feature Flags, Immutable Infrastructure, และ GitOps workflow

## 1. Blue-Green Deployment

Blue-Green Deployment เป็นกลยุทธ์ที่มีสอง Environment ทำงานพร้อมกัน (Blue = Current Production, Green = New Version) โดยเมื่อ Deploy เสร็จจะสลับ Traffic ทั้งหมดจาก Blue ไปยัง Green

### 1.1 Kubernetes Blue-Green Deployment

```yaml
# blue-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service-blue
  namespace: production
  labels:
    app: payment-service
    version: blue
    color: blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payment-service
      version: blue
  template:
    metadata:
      labels:
        app: payment-service
        version: blue
        color: blue
    spec:
      containers:
      - name: payment-service
        image: registry.example.com/payment-service:v1.2.0
        ports:
        - containerPort: 3000
        env:
        - name: NODE_ENV
          value: "production"
        - name: DB_HOST
          valueFrom:
            secretKeyRef:
              name: payment-secrets
              key: db-host
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 3
        livenessProbe:
          httpGet:
            path: /health/live
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
---
# green-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service-green
  namespace: production
  labels:
    app: payment-service
    version: green
    color: green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payment-service
      version: green
  template:
    metadata:
      labels:
        app: payment-service
        version: green
        color: green
    spec:
      containers:
      - name: payment-service
        image: registry.example.com/payment-service:v1.3.0
        ports:
        - containerPort: 3000
        env:
        - name: NODE_ENV
          value: "production"
        - name: DB_HOST
          valueFrom:
            secretKeyRef:
              name: payment-secrets
              key: db-host
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 3
        livenessProbe:
          httpGet:
            path: /health/live
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
---
# service.yaml - ชี้ไปที่ Blue (Current Active)
apiVersion: v1
kind: Service
metadata:
  name: payment-service
  namespace: production
  annotations:
    deployment.kubernetes.io/active-color: "blue"
spec:
  selector:
    app: payment-service
    color: blue  # เปลี่ยนเป็น green เมื่อต้องการ Switch
  ports:
  - protocol: TCP
    port: 80
    targetPort: 3000
  type: ClusterIP
```

### 1.2 TypeScript Blue-Green Deployment Controller

```typescript
// src/deployment/blue-green-controller.ts
import * as k8s from '@kubernetes/client-node';
import { logger } from '../utils/logger';
import { MetricsCollector } from '../monitoring/metrics';
import { AlertManager } from '../monitoring/alerts';

interface DeploymentConfig {
  namespace: string;
  serviceName: string;
  currentColor: 'blue' | 'green';
  newVersion: string;
  healthCheckTimeout: number;
  rollbackOnFailure: boolean;
}

interface HealthCheckResult {
  healthy: boolean;
  errorRate: number;
  p99Latency: number;
  availableReplicas: number;
}

export class BlueGreenController {
  private k8sApi: k8s.AppsV1Api;
  private k8sCoreApi: k8s.CoreV1Api;
  private metrics: MetricsCollector;
  private alerts: AlertManager;

  constructor(
    private config: DeploymentConfig,
    metrics: MetricsCollector,
    alerts: AlertManager
  ) {
    const kc = new k8s.KubeConfig();
    kc.loadFromDefault();
    this.k8sApi = kc.makeApiClient(k8s.AppsV1Api);
    this.k8sCoreApi = kc.makeApiClient(k8s.CoreV1Api);
    this.metrics = metrics;
    this.alerts = alerts;
  }

  async deploy(): Promise<{ success: boolean; activeColor: string }> {
    const targetColor = this.config.currentColor === 'blue' ? 'green' : 'blue';
    
    logger.info('Starting Blue-Green deployment', {
      service: this.config.serviceName,
      from: this.config.currentColor,
      to: targetColor,
      version: this.config.newVersion,
    });

    try {
      // Step 1: Deploy new version to inactive color
      await this.deployToInactiveEnvironment(targetColor);
      
      // Step 2: Wait for new pods to be ready
      await this.waitForPodsReady(targetColor);
      
      // Step 3: Run health checks on new environment
      const healthResult = await this.runHealthChecks(targetColor);
      
      if (!healthResult.healthy) {
        logger.error('Health checks failed on new environment', healthResult);
        await this.cleanupFailedDeployment(targetColor);
        throw new Error(`Health checks failed: error rate ${healthResult.errorRate}%`);
      }
      
      // Step 4: Switch traffic to new environment
      await this.switchTraffic(targetColor);
      
      // Step 5: Monitor post-switch
      const postSwitchHealth = await this.monitorPostSwitch(targetColor, 60000);
      
      if (!postSwitchHealth.healthy && this.config.rollbackOnFailure) {
        logger.warn('Post-switch monitoring failed, initiating rollback');
        await this.rollback(this.config.currentColor);
        return { success: false, activeColor: this.config.currentColor };
      }
      
      logger.info('Blue-Green deployment successful', {
        activeColor: targetColor,
        version: this.config.newVersion,
      });
      
      return { success: true, activeColor: targetColor };
      
    } catch (error) {
      logger.error('Blue-Green deployment failed', { error });
      
      if (this.config.rollbackOnFailure) {
        await this.rollback(this.config.currentColor);
      }
      
      throw error;
    }
  }

  private async deployToInactiveEnvironment(color: 'blue' | 'green'): Promise<void> {
    const deploymentName = `${this.config.serviceName}-${color}`;
    
    const patch = {
      spec: {
        template: {
          spec: {
            containers: [{
              name: this.config.serviceName,
              image: `registry.example.com/${this.config.serviceName}:${this.config.newVersion}`,
            }],
          },
        },
      },
    };
    
    await this.k8sApi.patchNamespacedDeployment(
      deploymentName,
      this.config.namespace,
      patch,
      undefined,
      undefined,
      undefined,
      undefined,
      undefined,
      { headers: { 'Content-Type': 'application/strategic-merge-patch+json' } }
    );
    
    logger.info(`Deployed ${this.config.newVersion} to ${color} environment`);
  }

  private async waitForPodsReady(
    color: 'blue' | 'green',
    timeoutMs = 300000
  ): Promise<void> {
    const deploymentName = `${this.config.serviceName}-${color}`;
    const startTime = Date.now();
    
    while (Date.now() - startTime < timeoutMs) {
      const deployment = await this.k8sApi.readNamespacedDeployment(
        deploymentName,
        this.config.namespace
      );
      
      const spec = deployment.body.spec?.replicas ?? 0;
      const ready = deployment.body.status?.readyReplicas ?? 0;
      
      logger.info(`Waiting for pods: ${ready}/${spec} ready`);
      
      if (ready >= spec) {
        logger.info(`All pods ready for ${color} environment`);
        return;
      }
      
      await this.sleep(5000);
    }
    
    throw new Error(`Timeout waiting for pods to be ready in ${color} environment`);
  }

  private async runHealthChecks(color: 'blue' | 'green'): Promise<HealthCheckResult> {
    const serviceUrl = `http://${this.config.serviceName}-${color}.${this.config.namespace}.svc.cluster.local`;
    
    // ทดสอบ endpoints ที่สำคัญของระบบชำระเงิน
    const testEndpoints = [
      { path: '/health/ready', expectedStatus: 200 },
      { path: '/api/v1/payments/status', expectedStatus: 200 },
      { path: '/api/v1/wallets/balance', expectedStatus: 200 },
    ];
    
    let successCount = 0;
    const latencies: number[] = [];
    
    for (const endpoint of testEndpoints) {
      try {
        const start = Date.now();
        const response = await fetch(`${serviceUrl}${endpoint.path}`, {
          signal: AbortSignal.timeout(5000),
        });
        const latency = Date.now() - start;
        latencies.push(latency);
        
        if (response.status === endpoint.expectedStatus) {
          successCount++;
        }
      } catch (error) {
        logger.warn(`Health check failed for ${endpoint.path}`, { error });
      }
    }
    
    const errorRate = ((testEndpoints.length - successCount) / testEndpoints.length) * 100;
    const p99Latency = this.calculatePercentile(latencies, 99);
    
    const deployment = await this.k8sApi.readNamespacedDeployment(
      `${this.config.serviceName}-${color}`,
      this.config.namespace
    );
    
    return {
      healthy: errorRate < 5 && p99Latency < 1000,
      errorRate,
      p99Latency,
      availableReplicas: deployment.body.status?.availableReplicas ?? 0,
    };
  }

  private async switchTraffic(targetColor: 'blue' | 'green'): Promise<void> {
    const patch = {
      spec: {
        selector: {
          color: targetColor,
        },
      },
      metadata: {
        annotations: {
          'deployment.kubernetes.io/active-color': targetColor,
          'deployment.kubernetes.io/switched-at': new Date().toISOString(),
        },
      },
    };
    
    await this.k8sCoreApi.patchNamespacedService(
      this.config.serviceName,
      this.config.namespace,
      patch,
      undefined,
      undefined,
      undefined,
      undefined,
      undefined,
      { headers: { 'Content-Type': 'application/strategic-merge-patch+json' } }
    );
    
    logger.info(`Traffic switched to ${targetColor} environment`);
    
    // Emit metric for monitoring
    this.metrics.increment('deployment.traffic_switch', {
      service: this.config.serviceName,
      color: targetColor,
      version: this.config.newVersion,
    });
  }

  private async monitorPostSwitch(
    activeColor: 'blue' | 'green',
    durationMs: number
  ): Promise<HealthCheckResult> {
    const checkInterval = 10000;
    const checks: HealthCheckResult[] = [];
    const endTime = Date.now() + durationMs;
    
    while (Date.now() < endTime) {
      const result = await this.runHealthChecks(activeColor);
      checks.push(result);
      
      if (result.errorRate > 10) {
        logger.error('Error rate exceeded threshold during post-switch monitoring', {
          errorRate: result.errorRate,
          threshold: 10,
        });
        return result;
      }
      
      await this.sleep(checkInterval);
    }
    
    const avgErrorRate = checks.reduce((sum, c) => sum + c.errorRate, 0) / checks.length;
    const avgLatency = checks.reduce((sum, c) => sum + c.p99Latency, 0) / checks.length;
    
    return {
      healthy: avgErrorRate < 5 && avgLatency < 1000,
      errorRate: avgErrorRate,
      p99Latency: avgLatency,
      availableReplicas: checks[checks.length - 1]?.availableReplicas ?? 0,
    };
  }

  async rollback(targetColor: 'blue' | 'green'): Promise<void> {
    logger.warn('Initiating rollback', {
      targetColor,
      service: this.config.serviceName,
    });
    
    await this.switchTraffic(targetColor);
    
    this.alerts.sendAlert({
      severity: 'warning',
      title: `Rollback initiated for ${this.config.serviceName}`,
      message: `Traffic rolled back to ${targetColor} environment`,
      service: this.config.serviceName,
    });
    
    logger.info('Rollback completed successfully');
  }

  private async cleanupFailedDeployment(color: 'blue' | 'green'): Promise<void> {
    const deploymentName = `${this.config.serviceName}-${color}`;
    
    // Rollback the failed deployment to previous image
    await this.k8sApi.createNamespacedDeploymentRollback(
      deploymentName,
      this.config.namespace,
      {
        name: deploymentName,
        rollbackTo: { revision: 0 }, // 0 = previous version
      }
    );
  }

  private calculatePercentile(values: number[], percentile: number): number {
    if (values.length === 0) return 0;
    const sorted = [...values].sort((a, b) => a - b);
    const index = Math.ceil((percentile / 100) * sorted.length) - 1;
    return sorted[index];
  }

  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}
```

## 2. Canary Deployment

Canary Deployment ค่อยๆ เพิ่ม Traffic ไปยัง New Version โดยเริ่มจากเปอร์เซ็นต์น้อยๆ และเพิ่มขึ้นเรื่อยๆ เมื่อมั่นใจว่า New Version ทำงานได้ปกติ

### 2.1 Istio-based Canary Deployment

```yaml
# canary-virtual-service.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: order-service-vs
  namespace: production
spec:
  hosts:
  - order-service
  http:
  - name: canary-route
    match:
    - headers:
        x-canary-user:
          exact: "true"
    route:
    - destination:
        host: order-service
        subset: canary
      weight: 100
  - name: main-route
    route:
    - destination:
        host: order-service
        subset: stable
      weight: 90  # 90% ไปที่ Stable
    - destination:
        host: order-service
        subset: canary
      weight: 10  # 10% ไปที่ Canary
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: order-service-dr
  namespace: production
spec:
  host: order-service
  subsets:
  - name: stable
    labels:
      version: stable
  - name: canary
    labels:
      version: canary
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 50
        http2MaxRequests: 100
    outlierDetection:
      consecutiveErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
```

### 2.2 Canary Deployment Automation

```typescript
// src/deployment/canary-controller.ts
import { PrometheusClient } from '../monitoring/prometheus';
import { KubernetesClient } from '../k8s/client';
import { logger } from '../utils/logger';

interface CanaryConfig {
  service: string;
  namespace: string;
  initialWeight: number;    // เริ่มต้นที่กี่ %
  incrementStep: number;    // เพิ่มทีละกี่ %
  maxWeight: number;        // สูงสุดที่กี่ %
  analysisInterval: number; // ตรวจสอบทุกกี่ milliseconds
  successThreshold: {
    errorRate: number;      // Error rate สูงสุดที่ยอมรับได้
    p99Latency: number;     // P99 Latency สูงสุดที่ยอมรับได้ (ms)
    successRate: number;    // Success rate ต่ำสุด
  };
}

interface CanaryMetrics {
  stableErrorRate: number;
  canaryErrorRate: number;
  stableP99Latency: number;
  canaryP99Latency: number;
  canaryRequestCount: number;
}

export class CanaryController {
  private prometheus: PrometheusClient;
  private k8s: KubernetesClient;
  private currentWeight: number;
  private isAnalysisRunning = false;

  constructor(
    private config: CanaryConfig,
    prometheus: PrometheusClient,
    k8s: KubernetesClient
  ) {
    this.prometheus = prometheus;
    this.k8s = k8s;
    this.currentWeight = config.initialWeight;
  }

  async startCanaryAnalysis(): Promise<'promote' | 'rollback'> {
    logger.info('Starting canary analysis', {
      service: this.config.service,
      initialWeight: this.config.initialWeight,
    });

    this.isAnalysisRunning = true;
    
    // ตั้งค่า Initial Canary Weight
    await this.updateCanaryWeight(this.currentWeight);

    while (this.isAnalysisRunning && this.currentWeight <= this.config.maxWeight) {
      await this.sleep(this.config.analysisInterval);
      
      const metrics = await this.collectMetrics();
      const analysis = this.analyzeMetrics(metrics);
      
      logger.info('Canary analysis result', {
        weight: this.currentWeight,
        metrics,
        analysis,
      });

      if (analysis.shouldRollback) {
        logger.error('Canary analysis failed, initiating rollback', {
          reason: analysis.reason,
          metrics,
        });
        await this.rollback();
        return 'rollback';
      }

      if (this.currentWeight >= this.config.maxWeight) {
        logger.info('Canary reached max weight, promoting to stable');
        await this.promote();
        return 'promote';
      }

      // เพิ่ม Weight ขึ้นเรื่อยๆ
      const newWeight = Math.min(
        this.currentWeight + this.config.incrementStep,
        this.config.maxWeight
      );
      
      await this.updateCanaryWeight(newWeight);
      this.currentWeight = newWeight;
      
      logger.info(`Increased canary weight to ${newWeight}%`);
    }

    return 'promote';
  }

  private async collectMetrics(): Promise<CanaryMetrics> {
    const timeRange = `${this.config.analysisInterval / 1000}s`;
    
    const [
      stableErrorRate,
      canaryErrorRate,
      stableP99Latency,
      canaryP99Latency,
      canaryRequestCount,
    ] = await Promise.all([
      this.prometheus.query(
        `sum(rate(http_requests_total{service="${this.config.service}",version="stable",status=~"5.."}[${timeRange}])) / sum(rate(http_requests_total{service="${this.config.service}",version="stable"}[${timeRange}])) * 100`
      ),
      this.prometheus.query(
        `sum(rate(http_requests_total{service="${this.config.service}",version="canary",status=~"5.."}[${timeRange}])) / sum(rate(http_requests_total{service="${this.config.service}",version="canary"}[${timeRange}])) * 100`
      ),
      this.prometheus.query(
        `histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket{service="${this.config.service}",version="stable"}[${timeRange}])) by (le)) * 1000`
      ),
      this.prometheus.query(
        `histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket{service="${this.config.service}",version="canary"}[${timeRange}])) by (le)) * 1000`
      ),
      this.prometheus.query(
        `sum(increase(http_requests_total{service="${this.config.service}",version="canary"}[${timeRange}]))`
      ),
    ]);

    return {
      stableErrorRate: parseFloat(stableErrorRate) || 0,
      canaryErrorRate: parseFloat(canaryErrorRate) || 0,
      stableP99Latency: parseFloat(stableP99Latency) || 0,
      canaryP99Latency: parseFloat(canaryP99Latency) || 0,
      canaryRequestCount: parseInt(canaryRequestCount) || 0,
    };
  }

  private analyzeMetrics(metrics: CanaryMetrics): {
    shouldRollback: boolean;
    reason?: string;
  } {
    const threshold = this.config.successThreshold;
    
    // ต้องมี Traffic เพียงพอก่อนตัดสิน
    if (metrics.canaryRequestCount < 100) {
      return { shouldRollback: false };
    }

    if (metrics.canaryErrorRate > threshold.errorRate) {
      return {
        shouldRollback: true,
        reason: `Canary error rate ${metrics.canaryErrorRate.toFixed(2)}% exceeds threshold ${threshold.errorRate}%`,
      };
    }

    if (metrics.canaryP99Latency > threshold.p99Latency) {
      return {
        shouldRollback: true,
        reason: `Canary P99 latency ${metrics.canaryP99Latency.toFixed(0)}ms exceeds threshold ${threshold.p99Latency}ms`,
      };
    }

    // เปรียบเทียบกับ Stable Version
    const errorRateIncrease = metrics.canaryErrorRate - metrics.stableErrorRate;
    if (errorRateIncrease > 2) {
      return {
        shouldRollback: true,
        reason: `Canary error rate increase ${errorRateIncrease.toFixed(2)}% compared to stable`,
      };
    }

    const latencyIncrease = metrics.canaryP99Latency - metrics.stableP99Latency;
    if (latencyIncrease > 200) {
      return {
        shouldRollback: true,
        reason: `Canary latency increase ${latencyIncrease.toFixed(0)}ms compared to stable`,
      };
    }

    return { shouldRollback: false };
  }

  private async updateCanaryWeight(weight: number): Promise<void> {
    await this.k8s.patchVirtualService({
      name: `${this.config.service}-vs`,
      namespace: this.config.namespace,
      stableWeight: 100 - weight,
      canaryWeight: weight,
    });
  }

  private async promote(): Promise<void> {
    logger.info('Promoting canary to stable', {
      service: this.config.service,
    });
    
    // อัพเดต Stable Deployment ด้วย Canary Image
    await this.k8s.promoteCanary({
      service: this.config.service,
      namespace: this.config.namespace,
    });
    
    // Reset weight กลับเป็น 0 สำหรับ Canary
    await this.updateCanaryWeight(0);
    this.isAnalysisRunning = false;
  }

  private async rollback(): Promise<void> {
    logger.warn('Rolling back canary deployment');
    
    await this.updateCanaryWeight(0);
    
    // Scale down canary deployment
    await this.k8s.scaleDeployment({
      name: `${this.config.service}-canary`,
      namespace: this.config.namespace,
      replicas: 0,
    });
    
    this.isAnalysisRunning = false;
  }

  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

// ตัวอย่างการใช้งาน
async function deployOrderServiceCanary() {
  const canaryController = new CanaryController(
    {
      service: 'order-service',
      namespace: 'production',
      initialWeight: 5,      // เริ่มที่ 5%
      incrementStep: 10,     // เพิ่มทีละ 10%
      maxWeight: 100,        // สูงสุด 100%
      analysisInterval: 300000, // วิเคราะห์ทุก 5 นาที
      successThreshold: {
        errorRate: 1.0,       // Error rate < 1%
        p99Latency: 500,      // P99 < 500ms
        successRate: 99.0,    // Success rate > 99%
      },
    },
    new PrometheusClient('http://prometheus:9090'),
    new KubernetesClient()
  );

  const result = await canaryController.startCanaryAnalysis();
  
  if (result === 'promote') {
    console.log('Canary deployment successful! New version is now stable.');
  } else {
    console.error('Canary deployment failed! Rolled back to previous version.');
  }
}
```

## 3. Rolling Updates

Rolling Update อัพเดต Pods ทีละกลุ่ม โดยไม่ Bring Down ทั้งหมดพร้อมกัน

### 3.1 Kubernetes Rolling Update Configuration

```yaml
# rolling-update-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-service
  namespace: production
spec:
  replicas: 6
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2          # สร้าง Pod ใหม่เพิ่มได้ 2 Pod
      maxUnavailable: 1    # Pod เก่าหยุดได้พร้อมกันไม่เกิน 1 Pod
  selector:
    matchLabels:
      app: product-service
  template:
    metadata:
      labels:
        app: product-service
    spec:
      terminationGracePeriodSeconds: 60
      containers:
      - name: product-service
        image: registry.example.com/product-service:v2.0.0
        ports:
        - containerPort: 3000
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]  # รอให้ Requests ที่ค้างอยู่จบก่อน
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 5
          successThreshold: 2
          failureThreshold: 3
        livenessProbe:
          httpGet:
            path: /health/live
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 15
          failureThreshold: 3
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
```

### 3.2 Graceful Shutdown Handler

```typescript
// src/server/graceful-shutdown.ts
import { Server } from 'http';
import { logger } from '../utils/logger';
import { DatabasePool } from '../database/pool';
import { MessageQueue } from '../queue/message-queue';

interface ShutdownConfig {
  gracePeriodMs: number;
  healthCheckPort: number;
}

export class GracefulShutdownHandler {
  private isShuttingDown = false;
  private activeConnections = new Set<any>();
  private server: Server | null = null;

  constructor(
    private config: ShutdownConfig,
    private db: DatabasePool,
    private queue: MessageQueue
  ) {}

  setupShutdownHandlers(server: Server): void {
    this.server = server;

    // Track active connections
    server.on('connection', (conn) => {
      this.activeConnections.add(conn);
      conn.on('close', () => this.activeConnections.delete(conn));
    });

    // Kubernetes ส่ง SIGTERM เมื่อต้องการหยุด Pod
    process.on('SIGTERM', () => this.shutdown('SIGTERM'));
    process.on('SIGINT', () => this.shutdown('SIGINT'));
    
    // จัดการ Uncaught Errors
    process.on('uncaughtException', (error) => {
      logger.error('Uncaught exception', { error });
      this.shutdown('uncaughtException');
    });
    
    process.on('unhandledRejection', (reason) => {
      logger.error('Unhandled rejection', { reason });
    });
  }

  isHealthy(): boolean {
    return !this.isShuttingDown;
  }

  private async shutdown(signal: string): Promise<void> {
    if (this.isShuttingDown) {
      logger.warn('Shutdown already in progress');
      return;
    }
    
    this.isShuttingDown = true;
    logger.info(`Received ${signal}, starting graceful shutdown`, {
      activeConnections: this.activeConnections.size,
    });

    // Step 1: หยุดรับ Requests ใหม่
    // Readiness Probe จะ Fail ทำให้ Load Balancer หยุดส่ง Traffic มา
    logger.info('Marking service as not ready (health check will fail)');

    // Step 2: รอให้ Kubernetes Load Balancer อัพเดต
    await this.sleep(15000); // รอ 15 วินาที

    // Step 3: หยุด HTTP Server
    if (this.server) {
      await new Promise<void>((resolve, reject) => {
        this.server!.close((err) => {
          if (err) reject(err);
          else resolve();
        });
      });
      logger.info('HTTP server stopped accepting new connections');
    }

    // Step 4: รอให้ Active Connections จบ
    const connectionWaitStart = Date.now();
    while (
      this.activeConnections.size > 0 &&
      Date.now() - connectionWaitStart < this.config.gracePeriodMs
    ) {
      logger.info(`Waiting for ${this.activeConnections.size} active connections to close`);
      await this.sleep(1000);
    }

    if (this.activeConnections.size > 0) {
      logger.warn(`Force closing ${this.activeConnections.size} remaining connections`);
      for (const conn of this.activeConnections) {
        conn.destroy();
      }
    }

    // Step 5: Drain Message Queue
    try {
      await this.queue.drain({ timeoutMs: 30000 });
      logger.info('Message queue drained successfully');
    } catch (error) {
      logger.error('Failed to drain message queue', { error });
    }

    // Step 6: Close Database connections
    try {
      await this.db.close();
      logger.info('Database connections closed');
    } catch (error) {
      logger.error('Failed to close database connections', { error });
    }

    logger.info('Graceful shutdown complete');
    process.exit(0);
  }

  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}
```

## 4. Feature Flags

Feature Flags ช่วยให้เราเปิด/ปิด Feature ได้โดยไม่ต้อง Deploy Code ใหม่

### 4.1 Feature Flag Service

```typescript
// src/feature-flags/feature-flag-service.ts
import Redis from 'ioredis';
import { logger } from '../utils/logger';

interface FlagConfig {
  name: string;
  enabled: boolean;
  rolloutPercentage: number;
  allowedUsers?: string[];
  allowedGroups?: string[];
  metadata?: Record<string, unknown>;
}

interface EvaluationContext {
  userId: string;
  userGroup?: string;
  countryCode?: string;
  deviceType?: string;
  appVersion?: string;
}

export class FeatureFlagService {
  private redis: Redis;
  private localCache = new Map<string, { config: FlagConfig; expiresAt: number }>();
  private cacheTtlMs = 60000; // Cache 1 นาที

  constructor(private redisClient: Redis) {
    this.redis = redisClient;
  }

  async isEnabled(
    flagName: string,
    context: EvaluationContext
  ): Promise<boolean> {
    try {
      const config = await this.getFlag(flagName);
      
      if (!config) {
        logger.warn(`Feature flag not found: ${flagName}`);
        return false;
      }

      if (!config.enabled) {
        return false;
      }

      // ตรวจสอบ Allowed Users
      if (config.allowedUsers && config.allowedUsers.length > 0) {
        if (config.allowedUsers.includes(context.userId)) {
          return true;
        }
      }

      // ตรวจสอบ Allowed Groups
      if (config.allowedGroups && config.allowedGroups.length > 0) {
        if (context.userGroup && config.allowedGroups.includes(context.userGroup)) {
          return true;
        }
      }

      // Percentage-based Rollout
      if (config.rolloutPercentage < 100) {
        const userHash = this.hashUserId(context.userId, flagName);
        const userPercentile = (userHash % 100) + 1;
        return userPercentile <= config.rolloutPercentage;
      }

      return true;
      
    } catch (error) {
      logger.error('Error evaluating feature flag', { flagName, error });
      return false; // Default to disabled on error
    }
  }

  async setFlag(flagName: string, config: Partial<FlagConfig>): Promise<void> {
    const existing = await this.getFlag(flagName);
    const updated: FlagConfig = {
      name: flagName,
      enabled: false,
      rolloutPercentage: 0,
      ...existing,
      ...config,
    };
    
    await this.redis.set(
      `feature_flag:${flagName}`,
      JSON.stringify(updated),
      'EX',
      86400 // 24 ชั่วโมง
    );
    
    // Clear local cache
    this.localCache.delete(flagName);
    
    // Publish change for other instances
    await this.redis.publish(
      'feature_flag_changes',
      JSON.stringify({ flagName, config: updated })
    );
    
    logger.info('Feature flag updated', { flagName, config: updated });
  }

  async enableForPercentage(flagName: string, percentage: number): Promise<void> {
    await this.setFlag(flagName, {
      enabled: true,
      rolloutPercentage: Math.min(100, Math.max(0, percentage)),
    });
  }

  async enableForUsers(flagName: string, userIds: string[]): Promise<void> {
    await this.setFlag(flagName, {
      enabled: true,
      rolloutPercentage: 0,
      allowedUsers: userIds,
    });
  }

  private async getFlag(flagName: string): Promise<FlagConfig | null> {
    // Check local cache first
    const cached = this.localCache.get(flagName);
    if (cached && cached.expiresAt > Date.now()) {
      return cached.config;
    }

    const data = await this.redis.get(`feature_flag:${flagName}`);
    
    if (!data) {
      return null;
    }

    const config = JSON.parse(data) as FlagConfig;
    
    // Update local cache
    this.localCache.set(flagName, {
      config,
      expiresAt: Date.now() + this.cacheTtlMs,
    });
    
    return config;
  }

  private hashUserId(userId: string, salt: string): number {
    const str = `${userId}:${salt}`;
    let hash = 0;
    
    for (let i = 0; i < str.length; i++) {
      const char = str.charCodeAt(i);
      hash = ((hash << 5) - hash) + char;
      hash = hash & hash; // Convert to 32-bit integer
    }
    
    return Math.abs(hash);
  }

  subscribeToChanges(callback: (flagName: string, config: FlagConfig) => void): void {
    const subscriber = this.redis.duplicate();
    
    subscriber.subscribe('feature_flag_changes', (err) => {
      if (err) {
        logger.error('Failed to subscribe to feature flag changes', { error: err });
      }
    });
    
    subscriber.on('message', (channel, message) => {
      if (channel === 'feature_flag_changes') {
        const { flagName, config } = JSON.parse(message);
        this.localCache.delete(flagName); // Invalidate cache
        callback(flagName, config);
      }
    });
  }
}

// ตัวอย่างการใช้งานใน E-commerce ไทย
class OrderController {
  constructor(
    private featureFlags: FeatureFlagService,
    private orderService: any
  ) {}

  async createOrder(userId: string, orderData: any) {
    const context: EvaluationContext = {
      userId,
      userGroup: orderData.userTier, // 'vip', 'regular', 'new'
      countryCode: 'TH',
    };

    // ตรวจสอบว่า Feature ใหม่เปิดใช้งานหรือไม่
    const useNewCheckout = await this.featureFlags.isEnabled(
      'new_checkout_flow',
      context
    );

    const useInstantPayment = await this.featureFlags.isEnabled(
      'promptpay_instant_payment',
      context
    );

    const useAIRecommendation = await this.featureFlags.isEnabled(
      'ai_product_recommendation',
      context
    );

    if (useNewCheckout) {
      return this.orderService.createWithNewFlow(orderData, {
        instantPayment: useInstantPayment,
        aiRecommendation: useAIRecommendation,
      });
    }

    return this.orderService.createWithLegacyFlow(orderData);
  }
}
```

## 5. GitOps Workflow with ArgoCD

### 5.1 ArgoCD Application Configuration

```yaml
# argocd/applications/payment-service.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: payment-service
  namespace: argocd
  finalizers:
  - resources-finalizer.argocd.argoproj.io
  annotations:
    notifications.argoproj.io/subscribe.on-sync-succeeded.slack: deployment-alerts
    notifications.argoproj.io/subscribe.on-sync-failed.slack: deployment-alerts
    notifications.argoproj.io/subscribe.on-health-degraded.slack: deployment-alerts
spec:
  project: production
  source:
    repoURL: https://github.com/company/k8s-configs
    targetRevision: HEAD
    path: services/payment-service/production
    helm:
      valueFiles:
      - values-production.yaml
      parameters:
      - name: image.tag
        value: "v1.3.0"
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true      # ลบ Resources ที่ไม่อยู่ใน Git แล้ว
      selfHeal: true   # Auto-sync เมื่อ State ต่างจาก Git
      allowEmpty: false
    syncOptions:
    - CreateNamespace=true
    - PrunePropagationPolicy=foreground
    - PruneLast=true
    - ApplyOutOfSyncOnly=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
  revisionHistoryLimit: 10
---
# argocd/projects/production.yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: production
  namespace: argocd
spec:
  description: Production environment project
  sourceRepos:
  - https://github.com/company/k8s-configs
  - https://charts.helm.sh/stable
  destinations:
  - namespace: production
    server: https://kubernetes.default.svc
  - namespace: monitoring
    server: https://kubernetes.default.svc
  clusterResourceWhitelist:
  - group: ''
    kind: Namespace
  namespaceResourceBlacklist:
  - group: ''
    kind: ResourceQuota
  roles:
  - name: developer
    description: Developer role with limited access
    policies:
    - p, proj:production:developer, applications, get, production/*, allow
    - p, proj:production:developer, applications, sync, production/*, allow
    groups:
    - company:developers
  - name: deployer
    description: CI/CD deployer role
    policies:
    - p, proj:production:deployer, applications, *, production/*, allow
    groups:
    - company:cicd
```

### 5.2 GitOps Deployment Pipeline

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production via GitOps

on:
  push:
    tags:
      - 'v*'

env:
  REGISTRY: registry.company.com
  SERVICE_NAME: payment-service
  K8S_CONFIG_REPO: company/k8s-configs

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      version: ${{ steps.get-version.outputs.version }}
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Get version
      id: get-version
      run: echo "version=${GITHUB_REF#refs/tags/}" >> $GITHUB_OUTPUT
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3
    
    - name: Login to Registry
      uses: docker/login-action@v3
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ secrets.REGISTRY_USERNAME }}
        password: ${{ secrets.REGISTRY_PASSWORD }}
    
    - name: Build and push
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: |
          ${{ env.REGISTRY }}/${{ env.SERVICE_NAME }}:${{ steps.get-version.outputs.version }}
          ${{ env.REGISTRY }}/${{ env.SERVICE_NAME }}:latest
        cache-from: type=gha
        cache-to: type=gha,mode=max
        build-args: |
          BUILD_DATE=${{ github.event.repository.updated_at }}
          GIT_COMMIT=${{ github.sha }}
          VERSION=${{ steps.get-version.outputs.version }}

  update-k8s-config:
    needs: build-and-push
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout K8s config repo
      uses: actions/checkout@v4
      with:
        repository: ${{ env.K8S_CONFIG_REPO }}
        token: ${{ secrets.K8S_CONFIG_PAT }}
    
    - name: Update image tag
      run: |
        cd services/${{ env.SERVICE_NAME }}/production
        
        # อัพเดต Helm values
        yq eval '.image.tag = "${{ needs.build-and-push.outputs.version }}"' \
          -i values-production.yaml
        
        # อัพเดต Kustomize (ถ้าใช้)
        kustomize edit set image \
          ${{ env.REGISTRY }}/${{ env.SERVICE_NAME }}:${{ needs.build-and-push.outputs.version }}
    
    - name: Commit and push changes
      run: |
        git config user.name "GitHub Actions Bot"
        git config user.email "bot@company.com"
        git add .
        git commit -m "chore: update ${{ env.SERVICE_NAME }} to ${{ needs.build-and-push.outputs.version }}
        
        Automated deployment triggered by release ${{ needs.build-and-push.outputs.version }}
        Build: ${{ github.run_id }}
        Commit: ${{ github.sha }}"
        git push
    
    - name: Wait for ArgoCD sync
      run: |
        # รอให้ ArgoCD Sync เสร็จ
        for i in $(seq 1 30); do
          STATUS=$(argocd app get ${{ env.SERVICE_NAME }} \
            --server ${{ secrets.ARGOCD_SERVER }} \
            --auth-token ${{ secrets.ARGOCD_TOKEN }} \
            --output json | jq -r '.status.sync.status')
          
          HEALTH=$(argocd app get ${{ env.SERVICE_NAME }} \
            --server ${{ secrets.ARGOCD_SERVER }} \
            --auth-token ${{ secrets.ARGOCD_TOKEN }} \
            --output json | jq -r '.status.health.status')
          
          echo "Sync: $STATUS, Health: $HEALTH"
          
          if [ "$STATUS" = "Synced" ] && [ "$HEALTH" = "Healthy" ]; then
            echo "Deployment successful!"
            exit 0
          fi
          
          sleep 30
        done
        
        echo "Deployment timed out or failed"
        exit 1
```

## 6. Health Checks

### 6.1 Comprehensive Health Check Implementation

```typescript
// src/health/health-check-service.ts
import { Router, Request, Response } from 'express';
import { DatabasePool } from '../database/pool';
import { Redis } from 'ioredis';
import { MessageQueue } from '../queue/message-queue';

interface HealthCheckResult {
  status: 'healthy' | 'degraded' | 'unhealthy';
  timestamp: string;
  version: string;
  uptime: number;
  checks: Record<string, ComponentHealth>;
}

interface ComponentHealth {
  status: 'up' | 'down' | 'degraded';
  latencyMs?: number;
  message?: string;
  details?: Record<string, unknown>;
}

export class HealthCheckService {
  private startTime = Date.now();

  constructor(
    private db: DatabasePool,
    private redis: Redis,
    private queue: MessageQueue,
    private version: string
  ) {}

  createRouter(): Router {
    const router = Router();

    // Liveness probe - Pod ยังทำงานอยู่หรือไม่
    router.get('/health/live', this.livenessCheck.bind(this));

    // Readiness probe - Pod พร้อมรับ Traffic หรือไม่
    router.get('/health/ready', this.readinessCheck.bind(this));

    // Full health check - สำหรับ Monitoring
    router.get('/health', this.fullHealthCheck.bind(this));

    // Startup probe - สำหรับ Container เพิ่งเริ่มต้น
    router.get('/health/startup', this.startupCheck.bind(this));

    return router;
  }

  private async livenessCheck(req: Request, res: Response): Promise<void> {
    // Liveness check ควรเบาที่สุด - แค่บอกว่า Process ยังทำงานอยู่
    res.json({
      status: 'alive',
      timestamp: new Date().toISOString(),
      uptime: Math.floor((Date.now() - this.startTime) / 1000),
    });
  }

  private async readinessCheck(req: Request, res: Response): Promise<void> {
    const checks: Record<string, ComponentHealth> = {};
    let isReady = true;

    // ตรวจสอบ Database
    try {
      const dbStart = Date.now();
      await this.db.query('SELECT 1');
      checks.database = {
        status: 'up',
        latencyMs: Date.now() - dbStart,
      };
    } catch (error) {
      checks.database = {
        status: 'down',
        message: 'Database connection failed',
      };
      isReady = false;
    }

    // ตรวจสอบ Redis
    try {
      const redisStart = Date.now();
      await this.redis.ping();
      checks.redis = {
        status: 'up',
        latencyMs: Date.now() - redisStart,
      };
    } catch (error) {
      checks.redis = {
        status: 'down',
        message: 'Redis connection failed',
      };
      isReady = false;
    }

    const statusCode = isReady ? 200 : 503;
    res.status(statusCode).json({
      status: isReady ? 'ready' : 'not ready',
      timestamp: new Date().toISOString(),
      checks,
    });
  }

  private async fullHealthCheck(req: Request, res: Response): Promise<void> {
    const checks: Record<string, ComponentHealth> = {};
    
    // ตรวจสอบ Database พร้อม Connection Pool Info
    try {
      const dbStart = Date.now();
      const poolStats = await this.db.getPoolStats();
      await this.db.query('SELECT 1');
      checks.database = {
        status: 'up',
        latencyMs: Date.now() - dbStart,
        details: {
          totalConnections: poolStats.total,
          activeConnections: poolStats.active,
          idleConnections: poolStats.idle,
          waitingClients: poolStats.waiting,
        },
      };
    } catch (error) {
      checks.database = { status: 'down', message: String(error) };
    }

    // ตรวจสอบ Redis
    try {
      const redisStart = Date.now();
      const info = await this.redis.info('memory');
      await this.redis.ping();
      const memMatch = info.match(/used_memory_human:(\S+)/);
      checks.redis = {
        status: 'up',
        latencyMs: Date.now() - redisStart,
        details: {
          memoryUsed: memMatch ? memMatch[1] : 'unknown',
        },
      };
    } catch (error) {
      checks.redis = { status: 'down', message: String(error) };
    }

    // ตรวจสอบ External Payment Gateway (PromptPay, etc.)
    try {
      const gatewayStart = Date.now();
      const response = await fetch('https://api.payment-gateway.th/health', {
        signal: AbortSignal.timeout(3000),
      });
      checks.paymentGateway = {
        status: response.ok ? 'up' : 'degraded',
        latencyMs: Date.now() - gatewayStart,
        details: { statusCode: response.status },
      };
    } catch (error) {
      checks.paymentGateway = {
        status: 'degraded',
        message: 'Payment gateway health check failed',
      };
    }

    // ตรวจสอบ Message Queue
    try {
      const queueStats = await this.queue.getStats();
      checks.messageQueue = {
        status: queueStats.consumerCount > 0 ? 'up' : 'degraded',
        details: {
          messageCount: queueStats.messageCount,
          consumerCount: queueStats.consumerCount,
          publishRate: queueStats.publishRate,
          consumeRate: queueStats.consumeRate,
        },
      };
    } catch (error) {
      checks.messageQueue = { status: 'down', message: String(error) };
    }

    const hasDown = Object.values(checks).some(c => c.status === 'down');
    const hasDegraded = Object.values(checks).some(c => c.status === 'degraded');
    
    const overallStatus: HealthCheckResult['status'] = hasDown
      ? 'unhealthy'
      : hasDegraded
      ? 'degraded'
      : 'healthy';

    const result: HealthCheckResult = {
      status: overallStatus,
      timestamp: new Date().toISOString(),
      version: this.version,
      uptime: Math.floor((Date.now() - this.startTime) / 1000),
      checks,
    };

    const statusCode = overallStatus === 'unhealthy' ? 503 : 200;
    res.status(statusCode).json(result);
  }

  private async startupCheck(req: Request, res: Response): Promise<void> {
    // ตรวจสอบว่า Application เริ่มต้นสำเร็จแล้ว
    // ใช้สำหรับ initialDelaySeconds ที่ยาวนาน
    const isStarted = Date.now() - this.startTime > 5000; // รอ 5 วินาที
    
    if (isStarted) {
      res.json({ status: 'started' });
    } else {
      res.status(503).json({ status: 'starting' });
    }
  }
}
```

## 7. Immutable Infrastructure

### 7.1 Dockerfile สำหรับ Immutable Container

```dockerfile
# Dockerfile - Multi-stage build สำหรับ Production
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production && \
    # ทำความสะอาดเพื่อลดขนาด
    npm cache clean --force

FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build && \
    npm prune --production

# Production image - Immutable
FROM node:20-alpine AS production
WORKDIR /app

# Security: ใช้ Non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

# Copy built files เท่านั้น
COPY --from=builder --chown=nodejs:nodejs /app/dist ./dist
COPY --from=builder --chown=nodejs:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nodejs:nodejs /app/package.json ./package.json

# Security: Read-only filesystem
USER nodejs

# Build arguments สำหรับ metadata
ARG BUILD_DATE
ARG GIT_COMMIT
ARG VERSION

# Labels สำหรับ Traceability
LABEL org.opencontainers.image.created="${BUILD_DATE}" \
      org.opencontainers.image.revision="${GIT_COMMIT}" \
      org.opencontainers.image.version="${VERSION}" \
      org.opencontainers.image.title="payment-service" \
      org.opencontainers.image.vendor="Company Name"

ENV NODE_ENV=production \
    PORT=3000 \
    # ไม่อนุญาตให้แก้ไข Environment Variables หลัง Deploy
    NODE_OPTIONS="--max-old-space-size=512"

EXPOSE 3000

# Healthcheck ใน Container เอง
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
  CMD wget -qO- http://localhost:3000/health/live || exit 1

ENTRYPOINT ["node", "dist/server.js"]
```

## สรุป

| กลยุทธ์ | ข้อดี | ข้อเสีย | เหมาะกับ |
|---------|-------|---------|---------|
| Blue-Green | Instant rollback, Zero downtime | ใช้ Resources 2 เท่า | Critical services เช่น Payment |
| Canary | ลด Risk, Gradual rollout | ซับซ้อน, ต้องมี Monitoring ดี | Feature rollout ใหม่ |
| Rolling Update | ประหยัด Resources | Rollback ช้ากว่า | Stateless services ทั่วไป |
| Feature Flags | Deploy ได้โดยไม่ Release | Code complexity เพิ่ม | A/B Testing, Gradual release |
| GitOps (ArgoCD) | Audit trail, Self-healing | Learning curve สูง | Team ขนาดกลาง-ใหญ่ |
| Immutable Infra | Predictable, Secure | Build time นานขึ้น | Production environments |
| Health Checks | Early failure detection | ต้องออกแบบให้ดี | ทุก Microservice |

การเลือกใช้กลยุทธ์ที่เหมาะสมขึ้นอยู่กับความต้องการของระบบ ความสำคัญของ Service และความพร้อมของทีม ระบบ E-commerce และ Fintech ไทยควรใช้ Blue-Green สำหรับ Payment Service และ Canary สำหรับ Feature ใหม่ๆ เพื่อลดความเสี่ยง

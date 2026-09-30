# Part 80: Kubernetes Patterns

## บทนำ

Kubernetes มี patterns หลากหลายที่ช่วยจัดการ workloads ที่ซับซ้อน บทนี้ครอบคลุม StatefulSet, DaemonSet, Jobs, CRDs, Operators, Controllers, Admission Webhooks, API Aggregation, Multi-tenancy และ GitOps กับ ArgoCD

---

## 1. StatefulSet vs Deployment Patterns

```typescript
// statefulset-concepts.ts
// เข้าใจความต่างระหว่าง StatefulSet และ Deployment

interface WorkloadCharacteristic {
  feature: string;
  deployment: string;
  statefulSet: string;
  recommendation: string;
}

const workloadComparison: WorkloadCharacteristic[] = [
  {
    feature: 'Pod Identity',
    deployment: 'Random (pod-xyz123)',
    statefulSet: 'Stable (pod-0, pod-1, pod-2)',
    recommendation: 'Use StatefulSet when pods need stable identity',
  },
  {
    feature: 'Storage',
    deployment: 'Ephemeral or shared volume',
    statefulSet: 'Dedicated PVC per pod',
    recommendation: 'Use StatefulSet for databases',
  },
  {
    feature: 'Network',
    deployment: 'ClusterIP (load balanced)',
    statefulSet: 'Headless service (individual DNS)',
    recommendation: 'StatefulSet for clustered apps needing peer discovery',
  },
  {
    feature: 'Scaling',
    deployment: 'Parallel (all at once)',
    statefulSet: 'Sequential (0,1,2...)',
    recommendation: 'StatefulSet for ordered startup/shutdown',
  },
  {
    feature: 'Updates',
    deployment: 'Rolling update (any order)',
    statefulSet: 'Ordered rolling update (N-1, N-2...)',
    recommendation: 'StatefulSet for strict update ordering',
  },
];
```

```yaml
# statefulset-postgres.yaml
# PostgreSQL ด้วย StatefulSet

apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgresql
  namespace: database
  labels:
    app: postgresql
spec:
  serviceName: postgresql-headless
  replicas: 3
  podManagementPolicy: OrderedReady
  
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0  # Update ทุก pod
  
  selector:
    matchLabels:
      app: postgresql
  
  template:
    metadata:
      labels:
        app: postgresql
    spec:
      terminationGracePeriodSeconds: 60
      
      initContainers:
        # Setup replication based on ordinal
        - name: init-replication
          image: postgres:15
          command:
            - /bin/bash
            - -c
            - |
              ORDINAL=${HOSTNAME##*-}
              if [ "$ORDINAL" = "0" ]; then
                echo "primary" > /data/role
              else
                echo "replica" > /data/role
                # Wait for primary
                until pg_isready -h postgresql-0.postgresql-headless; do sleep 2; done
              fi
          volumeMounts:
            - name: data
              mountPath: /data
      
      containers:
        - name: postgresql
          image: postgres:15
          env:
            - name: POSTGRES_DB
              value: orderdb
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: username
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
          
          ports:
            - containerPort: 5432
          
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
            - name: config
              mountPath: /etc/postgresql/postgresql.conf
              subPath: postgresql.conf
          
          resources:
            requests:
              memory: "1Gi"
              cpu: "500m"
            limits:
              memory: "4Gi"
              cpu: "2000m"
          
          livenessProbe:
            exec:
              command: ["pg_isready", "-U", "postgres"]
            initialDelaySeconds: 30
            periodSeconds: 10
          
          readinessProbe:
            exec:
              command: ["pg_isready", "-U", "postgres"]
            initialDelaySeconds: 5
            periodSeconds: 5
  
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: fast-ssd
        resources:
          requests:
            storage: 100Gi
---
# Headless Service สำหรับ Pod DNS
apiVersion: v1
kind: Service
metadata:
  name: postgresql-headless
  namespace: database
spec:
  clusterIP: None  # Headless
  selector:
    app: postgresql
  ports:
    - port: 5432
      targetPort: 5432
---
# Regular Service สำหรับ Read/Write
apiVersion: v1
kind: Service
metadata:
  name: postgresql-primary
  namespace: database
spec:
  selector:
    app: postgresql
    role: primary
  ports:
    - port: 5432
---
apiVersion: v1
kind: Service
metadata:
  name: postgresql-replica
  namespace: database
spec:
  selector:
    app: postgresql
    role: replica
  ports:
    - port: 5432
```

---

## 2. DaemonSet for Infrastructure

```yaml
# daemonset-patterns.yaml
# DaemonSet สำหรับ infrastructure workloads

# Node log collector
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-collector
  namespace: kube-system
  labels:
    app: fluentd
spec:
  selector:
    matchLabels:
      app: fluentd
  
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      serviceAccountName: fluentd
      tolerations:
        # รัน บน master nodes ด้วย
        - key: node-role.kubernetes.io/control-plane
          effect: NoSchedule
        # รัน บน nodes ที่มี special taint
        - key: node-type
          operator: Equal
          value: gpu
          effect: NoSchedule
      
      containers:
        - name: fluentd
          image: fluent/fluentd-kubernetes-daemonset:v1.16
          
          env:
            - name: FLUENT_ELASTICSEARCH_HOST
              value: elasticsearch.logging.svc.cluster.local
            - name: FLUENT_ELASTICSEARCH_PORT
              value: "9200"
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
          
          resources:
            requests:
              memory: "200Mi"
              cpu: "100m"
            limits:
              memory: "500Mi"
              cpu: "500m"
          
          volumeMounts:
            - name: varlog
              mountPath: /var/log
            - name: varlibdockercontainers
              mountPath: /var/lib/docker/containers
              readOnly: true
            - name: config
              mountPath: /fluentd/etc
      
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
        - name: varlibdockercontainers
          hostPath:
            path: /var/lib/docker/containers
        - name: config
          configMap:
            name: fluentd-config
      
      terminationGracePeriodSeconds: 30
---
# Node exporter สำหรับ Prometheus metrics
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      hostNetwork: true
      hostPID: true
      tolerations:
        - effect: NoSchedule
          operator: Exists
      containers:
        - name: node-exporter
          image: prom/node-exporter:latest
          args:
            - '--path.sysfs=/host/sys'
            - '--path.procfs=/host/proc'
            - '--collector.filesystem.mount-points-exclude=^/(dev|proc|sys|var/lib/docker/.+)($|/)'
          ports:
            - name: metrics
              containerPort: 9100
              hostPort: 9100
          volumeMounts:
            - name: proc
              mountPath: /host/proc
              readOnly: true
            - name: sys
              mountPath: /host/sys
              readOnly: true
          resources:
            requests:
              memory: "30Mi"
              cpu: "50m"
            limits:
              memory: "100Mi"
              cpu: "200m"
      volumes:
        - name: proc
          hostPath:
            path: /proc
        - name: sys
          hostPath:
            path: /sys
```

---

## 3. Job and CronJob Patterns

```yaml
# job-patterns.yaml

# Batch processing Job
apiVersion: batch/v1
kind: Job
metadata:
  name: data-migration-v2
  namespace: production
  labels:
    app: data-migration
    version: v2
spec:
  # Retry ถ้า job ล้มเหลว
  backoffLimit: 3
  # ลบ job pod หลัง complete
  ttlSecondsAfterFinished: 3600
  # Timeout
  activeDeadlineSeconds: 3600
  
  # Parallel processing
  parallelism: 5
  completions: 100
  completionMode: Indexed  # Each pod gets unique index
  
  template:
    metadata:
      labels:
        app: data-migration
    spec:
      restartPolicy: OnFailure
      
      containers:
        - name: migrator
          image: data-migrator:v2
          command: ["/app/migrate"]
          args:
            - "--batch-index=$(JOB_COMPLETION_INDEX)"
            - "--total-batches=100"
          env:
            - name: JOB_COMPLETION_INDEX
              valueFrom:
                fieldRef:
                  fieldPath: metadata.annotations['batch.kubernetes.io/job-completion-index']
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: url
          resources:
            requests:
              memory: "256Mi"
              cpu: "200m"
            limits:
              memory: "512Mi"
              cpu: "500m"
---
# CronJob สำหรับ scheduled tasks
apiVersion: batch/v1
kind: CronJob
metadata:
  name: order-report-generator
  namespace: production
spec:
  # รันทุกวัน 2am Bangkok time
  schedule: "0 19 * * *"  # UTC (Bangkok = UTC+7)
  timeZone: "Asia/Bangkok"
  
  concurrencyPolicy: Forbid  # ไม่รัน concurrent
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 5
  startingDeadlineSeconds: 300  # ถ้าช้ากว่า 5 นาที ให้ skip
  
  jobTemplate:
    spec:
      backoffLimit: 2
      ttlSecondsAfterFinished: 86400
      
      template:
        spec:
          restartPolicy: OnFailure
          serviceAccountName: report-generator-sa
          
          containers:
            - name: report-generator
              image: report-generator:latest
              command: ["node", "dist/generate-report.js"]
              env:
                - name: REPORT_DATE
                  value: "yesterday"
                - name: EMAIL_RECIPIENTS
                  value: "management@company.com"
              resources:
                requests:
                  memory: "512Mi"
                  cpu: "500m"
                limits:
                  memory: "1Gi"
                  cpu: "1000m"
```

```typescript
// job-controller.ts
// TypeScript controller สำหรับ managing Jobs

import { BatchV1Api, CoreV1Api } from '@kubernetes/client-node';

interface JobConfig {
  name: string;
  namespace: string;
  image: string;
  command: string[];
  env?: Record<string, string>;
  parallelism?: number;
  completions?: number;
  backoffLimit?: number;
  ttlSeconds?: number;
}

class JobManager {
  private batchApi: BatchV1Api;
  private coreApi: CoreV1Api;

  constructor(batchApi: BatchV1Api, coreApi: CoreV1Api) {
    this.batchApi = batchApi;
    this.coreApi = coreApi;
  }

  async createJob(config: JobConfig): Promise<void> {
    await this.batchApi.createNamespacedJob(config.namespace, {
      apiVersion: 'batch/v1',
      kind: 'Job',
      metadata: {
        name: config.name,
        namespace: config.namespace,
      },
      spec: {
        parallelism: config.parallelism || 1,
        completions: config.completions || 1,
        backoffLimit: config.backoffLimit || 3,
        ttlSecondsAfterFinished: config.ttlSeconds || 3600,
        template: {
          spec: {
            restartPolicy: 'OnFailure',
            containers: [
              {
                name: 'worker',
                image: config.image,
                command: config.command,
                env: config.env
                  ? Object.entries(config.env).map(([name, value]) => ({
                      name,
                      value,
                    }))
                  : [],
              },
            ],
          },
        },
      },
    });
  }

  async watchJob(
    name: string,
    namespace: string
  ): Promise<'succeeded' | 'failed'> {
    return new Promise((resolve, reject) => {
      const timeout = setTimeout(() => reject(new Error('Job timeout')), 3600000);

      const poll = async () => {
        const job = await this.batchApi.readNamespacedJob(name, namespace);
        const status = job.body.status;

        if (status?.succeeded && status.succeeded > 0) {
          clearTimeout(timeout);
          resolve('succeeded');
          return;
        }

        if (status?.failed && status.failed >= (job.body.spec?.backoffLimit || 3)) {
          clearTimeout(timeout);
          resolve('failed');
          return;
        }

        setTimeout(poll, 5000);
      };

      poll();
    });
  }
}
```

---

## 4. Custom Resource Definitions (CRD)

```yaml
# crd-definition.yaml
# Custom Resource Definition สำหรับ OrderPipeline

apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: orderpipelines.company.io
spec:
  group: company.io
  versions:
    - name: v1alpha1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              required: ["steps", "trigger"]
              properties:
                steps:
                  type: array
                  items:
                    type: object
                    required: ["name", "image"]
                    properties:
                      name:
                        type: string
                      image:
                        type: string
                      command:
                        type: array
                        items:
                          type: string
                      env:
                        type: object
                        additionalProperties:
                          type: string
                      dependsOn:
                        type: array
                        items:
                          type: string
                trigger:
                  type: object
                  properties:
                    type:
                      type: string
                      enum: ["manual", "schedule", "webhook"]
                    schedule:
                      type: string
                    webhookSecret:
                      type: string
                timeout:
                  type: string
                  pattern: '^[0-9]+(s|m|h)$'
            status:
              type: object
              properties:
                phase:
                  type: string
                  enum: ["Pending", "Running", "Succeeded", "Failed"]
                startTime:
                  type: string
                  format: date-time
                completionTime:
                  type: string
                  format: date-time
                steps:
                  type: array
                  items:
                    type: object
                    properties:
                      name:
                        type: string
                      status:
                        type: string
                      startTime:
                        type: string
                      completionTime:
                        type: string
      additionalPrinterColumns:
        - name: Phase
          type: string
          jsonPath: .status.phase
        - name: Age
          type: date
          jsonPath: .metadata.creationTimestamp
      subresources:
        status: {}
  scope: Namespaced
  names:
    plural: orderpipelines
    singular: orderpipeline
    kind: OrderPipeline
    shortNames:
      - op
---
# Custom Resource instance
apiVersion: company.io/v1alpha1
kind: OrderPipeline
metadata:
  name: order-fulfillment-pipeline
  namespace: production
spec:
  trigger:
    type: webhook
    webhookSecret: my-webhook-secret
  
  timeout: 30m
  
  steps:
    - name: validate-order
      image: order-validator:latest
      command: ["./validate"]
      env:
        ORDER_SERVICE_URL: "http://order-service:3000"
    
    - name: reserve-inventory
      image: inventory-service:latest
      command: ["./reserve"]
      dependsOn: ["validate-order"]
    
    - name: process-payment
      image: payment-service:latest
      command: ["./process"]
      dependsOn: ["reserve-inventory"]
    
    - name: create-shipment
      image: shipping-service:latest
      command: ["./ship"]
      dependsOn: ["process-payment"]
```

---

## 5. Kubernetes Operators with operator-sdk

```typescript
// order-pipeline-operator.ts
// TypeScript Operator สำหรับ OrderPipeline CRD

import {
  KubeConfig,
  CustomObjectsApi,
  BatchV1Api,
  CoreV1Api,
  AppsV1Api,
} from '@kubernetes/client-node';
import { EventEmitter } from 'events';

interface OrderPipelineSpec {
  steps: Array<{
    name: string;
    image: string;
    command?: string[];
    env?: Record<string, string>;
    dependsOn?: string[];
  }>;
  trigger: {
    type: 'manual' | 'schedule' | 'webhook';
    schedule?: string;
    webhookSecret?: string;
  };
  timeout?: string;
}

interface OrderPipelineStatus {
  phase: 'Pending' | 'Running' | 'Succeeded' | 'Failed';
  startTime?: string;
  completionTime?: string;
  steps?: Array<{
    name: string;
    status: string;
    startTime?: string;
    completionTime?: string;
  }>;
}

interface OrderPipeline {
  apiVersion: string;
  kind: string;
  metadata: {
    name: string;
    namespace: string;
    resourceVersion?: string;
    uid?: string;
    generation?: number;
  };
  spec: OrderPipelineSpec;
  status?: OrderPipelineStatus;
}

class OrderPipelineOperator {
  private customApi: CustomObjectsApi;
  private batchApi: BatchV1Api;
  private kubeConfig: KubeConfig;
  private reconciling: Set<string> = new Set();

  constructor() {
    this.kubeConfig = new KubeConfig();
    this.kubeConfig.loadFromDefault();
    this.customApi = this.kubeConfig.makeApiClient(CustomObjectsApi);
    this.batchApi = this.kubeConfig.makeApiClient(BatchV1Api);
  }

  async start(): Promise<void> {
    console.log('Starting OrderPipeline Operator');
    
    // Watch for OrderPipeline resources
    await this.watchResources();
  }

  private async watchResources(): Promise<void> {
    const watch = require('@kubernetes/client-node').Watch;
    const watcher = new watch(this.kubeConfig);

    await watcher.watch(
      '/apis/company.io/v1alpha1/orderpipelines',
      {},
      (type: string, obj: OrderPipeline) => {
        if (type === 'ADDED' || type === 'MODIFIED') {
          this.reconcile(obj).catch(err =>
            console.error(`Reconcile failed for ${obj.metadata.name}:`, err)
          );
        } else if (type === 'DELETED') {
          this.cleanup(obj).catch(err =>
            console.error(`Cleanup failed for ${obj.metadata.name}:`, err)
          );
        }
      },
      (err: Error) => {
        console.error('Watch error:', err);
        // Restart watch after delay
        setTimeout(() => this.watchResources(), 5000);
      }
    );
  }

  private async reconcile(pipeline: OrderPipeline): Promise<void> {
    const key = `${pipeline.metadata.namespace}/${pipeline.metadata.name}`;
    
    // Prevent concurrent reconciliation
    if (this.reconciling.has(key)) {
      console.log(`Already reconciling ${key}, skipping`);
      return;
    }

    this.reconciling.add(key);

    try {
      console.log(`Reconciling ${key}`);

      const currentStatus = pipeline.status?.phase || 'Pending';

      if (currentStatus === 'Pending') {
        await this.startPipeline(pipeline);
      } else if (currentStatus === 'Running') {
        await this.checkPipelineProgress(pipeline);
      }

    } finally {
      this.reconciling.delete(key);
    }
  }

  private async startPipeline(pipeline: OrderPipeline): Promise<void> {
    // Update status to Running
    await this.updateStatus(pipeline, {
      phase: 'Running',
      startTime: new Date().toISOString(),
      steps: pipeline.spec.steps.map(step => ({
        name: step.name,
        status: 'Pending',
      })),
    });

    // Execute steps in dependency order
    await this.executeSteps(pipeline);
  }

  private async executeSteps(pipeline: OrderPipeline): Promise<void> {
    const completed = new Set<string>();
    const steps = pipeline.spec.steps;

    while (completed.size < steps.length) {
      // Find steps ready to run (all dependencies completed)
      const ready = steps.filter(step => {
        if (completed.has(step.name)) return false;
        if (!step.dependsOn || step.dependsOn.length === 0) return true;
        return step.dependsOn.every(dep => completed.has(dep));
      });

      if (ready.length === 0) break; // No progress possible

      // Execute ready steps in parallel
      await Promise.all(
        ready.map(async step => {
          await this.executeStep(pipeline, step);
          completed.add(step.name);
        })
      );
    }

    // Update final status
    if (completed.size === steps.length) {
      await this.updateStatus(pipeline, {
        phase: 'Succeeded',
        completionTime: new Date().toISOString(),
      });
    } else {
      await this.updateStatus(pipeline, {
        phase: 'Failed',
        completionTime: new Date().toISOString(),
      });
    }
  }

  private async executeStep(
    pipeline: OrderPipeline,
    step: OrderPipelineSpec['steps'][0]
  ): Promise<void> {
    const jobName = `${pipeline.metadata.name}-${step.name}`;

    // Create Job for this step
    await this.batchApi.createNamespacedJob(pipeline.metadata.namespace, {
      apiVersion: 'batch/v1',
      kind: 'Job',
      metadata: {
        name: jobName,
        namespace: pipeline.metadata.namespace,
        ownerReferences: [
          {
            apiVersion: 'company.io/v1alpha1',
            kind: 'OrderPipeline',
            name: pipeline.metadata.name,
            uid: pipeline.metadata.uid!,
            controller: true,
            blockOwnerDeletion: true,
          },
        ],
      },
      spec: {
        backoffLimit: 2,
        template: {
          spec: {
            restartPolicy: 'OnFailure',
            containers: [
              {
                name: step.name,
                image: step.image,
                command: step.command,
                env: step.env
                  ? Object.entries(step.env).map(([name, value]) => ({
                      name,
                      value,
                    }))
                  : [],
              },
            ],
          },
        },
      },
    });

    // Wait for job completion
    const result = await this.waitForJob(jobName, pipeline.metadata.namespace);

    if (result !== 'succeeded') {
      throw new Error(`Step ${step.name} failed`);
    }
  }

  private async waitForJob(name: string, namespace: string): Promise<string> {
    const timeout = 3600000; // 1 hour
    const start = Date.now();

    while (Date.now() - start < timeout) {
      const job = await this.batchApi.readNamespacedJob(name, namespace);
      const status = job.body.status;

      if (status?.succeeded && status.succeeded > 0) return 'succeeded';
      if (status?.failed && status.failed >= 3) return 'failed';

      await new Promise(resolve => setTimeout(resolve, 5000));
    }

    return 'timeout';
  }

  private async updateStatus(
    pipeline: OrderPipeline,
    status: Partial<OrderPipelineStatus>
  ): Promise<void> {
    await this.customApi.patchNamespacedCustomObjectStatus(
      'company.io',
      'v1alpha1',
      pipeline.metadata.namespace,
      'orderpipelines',
      pipeline.metadata.name,
      [{ op: 'replace', path: '/status', value: { ...pipeline.status, ...status } }],
      undefined, undefined, undefined,
      { headers: { 'Content-Type': 'application/json-patch+json' } }
    );
  }

  private async checkPipelineProgress(pipeline: OrderPipeline): Promise<void> {
    // Check if any jobs have completed/failed
    console.log(`Checking progress for ${pipeline.metadata.name}`);
  }

  private async cleanup(pipeline: OrderPipeline): Promise<void> {
    // Owner references handle cascading deletion
    console.log(`Cleanup triggered for ${pipeline.metadata.name}`);
  }
}

// Start operator
const operator = new OrderPipelineOperator();
operator.start().catch(console.error);
```

---

## 6. Controller Pattern Implementation

```typescript
// controller-pattern.ts
// Generic Controller pattern สำหรับ Kubernetes

interface ReconcileResult {
  requeue: boolean;
  requeueAfter?: number; // milliseconds
}

abstract class Controller<T> {
  private queue: Array<{ key: string; object: T }> = [];
  private processing = false;

  protected abstract reconcile(obj: T): Promise<ReconcileResult>;
  protected abstract getKey(obj: T): string;

  enqueue(obj: T): void {
    const key = this.getKey(obj);
    
    // Dedup: remove existing if already in queue
    const existing = this.queue.findIndex(item => item.key === key);
    if (existing !== -1) {
      this.queue.splice(existing, 1);
    }
    
    this.queue.push({ key, object: obj });
    
    if (!this.processing) {
      this.processQueue();
    }
  }

  private async processQueue(): Promise<void> {
    this.processing = true;
    
    while (this.queue.length > 0) {
      const item = this.queue.shift()!;
      
      try {
        const result = await this.reconcile(item.object);
        
        if (result.requeue) {
          const delay = result.requeueAfter || 0;
          setTimeout(() => this.enqueue(item.object), delay);
        }
      } catch (error) {
        console.error(`Reconcile failed for ${item.key}:`, error);
        // Requeue with backoff
        setTimeout(() => this.enqueue(item.object), 5000);
      }
    }
    
    this.processing = false;
  }
}

// Concrete controller implementation
class DeploymentHealthController extends Controller<any> {
  private k8sAppsApi: AppsV1Api;

  constructor(k8sAppsApi: AppsV1Api) {
    super();
    this.k8sAppsApi = k8sAppsApi;
  }

  protected getKey(obj: any): string {
    return `${obj.metadata.namespace}/${obj.metadata.name}`;
  }

  protected async reconcile(deployment: any): Promise<ReconcileResult> {
    const { name, namespace } = deployment.metadata;
    
    // Check deployment health
    const deplResult = await this.k8sAppsApi.readNamespacedDeployment(name, namespace);
    const depl = deplResult.body;
    
    const desired = depl.spec?.replicas || 1;
    const ready = depl.status?.readyReplicas || 0;
    
    if (ready < desired) {
      console.warn(`${name}: only ${ready}/${desired} replicas ready`);
      
      // Check for stale pods and delete them
      const staleDurationMs = 10 * 60 * 1000; // 10 minutes
      
      // Requeue to check again in 30 seconds
      return { requeue: true, requeueAfter: 30000 };
    }
    
    return { requeue: false };
  }
}

import { AppsV1Api } from '@kubernetes/client-node';
```

---

## 7. Admission Webhooks (ValidatingWebhookConfiguration)

```typescript
// admission-webhook.ts
// Validating Admission Webhook

import express, { Request, Response } from 'express';
import https from 'https';
import fs from 'fs';

interface AdmissionRequest {
  uid: string;
  kind: { group: string; version: string; kind: string };
  resource: { group: string; version: string; resource: string };
  operation: 'CREATE' | 'UPDATE' | 'DELETE' | 'CONNECT';
  object: Record<string, unknown>;
  oldObject?: Record<string, unknown>;
  options?: Record<string, unknown>;
}

interface AdmissionResponse {
  uid: string;
  allowed: boolean;
  status?: {
    code: number;
    message: string;
  };
  warnings?: string[];
  patch?: string;
  patchType?: 'JSONPatch';
}

class AdmissionWebhookServer {
  private app: express.Application;

  constructor() {
    this.app = express();
    this.app.use(express.json());
    this.setupRoutes();
  }

  private setupRoutes(): void {
    this.app.post('/validate/pods', this.validatePod.bind(this));
    this.app.post('/validate/deployments', this.validateDeployment.bind(this));
    this.app.post('/mutate/pods', this.mutatePod.bind(this));
  }

  // Validate Pod: ตรวจสอบ security requirements
  private validatePod(req: Request, res: Response): void {
    const admissionRequest = req.body.request as AdmissionRequest;
    const pod = admissionRequest.object as any;
    
    const warnings: string[] = [];
    const errors: string[] = [];

    // Check 1: ต้อง run as non-root
    for (const container of pod.spec?.containers || []) {
      if (!container.securityContext?.runAsNonRoot) {
        warnings.push(`Container ${container.name} should run as non-root`);
      }

      // Check 2: ต้องมี resource limits
      if (!container.resources?.limits?.memory) {
        errors.push(`Container ${container.name} must have memory limits`);
      }

      if (!container.resources?.limits?.cpu) {
        errors.push(`Container ${container.name} must have CPU limits`);
      }

      // Check 3: Image must have specific tag
      if (container.image?.endsWith(':latest')) {
        errors.push(`Container ${container.name} must not use 'latest' tag`);
      }

      // Check 4: Privileged containers not allowed
      if (container.securityContext?.privileged) {
        errors.push(`Container ${container.name} cannot run as privileged`);
      }
    }

    const response: AdmissionResponse = {
      uid: admissionRequest.uid,
      allowed: errors.length === 0,
      warnings: warnings.length > 0 ? warnings : undefined,
    };

    if (errors.length > 0) {
      response.status = {
        code: 400,
        message: `Validation failed: ${errors.join('; ')}`,
      };
    }

    res.json({ response });
  }

  // Validate Deployment
  private validateDeployment(req: Request, res: Response): void {
    const admissionRequest = req.body.request as AdmissionRequest;
    const deployment = admissionRequest.object as any;

    const errors: string[] = [];

    // Check replicas
    if (deployment.spec?.replicas === 1) {
      errors.push('Production deployments must have at least 2 replicas');
    }

    // Check rolling update strategy
    if (deployment.spec?.strategy?.type !== 'RollingUpdate') {
      errors.push('Deployments must use RollingUpdate strategy');
    }

    // Check required labels
    const requiredLabels = ['app', 'version', 'team'];
    for (const label of requiredLabels) {
      if (!deployment.metadata?.labels?.[label]) {
        errors.push(`Deployment must have label: ${label}`);
      }
    }

    res.json({
      response: {
        uid: admissionRequest.uid,
        allowed: errors.length === 0,
        status: errors.length > 0
          ? { code: 400, message: errors.join('; ') }
          : undefined,
      },
    });
  }

  // Mutate Pod: inject sidecar, labels, etc.
  private mutatePod(req: Request, res: Response): void {
    const admissionRequest = req.body.request as AdmissionRequest;
    const pod = admissionRequest.object as any;

    const patches: Array<{ op: string; path: string; value: unknown }> = [];

    // Add default labels
    if (!pod.metadata?.labels?.['app.kubernetes.io/managed-by']) {
      patches.push({
        op: 'add',
        path: '/metadata/labels/app.kubernetes.io~1managed-by',
        value: 'company-operator',
      });
    }

    // Inject log level env var if not present
    for (let i = 0; i < (pod.spec?.containers?.length || 0); i++) {
      const container = pod.spec.containers[i];
      const hasLogLevel = container.env?.some((e: any) => e.name === 'LOG_LEVEL');
      
      if (!hasLogLevel) {
        patches.push({
          op: 'add',
          path: `/spec/containers/${i}/env`,
          value: [...(container.env || []), { name: 'LOG_LEVEL', value: 'info' }],
        });
      }
    }

    const patchStr = patches.length > 0
      ? Buffer.from(JSON.stringify(patches)).toString('base64')
      : undefined;

    res.json({
      response: {
        uid: admissionRequest.uid,
        allowed: true,
        patch: patchStr,
        patchType: patchStr ? 'JSONPatch' : undefined,
      },
    });
  }

  start(port: number): void {
    const server = https.createServer(
      {
        key: fs.readFileSync('/certs/tls.key'),
        cert: fs.readFileSync('/certs/tls.crt'),
      },
      this.app
    );

    server.listen(port, () => {
      console.log(`Admission webhook listening on port ${port}`);
    });
  }
}
```

```yaml
# validating-webhook-config.yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: company-validating-webhook
  annotations:
    cert-manager.io/inject-ca-from: company-system/webhook-cert
spec:
  webhooks:
    - name: validate-pods.company.io
      admissionReviewVersions: ["v1"]
      clientConfig:
        service:
          name: admission-webhook
          namespace: company-system
          path: /validate/pods
          port: 443
      rules:
        - apiGroups: [""]
          apiVersions: ["v1"]
          operations: ["CREATE", "UPDATE"]
          resources: ["pods"]
          scope: Namespaced
      namespaceSelector:
        matchLabels:
          admission.company.io/enabled: "true"
      failurePolicy: Fail
      sideEffects: None
      timeoutSeconds: 5
    
    - name: validate-deployments.company.io
      admissionReviewVersions: ["v1"]
      clientConfig:
        service:
          name: admission-webhook
          namespace: company-system
          path: /validate/deployments
          port: 443
      rules:
        - apiGroups: ["apps"]
          apiVersions: ["v1"]
          operations: ["CREATE", "UPDATE"]
          resources: ["deployments"]
          scope: Namespaced
      namespaceSelector:
        matchLabels:
          env: production
      failurePolicy: Fail
      sideEffects: None
---
# MutatingWebhookConfiguration
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: company-mutating-webhook
spec:
  webhooks:
    - name: mutate-pods.company.io
      admissionReviewVersions: ["v1"]
      clientConfig:
        service:
          name: admission-webhook
          namespace: company-system
          path: /mutate/pods
          port: 443
      rules:
        - apiGroups: [""]
          apiVersions: ["v1"]
          operations: ["CREATE"]
          resources: ["pods"]
      failurePolicy: Ignore  # ไม่บล็อกถ้า webhook ล้มเหลว
      sideEffects: None
      reinvocationPolicy: Never
```

---

## 8. Multi-tenancy with Namespaces and RBAC

```yaml
# multi-tenancy.yaml
# Multi-tenancy setup สำหรับ Kubernetes

# Tenant A namespace
apiVersion: v1
kind: Namespace
metadata:
  name: tenant-a
  labels:
    tenant: a
    env: production
    admission.company.io/enabled: "true"
---
# Resource Quota per tenant
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-a-quota
  namespace: tenant-a
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "50"
    services: "10"
    persistentvolumeclaims: "20"
    count/deployments.apps: "20"
    count/secrets: "50"
    count/configmaps: "50"
---
# LimitRange สำหรับ default values
apiVersion: v1
kind: LimitRange
metadata:
  name: tenant-a-limits
  namespace: tenant-a
spec:
  limits:
    - type: Container
      default:
        cpu: 200m
        memory: 256Mi
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      max:
        cpu: "4"
        memory: 4Gi
      min:
        cpu: 50m
        memory: 64Mi
    
    - type: Pod
      max:
        cpu: "8"
        memory: 8Gi
    
    - type: PersistentVolumeClaim
      max:
        storage: 50Gi
      min:
        storage: 1Gi
---
# Tenant admin Role
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: tenant-admin
  namespace: tenant-a
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "configmaps", "secrets", "persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets", "statefulsets"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["batch"]
    resources: ["jobs", "cronjobs"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  # Cannot manage RBAC (prevent privilege escalation)
  # Cannot access other namespaces
---
# Tenant developer Role (read + limited write)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: tenant-developer
  namespace: tenant-a
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "configmaps"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods/log", "pods/exec"]
    verbs: ["get", "create"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "update", "patch"]
---
# Bind roles to users/groups
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: tenant-a-admins
  namespace: tenant-a
subjects:
  - kind: Group
    name: tenant-a-admins
    apiGroup: rbac.authorization.k8s.io
  - kind: ServiceAccount
    name: tenant-a-ci-sa
    namespace: tenant-a
roleRef:
  kind: Role
  apiGroup: rbac.authorization.k8s.io
  name: tenant-admin
```

---

## 9. GitOps with ArgoCD

```yaml
# argocd-application.yaml
# ArgoCD Application สำหรับ GitOps workflow

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: order-service
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
  labels:
    app: order-service
    team: order-team
spec:
  project: microservices
  
  source:
    repoURL: https://github.com/company/microservices-k8s.git
    targetRevision: main
    path: services/order-service/production
    
    # Helm chart support
    helm:
      valueFiles:
        - values.production.yaml
      parameters:
        - name: image.tag
          value: "1.2.3"
        - name: replicaCount
          value: "3"
  
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  
  syncPolicy:
    automated:
      prune: true    # ลบ resources ที่ไม่อยู่ใน Git
      selfHeal: true # auto-fix drift
      allowEmpty: false
    
    syncOptions:
      - Validate=true
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
  
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas  # ignore HPA-managed replicas
---
# ArgoCD AppProject สำหรับ multi-team
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: microservices
  namespace: argocd
spec:
  description: Microservices project
  
  sourceRepos:
    - 'https://github.com/company/*'
    - 'https://charts.company.com/*'
  
  destinations:
    - namespace: production
      server: https://kubernetes.default.svc
    - namespace: staging
      server: https://kubernetes.default.svc
  
  clusterResourceWhitelist:
    - group: ''
      kind: Namespace
  
  namespaceResourceBlacklist:
    - group: ''
      kind: LimitRange
    - group: ''
      kind: ResourceQuota
  
  roles:
    - name: developer
      description: Developer role
      policies:
        - p, proj:microservices:developer, applications, get, microservices/*, allow
        - p, proj:microservices:developer, applications, sync, microservices/*, allow
      groups:
        - company:developers
    
    - name: ops
      description: Operations role
      policies:
        - p, proj:microservices:ops, applications, *, microservices/*, allow
      groups:
        - company:ops
---
# ApplicationSet สำหรับ multi-environment deployment
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: order-service-environments
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - env: staging
            namespace: staging
            values_file: values.staging.yaml
            cluster: staging-cluster
          - env: production
            namespace: production
            values_file: values.production.yaml
            cluster: production-cluster
  
  template:
    metadata:
      name: 'order-service-{{env}}'
    spec:
      project: microservices
      source:
        repoURL: https://github.com/company/microservices-k8s.git
        targetRevision: HEAD
        path: services/order-service
        helm:
          valueFiles:
            - '{{values_file}}'
      destination:
        server: 'https://{{cluster}}.company.com'
        namespace: '{{namespace}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

```typescript
// argocd-integration.ts
// TypeScript integration กับ ArgoCD API

interface ArgoCDApp {
  name: string;
  namespace: string;
  project: string;
  status: {
    sync: { status: string };
    health: { status: string };
    reconciledAt: string;
    operationState?: { phase: string };
  };
}

class ArgoCDClient {
  private baseUrl: string;
  private token: string;

  constructor(baseUrl: string, token: string) {
    this.baseUrl = baseUrl;
    this.token = token;
  }

  async syncApplication(appName: string): Promise<void> {
    const response = await fetch(
      `${this.baseUrl}/api/v1/applications/${appName}/sync`,
      {
        method: 'POST',
        headers: {
          Authorization: `Bearer ${this.token}`,
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({
          revision: 'HEAD',
          prune: false,
          dryRun: false,
          force: false,
          strategy: {
            hook: { force: false },
          },
        }),
      }
    );

    if (!response.ok) {
      throw new Error(`Sync failed: ${await response.text()}`);
    }

    console.log(`Sync triggered for ${appName}`);
  }

  async waitForSync(appName: string, timeout: number = 300000): Promise<void> {
    const start = Date.now();

    while (Date.now() - start < timeout) {
      const app = await this.getApplication(appName);

      if (
        app.status.sync.status === 'Synced' &&
        app.status.health.status === 'Healthy'
      ) {
        console.log(`${appName} is synced and healthy`);
        return;
      }

      if (app.status.operationState?.phase === 'Failed') {
        throw new Error(`Sync failed for ${appName}`);
      }

      await new Promise(resolve => setTimeout(resolve, 5000));
    }

    throw new Error(`Timeout waiting for ${appName} to sync`);
  }

  async getApplication(name: string): Promise<ArgoCDApp> {
    const response = await fetch(
      `${this.baseUrl}/api/v1/applications/${name}`,
      {
        headers: { Authorization: `Bearer ${this.token}` },
      }
    );

    if (!response.ok) {
      throw new Error(`Failed to get app: ${name}`);
    }

    return response.json() as Promise<ArgoCDApp>;
  }

  async createImageUpdaterAnnotation(
    appName: string,
    image: string,
    newTag: string
  ): Promise<void> {
    // Patch application with new image tag annotation
    const response = await fetch(
      `${this.baseUrl}/api/v1/applications/${appName}`,
      {
        method: 'PATCH',
        headers: {
          Authorization: `Bearer ${this.token}`,
          'Content-Type': 'application/merge-patch+json',
        },
        body: JSON.stringify({
          metadata: {
            annotations: {
              'argocd-image-updater.argoproj.io/image-list': `${image}:${newTag}`,
            },
          },
        }),
      }
    );

    if (!response.ok) {
      throw new Error(`Failed to update annotation: ${await response.text()}`);
    }
  }
}
```

---

## สรุปตาราง Kubernetes Patterns

| Pattern | Use Case | Key Features |
|---------|---------|-------------|
| Deployment | Stateless services | Rolling updates, scaling |
| StatefulSet | Databases, queues | Stable identity, ordered ops |
| DaemonSet | Node agents, monitoring | One pod per node |
| Job | One-time tasks, migration | Completion tracking |
| CronJob | Scheduled tasks | Cron expression |
| CRD + Operator | Custom automation | Domain-specific logic |

| Kubernetes Feature | Purpose | Complexity |
|-------------------|---------|-----------|
| HPA | Auto CPU/memory scaling | Low |
| VPA | Right-sizing resources | Medium |
| KEDA | Event-driven scaling | Medium |
| Admission Webhook | Policy enforcement | High |
| CRD + Operator | Custom controllers | High |
| GitOps (ArgoCD) | Declarative deployment | Medium |
| Multi-tenancy RBAC | Team isolation | Medium |

| GitOps Principle | Implementation | Tool |
|----------------|---------------|------|
| Single source of truth | Git repository | GitHub/GitLab |
| Declarative config | Kubernetes YAML | kubectl/Helm |
| Automated sync | Pull-based CD | ArgoCD/Flux |
| Drift detection | Continuous reconciliation | ArgoCD |
| Audit trail | Git history | Git |

---

## สรุป

Kubernetes Patterns ครอบคลุม:

1. **StatefulSet** สำหรับ stateful workloads เช่น databases ที่ต้องการ stable identity
2. **DaemonSet** สำหรับ infrastructure agents บนทุก node
3. **Jobs/CronJobs** สำหรับ batch processing และ scheduled tasks
4. **CRDs** ขยาย Kubernetes API ด้วย domain-specific resources
5. **Operators** automate complex operational tasks ด้วย controller pattern
6. **Admission Webhooks** enforce policies และ mutate resources ก่อน save
7. **Multi-tenancy** ด้วย Namespace, RBAC, ResourceQuota, LimitRange
8. **GitOps ด้วย ArgoCD** declarative deployment ที่ sync จาก Git repository
9. **Custom Controllers** implement reconciliation loops สำหรับ custom behavior
10. **ApplicationSets** deploy ไปหลาย environments อัตโนมัติ

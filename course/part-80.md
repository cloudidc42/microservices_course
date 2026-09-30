# Part 80: Kubernetes Advanced Patterns

## บทนำ

Kubernetes Advanced Patterns ครอบคลุม Custom Resource Definitions (CRDs), Operators, Admission Webhooks, Multi-tenancy, GitOps และ Event-driven Automation บทนี้จะช่วยให้เข้าใจการขยาย Kubernetes API และการสร้าง platform features บน Kubernetes

---

## 80.1 Custom Resource Definition (CRD)

### CRD YAML สำหรับ MicroserviceDeployment

```yaml
# microservice-deployment-crd.yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: microservicedeployments.platform.example.com
  annotations:
    controller-gen.kubebuilder.io/version: v0.13.0
spec:
  group: platform.example.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            apiVersion:
              type: string
            kind:
              type: string
            metadata:
              type: object
            spec:
              type: object
              required:
                - image
                - port
              properties:
                image:
                  type: string
                  description: "Container image to deploy"
                  pattern: "^[a-z0-9/:._-]+$"
                port:
                  type: integer
                  minimum: 1
                  maximum: 65535
                  description: "Port the service listens on"
                replicas:
                  type: object
                  properties:
                    min:
                      type: integer
                      minimum: 1
                      default: 2
                    max:
                      type: integer
                      minimum: 1
                      default: 10
                    targetCPUUtilization:
                      type: integer
                      minimum: 1
                      maximum: 100
                      default: 70
                resources:
                  type: object
                  properties:
                    cpu:
                      type: object
                      properties:
                        request:
                          type: string
                          default: "100m"
                          pattern: "^[0-9]+(m|[.][0-9]+)?$"
                        limit:
                          type: string
                          default: "1000m"
                    memory:
                      type: object
                      properties:
                        request:
                          type: string
                          default: "128Mi"
                        limit:
                          type: string
                          default: "512Mi"
                environment:
                  type: array
                  items:
                    type: object
                    required:
                      - name
                    properties:
                      name:
                        type: string
                      value:
                        type: string
                      valueFrom:
                        type: object
                        properties:
                          secretKeyRef:
                            type: object
                            properties:
                              name:
                                type: string
                              key:
                                type: string
                          configMapKeyRef:
                            type: object
                            properties:
                              name:
                                type: string
                              key:
                                type: string
                ingress:
                  type: object
                  properties:
                    enabled:
                      type: boolean
                      default: false
                    host:
                      type: string
                    path:
                      type: string
                      default: "/"
                    tlsEnabled:
                      type: boolean
                      default: true
                healthCheck:
                  type: object
                  properties:
                    path:
                      type: string
                      default: "/health"
                    port:
                      type: integer
                    initialDelaySeconds:
                      type: integer
                      default: 10
                    periodSeconds:
                      type: integer
                      default: 10
                monitoring:
                  type: object
                  properties:
                    enabled:
                      type: boolean
                      default: true
                    path:
                      type: string
                      default: "/metrics"
                    interval:
                      type: string
                      default: "30s"
                serviceAccount:
                  type: string
                  description: "Service account name"
                configMaps:
                  type: array
                  items:
                    type: string
                secrets:
                  type: array
                  items:
                    type: string
                strategy:
                  type: object
                  properties:
                    type:
                      type: string
                      enum:
                        - RollingUpdate
                        - Recreate
                        - BlueGreen
                        - Canary
                      default: "RollingUpdate"
                    canaryWeight:
                      type: integer
                      minimum: 0
                      maximum: 100
                      default: 10
            status:
              type: object
              properties:
                phase:
                  type: string
                  enum:
                    - Pending
                    - Deploying
                    - Running
                    - Failed
                    - Terminating
                replicas:
                  type: integer
                readyReplicas:
                  type: integer
                availableReplicas:
                  type: integer
                conditions:
                  type: array
                  items:
                    type: object
                    properties:
                      type:
                        type: string
                      status:
                        type: string
                      reason:
                        type: string
                      message:
                        type: string
                      lastTransitionTime:
                        type: string
                observedGeneration:
                  type: integer
                  format: int64
                deploymentRef:
                  type: string
                serviceRef:
                  type: string
                ingressRef:
                  type: string
      additionalPrinterColumns:
        - name: Phase
          type: string
          jsonPath: .status.phase
        - name: Replicas
          type: integer
          jsonPath: .status.replicas
        - name: Ready
          type: integer
          jsonPath: .status.readyReplicas
        - name: Age
          type: date
          jsonPath: .metadata.creationTimestamp
      subresources:
        status: {}
        scale:
          specReplicasPath: .spec.replicas.min
          statusReplicasPath: .status.replicas
  scope: Namespaced
  names:
    plural: microservicedeployments
    singular: microservicedeployment
    kind: MicroserviceDeployment
    shortNames:
      - msd
    categories:
      - platform
      - all
```

### TypeScript CRD Schema ด้วย zod

```typescript
// microservice-deployment-schema.ts
import { z } from 'zod';

// Resource definition schema
const ResourceAmountSchema = z.string().regex(/^\d+(m|([.]\d+))?$/, {
  message: 'Must be valid Kubernetes resource amount (e.g., "100m", "0.5")',
});

const MemoryAmountSchema = z.string().regex(/^\d+(Ki|Mi|Gi|Ti|Pi|Ei|k|M|G|T|P|E)?$/, {
  message: 'Must be valid memory amount (e.g., "128Mi", "1Gi")',
});

const ResourcesSchema = z.object({
  cpu: z.object({
    request: ResourceAmountSchema.default('100m'),
    limit: ResourceAmountSchema.default('1000m'),
  }).optional(),
  memory: z.object({
    request: MemoryAmountSchema.default('128Mi'),
    limit: MemoryAmountSchema.default('512Mi'),
  }).optional(),
});

const EnvVarSchema = z.union([
  z.object({
    name: z.string().min(1),
    value: z.string(),
  }),
  z.object({
    name: z.string().min(1),
    valueFrom: z.union([
      z.object({
        secretKeyRef: z.object({
          name: z.string(),
          key: z.string(),
        }),
      }),
      z.object({
        configMapKeyRef: z.object({
          name: z.string(),
          key: z.string(),
        }),
      }),
    ]),
  }),
]);

const ReplicasSchema = z.object({
  min: z.number().int().min(1).default(2),
  max: z.number().int().min(1).default(10),
  targetCPUUtilization: z.number().int().min(1).max(100).default(70),
}).refine(data => data.max >= data.min, {
  message: 'max replicas must be >= min replicas',
});

const IngressSchema = z.object({
  enabled: z.boolean().default(false),
  host: z.string().optional(),
  path: z.string().default('/'),
  tlsEnabled: z.boolean().default(true),
});

const HealthCheckSchema = z.object({
  path: z.string().default('/health'),
  port: z.number().int().optional(),
  initialDelaySeconds: z.number().int().default(10),
  periodSeconds: z.number().int().default(10),
  failureThreshold: z.number().int().default(3),
});

const MonitoringSchema = z.object({
  enabled: z.boolean().default(true),
  path: z.string().default('/metrics'),
  interval: z.string().default('30s'),
});

const StrategySchema = z.object({
  type: z.enum(['RollingUpdate', 'Recreate', 'BlueGreen', 'Canary']).default('RollingUpdate'),
  canaryWeight: z.number().int().min(0).max(100).default(10),
});

const MicroserviceDeploymentSpecSchema = z.object({
  image: z.string().regex(/^[a-z0-9/:._-]+$/, {
    message: 'Image must be a valid container image reference',
  }),
  port: z.number().int().min(1).max(65535),
  replicas: ReplicasSchema.optional(),
  resources: ResourcesSchema.optional(),
  environment: z.array(EnvVarSchema).optional(),
  ingress: IngressSchema.optional(),
  healthCheck: HealthCheckSchema.optional(),
  monitoring: MonitoringSchema.optional(),
  serviceAccount: z.string().optional(),
  configMaps: z.array(z.string()).optional(),
  secrets: z.array(z.string()).optional(),
  strategy: StrategySchema.optional(),
});

const MicroserviceDeploymentStatusSchema = z.object({
  phase: z.enum(['Pending', 'Deploying', 'Running', 'Failed', 'Terminating']).optional(),
  replicas: z.number().int().optional(),
  readyReplicas: z.number().int().optional(),
  availableReplicas: z.number().int().optional(),
  conditions: z.array(z.object({
    type: z.string(),
    status: z.string(),
    reason: z.string().optional(),
    message: z.string().optional(),
    lastTransitionTime: z.string().optional(),
  })).optional(),
  observedGeneration: z.number().int().optional(),
  deploymentRef: z.string().optional(),
  serviceRef: z.string().optional(),
  ingressRef: z.string().optional(),
});

const MicroserviceDeploymentSchema = z.object({
  apiVersion: z.literal('platform.example.com/v1'),
  kind: z.literal('MicroserviceDeployment'),
  metadata: z.object({
    name: z.string().min(1).max(253),
    namespace: z.string().optional(),
    labels: z.record(z.string()).optional(),
    annotations: z.record(z.string()).optional(),
  }),
  spec: MicroserviceDeploymentSpecSchema,
  status: MicroserviceDeploymentStatusSchema.optional(),
});

type MicroserviceDeploymentSpec = z.infer<typeof MicroserviceDeploymentSpecSchema>;
type MicroserviceDeployment = z.infer<typeof MicroserviceDeploymentSchema>;

function validateMicroserviceDeployment(data: unknown): MicroserviceDeployment {
  return MicroserviceDeploymentSchema.parse(data);
}

export {
  MicroserviceDeploymentSchema,
  MicroserviceDeploymentSpecSchema,
  validateMicroserviceDeployment,
  type MicroserviceDeployment,
  type MicroserviceDeploymentSpec,
};
```

---

## 80.2 Kubernetes Operator ด้วย TypeScript

### TypeScript Operator ใช้ @kubernetes/client-node

```typescript
// microservice-operator.ts
import * as k8s from '@kubernetes/client-node';
import { EventEmitter } from 'events';

// Types
interface MicroserviceDeployment {
  apiVersion: string;
  kind: string;
  metadata: k8s.V1ObjectMeta;
  spec: {
    image: string;
    port: number;
    replicas?: {
      min: number;
      max: number;
      targetCPUUtilization: number;
    };
    resources?: {
      cpu?: { request: string; limit: string };
      memory?: { request: string; limit: string };
    };
    environment?: Array<{ name: string; value?: string }>;
    ingress?: {
      enabled: boolean;
      host?: string;
      path: string;
    };
    healthCheck?: {
      path: string;
      initialDelaySeconds: number;
      periodSeconds: number;
    };
  };
  status?: {
    phase?: string;
    replicas?: number;
    readyReplicas?: number;
    conditions?: Array<{
      type: string;
      status: string;
      reason?: string;
      message?: string;
      lastTransitionTime?: string;
    }>;
    observedGeneration?: number;
  };
}

type ReconcileAction = 'create' | 'update' | 'delete' | 'noop';

interface ReconcileResult {
  action: ReconcileAction;
  success: boolean;
  error?: string;
  requeue?: boolean;
  requeueAfterMs?: number;
}

// Operator Class
class MicroserviceOperator extends EventEmitter {
  private kc: k8s.KubeConfig;
  private coreApi: k8s.CoreV1Api;
  private appsApi: k8s.AppsV1Api;
  private customApi: k8s.CustomObjectsApi;
  private autoscalingApi: k8s.AutoscalingV2Api;
  private networkingApi: k8s.NetworkingV1Api;
  private informer: k8s.Informer<MicroserviceDeployment> | null = null;
  private reconcileQueue: Map<string, MicroserviceDeployment>;
  private isProcessing = false;

  private readonly GROUP = 'platform.example.com';
  private readonly VERSION = 'v1';
  private readonly PLURAL = 'microservicedeployments';
  private readonly FINALIZER = 'platform.example.com/cleanup';
  private readonly FIELD_MANAGER = 'microservice-operator';

  constructor() {
    super();
    this.kc = new k8s.KubeConfig();
    this.kc.loadFromDefault();
    this.coreApi = this.kc.makeApiClient(k8s.CoreV1Api);
    this.appsApi = this.kc.makeApiClient(k8s.AppsV1Api);
    this.customApi = this.kc.makeApiClient(k8s.CustomObjectsApi);
    this.autoscalingApi = this.kc.makeApiClient(k8s.AutoscalingV2Api);
    this.networkingApi = this.kc.makeApiClient(k8s.NetworkingV1Api);
    this.reconcileQueue = new Map();
  }

  async start(): Promise<void> {
    console.log('Starting MicroserviceDeployment Operator...');
    
    await this.setupInformer();
    await this.processQueue();
    
    console.log('Operator started successfully');
  }

  private async setupInformer(): Promise<void> {
    const listFn = () => this.customApi.listClusterCustomObject(
      this.GROUP,
      this.VERSION,
      this.PLURAL
    );

    this.informer = k8s.makeInformer(
      this.kc,
      `/apis/${this.GROUP}/${this.VERSION}/${this.PLURAL}`,
      listFn as any
    );

    this.informer.on('add', (obj: MicroserviceDeployment) => {
      console.log(`[ADD] ${obj.metadata.namespace}/${obj.metadata.name}`);
      this.enqueueReconcile(obj);
    });

    this.informer.on('update', (obj: MicroserviceDeployment) => {
      console.log(`[UPDATE] ${obj.metadata.namespace}/${obj.metadata.name}`);
      this.enqueueReconcile(obj);
    });

    this.informer.on('delete', (obj: MicroserviceDeployment) => {
      console.log(`[DELETE] ${obj.metadata.namespace}/${obj.metadata.name}`);
      // ลบออกจาก queue
      const key = `${obj.metadata.namespace}/${obj.metadata.name}`;
      this.reconcileQueue.delete(key);
    });

    this.informer.on('error', (err: Error) => {
      console.error('Informer error:', err.message);
      // Restart informer after 5 seconds
      setTimeout(() => this.setupInformer(), 5000);
    });

    await this.informer.start();
  }

  private enqueueReconcile(obj: MicroserviceDeployment): void {
    const key = `${obj.metadata.namespace}/${obj.metadata.name}`;
    this.reconcileQueue.set(key, obj);
    
    if (!this.isProcessing) {
      this.processQueue();
    }
  }

  private async processQueue(): Promise<void> {
    this.isProcessing = true;
    
    while (this.reconcileQueue.size > 0) {
      const [key, obj] = this.reconcileQueue.entries().next().value;
      this.reconcileQueue.delete(key);
      
      try {
        const result = await this.reconcile(obj);
        
        if (result.requeue) {
          const delay = result.requeueAfterMs ?? 5000;
          setTimeout(() => this.enqueueReconcile(obj), delay);
        }
      } catch (err: any) {
        console.error(`Reconcile error for ${key}:`, err.message);
        // Requeue with backoff
        setTimeout(() => this.enqueueReconcile(obj), 10000);
      }
    }
    
    this.isProcessing = false;
  }

  async reconcile(msd: MicroserviceDeployment): Promise<ReconcileResult> {
    const { name, namespace, generation, finalizers } = msd.metadata;
    const ns = namespace ?? 'default';

    console.log(`Reconciling ${ns}/${name} (generation: ${generation})`);

    // Handle deletion
    if (msd.metadata.deletionTimestamp) {
      return this.handleDeletion(msd, ns);
    }

    // Add finalizer if not present
    if (!finalizers?.includes(this.FINALIZER)) {
      await this.addFinalizer(msd, ns);
    }

    // Update status to Deploying
    await this.updateStatus(msd, ns, {
      phase: 'Deploying',
      conditions: [{
        type: 'Progressing',
        status: 'True',
        reason: 'ReconcileStarted',
        message: 'Reconciliation started',
        lastTransitionTime: new Date().toISOString(),
      }],
    });

    try {
      // Reconcile Deployment
      await this.reconcileDeployment(msd, ns);
      
      // Reconcile Service
      await this.reconcileService(msd, ns);
      
      // Reconcile HPA
      await this.reconcileHPA(msd, ns);
      
      // Reconcile Ingress if enabled
      if (msd.spec.ingress?.enabled) {
        await this.reconcileIngress(msd, ns);
      }

      // Check deployment readiness
      const deployment = await this.appsApi.readNamespacedDeployment(name, ns);
      const readyReplicas = deployment.body.status?.readyReplicas ?? 0;
      const totalReplicas = deployment.body.status?.replicas ?? 0;

      await this.updateStatus(msd, ns, {
        phase: readyReplicas === totalReplicas ? 'Running' : 'Deploying',
        replicas: totalReplicas,
        readyReplicas,
        observedGeneration: generation,
        conditions: [{
          type: 'Available',
          status: readyReplicas > 0 ? 'True' : 'False',
          reason: readyReplicas > 0 ? 'MinimumReplicasAvailable' : 'MinimumReplicasUnavailable',
          message: `${readyReplicas}/${totalReplicas} replicas ready`,
          lastTransitionTime: new Date().toISOString(),
        }],
      });

      // Requeue if not fully ready
      if (readyReplicas < totalReplicas) {
        return { action: 'update', success: true, requeue: true, requeueAfterMs: 10000 };
      }

      return { action: 'update', success: true, requeue: false };
    } catch (err: any) {
      await this.updateStatus(msd, ns, {
        phase: 'Failed',
        conditions: [{
          type: 'Failed',
          status: 'True',
          reason: 'ReconcileError',
          message: err.message,
          lastTransitionTime: new Date().toISOString(),
        }],
      });

      return {
        action: 'update',
        success: false,
        error: err.message,
        requeue: true,
        requeueAfterMs: 30000,
      };
    }
  }

  private async reconcileDeployment(
    msd: MicroserviceDeployment,
    namespace: string
  ): Promise<void> {
    const { name } = msd.metadata;
    const spec = msd.spec;
    
    const deploymentSpec: k8s.V1Deployment = {
      metadata: {
        name,
        namespace,
        labels: {
          'app': name,
          'managed-by': 'microservice-operator',
          'platform.example.com/msd': name,
        },
        ownerReferences: [{
          apiVersion: `${this.GROUP}/${this.VERSION}`,
          kind: 'MicroserviceDeployment',
          name,
          uid: msd.metadata.uid!,
          controller: true,
          blockOwnerDeletion: true,
        }],
      },
      spec: {
        replicas: spec.replicas?.min ?? 2,
        selector: {
          matchLabels: { app: name },
        },
        template: {
          metadata: {
            labels: { app: name },
            annotations: {
              'prometheus.io/scrape': 'true',
              'prometheus.io/path': '/metrics',
              'prometheus.io/port': spec.port.toString(),
            },
          },
          spec: {
            containers: [{
              name,
              image: spec.image,
              ports: [{ containerPort: spec.port }],
              env: spec.environment as k8s.V1EnvVar[],
              resources: {
                requests: {
                  cpu: spec.resources?.cpu?.request ?? '100m',
                  memory: spec.resources?.memory?.request ?? '128Mi',
                },
                limits: {
                  cpu: spec.resources?.cpu?.limit ?? '1000m',
                  memory: spec.resources?.memory?.limit ?? '512Mi',
                },
              },
              readinessProbe: {
                httpGet: {
                  path: spec.healthCheck?.path ?? '/health',
                  port: spec.port as any,
                },
                initialDelaySeconds: spec.healthCheck?.initialDelaySeconds ?? 10,
                periodSeconds: spec.healthCheck?.periodSeconds ?? 10,
              },
              livenessProbe: {
                httpGet: {
                  path: spec.healthCheck?.path ?? '/health',
                  port: spec.port as any,
                },
                initialDelaySeconds: (spec.healthCheck?.initialDelaySeconds ?? 10) * 3,
                periodSeconds: spec.healthCheck?.periodSeconds ?? 10,
              },
            }],
          },
        },
        strategy: {
          type: 'RollingUpdate',
          rollingUpdate: {
            maxSurge: 1,
            maxUnavailable: 0,
          },
        },
      },
    };

    try {
      await this.appsApi.readNamespacedDeployment(name, namespace);
      // Update existing
      await this.appsApi.patchNamespacedDeployment(
        name,
        namespace,
        deploymentSpec,
        undefined,
        undefined,
        this.FIELD_MANAGER,
        undefined,
        undefined,
        { headers: { 'Content-Type': 'application/apply-patch+yaml' } }
      );
    } catch (err: any) {
      if (err.statusCode === 404) {
        await this.appsApi.createNamespacedDeployment(namespace, deploymentSpec);
      } else {
        throw err;
      }
    }
  }

  private async reconcileService(
    msd: MicroserviceDeployment,
    namespace: string
  ): Promise<void> {
    const { name } = msd.metadata;
    
    const service: k8s.V1Service = {
      metadata: {
        name,
        namespace,
        labels: { app: name, 'managed-by': 'microservice-operator' },
        ownerReferences: [{
          apiVersion: `${this.GROUP}/${this.VERSION}`,
          kind: 'MicroserviceDeployment',
          name,
          uid: msd.metadata.uid!,
          controller: true,
          blockOwnerDeletion: true,
        }],
      },
      spec: {
        selector: { app: name },
        ports: [{
          port: 80,
          targetPort: msd.spec.port as any,
          protocol: 'TCP',
          name: 'http',
        }],
        type: 'ClusterIP',
      },
    };

    try {
      await this.coreApi.readNamespacedService(name, namespace);
      await this.coreApi.patchNamespacedService(
        name,
        namespace,
        service,
        undefined,
        undefined,
        undefined,
        undefined,
        undefined,
        { headers: { 'Content-Type': 'application/merge-patch+json' } }
      );
    } catch (err: any) {
      if (err.statusCode === 404) {
        await this.coreApi.createNamespacedService(namespace, service);
      } else {
        throw err;
      }
    }
  }

  private async reconcileHPA(
    msd: MicroserviceDeployment,
    namespace: string
  ): Promise<void> {
    const { name } = msd.metadata;
    const replicas = msd.spec.replicas;
    
    if (!replicas) return;

    const hpa: k8s.V2HorizontalPodAutoscaler = {
      metadata: {
        name,
        namespace,
        ownerReferences: [{
          apiVersion: `${this.GROUP}/${this.VERSION}`,
          kind: 'MicroserviceDeployment',
          name,
          uid: msd.metadata.uid!,
          controller: true,
          blockOwnerDeletion: true,
        }],
      },
      spec: {
        scaleTargetRef: {
          apiVersion: 'apps/v1',
          kind: 'Deployment',
          name,
        },
        minReplicas: replicas.min,
        maxReplicas: replicas.max,
        metrics: [{
          type: 'Resource',
          resource: {
            name: 'cpu',
            target: {
              type: 'Utilization',
              averageUtilization: replicas.targetCPUUtilization,
            },
          },
        }],
      },
    };

    try {
      await this.autoscalingApi.readNamespacedHorizontalPodAutoscaler(name, namespace);
      await this.autoscalingApi.patchNamespacedHorizontalPodAutoscaler(
        name,
        namespace,
        hpa,
        undefined,
        undefined,
        undefined,
        undefined,
        undefined,
        { headers: { 'Content-Type': 'application/merge-patch+json' } }
      );
    } catch (err: any) {
      if (err.statusCode === 404) {
        await this.autoscalingApi.createNamespacedHorizontalPodAutoscaler(namespace, hpa);
      } else {
        throw err;
      }
    }
  }

  private async reconcileIngress(
    msd: MicroserviceDeployment,
    namespace: string
  ): Promise<void> {
    const { name } = msd.metadata;
    const ingressSpec = msd.spec.ingress!;
    
    if (!ingressSpec.host) {
      throw new Error('Ingress requires a host');
    }

    const ingress: k8s.V1Ingress = {
      metadata: {
        name,
        namespace,
        annotations: {
          'kubernetes.io/ingress.class': 'nginx',
          'cert-manager.io/cluster-issuer': 'letsencrypt-prod',
        },
      },
      spec: {
        tls: ingressSpec.tlsEnabled ? [{
          hosts: [ingressSpec.host],
          secretName: `${name}-tls`,
        }] : undefined,
        rules: [{
          host: ingressSpec.host,
          http: {
            paths: [{
              path: ingressSpec.path,
              pathType: 'Prefix',
              backend: {
                service: {
                  name,
                  port: { number: 80 },
                },
              },
            }],
          },
        }],
      },
    };

    try {
      await this.networkingApi.readNamespacedIngress(name, namespace);
      await this.networkingApi.patchNamespacedIngress(
        name,
        namespace,
        ingress,
        undefined,
        undefined,
        undefined,
        undefined,
        undefined,
        { headers: { 'Content-Type': 'application/merge-patch+json' } }
      );
    } catch (err: any) {
      if (err.statusCode === 404) {
        await this.networkingApi.createNamespacedIngress(namespace, ingress);
      } else {
        throw err;
      }
    }
  }

  private async handleDeletion(
    msd: MicroserviceDeployment,
    namespace: string
  ): Promise<ReconcileResult> {
    const { name, finalizers } = msd.metadata;
    
    if (!finalizers?.includes(this.FINALIZER)) {
      return { action: 'delete', success: true };
    }

    console.log(`Handling deletion of ${namespace}/${name}`);
    
    // Cleanup logic here (e.g., external resources)
    
    // Remove finalizer
    await this.removeFinalizer(msd, namespace);
    
    return { action: 'delete', success: true };
  }

  private async addFinalizer(msd: MicroserviceDeployment, namespace: string): Promise<void> {
    const { name, finalizers = [] } = msd.metadata;
    
    await this.customApi.patchNamespacedCustomObject(
      this.GROUP,
      this.VERSION,
      namespace,
      this.PLURAL,
      name,
      { metadata: { finalizers: [...finalizers, this.FINALIZER] } },
      undefined,
      undefined,
      undefined,
      { headers: { 'Content-Type': 'application/merge-patch+json' } }
    );
  }

  private async removeFinalizer(msd: MicroserviceDeployment, namespace: string): Promise<void> {
    const { name, finalizers = [] } = msd.metadata;
    const newFinalizers = finalizers.filter(f => f !== this.FINALIZER);
    
    await this.customApi.patchNamespacedCustomObject(
      this.GROUP,
      this.VERSION,
      namespace,
      this.PLURAL,
      name,
      { metadata: { finalizers: newFinalizers } },
      undefined,
      undefined,
      undefined,
      { headers: { 'Content-Type': 'application/merge-patch+json' } }
    );
  }

  private async updateStatus(
    msd: MicroserviceDeployment,
    namespace: string,
    status: Partial<MicroserviceDeployment['status']>
  ): Promise<void> {
    const { name } = msd.metadata;
    
    try {
      await this.customApi.patchNamespacedCustomObjectStatus(
        this.GROUP,
        this.VERSION,
        namespace,
        this.PLURAL,
        name,
        { status },
        undefined,
        undefined,
        undefined,
        { headers: { 'Content-Type': 'application/merge-patch+json' } }
      );
    } catch (err: any) {
      console.warn(`Failed to update status for ${namespace}/${name}:`, err.message);
    }
  }

  async stop(): Promise<void> {
    if (this.informer) {
      await this.informer.stop();
    }
  }
}

// Main entry point
async function main() {
  const operator = new MicroserviceOperator();

  process.on('SIGTERM', async () => {
    console.log('Received SIGTERM, shutting down...');
    await operator.stop();
    process.exit(0);
  });

  process.on('SIGINT', async () => {
    console.log('Received SIGINT, shutting down...');
    await operator.stop();
    process.exit(0);
  });

  await operator.start();
}

main().catch(console.error);
```

---

## 80.3 Controller Pattern: TypeScript ReconcileController with Exponential Backoff

```typescript
// reconcile-controller.ts

interface ReconcileRequest {
  name: string;
  namespace: string;
  generation?: number;
}

interface ReconcileError extends Error {
  retryable: boolean;
  suggestedDelayMs?: number;
}

abstract class BaseController<T> {
  private workQueue: Map<string, ReconcileRequest>;
  private retryMap: Map<string, { count: number; nextRetryMs: number }>;
  private isRunning = false;
  protected maxRetries = 5;
  protected baseDelayMs = 1000;
  protected maxDelayMs = 300000; // 5 minutes

  constructor() {
    this.workQueue = new Map();
    this.retryMap = new Map();
  }

  protected abstract reconcile(request: ReconcileRequest): Promise<ReconcileResult>;

  enqueue(request: ReconcileRequest): void {
    const key = `${request.namespace}/${request.name}`;
    
    // Deduplication: only keep latest request per object
    this.workQueue.set(key, request);
    
    if (!this.isRunning) {
      this.runLoop();
    }
  }

  private async runLoop(): Promise<void> {
    this.isRunning = true;
    
    while (this.workQueue.size > 0) {
      const entries = Array.from(this.workQueue.entries());
      const now = Date.now();
      
      // Process only requests that are ready (not in backoff)
      const readyEntries = entries.filter(([key]) => {
        const retry = this.retryMap.get(key);
        return !retry || now >= retry.nextRetryMs;
      });

      if (readyEntries.length === 0) {
        // All in backoff, wait a bit
        await this.sleep(100);
        continue;
      }

      // Process one at a time (can be parallelized for performance)
      const [key, request] = readyEntries[0];
      this.workQueue.delete(key);

      await this.processRequest(key, request);
    }
    
    this.isRunning = false;
  }

  private async processRequest(
    key: string,
    request: ReconcileRequest
  ): Promise<void> {
    try {
      const result = await this.reconcile(request);
      
      if (result.success) {
        // Clear retry state on success
        this.retryMap.delete(key);
        
        if (result.requeue) {
          setTimeout(
            () => this.enqueue(request),
            result.requeueAfterMs ?? 10000
          );
        }
      } else if (result.requeue) {
        this.handleRetry(key, request, result.requeueAfterMs);
      }
    } catch (err: any) {
      const isRetryable = (err as ReconcileError).retryable !== false;
      
      if (isRetryable) {
        this.handleRetry(key, request, (err as ReconcileError).suggestedDelayMs);
      } else {
        console.error(`[Controller] Non-retryable error for ${key}:`, err.message);
        this.retryMap.delete(key);
      }
    }
  }

  private handleRetry(
    key: string,
    request: ReconcileRequest,
    suggestedDelayMs?: number
  ): void {
    const existing = this.retryMap.get(key) ?? { count: 0, nextRetryMs: 0 };
    
    if (existing.count >= this.maxRetries) {
      console.error(`[Controller] Max retries (${this.maxRetries}) exceeded for ${key}`);
      this.retryMap.delete(key);
      return;
    }

    // Exponential backoff: delay = base * 2^count + jitter
    const exponentialDelay = Math.min(
      this.baseDelayMs * Math.pow(2, existing.count),
      this.maxDelayMs
    );
    
    // Add ±20% jitter to prevent thundering herd
    const jitter = exponentialDelay * 0.2 * (Math.random() - 0.5) * 2;
    const delayMs = suggestedDelayMs ?? Math.round(exponentialDelay + jitter);

    const newRetry = {
      count: existing.count + 1,
      nextRetryMs: Date.now() + delayMs,
    };
    
    this.retryMap.set(key, newRetry);
    this.workQueue.set(key, request);
    
    console.log(
      `[Controller] Retry ${newRetry.count}/${this.maxRetries} for ${key} ` +
      `in ${(delayMs / 1000).toFixed(1)}s`
    );
  }

  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

// Concrete Controller implementation
class MicroserviceReconcileController extends BaseController<any> {
  protected async reconcile(request: ReconcileRequest): Promise<ReconcileResult> {
    console.log(`[MicroserviceController] Reconciling ${request.namespace}/${request.name}`);
    
    // ตัวอย่าง reconcile logic
    // ในความเป็นจริงจะ interact กับ Kubernetes API
    await this.doReconcile(request.name, request.namespace);
    
    return { action: 'update', success: true, requeue: false };
  }

  private async doReconcile(name: string, namespace: string): Promise<void> {
    // Implement actual reconcile logic here
    console.log(`  Processing ${namespace}/${name}`);
  }
}

export { BaseController, MicroserviceReconcileController };
```

---

## 80.4 Admission Webhook: TypeScript ValidatingWebhookServer

```typescript
// validating-webhook-server.ts
import * as https from 'https';
import * as fs from 'fs';
import * as express from 'express';

interface AdmissionRequest {
  apiVersion: string;
  kind: string;
  request: {
    uid: string;
    kind: { group: string; version: string; kind: string };
    resource: { group: string; version: string; resource: string };
    name: string;
    namespace: string;
    operation: 'CREATE' | 'UPDATE' | 'DELETE' | 'CONNECT';
    object: any;
    oldObject?: any;
    dryRun: boolean;
    userInfo: {
      username: string;
      uid?: string;
      groups?: string[];
    };
  };
}

interface AdmissionResponse {
  apiVersion: string;
  kind: string;
  response: {
    uid: string;
    allowed: boolean;
    status?: {
      code: number;
      message: string;
    };
    warnings?: string[];
  };
}

type ValidationResult = {
  allowed: true;
  warnings?: string[];
} | {
  allowed: false;
  code: number;
  message: string;
};

class ValidatingWebhookServer {
  private app: express.Application;
  private validators: Map<string, (req: AdmissionRequest['request']) => Promise<ValidationResult>>;

  constructor() {
    this.app = express();
    this.validators = new Map();
    this.setupMiddleware();
    this.setupRoutes();
  }

  private setupMiddleware(): void {
    this.app.use(express.json());
    this.app.use((req, res, next) => {
      console.log(`${req.method} ${req.path} from ${req.ip}`);
      next();
    });
  }

  private setupRoutes(): void {
    this.app.post('/validate/:resource', async (req, res) => {
      const resource = req.params.resource;
      const admissionReview = req.body as AdmissionRequest;

      const validator = this.validators.get(resource);
      
      if (!validator) {
        return res.json(this.buildResponse(admissionReview.request.uid, {
          allowed: false,
          code: 400,
          message: `No validator registered for resource: ${resource}`,
        }));
      }

      try {
        const result = await validator(admissionReview.request);
        return res.json(this.buildResponse(admissionReview.request.uid, result));
      } catch (err: any) {
        console.error(`Validation error for ${resource}:`, err.message);
        return res.json(this.buildResponse(admissionReview.request.uid, {
          allowed: false,
          code: 500,
          message: `Internal webhook error: ${err.message}`,
        }));
      }
    });

    this.app.get('/healthz', (req, res) => {
      res.json({ status: 'ok', time: new Date().toISOString() });
    });
  }

  registerValidator(
    resource: string,
    validator: (req: AdmissionRequest['request']) => Promise<ValidationResult>
  ): void {
    this.validators.set(resource, validator);
    console.log(`Registered validator for resource: ${resource}`);
  }

  private buildResponse(uid: string, result: ValidationResult): AdmissionResponse {
    const response: AdmissionResponse = {
      apiVersion: 'admission.k8s.io/v1',
      kind: 'AdmissionReview',
      response: {
        uid,
        allowed: result.allowed,
      },
    };

    if (!result.allowed) {
      response.response.status = {
        code: result.code,
        message: result.message,
      };
    } else if (result.warnings) {
      response.response.warnings = result.warnings;
    }

    return response;
  }

  start(port: number, certFile: string, keyFile: string): void {
    const serverOptions: https.ServerOptions = {
      cert: fs.readFileSync(certFile),
      key: fs.readFileSync(keyFile),
    };

    const server = https.createServer(serverOptions, this.app);
    
    server.listen(port, () => {
      console.log(`Validating webhook server listening on port ${port}`);
    });
  }
}

// MicroserviceDeployment Validator
async function microserviceDeploymentValidator(
  req: AdmissionRequest['request']
): Promise<ValidationResult> {
  const msd = req.object;
  const warnings: string[] = [];

  // Validate image is not using 'latest' tag
  if (msd.spec?.image?.endsWith(':latest')) {
    return {
      allowed: false,
      code: 400,
      message: 'Image tag "latest" is not allowed in production. Please use a specific version tag.',
    };
  }

  // Validate resource limits are set
  if (!msd.spec?.resources?.cpu?.limit || !msd.spec?.resources?.memory?.limit) {
    return {
      allowed: false,
      code: 400,
      message: 'Resource limits (CPU and memory) are required for all deployments.',
    };
  }

  // Validate max replicas doesn't exceed cluster policy
  const maxAllowedReplicas = 50;
  if (msd.spec?.replicas?.max > maxAllowedReplicas) {
    return {
      allowed: false,
      code: 400,
      message: `Maximum replicas (${msd.spec.replicas.max}) exceeds cluster limit (${maxAllowedReplicas}).`,
    };
  }

  // Warning: no ingress but in production namespace
  if (!msd.spec?.ingress?.enabled && req.namespace === 'production') {
    warnings.push('Ingress is not enabled. Service will only be accessible within the cluster.');
  }

  // Warning: no monitoring
  if (!msd.spec?.monitoring?.enabled) {
    warnings.push('Monitoring is not enabled. Consider enabling Prometheus metrics scraping.');
  }

  return { allowed: true, warnings: warnings.length > 0 ? warnings : undefined };
}

// Server setup
const webhookServer = new ValidatingWebhookServer();
webhookServer.registerValidator('microservicedeployments', microserviceDeploymentValidator);
webhookServer.start(
  8443,
  process.env.TLS_CERT_FILE ?? '/tls/tls.crt',
  process.env.TLS_KEY_FILE ?? '/tls/tls.key'
);
```

### ValidatingWebhookConfiguration YAML

```yaml
# validating-webhook-config.yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: microservice-deployment-validator
  annotations:
    cert-manager.io/inject-ca-from: "platform-system/webhook-server-cert"
webhooks:
  - name: validate.microservicedeployments.platform.example.com
    admissionReviewVersions: ["v1"]
    clientConfig:
      service:
        name: microservice-webhook-server
        namespace: platform-system
        path: /validate/microservicedeployments
        port: 443
    rules:
      - apiGroups: ["platform.example.com"]
        apiVersions: ["v1"]
        resources: ["microservicedeployments"]
        operations:
          - CREATE
          - UPDATE
        scope: Namespaced
    namespaceSelector:
      matchExpressions:
        - key: kubernetes.io/metadata.name
          operator: NotIn
          values:
            - kube-system
            - kube-public
            - platform-system
    failurePolicy: Fail
    sideEffects: None
    timeoutSeconds: 10
```

---

## 80.5 MutatingWebhookConfiguration: Auto-inject Sidecar

```yaml
# mutating-webhook-config.yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: sidecar-injector
  annotations:
    cert-manager.io/inject-ca-from: "platform-system/webhook-server-cert"
webhooks:
  - name: inject-sidecar.platform.example.com
    admissionReviewVersions: ["v1"]
    clientConfig:
      service:
        name: sidecar-injector
        namespace: platform-system
        path: /mutate/pods
        port: 443
    rules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["pods"]
        operations:
          - CREATE
        scope: Namespaced
    namespaceSelector:
      matchLabels:
        sidecar-injection: enabled
    objectSelector:
      matchExpressions:
        - key: sidecar-injector/inject
          operator: NotIn
          values: ["false"]
    failurePolicy: Ignore  # Don't block pod creation if webhook fails
    sideEffects: None
    timeoutSeconds: 5
    reinvocationPolicy: IfNeeded
```

### TypeScript Mutating Webhook สำหรับ Sidecar Injection

```typescript
// mutating-webhook-server.ts
import * as express from 'express';
import * as https from 'https';
import * as fs from 'fs';

interface MutatingAdmissionResponse {
  apiVersion: string;
  kind: string;
  response: {
    uid: string;
    allowed: boolean;
    patchType?: 'JSONPatch';
    patch?: string;
    status?: { message: string };
  };
}

interface JSONPatch {
  op: 'add' | 'remove' | 'replace' | 'copy' | 'move' | 'test';
  path: string;
  value?: any;
  from?: string;
}

class MutatingSidecarInjector {
  private app: express.Application;

  private readonly SIDECAR_CONTAINER = {
    name: 'monitoring-sidecar',
    image: 'myregistry/monitoring-sidecar:v1.0.0',
    ports: [{ containerPort: 9091, name: 'metrics' }],
    resources: {
      requests: { cpu: '50m', memory: '64Mi' },
      limits: { cpu: '200m', memory: '128Mi' },
    },
    env: [
      { name: 'SIDECAR_ENABLED', value: 'true' },
      { name: 'POD_NAME', valueFrom: { fieldRef: { fieldPath: 'metadata.name' } } },
      { name: 'POD_NAMESPACE', valueFrom: { fieldRef: { fieldPath: 'metadata.namespace' } } },
    ],
    readinessProbe: {
      httpGet: { path: '/ready', port: 9091 },
      initialDelaySeconds: 5,
      periodSeconds: 10,
    },
  };

  constructor() {
    this.app = express();
    this.app.use(express.json());
    this.setupRoutes();
  }

  private setupRoutes(): void {
    this.app.post('/mutate/pods', async (req, res) => {
      const admissionReview = req.body;
      const pod = admissionReview.request?.object;
      const uid = admissionReview.request?.uid;

      // ตรวจสอบว่า sidecar ถูก inject แล้วหรือยัง
      const alreadyInjected = pod?.metadata?.annotations?.['sidecar-injected'] === 'true';
      const skipInjection = pod?.metadata?.annotations?.['sidecar-injector/inject'] === 'false';

      if (alreadyInjected || skipInjection) {
        return res.json(this.allowWithoutPatch(uid));
      }

      const patches = this.buildPatches(pod);
      return res.json(this.buildMutatingResponse(uid, patches));
    });

    this.app.get('/healthz', (req, res) => res.json({ status: 'ok' }));
  }

  private buildPatches(pod: any): JSONPatch[] {
    const patches: JSONPatch[] = [];
    const containers = pod.spec?.containers ?? [];

    // Add sidecar container
    patches.push({
      op: 'add',
      path: '/spec/containers/-',
      value: this.SIDECAR_CONTAINER,
    });

    // Add volume for shared metrics data
    const volumes = pod.spec?.volumes ?? [];
    if (volumes.length === 0) {
      patches.push({ op: 'add', path: '/spec/volumes', value: [] });
    }
    patches.push({
      op: 'add',
      path: '/spec/volumes/-',
      value: {
        name: 'monitoring-data',
        emptyDir: { medium: 'Memory', sizeLimit: '10Mi' },
      },
    });

    // Add annotation to mark as injected
    const annotations = pod.metadata?.annotations ?? {};
    if (Object.keys(annotations).length === 0) {
      patches.push({ op: 'add', path: '/metadata/annotations', value: {} });
    }
    patches.push({
      op: 'add',
      path: '/metadata/annotations/sidecar-injected',
      value: 'true',
    });
    patches.push({
      op: 'add',
      path: '/metadata/annotations/sidecar-injected-at',
      value: new Date().toISOString(),
    });

    return patches;
  }

  private buildMutatingResponse(uid: string, patches: JSONPatch[]): MutatingAdmissionResponse {
    const patchBytes = Buffer.from(JSON.stringify(patches)).toString('base64');
    
    return {
      apiVersion: 'admission.k8s.io/v1',
      kind: 'AdmissionReview',
      response: {
        uid,
        allowed: true,
        patchType: 'JSONPatch',
        patch: patchBytes,
      },
    };
  }

  private allowWithoutPatch(uid: string): MutatingAdmissionResponse {
    return {
      apiVersion: 'admission.k8s.io/v1',
      kind: 'AdmissionReview',
      response: { uid, allowed: true },
    };
  }

  start(port: number, certFile: string, keyFile: string): void {
    const server = https.createServer(
      { cert: fs.readFileSync(certFile), key: fs.readFileSync(keyFile) },
      this.app
    );
    
    server.listen(port, () => {
      console.log(`Mutating webhook server listening on port ${port}`);
    });
  }
}

const injector = new MutatingSidecarInjector();
injector.start(
  8443,
  process.env.TLS_CERT_FILE ?? '/tls/tls.crt',
  process.env.TLS_KEY_FILE ?? '/tls/tls.key'
);
```

---

## 80.6 Multi-tenancy: Namespace per Tenant, ResourceQuota, NetworkPolicy

```yaml
# tenant-namespace.yaml
# สำหรับ tenant แต่ละ tenant จะมี namespace ของตัวเอง
apiVersion: v1
kind: Namespace
metadata:
  name: tenant-acme
  labels:
    tenant: acme
    tenant-tier: premium
    environment: production
  annotations:
    tenant/owner: "admin@acme.com"
    tenant/cost-center: "CC-001"
    tenant/created-at: "2024-01-01"
---
# ResourceQuota จำกัด resource ต่อ tenant
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-acme-quota
  namespace: tenant-acme
spec:
  hard:
    # Compute
    requests.cpu: "20"
    limits.cpu: "40"
    requests.memory: "40Gi"
    limits.memory: "80Gi"
    
    # Storage
    requests.storage: "500Gi"
    persistentvolumeclaims: "20"
    
    # Objects
    pods: "100"
    services: "20"
    secrets: "50"
    configmaps: "50"
    deployments.apps: "20"
    statefulsets.apps: "10"
    
    # Load balancers (cost control)
    services.loadbalancers: "2"
    services.nodeports: "0"
---
# LimitRange default limits
apiVersion: v1
kind: LimitRange
metadata:
  name: tenant-acme-limits
  namespace: tenant-acme
spec:
  limits:
    - type: Container
      default:          # ค่า default limit ถ้าไม่ระบุ
        cpu: "500m"
        memory: "512Mi"
      defaultRequest:   # ค่า default request ถ้าไม่ระบุ
        cpu: "100m"
        memory: "128Mi"
      max:              # ค่าสูงสุดที่อนุญาต
        cpu: "4"
        memory: "8Gi"
      min:              # ค่าต่ำสุด
        cpu: "50m"
        memory: "64Mi"
    - type: PersistentVolumeClaim
      max:
        storage: "50Gi"
      min:
        storage: "1Gi"
---
# NetworkPolicy ป้องกัน cross-tenant communication
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: tenant-isolation
  namespace: tenant-acme
spec:
  podSelector: {}  # Apply to all pods in namespace
  policyTypes:
    - Ingress
    - Egress
  ingress:
    # Allow traffic only from same tenant namespace
    - from:
        - namespaceSelector:
            matchLabels:
              tenant: acme
    # Allow from ingress controller
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
          podSelector:
            matchLabels:
              app.kubernetes.io/name: ingress-nginx
    # Allow from monitoring
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
      ports:
        - protocol: TCP
          port: 9090   # Prometheus metrics
  egress:
    # Allow traffic to same namespace
    - to:
        - namespaceSelector:
            matchLabels:
              tenant: acme
    # Allow DNS
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
    # Allow external HTTPS
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 10.0.0.0/8
              - 172.16.0.0/12
              - 192.168.0.0/16
      ports:
        - protocol: TCP
          port: 443
---
# RBAC สำหรับ tenant admin
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: tenant-admin
  namespace: tenant-acme
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "configmaps", "secrets"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["apps"]
    resources: ["deployments", "statefulsets", "replicasets"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["autoscaling"]
    resources: ["horizontalpodautoscalers"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: [""]
    resources: ["resourcequotas", "limitranges"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: tenant-acme-admin-binding
  namespace: tenant-acme
subjects:
  - kind: User
    name: admin@acme.com
    apiGroup: rbac.authorization.k8s.io
  - kind: ServiceAccount
    name: tenant-acme-sa
    namespace: tenant-acme
roleRef:
  kind: Role
  name: tenant-admin
  apiGroup: rbac.authorization.k8s.io
```

---

## 80.7 GitOps กับ ArgoCD

### Application YAML

```yaml
# argocd-application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: order-service
  namespace: argocd
  labels:
    team: platform
    environment: production
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: microservices-production
  
  source:
    repoURL: https://github.com/myorg/k8s-manifests.git
    targetRevision: main
    path: apps/order-service/production
    
    # Helm chart support
    # helm:
    #   valueFiles:
    #     - values.yaml
    #     - values-production.yaml
    #   parameters:
    #     - name: image.tag
    #       value: "v1.2.3"
    
    # Kustomize support
    kustomize:
      images:
        - myregistry/order-service:v1.2.3
      nameSuffix: -production
      commonLabels:
        environment: production

  destination:
    server: https://kubernetes.default.svc
    namespace: production

  syncPolicy:
    automated:
      prune: true           # ลบ resources ที่ไม่อยู่ใน Git
      selfHeal: true        # Auto-heal drift
      allowEmpty: false     # ไม่อนุญาต empty commit
    syncOptions:
      - Validate=true
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true
      - ApplyOutOfSyncOnly=true  # Apply only changed resources
      - RespectIgnoreDifferences=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m

  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas    # ไม่สนใจ replicas (managed by HPA)
    - group: ""
      kind: Service
      jsonPointers:
        - /spec/clusterIP   # IP assigned by Kubernetes
    - group: autoscaling
      kind: HorizontalPodAutoscaler
      jsonPointers:
        - /spec/metrics    # May be modified by KEDA

  revisionHistoryLimit: 10

  info:
    - name: slack-channel
      value: "#deployments"
    - name: team
      value: "Platform Team"
---
# AppProject สำหรับ organization-level governance
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: microservices-production
  namespace: argocd
spec:
  description: "Production microservices project"
  
  # Source repositories ที่อนุญาต
  sourceRepos:
    - "https://github.com/myorg/k8s-manifests.git"
    - "https://charts.mycompany.com/*"
  
  # Destination clusters/namespaces ที่อนุญาต
  destinations:
    - server: https://kubernetes.default.svc
      namespace: "production"
    - server: https://kubernetes.default.svc
      namespace: "staging"
  
  # Cluster resources ที่อนุญาต (cluster-scoped)
  clusterResourceWhitelist:
    - group: ""
      kind: Namespace
    - group: rbac.authorization.k8s.io
      kind: ClusterRole
    - group: rbac.authorization.k8s.io
      kind: ClusterRoleBinding
  
  # Namespace resources ที่อนุญาต
  namespaceResourceWhitelist:
    - group: "*"
      kind: "*"
  
  # ห้ามใช้ resources เหล่านี้
  namespaceResourceBlacklist:
    - group: ""
      kind: ResourceQuota
    - group: ""
      kind: LimitRange
  
  # Sync windows - control when syncs are allowed
  syncWindows:
    - kind: allow
      schedule: "0 8-18 * * 1-5"   # Allow weekdays 8am-6pm
      duration: 10h
      applications:
        - "*"
      manualSync: true
    - kind: deny
      schedule: "0 0 * * 5"         # Deny Friday midnight
      duration: 24h
      applications:
        - "*"
  
  # Roles within project
  roles:
    - name: developer
      description: "Allow read access and manual sync"
      policies:
        - p, proj:microservices-production:developer, applications, get, microservices-production/*, allow
        - p, proj:microservices-production:developer, applications, sync, microservices-production/*, allow
      groups:
        - "developers"
    - name: platform-admin
      description: "Full access"
      policies:
        - p, proj:microservices-production:platform-admin, applications, *, microservices-production/*, allow
      groups:
        - "platform-team"
  
  # Orphaned resources
  orphanedResources:
    warn: true
    ignore:
      - group: apps
        kind: ReplicaSet
```

---

## 80.8 Namespace Lifecycle Management TypeScript Script

```typescript
// namespace-lifecycle.ts
import * as k8s from '@kubernetes/client-node';
import * as yaml from 'js-yaml';

interface NamespaceConfig {
  name: string;
  tenant: string;
  tier: 'free' | 'standard' | 'premium';
  environment: 'development' | 'staging' | 'production';
  owner: string;
  costCenter: string;
  labels?: Record<string, string>;
  annotations?: Record<string, string>;
}

const TIER_QUOTAS = {
  free: {
    'requests.cpu': '2',
    'limits.cpu': '4',
    'requests.memory': '4Gi',
    'limits.memory': '8Gi',
    'pods': '20',
    'services': '5',
  },
  standard: {
    'requests.cpu': '10',
    'limits.cpu': '20',
    'requests.memory': '20Gi',
    'limits.memory': '40Gi',
    'pods': '50',
    'services': '15',
  },
  premium: {
    'requests.cpu': '20',
    'limits.cpu': '40',
    'requests.memory': '40Gi',
    'limits.memory': '80Gi',
    'pods': '100',
    'services': '30',
  },
};

class NamespaceLifecycleManager {
  private kc: k8s.KubeConfig;
  private coreApi: k8s.CoreV1Api;
  private rbacApi: k8s.RbacAuthorizationV1Api;
  private networkingApi: k8s.NetworkingV1Api;

  constructor() {
    this.kc = new k8s.KubeConfig();
    this.kc.loadFromDefault();
    this.coreApi = this.kc.makeApiClient(k8s.CoreV1Api);
    this.rbacApi = this.kc.makeApiClient(k8s.RbacAuthorizationV1Api);
    this.networkingApi = this.kc.makeApiClient(k8s.NetworkingV1Api);
  }

  async provisionTenantNamespace(config: NamespaceConfig): Promise<void> {
    console.log(`Provisioning namespace: ${config.name}`);

    // 1. Create namespace
    await this.createNamespace(config);
    
    // 2. Apply ResourceQuota
    await this.applyResourceQuota(config);
    
    // 3. Apply LimitRange
    await this.applyLimitRange(config);
    
    // 4. Apply NetworkPolicy
    await this.applyNetworkPolicy(config);
    
    // 5. Create ServiceAccount
    await this.createServiceAccount(config);
    
    // 6. Setup RBAC
    await this.setupRBAC(config);
    
    console.log(`Namespace ${config.name} provisioned successfully`);
  }

  private async createNamespace(config: NamespaceConfig): Promise<void> {
    const namespace: k8s.V1Namespace = {
      metadata: {
        name: config.name,
        labels: {
          tenant: config.tenant,
          'tenant-tier': config.tier,
          environment: config.environment,
          'managed-by': 'namespace-lifecycle-manager',
          ...config.labels,
        },
        annotations: {
          'tenant/owner': config.owner,
          'tenant/cost-center': config.costCenter,
          'tenant/created-at': new Date().toISOString(),
          'tenant/provisioned-by': 'namespace-lifecycle-manager',
          ...config.annotations,
        },
      },
    };

    try {
      await this.coreApi.createNamespace(namespace);
      console.log(`  Created namespace: ${config.name}`);
    } catch (err: any) {
      if (err.statusCode === 409) {
        console.log(`  Namespace ${config.name} already exists, updating...`);
        await this.coreApi.patchNamespace(
          config.name,
          namespace,
          undefined,
          undefined,
          undefined,
          undefined,
          undefined,
          { headers: { 'Content-Type': 'application/merge-patch+json' } }
        );
      } else {
        throw err;
      }
    }
  }

  private async applyResourceQuota(config: NamespaceConfig): Promise<void> {
    const quota = TIER_QUOTAS[config.tier];
    
    const resourceQuota: k8s.V1ResourceQuota = {
      metadata: {
        name: 'tenant-quota',
        namespace: config.name,
      },
      spec: {
        hard: quota as any,
      },
    };

    try {
      await this.coreApi.createNamespacedResourceQuota(config.name, resourceQuota);
    } catch (err: any) {
      if (err.statusCode === 409) {
        await this.coreApi.replaceNamespacedResourceQuota(
          'tenant-quota',
          config.name,
          resourceQuota
        );
      } else {
        throw err;
      }
    }

    console.log(`  Applied ResourceQuota for tier: ${config.tier}`);
  }

  private async applyLimitRange(config: NamespaceConfig): Promise<void> {
    const limitRange: k8s.V1LimitRange = {
      metadata: {
        name: 'default-limits',
        namespace: config.name,
      },
      spec: {
        limits: [
          {
            type: 'Container' as any,
            default: { cpu: '500m', memory: '512Mi' },
            defaultRequest: { cpu: '100m', memory: '128Mi' },
            max: { cpu: '4', memory: '8Gi' },
            min: { cpu: '50m', memory: '64Mi' },
          },
        ],
      },
    };

    try {
      await this.coreApi.createNamespacedLimitRange(config.name, limitRange);
    } catch (err: any) {
      if (err.statusCode === 409) {
        await this.coreApi.replaceNamespacedLimitRange(
          'default-limits',
          config.name,
          limitRange
        );
      } else {
        throw err;
      }
    }

    console.log(`  Applied LimitRange`);
  }

  private async applyNetworkPolicy(config: NamespaceConfig): Promise<void> {
    const networkPolicy: k8s.V1NetworkPolicy = {
      metadata: {
        name: 'tenant-isolation',
        namespace: config.name,
      },
      spec: {
        podSelector: {},
        policyTypes: ['Ingress', 'Egress'],
        ingress: [
          {
            from: [
              { namespaceSelector: { matchLabels: { tenant: config.tenant } } },
              { namespaceSelector: { matchLabels: { 'kubernetes.io/metadata.name': 'ingress-nginx' } } },
              { namespaceSelector: { matchLabels: { 'kubernetes.io/metadata.name': 'monitoring' } } },
            ],
          },
        ],
        egress: [
          {
            to: [{ namespaceSelector: { matchLabels: { tenant: config.tenant } } }],
          },
          {
            to: [{ namespaceSelector: { matchLabels: { 'kubernetes.io/metadata.name': 'kube-system' } } }],
            ports: [{ protocol: 'UDP' as any, port: 53 as any }],
          },
        ],
      },
    };

    try {
      await this.networkingApi.createNamespacedNetworkPolicy(config.name, networkPolicy);
    } catch (err: any) {
      if (err.statusCode === 409) {
        await this.networkingApi.replaceNamespacedNetworkPolicy(
          'tenant-isolation',
          config.name,
          networkPolicy
        );
      } else {
        throw err;
      }
    }

    console.log(`  Applied NetworkPolicy`);
  }

  private async createServiceAccount(config: NamespaceConfig): Promise<void> {
    const sa: k8s.V1ServiceAccount = {
      metadata: {
        name: `${config.tenant}-sa`,
        namespace: config.name,
      },
    };

    try {
      await this.coreApi.createNamespacedServiceAccount(config.name, sa);
    } catch (err: any) {
      if (err.statusCode !== 409) throw err;
    }

    console.log(`  Created ServiceAccount: ${config.tenant}-sa`);
  }

  private async setupRBAC(config: NamespaceConfig): Promise<void> {
    // Create Role
    const role: k8s.V1Role = {
      metadata: { name: 'tenant-developer', namespace: config.name },
      rules: [
        {
          apiGroups: ['', 'apps', 'autoscaling'],
          resources: ['pods', 'services', 'deployments', 'configmaps', 'horizontalpodautoscalers'],
          verbs: ['get', 'list', 'watch', 'create', 'update', 'patch', 'delete'],
        },
      ],
    };

    try {
      await this.rbacApi.createNamespacedRole(config.name, role);
    } catch (err: any) {
      if (err.statusCode !== 409) throw err;
    }

    console.log(`  Setup RBAC`);
  }

  async deprovisionNamespace(namespaceName: string, dryRun = false): Promise<void> {
    console.log(`${dryRun ? '[DRY RUN] ' : ''}Deprovisioning namespace: ${namespaceName}`);

    if (!dryRun) {
      await this.coreApi.deleteNamespace(namespaceName);
      console.log(`  Namespace ${namespaceName} deleted`);
    } else {
      console.log(`  Would delete namespace: ${namespaceName}`);
    }
  }

  async listTenantNamespaces(tenant?: string): Promise<k8s.V1Namespace[]> {
    const labelSelector = tenant
      ? `tenant=${tenant}`
      : 'managed-by=namespace-lifecycle-manager';
    
    const result = await this.coreApi.listNamespace(
      undefined,
      undefined,
      undefined,
      undefined,
      labelSelector
    );
    
    return result.body.items;
  }
}

// ตัวอย่างการใช้งาน
async function main() {
  const manager = new NamespaceLifecycleManager();

  // สร้าง namespace สำหรับ tenant ใหม่
  await manager.provisionTenantNamespace({
    name: 'tenant-acme',
    tenant: 'acme',
    tier: 'premium',
    environment: 'production',
    owner: 'admin@acme.com',
    costCenter: 'CC-001',
  });

  // List ทุก tenant namespaces
  const namespaces = await manager.listTenantNamespaces();
  console.log('\nTenant namespaces:');
  namespaces.forEach(ns => {
    const labels = ns.metadata?.labels ?? {};
    console.log(`  ${ns.metadata?.name} (tenant: ${labels.tenant}, tier: ${labels['tenant-tier']})`);
  });
}

main().catch(console.error);
```

---

## 80.9 Kubernetes API Aggregation: APIService Registration

```yaml
# api-service-registration.yaml
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  name: v1alpha1.platform.example.com
  labels:
    app: platform-api-server
spec:
  service:
    name: platform-api-server
    namespace: platform-system
    port: 443
  group: platform.example.com
  version: v1alpha1
  insecureSkipTLSVerify: false
  caBundle: <base64-encoded-CA-bundle>
  groupPriorityMinimum: 1000
  versionPriority: 15
---
# Platform API Server Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: platform-api-server
  namespace: platform-system
spec:
  replicas: 2
  selector:
    matchLabels:
      app: platform-api-server
  template:
    metadata:
      labels:
        app: platform-api-server
    spec:
      serviceAccountName: platform-api-server
      containers:
        - name: platform-api-server
          image: myregistry/platform-api-server:v1.0.0
          ports:
            - containerPort: 443
              name: https
          args:
            - --tls-cert-file=/tls/tls.crt
            - --tls-private-key-file=/tls/tls.key
            - --audit-log-path=/var/log/audit.log
            - --audit-log-maxage=30
            - --v=4
          volumeMounts:
            - name: tls
              mountPath: /tls
              readOnly: true
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          readinessProbe:
            httpGet:
              path: /readyz
              port: 443
              scheme: HTTPS
            initialDelaySeconds: 10
            periodSeconds: 5
      volumes:
        - name: tls
          secret:
            secretName: platform-api-server-tls
---
apiVersion: v1
kind: Service
metadata:
  name: platform-api-server
  namespace: platform-system
spec:
  selector:
    app: platform-api-server
  ports:
    - port: 443
      targetPort: 443
  type: ClusterIP
```

---

## 80.10 Kubernetes Event-driven Automation: TypeScript EventWatcher

```typescript
// event-watcher.ts
import * as k8s from '@kubernetes/client-node';

interface WatchRule {
  resourceType: string;
  eventTypes: Array<'ADDED' | 'MODIFIED' | 'DELETED'>;
  labelSelector?: string;
  fieldSelector?: string;
  namespaces?: string[];
  handler: (event: WatchEvent) => Promise<void>;
}

interface WatchEvent {
  type: 'ADDED' | 'MODIFIED' | 'DELETED';
  object: k8s.KubernetesObject;
  namespace?: string;
  name?: string;
  labels?: Record<string, string>;
  annotations?: Record<string, string>;
}

interface AutomationAction {
  name: string;
  condition: (event: WatchEvent) => boolean;
  action: (event: WatchEvent) => Promise<void>;
}

class KubernetesEventWatcher {
  private kc: k8s.KubeConfig;
  private coreApi: k8s.CoreV1Api;
  private appsApi: k8s.AppsV1Api;
  private watchers: k8s.Watch;
  private activeWatches: Map<string, any>;
  private rules: WatchRule[];
  private automations: AutomationAction[];

  constructor() {
    this.kc = new k8s.KubeConfig();
    this.kc.loadFromDefault();
    this.coreApi = this.kc.makeApiClient(k8s.CoreV1Api);
    this.appsApi = this.kc.makeApiClient(k8s.AppsV1Api);
    this.watchers = new k8s.Watch(this.kc);
    this.activeWatches = new Map();
    this.rules = [];
    this.automations = [];
  }

  addRule(rule: WatchRule): void {
    this.rules.push(rule);
  }

  addAutomation(automation: AutomationAction): void {
    this.automations.push(automation);
  }

  async watchPods(namespace: string = ''): Promise<void> {
    const path = namespace
      ? `/api/v1/namespaces/${namespace}/pods`
      : '/api/v1/pods';

    const watch = await this.watchers.watch(
      path,
      { watch: true },
      async (type, obj: k8s.V1Pod) => {
        const event: WatchEvent = {
          type: type as any,
          object: obj,
          namespace: obj.metadata?.namespace,
          name: obj.metadata?.name,
          labels: obj.metadata?.labels,
          annotations: obj.metadata?.annotations,
        };

        await this.processEvent(event, 'pod');
      },
      (err) => {
        if (err) {
          console.error('Watch error:', err.message);
          // Restart watch after delay
          setTimeout(() => this.watchPods(namespace), 5000);
        }
      }
    );

    this.activeWatches.set(`pods-${namespace}`, watch);
    console.log(`Watching pods in ${namespace || 'all namespaces'}`);
  }

  async watchDeployments(namespace: string = ''): Promise<void> {
    const path = namespace
      ? `/apis/apps/v1/namespaces/${namespace}/deployments`
      : '/apis/apps/v1/deployments';

    const watch = await this.watchers.watch(
      path,
      {},
      async (type, obj: k8s.V1Deployment) => {
        const event: WatchEvent = {
          type: type as any,
          object: obj,
          namespace: obj.metadata?.namespace,
          name: obj.metadata?.name,
          labels: obj.metadata?.labels,
          annotations: obj.metadata?.annotations,
        };

        await this.processEvent(event, 'deployment');
      },
      (err) => {
        if (err) {
          setTimeout(() => this.watchDeployments(namespace), 5000);
        }
      }
    );

    this.activeWatches.set(`deployments-${namespace}`, watch);
  }

  async watchKubernetesEvents(namespace: string = ''): Promise<void> {
    const path = namespace
      ? `/api/v1/namespaces/${namespace}/events`
      : '/api/v1/events';

    const watch = await this.watchers.watch(
      path,
      {},
      async (type, event: k8s.CoreV1Event) => {
        if (event.type === 'Warning') {
          await this.handleWarningEvent(event);
        }
      },
      (err) => {
        if (err) {
          setTimeout(() => this.watchKubernetesEvents(namespace), 5000);
        }
      }
    );

    this.activeWatches.set(`events-${namespace}`, watch);
  }

  private async processEvent(event: WatchEvent, resourceType: string): Promise<void> {
    // Run all matching automations
    for (const automation of this.automations) {
      try {
        if (automation.condition(event)) {
          console.log(`[EventWatcher] Triggering automation: ${automation.name}`);
          await automation.action(event);
        }
      } catch (err: any) {
        console.error(`[EventWatcher] Automation ${automation.name} failed:`, err.message);
      }
    }
  }

  private async handleWarningEvent(event: k8s.CoreV1Event): Promise<void> {
    const reason = event.reason ?? '';
    const message = event.message ?? '';
    const name = event.involvedObject.name;
    const namespace = event.involvedObject.namespace;

    console.log(`[Warning] ${namespace}/${name}: ${reason} - ${message}`);

    // Auto-remediation actions
    if (reason === 'OOMKilled') {
      await this.handleOOMKilled(namespace!, name!, event);
    } else if (reason === 'BackOff') {
      await this.handleCrashLoopBackOff(namespace!, name!, event);
    } else if (reason === 'FailedScheduling') {
      await this.handleFailedScheduling(namespace!, name!, event);
    }
  }

  private async handleOOMKilled(
    namespace: string,
    podName: string,
    event: k8s.CoreV1Event
  ): Promise<void> {
    console.log(`[AutoRemediation] OOMKilled detected for ${namespace}/${podName}`);
    
    // ดึง deployment ที่เกี่ยวข้อง
    const pod = await this.coreApi.readNamespacedPod(podName, namespace);
    const deploymentName = pod.body.metadata?.ownerReferences?.find(
      o => o.kind === 'ReplicaSet'
    )?.name;
    
    if (!deploymentName) return;

    // เพิ่ม memory limit อัตโนมัติ 20%
    const deployment = await this.appsApi.readNamespacedDeployment(
      deploymentName.replace(/-\w+$/, ''),  // Remove ReplicaSet suffix
      namespace
    );

    const containers = deployment.body.spec?.template?.spec?.containers ?? [];
    
    for (const container of containers) {
      const currentLimit = container.resources?.limits?.memory ?? '512Mi';
      const newLimit = this.increaseMemory(currentLimit, 1.2);
      
      console.log(`  Increasing memory limit for ${container.name}: ${currentLimit} -> ${newLimit}`);
      
      // Patch deployment
      if (container.resources?.limits) {
        container.resources.limits.memory = newLimit;
      }
    }

    // Send notification (webhook/Slack)
    await this.sendNotification(
      `OOMKilled: Increased memory for ${namespace}/${deploymentName}`
    );
  }

  private async handleCrashLoopBackOff(
    namespace: string,
    podName: string,
    event: k8s.CoreV1Event
  ): Promise<void> {
    console.log(`[AutoRemediation] CrashLoopBackOff for ${namespace}/${podName}`);
    
    // ดึง logs จาก pod ที่ crash
    try {
      const logs = await this.coreApi.readNamespacedPodLog(podName, namespace, undefined, undefined, false, undefined, undefined, undefined, 100);
      console.log(`  Last 100 lines of logs:\n${logs.body.slice(-2000)}`);
    } catch (err) {
      console.log(`  Failed to get logs: ${err}`);
    }
  }

  private async handleFailedScheduling(
    namespace: string,
    podName: string,
    event: k8s.CoreV1Event
  ): Promise<void> {
    console.log(`[AutoRemediation] FailedScheduling for ${namespace}/${podName}: ${event.message}`);
    
    if (event.message?.includes('Insufficient memory')) {
      await this.sendNotification(
        `ALERT: Cluster running low on memory! Pod ${namespace}/${podName} cannot be scheduled.`
      );
    }
  }

  private increaseMemory(current: string, factor: number): string {
    const units: Record<string, number> = {
      'Ki': 1024,
      'Mi': 1024 * 1024,
      'Gi': 1024 * 1024 * 1024,
    };

    const match = current.match(/^(\d+)(Ki|Mi|Gi)$/);
    if (!match) return current;

    const value = parseInt(match[1]);
    const unit = match[2];
    const newValue = Math.ceil(value * factor);
    
    return `${newValue}${unit}`;
  }

  private async sendNotification(message: string): Promise<void> {
    const webhookUrl = process.env.SLACK_WEBHOOK_URL;
    if (!webhookUrl) return;

    await fetch(webhookUrl, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        text: `🤖 K8s Auto-Remediation: ${message}`,
        channel: '#alerts',
      }),
    });
  }

  stopAll(): void {
    this.activeWatches.forEach((watch, key) => {
      console.log(`Stopping watch: ${key}`);
      watch.abort();
    });
    this.activeWatches.clear();
  }
}

// Setup Event-driven Automations
async function main() {
  const watcher = new KubernetesEventWatcher();

  // Automation: Auto-label new pods from specific deployments
  watcher.addAutomation({
    name: 'auto-label-cost-center',
    condition: (event) => {
      return event.type === 'ADDED' &&
        event.labels?.['auto-cost-label'] !== 'done' &&
        event.namespace === 'production';
    },
    action: async (event) => {
      console.log(`Auto-labeling pod: ${event.namespace}/${event.name}`);
      // Implement labeling logic
    },
  });

  // Automation: Send notification when deployment fails
  watcher.addAutomation({
    name: 'deployment-failure-alert',
    condition: (event) => {
      const deployment = event.object as k8s.V1Deployment;
      const conditions = deployment.status?.conditions ?? [];
      const progressing = conditions.find(c => c.type === 'Progressing');
      return event.type === 'MODIFIED' &&
        progressing?.status === 'False' &&
        progressing?.reason === 'ProgressDeadlineExceeded';
    },
    action: async (event) => {
      console.log(`ALERT: Deployment ${event.namespace}/${event.name} failed!`);
      // Send notification
    },
  });

  // Start watching
  await Promise.all([
    watcher.watchPods('production'),
    watcher.watchDeployments('production'),
    watcher.watchKubernetesEvents('production'),
  ]);

  process.on('SIGTERM', () => {
    watcher.stopAll();
    process.exit(0);
  });

  console.log('Event watcher started. Watching for Kubernetes events...');
}

main().catch(console.error);
```

---

## สรุปบทที่ 80

| Pattern | เทคโนโลยี | ใช้สำหรับ |
|---------|-----------|----------|
| CRD | apiextensions.k8s.io/v1 | Custom Kubernetes resources |
| Operator | @kubernetes/client-node + Informer | Automate complex application lifecycle |
| ReconcileController | TypeScript + Exponential Backoff | Reliable state reconciliation |
| ValidatingWebhook | Express + TLS | Enforce policies before resource creation |
| MutatingWebhook | JSON Patch | Auto-inject sidecars, defaults |
| API Aggregation | APIService | Extend Kubernetes API |
| Multi-tenancy | Namespace + ResourceQuota + NetworkPolicy | Tenant isolation |
| GitOps ArgoCD | Application + AppProject | Declarative, Git-driven deployments |
| Namespace Lifecycle | TypeScript Manager | Automated tenant onboarding |
| EventWatcher | k8s Watch API | Event-driven automation and auto-remediation |

### Key Takeaways

1. **CRD = Custom Kubernetes Resources**: สร้าง abstraction layer บน Kubernetes API
2. **Operator Pattern = Automated Operations**: Replace manual runbooks ด้วย code
3. **Exponential Backoff = Resilient Reconciliation**: ป้องกัน thundering herd ใน controller
4. **Admission Webhooks = Policy Enforcement**: บังคับ best practices ก่อน resource สร้าง
5. **Multi-tenancy ต้อง Defense-in-Depth**: Namespace + ResourceQuota + NetworkPolicy + RBAC
6. **GitOps = Single Source of Truth**: Git เป็น desired state, ArgoCD sync เป็น actual state
7. **EventWatcher = Proactive Operations**: ตอบสนองต่อ events อัตโนมัติ ลด MTTR

---

*Part 80 จบแล้ว - จบ Series: Service Mesh Advanced, Scalability Patterns, Kubernetes Advanced Patterns*

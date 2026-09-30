# Part 69: Kubernetes Advanced Operations

## บทนำ

ใน Part นี้เราจะเรียนรู้ Kubernetes ขั้นสูง ตั้งแต่การสร้าง Custom Resource Definitions (CRDs) และ Operators ไปจนถึง Security, Admission Controllers และ Auto-scaling ขั้นสูง ซึ่งใช้งานจริงใน production environments

---

## 1. Custom Resource Definitions (CRDs)

CRDs ช่วยให้เราขยาย Kubernetes API ด้วย resource types ของเราเอง

```yaml
# crds/database-cluster.crd.yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databaseclusters.db.example.com
spec:
  group: db.example.com
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
              required:
                - engine
                - replicas
                - storage
              properties:
                engine:
                  type: string
                  enum: [postgresql, mysql, mongodb]
                version:
                  type: string
                  pattern: '^\d+\.\d+$'
                replicas:
                  type: integer
                  minimum: 1
                  maximum: 10
                storage:
                  type: object
                  required:
                    - size
                  properties:
                    size:
                      type: string
                      pattern: '^[0-9]+[KMGTP]i?$'
                    storageClass:
                      type: string
                      default: standard
                resources:
                  type: object
                  properties:
                    requests:
                      type: object
                      properties:
                        cpu:
                          type: string
                        memory:
                          type: string
                    limits:
                      type: object
                      properties:
                        cpu:
                          type: string
                        memory:
                          type: string
                backup:
                  type: object
                  properties:
                    enabled:
                      type: boolean
                      default: false
                    schedule:
                      type: string
                    retentionDays:
                      type: integer
                      default: 7
            status:
              type: object
              properties:
                phase:
                  type: string
                  enum: [Pending, Creating, Running, Updating, Failed, Deleting]
                readyReplicas:
                  type: integer
                primaryEndpoint:
                  type: string
                replicaEndpoints:
                  type: array
                  items:
                    type: string
                conditions:
                  type: array
                  items:
                    type: object
                    properties:
                      type:
                        type: string
                      status:
                        type: string
                      lastTransitionTime:
                        type: string
                      reason:
                        type: string
                      message:
                        type: string
      additionalPrinterColumns:
        - name: Engine
          type: string
          jsonPath: .spec.engine
        - name: Replicas
          type: integer
          jsonPath: .spec.replicas
        - name: Status
          type: string
          jsonPath: .status.phase
        - name: Age
          type: date
          jsonPath: .metadata.creationTimestamp
      subresources:
        status: {}
        scale:
          specReplicasPath: .spec.replicas
          statusReplicasPath: .status.readyReplicas
  scope: Namespaced
  names:
    plural: databaseclusters
    singular: databasecluster
    kind: DatabaseCluster
    shortNames:
      - dbc
```

```yaml
# ตัวอย่าง DatabaseCluster resource
apiVersion: db.example.com/v1alpha1
kind: DatabaseCluster
metadata:
  name: production-pg
  namespace: data
spec:
  engine: postgresql
  version: "15.4"
  replicas: 3
  storage:
    size: 100Gi
    storageClass: ssd-premium
  resources:
    requests:
      cpu: "500m"
      memory: "1Gi"
    limits:
      cpu: "2"
      memory: "4Gi"
  backup:
    enabled: true
    schedule: "0 2 * * *"  # Daily at 2am
    retentionDays: 30
```

---

## 2. Operator Pattern

Operator ใช้ Controller loop (reconcile loop) เพื่อจัดการ lifecycle ของ custom resources

```typescript
// operator/database-operator.ts
import {
  KubeConfig,
  CustomObjectsApi,
  AppsV1Api,
  CoreV1Api,
  makeInformer,
} from "@kubernetes/client-node";

interface DatabaseCluster {
  metadata: {
    name: string;
    namespace: string;
    generation?: number;
    resourceVersion?: string;
  };
  spec: {
    engine: string;
    version: string;
    replicas: number;
    storage: { size: string; storageClass?: string };
    resources?: {
      requests?: { cpu?: string; memory?: string };
      limits?: { cpu?: string; memory?: string };
    };
    backup?: { enabled: boolean; schedule?: string; retentionDays?: number };
  };
  status?: {
    phase: string;
    readyReplicas?: number;
    primaryEndpoint?: string;
    observedGeneration?: number;
  };
}

export class DatabaseClusterOperator {
  private readonly kc: KubeConfig;
  private readonly customObjects: CustomObjectsApi;
  private readonly appsApi: AppsV1Api;
  private readonly coreApi: CoreV1Api;

  private readonly GROUP = "db.example.com";
  private readonly VERSION = "v1alpha1";
  private readonly PLURAL = "databaseclusters";

  constructor() {
    this.kc = new KubeConfig();
    this.kc.loadFromDefault();

    this.customObjects = this.kc.makeApiClient(CustomObjectsApi);
    this.appsApi = this.kc.makeApiClient(AppsV1Api);
    this.coreApi = this.kc.makeApiClient(CoreV1Api);
  }

  async start(): Promise<void> {
    console.log("Starting DatabaseCluster Operator...");

    const watch = this.kc.makeApiClient(require("@kubernetes/client-node").Watch);

    // Create informer to watch DatabaseCluster resources
    const listFn = () =>
      this.customObjects.listClusterCustomObject(
        this.GROUP,
        this.VERSION,
        this.PLURAL
      );

    const informer = makeInformer(this.kc, `/apis/${this.GROUP}/${this.VERSION}/${this.PLURAL}`, listFn as any);

    informer.on("add", (obj: DatabaseCluster) => {
      console.log(`DatabaseCluster added: ${obj.metadata.name}`);
      this.reconcile(obj).catch((e) => console.error("Reconcile error:", e));
    });

    informer.on("update", (obj: DatabaseCluster) => {
      console.log(`DatabaseCluster updated: ${obj.metadata.name}`);
      this.reconcile(obj).catch((e) => console.error("Reconcile error:", e));
    });

    informer.on("delete", (obj: DatabaseCluster) => {
      console.log(`DatabaseCluster deleted: ${obj.metadata.name}`);
      this.cleanup(obj).catch((e) => console.error("Cleanup error:", e));
    });

    informer.on("error", (err: Error) => {
      console.error("Informer error:", err);
      // Restart informer after error
      setTimeout(() => informer.start(), 5000);
    });

    await informer.start();
    console.log("Operator started, watching for DatabaseCluster resources...");
  }

  async reconcile(cluster: DatabaseCluster): Promise<void> {
    const { name, namespace } = cluster.metadata;

    try {
      await this.updateStatus(cluster, { phase: "Creating" });

      // Create StatefulSet for the database
      await this.ensureStatefulSet(cluster);

      // Create Services
      await this.ensureServices(cluster);

      // Create PersistentVolumeClaims
      await this.ensurePVCs(cluster);

      // Setup backup CronJob if enabled
      if (cluster.spec.backup?.enabled) {
        await this.ensureBackupCronJob(cluster);
      }

      await this.updateStatus(cluster, {
        phase: "Running",
        observedGeneration: cluster.metadata.generation,
        primaryEndpoint: `${name}-primary.${namespace}.svc.cluster.local`,
      });

      console.log(`Successfully reconciled: ${name}`);
    } catch (error) {
      console.error(`Failed to reconcile ${name}:`, error);
      await this.updateStatus(cluster, {
        phase: "Failed",
      });
    }
  }

  private async ensureStatefulSet(cluster: DatabaseCluster): Promise<void> {
    const { name, namespace } = cluster.metadata;
    const statefulSetName = `${name}-db`;

    const statefulSet = {
      apiVersion: "apps/v1",
      kind: "StatefulSet",
      metadata: {
        name: statefulSetName,
        namespace,
        ownerReferences: [
          {
            apiVersion: `${this.GROUP}/${this.VERSION}`,
            kind: "DatabaseCluster",
            name,
            uid: cluster.metadata.resourceVersion ?? "",
            controller: true,
            blockOwnerDeletion: true,
          },
        ],
      },
      spec: {
        serviceName: `${name}-headless`,
        replicas: cluster.spec.replicas,
        selector: { matchLabels: { app: statefulSetName } },
        template: {
          metadata: { labels: { app: statefulSetName } },
          spec: {
            containers: [
              {
                name: "database",
                image: `${cluster.spec.engine}:${cluster.spec.version}`,
                ports: [{ containerPort: this.getPort(cluster.spec.engine) }],
                resources: cluster.spec.resources ?? {},
                volumeMounts: [
                  {
                    name: "data",
                    mountPath: "/var/lib/postgresql/data",
                  },
                ],
                env: this.getDatabaseEnv(cluster),
                livenessProbe: {
                  exec: {
                    command: this.getLivenessCommand(cluster.spec.engine),
                  },
                  initialDelaySeconds: 30,
                  periodSeconds: 10,
                },
                readinessProbe: {
                  exec: {
                    command: this.getReadinessCommand(cluster.spec.engine),
                  },
                  initialDelaySeconds: 5,
                  periodSeconds: 5,
                },
              },
            ],
          },
        },
        volumeClaimTemplates: [
          {
            metadata: { name: "data" },
            spec: {
              accessModes: ["ReadWriteOnce"],
              storageClassName: cluster.spec.storage.storageClass ?? "standard",
              resources: {
                requests: { storage: cluster.spec.storage.size },
              },
            },
          },
        ],
      },
    };

    try {
      await this.appsApi.readNamespacedStatefulSet(statefulSetName, namespace);
      // Update existing
      await this.appsApi.patchNamespacedStatefulSet(
        statefulSetName,
        namespace,
        statefulSet,
        undefined,
        undefined,
        undefined,
        undefined,
        undefined,
        { headers: { "Content-Type": "application/merge-patch+json" } }
      );
    } catch (error: any) {
      if (error.response?.statusCode === 404) {
        // Create new
        await this.appsApi.createNamespacedStatefulSet(namespace, statefulSet as any);
      } else {
        throw error;
      }
    }
  }

  private async updateStatus(
    cluster: DatabaseCluster,
    status: Partial<DatabaseCluster["status"]>
  ): Promise<void> {
    await this.customObjects.patchNamespacedCustomObjectStatus(
      this.GROUP,
      this.VERSION,
      cluster.metadata.namespace,
      this.PLURAL,
      cluster.metadata.name,
      { status },
      undefined,
      undefined,
      undefined,
      { headers: { "Content-Type": "application/merge-patch+json" } }
    );
  }

  private getPort(engine: string): number {
    const ports: Record<string, number> = {
      postgresql: 5432,
      mysql: 3306,
      mongodb: 27017,
    };
    return ports[engine] ?? 5432;
  }

  private getDatabaseEnv(cluster: DatabaseCluster) {
    return [
      { name: "POSTGRES_DB", value: "app" },
      {
        name: "POSTGRES_PASSWORD",
        valueFrom: {
          secretKeyRef: {
            name: `${cluster.metadata.name}-credentials`,
            key: "password",
          },
        },
      },
    ];
  }

  private getLivenessCommand(engine: string): string[] {
    const commands: Record<string, string[]> = {
      postgresql: ["pg_isready", "-U", "postgres"],
      mysql: ["mysqladmin", "ping", "-h", "localhost"],
      mongodb: ["mongo", "--eval", "db.adminCommand('ping')"],
    };
    return commands[engine] ?? ["true"];
  }

  private getReadinessCommand(engine: string): string[] {
    return this.getLivenessCommand(engine);
  }

  private async ensureServices(_cluster: DatabaseCluster): Promise<void> {
    // Implementation for creating primary and headless services
  }

  private async ensurePVCs(_cluster: DatabaseCluster): Promise<void> {
    // Implementation for creating PVCs
  }

  private async ensureBackupCronJob(_cluster: DatabaseCluster): Promise<void> {
    // Implementation for creating backup CronJob
  }

  async cleanup(cluster: DatabaseCluster): Promise<void> {
    // Kubernetes handles cleanup via ownerReferences
    console.log(`Cleanup triggered for: ${cluster.metadata.name}`);
  }
}

// main.ts
const operator = new DatabaseClusterOperator();
operator.start().catch(console.error);

// Handle signals
process.on("SIGTERM", () => {
  console.log("Received SIGTERM, shutting down...");
  process.exit(0);
});
```

---

## 3. Admission Controllers

Admission Controllers สกัดกั้น API requests ก่อนที่จะถูก persist เข้า etcd

```typescript
// admission-controller/webhook-server.ts
import express from "express";
import https from "https";
import { readFileSync } from "fs";
import { AdmissionReview, AdmissionResponse } from "@kubernetes/client-node";

interface AdmissionWebhook {
  name: string;
  handle: (request: AdmissionReview) => Promise<AdmissionResponse>;
}

class AdmissionWebhookServer {
  private readonly app: express.Application;
  private readonly webhooks: Map<string, AdmissionWebhook>;

  constructor() {
    this.app = express();
    this.webhooks = new Map();
    this.app.use(express.json());
    this.setupRoutes();
  }

  register(path: string, webhook: AdmissionWebhook): void {
    this.webhooks.set(path, webhook);
  }

  private setupRoutes(): void {
    this.app.post("/validate/*", this.handleAdmission.bind(this));
    this.app.post("/mutate/*", this.handleAdmission.bind(this));
    this.app.get("/healthz", (_req, res) => res.json({ status: "ok" }));
  }

  private async handleAdmission(
    req: express.Request,
    res: express.Response
  ): Promise<void> {
    const review = req.body as AdmissionReview;
    const webhook = this.webhooks.get(req.path);

    if (!webhook) {
      res.status(404).json({ error: "Unknown webhook path" });
      return;
    }

    try {
      const response = await webhook.handle(review);
      res.json({
        apiVersion: "admission.k8s.io/v1",
        kind: "AdmissionReview",
        response,
      });
    } catch (error) {
      res.json({
        apiVersion: "admission.k8s.io/v1",
        kind: "AdmissionReview",
        response: {
          uid: review.request?.uid ?? "",
          allowed: false,
          status: {
            code: 500,
            message: (error as Error).message,
          },
        },
      });
    }
  }

  start(port: number, certPath: string, keyPath: string): void {
    const server = https.createServer(
      {
        cert: readFileSync(certPath),
        key: readFileSync(keyPath),
      },
      this.app
    );

    server.listen(port, () => {
      console.log(`Admission webhook server listening on port ${port}`);
    });
  }
}

// webhooks/require-labels.webhook.ts
const requireLabelsWebhook: AdmissionWebhook = {
  name: "require-labels",
  async handle(review) {
    const request = review.request!;
    const object = request.object as any;
    const labels = object?.metadata?.labels ?? {};

    const requiredLabels = ["app", "version", "team"];
    const missingLabels = requiredLabels.filter((l) => !labels[l]);

    if (missingLabels.length > 0) {
      return {
        uid: request.uid,
        allowed: false,
        status: {
          code: 403,
          message: `Missing required labels: ${missingLabels.join(", ")}`,
        },
      };
    }

    return { uid: request.uid, allowed: true };
  },
};

// webhooks/inject-sidecar.webhook.ts (Mutating)
const injectSidecarWebhook: AdmissionWebhook = {
  name: "inject-sidecar",
  async handle(review) {
    const request = review.request!;
    const pod = request.object as any;

    // Check if sidecar injection is requested
    if (pod.metadata?.annotations?.["sidecar.inject/enabled"] !== "true") {
      return { uid: request.uid, allowed: true };
    }

    // Create JSON patch to add sidecar container
    const patch = [
      {
        op: "add",
        path: "/spec/containers/-",
        value: {
          name: "envoy-proxy",
          image: "envoyproxy/envoy:v1.28",
          ports: [{ containerPort: 9901 }],
          resources: {
            limits: { cpu: "100m", memory: "128Mi" },
            requests: { cpu: "50m", memory: "64Mi" },
          },
          volumeMounts: [
            {
              name: "envoy-config",
              mountPath: "/etc/envoy",
            },
          ],
        },
      },
      {
        op: "add",
        path: "/spec/volumes/-",
        value: {
          name: "envoy-config",
          configMap: { name: "envoy-config" },
        },
      },
    ];

    const patchBase64 = Buffer.from(JSON.stringify(patch)).toString("base64");

    return {
      uid: request.uid,
      allowed: true,
      patchType: "JSONPatch",
      patch: patchBase64,
    };
  },
};
```

```yaml
# k8s/admission/webhook-configuration.yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: require-labels-webhook
webhooks:
  - name: require-labels.example.com
    clientConfig:
      service:
        name: admission-webhook
        namespace: kube-system
        path: /validate/require-labels
      caBundle: <base64-encoded-ca-cert>
    rules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE", "UPDATE"]
        resources: ["pods"]
      - apiGroups: ["apps"]
        apiVersions: ["v1"]
        operations: ["CREATE", "UPDATE"]
        resources: ["deployments", "statefulsets"]
    admissionReviewVersions: ["v1"]
    sideEffects: None
    failurePolicy: Fail
    namespaceSelector:
      matchExpressions:
        - key: environment
          operator: In
          values: ["production", "staging"]

---
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: inject-sidecar-webhook
webhooks:
  - name: inject-sidecar.example.com
    clientConfig:
      service:
        name: admission-webhook
        namespace: kube-system
        path: /mutate/inject-sidecar
      caBundle: <base64-encoded-ca-cert>
    rules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE"]
        resources: ["pods"]
    admissionReviewVersions: ["v1"]
    sideEffects: None
    failurePolicy: Ignore  # ถ้า webhook ล้ม ยังสร้าง pod ได้
```

---

## 4. OPA Gatekeeper

OPA Gatekeeper ใช้ Rego language เขียน policies สำหรับ Kubernetes

```yaml
# gatekeeper/constraint-templates/require-resource-limits.yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredresourcelimits
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredResourceLimits
      validation:
        openAPIV3Schema:
          type: object
          properties:
            containers:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredresourcelimits

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.limits.cpu
          msg := sprintf("Container %v must have CPU limit", [container.name])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.limits.memory
          msg := sprintf("Container %v must have memory limit", [container.name])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.requests.cpu
          msg := sprintf("Container %v must have CPU request", [container.name])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.requests.memory
          msg := sprintf("Container %v must have memory request", [container.name])
        }

---
# Apply the constraint
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredResourceLimits
metadata:
  name: require-resource-limits
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    namespaces: ["production", "staging"]
  parameters: {}

---
# Constraint: ไม่อนุญาต privileged containers
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8snoprivilegedcontainer
spec:
  crd:
    spec:
      names:
        kind: K8sNoPrivilegedContainer
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8snoprivilegedcontainer

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          container.securityContext.privileged == true
          msg := sprintf("Privileged container not allowed: %v", [container.name])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.initContainers[_]
          container.securityContext.privileged == true
          msg := sprintf("Privileged init container not allowed: %v", [container.name])
        }

---
# Constraint: image registry whitelist
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sallowedrepos
spec:
  crd:
    spec:
      names:
        kind: K8sAllowedRepos
      validation:
        openAPIV3Schema:
          type: object
          properties:
            repos:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sallowedrepos

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          satisfied := [good | repo = input.parameters.repos[_]; good = startswith(container.image, repo)]
          not any(satisfied)
          msg := sprintf("Container image %v is not from an approved registry", [container.image])
        }

---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRepos
metadata:
  name: allowed-repos
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
  parameters:
    repos:
      - "myregistry.example.com/"
      - "gcr.io/google-containers/"
      - "registry.k8s.io/"
```

---

## 5. Pod Security Standards

```yaml
# namespace-security.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    # Enforce restricted pod security standard
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: v1.28
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: v1.28
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: v1.28

---
# Secure pod template ที่ผ่าน restricted policy
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-app
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: secure-app
  template:
    metadata:
      labels:
        app: secure-app
    spec:
      # ไม่ automount service account token ถ้าไม่จำเป็น
      automountServiceAccountToken: false

      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault

      containers:
        - name: app
          image: myregistry.example.com/app:1.0.0
          ports:
            - containerPort: 8080

          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL

          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"

          volumeMounts:
            - name: tmp
              mountPath: /tmp
            - name: cache
              mountPath: /app/cache

      volumes:
        - name: tmp
          emptyDir: {}
        - name: cache
          emptyDir: {}
```

---

## 6. VerticalPodAutoscaler (VPA)

```yaml
# vpa/order-service-vpa.yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: order-service-vpa
  namespace: production
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: order-service
  updatePolicy:
    updateMode: "Auto"  # Auto | Off | Initial | Recreate
  resourcePolicy:
    containerPolicies:
      - containerName: "order-service"
        minAllowed:
          cpu: "100m"
          memory: "128Mi"
        maxAllowed:
          cpu: "4"
          memory: "4Gi"
        controlledResources:
          - cpu
          - memory
        # VPA จะไม่แนะนำค่าที่ต่ำกว่า minAllowed หรือสูงกว่า maxAllowed

---
# VPA ใน Off mode — แนะนำเท่านั้น ไม่ auto-apply
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: payment-service-vpa
  namespace: production
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: payment-service
  updatePolicy:
    updateMode: "Off"  # แนะนำเท่านั้น ใช้ kubectl describe vpa ดู recommendations
  resourcePolicy:
    containerPolicies:
      - containerName: "*"
        controlledResources: ["cpu", "memory"]
```

---

## 7. KEDA Advanced Scaling

```yaml
# keda/order-processor-scaledobject.yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-processor-scaledobject
  namespace: production
spec:
  scaleTargetRef:
    name: order-processor
  pollingInterval: 15    # Poll triggers every 15 seconds
  cooldownPeriod: 300    # Wait 5 min before scaling down
  minReplicaCount: 0     # Scale to zero when no messages
  maxReplicaCount: 50
  fallback:
    failureThreshold: 3  # If trigger fails 3 times, use fallbackReplicas
    replicas: 5
  triggers:
    # Scale based on RabbitMQ queue depth
    - type: rabbitmq
      metadata:
        protocol: amqp
        queueName: order-processing
        mode: QueueLength
        value: "10"  # 1 replica per 10 messages
      authenticationRef:
        name: rabbitmq-trigger-auth

    # Also scale based on Prometheus metrics
    - type: prometheus
      metadata:
        serverAddress: http://prometheus.monitoring.svc:9090
        metricName: order_queue_processing_rate
        threshold: "100"
        query: |
          sum(rate(order_processing_duration_seconds_count[1m]))

---
# TriggerAuthentication สำหรับ KEDA
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: rabbitmq-trigger-auth
  namespace: production
spec:
  secretTargetRef:
    - parameter: host
      name: rabbitmq-secrets
      key: connection-string

---
# ScaledJob: scale jobs สำหรับ batch processing
apiVersion: keda.sh/v1alpha1
kind: ScaledJob
metadata:
  name: image-processor-scaledjob
  namespace: production
spec:
  jobTargetRef:
    parallelism: 5
    completions: 5
    activeDeadlineSeconds: 600
    backoffLimit: 3
    template:
      spec:
        containers:
          - name: image-processor
            image: myregistry.example.com/image-processor:1.0.0
            env:
              - name: BATCH_SIZE
                value: "10"
            resources:
              requests:
                cpu: "500m"
                memory: "512Mi"
              limits:
                cpu: "2"
                memory: "2Gi"
        restartPolicy: Never
  pollingInterval: 30
  maxReplicaCount: 20
  scalingStrategy:
    strategy: "accurate"
  triggers:
    - type: aws-sqs-queue
      metadata:
        queueURL: https://sqs.ap-southeast-1.amazonaws.com/123456789/image-processing
        queueLength: "5"
        awsRegion: ap-southeast-1
```

---

## 8. Resource Management

```yaml
# resource-management/namespaced-quotas.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    # Compute resources
    requests.cpu: "100"
    requests.memory: "200Gi"
    limits.cpu: "200"
    limits.memory: "400Gi"

    # Object count limits
    pods: "500"
    services: "100"
    secrets: "200"
    configmaps: "200"
    persistentvolumeclaims: "100"

    # LoadBalancer limit (expensive)
    services.loadbalancers: "5"

    # Storage
    requests.storage: "2Ti"

---
apiVersion: v1
kind: LimitRange
metadata:
  name: production-limits
  namespace: production
spec:
  limits:
    # Default limits for containers without explicit limits
    - type: Container
      default:
        cpu: "500m"
        memory: "512Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      max:
        cpu: "4"
        memory: "8Gi"
      min:
        cpu: "50m"
        memory: "64Mi"

    # Limit for PVCs
    - type: PersistentVolumeClaim
      max:
        storage: 100Gi
      min:
        storage: 1Gi

    # Pod-level limits
    - type: Pod
      max:
        cpu: "8"
        memory: "16Gi"
```

---

## 9. Monitoring Kubernetes Operations

```yaml
# monitoring/kubernetes-alerts.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: kubernetes-alerts
  namespace: monitoring
spec:
  groups:
    - name: kubernetes.pods
      rules:
        - alert: PodCrashLooping
          expr: |
            rate(kube_pod_container_status_restarts_total[15m]) > 0
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Pod {{ $labels.pod }} is crash looping"
            description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} container {{ $labels.container }} is restarting frequently"

        - alert: PodNotReady
          expr: |
            kube_pod_status_ready{condition="true"} == 0
          for: 10m
          labels:
            severity: critical
          annotations:
            summary: "Pod {{ $labels.pod }} not ready"

    - name: kubernetes.resources
      rules:
        - alert: HighCPUUsage
          expr: |
            sum(rate(container_cpu_usage_seconds_total{container!=""}[5m])) by (pod, namespace) /
            sum(kube_pod_container_resource_limits{resource="cpu"}) by (pod, namespace) > 0.9
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "High CPU usage in {{ $labels.namespace }}/{{ $labels.pod }}"

        - alert: HighMemoryUsage
          expr: |
            sum(container_memory_usage_bytes{container!=""}) by (pod, namespace) /
            sum(kube_pod_container_resource_limits{resource="memory"}) by (pod, namespace) > 0.85
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "High memory usage in {{ $labels.namespace }}/{{ $labels.pod }}"

        - alert: NodeDiskPressure
          expr: kube_node_status_condition{condition="DiskPressure",status="true"} == 1
          for: 1m
          labels:
            severity: critical
          annotations:
            summary: "Node {{ $labels.node }} has disk pressure"

    - name: kubernetes.cluster
      rules:
        - alert: TooManyPodRestarts
          expr: |
            sum(kube_pod_container_status_restarts_total) by (namespace) > 100
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "High pod restarts in namespace {{ $labels.namespace }}"
```

---

## สรุป

ใน Part นี้เราได้เรียนรู้ Kubernetes Advanced Operations ที่ครอบคลุม:

1. **CRDs** — ขยาย Kubernetes API ด้วย resource types ของตัวเอง พร้อม schema validation
2. **Operator Pattern** — สร้าง controller ที่จัดการ lifecycle ของ custom resources อัตโนมัติ
3. **Admission Controllers** — Validating และ Mutating webhooks สำหรับ policy enforcement
4. **OPA Gatekeeper** — Policy-as-code ด้วย Rego language
5. **Pod Security Standards** — Hardening pods ด้วย security contexts
6. **VPA** — Vertical Pod Autoscaler ปรับ resource requests/limits อัตโนมัติ
7. **KEDA** — Event-driven autoscaling จาก message queues, Prometheus, และ external sources
8. **Resource Management** — ResourceQuota และ LimitRange ควบคุมการใช้ resources

การใช้ patterns เหล่านี้ร่วมกันทำให้ Kubernetes cluster มีความปลอดภัย มีประสิทธิภาพ และดูแลรักษาได้ง่ายใน production

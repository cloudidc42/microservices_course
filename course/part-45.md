# Part 45: Kubernetes Autoscaling ครบวงจร

## บทนำ

Kubernetes Autoscaling เป็นกลไกที่ทำให้ application สามารถปรับขนาดตัวเองได้อัตโนมัติตาม workload ที่เข้ามา ในบทนี้เราจะเรียนรู้ autoscaling ทุกระดับตั้งแต่ Pod ไปจนถึง Node และ Cluster เพื่อให้ระบบมี performance ดีในทุกสถานการณ์ขณะใช้ทรัพยากรอย่างคุ้มค่า

Topics ที่ครอบคลุม:
- Horizontal Pod Autoscaler (HPA) แบบ custom metrics
- KEDA สำหรับ event-driven autoscaling
- Vertical Pod Autoscaler (VPA) สำหรับ right-sizing
- Cluster Autoscaler
- Pod Disruption Budgets
- Priority Classes
- Resource Quotas และ LimitRanges
- Node Affinity และ Pod Anti-Affinity
- Scale-to-Zero ด้วย KEDA

## 1. Horizontal Pod Autoscaler (HPA)

### 1.1 HPA พื้นฐาน - CPU และ Memory

```yaml
# hpa-basic.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: product-service-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: product-service
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: AverageValue
          averageValue: 400Mi
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Pods
          value: 4
          periodSeconds: 60
        - type: Percent
          value: 100
          periodSeconds: 60
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 2
          periodSeconds: 120
        - type: Percent
          value: 10
          periodSeconds: 120
      selectPolicy: Min
```

### 1.2 HPA ด้วย Custom Metrics จาก Prometheus

```yaml
# custom-metrics-hpa.yaml
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
  minReplicas: 2
  maxReplicas: 50
  metrics:
    # CPU baseline
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
    # Custom metric: requests per second per pod
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "100"
    # External metric: queue length
    - type: External
      external:
        metric:
          name: rabbitmq_queue_messages_ready
          selector:
            matchLabels:
              queue: order-processing
        target:
          type: AverageValue
          averageValue: "50"
    # Object metric: Ingress RPS
    - type: Object
      object:
        metric:
          name: requests-per-second
        describedObject:
          apiVersion: networking.k8s.io/v1
          kind: Ingress
          name: order-ingress
        target:
          type: Value
          value: "1000"
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
        - type: Pods
          value: 10
          periodSeconds: 30
        - type: Percent
          value: 50
          periodSeconds: 30
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 600
      policies:
        - type: Pods
          value: 1
          periodSeconds: 300
      selectPolicy: Min
```

### 1.3 Prometheus Adapter สำหรับ Custom Metrics

```yaml
# prometheus-adapter-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: adapter-config
  namespace: monitoring
data:
  config.yaml: |
    rules:
      # HTTP requests per second per pod
      - seriesQuery: 'http_requests_total{namespace!="",pod!=""}'
        resources:
          overrides:
            namespace:
              resource: namespace
            pod:
              resource: pod
        name:
          matches: "^(.*)_total$"
          as: "${1}_per_second"
        metricsQuery: 'sum(rate(<<.Series>>{<<.LabelMatchers>>}[2m])) by (<<.GroupBy>>)'
      
      # Queue depth
      - seriesQuery: 'rabbitmq_queue_messages_ready{namespace!="",queue!=""}'
        resources:
          overrides:
            namespace:
              resource: namespace
        name:
          matches: "^rabbitmq_(.*)$"
          as: "rabbitmq_${1}"
        metricsQuery: 'max(<<.Series>>{<<.LabelMatchers>>}) by (<<.GroupBy>>)'
      
      # Database connections
      - seriesQuery: 'pg_stat_activity_count{namespace!="",pod!=""}'
        resources:
          overrides:
            namespace:
              resource: namespace
            pod:
              resource: pod
        name:
          as: "postgres_connections"
        metricsQuery: 'sum(<<.Series>>{<<.LabelMatchers>>}) by (<<.GroupBy>>)'
      
      # Cache hit rate (scale down when high)
      - seriesQuery: 'redis_keyspace_hits_total{namespace!="",pod!=""}'
        resources:
          overrides:
            namespace:
              resource: namespace
            pod:
              resource: pod
        name:
          as: "redis_hit_rate"
        metricsQuery: |
          rate(redis_keyspace_hits_total{<<.LabelMatchers>>}[5m]) /
          (rate(redis_keyspace_hits_total{<<.LabelMatchers>>}[5m]) +
           rate(redis_keyspace_misses_total{<<.LabelMatchers>>}[5m]))
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus-adapter
  namespace: monitoring
spec:
  replicas: 2
  selector:
    matchLabels:
      app: prometheus-adapter
  template:
    metadata:
      labels:
        app: prometheus-adapter
    spec:
      serviceAccountName: prometheus-adapter
      containers:
        - name: prometheus-adapter
          image: k8s.gcr.io/prometheus-adapter/prometheus-adapter:v0.11.0
          args:
            - --cert-dir=/var/run/serving-cert
            - --config=/etc/adapter/config.yaml
            - --logtostderr=true
            - --metrics-relist-interval=30s
            - --prometheus-url=http://prometheus.monitoring:9090
            - --secure-port=6443
            - --tls-cipher-suites=TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
          ports:
            - containerPort: 6443
          volumeMounts:
            - name: config
              mountPath: /etc/adapter
            - name: serving-cert
              mountPath: /var/run/serving-cert
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi
      volumes:
        - name: config
          configMap:
            name: adapter-config
        - name: serving-cert
          emptyDir: {}
```

### 1.4 TypeScript HPA Manager

```typescript
// hpa-manager.ts
import * as k8s from "@kubernetes/client-node";
import {
  AutoscalingV2Api,
  V2HorizontalPodAutoscaler,
} from "@kubernetes/client-node";

interface HPAStatus {
  name: string;
  namespace: string;
  currentReplicas: number;
  desiredReplicas: number;
  minReplicas: number;
  maxReplicas: number;
  conditions: Array<{
    type: string;
    status: string;
    reason: string;
    message: string;
  }>;
  metrics: Array<{
    type: string;
    currentValue: string;
    targetValue: string;
  }>;
  lastScaleTime?: Date;
}

class HPAManager {
  private autoscalingApi: AutoscalingV2Api;

  constructor() {
    const kc = new k8s.KubeConfig();
    kc.loadFromDefault();
    this.autoscalingApi = kc.makeApiClient(AutoscalingV2Api);
  }

  async getHPAStatus(
    name: string,
    namespace: string
  ): Promise<HPAStatus | null> {
    try {
      const { body } =
        await this.autoscalingApi.readNamespacedHorizontalPodAutoscaler(
          name,
          namespace
        );

      const status = body.status;
      const spec = body.spec;

      if (!status || !spec) return null;

      const metrics = (status.currentMetrics || []).map((m) => {
        let currentValue = "unknown";
        let targetValue = "unknown";
        let type = m.type;

        if (m.type === "Resource" && m.resource) {
          type = `Resource/${m.resource.name}`;
          currentValue = m.resource.current?.averageUtilization?.toString() || 
            m.resource.current?.averageValue || "unknown";
          const targetSpec = spec.metrics?.find(
            (s) => s.type === "Resource" && s.resource?.name === m.resource!.name
          );
          targetValue =
            targetSpec?.resource?.target?.averageUtilization?.toString() ||
            targetSpec?.resource?.target?.averageValue ||
            "unknown";
        } else if (m.type === "Pods" && m.pods) {
          type = `Pods/${m.pods.metric.name}`;
          currentValue = m.pods.current?.averageValue || "unknown";
        } else if (m.type === "External" && m.external) {
          type = `External/${m.external.metric.name}`;
          currentValue = m.external.current?.averageValue || 
            m.external.current?.value || "unknown";
        }

        return { type, currentValue, targetValue };
      });

      return {
        name,
        namespace,
        currentReplicas: status.currentReplicas || 0,
        desiredReplicas: status.desiredReplicas || 0,
        minReplicas: spec.minReplicas || 1,
        maxReplicas: spec.maxReplicas,
        conditions: (status.conditions || []).map((c) => ({
          type: c.type,
          status: c.status,
          reason: c.reason || "",
          message: c.message || "",
        })),
        metrics,
        lastScaleTime: status.lastScaleTime
          ? new Date(status.lastScaleTime)
          : undefined,
      };
    } catch (error) {
      console.error(`Failed to get HPA status: ${error}`);
      return null;
    }
  }

  async updateHPALimits(
    name: string,
    namespace: string,
    minReplicas: number,
    maxReplicas: number
  ): Promise<void> {
    const patch = [
      { op: "replace", path: "/spec/minReplicas", value: minReplicas },
      { op: "replace", path: "/spec/maxReplicas", value: maxReplicas },
    ];

    await this.autoscalingApi.patchNamespacedHorizontalPodAutoscaler(
      name,
      namespace,
      patch,
      undefined,
      undefined,
      undefined,
      undefined,
      undefined,
      { headers: { "Content-Type": "application/json-patch+json" } }
    );

    console.log(
      `Updated HPA ${namespace}/${name}: min=${minReplicas}, max=${maxReplicas}`
    );
  }

  async listAllHPAs(namespace?: string): Promise<HPAStatus[]> {
    const result = namespace
      ? await this.autoscalingApi.listNamespacedHorizontalPodAutoscaler(
          namespace
        )
      : await this.autoscalingApi.listHorizontalPodAutoscalerForAllNamespaces();

    const statuses: HPAStatus[] = [];

    for (const hpa of result.body.items) {
      const status = await this.getHPAStatus(
        hpa.metadata?.name || "",
        hpa.metadata?.namespace || ""
      );
      if (status) statuses.push(status);
    }

    return statuses;
  }

  async watchHPAChanges(
    namespace: string,
    callback: (hpa: V2HorizontalPodAutoscaler, type: string) => void
  ): Promise<void> {
    const watch = new k8s.Watch(new k8s.KubeConfig());
    await watch.watch(
      `/apis/autoscaling/v2/namespaces/${namespace}/horizontalpodautoscalers`,
      {},
      (type, obj) => callback(obj as V2HorizontalPodAutoscaler, type),
      (err) => console.error("Watch error:", err)
    );
  }
}

// Scheduled scaling: scale up before business hours, scale down after
async function applyScheduledScaling(): Promise<void> {
  const manager = new HPAManager();
  const now = new Date();
  const hour = now.getHours();

  const services = [
    { name: "product-service-hpa", namespace: "production" },
    { name: "order-service-hpa", namespace: "production" },
    { name: "user-service-hpa", namespace: "production" },
  ];

  // Business hours: 8am-8pm = higher min replicas
  const isBusinessHours = hour >= 8 && hour < 20;
  const minReplicas = isBusinessHours ? 5 : 2;
  const maxReplicas = isBusinessHours ? 50 : 20;

  for (const svc of services) {
    try {
      await manager.updateHPALimits(
        svc.name,
        svc.namespace,
        minReplicas,
        maxReplicas
      );
      console.log(
        `Updated ${svc.name}: min=${minReplicas} max=${maxReplicas} (business hours: ${isBusinessHours})`
      );
    } catch (error) {
      console.error(`Failed to update ${svc.name}: ${error}`);
    }
  }
}

export { HPAManager, applyScheduledScaling };
```

## 2. KEDA สำหรับ Event-Driven Autoscaling

### 2.1 KEDA Installation

```bash
# ติดตั้ง KEDA ด้วย Helm
helm repo add kedacore https://kedacore.github.io/charts
helm repo update

helm install keda kedacore/keda \
  --namespace keda \
  --create-namespace \
  --set resources.operator.requests.cpu=100m \
  --set resources.operator.requests.memory=100Mi \
  --set resources.operator.limits.cpu=1000m \
  --set resources.operator.limits.memory=1000Mi \
  --set prometheus.metricServer.enabled=true \
  --set prometheus.metricServer.port=9022
```

### 2.2 KEDA ScaledObject สำหรับ Kafka

```yaml
# keda-kafka-scaler.yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-processor-kafka-scaler
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-processor
  pollingInterval: 15  # seconds
  cooldownPeriod: 60   # seconds
  idleReplicaCount: 0  # scale to zero when no messages
  minReplicaCount: 0
  maxReplicaCount: 100
  fallback:
    failureThreshold: 3
    replicas: 3
  advanced:
    restoreToOriginalReplicaCount: false
    horizontalPodAutoscalerConfig:
      behavior:
        scaleUp:
          stabilizationWindowSeconds: 30
          policies:
            - type: Pods
              value: 20
              periodSeconds: 30
        scaleDown:
          stabilizationWindowSeconds: 120
          policies:
            - type: Pods
              value: 5
              periodSeconds: 60
  triggers:
    - type: kafka
      metadata:
        bootstrapServers: "kafka-cluster.kafka:9092"
        consumerGroup: "order-processor-group"
        topic: "orders"
        lagThreshold: "50"          # Scale up when lag > 50
        activationLagThreshold: "5" # Activate scaler when lag > 5
        offsetResetPolicy: "latest"
        allowIdleConsumers: "true"
        scaleToZeroOnInvalidOffset: "false"
        excludePersistentLag: "false"
        version: "2.0.0"
        partitionLimitation: "0,1,2,3,4"  # Watch specific partitions
      authenticationRef:
        name: kafka-auth
---
# Kafka TriggerAuthentication
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: kafka-auth
  namespace: production
spec:
  secretTargetRef:
    - parameter: sasl
      name: kafka-secret
      key: sasl-mechanism
    - parameter: username
      name: kafka-secret
      key: username
    - parameter: password
      name: kafka-secret
      key: password
    - parameter: tls
      name: kafka-secret
      key: tls
    - parameter: ca
      name: kafka-secret
      key: ca-cert
    - parameter: cert
      name: kafka-secret
      key: client-cert
    - parameter: key
      name: kafka-secret
      key: client-key
---
apiVersion: v1
kind: Secret
metadata:
  name: kafka-secret
  namespace: production
type: Opaque
stringData:
  sasl-mechanism: "SCRAM-SHA-512"
  username: "order-processor"
  password: "your-password-here"
  tls: "enable"
```

### 2.3 KEDA ScaledObject สำหรับ RabbitMQ

```yaml
# keda-rabbitmq-scaler.yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: email-worker-rabbitmq-scaler
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: email-worker
  pollingInterval: 10
  cooldownPeriod: 30
  minReplicaCount: 0
  maxReplicaCount: 30
  triggers:
    - type: rabbitmq
      metadata:
        host: "amqp://rabbitmq.production:5672"
        queueName: "email-queue"
        mode: "QueueLength"    # QueueLength or MessageRate
        value: "10"            # Scale 1 pod per 10 messages
        activationValue: "1"   # Activate when queue > 1
        timeout: "30"
      authenticationRef:
        name: rabbitmq-auth
---
# RabbitMQ with management API
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: notification-worker-rabbitmq-scaler
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: notification-worker
  pollingInterval: 10
  cooldownPeriod: 60
  minReplicaCount: 1
  maxReplicaCount: 50
  triggers:
    - type: rabbitmq
      metadata:
        protocol: "amqp"
        queueName: "notifications"
        mode: "MessageRate"
        value: "100"          # Messages/sec per pod
        vhostName: "/production"
      authenticationRef:
        name: rabbitmq-auth
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: rabbitmq-auth
  namespace: production
spec:
  secretTargetRef:
    - parameter: host
      name: rabbitmq-secret
      key: connection-string
```

### 2.4 KEDA ScaledObject สำหรับ HTTP (HTTP Add-on)

```yaml
# keda-http-scaler.yaml
# ต้องติดตั้ง http-add-on ก่อน
apiVersion: http.keda.sh/v1alpha1
kind: HTTPScaledObject
metadata:
  name: api-service-http-scaler
  namespace: production
spec:
  host: "api.example.com"
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-service
    service: api-service
    port: 80
  replicas:
    min: 0
    max: 100
  scaledownPeriod: 300
  targetPendingRequests: 100
  scalingMetric:
    requestRate:
      granularity: 1s
      targetValue: 200  # 200 RPS per pod
      window: 1m
---
# KEDA ScaledJob สำหรับ batch processing
apiVersion: keda.sh/v1alpha1
kind: ScaledJob
metadata:
  name: report-generator
  namespace: production
spec:
  jobTargetRef:
    parallelism: 1
    completions: 1
    activeDeadlineSeconds: 600
    backoffLimit: 3
    template:
      spec:
        restartPolicy: Never
        containers:
          - name: report-generator
            image: myregistry/report-generator:latest
            env:
              - name: QUEUE_URL
                value: "sqs://report-queue"
            resources:
              requests:
                cpu: 500m
                memory: 1Gi
              limits:
                cpu: 2000m
                memory: 4Gi
  pollingInterval: 30
  maxReplicaCount: 10
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 5
  scalingStrategy:
    strategy: "accurate"  # accurate, default, or custom
    pendingPodConditions:
      - "Ready"
      - "PodScheduled"
    customScalingQueueLengthDeduction: 1
    customScalingRunningJobPercentage: "0.5"
  triggers:
    - type: aws-sqs-queue
      authenticationRef:
        name: aws-auth
      metadata:
        queueURL: "https://sqs.ap-southeast-1.amazonaws.com/123456789/report-queue"
        queueLength: "5"
        awsRegion: "ap-southeast-1"
        scaleOnInFlight: "true"
```

### 2.5 KEDA Prometheus Scaler

```yaml
# keda-prometheus-scaler.yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: recommendation-service-scaler
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: recommendation-service
  minReplicaCount: 1
  maxReplicaCount: 30
  triggers:
    # Scale on active user sessions
    - type: prometheus
      metadata:
        serverAddress: http://prometheus.monitoring:9090
        metricName: active_user_sessions
        threshold: "500"           # 500 active sessions per pod
        activationThreshold: "100"
        query: |
          sum(rate(http_sessions_active_total{
            namespace="production",
            service="recommendation-service"
          }[5m]))
        namespace: production
    # Scale on ML inference queue
    - type: prometheus
      metadata:
        serverAddress: http://prometheus.monitoring:9090
        metricName: ml_inference_queue_depth
        threshold: "20"
        query: |
          sum(ml_inference_queue_depth{
            namespace="production",
            service="recommendation-service"
          })
---
# KEDA External Scaler (custom gRPC scaler)
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: custom-workload-scaler
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: custom-worker
  minReplicaCount: 0
  maxReplicaCount: 50
  triggers:
    - type: external
      metadata:
        scalerAddress: "custom-scaler-service.production:9090"
        workloadType: "high-priority-batch"
        threshold: "10"
```

### 2.6 TypeScript KEDA ScaledObject Creator

```typescript
// keda-manager.ts
import * as k8s from "@kubernetes/client-node";
import { CustomObjectsApi } from "@kubernetes/client-node";

type ScalerTriggerType = "kafka" | "rabbitmq" | "redis" | "prometheus" | "aws-sqs-queue" | "azure-servicebus";

interface KafkaScalerConfig {
  bootstrapServers: string;
  consumerGroup: string;
  topic: string;
  lagThreshold: number;
  minReplicas?: number;
  maxReplicas?: number;
  scaleToZero?: boolean;
}

interface RabbitMQScalerConfig {
  host: string;
  queueName: string;
  messagesPerPod: number;
  minReplicas?: number;
  maxReplicas?: number;
}

interface ScaledObjectConfig {
  name: string;
  namespace: string;
  deploymentName: string;
  minReplicas: number;
  maxReplicas: number;
  cooldownPeriod?: number;
  pollingInterval?: number;
  triggerType: ScalerTriggerType;
  triggerConfig: KafkaScalerConfig | RabbitMQScalerConfig;
  authRef?: string;
}

class KEDAManager {
  private customApi: CustomObjectsApi;

  constructor() {
    const kc = new k8s.KubeConfig();
    kc.loadFromDefault();
    this.customApi = kc.makeApiClient(CustomObjectsApi);
  }

  async createKafkaScaledObject(
    name: string,
    namespace: string,
    deploymentName: string,
    config: KafkaScalerConfig
  ): Promise<void> {
    const scaledObject = {
      apiVersion: "keda.sh/v1alpha1",
      kind: "ScaledObject",
      metadata: {
        name,
        namespace,
        labels: {
          "scaler-type": "kafka",
          "managed-by": "keda-manager",
        },
      },
      spec: {
        scaleTargetRef: {
          apiVersion: "apps/v1",
          kind: "Deployment",
          name: deploymentName,
        },
        pollingInterval: 15,
        cooldownPeriod: 60,
        idleReplicaCount: config.scaleToZero ? 0 : config.minReplicas || 1,
        minReplicaCount: config.minReplicas || 0,
        maxReplicaCount: config.maxReplicas || 100,
        triggers: [
          {
            type: "kafka",
            metadata: {
              bootstrapServers: config.bootstrapServers,
              consumerGroup: config.consumerGroup,
              topic: config.topic,
              lagThreshold: config.lagThreshold.toString(),
              activationLagThreshold: Math.floor(
                config.lagThreshold * 0.1
              ).toString(),
              offsetResetPolicy: "latest",
              allowIdleConsumers: "true",
            },
          },
        ],
      },
    };

    await this.customApi.createNamespacedCustomObject(
      "keda.sh",
      "v1alpha1",
      namespace,
      "scaledobjects",
      scaledObject
    );

    console.log(`Created Kafka ScaledObject: ${namespace}/${name}`);
  }

  async getScaledObjectStatus(
    name: string,
    namespace: string
  ): Promise<Record<string, unknown>> {
    const result = await this.customApi.getNamespacedCustomObjectStatus(
      "keda.sh",
      "v1alpha1",
      namespace,
      "scaledobjects",
      name
    );

    return (result.body as Record<string, unknown>).status as Record<string, unknown> || {};
  }

  async pauseScaling(name: string, namespace: string): Promise<void> {
    const patch = [
      {
        op: "add",
        path: "/metadata/annotations",
        value: {
          "autoscaling.keda.sh/paused-replicas": "1",
        },
      },
    ];

    await this.customApi.patchNamespacedCustomObject(
      "keda.sh",
      "v1alpha1",
      namespace,
      "scaledobjects",
      name,
      patch,
      undefined,
      undefined,
      undefined,
      { headers: { "Content-Type": "application/json-patch+json" } }
    );

    console.log(`Paused scaling for ${namespace}/${name}`);
  }

  async resumeScaling(name: string, namespace: string): Promise<void> {
    const patch = [
      {
        op: "remove",
        path: "/metadata/annotations/autoscaling.keda.sh~1paused-replicas",
      },
    ];

    await this.customApi.patchNamespacedCustomObject(
      "keda.sh",
      "v1alpha1",
      namespace,
      "scaledobjects",
      name,
      patch,
      undefined,
      undefined,
      undefined,
      { headers: { "Content-Type": "application/json-patch+json" } }
    );

    console.log(`Resumed scaling for ${namespace}/${name}`);
  }

  async listScaledObjects(namespace: string): Promise<Array<{
    name: string;
    namespace: string;
    currentReplicas: number;
    desiredReplicas: number;
    active: boolean;
  }>> {
    const result = await this.customApi.listNamespacedCustomObject(
      "keda.sh",
      "v1alpha1",
      namespace,
      "scaledobjects"
    );

    const body = result.body as { items?: Array<Record<string, unknown>> };
    return (body.items || []).map((item) => {
      const metadata = item.metadata as Record<string, unknown>;
      const status = (item.status || {}) as Record<string, unknown>;
      return {
        name: metadata.name as string,
        namespace: metadata.namespace as string,
        currentReplicas: (status.currentReplicas as number) || 0,
        desiredReplicas: (status.desiredReplicas as number) || 0,
        active: (status.active as boolean) || false,
      };
    });
  }
}

export { KEDAManager };
```

## 3. Vertical Pod Autoscaler (VPA)

### 3.1 VPA Installation และ Configuration

```yaml
# vpa-installation.yaml
# ติดตั้ง VPA components
# kubectl apply -f https://github.com/kubernetes/autoscaler/releases/download/vertical-pod-autoscaler-0.14.0/vpa-v0.14.0.yaml
---
# VPA for right-sizing
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: product-service-vpa
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: product-service
  updatePolicy:
    updateMode: "Auto"        # Auto, Recreate, Initial, Off
    minReplicas: 2            # Don't scale if fewer than 2 replicas
    evictionRequirements:     # Minimum requirements to allow eviction
      - resources: ["cpu"]
        changeRequirement: TargetHigherThanRequests
  resourcePolicy:
    containerPolicies:
      - containerName: product-service
        minAllowed:
          cpu: 50m
          memory: 64Mi
        maxAllowed:
          cpu: 4000m
          memory: 8Gi
        controlledResources:
          - cpu
          - memory
        controlledValues: RequestsAndLimits
---
# VPA in "Off" mode for recommendations only
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: database-proxy-vpa
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: database-proxy
  updatePolicy:
    updateMode: "Off"   # Only recommendations, no automatic updates
  resourcePolicy:
    containerPolicies:
      - containerName: pgbouncer
        minAllowed:
          cpu: 100m
          memory: 256Mi
        maxAllowed:
          cpu: 2000m
          memory: 2Gi
---
# VPA for StatefulSet
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: redis-vpa
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: StatefulSet
    name: redis
  updatePolicy:
    updateMode: "Initial"   # Only set on pod creation
  resourcePolicy:
    containerPolicies:
      - containerName: redis
        minAllowed:
          cpu: 100m
          memory: 256Mi
        maxAllowed:
          cpu: 1000m
          memory: 4Gi
        controlledResources: ["memory"]   # Only control memory
```

### 3.2 TypeScript VPA Recommendation Reader

```typescript
// vpa-analyzer.ts
import * as k8s from "@kubernetes/client-node";
import { CustomObjectsApi } from "@kubernetes/client-node";

interface VPARecommendation {
  containerName: string;
  lowerBound: { cpu: string; memory: string };
  target: { cpu: string; memory: string };
  uncappedTarget: { cpu: string; memory: string };
  upperBound: { cpu: string; memory: string };
}

interface VPAReport {
  name: string;
  namespace: string;
  targetRef: { kind: string; name: string };
  updateMode: string;
  recommendations: VPARecommendation[];
  currentResources?: Array<{
    containerName: string;
    requestedCpu: string;
    requestedMemory: string;
    limitCpu: string;
    limitMemory: string;
  }>;
  savingsEstimate?: {
    cpuSavingsPercent: number;
    memorySavingsPercent: number;
  };
}

class VPAAnalyzer {
  private customApi: CustomObjectsApi;
  private coreApi: k8s.CoreV1Api;
  private appsApi: k8s.AppsV1Api;

  constructor() {
    const kc = new k8s.KubeConfig();
    kc.loadFromDefault();
    this.customApi = kc.makeApiClient(CustomObjectsApi);
    this.coreApi = kc.makeApiClient(k8s.CoreV1Api);
    this.appsApi = kc.makeApiClient(k8s.AppsV1Api);
  }

  parseResource(value: string): number {
    if (value.endsWith("m")) {
      return parseInt(value) / 1000;
    }
    if (value.endsWith("Mi")) {
      return parseInt(value);
    }
    if (value.endsWith("Gi")) {
      return parseInt(value) * 1024;
    }
    return parseInt(value) || 0;
  }

  async getVPARecommendations(namespace: string): Promise<VPAReport[]> {
    const result = await this.customApi.listNamespacedCustomObject(
      "autoscaling.k8s.io",
      "v1",
      namespace,
      "verticalpodautoscalers"
    );

    const body = result.body as { items?: Array<Record<string, unknown>> };
    const reports: VPAReport[] = [];

    for (const vpa of body.items || []) {
      const metadata = vpa.metadata as Record<string, string>;
      const spec = vpa.spec as Record<string, unknown>;
      const status = vpa.status as Record<string, unknown>;

      const targetRef = spec.targetRef as Record<string, string>;
      const updatePolicy = spec.updatePolicy as Record<string, string>;

      const recommendation = (status.recommendation || {}) as Record<string, unknown>;
      const containerRecs = (recommendation.containerRecommendations || []) as Array<Record<string, unknown>>;

      const recommendations: VPARecommendation[] = containerRecs.map((rec) => {
        const toResources = (obj: unknown): { cpu: string; memory: string } => {
          const r = obj as Record<string, string> || {};
          return { cpu: r.cpu || "0", memory: r.memory || "0" };
        };

        return {
          containerName: rec.containerName as string,
          lowerBound: toResources(rec.lowerBound),
          target: toResources(rec.target),
          uncappedTarget: toResources(rec.uncappedTarget),
          upperBound: toResources(rec.upperBound),
        };
      });

      const report: VPAReport = {
        name: metadata.name,
        namespace: metadata.namespace,
        targetRef: {
          kind: targetRef.kind,
          name: targetRef.name,
        },
        updateMode: updatePolicy?.updateMode || "Off",
        recommendations,
      };

      // Get current resource requests for comparison
      try {
        const deploy = await this.appsApi.readNamespacedDeployment(
          targetRef.name,
          namespace
        );
        const containers =
          deploy.body.spec?.template.spec?.containers || [];

        report.currentResources = containers.map((c) => ({
          containerName: c.name,
          requestedCpu: c.resources?.requests?.cpu || "0",
          requestedMemory: c.resources?.requests?.memory || "0",
          limitCpu: c.resources?.limits?.cpu || "0",
          limitMemory: c.resources?.limits?.memory || "0",
        }));

        // Calculate savings estimate
        if (report.currentResources.length > 0 && recommendations.length > 0) {
          const current = report.currentResources[0];
          const rec = recommendations[0];

          if (current && rec) {
            const currentCpu = this.parseResource(current.requestedCpu);
            const targetCpu = this.parseResource(rec.target.cpu);
            const currentMem = this.parseResource(current.requestedMemory);
            const targetMem = this.parseResource(rec.target.memory);

            report.savingsEstimate = {
              cpuSavingsPercent:
                currentCpu > 0
                  ? ((currentCpu - targetCpu) / currentCpu) * 100
                  : 0,
              memorySavingsPercent:
                currentMem > 0
                  ? ((currentMem - targetMem) / currentMem) * 100
                  : 0,
            };
          }
        }
      } catch {
        // Deployment might not exist or be a different resource type
      }

      reports.push(report);
    }

    return reports;
  }

  async generateOptimizationReport(namespace: string): Promise<void> {
    const reports = await this.getVPARecommendations(namespace);

    console.log(`\n=== VPA Optimization Report for namespace: ${namespace} ===\n`);

    for (const report of reports) {
      console.log(`Workload: ${report.targetRef.kind}/${report.targetRef.name}`);
      console.log(`VPA Mode: ${report.updateMode}`);

      if (report.recommendations.length === 0) {
        console.log("  No recommendations yet (collecting data...)\n");
        continue;
      }

      for (const rec of report.recommendations) {
        const current = report.currentResources?.find(
          (c) => c.containerName === rec.containerName
        );

        console.log(`\n  Container: ${rec.containerName}`);
        if (current) {
          console.log(
            `  Current:  CPU=${current.requestedCpu}, Memory=${current.requestedMemory}`
          );
        }
        console.log(
          `  Target:   CPU=${rec.target.cpu}, Memory=${rec.target.memory}`
        );
        console.log(
          `  Range:    CPU=${rec.lowerBound.cpu}-${rec.upperBound.cpu}, Memory=${rec.lowerBound.memory}-${rec.upperBound.memory}`
        );
      }

      if (report.savingsEstimate) {
        const est = report.savingsEstimate;
        console.log(`\n  Estimated savings:`);
        console.log(
          `  CPU: ${est.cpuSavingsPercent > 0 ? "-" : "+"}${Math.abs(est.cpuSavingsPercent).toFixed(1)}%`
        );
        console.log(
          `  Memory: ${est.memorySavingsPercent > 0 ? "-" : "+"}${Math.abs(est.memorySavingsPercent).toFixed(1)}%`
        );
      }

      console.log();
    }
  }
}

export { VPAAnalyzer };
```

## 4. Cluster Autoscaler

### 4.1 Cluster Autoscaler สำหรับ AWS EKS

```yaml
# cluster-autoscaler.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  labels:
    app: cluster-autoscaler
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    metadata:
      labels:
        app: cluster-autoscaler
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8085"
        cluster-autoscaler.kubernetes.io/safe-to-evict: "false"
    spec:
      priorityClassName: system-cluster-critical
      serviceAccountName: cluster-autoscaler
      securityContext:
        runAsNonRoot: true
        runAsUser: 65534
        fsGroup: 65534
      containers:
        - name: cluster-autoscaler
          image: k8s.gcr.io/autoscaling/cluster-autoscaler:v1.28.2
          command:
            - ./cluster-autoscaler
            - --v=4
            - --stderrthreshold=info
            - --cloud-provider=aws
            - --skip-nodes-with-local-storage=false
            - --expander=least-waste
            - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/production-cluster
            - --balance-similar-node-groups=true
            - --skip-nodes-with-system-pods=false
            - --scale-down-enabled=true
            - --scale-down-utilization-threshold=0.5
            - --scale-down-unneeded-time=10m
            - --scale-down-delay-after-add=10m
            - --scale-down-delay-after-delete=0s
            - --scale-down-delay-after-failure=3m
            - --max-graceful-termination-sec=600
            - --max-node-provision-time=15m
            - --scan-interval=10s
            - --max-nodes-total=200
            - --cores-total=0:9600
            - --memory-total=0:76800Gi
            - --emit-per-nodegroup-metrics=true
          env:
            - name: AWS_REGION
              value: ap-southeast-1
          resources:
            limits:
              cpu: 100m
              memory: 600Mi
            requests:
              cpu: 100m
              memory: 600Mi
          volumeMounts:
            - name: ssl-certs
              mountPath: /etc/ssl/certs/ca-certificates.crt
              readOnly: true
          livenessProbe:
            httpGet:
              path: /health-check
              port: 8085
            initialDelaySeconds: 60
            periodSeconds: 10
            failureThreshold: 3
      volumes:
        - name: ssl-certs
          hostPath:
            path: /etc/ssl/certs/ca-bundle.crt
---
# Node Group configurations
# aws autoscaling create-auto-scaling-group
# Tags required for CA discovery:
# k8s.io/cluster-autoscaler/enabled: "true"
# k8s.io/cluster-autoscaler/<cluster-name>: "owned"
# k8s.io/cluster-autoscaler/node-template/resources/ephemeral-storage: "100Gi"
---
# Cluster Autoscaler RBAC
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/cluster-autoscaler-role
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-autoscaler
rules:
  - apiGroups: [""]
    resources: ["events", "endpoints"]
    verbs: ["create", "patch"]
  - apiGroups: [""]
    resources: ["pods/eviction"]
    verbs: ["create"]
  - apiGroups: [""]
    resources: ["pods/status"]
    verbs: ["update"]
  - apiGroups: [""]
    resources: ["endpoints"]
    resourceNames: ["cluster-autoscaler"]
    verbs: ["get", "update"]
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["watch", "list", "get", "update"]
  - apiGroups: [""]
    resources: ["namespaces", "pods", "services", "replicationcontrollers", "persistentvolumeclaims", "persistentvolumes"]
    verbs: ["watch", "list", "get"]
  - apiGroups: ["extensions"]
    resources: ["replicasets", "daemonsets"]
    verbs: ["watch", "list", "get"]
  - apiGroups: ["policy"]
    resources: ["poddisruptionbudgets"]
    verbs: ["watch", "list"]
  - apiGroups: ["apps"]
    resources: ["statefulsets", "replicasets", "daemonsets"]
    verbs: ["watch", "list", "get"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses", "csinodes", "csidrivers", "csistoragecapacities"]
    verbs: ["watch", "list", "get"]
  - apiGroups: ["batch"]
    resources: ["jobs", "cronjobs"]
    verbs: ["watch", "list", "get"]
  - apiGroups: ["coordination.k8s.io"]
    resources: ["leases"]
    verbs: ["create"]
  - apiGroups: ["coordination.k8s.io"]
    resourceNames: ["cluster-autoscaler"]
    resources: ["leases"]
    verbs: ["get", "update"]
```

## 5. Pod Disruption Budgets

### 5.1 PDB สำหรับ Zero-Downtime

```yaml
# pod-disruption-budgets.yaml
# PDB สำหรับ critical services
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: product-service-pdb
  namespace: production
spec:
  minAvailable: 2   # ต้องมี pod running อย่างน้อย 2 ตัวเสมอ
  selector:
    matchLabels:
      app: product-service
---
# PDB ด้วย percentage
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: order-service-pdb
  namespace: production
spec:
  maxUnavailable: 25%  # ปิดได้ไม่เกิน 25% พร้อมกัน
  selector:
    matchLabels:
      app: order-service
---
# PDB สำหรับ StatefulSet database
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: postgres-pdb
  namespace: production
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: postgres
      role: primary
---
# PDB สำหรับ Redis cluster
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: redis-pdb
  namespace: production
spec:
  minAvailable: "51%"   # Quorum สำหรับ Redis Sentinel
  selector:
    matchLabels:
      app: redis
---
# Unhealthy PDB policy
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: search-service-pdb
  namespace: production
spec:
  minAvailable: 1
  maxUnavailable: 0   # ไม่อนุญาตให้ pod unavailable เลย
  unhealthyPodEvictionPolicy: IfHealthyBudget   # Allow evicting unhealthy pods
  selector:
    matchLabels:
      app: search-service
```

## 6. Priority Classes

### 6.1 Priority Classes สำหรับ Critical Workloads

```yaml
# priority-classes.yaml
# Critical infrastructure (highest priority)
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: critical-infrastructure
value: 1000000
globalDefault: false
preemptionPolicy: PreemptLowerPriority
description: "Critical infrastructure components: monitoring, logging, DNS"
---
# Business-critical services
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority-service
value: 100000
globalDefault: false
preemptionPolicy: PreemptLowerPriority
description: "Business-critical services: payment, order processing"
---
# Standard production services
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: standard-service
value: 10000
globalDefault: true
preemptionPolicy: PreemptLowerPriority
description: "Standard production services"
---
# Batch jobs
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: batch-job
value: 1000
globalDefault: false
preemptionPolicy: Never   # Don't preempt others
description: "Background batch processing jobs"
---
# Development/testing (lowest)
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: development
value: 100
globalDefault: false
preemptionPolicy: Never
description: "Development and testing workloads"
---
# Apply to Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payment-service
  template:
    metadata:
      labels:
        app: payment-service
    spec:
      priorityClassName: high-priority-service
      containers:
        - name: payment-service
          image: myregistry/payment-service:1.0.0
          resources:
            requests:
              cpu: 200m
              memory: 256Mi
            limits:
              cpu: 1000m
              memory: 1Gi
```

## 7. Resource Quotas และ LimitRanges

### 7.1 Resource Quotas ต่อ Namespace

```yaml
# resource-quotas.yaml
# Production namespace quota
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    # Compute resources
    requests.cpu: "100"         # Total CPU requests ไม่เกิน 100 cores
    requests.memory: 200Gi
    limits.cpu: "200"
    limits.memory: 400Gi
    
    # Object count limits
    count/pods: "500"
    count/services: "100"
    count/persistentvolumeclaims: "50"
    count/secrets: "200"
    count/configmaps: "200"
    count/replicationcontrollers: "20"
    count/deployments.apps: "100"
    count/statefulsets.apps: "20"
    count/jobs.batch: "50"
    count/cronjobs.batch: "20"
    
    # Storage
    requests.storage: 2Ti
    persistentvolumeclaims: "50"
    
    # Load balancer services
    count/services.loadbalancers: "5"
    count/services.nodeports: "0"
---
# Staging namespace quota (smaller limits)
apiVersion: v1
kind: ResourceQuota
metadata:
  name: staging-quota
  namespace: staging
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    count/pods: "100"
    requests.storage: 500Gi
---
# GPU quota for ML workloads
apiVersion: v1
kind: ResourceQuota
metadata:
  name: ml-quota
  namespace: ml-production
spec:
  hard:
    requests.nvidia.com/gpu: "8"
    limits.nvidia.com/gpu: "8"
    requests.cpu: "64"
    requests.memory: 256Gi
```

### 7.2 LimitRanges ต่อ Namespace

```yaml
# limit-ranges.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: production-limits
  namespace: production
spec:
  limits:
    # Container limits
    - type: Container
      default:
        cpu: 500m
        memory: 512Mi
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      max:
        cpu: 8000m
        memory: 16Gi
      min:
        cpu: 10m
        memory: 16Mi
      maxLimitRequestRatio:
        cpu: "10"        # limit ไม่เกิน 10x ของ request
        memory: "4"
    
    # Pod limits (sum of all containers)
    - type: Pod
      max:
        cpu: 32000m
        memory: 64Gi
      min:
        cpu: 50m
        memory: 64Mi
    
    # PersistentVolumeClaim limits
    - type: PersistentVolumeClaim
      max:
        storage: 200Gi
      min:
        storage: 1Gi
---
# LimitRange for development namespace
apiVersion: v1
kind: LimitRange
metadata:
  name: development-limits
  namespace: development
spec:
  limits:
    - type: Container
      default:
        cpu: 200m
        memory: 256Mi
      defaultRequest:
        cpu: 50m
        memory: 64Mi
      max:
        cpu: 2000m
        memory: 4Gi
      min:
        cpu: 10m
        memory: 16Mi
```

### 7.3 TypeScript Resource Quota Monitor

```typescript
// quota-monitor.ts
import * as k8s from "@kubernetes/client-node";
import { CoreV1Api, V1ResourceQuota } from "@kubernetes/client-node";

interface QuotaUsage {
  namespace: string;
  resource: string;
  used: number;
  hard: number;
  usagePercent: number;
  critical: boolean;
}

interface NamespaceQuotaReport {
  namespace: string;
  quotas: QuotaUsage[];
  overallHealth: "healthy" | "warning" | "critical";
  recommendations: string[];
}

class ResourceQuotaMonitor {
  private coreApi: CoreV1Api;

  constructor() {
    const kc = new k8s.KubeConfig();
    kc.loadFromDefault();
    this.coreApi = kc.makeApiClient(CoreV1Api);
  }

  parseResourceValue(value: string): number {
    if (!value) return 0;

    // CPU
    if (value.endsWith("m")) return parseInt(value) / 1000;
    if (value.endsWith("n")) return parseInt(value) / 1e9;

    // Memory
    const memMatch = value.match(/^(\d+)(Ki|Mi|Gi|Ti|K|M|G|T)?$/);
    if (memMatch) {
      const num = parseInt(memMatch[1] || "0");
      const unit = memMatch[2] || "";
      const multipliers: Record<string, number> = {
        Ki: 1024, Mi: 1024 ** 2, Gi: 1024 ** 3, Ti: 1024 ** 4,
        K: 1000, M: 1000 ** 2, G: 1000 ** 3, T: 1000 ** 4,
      };
      return num * (multipliers[unit] || 1);
    }

    return parseFloat(value) || 0;
  }

  async getNamespaceQuotaUsage(
    namespace: string
  ): Promise<NamespaceQuotaReport> {
    const { body } = await this.coreApi.listNamespacedResourceQuota(namespace);

    const quotaUsages: QuotaUsage[] = [];
    const recommendations: string[] = [];

    for (const quota of body.items) {
      const hard = quota.status?.hard || {};
      const used = quota.status?.used || {};

      for (const [resource, hardValue] of Object.entries(hard)) {
        const usedValue = used[resource] || "0";

        const hardNum = this.parseResourceValue(hardValue);
        const usedNum = this.parseResourceValue(usedValue);
        const usagePercent = hardNum > 0 ? (usedNum / hardNum) * 100 : 0;

        const usage: QuotaUsage = {
          namespace,
          resource,
          used: usedNum,
          hard: hardNum,
          usagePercent,
          critical: usagePercent > 90,
        };

        quotaUsages.push(usage);

        // Generate recommendations
        if (usagePercent > 90) {
          recommendations.push(
            `CRITICAL: ${resource} usage at ${usagePercent.toFixed(1)}% - consider increasing quota or reducing usage`
          );
        } else if (usagePercent > 75) {
          recommendations.push(
            `WARNING: ${resource} usage at ${usagePercent.toFixed(1)}% - monitor closely`
          );
        } else if (usagePercent < 10 && hardNum > 0) {
          recommendations.push(
            `OPTIMIZE: ${resource} usage only ${usagePercent.toFixed(1)}% - quota may be oversized`
          );
        }
      }
    }

    const criticalCount = quotaUsages.filter((u) => u.critical).length;
    const warningCount = quotaUsages.filter(
      (u) => u.usagePercent > 75 && !u.critical
    ).length;

    let overallHealth: "healthy" | "warning" | "critical" = "healthy";
    if (criticalCount > 0) overallHealth = "critical";
    else if (warningCount > 0) overallHealth = "warning";

    return {
      namespace,
      quotas: quotaUsages,
      overallHealth,
      recommendations,
    };
  }

  async generateClusterQuotaReport(
    namespaces: string[]
  ): Promise<NamespaceQuotaReport[]> {
    const reports = await Promise.all(
      namespaces.map((ns) => this.getNamespaceQuotaUsage(ns))
    );

    // Sort by health status (critical first)
    return reports.sort((a, b) => {
      const priority = { critical: 0, warning: 1, healthy: 2 };
      return priority[a.overallHealth] - priority[b.overallHealth];
    });
  }

  printReport(reports: NamespaceQuotaReport[]): void {
    console.log("\n=== Cluster Resource Quota Report ===\n");

    for (const report of reports) {
      const emoji = {
        healthy: "✓",
        warning: "⚠",
        critical: "✗",
      }[report.overallHealth];

      console.log(`[${emoji}] Namespace: ${report.namespace} (${report.overallHealth.toUpperCase()})`);

      // Show top usage metrics
      const sortedQuotas = [...report.quotas].sort(
        (a, b) => b.usagePercent - a.usagePercent
      );

      for (const quota of sortedQuotas.slice(0, 5)) {
        const bar = "█".repeat(Math.floor(quota.usagePercent / 10));
        console.log(
          `  ${quota.resource.padEnd(40)} ${bar.padEnd(10)} ${quota.usagePercent.toFixed(1)}%`
        );
      }

      if (report.recommendations.length > 0) {
        console.log("  Recommendations:");
        report.recommendations.forEach((r) => console.log(`    - ${r}`));
      }

      console.log();
    }
  }
}

export { ResourceQuotaMonitor };
```

## 8. Node Affinity และ Pod Anti-Affinity

### 8.1 Node Affinity Rules

```yaml
# node-affinity.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gpu-inference-service
  namespace: production
spec:
  replicas: 2
  selector:
    matchLabels:
      app: gpu-inference-service
  template:
    metadata:
      labels:
        app: gpu-inference-service
    spec:
      affinity:
        nodeAffinity:
          # Required: must run on GPU nodes
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: node.kubernetes.io/instance-type
                    operator: In
                    values:
                      - p3.2xlarge
                      - p3.8xlarge
                      - g4dn.xlarge
                      - g4dn.2xlarge
                  - key: nvidia.com/gpu
                    operator: Exists
          # Preferred: newer generation GPU
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              preference:
                matchExpressions:
                  - key: node.kubernetes.io/instance-type
                    operator: In
                    values:
                      - p4d.24xlarge
                      - p3.16xlarge
            - weight: 50
              preference:
                matchExpressions:
                  - key: topology.kubernetes.io/zone
                    operator: In
                    values:
                      - ap-southeast-1a
        podAntiAffinity:
          # Required: spread across availability zones
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - gpu-inference-service
              topologyKey: topology.kubernetes.io/zone
          # Preferred: spread across nodes
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: gpu-inference-service
                topologyKey: kubernetes.io/hostname
      tolerations:
        - key: nvidia.com/gpu
          operator: Exists
          effect: NoSchedule
        - key: dedicated
          operator: Equal
          value: gpu
          effect: NoSchedule
      containers:
        - name: inference
          image: myregistry/ml-inference:latest
          resources:
            requests:
              nvidia.com/gpu: "1"
              cpu: 2000m
              memory: 8Gi
            limits:
              nvidia.com/gpu: "1"
              cpu: 8000m
              memory: 32Gi
---
# Database pod anti-affinity (never same node)
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres-ha
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            # Never place two postgres pods on the same node
            - labelSelector:
                matchLabels:
                  app: postgres
              topologyKey: kubernetes.io/hostname
            # Never place two postgres pods in the same zone
            - labelSelector:
                matchLabels:
                  app: postgres
              topologyKey: topology.kubernetes.io/zone
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  # Must be on storage-optimized nodes
                  - key: workload-type
                    operator: In
                    values:
                      - storage-optimized
                  # Must be in production node group
                  - key: node-group
                    operator: In
                    values:
                      - production-db
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: postgres
      containers:
        - name: postgres
          image: postgres:15
          resources:
            requests:
              cpu: 2000m
              memory: 8Gi
            limits:
              cpu: 8000m
              memory: 32Gi
```

### 8.2 Topology Spread Constraints

```yaml
# topology-spread.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-service
  namespace: production
spec:
  replicas: 9
  selector:
    matchLabels:
      app: api-service
  template:
    metadata:
      labels:
        app: api-service
    spec:
      topologySpreadConstraints:
        # Spread evenly across zones
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: api-service
          minDomains: 3   # Expect 3 zones
        # Spread evenly across nodes within zones
        - maxSkew: 2
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: api-service
      containers:
        - name: api
          image: myregistry/api-service:latest
          resources:
            requests:
              cpu: 200m
              memory: 256Mi
            limits:
              cpu: 1000m
              memory: 1Gi
```

## 9. Scale-to-Zero ด้วย KEDA

### 9.1 ScaledObject สำหรับ Scale-to-Zero

```yaml
# keda-scale-to-zero.yaml
# Batch processor ที่ scale to zero เมื่อไม่มีงาน
apiVersion: apps/v1
kind: Deployment
metadata:
  name: report-processor
  namespace: production
spec:
  replicas: 0   # เริ่มต้น 0 replicas
  selector:
    matchLabels:
      app: report-processor
  template:
    metadata:
      labels:
        app: report-processor
    spec:
      containers:
        - name: processor
          image: myregistry/report-processor:latest
          env:
            - name: SQS_QUEUE_URL
              valueFrom:
                secretKeyRef:
                  name: aws-secret
                  key: sqs-url
          resources:
            requests:
              cpu: 500m
              memory: 512Mi
            limits:
              cpu: 2000m
              memory: 2Gi
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: report-processor-scaler
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: report-processor
  pollingInterval: 30
  cooldownPeriod: 120
  idleReplicaCount: 0     # Scale to zero
  minReplicaCount: 0      # Allow zero replicas
  maxReplicaCount: 20
  triggers:
    - type: aws-sqs-queue
      metadata:
        queueURL: "https://sqs.ap-southeast-1.amazonaws.com/123456789/reports"
        queueLength: "5"    # 1 pod per 5 messages
        awsRegion: "ap-southeast-1"
        scaleOnInFlight: "true"
        activationQueueLength: "1"  # Wake up when 1 message arrives
      authenticationRef:
        name: aws-keda-auth
---
# Kafka scale-to-zero for event-driven ML
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: ml-training-scaler
  namespace: ml-production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ml-trainer
  pollingInterval: 60
  cooldownPeriod: 300    # 5 minutes cooldown
  idleReplicaCount: 0
  minReplicaCount: 0
  maxReplicaCount: 5
  advanced:
    horizontalPodAutoscalerConfig:
      behavior:
        scaleDown:
          stabilizationWindowSeconds: 300   # Wait 5 min before scale down
          policies:
            - type: Pods
              value: 1
              periodSeconds: 300
  triggers:
    - type: kafka
      metadata:
        bootstrapServers: "kafka.kafka:9092"
        consumerGroup: "ml-training-group"
        topic: "training-jobs"
        lagThreshold: "1"
        activationLagThreshold: "1"
---
# HTTP scale-to-zero for dev environments
apiVersion: http.keda.sh/v1alpha1
kind: HTTPScaledObject
metadata:
  name: dev-api-scaler
  namespace: development
spec:
  host: "dev-api.example.com"
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: dev-api
    service: dev-api
    port: 80
  replicas:
    min: 0        # Scale to zero when no traffic
    max: 5
  scaledownPeriod: 60   # Scale down after 60s of no traffic
  targetPendingRequests: 10
```

### 9.2 Knative Serving สำหรับ Scale-to-Zero

```yaml
# knative-service.yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: ml-inference-api
  namespace: production
  annotations:
    autoscaling.knative.dev/class: kpa.autoscaling.knative.dev
    autoscaling.knative.dev/metric: rps
    autoscaling.knative.dev/target: "100"
    autoscaling.knative.dev/scale-to-zero-pod-retention-period: "1m"
    autoscaling.knative.dev/initial-scale: "1"
    autoscaling.knative.dev/min-scale: "0"
    autoscaling.knative.dev/max-scale: "50"
    autoscaling.knative.dev/scale-down-delay: "0s"
    autoscaling.knative.dev/window: "60s"
    autoscaling.knative.dev/panic-window-percentage: "10.0"
    autoscaling.knative.dev/panic-threshold-percentage: "200.0"
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/target-utilization-percentage: "70"
    spec:
      containerConcurrency: 100
      timeoutSeconds: 300
      containers:
        - name: inference
          image: myregistry/ml-inference:latest
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: 1000m
              memory: 2Gi
            limits:
              cpu: 4000m
              memory: 8Gi
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 30
```

## 10. Autoscaling Monitoring Dashboard

### 10.1 Prometheus Rules สำหรับ Autoscaling

```yaml
# prometheus-autoscaling-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: autoscaling-alerts
  namespace: monitoring
spec:
  groups:
    - name: autoscaling.rules
      interval: 30s
      rules:
        # HPA at max replicas
        - alert: HPAAtMaxReplicas
          expr: |
            kube_horizontalpodautoscaler_status_current_replicas
            >= kube_horizontalpodautoscaler_spec_max_replicas
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "HPA {{ $labels.namespace }}/{{ $labels.horizontalpodautoscaler }} at max replicas"
            description: "Consider increasing maxReplicas or optimizing the service"

        # HPA unable to scale
        - alert: HPAScalingLimited
          expr: |
            kube_horizontalpodautoscaler_status_condition{condition="ScalingLimited",status="true"} == 1
          for: 15m
          labels:
            severity: warning
          annotations:
            summary: "HPA scaling is limited for {{ $labels.namespace }}/{{ $labels.horizontalpodautoscaler }}"

        # KEDA ScaledObject not active
        - alert: KEDAScaledObjectNotActive
          expr: |
            keda_scaledobject_paused == 1
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "KEDA ScaledObject {{ $labels.namespace }}/{{ $labels.scaledObject }} is paused"

        # Pod pending due to insufficient resources
        - alert: PodPendingDueToResources
          expr: |
            kube_pod_status_phase{phase="Pending"} * on(pod, namespace)
            kube_pod_status_unschedulable == 1
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} pending due to insufficient resources"

        # Node memory pressure
        - alert: NodeMemoryPressure
          expr: |
            kube_node_status_condition{condition="MemoryPressure",status="true"} == 1
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Node {{ $labels.node }} has memory pressure"

        # Cluster autoscaler unable to scale
        - alert: ClusterAutoscalerUnableToScale
          expr: |
            cluster_autoscaler_unschedulable_pods_count > 0
          for: 15m
          labels:
            severity: critical
          annotations:
            summary: "{{ $value }} pods cannot be scheduled"
            description: "Cluster Autoscaler cannot provision nodes. Check node group limits and AWS quotas"

        # Resource quota approaching limit
        - alert: ResourceQuotaApproachingLimit
          expr: |
            (kube_resourcequota{type="used"} / kube_resourcequota{type="hard"}) > 0.85
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Resource quota {{ $labels.resource }} in {{ $labels.namespace }} above 85%"
```

### 10.2 TypeScript Autoscaling Report Generator

```typescript
// autoscaling-report.ts
import * as k8s from "@kubernetes/client-node";
import {
  AutoscalingV2Api,
  CoreV1Api,
  AppsV1Api,
  CustomObjectsApi,
} from "@kubernetes/client-node";

interface AutoscalingReport {
  timestamp: Date;
  namespaces: NamespaceAutoscalingReport[];
  clusterSummary: ClusterSummary;
}

interface NamespaceAutoscalingReport {
  namespace: string;
  hpaCount: number;
  totalReplicas: number;
  scalingEvents: number;
  atMaxReplicas: number;
  kedaScaledObjects: KEDAObjectStatus[];
  recommendations: string[];
}

interface ClusterSummary {
  totalNodes: number;
  totalPods: number;
  pendingPods: number;
  cpuUtilization: number;
  memoryUtilization: number;
  autoscalingHealth: "healthy" | "degraded" | "critical";
}

interface KEDAObjectStatus {
  name: string;
  active: boolean;
  currentReplicas: number;
  desiredReplicas: number;
  triggerType: string;
}

class AutoscalingReporter {
  private autoscalingApi: AutoscalingV2Api;
  private coreApi: CoreV1Api;
  private appsApi: AppsV1Api;
  private customApi: CustomObjectsApi;

  constructor() {
    const kc = new k8s.KubeConfig();
    kc.loadFromDefault();
    this.autoscalingApi = kc.makeApiClient(AutoscalingV2Api);
    this.coreApi = kc.makeApiClient(CoreV1Api);
    this.appsApi = kc.makeApiClient(AppsV1Api);
    this.customApi = kc.makeApiClient(CustomObjectsApi);
  }

  async generateReport(namespaces: string[]): Promise<AutoscalingReport> {
    const [nodesResult, allPodsResult] = await Promise.all([
      this.coreApi.listNode(),
      this.coreApi.listPodForAllNamespaces(),
    ]);

    const nodes = nodesResult.body.items;
    const allPods = allPodsResult.body.items;
    const pendingPods = allPods.filter(
      (p) => p.status?.phase === "Pending"
    ).length;

    // Calculate cluster resource utilization
    let totalCpuCapacity = 0;
    let totalMemCapacity = 0;
    let totalCpuAllocated = 0;
    let totalMemAllocated = 0;

    for (const node of nodes) {
      const cpu = node.status?.capacity?.cpu || "0";
      const mem = node.status?.capacity?.memory || "0";
      totalCpuCapacity += this.parseCPU(cpu);
      totalMemCapacity += this.parseMem(mem);
    }

    for (const pod of allPods.filter((p) => p.status?.phase === "Running")) {
      for (const container of pod.spec?.containers || []) {
        totalCpuAllocated += this.parseCPU(
          container.resources?.requests?.cpu || "0"
        );
        totalMemAllocated += this.parseMem(
          container.resources?.requests?.memory || "0"
        );
      }
    }

    const namespaceReports = await Promise.all(
      namespaces.map((ns) => this.generateNamespaceReport(ns))
    );

    const cpuUtilization =
      totalCpuCapacity > 0
        ? (totalCpuAllocated / totalCpuCapacity) * 100
        : 0;
    const memUtilization =
      totalMemCapacity > 0
        ? (totalMemAllocated / totalMemCapacity) * 100
        : 0;

    let autoscalingHealth: "healthy" | "degraded" | "critical" = "healthy";
    if (pendingPods > 10 || cpuUtilization > 90 || memUtilization > 90) {
      autoscalingHealth = "critical";
    } else if (pendingPods > 3 || cpuUtilization > 75 || memUtilization > 75) {
      autoscalingHealth = "degraded";
    }

    return {
      timestamp: new Date(),
      namespaces: namespaceReports,
      clusterSummary: {
        totalNodes: nodes.length,
        totalPods: allPods.length,
        pendingPods,
        cpuUtilization,
        memoryUtilization: memUtilization,
        autoscalingHealth,
      },
    };
  }

  private async generateNamespaceReport(
    namespace: string
  ): Promise<NamespaceAutoscalingReport> {
    const [hpaResult, kedaResult] = await Promise.allSettled([
      this.autoscalingApi.listNamespacedHorizontalPodAutoscaler(namespace),
      this.customApi.listNamespacedCustomObject(
        "keda.sh",
        "v1alpha1",
        namespace,
        "scaledobjects"
      ),
    ]);

    const hpas =
      hpaResult.status === "fulfilled" ? hpaResult.value.body.items : [];

    let totalReplicas = 0;
    let atMaxReplicas = 0;
    const recommendations: string[] = [];

    for (const hpa of hpas) {
      const current = hpa.status?.currentReplicas || 0;
      const max = hpa.spec?.maxReplicas || 0;
      totalReplicas += current;

      if (current >= max) {
        atMaxReplicas++;
        recommendations.push(
          `${hpa.metadata?.name} is at max replicas (${max}). Consider increasing maxReplicas.`
        );
      }
    }

    const kedaObjects: KEDAObjectStatus[] = [];
    if (kedaResult.status === "fulfilled") {
      const body = kedaResult.value.body as { items?: Array<Record<string, unknown>> };
      for (const so of body.items || []) {
        const metadata = so.metadata as Record<string, unknown>;
        const status = (so.status || {}) as Record<string, unknown>;
        const spec = (so.spec || {}) as Record<string, unknown>;
        const triggers = ((spec.triggers as Array<Record<string, unknown>>) || []);

        kedaObjects.push({
          name: metadata.name as string,
          active: (status.active as boolean) || false,
          currentReplicas: (status.currentReplicas as number) || 0,
          desiredReplicas: (status.desiredReplicas as number) || 0,
          triggerType: triggers[0]?.type as string || "unknown",
        });
      }
    }

    return {
      namespace,
      hpaCount: hpas.length,
      totalReplicas,
      scalingEvents: 0, // Would need event API
      atMaxReplicas,
      kedaScaledObjects: kedaObjects,
      recommendations,
    };
  }

  private parseCPU(value: string): number {
    if (value.endsWith("m")) return parseInt(value) / 1000;
    return parseFloat(value) || 0;
  }

  private parseMem(value: string): number {
    if (value.endsWith("Ki")) return parseInt(value) * 1024;
    if (value.endsWith("Mi")) return parseInt(value) * 1024 * 1024;
    if (value.endsWith("Gi")) return parseInt(value) * 1024 * 1024 * 1024;
    return parseInt(value) || 0;
  }

  printReport(report: AutoscalingReport): void {
    console.log(`\n=== Autoscaling Report - ${report.timestamp.toISOString()} ===\n`);

    const cs = report.clusterSummary;
    console.log(`Cluster Summary:`);
    console.log(`  Nodes: ${cs.totalNodes}`);
    console.log(`  Total Pods: ${cs.totalPods} (${cs.pendingPods} pending)`);
    console.log(`  CPU Utilization: ${cs.cpuUtilization.toFixed(1)}%`);
    console.log(`  Memory Utilization: ${cs.memoryUtilization.toFixed(1)}%`);
    console.log(`  Health: ${cs.autoscalingHealth.toUpperCase()}`);

    for (const ns of report.namespaces) {
      console.log(`\nNamespace: ${ns.namespace}`);
      console.log(`  HPAs: ${ns.hpaCount} (${ns.atMaxReplicas} at max)`);
      console.log(`  Total Replicas: ${ns.totalReplicas}`);
      console.log(`  KEDA ScaledObjects: ${ns.kedaScaledObjects.length}`);

      const activeKEDA = ns.kedaScaledObjects.filter((k) => k.active).length;
      const zeroKEDA = ns.kedaScaledObjects.filter(
        (k) => k.currentReplicas === 0
      ).length;
      if (ns.kedaScaledObjects.length > 0) {
        console.log(
          `    Active: ${activeKEDA}, Scaled-to-zero: ${zeroKEDA}`
        );
      }

      if (ns.recommendations.length > 0) {
        console.log(`  Recommendations:`);
        ns.recommendations.forEach((r) => console.log(`    - ${r}`));
      }
    }
  }
}

// Run report
async function main(): Promise<void> {
  const reporter = new AutoscalingReporter();
  const report = await reporter.generateReport([
    "production",
    "staging",
    "ml-production",
  ]);
  reporter.printReport(report);
}

main().catch(console.error);
```

## สรุป

| Component | ประเภท | Use Case | Scaling Trigger | Scale-to-Zero |
|-----------|--------|----------|-----------------|---------------|
| **HPA** | Pod | CPU/Memory-bound workloads | CPU%, Memory%, Custom metrics | ไม่รองรับ (min=1) |
| **KEDA Kafka** | Pod | Event stream processing | Message lag | รองรับ |
| **KEDA RabbitMQ** | Pod | Queue-based workers | Queue depth | รองรับ |
| **KEDA SQS** | Pod | AWS-based batch jobs | Queue length | รองรับ |
| **KEDA HTTP** | Pod | HTTP services dev/staging | Request rate | รองรับ |
| **KEDA Prometheus** | Pod | Custom metric scaling | Any Prometheus query | รองรับ |
| **VPA** | Pod (resources) | Right-sizing containers | Historical usage | ไม่เกี่ยวข้อง |
| **Cluster Autoscaler** | Node | Unschedulable pods | Pending pods | ไม่รองรับ |
| **Knative** | Pod | Serverless workloads | RPS, Concurrency | รองรับ |
| **ScaledJob** | Job | Batch processing | Queue/metric | N/A (jobs) |

### Key Takeaways

1. **ใช้ HPA สำหรับ** stateless web services ที่ scale ตาม CPU/memory หรือ custom HTTP metrics

2. **ใช้ KEDA สำหรับ** event-driven workloads เช่น queue consumers, batch processors ที่ต้องการ scale-to-zero เพื่อประหยัด cost

3. **ใช้ VPA สำหรับ** right-sizing — เปิด mode `Off` ก่อนเพื่อดู recommendations แล้วค่อย apply โดยเฉพาะ stateful workloads

4. **ใช้ HPA + KEDA ร่วมกันไม่ได้** บน workload เดียวกัน (conflict) — เลือกอย่างใดอย่างหนึ่ง

5. **PDB สำคัญมาก** สำหรับ zero-downtime ต้องกำหนดก่อน production deployment ทุกครั้ง

6. **Priority Classes** ช่วยให้ critical workloads ได้ resources ก่อนเมื่อ cluster ตึง

7. **Resource Quotas** ป้องกัน namespace หนึ่งใช้ resources ทั้งหมดใน cluster

8. **Pod Anti-Affinity** กับ `requiredDuringScheduling` สำหรับ HA databases/caches จะ prevent data loss เมื่อ node หรือ zone ล้มเหลว

9. **Cluster Autoscaler** ควรเปิด `balance-similar-node-groups=true` เพื่อให้ spread evenly และ ลด fragmentation

10. **Topology Spread Constraints** แทน podAntiAffinity ในกรณีที่ต้องการ even distribution โดยไม่ hard-block scheduling

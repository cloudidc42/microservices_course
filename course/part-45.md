# Part 45: Kubernetes Autoscaling

## บทนำ

Kubernetes มีกลไก Autoscaling หลายระดับที่ทำงานร่วมกันเพื่อรองรับ Load ที่เปลี่ยนแปลง ทั้ง HPA (Horizontal Pod Autoscaler), VPA (Vertical Pod Autoscaler), Cluster Autoscaler และ KEDA (Kubernetes Event-Driven Autoscaling) สำหรับ Custom Metrics ในบทนี้จะครอบคลุมการตั้งค่าที่ใช้ใน Production จริง พร้อม Resource Management และ Scheduling Policies

### สิ่งที่จะได้เรียนรู้

- HPA ด้วย CPU/Memory และ Custom Metrics
- KEDA สำหรับ Event-Driven Scaling
- VPA (Vertical Pod Autoscaler)
- Cluster Autoscaler
- Pod Disruption Budgets (PDB)
- Priority Classes
- Resource Quotas และ LimitRanges
- Node Affinity/Anti-affinity และ Pod Topology Spread

---

## 1. ภาพรวมของ Autoscaling

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Kubernetes Cluster                            │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                       Node Pool                              │   │
│  │  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐       │   │
│  │  │   Node 1    │   │   Node 2    │   │   Node 3    │       │   │
│  │  │  [Pod][Pod] │   │  [Pod][Pod] │   │  [Pod][Pod] │       │   │
│  │  └─────────────┘   └─────────────┘   └─────────────┘       │   │
│  │                                              ▲               │   │
│  │                             Cluster Autoscaler               │   │
│  └─────────────────────────────────────────────────────────────┘   │
│          ▲                                      ▲                   │
│    HPA/KEDA                                   VPA                   │
│  (scale pods)                          (resize pods)                │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. Horizontal Pod Autoscaler (HPA) — CPU และ Memory

```yaml
# hpa/order-service-basic.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
  namespace: microservices
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  
  minReplicas: 3
  maxReplicas: 50
  
  metrics:
    # CPU-based scaling
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60  # Scale up when avg CPU > 60%
    
    # Memory-based scaling
    - type: Resource
      resource:
        name: memory
        target:
          type: AverageValue
          averageValue: 400Mi  # Scale up when avg memory > 400Mi
  
  behavior:
    # Scale up behavior
    scaleUp:
      stabilizationWindowSeconds: 0  # React immediately
      policies:
        - type: Pods
          value: 4          # Add max 4 pods per 60s
          periodSeconds: 60
        - type: Percent
          value: 100        # Or double current replicas
          periodSeconds: 60
      selectPolicy: Max  # Use the most aggressive policy
    
    # Scale down behavior
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5 minutes before scaling down
      policies:
        - type: Pods
          value: 2          # Remove max 2 pods per 120s
          periodSeconds: 120
        - type: Percent
          value: 20         # Or remove max 20% per 120s
          periodSeconds: 120
      selectPolicy: Min  # Use the most conservative policy
```

---

## 3. HPA ด้วย Custom Metrics (Prometheus Adapter)

```yaml
# prometheus-adapter/custom-metrics-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: adapter-config
  namespace: monitoring
data:
  config.yaml: |
    rules:
      # HTTP Requests per second
      - seriesQuery: 'http_requests_total{namespace!="",pod!=""}'
        resources:
          overrides:
            namespace: {resource: "namespace"}
            pod: {resource: "pod"}
        name:
          matches: "^(.*)_total$"
          as: "${1}_per_second"
        metricsQuery: 'sum(rate(<<.Series>>{<<.LabelMatchers>>}[2m])) by (<<.GroupBy>>)'
      
      # Queue depth from RabbitMQ
      - seriesQuery: 'rabbitmq_queue_messages{namespace!="",queue!=""}'
        resources:
          overrides:
            namespace: {resource: "namespace"}
        name:
          matches: "^rabbitmq_queue_messages$"
          as: "rabbitmq_queue_depth"
        metricsQuery: 'max(<<.Series>>{<<.LabelMatchers>>}) by (queue)'
      
      # Active WebSocket connections
      - seriesQuery: 'websocket_connections_active{namespace!="",pod!=""}'
        resources:
          overrides:
            namespace: {resource: "namespace"}
            pod: {resource: "pod"}
        name:
          as: "websocket_connections"
        metricsQuery: 'sum(<<.Series>>{<<.LabelMatchers>>}) by (<<.GroupBy>>)'
```

```yaml
# hpa/order-service-custom-metrics.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa-custom
  namespace: microservices
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  
  minReplicas: 3
  maxReplicas: 100
  
  metrics:
    # CPU
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    
    # Custom: HTTP requests per second per pod
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "100"  # 100 RPS per pod
    
    # Custom: Queue depth (Object metric)
    - type: Object
      object:
        metric:
          name: rabbitmq_queue_depth
        describedObject:
          apiVersion: v1
          kind: Service
          name: rabbitmq
        target:
          type: Value
          value: "1000"  # Scale up when queue > 1000 messages
```

---

## 4. KEDA (Kubernetes Event-Driven Autoscaler)

```yaml
# keda/install.yaml
# ติดตั้ง KEDA
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
helm install keda kedacore/keda \
  --namespace keda \
  --create-namespace \
  --version 2.13.0 \
  --set resources.operator.requests.cpu=100m \
  --set resources.operator.requests.memory=100Mi
```

```yaml
# keda/order-worker-rabbitmq.yaml
# Scale Worker ตาม RabbitMQ Queue Depth
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-worker-scaler
  namespace: microservices
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-worker
  
  pollingInterval: 15      # Check every 15 seconds
  cooldownPeriod: 30       # Wait 30s after scaling down
  idleReplicaCount: 0      # Scale to 0 when queue is empty
  minReplicaCount: 0
  maxReplicaCount: 50
  
  advanced:
    restoreToOriginalReplicaCount: true
    scalingModifiers:
      formula: "queue_depth / 10"  # 1 worker per 10 messages
      target: "1"
    horizontalPodAutoscalerConfig:
      behavior:
        scaleDown:
          stabilizationWindowSeconds: 60
  
  triggers:
    - type: rabbitmq
      metadata:
        protocol: amqp
        queueName: order-processing
        mode: QueueLength  # Or MessageRate
        value: "10"        # Scale up when queue > 10 messages
        activationValue: "1"  # Activate (from 0) when queue > 1
      authenticationRef:
        name: rabbitmq-trigger-auth
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: rabbitmq-trigger-auth
  namespace: microservices
spec:
  secretTargetRef:
    - parameter: host
      name: rabbitmq-secret
      key: amqp-url
```

```yaml
# keda/notification-worker-redis.yaml
# Scale Worker ตาม Redis Stream Lag
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: notification-worker-scaler
  namespace: microservices
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: notification-worker
  
  minReplicaCount: 1
  maxReplicaCount: 20
  
  triggers:
    - type: redis-streams
      metadata:
        address: redis-master.microservices:6379
        stream: notifications
        consumerGroup: notification-workers
        pendingEntriesCount: "100"  # Scale up when lag > 100
        activationLagCount: "1"
      authenticationRef:
        name: redis-trigger-auth
```

```yaml
# keda/api-gateway-cron.yaml
# Pre-scale API Gateway ก่อนเวลา Peak Traffic
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: api-gateway-schedule-scaler
  namespace: microservices
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-gateway
  
  minReplicaCount: 3
  maxReplicaCount: 30
  
  triggers:
    # Scale up to 15 replicas during business hours (7am-9pm ICT)
    - type: cron
      metadata:
        timezone: "Asia/Bangkok"
        start: "0 7 * * 1-5"   # 7am weekdays
        end: "0 21 * * 1-5"    # 9pm weekdays
        desiredReplicas: "15"
    
    # Weekend peak: 9am-8pm
    - type: cron
      metadata:
        timezone: "Asia/Bangkok"
        start: "0 9 * * 6-7"   # 9am weekends
        end: "0 20 * * 6-7"    # 8pm weekends
        desiredReplicas: "10"
    
    # Also scale on CPU
    - type: cpu
      metricType: Utilization
      metadata:
        value: "70"
```

```yaml
# keda/report-generator-http.yaml
# Scale ตาม HTTP Prometheus metric
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: report-generator-scaler
  namespace: microservices
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: report-generator
  
  minReplicaCount: 0
  maxReplicaCount: 10
  
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://prometheus.monitoring:9090
        metricName: pending_reports_total
        threshold: "5"
        activationThreshold: "1"
        query: sum(pending_reports_total{namespace="microservices"})
```

---

## 5. Vertical Pod Autoscaler (VPA)

```yaml
# vpa/install.yaml
# ติดตั้ง VPA
kubectl apply -f https://github.com/kubernetes/autoscaler/releases/latest/download/vertical-pod-autoscaler.yaml
```

```yaml
# vpa/order-service-vpa.yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: order-service-vpa
  namespace: microservices
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  
  updatePolicy:
    updateMode: "Auto"  # Auto | Recreate | Initial | Off
    minReplicas: 2      # Minimum replicas during update
  
  resourcePolicy:
    containerPolicies:
      - containerName: order-service
        minAllowed:
          cpu: 100m
          memory: 128Mi
        maxAllowed:
          cpu: 4000m
          memory: 4Gi
        controlledResources:
          - cpu
          - memory
        controlledValues: RequestsAndLimits
```

```yaml
# vpa/postgres-vpa.yaml
# VPA แบบ Off (แค่ Recommend ไม่ Apply)
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: postgres-vpa
  namespace: microservices
spec:
  targetRef:
    apiVersion: apps/v1
    kind: StatefulSet
    name: postgres
  
  updatePolicy:
    updateMode: "Off"  # แค่ Recommend เท่านั้น
  
  resourcePolicy:
    containerPolicies:
      - containerName: postgres
        minAllowed:
          cpu: 250m
          memory: 512Mi
        maxAllowed:
          cpu: 8000m
          memory: 32Gi
```

```bash
# ดู VPA Recommendations
kubectl describe vpa order-service-vpa -n microservices
kubectl get vpa -n microservices -o yaml
```

---

## 6. Cluster Autoscaler

```yaml
# cluster-autoscaler/deployment.yaml
# สำหรับ AWS EKS
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
    spec:
      priorityClassName: system-cluster-critical
      serviceAccountName: cluster-autoscaler
      securityContext:
        runAsNonRoot: true
        runAsUser: 65534
        fsGroup: 65534
      containers:
        - image: registry.k8s.io/autoscaling/cluster-autoscaler:v1.29.0
          name: cluster-autoscaler
          resources:
            limits:
              cpu: 100m
              memory: 600Mi
            requests:
              cpu: 100m
              memory: 600Mi
          command:
            - ./cluster-autoscaler
            - --v=4
            - --stderrthreshold=info
            - --cloud-provider=aws
            - --skip-nodes-with-local-storage=false
            - --expander=least-waste
            - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/my-cluster
            # Scale down settings
            - --scale-down-enabled=true
            - --scale-down-unneeded-time=10m
            - --scale-down-delay-after-add=10m
            - --scale-down-utilization-threshold=0.5
            - --scale-down-non-empty-candidates-count=30
            # Scale up settings
            - --max-node-provision-time=15m
            - --scan-interval=10s
            # Node group sizes
            - --max-total-unready-percentage=33
            - --ok-total-unready-count=3
          volumeMounts:
            - name: ssl-certs
              mountPath: /etc/ssl/certs/ca-certificates.crt
              readOnly: true
          env:
            - name: AWS_REGION
              value: ap-southeast-1
      volumes:
        - name: ssl-certs
          hostPath:
            path: /etc/ssl/certs/ca-bundle.crt
```

```yaml
# cluster-autoscaler/node-groups.yaml
# AWS Node Groups สำหรับ EKS
# (จัดการผ่าน Terraform/eksctl)

# eksctl create nodegroup
# eksctl-nodegroup.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: production-cluster
  region: ap-southeast-1

managedNodeGroups:
  # General purpose nodes
  - name: general-purpose
    instanceType: m5.large
    minSize: 3
    maxSize: 20
    desiredCapacity: 5
    labels:
      node-type: general
    tags:
      k8s.io/cluster-autoscaler/enabled: "true"
      k8s.io/cluster-autoscaler/production-cluster: "owned"
    iam:
      withAddonPolicies:
        autoScaler: true
  
  # Memory optimized nodes for Databases/Cache
  - name: memory-optimized
    instanceType: r5.xlarge
    minSize: 2
    maxSize: 10
    labels:
      node-type: memory-optimized
    taints:
      - key: node-type
        value: memory-optimized
        effect: NoSchedule
    tags:
      k8s.io/cluster-autoscaler/enabled: "true"
      k8s.io/cluster-autoscaler/production-cluster: "owned"
  
  # GPU nodes for ML workloads
  - name: gpu-nodes
    instanceType: g4dn.xlarge
    minSize: 0
    maxSize: 5
    labels:
      node-type: gpu
      nvidia.com/gpu: "true"
    taints:
      - key: nvidia.com/gpu
        value: "true"
        effect: NoSchedule
    tags:
      k8s.io/cluster-autoscaler/enabled: "true"
      k8s.io/cluster-autoscaler/production-cluster: "owned"
```

---

## 7. Pod Disruption Budgets (PDB)

```yaml
# pdb/critical-services.yaml
# ป้องกัน Downtime ระหว่าง Maintenance

# Order Service: ต้องมีอย่างน้อย 2 pods ทำงาน
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: order-service-pdb
  namespace: microservices
spec:
  maxUnavailable: 1  # ยอมให้ unavailable ได้ไม่เกิน 1 pod
  selector:
    matchLabels:
      app: order-service
---
# Payment Service: ต้องมีอย่างน้อย 90% ทำงาน
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: payment-service-pdb
  namespace: microservices
spec:
  minAvailable: "90%"
  selector:
    matchLabels:
      app: payment-service
---
# API Gateway: ต้องมีอย่างน้อย 3 pods ทำงาน
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-gateway-pdb
  namespace: microservices
spec:
  minAvailable: 3
  selector:
    matchLabels:
      app: api-gateway
---
# Postgres: ไม่ให้ Disrupt เลย (StatefulSet)
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: postgres-pdb
  namespace: microservices
spec:
  maxUnavailable: 0  # ห้าม disrupt เด็ดขาด
  selector:
    matchLabels:
      app: postgres
```

---

## 8. Priority Classes

```yaml
# priority-classes/priority-classes.yaml

# System critical: istiod, CoreDNS
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: system-critical
value: 1000000
globalDefault: false
description: "Used for system-critical components like Istio, CoreDNS"
preemptionPolicy: PreemptLowerPriority
---
# Business critical: Payment, Order service
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: business-critical
value: 100000
globalDefault: false
description: "Revenue-generating services: payment, order"
preemptionPolicy: PreemptLowerPriority
---
# High priority: API Gateway, Auth service
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 50000
globalDefault: false
description: "Core user-facing services"
preemptionPolicy: PreemptLowerPriority
---
# Default: Most microservices
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: default-priority
value: 1000
globalDefault: true
description: "Default priority for most workloads"
preemptionPolicy: PreemptLowerPriority
---
# Low priority: Batch jobs, background workers
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: 100
globalDefault: false
description: "Non-critical batch jobs and workers"
preemptionPolicy: Never  # ไม่ Preempt ผู้อื่น
```

```yaml
# deployments/payment-service.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
  namespace: microservices
spec:
  replicas: 5
  selector:
    matchLabels:
      app: payment-service
  template:
    metadata:
      labels:
        app: payment-service
    spec:
      priorityClassName: business-critical  # กำหนด Priority
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

---

## 9. Resource Quotas

```yaml
# resource-quotas/production-namespace.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: microservices
spec:
  hard:
    # Pod limits
    pods: "200"
    
    # CPU limits
    requests.cpu: "80"        # Total 80 CPU cores
    limits.cpu: "160"         # Max 160 CPU cores
    
    # Memory limits
    requests.memory: 160Gi   # Total 160 GB RAM
    limits.memory: 320Gi     # Max 320 GB RAM
    
    # Storage
    requests.storage: "2Ti"
    persistentvolumeclaims: "50"
    
    # Services
    services: "100"
    services.loadbalancers: "5"
    services.nodeports: "10"
    
    # Config/Secrets
    configmaps: "100"
    secrets: "200"
```

```yaml
# resource-quotas/staging-namespace.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: staging-quota
  namespace: staging
spec:
  hard:
    pods: "50"
    requests.cpu: "20"
    limits.cpu: "40"
    requests.memory: 40Gi
    limits.memory: 80Gi
    requests.storage: "500Gi"
    persistentvolumeclaims: "20"
    services: "30"
    services.loadbalancers: "2"
```

---

## 10. LimitRanges

```yaml
# limit-ranges/microservices-limits.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: microservices-limits
  namespace: microservices
spec:
  limits:
    # Container defaults and limits
    - type: Container
      default:
        cpu: 200m
        memory: 256Mi
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      max:
        cpu: 8000m        # Max 8 CPU cores per container
        memory: 16Gi      # Max 16 GB per container
      min:
        cpu: 10m
        memory: 16Mi
      maxLimitRequestRatio:
        cpu: 10           # Limit cannot exceed 10x request
        memory: 4         # Limit cannot exceed 4x request
    
    # Pod limits
    - type: Pod
      max:
        cpu: 16000m       # Max 16 CPU cores per pod
        memory: 32Gi
      min:
        cpu: 10m
        memory: 16Mi
    
    # PVC size limits
    - type: PersistentVolumeClaim
      max:
        storage: 1Ti
      min:
        storage: 1Gi
```

---

## 11. Node Affinity และ Anti-affinity

```yaml
# affinity/order-service-spread.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: microservices
spec:
  replicas: 9
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      # Prefer to schedule on general nodes
      affinity:
        nodeAffinity:
          # Required: ต้องอยู่บน Linux x86_64 เท่านั้น
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: kubernetes.io/os
                    operator: In
                    values:
                      - linux
                  - key: kubernetes.io/arch
                    operator: In
                    values:
                      - amd64
          # Preferred: prefer general-purpose nodes
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              preference:
                matchExpressions:
                  - key: node-type
                    operator: In
                    values:
                      - general
            - weight: 50
              preference:
                matchExpressions:
                  - key: topology.kubernetes.io/zone
                    operator: In
                    values:
                      - ap-southeast-1a
                      - ap-southeast-1b
        
        # Pod Anti-affinity: กระจาย Pods ไปต่างโหนด
        podAntiAffinity:
          # Required: ห้ามสองโหนดเดียวกัน
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - order-service
              topologyKey: kubernetes.io/hostname
              namespaces:
                - microservices
      
      # Tolerations: ยอมทำงานบน tainted nodes
      tolerations:
        - key: "node.kubernetes.io/memory-pressure"
          operator: "Exists"
          effect: "NoSchedule"
      
      containers:
        - name: order-service
          image: myregistry/order-service:1.0.0
```

---

## 12. Pod Topology Spread Constraints

```yaml
# topology-spread/api-gateway.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
  namespace: microservices
spec:
  replicas: 12
  selector:
    matchLabels:
      app: api-gateway
  template:
    metadata:
      labels:
        app: api-gateway
    spec:
      # Spread pods evenly across zones and nodes
      topologySpreadConstraints:
        # Spread across availability zones
        - maxSkew: 1            # Imbalance ไม่เกิน 1 pod
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule  # หรือ ScheduleAnyway
          labelSelector:
            matchLabels:
              app: api-gateway
          minDomains: 3         # ต้องมีอย่างน้อย 3 zones
          nodeAffinityPolicy: Honor
          nodeTaintsPolicy: Honor
        
        # Also spread across nodes within each zone
        - maxSkew: 2
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway  # Soft constraint
          labelSelector:
            matchLabels:
              app: api-gateway
      
      containers:
        - name: api-gateway
          image: myregistry/api-gateway:1.0.0
          resources:
            requests:
              cpu: 200m
              memory: 256Mi
            limits:
              cpu: 1000m
              memory: 1Gi
```

---

## 13. Deployment ที่ Production-ready ครบถ้วน

```yaml
# deployments/order-service-full.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: microservices
  annotations:
    deployment.kubernetes.io/revision: "1"
spec:
  replicas: 5
  
  # Rolling update strategy
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1       # ไม่เกิน 1 pod unavailable
      maxSurge: 2             # สร้าง extra ได้ 2 pods
  
  selector:
    matchLabels:
      app: order-service
      version: v1
  
  template:
    metadata:
      labels:
        app: order-service
        version: v1
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
    
    spec:
      priorityClassName: business-critical
      serviceAccountName: order-service-sa
      
      # Security context
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault
      
      # Graceful shutdown
      terminationGracePeriodSeconds: 60
      
      # Init container: wait for DB
      initContainers:
        - name: wait-for-db
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              until nc -z $DB_HOST $DB_PORT; do
                echo "Waiting for database..."
                sleep 2
              done
              echo "Database is ready!"
          env:
            - name: DB_HOST
              value: postgres
            - name: DB_PORT
              value: "5432"
          securityContext:
            runAsNonRoot: true
            runAsUser: 1000
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
      
      containers:
        - name: order-service
          image: myregistry/order-service:1.5.2
          imagePullPolicy: IfNotPresent
          
          ports:
            - containerPort: 3000
              name: http
              protocol: TCP
            - containerPort: 9090
              name: metrics
              protocol: TCP
          
          env:
            - name: NODE_ENV
              value: production
            - name: PORT
              value: "3000"
            - name: LOG_LEVEL
              value: "warn"
            - name: DB_URL
              valueFrom:
                secretKeyRef:
                  name: order-service-secrets
                  key: database-url
            - name: REDIS_URL
              valueFrom:
                secretKeyRef:
                  name: order-service-secrets
                  key: redis-url
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
          
          resources:
            requests:
              cpu: 200m
              memory: 256Mi
            limits:
              cpu: 1000m
              memory: 1Gi
          
          # Liveness probe: restart if stuck
          livenessProbe:
            httpGet:
              path: /health/live
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 15
            timeoutSeconds: 5
            failureThreshold: 3
            successThreshold: 1
          
          # Readiness probe: remove from service if not ready
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 3
            successThreshold: 1
          
          # Startup probe: slow-starting containers
          startupProbe:
            httpGet:
              path: /health/startup
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 30  # 30 * 5s = 150s max startup time
          
          # Graceful shutdown
          lifecycle:
            preStop:
              exec:
                command:
                  - sh
                  - -c
                  - sleep 5  # Wait for LB to remove endpoint
          
          # Security
          securityContext:
            runAsNonRoot: true
            runAsUser: 1000
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL
          
          # Volume mounts
          volumeMounts:
            - name: tmp
              mountPath: /tmp
            - name: config
              mountPath: /app/config
              readOnly: true
      
      volumes:
        - name: tmp
          emptyDir: {}
        - name: config
          configMap:
            name: order-service-config
      
      # Spread across zones
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: order-service
      
      # Anti-affinity
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: order-service
                topologyKey: kubernetes.io/hostname
```

---

## 14. Health Check Endpoints

```typescript
// src/health/health.controller.ts
import express, { Request, Response } from 'express';
import { Pool } from 'pg';
import { createClient } from 'redis';

export function createHealthRouter(pool: Pool, redis: any) {
  const router = express.Router();
  
  let isReady = false;
  let isStarted = false;

  // Mark as started after initialization
  setTimeout(() => {
    isStarted = true;
    isReady = true;
  }, 5000);

  // Startup probe: เช็คว่าแอปเริ่มต้นสำเร็จหรือยัง
  router.get('/health/startup', (req: Request, res: Response) => {
    if (isStarted) {
      res.json({ status: 'started' });
    } else {
      res.status(503).json({ status: 'starting' });
    }
  });

  // Liveness probe: เช็คว่าแอปยังทำงานอยู่
  router.get('/health/live', (req: Request, res: Response) => {
    // Simple check — just return 200 if process is running
    res.json({
      status: 'alive',
      timestamp: new Date().toISOString(),
      uptime: process.uptime(),
    });
  });

  // Readiness probe: เช็คว่าพร้อมรับ Traffic
  router.get('/health/ready', async (req: Request, res: Response) => {
    if (!isReady) {
      return res.status(503).json({ status: 'not_ready' });
    }

    const checks: Record<string, boolean> = {};

    // Check database
    try {
      await pool.query('SELECT 1');
      checks.database = true;
    } catch {
      checks.database = false;
    }

    // Check Redis
    try {
      await redis.ping();
      checks.redis = true;
    } catch {
      checks.redis = false;
    }

    const allHealthy = Object.values(checks).every(Boolean);

    if (allHealthy) {
      res.json({ status: 'ready', checks });
    } else {
      res.status(503).json({ status: 'not_ready', checks });
    }
  });

  // Full health check (for monitoring)
  router.get('/health', async (req: Request, res: Response) => {
    const checks: Record<string, any> = {};

    try {
      const start = Date.now();
      await pool.query('SELECT 1');
      checks.database = { status: 'ok', latency: Date.now() - start };
    } catch (error) {
      checks.database = { status: 'error', error: (error as Error).message };
    }

    try {
      const start = Date.now();
      await redis.ping();
      checks.redis = { status: 'ok', latency: Date.now() - start };
    } catch (error) {
      checks.redis = { status: 'error', error: (error as Error).message };
    }

    const allHealthy = Object.values(checks).every(
      (c: any) => c.status === 'ok'
    );

    res.status(allHealthy ? 200 : 503).json({
      status: allHealthy ? 'healthy' : 'unhealthy',
      timestamp: new Date().toISOString(),
      version: process.env.SERVICE_VERSION || '1.0.0',
      checks,
      uptime: process.uptime(),
      memory: process.memoryUsage(),
    });
  });

  return router;
}
```

---

## 15. Monitoring HPA และ KEDA

```yaml
# monitoring/hpa-dashboard.yaml
# Prometheus rules สำหรับ HPA monitoring
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: hpa-alerts
  namespace: monitoring
spec:
  groups:
    - name: hpa
      interval: 30s
      rules:
        # Alert when HPA hits max replicas
        - alert: HPAMaxReplicasReached
          expr: |
            kube_horizontalpodautoscaler_status_current_replicas
            >= kube_horizontalpodautoscaler_spec_max_replicas
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "HPA {{ $labels.namespace }}/{{ $labels.horizontalpodautoscaler }} has reached max replicas"
            description: "HPA is at maximum capacity. Consider increasing maxReplicas."
        
        # Alert when deployment is scaling very rapidly
        - alert: RapidScaling
          expr: |
            rate(kube_horizontalpodautoscaler_status_current_replicas[10m]) > 2
          for: 1m
          labels:
            severity: warning
          annotations:
            summary: "Rapid scaling detected for {{ $labels.horizontalpodautoscaler }}"
        
        # Alert when KEDA ScaledObject is not ready
        - alert: KEDAScaledObjectNotReady
          expr: |
            keda_scaler_active == 0
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "KEDA ScaledObject {{ $labels.scaledObject }} is not active"
```

```bash
# ตรวจสอบสถานะ HPA
kubectl get hpa -n microservices
kubectl describe hpa order-service-hpa -n microservices

# ดู HPA events
kubectl get events -n microservices --field-selector reason=SuccessfulRescale

# ตรวจสอบ KEDA
kubectl get scaledobject -n microservices
kubectl describe scaledobject order-worker-scaler -n microservices

# ดู custom metrics
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1" | jq .
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/microservices/pods/*/http_requests_per_second" | jq .

# ดู VPA recommendations
kubectl get vpa -n microservices
kubectl describe vpa order-service-vpa -n microservices | grep -A20 "Recommendation"
```

---

## 16. Load Testing เพื่อทดสอบ Autoscaling

```typescript
// load-test/src/k6-test.ts
// k6 Load Test Script
// รัน: k6 run load-test.js

export const options = {
  scenarios: {
    // Gradual ramp-up
    ramp_up: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '2m', target: 10 },   // Warm up
        { duration: '5m', target: 100 },  // Normal load
        { duration: '3m', target: 500 },  // Peak load
        { duration: '2m', target: 100 },  // Cool down
        { duration: '2m', target: 0 },    // Ramp down
      ],
    },
    
    // Spike test
    spike: {
      executor: 'ramping-vus',
      startTime: '5m',
      startVUs: 0,
      stages: [
        { duration: '30s', target: 1000 }, // Sudden spike
        { duration: '1m', target: 1000 },  // Sustain
        { duration: '30s', target: 0 },    // Drop
      ],
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<500'],  // 95th percentile < 500ms
    http_req_failed: ['rate<0.01'],    // Error rate < 1%
  },
};

export default function () {
  const response = http.post(
    'http://api.myapp.com/api/v1/orders',
    JSON.stringify({
      customerId: `customer-${Math.floor(Math.random() * 1000)}`,
      items: [
        { productId: 'prod-1', quantity: 1 },
      ],
    }),
    {
      headers: {
        'Content-Type': 'application/json',
        Authorization: `Bearer ${__ENV.TEST_TOKEN}`,
      },
    }
  );

  check(response, {
    'status is 201': (r) => r.status === 201,
    'response time < 500ms': (r) => r.timings.duration < 500,
  });

  sleep(1);
}
```

---

## สรุป

| หัวข้อ | Resource/Tool | เมื่อไหร่ใช้ |
|--------|--------------|------------|
| **HPA (CPU/Memory)** | `HorizontalPodAutoscaler` v2 | Scale ตาม CPU/Memory ของ Pod |
| **HPA (Custom Metrics)** | Prometheus Adapter + HPA | Scale ตาม Business Metrics (RPS, Queue) |
| **KEDA** | `ScaledObject` + Triggers | Event-driven scaling, Scale to Zero |
| **KEDA Cron** | cron trigger | Pre-scale ก่อนเวลา Peak Traffic |
| **VPA** | `VerticalPodAutoscaler` | ปรับ CPU/Memory Request อัตโนมัติ |
| **Cluster Autoscaler** | Deployment + Node Groups | เพิ่ม/ลด Node ตาม Pending Pods |
| **PDB** | `PodDisruptionBudget` | ป้องกัน Downtime ระหว่าง Maintenance |
| **Priority Classes** | `PriorityClass` | เรียง Pods สำคัญก่อนเมื่อ Resource ขาด |
| **Resource Quotas** | `ResourceQuota` | จำกัด Resource การใช้ต่อ Namespace |
| **LimitRanges** | `LimitRange` | กำหนด Default + Max Resources ต่อ Container |
| **Node Affinity** | `nodeAffinity` | กำหนดให้ Pod ทำงานบน Node ชนิดใด |
| **Pod Anti-affinity** | `podAntiAffinity` | กระจาย Pods ไปต่างโหนด/Zone |
| **Topology Spread** | `topologySpreadConstraints` | Distribute evenly ทุก Zone |

# Part 17: Kubernetes - Introduction & Core Concepts

## บทนำ

Kubernetes (K8s) คือ container orchestration platform ที่ทรงพลังที่สุด สำหรับการจัดการ Microservices ในระดับ production โดย automate การ deploy, scale, และ self-heal ของ containers

---

## 17.1 ทำไมต้องใช้ Kubernetes?

### ปัญหาของ Docker Compose ใน Production

```
Docker Compose:
- ทำงานบน single machine เท่านั้น
- ไม่มี auto-scaling
- ไม่มี self-healing (container ล้ม = ต้อง restart เอง)
- ไม่มี rolling deployment
- ไม่มี load balancing ขั้นสูง
- Secret management ยาก
```

### Kubernetes แก้ปัญหาเหล่านี้:

```
Kubernetes:
✓ Multi-node cluster (หลาย machines)
✓ Auto-scaling (HPA, VPA, Cluster Autoscaler)
✓ Self-healing (restart failed pods, replace unhealthy nodes)
✓ Rolling deployment (zero-downtime updates)
✓ Service discovery & Load balancing
✓ Secret & ConfigMap management
✓ Storage orchestration
✓ Declarative configuration (GitOps friendly)
```

---

## 17.2 Kubernetes Architecture

```
                    Control Plane (Master)
┌─────────────────────────────────────────────────────┐
│  API Server  │  etcd  │  Scheduler  │  Controller   │
│  (REST API)  │ (data) │  (places    │  Manager      │
│              │        │   pods)     │  (ensures     │
│              │        │             │   desired     │
│              │        │             │   state)      │
└─────────────────────────────────────────────────────┘
                         │
              ┌──────────┼──────────┐
              ↓          ↓          ↓
           Node 1      Node 2     Node 3
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │ kubelet  │ │ kubelet  │ │ kubelet  │
        │ kube-    │ │ kube-    │ │ kube-    │
        │ proxy    │ │ proxy    │ │ proxy    │
        │ pods:    │ │ pods:    │ │ pods:    │
        │ [p1][p2] │ │ [p3][p4] │ │ [p5][p6] │
        └──────────┘ └──────────┘ └──────────┘
```

### Core Components

| Component | บทบาท |
|-----------|--------|
| **Pod** | Unit เล็กที่สุด - กลุ่มของ containers ที่ share network |
| **Deployment** | จัดการ Pod replicas, rolling updates |
| **Service** | Load balancing, service discovery |
| **ConfigMap** | Non-sensitive configuration |
| **Secret** | Sensitive data (passwords, tokens) |
| **Ingress** | HTTP/HTTPS routing rules |
| **Namespace** | Virtual cluster isolation |
| **PersistentVolume** | Storage |
| **HPA** | Horizontal Pod Autoscaler |

---

## 17.3 Setup Local Kubernetes

### Option 1: minikube

```bash
# Install minikube
brew install minikube  # macOS
# or
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Start cluster
minikube start --cpus=4 --memory=8192 --driver=docker

# Enable addons
minikube addons enable ingress
minikube addons enable metrics-server
minikube addons enable dashboard

# Get cluster IP
minikube ip

# Access dashboard
minikube dashboard
```

### Option 2: kind (Kubernetes in Docker)

```bash
# Install kind
go install sigs.k8s.io/kind@v0.20.0

# Create multi-node cluster
cat > kind-config.yaml <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
EOF

kind create cluster --name microservices --config kind-config.yaml

# Load local image into kind
kind load docker-image user-service:latest --name microservices
```

### Install kubectl

```bash
# macOS
brew install kubectl

# Linux
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Verify
kubectl version --client
kubectl cluster-info
```

---

## 17.4 Kubernetes YAML Manifests

### Namespace

```yaml
# k8s/namespaces.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: microservices
  labels:
    name: microservices
    environment: production
---
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
  labels:
    name: monitoring
```

### ConfigMap

```yaml
# k8s/user-service/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: user-service-config
  namespace: microservices
data:
  NODE_ENV: "production"
  PORT: "3001"
  LOG_LEVEL: "info"
  JWT_EXPIRES_IN: "15m"
  REDIS_HOST: "redis-service"
  REDIS_PORT: "6379"
```

### Secret

```yaml
# k8s/user-service/secret.yaml
# ❌ Don't commit actual values - use Sealed Secrets or External Secrets Operator
apiVersion: v1
kind: Secret
metadata:
  name: user-service-secrets
  namespace: microservices
type: Opaque
# Values should be base64 encoded
# echo -n 'your-secret' | base64
data:
  DATABASE_URL: cG9zdGdyZXNxbDovL3VzZXI6cGFzc0Bob3N0OjU0MzIvdXNlcnM=
  JWT_SECRET: c3VwZXItc2VjcmV0LWtleS1mb3ItcHJvZHVjdGlvbg==
  REDIS_PASSWORD: cmVkaXMxMjM=
```

### Deployment

```yaml
# k8s/user-service/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  namespace: microservices
  labels:
    app: user-service
    version: "1.0.0"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: user-service
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # Max pods above desired during update
      maxUnavailable: 0    # No downtime - always have desired replicas
  template:
    metadata:
      labels:
        app: user-service
        version: "1.0.0"
    spec:
      # Security context for the pod
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000

      # Image pull secret (for private registry)
      imagePullSecrets:
        - name: registry-credentials

      containers:
        - name: user-service
          image: your-registry/user-service:1.0.0
          imagePullPolicy: Always
          ports:
            - containerPort: 3001
              name: http

          # Environment from ConfigMap and Secret
          envFrom:
            - configMapRef:
                name: user-service-config
            - secretRef:
                name: user-service-secrets

          # Additional env vars
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: NODE_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.hostIP

          # Resource limits
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"

          # Health checks
          livenessProbe:
            httpGet:
              path: /health/live
              port: 3001
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3

          readinessProbe:
            httpGet:
              path: /health/ready
              port: 3001
            initialDelaySeconds: 10
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 3

          startupProbe:
            httpGet:
              path: /health/live
              port: 3001
            failureThreshold: 30
            periodSeconds: 10

          # Security context for container
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL

          # Volume mounts for writable dirs
          volumeMounts:
            - name: tmp
              mountPath: /tmp
            - name: logs
              mountPath: /app/logs

      volumes:
        - name: tmp
          emptyDir: {}
        - name: logs
          emptyDir: {}

      # Node affinity - prefer nodes with SSD
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 1
              preference:
                matchExpressions:
                  - key: disk
                    operator: In
                    values:
                      - ssd

        # Pod anti-affinity - spread across nodes
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - user-service
                topologyKey: "kubernetes.io/hostname"

      # Graceful shutdown
      terminationGracePeriodSeconds: 60

      # DNS config
      dnsConfig:
        options:
          - name: ndots
            value: "2"
```

### Service

```yaml
# k8s/user-service/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: user-service
  namespace: microservices
  labels:
    app: user-service
spec:
  selector:
    app: user-service
  ports:
    - protocol: TCP
      port: 3001       # Service port
      targetPort: 3001 # Container port
  type: ClusterIP      # Internal only (use Ingress for external)
```

### Horizontal Pod Autoscaler

```yaml
# k8s/user-service/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: user-service-hpa
  namespace: microservices
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: user-service
  minReplicas: 2
  maxReplicas: 10
  metrics:
    # CPU-based scaling
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70

    # Memory-based scaling
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80

    # Custom metric: HTTP requests per second
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "100"

  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Pods
          value: 4
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5 minutes before scale down
      policies:
        - type: Pods
          value: 1
          periodSeconds: 60
```

---

## 17.5 Ingress Controller

```yaml
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: microservices-ingress
  namespace: microservices
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/use-regex: "true"
    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "100"
    nginx.ingress.kubernetes.io/limit-connections: "20"
    # CORS
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://app.example.com"
    # SSL
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    # Proxy settings
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"
spec:
  tls:
    - hosts:
        - api.example.com
      secretName: api-tls-cert

  rules:
    - host: api.example.com
      http:
        paths:
          # Auth routes
          - path: /api/v1/auth
            pathType: Prefix
            backend:
              service:
                name: user-service
                port:
                  number: 3001

          # User routes
          - path: /api/v1/users
            pathType: Prefix
            backend:
              service:
                name: user-service
                port:
                  number: 3001

          # Product routes
          - path: /api/v1/products
            pathType: Prefix
            backend:
              service:
                name: product-service
                port:
                  number: 3002

          - path: /api/v1/categories
            pathType: Prefix
            backend:
              service:
                name: product-service
                port:
                  number: 3002

          # Order routes
          - path: /api/v1/orders
            pathType: Prefix
            backend:
              service:
                name: order-service
                port:
                  number: 3003

          # Default - API Gateway
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-gateway
                port:
                  number: 3000
```

---

## 17.6 Database in Kubernetes

### PostgreSQL with StatefulSet

```yaml
# k8s/databases/postgres-user.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-users-pvc
  namespace: microservices
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: standard  # Use fast-ssd in production
  resources:
    requests:
      storage: 10Gi
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres-users
  namespace: microservices
spec:
  serviceName: "postgres-users"
  replicas: 1
  selector:
    matchLabels:
      app: postgres-users
  template:
    metadata:
      labels:
        app: postgres-users
    spec:
      containers:
        - name: postgres
          image: postgres:15-alpine
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_DB
              value: users_db
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: postgres-users-secret
                  key: username
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-users-secret
                  key: password
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "1"
          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data
          livenessProbe:
            exec:
              command:
                - pg_isready
                - -U
                - $(POSTGRES_USER)
                - -d
                - $(POSTGRES_DB)
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            exec:
              command:
                - pg_isready
                - -U
                - $(POSTGRES_USER)
                - -d
                - $(POSTGRES_DB)
            initialDelaySeconds: 5
            periodSeconds: 5
      volumes:
        - name: postgres-data
          persistentVolumeClaim:
            claimName: postgres-users-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-users-service
  namespace: microservices
spec:
  selector:
    app: postgres-users
  ports:
    - port: 5432
      targetPort: 5432
  type: ClusterIP
```

---

## 17.7 kubectl Commands Reference

```bash
# Namespace
kubectl create namespace microservices
kubectl get namespaces

# Apply manifests
kubectl apply -f k8s/
kubectl apply -f k8s/user-service/

# Pods
kubectl get pods -n microservices
kubectl get pods -n microservices -w  # Watch
kubectl describe pod user-service-xxx -n microservices
kubectl logs user-service-xxx -n microservices
kubectl logs user-service-xxx -n microservices -f  # Follow
kubectl logs user-service-xxx -n microservices --previous  # Previous container
kubectl exec -it user-service-xxx -n microservices -- sh

# Deployments
kubectl get deployments -n microservices
kubectl rollout status deployment/user-service -n microservices
kubectl rollout history deployment/user-service -n microservices

# Rolling update
kubectl set image deployment/user-service user-service=new-image:v2 -n microservices

# Rollback
kubectl rollout undo deployment/user-service -n microservices
kubectl rollout undo deployment/user-service --to-revision=1 -n microservices

# Scale
kubectl scale deployment/user-service --replicas=5 -n microservices

# Services
kubectl get services -n microservices
kubectl describe service user-service -n microservices

# Port forwarding (for local testing)
kubectl port-forward service/user-service 3001:3001 -n microservices
kubectl port-forward pod/user-service-xxx 3001:3001 -n microservices

# ConfigMaps & Secrets
kubectl get configmaps -n microservices
kubectl get secrets -n microservices
kubectl describe configmap user-service-config -n microservices

# Events
kubectl get events -n microservices --sort-by='.lastTimestamp'

# Resource usage
kubectl top pods -n microservices
kubectl top nodes

# Delete
kubectl delete pod user-service-xxx -n microservices  # Pod auto-restarts
kubectl delete deployment user-service -n microservices  # Remove completely
```

---

## 17.8 Complete Microservices K8s Setup

### Directory Structure

```
k8s/
├── namespaces.yaml
├── databases/
│   ├── postgres-users.yaml
│   ├── postgres-orders.yaml
│   ├── mongodb-products.yaml
│   └── redis.yaml
├── user-service/
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── hpa.yaml
├── product-service/
│   ├── configmap.yaml
│   ├── deployment.yaml
│   └── service.yaml
├── order-service/
│   ├── configmap.yaml
│   ├── deployment.yaml
│   └── service.yaml
├── api-gateway/
│   ├── configmap.yaml
│   ├── deployment.yaml
│   └── service.yaml
├── ingress.yaml
└── monitoring/
    ├── prometheus.yaml
    └── grafana.yaml
```

### Deployment Script

```bash
#!/bin/bash
# scripts/deploy.sh

set -e

NAMESPACE="microservices"
REGISTRY="${REGISTRY:-your-registry.io}"
VERSION="${VERSION:-latest}"

echo "Deploying Microservices to Kubernetes..."
echo "Registry: $REGISTRY"
echo "Version: $VERSION"

# Create namespace if not exists
kubectl apply -f k8s/namespaces.yaml

# Deploy databases first
echo "Deploying databases..."
kubectl apply -f k8s/databases/ -n $NAMESPACE

# Wait for databases
echo "Waiting for databases to be ready..."
kubectl wait --for=condition=ready pod -l app=postgres-users -n $NAMESPACE --timeout=120s
kubectl wait --for=condition=ready pod -l app=mongodb-products -n $NAMESPACE --timeout=120s
kubectl wait --for=condition=ready pod -l app=redis -n $NAMESPACE --timeout=60s

# Deploy services
echo "Deploying microservices..."
for service in user-service product-service order-service api-gateway; do
  echo "Deploying $service..."
  
  # Update image version
  kubectl set image deployment/$service $service=$REGISTRY/$service:$VERSION -n $NAMESPACE \
    --record 2>/dev/null || \
  kubectl apply -f k8s/$service/ -n $NAMESPACE
  
  # Wait for rollout
  kubectl rollout status deployment/$service -n $NAMESPACE --timeout=300s
done

# Apply ingress
kubectl apply -f k8s/ingress.yaml -n $NAMESPACE

echo "Deployment complete!"
echo "Services status:"
kubectl get services -n $NAMESPACE
echo ""
echo "Pods status:"
kubectl get pods -n $NAMESPACE
```

---

## 17.9 Pod Disruption Budget

```yaml
# k8s/user-service/pdb.yaml
# Ensures minimum availability during cluster maintenance
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: user-service-pdb
  namespace: microservices
spec:
  minAvailable: 1  # At least 1 pod must be available
  selector:
    matchLabels:
      app: user-service
```

---

## 17.10 Resource Quota

```yaml
# k8s/resource-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: microservices-quota
  namespace: microservices
spec:
  hard:
    # Compute
    requests.cpu: "8"
    requests.memory: 16Gi
    limits.cpu: "16"
    limits.memory: 32Gi
    # Objects
    pods: "50"
    services: "20"
    persistentvolumeclaims: "20"
    secrets: "30"
    configmaps: "30"
---
# LimitRange: defaults for pods that don't specify resources
apiVersion: v1
kind: LimitRange
metadata:
  name: microservices-limits
  namespace: microservices
spec:
  limits:
    - type: Container
      default:
        memory: "256Mi"
        cpu: "250m"
      defaultRequest:
        memory: "128Mi"
        cpu: "100m"
      max:
        memory: "2Gi"
        cpu: "2"
      min:
        memory: "64Mi"
        cpu: "50m"
```

---

## 17.11 Network Policy

```yaml
# k8s/network-policy.yaml
# Restrict which services can communicate with each other

# Default: deny all ingress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: microservices
spec:
  podSelector: {}  # Apply to all pods
  policyTypes:
    - Ingress
    - Egress
---
# Allow API Gateway to communicate with all services
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-gateway
  namespace: microservices
spec:
  podSelector:
    matchLabels:
      app: api-gateway
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - {} # Allow all ingress to api-gateway (from Ingress controller)
  egress:
    - to:
        - podSelector:
            matchLabels:
              tier: backend
      ports:
        - protocol: TCP
          port: 3001
        - protocol: TCP
          port: 3002
        - protocol: TCP
          port: 3003
---
# Allow user-service to access its database and Redis
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: user-service-policy
  namespace: microservices
spec:
  podSelector:
    matchLabels:
      app: user-service
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: api-gateway
      ports:
        - protocol: TCP
          port: 3001
  egress:
    # PostgreSQL
    - to:
        - podSelector:
            matchLabels:
              app: postgres-users
      ports:
        - protocol: TCP
          port: 5432
    # Redis
    - to:
        - podSelector:
            matchLabels:
              app: redis
      ports:
        - protocol: TCP
          port: 6379
    # DNS
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
```

---

## สรุป Part 17

ในบทนี้เราได้เรียนรู้:
- **Kubernetes Architecture** - Control Plane, Worker Nodes, etcd
- **Core Objects** - Pod, Deployment, Service, ConfigMap, Secret
- **Setup** - minikube, kind สำหรับ local development
- **YAML Manifests** - Production-ready configurations
- **HPA** - Horizontal Pod Autoscaler
- **Ingress** - HTTP routing with nginx-ingress
- **StatefulSets** - สำหรับ databases
- **Network Policy** - ควบคุม service-to-service communication
- **Resource Quota** - จำกัด resource usage
- **kubectl** - command reference

**Next:** Part 18 - Kubernetes Advanced - Helm, GitOps, Service Mesh

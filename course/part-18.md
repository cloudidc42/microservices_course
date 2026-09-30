# Part 18: Kubernetes Advanced - Helm, GitOps & Service Mesh

## ภาพรวม

ใน Part นี้เราจะเรียนรู้เครื่องมือขั้นสูงสำหรับ Kubernetes:
- **Helm** - Package Manager สำหรับ Kubernetes
- **GitOps** - การจัดการ Infrastructure ผ่าน Git
- **ArgoCD** - GitOps Continuous Delivery
- **Service Mesh** - Istio สำหรับ Advanced Traffic Management
- **Kustomize** - Template-free Configuration Management

---

## 1. Helm - Kubernetes Package Manager

### 1.1 ทำความเข้าใจ Helm

Helm คือ Package Manager สำหรับ Kubernetes ที่ช่วยให้การ deploy applications ง่ายขึ้น

```
Helm Concepts:
├── Chart      - Package ของ Kubernetes resources
├── Repository - ที่เก็บ Charts
├── Release    - Instance ของ Chart ที่ deploy แล้ว
└── Values     - Configuration parameters
```

### 1.2 ติดตั้ง Helm

```bash
# macOS
brew install helm

# Linux
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Windows
choco install kubernetes-helm

# ตรวจสอบ
helm version
```

### 1.3 สร้าง Helm Chart สำหรับ User Service

```bash
# สร้าง Chart structure
helm create user-service-chart

# Structure ที่ได้
user-service-chart/
├── Chart.yaml          # Metadata
├── values.yaml         # Default values
├── values-dev.yaml     # Dev environment values
├── values-prod.yaml    # Production values
├── templates/
│   ├── _helpers.tpl    # Template helpers
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── hpa.yaml
│   ├── pdb.yaml
│   └── NOTES.txt
└── charts/             # Dependencies
```

### 1.4 Chart.yaml

```yaml
# user-service-chart/Chart.yaml
apiVersion: v2
name: user-service
description: User Service for Microservices Platform
type: application
version: 0.1.0
appVersion: "1.0.0"
keywords:
  - microservices
  - user
  - api
maintainers:
  - name: Platform Team
    email: platform@company.com
dependencies:
  - name: postgresql
    version: "12.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled
  - name: redis
    version: "17.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
```

### 1.5 values.yaml - Default Values

```yaml
# user-service-chart/values.yaml
# Global settings
global:
  imageRegistry: ""
  imagePullSecrets: []
  storageClass: ""

# Image settings
image:
  repository: company/user-service
  tag: "latest"
  pullPolicy: IfNotPresent

# Replica settings
replicaCount: 2

# Service settings
service:
  type: ClusterIP
  port: 80
  targetPort: 3001

# Ingress settings
ingress:
  enabled: false
  className: "nginx"
  annotations: {}
  hosts:
    - host: user-service.local
      paths:
        - path: /
          pathType: Prefix
  tls: []

# Resource settings
resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi

# Autoscaling
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80

# Environment variables
env:
  NODE_ENV: production
  PORT: "3001"
  LOG_LEVEL: info

# Secrets (referenced from K8s Secret)
envFromSecret:
  enabled: true
  secretName: user-service-secrets

# ConfigMap
config:
  DATABASE_HOST: postgresql
  DATABASE_PORT: "5432"
  DATABASE_NAME: userdb
  REDIS_HOST: redis-master

# Health checks
livenessProbe:
  enabled: true
  path: /health/live
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  enabled: true
  path: /health/ready
  initialDelaySeconds: 10
  periodSeconds: 5

# Pod Disruption Budget
podDisruptionBudget:
  enabled: true
  minAvailable: 1

# Security Context
podSecurityContext:
  fsGroup: 1000
  runAsNonRoot: true
  runAsUser: 1000

containerSecurityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop:
      - ALL

# Node selector and tolerations
nodeSelector: {}
tolerations: []
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
              - key: app.kubernetes.io/name
                operator: In
                values:
                  - user-service
          topologyKey: kubernetes.io/hostname

# PostgreSQL dependency
postgresql:
  enabled: true
  auth:
    username: userservice
    database: userdb
    existingSecret: postgresql-secret
  primary:
    persistence:
      enabled: true
      size: 10Gi

# Redis dependency
redis:
  enabled: true
  auth:
    enabled: true
    existingSecret: redis-secret
  master:
    persistence:
      enabled: true
      size: 2Gi
```

### 1.6 templates/_helpers.tpl

```yaml
{{/*
Expand the name of the chart.
*/}}
{{- define "user-service.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
*/}}
{{- define "user-service.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Create chart label
*/}}
{{- define "user-service.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "user-service.labels" -}}
helm.sh/chart: {{ include "user-service.chart" . }}
{{ include "user-service.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "user-service.selectorLabels" -}}
app.kubernetes.io/name: {{ include "user-service.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
Create the name of the service account to use
*/}}
{{- define "user-service.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "user-service.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}

{{/*
Image reference
*/}}
{{- define "user-service.image" -}}
{{- $registry := .Values.global.imageRegistry | default "" }}
{{- $repository := .Values.image.repository }}
{{- $tag := .Values.image.tag | default .Chart.AppVersion }}
{{- if $registry }}
{{- printf "%s/%s:%s" $registry $repository $tag }}
{{- else }}
{{- printf "%s:%s" $repository $tag }}
{{- end }}
{{- end }}
```

### 1.7 templates/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "user-service.fullname" . }}
  labels:
    {{- include "user-service.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "user-service.selectorLabels" . | nindent 6 }}
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
        checksum/secret: {{ include (print $.Template.BasePath "/secret.yaml") . | sha256sum }}
      labels:
        {{- include "user-service.selectorLabels" . | nindent 8 }}
    spec:
      {{- with .Values.global.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
      terminationGracePeriodSeconds: 60
      containers:
        - name: {{ .Chart.Name }}
          securityContext:
            {{- toYaml .Values.containerSecurityContext | nindent 12 }}
          image: {{ include "user-service.image" . }}
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.targetPort }}
              protocol: TCP
          env:
            {{- range $key, $value := .Values.env }}
            - name: {{ $key }}
              value: {{ $value | quote }}
            {{- end }}
          envFrom:
            - configMapRef:
                name: {{ include "user-service.fullname" . }}-config
            {{- if .Values.envFromSecret.enabled }}
            - secretRef:
                name: {{ .Values.envFromSecret.secretName }}
            {{- end }}
          {{- if .Values.livenessProbe.enabled }}
          livenessProbe:
            httpGet:
              path: {{ .Values.livenessProbe.path }}
              port: http
            initialDelaySeconds: {{ .Values.livenessProbe.initialDelaySeconds }}
            periodSeconds: {{ .Values.livenessProbe.periodSeconds }}
            timeoutSeconds: 5
            failureThreshold: 3
          {{- end }}
          {{- if .Values.readinessProbe.enabled }}
          readinessProbe:
            httpGet:
              path: {{ .Values.readinessProbe.path }}
              port: http
            initialDelaySeconds: {{ .Values.readinessProbe.initialDelaySeconds }}
            periodSeconds: {{ .Values.readinessProbe.periodSeconds }}
            timeoutSeconds: 3
            failureThreshold: 3
          {{- end }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
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
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

### 1.8 values-prod.yaml สำหรับ Production

```yaml
# user-service-chart/values-prod.yaml
image:
  tag: "1.2.3"  # Pinned version for production

replicaCount: 3

resources:
  limits:
    cpu: 1000m
    memory: 1Gi
  requests:
    cpu: 500m
    memory: 512Mi

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 20

ingress:
  enabled: true
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
  hosts:
    - host: api.company.com
      paths:
        - path: /users
          pathType: Prefix
  tls:
    - secretName: api-tls
      hosts:
        - api.company.com

env:
  NODE_ENV: production
  LOG_LEVEL: warn

podDisruptionBudget:
  enabled: true
  minAvailable: 2
```

### 1.9 Helm Commands

```bash
# เพิ่ม Repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Search Charts
helm search repo bitnami/postgresql

# Install Chart
helm install user-service ./user-service-chart \
  --namespace production \
  --create-namespace \
  -f values-prod.yaml

# Upgrade Release
helm upgrade user-service ./user-service-chart \
  --namespace production \
  -f values-prod.yaml

# Install หรือ Upgrade (upsert)
helm upgrade --install user-service ./user-service-chart \
  --namespace production \
  --create-namespace \
  -f values-prod.yaml \
  --atomic \
  --timeout 5m

# ดู Releases
helm list --namespace production
helm list --all-namespaces

# ดู Values ที่ใช้
helm get values user-service --namespace production

# Rollback
helm rollback user-service 1 --namespace production

# Dry run
helm upgrade --install user-service ./user-service-chart \
  --dry-run \
  --debug \
  -f values-prod.yaml

# Lint
helm lint ./user-service-chart

# Template (render YAML)
helm template user-service ./user-service-chart \
  -f values-prod.yaml > rendered.yaml

# Uninstall
helm uninstall user-service --namespace production

# Package Chart
helm package ./user-service-chart

# Push ไป OCI Registry
helm push user-service-0.1.0.tgz oci://registry.company.com/charts
```

---

## 2. Kustomize - Template-free Configuration

### 2.1 ความแตกต่างจาก Helm

```
Helm                          Kustomize
──────────────────────────    ──────────────────────────
Template engine               Overlay/Patch system
Values injection              Strategic merge patches
Good for reusable packages    Good for environment-specific
Complex templating            Simple, declarative
Package versioning            Git-based versioning
```

### 2.2 Kustomize Directory Structure

```
k8s/
├── base/                   # Base configurations
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── configmap.yaml
└── overlays/
    ├── development/
    │   ├── kustomization.yaml
    │   └── patches/
    │       └── deployment-patch.yaml
    ├── staging/
    │   ├── kustomization.yaml
    │   └── patches/
    │       └── deployment-patch.yaml
    └── production/
        ├── kustomization.yaml
        └── patches/
            ├── deployment-patch.yaml
            └── hpa-patch.yaml
```

### 2.3 base/kustomization.yaml

```yaml
# k8s/base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml
  - hpa.yaml

commonLabels:
  app: user-service
  team: platform

commonAnnotations:
  managed-by: kustomize
```

### 2.4 overlays/production/kustomization.yaml

```yaml
# k8s/overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: production

bases:
  - ../../base

patches:
  - path: patches/deployment-patch.yaml
    target:
      kind: Deployment
      name: user-service
  - path: patches/hpa-patch.yaml
    target:
      kind: HorizontalPodAutoscaler
      name: user-service

images:
  - name: company/user-service
    newTag: "1.2.3"

configMapGenerator:
  - name: user-service-config
    behavior: merge
    literals:
      - NODE_ENV=production
      - LOG_LEVEL=warn

replicas:
  - name: user-service
    count: 3
```

### 2.5 Production Deployment Patch

```yaml
# k8s/overlays/production/patches/deployment-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
spec:
  template:
    spec:
      containers:
        - name: user-service
          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
            limits:
              cpu: "1000m"
              memory: "1Gi"
          env:
            - name: NODE_ENV
              value: production
```

### 2.6 Kustomize Commands

```bash
# Build (render YAML)
kubectl kustomize k8s/overlays/production

# Apply
kubectl apply -k k8s/overlays/production

# Diff กับ cluster
kubectl diff -k k8s/overlays/production

# Delete
kubectl delete -k k8s/overlays/production
```

---

## 3. GitOps กับ ArgoCD

### 3.1 GitOps Principles

```
GitOps = Git เป็น Single Source of Truth

หลักการ:
1. Declarative  - ทุกอย่างเขียนเป็น YAML/config
2. Versioned    - ทุกการเปลี่ยนแปลงผ่าน Git commit
3. Automatic    - Agent sync cluster ให้ตรงกับ Git
4. Observable   - รู้ว่า cluster ต่างจาก desired state อย่างไร
```

### 3.2 ติดตั้ง ArgoCD

```bash
# สร้าง Namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# รอให้ pods พร้อม
kubectl wait --for=condition=available --timeout=300s \
  deployment/argocd-server -n argocd

# Port-forward เพื่อเข้า UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# ดู Initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# Install ArgoCD CLI
brew install argocd

# Login
argocd login localhost:8080 --username admin --insecure
```

### 3.3 Repository Structure สำหรับ GitOps

```
gitops-repo/
├── apps/                     # ArgoCD Application definitions
│   ├── user-service.yaml
│   ├── order-service.yaml
│   └── app-of-apps.yaml
├── infrastructure/           # Shared infrastructure
│   ├── cert-manager/
│   ├── ingress-nginx/
│   ├── monitoring/
│   └── databases/
└── services/                 # Service configurations
    ├── user-service/
    │   ├── base/
    │   └── overlays/
    │       ├── dev/
    │       ├── staging/
    │       └── prod/
    ├── order-service/
    └── ...
```

### 3.4 ArgoCD Application Definition

```yaml
# apps/user-service.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: user-service-prod
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: production

  source:
    repoURL: https://github.com/company/gitops-repo
    targetRevision: main
    path: services/user-service/overlays/prod
    # หรือใช้ Helm
    # chart: user-service
    # repoURL: https://charts.company.com
    # targetRevision: 1.2.3
    # helm:
    #   valueFiles:
    #     - values-prod.yaml

  destination:
    server: https://kubernetes.default.svc
    namespace: production

  syncPolicy:
    automated:
      prune: true          # ลบ resources ที่ไม่อยู่ใน Git
      selfHeal: true       # แก้ไขเมื่อ cluster drift
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m

  # Health checks
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas  # ให้ HPA จัดการ replicas
```

### 3.5 App of Apps Pattern

```yaml
# apps/app-of-apps.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-of-apps
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/company/gitops-repo
    targetRevision: main
    path: apps
    directory:
      recurse: true
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### 3.6 ArgoCD Project สำหรับ RBAC

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: production
  namespace: argocd
spec:
  description: Production environment

  sourceRepos:
    - https://github.com/company/gitops-repo
    - https://charts.company.com

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
    - name: read-only
      description: Read only access
      policies:
        - p, proj:production:read-only, applications, get, production/*, allow
      groups:
        - company:developers
    - name: sync-only
      description: Sync access
      policies:
        - p, proj:production:sync-only, applications, sync, production/*, allow
      groups:
        - company:devops
```

### 3.7 ArgoCD CLI Commands

```bash
# ดู Applications
argocd app list

# ดู Application details
argocd app get user-service-prod

# Sync Application
argocd app sync user-service-prod

# Sync with prune
argocd app sync user-service-prod --prune

# Rollback
argocd app rollback user-service-prod 2

# History
argocd app history user-service-prod

# Set manual sync
argocd app set user-service-prod --sync-policy none

# Delete Application
argocd app delete user-service-prod

# Diff กับ live state
argocd app diff user-service-prod

# Wait for sync
argocd app wait user-service-prod --sync
```

### 3.8 Image Updater - Auto Update Images

```bash
# ติดตั้ง ArgoCD Image Updater
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/manifests/install.yaml
```

```yaml
# Application with Image Updater annotations
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: user-service-prod
  namespace: argocd
  annotations:
    argocd-image-updater.argoproj.io/image-list: "user-service=company/user-service"
    argocd-image-updater.argoproj.io/user-service.update-strategy: semver
    argocd-image-updater.argoproj.io/user-service.allow-tags: "regexp:^1\\..*"
    argocd-image-updater.argoproj.io/write-back-method: git
    argocd-image-updater.argoproj.io/git-branch: main
spec:
  # ...
```

---

## 4. Service Mesh กับ Istio

### 4.1 Service Mesh คืออะไร?

```
Traditional Microservices:
Service A ──── HTTP ────> Service B
(ต้องจัดการ: retry, timeout, circuit breaker, mTLS ใน code)

With Service Mesh:
Service A ──── HTTP ──> [Sidecar A] ──── HTTP ──> [Sidecar B] ──> Service B
                                    (Istio จัดการทุกอย่าง)

Capabilities:
├── Traffic Management  - Load balancing, canary, circuit breaking
├── Security           - mTLS, authorization policies
├── Observability      - Metrics, tracing, logging (automatic)
└── Resilience         - Retries, timeouts, fault injection
```

### 4.2 ติดตั้ง Istio

```bash
# Download Istio
curl -L https://istio.io/downloadIstio | sh -
cd istio-1.x.x
export PATH=$PWD/bin:$PATH

# Install Istio (demo profile)
istioctl install --set profile=demo -y

# หรือ production profile
istioctl install --set profile=default \
  --set values.gateways.istio-ingressgateway.type=LoadBalancer

# Enable Istio injection สำหรับ namespace
kubectl label namespace production istio-injection=enabled

# ตรวจสอบ
kubectl get pods -n istio-system
istioctl verify-install
```

### 4.3 Virtual Service - Traffic Management

```yaml
# virtual-service.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: user-service
  namespace: production
spec:
  hosts:
    - user-service
  http:
    # Canary deployment: 90/10 traffic split
    - name: canary
      match:
        - headers:
            x-canary:
              exact: "true"
      route:
        - destination:
            host: user-service
            subset: v2
          weight: 100

    # Header-based routing
    - name: beta-users
      match:
        - headers:
            x-beta-user:
              exact: "true"
      route:
        - destination:
            host: user-service
            subset: v2

    # Default routing with weight
    - name: default
      route:
        - destination:
            host: user-service
            subset: v1
          weight: 90
        - destination:
            host: user-service
            subset: v2
          weight: 10
      # Retry policy
      retries:
        attempts: 3
        perTryTimeout: 2s
        retryOn: 5xx,reset,connect-failure
      # Timeout
      timeout: 10s
      # Fault injection (สำหรับ testing)
      fault:
        delay:
          percentage:
            value: 0.1
          fixedDelay: 5s
```

### 4.4 Destination Rule - Load Balancing & Circuit Breaker

```yaml
# destination-rule.yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: user-service
  namespace: production
spec:
  host: user-service
  trafficPolicy:
    # Connection pool settings
    connectionPool:
      tcp:
        maxConnections: 100
        connectTimeout: 30ms
      http:
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
        maxRetries: 3

    # Circuit breaker (Outlier Detection)
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
      minHealthPercent: 30

    # Load balancing
    loadBalancer:
      simple: LEAST_CONN

    # mTLS
    tls:
      mode: ISTIO_MUTUAL

  # Subsets for canary
  subsets:
    - name: v1
      labels:
        version: v1
      trafficPolicy:
        connectionPool:
          http:
            http1MaxPendingRequests: 200
    - name: v2
      labels:
        version: v2
```

### 4.5 Istio Gateway

```yaml
# gateway.yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: main-gateway
  namespace: production
spec:
  selector:
    istio: ingressgateway
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - "*.company.com"
      tls:
        httpsRedirect: true
    - port:
        number: 443
        name: https
        protocol: HTTPS
      hosts:
        - "*.company.com"
      tls:
        mode: SIMPLE
        credentialName: company-tls

---
# virtual-service for gateway
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: api-gateway
  namespace: production
spec:
  hosts:
    - "api.company.com"
  gateways:
    - main-gateway
  http:
    - match:
        - uri:
            prefix: /users
      route:
        - destination:
            host: user-service
            port:
              number: 80
    - match:
        - uri:
            prefix: /orders
      route:
        - destination:
            host: order-service
            port:
              number: 80
```

### 4.6 Authorization Policy - Zero Trust Security

```yaml
# authorization-policy.yaml

# Deny all by default
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: production
spec:
  {}  # Empty spec = deny all

---
# Allow API Gateway to call services
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-api-gateway
  namespace: production
spec:
  selector:
    matchLabels:
      app: user-service
  action: ALLOW
  rules:
    - from:
        - source:
            principals:
              - "cluster.local/ns/production/sa/api-gateway"
      to:
        - operation:
            methods: ["GET", "POST", "PUT", "DELETE"]
            paths: ["/users/*", "/auth/*"]

---
# Allow user-service to call other services
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-service-to-service
  namespace: production
spec:
  selector:
    matchLabels:
      app: order-service
  action: ALLOW
  rules:
    - from:
        - source:
            principals:
              - "cluster.local/ns/production/sa/user-service"
              - "cluster.local/ns/production/sa/api-gateway"
      to:
        - operation:
            methods: ["GET", "POST"]
```

### 4.7 PeerAuthentication - mTLS

```yaml
# peer-authentication.yaml

# Enable strict mTLS for production namespace
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT

---
# Per-service mTLS (อนุญาต permissive สำหรับ legacy)
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: legacy-service
  namespace: production
spec:
  selector:
    matchLabels:
      app: legacy-service
  mtls:
    mode: PERMISSIVE
```

### 4.8 Kiali - Service Mesh Dashboard

```bash
# Install Kiali, Prometheus, Grafana, Jaeger addons
kubectl apply -f samples/addons/

# Access Kiali
istioctl dashboard kiali

# Access Grafana
istioctl dashboard grafana

# Access Jaeger
istioctl dashboard jaeger
```

---

## 5. CI/CD Pipeline สำหรับ GitOps

### 5.1 GitHub Actions Workflow

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches: [main]
    paths:
      - 'services/user-service/**'
      - '.github/workflows/deploy.yml'

env:
  SERVICE: user-service
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}/user-service

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
          cache-dependency-path: services/user-service/package-lock.json

      - name: Install dependencies
        working-directory: services/user-service
        run: npm ci

      - name: Run tests
        working-directory: services/user-service
        run: npm test -- --coverage

      - name: Upload coverage
        uses: codecov/codecov-action@v3

  security-scan:
    runs-on: ubuntu-latest
    needs: test
    steps:
      - uses: actions/checkout@v4

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: 'services/user-service'
          format: 'sarif'
          output: 'trivy-results.sarif'

      - name: Upload Trivy results
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'

  build:
    runs-on: ubuntu-latest
    needs: [test, security-scan]
    outputs:
      image-tag: ${{ steps.meta.outputs.version }}

    steps:
      - uses: actions/checkout@v4

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=sha-
            type=semver,pattern={{version}}
            type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: services/user-service
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Scan built image
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ github.sha }}
          format: 'table'
          exit-code: '1'
          severity: 'CRITICAL,HIGH'

  update-gitops:
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Checkout GitOps repo
        uses: actions/checkout@v4
        with:
          repository: company/gitops-repo
          token: ${{ secrets.GITOPS_TOKEN }}
          path: gitops

      - name: Update image tag
        working-directory: gitops
        run: |
          IMAGE_TAG="${{ needs.build.outputs.image-tag }}"
          
          # Update kustomization.yaml
          cd services/user-service/overlays/prod
          kustomize edit set image \
            company/user-service=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${IMAGE_TAG}

      - name: Commit and push
        working-directory: gitops
        run: |
          git config user.name "GitHub Actions"
          git config user.email "actions@github.com"
          git add .
          git commit -m "chore(user-service): update image to ${{ needs.build.outputs.image-tag }}"
          git push

  verify-deployment:
    runs-on: ubuntu-latest
    needs: update-gitops
    steps:
      - name: Wait for ArgoCD sync
        run: |
          # ใช้ ArgoCD CLI รอให้ sync เสร็จ
          argocd app wait user-service-prod \
            --sync \
            --health \
            --timeout 300 \
            --server ${{ secrets.ARGOCD_SERVER }} \
            --auth-token ${{ secrets.ARGOCD_TOKEN }} \
            --insecure

      - name: Run smoke tests
        run: |
          curl -f https://api.company.com/users/health || exit 1
```

### 5.2 Makefile สำหรับ Local Development

```makefile
# Makefile
.PHONY: help install build test deploy clean

NAMESPACE ?= production
SERVICE ?= user-service
VERSION ?= $(shell git rev-parse --short HEAD)
REGISTRY ?= ghcr.io/company

help:
	@echo "Available targets:"
	@echo "  install    - Install dependencies"
	@echo "  build      - Build Docker image"
	@echo "  test       - Run tests"
	@echo "  deploy     - Deploy to Kubernetes"
	@echo "  rollback   - Rollback deployment"
	@echo "  logs       - Show service logs"
	@echo "  clean      - Clean up"

install:
	cd services/$(SERVICE) && npm ci

build:
	docker build -t $(REGISTRY)/$(SERVICE):$(VERSION) \
	  services/$(SERVICE)
	docker push $(REGISTRY)/$(SERVICE):$(VERSION)

test:
	cd services/$(SERVICE) && npm test

lint:
	helm lint charts/$(SERVICE)
	kubectl kustomize k8s/overlays/$(ENV)

deploy:
	helm upgrade --install $(SERVICE) charts/$(SERVICE) \
	  --namespace $(NAMESPACE) \
	  --set image.tag=$(VERSION) \
	  --values charts/$(SERVICE)/values-$(ENV).yaml \
	  --atomic \
	  --timeout 5m

rollback:
	helm rollback $(SERVICE) --namespace $(NAMESPACE)

logs:
	kubectl logs -l app=$(SERVICE) \
	  --namespace $(NAMESPACE) \
	  --tail=100 \
	  --follow

port-forward:
	kubectl port-forward svc/$(SERVICE) 3001:80 \
	  --namespace $(NAMESPACE)

clean:
	helm uninstall $(SERVICE) --namespace $(NAMESPACE)
```

---

## 6. Multi-Cluster Management

### 6.1 Fleet ด้วย ArgoCD

```yaml
# ArgoCD ApplicationSet สำหรับ Multi-cluster
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: user-service-all-clusters
  namespace: argocd
spec:
  generators:
    - clusters:
        selector:
          matchLabels:
            environment: production

  template:
    metadata:
      name: "user-service-{{name}}"
    spec:
      project: production
      source:
        repoURL: https://github.com/company/gitops-repo
        targetRevision: main
        path: "services/user-service/overlays/{{metadata.labels.region}}"
      destination:
        server: "{{server}}"
        namespace: production
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

---

## Workshop: สร้าง Complete GitOps Pipeline

### Step 1: เตรียม Project Structure

```bash
mkdir -p gitops-demo/{services,apps,infrastructure}
cd gitops-demo

# สร้าง service configuration
mkdir -p services/user-service/{base,overlays/{dev,prod}}

# สร้าง ArgoCD app definitions
mkdir -p apps
```

### Step 2: สร้าง Base Kubernetes Manifests

```bash
# services/user-service/base/deployment.yaml
cat > services/user-service/base/deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
spec:
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
    spec:
      containers:
        - name: user-service
          image: company/user-service:latest
          ports:
            - containerPort: 3001
          env:
            - name: PORT
              value: "3001"
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi
EOF

# services/user-service/base/kustomization.yaml
cat > services/user-service/base/kustomization.yaml << 'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
EOF
```

### Step 3: สร้าง Production Overlay

```bash
cat > services/user-service/overlays/prod/kustomization.yaml << 'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: production

bases:
  - ../../base

images:
  - name: company/user-service
    newTag: "1.0.0"

replicas:
  - name: user-service
    count: 3

patches:
  - path: deployment-patch.yaml
EOF
```

### Step 4: Deploy ArgoCD Application

```bash
# Apply Application definition
kubectl apply -f apps/user-service-prod.yaml

# Watch sync status
argocd app get user-service-prod --watch

# Trigger manual sync
argocd app sync user-service-prod
```

### Step 5: ทดสอบ Canary Deployment

```bash
# Deploy v2 ด้วย 10% traffic
kubectl apply -f - << 'EOF'
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: user-service
  namespace: production
spec:
  hosts:
    - user-service
  http:
    - route:
        - destination:
            host: user-service
            subset: v1
          weight: 90
        - destination:
            host: user-service
            subset: v2
          weight: 10
EOF

# Monitor error rate ใน Kiali
# ถ้าดีขึ้นเรื่อยๆ ให้ค่อยๆ เพิ่ม weight ของ v2
# 90/10 -> 70/30 -> 50/50 -> 0/100
```

---

## สรุป

ใน Part 18 เราได้เรียนรู้:

| เครื่องมือ | ใช้สำหรับ |
|-----------|-----------|
| **Helm** | Package และ deploy Kubernetes applications |
| **Kustomize** | Environment-specific configurations |
| **ArgoCD** | GitOps continuous delivery |
| **Istio** | Service mesh (traffic, security, observability) |
| **ApplicationSet** | Multi-cluster management |

**GitOps Flow:**
```
Developer pushes code
       ↓
GitHub Actions: Test → Build → Push Image
       ↓
GitHub Actions: Update GitOps repo (image tag)
       ↓
ArgoCD detects changes in Git
       ↓
ArgoCD syncs to Kubernetes cluster
       ↓
Istio manages traffic routing
```

**Next:** Part 19 - Advanced Observability: Distributed Tracing, SLOs, and Alerting

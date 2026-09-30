# Part 31: Advanced Kubernetes - Operators and Custom Resources

## ภาพรวม

ใน Part นี้เราจะเรียนรู้ Kubernetes ขั้นสูง:
- **Custom Resource Definitions (CRD)** - ขยาย Kubernetes API
- **Kubernetes Operators** - Automate complex operations
- **Controller Pattern** - Reconciliation loop
- **Admission Webhooks** - Intercept and validate resources
- **Kubectl Plugins** - เครื่องมือเพิ่มเติม

---

## 1. Custom Resource Definitions (CRD)

### 1.1 สร้าง CRD สำหรับ Microservice

```yaml
# crd/microservice.yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: microservices.platform.company.com
spec:
  group: platform.company.com
  scope: Namespaced
  names:
    plural: microservices
    singular: microservice
    kind: Microservice
    shortNames: [ms]
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          required: [spec]
          properties:
            spec:
              type: object
              required: [image, port]
              properties:
                image:
                  type: string
                  description: Docker image
                port:
                  type: integer
                  minimum: 1
                  maximum: 65535
                replicas:
                  type: integer
                  minimum: 1
                  maximum: 50
                  default: 2
                resources:
                  type: object
                  properties:
                    cpu:
                      type: string
                      default: "250m"
                    memory:
                      type: string
                      default: "256Mi"
                autoscaling:
                  type: object
                  properties:
                    enabled:
                      type: boolean
                      default: true
                    minReplicas:
                      type: integer
                      default: 2
                    maxReplicas:
                      type: integer
                      default: 10
                    targetCPUPercent:
                      type: integer
                      default: 70
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
                      default: /
                    tlsEnabled:
                      type: boolean
                      default: true
                monitoring:
                  type: object
                  properties:
                    enabled:
                      type: boolean
                      default: true
                    scrapeInterval:
                      type: string
                      default: "15s"
            status:
              type: object
              properties:
                phase:
                  type: string
                  enum: [Pending, Running, Failed, Degraded]
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
      additionalPrinterColumns:
        - name: Image
          type: string
          jsonPath: .spec.image
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
```

### 1.2 ตัวอย่าง Custom Resource

```yaml
# examples/user-service.yaml
apiVersion: platform.company.com/v1
kind: Microservice
metadata:
  name: user-service
  namespace: production
  labels:
    team: platform
    tier: backend
spec:
  image: ghcr.io/company/user-service:1.2.3
  port: 3001
  replicas: 3
  resources:
    cpu: "500m"
    memory: "512Mi"
  autoscaling:
    enabled: true
    minReplicas: 3
    maxReplicas: 20
    targetCPUPercent: 70
  ingress:
    enabled: true
    host: api.company.com
    path: /users
    tlsEnabled: true
  monitoring:
    enabled: true
    scrapeInterval: "15s"
```

---

## 2. Kubernetes Operator ด้วย Node.js

### 2.1 Operator Setup

```javascript
// operator/index.js
const k8s = require('@kubernetes/client-node');
const { MicroserviceController } = require('./controllers/microservice-controller');

async function main() {
  const kc = new k8s.KubeConfig();
  kc.loadFromDefault();

  const controller = new MicroserviceController(kc);
  await controller.start();
}

main().catch(console.error);
```

### 2.2 Controller Implementation

```javascript
// operator/controllers/microservice-controller.js
const k8s = require('@kubernetes/client-node');

class MicroserviceController {
  constructor(kubeConfig) {
    this.kc = kubeConfig;
    this.k8sApi = kubeConfig.makeApiClient(k8s.AppsV1Api);
    this.coreApi = kubeConfig.makeApiClient(k8s.CoreV1Api);
    this.networkingApi = kubeConfig.makeApiClient(k8s.NetworkingV1Api);
    this.customObjectsApi = kubeConfig.makeApiClient(k8s.CustomObjectsApi);
    this.group = 'platform.company.com';
    this.version = 'v1';
    this.plural = 'microservices';
  }

  async start() {
    console.log('Starting Microservice Operator...');
    
    // Watch for CRD changes
    const watch = new k8s.Watch(this.kc);
    
    const watchHandler = async (type, obj) => {
      try {
        await this.reconcile(type, obj);
      } catch (err) {
        console.error(`Reconciliation error for ${obj.metadata.name}:`, err);
        await this.updateStatus(obj, 'Failed', err.message);
      }
    };

    // Start watching all namespaces
    const watchRequest = await watch.watch(
      `/apis/${this.group}/${this.version}/${this.plural}`,
      {},
      watchHandler,
      (err) => {
        if (err) {
          console.error('Watch error:', err);
          // Reconnect after 5 seconds
          setTimeout(() => this.start(), 5000);
        }
      }
    );

    console.log('Watching Microservice resources...');
    
    // Graceful shutdown
    process.on('SIGTERM', () => {
      watchRequest.abort();
      process.exit(0);
    });
  }

  async reconcile(eventType, resource) {
    const { name, namespace } = resource.metadata;
    console.log(`Reconciling ${eventType} ${namespace}/${name}`);

    if (eventType === 'DELETED') {
      // Resources are garbage collected via owner references
      return;
    }

    await this.updateStatus(resource, 'Pending', 'Reconciling...');

    const { spec } = resource;

    // Create or update all child resources
    await Promise.all([
      this.reconcileDeployment(resource),
      this.reconcileService(resource),
      spec.autoscaling?.enabled && this.reconcileHPA(resource),
      spec.ingress?.enabled && this.reconcileIngress(resource),
      spec.monitoring?.enabled && this.reconcileServiceMonitor(resource)
    ].filter(Boolean));

    // Update status
    await this.updateStatus(resource, 'Running', 'All resources reconciled');
  }

  async reconcileDeployment(resource) {
    const { name, namespace } = resource.metadata;
    const { spec } = resource;

    const deployment = {
      apiVersion: 'apps/v1',
      kind: 'Deployment',
      metadata: {
        name,
        namespace,
        labels: this.getLabels(resource),
        ownerReferences: this.getOwnerReference(resource)
      },
      spec: {
        replicas: spec.replicas || 2,
        selector: { matchLabels: { 'app.kubernetes.io/name': name } },
        template: {
          metadata: {
            labels: this.getLabels(resource),
            annotations: {
              'prometheus.io/scrape': String(spec.monitoring?.enabled || true),
              'prometheus.io/port': String(spec.port)
            }
          },
          spec: {
            securityContext: {
              runAsNonRoot: true,
              runAsUser: 1000
            },
            containers: [{
              name,
              image: spec.image,
              ports: [{ containerPort: spec.port }],
              resources: {
                requests: {
                  cpu: spec.resources?.cpu || '250m',
                  memory: spec.resources?.memory || '256Mi'
                },
                limits: {
                  cpu: spec.resources?.cpu || '500m',
                  memory: spec.resources?.memory || '512Mi'
                }
              },
              securityContext: {
                allowPrivilegeEscalation: false,
                readOnlyRootFilesystem: true,
                capabilities: { drop: ['ALL'] }
              },
              livenessProbe: {
                httpGet: { path: '/health/live', port: spec.port },
                initialDelaySeconds: 30,
                periodSeconds: 10
              },
              readinessProbe: {
                httpGet: { path: '/health/ready', port: spec.port },
                initialDelaySeconds: 10,
                periodSeconds: 5
              },
              volumeMounts: [
                { name: 'tmp', mountPath: '/tmp' }
              ]
            }],
            volumes: [
              { name: 'tmp', emptyDir: {} }
            ]
          }
        }
      }
    };

    try {
      await this.k8sApi.readNamespacedDeployment(name, namespace);
      await this.k8sApi.patchNamespacedDeployment(
        name, namespace, deployment,
        undefined, undefined, undefined, undefined,
        { headers: { 'Content-Type': 'application/merge-patch+json' } }
      );
    } catch (err) {
      if (err.statusCode === 404) {
        await this.k8sApi.createNamespacedDeployment(namespace, deployment);
      } else {
        throw err;
      }
    }
  }

  async reconcileHPA(resource) {
    const { name, namespace } = resource.metadata;
    const { autoscaling } = resource.spec;

    const hpa = {
      apiVersion: 'autoscaling/v2',
      kind: 'HorizontalPodAutoscaler',
      metadata: {
        name,
        namespace,
        ownerReferences: this.getOwnerReference(resource)
      },
      spec: {
        scaleTargetRef: {
          apiVersion: 'apps/v1',
          kind: 'Deployment',
          name
        },
        minReplicas: autoscaling.minReplicas || 2,
        maxReplicas: autoscaling.maxReplicas || 10,
        metrics: [{
          type: 'Resource',
          resource: {
            name: 'cpu',
            target: {
              type: 'Utilization',
              averageUtilization: autoscaling.targetCPUPercent || 70
            }
          }
        }]
      }
    };

    await this.applyResource(
      'autoscaling',
      'v2',
      'horizontalpodautoscalers',
      namespace,
      name,
      hpa
    );
  }

  getLabels(resource) {
    return {
      'app.kubernetes.io/name': resource.metadata.name,
      'app.kubernetes.io/instance': resource.metadata.name,
      'app.kubernetes.io/managed-by': 'microservice-operator',
      'platform.company.com/team': resource.metadata.labels?.team || 'unknown'
    };
  }

  getOwnerReference(resource) {
    return [{
      apiVersion: `${this.group}/${this.version}`,
      kind: 'Microservice',
      name: resource.metadata.name,
      uid: resource.metadata.uid,
      controller: true,
      blockOwnerDeletion: true
    }];
  }

  async updateStatus(resource, phase, message) {
    const { name, namespace } = resource.metadata;
    
    try {
      await this.customObjectsApi.patchNamespacedCustomObjectStatus(
        this.group, this.version, namespace, this.plural, name,
        {
          status: {
            phase,
            conditions: [{
              type: phase,
              status: phase === 'Running' ? 'True' : 'False',
              reason: message,
              message,
              lastTransitionTime: new Date().toISOString()
            }]
          }
        },
        undefined, undefined, undefined,
        { headers: { 'Content-Type': 'application/merge-patch+json' } }
      );
    } catch (err) {
      console.error('Failed to update status:', err.message);
    }
  }
}

module.exports = { MicroserviceController };
```

---

## 3. Admission Webhooks

### 3.1 Validating Webhook

```javascript
// webhooks/validating-webhook.js
const express = require('express');
const https = require('https');
const fs = require('fs');

const app = express();
app.use(express.json());

// Validate Microservice resources
app.post('/validate', (req, res) => {
  const { request } = req.body;
  const { object } = request;

  const errors = [];

  // Validate image tag (no 'latest' in production)
  if (request.namespace === 'production') {
    if (object.spec.image.endsWith(':latest')) {
      errors.push('Image must use specific tag, not "latest" in production');
    }

    // Require resource limits
    if (!object.spec.resources?.cpu || !object.spec.resources?.memory) {
      errors.push('Resource requests/limits required in production');
    }

    // Minimum replicas in production
    if ((object.spec.replicas || 2) < 2) {
      errors.push('Minimum 2 replicas required in production');
    }
  }

  if (errors.length > 0) {
    return res.json({
      apiVersion: 'admission.k8s.io/v1',
      kind: 'AdmissionReview',
      response: {
        uid: request.uid,
        allowed: false,
        status: {
          code: 422,
          message: errors.join('; ')
        }
      }
    });
  }

  res.json({
    apiVersion: 'admission.k8s.io/v1',
    kind: 'AdmissionReview',
    response: {
      uid: request.uid,
      allowed: true
    }
  });
});

// Mutating webhook: inject default labels and annotations
app.post('/mutate', (req, res) => {
  const { request } = req.body;
  const { object } = request;

  const patches = [];

  // Add managed-by label
  if (!object.metadata.labels?.['app.kubernetes.io/managed-by']) {
    patches.push({
      op: 'add',
      path: '/metadata/labels/app.kubernetes.io~1managed-by',
      value: 'microservice-operator'
    });
  }

  // Add default monitoring annotation
  if (!object.spec.monitoring) {
    patches.push({
      op: 'add',
      path: '/spec/monitoring',
      value: { enabled: true, scrapeInterval: '15s' }
    });
  }

  res.json({
    apiVersion: 'admission.k8s.io/v1',
    kind: 'AdmissionReview',
    response: {
      uid: request.uid,
      allowed: true,
      patchType: 'JSONPatch',
      patch: Buffer.from(JSON.stringify(patches)).toString('base64')
    }
  });
});

// Start HTTPS server (webhooks require HTTPS)
const server = https.createServer({
  key: fs.readFileSync('/certs/tls.key'),
  cert: fs.readFileSync('/certs/tls.crt')
}, app);

server.listen(8443, () => {
  console.log('Webhook server listening on :8443');
});
```

### 3.2 Webhook Configuration

```yaml
# webhook-config.yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: microservice-validator
  annotations:
    cert-manager.io/inject-ca-from: platform-system/webhook-cert
spec:
  webhooks:
    - name: validate.microservices.platform.company.com
      rules:
        - apiGroups: ['platform.company.com']
          apiVersions: ['v1']
          operations: ['CREATE', 'UPDATE']
          resources: ['microservices']
      clientConfig:
        service:
          name: microservice-operator-webhook
          namespace: platform-system
          path: /validate
      admissionReviewVersions: ['v1']
      sideEffects: None
      failurePolicy: Fail

---
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: microservice-mutator
spec:
  webhooks:
    - name: mutate.microservices.platform.company.com
      rules:
        - apiGroups: ['platform.company.com']
          apiVersions: ['v1']
          operations: ['CREATE']
          resources: ['microservices']
      clientConfig:
        service:
          name: microservice-operator-webhook
          namespace: platform-system
          path: /mutate
      admissionReviewVersions: ['v1']
      sideEffects: None
      failurePolicy: Ignore
```

---

## 4. Kubectl Plugin

```bash
#!/usr/bin/env bash
# kubectl-ms - Plugin สำหรับ Microservice CRD
# ติดตั้ง: chmod +x kubectl-ms && sudo mv kubectl-ms /usr/local/bin/

set -euo pipefail

COMMAND=${1:-help}
shift || true

case "$COMMAND" in
  list|ls)
    NAMESPACE=${1:-default}
    kubectl get microservices -n "$NAMESPACE" \
      -o custom-columns=\
"NAME:.metadata.name,\
IMAGE:.spec.image,\
REPLICAS:.spec.replicas,\
STATUS:.status.phase,\
AGE:.metadata.creationTimestamp"
    ;;

  describe|desc)
    NAME=$1
    NAMESPACE=${2:-default}
    kubectl describe microservice "$NAME" -n "$NAMESPACE"
    kubectl get deploy,svc,hpa,ingress \
      -l "app.kubernetes.io/name=$NAME" -n "$NAMESPACE"
    ;;

  logs)
    NAME=$1
    NAMESPACE=${2:-default}
    kubectl logs -l "app.kubernetes.io/name=$NAME" \
      -n "$NAMESPACE" --tail=100 --follow
    ;;

  status)
    NAMESPACE=${1:-default}
    echo "=== Microservices Status ==="
    kubectl get microservices -n "$NAMESPACE" \
      -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\t"}{.status.readyReplicas}/{.spec.replicas}{"\n"}{end}'
    ;;

  scale)
    NAME=$1
    REPLICAS=$2
    NAMESPACE=${3:-default}
    kubectl patch microservice "$NAME" -n "$NAMESPACE" \
      --type merge -p "{\"spec\":{\"replicas\":$REPLICAS}}"
    echo "Scaled $NAME to $REPLICAS replicas"
    ;;

  rollout)
    NAME=$1
    NAMESPACE=${2:-default}
    kubectl rollout status deployment/"$NAME" -n "$NAMESPACE"
    ;;

  help|*)
    echo "kubectl ms - Microservice management plugin"
    echo ""
    echo "Usage:"
    echo "  kubectl ms list [namespace]          - List all microservices"
    echo "  kubectl ms describe NAME [namespace] - Describe a microservice"
    echo "  kubectl ms logs NAME [namespace]     - Stream logs"
    echo "  kubectl ms status [namespace]        - Show status summary"
    echo "  kubectl ms scale NAME REPLICAS       - Scale a service"
    echo "  kubectl ms rollout NAME [namespace]  - Check rollout status"
    ;;
esac
```

---

## 5. Platform Team Best Practices

### 5.1 Internal Developer Platform (IDP) Workflow

```yaml
# ก่อน: Developer ต้องเขียน YAML หลายไฟล์
# หลัง: Developer เขียนแค่ Microservice CR

# Developer สร้าง microservice ใหม่:
apiVersion: platform.company.com/v1
kind: Microservice
metadata:
  name: payment-service
  namespace: production
spec:
  image: company/payment-service:1.0.0
  port: 3005
  replicas: 3
  autoscaling:
    enabled: true
    maxReplicas: 15
  ingress:
    enabled: true
    host: api.company.com
    path: /payments

# Operator สร้าง:
# ✓ Deployment (ด้วย security contexts, probes)
# ✓ Service (ClusterIP)
# ✓ HPA (CPU + Memory)
# ✓ Ingress (with TLS from cert-manager)
# ✓ ServiceMonitor (for Prometheus)
# ✓ PodDisruptionBudget
# ✓ NetworkPolicy
```

### 5.2 GitOps Integration

```yaml
# .github/workflows/deploy-service.yml
name: Deploy Microservice

on:
  workflow_dispatch:
    inputs:
      service:
        description: 'Service name'
        required: true
      version:
        description: 'Image version'
        required: true
      namespace:
        description: 'Target namespace'
        default: 'staging'

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          repository: company/gitops-repo
          token: ${{ secrets.GITOPS_TOKEN }}
      
      - name: Update Microservice version
        run: |
          SERVICE="${{ inputs.service }}"
          VERSION="${{ inputs.version }}"
          NAMESPACE="${{ inputs.namespace }}"
          
          cat > "services/${SERVICE}/overlays/${NAMESPACE}/patch.yaml" << EOF
          apiVersion: platform.company.com/v1
          kind: Microservice
          metadata:
            name: ${SERVICE}
          spec:
            image: company/${SERVICE}:${VERSION}
          EOF
      
      - name: Commit and push
        run: |
          git config user.name "Deployment Bot"
          git config user.email "deploy@company.com"
          git add .
          git commit -m "deploy: ${SERVICE} → ${VERSION} on ${NAMESPACE}"
          git push
```

---

## สรุป

| Concept | Purpose |
|---------|---------|
| **CRD** | Define custom API resources in Kubernetes |
| **Operator** | Automate complex Day-2 operations |
| **Controller** | Reconciliation loop: desired state → actual state |
| **Admission Webhook** | Validate/mutate resources before storage |
| **kubectl Plugin** | Custom CLI commands for your platform |

**Operator Pattern = Software Engineering + Operations Knowledge**

Platform teams ใช้ Operators เพื่อ:
- ลด YAML complexity สำหรับ developers
- Enforce organizational policies automatically
- Automate Day-2 operations (backup, scaling, upgrades)

**Next:** Part 32 - Service Mesh Advanced: Traffic Management and Progressive Delivery

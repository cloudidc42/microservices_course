# Part 97: Open Source Microservices Tools

## บทนำ

Ecosystem ของ Open Source Tools สำหรับ Microservices มีมากมาย บทนี้จะแนะนำเครื่องมือที่สำคัญและใช้จริงใน Production ครอบคลุมตั้งแต่ Dapr, Temporal, OpenFaaS, Knative ไปจนถึง Tilt สำหรับ Local Development

---

## 1. Dapr - Distributed Application Runtime

### 1.1 ทำไมต้องใช้ Dapr?

```
ปัญหาที่ Dapr แก้:
┌──────────────────────────────────────────────────────────────────┐
│                    Without Dapr                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Order Service ต้องรู้จัก:                                │    │
│  │ - Kafka SDK + configuration                             │    │
│  │ - Redis SDK + connection pooling                        │    │
│  │ - Service Discovery (Consul/Kubernetes)                  │    │
│  │ - Circuit Breaker library                               │    │
│  │ - Distributed Tracing setup                             │    │
│  │ ═══ ทั้งหมดนี้ต้องเขียนเองทุก Service!                  │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                    │
│                    With Dapr                                       │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Order Service ต้องรู้แค่:                                │    │
│  │ - HTTP/gRPC calls to Dapr sidecar                       │    │
│  │ - Component names (not implementations)                  │    │
│  │ ═══ Dapr จัดการทุกอย่าง!                                │    │
│  └─────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────┘
```

### 1.2 Dapr Installation & Configuration

```bash
# ติดตั้ง Dapr CLI
wget -q https://raw.githubusercontent.com/dapr/cli/master/install/install.sh -O - | /bin/bash

# Initialize Dapr (สำหรับ Kubernetes)
dapr init --kubernetes --wait

# Verify
kubectl get pods --namespace dapr-system

# Expected:
# dapr-dashboard-xxx       Running
# dapr-operator-xxx        Running
# dapr-placement-server-0  Running
# dapr-sentry-xxx          Running
# dapr-sidecar-injector-xx Running
```

```yaml
# dapr/components/redis-state.yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
  namespace: production
spec:
  type: state.redis
  version: v1
  metadata:
  - name: redisHost
    value: "redis-master.production.svc.cluster.local:6379"
  - name: redisPassword
    secretKeyRef:
      name: redis-secret
      key: password
  - name: actorStateStore
    value: "true"
  - name: keyPrefix
    value: "appid"
  - name: enableTLS
    value: "false"
  auth:
    secretStore: kubernetes

---
# dapr/components/kafka-pubsub.yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: pubsub
  namespace: production
spec:
  type: pubsub.kafka
  version: v1
  metadata:
  - name: brokers
    value: "kafka-0.kafka.production.svc.cluster.local:9092"
  - name: consumerGroup
    value: "dapr-consumer-group"
  - name: authRequired
    value: "false"
  - name: maxMessageBytes
    value: "1024000"
  - name: consumeRetryInterval
    value: "200ms"
```

### 1.3 Dapr Workflow (Saga Pattern)

```go
// order-workflow/workflow.go
package main

import (
    "context"
    "fmt"
    "time"
    
    "github.com/dapr/durabletask-go/task"
    "github.com/dapr/go-sdk/workflow"
)

// Dapr Workflow แทน Saga Orchestrator

func orderWorkflow(ctx *workflow.WorkflowContext) (any, error) {
    var req CreateOrderRequest
    if err := ctx.GetInput(&req); err != nil {
        return nil, err
    }

    // Step 1: Reserve Inventory
    var reservationID string
    err := ctx.CallActivity(reserveInventoryActivity, 
        workflow.ActivityInput(req.Items)).Await(&reservationID)
    if err != nil {
        return nil, fmt.Errorf("inventory reservation failed: %w", err)
    }

    // Step 2: Process Payment
    var paymentID string
    err = ctx.CallActivity(processPaymentActivity,
        workflow.ActivityInput(PaymentInput{
            Amount:    req.Total,
            Method:    req.PaymentMethod,
        })).Await(&paymentID)
    if err != nil {
        // Compensate: Release inventory
        ctx.CallActivity(releaseInventoryActivity,
            workflow.ActivityInput(reservationID))
        return nil, fmt.Errorf("payment failed: %w", err)
    }

    // Step 3: Create Order
    var orderID string
    err = ctx.CallActivity(createOrderActivity,
        workflow.ActivityInput(OrderInput{
            ReservationID: reservationID,
            PaymentID:     paymentID,
            Items:         req.Items,
        })).Await(&orderID)
    if err != nil {
        // Compensate: Refund + Release
        ctx.CallActivity(refundPaymentActivity,
            workflow.ActivityInput(paymentID))
        ctx.CallActivity(releaseInventoryActivity,
            workflow.ActivityInput(reservationID))
        return nil, err
    }

    // Step 4: Send notification (fire-and-forget)
    ctx.CallActivity(sendNotificationActivity,
        workflow.ActivityInput(NotificationInput{
            UserID:  req.UserID,
            OrderID: orderID,
        }))

    return OrderResult{OrderID: orderID, Status: "confirmed"}, nil
}

func reserveInventoryActivity(ctx context.Context, input []OrderItem) (string, error) {
    // Call inventory service via Dapr service invocation
    client, _ := dapr.NewClient()
    defer client.Close()
    
    data, _ := json.Marshal(input)
    resp, err := client.InvokeMethodWithContent(ctx,
        "inventory-service", "reserve", "POST",
        &dapr.DataContent{ContentType: "application/json", Data: data})
    
    if err != nil {
        return "", err
    }
    
    var result ReservationResult
    json.Unmarshal(resp, &result)
    return result.ReservationID, nil
}

// Register workflow
func main() {
    w, err := workflow.NewWorker()
    if err != nil {
        log.Fatal(err)
    }
    
    w.RegisterWorkflow(orderWorkflow)
    w.RegisterActivity(reserveInventoryActivity)
    w.RegisterActivity(processPaymentActivity)
    w.RegisterActivity(createOrderActivity)
    w.RegisterActivity(sendNotificationActivity)
    w.RegisterActivity(releaseInventoryActivity)
    w.RegisterActivity(refundPaymentActivity)
    
    if err := w.Start(); err != nil {
        log.Fatal(err)
    }
}
```

---

## 2. Temporal - Workflow Engine

### 2.1 Temporal vs Dapr Workflow

```
Temporal ดีกว่า Dapr Workflow เมื่อ:
- ต้องการ Workflow ที่ซับซ้อนมากๆ
- Long-running workflows (days, weeks, months)
- Fine-grained visibility ทุก step
- Deterministic replay (เหมือน Event Sourcing)
- Large-scale (Uber ใช้ process 1B+ workflows/day)

Dapr Workflow ดีกว่าเมื่อ:
- ใช้ Dapr อยู่แล้ว
- ต้องการ simplicity
- Short-lived workflows
```

### 2.2 Temporal Setup

```yaml
# docker-compose.temporal.yml
version: "3.9"
services:
  temporal:
    image: temporalio/auto-setup:1.22
    ports:
      - "7233:7233"
    environment:
      - DB=postgresql
      - DB_PORT=5432
      - POSTGRES_USER=temporal
      - POSTGRES_PWD=temporal
      - POSTGRES_SEEDS=postgres
    depends_on:
      - postgres
  
  temporal-web:
    image: temporalio/ui:2.20
    ports:
      - "8088:8080"
    environment:
      - TEMPORAL_ADDRESS=temporal:7233
  
  postgres:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: temporal
      POSTGRES_USER: temporal
```

### 2.3 Temporal Workflow Implementation

```go
// temporal-workflows/order_workflow.go
package workflows

import (
    "time"
    
    "go.temporal.io/sdk/temporal"
    "go.temporal.io/sdk/workflow"
    "go.temporal.io/sdk/activity"
)

// Workflow Definition
func OrderFulfillmentWorkflow(ctx workflow.Context, req OrderRequest) (OrderResult, error) {
    logger := workflow.GetLogger(ctx)
    
    // Retry policy สำหรับ Activities
    retryPolicy := &temporal.RetryPolicy{
        MaximumAttempts:    3,
        InitialInterval:    time.Second,
        MaximumInterval:    10 * time.Second,
        BackoffCoefficient: 2.0,
    }
    
    activityOpts := workflow.ActivityOptions{
        StartToCloseTimeout: 30 * time.Second,
        RetryPolicy:         retryPolicy,
    }
    ctx = workflow.WithActivityOptions(ctx, activityOpts)
    
    // Step 1: Validate Order
    var validationResult ValidationResult
    if err := workflow.ExecuteActivity(ctx, ValidateOrderActivity, req).Get(ctx, &validationResult); err != nil {
        return OrderResult{}, err
    }
    
    // Step 2: Reserve Inventory
    var reservation InventoryReservation
    if err := workflow.ExecuteActivity(ctx, ReserveInventoryActivity, req.Items).Get(ctx, &reservation); err != nil {
        return OrderResult{}, err
    }
    
    // Step 3: Charge Payment
    var payment PaymentResult
    if err := workflow.ExecuteActivity(ctx, ChargePaymentActivity, ChargeRequest{
        OrderID: req.OrderID,
        Amount:  req.Total,
        Method:  req.PaymentMethod,
    }).Get(ctx, &payment); err != nil {
        // Compensation
        compensationCtx := workflow.WithActivityOptions(ctx, workflow.ActivityOptions{
            StartToCloseTimeout: 10 * time.Second,
            RetryPolicy: &temporal.RetryPolicy{MaximumAttempts: 5},
        })
        workflow.ExecuteActivity(compensationCtx, ReleaseInventoryActivity, reservation.ID).Get(ctx, nil)
        return OrderResult{}, err
    }
    
    // Step 4: Wait for Restaurant Confirmation (External Signal)
    restaurantSig := workflow.GetSignalChannel(ctx, "restaurant-confirmation")
    
    var restaurantResponse RestaurantResponse
    
    // Timeout หากร้านไม่ยืนยันใน 5 นาที
    timerCtx, cancel := workflow.WithCancel(ctx)
    defer cancel()
    
    timer := workflow.NewTimer(timerCtx, 5*time.Minute)
    
    selector := workflow.NewSelector(ctx)
    selector.AddReceive(restaurantSig, func(c workflow.ReceiveChannel, more bool) {
        c.Receive(ctx, &restaurantResponse)
        cancel()
    })
    selector.AddFuture(timer, func(f workflow.Future) {
        restaurantResponse = RestaurantResponse{Accepted: false, Reason: "timeout"}
    })
    selector.Select(ctx)
    
    if !restaurantResponse.Accepted {
        // Refund if restaurant doesn't accept
        refundCtx := workflow.WithActivityOptions(ctx, workflow.ActivityOptions{
            StartToCloseTimeout: 30 * time.Second,
        })
        workflow.ExecuteActivity(refundCtx, RefundPaymentActivity, payment.PaymentID).Get(ctx, nil)
        return OrderResult{}, temporal.NewApplicationError("restaurant declined", "RESTAURANT_DECLINED")
    }
    
    // Step 5: Assign Driver (Async - don't wait)
    workflow.Go(ctx, func(ctx workflow.Context) {
        workflow.ExecuteActivity(ctx, AssignDriverActivity, req.OrderID).Get(ctx, nil)
    })
    
    // Step 6: Update order status
    workflow.ExecuteActivity(ctx, UpdateOrderStatusActivity, OrderStatusUpdate{
        OrderID: req.OrderID,
        Status:  "confirmed",
    }).Get(ctx, nil)
    
    logger.Info("Order workflow completed", "orderID", req.OrderID)
    
    return OrderResult{
        OrderID:   req.OrderID,
        Status:    "confirmed",
        PaymentID: payment.PaymentID,
    }, nil
}

// Activities
func ValidateOrderActivity(ctx context.Context, req OrderRequest) (ValidationResult, error) {
    // Validate restaurant is open, items exist, etc.
    return ValidationResult{Valid: true}, nil
}

func ReserveInventoryActivity(ctx context.Context, items []OrderItem) (InventoryReservation, error) {
    info := activity.GetInfo(ctx)
    logger := activity.GetLogger(ctx)
    
    logger.Info("Reserving inventory", "attempt", info.Attempt)
    
    // Call inventory service
    client := getInventoryClient()
    return client.Reserve(ctx, items)
}
```

### 2.4 Running Temporal Worker

```go
// temporal-worker/main.go
package main

import (
    "log"
    
    "go.temporal.io/sdk/client"
    "go.temporal.io/sdk/worker"
    
    "github.com/company/workflows"
    "github.com/company/activities"
)

func main() {
    c, err := client.Dial(client.Options{
        HostPort:  "temporal:7233",
        Namespace: "production",
    })
    if err != nil {
        log.Fatalf("Unable to create client: %v", err)
    }
    defer c.Close()
    
    w := worker.New(c, "order-task-queue", worker.Options{
        MaxConcurrentActivityExecutionSize: 100,
        MaxConcurrentWorkflowTaskExecutionSize: 50,
    })
    
    // Register workflows and activities
    w.RegisterWorkflow(workflows.OrderFulfillmentWorkflow)
    w.RegisterActivity(activities.ValidateOrderActivity)
    w.RegisterActivity(activities.ReserveInventoryActivity)
    w.RegisterActivity(activities.ChargePaymentActivity)
    w.RegisterActivity(activities.AssignDriverActivity)
    w.RegisterActivity(activities.UpdateOrderStatusActivity)
    w.RegisterActivity(activities.ReleaseInventoryActivity)
    w.RegisterActivity(activities.RefundPaymentActivity)
    
    if err := w.Run(worker.InterruptCh()); err != nil {
        log.Fatalf("Unable to start worker: %v", err)
    }
}
```

---

## 3. OpenFaaS - Functions as a Service

### 3.1 OpenFaaS Overview

```
OpenFaaS = Open Source Serverless สำหรับ Kubernetes
ใช้ได้กับ: AWS, GCP, Azure, On-premise, Raspberry Pi!

Architecture:
┌─────────────────────────────────────────────────────────────┐
│                        OpenFaaS                             │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  API Gateway / Function Router                          │ │
│  │  faas.company.com/function/resize-image                │ │
│  └──────────────────────┬─────────────────────────────────┘ │
│                          │                                   │
│  ┌───────────────────────▼─────────────────────────────┐   │
│  │              Function Pods                           │   │
│  │                                                      │   │
│  │  ┌──────────────────┐  ┌──────────────────────────┐ │   │
│  │  │ resize-image     │  │ send-email              │ │   │
│  │  │ (Node.js)        │  │ (Python)                │ │   │
│  │  │ Scales 0 → 100   │  │ Scales 0 → 50           │ │   │
│  │  └──────────────────┘  └──────────────────────────┘ │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  Queue Worker (NATS Streaming) for async functions          │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 OpenFaaS Function Examples

```bash
# ติดตั้ง faas-cli
curl -sSL https://cli.openfaas.com | sudo sh

# Deploy OpenFaaS บน Kubernetes
helm repo add openfaas https://openfaas.github.io/faas-netes/
helm upgrade --install openfaas openfaas/openfaas \
  --namespace openfaas \
  --set functionNamespace=openfaas-fn \
  --set generateBasicAuth=true \
  --set autoscaler.enabled=true \
  --set autoscaler.image=ghcr.io/openfaas/autoscaler:0.1.3 \
  --set operator.create=true

# Create function from template
faas-cli new --lang go resize-image
faas-cli new --lang python3 send-notification
faas-cli new --lang node18 pdf-generator
```

```go
// functions/resize-image/handler.go
package function

import (
    "bytes"
    "encoding/base64"
    "fmt"
    "image"
    "image/jpeg"
    "image/png"
    "net/http"
    "strconv"
    
    "golang.org/x/image/draw"
)

type ResizeRequest struct {
    ImageBase64 string `json:"image"`
    Width       int    `json:"width"`
    Height      int    `json:"height"`
    Format      string `json:"format"` // jpeg, png, webp
}

type ResizeResponse struct {
    ImageBase64 string `json:"image"`
    Width       int    `json:"width"`
    Height      int    `json:"height"`
    SizeBytes   int    `json:"size_bytes"`
}

// Handle runs for each invocation
func Handle(w http.ResponseWriter, r *http.Request) {
    var req ResizeRequest
    json.NewDecoder(r.Body).Decode(&req)
    
    // Decode base64 image
    imgData, err := base64.StdEncoding.DecodeString(req.ImageBase64)
    if err != nil {
        http.Error(w, "Invalid image data", http.StatusBadRequest)
        return
    }
    
    // Decode image
    src, _, err := image.Decode(bytes.NewReader(imgData))
    if err != nil {
        http.Error(w, "Cannot decode image", http.StatusBadRequest)
        return
    }
    
    // Resize
    dst := image.NewRGBA(image.Rect(0, 0, req.Width, req.Height))
    draw.BiLinear.Scale(dst, dst.Bounds(), src, src.Bounds(), draw.Over, nil)
    
    // Encode
    var buf bytes.Buffer
    switch req.Format {
    case "png":
        png.Encode(&buf, dst)
    default:
        jpeg.Encode(&buf, dst, &jpeg.Options{Quality: 85})
    }
    
    result := ResizeResponse{
        ImageBase64: base64.StdEncoding.EncodeToString(buf.Bytes()),
        Width:       req.Width,
        Height:      req.Height,
        SizeBytes:   buf.Len(),
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(result)
}
```

```yaml
# functions/stack.yaml
version: 1.0
provider:
  name: openfaas
  gateway: https://openfaas.company.com

functions:
  resize-image:
    lang: go
    handler: ./resize-image
    image: registry.company.com/resize-image:latest
    labels:
      com.openfaas.scale.min: "0"
      com.openfaas.scale.max: "50"
      com.openfaas.scale.factor: "5"
      com.openfaas.scale.target: "5"  # concurrent requests per pod
    environment:
      MAX_SIZE: "10485760"  # 10MB
    limits:
      memory: 128m
      cpu: "500m"
    requests:
      memory: 64m
      cpu: "100m"
    
  pdf-generator:
    lang: node18
    handler: ./pdf-generator
    image: registry.company.com/pdf-generator:latest
    labels:
      com.openfaas.scale.min: "1"    # ไม่ Scale to zero (preload)
      com.openfaas.scale.max: "20"
    environment:
      CHROME_FLAGS: "--headless --no-sandbox"
    limits:
      memory: 512m
      cpu: "1000m"
    annotations:
      topic: "pdf.generate"          # Queue-based trigger
```

---

## 4. Knative - Kubernetes-native Serverless

### 4.1 Knative Serving

```bash
# ติดตั้ง Knative
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.12.0/serving-crds.yaml
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.12.0/serving-core.yaml

# ติดตั้ง Kourier (networking layer)
kubectl apply -f https://github.com/knative/net-kourier/releases/download/knative-v1.12.0/kourier.yaml
kubectl patch configmap/config-network \
  --namespace knative-serving \
  --type merge \
  --patch '{"data":{"ingress-class":"kourier.ingress.networking.knative.dev"}}'
```

```yaml
# knative/email-service.yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: email-service
  namespace: production
  annotations:
    serving.knative.dev/visibility: cluster-local
spec:
  template:
    metadata:
      annotations:
        # Scale to zero after 60 seconds of inactivity
        autoscaling.knative.dev/window: "60s"
        autoscaling.knative.dev/target: "100"  # 100 req/pod
        autoscaling.knative.dev/minScale: "0"
        autoscaling.knative.dev/maxScale: "50"
        # KEDA-style panic threshold (scale fast when traffic spikes)
        autoscaling.knative.dev/panicThresholdPercentage: "200.0"
        autoscaling.knative.dev/panicWindowPercentage: "10.0"
    spec:
      containers:
      - image: email-service:v1.2
        env:
        - name: SMTP_HOST
          valueFrom:
            secretKeyRef:
              name: smtp-secret
              key: host
        - name: SES_REGION
          value: ap-southeast-1
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "256Mi"
            cpu: "500m"

---
# Knative Eventing: Trigger email on order.confirmed
apiVersion: eventing.knative.dev/v1
kind: Trigger
metadata:
  name: order-email-trigger
  namespace: production
spec:
  broker: default
  filter:
    attributes:
      type: order.confirmed
      source: order-service
  subscriber:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: email-service
    uri: /send-confirmation
```

### 4.2 Knative Traffic Splitting (Canary)

```yaml
# Blue-Green / Canary deployments with Knative
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: product-service
spec:
  traffic:
  - tag: current
    revisionName: product-service-v1  
    percent: 90    # 90% traffic → stable version
  - tag: candidate
    revisionName: product-service-v2
    percent: 10    # 10% traffic → new version (canary)
  - tag: latest
    latestRevision: true
    percent: 0     # Ready but no traffic yet
```

---

## 5. Tilt - Local Development Tool

### 5.1 Tilt Overview

```
Tilt แก้ปัญหา Inner Dev Loop ใน Microservices:

ปัญหาเดิม (โดย dev):
1. แก้โค้ด
2. docker build (2 นาที)
3. docker push (30 วินาที)
4. kubectl apply (10 วินาที)
5. รอ pod restart (20 วินาที)
6. ทดสอบ
= ใช้เวลา ~3 นาที ต่อ 1 change!

ด้วย Tilt:
1. แก้โค้ด
2. Tilt detect change → rebuild → deploy
= ใช้เวลา ~10 วินาที! (fast feedback loop)

Features:
- Live reload สำหรับ code changes
- Unified log view จากทุก services
- Web UI แสดงสถานะทุก service
- Port forwarding อัตโนมัติ
- Smart rebuild (rebuild เฉพาะที่เปลี่ยน)
```

### 5.2 Tiltfile Configuration

```python
# Tiltfile (Python-like syntax)

# Global settings
allow_k8s_contexts(['docker-desktop', 'minikube', 'kind-dev'])
update_settings(max_parallel_updates=6)

# Database dependencies (start first)
k8s_yaml('k8s/local/postgres.yaml')
k8s_yaml('k8s/local/redis.yaml')
k8s_yaml('k8s/local/kafka.yaml')

# Function: Build and deploy a Go service
def go_service(name, port, deps=[]):
    # Build with hot reload
    docker_build(
        f'registry.local/{name}',
        context='.',
        dockerfile=f'services/{name}/Dockerfile',
        only=[f'services/{name}', 'shared/'],
        live_update=[
            # Sync source files (no rebuild)
            sync(f'services/{name}', f'/app/services/{name}'),
            # Run go build in container
            run('cd /app && go build -o /bin/service ./services/' + name + '/...'),
            # Restart the process
            restart_process('/bin/service'),
        ],
    )
    
    # Deploy to Kubernetes
    k8s_yaml(f'k8s/services/{name}.yaml')
    
    # Configure port forwarding
    k8s_resource(
        name,
        port_forwards=[f'{port}:{port}'],
        resource_deps=deps,
        labels=[name],
    )

# Deploy services
go_service('user-service', 8001, deps=['postgres', 'redis'])
go_service('product-service', 8002, deps=['postgres', 'redis', 'elasticsearch'])
go_service('order-service', 8003, deps=['postgres', 'kafka', 'user-service', 'product-service'])
go_service('payment-service', 8004, deps=['postgres', 'order-service'])
go_service('notification-service', 8007, deps=['kafka', 'redis'])

# Python service (different build)
docker_build(
    'registry.local/search-service',
    context='services/search-service',
    live_update=[
        sync('services/search-service', '/app'),
        run('pip install -r /app/requirements.txt', trigger='services/search-service/requirements.txt'),
        restart_process('uvicorn app.main:app --reload'),
    ],
)

k8s_yaml('k8s/services/search-service.yaml')
k8s_resource('search-service', port_forwards=['8006:8006'], 
             resource_deps=['elasticsearch'])

# Frontend
docker_build(
    'registry.local/frontend',
    context='frontend',
    live_update=[
        sync('frontend/src', '/app/src'),
        run('npm run build', trigger='frontend/package.json'),
    ],
)

k8s_resource('frontend', port_forwards=['3000:80'])

# Run tests on file changes
local_resource(
    'unit-tests',
    cmd='cd services && go test ./... -short',
    deps=['services/'],
    auto_init=False,
    trigger_mode=TRIGGER_MODE_MANUAL,
    labels=['testing'],
)

local_resource(
    'integration-tests',
    cmd='./scripts/run-integration-tests.sh',
    resource_deps=['order-service', 'payment-service'],
    auto_init=False,
    trigger_mode=TRIGGER_MODE_MANUAL,
    labels=['testing'],
)
```

---

## 6. Additional Essential Tools

### 6.1 OpenTelemetry

```go
// shared/telemetry/setup.go
package telemetry

import (
    "context"
    
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
    "go.opentelemetry.io/otel/propagation"
    "go.opentelemetry.io/otel/sdk/resource"
    sdktrace "go.opentelemetry.io/otel/sdk/trace"
    semconv "go.opentelemetry.io/otel/semconv/v1.21.0"
)

func InitTracing(ctx context.Context, serviceName, serviceVersion string) (func(), error) {
    // Export to Jaeger/Tempo/Honeycomb via OTLP
    exporter, err := otlptracegrpc.New(ctx,
        otlptracegrpc.WithEndpoint("otel-collector:4317"),
        otlptracegrpc.WithInsecure(),
    )
    if err != nil {
        return nil, err
    }
    
    res, err := resource.New(ctx,
        resource.WithAttributes(
            semconv.ServiceName(serviceName),
            semconv.ServiceVersion(serviceVersion),
            semconv.DeploymentEnvironment("production"),
        ),
    )
    
    tp := sdktrace.NewTracerProvider(
        sdktrace.WithBatcher(exporter),
        sdktrace.WithResource(res),
        sdktrace.WithSampler(sdktrace.ParentBased(
            sdktrace.TraceIDRatioBased(0.1), // Sample 10% in production
        )),
    )
    
    otel.SetTracerProvider(tp)
    otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
        propagation.TraceContext{},
        propagation.Baggage{},
    ))
    
    return func() {
        tp.Shutdown(ctx)
    }, nil
}
```

### 6.2 Crossplane - Infrastructure as Code

```yaml
# crossplane/rds-claim.yaml
# สร้าง PostgreSQL RDS ผ่าน Kubernetes API!
apiVersion: database.example.com/v1alpha1
kind: PostgreSQLDatabase
metadata:
  name: order-service-db
  namespace: production
spec:
  parameters:
    version: "15"
    storageGB: 100
    instanceClass: db.t3.large
    multiAZ: true
    backupRetentionDays: 7
    region: ap-southeast-1
  compositionRef:
    name: postgresql-aws
  writeConnectionSecretToRef:
    name: order-service-db-credentials
```

### 6.3 Chaos Engineering with Chaos Mesh

```yaml
# chaos/pod-failure.yaml
# ทดสอบว่าระบบ resilient จริงไหม
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: order-service-pod-failure
  namespace: testing
spec:
  action: pod-failure
  mode: random-max-percent
  value: "30"    # Kill 30% of pods randomly
  duration: "5m"
  selector:
    namespaces:
    - production
    labelSelectors:
      app: order-service
  scheduler:
    cron: "@every 1h"  # Run every hour in staging

---
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: payment-service-latency
spec:
  action: delay
  mode: all
  selector:
    namespaces:
    - production
    labelSelectors:
      app: payment-service
  delay:
    latency: "2000ms"   # Inject 2 second latency
    correlation: "25"
    jitter: "500ms"
  direction: to
  duration: "10m"
```

---

## 7. Tools Comparison Matrix

```
เปรียบเทียบ Tools สำหรับ Use Cases ต่างๆ:

┌─────────────────────────────────────────────────────────────────────┐
│                     Use Case → Best Tool                            │
├──────────────────────────┬──────────────────────────────────────────┤
│ Use Case                 │ Recommended Tool(s)                      │
├──────────────────────────┼──────────────────────────────────────────┤
│ Saga/Workflow Orchestration│ Temporal (complex) / Dapr (simple)    │
├──────────────────────────┼──────────────────────────────────────────┤
│ Serverless Functions     │ Knative (K8s) / OpenFaaS (flexible)     │
├──────────────────────────┼──────────────────────────────────────────┤
│ Local Dev Loop           │ Tilt (best-in-class)                     │
├──────────────────────────┼──────────────────────────────────────────┤
│ Service Mesh             │ Istio (feature-rich) / Linkerd (simple) │
├──────────────────────────┼──────────────────────────────────────────┤
│ Observability (Metrics)  │ Prometheus + Grafana                    │
├──────────────────────────┼──────────────────────────────────────────┤
│ Distributed Tracing      │ Jaeger / Tempo (cheaper storage)        │
├──────────────────────────┼──────────────────────────────────────────┤
│ Log Aggregation          │ Loki (cheap) / Elasticsearch (powerful) │
├──────────────────────────┼──────────────────────────────────────────┤
│ GitOps Deployment        │ ArgoCD (UI/RBAC) / Flux (lightweight)   │
├──────────────────────────┼──────────────────────────────────────────┤
│ Secret Management        │ Vault (powerful) / ESO (K8s native)     │
├──────────────────────────┼──────────────────────────────────────────┤
│ Chaos Engineering        │ Chaos Mesh / Litmus                     │
├──────────────────────────┼──────────────────────────────────────────┤
│ Infrastructure as Code   │ Crossplane (K8s) / Terraform (standard) │
├──────────────────────────┼──────────────────────────────────────────┤
│ API Gateway              │ Kong (powerful) / Traefik (simple)      │
├──────────────────────────┼──────────────────────────────────────────┤
│ Message Queue            │ Kafka (throughput) / NATS (low latency) │
├──────────────────────────┼──────────────────────────────────────────┤
│ In-process Cache         │ Ristretto (Go) / Caffeine (Java)        │
└──────────────────────────┴──────────────────────────────────────────┘
```

---

## สรุป

Open Source Ecosystem สำหรับ Microservices ในปี 2024-2025 มีครบทุก Layer:

1. **Dapr** - Portable runtime abstraction สำหรับ messaging, state, workflows
2. **Temporal** - Enterprise-grade workflow engine สำหรับ complex long-running processes
3. **OpenFaaS** - Serverless ที่ยืดหยุ่น รันได้บน K8s ทุกที่
4. **Knative** - Serverless standard บน Kubernetes พร้อม Eventing
5. **Tilt** - Local development tool ที่ช่วยให้ feedback loop เร็วขึ้น 20x
6. **OpenTelemetry** - Standard observability framework
7. **Chaos Mesh** - Chaos engineering บน Kubernetes

การเลือก Tools ที่ดีควรพิจารณา:
- Team expertise
- Operational complexity
- Community support
- Long-term maintenance

> "Don't adopt a tool just because it's trendy. Adopt it because it solves a real problem you have."

---

*ถัดไป: Part 98 - Microservices Cost Calculator*

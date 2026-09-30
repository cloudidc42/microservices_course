# Part 96: Microservices Roadmap 2024-2025

## บทนำ

โลกของ Microservices กำลังเปลี่ยนแปลงอย่างรวดเร็ว บทนี้จะสำรวจ Emerging Trends, เทคโนโลยีใหม่ที่กำลังมา, และทิศทางที่นักพัฒนาต้องเตรียมตัว ตั้งแต่ eBPF, WebAssembly, Dapr, Next-gen Service Mesh, Serverless Microservices, AI-augmented Operations ไปจนถึง Green Computing

---

## 1. eBPF: The Future of Observability & Networking

### 1.1 eBPF คืออะไร?

```
eBPF (Extended Berkeley Packet Filter) คือ Technology ที่ให้เรา
Run Code ใน Linux Kernel โดยไม่ต้องแก้ Kernel Source Code

Traditional Approach:        eBPF Approach:
┌────────────────────┐       ┌────────────────────────────────┐
│   User Space       │       │   User Space                   │
│  ┌──────────────┐  │       │  ┌──────────────┐              │
│  │ Application  │  │       │  │ eBPF Programs│              │
│  └──────────────┘  │       │  └──────────┬───┘              │
│  ┌──────────────┐  │       │             │ Load & Verify     │
│  │ Kernel Module│  │       └─────────────┼──────────────────┘
│  │ (risky!)     │  │       ┌─────────────▼──────────────────┐
│  └──────────────┘  │       │   Kernel Space                 │
│         ↕          │       │  ┌──────────────────────────┐  │
│  Kernel Space      │       │  │ eBPF JIT-compiled code   │  │
└────────────────────┘       │  │ (safe sandbox)            │  │
                              │  └──────────────────────────┘  │
                              └────────────────────────────────┘

ความสามารถ:
- Zero-overhead observability
- Network policy enforcement ใน kernel
- Performance profiling
- Security monitoring (รู้ทันที มีใครพยายาม escape container)
```

### 1.2 eBPF Use Cases ใน Microservices

```
1. Network Observability (Cilium Hubble):
   - ดู L3/L4/L7 traffic ระหว่าง Services โดยไม่ต้องแก้ Application code
   - Network policy enforcement
   - Load balancing ใน kernel (เร็วกว่า iptables 100x)

2. Distributed Tracing ไม่ต้อง Instrument:
   - Pixie (ของ New Relic): Auto-instruments requests
   - Pyroscope: Continuous profiling

3. Security:
   - Tetragon (Cilium): Runtime security
   - Falco: Container runtime security
   - ตรวจจับ Privilege escalation ได้ทันที

ตัวอย่าง: Cilium บน Kubernetes
```

### 1.3 Cilium Setup

```yaml
# k8s/cilium-values.yaml
# ติดตั้ง Cilium แทน kube-proxy
cilium:
  kubeProxyReplacement: "strict"
  hubble:
    enabled: true
    relay:
      enabled: true
    ui:
      enabled: true
  loadBalancer:
    algorithm: "maglev"  # Better than round-robin
  bpf:
    masquerade: true
  
  # Network Policy เพิ่ม L7 (HTTP)
  # ตัวอย่าง: Order Service ได้รับแค่ GET/POST เท่านั้น
```

```yaml
# Network Policy ระดับ L7 ด้วย Cilium
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: order-service-policy
spec:
  endpointSelector:
    matchLabels:
      app: order-service
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: api-gateway
    toPorts:
    - ports:
      - port: "8080"
        protocol: TCP
      rules:
        http:
        - method: "GET"
          path: "/api/v1/orders/.*"
        - method: "POST"
          path: "/api/v1/orders"
        # Block DELETE explicitly
```

---

## 2. WebAssembly (Wasm) ใน Microservices

### 2.1 Wasm Edge Computing

```
WebAssembly กำลังเปลี่ยนจาก "Browser only" เป็น "Everywhere"

Traditional Container:         Wasm Module:
┌─────────────────────┐        ┌──────────────────────────┐
│  Container Image    │        │  Wasm Binary (.wasm)     │
│  ┌───────────────┐  │        │  ┌──────────────────┐    │
│  │ OS Libs (200MB│  │        │  │  Pure Business   │    │
│  │ Runtime       │  │        │  │  Logic (100KB)   │    │
│  │ App Binary    │  │        │  └──────────────────┘    │
│  └───────────────┘  │        │  + WASI (System calls)   │
│  Start time: 100ms+ │        │  Start time: < 1ms!      │
└─────────────────────┘        └──────────────────────────┘

Use Cases:
1. Edge Functions (Cloudflare Workers, Fastly Compute)
2. Plugin Systems (eBPF alternative)
3. Serverless ที่เริ่ม Fast กว่า Container
4. Cross-platform Business Logic (same code, any platform)
```

### 2.2 Wasm Serverless Function

```rust
// edge-functions/src/rate_limiter.rs
// Runs on Cloudflare Workers as Wasm

use worker::*;
use std::collections::HashMap;

#[event(fetch)]
pub async fn main(req: Request, env: Env, _ctx: Context) -> Result<Response> {
    let url = req.url()?;
    let path = url.path();
    
    // Get client IP
    let ip = req.headers().get("CF-Connecting-IP")?
        .unwrap_or("unknown".to_string());
    
    // Rate limit check using Cloudflare KV
    let kv = env.kv("RATE_LIMITS")?;
    let key = format!("{}:{}", ip, &path[..path.find('/').unwrap_or(path.len())]);
    
    let count: u32 = kv.get(&key)
        .text()
        .await?
        .and_then(|v| v.parse().ok())
        .unwrap_or(0);
    
    if count >= 100 {
        return Response::error("Rate limit exceeded", 429);
    }
    
    // Increment counter
    kv.put(&key, (count + 1).to_string())?
        .expiration_ttl(60) // Reset every minute
        .execute()
        .await?;
    
    // Forward to origin
    let mut headers = Headers::new();
    headers.set("X-RateLimit-Remaining", &(100 - count - 1).to_string())?;
    
    let mut init = RequestInit::new();
    init.with_headers(headers);
    
    Fetch::Request(Request::new_with_init(
        &format!("https://origin.example.com{}", path),
        &init
    )?)
    .send()
    .await
}
```

---

## 3. Dapr (Distributed Application Runtime)

### 3.1 Dapr Overview

```
Dapr คือ Portable, Event-driven Runtime สำหรับ Microservices
ที่แก้ปัญหา Distributed Systems โดยไม่ต้องเขียน Boilerplate Code

What Dapr provides:
┌────────────────────────────────────────────────────────────────┐
│                        Dapr Building Blocks                     │
├──────────────────┬─────────────────────────────────────────────┤
│ Service Invoke   │ gRPC/HTTP service-to-service calls          │
│                  │ + auto retry, mTLS, tracing                 │
├──────────────────┼─────────────────────────────────────────────┤
│ State Management │ Key-Value store (Redis, Cosmos, DynamoDB)    │
│                  │ Transactional operations                     │
├──────────────────┼─────────────────────────────────────────────┤
│ Pub/Sub          │ Message broker (Kafka, Redis, NATS)         │
│                  │ At-least-once delivery                       │
├──────────────────┼─────────────────────────────────────────────┤
│ Bindings         │ Input/Output to external systems            │
│                  │ (S3, databases, email, etc.)                │
├──────────────────┼─────────────────────────────────────────────┤
│ Actors           │ Virtual Actors pattern                       │
│                  │ (stateful, single-threaded compute)         │
├──────────────────┼─────────────────────────────────────────────┤
│ Secrets          │ Vault, Kubernetes secrets                   │
├──────────────────┼─────────────────────────────────────────────┤
│ Config           │ Configuration management                     │
├──────────────────┼─────────────────────────────────────────────┤
│ Workflow         │ Long-running workflows (Saga replacement)   │
└──────────────────┴─────────────────────────────────────────────┘
```

### 3.2 Dapr Implementation ตัวอย่าง

```go
// order-service/main.go with Dapr
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "net/http"
    
    dapr "github.com/dapr/go-sdk/client"
    "github.com/dapr/go-sdk/service/common"
    daprd "github.com/dapr/go-sdk/service/http"
)

func main() {
    // Create Dapr service
    s := daprd.NewService(":8080")
    
    // Subscribe to events
    s.AddTopicEventHandler(&common.Subscription{
        PubsubName: "order-pubsub",
        Topic:      "payment.completed",
    }, handlePaymentCompleted)
    
    // Add service invocation handler
    s.AddServiceInvocationHandler("create-order", createOrderHandler)
    
    s.Start()
}

func createOrderHandler(ctx context.Context, in *common.InvocationEvent) (*common.Content, error) {
    client, err := dapr.NewClient()
    if err != nil {
        return nil, err
    }
    defer client.Close()
    
    var req CreateOrderRequest
    json.Unmarshal(in.Data, &req)
    
    order := processOrder(req)
    
    // Save state via Dapr (abstracts Redis/Cosmos/etc.)
    orderData, _ := json.Marshal(order)
    client.SaveState(ctx, "order-store", order.ID, orderData, nil)
    
    // Publish event via Dapr (abstracts Kafka/RabbitMQ/etc.)
    eventData, _ := json.Marshal(OrderCreatedEvent{
        OrderID:   order.ID,
        CustomerID: req.CustomerID,
        Amount:    order.Total,
    })
    client.PublishEvent(ctx, "order-pubsub", "order.created", eventData)
    
    // Call payment service via Dapr service invocation
    paymentResp, err := client.InvokeMethodWithContent(ctx,
        "payment-service",   // Dapr App ID
        "charge",            // Method
        "POST",
        &dapr.DataContent{
            ContentType: "application/json",
            Data:        chargeData,
        },
    )
    
    respData, _ := json.Marshal(order)
    return &common.Content{
        ContentType: "application/json",
        Data:        respData,
    }, nil
}

func handlePaymentCompleted(ctx context.Context, e *common.TopicEvent) (retry bool, err error) {
    var event PaymentCompletedEvent
    json.Unmarshal(e.RawData, &event)
    
    client, _ := dapr.NewClient()
    defer client.Close()
    
    // Update order status
    orderData, _ := client.GetState(ctx, "order-store", event.OrderID, nil)
    
    var order Order
    json.Unmarshal(orderData.Value, &order)
    order.Status = "confirmed"
    
    newData, _ := json.Marshal(order)
    client.SaveState(ctx, "order-store", order.ID, newData, nil)
    
    return false, nil
}
```

### 3.3 Dapr Component Configuration

```yaml
# dapr/components/pubsub.yaml
# เปลี่ยน Backend ได้โดยไม่แก้ Code!
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: order-pubsub
spec:
  type: pubsub.kafka  # เปลี่ยนเป็น pubsub.redis ได้ทันที
  version: v1
  metadata:
  - name: brokers
    value: "kafka-broker:9092"
  - name: consumerGroup
    value: "order-service"
  - name: authType
    value: "none"

---
# dapr/components/statestore.yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: order-store
spec:
  type: state.redis   # เปลี่ยนเป็น state.postgresql ได้ทันที
  version: v1
  metadata:
  - name: redisHost
    value: "redis:6379"
  - name: enableTLS
    value: "false"
```

---

## 4. Next-Generation Service Mesh

### 4.1 Ambient Mesh (Istio ไม่ต้อง Sidecar)

```
ปัญหาของ Traditional Sidecar Mesh:
- ทุก Pod ต้องมี Envoy sidecar (+50MB memory each)
- Pod startup latency เพิ่มขึ้น
- Operational complexity สูง

Istio Ambient Mesh (2023+):
ไม่ต้องใช้ Sidecar แล้ว!

Traditional (Sidecar):          Ambient Mesh:
┌─────────────────────┐         ┌─────────────────────────────────┐
│  Pod                │         │  Node                            │
│  ┌─────┐ ┌───────┐  │         │  ┌──────────────────────────┐   │
│  │ App │ │Envoy  │  │         │  │ ztunnel (per-node)       │   │
│  │     │ │Sidecar│  │         │  │ L4 Security (mTLS, etc.) │   │
│  └─────┘ └───────┘  │         │  └──────────────────────────┘   │
└─────────────────────┘         │  ┌─────────┐ ┌─────────┐        │
                                │  │  Pod A  │ │  Pod B  │        │
                                │  │  (App)  │ │  (App)  │        │
                                │  └─────────┘ └─────────┘        │
                                │  (No sidecar needed!)            │
                                └─────────────────────────────────┘
                                
Memory savings: -50MB per Pod
Startup speed: Faster
Traffic: L7 via Waypoint Proxy (per namespace, optional)
```

### 4.2 SPIFFE/SPIRE - Workload Identity

```
SPIFFE (Secure Production Identity Framework For Everyone):
แทน API Keys ด้วย Cryptographic Identity

Service A ──> SPIRE Agent ──> SVID (X.509 cert)
                                    │
                              ┌─────▼──────────────────┐
                              │ Service B               │
                              │ ตรวจสอบ cert ทันที      │
                              │ ไม่ต้องใช้ API Key      │
                              └─────────────────────────┘

ทำงานร่วมกับ Istio/Cilium ได้ดี
```

---

## 5. Serverless Microservices

### 5.1 Knative - Kubernetes-native Serverless

```
Knative Components:
┌─────────────────────────────────────────────────────────────┐
│                        Knative                               │
├─────────────────────────────────────────────────────────────┤
│ Serving:                                                      │
│ - Scale to zero (ประหยัดค่าใช้จ่าย 80%+ สำหรับ low-traffic) │
│ - Auto-scale based on requests per second                    │
│ - Traffic splitting for A/B testing                          │
│                                                               │
│ Eventing:                                                     │
│ - Event routing and filtering                                │
│ - Source → Trigger → Service                                 │
│ - Built-in CloudEvents standard                              │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 Knative Service Definition

```yaml
# knative/image-processor.yaml
# Scale to zero เมื่อไม่มีงาน
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: image-processor
  namespace: production
spec:
  template:
    metadata:
      annotations:
        # Scale from 0 to 100 based on concurrency
        autoscaling.knative.dev/class: "kpa.autoscaling.knative.dev"
        autoscaling.knative.dev/metric: "concurrency"
        autoscaling.knative.dev/target: "10"
        autoscaling.knative.dev/minScale: "0"  # Scale to zero!
        autoscaling.knative.dev/maxScale: "100"
        autoscaling.knative.dev/scaleToZeroGracePeriod: "30s"
    spec:
      containerConcurrency: 10
      timeoutSeconds: 300
      containers:
      - image: image-processor:v1.0
        resources:
          requests:
            memory: "256Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "2000m"
        env:
        - name: S3_BUCKET
          valueFrom:
            secretKeyRef:
              name: aws-secrets
              key: bucket
              
---
# Event trigger: เมื่อ upload ไป S3 → trigger image processor
apiVersion: eventing.knative.dev/v1
kind: Trigger
metadata:
  name: image-upload-trigger
spec:
  broker: default
  filter:
    attributes:
      type: com.amazonaws.s3.objectcreated
      source: my-bucket
  subscriber:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: image-processor
```

---

## 6. AI-Augmented Operations (AIOps)

### 6.1 AI ใน Operations

```
AI กำลังเปลี่ยน Operations:

Traditional Ops:              AI-Augmented Ops:
┌─────────────────────┐       ┌─────────────────────────────────┐
│ Alert fires          │       │ AI detects anomaly before alert │
│ On-call engineer     │       │ AI suggests root cause          │
│ Manual investigation │       │ AI recommends fix               │
│ Trial and error fix  │       │ Auto-remediation (supervised)   │
│ Post-mortem          │       │ AI learns from incident         │
└─────────────────────┘       └─────────────────────────────────┘

Tools:
- Dynatrace Davis AI: Root cause analysis
- Datadog Watchdog: Anomaly detection  
- AWS DevOps Guru: ML-powered recommendations
- GitHub Copilot for Ops: Generate runbooks
```

### 6.2 Anomaly Detection Pipeline

```python
# aiops/anomaly_detection/detector.py
import numpy as np
from sklearn.ensemble import IsolationForest
from prometheus_client import CollectorRegistry, Gauge
import pandas as pd
from typing import List, Dict
import asyncio

class MetricsAnomalyDetector:
    """Detect anomalies in service metrics using ML"""
    
    def __init__(self, prometheus_url: str):
        self.prometheus_url = prometheus_url
        self.models: Dict[str, IsolationForest] = {}
        self.baseline_windows: Dict[str, list] = {}
    
    async def train_baseline(self, service_name: str, days: int = 14):
        """Train on 2 weeks of historical data"""
        # Query Prometheus for historical data
        query = f"""
        {{
            http_request_duration_seconds_p99{{service="{service_name}"}},
            http_requests_total{{service="{service_name}"}},
            go_goroutines{{service="{service_name}"}},
            process_resident_memory_bytes{{service="{service_name}"}}
        }}
        """
        
        # In real implementation: use Prometheus API
        metrics_df = await self.query_prometheus_range(service_name, days)
        
        # Feature engineering
        features = pd.DataFrame({
            'p99_latency': metrics_df['p99_latency'],
            'request_rate': metrics_df['request_rate'],
            'goroutines': metrics_df['goroutines'],
            'memory': metrics_df['memory'],
            'hour_of_day': metrics_df.index.hour,
            'day_of_week': metrics_df.index.dayofweek,
            # Rolling features
            'p99_latency_rolling_mean': metrics_df['p99_latency'].rolling(12).mean(),
            'p99_latency_rolling_std': metrics_df['p99_latency'].rolling(12).std(),
        }).dropna()
        
        # Train Isolation Forest
        model = IsolationForest(
            n_estimators=200,
            contamination=0.05,  # Expect 5% anomalies
            random_state=42,
            n_jobs=-1
        )
        model.fit(features)
        
        self.models[service_name] = model
        print(f"Model trained for {service_name} with {len(features)} samples")
    
    async def detect_anomaly(self, service_name: str, current_metrics: dict) -> AnomalyResult:
        if service_name not in self.models:
            return AnomalyResult(is_anomaly=False, reason="No model trained")
        
        model = self.models[service_name]
        
        features = np.array([[
            current_metrics.get('p99_latency', 0),
            current_metrics.get('request_rate', 0),
            current_metrics.get('goroutines', 0),
            current_metrics.get('memory', 0),
            pd.Timestamp.now().hour,
            pd.Timestamp.now().dayofweek,
            0, 0,  # Rolling features (approximated)
        ]])
        
        score = model.score_samples(features)[0]
        is_anomaly = model.predict(features)[0] == -1
        
        reason = ""
        if is_anomaly:
            # Identify which metric contributed most
            reason = self.explain_anomaly(service_name, current_metrics)
        
        return AnomalyResult(
            is_anomaly=is_anomaly,
            anomaly_score=score,
            reason=reason,
            severity=self.calculate_severity(score),
        )
    
    def explain_anomaly(self, service_name: str, metrics: dict) -> str:
        """Simple rule-based explanation"""
        reasons = []
        
        baseline = self.get_baseline_stats(service_name)
        
        if metrics.get('p99_latency', 0) > baseline['p99_latency_mean'] + (3 * baseline['p99_latency_std']):
            reasons.append(f"P99 latency is {metrics['p99_latency']:.0f}ms (normal: {baseline['p99_latency_mean']:.0f}ms)")
        
        if metrics.get('error_rate', 0) > 0.05:
            reasons.append(f"Error rate is {metrics['error_rate']*100:.1f}% (threshold: 5%)")
        
        if metrics.get('goroutines', 0) > baseline['goroutines_p99']:
            reasons.append(f"Goroutine count is unusually high: {metrics['goroutines']}")
        
        return "; ".join(reasons) if reasons else "Multivariate anomaly detected"


class AutoRemediationService:
    """Automatic remediation for known issues"""
    
    REMEDIATION_PLAYBOOKS = {
        "HIGH_MEMORY": [
            "Trigger GC",
            "Scale up pods",
            "Alert on-call if not resolved in 5 min",
        ],
        "HIGH_ERROR_RATE": [
            "Check downstream services",
            "Enable circuit breaker",
            "Alert on-call",
        ],
        "POD_CRASHLOOP": [
            "Restart pod",
            "Check logs",
            "Scale down if DB connection issue",
        ],
    }
    
    async def remediate(self, alert: Alert):
        playbook = self.REMEDIATION_PLAYBOOKS.get(alert.type, [])
        
        for step in playbook:
            success = await self.execute_step(step, alert)
            if not success:
                # Escalate to human
                await self.notify_oncall(alert, step)
                return
```

---

## 7. Green Computing

### 7.1 Sustainability ใน Microservices

```
Carbon-Aware Computing:

Traditional:                    Green Computing:
Run 24/7 at full power          Scale to zero when idle
Deploy in cheapest region       Deploy in cleanest energy region
Ignore energy usage             Track carbon footprint
No efficiency metrics           Optimize for carbon, not just $

Tools:
- Cloud Carbon Footprint (opensource)
- AWS Customer Carbon Footprint Tool
- Google Cloud Carbon Footprint
- Kepler (Kubernetes Energy Efficiency)
```

### 7.2 Kepler - Kubernetes Energy Monitor

```yaml
# kepler/kepler.yaml
# ติดตาม Energy consumption ระดับ Pod
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kepler
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: kepler
  template:
    spec:
      containers:
      - name: kepler
        image: quay.io/sustainable_computing_io/kepler:latest
        env:
        - name: ENABLE_EBPF_CGROUPID
          value: "true"
        - name: ENABLE_GPU
          value: "false"
        - name: ENABLE_CRI_RUNTIME_CTR_STAT  
          value: "true"
        ports:
        - name: http
          containerPort: 9102
          hostPort: 9102
```

```python
# green-ops/carbon_scheduler.py
# Schedule batch jobs ในช่วงที่ Carbon Intensity ต่ำ
import requests
from datetime import datetime, timedelta
import pytz

BANGKOK_TZ = pytz.timezone('Asia/Bangkok')

class CarbonAwareScheduler:
    """Schedule workloads based on carbon intensity"""
    
    def __init__(self, carbon_api_key: str):
        self.api_key = carbon_api_key
        self.base_url = "https://api.electricitymap.org/v3"
    
    def get_carbon_intensity(self, zone: str = "TH") -> dict:
        """Get current carbon intensity (gCO2/kWh)"""
        response = requests.get(
            f"{self.base_url}/carbon-intensity/latest?zone={zone}",
            headers={"auth-token": self.api_key}
        )
        return response.json()
    
    def get_best_time_to_run(self, zone: str = "TH", hours_ahead: int = 24) -> datetime:
        """Find the greenest time to run a batch job"""
        response = requests.get(
            f"{self.base_url}/carbon-intensity/forecast?zone={zone}",
            headers={"auth-token": self.api_key}
        )
        forecast = response.json()
        
        # Find minimum carbon intensity in next N hours
        forecasts = forecast.get('forecast', [])[:hours_ahead]
        
        min_carbon = float('inf')
        best_time = datetime.now(BANGKOK_TZ)
        
        for point in forecasts:
            if point['carbonIntensity'] < min_carbon:
                min_carbon = point['carbonIntensity']
                best_time = datetime.fromisoformat(point['datetime'])
        
        return best_time
    
    def should_defer_job(self, zone: str = "TH", threshold: int = 200) -> tuple[bool, str]:
        """
        Should we defer this job until carbon is lower?
        Returns (should_defer, reason)
        """
        current = self.get_carbon_intensity(zone)
        current_intensity = current.get('carbonIntensity', 0)
        
        if current_intensity > threshold:
            best_time = self.get_best_time_to_run(zone)
            minutes_to_wait = (best_time - datetime.now(BANGKOK_TZ)).total_seconds() / 60
            return True, f"Current: {current_intensity}gCO2/kWh, best time in {minutes_to_wait:.0f} minutes"
        
        return False, f"Current carbon intensity OK: {current_intensity}gCO2/kWh"
```

---

## 8. Platform Engineering & Internal Developer Platform

### 8.1 IDPx (Internal Developer Platform)

```
Platform Engineering Trend 2024-2025:

เปลี่ยนจาก: "DevOps team ต้องทำ Infrastructure ทุกอย่าง"
เป็น: "Platform team สร้าง Self-service capabilities"

Internal Developer Platform:
┌────────────────────────────────────────────────────────────────┐
│                  Developer Portal (Backstage)                   │
│                                                                  │
│  ┌───────────────┐  ┌───────────────┐  ┌──────────────────┐   │
│  │ Service       │  │ API Catalog   │  │  Tech Radar      │   │
│  │ Templates     │  │               │  │                  │   │
│  └───────────────┘  └───────────────┘  └──────────────────┘   │
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │              Golden Path (Scaffolding)                     │ │
│  │  "Create new microservice" →                               │ │
│  │   GitHub Repo + CI/CD + Monitoring + Runbook auto-created │ │
│  └───────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────┘

Tools:
- Backstage (Spotify's Developer Portal)
- Cortex (Service Catalog)
- Humanitec (Platform Orchestrator)
- Port (Internal Developer Portal)
```

### 8.2 Backstage Setup

```yaml
# backstage/catalog-info.yaml
# ทุก Service ต้องมีไฟล์นี้ (Service Catalog)
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: order-service
  description: "Manages order lifecycle for Thai e-commerce platform"
  annotations:
    github.com/project-slug: "company/order-service"
    backstage.io/techdocs-ref: dir:.
    pagerduty.com/service-id: "P123456"
    datadoghq.com/service-name: "order-service"
    prometheus.io/alert: "order-service-*"
  tags:
    - go
    - microservice
    - critical
  links:
  - url: "https://grafana.internal/order-service"
    title: Grafana Dashboard
  - url: "https://jaeger.internal/order-service"
    title: Distributed Traces
  - url: "https://wiki.internal/runbooks/order-service"
    title: Runbook
spec:
  type: service
  lifecycle: production
  owner: team-orders
  system: e-commerce-platform
  dependsOn:
  - component:payment-service
  - component:inventory-service
  - resource:orders-postgres
  - resource:kafka
  providesApis:
  - order-service-api
```

---

## 9. Emerging Patterns Summary

### 9.1 Technology Adoption Timeline

```
Technology Adoption 2024-2025:

NOW (Adopt):
✅ Kubernetes + Helm/ArgoCD
✅ Observability (Prometheus, Jaeger, Loki)
✅ Service Mesh (Istio/Linkerd)
✅ GitOps (ArgoCD/Flux)
✅ Platform Engineering (Backstage)

TRIAL (Evaluate):
🔬 Dapr (Distributed App Runtime)
🔬 eBPF-based networking (Cilium)
🔬 Knative (Serverless on K8s)
🔬 AI-Assisted Operations
🔬 Carbon-aware scheduling

ASSESS (Watch):
👀 WebAssembly microservices
👀 Ambient Mesh (Istio without sidecar)
👀 SPIFFE/SPIRE workload identity
👀 Rust for high-performance services

HOLD (Avoid for now):
⏸ Blockchain for most use cases
⏸ Overly complex service mesh setups
⏸ Premature microservices decomposition
```

### 9.2 Skills Roadmap สำหรับ 2025

```
Backend/Platform Engineer Learning Path:

Foundation (ต้องรู้):
□ Container fundamentals (Docker)
□ Kubernetes (CKA level)
□ CI/CD (GitHub Actions, ArgoCD)
□ Observability (Prometheus, Grafana, Jaeger)
□ API Design (REST, gRPC, GraphQL)

Intermediate (ควรรู้):
□ Service Mesh (Istio basics)
□ Kafka/Event-driven architecture
□ Infrastructure as Code (Terraform)
□ Security (RBAC, mTLS, SAST/DAST)
□ Platform Engineering basics

Advanced (เพิ่มคุณค่า):
□ eBPF concepts (Cilium/Hubble)
□ Wasm serverless functions
□ AI/ML Operations (MLOps integration)
□ Cost optimization (FinOps)
□ Green computing practices

Leadership:
□ Architecture Decision Records (ADR)
□ Technology radar management
□ Platform team practices
□ Developer Experience (DevEx)
```

---

## สรุป

Microservices Roadmap 2024-2025 ชี้ให้เห็นทิศทางสำคัญ:

1. **eBPF** - Zero-overhead observability และ Network security ใน Kernel
2. **WebAssembly** - Edge computing และ Serverless runtime ที่เร็วกว่า Container
3. **Dapr** - Abstraction layer สำหรับ Distributed System primitives
4. **Ambient Mesh** - Service Mesh ไม่ต้องใช้ Sidecar (ลด overhead)
5. **AI Operations** - Automated anomaly detection และ Self-healing systems
6. **Green Computing** - Carbon-aware workload scheduling
7. **Platform Engineering** - Self-service Developer Portal ช่วย Developer Productivity

> "The best technology is invisible — it enables developers to focus on business value, not infrastructure plumbing"

---

*ถัดไป: Part 97 - Open Source Microservices Tools*

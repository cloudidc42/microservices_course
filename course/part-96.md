# Part 96: Microservices Technology Roadmap

## แผนที่เทคโนโลยี Microservices (2024-2025 และอนาคต)

ในบทนี้เราจะสำรวจทิศทางการพัฒนาของ Microservices ecosystem รวมถึงเทคโนโลยีใหม่ที่กำลังเข้ามามีบทบาท

---

## 1. สถานะปัจจุบันของ Microservices Ecosystem (2024-2025)

### ภาพรวม

Microservices ได้กลายเป็นแนวทางหลักสำหรับระบบขนาดใหญ่แล้ว โดยมีการนำมาใช้อย่างแพร่หลาย:

| ด้าน | สถานะ 2024-2025 |
|------|----------------|
| Container Orchestration | Kubernetes เป็น Standard de facto |
| Service Mesh | Istio และ Linkerd เติบโตสูง |
| GitOps | ArgoCD/Flux กลายเป็น Default |
| Observability | OpenTelemetry เป็น Standard |
| API Gateway | Kong, Traefik, Envoy แข่งขันกัน |
| Serverless | Lambda/Cloud Functions รวมกับ Microservices |

---

## 2. eBPF - เทคโนโลยีที่เปลี่ยนเกม

eBPF (extended Berkeley Packet Filter) ช่วยให้รัน Code ใน Linux Kernel ได้โดยตรง ทำให้ Observability และ Security เปลี่ยนไปมาก

```
                    eBPF Architecture
    ┌─────────────────────────────────────────────┐
    │              User Space                      │
    │  BPF Program  ──►  BPF Verifier              │
    │                         │                    │
    │                    JIT Compiler              │
    │                         │                    │
    ├─────────────────────────┼────────────────────┤
    │           Kernel Space  │                    │
    │                         ▼                    │
    │              BPF Maps ◄──► BPF Programs      │
    │                    │                         │
    │         ┌───────────┼───────────┐            │
    │         ▼           ▼           ▼            │
    │      Network     Tracing     Security        │
    │     (XDP/TC)    (kprobe)     (LSM)           │
    └─────────────────────────────────────────────┘
```

### eBPF ใน Microservices

```typescript
// cilium-network-policy.yaml
// ใช้ Cilium (eBPF-based) แทน iptables สำหรับ Network Policy
```

```yaml
# cilium-network-policy.yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-order-to-payment
  namespace: production
spec:
  endpointSelector:
    matchLabels:
      app: payment-service
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: order-service
      toPorts:
        - ports:
            - port: "3000"
              protocol: TCP
          rules:
            http:
              - method: POST
                path: /api/v1/payments
              - method: GET
                path: /api/v1/payments/.*
  egress:
    - toEndpoints:
        - matchLabels:
            app: postgres
      toPorts:
        - ports:
            - port: "5432"
              protocol: TCP
---
# eBPF-based Observability ด้วย Tetragon
apiVersion: cilium.io/v1alpha1
kind: TracingPolicy
metadata:
  name: payment-syscall-monitor
spec:
  kprobes:
    - call: sys_connect
      syscall: true
      args:
        - index: 0
          type: int
      selectors:
        - matchBinaries:
            - operator: In
              values:
                - /usr/local/bin/node
```

### eBPF ใน Observability

```yaml
# hubble-flow-tracing.yaml
# Hubble คือ Network Observability ที่ใช้ eBPF ผ่าน Cilium

# ดู Network Flow ระหว่าง Services
# hubble observe --namespace production --follow

# ตัวอย่าง Output:
# Jul  4 10:00:00.000 production/order-service:50234 
#   -> production/payment-service:3000 http-request 
#   FORWARDED (TCP Flags: ACK)
```

---

## 3. WebAssembly (WASM) ใน Microservices

WASM กำลังเปลี่ยนวิธีที่เราทำ Microservices โดยเฉพาะใน Edge Computing

```typescript
// wasm-plugin/src/plugin.ts
// Envoy Proxy WASM Plugin สำหรับ Request Transformation

// Plugin นี้เขียนด้วย Rust แล้วคอมไพล์เป็น WASM
// ตัวอย่างนี้แสดง Concept

export class RequestTransformPlugin {
  // ใช้ WASM เพิ่ม Header ให้ทุก Request
  onRequestHeaders(numHeaders: number): FilterHeadersStatus {
    this.addRequestHeader('X-Request-ID', generateUUID());
    this.addRequestHeader('X-Service-Name', 'my-microservice');
    
    // ตรวจสอบ JWT Token
    const authHeader = this.getRequestHeader('Authorization');
    if (authHeader) {
      const userId = this.extractUserIdFromJWT(authHeader);
      if (userId) {
        this.addRequestHeader('X-User-ID', userId);
      }
    }
    
    return FilterHeadersStatus.Continue;
  }

  private generateUUID(): string {
    return 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, (c) => {
      const r = Math.random() * 16 | 0;
      return (c === 'x' ? r : (r & 0x3 | 0x8)).toString(16);
    });
  }

  private extractUserIdFromJWT(authHeader: string): string | null {
    try {
      const token = authHeader.replace('Bearer ', '');
      const [, payload] = token.split('.');
      const decoded = JSON.parse(atob(payload));
      return decoded.sub || null;
    } catch {
      return null;
    }
  }
}
```

```yaml
# envoy-wasm-config.yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: request-transform-filter
  namespace: production
spec:
  configPatches:
    - applyTo: HTTP_FILTER
      match:
        context: SIDECAR_INBOUND
      patch:
        operation: INSERT_BEFORE
        value:
          name: envoy.filters.http.wasm
          typedConfig:
            "@type": type.googleapis.com/envoy.extensions.filters.http.wasm.v3.Wasm
            config:
              name: request_transform
              vmConfig:
                runtime: envoy.wasm.runtime.v8
                code:
                  local:
                    filename: /etc/wasm/request-transform.wasm
```

### Fermyon Spin - WASM Microservices Framework

```rust
// spin-app/src/lib.rs
use spin_sdk::http::{IntoResponse, Request, Response};
use spin_sdk::http_component;

/// Simple Rust WASM Microservice
#[http_component]
fn handle_request(req: Request) -> anyhow::Result<impl IntoResponse> {
    println!("Handling request to {:?}", req.header("spin-path-info"));
    
    Ok(Response::builder()
        .status(200)
        .header("content-type", "application/json")
        .body(r#"{"message": "Hello from WASM Microservice!"}"#)
        .build())
}
```

```toml
# spin.toml - Fermyon Spin Config
spin_manifest_version = 2

[application]
name = "payment-wasm"
version = "1.0.0"

[[trigger.http]]
route = "/api/..."
component = "payment"

[component.payment]
source = "target/wasm32-wasi/release/payment_service.wasm"
allowed_outbound_hosts = ["https://api.stripe.com"]
key_value_stores = ["default"]
```

---

## 4. Dapr - Distributed Application Runtime

Dapr ช่วยให้ Microservices ไม่ต้องรู้จัก Infrastructure ที่อยู่เบื้องหลัง

```yaml
# dapr-components/pubsub.yaml
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
      value: kafka:9092
    - name: consumerGroup
      value: order-service
    - name: authType
      value: none
---
# dapr-components/statestore.yaml
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
      value: redis:6379
    - name: actorStateStore
      value: "true"
```

```typescript
// order-service/src/order-dapr.service.ts
import { Injectable } from '@nestjs/common';
import { DaprClient, DaprServer, CommunicationProtocolEnum } from '@dapr/dapr';

@Injectable()
export class OrderDaprService {
  private readonly daprClient: DaprClient;

  constructor() {
    this.daprClient = new DaprClient({
      daprHost: 'localhost',
      daprPort: '3500',
      communicationProtocol: CommunicationProtocolEnum.HTTP,
    });
  }

  // Publish Event ผ่าน Dapr Pub/Sub
  async publishOrderCreated(order: any): Promise<void> {
    await this.daprClient.pubsub.publish('pubsub', 'order-created', order);
  }

  // บันทึก State ผ่าน Dapr State Store
  async saveOrderState(orderId: string, state: any): Promise<void> {
    await this.daprClient.state.save('statestore', [{
      key: `order:${orderId}`,
      value: state,
    }]);
  }

  // เรียก Service อื่นผ่าน Dapr Service Invocation
  async callPaymentService(paymentRequest: any): Promise<any> {
    return this.daprClient.invoker.invoke(
      'payment-service',
      'process-payment',
      'POST',
      paymentRequest,
    );
  }

  // ใช้ Dapr Secret Store
  async getStripeApiKey(): Promise<string> {
    const secret = await this.daprClient.secret.get(
      'vault-secret-store',
      'stripe-api-key',
    );
    return secret['stripe-api-key'];
  }

  // Dapr Actor สำหรับ Order State Machine
  async createOrderActor(orderId: string): Promise<any> {
    return this.daprClient.actor.proxy.create<OrderActor>(
      OrderActorImpl,
      orderId,
    );
  }
}
```

```typescript
// actors/order-actor.ts
import { AbstractActor } from '@dapr/dapr';

export interface OrderActor {
  processOrder(request: any): Promise<void>;
  getStatus(): Promise<string>;
  cancelOrder(reason: string): Promise<void>;
}

export class OrderActorImpl extends AbstractActor implements OrderActor {
  async processOrder(request: any): Promise<void> {
    const state = await this.getActorStateManager().getState<any>('order');
    
    if (state?.status === 'PROCESSING') {
      throw new Error('Order already being processed');
    }

    await this.getActorStateManager().setState('order', {
      ...request,
      status: 'PROCESSING',
      updatedAt: new Date(),
    });

    // จัดเก็บ Timer สำหรับ Timeout
    await this.registerActorTimer(
      'order-timeout',
      'handleTimeout',
      new Date(Date.now() + 30 * 60 * 1000), // 30 นาที
      undefined,
    );
  }

  async getStatus(): Promise<string> {
    const state = await this.getActorStateManager().getState<any>('order');
    return state?.status || 'UNKNOWN';
  }

  async cancelOrder(reason: string): Promise<void> {
    const state = await this.getActorStateManager().getState<any>('order');
    
    await this.getActorStateManager().setState('order', {
      ...state,
      status: 'CANCELLED',
      cancellationReason: reason,
      cancelledAt: new Date(),
    });
  }

  async handleTimeout(): Promise<void> {
    const status = await this.getStatus();
    if (status === 'PROCESSING') {
      await this.cancelOrder('Timeout');
    }
  }
}
```

---

## 5. Service Mesh Evolution

### Ambient Mesh - Istio ไม่ต้องใช้ Sidecar

```yaml
# ambient-mesh-label.yaml
# เปิดใช้ Ambient Mesh โดยไม่ต้องมี Sidecar Proxy
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    istio.io/dataplane-mode: ambient  # ใช้ ztunnel แทน Envoy Sidecar
---
# L4 Policy ผ่าน ztunnel (ไม่ต้องการ Sidecar)
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: payment-policy
  namespace: production
spec:
  selector:
    matchLabels:
      app: payment-service
  action: ALLOW
  rules:
    - from:
        - source:
            principals:
              - cluster.local/ns/production/sa/order-service
```

### Linkerd 2.x - Ultra-light Service Mesh

```yaml
# linkerd-service-profile.yaml
apiVersion: linkerd.io/v1alpha2
kind: ServiceProfile
metadata:
  name: payment-service.production.svc.cluster.local
  namespace: production
spec:
  routes:
    - name: POST /api/v1/payments
      condition:
        method: POST
        pathRegex: /api/v1/payments
      responseClasses:
        - condition:
            status:
              min: 500
              max: 599
          isFailure: true
      timeout: 10s
      retryBudget:
        retryRatio: 0.2
        minRetriesPerSecond: 10
        ttl: 10s
    - name: GET /api/v1/payments/{id}
      condition:
        method: GET
        pathRegex: /api/v1/payments/[^/]*
      isRetryable: true
      timeout: 5s
```

---

## 6. Serverless Microservices Convergence

### AWS Lambda + Container Images

```typescript
// lambda-handler/src/handler.ts
import { Handler, APIGatewayProxyEvent, APIGatewayProxyResult } from 'aws-lambda';
import { NestFactory } from '@nestjs/core';
import { ExpressAdapter } from '@nestjs/platform-express';
import express from 'express';
import serverlessExpress from '@vendia/serverless-express';
import { AppModule } from './app.module';

let serverlessExpressInstance: any;

async function bootstrapServer() {
  if (!serverlessExpressInstance) {
    const expressApp = express();
    const nestApp = await NestFactory.create(AppModule, new ExpressAdapter(expressApp));
    await nestApp.init();
    serverlessExpressInstance = serverlessExpress({ app: expressApp });
  }
  return serverlessExpressInstance;
}

export const handler: Handler = async (
  event: APIGatewayProxyEvent,
  context: any,
  callback: any,
) => {
  const server = await bootstrapServer();
  return server(event, context, callback);
};
```

```yaml
# serverless.yml (Serverless Framework)
service: payment-microservice

provider:
  name: aws
  runtime: nodejs20.x
  region: ap-southeast-1
  environment:
    DATABASE_URL: ${ssm:/production/payment/database-url}
    REDIS_URL: ${ssm:/production/payment/redis-url}
  iam:
    role:
      statements:
        - Effect: Allow
          Action:
            - ssm:GetParameter
          Resource: arn:aws:ssm:ap-southeast-1:*:parameter/production/*
        - Effect: Allow
          Action:
            - sqs:SendMessage
            - sqs:ReceiveMessage
          Resource: !GetAtt PaymentQueue.Arn

functions:
  processPayment:
    handler: dist/handler.handler
    events:
      - http:
          path: /api/v1/payments
          method: POST
          authorizer:
            name: jwtAuthorizer
            type: TOKEN
    timeout: 30
    memorySize: 512
    reservedConcurrency: 100
    
  reconciliation:
    handler: dist/reconciliation.handler
    events:
      - schedule:
          rate: cron(0 2 * * ? *)  # 02:00 UTC ทุกวัน
    timeout: 900  # 15 นาที

resources:
  Resources:
    PaymentQueue:
      Type: AWS::SQS::Queue
      Properties:
        QueueName: payment-queue.fifo
        FifoQueue: true
        ContentBasedDeduplication: true
        VisibilityTimeout: 60
```

### Knative - Kubernetes-native Serverless

```yaml
# knative-service.yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: recommendation-service
  namespace: production
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/minScale: "1"
        autoscaling.knative.dev/maxScale: "50"
        autoscaling.knative.dev/target: "100"  # 100 concurrent requests per pod
        autoscaling.knative.dev/scale-to-zero-pod-retention-period: "1m"
    spec:
      containerConcurrency: 100
      timeoutSeconds: 30
      containers:
        - image: your-registry/recommendation-service:1.0.0
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: 1000m
              memory: 1Gi
          env:
            - name: ML_MODEL_PATH
              value: /models/recommendation
```

---

## 7. AI/ML Integration Trends

### ML Feature Store

```typescript
// ml-feature-store/src/feature.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { InjectRedis } from '@liaoliaots/nestjs-redis';
import Redis from 'ioredis';

export interface FeatureRequest {
  entityId: string;
  entityType: 'user' | 'product' | 'order';
  features: string[];
  maxAge?: number; // วินาที
}

export interface FeatureSet {
  entityId: string;
  features: Record<string, number | string | boolean>;
  computedAt: Date;
}

@Injectable()
export class FeatureStoreService {
  private readonly logger = new Logger(FeatureStoreService.name);
  private readonly DEFAULT_TTL = 3600; // 1 ชั่วโมง

  constructor(
    @InjectRedis() private readonly redis: Redis,
    private readonly featureComputer: FeatureComputerService,
  ) {}

  async getFeatures(request: FeatureRequest): Promise<FeatureSet> {
    const cacheKey = `features:${request.entityType}:${request.entityId}`;
    const maxAge = request.maxAge || this.DEFAULT_TTL;
    
    // ตรวจสอบ Cache
    const cached = await this.redis.get(cacheKey);
    if (cached) {
      const parsed = JSON.parse(cached);
      const age = (Date.now() - new Date(parsed.computedAt).getTime()) / 1000;
      
      if (age < maxAge) {
        return parsed;
      }
    }

    // คำนวณ Features ใหม่
    const features = await this.featureComputer.compute(
      request.entityId,
      request.entityType,
      request.features,
    );

    const featureSet: FeatureSet = {
      entityId: request.entityId,
      features,
      computedAt: new Date(),
    };

    await this.redis.setex(cacheKey, this.DEFAULT_TTL, JSON.stringify(featureSet));

    return featureSet;
  }

  async storeFeatures(
    entityId: string,
    entityType: string,
    features: Record<string, any>,
  ): Promise<void> {
    const cacheKey = `features:${entityType}:${entityId}`;
    
    const featureSet: FeatureSet = {
      entityId,
      features,
      computedAt: new Date(),
    };

    await this.redis.setex(cacheKey, this.DEFAULT_TTL, JSON.stringify(featureSet));
  }
}
```

### LLM Integration ใน Microservices

```typescript
// ai-service/src/llm.service.ts
import { Injectable, Logger } from '@nestjs/common';
import Anthropic from '@anthropic-ai/sdk';

export interface ChatMessage {
  role: 'user' | 'assistant';
  content: string;
}

@Injectable()
export class LLMService {
  private readonly logger = new Logger(LLMService.name);
  private readonly client: Anthropic;

  constructor() {
    this.client = new Anthropic({
      apiKey: process.env.ANTHROPIC_API_KEY,
    });
  }

  // ใช้ AI สำหรับ Customer Support Chatbot
  async handleCustomerQuery(
    message: string,
    conversationHistory: ChatMessage[],
    contextData: {
      userId: string;
      recentOrders?: any[];
      accountInfo?: any;
    },
  ): Promise<string> {
    const systemPrompt = `
You are a helpful customer service agent for a food delivery platform.
You have access to the following customer data:
- Customer ID: ${contextData.userId}
- Recent Orders: ${JSON.stringify(contextData.recentOrders)}
- Account Info: ${JSON.stringify(contextData.accountInfo)}

Respond in the same language as the user. Be helpful, concise, and professional.
`;

    const messages = [
      ...conversationHistory.map(msg => ({
        role: msg.role as 'user' | 'assistant',
        content: msg.content,
      })),
      { role: 'user' as const, content: message },
    ];

    const response = await this.client.messages.create({
      model: 'claude-opus-4-5',
      max_tokens: 1024,
      system: systemPrompt,
      messages,
    });

    return response.content[0].type === 'text' ? response.content[0].text : '';
  }

  // ใช้ AI สำหรับ Product Description Generation
  async generateProductDescription(
    product: {
      name: string;
      ingredients: string[];
      category: string;
    },
    language: 'th' | 'en' = 'th',
  ): Promise<string> {
    const prompt = language === 'th'
      ? `เขียนคำอธิบายสั้นๆ น่ารับประทานสำหรับเมนู "${product.name}" ที่มีส่วนประกอบ: ${product.ingredients.join(', ')} ความยาวไม่เกิน 100 คำ`
      : `Write a short appetizing description for "${product.name}" with ingredients: ${product.ingredients.join(', ')}. Max 100 words.`;

    const response = await this.client.messages.create({
      model: 'claude-haiku-4-5',
      max_tokens: 200,
      messages: [{ role: 'user', content: prompt }],
    });

    return response.content[0].type === 'text' ? response.content[0].text : '';
  }

  // ใช้ AI สำหรับ Fraud Detection Explanation
  async explainFraudDecision(
    fraudFlags: string[],
    riskScore: number,
    transactionData: any,
  ): Promise<string> {
    const response = await this.client.messages.create({
      model: 'claude-haiku-4-5',
      max_tokens: 300,
      messages: [{
        role: 'user',
        content: `
Explain this fraud detection decision briefly:
- Risk Score: ${riskScore}/100
- Flags: ${fraudFlags.join(', ')}
- Transaction: ${JSON.stringify(transactionData)}

Provide a brief, clear explanation for the customer.
`,
      }],
    });

    return response.content[0].type === 'text' ? response.content[0].text : '';
  }
}
```

---

## 8. Edge Computing Growth

```typescript
// edge-service/src/edge-worker.ts
// Cloudflare Workers - Edge Microservice

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);

    // Edge Caching สำหรับ Product Catalog
    if (url.pathname.startsWith('/api/v1/products')) {
      return handleProductRequest(request, env);
    }

    // Edge Authentication
    if (request.headers.get('Authorization')) {
      const userId = await verifyToken(
        request.headers.get('Authorization')!,
        env.JWT_SECRET,
      );
      
      if (!userId) {
        return new Response('Unauthorized', { status: 401 });
      }
    }

    // Forward ไปยัง Origin
    return fetch(request);
  },
};

async function handleProductRequest(request: Request, env: Env): Promise<Response> {
  const cacheKey = new Request(request.url, { method: 'GET' });
  const cache = caches.default;

  // ตรวจสอบ Edge Cache ก่อน
  let response = await cache.match(cacheKey);
  
  if (!response) {
    // ดึงจาก Origin
    response = await fetch(request);
    
    if (response.ok) {
      // Cache ที่ Edge เป็นเวลา 5 นาที
      const responseToCache = new Response(response.body, response);
      responseToCache.headers.set('Cache-Control', 'public, max-age=300');
      await cache.put(cacheKey, responseToCache);
    }
  }

  return response;
}

async function verifyToken(token: string, secret: string): Promise<string | null> {
  try {
    // JWT verification ที่ Edge
    const [header, payload, signature] = token.replace('Bearer ', '').split('.');
    
    const encoder = new TextEncoder();
    const data = encoder.encode(`${header}.${payload}`);
    const key = await crypto.subtle.importKey(
      'raw',
      encoder.encode(secret),
      { name: 'HMAC', hash: 'SHA-256' },
      false,
      ['verify'],
    );

    const isValid = await crypto.subtle.verify(
      'HMAC',
      key,
      Uint8Array.from(atob(signature.replace(/-/g, '+').replace(/_/g, '/')), c => c.charCodeAt(0)),
      data,
    );

    if (!isValid) return null;

    const decoded = JSON.parse(atob(payload));
    return decoded.sub;
  } catch {
    return null;
  }
}

interface Env {
  JWT_SECRET: string;
  ORIGIN_URL: string;
}
```

---

## 9. GitOps Maturity

### ArgoCD ApplicationSet

```yaml
# argocd-applicationset.yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: microservices
  namespace: argocd
spec:
  generators:
    - git:
        repoURL: https://github.com/your-org/microservices-gitops
        revision: main
        directories:
          - path: apps/*
    - matrix:
        generators:
          - list:
              elements:
                - env: staging
                  cluster: staging-cluster
                - env: production
                  cluster: production-cluster
          - git:
              repoURL: https://github.com/your-org/microservices-gitops
              revision: main
              directories:
                - path: apps/*
  template:
    metadata:
      name: '{{env}}-{{path.basename}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/your-org/microservices-gitops
        targetRevision: main
        path: '{{path}}'
        helm:
          valueFiles:
            - values-{{env}}.yaml
      destination:
        server: '{{cluster}}'
        namespace: '{{env}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
          allowEmpty: false
        syncOptions:
          - CreateNamespace=true
          - PrunePropagationPolicy=foreground
          - PruneLast=true
        retry:
          limit: 5
          backoff:
            duration: 5s
            maxDuration: 3m
            factor: 2
```

### Flux CD Progressive Delivery

```yaml
# fluxcd-kustomization.yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: payment-service
  namespace: flux-system
spec:
  interval: 10m
  sourceRef:
    kind: GitRepository
    name: microservices-repo
  path: ./k8s/payment-service
  prune: true
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: payment-service
      namespace: production
  postBuild:
    substitute:
      APP_VERSION: "${APP_VERSION}"
      ENVIRONMENT: production
---
# Progressive Delivery ด้วย Flagger
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: payment-service
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: payment-service
  progressDeadlineSeconds: 60
  service:
    port: 3000
    targetPort: 3000
    trafficPolicy:
      tls:
        mode: ISTIO_MUTUAL
  analysis:
    interval: 1m
    threshold: 5      # จำนวนครั้งที่ Fail ก่อน Rollback
    maxWeight: 50     # Traffic สูงสุดที่ส่งไป Canary
    stepWeight: 10    # เพิ่มทีละ 10%
    metrics:
      - name: request-success-rate
        min: 99       # ต้องสำเร็จ >= 99%
        interval: 1m
      - name: request-duration
        max: 500      # ต้องไม่เกิน 500ms (p99)
        interval: 30s
    webhooks:
      - name: acceptance-test
        type: pre-rollout
        url: http://flagger-loadtester.test/
        timeout: 30s
        metadata:
          type: bash
          cmd: "curl -sd 'test' http://payment-service.production/health | grep 200"
```

---

## 10. Platform Engineering Future

### Internal Developer Platform (IDP)

```typescript
// idp/src/platform.service.ts
// Internal Developer Platform - Self-service สำหรับ Dev Teams

import { Injectable, Logger } from '@nestjs/common';

export interface ServiceTemplate {
  id: string;
  name: string;
  language: 'typescript' | 'python' | 'go' | 'java';
  framework: string;
  databases: string[];
  messaging: string[];
  features: string[];
}

export interface ServiceCreationRequest {
  serviceName: string;
  teamName: string;
  templateId: string;
  environment: 'development' | 'staging' | 'production';
  config: Record<string, string>;
}

@Injectable()
export class PlatformService {
  private readonly logger = new Logger(PlatformService.name);

  private readonly templates: ServiceTemplate[] = [
    {
      id: 'nestjs-rest-api',
      name: 'NestJS REST API',
      language: 'typescript',
      framework: 'nestjs',
      databases: ['postgresql', 'redis'],
      messaging: ['kafka'],
      features: ['authentication', 'logging', 'metrics', 'tracing'],
    },
    {
      id: 'python-ml-service',
      name: 'Python ML Service',
      language: 'python',
      framework: 'fastapi',
      databases: ['postgresql'],
      messaging: ['rabbitmq'],
      features: ['model-serving', 'batch-processing', 'metrics'],
    },
  ];

  async createService(request: ServiceCreationRequest): Promise<{
    repositoryUrl: string;
    cicdUrl: string;
    dashboardUrl: string;
  }> {
    const template = this.templates.find(t => t.id === request.templateId);
    
    if (!template) {
      throw new Error(`Template ${request.templateId} not found`);
    }

    this.logger.log(`Creating service ${request.serviceName} from template ${request.templateId}`);

    // 1. สร้าง Git Repository
    const repositoryUrl = await this.createGitRepository(request.serviceName, template);
    
    // 2. สร้าง CI/CD Pipeline
    const cicdUrl = await this.createCICDPipeline(request.serviceName, request.teamName);
    
    // 3. สร้าง Kubernetes Namespace และ Resources
    await this.createKubernetesResources(request.serviceName, request.teamName, request.environment);
    
    // 4. สร้าง Secrets ใน Vault
    await this.createVaultSecrets(request.serviceName, request.environment, request.config);
    
    // 5. สร้าง Monitoring Dashboard
    const dashboardUrl = await this.createMonitoringDashboard(request.serviceName);
    
    // 6. ลงทะเบียนใน Service Catalog
    await this.registerInServiceCatalog({
      name: request.serviceName,
      team: request.teamName,
      template: request.templateId,
      repositoryUrl,
    });

    return { repositoryUrl, cicdUrl, dashboardUrl };
  }

  private async createGitRepository(
    serviceName: string,
    template: ServiceTemplate,
  ): Promise<string> {
    // Clone template และ customize
    // เรียก GitHub API สร้าง Repository
    const repoUrl = `https://github.com/your-org/${serviceName}`;
    this.logger.log(`Created repository: ${repoUrl}`);
    return repoUrl;
  }

  private async createKubernetesResources(
    serviceName: string,
    teamName: string,
    environment: string,
  ): Promise<void> {
    // สร้าง Namespace, ResourceQuota, NetworkPolicy, RBAC
    const manifests = this.generateKubernetesManifests(serviceName, teamName, environment);
    
    // Apply ผ่าน kubectl หรือ Kubernetes API
    this.logger.log(`Created Kubernetes resources for ${serviceName}`);
  }

  private generateKubernetesManifests(
    serviceName: string,
    teamName: string,
    environment: string,
  ): string {
    return `
apiVersion: v1
kind: Namespace
metadata:
  name: ${environment}-${teamName}
  labels:
    team: ${teamName}
    environment: ${environment}
    managed-by: idp
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: ${teamName}-quota
  namespace: ${environment}-${teamName}
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "50"
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: ${environment}-${teamName}
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
`;
  }

  private async createVaultSecrets(
    serviceName: string,
    environment: string,
    config: Record<string, string>,
  ): Promise<void> {
    // บันทึก Secrets ใน HashiCorp Vault
    this.logger.log(`Created Vault secrets for ${serviceName}`);
  }

  private async createCICDPipeline(
    serviceName: string,
    teamName: string,
  ): Promise<string> {
    const pipelineUrl = `https://ci.your-org.com/${teamName}/${serviceName}`;
    this.logger.log(`Created CI/CD pipeline: ${pipelineUrl}`);
    return pipelineUrl;
  }

  private async createMonitoringDashboard(serviceName: string): Promise<string> {
    const dashboardUrl = `https://grafana.your-org.com/d/${serviceName}`;
    this.logger.log(`Created monitoring dashboard: ${dashboardUrl}`);
    return dashboardUrl;
  }

  private async registerInServiceCatalog(serviceInfo: any): Promise<void> {
    // บันทึกใน Backstage Service Catalog
    this.logger.log(`Registered ${serviceInfo.name} in service catalog`);
  }
}
```

---

## 11. FinOps Practices

```typescript
// finops/src/cost-monitor.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { Cron } from '@nestjs/schedule';

export interface ServiceCost {
  serviceName: string;
  teamName: string;
  compute: number;
  storage: number;
  network: number;
  total: number;
  currency: string;
  period: string;
}

export interface CostAnomaly {
  serviceName: string;
  currentCost: number;
  expectedCost: number;
  variance: number;
  severity: 'LOW' | 'MEDIUM' | 'HIGH';
}

@Injectable()
export class FinOpsService {
  private readonly logger = new Logger(FinOpsService.name);

  @Cron('0 8 * * 1') // ทุกวันจันทร์ 08:00
  async generateWeeklyCostReport(): Promise<void> {
    const costs = await this.getServiceCosts('weekly');
    const anomalies = await this.detectCostAnomalies(costs);
    
    // ส่ง Report ไปยัง Teams
    for (const cost of costs) {
      await this.sendCostReport(cost);
    }

    if (anomalies.length > 0) {
      await this.sendCostAnomalyAlert(anomalies);
    }
  }

  async getServiceCosts(period: 'daily' | 'weekly' | 'monthly'): Promise<ServiceCost[]> {
    // ดึงข้อมูลจาก AWS Cost Explorer / GCP Billing
    // หรือ Kubecost API
    return [];
  }

  async detectCostAnomalies(costs: ServiceCost[]): Promise<CostAnomaly[]> {
    const anomalies: CostAnomaly[] = [];
    
    for (const cost of costs) {
      const historicalAvg = await this.getHistoricalAverage(cost.serviceName);
      const variance = ((cost.total - historicalAvg) / historicalAvg) * 100;
      
      if (Math.abs(variance) > 20) {
        anomalies.push({
          serviceName: cost.serviceName,
          currentCost: cost.total,
          expectedCost: historicalAvg,
          variance,
          severity: Math.abs(variance) > 50 ? 'HIGH' : Math.abs(variance) > 30 ? 'MEDIUM' : 'LOW',
        });
      }
    }
    
    return anomalies;
  }

  private async getHistoricalAverage(serviceName: string): Promise<number> {
    // ดึง Average ย้อนหลัง 4 สัปดาห์
    return 100; // Mock
  }

  private async sendCostReport(cost: ServiceCost): Promise<void> {
    this.logger.log(`Sending cost report for ${cost.serviceName}: $${cost.total}`);
  }

  private async sendCostAnomalyAlert(anomalies: CostAnomaly[]): Promise<void> {
    this.logger.warn(`Cost anomalies detected: ${anomalies.length}`);
  }
}
```

---

## 12. Learning Path Recommendations

```
Microservices Engineer Learning Path 2024-2025
═══════════════════════════════════════════════

Level 1: Foundation (1-3 เดือน)
├── Docker & Containers
├── Kubernetes Basics
├── REST API Design
├── Basic Messaging (Kafka/RabbitMQ)
└── CI/CD with GitHub Actions

Level 2: Intermediate (3-6 เดือน)
├── Service Mesh (Istio/Linkerd)
├── Observability (Prometheus + Grafana + Jaeger)
├── API Gateway (Kong/Traefik)
├── Security (mTLS, JWT, OIDC)
└── Database Patterns (CQRS, Event Sourcing)

Level 3: Advanced (6-12 เดือน)
├── Platform Engineering & IDP
├── GitOps (ArgoCD/Flux)
├── Progressive Delivery (Canary/Blue-Green)
├── Chaos Engineering
└── Performance Optimization

Level 4: Expert (12+ เดือน)
├── eBPF & Cilium
├── WebAssembly (WASM)
├── Dapr Framework
├── ML/AI Integration
└── FinOps & Cost Optimization
```

---

## สรุปบทที่ 96

| เทคโนโลยี | ระดับ Maturity | Use Case |
|-----------|---------------|---------|
| **Kubernetes** | Production-ready | Container Orchestration |
| **eBPF/Cilium** | Growing | Network + Observability |
| **WASM** | Early Adopter | Edge Computing + Plugins |
| **Dapr** | Growing | Cloud-agnostic Microservices |
| **Ambient Mesh** | Beta | Service Mesh ไม่มี Sidecar |
| **Knative** | Stable | Serverless on Kubernetes |
| **GitOps** | Mature | Deployment Automation |
| **Platform Engineering** | Growing | Developer Self-service |
| **FinOps** | Growing | Cost Awareness |
| **AI/ML Integration** | Rapidly Growing | Smart Services |
| **Edge Computing** | Growing | Low-latency Global Apps |

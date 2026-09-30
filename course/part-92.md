# Part 92: Case Study: E-Commerce Platform (Thai)

## บทนำ

ในบทนี้เราจะออกแบบ E-Commerce Platform สำหรับตลาดไทย คล้ายกับ Shopee หรือ Lazada ตั้งแต่การแบ่ง Service, การเลือก Technology Stack, กลยุทธ์การ Scale และการคำนวณต้นทุน

---

## 1. Requirements Analysis

### 1.1 Functional Requirements

```
ฟีเจอร์หลัก:
✅ สมัครสมาชิก/เข้าสู่ระบบ (LINE Login, Facebook, Email)
✅ ค้นหาสินค้า (Full-text, Filter, Sort)
✅ ดูรายละเอียดสินค้า (รูปภาพ, รีวิว, ราคา)
✅ ตะกร้าสินค้า (Shopping Cart)
✅ สั่งซื้อสินค้า
✅ ชำระเงิน (PromptPay, บัตรเครดิต, COD, Shopee Pay)
✅ ติดตามสถานะออเดอร์
✅ ระบบรีวิว/Rating
✅ Flash Sale / โปรโมชั่น
✅ Seller Dashboard
✅ Admin Panel
✅ Notification (Push, Email, SMS, LINE)
```

### 1.2 Non-Functional Requirements

```
Scale Requirements (ระดับ Shopee Thailand):
┌─────────────────────────────────────────────────────┐
│ Metric              │ Normal    │ Peak (11.11)       │
├─────────────────────┼───────────┼────────────────────┤
│ DAU                 │ 5M        │ 15M                │
│ Concurrent Users    │ 500K      │ 3M                 │
│ Product Listings    │ 50M       │ 50M                │
│ Orders/day          │ 500K      │ 5M                 │
│ Order Peak TPS      │ 5,800     │ 58,000             │
│ Search QPS          │ 10,000    │ 50,000             │
│ Image Storage       │ 500TB     │ 500TB              │
│ API Latency (p99)   │ < 200ms   │ < 500ms            │
│ Availability        │ 99.99%    │ 99.99%             │
└─────────────────────────────────────────────────────┘
```

---

## 2. Service Decomposition

### 2.1 Domain-Driven Design ก่อน Decompose

```
Bounded Contexts:
┌────────────────────────────────────────────────────────────────┐
│                      E-Commerce Domain                          │
├────────────────┬───────────────┬───────────────┬───────────────┤
│  User Domain   │ Product Domain│ Order Domain  │Payment Domain │
│                │               │               │               │
│ - Registration │ - Catalog     │ - Cart        │ - Payment     │
│ - Profile      │ - Inventory   │ - Order       │ - Wallet      │
│ - Auth         │ - Search      │ - Fulfillment │ - Refund      │
│ - Address      │ - Review      │ - Tracking    │ - Voucher     │
└────────────────┴───────────────┴───────────────┴───────────────┘
         │                │                │              │
    ┌────▼──────┐   ┌──────▼────┐   ┌──────▼────┐  ┌────▼──────┐
    │Notification│   │Seller     │   │Logistics  │  │Analytics  │
    │Domain      │   │Domain     │   │Domain     │  │Domain     │
    │            │   │           │   │           │  │           │
    │- Email     │   │- Shop Mgmt│   │- Shipping │  │- Metrics  │
    │- SMS       │   │- Products │   │- Tracking │  │- Reports  │
    │- Push      │   │- Finance  │   │- Returns  │  │- BI       │
    └────────────┘   └───────────┘   └───────────┘  └───────────┘
```

### 2.2 Microservices Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Client Layer                                   │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌──────────────┐  │
│  │Mobile App  │  │Web App     │  │Seller App  │  │Admin Panel   │  │
│  │(React Native│  │(Next.js)  │  │(React)     │  │(React)       │  │
│  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘  └──────┬───────┘  │
└────────┼───────────────┼───────────────┼────────────────┼──────────┘
         └───────────────┴───────────────┴────────────────┘
                                  │
                    ┌─────────────▼─────────────┐
                    │      Kong API Gateway       │
                    │  - JWT Authentication        │
                    │  - Rate Limiting (Redis)     │
                    │  - Request Routing           │
                    │  - SSL Termination           │
                    │  - Request/Response Transform│
                    └─────────────┬───────────────┘
                                  │
          ┌───────────────────────┼──────────────────────────┐
          │                       │                          │
    ┌─────▼──────┐      ┌─────────▼────────┐      ┌─────────▼───────┐
    │User Service │      │Product Service   │      │Order Service    │
    │:8001        │      │:8002             │      │:8003            │
    │             │      │                  │      │                 │
    │PostgreSQL   │      │MySQL+Elastic     │      │PostgreSQL       │
    │Redis(session│      │Redis(cache)      │      │Kafka(events)    │
    └─────────────┘      └──────────────────┘      └─────────────────┘
          │                       │                          │
    ┌─────▼──────┐      ┌─────────▼────────┐      ┌─────────▼───────┐
    │Payment Svc  │      │Inventory Service │      │Search Service   │
    │:8004        │      │:8005             │      │:8006            │
    │             │      │                  │      │                 │
    │PostgreSQL   │      │PostgreSQL+Redis  │      │Elasticsearch    │
    │(encrypted)  │      │(dist. lock)      │      │                 │
    └─────────────┘      └──────────────────┘      └─────────────────┘
          │                       │                          │
    ┌─────▼──────┐      ┌─────────▼────────┐      ┌─────────▼───────┐
    │Notification │      │Review Service    │      │Recommendation   │
    │Service:8007 │      │:8008             │      │Service:8009     │
    │             │      │                  │      │                 │
    │Kafka        │      │MongoDB           │      │MongoDB+Redis    │
    │Firebase/LINE│      │                  │      │(ML Model)       │
    └─────────────┘      └──────────────────┘      └─────────────────┘
```

---

## 3. Technology Stack Choices

### 3.1 Backend Services

```yaml
# technology-stack.yaml
services:
  user-service:
    language: Go
    reason: "High concurrency, Low latency, Small binary"
    database:
      primary: PostgreSQL 15
      cache: Redis 7
      reason: "ACID transactions, rich query support"
    
  product-service:
    language: Go
    reason: "High read throughput"
    database:
      primary: MySQL 8 (write)
      search: Elasticsearch 8 (read/search)
      cache: Redis (product detail cache, 10min TTL)
      cdn: CloudFront (product images)
    
  order-service:
    language: Go
    reason: "Saga pattern, complex business logic"
    database:
      primary: PostgreSQL 15
      events: Kafka (order events)
    patterns:
      - Saga Orchestration
      - Outbox Pattern
    
  payment-service:
    language: Java (Spring Boot)
    reason: "Mature financial libraries, PCI compliance"
    database:
      primary: PostgreSQL 15 (encrypted columns)
    external:
      - Omise (credit card)
      - PromptPay (SCB/KBank API)
      - TrueWallet
    
  inventory-service:
    language: Go
    reason: "High concurrency for flash sales"
    database:
      primary: PostgreSQL 15
      cache: Redis (stock cache, Lua scripts for atomic operations)
    
  search-service:
    language: Python (FastAPI)
    reason: "ML/NLP libraries available"
    database:
      search: Elasticsearch 8
      autocomplete: Redis (sorted sets)
    features:
      - Thai language tokenization (PyThaiNLP)
      - Semantic search
      - Filters and facets
    
  notification-service:
    language: Go
    database:
      queue: Kafka
    channels:
      - Firebase FCM (Push notification)
      - LINE Notify
      - AWS SES (Email)
      - Twilio (SMS)
    
  analytics-service:
    language: Python
    database:
      warehouse: BigQuery
      streaming: Kafka + Spark Streaming
      visualization: Metabase
```

### 3.2 Infrastructure

```yaml
# infrastructure.yaml
infrastructure:
  container_orchestration: Kubernetes (EKS)
  service_mesh: Istio
  api_gateway: Kong
  
  databases:
    postgresql:
      setup: RDS Multi-AZ
      version: "15"
      backup: Daily snapshots + WAL archiving
    
    redis:
      setup: ElastiCache Cluster Mode
      nodes: 6 (3 shards × 2 replicas)
    
    elasticsearch:
      setup: Amazon OpenSearch
      nodes: 6 (3 data + 3 master)
      replicas: 1
    
    kafka:
      setup: MSK (Managed Streaming)
      brokers: 3
      topics:
        - order.created (100 partitions)
        - payment.processed (50 partitions)
        - inventory.updated (50 partitions)
        - notification.send (100 partitions)
  
  cdn: CloudFront (Product images, Static assets)
  
  storage:
    product_images: S3 (+ CloudFront CDN)
    videos: S3 (+ Transcoding via MediaConvert)
  
  monitoring:
    metrics: Prometheus + Grafana
    tracing: Jaeger (OpenTelemetry)
    logging: ELK Stack (Elasticsearch, Logstash, Kibana)
    alerting: PagerDuty
  
  ci_cd:
    pipeline: GitHub Actions
    registry: ECR
    deployment: ArgoCD (GitOps)
```

---

## 4. Core Service Implementation

### 4.1 Product Service

```go
// product-service/internal/handler/product.go
package handler

import (
    "context"
    "encoding/json"
    "time"
    
    "github.com/shopee-th/product-service/internal/domain"
    "github.com/shopee-th/product-service/internal/repository"
    "github.com/redis/go-redis/v9"
    "go.opentelemetry.io/otel"
)

type ProductHandler struct {
    repo  repository.ProductRepository
    cache *redis.Client
    search SearchService
}

func (h *ProductHandler) GetProduct(ctx context.Context, req *GetProductRequest) (*Product, error) {
    tracer := otel.Tracer("product-service")
    ctx, span := tracer.Start(ctx, "GetProduct")
    defer span.End()

    // L1: Check Redis Cache
    cacheKey := fmt.Sprintf("product:%s", req.ProductID)
    cached, err := h.cache.Get(ctx, cacheKey).Bytes()
    if err == nil {
        var product domain.Product
        if err := json.Unmarshal(cached, &product); err == nil {
            span.SetAttributes(attribute.Bool("cache.hit", true))
            return toProto(&product), nil
        }
    }

    // L2: Database
    product, err := h.repo.FindByID(ctx, req.ProductID)
    if err != nil {
        return nil, status.Errorf(codes.NotFound, "product not found: %v", err)
    }

    // Store in cache (10 minutes TTL)
    if data, err := json.Marshal(product); err == nil {
        h.cache.Set(ctx, cacheKey, data, 10*time.Minute)
    }

    span.SetAttributes(attribute.Bool("cache.hit", false))
    return toProto(product), nil
}

// Flash Sale: Inventory Check with Distributed Lock
func (h *ProductHandler) PurchaseFlashSale(ctx context.Context, req *PurchaseRequest) (*PurchaseResponse, error) {
    lockKey := fmt.Sprintf("flashsale:lock:%s", req.ProductID)
    lock, err := h.distLock.Acquire(ctx, lockKey, 5*time.Second)
    if err != nil {
        return nil, status.Error(codes.ResourceExhausted, "too many requests")
    }
    defer lock.Release(ctx)

    // Atomic check and decrement with Lua script
    script := redis.NewScript(`
        local stock = tonumber(redis.call('GET', KEYS[1]))
        if stock == nil or stock <= 0 then
            return -1
        end
        return redis.call('DECRBY', KEYS[1], tonumber(ARGV[1]))
    `)

    stockKey := fmt.Sprintf("flashsale:stock:%s", req.ProductID)
    remaining, err := script.Run(ctx, h.cache, []string{stockKey}, req.Quantity).Int64()
    if err != nil || remaining < 0 {
        return nil, status.Error(codes.ResourceExhausted, "out of stock")
    }

    return &PurchaseResponse{Success: true, Remaining: remaining}, nil
}
```

### 4.2 Order Service (Saga Pattern)

```go
// order-service/internal/saga/create_order_saga.go
package saga

import (
    "context"
    "encoding/json"
    "time"
    
    "github.com/google/uuid"
)

type OrderSagaState struct {
    OrderID      string
    UserID       string
    Items        []OrderItem
    TotalAmount  float64
    
    // Saga state
    PaymentID    string
    ReservationID string
    Status       SagaStatus
}

type CreateOrderSaga struct {
    orderRepo    OrderRepository
    paymentSvc   PaymentServiceClient
    inventorySvc InventoryServiceClient
    notifySvc    NotificationServiceClient
    publisher    EventPublisher
}

func (s *CreateOrderSaga) Execute(ctx context.Context, req CreateOrderRequest) (*Order, error) {
    sagaID := uuid.New().String()
    state := &OrderSagaState{
        OrderID:     uuid.New().String(),
        UserID:      req.UserID,
        Items:       req.Items,
        TotalAmount: req.TotalAmount,
        Status:      SagaStarted,
    }

    // Step 1: Create Order (PENDING)
    order, err := s.orderRepo.Create(ctx, &Order{
        ID:          state.OrderID,
        UserID:      state.UserID,
        Items:       state.Items,
        TotalAmount: state.TotalAmount,
        Status:      OrderPending,
    })
    if err != nil {
        return nil, fmt.Errorf("create order failed: %w", err)
    }

    // Step 2: Reserve Inventory
    reservation, err := s.inventorySvc.Reserve(ctx, &ReserveRequest{
        OrderID: state.OrderID,
        Items:   state.Items,
    })
    if err != nil {
        // Compensate: Cancel Order
        s.orderRepo.UpdateStatus(ctx, state.OrderID, OrderCancelled)
        return nil, fmt.Errorf("inventory reservation failed: %w", err)
    }
    state.ReservationID = reservation.ReservationID

    // Step 3: Process Payment
    payment, err := s.paymentSvc.Charge(ctx, &ChargeRequest{
        OrderID:  state.OrderID,
        UserID:   state.UserID,
        Amount:   state.TotalAmount,
        Method:   req.PaymentMethod,
    })
    if err != nil {
        // Compensate: Release Inventory + Cancel Order
        s.inventorySvc.Release(ctx, state.ReservationID)
        s.orderRepo.UpdateStatus(ctx, state.OrderID, OrderCancelled)
        return nil, fmt.Errorf("payment failed: %w", err)
    }
    state.PaymentID = payment.PaymentID

    // Step 4: Confirm Order
    if err := s.orderRepo.UpdateStatus(ctx, state.OrderID, OrderConfirmed); err != nil {
        return nil, err
    }

    // Step 5: Publish Events (non-critical, async)
    s.publisher.Publish(ctx, "order.confirmed", OrderConfirmedEvent{
        OrderID:   state.OrderID,
        UserID:    state.UserID,
        PaymentID: state.PaymentID,
    })

    return order, nil
}
```

### 4.3 Search Service

```python
# search-service/app/services/product_search.py
from elasticsearch import AsyncElasticsearch
from pythainlp.tokenize import word_tokenize
from typing import List, Optional
import asyncio

class ProductSearchService:
    def __init__(self, es: AsyncElasticsearch):
        self.es = es
        self.index = "products"
    
    async def search(
        self,
        query: str,
        category: Optional[str] = None,
        min_price: Optional[float] = None,
        max_price: Optional[float] = None,
        sort_by: str = "relevance",
        page: int = 1,
        page_size: int = 20
    ) -> SearchResult:
        # Thai word tokenization
        tokens = word_tokenize(query, engine="newmm")
        thai_query = " ".join(tokens)
        
        must_queries = [
            {
                "multi_match": {
                    "query": thai_query,
                    "fields": [
                        "name^3",          # Title has highest weight
                        "name.thai^3",     # Thai analyzed
                        "description^1",
                        "brand^2",
                        "category.name^2",
                        "tags^1"
                    ],
                    "type": "best_fields",
                    "fuzziness": "AUTO"
                }
            }
        ]
        
        filter_queries = []
        
        if category:
            filter_queries.append({"term": {"category.slug": category}})
        
        if min_price is not None or max_price is not None:
            price_range = {}
            if min_price is not None:
                price_range["gte"] = min_price
            if max_price is not None:
                price_range["lte"] = max_price
            filter_queries.append({"range": {"price": price_range}})
        
        # Always filter active products with stock
        filter_queries.append({"term": {"status": "active"}})
        filter_queries.append({"range": {"stock": {"gt": 0}}})
        
        sort = self._build_sort(sort_by)
        
        query_body = {
            "query": {
                "bool": {
                    "must": must_queries,
                    "filter": filter_queries
                }
            },
            "aggs": {
                "categories": {
                    "terms": {"field": "category.slug", "size": 10}
                },
                "price_ranges": {
                    "range": {
                        "field": "price",
                        "ranges": [
                            {"to": 100},
                            {"from": 100, "to": 500},
                            {"from": 500, "to": 1000},
                            {"from": 1000}
                        ]
                    }
                }
            },
            "highlight": {
                "fields": {
                    "name": {},
                    "description": {"fragment_size": 150}
                }
            },
            "from": (page - 1) * page_size,
            "size": page_size,
            "sort": sort
        }
        
        response = await self.es.search(index=self.index, body=query_body)
        return self._parse_response(response, page, page_size)
    
    def _build_sort(self, sort_by: str) -> list:
        sorts = {
            "relevance": [{"_score": "desc"}],
            "price_asc": [{"price": "asc"}, {"_score": "desc"}],
            "price_desc": [{"price": "desc"}, {"_score": "desc"}],
            "newest": [{"created_at": "desc"}],
            "popular": [{"sales_count": "desc"}, {"_score": "desc"}],
            "rating": [{"rating": "desc"}, {"_score": "desc"}]
        }
        return sorts.get(sort_by, sorts["relevance"])
    
    async def autocomplete(self, prefix: str, size: int = 10) -> List[str]:
        response = await self.es.search(
            index=self.index,
            body={
                "suggest": {
                    "product_suggest": {
                        "prefix": prefix,
                        "completion": {
                            "field": "suggest",
                            "size": size,
                            "fuzzy": {"fuzziness": 1}
                        }
                    }
                }
            }
        )
        
        suggestions = response["suggest"]["product_suggest"][0]["options"]
        return [s["_source"]["name"] for s in suggestions]
```

### 4.4 Flash Sale Service

```go
// flash-sale-service/internal/service/flash_sale.go
package service

import (
    "context"
    "fmt"
    "time"
    
    "github.com/redis/go-redis/v9"
)

// Flash Sale Architecture:
// ┌─────────────────────────────────────────────────────────┐
// │                    Flash Sale Flow                       │
// │                                                          │
// │ Pre-load stock to Redis before sale starts              │
// │ During sale: Atomic decrement via Lua Script            │
// │ Post-sale: Sync Redis → Database                        │
// └─────────────────────────────────────────────────────────┘

type FlashSaleService struct {
    redis  *redis.ClusterClient
    repo   FlashSaleRepository
    queue  EventPublisher
}

// Pre-load stock before flash sale
func (s *FlashSaleService) PreloadStock(ctx context.Context, saleID string, items []SaleItem) error {
    pipe := s.redis.Pipeline()
    
    for _, item := range items {
        stockKey := fmt.Sprintf("fs:stock:%s:%s", saleID, item.ProductID)
        limitKey := fmt.Sprintf("fs:limit:%s:%s", saleID, item.ProductID)
        
        pipe.Set(ctx, stockKey, item.Stock, 24*time.Hour)
        pipe.Set(ctx, limitKey, item.LimitPerUser, 24*time.Hour)
    }
    
    _, err := pipe.Exec(ctx)
    return err
}

var purchaseScript = redis.NewScript(`
    local stock_key = KEYS[1]
    local user_key = KEYS[2]
    local limit_key = KEYS[3]
    
    -- Check user purchase limit
    local purchased = tonumber(redis.call('GET', user_key) or 0)
    local limit = tonumber(redis.call('GET', limit_key) or 1)
    
    if purchased >= limit then
        return {-1, "EXCEEDED_LIMIT"}
    end
    
    -- Check and decrement stock
    local stock = tonumber(redis.call('GET', stock_key) or 0)
    if stock <= 0 then
        return {-2, "OUT_OF_STOCK"}
    end
    
    -- Atomic update
    redis.call('DECRBY', stock_key, ARGV[1])
    redis.call('INCRBY', user_key, ARGV[1])
    redis.call('EXPIRE', user_key, 86400)
    
    return {stock - tonumber(ARGV[1]), "SUCCESS"}
`)

func (s *FlashSaleService) Purchase(ctx context.Context, req PurchaseRequest) (*PurchaseResult, error) {
    stockKey := fmt.Sprintf("fs:stock:%s:%s", req.SaleID, req.ProductID)
    userKey := fmt.Sprintf("fs:user:%s:%s:%s", req.SaleID, req.ProductID, req.UserID)
    limitKey := fmt.Sprintf("fs:limit:%s:%s", req.SaleID, req.ProductID)
    
    result, err := purchaseScript.Run(
        ctx, s.redis,
        []string{stockKey, userKey, limitKey},
        req.Quantity,
    ).Slice()
    
    if err != nil {
        return nil, err
    }
    
    code := result[1].(string)
    switch code {
    case "EXCEEDED_LIMIT":
        return nil, ErrExceededLimit
    case "OUT_OF_STOCK":
        return nil, ErrOutOfStock
    }
    
    remaining := result[0].(int64)
    
    // Async: Create order
    s.queue.Publish(ctx, "flashsale.purchased", FlashSalePurchasedEvent{
        SaleID:    req.SaleID,
        ProductID: req.ProductID,
        UserID:    req.UserID,
        Quantity:  req.Quantity,
    })
    
    return &PurchaseResult{
        Success:   true,
        Remaining: remaining,
    }, nil
}
```

---

## 5. Scaling Strategy

### 5.1 Traffic Pattern Analysis

```
Thai E-Commerce Traffic Patterns:

Daily Pattern:
12AM  ━━━━━━━━━━━━━━━━━━━
6AM   ━━━━━━━━━━━━━━━━━━━━━━
9AM   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
12PM  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
3PM   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
6PM   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
9PM   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
11PM  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
       (Peak ช่วง 9PM - Flash sales, after work)

Annual Events:
11.11 Single Day Sale    → 10x normal traffic
12.12 Year-end Sale      → 8x normal traffic
สงกรานต์                 → 3x normal traffic
วาเลนไทน์               → 2x normal traffic
```

### 5.2 Auto-scaling Strategy

```yaml
# k8s/hpa-order-service.yaml
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
  minReplicas: 10
  maxReplicas: 200
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
        type: Utilization
        averageUtilization: 80
  - type: External
    external:
      metric:
        name: kafka_consumer_lag
        selector:
          matchLabels:
            topic: order.created
      target:
        type: AverageValue
        averageValue: "1000"
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Pods
        value: 20         # Scale up 20 pods at a time
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5min before scale down
      policies:
      - type: Pods
        value: 5
        periodSeconds: 60

---
# KEDA for Kafka-based autoscaling
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: notification-service-scaler
spec:
  scaleTargetRef:
    name: notification-service
  minReplicaCount: 5
  maxReplicaCount: 100
  triggers:
  - type: kafka
    metadata:
      bootstrapServers: kafka:9092
      consumerGroup: notification-group
      topic: notification.send
      lagThreshold: "500"
```

### 5.3 Caching Strategy

```
Cache Layers:
┌────────────────────────────────────────────────────┐
│                  Caching Strategy                   │
├──────────────────┬─────────────────────────────────┤
│ Layer            │ What to Cache                   │
├──────────────────┼─────────────────────────────────┤
│ CDN (CloudFront) │ Static assets, Product images   │
│                  │ TTL: 7 days                     │
├──────────────────┼─────────────────────────────────┤
│ API Response     │ Product list, Category tree      │
│ (Nginx cache)    │ TTL: 1 minute                   │
├──────────────────┼─────────────────────────────────┤
│ Redis            │ Product details: 10min           │
│                  │ User session: 24hr               │
│                  │ Cart: 7 days                     │
│                  │ Flash sale stock: real-time      │
│                  │ Search results: 5min             │
├──────────────────┼─────────────────────────────────┤
│ Application      │ Config, Feature flags            │
│ (In-process)     │ TTL: 5 minutes                  │
└──────────────────┴─────────────────────────────────┘

Cache Invalidation Strategy:
- Product update → Pub/Sub → Invalidate all product caches
- Flash sale start → Pre-warm cache with sale data
- Inventory change → Invalidate product stock cache

// Cache warming on startup
func (s *ProductService) WarmCache(ctx context.Context) {
    // Get top 10,000 products by view count
    products, _ := s.repo.GetTopProducts(ctx, 10000)
    
    pipe := s.redis.Pipeline()
    for _, p := range products {
        data, _ := json.Marshal(p)
        pipe.Set(ctx, "product:"+p.ID, data, 10*time.Minute)
    }
    pipe.Exec(ctx)
    
    log.Info("Cache warmed with top products", "count", len(products))
}
```

---

## 6. Cost Breakdown

### 6.1 AWS Infrastructure Cost (Monthly)

```
สำหรับระดับ ~5M DAU, 500K orders/day

Compute (EKS):
┌──────────────────────────────────────────────────────────────┐
│ Service           │ Nodes      │ Instance    │ Cost/month     │
├──────────────────┼────────────┼─────────────┼────────────────┤
│ Core Services    │ 20 nodes   │ m5.2xlarge  │ ~$3,400        │
│ (order,user,pay) │            │ (8vCPU,32GB)│                │
├──────────────────┼────────────┼─────────────┼────────────────┤
│ Search Service   │ 5 nodes    │ r5.2xlarge  │ ~$1,200        │
│                  │            │ (8vCPU,64GB)│                │
├──────────────────┼────────────┼─────────────┼────────────────┤
│ Notification Svc │ 10 nodes   │ m5.xlarge   │ ~$850          │
│                  │            │ (4vCPU,16GB)│                │
└──────────────────┴────────────┴─────────────┴────────────────┘
Total Compute: ~$5,450/month

Database:
┌──────────────────────────────────────────────────────────────┐
│ Database          │ Setup      │ Cost/month                   │
├──────────────────┼────────────┼──────────────────────────────┤
│ RDS PostgreSQL   │ Multi-AZ   │ ~$2,000                      │
│ (Order/User/Pay) │ r5.2xlarge │                              │
├──────────────────┼────────────┼──────────────────────────────┤
│ ElastiCache Redis│ 6 nodes    │ ~$1,500                      │
│                  │ r6g.xlarge │                              │
├──────────────────┼────────────┼──────────────────────────────┤
│ Amazon OpenSearch│ 6 nodes    │ ~$2,400                      │
│                  │ r5.xlarge  │                              │
├──────────────────┼────────────┼──────────────────────────────┤
│ MSK (Kafka)      │ 3 brokers  │ ~$800                        │
└──────────────────┴────────────┴──────────────────────────────┘
Total Database: ~$6,700/month

Storage & CDN:
┌──────────────────────────────────────────────────────────────┐
│ S3 Storage (500TB)               │ ~$11,500/month            │
│ CloudFront (100TB transfer)      │ ~$8,500/month             │
│ Data Transfer                    │ ~$2,000/month             │
└──────────────────────────────────┴───────────────────────────┘
Total Storage/CDN: ~$22,000/month

Monitoring & Tools:
- ELK Stack: ~$1,500/month
- Datadog: ~$2,000/month  
- PagerDuty: ~$500/month
Total: ~$4,000/month

GRAND TOTAL: ~$38,150/month (≈ 1.4M THB/month)

Flash Sale Day (11.11) Surge Cost:
- Compute scale up: +$5,000 (1 day)
- Database: +$500 (read replicas)
- CDN: +$2,000 (traffic spike)
Daily 11.11 Extra: ~$7,500 (≈ 280,000 THB)
```

### 6.2 Cost Optimization Strategies

```
Cost Optimization Techniques:

1. Spot Instances สำหรับ Batch Jobs
   - Analytics processing: ใช้ Spot (ประหยัด 70%)
   - Non-critical workers: ใช้ Spot
   - ประหยัดได้: ~$2,000/month

2. Reserved Instances สำหรับ Stable Workloads
   - Database servers: 1-year Reserved (ประหยัด 40%)
   - Core API servers: 1-year Reserved
   - ประหยัดได้: ~$3,000/month

3. S3 Intelligent-Tiering
   - Product images ที่ไม่ค่อยเรียกใช้ → Infrequent Access
   - ประหยัดได้: ~$2,000/month

4. CDN Optimization
   - Compress images (WebP format): -40% bandwidth
   - Lazy loading: -20% bandwidth
   - ประหยัดได้: ~$1,700/month

5. Database Optimization
   - Read Replicas แทน Vertical Scaling
   - Connection Pooling (PgBouncer)
   - Query Optimization
   - ประหยัดได้: ~$1,000/month

Total Monthly Savings: ~$9,700
Optimized Monthly Cost: ~$28,450 (≈ 1M THB/month)
```

---

## 7. Database Schema Design

### 7.1 Product Schema

```sql
-- products table
CREATE TABLE products (
    id              UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    seller_id       UUID NOT NULL REFERENCES sellers(id),
    name            VARCHAR(500) NOT NULL,
    slug            VARCHAR(500) UNIQUE NOT NULL,
    description     TEXT,
    category_id     UUID NOT NULL REFERENCES categories(id),
    brand_id        UUID REFERENCES brands(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'draft', -- draft, active, inactive
    created_at      TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at      TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    INDEX idx_seller_id (seller_id),
    INDEX idx_category_id (category_id),
    INDEX idx_status (status),
    INDEX idx_created_at (created_at DESC)
);

-- product_variants (SKU)
CREATE TABLE product_variants (
    id          UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    product_id  UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    sku         VARCHAR(100) UNIQUE NOT NULL,
    name        VARCHAR(200),  -- "สีแดง, Size M"
    price       DECIMAL(12,2) NOT NULL,
    compare_at_price DECIMAL(12,2),
    stock       INTEGER NOT NULL DEFAULT 0,
    weight      DECIMAL(8,2),  -- kg
    
    CONSTRAINT check_price CHECK (price >= 0),
    CONSTRAINT check_stock CHECK (stock >= 0),
    INDEX idx_product_id (product_id),
    INDEX idx_sku (sku)
);

-- orders table
CREATE TABLE orders (
    id              UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    order_number    VARCHAR(50) UNIQUE NOT NULL,  -- TH20241111-00001
    user_id         UUID NOT NULL REFERENCES users(id),
    seller_id       UUID NOT NULL REFERENCES sellers(id),
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
    subtotal        DECIMAL(12,2) NOT NULL,
    shipping_fee    DECIMAL(10,2) NOT NULL DEFAULT 0,
    discount        DECIMAL(10,2) NOT NULL DEFAULT 0,
    total           DECIMAL(12,2) NOT NULL,
    shipping_address_id UUID REFERENCES user_addresses(id),
    payment_method  VARCHAR(50),
    payment_status  VARCHAR(30) DEFAULT 'pending',
    notes           TEXT,
    created_at      TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at      TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    confirmed_at    TIMESTAMP WITH TIME ZONE,
    shipped_at      TIMESTAMP WITH TIME ZONE,
    delivered_at    TIMESTAMP WITH TIME ZONE,
    
    INDEX idx_user_id (user_id),
    INDEX idx_seller_id (seller_id),
    INDEX idx_status (status),
    INDEX idx_created_at (created_at DESC),
    INDEX idx_order_number (order_number)
) PARTITION BY RANGE (created_at);

-- Partition by month for performance
CREATE TABLE orders_2024_01 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
CREATE TABLE orders_2024_02 PARTITION OF orders
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');
-- ... continue monthly

-- order_items table
CREATE TABLE order_items (
    id              UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    order_id        UUID NOT NULL REFERENCES orders(id),
    product_id      UUID NOT NULL REFERENCES products(id),
    variant_id      UUID NOT NULL REFERENCES product_variants(id),
    quantity        INTEGER NOT NULL,
    unit_price      DECIMAL(12,2) NOT NULL,
    discount        DECIMAL(10,2) DEFAULT 0,
    total           DECIMAL(12,2) NOT NULL,
    
    CONSTRAINT check_quantity CHECK (quantity > 0),
    INDEX idx_order_id (order_id),
    INDEX idx_product_id (product_id)
);
```

---

## 8. Production Deployment

### 8.1 Zero-downtime Deployment

```yaml
# k8s/deployments/product-service.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-service
  namespace: production
  annotations:
    kubernetes.io/change-cause: "Deploy v2.1.3 - Add flash sale support"
spec:
  replicas: 20
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 5        # 25% surge
      maxUnavailable: 0  # Zero downtime
  selector:
    matchLabels:
      app: product-service
  template:
    metadata:
      labels:
        app: product-service
        version: v2.1.3
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: product-service
            topologyKey: kubernetes.io/hostname  # Spread across nodes
      containers:
      - name: product-service
        image: 123456789.dkr.ecr.ap-southeast-1.amazonaws.com/product-service:v2.1.3
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "2000m"
            memory: "2Gi"
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 3
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        env:
        - name: DB_HOST
          valueFrom:
            secretKeyRef:
              name: product-service-secrets
              key: db-host
        - name: REDIS_URL
          valueFrom:
            secretKeyRef:
              name: product-service-secrets
              key: redis-url
```

---

## สรุป

เราได้ออกแบบ E-Commerce Platform ระดับ Production ที่รองรับตลาดไทย โดยมีประเด็นสำคัญ:

1. **Service Decomposition** - แบ่งตาม Domain: User, Product, Order, Payment, Inventory, Search, Notification
2. **Technology Choices** - Go สำหรับ High-performance services, Python สำหรับ ML/Search, Java สำหรับ Payment
3. **Flash Sale Handling** - Redis + Lua Script สำหรับ Atomic stock management
4. **Scaling Strategy** - HPA + KEDA สำหรับ Kubernetes auto-scaling
5. **Cost Management** - ~1M THB/month สำหรับ 5M DAU พร้อม optimization strategies

> สิ่งสำคัญที่สุดคือ Start simple แล้วค่อย Scale ตาม Business growth

---

*ถัดไป: Part 93 - Case Study: Payment System*

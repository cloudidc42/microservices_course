# Part 99: Final Project — World-Class Microservices System

## บทนำ

ถึงเวลาของ Final Project แล้ว! เราจะสร้าง E-Commerce System แบบ World-Class ตั้งแต่ต้น โดยรวมทุกสิ่งที่เรียนรู้ตลอด 99 Parts เข้าด้วยกัน ตั้งแต่ Architecture Design, Service Implementation, Observability, Security, Deployment ไปจนถึง Production Checklist

---

## 1. Project Overview: ShopThai Platform

### 1.1 Vision

```
ShopThai คือ Thai-first E-Commerce Platform ที่:
- รองรับ Thai Payment (PromptPay, TrueMoney, Bank Transfer)
- Thai Language UI + SEO
- รองรับ 10M DAU, 1M orders/day
- 99.99% Availability
- < 100ms API Latency (p99)
- PCI-DSS Compliant
- PDPA Compliant (Personal Data Protection Act Thailand)
```

### 1.2 Final Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ShopThai World-Class Architecture                     │
│                                                                           │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                     Client Layer                                  │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────┐ │   │
│  │  │  Web (Next.js│  │ iOS App      │  │    Android App         │ │   │
│  │  │  + SSR/ISR)  │  │(Swift/React  │  │    (React Native)      │ │   │
│  │  └──────┬───────┘  │  Native)     │  └──────────┬─────────────┘ │   │
│  │         └──────────┴──────────────┘             │               │   │
│  └─────────────────────────────────────────────────┼───────────────┘   │
│                                                     │                   │
│  ┌──────────────────────────────────────────────────▼───────────────┐  │
│  │                    Edge / CDN Layer                               │  │
│  │  CloudFront (Bangkok Edge + Singapore PoP)                       │  │
│  │  - Static Assets, Images, Videos                                 │  │
│  │  - SSL Termination                                               │  │
│  │  - DDoS Protection (AWS Shield)                                  │  │
│  │  - WAF (OWASP Rules)                                             │  │
│  └─────────────────────────────┬─────────────────────────────────────┘  │
│                                │                                         │
│  ┌─────────────────────────────▼─────────────────────────────────────┐  │
│  │                   API Gateway Layer                                │  │
│  │  Kong + Kubernetes Ingress                                        │  │
│  │  - JWT Auth (Auth Service)                                        │  │
│  │  - Rate Limiting (Redis)                                          │  │
│  │  - Request/Response Transform                                     │  │
│  │  - API Versioning (v1, v2)                                        │  │
│  └─────────────────────────────┬─────────────────────────────────────┘  │
│                                │                                         │
│  ┌─────────────────────────────▼─────────────────────────────────────┐  │
│  │              Service Mesh (Istio Ambient)                          │  │
│  │              mTLS, Observability, Traffic Management               │  │
│  └─────────────────────────────┬─────────────────────────────────────┘  │
│                                │                                         │
│  ┌──────────────────────────────┴─────────────────────────────────────┐ │
│  │                     Microservices Layer                             │ │
│  │                                                                     │ │
│  │  ┌─────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐ │ │
│  │  │User Service │  │Product Svc │  │ Order Svc  │  │Payment Svc │ │ │
│  │  │(Go)         │  │(Go)        │  │(Go)        │  │(Go/Java)   │ │ │
│  │  └─────────────┘  └────────────┘  └────────────┘  └────────────┘ │ │
│  │  ┌─────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐ │ │
│  │  │Inventory Svc│  │Search Svc  │  │Notif. Svc  │  │Analytics   │ │ │
│  │  │(Go)         │  │(Python)    │  │(Go)        │  │(Python)    │ │ │
│  │  └─────────────┘  └────────────┘  └────────────┘  └────────────┘ │ │
│  └─────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                        Data Layer                                 │  │
│  │  PostgreSQL(RDS) | Redis(ElastiCache) | Elasticsearch | Kafka    │  │
│  │  S3 | BigQuery | TimescaleDB(metrics)                            │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                  Observability Stack                              │  │
│  │  Prometheus + Grafana | Jaeger | Loki | PagerDuty                │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Repository Structure

```
shopthai/
├── services/
│   ├── user-service/
│   │   ├── cmd/main.go
│   │   ├── internal/
│   │   │   ├── domain/
│   │   │   ├── handler/
│   │   │   ├── repository/
│   │   │   └── service/
│   │   ├── Dockerfile
│   │   └── go.mod
│   ├── product-service/
│   ├── order-service/
│   ├── payment-service/
│   ├── inventory-service/
│   ├── search-service/
│   ├── notification-service/
│   └── analytics-service/
├── shared/
│   ├── proto/              # gRPC protobuf definitions
│   ├── events/             # Kafka event schemas
│   ├── telemetry/          # OpenTelemetry setup
│   ├── middleware/         # Common HTTP middleware
│   └── errors/             # Common error types
├── infrastructure/
│   ├── k8s/               # Kubernetes manifests
│   │   ├── base/          # Kustomize base
│   │   ├── overlays/
│   │   │   ├── dev/
│   │   │   ├── staging/
│   │   │   └── production/
│   │   └── services/
│   ├── terraform/         # Infrastructure as Code
│   │   ├── modules/
│   │   │   ├── eks/
│   │   │   ├── rds/
│   │   │   ├── elasticache/
│   │   │   └── kafka/
│   │   ├── environments/
│   │   │   ├── dev/
│   │   │   ├── staging/
│   │   │   └── production/
│   │   └── main.tf
│   ├── helm/              # Helm charts
│   └── argocd/           # ArgoCD applications
├── .github/
│   └── workflows/
│       ├── ci.yml
│       ├── cd-staging.yml
│       └── cd-production.yml
├── docs/
│   ├── adr/               # Architecture Decision Records
│   ├── api/               # API documentation
│   └── runbooks/          # Operational runbooks
├── Tiltfile               # Local dev
├── docker-compose.yml     # Local dev alternative
└── Makefile
```

---

## 3. Step-by-Step Guide

### Step 1: Project Setup

```bash
# 1. Clone/Create Repository
git clone https://github.com/yourorg/shopthai

# 2. Setup Local Environment
make setup-local

# 3. Start Dependencies
docker-compose up -d postgres redis kafka elasticsearch

# 4. Run Migrations
make migrate-all

# 5. Start Services with Tilt
tilt up

# Makefile
.PHONY: setup-local migrate-all test lint build deploy

setup-local:
	@echo "Setting up local development environment..."
	brew install kubectl helm argocd-cli temporal-cli dapr
	kubectl apply -f k8s/local/
	
migrate-all:
	@for svc in user-service product-service order-service payment-service; do \
		echo "Migrating $$svc..."; \
		go run ./services/$$svc/cmd/migrate/...; \
	done

test:
	go test ./... -v -race -coverprofile=coverage.out
	go tool cover -html=coverage.out

lint:
	golangci-lint run ./...

build:
	@for svc in $(SERVICES); do \
		docker build -t registry.io/shopthai/$$svc:$(VERSION) ./services/$$svc; \
	done

deploy-staging:
	argocd app sync shopthai-staging
	
deploy-production:
	./scripts/production-deploy.sh $(VERSION)
```

### Step 2: User Service Implementation

```go
// services/user-service/cmd/main.go
package main

import (
    "context"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"
    
    "github.com/shopthai/user-service/internal/handler"
    "github.com/shopthai/user-service/internal/repository"
    "github.com/shopthai/user-service/internal/service"
    "github.com/shopthai/shared/telemetry"
    "github.com/go-chi/chi/v5"
    "github.com/go-chi/chi/v5/middleware"
    "go.uber.org/zap"
)

func main() {
    // Initialize logger
    logger, _ := zap.NewProduction()
    defer logger.Sync()
    
    ctx := context.Background()
    
    // Initialize tracing
    cleanup, err := telemetry.InitTracing(ctx, "user-service", "1.0.0")
    if err != nil {
        logger.Fatal("Failed to init tracing", zap.Error(err))
    }
    defer cleanup()
    
    // Initialize metrics
    telemetry.InitMetrics("user-service")
    
    // Database
    db, err := repository.NewPostgresDB(os.Getenv("DATABASE_URL"))
    if err != nil {
        logger.Fatal("Failed to connect to database", zap.Error(err))
    }
    
    // Redis
    cache, err := repository.NewRedisClient(os.Getenv("REDIS_URL"))
    if err != nil {
        logger.Fatal("Failed to connect to Redis", zap.Error(err))
    }
    
    // Initialize layers
    userRepo := repository.NewUserRepository(db)
    userService := service.NewUserService(userRepo, cache, logger)
    userHandler := handler.NewUserHandler(userService, logger)
    
    // Router
    r := chi.NewRouter()
    
    // Global middleware
    r.Use(middleware.RequestID)
    r.Use(middleware.RealIP)
    r.Use(middleware.Logger)
    r.Use(middleware.Recoverer)
    r.Use(middleware.Timeout(30 * time.Second))
    r.Use(telemetry.TraceMiddleware)
    r.Use(telemetry.MetricsMiddleware)
    
    // Health checks
    r.Get("/health/live", func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
        w.Write([]byte("ok"))
    })
    r.Get("/health/ready", userHandler.HealthReady)
    
    // API Routes
    r.Route("/api/v1", func(r chi.Router) {
        r.Route("/users", func(r chi.Router) {
            r.Post("/register", userHandler.Register)
            r.Post("/login", userHandler.Login)
            r.Post("/login/line", userHandler.LoginWithLINE)
            r.Post("/refresh", userHandler.RefreshToken)
            
            r.Group(func(r chi.Router) {
                r.Use(authMiddleware)
                r.Get("/me", userHandler.GetMe)
                r.Put("/me", userHandler.UpdateProfile)
                r.Get("/me/addresses", userHandler.GetAddresses)
                r.Post("/me/addresses", userHandler.AddAddress)
                r.Delete("/me/addresses/{id}", userHandler.DeleteAddress)
            })
        })
    })
    
    // Start server
    server := &http.Server{
        Addr:         ":" + getEnv("PORT", "8001"),
        Handler:      r,
        ReadTimeout:  15 * time.Second,
        WriteTimeout: 15 * time.Second,
        IdleTimeout:  60 * time.Second,
    }
    
    // Graceful shutdown
    go func() {
        logger.Info("Starting user-service", zap.String("addr", server.Addr))
        if err := server.ListenAndServe(); err != http.ErrServerClosed {
            logger.Fatal("Server failed", zap.Error(err))
        }
    }()
    
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit
    
    logger.Info("Shutting down gracefully...")
    
    shutdownCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    
    if err := server.Shutdown(shutdownCtx); err != nil {
        logger.Fatal("Server forced shutdown", zap.Error(err))
    }
    
    logger.Info("Server stopped")
}
```

### Step 3: CI/CD Pipeline

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

on:
  push:
    branches: [main, develop, 'feature/**']
  pull_request:
    branches: [main, develop]

env:
  GO_VERSION: "1.21"
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    name: Test
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: test
          POSTGRES_DB: shopthai_test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      
      redis:
        image: redis:7-alpine
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
    
    steps:
    - uses: actions/checkout@v4
    
    - uses: actions/setup-go@v4
      with:
        go-version: ${{ env.GO_VERSION }}
        cache: true
    
    - name: Run unit tests
      run: |
        go test ./... -race -short -coverprofile=coverage.out
        go tool cover -func coverage.out
    
    - name: Run integration tests
      env:
        DATABASE_URL: postgres://postgres:test@localhost/shopthai_test
        REDIS_URL: redis://localhost:6379
      run: go test ./... -tags=integration -race
    
    - name: Upload coverage
      uses: codecov/codecov-action@v3
      with:
        files: ./coverage.out
    
    - name: SonarQube Analysis
      uses: SonarSource/sonarcloud-github-action@master
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
  
  lint:
    name: Lint
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: golangci/golangci-lint-action@v3
      with:
        version: v1.54
    
    - name: Check for security issues
      uses: securego/gosec@master
      with:
        args: ./...
  
  build:
    name: Build & Push
    needs: [test, lint]
    runs-on: ubuntu-latest
    if: github.event_name == 'push'
    
    strategy:
      matrix:
        service: [user-service, product-service, order-service, payment-service, 
                  inventory-service, search-service, notification-service]
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3
    
    - name: Log in to registry
      uses: docker/login-action@v3
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    
    - name: Build and push
      uses: docker/build-push-action@v5
      with:
        context: services/${{ matrix.service }}
        push: true
        tags: |
          ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/${{ matrix.service }}:${{ github.sha }}
          ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/${{ matrix.service }}:latest
        cache-from: type=gha
        cache-to: type=gha,mode=max
        build-args: |
          BUILD_DATE=${{ github.event.head_commit.timestamp }}
          GIT_SHA=${{ github.sha }}
    
    - name: Run container security scan
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/${{ matrix.service }}:${{ github.sha }}
        exit-code: '1'
        severity: 'CRITICAL,HIGH'

  deploy-staging:
    name: Deploy to Staging
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/develop'
    environment: staging
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Update image tags
      run: |
        cd infrastructure/k8s/overlays/staging
        kustomize edit set image \
          user-service=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/user-service:${{ github.sha }} \
          order-service=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/order-service:${{ github.sha }}
        git config user.email "ci@shopthai.com"
        git config user.name "CI Bot"
        git add .
        git commit -m "ci: deploy ${{ github.sha }} to staging"
        git push
    
    - name: Sync ArgoCD
      run: |
        argocd app sync shopthai-staging \
          --server argocd.internal.shopthai.com \
          --auth-token ${{ secrets.ARGOCD_TOKEN }} \
          --timeout 300
    
    - name: Run smoke tests
      run: |
        sleep 30  # Wait for pods to be ready
        ./scripts/smoke-tests.sh https://staging.shopthai.com
```

### Step 4: Kubernetes Production Config

```yaml
# infrastructure/k8s/overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: production

resources:
- ../../base
- hpa/
- pdb/
- network-policies/

images:
- name: user-service
  newTag: "v1.5.2"
- name: order-service
  newTag: "v2.1.0"
- name: payment-service
  newTag: "v1.3.1"

patches:
# Production replicas
- patch: |-
    - op: replace
      path: /spec/replicas
      value: 10
  target:
    kind: Deployment
    name: user-service
- patch: |-
    - op: replace
      path: /spec/replicas
      value: 20
  target:
    kind: Deployment
    name: order-service

# Resource limits for production
- patch: |-
    - op: replace
      path: /spec/template/spec/containers/0/resources
      value:
        requests:
          cpu: "500m"
          memory: "512Mi"
        limits:
          cpu: "2000m"
          memory: "2Gi"
  target:
    kind: Deployment
    name: order-service

---
# PodDisruptionBudget สำหรับ Zero-downtime deployments
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: order-service-pdb
  namespace: production
spec:
  maxUnavailable: 1
  selector:
    matchLabels:
      app: order-service
```

---

## 4. Full Order Flow Implementation

```go
// services/order-service/internal/service/order_service.go
package service

import (
    "context"
    "fmt"
    "time"
    
    "github.com/google/uuid"
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/attribute"
    "go.uber.org/zap"
)

type OrderService struct {
    orderRepo    OrderRepository
    productClient ProductServiceClient
    paymentClient PaymentServiceClient
    inventoryClient InventoryServiceClient
    notifClient  NotificationServiceClient
    publisher    EventPublisher
    logger       *zap.Logger
}

func (s *OrderService) CreateOrder(ctx context.Context, req CreateOrderRequest) (*Order, error) {
    tracer := otel.Tracer("order-service")
    ctx, span := tracer.Start(ctx, "CreateOrder")
    defer span.End()
    
    span.SetAttributes(
        attribute.String("customer.id", req.CustomerID),
        attribute.Int("items.count", len(req.Items)),
    )
    
    // Validate products and get current prices
    ctx, validateSpan := tracer.Start(ctx, "ValidateProducts")
    var orderItems []OrderItem
    var subtotal float64
    
    for _, item := range req.Items {
        product, err := s.productClient.GetProduct(ctx, item.ProductID)
        if err != nil {
            validateSpan.End()
            return nil, fmt.Errorf("product %s not found: %w", item.ProductID, err)
        }
        
        if product.Stock < item.Quantity {
            validateSpan.End()
            return nil, fmt.Errorf("insufficient stock for product %s", item.ProductID)
        }
        
        orderItems = append(orderItems, OrderItem{
            ProductID:   item.ProductID,
            ProductName: product.Name,
            VariantID:   item.VariantID,
            Quantity:    item.Quantity,
            UnitPrice:   product.Price,
            Total:       product.Price * float64(item.Quantity),
        })
        subtotal += product.Price * float64(item.Quantity)
    }
    validateSpan.End()
    
    // Apply promotions
    discount := s.calculateDiscount(ctx, req.PromoCode, subtotal, req.CustomerID)
    
    // Calculate shipping
    shippingFee := s.calculateShipping(ctx, req.ShippingAddress)
    
    totalAmount := subtotal + shippingFee - discount
    
    // Create order record
    order := &Order{
        ID:           fmt.Sprintf("ST-%s-%s", time.Now().Format("20060102"), uuid.New().String()[:8]),
        CustomerID:   req.CustomerID,
        Items:        orderItems,
        Subtotal:     subtotal,
        ShippingFee:  shippingFee,
        Discount:     discount,
        Total:        totalAmount,
        Status:       OrderStatusPendingPayment,
        ShippingAddr: req.ShippingAddress,
        Notes:        req.Notes,
        CreatedAt:    time.Now(),
        ExpiresAt:    time.Now().Add(15 * time.Minute), // Payment window
    }
    
    // Persist
    if err := s.orderRepo.Create(ctx, order); err != nil {
        return nil, fmt.Errorf("create order: %w", err)
    }
    
    // Reserve inventory (soft lock)
    ctx, invSpan := tracer.Start(ctx, "ReserveInventory")
    if err := s.inventoryClient.Reserve(ctx, ReserveRequest{
        OrderID: order.ID,
        Items:   req.Items,
    }); err != nil {
        invSpan.End()
        s.orderRepo.UpdateStatus(ctx, order.ID, OrderStatusCancelled)
        return nil, fmt.Errorf("inventory reservation failed: %w", err)
    }
    invSpan.End()
    
    // Initiate payment
    ctx, paySpan := tracer.Start(ctx, "InitiatePayment")
    paymentResult, err := s.paymentClient.Initiate(ctx, InitiatePaymentRequest{
        OrderID:       order.ID,
        CustomerID:    req.CustomerID,
        Amount:        totalAmount,
        Currency:      "THB",
        Method:        req.PaymentMethod,
        ExpiresAt:     order.ExpiresAt,
        CallbackURL:   fmt.Sprintf("https://api.shopthai.com/webhooks/payment/%s", order.ID),
        ReturnURL:     fmt.Sprintf("https://shopthai.com/orders/%s", order.ID),
    })
    if err != nil {
        paySpan.End()
        // Release inventory
        s.inventoryClient.Release(ctx, order.ID)
        s.orderRepo.UpdateStatus(ctx, order.ID, OrderStatusCancelled)
        return nil, fmt.Errorf("payment initiation failed: %w", err)
    }
    paySpan.End()
    
    // Update order with payment reference
    s.orderRepo.UpdatePaymentRef(ctx, order.ID, paymentResult.PaymentRef)
    
    // Publish domain event
    s.publisher.Publish(ctx, "order.created", OrderCreatedEvent{
        OrderID:    order.ID,
        CustomerID: req.CustomerID,
        Total:      totalAmount,
        Items:      orderItems,
        CreatedAt:  order.CreatedAt,
    })
    
    s.logger.Info("Order created",
        zap.String("order_id", order.ID),
        zap.String("customer_id", req.CustomerID),
        zap.Float64("total", totalAmount),
    )
    
    return order, nil
}
```

---

## 5. Observability Setup

```go
// shared/telemetry/setup.go
package telemetry

import (
    "context"
    "net/http"
    "time"
    
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
    "github.com/prometheus/client_golang/prometheus/promhttp"
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
    sdktrace "go.opentelemetry.io/otel/sdk/trace"
)

// Standard Business Metrics
var (
    OrdersTotal = promauto.NewCounterVec(prometheus.CounterOpts{
        Namespace: "shopthai",
        Name:      "orders_total",
        Help:      "Total number of orders",
    }, []string{"status", "payment_method"})
    
    OrderAmount = promauto.NewHistogramVec(prometheus.HistogramOpts{
        Namespace: "shopthai",
        Name:      "order_amount_baht",
        Help:      "Order amounts in Thai Baht",
        Buckets:   []float64{100, 500, 1000, 5000, 10000, 50000, 100000},
    }, []string{"category"})
    
    APILatency = promauto.NewHistogramVec(prometheus.HistogramOpts{
        Namespace: "shopthai",
        Name:      "api_request_duration_seconds",
        Help:      "API request latency",
        Buckets:   prometheus.DefBuckets,
    }, []string{"method", "path", "status"})
    
    ActiveUsers = promauto.NewGauge(prometheus.GaugeOpts{
        Namespace: "shopthai",
        Name:      "active_users",
        Help:      "Current active users",
    })
    
    CacheHitRatio = promauto.NewGaugeVec(prometheus.GaugeOpts{
        Namespace: "shopthai",
        Name:      "cache_hit_ratio",
        Help:      "Cache hit ratio by cache name",
    }, []string{"cache_name"})
)

// Middleware to record metrics for every HTTP request
func MetricsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        
        rw := newResponseWriter(w)
        next.ServeHTTP(rw, r)
        
        duration := time.Since(start).Seconds()
        
        APILatency.WithLabelValues(
            r.Method,
            sanitizePath(r.URL.Path),
            fmt.Sprintf("%d", rw.status),
        ).Observe(duration)
    })
}
```

```yaml
# monitoring/grafana-dashboards/shopthai-overview.json
# Key Dashboards:

Dashboard 1: Business Metrics
- Orders per minute (real-time)
- GMV per hour
- Conversion rate (view → cart → order → payment)
- Payment success rate
- Active users

Dashboard 2: Service Health
- Request rate per service
- Error rate per service
- Latency P50/P95/P99 per service
- Pod count per service
- Circuit breaker status

Dashboard 3: Infrastructure
- CPU/Memory per node
- Database connection pool utilization
- Redis memory usage + eviction rate
- Kafka consumer lag per topic

Dashboard 4: Cost (FinOps)
- Daily cloud spend
- Cost per service
- Cost per order
- Spot vs On-demand ratio
```

---

## 6. Production Deployment Checklist

```
PRE-DEPLOYMENT (48 hours before):
Security:
□ Security scan passed (Trivy, Snyk)
□ SAST analysis clean
□ Dependency updates applied
□ Secrets rotated (if needed)
□ PCI-DSS controls verified (for payment changes)
□ PDPA compliance checked (for user data changes)

Testing:
□ Unit tests: 100% passing, >80% coverage
□ Integration tests: 100% passing
□ Performance tests: No regression vs previous version
□ Load test: Passed at 150% expected traffic
□ Chaos test: Circuit breakers work correctly
□ E2E tests: Critical user journeys passing

Infrastructure:
□ Database migrations tested on staging (same data size)
□ Migration rollback script tested
□ Feature flags configured
□ Rollback plan documented
□ On-call briefed and available
□ Runbook updated

DEPLOYMENT:
Blue-Green Deployment Process:
□ Deploy to Green environment
□ Run smoke tests on Green
□ Shift 5% traffic → Green
□ Monitor for 15 minutes
□ Shift 50% traffic → Green  
□ Monitor for 30 minutes
□ Shift 100% traffic → Green
□ Keep Blue warm for 1 hour (rollback capability)
□ Decommission Blue

POST-DEPLOYMENT (First 24 hours):
□ Monitor error rates (target: < 0.1%)
□ Monitor latency (target: p99 < 100ms)
□ Monitor business metrics (orders, payments)
□ Check database query performance
□ Verify log aggregation working
□ Confirm alerts functioning
□ Update status page
□ Send deployment notification to stakeholders
```

---

## 7. Disaster Recovery

```
Recovery Objectives:
- RTO (Recovery Time Objective): 15 minutes
- RPO (Recovery Point Objective): 5 minutes

Disaster Scenarios:

1. Primary Region (Singapore) goes down:
   - Route53 Health Check → failover to backup region (Tokyo)
   - Read replica promoted to primary (< 2 minutes)
   - ETA to full recovery: 5-10 minutes

2. Database corruption:
   - Restore from automated backup (point-in-time)
   - RDS Automated backup: every 5 minutes
   - ETA: depends on how old the restore point

3. Kafka cluster failure:
   - MSK Multi-AZ: auto-failover
   - Consumer lag monitoring: alert if lag > 10K messages
   - ETA: 2-3 minutes

4. DDoS Attack:
   - AWS Shield Standard: auto-mitigation
   - WAF rules: rate limiting, IP blocking
   - Escalate to AWS Shield Advanced if needed
   - ETA: immediate (automated)

DR Testing Schedule:
□ Monthly: Failover test (off-peak hours)
□ Quarterly: Full DR drill (simulate region failure)
□ Annually: Full incident simulation with all teams
```

---

## สรุป

Final Project ShopThai Platform แสดงให้เห็นว่า World-Class Microservices ต้องประกอบด้วย:

1. **Clean Architecture** - Domain-driven, ง่ายต่อการ test และ maintain
2. **Robust CI/CD** - Automated testing, security scanning, zero-downtime deployment
3. **Comprehensive Observability** - Metrics, Tracing, Logging ตั้งแต่วันแรก
4. **Production-Ready Config** - HPA, PDB, Resource limits, Network policies
5. **Security First** - mTLS, RBAC, secrets management, PCI-DSS compliance
6. **Disaster Recovery** - ทดสอบ failover สม่ำเสมอ
7. **FinOps** - Cost tracking และ optimization
8. **Team Practices** - Runbooks, ADRs, On-call procedures

```
Final Words สำหรับ Engineers ที่จะสร้าง Production Systems:

"Perfect is the enemy of good"
- เริ่มต้น simple แล้วค่อย scale
- Automate everything possible  
- เตรียมพร้อมสำหรับ failure
- ใส่ใจ Developer Experience
- วัดผล ปรับปรุง วนซ้ำ

You're now ready to build world-class systems. 🚀
```

---

*ถัดไป: Part 100 - หลักสูตรสรุปและก้าวต่อไป*

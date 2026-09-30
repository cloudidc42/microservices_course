# Part 36: API Gateway Patterns - Advanced

## บทนำ

API Gateway ไม่ใช่แค่ reverse proxy ธรรมดา แต่เป็น intelligent layer ที่จัดการ cross-cutting concerns ทั้งหมด บทนี้จะลงลึกถึง advanced patterns เช่น rate limiting ด้วย Redis sliding window, request transformation, API composition, GraphQL gateway และการ integrate กับ OpenTelemetry

## เนื้อหาในบทนี้

- Rate Limiting per User/Tier ด้วย Redis Sliding Window
- Request/Response Transformation Middleware
- API Composition (Aggregate Multiple Service Calls)
- GraphQL to REST Gateway
- gRPC-Gateway Transcoding
- Request Caching at Gateway Level
- API Analytics และ Tracing ด้วย OpenTelemetry
- Kong/Nginx-based Gateway Configuration

---

## 1. Rate Limiting ด้วย Redis Sliding Window

### หลักการ Sliding Window Algorithm

Sliding Window ดีกว่า Fixed Window เพราะป้องกัน burst request ที่ขอบ window ได้

```
Fixed Window (มีปัญหา):
[0s---10s] [10s---20s]
  5 req       5 req    = limit 5 req/10s
แต่ถ้า req 5 ตัวมาที่ 9.9s และอีก 5 ตัวมาที่ 10.1s = 10 req ใน 200ms!

Sliding Window (ดีกว่า):
ทุก request ดูย้อนหลัง 10 วินาที = ไม่มีปัญหา burst
```

### Redis Sliding Window Implementation

```go
// ratelimiter/sliding_window.go
package ratelimiter

import (
    "context"
    "fmt"
    "time"

    "github.com/redis/go-redis/v9"
)

type SlidingWindowLimiter struct {
    client     *redis.Client
    windowSize time.Duration
    maxTokens  int
    keyPrefix  string
}

func NewSlidingWindowLimiter(
    client *redis.Client,
    windowSize time.Duration,
    maxTokens int,
) *SlidingWindowLimiter {
    return &SlidingWindowLimiter{
        client:     client,
        windowSize: windowSize,
        maxTokens:  maxTokens,
        keyPrefix:  "ratelimit:sw",
    }
}

// Lua script เพื่อให้ atomic operation
var slidingWindowScript = redis.NewScript(`
    local key = KEYS[1]
    local now = tonumber(ARGV[1])
    local window = tonumber(ARGV[2])
    local max_tokens = tonumber(ARGV[3])
    local request_id = ARGV[4]
    
    -- ลบ entries ที่หมด window แล้ว
    redis.call('ZREMRANGEBYSCORE', key, '-inf', now - window)
    
    -- นับ requests ใน window ปัจจุบัน
    local count = redis.call('ZCARD', key)
    
    if count < max_tokens then
        -- เพิ่ม request ใหม่
        redis.call('ZADD', key, now, request_id)
        -- Set TTL เพื่อ auto cleanup
        redis.call('PEXPIRE', key, window)
        return {1, max_tokens - count - 1}  -- allowed, remaining
    else
        -- คำนวณเวลาที่ต้อง wait
        local oldest = redis.call('ZRANGE', key, 0, 0, 'WITHSCORES')
        local retry_after = 0
        if oldest[2] then
            retry_after = window - (now - tonumber(oldest[2]))
        end
        return {0, 0, retry_after}  -- denied, remaining, retry_after_ms
    end
`)

type RateLimitResult struct {
    Allowed    bool
    Remaining  int
    RetryAfter time.Duration
}

func (r *SlidingWindowLimiter) Allow(ctx context.Context, key string) (*RateLimitResult, error) {
    redisKey := fmt.Sprintf("%s:%s", r.keyPrefix, key)
    now := time.Now().UnixMilli()
    windowMs := r.windowSize.Milliseconds()
    requestID := fmt.Sprintf("%d-%d", now, generateRequestID())

    result, err := slidingWindowScript.Run(
        ctx,
        r.client,
        []string{redisKey},
        now,
        windowMs,
        r.maxTokens,
        requestID,
    ).Int64Slice()

    if err != nil {
        // Fail open - ถ้า Redis ล่ม ให้ผ่านได้ (หรือ fail closed ขึ้นกับ policy)
        return &RateLimitResult{Allowed: true}, nil
    }

    if result[0] == 1 {
        return &RateLimitResult{
            Allowed:   true,
            Remaining: int(result[1]),
        }, nil
    }

    return &RateLimitResult{
        Allowed:    false,
        Remaining:  0,
        RetryAfter: time.Duration(result[2]) * time.Millisecond,
    }, nil
}
```

### Rate Limiting Middleware ด้วย Tier System

```go
// middleware/rate_limit_middleware.go
package middleware

import (
    "net/http"
    "strconv"
    "time"
)

type UserTier string

const (
    TierFree       UserTier = "free"
    TierBasic      UserTier = "basic"
    TierProfessional UserTier = "professional"
    TierEnterprise UserTier = "enterprise"
)

type TierConfig struct {
    RequestsPerMinute int
    RequestsPerHour   int
    RequestsPerDay    int
    BurstSize         int
}

var TierConfigs = map[UserTier]TierConfig{
    TierFree:         {RequestsPerMinute: 10, RequestsPerHour: 100, RequestsPerDay: 1000, BurstSize: 20},
    TierBasic:        {RequestsPerMinute: 60, RequestsPerHour: 1000, RequestsPerDay: 10000, BurstSize: 100},
    TierProfessional: {RequestsPerMinute: 300, RequestsPerHour: 5000, RequestsPerDay: 50000, BurstSize: 500},
    TierEnterprise:   {RequestsPerMinute: 1000, RequestsPerHour: 20000, RequestsPerDay: 200000, BurstSize: 2000},
}

type RateLimitMiddleware struct {
    minuteLimiter *SlidingWindowLimiter
    hourLimiter   *SlidingWindowLimiter
    dayLimiter    *SlidingWindowLimiter
    userService   UserTierService
}

func (m *RateLimitMiddleware) Handler(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // ดึง user ID จาก JWT token
        userID := extractUserID(r)
        tier := m.userService.GetUserTier(userID)
        config := TierConfigs[tier]

        // Check rate limits ทุก level
        checks := []struct {
            limiter *SlidingWindowLimiter
            key     string
            limit   int
        }{
            {m.minuteLimiter, fmt.Sprintf("min:%s", userID), config.RequestsPerMinute},
            {m.hourLimiter, fmt.Sprintf("hr:%s", userID), config.RequestsPerHour},
            {m.dayLimiter, fmt.Sprintf("day:%s", userID), config.RequestsPerDay},
        }

        for _, check := range checks {
            result, err := check.limiter.Allow(r.Context(), check.key)
            if err != nil {
                // Log error แต่ allow pass (fail open)
                continue
            }

            if !result.Allowed {
                w.Header().Set("X-RateLimit-Limit", strconv.Itoa(check.limit))
                w.Header().Set("X-RateLimit-Remaining", "0")
                w.Header().Set("X-RateLimit-Reset", 
                    strconv.FormatInt(time.Now().Add(result.RetryAfter).Unix(), 10))
                w.Header().Set("Retry-After", 
                    strconv.FormatInt(int64(result.RetryAfter.Seconds()), 10))
                
                http.Error(w, `{"error":"rate_limit_exceeded","message":"Too Many Requests"}`, 
                    http.StatusTooManyRequests)
                return
            }

            // Set headers สำหรับ monitoring
            w.Header().Set("X-RateLimit-Remaining", strconv.Itoa(result.Remaining))
        }

        next.ServeHTTP(w, r)
    })
}
```

---

## 2. Request/Response Transformation Middleware

```go
// middleware/transform.go
package middleware

import (
    "bytes"
    "encoding/json"
    "io"
    "net/http"
)

// TransformationConfig กำหนด rules การ transform
type TransformationConfig struct {
    Request  RequestTransform
    Response ResponseTransform
}

type RequestTransform struct {
    // เพิ่ม/ลบ/แก้ไข headers
    AddHeaders    map[string]string
    RemoveHeaders []string
    
    // แก้ไข path
    PathRewrite map[string]string
    
    // เพิ่ม query params
    AddQueryParams map[string]string
    
    // Transform body
    BodyTransforms []BodyTransform
}

type BodyTransform struct {
    // JSONPath-like transformations
    Source      string
    Destination string
    Transform   func(interface{}) interface{}
}

type ResponseTransform struct {
    AddHeaders    map[string]string
    RemoveHeaders []string
    BodyTransforms []BodyTransform
    
    // Filter fields ที่ไม่ต้องการส่ง client
    RemoveFields []string
}

// ResponseWriter wrapper สำหรับ capture response
type captureResponseWriter struct {
    http.ResponseWriter
    body       bytes.Buffer
    statusCode int
}

func (c *captureResponseWriter) Write(b []byte) (int, error) {
    return c.body.Write(b)
}

func (c *captureResponseWriter) WriteHeader(statusCode int) {
    c.statusCode = statusCode
}

// TransformationMiddleware
func NewTransformationMiddleware(config TransformationConfig) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            // Transform request
            r = transformRequest(r, config.Request)

            // Capture response
            crw := &captureResponseWriter{
                ResponseWriter: w,
                statusCode:     http.StatusOK,
            }

            next.ServeHTTP(crw, r)

            // Transform response
            transformedBody := transformResponse(crw.body.Bytes(), config.Response)

            // Apply response header transforms
            for k, v := range config.Response.AddHeaders {
                w.Header().Set(k, v)
            }
            for _, h := range config.Response.RemoveHeaders {
                w.Header().Del(h)
            }

            // Write transformed response
            w.Header().Set("Content-Length", strconv.Itoa(len(transformedBody)))
            w.WriteHeader(crw.statusCode)
            w.Write(transformedBody)
        })
    }
}

func transformRequest(r *http.Request, config RequestTransform) *http.Request {
    // Clone request
    newReq := r.Clone(r.Context())

    // Add headers
    for k, v := range config.AddHeaders {
        newReq.Header.Set(k, v)
    }

    // Remove headers
    for _, h := range config.RemoveHeaders {
        newReq.Header.Del(h)
    }

    // Add correlation ID ถ้ายังไม่มี
    if newReq.Header.Get("X-Correlation-ID") == "" {
        newReq.Header.Set("X-Correlation-ID", generateUUID())
    }

    // Transform body
    if len(config.BodyTransforms) > 0 && newReq.Body != nil {
        body, err := io.ReadAll(newReq.Body)
        if err == nil {
            transformed := applyBodyTransforms(body, config.BodyTransforms)
            newReq.Body = io.NopCloser(bytes.NewReader(transformed))
            newReq.ContentLength = int64(len(transformed))
        }
    }

    return newReq
}

func transformResponse(body []byte, config ResponseTransform) []byte {
    if len(config.RemoveFields) == 0 && len(config.BodyTransforms) == 0 {
        return body
    }

    var data map[string]interface{}
    if err := json.Unmarshal(body, &data); err != nil {
        return body  // ไม่ใช่ JSON, return as-is
    }

    // Remove sensitive fields
    for _, field := range config.RemoveFields {
        removeNestedField(data, field)
    }

    // Apply transforms
    data = applyMapTransforms(data, config.BodyTransforms)

    result, err := json.Marshal(data)
    if err != nil {
        return body
    }
    return result
}

// ตัวอย่าง: ลบ internal fields
func removeNestedField(data map[string]interface{}, path string) {
    parts := strings.SplitN(path, ".", 2)
    if len(parts) == 1 {
        delete(data, parts[0])
        return
    }

    if nested, ok := data[parts[0]].(map[string]interface{}); ok {
        removeNestedField(nested, parts[1])
    }
}
```

---

## 3. API Composition (Aggregate Multiple Service Calls)

API Composition คือการ aggregate ข้อมูลจาก multiple services ใน single API call

```go
// composition/order_detail_composer.go
package composition

import (
    "context"
    "sync"
    "time"
)

// OrderDetailResponse คือ aggregated response
type OrderDetailResponse struct {
    Order    *OrderData    `json:"order"`
    User     *UserData     `json:"user"`
    Products []ProductData `json:"products"`
    Shipping *ShippingData `json:"shipping"`
    Payment  *PaymentData  `json:"payment"`
}

type OrderComposer struct {
    orderSvc    OrderService
    userSvc     UserService
    productSvc  ProductService
    shippingSvc ShippingService
    paymentSvc  PaymentService
    timeout     time.Duration
}

func (c *OrderComposer) GetOrderDetail(ctx context.Context, orderID string) (*OrderDetailResponse, error) {
    ctx, cancel := context.WithTimeout(ctx, c.timeout)
    defer cancel()

    // Step 1: Get order data (ต้องได้ก่อน เพราะ services อื่นต้องใช้ข้อมูลนี้)
    order, err := c.orderSvc.GetOrder(ctx, orderID)
    if err != nil {
        return nil, fmt.Errorf("failed to get order: %w", err)
    }

    // Step 2: Fetch ข้อมูลที่เหลือแบบ parallel
    var (
        wg          sync.WaitGroup
        mu          sync.Mutex
        errors      []error
        user        *UserData
        products    []ProductData
        shipping    *ShippingData
        payment     *PaymentData
    )

    // Fan-out: ยิง requests พร้อมกัน
    wg.Add(4)

    // Fetch user
    go func() {
        defer wg.Done()
        u, err := c.userSvc.GetUser(ctx, order.UserID)
        mu.Lock()
        defer mu.Unlock()
        if err != nil {
            errors = append(errors, fmt.Errorf("user: %w", err))
        } else {
            user = u
        }
    }()

    // Fetch products
    go func() {
        defer wg.Done()
        productIDs := extractProductIDs(order.Items)
        p, err := c.productSvc.GetProductsBatch(ctx, productIDs)
        mu.Lock()
        defer mu.Unlock()
        if err != nil {
            errors = append(errors, fmt.Errorf("products: %w", err))
        } else {
            products = p
        }
    }()

    // Fetch shipping
    go func() {
        defer wg.Done()
        s, err := c.shippingSvc.GetShippingStatus(ctx, order.ID)
        mu.Lock()
        defer mu.Unlock()
        if err != nil {
            // Shipping info is optional - don't fail entire request
            log.Warnf("Failed to get shipping for order %s: %v", order.ID, err)
        } else {
            shipping = s
        }
    }()

    // Fetch payment
    go func() {
        defer wg.Done()
        p, err := c.paymentSvc.GetPayment(ctx, order.PaymentID)
        mu.Lock()
        defer mu.Unlock()
        if err != nil {
            errors = append(errors, fmt.Errorf("payment: %w", err))
        } else {
            payment = p
        }
    }()

    wg.Wait()

    // Check for critical errors
    if len(errors) > 0 {
        return nil, fmt.Errorf("composition errors: %v", errors)
    }

    // Assemble response
    return &OrderDetailResponse{
        Order:    order,
        User:     user,
        Products: products,
        Shipping: shipping,
        Payment:  payment,
    }, nil
}
```

### Circuit Breaker ใน API Composition

```go
// composition/resilient_composer.go
package composition

import (
    "github.com/sony/gobreaker"
)

type ResilientComposer struct {
    composer    *OrderComposer
    breakers    map[string]*gobreaker.CircuitBreaker
}

func NewResilientComposer(composer *OrderComposer) *ResilientComposer {
    settings := gobreaker.Settings{
        Name:        "api-composition",
        MaxRequests: 3,
        Interval:    10 * time.Second,
        Timeout:     30 * time.Second,
        ReadyToTrip: func(counts gobreaker.Counts) bool {
            return counts.ConsecutiveFailures > 5
        },
    }

    return &ResilientComposer{
        composer: composer,
        breakers: map[string]*gobreaker.CircuitBreaker{
            "user":     gobreaker.NewCircuitBreaker(settings),
            "product":  gobreaker.NewCircuitBreaker(settings),
            "shipping": gobreaker.NewCircuitBreaker(settings),
            "payment":  gobreaker.NewCircuitBreaker(settings),
        },
    }
}

func (r *ResilientComposer) GetUserWithFallback(ctx context.Context, userID string) (*UserData, error) {
    result, err := r.breakers["user"].Execute(func() (interface{}, error) {
        return r.composer.userSvc.GetUser(ctx, userID)
    })

    if err != nil {
        // Return cached/default data เมื่อ circuit open
        return r.getCachedUser(userID), nil
    }

    return result.(*UserData), nil
}
```

---

## 4. GraphQL to REST Gateway

```javascript
// graphql-gateway/src/schema.js
const { ApolloServer, gql } = require('@apollo/server');
const { expressMiddleware } = require('@apollo/server/express4');

const typeDefs = gql`
    type Query {
        order(id: ID!): Order
        orders(userId: ID!, page: Int, pageSize: Int): OrderConnection
        user(id: ID!): User
        product(id: ID!): Product
    }

    type Order {
        id: ID!
        status: OrderStatus!
        totalAmount: Float!
        currency: String!
        user: User!
        items: [OrderItem!]!
        shipping: ShippingInfo
        createdAt: String!
    }

    type OrderItem {
        id: ID!
        product: Product!
        quantity: Int!
        price: Float!
    }

    type User {
        id: ID!
        email: String!
        firstName: String!
        lastName: String!
        orders(page: Int, pageSize: Int): OrderConnection
    }

    type Product {
        id: ID!
        name: String!
        price: Float!
        stock: Int!
        category: Category
    }

    type ShippingInfo {
        trackingNumber: String
        carrier: String
        status: ShippingStatus
        estimatedDelivery: String
    }

    type OrderConnection {
        nodes: [Order!]!
        totalCount: Int!
        pageInfo: PageInfo!
    }

    type PageInfo {
        hasNextPage: Boolean!
        hasPreviousPage: Boolean!
        totalPages: Int!
    }

    enum OrderStatus {
        PENDING
        CONFIRMED
        PROCESSING
        SHIPPED
        DELIVERED
        CANCELLED
    }

    enum ShippingStatus {
        NOT_SHIPPED
        IN_TRANSIT
        OUT_FOR_DELIVERY
        DELIVERED
    }
`;

// Resolvers พร้อม DataLoader เพื่อแก้ N+1 Problem
const resolvers = {
    Query: {
        order: async (_, { id }, { dataSources }) => {
            return dataSources.orderAPI.getOrder(id);
        },
        orders: async (_, { userId, page = 1, pageSize = 20 }, { dataSources }) => {
            return dataSources.orderAPI.getUserOrders(userId, page, pageSize);
        },
        user: async (_, { id }, { dataSources }) => {
            return dataSources.userAPI.getUser(id);
        },
    },

    Order: {
        // DataLoader แก้ N+1 Problem
        user: async (order, _, { loaders }) => {
            return loaders.userLoader.load(order.userId);
        },
        items: async (order, _, { dataSources }) => {
            return dataSources.orderAPI.getOrderItems(order.id);
        },
        shipping: async (order, _, { dataSources }) => {
            try {
                return await dataSources.shippingAPI.getShipping(order.id);
            } catch (e) {
                return null;  // Optional field
            }
        },
    },

    OrderItem: {
        product: async (item, _, { loaders }) => {
            return loaders.productLoader.load(item.productId);
        },
    },
};

// DataLoaders สำหรับ batching requests
const { DataLoader } = require('dataloader');

function createLoaders(dataSources) {
    return {
        userLoader: new DataLoader(
            async (userIds) => {
                // Batch: load users ทั้งหมดใน single request
                const users = await dataSources.userAPI.getUsersBatch(userIds);
                // Return ตาม order ของ userIds
                return userIds.map(id => users.find(u => u.id === id) || null);
            },
            { cache: true, maxBatchSize: 100 }
        ),
        productLoader: new DataLoader(
            async (productIds) => {
                const products = await dataSources.productAPI.getProductsBatch(productIds);
                return productIds.map(id => products.find(p => p.id === id) || null);
            },
            { cache: true, maxBatchSize: 100 }
        ),
    };
}

module.exports = { typeDefs, resolvers, createLoaders };
```

---

## 5. gRPC-Gateway Transcoding

```protobuf
// proto/order/v1/order.proto
syntax = "proto3";

package order.v1;

import "google/api/annotations.proto";
import "google/protobuf/timestamp.proto";
import "protoc-gen-openapiv2/options/annotations.proto";

option go_package = "github.com/example/order-service/gen/order/v1;orderv1";

service OrderService {
    rpc CreateOrder(CreateOrderRequest) returns (Order) {
        option (google.api.http) = {
            post: "/v1/orders"
            body: "*"
        };
    }

    rpc GetOrder(GetOrderRequest) returns (Order) {
        option (google.api.http) = {
            get: "/v1/orders/{order_id}"
        };
    }

    rpc ListOrders(ListOrdersRequest) returns (ListOrdersResponse) {
        option (google.api.http) = {
            get: "/v1/orders"
        };
    }

    rpc UpdateOrderStatus(UpdateOrderStatusRequest) returns (Order) {
        option (google.api.http) = {
            patch: "/v1/orders/{order_id}/status"
            body: "*"
        };
    }
}

message Order {
    string id = 1;
    string user_id = 2;
    OrderStatus status = 3;
    double total_amount = 4;
    string currency = 5;
    repeated OrderItem items = 6;
    google.protobuf.Timestamp created_at = 7;
    google.protobuf.Timestamp updated_at = 8;
}

enum OrderStatus {
    ORDER_STATUS_UNSPECIFIED = 0;
    ORDER_STATUS_PENDING = 1;
    ORDER_STATUS_CONFIRMED = 2;
    ORDER_STATUS_PROCESSING = 3;
    ORDER_STATUS_SHIPPED = 4;
    ORDER_STATUS_DELIVERED = 5;
    ORDER_STATUS_CANCELLED = 6;
}
```

```go
// gateway/main.go
package main

import (
    "context"
    "net"
    "net/http"

    "github.com/grpc-ecosystem/grpc-gateway/v2/runtime"
    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"

    orderv1 "github.com/example/order-service/gen/order/v1"
)

func main() {
    // Start gRPC server
    go startGRPCServer()

    // Start gRPC-Gateway (HTTP → gRPC transcoding)
    ctx := context.Background()
    mux := runtime.NewServeMux(
        runtime.WithForwardResponseOption(httpResponseModifier),
        runtime.WithErrorHandler(customErrorHandler),
        runtime.WithIncomingHeaderMatcher(customHeaderMatcher),
    )

    opts := []grpc.DialOption{
        grpc.WithTransportCredentials(insecure.NewCredentials()),
    }

    err := orderv1.RegisterOrderServiceHandlerFromEndpoint(
        ctx, mux,
        "localhost:50051",  // gRPC server address
        opts,
    )
    if err != nil {
        log.Fatal("Failed to register handler:", err)
    }

    // Wrap with middleware
    handler := corsMiddleware(
        authMiddleware(
            loggingMiddleware(mux),
        ),
    )

    log.Println("HTTP Gateway starting on :8080")
    log.Fatal(http.ListenAndServe(":8080", handler))
}

// ปรับแต่ง HTTP response
func httpResponseModifier(ctx context.Context, w http.ResponseWriter, resp proto.Message) error {
    w.Header().Set("X-Service-Version", "1.0")
    w.Header().Set("Cache-Control", "no-cache")
    return nil
}

// Custom error handler
func customErrorHandler(ctx context.Context, mux *runtime.ServeMux, 
    marshaler runtime.Marshaler, w http.ResponseWriter, r *http.Request, err error) {

    s, ok := status.FromError(err)
    if !ok {
        runtime.DefaultHTTPErrorHandler(ctx, mux, marshaler, w, r, err)
        return
    }

    httpStatus := runtime.HTTPStatusFromCode(s.Code())
    
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(httpStatus)
    
    json.NewEncoder(w).Encode(map[string]interface{}{
        "error": map[string]interface{}{
            "code":    s.Code().String(),
            "message": s.Message(),
            "details": s.Details(),
        },
    })
}
```

---

## 6. Request Caching at Gateway Level

```go
// cache/gateway_cache.go
package cache

import (
    "crypto/sha256"
    "encoding/hex"
    "encoding/json"
    "net/http"
    "time"

    "github.com/redis/go-redis/v9"
)

type CacheConfig struct {
    DefaultTTL    time.Duration
    MaxBodySize   int64
    CacheableRoutes map[string]RouteConfig
}

type RouteConfig struct {
    TTL        time.Duration
    VaryBy     []string    // Headers ที่ใช้ vary cache key
    Conditions []CacheCondition
}

type CacheCondition func(r *http.Request) bool

type GatewayCache struct {
    client *redis.Client
    config CacheConfig
}

// สร้าง cache key จาก request
func (c *GatewayCache) buildCacheKey(r *http.Request, varyBy []string) string {
    h := sha256.New()
    
    // Base key: method + path + query
    h.Write([]byte(r.Method))
    h.Write([]byte(r.URL.Path))
    h.Write([]byte(r.URL.RawQuery))
    
    // Vary by headers (e.g., Authorization for per-user cache)
    for _, header := range varyBy {
        h.Write([]byte(r.Header.Get(header)))
    }
    
    return "gw:cache:" + hex.EncodeToString(h.Sum(nil))
}

type CachedResponse struct {
    StatusCode int               `json:"status_code"`
    Headers    map[string]string `json:"headers"`
    Body       []byte            `json:"body"`
    CachedAt   time.Time         `json:"cached_at"`
}

func (c *GatewayCache) Middleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Cache เฉพาะ GET requests
        if r.Method != http.MethodGet {
            next.ServeHTTP(w, r)
            return
        }

        // ดึง route config
        routeConfig, ok := c.config.CacheableRoutes[r.URL.Path]
        if !ok {
            next.ServeHTTP(w, r)
            return
        }

        // Check conditions
        for _, cond := range routeConfig.Conditions {
            if !cond(r) {
                next.ServeHTTP(w, r)
                return
            }
        }

        cacheKey := c.buildCacheKey(r, routeConfig.VaryBy)

        // Try cache hit
        cached, err := c.client.Get(r.Context(), cacheKey).Bytes()
        if err == nil {
            var resp CachedResponse
            if json.Unmarshal(cached, &resp) == nil {
                // Cache hit!
                for k, v := range resp.Headers {
                    w.Header().Set(k, v)
                }
                w.Header().Set("X-Cache", "HIT")
                w.Header().Set("Age", strconv.FormatInt(
                    int64(time.Since(resp.CachedAt).Seconds()), 10))
                w.WriteHeader(resp.StatusCode)
                w.Write(resp.Body)
                return
            }
        }

        // Cache miss - capture response
        crw := &captureResponseWriter{ResponseWriter: w, statusCode: http.StatusOK}
        next.ServeHTTP(crw, r)

        // Store in cache ถ้า response successful
        if crw.statusCode == http.StatusOK && 
           int64(crw.body.Len()) <= c.config.MaxBodySize {
            
            resp := CachedResponse{
                StatusCode: crw.statusCode,
                Headers:    captureHeaders(crw.Header()),
                Body:       crw.body.Bytes(),
                CachedAt:   time.Now(),
            }

            if data, err := json.Marshal(resp); err == nil {
                ttl := routeConfig.TTL
                if ttl == 0 {
                    ttl = c.config.DefaultTTL
                }
                c.client.SetEx(r.Context(), cacheKey, data, ttl)
            }
        }

        w.Header().Set("X-Cache", "MISS")
        w.WriteHeader(crw.statusCode)
        w.Write(crw.body.Bytes())
    })
}

// Cache invalidation เมื่อมี write operations
func (c *GatewayCache) InvalidatePattern(ctx context.Context, pattern string) error {
    keys, err := c.client.Keys(ctx, "gw:cache:"+pattern+"*").Result()
    if err != nil {
        return err
    }

    if len(keys) > 0 {
        return c.client.Del(ctx, keys...).Err()
    }
    return nil
}
```

---

## 7. API Analytics และ Tracing ด้วย OpenTelemetry

```go
// telemetry/otel_setup.go
package telemetry

import (
    "context"
    "time"

    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
    "go.opentelemetry.io/otel/exporters/otlp/otlpmetric/otlpmetricgrpc"
    "go.opentelemetry.io/otel/sdk/metric"
    "go.opentelemetry.io/otel/sdk/resource"
    "go.opentelemetry.io/otel/sdk/trace"
    semconv "go.opentelemetry.io/otel/semconv/v1.21.0"
)

func InitTelemetry(ctx context.Context, serviceName, serviceVersion string) (func(), error) {
    res, err := resource.New(ctx,
        resource.WithAttributes(
            semconv.ServiceName(serviceName),
            semconv.ServiceVersion(serviceVersion),
        ),
    )
    if err != nil {
        return nil, err
    }

    // Trace exporter
    traceExporter, err := otlptracegrpc.New(ctx,
        otlptracegrpc.WithEndpoint("otel-collector:4317"),
        otlptracegrpc.WithInsecure(),
    )
    if err != nil {
        return nil, err
    }

    tracerProvider := trace.NewTracerProvider(
        trace.WithBatcher(traceExporter),
        trace.WithResource(res),
        trace.WithSampler(trace.ParentBased(
            trace.TraceIDRatioBased(0.1),  // Sample 10% ใน production
        )),
    )
    otel.SetTracerProvider(tracerProvider)

    // Metric exporter
    metricExporter, err := otlpmetricgrpc.New(ctx,
        otlpmetricgrpc.WithEndpoint("otel-collector:4317"),
        otlpmetricgrpc.WithInsecure(),
    )
    if err != nil {
        return nil, err
    }

    meterProvider := metric.NewMeterProvider(
        metric.WithReader(
            metric.NewPeriodicReader(metricExporter,
                metric.WithInterval(10*time.Second),
            ),
        ),
        metric.WithResource(res),
    )
    otel.SetMeterProvider(meterProvider)

    // Cleanup function
    return func() {
        ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
        defer cancel()
        tracerProvider.Shutdown(ctx)
        meterProvider.Shutdown(ctx)
    }, nil
}
```

```go
// telemetry/api_analytics.go
package telemetry

import (
    "net/http"
    "time"

    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/attribute"
    "go.opentelemetry.io/otel/metric"
)

type APIAnalytics struct {
    requestsTotal   metric.Int64Counter
    requestDuration metric.Float64Histogram
    activeRequests  metric.Int64UpDownCounter
    requestSize     metric.Int64Histogram
    responseSize    metric.Int64Histogram
}

func NewAPIAnalytics() (*APIAnalytics, error) {
    meter := otel.GetMeterProvider().Meter("api-gateway")

    requestsTotal, _ := meter.Int64Counter(
        "api_gateway.requests.total",
        metric.WithDescription("Total number of API requests"),
    )

    requestDuration, _ := meter.Float64Histogram(
        "api_gateway.request.duration",
        metric.WithDescription("API request duration in milliseconds"),
        metric.WithUnit("ms"),
    )

    activeRequests, _ := meter.Int64UpDownCounter(
        "api_gateway.requests.active",
        metric.WithDescription("Number of active requests"),
    )

    return &APIAnalytics{
        requestsTotal:   requestsTotal,
        requestDuration: requestDuration,
        activeRequests:  activeRequests,
    }, nil
}

func (a *APIAnalytics) Middleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        
        attrs := []attribute.KeyValue{
            attribute.String("http.method", r.Method),
            attribute.String("http.route", getRouteTemplate(r)),
            attribute.String("http.target", r.URL.Path),
        }

        a.activeRequests.Add(r.Context(), 1, metric.WithAttributes(attrs...))
        defer a.activeRequests.Add(r.Context(), -1, metric.WithAttributes(attrs...))

        crw := &statusRecorder{ResponseWriter: w, statusCode: http.StatusOK}
        next.ServeHTTP(crw, r)

        duration := float64(time.Since(start).Milliseconds())
        statusAttrs := append(attrs,
            attribute.Int("http.status_code", crw.statusCode),
            attribute.String("http.status_class", getStatusClass(crw.statusCode)),
        )

        a.requestsTotal.Add(r.Context(), 1, metric.WithAttributes(statusAttrs...))
        a.requestDuration.Record(r.Context(), duration, metric.WithAttributes(statusAttrs...))
    })
}
```

---

## 8. Kong Gateway Configuration

```yaml
# kong/kong.yml (deck declarative config)
_format_version: "3.0"
_transform: true

services:
  - name: user-service
    url: http://user-service:8080
    connect_timeout: 5000
    write_timeout: 10000
    read_timeout: 10000
    retries: 3
    routes:
      - name: user-routes
        paths:
          - /api/v1/users
        strip_path: false
        methods:
          - GET
          - POST
          - PUT
          - PATCH
          - DELETE

  - name: order-service
    url: http://order-service:8080
    routes:
      - name: order-routes
        paths:
          - /api/v1/orders

plugins:
  # Rate Limiting
  - name: rate-limiting-advanced
    service: user-service
    config:
      limit:
        - 100
        - 1000
        - 10000
      window_size:
        - 60     # per minute
        - 3600   # per hour
        - 86400  # per day
      identifier: consumer
      sync_rate: 0
      strategy: redis
      redis:
        host: redis
        port: 6379
        database: 0

  # Authentication
  - name: jwt
    config:
      key_claim_name: kid
      claims_to_verify:
        - exp
        - nbf
      maximum_expiration: 3600

  # Request Transformation
  - name: request-transformer
    config:
      add:
        headers:
          - "X-Gateway-Version:1.0"
          - "X-Request-Time:$(date)"
        querystring: []
      remove:
        headers:
          - Authorization  # ลบ Authorization header ก่อนส่งต่อ (ส่ง user info แทน)
      replace:
        headers: []

  # Response Transformation
  - name: response-transformer
    config:
      remove:
        headers:
          - X-Internal-Service-Id
          - X-Debug-Info
        json:
          - internal_id
          - debug_data

  # CORS
  - name: cors
    config:
      origins:
        - "https://app.example.com"
        - "https://admin.example.com"
      methods:
        - GET
        - POST
        - PUT
        - PATCH
        - DELETE
        - OPTIONS
      headers:
        - Content-Type
        - Authorization
        - X-Request-ID
      exposed_headers:
        - X-RateLimit-Remaining
        - X-RateLimit-Limit
      credentials: true
      max_age: 3600

  # OpenTelemetry
  - name: opentelemetry
    config:
      endpoint: http://otel-collector:4317
      resource_attributes:
        service.name: api-gateway
        service.version: "1.0"
      header_tags:
        - header: X-User-ID
          tag: user.id
        - header: X-Tenant-ID
          tag: tenant.id
      batch_span_count: 200
      batch_flush_delay: 3

consumers:
  - username: mobile-app
    custom_id: mobile-app-v1
    tags:
      - mobile
      - tier:professional

  - username: web-app
    custom_id: web-app-v1
    tags:
      - web
      - tier:enterprise
```

### Nginx API Gateway Configuration

```nginx
# nginx/nginx.conf
upstream user_service {
    least_conn;
    server user-service-1:8080 max_fails=3 fail_timeout=30s;
    server user-service-2:8080 max_fails=3 fail_timeout=30s;
    keepalive 32;
}

upstream order_service {
    least_conn;
    server order-service-1:8080 max_fails=3 fail_timeout=30s;
    server order-service-2:8080 max_fails=3 fail_timeout=30s;
    keepalive 32;
}

# Rate limiting zones
limit_req_zone $binary_remote_addr zone=api_rate_limit:10m rate=100r/m;
limit_req_zone $http_x_user_id     zone=user_rate_limit:10m rate=60r/m;
limit_req_zone $http_x_api_tier    zone=tier_rate_limit:10m rate=300r/m;

# Caching
proxy_cache_path /var/cache/nginx/api 
    levels=1:2 
    keys_zone=api_cache:10m 
    max_size=1g 
    inactive=60m 
    use_temp_path=off;

server {
    listen 80;
    server_name api.example.com;
    
    # Security headers
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "DENY" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # Request ID สำหรับ distributed tracing
    set $request_id $http_x_request_id;
    if ($request_id = "") {
        set $request_id $request_id;
    }

    location /api/v1/users {
        # Rate limiting
        limit_req zone=api_rate_limit burst=20 nodelay;
        limit_req zone=user_rate_limit burst=5 nodelay;
        
        # Cache GET requests
        proxy_cache api_cache;
        proxy_cache_valid 200 1m;
        proxy_cache_methods GET HEAD;
        proxy_cache_key "$scheme$request_method$host$request_uri$http_authorization";
        proxy_cache_lock on;
        proxy_cache_use_stale error timeout updating;
        
        add_header X-Cache-Status $upstream_cache_status;
        
        # Proxy settings
        proxy_pass http://user_service;
        proxy_set_header Host            $host;
        proxy_set_header X-Real-IP       $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Request-ID    $request_id;
        
        # Remove internal headers from response
        proxy_hide_header X-Internal-Server;
        proxy_hide_header X-Powered-By;
        
        # Timeouts
        proxy_connect_timeout 5s;
        proxy_send_timeout    10s;
        proxy_read_timeout    30s;
    }

    location /api/v1/orders {
        limit_req zone=api_rate_limit burst=20 nodelay;
        
        proxy_pass http://order_service;
        proxy_set_header Host         $host;
        proxy_set_header X-Real-IP    $remote_addr;
        proxy_set_header X-Request-ID $request_id;
        
        proxy_connect_timeout 5s;
        proxy_send_timeout    30s;
        proxy_read_timeout    60s;
    }

    # Health check endpoint (ไม่ rate limit)
    location /health {
        access_log off;
        return 200 '{"status":"ok"}';
        add_header Content-Type application/json;
    }
}
```

---

## สรุป

Advanced API Gateway Patterns ที่ควรนำไปใช้:

1. **Sliding Window Rate Limiting** - ป้องกัน burst requests ได้ดีกว่า Fixed Window
2. **Tier-based Rate Limits** - แยก limits ตาม user tier (Free/Basic/Pro/Enterprise)
3. **API Composition** - Aggregate multiple service calls เพื่อลด round trips
4. **DataLoader Pattern** - แก้ N+1 problem ใน GraphQL gateway
5. **Response Caching** - Cache responses ที่ gateway เพื่อลด load ของ services
6. **OpenTelemetry** - Distributed tracing และ metrics ข้าม services

### Key Takeaways

- Rate limiting ควรมีหลาย levels (per-minute, per-hour, per-day)
- API Composition ต้องมี Circuit Breaker เพื่อ resilience
- Cache key ต้อง vary ตาม Authorization เพื่อความปลอดภัย
- Use CONCURRENTLY operations ใน fan-out patterns

---

## แบบฝึกหัด

1. **Rate Limiter**: Implement Token Bucket algorithm ด้วย Redis ที่รองรับ burst และ compare กับ Sliding Window ว่าแตกต่างกันอย่างไรใน edge cases

2. **API Composition**: สร้าง Product Catalog API ที่ aggregate ข้อมูลจาก product-service, inventory-service, และ pricing-service พร้อม Circuit Breaker และ fallback responses

3. **GraphQL Gateway**: Implement GraphQL subscription สำหรับ real-time order status updates โดย subscribe ไปยัง order-service ผ่าน WebSocket หรือ SSE

4. **Cache Invalidation**: ออกแบบ cache invalidation strategy สำหรับ API Gateway ที่ invalidate cache เมื่อ upstream service ส่ง webhook event มา

5. **Kong Plugin**: เขียน custom Kong plugin ด้วย Lua ที่เพิ่ม user's subscription tier information เข้าไปใน request headers โดยดึงจาก Redis cache

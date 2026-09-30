# Part 91: Microservices Interview Preparation

## บทนำ

การเตรียมตัวสัมภาษณ์งาน Software Engineer หรือ Backend Engineer ที่เน้น Microservices ต้องการความเข้าใจทั้งในเชิงทฤษฎีและปฏิบัติ บทนี้จะครอบคลุมคำถามที่พบบ่อย, การออกแบบระบบ, การเขียนโค้ด และกับดักที่ควรหลีกเลี่ยง

---

## 1. คำถามสัมภาษณ์ที่พบบ่อย (Common Interview Questions)

### 1.1 คำถามพื้นฐาน Microservices

**Q: Microservices คืออะไร และแตกต่างจาก Monolithic อย่างไร?**

**A (คำตอบที่ดี):**
```
Microservices คือรูปแบบสถาปัตยกรรมซอฟต์แวร์ที่แบ่งแอปพลิเคชันออกเป็น
Service ขนาดเล็กๆ ที่ทำงานได้อิสระ โดยแต่ละ Service:
- มีหน้าที่รับผิดชอบชัดเจน (Single Responsibility)
- สื่อสารผ่าน API (REST, gRPC, Message Queue)
- Deploy ได้อิสระ
- Scale ได้อิสระ
- ใช้ Database แยกกัน (Database per Service)

เปรียบเทียบ:
┌─────────────────────────────────────────────────────────┐
│           Monolithic vs Microservices                    │
├──────────────────┬──────────────────────────────────────┤
│    Monolithic     │         Microservices                │
├──────────────────┼──────────────────────────────────────┤
│ Deploy ทั้งก้อน   │ Deploy แต่ละ Service อิสระ            │
│ Scale ทั้งก้อน   │ Scale เฉพาะ Service ที่ต้องการ         │
│ ง่ายในช่วงแรก    │ ซับซ้อนกว่า แต่ scale ได้ดีกว่า        │
│ Single DB        │ Database per Service                 │
│ 1 Tech Stack     │ หลาย Tech Stack ได้                   │
└──────────────────┴──────────────────────────────────────┘
```

**Q: เมื่อไหร่ควรใช้ Microservices?**

**A:**
```
ควรใช้เมื่อ:
✅ ทีมใหญ่ (10+ คน) ที่ต้องทำงานพร้อมกัน
✅ ต้องการ Scale บางส่วนของระบบแยกกัน
✅ ระบบมีความซับซ้อนสูง (Complex Business Logic)
✅ ต้องการ High Availability
✅ แต่ละส่วนมี Traffic Pattern ต่างกัน

ไม่ควรใช้เมื่อ:
❌ ทีมเล็ก (< 5 คน)
❌ แอปพลิเคชันยังอยู่ในช่วงเริ่มต้น
❌ Business Logic ยังไม่ชัดเจน
❌ ไม่มี Infrastructure ที่พร้อม
```

### 1.2 คำถามเกี่ยวกับ Communication

**Q: REST vs gRPC vs Message Queue - เลือกใช้อะไรเมื่อไหร่?**

**A:**
```go
// REST - ใช้สำหรับ Public API, External Client
// เหมาะกับ: CRUD Operations, External-facing APIs
GET /api/v1/users/{id}
POST /api/v1/orders

// gRPC - ใช้สำหรับ Internal Service Communication
// เหมาะกับ: High Performance, Streaming, Strong Type
service OrderService {
  rpc GetOrder(GetOrderRequest) returns (Order);
  rpc StreamOrders(StreamRequest) returns (stream Order);
}

// Message Queue - ใช้สำหรับ Async, Event-driven
// เหมาะกับ: Decoupling, High Throughput, Fire-and-forget
// Kafka Topic: order.created -> email-service, inventory-service
```

```
เมื่อไหร่ใช้อะไร:
┌─────────────────┬────────────────────────────────────────┐
│ Protocol        │ Use Case                               │
├─────────────────┼────────────────────────────────────────┤
│ REST            │ Public API, CRUD, Client-facing        │
│ gRPC            │ Internal sync calls, Streaming, Perf   │
│ Kafka/RabbitMQ  │ Event sourcing, Async processing       │
│ WebSocket       │ Real-time, Chat, Live updates          │
│ GraphQL         │ Flexible queries, Mobile/Web clients   │
└─────────────────┴────────────────────────────────────────┘
```

**Q: อธิบาย Saga Pattern และบอกว่าใช้เมื่อไหร่?**

**A:**
```
Saga Pattern แก้ปัญหา Distributed Transactions โดยแบ่ง Transaction
ใหญ่เป็น Local Transactions เล็กๆ ที่มี Compensating Transactions

Choreography Saga (Event-driven):
┌────────────┐    order.created    ┌────────────────┐
│Order Service│ ─────────────────> │Payment Service │
└────────────┘                     └────────────────┘
                                          │ payment.completed
                                          ▼
                                   ┌────────────────┐
                                   │Inventory Service│
                                   └────────────────┘

Orchestration Saga (Centralized):
┌─────────────────────────────────────────┐
│           Saga Orchestrator              │
│  1. Create Order                         │
│  2. Reserve Payment → if fail: rollback  │
│  3. Update Inventory → if fail: rollback │
│  4. Send Confirmation                    │
└─────────────────────────────────────────┘
```

```go
// Orchestration Saga ตัวอย่าง
type OrderSaga struct {
    orderService     OrderService
    paymentService   PaymentService
    inventoryService InventoryService
}

func (s *OrderSaga) Execute(ctx context.Context, order Order) error {
    // Step 1: Create Order
    orderID, err := s.orderService.Create(ctx, order)
    if err != nil {
        return err
    }

    // Step 2: Reserve Payment
    paymentID, err := s.paymentService.Reserve(ctx, order.TotalAmount)
    if err != nil {
        // Compensate: Cancel Order
        s.orderService.Cancel(ctx, orderID)
        return err
    }

    // Step 3: Update Inventory
    if err := s.inventoryService.Reserve(ctx, order.Items); err != nil {
        // Compensate: Refund Payment + Cancel Order
        s.paymentService.Release(ctx, paymentID)
        s.orderService.Cancel(ctx, orderID)
        return err
    }

    return s.orderService.Confirm(ctx, orderID)
}
```

### 1.3 คำถามเกี่ยวกับ Data Management

**Q: Database per Service Pattern มีข้อดีข้อเสียอย่างไร?**

**A:**
```
ข้อดี:
✅ Service แต่ละตัว Scale Database ได้อิสระ
✅ เลือก Database Technology ที่เหมาะสมกับ Use Case ได้
✅ ไม่มี Single Point of Failure ด้าน Database
✅ ทีมทำงานอิสระ ไม่ต้อง Coordinate Schema Change

ข้อเสีย:
❌ ยากต่อการ Query ข้าม Service (Cross-service queries)
❌ ต้องจัดการ Data Consistency เอง
❌ เพิ่ม Complexity ด้าน Infrastructure
❌ Data Duplication อาจเกิดขึ้น

วิธีแก้ปัญหา Cross-service queries:
1. API Composition - Query หลาย Service แล้ว Join ที่ Application Layer
2. CQRS + Event Sourcing - สร้าง Read Model ที่รวม Data
3. GraphQL Federation - รวม Data จากหลาย Service
```

**Q: CQRS คืออะไร? ใช้เมื่อไหร่?**

**A:**
```
CQRS = Command Query Responsibility Segregation
แยก Write (Command) และ Read (Query) ออกจากกัน

Write Side (Command):
User ─── CreateOrder ──> OrderCommandHandler ──> Write DB (PostgreSQL)
                                                        │
                                                  Event Published
                                                        │
Read Side (Query):                                      ▼
User ─── GetOrders ───> OrderQueryHandler  <── Read DB (Elasticsearch)
                                              (Updated by Event Consumer)

ใช้เมื่อ:
- Read/Write มี Scaling Requirement ต่างกัน
- ต้องการ Multiple Read Models
- Complex Query Requirements
- Event Sourcing
```

### 1.4 คำถามเกี่ยวกับ Resilience

**Q: Circuit Breaker คืออะไร? Implement อย่างไร?**

**A:**
```go
// Circuit Breaker States:
// CLOSED (ปกติ) → OPEN (เกิดปัญหา) → HALF-OPEN (ทดสอบ) → CLOSED

type CircuitBreaker struct {
    state          State
    failureCount   int
    failureThreshold int
    timeout        time.Duration
    lastFailureTime time.Time
    mu             sync.RWMutex
}

func (cb *CircuitBreaker) Execute(fn func() error) error {
    cb.mu.Lock()
    defer cb.mu.Unlock()

    switch cb.state {
    case Open:
        if time.Since(cb.lastFailureTime) > cb.timeout {
            cb.state = HalfOpen
        } else {
            return ErrCircuitOpen
        }
    }

    err := fn()
    if err != nil {
        cb.failureCount++
        cb.lastFailureTime = time.Now()
        if cb.failureCount >= cb.failureThreshold {
            cb.state = Open
        }
        return err
    }

    // Success - Reset
    cb.failureCount = 0
    cb.state = Closed
    return nil
}

// ใช้กับ go-resilience หรือ Sony's Gobreaker
import "github.com/sony/gobreaker"

cb := gobreaker.NewCircuitBreaker(gobreaker.Settings{
    Name:        "payment-service",
    MaxRequests: 3,
    Interval:    60 * time.Second,
    Timeout:     30 * time.Second,
    ReadyToTrip: func(counts gobreaker.Counts) bool {
        return counts.ConsecutiveFailures > 5
    },
})

result, err := cb.Execute(func() (interface{}, error) {
    return paymentService.Charge(ctx, amount)
})
```

---

## 2. System Design Patterns สำหรับการสัมภาษณ์

### 2.1 Framework การออกแบบระบบ

```
STEP-BY-STEP System Design Framework:

1. Clarify Requirements (5 นาที)
   - Functional Requirements: ระบบต้องทำอะไร?
   - Non-functional: Scale, Availability, Latency
   - Out of Scope: อะไรที่ไม่ต้องทำ?

2. Estimate Scale (3 นาที)
   - DAU (Daily Active Users)
   - Read/Write Ratio
   - Storage Requirements
   - Bandwidth

3. High-Level Design (10 นาที)
   - Service Components
   - Communication Pattern
   - Data Flow

4. Detailed Design (15 นาที)
   - Database Schema
   - API Design
   - Core Algorithms

5. Identify Bottlenecks (5 นาที)
   - Single Points of Failure
   - Scaling Challenges
   - Solutions
```

### 2.2 Estimation Techniques

```
Back-of-envelope Calculations:

Memory:
- 1 Byte = 8 bits
- 1 KB = 1,000 Bytes
- 1 MB = 1,000 KB
- 1 GB = 1,000 MB
- 1 TB = 1,000 GB

Time:
- L1 Cache: 1 ns
- L2 Cache: 10 ns
- RAM: 100 ns
- Network (same DC): 500 μs
- Network (cross DC): 10 ms
- Disk Read: 1-10 ms

ตัวอย่าง: Design Twitter-like system
DAU = 100M users
Tweet per day per user = 2 tweets
Total tweets/day = 200M tweets
TPS = 200M / 86,400 ≈ 2,300 writes/sec
Read:Write = 100:1
Read TPS = 230,000 reads/sec
Storage per tweet = 280 chars = ~280 bytes
Daily storage = 200M × 280 bytes = 56GB/day
5-year storage = 56GB × 365 × 5 ≈ 100TB
```

### 2.3 Design Patterns ที่ต้องรู้

```
Patterns ที่สำคัญในการสัมภาษณ์:

1. API Gateway Pattern
   ┌──────────┐     ┌────────────────┐     ┌──────────────┐
   │  Client  │────>│  API Gateway   │────>│ User Service │
   └──────────┘     │                │     └──────────────┘
                    │ - Auth         │────>┌──────────────┐
                    │ - Rate Limit   │     │Order Service │
                    │ - Load Balance │     └──────────────┘
                    │ - SSL Term     │────>┌──────────────┐
                    └────────────────┘     │Pay Service   │
                                           └──────────────┘

2. Service Discovery Pattern
   ┌──────────────────────────────────────────────────────┐
   │                  Service Registry (Consul/Eureka)     │
   │                                                       │
   │  service-a: [192.168.1.1:8080, 192.168.1.2:8080]    │
   │  service-b: [192.168.1.3:9090]                       │
   └──────────────────────────────────────────────────────┘
        ▲                              ▲
   Register                       Discover
        │                              │
   ┌────────┐                    ┌────────┐
   │Service A│                   │Service B│
   └────────┘                    └────────┘

3. Sidecar Pattern (Istio Service Mesh)
   ┌──────────────────────────────────────┐
   │              Pod                      │
   │  ┌────────────┐  ┌────────────────┐  │
   │  │ App        │  │ Envoy Sidecar  │  │
   │  │ Container  │  │ - mTLS         │  │
   │  │            │  │ - Observability│  │
   │  └────────────┘  │ - Retry/CB     │  │
   │                  └────────────────┘  │
   └──────────────────────────────────────┘

4. Event Sourcing Pattern
   Commands ──> Events (Immutable Log) ──> Aggregate State
   
   CreateOrder ──> OrderCreated ──> Order{status: "created"}
   PayOrder    ──> OrderPaid    ──> Order{status: "paid"}
   ShipOrder   ──> OrderShipped ──> Order{status: "shipped"}
```

---

## 3. Coding Exercises

### 3.1 Rate Limiter Implementation

```go
// คำถาม: Implement Token Bucket Rate Limiter
package ratelimiter

import (
    "sync"
    "time"
)

type TokenBucket struct {
    capacity    int64
    tokens      int64
    refillRate  int64 // tokens per second
    lastRefill  time.Time
    mu          sync.Mutex
}

func NewTokenBucket(capacity, refillRate int64) *TokenBucket {
    return &TokenBucket{
        capacity:   capacity,
        tokens:     capacity,
        refillRate: refillRate,
        lastRefill: time.Now(),
    }
}

func (tb *TokenBucket) Allow() bool {
    tb.mu.Lock()
    defer tb.mu.Unlock()

    now := time.Now()
    elapsed := now.Sub(tb.lastRefill).Seconds()
    
    // Refill tokens
    tokensToAdd := int64(elapsed * float64(tb.refillRate))
    tb.tokens = min(tb.capacity, tb.tokens+tokensToAdd)
    tb.lastRefill = now

    if tb.tokens > 0 {
        tb.tokens--
        return true
    }
    return false
}

func min(a, b int64) int64 {
    if a < b {
        return a
    }
    return b
}

// Redis-based Distributed Rate Limiter
func (r *RedisRateLimiter) Allow(ctx context.Context, key string, limit int64) (bool, error) {
    pipe := r.client.Pipeline()
    
    now := time.Now().UnixMilli()
    windowStart := now - 1000 // 1 second window
    
    // Sliding Window Log
    pipe.ZRemRangeByScore(ctx, key, "0", strconv.FormatInt(windowStart, 10))
    pipe.ZAdd(ctx, key, &redis.Z{Score: float64(now), Member: now})
    pipe.ZCard(ctx, key)
    pipe.Expire(ctx, key, 2*time.Second)
    
    cmds, err := pipe.Exec(ctx)
    if err != nil {
        return false, err
    }
    
    count := cmds[2].(*redis.IntCmd).Val()
    return count <= limit, nil
}
```

### 3.2 Consistent Hashing

```go
// คำถาม: Implement Consistent Hashing สำหรับ Cache Distribution
package consistenthash

import (
    "crypto/sha256"
    "encoding/binary"
    "fmt"
    "sort"
    "sync"
)

type ConsistentHash struct {
    replicas int
    ring     map[uint32]string // hash → node
    nodes    []uint32          // sorted hashes
    mu       sync.RWMutex
}

func New(replicas int) *ConsistentHash {
    return &ConsistentHash{
        replicas: replicas,
        ring:     make(map[uint32]string),
    }
}

func (ch *ConsistentHash) hash(key string) uint32 {
    h := sha256.Sum256([]byte(key))
    return binary.BigEndian.Uint32(h[:4])
}

func (ch *ConsistentHash) AddNode(node string) {
    ch.mu.Lock()
    defer ch.mu.Unlock()
    
    for i := 0; i < ch.replicas; i++ {
        virtualNode := fmt.Sprintf("%s#%d", node, i)
        hash := ch.hash(virtualNode)
        ch.ring[hash] = node
        ch.nodes = append(ch.nodes, hash)
    }
    sort.Slice(ch.nodes, func(i, j int) bool {
        return ch.nodes[i] < ch.nodes[j]
    })
}

func (ch *ConsistentHash) GetNode(key string) string {
    ch.mu.RLock()
    defer ch.mu.RUnlock()
    
    if len(ch.nodes) == 0 {
        return ""
    }
    
    hash := ch.hash(key)
    
    // Binary search for the first node >= hash
    idx := sort.Search(len(ch.nodes), func(i int) bool {
        return ch.nodes[i] >= hash
    })
    
    // Wrap around to first node
    if idx == len(ch.nodes) {
        idx = 0
    }
    
    return ch.ring[ch.nodes[idx]]
}

// Usage
func main() {
    ch := New(3) // 3 virtual nodes per real node
    
    ch.AddNode("cache-01")
    ch.AddNode("cache-02")
    ch.AddNode("cache-03")
    
    keys := []string{"user:1", "user:2", "order:100", "product:50"}
    for _, key := range keys {
        node := ch.GetNode(key)
        fmt.Printf("Key %s -> Node %s\n", key, node)
    }
}
```

### 3.3 Distributed Lock

```go
// คำถาม: Implement Distributed Lock ด้วย Redis
package distlock

import (
    "context"
    "crypto/rand"
    "encoding/hex"
    "time"
    
    "github.com/redis/go-redis/v9"
)

type DistributedLock struct {
    client *redis.Client
    key    string
    value  string
    ttl    time.Duration
}

func NewLock(client *redis.Client, key string, ttl time.Duration) (*DistributedLock, error) {
    b := make([]byte, 16)
    if _, err := rand.Read(b); err != nil {
        return nil, err
    }
    
    return &DistributedLock{
        client: client,
        key:    "lock:" + key,
        value:  hex.EncodeToString(b),
        ttl:    ttl,
    }, nil
}

func (l *DistributedLock) Acquire(ctx context.Context) (bool, error) {
    return l.client.SetNX(ctx, l.key, l.value, l.ttl).Result()
}

// Lua script เพื่อ Atomic check-and-delete
var releaseScript = redis.NewScript(`
    if redis.call("GET", KEYS[1]) == ARGV[1] then
        return redis.call("DEL", KEYS[1])
    else
        return 0
    end
`)

func (l *DistributedLock) Release(ctx context.Context) error {
    return releaseScript.Run(ctx, l.client, []string{l.key}, l.value).Err()
}

// Usage pattern
func processOrder(ctx context.Context, orderID string) error {
    lock, _ := NewLock(redisClient, "order:"+orderID, 30*time.Second)
    
    acquired, err := lock.Acquire(ctx)
    if err != nil || !acquired {
        return ErrLockNotAcquired
    }
    defer lock.Release(ctx)
    
    // Process order safely
    return doProcess(ctx, orderID)
}
```

### 3.4 Message Queue Consumer

```go
// คำถาม: Implement Idempotent Message Consumer
package consumer

import (
    "context"
    "crypto/sha256"
    "encoding/hex"
    "encoding/json"
)

type IdempotentConsumer struct {
    processedKeys map[string]bool
    store         IdempotencyStore
    handler       MessageHandler
}

func (c *IdempotentConsumer) Process(ctx context.Context, msg Message) error {
    // สร้าง idempotency key จาก message content
    key := c.generateKey(msg)
    
    // ตรวจสอบว่าเคย process แล้วหรือยัง
    if processed, _ := c.store.IsProcessed(ctx, key); processed {
        return nil // Skip duplicate
    }
    
    // Process message
    if err := c.handler.Handle(ctx, msg); err != nil {
        return err
    }
    
    // Mark as processed
    return c.store.MarkProcessed(ctx, key, 24*time.Hour)
}

func (c *IdempotentConsumer) generateKey(msg Message) string {
    data, _ := json.Marshal(msg)
    hash := sha256.Sum256(data)
    return hex.EncodeToString(hash[:])
}
```

---

## 4. Design a System Like LINE/Lazada/Grab

### 4.1 Design LINE (Messaging App)

```
คำถาม: "Design a messaging system like LINE that supports 
100M users with real-time messaging"

STEP 1: Requirements Clarification
- Functional: 1-1 chat, Group chat, Media sharing, Push notification
- Non-functional: 99.99% availability, < 100ms latency, 100M DAU
- Scale: 50 messages/user/day = 5B messages/day = ~58,000 msg/sec

STEP 2: High-Level Design
┌─────────────────────────────────────────────────────────────────┐
│                         LINE Architecture                        │
│                                                                   │
│  ┌──────┐    ┌──────────┐    ┌─────────────────────────────┐   │
│  │Mobile│    │ API GW   │    │    Chat Service Cluster       │   │
│  │App   │───>│(Kong)    │───>│  ┌──────────────────────┐    │   │
│  └──────┘    └──────────┘    │  │  WebSocket Servers   │    │   │
│                               │  │  (Stateful)          │    │   │
│  ┌──────┐    ┌──────────┐    │  └──────────────────────┘    │   │
│  │Web   │───>│Load      │───>│                               │   │
│  │Client│    │Balancer  │    │  ┌──────────────────────┐    │   │
│  └──────┘    └──────────┘    │  │  Message Router      │    │   │
│                               │  └──────────────────────┘    │   │
│                               └─────────────────────────────┘   │
│                                          │                        │
│                          ┌───────────────┴───────────────┐       │
│                          │                               │       │
│                    ┌─────▼──────┐               ┌───────▼─────┐  │
│                    │  Kafka     │               │   Redis     │  │
│                    │  (Events)  │               │  (Presence) │  │
│                    └─────┬──────┘               └─────────────┘  │
│                          │                                        │
│               ┌──────────┴──────────┐                           │
│               │                     │                           │
│        ┌──────▼──────┐    ┌─────────▼────┐                     │
│        │  Message DB  │    │ Push Notif.  │                     │
│        │(Cassandra)   │    │ Service      │                     │
│        └─────────────┘    └──────────────┘                     │
└─────────────────────────────────────────────────────────────────┘

STEP 3: Key Design Decisions

1. WebSocket Connection Management:
   - User connects to WebSocket Server
   - Connection info stored in Redis (userID → serverID)
   - Router checks Redis to find correct server

2. Message Storage (Cassandra):
   CREATE TABLE messages (
     conversation_id UUID,
     message_id TIMEUUID,  -- TimeUUID for ordering
     sender_id UUID,
     content TEXT,
     created_at TIMESTAMP,
     PRIMARY KEY (conversation_id, message_id)
   ) WITH CLUSTERING ORDER BY (message_id DESC);

3. Fan-out Strategy:
   - Small groups: Fan-out on Write (push to each member's mailbox)
   - Large groups: Fan-out on Read (pull from group's message log)
```

### 4.2 Design Lazada/Shopee (E-Commerce)

```
คำถาม: "Design an e-commerce platform like Lazada 
supporting 10M products and 1M orders/day in Thailand"

ดู Part 92 สำหรับ Full Architecture

Key Services:
┌──────────────────────────────────────────────────────────────┐
│                    E-Commerce Services                        │
├──────────────────┬───────────────────────────────────────────┤
│ Service          │ Technology                                 │
├──────────────────┼───────────────────────────────────────────┤
│ Product Catalog  │ Elasticsearch + MySQL + Redis             │
│ User Service     │ PostgreSQL + Redis (Session)              │
│ Order Service    │ PostgreSQL + Kafka (Events)               │
│ Payment Service  │ PostgreSQL + External PSP (Omise/2C2P)   │
│ Inventory        │ Redis (Distributed Lock) + PostgreSQL     │
│ Search           │ Elasticsearch                             │
│ Recommendation   │ MongoDB + ML Model                        │
│ Notification     │ Kafka + Firebase + LINE Notify            │
└──────────────────┴───────────────────────────────────────────┘
```

### 4.3 Design Grab (Ride-hailing)

```
คำถาม: "Design Grab's driver matching system"

Core Challenge: Match Rider → Driver ใน Real-time

Location Storage Options:
1. Geohash (Redis GEOADD): ง่าย, เร็ว แต่ Boundary Problem
2. Quadtree: Dynamic partitioning, เหมาะกับ Dense areas
3. S2 Sphere (Google's): ใช้ใน Production จริง

Driver Matching Algorithm:
┌──────────────────────────────────────────────────────┐
│                 Matching Service                       │
│                                                        │
│  1. รับ Ride Request จาก Rider                        │
│  2. Query Redis GEOSEARCH (ใน 5km radius)             │
│  3. Filter ขับรถว่าง (Available Status)               │
│  4. คำนวณ ETA + Distance สำหรับแต่ละคนขับ            │
│  5. Sort by (ETA + rating_factor + surge_multiplier)  │
│  6. ส่ง Push Notification ไป 3 คนขับแรก              │
│  7. รอ Accept (30 seconds timeout)                    │
│  8. ถ้าไม่มีใช้ Accept → ขยาย Radius → Retry          │
└──────────────────────────────────────────────────────┘

// Redis Geospatial Commands
GEOADD drivers 100.5231 13.7367 "driver:001"  // lon lat member
GEOSEARCH drivers FROMMEMBER rider:req BYRADIUS 5 km ASC COUNT 10

// Driver Location Update (every 5 seconds)
GEOADD drivers 100.5234 13.7370 "driver:001"
EXPIRE drivers:session:001 30  // Auto-expire inactive drivers
```

---

## 5. Common Pitfalls in Interviews

### 5.1 กับดักที่ควรหลีกเลี่ยง

```
❌ Pitfall 1: เริ่ม Design โดยไม่ Clarify Requirements
   แก้: ถามเสมอ - "Can I ask a few clarifying questions first?"

❌ Pitfall 2: Over-engineer จากเริ่มต้น
   แก้: เริ่มด้วย Simple design แล้วค่อย Scale

❌ Pitfall 3: ไม่ประมาณ Scale ก่อน
   แก้: คำนวณ DAU, TPS, Storage ก่อนออกแบบ

❌ Pitfall 4: ลืม Non-functional Requirements
   แก้: ถามเสมอเรื่อง Availability, Latency, Consistency

❌ Pitfall 5: ไม่พูดถึง Trade-offs
   แก้: ทุก Decision ต้องบอก Why และ Trade-off

❌ Pitfall 6: ใช้ Technology โดยไม่รู้ว่าทำไม
   แก้: "เราใช้ Kafka เพราะต้องการ High Throughput และ Replay capability"

❌ Pitfall 7: ไม่ Identify Bottlenecks
   แก้: ถามตัวเองเสมอ "อะไรคือ Single Point of Failure?"
```

### 5.2 สิ่งที่ Interviewer ต้องการเห็น

```
✅ Communication Skills:
   - อธิบาย Thought process ชัดเจน
   - ถามคำถามที่ดี
   - Collaborate ไม่ใช่ทำคนเดียว

✅ Technical Depth:
   - รู้จัก Trade-offs
   - มีประสบการณ์จริง
   - ไม่ใช้ Buzzwords โดยไม่เข้าใจ

✅ Structured Thinking:
   - วิเคราะห์จากใหญ่ไปเล็ก
   - Prioritize ได้ถูกต้อง
   - มี Framework ในการแก้ปัญหา

✅ Adaptability:
   - รับ Feedback ได้
   - ปรับ Design ตาม Requirements ที่เปลี่ยน
   - ยอมรับเมื่อไม่รู้
```

### 5.3 คำถามที่ควรถาม Interviewer

```
คำถามดีๆ ที่แสดงให้เห็น Senior Thinking:

1. "How are we handling database schema migrations in this microservices setup?"
2. "What's the strategy for handling partial failures in the order flow?"
3. "How do we ensure data consistency when a service crashes mid-transaction?"
4. "What's the approach for end-to-end testing across services?"
5. "How do we handle the 'thundering herd' problem during traffic spikes?"
6. "What observability stack are you using?"
7. "How do you manage service dependencies and prevent cascading failures?"
```

### 5.4 Technical Interview Checklist

```
ก่อนสัมภาษณ์:
□ ทบทวน CAP Theorem
□ ทบทวน SOLID Principles
□ ฝึก System Design 5-10 ระบบ
□ รู้จัก Trade-offs ของ Technologies หลัก
□ ฝึก Whiteboard Coding (ไม่ใช้ IDE)
□ ทบทวน Distributed Systems Concepts

ระหว่างสัมภาษณ์:
□ ถาม Clarifying Questions ก่อนเสมอ
□ Think aloud - พูดความคิดออกมาเสมอ
□ เริ่มด้วย High-level แล้ว Drill down
□ Identify Trade-offs ทุก Decision
□ สังเกตว่า Interviewer ต้องการ Dive deeper ที่ไหน
□ จัดการเวลา - อย่า Over-design ส่วนเดียว

หลังสัมภาษณ์:
□ ส่ง Thank you note
□ Reflect สิ่งที่ทำได้ดี/ทำได้ดีกว่า
□ จดบันทึกคำถามที่ถามมาเพื่อเตรียมครั้งต่อไป
```

---

## 6. Resources และการเตรียมตัว

### 6.1 แนะนำหนังสือและ Resources

```
หนังสือที่ต้องอ่าน:
1. "Designing Data-Intensive Applications" - Martin Kleppmann
   → Bible ของ Distributed Systems

2. "System Design Interview" - Alex Xu Vol.1 & 2
   → Framework การสัมภาษณ์

3. "Building Microservices" - Sam Newman
   → Deep dive Microservices

4. "Clean Architecture" - Robert C. Martin
   → Architecture Principles

Online Resources:
- ByteByteGo (YouTube + Newsletter)
- High Scalability blog
- Netflix Tech Blog
- Uber Engineering Blog
- Martin Fowler's blog (martinfowler.com)

YouTube Channels:
- ByteByteGo
- Gaurav Sen (System Design)
- TechDummies Narendra L
```

### 6.2 Mock Interview Practice

```python
# Interview Practice Plan (4 สัปดาห์)

Week 1: Foundation
- Day 1-2: Review Distributed Systems basics
- Day 3-4: Practice URL Shortener, Pastebin
- Day 5-7: Review Databases (SQL, NoSQL, Cache)

Week 2: Core Patterns
- Day 1-2: Message Queue patterns (Kafka)
- Day 3-4: Design Chat System (WhatsApp/LINE)
- Day 5-7: Design Rate Limiter

Week 3: Complex Systems
- Day 1-2: Design Search Autocomplete
- Day 3-4: Design Uber/Grab Location service
- Day 5-7: Design YouTube/Netflix

Week 4: Review & Mock
- Day 1-3: Mock interviews with peers
- Day 4-5: Review weak areas
- Day 6-7: Rest & Mental preparation
```

---

## สรุป

การเตรียมตัวสัมภาษณ์ Microservices ต้องการทั้งความรู้เชิงทฤษฎีและทักษะการสื่อสาร สิ่งสำคัญที่ต้องจำ:

1. **ถามก่อนออกแบบ** - Clarify requirements เสมอ
2. **คำนวณ Scale** - Back-of-envelope calculation ก่อน
3. **รู้ Trade-offs** - ทุก Technology มี Trade-off
4. **Practice อย่างสม่ำเสมอ** - System Design เป็น Skill ที่ฝึกได้
5. **เรียนจาก Real Systems** - อ่าน Tech Blog ของบริษัทใหญ่

> "The best engineers are not those who know all the answers, but those who ask the right questions."

---

*ถัดไป: Part 92 - Case Study: E-Commerce Platform*

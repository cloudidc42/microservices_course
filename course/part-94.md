# Part 94: Case Study: Food Delivery System

## บทนำ

ระบบ Food Delivery คือหนึ่งในระบบที่ซับซ้อนที่สุดเนื่องจากต้องจัดการ Real-time Location, Matching Algorithm, และ Multi-party Coordination (ลูกค้า, ร้านอาหาร, พนักงานส่ง) พร้อมกัน บทนี้จะออกแบบระบบคล้าย GrabFood, Foodpanda หรือ Line Man

---

## 1. System Requirements

### 1.1 Functional Requirements

```
Core Features:
✅ สมัคร/Login (ลูกค้า, ร้านอาหาร, คนขับ)
✅ ค้นหาร้านอาหาร (ตาม Location, Category, Cuisine)
✅ ดูเมนู, สั่งอาหาร
✅ ชำระเงิน (PromptPay, บัตร, Wallet)
✅ Real-time Order Tracking (แผนที่จริงๆ)
✅ Matching: จับคู่คนขับ กับ ออเดอร์
✅ Push Notification (ทุก Status Change)
✅ Chat ระหว่าง ลูกค้า-คนขับ
✅ Rating & Review
✅ Promo Code / Discount
✅ Surge Pricing (ช่วงฝน/ชั่วโมงเร่งด่วน)
✅ Scheduled Orders
```

### 1.2 Scale Requirements

```
ระดับ GrabFood Thailand:
┌─────────────────────────────────────────────────────────────┐
│ Metric                │ Normal      │ Peak (Lunch/Dinner)   │
├───────────────────────┼─────────────┼───────────────────────┤
│ DAU                   │ 2M          │ 2M                    │
│ Active Drivers        │ 50,000      │ 100,000               │
│ Orders/hour           │ 50,000      │ 500,000               │
│ Location Updates/sec  │ 500,000     │ 1,000,000             │
│ API Latency (p99)     │ < 100ms     │ < 300ms               │
│ Matching Latency      │ < 5 seconds │ < 10 seconds          │
│ Availability          │ 99.99%      │ 99.99%                │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Food Delivery Platform                            │
│                                                                       │
│  ┌──────────────┐  ┌──────────────────┐  ┌───────────────────────┐ │
│  │Customer App  │  │  Restaurant App  │  │     Driver App        │ │
│  │(React Native)│  │  (React Native/  │  │  (React Native)       │ │
│  │              │  │   Web)           │  │  - GPS tracking       │ │
│  └──────┬───────┘  └─────────┬────────┘  └──────────┬────────────┘ │
│         └─────────────────────┴───────────────────────┘             │
│                               │                                      │
│                   ┌───────────▼──────────┐                          │
│                   │    Kong API Gateway   │                          │
│                   │  + WebSocket Proxy   │                          │
│                   └───────────┬──────────┘                          │
│                               │                                      │
│    ┌──────────────────────────┼─────────────────────────────────┐   │
│    │                          │                                  │   │
│  ┌─▼──────────┐  ┌────────────▼────┐  ┌──────────────────────┐ │   │
│  │User/Auth   │  │  Order Service  │  │  Location Service    │ │   │
│  │Service     │  │  (Saga)         │  │  (Real-time GPS)     │ │   │
│  └────────────┘  └────────┬────────┘  └──────────────────────┘ │   │
│                            │                                     │   │
│  ┌─────────────┐  ┌────────▼────────┐  ┌──────────────────────┐│   │
│  │Restaurant   │  │  Matching       │  │  Notification        ││   │
│  │Service      │  │  Service        │  │  Service             ││   │
│  └─────────────┘  └────────┬────────┘  └──────────────────────┘│   │
│                            │                                     │   │
│  ┌─────────────┐  ┌────────▼────────┐  ┌──────────────────────┐│   │
│  │Payment      │  │  Routing        │  │  Analytics           ││   │
│  │Service      │  │  Service (ETA)  │  │  Service             ││   │
│  └─────────────┘  └─────────────────┘  └──────────────────────┘│   │
│    └──────────────────────────────────────────────────────────┘    │
│                                                                       │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ Event Bus (Kafka)                                            │   │
│  │ Topics: order.*, driver.*, location.*, notification.*       │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. Real-time Order Tracking

### 3.1 Location Service Architecture

```
Location Data Flow:
Driver App ──(every 5 sec)──> Location Service ──> Redis Geo
                                                        │
                                                    Order Service
                                                        │
                                              WebSocket Server
                                                        │
                                               Customer App (Map)

Location Storage Decision:
- Redis GEOADD: O(log N) insert, O(N+log N) range query
- H3 Hexagonal Grid (Uber): Better for area queries
- S2 Cells (Google): Used in production

เราจะใช้ Redis GEO + H3 combination
```

### 3.2 Location Service Implementation

```go
// location-service/internal/service/location.go
package service

import (
    "context"
    "fmt"
    "time"
    
    "github.com/redis/go-redis/v9"
    "github.com/uber/h3-go/v4"
)

type LocationService struct {
    redis    *redis.ClusterClient
    kafka    EventPublisher
    orderSvc OrderServiceClient
}

type DriverLocation struct {
    DriverID  string
    Lat       float64
    Lng       float64
    Heading   float64
    Speed     float64
    Timestamp time.Time
    Status    string // available, busy, offline
}

// Update driver location (called every 5 seconds)
func (s *LocationService) UpdateDriverLocation(ctx context.Context, loc DriverLocation) error {
    pipe := s.redis.Pipeline()
    
    // 1. Update Redis GEO (for radius search)
    geoKey := "drivers:geo"
    pipe.GeoAdd(ctx, geoKey, &redis.GeoLocation{
        Name:      loc.DriverID,
        Longitude: loc.Lng,
        Latitude:  loc.Lat,
    })
    
    // 2. Update driver detail
    detailKey := fmt.Sprintf("driver:location:%s", loc.DriverID)
    pipe.HSet(ctx, detailKey, map[string]interface{}{
        "lat":       loc.Lat,
        "lng":       loc.Lng,
        "heading":   loc.Heading,
        "speed":     loc.Speed,
        "status":    loc.Status,
        "updated_at": loc.Timestamp.Unix(),
    })
    pipe.Expire(ctx, detailKey, 60*time.Second) // Auto-expire offline drivers
    
    // 3. H3 Index for zone-based operations
    h3Index := h3.LatLngToCell(h3.LatLng{Lat: loc.Lat, Lng: loc.Lng}, 8) // Resolution 8 ≈ 0.7km²
    zoneKey := fmt.Sprintf("zone:drivers:%s", h3Index)
    pipe.SAdd(ctx, zoneKey, loc.DriverID)
    pipe.Expire(ctx, zoneKey, 30*time.Second)
    
    if _, err := pipe.Exec(ctx); err != nil {
        return err
    }
    
    // 4. Publish location event for order tracking
    if loc.Status == "busy" {
        s.kafka.Publish(ctx, "driver.location.updated", DriverLocationEvent{
            DriverID:  loc.DriverID,
            Lat:       loc.Lat,
            Lng:       loc.Lng,
            Heading:   loc.Heading,
            Timestamp: loc.Timestamp,
        })
    }
    
    return nil
}

// Find available drivers near a location
func (s *LocationService) FindNearbyDrivers(ctx context.Context, lat, lng float64, radiusKm float64) ([]*DriverInfo, error) {
    results, err := s.redis.GeoSearch(ctx, "drivers:geo", &redis.GeoSearchQuery{
        Longitude:  lng,
        Latitude:   lat,
        Radius:     radiusKm,
        RadiusUnit: "km",
        Sort:       "ASC",
        Count:      50,
        WithCoord:  true,
        WithDist:   true,
    }).Result()
    
    if err != nil {
        return nil, err
    }
    
    var drivers []*DriverInfo
    for _, result := range results {
        // Get driver detail
        detailKey := fmt.Sprintf("driver:location:%s", result.Name)
        detail, err := s.redis.HGetAll(ctx, detailKey).Result()
        if err != nil || detail["status"] != "available" {
            continue
        }
        
        drivers = append(drivers, &DriverInfo{
            DriverID:     result.Name,
            Lat:          result.Latitude,
            Lng:          result.Longitude,
            DistanceKm:   result.Dist,
            Heading:      parseFloat(detail["heading"]),
            Speed:        parseFloat(detail["speed"]),
        })
    }
    
    return drivers, nil
}
```

### 3.3 WebSocket for Real-time Tracking

```go
// tracking-service/internal/websocket/hub.go
package websocket

import (
    "context"
    "encoding/json"
    "sync"
    "time"
    
    "github.com/gorilla/websocket"
    "github.com/redis/go-redis/v9"
)

type Hub struct {
    clients    map[string]*Client // orderID → client
    mu         sync.RWMutex
    redis      *redis.Client
    register   chan *Client
    unregister chan *Client
    broadcast  chan LocationUpdate
}

type Client struct {
    orderID string
    conn    *websocket.Conn
    send    chan []byte
}

func (h *Hub) Run(ctx context.Context) {
    // Subscribe to Redis Pub/Sub for driver location updates
    pubsub := h.redis.Subscribe(ctx, "driver:location:*")
    defer pubsub.Close()
    
    go func() {
        for msg := range pubsub.Channel() {
            var update LocationUpdate
            if err := json.Unmarshal([]byte(msg.Payload), &update); err != nil {
                continue
            }
            h.broadcast <- update
        }
    }()
    
    for {
        select {
        case client := <-h.register:
            h.mu.Lock()
            h.clients[client.orderID] = client
            h.mu.Unlock()
            
        case client := <-h.unregister:
            h.mu.Lock()
            delete(h.clients, client.orderID)
            close(client.send)
            h.mu.Unlock()
            
        case update := <-h.broadcast:
            h.mu.RLock()
            client, exists := h.clients[update.OrderID]
            h.mu.RUnlock()
            
            if exists {
                data, _ := json.Marshal(update)
                select {
                case client.send <- data:
                default:
                    // Client channel full, skip
                }
            }
            
        case <-ctx.Done():
            return
        }
    }
}

// WebSocket Handler
func (h *Hub) ServeWS(w http.ResponseWriter, r *http.Request) {
    orderID := chi.URLParam(r, "orderID")
    
    upgrader := websocket.Upgrader{
        CheckOrigin: func(r *http.Request) bool { return true },
    }
    
    conn, err := upgrader.Upgrade(w, r, nil)
    if err != nil {
        return
    }
    
    client := &Client{
        orderID: orderID,
        conn:    conn,
        send:    make(chan []byte, 256),
    }
    
    h.register <- client
    
    // Write pump
    go func() {
        defer func() {
            h.unregister <- client
            conn.Close()
        }()
        
        for {
            select {
            case msg, ok := <-client.send:
                if !ok {
                    conn.WriteMessage(websocket.CloseMessage, []byte{})
                    return
                }
                conn.WriteMessage(websocket.TextMessage, msg)
                
            case <-time.After(30 * time.Second):
                conn.WriteMessage(websocket.PingMessage, nil)
            }
        }
    }()
    
    // Read pump (handle pong)
    go func() {
        defer conn.Close()
        conn.SetReadDeadline(time.Now().Add(60 * time.Second))
        conn.SetPongHandler(func(string) error {
            conn.SetReadDeadline(time.Now().Add(60 * time.Second))
            return nil
        })
        
        for {
            _, _, err := conn.ReadMessage()
            if err != nil {
                break
            }
        }
    }()
}
```

---

## 4. Driver Matching Algorithm

### 4.1 Matching System Design

```
Matching Algorithm Requirements:
- ต้องจับคู่ภายใน 5-10 วินาที
- คำนึงถึง: Distance, Driver Rating, Estimated Pickup Time
- Handle concurrent orders efficiently
- Fair distribution ไม่ให้ Driver บางคนได้งานทุกครั้ง

Algorithm: Multi-factor Scoring
Score = w1*ETA + w2*Rating + w3*FairnessScore + w4*AcceptanceRate

ETA Calculation:
- Google Maps API / OpenStreetMap
- Historical traffic patterns
- Current traffic (TomTom/HERE API)
```

### 4.2 Matching Service Implementation

```go
// matching-service/internal/matcher/matcher.go
package matcher

import (
    "context"
    "sort"
    "time"
)

type MatchingService struct {
    locationSvc  LocationServiceClient
    routingSvc   RoutingServiceClient
    driverRepo   DriverRepository
    cache        *redis.Client
    notifier     NotificationService
}

type MatchRequest struct {
    OrderID        string
    RestaurantLat  float64
    RestaurantLng  float64
    CustomerLat    float64
    CustomerLng    float64
    EstimatedTime  int  // estimated food prep time (minutes)
    OrderValue     float64
}

type DriverCandidate struct {
    DriverID       string
    Lat            float64
    Lng            float64
    DistanceKm     float64
    ETA            int // minutes to restaurant
    Rating         float64
    TodayOrders    int
    AcceptanceRate float64
    Score          float64
}

func (s *MatchingService) Match(ctx context.Context, req MatchRequest) (*MatchResult, error) {
    // Step 1: Find nearby available drivers (radius = 5km initially)
    drivers, err := s.locationSvc.FindNearbyDrivers(ctx, 
        req.RestaurantLat, req.RestaurantLng, 5.0)
    if err != nil {
        return nil, err
    }
    
    if len(drivers) == 0 {
        // Expand radius to 10km
        drivers, err = s.locationSvc.FindNearbyDrivers(ctx,
            req.RestaurantLat, req.RestaurantLng, 10.0)
        if err != nil || len(drivers) == 0 {
            return nil, ErrNoDriversAvailable
        }
    }
    
    // Step 2: Get driver profiles and calculate ETAs
    candidates, err := s.enrichDriverData(ctx, drivers, req)
    if err != nil {
        return nil, err
    }
    
    // Step 3: Score and rank candidates
    rankedDrivers := s.rankDrivers(candidates, req)
    
    // Step 4: Offer to top 3 drivers (parallel)
    result := s.offerToDrivers(ctx, rankedDrivers[:min(3, len(rankedDrivers))], req)
    
    return result, nil
}

func (s *MatchingService) enrichDriverData(ctx context.Context, drivers []*DriverInfo, req MatchRequest) ([]*DriverCandidate, error) {
    // Batch get driver profiles
    driverIDs := make([]string, len(drivers))
    for i, d := range drivers {
        driverIDs[i] = d.DriverID
    }
    
    profiles, err := s.driverRepo.BatchGet(ctx, driverIDs)
    if err != nil {
        return nil, err
    }
    
    // Batch calculate ETAs (parallel)
    type etaResult struct {
        driverID string
        eta      int
        err      error
    }
    
    etaChan := make(chan etaResult, len(drivers))
    
    for _, driver := range drivers {
        go func(d *DriverInfo) {
            eta, err := s.routingSvc.CalculateETA(ctx, ETARequest{
                FromLat: d.Lat,
                FromLng: d.Lng,
                ToLat:   req.RestaurantLat,
                ToLng:   req.RestaurantLng,
            })
            etaChan <- etaResult{driverID: d.DriverID, eta: eta, err: err}
        }(driver)
    }
    
    etaMap := make(map[string]int)
    for range drivers {
        result := <-etaChan
        if result.err == nil {
            etaMap[result.driverID] = result.eta
        }
    }
    
    // Build candidates
    var candidates []*DriverCandidate
    profileMap := make(map[string]*DriverProfile)
    for _, p := range profiles {
        profileMap[p.DriverID] = p
    }
    
    for _, driver := range drivers {
        profile := profileMap[driver.DriverID]
        if profile == nil || !profile.IsActive {
            continue
        }
        
        candidates = append(candidates, &DriverCandidate{
            DriverID:       driver.DriverID,
            Lat:            driver.Lat,
            Lng:            driver.Lng,
            DistanceKm:     driver.DistanceKm,
            ETA:            etaMap[driver.DriverID],
            Rating:         profile.Rating,
            TodayOrders:    profile.TodayOrders,
            AcceptanceRate: profile.AcceptanceRate,
        })
    }
    
    return candidates, nil
}

func (s *MatchingService) rankDrivers(candidates []*DriverCandidate, req MatchRequest) []*DriverCandidate {
    for _, c := range candidates {
        // Normalize ETA (lower is better): 0-100
        etaScore := 100 - min(100, float64(c.ETA)*5) // 5 points per minute
        
        // Rating score (0-100)
        ratingScore := c.Rating * 20 // 5.0 → 100
        
        // Fairness: drivers with fewer orders today get bonus
        fairnessScore := 100 - float64(c.TodayOrders)*2 // penalty for high orders
        fairnessScore = max(0, fairnessScore)
        
        // Acceptance rate (drivers who accept more get priority)
        acceptanceScore := c.AcceptanceRate * 100
        
        // Weighted score
        c.Score = (etaScore * 0.40) +        // ETA: 40% weight
                  (ratingScore * 0.30) +      // Rating: 30% weight
                  (fairnessScore * 0.15) +    // Fairness: 15% weight
                  (acceptanceScore * 0.15)    // Acceptance: 15% weight
    }
    
    // Sort by score descending
    sort.Slice(candidates, func(i, j int) bool {
        return candidates[i].Score > candidates[j].Score
    })
    
    return candidates
}

// Send offer to drivers in parallel, first to accept wins
func (s *MatchingService) offerToDrivers(ctx context.Context, candidates []*DriverCandidate, req MatchRequest) *MatchResult {
    acceptChan := make(chan string, 1)
    timeout := time.After(30 * time.Second)
    
    // Offer to all candidates simultaneously
    for _, candidate := range candidates {
        go func(c *DriverCandidate) {
            // Send push notification with order details
            s.notifier.SendOrderOffer(ctx, c.DriverID, OrderOffer{
                OrderID:     req.OrderID,
                Distance:    c.DistanceKm,
                ETA:         c.ETA,
                OrderValue:  req.OrderValue,
                ExpiresIn:   30, // seconds
            })
            
            // Wait for driver response via Redis pub/sub
            response := s.waitForDriverResponse(ctx, c.DriverID, req.OrderID, 30*time.Second)
            if response == "accepted" {
                select {
                case acceptChan <- c.DriverID:
                default: // Another driver already accepted
                }
            }
        }(candidate)
    }
    
    select {
    case driverID := <-acceptChan:
        // Cancel offers to other drivers
        s.notifier.CancelOrderOffers(ctx, req.OrderID, driverID)
        return &MatchResult{
            OrderID:  req.OrderID,
            DriverID: driverID,
            Status:   "matched",
        }
    case <-timeout:
        return &MatchResult{
            OrderID: req.OrderID,
            Status:  "no_driver_found",
        }
    }
}
```

---

## 5. Order Management

### 5.1 Order State Machine

```
Order States:
PLACED
  │
  ▼
CONFIRMED_BY_RESTAURANT ───> CANCELLED_BY_RESTAURANT
  │
  ▼
BEING_PREPARED
  │
  ▼
READY_FOR_PICKUP
  │
  ▼ (Driver accepts)
DRIVER_ASSIGNED
  │
  ▼
PICKED_UP
  │
  ▼
ON_THE_WAY ────────────────> (can show real-time location)
  │
  ▼
DELIVERED
  │
  ▼
COMPLETED (after rating)
```

### 5.2 Order Service

```go
// order-service/internal/service/order.go
package service

type OrderService struct {
    repo         OrderRepository
    restaurantSvc RestaurantServiceClient
    paymentSvc   PaymentServiceClient
    matchingSvc  MatchingServiceClient
    notifySvc    NotificationServiceClient
    publisher    EventPublisher
}

func (s *OrderService) PlaceOrder(ctx context.Context, req PlaceOrderRequest) (*Order, error) {
    // Validate restaurant is open
    restaurant, err := s.restaurantSvc.GetRestaurant(ctx, req.RestaurantID)
    if err != nil || !restaurant.IsOpen {
        return nil, ErrRestaurantClosed
    }
    
    // Validate menu items and calculate total
    orderItems, totalAmount, err := s.validateAndCalculate(ctx, req.Items, req.RestaurantID)
    if err != nil {
        return nil, err
    }
    
    // Calculate delivery fee (based on distance)
    deliveryFee, err := s.calculateDeliveryFee(ctx, req.CustomerAddress, restaurant.Address)
    if err != nil {
        return nil, err
    }
    
    // Apply promo code if any
    discount := decimal.Zero
    if req.PromoCode != "" {
        discount, err = s.applyPromoCode(ctx, req.PromoCode, totalAmount, req.CustomerID)
        if err != nil {
            return nil, err
        }
    }
    
    finalAmount := totalAmount.Add(deliveryFee).Sub(discount)
    
    // Create order
    order := &Order{
        ID:           generateOrderID(), // GRAB-TH-20241111-123456
        CustomerID:   req.CustomerID,
        RestaurantID: req.RestaurantID,
        Items:        orderItems,
        Subtotal:     totalAmount,
        DeliveryFee:  deliveryFee,
        Discount:     discount,
        Total:        finalAmount,
        Status:       OrderStatusPlaced,
        Address:      req.CustomerAddress,
        Notes:        req.Notes,
        CreatedAt:    time.Now(),
    }
    
    if err := s.repo.Create(ctx, order); err != nil {
        return nil, err
    }
    
    // Process payment
    payment, err := s.paymentSvc.Charge(ctx, ChargeRequest{
        OrderID:  order.ID,
        Amount:   finalAmount,
        Method:   req.PaymentMethod,
        Token:    req.PaymentToken,
    })
    if err != nil {
        s.repo.UpdateStatus(ctx, order.ID, OrderStatusPaymentFailed)
        return nil, err
    }
    
    // Update order with payment info
    s.repo.UpdatePayment(ctx, order.ID, payment.PaymentID)
    
    // Notify restaurant (async)
    s.publisher.Publish(ctx, "order.placed", OrderPlacedEvent{
        OrderID:      order.ID,
        RestaurantID: req.RestaurantID,
        CustomerID:   req.CustomerID,
        Items:        orderItems,
        TotalAmount:  finalAmount,
    })
    
    // Notify customer
    s.notifySvc.SendOrderPlaced(ctx, req.CustomerID, order.ID)
    
    return order, nil
}

// Restaurant confirms order
func (s *OrderService) ConfirmOrder(ctx context.Context, orderID, restaurantID string, estimatedMinutes int) error {
    order, err := s.repo.GetByID(ctx, orderID)
    if err != nil {
        return err
    }
    
    if order.RestaurantID != restaurantID {
        return ErrUnauthorized
    }
    
    if order.Status != OrderStatusPlaced {
        return ErrInvalidStatus
    }
    
    // Update order
    s.repo.UpdateStatus(ctx, orderID, OrderStatusConfirmed)
    s.repo.SetEstimatedTime(ctx, orderID, estimatedMinutes)
    
    // Start finding driver (async - don't want to delay restaurant confirmation)
    go func() {
        ctx := context.Background()
        result, err := s.matchingSvc.Match(ctx, MatchRequest{
            OrderID:       orderID,
            RestaurantLat: order.Restaurant.Lat,
            RestaurantLng: order.Restaurant.Lng,
            CustomerLat:   order.Customer.Lat,
            CustomerLng:   order.Customer.Lng,
            EstimatedTime: estimatedMinutes,
            OrderValue:    order.Total.InexactFloat64(),
        })
        
        if err != nil || result.Status == "no_driver_found" {
            s.notifySvc.AlertNoDriver(ctx, orderID)
            return
        }
        
        s.repo.UpdateDriver(ctx, orderID, result.DriverID)
        s.repo.UpdateStatus(ctx, orderID, OrderStatusDriverAssigned)
        
        // Notify customer about driver
        s.notifySvc.SendDriverAssigned(ctx, order.CustomerID, orderID, result.DriverID)
    }()
    
    // Notify customer
    s.notifySvc.SendOrderConfirmed(ctx, order.CustomerID, orderID, estimatedMinutes)
    
    return nil
}
```

---

## 6. Notification System

### 6.1 Multi-channel Notification

```go
// notification-service/internal/service/notifier.go
package service

import (
    "context"
    "encoding/json"
)

type NotificationService struct {
    fcm       FCMClient      // Firebase (iOS/Android Push)
    lineNotify LineNotifyClient  // LINE Notify
    sms       SMSClient      // Twilio SMS
    kafka     EventPublisher
}

type Notification struct {
    UserID  string
    Type    string  // push, line, sms
    Title   string
    Body    string
    Data    map[string]string
}

func (s *NotificationService) SendOrderStatusUpdate(ctx context.Context, userID string, status OrderStatus) error {
    user, err := s.getUserPreferences(ctx, userID)
    if err != nil {
        return err
    }
    
    messages := buildStatusMessages(status)
    
    var errs []error
    
    // Push Notification (สำคัญที่สุด)
    if user.PushEnabled && user.FCMToken != "" {
        if err := s.fcm.Send(ctx, FCMMessage{
            Token: user.FCMToken,
            Notification: FCMNotification{
                Title: messages.Title,
                Body:  messages.Body,
            },
            Data: map[string]string{
                "order_id": status.OrderID,
                "status":   string(status.Status),
                "type":     "order_update",
            },
            Android: &AndroidConfig{
                Priority: "HIGH",
                Notification: &AndroidNotification{
                    Sound:       "default",
                    ChannelID:   "order_updates",
                    ClickAction: "OPEN_ORDER",
                },
            },
            APNS: &APNSConfig{
                Payload: &APNSPayload{
                    APS: &APS{
                        Sound:   "default",
                        Badge:   1,
                        ContentAvailable: true,
                    },
                },
            },
        }); err != nil {
            errs = append(errs, err)
        }
    }
    
    // LINE Notify (ถ้าลูกค้า Connect LINE)
    if user.LineToken != "" {
        lineMsg := fmt.Sprintf("🍔 %s\n%s\nออเดอร์: %s", 
            messages.Title, messages.Body, status.OrderID)
        if err := s.lineNotify.Send(ctx, user.LineToken, lineMsg); err != nil {
            errs = append(errs, err)
        }
    }
    
    // SMS fallback (สำหรับ Driver เท่านั้น หรือเมื่อ Push fail)
    if user.IsDriver && user.PhoneNumber != "" {
        if err := s.sms.Send(ctx, user.PhoneNumber, messages.Body); err != nil {
            errs = append(errs, err)
        }
    }
    
    return nil
}

func buildStatusMessages(status OrderStatus) NotificationMessages {
    templates := map[string]NotificationMessages{
        "confirmed": {
            Title: "ร้านอาหารรับออเดอร์แล้ว!",
            Body:  fmt.Sprintf("ประมาณ %d นาที อาหารจะพร้อม", status.EstimatedMinutes),
        },
        "driver_assigned": {
            Title: "พบคนส่งแล้ว!",
            Body:  fmt.Sprintf("%s กำลังมารับอาหาร", status.DriverName),
        },
        "picked_up": {
            Title: "คนส่งรับอาหารแล้ว",
            Body:  fmt.Sprintf("กำลังเดินทางมาหาคุณ ใช้เวลาประมาณ %d นาที", status.ETAMinutes),
        },
        "delivered": {
            Title: "อาหารมาถึงแล้ว! 🎉",
            Body:  "กรุณาให้คะแนนการบริการด้วยนะคะ",
        },
    }
    
    msg, exists := templates[status.Status]
    if !exists {
        return NotificationMessages{
            Title: "อัปเดตออเดอร์",
            Body:  fmt.Sprintf("สถานะออเดอร์: %s", status.Status),
        }
    }
    return msg
}
```

---

## 7. Geolocation Services

### 7.1 ETA Calculation

```go
// routing-service/internal/service/eta.go
package service

import (
    "context"
    "time"
)

type ETAService struct {
    googleMaps GoogleMapsClient
    tomtom     TomTomClient
    cache      *redis.Client
    mlModel    ETAModel
}

func (s *ETAService) CalculateETA(ctx context.Context, req ETARequest) (*ETAResult, error) {
    // Cache key based on approximate location (rounded to 2 decimal places)
    cacheKey := fmt.Sprintf("eta:%.2f:%.2f:%.2f:%.2f",
        req.FromLat, req.FromLng, req.ToLat, req.ToLng)
    
    // Check cache (1 minute TTL for traffic-sensitive data)
    if cached, err := s.cache.Get(ctx, cacheKey).Bytes(); err == nil {
        var result ETAResult
        if json.Unmarshal(cached, &result) == nil {
            return &result, nil
        }
    }
    
    // Get ETA from Google Maps
    googleResult, err := s.googleMaps.GetDirections(ctx, DirectionsRequest{
        Origin:      fmt.Sprintf("%f,%f", req.FromLat, req.FromLng),
        Destination: fmt.Sprintf("%f,%f", req.ToLat, req.ToLng),
        Mode:        "driving",
        DepartureTime: "now",
        TrafficModel: "best_guess",
    })
    
    baseETA := 10 // default minutes
    if err == nil {
        baseETA = googleResult.Duration.Minutes
    }
    
    // ML adjustment based on historical data
    // (time of day, day of week, weather, area)
    adjustedETA := s.mlModel.Adjust(baseETA, AdjustmentContext{
        Hour:       time.Now().Hour(),
        DayOfWeek:  int(time.Now().Weekday()),
        AreaCode:   s.getAreaCode(req.FromLat, req.FromLng),
    })
    
    result := &ETAResult{
        MinutesMin: adjustedETA - 2,
        MinutesMid: adjustedETA,
        MinutesMax: adjustedETA + 5,
        Distance:   googleResult.Distance,
    }
    
    // Cache result
    if data, err := json.Marshal(result); err == nil {
        s.cache.Set(ctx, cacheKey, data, 60*time.Second)
    }
    
    return result, nil
}
```

### 7.2 Surge Pricing

```go
// pricing-service/internal/service/surge.go
package service

import (
    "context"
    "math"
)

type SurgePricingService struct {
    locationSvc LocationService
    orderRepo   OrderRepository
    cache       *redis.Client
}

// คำนวณ Surge Multiplier ตาม Supply/Demand
func (s *SurgePricingService) GetSurgeMultiplier(ctx context.Context, lat, lng float64) (float64, error) {
    // ใช้ H3 Hexagonal cell เพื่อ group area
    h3Cell := h3.LatLngToCell(h3.LatLng{Lat: lat, Lng: lng}, 7) // ~5km²
    cacheKey := fmt.Sprintf("surge:%s", h3Cell)
    
    // Check cache (update every 5 minutes)
    if cached, err := s.cache.Get(ctx, cacheKey).Float64(); err == nil {
        return cached, nil
    }
    
    // Get supply (available drivers) in area
    drivers, err := s.locationSvc.FindNearbyDrivers(ctx, lat, lng, 3.0)
    if err != nil {
        return 1.0, nil // No surge on error
    }
    
    // Get demand (pending orders) in area
    pendingOrders, err := s.orderRepo.GetPendingInArea(ctx, lat, lng, 3.0)
    if err != nil {
        return 1.0, nil
    }
    
    supply := len(drivers)
    demand := pendingOrders
    
    // Calculate ratio
    var ratio float64
    if supply == 0 {
        ratio = 3.0 // Max surge when no drivers
    } else {
        ratio = float64(demand) / float64(supply)
    }
    
    // Apply surge formula
    // ratio < 0.5: 1.0x (plenty of drivers)
    // ratio 0.5-1.0: 1.0-1.5x
    // ratio 1.0-2.0: 1.5-2.0x
    // ratio > 2.0: 2.0x cap
    var multiplier float64
    switch {
    case ratio < 0.5:
        multiplier = 1.0
    case ratio < 1.0:
        multiplier = 1.0 + (ratio * 0.5)
    case ratio < 2.0:
        multiplier = 1.5 + ((ratio - 1.0) * 0.5)
    default:
        multiplier = 2.0 // Cap at 2x
    }
    
    // Round to nearest 0.1
    multiplier = math.Round(multiplier*10) / 10
    
    // Cache for 5 minutes
    s.cache.Set(ctx, cacheKey, multiplier, 5*time.Minute)
    
    return multiplier, nil
}
```

---

## 8. Analytics Pipeline

```python
# analytics-service/app/pipeline/order_analytics.py
from pyspark.sql import SparkSession
from pyspark.sql import functions as F
from pyspark.sql.types import *

class OrderAnalyticsPipeline:
    def __init__(self):
        self.spark = SparkSession.builder \
            .appName("FoodDeliveryAnalytics") \
            .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension") \
            .getOrCreate()
    
    def process_daily_metrics(self, date: str):
        """Process daily order metrics"""
        
        # Read from Kafka (batch mode for daily aggregation)
        orders_df = self.spark.read \
            .format("kafka") \
            .option("kafka.bootstrap.servers", "kafka:9092") \
            .option("subscribe", "order.completed") \
            .option("startingOffsets", f'{{"order.completed":{{"0":{date_to_offset(date)}}}}}') \
            .load()
        
        orders = orders_df.selectExpr("CAST(value AS STRING)") \
            .select(F.from_json("value", order_schema).alias("order")) \
            .select("order.*")
        
        # Calculate key metrics
        daily_metrics = orders.groupBy(
            F.date_trunc("hour", "created_at").alias("hour")
        ).agg(
            F.count("*").alias("total_orders"),
            F.sum("total_amount").alias("gmv"),
            F.avg("delivery_time_minutes").alias("avg_delivery_time"),
            F.avg("rating").alias("avg_rating"),
            F.countDistinct("customer_id").alias("unique_customers"),
            F.countDistinct("restaurant_id").alias("active_restaurants"),
            F.countDistinct("driver_id").alias("active_drivers"),
            F.sum(F.when(F.col("status") == "cancelled", 1).otherwise(0)).alias("cancellations"),
        )
        
        # Restaurant performance
        restaurant_metrics = orders.groupBy("restaurant_id").agg(
            F.count("*").alias("order_count"),
            F.sum("total_amount").alias("revenue"),
            F.avg("rating").alias("avg_rating"),
            F.avg("prep_time_minutes").alias("avg_prep_time"),
        )
        
        # Driver performance
        driver_metrics = orders.groupBy("driver_id").agg(
            F.count("*").alias("deliveries"),
            F.sum("delivery_fee").alias("earnings"),
            F.avg("delivery_time_minutes").alias("avg_delivery_time"),
            F.avg("driver_rating").alias("avg_rating"),
        )
        
        # Write to BigQuery
        daily_metrics.write \
            .format("bigquery") \
            .option("table", f"analytics.daily_metrics_{date}") \
            .save()
        
        restaurant_metrics.write \
            .format("bigquery") \
            .option("table", f"analytics.restaurant_metrics_{date}") \
            .save()
        
        return {
            "total_orders": daily_metrics.agg(F.sum("total_orders")).collect()[0][0],
            "total_gmv": daily_metrics.agg(F.sum("gmv")).collect()[0][0],
        }
```

---

## 9. Production Checklist

```
□ Pre-launch Checklist:

Infrastructure:
□ Load testing (5x expected peak)
□ Circuit breakers configured
□ Rate limiting on all APIs
□ Database connection pooling
□ Redis cluster mode enabled
□ Kafka topic partitioning sized correctly
□ CDN configured for static assets

Reliability:
□ Health checks on all services
□ Graceful shutdown handling
□ Retry with exponential backoff
□ Dead letter queues for failed messages
□ Database read replicas

Monitoring:
□ Latency P50/P95/P99 dashboards
□ Error rate alerts
□ Driver location staleness alerts
□ Order assignment failure alerts
□ Payment failure rate alerts

Security:
□ JWT expiry set correctly
□ API keys rotated
□ Driver location data encrypted
□ PII data masked in logs
□ Rate limiting per user and per IP
```

---

## สรุป

การออกแบบ Food Delivery System ต้องให้ความสำคัญกับ:

1. **Real-time Location** - Redis GEO + WebSocket สำหรับ live tracking
2. **Matching Algorithm** - Multi-factor scoring ที่ยุติธรรมทั้งลูกค้าและคนขับ
3. **Order State Machine** - ชัดเจน, Idempotent, เหมาะกับ Microservices
4. **Notification System** - Multi-channel (Push, LINE, SMS)
5. **Surge Pricing** - Dynamic pricing ตาม Supply/Demand
6. **Analytics Pipeline** - Real-time + Batch สำหรับ Business Insights

> "Great food delivery is about reliability and trust — customers trust their food arrives, drivers trust they get paid, restaurants trust orders are real"

---

*ถัดไป: Part 95 - Case Study: Streaming Platform*

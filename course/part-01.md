# Part 01: Introduction to Microservices
## แนะนำ Microservices - ทำความรู้จักสถาปัตยกรรมที่เปลี่ยนโลก Software

> **ระดับ:** ⭐ เริ่มต้น | **เวลาเรียน:** 3-4 ชั่วโมง | **Prerequisites:** ความรู้พื้นฐาน Programming

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Microservices คืออะไร และทำไมถึงสำคัญ
- ประวัติและวิวัฒนาการของสถาปัตยกรรม Software
- เปรียบเทียบ Monolithic vs Microservices
- ตัวอย่างบริษัทระดับโลกที่ใช้ Microservices
- เมื่อไหร่ควรใช้ Microservices และเมื่อไหร่ไม่ควรใช้

---

## 1. Microservices คืออะไร?

**Microservices Architecture** คือวิธีการออกแบบและพัฒนา Software โดยแบ่งระบบขนาดใหญ่ออกเป็น **บริการเล็กๆ ที่ทำงานอิสระจากกัน** แต่ละบริการ (Service) มีหน้าที่รับผิดชอบเฉพาะส่วน และสื่อสารกันผ่าน Network API

### นิยามอย่างเป็นทางการ

จาก Martin Fowler (ผู้บัญญัติศัพท์ Microservices):

> *"The microservice architectural style is an approach to developing a single application as a suite of small services, each running in its own process and communicating with lightweight mechanisms, often an HTTP resource API."*

### ลักษณะสำคัญของ Microservices

```
┌─────────────────────────────────────────────────────────────┐
│                    MICROSERVICES SYSTEM                      │
│                                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │  User    │  │  Order   │  │ Payment  │  │Inventory │  │
│  │ Service  │  │ Service  │  │ Service  │  │ Service  │  │
│  │  :3001   │  │  :3002   │  │  :3003   │  │  :3004   │  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  │
│       │              │              │              │         │
│  ┌────▼─────┐  ┌────▼─────┐  ┌────▼─────┐  ┌────▼─────┐  │
│  │  Users   │  │  Orders  │  │Payments  │  │Products  │  │
│  │   DB     │  │   DB     │  │   DB     │  │   DB     │  │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘  │
└─────────────────────────────────────────────────────────────┘
```

แต่ละ Service:
1. **ทำงานแยกกัน** (Independent Process)
2. **มี Database ของตัวเอง** (Database per Service)
3. **สื่อสารผ่าน API** (REST, gRPC, Message Queue)
4. **Deploy แยกกันได้** (Independent Deployability)
5. **Scale แยกกันได้** (Independent Scalability)

---

## 2. ประวัติและวิวัฒนาการ

### ยุคที่ 1: Mainframe Computing (1960s-1980s)
```
┌─────────────────────────────────┐
│           MAINFRAME             │
│  ┌─────────────────────────┐   │
│  │   ALL BUSINESS LOGIC    │   │
│  │   ALL DATA              │   │
│  │   ALL PROCESSING        │   │
│  └─────────────────────────┘   │
└─────────────────────────────────┘
         Terminal Terminal Terminal
```
ทุกอย่างรวมอยู่ใน Mainframe เครื่องเดียว ง่ายแต่ไม่ยืดหยุ่น

### ยุคที่ 2: Client-Server Architecture (1980s-1990s)
```
┌──────────┐     Network     ┌──────────────┐
│  Client  │◄───────────────►│   Server     │
│ (UI/App) │                 │ (Business +  │
└──────────┘                 │  Database)   │
                             └──────────────┘
```
แยก Client ออกจาก Server ทำให้ยืดหยุ่นขึ้น

### ยุคที่ 3: Three-Tier / N-Tier Architecture (1990s-2000s)
```
┌──────────┐
│Presentation│  ← Frontend / UI
└─────┬─────┘
      │
┌─────▼─────┐
│  Business  │  ← Application Server
│   Logic    │
└─────┬─────┘
      │
┌─────▼─────┐
│   Data     │  ← Database Server
│   Layer    │
└────────────┘
```

### ยุคที่ 4: Service-Oriented Architecture / SOA (2000s)
```
┌──────────────────────────────────────────┐
│           Enterprise Service Bus (ESB)   │
├──────────┬───────────┬────────────────────┤
│ Service A│ Service B │ Service C           │
└──────────┴───────────┴────────────────────┘
```
SOA เป็นต้นแบบของ Microservices แต่ซับซ้อนและหนักกว่า

### ยุคที่ 5: Monolithic Web Applications (2000s-2010s)
```
┌────────────────────────────────────────────┐
│              MONOLITHIC APP                 │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐   │
│  │  Users   │ │  Orders  │ │  Products│   │
│  │  Module  │ │  Module  │ │  Module  │   │
│  └──────────┘ └──────────┘ └──────────┘   │
│                                             │
│  ┌──────────────────────────────────────┐  │
│  │         Shared Database              │  │
│  └──────────────────────────────────────┘  │
└────────────────────────────────────────────┘
```

### ยุคที่ 6: Microservices Era (2010s-ปัจจุบัน)
Amazon, Netflix, Uber นำร่องด้วย Microservices
```
Service A ──► Service B ──► Service C
    │                           │
    └──────────────────────────┘
         (through API / Message Queue)
```

---

## 3. ทำไม Netflix, Amazon, Uber ถึงเลือก Microservices?

### 🎬 Netflix - กรณีศึกษา

**ปัญหาเดิม (2008):** Netflix ใช้ Monolithic Application และเกิดปัญหาใหญ่
- Database corruption ทำให้ระบบ down 3 วัน
- ไม่สามารถ scale เฉพาะส่วนที่ต้องการได้
- Deployment ใหม่แต่ละครั้งเสี่ยงมาก

**การเปลี่ยนแปลง (2009-2012):**
- แยกระบบออกเป็น Microservices กว่า 700 services
- ใช้ AWS Cloud
- สร้างเครื่องมือของตัวเอง: Hystrix, Eureka, Zuul

**ผลลัพธ์:**
- รองรับ 200+ ล้าน subscribers ทั่วโลก
- Stream 100+ ล้านชั่วโมงต่อวัน
- Deploy 1000+ ครั้งต่อวัน

### 🛒 Amazon - กรณีศึกษา

Jeff Bezos ออก Memo ที่โด่งดัง (Bezos Mandate) ว่า:
1. ทุก Team ต้องเปิดเผย Data และ Functionality ผ่าน Service Interfaces
2. Services ต้องสื่อสารผ่าน Interfaces เหล่านี้เท่านั้น
3. ไม่อนุญาตให้ใช้ Direct Linking หรือ Direct Database Access ระหว่าง Services
4. ไม่ว่าจะใช้ Technology อะไร: HTTP, Corba, Pubsub, etc.
5. **ไม่ทำตาม = ไล่ออก**

ผลลัพธ์: Amazon กลายเป็นผู้นำ Cloud Computing ผ่าน AWS

### 🚗 Uber - กรณีศึกษา

**ปัญหาเดิม:** ระบบ Monolith ที่เริ่มในปี 2010
- เพิ่ม Feature ใหม่แต่ละครั้ง ใช้เวลานาน
- Bug ใน Module หนึ่ง กระทบทั้งระบบ
- Developer ทำงานขัดกัน

**Microservices ของ Uber (2018+):**
- 2,200+ Microservices
- 4,000+ Engineers
- Deploy หลายพันครั้งต่อวัน

---

## 4. แนวคิดหลักของ Microservices

### 4.1 Single Responsibility Principle (SRP)
แต่ละ Service ทำหน้าที่เดียว ทำให้ดี

```
❌ ไม่ดี - God Service
┌────────────────────────────────────┐
│         UserOrderService           │
│  - Create User                     │
│  - Delete User                     │
│  - Create Order                    │
│  - Calculate Price                 │
│  - Send Email                      │
│  - Process Payment                 │
└────────────────────────────────────┘

✅ ดี - Single Responsibility
┌──────────┐  ┌──────────┐  ┌──────────┐
│   User   │  │  Order   │  │  Email   │
│ Service  │  │ Service  │  │ Service  │
└──────────┘  └──────────┘  └──────────┘
```

### 4.2 Loose Coupling (การเชื่อมต่อแบบหลวมๆ)
Services ไม่ควรรู้รายละเอียดภายในของ Service อื่น

```python
# ❌ Tight Coupling - Service A รู้รายละเอียด Database ของ Service B
class OrderService:
    def get_user(self, user_id):
        # เชื่อมตรงกับ Database ของ User Service!
        return db.query("SELECT * FROM users WHERE id = %s", user_id)

# ✅ Loose Coupling - ใช้ API
class OrderService:
    def get_user(self, user_id):
        # เรียกผ่าน API เท่านั้น
        response = requests.get(f"http://user-service/users/{user_id}")
        return response.json()
```

### 4.3 High Cohesion (ความเกาะเกี่ยวภายใน)
Code ที่เกี่ยวข้องกันควรอยู่ใน Service เดียวกัน

```
✅ High Cohesion - User Service
user-service/
├── routes/
│   ├── user.routes.js      # User API endpoints
│   └── auth.routes.js      # Authentication endpoints
├── models/
│   └── user.model.js       # User data model
├── services/
│   └── user.service.js     # User business logic
└── repositories/
    └── user.repository.js  # Database operations
```

### 4.4 Decentralized Data Management
แต่ละ Service มี Database ของตัวเอง

```
┌────────────────────────────────────────────────────────┐
│  User Service  │  Order Service  │  Payment Service    │
│  ┌──────────┐  │  ┌──────────┐  │  ┌──────────┐       │
│  │PostgreSQL│  │  │ MongoDB  │  │  │  MySQL   │       │
│  └──────────┘  │  └──────────┘  │  └──────────┘       │
└────────────────────────────────────────────────────────┘
```

---

## 5. ข้อดีและข้อเสียของ Microservices

### ✅ ข้อดี

#### 1. Independent Deployment
```bash
# Deploy เฉพาะ payment-service โดยไม่กระทบ services อื่น
docker push myapp/payment-service:v2.1.0
kubectl set image deployment/payment-service \
  payment-service=myapp/payment-service:v2.1.0
```

#### 2. Technology Flexibility (Polyglot)
```
User Service   → Node.js + PostgreSQL
Order Service  → Python + MongoDB
Payment Service → Java Spring Boot + MySQL
Search Service → Go + Elasticsearch
ML Service     → Python + TensorFlow
```

#### 3. Independent Scaling
```yaml
# Scale เฉพาะ payment service ที่มี Load สูง
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
spec:
  replicas: 10  # Scale up payment service
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
spec:
  replicas: 2   # user service ไม่ต้องการ replicas มาก
```

#### 4. Fault Isolation
```
❌ Monolith: Payment bug → ทั้งระบบ down

✅ Microservices: Payment service down →
   - User service ยังทำงานได้
   - Order service ยังทำงานได้ (บันทึก order รอ payment ฟื้นตัว)
   - Notification service ยังส่ง email ได้
```

#### 5. Team Independence (Conway's Law)
```
Team A → User Service
Team B → Order Service  
Team C → Payment Service
```
แต่ละทีมทำงานอิสระ ไม่ต้อง coordinate กันมาก

### ❌ ข้อเสีย

#### 1. Distributed System Complexity
```
Monolith: Function A calls Function B → Simple!

Microservices: Service A calls Service B over Network
  - Network latency
  - Service B might be down
  - Need retry logic
  - Need circuit breaker
  - Need distributed tracing
```

#### 2. Data Consistency
```sql
-- Monolith: Transaction ง่ายมาก
BEGIN TRANSACTION;
  UPDATE orders SET status = 'confirmed';
  UPDATE inventory SET stock = stock - 1;
  INSERT INTO payments VALUES (...);
COMMIT;

-- Microservices: ต้องใช้ Saga Pattern
-- ซับซ้อนกว่ามาก!
```

#### 3. Testing Complexity
```
Monolith: Unit Test + Integration Test = done

Microservices:
- Unit Test (each service)
- Integration Test (between services)
- Contract Test (API compatibility)
- End-to-End Test (entire flow)
- Performance Test (distributed)
```

#### 4. Operational Overhead
```
Monolith: 1 application to monitor

Microservices:
- 20+ services to monitor
- 20+ databases to manage
- Complex networking
- Service discovery
- Load balancing
- Distributed logging
```

---

## 6. เมื่อไหร่ควรใช้ Microservices?

### ✅ ควรใช้เมื่อ:

1. **ทีมขนาดใหญ่** (50+ developers)
   - แต่ละทีมสามารถ own service ของตัวเองได้

2. **ระบบที่ต้องการ Scale แตกต่างกัน**
   - Search ต้องการ 100 instances แต่ Admin ใช้ 2 instances

3. **ต้องการ Deploy บ่อย**
   - Deploy ใหม่ทุกวันหรือทุกชั่วโมง

4. **แต่ละส่วนมีความต้องการ Technology ต่างกัน**
   - ML Service ต้องการ Python/GPU
   - Realtime Service ต้องการ Node.js/WebSocket

5. **Business Domain ชัดเจน**
   - User, Order, Payment, Inventory แยกกันได้ชัดเจน

### ❌ ไม่ควรใช้เมื่อ:

1. **ทีมเล็ก** (< 10 developers)
   - Operational overhead สูงเกินไป

2. **ระบบใหม่ที่ยังไม่รู้ Domain ชัดเจน**
   - "Don't start with Microservices" - Martin Fowler

3. **Domain Logic ยังไม่นิ่ง**
   - ถ้า Service boundaries เปลี่ยนบ่อย จะยิ่งซับซ้อน

4. **ไม่มี DevOps/Infrastructure Expertise**
   - ต้องการความรู้ Docker, Kubernetes, CI/CD

5. **Simple Application**
   - Todo App, Blog ไม่จำเป็นต้องใช้ Microservices

### 🎯 The Monolith First Approach

Martin Fowler แนะนำว่า:
```
Phase 1: เริ่มด้วย Monolith ที่ดี (Well-structured Monolith)
Phase 2: เข้าใจ Domain ให้ชัดเจน
Phase 3: แยกออกเป็น Microservices เมื่อจำเป็น
```

---

## 7. Microservices vs SOA

| คุณสมบัติ | SOA | Microservices |
|-----------|-----|---------------|
| ขนาด Service | ใหญ่ | เล็ก |
| Communication | Enterprise Service Bus (ESB) | Direct API |
| Data Management | Shared Database | Database per Service |
| Deployment | Monolithic WAR/EAR | Container (Docker) |
| Technology | Homogeneous | Polyglot |
| เน้น | Reuse | Autonomy |

---

## 8. Real-World Microservices Architecture

### ตัวอย่าง: E-Commerce Platform

```
                         ┌──────────────────┐
                         │   API Gateway    │
                         │   (Kong/Nginx)   │
                         └────────┬─────────┘
                                  │
              ┌───────────────────┼───────────────────┐
              │                   │                   │
    ┌─────────▼──────┐  ┌────────▼────────┐  ┌───────▼────────┐
    │   User Service  │  │  Product Service │  │  Order Service │
    │  (Node.js/PG)   │  │  (Go/MongoDB)    │  │  (Java/MySQL)  │
    └────────┬────────┘  └────────┬─────────┘  └───────┬────────┘
             │                    │                     │
    ┌────────▼────────┐           │            ┌───────▼────────┐
    │  Auth Service   │           │            │Payment Service │
    │  (Node.js/Redis)│           │            │ (Python/PG)    │
    └─────────────────┘           │            └───────┬────────┘
                                  │                    │
                        ┌─────────▼────────────────────▼────┐
                        │         Message Broker (Kafka)     │
                        └─────────────────────────────────────┘
                                  │
                   ┌──────────────┴──────────────┐
          ┌────────▼────────┐          ┌──────────▼────────┐
          │ Email Service   │          │Inventory Service   │
          │ (Node.js)       │          │ (Go/Cassandra)     │
          └─────────────────┘          └───────────────────┘
```

### Flow การสั่งซื้อสินค้า:
```
1. User → API Gateway → Order Service: POST /orders
2. Order Service → Product Service: GET /products/{id} (ตรวจสอบสินค้า)
3. Order Service → User Service: GET /users/{id} (ตรวจสอบข้อมูล user)
4. Order Service → Payment Service: POST /payments
5. Payment Service → Order Service: Payment Success Event
6. Order Service → Kafka: Publish "order.confirmed" event
7. Inventory Service ← Kafka: Subscribe "order.confirmed" → Update stock
8. Email Service ← Kafka: Subscribe "order.confirmed" → Send confirmation email
```

---

## 9. เริ่มต้นเรียนรู้: Workshop แรก

มาลอง Run ตัวอย่าง Microservices ง่ายๆ กัน

### Prerequisites
```bash
# ตรวจสอบว่ามี Node.js
node --version  # v18+

# ตรวจสอบว่ามี Docker
docker --version  # 20+

# ตรวจสอบว่ามี Docker Compose
docker-compose --version  # 2+
```

### โครงสร้าง Project
```
simple-microservices/
├── user-service/
│   ├── index.js
│   └── package.json
├── order-service/
│   ├── index.js
│   └── package.json
└── docker-compose.yml
```

### User Service (user-service/index.js)
```javascript
const express = require('express');
const app = express();
app.use(express.json());

// In-memory database (สำหรับ demo)
const users = [
  { id: 1, name: 'สมชาย ใจดี', email: 'somchai@example.com' },
  { id: 2, name: 'สมหญิง รักดี', email: 'somying@example.com' },
];

// Get all users
app.get('/users', (req, res) => {
  res.json({ 
    success: true, 
    data: users,
    service: 'user-service',
    timestamp: new Date().toISOString()
  });
});

// Get user by ID
app.get('/users/:id', (req, res) => {
  const user = users.find(u => u.id === parseInt(req.params.id));
  
  if (!user) {
    return res.status(404).json({ 
      success: false, 
      message: 'User not found' 
    });
  }
  
  res.json({ success: true, data: user });
});

// Create user
app.post('/users', (req, res) => {
  const { name, email } = req.body;
  
  if (!name || !email) {
    return res.status(400).json({ 
      success: false, 
      message: 'Name and email are required' 
    });
  }
  
  const newUser = {
    id: users.length + 1,
    name,
    email
  };
  
  users.push(newUser);
  res.status(201).json({ success: true, data: newUser });
});

// Health check endpoint
app.get('/health', (req, res) => {
  res.json({ 
    status: 'healthy', 
    service: 'user-service',
    uptime: process.uptime()
  });
});

const PORT = process.env.PORT || 3001;
app.listen(PORT, () => {
  console.log(`🚀 User Service running on port ${PORT}`);
});
```

### User Service (user-service/package.json)
```json
{
  "name": "user-service",
  "version": "1.0.0",
  "description": "Microservice for User Management",
  "main": "index.js",
  "scripts": {
    "start": "node index.js",
    "dev": "nodemon index.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  },
  "devDependencies": {
    "nodemon": "^3.0.1"
  }
}
```

### Order Service (order-service/index.js)
```javascript
const express = require('express');
const axios = require('axios');
const app = express();
app.use(express.json());

// In-memory orders
const orders = [];

// User Service URL (จาก environment variable)
const USER_SERVICE_URL = process.env.USER_SERVICE_URL || 'http://localhost:3001';

// Get all orders
app.get('/orders', (req, res) => {
  res.json({ 
    success: true, 
    data: orders,
    service: 'order-service'
  });
});

// Get order by ID
app.get('/orders/:id', (req, res) => {
  const order = orders.find(o => o.id === parseInt(req.params.id));
  
  if (!order) {
    return res.status(404).json({ 
      success: false, 
      message: 'Order not found' 
    });
  }
  
  res.json({ success: true, data: order });
});

// Create order
app.post('/orders', async (req, res) => {
  const { userId, products, totalAmount } = req.body;
  
  if (!userId || !products || !totalAmount) {
    return res.status(400).json({ 
      success: false, 
      message: 'userId, products, and totalAmount are required' 
    });
  }
  
  try {
    // เรียก User Service เพื่อตรวจสอบ User (Service-to-Service Communication)
    const userResponse = await axios.get(`${USER_SERVICE_URL}/users/${userId}`);
    const user = userResponse.data.data;
    
    const newOrder = {
      id: orders.length + 1,
      userId,
      userInfo: {
        name: user.name,
        email: user.email
      },
      products,
      totalAmount,
      status: 'pending',
      createdAt: new Date().toISOString()
    };
    
    orders.push(newOrder);
    
    res.status(201).json({ 
      success: true, 
      data: newOrder,
      message: 'Order created successfully'
    });
    
  } catch (error) {
    if (error.response?.status === 404) {
      return res.status(404).json({ 
        success: false, 
        message: 'User not found' 
      });
    }
    
    // User Service ไม่ตอบสนอง
    res.status(503).json({ 
      success: false, 
      message: 'User service unavailable. Please try again later.',
      error: error.message
    });
  }
});

// Health check
app.get('/health', (req, res) => {
  res.json({ 
    status: 'healthy', 
    service: 'order-service',
    uptime: process.uptime(),
    dependencies: {
      userService: USER_SERVICE_URL
    }
  });
});

const PORT = process.env.PORT || 3002;
app.listen(PORT, () => {
  console.log(`🚀 Order Service running on port ${PORT}`);
});
```

### Order Service (order-service/package.json)
```json
{
  "name": "order-service",
  "version": "1.0.0",
  "description": "Microservice for Order Management",
  "main": "index.js",
  "scripts": {
    "start": "node index.js",
    "dev": "nodemon index.js"
  },
  "dependencies": {
    "axios": "^1.4.0",
    "express": "^4.18.2"
  },
  "devDependencies": {
    "nodemon": "^3.0.1"
  }
}
```

### Docker Compose (docker-compose.yml)
```yaml
version: '3.8'

services:
  user-service:
    build: ./user-service
    ports:
      - "3001:3001"
    environment:
      - PORT=3001
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3001/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  order-service:
    build: ./order-service
    ports:
      - "3002:3002"
    environment:
      - PORT=3002
      - USER_SERVICE_URL=http://user-service:3001
    depends_on:
      user-service:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3002/health"]
      interval: 30s
      timeout: 10s
      retries: 3
```

### Dockerfile สำหรับแต่ละ Service
สร้าง `Dockerfile` ใน user-service/ และ order-service/
```dockerfile
# Dockerfile (ใช้ได้ทั้ง 2 services)
FROM node:18-alpine

WORKDIR /app

# Copy package files first (Docker layer caching)
COPY package*.json ./

# Install dependencies
RUN npm install --production

# Copy source code
COPY . .

# Expose port
EXPOSE 3001

# Start application
CMD ["node", "index.js"]
```

### รัน Workshop
```bash
# 1. สร้างโครงสร้าง directory
mkdir -p simple-microservices/user-service
mkdir -p simple-microservices/order-service

# 2. สร้างไฟล์ (copy code ด้านบน)

# 3. Run ด้วย Docker Compose
cd simple-microservices
docker-compose up --build

# 4. ทดสอบ APIs
# User Service
curl http://localhost:3001/users
curl http://localhost:3001/users/1
curl -X POST http://localhost:3001/users \
  -H "Content-Type: application/json" \
  -d '{"name": "ทดสอบ", "email": "test@example.com"}'

# Order Service
curl -X POST http://localhost:3002/orders \
  -H "Content-Type: application/json" \
  -d '{
    "userId": 1,
    "products": [{"id": 1, "name": "MacBook", "quantity": 1}],
    "totalAmount": 50000
  }'

# Health Checks
curl http://localhost:3001/health
curl http://localhost:3002/health
```

### ผลลัพธ์ที่คาดหวัง
```json
// POST /orders
{
  "success": true,
  "data": {
    "id": 1,
    "userId": 1,
    "userInfo": {
      "name": "สมชาย ใจดี",
      "email": "somchai@example.com"
    },
    "products": [{"id": 1, "name": "MacBook", "quantity": 1}],
    "totalAmount": 50000,
    "status": "pending",
    "createdAt": "2024-01-01T10:00:00.000Z"
  },
  "message": "Order created successfully"
}
```

---

## 10. สรุปบทที่ 1

### Key Takeaways

1. **Microservices** คือการแบ่งระบบใหญ่เป็น Services เล็กๆ ที่ทำงานอิสระ
2. **ข้อดี:** Independent deployment, Technology flexibility, Scalability, Fault isolation
3. **ข้อเสีย:** Distributed complexity, Testing difficulty, Operational overhead
4. **ใช้เมื่อ:** ทีมใหญ่, ต้องการ Scale ต่างกัน, Domain ชัดเจน
5. **ไม่ใช้เมื่อ:** ทีมเล็ก, ระบบใหม่, Domain ไม่ชัด

### คำถามทบทวน

1. Microservices ต่างจาก Monolith อย่างไร?
2. บริษัทระดับโลกอย่าง Netflix เปลี่ยนไปใช้ Microservices เพราะอะไร?
3. เมื่อไหร่ที่ **ไม่ควร** ใช้ Microservices?
4. "Database per Service Pattern" หมายความว่าอะไร?
5. ในตัวอย่าง Workshop Order Service สื่อสารกับ User Service อย่างไร?

### แบบฝึกหัด

1. ปรับปรุง User Service ให้เพิ่ม endpoint `DELETE /users/:id`
2. เพิ่ม endpoint ใน Order Service ที่สามารถ update status ของ order ได้
3. ทดสอบสิ่งที่เกิดขึ้นเมื่อ User Service down แต่ Order Service ยังทำงาน
4. เพิ่ม `Product Service` ที่มี endpoint GET /products และ GET /products/:id

---

## 📚 อ่านเพิ่มเติม

- [Martin Fowler - Microservices](https://martinfowler.com/articles/microservices.html)
- [Microservices.io Patterns](https://microservices.io/patterns/index.html)
- [Netflix Tech Blog](https://netflixtechblog.com/)
- [Uber Engineering Blog](https://www.uber.com/en-US/blog/engineering/)

---

**ต่อไป:** [Part 02 - Monolithic vs Microservices Architecture →](part-02.md)

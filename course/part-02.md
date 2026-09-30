# Part 02: Monolithic vs Microservices Architecture
## เปรียบเทียบเชิงลึก: เลือกแบบไหนดีสำหรับโปรเจคของคุณ?

> **ระดับ:** ⭐ เริ่มต้น | **เวลาเรียน:** 3 ชั่วโมง | **Prerequisites:** Part 01

---

## 🎯 สิ่งที่จะได้เรียนรู้

- ทำความเข้าใจ Monolithic Architecture อย่างลึกซึ้ง
- เปรียบเทียบ Monolithic vs Microservices ในทุกมิติ
- เรียนรู้ Modular Monolith (ทางเลือกที่ดี)
- Decision Framework สำหรับเลือกสถาปัตยกรรม
- Workshop: Refactor Monolith เป็น Microservices

---

## 1. Monolithic Architecture

### 1.1 คืออะไร?

Monolithic Application คือ Application ที่รวมทุกอย่างไว้ใน **Codebase เดียว** และ **Deploy เป็น Unit เดียว**

```
┌─────────────────────────────────────────────────────────┐
│                   MONOLITHIC APPLICATION                 │
│                                                         │
│  ┌───────────────┐   ┌───────────────┐                  │
│  │  Presentation │   │     Web UI    │                  │
│  │     Layer     │   │   (React/Vue) │                  │
│  └───────┬───────┘   └───────┬───────┘                  │
│          │                   │                           │
│  ┌───────▼───────────────────▼───────┐                  │
│  │          Business Logic Layer      │                  │
│  │  ┌──────────┐ ┌──────────┐        │                  │
│  │  │  Users   │ │  Orders  │        │                  │
│  │  │  Module  │ │  Module  │        │                  │
│  │  └──────────┘ └──────────┘        │                  │
│  │  ┌──────────┐ ┌──────────┐        │                  │
│  │  │ Products │ │ Payments │        │                  │
│  │  │  Module  │ │  Module  │        │                  │
│  │  └──────────┘ └──────────┘        │                  │
│  └───────────────────────────────────┘                  │
│                                                         │
│  ┌───────────────────────────────────┐                  │
│  │          Data Access Layer        │                  │
│  └───────────────────────────────────┘                  │
│                                                         │
│  ┌───────────────────────────────────┐                  │
│  │          Shared Database          │                  │
│  │     (Single PostgreSQL/MySQL)     │                  │
│  └───────────────────────────────────┘                  │
└─────────────────────────────────────────────────────────┘
```

### 1.2 โครงสร้างของ Monolith ทั่วไป

```
ecommerce-monolith/
├── src/
│   ├── controllers/
│   │   ├── UserController.js
│   │   ├── OrderController.js
│   │   ├── ProductController.js
│   │   └── PaymentController.js
│   ├── models/
│   │   ├── User.js
│   │   ├── Order.js
│   │   ├── Product.js
│   │   └── Payment.js
│   ├── services/
│   │   ├── UserService.js
│   │   ├── OrderService.js
│   │   ├── ProductService.js
│   │   └── PaymentService.js
│   ├── repositories/
│   │   ├── UserRepository.js
│   │   ├── OrderRepository.js
│   │   └── ProductRepository.js
│   ├── middleware/
│   │   ├── auth.js
│   │   └── validation.js
│   └── app.js
├── database/
│   ├── migrations/
│   └── seeds/
├── tests/
│   ├── unit/
│   └── integration/
├── package.json
└── Dockerfile
```

### 1.3 ตัวอย่าง Monolith Code

```javascript
// app.js - Entry point ของ Monolith
const express = require('express');
const { Pool } = require('pg');

const app = express();
app.use(express.json());

// Database connection เดียวสำหรับทั้งระบบ
const db = new Pool({
  host: process.env.DB_HOST || 'localhost',
  database: process.env.DB_NAME || 'ecommerce',
  user: process.env.DB_USER || 'postgres',
  password: process.env.DB_PASSWORD,
  port: 5432,
});

// ──── USER ROUTES ────
app.get('/users', async (req, res) => {
  const result = await db.query('SELECT * FROM users');
  res.json(result.rows);
});

app.get('/users/:id', async (req, res) => {
  const { id } = req.params;
  const result = await db.query('SELECT * FROM users WHERE id = $1', [id]);
  if (!result.rows.length) return res.status(404).json({ error: 'Not found' });
  res.json(result.rows[0]);
});

// ──── ORDER ROUTES ────
app.post('/orders', async (req, res) => {
  const { userId, productId, quantity } = req.body;
  
  // Monolith สามารถ query หลาย tables ใน transaction เดียวได้ง่ายมาก
  const client = await db.connect();
  try {
    await client.query('BEGIN');
    
    // ตรวจสอบ User
    const userResult = await client.query(
      'SELECT * FROM users WHERE id = $1', [userId]
    );
    if (!userResult.rows.length) throw new Error('User not found');
    
    // ตรวจสอบ Product และ Stock
    const productResult = await client.query(
      'SELECT * FROM products WHERE id = $1 AND stock >= $2',
      [productId, quantity]
    );
    if (!productResult.rows.length) throw new Error('Product not available');
    
    const product = productResult.rows[0];
    const totalAmount = product.price * quantity;
    
    // สร้าง Order
    const orderResult = await client.query(
      `INSERT INTO orders (user_id, product_id, quantity, total_amount, status)
       VALUES ($1, $2, $3, $4, 'pending') RETURNING *`,
      [userId, productId, quantity, totalAmount]
    );
    
    // ลด Stock
    await client.query(
      'UPDATE products SET stock = stock - $1 WHERE id = $2',
      [quantity, productId]
    );
    
    await client.query('COMMIT');
    res.status(201).json(orderResult.rows[0]);
    
  } catch (error) {
    await client.query('ROLLBACK');
    res.status(400).json({ error: error.message });
  } finally {
    client.release();
  }
});

// ──── PRODUCT ROUTES ────
app.get('/products', async (req, res) => {
  const result = await db.query('SELECT * FROM products WHERE stock > 0');
  res.json(result.rows);
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => console.log(`App running on port ${PORT}`));
```

### 1.4 ข้อดีของ Monolith

#### ✅ Simple Development
```bash
# Clone และ Run ได้เลย
git clone https://github.com/myapp/ecommerce.git
cd ecommerce
npm install
npm run dev

# ไม่ต้อง start หลาย services
# ไม่ต้อง manage network
# Debug ง่าย
```

#### ✅ Simple Testing
```javascript
// Integration Test ง่ายมากใน Monolith
describe('Order Creation', () => {
  it('should create order and update stock', async () => {
    // Setup
    await db.query('INSERT INTO users VALUES (1, "test@test.com")');
    await db.query('INSERT INTO products VALUES (1, "MacBook", 50000, 5)');
    
    // Execute
    const response = await request(app)
      .post('/orders')
      .send({ userId: 1, productId: 1, quantity: 1 });
    
    // Assert - ทุกอย่างอยู่ใน DB เดียวกัน
    expect(response.status).toBe(201);
    
    const product = await db.query('SELECT * FROM products WHERE id = 1');
    expect(product.rows[0].stock).toBe(4); // stock ลดลง 1
  });
});
```

#### ✅ Simple Deployment
```yaml
# docker-compose.yml สำหรับ Monolith
version: '3.8'
services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgres://...
  
  postgres:
    image: postgres:15
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

#### ✅ No Network Latency
```
Monolith: OrderService → UserService = Function Call (nanoseconds)
Microservices: Order Service → User Service = HTTP Request (milliseconds)
```

### 1.5 ข้อเสียของ Monolith

#### ❌ Scaling ทั้ง Application
```
ปัญหา: Payment processing ต้องการ CPU มาก
แต่ต้อง Scale ทั้ง Application

Monolith Scaling:
┌─────────────────────────────────────────┐
│  Monolith Instance 1 (All Modules)      │
├─────────────────────────────────────────┤
│  Monolith Instance 2 (All Modules)      │ ← scale ทั้งหมด แม้
├─────────────────────────────────────────┤   จะต้องการเฉพาะ Payment
│  Monolith Instance 3 (All Modules)      │   = สิ้นเปลือง resource
└─────────────────────────────────────────┘
```

#### ❌ Technology Lock-in
```
ถ้าเริ่มด้วย Node.js + PostgreSQL
→ ทุกส่วนต้องใช้ Node.js + PostgreSQL

ไม่สามารถเลือกใช้:
- Python สำหรับ ML features
- Go สำหรับ High-performance parts
- MongoDB สำหรับ Document storage
```

#### ❌ Code Complexity เมื่อระบบใหญ่ขึ้น
```
เมื่อ Codebase ใหญ่:
- เพิ่ม Feature ใหม่ยาก (กลัวกระทบส่วนอื่น)
- Build time นาน (แก้ 1 บรรทัด → Build ทั้ง project)
- IDE ช้า (เปิด project 100,000 ไฟล์)
- Onboard developer ใหม่ยาก
```

#### ❌ Deployment Risk สูง
```
แก้ไข Bug เล็กน้อยใน Payment Module
→ ต้อง Deploy ทั้ง Application
→ Risk สูงที่จะ Break ส่วนอื่น
→ ต้อง Test ทั้งระบบทุกครั้ง
```

---

## 2. Microservices Architecture (ทบทวน)

### 2.1 โครงสร้าง

```
ecommerce-microservices/
├── user-service/
│   ├── src/
│   ├── Dockerfile
│   └── package.json
├── order-service/
│   ├── src/
│   ├── Dockerfile
│   └── package.json
├── product-service/
│   ├── src/
│   ├── Dockerfile
│   └── package.json
├── payment-service/
│   ├── src/
│   ├── Dockerfile
│   └── package.json
├── api-gateway/
│   ├── nginx.conf
│   └── Dockerfile
└── docker-compose.yml
```

### 2.2 Scaling แบบ Microservices
```yaml
# scale payment-service เฉพาะส่วนที่ต้องการ
docker-compose up --scale payment-service=5
docker-compose up --scale user-service=1

# ผล:
# payment-service: 5 instances (High load)
# user-service: 1 instance (Low load)
# order-service: 1 instance (Normal load)
```

---

## 3. การเปรียบเทียบเชิงลึก

### 3.1 Development Speed

| ช่วงเวลา | Monolith | Microservices |
|---------|---------|---------------|
| เริ่มต้น Project (0-6 เดือน) | ✅ เร็วกว่า | ❌ ช้ากว่า (setup overhead) |
| Project เติบโต (6-18 เดือน) | ⚖️ เท่ากัน | ⚖️ เท่ากัน |
| Project ใหญ่ (18+ เดือน) | ❌ ช้ากว่า | ✅ เร็วกว่า |

### 3.2 Team Structure

**Monolith Team Structure:**
```
┌─────────────────────────────────────────────────────┐
│                   Backend Team                       │
│  Dev1 Dev2 Dev3 Dev4 Dev5 Dev6 Dev7 Dev8 Dev9 Dev10  │
│         (ทุกคนทำงานใน Codebase เดียวกัน)              │
└─────────────────────────────────────────────────────┘
ปัญหา: Merge conflicts, Code ownership ไม่ชัดเจน
```

**Microservices Team Structure (Conway's Law):**
```
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  Team Users  │  │  Team Orders │  │ Team Payment │
│  Dev1 Dev2   │  │  Dev3 Dev4   │  │  Dev5 Dev6   │
│  Dev3        │  │  Dev5        │  │  Dev7        │
└──────────────┘  └──────────────┘  └──────────────┘
แต่ละทีม own service ของตัวเอง
```

### 3.3 Database Strategy

**Monolith:**
```sql
-- ง่ายมาก - query ข้ามตารางได้เลย
SELECT 
  u.name,
  o.total_amount,
  p.name as product_name,
  pay.status as payment_status
FROM orders o
  JOIN users u ON o.user_id = u.id
  JOIN products p ON o.product_id = p.id
  LEFT JOIN payments pay ON o.id = pay.order_id
WHERE o.created_at > '2024-01-01'
ORDER BY o.created_at DESC;
```

**Microservices:**
```javascript
// ต้องทำ "API Composition" หรือ "CQRS"
async function getOrderDetails(orderId) {
  const [order, user, product, payment] = await Promise.all([
    orderService.getOrder(orderId),           // HTTP call
    userService.getUser(order.userId),         // HTTP call  
    productService.getProduct(order.productId), // HTTP call
    paymentService.getPayment(orderId)          // HTTP call
  ]);
  
  return { order, user, product, payment };
}
// ซับซ้อนกว่า แต่แต่ละ service เป็นอิสระ
```

### 3.4 Fault Tolerance

**Monolith Failure:**
```
Memory leak ใน Payment Module
↓
ทั้ง Application crash
↓
ทุก feature ใช้ไม่ได้ (Users, Orders, Products, etc.)
```

**Microservices Failure:**
```
Payment Service crash
↓
Order Service ยังทำงาน (บันทึก order, รอ retry payment)
User Service ยังทำงาน (login, profile)
Product Service ยังทำงาน (browse products)
↓
ผู้ใช้บางส่วนยังได้รับบริการ
```

### 3.5 Testing Pyramid

```
         Monolith                    Microservices
           /\                             /\
          /  \                           /  \
         / E2E\                         / E2E\
        /──────\                       /──────\
       /Integra-\                     /Contract\
      /  tion    \                   /──────────\
     /────────────\                 /Integration \
    /   Unit Tests  \              /──────────────\
   /──────────────────\           /   Unit Tests   \
  /────────────────────\         /──────────────────\
```

### 3.6 Deployment Comparison

**Monolith Deployment:**
```
1. รอ Developer ทุกคน merge code
2. Run full test suite (อาจใช้ 1-2 ชั่วโมง)
3. Deploy ทั้ง Application
4. Rolling restart ทุก instance
5. Rollback = revert ทั้ง version
```

**Microservices Deployment:**
```
1. Team Payment เสร็จงาน
2. Run tests เฉพาะ payment-service (10 นาที)
3. Deploy payment-service เท่านั้น
4. Services อื่นไม่ถูกกระทบ
5. Rollback เฉพาะ payment-service
```

---

## 4. Modular Monolith - ทางเลือกที่ดี

**Modular Monolith** คือแนวทางที่ได้ประโยชน์จากทั้งสองฝ่าย

### 4.1 แนวคิด
```
┌─────────────────────────────────────────────────────────┐
│                   MODULAR MONOLITH                       │
│                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │  User Module  │  │ Order Module │  │Payment Module│  │
│  │              │  │              │  │              │  │
│  │  - Own model │  │  - Own model │  │  - Own model │  │
│  │  - Own logic │  │  - Own logic │  │  - Own logic │  │
│  │  - Own tests │  │  - Own tests │  │  - Own tests │  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  │
│         │                 │                  │           │
│         └─────────────────┴──────────────────┘           │
│                           │                               │
│               Module API (Internal Interface)             │
│                                                         │
│  ┌─────────────────────────────────────────────────┐    │
│  │                Shared Database                   │    │
│  │           (แต่แยก schema/table prefix)            │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘

Deploy เป็น Unit เดียว แต่ Code แยก Module ชัดเจน
```

### 4.2 โครงสร้าง Modular Monolith
```
modular-ecommerce/
├── modules/
│   ├── users/
│   │   ├── api/
│   │   │   ├── routes.js
│   │   │   └── validators.js
│   │   ├── domain/
│   │   │   ├── User.js         # Domain model
│   │   │   └── UserService.js  # Business logic
│   │   ├── infrastructure/
│   │   │   └── UserRepository.js
│   │   └── index.js            # Public interface ของ module
│   │
│   ├── orders/
│   │   ├── api/
│   │   ├── domain/
│   │   ├── infrastructure/
│   │   └── index.js
│   │
│   └── payments/
│       ├── api/
│       ├── domain/
│       ├── infrastructure/
│       └── index.js
│
├── shared/
│   ├── database/
│   ├── middleware/
│   └── utils/
│
└── app.js
```

### 4.3 Module Interface
```javascript
// modules/users/index.js - Public Interface
// ✅ Module อื่นเรียกผ่าน Interface นี้เท่านั้น
module.exports = {
  // API Routes
  routes: require('./api/routes'),
  
  // Public Domain Operations
  async getUser(id) {
    return UserService.getById(id);
  },
  
  async validateUser(id) {
    const user = await UserService.getById(id);
    return user !== null;
  }
};

// modules/orders/domain/OrderService.js
const UserModule = require('../users'); // ✅ เรียกผ่าน Module Interface

class OrderService {
  async createOrder(userId, productId) {
    // ✅ ไม่ access UserRepository โดยตรง
    const userExists = await UserModule.validateUser(userId);
    if (!userExists) throw new Error('User not found');
    
    // ... create order logic
  }
}
```

### 4.4 ข้อดีของ Modular Monolith
- ✅ Simple Deployment (เหมือน Monolith)
- ✅ Simple Testing (เหมือน Monolith)  
- ✅ Clear Code Boundaries (เหมือน Microservices)
- ✅ Easy to extract to Microservices เมื่อพร้อม

---

## 5. Decision Framework

### 5.1 The Architecture Choice Matrix

```
┌─────────────────────────────────────────────────────────────┐
│                  TEAM SIZE                                   │
│           Small         Medium          Large                │
│         (< 10 dev)   (10-50 dev)      (50+ dev)             │
├─────────────────┬──────────────────┬──────────────────────── │
│ Simple Domain   │   Monolith       │  Modular Monolith       │
│                 │   ✅ Recommended  │  ✅ Recommended          │
├─────────────────┼──────────────────┼──────────────────────── │
│ Complex Domain  │   Modular        │  Microservices          │
│                 │   Monolith       │  ✅ Recommended          │
│                 │   ✅ Recommended  │                          │
├─────────────────┼──────────────────┼──────────────────────── │
│ High Scale Req. │   Consider       │  Microservices          │
│ Different Parts │   Microservices  │  ✅ Recommended          │
└─────────────────┴──────────────────┴──────────────────────── │
```

### 5.2 Checklist ก่อนเลือก Microservices

ก่อนตัดสินใจใช้ Microservices ตอบคำถามเหล่านี้:

**Team & Organization:**
- [ ] มีทีมมากกว่า 3 ทีม (แต่ละทีม 3+ คน)?
- [ ] แต่ละทีม can own และ operate service ของตัวเองได้?
- [ ] มี DevOps/Platform team สนับสนุน?

**Technical Requirements:**
- [ ] ต้องการ Scale แต่ละส่วนต่างกัน?
- [ ] ต้องการใช้ Technology ที่แตกต่างกันใน แต่ละส่วน?
- [ ] มีความต้องการ High Availability (ถ้าส่วนหนึ่ง down ส่วนอื่นยังทำงาน)?

**Business Domain:**
- [ ] Business Domain ชัดเจน (Users, Orders, Products แยกกันได้)?
- [ ] แต่ละ Domain มีทีม ownership ชัดเจน?
- [ ] Domain boundaries stable (ไม่เปลี่ยนบ่อย)?

**Infrastructure:**
- [ ] มี Container Orchestration (Kubernetes, ECS)?
- [ ] มี CI/CD Pipeline?
- [ ] มี Monitoring & Observability?
- [ ] มี Service Discovery?

**ถ้าตอบ "ใช่" น้อยกว่า 70% → เริ่มด้วย Monolith ก่อน**

---

## 6. Strangler Fig Pattern - การ Migrate จาก Monolith

เมื่อมี Monolith อยู่แล้ว และต้องการ Migrate เป็น Microservices

### 6.1 แนวคิด

```
Step 1: Monolith ทั้งหมด
┌──────────────────────────────────┐
│          MONOLITH                │
│  Users | Orders | Products | Pay │
└──────────────────────────────────┘

Step 2: Extract Payment เป็น Service แรก
┌─────────────────────────┐  ┌──────────────┐
│          MONOLITH        │  │   Payment    │
│  Users | Orders | Products│  │   Service   │
└─────────────────────────┘  └──────────────┘
                    ↑  Routes new payment requests here

Step 3: Extract Products
┌────────────────┐  ┌──────────────┐  ┌──────────────┐
│    MONOLITH    │  │   Payment    │  │   Products   │
│  Users | Orders│  │   Service   │  │   Service   │
└────────────────┘  └──────────────┘  └──────────────┘

Step 4: Eventually Migrate Everything
┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│    Users     │  │   Orders     │  │   Products   │  │   Payment    │
│   Service   │  │   Service   │  │   Service   │  │   Service   │
└──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘
```

### 6.2 Step-by-Step Guide

```javascript
// Step 1: ระบุ Bounded Contexts ใน Monolith
// ค้นหาส่วนที่มี cohesion สูง และ coupling ต่ำกับส่วนอื่น

// Step 2: สร้าง Strangler Fig Facade (API Gateway)
// nginx.conf
location /api/payments/ {
  # Route ไปที่ payment microservice
  proxy_pass http://payment-service:3003/;
}

location /api/ {
  # Routes อื่นยังไปที่ monolith
  proxy_pass http://monolith:3000/;
}

// Step 3: Extract Payment Feature
// payment-service/src/index.js
// (copy และ refactor payment code จาก monolith)

// Step 4: เพิ่ม Feature Flags เพื่อ Switch Traffic
const USE_PAYMENT_SERVICE = process.env.USE_PAYMENT_SERVICE === 'true';

async function processPayment(orderId, amount) {
  if (USE_PAYMENT_SERVICE) {
    return await paymentServiceClient.process(orderId, amount);
  } else {
    return await legacyPaymentProcessor.process(orderId, amount);
  }
}
```

---

## 7. Anti-patterns ที่ควรหลีกเลี่ยง

### 7.1 Distributed Monolith (Worst of Both Worlds)
```
❌ Anti-pattern: Services ที่ depend กันมากเกินไป
Payment Service → User Service → Order Service → Product Service
                    ↑
                    Order Service ยัง depend กลับ!
                    (Circular Dependency)

การ Deploy service เดียว = ต้อง deploy ทั้งหมด
= Monolith ที่แย่กว่าเดิม!
```

### 7.2 Nano Services
```
❌ Anti-pattern: Services เล็กเกินไป
get-user-name-service
get-user-email-service  
update-user-name-service
update-user-email-service
validate-user-service

✅ ถูกต้อง: User Service ที่ handle ทุกอย่างเกี่ยวกับ User
user-service (GET /users, POST /users, PUT /users/:id, etc.)
```

### 7.3 Shared Database
```
❌ Anti-pattern: Services share database
┌──────────────┐  ┌──────────────┐
│ User Service │  │ Order Service│
└──────┬───────┘  └──────┬───────┘
       └─────────┬────────┘
                 ▼
         ┌──────────────┐
         │ Shared DB    │  ← ทำให้ tight coupling
         └──────────────┘     ไม่สามารถ deploy/scale อิสระได้
```

### 7.4 Chatty Services
```
❌ Anti-pattern: Service call กันมากเกินไป
สร้าง Order:
1. Order Service → User Service (1 call)
2. Order Service → Product Service (1 call)
3. Order Service → Inventory Service (1 call)
4. Order Service → Pricing Service (1 call)
5. Order Service → Discount Service (1 call)
6. Order Service → Tax Service (1 call)
= 6 Network calls ต่อ request!

✅ ถูกต้อง: รวม services ที่ควรอยู่ด้วยกัน
หรือใช้ Event-driven architecture
```

---

## 8. Workshop: Analyze and Improve

### 8.1 วิเคราะห์ Monolith ที่มีอยู่

```javascript
// ❌ Bad Monolith Code - ปัญหาอะไรบ้าง?
class OrderController {
  async createOrder(req, res) {
    const { userId, items } = req.body;
    
    // Direct DB query (ควรใช้ Repository Pattern)
    const user = await db.query(`SELECT * FROM users WHERE id = ${userId}`);
    
    // SQL Injection vulnerability!
    
    // Business logic ใน Controller (ควรแยกออก)
    const total = items.reduce((sum, item) => sum + item.price * item.qty, 0);
    
    // Hard-coded business rules
    if (total > 100000) {
      total = total * 0.9; // 10% discount
    }
    
    // Email sending ใน Controller (ควรแยกเป็น Service)
    await sendEmail(user.email, `Your order total: ${total}`);
    
    // Payment processing ใน Controller
    const payment = await stripe.charge(user.cardToken, total);
    
    // ทุกอย่างใน function เดียว = God Function
    await db.query(`INSERT INTO orders ...`);
    
    res.json({ success: true });
  }
}
```

### 8.2 Refactor เป็น Well-Structured Monolith

```javascript
// ✅ Better Structure

// domain/Order.js - Domain Model
class Order {
  constructor(userId, items) {
    this.userId = userId;
    this.items = items;
    this.status = 'pending';
    this.createdAt = new Date();
  }
  
  calculateTotal() {
    const subtotal = this.items.reduce(
      (sum, item) => sum + item.price * item.quantity, 0
    );
    return subtotal > 100000 ? subtotal * 0.9 : subtotal;
  }
  
  confirm() {
    this.status = 'confirmed';
  }
}

// repositories/OrderRepository.js
class OrderRepository {
  async save(order) {
    const result = await db.query(
      'INSERT INTO orders (user_id, status, total) VALUES ($1, $2, $3) RETURNING *',
      [order.userId, order.status, order.calculateTotal()]
    );
    return result.rows[0];
  }
  
  async findById(id) {
    const result = await db.query('SELECT * FROM orders WHERE id = $1', [id]);
    return result.rows[0];
  }
}

// services/OrderService.js
class OrderService {
  constructor(orderRepo, userRepo, emailService, paymentService) {
    this.orderRepo = orderRepo;
    this.userRepo = userRepo;
    this.emailService = emailService;
    this.paymentService = paymentService;
  }
  
  async createOrder(userId, items) {
    const user = await this.userRepo.findById(userId);
    if (!user) throw new Error('User not found');
    
    const order = new Order(userId, items);
    const total = order.calculateTotal();
    
    // Process payment
    await this.paymentService.charge(user.cardToken, total);
    
    // Save order
    order.confirm();
    const savedOrder = await this.orderRepo.save(order);
    
    // Send notification (async - ไม่รอผล)
    this.emailService.sendOrderConfirmation(user.email, savedOrder)
      .catch(err => console.error('Email failed:', err));
    
    return savedOrder;
  }
}

// controllers/OrderController.js
class OrderController {
  constructor(orderService) {
    this.orderService = orderService;
  }
  
  async createOrder(req, res) {
    try {
      const { userId, items } = req.body;
      const order = await this.orderService.createOrder(userId, items);
      res.status(201).json({ success: true, data: order });
    } catch (error) {
      res.status(400).json({ success: false, error: error.message });
    }
  }
}
```

---

## 9. สรุป Comparison Table

| Feature | Monolith | Modular Monolith | Microservices |
|---------|---------|-----------------|---------------|
| **Initial Setup** | ✅ Easy | ✅ Easy | ❌ Complex |
| **Development Speed (Early)** | ✅ Fast | ✅ Fast | ❌ Slow |
| **Development Speed (Scale)** | ❌ Slow | ⚖️ Medium | ✅ Fast |
| **Deployment** | ✅ Simple | ✅ Simple | ❌ Complex |
| **Testing** | ✅ Simple | ✅ Simple | ❌ Complex |
| **Debugging** | ✅ Easy | ✅ Easy | ❌ Hard |
| **Scaling** | ❌ All or nothing | ❌ All or nothing | ✅ Independent |
| **Technology Choice** | ❌ Limited | ❌ Limited | ✅ Polyglot |
| **Fault Isolation** | ❌ None | ❌ Limited | ✅ Good |
| **Team Independence** | ❌ Conflicts | ⚖️ Medium | ✅ High |
| **Operational Complexity** | ✅ Low | ✅ Low | ❌ High |
| **Recommended Team Size** | < 10 | 10-50 | 50+ |

---

## 10. คำถามทบทวนและแบบฝึกหัด

### คำถามทบทวน
1. อธิบาย "Distributed Monolith" และทำไมมันถึงแย่กว่า Regular Monolith
2. Modular Monolith ต่างจาก Microservices อย่างไร?
3. Conway's Law คืออะไร และเกี่ยวข้องกับ Microservices อย่างไร?
4. ทำไม "Database per Service" ถึงสำคัญใน Microservices?
5. Strangler Fig Pattern ช่วยในการ Migrate จาก Monolith ได้อย่างไร?

### แบบฝึกหัด

**Exercise 1:** วิเคราะห์ Codebase
ดู Monolith code ที่ให้มาใน Workshop และระบุ:
- ส่วนไหนสามารถ extract เป็น Microservice ได้ง่ายที่สุด?
- ส่วนไหนที่ยากที่สุดในการ extract?
- ทำไม?

**Exercise 2:** Design Microservices Boundaries
สำหรับ Hospital Management System:
- Patients (ข้อมูลผู้ป่วย)
- Doctors (แพทย์)
- Appointments (นัดหมาย)
- Medical Records (ประวัติการรักษา)
- Billing (ใบเสร็จ)
- Pharmacy (ยา)

วาด Diagram แสดงว่าจะแบ่ง Microservices อย่างไร?
Service แต่ละอันจะสื่อสารกันอย่างไร?

**Exercise 3:** Implement Modular Monolith
จาก Code ใน Workshop 8.1 ที่มีปัญหา:
1. แก้ไข SQL Injection vulnerability
2. Refactor ให้ใช้ Repository Pattern
3. แยก Business Logic ออกจาก Controller
4. เพิ่ม Unit Tests

---

**ต่อไป:** [Part 03 - Core Principles & The 12-Factor App →](part-03.md)

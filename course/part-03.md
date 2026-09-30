# Part 03: Core Principles & The 12-Factor App
## หลักการสำคัญของ Microservices และ 12-Factor App Methodology

> **ระดับ:** ⭐ เริ่มต้น | **เวลาเรียน:** 4 ชั่วโมง | **Prerequisites:** Parts 01-02

---

## 🎯 สิ่งที่จะได้เรียนรู้

- หลักการ 8 ข้อของ Microservices
- The 12-Factor App Methodology (ครบทั้ง 12 ข้อ)
- Domain-Driven Design Basics
- Bounded Context คืออะไร
- Workshop: Apply 12-Factor App ในโปรเจคจริง

---

## 1. หลักการ 8 ข้อของ Microservices

### Principle 1: Single Responsibility

```
❌ Bad: Order Service ทำหน้าที่หลายอย่าง
- จัดการ Orders
- ส่ง Email notifications
- จัดการ Payment
- อัปเดต Inventory

✅ Good: แต่ละ Service ทำหน้าที่เดียว
- Order Service: จัดการ Orders เท่านั้น
- Notification Service: ส่ง Email/SMS
- Payment Service: จัดการ Payment
- Inventory Service: จัดการ Stock
```

**ทำไมต้อง Single Responsibility?**
```
ถ้า Email ใช้ไม่ได้ ไม่ควรกระทบ Order creation
ถ้า Inventory update ช้า ไม่ควร block payment
แต่ละ Service ควร fail independently
```

### Principle 2: Loose Coupling

Coupling ต้องต่ำ - Services ไม่ควรรู้รายละเอียดภายในของกันและกัน

```javascript
// ❌ Tight Coupling - Order Service รู้โครงสร้าง DB ของ User Service
class OrderService {
  async createOrder(userId, items) {
    // Direct Database Access ของ Service อื่น - ห้ามทำ!
    const user = await userDb.query('SELECT * FROM users WHERE id = $1', [userId]);
    // ...
  }
}

// ❌ Tight Coupling - Import code โดยตรงจาก Service อื่น
const { validateUser } = require('../user-service/validators');

// ✅ Loose Coupling - สื่อสารผ่าน API เท่านั้น
class OrderService {
  constructor(userServiceClient) {
    this.userClient = userServiceClient;
  }
  
  async createOrder(userId, items) {
    // สื่อสารผ่าน API/Contract เท่านั้น
    const user = await this.userClient.getUser(userId);
    // ...
  }
}
```

**Types of Coupling (จาก Low ไป High):**
```
1. Data Coupling      - แลกเปลี่ยนข้อมูลอย่างง่าย      ✅ ดี
2. Message Coupling   - สื่อสารผ่าน Message              ✅ ดี  
3. External Coupling  - Share external format            ⚠️ ระวัง
4. Control Coupling   - ส่ง control flag                 ❌ หลีกเลี่ยง
5. Content Coupling   - Access internal data/code        ❌ ห้ามทำ
```

### Principle 3: High Cohesion

โค้ดที่เกี่ยวข้องกันควรอยู่ใกล้กัน

```
✅ High Cohesion - User Service
/users/
  ├── create_account()      # เกี่ยวกับ User ทั้งหมด
  ├── update_profile()      
  ├── change_password()     
  ├── delete_account()      
  └── get_user_by_email()   

❌ Low Cohesion - Mixed Service
/mixed-service/
  ├── create_user()         # User stuff
  ├── send_email()          # Email stuff (ไม่เกี่ยว)
  ├── process_payment()     # Payment stuff (ไม่เกี่ยว)
  └── update_inventory()    # Inventory stuff (ไม่เกี่ยว)
```

### Principle 4: Service Autonomy

แต่ละ Service ต้องสามารถ:
- **Deploy** โดยไม่ขึ้นกับ Service อื่น
- **Scale** อิสระ
- **Fail** โดยไม่ crash Service อื่น
- **Evolve** ไม่ทำ breaking changes กับ Service อื่น

```yaml
# Autonomous Deployment
# Deploy payment-service โดยไม่ต้อง redeploy ทั้งระบบ
kubectl rollout restart deployment/payment-service

# Autonomous Scaling
kubectl scale deployment/payment-service --replicas=10
kubectl scale deployment/user-service --replicas=2

# Services ยังทำงานได้เมื่อ payment-service down
# (ด้วย Circuit Breaker Pattern)
```

### Principle 5: Decentralized Data Management

```
✅ ถูกต้อง: Database per Service
┌─────────────────────────────────────────────────────────┐
│ User Service │ Order Service │ Payment Service          │
│  PostgreSQL  │   MongoDB     │    MySQL                  │
│  (users,     │  (orders,     │  (transactions,           │
│   profiles)  │   items)      │   payments)               │
└─────────────────────────────────────────────────────────┘

❌ ผิด: Shared Database
┌──────────────────────────────────────────────────────────┐
│ User Service │ Order Service │ Payment Service           │
└──────────────┴───────────────┴──────────────────────────┘
                          │
                ┌─────────▼──────────┐
                │   Shared Database  │  ← ไม่ควรทำ
                └────────────────────┘
```

**ทำไมต้อง Database per Service?**
```
1. Freedom to choose technology
   - User Service: PostgreSQL (ACID transactions)
   - Product Service: Elasticsearch (full-text search)
   - Session Service: Redis (fast in-memory)

2. Independent scaling
   - Scale database ตาม service load

3. No cross-service DB coupling
   - Schema changes ใน User Service ไม่กระทบ Order Service

4. Fault isolation
   - User DB down ≠ Order DB down
```

### Principle 6: Failure as a Feature

ระบบ Distributed มีโอกาสล้มเหลวสูง ต้องออกแบบให้รองรับความล้มเหลว

```javascript
// Resilient Service Client
class UserServiceClient {
  constructor() {
    this.baseUrl = process.env.USER_SERVICE_URL;
    this.timeout = 5000; // 5 seconds
    this.retries = 3;
  }
  
  async getUser(userId) {
    for (let attempt = 1; attempt <= this.retries; attempt++) {
      try {
        const response = await axios.get(
          `${this.baseUrl}/users/${userId}`,
          { timeout: this.timeout }
        );
        return response.data;
        
      } catch (error) {
        const isLastAttempt = attempt === this.retries;
        
        if (isLastAttempt) {
          // Return fallback หรือ throw error
          return null; // graceful degradation
        }
        
        // Exponential backoff
        await new Promise(resolve => 
          setTimeout(resolve, Math.pow(2, attempt) * 100)
        );
      }
    }
  }
}
```

### Principle 7: Design for Change

```javascript
// ✅ API Versioning - รองรับการเปลี่ยนแปลง
// v1 - เวอร์ชันเดิม
app.get('/v1/users/:id', (req, res) => {
  res.json({ id, name, email });  // โครงสร้างเดิม
});

// v2 - เวอร์ชันใหม่ (ไม่กระทบ v1)
app.get('/v2/users/:id', (req, res) => {
  res.json({ 
    id, 
    name, 
    email,
    profile: { avatar, bio }  // เพิ่ม field ใหม่
  });
});
```

### Principle 8: Observability

```
ระบบ Microservices ต้องมี:

1. Logging         - บันทึกทุก event/error
2. Metrics         - วัดประสิทธิภาพ
3. Distributed Tracing - ติดตาม request ข้าม services
4. Health Checks   - ตรวจสอบสุขภาพของ service
5. Alerting        - แจ้งเตือนเมื่อมีปัญหา
```

---

## 2. The 12-Factor App Methodology

The 12-Factor App เป็น methodology ที่สร้างโดย Heroku สำหรับการสร้าง Software as a Service (SaaS) ที่ดี เหมาะมากสำหรับ Microservices

### Factor 1: Codebase (One codebase, many deploys)

```
✅ หลักการ: 1 Codebase = 1 Service
ใช้ Git Repository หนึ่งต่อ Service

git/
├── user-service/        (github.com/company/user-service)
├── order-service/       (github.com/company/order-service)
└── payment-service/     (github.com/company/payment-service)

❌ ผิด: หลาย Services ใน Repo เดียว (Monorepo ต้องจัดการต่างออกไป)
หรือ Service เดียวใน หลาย Repos
```

```bash
# การ Deploy ใน Environment ต่างกัน จาก Codebase เดียว
git clone https://github.com/company/payment-service

# Deploy to dev
ENV=development docker-compose up

# Deploy to staging
ENV=staging docker-compose -f docker-compose.staging.yml up

# Deploy to production
ENV=production kubectl apply -f k8s/
```

### Factor 2: Dependencies (Explicitly declare and isolate)

```json
// ✅ ดี: Explicit dependencies ใน package.json
{
  "name": "user-service",
  "dependencies": {
    "express": "^4.18.2",      // ระบุ version ชัดเจน
    "pg": "^8.11.0",
    "bcrypt": "^5.1.0"
  }
}
```

```dockerfile
# ✅ Dockerfile ที่ดี - isolate dependencies ใน container
FROM node:18-alpine

WORKDIR /app

# Copy dependency files first (cache optimization)
COPY package*.json ./
RUN npm ci --only=production  # ใช้ ci ไม่ใช่ install (reproducible)

COPY . .

CMD ["node", "src/index.js"]
```

```bash
# ❌ ผิด: Assume global tools มีอยู่
# ถ้า imagemagick ไม่ได้ติดตั้ง = fail

# ✅ ถูก: declare ใน Dockerfile
RUN apk add --no-cache imagemagick
```

### Factor 3: Config (Store config in environment)

```javascript
// ❌ ผิด: Hard-code configuration ใน code
const db = new Pool({
  host: 'production-db.company.com',  // Hard-coded!
  password: 'super-secret-password',   // Sensitive!
  port: 5432
});

// ✅ ถูก: ใช้ Environment Variables
const db = new Pool({
  host: process.env.DB_HOST,
  password: process.env.DB_PASSWORD,
  port: parseInt(process.env.DB_PORT) || 5432,
  database: process.env.DB_NAME,
  user: process.env.DB_USER
});
```

```bash
# ✅ ตัวอย่าง .env file (ไม่ commit ขึ้น git!)
# .env.example (commit ขึ้น git - template)
DB_HOST=localhost
DB_PORT=5432
DB_NAME=userdb
DB_USER=postgres
DB_PASSWORD=  # กรอกเอง
JWT_SECRET=   # กรอกเอง
```

```yaml
# kubernetes secret
apiVersion: v1
kind: Secret
metadata:
  name: user-service-secrets
type: Opaque
data:
  db-password: <base64-encoded-password>
  jwt-secret: <base64-encoded-secret>
---
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: user-service
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: user-service-secrets
              key: db-password
```

### Factor 4: Backing Services (Treat as attached resources)

```javascript
// ✅ Backing Services เป็น Attached Resources
// เปลี่ยนได้โดยแค่เปลี่ยน URL ใน config

const config = {
  // Database - attached resource
  database: process.env.DATABASE_URL || 'postgresql://localhost/mydb',
  
  // Cache - attached resource
  redis: process.env.REDIS_URL || 'redis://localhost:6379',
  
  // Message Queue - attached resource
  messageQueue: process.env.AMQP_URL || 'amqp://localhost:5672',
  
  // Email Service - attached resource  
  smtp: process.env.SMTP_URL || 'smtp://localhost:1025',
  
  // External API - attached resource
  paymentGateway: process.env.PAYMENT_API_URL,
};

// ✅ สามารถ swap local DB เป็น Production DB ได้ง่าย
// DATABASE_URL=postgresql://prod-server/mydb npm start
```

### Factor 5: Build, Release, Run (Separate stages)

```
BUILD Stage
├── Compile code
├── Bundle assets
├── Run tests
└── Create Docker image

RELEASE Stage
├── Combine Build artifact + Config
├── Tag with version
└── Store in registry

RUN Stage
├── Pull from registry
├── Start process(es)
└── Health check
```

```dockerfile
# Multi-stage Dockerfile (Build stage แยกจาก Run stage)

# Stage 1: Build
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build    # Transpile TypeScript, etc.
RUN npm test         # Run tests ใน build stage

# Stage 2: Production (Run stage)
FROM node:18-alpine AS production
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/package*.json ./
RUN npm ci --only=production

ENV NODE_ENV=production
EXPOSE 3000
USER node  # ไม่ run เป็น root

CMD ["node", "dist/index.js"]
```

```bash
# Build
docker build -t user-service:1.2.3 .

# Release (push to registry)
docker tag user-service:1.2.3 registry.company.com/user-service:1.2.3
docker push registry.company.com/user-service:1.2.3

# Run (pull from registry)
docker pull registry.company.com/user-service:1.2.3
docker run -e DB_HOST=prod-db registry.company.com/user-service:1.2.3
```

### Factor 6: Processes (Stateless)

```javascript
// ❌ ผิด: Storing state in process memory
let userSessions = {};  // In-memory sessions

app.post('/login', (req, res) => {
  const token = generateToken();
  userSessions[token] = { userId: req.user.id };  // Store in memory
  res.json({ token });
});

// ปัญหา: ถ้า restart หรือ scale up instance ใหม่ 
// sessions หาย! User ต้อง login ใหม่

// ✅ ถูก: Stateless - store state ใน backing service
const redis = new Redis(process.env.REDIS_URL);

app.post('/login', async (req, res) => {
  const token = generateToken();
  await redis.set(
    `session:${token}`, 
    JSON.stringify({ userId: req.user.id }),
    'EX', 
    3600  // expire in 1 hour
  );
  res.json({ token });
});

// ✅ ทุก instance สามารถ serve request ได้ถูกต้อง
// เพราะ session อยู่ใน Redis (shared)
```

```yaml
# ✅ Stateless = scale ง่าย
# ไม่มี state ใน process = scale ได้อิสระ
kubectl scale deployment/user-service --replicas=10
# ทุก 10 instances ทำงานได้ถูกต้อง
```

### Factor 7: Port Binding (Export services via port binding)

```javascript
// ✅ Service binds port ตาม Environment Variable
const PORT = parseInt(process.env.PORT) || 3000;
const HOST = process.env.HOST || '0.0.0.0';

app.listen(PORT, HOST, () => {
  console.log(`Server running on ${HOST}:${PORT}`);
});
```

```yaml
# docker-compose.yml
services:
  user-service:
    image: user-service:latest
    environment:
      - PORT=3001  # Set port via env
    ports:
      - "3001:3001"  # Map container port to host
  
  order-service:
    image: order-service:latest
    environment:
      - PORT=3002
    ports:
      - "3002:3002"
```

### Factor 8: Concurrency (Scale out via process model)

```javascript
// ✅ ออกแบบให้ scale ออกด้วยหลาย processes
// ไม่ใช่ scale up ด้วย threads

// Cluster mode - Node.js
const cluster = require('cluster');
const numCPUs = require('os').cpus().length;

if (cluster.isMaster) {
  // Fork workers
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }
  
  cluster.on('exit', (worker, code) => {
    console.log(`Worker ${worker.process.pid} died, restarting...`);
    cluster.fork();  // Restart dead worker
  });
  
} else {
  // Worker process
  const app = require('./app');
  app.listen(process.env.PORT || 3000);
}
```

```yaml
# ✅ Scale process type independently ใน Kubernetes
# Web processes
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-workers
spec:
  replicas: 5    # Scale web workers

---
# Background job processes
apiVersion: apps/v1  
kind: Deployment
metadata:
  name: job-workers
spec:
  replicas: 3    # Scale job workers separately
```

### Factor 9: Disposability (Fast startup, graceful shutdown)

```javascript
// ✅ Graceful Shutdown
const server = app.listen(PORT, () => {
  console.log(`Server started on port ${PORT}`);
});

// Handle termination signals
process.on('SIGTERM', gracefulShutdown);
process.on('SIGINT', gracefulShutdown);

async function gracefulShutdown(signal) {
  console.log(`Received ${signal}, starting graceful shutdown...`);
  
  // หยุดรับ request ใหม่
  server.close(async () => {
    try {
      // Close database connections
      await db.end();
      
      // Close message queue connections
      await mqConnection.close();
      
      // Finish in-flight requests (รอจนเสร็จ)
      console.log('Graceful shutdown complete');
      process.exit(0);
      
    } catch (error) {
      console.error('Error during shutdown:', error);
      process.exit(1);
    }
  });
  
  // Force shutdown หลัง 30 วินาที
  setTimeout(() => {
    console.error('Forced shutdown after timeout');
    process.exit(1);
  }, 30000);
}
```

```dockerfile
# ✅ Fast startup - ใช้ Alpine Linux (ขนาดเล็ก)
FROM node:18-alpine

# Pre-build dependencies สำหรับ faster startup
RUN npm ci --only=production && \
    node -e "require('./src/index')" || true  # warm up module cache
```

### Factor 10: Dev/Prod Parity (Keep environments similar)

```yaml
# ❌ ผิด: Development และ Production ต่างกันมาก
# Development: SQLite
# Production: PostgreSQL
# = Bugs ที่ไม่เจอใน dev แต่เจอใน prod!

# ✅ ถูก: ใช้ Docker Compose ให้ Dev environment เหมือน Prod
version: '3.8'
services:
  app:
    build: .
    environment:
      - NODE_ENV=development
      - DB_HOST=postgres
    volumes:
      - ./src:/app/src  # Hot reload
    
  postgres:
    image: postgres:15  # Same version as production!
    environment:
      - POSTGRES_DB=myapp
      - POSTGRES_PASSWORD=localpassword
    
  redis:
    image: redis:7-alpine  # Same version as production!
```

```bash
# ✅ Use same Docker image in all environments
# Dev: docker run -e ENV=dev myapp:1.2.3
# Staging: docker run -e ENV=staging myapp:1.2.3
# Prod: docker run -e ENV=prod myapp:1.2.3
# Same image, different config via env vars
```

### Factor 11: Logs (Treat as event streams)

```javascript
// ❌ ผิด: เขียน log ลง file
const winston = require('winston');
const logger = winston.createLogger({
  transports: [
    new winston.transports.File({ filename: '/var/log/app.log' })
  ]
});

// ✅ ถูก: Write logs to stdout (และให้ platform จัดการ)
const logger = winston.createLogger({
  transports: [
    new winston.transports.Console({
      format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.json()  // Structured logging
      )
    })
  ]
});

// Structured log format
logger.info('Order created', {
  orderId: '12345',
  userId: '67890',
  amount: 1500,
  timestamp: new Date().toISOString(),
  service: 'order-service',
  environment: process.env.NODE_ENV
});
```

```json
// ✅ Structured Log Output (JSON)
{
  "level": "info",
  "message": "Order created",
  "orderId": "12345",
  "userId": "67890", 
  "amount": 1500,
  "timestamp": "2024-01-15T10:30:00.000Z",
  "service": "order-service",
  "environment": "production"
}
```

```bash
# Platform (Kubernetes/Docker) จัดการ log streams
# Log aggregation ด้วย Fluentd/Logstash
docker logs user-service 2>&1 | fluentd -c fluent.conf
```

### Factor 12: Admin Processes (Run as one-off processes)

```javascript
// ✅ Admin tasks เป็น one-off processes
// database/migrate.js
const { Pool } = require('pg');
const db = new Pool({ connectionString: process.env.DATABASE_URL });

async function runMigrations() {
  console.log('Running database migrations...');
  
  await db.query(`
    CREATE TABLE IF NOT EXISTS users (
      id SERIAL PRIMARY KEY,
      email VARCHAR(255) UNIQUE NOT NULL,
      name VARCHAR(255),
      created_at TIMESTAMP DEFAULT NOW()
    )
  `);
  
  await db.query(`
    ALTER TABLE users ADD COLUMN IF NOT EXISTS phone VARCHAR(20)
  `);
  
  console.log('Migrations complete');
  await db.end();
}

runMigrations()
  .then(() => process.exit(0))
  .catch(err => {
    console.error(err);
    process.exit(1);
  });
```

```yaml
# Kubernetes Job สำหรับ one-off admin tasks
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
spec:
  template:
    spec:
      containers:
      - name: migration
        image: user-service:1.2.3
        command: ["node", "database/migrate.js"]
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: url
      restartPolicy: Never
  backoffLimit: 3
```

---

## 3. Domain-Driven Design (DDD) Basics

DDD เป็น approach ในการออกแบบ Software ที่ถูก Develop โดย Eric Evans เหมาะมากสำหรับการแบ่ง Microservices

### 3.1 Core Concepts

**Domain:** ความรู้/กิจกรรมของ Business ที่เราสร้าง Software ให้
```
E-Commerce Domain:
- ขายสินค้า
- จัดการ inventory
- ประมวลผล payment
- จัดส่งสินค้า
- ดูแลลูกค้า
```

**Ubiquitous Language:** ภาษาร่วมกันระหว่าง Developer และ Business
```
❌ Developer พูดว่า: "User Entity ที่มี FK ไปยัง Order Table"
✅ Developer พูดว่า: "ลูกค้าสั่งซื้อสินค้า" (เหมือนที่ Business พูด)
```

**Bounded Context:** ขอบเขตที่ Model มีความหมายเฉพาะ

```
┌─────────────────────────────────────────────────────────────┐
│                    E-Commerce System                         │
│                                                             │
│  ┌─────────────────────┐   ┌─────────────────────────────┐ │
│  │   Sales Context     │   │    Inventory Context        │ │
│  │                     │   │                             │ │
│  │  Customer: buyer    │   │  Product: SKU, stock        │ │
│  │  Order: purchase    │   │  Warehouse: location        │ │
│  │  Product: item      │   │  Product: physical item     │ │
│  └─────────────────────┘   └─────────────────────────────┘ │
│                                                             │
│  ┌─────────────────────┐   ┌─────────────────────────────┐ │
│  │  Payment Context    │   │    Shipping Context         │ │
│  │                     │   │                             │ │
│  │  Customer: payer    │   │  Customer: recipient        │ │
│  │  Order: invoice     │   │  Order: shipment            │ │
│  │  Account: source    │   │  Product: package           │ │
│  └─────────────────────┘   └─────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘

"Product" มีความหมายต่างกันในแต่ละ Context!
Sales: Product คือสิ่งที่ขาย (ราคา, รายละเอียด)
Inventory: Product คือของจริงที่มีในคลัง (ปริมาณ, ที่เก็บ)
Shipping: Product คือพัสดุที่ต้องส่ง (น้ำหนัก, ขนาด)
```

### 3.2 Identifying Bounded Contexts

```
คำถามที่ช่วย identify:
1. "ใคร owns ข้อมูลนี้?" 
   → ถ้า 2 teams ต้องการข้อมูลเดียวกัน = อาจแยก context ผิด

2. "เมื่อ Business พูดถึง X ในบริบทนี้ มันหมายถึงอะไร?"
   → "Product" ในบริบท Sales ≠ "Product" ในบริบท Inventory

3. "ถ้าเราเปลี่ยน Context นี้ Context อื่นจะกระทบไหม?"
   → ถ้ากระทบมาก = coupling สูงเกินไป

4. "ข้อมูลอะไรที่ Change Together?"
   → ข้อมูลที่เปลี่ยนพร้อมกันบ่อยควรอยู่ใน Context เดียวกัน
```

### 3.3 Context Mapping

```
Bounded Contexts สื่อสารกันผ่าน:

1. Partnership (ร่วมมือกัน)
   Team A ──────── Team B
   ├── Shared Planning
   └── Coordinate changes together

2. Shared Kernel (ใช้ Code ร่วมกัน)
   Team A ──▶ [Shared Library] ◀── Team B
   ├── Risky: Changes affect both
   └── Use sparingly

3. Customer-Supplier (ผู้ขาย-ผู้ซื้อ)
   Supplier Team ──▶ Customer Team
   ├── Supplier defines API
   └── Customer depends on Supplier

4. Anti-corruption Layer (ป้องกันการปนเปื้อน)
   Team A ──▶ [ACL] ──▶ Legacy System
   ├── ACL translates between models
   └── Protect your model from external influence

5. Published Language (ภาษาที่ตกลงกัน)
   Service A ──▶ [Event Schema] ◀── Service B
   ├── Well-documented event/API format
   └── Kafka topic schemas, OpenAPI specs
```

---

## 4. Workshop: Apply 12-Factor App

### 4.1 Project Setup

```bash
mkdir twelve-factor-demo
cd twelve-factor-demo
git init

# โครงสร้าง
mkdir -p src/{routes,models,repositories,services}
mkdir -p database/{migrations,seeds}
```

### 4.2 การ Apply Factor 3 (Config) อย่างถูกต้อง

```javascript
// config/index.js
const config = {
  app: {
    port: parseInt(process.env.PORT, 10) || 3000,
    host: process.env.HOST || '0.0.0.0',
    env: process.env.NODE_ENV || 'development',
    name: process.env.APP_NAME || 'user-service',
  },
  
  database: {
    host: process.env.DB_HOST || 'localhost',
    port: parseInt(process.env.DB_PORT, 10) || 5432,
    name: process.env.DB_NAME || 'userdb',
    user: process.env.DB_USER || 'postgres',
    password: process.env.DB_PASSWORD,
    poolMin: parseInt(process.env.DB_POOL_MIN, 10) || 2,
    poolMax: parseInt(process.env.DB_POOL_MAX, 10) || 10,
  },
  
  redis: {
    url: process.env.REDIS_URL || 'redis://localhost:6379',
  },
  
  jwt: {
    secret: process.env.JWT_SECRET,
    expiresIn: process.env.JWT_EXPIRES_IN || '7d',
  },
  
  logging: {
    level: process.env.LOG_LEVEL || 'info',
    format: process.env.LOG_FORMAT || 'json',
  }
};

// Validate required config
const required = ['DB_PASSWORD', 'JWT_SECRET'];
required.forEach(key => {
  if (!process.env[key]) {
    throw new Error(`Required environment variable ${key} is not set`);
  }
});

module.exports = config;
```

### 4.3 Structured Logging (Factor 11)

```javascript
// utils/logger.js
const winston = require('winston');
const config = require('../config');

const logger = winston.createLogger({
  level: config.logging.level,
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    config.logging.format === 'json' 
      ? winston.format.json()
      : winston.format.simple()
  ),
  defaultMeta: {
    service: config.app.name,
    environment: config.app.env,
    version: process.env.APP_VERSION || '1.0.0',
  },
  transports: [
    new winston.transports.Console()  // Factor 11: stdout only
  ]
});

// Request logging middleware
logger.requestMiddleware = (req, res, next) => {
  const start = Date.now();
  
  res.on('finish', () => {
    const duration = Date.now() - start;
    
    logger.info('HTTP Request', {
      method: req.method,
      url: req.url,
      statusCode: res.statusCode,
      duration,
      ip: req.ip,
      userAgent: req.get('User-Agent'),
      requestId: req.headers['x-request-id'],
    });
  });
  
  next();
};

module.exports = logger;
```

### 4.4 Health Check (Factor 8 & 9)

```javascript
// routes/health.js
const router = require('express').Router();
const db = require('../database');
const redis = require('../cache');
const logger = require('../utils/logger');

// Simple health check
router.get('/health', (req, res) => {
  res.json({
    status: 'healthy',
    uptime: process.uptime(),
    timestamp: new Date().toISOString(),
  });
});

// Detailed health check
router.get('/health/detailed', async (req, res) => {
  const checks = {};
  
  // Check database
  try {
    await db.query('SELECT 1');
    checks.database = { status: 'healthy' };
  } catch (error) {
    checks.database = { status: 'unhealthy', error: error.message };
  }
  
  // Check Redis
  try {
    await redis.ping();
    checks.redis = { status: 'healthy' };
  } catch (error) {
    checks.redis = { status: 'unhealthy', error: error.message };
  }
  
  const isHealthy = Object.values(checks).every(c => c.status === 'healthy');
  
  res.status(isHealthy ? 200 : 503).json({
    status: isHealthy ? 'healthy' : 'degraded',
    checks,
    uptime: process.uptime(),
    version: process.env.APP_VERSION,
    timestamp: new Date().toISOString(),
  });
});

// Readiness check (used by Kubernetes)
router.get('/ready', async (req, res) => {
  try {
    await db.query('SELECT 1');
    res.json({ status: 'ready' });
  } catch {
    res.status(503).json({ status: 'not ready' });
  }
});

// Liveness check (used by Kubernetes)  
router.get('/live', (req, res) => {
  // ถ้า process ยังทำงานอยู่ = alive
  res.json({ status: 'alive', uptime: process.uptime() });
});

module.exports = router;
```

### 4.5 Graceful Shutdown (Factor 9)

```javascript
// src/server.js
const app = require('./app');
const db = require('./database');
const logger = require('./utils/logger');
const config = require('./config');

const server = app.listen(config.app.port, config.app.host, () => {
  logger.info('Server started', {
    port: config.app.port,
    host: config.app.host,
    env: config.app.env,
  });
});

// Track active connections
let connections = new Set();

server.on('connection', conn => {
  connections.add(conn);
  conn.on('close', () => connections.delete(conn));
});

// Graceful shutdown handler
async function gracefulShutdown(signal) {
  logger.info(`Received ${signal}, starting graceful shutdown`);
  
  // Stop accepting new connections
  server.close(async () => {
    logger.info('HTTP server closed');
    
    try {
      // Close all existing connections
      connections.forEach(conn => conn.destroy());
      
      // Close database pool
      await db.end();
      logger.info('Database connections closed');
      
      logger.info('Graceful shutdown complete');
      process.exit(0);
      
    } catch (error) {
      logger.error('Error during shutdown', { error: error.message });
      process.exit(1);
    }
  });
  
  // Force shutdown after 30 seconds
  setTimeout(() => {
    logger.error('Forced shutdown - graceful shutdown took too long');
    process.exit(1);
  }, 30_000);
}

process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));
process.on('SIGINT', () => gracefulShutdown('SIGINT'));

// Handle unhandled errors
process.on('unhandledRejection', (reason, promise) => {
  logger.error('Unhandled Rejection', { reason, promise });
});

process.on('uncaughtException', (error) => {
  logger.error('Uncaught Exception', { error: error.message, stack: error.stack });
  process.exit(1);  // Must exit on uncaught exception
});

module.exports = server;
```

---

## 5. 12-Factor App Summary Card

```
┌─────────────────────────────────────────────────────────────────┐
│                   THE 12-FACTOR APP                              │
├────┬──────────────────────────────────────────────────────────── │
│ 1  │ CODEBASE      One repo per service, multiple deploys        │
│ 2  │ DEPENDENCIES  Explicit declaration in package.json          │
│ 3  │ CONFIG        Environment variables, never hardcode         │
│ 4  │ BACKING SVC   Database, cache = attached resources          │
│ 5  │ BUILD/RUN     Separate build, release, run stages           │
│ 6  │ PROCESSES     Stateless processes, state in backing svc     │
│ 7  │ PORT BINDING  Service listens on PORT env var               │
│ 8  │ CONCURRENCY   Scale out with processes, not threads         │
│ 9  │ DISPOSABILITY Fast startup, graceful shutdown               │
│ 10 │ DEV/PROD      Use same backing services in all envs         │
│ 11 │ LOGS          Write to stdout, let platform aggregate       │
│ 12 │ ADMIN         DB migrations = one-off processes             │
└────┴──────────────────────────────────────────────────────────── │
```

---

## 6. สรุปและคำถามทบทวน

### Key Takeaways
1. **8 หลักการ Microservices:** Single Responsibility, Loose Coupling, High Cohesion, Autonomy, Decentralized Data, Failure Design, Design for Change, Observability
2. **12-Factor App:** Methodology ที่ทำให้ Microservices portable, scalable, maintainable
3. **DDD:** ช่วยออกแบบ Service boundaries ตาม Business domain
4. **Bounded Context:** แต่ละ Service มี "ภาษา" ของตัวเอง

### คำถามทบทวน
1. อธิบายความแตกต่างระหว่าง Loose Coupling และ High Cohesion
2. Factor 3 (Config) บอกว่าอะไร? ทำไมถึงสำคัญ?
3. Bounded Context คืออะไร? ยกตัวอย่าง
4. Factor 6 (Processes) คืออะไร? ทำไม Stateless จึงสำคัญ?
5. DDD และ Microservices เกี่ยวข้องกันอย่างไร?

### แบบฝึกหัด

**Exercise 1:** 12-Factor Audit
ตรวจสอบ Project ที่มีอยู่ว่า comply กับ 12-Factor App หรือไม่ ในแต่ละ Factor ให้ระบุว่า:
- ✅ ทำตาม
- ⚠️ ทำบ้าง
- ❌ ยังไม่ได้ทำ

**Exercise 2:** Config Refactor
แก้ไข Code ต่อไปนี้ให้ Follow Factor 3:
```javascript
// แก้ไข code นี้ให้ถูกต้องตาม 12-Factor
const db = new Pool({
  host: 'prod-db.company.internal',
  user: 'admin',
  password: 'P@ssw0rd123',
  database: 'ecommerce',
  port: 5432
});
```

**Exercise 3:** Identify Bounded Contexts
สำหรับ Hospital Management System แบ่ง Bounded Contexts:
- Patients, Doctors, Appointments, Medical Records, Billing, Pharmacy, Lab Results

---

**ต่อไป:** [Part 04 - Setting Up Development Environment →](part-04.md)

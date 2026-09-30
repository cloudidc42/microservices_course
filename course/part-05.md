# Part 05: Docker Fundamentals สำหรับ Microservices
## เรียนรู้ Docker ตั้งแต่พื้นฐานถึงระดับ Production

> **ระดับ:** ⭐⭐ พื้นฐาน | **เวลาเรียน:** 5-6 ชั่วโมง | **Prerequisites:** Parts 01-04

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Docker Architecture และแนวคิดหลัก
- Dockerfile Best Practices สำหรับ Microservices
- Docker Commands ที่ใช้บ่อย
- Image Optimization เทคนิค
- Container Networking
- Docker Security Best Practices
- Workshop: Dockerize Microservices

---

## 1. Docker Architecture

### 1.1 ทำไมต้อง Docker?

**ปัญหาก่อน Docker:**
```
Developer: "มันทำงานบนเครื่องผมนะ"
DevOps:    "แต่บน Production ไม่ทำงาน"

สาเหตุ:
- Different OS versions
- Different library versions
- Different environment variables
- Different file paths
```

**Docker แก้ปัญหา:**
```
"Build Once, Run Anywhere"

Container = Application + Dependencies + Runtime + Config
ทุกอย่างอยู่ใน Container → ทำงานเหมือนกันทุก Environment
```

### 1.2 VM vs Container

```
Virtual Machine:
┌──────────────────────────────────┐
│  App A    │  App B    │  App C   │
├──────────────────────────────────┤
│  Guest OS │  Guest OS │  Guest OS│
├──────────────────────────────────┤
│              Hypervisor          │
├──────────────────────────────────┤
│          Host Operating System   │
├──────────────────────────────────┤
│              Hardware            │
└──────────────────────────────────┘
ขนาด: GB | Startup: นาที | Overhead: สูง

Container:
┌────────────────────────────────────┐
│  App A  │  App B  │  App C         │
├─────────┼─────────┼─────────────── │
│ Bins/   │ Bins/   │ Bins/          │
│ Libs    │ Libs    │ Libs           │
├─────────────────────────────────── │
│           Docker Engine           │
├────────────────────────────────────┤
│         Host Operating System     │
├────────────────────────────────────┤
│              Hardware             │
└────────────────────────────────────┘
ขนาด: MB | Startup: วินาที | Overhead: ต่ำ
```

### 1.3 Docker Components

```
┌─────────────────────────────────────────────────────────┐
│                   Docker Architecture                    │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Docker Client (CLI)                │   │
│  │   docker build, docker run, docker pull         │   │
│  └──────────────────────┬──────────────────────────┘   │
│                         │ REST API                      │
│  ┌──────────────────────▼──────────────────────────┐   │
│  │              Docker Daemon (dockerd)             │   │
│  │                                                  │   │
│  │  ┌─────────┐  ┌──────────┐  ┌───────────────┐  │   │
│  │  │ Images  │  │Containers│  │   Volumes     │  │   │
│  │  └─────────┘  └──────────┘  └───────────────┘  │   │
│  │                                                  │   │
│  │  ┌──────────────────────────────────────────┐   │   │
│  │  │           Container Runtime              │   │   │
│  │  │     (containerd + runc)                  │   │   │
│  │  └──────────────────────────────────────────┘   │   │
│  └──────────────────────────────────────────────────┘   │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │           Docker Registry (Docker Hub)           │   │
│  │   nginx:alpine, node:18, postgres:15             │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

### 1.4 Key Concepts

```
Image:    Blueprint/Template สำหรับสร้าง Container
          (Read-only layers)

Container: Instance ที่รันจาก Image
           (Writable layer on top of image)

Registry:  ที่เก็บ Images (Docker Hub, ECR, GCR, etc.)

Volume:    Persistent storage สำหรับ Container

Network:   Virtual network สำหรับ Containers สื่อสารกัน

Layer:     แต่ละ instruction ใน Dockerfile = 1 Layer
           Layers ถูก cache → Build เร็วขึ้น
```

---

## 2. Dockerfile

### 2.1 Dockerfile Instructions

```dockerfile
# ── FROM ──
# Base image
FROM node:18-alpine

# ── LABEL ──
# Metadata
LABEL maintainer="dev@company.com"
LABEL version="1.0.0"
LABEL description="User Microservice"

# ── ARG ──
# Build-time variables
ARG NODE_ENV=production
ARG APP_VERSION=1.0.0

# ── ENV ──
# Runtime environment variables
ENV NODE_ENV=${NODE_ENV}
ENV APP_VERSION=${APP_VERSION}
ENV PORT=3000

# ── WORKDIR ──
# Set working directory
WORKDIR /app

# ── COPY ──
# Copy files from host to container
COPY package*.json ./
COPY src/ ./src/

# ── ADD ──
# Like COPY but can handle URLs and auto-extract archives
# (Prefer COPY over ADD for simple files)
# ADD https://example.com/file.tar.gz /data/

# ── RUN ──
# Execute command during BUILD
RUN npm ci --only=production

# ── EXPOSE ──
# Document which port container listens on
# (Does not actually publish the port)
EXPOSE 3000

# ── USER ──
# Run as non-root user (Security!)
USER node

# ── VOLUME ──
# Create mount point for external volumes
VOLUME ["/app/data"]

# ── CMD ──
# Default command when container starts
# (Can be overridden by docker run)
CMD ["node", "src/server.js"]

# ── ENTRYPOINT ──
# Like CMD but harder to override
# ENTRYPOINT ["node", "src/server.js"]

# ── HEALTHCHECK ──
# Define how to test container health
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1
```

### 2.2 Practical Dockerfile สำหรับ Node.js Microservice

```dockerfile
# Dockerfile
# Multi-stage build สำหรับ Production

# ── Stage 1: Dependencies ──
FROM node:18-alpine AS deps

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install all dependencies (including dev)
RUN npm ci

# ── Stage 2: Build ──
FROM node:18-alpine AS builder

WORKDIR /app

# Copy from deps stage
COPY --from=deps /app/node_modules ./node_modules
COPY . .

# Build TypeScript (ถ้าใช้ TypeScript)
# RUN npm run build

# ── Stage 3: Production ──
FROM node:18-alpine AS production

# Security: Create non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install production dependencies only
RUN npm ci --only=production && npm cache clean --force

# Copy built application
# COPY --from=builder /app/dist ./dist  # For TypeScript
COPY --from=builder /app/src ./src

# Set ownership
RUN chown -R appuser:appgroup /app

# Switch to non-root user
USER appuser

# Set environment
ENV NODE_ENV=production
ENV PORT=3000

# Document port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3000/health', (r) => { process.exit(r.statusCode === 200 ? 0 : 1) })"

# Start application
CMD ["node", "src/server.js"]
```

### 2.3 .dockerignore

```dockerignore
# .dockerignore

# Dependencies (จะ install ใหม่ใน container)
node_modules/
npm-debug.log

# Development files
.env
.env.local
.env.*.local

# Test files
tests/
coverage/
*.test.js
*.spec.js

# Documentation
*.md
docs/

# Git
.git/
.gitignore

# IDE
.vscode/
.idea/

# OS files
.DS_Store
Thumbs.db

# Build artifacts
dist/
build/

# Log files
logs/
*.log
```

---

## 3. Docker Layer Caching

### 3.1 ทำความเข้าใจ Layer Caching

```dockerfile
# ❌ ไม่ดี: ทุกครั้งที่ Code เปลี่ยน = Install npm อีกครั้ง
FROM node:18-alpine

WORKDIR /app

COPY . .           # Copy ทุกไฟล์ก่อน
RUN npm install    # ถ้า code เปลี่ยน = invalidate cache = npm install ใหม่

CMD ["node", "index.js"]
```

```dockerfile
# ✅ ดี: Copy package.json ก่อน เพื่อ cache npm install
FROM node:18-alpine

WORKDIR /app

# Copy only package files FIRST
COPY package*.json ./      # Layer 1: เปลี่ยนน้อย

# Install dependencies
RUN npm ci                 # Layer 2: Cache นาน ถ้า package.json ไม่เปลี่ยน

# Copy source code LAST
COPY . .                   # Layer 3: เปลี่ยนบ่อย แต่ไม่ affect npm install

CMD ["node", "src/index.js"]
```

```
Build ครั้งแรก:
Layer 1: FROM node:18-alpine        [BUILT]  - 3s
Layer 2: WORKDIR /app               [BUILT]  - 0.1s
Layer 3: COPY package*.json         [BUILT]  - 0.1s
Layer 4: RUN npm ci                 [BUILT]  - 45s  ← ช้า
Layer 5: COPY . .                   [BUILT]  - 0.5s
Total: ~49s

Build ครั้งถัดไป (แก้ code แต่ไม่แก้ package.json):
Layer 1: FROM node:18-alpine        [CACHED] - 0s
Layer 2: WORKDIR /app               [CACHED] - 0s
Layer 3: COPY package*.json         [CACHED] - 0s
Layer 4: RUN npm ci                 [CACHED] - 0s  ← จาก cache!
Layer 5: COPY . .                   [BUILT]  - 0.5s
Total: ~0.5s  ← เร็วขึ้น 100x!
```

---

## 4. Docker Commands

### 4.1 Image Commands

```bash
# Build image
docker build -t user-service:1.0.0 .
docker build -t user-service:1.0.0 --build-arg NODE_ENV=production .
docker build -f Dockerfile.prod -t user-service:latest .

# List images
docker images
docker image ls

# Pull image from registry
docker pull node:18-alpine
docker pull postgres:15

# Remove image
docker rmi user-service:1.0.0
docker image rm user-service:1.0.0

# Remove all unused images
docker image prune
docker image prune -a  # Remove all unused

# Inspect image
docker inspect user-service:1.0.0
docker history user-service:1.0.0  # ดู layers

# Tag image
docker tag user-service:1.0.0 registry.company.com/user-service:1.0.0

# Push to registry
docker push registry.company.com/user-service:1.0.0
```

### 4.2 Container Commands

```bash
# Run container
docker run user-service:1.0.0
docker run -d user-service:1.0.0                    # detached mode
docker run -p 3001:3000 user-service:1.0.0          # port mapping
docker run --name user-svc user-service:1.0.0       # custom name
docker run -e PORT=3000 user-service:1.0.0          # env variable
docker run -e PORT=3000 --env-file .env user-service:1.0.0
docker run --rm user-service:1.0.0                  # auto-remove when stopped

# List containers
docker ps                  # running containers
docker ps -a               # all containers (including stopped)

# Stop/Start/Restart
docker stop user-svc
docker start user-svc
docker restart user-svc

# Remove container
docker rm user-svc
docker rm -f user-svc      # force remove (even if running)
docker container prune     # remove all stopped containers

# View logs
docker logs user-svc
docker logs -f user-svc    # follow logs
docker logs --tail 50 user-svc  # last 50 lines

# Execute command in container
docker exec -it user-svc sh          # interactive shell
docker exec user-svc npm test        # run command
docker exec -it user-svc psql -U postgres  # PostgreSQL

# Copy files
docker cp user-svc:/app/logs ./local-logs
docker cp ./config.json user-svc:/app/config.json

# Inspect container
docker inspect user-svc
docker stats user-svc      # real-time resource usage
docker top user-svc        # processes in container
```

### 4.3 Volume Commands

```bash
# Create volume
docker volume create pgdata

# List volumes
docker volume ls

# Inspect volume
docker volume inspect pgdata

# Remove volume
docker volume rm pgdata
docker volume prune        # remove all unused volumes

# Mount volume
docker run -v pgdata:/var/lib/postgresql/data postgres:15

# Mount directory (bind mount)
docker run -v $(pwd)/src:/app/src user-service:dev

# Read-only mount
docker run -v $(pwd)/config:/app/config:ro user-service:1.0.0
```

### 4.4 Network Commands

```bash
# Create network
docker network create microservices-net
docker network create --driver bridge my-network
docker network create --subnet=172.20.0.0/16 custom-network

# List networks
docker network ls

# Connect container to network
docker network connect microservices-net user-svc
docker run --network microservices-net user-service:1.0.0

# Disconnect from network
docker network disconnect microservices-net user-svc

# Inspect network
docker network inspect microservices-net

# Remove network
docker network rm microservices-net
docker network prune       # remove all unused networks
```

---

## 5. Container Networking

### 5.1 Network Types

```
Bridge Network (default):
┌─────────────────────────────────────────────────────────┐
│                Docker Host                              │
│                                                         │
│  Container A ──────┐                                   │
│  (172.17.0.2)      │                                   │
│                    ▼                                   │
│  Container B ── docker0 bridge (172.17.0.1) ──► Host  │
│  (172.17.0.3)      ▲                                   │
│                    │                                   │
│  Container C ──────┘                                   │
│  (172.17.0.4)                                          │
└─────────────────────────────────────────────────────────┘
Containers สื่อสารกันได้ผ่าน IP

Custom Bridge Network:
Containers สามารถ ใช้ container name แทน IP ได้!
Container A → http://user-service:3001 (DNS resolution)
```

### 5.2 Container DNS

```bash
# สร้าง custom network
docker network create microservices-net

# Run containers ใน network เดียวกัน
docker run -d --name user-service \
  --network microservices-net \
  -p 3001:3001 \
  user-service:1.0.0

docker run -d --name order-service \
  --network microservices-net \
  -p 3002:3002 \
  -e USER_SERVICE_URL=http://user-service:3001 \
  order-service:1.0.0

# Order Service สามารถเรียก User Service ด้วย
# http://user-service:3001 (Container name เป็น DNS)
```

---

## 6. Image Optimization

### 6.1 เลือก Base Image ที่เหมาะสม

```dockerfile
# ขนาดเปรียบเทียบ:
FROM node:18           # ~950MB - Full image (ไม่จำเป็น)
FROM node:18-slim      # ~170MB - Smaller, less tools
FROM node:18-alpine    # ~55MB  - Smallest, Alpine Linux
FROM node:18-distroless # ~35MB  - No shell, super secure

# สำหรับ Production ใช้ alpine หรือ distroless
FROM node:18-alpine AS production
```

### 6.2 Multi-Stage Build

```dockerfile
# Dockerfile.prod - Optimized for production

# Stage 1: Build TypeScript
FROM node:18-alpine AS build
WORKDIR /app
COPY package*.json tsconfig.json ./
RUN npm ci
COPY src/ ./src/
RUN npm run build   # Compile TypeScript → JavaScript

# Stage 2: Production
FROM node:18-alpine AS production
WORKDIR /app

# Only production dependencies
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

# Only compiled output
COPY --from=build /app/dist ./dist

USER node
ENV NODE_ENV=production
EXPOSE 3000
CMD ["node", "dist/server.js"]

# Build: docker build -f Dockerfile.prod -t myapp:prod .
# ขนาดลดลงจาก 500MB → 80MB
```

### 6.3 Minimize Layers

```dockerfile
# ❌ หลาย RUN commands = หลาย layers
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y git
RUN apt-get clean

# ✅ รวม RUN commands ด้วย && 
RUN apt-get update -y && \
    apt-get install -y curl git && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

---

## 7. Docker Security Best Practices

### 7.1 Run as Non-Root

```dockerfile
# ❌ ไม่ดี: Run เป็น root
FROM node:18-alpine
WORKDIR /app
COPY . .
CMD ["node", "index.js"]
# Container runs as root → Security risk!

# ✅ ดี: Run เป็น non-root user
FROM node:18-alpine

# node user มาพร้อม node:alpine image แล้ว
USER node

WORKDIR /app

# ต้อง chown ก่อน switch user
COPY --chown=node:node . .

CMD ["node", "index.js"]
```

### 7.2 Read-Only Filesystem

```bash
# Run container ด้วย read-only filesystem
docker run --read-only \
  --tmpfs /tmp \                    # Writable /tmp
  --tmpfs /var/run \                # Writable /var/run
  user-service:1.0.0
```

### 7.3 Limit Resources

```bash
# จำกัด Resources
docker run \
  --memory="512m" \          # Max 512MB RAM
  --memory-swap="512m" \     # No swap
  --cpus="0.5" \             # Max 50% CPU
  user-service:1.0.0

# จำกัด Capabilities
docker run \
  --cap-drop ALL \           # Drop all capabilities
  --cap-add NET_BIND_SERVICE \  # Add only needed
  user-service:1.0.0
```

### 7.4 Scan for Vulnerabilities

```bash
# Docker Scout (built-in)
docker scout cves user-service:1.0.0

# Trivy
trivy image user-service:1.0.0

# Snyk
snyk container test user-service:1.0.0
```

---

## 8. Workshop: Dockerize Microservices

### 8.1 โปรเจค: E-Commerce Microservices

```bash
# โครงสร้าง
ecommerce/
├── user-service/
│   ├── src/
│   │   ├── app.js
│   │   └── server.js
│   ├── Dockerfile
│   ├── .dockerignore
│   └── package.json
├── product-service/
│   ├── src/
│   ├── Dockerfile
│   ├── .dockerignore
│   └── package.json
├── order-service/
│   ├── src/
│   ├── Dockerfile
│   ├── .dockerignore
│   └── package.json
└── docker-compose.yml
```

### 8.2 User Service

```javascript
// user-service/src/app.js
const express = require('express');
const app = express();
app.use(express.json());

// ใช้ Map แทน Database (demo)
const users = new Map([
  [1, { id: 1, name: 'สมชาย', email: 'somchai@example.com', role: 'customer' }],
  [2, { id: 2, name: 'สมหญิง', email: 'somying@example.com', role: 'admin' }],
]);

app.get('/health', (req, res) => res.json({ status: 'healthy', service: 'user-service' }));

app.get('/users', (req, res) => {
  res.json({ success: true, data: Array.from(users.values()) });
});

app.get('/users/:id', (req, res) => {
  const user = users.get(parseInt(req.params.id));
  if (!user) return res.status(404).json({ success: false, error: 'User not found' });
  res.json({ success: true, data: user });
});

app.post('/users', (req, res) => {
  const { name, email, role = 'customer' } = req.body;
  if (!name || !email) return res.status(400).json({ error: 'name and email required' });
  
  const id = users.size + 1;
  const user = { id, name, email, role, createdAt: new Date().toISOString() };
  users.set(id, user);
  res.status(201).json({ success: true, data: user });
});

module.exports = app;
```

### 8.3 Dockerfile สำหรับ User Service

```dockerfile
# user-service/Dockerfile

FROM node:18-alpine AS production

# Create app directory
WORKDIR /app

# Install dependencies
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

# Copy source
COPY src/ ./src/

# Security: run as non-root
USER node

# Configuration
ENV NODE_ENV=production
ENV PORT=3001

EXPOSE 3001

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD wget -q --spider http://localhost:3001/health || exit 1

CMD ["node", "src/server.js"]
```

### 8.4 Product Service

```javascript
// product-service/src/app.js
const express = require('express');
const app = express();
app.use(express.json());

const products = new Map([
  [1, { id: 1, name: 'MacBook Pro M2', price: 75000, stock: 10, category: 'laptop' }],
  [2, { id: 2, name: 'iPhone 15 Pro', price: 42000, stock: 25, category: 'phone' }],
  [3, { id: 3, name: 'AirPods Pro', price: 9500, stock: 50, category: 'audio' }],
]);

app.get('/health', (req, res) => res.json({ status: 'healthy', service: 'product-service' }));

app.get('/products', (req, res) => {
  const { category } = req.query;
  let data = Array.from(products.values());
  
  if (category) {
    data = data.filter(p => p.category === category);
  }
  
  res.json({ success: true, data, total: data.length });
});

app.get('/products/:id', (req, res) => {
  const product = products.get(parseInt(req.params.id));
  if (!product) return res.status(404).json({ error: 'Product not found' });
  res.json({ success: true, data: product });
});

// Check and reserve stock (for order service)
app.post('/products/:id/reserve', (req, res) => {
  const product = products.get(parseInt(req.params.id));
  if (!product) return res.status(404).json({ error: 'Product not found' });
  
  const { quantity } = req.body;
  if (product.stock < quantity) {
    return res.status(400).json({ error: 'Insufficient stock' });
  }
  
  product.stock -= quantity;
  res.json({ success: true, remainingStock: product.stock });
});

module.exports = app;
```

### 8.5 Order Service

```javascript
// order-service/src/app.js
const express = require('express');
const axios = require('axios');
const app = express();
app.use(express.json());

const USER_SERVICE = process.env.USER_SERVICE_URL || 'http://localhost:3001';
const PRODUCT_SERVICE = process.env.PRODUCT_SERVICE_URL || 'http://localhost:3003';

const orders = new Map();
let orderId = 1;

app.get('/health', async (req, res) => {
  // Check dependencies
  const deps = {};
  
  try {
    await axios.get(`${USER_SERVICE}/health`, { timeout: 2000 });
    deps.userService = 'healthy';
  } catch {
    deps.userService = 'unavailable';
  }
  
  try {
    await axios.get(`${PRODUCT_SERVICE}/health`, { timeout: 2000 });
    deps.productService = 'healthy';
  } catch {
    deps.productService = 'unavailable';
  }
  
  const isHealthy = Object.values(deps).every(v => v === 'healthy');
  
  res.status(isHealthy ? 200 : 503).json({
    status: isHealthy ? 'healthy' : 'degraded',
    service: 'order-service',
    dependencies: deps
  });
});

app.get('/orders', (req, res) => {
  res.json({ success: true, data: Array.from(orders.values()) });
});

app.post('/orders', async (req, res) => {
  const { userId, productId, quantity } = req.body;
  
  if (!userId || !productId || !quantity) {
    return res.status(400).json({ error: 'userId, productId, quantity required' });
  }
  
  try {
    // Validate user
    const userRes = await axios.get(`${USER_SERVICE}/users/${userId}`);
    const user = userRes.data.data;
    
    // Get product
    const productRes = await axios.get(`${PRODUCT_SERVICE}/products/${productId}`);
    const product = productRes.data.data;
    
    // Reserve stock
    await axios.post(`${PRODUCT_SERVICE}/products/${productId}/reserve`, { quantity });
    
    // Create order
    const order = {
      id: orderId++,
      userId,
      productId,
      quantity,
      totalAmount: product.price * quantity,
      status: 'confirmed',
      customerName: user.name,
      productName: product.name,
      createdAt: new Date().toISOString()
    };
    
    orders.set(order.id, order);
    res.status(201).json({ success: true, data: order });
    
  } catch (error) {
    if (error.response) {
      return res.status(error.response.status).json({ error: error.response.data.error });
    }
    res.status(503).json({ error: 'Service unavailable', message: error.message });
  }
});

module.exports = app;
```

### 8.6 Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

# Custom network for service discovery
networks:
  microservices:
    driver: bridge

services:
  # ── User Service ──
  user-service:
    build: 
      context: ./user-service
      dockerfile: Dockerfile
    container_name: user-service
    ports:
      - "3001:3001"
    environment:
      - NODE_ENV=production
      - PORT=3001
    networks:
      - microservices
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:3001/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s

  # ── Product Service ──
  product-service:
    build:
      context: ./product-service
      dockerfile: Dockerfile
    container_name: product-service
    ports:
      - "3003:3003"
    environment:
      - NODE_ENV=production
      - PORT=3003
    networks:
      - microservices
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:3003/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s

  # ── Order Service ──
  order-service:
    build:
      context: ./order-service
      dockerfile: Dockerfile
    container_name: order-service
    ports:
      - "3002:3002"
    environment:
      - NODE_ENV=production
      - PORT=3002
      - USER_SERVICE_URL=http://user-service:3001
      - PRODUCT_SERVICE_URL=http://product-service:3003
    networks:
      - microservices
    depends_on:
      user-service:
        condition: service_healthy
      product-service:
        condition: service_healthy
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:3002/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 15s
```

### 8.7 Build และ Test

```bash
# Build ทุก services
docker-compose build

# Start
docker-compose up -d

# ดู status
docker-compose ps

# Test APIs
echo "=== Test User Service ==="
curl -s http://localhost:3001/users | jq

echo ""
echo "=== Test Product Service ==="
curl -s http://localhost:3003/products | jq

echo ""
echo "=== Test Create Order ==="
curl -s -X POST http://localhost:3002/orders \
  -H "Content-Type: application/json" \
  -d '{"userId": 1, "productId": 1, "quantity": 1}' | jq

echo ""
echo "=== Test Health Checks ==="
curl -s http://localhost:3001/health | jq
curl -s http://localhost:3002/health | jq
curl -s http://localhost:3003/health | jq

# View logs
docker-compose logs -f

# Clean up
docker-compose down
```

### 8.8 Expected Output

```json
// GET /orders after creating an order
{
  "success": true,
  "data": [
    {
      "id": 1,
      "userId": 1,
      "productId": 1,
      "quantity": 1,
      "totalAmount": 75000,
      "status": "confirmed",
      "customerName": "สมชาย",
      "productName": "MacBook Pro M2",
      "createdAt": "2024-01-15T10:30:00.000Z"
    }
  ]
}
```

---

## 9. สรุปและ Best Practices

### Docker Best Practices Checklist

```
Image:
✅ ใช้ Alpine base image เพื่อลดขนาด
✅ Multi-stage builds
✅ .dockerignore ที่ครบถ้วน
✅ Copy package.json ก่อน source code (caching)
✅ Pin specific versions (node:18-alpine ไม่ใช่ node:latest)

Security:
✅ Run as non-root user
✅ Read-only filesystem (ถ้าเป็นไปได้)
✅ Scan images for vulnerabilities
✅ Minimal capabilities (--cap-drop ALL)
✅ ไม่ใส่ secrets ใน Dockerfile หรือ image

Runtime:
✅ Health checks
✅ Resource limits (--memory, --cpus)
✅ Graceful shutdown (SIGTERM handling)
✅ Structured logging to stdout
```

### คำถามทบทวน

1. ต่างกันอย่างไระหว่าง `CMD` และ `ENTRYPOINT`?
2. Layer caching ทำงานอย่างไร? ทำไมต้อง copy package.json ก่อน?
3. เมื่อไหร่ใช้ Multi-stage builds?
4. ทำไม Container ถึงควร Run เป็น non-root user?
5. Container DNS ทำงานอย่างไรใน Custom Bridge Network?

### แบบฝึกหัด

1. เพิ่ม Notification Service ที่รับ events และ log ออก console
2. ทดสอบว่าเกิดอะไรขึ้นเมื่อ user-service down ขณะ create order
3. เพิ่ม Resource Limits ใน docker-compose.yml
4. Scan image ด้วย Trivy และ fix vulnerabilities ที่พบ
5. Optimize Dockerfile เพื่อลดขนาด image ลง 30%

---

**ต่อไป:** [Part 06 - Docker Compose สำหรับ Multi-Container Management →](part-06.md)

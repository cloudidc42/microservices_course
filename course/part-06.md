# Part 06: Docker Compose สำหรับ Multi-Container Management
## จัดการ Microservices หลายตัวพร้อมกันด้วย Docker Compose

> **ระดับ:** ⭐⭐ พื้นฐาน | **เวลาเรียน:** 4-5 ชั่วโมง | **Prerequisites:** Part 05

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Docker Compose Architecture และ YAML Syntax
- Service Configuration ที่สมบูรณ์
- Health Checks และ Dependencies
- Volumes และ Networks
- Environment Management
- Profiles สำหรับ Different Setups
- Workshop: Complete E-Commerce Stack

---

## 1. Docker Compose Overview

Docker Compose ช่วยให้เราสามารถ Define และ Run Multi-Container Applications ด้วยไฟล์ YAML เดียว

```yaml
# docker-compose.yml โครงสร้างหลัก
version: '3.8'

services:
  # แต่ละ service = 1 container
  service-name:
    image: ...
    build: ...
    ports: ...
    environment: ...
    volumes: ...
    networks: ...
    depends_on: ...
    healthcheck: ...

volumes:
  # Named volumes
  data-volume:

networks:
  # Custom networks
  my-network:
```

### Docker Compose vs Docker Run

```bash
# Docker Run - ต้องพิมพ์ทุกครั้ง
docker run -d \
  --name user-service \
  --network microservices \
  -p 3001:3001 \
  -e NODE_ENV=production \
  -e PORT=3001 \
  -e DB_HOST=postgres \
  --restart unless-stopped \
  user-service:1.0.0

# Docker Compose - Define once, run anytime
docker-compose up -d
```

---

## 2. Complete Service Configuration

### 2.1 Service Options ทั้งหมด

```yaml
version: '3.8'

services:
  web:
    # ── Image ──
    image: nginx:alpine                    # ใช้ existing image
    
    # หรือ Build จาก Dockerfile
    build:
      context: ./web                       # Path to directory
      dockerfile: Dockerfile.prod          # Dockerfile filename
      args:                               # Build arguments
        NODE_ENV: production
        APP_VERSION: "1.0.0"
      target: production                  # Multi-stage target
      cache_from:                         # Cache sources
        - registry.company.com/web:latest
      labels:
        app.version: "1.0.0"
    
    # ── Container Name ──
    container_name: web-app               # Custom name
    
    # ── Ports ──
    ports:
      - "80:80"                           # HOST:CONTAINER
      - "443:443"
      - "127.0.0.1:8080:8080"            # Bind to specific IP
    
    # ── Environment ──
    environment:
      - NODE_ENV=production               # List format
      - PORT=80
    
    # หรือ Map format
    environment:
      NODE_ENV: production
      PORT: "80"
    
    # Load from file
    env_file:
      - .env                             # Common env
      - .env.production                  # Environment-specific
    
    # ── Volumes ──
    volumes:
      - ./src:/app/src                   # Bind mount
      - app-data:/data                   # Named volume
      - /tmp:/tmp:ro                     # Read-only
    
    # ── Networks ──
    networks:
      - frontend
      - backend
    
    # Network alias
    networks:
      backend:
        aliases:
          - web-internal
    
    # ── Dependencies ──
    depends_on:
      - database
      - redis
    
    # With health condition
    depends_on:
      database:
        condition: service_healthy
      redis:
        condition: service_started
    
    # ── Health Check ──
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    
    # ── Restart Policy ──
    restart: unless-stopped              # no | always | on-failure | unless-stopped
    
    # ── Resource Limits ──
    deploy:
      resources:
        limits:
          cpus: '0.50'
          memory: 512M
        reservations:
          memory: 128M
    
    # ── Logging ──
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
    
    # ── Labels ──
    labels:
      app.name: "web"
      app.version: "1.0.0"
      traefik.enable: "true"
    
    # ── Extra Hosts ──
    extra_hosts:
      - "host.docker.internal:host-gateway"
    
    # ── Profiles ──
    profiles:
      - debug
```

---

## 3. Complete E-Commerce Stack

### 3.1 Services ทั้งหมด

```yaml
# docker-compose.yml - Complete E-Commerce Microservices

version: '3.8'

# ── Networks ──
networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
  monitoring:
    driver: bridge

# ── Volumes ──
volumes:
  postgres_data:
    driver: local
  redis_data:
    driver: local
  mongodb_data:
    driver: local
  elasticsearch_data:
    driver: local
  grafana_data:
    driver: local

services:

  # ════════════════════════════════════════════
  # API GATEWAY
  # ════════════════════════════════════════════
  api-gateway:
    image: nginx:alpine
    container_name: api-gateway
    ports:
      - "8080:80"
    volumes:
      - ./api-gateway/nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - user-service
      - order-service
      - product-service
    networks:
      - frontend
      - backend
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "nginx", "-t"]
      interval: 30s
      timeout: 10s
      retries: 3

  # ════════════════════════════════════════════
  # MICROSERVICES
  # ════════════════════════════════════════════
  
  user-service:
    build:
      context: ./services/user-service
      target: production
    container_name: user-service
    ports:
      - "3001:3001"
    environment:
      - NODE_ENV=production
      - PORT=3001
      - DB_HOST=postgres
      - DB_PORT=5432
      - DB_NAME=userdb
      - DB_USER=postgres
      - DB_PASSWORD=${POSTGRES_PASSWORD}
      - REDIS_URL=redis://redis:6379
      - JWT_SECRET=${JWT_SECRET}
    env_file:
      - .env
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - backend
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:3001/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 20s
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "5"

  order-service:
    build:
      context: ./services/order-service
      target: production
    container_name: order-service
    ports:
      - "3002:3002"
    environment:
      - NODE_ENV=production
      - PORT=3002
      - DB_HOST=mongodb
      - DB_PORT=27017
      - DB_NAME=orderdb
      - USER_SERVICE_URL=http://user-service:3001
      - PRODUCT_SERVICE_URL=http://product-service:3003
      - PAYMENT_SERVICE_URL=http://payment-service:3004
      - RABBITMQ_URL=amqp://admin:${RABBITMQ_PASSWORD}@rabbitmq:5672
    depends_on:
      mongodb:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy
      user-service:
        condition: service_healthy
    networks:
      - backend
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:3002/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 25s

  product-service:
    build:
      context: ./services/product-service
      target: production
    container_name: product-service
    ports:
      - "3003:3003"
    environment:
      - NODE_ENV=production
      - PORT=3003
      - DB_HOST=postgres
      - DB_NAME=productdb
      - DB_USER=postgres
      - DB_PASSWORD=${POSTGRES_PASSWORD}
      - ELASTICSEARCH_URL=http://elasticsearch:9200
      - REDIS_URL=redis://redis:6379
    depends_on:
      postgres:
        condition: service_healthy
      elasticsearch:
        condition: service_healthy
    networks:
      - backend
    restart: unless-stopped

  payment-service:
    build:
      context: ./services/payment-service
      target: production
    container_name: payment-service
    ports:
      - "3004:3004"
    environment:
      - NODE_ENV=production
      - PORT=3004
      - DB_HOST=postgres
      - DB_NAME=paymentdb
      - STRIPE_KEY=${STRIPE_SECRET_KEY}
      - RABBITMQ_URL=amqp://admin:${RABBITMQ_PASSWORD}@rabbitmq:5672
    depends_on:
      postgres:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy
    networks:
      - backend
    restart: unless-stopped

  notification-service:
    build:
      context: ./services/notification-service
      target: production
    container_name: notification-service
    environment:
      - NODE_ENV=production
      - SMTP_HOST=mailhog
      - SMTP_PORT=1025
      - RABBITMQ_URL=amqp://admin:${RABBITMQ_PASSWORD}@rabbitmq:5672
    depends_on:
      rabbitmq:
        condition: service_healthy
    networks:
      - backend
    restart: unless-stopped

  # ════════════════════════════════════════════
  # DATABASES
  # ════════════════════════════════════════════

  postgres:
    image: postgres:15-alpine
    container_name: postgres
    environment:
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_USER=postgres
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./database/init:/docker-entrypoint-initdb.d  # Init scripts
    networks:
      - backend
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

  mongodb:
    image: mongo:6-jammy
    container_name: mongodb
    environment:
      - MONGO_INITDB_ROOT_USERNAME=admin
      - MONGO_INITDB_ROOT_PASSWORD=${MONGO_PASSWORD}
      - MONGO_INITDB_DATABASE=orderdb
    volumes:
      - mongodb_data:/data/db
      - ./database/mongo-init.js:/docker-entrypoint-initdb.d/init.js:ro
    networks:
      - backend
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    container_name: redis
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data
    networks:
      - backend
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "--no-auth-warning", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  elasticsearch:
    image: elasticsearch:8.10.0
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - ES_JAVA_OPTS=-Xms512m -Xmx512m
    volumes:
      - elasticsearch_data:/usr/share/elasticsearch/data
    networks:
      - backend
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:9200/_cluster/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s

  # ════════════════════════════════════════════
  # MESSAGE BROKER
  # ════════════════════════════════════════════

  rabbitmq:
    image: rabbitmq:3-management-alpine
    container_name: rabbitmq
    environment:
      - RABBITMQ_DEFAULT_USER=admin
      - RABBITMQ_DEFAULT_PASS=${RABBITMQ_PASSWORD}
    ports:
      - "5672:5672"
      - "15672:15672"   # Management UI
    volumes:
      - ./rabbitmq/rabbitmq.conf:/etc/rabbitmq/rabbitmq.conf:ro
    networks:
      - backend
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "check_port_connectivity"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s

  # ════════════════════════════════════════════
  # MONITORING (Development Profile)
  # ════════════════════════════════════════════

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml:ro
    networks:
      - backend
      - monitoring
    profiles:
      - monitoring
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3100:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}
    volumes:
      - grafana_data:/var/lib/grafana
      - ./monitoring/grafana/dashboards:/etc/grafana/provisioning/dashboards:ro
    networks:
      - monitoring
    depends_on:
      - prometheus
    profiles:
      - monitoring
    restart: unless-stopped

  # ════════════════════════════════════════════
  # DEVELOPMENT TOOLS (Dev Profile)
  # ════════════════════════════════════════════

  mailhog:
    image: mailhog/mailhog:latest
    container_name: mailhog
    ports:
      - "1025:1025"   # SMTP
      - "8025:8025"   # Web UI
    networks:
      - backend
    profiles:
      - dev
    restart: unless-stopped

  pgadmin:
    image: dpage/pgadmin4:latest
    container_name: pgadmin
    environment:
      - PGADMIN_DEFAULT_EMAIL=admin@company.com
      - PGADMIN_DEFAULT_PASSWORD=${PGADMIN_PASSWORD}
    ports:
      - "5050:80"
    networks:
      - backend
    depends_on:
      - postgres
    profiles:
      - dev
    restart: unless-stopped
```

### 3.2 API Gateway Configuration

```nginx
# api-gateway/nginx.conf
events {
  worker_connections 1024;
}

http {
  # ── Upstream Services ──
  upstream user_service {
    server user-service:3001;
    keepalive 32;
  }
  
  upstream order_service {
    server order-service:3002;
    keepalive 32;
  }
  
  upstream product_service {
    server product-service:3003;
    keepalive 32;
  }
  
  upstream payment_service {
    server payment-service:3004;
    keepalive 32;
  }

  # ── Main Server ──
  server {
    listen 80;
    
    # Security headers
    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";
    add_header X-XSS-Protection "1; mode=block";
    
    # Request ID
    add_header X-Request-ID $request_id;
    
    # ── Routes ──
    location /api/users {
      proxy_pass http://user_service;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Request-ID $request_id;
      proxy_connect_timeout 10s;
      proxy_read_timeout 30s;
    }
    
    location /api/orders {
      proxy_pass http://order_service;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Request-ID $request_id;
    }
    
    location /api/products {
      proxy_pass http://product_service;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Request-ID $request_id;
    }
    
    location /api/payments {
      proxy_pass http://payment_service;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Request-ID $request_id;
    }
    
    # ── Health Check ──
    location /health {
      return 200 'Gateway OK\n';
      add_header Content-Type text/plain;
    }
  }
}
```

### 3.3 Environment File

```bash
# .env.example
# Copy to .env and fill in values

# Database
POSTGRES_PASSWORD=strongpassword123
MONGO_PASSWORD=mongopassword123
REDIS_PASSWORD=redispassword123

# Auth
JWT_SECRET=your-super-secret-jwt-key-minimum-32-chars

# External Services
STRIPE_SECRET_KEY=sk_test_...

# Message Broker
RABBITMQ_PASSWORD=rabbitpassword123

# Monitoring
GRAFANA_PASSWORD=grafanapassword123

# Dev Tools
PGADMIN_PASSWORD=pgadminpassword123
```

---

## 4. Docker Compose Commands

### 4.1 Essential Commands

```bash
# ── Start/Stop ──
docker-compose up                        # Start (foreground)
docker-compose up -d                     # Start (background)
docker-compose up --build                # Build then start
docker-compose up --force-recreate       # Recreate containers
docker-compose up -d user-service        # Start specific service

docker-compose down                      # Stop and remove
docker-compose down -v                   # + remove volumes
docker-compose down --remove-orphans     # + remove orphan containers

docker-compose stop                      # Stop (keep containers)
docker-compose start                     # Start stopped containers
docker-compose restart user-service      # Restart specific service

# ── Profiles ──
docker-compose --profile dev up -d       # Start with dev profile
docker-compose --profile monitoring up -d  # Start with monitoring

# Multiple profiles
docker-compose --profile dev --profile monitoring up -d

# ── Status ──
docker-compose ps                        # List services
docker-compose ps --services             # Service names only
docker-compose top                       # Running processes

# ── Logs ──
docker-compose logs                      # All logs
docker-compose logs -f                   # Follow logs
docker-compose logs user-service         # Specific service
docker-compose logs -f --tail=50 user-service

# ── Execute ──
docker-compose exec user-service sh      # Shell
docker-compose exec postgres psql -U postgres
docker-compose run --rm user-service npm test

# ── Scaling ──
docker-compose up -d --scale user-service=3

# ── Config ──
docker-compose config                    # Validate and view config
docker-compose config --services         # List services
```

### 4.2 Override Files

```yaml
# docker-compose.yml (base)
services:
  user-service:
    image: user-service:1.0.0
    environment:
      - NODE_ENV=production

# docker-compose.dev.yml (development override)
services:
  user-service:
    build: ./user-service   # Build instead of pull
    volumes:
      - ./user-service/src:/app/src  # Hot reload
    environment:
      - NODE_ENV=development
      - DEBUG=true
    command: npm run dev    # Override command
```

```bash
# ใช้ Override Files
docker-compose -f docker-compose.yml -f docker-compose.dev.yml up

# หรือตั้ง Environment Variable
export COMPOSE_FILE=docker-compose.yml:docker-compose.dev.yml
docker-compose up
```

---

## 5. Database Initialization

### 5.1 PostgreSQL Init Scripts

```sql
-- database/init/01_create_databases.sql
-- Script นี้รันตอน PostgreSQL container เริ่มต้น

CREATE DATABASE userdb;
CREATE DATABASE productdb;
CREATE DATABASE paymentdb;

-- Create databases with specific settings
CREATE DATABASE analyticsdb
  WITH 
  ENCODING='UTF8'
  LC_COLLATE='en_US.UTF-8'
  TEMPLATE=template0;
```

```sql
-- database/init/02_create_schemas.sql
\c userdb;

CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  email VARCHAR(255) UNIQUE NOT NULL,
  name VARCHAR(255) NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  role VARCHAR(50) DEFAULT 'customer',
  is_active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);

\c productdb;

CREATE TABLE products (
  id SERIAL PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  description TEXT,
  price DECIMAL(10,2) NOT NULL,
  stock INTEGER DEFAULT 0,
  category VARCHAR(100),
  image_url VARCHAR(500),
  is_active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMP DEFAULT NOW()
);
```

### 5.2 MongoDB Init Script

```javascript
// database/mongo-init.js
// รันตอน MongoDB container เริ่มต้น

db = db.getSiblingDB('orderdb');

// Create collections with validation
db.createCollection('orders', {
  validator: {
    $jsonSchema: {
      bsonType: 'object',
      required: ['userId', 'items', 'status'],
      properties: {
        userId: { bsonType: 'int' },
        status: { 
          enum: ['pending', 'confirmed', 'shipped', 'delivered', 'cancelled'] 
        },
        items: {
          bsonType: 'array',
          items: {
            bsonType: 'object',
            required: ['productId', 'quantity', 'price'],
          }
        }
      }
    }
  }
});

// Create indexes
db.orders.createIndex({ userId: 1, createdAt: -1 });
db.orders.createIndex({ status: 1 });
db.orders.createIndex({ createdAt: -1 }, { expireAfterSeconds: 31536000 }); // TTL 1 year

print('MongoDB initialization complete');
```

---

## 6. Health Checks และ Startup Order

### 6.1 Dependency Chain

```yaml
# Service startup order กับ health conditions
services:
  # ── ระดับที่ 1: Databases (ไม่มี dependencies) ──
  postgres:
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      retries: 5

  redis:
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      retries: 5

  # ── ระดับที่ 2: Services (depend on databases) ──
  user-service:
    depends_on:
      postgres:
        condition: service_healthy  # รอจน postgres healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:3001/health"]
      interval: 30s
      start_period: 20s  # รอ 20 วินาทีก่อน check

  # ── ระดับที่ 3: Services (depend on other services) ──
  order-service:
    depends_on:
      user-service:
        condition: service_healthy  # รอจน user-service healthy
      payment-service:
        condition: service_healthy

  # ── ระดับที่ 4: Gateway (depend on all services) ──
  api-gateway:
    depends_on:
      - user-service
      - order-service
      - product-service
```

### 6.2 Custom Health Check

```javascript
// health.js - Comprehensive health check
const express = require('express');
const { Pool } = require('pg');
const Redis = require('ioredis');
const axios = require('axios');

const router = express.Router();
const db = new Pool({ connectionString: process.env.DATABASE_URL });
const redis = new Redis(process.env.REDIS_URL);

// Basic liveness probe
router.get('/live', (req, res) => {
  res.json({ status: 'alive', uptime: process.uptime() });
});

// Readiness probe (ใช้ by Kubernetes)
router.get('/ready', async (req, res) => {
  try {
    await db.query('SELECT 1');
    await redis.ping();
    res.json({ status: 'ready' });
  } catch (error) {
    res.status(503).json({ status: 'not ready', error: error.message });
  }
});

// Full health check
router.get('/health', async (req, res) => {
  const health = {
    service: process.env.SERVICE_NAME || 'unknown',
    version: process.env.APP_VERSION || '1.0.0',
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
    checks: {}
  };
  
  // Check PostgreSQL
  try {
    const start = Date.now();
    await db.query('SELECT 1');
    health.checks.database = {
      status: 'healthy',
      latency: `${Date.now() - start}ms`
    };
  } catch (error) {
    health.checks.database = {
      status: 'unhealthy',
      error: error.message
    };
  }
  
  // Check Redis
  try {
    const start = Date.now();
    await redis.ping();
    health.checks.cache = {
      status: 'healthy',
      latency: `${Date.now() - start}ms`
    };
  } catch (error) {
    health.checks.cache = {
      status: 'unhealthy',
      error: error.message
    };
  }
  
  // Check external dependencies
  if (process.env.USER_SERVICE_URL) {
    try {
      const start = Date.now();
      await axios.get(`${process.env.USER_SERVICE_URL}/live`, { timeout: 3000 });
      health.checks.userService = {
        status: 'healthy',
        latency: `${Date.now() - start}ms`
      };
    } catch (error) {
      health.checks.userService = {
        status: 'unhealthy',
        error: error.message
      };
    }
  }
  
  const isHealthy = Object.values(health.checks).every(c => c.status === 'healthy');
  health.status = isHealthy ? 'healthy' : 'degraded';
  
  res.status(isHealthy ? 200 : 503).json(health);
});

module.exports = router;
```

---

## 7. Development vs Production

### 7.1 docker-compose.dev.yml

```yaml
# docker-compose.dev.yml
version: '3.8'

services:
  user-service:
    build:
      context: ./services/user-service
      target: development    # Development stage
    volumes:
      - ./services/user-service/src:/app/src  # Hot reload
    environment:
      - NODE_ENV=development
      - DEBUG=user-service:*
    command: npm run dev
    ports:
      - "9229:9229"    # Node.js debugger port

  order-service:
    build:
      context: ./services/order-service
      target: development
    volumes:
      - ./services/order-service/src:/app/src
    environment:
      - NODE_ENV=development
    command: npm run dev

  # Dev tools
  mailhog:
    image: mailhog/mailhog:latest
    ports:
      - "1025:1025"
      - "8025:8025"

  pgadmin:
    image: dpage/pgadmin4:latest
    environment:
      - PGADMIN_DEFAULT_EMAIL=dev@localhost.com
      - PGADMIN_DEFAULT_PASSWORD=devpassword
    ports:
      - "5050:80"
    depends_on:
      - postgres
```

### 7.2 Run Different Environments

```bash
# Development
docker-compose -f docker-compose.yml -f docker-compose.dev.yml up

# Production
docker-compose up -d

# With monitoring
docker-compose --profile monitoring up -d

# Shortcut using COMPOSE_FILE
# .env
COMPOSE_FILE=docker-compose.yml:docker-compose.dev.yml

docker-compose up  # Auto picks up files from COMPOSE_FILE
```

---

## 8. Monitoring Container Status

### 8.1 Script ตรวจสอบสถานะ

```bash
#!/bin/bash
# check-services.sh - ตรวจสอบสถานะทุก services

echo "🔍 Checking Microservices Status..."
echo "=================================="

services=(
  "user-service:3001"
  "order-service:3002"
  "product-service:3003"
  "payment-service:3004"
)

all_healthy=true

for service in "${services[@]}"; do
  name="${service%%:*}"
  port="${service##*:}"
  
  # Check HTTP health
  response=$(curl -s -o /dev/null -w "%{http_code}" \
    http://localhost:${port}/health 2>/dev/null)
  
  if [ "$response" = "200" ]; then
    echo "✅ ${name} (port ${port}): Healthy"
  else
    echo "❌ ${name} (port ${port}): Unhealthy (HTTP ${response})"
    all_healthy=false
  fi
done

echo ""
echo "🐳 Docker Container Status:"
docker-compose ps --format "table {{.Name}}\t{{.Status}}\t{{.Ports}}"

if $all_healthy; then
  echo ""
  echo "✅ All services are healthy!"
else
  echo ""
  echo "⚠️  Some services need attention!"
  exit 1
fi
```

---

## 9. สรุป

### Docker Compose Best Practices

```
Configuration:
✅ ใช้ named volumes (ไม่ใช่ anonymous)
✅ Custom networks สำหรับ isolation
✅ Health checks ทุก service สำคัญ
✅ depends_on with health condition
✅ Resource limits

Security:
✅ Secrets ผ่าน .env file (ไม่ commit)
✅ ไม่ expose ports โดยไม่จำเป็น (internal services)
✅ Logging limits

Maintainability:
✅ แยก base / dev / prod compose files
✅ ใช้ Profiles สำหรับ optional services
✅ Comments ใน YAML
✅ Makefile สำหรับ common commands
```

---

**ต่อไป:** [Part 07 - Building First Microservice with Node.js →](part-07.md)

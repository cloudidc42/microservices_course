# Part 04: Setting Up Development Environment
## ติดตั้งและตั้งค่า Environment สำหรับการพัฒนา Microservices

> **ระดับ:** ⭐ เริ่มต้น | **เวลาเรียน:** 2-3 ชั่วโมง | **Prerequisites:** Parts 01-03

---

## 🎯 สิ่งที่จะได้เรียนรู้

- ติดตั้งเครื่องมือทั้งหมดที่จำเป็น
- ตั้งค่า VS Code สำหรับ Microservices Development
- ตั้งค่า Git และ GitHub
- สร้างโครงสร้าง Project แรก
- Script อัตโนมัติสำหรับ Setup

---

## 1. เครื่องมือที่ต้องการ

### 1.1 รายการเครื่องมือ

| เครื่องมือ | เวอร์ชันแนะนำ | ใช้ทำอะไร |
|-----------|------------|---------|
| Node.js | 18 LTS+ | Runtime สำหรับ JavaScript services |
| npm | 9+ | Package manager |
| Docker | 24+ | Container runtime |
| Docker Compose | 2.20+ | Multi-container management |
| VS Code | Latest | Code editor |
| Git | 2.40+ | Version control |
| curl / httpie | Latest | API testing |
| Postman / Insomnia | Latest | API testing GUI |
| k9s | Latest | Kubernetes TUI (optional) |
| kubectl | 1.28+ | Kubernetes CLI |

---

## 2. Installation Guide

### 2.1 Linux (Ubuntu/Debian)

```bash
#!/bin/bash
# setup-dev.sh - Complete development setup script

set -e  # Exit on error

echo "🚀 Starting Microservices Development Setup..."

# ─── Update System ───
echo "📦 Updating system packages..."
sudo apt-get update -y
sudo apt-get upgrade -y

# ─── Install Node.js (via nvm) ───
echo "📦 Installing Node.js via NVM..."
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash

# Load nvm
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"

nvm install 18
nvm use 18
nvm alias default 18

echo "Node.js version: $(node --version)"
echo "npm version: $(npm --version)"

# ─── Install Docker ───
echo "🐳 Installing Docker..."
sudo apt-get install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

# Add Docker's official GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Add Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update -y
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Add user to docker group (no sudo required)
sudo usermod -aG docker $USER

echo "Docker version: $(docker --version)"
echo "Docker Compose version: $(docker compose version)"

# ─── Install Git ───
echo "📝 Installing Git..."
sudo apt-get install -y git

git config --global core.editor "code --wait"
git config --global init.defaultBranch main

echo "Git version: $(git --version)"

# ─── Install httpie ───
echo "🌐 Installing httpie..."
pip3 install httpie

# ─── Install kubectl ───
echo "☸️  Installing kubectl..."
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
rm kubectl

echo "kubectl version: $(kubectl version --client)"

# ─── Install VS Code ───
echo "💻 Installing VS Code..."
sudo snap install --classic code

echo ""
echo "✅ Setup Complete!"
echo "⚠️  Please logout and login again to apply Docker group changes"
```

### 2.2 macOS

```bash
#!/bin/bash
# setup-mac.sh

echo "🚀 Setting up macOS for Microservices Development..."

# Install Homebrew
if ! command -v brew &> /dev/null; then
    echo "📦 Installing Homebrew..."
    /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
fi

# Install Node.js via nvm
echo "📦 Installing NVM and Node.js..."
brew install nvm

mkdir -p ~/.nvm
echo 'export NVM_DIR="$HOME/.nvm"' >> ~/.zshrc
echo '[ -s "/opt/homebrew/opt/nvm/nvm.sh" ] && \. "/opt/homebrew/opt/nvm/nvm.sh"' >> ~/.zshrc
source ~/.zshrc

nvm install 18
nvm use 18
nvm alias default 18

# Install Docker Desktop (manual installation recommended)
echo "🐳 Please install Docker Desktop from: https://docs.docker.com/desktop/mac/"
echo "   Or install via: brew install --cask docker"
brew install --cask docker

# Install useful tools
echo "🛠️  Installing development tools..."
brew install git
brew install httpie
brew install kubectl
brew install helm
brew install k9s
brew install jq

# Install VS Code
echo "💻 Installing VS Code..."
brew install --cask visual-studio-code

echo ""
echo "✅ Setup Complete!"
```

### 2.3 Windows (WSL2)

```powershell
# Run in PowerShell as Administrator
# setup-windows.ps1

Write-Host "🚀 Setting up Windows for Microservices Development..." -ForegroundColor Green

# Enable WSL2
Write-Host "📦 Enabling WSL2..."
wsl --install
wsl --set-default-version 2

# Install Ubuntu in WSL2
Write-Host "📦 Installing Ubuntu..."
winget install -e --id Canonical.Ubuntu.2204

# Install Docker Desktop
Write-Host "🐳 Installing Docker Desktop..."
winget install -e --id Docker.DockerDesktop

# Install VS Code
Write-Host "💻 Installing VS Code..."
winget install -e --id Microsoft.VisualStudioCode

# Install Git
Write-Host "📝 Installing Git..."
winget install -e --id Git.Git

Write-Host ""
Write-Host "✅ Base setup complete! Now open Ubuntu WSL2 and run the Linux setup script" -ForegroundColor Green
Write-Host "   1. Open Ubuntu from Start Menu"
Write-Host "   2. Run: curl -fsSL https://raw.githubusercontent.com/your/setup/main/setup-dev.sh | bash"
```

---

## 3. ตั้งค่า VS Code

### 3.1 Extensions ที่จำเป็น

```json
// extensions.json (สร้างใน .vscode/ folder)
{
  "recommendations": [
    // JavaScript/Node.js
    "dbaeumer.vscode-eslint",
    "esbenp.prettier-vscode",
    "christian-kohler.npm-intellisense",
    
    // Docker
    "ms-azuretools.vscode-docker",
    
    // Kubernetes
    "ms-kubernetes-tools.vscode-kubernetes-tools",
    
    // REST API Testing
    "humao.rest-client",
    
    // Git
    "eamodio.gitlens",
    "mhutchie.git-graph",
    
    // Productivity
    "pkief.material-icon-theme",
    "antfu.theme-vitesse",
    "ms-vscode.remote-containers",
    "ms-vscode-remote.remote-wsl",
    
    // Code Quality
    "streetsidesoftware.code-spell-checker",
    "sonarsource.sonarlint-vscode",
    
    // YAML
    "redhat.vscode-yaml",
    
    // JSON
    "zainchen.json",
    
    // Markdown
    "yzhang.markdown-all-in-one"
  ]
}
```

### 3.2 VS Code Settings

```json
// .vscode/settings.json
{
  // Editor
  "editor.tabSize": 2,
  "editor.insertSpaces": true,
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": true
  },
  
  // Files
  "files.trimTrailingWhitespace": true,
  "files.insertFinalNewline": true,
  "files.exclude": {
    "**/node_modules": true,
    "**/.git": true,
    "**/dist": true,
    "**/coverage": true
  },
  
  // Terminal
  "terminal.integrated.defaultProfile.linux": "bash",
  "terminal.integrated.fontSize": 13,
  
  // ESLint
  "eslint.validate": ["javascript", "javascriptreact", "typescript", "typescriptreact"],
  
  // Docker
  "docker.showStartPage": false,
  
  // Kubernetes
  "vs-kubernetes.vs-kubernetes.namespace": "default",
  
  // REST Client
  "rest-client.environmentVariables": {
    "local": {
      "userServiceUrl": "http://localhost:3001",
      "orderServiceUrl": "http://localhost:3002",
      "apiGatewayUrl": "http://localhost:8080"
    },
    "staging": {
      "userServiceUrl": "https://api-staging.company.com/users",
      "orderServiceUrl": "https://api-staging.company.com/orders"
    }
  }
}
```

### 3.3 REST Client สำหรับ API Testing

```http
### tests/api-tests.http

# User Service Tests

### Get all users
GET {{userServiceUrl}}/users
Content-Type: application/json

###

### Get user by ID
GET {{userServiceUrl}}/users/1

###

### Create new user
POST {{userServiceUrl}}/users
Content-Type: application/json

{
  "name": "ทดสอบ ระบบ",
  "email": "test@example.com",
  "phone": "0812345678"
}

###

# Order Service Tests

### Create order
POST {{orderServiceUrl}}/orders
Content-Type: application/json

{
  "userId": 1,
  "items": [
    {"productId": 1, "quantity": 2, "price": 500}
  ]
}

###

### Get orders by user
GET {{orderServiceUrl}}/orders?userId=1
```

---

## 4. Git Setup และ Best Practices

### 4.1 Initial Git Configuration

```bash
# ตั้งค่า Git global
git config --global user.name "ชื่อ นามสกุล"
git config --global user.email "email@example.com"
git config --global core.autocrlf input   # สำคัญสำหรับ Linux/Mac
git config --global pull.rebase false
git config --global init.defaultBranch main

# ตั้งค่า alias ที่มีประโยชน์
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.cm "commit -m"
git config --global alias.lg "log --oneline --graph --decorate --all"
git config --global alias.unstage "reset HEAD --"
```

### 4.2 .gitignore สำหรับ Node.js Microservices

```gitignore
# .gitignore

# Dependencies
node_modules/
.pnp
.pnp.js

# Build outputs
dist/
build/
coverage/
.nyc_output/

# Environment variables (NEVER commit these)
.env
.env.local
.env.development.local
.env.test.local
.env.production.local

# Log files
*.log
logs/
npm-debug.log*
yarn-debug.log*
yarn-error.log*

# OS files
.DS_Store
.DS_Store?
._*
Thumbs.db

# IDE files
.vscode/
.idea/
*.swp
*.swo

# Docker
docker-compose.override.yml

# Test artifacts
jest.config.js.old

# Misc
*.pid
*.seed
*.pid.lock
```

### 4.3 Git Branching Strategy

```bash
# Git Flow สำหรับ Microservices

# Main branches
main           # Production code
develop        # Integration branch

# Feature branches (ต่อ feature)
feature/user-authentication
feature/payment-gateway
feature/order-notifications

# Release branches
release/1.2.0

# Hotfix branches
hotfix/payment-bug-fix

# Workflow
git checkout develop
git pull origin develop
git checkout -b feature/user-authentication

# ... ทำงาน ...
git add .
git commit -m "feat(auth): add JWT authentication"

git push origin feature/user-authentication
# สร้าง Pull Request บน GitHub
```

### 4.4 Conventional Commits

```bash
# Format: <type>(<scope>): <description>

# Types:
feat      # New feature
fix       # Bug fix
docs      # Documentation
style     # Formatting
refactor  # Code refactoring (not a fix or feature)
test      # Adding tests
chore     # Maintenance
perf      # Performance improvement
ci        # CI/CD changes
build     # Build system changes

# ตัวอย่าง
git commit -m "feat(users): add email verification"
git commit -m "fix(orders): handle payment timeout correctly"
git commit -m "docs(api): update OpenAPI spec for user endpoints"
git commit -m "refactor(auth): extract token validation to middleware"
git commit -m "test(payment): add integration tests for Stripe webhook"
git commit -m "chore(deps): upgrade express to 4.18.2"
```

---

## 5. โครงสร้าง Project สำหรับ Microservices

### 5.1 Repository Structure Options

**Option 1: Polyrepo (แยก Repo ต่อ Service)**
```
github.com/company/
├── user-service/
├── order-service/
├── payment-service/
├── notification-service/
├── api-gateway/
└── infrastructure/

ข้อดี: แต่ละ service มี lifecycle อิสระ
ข้อเสีย: ยากต่อการ shared code และ cross-service changes
```

**Option 2: Monorepo (ทุก Service ใน Repo เดียว)**
```
github.com/company/backend/
├── services/
│   ├── user-service/
│   ├── order-service/
│   └── payment-service/
├── packages/
│   ├── shared-types/      # TypeScript types
│   ├── common-utils/      # Utilities
│   └── test-helpers/      # Test utilities
├── infrastructure/
│   ├── kubernetes/
│   └── terraform/
└── scripts/

ข้อดี: ง่ายต่อ shared code, atomic changes
ข้อเสีย: ต้องการ tooling (Nx, Turborepo)
```

### 5.2 Service Directory Structure

```
user-service/
├── src/
│   ├── app.js                  # Express app setup
│   ├── server.js               # Server startup & graceful shutdown
│   ├── config/
│   │   └── index.js            # All configuration
│   ├── routes/
│   │   ├── index.js            # Route aggregator
│   │   ├── users.routes.js     # User routes
│   │   └── health.routes.js    # Health check routes
│   ├── controllers/
│   │   └── users.controller.js # Request/Response handling
│   ├── services/
│   │   └── users.service.js    # Business logic
│   ├── repositories/
│   │   └── users.repository.js # Database operations
│   ├── models/
│   │   └── user.model.js       # Data model/schema
│   ├── middleware/
│   │   ├── auth.middleware.js   # Authentication
│   │   ├── validate.middleware.js # Request validation
│   │   └── error.middleware.js  # Error handling
│   └── utils/
│       ├── logger.js           # Logging utility
│       └── validators.js       # Validation schemas
├── tests/
│   ├── unit/
│   │   └── users.service.test.js
│   └── integration/
│       └── users.api.test.js
├── database/
│   ├── migrations/
│   │   └── 001_create_users.sql
│   └── seeds/
│       └── users.seed.js
├── .env.example                # Environment variables template
├── .gitignore
├── Dockerfile
├── docker-compose.yml          # For local development
├── package.json
└── README.md
```

### 5.3 Template: app.js

```javascript
// src/app.js
const express = require('express');
const cors = require('cors');
const helmet = require('helmet');
const compression = require('compression');
const rateLimit = require('express-rate-limit');

const logger = require('./utils/logger');
const config = require('./config');
const routes = require('./routes');
const { errorHandler, notFoundHandler } = require('./middleware/error.middleware');

function createApp() {
  const app = express();
  
  // ── Security Middleware ──
  app.use(helmet());
  app.use(cors({
    origin: config.app.allowedOrigins,
    credentials: true,
  }));
  
  // ── Rate Limiting ──
  const limiter = rateLimit({
    windowMs: 15 * 60 * 1000,  // 15 minutes
    max: 100,                    // max 100 requests per window
    message: { error: 'Too many requests, please try again later' }
  });
  app.use('/api/', limiter);
  
  // ── Body Parsing ──
  app.use(express.json({ limit: '10mb' }));
  app.use(express.urlencoded({ extended: true }));
  
  // ── Compression ──
  app.use(compression());
  
  // ── Request Logging ──
  app.use(logger.requestMiddleware);
  
  // ── Routes ──
  app.use('/api', routes);
  
  // ── Error Handling ──
  app.use(notFoundHandler);
  app.use(errorHandler);
  
  return app;
}

module.exports = createApp();
```

### 5.4 Template: Error Middleware

```javascript
// src/middleware/error.middleware.js

const logger = require('../utils/logger');

// ไม่พบ Route
function notFoundHandler(req, res, next) {
  res.status(404).json({
    success: false,
    error: 'Not Found',
    message: `Route ${req.method} ${req.url} not found`,
    timestamp: new Date().toISOString(),
  });
}

// Custom Error Classes
class AppError extends Error {
  constructor(message, statusCode = 500, code = 'INTERNAL_ERROR') {
    super(message);
    this.statusCode = statusCode;
    this.code = code;
    this.isOperational = true;
  }
}

class ValidationError extends AppError {
  constructor(message, details) {
    super(message, 400, 'VALIDATION_ERROR');
    this.details = details;
  }
}

class NotFoundError extends AppError {
  constructor(resource) {
    super(`${resource} not found`, 404, 'NOT_FOUND');
  }
}

class UnauthorizedError extends AppError {
  constructor(message = 'Unauthorized') {
    super(message, 401, 'UNAUTHORIZED');
  }
}

// Global Error Handler
function errorHandler(err, req, res, next) {
  // Log error
  if (!err.isOperational) {
    logger.error('Unexpected error', {
      error: err.message,
      stack: err.stack,
      url: req.url,
      method: req.method,
    });
  }
  
  const statusCode = err.statusCode || 500;
  const response = {
    success: false,
    error: err.code || 'INTERNAL_ERROR',
    message: err.message || 'An unexpected error occurred',
    timestamp: new Date().toISOString(),
    requestId: req.headers['x-request-id'],
  };
  
  // Include validation details if available
  if (err.details) {
    response.details = err.details;
  }
  
  // Don't expose stack trace in production
  if (process.env.NODE_ENV === 'development') {
    response.stack = err.stack;
  }
  
  res.status(statusCode).json(response);
}

module.exports = {
  notFoundHandler,
  errorHandler,
  AppError,
  ValidationError,
  NotFoundError,
  UnauthorizedError,
};
```

---

## 6. Development Workflow

### 6.1 Daily Development Commands

```bash
# เริ่ม Development Environment
docker-compose up -d

# ดู logs ของ specific service
docker-compose logs -f user-service

# Restart specific service (หลัง code change)
docker-compose restart user-service

# หยุดทุก services
docker-compose down

# หยุดและลบ volumes (reset database)
docker-compose down -v

# ดู status ของ containers
docker-compose ps

# exec command ใน container
docker-compose exec user-service sh

# Run tests ใน container
docker-compose exec user-service npm test
```

### 6.2 Hot Reload Setup

```yaml
# docker-compose.yml สำหรับ Development
version: '3.8'

services:
  user-service:
    build:
      context: ./user-service
      dockerfile: Dockerfile.dev
    ports:
      - "3001:3001"
    volumes:
      - ./user-service/src:/app/src   # Mount source code
      - /app/node_modules             # Exclude node_modules
    environment:
      - NODE_ENV=development
      - PORT=3001
    command: npm run dev              # nodemon สำหรับ hot reload
```

```dockerfile
# user-service/Dockerfile.dev
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install  # Include devDependencies

COPY . .

CMD ["npm", "run", "dev"]
```

```json
// package.json
{
  "scripts": {
    "start": "node src/server.js",
    "dev": "nodemon --watch src --ext js,json src/server.js",
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage",
    "lint": "eslint src/**/*.js",
    "lint:fix": "eslint src/**/*.js --fix"
  }
}
```

### 6.3 Makefile สำหรับ Common Tasks

```makefile
# Makefile
.PHONY: help up down logs build test clean

help:  ## Show this help
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | sort | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-20s\033[0m %s\n", $$1, $$2}'

up:  ## Start all services
	docker-compose up -d

down:  ## Stop all services
	docker-compose down

logs:  ## Show logs (usage: make logs s=user-service)
	docker-compose logs -f $(s)

build:  ## Build all images
	docker-compose build

test:  ## Run tests (usage: make test s=user-service)
	docker-compose exec $(s) npm test

clean:  ## Remove containers and volumes
	docker-compose down -v --remove-orphans

install:  ## Install dependencies in all services
	for dir in */; do \
		if [ -f "$$dir/package.json" ]; then \
			echo "Installing in $$dir"; \
			(cd "$$dir" && npm install); \
		fi \
	done

lint:  ## Lint all services
	for dir in */; do \
		if [ -f "$$dir/package.json" ]; then \
			echo "Linting $$dir"; \
			(cd "$$dir" && npm run lint) || true; \
		fi \
	done

# Example: make up, make logs s=user-service, make test s=order-service
```

---

## 7. Database Tools Setup

### 7.1 PostgreSQL Client

```bash
# ติดตั้ง pgAdmin (GUI)
# Ubuntu
curl -fsSL https://www.pgadmin.org/static/packages_pgadmin_org.pub | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/pgadmin.gpg
sudo sh -c 'echo "deb https://ftp.postgresql.org/pub/pgadmin/pgadmin4/apt/$(lsb_release -cs) pgadmin4 main" > /etc/apt/sources.list.d/pgadmin4.list'
sudo apt-get update
sudo apt-get install -y pgadmin4

# หรือใช้ DBeaver (รองรับทุก Database)
sudo snap install dbeaver-ce
```

```bash
# ใช้ psql ผ่าน Docker (ไม่ต้องติดตั้ง)
docker run --rm -it \
  --network host \
  postgres:15 \
  psql -h localhost -U postgres -d userdb
```

### 7.2 Redis Client

```bash
# Redis Desktop Manager
sudo snap install redis-desktop-manager

# หรือใช้ redis-cli ผ่าน Docker
docker run --rm -it \
  --network host \
  redis:7-alpine \
  redis-cli -h localhost

# ใน redis-cli
PING                    # Test connection
SET key "value"         # Set value
GET key                 # Get value
KEYS *                  # List all keys
TTL key                 # Check TTL
FLUSHALL               # Clear all data (dev only!)
```

### 7.3 MongoDB Client

```bash
# MongoDB Compass (GUI)
# Download from: https://www.mongodb.com/try/download/compass

# หรือใช้ mongosh ผ่าน Docker
docker run --rm -it \
  --network host \
  mongo:6 \
  mongosh mongodb://localhost:27017

# ใน mongosh
show dbs                              # List databases
use mydb                              # Switch database
db.users.find()                       # Query collection
db.users.insertOne({name: "test"})    # Insert document
```

---

## 8. Verify Everything Works

### 8.1 Verification Script

```bash
#!/bin/bash
# verify-setup.sh

echo "🔍 Verifying Development Environment..."
echo ""

check() {
  if command -v $1 &> /dev/null; then
    echo "✅ $1: $($1 --version 2>&1 | head -1)"
  else
    echo "❌ $1: Not found!"
  fi
}

check node
check npm
check docker
check git
check kubectl

# Check Docker
if docker ps &> /dev/null; then
  echo "✅ Docker daemon: Running"
else
  echo "❌ Docker daemon: Not running"
fi

echo ""
echo "📊 System Resources:"
echo "   CPU: $(nproc) cores"
echo "   RAM: $(free -h | awk '/^Mem:/ {print $2}') total"
echo "   Disk: $(df -h / | awk 'NR==2 {print $4}') available"

echo ""
echo "🔧 Verification complete!"
```

### 8.2 Test ว่าทุกอย่างทำงาน

```bash
# 1. Test Node.js
node -e "console.log('Node.js works!')"

# 2. Test Docker
docker run --rm hello-world

# 3. Test Docker Compose
cat > /tmp/test-compose.yml << 'EOF'
version: '3.8'
services:
  test:
    image: nginx:alpine
    ports:
      - "8888:80"
EOF

docker-compose -f /tmp/test-compose.yml up -d
curl http://localhost:8888
docker-compose -f /tmp/test-compose.yml down
rm /tmp/test-compose.yml

echo "✅ All tools working!"
```

---

## 9. สรุปและ Next Steps

### Checklist

- [ ] ติดตั้ง Node.js 18+ ✅
- [ ] ติดตั้ง Docker และ Docker Compose ✅
- [ ] ตั้งค่า VS Code พร้อม Extensions ✅
- [ ] ตั้งค่า Git ✅
- [ ] สร้าง Directory Structure ✅
- [ ] ทดสอบว่าทุกอย่างทำงาน ✅

### เริ่มต้น Project แรก

```bash
# Clone template project
git clone https://github.com/course/microservices-template
cd microservices-template

# ดู Structure
ls -la

# Start services
docker-compose up --build

# Test APIs
curl http://localhost:3001/health
curl http://localhost:3002/health
```

---

**ต่อไป:** [Part 05 - Docker Fundamentals สำหรับ Microservices →](part-05.md)

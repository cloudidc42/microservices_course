# Part 21: Security Hardening - OWASP Top 10 & Production Security

## ภาพรวม

ใน Part นี้เราจะเรียนรู้การ hardening ระบบ Microservices ให้ปลอดภัย:
- **OWASP Top 10** สำหรับ APIs
- **Input Validation & Sanitization**
- **SQL/NoSQL Injection Prevention**
- **Rate Limiting & DDoS Protection**
- **Security Headers**
- **Secrets Management** ด้วย HashiCorp Vault
- **Container Security** Best Practices

---

## 1. OWASP API Security Top 10

```
OWASP API Security Top 10 (2023):
1.  Broken Object Level Authorization (BOLA)
2.  Broken Authentication
3.  Broken Object Property Level Authorization
4.  Unrestricted Resource Consumption
5.  Broken Function Level Authorization
6.  Unrestricted Access to Sensitive Business Flows
7.  Server Side Request Forgery (SSRF)
8.  Security Misconfiguration
9.  Improper Inventory Management
10. Unsafe Consumption of APIs
```

---

## 2. Input Validation ด้วย Zod

### 2.1 Comprehensive Validation Schemas

```javascript
// validation/schemas.js
const { z } = require('zod');

// ป้องกัน common injection patterns
const safeString = z.string()
  .min(1)
  .max(1000)
  .refine(
    val => !/<script|javascript:|vbscript:|on\w+\s*=/i.test(val),
    'Contains potentially dangerous content'
  );

const safeEmail = z.string().email().max(254).toLowerCase();

const phoneNumber = z.string()
  .regex(/^\+?[1-9]\d{1,14}$/, 'Invalid phone number format');

const thaiId = z.string()
  .length(13)
  .regex(/^\d{13}$/, 'Thai ID must be 13 digits')
  .refine(val => validateThaiIdChecksum(val), 'Invalid Thai ID checksum');

// User registration schema
const createUserSchema = z.object({
  email: safeEmail,
  password: z.string()
    .min(8)
    .max(128)
    .regex(/[A-Z]/, 'Must contain uppercase letter')
    .regex(/[a-z]/, 'Must contain lowercase letter')
    .regex(/[0-9]/, 'Must contain number')
    .regex(/[^A-Za-z0-9]/, 'Must contain special character'),
  name: safeString.min(2).max(100),
  phone: phoneNumber.optional(),
  dateOfBirth: z.string().datetime().optional()
    .refine(val => {
      if (!val) return true;
      const age = calculateAge(new Date(val));
      return age >= 18;
    }, 'Must be 18 or older'),
  address: z.object({
    street: safeString.max(200),
    city: safeString.max(100),
    postalCode: z.string().regex(/^\d{5}$/, 'Invalid postal code'),
    country: z.string().length(2)  // ISO 3166-1 alpha-2
  }).optional()
});

// Query parameters schema (prevent injection in GET params)
const getUsersQuerySchema = z.object({
  search: safeString.max(100).optional(),
  status: z.enum(['active', 'inactive', 'suspended']).optional(),
  page: z.coerce.number().int().positive().default(1),
  limit: z.coerce.number().int().min(1).max(100).default(20),
  sortBy: z.enum(['name', 'email', 'createdAt']).default('createdAt'),
  sortOrder: z.enum(['asc', 'desc']).default('desc')
});

// UUID validation
const uuidSchema = z.string().uuid('Invalid ID format');

// Monetary amount (avoid floating point issues)
const monetaryAmount = z.object({
  amount: z.number().int().positive(),  // Store in cents
  currency: z.string().length(3).toUpperCase()  // ISO 4217
});

function validateThaiIdChecksum(id) {
  const digits = id.split('').map(Number);
  const sum = digits.slice(0, 12).reduce((acc, d, i) => acc + d * (13 - i), 0);
  const checkDigit = (11 - (sum % 11)) % 10;
  return checkDigit === digits[12];
}

module.exports = {
  createUserSchema,
  getUsersQuerySchema,
  uuidSchema,
  monetaryAmount,
  safeString,
  safeEmail
};
```

### 2.2 Validation Middleware

```javascript
// middleware/validate.js
const { ZodError } = require('zod');

function validate(schema, target = 'body') {
  return (req, res, next) => {
    try {
      const data = req[target];
      const validated = schema.parse(data);
      
      // Replace with validated (sanitized) data
      req[target] = validated;
      next();
    } catch (err) {
      if (err instanceof ZodError) {
        const errors = err.errors.map(e => ({
          field: e.path.join('.'),
          message: e.message,
          code: e.code
        }));
        
        return res.status(400).json({
          error: 'Validation failed',
          errors
        });
      }
      next(err);
    }
  };
}

// ใช้งาน
router.post('/users',
  validate(createUserSchema, 'body'),
  createUserController
);

router.get('/users',
  validate(getUsersQuerySchema, 'query'),
  getUsersController
);

router.get('/users/:id',
  validate(z.object({ id: uuidSchema }), 'params'),
  getUserController
);
```

---

## 3. SQL Injection Prevention

### 3.1 Parameterized Queries เท่านั้น

```javascript
// db/safe-queries.js

// ❌ NEVER DO THIS - SQL Injection vulnerability
async function getUserByEmail_VULNERABLE(email) {
  return db.query(`SELECT * FROM users WHERE email = '${email}'`);
  // email = "'; DROP TABLE users; --" -> DISASTER!
}

// ✅ Always use parameterized queries
async function getUserByEmail(email) {
  return db.query(
    'SELECT * FROM users WHERE email = $1',
    [email]
  );
}

// ✅ Dynamic ORDER BY (safe)
async function getUsers(sortBy, sortOrder) {
  // Whitelist allowed columns
  const ALLOWED_SORT_COLUMNS = {
    name: 'u.name',
    email: 'u.email',
    createdAt: 'u.created_at'
  };
  
  const column = ALLOWED_SORT_COLUMNS[sortBy] || 'u.created_at';
  const order = sortOrder === 'asc' ? 'ASC' : 'DESC';
  
  // Column names cannot be parameterized, but we whitelist them
  return db.query(
    `SELECT * FROM users u ORDER BY ${column} ${order} LIMIT $1 OFFSET $2`,
    [limit, offset]
  );
}

// ✅ Dynamic filter (safe builder)
class SafeQueryBuilder {
  constructor(table) {
    this.table = table;
    this.conditions = [];
    this.params = [];
    this.paramCount = 0;
  }

  addCondition(column, operator, value) {
    const ALLOWED_OPERATORS = ['=', '!=', '<', '>', '<=', '>=', 'LIKE', 'IN', 'IS NULL', 'IS NOT NULL'];
    
    if (!ALLOWED_OPERATORS.includes(operator.toUpperCase())) {
      throw new Error(`Invalid operator: ${operator}`);
    }
    
    if (operator.toUpperCase() === 'IN') {
      const placeholders = value.map(() => `$${++this.paramCount}`).join(', ');
      this.conditions.push(`${column} IN (${placeholders})`);
      this.params.push(...value);
    } else if (operator.toUpperCase() === 'IS NULL') {
      this.conditions.push(`${column} IS NULL`);
    } else {
      this.paramCount++;
      this.conditions.push(`${column} ${operator} $${this.paramCount}`);
      this.params.push(value);
    }
    
    return this;
  }

  build(select = '*') {
    const whereClause = this.conditions.length > 0
      ? `WHERE ${this.conditions.join(' AND ')}`
      : '';
    
    return {
      query: `SELECT ${select} FROM ${this.table} ${whereClause}`,
      params: this.params
    };
  }
}

// ใช้งาน
const builder = new SafeQueryBuilder('users');
builder
  .addCondition('status', '=', 'active')
  .addCondition('email', 'LIKE', '%@company.com')
  .addCondition('role', 'IN', ['admin', 'manager']);

const { query, params } = builder.build('id, name, email');
const result = await db.query(query, params);
```

---

## 4. Authorization: BOLA Prevention

### 4.1 Resource Ownership Verification

```javascript
// middleware/ownership.js

// BOLA (Broken Object Level Authorization)
// ตรวจสอบว่า user เป็นเจ้าของ resource นั้นจริงๆ

function requireOwnership(resourceType, options = {}) {
  return async (req, res, next) => {
    const resourceId = req.params.id || req.params[`${resourceType}Id`];
    const userId = req.user.id;
    const userRole = req.user.role;

    // Admin/superadmin สามารถเข้าถึงทุก resource
    if (options.adminBypass && ['admin', 'superadmin'].includes(userRole)) {
      return next();
    }

    try {
      const isOwner = await checkOwnership(resourceType, resourceId, userId);
      
      if (!isOwner) {
        // Return 404 แทน 403 เพื่อป้องกัน enumeration
        return res.status(404).json({ error: 'Resource not found' });
      }
      
      next();
    } catch (err) {
      next(err);
    }
  };
}

async function checkOwnership(resourceType, resourceId, userId) {
  const OWNERSHIP_QUERIES = {
    order: 'SELECT 1 FROM orders WHERE id = $1 AND user_id = $2 AND deleted_at IS NULL',
    address: 'SELECT 1 FROM addresses WHERE id = $1 AND user_id = $2',
    payment_method: 'SELECT 1 FROM payment_methods WHERE id = $1 AND user_id = $2'
  };

  const query = OWNERSHIP_QUERIES[resourceType];
  if (!query) throw new Error(`Unknown resource type: ${resourceType}`);
  
  const result = await db.query(query, [resourceId, userId]);
  return result.rows.length > 0;
}

// ใช้งาน
router.get('/orders/:id',
  authenticate,
  requireOwnership('order', { adminBypass: true }),
  getOrderController
);

router.put('/orders/:id',
  authenticate,
  requireOwnership('order'),
  updateOrderController
);
```

---

## 5. Security Headers

### 5.1 Helmet.js Configuration

```javascript
// middleware/security-headers.js
const helmet = require('helmet');
const crypto = require('crypto');

function securityHeaders() {
  return [
    // Generate nonce สำหรับ CSP
    (req, res, next) => {
      res.locals.nonce = crypto.randomBytes(16).toString('base64');
      next();
    },

    helmet({
      // Content Security Policy
      contentSecurityPolicy: {
        directives: {
          defaultSrc: ["'self'"],
          scriptSrc: [
            "'self'",
            (req, res) => `'nonce-${res.locals.nonce}'`  // Dynamic nonce
          ],
          styleSrc: ["'self'", "'unsafe-inline'"],
          imgSrc: ["'self'", 'data:', 'https://cdn.company.com'],
          connectSrc: ["'self'", 'https://api.company.com'],
          fontSrc: ["'self'", 'https://fonts.gstatic.com'],
          objectSrc: ["'none'"],
          mediaSrc: ["'none'"],
          frameSrc: ["'none'"],
          upgradeInsecureRequests: []
        },
        reportOnly: false
      },

      // HTTP Strict Transport Security
      hsts: {
        maxAge: 31536000,      // 1 year
        includeSubDomains: true,
        preload: true
      },

      // Prevent clickjacking
      frameguard: { action: 'deny' },

      // Prevent MIME type sniffing
      noSniff: true,

      // Referrer Policy
      referrerPolicy: {
        policy: 'strict-origin-when-cross-origin'
      },

      // Permissions Policy
      permissionsPolicy: {
        camera: [],
        microphone: [],
        geolocation: [],
        payment: ["'self'"]
      },

      // Cross-Origin Embedder Policy
      crossOriginEmbedderPolicy: false,

      // Hide X-Powered-By
      hidePoweredBy: true
    }),

    // Custom headers
    (req, res, next) => {
      // Remove server info
      res.removeHeader('Server');
      res.removeHeader('X-Powered-By');
      
      // Cache control for APIs
      res.setHeader('Cache-Control', 'no-store, no-cache, must-revalidate');
      res.setHeader('Pragma', 'no-cache');
      
      // X-Request-ID for tracing
      if (!req.headers['x-request-id']) {
        res.setHeader('X-Request-Id', crypto.randomUUID());
      }
      
      next();
    }
  ];
}

module.exports = securityHeaders;
```

---

## 6. Rate Limiting Advanced

### 6.1 Adaptive Rate Limiting

```javascript
// middleware/adaptive-rate-limit.js
const Redis = require('ioredis');
const redis = new Redis();

class AdaptiveRateLimiter {
  constructor(options = {}) {
    this.config = {
      // Base limits
      anonymous: {
        requests: options.anonymousLimit || 60,
        window: options.window || 60 // seconds
      },
      authenticated: {
        requests: options.authenticatedLimit || 300,
        window: options.window || 60
      },
      premium: {
        requests: options.premiumLimit || 1000,
        window: options.window || 60
      },
      // Endpoint-specific overrides
      endpoints: options.endpoints || {}
    };
  }

  middleware() {
    return async (req, res, next) => {
      const tier = this.getUserTier(req);
      const endpoint = this.normalizeEndpoint(req.path);
      
      // Get limit for this specific endpoint or use tier default
      const limit = this.config.endpoints[endpoint]?.[tier]
        || this.config[tier];

      const key = this.getRateLimitKey(req, tier);
      
      try {
        const result = await this.checkRateLimit(key, limit);
        
        // Set headers
        res.setHeader('X-RateLimit-Limit', limit.requests);
        res.setHeader('X-RateLimit-Remaining', Math.max(0, result.remaining));
        res.setHeader('X-RateLimit-Reset', result.resetTime);
        res.setHeader('X-RateLimit-Policy', `${limit.requests};w=${limit.window}`);
        
        if (!result.allowed) {
          res.setHeader('Retry-After', result.retryAfter);
          return res.status(429).json({
            error: 'Too Many Requests',
            message: `Rate limit exceeded. Retry after ${result.retryAfter} seconds.`,
            retryAfter: result.retryAfter
          });
        }
        
        next();
      } catch (err) {
        // Fail open (allow request) if Redis is down
        console.error('Rate limiter error:', err);
        next();
      }
    };
  }

  async checkRateLimit(key, limit) {
    const windowKey = `rl:${key}:${Math.floor(Date.now() / 1000 / limit.window)}`;
    
    const pipeline = redis.pipeline();
    pipeline.incr(windowKey);
    pipeline.expire(windowKey, limit.window * 2);
    
    const [[, count]] = await pipeline.exec();
    
    const resetTime = Math.ceil(Date.now() / 1000 / limit.window) * limit.window;
    const retryAfter = resetTime - Math.floor(Date.now() / 1000);
    
    return {
      allowed: count <= limit.requests,
      count,
      remaining: limit.requests - count,
      resetTime,
      retryAfter
    };
  }

  getRateLimitKey(req, tier) {
    if (req.user) {
      return `user:${req.user.id}`;
    }
    // Use IP for anonymous users
    const ip = req.ip || req.connection.remoteAddress;
    return `ip:${ip}`;
  }

  getUserTier(req) {
    if (!req.user) return 'anonymous';
    if (req.user.tier === 'premium') return 'premium';
    return 'authenticated';
  }

  normalizeEndpoint(path) {
    // /users/123/orders -> /users/:id/orders
    return path.replace(/\/[0-9a-f-]{8,}/g, '/:id');
  }
}

// ใช้งาน
const rateLimiter = new AdaptiveRateLimiter({
  anonymousLimit: 30,
  authenticatedLimit: 200,
  premiumLimit: 1000,
  window: 60,
  endpoints: {
    '/auth/login': {
      anonymous: { requests: 5, window: 300 },    // 5 req per 5 min
      authenticated: { requests: 10, window: 300 }
    },
    '/auth/register': {
      anonymous: { requests: 3, window: 3600 }    // 3 per hour
    }
  }
});

app.use(rateLimiter.middleware());
```

---

## 7. Secrets Management ด้วย Vault

### 7.1 HashiCorp Vault Integration

```javascript
// secrets/vault-client.js
const vault = require('node-vault');

class VaultSecretManager {
  constructor(config = {}) {
    this.client = vault({
      apiVersion: 'v1',
      endpoint: config.endpoint || process.env.VAULT_ADDR,
      token: config.token || process.env.VAULT_TOKEN
    });
    
    this.secretsCache = new Map();
    this.cacheTimeout = config.cacheTimeout || 300000; // 5 minutes
  }

  async initialize() {
    // Authenticate using AppRole (for production)
    if (process.env.VAULT_ROLE_ID && process.env.VAULT_SECRET_ID) {
      const result = await this.client.approleLogin({
        role_id: process.env.VAULT_ROLE_ID,
        secret_id: process.env.VAULT_SECRET_ID
      });
      
      this.client = vault({
        apiVersion: 'v1',
        endpoint: process.env.VAULT_ADDR,
        token: result.auth.client_token
      });
      
      // Schedule token renewal
      const leaseDuration = result.auth.lease_duration;
      setTimeout(
        () => this.initialize(),
        (leaseDuration - 60) * 1000  // Renew 60s before expiry
      );
    }
  }

  async getSecret(path, key) {
    const cacheKey = `${path}:${key}`;
    const cached = this.secretsCache.get(cacheKey);
    
    if (cached && cached.expiresAt > Date.now()) {
      return cached.value;
    }
    
    const result = await this.client.read(`secret/data/${path}`);
    const value = result.data.data[key];
    
    if (!value) {
      throw new Error(`Secret not found: ${path}.${key}`);
    }
    
    this.secretsCache.set(cacheKey, {
      value,
      expiresAt: Date.now() + this.cacheTimeout
    });
    
    return value;
  }

  async getDynamicDBCredentials(role) {
    const result = await this.client.read(`database/creds/${role}`);
    
    return {
      username: result.data.username,
      password: result.data.password,
      leaseDuration: result.lease_duration,
      leaseId: result.lease_id
    };
  }

  async renewLease(leaseId) {
    return this.client.write('sys/leases/renew', {
      lease_id: leaseId
    });
  }
}

// ใช้งาน
const vaultManager = new VaultSecretManager();

async function getDatabaseConnection() {
  const creds = await vaultManager.getDynamicDBCredentials('user-service');
  
  const pool = new Pool({
    host: await vaultManager.getSecret('services/user-service', 'DB_HOST'),
    database: 'userdb',
    user: creds.username,
    password: creds.password
  });
  
  // Schedule credential renewal
  setTimeout(
    async () => {
      await vaultManager.renewLease(creds.leaseId);
    },
    (creds.leaseDuration - 60) * 1000
  );
  
  return pool;
}
```

---

## 8. Container Security

### 8.1 Secure Dockerfile

```dockerfile
# Secure Node.js Dockerfile

# ===== Build Stage =====
FROM node:20-alpine AS builder

# Security: don't run as root
WORKDIR /build

# Copy only dependency files first (better layer caching)
COPY package*.json ./

# Install with exact versions, no scripts
RUN npm ci --ignore-scripts --production=false

COPY . .
RUN npm run build

# ===== Production Stage =====
FROM node:20-alpine AS production

# Install security updates
RUN apk update && \
    apk upgrade && \
    apk add --no-cache dumb-init && \
    rm -rf /var/cache/apk/*

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S -u 1001 -G nodejs nodejs

WORKDIR /app

# Copy only production artifacts
COPY --from=builder --chown=nodejs:nodejs /build/node_modules ./node_modules
COPY --from=builder --chown=nodejs:nodejs /build/dist ./dist
COPY --from=builder --chown=nodejs:nodejs /build/package*.json ./

# Remove package-lock.json (sensitive info)
RUN rm -f package-lock.json

# Switch to non-root
USER nodejs

# Expose non-privileged port
EXPOSE 3000

# Use dumb-init to handle signals properly
ENTRYPOINT ["dumb-init", "--"]
CMD ["node", "dist/server.js"]

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1
```

### 8.2 Kubernetes Pod Security

```yaml
# pod-security-policy.yaml (Kubernetes 1.25+ uses Pod Security Standards)
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    # Enforce restricted pod security standard
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted

---
# Deployment with all security settings
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
spec:
  template:
    spec:
      automountServiceAccountToken: false  # Don't mount SA token unless needed
      
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
        runAsGroup: 1001
        fsGroup: 1001
        seccompProfile:
          type: RuntimeDefault
        supplementalGroups: []
      
      containers:
        - name: user-service
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL
              add: []  # Add only if absolutely needed
          
          # Run as specific user
          runAsUser: 1001
          runAsNonRoot: true
          
          # Resource limits (prevent DoS)
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          
          # Volume mounts for writable dirs
          volumeMounts:
            - name: tmp
              mountPath: /tmp
            - name: app-logs
              mountPath: /app/logs
      
      volumes:
        - name: tmp
          emptyDir: {}
        - name: app-logs
          emptyDir: {}
      
      # Don't schedule on same node as other critical workloads
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: user-service
```

---

## 9. Security Scanning ใน CI/CD

```yaml
# .github/workflows/security-scan.yml
name: Security Scan

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 0 * * 1'  # Weekly scan

jobs:
  dependency-audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Audit npm dependencies
        run: npm audit --audit-level=high
      
      - name: Check for known vulnerabilities
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}

  sast:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Semgrep SAST
        uses: semgrep/semgrep-action@v1
        with:
          config: >-
            p/javascript
            p/nodejs
            p/owasp-top-ten
          
      - name: SonarQube Scan
        uses: SonarSource/sonarcloud-github-action@master
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}

  container-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Build image for scanning
        run: docker build -t scan-target .
      
      - name: Trivy container scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: scan-target
          format: 'sarif'
          output: 'trivy-results.sarif'
          exit-code: '1'
          severity: 'CRITICAL,HIGH'
      
      - name: Grype container scan
        uses: anchore/scan-action@v3
        with:
          image: scan-target
          fail-build: true
          severity-cutoff: high

  secrets-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Full history for secret scanning
      
      - name: Detect hardcoded secrets
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  iac-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Scan IaC with Checkov
        uses: bridgecrewio/checkov-action@master
        with:
          directory: k8s/
          framework: kubernetes
          soft_fail: false
```

---

## สรุป Security Checklist

```
Security Checklist สำหรับ Production:

Authentication & Authorization:
☑ JWT ใช้ RS256 (ไม่ใช่ HS256)
☑ Token expiry สั้น (15-60 นาที)
☑ Refresh token rotation
☑ BOLA checks บน ทุก resource endpoint
☑ RBAC ตรวจสอบ permissions อย่างละเอียด

Input Validation:
☑ Validate ทุก input ด้วย schema
☑ Parameterized queries ทุกที่
☑ Rate limiting บน sensitive endpoints
☑ File upload validation (type, size, content)

Transport Security:
☑ HTTPS everywhere
☑ mTLS ระหว่าง services
☑ Security headers ครบ (Helmet.js)
☑ CORS configuration ถูกต้อง

Infrastructure:
☑ Non-root containers
☑ Read-only filesystem
☑ Secrets ใน Vault (ไม่ใน env vars/code)
☑ Network policies (least privilege)
☑ Pod Security Standards: restricted

Monitoring:
☑ Log authentication failures
☑ Alert on unusual patterns
☑ Dependency vulnerability scanning
☑ SAST/DAST ใน CI/CD
☑ Container image scanning
```

**Next:** Part 22 - Event Sourcing and CQRS Deep Dive

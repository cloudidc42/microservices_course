# Part 77: Advanced Security Patterns

## บทนำ

Security ใน Microservices มีความซับซ้อนมากกว่า Monolith เพราะมี attack surface หลายจุด บทนี้ครอบคลุม OWASP Top 10, Penetration Testing, Security Scanning, Container Hardening, Secrets Management, RBAC/ABAC, Rate Limiting และ WAF

---

## 1. OWASP Top 10 for Microservices

```typescript
// owasp-mitigations.ts
// การป้องกัน OWASP Top 10 ใน Microservices

import express, { Request, Response, NextFunction } from 'express';
import helmet from 'helmet';
import rateLimit from 'express-rate-limit';
import { body, validationResult } from 'express-validator';
import hpp from 'hpp';
import mongoSanitize from 'express-mongo-sanitize';

const app = express();

// A1: Broken Access Control
class AccessControlMiddleware {
  static checkOwnership(resourceType: string) {
    return async (req: Request, res: Response, next: NextFunction) => {
      const resourceId = req.params.id;
      const userId = req.user?.id;

      if (!userId) {
        return res.status(401).json({ error: 'Unauthorized' });
      }

      const resource = await getResource(resourceType, resourceId);
      
      if (!resource) {
        return res.status(404).json({ error: 'Not found' });
      }

      if (resource.ownerId !== userId && !req.user?.roles?.includes('admin')) {
        return res.status(403).json({ error: 'Forbidden' });
      }

      next();
    };
  }
}

// A2: Cryptographic Failures
class CryptographyService {
  // ใช้ bcrypt สำหรับ passwords
  async hashPassword(password: string): Promise<string> {
    const bcrypt = await import('bcrypt');
    return bcrypt.hash(password, 12); // cost factor 12
  }

  async verifyPassword(password: string, hash: string): Promise<boolean> {
    const bcrypt = await import('bcrypt');
    return bcrypt.compare(password, hash);
  }

  // ใช้ AES-256-GCM สำหรับ sensitive data
  encryptSensitiveData(plaintext: string, key: Buffer): {
    encrypted: string;
    iv: string;
    tag: string;
  } {
    const crypto = require('crypto');
    const iv = crypto.randomBytes(12);
    const cipher = crypto.createCipheriv('aes-256-gcm', key, iv);
    
    let encrypted = cipher.update(plaintext, 'utf8', 'base64');
    encrypted += cipher.final('base64');
    
    const tag = cipher.getAuthTag();

    return {
      encrypted,
      iv: iv.toString('base64'),
      tag: tag.toString('base64'),
    };
  }

  decryptSensitiveData(
    encryptedData: { encrypted: string; iv: string; tag: string },
    key: Buffer
  ): string {
    const crypto = require('crypto');
    const iv = Buffer.from(encryptedData.iv, 'base64');
    const tag = Buffer.from(encryptedData.tag, 'base64');
    
    const decipher = crypto.createDecipheriv('aes-256-gcm', key, iv);
    decipher.setAuthTag(tag);
    
    let decrypted = decipher.update(encryptedData.encrypted, 'base64', 'utf8');
    decrypted += decipher.final('utf8');
    
    return decrypted;
  }
}

// A3: Injection Prevention
const injectionPreventionMiddleware = [
  // SQL Injection prevention via parameterized queries (enforced by ORM)
  // NoSQL injection prevention
  mongoSanitize({
    replaceWith: '_',
    allowDots: false,
  }),
  
  // HTTP Parameter Pollution
  hpp({
    whitelist: ['filter', 'sort', 'fields'],
  }),
];

// A4: Insecure Design - Validate all inputs
const orderValidationRules = [
  body('userId').isUUID().withMessage('Invalid userId'),
  body('items').isArray({ min: 1 }).withMessage('Items must be non-empty array'),
  body('items.*.productId').isString().notEmpty(),
  body('items.*.quantity').isInt({ min: 1, max: 100 }),
  body('items.*.price').isFloat({ min: 0 }),
];

const validateRequest = (req: Request, res: Response, next: NextFunction) => {
  const errors = validationResult(req);
  if (!errors.isEmpty()) {
    return res.status(400).json({
      error: 'Validation failed',
      details: errors.array(),
    });
  }
  next();
};

// A5: Security Misconfiguration - Use Helmet
app.use(
  helmet({
    contentSecurityPolicy: {
      directives: {
        defaultSrc: ["'self'"],
        scriptSrc: ["'self'", "'nonce-{nonce}'"],
        styleSrc: ["'self'", "'unsafe-inline'"],
        imgSrc: ["'self'", 'data:', 'https:'],
        connectSrc: ["'self'", 'https://api.company.com'],
        frameSrc: ["'none'"],
        objectSrc: ["'none'"],
      },
    },
    hsts: {
      maxAge: 31536000,
      includeSubDomains: true,
      preload: true,
    },
    noSniff: true,
    referrerPolicy: { policy: 'strict-origin-when-cross-origin' },
    xssFilter: true,
  })
);

// A7: Identification and Authentication Failures
class AuthenticationService {
  private readonly MAX_LOGIN_ATTEMPTS = 5;
  private readonly LOCKOUT_DURATION = 15 * 60 * 1000; // 15 minutes
  private loginAttempts: Map<string, { count: number; lastAttempt: number }> = new Map();

  async login(email: string, password: string): Promise<{ token: string } | null> {
    // Check account lockout
    if (this.isAccountLocked(email)) {
      throw new Error('Account temporarily locked due to too many failed attempts');
    }

    const user = await findUserByEmail(email);
    if (!user) {
      this.recordFailedAttempt(email);
      // Constant time comparison to prevent timing attacks
      await this.fakePasswordCheck();
      return null;
    }

    const crypto = await import('crypto');
    const passwordValid = await verifyPassword(password, user.passwordHash);
    
    if (!passwordValid) {
      this.recordFailedAttempt(email);
      return null;
    }

    // Reset attempts on success
    this.loginAttempts.delete(email);

    // Generate secure token
    const token = this.generateSecureToken(user);
    
    // Log successful login
    await logSecurityEvent('LOGIN_SUCCESS', { userId: user.id, email });

    return { token };
  }

  private isAccountLocked(email: string): boolean {
    const attempts = this.loginAttempts.get(email);
    if (!attempts) return false;
    
    if (attempts.count >= this.MAX_LOGIN_ATTEMPTS) {
      const timeSinceLock = Date.now() - attempts.lastAttempt;
      if (timeSinceLock < this.LOCKOUT_DURATION) {
        return true;
      }
      // Reset after lockout period
      this.loginAttempts.delete(email);
    }
    
    return false;
  }

  private recordFailedAttempt(email: string): void {
    const current = this.loginAttempts.get(email) || { count: 0, lastAttempt: 0 };
    this.loginAttempts.set(email, {
      count: current.count + 1,
      lastAttempt: Date.now(),
    });
  }

  private async fakePasswordCheck(): Promise<void> {
    // Prevent timing attacks
    const bcrypt = await import('bcrypt');
    await bcrypt.compare('fake', '$2b$12$fake-hash-to-prevent-timing-attacks');
  }

  private generateSecureToken(user: any): string {
    const jwt = require('jsonwebtoken');
    return jwt.sign(
      {
        sub: user.id,
        email: user.email,
        roles: user.roles,
        jti: require('crypto').randomUUID(), // Unique token ID
      },
      process.env.JWT_SECRET!,
      {
        expiresIn: '15m',
        issuer: 'company.com',
        audience: 'api.company.com',
        algorithm: 'HS256',
      }
    );
  }
}

async function getResource(type: string, id: string): Promise<any> {
  return null;
}

async function findUserByEmail(email: string): Promise<any> {
  return null;
}

async function verifyPassword(password: string, hash: string): Promise<boolean> {
  return false;
}

async function logSecurityEvent(event: string, data: any): Promise<void> {}
```

---

## 2. Penetration Testing for APIs

```typescript
// pen-testing-scripts.ts
// Security testing utilities

import axios from 'axios';

class APISecurityTester {
  private baseUrl: string;
  private results: SecurityTestResult[] = [];

  constructor(baseUrl: string) {
    this.baseUrl = baseUrl;
  }

  interface SecurityTestResult {
    test: string;
    severity: 'CRITICAL' | 'HIGH' | 'MEDIUM' | 'LOW' | 'INFO';
    passed: boolean;
    details: string;
  }

  // Test for SQL Injection
  async testSQLInjection(): Promise<void> {
    const sqlPayloads = [
      "' OR '1'='1",
      "'; DROP TABLE users; --",
      "1' UNION SELECT username, password FROM users--",
      "admin'--",
      "1; EXEC xp_cmdshell('dir')--",
    ];

    for (const payload of sqlPayloads) {
      try {
        const response = await axios.get(`${this.baseUrl}/users`, {
          params: { search: payload },
          validateStatus: () => true,
        });

        // Check for error messages revealing DB structure
        const body = JSON.stringify(response.data);
        const sqlErrorIndicators = [
          'syntax error', 'mysql_fetch', 'ORA-', 'PG::',
          'sqlite3', 'Microsoft SQL', 'ODBC Error',
        ];

        const hasError = sqlErrorIndicators.some(indicator =>
          body.toLowerCase().includes(indicator.toLowerCase())
        );

        this.results.push({
          test: `SQL Injection: ${payload.substring(0, 20)}...`,
          severity: 'CRITICAL',
          passed: !hasError && response.status !== 500,
          details: hasError
            ? `Potential SQL injection vulnerability with payload: ${payload}`
            : 'No SQL error revealed',
        });
      } catch (error) {
        // Connection error = good (service protected itself)
      }
    }
  }

  // Test for IDOR (Insecure Direct Object Reference)
  async testIDOR(ownToken: string, otherUserId: string): Promise<void> {
    try {
      const response = await axios.get(
        `${this.baseUrl}/users/${otherUserId}/orders`,
        {
          headers: { Authorization: `Bearer ${ownToken}` },
          validateStatus: () => true,
        }
      );

      this.results.push({
        test: 'IDOR: Access other user orders',
        severity: 'CRITICAL',
        passed: response.status === 403 || response.status === 404,
        details: response.status === 200
          ? 'IDOR VULNERABILITY: Can access other user data!'
          : `Access correctly denied with ${response.status}`,
      });
    } catch (error) {
      console.error('IDOR test error:', error);
    }
  }

  // Test for Authentication bypass
  async testAuthBypass(): Promise<void> {
    const protectedEndpoints = [
      '/orders',
      '/users/profile',
      '/admin/dashboard',
    ];

    for (const endpoint of protectedEndpoints) {
      const tests = [
        { name: 'No token', headers: {} },
        { name: 'Invalid token', headers: { Authorization: 'Bearer invalid.token.here' } },
        { name: 'Expired token', headers: { Authorization: 'Bearer eyJhbGciOiJIUzI1NiJ9.eyJleHAiOjE2MDAwMDAwMDB9.invalid' } },
        { name: 'SQL in token', headers: { Authorization: "Bearer '; DROP TABLE--" } },
      ];

      for (const test of tests) {
        try {
          const response = await axios.get(`${this.baseUrl}${endpoint}`, {
            headers: test.headers,
            validateStatus: () => true,
          });

          this.results.push({
            test: `Auth Bypass ${endpoint}: ${test.name}`,
            severity: 'CRITICAL',
            passed: response.status === 401,
            details: response.status === 200
              ? `AUTH BYPASS VULNERABILITY on ${endpoint}!`
              : `Correctly returned ${response.status}`,
          });
        } catch (error) {
          console.error('Auth bypass test error:', error);
        }
      }
    }
  }

  // Test rate limiting
  async testRateLimiting(): Promise<void> {
    const requestCount = 150;
    let rateLimitedCount = 0;

    const requests = Array.from({ length: requestCount }, () =>
      axios.get(`${this.baseUrl}/health`, {
        validateStatus: () => true,
      })
    );

    const responses = await Promise.allSettled(requests);
    
    responses.forEach(result => {
      if (result.status === 'fulfilled' && result.value.status === 429) {
        rateLimitedCount++;
      }
    });

    this.results.push({
      test: 'Rate Limiting',
      severity: 'HIGH',
      passed: rateLimitedCount > 0,
      details: rateLimitedCount > 0
        ? `Rate limiting active: ${rateLimitedCount}/${requestCount} requests blocked`
        : 'RATE LIMITING NOT ENFORCED!',
    });
  }

  // Test for sensitive data exposure
  async testSensitiveDataExposure(token: string): Promise<void> {
    const response = await axios.get(`${this.baseUrl}/users/me`, {
      headers: { Authorization: `Bearer ${token}` },
      validateStatus: () => true,
    });

    if (response.status === 200) {
      const sensitiveFields = ['password', 'passwordHash', 'ssn', 'cvv', 'creditCard'];
      const body = JSON.stringify(response.data);
      
      const exposedFields = sensitiveFields.filter(field =>
        body.toLowerCase().includes(field.toLowerCase())
      );

      this.results.push({
        test: 'Sensitive Data Exposure',
        severity: 'HIGH',
        passed: exposedFields.length === 0,
        details: exposedFields.length > 0
          ? `SENSITIVE DATA EXPOSED: ${exposedFields.join(', ')}`
          : 'No sensitive data exposed in response',
      });
    }
  }

  getReport(): SecurityTestResult[] {
    return this.results;
  }

  printReport(): void {
    console.log('\n=== API Security Test Report ===\n');
    
    const failed = this.results.filter(r => !r.passed);
    const critical = failed.filter(r => r.severity === 'CRITICAL');

    console.log(`Total tests: ${this.results.length}`);
    console.log(`Passed: ${this.results.filter(r => r.passed).length}`);
    console.log(`Failed: ${failed.length}`);
    console.log(`Critical failures: ${critical.length}`);

    if (critical.length > 0) {
      console.error('\nCRITICAL VULNERABILITIES:');
      critical.forEach(r => console.error(`  ❌ ${r.test}: ${r.details}`));
    }
  }
}
```

---

## 3. Security Scanning in CI/CD (Trivy, Snyk)

```yaml
# .github/workflows/security-scan.yml
name: Security Scanning

on:
  push:
    branches: [main, develop]
  pull_request:
  schedule:
    - cron: '0 6 * * 1'  # Weekly Monday 6am

jobs:
  trivy-container-scan:
    name: Trivy Container Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Build image for scanning
        run: docker build -t order-service:scan .
      
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'order-service:scan'
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'
          ignore-unfixed: true
      
      - name: Upload Trivy results to GitHub Security
        uses: github/codeql-action/upload-sarif@v2
        if: always()
        with:
          sarif_file: 'trivy-results.sarif'
      
      - name: Run Trivy filesystem scan
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          format: 'table'
          severity: 'CRITICAL,HIGH'

  snyk-dependency-scan:
    name: Snyk Dependency Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Snyk to check for vulnerabilities
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high --sarif-file-output=snyk.sarif
      
      - name: Upload Snyk results to GitHub Security
        uses: github/codeql-action/upload-sarif@v2
        if: always()
        with:
          sarif_file: snyk.sarif

  secrets-scan:
    name: Secrets Scanning
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: GitLeaks scan
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }}
      
      - name: TruffleHog scan
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}
          head: HEAD
          extra_args: --debug --only-verified

  sast-scan:
    name: Static Analysis (CodeQL)
    runs-on: ubuntu-latest
    permissions:
      security-events: write
    steps:
      - uses: actions/checkout@v4
      
      - name: Initialize CodeQL
        uses: github/codeql-action/init@v2
        with:
          languages: 'javascript,typescript'
          queries: security-and-quality
      
      - name: Autobuild
        uses: github/codeql-action/autobuild@v2
      
      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v2
        with:
          category: "/language:javascript"

  dast-scan:
    name: Dynamic Application Security Testing
    runs-on: ubuntu-latest
    needs: [trivy-container-scan]
    steps:
      - uses: actions/checkout@v4
      
      - name: Start application
        run: docker-compose -f docker-compose.test.yml up -d
      
      - name: Wait for application
        run: sleep 30
      
      - name: OWASP ZAP Scan
        uses: zaproxy/action-api-scan@v0.5.0
        with:
          target: 'http://localhost:3000/api-docs/swagger.json'
          format: openapi
          rules_file_name: '.zap/rules.tsv'
          fail_action: true
          cmd_options: '-I'
      
      - uses: actions/upload-artifact@v3
        if: always()
        with:
          name: zap-report
          path: report_html.html
```

---

## 4. Container Image Hardening

```dockerfile
# Dockerfile.hardened
# Multi-stage hardened production build

# Build stage
FROM node:20-alpine AS builder
WORKDIR /build

# Copy dependency files first for better caching
COPY package*.json ./
RUN npm ci --only=production && \
    npm cache clean --force

COPY tsconfig.json ./
COPY src/ ./src/
RUN npm run build && \
    # Remove dev dependencies and source
    npm prune --production

# Final hardened stage
FROM node:20-alpine AS production

# Install security updates
RUN apk update && \
    apk upgrade && \
    apk add --no-cache \
    dumb-init \
    && rm -rf /var/cache/apk/*

# Create non-root user
RUN addgroup -g 1001 -S nodegroup && \
    adduser -u 1001 -S nodeuser -G nodegroup

# Set working directory
WORKDIR /app

# Copy built artifacts
COPY --from=builder --chown=nodeuser:nodegroup /build/node_modules ./node_modules
COPY --from=builder --chown=nodeuser:nodegroup /build/dist ./dist
COPY --chown=nodeuser:nodegroup package.json ./

# Remove unnecessary tools
RUN rm -rf /usr/local/lib/node_modules/npm && \
    rm -f /usr/local/bin/npm /usr/local/bin/npx

# Use non-root user
USER nodeuser

# Set secure environment
ENV NODE_ENV=production \
    NODE_OPTIONS="--max-old-space-size=512" \
    NPM_CONFIG_LOGLEVEL=error

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3000/health', r => process.exit(r.statusCode === 200 ? 0 : 1)).on('error', () => process.exit(1))"

# Use dumb-init to handle signals properly
ENTRYPOINT ["dumb-init", "--"]
CMD ["node", "dist/index.js"]

# Security labels
LABEL \
  maintainer="security@company.com" \
  security.scan="required" \
  org.opencontainers.image.source="https://github.com/company/order-service"
```

```yaml
# image-scanning-policy.yaml
# OPA policy สำหรับ container security

package kubernetes.admission

import future.keywords

# Deny privileged containers
deny[msg] {
  input.request.kind.kind == "Pod"
  container := input.request.object.spec.containers[_]
  container.securityContext.privileged == true
  msg := sprintf("Privileged containers are not allowed: %v", [container.name])
}

# Require non-root user
deny[msg] {
  input.request.kind.kind == "Pod"
  container := input.request.object.spec.containers[_]
  not container.securityContext.runAsNonRoot
  msg := sprintf("Container must run as non-root: %v", [container.name])
}

# Require read-only root filesystem
warn[msg] {
  input.request.kind.kind == "Pod"
  container := input.request.object.spec.containers[_]
  not container.securityContext.readOnlyRootFilesystem
  msg := sprintf("Container should use read-only root filesystem: %v", [container.name])
}

# Require resource limits
deny[msg] {
  input.request.kind.kind == "Pod"
  container := input.request.object.spec.containers[_]
  not container.resources.limits.memory
  msg := sprintf("Container must have memory limits: %v", [container.name])
}

# Require image from trusted registry
deny[msg] {
  input.request.kind.kind == "Pod"
  container := input.request.object.spec.containers[_]
  not startswith(container.image, "registry.company.com/")
  not startswith(container.image, "gcr.io/company/")
  msg := sprintf("Image must come from trusted registry: %v", [container.image])
}

# Require image tag (not latest)
deny[msg] {
  input.request.kind.kind == "Pod"
  container := input.request.object.spec.containers[_]
  endswith(container.image, ":latest")
  msg := sprintf("Image must use specific tag, not latest: %v", [container.image])
}
```

---

## 5. Network Segmentation

```yaml
# network-security.yaml
# Kubernetes Network Policies สำหรับ Microsegmentation

# Default: deny all ingress and egress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
---
# Allow DNS egress (needed for all pods)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
---
# Order Service specific network policy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: order-service-network-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: order-service
  policyTypes:
    - Ingress
    - Egress
  
  ingress:
    # From API Gateway only
    - from:
        - podSelector:
            matchLabels:
              app: api-gateway
        - namespaceSelector:
            matchLabels:
              name: ingress
      ports:
        - port: 3000
    # From monitoring
    - from:
        - namespaceSelector:
            matchLabels:
              name: monitoring
      ports:
        - port: 9090  # Prometheus metrics
  
  egress:
    # To PostgreSQL
    - to:
        - podSelector:
            matchLabels:
              app: postgresql
      ports:
        - port: 5432
    # To Redis
    - to:
        - podSelector:
            matchLabels:
              app: redis
      ports:
        - port: 6379
    # To Inventory Service
    - to:
        - podSelector:
            matchLabels:
              app: inventory-service
      ports:
        - port: 3000
    # To Payment Service
    - to:
        - podSelector:
            matchLabels:
              app: payment-service
      ports:
        - port: 3000
    # To Kafka
    - to:
        - podSelector:
            matchLabels:
              app: kafka
      ports:
        - port: 9092
---
# Service Mesh mTLS Policy (Istio)
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default-mtls
  namespace: production
spec:
  mtls:
    mode: STRICT  # Require mTLS for all service-to-service communication
---
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: order-service-authz
  namespace: production
spec:
  selector:
    matchLabels:
      app: order-service
  rules:
    - from:
        - source:
            principals:
              - "cluster.local/ns/production/sa/api-gateway-sa"
              - "cluster.local/ns/production/sa/frontend-sa"
    - to:
        - operation:
            methods: ["GET", "POST", "PUT", "PATCH"]
            paths: ["/api/*"]
```

---

## 6. Secrets Scanning (GitLeaks, TruffleHog)

```typescript
// secrets-management.ts
// ใช้ HashiCorp Vault สำหรับ secrets management

import Vault from 'node-vault';

class SecretsManager {
  private vault: Vault.client;
  private cache: Map<string, { value: string; expiresAt: number }> = new Map();

  constructor() {
    this.vault = Vault({
      endpoint: process.env.VAULT_ADDR || 'http://vault:8200',
      token: process.env.VAULT_TOKEN,
    });
  }

  async getSecret(path: string, key: string): Promise<string> {
    const cacheKey = `${path}:${key}`;
    const cached = this.cache.get(cacheKey);
    
    if (cached && cached.expiresAt > Date.now()) {
      return cached.value;
    }

    try {
      const result = await this.vault.read(path);
      const value = result.data.data[key] as string;
      
      // Cache for 5 minutes
      this.cache.set(cacheKey, {
        value,
        expiresAt: Date.now() + 5 * 60 * 1000,
      });

      return value;
    } catch (error) {
      throw new Error(`Failed to fetch secret ${path}:${key}: ${error}`);
    }
  }

  async getDatabaseCredentials(): Promise<{
    host: string;
    port: number;
    database: string;
    username: string;
    password: string;
  }> {
    // Dynamic secrets - fresh credentials each time
    const result = await this.vault.read('database/creds/order-service');
    
    return {
      host: process.env.DB_HOST!,
      port: parseInt(process.env.DB_PORT || '5432'),
      database: process.env.DB_NAME!,
      username: result.data.username,
      password: result.data.password,
    };
  }

  async rotateAppSecret(path: string, key: string): Promise<string> {
    const crypto = require('crypto');
    const newSecret = crypto.randomBytes(64).toString('base64');
    
    await this.vault.write(path, {
      data: { [key]: newSecret },
    });

    // Invalidate cache
    this.cache.delete(`${path}:${key}`);

    return newSecret;
  }
}

// .gitleaks.toml - GitLeaks configuration
const gitleaksConfig = `
title = "Company GitLeaks Config"

[extend]
useDefault = true

[[rules]]
id = "company-api-key"
description = "Company Internal API Key"
regex = '''company_[a-zA-Z0-9]{32}'''
tags = ["api-key", "company"]
severity = "CRITICAL"

[[rules]]
id = "database-url"
description = "Database Connection String"
regex = '''(postgres|mysql|mongodb):\/\/[^:]+:[^@]+@'''
tags = ["database", "credential"]
severity = "CRITICAL"

[allowlist]
description = "Global allowlist"
regexes = [
  # Test/example values
  '''EXAMPLE_''',
  '''TEST_''',
  '''your-secret-here''',
]
paths = [
  # Documentation and tests
  '''\.md$''',
  '''\.test\.ts$''',
  '''tests/''',
]
`;
```

---

## 7. RBAC vs ABAC for APIs

```typescript
// rbac-abac.ts
// Implementation ของ RBAC และ ABAC

// RBAC (Role-Based Access Control)
interface RBACPolicy {
  role: string;
  permissions: string[];
}

class RBACEngine {
  private policies: Map<string, Set<string>> = new Map();

  addPolicy(role: string, permissions: string[]): void {
    const existing = this.policies.get(role) || new Set();
    permissions.forEach(p => existing.add(p));
    this.policies.set(role, existing);
  }

  can(role: string, permission: string): boolean {
    // Check direct role
    if (this.policies.get(role)?.has(permission)) return true;
    
    // Check wildcard
    const [resource, action] = permission.split(':');
    if (this.policies.get(role)?.has(`${resource}:*`)) return true;
    if (this.policies.get(role)?.has('*:*')) return true;
    
    return false;
  }

  canWithRoles(roles: string[], permission: string): boolean {
    return roles.some(role => this.can(role, permission));
  }
}

// Setup RBAC policies
const rbac = new RBACEngine();
rbac.addPolicy('admin', ['*:*']);
rbac.addPolicy('order-manager', [
  'orders:read', 'orders:write', 'orders:cancel',
  'inventory:read',
]);
rbac.addPolicy('customer', [
  'orders:read-own', 'orders:create', 'orders:cancel-own',
]);
rbac.addPolicy('read-only', ['orders:read', 'inventory:read']);

// ABAC (Attribute-Based Access Control)
interface ABACSubject {
  userId: string;
  roles: string[];
  department: string;
  clearanceLevel: number;
}

interface ABACResource {
  type: string;
  ownerId?: string;
  classification?: 'public' | 'internal' | 'confidential' | 'secret';
  region?: string;
}

interface ABACEnvironment {
  time: Date;
  ipAddress: string;
  userAgent: string;
  mfaVerified: boolean;
}

type ABACAction = 'read' | 'write' | 'delete' | 'approve';

interface ABACPolicy {
  name: string;
  description: string;
  condition: (
    subject: ABACSubject,
    resource: ABACResource,
    action: ABACAction,
    environment: ABACEnvironment
  ) => boolean;
}

class ABACEngine {
  private policies: ABACPolicy[] = [];

  addPolicy(policy: ABACPolicy): void {
    this.policies.push(policy);
  }

  evaluate(
    subject: ABACSubject,
    resource: ABACResource,
    action: ABACAction,
    environment: ABACEnvironment
  ): { allowed: boolean; reason: string } {
    // All policies must pass (AND logic)
    for (const policy of this.policies) {
      const result = policy.condition(subject, resource, action, environment);
      if (!result) {
        return { allowed: false, reason: `Denied by policy: ${policy.name}` };
      }
    }
    
    return { allowed: true, reason: 'All policies passed' };
  }
}

// Setup ABAC policies
const abac = new ABACEngine();

abac.addPolicy({
  name: 'MFA Required for Sensitive Operations',
  description: 'Write operations on confidential data require MFA',
  condition: (subject, resource, action, environment) => {
    if (
      action === 'write' &&
      resource.classification === 'confidential' &&
      !environment.mfaVerified
    ) {
      return false;
    }
    return true;
  },
});

abac.addPolicy({
  name: 'Resource Ownership',
  description: 'Users can only modify their own resources',
  condition: (subject, resource, action) => {
    if (['write', 'delete'].includes(action)) {
      if (
        resource.ownerId &&
        resource.ownerId !== subject.userId &&
        !subject.roles.includes('admin')
      ) {
        return false;
      }
    }
    return true;
  },
});

abac.addPolicy({
  name: 'Business Hours for Admin Operations',
  description: 'Critical admin operations only during business hours',
  condition: (subject, resource, action, environment) => {
    if (subject.roles.includes('admin') && action === 'delete') {
      const hour = environment.time.getHours();
      return hour >= 9 && hour < 18;
    }
    return true;
  },
});

abac.addPolicy({
  name: 'Geographic Restrictions',
  description: 'Certain resources restricted to specific regions',
  condition: (subject, resource) => {
    if (resource.region === 'EU' && subject.department !== 'EU-team') {
      if (!subject.roles.includes('admin')) return false;
    }
    return true;
  },
});

// Express middleware combining RBAC and ABAC
function createAuthzMiddleware(permission: string, resourceFetcher?: (req: any) => Promise<ABACResource>) {
  return async (req: any, res: any, next: any) => {
    const user = req.user as ABACSubject;

    // RBAC check first
    if (!rbac.canWithRoles(user.roles, permission)) {
      return res.status(403).json({ error: 'Insufficient permissions' });
    }

    // ABAC check if resource fetcher provided
    if (resourceFetcher) {
      const resource = await resourceFetcher(req);
      const environment: ABACEnvironment = {
        time: new Date(),
        ipAddress: req.ip,
        userAgent: req.headers['user-agent'] || '',
        mfaVerified: req.user.mfaVerified || false,
      };

      const result = abac.evaluate(user, resource, 'write', environment);
      if (!result.allowed) {
        return res.status(403).json({ error: result.reason });
      }
    }

    next();
  };
}
```

---

## 8. API Rate Limiting and DDoS Protection

```typescript
// rate-limiting.ts
// Advanced rate limiting strategies

import Redis from 'ioredis';

interface RateLimitConfig {
  windowMs: number;
  maxRequests: number;
  keyGenerator?: (req: any) => string;
  skipSuccessfulRequests?: boolean;
  message?: string;
}

// Token Bucket Algorithm
class TokenBucketRateLimiter {
  private redis: Redis;
  private readonly BUCKET_CAPACITY: number;
  private readonly REFILL_RATE: number; // tokens per second

  constructor(redis: Redis, capacity: number, refillRate: number) {
    this.redis = redis;
    this.BUCKET_CAPACITY = capacity;
    this.refillRate = refillRate;
  }

  private refillRate: number;

  async consume(key: string, tokens: number = 1): Promise<{
    allowed: boolean;
    tokensRemaining: number;
    resetAfterMs: number;
  }> {
    const now = Date.now();
    const bucketKey = `bucket:${key}`;

    const script = `
      local key = KEYS[1]
      local capacity = tonumber(ARGV[1])
      local refillRate = tonumber(ARGV[2])
      local now = tonumber(ARGV[3])
      local requested = tonumber(ARGV[4])
      
      local data = redis.call('HMGET', key, 'tokens', 'lastRefill')
      local tokens = tonumber(data[1]) or capacity
      local lastRefill = tonumber(data[2]) or now
      
      -- Refill tokens based on elapsed time
      local elapsed = (now - lastRefill) / 1000
      tokens = math.min(capacity, tokens + elapsed * refillRate)
      
      if tokens >= requested then
        tokens = tokens - requested
        redis.call('HMSET', key, 'tokens', tokens, 'lastRefill', now)
        redis.call('EXPIRE', key, 3600)
        return {1, tokens, 0}
      else
        -- Calculate wait time
        local deficit = requested - tokens
        local waitMs = math.ceil(deficit / refillRate * 1000)
        return {0, tokens, waitMs}
      end
    `;

    const result = await this.redis.eval(
      script,
      1,
      bucketKey,
      String(this.BUCKET_CAPACITY),
      String(this.REFILL_RATE),
      String(now),
      String(tokens)
    ) as [number, number, number];

    return {
      allowed: result[0] === 1,
      tokensRemaining: result[1],
      resetAfterMs: result[2],
    };
  }
}

// Sliding Window Rate Limiter
class SlidingWindowRateLimiter {
  private redis: Redis;

  constructor(redis: Redis) {
    this.redis = redis;
  }

  async isAllowed(
    key: string,
    limit: number,
    windowMs: number
  ): Promise<{
    allowed: boolean;
    count: number;
    resetAfterMs: number;
  }> {
    const now = Date.now();
    const windowStart = now - windowMs;
    const redisKey = `sliding:${key}`;

    const multi = this.redis.multi();
    
    // Remove old entries
    multi.zremrangebyscore(redisKey, 0, windowStart);
    // Add current request
    multi.zadd(redisKey, now, `${now}-${Math.random()}`);
    // Count requests in window
    multi.zcard(redisKey);
    // Set expiry
    multi.pexpire(redisKey, windowMs);

    const results = await multi.exec();
    const count = results?.[2]?.[1] as number || 0;

    if (count > limit) {
      // Get oldest entry to calculate reset time
      const oldest = await this.redis.zrange(redisKey, 0, 0, 'WITHSCORES');
      const oldestTimestamp = oldest[1] ? parseInt(oldest[1]) : now;
      const resetAfterMs = windowMs - (now - oldestTimestamp);

      return { allowed: false, count, resetAfterMs };
    }

    return { allowed: true, count, resetAfterMs: 0 };
  }
}

// DDoS Protection Middleware
class DDoSProtection {
  private redis: Redis;
  private tokenBucket: TokenBucketRateLimiter;
  private slidingWindow: SlidingWindowRateLimiter;

  constructor(redis: Redis) {
    this.redis = redis;
    this.tokenBucket = new TokenBucketRateLimiter(redis, 100, 10); // 100 tokens, refill 10/sec
    this.slidingWindow = new SlidingWindowRateLimiter(redis);
  }

  middleware() {
    return async (req: any, res: any, next: any) => {
      const ip = req.ip || req.connection.remoteAddress;
      const userId = req.user?.id;

      // IP-based rate limiting
      const ipResult = await this.tokenBucket.consume(`ip:${ip}`, 1);
      if (!ipResult.allowed) {
        res.set({
          'Retry-After': Math.ceil(ipResult.resetAfterMs / 1000),
          'X-RateLimit-Limit': '100',
          'X-RateLimit-Remaining': '0',
          'X-RateLimit-Reset': new Date(Date.now() + ipResult.resetAfterMs).toISOString(),
        });
        return res.status(429).json({
          error: 'Too many requests',
          retryAfter: Math.ceil(ipResult.resetAfterMs / 1000),
        });
      }

      // User-based stricter limiting
      if (userId) {
        const userResult = await this.slidingWindow.isAllowed(
          `user:${userId}`,
          50, // 50 requests
          60000 // per minute
        );

        if (!userResult.allowed) {
          return res.status(429).json({
            error: 'User rate limit exceeded',
            retryAfter: Math.ceil(userResult.resetAfterMs / 1000),
          });
        }
      }

      // Set rate limit headers
      res.set({
        'X-RateLimit-Limit': '100',
        'X-RateLimit-Remaining': ipResult.tokensRemaining,
      });

      next();
    };
  }
}
```

---

## 9. WAF Configuration

```typescript
// waf-config.ts
// Web Application Firewall rules

interface WAFRule {
  id: string;
  name: string;
  description: string;
  pattern: RegExp | string;
  action: 'BLOCK' | 'CHALLENGE' | 'LOG';
  severity: 'CRITICAL' | 'HIGH' | 'MEDIUM' | 'LOW';
}

class WAFEngine {
  private rules: WAFRule[] = [];

  constructor() {
    this.loadDefaultRules();
  }

  private loadDefaultRules(): void {
    this.rules = [
      // SQL Injection
      {
        id: 'WAF-001',
        name: 'SQL Injection',
        description: 'Common SQL injection patterns',
        pattern: /(\b(select|insert|update|delete|drop|union|exec|execute)\b.*\b(from|into|where|values)\b|--\s|\/\*|\*\/)/i,
        action: 'BLOCK',
        severity: 'CRITICAL',
      },
      // XSS
      {
        id: 'WAF-002',
        name: 'Cross-Site Scripting',
        description: 'XSS attack patterns',
        pattern: /<script[\s\S]*?>[\s\S]*?<\/script>|javascript:|on\w+\s*=|eval\(|alert\(/i,
        action: 'BLOCK',
        severity: 'HIGH',
      },
      // Path Traversal
      {
        id: 'WAF-003',
        name: 'Path Traversal',
        description: 'Directory traversal attempts',
        pattern: /\.\.[\/\\]|\.\.%2f|\.\.%5c/i,
        action: 'BLOCK',
        severity: 'HIGH',
      },
      // Command Injection
      {
        id: 'WAF-004',
        name: 'Command Injection',
        description: 'OS command injection',
        pattern: /[;&|`$]|\$\(|`[^`]*`/,
        action: 'BLOCK',
        severity: 'CRITICAL',
      },
      // Large payload
      {
        id: 'WAF-005',
        name: 'Oversized Request',
        description: 'Request body exceeds limit',
        pattern: 'BODY_SIZE_CHECK',
        action: 'BLOCK',
        severity: 'MEDIUM',
      },
    ];
  }

  inspect(request: {
    method: string;
    url: string;
    headers: Record<string, string>;
    body?: string;
    ip: string;
  }): {
    allowed: boolean;
    violations: Array<{ rule: string; field: string }>;
  } {
    const violations: Array<{ rule: string; field: string }> = [];

    for (const rule of this.rules) {
      if (rule.pattern === 'BODY_SIZE_CHECK') {
        if (request.body && request.body.length > 1024 * 1024) { // 1MB limit
          violations.push({ rule: rule.id, field: 'body' });
        }
        continue;
      }

      const pattern = rule.pattern as RegExp;

      // Check URL
      if (pattern.test(request.url)) {
        violations.push({ rule: rule.id, field: 'url' });
      }

      // Check headers
      for (const [key, value] of Object.entries(request.headers)) {
        if (pattern.test(value)) {
          violations.push({ rule: rule.id, field: `header:${key}` });
        }
      }

      // Check body
      if (request.body && pattern.test(request.body)) {
        violations.push({ rule: rule.id, field: 'body' });
      }
    }

    const blockViolations = violations.filter(v => {
      const rule = this.rules.find(r => r.id === v.rule);
      return rule?.action === 'BLOCK';
    });

    return {
      allowed: blockViolations.length === 0,
      violations,
    };
  }

  middleware() {
    return (req: any, res: any, next: any) => {
      const inspection = this.inspect({
        method: req.method,
        url: req.url,
        headers: req.headers,
        body: req.body ? JSON.stringify(req.body) : undefined,
        ip: req.ip,
      });

      if (!inspection.allowed) {
        // Log violation
        console.warn('WAF BLOCK:', {
          ip: req.ip,
          url: req.url,
          violations: inspection.violations,
        });

        return res.status(403).json({
          error: 'Request blocked by security policy',
          requestId: req.id,
        });
      }

      if (inspection.violations.length > 0) {
        // Log warnings
        console.warn('WAF WARN:', {
          ip: req.ip,
          violations: inspection.violations,
        });
      }

      next();
    };
  }
}
```

---

## 10. Security Incident Response

```typescript
// incident-response.ts

type IncidentSeverity = 'P1' | 'P2' | 'P3' | 'P4';

interface SecurityIncident {
  id: string;
  type: string;
  severity: IncidentSeverity;
  description: string;
  affectedSystems: string[];
  detectedAt: Date;
  status: 'OPEN' | 'INVESTIGATING' | 'CONTAINED' | 'RESOLVED';
  timeline: IncidentEvent[];
}

interface IncidentEvent {
  timestamp: Date;
  action: string;
  actor: string;
  details: string;
}

class SecurityIncidentResponder {
  private incidents: Map<string, SecurityIncident> = new Map();

  async declareIncident(
    type: string,
    severity: IncidentSeverity,
    description: string,
    affectedSystems: string[]
  ): Promise<SecurityIncident> {
    const incident: SecurityIncident = {
      id: crypto.randomUUID(),
      type,
      severity,
      description,
      affectedSystems,
      detectedAt: new Date(),
      status: 'OPEN',
      timeline: [{
        timestamp: new Date(),
        action: 'INCIDENT_DECLARED',
        actor: 'system',
        details: description,
      }],
    };

    this.incidents.set(incident.id, incident);

    // Execute immediate response
    await this.executeImmediateResponse(incident);

    return incident;
  }

  private async executeImmediateResponse(incident: SecurityIncident): Promise<void> {
    switch (incident.type) {
      case 'DATA_BREACH':
        await this.handleDataBreach(incident);
        break;
      case 'UNAUTHORIZED_ACCESS':
        await this.handleUnauthorizedAccess(incident);
        break;
      case 'DDOS_ATTACK':
        await this.handleDDoSAttack(incident);
        break;
      case 'COMPROMISED_CREDENTIALS':
        await this.handleCompromisedCredentials(incident);
        break;
    }
  }

  private async handleDataBreach(incident: SecurityIncident): Promise<void> {
    console.log('DATA BREACH - Executing response plan');
    
    // 1. Isolate affected systems
    await this.isolateSystems(incident.affectedSystems);
    
    // 2. Revoke all active sessions
    await this.revokeAllSessions();
    
    // 3. Enable enhanced logging
    await this.enableForensicLogging();
    
    // 4. Notify security team
    await this.notifySecurityTeam(incident, 'URGENT: Data breach detected');
    
    // 5. Preserve evidence
    await this.preserveForensicEvidence(incident);
    
    this.addToTimeline(incident.id, 'DATA_BREACH_RESPONSE_INITIATED', 'system', 
      'Automated response: isolated systems, revoked sessions, enabled forensic logging');
  }

  private async handleDDoSAttack(incident: SecurityIncident): Promise<void> {
    // 1. Enable rate limiting
    // 2. Enable WAF in block mode
    // 3. Enable CAPTCHA challenges
    // 4. Contact CDN provider for DDoS mitigation
    console.log('DDoS response initiated');
    
    this.addToTimeline(incident.id, 'DDOS_MITIGATION_STARTED', 'system',
      'Enabled rate limiting, WAF block mode, and CDN DDoS mitigation');
  }

  private async handleCompromisedCredentials(incident: SecurityIncident): Promise<void> {
    // 1. Force password reset for affected users
    // 2. Revoke all tokens
    // 3. Enable enhanced MFA requirements
    console.log('Compromised credentials response initiated');
  }

  private async isolateSystems(systems: string[]): Promise<void> {
    console.log(`Isolating systems: ${systems.join(', ')}`);
  }

  private async revokeAllSessions(): Promise<void> {
    console.log('Revoking all active sessions');
  }

  private async enableForensicLogging(): Promise<void> {
    console.log('Enabling enhanced forensic logging');
  }

  private async notifySecurityTeam(incident: SecurityIncident, message: string): Promise<void> {
    console.log(`Security team notified: ${message}`);
  }

  private async preserveForensicEvidence(incident: SecurityIncident): Promise<void> {
    console.log(`Preserving forensic evidence for incident ${incident.id}`);
  }

  private async handleUnauthorizedAccess(incident: SecurityIncident): Promise<void> {
    console.log('Unauthorized access response initiated');
  }

  private addToTimeline(incidentId: string, action: string, actor: string, details: string): void {
    const incident = this.incidents.get(incidentId);
    if (incident) {
      incident.timeline.push({
        timestamp: new Date(),
        action,
        actor,
        details,
      });
    }
  }
}
```

---

## สรุปตาราง Security Patterns

| OWASP Category | Mitigation | Tools |
|---------------|------------|-------|
| A1: Broken Access Control | RBAC/ABAC, ownership checks | Casbin, OPA |
| A2: Cryptographic Failures | AES-256, bcrypt, TLS 1.3 | Node crypto, bcrypt |
| A3: Injection | Parameterized queries, input validation | Prisma, express-validator |
| A4: Insecure Design | Threat modeling, security reviews | OWASP TM |
| A5: Security Misconfiguration | Helmet, CIS benchmarks | Helmet.js, Trivy |
| A6: Vulnerable Components | Dependency scanning | Snyk, Dependabot |
| A7: Auth Failures | MFA, account lockout, JWT | Passport.js |
| A8: SSRF | Allowlist URLs, no user-supplied URLs | Custom middleware |
| A9: Security Logging | SIEM, audit logs | ELK, Datadog |
| A10: Server-Side Request Forgery | URL validation | Custom validation |

| Security Tool | Purpose | Integration |
|--------------|---------|-------------|
| Trivy | Container/filesystem scanning | CI/CD |
| Snyk | Dependency vulnerability | CI/CD, IDE |
| GitLeaks | Secret scanning | Pre-commit, CI |
| TruffleHog | Deep secret scanning | CI/CD |
| OWASP ZAP | DAST | CI/CD |
| CodeQL | SAST | GitHub Actions |
| Falco | Runtime security | Kubernetes |
| OPA/Gatekeeper | Policy enforcement | Kubernetes |

| Rate Limiting Algorithm | Best For | Trade-off |
|------------------------|---------|-----------|
| Fixed Window | Simple cases | Burst at boundary |
| Sliding Window | Smooth limiting | Higher memory |
| Token Bucket | API rate limiting | Complex implementation |
| Leaky Bucket | Traffic shaping | Delayed responses |

---

## สรุป

Advanced Security Patterns ครอบคลุม:

1. **OWASP Top 10**: mitigation สำหรับแต่ละ vulnerability category
2. **Penetration Testing**: automated security tests สำหรับ CI/CD
3. **Security Scanning**: Trivy, Snyk, GitLeaks ใน pipeline
4. **Container Hardening**: non-root, read-only FS, minimal image
5. **Network Segmentation**: Kubernetes NetworkPolicy, Istio mTLS
6. **Secrets Management**: HashiCorp Vault dynamic credentials
7. **RBAC/ABAC**: layered access control
8. **Rate Limiting**: Token bucket algorithm สำหรับ DDoS protection
9. **WAF**: request inspection และ blocking
10. **Incident Response**: automated response procedures

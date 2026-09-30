# Part 37: Zero Trust Security Architecture สำหรับ Microservices

## บทนำ

Zero Trust Security คือแนวคิดที่ว่า "Never Trust, Always Verify" — ไม่มีการเชื่อถือโดยอัตโนมัติไม่ว่าจะมาจากภายในหรือภายนอกเครือข่าย ทุก request ต้องได้รับการตรวจสอบและอนุญาตอย่างต่อเนื่อง

## หัวข้อที่ครอบคลุม

1. Zero Trust Principles ใน Microservices
2. mTLS ระหว่าง Services
3. JWT + JWKS Rotation
4. Open Policy Agent (OPA) สำหรับ Authorization
5. Secret Management ด้วย HashiCorp Vault
6. Network Policies ใน Kubernetes
7. Runtime Security ด้วย Falco
8. Supply Chain Security

---

## 1. Zero Trust Principles

### 1.1 สถาปัตยกรรม Zero Trust

```
                    ┌─────────────────────────────────┐
                    │         Zero Trust Perimeter     │
                    │                                  │
  Client ──HTTPS──▶ │  API Gateway                    │
                    │    │                             │
                    │    ├──▶ Auth Service (OIDC)      │
                    │    │      │                      │
                    │    │    Validate Token           │
                    │    │      │                      │
                    │    ▼                             │
                    │  Policy Engine (OPA)             │
                    │    │                             │
                    │    ▼                             │
                    │  Service Mesh (mTLS)             │
                    │    │                             │
                    │    ├──▶ Service A                │
                    │    ├──▶ Service B                │
                    │    └──▶ Service C                │
                    │                                  │
                    └─────────────────────────────────┘
```

### 1.2 Zero Trust Checklist

```typescript
// zero-trust/checklist.ts
export const zeroTrustChecklist = {
  identity: {
    strongAuthentication: true,     // MFA หรือ certificate-based
    continuousVerification: true,   // verify ทุก request ไม่ใช่แค่ครั้งแรก
    leastPrivilege: true,          // access เฉพาะที่จำเป็น
    deviceTrust: true,             // ตรวจสอบ device ด้วย
  },
  network: {
    microsegmentation: true,       // แบ่ง network เป็น segment เล็กๆ
    encryptAllTraffic: true,       // mTLS ทุก connection
    noImplicitTrust: true,         // ไม่เชื่อถือ internal network
    continuousMonitoring: true,    // monitor ตลอดเวลา
  },
  data: {
    classifyData: true,            // จำแนกประเภทข้อมูล
    encryptAtRest: true,           // เข้ารหัสข้อมูลที่เก็บ
    encryptInTransit: true,        // เข้ารหัสข้อมูลที่ส่ง
    dataLossPrevention: true,      // ป้องกันข้อมูลรั่วไหล
  },
  applications: {
    secureByDesign: true,          // ออกแบบให้ปลอดภัยตั้งแต่ต้น
    vulnerabilityScanning: true,   // scan หา vulnerability ต่อเนื่อง
    runtimeProtection: true,       // ป้องกัน runtime attacks
    auditLogging: true,           // บันทึก audit log ทุก action
  }
};
```

---

## 2. Mutual TLS (mTLS) ระหว่าง Services

### 2.1 Certificate Authority Setup

```bash
# scripts/setup-pki.sh
#!/bin/bash

set -euo pipefail

PKI_DIR="./pki"
mkdir -p "$PKI_DIR"/{ca,services,clients}

# สร้าง Root CA
echo "Creating Root CA..."
openssl genrsa -out "$PKI_DIR/ca/ca-key.pem" 4096

openssl req -new -x509 \
  -key "$PKI_DIR/ca/ca-key.pem" \
  -out "$PKI_DIR/ca/ca-cert.pem" \
  -days 3650 \
  -subj "/C=TH/O=MyOrg/CN=MyOrg Root CA"

# ฟังก์ชัน สร้าง service certificate
create_service_cert() {
  local SERVICE=$1
  local DNS_NAME=$2
  
  echo "Creating certificate for $SERVICE..."
  
  # สร้าง private key
  openssl genrsa -out "$PKI_DIR/services/$SERVICE-key.pem" 2048
  
  # สร้าง CSR
  openssl req -new \
    -key "$PKI_DIR/services/$SERVICE-key.pem" \
    -out "$PKI_DIR/services/$SERVICE-csr.pem" \
    -subj "/C=TH/O=MyOrg/CN=$SERVICE"
  
  # สร้าง extension config
  cat > "$PKI_DIR/services/$SERVICE-ext.cnf" << EOF
[v3_req]
subjectAltName = @alt_names

[alt_names]
DNS.1 = $DNS_NAME
DNS.2 = $SERVICE
DNS.3 = $SERVICE.default.svc.cluster.local
EOF
  
  # Sign ด้วย CA
  openssl x509 -req \
    -in "$PKI_DIR/services/$SERVICE-csr.pem" \
    -CA "$PKI_DIR/ca/ca-cert.pem" \
    -CAkey "$PKI_DIR/ca/ca-key.pem" \
    -CAcreateserial \
    -out "$PKI_DIR/services/$SERVICE-cert.pem" \
    -days 365 \
    -extensions v3_req \
    -extfile "$PKI_DIR/services/$SERVICE-ext.cnf"
  
  echo "Certificate for $SERVICE created successfully"
}

# สร้าง certificates สำหรับ services ทั้งหมด
create_service_cert "order-service" "order-service.default.svc.cluster.local"
create_service_cert "payment-service" "payment-service.default.svc.cluster.local"
create_service_cert "user-service" "user-service.default.svc.cluster.local"
create_service_cert "notification-service" "notification-service.default.svc.cluster.local"

echo "PKI setup complete!"
```

### 2.2 mTLS HTTP Client/Server

```typescript
// security/mtls-client.ts
import https from 'https';
import fs from 'fs';
import axios from 'axios';
import { AxiosInstance } from 'axios';

interface MTLSConfig {
  certPath: string;
  keyPath: string;
  caPath: string;
  serviceName: string;
}

export class MTLSHttpClient {
  private client: AxiosInstance;
  
  constructor(config: MTLSConfig) {
    const httpsAgent = new https.Agent({
      cert: fs.readFileSync(config.certPath),
      key: fs.readFileSync(config.keyPath),
      ca: fs.readFileSync(config.caPath),
      rejectUnauthorized: true,  // ต้องเป็น true เสมอ
      checkServerIdentity: (hostname, cert) => {
        // ตรวจสอบว่า certificate เป็นของ hostname จริง
        const sanList = cert.subjectaltname?.split(', ') || [];
        const validSAN = sanList.some(san => {
          const dnsMatch = san.match(/^DNS:(.+)$/);
          if (dnsMatch) {
            const pattern = dnsMatch[1].replace(/\*/g, '[^.]+');
            return new RegExp(`^${pattern}$`).test(hostname);
          }
          return false;
        });
        
        if (!validSAN) {
          return new Error(`Certificate hostname mismatch: ${hostname}`);
        }
        return undefined;
      }
    });
    
    this.client = axios.create({
      httpsAgent,
      headers: {
        'X-Service-Name': config.serviceName,
        'X-Request-ID': '', // จะถูก set ใน interceptor
      }
    });
    
    // เพิ่ม request ID ทุก request
    this.client.interceptors.request.use((config) => {
      config.headers['X-Request-ID'] = crypto.randomUUID();
      return config;
    });
    
    // Log ทุก request/response
    this.client.interceptors.response.use(
      (response) => {
        console.log({
          event: 'mtls_request_success',
          url: response.config.url,
          status: response.status,
          requestId: response.config.headers['X-Request-ID'],
        });
        return response;
      },
      (error) => {
        console.error({
          event: 'mtls_request_error',
          url: error.config?.url,
          error: error.message,
          requestId: error.config?.headers?.['X-Request-ID'],
        });
        throw error;
      }
    );
  }
  
  async get<T>(url: string): Promise<T> {
    const response = await this.client.get<T>(url);
    return response.data;
  }
  
  async post<T>(url: string, data: unknown): Promise<T> {
    const response = await this.client.post<T>(url, data);
    return response.data;
  }
}

// security/mtls-server.ts
import https from 'https';
import fs from 'fs';
import express from 'express';

export function createMTLSServer(app: express.Application, port: number): https.Server {
  const options: https.ServerOptions = {
    cert: fs.readFileSync('./pki/services/order-service-cert.pem'),
    key: fs.readFileSync('./pki/services/order-service-key.pem'),
    ca: fs.readFileSync('./pki/ca/ca-cert.pem'),
    requestCert: true,    // บังคับให้ client ส่ง certificate
    rejectUnauthorized: true,  // reject ถ้า cert ไม่ valid
  };
  
  const server = https.createServer(options, app);
  
  // Middleware ตรวจสอบ client certificate
  app.use((req: express.Request, res: express.Response, next: express.NextFunction) => {
    const socket = req.socket as any;
    const clientCert = socket.getPeerCertificate();
    
    if (!clientCert || Object.keys(clientCert).length === 0) {
      return res.status(401).json({ error: 'Client certificate required' });
    }
    
    // ดึงชื่อ service จาก certificate
    const serviceName = clientCert.subject?.CN;
    if (!serviceName) {
      return res.status(401).json({ error: 'Invalid client certificate' });
    }
    
    // เพิ่มข้อมูล service ใน request
    (req as any).clientService = {
      name: serviceName,
      cert: clientCert,
    };
    
    next();
  });
  
  server.listen(port, () => {
    console.log(`mTLS server listening on port ${port}`);
  });
  
  return server;
}
```

### 2.3 Certificate Rotation

```typescript
// security/cert-rotation.ts
import { execSync } from 'child_process';
import fs from 'fs';
import path from 'path';
import { EventEmitter } from 'events';

export class CertificateRotationManager extends EventEmitter {
  private rotationIntervalMs: number;
  private certDir: string;
  private checkIntervalId: NodeJS.Timeout | null = null;
  
  constructor(
    private serviceName: string,
    private vaultClient: VaultClient,
    rotationIntervalHours: number = 24
  ) {
    super();
    this.rotationIntervalMs = rotationIntervalHours * 60 * 60 * 1000;
    this.certDir = `/var/run/secrets/${serviceName}`;
  }
  
  async start(): Promise<void> {
    // Fetch certificates ครั้งแรก
    await this.fetchAndSaveCerts();
    
    // Schedule rotation ตาม interval
    this.checkIntervalId = setInterval(async () => {
      await this.checkAndRotate();
    }, this.rotationIntervalMs / 4); // check บ่อยกว่า rotation period
    
    console.log(`Certificate rotation manager started for ${this.serviceName}`);
  }
  
  private async checkAndRotate(): Promise<void> {
    const certPath = path.join(this.certDir, 'cert.pem');
    
    if (!fs.existsSync(certPath)) {
      await this.fetchAndSaveCerts();
      return;
    }
    
    // ตรวจสอบว่า cert จะหมดอายุใน 25% ของ validity period ไหม
    const certInfo = this.parseCertificate(certPath);
    const now = Date.now();
    const expiryTime = certInfo.notAfter.getTime();
    const issuedTime = certInfo.notBefore.getTime();
    const validityDuration = expiryTime - issuedTime;
    const rotationThreshold = expiryTime - (validityDuration * 0.25);
    
    if (now >= rotationThreshold) {
      console.log(`Rotating certificate for ${this.serviceName}...`);
      await this.fetchAndSaveCerts();
      this.emit('rotated', { serviceName: this.serviceName });
    }
  }
  
  private async fetchAndSaveCerts(): Promise<void> {
    try {
      // ขอ certificate จาก Vault PKI
      const certData = await this.vaultClient.issueCertificate({
        role: this.serviceName,
        commonName: `${this.serviceName}.default.svc.cluster.local`,
        ttl: '24h',
        altNames: [
          this.serviceName,
          `${this.serviceName}.default`,
          `${this.serviceName}.default.svc.cluster.local`,
        ]
      });
      
      // บันทึก certificates
      fs.mkdirSync(this.certDir, { recursive: true });
      
      fs.writeFileSync(
        path.join(this.certDir, 'cert.pem'),
        certData.certificate,
        { mode: 0o600 }
      );
      
      fs.writeFileSync(
        path.join(this.certDir, 'key.pem'),
        certData.privateKey,
        { mode: 0o600 }
      );
      
      fs.writeFileSync(
        path.join(this.certDir, 'ca.pem'),
        certData.issuingCA,
        { mode: 0o644 }
      );
      
      console.log(`Certificates saved for ${this.serviceName}`);
    } catch (error) {
      console.error(`Failed to fetch certificates for ${this.serviceName}:`, error);
      throw error;
    }
  }
  
  private parseCertificate(certPath: string): { notBefore: Date; notAfter: Date } {
    const output = execSync(
      `openssl x509 -in ${certPath} -noout -dates`,
      { encoding: 'utf8' }
    );
    
    const notBeforeMatch = output.match(/notBefore=(.+)/);
    const notAfterMatch = output.match(/notAfter=(.+)/);
    
    return {
      notBefore: new Date(notBeforeMatch![1]),
      notAfter: new Date(notAfterMatch![1]),
    };
  }
  
  stop(): void {
    if (this.checkIntervalId) {
      clearInterval(this.checkIntervalId);
    }
  }
}

// Vault Client interface
interface VaultClient {
  issueCertificate(params: {
    role: string;
    commonName: string;
    ttl: string;
    altNames: string[];
  }): Promise<{
    certificate: string;
    privateKey: string;
    issuingCA: string;
  }>;
}
```

---

## 3. JWT + JWKS Rotation

### 3.1 JWKS Endpoint

```typescript
// auth/jwks-manager.ts
import { generateKeyPairSync, createSign, createVerify } from 'crypto';
import { JWK, JWKSet } from 'jose';

interface KeyPair {
  kid: string;
  privateKey: string;
  publicKey: string;
  createdAt: Date;
  expiresAt: Date;
}

export class JWKSManager {
  private keyPairs: Map<string, KeyPair> = new Map();
  private activeKid: string | null = null;
  
  constructor(
    private keyRotationIntervalHours: number = 24,
    private maxKeysCount: number = 3
  ) {}
  
  async initialize(): Promise<void> {
    await this.generateNewKeyPair();
    
    // Rotate key ตาม interval
    setInterval(async () => {
      await this.rotateKeys();
    }, this.keyRotationIntervalHours * 60 * 60 * 1000);
  }
  
  private async generateNewKeyPair(): Promise<string> {
    const { privateKey, publicKey } = generateKeyPairSync('rsa', {
      modulusLength: 2048,
      publicKeyEncoding: { type: 'spki', format: 'pem' },
      privateKeyEncoding: { type: 'pkcs8', format: 'pem' },
    });
    
    const kid = `key-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
    const now = new Date();
    const expiresAt = new Date(
      now.getTime() + this.keyRotationIntervalHours * 2 * 60 * 60 * 1000
    );
    
    const keyPair: KeyPair = {
      kid,
      privateKey: privateKey as string,
      publicKey: publicKey as string,
      createdAt: now,
      expiresAt,
    };
    
    this.keyPairs.set(kid, keyPair);
    this.activeKid = kid;
    
    console.log(`Generated new key pair: ${kid}`);
    return kid;
  }
  
  private async rotateKeys(): Promise<void> {
    console.log('Rotating JWKS keys...');
    
    // ลบ keys ที่หมดอายุแล้ว
    const now = new Date();
    for (const [kid, keyPair] of this.keyPairs.entries()) {
      if (keyPair.expiresAt < now) {
        this.keyPairs.delete(kid);
        console.log(`Removed expired key: ${kid}`);
      }
    }
    
    // ถ้ามี keys มากเกินไป ลบ key เก่าสุด
    while (this.keyPairs.size >= this.maxKeysCount) {
      const oldestKid = [...this.keyPairs.entries()]
        .sort(([, a], [, b]) => a.createdAt.getTime() - b.createdAt.getTime())[0][0];
      this.keyPairs.delete(oldestKid);
      console.log(`Removed oldest key: ${oldestKid}`);
    }
    
    // สร้าง key ใหม่
    await this.generateNewKeyPair();
  }
  
  getJWKS(): object {
    const keys = [...this.keyPairs.values()].map((keyPair) => {
      // แปลง PEM เป็น JWK format
      const publicKeyBuffer = Buffer.from(
        keyPair.publicKey
          .replace(/-----BEGIN PUBLIC KEY-----/, '')
          .replace(/-----END PUBLIC KEY-----/, '')
          .replace(/\n/g, ''),
        'base64'
      );
      
      // Simplified JWK representation (production ควรใช้ library เช่น jose)
      return {
        kty: 'RSA',
        kid: keyPair.kid,
        use: 'sig',
        alg: 'RS256',
        // n, e จะได้จากการ parse public key จริงๆ
      };
    });
    
    return { keys };
  }
  
  signToken(payload: object): string {
    if (!this.activeKid) {
      throw new Error('No active signing key');
    }
    
    const keyPair = this.keyPairs.get(this.activeKid)!;
    const header = Buffer.from(
      JSON.stringify({ alg: 'RS256', typ: 'JWT', kid: this.activeKid })
    ).toString('base64url');
    
    const body = Buffer.from(JSON.stringify({
      ...payload,
      iat: Math.floor(Date.now() / 1000),
      exp: Math.floor(Date.now() / 1000) + 3600,
    })).toString('base64url');
    
    const sign = createSign('sha256');
    sign.update(`${header}.${body}`);
    const signature = sign.sign(keyPair.privateKey, 'base64url');
    
    return `${header}.${body}.${signature}`;
  }
  
  verifyToken(token: string): object {
    const parts = token.split('.');
    if (parts.length !== 3) {
      throw new Error('Invalid JWT format');
    }
    
    const header = JSON.parse(Buffer.from(parts[0], 'base64url').toString());
    const payload = JSON.parse(Buffer.from(parts[1], 'base64url').toString());
    
    // ตรวจสอบ kid
    const keyPair = this.keyPairs.get(header.kid);
    if (!keyPair) {
      throw new Error(`Unknown key ID: ${header.kid}`);
    }
    
    // ตรวจสอบ signature
    const verify = createVerify('sha256');
    verify.update(`${parts[0]}.${parts[1]}`);
    const isValid = verify.verify(keyPair.publicKey, parts[2], 'base64url');
    
    if (!isValid) {
      throw new Error('Invalid JWT signature');
    }
    
    // ตรวจสอบ expiry
    const now = Math.floor(Date.now() / 1000);
    if (payload.exp && payload.exp < now) {
      throw new Error('JWT has expired');
    }
    
    return payload;
  }
}

// auth/jwks-routes.ts
import express from 'express';
import { JWKSManager } from './jwks-manager';

export function createJWKSRoutes(jwksManager: JWKSManager): express.Router {
  const router = express.Router();
  
  // JWKS endpoint - สาธารณะ ให้ services อื่น verify token ได้
  router.get('/.well-known/jwks.json', (req, res) => {
    const jwks = jwksManager.getJWKS();
    res.set('Cache-Control', 'public, max-age=3600'); // cache 1 ชั่วโมง
    res.json(jwks);
  });
  
  // Token endpoint
  router.post('/token', async (req, res) => {
    try {
      const { username, password, scope } = req.body;
      
      // Authenticate user (simplified)
      const user = await authenticateUser(username, password);
      if (!user) {
        return res.status(401).json({ error: 'Invalid credentials' });
      }
      
      const token = jwksManager.signToken({
        sub: user.id,
        email: user.email,
        roles: user.roles,
        scope: scope || 'read',
      });
      
      res.json({
        access_token: token,
        token_type: 'Bearer',
        expires_in: 3600,
      });
    } catch (error) {
      res.status(500).json({ error: 'Token generation failed' });
    }
  });
  
  return router;
}

async function authenticateUser(username: string, password: string) {
  // ตรวจสอบ credentials จาก database
  return null; // placeholder
}
```

---

## 4. Open Policy Agent (OPA)

### 4.1 OPA Rego Policies

```rego
# policies/authz.rego
package authz

import future.keywords.if
import future.keywords.in

# Default deny
default allow := false

# อนุญาตถ้าผ่านทุก check
allow if {
  token_valid
  has_required_scope
  resource_allowed
  not rate_limited
}

# ตรวจสอบ JWT token
token_valid if {
  token := input.token
  decoded := io.jwt.decode(token)
  decoded[1].exp > time.now_ns() / 1000000000
  decoded[1].iss == "https://auth.myapp.com"
}

# ตรวจสอบ scope ที่จำเป็น
has_required_scope if {
  required_scope := data.resource_scopes[input.resource][input.method]
  token := input.token
  decoded := io.jwt.decode(token)
  scopes := split(decoded[1].scope, " ")
  required_scope in scopes
}

# ตรวจสอบการเข้าถึง resource
resource_allowed if {
  input.method == "GET"
  # GET อนุญาตสำหรับทุก authenticated user
  token_valid
}

resource_allowed if {
  input.method in ["POST", "PUT", "DELETE"]
  # Write operations ต้องการ role
  token := input.token
  decoded := io.jwt.decode(token)
  "admin" in decoded[1].roles
}

# Rate limiting check (จาก external data)
rate_limited if {
  client_id := input.client_id
  count(data.rate_limit_violations[client_id]) > 100
}

# ตรวจสอบ IP allowlist
ip_allowed if {
  client_ip := input.client_ip
  allowed_ranges := data.ip_allowlist
  some range in allowed_ranges
  net.cidr_contains(range, client_ip)
}
```

```rego
# policies/rbac.rego
package rbac

import future.keywords.if
import future.keywords.in

# Role definitions
roles := {
  "admin": {
    "permissions": ["read", "write", "delete", "admin"],
    "resources": ["*"]
  },
  "editor": {
    "permissions": ["read", "write"],
    "resources": ["products", "orders", "users"]
  },
  "viewer": {
    "permissions": ["read"],
    "resources": ["products", "orders"]
  },
  "service": {
    "permissions": ["read", "write"],
    "resources": ["internal"]
  }
}

# ตรวจสอบว่า user มี permission สำหรับ resource
has_permission(user_roles, permission, resource) if {
  some role in user_roles
  role_def := roles[role]
  permission in role_def.permissions
  resource_allowed(role_def.resources, resource)
}

resource_allowed(allowed_resources, resource) if {
  "*" in allowed_resources
}

resource_allowed(allowed_resources, resource) if {
  resource in allowed_resources
}

# สร้าง response สำหรับ authorization decision
decision := {
  "allow": allow,
  "reason": reason,
  "metadata": {
    "policy_version": "1.0.0",
    "evaluated_at": time.now_ns()
  }
}

allow if {
  has_permission(input.user.roles, input.permission, input.resource)
}

reason := "Access granted" if { allow }
reason := "Insufficient permissions" if { not allow }
```

### 4.2 OPA Middleware

```typescript
// middleware/opa-middleware.ts
import axios from 'axios';
import { Request, Response, NextFunction } from 'express';

interface OPAResult {
  allow: boolean;
  reason: string;
  metadata?: Record<string, unknown>;
}

interface OPAInput {
  token: string;
  method: string;
  path: string;
  resource: string;
  permission: string;
  client_ip: string;
  user?: {
    id: string;
    roles: string[];
  };
}

export class OPAMiddleware {
  constructor(
    private opaUrl: string = 'http://opa:8181',
    private policyPath: string = 'v1/data/authz/allow'
  ) {}
  
  authorize(resource: string, permission: string) {
    return async (req: Request, res: Response, next: NextFunction) => {
      try {
        const token = this.extractToken(req);
        if (!token) {
          return res.status(401).json({ error: 'No token provided' });
        }
        
        const input: OPAInput = {
          token,
          method: req.method,
          path: req.path,
          resource,
          permission,
          client_ip: req.ip || '',
          user: (req as any).user,
        };
        
        const result = await this.queryOPA(input);
        
        if (!result.allow) {
          return res.status(403).json({
            error: 'Forbidden',
            reason: result.reason,
          });
        }
        
        // เพิ่ม OPA result ใน request
        (req as any).opaResult = result;
        next();
      } catch (error) {
        console.error('OPA authorization error:', error);
        // Fail-closed: deny ถ้า OPA ไม่ตอบสนอง
        res.status(503).json({ error: 'Authorization service unavailable' });
      }
    };
  }
  
  private async queryOPA(input: OPAInput): Promise<OPAResult> {
    const response = await axios.post(
      `${this.opaUrl}/${this.policyPath}`,
      { input },
      {
        timeout: 500, // OPA ควรตอบสนองภายใน 500ms
        headers: { 'Content-Type': 'application/json' }
      }
    );
    
    return response.data.result;
  }
  
  private extractToken(req: Request): string | null {
    const authHeader = req.headers.authorization;
    if (authHeader?.startsWith('Bearer ')) {
      return authHeader.slice(7);
    }
    return null;
  }
}

// ตัวอย่างการใช้งาน
const opa = new OPAMiddleware('http://opa:8181');

// Order routes
app.get('/orders', opa.authorize('orders', 'read'), getOrders);
app.post('/orders', opa.authorize('orders', 'write'), createOrder);
app.delete('/orders/:id', opa.authorize('orders', 'delete'), deleteOrder);
```

### 4.3 OPA Deployment ใน Kubernetes

```yaml
# k8s/opa/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: opa
  namespace: security
spec:
  replicas: 3
  selector:
    matchLabels:
      app: opa
  template:
    metadata:
      labels:
        app: opa
    spec:
      containers:
      - name: opa
        image: openpolicyagent/opa:0.58.0
        args:
        - run
        - --server
        - --log-level=info
        - --log-format=json
        - --set=decision_logs.console=true
        - /policies
        ports:
        - containerPort: 8181
        livenessProbe:
          httpGet:
            path: /health
            port: 8181
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health?plugins
            port: 8181
          initialDelaySeconds: 5
          periodSeconds: 5
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 256Mi
        volumeMounts:
        - name: policies
          mountPath: /policies
      volumes:
      - name: policies
        configMap:
          name: opa-policies
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: opa-policies
  namespace: security
data:
  authz.rego: |
    package authz
    # ... (policy content)
  rbac.rego: |
    package rbac
    # ... (policy content)
---
apiVersion: v1
kind: Service
metadata:
  name: opa
  namespace: security
spec:
  selector:
    app: opa
  ports:
  - port: 8181
    targetPort: 8181
```

---

## 5. HashiCorp Vault Integration

### 5.1 Vault Setup และ Secret Management

```typescript
// security/vault-client.ts
import vault from 'node-vault';
import { EventEmitter } from 'events';

interface VaultConfig {
  endpoint: string;
  roleId: string;
  secretId: string;
  namespace?: string;
}

interface SecretData {
  value: string;
  metadata?: {
    version: number;
    createdTime: string;
    deletionTime: string | null;
  };
}

export class VaultClient extends EventEmitter {
  private client: vault.client;
  private token: string | null = null;
  private tokenRenewalTimer: NodeJS.Timeout | null = null;
  
  constructor(private config: VaultConfig) {
    super();
    this.client = vault({
      endpoint: config.endpoint,
      namespace: config.namespace,
    });
  }
  
  async authenticate(): Promise<void> {
    // AppRole authentication
    const result = await this.client.approleLogin({
      role_id: this.config.roleId,
      secret_id: this.config.secretId,
    });
    
    this.token = result.auth.client_token;
    this.client.token = this.token;
    
    // Schedule token renewal
    const leaseDuration = result.auth.lease_duration;
    const renewAt = (leaseDuration * 0.75) * 1000; // Renew at 75% of lease
    
    this.tokenRenewalTimer = setTimeout(async () => {
      await this.renewToken();
    }, renewAt);
    
    console.log('Authenticated with Vault, token lease:', leaseDuration);
  }
  
  private async renewToken(): Promise<void> {
    try {
      const result = await this.client.tokenRenewSelf();
      const leaseDuration = result.auth.lease_duration;
      const renewAt = (leaseDuration * 0.75) * 1000;
      
      this.tokenRenewalTimer = setTimeout(async () => {
        await this.renewToken();
      }, renewAt);
      
      this.emit('token_renewed');
    } catch (error) {
      console.error('Token renewal failed, re-authenticating...', error);
      await this.authenticate();
    }
  }
  
  async getSecret(path: string): Promise<SecretData> {
    const result = await this.client.read(`secret/data/${path}`);
    return {
      value: result.data.data,
      metadata: result.data.metadata,
    };
  }
  
  async writeSecret(path: string, data: Record<string, string>): Promise<void> {
    await this.client.write(`secret/data/${path}`, { data });
  }
  
  async getDatabaseCredentials(role: string): Promise<{
    username: string;
    password: string;
    leaseId: string;
    leaseDuration: number;
  }> {
    const result = await this.client.read(`database/creds/${role}`);
    return {
      username: result.data.username,
      password: result.data.password,
      leaseId: result.lease_id,
      leaseDuration: result.lease_duration,
    };
  }
  
  async issueCertificate(params: {
    role: string;
    commonName: string;
    ttl: string;
    altNames: string[];
  }): Promise<{
    certificate: string;
    privateKey: string;
    issuingCA: string;
  }> {
    const result = await this.client.write(`pki/issue/${params.role}`, {
      common_name: params.commonName,
      ttl: params.ttl,
      alt_names: params.altNames.join(','),
    });
    
    return {
      certificate: result.data.certificate,
      privateKey: result.data.private_key,
      issuingCA: result.data.issuing_ca,
    };
  }
  
  async encryptData(keyName: string, plaintext: string): Promise<string> {
    const base64 = Buffer.from(plaintext).toString('base64');
    const result = await this.client.write(`transit/encrypt/${keyName}`, {
      plaintext: base64,
    });
    return result.data.ciphertext;
  }
  
  async decryptData(keyName: string, ciphertext: string): Promise<string> {
    const result = await this.client.write(`transit/decrypt/${keyName}`, {
      ciphertext,
    });
    const decoded = Buffer.from(result.data.plaintext, 'base64').toString();
    return decoded;
  }
  
  destroy(): void {
    if (this.tokenRenewalTimer) {
      clearTimeout(this.tokenRenewalTimer);
    }
  }
}

// security/dynamic-db-credentials.ts
import { Pool } from 'pg';
import { VaultClient } from './vault-client';

export class DynamicDatabasePool {
  private pool: Pool | null = null;
  private leaseId: string | null = null;
  private renewalTimer: NodeJS.Timeout | null = null;
  
  constructor(
    private vault: VaultClient,
    private dbRole: string,
    private connectionConfig: {
      host: string;
      port: number;
      database: string;
    }
  ) {}
  
  async initialize(): Promise<void> {
    await this.refreshCredentials();
  }
  
  private async refreshCredentials(): Promise<void> {
    const credentials = await this.vault.getDatabaseCredentials(this.dbRole);
    
    // ปิด pool เดิม
    if (this.pool) {
      await this.pool.end();
    }
    
    // สร้าง pool ใหม่ด้วย credentials ใหม่
    this.pool = new Pool({
      ...this.connectionConfig,
      user: credentials.username,
      password: credentials.password,
      max: 10,
      idleTimeoutMillis: 30000,
      connectionTimeoutMillis: 2000,
    });
    
    this.leaseId = credentials.leaseId;
    
    // Renew credentials ก่อนหมดอายุ
    const renewAt = (credentials.leaseDuration * 0.75) * 1000;
    this.renewalTimer = setTimeout(async () => {
      await this.refreshCredentials();
    }, renewAt);
    
    console.log(`Database pool refreshed with new credentials (lease: ${credentials.leaseDuration}s)`);
  }
  
  async query(text: string, params?: unknown[]): Promise<unknown> {
    if (!this.pool) {
      throw new Error('Database pool not initialized');
    }
    return this.pool.query(text, params);
  }
  
  async destroy(): Promise<void> {
    if (this.renewalTimer) {
      clearTimeout(this.renewalTimer);
    }
    if (this.pool) {
      await this.pool.end();
    }
  }
}
```

---

## 6. Network Policies ใน Kubernetes

### 6.1 Default Deny Policy

```yaml
# k8s/network-policies/default-deny.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}  # ใช้กับ pod ทุกตัว
  policyTypes:
  - Ingress
  - Egress
---
# อนุญาต DNS resolution
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
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
```

### 6.2 Service-Specific Policies

```yaml
# k8s/network-policies/order-service.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: order-service-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: order-service
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # อนุญาต traffic จาก API Gateway เท่านั้น
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress
      podSelector:
        matchLabels:
          app: api-gateway
    ports:
    - protocol: TCP
      port: 3000
  # อนุญาต Prometheus scraping
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring
      podSelector:
        matchLabels:
          app: prometheus
    ports:
    - protocol: TCP
      port: 9090
  egress:
  # อนุญาตเชื่อมต่อ database
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432
  # อนุญาตเชื่อมต่อ payment-service
  - to:
    - podSelector:
        matchLabels:
          app: payment-service
    ports:
    - protocol: TCP
      port: 3001
  # อนุญาตเชื่อมต่อ message broker
  - to:
    - podSelector:
        matchLabels:
          app: rabbitmq
    ports:
    - protocol: TCP
      port: 5672
  # อนุญาตเชื่อมต่อ Vault
  - to:
    - namespaceSelector:
        matchLabels:
          name: security
      podSelector:
        matchLabels:
          app: vault
    ports:
    - protocol: TCP
      port: 8200
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: payment-service-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: payment-service
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # อนุญาตเฉพาะ order-service
  - from:
    - podSelector:
        matchLabels:
          app: order-service
    ports:
    - protocol: TCP
      port: 3001
  egress:
  # Payment service เชื่อมต่อ external payment gateway
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 10.0.0.0/8
        - 172.16.0.0/12
        - 192.168.0.0/16
    ports:
    - protocol: TCP
      port: 443
  # Database
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432
```

---

## 7. Runtime Security ด้วย Falco

### 7.1 Falco Rules

```yaml
# falco/custom-rules.yaml
- rule: Unexpected Network Connection from Service
  desc: Detect unexpected outbound connections from services
  condition: >
    outbound and
    container and
    container.image.repository != "docker.io/library/curl" and
    not k8s.ns.name in (kube-system, monitoring) and
    not proc.name in (node, java, python, ruby) and
    fd.sport in (80, 443, 8080, 8443)
  output: >
    Unexpected network connection from service
    (user=%user.name command=%proc.cmdline
     connection=%fd.name container=%container.name
     image=%container.image.repository)
  priority: WARNING

- rule: Sensitive File Access
  desc: Detect access to sensitive files in containers
  condition: >
    open_read and
    container and
    fd.name in (/etc/passwd, /etc/shadow, /etc/sudoers,
                /proc/1/environ, /proc/self/environ)
  output: >
    Sensitive file accessed
    (user=%user.name command=%proc.cmdline
     file=%fd.name container=%container.name)
  priority: CRITICAL

- rule: Container Shell Spawned
  desc: Detect shell execution inside container
  condition: >
    spawned_process and
    container and
    proc.name in (bash, sh, dash, zsh, tcsh, csh) and
    container.image.repository != "debug-tools"
  output: >
    Shell spawned in container
    (user=%user.name shell=%proc.name
     args=%proc.args container=%container.name
     image=%container.image.repository)
  priority: WARNING

- rule: Privilege Escalation Attempt
  desc: Detect attempts to escalate privileges
  condition: >
    spawned_process and
    container and
    (proc.name = sudo or proc.name = su or
     proc.name = newgrp or
     (proc.name = python and proc.args contains "setuid"))
  output: >
    Privilege escalation attempt
    (user=%user.name command=%proc.cmdline
     container=%container.name)
  priority: CRITICAL

- rule: Cryptocurrency Mining Detected
  desc: Detect potential crypto mining activity
  condition: >
    spawned_process and
    (proc.name in (xmrig, cpuminer, minerd, cgminer,
                   bfgminer, nheqminer) or
     proc.args contains "stratum+tcp" or
     proc.args contains "stratum+ssl")
  output: >
    Cryptocurrency mining detected
    (user=%user.name command=%proc.cmdline
     container=%container.name)
  priority: CRITICAL

- rule: Kubernetes Secret Access
  desc: Detect unauthorized access to Kubernetes secrets
  condition: >
    ka.verb in (get, list, watch) and
    ka.target.resource = secrets and
    not ka.user.name in (system:serviceaccounts:kube-system)
  output: >
    K8s Secret accessed
    (user=%ka.user.name verb=%ka.verb
     resource=%ka.target.resource
     namespace=%ka.target.namespace)
  priority: WARNING
```

### 7.2 Falco Alert Handler

```typescript
// security/falco-alert-handler.ts
import express from 'express';
import { AlertManager } from './alert-manager';
import { IncidentManager } from './incident-manager';

interface FalcoAlert {
  priority: 'DEBUG' | 'INFO' | 'NOTICE' | 'WARNING' | 'ERROR' | 'CRITICAL' | 'ALERT' | 'EMERGENCY';
  rule: string;
  time: string;
  output: string;
  output_fields: Record<string, string>;
}

export class FalcoAlertHandler {
  constructor(
    private alertManager: AlertManager,
    private incidentManager: IncidentManager
  ) {}
  
  async handleAlert(alert: FalcoAlert): Promise<void> {
    console.log('Falco alert received:', {
      priority: alert.priority,
      rule: alert.rule,
      time: alert.time,
    });
    
    // ส่ง alert ตาม priority
    switch (alert.priority) {
      case 'CRITICAL':
      case 'ALERT':
      case 'EMERGENCY':
        await this.handleCriticalAlert(alert);
        break;
      case 'ERROR':
      case 'WARNING':
        await this.handleWarningAlert(alert);
        break;
      default:
        await this.handleInfoAlert(alert);
    }
  }
  
  private async handleCriticalAlert(alert: FalcoAlert): Promise<void> {
    // สร้าง incident
    const incident = await this.incidentManager.createIncident({
      title: `Security Alert: ${alert.rule}`,
      severity: 'critical',
      description: alert.output,
      metadata: alert.output_fields,
    });
    
    // ส่ง notification ด่วน
    await this.alertManager.sendPagerDuty({
      summary: `CRITICAL: ${alert.rule}`,
      severity: 'critical',
      details: alert.output,
      dedup_key: `falco-${alert.rule}-${alert.output_fields['container.name']}`,
    });
    
    // อาจ quarantine container ถ้ามีนโยบาย
    if (this.shouldQuarantine(alert)) {
      await this.quarantineContainer(alert.output_fields['container.id']);
    }
  }
  
  private async handleWarningAlert(alert: FalcoAlert): Promise<void> {
    await this.alertManager.sendSlack({
      channel: '#security-alerts',
      message: `⚠️ Security Warning: ${alert.rule}\n${alert.output}`,
    });
  }
  
  private async handleInfoAlert(alert: FalcoAlert): Promise<void> {
    // Log เท่านั้น
    console.info('Security info alert:', alert.rule);
  }
  
  private shouldQuarantine(alert: FalcoAlert): boolean {
    const quarantineRules = [
      'Cryptocurrency Mining Detected',
      'Privilege Escalation Attempt',
    ];
    return quarantineRules.includes(alert.rule);
  }
  
  private async quarantineContainer(containerId: string): Promise<void> {
    // ใช้ Kubernetes API ลบ Pod
    console.log(`Quarantining container: ${containerId}`);
    // ในการใช้งานจริง จะใช้ @kubernetes/client-node
  }
}

// HTTP endpoint สำหรับรับ Falco webhooks
export function createFalcoWebhookServer(handler: FalcoAlertHandler): express.Application {
  const app = express();
  app.use(express.json());
  
  app.post('/falco/webhook', async (req, res) => {
    try {
      const alert = req.body as FalcoAlert;
      await handler.handleAlert(alert);
      res.status(200).json({ status: 'ok' });
    } catch (error) {
      console.error('Error handling Falco alert:', error);
      res.status(500).json({ error: 'Internal server error' });
    }
  });
  
  return app;
}

// Placeholder interfaces
interface AlertManager {
  sendPagerDuty(params: any): Promise<void>;
  sendSlack(params: any): Promise<void>;
}

interface IncidentManager {
  createIncident(params: any): Promise<any>;
}
```

---

## 8. Supply Chain Security

### 8.1 Container Image Scanning

```yaml
# .github/workflows/security-scan.yaml
name: Security Scan Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  dependency-scan:
    name: Dependency Vulnerability Scan
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Run npm audit
      run: npm audit --production --audit-level=high
    
    - name: Run Snyk scan
      uses: snyk/actions/node@master
      env:
        SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
      with:
        args: --severity-threshold=high
    
    - name: OWASP Dependency Check
      uses: dependency-check/Dependency-Check_Action@main
      with:
        project: 'microservices'
        path: '.'
        format: 'HTML'
        out: 'reports'

  sast:
    name: Static Application Security Testing
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Run Semgrep
      uses: returntocorp/semgrep-action@v1
      with:
        config: >-
          p/security-audit
          p/secrets
          p/owasp-top-ten
    
    - name: ESLint Security Plugin
      run: |
        npm ci
        npx eslint . --ext .ts,.js \
          --plugin security \
          --rule 'security/detect-non-literal-fs-filename: error' \
          --rule 'security/detect-sql-injection: error'

  container-scan:
    name: Container Image Vulnerability Scan
    runs-on: ubuntu-latest
    needs: [dependency-scan, sast]
    steps:
    - uses: actions/checkout@v3
    
    - name: Build Docker image
      run: docker build -t order-service:${{ github.sha }} .
    
    - name: Run Trivy scan
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: 'order-service:${{ github.sha }}'
        format: 'sarif'
        output: 'trivy-results.sarif'
        severity: 'CRITICAL,HIGH'
        exit-code: '1'
    
    - name: Upload results to GitHub Security
      uses: github/codeql-action/upload-sarif@v2
      with:
        sarif_file: 'trivy-results.sarif'
    
    - name: Sign image with Cosign
      if: github.ref == 'refs/heads/main'
      uses: sigstore/cosign-installer@main
      with:
        cosign-release: 'v2.2.0'
    
    - name: Sign and push image
      if: github.ref == 'refs/heads/main'
      env:
        COSIGN_PRIVATE_KEY: ${{ secrets.COSIGN_PRIVATE_KEY }}
        COSIGN_PASSWORD: ${{ secrets.COSIGN_PASSWORD }}
      run: |
        # Push image
        docker tag order-service:${{ github.sha }} \
          registry.example.com/order-service:${{ github.sha }}
        docker push registry.example.com/order-service:${{ github.sha }}
        
        # Sign image
        cosign sign --key env://COSIGN_PRIVATE_KEY \
          registry.example.com/order-service:${{ github.sha }}

  secret-detection:
    name: Secret Detection
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
      with:
        fetch-depth: 0
    
    - name: Gitleaks scan
      uses: zricethezav/gitleaks-action@v2
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }}
```

### 8.2 SBOM Generation

```typescript
// security/sbom-generator.ts
import { execSync } from 'child_process';
import fs from 'fs';
import path from 'path';
import crypto from 'crypto';

interface SBOMComponent {
  name: string;
  version: string;
  type: 'npm' | 'container';
  licenses: string[];
  purl: string;
  hashes: {
    sha256: string;
  };
  vulnerabilities?: CVE[];
}

interface CVE {
  id: string;
  severity: 'critical' | 'high' | 'medium' | 'low';
  description: string;
  fixedVersion?: string;
}

interface SBOM {
  bomFormat: 'CycloneDX';
  specVersion: '1.4';
  serialNumber: string;
  version: number;
  metadata: {
    timestamp: string;
    tools: { name: string; version: string }[];
    component: {
      type: 'application';
      name: string;
      version: string;
    };
  };
  components: SBOMComponent[];
}

export class SBOMGenerator {
  async generate(options: {
    projectName: string;
    projectVersion: string;
    packageJsonPath: string;
  }): Promise<SBOM> {
    const packageJson = JSON.parse(
      fs.readFileSync(options.packageJsonPath, 'utf8')
    );
    
    const components = await this.getComponents(options.packageJsonPath);
    
    const sbom: SBOM = {
      bomFormat: 'CycloneDX',
      specVersion: '1.4',
      serialNumber: `urn:uuid:${crypto.randomUUID()}`,
      version: 1,
      metadata: {
        timestamp: new Date().toISOString(),
        tools: [
          { name: 'custom-sbom-generator', version: '1.0.0' }
        ],
        component: {
          type: 'application',
          name: options.projectName,
          version: options.projectVersion,
        }
      },
      components,
    };
    
    return sbom;
  }
  
  private async getComponents(packageJsonPath: string): Promise<SBOMComponent[]> {
    // ใช้ npm list เพื่อดู dependencies ทั้งหมด
    const listOutput = execSync(
      `npm list --json --production --prefix ${path.dirname(packageJsonPath)}`,
      { encoding: 'utf8', stdio: ['pipe', 'pipe', 'ignore'] }
    );
    
    const npmList = JSON.parse(listOutput);
    const components: SBOMComponent[] = [];
    
    const extractDeps = (deps: Record<string, any>, prefix = '') => {
      for (const [name, info] of Object.entries(deps || {})) {
        components.push({
          name,
          version: (info as any).version || 'unknown',
          type: 'npm',
          licenses: [(info as any).license || 'UNKNOWN'],
          purl: `pkg:npm/${name}@${(info as any).version}`,
          hashes: {
            sha256: this.computePackageHash(name, (info as any).version),
          }
        });
        
        if ((info as any).dependencies) {
          extractDeps((info as any).dependencies, `${prefix}${name}/`);
        }
      }
    };
    
    extractDeps(npmList.dependencies);
    return components;
  }
  
  private computePackageHash(name: string, version: string): string {
    try {
      const packagePath = `node_modules/${name}/package.json`;
      if (fs.existsSync(packagePath)) {
        const content = fs.readFileSync(packagePath);
        return crypto.createHash('sha256').update(content).digest('hex');
      }
    } catch {}
    return '';
  }
  
  async saveToFile(sbom: SBOM, outputPath: string): Promise<void> {
    fs.writeFileSync(outputPath, JSON.stringify(sbom, null, 2));
    console.log(`SBOM saved to ${outputPath}`);
  }
}
```

---

## 9. Security Testing Automation

### 9.1 OWASP ZAP Integration

```typescript
// security/zap-scanner.ts
import axios from 'axios';

interface ZAPScanResult {
  alerts: ZAPAlert[];
  summary: {
    high: number;
    medium: number;
    low: number;
    informational: number;
  };
}

interface ZAPAlert {
  riskcode: string;
  riskdesc: string;
  name: string;
  description: string;
  solution: string;
  url: string;
  param: string;
  evidence: string;
}

export class ZAPScanner {
  private zapApiUrl: string;
  
  constructor(zapUrl: string = 'http://localhost:8080') {
    this.zapApiUrl = `${zapUrl}/JSON`;
  }
  
  async spiderScan(targetUrl: string): Promise<void> {
    console.log(`Starting spider scan on ${targetUrl}...`);
    
    const response = await axios.get(`${this.zapApiUrl}/spider/action/scan/`, {
      params: { url: targetUrl, maxChildren: 5, recurse: true }
    });
    
    const scanId = response.data.scan;
    await this.waitForScan('spider', scanId);
  }
  
  async activeScan(targetUrl: string): Promise<string> {
    console.log(`Starting active scan on ${targetUrl}...`);
    
    const response = await axios.get(`${this.zapApiUrl}/ascan/action/scan/`, {
      params: { url: targetUrl, recurse: true, scanPolicyName: 'Default Policy' }
    });
    
    const scanId = response.data.scan;
    await this.waitForScan('ascan', scanId);
    return scanId;
  }
  
  async getResults(): Promise<ZAPScanResult> {
    const response = await axios.get(`${this.zapApiUrl}/core/view/alerts/`);
    const alerts: ZAPAlert[] = response.data.alerts;
    
    const summary = alerts.reduce((acc, alert) => {
      const risk = parseInt(alert.riskcode);
      if (risk === 3) acc.high++;
      else if (risk === 2) acc.medium++;
      else if (risk === 1) acc.low++;
      else acc.informational++;
      return acc;
    }, { high: 0, medium: 0, low: 0, informational: 0 });
    
    return { alerts, summary };
  }
  
  async generateReport(format: 'html' | 'json' | 'xml' = 'html'): Promise<string> {
    const endpoint = format === 'html' ? 'htmlreport' : `${format}report`;
    const response = await axios.get(`${this.zapApiUrl}/core/other/${endpoint}/`);
    return response.data;
  }
  
  private async waitForScan(type: 'spider' | 'ascan', scanId: string): Promise<void> {
    const statusEndpoint = type === 'spider'
      ? `${this.zapApiUrl}/spider/view/status/`
      : `${this.zapApiUrl}/ascan/view/status/`;
    
    while (true) {
      const response = await axios.get(statusEndpoint, {
        params: { scanId }
      });
      
      const progress = parseInt(response.data.status);
      console.log(`${type} scan progress: ${progress}%`);
      
      if (progress >= 100) break;
      await new Promise(resolve => setTimeout(resolve, 2000));
    }
  }
}

// security/run-security-tests.ts
async function runSecurityTests(targetUrl: string): Promise<void> {
  const scanner = new ZAPScanner();
  
  await scanner.spiderScan(targetUrl);
  await scanner.activeScan(targetUrl);
  
  const results = await scanner.getResults();
  
  console.log('Security scan complete:');
  console.log(`  High severity: ${results.summary.high}`);
  console.log(`  Medium severity: ${results.summary.medium}`);
  console.log(`  Low severity: ${results.summary.low}`);
  
  if (results.summary.high > 0) {
    console.error('FAIL: High severity vulnerabilities found!');
    process.exit(1);
  }
  
  const report = await scanner.generateReport('html');
  require('fs').writeFileSync('security-report.html', report);
  console.log('Security report saved to security-report.html');
}
```

---

## 10. Security Monitoring Dashboard

### 10.1 Security Metrics

```typescript
// monitoring/security-metrics.ts
import { Registry, Counter, Histogram, Gauge } from 'prom-client';

export class SecurityMetricsCollector {
  private registry: Registry;
  
  // Auth metrics
  private authAttempts: Counter;
  private authFailures: Counter;
  private tokenValidations: Counter;
  private tokenValidationErrors: Counter;
  
  // Access control metrics
  private accessDenials: Counter;
  private privilegedOperations: Counter;
  
  // Certificate metrics
  private certExpirySoon: Gauge;
  private certRotations: Counter;
  
  // Security incident metrics
  private securityAlerts: Counter;
  
  constructor() {
    this.registry = new Registry();
    this.initializeMetrics();
  }
  
  private initializeMetrics(): void {
    this.authAttempts = new Counter({
      name: 'auth_attempts_total',
      help: 'Total authentication attempts',
      labelNames: ['method', 'service'],
      registers: [this.registry],
    });
    
    this.authFailures = new Counter({
      name: 'auth_failures_total',
      help: 'Total authentication failures',
      labelNames: ['method', 'service', 'reason'],
      registers: [this.registry],
    });
    
    this.accessDenials = new Counter({
      name: 'access_denials_total',
      help: 'Total access denials',
      labelNames: ['resource', 'method', 'service'],
      registers: [this.registry],
    });
    
    this.certExpirySoon = new Gauge({
      name: 'cert_expiry_seconds',
      help: 'Seconds until certificate expires',
      labelNames: ['service', 'cert_type'],
      registers: [this.registry],
    });
    
    this.certRotations = new Counter({
      name: 'cert_rotations_total',
      help: 'Total certificate rotations',
      labelNames: ['service'],
      registers: [this.registry],
    });
    
    this.securityAlerts = new Counter({
      name: 'security_alerts_total',
      help: 'Total security alerts from Falco',
      labelNames: ['priority', 'rule'],
      registers: [this.registry],
    });
  }
  
  recordAuthAttempt(method: string, service: string): void {
    this.authAttempts.inc({ method, service });
  }
  
  recordAuthFailure(method: string, service: string, reason: string): void {
    this.authFailures.inc({ method, service, reason });
  }
  
  recordAccessDenial(resource: string, method: string, service: string): void {
    this.accessDenials.inc({ resource, method, service });
  }
  
  updateCertExpiry(service: string, certType: string, expiryDate: Date): void {
    const secondsUntilExpiry = (expiryDate.getTime() - Date.now()) / 1000;
    this.certExpirySoon.set({ service, cert_type: certType }, secondsUntilExpiry);
  }
  
  recordCertRotation(service: string): void {
    this.certRotations.inc({ service });
  }
  
  recordSecurityAlert(priority: string, rule: string): void {
    this.securityAlerts.inc({ priority, rule });
  }
  
  getMetrics(): Promise<string> {
    return this.registry.metrics();
  }
}
```

### 10.2 Grafana Dashboard Configuration

```json
{
  "dashboard": {
    "title": "Zero Trust Security Dashboard",
    "panels": [
      {
        "title": "Authentication Failure Rate",
        "type": "stat",
        "targets": [
          {
            "expr": "rate(auth_failures_total[5m]) / rate(auth_attempts_total[5m]) * 100",
            "legendFormat": "Failure Rate %"
          }
        ],
        "thresholds": {
          "steps": [
            { "color": "green", "value": 0 },
            { "color": "yellow", "value": 5 },
            { "color": "red", "value": 10 }
          ]
        }
      },
      {
        "title": "Access Denials by Resource",
        "type": "bar",
        "targets": [
          {
            "expr": "sum by (resource) (rate(access_denials_total[1h]))",
            "legendFormat": "{{resource}}"
          }
        ]
      },
      {
        "title": "Certificate Expiry Status",
        "type": "table",
        "targets": [
          {
            "expr": "cert_expiry_seconds",
            "legendFormat": "{{service}} - {{cert_type}}"
          }
        ]
      },
      {
        "title": "Security Alerts Timeline",
        "type": "timeseries",
        "targets": [
          {
            "expr": "sum by (priority) (rate(security_alerts_total[5m]))",
            "legendFormat": "{{priority}}"
          }
        ]
      }
    ]
  }
}
```

---

## สรุป

Zero Trust Security Architecture สำหรับ Microservices ประกอบด้วย:

| Component | Tool/Technology | หน้าที่ |
|-----------|----------------|---------|
| Identity Verification | JWT RS256 + JWKS | ยืนยันตัวตนทุก request |
| Service-to-Service Auth | mTLS | เข้ารหัสและยืนยันตัวตน service |
| Authorization | OPA (Rego Policies) | ตรวจสอบสิทธิ์แบบ policy-as-code |
| Secret Management | HashiCorp Vault | จัดการ secrets, certificates, credentials |
| Network Segmentation | Kubernetes NetworkPolicy | จำกัด traffic ระหว่าง services |
| Runtime Security | Falco | ตรวจจับ anomalies ใน runtime |
| Supply Chain | Trivy, Cosign, SBOM | ตรวจสอบ container images และ dependencies |
| Security Testing | OWASP ZAP, Semgrep | ทดสอบช่องโหว่อัตโนมัติ |

หลักการสำคัญ:
- **Verify Explicitly**: ตรวจสอบทุก request ด้วย identity, context, signals
- **Least Privilege Access**: ให้ access เฉพาะที่จำเป็น
- **Assume Breach**: ออกแบบราวกับว่า network ถูก compromise แล้ว
- **Continuous Monitoring**: monitor ตลอดเวลา ไม่ใช่แค่ตอน deploy

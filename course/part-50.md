# Part 50: Microservices Testing ขั้นสูง — TestContainers, Contract Testing, Performance, และ Chaos Engineering

การจัดการ Configuration และ Secrets เป็นหัวใจสำคัญของ Microservices ที่ทำงานใน Production
ระบบที่ดีต้องสามารถแยก Configuration ออกจาก Code ได้อย่างสมบูรณ์ และจัดการ Secrets
อย่างปลอดภัยโดยไม่ให้รั่วไหล บทนี้จะครอบคลุมเครื่องมือและ Pattern ที่ใช้งานจริงใน Enterprise

---

## 1. ConfigMap and Secret in Kubernetes

### ConfigMap คืออะไร

ConfigMap เป็น Kubernetes object สำหรับเก็บ Configuration data แบบ key-value
ที่ไม่ sensitive เช่น database host, port, feature flags

```yaml
# configmap-basic.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
  labels:
    app: order-service
    version: "1.0"
data:
  # Simple values
  DATABASE_HOST: "postgres.production.svc.cluster.local"
  DATABASE_PORT: "5432"
  DATABASE_NAME: "orders_db"
  REDIS_HOST: "redis.production.svc.cluster.local"
  REDIS_PORT: "6379"
  LOG_LEVEL: "info"
  MAX_CONNECTIONS: "100"
  
  # Multi-line configuration file
  app.properties: |
    server.port=3000
    server.timeout=30000
    cache.ttl=3600
    feature.new-checkout=true
    feature.loyalty-program=false
    
  # JSON configuration
  rate-limit.json: |
    {
      "windowMs": 900000,
      "max": 100,
      "message": "Too many requests",
      "standardHeaders": true,
      "legacyHeaders": false
    }
    
  # YAML configuration
  logging.yaml: |
    level: info
    format: json
    outputs:
      - console
      - file
    file:
      path: /var/log/app
      maxSize: 10m
      maxFiles: 5
```

### Secret สำหรับข้อมูล Sensitive

```yaml
# secret-basic.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: production
  annotations:
    # หมายเหตุ: ค่าเหล่านี้ใช้สำหรับ demo เท่านั้น
    managed-by: "external-secrets-operator"
type: Opaque
# ค่าต้องถูก base64 encoded
data:
  DATABASE_PASSWORD: cGFzc3dvcmQxMjM=  # password123
  JWT_SECRET: c3VwZXJzZWNyZXRqd3Rr  # supersecretjwtk
  API_KEY: YXBpa2V5MTIzNDU2  # apikey123456
  
stringData:
  # stringData จะถูก encode อัตโนมัติ
  SMTP_PASSWORD: "smtp-password-here"
  STRIPE_SECRET_KEY: "sk_test_xxxxx"
```

### การใช้ ConfigMap และ Secret ใน Pod

```yaml
# deployment-with-config.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
        - name: order-service
          image: registry.company.com/order-service:1.0.0
          ports:
            - containerPort: 3000
          
          # วิธีที่ 1: ใช้เป็น Environment Variables
          env:
            - name: DATABASE_HOST
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: DATABASE_HOST
            - name: DATABASE_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: DATABASE_PASSWORD
                  
          # วิธีที่ 2: โหลดทั้ง ConfigMap เป็น env vars
          envFrom:
            - configMapRef:
                name: app-config
            - secretRef:
                name: app-secrets
                
          # วิธีที่ 3: Mount เป็น Volume
          volumeMounts:
            - name: config-volume
              mountPath: /etc/config
              readOnly: true
            - name: secret-volume
              mountPath: /etc/secrets
              readOnly: true
              
      volumes:
        - name: config-volume
          configMap:
            name: app-config
            items:
              - key: app.properties
                path: app.properties
              - key: logging.yaml
                path: logging.yaml
        - name: secret-volume
          secret:
            secretName: app-secrets
            defaultMode: 0400  # read-only สำหรับ owner เท่านั้น
```

---

## 2. HashiCorp Vault Agent Injector

### การติดตั้ง Vault ด้วย Helm

```bash
# ติดตั้ง Vault ด้วย Helm
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update

# Install Vault
helm install vault hashicorp/vault \
  --namespace vault \
  --create-namespace \
  --set server.ha.enabled=true \
  --set server.ha.replicas=3 \
  --set injector.enabled=true
```

```yaml
# vault-values.yaml สำหรับ Production
server:
  ha:
    enabled: true
    replicas: 3
    raft:
      enabled: true
      config: |
        ui = true
        listener "tcp" {
          tls_disable = 1
          address = "[::]:8200"
          cluster_address = "[::]:8201"
        }
        storage "raft" {
          path = "/vault/data"
        }
        service_registration "kubernetes" {}
        
  resources:
    requests:
      memory: 256Mi
      cpu: 250m
    limits:
      memory: 256Mi
      
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app.kubernetes.io/name: vault
              component: server
          topologyKey: kubernetes.io/hostname

injector:
  enabled: true
  resources:
    requests:
      memory: 256Mi
      cpu: 250m
```

### Vault Policy และ Kubernetes Auth

```bash
# เปิด Kubernetes authentication
vault auth enable kubernetes

# Configure Kubernetes auth
vault write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
  token_reviewer_jwt=@/var/run/secrets/kubernetes.io/serviceaccount/token

# สร้าง policy สำหรับ order-service
vault policy write order-service - <<EOF
path "secret/data/production/order-service/*" {
  capabilities = ["read", "list"]
}
path "database/creds/order-service-role" {
  capabilities = ["read"]
}
path "pki/issue/order-service" {
  capabilities = ["create", "update"]
}
EOF

# สร้าง role
vault write auth/kubernetes/role/order-service \
  bound_service_account_names=order-service \
  bound_service_account_namespaces=production \
  policies=order-service \
  ttl=1h
```

### การใช้ Vault Agent Injector

```yaml
# deployment-with-vault.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
      annotations:
        # Vault Agent Injector annotations
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "order-service"
        vault.hashicorp.com/agent-inject-status: "update"
        
        # Inject database credentials
        vault.hashicorp.com/agent-inject-secret-db-creds: "database/creds/order-service-role"
        vault.hashicorp.com/agent-inject-template-db-creds: |
          {{- with secret "database/creds/order-service-role" -}}
          DATABASE_USERNAME={{ .Data.username }}
          DATABASE_PASSWORD={{ .Data.password }}
          {{- end }}
          
        # Inject application secrets
        vault.hashicorp.com/agent-inject-secret-app-secrets: "secret/data/production/order-service/app"
        vault.hashicorp.com/agent-inject-template-app-secrets: |
          {{- with secret "secret/data/production/order-service/app" -}}
          JWT_SECRET={{ .Data.data.jwt_secret }}
          STRIPE_SECRET_KEY={{ .Data.data.stripe_secret_key }}
          {{- end }}
          
    spec:
      serviceAccountName: order-service
      containers:
        - name: order-service
          image: registry.company.com/order-service:1.0.0
          command: ["/bin/sh", "-c"]
          args:
            - |
              # Source secrets จาก Vault-injected files
              export $(cat /vault/secrets/db-creds | xargs)
              export $(cat /vault/secrets/app-secrets | xargs)
              exec node dist/main.js
```

---

## 3. External Secrets Operator

### การติดตั้ง External Secrets Operator

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets \
  -n external-secrets-system \
  --create-namespace \
  --set installCRDs=true
```

### การเชื่อมต่อกับ AWS Secrets Manager

```yaml
# secret-store-aws.yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secrets-manager
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-southeast-1
      auth:
        # ใช้ IRSA (IAM Roles for Service Accounts)
        jwt:
          serviceAccountRef:
            name: external-secrets
            namespace: external-secrets-system
---
# external-secret.yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: order-service-secrets
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: order-service-secrets  # ชื่อ Kubernetes Secret ที่จะสร้าง
    creationPolicy: Owner
    deletionPolicy: Delete
    template:
      type: Opaque
      metadata:
        labels:
          managed-by: external-secrets
  data:
    - secretKey: DATABASE_PASSWORD
      remoteRef:
        key: production/order-service/database
        property: password
    - secretKey: JWT_SECRET
      remoteRef:
        key: production/order-service/jwt
        property: secret
  dataFrom:
    - extract:
        key: production/order-service/all-secrets
```

### การเชื่อมต่อกับ GCP Secret Manager

```yaml
# secret-store-gcp.yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: gcp-secret-manager
  namespace: production
spec:
  provider:
    gcpsm:
      projectID: my-project-id
      auth:
        workloadIdentity:
          clusterLocation: asia-southeast1
          clusterName: production-cluster
          serviceAccountRef:
            name: external-secrets-sa
---
# external-secret-gcp.yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: payment-service-secrets
  namespace: production
spec:
  refreshInterval: 30m
  secretStoreRef:
    name: gcp-secret-manager
    kind: SecretStore
  target:
    name: payment-service-secrets
    creationPolicy: Owner
  data:
    - secretKey: STRIPE_SECRET_KEY
      remoteRef:
        key: stripe-secret-key
        version: latest
    - secretKey: PAYPAL_CLIENT_SECRET
      remoteRef:
        key: paypal-client-secret
```

---

## 4. Sealed Secrets

### การติดตั้ง Sealed Secrets

```bash
# ติดตั้ง Sealed Secrets Controller
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm install sealed-secrets-controller sealed-secrets/sealed-secrets \
  --namespace kube-system

# ติดตั้ง kubeseal CLI
brew install kubeseal
# หรือ
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/kubeseal-0.24.0-linux-amd64.tar.gz
tar xfz kubeseal-0.24.0-linux-amd64.tar.gz
sudo install -m 755 kubeseal /usr/local/bin/kubeseal
```

### การสร้าง Sealed Secret

```bash
# สร้าง Secret ปกติก่อน (อย่า apply)
kubectl create secret generic my-secret \
  --from-literal=password=mysecretpassword \
  --dry-run=client \
  -o yaml > secret.yaml

# Seal ด้วย kubeseal
kubeseal \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=kube-system \
  --format yaml \
  < secret.yaml \
  > sealed-secret.yaml

# Apply Sealed Secret (safe to commit to git!)
kubectl apply -f sealed-secret.yaml
```

```yaml
# sealed-secret-example.yaml (safe to commit to git)
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: production-secrets
  namespace: production
spec:
  encryptedData:
    DATABASE_PASSWORD: AgBy8hJgT...encrypted...data
    JWT_SECRET: AgCx9kLhU...encrypted...data
    API_KEY: AgDz1mNiV...encrypted...data
  template:
    metadata:
      name: production-secrets
      namespace: production
    type: Opaque
```

---

## 5. TypeScript ConfigService พร้อม Zod Validation

### การสร้าง Config Schema

```typescript
// src/config/config.schema.ts
import { z } from 'zod';

const DatabaseConfigSchema = z.object({
  host: z.string().min(1, 'Database host is required'),
  port: z.coerce.number().int().min(1).max(65535).default(5432),
  name: z.string().min(1, 'Database name is required'),
  username: z.string().min(1, 'Database username is required'),
  password: z.string().min(1, 'Database password is required'),
  poolMin: z.coerce.number().int().min(0).default(2),
  poolMax: z.coerce.number().int().min(1).default(10),
  ssl: z.coerce.boolean().default(false),
});

const RedisConfigSchema = z.object({
  host: z.string().default('localhost'),
  port: z.coerce.number().int().default(6379),
  password: z.string().optional(),
  db: z.coerce.number().int().default(0),
  tls: z.coerce.boolean().default(false),
  keyPrefix: z.string().default('app:'),
});

const JwtConfigSchema = z.object({
  secret: z.string().min(32, 'JWT secret must be at least 32 characters'),
  expiresIn: z.string().default('1h'),
  refreshExpiresIn: z.string().default('7d'),
  algorithm: z.enum(['HS256', 'HS384', 'HS512', 'RS256']).default('HS256'),
});

const ServerConfigSchema = z.object({
  port: z.coerce.number().int().default(3000),
  host: z.string().default('0.0.0.0'),
  env: z.enum(['development', 'test', 'staging', 'production']).default('development'),
  logLevel: z.enum(['error', 'warn', 'info', 'debug', 'verbose']).default('info'),
  corsOrigins: z.string().transform(s => s.split(',')).default('*'),
  rateLimitWindowMs: z.coerce.number().int().default(900000),
  rateLimitMax: z.coerce.number().int().default(100),
});

const FeatureFlagsSchema = z.object({
  newCheckout: z.coerce.boolean().default(false),
  loyaltyProgram: z.coerce.boolean().default(false),
  experimentalApi: z.coerce.boolean().default(false),
  maintenanceMode: z.coerce.boolean().default(false),
});

export const AppConfigSchema = z.object({
  server: ServerConfigSchema,
  database: DatabaseConfigSchema,
  redis: RedisConfigSchema,
  jwt: JwtConfigSchema,
  features: FeatureFlagsSchema,
});

export type AppConfig = z.infer<typeof AppConfigSchema>;
export type DatabaseConfig = z.infer<typeof DatabaseConfigSchema>;
export type RedisConfig = z.infer<typeof RedisConfigSchema>;
```

### ConfigService Implementation

```typescript
// src/config/config.service.ts
import { Injectable, Logger, OnModuleInit } from '@nestjs/common';
import { EventEmitter } from 'events';
import * as fs from 'fs';
import * as path from 'path';
import { AppConfig, AppConfigSchema } from './config.schema';
import { ZodError } from 'zod';

interface ConfigChangeEvent {
  key: string;
  oldValue: unknown;
  newValue: unknown;
  timestamp: Date;
}

@Injectable()
export class ConfigService extends EventEmitter implements OnModuleInit {
  private readonly logger = new Logger(ConfigService.name);
  private config: AppConfig;
  private readonly configFilePath: string;
  private fileWatcher?: fs.FSWatcher;
  private reloadDebounceTimer?: NodeJS.Timeout;

  constructor() {
    super();
    this.configFilePath = process.env.CONFIG_FILE_PATH || path.join(process.cwd(), 'config', 'app.json');
  }

  async onModuleInit(): Promise<void> {
    await this.loadConfig();
    this.setupHotReload();
  }

  private async loadConfig(): Promise<void> {
    try {
      // โหลด config จาก environment variables
      const envConfig = this.loadFromEnvironment();
      
      // โหลด config จาก file ถ้ามี
      const fileConfig = await this.loadFromFile();
      
      // Merge โดย env vars มี priority สูงกว่า
      const rawConfig = this.deepMerge(fileConfig, envConfig);
      
      // Validate และ parse ด้วย Zod
      const result = AppConfigSchema.safeParse(rawConfig);
      
      if (!result.success) {
        const errors = this.formatZodErrors(result.error);
        this.logger.error('Configuration validation failed:', errors);
        throw new Error(`Invalid configuration: ${errors}`);
      }
      
      const oldConfig = this.config;
      this.config = result.data;
      
      if (oldConfig) {
        this.detectAndEmitChanges(oldConfig, this.config);
      }
      
      this.logger.log(`Configuration loaded successfully (env: ${this.config.server.env})`);
    } catch (error) {
      if (error instanceof Error) {
        this.logger.error('Failed to load configuration:', error.message);
        throw error;
      }
    }
  }

  private loadFromEnvironment(): Record<string, unknown> {
    return {
      server: {
        port: process.env.PORT,
        host: process.env.HOST,
        env: process.env.NODE_ENV,
        logLevel: process.env.LOG_LEVEL,
        corsOrigins: process.env.CORS_ORIGINS,
        rateLimitWindowMs: process.env.RATE_LIMIT_WINDOW_MS,
        rateLimitMax: process.env.RATE_LIMIT_MAX,
      },
      database: {
        host: process.env.DATABASE_HOST,
        port: process.env.DATABASE_PORT,
        name: process.env.DATABASE_NAME,
        username: process.env.DATABASE_USERNAME,
        password: process.env.DATABASE_PASSWORD,
        poolMin: process.env.DATABASE_POOL_MIN,
        poolMax: process.env.DATABASE_POOL_MAX,
        ssl: process.env.DATABASE_SSL,
      },
      redis: {
        host: process.env.REDIS_HOST,
        port: process.env.REDIS_PORT,
        password: process.env.REDIS_PASSWORD,
        db: process.env.REDIS_DB,
        tls: process.env.REDIS_TLS,
      },
      jwt: {
        secret: process.env.JWT_SECRET,
        expiresIn: process.env.JWT_EXPIRES_IN,
        refreshExpiresIn: process.env.JWT_REFRESH_EXPIRES_IN,
        algorithm: process.env.JWT_ALGORITHM,
      },
      features: {
        newCheckout: process.env.FEATURE_NEW_CHECKOUT,
        loyaltyProgram: process.env.FEATURE_LOYALTY_PROGRAM,
        experimentalApi: process.env.FEATURE_EXPERIMENTAL_API,
        maintenanceMode: process.env.FEATURE_MAINTENANCE_MODE,
      },
    };
  }

  private async loadFromFile(): Promise<Record<string, unknown>> {
    if (!fs.existsSync(this.configFilePath)) {
      return {};
    }
    
    try {
      const content = await fs.promises.readFile(this.configFilePath, 'utf-8');
      return JSON.parse(content);
    } catch (error) {
      this.logger.warn(`Failed to load config file: ${this.configFilePath}`);
      return {};
    }
  }

  private setupHotReload(): void {
    if (!fs.existsSync(this.configFilePath)) return;
    
    this.fileWatcher = fs.watch(this.configFilePath, (event) => {
      if (event === 'change') {
        // Debounce reload เพื่อป้องกัน multiple rapid changes
        if (this.reloadDebounceTimer) {
          clearTimeout(this.reloadDebounceTimer);
        }
        this.reloadDebounceTimer = setTimeout(async () => {
          this.logger.log('Config file changed, reloading...');
          await this.loadConfig();
        }, 500);
      }
    });
    
    this.logger.log(`Hot-reload enabled for: ${this.configFilePath}`);
  }

  private detectAndEmitChanges(oldConfig: AppConfig, newConfig: AppConfig): void {
    const changes = this.findChanges('', oldConfig as Record<string, unknown>, newConfig as Record<string, unknown>);
    
    for (const change of changes) {
      this.emit('config:changed', change);
      this.logger.log(`Config changed: ${change.key}`);
    }
  }

  private findChanges(
    prefix: string,
    oldObj: Record<string, unknown>,
    newObj: Record<string, unknown>
  ): ConfigChangeEvent[] {
    const changes: ConfigChangeEvent[] = [];
    
    const allKeys = new Set([...Object.keys(oldObj), ...Object.keys(newObj)]);
    
    for (const key of allKeys) {
      const fullKey = prefix ? `${prefix}.${key}` : key;
      const oldValue = oldObj[key];
      const newValue = newObj[key];
      
      if (typeof oldValue === 'object' && typeof newValue === 'object' && oldValue !== null && newValue !== null) {
        changes.push(
          ...this.findChanges(
            fullKey,
            oldValue as Record<string, unknown>,
            newValue as Record<string, unknown>
          )
        );
      } else if (JSON.stringify(oldValue) !== JSON.stringify(newValue)) {
        changes.push({
          key: fullKey,
          oldValue,
          newValue,
          timestamp: new Date(),
        });
      }
    }
    
    return changes;
  }

  private deepMerge(target: Record<string, unknown>, source: Record<string, unknown>): Record<string, unknown> {
    const result = { ...target };
    
    for (const key of Object.keys(source)) {
      const sourceValue = source[key];
      const targetValue = target[key];
      
      if (sourceValue === undefined || sourceValue === null || sourceValue === '') {
        continue;
      }
      
      if (
        typeof sourceValue === 'object' &&
        !Array.isArray(sourceValue) &&
        typeof targetValue === 'object' &&
        !Array.isArray(targetValue)
      ) {
        result[key] = this.deepMerge(
          targetValue as Record<string, unknown>,
          sourceValue as Record<string, unknown>
        );
      } else {
        result[key] = sourceValue;
      }
    }
    
    return result;
  }

  private formatZodErrors(error: ZodError): string {
    return error.errors
      .map(e => `${e.path.join('.')}: ${e.message}`)
      .join(', ');
  }

  // Getter methods
  get<K extends keyof AppConfig>(key: K): AppConfig[K] {
    return this.config[key];
  }

  get serverConfig() {
    return this.config.server;
  }

  get databaseConfig() {
    return this.config.database;
  }

  get redisConfig() {
    return this.config.redis;
  }

  get jwtConfig() {
    return this.config.jwt;
  }

  get features() {
    return this.config.features;
  }

  isFeatureEnabled(feature: keyof AppConfig['features']): boolean {
    return this.config.features[feature];
  }

  isProduction(): boolean {
    return this.config.server.env === 'production';
  }

  isDevelopment(): boolean {
    return this.config.server.env === 'development';
  }

  onConfigChange(handler: (event: ConfigChangeEvent) => void): void {
    this.on('config:changed', handler);
  }

  async destroy(): Promise<void> {
    if (this.fileWatcher) {
      this.fileWatcher.close();
    }
    if (this.reloadDebounceTimer) {
      clearTimeout(this.reloadDebounceTimer);
    }
  }
}
```

---

## 6. Feature Flags with LaunchDarkly-Style Implementation

### Feature Flag Service

```typescript
// src/feature-flags/feature-flag.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { Redis } from 'ioredis';
import { InjectRedis } from '@nestjs-modules/ioredis';

export interface FeatureFlag {
  key: string;
  enabled: boolean;
  rolloutPercentage?: number;  // 0-100
  userSegments?: string[];
  attributes?: Record<string, unknown>;
  expiresAt?: Date;
}

export interface EvaluationContext {
  userId?: string;
  userEmail?: string;
  userSegment?: string;
  country?: string;
  appVersion?: string;
  customAttributes?: Record<string, unknown>;
}

@Injectable()
export class FeatureFlagService {
  private readonly logger = new Logger(FeatureFlagService.name);
  private readonly FLAG_PREFIX = 'feature:flag:';
  private readonly FLAGS_SET_KEY = 'feature:flags:all';
  private localCache = new Map<string, { flag: FeatureFlag; cachedAt: number }>();
  private readonly CACHE_TTL_MS = 5000; // 5 seconds local cache

  constructor(@InjectRedis() private readonly redis: Redis) {}

  async isEnabled(flagKey: string, context?: EvaluationContext): Promise<boolean> {
    const flag = await this.getFlag(flagKey);
    
    if (!flag) {
      this.logger.debug(`Feature flag not found: ${flagKey}, defaulting to false`);
      return false;
    }
    
    return this.evaluate(flag, context);
  }

  private async getFlag(key: string): Promise<FeatureFlag | null> {
    // ตรวจสอบ local cache ก่อน
    const cached = this.localCache.get(key);
    if (cached && Date.now() - cached.cachedAt < this.CACHE_TTL_MS) {
      return cached.flag;
    }
    
    try {
      const data = await this.redis.get(`${this.FLAG_PREFIX}${key}`);
      if (!data) return null;
      
      const flag: FeatureFlag = JSON.parse(data);
      
      // อัพเดต local cache
      this.localCache.set(key, { flag, cachedAt: Date.now() });
      
      return flag;
    } catch (error) {
      this.logger.error(`Failed to get feature flag: ${key}`, error);
      return null;
    }
  }

  private evaluate(flag: FeatureFlag, context?: EvaluationContext): boolean {
    // ตรวจสอบ expiry
    if (flag.expiresAt && new Date() > new Date(flag.expiresAt)) {
      return false;
    }
    
    // ถ้าปิดอยู่ให้ return false เลย
    if (!flag.enabled) return false;
    
    // ตรวจสอบ user segments
    if (flag.userSegments && flag.userSegments.length > 0 && context?.userSegment) {
      if (!flag.userSegments.includes(context.userSegment)) {
        return false;
      }
    }
    
    // Rollout percentage
    if (flag.rolloutPercentage !== undefined && flag.rolloutPercentage < 100) {
      if (!context?.userId) return false;
      
      const hash = this.hashUserId(context.userId, flag.key);
      const percentile = hash % 100;
      
      if (percentile >= flag.rolloutPercentage) {
        return false;
      }
    }
    
    return true;
  }

  private hashUserId(userId: string, flagKey: string): number {
    // Simple hash function สำหรับ consistent rollout
    const str = `${flagKey}:${userId}`;
    let hash = 0;
    
    for (let i = 0; i < str.length; i++) {
      const char = str.charCodeAt(i);
      hash = ((hash << 5) - hash) + char;
      hash = hash & hash; // Convert to 32-bit integer
    }
    
    return Math.abs(hash);
  }

  async setFlag(flag: FeatureFlag): Promise<void> {
    const key = `${this.FLAG_PREFIX}${flag.key}`;
    await this.redis.set(key, JSON.stringify(flag));
    await this.redis.sadd(this.FLAGS_SET_KEY, flag.key);
    
    // Invalidate local cache
    this.localCache.delete(flag.key);
    
    // Publish change event
    await this.redis.publish('feature:flag:changed', JSON.stringify({
      key: flag.key,
      enabled: flag.enabled,
      timestamp: new Date(),
    }));
    
    this.logger.log(`Feature flag updated: ${flag.key} = ${flag.enabled}`);
  }

  async getAllFlags(): Promise<FeatureFlag[]> {
    const keys = await this.redis.smembers(this.FLAGS_SET_KEY);
    const flags: FeatureFlag[] = [];
    
    for (const key of keys) {
      const flag = await this.getFlag(key);
      if (flag) flags.push(flag);
    }
    
    return flags;
  }

  async deleteFlag(key: string): Promise<void> {
    await this.redis.del(`${this.FLAG_PREFIX}${key}`);
    await this.redis.srem(this.FLAGS_SET_KEY, key);
    this.localCache.delete(key);
  }

  // Decorator สำหรับ method
  featureGate(flagKey: string, fallback?: () => unknown) {
    return (target: unknown, propertyKey: string, descriptor: PropertyDescriptor) => {
      const originalMethod = descriptor.value;
      
      descriptor.value = async function (...args: unknown[]) {
        const isEnabled = await this.featureFlagService?.isEnabled(flagKey);
        
        if (!isEnabled) {
          if (fallback) return fallback();
          throw new Error(`Feature ${flagKey} is not enabled`);
        }
        
        return originalMethod.apply(this, args);
      };
      
      return descriptor;
    };
  }
}
```

### Feature Flag Controller

```typescript
// src/feature-flags/feature-flag.controller.ts
import { Controller, Get, Post, Delete, Body, Param } from '@nestjs/common';
import { FeatureFlagService, FeatureFlag, EvaluationContext } from './feature-flag.service';

@Controller('admin/feature-flags')
export class FeatureFlagController {
  constructor(private readonly featureFlagService: FeatureFlagService) {}

  @Get()
  async getAllFlags() {
    return this.featureFlagService.getAllFlags();
  }

  @Post()
  async setFlag(@Body() flag: FeatureFlag) {
    await this.featureFlagService.setFlag(flag);
    return { success: true, flag };
  }

  @Post(':key/evaluate')
  async evaluateFlag(
    @Param('key') key: string,
    @Body() context: EvaluationContext
  ) {
    const enabled = await this.featureFlagService.isEnabled(key, context);
    return { key, enabled, context };
  }

  @Delete(':key')
  async deleteFlag(@Param('key') key: string) {
    await this.featureFlagService.deleteFlag(key);
    return { success: true };
  }
}
```

---

## 7. Secrets Rotation Automation

### Secret Rotation Service

```typescript
// src/secrets/secret-rotation.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { Cron, CronExpression } from '@nestjs/schedule';
import * as crypto from 'crypto';

interface RotationResult {
  secretName: string;
  success: boolean;
  rotatedAt: Date;
  error?: string;
}

@Injectable()
export class SecretRotationService {
  private readonly logger = new Logger(SecretRotationService.name);

  // Rotate database passwords ทุก 30 วัน
  @Cron(CronExpression.EVERY_30_DAYS)
  async rotateDatabasePasswords(): Promise<void> {
    this.logger.log('Starting database password rotation...');
    
    const services = ['order-service', 'payment-service', 'user-service'];
    const results: RotationResult[] = [];
    
    for (const service of services) {
      try {
        const result = await this.rotateServiceDatabasePassword(service);
        results.push(result);
      } catch (error) {
        this.logger.error(`Failed to rotate password for ${service}:`, error);
        results.push({
          secretName: `${service}-db-password`,
          success: false,
          rotatedAt: new Date(),
          error: error instanceof Error ? error.message : 'Unknown error',
        });
      }
    }
    
    this.logRotationResults(results);
  }

  private async rotateServiceDatabasePassword(serviceName: string): Promise<RotationResult> {
    const secretName = `${serviceName}-db-password`;
    
    // 1. สร้าง password ใหม่
    const newPassword = this.generateSecurePassword(32);
    
    // 2. อัพเดต password ใน database
    await this.updateDatabasePassword(serviceName, newPassword);
    
    // 3. อัพเดต secret ใน Vault/K8s
    await this.updateVaultSecret(`production/${serviceName}/database`, {
      password: newPassword,
    });
    
    // 4. Trigger pod restart เพื่อโหลด credentials ใหม่
    await this.triggerPodRestart(serviceName);
    
    return {
      secretName,
      success: true,
      rotatedAt: new Date(),
    };
  }

  private generateSecurePassword(length: number): string {
    const charset = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789!@#$%^&*';
    const bytes = crypto.randomBytes(length);
    let password = '';
    
    for (let i = 0; i < length; i++) {
      password += charset[bytes[i] % charset.length];
    }
    
    return password;
  }

  private async updateDatabasePassword(serviceName: string, newPassword: string): Promise<void> {
    // Implementation จะแตกต่างกันตาม database
    this.logger.log(`Updating database password for ${serviceName}`);
    // await db.query(`ALTER USER ${serviceName} WITH PASSWORD '${newPassword}'`);
  }

  private async updateVaultSecret(path: string, data: Record<string, string>): Promise<void> {
    this.logger.log(`Updating Vault secret at ${path}`);
    // await vaultClient.write(`secret/data/${path}`, { data });
  }

  private async triggerPodRestart(serviceName: string): Promise<void> {
    this.logger.log(`Triggering rolling restart for ${serviceName}`);
    // await k8sApi.patchNamespacedDeployment(serviceName, 'production', patch);
  }

  private logRotationResults(results: RotationResult[]): void {
    const successful = results.filter(r => r.success).length;
    const failed = results.filter(r => !r.success).length;
    
    this.logger.log(`Rotation complete: ${successful} successful, ${failed} failed`);
    
    if (failed > 0) {
      const failures = results
        .filter(r => !r.success)
        .map(r => `${r.secretName}: ${r.error}`)
        .join(', ');
      this.logger.error(`Failed rotations: ${failures}`);
    }
  }
}
```

---

## 8. Environment-Specific Configurations

### การจัดการ Config หลาย Environment

```typescript
// src/config/environment.config.ts
export const environments = {
  development: {
    logLevel: 'debug',
    database: {
      ssl: false,
      poolMax: 5,
    },
    redis: {
      db: 0,
    },
    features: {
      experimentalApi: true,
    },
  },
  
  staging: {
    logLevel: 'info',
    database: {
      ssl: true,
      poolMax: 10,
    },
    redis: {
      db: 1,
    },
    features: {
      experimentalApi: true,
    },
  },
  
  production: {
    logLevel: 'warn',
    database: {
      ssl: true,
      poolMax: 50,
    },
    redis: {
      db: 0,
      tls: true,
    },
    features: {
      experimentalApi: false,
    },
  },
} as const;
```

```yaml
# k8s/overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

namespace: production

patches:
  - path: deployment-patch.yaml
  - path: configmap-patch.yaml

configMapGenerator:
  - name: app-config
    behavior: merge
    literals:
      - LOG_LEVEL=warn
      - MAX_CONNECTIONS=100

images:
  - name: order-service
    newTag: "1.2.3"
```

```yaml
# k8s/base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 1
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
        - name: order-service
          image: order-service:latest
          envFrom:
            - configMapRef:
                name: app-config
            - secretRef:
                name: app-secrets
```

---

## 9. Config Module for NestJS

```typescript
// src/config/config.module.ts
import { Module, Global } from '@nestjs/common';
import { ConfigService } from './config.service';
import { FeatureFlagService } from '../feature-flags/feature-flag.service';
import { FeatureFlagController } from '../feature-flags/feature-flag.controller';
import { SecretRotationService } from '../secrets/secret-rotation.service';
import { ScheduleModule } from '@nestjs/schedule';

@Global()
@Module({
  imports: [ScheduleModule.forRoot()],
  providers: [ConfigService, FeatureFlagService, SecretRotationService],
  controllers: [FeatureFlagController],
  exports: [ConfigService, FeatureFlagService],
})
export class ConfigModule {}
```

```typescript
// src/main.ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { ConfigService } from './config/config.service';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  
  const configService = app.get(ConfigService);
  
  // Listen สำหรับ config changes
  configService.onConfigChange((event) => {
    console.log(`Config changed: ${event.key}`, {
      old: event.oldValue,
      new: event.newValue,
      at: event.timestamp,
    });
    
    // Reload ส่วนที่เกี่ยวข้อง
    if (event.key.startsWith('features.')) {
      console.log('Feature flag changed, updating behavior...');
    }
  });
  
  const port = configService.serverConfig.port;
  await app.listen(port);
  console.log(`Application is running on port ${port}`);
}

bootstrap();
```

---

## 10. Docker Compose สำหรับ Local Development

```yaml
# docker-compose.yml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: app_user
      POSTGRES_PASSWORD: dev_password
      POSTGRES_DB: orders_db
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    command: redis-server --requirepass dev_redis_password
    ports:
      - "6379:6379"

  vault:
    image: vault:1.15
    cap_add:
      - IPC_LOCK
    environment:
      VAULT_DEV_ROOT_TOKEN_ID: dev-root-token
      VAULT_DEV_LISTEN_ADDRESS: 0.0.0.0:8200
    ports:
      - "8200:8200"
    command: vault server -dev

  vault-init:
    image: vault:1.15
    depends_on:
      - vault
    environment:
      VAULT_ADDR: http://vault:8200
      VAULT_TOKEN: dev-root-token
    command: |
      sh -c "
        sleep 2
        vault kv put secret/development/order-service/app \
          jwt_secret=dev-jwt-secret-for-development-only \
          stripe_secret_key=sk_test_dev
        vault kv put secret/development/order-service/database \
          password=dev_password
        echo 'Vault initialized!'
      "

  order-service:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    environment:
      NODE_ENV: development
      DATABASE_HOST: postgres
      DATABASE_PORT: 5432
      DATABASE_NAME: orders_db
      DATABASE_USERNAME: app_user
      DATABASE_PASSWORD: dev_password
      REDIS_HOST: redis
      REDIS_PORT: 6379
      REDIS_PASSWORD: dev_redis_password
      VAULT_ADDR: http://vault:8200
      VAULT_TOKEN: dev-root-token
    depends_on:
      - postgres
      - redis
      - vault-init

volumes:
  postgres_data:
```

---

## สรุป

| หัวข้อ | เครื่องมือ | ประโยชน์ |
|--------|-----------|---------|
| ConfigMap | Kubernetes | จัดการ non-sensitive config |
| Secret | Kubernetes | จัดการ sensitive data แบบ encrypted |
| Vault Agent | HashiCorp Vault | Dynamic secrets, auto-rotation |
| External Secrets | External Secrets Operator | Sync จาก AWS/GCP/Azure |
| Sealed Secrets | Bitnami | Encrypt secrets สำหรับ Git |
| ConfigService | TypeScript + Zod | Type-safe config พร้อม validation |
| Feature Flags | Redis-based | Toggle features โดยไม่ต้อง deploy |
| Hot-reload | fs.watch | Update config โดยไม่ต้อง restart |
| Secret Rotation | @nestjs/schedule | Rotate secrets อัตโนมัติ |
| Environment Config | Kustomize | จัดการ config หลาย environment |

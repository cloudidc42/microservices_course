# Part 55: Configuration Management — Kubernetes Secrets, External Secrets, Hot-reload, และ Feature Flags

ในบทนี้เราจะเรียนรู้การจัดการ Configuration ใน Microservices อย่างครบถ้วน ตั้งแต่ Kubernetes ConfigMaps/Secrets, External Secrets Operator, Hot-reload Config, Feature Flags ด้วย Flagsmith, ไปจนถึง Config Validation

---

## 1. Kubernetes ConfigMaps และ Secrets Best Practices

### 1.1 ConfigMap สำหรับ Non-sensitive Config

```yaml
# kubernetes/config/order-service-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: order-service-config
  namespace: production
  labels:
    app: order-service
    version: "1.5.0"
  annotations:
    config.kubernetes.io/last-applied-configuration: "managed-by-gitops"
data:
  # Application config
  APP_PORT: "3000"
  LOG_LEVEL: "info"
  LOG_FORMAT: "json"
  NODE_ENV: "production"
  SERVICE_NAME: "order-service"
  SERVICE_VERSION: "1.5.0"
  
  # Feature flags (non-sensitive)
  FEATURE_NEW_CHECKOUT: "true"
  FEATURE_LOYALTY_POINTS: "false"
  
  # Performance tuning
  DB_POOL_MIN: "5"
  DB_POOL_MAX: "20"
  DB_STATEMENT_TIMEOUT: "30000"
  REDIS_POOL_SIZE: "10"
  
  # Timeouts
  HTTP_TIMEOUT_MS: "30000"
  EXTERNAL_SERVICE_TIMEOUT_MS: "5000"
  
  # Business rules
  MAX_ORDER_ITEMS: "50"
  MAX_ORDER_VALUE_USD: "10000"
  ORDER_EXPIRY_MINUTES: "30"
  
  # OpenTelemetry
  OTEL_COLLECTOR_URL: "http://otel-collector.observability:4318"
  OTEL_SAMPLE_RATE: "0.1"
  
  # Mounted as file for complex config
  app-config.json: |
    {
      "rateLimiting": {
        "windowMs": 900000,
        "maxRequests": 100,
        "skipSuccessfulRequests": false
      },
      "cors": {
        "allowedOrigins": [
          "https://app.example.com",
          "https://admin.example.com"
        ],
        "allowedMethods": ["GET", "POST", "PUT", "DELETE"]
      },
      "pagination": {
        "defaultLimit": 20,
        "maxLimit": 100
      }
    }

---
# Separate ConfigMap for database config
apiVersion: v1
kind: ConfigMap
metadata:
  name: order-service-db-config
  namespace: production
data:
  DB_HOST: "postgres-primary.database:5432"
  DB_READ_HOST: "postgres-replica.database:5432"
  DB_NAME: "orders"
  DB_SSL_MODE: "require"
  DB_SSL_CA_CERT_PATH: "/etc/ssl/certs/rds-ca.pem"
```

### 1.2 Secrets Management Best Practices

```yaml
# kubernetes/secrets/order-service-secrets.yaml
# NEVER commit actual secrets to git
# Use sealed-secrets or external-secrets instead

# Option 1: Sealed Secrets (encrypted at rest in git)
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: order-service-secrets
  namespace: production
spec:
  encryptedData:
    # Encrypted with cluster's public key
    DB_PASSWORD: AgBy8hCl...truncated...
    JWT_SECRET: AgCX8pqr...truncated...
    STRIPE_SECRET_KEY: AgDY7mnp...truncated...

---
# Option 2: ExternalSecret (recommended for production)
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
    name: order-service-secrets
    creationPolicy: Owner
    template:
      type: Opaque
      metadata:
        labels:
          app: order-service
      data:
        # Template to transform secret values if needed
        DATABASE_URL: "postgresql://{{ .username }}:{{ .password }}@{{ .host }}/{{ .database }}?ssl=true"
  data:
    - secretKey: DB_PASSWORD
      remoteRef:
        key: production/order-service/database
        property: password
    - secretKey: DB_USERNAME
      remoteRef:
        key: production/order-service/database
        property: username
    - secretKey: JWT_SECRET
      remoteRef:
        key: production/order-service/jwt
        property: secret
    - secretKey: STRIPE_SECRET_KEY
      remoteRef:
        key: production/order-service/stripe
        property: secret_key
    - secretKey: REDIS_PASSWORD
      remoteRef:
        key: production/infrastructure/redis
        property: password
  dataFrom:
    # Bulk import all fields from a secret
    - extract:
        key: production/order-service/oauth
        conversionStrategy: Default
        decodingStrategy: None
```

### 1.3 Deployment ที่ใช้ ConfigMap และ Secrets

```yaml
# kubernetes/deployments/order-service.yaml
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
        version: "1.5.0"
      annotations:
        # Force pod restart when config changes
        checksum/config: "{{ include (print $.Template.BasePath '/configmap.yaml') . | sha256sum }}"
        checksum/secret: "{{ include (print $.Template.BasePath '/secret.yaml') . | sha256sum }}"
    spec:
      containers:
        - name: order-service
          image: order-service:1.5.0
          ports:
            - containerPort: 3000
          
          # Environment variables from ConfigMap
          envFrom:
            - configMapRef:
                name: order-service-config
            - configMapRef:
                name: order-service-db-config
            - secretRef:
                name: order-service-secrets
          
          # Additional specific env vars
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: NODE_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.hostIP
          
          # Mount config file
          volumeMounts:
            - name: app-config
              mountPath: /app/config
              readOnly: true
            - name: ssl-certs
              mountPath: /etc/ssl/certs
              readOnly: true
          
          resources:
            limits:
              cpu: 500m
              memory: 512Mi
            requests:
              cpu: 100m
              memory: 128Mi
          
          livenessProbe:
            httpGet:
              path: /health/live
              port: 3000
            initialDelaySeconds: 15
            periodSeconds: 20
          
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 10
      
      volumes:
        - name: app-config
          configMap:
            name: order-service-config
            items:
              - key: app-config.json
                path: app-config.json
        - name: ssl-certs
          secret:
            secretName: rds-ca-certs
```

---

## 2. External Secrets Operator กับ AWS Secrets Manager

### 2.1 ClusterSecretStore Setup

```yaml
# kubernetes/external-secrets/cluster-secret-store.yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secrets-manager
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      # Use IRSA (IAM Roles for Service Accounts)
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
            namespace: external-secrets

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: external-secrets-sa
  namespace: external-secrets
  annotations:
    # IRSA annotation pointing to IAM role
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/external-secrets-role
```

### 2.2 IAM Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSecretManagerRead",
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret",
        "secretsmanager:ListSecretVersionIds"
      ],
      "Resource": [
        "arn:aws:secretsmanager:us-east-1:123456789:secret:production/*"
      ]
    },
    {
      "Sid": "AllowKMSDecrypt",
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt",
        "kms:DescribeKey"
      ],
      "Resource": [
        "arn:aws:kms:us-east-1:123456789:key/your-kms-key-id"
      ]
    }
  ]
}
```

### 2.3 TypeScript Secret Manager Client

```typescript
// src/config/secrets-manager.ts
import {
  SecretsManagerClient,
  GetSecretValueCommand,
  ListSecretVersionIdsCommand,
} from '@aws-sdk/client-secrets-manager';
import NodeCache from 'node-cache';

const secretsCache = new NodeCache({ stdTTL: 3600, checkperiod: 600 });

export class SecretsManager {
  private readonly client: SecretsManagerClient;
  
  constructor() {
    this.client = new SecretsManagerClient({
      region: process.env.AWS_REGION || 'us-east-1',
      // Uses IRSA in Kubernetes, instance profile on EC2, or env vars locally
    });
  }
  
  async getSecret<T = Record<string, string>>(secretId: string): Promise<T> {
    const cached = secretsCache.get<T>(secretId);
    if (cached !== undefined) {
      return cached;
    }
    
    try {
      const command = new GetSecretValueCommand({ SecretId: secretId });
      const response = await this.client.send(command);
      
      let secret: T;
      if (response.SecretString) {
        try {
          secret = JSON.parse(response.SecretString) as T;
        } catch {
          secret = response.SecretString as unknown as T;
        }
      } else if (response.SecretBinary) {
        secret = Buffer.from(response.SecretBinary).toString('base64') as unknown as T;
      } else {
        throw new Error(`Secret ${secretId} has no value`);
      }
      
      secretsCache.set(secretId, secret);
      return secret;
    } catch (error) {
      console.error(`Failed to retrieve secret ${secretId}:`, error);
      throw error;
    }
  }
  
  async invalidateCache(secretId: string): Promise<void> {
    secretsCache.del(secretId);
  }
}

export const secretsManager = new SecretsManager();
```

---

## 3. Hot-reload Config โดยไม่ Restart

```typescript
// src/config/hot-reload.ts
import fs from 'fs';
import path from 'path';
import { EventEmitter } from 'events';
import chokidar from 'chokidar';
import { z } from 'zod';

const AppConfigSchema = z.object({
  rateLimiting: z.object({
    windowMs: z.number().int().positive(),
    maxRequests: z.number().int().positive(),
    skipSuccessfulRequests: z.boolean(),
  }),
  cors: z.object({
    allowedOrigins: z.array(z.string().url()),
    allowedMethods: z.array(z.string()),
  }),
  pagination: z.object({
    defaultLimit: z.number().int().positive().max(100),
    maxLimit: z.number().int().positive().max(1000),
  }),
  features: z.record(z.boolean()).optional().default({}),
});

type AppConfig = z.infer<typeof AppConfigSchema>;

class ConfigManager extends EventEmitter {
  private config: AppConfig;
  private readonly configPath: string;
  private watcher?: chokidar.FSWatcher;
  
  constructor(configPath: string) {
    super();
    this.configPath = path.resolve(configPath);
    this.config = this.loadConfig();
  }
  
  private loadConfig(): AppConfig {
    try {
      const raw = fs.readFileSync(this.configPath, 'utf-8');
      const parsed = JSON.parse(raw);
      const validated = AppConfigSchema.parse(parsed);
      console.log(`Config loaded from ${this.configPath}`);
      return validated;
    } catch (error) {
      console.error(`Failed to load config from ${this.configPath}:`, error);
      
      if (this.config) {
        console.warn('Using previous config due to load error');
        return this.config;
      }
      
      throw error;
    }
  }
  
  startWatching(): void {
    this.watcher = chokidar.watch(this.configPath, {
      persistent: true,
      ignoreInitial: true,
      awaitWriteFinish: {
        stabilityThreshold: 500,
        pollInterval: 100,
      },
    });
    
    this.watcher.on('change', (filePath) => {
      console.log(`Config file changed: ${filePath}`);
      
      const oldConfig = { ...this.config };
      const newConfig = this.loadConfig();
      
      if (JSON.stringify(oldConfig) !== JSON.stringify(newConfig)) {
        this.config = newConfig;
        this.emit('config:changed', newConfig, oldConfig);
        console.log('Config reloaded successfully');
      }
    });
    
    this.watcher.on('error', (error) => {
      console.error('Config watcher error:', error);
    });
    
    console.log(`Watching config file: ${this.configPath}`);
  }
  
  async stopWatching(): Promise<void> {
    if (this.watcher) {
      await this.watcher.close();
      this.watcher = undefined;
    }
  }
  
  get<K extends keyof AppConfig>(key: K): AppConfig[K] {
    return this.config[key];
  }
  
  getAll(): Readonly<AppConfig> {
    return Object.freeze({ ...this.config });
  }
  
  isFeatureEnabled(featureName: string): boolean {
    return this.config.features?.[featureName] ?? false;
  }
}

export const configManager = new ConfigManager(
  process.env.APP_CONFIG_PATH || '/app/config/app-config.json'
);

// Start watching in production
if (process.env.NODE_ENV === 'production') {
  configManager.startWatching();
}

// Log config changes
configManager.on('config:changed', (newConfig: AppConfig, oldConfig: AppConfig) => {
  console.log('Config change detected:', {
    changes: detectChanges(oldConfig, newConfig),
    timestamp: new Date().toISOString(),
  });
});

function detectChanges(old: any, next: any, path = ''): string[] {
  const changes: string[] = [];
  
  for (const key of Object.keys({ ...old, ...next })) {
    const fullPath = path ? `${path}.${key}` : key;
    
    if (JSON.stringify(old[key]) !== JSON.stringify(next[key])) {
      changes.push(fullPath);
    }
  }
  
  return changes;
}

// Hot-reload middleware for rate limiting
export function createHotReloadableRateLimiter() {
  let currentLimiter = createRateLimiter(configManager.get('rateLimiting'));
  
  configManager.on('config:changed', (newConfig: AppConfig) => {
    currentLimiter = createRateLimiter(newConfig.rateLimiting);
    console.log('Rate limiter config updated', newConfig.rateLimiting);
  });
  
  return (req: express.Request, res: express.Response, next: express.NextFunction) => {
    currentLimiter(req, res, next);
  };
}

function createRateLimiter(config: AppConfig['rateLimiting']) {
  return rateLimit({
    windowMs: config.windowMs,
    max: config.maxRequests,
    skipSuccessfulRequests: config.skipSuccessfulRequests,
    standardHeaders: true,
    legacyHeaders: false,
  });
}
```

---

## 4. Feature Flags ด้วย Flagsmith (Self-hosted)

### 4.1 Flagsmith Kubernetes Deployment

```yaml
# kubernetes/flagsmith/flagsmith.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flagsmith
  namespace: flagsmith
spec:
  replicas: 2
  selector:
    matchLabels:
      app: flagsmith
  template:
    metadata:
      labels:
        app: flagsmith
    spec:
      containers:
        - name: flagsmith
          image: flagsmith/flagsmith:2.95.0
          ports:
            - containerPort: 8000
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: flagsmith-secrets
                  key: DATABASE_URL
            - name: SECRET_KEY
              valueFrom:
                secretKeyRef:
                  name: flagsmith-secrets
                  key: SECRET_KEY
            - name: DJANGO_ALLOWED_HOSTS
              value: "flagsmith.internal.example.com,flagsmith.flagsmith"
            - name: REDIS_URL
              valueFrom:
                secretKeyRef:
                  name: flagsmith-secrets
                  key: REDIS_URL
            - name: ALLOW_REGISTRATION_WITHOUT_INVITE
              value: "false"
            - name: ENABLE_TELEMETRY
              value: "false"
          resources:
            limits:
              cpu: 500m
              memory: 512Mi
            requests:
              cpu: 100m
              memory: 256Mi
```

### 4.2 Feature Flag Service TypeScript

```typescript
// src/config/feature-flags.ts
import Flagsmith from 'flagsmith-nodejs';
import { log } from '../telemetry/logger';

interface FeatureFlag {
  enabled: boolean;
  value?: string | number | boolean | null;
}

interface FlagsmithConfig {
  environmentKey: string;
  apiUrl?: string;
  enableLocalEvaluation?: boolean;
  environmentRefreshIntervalSeconds?: number;
}

class FeatureFlagService {
  private client: ReturnType<typeof Flagsmith.init> | null = null;
  private localCache = new Map<string, FeatureFlag>();
  private initialized = false;
  
  async initialize(config: FlagsmithConfig): Promise<void> {
    this.client = Flagsmith.init({
      environmentKey: config.environmentKey,
      apiUrl: config.apiUrl || 'https://flagsmith.internal.example.com/api/v1/',
      enableLocalEvaluation: config.enableLocalEvaluation ?? true,
      environmentRefreshIntervalSeconds: config.environmentRefreshIntervalSeconds ?? 60,
      onEnvironmentChange: (flags) => {
        log.info('Feature flags updated', {
          flagCount: Object.keys(flags.flags).length,
        });
      },
    });
    
    await this.client.init();
    this.initialized = true;
    
    log.info('Feature flag service initialized');
  }
  
  async isEnabled(featureName: string, userId?: string): Promise<boolean> {
    if (!this.initialized || !this.client) {
      return this.localCache.get(featureName)?.enabled ?? false;
    }
    
    try {
      if (userId) {
        const flags = await this.client.getIdentityFlags(userId);
        return flags.isFeatureEnabled(featureName);
      }
      
      const flags = await this.client.getEnvironmentFlags();
      return flags.isFeatureEnabled(featureName);
    } catch (error) {
      log.error('Failed to get feature flag', error as Error, { featureName });
      // Fall back to cache or default
      return this.localCache.get(featureName)?.enabled ?? false;
    }
  }
  
  async getValue<T = string>(
    featureName: string,
    defaultValue: T,
    userId?: string
  ): Promise<T> {
    if (!this.initialized || !this.client) {
      return (this.localCache.get(featureName)?.value as T) ?? defaultValue;
    }
    
    try {
      let flags;
      if (userId) {
        flags = await this.client.getIdentityFlags(userId);
      } else {
        flags = await this.client.getEnvironmentFlags();
      }
      
      const value = flags.getFeatureValue(featureName);
      return (value as T) ?? defaultValue;
    } catch (error) {
      log.error('Failed to get feature flag value', error as Error, { featureName });
      return defaultValue;
    }
  }
  
  // Segment-based targeting
  async isEnabledForUser(
    featureName: string,
    user: {
      id: string;
      email?: string;
      plan?: string;
      country?: string;
      createdAt?: Date;
    }
  ): Promise<boolean> {
    if (!this.initialized || !this.client) {
      return false;
    }
    
    const traits: Record<string, string | number | boolean> = {};
    if (user.email) traits['email'] = user.email;
    if (user.plan) traits['plan'] = user.plan;
    if (user.country) traits['country'] = user.country;
    if (user.createdAt) {
      traits['days_since_signup'] = Math.floor(
        (Date.now() - user.createdAt.getTime()) / 86400000
      );
    }
    
    const flags = await this.client.getIdentityFlags(user.id, traits);
    return flags.isFeatureEnabled(featureName);
  }
  
  // Local fallback for testing
  setLocalFlag(name: string, enabled: boolean, value?: string): void {
    this.localCache.set(name, { enabled, value });
  }
  
  clearLocalFlags(): void {
    this.localCache.clear();
  }
}

export const featureFlags = new FeatureFlagService();

// Initialize on startup
featureFlags.initialize({
  environmentKey: process.env.FLAGSMITH_ENVIRONMENT_KEY!,
  apiUrl: process.env.FLAGSMITH_API_URL,
  enableLocalEvaluation: true,
  environmentRefreshIntervalSeconds: 60,
}).catch(err => {
  console.error('Failed to initialize feature flags:', err);
  // Non-fatal: service continues with defaults
});

// Feature flag middleware
export function featureFlagMiddleware(flagName: string, fallback?: (req: any, res: any) => void) {
  return async (req: express.Request, res: express.Response, next: express.NextFunction) => {
    const userId = req.user?.id;
    const isEnabled = await featureFlags.isEnabled(flagName, userId);
    
    if (!isEnabled) {
      if (fallback) {
        return fallback(req, res);
      }
      return res.status(404).json({ error: 'feature_not_available' });
    }
    
    next();
  };
}

// Usage example:
// app.post('/api/orders/express-checkout',
//   authenticate(),
//   featureFlagMiddleware('express-checkout'),
//   expressCheckoutHandler
// );
```

---

## 5. Environment-specific Config ด้วย dotenv-flow

```typescript
// src/config/env.ts
import 'dotenv-flow/config';
import { z } from 'zod';

// dotenv-flow loads:
// .env (base)
// .env.local (local overrides, gitignored)
// .env.{NODE_ENV} (e.g., .env.production, .env.test)
// .env.{NODE_ENV}.local

const EnvironmentSchema = z.object({
  // Server
  NODE_ENV: z.enum(['development', 'test', 'staging', 'production']),
  PORT: z.coerce.number().int().min(1024).max(65535).default(3000),
  SERVICE_NAME: z.string().min(1),
  SERVICE_VERSION: z.string().regex(/^\d+\.\d+\.\d+/).default('0.0.0'),
  
  // Database
  DATABASE_URL: z.string().url().refine(
    url => url.startsWith('postgresql://') || url.startsWith('postgres://'),
    'Must be a PostgreSQL URL'
  ),
  DB_POOL_MIN: z.coerce.number().int().min(1).default(2),
  DB_POOL_MAX: z.coerce.number().int().min(1).default(10),
  DB_STATEMENT_TIMEOUT: z.coerce.number().int().positive().default(30000),
  DB_IDLE_TIMEOUT: z.coerce.number().int().positive().default(10000),
  
  // Redis
  REDIS_URL: z.string().url(),
  REDIS_POOL_SIZE: z.coerce.number().int().min(1).default(10),
  
  // Auth
  JWT_SECRET: z.string().min(32, 'JWT secret must be at least 32 characters'),
  JWT_EXPIRES_IN: z.string().default('1h'),
  OAUTH_ISSUER: z.string().url(),
  OAUTH_AUDIENCE: z.string().url(),
  
  // External services
  PRODUCT_SERVICE_URL: z.string().url(),
  PAYMENT_SERVICE_URL: z.string().url(),
  NOTIFICATION_SERVICE_URL: z.string().url().optional(),
  
  // Message broker
  RABBITMQ_URL: z.string().url(),
  RABBITMQ_EXCHANGE: z.string().default('orders'),
  
  // Observability
  OTEL_COLLECTOR_URL: z.string().url().optional(),
  LOG_LEVEL: z.enum(['debug', 'info', 'warn', 'error']).default('info'),
  LOG_FORMAT: z.enum(['json', 'pretty']).default('json'),
  
  // Feature flags
  FLAGSMITH_ENVIRONMENT_KEY: z.string().min(1).optional(),
  FLAGSMITH_API_URL: z.string().url().optional(),
  
  // AWS (for Secrets Manager)
  AWS_REGION: z.string().default('us-east-1'),
  
  // API keys
  PAGERDUTY_ROUTING_KEY: z.string().optional(),
  API_KEY_HMAC_SECRET: z.string().min(32),
  
  // Metrics
  METRICS_AUTH_TOKEN: z.string().optional(),
}).superRefine((data, ctx) => {
  // Production-specific validation
  if (data.NODE_ENV === 'production') {
    if (!data.PAGERDUTY_ROUTING_KEY) {
      ctx.addIssue({
        code: z.ZodIssueCode.custom,
        message: 'PAGERDUTY_ROUTING_KEY is required in production',
        path: ['PAGERDUTY_ROUTING_KEY'],
      });
    }
    
    if (!data.OTEL_COLLECTOR_URL) {
      ctx.addIssue({
        code: z.ZodIssueCode.custom,
        message: 'OTEL_COLLECTOR_URL is required in production',
        path: ['OTEL_COLLECTOR_URL'],
      });
    }
    
    if (!data.FLAGSMITH_ENVIRONMENT_KEY) {
      ctx.addIssue({
        code: z.ZodIssueCode.custom,
        message: 'FLAGSMITH_ENVIRONMENT_KEY is required in production',
        path: ['FLAGSMITH_ENVIRONMENT_KEY'],
      });
    }
  }
});

type Env = z.infer<typeof EnvironmentSchema>;

let _env: Env | null = null;

export function getEnv(): Env {
  if (_env) return _env;
  
  const result = EnvironmentSchema.safeParse(process.env);
  
  if (!result.success) {
    const errors = result.error.errors.map(e =>
      `  ${e.path.join('.')}: ${e.message}`
    ).join('\n');
    
    throw new Error(`Invalid environment configuration:\n${errors}`);
  }
  
  _env = result.data;
  return _env;
}

// Validate immediately on import in production
if (process.env.NODE_ENV === 'production') {
  try {
    getEnv();
    console.log('Environment configuration validated successfully');
  } catch (error) {
    console.error('FATAL: Invalid environment configuration');
    console.error(error);
    process.exit(1);
  }
}

export type { Env };
```

---

## 6. Config Validation ด้วย AJV

```typescript
// src/config/ajv-validator.ts
import Ajv, { JSONSchemaType } from 'ajv';
import addFormats from 'ajv-formats';
import addKeywords from 'ajv-keywords';

const ajv = new Ajv({
  allErrors: true,
  coerceTypes: true,
  useDefaults: true,
  strict: true,
});

addFormats(ajv);
addKeywords(ajv, ['transform']);

interface DatabaseConfig {
  host: string;
  port: number;
  name: string;
  poolMin: number;
  poolMax: number;
  statementTimeout: number;
  ssl: {
    enabled: boolean;
    caPath?: string;
  };
}

const databaseConfigSchema: JSONSchemaType<DatabaseConfig> = {
  type: 'object',
  properties: {
    host: { type: 'string', minLength: 1 },
    port: { type: 'integer', minimum: 1024, maximum: 65535, default: 5432 },
    name: { type: 'string', minLength: 1 },
    poolMin: { type: 'integer', minimum: 1, maximum: 10, default: 2 },
    poolMax: { type: 'integer', minimum: 1, maximum: 50, default: 10 },
    statementTimeout: { type: 'integer', minimum: 1000, default: 30000 },
    ssl: {
      type: 'object',
      properties: {
        enabled: { type: 'boolean', default: true },
        caPath: { type: 'string', nullable: true },
      },
      required: ['enabled'],
      additionalProperties: false,
    },
  },
  required: ['host', 'name', 'ssl'],
  additionalProperties: false,
};

const validateDatabaseConfig = ajv.compile(databaseConfigSchema);

export function parseDatabaseConfig(raw: unknown): DatabaseConfig {
  const data = JSON.parse(JSON.stringify(raw)); // Deep clone for coercion
  
  if (!validateDatabaseConfig(data)) {
    const errors = ajv.errorsText(validateDatabaseConfig.errors, { separator: '\n' });
    throw new Error(`Invalid database config:\n${errors}`);
  }
  
  // Additional cross-field validation
  if (data.poolMin > data.poolMax) {
    throw new Error('poolMin must be <= poolMax');
  }
  
  return data;
}

// Runtime config update validation
export class ConfigValidator {
  private readonly validators = new Map<string, ReturnType<typeof ajv.compile>>();
  
  register<T>(name: string, schema: object): void {
    this.validators.set(name, ajv.compile(schema));
  }
  
  validate<T>(name: string, data: unknown): T {
    const validator = this.validators.get(name);
    if (!validator) {
      throw new Error(`No validator registered for: ${name}`);
    }
    
    const cloned = JSON.parse(JSON.stringify(data));
    
    if (!validator(cloned)) {
      const errors = ajv.errorsText(validator.errors, {
        separator: '\n',
        dataVar: name,
      });
      throw new Error(`Validation failed for ${name}:\n${errors}`);
    }
    
    return cloned as T;
  }
}
```

---

## 7. Config Hierarchy và Override Pattern

```typescript
// src/config/config-hierarchy.ts
import { getEnv } from './env';
import { secretsManager } from './secrets-manager';
import { configManager } from './hot-reload';
import { featureFlags } from './feature-flags';

// Unified config access point
export class AppConfiguration {
  private static instance: AppConfiguration;
  private readonly env = getEnv();
  
  static getInstance(): AppConfiguration {
    if (!this.instance) {
      this.instance = new AppConfiguration();
    }
    return this.instance;
  }
  
  // Database config with secrets
  async getDatabaseConfig() {
    const secret = await secretsManager.getSecret<{
      username: string;
      password: string;
    }>(`production/${this.env.SERVICE_NAME}/database`);
    
    return {
      connectionString: `postgresql://${secret.username}:${secret.password}@${this.env.DATABASE_URL}`,
      pool: {
        min: this.env.DB_POOL_MIN,
        max: this.env.DB_POOL_MAX,
        idleTimeoutMillis: this.env.DB_IDLE_TIMEOUT,
        statementTimeout: this.env.DB_STATEMENT_TIMEOUT,
      },
      ssl: this.env.NODE_ENV === 'production' ? { rejectUnauthorized: true } : false,
    };
  }
  
  // Runtime feature flag
  async isFeatureEnabled(flag: string, userId?: string): Promise<boolean> {
    // Priority: runtime flag > env var override > flagsmith
    const envOverride = process.env[`FEATURE_${flag.toUpperCase().replace(/-/g, '_')}`];
    if (envOverride !== undefined) {
      return envOverride === 'true';
    }
    
    return featureFlags.isEnabled(flag, userId);
  }
  
  // Dynamic config from file with hot-reload
  getRateLimitConfig() {
    return configManager.get('rateLimiting');
  }
  
  getCorsConfig() {
    return configManager.get('cors');
  }
  
  // Static config from environment
  getServiceConfig() {
    return {
      name: this.env.SERVICE_NAME,
      version: this.env.SERVICE_VERSION,
      port: this.env.PORT,
      environment: this.env.NODE_ENV,
    };
  }
}

export const appConfig = AppConfiguration.getInstance();
```

---

## สรุป

ในบทนี้เราได้เรียนรู้ Configuration Management อย่างครบถ้วน:

1. **Kubernetes ConfigMaps/Secrets** — Best practices สำหรับ non-sensitive config ใน ConfigMap, Sealed Secrets สำหรับ encrypted secrets ใน git

2. **External Secrets Operator** — Integration กับ AWS Secrets Manager ผ่าน IRSA, refreshInterval, และ template transformation

3. **Hot-reload Config** — chokidar/fs.watch สำหรับ config file changes โดยไม่ต้อง restart, validation ก่อน apply

4. **Feature Flags** — Flagsmith self-hosted deployment บน Kubernetes, TypeScript SDK พร้อม local evaluation, segment-based targeting

5. **dotenv-flow** — Environment-specific config hierarchy (.env → .env.local → .env.production) พร้อม Zod validation

6. **Config Validation** — AJV สำหรับ JSON Schema validation, cross-field validation, runtime config updates

7. **Config Hierarchy** — Unified access point ที่รวม secrets, dynamic config, และ feature flags ด้วย clear priority rules

Key takeaways:
- แยก sensitive config (secrets) กับ non-sensitive config (configmap) ให้ชัดเจน
- Validate ทุก config ตั้งแต่ startup เพื่อ fail fast แทนที่จะ fail ใน runtime
- Hot-reload ช่วยให้ tune parameters ใน production ได้โดยไม่ต้อง redeploy
- Feature flags เป็น tool ที่ทรงพลังสำหรับ progressive rollouts และ A/B testing
- ใช้ External Secrets Operator แทนการเก็บ secrets ใน Kubernetes โดยตรง

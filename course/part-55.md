# Part 55: Configuration Management

## บทนำ

Configuration Management ใน Microservices เป็นเรื่องสำคัญมาก เพราะ Service หลายสิบตัวต้องจัดการ Config ที่แตกต่างกันในแต่ละ Environment (dev/staging/production) ปัญหาที่พบบ่อยคือ Config Drift, Secret Management, และการ Reload Config โดยไม่ต้อง Restart Service

---

## 1. Kubernetes ConfigMaps

### 1.1 สร้าง ConfigMap

```yaml
# k8s/configmaps/order-service.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: order-service-config
  namespace: microservices
  labels:
    app: order-service
    managed-by: helm
data:
  # Simple key-value
  LOG_LEVEL: "info"
  MAX_CONNECTIONS: "100"
  REQUEST_TIMEOUT_MS: "30000"
  ENABLE_METRICS: "true"
  
  # JSON config
  rate-limit.json: |
    {
      "windowMs": 60000,
      "maxRequests": 1000,
      "skipFailedRequests": false,
      "keyBy": "userId"
    }
  
  # YAML config
  features.yaml: |
    features:
      new_checkout_flow: false
      loyalty_points: true
      express_delivery: true
      cod_payment: false
    
    ab_tests:
      checkout_v2:
        enabled: true
        percentage: 20
```

### 1.2 Mount ConfigMap ใน Pod

```yaml
# k8s/deployments/order-service.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: microservices
spec:
  template:
    spec:
      volumes:
        - name: config
          configMap:
            name: order-service-config
            items:
              - key: rate-limit.json
                path: rate-limit.json
              - key: features.yaml
                path: features.yaml
      containers:
        - name: order-service
          image: myrepo/order-service:1.0.0
          env:
            # Environment variables from ConfigMap
            - name: LOG_LEVEL
              valueFrom:
                configMapKeyRef:
                  name: order-service-config
                  key: LOG_LEVEL
            - name: MAX_CONNECTIONS
              valueFrom:
                configMapKeyRef:
                  name: order-service-config
                  key: MAX_CONNECTIONS
          volumeMounts:
            - name: config
              mountPath: /app/config
              readOnly: true
```

---

## 2. Kubernetes Secrets

```yaml
# k8s/secrets/order-service.yaml
apiVersion: v1
kind: Secret
metadata:
  name: order-service-secrets
  namespace: microservices
  annotations:
    # ใช้ External Secrets Operator annotations
    externalsecrets.io/source: aws-secrets-manager
type: Opaque
stringData:
  DATABASE_URL: "postgresql://user:pass@postgres:5432/orders"
  JWT_SECRET: "change-me-in-production"
  STRIPE_SECRET_KEY: "sk_live_..."
---
# RBAC สำหรับ Service Account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: order-service-sa
  namespace: microservices
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/order-service-role
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: order-service-role
  namespace: microservices
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["order-service-config"]
    verbs: ["get", "watch", "list"]
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["order-service-secrets"]
    verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: order-service-role-binding
  namespace: microservices
subjects:
  - kind: ServiceAccount
    name: order-service-sa
roleRef:
  kind: Role
  name: order-service-role
  apiGroup: rbac.authorization.k8s.io
```

---

## 3. External Secrets Operator กับ AWS Secrets Manager

```yaml
# k8s/external-secrets/secret-store.yaml
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
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
            namespace: external-secrets
---
# ExternalSecret สำหรับ Order Service
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: order-service-secrets
  namespace: microservices
spec:
  refreshInterval: 5m  # ดึงใหม่ทุก 5 นาที
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: order-service-secrets
    creationPolicy: Owner
    deletionPolicy: Retain
    template:
      type: Opaque
      data:
        DATABASE_URL: "{{ .db_url }}"
        JWT_SECRET: "{{ .jwt_secret }}"
        STRIPE_SECRET_KEY: "{{ .stripe_key }}"
  data:
    - secretKey: db_url
      remoteRef:
        key: microservices/production/order-service
        property: database_url
    - secretKey: jwt_secret
      remoteRef:
        key: microservices/production/shared
        property: jwt_secret
    - secretKey: stripe_key
      remoteRef:
        key: microservices/production/payment
        property: stripe_secret_key
```

### 3.1 AWS Secrets Manager Client

```typescript
// packages/shared/src/config/secrets.manager.ts

import {
  SecretsManagerClient,
  GetSecretValueCommand,
  DescribeSecretCommand,
} from '@aws-sdk/client-secrets-manager';

interface SecretCache {
  value: Record<string, string>;
  expiresAt: number;
  versionId: string;
}

export class SecretsManager {
  private readonly client: SecretsManagerClient;
  private readonly cache: Map<string, SecretCache> = new Map();
  private readonly ttlMs: number;

  constructor(
    region = process.env.AWS_REGION ?? 'ap-southeast-1',
    cacheTtlMs = 5 * 60 * 1000 // 5 minutes
  ) {
    this.client = new SecretsManagerClient({ region });
    this.ttlMs = cacheTtlMs;
  }

  async getSecret(secretName: string): Promise<Record<string, string>> {
    const cached = this.cache.get(secretName);
    if (cached && Date.now() < cached.expiresAt) {
      return cached.value;
    }

    const command = new GetSecretValueCommand({
      SecretId: secretName,
      VersionStage: 'AWSCURRENT',
    });

    const response = await this.client.send(command);
    const secretString = response.SecretString;

    if (!secretString) {
      throw new Error(`Secret ${secretName} has no string value`);
    }

    const value = JSON.parse(secretString) as Record<string, string>;

    this.cache.set(secretName, {
      value,
      expiresAt: Date.now() + this.ttlMs,
      versionId: response.VersionId ?? '',
    });

    return value;
  }

  async getSecretValue(secretName: string, key: string): Promise<string> {
    const secret = await this.getSecret(secretName);
    const value = secret[key];
    if (!value) {
      throw new Error(`Key ${key} not found in secret ${secretName}`);
    }
    return value;
  }

  // Rotate secret — invalidate cache
  invalidateCache(secretName: string): void {
    this.cache.delete(secretName);
  }

  invalidateAllCaches(): void {
    this.cache.clear();
  }
}
```

---

## 4. Config Hot-Reloading

```typescript
// packages/shared/src/config/config.watcher.ts
// Watch ConfigMap files สำหรับ hot-reload

import fs from 'fs';
import path from 'path';
import { EventEmitter } from 'events';
import yaml from 'js-yaml';

type ConfigValue = string | number | boolean | Record<string, unknown> | ConfigValue[];

interface ConfigChangeEvent {
  key: string;
  oldValue: ConfigValue;
  newValue: ConfigValue;
}

export class ConfigWatcher extends EventEmitter {
  private readonly watchers: Map<string, fs.FSWatcher> = new Map();
  private readonly configs: Map<string, ConfigValue> = new Map();
  private readonly debounceTimers: Map<string, ReturnType<typeof setTimeout>> = new Map();

  constructor(private readonly configDir: string) {
    super();
  }

  watch(filename: string, key: string): void {
    const filePath = path.join(this.configDir, filename);

    // Initial load
    this.loadFile(filePath, key);

    // Watch for changes (Kubernetes updates ConfigMap files atomically)
    const watcher = fs.watch(filePath, { persistent: false }, (event) => {
      if (event === 'change' || event === 'rename') {
        // Debounce เพราะ Kubernetes อาจ write หลายครั้ง
        const existing = this.debounceTimers.get(key);
        if (existing) clearTimeout(existing);

        this.debounceTimers.set(
          key,
          setTimeout(() => {
            this.loadFile(filePath, key);
            this.debounceTimers.delete(key);
          }, 100)
        );
      }
    });

    this.watchers.set(key, watcher);
    console.log(`[ConfigWatcher] Watching: ${filePath} as "${key}"`);
  }

  get<T = ConfigValue>(key: string): T {
    return this.configs.get(key) as T;
  }

  stopAll(): void {
    for (const [key, watcher] of this.watchers) {
      watcher.close();
      this.watchers.delete(key);
    }
    for (const timer of this.debounceTimers.values()) {
      clearTimeout(timer);
    }
    this.debounceTimers.clear();
  }

  private loadFile(filePath: string, key: string): void {
    try {
      const content = fs.readFileSync(filePath, 'utf-8');
      const ext = path.extname(filePath).toLowerCase();

      let newValue: ConfigValue;

      if (ext === '.json') {
        newValue = JSON.parse(content) as Record<string, unknown>;
      } else if (ext === '.yaml' || ext === '.yml') {
        newValue = yaml.load(content) as Record<string, unknown>;
      } else {
        newValue = content.trim();
      }

      const oldValue = this.configs.get(key);
      this.configs.set(key, newValue);

      if (oldValue !== undefined) {
        const event: ConfigChangeEvent = { key, oldValue, newValue };
        this.emit('change', event);
        this.emit(`change:${key}`, event);
        console.log(`[ConfigWatcher] Config changed: ${key}`);
      } else {
        this.emit('loaded', { key, value: newValue });
      }
    } catch (error) {
      console.error(`[ConfigWatcher] Error loading ${filePath}:`, error);
      this.emit('error', { key, error });
    }
  }
}

// ─── Application Config Manager ───────────────────────────────────────────

export class AppConfigManager {
  private readonly watcher: ConfigWatcher;
  private readonly rateLimitListeners: Array<(config: RateLimitConfig) => void> = [];
  private readonly featureFlagListeners: Array<(config: FeatureFlagsConfig) => void> = [];

  constructor(configDir = process.env.CONFIG_DIR ?? '/app/config') {
    this.watcher = new ConfigWatcher(configDir);

    this.watcher.watch('rate-limit.json', 'rateLimit');
    this.watcher.watch('features.yaml', 'features');

    this.watcher.on('change:rateLimit', ({ newValue }) => {
      const config = newValue as RateLimitConfig;
      console.log('[Config] Rate limit config updated:', config);
      this.rateLimitListeners.forEach((fn) => fn(config));
    });

    this.watcher.on('change:features', ({ newValue }) => {
      const config = newValue as FeatureFlagsConfig;
      console.log('[Config] Feature flags updated:', config);
      this.featureFlagListeners.forEach((fn) => fn(config));
    });
  }

  getRateLimitConfig(): RateLimitConfig {
    return this.watcher.get<RateLimitConfig>('rateLimit') ?? defaultRateLimitConfig;
  }

  getFeatureFlags(): FeatureFlagsConfig {
    return this.watcher.get<FeatureFlagsConfig>('features') ?? defaultFeatureFlags;
  }

  isFeatureEnabled(featureName: string): boolean {
    const flags = this.getFeatureFlags();
    return flags.features?.[featureName] === true;
  }

  onRateLimitChange(listener: (config: RateLimitConfig) => void): void {
    this.rateLimitListeners.push(listener);
  }

  onFeatureFlagsChange(listener: (config: FeatureFlagsConfig) => void): void {
    this.featureFlagListeners.push(listener);
  }

  destroy(): void {
    this.watcher.stopAll();
  }
}

interface RateLimitConfig {
  windowMs: number;
  maxRequests: number;
  skipFailedRequests: boolean;
  keyBy: string;
}

interface FeatureFlagsConfig {
  features: Record<string, boolean>;
  ab_tests: Record<string, { enabled: boolean; percentage: number }>;
}

const defaultRateLimitConfig: RateLimitConfig = {
  windowMs: 60000,
  maxRequests: 1000,
  skipFailedRequests: false,
  keyBy: 'ip',
};

const defaultFeatureFlags: FeatureFlagsConfig = {
  features: {},
  ab_tests: {},
};
```

---

## 5. Feature Flags กับ Flagsmith

```typescript
// packages/shared/src/feature-flags/flagsmith.client.ts

import Flagsmith from 'flagsmith-nodejs';

interface FeatureFlagContext {
  userId?: string;
  email?: string;
  traits?: Record<string, string | number | boolean>;
}

export class FeatureFlagService {
  private readonly flagsmith: Flagsmith;
  private initialized = false;

  constructor(
    private readonly environmentKey: string,
    options: {
      apiUrl?: string;
      cacheTtlSeconds?: number;
      enableLocalEvaluation?: boolean;
    } = {}
  ) {
    this.flagsmith = new Flagsmith({
      environmentKey,
      apiUrl: options.apiUrl ?? 'https://edge.api.flagsmith.com/api/v1/',
      enableLocalEvaluation: options.enableLocalEvaluation ?? true,
      environmentRefreshIntervalSeconds: options.cacheTtlSeconds ?? 60,
      onEnvironmentChange: (oldEnv, newEnv) => {
        console.log('[Flagsmith] Environment updated');
      },
    });
  }

  async initialize(): Promise<void> {
    if (!this.initialized) {
      await this.flagsmith.init();
      this.initialized = true;
      console.log('[Flagsmith] Initialized');
    }
  }

  async isEnabled(
    featureName: string,
    context?: FeatureFlagContext
  ): Promise<boolean> {
    await this.initialize();

    if (context?.userId) {
      const flags = await this.flagsmith.getIdentityFlags(
        context.userId,
        context.traits
      );
      return flags.isFeatureEnabled(featureName);
    }

    const flags = await this.flagsmith.getEnvironmentFlags();
    return flags.isFeatureEnabled(featureName);
  }

  async getValue<T = unknown>(
    featureName: string,
    context?: FeatureFlagContext
  ): Promise<T | null> {
    await this.initialize();

    if (context?.userId) {
      const flags = await this.flagsmith.getIdentityFlags(
        context.userId,
        context.traits
      );
      return flags.getFeatureValue(featureName) as T | null;
    }

    const flags = await this.flagsmith.getEnvironmentFlags();
    return flags.getFeatureValue(featureName) as T | null;
  }

  // A/B Test helper
  async getABTestVariant(
    testName: string,
    userId: string,
    variants: string[] = ['control', 'treatment']
  ): Promise<string> {
    const value = await this.getValue<string>(testName, { userId });
    if (value && variants.includes(value)) {
      return value;
    }

    // Deterministic assignment based on userId hash
    const hash = userId.split('').reduce((acc, char) => acc + char.charCodeAt(0), 0);
    return variants[hash % variants.length];
  }
}

// Express Middleware สำหรับ Feature Flags
import { Request, Response, NextFunction } from 'express';

export function createFeatureFlagMiddleware(flagService: FeatureFlagService) {
  return (req: Request, _res: Response, next: NextFunction) => {
    // @ts-expect-error — adding custom property
    req.isFeatureEnabled = async (
      featureName: string
    ): Promise<boolean> => {
      // @ts-expect-error — reading user from auth middleware
      const userId = req.user?.id;
      return flagService.isEnabled(featureName, { userId });
    };

    next();
  };
}
```

---

## 6. LaunchDarkly Integration

```typescript
// packages/shared/src/feature-flags/launchdarkly.client.ts

import * as LaunchDarkly from '@launchdarkly/node-server-sdk';

interface LDUser {
  key: string;
  email?: string;
  name?: string;
  country?: string;
  custom?: Record<string, LaunchDarkly.LDFlagValue>;
}

export class LaunchDarklyService {
  private client!: LaunchDarkly.LDClient;
  private ready = false;

  constructor(private readonly sdkKey: string) {}

  async initialize(): Promise<void> {
    this.client = LaunchDarkly.init(this.sdkKey, {
      logger: LaunchDarkly.basicLogger({
        level: 'warn',
        destination: console.error,
      }),
      // Use streaming for real-time updates
      stream: true,
      // Offline mode for testing
      offline: process.env.NODE_ENV === 'test',
    });

    await this.client.waitForInitialization({ timeout: 10 });
    this.ready = true;
    console.log('[LaunchDarkly] SDK initialized');

    // Listen for flag changes
    this.client.on('update', (settings) => {
      console.log(`[LaunchDarkly] Flag updated:`, Object.keys(settings.updates));
    });
  }

  async variation<T extends LaunchDarkly.LDFlagValue>(
    flagKey: string,
    user: LDUser,
    defaultValue: T
  ): Promise<T> {
    if (!this.ready) {
      console.warn('[LaunchDarkly] Not initialized, returning default');
      return defaultValue;
    }

    return this.client.variation(flagKey, user, defaultValue) as T;
  }

  async boolVariation(
    flagKey: string,
    user: LDUser,
    defaultValue = false
  ): Promise<boolean> {
    return this.client.boolVariation(flagKey, user, defaultValue);
  }

  async stringVariation(
    flagKey: string,
    user: LDUser,
    defaultValue = ''
  ): Promise<string> {
    return this.client.stringVariation(flagKey, user, defaultValue);
  }

  async jsonVariation<T>(
    flagKey: string,
    user: LDUser,
    defaultValue: T
  ): Promise<T> {
    return this.client.jsonVariation(flagKey, user, defaultValue) as T;
  }

  // Get all flags for a user (for debugging/inspection)
  async allFlagsState(user: LDUser): Promise<Record<string, LaunchDarkly.LDFlagValue>> {
    const state = this.client.allFlagsState(user);
    return state.toValuesMap();
  }

  async close(): Promise<void> {
    await this.client.close();
  }
}
```

---

## 7. Config Validation ด้วย JSON Schema / Zod

```typescript
// packages/shared/src/config/config.validator.ts

import { z } from 'zod';

// ─── Schema Definitions ────────────────────────────────────────────────────

const DatabaseConfigSchema = z.object({
  host: z.string().min(1),
  port: z.number().int().min(1).max(65535).default(5432),
  database: z.string().min(1),
  user: z.string().min(1),
  password: z.string().min(1),
  ssl: z.boolean().default(false),
  maxConnections: z.number().int().min(1).max(1000).default(20),
  connectionTimeoutMs: z.number().int().min(100).default(5000),
  idleTimeoutMs: z.number().int().min(1000).default(30000),
});

const RedisConfigSchema = z.object({
  url: z.string().url().optional(),
  host: z.string().default('localhost'),
  port: z.number().int().default(6379),
  password: z.string().optional(),
  db: z.number().int().min(0).max(15).default(0),
  maxRetriesPerRequest: z.number().int().default(3),
  enableReadyCheck: z.boolean().default(true),
});

const RabbitMQConfigSchema = z.object({
  url: z.string().default('amqp://guest:guest@localhost:5672'),
  heartbeat: z.number().int().default(60),
  prefetchCount: z.number().int().min(1).default(10),
  reconnectDelay: z.number().int().default(5000),
  maxReconnectAttempts: z.number().int().default(-1),
});

const HttpServerConfigSchema = z.object({
  port: z.number().int().min(1).max(65535),
  host: z.string().default('0.0.0.0'),
  requestTimeoutMs: z.number().int().default(30000),
  keepAliveTimeoutMs: z.number().int().default(65000),
  maxBodySizeBytes: z.number().int().default(10 * 1024 * 1024), // 10MB
  corsOrigins: z.array(z.string()).default(['*']),
  trustProxy: z.boolean().default(true),
});

const JwtConfigSchema = z.object({
  secret: z.string().min(32, 'JWT secret must be at least 32 characters'),
  expiresIn: z.string().default('1h'),
  issuer: z.string().default('microservices'),
  audience: z.string().optional(),
});

export const AppConfigSchema = z.object({
  environment: z.enum(['development', 'staging', 'production', 'test']),
  serviceName: z.string().min(1),
  version: z.string().default('1.0.0'),
  logLevel: z.enum(['debug', 'info', 'warn', 'error']).default('info'),
  
  server: HttpServerConfigSchema,
  database: DatabaseConfigSchema,
  redis: RedisConfigSchema.optional(),
  rabbitmq: RabbitMQConfigSchema.optional(),
  jwt: JwtConfigSchema,
  
  tracing: z.object({
    enabled: z.boolean().default(true),
    endpoint: z.string().url().optional(),
    sampleRate: z.number().min(0).max(1).default(0.1),
  }).default({}),
  
  metrics: z.object({
    enabled: z.boolean().default(true),
    path: z.string().default('/metrics'),
  }).default({}),
});

export type AppConfig = z.infer<typeof AppConfigSchema>;

// ─── Config Loader ─────────────────────────────────────────────────────────

export function loadConfig(overrides?: Partial<AppConfig>): AppConfig {
  const raw = {
    environment: process.env.NODE_ENV ?? 'development',
    serviceName: process.env.SERVICE_NAME ?? 'unknown-service',
    version: process.env.APP_VERSION ?? '1.0.0',
    logLevel: process.env.LOG_LEVEL ?? 'info',
    
    server: {
      port: parseInt(process.env.PORT ?? '3000'),
      host: process.env.HOST ?? '0.0.0.0',
      requestTimeoutMs: parseInt(process.env.REQUEST_TIMEOUT_MS ?? '30000'),
      corsOrigins: (process.env.CORS_ORIGINS ?? '*').split(','),
      trustProxy: process.env.TRUST_PROXY !== 'false',
    },
    
    database: {
      host: process.env.DB_HOST ?? 'localhost',
      port: parseInt(process.env.DB_PORT ?? '5432'),
      database: process.env.DB_NAME ?? '',
      user: process.env.DB_USER ?? 'postgres',
      password: process.env.DB_PASSWORD ?? '',
      ssl: process.env.DB_SSL === 'true',
      maxConnections: parseInt(process.env.DB_MAX_CONNECTIONS ?? '20'),
    },
    
    redis: process.env.REDIS_URL
      ? { url: process.env.REDIS_URL }
      : process.env.REDIS_HOST
      ? {
          host: process.env.REDIS_HOST,
          port: parseInt(process.env.REDIS_PORT ?? '6379'),
          password: process.env.REDIS_PASSWORD,
        }
      : undefined,
    
    rabbitmq: process.env.RABBITMQ_URL
      ? { url: process.env.RABBITMQ_URL }
      : undefined,
    
    jwt: {
      secret: process.env.JWT_SECRET ?? '',
      expiresIn: process.env.JWT_EXPIRES_IN ?? '1h',
      issuer: process.env.JWT_ISSUER ?? 'microservices',
    },
    
    tracing: {
      enabled: process.env.TRACING_ENABLED !== 'false',
      endpoint: process.env.OTEL_EXPORTER_OTLP_ENDPOINT,
      sampleRate: parseFloat(process.env.TRACING_SAMPLE_RATE ?? '0.1'),
    },
    
    ...overrides,
  };

  try {
    return AppConfigSchema.parse(raw);
  } catch (error) {
    if (error instanceof z.ZodError) {
      const issues = error.issues.map(
        (i) => `  - ${i.path.join('.')}: ${i.message}`
      ).join('\n');
      throw new Error(`Configuration validation failed:\n${issues}`);
    }
    throw error;
  }
}

// ─── Validation Middleware ─────────────────────────────────────────────────

export function validateEnvVars(requiredVars: string[]): void {
  const missing = requiredVars.filter(
    (key) => !process.env[key] || process.env[key]!.trim() === ''
  );

  if (missing.length > 0) {
    throw new Error(
      `Missing required environment variables:\n${missing.map(v => `  - ${v}`).join('\n')}`
    );
  }
}
```

---

## 8. Environment-specific Config Management

```typescript
// packages/shared/src/config/env.config.ts
// จัดการ config ที่แตกต่างกันในแต่ละ environment

type Environment = 'development' | 'staging' | 'production' | 'test';

interface EnvironmentConfig {
  logLevel: 'debug' | 'info' | 'warn' | 'error';
  dbPoolSize: number;
  cacheEnabled: boolean;
  cacheTtlSeconds: number;
  rateLimitPerMinute: number;
  jwtExpiresIn: string;
  corsOrigins: string[];
  tracingSampleRate: number;
  circuitBreakerThreshold: number;
}

const environmentConfigs: Record<Environment, EnvironmentConfig> = {
  development: {
    logLevel: 'debug',
    dbPoolSize: 5,
    cacheEnabled: false,
    cacheTtlSeconds: 60,
    rateLimitPerMinute: 10000,
    jwtExpiresIn: '7d',
    corsOrigins: ['http://localhost:3000', 'http://localhost:3001'],
    tracingSampleRate: 1.0, // Sample everything in dev
    circuitBreakerThreshold: 10,
  },
  staging: {
    logLevel: 'info',
    dbPoolSize: 10,
    cacheEnabled: true,
    cacheTtlSeconds: 300,
    rateLimitPerMinute: 1000,
    jwtExpiresIn: '1d',
    corsOrigins: ['https://staging.myapp.com'],
    tracingSampleRate: 0.5,
    circuitBreakerThreshold: 5,
  },
  production: {
    logLevel: 'warn',
    dbPoolSize: 25,
    cacheEnabled: true,
    cacheTtlSeconds: 600,
    rateLimitPerMinute: 100,
    jwtExpiresIn: '1h',
    corsOrigins: ['https://myapp.com', 'https://www.myapp.com'],
    tracingSampleRate: 0.1, // Sample 10% in production
    circuitBreakerThreshold: 5,
  },
  test: {
    logLevel: 'error',
    dbPoolSize: 2,
    cacheEnabled: false,
    cacheTtlSeconds: 0,
    rateLimitPerMinute: 100000,
    jwtExpiresIn: '7d',
    corsOrigins: ['*'],
    tracingSampleRate: 0,
    circuitBreakerThreshold: 100,
  },
};

export function getEnvironmentConfig(
  env: Environment = (process.env.NODE_ENV as Environment) ?? 'development'
): EnvironmentConfig {
  const config = environmentConfigs[env];
  if (!config) {
    throw new Error(`Unknown environment: ${env}`);
  }
  return config;
}

// Override ด้วย environment variables
export function mergeWithEnvOverrides(
  base: EnvironmentConfig
): EnvironmentConfig {
  return {
    ...base,
    logLevel:
      (process.env.LOG_LEVEL as EnvironmentConfig['logLevel']) ??
      base.logLevel,
    dbPoolSize: parseInt(
      process.env.DB_POOL_SIZE ?? String(base.dbPoolSize)
    ),
    cacheEnabled:
      process.env.CACHE_ENABLED !== undefined
        ? process.env.CACHE_ENABLED === 'true'
        : base.cacheEnabled,
    cacheTtlSeconds: parseInt(
      process.env.CACHE_TTL_SECONDS ?? String(base.cacheTtlSeconds)
    ),
    rateLimitPerMinute: parseInt(
      process.env.RATE_LIMIT_PER_MINUTE ?? String(base.rateLimitPerMinute)
    ),
    tracingSampleRate: parseFloat(
      process.env.TRACING_SAMPLE_RATE ?? String(base.tracingSampleRate)
    ),
  };
}
```

---

## 9. Config Reload Endpoint

```typescript
// packages/shared/src/config/config.endpoint.ts

import { Router, Request, Response } from 'express';
import { AppConfigManager } from './config.watcher';

export function createConfigRouter(configManager: AppConfigManager) {
  const router = Router();

  // GET /config/features — ดู feature flags ปัจจุบัน
  router.get('/features', (_req: Request, res: Response) => {
    const flags = configManager.getFeatureFlags();
    res.json(flags);
  });

  // GET /config/rate-limit
  router.get('/rate-limit', (_req: Request, res: Response) => {
    const config = configManager.getRateLimitConfig();
    res.json(config);
  });

  // GET /config/feature/:name — check single feature
  router.get('/feature/:name', (req: Request, res: Response) => {
    const { name } = req.params;
    const enabled = configManager.isFeatureEnabled(name);
    res.json({ feature: name, enabled });
  });

  return router;
}
```

---

## 10. Docker Compose ครบสมบูรณ์

```yaml
# docker-compose.config.yml
version: '3.8'

services:
  order-service:
    image: myrepo/order-service:${VERSION:-latest}
    environment:
      NODE_ENV: production
      PORT: 3001
      SERVICE_NAME: order-service
      
      # Database
      DB_HOST: postgres
      DB_PORT: 5432
      DB_NAME: orders
      DB_USER: orders_user
      DB_PASSWORD_FILE: /run/secrets/db_password
      DB_MAX_CONNECTIONS: 25
      DB_SSL: "true"
      
      # Redis
      REDIS_HOST: redis
      REDIS_PORT: 6379
      
      # JWT
      JWT_EXPIRES_IN: 1h
      
      # Feature Flags
      FLAGSMITH_ENVIRONMENT_KEY_FILE: /run/secrets/flagsmith_key
      
      # Tracing
      OTEL_EXPORTER_OTLP_ENDPOINT: http://tempo:4318
      TRACING_SAMPLE_RATE: "0.1"
      
      CONFIG_DIR: /app/config
      
    configs:
      - source: order-service-config
        target: /app/config/rate-limit.json
      - source: features-config
        target: /app/config/features.yaml
    
    secrets:
      - db_password
      - jwt_secret
      - flagsmith_key
    
    volumes:
      - order_logs:/app/logs
    
    deploy:
      replicas: 2
      update_config:
        order: rolling-update
        failure_action: rollback
      resources:
        limits:
          memory: 256M
          cpus: '0.5'

configs:
  order-service-config:
    file: ./config/rate-limit.json
  features-config:
    file: ./config/features.yaml

secrets:
  db_password:
    external: true
  jwt_secret:
    external: true
  flagsmith_key:
    external: true

volumes:
  order_logs:
```

---

## 11. Helm Chart Values สำหรับ Config Management

```yaml
# helm/order-service/values.yaml
replicaCount: 2

image:
  repository: myrepo/order-service
  tag: "1.2.0"
  pullPolicy: IfNotPresent

config:
  logLevel: info
  maxConnections: 25
  requestTimeoutMs: 30000
  rateLimitPerMinute: 1000

  rateLimit:
    windowMs: 60000
    maxRequests: 1000
    skipFailedRequests: false
    keyBy: userId

  features:
    new_checkout_flow: false
    loyalty_points: true
    express_delivery: true
    cod_payment: false

externalSecrets:
  enabled: true
  secretStore: aws-secrets-manager
  refreshInterval: 5m
  secrets:
    - name: DATABASE_URL
      remoteKey: microservices/production/order-service
      property: database_url
    - name: JWT_SECRET
      remoteKey: microservices/production/shared
      property: jwt_secret

featureFlags:
  provider: flagsmith  # or launchdarkly
  environmentKeySecret:
    name: feature-flags-secret
    key: FLAGSMITH_ENVIRONMENT_KEY

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi
```

```yaml
# helm/order-service/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "order-service.fullname" . }}-config
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "order-service.labels" . | nindent 4 }}
data:
  LOG_LEVEL: {{ .Values.config.logLevel | quote }}
  MAX_CONNECTIONS: {{ .Values.config.maxConnections | quote }}
  REQUEST_TIMEOUT_MS: {{ .Values.config.requestTimeoutMs | quote }}
  rate-limit.json: |
    {{ .Values.config.rateLimit | toJson }}
  features.yaml: |
    features:
    {{- range $key, $val := .Values.config.features }}
      {{ $key }}: {{ $val }}
    {{- end }}
```

---

## 12. Config Drift Detection

```typescript
// scripts/config-drift-detector.ts
// ตรวจสอบว่า config ที่ deploy จริงตรงกับที่คาดหวังหรือไม่

import axios from 'axios';
import yaml from 'js-yaml';
import fs from 'fs';

interface ServiceConfigSnapshot {
  service: string;
  endpoint: string;
  expectedConfig: Record<string, unknown>;
}

const services: ServiceConfigSnapshot[] = [
  {
    service: 'order-service',
    endpoint: 'http://order-service:3001/config/features',
    expectedConfig: {
      features: {
        new_checkout_flow: false,
        loyalty_points: true,
        express_delivery: true,
      },
    },
  },
];

async function detectConfigDrift(): Promise<void> {
  console.log('Checking for configuration drift...\n');

  let driftFound = false;

  for (const snapshot of services) {
    try {
      const response = await axios.get(snapshot.endpoint, { timeout: 5000 });
      const actual = response.data;

      const diffs = findDiffs(snapshot.expectedConfig, actual, '');

      if (diffs.length > 0) {
        driftFound = true;
        console.error(`DRIFT DETECTED in ${snapshot.service}:`);
        diffs.forEach((d) => console.error(`  ${d}`));
      } else {
        console.log(`OK: ${snapshot.service} — no drift`);
      }
    } catch (error) {
      console.error(`ERROR checking ${snapshot.service}:`, (error as Error).message);
      driftFound = true;
    }
  }

  if (driftFound) {
    process.exit(1);
  }
}

function findDiffs(
  expected: Record<string, unknown>,
  actual: Record<string, unknown>,
  prefix: string
): string[] {
  const diffs: string[] = [];

  for (const [key, expectedVal] of Object.entries(expected)) {
    const path = prefix ? `${prefix}.${key}` : key;
    const actualVal = actual[key];

    if (actualVal === undefined) {
      diffs.push(`Missing key: ${path} (expected: ${JSON.stringify(expectedVal)})`);
    } else if (
      typeof expectedVal === 'object' &&
      expectedVal !== null &&
      typeof actualVal === 'object' &&
      actualVal !== null
    ) {
      diffs.push(
        ...findDiffs(
          expectedVal as Record<string, unknown>,
          actualVal as Record<string, unknown>,
          path
        )
      );
    } else if (expectedVal !== actualVal) {
      diffs.push(
        `Value mismatch at ${path}: expected=${JSON.stringify(expectedVal)}, actual=${JSON.stringify(actualVal)}`
      );
    }
  }

  return diffs;
}

detectConfigDrift().catch(console.error);
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Kubernetes ConfigMaps/Secrets** — วิธีจัดการ Config และ Secret ใน Kubernetes ด้วย RBAC ที่ถูกต้อง

2. **External Secrets Operator** — ดึง Secret จาก AWS Secrets Manager เข้า Kubernetes โดยอัตโนมัติพร้อม Auto-refresh

3. **Config Hot-Reloading** — Watch file changes ใน ConfigMap Volume และ reload config โดยไม่ต้อง restart service

4. **Feature Flags** — ใช้ Flagsmith และ LaunchDarkly สำหรับ A/B Testing และ Gradual rollout

5. **Config Validation ด้วย Zod** — Validate config ตั้งแต่ startup เพื่อ fail fast ก่อน deploy

6. **Environment-specific Config** — จัดการ config ที่แตกต่างกันในแต่ละ environment อย่างเป็นระบบ

**Best Practice:** เก็บ Secret ใน AWS Secrets Manager หรือ Vault ไม่ใช่ใน ConfigMap หรือ environment variables โดยตรง และใช้ External Secrets Operator เพื่อ sync เข้า Kubernetes

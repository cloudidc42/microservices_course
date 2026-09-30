# Part 76: Advanced Security Patterns

## บทนำ

Security ใน Microservices ต้องการแนวทางที่ครอบคลุมหลายมิติ บทนี้จะครอบคลุม OWASP API Security Top 10 mitigations, Bot Detection, Fraud Detection, PCI-DSS compliance, GDPR data handling, Data Masking/Tokenization และ Penetration Testing basics

---

## 1. OWASP API Security Top 10 Mitigations

### 1.1 API1 - Broken Object Level Authorization (BOLA)

```typescript
// src/security/authorization/bola-protection.ts
import { Request, Response, NextFunction } from 'express';
import { Pool } from 'pg';

interface ResourcePolicy {
  resource: string;
  actions: ('read' | 'write' | 'delete')[];
  ownerField: string;
  adminRoles?: string[];
}

const RESOURCE_POLICIES: Record<string, ResourcePolicy> = {
  orders: {
    resource: 'orders',
    actions: ['read', 'write', 'delete'],
    ownerField: 'user_id',
    adminRoles: ['admin', 'support'],
  },
  profiles: {
    resource: 'profiles',
    actions: ['read', 'write'],
    ownerField: 'id',
    adminRoles: ['admin'],
  },
  payment_methods: {
    resource: 'payment_methods',
    actions: ['read', 'write', 'delete'],
    ownerField: 'user_id',
    adminRoles: ['admin'],
  },
};

export function objectLevelAuthMiddleware(
  resourceType: string,
  action: 'read' | 'write' | 'delete'
) {
  return async (req: Request, res: Response, next: NextFunction) => {
    const policy = RESOURCE_POLICIES[resourceType];
    if (!policy) return next();

    const userId = req.user?.id;
    const userRole = req.user?.role;
    const resourceId = req.params.id;

    if (!userId) {
      return res.status(401).json({ error: 'Unauthorized' });
    }

    // Admins can access everything
    if (policy.adminRoles?.includes(userRole)) {
      return next();
    }

    // Check if user owns the resource
    const db: Pool = req.app.locals.db;
    const { rows } = await db.query(
      `SELECT ${policy.ownerField} FROM ${resourceType} WHERE id = $1`,
      [resourceId]
    );

    if (rows.length === 0) {
      return res.status(404).json({ error: 'Resource not found' });
    }

    if (rows[0][policy.ownerField] !== userId) {
      // Log the unauthorized access attempt
      req.app.locals.securityLogger.warn('BOLA attempt detected', {
        userId,
        resourceType,
        resourceId,
        requestedAction: action,
        ownerId: rows[0][policy.ownerField],
        ip: req.ip,
      });

      return res.status(403).json({ error: 'Forbidden' });
    }

    next();
  };
}

// Usage
// router.get('/orders/:id', objectLevelAuthMiddleware('orders', 'read'), getOrderHandler);
```

### 1.2 API3 - Excessive Data Exposure

```typescript
// src/security/serialization/response-filter.ts
import { Request, Response, NextFunction } from 'express';

type FieldFilter = Record<string, string[]>;

// Define what fields each role can see
const FIELD_PERMISSIONS: Record<string, FieldFilter> = {
  public: {
    user: ['id', 'name', 'avatar'],
    order: ['id', 'status', 'createdAt', 'total'],
  },
  user: {
    user: ['id', 'name', 'email', 'avatar', 'phone', 'createdAt'],
    order: ['id', 'status', 'items', 'total', 'shippingAddress', 'createdAt'],
    payment_method: ['id', 'type', 'last4', 'expiryMonth', 'expiryYear'],
  },
  admin: {
    user: '*', // all fields
    order: '*',
    payment_method: '*',
  },
};

export function filterResponse(resourceType: string) {
  return (req: Request, res: Response, next: NextFunction) => {
    const originalJson = res.json.bind(res);
    const role = req.user?.role || 'public';
    const permissions = FIELD_PERMISSIONS[role]?.[resourceType];

    if (!permissions || permissions === '*') {
      return next();
    }

    res.json = function (data: any) {
      const filtered = filterObject(data, permissions as string[]);
      return originalJson(filtered);
    };

    next();
  };
}

function filterObject(obj: any, allowedFields: string[]): any {
  if (Array.isArray(obj)) {
    return obj.map((item) => filterObject(item, allowedFields));
  }

  if (obj && typeof obj === 'object') {
    const filtered: Record<string, any> = {};
    for (const field of allowedFields) {
      if (field in obj) {
        filtered[field] = obj[field];
      }
    }
    return filtered;
  }

  return obj;
}
```

### 1.3 API4 - Lack of Resources & Rate Limiting

```typescript
// src/security/rate-limiter/advanced-rate-limiter.ts
import { Request, Response, NextFunction } from 'express';
import { Redis } from 'ioredis';
import { createHash } from 'crypto';

interface RateLimitRule {
  name: string;
  windowMs: number;
  max: number;
  keyFn: (req: Request) => string;
  skipFn?: (req: Request) => boolean;
  onLimitReached?: (req: Request, res: Response) => void;
}

const RATE_LIMIT_RULES: RateLimitRule[] = [
  {
    name: 'global-ip',
    windowMs: 60000,
    max: 300,
    keyFn: (req) => `ratelimit:ip:${req.ip}`,
  },
  {
    name: 'login-attempts',
    windowMs: 900000, // 15 minutes
    max: 10,
    keyFn: (req) => `ratelimit:login:${req.ip}:${req.body?.email}`,
  },
  {
    name: 'api-user',
    windowMs: 60000,
    max: 120,
    keyFn: (req) => `ratelimit:user:${req.user?.id}`,
    skipFn: (req) => !req.user,
  },
  {
    name: 'expensive-endpoints',
    windowMs: 60000,
    max: 10,
    keyFn: (req) => `ratelimit:expensive:${req.ip}:${req.path}`,
  },
];

export class AdvancedRateLimiter {
  constructor(private redis: Redis) {}

  middleware(ruleName: string) {
    const rule = RATE_LIMIT_RULES.find((r) => r.name === ruleName);
    if (!rule) throw new Error(`Rate limit rule '${ruleName}' not found`);

    return async (req: Request, res: Response, next: NextFunction) => {
      if (rule.skipFn?.(req)) return next();

      const key = rule.keyFn(req);
      const windowSeconds = Math.floor(rule.windowMs / 1000);

      // Sliding window using Redis sorted set
      const now = Date.now();
      const windowStart = now - rule.windowMs;

      const pipeline = this.redis.pipeline();
      pipeline.zremrangebyscore(key, 0, windowStart);
      pipeline.zadd(key, now, `${now}-${Math.random()}`);
      pipeline.zcard(key);
      pipeline.expire(key, windowSeconds + 1);
      const results = await pipeline.exec();

      const count = (results?.[2]?.[1] as number) || 0;
      const remaining = Math.max(0, rule.max - count);
      const resetTime = Math.floor((now + rule.windowMs) / 1000);

      res.setHeader('X-RateLimit-Limit', rule.max);
      res.setHeader('X-RateLimit-Remaining', remaining);
      res.setHeader('X-RateLimit-Reset', resetTime);
      res.setHeader('X-RateLimit-Policy', `${rule.max};w=${windowSeconds}`);

      if (count > rule.max) {
        if (rule.onLimitReached) {
          return rule.onLimitReached(req, res);
        }

        return res.status(429).json({
          error: 'Too Many Requests',
          retryAfter: windowSeconds,
          limit: rule.max,
          window: `${windowSeconds}s`,
        });
      }

      next();
    };
  }
}
```

---

## 2. Bot Detection

### 2.1 Bot Detection Service

```typescript
// src/security/bot-detection/bot-detector.ts
import { Request, Response, NextFunction } from 'express';
import { Redis } from 'ioredis';
import * as ua from 'ua-parser-js';

interface BotSignal {
  name: string;
  weight: number;
  detected: boolean;
}

interface BotDetectionResult {
  isBot: boolean;
  confidence: number;
  signals: BotSignal[];
  action: 'allow' | 'challenge' | 'block';
}

export class BotDetectionService {
  constructor(private redis: Redis) {}

  async analyze(req: Request): Promise<BotDetectionResult> {
    const signals = await this.collectSignals(req);
    const confidence = this.calculateConfidence(signals);

    const result: BotDetectionResult = {
      isBot: confidence > 0.6,
      confidence,
      signals: signals.filter((s) => s.detected),
      action: this.determineAction(confidence),
    };

    // Store for pattern analysis
    await this.storeSignal(req.ip || '', confidence);

    return result;
  }

  private async collectSignals(req: Request): Promise<BotSignal[]> {
    const signals: BotSignal[] = [];
    const userAgent = req.headers['user-agent'] || '';
    const parsed = ua.UAParser(userAgent);

    // Signal: Missing or suspicious headers
    signals.push({
      name: 'missing_accept_header',
      weight: 0.3,
      detected: !req.headers['accept'],
    });

    signals.push({
      name: 'missing_accept_language',
      weight: 0.2,
      detected: !req.headers['accept-language'],
    });

    signals.push({
      name: 'missing_accept_encoding',
      weight: 0.2,
      detected: !req.headers['accept-encoding'],
    });

    // Signal: Known bot user agents
    const botPatterns = [
      /bot/i, /crawler/i, /spider/i, /scraper/i,
      /wget/i, /curl/i, /python-requests/i, /go-http/i,
      /java\//, /phantom/i, /headless/i,
    ];

    signals.push({
      name: 'bot_user_agent',
      weight: 0.8,
      detected: botPatterns.some((p) => p.test(userAgent)),
    });

    // Signal: No browser detected
    signals.push({
      name: 'no_browser_detected',
      weight: 0.4,
      detected: !parsed.browser.name,
    });

    // Signal: Abnormal request rate from IP
    const requestRate = await this.getRequestRate(req.ip || '');
    signals.push({
      name: 'high_request_rate',
      weight: 0.6,
      detected: requestRate > 30, // > 30 req/minute
    });

    // Signal: IP reputation
    const ipReputation = await this.checkIPReputation(req.ip || '');
    signals.push({
      name: 'bad_ip_reputation',
      weight: 0.9,
      detected: ipReputation === 'bad',
    });

    // Signal: No JavaScript execution (no session token)
    signals.push({
      name: 'no_js_execution',
      weight: 0.3,
      detected: !req.cookies?.['_cf_clearance'] && !req.cookies?.['cf_bot_detection'],
    });

    // Signal: Honeypot field filled
    signals.push({
      name: 'honeypot_triggered',
      weight: 1.0,
      detected: !!(req.body?.website || req.body?.url || req.body?._gotcha),
    });

    return signals;
  }

  private calculateConfidence(signals: BotSignal[]): number {
    const detectedSignals = signals.filter((s) => s.detected);
    if (detectedSignals.length === 0) return 0;

    // Weighted average
    const totalWeight = detectedSignals.reduce((sum, s) => sum + s.weight, 0);
    const maxPossibleWeight = signals.reduce((sum, s) => sum + s.weight, 0);

    return totalWeight / maxPossibleWeight;
  }

  private determineAction(confidence: number): 'allow' | 'challenge' | 'block' {
    if (confidence < 0.4) return 'allow';
    if (confidence < 0.7) return 'challenge';
    return 'block';
  }

  private async getRequestRate(ip: string): Promise<number> {
    const key = `bot:rate:${ip}:${Math.floor(Date.now() / 60000)}`;
    const count = await this.redis.incr(key);
    await this.redis.expire(key, 120);
    return count;
  }

  private async checkIPReputation(ip: string): Promise<'good' | 'neutral' | 'bad'> {
    const cached = await this.redis.get(`ip:reputation:${ip}`);
    if (cached) return cached as any;

    // In production, check against threat intelligence APIs
    // (e.g., AbuseIPDB, MaxMind, etc.)
    return 'neutral';
  }

  private async storeSignal(ip: string, confidence: number): Promise<void> {
    const key = `bot:history:${ip}`;
    await this.redis.lpush(key, JSON.stringify({ confidence, timestamp: Date.now() }));
    await this.redis.ltrim(key, 0, 99);
    await this.redis.expire(key, 86400);
  }
}

export function botDetectionMiddleware(detector: BotDetectionService) {
  return async (req: Request, res: Response, next: NextFunction) => {
    const result = await detector.analyze(req);

    res.locals.botDetection = result;
    res.setHeader('X-Bot-Score', result.confidence.toFixed(2));

    if (result.action === 'block') {
      return res.status(403).json({
        error: 'Access denied',
        code: 'BOT_DETECTED',
      });
    }

    if (result.action === 'challenge') {
      // In production, redirect to CAPTCHA challenge
      res.setHeader('X-Bot-Challenge', 'required');
    }

    next();
  };
}
```

---

## 3. Fraud Detection Patterns

### 3.1 Transaction Fraud Detection

```typescript
// src/security/fraud/fraud-detector.ts
import { Redis } from 'ioredis';
import { Pool } from 'pg';

interface Transaction {
  id: string;
  userId: string;
  amount: number;
  currency: string;
  merchantId: string;
  cardId?: string;
  ipAddress: string;
  userAgent: string;
  country: string;
  timestamp: Date;
  metadata?: Record<string, any>;
}

interface FraudScore {
  score: number;           // 0-100
  riskLevel: 'low' | 'medium' | 'high' | 'critical';
  flags: string[];
  action: 'approve' | 'review' | 'decline' | '3ds_challenge';
  reasons: string[];
}

export class FraudDetectionService {
  constructor(
    private redis: Redis,
    private db: Pool
  ) {}

  async assessTransaction(tx: Transaction): Promise<FraudScore> {
    const [
      velocityFlags,
      geolocationFlags,
      behaviorFlags,
      deviceFlags,
      historicalFlags,
    ] = await Promise.all([
      this.checkVelocity(tx),
      this.checkGeolocation(tx),
      this.checkBehaviorPatterns(tx),
      this.checkDeviceFingerprint(tx),
      this.checkHistoricalPatterns(tx),
    ]);

    const allFlags = [
      ...velocityFlags,
      ...geolocationFlags,
      ...behaviorFlags,
      ...deviceFlags,
      ...historicalFlags,
    ];

    const score = this.calculateFraudScore(allFlags);
    const riskLevel = this.getRiskLevel(score);
    const action = this.determineAction(score, allFlags);

    return {
      score,
      riskLevel,
      flags: allFlags.map((f) => f.name),
      action,
      reasons: allFlags.filter((f) => f.critical).map((f) => f.description),
    };
  }

  private async checkVelocity(tx: Transaction): Promise<FraudFlag[]> {
    const flags: FraudFlag[] = [];
    const userId = tx.userId;

    // Check transaction count in time windows
    const windows = [
      { name: '1m', seconds: 60, maxCount: 3 },
      { name: '1h', seconds: 3600, maxCount: 20 },
      { name: '24h', seconds: 86400, maxCount: 50 },
    ];

    for (const window of windows) {
      const key = `fraud:velocity:${userId}:${window.name}`;
      const count = await this.redis.incr(key);
      await this.redis.expire(key, window.seconds);

      if (count > window.maxCount) {
        flags.push({
          name: `high_velocity_${window.name}`,
          description: `Too many transactions in ${window.name}: ${count}/${window.maxCount}`,
          weight: 20,
          critical: count > window.maxCount * 2,
        });
      }
    }

    // Check amount velocity
    const amountKey = `fraud:amount:${userId}:1h`;
    const totalAmount = await this.redis.incrbyfloat(amountKey, tx.amount);
    await this.redis.expire(amountKey, 3600);

    if (totalAmount > 50000) { // $500 limit per hour
      flags.push({
        name: 'high_amount_velocity',
        description: `High amount velocity: $${totalAmount} in 1h`,
        weight: 25,
        critical: totalAmount > 100000,
      });
    }

    return flags;
  }

  private async checkGeolocation(tx: Transaction): Promise<FraudFlag[]> {
    const flags: FraudFlag[] = [];

    // Get user's typical countries
    const historicalCountries = await this.getUserHistoricalCountries(tx.userId);

    if (historicalCountries.length > 0 && !historicalCountries.includes(tx.country)) {
      flags.push({
        name: 'unusual_country',
        description: `Transaction from unusual country: ${tx.country}`,
        weight: 15,
        critical: false,
      });
    }

    // Check for impossible travel
    const lastTransaction = await this.getLastTransaction(tx.userId);
    if (lastTransaction && lastTransaction.country !== tx.country) {
      const timeDiff = tx.timestamp.getTime() - new Date(lastTransaction.timestamp).getTime();
      const hoursDiff = timeDiff / (1000 * 60 * 60);

      if (hoursDiff < 2 && lastTransaction.country !== tx.country) {
        flags.push({
          name: 'impossible_travel',
          description: `Impossible travel detected: ${lastTransaction.country} -> ${tx.country} in ${hoursDiff.toFixed(1)}h`,
          weight: 40,
          critical: true,
        });
      }
    }

    return flags;
  }

  private async checkBehaviorPatterns(tx: Transaction): Promise<FraudFlag[]> {
    const flags: FraudFlag[] = [];

    // Check if transaction amount is unusual for user
    const { rows } = await this.db.query<{ avg_amount: string; std_amount: string }>(
      `SELECT AVG(amount) as avg_amount, STDDEV(amount) as std_amount
       FROM transactions
       WHERE user_id = $1 AND created_at > NOW() - INTERVAL '90 days'`,
      [tx.userId]
    );

    if (rows.length > 0) {
      const avgAmount = parseFloat(rows[0].avg_amount || '0');
      const stdAmount = parseFloat(rows[0].std_amount || '1');

      const zScore = Math.abs((tx.amount - avgAmount) / (stdAmount || 1));

      if (zScore > 3) {
        flags.push({
          name: 'unusual_amount',
          description: `Transaction amount is ${zScore.toFixed(1)} standard deviations above average`,
          weight: 15,
          critical: zScore > 5,
        });
      }
    }

    return flags;
  }

  private async checkDeviceFingerprint(tx: Transaction): Promise<FraudFlag[]> {
    const flags: FraudFlag[] = [];

    // Check if same device used by multiple users
    const deviceKey = `fraud:device:${tx.userAgent}:${tx.ipAddress}`;
    const usersOnDevice = await this.redis.sadd(deviceKey, tx.userId);
    await this.redis.expire(deviceKey, 86400);

    const deviceUserCount = await this.redis.scard(deviceKey);
    if (deviceUserCount > 3) {
      flags.push({
        name: 'shared_device_multiple_users',
        description: `Device used by ${deviceUserCount} different users`,
        weight: 30,
        critical: deviceUserCount > 10,
      });
    }

    return flags;
  }

  private async checkHistoricalPatterns(tx: Transaction): Promise<FraudFlag[]> {
    const flags: FraudFlag[] = [];

    // Check if user has previous fraud
    const { rows } = await this.db.query(
      `SELECT COUNT(*) as count FROM transactions 
       WHERE user_id = $1 AND is_fraud = true`,
      [tx.userId]
    );

    if (parseInt(rows[0].count) > 0) {
      flags.push({
        name: 'previous_fraud_history',
        description: 'User has previous fraud history',
        weight: 50,
        critical: true,
      });
    }

    return flags;
  }

  private calculateFraudScore(flags: FraudFlag[]): number {
    const totalWeight = flags.reduce((sum, f) => sum + f.weight, 0);
    return Math.min(100, totalWeight);
  }

  private getRiskLevel(score: number): FraudScore['riskLevel'] {
    if (score < 20) return 'low';
    if (score < 40) return 'medium';
    if (score < 70) return 'high';
    return 'critical';
  }

  private determineAction(score: number, flags: FraudFlag[]): FraudScore['action'] {
    const hasCritical = flags.some((f) => f.critical);
    if (hasCritical || score >= 70) return 'decline';
    if (score >= 40) return '3ds_challenge';
    if (score >= 20) return 'review';
    return 'approve';
  }

  private async getUserHistoricalCountries(userId: string): Promise<string[]> {
    const { rows } = await this.db.query(
      `SELECT DISTINCT country FROM transactions 
       WHERE user_id = $1 AND created_at > NOW() - INTERVAL '1 year'
       LIMIT 20`,
      [userId]
    );
    return rows.map((r) => r.country);
  }

  private async getLastTransaction(userId: string): Promise<any> {
    const { rows } = await this.db.query(
      `SELECT * FROM transactions WHERE user_id = $1 ORDER BY created_at DESC LIMIT 1`,
      [userId]
    );
    return rows[0] || null;
  }
}

interface FraudFlag {
  name: string;
  description: string;
  weight: number;
  critical: boolean;
}
```

---

## 4. Data Masking และ Tokenization

### 4.1 PII Data Masking Service

```typescript
// src/security/data-masking/masking-service.ts
import crypto from 'crypto';
import { Pool } from 'pg';

type MaskType = 'email' | 'phone' | 'credit_card' | 'ssn' | 'name' | 'address' | 'custom';

interface MaskingConfig {
  field: string;
  type: MaskType;
  preserveLength?: boolean;
  customPattern?: RegExp;
  customReplacement?: string;
}

export class DataMaskingService {
  mask(value: string, type: MaskType): string {
    switch (type) {
      case 'email':
        return this.maskEmail(value);
      case 'phone':
        return this.maskPhone(value);
      case 'credit_card':
        return this.maskCreditCard(value);
      case 'ssn':
        return this.maskSSN(value);
      case 'name':
        return this.maskName(value);
      default:
        return this.maskDefault(value);
    }
  }

  private maskEmail(email: string): string {
    const [local, domain] = email.split('@');
    if (!domain) return '***@***.***';
    const maskedLocal = local.length > 2
      ? local[0] + '*'.repeat(local.length - 2) + local[local.length - 1]
      : '**';
    const [domainName, tld] = domain.split('.');
    const maskedDomain = domainName[0] + '*'.repeat(Math.max(0, domainName.length - 1));
    return `${maskedLocal}@${maskedDomain}.${tld}`;
  }

  private maskPhone(phone: string): string {
    const digits = phone.replace(/\D/g, '');
    if (digits.length < 4) return '****';
    return '*'.repeat(digits.length - 4) + digits.slice(-4);
  }

  private maskCreditCard(cardNumber: string): string {
    const digits = cardNumber.replace(/\D/g, '');
    return '*'.repeat(digits.length - 4) + digits.slice(-4);
  }

  private maskSSN(ssn: string): string {
    return ssn.replace(/\d(?=\d{4})/g, '*');
  }

  private maskName(name: string): string {
    const parts = name.trim().split(' ');
    return parts
      .map((part, idx) => {
        if (idx === 0 && parts.length > 1) return part; // Keep first name
        if (part.length <= 1) return part;
        return part[0] + '*'.repeat(part.length - 1);
      })
      .join(' ');
  }

  private maskDefault(value: string): string {
    if (value.length <= 4) return '*'.repeat(value.length);
    return value.substring(0, 2) + '*'.repeat(value.length - 4) + value.substring(value.length - 2);
  }

  maskObject<T extends Record<string, any>>(
    obj: T,
    configs: MaskingConfig[]
  ): T {
    const result = { ...obj };
    for (const config of configs) {
      if (config.field in result && result[config.field]) {
        result[config.field] = this.mask(String(result[config.field]), config.type);
      }
    }
    return result;
  }
}

// Tokenization Service
export class TokenizationService {
  private encryptionKey: Buffer;

  constructor(private db: Pool, encryptionKey: string) {
    this.encryptionKey = Buffer.from(encryptionKey, 'hex');
  }

  async tokenize(value: string, context: string): Promise<string> {
    // Check if value already has a token
    const existing = await this.db.query(
      'SELECT token FROM tokenization_vault WHERE value_hash = $1 AND context = $2',
      [this.hash(value), context]
    );

    if (existing.rows.length > 0) {
      return existing.rows[0].token;
    }

    // Generate new token
    const token = 'tok_' + crypto.randomBytes(16).toString('hex');

    // Encrypt the value
    const encrypted = this.encrypt(value);

    // Store token
    await this.db.query(
      `INSERT INTO tokenization_vault (token, encrypted_value, value_hash, context, created_at)
       VALUES ($1, $2, $3, $4, NOW())`,
      [token, encrypted, this.hash(value), context]
    );

    return token;
  }

  async detokenize(token: string, context: string): Promise<string | null> {
    const { rows } = await this.db.query(
      'SELECT encrypted_value FROM tokenization_vault WHERE token = $1 AND context = $2',
      [token, context]
    );

    if (rows.length === 0) return null;
    return this.decrypt(rows[0].encrypted_value);
  }

  async tokenizeCard(cardNumber: string): Promise<string> {
    const token = await this.tokenize(cardNumber, 'credit_card');
    // Return last 4 digits visible
    const last4 = cardNumber.replace(/\D/g, '').slice(-4);
    return `${token}:${last4}`;
  }

  private encrypt(value: string): string {
    const iv = crypto.randomBytes(16);
    const cipher = crypto.createCipheriv('aes-256-gcm', this.encryptionKey, iv);
    const encrypted = Buffer.concat([cipher.update(value, 'utf8'), cipher.final()]);
    const authTag = cipher.getAuthTag();
    return Buffer.concat([iv, authTag, encrypted]).toString('base64');
  }

  private decrypt(encryptedValue: string): string {
    const buffer = Buffer.from(encryptedValue, 'base64');
    const iv = buffer.subarray(0, 16);
    const authTag = buffer.subarray(16, 32);
    const encrypted = buffer.subarray(32);
    const decipher = crypto.createDecipheriv('aes-256-gcm', this.encryptionKey, iv);
    decipher.setAuthTag(authTag);
    return decipher.update(encrypted) + decipher.final('utf8');
  }

  private hash(value: string): string {
    return crypto
      .createHmac('sha256', this.encryptionKey)
      .update(value)
      .digest('hex');
  }
}
```

---

## 5. GDPR Data Handling

### 5.1 GDPR Compliance Service

```typescript
// src/security/gdpr/gdpr-service.ts
import { Pool } from 'pg';
import { Redis } from 'ioredis';
import { EventEmitter } from 'events';

interface ConsentRecord {
  userId: string;
  consentType: ConsentType;
  granted: boolean;
  timestamp: Date;
  version: string;
  ipAddress: string;
  userAgent: string;
}

type ConsentType =
  | 'marketing_email'
  | 'analytics'
  | 'personalization'
  | 'third_party_sharing'
  | 'essential';

interface DataSubjectRequest {
  type: 'access' | 'rectification' | 'erasure' | 'portability' | 'restriction';
  userId: string;
  requestId: string;
  timestamp: Date;
  reason?: string;
}

export class GDPRComplianceService extends EventEmitter {
  constructor(
    private db: Pool,
    private redis: Redis
  ) {
    super();
  }

  // Consent Management
  async recordConsent(consent: ConsentRecord): Promise<void> {
    await this.db.query(
      `INSERT INTO consent_records 
       (user_id, consent_type, granted, consent_version, ip_address, user_agent, created_at)
       VALUES ($1, $2, $3, $4, $5, $6, $7)`,
      [
        consent.userId,
        consent.consentType,
        consent.granted,
        consent.version,
        consent.ipAddress,
        consent.userAgent,
        consent.timestamp,
      ]
    );

    await this.redis.set(
      `consent:${consent.userId}:${consent.consentType}`,
      JSON.stringify({ granted: consent.granted, timestamp: consent.timestamp }),
      'EX',
      86400
    );

    this.emit('consent:recorded', consent);
  }

  async checkConsent(userId: string, consentType: ConsentType): Promise<boolean> {
    // Essential consent is always granted
    if (consentType === 'essential') return true;

    const cached = await this.redis.get(`consent:${userId}:${consentType}`);
    if (cached) {
      return JSON.parse(cached).granted;
    }

    const { rows } = await this.db.query(
      `SELECT granted FROM consent_records
       WHERE user_id = $1 AND consent_type = $2
       ORDER BY created_at DESC LIMIT 1`,
      [userId, consentType]
    );

    return rows.length > 0 ? rows[0].granted : false;
  }

  // Right to Access (Article 15)
  async handleAccessRequest(request: DataSubjectRequest): Promise<any> {
    const userData: Record<string, any> = {};

    // Collect all user data from different tables
    const tables = [
      { table: 'users', fields: ['id', 'email', 'name', 'phone', 'created_at'] },
      { table: 'orders', fields: ['id', 'total', 'status', 'created_at'] },
      { table: 'addresses', fields: ['line1', 'city', 'country', 'postal_code'] },
      { table: 'consent_records', fields: ['consent_type', 'granted', 'created_at'] },
    ];

    for (const { table, fields } of tables) {
      const { rows } = await this.db.query(
        `SELECT ${fields.join(', ')} FROM ${table} WHERE user_id = $1`,
        [request.userId]
      );
      userData[table] = rows;
    }

    // Log the access request
    await this.logDataSubjectRequest(request, 'completed');

    return {
      requestId: request.requestId,
      userId: request.userId,
      requestedAt: request.timestamp,
      fulfilledAt: new Date(),
      data: userData,
    };
  }

  // Right to Erasure (Article 17)
  async handleErasureRequest(request: DataSubjectRequest): Promise<void> {
    // Check if erasure is legally allowed
    const canErase = await this.canEraseUserData(request.userId);
    if (!canErase.allowed) {
      await this.logDataSubjectRequest(request, 'denied', canErase.reason);
      throw new Error(`Cannot erase data: ${canErase.reason}`);
    }

    // Anonymize instead of delete (to maintain referential integrity)
    await this.db.query('BEGIN');
    try {
      // Anonymize user record
      await this.db.query(
        `UPDATE users SET 
         email = 'deleted_' || id || '@deleted.com',
         name = 'Deleted User',
         phone = NULL,
         date_of_birth = NULL,
         is_deleted = true,
         deleted_at = NOW()
         WHERE id = $1`,
        [request.userId]
      );

      // Anonymize addresses
      await this.db.query(
        `UPDATE addresses SET 
         line1 = '[REDACTED]', line2 = NULL,
         city = '[REDACTED]', postal_code = '[REDACTED]'
         WHERE user_id = $1`,
        [request.userId]
      );

      // Remove sensitive payment data
      await this.db.query(
        'DELETE FROM payment_methods WHERE user_id = $1',
        [request.userId]
      );

      // Remove from Redis
      const keys = await this.redis.keys(`user:${request.userId}:*`);
      if (keys.length > 0) {
        await this.redis.del(...keys);
      }

      await this.db.query('COMMIT');
      await this.logDataSubjectRequest(request, 'completed');
      this.emit('gdpr:erasure:completed', request);
    } catch (error) {
      await this.db.query('ROLLBACK');
      throw error;
    }
  }

  // Right to Data Portability (Article 20)
  async handlePortabilityRequest(request: DataSubjectRequest): Promise<string> {
    const data = await this.handleAccessRequest(request);
    
    // Return as JSON (machine-readable format)
    const exportData = {
      exported_at: new Date().toISOString(),
      user_id: request.userId,
      format_version: '1.0',
      data,
    };

    return JSON.stringify(exportData, null, 2);
  }

  private async canEraseUserData(userId: string): Promise<{ allowed: boolean; reason?: string }> {
    // Cannot erase if there are pending transactions
    const { rows } = await this.db.query(
      `SELECT COUNT(*) as count FROM orders 
       WHERE user_id = $1 AND status IN ('pending', 'processing')`,
      [userId]
    );

    if (parseInt(rows[0].count) > 0) {
      return { allowed: false, reason: 'Pending orders exist' };
    }

    // Cannot erase if within retention period for legal purposes
    const { rows: recentOrders } = await this.db.query(
      `SELECT COUNT(*) as count FROM orders 
       WHERE user_id = $1 AND created_at > NOW() - INTERVAL '7 years'`,
      [userId]
    );

    if (parseInt(recentOrders[0].count) > 0) {
      return {
        allowed: false,
        reason: 'Transaction records must be retained for 7 years for tax/legal compliance',
      };
    }

    return { allowed: true };
  }

  private async logDataSubjectRequest(
    request: DataSubjectRequest,
    status: 'completed' | 'denied' | 'pending',
    reason?: string
  ): Promise<void> {
    await this.db.query(
      `INSERT INTO data_subject_requests 
       (request_id, user_id, request_type, status, reason, created_at, completed_at)
       VALUES ($1, $2, $3, $4, $5, $6, NOW())`,
      [request.requestId, request.userId, request.type, status, reason, request.timestamp]
    );
  }
}
```

---

## 6. PCI-DSS Compliance

### 6.1 PCI-DSS Audit Logger

```typescript
// src/security/pci/audit-logger.ts
import { Pool } from 'pg';
import crypto from 'crypto';

interface AuditEvent {
  eventType: string;
  userId?: string;
  adminId?: string;
  resourceType: string;
  resourceId?: string;
  action: string;
  result: 'success' | 'failure';
  ipAddress: string;
  userAgent: string;
  details?: Record<string, any>;
}

export class PCIAuditLogger {
  private readonly HMAC_KEY: Buffer;

  constructor(private db: Pool, hmacKey: string) {
    this.HMAC_KEY = Buffer.from(hmacKey, 'hex');
  }

  async log(event: AuditEvent): Promise<void> {
    const timestamp = new Date();
    const eventId = crypto.randomUUID();

    // Create tamper-evident hash chain
    const previousHash = await this.getLastHash();
    const eventData = JSON.stringify({
      ...event,
      eventId,
      timestamp: timestamp.toISOString(),
      previousHash,
    });
    const hash = this.computeHash(eventData);

    await this.db.query(
      `INSERT INTO pci_audit_log 
       (event_id, event_type, user_id, admin_id, resource_type, resource_id,
        action, result, ip_address, user_agent, details, hash, previous_hash, created_at)
       VALUES ($1,$2,$3,$4,$5,$6,$7,$8,$9,$10,$11,$12,$13,$14)`,
      [
        eventId,
        event.eventType,
        event.userId,
        event.adminId,
        event.resourceType,
        event.resourceId,
        event.action,
        event.result,
        event.ipAddress,
        event.userAgent,
        JSON.stringify(event.details || {}),
        hash,
        previousHash,
        timestamp,
      ]
    );
  }

  async verifyIntegrity(startDate: Date, endDate: Date): Promise<boolean> {
    const { rows } = await this.db.query(
      `SELECT * FROM pci_audit_log 
       WHERE created_at BETWEEN $1 AND $2 
       ORDER BY created_at ASC`,
      [startDate, endDate]
    );

    for (let i = 0; i < rows.length; i++) {
      const row = rows[i];
      const recomputedHash = this.computeHash(JSON.stringify({
        eventType: row.event_type,
        userId: row.user_id,
        action: row.action,
        result: row.result,
        ipAddress: row.ip_address,
        timestamp: row.created_at.toISOString(),
        previousHash: row.previous_hash,
      }));

      if (recomputedHash !== row.hash) {
        console.error(`Audit log integrity violation at event ${row.event_id}`);
        return false;
      }

      if (i > 0 && rows[i - 1].hash !== row.previous_hash) {
        console.error(`Audit log chain broken at event ${row.event_id}`);
        return false;
      }
    }

    return true;
  }

  private async getLastHash(): Promise<string> {
    const { rows } = await this.db.query(
      'SELECT hash FROM pci_audit_log ORDER BY created_at DESC LIMIT 1'
    );
    return rows[0]?.hash || '0'.repeat(64);
  }

  private computeHash(data: string): string {
    return crypto
      .createHmac('sha256', this.HMAC_KEY)
      .update(data)
      .digest('hex');
  }
}
```

---

## 7. Security Headers และ CORS

### 7.1 Comprehensive Security Headers Middleware

```typescript
// src/security/headers/security-headers.ts
import { Request, Response, NextFunction } from 'express';
import helmet from 'helmet';

export function configureSecurityHeaders() {
  return helmet({
    contentSecurityPolicy: {
      directives: {
        defaultSrc: ["'self'"],
        scriptSrc: [
          "'self'",
          "'nonce-{nonce}'",
          'https://cdn.company.com',
          'https://www.googletagmanager.com',
        ],
        styleSrc: [
          "'self'",
          "'unsafe-inline'",
          'https://fonts.googleapis.com',
        ],
        fontSrc: [
          "'self'",
          'https://fonts.gstatic.com',
        ],
        imgSrc: ["'self'", 'data:', 'https:'],
        connectSrc: [
          "'self'",
          'https://api.company.com',
          'wss://ws.company.com',
        ],
        frameSrc: ["'none'"],
        objectSrc: ["'none'"],
        upgradeInsecureRequests: [],
        reportUri: '/csp-report',
      },
    },
    crossOriginEmbedderPolicy: { policy: 'require-corp' },
    crossOriginOpenerPolicy: { policy: 'same-origin' },
    crossOriginResourcePolicy: { policy: 'same-site' },
    dnsPrefetchControl: { allow: false },
    frameguard: { action: 'deny' },
    hidePoweredBy: true,
    hsts: {
      maxAge: 31536000,
      includeSubDomains: true,
      preload: true,
    },
    ieNoOpen: true,
    noSniff: true,
    originAgentCluster: true,
    permittedCrossDomainPolicies: { permittedPolicies: 'none' },
    referrerPolicy: { policy: 'strict-origin-when-cross-origin' },
    xssFilter: true,
  });
}

// CORS Configuration
export function configureCORS(allowedOrigins: string[]) {
  return (req: Request, res: Response, next: NextFunction) => {
    const origin = req.headers.origin || '';

    if (allowedOrigins.includes(origin) || allowedOrigins.includes('*')) {
      res.setHeader('Access-Control-Allow-Origin', origin);
      res.setHeader('Vary', 'Origin');
    }

    res.setHeader(
      'Access-Control-Allow-Methods',
      'GET, POST, PUT, PATCH, DELETE, OPTIONS'
    );
    res.setHeader(
      'Access-Control-Allow-Headers',
      'Content-Type, Authorization, X-Request-ID, X-Idempotency-Key'
    );
    res.setHeader('Access-Control-Allow-Credentials', 'true');
    res.setHeader('Access-Control-Max-Age', '86400');
    res.setHeader('Access-Control-Expose-Headers', 'X-Request-ID, X-RateLimit-Remaining');

    if (req.method === 'OPTIONS') {
      return res.status(204).end();
    }

    next();
  };
}
```

---

## 8. Penetration Testing Basics

### 8.1 Security Scanner Script

```python
# scripts/security_scan.py
import requests
import json
import sys
from typing import List, Dict
import argparse
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class APISecurityScanner:
    def __init__(self, base_url: str, auth_token: str = None):
        self.base_url = base_url.rstrip('/')
        self.session = requests.Session()
        if auth_token:
            self.session.headers['Authorization'] = f'Bearer {auth_token}'
        self.findings: List[Dict] = []

    def scan(self) -> List[Dict]:
        logger.info(f"Starting security scan on {self.base_url}")
        
        self.check_security_headers()
        self.check_error_disclosure()
        self.check_rate_limiting()
        self.check_sql_injection()
        self.check_xss()
        self.check_idor()
        self.check_cors()
        
        return self.findings

    def check_security_headers(self) -> None:
        resp = self.session.get(f"{self.base_url}/")
        required_headers = [
            'Strict-Transport-Security',
            'X-Content-Type-Options',
            'X-Frame-Options',
            'Content-Security-Policy',
        ]
        
        for header in required_headers:
            if header not in resp.headers:
                self.add_finding('MEDIUM', 'Missing Security Header',
                    f'Header {header} is missing', 'security_headers')

    def check_error_disclosure(self) -> None:
        # Test with invalid endpoint
        resp = self.session.get(f"{self.base_url}/nonexistent-endpoint-abc123")
        body = resp.text.lower()
        
        sensitive_patterns = [
            'stack trace', 'exception', 'traceback',
            'at line', 'syntax error', 'undefined variable',
            'database error', 'sql error', 'internal server error details',
        ]
        
        for pattern in sensitive_patterns:
            if pattern in body:
                self.add_finding('HIGH', 'Information Disclosure',
                    f'Error response contains sensitive info: {pattern}',
                    'error_disclosure')
                break

    def check_rate_limiting(self) -> None:
        successful_requests = 0
        for i in range(110):
            resp = self.session.get(f"{self.base_url}/api/products")
            if resp.status_code == 200:
                successful_requests += 1
            elif resp.status_code == 429:
                break
        
        if successful_requests >= 100:
            self.add_finding('MEDIUM', 'Missing Rate Limiting',
                f'Made {successful_requests} requests without rate limiting',
                'rate_limiting')

    def check_sql_injection(self) -> None:
        sql_payloads = [
            "' OR '1'='1",
            "'; DROP TABLE users; --",
            "' UNION SELECT NULL--",
            "1; SELECT SLEEP(5)--",
        ]
        
        for payload in sql_payloads:
            try:
                resp = self.session.get(
                    f"{self.base_url}/api/products",
                    params={'search': payload},
                    timeout=10
                )
                
                # Check for SQL error messages in response
                body = resp.text.lower()
                sql_errors = ['syntax error', 'mysql', 'postgresql', 'sqlite', 'ora-', 'sql server']
                
                for error in sql_errors:
                    if error in body:
                        self.add_finding('CRITICAL', 'Potential SQL Injection',
                            f'SQL error detected with payload: {payload}',
                            'sql_injection')
                        break
            except Exception:
                pass

    def check_xss(self) -> None:
        xss_payloads = [
            "<script>alert(1)</script>",
            "<img src=x onerror=alert(1)>",
            "javascript:alert(1)",
        ]
        
        for payload in xss_payloads:
            resp = self.session.get(
                f"{self.base_url}/api/search",
                params={'q': payload},
            )
            
            if payload in resp.text:
                self.add_finding('HIGH', 'Potential XSS',
                    f'Input reflected without encoding: {payload[:50]}',
                    'xss')
                break

    def check_idor(self) -> None:
        # Test accessing resource with sequential IDs
        test_ids = ['1', '2', '3', '100', '1000']
        
        for id_ in test_ids:
            resp = self.session.get(f"{self.base_url}/api/orders/{id_}")
            if resp.status_code == 200:
                data = resp.json()
                if data.get('userId') and data.get('userId') != self.get_current_user_id():
                    self.add_finding('CRITICAL', 'IDOR Vulnerability',
                        f'Accessed order {id_} belonging to different user',
                        'idor')
                    break

    def check_cors(self) -> None:
        test_origins = [
            'https://evil.com',
            'null',
            'https://company.com.evil.com',
        ]
        
        for origin in test_origins:
            resp = self.session.options(
                f"{self.base_url}/api/products",
                headers={'Origin': origin, 'Access-Control-Request-Method': 'GET'}
            )
            
            allowed_origin = resp.headers.get('Access-Control-Allow-Origin', '')
            if allowed_origin == origin or allowed_origin == '*':
                self.add_finding('HIGH', 'Permissive CORS Policy',
                    f'CORS allows origin: {origin}',
                    'cors')

    def add_finding(self, severity: str, title: str, description: str, category: str) -> None:
        finding = {
            'severity': severity,
            'title': title,
            'description': description,
            'category': category,
        }
        self.findings.append(finding)
        logger.warning(f"[{severity}] {title}: {description}")

    def get_current_user_id(self) -> str:
        resp = self.session.get(f"{self.base_url}/api/me")
        return resp.json().get('id', '') if resp.ok else ''


if __name__ == '__main__':
    parser = argparse.ArgumentParser(description='API Security Scanner')
    parser.add_argument('url', help='Base URL to scan')
    parser.add_argument('--token', help='Auth token', default=None)
    parser.add_argument('--output', help='Output file', default=None)
    args = parser.parse_args()

    scanner = APISecurityScanner(args.url, args.token)
    findings = scanner.scan()

    print(f"\n=== Security Scan Results ===")
    print(f"Total findings: {len(findings)}")
    
    by_severity = {}
    for f in findings:
        by_severity.setdefault(f['severity'], []).append(f)
    
    for severity in ['CRITICAL', 'HIGH', 'MEDIUM', 'LOW']:
        count = len(by_severity.get(severity, []))
        if count > 0:
            print(f"  {severity}: {count}")

    if args.output:
        with open(args.output, 'w') as fp:
            json.dump(findings, fp, indent=2)
        print(f"\nResults saved to {args.output}")
```

---

## สรุป

บทนี้ครอบคลุม Advanced Security Patterns อย่างครบถ้วน:

1. **OWASP API Security Top 10** - BOLA protection, excessive data exposure mitigation, rate limiting
2. **Bot Detection** - Multi-signal bot scoring และ challenge mechanisms
3. **Fraud Detection** - Velocity checks, geolocation anomaly, behavioral analysis
4. **Data Masking** - PII masking สำหรับ email, phone, credit card
5. **Tokenization** - AES-256-GCM encryption สำหรับ sensitive data
6. **GDPR Compliance** - Consent management, right to erasure, data portability
7. **PCI-DSS** - Tamper-evident audit logging with hash chain
8. **Security Headers** - Comprehensive Helmet.js configuration
9. **Penetration Testing** - Automated security scanner สำหรับ API vulnerabilities

Key Takeaways:
- Defense in depth: ใช้หลาย security layer ไม่พึ่งพา layer เดียว
- BOLA เป็น vulnerability ที่พบบ่อยที่สุดใน APIs - ตรวจสอบ ownership ทุกครั้ง
- เข้ารหัส sensitive data ที่ application layer เสมอ แม้ database จะ encrypted
- GDPR erasure ควร anonymize ไม่ใช่ hard delete เพื่อรักษา referential integrity
- Fraud detection ต้องใช้หลาย signals ร่วมกัน ไม่ใช่แค่ rule เดียว

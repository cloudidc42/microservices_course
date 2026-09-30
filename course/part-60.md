# Part 60: API Rate Limiting และ Throttling

## บทนำ

Rate Limiting เป็นกลไกสำคัญในการป้องกัน API จากการถูกใช้งานเกินขีดจำกัด ไม่ว่าจะเป็นการโจมตีแบบ DDoS หรือการใช้งานที่เกินกว่าที่ตกลงไว้ใน Service Level Agreement (SLA) บทนี้จะครอบคลุมอัลกอริทึมหลัก เทคนิคการ Implement บน Redis สำหรับระบบ Distributed รวมถึงการป้องกัน DDoS

## 1. Token Bucket Algorithm

Token Bucket เป็นอัลกอริทึมที่ยืดหยุ่นที่สุด อนุญาตให้มี Burst Traffic ได้ในระยะสั้น

### 1.1 Token Bucket Implementation

```typescript
// src/rate-limiting/token-bucket.ts
import Redis from 'ioredis';
import { logger } from '../utils/logger';

interface TokenBucketConfig {
  capacity: number;        // จำนวน Token สูงสุดใน Bucket
  refillRate: number;      // Token ที่เติมต่อวินาที
  refillInterval: number;  // ช่วงเวลาในการเติม Token (milliseconds)
}

interface RateLimitResult {
  allowed: boolean;
  remaining: number;
  resetAt: number;
  retryAfter?: number;
  tokensConsumed: number;
}

export class TokenBucketRateLimiter {
  private redis: Redis;
  private script: string;

  constructor(redis: Redis) {
    this.redis = redis;
    
    // Lua Script สำหรับ Atomic Token Bucket Operation
    this.script = `
      local key = KEYS[1]
      local capacity = tonumber(ARGV[1])
      local refillRate = tonumber(ARGV[2])
      local refillInterval = tonumber(ARGV[3])
      local requested = tonumber(ARGV[4])
      local now = tonumber(ARGV[5])
      
      -- ดึงข้อมูล Bucket ปัจจุบัน
      local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
      local tokens = tonumber(bucket[1]) or capacity
      local lastRefill = tonumber(bucket[2]) or now
      
      -- คำนวณ Token ที่ควรเติม
      local elapsed = now - lastRefill
      local tokensToAdd = math.floor(elapsed / refillInterval) * refillRate
      tokens = math.min(capacity, tokens + tokensToAdd)
      
      -- คำนวณเวลา Refill ล่าสุด
      if tokensToAdd > 0 then
        lastRefill = now - (elapsed % refillInterval)
      end
      
      -- ตรวจสอบว่ามี Token พอหรือไม่
      if tokens >= requested then
        tokens = tokens - requested
        
        -- อัพเดต Bucket
        redis.call('HMSET', key, 'tokens', tokens, 'last_refill', lastRefill)
        redis.call('PEXPIRE', key, capacity / refillRate * refillInterval * 2)
        
        -- คำนวณเวลา Reset
        local timeToFull = math.ceil((capacity - tokens) / refillRate) * refillInterval
        return {1, tokens, lastRefill + timeToFull, 0}
      else
        -- คำนวณเวลาที่ต้องรอ
        local needed = requested - tokens
        local waitTime = math.ceil(needed / refillRate) * refillInterval
        
        return {0, tokens, lastRefill + waitTime, waitTime}
      end
    `;
  }

  async consume(
    key: string,
    config: TokenBucketConfig,
    tokens = 1
  ): Promise<RateLimitResult> {
    const now = Date.now();
    
    try {
      const result = await this.redis.eval(
        this.script,
        1,
        `rate_limit:token_bucket:${key}`,
        config.capacity.toString(),
        config.refillRate.toString(),
        config.refillInterval.toString(),
        tokens.toString(),
        now.toString()
      ) as [number, number, number, number];

      const [allowed, remaining, resetAt, waitTime] = result;
      
      return {
        allowed: allowed === 1,
        remaining,
        resetAt,
        retryAfter: waitTime > 0 ? Math.ceil(waitTime / 1000) : undefined,
        tokensConsumed: allowed === 1 ? tokens : 0,
      };
      
    } catch (error) {
      logger.error('Token bucket rate limiter error', { key, error });
      // Fail open - อนุญาตเมื่อ Redis ล้มเหลว
      return {
        allowed: true,
        remaining: config.capacity,
        resetAt: now + config.refillInterval,
        tokensConsumed: tokens,
      };
    }
  }
}
```

## 2. Sliding Window Algorithm

Sliding Window แม่นยำกว่า Fixed Window โดยนับ Requests ใน Window ที่เคลื่อนที่ตาม Time

### 2.1 Sliding Window Log Implementation

```typescript
// src/rate-limiting/sliding-window.ts
import Redis from 'ioredis';
import { logger } from '../utils/logger';

interface SlidingWindowConfig {
  windowSizeMs: number;  // ขนาด Window ใน milliseconds
  maxRequests: number;   // จำนวน Request สูงสุดใน Window
}

export class SlidingWindowRateLimiter {
  private redis: Redis;
  private script: string;

  constructor(redis: Redis) {
    this.redis = redis;
    
    this.script = `
      local key = KEYS[1]
      local windowSize = tonumber(ARGV[1])
      local maxRequests = tonumber(ARGV[2])
      local now = tonumber(ARGV[3])
      local windowStart = now - windowSize
      
      -- ลบ Entries ที่เก่าเกิน Window
      redis.call('ZREMRANGEBYSCORE', key, '-inf', windowStart)
      
      -- นับจำนวน Requests ใน Window ปัจจุบัน
      local count = redis.call('ZCARD', key)
      
      if count < maxRequests then
        -- เพิ่ม Request ปัจจุบัน
        redis.call('ZADD', key, now, now .. '-' .. math.random(1000000))
        redis.call('PEXPIRE', key, windowSize)
        return {1, maxRequests - count - 1, now + windowSize}
      else
        -- ดึงเวลาของ Request ที่เก่าที่สุดใน Window
        local oldest = redis.call('ZRANGE', key, 0, 0, 'WITHSCORES')
        local resetAt = 0
        if oldest[2] then
          resetAt = tonumber(oldest[2]) + windowSize
        end
        return {0, 0, resetAt}
      end
    `;
  }

  async isAllowed(
    key: string,
    config: SlidingWindowConfig
  ): Promise<RateLimitResult> {
    const now = Date.now();
    
    try {
      const result = await this.redis.eval(
        this.script,
        1,
        `rate_limit:sliding_window:${key}`,
        config.windowSizeMs.toString(),
        config.maxRequests.toString(),
        now.toString()
      ) as [number, number, number];

      const [allowed, remaining, resetAt] = result;
      
      return {
        allowed: allowed === 1,
        remaining: Math.max(0, remaining),
        resetAt,
        retryAfter: allowed === 0 ? Math.ceil((resetAt - now) / 1000) : undefined,
        tokensConsumed: allowed === 1 ? 1 : 0,
      };
      
    } catch (error) {
      logger.error('Sliding window rate limiter error', { key, error });
      return {
        allowed: true,
        remaining: config.maxRequests,
        resetAt: now + config.windowSizeMs,
        tokensConsumed: 1,
      };
    }
  }
}

interface RateLimitResult {
  allowed: boolean;
  remaining: number;
  resetAt: number;
  retryAfter?: number;
  tokensConsumed: number;
}
```

## 3. Redis-based Distributed Rate Limiting

### 3.1 Distributed Rate Limiter with Multiple Strategies

```typescript
// src/rate-limiting/distributed-rate-limiter.ts
import Redis from 'ioredis';
import { TokenBucketRateLimiter } from './token-bucket';
import { SlidingWindowRateLimiter } from './sliding-window';
import { logger } from '../utils/logger';
import { MetricsCollector } from '../monitoring/metrics';

export type RateLimitStrategy = 'token_bucket' | 'sliding_window' | 'fixed_window';

export interface RateLimitPolicy {
  strategy: RateLimitStrategy;
  windowSizeMs: number;
  maxRequests: number;
  burstCapacity?: number;  // สำหรับ Token Bucket
  costPerRequest?: number; // Cost ของแต่ละ Request (default 1)
}

export interface RateLimitIdentifier {
  type: 'api_key' | 'user_id' | 'ip' | 'composite';
  value: string;
}

export class DistributedRateLimiter {
  private tokenBucket: TokenBucketRateLimiter;
  private slidingWindow: SlidingWindowRateLimiter;

  constructor(
    private redis: Redis,
    private metrics: MetricsCollector
  ) {
    this.tokenBucket = new TokenBucketRateLimiter(redis);
    this.slidingWindow = new SlidingWindowRateLimiter(redis);
  }

  async checkLimit(
    identifier: RateLimitIdentifier,
    policy: RateLimitPolicy
  ): Promise<RateLimitResult> {
    const key = this.buildKey(identifier);
    const cost = policy.costPerRequest ?? 1;
    
    let result: RateLimitResult;
    
    switch (policy.strategy) {
      case 'token_bucket':
        result = await this.tokenBucket.consume(key, {
          capacity: policy.burstCapacity ?? policy.maxRequests,
          refillRate: Math.floor(policy.maxRequests / (policy.windowSizeMs / 1000)),
          refillInterval: Math.floor(policy.windowSizeMs / policy.maxRequests),
        }, cost);
        break;
        
      case 'sliding_window':
        result = await this.slidingWindow.isAllowed(key, {
          windowSizeMs: policy.windowSizeMs,
          maxRequests: policy.maxRequests,
        });
        break;
        
      default:
        result = await this.fixedWindow(key, policy);
    }
    
    // บันทึก Metrics
    this.metrics.increment('rate_limit.check', {
      identifier_type: identifier.type,
      strategy: policy.strategy,
      result: result.allowed ? 'allowed' : 'rejected',
    });
    
    if (!result.allowed) {
      logger.warn('Rate limit exceeded', {
        key,
        identifierType: identifier.type,
        strategy: policy.strategy,
        remaining: result.remaining,
      });
    }
    
    return result;
  }

  private async fixedWindow(
    key: string,
    policy: RateLimitPolicy
  ): Promise<RateLimitResult> {
    const windowKey = `rate_limit:fixed:${key}:${Math.floor(Date.now() / policy.windowSizeMs)}`;
    const now = Date.now();
    
    const pipeline = this.redis.pipeline();
    pipeline.incr(windowKey);
    pipeline.pexpire(windowKey, policy.windowSizeMs);
    
    const results = await pipeline.exec();
    const count = (results?.[0]?.[1] as number) ?? 0;
    
    const resetAt = (Math.floor(now / policy.windowSizeMs) + 1) * policy.windowSizeMs;
    const remaining = Math.max(0, policy.maxRequests - count);
    
    return {
      allowed: count <= policy.maxRequests,
      remaining,
      resetAt,
      retryAfter: count > policy.maxRequests ? Math.ceil((resetAt - now) / 1000) : undefined,
      tokensConsumed: 1,
    };
  }

  private buildKey(identifier: RateLimitIdentifier): string {
    return `${identifier.type}:${identifier.value}`;
  }
}
```

## 4. Per-User และ Per-API-Key Rate Limits

### 4.1 Multi-Tier Rate Limiting Middleware

```typescript
// src/middleware/rate-limit-middleware.ts
import { Request, Response, NextFunction } from 'express';
import { DistributedRateLimiter, RateLimitPolicy } from '../rate-limiting/distributed-rate-limiter';
import { ApiKeyService } from '../auth/api-key-service';
import { logger } from '../utils/logger';

interface RateLimitTier {
  name: string;
  policy: RateLimitPolicy;
}

// กำหนด Policy สำหรับระดับต่างๆ ของ API Key ในระบบ Thai E-commerce
const RATE_LIMIT_TIERS: Record<string, RateLimitTier> = {
  free: {
    name: 'Free Tier',
    policy: {
      strategy: 'sliding_window',
      windowSizeMs: 60000,      // 1 นาที
      maxRequests: 60,           // 60 requests/minute
    },
  },
  basic: {
    name: 'Basic Tier',
    policy: {
      strategy: 'token_bucket',
      windowSizeMs: 60000,
      maxRequests: 300,          // 300 requests/minute
      burstCapacity: 600,        // Burst ได้ถึง 600
    },
  },
  premium: {
    name: 'Premium Tier',
    policy: {
      strategy: 'token_bucket',
      windowSizeMs: 60000,
      maxRequests: 1000,         // 1,000 requests/minute
      burstCapacity: 2000,
    },
  },
  enterprise: {
    name: 'Enterprise Tier',
    policy: {
      strategy: 'token_bucket',
      windowSizeMs: 60000,
      maxRequests: 10000,        // 10,000 requests/minute
      burstCapacity: 20000,
    },
  },
};

// Rate Limits พิเศษสำหรับ Payment API
const PAYMENT_API_LIMITS: RateLimitPolicy = {
  strategy: 'sliding_window',
  windowSizeMs: 60000,
  maxRequests: 10,    // Payment API จำกัดเข้มงวดกว่า
};

export function createRateLimitMiddleware(
  rateLimiter: DistributedRateLimiter,
  apiKeyService: ApiKeyService
) {
  return async (req: Request, res: Response, next: NextFunction): Promise<void> => {
    try {
      const apiKey = extractApiKey(req);
      const userId = (req as any).user?.id;
      const clientIp = getClientIp(req);
      
      // Layer 1: IP-based rate limiting (ป้องกัน DDoS)
      const ipResult = await rateLimiter.checkLimit(
        { type: 'ip', value: clientIp },
        { strategy: 'sliding_window', windowSizeMs: 60000, maxRequests: 1000 }
      );
      
      if (!ipResult.allowed) {
        setRateLimitHeaders(res, ipResult, 'ip');
        res.status(429).json({
          error: 'Too Many Requests',
          message: 'IP rate limit exceeded',
          retryAfter: ipResult.retryAfter,
        });
        return;
      }

      // Layer 2: API Key rate limiting
      if (apiKey) {
        const keyInfo = await apiKeyService.getKeyInfo(apiKey);
        
        if (!keyInfo) {
          res.status(401).json({ error: 'Invalid API Key' });
          return;
        }
        
        const tier = RATE_LIMIT_TIERS[keyInfo.tier] ?? RATE_LIMIT_TIERS.free;
        
        const keyResult = await rateLimiter.checkLimit(
          { type: 'api_key', value: apiKey },
          tier.policy
        );
        
        if (!keyResult.allowed) {
          setRateLimitHeaders(res, keyResult, 'api_key');
          res.status(429).json({
            error: 'Too Many Requests',
            message: `${tier.name} rate limit exceeded`,
            limit: tier.policy.maxRequests,
            window: `${tier.policy.windowSizeMs / 1000}s`,
            retryAfter: keyResult.retryAfter,
            upgradeUrl: 'https://api.company.th/plans',
          });
          return;
        }
        
        setRateLimitHeaders(res, keyResult, 'api_key');
      }

      // Layer 3: User-specific rate limiting
      if (userId) {
        // Payment endpoints มี Rate Limit พิเศษ
        if (req.path.startsWith('/api/v1/payments') || req.path.startsWith('/api/v1/transfers')) {
          const paymentResult = await rateLimiter.checkLimit(
            { type: 'user_id', value: `${userId}:payment` },
            PAYMENT_API_LIMITS
          );
          
          if (!paymentResult.allowed) {
            res.status(429).json({
              error: 'Too Many Requests',
              message: 'Payment API rate limit exceeded for safety purposes',
              retryAfter: paymentResult.retryAfter,
            });
            return;
          }
        }
      }

      next();
      
    } catch (error) {
      logger.error('Rate limit middleware error', { error });
      next(); // Fail open
    }
  };
}

function extractApiKey(req: Request): string | null {
  // ตรวจสอบ Header ก่อน
  const headerKey = req.headers['x-api-key'] as string;
  if (headerKey) return headerKey;
  
  // ตรวจสอบ Bearer Token
  const authHeader = req.headers.authorization;
  if (authHeader?.startsWith('Bearer ')) {
    return authHeader.substring(7);
  }
  
  // ตรวจสอบ Query Parameter (ไม่แนะนำแต่ Support เพื่อ Backward Compatibility)
  return (req.query.api_key as string) ?? null;
}

function getClientIp(req: Request): string {
  // รองรับ Reverse Proxy เช่น Nginx, AWS ELB
  const forwarded = req.headers['x-forwarded-for'] as string;
  if (forwarded) {
    return forwarded.split(',')[0].trim();
  }
  
  const realIp = req.headers['x-real-ip'] as string;
  if (realIp) return realIp;
  
  return req.socket.remoteAddress ?? 'unknown';
}

function setRateLimitHeaders(
  res: Response,
  result: any,
  scope: string
): void {
  res.setHeader('X-RateLimit-Scope', scope);
  res.setHeader('X-RateLimit-Remaining', result.remaining.toString());
  res.setHeader('X-RateLimit-Reset', Math.ceil(result.resetAt / 1000).toString());
  
  if (result.retryAfter !== undefined) {
    res.setHeader('Retry-After', result.retryAfter.toString());
  }
}
```

## 5. Burst Handling

### 5.1 Leaky Bucket สำหรับ Burst Traffic Control

```typescript
// src/rate-limiting/leaky-bucket.ts
import Redis from 'ioredis';

interface LeakyBucketConfig {
  capacity: number;     // จำนวน Requests สูงสุดที่ Queue ได้
  leakRateMs: number;  // ปล่อย Request ออกทุก N milliseconds
}

export class LeakyBucketRateLimiter {
  private script: string;

  constructor(private redis: Redis) {
    this.script = `
      local key = KEYS[1]
      local capacity = tonumber(ARGV[1])
      local leakRateMs = tonumber(ARGV[2])
      local now = tonumber(ARGV[3])
      
      local water = redis.call('HMGET', key, 'water', 'last_leak')
      local currentWater = tonumber(water[1]) or 0
      local lastLeak = tonumber(water[2]) or now
      
      -- คำนวณ Water ที่รั่วออกไปตั้งแต่ครั้งล่าสุด
      local elapsed = now - lastLeak
      local leaked = math.floor(elapsed / leakRateMs)
      currentWater = math.max(0, currentWater - leaked)
      
      if leaked > 0 then
        lastLeak = now
      end
      
      -- ตรวจสอบว่า Bucket เต็มหรือไม่
      if currentWater < capacity then
        currentWater = currentWater + 1
        redis.call('HMSET', key, 'water', currentWater, 'last_leak', lastLeak)
        redis.call('PEXPIRE', key, capacity * leakRateMs)
        
        -- คำนวณเวลาที่ Request นี้จะถูก Process
        local processAt = now + (currentWater * leakRateMs)
        return {1, capacity - currentWater, processAt}
      else
        -- Bucket เต็ม - Reject Request
        local waitTime = (currentWater - capacity + 1) * leakRateMs
        return {0, 0, now + waitTime}
      end
    `;
  }

  async enqueue(
    key: string,
    config: LeakyBucketConfig
  ): Promise<{ accepted: boolean; processAt?: number; waitMs?: number }> {
    const now = Date.now();
    
    const result = await this.redis.eval(
      this.script,
      1,
      `rate_limit:leaky_bucket:${key}`,
      config.capacity.toString(),
      config.leakRateMs.toString(),
      now.toString()
    ) as [number, number, number];

    const [accepted, remaining, time] = result;
    
    if (accepted === 1) {
      return {
        accepted: true,
        processAt: time,
      };
    }
    
    return {
      accepted: false,
      waitMs: time - now,
    };
  }
}
```

## 6. Rate Limit Headers

### 6.1 Standardized Rate Limit Headers (RFC 6585 & Draft RFC)

```typescript
// src/rate-limiting/rate-limit-headers.ts
import { Response } from 'express';

interface RateLimitInfo {
  limit: number;
  remaining: number;
  resetAt: number;       // Unix timestamp (seconds)
  retryAfter?: number;   // Seconds to wait
  policy?: string;       // ชื่อ Policy ที่ใช้
}

export function setStandardRateLimitHeaders(
  res: Response,
  info: RateLimitInfo
): void {
  // Standard Headers ตาม IETF Draft
  res.setHeader('RateLimit-Limit', info.limit);
  res.setHeader('RateLimit-Remaining', Math.max(0, info.remaining));
  res.setHeader('RateLimit-Reset', Math.ceil(info.resetAt));
  
  if (info.policy) {
    res.setHeader('RateLimit-Policy', info.policy);
  }
  
  // X-RateLimit headers (de facto standard ที่ใช้กันแพร่หลาย)
  res.setHeader('X-RateLimit-Limit', info.limit);
  res.setHeader('X-RateLimit-Remaining', Math.max(0, info.remaining));
  res.setHeader('X-RateLimit-Reset', Math.ceil(info.resetAt));
  
  if (info.retryAfter !== undefined) {
    res.setHeader('Retry-After', info.retryAfter);
    res.setHeader('X-RateLimit-Retry-After', info.retryAfter);
  }
}

// Helper สำหรับสร้าง Rate Limit Response Body
export function createRateLimitResponse(
  info: RateLimitInfo & { tier?: string; upgradeUrl?: string }
) {
  return {
    error: {
      code: 'RATE_LIMIT_EXCEEDED',
      message: 'คุณส่ง Request เกินจำนวนที่กำหนด กรุณารอสักครู่แล้วลองใหม่',
      details: {
        limit: info.limit,
        remaining: 0,
        resetAt: new Date(info.resetAt * 1000).toISOString(),
        retryAfterSeconds: info.retryAfter,
        tier: info.tier,
      },
      links: info.upgradeUrl ? {
        upgrade: info.upgradeUrl,
        documentation: 'https://docs.api.company.th/rate-limiting',
      } : undefined,
    },
  };
}
```

## 7. DDoS Protection Patterns

### 7.1 Multi-Layer DDoS Protection

```typescript
// src/security/ddos-protection.ts
import Redis from 'ioredis';
import { Request, Response, NextFunction } from 'express';
import { logger } from '../utils/logger';
import { AlertManager } from '../monitoring/alerts';

interface DDoSConfig {
  // Layer 1: Connection limits
  maxConnectionsPerIp: number;
  connectionWindowMs: number;
  
  // Layer 2: Request rate
  maxRequestsPerIp: number;
  requestWindowMs: number;
  
  // Layer 3: Suspicious pattern detection
  maxErrorsBeforeBlock: number;
  errorWindowMs: number;
  blockDurationMs: number;
  
  // Layer 4: Geographic restrictions (optional)
  allowedCountries?: string[];
}

export class DDoSProtection {
  constructor(
    private redis: Redis,
    private config: DDoSConfig,
    private alerts: AlertManager
  ) {}

  createMiddleware() {
    return async (req: Request, res: Response, next: NextFunction): Promise<void> => {
      const ip = this.getClientIp(req);
      
      try {
        // ตรวจสอบ Blocklist ก่อน
        const isBlocked = await this.isBlocked(ip);
        if (isBlocked) {
          logger.warn('Blocked IP attempted access', { ip, path: req.path });
          res.status(403).json({
            error: 'Access Denied',
            message: 'IP address has been temporarily blocked due to suspicious activity',
          });
          return;
        }
        
        // Layer 1: Connection Rate Check
        const connAllowed = await this.checkConnectionRate(ip);
        if (!connAllowed) {
          await this.blockIp(ip, 'connection_rate_exceeded');
          res.status(429).json({
            error: 'Too Many Connections',
            message: 'Connection rate limit exceeded',
          });
          return;
        }
        
        // Layer 2: Request Rate Check
        const reqAllowed = await this.checkRequestRate(ip);
        if (!reqAllowed) {
          await this.handleSuspiciousActivity(ip, 'request_rate_exceeded');
          res.status(429).json({
            error: 'Too Many Requests',
            message: 'Please slow down your requests',
          });
          return;
        }
        
        // Layer 3: Behavior Analysis
        res.on('finish', () => {
          if (res.statusCode >= 400) {
            this.recordError(ip, res.statusCode).catch(err => {
              logger.error('Failed to record error', { err });
            });
          }
        });
        
        next();
        
      } catch (error) {
        logger.error('DDoS protection error', { error });
        next(); // Fail open
      }
    };
  }

  private async checkConnectionRate(ip: string): Promise<boolean> {
    const key = `ddos:connections:${ip}`;
    const count = await this.redis.incr(key);
    
    if (count === 1) {
      await this.redis.pexpire(key, this.config.connectionWindowMs);
    }
    
    return count <= this.config.maxConnectionsPerIp;
  }

  private async checkRequestRate(ip: string): Promise<boolean> {
    const windowKey = `ddos:requests:${ip}:${Math.floor(Date.now() / this.config.requestWindowMs)}`;
    const count = await this.redis.incr(windowKey);
    
    if (count === 1) {
      await this.redis.pexpire(windowKey, this.config.requestWindowMs);
    }
    
    return count <= this.config.maxRequestsPerIp;
  }

  private async recordError(ip: string, statusCode: number): Promise<void> {
    const key = `ddos:errors:${ip}`;
    const count = await this.redis.incr(key);
    
    if (count === 1) {
      await this.redis.pexpire(key, this.config.errorWindowMs);
    }
    
    if (count >= this.config.maxErrorsBeforeBlock) {
      await this.blockIp(ip, `error_threshold_exceeded:${count}_errors`);
    }
  }

  private async blockIp(ip: string, reason: string): Promise<void> {
    const key = `ddos:blocked:${ip}`;
    await this.redis.set(key, reason, 'PX', this.config.blockDurationMs);
    
    logger.warn('IP blocked for suspicious activity', { ip, reason });
    
    this.alerts.sendAlert({
      severity: 'warning',
      title: 'IP Blocked',
      message: `IP ${ip} blocked: ${reason}`,
      details: { ip, reason, blockedUntil: new Date(Date.now() + this.config.blockDurationMs).toISOString() },
    });
  }

  private async isBlocked(ip: string): Promise<boolean> {
    const result = await this.redis.exists(`ddos:blocked:${ip}`);
    return result === 1;
  }

  async unblockIp(ip: string): Promise<void> {
    await this.redis.del(`ddos:blocked:${ip}`);
    logger.info('IP manually unblocked', { ip });
  }

  async getBlockList(): Promise<Array<{ ip: string; reason: string; expiresAt: number }>> {
    const keys = await this.redis.keys('ddos:blocked:*');
    const result = [];
    
    for (const key of keys) {
      const [reason, ttl] = await Promise.all([
        this.redis.get(key),
        this.redis.pttl(key),
      ]);
      
      const ip = key.replace('ddos:blocked:', '');
      result.push({
        ip,
        reason: reason ?? 'unknown',
        expiresAt: Date.now() + ttl,
      });
    }
    
    return result;
  }

  private getClientIp(req: Request): string {
    const forwarded = req.headers['x-forwarded-for'] as string;
    if (forwarded) return forwarded.split(',')[0].trim();
    return req.socket.remoteAddress ?? 'unknown';
  }
}

// ตัวอย่างการ Configure สำหรับระบบ Payment API ของไทย
export const paymentApiDDoSConfig: DDoSConfig = {
  maxConnectionsPerIp: 100,     // 100 connections ต่อนาที
  connectionWindowMs: 60000,
  maxRequestsPerIp: 300,        // 300 requests ต่อนาที
  requestWindowMs: 60000,
  maxErrorsBeforeBlock: 50,     // Block เมื่อ Error > 50 ครั้งใน 5 นาที
  errorWindowMs: 300000,
  blockDurationMs: 3600000,     // Block 1 ชั่วโมง
};
```

## 8. Rate Limit Dashboard และ Admin API

### 8.1 Admin Rate Limit Management

```typescript
// src/admin/rate-limit-admin.ts
import { Router, Request, Response } from 'express';
import { DistributedRateLimiter } from '../rate-limiting/distributed-rate-limiter';
import { DDoSProtection } from '../security/ddos-protection';
import Redis from 'ioredis';

export function createAdminRateLimitRouter(
  rateLimiter: DistributedRateLimiter,
  ddosProtection: DDoSProtection,
  redis: Redis
): Router {
  const router = Router();

  // ดูสถานะ Rate Limit ของ API Key
  router.get('/rate-limits/:apiKey/status', async (req: Request, res: Response) => {
    const { apiKey } = req.params;
    
    const [tokenBucketData, slidingWindowData] = await Promise.all([
      redis.hgetall(`rate_limit:token_bucket:api_key:${apiKey}`),
      redis.zcard(`rate_limit:sliding_window:api_key:${apiKey}`),
    ]);
    
    res.json({
      apiKey,
      tokenBucket: {
        tokens: tokenBucketData?.tokens ?? 'N/A',
        lastRefill: tokenBucketData?.last_refill
          ? new Date(parseInt(tokenBucketData.last_refill)).toISOString()
          : 'N/A',
      },
      slidingWindow: {
        currentRequests: slidingWindowData,
      },
    });
  });

  // Reset Rate Limit ของ API Key
  router.delete('/rate-limits/:apiKey/reset', async (req: Request, res: Response) => {
    const { apiKey } = req.params;
    
    const keys = await redis.keys(`rate_limit:*:api_key:${apiKey}*`);
    
    if (keys.length > 0) {
      await redis.del(...keys);
    }
    
    res.json({
      success: true,
      message: `Rate limit reset for API key: ${apiKey}`,
      keysDeleted: keys.length,
    });
  });

  // ดูรายการ IP ที่ถูก Block
  router.get('/blocked-ips', async (req: Request, res: Response) => {
    const blockedIps = await ddosProtection.getBlockList();
    res.json({
      total: blockedIps.length,
      ips: blockedIps,
    });
  });

  // Unblock IP
  router.delete('/blocked-ips/:ip', async (req: Request, res: Response) => {
    await ddosProtection.unblockIp(req.params.ip);
    res.json({
      success: true,
      message: `IP ${req.params.ip} has been unblocked`,
    });
  });

  // ดู Rate Limit Statistics
  router.get('/stats', async (req: Request, res: Response) => {
    const [
      totalKeys,
      blockedCount,
    ] = await Promise.all([
      redis.dbsize(),
      (await redis.keys('ddos:blocked:*')).length,
    ]);
    
    res.json({
      totalTrackedKeys: totalKeys,
      currentlyBlocked: blockedCount,
      timestamp: new Date().toISOString(),
    });
  });

  return router;
}
```

## 9. ตัวอย่าง Integration ครบวงจร

### 9.1 Express App Setup

```typescript
// src/app.ts
import express from 'express';
import Redis from 'ioredis';
import { DistributedRateLimiter } from './rate-limiting/distributed-rate-limiter';
import { DDoSProtection, paymentApiDDoSConfig } from './security/ddos-protection';
import { createRateLimitMiddleware } from './middleware/rate-limit-middleware';
import { MetricsCollector } from './monitoring/metrics';
import { AlertManager } from './monitoring/alerts';
import { ApiKeyService } from './auth/api-key-service';

const app = express();

// Clients
const redis = new Redis({
  host: process.env.REDIS_HOST ?? 'localhost',
  port: parseInt(process.env.REDIS_PORT ?? '6379'),
  maxRetriesPerRequest: 3,
  enableReadyCheck: true,
  lazyConnect: true,
  // Cluster support สำหรับ Production
  // รองรับ Redis Cluster เพื่อ High Availability
});

const metrics = new MetricsCollector();
const alerts = new AlertManager();
const apiKeyService = new ApiKeyService(redis);

// Rate Limiters
const rateLimiter = new DistributedRateLimiter(redis, metrics);
const ddosProtection = new DDoSProtection(redis, paymentApiDDoSConfig, alerts);

// Apply Middleware ตามลำดับ
app.use(ddosProtection.createMiddleware());
app.use(createRateLimitMiddleware(rateLimiter, apiKeyService));

// Payment Routes
app.post('/api/v1/payments', async (req, res) => {
  // Business logic here
  res.json({ success: true, transactionId: 'txn_xxx' });
});

// PromptPay QR Code Generation (จำกัดเข้มงวด)
app.post('/api/v1/payments/promptpay/qr', async (req, res) => {
  res.json({
    qrCode: 'data:image/png;base64,...',
    expiresAt: new Date(Date.now() + 300000).toISOString(),
  });
});

export default app;
```

## สรุป

| อัลกอริทึม | ความแม่นยำ | รองรับ Burst | ความซับซ้อน | เหมาะกับ |
|-----------|-----------|-------------|------------|---------|
| Fixed Window | ต่ำ | ไม่รองรับ | น้อย | API ทั่วไปที่ไม่สำคัญมาก |
| Sliding Window | สูง | ไม่รองรับ | ปานกลาง | API ที่ต้องการความแม่นยำสูง |
| Token Bucket | สูง | รองรับ | ปานกลาง | E-commerce, ระบบที่ต้องการ Burst |
| Leaky Bucket | สูง | ควบคุม Burst | สูง | Real-time Processing |
| DDoS Protection | - | - | สูง | Production Security Layer |

Rate Limiting ที่ดีควรมีหลาย Layer ตั้งแต่ IP Level จนถึง User Level และควรมีการ Monitor และ Alert เมื่อมีการ Exceed Limit เพื่อให้ทีมตอบสนองได้อย่างรวดเร็ว สำหรับระบบ Payment และ FinTech ไทย ควรตั้งค่า Rate Limit ที่เข้มงวดกว่า API ทั่วไป เพื่อป้องกันการฉ้อโกงและการโจมตี

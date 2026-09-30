# Part 48: API Security

## บทนำ

API Security เป็นส่วนสำคัญที่สุดในการพัฒนา Microservices ที่ production-ready ในบทนี้จะครอบคลุม OAuth 2.0 PKCE Flow, Token Introspection, Scope-based Authorization, API Key Management, Input Validation ด้วย Zod, SQL Injection Prevention, XSS Prevention และ CORS Configuration

---

## 1. OAuth 2.0 PKCE Flow

### 1.1 PKCE Implementation (Authorization Server)

```typescript
// src/auth/pkce/pkce.service.ts
import { Injectable } from '@nestjs/common';
import { createHash, randomBytes } from 'crypto';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { AuthorizationCode } from './entities/authorization-code.entity';
import { JwtService } from '@nestjs/jwt';

interface PKCEChallenge {
  codeVerifier: string;
  codeChallenge: string;
  codeChallengeMethod: 'S256' | 'plain';
}

interface AuthorizationRequest {
  clientId: string;
  redirectUri: string;
  scope: string;
  state: string;
  codeChallenge: string;
  codeChallengeMethod: 'S256' | 'plain';
  userId: string;
}

@Injectable()
export class PKCEService {
  constructor(
    @InjectRepository(AuthorizationCode)
    private readonly authCodeRepo: Repository<AuthorizationCode>,
    private readonly jwtService: JwtService,
  ) {}

  // สร้าง PKCE challenge (ฝั่ง client)
  static generatePKCEChallenge(): PKCEChallenge {
    // สร้าง code verifier (random string 43-128 chars)
    const codeVerifier = randomBytes(32)
      .toString('base64url')
      .slice(0, 128);

    // สร้าง code challenge จาก verifier
    const codeChallenge = createHash('sha256')
      .update(codeVerifier)
      .digest('base64url');

    return {
      codeVerifier,
      codeChallenge,
      codeChallengeMethod: 'S256',
    };
  }

  // สร้าง authorization code (ฝั่ง server)
  async createAuthorizationCode(request: AuthorizationRequest): Promise<string> {
    // ตรวจสอบ client ว่า valid
    await this.validateClient(request.clientId, request.redirectUri);

    // สร้าง authorization code
    const code = randomBytes(32).toString('hex');
    const expiresAt = new Date(Date.now() + 10 * 60 * 1000); // 10 minutes

    await this.authCodeRepo.save({
      code,
      clientId: request.clientId,
      userId: request.userId,
      redirectUri: request.redirectUri,
      scope: request.scope,
      codeChallenge: request.codeChallenge,
      codeChallengeMethod: request.codeChallengeMethod,
      expiresAt,
      used: false,
    });

    return code;
  }

  // แลก authorization code เป็น tokens
  async exchangeCodeForTokens(
    code: string,
    codeVerifier: string,
    clientId: string,
    redirectUri: string,
  ): Promise<{
    accessToken: string;
    refreshToken: string;
    tokenType: string;
    expiresIn: number;
    scope: string;
  }> {
    // ดึง authorization code
    const authCode = await this.authCodeRepo.findOne({
      where: { code, clientId, used: false },
    });

    if (!authCode) {
      throw new Error('Invalid authorization code');
    }

    // ตรวจสอบว่า code ยังไม่หมดอายุ
    if (authCode.expiresAt < new Date()) {
      await this.authCodeRepo.delete({ code });
      throw new Error('Authorization code expired');
    }

    // ตรวจสอบ redirect URI
    if (authCode.redirectUri !== redirectUri) {
      throw new Error('Redirect URI mismatch');
    }

    // ตรวจสอบ PKCE code verifier
    this.verifyCodeChallenge(
      codeVerifier,
      authCode.codeChallenge,
      authCode.codeChallengeMethod,
    );

    // Mark code as used (ป้องกัน replay attack)
    await this.authCodeRepo.update({ code }, { used: true });

    // สร้าง tokens
    const scopes = authCode.scope.split(' ');
    const accessToken = this.jwtService.sign(
      {
        sub: authCode.userId,
        client_id: clientId,
        scope: authCode.scope,
        token_type: 'access',
      },
      { expiresIn: '1h' },
    );

    const refreshToken = this.jwtService.sign(
      {
        sub: authCode.userId,
        client_id: clientId,
        scope: authCode.scope,
        token_type: 'refresh',
      },
      { expiresIn: '30d' },
    );

    return {
      accessToken,
      refreshToken,
      tokenType: 'Bearer',
      expiresIn: 3600,
      scope: authCode.scope,
    };
  }

  private verifyCodeChallenge(
    codeVerifier: string,
    codeChallenge: string,
    method: 'S256' | 'plain',
  ): void {
    let computedChallenge: string;

    if (method === 'S256') {
      computedChallenge = createHash('sha256')
        .update(codeVerifier)
        .digest('base64url');
    } else {
      computedChallenge = codeVerifier;
    }

    // Timing-safe comparison เพื่อป้องกัน timing attacks
    const expectedBuffer = Buffer.from(codeChallenge);
    const actualBuffer = Buffer.from(computedChallenge);

    if (
      expectedBuffer.length !== actualBuffer.length ||
      !require('crypto').timingSafeEqual(expectedBuffer, actualBuffer)
    ) {
      throw new Error('Code verifier does not match code challenge');
    }
  }

  private async validateClient(clientId: string, redirectUri: string): Promise<void> {
    // ตรวจสอบ client registration
    // Implementation depends on your client registry
  }
}
```

### 1.2 Token Introspection Endpoint

```typescript
// src/auth/token/token-introspection.service.ts
import { Injectable } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { InjectRedis } from '@nestjs-modules/ioredis';
import Redis from 'ioredis';

interface TokenIntrospectionResponse {
  active: boolean;
  scope?: string;
  client_id?: string;
  username?: string;
  token_type?: string;
  exp?: number;
  iat?: number;
  nbf?: number;
  sub?: string;
  aud?: string[];
  iss?: string;
  jti?: string;
}

@Injectable()
export class TokenIntrospectionService {
  constructor(
    private readonly jwtService: JwtService,
    @InjectRedis() private readonly redis: Redis,
  ) {}

  async introspect(token: string): Promise<TokenIntrospectionResponse> {
    try {
      // ตรวจสอบว่า token ถูก revoke แล้วหรือยัง
      const isRevoked = await this.redis.get(`revoked:token:${token}`);
      if (isRevoked) {
        return { active: false };
      }

      // Verify JWT signature
      const payload = await this.jwtService.verifyAsync(token);

      // ตรวจสอบ token type (ป้องกันใช้ refresh token แทน access token)
      if (payload.token_type !== 'access') {
        return { active: false };
      }

      return {
        active: true,
        scope: payload.scope,
        client_id: payload.client_id,
        username: payload.username,
        token_type: 'Bearer',
        exp: payload.exp,
        iat: payload.iat,
        sub: payload.sub,
        iss: payload.iss,
        jti: payload.jti,
      };
    } catch {
      return { active: false };
    }
  }

  // Revoke token (logout)
  async revokeToken(token: string): Promise<void> {
    try {
      const payload = this.jwtService.decode(token) as { exp?: number };
      
      if (payload?.exp) {
        const ttl = payload.exp - Math.floor(Date.now() / 1000);
        if (ttl > 0) {
          // เก็บ revoked token ไว้จนกว่าจะหมดอายุ
          await this.redis.setex(`revoked:token:${token}`, ttl, '1');
        }
      }
    } catch {
      // ignore decode errors
    }
  }
}
```

---

## 2. Scope-based Authorization

```typescript
// src/auth/authorization/scope.guard.ts
import {
  Injectable,
  CanActivate,
  ExecutionContext,
  ForbiddenException,
  UnauthorizedException,
} from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { Request } from 'express';

export const SCOPES_KEY = 'required_scopes';

export function RequireScopes(...scopes: string[]): MethodDecorator & ClassDecorator {
  return (target: any, key?: string | symbol, descriptor?: any) => {
    const decoratorFactory = Reflect.metadata(SCOPES_KEY, scopes);
    if (descriptor) {
      decoratorFactory(target, key!, descriptor);
      return descriptor;
    }
    decoratorFactory(target);
    return target;
  };
}

@Injectable()
export class ScopeGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredScopes = this.reflector.getAllAndOverride<string[]>(
      SCOPES_KEY,
      [context.getHandler(), context.getClass()],
    );

    // ถ้าไม่มี scope requirement ให้ผ่านได้
    if (!requiredScopes || requiredScopes.length === 0) {
      return true;
    }

    const request = context.switchToHttp().getRequest<Request>();
    const user = request.user as { scope?: string; sub?: string } | undefined;

    if (!user) {
      throw new UnauthorizedException('Authentication required');
    }

    const userScopes = (user.scope || '').split(' ').filter(Boolean);

    // ตรวจสอบว่ามี scope ที่ต้องการ (ALL scopes ต้องมี)
    const hasAllScopes = requiredScopes.every((scope) =>
      this.checkScope(userScopes, scope),
    );

    if (!hasAllScopes) {
      throw new ForbiddenException(
        `Insufficient scope. Required: ${requiredScopes.join(', ')}`,
      );
    }

    return true;
  }

  // รองรับ wildcard scopes (เช่น read:* ครอบคลุม read:users, read:orders)
  private checkScope(userScopes: string[], requiredScope: string): boolean {
    if (userScopes.includes(requiredScope)) return true;
    if (userScopes.includes('*')) return true;

    // ตรวจสอบ wildcard
    const [requiredAction, requiredResource] = requiredScope.split(':');

    return userScopes.some((scope) => {
      const [action, resource] = scope.split(':');
      
      if (action === requiredAction && resource === '*') return true;
      if (action === '*') return true;
      
      return false;
    });
  }
}

// Usage example:
// @RequireScopes('read:users', 'write:users')
// @UseGuards(JwtAuthGuard, ScopeGuard)
// async updateUser() {}
```

---

## 3. API Key Management

```typescript
// src/auth/api-keys/api-key.service.ts
import { Injectable, NotFoundException } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { createHash, randomBytes, timingSafeEqual } from 'crypto';
import { ApiKey } from './entities/api-key.entity';
import { ApiKeyAuditLog } from './entities/api-key-audit.entity';
import { InjectRedis } from '@nestjs-modules/ioredis';
import Redis from 'ioredis';

export interface CreateApiKeyDto {
  name: string;
  userId: string;
  scopes: string[];
  expiresAt?: Date;
  allowedIps?: string[];
  allowedDomains?: string[];
  rateLimit?: {
    requests: number;
    windowSeconds: number;
  };
}

export interface ApiKeyResult {
  id: string;
  key: string; // แสดงครั้งเดียวตอนสร้าง
  prefix: string;
  name: string;
  scopes: string[];
  expiresAt?: Date;
}

@Injectable()
export class ApiKeyService {
  private readonly KEY_PREFIX = 'sk_';
  private readonly KEY_LENGTH = 32;

  constructor(
    @InjectRepository(ApiKey)
    private readonly apiKeyRepo: Repository<ApiKey>,
    @InjectRepository(ApiKeyAuditLog)
    private readonly auditRepo: Repository<ApiKeyAuditLog>,
    @InjectRedis() private readonly redis: Redis,
  ) {}

  // สร้าง API key ใหม่
  async create(dto: CreateApiKeyDto): Promise<ApiKeyResult> {
    // สร้าง key
    const rawKey = randomBytes(this.KEY_LENGTH).toString('hex');
    const fullKey = `${this.KEY_PREFIX}${rawKey}`;
    const prefix = fullKey.slice(0, 8); // เก็บ prefix สำหรับ identification

    // Hash key ก่อนเก็บใน database (ไม่เก็บ plaintext)
    const hashedKey = this.hashKey(fullKey);

    const apiKey = await this.apiKeyRepo.save({
      hashedKey,
      prefix,
      name: dto.name,
      userId: dto.userId,
      scopes: dto.scopes,
      expiresAt: dto.expiresAt,
      allowedIps: dto.allowedIps || [],
      allowedDomains: dto.allowedDomains || [],
      rateLimit: dto.rateLimit,
      isActive: true,
      lastUsedAt: null,
      usageCount: 0,
    });

    // Audit log
    await this.logAudit({
      apiKeyId: apiKey.id,
      action: 'created',
      userId: dto.userId,
      metadata: { name: dto.name, scopes: dto.scopes },
    });

    return {
      id: apiKey.id,
      key: fullKey, // แสดงครั้งเดียว ไม่เก็บ
      prefix,
      name: dto.name,
      scopes: dto.scopes,
      expiresAt: dto.expiresAt,
    };
  }

  // ตรวจสอบ API key
  async verify(
    rawKey: string,
    ip?: string,
    domain?: string,
  ): Promise<{ valid: boolean; apiKey?: ApiKey; reason?: string }> {
    // ตรวจสอบ format
    if (!rawKey.startsWith(this.KEY_PREFIX)) {
      return { valid: false, reason: 'Invalid key format' };
    }

    // ดึง prefix จาก key
    const prefix = rawKey.slice(0, 8);

    // ตรวจสอบ cache ก่อน (ลด database load)
    const cacheKey = `apikey:${prefix}`;
    const cached = await this.redis.get(cacheKey);
    
    if (cached === 'invalid') {
      return { valid: false, reason: 'Invalid key (cached)' };
    }

    // หา key จาก database ด้วย prefix
    const apiKeys = await this.apiKeyRepo.find({
      where: { prefix, isActive: true },
    });

    if (apiKeys.length === 0) {
      await this.redis.setex(cacheKey, 300, 'invalid');
      return { valid: false, reason: 'Key not found' };
    }

    // ตรวจสอบ hash (timing-safe comparison)
    const hashedInputKey = this.hashKey(rawKey);
    const matchingKey = apiKeys.find((key) => {
      const expected = Buffer.from(key.hashedKey, 'hex');
      const actual = Buffer.from(hashedInputKey, 'hex');
      
      if (expected.length !== actual.length) return false;
      return timingSafeEqual(expected, actual);
    });

    if (!matchingKey) {
      await this.redis.setex(cacheKey, 300, 'invalid');
      return { valid: false, reason: 'Key hash mismatch' };
    }

    // ตรวจสอบ expiry
    if (matchingKey.expiresAt && matchingKey.expiresAt < new Date()) {
      return { valid: false, reason: 'Key expired' };
    }

    // ตรวจสอบ IP whitelist
    if (matchingKey.allowedIps.length > 0 && ip) {
      if (!matchingKey.allowedIps.includes(ip)) {
        return { valid: false, reason: 'IP not allowed' };
      }
    }

    // ตรวจสอบ domain whitelist
    if (matchingKey.allowedDomains.length > 0 && domain) {
      const isAllowed = matchingKey.allowedDomains.some((allowed) => {
        if (allowed.startsWith('*.')) {
          return domain.endsWith(allowed.slice(2));
        }
        return domain === allowed;
      });
      
      if (!isAllowed) {
        return { valid: false, reason: 'Domain not allowed' };
      }
    }

    // ตรวจสอบ rate limit
    if (matchingKey.rateLimit) {
      const rateLimitKey = `ratelimit:apikey:${matchingKey.id}`;
      const current = await this.redis.incr(rateLimitKey);
      
      if (current === 1) {
        await this.redis.expire(rateLimitKey, matchingKey.rateLimit.windowSeconds);
      }
      
      if (current > matchingKey.rateLimit.requests) {
        return { valid: false, reason: 'Rate limit exceeded' };
      }
    }

    // อัปเดต last used (async, ไม่ต้อง await)
    this.updateLastUsed(matchingKey.id).catch(console.error);

    return { valid: true, apiKey: matchingKey };
  }

  // Rotate API key
  async rotate(keyId: string, userId: string): Promise<ApiKeyResult> {
    const existingKey = await this.apiKeyRepo.findOne({
      where: { id: keyId, userId, isActive: true },
    });

    if (!existingKey) {
      throw new NotFoundException('API key not found');
    }

    // สร้าง key ใหม่ด้วย config เดิม
    const newKey = await this.create({
      name: existingKey.name,
      userId: existingKey.userId,
      scopes: existingKey.scopes,
      expiresAt: existingKey.expiresAt,
      allowedIps: existingKey.allowedIps,
      allowedDomains: existingKey.allowedDomains,
      rateLimit: existingKey.rateLimit,
    });

    // Deactivate key เก่า (grace period 24 ชั่วโมง)
    await this.apiKeyRepo.update(keyId, {
      isActive: false,
      rotatedAt: new Date(),
      rotatedToId: newKey.id,
    });

    // Audit log
    await this.logAudit({
      apiKeyId: keyId,
      action: 'rotated',
      userId,
      metadata: { newKeyId: newKey.id },
    });

    // ล้าง cache
    await this.redis.del(`apikey:${existingKey.prefix}`);

    return newKey;
  }

  // Revoke API key
  async revoke(keyId: string, userId: string, reason: string): Promise<void> {
    const key = await this.apiKeyRepo.findOne({
      where: { id: keyId, userId },
    });

    if (!key) {
      throw new NotFoundException('API key not found');
    }

    await this.apiKeyRepo.update(keyId, {
      isActive: false,
      revokedAt: new Date(),
      revokedReason: reason,
    });

    // ล้าง cache ทันที
    await this.redis.del(`apikey:${key.prefix}`);

    // Audit log
    await this.logAudit({
      apiKeyId: keyId,
      action: 'revoked',
      userId,
      metadata: { reason },
    });
  }

  // ดู audit log ของ API key
  async getAuditLog(keyId: string, userId: string): Promise<ApiKeyAuditLog[]> {
    return this.auditRepo.find({
      where: { apiKeyId: keyId },
      order: { createdAt: 'DESC' },
      take: 100,
    });
  }

  private hashKey(key: string): string {
    return createHash('sha256').update(key).digest('hex');
  }

  private async updateLastUsed(keyId: string): Promise<void> {
    await this.apiKeyRepo.increment({ id: keyId }, 'usageCount', 1);
    await this.apiKeyRepo.update(keyId, { lastUsedAt: new Date() });
  }

  private async logAudit(data: {
    apiKeyId: string;
    action: string;
    userId: string;
    metadata?: Record<string, unknown>;
  }): Promise<void> {
    await this.auditRepo.save(data);
  }
}
```

---

## 4. Input Validation ด้วย Zod

```typescript
// src/validation/schemas/user.schema.ts
import { z } from 'zod';

// Custom validators
const thaiPhoneNumber = z
  .string()
  .regex(/^(0[689]\d{8}|66[689]\d{8}|\+66[689]\d{8})$/, {
    message: 'Invalid Thai phone number format',
  });

const strongPassword = z
  .string()
  .min(8, 'Password must be at least 8 characters')
  .max(128, 'Password must be at most 128 characters')
  .regex(/[A-Z]/, 'Password must contain at least one uppercase letter')
  .regex(/[a-z]/, 'Password must contain at least one lowercase letter')
  .regex(/[0-9]/, 'Password must contain at least one number')
  .regex(/[^A-Za-z0-9]/, 'Password must contain at least one special character')
  .refine(
    (password) => {
      // ตรวจสอบ common passwords
      const commonPasswords = ['Password1!', 'Admin123!', 'Welcome1!'];
      return !commonPasswords.includes(password);
    },
    { message: 'Password is too common' },
  );

// User registration schema
export const CreateUserSchema = z.object({
  email: z
    .string()
    .email('Invalid email format')
    .toLowerCase()
    .max(255),
  
  username: z
    .string()
    .min(3, 'Username must be at least 3 characters')
    .max(50, 'Username must be at most 50 characters')
    .regex(/^[a-zA-Z0-9_-]+$/, 'Username can only contain letters, numbers, underscores, and hyphens'),
  
  password: strongPassword,
  
  firstName: z
    .string()
    .min(1, 'First name is required')
    .max(100)
    .regex(/^[฀-๿a-zA-Z\s'-]+$/, 'Invalid characters in name'),
  
  lastName: z
    .string()
    .min(1, 'Last name is required')
    .max(100)
    .regex(/^[฀-๿a-zA-Z\s'-]+$/, 'Invalid characters in name'),
  
  phone: thaiPhoneNumber.optional(),
  
  dateOfBirth: z
    .string()
    .regex(/^\d{4}-\d{2}-\d{2}$/, 'Date must be in YYYY-MM-DD format')
    .refine(
      (date) => {
        const d = new Date(date);
        const now = new Date();
        const age = now.getFullYear() - d.getFullYear();
        return age >= 13 && age <= 120;
      },
      { message: 'Age must be between 13 and 120 years' },
    )
    .optional(),
  
  acceptTerms: z.literal(true, {
    errorMap: () => ({ message: 'You must accept the terms and conditions' }),
  }),
});

export type CreateUserDto = z.infer<typeof CreateUserSchema>;

// Order creation schema
export const CreateOrderSchema = z.object({
  items: z
    .array(
      z.object({
        productId: z.string().uuid('Invalid product ID'),
        quantity: z
          .number()
          .int('Quantity must be an integer')
          .positive('Quantity must be positive')
          .max(1000, 'Quantity cannot exceed 1000'),
        price: z
          .number()
          .positive('Price must be positive')
          .multipleOf(0.01, 'Price must have at most 2 decimal places'),
      }),
    )
    .min(1, 'Order must have at least one item')
    .max(50, 'Order cannot have more than 50 items'),
  
  shippingAddress: z.object({
    street: z.string().min(1).max(255),
    city: z.string().min(1).max(100),
    province: z.string().min(1).max(100),
    postalCode: z
      .string()
      .regex(/^\d{5}$/, 'Postal code must be 5 digits'),
    country: z.enum(['TH', 'SG', 'MY', 'US', 'GB']),
  }),
  
  paymentMethod: z.enum(['credit_card', 'debit_card', 'bank_transfer', 'promptpay']),
  
  couponCode: z
    .string()
    .regex(/^[A-Z0-9]{6,20}$/, 'Invalid coupon code format')
    .optional(),
  
  notes: z
    .string()
    .max(500, 'Notes cannot exceed 500 characters')
    .optional()
    .transform((val) => val?.trim()),
});

export type CreateOrderDto = z.infer<typeof CreateOrderSchema>;

// Pagination schema
export const PaginationSchema = z.object({
  page: z
    .string()
    .optional()
    .transform((val) => (val ? parseInt(val, 10) : 1))
    .pipe(z.number().int().positive().max(10000)),
  
  limit: z
    .string()
    .optional()
    .transform((val) => (val ? parseInt(val, 10) : 20))
    .pipe(z.number().int().positive().max(100)),
  
  sort: z
    .string()
    .optional()
    .refine(
      (val) => !val || /^[a-zA-Z_]+(:(asc|desc))?$/.test(val),
      { message: 'Invalid sort format' },
    ),
  
  search: z
    .string()
    .max(100)
    .optional()
    .transform((val) => val?.trim()),
});
```

### 4.1 Zod Validation Middleware

```typescript
// src/validation/zod-validation.pipe.ts
import {
  PipeTransform,
  Injectable,
  ArgumentMetadata,
  BadRequestException,
} from '@nestjs/common';
import { ZodSchema, ZodError } from 'zod';

@Injectable()
export class ZodValidationPipe implements PipeTransform {
  constructor(private readonly schema: ZodSchema) {}

  transform(value: unknown, _metadata: ArgumentMetadata) {
    try {
      return this.schema.parse(value);
    } catch (error) {
      if (error instanceof ZodError) {
        const formattedErrors = error.errors.map((err) => ({
          field: err.path.join('.'),
          message: err.message,
          code: err.code,
        }));

        throw new BadRequestException({
          statusCode: 400,
          message: 'Validation failed',
          errors: formattedErrors,
        });
      }
      throw error;
    }
  }
}
```

---

## 5. SQL Injection Prevention

```typescript
// src/database/query-builder.ts
import { Pool, QueryConfig } from 'pg';

export class SafeQueryBuilder {
  private readonly pool: Pool;

  constructor(pool: Pool) {
    this.pool = pool;
  }

  // ✅ Parameterized queries เสมอ
  async findUserByEmail(email: string): Promise<any> {
    const query: QueryConfig = {
      text: 'SELECT id, email, username FROM users WHERE email = $1 AND status = $2',
      values: [email, 'active'],
    };

    const result = await this.pool.query(query);
    return result.rows[0];
  }

  // ✅ Dynamic column names ต้องผ่าน whitelist
  async findUsers(options: {
    sortColumn?: string;
    sortOrder?: 'ASC' | 'DESC';
    limit?: number;
    offset?: number;
    filters?: Record<string, unknown>;
  }): Promise<any[]> {
    // Whitelist ของ columns ที่อนุญาต
    const allowedColumns = new Set(['id', 'email', 'username', 'created_at', 'status']);
    
    const sortColumn = options.sortColumn && allowedColumns.has(options.sortColumn)
      ? options.sortColumn
      : 'created_at';
    
    // ✅ Sort order ต้องตรวจสอบก่อนใช้
    const sortOrder = options.sortOrder === 'ASC' ? 'ASC' : 'DESC';
    
    const params: unknown[] = [];
    const whereConditions: string[] = [];
    
    // Dynamic filters พร้อม parameterized values
    if (options.filters) {
      for (const [key, value] of Object.entries(options.filters)) {
        // ✅ ตรวจสอบ column name ก่อนใช้
        if (!allowedColumns.has(key)) continue;
        
        params.push(value);
        whereConditions.push(`${key} = $${params.length}`);
      }
    }
    
    const whereClause = whereConditions.length > 0
      ? `WHERE ${whereConditions.join(' AND ')}`
      : '';
    
    const limit = Math.min(options.limit || 20, 100);
    const offset = options.offset || 0;
    
    // ✅ limit และ offset เป็น integer ไม่สามารถ inject ได้
    const query = `
      SELECT id, email, username, created_at, status
      FROM users
      ${whereClause}
      ORDER BY ${sortColumn} ${sortOrder}
      LIMIT ${limit}
      OFFSET ${offset}
    `;
    
    const result = await this.pool.query(query, params);
    return result.rows;
  }

  // ✅ Full-text search แบบปลอดภัย
  async searchUsers(searchTerm: string): Promise<any[]> {
    // ❌ ไม่ทำแบบนี้: `WHERE username LIKE '%${searchTerm}%'`
    
    // ✅ ใช้ parameterized query
    const result = await this.pool.query(
      `SELECT id, email, username
       FROM users
       WHERE 
         username ILIKE $1 OR
         email ILIKE $1 OR
         to_tsvector('english', username || ' ' || email) @@ plainto_tsquery('english', $2)
       LIMIT 20`,
      [`%${searchTerm.replace(/[%_\\]/g, '\\$&')}%`, searchTerm],
    );
    
    return result.rows;
  }

  // ✅ Bulk insert แบบปลอดภัย
  async bulkInsertUsers(users: Array<{
    email: string;
    username: string;
    hashedPassword: string;
  }>): Promise<void> {
    if (users.length === 0) return;
    
    // สร้าง parameterized bulk insert
    const placeholders = users.map(
      (_, i) => `($${i * 3 + 1}, $${i * 3 + 2}, $${i * 3 + 3})`,
    ).join(', ');
    
    const values = users.flatMap((u) => [u.email, u.username, u.hashedPassword]);
    
    await this.pool.query(
      `INSERT INTO users (email, username, hashed_password)
       VALUES ${placeholders}
       ON CONFLICT (email) DO NOTHING`,
      values,
    );
  }
}
```

---

## 6. XSS Prevention

```typescript
// src/security/xss-sanitizer.ts
import * as DOMPurify from 'isomorphic-dompurify';
import { JSDOM } from 'jsdom';
import * as he from 'he';

const { window } = new JSDOM('');
const purify = DOMPurify(window as unknown as Window & typeof globalThis);

export class XSSSanitizer {
  // Sanitize HTML content (สำหรับ rich text)
  static sanitizeHtml(dirty: string): string {
    return purify.sanitize(dirty, {
      ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'p', 'br', 'ul', 'ol', 'li', 'a'],
      ALLOWED_ATTR: ['href', 'target', 'rel'],
      ALLOW_DATA_ATTR: false,
      ADD_ATTR: ['target'],
      // Force all links to be safe
      FORCE_BODY: true,
    });
  }

  // Escape HTML entities (สำหรับ plain text)
  static escapeHtml(text: string): string {
    return he.encode(text, {
      useNamedReferences: true,
      decimal: false,
      encodeEverything: false,
    });
  }

  // Sanitize สำหรับ JSON context
  static sanitizeForJson(value: string): string {
    return value
      .replace(/</g, '\\u003C')
      .replace(/>/g, '\\u003E')
      .replace(/&/g, '\\u0026')
      .replace(/'/g, '\\u0027');
  }

  // Sanitize URL
  static sanitizeUrl(url: string): string | null {
    try {
      const parsed = new URL(url);
      
      // อนุญาตเฉพาะ http และ https
      if (!['http:', 'https:'].includes(parsed.protocol)) {
        return null;
      }
      
      return parsed.toString();
    } catch {
      return null;
    }
  }

  // Deep sanitize object
  static sanitizeObject<T extends object>(obj: T): T {
    const sanitized = { ...obj };
    
    for (const [key, value] of Object.entries(sanitized)) {
      if (typeof value === 'string') {
        (sanitized as Record<string, unknown>)[key] = this.escapeHtml(value);
      } else if (typeof value === 'object' && value !== null && !Array.isArray(value)) {
        (sanitized as Record<string, unknown>)[key] = this.sanitizeObject(value as object);
      } else if (Array.isArray(value)) {
        (sanitized as Record<string, unknown>)[key] = value.map((item) =>
          typeof item === 'string' ? this.escapeHtml(item) : item,
        );
      }
    }
    
    return sanitized;
  }
}
```

### 6.1 Security Headers Middleware

```typescript
// src/security/security-headers.middleware.ts
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';
import helmet from 'helmet';

@Injectable()
export class SecurityHeadersMiddleware implements NestMiddleware {
  private readonly helmetMiddleware: ReturnType<typeof helmet>;

  constructor() {
    this.helmetMiddleware = helmet({
      // Content Security Policy
      contentSecurityPolicy: {
        directives: {
          defaultSrc: ["'self'"],
          styleSrc: ["'self'", "'unsafe-inline'", 'https://fonts.googleapis.com'],
          fontSrc: ["'self'", 'https://fonts.gstatic.com'],
          imgSrc: ["'self'", 'data:', 'https:'],
          scriptSrc: ["'self'"],
          connectSrc: ["'self'", 'https://api.myapp.com'],
          frameSrc: ["'none'"],
          objectSrc: ["'none'"],
          upgradeInsecureRequests: [],
        },
      },
      
      // ป้องกัน clickjacking
      frameguard: { action: 'deny' },
      
      // ป้องกัน MIME sniffing
      noSniff: true,
      
      // Force HTTPS
      strictTransportSecurity: {
        maxAge: 31536000,
        includeSubDomains: true,
        preload: true,
      },
      
      // ปิด X-Powered-By header
      hidePoweredBy: true,
      
      // Referrer policy
      referrerPolicy: { policy: 'strict-origin-when-cross-origin' },
      
      // Permissions policy
      permittedCrossDomainPolicies: false,
    });
  }

  use(req: Request, res: Response, next: NextFunction): void {
    this.helmetMiddleware(req, res, next);
  }
}
```

---

## 7. CORS Configuration

```typescript
// src/config/cors.config.ts
import { CorsOptions } from '@nestjs/common/interfaces/external/cors-options.interface';

const ALLOWED_ORIGINS_PROD = [
  'https://myapp.com',
  'https://www.myapp.com',
  'https://admin.myapp.com',
  'https://app.myapp.com',
];

const ALLOWED_ORIGINS_STAGING = [
  'https://staging.myapp.com',
  'https://staging-admin.myapp.com',
];

const ALLOWED_ORIGINS_DEV = [
  'http://localhost:3000',
  'http://localhost:3001',
  'http://127.0.0.1:3000',
];

export function getCorsConfig(): CorsOptions {
  const env = process.env.NODE_ENV || 'development';

  let allowedOrigins: string[];

  switch (env) {
    case 'production':
      allowedOrigins = ALLOWED_ORIGINS_PROD;
      break;
    case 'staging':
      allowedOrigins = [...ALLOWED_ORIGINS_STAGING, ...ALLOWED_ORIGINS_PROD];
      break;
    default:
      allowedOrigins = [
        ...ALLOWED_ORIGINS_DEV,
        ...ALLOWED_ORIGINS_STAGING,
      ];
  }

  return {
    origin: (origin, callback) => {
      // อนุญาต requests ที่ไม่มี origin (mobile apps, curl, etc.)
      if (!origin) {
        callback(null, true);
        return;
      }

      if (allowedOrigins.includes(origin)) {
        callback(null, true);
      } else {
        callback(new Error(`CORS: Origin ${origin} not allowed`));
      }
    },
    methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS'],
    allowedHeaders: [
      'Content-Type',
      'Authorization',
      'X-API-Key',
      'X-Request-ID',
      'X-Correlation-ID',
    ],
    exposedHeaders: [
      'X-Request-ID',
      'X-Rate-Limit-Remaining',
      'X-Rate-Limit-Reset',
    ],
    credentials: true,
    maxAge: 86400, // 24 hours preflight cache
    preflightContinue: false,
    optionsSuccessStatus: 204,
  };
}
```

---

## 8. Rate Limiting

```typescript
// src/security/rate-limiter.ts
import { Injectable } from '@nestjs/common';
import { InjectRedis } from '@nestjs-modules/ioredis';
import Redis from 'ioredis';
import { TooManyRequestsException } from '@nestjs/common';

interface RateLimitConfig {
  windowSeconds: number;
  maxRequests: number;
  keyPrefix: string;
}

@Injectable()
export class RateLimiterService {
  constructor(@InjectRedis() private readonly redis: Redis) {}

  async checkRateLimit(
    identifier: string,
    config: RateLimitConfig,
  ): Promise<{
    allowed: boolean;
    remaining: number;
    resetAt: number;
    retryAfter?: number;
  }> {
    const key = `${config.keyPrefix}:${identifier}`;
    const now = Date.now();
    const windowStart = now - config.windowSeconds * 1000;

    // ใช้ sliding window algorithm
    const pipeline = this.redis.pipeline();
    
    // ลบ requests เก่า
    pipeline.zremrangebyscore(key, 0, windowStart);
    
    // นับ requests ปัจจุบัน
    pipeline.zcard(key);
    
    // เพิ่ม request ปัจจุบัน
    pipeline.zadd(key, now, `${now}-${Math.random()}`);
    
    // Set expiry
    pipeline.expire(key, config.windowSeconds + 1);
    
    const results = await pipeline.exec();
    const currentCount = (results?.[1]?.[1] as number) || 0;

    const allowed = currentCount < config.maxRequests;
    const remaining = Math.max(0, config.maxRequests - currentCount - 1);
    const resetAt = Math.floor((now + config.windowSeconds * 1000) / 1000);

    if (!allowed) {
      // คำนวณเวลาที่ต้องรอ
      const oldestRequest = await this.redis.zrange(key, 0, 0, 'WITHSCORES');
      const oldestTime = oldestRequest[1] ? parseInt(oldestRequest[1]) : now;
      const retryAfter = Math.ceil((oldestTime + config.windowSeconds * 1000 - now) / 1000);

      return { allowed: false, remaining: 0, resetAt, retryAfter };
    }

    return { allowed: true, remaining, resetAt };
  }

  // Rate limit สำหรับ different endpoints
  async checkApiRateLimit(userId: string, endpoint: string): Promise<void> {
    const configs: Record<string, RateLimitConfig> = {
      '/api/auth/login': {
        windowSeconds: 900, // 15 minutes
        maxRequests: 5,
        keyPrefix: 'ratelimit:login',
      },
      '/api/auth/forgot-password': {
        windowSeconds: 3600,
        maxRequests: 3,
        keyPrefix: 'ratelimit:forgot-password',
      },
      default: {
        windowSeconds: 60,
        maxRequests: 100,
        keyPrefix: 'ratelimit:api',
      },
    };

    const config = configs[endpoint] || configs.default;
    const result = await this.checkRateLimit(userId, config);

    if (!result.allowed) {
      throw new TooManyRequestsException({
        message: 'Too many requests',
        retryAfter: result.retryAfter,
        resetAt: result.resetAt,
      });
    }
  }
}
```

---

## 9. Security Testing

```typescript
// src/security/__tests__/sql-injection.spec.ts
import { SafeQueryBuilder } from '../database/query-builder';
import { Pool } from 'pg';

describe('SQL Injection Prevention', () => {
  let queryBuilder: SafeQueryBuilder;
  let pool: Pool;

  beforeAll(async () => {
    pool = new Pool({ connectionString: process.env.TEST_DATABASE_URL });
    queryBuilder = new SafeQueryBuilder(pool);
  });

  afterAll(async () => {
    await pool.end();
  });

  const sqlInjectionPayloads = [
    "'; DROP TABLE users; --",
    "' OR '1'='1",
    "' OR 1=1 --",
    "'; INSERT INTO users VALUES ('hacker', 'hacked', 'hacked@evil.com'); --",
    "' UNION SELECT * FROM users --",
    "1; SELECT * FROM information_schema.tables",
    "admin'--",
    "' OR ''='",
    "'; EXEC xp_cmdshell('dir'); --",
  ];

  for (const payload of sqlInjectionPayloads) {
    it(`should safely handle SQL injection: ${payload.slice(0, 30)}...`, async () => {
      // ไม่ควร throw และไม่ควรดึงข้อมูลที่ไม่ควรเห็น
      const result = await queryBuilder.findUserByEmail(payload);
      expect(result).toBeUndefined();
    });
  }

  it('should use parameterized queries', async () => {
    // ตรวจสอบว่า query ใช้ parameterized form
    const spy = jest.spyOn(pool, 'query');
    
    await queryBuilder.findUserByEmail('test@example.com');
    
    expect(spy).toHaveBeenCalledWith(
      expect.objectContaining({
        text: expect.stringContaining('$1'),
        values: expect.arrayContaining(['test@example.com']),
      }),
    );
  });
});
```

---

## 10. Main Application Security Setup

```typescript
// src/main.ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { getCorsConfig } from './config/cors.config';
import * as compression from 'compression';

async function bootstrap() {
  const app = await NestFactory.create(AppModule, {
    logger: ['error', 'warn', 'log'],
    // ป้องกัน request body ใหญ่เกินไป
    bodyParser: false,
  });

  // ตั้งค่า CORS
  app.enableCors(getCorsConfig());

  // Body parser พร้อม size limit
  const bodyParserLib = require('body-parser');
  app.use(bodyParserLib.json({ limit: '1mb' }));
  app.use(bodyParserLib.urlencoded({ extended: true, limit: '1mb' }));

  // Compression
  app.use(compression());

  // Global prefix
  app.setGlobalPrefix('api/v1');

  // Versioning
  app.enableVersioning();

  await app.listen(3000);
  console.log('Application started on port 3000');
}

bootstrap();
```

---

## สรุป

| หัวข้อ | เทคโนโลยี | วัตถุประสงค์ |
|--------|-----------|-------------|
| OAuth 2.0 PKCE | Authorization Code + PKCE | ป้องกัน authorization code interception ใน SPAs |
| Token Introspection | JWT + Redis revocation | ตรวจสอบ token validity และ revoke tokens |
| Scope-based Auth | Custom decorator + Guard | Fine-grained permission control |
| API Key Management | Hash + rotation + audit | จัดการ API keys สำหรับ machine-to-machine |
| Input Validation | Zod schema validation | ป้องกัน invalid data เข้าระบบ |
| SQL Injection | Parameterized queries | ป้องกัน database attacks |
| XSS Prevention | DOMPurify + CSP headers | ป้องกัน cross-site scripting |
| CORS | Origin whitelist | ป้องกัน unauthorized cross-origin requests |
| Rate Limiting | Redis sliding window | ป้องกัน brute force และ DDoS |
| Security Headers | Helmet.js | HTTP security headers (CSP, HSTS, etc.) |

> **Best Practice**: ใช้ Defense in Depth - ป้องกันหลายชั้น ไม่พึ่งเพียง layer เดียว และ rotate credentials เป็นประจำ

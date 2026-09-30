# Part 48: API Security Advanced — OAuth 2.0, Token Management, และการป้องกันช่องโหว่

ในบทนี้เราจะเรียนรู้การรักษาความปลอดภัย API ในระดับ Production รวมถึง OAuth 2.0 PKCE Flow, การจัดการ Token, Scope-based Authorization, การจัดการ API Key, และการป้องกันช่องโหว่ต่างๆ

---

## 1. OAuth 2.0 PKCE Flow Implementation

PKCE (Proof Key for Code Exchange) เป็น extension ของ OAuth 2.0 ที่ออกแบบมาเพื่อป้องกัน authorization code interception attacks โดยเฉพาะสำหรับ public clients เช่น SPA และ Mobile Apps

### 1.1 ทำความเข้าใจ PKCE Flow

```
Client                          Authorization Server
  |                                      |
  |--1. Generate code_verifier---------->|
  |--2. Hash to code_challenge          |
  |--3. Authorization Request+challenge->|
  |                                      |--4. User authenticates
  |<--5. Authorization Code--------------|
  |--6. Token Request+code_verifier----->|
  |                                      |--7. Verify: hash(code_verifier)==code_challenge
  |<--8. Access Token + Refresh Token----|
```

### 1.2 Authorization Server Implementation

```typescript
// src/auth/pkce-auth-server.ts
import express from 'express';
import crypto from 'crypto';
import { z } from 'zod';
import jwt from 'jsonwebtoken';
import { Redis } from 'ioredis';
import { db } from '../database';

const redis = new Redis(process.env.REDIS_URL!);
const router = express.Router();

// Schema validation
const AuthorizeRequestSchema = z.object({
  response_type: z.literal('code'),
  client_id: z.string().min(1),
  redirect_uri: z.string().url(),
  scope: z.string(),
  state: z.string().min(16),
  code_challenge: z.string().min(43).max(128),
  code_challenge_method: z.enum(['S256', 'plain']),
});

const TokenRequestSchema = z.object({
  grant_type: z.enum(['authorization_code', 'refresh_token', 'client_credentials']),
  code: z.string().optional(),
  redirect_uri: z.string().url().optional(),
  client_id: z.string(),
  client_secret: z.string().optional(),
  code_verifier: z.string().min(43).max(128).optional(),
  refresh_token: z.string().optional(),
  scope: z.string().optional(),
});

interface AuthorizationCode {
  code: string;
  clientId: string;
  userId: string;
  redirectUri: string;
  scope: string[];
  codeChallenge: string;
  codeChallengeMethod: 'S256' | 'plain';
  expiresAt: number;
}

// Generate secure authorization code
function generateAuthCode(): string {
  return crypto.randomBytes(32).toString('base64url');
}

// Verify PKCE code verifier against stored challenge
function verifyCodeChallenge(
  verifier: string,
  challenge: string,
  method: 'S256' | 'plain'
): boolean {
  if (method === 'S256') {
    const hash = crypto
      .createHash('sha256')
      .update(verifier)
      .digest('base64url');
    // Constant-time comparison to prevent timing attacks
    return crypto.timingSafeEqual(
      Buffer.from(hash),
      Buffer.from(challenge)
    );
  }
  // 'plain' method (not recommended for production)
  return crypto.timingSafeEqual(
    Buffer.from(verifier),
    Buffer.from(challenge)
  );
}

// Authorization endpoint
router.get('/authorize', async (req, res) => {
  try {
    const params = AuthorizeRequestSchema.parse(req.query);
    
    // Validate client
    const client = await db('oauth_clients')
      .where({ client_id: params.client_id, is_active: true })
      .first();
    
    if (!client) {
      return res.redirect(`${params.redirect_uri}?error=invalid_client`);
    }
    
    // Validate redirect URI against registered URIs
    const allowedUris: string[] = client.redirect_uris;
    if (!allowedUris.includes(params.redirect_uri)) {
      return res.status(400).json({ error: 'invalid_redirect_uri' });
    }
    
    // Validate requested scopes
    const requestedScopes = params.scope.split(' ');
    const allowedScopes: string[] = client.allowed_scopes;
    const invalidScopes = requestedScopes.filter(s => !allowedScopes.includes(s));
    
    if (invalidScopes.length > 0) {
      return res.redirect(
        `${params.redirect_uri}?error=invalid_scope&error_description=${encodeURIComponent(`Invalid scopes: ${invalidScopes.join(', ')}`)}`
      );
    }
    
    // Store authorization request in session
    const sessionKey = `auth_session:${params.state}`;
    await redis.setex(sessionKey, 600, JSON.stringify({
      clientId: params.client_id,
      redirectUri: params.redirect_uri,
      scope: requestedScopes,
      codeChallenge: params.code_challenge,
      codeChallengeMethod: params.code_challenge_method,
      state: params.state,
    }));
    
    // Redirect to login page if not authenticated
    if (!req.session?.userId) {
      return res.redirect(`/login?return_to=${encodeURIComponent(req.originalUrl)}`);
    }
    
    // Show consent screen
    res.render('consent', {
      client,
      scopes: requestedScopes,
      state: params.state,
    });
  } catch (error) {
    if (error instanceof z.ZodError) {
      return res.status(400).json({
        error: 'invalid_request',
        error_description: error.errors.map(e => e.message).join(', '),
      });
    }
    console.error('Authorization error:', error);
    res.status(500).json({ error: 'server_error' });
  }
});

// Consent submission endpoint
router.post('/authorize/consent', async (req, res) => {
  const { state, approved } = req.body;
  const userId = req.session?.userId;
  
  if (!userId) {
    return res.redirect('/login');
  }
  
  const sessionKey = `auth_session:${state}`;
  const sessionData = await redis.get(sessionKey);
  
  if (!sessionData) {
    return res.status(400).json({ error: 'invalid_state' });
  }
  
  const authSession = JSON.parse(sessionData);
  
  if (!approved) {
    await redis.del(sessionKey);
    return res.redirect(
      `${authSession.redirectUri}?error=access_denied&state=${state}`
    );
  }
  
  // Generate authorization code
  const code = generateAuthCode();
  const authCodeData: AuthorizationCode = {
    code,
    clientId: authSession.clientId,
    userId,
    redirectUri: authSession.redirectUri,
    scope: authSession.scope,
    codeChallenge: authSession.codeChallenge,
    codeChallengeMethod: authSession.codeChallengeMethod,
    expiresAt: Date.now() + 10 * 60 * 1000, // 10 minutes
  };
  
  // Store authorization code (short-lived)
  await redis.setex(
    `auth_code:${code}`,
    600,
    JSON.stringify(authCodeData)
  );
  
  await redis.del(sessionKey);
  
  res.redirect(
    `${authSession.redirectUri}?code=${code}&state=${state}`
  );
});

// Token endpoint
router.post('/token', async (req, res) => {
  try {
    const params = TokenRequestSchema.parse(req.body);
    
    if (params.grant_type === 'authorization_code') {
      return handleAuthorizationCodeGrant(params, res);
    } else if (params.grant_type === 'refresh_token') {
      return handleRefreshTokenGrant(params, res);
    } else if (params.grant_type === 'client_credentials') {
      return handleClientCredentialsGrant(params, res);
    }
    
    return res.status(400).json({ error: 'unsupported_grant_type' });
  } catch (error) {
    if (error instanceof z.ZodError) {
      return res.status(400).json({
        error: 'invalid_request',
        error_description: error.errors.map(e => e.message).join(', '),
      });
    }
    console.error('Token error:', error);
    res.status(500).json({ error: 'server_error' });
  }
});

async function handleAuthorizationCodeGrant(params: any, res: any) {
  if (!params.code || !params.redirect_uri || !params.code_verifier) {
    return res.status(400).json({
      error: 'invalid_request',
      error_description: 'Missing required parameters',
    });
  }
  
  // Retrieve authorization code
  const codeData = await redis.get(`auth_code:${params.code}`);
  
  if (!codeData) {
    return res.status(400).json({ error: 'invalid_grant' });
  }
  
  const authCode: AuthorizationCode = JSON.parse(codeData);
  
  // Delete code immediately (single use)
  await redis.del(`auth_code:${params.code}`);
  
  // Verify expiration
  if (Date.now() > authCode.expiresAt) {
    return res.status(400).json({
      error: 'invalid_grant',
      error_description: 'Authorization code expired',
    });
  }
  
  // Verify client
  if (authCode.clientId !== params.client_id) {
    return res.status(400).json({
      error: 'invalid_grant',
      error_description: 'Client mismatch',
    });
  }
  
  // Verify redirect URI
  if (authCode.redirectUri !== params.redirect_uri) {
    return res.status(400).json({
      error: 'invalid_grant',
      error_description: 'Redirect URI mismatch',
    });
  }
  
  // Verify PKCE
  if (!verifyCodeChallenge(
    params.code_verifier,
    authCode.codeChallenge,
    authCode.codeChallengeMethod
  )) {
    return res.status(400).json({
      error: 'invalid_grant',
      error_description: 'Code verifier mismatch',
    });
  }
  
  // Generate tokens
  const accessToken = generateAccessToken(authCode.userId, authCode.scope, params.client_id);
  const refreshToken = generateRefreshToken();
  
  // Store refresh token
  await redis.setex(
    `refresh_token:${refreshToken}`,
    30 * 24 * 60 * 60, // 30 days
    JSON.stringify({
      userId: authCode.userId,
      clientId: params.client_id,
      scope: authCode.scope,
    })
  );
  
  res.json({
    access_token: accessToken,
    token_type: 'Bearer',
    expires_in: 3600,
    refresh_token: refreshToken,
    scope: authCode.scope.join(' '),
  });
}

async function handleRefreshTokenGrant(params: any, res: any) {
  if (!params.refresh_token) {
    return res.status(400).json({
      error: 'invalid_request',
      error_description: 'Missing refresh_token',
    });
  }
  
  const tokenData = await redis.get(`refresh_token:${params.refresh_token}`);
  
  if (!tokenData) {
    return res.status(400).json({ error: 'invalid_grant' });
  }
  
  const stored = JSON.parse(tokenData);
  
  if (stored.clientId !== params.client_id) {
    return res.status(400).json({ error: 'invalid_grant' });
  }
  
  // Rotate refresh token
  await redis.del(`refresh_token:${params.refresh_token}`);
  
  const newAccessToken = generateAccessToken(stored.userId, stored.scope, params.client_id);
  const newRefreshToken = generateRefreshToken();
  
  await redis.setex(
    `refresh_token:${newRefreshToken}`,
    30 * 24 * 60 * 60,
    JSON.stringify(stored)
  );
  
  res.json({
    access_token: newAccessToken,
    token_type: 'Bearer',
    expires_in: 3600,
    refresh_token: newRefreshToken,
    scope: stored.scope.join(' '),
  });
}

async function handleClientCredentialsGrant(params: any, res: any) {
  if (!params.client_secret) {
    return res.status(401).json({ error: 'invalid_client' });
  }
  
  const client = await db('oauth_clients')
    .where({ client_id: params.client_id, is_active: true })
    .first();
  
  if (!client) {
    return res.status(401).json({ error: 'invalid_client' });
  }
  
  const secretHash = crypto
    .createHash('sha256')
    .update(params.client_secret)
    .digest('hex');
  
  if (!crypto.timingSafeEqual(
    Buffer.from(secretHash),
    Buffer.from(client.client_secret_hash)
  )) {
    return res.status(401).json({ error: 'invalid_client' });
  }
  
  const requestedScopes = params.scope ? params.scope.split(' ') : client.allowed_scopes;
  const accessToken = generateAccessToken(
    `client:${params.client_id}`,
    requestedScopes,
    params.client_id
  );
  
  res.json({
    access_token: accessToken,
    token_type: 'Bearer',
    expires_in: 3600,
    scope: requestedScopes.join(' '),
  });
}

function generateAccessToken(userId: string, scope: string[], clientId: string): string {
  return jwt.sign(
    {
      sub: userId,
      scope: scope.join(' '),
      client_id: clientId,
      iat: Math.floor(Date.now() / 1000),
    },
    process.env.JWT_SECRET!,
    {
      expiresIn: '1h',
      issuer: process.env.OAUTH_ISSUER,
      audience: process.env.OAUTH_AUDIENCE,
    }
  );
}

function generateRefreshToken(): string {
  return crypto.randomBytes(48).toString('base64url');
}

export { router as authRouter };
```

---

## 2. Token Introspection Endpoint

Token Introspection (RFC 7662) ช่วยให้ Resource Server ตรวจสอบ token ได้โดยไม่ต้องมี shared secret

```typescript
// src/auth/token-introspection.ts
import express from 'express';
import jwt from 'jsonwebtoken';
import { Redis } from 'ioredis';
import { db } from '../database';

const redis = new Redis(process.env.REDIS_URL!);
const router = express.Router();

interface IntrospectionResponse {
  active: boolean;
  scope?: string;
  client_id?: string;
  username?: string;
  token_type?: string;
  exp?: number;
  iat?: number;
  nbf?: number;
  sub?: string;
  aud?: string | string[];
  iss?: string;
  jti?: string;
}

// Middleware to authenticate introspection requests
async function authenticateIntrospectionClient(
  req: express.Request,
  res: express.Response,
  next: express.NextFunction
) {
  const authHeader = req.headers.authorization;
  
  if (!authHeader || !authHeader.startsWith('Basic ')) {
    return res.status(401).json({ error: 'invalid_client' });
  }
  
  const credentials = Buffer.from(
    authHeader.slice(6),
    'base64'
  ).toString('utf-8');
  
  const [clientId, clientSecret] = credentials.split(':');
  
  if (!clientId || !clientSecret) {
    return res.status(401).json({ error: 'invalid_client' });
  }
  
  const client = await db('oauth_clients')
    .where({ client_id: clientId, is_active: true, can_introspect: true })
    .first();
  
  if (!client) {
    return res.status(401).json({ error: 'invalid_client' });
  }
  
  const secretHash = crypto
    .createHash('sha256')
    .update(clientSecret)
    .digest('hex');
  
  if (!crypto.timingSafeEqual(
    Buffer.from(secretHash),
    Buffer.from(client.client_secret_hash)
  )) {
    return res.status(401).json({ error: 'invalid_client' });
  }
  
  req.introspectionClient = client;
  next();
}

router.post('/introspect', authenticateIntrospectionClient, async (req, res) => {
  const { token, token_type_hint } = req.body;
  
  if (!token) {
    return res.status(400).json({ error: 'invalid_request' });
  }
  
  // Check revocation list first
  const isRevoked = await redis.sismember('revoked_tokens', token);
  if (isRevoked) {
    return res.json({ active: false });
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET!) as any;
    
    // Get user info if sub is a user (not client credentials)
    let username: string | undefined;
    if (!decoded.sub.startsWith('client:')) {
      const user = await db('users')
        .where({ id: decoded.sub })
        .select('username', 'email')
        .first();
      username = user?.username;
    }
    
    const response: IntrospectionResponse = {
      active: true,
      scope: decoded.scope,
      client_id: decoded.client_id,
      username,
      token_type: 'Bearer',
      exp: decoded.exp,
      iat: decoded.iat,
      sub: decoded.sub,
      aud: decoded.aud,
      iss: decoded.iss,
      jti: decoded.jti,
    };
    
    res.json(response);
  } catch (error) {
    // Token is invalid or expired
    res.json({ active: false });
  }
});

// Token revocation endpoint (RFC 7009)
router.post('/revoke', authenticateIntrospectionClient, async (req, res) => {
  const { token, token_type_hint } = req.body;
  
  if (!token) {
    return res.status(400).json({ error: 'invalid_request' });
  }
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET!, {
      ignoreExpiration: true
    }) as any;
    
    // Add to revocation set with expiry matching token expiry
    const ttl = decoded.exp - Math.floor(Date.now() / 1000);
    if (ttl > 0) {
      await redis.setex(`revoked:${decoded.jti || token}`, ttl, '1');
    }
    
    // Also handle refresh tokens
    if (token_type_hint === 'refresh_token') {
      await redis.del(`refresh_token:${token}`);
    }
  } catch {
    // Invalid token - still return success per RFC 7009
  }
  
  res.status(200).send();
});

export { router as introspectionRouter };
```

---

## 3. Scope-based Authorization Middleware

```typescript
// src/middleware/scope-authorization.ts
import { Request, Response, NextFunction } from 'express';
import jwt from 'jsonwebtoken';
import { Redis } from 'ioredis';

const redis = new Redis(process.env.REDIS_URL!);

interface TokenPayload {
  sub: string;
  scope: string;
  client_id: string;
  exp: number;
  iat: number;
  iss: string;
}

declare global {
  namespace Express {
    interface Request {
      user?: {
        id: string;
        scopes: string[];
        clientId: string;
      };
    }
  }
}

// Authenticate and extract token claims
export function authenticate() {
  return async (req: Request, res: Response, next: NextFunction) => {
    const authHeader = req.headers.authorization;
    
    if (!authHeader || !authHeader.startsWith('Bearer ')) {
      return res.status(401).json({
        error: 'unauthorized',
        message: 'Missing or invalid authorization header',
      });
    }
    
    const token = authHeader.slice(7);
    
    try {
      // Check if token is revoked
      const payload = jwt.decode(token) as TokenPayload;
      if (payload?.jti) {
        const isRevoked = await redis.exists(`revoked:${payload.jti}`);
        if (isRevoked) {
          return res.status(401).json({
            error: 'token_revoked',
            message: 'Token has been revoked',
          });
        }
      }
      
      const verified = jwt.verify(token, process.env.JWT_SECRET!, {
        issuer: process.env.OAUTH_ISSUER,
        audience: process.env.OAUTH_AUDIENCE,
      }) as TokenPayload;
      
      req.user = {
        id: verified.sub,
        scopes: verified.scope ? verified.scope.split(' ') : [],
        clientId: verified.client_id,
      };
      
      next();
    } catch (error) {
      if (error instanceof jwt.TokenExpiredError) {
        return res.status(401).json({
          error: 'token_expired',
          message: 'Access token has expired',
        });
      }
      
      return res.status(401).json({
        error: 'invalid_token',
        message: 'Invalid access token',
      });
    }
  };
}

// Require specific scopes
export function requireScopes(...requiredScopes: string[]) {
  return (req: Request, res: Response, next: NextFunction) => {
    if (!req.user) {
      return res.status(401).json({ error: 'unauthorized' });
    }
    
    const userScopes = new Set(req.user.scopes);
    const missingScopes = requiredScopes.filter(s => !userScopes.has(s));
    
    if (missingScopes.length > 0) {
      return res.status(403).json({
        error: 'insufficient_scope',
        message: `Required scopes: ${requiredScopes.join(', ')}`,
        missing_scopes: missingScopes,
      });
    }
    
    next();
  };
}

// Require any of the specified scopes
export function requireAnyScope(...scopes: string[]) {
  return (req: Request, res: Response, next: NextFunction) => {
    if (!req.user) {
      return res.status(401).json({ error: 'unauthorized' });
    }
    
    const userScopes = new Set(req.user.scopes);
    const hasScope = scopes.some(s => userScopes.has(s));
    
    if (!hasScope) {
      return res.status(403).json({
        error: 'insufficient_scope',
        message: `Required one of: ${scopes.join(', ')}`,
      });
    }
    
    next();
  };
}

// Role-based scope checking
export function requireRole(role: string) {
  const scopeMap: Record<string, string[]> = {
    admin: ['admin:read', 'admin:write', 'users:read', 'users:write'],
    editor: ['content:read', 'content:write'],
    viewer: ['content:read'],
  };
  
  const requiredScopes = scopeMap[role];
  if (!requiredScopes) {
    throw new Error(`Unknown role: ${role}`);
  }
  
  return requireAnyScope(...requiredScopes);
}

// Usage example
const app = express();

app.get('/api/users',
  authenticate(),
  requireScopes('users:read'),
  async (req, res) => {
    // Handler
  }
);

app.post('/api/admin/users',
  authenticate(),
  requireScopes('admin:write'),
  async (req, res) => {
    // Handler
  }
);
```

---

## 4. API Key Management

```typescript
// src/services/api-key.service.ts
import crypto from 'crypto';
import { db } from '../database';
import { Redis } from 'ioredis';

const redis = new Redis(process.env.REDIS_URL!);

interface ApiKeyConfig {
  name: string;
  ownerId: string;
  scopes: string[];
  expiresAt?: Date;
  rateLimit?: {
    requestsPerMinute: number;
    requestsPerDay: number;
  };
  allowedIps?: string[];
  environment?: 'production' | 'staging' | 'development';
}

interface ApiKey {
  id: string;
  key: string; // Only returned on creation
  keyHash: string;
  keyPrefix: string; // For display
  name: string;
  ownerId: string;
  scopes: string[];
  expiresAt?: Date;
  rateLimit: {
    requestsPerMinute: number;
    requestsPerDay: number;
  };
  allowedIps?: string[];
  environment: string;
  createdAt: Date;
  lastUsedAt?: Date;
  rotatedAt?: Date;
}

interface AuditEvent {
  apiKeyId: string;
  action: 'created' | 'used' | 'rotated' | 'revoked' | 'expired';
  ipAddress?: string;
  userAgent?: string;
  endpoint?: string;
  statusCode?: number;
  timestamp: Date;
  metadata?: Record<string, unknown>;
}

export class ApiKeyService {
  private readonly KEY_PREFIX = 'sk_';
  private readonly HASH_ALGORITHM = 'sha256';
  
  async createApiKey(config: ApiKeyConfig): Promise<{ key: string; apiKey: ApiKey }> {
    // Generate cryptographically secure key
    const rawKey = crypto.randomBytes(32).toString('base64url');
    const fullKey = `${this.KEY_PREFIX}${config.environment?.[0] ?? 'p'}_${rawKey}`;
    
    // Hash for storage
    const keyHash = this.hashKey(fullKey);
    const keyPrefix = fullKey.substring(0, 12) + '...';
    
    const [apiKey] = await db('api_keys').insert({
      id: crypto.randomUUID(),
      key_hash: keyHash,
      key_prefix: keyPrefix,
      name: config.name,
      owner_id: config.ownerId,
      scopes: JSON.stringify(config.scopes),
      expires_at: config.expiresAt,
      rate_limit_per_minute: config.rateLimit?.requestsPerMinute ?? 60,
      rate_limit_per_day: config.rateLimit?.requestsPerDay ?? 10000,
      allowed_ips: config.allowedIps ? JSON.stringify(config.allowedIps) : null,
      environment: config.environment ?? 'production',
      created_at: new Date(),
      is_active: true,
    }).returning('*');
    
    // Cache key hash for fast lookups
    await redis.setex(
      `api_key:${keyHash}`,
      3600,
      JSON.stringify({
        id: apiKey.id,
        ownerId: apiKey.owner_id,
        scopes: config.scopes,
        isActive: true,
      })
    );
    
    // Audit log
    await this.createAuditEvent({
      apiKeyId: apiKey.id,
      action: 'created',
      timestamp: new Date(),
      metadata: { name: config.name, scopes: config.scopes },
    });
    
    return {
      key: fullKey, // Return only once
      apiKey: this.mapApiKey(apiKey),
    };
  }
  
  async validateApiKey(
    key: string,
    requestContext: {
      ipAddress: string;
      userAgent: string;
      endpoint: string;
    }
  ): Promise<ApiKey | null> {
    const keyHash = this.hashKey(key);
    
    // Check cache first
    const cached = await redis.get(`api_key:${keyHash}`);
    
    let apiKey;
    if (cached) {
      const cachedData = JSON.parse(cached);
      if (!cachedData.isActive) return null;
      
      // Fetch full record for validation
      apiKey = await db('api_keys')
        .where({ key_hash: keyHash, is_active: true })
        .first();
    } else {
      apiKey = await db('api_keys')
        .where({ key_hash: keyHash, is_active: true })
        .first();
    }
    
    if (!apiKey) return null;
    
    // Check expiration
    if (apiKey.expires_at && new Date(apiKey.expires_at) < new Date()) {
      await this.revokeApiKey(apiKey.id, 'expired');
      return null;
    }
    
    // Check IP allowlist
    if (apiKey.allowed_ips) {
      const allowedIps = JSON.parse(apiKey.allowed_ips) as string[];
      if (allowedIps.length > 0 && !allowedIps.includes(requestContext.ipAddress)) {
        await this.createAuditEvent({
          apiKeyId: apiKey.id,
          action: 'used',
          ipAddress: requestContext.ipAddress,
          userAgent: requestContext.userAgent,
          endpoint: requestContext.endpoint,
          statusCode: 403,
          timestamp: new Date(),
          metadata: { reason: 'ip_not_allowed' },
        });
        return null;
      }
    }
    
    // Rate limiting check
    const withinLimit = await this.checkRateLimit(
      apiKey.id,
      apiKey.rate_limit_per_minute,
      apiKey.rate_limit_per_day
    );
    
    if (!withinLimit) {
      return null;
    }
    
    // Update last used timestamp asynchronously
    db('api_keys')
      .where({ id: apiKey.id })
      .update({ last_used_at: new Date() })
      .catch(err => console.error('Failed to update last_used_at:', err));
    
    // Audit log
    await this.createAuditEvent({
      apiKeyId: apiKey.id,
      action: 'used',
      ipAddress: requestContext.ipAddress,
      userAgent: requestContext.userAgent,
      endpoint: requestContext.endpoint,
      statusCode: 200,
      timestamp: new Date(),
    });
    
    return this.mapApiKey(apiKey);
  }
  
  async rotateApiKey(keyId: string): Promise<{ newKey: string; apiKey: ApiKey }> {
    const oldKey = await db('api_keys').where({ id: keyId }).first();
    
    if (!oldKey) {
      throw new Error('API key not found');
    }
    
    // Create new key with same config
    const result = await this.createApiKey({
      name: `${oldKey.name} (rotated)`,
      ownerId: oldKey.owner_id,
      scopes: JSON.parse(oldKey.scopes),
      expiresAt: oldKey.expires_at,
      rateLimit: {
        requestsPerMinute: oldKey.rate_limit_per_minute,
        requestsPerDay: oldKey.rate_limit_per_day,
      },
      allowedIps: oldKey.allowed_ips ? JSON.parse(oldKey.allowed_ips) : undefined,
      environment: oldKey.environment,
    });
    
    // Deactivate old key with grace period (30 minutes)
    const deactivateAt = new Date(Date.now() + 30 * 60 * 1000);
    await db('api_keys').where({ id: keyId }).update({
      is_active: false,
      rotated_at: new Date(),
      deactivated_at: deactivateAt,
    });
    
    // Update cache
    await redis.del(`api_key:${oldKey.key_hash}`);
    
    await this.createAuditEvent({
      apiKeyId: keyId,
      action: 'rotated',
      timestamp: new Date(),
      metadata: { newKeyId: result.apiKey.id },
    });
    
    return result;
  }
  
  async revokeApiKey(keyId: string, reason?: string): Promise<void> {
    const key = await db('api_keys').where({ id: keyId }).first();
    
    if (!key) return;
    
    await db('api_keys').where({ id: keyId }).update({
      is_active: false,
      deactivated_at: new Date(),
    });
    
    // Invalidate cache
    await redis.del(`api_key:${key.key_hash}`);
    
    await this.createAuditEvent({
      apiKeyId: keyId,
      action: 'revoked',
      timestamp: new Date(),
      metadata: { reason },
    });
  }
  
  async getAuditTrail(
    apiKeyId: string,
    options: { limit?: number; offset?: number; startDate?: Date; endDate?: Date }
  ): Promise<AuditEvent[]> {
    let query = db('api_key_audit_events')
      .where({ api_key_id: apiKeyId })
      .orderBy('timestamp', 'desc')
      .limit(options.limit ?? 100)
      .offset(options.offset ?? 0);
    
    if (options.startDate) {
      query = query.where('timestamp', '>=', options.startDate);
    }
    
    if (options.endDate) {
      query = query.where('timestamp', '<=', options.endDate);
    }
    
    return query;
  }
  
  private async checkRateLimit(
    keyId: string,
    perMinute: number,
    perDay: number
  ): Promise<boolean> {
    const now = Date.now();
    const minuteKey = `rate:${keyId}:minute:${Math.floor(now / 60000)}`;
    const dayKey = `rate:${keyId}:day:${Math.floor(now / 86400000)}`;
    
    const pipeline = redis.pipeline();
    pipeline.incr(minuteKey);
    pipeline.expire(minuteKey, 60);
    pipeline.incr(dayKey);
    pipeline.expire(dayKey, 86400);
    
    const results = await pipeline.exec();
    
    const minuteCount = results![0][1] as number;
    const dayCount = results![2][1] as number;
    
    return minuteCount <= perMinute && dayCount <= perDay;
  }
  
  private hashKey(key: string): string {
    return crypto
      .createHmac(this.HASH_ALGORITHM, process.env.API_KEY_HMAC_SECRET!)
      .update(key)
      .digest('hex');
  }
  
  private async createAuditEvent(event: AuditEvent): Promise<void> {
    await db('api_key_audit_events').insert({
      api_key_id: event.apiKeyId,
      action: event.action,
      ip_address: event.ipAddress,
      user_agent: event.userAgent,
      endpoint: event.endpoint,
      status_code: event.statusCode,
      timestamp: event.timestamp,
      metadata: event.metadata ? JSON.stringify(event.metadata) : null,
    });
  }
  
  private mapApiKey(row: any): ApiKey {
    return {
      id: row.id,
      keyHash: row.key_hash,
      keyPrefix: row.key_prefix,
      name: row.name,
      ownerId: row.owner_id,
      scopes: JSON.parse(row.scopes),
      expiresAt: row.expires_at,
      rateLimit: {
        requestsPerMinute: row.rate_limit_per_minute,
        requestsPerDay: row.rate_limit_per_day,
      },
      allowedIps: row.allowed_ips ? JSON.parse(row.allowed_ips) : undefined,
      environment: row.environment,
      createdAt: row.created_at,
      lastUsedAt: row.last_used_at,
      rotatedAt: row.rotated_at,
    };
  }
}
```

---

## 5. Input Validation with Zod Schemas

```typescript
// src/validation/schemas.ts
import { z } from 'zod';
import { Request, Response, NextFunction } from 'express';

// Custom validators
const sanitizedString = z.string()
  .transform(val => val.trim())
  .refine(val => !/<script>/i.test(val), 'XSS detected');

const safeEmail = z.string()
  .email()
  .toLowerCase()
  .max(254);

const strongPassword = z.string()
  .min(8, 'Password must be at least 8 characters')
  .max(128)
  .regex(/[A-Z]/, 'Must contain uppercase letter')
  .regex(/[a-z]/, 'Must contain lowercase letter')
  .regex(/[0-9]/, 'Must contain number')
  .regex(/[^A-Za-z0-9]/, 'Must contain special character');

const uuid = z.string().uuid();

const pagination = z.object({
  page: z.coerce.number().int().min(1).default(1),
  limit: z.coerce.number().int().min(1).max(100).default(20),
  sortBy: z.string().optional(),
  sortOrder: z.enum(['asc', 'desc']).default('desc'),
});

// Business schemas
const CreateUserSchema = z.object({
  email: safeEmail,
  password: strongPassword,
  firstName: sanitizedString.min(1).max(50),
  lastName: sanitizedString.min(1).max(50),
  role: z.enum(['user', 'admin', 'moderator']).default('user'),
  metadata: z.record(z.unknown()).optional(),
});

const UpdateUserSchema = CreateUserSchema.partial().omit({ password: true });

const CreateProductSchema = z.object({
  name: sanitizedString.min(1).max(200),
  description: sanitizedString.max(5000),
  price: z.number().positive().multipleOf(0.01),
  currency: z.enum(['USD', 'EUR', 'GBP', 'THB']),
  stock: z.number().int().min(0),
  categoryId: uuid,
  tags: z.array(sanitizedString.max(50)).max(20).default([]),
  images: z.array(z.string().url()).max(10).default([]),
  attributes: z.record(z.union([z.string(), z.number(), z.boolean()])).optional(),
});

const SearchQuerySchema = z.object({
  q: sanitizedString.max(500).optional(),
  category: z.string().optional(),
  minPrice: z.coerce.number().min(0).optional(),
  maxPrice: z.coerce.number().positive().optional(),
  inStock: z.coerce.boolean().optional(),
  ...pagination.shape,
}).refine(
  data => !data.minPrice || !data.maxPrice || data.minPrice <= data.maxPrice,
  { message: 'minPrice must be <= maxPrice', path: ['minPrice'] }
);

// Validation middleware factory
export function validate<T extends z.ZodType>(
  schema: T,
  source: 'body' | 'query' | 'params' = 'body'
) {
  return (req: Request, res: Response, next: NextFunction) => {
    try {
      const data = schema.parse(req[source]);
      req[source] = data;
      next();
    } catch (error) {
      if (error instanceof z.ZodError) {
        return res.status(400).json({
          error: 'validation_error',
          message: 'Request validation failed',
          details: error.errors.map(e => ({
            field: e.path.join('.'),
            message: e.message,
            code: e.code,
          })),
        });
      }
      next(error);
    }
  };
}

export {
  CreateUserSchema,
  UpdateUserSchema,
  CreateProductSchema,
  SearchQuerySchema,
  pagination,
};
```

---

## 6. SQL Injection Prevention

```typescript
// src/database/safe-query.ts
import knex, { Knex } from 'knex';

// Always use parameterized queries with Knex
export class UserRepository {
  constructor(private readonly db: Knex) {}
  
  // CORRECT: Parameterized query
  async findByEmail(email: string) {
    return this.db('users')
      .where({ email: email.toLowerCase() })
      .select('id', 'email', 'created_at')
      .first();
  }
  
  // CORRECT: Safe dynamic column sorting
  async findAll(options: {
    sortBy?: string;
    sortOrder?: 'asc' | 'desc';
    limit?: number;
    offset?: number;
  }) {
    const ALLOWED_SORT_COLUMNS = ['created_at', 'email', 'last_name', 'first_name'];
    const sortColumn = ALLOWED_SORT_COLUMNS.includes(options.sortBy || '')
      ? options.sortBy!
      : 'created_at';
    
    return this.db('users')
      .orderBy(sortColumn, options.sortOrder ?? 'desc')
      .limit(Math.min(options.limit ?? 20, 100))
      .offset(options.offset ?? 0);
  }
  
  // CORRECT: Safe search with LIKE
  async search(query: string) {
    // Using knex parameterized binding, not string concatenation
    return this.db('users')
      .where('email', 'ilike', `%${query.replace(/%/g, '\\%').replace(/_/g, '\\_')}%`)
      .orWhere('first_name', 'ilike', `%${query.replace(/%/g, '\\%')}%`)
      .limit(50);
  }
  
  // CORRECT: Raw query with proper binding
  async findByComplexCriteria(criteria: {
    minAge?: number;
    maxAge?: number;
    roles?: string[];
  }) {
    let query = this.db('users').select('*');
    
    if (criteria.minAge !== undefined) {
      // Use binding parameter, never interpolate directly
      query = query.whereRaw(
        'EXTRACT(YEAR FROM AGE(birth_date)) >= ?',
        [criteria.minAge]
      );
    }
    
    if (criteria.roles && criteria.roles.length > 0) {
      // Validate enum values before use
      const VALID_ROLES = ['user', 'admin', 'moderator'] as const;
      const safeRoles = criteria.roles.filter(r =>
        VALID_ROLES.includes(r as any)
      );
      query = query.whereIn('role', safeRoles);
    }
    
    return query;
  }
  
  // WRONG - Never do this:
  // async dangerousFind(name: string) {
  //   return this.db.raw(`SELECT * FROM users WHERE name = '${name}'`); // SQL INJECTION!
  // }
}
```

---

## 7. XSS Prevention in API Responses

```typescript
// src/middleware/xss-prevention.ts
import { Request, Response, NextFunction } from 'express';
import DOMPurify from 'isomorphic-dompurify';
import { JSDOM } from 'jsdom';

const window = new JSDOM('').window;
const purify = DOMPurify(window as any);

// Sanitize HTML content in responses
export function sanitizeHtmlOutput() {
  return (req: Request, res: Response, next: NextFunction) => {
    const originalJson = res.json.bind(res);
    
    res.json = function(data: any) {
      const sanitized = deepSanitize(data);
      return originalJson(sanitized);
    };
    
    next();
  };
}

function deepSanitize(obj: unknown): unknown {
  if (typeof obj === 'string') {
    return obj; // Don't sanitize - encode at template level
  }
  
  if (Array.isArray(obj)) {
    return obj.map(deepSanitize);
  }
  
  if (obj !== null && typeof obj === 'object') {
    const sanitized: Record<string, unknown> = {};
    for (const [key, value] of Object.entries(obj)) {
      sanitized[key] = deepSanitize(value);
    }
    return sanitized;
  }
  
  return obj;
}

// Sanitize HTML content that needs to be rendered
export function sanitizeHtmlContent(html: string): string {
  return purify.sanitize(html, {
    ALLOWED_TAGS: ['p', 'b', 'i', 'em', 'strong', 'a', 'ul', 'li', 'ol', 'br'],
    ALLOWED_ATTR: ['href', 'target', 'rel'],
    ALLOW_DATA_ATTR: false,
    ADD_ATTR: ['rel'], // Force rel="noopener noreferrer" on links
    FORBID_SCRIPTS: true,
    FORBID_TAGS: ['script', 'style', 'iframe', 'object', 'embed'],
    FORCE_BODY: false,
  });
}

// Security headers middleware
export function securityHeaders() {
  return (req: Request, res: Response, next: NextFunction) => {
    // Prevent XSS
    res.setHeader('X-Content-Type-Options', 'nosniff');
    res.setHeader('X-Frame-Options', 'DENY');
    res.setHeader('X-XSS-Protection', '1; mode=block');
    
    // Content Security Policy
    res.setHeader('Content-Security-Policy', [
      "default-src 'self'",
      "script-src 'self' 'nonce-{nonce}'",
      "style-src 'self' 'unsafe-inline'",
      "img-src 'self' data: https:",
      "connect-src 'self'",
      "font-src 'self'",
      "frame-ancestors 'none'",
      "form-action 'self'",
      "base-uri 'self'",
    ].join('; '));
    
    // Prevent MIME sniffing
    res.setHeader('Referrer-Policy', 'strict-origin-when-cross-origin');
    res.setHeader('Permissions-Policy', 'camera=(), microphone=(), geolocation=()');
    
    next();
  };
}
```

---

## 8. CORS Configuration for Microservices

```typescript
// src/middleware/cors.ts
import cors from 'cors';
import { Request, Response, NextFunction } from 'express';
import { Redis } from 'ioredis';

const redis = new Redis(process.env.REDIS_URL!);

interface CorsConfig {
  allowedOrigins: string[];
  allowedMethods: string[];
  allowedHeaders: string[];
  exposedHeaders: string[];
  credentials: boolean;
  maxAge: number;
}

const SERVICE_CORS_CONFIG: Record<string, CorsConfig> = {
  'api-gateway': {
    allowedOrigins: [
      'https://app.example.com',
      'https://admin.example.com',
      ...(process.env.NODE_ENV !== 'production'
        ? ['http://localhost:3000', 'http://localhost:3001']
        : []),
    ],
    allowedMethods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS'],
    allowedHeaders: [
      'Authorization',
      'Content-Type',
      'X-Request-ID',
      'X-API-Key',
      'X-Idempotency-Key',
    ],
    exposedHeaders: [
      'X-Request-ID',
      'X-Rate-Limit-Limit',
      'X-Rate-Limit-Remaining',
      'X-Rate-Limit-Reset',
    ],
    credentials: true,
    maxAge: 86400, // 24 hours preflight cache
  },
  'internal-service': {
    // Internal services only accept requests from gateway
    allowedOrigins: [
      process.env.GATEWAY_INTERNAL_URL || 'http://api-gateway:3000',
    ],
    allowedMethods: ['GET', 'POST', 'PUT', 'DELETE'],
    allowedHeaders: [
      'Authorization',
      'Content-Type',
      'X-Request-ID',
      'X-Service-Token',
    ],
    exposedHeaders: ['X-Request-ID'],
    credentials: false,
    maxAge: 3600,
  },
};

export function configureCors(serviceType: keyof typeof SERVICE_CORS_CONFIG = 'api-gateway') {
  const config = SERVICE_CORS_CONFIG[serviceType];
  
  return cors({
    origin: (origin, callback) => {
      // Allow requests with no origin (server-to-server, mobile apps)
      if (!origin) {
        return callback(null, true);
      }
      
      if (config.allowedOrigins.includes(origin)) {
        callback(null, true);
      } else {
        callback(new Error(`CORS policy: origin ${origin} not allowed`));
      }
    },
    methods: config.allowedMethods,
    allowedHeaders: config.allowedHeaders,
    exposedHeaders: config.exposedHeaders,
    credentials: config.credentials,
    maxAge: config.maxAge,
    preflightContinue: false,
    optionsSuccessStatus: 204,
  });
}

// Dynamic CORS for multi-tenant applications
export function dynamicCors() {
  return async (req: Request, res: Response, next: NextFunction) => {
    const origin = req.headers.origin;
    
    if (!origin) {
      return next();
    }
    
    try {
      // Check dynamic allowed origins from database/cache
      const tenantId = extractTenantFromRequest(req);
      const cacheKey = `cors:${tenantId}`;
      
      let allowedOrigins = await redis.smembers(cacheKey);
      
      if (allowedOrigins.length === 0) {
        // Fetch from database and cache
        const tenant = await getTenantAllowedOrigins(tenantId);
        if (tenant.allowedOrigins.length > 0) {
          await redis.sadd(cacheKey, ...tenant.allowedOrigins);
          await redis.expire(cacheKey, 300); // 5 minute cache
          allowedOrigins = tenant.allowedOrigins;
        }
      }
      
      if (allowedOrigins.includes(origin)) {
        res.setHeader('Access-Control-Allow-Origin', origin);
        res.setHeader('Access-Control-Allow-Credentials', 'true');
        res.setHeader('Vary', 'Origin');
        
        if (req.method === 'OPTIONS') {
          res.setHeader(
            'Access-Control-Allow-Methods',
            'GET, POST, PUT, PATCH, DELETE'
          );
          res.setHeader(
            'Access-Control-Allow-Headers',
            'Authorization, Content-Type, X-Request-ID'
          );
          res.setHeader('Access-Control-Max-Age', '86400');
          return res.status(204).end();
        }
      }
    } catch (error) {
      console.error('CORS check failed:', error);
    }
    
    next();
  };
}

function extractTenantFromRequest(req: Request): string {
  return req.headers['x-tenant-id'] as string || 'default';
}

async function getTenantAllowedOrigins(tenantId: string): Promise<{ allowedOrigins: string[] }> {
  // Implementation would fetch from database
  return { allowedOrigins: [] };
}
```

---

## 9. Database Migration สำหรับ Security Tables

```sql
-- migrations/001_security_tables.sql

-- OAuth Clients
CREATE TABLE oauth_clients (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  client_id VARCHAR(100) UNIQUE NOT NULL,
  client_secret_hash VARCHAR(64), -- NULL for public clients
  name VARCHAR(200) NOT NULL,
  client_type VARCHAR(20) NOT NULL CHECK (client_type IN ('public', 'confidential')),
  redirect_uris JSONB NOT NULL DEFAULT '[]',
  allowed_scopes JSONB NOT NULL DEFAULT '[]',
  allowed_grant_types JSONB NOT NULL DEFAULT '["authorization_code"]',
  can_introspect BOOLEAN NOT NULL DEFAULT FALSE,
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- API Keys
CREATE TABLE api_keys (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  key_hash VARCHAR(64) UNIQUE NOT NULL,
  key_prefix VARCHAR(20) NOT NULL,
  name VARCHAR(200) NOT NULL,
  owner_id UUID NOT NULL,
  scopes JSONB NOT NULL DEFAULT '[]',
  expires_at TIMESTAMPTZ,
  rate_limit_per_minute INTEGER NOT NULL DEFAULT 60,
  rate_limit_per_day INTEGER NOT NULL DEFAULT 10000,
  allowed_ips JSONB,
  environment VARCHAR(20) NOT NULL DEFAULT 'production',
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  last_used_at TIMESTAMPTZ,
  rotated_at TIMESTAMPTZ,
  deactivated_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_api_keys_owner ON api_keys (owner_id);
CREATE INDEX idx_api_keys_active ON api_keys (is_active, expires_at);

-- API Key Audit Events
CREATE TABLE api_key_audit_events (
  id BIGSERIAL PRIMARY KEY,
  api_key_id UUID NOT NULL REFERENCES api_keys(id),
  action VARCHAR(50) NOT NULL,
  ip_address INET,
  user_agent TEXT,
  endpoint TEXT,
  status_code SMALLINT,
  timestamp TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  metadata JSONB
);

CREATE INDEX idx_audit_api_key ON api_key_audit_events (api_key_id, timestamp DESC);
CREATE INDEX idx_audit_action ON api_key_audit_events (action, timestamp DESC);

-- Partition by month for performance
CREATE TABLE api_key_audit_events_2024_01 PARTITION OF api_key_audit_events
  FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
```

---

## 10. Integration Example

```typescript
// src/app.ts
import express from 'express';
import helmet from 'helmet';
import { authRouter } from './auth/pkce-auth-server';
import { introspectionRouter } from './auth/token-introspection';
import { authenticate, requireScopes } from './middleware/scope-authorization';
import { validate, CreateProductSchema, SearchQuerySchema } from './validation/schemas';
import { configureCors } from './middleware/cors';
import { securityHeaders } from './middleware/xss-prevention';
import { ApiKeyService } from './services/api-key.service';

const app = express();

// Security middleware
app.use(helmet());
app.use(configureCors());
app.use(securityHeaders());
app.use(express.json({ limit: '10mb' }));

// Rate limiting
import rateLimit from 'express-rate-limit';
app.use('/api/', rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 100,
  standardHeaders: true,
  legacyHeaders: false,
}));

// Auth routes
app.use('/oauth', authRouter);
app.use('/oauth', introspectionRouter);

// API key middleware
const apiKeyService = new ApiKeyService();

async function apiKeyAuth(req: express.Request, res: express.Response, next: express.NextFunction) {
  const apiKey = req.headers['x-api-key'] as string;
  
  if (!apiKey) {
    return next(); // Fall through to JWT auth
  }
  
  const key = await apiKeyService.validateApiKey(apiKey, {
    ipAddress: req.ip || '0.0.0.0',
    userAgent: req.headers['user-agent'] || '',
    endpoint: req.path,
  });
  
  if (!key) {
    return res.status(401).json({ error: 'invalid_api_key' });
  }
  
  req.user = {
    id: key.ownerId,
    scopes: key.scopes,
    clientId: `api_key:${key.id}`,
  };
  
  next();
}

// Protected routes
app.get('/api/products',
  apiKeyAuth,
  validate(SearchQuerySchema, 'query'),
  requireScopes('products:read'),
  async (req, res) => {
    const { page, limit, q } = req.query;
    // Handler implementation
    res.json({ products: [], total: 0, page, limit });
  }
);

app.post('/api/products',
  authenticate(),
  requireScopes('products:write'),
  validate(CreateProductSchema),
  async (req, res) => {
    // Create product
    res.status(201).json({ product: req.body });
  }
);

export { app };
```

---

## สรุป

ในบทนี้เราได้เรียนรู้การรักษาความปลอดภัย API ในระดับ Production ครอบคลุม:

1. **OAuth 2.0 PKCE Flow** — ป้องกัน authorization code interception ด้วย code_challenge/code_verifier

2. **Token Introspection** — ให้ Resource Server ตรวจสอบ token validity โดยไม่ต้องแชร์ secret

3. **Scope-based Authorization** — middleware ที่ enforce permission granularity ระดับ operation

4. **API Key Management** — ระบบ complete ครอบคลุม creation, validation, rotation, revocation, rate limiting, IP allowlisting, และ audit trail

5. **Input Validation** — Zod schemas แบบ composable ที่ป้องกัน bad input ตั้งแต่ edge

6. **SQL Injection Prevention** — parameterized queries, whitelist-based sorting, safe LIKE patterns

7. **XSS Prevention** — security headers, Content-Security-Policy, HTML sanitization

8. **CORS Configuration** — per-service CORS config และ dynamic multi-tenant CORS

Key takeaways:
- ใช้ PKCE สำหรับทุก public client ไม่ว่าจะเป็น SPA หรือ Mobile App
- เก็บ API key เป็น hash เท่านั้น ไม่เคยเก็บ plaintext
- Validate input ที่ boundary ของ service เสมอ ก่อนที่จะ process
- Security headers เป็น defense-in-depth ที่ต้องมีทุก service

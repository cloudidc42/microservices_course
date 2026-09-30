# Part 12: Authentication & Authorization ใน Microservices

## บทนำ

Authentication (การยืนยันตัวตน) และ Authorization (การอนุญาต) คือ cross-cutting concerns ที่ต้องออกแบบอย่างรอบคอบใน Microservices เพราะทุก service ต้องรู้ว่าใครกำลัง request มา

---

## 12.1 Authentication Strategies ใน Microservices

### Option 1: Centralized Auth Service (Recommended)

```
Client → API Gateway → Auth Service (ตรวจสอบ token)
                     ↘ User Service
                       Product Service   (รับ user context จาก Gateway)
                       Order Service
```

**ข้อดี:** ตรวจสอบ token ที่เดียว, ง่ายต่อการ revoke
**ข้อเสีย:** Auth Service เป็น single point of failure

### Option 2: Distributed JWT Verification

```
Client → API Gateway → ตรวจสอบ JWT signature เอง (ใช้ public key)
                     ↘ Services แต่ละตัวก็ตรวจ JWT เองได้
```

**ข้อดี:** ไม่มี single point of failure, ไม่ต้องเรียก auth service ทุกครั้ง
**ข้อเสีย:** ยาก token revocation, ทุก service ต้องมี validation logic

### Option 3: Service Mesh with mTLS

```
Services communicate with mutual TLS certificates
Identity verified at infrastructure level
```

---

## 12.2 JWT Deep Dive

### JWT Structure

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.          ← Header (base64)
eyJzdWIiOiJ1c2VyLTEyMyIsInJvbGUiOiJ1c2VyIn0.   ← Payload (base64)
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c    ← Signature
```

### JWT Payload Design

```javascript
// Good JWT payload (lean and secure)
{
  "sub": "user-123",           // Subject (user ID)
  "iat": 1704067200,           // Issued At
  "exp": 1704153600,           // Expiration (24 hours)
  "jti": "jwt-unique-id-456",  // JWT ID (for revocation)
  "iss": "auth.example.com",   // Issuer
  "aud": "api.example.com",    // Audience
  "role": "user",              // User role
  "email": "user@example.com", // Email (commonly needed)
  "firstName": "John"          // Display name
}

// ❌ Don't include sensitive data
{
  "password": "...",     // Never!
  "creditCard": "...",   // Never!
  "ssn": "...",          // Never!
  "fullProfile": {...}   // Too large - keep JWT small
}
```

### RS256 vs HS256

```javascript
// HS256: Symmetric (shared secret)
// + Simple, fast
// - Same secret used to sign AND verify
// - All services need the secret → security risk

// RS256: Asymmetric (public/private key pair)
// + Private key only in Auth Service
// + Public key shared freely with all services
// + Services verify without being able to create tokens
// - Slightly slower

// Generate RSA key pair
const { generateKeyPairSync } = require('crypto');

const { privateKey, publicKey } = generateKeyPairSync('rsa', {
  modulusLength: 2048,
  publicKeyEncoding: { type: 'spki', format: 'pem' },
  privateKeyEncoding: { type: 'pkcs8', format: 'pem' },
});
```

---

## 12.3 Auth Service Implementation

### Project Structure

```
auth-service/
├── src/
│   ├── config/
│   │   └── index.js
│   ├── controllers/
│   │   └── authController.js
│   ├── services/
│   │   ├── authService.js
│   │   └── tokenService.js
│   ├── models/
│   │   └── RefreshToken.js
│   ├── middleware/
│   │   └── validate.js
│   ├── validators/
│   │   └── auth.validators.js
│   ├── routes/
│   │   └── auth.routes.js
│   └── app.js
├── keys/
│   ├── private.pem
│   └── public.pem
└── package.json
```

### src/services/tokenService.js

```javascript
const jwt = require('jsonwebtoken');
const crypto = require('crypto');
const fs = require('fs');
const path = require('path');
const Redis = require('ioredis');
const config = require('../config');
const logger = require('../utils/logger');

class TokenService {
  constructor() {
    // Load RSA keys
    this.privateKey = fs.readFileSync(
      process.env.JWT_PRIVATE_KEY_PATH || path.join(__dirname, '../../keys/private.pem')
    );
    this.publicKey = fs.readFileSync(
      process.env.JWT_PUBLIC_KEY_PATH || path.join(__dirname, '../../keys/public.pem')
    );

    // Redis for token blacklist and refresh token storage
    this.redis = new Redis({
      host: config.redis.host,
      port: config.redis.port,
      password: config.redis.password || undefined,
    });
  }

  generateAccessToken(user) {
    const payload = {
      sub: user.id,
      email: user.email,
      role: user.role,
      firstName: user.firstName,
      jti: crypto.randomUUID(),
    };

    return jwt.sign(payload, this.privateKey, {
      algorithm: 'RS256',
      expiresIn: config.jwt.accessTokenExpiry || '15m',
      issuer: config.jwt.issuer,
      audience: config.jwt.audience,
    });
  }

  generateRefreshToken() {
    return crypto.randomBytes(64).toString('hex');
  }

  async saveRefreshToken(userId, refreshToken, deviceInfo = {}) {
    const tokenHash = crypto
      .createHash('sha256')
      .update(refreshToken)
      .digest('hex');

    const key = `refresh:${tokenHash}`;
    const data = {
      userId,
      deviceInfo,
      createdAt: new Date().toISOString(),
    };

    // Store with TTL (7 days)
    await this.redis.setex(key, 7 * 24 * 60 * 60, JSON.stringify(data));

    return tokenHash;
  }

  async verifyRefreshToken(refreshToken) {
    const tokenHash = crypto
      .createHash('sha256')
      .update(refreshToken)
      .digest('hex');

    const key = `refresh:${tokenHash}`;
    const data = await this.redis.get(key);

    if (!data) {
      return null;
    }

    return { ...JSON.parse(data), tokenHash };
  }

  async revokeRefreshToken(refreshToken) {
    const tokenHash = crypto
      .createHash('sha256')
      .update(refreshToken)
      .digest('hex');

    await this.redis.del(`refresh:${tokenHash}`);
  }

  async revokeAllUserTokens(userId) {
    // Get all refresh tokens for user
    const pattern = `refresh:*`;
    const keys = await this.redis.keys(pattern);

    const pipeline = this.redis.pipeline();
    for (const key of keys) {
      const data = await this.redis.get(key);
      if (data) {
        const parsed = JSON.parse(data);
        if (parsed.userId === userId) {
          pipeline.del(key);
        }
      }
    }

    await pipeline.exec();
    logger.info('Revoked all tokens for user', { userId });
  }

  async blacklistAccessToken(jti, expiresAt) {
    const ttl = Math.floor((new Date(expiresAt * 1000) - Date.now()) / 1000);
    if (ttl > 0) {
      await this.redis.setex(`blacklist:${jti}`, ttl, '1');
    }
  }

  async isTokenBlacklisted(jti) {
    return this.redis.exists(`blacklist:${jti}`);
  }

  verifyAccessToken(token) {
    return jwt.verify(token, this.publicKey, {
      algorithms: ['RS256'],
      issuer: config.jwt.issuer,
      audience: config.jwt.audience,
    });
  }

  getPublicKey() {
    return this.publicKey.toString();
  }

  decodeToken(token) {
    return jwt.decode(token);
  }
}

module.exports = new TokenService();
```

### src/services/authService.js

```javascript
const bcrypt = require('bcryptjs');
const tokenService = require('./tokenService');
const logger = require('../utils/logger');

class AuthService {
  constructor({ userRepository, emailService }) {
    this.userRepository = userRepository;
    this.emailService = emailService;
  }

  async register({ email, password, firstName, lastName, phone }) {
    // Check email exists
    const existing = await this.userRepository.findByEmail(email);
    if (existing) {
      const error = new Error('Email already registered');
      error.code = 'EMAIL_EXISTS';
      throw error;
    }

    // Hash password
    const passwordHash = await bcrypt.hash(password, 12);

    // Create user
    const user = await this.userRepository.create({
      email,
      passwordHash,
      firstName,
      lastName,
      phone,
      role: 'user',
      status: 'pending_verification',
    });

    // Send verification email
    const verificationToken = tokenService.generateRefreshToken();
    await tokenService.saveRefreshToken(user.id, verificationToken, { type: 'email_verification' });
    await this.emailService.sendVerification(user.email, verificationToken);

    logger.info('User registered', { userId: user.id, email: user.email });

    return { user: this.sanitizeUser(user) };
  }

  async login({ email, password, deviceInfo }) {
    // Find user
    const user = await this.userRepository.findByEmail(email);
    if (!user) {
      throw Object.assign(new Error('Invalid credentials'), { code: 'INVALID_CREDENTIALS' });
    }

    // Check status
    if (user.status === 'suspended') {
      throw Object.assign(new Error('Account suspended'), { code: 'ACCOUNT_SUSPENDED' });
    }

    if (user.status === 'inactive') {
      throw Object.assign(new Error('Account inactive'), { code: 'ACCOUNT_INACTIVE' });
    }

    // Verify password
    const isValid = await bcrypt.compare(password, user.passwordHash);
    if (!isValid) {
      // Track failed attempts
      await this.userRepository.incrementLoginAttempts(user.id);
      throw Object.assign(new Error('Invalid credentials'), { code: 'INVALID_CREDENTIALS' });
    }

    // Generate tokens
    const accessToken = tokenService.generateAccessToken(user);
    const refreshToken = tokenService.generateRefreshToken();
    await tokenService.saveRefreshToken(user.id, refreshToken, deviceInfo);

    // Update last login
    await this.userRepository.updateLastLogin(user.id);

    logger.info('User logged in', { userId: user.id, email: user.email });

    return {
      accessToken,
      refreshToken,
      expiresIn: 900, // 15 minutes
      user: this.sanitizeUser(user),
    };
  }

  async refreshTokens(refreshToken) {
    const tokenData = await tokenService.verifyRefreshToken(refreshToken);
    if (!tokenData) {
      throw Object.assign(new Error('Invalid refresh token'), { code: 'INVALID_REFRESH_TOKEN' });
    }

    const user = await this.userRepository.findById(tokenData.userId);
    if (!user || user.status !== 'active') {
      await tokenService.revokeRefreshToken(refreshToken);
      throw Object.assign(new Error('User not found or inactive'), { code: 'UNAUTHORIZED' });
    }

    // Rotate: revoke old, issue new
    await tokenService.revokeRefreshToken(refreshToken);
    const newRefreshToken = tokenService.generateRefreshToken();
    await tokenService.saveRefreshToken(user.id, newRefreshToken, tokenData.deviceInfo);

    const newAccessToken = tokenService.generateAccessToken(user);

    logger.info('Tokens refreshed', { userId: user.id });

    return {
      accessToken: newAccessToken,
      refreshToken: newRefreshToken,
      expiresIn: 900,
    };
  }

  async logout(accessToken, refreshToken) {
    try {
      // Blacklist access token
      const decoded = tokenService.decodeToken(accessToken);
      if (decoded?.jti && decoded?.exp) {
        await tokenService.blacklistAccessToken(decoded.jti, decoded.exp);
      }
    } catch {
      // Ignore decode errors during logout
    }

    if (refreshToken) {
      await tokenService.revokeRefreshToken(refreshToken);
    }

    logger.info('User logged out');
  }

  async logoutAll(userId, currentAccessToken) {
    // Revoke all refresh tokens
    await tokenService.revokeAllUserTokens(userId);

    // Blacklist current access token
    try {
      const decoded = tokenService.decodeToken(currentAccessToken);
      if (decoded?.jti && decoded?.exp) {
        await tokenService.blacklistAccessToken(decoded.jti, decoded.exp);
      }
    } catch {}

    logger.info('User logged out from all devices', { userId });
  }

  async forgotPassword(email) {
    const user = await this.userRepository.findByEmail(email);

    // Always return success to prevent email enumeration
    if (!user) {
      logger.warn('Password reset requested for non-existent email', { email });
      return;
    }

    const resetToken = tokenService.generateRefreshToken();
    const expiresAt = new Date(Date.now() + 60 * 60 * 1000); // 1 hour

    await this.userRepository.createPasswordResetToken(user.id, resetToken, expiresAt);
    await this.emailService.sendPasswordReset(user.email, resetToken);

    logger.info('Password reset requested', { userId: user.id });
  }

  async resetPassword(token, newPassword) {
    const resetData = await this.userRepository.findPasswordResetToken(token);
    if (!resetData || resetData.expiresAt < new Date()) {
      throw Object.assign(new Error('Invalid or expired reset token'), { code: 'INVALID_RESET_TOKEN' });
    }

    const passwordHash = await bcrypt.hash(newPassword, 12);
    await this.userRepository.updatePassword(resetData.userId, passwordHash);
    await this.userRepository.markResetTokenUsed(resetData.id);

    // Revoke all existing sessions
    await tokenService.revokeAllUserTokens(resetData.userId);

    logger.info('Password reset completed', { userId: resetData.userId });
  }

  async verifyEmail(token) {
    const tokenData = await tokenService.verifyRefreshToken(token);
    if (!tokenData || tokenData.deviceInfo?.type !== 'email_verification') {
      throw Object.assign(new Error('Invalid verification token'), { code: 'INVALID_TOKEN' });
    }

    await this.userRepository.markEmailVerified(tokenData.userId);
    await tokenService.revokeRefreshToken(token);

    logger.info('Email verified', { userId: tokenData.userId });
  }

  sanitizeUser(user) {
    const { passwordHash, ...safe } = user;
    return safe;
  }
}

module.exports = AuthService;
```

### src/controllers/authController.js

```javascript
const AuthService = require('../services/authService');
const tokenService = require('../services/tokenService');

// Initialize with dependencies
const authService = new AuthService({
  userRepository: require('../repositories/userRepository'),
  emailService: require('../services/emailService'),
});

module.exports = {
  async register(req, res) {
    const result = await authService.register(req.body);

    res.status(201).json({
      success: true,
      message: 'Registration successful. Please verify your email.',
      data: result,
    });
  },

  async login(req, res) {
    const deviceInfo = {
      userAgent: req.headers['user-agent'],
      ip: req.ip,
      type: 'web',
    };

    const result = await authService.login({
      ...req.body,
      deviceInfo,
    });

    // Set refresh token in HTTP-only cookie
    res.cookie('refreshToken', result.refreshToken, {
      httpOnly: true,
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'strict',
      maxAge: 7 * 24 * 60 * 60 * 1000, // 7 days
    });

    res.json({
      success: true,
      data: {
        accessToken: result.accessToken,
        expiresIn: result.expiresIn,
        user: result.user,
      },
    });
  },

  async refresh(req, res) {
    // Get refresh token from cookie or body
    const refreshToken = req.cookies?.refreshToken || req.body?.refreshToken;
    if (!refreshToken) {
      return res.status(401).json({
        success: false,
        error: { code: 'NO_REFRESH_TOKEN', message: 'Refresh token required' },
      });
    }

    const result = await authService.refreshTokens(refreshToken);

    // Rotate cookie
    res.cookie('refreshToken', result.refreshToken, {
      httpOnly: true,
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'strict',
      maxAge: 7 * 24 * 60 * 60 * 1000,
    });

    res.json({
      success: true,
      data: {
        accessToken: result.accessToken,
        expiresIn: result.expiresIn,
      },
    });
  },

  async logout(req, res) {
    const refreshToken = req.cookies?.refreshToken || req.body?.refreshToken;
    const accessToken = req.headers.authorization?.slice(7);

    await authService.logout(accessToken, refreshToken);

    res.clearCookie('refreshToken');

    res.json({
      success: true,
      message: 'Logged out successfully',
    });
  },

  async logoutAll(req, res) {
    const accessToken = req.headers.authorization?.slice(7);
    await authService.logoutAll(req.user.id, accessToken);

    res.clearCookie('refreshToken');

    res.json({
      success: true,
      message: 'Logged out from all devices',
    });
  },

  async forgotPassword(req, res) {
    await authService.forgotPassword(req.body.email);

    // Always return 200 to prevent email enumeration
    res.json({
      success: true,
      message: 'If the email exists, a password reset link has been sent',
    });
  },

  async resetPassword(req, res) {
    await authService.resetPassword(req.body.token, req.body.password);

    res.json({
      success: true,
      message: 'Password reset successfully. Please login with your new password.',
    });
  },

  async verifyEmail(req, res) {
    await authService.verifyEmail(req.query.token);

    res.json({
      success: true,
      message: 'Email verified successfully',
    });
  },

  // Endpoint for other services to get public key
  async getPublicKey(req, res) {
    res.json({
      success: true,
      data: {
        publicKey: tokenService.getPublicKey(),
        algorithm: 'RS256',
      },
    });
  },

  // Endpoint for other services to verify a token
  async introspect(req, res) {
    const { token } = req.body;
    if (!token) {
      return res.status(400).json({
        success: false,
        error: { code: 'TOKEN_REQUIRED', message: 'Token is required' },
      });
    }

    try {
      const decoded = tokenService.verifyAccessToken(token);
      const blacklisted = await tokenService.isTokenBlacklisted(decoded.jti);

      if (blacklisted) {
        return res.json({
          success: true,
          data: { active: false, reason: 'revoked' },
        });
      }

      res.json({
        success: true,
        data: {
          active: true,
          userId: decoded.sub,
          email: decoded.email,
          role: decoded.role,
          exp: decoded.exp,
          iat: decoded.iat,
        },
      });
    } catch (error) {
      res.json({
        success: true,
        data: { active: false, reason: error.message },
      });
    }
  },
};
```

### src/routes/auth.routes.js

```javascript
const express = require('express');
const router = express.Router();
const authController = require('../controllers/authController');
const validate = require('../middleware/validate');
const {
  registerSchema,
  loginSchema,
  forgotPasswordSchema,
  resetPasswordSchema,
} = require('../validators/auth.validators');

// Public auth routes
router.post('/register', validate(registerSchema), authController.register);
router.post('/login', validate(loginSchema), authController.login);
router.post('/refresh', authController.refresh);
router.post('/forgot-password', validate(forgotPasswordSchema), authController.forgotPassword);
router.post('/reset-password', validate(resetPasswordSchema), authController.resetPassword);
router.get('/verify-email', authController.verifyEmail);

// Protected routes
router.post('/logout', authController.logout);
router.post('/logout-all', authController.logoutAll);

// Service-to-service endpoints
router.get('/public-key', authController.getPublicKey);
router.post('/introspect', authController.introspect);

module.exports = router;
```

---

## 12.4 Distributed Token Verification

### JWT Verification Middleware (For Each Service)

```javascript
// shared/src/middleware/jwtVerify.js
// This middleware is used by ALL services to verify JWT tokens

const jwt = require('jsonwebtoken');
const axios = require('axios');
const NodeCache = require('node-cache');

// Cache the public key (fetch from auth service once)
const keyCache = new NodeCache({ stdTTL: 3600 }); // 1 hour cache

class JWTVerifier {
  constructor({ authServiceUrl, issuer, audience }) {
    this.authServiceUrl = authServiceUrl;
    this.issuer = issuer;
    this.audience = audience;
    this._publicKey = null;
  }

  async getPublicKey() {
    const cached = keyCache.get('publicKey');
    if (cached) return cached;

    try {
      const response = await axios.get(`${this.authServiceUrl}/auth/public-key`, {
        timeout: 5000,
      });
      const publicKey = response.data.data.publicKey;
      keyCache.set('publicKey', publicKey);
      return publicKey;
    } catch (error) {
      throw new Error(`Failed to fetch public key: ${error.message}`);
    }
  }

  async verify(token) {
    const publicKey = await this.getPublicKey();

    return jwt.verify(token, publicKey, {
      algorithms: ['RS256'],
      issuer: this.issuer,
      audience: this.audience,
    });
  }

  middleware() {
    return async (req, res, next) => {
      const authHeader = req.headers.authorization;
      if (!authHeader?.startsWith('Bearer ')) {
        return res.status(401).json({
          success: false,
          error: { code: 'UNAUTHORIZED', message: 'Bearer token required' },
        });
      }

      const token = authHeader.slice(7);

      try {
        const decoded = await this.verify(token);
        req.user = {
          id: decoded.sub,
          email: decoded.email,
          role: decoded.role,
          firstName: decoded.firstName,
          jti: decoded.jti,
        };
        next();
      } catch (error) {
        const code = error.name === 'TokenExpiredError' ? 'TOKEN_EXPIRED' : 'INVALID_TOKEN';
        return res.status(401).json({
          success: false,
          error: { code, message: error.message },
        });
      }
    };
  }

  optionalMiddleware() {
    return async (req, res, next) => {
      const authHeader = req.headers.authorization;
      if (!authHeader?.startsWith('Bearer ')) {
        return next(); // No token is OK
      }

      const token = authHeader.slice(7);
      try {
        const decoded = await this.verify(token);
        req.user = {
          id: decoded.sub,
          email: decoded.email,
          role: decoded.role,
        };
      } catch {
        // Ignore verification errors for optional auth
      }
      next();
    };
  }
}

module.exports = JWTVerifier;
```

### Using JWTVerifier in a Service

```javascript
// product-service/src/middleware/auth.js
const JWTVerifier = require('shared/middleware/jwtVerify');
const config = require('../config');

const verifier = new JWTVerifier({
  authServiceUrl: config.services.auth,
  issuer: 'auth.example.com',
  audience: 'api.example.com',
});

module.exports = {
  authenticate: verifier.middleware(),
  optionalAuth: verifier.optionalMiddleware(),
  authorize: (...roles) => (req, res, next) => {
    if (!req.user) {
      return res.status(401).json({
        success: false,
        error: { code: 'UNAUTHORIZED', message: 'Authentication required' },
      });
    }
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({
        success: false,
        error: { code: 'FORBIDDEN', message: 'Insufficient permissions' },
      });
    }
    next();
  },
};
```

---

## 12.5 Role-Based Access Control (RBAC)

### RBAC Design

```javascript
// shared/src/rbac/permissions.js

const PERMISSIONS = {
  // User permissions
  'users:read:own': 'Read own user profile',
  'users:update:own': 'Update own user profile',
  'users:read:any': 'Read any user profile',
  'users:update:any': 'Update any user profile',
  'users:delete:any': 'Delete any user',

  // Product permissions
  'products:read': 'Read products (public)',
  'products:create': 'Create products',
  'products:update:own': 'Update own products',
  'products:update:any': 'Update any product',
  'products:delete:own': 'Delete own products',
  'products:delete:any': 'Delete any product',

  // Order permissions
  'orders:read:own': 'Read own orders',
  'orders:create': 'Create orders',
  'orders:cancel:own': 'Cancel own orders',
  'orders:read:any': 'Read any order',
  'orders:update:any': 'Update any order',

  // Admin permissions
  'admin:users:manage': 'Manage all users',
  'admin:reports:view': 'View admin reports',
};

const ROLES = {
  guest: [],

  user: [
    'users:read:own',
    'users:update:own',
    'products:read',
    'orders:read:own',
    'orders:create',
    'orders:cancel:own',
  ],

  seller: [
    'users:read:own',
    'users:update:own',
    'products:read',
    'products:create',
    'products:update:own',
    'products:delete:own',
    'orders:read:own',
    'orders:create',
  ],

  admin: Object.keys(PERMISSIONS), // Admin has all permissions
};

class RBAC {
  constructor(roles = ROLES) {
    this.roles = roles;
  }

  can(role, permission) {
    const rolePermissions = this.roles[role] || [];
    return rolePermissions.includes(permission);
  }

  checkPermission(permission) {
    return (req, res, next) => {
      if (!req.user) {
        return res.status(401).json({
          success: false,
          error: { code: 'UNAUTHORIZED', message: 'Authentication required' },
        });
      }

      if (!this.can(req.user.role, permission)) {
        return res.status(403).json({
          success: false,
          error: {
            code: 'FORBIDDEN',
            message: `Permission '${permission}' required`,
          },
        });
      }

      next();
    };
  }

  // Check if user can access a resource (own vs any)
  checkResourceAccess(resourceOwnerId) {
    return (req, res, next) => {
      if (!req.user) {
        return res.status(401).json({
          success: false,
          error: { code: 'UNAUTHORIZED', message: 'Authentication required' },
        });
      }

      const isOwner = req.user.id === resourceOwnerId;
      const isAdmin = req.user.role === 'admin';

      if (!isOwner && !isAdmin) {
        return res.status(403).json({
          success: false,
          error: { code: 'FORBIDDEN', message: 'Access denied' },
        });
      }

      req.isOwner = isOwner;
      next();
    };
  }
}

module.exports = { RBAC, ROLES, PERMISSIONS, rbac: new RBAC() };
```

### Using RBAC in Routes

```javascript
// order-service/src/routes/orders.routes.js
const express = require('express');
const router = express.Router();
const { authenticate } = require('../middleware/auth');
const { rbac } = require('shared/rbac/permissions');
const orderController = require('../controllers/orderController');

// Create order - requires 'orders:create' permission
router.post('/',
  authenticate,
  rbac.checkPermission('orders:create'),
  orderController.create
);

// Get user's own orders
router.get('/',
  authenticate,
  rbac.checkPermission('orders:read:own'),
  orderController.getMyOrders
);

// Get specific order - controller will check ownership
router.get('/:id',
  authenticate,
  orderController.getById  // Controller checks if user owns the order or is admin
);

// Cancel order
router.post('/:id/cancel',
  authenticate,
  rbac.checkPermission('orders:cancel:own'),
  orderController.cancel
);

// Admin: view all orders
router.get('/admin/all',
  authenticate,
  rbac.checkPermission('orders:read:any'),
  orderController.getAllOrders
);

module.exports = router;
```

---

## 12.6 API Key Authentication (for Service-to-Service)

```javascript
// shared/src/middleware/apiKey.js
const crypto = require('crypto');
const Redis = require('ioredis');
const config = require('../config');

const redis = new Redis(config.redis);

class ApiKeyAuth {
  // Generate a new API key
  static generateApiKey() {
    const prefix = 'sk_live_'; // or 'sk_test_' for test keys
    const randomBytes = crypto.randomBytes(32).toString('hex');
    return `${prefix}${randomBytes}`;
  }

  static hashApiKey(apiKey) {
    return crypto.createHash('sha256').update(apiKey).digest('hex');
  }

  // Save API key to Redis (or database)
  static async saveApiKey(serviceId, apiKey, permissions = []) {
    const keyHash = this.hashApiKey(apiKey);
    const data = {
      serviceId,
      permissions,
      createdAt: new Date().toISOString(),
      lastUsedAt: null,
    };

    await redis.set(`apikey:${keyHash}`, JSON.stringify(data));
    return keyHash;
  }

  // Verify API key middleware
  static middleware(requiredPermissions = []) {
    return async (req, res, next) => {
      const apiKey = req.headers['x-api-key'];
      if (!apiKey) {
        return res.status(401).json({
          success: false,
          error: { code: 'API_KEY_REQUIRED', message: 'API key is required' },
        });
      }

      const keyHash = ApiKeyAuth.hashApiKey(apiKey);
      const data = await redis.get(`apikey:${keyHash}`);

      if (!data) {
        return res.status(401).json({
          success: false,
          error: { code: 'INVALID_API_KEY', message: 'Invalid API key' },
        });
      }

      const keyData = JSON.parse(data);

      // Check permissions
      if (requiredPermissions.length > 0) {
        const hasPermissions = requiredPermissions.every(p =>
          keyData.permissions.includes(p)
        );
        if (!hasPermissions) {
          return res.status(403).json({
            success: false,
            error: { code: 'INSUFFICIENT_PERMISSIONS', message: 'API key lacks required permissions' },
          });
        }
      }

      // Update last used (fire and forget)
      redis.set(`apikey:${keyHash}`, JSON.stringify({
        ...keyData,
        lastUsedAt: new Date().toISOString(),
      }));

      req.service = { id: keyData.serviceId, permissions: keyData.permissions };
      next();
    };
  }
}

module.exports = ApiKeyAuth;
```

---

## 12.7 OAuth 2.0 / Social Login

```javascript
// auth-service/src/services/oauthService.js
const axios = require('axios');
const config = require('../config');

class OAuthService {
  async getGoogleAuthUrl() {
    const params = new URLSearchParams({
      client_id: config.oauth.google.clientId,
      redirect_uri: config.oauth.google.redirectUri,
      response_type: 'code',
      scope: 'openid email profile',
      access_type: 'offline',
    });

    return `https://accounts.google.com/o/oauth2/v2/auth?${params}`;
  }

  async handleGoogleCallback(code) {
    // Exchange code for tokens
    const tokenResponse = await axios.post('https://oauth2.googleapis.com/token', {
      code,
      client_id: config.oauth.google.clientId,
      client_secret: config.oauth.google.clientSecret,
      redirect_uri: config.oauth.google.redirectUri,
      grant_type: 'authorization_code',
    });

    const { access_token } = tokenResponse.data;

    // Get user info
    const userResponse = await axios.get(
      'https://www.googleapis.com/oauth2/v2/userinfo',
      { headers: { Authorization: `Bearer ${access_token}` } }
    );

    return this.processOAuthUser({
      provider: 'google',
      providerId: userResponse.data.id,
      email: userResponse.data.email,
      firstName: userResponse.data.given_name,
      lastName: userResponse.data.family_name,
      avatar: userResponse.data.picture,
    });
  }

  async processOAuthUser({ provider, providerId, email, firstName, lastName, avatar }) {
    const userRepository = require('../repositories/userRepository');
    const tokenService = require('./tokenService');

    // Find existing OAuth connection
    let user = await userRepository.findByOAuthProvider(provider, providerId);

    if (!user) {
      // Find by email (merge accounts)
      user = await userRepository.findByEmail(email);

      if (user) {
        // Link OAuth to existing account
        await userRepository.addOAuthProvider(user.id, { provider, providerId });
      } else {
        // Create new user
        user = await userRepository.create({
          email,
          firstName,
          lastName,
          avatar,
          status: 'active', // OAuth users are pre-verified
          emailVerifiedAt: new Date(),
          oauthProviders: [{ provider, providerId }],
        });
      }
    }

    const accessToken = tokenService.generateAccessToken(user);
    const refreshToken = tokenService.generateRefreshToken();
    await tokenService.saveRefreshToken(user.id, refreshToken, { type: 'oauth' });

    return { user, accessToken, refreshToken };
  }
}

module.exports = new OAuthService();
```

---

## 12.8 Security Best Practices

### Password Policy

```javascript
// shared/src/validators/passwordPolicy.js
const zxcvbn = require('zxcvbn');

function validatePassword(password, userInputs = []) {
  const errors = [];

  if (password.length < 8) {
    errors.push('Password must be at least 8 characters');
  }

  if (!/[A-Z]/.test(password)) {
    errors.push('Password must contain at least one uppercase letter');
  }

  if (!/[a-z]/.test(password)) {
    errors.push('Password must contain at least one lowercase letter');
  }

  if (!/\d/.test(password)) {
    errors.push('Password must contain at least one number');
  }

  if (!/[!@#$%^&*()_+\-=\[\]{};':"\\|,.<>\/?]/.test(password)) {
    errors.push('Password must contain at least one special character');
  }

  // Check password strength
  const strength = zxcvbn(password, userInputs);
  if (strength.score < 2) {
    errors.push(`Password is too weak: ${strength.feedback.warning || strength.feedback.suggestions[0]}`);
  }

  return {
    valid: errors.length === 0,
    errors,
    score: strength.score, // 0-4
  };
}

module.exports = { validatePassword };
```

### Brute Force Protection

```javascript
// auth-service/src/middleware/bruteForce.js
const Redis = require('ioredis');
const config = require('../config');

const redis = new Redis(config.redis);

const LIMITS = {
  maxAttempts: 5,
  windowMs: 15 * 60 * 1000, // 15 minutes
  lockDurationMs: 30 * 60 * 1000, // 30 minutes
};

async function checkBruteForce(req, res, next) {
  const key = `brute:${req.ip}:${req.body.email}`;
  const lockKey = `locked:${req.ip}:${req.body.email}`;

  // Check if locked
  const isLocked = await redis.get(lockKey);
  if (isLocked) {
    return res.status(429).json({
      success: false,
      error: {
        code: 'ACCOUNT_LOCKED',
        message: 'Too many failed attempts. Account temporarily locked.',
        lockedUntil: new Date(Date.now() + LIMITS.lockDurationMs).toISOString(),
      },
    });
  }

  req._bruteForce = { key, lockKey };
  next();
}

async function recordFailedAttempt(key, lockKey) {
  const multi = redis.multi();
  multi.incr(key);
  multi.expire(key, Math.ceil(LIMITS.windowMs / 1000));
  const results = await multi.exec();

  const attempts = results[0][1];
  if (attempts >= LIMITS.maxAttempts) {
    await redis.setex(lockKey, Math.ceil(LIMITS.lockDurationMs / 1000), '1');
    await redis.del(key);
  }
}

async function clearAttempts(key) {
  await redis.del(key);
}

module.exports = { checkBruteForce, recordFailedAttempt, clearAttempts };
```

---

## 12.9 Testing Auth Service

```javascript
// tests/auth.test.js
const request = require('supertest');
const app = require('../src/app');
const { Pool } = require('pg');
const Redis = require('ioredis');

describe('Auth Service', () => {
  let db;
  let redis;

  beforeAll(async () => {
    db = new Pool({ connectionString: process.env.TEST_DATABASE_URL });
    redis = new Redis(process.env.TEST_REDIS_URL);
  });

  afterAll(async () => {
    await db.end();
    await redis.quit();
  });

  beforeEach(async () => {
    await db.query('DELETE FROM users WHERE email LIKE $1', ['test-%@example.com']);
    await redis.flushdb();
  });

  describe('POST /auth/register', () => {
    it('should register a new user', async () => {
      const response = await request(app)
        .post('/auth/register')
        .send({
          email: 'test-new@example.com',
          password: 'SecureP@ss123',
          firstName: 'Test',
          lastName: 'User',
        })
        .expect(201);

      expect(response.body.success).toBe(true);
      expect(response.body.data.user.email).toBe('test-new@example.com');
      expect(response.body.data.user.passwordHash).toBeUndefined();
    });

    it('should reject duplicate email', async () => {
      const userData = {
        email: 'test-dup@example.com',
        password: 'SecureP@ss123',
        firstName: 'Test',
        lastName: 'User',
      };

      await request(app).post('/auth/register').send(userData).expect(201);
      const response = await request(app).post('/auth/register').send(userData).expect(409);

      expect(response.body.error.code).toBe('EMAIL_EXISTS');
    });

    it('should reject weak password', async () => {
      const response = await request(app)
        .post('/auth/register')
        .send({
          email: 'test-weak@example.com',
          password: 'weak',
          firstName: 'Test',
          lastName: 'User',
        })
        .expect(400);

      expect(response.body.success).toBe(false);
    });
  });

  describe('POST /auth/login', () => {
    beforeEach(async () => {
      await request(app).post('/auth/register').send({
        email: 'test-login@example.com',
        password: 'SecureP@ss123',
        firstName: 'Test',
        lastName: 'User',
      });

      // Manually verify email for test
      await db.query(
        'UPDATE users SET status = $1, email_verified_at = NOW() WHERE email = $2',
        ['active', 'test-login@example.com']
      );
    });

    it('should login with valid credentials and return tokens', async () => {
      const response = await request(app)
        .post('/auth/login')
        .send({
          email: 'test-login@example.com',
          password: 'SecureP@ss123',
        })
        .expect(200);

      expect(response.body.data.accessToken).toBeDefined();
      expect(response.body.data.expiresIn).toBe(900);
      expect(response.headers['set-cookie']).toBeDefined(); // refresh token cookie
    });

    it('should reject invalid password', async () => {
      const response = await request(app)
        .post('/auth/login')
        .send({
          email: 'test-login@example.com',
          password: 'WrongPassword',
        })
        .expect(401);

      expect(response.body.error.code).toBe('INVALID_CREDENTIALS');
    });
  });

  describe('POST /auth/refresh', () => {
    it('should refresh tokens using refresh token cookie', async () => {
      // Login to get cookies
      const loginRes = await request(app)
        .post('/auth/login')
        .send({ email: 'test-login@example.com', password: 'SecureP@ss123' });

      const cookies = loginRes.headers['set-cookie'];

      // Use cookie to refresh
      const response = await request(app)
        .post('/auth/refresh')
        .set('Cookie', cookies)
        .expect(200);

      expect(response.body.data.accessToken).toBeDefined();
      // New refresh token cookie should be set
      expect(response.headers['set-cookie']).toBeDefined();
    });
  });

  describe('Token Verification', () => {
    it('should provide valid public key', async () => {
      const response = await request(app)
        .get('/auth/public-key')
        .expect(200);

      expect(response.body.data.publicKey).toContain('BEGIN PUBLIC KEY');
      expect(response.body.data.algorithm).toBe('RS256');
    });

    it('should verify valid access token via introspect', async () => {
      const loginRes = await request(app)
        .post('/auth/login')
        .send({ email: 'test-login@example.com', password: 'SecureP@ss123' });

      const { accessToken } = loginRes.body.data;

      const response = await request(app)
        .post('/auth/introspect')
        .send({ token: accessToken })
        .expect(200);

      expect(response.body.data.active).toBe(true);
      expect(response.body.data.email).toBe('test-login@example.com');
    });
  });
});
```

---

## สรุป Part 12

ในบทนี้เราได้เรียนรู้:
- **Authentication Strategies** - Centralized Auth vs Distributed JWT vs Service Mesh
- **JWT Deep Dive** - RS256 vs HS256, payload design, security
- **Auth Service** - สร้าง production-ready auth service ด้วย Node.js
- **Token Service** - Access token (RS256), Refresh token rotation
- **Distributed Verification** - Services verify JWT ด้วย public key
- **RBAC** - Role-Based Access Control ด้วย permissions
- **API Key Auth** - สำหรับ service-to-service communication
- **OAuth 2.0** - Social login (Google)
- **Security Best Practices** - Brute force protection, password policy
- **Testing** - Unit & Integration tests

**Next:** Part 13 - Message Queues & Asynchronous Communication

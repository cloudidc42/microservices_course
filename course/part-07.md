# Part 07: Building First Production-Ready Microservice with Node.js
## สร้าง Microservice แรกที่พร้อมใช้งานจริงด้วย Node.js และ Express

> **ระดับ:** ⭐⭐ พื้นฐาน | **เวลาเรียน:** 6-8 ชั่วโมง | **Prerequisites:** Parts 01-06

---

## 🎯 สิ่งที่จะได้เรียนรู้

- Project Structure ที่ถูกต้องสำหรับ Microservice
- Repository Pattern และ Service Layer
- Input Validation ด้วย Joi/Zod
- Error Handling ที่ครบถ้วน
- Database Connection Pooling
- API Documentation ด้วย Swagger/OpenAPI
- Testing พื้นฐาน
- Workshop: User Service แบบสมบูรณ์

---

## 1. Architecture ของ Production-Ready Microservice

```
Request → Routes → Middleware → Controller → Service → Repository → Database
                       ↑              ↑           ↑          ↑
                  Validation     Business     Data Access  Database
                  Auth Check      Logic        Layer       Layer
```

```
user-service/
├── src/
│   ├── app.js              # Express app setup
│   ├── server.js           # Server + graceful shutdown
│   ├── config/
│   │   └── index.js        # All config from env vars
│   ├── routes/
│   │   ├── index.js        # Route aggregator
│   │   └── users.routes.js
│   ├── controllers/
│   │   └── users.controller.js  # HTTP layer (req/res)
│   ├── services/
│   │   └── users.service.js     # Business logic
│   ├── repositories/
│   │   └── users.repository.js  # Database operations
│   ├── models/
│   │   └── user.model.js        # Data model
│   ├── middleware/
│   │   ├── auth.middleware.js
│   │   ├── validate.middleware.js
│   │   └── error.middleware.js
│   ├── validators/
│   │   └── user.validator.js    # Joi schemas
│   ├── utils/
│   │   ├── logger.js
│   │   ├── database.js
│   │   └── AppError.js
│   └── docs/
│       └── swagger.js           # API documentation
├── tests/
│   ├── unit/
│   │   └── users.service.test.js
│   └── integration/
│       └── users.api.test.js
├── database/
│   └── migrations/
│       └── 001_create_users.sql
├── .env.example
├── .gitignore
├── Dockerfile
├── docker-compose.yml
└── package.json
```

---

## 2. Project Setup

### 2.1 package.json

```json
{
  "name": "user-service",
  "version": "1.0.0",
  "description": "User Management Microservice",
  "main": "src/server.js",
  "scripts": {
    "start": "NODE_ENV=production node src/server.js",
    "dev": "nodemon --watch src --ext js,json src/server.js",
    "test": "jest --forceExit",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage --forceExit",
    "lint": "eslint 'src/**/*.js'",
    "lint:fix": "eslint 'src/**/*.js' --fix",
    "migrate": "node database/migrate.js",
    "seed": "node database/seed.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "pg": "^8.11.3",
    "bcryptjs": "^2.4.3",
    "jsonwebtoken": "^9.0.2",
    "joi": "^17.11.0",
    "winston": "^3.11.0",
    "helmet": "^7.1.0",
    "cors": "^2.8.5",
    "compression": "^1.7.4",
    "express-rate-limit": "^7.1.5",
    "uuid": "^9.0.0",
    "swagger-jsdoc": "^6.2.8",
    "swagger-ui-express": "^5.0.0"
  },
  "devDependencies": {
    "nodemon": "^3.0.1",
    "jest": "^29.7.0",
    "supertest": "^6.3.3",
    "eslint": "^8.54.0"
  }
}
```

---

## 3. Configuration

```javascript
// src/config/index.js
const config = {
  app: {
    name: process.env.SERVICE_NAME || 'user-service',
    port: parseInt(process.env.PORT, 10) || 3001,
    env: process.env.NODE_ENV || 'development',
    version: process.env.APP_VERSION || '1.0.0',
    logLevel: process.env.LOG_LEVEL || 'info',
  },
  
  database: {
    host: process.env.DB_HOST || 'localhost',
    port: parseInt(process.env.DB_PORT, 10) || 5432,
    name: process.env.DB_NAME || 'userdb',
    user: process.env.DB_USER || 'postgres',
    password: process.env.DB_PASSWORD,
    maxConnections: parseInt(process.env.DB_MAX_CONNECTIONS, 10) || 10,
    idleTimeout: parseInt(process.env.DB_IDLE_TIMEOUT, 10) || 30000,
    connectionTimeout: parseInt(process.env.DB_CONNECTION_TIMEOUT, 10) || 2000,
  },
  
  jwt: {
    secret: process.env.JWT_SECRET,
    expiresIn: process.env.JWT_EXPIRES_IN || '7d',
    refreshExpiresIn: process.env.JWT_REFRESH_EXPIRES_IN || '30d',
  },
  
  bcrypt: {
    saltRounds: parseInt(process.env.BCRYPT_SALT_ROUNDS, 10) || 12,
  },
  
  rateLimit: {
    windowMs: parseInt(process.env.RATE_LIMIT_WINDOW_MS, 10) || 15 * 60 * 1000,
    max: parseInt(process.env.RATE_LIMIT_MAX, 10) || 100,
  },
  
  cors: {
    origin: process.env.CORS_ORIGIN ? 
      process.env.CORS_ORIGIN.split(',') : 
      ['http://localhost:3000'],
    credentials: true,
  },
};

// Validate required configuration
function validateConfig(cfg) {
  const required = [];
  
  if (cfg.app.env === 'production') {
    if (!process.env.DB_PASSWORD) required.push('DB_PASSWORD');
    if (!process.env.JWT_SECRET) required.push('JWT_SECRET');
  }
  
  if (required.length > 0) {
    throw new Error(
      `Missing required environment variables: ${required.join(', ')}`
    );
  }
}

validateConfig(config);

module.exports = config;
```

---

## 4. Database Setup

```javascript
// src/utils/database.js
const { Pool } = require('pg');
const config = require('../config');
const logger = require('./logger');

class Database {
  constructor() {
    this.pool = null;
  }
  
  connect() {
    if (this.pool) return this.pool;
    
    this.pool = new Pool({
      host: config.database.host,
      port: config.database.port,
      database: config.database.name,
      user: config.database.user,
      password: config.database.password,
      max: config.database.maxConnections,
      idleTimeoutMillis: config.database.idleTimeout,
      connectionTimeoutMillis: config.database.connectionTimeout,
    });
    
    this.pool.on('connect', () => {
      logger.debug('New database connection established');
    });
    
    this.pool.on('error', (err) => {
      logger.error('Unexpected database error', { error: err.message });
    });
    
    return this.pool;
  }
  
  async query(text, params) {
    const start = Date.now();
    
    try {
      const result = await this.pool.query(text, params);
      const duration = Date.now() - start;
      
      logger.debug('Query executed', {
        query: text.substring(0, 100),
        duration,
        rows: result.rowCount,
      });
      
      return result;
    } catch (error) {
      logger.error('Query failed', {
        query: text.substring(0, 100),
        error: error.message,
      });
      throw error;
    }
  }
  
  async transaction(callback) {
    const client = await this.pool.connect();
    
    try {
      await client.query('BEGIN');
      const result = await callback(client);
      await client.query('COMMIT');
      return result;
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }
  
  async end() {
    if (this.pool) {
      await this.pool.end();
      this.pool = null;
      logger.info('Database pool closed');
    }
  }
  
  async healthCheck() {
    try {
      const result = await this.query('SELECT NOW()');
      return { status: 'healthy', timestamp: result.rows[0].now };
    } catch (error) {
      return { status: 'unhealthy', error: error.message };
    }
  }
}

// Singleton
const db = new Database();
db.connect();

module.exports = db;
```

---

## 5. Models

```javascript
// src/models/user.model.js
class User {
  constructor(data) {
    this.id = data.id;
    this.email = data.email;
    this.name = data.name;
    this.passwordHash = data.password_hash || data.passwordHash;
    this.role = data.role || 'customer';
    this.isActive = data.is_active !== undefined ? data.is_active : true;
    this.createdAt = data.created_at || data.createdAt;
    this.updatedAt = data.updated_at || data.updatedAt;
  }
  
  // Serialize for API response (exclude sensitive fields)
  toJSON() {
    return {
      id: this.id,
      email: this.email,
      name: this.name,
      role: this.role,
      isActive: this.isActive,
      createdAt: this.createdAt,
      updatedAt: this.updatedAt,
    };
  }
  
  // Create from database row
  static fromDB(row) {
    if (!row) return null;
    return new User(row);
  }
}

module.exports = User;
```

---

## 6. Repository Layer

```javascript
// src/repositories/users.repository.js
const db = require('../utils/database');
const User = require('../models/user.model');
const logger = require('../utils/logger');

class UsersRepository {
  // ── Find Operations ──
  
  async findAll({ page = 1, limit = 10, role, search } = {}) {
    const offset = (page - 1) * limit;
    const params = [];
    let whereClause = 'WHERE is_active = true';
    
    if (role) {
      params.push(role);
      whereClause += ` AND role = $${params.length}`;
    }
    
    if (search) {
      params.push(`%${search}%`);
      whereClause += ` AND (name ILIKE $${params.length} OR email ILIKE $${params.length})`;
    }
    
    params.push(limit, offset);
    
    const [dataResult, countResult] = await Promise.all([
      db.query(
        `SELECT * FROM users ${whereClause} 
         ORDER BY created_at DESC 
         LIMIT $${params.length - 1} OFFSET $${params.length}`,
        params
      ),
      db.query(
        `SELECT COUNT(*) FROM users ${whereClause}`,
        params.slice(0, -2)  // Remove limit and offset
      )
    ]);
    
    return {
      users: dataResult.rows.map(User.fromDB),
      total: parseInt(countResult.rows[0].count, 10),
      page,
      limit,
      totalPages: Math.ceil(parseInt(countResult.rows[0].count, 10) / limit),
    };
  }
  
  async findById(id) {
    const result = await db.query(
      'SELECT * FROM users WHERE id = $1 AND is_active = true',
      [id]
    );
    return User.fromDB(result.rows[0]);
  }
  
  async findByEmail(email) {
    const result = await db.query(
      'SELECT * FROM users WHERE email = $1',
      [email.toLowerCase()]
    );
    return User.fromDB(result.rows[0]);
  }
  
  // ── Write Operations ──
  
  async create({ email, name, passwordHash, role = 'customer' }) {
    const result = await db.query(
      `INSERT INTO users (email, name, password_hash, role, is_active)
       VALUES ($1, $2, $3, $4, true)
       RETURNING *`,
      [email.toLowerCase(), name, passwordHash, role]
    );
    return User.fromDB(result.rows[0]);
  }
  
  async update(id, { name, role }) {
    const fields = [];
    const values = [];
    
    if (name !== undefined) {
      values.push(name);
      fields.push(`name = $${values.length}`);
    }
    
    if (role !== undefined) {
      values.push(role);
      fields.push(`role = $${values.length}`);
    }
    
    if (fields.length === 0) return this.findById(id);
    
    values.push(new Date(), id);
    
    const result = await db.query(
      `UPDATE users 
       SET ${fields.join(', ')}, updated_at = $${values.length - 1}
       WHERE id = $${values.length} AND is_active = true
       RETURNING *`,
      values
    );
    
    return User.fromDB(result.rows[0]);
  }
  
  async updatePassword(id, passwordHash) {
    const result = await db.query(
      `UPDATE users SET password_hash = $1, updated_at = NOW()
       WHERE id = $2 RETURNING *`,
      [passwordHash, id]
    );
    return User.fromDB(result.rows[0]);
  }
  
  async delete(id) {
    // Soft delete
    const result = await db.query(
      `UPDATE users SET is_active = false, updated_at = NOW()
       WHERE id = $1 RETURNING *`,
      [id]
    );
    return result.rowCount > 0;
  }
  
  async emailExists(email, excludeId = null) {
    const params = [email.toLowerCase()];
    let query = 'SELECT id FROM users WHERE email = $1';
    
    if (excludeId) {
      params.push(excludeId);
      query += ` AND id != $2`;
    }
    
    const result = await db.query(query, params);
    return result.rows.length > 0;
  }
}

module.exports = new UsersRepository();
```

---

## 7. Service Layer (Business Logic)

```javascript
// src/services/users.service.js
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');
const usersRepository = require('../repositories/users.repository');
const config = require('../config');
const { AppError, ConflictError, NotFoundError, UnauthorizedError } = require('../utils/AppError');
const logger = require('../utils/logger');

class UsersService {
  // ── User CRUD ──
  
  async getUsers(options) {
    return usersRepository.findAll(options);
  }
  
  async getUserById(id) {
    const user = await usersRepository.findById(id);
    if (!user) throw new NotFoundError('User');
    return user;
  }
  
  async createUser({ email, name, password, role }) {
    // Check email uniqueness
    const exists = await usersRepository.emailExists(email);
    if (exists) {
      throw new ConflictError('Email already registered');
    }
    
    // Hash password
    const passwordHash = await bcrypt.hash(password, config.bcrypt.saltRounds);
    
    // Create user
    const user = await usersRepository.create({
      email,
      name,
      passwordHash,
      role,
    });
    
    logger.info('User created', { userId: user.id, email: user.email });
    
    return user;
  }
  
  async updateUser(id, updates) {
    // Verify user exists
    await this.getUserById(id);
    
    const user = await usersRepository.update(id, updates);
    if (!user) throw new NotFoundError('User');
    
    logger.info('User updated', { userId: id });
    
    return user;
  }
  
  async deleteUser(id) {
    // Verify user exists
    await this.getUserById(id);
    
    const deleted = await usersRepository.delete(id);
    if (!deleted) throw new NotFoundError('User');
    
    logger.info('User deleted', { userId: id });
    
    return { message: 'User deleted successfully' };
  }
  
  // ── Authentication ──
  
  async login({ email, password }) {
    // Find user with password hash (special query)
    const result = await require('../utils/database').query(
      'SELECT * FROM users WHERE email = $1 AND is_active = true',
      [email.toLowerCase()]
    );
    
    const userRow = result.rows[0];
    
    if (!userRow) {
      throw new UnauthorizedError('Invalid email or password');
    }
    
    // Verify password
    const isValidPassword = await bcrypt.compare(password, userRow.password_hash);
    if (!isValidPassword) {
      logger.warn('Failed login attempt', { email });
      throw new UnauthorizedError('Invalid email or password');
    }
    
    // Generate tokens
    const { accessToken, refreshToken } = this.generateTokens(userRow);
    
    const User = require('../models/user.model');
    const user = User.fromDB(userRow);
    
    logger.info('User logged in', { userId: user.id });
    
    return { user, accessToken, refreshToken };
  }
  
  async changePassword(userId, { currentPassword, newPassword }) {
    const result = await require('../utils/database').query(
      'SELECT * FROM users WHERE id = $1',
      [userId]
    );
    
    const userRow = result.rows[0];
    if (!userRow) throw new NotFoundError('User');
    
    // Verify current password
    const isValid = await bcrypt.compare(currentPassword, userRow.password_hash);
    if (!isValid) {
      throw new UnauthorizedError('Current password is incorrect');
    }
    
    // Hash new password
    const newHash = await bcrypt.hash(newPassword, config.bcrypt.saltRounds);
    await usersRepository.updatePassword(userId, newHash);
    
    logger.info('Password changed', { userId });
    
    return { message: 'Password changed successfully' };
  }
  
  generateTokens(user) {
    const payload = {
      sub: user.id,
      email: user.email,
      role: user.role,
    };
    
    const accessToken = jwt.sign(payload, config.jwt.secret, {
      expiresIn: config.jwt.expiresIn,
      issuer: config.app.name,
    });
    
    const refreshToken = jwt.sign(
      { sub: user.id, type: 'refresh' },
      config.jwt.secret,
      { expiresIn: config.jwt.refreshExpiresIn }
    );
    
    return { accessToken, refreshToken };
  }
  
  verifyToken(token) {
    try {
      return jwt.verify(token, config.jwt.secret);
    } catch (error) {
      throw new UnauthorizedError('Invalid or expired token');
    }
  }
}

module.exports = new UsersService();
```

---

## 8. Validators

```javascript
// src/validators/user.validator.js
const Joi = require('joi');

const passwordSchema = Joi.string()
  .min(8)
  .max(128)
  .pattern(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/)
  .messages({
    'string.pattern.base': 'Password must contain uppercase, lowercase, and number',
    'string.min': 'Password must be at least 8 characters',
  });

const schemas = {
  // POST /users
  createUser: Joi.object({
    email: Joi.string().email().lowercase().required(),
    name: Joi.string().min(2).max(100).required(),
    password: passwordSchema.required(),
    role: Joi.string().valid('customer', 'admin', 'staff').default('customer'),
  }),
  
  // PUT /users/:id
  updateUser: Joi.object({
    name: Joi.string().min(2).max(100),
    role: Joi.string().valid('customer', 'admin', 'staff'),
  }).min(1).messages({
    'object.min': 'At least one field must be provided for update',
  }),
  
  // POST /auth/login
  login: Joi.object({
    email: Joi.string().email().lowercase().required(),
    password: Joi.string().required(),
  }),
  
  // POST /auth/change-password
  changePassword: Joi.object({
    currentPassword: Joi.string().required(),
    newPassword: passwordSchema.required(),
  }),
  
  // GET /users (query params)
  getUsers: Joi.object({
    page: Joi.number().integer().min(1).default(1),
    limit: Joi.number().integer().min(1).max(100).default(10),
    role: Joi.string().valid('customer', 'admin', 'staff'),
    search: Joi.string().max(100),
  }),
};

module.exports = schemas;
```

---

## 9. Middleware

```javascript
// src/middleware/validate.middleware.js
const logger = require('../utils/logger');

function validate(schema, source = 'body') {
  return (req, res, next) => {
    const data = req[source];
    
    const { error, value } = schema.validate(data, {
      abortEarly: false,    // Report all errors, not just first
      stripUnknown: true,   // Remove unknown fields
      convert: true,        // Type coercion (string → number)
    });
    
    if (error) {
      const details = error.details.map(d => ({
        field: d.path.join('.'),
        message: d.message,
      }));
      
      logger.warn('Validation failed', { details });
      
      return res.status(400).json({
        success: false,
        error: 'VALIDATION_ERROR',
        message: 'Invalid input data',
        details,
      });
    }
    
    // Replace with validated and sanitized value
    req[source] = value;
    next();
  };
}

module.exports = validate;
```

```javascript
// src/middleware/auth.middleware.js
const usersService = require('../services/users.service');
const { UnauthorizedError, ForbiddenError } = require('../utils/AppError');

function authenticate(req, res, next) {
  const authHeader = req.headers.authorization;
  
  if (!authHeader || !authHeader.startsWith('Bearer ')) {
    return next(new UnauthorizedError('No token provided'));
  }
  
  const token = authHeader.split(' ')[1];
  
  try {
    const payload = usersService.verifyToken(token);
    req.user = payload;
    next();
  } catch (error) {
    next(error);
  }
}

function authorize(...roles) {
  return (req, res, next) => {
    if (!req.user) {
      return next(new UnauthorizedError('Authentication required'));
    }
    
    if (roles.length > 0 && !roles.includes(req.user.role)) {
      return next(new ForbiddenError('Insufficient permissions'));
    }
    
    next();
  };
}

module.exports = { authenticate, authorize };
```

```javascript
// src/utils/AppError.js
class AppError extends Error {
  constructor(message, statusCode = 500, code = 'INTERNAL_ERROR') {
    super(message);
    this.name = this.constructor.name;
    this.statusCode = statusCode;
    this.code = code;
    this.isOperational = true;
    Error.captureStackTrace(this, this.constructor);
  }
}

class ValidationError extends AppError {
  constructor(message, details) {
    super(message, 400, 'VALIDATION_ERROR');
    this.details = details;
  }
}

class NotFoundError extends AppError {
  constructor(resource = 'Resource') {
    super(`${resource} not found`, 404, 'NOT_FOUND');
  }
}

class ConflictError extends AppError {
  constructor(message) {
    super(message, 409, 'CONFLICT');
  }
}

class UnauthorizedError extends AppError {
  constructor(message = 'Unauthorized') {
    super(message, 401, 'UNAUTHORIZED');
  }
}

class ForbiddenError extends AppError {
  constructor(message = 'Forbidden') {
    super(message, 403, 'FORBIDDEN');
  }
}

module.exports = {
  AppError,
  ValidationError,
  NotFoundError,
  ConflictError,
  UnauthorizedError,
  ForbiddenError,
};
```

---

## 10. Controllers

```javascript
// src/controllers/users.controller.js
const usersService = require('../services/users.service');
const logger = require('../utils/logger');

class UsersController {
  async getUsers(req, res, next) {
    try {
      const result = await usersService.getUsers(req.query);
      
      res.json({
        success: true,
        data: result.users.map(u => u.toJSON()),
        pagination: {
          page: result.page,
          limit: result.limit,
          total: result.total,
          totalPages: result.totalPages,
        },
      });
    } catch (error) {
      next(error);
    }
  }
  
  async getUserById(req, res, next) {
    try {
      const user = await usersService.getUserById(parseInt(req.params.id));
      res.json({ success: true, data: user.toJSON() });
    } catch (error) {
      next(error);
    }
  }
  
  async createUser(req, res, next) {
    try {
      const user = await usersService.createUser(req.body);
      
      res.status(201).json({
        success: true,
        message: 'User created successfully',
        data: user.toJSON(),
      });
    } catch (error) {
      next(error);
    }
  }
  
  async updateUser(req, res, next) {
    try {
      const user = await usersService.updateUser(
        parseInt(req.params.id), 
        req.body
      );
      
      res.json({
        success: true,
        message: 'User updated successfully',
        data: user.toJSON(),
      });
    } catch (error) {
      next(error);
    }
  }
  
  async deleteUser(req, res, next) {
    try {
      const result = await usersService.deleteUser(parseInt(req.params.id));
      res.json({ success: true, ...result });
    } catch (error) {
      next(error);
    }
  }
  
  async login(req, res, next) {
    try {
      const { user, accessToken, refreshToken } = await usersService.login(req.body);
      
      res.json({
        success: true,
        message: 'Login successful',
        data: {
          user: user.toJSON(),
          accessToken,
          refreshToken,
        },
      });
    } catch (error) {
      next(error);
    }
  }
  
  async getProfile(req, res, next) {
    try {
      const user = await usersService.getUserById(req.user.sub);
      res.json({ success: true, data: user.toJSON() });
    } catch (error) {
      next(error);
    }
  }
  
  async changePassword(req, res, next) {
    try {
      const result = await usersService.changePassword(req.user.sub, req.body);
      res.json({ success: true, ...result });
    } catch (error) {
      next(error);
    }
  }
}

module.exports = new UsersController();
```

---

## 11. Routes

```javascript
// src/routes/users.routes.js
const express = require('express');
const router = express.Router();
const usersController = require('../controllers/users.controller');
const validate = require('../middleware/validate.middleware');
const { authenticate, authorize } = require('../middleware/auth.middleware');
const userSchemas = require('../validators/user.validator');

/**
 * @swagger
 * /api/users:
 *   get:
 *     tags: [Users]
 *     summary: Get all users
 *     security:
 *       - bearerAuth: []
 *     parameters:
 *       - in: query
 *         name: page
 *         schema:
 *           type: integer
 *       - in: query
 *         name: limit
 *         schema:
 *           type: integer
 *     responses:
 *       200:
 *         description: List of users
 */

// Public routes
router.post('/auth/login',
  validate(userSchemas.login),
  usersController.login
);

// Protected routes
router.use(authenticate);

// Get profile (any authenticated user)
router.get('/profile', usersController.getProfile);

router.post('/auth/change-password',
  validate(userSchemas.changePassword),
  usersController.changePassword
);

// Admin routes
router.get('/',
  authorize('admin'),
  validate(userSchemas.getUsers, 'query'),
  usersController.getUsers
);

router.get('/:id',
  authorize('admin', 'staff'),
  usersController.getUserById
);

router.post('/',
  authorize('admin'),
  validate(userSchemas.createUser),
  usersController.createUser
);

router.put('/:id',
  authorize('admin'),
  validate(userSchemas.updateUser),
  usersController.updateUser
);

router.delete('/:id',
  authorize('admin'),
  usersController.deleteUser
);

module.exports = router;
```

---

## 12. Testing

```javascript
// tests/unit/users.service.test.js
const usersService = require('../../src/services/users.service');
const usersRepository = require('../../src/repositories/users.repository');
const bcrypt = require('bcryptjs');

// Mock repository
jest.mock('../../src/repositories/users.repository');
jest.mock('bcryptjs');

describe('UsersService', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });
  
  describe('createUser', () => {
    it('should create a new user successfully', async () => {
      // Arrange
      const userData = {
        email: 'test@example.com',
        name: 'Test User',
        password: 'Password123',
      };
      
      const mockUser = {
        id: 1,
        email: userData.email,
        name: userData.name,
        role: 'customer',
        toJSON: () => ({ id: 1, email: userData.email, name: userData.name }),
      };
      
      usersRepository.emailExists.mockResolvedValue(false);
      bcrypt.hash.mockResolvedValue('hashedpassword');
      usersRepository.create.mockResolvedValue(mockUser);
      
      // Act
      const result = await usersService.createUser(userData);
      
      // Assert
      expect(usersRepository.emailExists).toHaveBeenCalledWith(userData.email);
      expect(bcrypt.hash).toHaveBeenCalledWith(userData.password, expect.any(Number));
      expect(usersRepository.create).toHaveBeenCalledWith({
        email: userData.email,
        name: userData.name,
        passwordHash: 'hashedpassword',
        role: 'customer',
      });
      expect(result).toBe(mockUser);
    });
    
    it('should throw ConflictError if email already exists', async () => {
      // Arrange
      usersRepository.emailExists.mockResolvedValue(true);
      
      // Act & Assert
      await expect(usersService.createUser({
        email: 'existing@example.com',
        name: 'Test',
        password: 'Password123',
      })).rejects.toThrow('Email already registered');
    });
  });
  
  describe('getUserById', () => {
    it('should return user if exists', async () => {
      const mockUser = { id: 1, email: 'test@test.com', toJSON: () => ({}) };
      usersRepository.findById.mockResolvedValue(mockUser);
      
      const result = await usersService.getUserById(1);
      expect(result).toBe(mockUser);
    });
    
    it('should throw NotFoundError if user does not exist', async () => {
      usersRepository.findById.mockResolvedValue(null);
      
      await expect(usersService.getUserById(999))
        .rejects.toThrow('User not found');
    });
  });
});
```

```javascript
// tests/integration/users.api.test.js
const request = require('supertest');
const app = require('../../src/app');
const db = require('../../src/utils/database');

describe('Users API', () => {
  let adminToken;
  
  beforeAll(async () => {
    // Setup test database
    await db.query(`
      INSERT INTO users (email, name, password_hash, role)
      VALUES ('admin@test.com', 'Admin User', '$2a$12$...', 'admin')
      ON CONFLICT DO NOTHING
    `);
    
    // Login to get token
    const response = await request(app)
      .post('/api/auth/login')
      .send({ email: 'admin@test.com', password: 'AdminPassword123' });
    
    adminToken = response.body.data.accessToken;
  });
  
  afterAll(async () => {
    await db.query("DELETE FROM users WHERE email LIKE '%@test.com'");
    await db.end();
  });
  
  describe('GET /api/users', () => {
    it('should return users list for admin', async () => {
      const response = await request(app)
        .get('/api/users')
        .set('Authorization', `Bearer ${adminToken}`)
        .expect(200);
      
      expect(response.body.success).toBe(true);
      expect(Array.isArray(response.body.data)).toBe(true);
      expect(response.body.pagination).toBeDefined();
    });
    
    it('should return 401 without token', async () => {
      await request(app)
        .get('/api/users')
        .expect(401);
    });
  });
  
  describe('POST /api/users', () => {
    it('should create a new user', async () => {
      const response = await request(app)
        .post('/api/users')
        .set('Authorization', `Bearer ${adminToken}`)
        .send({
          email: 'newuser@test.com',
          name: 'New User',
          password: 'NewPassword123',
        })
        .expect(201);
      
      expect(response.body.success).toBe(true);
      expect(response.body.data.email).toBe('newuser@test.com');
      expect(response.body.data.passwordHash).toBeUndefined();  // Sensitive field hidden
    });
    
    it('should return 409 for duplicate email', async () => {
      await request(app)
        .post('/api/users')
        .set('Authorization', `Bearer ${adminToken}`)
        .send({
          email: 'admin@test.com',  // Already exists
          name: 'Duplicate',
          password: 'Password123',
        })
        .expect(409);
    });
  });
});
```

---

## 13. API Documentation

```javascript
// src/docs/swagger.js
const swaggerJsdoc = require('swagger-jsdoc');
const swaggerUi = require('swagger-ui-express');
const config = require('../config');

const options = {
  definition: {
    openapi: '3.0.0',
    info: {
      title: 'User Service API',
      version: config.app.version,
      description: 'Microservice for user management, authentication, and authorization',
    },
    servers: [
      { url: `http://localhost:${config.app.port}`, description: 'Development' },
      { url: 'https://api.company.com', description: 'Production' },
    ],
    components: {
      securitySchemes: {
        bearerAuth: {
          type: 'http',
          scheme: 'bearer',
          bearerFormat: 'JWT',
        },
      },
      schemas: {
        User: {
          type: 'object',
          properties: {
            id: { type: 'integer' },
            email: { type: 'string', format: 'email' },
            name: { type: 'string' },
            role: { type: 'string', enum: ['customer', 'admin', 'staff'] },
            isActive: { type: 'boolean' },
            createdAt: { type: 'string', format: 'date-time' },
          },
        },
        Error: {
          type: 'object',
          properties: {
            success: { type: 'boolean', example: false },
            error: { type: 'string' },
            message: { type: 'string' },
          },
        },
      },
    },
  },
  apis: ['./src/routes/*.js'],
};

const swaggerSpec = swaggerJsdoc(options);

function setupSwagger(app) {
  app.use('/api/docs', swaggerUi.serve, swaggerUi.setup(swaggerSpec));
  app.get('/api/docs.json', (req, res) => res.json(swaggerSpec));
}

module.exports = setupSwagger;
```

---

## 14. Running and Testing

```bash
# Setup
npm install
cp .env.example .env
# Edit .env with your values

# Development
docker-compose up -d postgres  # Start only database
npm run dev                     # Start service with hot reload

# Test
npm test
npm run test:coverage

# API Documentation
# Open: http://localhost:3001/api/docs

# Test with curl
# Login
curl -X POST http://localhost:3001/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "admin@test.com", "password": "AdminPassword123"}'

# Get users (with token)
TOKEN="your-jwt-token"
curl http://localhost:3001/api/users \
  -H "Authorization: Bearer $TOKEN"
```

---

## 15. สรุป

ใน Part นี้เราได้สร้าง Production-Ready Microservice ที่มี:
- **Clean Architecture:** Controller → Service → Repository
- **Input Validation** ด้วย Joi
- **Proper Error Handling** ด้วย Custom Error classes
- **JWT Authentication** 
- **Database Connection Pooling**
- **Structured Logging**
- **Unit และ Integration Tests**
- **API Documentation** ด้วย Swagger

---

**ต่อไป:** [Part 08 - REST API Design Best Practices →](part-08.md)

# Part 11: Database per Service Pattern

## บทนำ

"Database per Service" คือหนึ่งในหลักการสำคัญที่สุดของ Microservices - แต่ละ service มี database ของตัวเองและไม่แชร์กับ service อื่น

---

## 11.1 ทำไมต้องมี Database per Service?

### ปัญหาของ Shared Database

```
❌ ANTI-PATTERN: Shared Database

User Service ─────────────────────┐
Product Service ──────────────────┤── Shared DB
Order Service ────────────────────┤   (PostgreSQL)
Payment Service ──────────────────┘
```

**ปัญหาที่เกิดขึ้น:**
1. **Tight Coupling** - schema เปลี่ยนแล้วกระทบทุก service
2. **Single Point of Failure** - DB ล้มแล้วทุก service ล้มตาม
3. **Performance Bottleneck** - ทุก service แย่ง connection pool
4. **Independent Deployment** - ไม่สามารถ deploy แยกได้จริง ๆ
5. **Technology Lock-in** - ทุก service ต้องใช้ DB เดียวกัน

### Database per Service แก้ได้อย่างไร?

```
✅ GOOD PATTERN: Database per Service

User Service ──── PostgreSQL (users DB)
Product Service ─ MongoDB (products DB)
Order Service ─── PostgreSQL (orders DB)
Payment Service ─ PostgreSQL (payments DB)
Review Service ── MongoDB (reviews DB)
Search Service ── Elasticsearch (search index)
Cache Service ─── Redis (cache)
Analytics ──────── ClickHouse (analytics)
```

**ข้อดี:**
- แต่ละ service เลือก database ที่เหมาะกับ use case
- Scale แต่ละ DB แยกกัน
- ไม่มี shared schema
- Deploy แยก service ได้อิสระ
- ความเสียหายอยู่ใน service เดียว

---

## 11.2 เลือก Database ให้เหมาะกับ Service

### Decision Matrix

| Service | Use Case | Database ที่เหมาะ | เหตุผล |
|---------|----------|------------------|--------|
| User Service | CRUD, Relational | PostgreSQL | Strong consistency, ACID |
| Product Service | Flexible schema, Catalog | MongoDB | Flexible schema, ง่ายต่อการเพิ่ม attributes |
| Order Service | Transactions | PostgreSQL | ACID transactions, foreign keys |
| Payment Service | Critical financial | PostgreSQL | Strong consistency, audit trail |
| Session/Cache | Ephemeral, Fast | Redis | In-memory, TTL support |
| Search Service | Full-text search | Elasticsearch | Inverted index, scoring |
| Notification | Time-series, Logs | MongoDB/ClickHouse | Write-heavy, append-only |
| Analytics | Aggregations | ClickHouse/BigQuery | Columnar storage |
| File Metadata | Graph relations | Neo4j / DynamoDB | - |
| Real-time Chat | Time-ordered | Cassandra | Write-heavy, time-series |

---

## 11.3 PostgreSQL สำหรับ User Service

### Migration Script

```sql
-- migrations/001_create_users_table.sql

-- Enable UUID extension
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

-- Create enum types
CREATE TYPE user_role AS ENUM ('admin', 'seller', 'user');
CREATE TYPE user_status AS ENUM ('active', 'inactive', 'suspended', 'pending_verification');

-- Users table
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    phone VARCHAR(20),
    role user_role NOT NULL DEFAULT 'user',
    status user_status NOT NULL DEFAULT 'pending_verification',
    email_verified_at TIMESTAMPTZ,
    last_login_at TIMESTAMPTZ,
    login_count INTEGER NOT NULL DEFAULT 0,
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at TIMESTAMPTZ
);

-- Indexes for performance
CREATE INDEX idx_users_email ON users(email) WHERE deleted_at IS NULL;
CREATE INDEX idx_users_status ON users(status) WHERE deleted_at IS NULL;
CREATE INDEX idx_users_role ON users(role) WHERE deleted_at IS NULL;
CREATE INDEX idx_users_created_at ON users(created_at DESC);
CREATE INDEX idx_users_metadata ON users USING GIN(metadata);

-- Addresses table
CREATE TABLE user_addresses (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    label VARCHAR(50) DEFAULT 'home',
    street_address TEXT NOT NULL,
    city VARCHAR(100) NOT NULL,
    state VARCHAR(100),
    postal_code VARCHAR(20),
    country VARCHAR(2) NOT NULL DEFAULT 'TH',
    is_default BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_addresses_user_id ON user_addresses(user_id);
CREATE INDEX idx_addresses_default ON user_addresses(user_id, is_default) WHERE is_default = true;

-- Password reset tokens
CREATE TABLE password_reset_tokens (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    token_hash VARCHAR(255) NOT NULL,
    expires_at TIMESTAMPTZ NOT NULL,
    used_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_reset_tokens_user_id ON password_reset_tokens(user_id);
CREATE INDEX idx_reset_tokens_token ON password_reset_tokens(token_hash) WHERE used_at IS NULL;

-- Refresh tokens
CREATE TABLE refresh_tokens (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    token_hash VARCHAR(255) UNIQUE NOT NULL,
    expires_at TIMESTAMPTZ NOT NULL,
    revoked_at TIMESTAMPTZ,
    ip_address INET,
    user_agent TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_refresh_tokens_user_id ON refresh_tokens(user_id);
CREATE INDEX idx_refresh_tokens_token ON refresh_tokens(token_hash) WHERE revoked_at IS NULL;

-- Audit log
CREATE TABLE user_audit_log (
    id BIGSERIAL PRIMARY KEY,
    user_id UUID REFERENCES users(id),
    action VARCHAR(100) NOT NULL,
    details JSONB DEFAULT '{}',
    ip_address INET,
    user_agent TEXT,
    performed_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_audit_log_user_id ON user_audit_log(user_id);
CREATE INDEX idx_audit_log_created_at ON user_audit_log(created_at DESC);

-- Auto-update updated_at trigger
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER update_users_updated_at
    BEFORE UPDATE ON users
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_addresses_updated_at
    BEFORE UPDATE ON user_addresses
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
```

### Migration Runner

```javascript
// src/database/migrate.js
const { Pool } = require('pg');
const fs = require('fs');
const path = require('path');
const config = require('../config');

const pool = new Pool({ connectionString: config.database.url });

async function getMigrations() {
  const migrationsDir = path.join(__dirname, '../../migrations');
  const files = fs.readdirSync(migrationsDir)
    .filter(f => f.endsWith('.sql'))
    .sort();

  return files.map(file => ({
    name: file,
    path: path.join(migrationsDir, file),
    sql: fs.readFileSync(path.join(migrationsDir, file), 'utf8'),
  }));
}

async function runMigrations() {
  const client = await pool.connect();

  try {
    // Create migrations table if not exists
    await client.query(`
      CREATE TABLE IF NOT EXISTS schema_migrations (
        id SERIAL PRIMARY KEY,
        name VARCHAR(255) UNIQUE NOT NULL,
        applied_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
      )
    `);

    const migrations = await getMigrations();
    const { rows: applied } = await client.query(
      'SELECT name FROM schema_migrations ORDER BY name'
    );
    const appliedNames = new Set(applied.map(r => r.name));

    const pending = migrations.filter(m => !appliedNames.has(m.name));

    if (pending.length === 0) {
      console.log('No pending migrations');
      return;
    }

    console.log(`Running ${pending.length} migrations...`);

    for (const migration of pending) {
      await client.query('BEGIN');
      try {
        console.log(`Applying migration: ${migration.name}`);
        await client.query(migration.sql);
        await client.query(
          'INSERT INTO schema_migrations (name) VALUES ($1)',
          [migration.name]
        );
        await client.query('COMMIT');
        console.log(`✓ Applied: ${migration.name}`);
      } catch (error) {
        await client.query('ROLLBACK');
        console.error(`✗ Failed: ${migration.name}`, error.message);
        throw error;
      }
    }

    console.log('All migrations applied successfully');
  } finally {
    client.release();
    await pool.end();
  }
}

if (require.main === module) {
  runMigrations().catch(err => {
    console.error('Migration failed:', err);
    process.exit(1);
  });
}

module.exports = { runMigrations };
```

---

## 11.4 MongoDB สำหรับ Product Service

### Mongoose Schema Design

```javascript
// src/models/Product.js
const mongoose = require('mongoose');

// Sub-schemas
const VariantSchema = new mongoose.Schema({
  sku: { type: String, required: true, unique: true },
  name: { type: String, required: true },
  price: { type: Number, required: true, min: 0 },
  comparePrice: { type: Number, min: 0 },
  stock: { type: Number, required: true, default: 0, min: 0 },
  attributes: { type: Map, of: String }, // { color: 'red', size: 'M' }
  images: [{ type: String }],
  weight: { type: Number },
  barcode: { type: String },
}, { _id: true });

const ReviewSummarySchema = new mongoose.Schema({
  totalCount: { type: Number, default: 0 },
  averageRating: { type: Number, default: 0, min: 0, max: 5 },
  ratingDistribution: {
    1: { type: Number, default: 0 },
    2: { type: Number, default: 0 },
    3: { type: Number, default: 0 },
    4: { type: Number, default: 0 },
    5: { type: Number, default: 0 },
  },
}, { _id: false });

const DimensionSchema = new mongoose.Schema({
  length: Number,
  width: Number,
  height: Number,
  unit: { type: String, enum: ['cm', 'inch'], default: 'cm' },
}, { _id: false });

// Main Product Schema
const ProductSchema = new mongoose.Schema({
  name: {
    type: String,
    required: true,
    trim: true,
    index: 'text', // full-text search
  },
  slug: {
    type: String,
    required: true,
    unique: true,
    lowercase: true,
    trim: true,
  },
  description: {
    type: String,
    required: true,
  },
  shortDescription: {
    type: String,
    maxlength: 200,
  },
  sku: { // Master SKU
    type: String,
    required: true,
    unique: true,
    uppercase: true,
    trim: true,
  },
  brand: {
    type: String,
    trim: true,
    index: true,
  },
  category: {
    id: { type: String, required: true, index: true },
    name: { type: String, required: true },
    path: [String], // ['Electronics', 'Mobile', 'Smartphones']
  },
  tags: [{
    type: String,
    lowercase: true,
    trim: true,
  }],
  images: {
    main: { type: String, required: true },
    gallery: [String],
    thumbnail: String,
  },
  pricing: {
    basePrice: { type: Number, required: true, min: 0 },
    comparePrice: { type: Number, min: 0 }, // original price (for showing discount)
    costPrice: { type: Number, min: 0 }, // internal cost
    currency: { type: String, default: 'THB' },
  },
  inventory: {
    trackQuantity: { type: Boolean, default: true },
    quantity: { type: Number, default: 0, min: 0 },
    lowStockThreshold: { type: Number, default: 10 },
    allowBackorder: { type: Boolean, default: false },
  },
  variants: [VariantSchema],
  attributes: {
    type: Map,
    of: mongoose.Schema.Types.Mixed,
  },
  specifications: [{
    group: String,
    items: [{
      name: String,
      value: String,
    }],
  }],
  shipping: {
    weight: { type: Number },
    dimensions: DimensionSchema,
    freeShipping: { type: Boolean, default: false },
    shippingClass: { type: String },
  },
  seo: {
    title: String,
    description: String,
    keywords: [String],
  },
  status: {
    type: String,
    enum: ['draft', 'active', 'inactive', 'archived'],
    default: 'draft',
    index: true,
  },
  visibility: {
    type: String,
    enum: ['public', 'private', 'hidden'],
    default: 'public',
    index: true,
  },
  featuredAt: Date,
  publishedAt: Date,
  sellerId: {
    type: String,
    required: true,
    index: true,
  },
  reviewSummary: ReviewSummarySchema,
  salesCount: { type: Number, default: 0 },
  viewCount: { type: Number, default: 0 },
  metadata: { type: Map, of: mongoose.Schema.Types.Mixed },
}, {
  timestamps: true,
  toJSON: {
    virtuals: true,
    transform: (doc, ret) => {
      ret.id = ret._id;
      delete ret._id;
      delete ret.__v;
      delete ret.pricing.costPrice; // don't expose internal cost
      return ret;
    },
  },
});

// Virtual fields
ProductSchema.virtual('isOnSale').get(function() {
  return this.pricing.comparePrice > this.pricing.basePrice;
});

ProductSchema.virtual('discountPercent').get(function() {
  if (!this.isOnSale) return 0;
  return Math.round((1 - this.pricing.basePrice / this.pricing.comparePrice) * 100);
});

ProductSchema.virtual('inStock').get(function() {
  if (!this.inventory.trackQuantity) return true;
  return this.inventory.quantity > 0 || this.inventory.allowBackorder;
});

// Indexes
ProductSchema.index({ name: 'text', description: 'text', tags: 'text' });
ProductSchema.index({ 'category.id': 1, status: 1 });
ProductSchema.index({ brand: 1, status: 1 });
ProductSchema.index({ 'pricing.basePrice': 1 });
ProductSchema.index({ status: 1, visibility: 1, publishedAt: -1 });
ProductSchema.index({ sellerId: 1, status: 1 });
ProductSchema.index({ salesCount: -1 });
ProductSchema.index({ 'reviewSummary.averageRating': -1 });

// Pre-save middleware
ProductSchema.pre('save', function(next) {
  if (this.isModified('name') && !this.slug) {
    this.slug = this.name
      .toLowerCase()
      .replace(/[^\w\s-]/g, '')
      .replace(/[\s_-]+/g, '-')
      .replace(/^-+|-+$/g, '');
  }
  next();
});

// Static methods
ProductSchema.statics.findByCategory = function(categoryId, options = {}) {
  const {
    page = 1,
    limit = 20,
    sort = '-createdAt',
    minPrice,
    maxPrice,
    inStock,
  } = options;

  const query = {
    'category.id': categoryId,
    status: 'active',
    visibility: 'public',
  };

  if (minPrice !== undefined || maxPrice !== undefined) {
    query['pricing.basePrice'] = {};
    if (minPrice !== undefined) query['pricing.basePrice'].$gte = minPrice;
    if (maxPrice !== undefined) query['pricing.basePrice'].$lte = maxPrice;
  }

  if (inStock) {
    query['inventory.quantity'] = { $gt: 0 };
  }

  return this.find(query)
    .sort(sort)
    .skip((page - 1) * limit)
    .limit(limit);
};

ProductSchema.statics.search = function(searchText, options = {}) {
  const { page = 1, limit = 20 } = options;

  return this.find(
    { $text: { $search: searchText }, status: 'active', visibility: 'public' },
    { score: { $meta: 'textScore' } }
  )
    .sort({ score: { $meta: 'textScore' } })
    .skip((page - 1) * limit)
    .limit(limit);
};

// Instance methods
ProductSchema.methods.updateReviewSummary = async function(newRating, operation = 'add') {
  const summary = this.reviewSummary;

  if (operation === 'add') {
    const newTotal = summary.totalCount + 1;
    const newAvg = (summary.averageRating * summary.totalCount + newRating) / newTotal;
    summary.totalCount = newTotal;
    summary.averageRating = Math.round(newAvg * 10) / 10;
    summary.ratingDistribution[newRating] += 1;
  } else if (operation === 'remove') {
    if (summary.totalCount <= 1) {
      summary.totalCount = 0;
      summary.averageRating = 0;
      summary.ratingDistribution[newRating] = Math.max(0, summary.ratingDistribution[newRating] - 1);
    } else {
      const newTotal = summary.totalCount - 1;
      const newAvg = (summary.averageRating * summary.totalCount - newRating) / newTotal;
      summary.totalCount = newTotal;
      summary.averageRating = Math.round(newAvg * 10) / 10;
      summary.ratingDistribution[newRating] = Math.max(0, summary.ratingDistribution[newRating] - 1);
    }
  }

  return this.save();
};

module.exports = mongoose.model('Product', ProductSchema);
```

### Product Repository

```javascript
// src/repositories/ProductRepository.js
const Product = require('../models/Product');
const logger = require('../utils/logger');

class ProductRepository {
  async findAll({ page = 1, limit = 20, sort = '-createdAt', ...filters } = {}) {
    const query = this.buildQuery(filters);

    const [products, total] = await Promise.all([
      Product.find(query)
        .sort(sort)
        .skip((page - 1) * limit)
        .limit(limit)
        .lean(),
      Product.countDocuments(query),
    ]);

    return {
      products,
      pagination: {
        page,
        limit,
        total,
        pages: Math.ceil(total / limit),
        hasNext: page < Math.ceil(total / limit),
        hasPrev: page > 1,
      },
    };
  }

  buildQuery(filters) {
    const query = { status: 'active', visibility: 'public' };

    if (filters.categoryId) query['category.id'] = filters.categoryId;
    if (filters.brand) query.brand = filters.brand;
    if (filters.sellerId) query.sellerId = filters.sellerId;
    if (filters.tags) query.tags = { $in: Array.isArray(filters.tags) ? filters.tags : [filters.tags] };
    if (filters.inStock) query['inventory.quantity'] = { $gt: 0 };

    if (filters.minPrice !== undefined || filters.maxPrice !== undefined) {
      query['pricing.basePrice'] = {};
      if (filters.minPrice !== undefined) query['pricing.basePrice'].$gte = Number(filters.minPrice);
      if (filters.maxPrice !== undefined) query['pricing.basePrice'].$lte = Number(filters.maxPrice);
    }

    if (filters.minRating !== undefined) {
      query['reviewSummary.averageRating'] = { $gte: Number(filters.minRating) };
    }

    if (filters.status) query.status = filters.status; // for admin

    return query;
  }

  async findById(id) {
    return Product.findById(id);
  }

  async findBySlug(slug) {
    return Product.findOne({ slug, status: 'active', visibility: 'public' });
  }

  async findBySku(sku) {
    return Product.findOne({ sku: sku.toUpperCase() });
  }

  async search(searchText, options = {}) {
    const { page = 1, limit = 20 } = options;

    const [products, total] = await Promise.all([
      Product.find(
        { $text: { $search: searchText }, status: 'active', visibility: 'public' },
        { score: { $meta: 'textScore' } }
      )
        .sort({ score: { $meta: 'textScore' } })
        .skip((page - 1) * limit)
        .limit(limit)
        .lean(),
      Product.countDocuments({ $text: { $search: searchText }, status: 'active' }),
    ]);

    return { products, total, page, limit };
  }

  async create(data) {
    const product = new Product(data);
    return product.save();
  }

  async update(id, data) {
    return Product.findByIdAndUpdate(
      id,
      { $set: data },
      { new: true, runValidators: true }
    );
  }

  async updateInventory(id, quantity, operation = 'set') {
    let update;
    switch (operation) {
      case 'increment':
        update = { $inc: { 'inventory.quantity': quantity } };
        break;
      case 'decrement':
        update = { $inc: { 'inventory.quantity': -quantity } };
        break;
      default:
        update = { $set: { 'inventory.quantity': quantity } };
    }

    return Product.findByIdAndUpdate(id, update, {
      new: true,
      runValidators: true,
    });
  }

  async decrementStock(id, quantity) {
    // Atomic decrement with minimum check (avoid negative stock)
    return Product.findOneAndUpdate(
      {
        _id: id,
        'inventory.quantity': { $gte: quantity },
      },
      {
        $inc: {
          'inventory.quantity': -quantity,
          salesCount: quantity,
        },
      },
      { new: true }
    );
  }

  async delete(id) {
    return Product.findByIdAndUpdate(
      id,
      { status: 'archived' },
      { new: true }
    );
  }

  async getRecommendations(userId, limit = 6) {
    // Simple recommendation - featured products with high ratings
    return Product.find({
      status: 'active',
      visibility: 'public',
      'reviewSummary.averageRating': { $gte: 4 },
    })
      .sort({ salesCount: -1, 'reviewSummary.averageRating': -1 })
      .limit(limit)
      .lean();
  }

  async getTopProducts(limit = 10) {
    return Product.find({ status: 'active' })
      .sort({ salesCount: -1 })
      .limit(limit)
      .lean();
  }
}

module.exports = new ProductRepository();
```

---

## 11.5 Cross-Service Data Consistency

### ปัญหาที่ท้าทายที่สุดของ Database per Service

เมื่อ Order Service สร้าง order ใหม่:
1. ต้องตรวจสอบว่า user มีอยู่จริง (User Service)
2. ต้องตรวจสอบว่า product มีสต็อก (Product Service)
3. ต้องลด stock ของ product (Product Service)
4. ต้องสร้าง order record (Order Service)
5. ต้องส่ง notification ไป User (Notification Service)

**ถ้า step 3 สำเร็จแต่ step 4 fail → stock ลดแล้วแต่ order ไม่ถูกสร้าง!**

### แนวทางแก้ไข

#### 1. Saga Pattern (Choreography)

```javascript
// order-service/src/sagas/createOrderSaga.js

const EventEmitter = require('events');
const logger = require('../utils/logger');

class CreateOrderSaga {
  constructor({ userClient, productClient, orderRepo, eventBus }) {
    this.userClient = userClient;
    this.productClient = productClient;
    this.orderRepo = orderRepo;
    this.eventBus = eventBus;
  }

  async execute(orderData) {
    const saga = {
      id: `saga-${Date.now()}`,
      orderId: null,
      stockReserved: [],
      status: 'running',
    };

    try {
      // Step 1: Validate user
      logger.info('Saga step 1: Validating user', { sagaId: saga.id });
      const user = await this.userClient.getUser(orderData.userId);
      if (!user) throw new Error('User not found');

      // Step 2: Reserve stock for each item
      logger.info('Saga step 2: Reserving stock', { sagaId: saga.id });
      for (const item of orderData.items) {
        const reserved = await this.productClient.reserveStock(item.productId, item.quantity);
        if (!reserved) {
          throw new Error(`Insufficient stock for product ${item.productId}`);
        }
        saga.stockReserved.push({ productId: item.productId, quantity: item.quantity });
      }

      // Step 3: Calculate total
      const total = await this.calculateTotal(orderData.items);

      // Step 4: Create order
      logger.info('Saga step 4: Creating order', { sagaId: saga.id });
      const order = await this.orderRepo.create({
        ...orderData,
        total,
        status: 'confirmed',
        sagaId: saga.id,
      });
      saga.orderId = order.id;

      // Step 5: Confirm stock reservation
      logger.info('Saga step 5: Confirming stock', { sagaId: saga.id });
      for (const item of orderData.items) {
        await this.productClient.confirmStockReservation(item.productId, item.quantity);
      }

      // Step 6: Publish order created event
      this.eventBus.publish('ORDER_CREATED', {
        orderId: order.id,
        userId: orderData.userId,
        items: orderData.items,
        total,
      });

      saga.status = 'completed';
      logger.info('Saga completed', { sagaId: saga.id, orderId: order.id });

      return order;

    } catch (error) {
      saga.status = 'failed';
      logger.error('Saga failed, running compensation', {
        sagaId: saga.id,
        error: error.message,
      });

      // COMPENSATION: Run in reverse order
      await this.compensate(saga, error);

      throw error;
    }
  }

  async compensate(saga, originalError) {
    const compensations = [];

    // Compensate stock reservations
    for (const { productId, quantity } of saga.stockReserved) {
      compensations.push(
        this.productClient.releaseStockReservation(productId, quantity)
          .catch(err => logger.error('Failed to release stock', { productId, err: err.message }))
      );
    }

    // Compensate order if created
    if (saga.orderId) {
      compensations.push(
        this.orderRepo.cancel(saga.orderId, 'saga_compensation')
          .catch(err => logger.error('Failed to cancel order', { err: err.message }))
      );
    }

    await Promise.allSettled(compensations);
  }

  async calculateTotal(items) {
    let total = 0;
    for (const item of items) {
      const product = await this.productClient.getProduct(item.productId);
      total += product.pricing.basePrice * item.quantity;
    }
    return total;
  }
}

module.exports = CreateOrderSaga;
```

#### 2. Outbox Pattern (Transactional Outbox)

```javascript
// src/repositories/OrderRepository.js
const { Pool } = require('pg');
const config = require('../config');

class OrderRepository {
  constructor() {
    this.pool = new Pool({ connectionString: config.database.url });
  }

  // Create order and event in same transaction
  async createWithEvent(orderData, event) {
    const client = await this.pool.connect();

    try {
      await client.query('BEGIN');

      // Insert order
      const orderResult = await client.query(
        `INSERT INTO orders (user_id, items, total, status, shipping_address)
         VALUES ($1, $2, $3, $4, $5)
         RETURNING *`,
        [
          orderData.userId,
          JSON.stringify(orderData.items),
          orderData.total,
          'pending',
          JSON.stringify(orderData.shippingAddress),
        ]
      );

      const order = orderResult.rows[0];

      // Insert event to outbox (same transaction!)
      await client.query(
        `INSERT INTO outbox_events (aggregate_type, aggregate_id, event_type, payload, created_at)
         VALUES ($1, $2, $3, $4, NOW())`,
        [
          'Order',
          order.id,
          event.type,
          JSON.stringify({ ...event.payload, orderId: order.id }),
        ]
      );

      await client.query('COMMIT');

      return order;
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }
}
```

#### 3. Outbox Processor (Polling Publisher)

```javascript
// src/workers/outboxProcessor.js
const { Pool } = require('pg');
const config = require('../config');
const logger = require('../utils/logger');
const messageBroker = require('../utils/messageBroker');

class OutboxProcessor {
  constructor() {
    this.pool = new Pool({ connectionString: config.database.url });
    this.running = false;
    this.intervalMs = 1000; // Process every 1 second
  }

  async start() {
    this.running = true;
    logger.info('Outbox processor started');

    while (this.running) {
      try {
        await this.processEvents();
      } catch (error) {
        logger.error('Outbox processing error', { error: error.message });
      }

      await new Promise(resolve => setTimeout(resolve, this.intervalMs));
    }
  }

  async processEvents() {
    const client = await this.pool.connect();

    try {
      // Lock and get unprocessed events (SELECT FOR UPDATE SKIP LOCKED for concurrent workers)
      const { rows: events } = await client.query(`
        SELECT * FROM outbox_events
        WHERE processed_at IS NULL
        ORDER BY created_at ASC
        LIMIT 100
        FOR UPDATE SKIP LOCKED
      `);

      if (events.length === 0) return;

      for (const event of events) {
        try {
          // Publish to message broker
          await messageBroker.publish(event.event_type, JSON.parse(event.payload));

          // Mark as processed
          await client.query(
            'UPDATE outbox_events SET processed_at = NOW() WHERE id = $1',
            [event.id]
          );

          logger.debug('Outbox event published', {
            eventId: event.id,
            type: event.event_type,
          });
        } catch (error) {
          logger.error('Failed to publish outbox event', {
            eventId: event.id,
            error: error.message,
          });

          // Increment retry count
          await client.query(
            `UPDATE outbox_events
             SET retry_count = retry_count + 1,
                 last_error = $2,
                 next_retry_at = NOW() + INTERVAL '${Math.pow(2, event.retry_count)} minutes'
             WHERE id = $1`,
            [event.id, error.message]
          );
        }
      }
    } finally {
      client.release();
    }
  }

  stop() {
    this.running = false;
    logger.info('Outbox processor stopped');
  }
}

module.exports = new OutboxProcessor();
```

---

## 11.6 Data Synchronization Patterns

### CQRS - Command Query Responsibility Segregation (Introduction)

```javascript
// product-service/src/cqrs/commands/CreateProductCommand.js
class CreateProductCommand {
  constructor({ name, sku, price, categoryId, sellerId }) {
    this.name = name;
    this.sku = sku;
    this.price = price;
    this.categoryId = categoryId;
    this.sellerId = sellerId;
    this.timestamp = new Date();
  }
}

// product-service/src/cqrs/handlers/CreateProductHandler.js
class CreateProductHandler {
  constructor({ productRepository, eventBus, searchIndexer }) {
    this.productRepository = productRepository;
    this.eventBus = eventBus;
    this.searchIndexer = searchIndexer;
  }

  async handle(command) {
    // Write to main database
    const product = await this.productRepository.create({
      name: command.name,
      sku: command.sku,
      pricing: { basePrice: command.price },
      'category.id': command.categoryId,
      sellerId: command.sellerId,
    });

    // Publish event for read model update
    await this.eventBus.publish('PRODUCT_CREATED', {
      id: product.id,
      name: product.name,
      sku: product.sku,
      price: command.price,
      categoryId: command.categoryId,
    });

    return product;
  }
}

// search-service/src/handlers/ProductCreatedHandler.js
class ProductCreatedHandler {
  constructor({ elasticsearchClient }) {
    this.es = elasticsearchClient;
  }

  async handle(event) {
    // Update search index (read model)
    await this.es.index({
      index: 'products',
      id: event.id,
      document: {
        id: event.id,
        name: event.name,
        sku: event.sku,
        price: event.price,
        categoryId: event.categoryId,
        createdAt: new Date(),
      },
    });
  }
}
```

### Read Models with Redis

```javascript
// order-service/src/readModels/OrderSummaryReadModel.js
const Redis = require('ioredis');
const config = require('../config');

class OrderSummaryReadModel {
  constructor() {
    this.redis = new Redis(config.redis);
    this.TTL = 3600; // 1 hour
  }

  getKey(userId) {
    return `order:summary:${userId}`;
  }

  async get(userId) {
    const cached = await this.redis.get(this.getKey(userId));
    return cached ? JSON.parse(cached) : null;
  }

  async set(userId, summary) {
    await this.redis.setex(
      this.getKey(userId),
      this.TTL,
      JSON.stringify(summary)
    );
  }

  async invalidate(userId) {
    await this.redis.del(this.getKey(userId));
  }

  async handleOrderCreated(event) {
    // Invalidate user's order summary cache when order created
    await this.invalidate(event.userId);
  }

  async handleOrderStatusChanged(event) {
    await this.invalidate(event.userId);
  }
}

module.exports = new OrderSummaryReadModel();
```

---

## 11.7 Database Migrations Strategy

### Zero-Downtime Migrations

```javascript
// Example: Adding a column safely (zero downtime)

// Phase 1: Add nullable column (backward compatible)
// Migration: 002_add_phone_to_users.sql
ALTER TABLE users ADD COLUMN IF NOT EXISTS phone VARCHAR(20);
-- No constraints yet - old service versions still work

// Phase 2: Deploy new service version that writes to new column
// Wait for all old instances to drain

// Phase 3: Backfill existing data
UPDATE users SET phone = metadata->>'phone'
WHERE phone IS NULL AND metadata->>'phone' IS NOT NULL;

// Phase 4: Add NOT NULL constraint with default (or leave nullable)
-- ALTER TABLE users ALTER COLUMN phone SET NOT NULL;
-- Only do this after all rows have data
```

### DB Migration with Node.js (db-migrate)

```javascript
// package.json
{
  "scripts": {
    "migrate": "db-migrate up",
    "migrate:down": "db-migrate down",
    "migrate:create": "db-migrate create"
  }
}

// migrations/20240101-create-orders.js
'use strict';

var dbm;
var type;
var seed;

exports.setup = function(options, seedLink) {
  dbm = options.dbmigrate;
  type = dbm.dataType;
  seed = seedLink;
};

exports.up = function(db) {
  return db.runSql(`
    CREATE TABLE IF NOT EXISTS orders (
      id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
      user_id UUID NOT NULL,
      status VARCHAR(50) NOT NULL DEFAULT 'pending',
      items JSONB NOT NULL DEFAULT '[]',
      subtotal DECIMAL(10,2) NOT NULL DEFAULT 0,
      tax DECIMAL(10,2) NOT NULL DEFAULT 0,
      shipping_fee DECIMAL(10,2) NOT NULL DEFAULT 0,
      total DECIMAL(10,2) NOT NULL DEFAULT 0,
      shipping_address JSONB,
      notes TEXT,
      created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
      updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
    );
    
    CREATE INDEX idx_orders_user_id ON orders(user_id);
    CREATE INDEX idx_orders_status ON orders(status);
    CREATE INDEX idx_orders_created_at ON orders(created_at DESC);
  `);
};

exports.down = function(db) {
  return db.runSql('DROP TABLE IF EXISTS orders CASCADE;');
};
```

---

## 11.8 Workshop: Full E-Commerce Database Setup

### Docker Compose for Databases

```yaml
# docker-compose.databases.yml
version: '3.8'

services:
  # PostgreSQL for Users
  postgres-users:
    image: postgres:15-alpine
    container_name: postgres-users
    environment:
      POSTGRES_DB: users_db
      POSTGRES_USER: users_service
      POSTGRES_PASSWORD: ${POSTGRES_USERS_PASSWORD:-users123}
    ports:
      - "5432:5432"
    volumes:
      - postgres-users-data:/var/lib/postgresql/data
      - ./user-service/migrations:/docker-entrypoint-initdb.d
    networks:
      - databases
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U users_service -d users_db"]
      interval: 5s
      timeout: 5s
      retries: 10

  # PostgreSQL for Orders
  postgres-orders:
    image: postgres:15-alpine
    container_name: postgres-orders
    environment:
      POSTGRES_DB: orders_db
      POSTGRES_USER: orders_service
      POSTGRES_PASSWORD: ${POSTGRES_ORDERS_PASSWORD:-orders123}
    ports:
      - "5433:5432"
    volumes:
      - postgres-orders-data:/var/lib/postgresql/data
      - ./order-service/migrations:/docker-entrypoint-initdb.d
    networks:
      - databases
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U orders_service -d orders_db"]
      interval: 5s
      timeout: 5s
      retries: 10

  # PostgreSQL for Payments
  postgres-payments:
    image: postgres:15-alpine
    container_name: postgres-payments
    environment:
      POSTGRES_DB: payments_db
      POSTGRES_USER: payments_service
      POSTGRES_PASSWORD: ${POSTGRES_PAYMENTS_PASSWORD:-payments123}
    ports:
      - "5434:5432"
    volumes:
      - postgres-payments-data:/var/lib/postgresql/data
    networks:
      - databases
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U payments_service -d payments_db"]
      interval: 5s
      timeout: 5s
      retries: 10

  # MongoDB for Products
  mongodb-products:
    image: mongo:7
    container_name: mongodb-products
    environment:
      MONGO_INITDB_ROOT_USERNAME: products_service
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_PRODUCTS_PASSWORD:-products123}
      MONGO_INITDB_DATABASE: products_db
    ports:
      - "27017:27017"
    volumes:
      - mongodb-products-data:/data/db
      - ./product-service/mongo-init:/docker-entrypoint-initdb.d
    networks:
      - databases
    healthcheck:
      test: echo 'db.runCommand("ping").ok' | mongosh -u products_service -p products123 localhost:27017/products_db --quiet
      interval: 5s
      timeout: 5s
      retries: 10

  # Redis for Cache and Sessions
  redis:
    image: redis:7-alpine
    container_name: redis
    command: >
      redis-server
      --requirepass ${REDIS_PASSWORD:-redis123}
      --save 60 1
      --loglevel warning
      --maxmemory 256mb
      --maxmemory-policy allkeys-lru
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    networks:
      - databases
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD:-redis123}", "ping"]
      interval: 5s
      timeout: 5s
      retries: 10

  # Elasticsearch for Search
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - ES_JAVA_OPTS=-Xms512m -Xmx512m
      - xpack.security.enabled=false
    ports:
      - "9200:9200"
    volumes:
      - elasticsearch-data:/usr/share/elasticsearch/data
    networks:
      - databases
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:9200/_health || exit 1"]
      interval: 10s
      timeout: 10s
      retries: 5

networks:
  databases:
    driver: bridge

volumes:
  postgres-users-data:
  postgres-orders-data:
  postgres-payments-data:
  mongodb-products-data:
  redis-data:
  elasticsearch-data:
```

### Database Connection Manager

```javascript
// shared/src/database/ConnectionManager.js
const { Pool } = require('pg');
const mongoose = require('mongoose');
const Redis = require('ioredis');
const logger = require('../utils/logger');

class ConnectionManager {
  constructor() {
    this.connections = new Map();
  }

  // PostgreSQL
  createPostgresPool(name, config) {
    const pool = new Pool({
      connectionString: config.url,
      max: config.poolSize || 20,
      idleTimeoutMillis: config.idleTimeout || 30000,
      connectionTimeoutMillis: config.connectionTimeout || 5000,
    });

    pool.on('error', (err) => {
      logger.error(`Unexpected PostgreSQL error on idle client [${name}]`, { error: err.message });
    });

    this.connections.set(`postgres:${name}`, pool);
    return pool;
  }

  // MongoDB
  async connectMongo(name, uri, options = {}) {
    const connection = await mongoose.createConnection(uri, {
      maxPoolSize: options.poolSize || 10,
      serverSelectionTimeoutMS: 5000,
      socketTimeoutMS: 45000,
      ...options,
    }).asPromise();

    connection.on('disconnected', () => {
      logger.warn(`MongoDB disconnected [${name}]`);
    });

    connection.on('reconnected', () => {
      logger.info(`MongoDB reconnected [${name}]`);
    });

    this.connections.set(`mongo:${name}`, connection);
    return connection;
  }

  // Redis
  createRedisClient(name, config) {
    const client = new Redis({
      host: config.host,
      port: config.port,
      password: config.password,
      retryStrategy: (times) => {
        const delay = Math.min(times * 50, 2000);
        return delay;
      },
    });

    client.on('error', (err) => {
      logger.error(`Redis error [${name}]`, { error: err.message });
    });

    client.on('connect', () => {
      logger.info(`Redis connected [${name}]`);
    });

    this.connections.set(`redis:${name}`, client);
    return client;
  }

  async closeAll() {
    const closePromises = [];

    for (const [name, conn] of this.connections.entries()) {
      if (name.startsWith('postgres:')) {
        closePromises.push(conn.end());
      } else if (name.startsWith('mongo:')) {
        closePromises.push(conn.close());
      } else if (name.startsWith('redis:')) {
        closePromises.push(conn.quit());
      }
    }

    await Promise.allSettled(closePromises);
    logger.info('All database connections closed');
  }
}

module.exports = new ConnectionManager();
```

---

## สรุป Part 11

ในบทนี้เราได้เรียนรู้:
- **Database per Service** - ทำไมต้องแยก database ต่อ service
- **Database Selection** - เลือก database ที่เหมาะกับ use case แต่ละ service
- **PostgreSQL** - schema design, migrations, indexes
- **MongoDB** - Mongoose schema, indexes, virtual fields, static methods
- **Cross-Service Consistency** - Saga Pattern, Outbox Pattern
- **Read Models** - CQRS intro, Redis caching
- **Zero-Downtime Migrations** - strategy สำหรับ schema changes
- **Docker Compose** - สร้าง full database infrastructure

**Next:** Part 12 - Authentication & JWT Deep Dive

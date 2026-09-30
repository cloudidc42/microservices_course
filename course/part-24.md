# Part 24: Multi-tenancy Architecture

## ภาพรวม

ใน Part นี้เราจะเรียนรู้การออกแบบระบบ Multi-tenant:
- **Tenant Isolation Strategies** - Database per tenant, Schema per tenant, Row-level security
- **Tenant Context** - ส่ง tenant context ผ่าน middleware
- **Data Partitioning** - แยกข้อมูล tenant ได้อย่างปลอดภัย
- **Custom Domains** - แต่ละ tenant มี domain ของตัวเอง
- **Feature Flags** per tenant

---

## 1. Multi-tenancy Strategies

```
Strategy 1: Database per Tenant
Tenant A → Database A
Tenant B → Database B
Tenant C → Database C
✓ Complete isolation
✓ Easy per-tenant backup
✗ Many connections, high cost
✗ Schema changes across all DBs

Strategy 2: Schema per Tenant (PostgreSQL)
One Database:
├── tenant_a.users
├── tenant_a.orders
├── tenant_b.users
└── tenant_b.orders
✓ Good isolation
✓ Easy tenant migration
✗ Complex query routing

Strategy 3: Row-Level Security (RLS)
One Database, One Schema:
users (id, tenant_id, ...)
orders (id, tenant_id, ...)
✓ Simple, cost-effective
✓ Easy to scale
✗ Risk of data leakage if RLS misconfigured
```

---

## 2. PostgreSQL Row-Level Security

### 2.1 Setup RLS

```sql
-- migrations/setup_rls.sql

-- Enable RLS on all tenant tables
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
ALTER TABLE products ENABLE ROW LEVEL SECURITY;

-- Create policy: users can only see their tenant's data
CREATE POLICY tenant_isolation_users ON users
  USING (tenant_id = current_setting('app.current_tenant_id')::UUID);

CREATE POLICY tenant_isolation_orders ON orders
  USING (tenant_id = current_setting('app.current_tenant_id')::UUID);

CREATE POLICY tenant_isolation_products ON products
  USING (tenant_id = current_setting('app.current_tenant_id')::UUID);

-- Super admin bypass policy
CREATE POLICY superadmin_bypass ON users
  USING (current_setting('app.is_superadmin', true) = 'true');

-- Create dedicated app user (not superuser, so RLS applies)
CREATE USER app_user WITH PASSWORD 'strong_password';
GRANT USAGE ON SCHEMA public TO app_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_user;
```

### 2.2 Tenant Context Middleware

```javascript
// middleware/tenant-context.js
const { AsyncLocalStorage } = require('async_hooks');

const tenantContext = new AsyncLocalStorage();

class TenantMiddleware {
  constructor(tenantService) {
    this.tenantService = tenantService;
  }

  // Extract tenant from subdomain, header, or JWT
  middleware() {
    return async (req, res, next) => {
      try {
        const tenant = await this.resolveTenant(req);
        
        if (!tenant) {
          return res.status(401).json({ error: 'Tenant not found' });
        }

        if (tenant.status === 'suspended') {
          return res.status(403).json({
            error: 'Tenant account suspended',
            contact: 'support@saas.com'
          });
        }

        // Store in AsyncLocalStorage (thread-safe context)
        tenantContext.run({ tenant }, () => {
          req.tenant = tenant;
          next();
        });
      } catch (err) {
        next(err);
      }
    };
  }

  async resolveTenant(req) {
    // Method 1: From subdomain (tenant.saas.com)
    const host = req.hostname;
    const subdomain = host.split('.')[0];
    
    if (subdomain && subdomain !== 'www' && subdomain !== 'api') {
      return this.tenantService.findBySubdomain(subdomain);
    }

    // Method 2: From custom domain
    const tenant = await this.tenantService.findByDomain(host);
    if (tenant) return tenant;

    // Method 3: From JWT claim
    if (req.user?.tenantId) {
      return this.tenantService.findById(req.user.tenantId);
    }

    // Method 4: From header (for internal service calls)
    const tenantId = req.headers['x-tenant-id'];
    if (tenantId) {
      return this.tenantService.findById(tenantId);
    }

    return null;
  }
}

// Get current tenant context anywhere in the app
function getCurrentTenant() {
  const store = tenantContext.getStore();
  if (!store?.tenant) {
    throw new Error('No tenant context found');
  }
  return store.tenant;
}

module.exports = { TenantMiddleware, getCurrentTenant, tenantContext };
```

### 2.3 Tenant-aware Database Pool

```javascript
// db/tenant-db.js
const { Pool } = require('pg');
const { getCurrentTenant } = require('../middleware/tenant-context');

class TenantAwarePool {
  constructor(config) {
    this.pool = new Pool(config);
  }

  async query(sql, params) {
    const tenant = getCurrentTenant();
    const client = await this.pool.connect();
    
    try {
      // Set tenant context for RLS
      await client.query(
        `SET app.current_tenant_id = '${tenant.id}'`
      );
      
      const result = await client.query(sql, params);
      return result;
    } finally {
      // Reset context and release connection
      await client.query('RESET app.current_tenant_id');
      client.release();
    }
  }

  async transaction(fn) {
    const tenant = getCurrentTenant();
    const client = await this.pool.connect();
    
    try {
      await client.query('BEGIN');
      await client.query(`SET LOCAL app.current_tenant_id = '${tenant.id}'`);
      
      const result = await fn(client);
      await client.query('COMMIT');
      return result;
    } catch (err) {
      await client.query('ROLLBACK');
      throw err;
    } finally {
      client.release();
    }
  }
}
```

---

## 3. Tenant Management Service

```javascript
// services/tenant-service.js
class TenantService {
  constructor(db, redis) {
    this.db = db;
    this.cache = redis;
  }

  async createTenant(data) {
    const { name, subdomain, plan, adminEmail } = data;

    // Validate subdomain uniqueness
    const existing = await this.db.query(
      'SELECT id FROM tenants WHERE subdomain = $1',
      [subdomain]
    );
    
    if (existing.rows.length > 0) {
      throw new ConflictError('Subdomain already taken');
    }

    // Create tenant
    const result = await this.db.query(
      `INSERT INTO tenants (name, subdomain, plan, status, settings, created_at)
       VALUES ($1, $2, $3, 'active', $4, NOW())
       RETURNING *`,
      [name, subdomain, plan, JSON.stringify(this.getDefaultSettings(plan))]
    );
    
    const tenant = result.rows[0];

    // Create initial admin user
    await this.createTenantAdmin(tenant.id, adminEmail);

    // Setup tenant resources (schemas, storage buckets, etc.)
    await this.provisionTenantResources(tenant);

    return tenant;
  }

  async provisionTenantResources(tenant) {
    // For schema-per-tenant strategy
    // await this.createTenantSchema(tenant.id);
    
    // Create default resources
    await Promise.all([
      this.createDefaultCategories(tenant.id),
      this.setupStorageBucket(tenant.id),
      this.configureEmailSettings(tenant.id)
    ]);
  }

  async findBySubdomain(subdomain) {
    const cacheKey = `tenant:subdomain:${subdomain}`;
    const cached = await this.cache.get(cacheKey);
    
    if (cached) return JSON.parse(cached);
    
    const result = await this.db.query(
      'SELECT * FROM tenants WHERE subdomain = $1 AND deleted_at IS NULL',
      [subdomain]
    );
    
    if (result.rows.length === 0) return null;
    
    const tenant = result.rows[0];
    await this.cache.setex(cacheKey, 300, JSON.stringify(tenant));
    
    return tenant;
  }

  async findByDomain(domain) {
    const result = await this.db.query(
      `SELECT t.* FROM tenants t
       JOIN tenant_domains td ON td.tenant_id = t.id
       WHERE td.domain = $1 AND td.verified = true
       AND t.deleted_at IS NULL`,
      [domain]
    );
    
    return result.rows[0] || null;
  }

  getDefaultSettings(plan) {
    const settings = {
      free: {
        maxUsers: 5,
        maxStorage: 1024,  // MB
        features: ['basic_analytics'],
        apiRateLimit: 1000  // requests/hour
      },
      starter: {
        maxUsers: 20,
        maxStorage: 10240,
        features: ['basic_analytics', 'custom_branding'],
        apiRateLimit: 10000
      },
      pro: {
        maxUsers: 100,
        maxStorage: 102400,
        features: ['advanced_analytics', 'custom_branding', 'api_access', 'sso'],
        apiRateLimit: 100000
      },
      enterprise: {
        maxUsers: -1,  // Unlimited
        maxStorage: -1,
        features: ['all'],
        apiRateLimit: -1,
        sla: '99.9%'
      }
    };
    
    return settings[plan] || settings.free;
  }
}
```

---

## 4. Feature Flags per Tenant

```javascript
// features/feature-flags.js
class FeatureFlagService {
  constructor(db, redis) {
    this.db = db;
    this.cache = redis;
  }

  async isEnabled(featureName, tenantId, userId = null) {
    const cacheKey = `feature:${featureName}:${tenantId}:${userId || 'global'}`;
    const cached = await this.cache.get(cacheKey);
    
    if (cached !== null) {
      return cached === 'true';
    }

    const enabled = await this.checkFeature(featureName, tenantId, userId);
    
    await this.cache.setex(cacheKey, 60, enabled.toString());
    
    return enabled;
  }

  async checkFeature(featureName, tenantId, userId) {
    // Check in order: user-specific → tenant-specific → plan-based → global
    
    // 1. User-specific override
    if (userId) {
      const userOverride = await this.getUserOverride(featureName, userId);
      if (userOverride !== null) return userOverride;
    }

    // 2. Tenant-specific override
    const tenantOverride = await this.getTenantOverride(featureName, tenantId);
    if (tenantOverride !== null) return tenantOverride;

    // 3. Plan-based feature
    const tenant = await this.getTenant(tenantId);
    const planFeatures = tenant.settings.features;
    
    if (planFeatures.includes('all') || planFeatures.includes(featureName)) {
      return true;
    }

    // 4. Global rollout
    const globalFlag = await this.getGlobalFlag(featureName);
    if (globalFlag) {
      return this.checkRolloutPercentage(globalFlag, tenantId);
    }

    return false;
  }

  checkRolloutPercentage(flag, tenantId) {
    if (flag.rolloutPercentage >= 100) return true;
    if (flag.rolloutPercentage <= 0) return false;
    
    // Consistent hashing for consistent rollout
    const hash = this.hashTenantId(tenantId);
    return hash % 100 < flag.rolloutPercentage;
  }

  hashTenantId(tenantId) {
    const crypto = require('crypto');
    const hash = crypto.createHash('md5').update(tenantId).digest('hex');
    return parseInt(hash.substring(0, 8), 16) % 100;
  }
}

// Middleware ตรวจสอบ feature
function requireFeature(featureName) {
  return async (req, res, next) => {
    const tenant = req.tenant;
    const userId = req.user?.id;
    
    const featureService = req.app.get('featureFlagService');
    const enabled = await featureService.isEnabled(featureName, tenant.id, userId);
    
    if (!enabled) {
      return res.status(403).json({
        error: 'Feature not available',
        feature: featureName,
        upgradeUrl: `/pricing?feature=${featureName}`
      });
    }
    
    next();
  };
}

// ใช้งาน
router.post('/reports/advanced',
  authenticate,
  requireFeature('advanced_analytics'),
  generateAdvancedReport
);
```

---

## 5. Custom Domain Management

```javascript
// services/custom-domain-service.js
const dns = require('dns').promises;

class CustomDomainService {
  constructor(db, certManager) {
    this.db = db;
    this.certManager = certManager;
  }

  async addDomain(tenantId, domain) {
    // Validate domain format
    if (!/^[a-z0-9]+([\-\.]{1}[a-z0-9]+)*\.[a-z]{2,}$/.test(domain)) {
      throw new ValidationError('Invalid domain format');
    }

    // Check if domain is already used
    const existing = await this.db.query(
      'SELECT id FROM tenant_domains WHERE domain = $1',
      [domain]
    );
    
    if (existing.rows.length > 0) {
      throw new ConflictError('Domain already in use');
    }

    // Generate verification token
    const verificationToken = crypto.randomBytes(32).toString('hex');

    await this.db.query(
      `INSERT INTO tenant_domains (tenant_id, domain, verification_token, verified, created_at)
       VALUES ($1, $2, $3, false, NOW())`,
      [tenantId, domain, verificationToken]
    );

    return {
      domain,
      verificationToken,
      instructions: {
        type: 'TXT',
        host: `_saas-verify.${domain}`,
        value: `saas-verify=${verificationToken}`,
        message: 'Add this TXT record to verify domain ownership'
      }
    };
  }

  async verifyDomain(tenantId, domain) {
    const domainRecord = await this.db.query(
      'SELECT * FROM tenant_domains WHERE tenant_id = $1 AND domain = $2',
      [tenantId, domain]
    );
    
    if (domainRecord.rows.length === 0) {
      throw new NotFoundError('Domain not found');
    }
    
    const record = domainRecord.rows[0];
    const expectedValue = `saas-verify=${record.verification_token}`;

    try {
      // Check DNS TXT record
      const txtRecords = await dns.resolveTxt(`_saas-verify.${domain}`);
      const verified = txtRecords.flat().includes(expectedValue);
      
      if (!verified) {
        return { verified: false, message: 'TXT record not found or incorrect' };
      }

      // Mark as verified
      await this.db.query(
        'UPDATE tenant_domains SET verified = true, verified_at = NOW() WHERE id = $1',
        [record.id]
      );

      // Provision SSL certificate
      await this.certManager.requestCertificate(domain);

      return { verified: true };
    } catch (err) {
      return { verified: false, message: 'DNS lookup failed' };
    }
  }
}
```

---

## 6. Tenant Billing & Usage Tracking

```javascript
// services/usage-tracking.js
class UsageTracker {
  constructor(redis, db) {
    this.redis = redis;
    this.db = db;
  }

  // Track API usage per tenant
  async trackAPICall(tenantId, endpoint) {
    const hour = Math.floor(Date.now() / 3600000);
    const keys = [
      `usage:api:${tenantId}:${hour}`,
      `usage:api:${tenantId}:total`
    ];

    const pipeline = this.redis.pipeline();
    for (const key of keys) {
      pipeline.incr(key);
    }
    pipeline.expire(keys[0], 3600 * 25); // Keep 25 hours
    
    const results = await pipeline.exec();
    const hourlyCount = results[0][1];

    // Check rate limit
    const tenant = await this.getTenant(tenantId);
    const limit = tenant.settings.apiRateLimit;
    
    if (limit > 0 && hourlyCount > limit) {
      throw new RateLimitError(`API rate limit exceeded: ${hourlyCount}/${limit} per hour`);
    }

    return { hourlyCount, limit };
  }

  async getUsageSummary(tenantId, period = 'month') {
    const now = new Date();
    let startDate;
    
    if (period === 'month') {
      startDate = new Date(now.getFullYear(), now.getMonth(), 1);
    } else if (period === 'week') {
      startDate = new Date(now - 7 * 24 * 3600000);
    }

    const result = await this.db.query(
      `SELECT 
        SUM(api_calls) as total_api_calls,
        SUM(storage_bytes) as total_storage,
        COUNT(DISTINCT user_id) as active_users
       FROM usage_events
       WHERE tenant_id = $1 AND created_at >= $2`,
      [tenantId, startDate]
    );

    return result.rows[0];
  }

  async checkStorageLimit(tenantId, additionalBytes) {
    const tenant = await this.getTenant(tenantId);
    const maxStorage = tenant.settings.maxStorage;
    
    if (maxStorage < 0) return; // Unlimited
    
    const currentUsage = await this.getCurrentStorageUsage(tenantId);
    const totalMB = (currentUsage + additionalBytes) / 1024 / 1024;
    
    if (totalMB > maxStorage) {
      throw new StorageLimitError(
        `Storage limit exceeded: ${totalMB.toFixed(0)}MB / ${maxStorage}MB`
      );
    }
  }
}
```

---

## Workshop: Multi-tenant SaaS Setup

```bash
# Database setup
psql -c "
CREATE TABLE tenants (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  subdomain VARCHAR(100) UNIQUE NOT NULL,
  plan VARCHAR(50) NOT NULL DEFAULT 'free',
  status VARCHAR(50) NOT NULL DEFAULT 'active',
  settings JSONB DEFAULT '{}',
  created_at TIMESTAMPTZ DEFAULT NOW(),
  deleted_at TIMESTAMPTZ
);

CREATE TABLE tenant_domains (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  tenant_id UUID REFERENCES tenants(id),
  domain VARCHAR(255) NOT NULL,
  verification_token VARCHAR(100),
  verified BOOLEAN DEFAULT false,
  verified_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Add tenant_id to all data tables
ALTER TABLE users ADD COLUMN tenant_id UUID REFERENCES tenants(id);
ALTER TABLE orders ADD COLUMN tenant_id UUID REFERENCES tenants(id);
ALTER TABLE products ADD COLUMN tenant_id UUID REFERENCES tenants(id);

-- Row Level Security
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
ALTER TABLE products ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON users
  USING (tenant_id = current_setting('app.current_tenant_id')::UUID);

CREATE POLICY tenant_isolation ON orders
  USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
"

# Test tenant isolation
psql -c "
SET app.current_tenant_id = 'tenant-a-uuid';
SELECT count(*) FROM users; -- Only tenant A's users

SET app.current_tenant_id = 'tenant-b-uuid';
SELECT count(*) FROM users; -- Only tenant B's users
"
```

---

## สรุป

| Strategy | Use Case | Isolation | Cost |
|----------|----------|-----------|------|
| DB per tenant | Enterprise, regulated | Highest | High |
| Schema per tenant | Mid-market, < 1000 tenants | High | Medium |
| RLS | SMB, > 1000 tenants | Good | Low |

**Key Points:**
- เสมอ pass tenant context ผ่าน `AsyncLocalStorage`
- ใช้ RLS เป็น safety net แม้จะมี application-level checks
- Cache tenant lookup เพื่อ performance
- Monitor per-tenant usage สำหรับ billing

**Next:** Part 25 - GraphQL Federation for Microservices

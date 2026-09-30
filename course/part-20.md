# Part 20: Performance Optimization & Advanced Caching

## ภาพรวม

ใน Part นี้เราจะเรียนรู้การ optimize ระบบ Microservices:
- **Caching Strategies** - Cache-aside, Write-through, Write-behind
- **Redis Advanced** - Lua scripting, Pipeline, Cluster
- **Database Optimization** - Query optimization, Connection pooling, Read replicas
- **API Performance** - Response compression, HTTP/2, Pagination
- **CDN & Edge Caching** - Cloudflare, Fastly patterns
- **Node.js Performance** - Profiling, Memory leaks, Worker threads

---

## 1. Caching Strategies

### 1.1 Cache-Aside Pattern (Lazy Loading)

```javascript
// cache/strategies/cache-aside.js
class CacheAsideStrategy {
  constructor(redis, db, options = {}) {
    this.redis = redis;
    this.db = db;
    this.ttl = options.ttl || 3600; // 1 hour default
    this.prefix = options.prefix || 'cache:';
    this.nullTTL = options.nullTTL || 60; // cache null for 60s (prevent cache penetration)
  }

  async get(key, fetchFn) {
    const cacheKey = `${this.prefix}${key}`;
    
    // 1. Check cache
    const cached = await this.redis.get(cacheKey);
    
    if (cached !== null) {
      const value = JSON.parse(cached);
      
      // null sentinel = ไม่มีข้อมูลใน DB
      if (value === '__NULL__') return null;
      
      return value;
    }

    // 2. Fetch from DB
    const data = await fetchFn();
    
    // 3. Cache the result
    if (data === null) {
      // Cache null ด้วย TTL สั้นกว่า
      await this.redis.setex(cacheKey, this.nullTTL, JSON.stringify('__NULL__'));
    } else {
      await this.redis.setex(cacheKey, this.ttl, JSON.stringify(data));
    }
    
    return data;
  }

  async invalidate(key) {
    const cacheKey = `${this.prefix}${key}`;
    await this.redis.del(cacheKey);
  }

  async invalidatePattern(pattern) {
    // ระวัง: KEYS command ช้าใน production
    // ใช้ SCAN แทน
    const keys = await this.scanKeys(`${this.prefix}${pattern}`);
    if (keys.length > 0) {
      await this.redis.del(...keys);
    }
  }

  async scanKeys(pattern) {
    const keys = [];
    let cursor = '0';
    
    do {
      const [nextCursor, batchKeys] = await this.redis.scan(
        cursor,
        'MATCH', pattern,
        'COUNT', 100
      );
      cursor = nextCursor;
      keys.push(...batchKeys);
    } while (cursor !== '0');
    
    return keys;
  }
}

// ใช้งาน
class UserService {
  constructor(redis, userRepository) {
    this.cache = new CacheAsideStrategy(redis, userRepository, {
      prefix: 'user:',
      ttl: 1800 // 30 minutes
    });
    this.userRepository = userRepository;
  }

  async getUserById(userId) {
    return this.cache.get(
      `id:${userId}`,
      () => this.userRepository.findById(userId)
    );
  }

  async updateUser(userId, data) {
    const updated = await this.userRepository.update(userId, data);
    
    // Invalidate cache
    await this.cache.invalidate(`id:${userId}`);
    await this.cache.invalidate(`email:${updated.email}`);
    
    return updated;
  }
}
```

### 1.2 Write-Through Pattern

```javascript
// cache/strategies/write-through.js
class WriteThroughStrategy {
  constructor(redis, db, options = {}) {
    this.redis = redis;
    this.db = db;
    this.ttl = options.ttl || 3600;
    this.prefix = options.prefix || 'wt:';
  }

  async write(key, data, writeFn) {
    const cacheKey = `${this.prefix}${key}`;
    
    // Write to DB and cache simultaneously
    const [dbResult] = await Promise.all([
      writeFn(data),
      this.redis.setex(cacheKey, this.ttl, JSON.stringify(data))
    ]);
    
    return dbResult;
  }

  async read(key, fetchFn) {
    const cacheKey = `${this.prefix}${key}`;
    const cached = await this.redis.get(cacheKey);
    
    if (cached !== null) {
      return JSON.parse(cached);
    }

    // Cache miss - ปกติไม่ควรเกิดถ้า write-through ทำงานถูกต้อง
    const data = await fetchFn();
    if (data) {
      await this.redis.setex(cacheKey, this.ttl, JSON.stringify(data));
    }
    
    return data;
  }
}
```

### 1.3 Write-Behind (Write-Back) Pattern

```javascript
// cache/strategies/write-behind.js
const Queue = require('bull');

class WriteBehindStrategy {
  constructor(redis, db, options = {}) {
    this.redis = redis;
    this.db = db;
    this.ttl = options.ttl || 3600;
    this.prefix = options.prefix || 'wb:';
    this.flushInterval = options.flushInterval || 5000; // 5 seconds
    
    // Queue สำหรับ write operations
    this.writeQueue = new Queue('cache-write-behind', {
      redis: { port: 6379, host: 'redis' }
    });
    
    // Process writes
    this.writeQueue.process(async (job) => {
      const { operation, key, data } = job.data;
      
      if (operation === 'update') {
        await this.db.update(key, data);
      } else if (operation === 'delete') {
        await this.db.delete(key);
      }
    });
  }

  async write(key, data) {
    const cacheKey = `${this.prefix}${key}`;
    
    // Write to cache immediately
    await this.redis.setex(cacheKey, this.ttl, JSON.stringify(data));
    
    // Queue DB write (async, non-blocking)
    await this.writeQueue.add(
      { operation: 'update', key, data },
      {
        delay: 0,
        attempts: 3,
        backoff: { type: 'exponential', delay: 1000 },
        removeOnComplete: true
      }
    );
  }

  async read(key, fetchFn) {
    const cacheKey = `${this.prefix}${key}`;
    const cached = await this.redis.get(cacheKey);
    
    if (cached !== null) {
      return JSON.parse(cached);
    }
    
    const data = await fetchFn();
    if (data) {
      await this.redis.setex(cacheKey, this.ttl, JSON.stringify(data));
    }
    
    return data;
  }
}
```

### 1.4 Multi-Level Cache (L1/L2)

```javascript
// cache/multi-level-cache.js
const LRU = require('lru-cache');

class MultiLevelCache {
  constructor(redis, options = {}) {
    // L1: In-memory LRU cache (เร็วมาก แต่เล็ก)
    this.l1Cache = new LRU({
      max: options.l1Size || 500,         // max 500 items
      maxAge: (options.l1TTL || 60) * 1000, // 60 seconds
      updateAgeOnGet: true
    });

    // L2: Redis (เร็ว แต่ network overhead)
    this.l2Cache = redis;
    this.l2TTL = options.l2TTL || 3600;
    this.prefix = options.prefix || 'ml:';
  }

  async get(key, fetchFn) {
    // Check L1
    const l1Value = this.l1Cache.get(key);
    if (l1Value !== undefined) {
      return l1Value;
    }

    // Check L2
    const cacheKey = `${this.prefix}${key}`;
    const l2Value = await this.l2Cache.get(cacheKey);
    
    if (l2Value !== null) {
      const data = JSON.parse(l2Value);
      // Populate L1
      this.l1Cache.set(key, data);
      return data;
    }

    // Fetch from source
    const data = await fetchFn();
    
    if (data !== null && data !== undefined) {
      // Set both L1 and L2
      this.l1Cache.set(key, data);
      await this.l2Cache.setex(cacheKey, this.l2TTL, JSON.stringify(data));
    }
    
    return data;
  }

  async set(key, value) {
    this.l1Cache.set(key, value);
    const cacheKey = `${this.prefix}${key}`;
    await this.l2Cache.setex(cacheKey, this.l2TTL, JSON.stringify(value));
  }

  async invalidate(key) {
    this.l1Cache.del(key);
    const cacheKey = `${this.prefix}${key}`;
    await this.l2Cache.del(cacheKey);
  }

  getStats() {
    return {
      l1: {
        size: this.l1Cache.itemCount,
        maxSize: this.l1Cache.max,
        hitRate: this.l1Cache.hitCount / (this.l1Cache.hitCount + this.l1Cache.missCount)
      }
    };
  }
}
```

---

## 2. Redis Advanced Patterns

### 2.1 Redis Pipeline สำหรับ Batch Operations

```javascript
// redis/pipeline.js
class RedisOptimizer {
  constructor(redis) {
    this.redis = redis;
  }

  // ดึงข้อมูล multiple users ด้วย pipeline
  async batchGetUsers(userIds) {
    const pipeline = this.redis.pipeline();
    
    for (const userId of userIds) {
      pipeline.get(`user:${userId}`);
    }
    
    const results = await pipeline.exec();
    
    return userIds.reduce((acc, userId, index) => {
      const [err, value] = results[index];
      if (!err && value) {
        acc[userId] = JSON.parse(value);
      }
      return acc;
    }, {});
  }

  // Cache multiple items ด้วย pipeline
  async batchSetWithTTL(items, ttl = 3600) {
    const pipeline = this.redis.pipeline();
    
    for (const { key, value } of items) {
      pipeline.setex(key, ttl, JSON.stringify(value));
    }
    
    return pipeline.exec();
  }

  // Atomic increment สำหรับ rate limiting
  async rateLimit(key, limit, windowSeconds) {
    const pipeline = this.redis.pipeline();
    const now = Date.now();
    const windowStart = now - windowSeconds * 1000;
    
    pipeline
      .zremrangebyscore(key, 0, windowStart)  // Remove expired entries
      .zadd(key, now, `${now}`)               // Add current request
      .zcard(key)                              // Count requests in window
      .expire(key, windowSeconds);             // Set expiry
    
    const [[,], [,], [, count]] = await pipeline.exec();
    
    return {
      allowed: count <= limit,
      count,
      remaining: Math.max(0, limit - count),
      resetTime: new Date(now + windowSeconds * 1000)
    };
  }
}
```

### 2.2 Redis Lua Scripting สำหรับ Atomic Operations

```javascript
// redis/lua-scripts.js
const Redis = require('ioredis');

const redis = new Redis();

// Lua script สำหรับ atomic stock decrement
const DECREMENT_STOCK_SCRIPT = `
local key = KEYS[1]
local amount = tonumber(ARGV[1])
local min_stock = tonumber(ARGV[2])

local current = tonumber(redis.call('GET', key))

if current == nil then
  return -1  -- Key not found
end

if current < amount then
  return -2  -- Insufficient stock
end

if (current - amount) < min_stock then
  return -3  -- Would go below minimum
end

return redis.call('DECRBY', key, amount)
`;

// Register script (get SHA)
const DECREMENT_STOCK_SHA = redis.defineCommand('decrementStock', {
  numberOfKeys: 1,
  lua: DECREMENT_STOCK_SCRIPT
});

async function reserveStock(productId, quantity, minStock = 0) {
  const key = `stock:${productId}`;
  const result = await redis.decrementStock(key, quantity, minStock);
  
  if (result === -1) throw new Error('Product not found in cache');
  if (result === -2) throw new Error('Insufficient stock');
  if (result === -3) throw new Error('Would go below minimum stock');
  
  return result; // New stock level
}

// Lua script สำหรับ distributed lock
const ACQUIRE_LOCK_SCRIPT = `
local key = KEYS[1]
local value = ARGV[1]
local ttl = tonumber(ARGV[2])

if redis.call('SET', key, value, 'NX', 'EX', ttl) then
  return 1
else
  local current = redis.call('GET', key)
  if current == value then
    redis.call('EXPIRE', key, ttl)  -- Renew lock
    return 1
  end
  return 0
end
`;

class DistributedLock {
  constructor(redis) {
    this.redis = redis;
    
    redis.defineCommand('acquireLock', {
      numberOfKeys: 1,
      lua: ACQUIRE_LOCK_SCRIPT
    });
  }

  async acquire(resource, ttlSeconds = 30) {
    const lockKey = `lock:${resource}`;
    const lockValue = `${process.pid}:${Date.now()}:${Math.random()}`;
    
    const acquired = await this.redis.acquireLock(lockKey, lockValue, ttlSeconds);
    
    if (!acquired) {
      return null;
    }
    
    return {
      key: lockKey,
      value: lockValue,
      release: async () => this.release(lockKey, lockValue)
    };
  }

  async release(lockKey, lockValue) {
    const RELEASE_SCRIPT = `
      if redis.call('GET', KEYS[1]) == ARGV[1] then
        return redis.call('DEL', KEYS[1])
      else
        return 0
      end
    `;
    
    return this.redis.eval(RELEASE_SCRIPT, 1, lockKey, lockValue);
  }

  async withLock(resource, ttl, fn) {
    const lock = await this.acquire(resource, ttl);
    
    if (!lock) {
      throw new Error(`Could not acquire lock for ${resource}`);
    }
    
    try {
      return await fn();
    } finally {
      await lock.release();
    }
  }
}

// ใช้งาน
const lock = new DistributedLock(redis);

async function processOrder(orderId) {
  return lock.withLock(`order:${orderId}`, 30, async () => {
    // Critical section - only one instance processes at a time
    const order = await getOrder(orderId);
    await updateOrder(orderId, { status: 'processing' });
    await chargePayment(order);
    await updateOrder(orderId, { status: 'completed' });
  });
}
```

### 2.3 Redis Streams สำหรับ Real-time Events

```javascript
// redis/streams.js
class RedisStreamPublisher {
  constructor(redis, streamName) {
    this.redis = redis;
    this.streamName = streamName;
  }

  async publish(event, data) {
    return this.redis.xadd(
      this.streamName,
      '*',  // Auto-generate ID
      'event', event,
      'data', JSON.stringify(data),
      'timestamp', Date.now().toString()
    );
  }

  // Publish batch
  async publishBatch(events) {
    const pipeline = this.redis.pipeline();
    
    for (const { event, data } of events) {
      pipeline.xadd(
        this.streamName,
        '*',
        'event', event,
        'data', JSON.stringify(data),
        'timestamp', Date.now().toString()
      );
    }
    
    return pipeline.exec();
  }
}

class RedisStreamConsumer {
  constructor(redis, streamName, groupName, consumerName) {
    this.redis = redis;
    this.streamName = streamName;
    this.groupName = groupName;
    this.consumerName = consumerName;
    this.running = false;
  }

  async initialize() {
    try {
      // สร้าง consumer group
      await this.redis.xgroup(
        'CREATE',
        this.streamName,
        this.groupName,
        '$',  // Start from latest
        'MKSTREAM'  // Create stream if not exists
      );
    } catch (err) {
      if (!err.message.includes('BUSYGROUP')) throw err;
    }
  }

  async consume(handler, options = {}) {
    const { batchSize = 10, blockMs = 2000 } = options;
    this.running = true;

    while (this.running) {
      try {
        const messages = await this.redis.xreadgroup(
          'GROUP', this.groupName, this.consumerName,
          'COUNT', batchSize,
          'BLOCK', blockMs,
          'STREAMS', this.streamName, '>'  // Only undelivered
        );

        if (!messages) continue;

        for (const [stream, entries] of messages) {
          for (const [id, fields] of entries) {
            const message = this.parseFields(fields);
            
            try {
              await handler(message);
              // Acknowledge
              await this.redis.xack(this.streamName, this.groupName, id);
            } catch (err) {
              console.error(`Failed to process message ${id}:`, err);
              // Don't ack - will be redelivered
            }
          }
        }

        // Process pending (previously unacked)
        await this.processPending();

      } catch (err) {
        if (this.running) {
          console.error('Stream consumer error:', err);
          await new Promise(resolve => setTimeout(resolve, 1000));
        }
      }
    }
  }

  async processPending() {
    const pending = await this.redis.xpending(
      this.streamName,
      this.groupName,
      '-', '+',  // All pending
      10
    );

    for (const [id, consumer, idle, deliveries] of pending) {
      if (idle > 30000 && deliveries > 3) {
        // Move to dead letter stream after 3 failed deliveries
        const messages = await this.redis.xrange(this.streamName, id, id);
        if (messages.length > 0) {
          await this.redis.xadd(
            `${this.streamName}:dead-letter`,
            '*',
            'original_id', id,
            'deliveries', deliveries,
            ...messages[0][1]
          );
          await this.redis.xack(this.streamName, this.groupName, id);
        }
      }
    }
  }

  parseFields(fields) {
    const obj = {};
    for (let i = 0; i < fields.length; i += 2) {
      obj[fields[i]] = fields[i + 1];
    }
    return {
      event: obj.event,
      data: JSON.parse(obj.data),
      timestamp: parseInt(obj.timestamp)
    };
  }

  stop() {
    this.running = false;
  }
}
```

---

## 3. Database Performance Optimization

### 3.1 Query Optimization

```javascript
// db/query-optimizer.js

// ❌ ปัญหา: N+1 Query
async function getUsersWithOrders_BAD() {
  const users = await db.query('SELECT * FROM users');
  
  // N+1: query สำหรับแต่ละ user!
  for (const user of users) {
    user.orders = await db.query(
      'SELECT * FROM orders WHERE user_id = $1',
      [user.id]
    );
  }
  
  return users;
}

// ✅ แก้ไข: JOIN หรือ batch query
async function getUsersWithOrders_GOOD() {
  // Option 1: JOIN
  const result = await db.query(`
    SELECT 
      u.id, u.name, u.email,
      json_agg(
        json_build_object(
          'id', o.id,
          'total', o.total,
          'status', o.status
        ) ORDER BY o.created_at DESC
      ) FILTER (WHERE o.id IS NOT NULL) as orders
    FROM users u
    LEFT JOIN orders o ON o.user_id = u.id
    GROUP BY u.id
    LIMIT 100
  `);
  
  return result.rows;
}

// ✅ Option 2: DataLoader pattern (batch + cache)
const DataLoader = require('dataloader');

class OrdersDataLoader {
  constructor(db) {
    this.loader = new DataLoader(
      userIds => this.batchLoad(db, userIds),
      { cache: true, maxBatchSize: 1000 }
    );
  }

  async batchLoad(db, userIds) {
    const result = await db.query(
      'SELECT * FROM orders WHERE user_id = ANY($1)',
      [userIds]
    );
    
    // Group by user_id
    const ordersByUser = result.rows.reduce((acc, order) => {
      if (!acc[order.user_id]) acc[order.user_id] = [];
      acc[order.user_id].push(order);
      return acc;
    }, {});
    
    // Return in same order as userIds
    return userIds.map(id => ordersByUser[id] || []);
  }

  async loadForUser(userId) {
    return this.loader.load(userId);
  }
}
```

### 3.2 Database Connection Pool Optimization

```javascript
// db/connection-pool.js
const { Pool } = require('pg');

class OptimizedConnectionPool {
  constructor() {
    // Main pool สำหรับ writes
    this.writePool = new Pool({
      host: process.env.DB_PRIMARY_HOST,
      port: 5432,
      database: process.env.DB_NAME,
      user: process.env.DB_USER,
      password: process.env.DB_PASSWORD,
      
      // Connection pool settings
      min: 5,          // Minimum connections
      max: 20,         // Maximum connections
      idleTimeoutMillis: 30000,
      connectionTimeoutMillis: 5000,
      
      // Statement timeout
      statement_timeout: 30000,  // 30 seconds
      
      // Slow query logging
      application_name: `${process.env.SERVICE_NAME}-write`
    });

    // Read replica pool สำหรับ reads
    this.readPool = new Pool({
      host: process.env.DB_REPLICA_HOST,
      port: 5432,
      database: process.env.DB_NAME,
      user: process.env.DB_USER,
      password: process.env.DB_PASSWORD,
      
      min: 10,         // More connections for reads
      max: 50,
      idleTimeoutMillis: 60000,
      connectionTimeoutMillis: 5000,
      
      statement_timeout: 10000,  // Stricter timeout for reads
      application_name: `${process.env.SERVICE_NAME}-read`
    });

    this.setupMonitoring();
  }

  setupMonitoring() {
    const pools = [
      { name: 'write', pool: this.writePool },
      { name: 'read', pool: this.readPool }
    ];

    for (const { name, pool } of pools) {
      pool.on('connect', () => {
        metrics.dbConnectionsTotal.inc({ pool: name });
      });

      pool.on('remove', () => {
        metrics.dbConnectionsTotal.dec({ pool: name });
      });

      pool.on('error', (err) => {
        console.error(`Pool ${name} error:`, err);
        metrics.dbErrors.inc({ pool: name });
      });

      // Expose pool stats
      setInterval(() => {
        metrics.dbConnectionsActive.set(
          { pool: name },
          pool.totalCount - pool.idleCount
        );
        metrics.dbConnectionsIdle.set(
          { pool: name },
          pool.idleCount
        );
        metrics.dbConnectionsWaiting.set(
          { pool: name },
          pool.waitingCount
        );
      }, 5000);
    }
  }

  async query(sql, params, options = {}) {
    const pool = options.useReplica ? this.readPool : this.writePool;
    const startTime = Date.now();
    
    try {
      const result = await pool.query(sql, params);
      
      const duration = Date.now() - startTime;
      metrics.dbQueryDuration.observe({ query: extractQueryName(sql) }, duration / 1000);
      
      if (duration > 1000) {
        console.warn('Slow query detected:', { sql, duration, params });
      }
      
      return result;
    } catch (err) {
      metrics.dbQueryErrors.inc({ query: extractQueryName(sql) });
      throw err;
    }
  }

  async transaction(fn) {
    const client = await this.writePool.connect();
    
    try {
      await client.query('BEGIN');
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

### 3.3 PostgreSQL Index Optimization

```sql
-- indexes.sql

-- ❌ ปัญหา: Full table scan
EXPLAIN ANALYZE
SELECT * FROM orders WHERE user_id = 123 AND status = 'pending';

-- ✅ Composite index
CREATE INDEX CONCURRENTLY idx_orders_user_status 
ON orders(user_id, status) 
WHERE status IN ('pending', 'processing');

-- Partial index สำหรับ active records
CREATE INDEX CONCURRENTLY idx_users_active
ON users(email)
WHERE deleted_at IS NULL;

-- Index สำหรับ JSON column
CREATE INDEX CONCURRENTLY idx_orders_metadata_payment
ON orders((metadata->>'payment_method'));

-- Index สำหรับ LIKE queries (text search)
CREATE INDEX CONCURRENTLY idx_products_name_trgm
ON products USING gin(name gin_trgm_ops);

-- ดู slow queries
SELECT 
  query,
  calls,
  total_exec_time / calls AS avg_ms,
  stddev_exec_time,
  rows / calls AS avg_rows
FROM pg_stat_statements
WHERE calls > 100
ORDER BY total_exec_time DESC
LIMIT 20;

-- ดู unused indexes
SELECT 
  schemaname,
  tablename,
  indexname,
  idx_scan,
  pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;

-- Analyze query plan
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)
SELECT u.id, u.name, COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE u.created_at > NOW() - INTERVAL '30 days'
GROUP BY u.id, u.name;
```

---

## 4. API Performance Optimization

### 4.1 Response Compression

```javascript
// middleware/compression.js
const compression = require('compression');
const zlib = require('zlib');

// Standard compression middleware
app.use(compression({
  level: 6,  // 1-9 (1=fast, 9=best compression)
  threshold: 1024,  // Only compress > 1KB
  filter: (req, res) => {
    // Don't compress streaming responses
    if (req.headers['x-no-compression']) return false;
    return compression.filter(req, res);
  }
}));

// Brotli compression (better than gzip for text)
const { createBrotliCompress, createGzip, createDeflate } = zlib;

function getCompressionStream(encoding) {
  switch (encoding) {
    case 'br': return createBrotliCompress({ params: { [zlib.constants.BROTLI_PARAM_QUALITY]: 4 } });
    case 'gzip': return createGzip({ level: 6 });
    case 'deflate': return createDeflate();
    default: return null;
  }
}

// Custom compression middleware with Brotli support
function advancedCompression(req, res, next) {
  const acceptEncoding = req.headers['accept-encoding'] || '';
  
  let encoding = null;
  if (acceptEncoding.includes('br')) encoding = 'br';
  else if (acceptEncoding.includes('gzip')) encoding = 'gzip';
  else if (acceptEncoding.includes('deflate')) encoding = 'deflate';
  
  if (!encoding) return next();
  
  const originalJson = res.json.bind(res);
  
  res.json = function(data) {
    const jsonStr = JSON.stringify(data);
    
    if (jsonStr.length < 1024) {
      return originalJson(data);
    }
    
    const stream = getCompressionStream(encoding);
    
    res.setHeader('Content-Encoding', encoding);
    res.setHeader('Vary', 'Accept-Encoding');
    res.removeHeader('Content-Length');
    
    stream.pipe(res);
    stream.end(jsonStr);
  };
  
  next();
}
```

### 4.2 Cursor-based Pagination

```javascript
// api/pagination.js

// ❌ Offset pagination - ช้าเมื่อ offset ใหญ่
async function getOrdersOffset(userId, page, pageSize) {
  const offset = (page - 1) * pageSize;
  
  return db.query(
    'SELECT * FROM orders WHERE user_id = $1 ORDER BY id DESC LIMIT $2 OFFSET $3',
    [userId, pageSize, offset]  // OFFSET scan ทุก row ก่อนถึง offset!
  );
}

// ✅ Cursor-based pagination - เร็วคงที่
class CursorPaginator {
  constructor(db) {
    this.db = db;
  }

  async paginate(options) {
    const {
      table,
      conditions = [],
      params = [],
      cursor,
      limit = 20,
      orderBy = 'id',
      direction = 'DESC'
    } = options;

    const decodedCursor = cursor ? this.decodeCursor(cursor) : null;
    
    let whereClause = conditions.length > 0
      ? `WHERE ${conditions.join(' AND ')}`
      : '';
    
    // Add cursor condition
    if (decodedCursor) {
      const op = direction === 'DESC' ? '<' : '>';
      const cursorCondition = `${orderBy} ${op} $${params.length + 1}`;
      
      whereClause = whereClause
        ? `${whereClause} AND ${cursorCondition}`
        : `WHERE ${cursorCondition}`;
      
      params.push(decodedCursor.value);
    }

    const query = `
      SELECT * FROM ${table}
      ${whereClause}
      ORDER BY ${orderBy} ${direction}
      LIMIT ${limit + 1}  -- Fetch one extra to determine hasNextPage
    `;

    const result = await this.db.query(query, params);
    const rows = result.rows;
    
    const hasNextPage = rows.length > limit;
    const items = hasNextPage ? rows.slice(0, limit) : rows;
    
    const nextCursor = hasNextPage
      ? this.encodeCursor({ value: items[items.length - 1][orderBy] })
      : null;

    return {
      items,
      pageInfo: {
        hasNextPage,
        nextCursor,
        count: items.length
      }
    };
  }

  encodeCursor(data) {
    return Buffer.from(JSON.stringify(data)).toString('base64url');
  }

  decodeCursor(cursor) {
    try {
      return JSON.parse(Buffer.from(cursor, 'base64url').toString());
    } catch {
      throw new ValidationError('Invalid cursor');
    }
  }
}

// ใช้งาน
const paginator = new CursorPaginator(db);

async function getOrders(req, res) {
  const { cursor, limit } = req.query;
  const userId = req.user.id;

  const result = await paginator.paginate({
    table: 'orders',
    conditions: ['user_id = $1'],
    params: [userId],
    cursor,
    limit: parseInt(limit) || 20,
    orderBy: 'created_at',
    direction: 'DESC'
  });

  res.json(result);
}
```

### 4.3 HTTP/2 Server Push

```javascript
// server/http2.js
const http2 = require('http2');
const fs = require('fs');

const server = http2.createSecureServer({
  key: fs.readFileSync('key.pem'),
  cert: fs.readFileSync('cert.pem')
});

server.on('stream', (stream, headers) => {
  const path = headers[':path'];
  
  if (path === '/api/dashboard') {
    // Push related resources
    const pushes = [
      { path: '/api/users/me', resource: getUserProfile },
      { path: '/api/notifications/unread', resource: getNotifications },
      { path: '/api/orders/recent', resource: getRecentOrders }
    ];
    
    for (const push of pushes) {
      const pushHeaders = {
        ':status': 200,
        'content-type': 'application/json',
        ':path': push.path
      };
      
      stream.pushStream({ ':path': push.path }, (err, pushStream) => {
        if (err) return;
        
        pushStream.respond(pushHeaders);
        push.resource().then(data => {
          pushStream.end(JSON.stringify(data));
        });
      });
    }
    
    // Respond to original request
    stream.respond({ ':status': 200, 'content-type': 'application/json' });
    getDashboardData().then(data => stream.end(JSON.stringify(data)));
  }
});
```

---

## 5. Node.js Performance Profiling

### 5.1 Memory Profiling

```javascript
// profiling/memory-profiler.js
const v8 = require('v8');
const memwatch = require('@airbnb/node-memwatch');

class MemoryProfiler {
  constructor() {
    this.leaks = [];
    this.snapshots = [];
    
    // Detect memory leaks
    memwatch.on('leak', (info) => {
      console.warn('Memory leak detected:', info);
      this.leaks.push({
        ...info,
        timestamp: new Date(),
        heapSnapshot: this.takeHeapSnapshot()
      });
      
      // Alert if many leaks
      if (this.leaks.length > 5) {
        alerting.send('critical', 'Multiple memory leaks detected', {
          count: this.leaks.length,
          growth: info.growth
        });
      }
    });

    // Log heap stats
    setInterval(() => {
      const stats = v8.getHeapStatistics();
      
      metrics.heapUsed.set(stats.used_heap_size);
      metrics.heapTotal.set(stats.total_heap_size);
      metrics.externalMemory.set(stats.external);
      
      // Force GC if heap is too large (careful in production!)
      if (stats.used_heap_size / stats.heap_size_limit > 0.85) {
        console.warn('High heap usage, suggesting GC');
        if (global.gc) global.gc();
      }
    }, 10000);
  }

  takeHeapSnapshot() {
    const snapshot = v8.writeHeapSnapshot();
    this.snapshots.push({ path: snapshot, timestamp: new Date() });
    return snapshot;
  }

  getHeapStats() {
    return {
      stats: v8.getHeapStatistics(),
      processMemory: process.memoryUsage(),
      leaks: this.leaks.length
    };
  }
}
```

### 5.2 Worker Threads สำหรับ CPU-intensive Tasks

```javascript
// workers/cpu-intensive.js
const { Worker, isMainThread, parentPort, workerData } = require('worker_threads');
const os = require('os');

if (isMainThread) {
  // Worker Pool
  class WorkerPool {
    constructor(workerFile, poolSize) {
      this.workerFile = workerFile;
      this.poolSize = poolSize || os.cpus().length;
      this.workers = [];
      this.queue = [];
      this.freeWorkers = [];
      
      this.initialize();
    }

    initialize() {
      for (let i = 0; i < this.poolSize; i++) {
        const worker = new Worker(this.workerFile);
        
        worker.on('message', (result) => {
          const { resolve, reject } = worker._currentTask;
          worker._currentTask = null;
          this.freeWorkers.push(worker);
          
          if (result.error) {
            reject(new Error(result.error));
          } else {
            resolve(result.data);
          }
          
          this.processQueue();
        });

        worker.on('error', (err) => {
          if (worker._currentTask) {
            worker._currentTask.reject(err);
          }
          
          // Replace dead worker
          this.workers.splice(this.workers.indexOf(worker), 1);
          this.createWorker();
        });

        this.workers.push(worker);
        this.freeWorkers.push(worker);
      }
    }

    runTask(data) {
      return new Promise((resolve, reject) => {
        if (this.freeWorkers.length > 0) {
          const worker = this.freeWorkers.pop();
          worker._currentTask = { resolve, reject };
          worker.postMessage(data);
        } else {
          this.queue.push({ data, resolve, reject });
        }
      });
    }

    processQueue() {
      if (this.queue.length > 0 && this.freeWorkers.length > 0) {
        const { data, resolve, reject } = this.queue.shift();
        const worker = this.freeWorkers.pop();
        worker._currentTask = { resolve, reject };
        worker.postMessage(data);
      }
    }

    terminate() {
      this.workers.forEach(w => w.terminate());
    }
  }

  // Export pool
  const imageProcessingPool = new WorkerPool('./image-worker.js');
  const reportGenerationPool = new WorkerPool('./report-worker.js');

  // ใช้งาน
  async function processImage(imageBuffer) {
    return imageProcessingPool.runTask({
      operation: 'resize',
      buffer: imageBuffer,
      width: 800,
      height: 600
    });
  }

} else {
  // Worker thread code
  parentPort.on('message', async (task) => {
    try {
      let result;
      
      switch (task.operation) {
        case 'resize':
          result = await sharp(task.buffer)
            .resize(task.width, task.height)
            .toBuffer();
          break;
        
        case 'generate-report':
          result = await generatePDFReport(task.data);
          break;
        
        default:
          throw new Error(`Unknown operation: ${task.operation}`);
      }
      
      parentPort.postMessage({ data: result });
    } catch (err) {
      parentPort.postMessage({ error: err.message });
    }
  });
}
```

---

## Workshop: ทดสอบ Cache Performance

```javascript
// test/cache-performance.test.js
const { performance } = require('perf_hooks');

async function benchmarkCachingStrategies() {
  const redis = new Redis();
  const db = new Pool({ /* ... */ });
  
  const USER_IDS = Array.from({ length: 1000 }, (_, i) => i + 1);
  const ITERATIONS = 10;

  // Benchmark 1: No cache
  console.time('No cache - 1000 users');
  for (const userId of USER_IDS) {
    await db.query('SELECT * FROM users WHERE id = $1', [userId]);
  }
  console.timeEnd('No cache - 1000 users');

  // Benchmark 2: Cache-aside
  const cache = new CacheAsideStrategy(redis, db);
  
  // Warm up cache
  for (const userId of USER_IDS) {
    await cache.get(`user:${userId}`, () =>
      db.query('SELECT * FROM users WHERE id = $1', [userId])
        .then(r => r.rows[0])
    );
  }
  
  console.time('Cache-aside (warm) - 1000 users');
  for (const userId of USER_IDS) {
    await cache.get(`user:${userId}`, () =>
      db.query('SELECT * FROM users WHERE id = $1', [userId])
        .then(r => r.rows[0])
    );
  }
  console.timeEnd('Cache-aside (warm) - 1000 users');

  // Benchmark 3: Multi-level cache
  const multiCache = new MultiLevelCache(redis);
  
  console.time('Multi-level cache (warm) - 1000 users');
  for (const userId of USER_IDS) {
    await multiCache.get(`user:${userId}`, async () => {
      const result = await db.query(
        'SELECT * FROM users WHERE id = $1',
        [userId]
      );
      return result.rows[0];
    });
  }
  console.timeEnd('Multi-level cache (warm) - 1000 users');

  // Benchmark 4: Pipeline
  console.time('Redis pipeline - 1000 users');
  const optimizer = new RedisOptimizer(redis);
  await optimizer.batchGetUsers(USER_IDS);
  console.timeEnd('Redis pipeline - 1000 users');
}

benchmarkCachingStrategies()
  .then(() => process.exit(0))
  .catch(console.error);
```

---

## สรุป

| เทคนิค | ใช้เมื่อ | Benefit |
|--------|---------|---------|
| **Cache-Aside** | Read-heavy, occasional updates | ลด DB load 70-90% |
| **Write-Through** | Consistent reads critical | ไม่มี stale data |
| **Write-Behind** | Write-heavy, eventual consistency OK | เพิ่ม write throughput |
| **Multi-Level** | Ultra-high read throughput needed | P99 latency < 1ms |
| **Cursor Pagination** | Large datasets | Consistent O(1) performance |
| **Worker Threads** | CPU-intensive tasks | Utilize all CPU cores |
| **Pipeline** | Multiple Redis ops | ลด network roundtrips |

**Performance Checklist:**
- [ ] Database indexes สำหรับทุก query pattern
- [ ] Connection pool ตั้งค่าถูก (min/max)
- [ ] N+1 queries ถูกกำจัดทั้งหมด
- [ ] Response compression เปิดใช้งาน
- [ ] Cursor pagination แทน offset
- [ ] Cache hot data ใน Redis
- [ ] Worker threads สำหรับ CPU work

**Next:** Part 21 - Security Hardening: OWASP Top 10 and Production Security

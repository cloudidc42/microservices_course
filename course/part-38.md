# Part 38: Distributed Caching Patterns สำหรับ Microservices

## บทนำ

Distributed Caching เป็นหนึ่งในกลยุทธ์ที่สำคัญที่สุดในการเพิ่มประสิทธิภาพของ Microservices ในระดับ Production การเข้าใจ patterns ต่างๆ จะช่วยให้เลือกใช้ได้อย่างเหมาะสมกับแต่ละสถานการณ์

## หัวข้อที่ครอบคลุม

1. Cache-Aside Pattern
2. Write-Through Pattern  
3. Write-Behind (Write-Back) Pattern
4. Read-Through Pattern
5. Multi-Level Cache
6. Cache Stampede Prevention
7. Cache Invalidation Strategies
8. Distributed Cache ด้วย Redis Cluster
9. Cache Warming
10. Cache Observability

---

## 1. Cache-Aside Pattern

Cache-Aside (หรือ Lazy Loading) เป็น pattern พื้นฐานที่ application จัดการ cache โดยตรง

```typescript
// caching/cache-aside.ts
import Redis from 'ioredis';
import { Pool } from 'pg';

interface CacheConfig {
  ttlSeconds: number;
  keyPrefix: string;
}

export class CacheAsideService<T> {
  constructor(
    private redis: Redis,
    private db: Pool,
    private config: CacheConfig
  ) {}
  
  async get(
    key: string,
    fetchFromDB: () => Promise<T | null>
  ): Promise<T | null> {
    const cacheKey = `${this.config.keyPrefix}:${key}`;
    
    // 1. ตรวจสอบ cache
    const cached = await this.redis.get(cacheKey);
    if (cached !== null) {
      return JSON.parse(cached) as T;
    }
    
    // 2. Cache miss → ดึงจาก DB
    const data = await fetchFromDB();
    
    // 3. เก็บลง cache (ถ้ามีข้อมูล)
    if (data !== null) {
      await this.redis.setex(
        cacheKey,
        this.config.ttlSeconds,
        JSON.stringify(data)
      );
    }
    
    return data;
  }
  
  async invalidate(key: string): Promise<void> {
    const cacheKey = `${this.config.keyPrefix}:${key}`;
    await this.redis.del(cacheKey);
  }
  
  async invalidatePattern(pattern: string): Promise<void> {
    const keys = await this.redis.keys(`${this.config.keyPrefix}:${pattern}`);
    if (keys.length > 0) {
      await this.redis.del(...keys);
    }
  }
}

// ตัวอย่างการใช้งาน: Product Service
export class ProductCacheService {
  private cache: CacheAsideService<Product>;
  
  constructor(redis: Redis, db: Pool) {
    this.cache = new CacheAsideService<Product>(redis, db, {
      ttlSeconds: 3600,
      keyPrefix: 'product',
    });
  }
  
  async getProduct(productId: string): Promise<Product | null> {
    return this.cache.get(productId, async () => {
      const result = await this.db.query(
        'SELECT * FROM products WHERE id = $1',
        [productId]
      );
      return result.rows[0] || null;
    });
  }
  
  async updateProduct(productId: string, data: Partial<Product>): Promise<Product> {
    // Update DB
    const result = await this.db.query(
      `UPDATE products SET name = $1, price = $2, updated_at = NOW()
       WHERE id = $3 RETURNING *`,
      [data.name, data.price, productId]
    );
    
    // Invalidate cache
    await this.cache.invalidate(productId);
    
    return result.rows[0];
  }
  
  async getProductsByCategory(categoryId: string): Promise<Product[]> {
    return this.cache.get(`category:${categoryId}`, async () => {
      const result = await this.db.query(
        'SELECT * FROM products WHERE category_id = $1',
        [categoryId]
      );
      return result.rows;
    }) as Promise<Product[]>;
  }
  
  async invalidateCategory(categoryId: string): Promise<void> {
    await this.cache.invalidate(`category:${categoryId}`);
    // ลบ cache ของ products ทุกตัวใน category
    await this.cache.invalidatePattern(`category:${categoryId}:*`);
  }
}

interface Product {
  id: string;
  name: string;
  price: number;
  categoryId: string;
}
```

---

## 2. Write-Through Pattern

Write-Through เขียนข้อมูลลง cache และ database พร้อมกัน ทำให้ cache ถูก update เสมอ

```typescript
// caching/write-through.ts
import Redis from 'ioredis';
import { Pool } from 'pg';

export class WriteThroughCache<T> {
  constructor(
    private redis: Redis,
    private db: Pool,
    private keyPrefix: string,
    private ttlSeconds: number = 3600
  ) {}
  
  async get(key: string): Promise<T | null> {
    const cacheKey = `${this.keyPrefix}:${key}`;
    const cached = await this.redis.get(cacheKey);
    
    if (cached !== null) {
      return JSON.parse(cached) as T;
    }
    
    // Cache miss (ไม่ควรเกิดบ่อย ถ้า write-through ทำงานถูกต้อง)
    return null;
  }
  
  async set(
    key: string,
    data: T,
    dbWriter: (data: T) => Promise<void>
  ): Promise<void> {
    const cacheKey = `${this.keyPrefix}:${key}`;
    
    // เขียนลง DB และ Cache พร้อมกัน
    await Promise.all([
      dbWriter(data),
      this.redis.setex(cacheKey, this.ttlSeconds, JSON.stringify(data)),
    ]);
  }
  
  async delete(
    key: string,
    dbDeleter: () => Promise<void>
  ): Promise<void> {
    const cacheKey = `${this.keyPrefix}:${key}`;
    
    await Promise.all([
      dbDeleter(),
      this.redis.del(cacheKey),
    ]);
  }
}

// User Session Cache - ตัวอย่างที่เหมาะกับ Write-Through
export class UserSessionCache {
  private cache: WriteThroughCache<UserSession>;
  
  constructor(redis: Redis, db: Pool) {
    this.cache = new WriteThroughCache<UserSession>(
      redis, db, 'session', 86400 // 24 ชั่วโมง
    );
  }
  
  async createSession(session: UserSession): Promise<void> {
    await this.cache.set(
      session.sessionId,
      session,
      async (s) => {
        await this.db.query(
          `INSERT INTO user_sessions 
           (session_id, user_id, data, expires_at)
           VALUES ($1, $2, $3, $4)`,
          [s.sessionId, s.userId, JSON.stringify(s.data), s.expiresAt]
        );
      }
    );
  }
  
  async getSession(sessionId: string): Promise<UserSession | null> {
    return this.cache.get(sessionId);
  }
  
  async deleteSession(sessionId: string): Promise<void> {
    await this.cache.delete(sessionId, async () => {
      await this.db.query(
        'DELETE FROM user_sessions WHERE session_id = $1',
        [sessionId]
      );
    });
  }
  
  private db: Pool = {} as Pool; // placeholder
}

interface UserSession {
  sessionId: string;
  userId: string;
  data: Record<string, unknown>;
  expiresAt: Date;
}
```

---

## 3. Write-Behind (Write-Back) Pattern

Write-Behind เขียนลง cache ก่อน แล้ว async เขียนลง database ในภายหลัง เพิ่ม write performance อย่างมาก

```typescript
// caching/write-behind.ts
import Redis from 'ioredis';
import { Pool } from 'pg';
import { EventEmitter } from 'events';

interface WriteBehindConfig {
  keyPrefix: string;
  ttlSeconds: number;
  flushIntervalMs: number;
  maxBatchSize: number;
  pendingWritesKey: string;
}

export class WriteBehindCache<T> extends EventEmitter {
  private flushTimer: NodeJS.Timeout | null = null;
  private pendingWrites: Map<string, { data: T; timestamp: number }> = new Map();
  private isShuttingDown = false;
  
  constructor(
    private redis: Redis,
    private db: Pool,
    private config: WriteBehindConfig,
    private dbWriter: (writes: Array<{ key: string; data: T }>) => Promise<void>
  ) {
    super();
    this.startFlushScheduler();
  }
  
  async set(key: string, data: T): Promise<void> {
    const cacheKey = `${this.config.keyPrefix}:${key}`;
    
    // เขียนลง Redis ทันที
    await this.redis.setex(
      cacheKey,
      this.config.ttlSeconds,
      JSON.stringify(data)
    );
    
    // เพิ่มลง pending writes queue
    await this.redis.zadd(
      this.config.pendingWritesKey,
      Date.now(),
      key
    );
    
    // เก็บใน memory buffer ด้วย
    this.pendingWrites.set(key, { data, timestamp: Date.now() });
    
    // ถ้า buffer เต็ม flush ทันที
    if (this.pendingWrites.size >= this.config.maxBatchSize) {
      await this.flush();
    }
  }
  
  async get(key: string): Promise<T | null> {
    const cacheKey = `${this.config.keyPrefix}:${key}`;
    const cached = await this.redis.get(cacheKey);
    return cached ? JSON.parse(cached) as T : null;
  }
  
  private startFlushScheduler(): void {
    this.flushTimer = setInterval(async () => {
      if (!this.isShuttingDown) {
        await this.flush();
      }
    }, this.config.flushIntervalMs);
  }
  
  async flush(): Promise<void> {
    if (this.pendingWrites.size === 0) return;
    
    // Copy pending writes
    const writes = [...this.pendingWrites.entries()].map(([key, { data }]) => ({
      key,
      data,
    }));
    
    this.pendingWrites.clear();
    
    try {
      await this.dbWriter(writes);
      
      // ลบออกจาก pending queue
      const keys = writes.map(w => w.key);
      if (keys.length > 0) {
        await this.redis.zrem(this.config.pendingWritesKey, ...keys);
      }
      
      this.emit('flush_success', { count: writes.length });
      console.log(`Write-behind: flushed ${writes.length} writes to DB`);
    } catch (error) {
      // ถ้า flush ไม่สำเร็จ ให้ retry
      console.error('Write-behind flush failed, will retry:', error);
      for (const write of writes) {
        this.pendingWrites.set(write.key, {
          data: write.data,
          timestamp: Date.now(),
        });
      }
      this.emit('flush_error', { error, count: writes.length });
    }
  }
  
  async shutdown(): Promise<void> {
    this.isShuttingDown = true;
    
    if (this.flushTimer) {
      clearInterval(this.flushTimer);
    }
    
    // Flush ที่เหลือก่อน shutdown
    await this.flush();
    console.log('Write-behind cache shut down gracefully');
  }
  
  // ดึง pending writes ที่ยังไม่ได้ flush (สำหรับ monitoring)
  getPendingWritesCount(): number {
    return this.pendingWrites.size;
  }
}

// ตัวอย่าง: Analytics Counter
export class AnalyticsCounterCache {
  private cache: WriteBehindCache<ProductAnalytics>;
  
  constructor(redis: Redis, db: Pool) {
    this.cache = new WriteBehindCache<ProductAnalytics>(
      redis, db,
      {
        keyPrefix: 'analytics',
        ttlSeconds: 86400,
        flushIntervalMs: 5000,   // flush ทุก 5 วินาที
        maxBatchSize: 1000,
        pendingWritesKey: 'analytics:pending',
      },
      async (writes) => {
        // Batch upsert ลง DB
        const values = writes.map((w, i) =>
          `($${i * 3 + 1}, $${i * 3 + 2}, $${i * 3 + 3})`
        ).join(', ');
        
        const params = writes.flatMap(w => [
          w.key,
          w.data.viewCount,
          w.data.purchaseCount,
        ]);
        
        await db.query(
          `INSERT INTO product_analytics (product_id, view_count, purchase_count)
           VALUES ${values}
           ON CONFLICT (product_id) DO UPDATE
           SET view_count = EXCLUDED.view_count,
               purchase_count = EXCLUDED.purchase_count,
               updated_at = NOW()`,
          params
        );
      }
    );
  }
  
  async incrementView(productId: string): Promise<void> {
    const current = await this.cache.get(productId) || {
      viewCount: 0,
      purchaseCount: 0,
    };
    
    await this.cache.set(productId, {
      ...current,
      viewCount: current.viewCount + 1,
    });
  }
  
  async incrementPurchase(productId: string): Promise<void> {
    const current = await this.cache.get(productId) || {
      viewCount: 0,
      purchaseCount: 0,
    };
    
    await this.cache.set(productId, {
      ...current,
      purchaseCount: current.purchaseCount + 1,
    });
  }
}

interface ProductAnalytics {
  viewCount: number;
  purchaseCount: number;
}
```

---

## 4. Multi-Level Cache

```typescript
// caching/multi-level-cache.ts

// L1: In-Memory Cache (fastest, smallest)
class L1MemoryCache<T> {
  private cache = new Map<string, { data: T; expiresAt: number }>();
  private maxSize: number;
  
  constructor(maxSize: number = 1000) {
    this.maxSize = maxSize;
  }
  
  get(key: string): T | null {
    const entry = this.cache.get(key);
    if (!entry) return null;
    
    if (Date.now() > entry.expiresAt) {
      this.cache.delete(key);
      return null;
    }
    
    return entry.data;
  }
  
  set(key: string, data: T, ttlMs: number): void {
    // Evict เมื่อ cache เต็ม (LRU-like: ลบรายการแรก)
    if (this.cache.size >= this.maxSize) {
      const firstKey = this.cache.keys().next().value;
      if (firstKey) this.cache.delete(firstKey);
    }
    
    this.cache.set(key, {
      data,
      expiresAt: Date.now() + ttlMs,
    });
  }
  
  delete(key: string): void {
    this.cache.delete(key);
  }
  
  clear(): void {
    this.cache.clear();
  }
  
  size(): number {
    return this.cache.size;
  }
}

// L2: Redis Cache (fast, larger, distributed)
// L3: Database (slow, unlimited, persistent)

export class MultiLevelCacheService<T> {
  private l1: L1MemoryCache<T>;
  
  constructor(
    private redis: Redis,
    private db: Pool,
    private config: {
      keyPrefix: string;
      l1TtlMs: number;      // L1: 30 seconds
      l2TtlSeconds: number; // L2: 1 hour
      l3Query: (key: string) => Promise<T | null>;
    }
  ) {
    this.l1 = new L1MemoryCache<T>(500);
  }
  
  async get(key: string): Promise<T | null> {
    const cacheKey = `${this.config.keyPrefix}:${key}`;
    
    // L1: Memory check
    const l1Data = this.l1.get(cacheKey);
    if (l1Data !== null) {
      this.recordCacheHit('l1');
      return l1Data;
    }
    
    // L2: Redis check
    const l2Data = await this.redis.get(cacheKey);
    if (l2Data !== null) {
      const parsed = JSON.parse(l2Data) as T;
      // Backfill L1
      this.l1.set(cacheKey, parsed, this.config.l1TtlMs);
      this.recordCacheHit('l2');
      return parsed;
    }
    
    // L3: Database
    const l3Data = await this.config.l3Query(key);
    if (l3Data !== null) {
      // Backfill L2 and L1
      await this.redis.setex(
        cacheKey,
        this.config.l2TtlSeconds,
        JSON.stringify(l3Data)
      );
      this.l1.set(cacheKey, l3Data, this.config.l1TtlMs);
      this.recordCacheHit('l3');
    } else {
      this.recordCacheMiss();
    }
    
    return l3Data;
  }
  
  async set(key: string, data: T): Promise<void> {
    const cacheKey = `${this.config.keyPrefix}:${key}`;
    
    // Update ทุก level
    this.l1.set(cacheKey, data, this.config.l1TtlMs);
    await this.redis.setex(
      cacheKey,
      this.config.l2TtlSeconds,
      JSON.stringify(data)
    );
    // L3 (DB) จะถูก update โดย caller
  }
  
  async invalidate(key: string): Promise<void> {
    const cacheKey = `${this.config.keyPrefix}:${key}`;
    this.l1.delete(cacheKey);
    await this.redis.del(cacheKey);
  }
  
  async invalidateAll(): Promise<void> {
    this.l1.clear();
    const keys = await this.redis.keys(`${this.config.keyPrefix}:*`);
    if (keys.length > 0) {
      await this.redis.del(...keys);
    }
  }
  
  private recordCacheHit(level: 'l1' | 'l2' | 'l3'): void {
    // Prometheus metrics
    cacheHitsCounter.inc({ level, prefix: this.config.keyPrefix });
  }
  
  private recordCacheMiss(): void {
    cacheMissesCounter.inc({ prefix: this.config.keyPrefix });
  }
}

// Prometheus counters (ประกาศภายนอก)
declare const cacheHitsCounter: any;
declare const cacheMissesCounter: any;
declare const Redis: any;
declare const Pool: any;
```

---

## 5. Cache Stampede Prevention

Cache Stampede เกิดเมื่อ cache หมดอายุพร้อมกัน และ requests จำนวนมากพุ่งเข้า database ในเวลาเดียวกัน

### 5.1 Mutex Lock Pattern

```typescript
// caching/stampede-prevention.ts
import Redis from 'ioredis';

export class StampedePreventionCache<T> {
  private lockPrefix = 'lock';
  private lockTtlMs = 5000; // 5 seconds
  
  constructor(
    private redis: Redis,
    private keyPrefix: string,
    private ttlSeconds: number
  ) {}
  
  async getOrFetch(
    key: string,
    fetcher: () => Promise<T>
  ): Promise<T> {
    const cacheKey = `${this.keyPrefix}:${key}`;
    const lockKey = `${this.lockPrefix}:${cacheKey}`;
    
    // ตรวจสอบ cache
    const cached = await this.redis.get(cacheKey);
    if (cached !== null) {
      return JSON.parse(cached) as T;
    }
    
    // พยายาม acquire lock
    const lockAcquired = await this.redis.set(
      lockKey,
      '1',
      'PX', this.lockTtlMs,
      'NX'  // Only set if not exists
    );
    
    if (lockAcquired) {
      // เราได้ lock → fetch และ populate cache
      try {
        const data = await fetcher();
        await this.redis.setex(
          cacheKey,
          this.ttlSeconds,
          JSON.stringify(data)
        );
        return data;
      } finally {
        await this.redis.del(lockKey);
      }
    } else {
      // ไม่ได้ lock → รอและ retry
      return this.waitAndGet(cacheKey, fetcher);
    }
  }
  
  private async waitAndGet(
    cacheKey: string,
    fetcher: () => Promise<T>,
    maxRetries: number = 10,
    retryDelayMs: number = 200
  ): Promise<T> {
    for (let i = 0; i < maxRetries; i++) {
      await new Promise(resolve => setTimeout(resolve, retryDelayMs));
      
      const cached = await this.redis.get(cacheKey);
      if (cached !== null) {
        return JSON.parse(cached) as T;
      }
    }
    
    // ถ้ารอนานเกินไป ดึงข้อมูลเองเลย
    console.warn(`Cache lock wait timeout for ${cacheKey}, fetching directly`);
    return fetcher();
  }
}

// 5.2 Probabilistic Early Expiration (PER)
// ป้องกัน stampede โดย expire cache เร็วขึ้นแบบ probabilistic
export class ProbabilisticEarlyExpiration<T> {
  constructor(
    private redis: Redis,
    private keyPrefix: string,
    private ttlSeconds: number,
    private beta: number = 1.0 // ค่า beta สูง = expire เร็วขึ้น
  ) {}
  
  async get(
    key: string,
    fetcher: () => Promise<T>,
    fetchTimeMs?: number
  ): Promise<T> {
    const cacheKey = `${this.keyPrefix}:${key}`;
    const metaKey = `${cacheKey}:meta`;
    
    const [cached, meta] = await Promise.all([
      this.redis.get(cacheKey),
      this.redis.get(metaKey),
    ]);
    
    if (cached !== null && meta !== null) {
      const metadata = JSON.parse(meta);
      const remainingTtl = await this.redis.ttl(cacheKey);
      
      // PER formula: X - β × δ × ln(rand())
      // X = current TTL, β = beta param, δ = fetch time
      const delta = fetchTimeMs || metadata.fetchTimeMs || 100;
      const threshold = -this.beta * (delta / 1000) * Math.log(Math.random());
      
      if (remainingTtl > threshold) {
        // ยังไม่ต้อง refresh
        return JSON.parse(cached) as T;
      }
      // Probabilistically decide to refresh early
    }
    
    // Fetch และ cache
    const startTime = Date.now();
    const data = await fetcher();
    const fetchDuration = Date.now() - startTime;
    
    await Promise.all([
      this.redis.setex(cacheKey, this.ttlSeconds, JSON.stringify(data)),
      this.redis.setex(metaKey, this.ttlSeconds + 60, JSON.stringify({
        fetchTimeMs: fetchDuration,
        cachedAt: Date.now(),
      })),
    ]);
    
    return data;
  }
}
```

### 5.2 Stale-While-Revalidate Pattern

```typescript
// caching/stale-while-revalidate.ts
import Redis from 'ioredis';

interface SWRConfig {
  ttlSeconds: number;
  staleSeconds: number; // กี่วินาทีหลังจาก TTL ยังให้ stale data ได้
}

interface CacheEntry<T> {
  data: T;
  cachedAt: number;
  revalidating: boolean;
}

export class StaleWhileRevalidateCache<T> {
  private revalidationSet = new Set<string>();
  
  constructor(
    private redis: Redis,
    private keyPrefix: string,
    private config: SWRConfig
  ) {}
  
  async get(key: string, fetcher: () => Promise<T>): Promise<T> {
    const cacheKey = `${this.keyPrefix}:${key}`;
    const cached = await this.redis.get(cacheKey);
    
    if (cached === null) {
      // ไม่มี cache เลย → fetch ทันที
      const data = await this.fetchAndCache(key, fetcher);
      return data;
    }
    
    const entry = JSON.parse(cached) as CacheEntry<T>;
    const age = (Date.now() - entry.cachedAt) / 1000;
    
    if (age < this.config.ttlSeconds) {
      // Fresh data → return ทันที
      return entry.data;
    } else if (age < this.config.ttlSeconds + this.config.staleSeconds) {
      // Stale แต่ยังอยู่ในช่วง stale window → return stale + revalidate ใน background
      if (!this.revalidationSet.has(key)) {
        this.revalidationSet.add(key);
        // Revalidate ใน background
        this.fetchAndCache(key, fetcher)
          .catch(err => console.error(`SWR revalidation failed for ${key}:`, err))
          .finally(() => this.revalidationSet.delete(key));
      }
      return entry.data;
    } else {
      // Too stale → fetch แบบ blocking
      return this.fetchAndCache(key, fetcher);
    }
  }
  
  private async fetchAndCache(key: string, fetcher: () => Promise<T>): Promise<T> {
    const data = await fetcher();
    const cacheKey = `${this.keyPrefix}:${key}`;
    
    const entry: CacheEntry<T> = {
      data,
      cachedAt: Date.now(),
      revalidating: false,
    };
    
    // Store longer than TTL เพื่อให้มี stale window
    const totalTtl = this.config.ttlSeconds + this.config.staleSeconds;
    await this.redis.setex(cacheKey, totalTtl, JSON.stringify(entry));
    
    return data;
  }
}
```

---

## 6. Cache Invalidation Strategies

### 6.1 Event-Based Invalidation

```typescript
// caching/event-based-invalidation.ts
import { EventEmitter } from 'events';
import Redis from 'ioredis';

interface CacheInvalidationEvent {
  type: 'entity_updated' | 'entity_deleted' | 'bulk_update';
  entityType: string;
  entityId?: string;
  metadata?: Record<string, unknown>;
}

export class EventBasedCacheInvalidator {
  private subscriber: Redis;
  private publisher: Redis;
  private invalidationHandlers: Map<string, Set<() => Promise<void>>> = new Map();
  
  constructor(private redis: Redis) {
    this.subscriber = redis.duplicate();
    this.publisher = redis.duplicate();
    
    this.setupSubscription();
  }
  
  private setupSubscription(): void {
    this.subscriber.subscribe('cache:invalidate', (err) => {
      if (err) console.error('Subscribe error:', err);
    });
    
    this.subscriber.on('message', async (channel, message) => {
      if (channel !== 'cache:invalidate') return;
      
      try {
        const event = JSON.parse(message) as CacheInvalidationEvent;
        await this.processInvalidation(event);
      } catch (err) {
        console.error('Cache invalidation error:', err);
      }
    });
  }
  
  private async processInvalidation(event: CacheInvalidationEvent): Promise<void> {
    const handlers = this.invalidationHandlers.get(event.entityType) || new Set();
    
    for (const handler of handlers) {
      await handler();
    }
    
    // Invalidate keys ตาม pattern
    let pattern: string;
    switch (event.type) {
      case 'entity_updated':
      case 'entity_deleted':
        pattern = `${event.entityType}:${event.entityId}*`;
        break;
      case 'bulk_update':
        pattern = `${event.entityType}:*`;
        break;
    }
    
    const keys = await this.redis.keys(pattern);
    if (keys.length > 0) {
      await this.redis.del(...keys);
      console.log(`Cache invalidated: ${keys.length} keys matching ${pattern}`);
    }
  }
  
  async publishInvalidation(event: CacheInvalidationEvent): Promise<void> {
    await this.publisher.publish('cache:invalidate', JSON.stringify(event));
  }
  
  onInvalidation(entityType: string, handler: () => Promise<void>): void {
    if (!this.invalidationHandlers.has(entityType)) {
      this.invalidationHandlers.set(entityType, new Set());
    }
    this.invalidationHandlers.get(entityType)!.add(handler);
  }
}

// การใช้งาน
const invalidator = new EventBasedCacheInvalidator(redis);

// Subscribe ให้ Product service
invalidator.onInvalidation('product', async () => {
  // Clear product-related aggregations
  await redis.del('product:featured');
  await redis.del('product:trending');
});

// เมื่อ product ถูก update ใน service อื่น
await invalidator.publishInvalidation({
  type: 'entity_updated',
  entityType: 'product',
  entityId: 'prod_123',
});
```

### 6.2 Tag-Based Cache Invalidation

```typescript
// caching/tag-based-invalidation.ts
import Redis from 'ioredis';

export class TagBasedCache {
  constructor(private redis: Redis) {}
  
  async set(
    key: string,
    value: unknown,
    ttlSeconds: number,
    tags: string[]
  ): Promise<void> {
    const pipeline = this.redis.pipeline();
    
    // Store the value
    pipeline.setex(key, ttlSeconds, JSON.stringify(value));
    
    // Associate key with tags
    for (const tag of tags) {
      const tagKey = `tag:${tag}`;
      pipeline.sadd(tagKey, key);
      pipeline.expire(tagKey, ttlSeconds + 60);
    }
    
    // Store tags for this key
    const keyTagsKey = `keytags:${key}`;
    pipeline.sadd(keyTagsKey, ...tags);
    pipeline.expire(keyTagsKey, ttlSeconds + 60);
    
    await pipeline.exec();
  }
  
  async get(key: string): Promise<unknown | null> {
    const value = await this.redis.get(key);
    return value ? JSON.parse(value) : null;
  }
  
  async invalidateByTag(tag: string): Promise<number> {
    const tagKey = `tag:${tag}`;
    
    // ดึง keys ทั้งหมดที่ associate กับ tag
    const keys = await this.redis.smembers(tagKey);
    
    if (keys.length === 0) return 0;
    
    const pipeline = this.redis.pipeline();
    
    // ลบ keys
    for (const key of keys) {
      pipeline.del(key);
      pipeline.del(`keytags:${key}`);
    }
    
    // ลบ tag set
    pipeline.del(tagKey);
    
    await pipeline.exec();
    
    console.log(`Invalidated ${keys.length} cache entries for tag: ${tag}`);
    return keys.length;
  }
  
  async invalidateByTags(tags: string[]): Promise<number> {
    let totalInvalidated = 0;
    for (const tag of tags) {
      totalInvalidated += await this.invalidateByTag(tag);
    }
    return totalInvalidated;
  }
}

// ตัวอย่างการใช้งาน
const tagCache = new TagBasedCache(redis);

// Cache product ด้วย tags ต่างๆ
await tagCache.set(
  'product:123',
  { id: '123', name: 'Product A', categoryId: 'cat_1' },
  3600,
  ['product:123', 'category:cat_1', 'brand:brand_1']
);

await tagCache.set(
  'product:456',
  { id: '456', name: 'Product B', categoryId: 'cat_1' },
  3600,
  ['product:456', 'category:cat_1', 'brand:brand_2']
);

// เมื่อ Category 1 ถูก update ลบ cache ทุกอย่างที่เกี่ยวข้อง
await tagCache.invalidateByTag('category:cat_1');
// ลบทั้ง product:123 และ product:456
```

---

## 7. Redis Cluster Configuration

### 7.1 Redis Cluster Setup

```yaml
# docker-compose.redis-cluster.yaml
version: '3.8'

services:
  redis-node-1:
    image: redis:7-alpine
    command: redis-server --cluster-enabled yes --cluster-config-file nodes.conf --cluster-node-timeout 5000 --appendonly yes --port 7001
    ports:
    - "7001:7001"
    - "17001:17001"
    volumes:
    - redis-node-1-data:/data
    networks:
      redis-cluster:
        ipv4_address: 172.20.0.11

  redis-node-2:
    image: redis:7-alpine
    command: redis-server --cluster-enabled yes --cluster-config-file nodes.conf --cluster-node-timeout 5000 --appendonly yes --port 7002
    ports:
    - "7002:7002"
    - "17002:17002"
    volumes:
    - redis-node-2-data:/data
    networks:
      redis-cluster:
        ipv4_address: 172.20.0.12

  redis-node-3:
    image: redis:7-alpine
    command: redis-server --cluster-enabled yes --cluster-config-file nodes.conf --cluster-node-timeout 5000 --appendonly yes --port 7003
    ports:
    - "7003:7003"
    - "17003:17003"
    volumes:
    - redis-node-3-data:/data
    networks:
      redis-cluster:
        ipv4_address: 172.20.0.13

  redis-node-4:
    image: redis:7-alpine
    command: redis-server --cluster-enabled yes --cluster-config-file nodes.conf --cluster-node-timeout 5000 --appendonly yes --port 7004
    ports:
    - "7004:7004"
    - "17004:17004"
    volumes:
    - redis-node-4-data:/data
    networks:
      redis-cluster:
        ipv4_address: 172.20.0.14

  redis-node-5:
    image: redis:7-alpine
    command: redis-server --cluster-enabled yes --cluster-config-file nodes.conf --cluster-node-timeout 5000 --appendonly yes --port 7005
    ports:
    - "7005:7005"
    - "17005:17005"
    volumes:
    - redis-node-5-data:/data
    networks:
      redis-cluster:
        ipv4_address: 172.20.0.15

  redis-node-6:
    image: redis:7-alpine
    command: redis-server --cluster-enabled yes --cluster-config-file nodes.conf --cluster-node-timeout 5000 --appendonly yes --port 7006
    ports:
    - "7006:7006"
    - "17006:17006"
    volumes:
    - redis-node-6-data:/data
    networks:
      redis-cluster:
        ipv4_address: 172.20.0.16

  redis-cluster-init:
    image: redis:7-alpine
    command: >
      sh -c "redis-cli --cluster create
        172.20.0.11:7001 172.20.0.12:7002 172.20.0.13:7003
        172.20.0.14:7004 172.20.0.15:7005 172.20.0.16:7006
        --cluster-replicas 1 --cluster-yes"
    depends_on:
    - redis-node-1
    - redis-node-2
    - redis-node-3
    - redis-node-4
    - redis-node-5
    - redis-node-6
    networks:
    - redis-cluster

volumes:
  redis-node-1-data:
  redis-node-2-data:
  redis-node-3-data:
  redis-node-4-data:
  redis-node-5-data:
  redis-node-6-data:

networks:
  redis-cluster:
    driver: bridge
    ipam:
      config:
      - subnet: 172.20.0.0/24
```

### 7.2 Redis Cluster Client

```typescript
// caching/redis-cluster-client.ts
import Redis, { Cluster } from 'ioredis';
import { Registry, Counter, Histogram } from 'prom-client';

interface ClusterConfig {
  nodes: Array<{ host: string; port: number }>;
  options?: {
    maxRedirections?: number;
    retryDelayOnFailover?: number;
    retryDelayOnClusterDown?: number;
    retryDelayOnTryAgain?: number;
    slotsRefreshTimeout?: number;
  };
}

export class RedisClusterClient {
  private cluster: Cluster;
  private metrics: {
    commands: Counter;
    errors: Counter;
    latency: Histogram;
  };
  
  constructor(config: ClusterConfig, registry: Registry) {
    this.cluster = new Redis.Cluster(config.nodes, {
      redisOptions: {
        connectTimeout: 10000,
        commandTimeout: 3000,
        enableReadyCheck: true,
        maxRetriesPerRequest: 3,
      },
      clusterRetryStrategy: (times: number) => {
        if (times > 3) return null; // หยุด retry
        return Math.min(times * 100, 2000);
      },
      maxRedirections: config.options?.maxRedirections || 16,
      retryDelayOnFailover: config.options?.retryDelayOnFailover || 100,
      retryDelayOnClusterDown: config.options?.retryDelayOnClusterDown || 300,
      retryDelayOnTryAgain: config.options?.retryDelayOnTryAgain || 100,
      slotsRefreshTimeout: config.options?.slotsRefreshTimeout || 2000,
    });
    
    this.setupEventHandlers();
    this.initializeMetrics(registry);
  }
  
  private setupEventHandlers(): void {
    this.cluster.on('connect', () => {
      console.log('Redis Cluster connected');
    });
    
    this.cluster.on('error', (err) => {
      console.error('Redis Cluster error:', err);
      this.metrics.errors.inc({ type: 'cluster_error' });
    });
    
    this.cluster.on('node error', (err, address) => {
      console.error(`Redis Cluster node error at ${address}:`, err);
      this.metrics.errors.inc({ type: 'node_error', address });
    });
    
    this.cluster.on('+node', (node) => {
      console.log(`Node added to cluster: ${node.options.host}:${node.options.port}`);
    });
    
    this.cluster.on('-node', (node) => {
      console.log(`Node removed from cluster: ${node.options.host}:${node.options.port}`);
    });
  }
  
  private initializeMetrics(registry: Registry): void {
    this.metrics = {
      commands: new Counter({
        name: 'redis_cluster_commands_total',
        help: 'Total Redis cluster commands',
        labelNames: ['command', 'status'],
        registers: [registry],
      }),
      errors: new Counter({
        name: 'redis_cluster_errors_total',
        help: 'Total Redis cluster errors',
        labelNames: ['type', 'address'],
        registers: [registry],
      }),
      latency: new Histogram({
        name: 'redis_cluster_latency_seconds',
        help: 'Redis cluster command latency',
        labelNames: ['command'],
        buckets: [0.001, 0.005, 0.01, 0.05, 0.1, 0.5],
        registers: [registry],
      }),
    };
  }
  
  async get(key: string): Promise<string | null> {
    const end = this.metrics.latency.startTimer({ command: 'get' });
    try {
      const result = await this.cluster.get(key);
      this.metrics.commands.inc({ command: 'get', status: 'success' });
      return result;
    } catch (err) {
      this.metrics.commands.inc({ command: 'get', status: 'error' });
      throw err;
    } finally {
      end();
    }
  }
  
  async set(key: string, value: string, ttlSeconds?: number): Promise<void> {
    const end = this.metrics.latency.startTimer({ command: 'set' });
    try {
      if (ttlSeconds) {
        await this.cluster.setex(key, ttlSeconds, value);
      } else {
        await this.cluster.set(key, value);
      }
      this.metrics.commands.inc({ command: 'set', status: 'success' });
    } catch (err) {
      this.metrics.commands.inc({ command: 'set', status: 'error' });
      throw err;
    } finally {
      end();
    }
  }
  
  // Multi-key operations (ต้องระวัง hash slots)
  async mget(keys: string[]): Promise<(string | null)[]> {
    // ใน cluster mode, MGET ทำงานได้เฉพาะ keys ที่อยู่ใน slot เดียวกัน
    // ใช้ pipeline แทน
    const pipeline = this.cluster.pipeline();
    keys.forEach(key => pipeline.get(key));
    const results = await pipeline.exec();
    return results?.map(([err, val]) => err ? null : val as string | null) || [];
  }
  
  async getClient(): Promise<Cluster> {
    return this.cluster;
  }
  
  async quit(): Promise<void> {
    await this.cluster.quit();
  }
}
```

---

## 8. Cache Warming

Cache warming คือการ pre-populate cache ก่อนที่ traffic จะมาถึง

```typescript
// caching/cache-warmer.ts
import { Pool } from 'pg';
import Redis from 'ioredis';

interface WarmupConfig {
  batchSize: number;
  concurrency: number;
  keyPrefix: string;
  ttlSeconds: number;
}

export class CacheWarmer {
  constructor(
    private redis: Redis,
    private db: Pool,
    private config: WarmupConfig
  ) {}
  
  async warmProducts(limit: number = 10000): Promise<void> {
    console.log(`Starting cache warm-up for ${limit} products...`);
    const startTime = Date.now();
    let processed = 0;
    let offset = 0;
    
    while (offset < limit) {
      const batch = await this.fetchProductBatch(offset, this.config.batchSize);
      if (batch.length === 0) break;
      
      await this.cacheProductBatch(batch);
      
      processed += batch.length;
      offset += this.config.batchSize;
      
      if (processed % 1000 === 0) {
        const elapsed = (Date.now() - startTime) / 1000;
        const rate = processed / elapsed;
        console.log(`Warmed up ${processed} products (${rate.toFixed(0)}/s)`);
      }
    }
    
    const totalTime = (Date.now() - startTime) / 1000;
    console.log(`Cache warm-up complete: ${processed} products in ${totalTime.toFixed(1)}s`);
  }
  
  private async fetchProductBatch(
    offset: number,
    limit: number
  ): Promise<Product[]> {
    const result = await this.db.query(
      `SELECT p.*, 
              c.name as category_name,
              COUNT(r.id) as review_count,
              AVG(r.rating) as avg_rating
       FROM products p
       LEFT JOIN categories c ON c.id = p.category_id
       LEFT JOIN reviews r ON r.product_id = p.id
       WHERE p.is_active = true
       GROUP BY p.id, c.name
       ORDER BY p.updated_at DESC
       LIMIT $1 OFFSET $2`,
      [limit, offset]
    );
    return result.rows;
  }
  
  private async cacheProductBatch(products: Product[]): Promise<void> {
    const pipeline = this.redis.pipeline();
    
    for (const product of products) {
      const key = `${this.config.keyPrefix}:${product.id}`;
      pipeline.setex(key, this.config.ttlSeconds, JSON.stringify(product));
    }
    
    await pipeline.exec();
  }
  
  async warmTopProducts(topN: number = 100): Promise<void> {
    const result = await this.db.query(
      `SELECT p.*, 
              SUM(oi.quantity) as total_sold
       FROM products p
       JOIN order_items oi ON oi.product_id = p.id
       JOIN orders o ON o.id = oi.order_id
       WHERE o.created_at > NOW() - INTERVAL '30 days'
       GROUP BY p.id
       ORDER BY total_sold DESC
       LIMIT $1`,
      [topN]
    );
    
    await this.cacheProductBatch(result.rows);
    console.log(`Warmed up top ${result.rows.length} products`);
  }
  
  async schedulePeriodicWarmup(intervalHours: number = 6): void {
    const warmup = async () => {
      try {
        await this.warmTopProducts(200);
      } catch (err) {
        console.error('Periodic cache warmup failed:', err);
      }
    };
    
    // ทำทันที
    await warmup();
    
    // ทำซ้ำตาม interval
    setInterval(warmup, intervalHours * 60 * 60 * 1000);
  }
}

interface Product {
  id: string;
  name: string;
  price: number;
  categoryId: string;
  [key: string]: unknown;
}
```

---

## 9. Cache Observability

### 9.1 Cache Metrics Collector

```typescript
// caching/cache-metrics.ts
import Redis from 'ioredis';
import { Registry, Gauge, Counter, Histogram } from 'prom-client';

export class CacheMetricsCollector {
  private registry: Registry;
  private hitRatio: Gauge;
  private memoryUsage: Gauge;
  private keyCount: Gauge;
  private evictions: Gauge;
  private commands: Counter;
  private latency: Histogram;
  
  constructor(private redis: Redis) {
    this.registry = new Registry();
    this.initializeMetrics();
    this.startCollection();
  }
  
  private initializeMetrics(): void {
    this.hitRatio = new Gauge({
      name: 'redis_cache_hit_ratio',
      help: 'Cache hit ratio (0-1)',
      registers: [this.registry],
    });
    
    this.memoryUsage = new Gauge({
      name: 'redis_memory_used_bytes',
      help: 'Redis memory usage in bytes',
      registers: [this.registry],
    });
    
    this.keyCount = new Gauge({
      name: 'redis_key_count',
      help: 'Number of keys in Redis',
      labelNames: ['db'],
      registers: [this.registry],
    });
    
    this.evictions = new Gauge({
      name: 'redis_evictions_total',
      help: 'Total evicted keys',
      registers: [this.registry],
    });
    
    this.commands = new Counter({
      name: 'redis_commands_processed_total',
      help: 'Total Redis commands processed',
      registers: [this.registry],
    });
    
    this.latency = new Histogram({
      name: 'redis_command_latency_seconds',
      help: 'Redis command latency',
      buckets: [0.0001, 0.001, 0.005, 0.01, 0.05, 0.1],
      registers: [this.registry],
    });
  }
  
  private startCollection(): void {
    setInterval(async () => {
      await this.collectMetrics();
    }, 10000); // ทุก 10 วินาที
  }
  
  private async collectMetrics(): Promise<void> {
    try {
      const info = await this.redis.info();
      const stats = this.parseRedisInfo(info);
      
      // Hit ratio
      const hits = parseInt(stats['keyspace_hits'] || '0');
      const misses = parseInt(stats['keyspace_misses'] || '0');
      const total = hits + misses;
      if (total > 0) {
        this.hitRatio.set(hits / total);
      }
      
      // Memory
      const memUsed = parseInt(stats['used_memory'] || '0');
      this.memoryUsage.set(memUsed);
      
      // Evictions
      const evictions = parseInt(stats['evicted_keys'] || '0');
      this.evictions.set(evictions);
      
      // Commands
      const cmdCount = parseInt(stats['total_commands_processed'] || '0');
      
      // Key count per DB
      const dbPattern = /db(\d+):keys=(\d+)/g;
      let match;
      while ((match = dbPattern.exec(info)) !== null) {
        this.keyCount.set({ db: match[1] }, parseInt(match[2]));
      }
    } catch (err) {
      console.error('Cache metrics collection failed:', err);
    }
  }
  
  private parseRedisInfo(info: string): Record<string, string> {
    const result: Record<string, string> = {};
    for (const line of info.split('\r\n')) {
      const colonIndex = line.indexOf(':');
      if (colonIndex !== -1) {
        const key = line.slice(0, colonIndex).trim();
        const value = line.slice(colonIndex + 1).trim();
        result[key] = value;
      }
    }
    return result;
  }
  
  recordLatency(command: string, latencyMs: number): void {
    this.latency.observe(latencyMs / 1000);
  }
  
  getMetrics(): Promise<string> {
    return this.registry.metrics();
  }
}
```

### 9.2 Cache Health Check

```typescript
// caching/cache-health.ts
import Redis from 'ioredis';

interface HealthCheckResult {
  status: 'healthy' | 'degraded' | 'unhealthy';
  latencyMs: number;
  hitRatio: number;
  memoryUsagePercent: number;
  connectedClients: number;
  details: string[];
}

export class CacheHealthChecker {
  constructor(private redis: Redis) {}
  
  async check(): Promise<HealthCheckResult> {
    const details: string[] = [];
    let status: 'healthy' | 'degraded' | 'unhealthy' = 'healthy';
    
    // Latency check
    const start = Date.now();
    await this.redis.ping();
    const latencyMs = Date.now() - start;
    
    if (latencyMs > 100) {
      status = 'degraded';
      details.push(`High latency: ${latencyMs}ms`);
    } else if (latencyMs > 500) {
      status = 'unhealthy';
      details.push(`Very high latency: ${latencyMs}ms`);
    }
    
    // Memory check
    const info = await this.redis.info('memory');
    const memStats = this.parseInfo(info);
    const usedMemory = parseInt(memStats['used_memory'] || '0');
    const maxMemory = parseInt(memStats['maxmemory'] || '0');
    const memoryUsagePercent = maxMemory > 0 ? (usedMemory / maxMemory) * 100 : 0;
    
    if (memoryUsagePercent > 90) {
      status = 'unhealthy';
      details.push(`Memory usage critical: ${memoryUsagePercent.toFixed(1)}%`);
    } else if (memoryUsagePercent > 75) {
      status = status === 'healthy' ? 'degraded' : status;
      details.push(`Memory usage high: ${memoryUsagePercent.toFixed(1)}%`);
    }
    
    // Hit ratio check
    const statsInfo = await this.redis.info('stats');
    const statsData = this.parseInfo(statsInfo);
    const hits = parseInt(statsData['keyspace_hits'] || '0');
    const misses = parseInt(statsData['keyspace_misses'] || '0');
    const total = hits + misses;
    const hitRatio = total > 0 ? hits / total : 1;
    
    if (hitRatio < 0.5) {
      status = status === 'healthy' ? 'degraded' : status;
      details.push(`Low hit ratio: ${(hitRatio * 100).toFixed(1)}%`);
    }
    
    // Connected clients
    const clientInfo = await this.redis.info('clients');
    const clientData = this.parseInfo(clientInfo);
    const connectedClients = parseInt(clientData['connected_clients'] || '0');
    
    if (connectedClients > 1000) {
      status = status === 'healthy' ? 'degraded' : status;
      details.push(`High connection count: ${connectedClients}`);
    }
    
    return {
      status,
      latencyMs,
      hitRatio,
      memoryUsagePercent,
      connectedClients,
      details,
    };
  }
  
  private parseInfo(info: string): Record<string, string> {
    const result: Record<string, string> = {};
    for (const line of info.split('\r\n')) {
      const colonIndex = line.indexOf(':');
      if (colonIndex !== -1) {
        result[line.slice(0, colonIndex)] = line.slice(colonIndex + 1);
      }
    }
    return result;
  }
}
```

---

## 10. Production Configuration

### 10.1 Redis Configuration สำหรับ Production

```conf
# redis/redis.conf

# Network
bind 0.0.0.0
port 6379
protected-mode yes
requirepass your-strong-password-here

# Memory
maxmemory 4gb
maxmemory-policy allkeys-lru
maxmemory-samples 10

# Persistence
save 900 1
save 300 10
save 60 10000
stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
dbfilename dump.rdb

# AOF
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

# Cluster (ถ้าใช้ cluster mode)
# cluster-enabled yes
# cluster-config-file nodes.conf
# cluster-node-timeout 15000

# Latency
latency-monitor-threshold 100
latency-history-event-backlog 256

# Slow log
slowlog-log-slower-than 10000
slowlog-max-len 128

# Security
rename-command FLUSHDB ""
rename-command FLUSHALL ""
rename-command CONFIG "CONFIG_ADMIN_ONLY"
rename-command DEBUG ""
rename-command SHUTDOWN "SHUTDOWN_ADMIN_ONLY"

# TLS (สำหรับ production)
# tls-port 6380
# tls-cert-file /etc/ssl/redis/redis.crt
# tls-key-file /etc/ssl/redis/redis.key
# tls-ca-cert-file /etc/ssl/redis/ca.crt
# tls-auth-clients yes
```

### 10.2 Kubernetes ConfigMap สำหรับ Redis

```yaml
# k8s/redis/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-config
  namespace: production
data:
  redis.conf: |
    maxmemory 2gb
    maxmemory-policy allkeys-lru
    appendonly yes
    appendfsync everysec
    slowlog-log-slower-than 10000
    latency-monitor-threshold 50
    
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
  namespace: production
spec:
  serviceName: redis
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9121"
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        command:
        - redis-server
        - /etc/redis/redis.conf
        ports:
        - containerPort: 6379
        resources:
          requests:
            cpu: 500m
            memory: 2Gi
          limits:
            cpu: 2
            memory: 4Gi
        volumeMounts:
        - name: config
          mountPath: /etc/redis
        - name: data
          mountPath: /data
        livenessProbe:
          exec:
            command: [redis-cli, ping]
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command: [redis-cli, ping]
          initialDelaySeconds: 5
          periodSeconds: 5
      - name: redis-exporter
        image: oliver006/redis_exporter:latest
        ports:
        - containerPort: 9121
          name: metrics
        env:
        - name: REDIS_ADDR
          value: redis://localhost:6379
        resources:
          requests:
            cpu: 100m
            memory: 64Mi
          limits:
            cpu: 200m
            memory: 128Mi
      volumes:
      - name: config
        configMap:
          name: redis-config
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ReadWriteOnce]
      storageClassName: gp3
      resources:
        requests:
          storage: 20Gi
```

---

## สรุป Cache Pattern Selection Guide

| Pattern | เมื่อใช้ | ข้อดี | ข้อเสีย |
|---------|---------|-------|---------|
| Cache-Aside | Read-heavy, tolerance สำหรับ stale data | ง่าย, resilient | Cache miss penalty |
| Write-Through | Read-heavy, data consistency สำคัญ | ไม่มี stale data | Write latency เพิ่มขึ้น |
| Write-Behind | Write-heavy, eventual consistency OK | Write performance สูง | Risk of data loss |
| Read-Through | ไม่ต้องการจัดการ cache โดยตรง | Simple application code | Harder to test |
| Multi-Level | Very high performance ต้องการ | Cache hit ratio สูงมาก | Complexity สูง |
| SWR | API responses, UX-first | Fast UX, background refresh | Stale data บางครั้ง |

### แนวทางการเลือก

1. **E-commerce product pages** → Cache-Aside + SWR + Multi-Level
2. **User sessions** → Write-Through (consistency critical)
3. **Analytics counters** → Write-Behind (write-heavy)
4. **API rate limiting** → Redis native counters
5. **Search results** → Cache-Aside + Tag-based invalidation

### Production Checklist

- [ ] กำหนด maxmemory และ eviction policy
- [ ] เปิด persistence (AOF + RDB)
- [ ] ตั้ง Cache Stampede prevention
- [ ] Monitor hit ratio (ควร >80%)
- [ ] ตั้ง alert สำหรับ memory usage >75%
- [ ] Cache warming สำหรับ cold start
- [ ] Tag-based invalidation สำหรับ complex entities
- [ ] mTLS สำหรับ Redis connection ใน production
- [ ] Disable FLUSHALL/FLUSHDB ใน production

# Part 78: Microservices Scalability Patterns

## บทนำ

Scalability เป็นความสามารถของระบบในการรองรับ load ที่เพิ่มขึ้น บทนี้จะครอบคลุม Horizontal vs Vertical Scaling, Stateless Service Design, Session Management ใน Distributed Systems, Read/Write Splitting, Database Federation, Message Queue Scaling และ Auto-scaling Strategies

---

## 1. Horizontal vs Vertical Scaling

### 1.1 Horizontal Scaling Design Principles

```typescript
// src/patterns/scalability/stateless-service.ts
import { createClient } from 'redis';
import { Pool } from 'pg';

// ✅ Stateless service design - ไม่เก็บ state ใน memory
export class ProductService {
  constructor(
    private db: Pool,
    private cache: ReturnType<typeof createClient>,
    private eventBus: EventBus
  ) {}

  // ✅ Each request is independent - ไม่ depend on previous requests
  async getProduct(id: string, requestContext: RequestContext): Promise<Product> {
    const cacheKey = `product:${id}:${requestContext.language}`;
    
    // Try distributed cache (accessible from all instances)
    const cached = await this.cache.get(cacheKey);
    if (cached) {
      return JSON.parse(cached);
    }

    const { rows } = await this.db.query(
      `SELECT p.*, 
        t.title as translated_title,
        t.description as translated_description
       FROM products p
       LEFT JOIN product_translations t ON t.product_id = p.id AND t.language = $2
       WHERE p.id = $1`,
      [id, requestContext.language]
    );

    if (rows.length === 0) {
      throw new NotFoundError(`Product ${id} not found`);
    }

    const product = this.mapProduct(rows[0]);
    
    // Store in distributed cache (TTL = 5 minutes)
    await this.cache.setEx(cacheKey, 300, JSON.stringify(product));
    
    return product;
  }

  // ✅ Idempotent operations - safe to retry
  async updateInventory(
    productId: string,
    delta: number,
    idempotencyKey: string
  ): Promise<InventoryResult> {
    // Check if operation already processed
    const existingResult = await this.cache.get(`idempotency:${idempotencyKey}`);
    if (existingResult) {
      return JSON.parse(existingResult);
    }

    const client = await this.db.connect();
    try {
      await client.query('BEGIN');

      const { rows } = await client.query(
        `UPDATE products 
         SET stock_quantity = stock_quantity + $1,
             updated_at = NOW()
         WHERE id = $2 AND (stock_quantity + $1) >= 0
         RETURNING id, stock_quantity`,
        [delta, productId]
      );

      if (rows.length === 0) {
        await client.query('ROLLBACK');
        throw new InsufficientStockError(`Insufficient stock for product ${productId}`);
      }

      await client.query('COMMIT');

      const result: InventoryResult = {
        productId,
        newQuantity: rows[0].stock_quantity,
        delta,
        timestamp: new Date(),
      };

      // Store result for idempotency (24 hours)
      await this.cache.setEx(`idempotency:${idempotencyKey}`, 86400, JSON.stringify(result));

      await this.eventBus.publish('inventory.updated', result);
      return result;
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }

  private mapProduct(row: any): Product {
    return {
      id: row.id,
      title: row.translated_title || row.title,
      description: row.translated_description || row.description,
      price: parseFloat(row.price),
      stockQuantity: row.stock_quantity,
      category: row.category_id,
      updatedAt: row.updated_at,
    };
  }
}
```

---

## 2. Session Management ใน Distributed Systems

### 2.1 Distributed Session Store

```typescript
// src/session/distributed-session.ts
import { Redis } from 'ioredis';
import { createHash, randomBytes } from 'crypto';
import { Request, Response, NextFunction } from 'express';

interface SessionData {
  userId: string;
  roles: string[];
  permissions: string[];
  createdAt: number;
  lastAccessedAt: number;
  expiresAt: number;
  metadata: Record<string, any>;
  csrfToken: string;
}

interface SessionConfig {
  ttlSeconds: number;
  renewalThresholdSeconds: number;
  maxSessionsPerUser: number;
  cookieName: string;
  cookieSecure: boolean;
  cookieSameSite: 'strict' | 'lax' | 'none';
}

export class DistributedSessionStore {
  private readonly PREFIX = 'session:';
  private readonly USER_SESSIONS_PREFIX = 'user:sessions:';

  constructor(
    private redis: Redis,
    private config: SessionConfig
  ) {}

  async create(userId: string, roles: string[], metadata: Record<string, any> = {}): Promise<string> {
    // Enforce max sessions per user
    await this.enforceSessionLimit(userId);

    const sessionId = this.generateSessionId();
    const now = Date.now();
    const csrfToken = randomBytes(32).toString('hex');

    const session: SessionData = {
      userId,
      roles,
      permissions: this.expandPermissions(roles),
      createdAt: now,
      lastAccessedAt: now,
      expiresAt: now + this.config.ttlSeconds * 1000,
      metadata,
      csrfToken,
    };

    const pipeline = this.redis.pipeline();
    pipeline.set(
      `${this.PREFIX}${sessionId}`,
      JSON.stringify(session),
      'EX',
      this.config.ttlSeconds
    );
    pipeline.sadd(`${this.USER_SESSIONS_PREFIX}${userId}`, sessionId);
    pipeline.expire(`${this.USER_SESSIONS_PREFIX}${userId}`, this.config.ttlSeconds * 2);
    await pipeline.exec();

    return sessionId;
  }

  async get(sessionId: string): Promise<SessionData | null> {
    const data = await this.redis.get(`${this.PREFIX}${sessionId}`);
    if (!data) return null;

    const session: SessionData = JSON.parse(data);
    const now = Date.now();

    // Check expiry
    if (session.expiresAt < now) {
      await this.destroy(sessionId);
      return null;
    }

    // Auto-renew if close to expiry
    const timeToExpiry = session.expiresAt - now;
    if (timeToExpiry < this.config.renewalThresholdSeconds * 1000) {
      session.lastAccessedAt = now;
      session.expiresAt = now + this.config.ttlSeconds * 1000;
      await this.redis.set(
        `${this.PREFIX}${sessionId}`,
        JSON.stringify(session),
        'EX',
        this.config.ttlSeconds
      );
    }

    return session;
  }

  async update(sessionId: string, updates: Partial<SessionData>): Promise<void> {
    const session = await this.get(sessionId);
    if (!session) throw new Error('Session not found');

    const updated = { ...session, ...updates, lastAccessedAt: Date.now() };
    const ttl = await this.redis.ttl(`${this.PREFIX}${sessionId}`);

    await this.redis.set(
      `${this.PREFIX}${sessionId}`,
      JSON.stringify(updated),
      'EX',
      ttl > 0 ? ttl : this.config.ttlSeconds
    );
  }

  async destroy(sessionId: string): Promise<void> {
    const session = await this.get(sessionId);
    
    const pipeline = this.redis.pipeline();
    pipeline.del(`${this.PREFIX}${sessionId}`);
    if (session) {
      pipeline.srem(`${this.USER_SESSIONS_PREFIX}${session.userId}`, sessionId);
    }
    await pipeline.exec();
  }

  async destroyAllUserSessions(userId: string): Promise<number> {
    const sessionIds = await this.redis.smembers(`${this.USER_SESSIONS_PREFIX}${userId}`);
    
    const pipeline = this.redis.pipeline();
    for (const id of sessionIds) {
      pipeline.del(`${this.PREFIX}${id}`);
    }
    pipeline.del(`${this.USER_SESSIONS_PREFIX}${userId}`);
    await pipeline.exec();

    return sessionIds.length;
  }

  private async enforceSessionLimit(userId: string): Promise<void> {
    const sessions = await this.redis.smembers(`${this.USER_SESSIONS_PREFIX}${userId}`);
    
    if (sessions.length >= this.config.maxSessionsPerUser) {
      // Get all sessions and remove oldest
      const sessionData = await Promise.all(
        sessions.map(async (id) => {
          const data = await this.redis.get(`${this.PREFIX}${id}`);
          return { id, data: data ? JSON.parse(data) : null };
        })
      );

      const valid = sessionData.filter((s) => s.data !== null);
      valid.sort((a, b) => a.data.lastAccessedAt - b.data.lastAccessedAt);

      // Remove oldest sessions
      const toRemove = valid.slice(0, valid.length - this.config.maxSessionsPerUser + 1);
      for (const session of toRemove) {
        await this.destroy(session.id);
      }
    }
  }

  private generateSessionId(): string {
    return randomBytes(32).toString('base64url');
  }

  private expandPermissions(roles: string[]): string[] {
    const rolePermissions: Record<string, string[]> = {
      admin: ['*'],
      user: ['products:read', 'orders:read', 'orders:write', 'profile:read', 'profile:write'],
      premium: ['products:read', 'orders:read', 'orders:write', 'profile:read', 'profile:write', 'analytics:read'],
    };

    const permissions = new Set<string>();
    for (const role of roles) {
      const perms = rolePermissions[role] || [];
      perms.forEach((p) => permissions.add(p));
    }
    return Array.from(permissions);
  }
}

// Session Middleware
export function sessionMiddleware(store: DistributedSessionStore, config: SessionConfig) {
  return async (req: Request, res: Response, next: NextFunction) => {
    const sessionId = req.cookies?.[config.cookieName] ||
                      req.headers['x-session-id'] as string;

    if (!sessionId) return next();

    const session = await store.get(sessionId);
    if (!session) {
      res.clearCookie(config.cookieName);
      return next();
    }

    (req as any).session = session;
    (req as any).sessionId = sessionId;
    (req as any).user = {
      id: session.userId,
      roles: session.roles,
      permissions: session.permissions,
    };

    next();
  };
}
```

---

## 3. Read/Write Splitting

### 3.1 Database Read/Write Splitter

```typescript
// src/database/read-write-splitter.ts
import { Pool, PoolClient } from 'pg';
import { EventEmitter } from 'events';

interface ReplicaConfig {
  id: string;
  host: string;
  port: number;
  database: string;
  user: string;
  password: string;
  weight: number;  // Load balancing weight
  maxConnections: number;
  readOnly: boolean;
}

interface SplitterConfig {
  primary: Omit<ReplicaConfig, 'readOnly' | 'weight'>;
  replicas: ReplicaConfig[];
  replicaLagThresholdMs: number;
  healthCheckIntervalMs: number;
}

export class ReadWriteSplitter extends EventEmitter {
  private primaryPool: Pool;
  private replicaPools: Map<string, { pool: Pool; config: ReplicaConfig; healthy: boolean }> = new Map();
  private replicaLag: Map<string, number> = new Map();
  private roundRobinIndex = 0;

  constructor(private config: SplitterConfig) {
    super();
    this.primaryPool = this.createPool(config.primary, false);
    
    for (const replica of config.replicas) {
      const pool = this.createPool(replica, true);
      this.replicaPools.set(replica.id, { pool, config: replica, healthy: true });
    }

    this.startHealthChecks();
  }

  // Write operations — always go to primary
  async write<T>(fn: (client: PoolClient) => Promise<T>): Promise<T> {
    const client = await this.primaryPool.connect();
    try {
      const result = await fn(client);
      return result;
    } finally {
      client.release();
    }
  }

  // Read operations — route to healthy replica
  async read<T>(fn: (client: PoolClient) => Promise<T>, options: ReadOptions = {}): Promise<T> {
    if (options.consistencyRequired) {
      // Strong consistency: read from primary
      return this.write(fn);
    }

    const replica = this.selectReplica(options.preferredRegion);
    if (!replica) {
      // Fallback to primary if no healthy replica
      return this.write(fn);
    }

    const client = await replica.pool.connect();
    try {
      return await fn(client);
    } catch (error) {
      // On error, fallback to primary
      client.release();
      this.emit('replica:error', { replicaId: replica.config.id, error });
      return this.write(fn);
    } finally {
      try { client.release(); } catch {}
    }
  }

  // Transaction — always use primary
  async transaction<T>(fn: (client: PoolClient) => Promise<T>): Promise<T> {
    const client = await this.primaryPool.connect();
    try {
      await client.query('BEGIN');
      const result = await fn(client);
      await client.query('COMMIT');
      return result;
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }

  private selectReplica(preferredRegion?: string): { pool: Pool; config: ReplicaConfig } | null {
    const healthyReplicas = Array.from(this.replicaPools.values()).filter((r) => {
      if (!r.healthy) return false;
      const lag = this.replicaLag.get(r.config.id) || 0;
      return lag < this.config.replicaLagThresholdMs;
    });

    if (healthyReplicas.length === 0) return null;

    // Prefer specific region if requested
    if (preferredRegion) {
      const regionReplica = healthyReplicas.find(
        (r) => r.config.id.includes(preferredRegion)
      );
      if (regionReplica) return regionReplica;
    }

    // Weighted round-robin selection
    const totalWeight = healthyReplicas.reduce((sum, r) => sum + r.config.weight, 0);
    let random = Math.random() * totalWeight;

    for (const replica of healthyReplicas) {
      random -= replica.config.weight;
      if (random <= 0) return replica;
    }

    return healthyReplicas[this.roundRobinIndex++ % healthyReplicas.length];
  }

  private async startHealthChecks(): Promise<void> {
    setInterval(async () => {
      for (const [id, replica] of this.replicaPools) {
        try {
          const start = Date.now();
          const client = await replica.pool.connect();
          
          try {
            // Check replica lag
            const { rows } = await client.query(
              `SELECT EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp())) * 1000 AS lag_ms`
            );
            const lagMs = parseFloat(rows[0]?.lag_ms || '0');
            this.replicaLag.set(id, lagMs);

            if (!replica.healthy) {
              replica.healthy = true;
              this.emit('replica:recovered', { replicaId: id });
            }
          } finally {
            client.release();
          }
        } catch (error) {
          if (replica.healthy) {
            replica.healthy = false;
            this.emit('replica:unhealthy', { replicaId: id, error });
          }
        }
      }
    }, this.config.healthCheckIntervalMs);
  }

  private createPool(config: any, readOnly: boolean): Pool {
    return new Pool({
      host: config.host,
      port: config.port || 5432,
      database: config.database,
      user: config.user,
      password: config.password,
      max: config.maxConnections || 20,
      idleTimeoutMillis: 10000,
      connectionTimeoutMillis: 5000,
    });
  }
}

interface ReadOptions {
  consistencyRequired?: boolean;
  preferredRegion?: string;
}
```

---

## 4. Database Federation

### 4.1 Database Sharding Strategy

```typescript
// src/database/sharding/shard-manager.ts
import { Pool } from 'pg';
import { createHash } from 'crypto';

interface ShardConfig {
  id: string;
  host: string;
  port: number;
  database: string;
  user: string;
  password: string;
  rangeStart?: number;
  rangeEnd?: number;
  hashMod?: number;
  region?: string;
}

type ShardStrategy = 'hash' | 'range' | 'geographic' | 'consistent-hash';

export class ShardManager {
  private shards: Map<string, Pool> = new Map();
  private consistentHashRing: ConsistentHashRing;

  constructor(
    private shardConfigs: ShardConfig[],
    private strategy: ShardStrategy = 'consistent-hash'
  ) {
    for (const config of shardConfigs) {
      this.shards.set(config.id, new Pool({
        host: config.host,
        port: config.port,
        database: config.database,
        user: config.user,
        password: config.password,
        max: 20,
      }));
    }

    if (strategy === 'consistent-hash') {
      this.consistentHashRing = new ConsistentHashRing(
        shardConfigs.map((c) => c.id),
        150 // virtual nodes per shard
      );
    }
  }

  getShardForKey(key: string): Pool {
    const shardId = this.resolveShardId(key);
    const pool = this.shards.get(shardId);
    if (!pool) {
      throw new Error(`No shard found for key: ${key} -> shardId: ${shardId}`);
    }
    return pool;
  }

  getShardForRange(value: number): Pool {
    const config = this.shardConfigs.find(
      (c) => c.rangeStart !== undefined &&
             c.rangeEnd !== undefined &&
             value >= c.rangeStart &&
             value < c.rangeEnd
    );
    if (!config) throw new Error(`No shard for range value: ${value}`);
    return this.shards.get(config.id)!;
  }

  getShardForRegion(region: string): Pool {
    const config = this.shardConfigs.find((c) => c.region === region);
    if (!config) {
      // Fallback to default region
      return this.getShardForKey(region);
    }
    return this.shards.get(config.id)!;
  }

  async executeOnAllShards<T>(
    fn: (pool: Pool, shardId: string) => Promise<T>
  ): Promise<Array<{ shardId: string; result: T }>> {
    const results = await Promise.allSettled(
      Array.from(this.shards.entries()).map(async ([shardId, pool]) => ({
        shardId,
        result: await fn(pool, shardId),
      }))
    );

    return results
      .filter((r): r is PromiseFulfilledResult<{ shardId: string; result: T }> =>
        r.status === 'fulfilled'
      )
      .map((r) => r.value);
  }

  // Cross-shard query (scatter-gather)
  async scatter<T>(
    fn: (pool: Pool) => Promise<T[]>,
    merge: (results: T[][]) => T[]
  ): Promise<T[]> {
    const shardResults = await this.executeOnAllShards((pool) => fn(pool));
    return merge(shardResults.map((r) => r.result));
  }

  private resolveShardId(key: string): string {
    switch (this.strategy) {
      case 'hash':
        return this.hashShard(key);
      case 'consistent-hash':
        return this.consistentHashRing.getNode(key);
      default:
        return this.hashShard(key);
    }
  }

  private hashShard(key: string): string {
    const hash = createHash('md5').update(key).digest('hex');
    const num = parseInt(hash.substring(0, 8), 16);
    const index = num % this.shardConfigs.length;
    return this.shardConfigs[index].id;
  }
}

// Consistent Hash Ring
class ConsistentHashRing {
  private ring: Map<number, string> = new Map();
  private sortedKeys: number[] = [];

  constructor(nodes: string[], virtualNodes: number = 150) {
    for (const node of nodes) {
      for (let i = 0; i < virtualNodes; i++) {
        const key = this.hash(`${node}:${i}`);
        this.ring.set(key, node);
      }
    }
    this.sortedKeys = Array.from(this.ring.keys()).sort((a, b) => a - b);
  }

  getNode(key: string): string {
    if (this.sortedKeys.length === 0) throw new Error('Empty ring');
    
    const hash = this.hash(key);
    
    // Find first key >= hash (clockwise)
    const idx = this.sortedKeys.findIndex((k) => k >= hash);
    const ringKey = idx === -1 ? this.sortedKeys[0] : this.sortedKeys[idx];
    
    return this.ring.get(ringKey)!;
  }

  private hash(key: string): number {
    const h = createHash('md5').update(key).digest('hex');
    return parseInt(h.substring(0, 8), 16);
  }
}
```

---

## 5. Message Queue Scaling

### 5.1 Kafka Consumer Group Scaling

```typescript
// src/messaging/kafka/scalable-consumer.ts
import { Kafka, Consumer, EachMessagePayload } from 'kafkajs';
import { Redis } from 'ioredis';

interface ConsumerConfig {
  topic: string;
  groupId: string;
  concurrency: number;
  maxBatchSize: number;
  processingTimeoutMs: number;
  retryDelays: number[];
  deadLetterTopic: string;
}

interface ProcessingResult {
  success: boolean;
  retryable: boolean;
  error?: string;
}

export class ScalableKafkaConsumer {
  private consumer: Consumer;
  private processing = false;
  private inflightCount = 0;
  private processingQueue: Promise<void>[] = [];

  constructor(
    private kafka: Kafka,
    private config: ConsumerConfig,
    private redis: Redis,
    private messageHandler: (message: any) => Promise<ProcessingResult>
  ) {
    this.consumer = kafka.consumer({
      groupId: config.groupId,
      maxInFlightRequests: config.concurrency,
      sessionTimeout: 30000,
      heartbeatInterval: 3000,
      rebalanceTimeout: 60000,
    });
  }

  async start(): Promise<void> {
    await this.consumer.connect();
    await this.consumer.subscribe({
      topic: this.config.topic,
      fromBeginning: false,
    });

    this.processing = true;

    await this.consumer.run({
      autoCommit: false,
      partitionsConsumedConcurrently: this.config.concurrency,
      eachBatch: async ({ batch, resolveOffset, heartbeat, isRunning }) => {
        const messages = batch.messages;
        const batchPromises: Promise<void>[] = [];

        for (const message of messages) {
          if (!isRunning()) break;

          const promise = this.processWithRetry(message, batch.partition)
            .then(() => {
              resolveOffset(message.offset);
            });

          batchPromises.push(promise);

          // Limit concurrency
          if (batchPromises.length >= this.config.concurrency) {
            await Promise.all(batchPromises.splice(0, this.config.concurrency));
            await heartbeat();
          }
        }

        await Promise.all(batchPromises);

        // Commit after processing batch
        await this.consumer.commitOffsets([{
          topic: this.config.topic,
          partition: batch.partition,
          offset: (parseInt(batch.lastOffset()) + 1).toString(),
        }]);
      },
    });
  }

  private async processWithRetry(
    message: any,
    partition: number
  ): Promise<void> {
    const key = message.key?.toString();
    const value = message.value ? JSON.parse(message.value.toString()) : null;
    const retryCount = parseInt(message.headers?.['x-retry-count']?.toString() || '0');

    try {
      const result = await this.withTimeout(
        this.messageHandler(value),
        this.config.processingTimeoutMs
      );

      if (!result.success && result.retryable && retryCount < this.config.retryDelays.length) {
        await this.scheduleRetry(key, value, retryCount + 1);
      } else if (!result.success) {
        await this.sendToDeadLetter(key, value, result.error || 'Max retries exceeded');
      }
    } catch (error: any) {
      if (retryCount < this.config.retryDelays.length) {
        await this.scheduleRetry(key, value, retryCount + 1);
      } else {
        await this.sendToDeadLetter(key, value, error.message);
      }
    }
  }

  private async scheduleRetry(key: string | undefined, value: any, retryCount: number): Promise<void> {
    const delay = this.config.retryDelays[retryCount - 1] || 60000;
    
    // Use Redis sorted set as delay queue
    const executeAt = Date.now() + delay;
    await this.redis.zadd(
      `retry:${this.config.topic}`,
      executeAt,
      JSON.stringify({ key, value, retryCount, topic: this.config.topic })
    );
  }

  private async sendToDeadLetter(key: string | undefined, value: any, error: string): Promise<void> {
    const producer = this.kafka.producer();
    await producer.connect();
    await producer.send({
      topic: this.config.deadLetterTopic,
      messages: [{
        key: key || null,
        value: JSON.stringify({
          originalMessage: value,
          error,
          failedAt: new Date().toISOString(),
          originalTopic: this.config.topic,
        }),
        headers: {
          'x-original-topic': this.config.topic,
          'x-failure-reason': error,
        },
      }],
    });
    await producer.disconnect();
  }

  private withTimeout<T>(promise: Promise<T>, ms: number): Promise<T> {
    return Promise.race([
      promise,
      new Promise<T>((_, reject) =>
        setTimeout(() => reject(new Error(`Processing timeout after ${ms}ms`)), ms)
      ),
    ]);
  }

  async stop(): Promise<void> {
    this.processing = false;
    await this.consumer.stop();
    await this.consumer.disconnect();
  }
}
```

---

## 6. Auto-scaling Triggers และ Strategies

### 6.1 Custom Metrics Auto-scaling

```yaml
# kubernetes/custom-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: product-service-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: product-service
  minReplicas: 3
  maxReplicas: 50
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60   # Wait 60s before scaling up more
      policies:
        - type: Percent
          value: 100                   # Double pods
          periodSeconds: 60
        - type: Pods
          value: 5                     # Max 5 pods per scaling event
          periodSeconds: 60
      selectPolicy: Max                # Use policy that results in more pods
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5 minutes before scaling down
      policies:
        - type: Percent
          value: 10                    # Remove max 10% per period
          periodSeconds: 60
      selectPolicy: Min                # Conservative scale down
  metrics:
    # CPU-based scaling
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
    
    # Memory-based scaling
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 70

    # Custom metric: Request rate
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "100"

    # External metric: Queue depth
    - type: External
      external:
        metric:
          name: kafka_consumer_lag
          selector:
            matchLabels:
              topic: product-events
              consumer-group: product-service
        target:
          type: Value
          value: "1000"  # Scale up if lag > 1000 messages

    # Custom metric: Response time P95
    - type: Pods
      pods:
        metric:
          name: http_request_duration_p95_ms
        target:
          type: AverageValue
          averageValue: "500"   # Scale if P95 > 500ms
---
# KEDA ScaledObject for advanced scaling
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: product-service-keda
  namespace: production
spec:
  scaleTargetRef:
    name: product-service
  minReplicaCount: 3
  maxReplicaCount: 50
  cooldownPeriod: 300
  pollingInterval: 30
  triggers:
    # Scale based on Kafka consumer lag
    - type: kafka
      metadata:
        bootstrapServers: kafka-broker:9092
        consumerGroup: product-service
        topic: product-events
        lagThreshold: "500"
        activationLagThreshold: "100"
      authenticationRef:
        name: kafka-trigger-auth

    # Scale based on Redis queue length
    - type: redis
      metadata:
        address: redis:6379
        listName: product-processing-queue
        listLength: "100"
        activationListLength: "50"
      authenticationRef:
        name: redis-trigger-auth

    # Scale based on Prometheus metric
    - type: prometheus
      metadata:
        serverAddress: http://prometheus:9090
        metricName: http_requests_pending
        query: sum(rate(http_server_requests_total{service="product-service"}[2m]))
        threshold: "200"
        activationThreshold: "50"

    # Schedule-based scaling (ช่วงเวลา peak)
    - type: cron
      metadata:
        timezone: Asia/Bangkok
        start: "0 9 * * 1-5"    # 09:00 Monday-Friday
        end: "0 22 * * 1-5"     # 22:00 Monday-Friday
        desiredReplicas: "10"
```

### 6.2 Predictive Auto-scaling Service

```python
# src/scaling/predictive_scaler.py
import numpy as np
import pandas as pd
from sklearn.linear_model import LinearRegression
from sklearn.preprocessing import PolynomialFeatures
from datetime import datetime, timedelta
import boto3
import logging
from typing import List, Tuple

logger = logging.getLogger(__name__)

class PredictiveScaler:
    def __init__(self, service_name: str, k8s_client):
        self.service_name = service_name
        self.k8s_client = k8s_client
        self.cloudwatch = boto3.client('cloudwatch')
        self.model = None
        self.poly_features = PolynomialFeatures(degree=3)

    def train(self, historical_data: pd.DataFrame) -> None:
        """Train prediction model on historical metrics"""
        # Features: hour_of_day, day_of_week, is_weekend, month
        X = pd.DataFrame({
            'hour': historical_data['timestamp'].dt.hour,
            'day_of_week': historical_data['timestamp'].dt.dayofweek,
            'is_weekend': (historical_data['timestamp'].dt.dayofweek >= 5).astype(int),
            'month': historical_data['timestamp'].dt.month,
            'day_of_month': historical_data['timestamp'].dt.day,
        })

        y = historical_data['replicas']

        X_poly = self.poly_features.fit_transform(X)
        self.model = LinearRegression()
        self.model.fit(X_poly, y)
        
        score = self.model.score(X_poly, y)
        logger.info(f"Model trained with R² score: {score:.3f}")

    def predict_replicas(self, future_time: datetime) -> int:
        """Predict required replicas for a future time"""
        if self.model is None:
            raise RuntimeError("Model not trained")

        X = pd.DataFrame([{
            'hour': future_time.hour,
            'day_of_week': future_time.weekday(),
            'is_weekend': int(future_time.weekday() >= 5),
            'month': future_time.month,
            'day_of_month': future_time.day,
        }])

        X_poly = self.poly_features.transform(X)
        predicted = self.model.predict(X_poly)[0]
        
        # Add safety buffer (10%)
        return max(3, int(np.ceil(predicted * 1.1)))

    def get_scaling_schedule(
        self,
        hours_ahead: int = 24
    ) -> List[Tuple[datetime, int]]:
        """Generate scaling schedule for the next N hours"""
        schedule = []
        now = datetime.now()
        
        for hour in range(hours_ahead):
            future_time = now + timedelta(hours=hour)
            replicas = self.predict_replicas(future_time)
            schedule.append((future_time, replicas))
        
        return schedule

    def apply_predictive_scaling(self) -> None:
        """Pre-emptively scale up before predicted traffic spike"""
        now = datetime.now()
        
        # Look 15 minutes ahead
        future_time = now + timedelta(minutes=15)
        predicted_replicas = self.predict_replicas(future_time)
        
        current_replicas = self.get_current_replicas()
        
        if predicted_replicas > current_replicas * 1.2:  # Need 20% more replicas
            logger.info(
                f"Pre-emptive scale up: {current_replicas} -> {predicted_replicas} "
                f"for predicted load at {future_time}"
            )
            self.scale_to(predicted_replicas)
            
            # Record metric
            self.cloudwatch.put_metric_data(
                Namespace='CustomMetrics/Scaling',
                MetricData=[{
                    'MetricName': 'PredictiveScaleUp',
                    'Value': predicted_replicas - current_replicas,
                    'Unit': 'Count',
                    'Dimensions': [
                        {'Name': 'Service', 'Value': self.service_name},
                    ],
                }]
            )

    def get_current_replicas(self) -> int:
        deployment = self.k8s_client.AppsV1Api().read_namespaced_deployment(
            name=self.service_name,
            namespace='production'
        )
        return deployment.spec.replicas or 1

    def scale_to(self, replicas: int) -> None:
        from kubernetes import client as k8s
        apps_v1 = k8s.AppsV1Api()
        
        body = {'spec': {'replicas': replicas}}
        apps_v1.patch_namespaced_deployment_scale(
            name=self.service_name,
            namespace='production',
            body=body
        )
```

---

## 7. Connection Pool Scaling

### 7.1 PgBouncer Configuration

```ini
# pgbouncer/pgbouncer.ini
[databases]
production_db = host=postgres-primary port=5432 dbname=production_db pool_size=50
staging_db = host=postgres-replica1 port=5432 dbname=production_db pool_size=20

[pgbouncer]
# Connection mode
pool_mode = transaction         # Best for microservices

# Limits
max_client_conn = 5000          # Max connections from clients
default_pool_size = 50          # Pool size per (db, user) pair
min_pool_size = 10              # Minimum idle connections
reserve_pool_size = 10          # Emergency reserve pool
reserve_pool_timeout = 3.0      # Seconds before using reserve pool

# Timeouts
server_connect_timeout = 5      # Timeout for new backend connections
server_idle_timeout = 600       # Close idle backend connections after N seconds
client_idle_timeout = 60        # Close idle client connections after N seconds
server_lifetime = 3600          # Max backend connection age in seconds
client_login_timeout = 60       # Max time for client to login

# Logging
log_connections = 1
log_disconnections = 1
log_pooler_errors = 1
stats_period = 60

# Monitoring
stats_users = pgbouncer_monitor

# Authentication
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt

# Network
listen_addr = 0.0.0.0
listen_port = 6432

# TLS
server_tls_sslmode = require
server_tls_ca_file = /etc/ssl/certs/ca-certificates.crt
client_tls_sslmode = require
client_tls_key_file = /etc/ssl/private/server.key
client_tls_cert_file = /etc/ssl/certs/server.crt
```

---

## 8. Cache Scaling Patterns

### 8.1 Redis Cluster Configuration

```typescript
// src/cache/redis-cluster.ts
import { Cluster, ClusterOptions } from 'ioredis';

const clusterOptions: ClusterOptions = {
  clusterRetryStrategy: (times) => Math.min(100 + times * 2, 2000),
  enableOfflineQueue: false,
  enableReadyCheck: true,
  scaleReads: 'slave',  // Route reads to replicas
  maxRedirections: 16,
  retryDelayOnClusterDown: 300,
  retryDelayOnFailover: 1000,
  retryDelayOnTryAgain: 100,
  slotsRefreshTimeout: 10000,
  slotsRefreshInterval: 5000,
  
  redisOptions: {
    connectTimeout: 10000,
    commandTimeout: 5000,
    maxRetriesPerRequest: 3,
    enableAutoPipelining: true,   // Batch commands automatically
    lazyConnect: true,
  },
};

const CLUSTER_NODES = [
  { host: 'redis-cluster-0.redis-cluster.production.svc', port: 6379 },
  { host: 'redis-cluster-1.redis-cluster.production.svc', port: 6379 },
  { host: 'redis-cluster-2.redis-cluster.production.svc', port: 6379 },
  { host: 'redis-cluster-3.redis-cluster.production.svc', port: 6379 },
  { host: 'redis-cluster-4.redis-cluster.production.svc', port: 6379 },
  { host: 'redis-cluster-5.redis-cluster.production.svc', port: 6379 },
];

export class RedisClusterManager {
  private cluster: Cluster;
  private stats = { hits: 0, misses: 0, errors: 0 };

  constructor() {
    this.cluster = new Cluster(CLUSTER_NODES, clusterOptions);
    this.cluster.on('error', (err) => {
      this.stats.errors++;
      console.error('Redis cluster error:', err.message);
    });
    this.cluster.on('node error', (err, address) => {
      console.error(`Redis node ${address} error:`, err.message);
    });
  }

  async get<T>(key: string): Promise<T | null> {
    try {
      const value = await this.cluster.get(key);
      if (value === null) {
        this.stats.misses++;
        return null;
      }
      this.stats.hits++;
      return JSON.parse(value);
    } catch (error) {
      this.stats.errors++;
      return null;
    }
  }

  async set(key: string, value: any, ttlSeconds?: number): Promise<void> {
    const serialized = JSON.stringify(value);
    if (ttlSeconds) {
      await this.cluster.setex(key, ttlSeconds, serialized);
    } else {
      await this.cluster.set(key, serialized);
    }
  }

  // Multi-get for batch operations
  async mget<T>(keys: string[]): Promise<Array<T | null>> {
    if (keys.length === 0) return [];
    
    try {
      const values = await this.cluster.mget(...keys);
      return values.map((v) => (v ? JSON.parse(v) : null));
    } catch {
      return keys.map(() => null);
    }
  }

  // Pipeline for batch writes
  async mset(entries: Array<{ key: string; value: any; ttl?: number }>): Promise<void> {
    const pipeline = this.cluster.pipeline();
    for (const entry of entries) {
      if (entry.ttl) {
        pipeline.setex(entry.key, entry.ttl, JSON.stringify(entry.value));
      } else {
        pipeline.set(entry.key, JSON.stringify(entry.value));
      }
    }
    await pipeline.exec();
  }

  getStats(): typeof this.stats & { hitRate: number } {
    const total = this.stats.hits + this.stats.misses;
    return {
      ...this.stats,
      hitRate: total > 0 ? this.stats.hits / total : 0,
    };
  }
}
```

---

## สรุป

บทนี้ครอบคลุม Microservices Scalability Patterns อย่างครบถ้วน:

1. **Horizontal Scaling** - Stateless service design, idempotent operations
2. **Session Management** - Distributed session store ด้วย Redis, session limit enforcement
3. **Read/Write Splitting** - Weighted round-robin replica selection, lag monitoring
4. **Database Federation** - Consistent hash ring, scatter-gather pattern
5. **Message Queue Scaling** - Kafka consumer groups, retry queues, dead letter topics
6. **Auto-scaling** - HPA, KEDA with custom metrics, predictive scaling
7. **Connection Pooling** - PgBouncer transaction mode สำหรับ microservices
8. **Cache Scaling** - Redis Cluster with read replica routing

Key Takeaways:
- Stateless services คือ prerequisite สำหรับ horizontal scaling
- ใช้ session store แบบ distributed ไม่ใช่ sticky sessions
- Read/write splitting ช่วยให้ database scale ได้โดยไม่แตะ schema
- Predictive scaling ช่วยลด response time ช่วง traffic spike
- KEDA ให้ event-driven scaling ที่ละเอียดกว่า standard HPA
- Connection pooling ด้วย PgBouncer เป็น must-have สำหรับ microservices ขนาดใหญ่

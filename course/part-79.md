# Part 79: Scalability Patterns

## บทนำ

Scalability คือความสามารถของระบบในการรองรับ load ที่เพิ่มขึ้น โดยไม่ให้ performance ลดลง บทนี้จะครอบคลุม patterns สำคัญสำหรับการ scale microservices ตั้งแต่ horizontal scaling ไปจนถึง global load balancing

---

## 79.1 Horizontal Scaling: Stateless Service Design

### หลักการ Stateless Service

Stateless service หมายความว่า instance ใดๆ ก็สามารถ handle request ใดๆ ได้ โดยไม่ต้องพึ่งพา local state

**ข้อดี:**
- Scale in/out ได้อิสระ
- ทน fault ได้ดีกว่า
- Deploy แบบ rolling update ได้ง่าย
- Load balancing ทำงานได้อย่างมีประสิทธิภาพ

### Session Externalization ไปยัง Redis

```typescript
// session-manager.ts
import { createClient, RedisClientType } from 'redis';
import * as crypto from 'crypto';

interface Session {
  id: string;
  userId: string;
  email: string;
  roles: string[];
  data: Record<string, unknown>;
  createdAt: Date;
  lastAccessedAt: Date;
  expiresAt: Date;
}

interface SessionStore {
  get(sessionId: string): Promise<Session | null>;
  set(session: Session): Promise<void>;
  delete(sessionId: string): Promise<void>;
  refresh(sessionId: string, ttlSeconds: number): Promise<boolean>;
  getUserSessions(userId: string): Promise<string[]>;
  deleteUserSessions(userId: string): Promise<number>;
}

class RedisSessionStore implements SessionStore {
  private client: RedisClientType;
  private keyPrefix: string;
  private userIndexPrefix: string;
  private defaultTtlSeconds: number;

  constructor(
    redisUrl: string,
    options: {
      keyPrefix?: string;
      defaultTtlSeconds?: number;
    } = {}
  ) {
    this.client = createClient({ url: redisUrl }) as RedisClientType;
    this.keyPrefix = options.keyPrefix ?? 'session:';
    this.userIndexPrefix = 'user_sessions:';
    this.defaultTtlSeconds = options.defaultTtlSeconds ?? 3600; // 1 hour
  }

  async connect(): Promise<void> {
    await this.client.connect();
  }

  async disconnect(): Promise<void> {
    await this.client.disconnect();
  }

  private sessionKey(sessionId: string): string {
    return `${this.keyPrefix}${sessionId}`;
  }

  private userIndexKey(userId: string): string {
    return `${this.userIndexPrefix}${userId}`;
  }

  async get(sessionId: string): Promise<Session | null> {
    const data = await this.client.get(this.sessionKey(sessionId));
    if (!data) return null;

    const session = JSON.parse(data) as Session;
    session.createdAt = new Date(session.createdAt);
    session.lastAccessedAt = new Date(session.lastAccessedAt);
    session.expiresAt = new Date(session.expiresAt);

    // Update lastAccessedAt
    session.lastAccessedAt = new Date();
    await this.set(session);

    return session;
  }

  async set(session: Session, ttlSeconds?: number): Promise<void> {
    const ttl = ttlSeconds ?? this.defaultTtlSeconds;
    const key = this.sessionKey(session.id);
    const data = JSON.stringify(session);

    // Use pipeline สำหรับ atomic operation
    const pipeline = this.client.multi();
    pipeline.set(key, data, { EX: ttl });
    pipeline.sAdd(this.userIndexKey(session.userId), session.id);
    pipeline.expire(this.userIndexKey(session.userId), ttl + 60); // User index expires slightly later
    await pipeline.exec();
  }

  async delete(sessionId: string): Promise<void> {
    const session = await this.get(sessionId);
    if (!session) return;

    const pipeline = this.client.multi();
    pipeline.del(this.sessionKey(sessionId));
    pipeline.sRem(this.userIndexKey(session.userId), sessionId);
    await pipeline.exec();
  }

  async refresh(sessionId: string, ttlSeconds: number): Promise<boolean> {
    const exists = await this.client.expire(
      this.sessionKey(sessionId),
      ttlSeconds
    );
    return exists === 1;
  }

  async getUserSessions(userId: string): Promise<string[]> {
    return this.client.sMembers(this.userIndexKey(userId));
  }

  async deleteUserSessions(userId: string): Promise<number> {
    const sessionIds = await this.getUserSessions(userId);
    
    if (sessionIds.length === 0) return 0;

    const pipeline = this.client.multi();
    sessionIds.forEach(id => pipeline.del(this.sessionKey(id)));
    pipeline.del(this.userIndexKey(userId));
    
    const results = await pipeline.exec();
    return sessionIds.length;
  }
}

class StatelessSessionManager {
  private store: RedisSessionStore;

  constructor(redisUrl: string) {
    this.store = new RedisSessionStore(redisUrl, {
      keyPrefix: 'app:session:',
      defaultTtlSeconds: 3600,
    });
  }

  async initialize(): Promise<void> {
    await this.store.connect();
  }

  async createSession(
    userId: string,
    email: string,
    roles: string[],
    additionalData?: Record<string, unknown>
  ): Promise<Session> {
    const now = new Date();
    const session: Session = {
      id: crypto.randomBytes(32).toString('hex'),
      userId,
      email,
      roles,
      data: additionalData ?? {},
      createdAt: now,
      lastAccessedAt: now,
      expiresAt: new Date(now.getTime() + 3600 * 1000),
    };

    await this.store.set(session);
    return session;
  }

  async getSession(sessionId: string): Promise<Session | null> {
    return this.store.get(sessionId);
  }

  async invalidateSession(sessionId: string): Promise<void> {
    await this.store.delete(sessionId);
  }

  async invalidateUserSessions(userId: string): Promise<number> {
    return this.store.deleteUserSessions(userId);
  }
}

// Express middleware
import { Request, Response, NextFunction } from 'express';

function sessionMiddleware(manager: StatelessSessionManager) {
  return async (req: Request, res: Response, next: NextFunction) => {
    const sessionId = req.cookies?.['session_id'] ?? 
      req.headers['x-session-id'] as string;

    if (!sessionId) {
      return next();
    }

    const session = await manager.getSession(sessionId);
    if (!session) {
      res.clearCookie('session_id');
      return next();
    }

    // Inject session into request
    (req as any).session = session;
    (req as any).user = {
      id: session.userId,
      email: session.email,
      roles: session.roles,
    };

    next();
  };
}

export { StatelessSessionManager, RedisSessionStore, sessionMiddleware };
```

### Kubernetes HorizontalPodAutoscaler สำหรับ Stateless Service

```yaml
# stateless-service-hpa.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-service
  namespace: production
spec:
  replicas: 3                   # Initial replicas
  selector:
    matchLabels:
      app: api-service
  template:
    metadata:
      labels:
        app: api-service
    spec:
      # ไม่มี local state - ทุก pod เหมือนกัน
      containers:
        - name: api-service
          image: myregistry/api-service:latest
          ports:
            - containerPort: 8080
          env:
            - name: REDIS_URL
              valueFrom:
                secretKeyRef:
                  name: redis-credentials
                  key: url
            - name: DB_URL
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: url
            # ไม่มี NODE_ID หรือ POD_IP ที่ใช้เก็บ state
          resources:
            requests:
              cpu: "200m"
              memory: "256Mi"
            limits:
              cpu: "1000m"
              memory: "512Mi"
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
            successThreshold: 1
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 5"]  # Graceful shutdown
      terminationGracePeriodSeconds: 30
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                topologyKey: kubernetes.io/hostname
                labelSelector:
                  matchLabels:
                    app: api-service
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-service-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-service
  minReplicas: 3
  maxReplicas: 50
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
        - type: Pods
          value: 5
          periodSeconds: 60
        - type: Percent
          value: 100
          periodSeconds: 60
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 2
          periodSeconds: 60
      selectPolicy: Min
```

---

## 79.2 Database Read Replicas: TypeScript ReadReplicaPool

### Architecture แบบ Primary-Replica

```typescript
// read-replica-pool.ts
import { Pool, PoolConfig, QueryResult } from 'pg';

interface DatabaseConfig {
  connectionString: string;
  maxConnections: number;
  idleTimeoutMillis: number;
  connectionTimeoutMillis: number;
}

interface ReadReplicaPoolConfig {
  primary: DatabaseConfig;
  replicas: DatabaseConfig[];
  replicaLoadBalancing: 'round-robin' | 'random' | 'least-connections';
  replicaHealthCheckIntervalMs: number;
  retryOnReplicaFailure: boolean;
}

interface PoolHealth {
  connectionString: string;
  healthy: boolean;
  activeConnections: number;
  idleConnections: number;
  waitingRequests: number;
  lastChecked: Date;
  errorCount: number;
}

class ReadReplicaPool {
  private primaryPool: Pool;
  private replicaPools: Pool[];
  private config: ReadReplicaPoolConfig;
  private currentReplicaIndex = 0;
  private replicaHealthMap: Map<Pool, PoolHealth>;
  private healthCheckInterval: NodeJS.Timeout | null = null;

  constructor(config: ReadReplicaPoolConfig) {
    this.config = config;
    this.replicaHealthMap = new Map();

    // สร้าง Primary Pool
    this.primaryPool = new Pool({
      connectionString: config.primary.connectionString,
      max: config.primary.maxConnections,
      idleTimeoutMillis: config.primary.idleTimeoutMillis,
      connectionTimeoutMillis: config.primary.connectionTimeoutMillis,
    });

    // สร้าง Replica Pools
    this.replicaPools = config.replicas.map(replicaConfig => {
      const pool = new Pool({
        connectionString: replicaConfig.connectionString,
        max: replicaConfig.maxConnections,
        idleTimeoutMillis: replicaConfig.idleTimeoutMillis,
        connectionTimeoutMillis: replicaConfig.connectionTimeoutMillis,
      });

      // Initialize health map
      this.replicaHealthMap.set(pool, {
        connectionString: replicaConfig.connectionString,
        healthy: true,
        activeConnections: 0,
        idleConnections: 0,
        waitingRequests: 0,
        lastChecked: new Date(),
        errorCount: 0,
      });

      return pool;
    });

    // Setup error handlers
    this.setupErrorHandlers();
    
    // Start health checks
    this.startHealthChecks();
  }

  private setupErrorHandlers(): void {
    this.primaryPool.on('error', (err) => {
      console.error('[ReadReplicaPool] Primary pool error:', err.message);
    });

    this.replicaPools.forEach((pool, index) => {
      pool.on('error', (err) => {
        console.error(`[ReadReplicaPool] Replica ${index} error:`, err.message);
        const health = this.replicaHealthMap.get(pool);
        if (health) {
          health.errorCount++;
          if (health.errorCount > 5) {
            health.healthy = false;
          }
        }
      });
    });
  }

  private startHealthChecks(): void {
    if (this.config.replicaHealthCheckIntervalMs <= 0) return;

    this.healthCheckInterval = setInterval(
      () => this.checkReplicaHealth(),
      this.config.replicaHealthCheckIntervalMs
    );
  }

  private async checkReplicaHealth(): Promise<void> {
    const checks = this.replicaPools.map(async (pool) => {
      const health = this.replicaHealthMap.get(pool)!;
      
      try {
        const client = await pool.connect();
        await client.query('SELECT 1');
        client.release();

        health.healthy = true;
        health.errorCount = 0;
        health.activeConnections = pool.totalCount - pool.idleCount;
        health.idleConnections = pool.idleCount;
        health.waitingRequests = pool.waitingCount;
        health.lastChecked = new Date();
      } catch (err: any) {
        health.healthy = false;
        health.errorCount++;
        health.lastChecked = new Date();
      }
    });

    await Promise.allSettled(checks);
  }

  private getHealthyReplicas(): Pool[] {
    return this.replicaPools.filter(pool => {
      const health = this.replicaHealthMap.get(pool);
      return health?.healthy ?? false;
    });
  }

  private selectReplica(): Pool | null {
    const healthyReplicas = this.getHealthyReplicas();
    
    if (healthyReplicas.length === 0) return null;

    switch (this.config.replicaLoadBalancing) {
      case 'round-robin':
        this.currentReplicaIndex = 
          (this.currentReplicaIndex + 1) % healthyReplicas.length;
        return healthyReplicas[this.currentReplicaIndex];

      case 'random':
        return healthyReplicas[
          Math.floor(Math.random() * healthyReplicas.length)
        ];

      case 'least-connections':
        return healthyReplicas.reduce((min, pool) => {
          const minHealth = this.replicaHealthMap.get(min)!;
          const poolHealth = this.replicaHealthMap.get(pool)!;
          return poolHealth.activeConnections < minHealth.activeConnections
            ? pool
            : min;
        });

      default:
        return healthyReplicas[0];
    }
  }

  // WRITE operations → Primary
  async write<T = any>(
    query: string,
    params?: any[]
  ): Promise<QueryResult<T>> {
    return this.primaryPool.query<T>(query, params);
  }

  // READ operations → Replica (fallback to primary)
  async read<T = any>(
    query: string,
    params?: any[]
  ): Promise<QueryResult<T>> {
    const replica = this.selectReplica();

    if (!replica) {
      console.warn('[ReadReplicaPool] No healthy replicas, falling back to primary');
      return this.primaryPool.query<T>(query, params);
    }

    try {
      return await replica.query<T>(query, params);
    } catch (err) {
      if (this.config.retryOnReplicaFailure) {
        console.warn('[ReadReplicaPool] Replica query failed, retrying on primary');
        const health = this.replicaHealthMap.get(replica);
        if (health) {
          health.healthy = false;
          health.errorCount++;
        }
        return this.primaryPool.query<T>(query, params);
      }
      throw err;
    }
  }

  // Transaction → Primary
  async transaction<T>(
    callback: (client: any) => Promise<T>
  ): Promise<T> {
    const client = await this.primaryPool.connect();
    
    try {
      await client.query('BEGIN');
      const result = await callback(client);
      await client.query('COMMIT');
      return result;
    } catch (err) {
      await client.query('ROLLBACK');
      throw err;
    } finally {
      client.release();
    }
  }

  getPoolStats(): Record<string, any> {
    return {
      primary: {
        total: this.primaryPool.totalCount,
        idle: this.primaryPool.idleCount,
        waiting: this.primaryPool.waitingCount,
      },
      replicas: this.replicaPools.map((pool, i) => ({
        index: i,
        health: this.replicaHealthMap.get(pool),
        total: pool.totalCount,
        idle: pool.idleCount,
        waiting: pool.waitingCount,
      })),
    };
  }

  async destroy(): Promise<void> {
    if (this.healthCheckInterval) {
      clearInterval(this.healthCheckInterval);
    }
    
    await Promise.all([
      this.primaryPool.end(),
      ...this.replicaPools.map(p => p.end()),
    ]);
  }
}

// ตัวอย่างการใช้งาน
async function setupDatabase(): Promise<ReadReplicaPool> {
  const pool = new ReadReplicaPool({
    primary: {
      connectionString: process.env.DB_PRIMARY_URL!,
      maxConnections: 20,
      idleTimeoutMillis: 30000,
      connectionTimeoutMillis: 5000,
    },
    replicas: [
      {
        connectionString: process.env.DB_REPLICA_1_URL!,
        maxConnections: 30,
        idleTimeoutMillis: 30000,
        connectionTimeoutMillis: 5000,
      },
      {
        connectionString: process.env.DB_REPLICA_2_URL!,
        maxConnections: 30,
        idleTimeoutMillis: 30000,
        connectionTimeoutMillis: 5000,
      },
    ],
    replicaLoadBalancing: 'round-robin',
    replicaHealthCheckIntervalMs: 30000,
    retryOnReplicaFailure: true,
  });

  return pool;
}

// Usage in service layer
async function getUser(db: ReadReplicaPool, userId: string) {
  // READ → uses replica
  const result = await db.read(
    'SELECT id, email, name, created_at FROM users WHERE id = $1',
    [userId]
  );
  return result.rows[0] ?? null;
}

async function createUser(
  db: ReadReplicaPool,
  email: string,
  name: string
) {
  // WRITE → uses primary
  const result = await db.write(
    'INSERT INTO users (email, name, created_at) VALUES ($1, $2, NOW()) RETURNING *',
    [email, name]
  );
  return result.rows[0];
}

export { ReadReplicaPool, setupDatabase };
```

---

## 79.3 CQRS for Scalability

### Command DB (Write) vs Query DB (Read Replica) Routing

```typescript
// cqrs-router.ts
import { Pool } from 'pg';
import { EventEmitter } from 'events';

// =================== Commands ===================
interface Command {
  type: string;
  payload: Record<string, unknown>;
  metadata: {
    userId: string;
    correlationId: string;
    timestamp: Date;
  };
}

interface CommandResult<T = any> {
  success: boolean;
  data?: T;
  error?: string;
  aggregateId?: string;
}

// =================== Queries ===================
interface Query {
  type: string;
  params: Record<string, unknown>;
}

interface QueryResult<T = any> {
  data: T;
  total?: number;
  page?: number;
  pageSize?: number;
}

// =================== Event ===================
interface DomainEvent {
  id: string;
  type: string;
  aggregateId: string;
  aggregateType: string;
  payload: Record<string, unknown>;
  metadata: {
    userId?: string;
    correlationId?: string;
    timestamp: Date;
    version: number;
  };
}

type CommandHandler<C extends Command = Command, R = any> = (
  command: C,
  writeDb: Pool
) => Promise<CommandResult<R>>;

type QueryHandler<Q extends Query = Query, R = any> = (
  query: Q,
  readDb: Pool
) => Promise<QueryResult<R>>;

class CQRSRouter extends EventEmitter {
  private writePool: Pool;    // Primary DB - for writes
  private readPool: Pool;     // Read Replica - for reads
  private commandHandlers: Map<string, CommandHandler>;
  private queryHandlers: Map<string, QueryHandler>;
  private eventOutbox: DomainEvent[] = [];

  constructor(writeDsn: string, readDsn: string) {
    super();
    
    this.writePool = new Pool({
      connectionString: writeDsn,
      max: 10,  // Write connections - ต้องการน้อยกว่า
    });

    this.readPool = new Pool({
      connectionString: readDsn,
      max: 50,  // Read connections - ต้องการมากกว่า
    });

    this.commandHandlers = new Map();
    this.queryHandlers = new Map();

    this.setupOutboxProcessor();
  }

  registerCommandHandler<C extends Command, R = any>(
    commandType: string,
    handler: CommandHandler<C, R>
  ): void {
    this.commandHandlers.set(commandType, handler as CommandHandler);
  }

  registerQueryHandler<Q extends Query, R = any>(
    queryType: string,
    handler: QueryHandler<Q, R>
  ): void {
    this.queryHandlers.set(queryType, handler as QueryHandler);
  }

  async executeCommand<R = any>(command: Command): Promise<CommandResult<R>> {
    const handler = this.commandHandlers.get(command.type);
    
    if (!handler) {
      throw new Error(`No handler registered for command: ${command.type}`);
    }

    const client = await this.writePool.connect();
    
    try {
      await client.query('BEGIN');
      
      const result = await handler(command, this.writePool) as CommandResult<R>;
      
      // Commit transaction
      await client.query('COMMIT');
      
      // Emit event for read-side update (eventual consistency)
      this.emit('command.executed', {
        command,
        result,
        timestamp: new Date(),
      });

      return result;
    } catch (err: any) {
      await client.query('ROLLBACK');
      return {
        success: false,
        error: err.message,
      };
    } finally {
      client.release();
    }
  }

  async executeQuery<R = any>(query: Query): Promise<QueryResult<R>> {
    const handler = this.queryHandlers.get(query.type);
    
    if (!handler) {
      throw new Error(`No handler registered for query: ${query.type}`);
    }

    // Queries go to read replica
    return handler(query, this.readPool) as Promise<QueryResult<R>>;
  }

  private setupOutboxProcessor(): void {
    // Process outbox events every 100ms
    setInterval(async () => {
      if (this.eventOutbox.length === 0) return;
      
      const events = this.eventOutbox.splice(0, 100);
      this.emit('events.batch', events);
    }, 100);
  }

  async destroy(): Promise<void> {
    await Promise.all([
      this.writePool.end(),
      this.readPool.end(),
    ]);
  }
}

// =================== ตัวอย่าง Order Service ===================

interface CreateOrderCommand extends Command {
  type: 'CreateOrder';
  payload: {
    customerId: string;
    items: Array<{
      productId: string;
      quantity: number;
      price: number;
    }>;
    shippingAddress: string;
  };
}

interface GetOrdersQuery extends Query {
  type: 'GetOrders';
  params: {
    customerId: string;
    page: number;
    pageSize: number;
    status?: string;
  };
}

// Command Handler - goes to Write DB
const createOrderHandler: CommandHandler<CreateOrderCommand> = async (
  command,
  writeDb
) => {
  const { customerId, items, shippingAddress } = command.payload;
  
  const total = items.reduce((sum, item) => sum + item.price * item.quantity, 0);
  
  const result = await writeDb.query(
    `INSERT INTO orders (customer_id, items, total, shipping_address, status, created_at)
     VALUES ($1, $2, $3, $4, 'pending', NOW())
     RETURNING id`,
    [customerId, JSON.stringify(items), total, shippingAddress]
  );

  const orderId = result.rows[0].id;

  // Insert order items
  for (const item of items) {
    await writeDb.query(
      `INSERT INTO order_items (order_id, product_id, quantity, price)
       VALUES ($1, $2, $3, $4)`,
      [orderId, item.productId, item.quantity, item.price]
    );
  }

  return {
    success: true,
    aggregateId: orderId,
    data: { orderId, total },
  };
};

// Query Handler - goes to Read Replica
const getOrdersQueryHandler: QueryHandler<GetOrdersQuery> = async (
  query,
  readDb
) => {
  const { customerId, page, pageSize, status } = query.params;
  const offset = (page - 1) * pageSize;

  const whereClause = status
    ? 'WHERE o.customer_id = $1 AND o.status = $4'
    : 'WHERE o.customer_id = $1';

  const params = status
    ? [customerId, pageSize, offset, status]
    : [customerId, pageSize, offset];

  const [ordersResult, countResult] = await Promise.all([
    readDb.query(
      `SELECT o.id, o.total, o.status, o.created_at,
              json_agg(oi.*) as items
       FROM orders o
       LEFT JOIN order_items oi ON oi.order_id = o.id
       ${whereClause}
       GROUP BY o.id
       ORDER BY o.created_at DESC
       LIMIT $2 OFFSET $3`,
      params
    ),
    readDb.query(
      `SELECT COUNT(*) as total FROM orders o ${whereClause}`,
      status ? [customerId, status] : [customerId]
    ),
  ]);

  return {
    data: ordersResult.rows,
    total: parseInt(countResult.rows[0].total),
    page,
    pageSize,
  };
};

// Setup CQRS Router
function setupOrderCQRS(): CQRSRouter {
  const router = new CQRSRouter(
    process.env.WRITE_DB_URL!,
    process.env.READ_DB_URL!
  );

  router.registerCommandHandler('CreateOrder', createOrderHandler);
  router.registerQueryHandler('GetOrders', getOrdersQueryHandler);

  return router;
}

export { CQRSRouter, setupOrderCQRS };
```

---

## 79.4 Stateless JWT vs Stateful Sessions

### TypeScript JWT Middleware

```typescript
// jwt-middleware.ts
import * as jwt from 'jsonwebtoken';
import { Request, Response, NextFunction } from 'express';
import { createClient } from 'redis';

interface JWTPayload {
  sub: string;          // User ID
  email: string;
  roles: string[];
  iss: string;          // Issuer
  aud: string[];        // Audience
  iat: number;          // Issued at
  exp: number;          // Expiry
  jti: string;          // JWT ID (for revocation)
}

interface TokenPair {
  accessToken: string;
  refreshToken: string;
  expiresIn: number;
}

interface JWTConfig {
  accessTokenSecret: string;
  refreshTokenSecret: string;
  accessTokenTtlSeconds: number;
  refreshTokenTtlSeconds: number;
  issuer: string;
  audience: string[];
  redisUrl: string;     // สำหรับ token revocation list
}

class JWTService {
  private config: JWTConfig;
  private redisClient: ReturnType<typeof createClient>;
  private revokedTokenPrefix = 'revoked_token:';

  constructor(config: JWTConfig) {
    this.config = config;
    this.redisClient = createClient({ url: config.redisUrl });
    this.redisClient.connect().catch(console.error);
  }

  async generateTokenPair(
    userId: string,
    email: string,
    roles: string[]
  ): Promise<TokenPair> {
    const jti = `${userId}-${Date.now()}-${Math.random().toString(36).substr(2)}`;
    
    const accessTokenPayload: Omit<JWTPayload, 'iat' | 'exp'> = {
      sub: userId,
      email,
      roles,
      iss: this.config.issuer,
      aud: this.config.audience,
      jti,
    };

    const accessToken = jwt.sign(
      accessTokenPayload,
      this.config.accessTokenSecret,
      { expiresIn: this.config.accessTokenTtlSeconds }
    );

    const refreshToken = jwt.sign(
      {
        sub: userId,
        type: 'refresh',
        jti: `refresh-${jti}`,
        iss: this.config.issuer,
      },
      this.config.refreshTokenSecret,
      { expiresIn: this.config.refreshTokenTtlSeconds }
    );

    return {
      accessToken,
      refreshToken,
      expiresIn: this.config.accessTokenTtlSeconds,
    };
  }

  verifyAccessToken(token: string): JWTPayload {
    const decoded = jwt.verify(token, this.config.accessTokenSecret, {
      issuer: this.config.issuer,
      audience: this.config.audience,
    }) as JWTPayload;

    return decoded;
  }

  verifyRefreshToken(token: string): any {
    return jwt.verify(token, this.config.refreshTokenSecret, {
      issuer: this.config.issuer,
    });
  }

  async revokeToken(jti: string, expiresIn: number): Promise<void> {
    await this.redisClient.set(
      `${this.revokedTokenPrefix}${jti}`,
      '1',
      { EX: expiresIn }
    );
  }

  async isTokenRevoked(jti: string): Promise<boolean> {
    const result = await this.redisClient.exists(
      `${this.revokedTokenPrefix}${jti}`
    );
    return result === 1;
  }

  async revokeAllUserTokens(userId: string): Promise<void> {
    // Store a timestamp - any token issued before this time is revoked
    await this.redisClient.set(
      `user_revoked_before:${userId}`,
      Date.now().toString(),
      { EX: this.config.refreshTokenTtlSeconds }
    );
  }

  async refreshTokenPair(refreshToken: string): Promise<TokenPair | null> {
    try {
      const decoded = this.verifyRefreshToken(refreshToken) as any;
      
      if (await this.isTokenRevoked(decoded.jti)) {
        return null;
      }

      // Revoke old refresh token (rotation)
      const remainingTtl = decoded.exp - Math.floor(Date.now() / 1000);
      await this.revokeToken(decoded.jti, Math.max(remainingTtl, 60));

      // Issue new token pair
      // ดึง user info จาก database
      // (simplified - ในกรณีจริงต้อง query จาก DB)
      const userId = decoded.sub;
      
      return this.generateTokenPair(userId, '', []);
    } catch {
      return null;
    }
  }
}

// Express Middleware Factory
function createJWTMiddleware(jwtService: JWTService) {
  return {
    authenticate: async (req: Request, res: Response, next: NextFunction) => {
      const authHeader = req.headers.authorization;
      
      if (!authHeader?.startsWith('Bearer ')) {
        return res.status(401).json({ error: 'Missing or invalid authorization header' });
      }

      const token = authHeader.substring(7);

      try {
        const payload = jwtService.verifyAccessToken(token);

        // Check token revocation
        if (await jwtService.isTokenRevoked(payload.jti)) {
          return res.status(401).json({ error: 'Token has been revoked' });
        }

        (req as any).user = payload;
        next();
      } catch (err: any) {
        if (err.name === 'TokenExpiredError') {
          return res.status(401).json({ error: 'Token expired', code: 'TOKEN_EXPIRED' });
        }
        return res.status(401).json({ error: 'Invalid token' });
      }
    },

    requireRole: (...roles: string[]) => {
      return (req: Request, res: Response, next: NextFunction) => {
        const user = (req as any).user as JWTPayload;
        
        if (!user) {
          return res.status(401).json({ error: 'Not authenticated' });
        }

        const hasRole = roles.some(role => user.roles.includes(role));
        
        if (!hasRole) {
          return res.status(403).json({ 
            error: 'Insufficient permissions',
            required: roles,
            actual: user.roles,
          });
        }

        next();
      };
    },
  };
}

export { JWTService, createJWTMiddleware, type TokenPair, type JWTPayload };
```

---

## 79.5 Message Queue Scaling: Kafka Partition Calculation

### การคำนวณจำนวน Partition ที่เหมาะสม

```typescript
// kafka-partition-calculator.ts

interface KafkaScalingConfig {
  targetRps: number;             // target requests per second
  averageMessageSizeBytes: number;
  processingTimeMs: number;      // เวลาที่ consumer ใช้ต่อ message
  consumerGroupCount: number;    // จำนวน consumer groups
  replicationFactor: number;     // Kafka replication factor
  brokerCount: number;
  retentionHours: number;
  diskPerBrokerGB: number;
}

interface KafkaScalingRecommendation {
  partitionCount: number;
  consumerCount: number;
  brokerCount: number;
  estimatedDiskUsageGB: number;
  estimatedThroughputMBs: number;
  reasoning: string[];
}

class KafkaPartitionCalculator {
  calculate(config: KafkaScalingConfig): KafkaScalingRecommendation {
    const reasoning: string[] = [];

    // 1. คำนวณ throughput ที่ต้องการ
    const throughputBytesPerSec = config.targetRps * config.averageMessageSizeBytes;
    const throughputMBs = throughputBytesPerSec / (1024 * 1024);
    reasoning.push(
      `Target throughput: ${config.targetRps} msg/s × ${config.averageMessageSizeBytes} bytes = ` +
      `${throughputMBs.toFixed(2)} MB/s`
    );

    // 2. คำนวณ consumer throughput ต่อ partition
    // consumer สามารถ process ได้ 1000ms / processingTimeMs messages ต่อวินาที
    const consumerThroughputPerSec = Math.floor(1000 / config.processingTimeMs);
    const throughputPerConsumer = consumerThroughputPerSec * config.averageMessageSizeBytes;
    reasoning.push(
      `Consumer throughput: ${consumerThroughputPerSec} msg/s per consumer ` +
      `(${config.processingTimeMs}ms per message)`
    );

    // 3. คำนวณจำนวน consumer ที่ต้องการ
    const requiredConsumers = Math.ceil(config.targetRps / consumerThroughputPerSec);
    reasoning.push(`Required consumers: ${requiredConsumers}`);

    // 4. Partition count = max(consumers, brokers × 2)
    // Kafka best practice: partitions >= consumer count, และ divisible by broker count
    let partitionCount = Math.max(
      requiredConsumers * config.consumerGroupCount,
      config.brokerCount * 2
    );

    // Round up to nearest multiple of broker count
    partitionCount = Math.ceil(partitionCount / config.brokerCount) * config.brokerCount;
    
    // Minimum 6 partitions, maximum 200 per topic (Kafka recommendation)
    partitionCount = Math.max(6, Math.min(200, partitionCount));
    
    reasoning.push(
      `Partition count: max(${requiredConsumers * config.consumerGroupCount}, ${config.brokerCount * 2}) ` +
      `rounded to multiple of ${config.brokerCount} = ${partitionCount}`
    );

    // 5. คำนวณ disk usage
    const bytesPerSecond = throughputBytesPerSec;
    const retentionSeconds = config.retentionHours * 3600;
    const totalDataGB = (bytesPerSecond * retentionSeconds * config.replicationFactor) / (1024 ** 3);
    const diskPerBrokerGB = totalDataGB / config.brokerCount;
    
    reasoning.push(
      `Estimated disk: ${totalDataGB.toFixed(1)} GB total, ` +
      `${diskPerBrokerGB.toFixed(1)} GB per broker ` +
      `(${config.diskPerBrokerGB} GB available)`
    );

    if (diskPerBrokerGB > config.diskPerBrokerGB * 0.8) {
      reasoning.push(
        `WARNING: Disk usage (${diskPerBrokerGB.toFixed(1)} GB) ` +
        `exceeds 80% of available (${config.diskPerBrokerGB} GB). Consider reducing retention or adding brokers.`
      );
    }

    return {
      partitionCount,
      consumerCount: requiredConsumers,
      brokerCount: Math.max(config.brokerCount, config.replicationFactor),
      estimatedDiskUsageGB: diskPerBrokerGB,
      estimatedThroughputMBs: throughputMBs,
      reasoning,
    };
  }

  printReport(config: KafkaScalingConfig): void {
    const rec = this.calculate(config);
    
    console.log('\n=== Kafka Partition Scaling Calculator ===\n');
    console.log('Input:');
    console.log(`  Target RPS: ${config.targetRps.toLocaleString()}`);
    console.log(`  Message Size: ${config.averageMessageSizeBytes} bytes`);
    console.log(`  Processing Time: ${config.processingTimeMs}ms`);
    console.log(`  Consumer Groups: ${config.consumerGroupCount}`);
    console.log(`  Brokers: ${config.brokerCount}`);
    console.log(`  Replication Factor: ${config.replicationFactor}`);
    console.log('');
    console.log('Recommendation:');
    console.log(`  Partition Count: ${rec.partitionCount}`);
    console.log(`  Consumer Count per Group: ${rec.consumerCount}`);
    console.log(`  Min Brokers: ${rec.brokerCount}`);
    console.log(`  Throughput: ${rec.estimatedThroughputMBs.toFixed(2)} MB/s`);
    console.log(`  Disk per Broker: ${rec.estimatedDiskUsageGB.toFixed(1)} GB`);
    console.log('');
    console.log('Reasoning:');
    rec.reasoning.forEach(r => console.log(`  - ${r}`));
  }
}

// Consumer Group Rebalancing Monitor
interface ConsumerGroupMember {
  memberId: string;
  clientId: string;
  partitions: number[];
}

interface ConsumerGroupStatus {
  groupId: string;
  state: 'Stable' | 'PreparingRebalance' | 'CompletingRebalance' | 'Empty' | 'Dead';
  members: ConsumerGroupMember[];
  lag: number;
  isRebalancing: boolean;
}

// Example usage
const calculator = new KafkaPartitionCalculator();
calculator.printReport({
  targetRps: 100000,
  averageMessageSizeBytes: 1024,
  processingTimeMs: 10,
  consumerGroupCount: 3,
  replicationFactor: 3,
  brokerCount: 9,
  retentionHours: 24,
  diskPerBrokerGB: 500,
});
```

### Kafka Consumer Group YAML

```yaml
# kafka-consumer-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-event-consumer
  namespace: production
spec:
  replicas: 12   # Equal to partition count (or factor of it)
  selector:
    matchLabels:
      app: order-event-consumer
  template:
    metadata:
      labels:
        app: order-event-consumer
    spec:
      containers:
        - name: consumer
          image: myregistry/order-consumer:latest
          env:
            - name: KAFKA_BROKERS
              value: "kafka-0:9092,kafka-1:9092,kafka-2:9092"
            - name: KAFKA_GROUP_ID
              value: "order-processor-v1"
            - name: KAFKA_TOPIC
              value: "order-events"
            - name: KAFKA_AUTO_OFFSET_RESET
              value: "latest"
            - name: KAFKA_SESSION_TIMEOUT_MS
              value: "30000"
            - name: KAFKA_HEARTBEAT_INTERVAL_MS
              value: "3000"
          resources:
            requests:
              cpu: "200m"
              memory: "256Mi"
            limits:
              cpu: "1000m"
              memory: "512Mi"
```

---

## 79.6 Auto-scaling Triggers: Custom Prometheus Metrics → HPA, KEDA ScaledObject

### Custom Metrics HPA

```yaml
# custom-metrics-hpa.yaml
# ต้องมี Prometheus Adapter ก่อน
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-custom-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 3
  maxReplicas: 30
  metrics:
    # CPU (standard metric)
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    # Custom: orders per second per pod
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "100"   # 100 req/s per pod
    # External: Kafka consumer lag
    - type: External
      external:
        metric:
          name: kafka_consumer_lag
          selector:
            matchLabels:
              topic: order-events
              group: order-processor
        target:
          type: AverageValue
          averageValue: "1000"  # Scale when lag > 1000 per replica
```

### KEDA ScaledObject สำหรับ Kafka

```yaml
# keda-kafka-scaler.yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-consumer-scaler
  namespace: production
spec:
  scaleTargetRef:
    name: order-event-consumer
  pollingInterval: 15        # ตรวจทุก 15 วินาที
  cooldownPeriod: 60         # รอ 60 วินาทีก่อน scale down
  minReplicaCount: 2
  maxReplicaCount: 24        # = partition count * 2
  advanced:
    horizontalPodAutoscalerConfig:
      behavior:
        scaleUp:
          stabilizationWindowSeconds: 30
          policies:
            - type: Percent
              value: 100
              periodSeconds: 30
        scaleDown:
          stabilizationWindowSeconds: 120
          policies:
            - type: Pods
              value: 2
              periodSeconds: 60
  triggers:
    - type: kafka
      metadata:
        bootstrapServers: "kafka-0.kafka:9092,kafka-1.kafka:9092,kafka-2.kafka:9092"
        consumerGroup: "order-processor-v1"
        topic: "order-events"
        lagThreshold: "500"       # Scale up when lag > 500 per replica
        activationLagThreshold: "10"  # Start from 0 when lag > 10
      authenticationRef:
        name: kafka-trigger-auth
---
# TriggerAuthentication สำหรับ KEDA
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: kafka-trigger-auth
  namespace: production
spec:
  secretTargetRef:
    - parameter: sasl.username
      name: kafka-credentials
      key: username
    - parameter: sasl.password
      name: kafka-credentials
      key: password
---
# KEDA ScaledObject สำหรับ HTTP request rate
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: api-gateway-http-scaler
  namespace: production
spec:
  scaleTargetRef:
    name: api-gateway
  pollingInterval: 10
  cooldownPeriod: 120
  minReplicaCount: 3
  maxReplicaCount: 50
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://prometheus.monitoring.svc:9090
        metricName: http_requests_total
        query: |
          sum(rate(http_requests_total{
            namespace="production",
            service="api-gateway"
          }[2m]))
        threshold: "500"         # Scale when > 500 req/s total
        activationThreshold: "10"
```

---

## 79.7 Load Testing กับ k6

### Ramp-up, Stress, และ Soak Tests

```javascript
// k6-load-tests.js

import http from 'k6/http';
import { check, sleep, group } from 'k6';
import { Rate, Trend, Counter } from 'k6/metrics';

// Custom metrics
const errorRate = new Rate('error_rate');
const orderCreationDuration = new Trend('order_creation_duration', true);
const orderQueryDuration = new Trend('order_query_duration', true);
const totalOrders = new Counter('total_orders_created');

const BASE_URL = __ENV.BASE_URL || 'http://api-gateway.production.svc';
const USERS_COUNT = parseInt(__ENV.USERS_COUNT || '100');

// =================== Ramp-up Test ===================
export const rampUpOptions = {
  scenarios: {
    ramp_up: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '2m', target: 10 },     // Ramp up ช้าๆ
        { duration: '5m', target: 50 },     // เพิ่มขึ้น
        { duration: '5m', target: 100 },    // เพิ่มอีก
        { duration: '3m', target: 0 },      // Ramp down
      ],
      gracefulRampDown: '30s',
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<500', 'p(99)<1000'],
    error_rate: ['rate<0.01'],
    http_req_failed: ['rate<0.01'],
  },
};

// =================== Stress Test ===================
export const stressOptions = {
  scenarios: {
    stress: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '1m', target: 100 },
        { duration: '2m', target: 200 },
        { duration: '2m', target: 300 },
        { duration: '2m', target: 400 },
        { duration: '2m', target: 500 },    // ผลักไปถึง breaking point
        { duration: '5m', target: 500 },    // ทดสอบที่ peak
        { duration: '3m', target: 0 },      // Recovery
      ],
    },
  },
  thresholds: {
    http_req_duration: ['p(99)<2000'],
    error_rate: ['rate<0.05'],
  },
};

// =================== Soak Test ===================
export const soakOptions = {
  scenarios: {
    soak: {
      executor: 'constant-vus',
      vus: 100,
      duration: '4h',    // ทดสอบ 4 ชั่วโมง
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<500'],
    error_rate: ['rate<0.01'],
    // Memory leak detection: latency ไม่ควรเพิ่มขึ้นตาม time
  },
};

// Export สำหรับ default test (ramp-up)
export const options = rampUpOptions;

// Test data
const TEST_USERS = Array.from({ length: USERS_COUNT }, (_, i) => ({
  id: `user-${i + 1}`,
  email: `user${i + 1}@test.com`,
  token: `test-token-${i + 1}`,
}));

function getRandomUser() {
  return TEST_USERS[Math.floor(Math.random() * TEST_USERS.length)];
}

function getRandomProduct() {
  const products = [
    { id: 'prod-1', name: 'Widget A', price: 29.99 },
    { id: 'prod-2', name: 'Widget B', price: 49.99 },
    { id: 'prod-3', name: 'Widget C', price: 19.99 },
  ];
  return products[Math.floor(Math.random() * products.length)];
}

// Main test function
export default function () {
  const user = getRandomUser();
  const headers = {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${user.token}`,
  };

  group('User Journey', () => {
    // 1. Create Order
    group('Create Order', () => {
      const product = getRandomProduct();
      const payload = JSON.stringify({
        customerId: user.id,
        items: [
          {
            productId: product.id,
            quantity: Math.floor(Math.random() * 3) + 1,
            price: product.price,
          },
        ],
        shippingAddress: '123 Test Street, Bangkok 10100',
      });

      const startTime = Date.now();
      const response = http.post(
        `${BASE_URL}/api/orders`,
        payload,
        { headers, timeout: '10s' }
      );
      orderCreationDuration.add(Date.now() - startTime);

      const success = check(response, {
        'order created': (r) => r.status === 201,
        'has order id': (r) => {
          try {
            return JSON.parse(r.body as string).orderId !== undefined;
          } catch {
            return false;
          }
        },
      });

      errorRate.add(!success);
      if (success) totalOrders.add(1);
    });

    sleep(Math.random() * 2 + 0.5);

    // 2. List Orders
    group('List Orders', () => {
      const startTime = Date.now();
      const response = http.get(
        `${BASE_URL}/api/orders?customerId=${user.id}&page=1&pageSize=10`,
        { headers, timeout: '5s' }
      );
      orderQueryDuration.add(Date.now() - startTime);

      const success = check(response, {
        'orders listed': (r) => r.status === 200,
        'has data': (r) => {
          try {
            const body = JSON.parse(r.body as string);
            return Array.isArray(body.data);
          } catch {
            return false;
          }
        },
        'response time < 500ms': (r) => r.timings.duration < 500,
      });

      errorRate.add(!success);
    });

    sleep(Math.random() * 1 + 0.2);

    // 3. Health check
    group('Health Check', () => {
      const response = http.get(`${BASE_URL}/health`, { timeout: '2s' });
      check(response, {
        'healthy': (r) => r.status === 200,
      });
    });
  });

  sleep(1);
}

// Teardown
export function teardown() {
  console.log('Load test completed');
}
```

---

## 79.8 Capacity Modeling: TypeScript CapacityCalculator

```typescript
// capacity-calculator.ts

interface ServiceProfile {
  name: string;
  peakRps: number;                // Peak requests per second
  avgResponseTimeMs: number;      // Average response time
  cpuPerRequestMs: number;        // CPU ms consumed per request
  memoryPerPodMB: number;         // Memory needed per pod
  networkInKBPerRequest: number;  // Inbound bytes per request
  networkOutKBPerRequest: number; // Outbound bytes per request
  dbQueryCount: number;           // DB queries per request
  dbQueryTimeMs: number;          // Average DB query time
}

interface ResourceRequirements {
  cpu: {
    requested: string;
    limit: string;
    totalCores: number;
  };
  memory: {
    requested: string;
    limit: string;
    totalGB: number;
  };
  replicas: {
    minimum: number;
    maximum: number;
    recommended: number;
  };
  network: {
    inboundMBps: number;
    outboundMBps: number;
  };
  database: {
    queriesPerSecond: number;
    connectionsNeeded: number;
  };
}

interface ClusterCapacity {
  totalCPUCores: number;
  totalMemoryGB: number;
  nodeCount: number;
  nodeType: string;
}

class CapacityCalculator {
  private readonly cpuOverheadFactor = 1.3;    // 30% overhead สำหรับ JVM, sidecar ฯลฯ
  private readonly memoryOverheadFactor = 1.5;  // 50% overhead
  private readonly safetyMargin = 1.25;         // 25% safety margin
  private readonly replicaOverhead = 1.2;       // Extra 20% for rolling updates

  calculateServiceRequirements(profile: ServiceProfile): ResourceRequirements {
    // คำนวณ CPU ที่ต้องการ
    // CPU = (RPS × CPU_per_request_ms) / 1000ms × overhead
    const cpuCoresNeeded = 
      (profile.peakRps * profile.cpuPerRequestMs / 1000) * 
      this.cpuOverheadFactor * 
      this.safetyMargin;

    // คำนวณ replicas ที่ต้องการ
    // ให้แต่ละ pod ใช้ CPU ไม่เกิน 80%
    const cpuPerPod = 0.5;  // 0.5 core per pod (for medium-sized services)
    const cpuUtilizationTarget = 0.70;
    
    const replicasForCPU = Math.ceil(
      cpuCoresNeeded / (cpuPerPod * cpuUtilizationTarget)
    );

    // คำนวณ replicas สำหรับ concurrency
    // Little's Law: N = λ × W (N = concurrency, λ = arrival rate, W = service time)
    const avgConcurrency = 
      profile.peakRps * (profile.avgResponseTimeMs / 1000);
    
    const maxConcurrencyPerPod = 100; // typical for Node.js/Go
    const replicasForConcurrency = Math.ceil(
      avgConcurrency / (maxConcurrencyPerPod * cpuUtilizationTarget)
    );

    const recommendedReplicas = Math.max(
      3,  // minimum for HA
      Math.max(replicasForCPU, replicasForConcurrency)
    );

    const finalReplicas = Math.ceil(recommendedReplicas * this.replicaOverhead);
    
    // Memory calculation
    const totalMemoryGB = 
      (profile.memoryPerPodMB * finalReplicas * this.memoryOverheadFactor) / 1024;

    // Network bandwidth
    const inboundMBps = 
      (profile.peakRps * profile.networkInKBPerRequest) / 1024;
    const outboundMBps = 
      (profile.peakRps * profile.networkOutKBPerRequest) / 1024;

    // Database connections
    // Each pod maintains a connection pool
    const connectionsPerPod = 10;
    const totalDbConnections = finalReplicas * connectionsPerPod;

    return {
      cpu: {
        requested: `${Math.ceil(cpuCoresNeeded / finalReplicas * 1000)}m`,
        limit: `${Math.ceil(cpuCoresNeeded / finalReplicas * 1000 * 2)}m`,
        totalCores: cpuCoresNeeded,
      },
      memory: {
        requested: `${profile.memoryPerPodMB}Mi`,
        limit: `${Math.ceil(profile.memoryPerPodMB * 1.5)}Mi`,
        totalGB: totalMemoryGB,
      },
      replicas: {
        minimum: 3,
        maximum: finalReplicas * 3,
        recommended: finalReplicas,
      },
      network: {
        inboundMBps,
        outboundMBps,
      },
      database: {
        queriesPerSecond: profile.peakRps * profile.dbQueryCount,
        connectionsNeeded: totalDbConnections,
      },
    };
  }

  checkClusterFit(
    requirements: ResourceRequirements[],
    cluster: ClusterCapacity
  ): { fits: boolean; utilizationCPU: number; utilizationMemory: number; warnings: string[] } {
    const totalCPU = requirements.reduce((sum, r) => sum + r.cpu.totalCores, 0);
    const totalMemory = requirements.reduce((sum, r) => sum + r.memory.totalGB, 0);
    
    const utilizationCPU = (totalCPU / cluster.totalCPUCores) * 100;
    const utilizationMemory = (totalMemory / cluster.totalMemoryGB) * 100;
    
    const warnings: string[] = [];
    
    if (utilizationCPU > 80) {
      warnings.push(`CPU utilization ${utilizationCPU.toFixed(1)}% exceeds 80% target`);
    }
    if (utilizationMemory > 80) {
      warnings.push(`Memory utilization ${utilizationMemory.toFixed(1)}% exceeds 80% target`);
    }
    
    return {
      fits: utilizationCPU <= 80 && utilizationMemory <= 80,
      utilizationCPU,
      utilizationMemory,
      warnings,
    };
  }

  generateReport(
    profiles: ServiceProfile[],
    cluster: ClusterCapacity
  ): void {
    console.log('\n=== Capacity Planning Report ===\n');
    
    const allRequirements: ResourceRequirements[] = [];
    
    profiles.forEach(profile => {
      const req = this.calculateServiceRequirements(profile);
      allRequirements.push(req);
      
      console.log(`Service: ${profile.name}`);
      console.log(`  Peak RPS: ${profile.peakRps.toLocaleString()}`);
      console.log(`  Recommended Replicas: ${req.replicas.recommended} (max: ${req.replicas.maximum})`);
      console.log(`  CPU per pod: ${req.cpu.requested} (limit: ${req.cpu.limit})`);
      console.log(`  Memory per pod: ${req.memory.requested} (limit: ${req.memory.limit})`);
      console.log(`  Total CPU needed: ${req.cpu.totalCores.toFixed(2)} cores`);
      console.log(`  Total Memory needed: ${req.memory.totalGB.toFixed(1)} GB`);
      console.log(`  Network: ${req.network.inboundMBps.toFixed(1)} MB/s in, ${req.network.outboundMBps.toFixed(1)} MB/s out`);
      console.log(`  DB connections needed: ${req.database.connectionsNeeded}`);
      console.log('');
    });
    
    const { fits, utilizationCPU, utilizationMemory, warnings } = 
      this.checkClusterFit(allRequirements, cluster);
    
    console.log(`Cluster: ${cluster.nodeCount}x ${cluster.nodeType}`);
    console.log(`  Total CPU: ${cluster.totalCPUCores} cores`);
    console.log(`  Total Memory: ${cluster.totalMemoryGB} GB`);
    console.log(`  CPU Utilization: ${utilizationCPU.toFixed(1)}%`);
    console.log(`  Memory Utilization: ${utilizationMemory.toFixed(1)}%`);
    console.log(`  Fits: ${fits ? 'YES' : 'NO - NEED MORE CAPACITY'}`);
    
    if (warnings.length > 0) {
      console.log('\nWarnings:');
      warnings.forEach(w => console.log(`  ⚠️  ${w}`));
    }
  }
}

// ตัวอย่างการใช้งาน
const calculator = new CapacityCalculator();
calculator.generateReport(
  [
    {
      name: 'api-gateway',
      peakRps: 10000,
      avgResponseTimeMs: 50,
      cpuPerRequestMs: 2,
      memoryPerPodMB: 512,
      networkInKBPerRequest: 2,
      networkOutKBPerRequest: 5,
      dbQueryCount: 0,
      dbQueryTimeMs: 0,
    },
    {
      name: 'order-service',
      peakRps: 2000,
      avgResponseTimeMs: 200,
      cpuPerRequestMs: 15,
      memoryPerPodMB: 256,
      networkInKBPerRequest: 5,
      networkOutKBPerRequest: 3,
      dbQueryCount: 5,
      dbQueryTimeMs: 20,
    },
    {
      name: 'user-service',
      peakRps: 5000,
      avgResponseTimeMs: 30,
      cpuPerRequestMs: 3,
      memoryPerPodMB: 256,
      networkInKBPerRequest: 1,
      networkOutKBPerRequest: 2,
      dbQueryCount: 2,
      dbQueryTimeMs: 5,
    },
  ],
  {
    totalCPUCores: 96,
    totalMemoryGB: 384,
    nodeCount: 12,
    nodeType: 'm5.2xlarge (8 CPU, 32 GB)',
  }
);
```

---

## 79.9 Connection Pooling at Scale: PgBouncer

### PgBouncer Configuration

```ini
# pgbouncer.ini

[databases]
# production database
production = host=postgres-primary.production.svc port=5432 dbname=production
production_read = host=postgres-replica.production.svc port=5432 dbname=production

[pgbouncer]
# Listening
listen_addr = 0.0.0.0
listen_port = 5432

# Authentication
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt

# Connection pooling mode
# transaction: สำหรับ microservices (default สำหรับ scalability)
# session: สำหรับ applications ที่ใช้ SET, LISTEN, prepared statements
# statement: สำหรับ simple query workloads
pool_mode = transaction

# Connection limits
# Maximum connections to PostgreSQL backend
max_client_conn = 10000      # Maximum client connections
default_pool_size = 25       # Server connections per database/user combination
min_pool_size = 5            # Minimum connections kept in pool
reserve_pool_size = 5        # Extra connections available during peak
reserve_pool_timeout = 3     # Seconds to wait before using reserve pool

# Connection timeouts
server_connect_timeout = 15  # Seconds to wait for new server connection
server_idle_timeout = 600    # Seconds before idle server connection removed
client_idle_timeout = 0      # Disable (managed by application)
query_timeout = 0            # Disable query timeout (managed by application)
query_wait_timeout = 120     # Seconds client waits for server connection

# Keep-alive
tcp_keepalive = 1
tcp_keepcnt = 3
tcp_keepidle = 300
tcp_keepintvl = 5

# Logging
log_connections = 1
log_disconnections = 1
log_pooler_errors = 1
stats_period = 60

# Admin access
admin_users = pgbouncer
stats_users = pgbouncer, monitoring

# TLS (for encryption in transit)
server_tls_sslmode = require
server_tls_ca_file = /etc/pgbouncer/ca.crt
client_tls_sslmode = prefer
client_tls_cert_file = /etc/pgbouncer/server.crt
client_tls_key_file = /etc/pgbouncer/server.key
```

### Kubernetes Deployment สำหรับ PgBouncer

```yaml
# pgbouncer-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pgbouncer
  namespace: production
spec:
  replicas: 3    # Multiple PgBouncer instances for HA
  selector:
    matchLabels:
      app: pgbouncer
  template:
    metadata:
      labels:
        app: pgbouncer
    spec:
      containers:
        - name: pgbouncer
          image: bitnami/pgbouncer:1.21.0
          ports:
            - containerPort: 5432
              name: postgres
          env:
            - name: PGBOUNCER_DATABASE
              value: "production"
            - name: POSTGRESQL_HOST
              value: "postgres-primary.production.svc"
            - name: POSTGRESQL_PORT
              value: "5432"
            - name: POSTGRESQL_USERNAME
              valueFrom:
                secretKeyRef:
                  name: postgres-credentials
                  key: username
            - name: POSTGRESQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-credentials
                  key: password
            - name: PGBOUNCER_POOL_MODE
              value: "transaction"
            - name: PGBOUNCER_MAX_CLIENT_CONN
              value: "10000"
            - name: PGBOUNCER_DEFAULT_POOL_SIZE
              value: "25"
          resources:
            requests:
              cpu: "100m"
              memory: "64Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
          readinessProbe:
            tcpSocket:
              port: 5432
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            tcpSocket:
              port: 5432
            initialDelaySeconds: 15
            periodSeconds: 20
---
apiVersion: v1
kind: Service
metadata:
  name: pgbouncer
  namespace: production
spec:
  selector:
    app: pgbouncer
  ports:
    - port: 5432
      targetPort: 5432
  type: ClusterIP
```

---

## 79.10 Global Load Balancing: AWS Route53 Latency Routing, GeoDNS

### AWS Route53 Latency-Based Routing

```typescript
// route53-latency-routing.ts
import {
  Route53Client,
  ChangeResourceRecordSetsCommand,
  CreateHealthCheckCommand,
  ListHealthChecksCommand,
  Change,
  ResourceRecord,
} from '@aws-sdk/client-route-53';

interface RegionalEndpoint {
  region: string;
  endpoint: string;
  ipAddress: string;
  healthCheckId?: string;
}

interface LatencyRoutingConfig {
  hostedZoneId: string;
  recordName: string;
  recordType: 'A' | 'CNAME' | 'AAAA';
  ttl: number;
  endpoints: RegionalEndpoint[];
}

class Route53LatencyRoutingManager {
  private client: Route53Client;

  constructor(region: string = 'us-east-1') {
    this.client = new Route53Client({ region });
  }

  async createHealthCheck(endpoint: RegionalEndpoint): Promise<string> {
    const command = new CreateHealthCheckCommand({
      CallerReference: `${endpoint.region}-${Date.now()}`,
      HealthCheckConfig: {
        Type: 'HTTP',
        IPAddress: endpoint.ipAddress,
        Port: 80,
        ResourcePath: '/health',
        FullyQualifiedDomainName: endpoint.endpoint,
        RequestInterval: 30,
        FailureThreshold: 3,
        MeasureLatency: true,
        EnableSNI: true,
        Regions: ['us-east-1', 'us-west-2', 'eu-west-1'],
      },
    });

    const response = await this.client.send(command);
    return response.HealthCheck!.Id!;
  }

  async setupLatencyRouting(config: LatencyRoutingConfig): Promise<void> {
    const changes: Change[] = [];

    for (const endpoint of config.endpoints) {
      let healthCheckId = endpoint.healthCheckId;
      
      if (!healthCheckId) {
        console.log(`Creating health check for ${endpoint.region}...`);
        healthCheckId = await this.createHealthCheck(endpoint);
        endpoint.healthCheckId = healthCheckId;
      }

      const record: ResourceRecord = config.recordType === 'CNAME'
        ? { Value: endpoint.endpoint }
        : { Value: endpoint.ipAddress };

      changes.push({
        Action: 'UPSERT',
        ResourceRecordSet: {
          Name: config.recordName,
          Type: config.recordType,
          Region: endpoint.region as any,
          SetIdentifier: `latency-${endpoint.region}`,
          TTL: config.ttl,
          HealthCheckId: healthCheckId,
          ResourceRecords: [record],
        },
      });
    }

    const command = new ChangeResourceRecordSetsCommand({
      HostedZoneId: config.hostedZoneId,
      ChangeBatch: {
        Comment: `Latency routing for ${config.recordName}`,
        Changes: changes,
      },
    });

    await this.client.send(command);
    console.log(`Latency routing configured for ${config.recordName}`);
  }

  async setupGeoDNS(
    hostedZoneId: string,
    recordName: string,
    geoRoutes: Array<{
      continentCode?: string;
      countryCode?: string;
      endpoint: string;
      setIdentifier: string;
    }>
  ): Promise<void> {
    const changes: Change[] = geoRoutes.map(route => ({
      Action: 'UPSERT' as const,
      ResourceRecordSet: {
        Name: recordName,
        Type: 'CNAME' as const,
        GeoLocation: route.continentCode
          ? { ContinentCode: route.continentCode }
          : { CountryCode: route.countryCode },
        SetIdentifier: route.setIdentifier,
        TTL: 60,
        ResourceRecords: [{ Value: route.endpoint }],
      },
    }));

    const command = new ChangeResourceRecordSetsCommand({
      HostedZoneId: hostedZoneId,
      ChangeBatch: {
        Comment: `GeoDNS routing for ${recordName}`,
        Changes: changes,
      },
    });

    await this.client.send(command);
    console.log(`GeoDNS configured for ${recordName}`);
  }
}

// ตัวอย่างการใช้งาน
async function setupGlobalLoadBalancing() {
  const manager = new Route53LatencyRoutingManager();
  
  // Latency-based routing - ส่ง traffic ไป region ที่ latency ต่ำสุด
  await manager.setupLatencyRouting({
    hostedZoneId: process.env.HOSTED_ZONE_ID!,
    recordName: 'api.example.com',
    recordType: 'CNAME',
    ttl: 60,
    endpoints: [
      {
        region: 'us-east-1',
        endpoint: 'alb-us-east.example.com',
        ipAddress: '52.1.2.3',
      },
      {
        region: 'ap-southeast-1',
        endpoint: 'alb-ap-se.example.com',
        ipAddress: '54.1.2.3',
      },
      {
        region: 'eu-west-1',
        endpoint: 'alb-eu-west.example.com',
        ipAddress: '34.1.2.3',
      },
    ],
  });

  // GeoDNS - routing ตาม geography
  await manager.setupGeoDNS(
    process.env.HOSTED_ZONE_ID!,
    'static.example.com',
    [
      {
        continentCode: 'NA',
        endpoint: 'cdn-us.example.com',
        setIdentifier: 'north-america',
      },
      {
        continentCode: 'EU',
        endpoint: 'cdn-eu.example.com',
        setIdentifier: 'europe',
      },
      {
        continentCode: 'AS',
        endpoint: 'cdn-ap.example.com',
        setIdentifier: 'asia',
      },
    ]
  );
}

setupGlobalLoadBalancing().catch(console.error);
```

---

## สรุปบทที่ 79

| Pattern | เมื่อใช้ | ประโยชน์ |
|---------|----------|----------|
| Stateless Service + Redis Session | ทุก microservice ที่ scale out | Scale ได้ไม่จำกัด, ทน fault |
| Read Replicas | DB read-heavy workloads | ลด load บน primary, เพิ่ม read throughput |
| CQRS | Complex domain, read/write ratio ต่าง | Optimize แต่ละ side แยกกัน |
| JWT (Stateless Auth) | Distributed systems | ไม่ต้องแชร์ state, scale ได้ |
| Kafka Partitioning | Event streaming at scale | Parallelism, ordering guarantees |
| KEDA ScaledObject | Event-driven scaling | Scale to zero, scale by external metrics |
| k6 Load Testing | ก่อน production deployment | ค้นหา bottleneck, ทดสอบ capacity |
| CapacityCalculator | Capacity planning | ป้องกัน over/under-provisioning |
| PgBouncer | DB connection pooling | ลด DB connection overhead |
| Route53 Latency | Multi-region deployment | ลด latency สำหรับ users ทั่วโลก |

### Key Takeaways

1. **Stateless first**: ออกแบบ service ให้ stateless ก่อนเสมอ ใช้ Redis สำหรับ session
2. **แยก Read/Write**: ใช้ read replicas + CQRS เพื่อ scale read workloads แยกจาก write
3. **JWT สำหรับ auth**: ลดการ query DB ทุก request
4. **Kafka partitions = max consumers**: ออกแบบ partitions ให้พอกับ consumer groups
5. **KEDA สำหรับ event-driven**: Scale by business metrics ไม่ใช่แค่ CPU/memory
6. **Test before production**: k6 load test ทุก deployment สำคัญ
7. **Capacity planning**: คำนวณ resource ล่วงหน้าก่อน go-live

---

*Part 79 จบแล้ว - ต่อไปบทที่ 80: Kubernetes Advanced Patterns*

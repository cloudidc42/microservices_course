# Part 79: Scalability Patterns

## บทนำ

Scalability คือความสามารถของระบบในการรองรับ load ที่เพิ่มขึ้น บทนี้ครอบคลุม Horizontal/Vertical Scaling, Database Replicas, CQRS, Event Sourcing, Stateless Design, Session Management, Caching, Kafka Scaling, Auto-scaling และ Load Testing

---

## 1. Horizontal vs Vertical Scaling

```typescript
// scaling-strategies.ts
// เปรียบเทียบและ implement scaling strategies

interface ScalingStrategy {
  type: 'horizontal' | 'vertical';
  trigger: 'manual' | 'auto';
  metric: string;
  threshold: number;
  cooldown: number; // seconds
}

// Horizontal Scaling Configuration
const horizontalScalingConfig = {
  minReplicas: 2,
  maxReplicas: 100,
  targetCPUUtilizationPercentage: 60,
  targetMemoryUtilizationPercentage: 70,
  scaleUpCooldown: 60,   // seconds
  scaleDownCooldown: 300, // seconds
  scaleUpSteps: [
    { pods: 1, periodSeconds: 60 },    // Add 1 pod in 60s
    { pods: 5, periodSeconds: 60 },    // Add up to 5 pods in 60s
  ],
  scaleDownSteps: [
    { pods: 1, periodSeconds: 120 },   // Remove 1 pod in 120s (careful)
  ],
};

// Kubernetes HPA with KEDA
const scalingDecision = {
  cpuBased: {
    advantages: [
      'Simple to configure',
      'Built-in Kubernetes metric',
      'Works for CPU-intensive workloads',
    ],
    disadvantages: [
      'Lagging indicator',
      'Not suitable for I/O-bound workloads',
    ],
  },
  customMetricBased: {
    advantages: [
      'Business-relevant metrics',
      'Request queue depth',
      'More predictive scaling',
    ],
    examples: [
      'kafka_consumer_lag',
      'http_requests_per_second',
      'active_connections',
      'queue_depth',
    ],
  },
};
```

```yaml
# hpa-advanced.yaml
# Advanced HPA with custom metrics

apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
  namespace: microservices
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  
  minReplicas: 2
  maxReplicas: 50
  
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
    
    # Custom metric: RPS per pod
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: 100
    
    # External metric: Kafka consumer lag
    - type: External
      external:
        metric:
          name: kafka_consumer_lag_sum
          selector:
            matchLabels:
              topic: order-events
              consumer_group: order-processor
        target:
          type: AverageValue
          averageValue: 1000
  
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Pods
          value: 5
          periodSeconds: 60
        - type: Percent
          value: 50
          periodSeconds: 60
      selectPolicy: Max
    
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 1
          periodSeconds: 120
      selectPolicy: Min
---
# KEDA ScaledObject for Kafka-based scaling
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-service-keda
  namespace: microservices
spec:
  scaleTargetRef:
    name: order-service
  minReplicaCount: 2
  maxReplicaCount: 50
  pollingInterval: 15
  cooldownPeriod: 60
  
  triggers:
    - type: kafka
      metadata:
        bootstrapServers: kafka:9092
        consumerGroup: order-processor
        topic: order-events
        lagThreshold: "100"
        offsetResetPolicy: latest
    
    - type: prometheus
      metadata:
        serverAddress: http://prometheus:9090
        metricName: http_requests_per_second
        threshold: "100"
        query: sum(rate(http_requests_total{app="order-service"}[2m]))
```

---

## 2. Database Read Replicas and Connection Pooling

```typescript
// db-scaling.ts
// Database read replicas และ connection pooling

import { Pool, PoolConfig } from 'pg';
import PGBouncer from 'pg-bouncer'; // conceptual

interface DatabaseConfig {
  primary: {
    host: string;
    port: number;
    database: string;
    user: string;
    password: string;
    maxConnections: number;
  };
  replicas: Array<{
    host: string;
    port: number;
    weight: number;
  }>;
  poolerConfig: {
    poolMode: 'transaction' | 'session' | 'statement';
    maxClientConn: number;
    defaultPoolSize: number;
  };
}

class DatabaseCluster {
  private primaryPool: Pool;
  private replicaPools: Pool[];
  private currentReplicaIndex = 0;

  constructor(config: DatabaseConfig) {
    // Primary pool - lower connection limit, high priority
    this.primaryPool = new Pool({
      host: config.primary.host,
      port: config.primary.port,
      database: config.primary.database,
      user: config.primary.user,
      password: config.primary.password,
      max: config.primary.maxConnections,
      idleTimeoutMillis: 30000,
      connectionTimeoutMillis: 2000,
      keepAlive: true,
      keepAliveInitialDelayMillis: 10000,
    });

    // Replica pools - higher connection limit, read only
    this.replicaPools = config.replicas.map(replica =>
      new Pool({
        host: replica.host,
        port: replica.port,
        database: config.primary.database,
        user: config.primary.user,
        password: config.primary.password,
        max: config.primary.maxConnections * 2,
        idleTimeoutMillis: 30000,
        connectionTimeoutMillis: 2000,
      })
    );

    this.setupHealthMonitoring();
  }

  // Write to primary
  async write<T>(
    query: string,
    params?: unknown[]
  ): Promise<{ rows: T[]; rowCount: number }> {
    return this.primaryPool.query(query, params);
  }

  // Read from replica (round-robin)
  async read<T>(
    query: string,
    params?: unknown[]
  ): Promise<{ rows: T[]; rowCount: number }> {
    if (this.replicaPools.length === 0) {
      // Fallback to primary if no replicas
      return this.primaryPool.query(query, params);
    }

    const replica = this.getNextReplica();
    
    try {
      return await replica.query(query, params);
    } catch (error) {
      // Fallback to primary on replica failure
      console.warn('Replica read failed, falling back to primary:', error);
      return this.primaryPool.query(query, params);
    }
  }

  // Consistent read (must read from primary)
  async consistentRead<T>(
    query: string,
    params?: unknown[]
  ): Promise<{ rows: T[]; rowCount: number }> {
    return this.primaryPool.query(query, params);
  }

  private getNextReplica(): Pool {
    const replica = this.replicaPools[this.currentReplicaIndex];
    this.currentReplicaIndex = (this.currentReplicaIndex + 1) % this.replicaPools.length;
    return replica;
  }

  private setupHealthMonitoring(): void {
    setInterval(async () => {
      // Monitor primary lag
      try {
        const result = await this.primaryPool.query<{ lag: string }>(
          `SELECT CASE 
             WHEN pg_is_in_recovery() THEN pg_last_wal_receive_lsn() - pg_last_wal_replay_lsn()
             ELSE 0
           END as lag`
        );
        const lagBytes = parseInt(result.rows[0].lag);
        if (lagBytes > 1024 * 1024) { // 1MB lag
          console.warn(`Replica lag: ${lagBytes} bytes`);
        }
      } catch (error) {
        console.error('Health check failed:', error);
      }
    }, 10000);
  }

  async getPoolStats(): Promise<{
    primary: { total: number; idle: number; waiting: number };
    replicas: Array<{ total: number; idle: number; waiting: number }>;
  }> {
    return {
      primary: {
        total: this.primaryPool.totalCount,
        idle: this.primaryPool.idleCount,
        waiting: this.primaryPool.waitingCount,
      },
      replicas: this.replicaPools.map(pool => ({
        total: pool.totalCount,
        idle: pool.idleCount,
        waiting: pool.waitingCount,
      })),
    };
  }
}

// PgBouncer configuration
const pgBouncerConfig = `
[databases]
order_db = host=postgres-primary port=5432 dbname=order_db
order_db_replica = host=postgres-replica port=5432 dbname=order_db

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 5432
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt
pool_mode = transaction
max_client_conn = 10000
default_pool_size = 25
min_pool_size = 5
reserve_pool_size = 5
reserve_pool_timeout = 3
server_idle_timeout = 600
server_connect_timeout = 5
server_login_retry = 3
query_timeout = 30
client_idle_timeout = 60
`;
```

---

## 3. CQRS for Scalability

```typescript
// cqrs-pattern.ts
// Command Query Responsibility Segregation

// Commands (Write Side)
interface CreateOrderCommand {
  type: 'CREATE_ORDER';
  userId: string;
  items: OrderItem[];
  correlationId: string;
}

interface UpdateOrderStatusCommand {
  type: 'UPDATE_ORDER_STATUS';
  orderId: string;
  newStatus: OrderStatus;
  correlationId: string;
}

type Command = CreateOrderCommand | UpdateOrderStatusCommand;

// Queries (Read Side)
interface GetOrderQuery {
  type: 'GET_ORDER';
  orderId: string;
}

interface GetOrdersByUserQuery {
  type: 'GET_ORDERS_BY_USER';
  userId: string;
  page: number;
  pageSize: number;
  status?: OrderStatus;
}

interface GetOrderSummaryQuery {
  type: 'GET_ORDER_SUMMARY';
  userId: string;
  dateFrom: Date;
  dateTo: Date;
}

type Query = GetOrderQuery | GetOrdersByUserQuery | GetOrderSummaryQuery;

// Write Model
class OrderCommandHandler {
  private writeDb: Pool;
  private eventBus: EventEmitter;

  constructor(writeDb: Pool, eventBus: EventEmitter) {
    this.writeDb = writeDb;
    this.eventBus = eventBus;
  }

  async handle(command: Command): Promise<string> {
    switch (command.type) {
      case 'CREATE_ORDER':
        return this.handleCreateOrder(command);
      case 'UPDATE_ORDER_STATUS':
        return this.handleUpdateStatus(command);
      default:
        throw new Error(`Unknown command type`);
    }
  }

  private async handleCreateOrder(command: CreateOrderCommand): Promise<string> {
    const orderId = crypto.randomUUID();
    const totalAmount = command.items.reduce(
      (sum, item) => sum + item.price * item.quantity,
      0
    );

    await this.writeDb.query(
      `INSERT INTO orders (id, user_id, items, total_amount, status, created_at)
       VALUES ($1, $2, $3, $4, 'PENDING', NOW())`,
      [orderId, command.userId, JSON.stringify(command.items), totalAmount]
    );

    // Publish event for read model projection
    this.eventBus.emit('OrderCreated', {
      orderId,
      userId: command.userId,
      items: command.items,
      totalAmount,
      status: 'PENDING',
      createdAt: new Date(),
    });

    return orderId;
  }

  private async handleUpdateStatus(command: UpdateOrderStatusCommand): Promise<string> {
    await this.writeDb.query(
      'UPDATE orders SET status=$1, updated_at=NOW() WHERE id=$2',
      [command.newStatus, command.orderId]
    );

    this.eventBus.emit('OrderStatusUpdated', {
      orderId: command.orderId,
      newStatus: command.newStatus,
      updatedAt: new Date(),
    });

    return command.orderId;
  }
}

// Read Model (Denormalized, optimized for queries)
class OrderQueryHandler {
  private readDb: Pool;

  constructor(readDb: Pool) {
    this.readDb = readDb;
  }

  async handle(query: Query): Promise<unknown> {
    switch (query.type) {
      case 'GET_ORDER':
        return this.getOrder(query);
      case 'GET_ORDERS_BY_USER':
        return this.getOrdersByUser(query);
      case 'GET_ORDER_SUMMARY':
        return this.getOrderSummary(query);
    }
  }

  private async getOrder(query: GetOrderQuery): Promise<Order | null> {
    // Read from optimized read model (denormalized)
    const result = await this.readDb.query<Order>(
      `SELECT * FROM order_read_model WHERE id = $1`,
      [query.orderId]
    );
    return result.rows[0] || null;
  }

  private async getOrdersByUser(query: GetOrdersByUserQuery): Promise<{
    orders: Order[];
    total: number;
    page: number;
    pageSize: number;
  }> {
    const offset = (query.page - 1) * query.pageSize;
    
    const [ordersResult, countResult] = await Promise.all([
      this.readDb.query<Order>(
        `SELECT * FROM order_read_model 
         WHERE user_id = $1 
         ${query.status ? 'AND status = $3' : ''}
         ORDER BY created_at DESC 
         LIMIT $2 OFFSET ${query.status ? '$4' : '$3'}`,
        query.status
          ? [query.userId, query.pageSize, query.status, offset]
          : [query.userId, query.pageSize, offset]
      ),
      this.readDb.query<{ count: string }>(
        `SELECT COUNT(*) FROM order_read_model WHERE user_id = $1`,
        [query.userId]
      ),
    ]);

    return {
      orders: ordersResult.rows,
      total: parseInt(countResult.rows[0].count),
      page: query.page,
      pageSize: query.pageSize,
    };
  }

  private async getOrderSummary(query: GetOrderSummaryQuery): Promise<{
    totalOrders: number;
    totalSpend: number;
    avgOrderValue: number;
    statusBreakdown: Record<string, number>;
  }> {
    // Use pre-aggregated materialized view for performance
    const result = await this.readDb.query<{
      total_orders: string;
      total_spend: string;
      avg_order_value: string;
    }>(
      `SELECT 
         COUNT(*) as total_orders,
         SUM(total_amount) as total_spend,
         AVG(total_amount) as avg_order_value
       FROM order_read_model
       WHERE user_id = $1 AND created_at BETWEEN $2 AND $3`,
      [query.userId, query.dateFrom, query.dateTo]
    );

    const statusResult = await this.readDb.query<{
      status: string;
      count: string;
    }>(
      `SELECT status, COUNT(*) as count 
       FROM order_read_model
       WHERE user_id = $1 AND created_at BETWEEN $2 AND $3
       GROUP BY status`,
      [query.userId, query.dateFrom, query.dateTo]
    );

    const row = result.rows[0];
    return {
      totalOrders: parseInt(row.total_orders),
      totalSpend: parseFloat(row.total_spend || '0'),
      avgOrderValue: parseFloat(row.avg_order_value || '0'),
      statusBreakdown: Object.fromEntries(
        statusResult.rows.map(r => [r.status, parseInt(r.count)])
      ),
    };
  }
}

// Read Model Projector
class OrderProjector {
  private readDb: Pool;

  constructor(readDb: Pool) {
    this.readDb = readDb;
  }

  async handleOrderCreated(event: {
    orderId: string;
    userId: string;
    items: OrderItem[];
    totalAmount: number;
    status: string;
    createdAt: Date;
  }): Promise<void> {
    // Upsert to read model
    await this.readDb.query(
      `INSERT INTO order_read_model 
         (id, user_id, items, total_amount, status, item_count, created_at, updated_at)
       VALUES ($1, $2, $3, $4, $5, $6, $7, $7)
       ON CONFLICT (id) DO UPDATE
       SET status = excluded.status, updated_at = excluded.updated_at`,
      [
        event.orderId,
        event.userId,
        JSON.stringify(event.items),
        event.totalAmount,
        event.status,
        event.items.length,
        event.createdAt,
      ]
    );
  }
}

type OrderStatus = 'PENDING' | 'CONFIRMED' | 'SHIPPED' | 'DELIVERED' | 'CANCELLED';
interface OrderItem {
  productId: string;
  quantity: number;
  price: number;
}
interface Order {
  id: string;
  userId: string;
  status: OrderStatus;
  totalAmount: number;
  createdAt: Date;
}
import { Pool } from 'pg';
import { EventEmitter } from 'events';
```

---

## 4. Event Sourcing for Audit and Scalability

```typescript
// event-sourcing.ts

interface DomainEvent {
  eventId: string;
  eventType: string;
  aggregateId: string;
  aggregateType: string;
  version: number;
  payload: Record<string, unknown>;
  metadata: {
    correlationId: string;
    causationId: string;
    userId: string;
    timestamp: Date;
  };
}

class EventStore {
  private db: Pool;

  constructor(db: Pool) {
    this.db = db;
  }

  async append(
    aggregateId: string,
    events: Omit<DomainEvent, 'eventId' | 'version'>[],
    expectedVersion: number
  ): Promise<void> {
    const client = await this.db.connect();
    
    try {
      await client.query('BEGIN');

      // Optimistic locking check
      const currentVersion = await this.getCurrentVersion(aggregateId, client);
      
      if (currentVersion !== expectedVersion) {
        throw new Error(
          `Concurrency conflict: expected version ${expectedVersion}, got ${currentVersion}`
        );
      }

      // Append events
      for (let i = 0; i < events.length; i++) {
        const event = events[i];
        const version = expectedVersion + i + 1;

        await client.query(
          `INSERT INTO event_store 
           (event_id, event_type, aggregate_id, aggregate_type, version, payload, metadata)
           VALUES ($1, $2, $3, $4, $5, $6, $7)`,
          [
            crypto.randomUUID(),
            event.eventType,
            aggregateId,
            event.aggregateType,
            version,
            JSON.stringify(event.payload),
            JSON.stringify(event.metadata),
          ]
        );
      }

      await client.query('COMMIT');
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }

  async load(
    aggregateId: string,
    fromVersion?: number
  ): Promise<DomainEvent[]> {
    const result = await this.db.query<{
      event_id: string;
      event_type: string;
      aggregate_id: string;
      aggregate_type: string;
      version: number;
      payload: Record<string, unknown>;
      metadata: Record<string, unknown>;
      created_at: Date;
    }>(
      `SELECT * FROM event_store 
       WHERE aggregate_id = $1 
       ${fromVersion ? 'AND version > $2' : ''}
       ORDER BY version ASC`,
      fromVersion ? [aggregateId, fromVersion] : [aggregateId]
    );

    return result.rows.map(row => ({
      eventId: row.event_id,
      eventType: row.event_type,
      aggregateId: row.aggregate_id,
      aggregateType: row.aggregate_type,
      version: row.version,
      payload: row.payload,
      metadata: {
        correlationId: (row.metadata as any).correlationId,
        causationId: (row.metadata as any).causationId,
        userId: (row.metadata as any).userId,
        timestamp: row.created_at,
      },
    }));
  }

  private async getCurrentVersion(
    aggregateId: string,
    client: any
  ): Promise<number> {
    const result = await client.query(
      'SELECT MAX(version) as version FROM event_store WHERE aggregate_id = $1',
      [aggregateId]
    );
    return result.rows[0].version || 0;
  }
}

// Aggregate with Event Sourcing
class OrderAggregate {
  private id: string;
  private userId: string;
  private status: OrderStatus = 'PENDING';
  private items: OrderItem[] = [];
  private totalAmount: number = 0;
  private version: number = 0;
  private uncommittedEvents: DomainEvent[] = [];

  constructor(id: string) {
    this.id = id;
  }

  static fromEvents(events: DomainEvent[]): OrderAggregate {
    const aggregate = new OrderAggregate(events[0].aggregateId);
    
    for (const event of events) {
      aggregate.apply(event);
    }
    
    return aggregate;
  }

  create(userId: string, items: OrderItem[]): void {
    const totalAmount = items.reduce((sum, item) => sum + item.price * item.quantity, 0);
    
    this.raise({
      eventType: 'OrderCreated',
      payload: { userId, items, totalAmount },
    });
  }

  cancel(reason: string): void {
    if (this.status !== 'PENDING' && this.status !== 'CONFIRMED') {
      throw new Error(`Cannot cancel order in ${this.status} status`);
    }
    
    this.raise({
      eventType: 'OrderCancelled',
      payload: { reason },
    });
  }

  private raise(event: { eventType: string; payload: Record<string, unknown> }): void {
    const domainEvent: DomainEvent = {
      eventId: crypto.randomUUID(),
      eventType: event.eventType,
      aggregateId: this.id,
      aggregateType: 'Order',
      version: this.version + 1,
      payload: event.payload,
      metadata: {
        correlationId: crypto.randomUUID(),
        causationId: crypto.randomUUID(),
        userId: this.userId,
        timestamp: new Date(),
      },
    };

    this.apply(domainEvent);
    this.uncommittedEvents.push(domainEvent);
  }

  private apply(event: DomainEvent): void {
    switch (event.eventType) {
      case 'OrderCreated':
        this.userId = event.payload.userId as string;
        this.items = event.payload.items as OrderItem[];
        this.totalAmount = event.payload.totalAmount as number;
        this.status = 'PENDING';
        break;
      case 'OrderCancelled':
        this.status = 'CANCELLED';
        break;
    }
    this.version = event.version;
  }

  getUncommittedEvents(): DomainEvent[] {
    return this.uncommittedEvents;
  }

  clearUncommittedEvents(): void {
    this.uncommittedEvents = [];
  }

  getVersion(): number {
    return this.version;
  }
}
```

---

## 5. Stateless Services Design

```typescript
// stateless-services.ts
// Design principles สำหรับ stateless services

// Bad: Stateful service (ทำให้ scale ยาก)
class StatefulOrderService {
  private sessionCache = new Map<string, any>(); // State ใน memory!
  private userSessions = new Map<string, string>();

  async processOrder(sessionId: string, items: OrderItem[]): Promise<string> {
    // ปัญหา: ถ้า restart หรือ scale out -> state หาย
    const session = this.sessionCache.get(sessionId);
    if (!session) throw new Error('Session not found');
    
    return 'order-id'; // State ผูกกับ instance นี้
  }
}

// Good: Stateless service (scale ได้ง่าย)
class StatelessOrderService {
  constructor(
    private redis: Redis,      // External state
    private db: Pool,          // External persistence
    private tokenService: TokenService // Stateless token validation
  ) {}

  async processOrder(
    userId: string,
    items: OrderItem[],
    idempotencyKey: string  // ป้องกัน duplicate processing
  ): Promise<string> {
    // Idempotency check in Redis (shared state)
    const existing = await this.redis.get(`idempotency:${idempotencyKey}`);
    if (existing) return existing;

    // Process order
    const orderId = await this.createOrder(userId, items);

    // Store idempotency key (expires after 24 hours)
    await this.redis.setex(`idempotency:${idempotencyKey}`, 86400, orderId);

    return orderId;
  }

  private async createOrder(userId: string, items: OrderItem[]): Promise<string> {
    const result = await this.db.query<{ id: string }>(
      'INSERT INTO orders (user_id, items, status) VALUES ($1, $2, $3) RETURNING id',
      [userId, JSON.stringify(items), 'PENDING']
    );
    return result.rows[0].id;
  }
}

// JWT-based stateless authentication
class TokenService {
  private readonly secret: string;

  constructor(secret: string) {
    this.secret = secret;
  }

  generateToken(userId: string, roles: string[]): string {
    const jwt = require('jsonwebtoken');
    return jwt.sign(
      { sub: userId, roles, jti: crypto.randomUUID() },
      this.secret,
      { expiresIn: '15m' }
    );
  }

  validateToken(token: string): {
    valid: boolean;
    userId?: string;
    roles?: string[];
  } {
    const jwt = require('jsonwebtoken');
    try {
      const payload = jwt.verify(token, this.secret);
      return {
        valid: true,
        userId: payload.sub,
        roles: payload.roles,
      };
    } catch {
      return { valid: false };
    }
  }
}

import { Redis } from 'ioredis';
```

---

## 6. Session Management at Scale

```typescript
// session-management.ts
// Distributed session management

interface Session {
  id: string;
  userId: string;
  data: Record<string, unknown>;
  createdAt: number;
  lastAccessedAt: number;
  expiresAt: number;
}

class DistributedSessionManager {
  private redis: Redis;
  private readonly SESSION_TTL = 3600; // 1 hour
  private readonly SESSION_PREFIX = 'session:';

  constructor(redis: Redis) {
    this.redis = redis;
  }

  async create(userId: string, data: Record<string, unknown> = {}): Promise<string> {
    const sessionId = crypto.randomUUID();
    const now = Date.now();
    
    const session: Session = {
      id: sessionId,
      userId,
      data,
      createdAt: now,
      lastAccessedAt: now,
      expiresAt: now + this.SESSION_TTL * 1000,
    };

    await this.redis.setex(
      `${this.SESSION_PREFIX}${sessionId}`,
      this.SESSION_TTL,
      JSON.stringify(session)
    );

    // Track user sessions for revocation
    await this.redis.sadd(`user-sessions:${userId}`, sessionId);
    await this.redis.expire(`user-sessions:${userId}`, this.SESSION_TTL * 2);

    return sessionId;
  }

  async get(sessionId: string): Promise<Session | null> {
    const data = await this.redis.get(`${this.SESSION_PREFIX}${sessionId}`);
    
    if (!data) return null;

    const session = JSON.parse(data) as Session;
    
    // Check expiration
    if (session.expiresAt < Date.now()) {
      await this.delete(sessionId);
      return null;
    }

    // Sliding expiration - renew on access
    session.lastAccessedAt = Date.now();
    session.expiresAt = Date.now() + this.SESSION_TTL * 1000;
    
    await this.redis.setex(
      `${this.SESSION_PREFIX}${sessionId}`,
      this.SESSION_TTL,
      JSON.stringify(session)
    );

    return session;
  }

  async update(
    sessionId: string,
    updates: Partial<Session['data']>
  ): Promise<void> {
    const session = await this.get(sessionId);
    if (!session) throw new Error('Session not found');

    session.data = { ...session.data, ...updates };
    session.lastAccessedAt = Date.now();

    await this.redis.setex(
      `${this.SESSION_PREFIX}${sessionId}`,
      this.SESSION_TTL,
      JSON.stringify(session)
    );
  }

  async delete(sessionId: string): Promise<void> {
    const session = await this.get(sessionId);
    if (session) {
      await this.redis.srem(`user-sessions:${session.userId}`, sessionId);
    }
    await this.redis.del(`${this.SESSION_PREFIX}${sessionId}`);
  }

  async revokeAllUserSessions(userId: string): Promise<void> {
    const sessionIds = await this.redis.smembers(`user-sessions:${userId}`);
    
    const pipeline = this.redis.pipeline();
    sessionIds.forEach(id => pipeline.del(`${this.SESSION_PREFIX}${id}`));
    pipeline.del(`user-sessions:${userId}`);
    
    await pipeline.exec();
  }
}
```

---

## 7. Caching Strategies for Scale

```typescript
// caching-strategies.ts
// Multi-level caching

interface CacheConfig {
  l1: {
    type: 'memory';
    maxSize: number;
    ttl: number;
  };
  l2: {
    type: 'redis';
    ttl: number;
    cluster?: boolean;
  };
  l3: {
    type: 'cdn';
    ttl: number;
    regions: string[];
  };
}

class MultiLevelCache<T> {
  private l1Cache: Map<string, { value: T; expiresAt: number }>;
  private redis: Redis;
  private readonly maxL1Size: number;
  private readonly l1TTL: number;
  private readonly l2TTL: number;

  constructor(config: CacheConfig, redis: Redis) {
    this.l1Cache = new Map();
    this.redis = redis;
    this.maxL1Size = config.l1.maxSize;
    this.l1TTL = config.l1.ttl;
    this.l2TTL = config.l2.ttl;
  }

  async get(key: string): Promise<T | null> {
    // L1: In-memory (fastest)
    const l1Value = this.getFromL1(key);
    if (l1Value !== null) {
      return l1Value;
    }

    // L2: Redis (distributed)
    const l2Value = await this.getFromL2(key);
    if (l2Value !== null) {
      // Populate L1
      this.setInL1(key, l2Value);
      return l2Value;
    }

    return null;
  }

  async set(key: string, value: T): Promise<void> {
    // Set in all layers
    this.setInL1(key, value);
    await this.setInL2(key, value);
  }

  async invalidate(key: string): Promise<void> {
    this.l1Cache.delete(key);
    await this.redis.del(key);
  }

  async invalidatePattern(pattern: string): Promise<void> {
    // Clear L1 matching pattern
    for (const key of this.l1Cache.keys()) {
      if (this.matchesPattern(key, pattern)) {
        this.l1Cache.delete(key);
      }
    }

    // Clear L2 using SCAN (avoid KEYS in production)
    let cursor = '0';
    do {
      const result = await this.redis.scan(cursor, 'MATCH', pattern, 'COUNT', 100);
      cursor = result[0];
      const keys = result[1];
      
      if (keys.length > 0) {
        await this.redis.del(...keys);
      }
    } while (cursor !== '0');
  }

  private getFromL1(key: string): T | null {
    const entry = this.l1Cache.get(key);
    if (!entry) return null;
    
    if (entry.expiresAt < Date.now()) {
      this.l1Cache.delete(key);
      return null;
    }
    
    return entry.value;
  }

  private setInL1(key: string, value: T): void {
    // Evict if at capacity (simple LRU approximation)
    if (this.l1Cache.size >= this.maxL1Size) {
      const oldestKey = this.l1Cache.keys().next().value;
      if (oldestKey) this.l1Cache.delete(oldestKey);
    }

    this.l1Cache.set(key, {
      value,
      expiresAt: Date.now() + this.l1TTL * 1000,
    });
  }

  private async getFromL2(key: string): Promise<T | null> {
    const data = await this.redis.get(key);
    if (!data) return null;
    return JSON.parse(data) as T;
  }

  private async setInL2(key: string, value: T): Promise<void> {
    await this.redis.setex(key, this.l2TTL, JSON.stringify(value));
  }

  private matchesPattern(key: string, pattern: string): boolean {
    const regex = new RegExp(pattern.replace(/\*/g, '.*'));
    return regex.test(key);
  }
}

// Read-Through cache
class ReadThroughCache<T> {
  private cache: MultiLevelCache<T>;

  constructor(cache: MultiLevelCache<T>) {
    this.cache = cache;
  }

  async get(
    key: string,
    loader: () => Promise<T>,
    ttlOverride?: number
  ): Promise<T> {
    const cached = await this.cache.get(key);
    if (cached !== null) return cached;

    const fresh = await loader();
    await this.cache.set(key, fresh);
    return fresh;
  }
}

// Write-Through cache
class WriteThroughCache<T> {
  private cache: MultiLevelCache<T>;

  constructor(cache: MultiLevelCache<T>) {
    this.cache = cache;
  }

  async set(
    key: string,
    value: T,
    writer: (value: T) => Promise<void>
  ): Promise<void> {
    await writer(value);
    await this.cache.set(key, value);
  }
}
```

---

## 8. Message Queue Scaling (Kafka Partition Scaling)

```typescript
// kafka-scaling.ts
// Kafka partition management สำหรับ scaling

import { Kafka, Admin, Consumer, Producer } from 'kafkajs';

class KafkaScalingManager {
  private kafka: Kafka;
  private admin: Admin;

  constructor(brokers: string[]) {
    this.kafka = new Kafka({
      brokers,
      retry: {
        maxRetryTime: 30000,
        initialRetryTime: 300,
        retries: 10,
      },
    });
    this.admin = this.kafka.admin();
  }

  async scalePartitions(
    topic: string,
    newPartitionCount: number
  ): Promise<void> {
    await this.admin.connect();

    try {
      const topics = await this.admin.fetchTopicMetadata({ topics: [topic] });
      const currentPartitions = topics.topics[0].partitions.length;

      if (newPartitionCount <= currentPartitions) {
        throw new Error(
          `Cannot decrease partitions. Current: ${currentPartitions}, Requested: ${newPartitionCount}`
        );
      }

      console.log(`Scaling ${topic}: ${currentPartitions} -> ${newPartitionCount} partitions`);

      await this.admin.createPartitions({
        validateOnly: false,
        timeout: 30000,
        topicPartitions: [
          {
            topic,
            count: newPartitionCount,
          },
        ],
      });

      console.log(`Successfully scaled ${topic} to ${newPartitionCount} partitions`);
    } finally {
      await this.admin.disconnect();
    }
  }

  async getConsumerGroupLag(
    groupId: string
  ): Promise<Record<string, { partition: number; lag: bigint }[]>> {
    await this.admin.connect();

    try {
      const offsets = await this.admin.fetchOffsets({
        groupId,
        topics: [],
        resolveOffsets: false,
      });

      const topicLag: Record<string, { partition: number; lag: bigint }[]> = {};

      for (const topicOffset of offsets) {
        topicLag[topicOffset.topic] = topicOffset.partitions.map(p => ({
          partition: p.partition,
          lag: BigInt(0), // Simplified
        }));
      }

      return topicLag;
    } finally {
      await this.admin.disconnect();
    }
  }

  async rebalanceConsumerGroup(groupId: string): Promise<void> {
    // Trigger rebalance by deleting group offsets for empty partitions
    console.log(`Triggering rebalance for consumer group: ${groupId}`);
  }
}

// High-performance Kafka consumer
class ScalableKafkaConsumer {
  private kafka: Kafka;
  private consumers: Consumer[] = [];

  constructor(brokers: string[]) {
    this.kafka = new Kafka({ brokers });
  }

  async startConsumerPool(
    groupId: string,
    topics: string[],
    concurrency: number,
    handler: (message: any) => Promise<void>
  ): Promise<void> {
    // Create multiple consumer instances for parallelism
    for (let i = 0; i < concurrency; i++) {
      const consumer = this.kafka.consumer({
        groupId: `${groupId}-${i}`,
        maxInFlightRequests: 10,
        sessionTimeout: 30000,
        heartbeatInterval: 3000,
        maxBytesPerPartition: 1048576, // 1MB
        retry: {
          maxRetryTime: 30000,
          initialRetryTime: 300,
          retries: 10,
        },
      });

      await consumer.connect();
      await consumer.subscribe({ topics, fromBeginning: false });

      await consumer.run({
        partitionsConsumedConcurrently: 4,
        eachMessage: async ({ topic, partition, message }) => {
          if (!message.value) return;

          await handler({
            topic,
            partition,
            key: message.key?.toString(),
            value: JSON.parse(message.value.toString()),
            timestamp: message.timestamp,
            headers: message.headers,
          });
        },
      });

      this.consumers.push(consumer);
    }

    console.log(`Started ${concurrency} consumers for groups ${groupId}-*`);
  }

  async stop(): Promise<void> {
    await Promise.all(this.consumers.map(c => c.disconnect()));
  }
}
```

---

## 9. Auto-scaling Triggers and Policies

```yaml
# vpa-config.yaml
# Vertical Pod Autoscaler

apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: order-service-vpa
  namespace: microservices
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  
  updatePolicy:
    updateMode: Auto  # จะ restart pods เพื่อ apply
  
  resourcePolicy:
    containerPolicies:
      - containerName: order-service
        minAllowed:
          cpu: 100m
          memory: 128Mi
        maxAllowed:
          cpu: 4000m
          memory: 4Gi
        controlledResources: ["cpu", "memory"]
        controlledValues: RequestsAndLimits
---
# Cluster Autoscaler
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-autoscaler-config
  namespace: kube-system
data:
  config.yaml: |
    balance-similar-node-groups: true
    skip-nodes-with-system-pods: false
    skip-nodes-with-local-storage: false
    scale-down-utilization-threshold: 0.5
    scale-down-delay-after-add: 10m
    scale-down-delay-after-delete: 1m
    scale-down-delay-after-failure: 3m
    scale-down-unneeded-time: 10m
    max-graceful-termination-sec: 600
    node-group-auto-discovery: |
      asg:tag=k8s.io/cluster-autoscaler/enabled,
           k8s.io/cluster-autoscaler/production-cluster
```

```typescript
// custom-autoscaler.ts
// Custom autoscaling logic

interface AutoScalerPolicy {
  name: string;
  namespace: string;
  deployment: string;
  metrics: ScalingMetric[];
  minReplicas: number;
  maxReplicas: number;
  cooldownSeconds: number;
}

interface ScalingMetric {
  name: string;
  type: 'up' | 'down' | 'both';
  source: 'prometheus' | 'kafka' | 'custom';
  query: string;
  targetValue: number;
  weight: number;
}

class CustomAutoScaler {
  private lastScaleTime: Map<string, number> = new Map();

  async reconcile(policy: AutoScalerPolicy): Promise<void> {
    const now = Date.now();
    const lastScale = this.lastScaleTime.get(policy.name) || 0;

    if (now - lastScale < policy.cooldownSeconds * 1000) {
      console.log(`Cooldown active for ${policy.name}`);
      return;
    }

    const currentReplicas = await this.getCurrentReplicas(
      policy.deployment,
      policy.namespace
    );

    const desiredReplicas = await this.calculateDesiredReplicas(
      policy,
      currentReplicas
    );

    if (desiredReplicas !== currentReplicas) {
      console.log(
        `Scaling ${policy.deployment}: ${currentReplicas} -> ${desiredReplicas}`
      );
      await this.scale(policy.deployment, policy.namespace, desiredReplicas);
      this.lastScaleTime.set(policy.name, now);
    }
  }

  private async calculateDesiredReplicas(
    policy: AutoScalerPolicy,
    currentReplicas: number
  ): Promise<number> {
    let totalScore = 0;
    let totalWeight = 0;

    for (const metric of policy.metrics) {
      const value = await this.fetchMetric(metric);
      const ratio = value / metric.targetValue;
      totalScore += ratio * metric.weight;
      totalWeight += metric.weight;
    }

    const overallRatio = totalScore / totalWeight;
    
    let desiredReplicas = Math.ceil(currentReplicas * overallRatio);
    desiredReplicas = Math.max(policy.minReplicas, desiredReplicas);
    desiredReplicas = Math.min(policy.maxReplicas, desiredReplicas);

    return desiredReplicas;
  }

  private async fetchMetric(metric: ScalingMetric): Promise<number> {
    if (metric.source === 'prometheus') {
      const response = await fetch(
        `http://prometheus:9090/api/v1/query?query=${encodeURIComponent(metric.query)}`
      );
      const data = await response.json() as {
        data: { result: [{ value: [number, string] }] }
      };
      return parseFloat(data.data.result[0]?.value[1] || '0');
    }
    return 0;
  }

  private async getCurrentReplicas(deployment: string, namespace: string): Promise<number> {
    return 3; // Would use Kubernetes API
  }

  private async scale(deployment: string, namespace: string, replicas: number): Promise<void> {
    // Would use Kubernetes API: kubectl scale
    console.log(`Scaled ${deployment} to ${replicas}`);
  }
}
```

---

## 10. Load Testing and Capacity Modeling

```typescript
// capacity-modeling.ts
// Load testing analysis และ capacity modeling

interface LoadTestResult {
  scenario: string;
  duration: number;
  virtualUsers: number;
  totalRequests: number;
  successRate: number;
  throughput: number; // RPS
  latencies: {
    p50: number;
    p75: number;
    p90: number;
    p95: number;
    p99: number;
    max: number;
  };
  errorRate: number;
  resourceUsage: {
    cpu: number;
    memory: number;
    networkIn: number;
    networkOut: number;
  };
}

class CapacityModel {
  calculateMaxCapacity(
    results: LoadTestResult[],
    sloLatency: number,
    sloErrorRate: number
  ): {
    maxThroughput: number;
    maxVirtualUsers: number;
    bottleneck: string;
    recommendations: string[];
  } {
    // Find last point within SLO
    const withinSLO = results.filter(
      r => r.latencies.p95 <= sloLatency && r.errorRate <= sloErrorRate
    );

    if (withinSLO.length === 0) {
      return {
        maxThroughput: 0,
        maxVirtualUsers: 0,
        bottleneck: 'System cannot meet SLO at any load',
        recommendations: ['Investigate baseline performance'],
      };
    }

    const maxPoint = withinSLO[withinSLO.length - 1];
    const bottleneckPoint = results.find(
      r => r.latencies.p95 > sloLatency || r.errorRate > sloErrorRate
    );

    const bottleneck = this.identifyBottleneck(
      maxPoint,
      bottleneckPoint,
      results
    );

    return {
      maxThroughput: maxPoint.throughput,
      maxVirtualUsers: maxPoint.virtualUsers,
      bottleneck: bottleneck.area,
      recommendations: bottleneck.recommendations,
    };
  }

  private identifyBottleneck(
    lastGoodPoint: LoadTestResult,
    firstBadPoint: LoadTestResult | undefined,
    allResults: LoadTestResult[]
  ): { area: string; recommendations: string[] } {
    if (!firstBadPoint) {
      return {
        area: 'No bottleneck identified',
        recommendations: ['Continue increasing load to find limits'],
      };
    }

    const cpuJump = firstBadPoint.resourceUsage.cpu - lastGoodPoint.resourceUsage.cpu;
    const memJump = firstBadPoint.resourceUsage.memory - lastGoodPoint.resourceUsage.memory;
    const latencyJump = firstBadPoint.latencies.p95 - lastGoodPoint.latencies.p95;

    if (firstBadPoint.errorRate > 0.05 && cpuJump < 10) {
      return {
        area: 'Database/downstream service saturation',
        recommendations: [
          'Add database connection pool capacity',
          'Scale database with read replicas',
          'Implement caching layer',
          'Check downstream service limits',
        ],
      };
    }

    if (firstBadPoint.resourceUsage.cpu > 80) {
      return {
        area: 'CPU saturation',
        recommendations: [
          'Scale horizontally (add replicas)',
          'Profile CPU-intensive code paths',
          'Implement caching to reduce computation',
          'Consider vertical scaling (larger instances)',
        ],
      };
    }

    if (firstBadPoint.resourceUsage.memory > 85) {
      return {
        area: 'Memory pressure',
        recommendations: [
          'Investigate memory leaks',
          'Increase memory limits',
          'Implement memory-efficient data structures',
          'Enable garbage collection optimization',
        ],
      };
    }

    if (latencyJump > 200) {
      return {
        area: 'Latency degradation',
        recommendations: [
          'Check database query performance',
          'Review slow external API calls',
          'Implement circuit breakers',
          'Add caching for hot paths',
        ],
      };
    }

    return {
      area: 'Unknown bottleneck',
      recommendations: ['Perform detailed profiling', 'Check application logs'],
    };
  }

  projectCapacityNeeds(
    currentThroughput: number,
    growthRatePerMonth: number,
    maxCapacity: number,
    safetyMargin: number = 0.7
  ): {
    monthsUntilCapacityNeeded: number;
    projectedThroughputByMonth: Record<number, number>;
    scalingRecommendation: string;
  } {
    const projections: Record<number, number> = {};
    let monthsNeeded = 0;

    for (let month = 1; month <= 24; month++) {
      const projected = currentThroughput * Math.pow(1 + growthRatePerMonth, month);
      projections[month] = Math.round(projected);

      if (projected > maxCapacity * safetyMargin && monthsNeeded === 0) {
        monthsNeeded = month;
      }
    }

    const recommendation = monthsNeeded === 0
      ? 'No scaling needed within 24 months'
      : `Plan scaling capacity by month ${monthsNeeded} (${projections[monthsNeeded]} RPS)`;

    return {
      monthsUntilCapacityNeeded: monthsNeeded,
      projectedThroughputByMonth: projections,
      scalingRecommendation: recommendation,
    };
  }
}
```

---

## สรุปตาราง Scalability Patterns

| Pattern | Scalability Type | Complexity | Use Case |
|---------|-----------------|-----------|---------|
| Horizontal Scaling | สูง | ต่ำ | Stateless services |
| Read Replicas | Read สูง | ปานกลาง | Read-heavy workloads |
| Connection Pooling | Connection สูง | ต่ำ | Database bottleneck |
| CQRS | Read+Write แยก | สูง | Complex query requirements |
| Event Sourcing | Audit + Scale | สูง | Compliance, scalable reads |
| Caching (Multi-level) | สูงมาก | ปานกลาง | Read-heavy, repeated queries |
| Kafka Partitioning | Message สูง | ปานกลาง | Event streaming |
| HPA | Auto scaling | ต่ำ | Varying load |
| KEDA | Event-driven scaling | ปานกลาง | Queue-based workloads |

| Scaling Decision | When to Use | Expected Improvement |
|-----------------|-------------|---------------------|
| Scale out (more pods) | CPU/Memory > 70% | Linear with pods |
| Add read replica | DB read load high | 2-3x read throughput |
| Add cache layer | Same data queried repeatedly | 80-95% cache hit reduction |
| Increase Kafka partitions | Consumer lag growing | Linear with partitions |
| CQRS | Complex query patterns | 5-10x query performance |
| Add PgBouncer | Too many DB connections | 10x connection capacity |

---

## สรุป

Scalability Patterns ที่ครอบคลุม:

1. **Horizontal Scaling** คือ default สำหรับ microservices - เพิ่ม replicas
2. **Database Read Replicas** แยก read/write workload
3. **Connection Pooling** ด้วย PgBouncer ลด connection overhead
4. **CQRS** แยก read/write model สำหรับ complex queries
5. **Event Sourcing** เป็น single source of truth ที่ scalable
6. **Stateless Design** รับประกัน any instance สามารถ handle any request
7. **Multi-level Caching** ลด latency และ database load
8. **Kafka Partition Scaling** เพิ่ม throughput แบบ linear
9. **Auto-scaling** ด้วย HPA และ KEDA ตาม business metrics
10. **Load Testing + Capacity Modeling** วางแผน scaling ล่วงหน้า

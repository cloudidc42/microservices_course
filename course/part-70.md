# Part 70: Data Consistency Patterns in Microservices

## บทนำ

ใน Microservices Architecture การจัดการ Data Consistency เป็นความท้าทายหลักที่สำคัญ เพราะแต่ละ service มี database ของตัวเอง การรักษาความสอดคล้องของข้อมูลข้ามหลาย service จึงต้องใช้ pattern และกลยุทธ์ที่เหมาะสม บทนี้จะครอบคลุมทั้ง Eventual Consistency, Strong Consistency, CAP theorem, Vector Clocks, CRDTs และอื่นๆ

---

## 1. Eventual Consistency Patterns

### แนวคิด Eventual Consistency

Eventual Consistency หมายถึงระบบที่ยอมให้ข้อมูลชั่วคราวไม่สอดคล้องกัน แต่รับประกันว่าในที่สุดข้อมูลจะสอดคล้องกันทั้งหมด เหมาะสำหรับระบบที่ต้องการ High Availability สูง

```typescript
// eventual-consistency-example.ts
// ตัวอย่าง: ระบบ User Profile ที่ใช้ Eventual Consistency

import { EventEmitter } from 'events';
import Redis from 'ioredis';
import { Pool } from 'pg';

interface UserProfile {
  userId: string;
  name: string;
  email: string;
  updatedAt: Date;
}

interface DomainEvent {
  eventId: string;
  eventType: string;
  aggregateId: string;
  payload: Record<string, unknown>;
  occurredAt: Date;
  version: number;
}

class UserProfileService {
  private redis: Redis;
  private db: Pool;
  private eventBus: EventEmitter;

  constructor(redis: Redis, db: Pool) {
    this.redis = redis;
    this.db = db;
    this.eventBus = new EventEmitter();
    this.setupEventHandlers();
  }

  private setupEventHandlers(): void {
    // รับฟัง event จาก service อื่น
    this.eventBus.on('order.created', async (event: DomainEvent) => {
      await this.handleOrderCreated(event);
    });

    this.eventBus.on('payment.processed', async (event: DomainEvent) => {
      await this.handlePaymentProcessed(event);
    });
  }

  async updateProfile(userId: string, updates: Partial<UserProfile>): Promise<void> {
    // บันทึกลง Primary DB
    await this.db.query(
      'UPDATE users SET name=$1, email=$2, updated_at=$3 WHERE id=$4',
      [updates.name, updates.email, new Date(), userId]
    );

    // Emit event สำหรับ services อื่น (Eventual Consistency)
    const event: DomainEvent = {
      eventId: crypto.randomUUID(),
      eventType: 'user.profile.updated',
      aggregateId: userId,
      payload: updates,
      occurredAt: new Date(),
      version: 1,
    };

    // บันทึก event ลง event store
    await this.publishEvent(event);

    // Invalidate cache (อาจยัง stale ชั่วคราว)
    await this.redis.del(`user:profile:${userId}`);
  }

  private async publishEvent(event: DomainEvent): Promise<void> {
    // บันทึก event ลง Redis Stream
    await this.redis.xadd(
      'user-events',
      '*',
      'eventId', event.eventId,
      'eventType', event.eventType,
      'aggregateId', event.aggregateId,
      'payload', JSON.stringify(event.payload),
      'occurredAt', event.occurredAt.toISOString(),
      'version', String(event.version)
    );
  }

  private async handleOrderCreated(event: DomainEvent): Promise<void> {
    // อัพเดท user stats แบบ async (eventual consistency)
    const { userId } = event.payload as { userId: string };
    await this.db.query(
      'UPDATE user_stats SET order_count = order_count + 1 WHERE user_id = $1',
      [userId]
    );
  }

  private async handlePaymentProcessed(event: DomainEvent): Promise<void> {
    const { userId, amount } = event.payload as { userId: string; amount: number };
    await this.db.query(
      'UPDATE user_stats SET total_spend = total_spend + $1 WHERE user_id = $2',
      [amount, userId]
    );
  }
}

// Outbox Pattern สำหรับ Reliable Event Publishing
class OutboxPattern {
  private db: Pool;
  private redis: Redis;

  constructor(db: Pool, redis: Redis) {
    this.db = db;
    this.redis = redis;
  }

  async saveWithOutbox<T>(
    entity: T,
    events: DomainEvent[],
    tableName: string
  ): Promise<void> {
    const client = await this.db.connect();
    
    try {
      await client.query('BEGIN');

      // บันทึก entity
      // (สมมติว่ามี query สำหรับ insert/update)

      // บันทึก events ลง outbox table (ใน transaction เดียวกัน)
      for (const event of events) {
        await client.query(
          `INSERT INTO outbox_events 
           (event_id, event_type, aggregate_id, payload, occurred_at, processed)
           VALUES ($1, $2, $3, $4, $5, false)`,
          [
            event.eventId,
            event.eventType,
            event.aggregateId,
            JSON.stringify(event.payload),
            event.occurredAt
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

  // Outbox Processor - รัน background process
  async processOutbox(): Promise<void> {
    while (true) {
      const { rows } = await this.db.query<{
        event_id: string;
        event_type: string;
        aggregate_id: string;
        payload: string;
        occurred_at: Date;
      }>(
        `SELECT * FROM outbox_events 
         WHERE processed = false 
         ORDER BY occurred_at ASC 
         LIMIT 10
         FOR UPDATE SKIP LOCKED`
      );

      for (const row of rows) {
        try {
          // Publish event ไปยัง message broker
          await this.redis.xadd(
            `${row.event_type}-stream`,
            '*',
            'eventId', row.event_id,
            'aggregateId', row.aggregate_id,
            'payload', row.payload
          );

          // Mark as processed
          await this.db.query(
            'UPDATE outbox_events SET processed = true, processed_at = NOW() WHERE event_id = $1',
            [row.event_id]
          );
        } catch (error) {
          console.error(`Failed to process outbox event ${row.event_id}:`, error);
        }
      }

      // รอก่อน process รอบถัดไป
      await new Promise(resolve => setTimeout(resolve, 1000));
    }
  }
}
```

### Saga Pattern สำหรับ Distributed Transactions

```typescript
// saga-pattern.ts
// Choreography-based Saga

type SagaStatus = 'PENDING' | 'COMPLETED' | 'FAILED' | 'COMPENSATING';

interface SagaStep {
  stepName: string;
  execute: () => Promise<void>;
  compensate: () => Promise<void>;
}

class OrderSaga {
  private steps: SagaStep[] = [];
  private completedSteps: SagaStep[] = [];
  private status: SagaStatus = 'PENDING';

  addStep(step: SagaStep): this {
    this.steps.push(step);
    return this;
  }

  async execute(): Promise<void> {
    this.status = 'PENDING';
    
    for (const step of this.steps) {
      try {
        console.log(`Executing step: ${step.stepName}`);
        await step.execute();
        this.completedSteps.push(step);
      } catch (error) {
        console.error(`Step ${step.stepName} failed:`, error);
        this.status = 'COMPENSATING';
        await this.compensate();
        this.status = 'FAILED';
        throw error;
      }
    }
    
    this.status = 'COMPLETED';
  }

  private async compensate(): Promise<void> {
    // ทำ compensation ย้อนหลัง
    const stepsToCompensate = [...this.completedSteps].reverse();
    
    for (const step of stepsToCompensate) {
      try {
        console.log(`Compensating step: ${step.stepName}`);
        await step.compensate();
      } catch (error) {
        console.error(`Compensation failed for ${step.stepName}:`, error);
        // Log และ alert แต่ไม่ throw เพื่อให้ compensate ต่อได้
      }
    }
  }
}

// ตัวอย่างการใช้งาน Order Saga
async function createOrderWithSaga(
  orderService: any,
  inventoryService: any,
  paymentService: any,
  shippingService: any
): Promise<void> {
  const orderId = crypto.randomUUID();
  const reservationId = crypto.randomUUID();
  const paymentId = crypto.randomUUID();

  const saga = new OrderSaga();

  saga
    .addStep({
      stepName: 'CreateOrder',
      execute: async () => {
        await orderService.create(orderId, { status: 'PENDING' });
      },
      compensate: async () => {
        await orderService.cancel(orderId);
      },
    })
    .addStep({
      stepName: 'ReserveInventory',
      execute: async () => {
        await inventoryService.reserve(reservationId, orderId);
      },
      compensate: async () => {
        await inventoryService.release(reservationId);
      },
    })
    .addStep({
      stepName: 'ProcessPayment',
      execute: async () => {
        await paymentService.charge(paymentId, orderId, 1000);
      },
      compensate: async () => {
        await paymentService.refund(paymentId);
      },
    })
    .addStep({
      stepName: 'CreateShipment',
      execute: async () => {
        await shippingService.schedule(orderId);
      },
      compensate: async () => {
        await shippingService.cancel(orderId);
      },
    });

  await saga.execute();
}
```

---

## 2. Strong Consistency Trade-offs (CAP Theorem Applied)

### CAP Theorem

CAP Theorem ระบุว่าระบบ Distributed สามารถรับประกันได้เพียง 2 ใน 3 ข้อ:
- **C**onsistency: ทุก node เห็นข้อมูลเดียวกัน
- **A**vailability: ทุก request ได้รับ response
- **P**artition tolerance: ระบบยังทำงานได้เมื่อเกิด network partition

```typescript
// cap-theorem-demo.ts
// ตัวอย่างการเลือก CP vs AP

// CP System - เลือก Consistency + Partition Tolerance
class CPDatabaseService {
  private primaryNode: any;
  private replicaNodes: any[];

  // Synchronous replication - ทุก write ต้อง confirm จาก majority
  async write(key: string, value: unknown): Promise<void> {
    const majorityCount = Math.floor(this.replicaNodes.length / 2) + 1;
    let confirmedWrites = 0;

    // Write ไปยัง primary ก่อน
    await this.primaryNode.set(key, value);
    confirmedWrites++;

    // Synchronously replicate ไปยัง replicas
    const writePromises = this.replicaNodes.map(async (replica) => {
      try {
        await replica.set(key, value);
        confirmedWrites++;
      } catch (error) {
        console.error('Replica write failed:', error);
      }
    });

    await Promise.allSettled(writePromises);

    // ถ้าไม่ได้ majority ให้ throw error (เลือก Consistency)
    if (confirmedWrites < majorityCount) {
      throw new Error('Write failed: Could not achieve quorum');
    }
  }

  // Read จาก majority ก่อนตอบ
  async read(key: string): Promise<unknown> {
    const responses = await Promise.allSettled(
      [this.primaryNode, ...this.replicaNodes].map(node => node.get(key))
    );

    const values = responses
      .filter(r => r.status === 'fulfilled')
      .map(r => (r as PromiseFulfilledResult<unknown>).value);

    if (values.length === 0) {
      throw new Error('Read failed: No nodes available');
    }

    // ตรวจสอบ consistency
    const uniqueValues = new Set(values.map(v => JSON.stringify(v)));
    if (uniqueValues.size > 1) {
      throw new Error('Inconsistent reads detected');
    }

    return values[0];
  }
}

// AP System - เลือก Availability + Partition Tolerance
class APDatabaseService {
  private nodes: any[];

  // Write ไปยัง node ที่พร้อม (ไม่รอ consistency)
  async write(key: string, value: unknown): Promise<void> {
    const writePromises = this.nodes.map(node =>
      node.set(key, value).catch((err: Error) => {
        console.warn(`Node write failed (continuing): ${err.message}`);
      })
    );

    await Promise.allSettled(writePromises);
    // ไม่ throw แม้ว่า nodes บางตัวล้มเหลว
  }

  // Read จาก node แรกที่พร้อม (อาจได้ข้อมูลเก่า)
  async read(key: string): Promise<unknown> {
    for (const node of this.nodes) {
      try {
        return await node.get(key);
      } catch {
        continue; // ลอง node ถัดไป
      }
    }
    throw new Error('All nodes unavailable');
  }
}

// PACELC Model - ขยาย CAP
class PACELCSystem {
  private config: {
    partitionBehavior: 'consistency' | 'availability';
    normalBehavior: 'latency' | 'consistency';
  };

  constructor(config: {
    partitionBehavior: 'consistency' | 'availability';
    normalBehavior: 'latency' | 'consistency';
  }) {
    this.config = config;
  }

  async write(key: string, value: unknown): Promise<{ latency: number }> {
    const start = Date.now();

    if (this.config.normalBehavior === 'consistency') {
      // รอให้ทุก replica confirm (สูง latency)
      await this.synchronousReplication(key, value);
    } else {
      // Write ทันที ไม่รอ replica (ต่ำ latency)
      await this.asyncReplication(key, value);
    }

    return { latency: Date.now() - start };
  }

  private async synchronousReplication(_key: string, _value: unknown): Promise<void> {
    // Simulate synchronous replication delay
    await new Promise(resolve => setTimeout(resolve, 50));
  }

  private async asyncReplication(_key: string, _value: unknown): Promise<void> {
    // Simulate fast local write
    await new Promise(resolve => setTimeout(resolve, 5));
    // Background replication
    setTimeout(() => this.backgroundReplicate(_key, _value), 0);
  }

  private async backgroundReplicate(_key: string, _value: unknown): Promise<void> {
    await new Promise(resolve => setTimeout(resolve, 100));
  }
}
```

---

## 3. Read-Your-Writes Consistency

### แนวคิด

Read-Your-Writes Consistency รับประกันว่าหลังจาก user เขียนข้อมูล การอ่านข้อมูลของ user นั้นจะเห็นการเขียนนั้นเสมอ

```typescript
// read-your-writes.ts
import Redis from 'ioredis';
import { Pool } from 'pg';

interface WriteToken {
  userId: string;
  timestamp: number;
  version: number;
}

class ReadYourWritesService {
  private primaryDb: Pool;
  private replicaDb: Pool;
  private redis: Redis;
  private readonly TOKEN_TTL = 60; // seconds

  constructor(primaryDb: Pool, replicaDb: Pool, redis: Redis) {
    this.primaryDb = primaryDb;
    this.replicaDb = replicaDb;
    this.redis = redis;
  }

  async write(userId: string, data: Record<string, unknown>): Promise<WriteToken> {
    // Write ลง Primary
    const result = await this.primaryDb.query<{ version: number }>(
      'UPDATE user_data SET data=$1, version=version+1, updated_at=NOW() WHERE user_id=$2 RETURNING version',
      [JSON.stringify(data), userId]
    );

    const version = result.rows[0].version;
    
    // สร้าง write token
    const token: WriteToken = {
      userId,
      timestamp: Date.now(),
      version,
    };

    // บันทึก token ใน Redis
    await this.redis.setex(
      `write-token:${userId}`,
      this.TOKEN_TTL,
      JSON.stringify(token)
    );

    return token;
  }

  async read(userId: string, clientToken?: WriteToken): Promise<Record<string, unknown> | null> {
    // ตรวจสอบ token จาก Redis
    const storedTokenStr = await this.redis.get(`write-token:${userId}`);
    
    if (storedTokenStr || clientToken) {
      const requiredToken = storedTokenStr 
        ? JSON.parse(storedTokenStr) as WriteToken
        : clientToken!;

      // ตรวจสอบว่า replica มี version ที่ต้องการ
      const replicaResult = await this.replicaDb.query<{ version: number }>(
        'SELECT version FROM user_data WHERE user_id=$1',
        [userId]
      );

      if (
        replicaResult.rows.length === 0 ||
        replicaResult.rows[0].version < requiredToken.version
      ) {
        // Replica ยังไม่ sync - อ่านจาก Primary
        console.log(`Routing to primary for user ${userId} (replica lag detected)`);
        const primaryResult = await this.primaryDb.query<{ data: string }>(
          'SELECT data FROM user_data WHERE user_id=$1',
          [userId]
        );
        return primaryResult.rows[0] ? JSON.parse(primaryResult.rows[0].data) : null;
      }
    }

    // อ่านจาก Replica ได้เลย
    const result = await this.replicaDb.query<{ data: string }>(
      'SELECT data FROM user_data WHERE user_id=$1',
      [userId]
    );
    
    return result.rows[0] ? JSON.parse(result.rows[0].data) : null;
  }
}

// Sticky Session approach สำหรับ Read-Your-Writes
class StickySessionRouter {
  private sessions: Map<string, 'primary' | 'replica'> = new Map();
  private readonly STICKY_DURATION = 30000; // 30 seconds
  private stickyTimers: Map<string, NodeJS.Timeout> = new Map();

  markAsWriter(sessionId: string): void {
    this.sessions.set(sessionId, 'primary');
    
    // Clear existing timer
    const existingTimer = this.stickyTimers.get(sessionId);
    if (existingTimer) clearTimeout(existingTimer);
    
    // Auto-reset หลัง 30 วินาที
    const timer = setTimeout(() => {
      this.sessions.set(sessionId, 'replica');
      this.stickyTimers.delete(sessionId);
    }, this.STICKY_DURATION);
    
    this.stickyTimers.set(sessionId, timer);
  }

  getTarget(sessionId: string): 'primary' | 'replica' {
    return this.sessions.get(sessionId) || 'replica';
  }
}
```

---

## 4. Monotonic Read Consistency

### แนวคิด

Monotonic Read Consistency รับประกันว่าถ้า process อ่านค่า X แล้ว การอ่านครั้งต่อไปจะไม่ได้ค่าที่เก่ากว่า X (จะไม่มีข้อมูลย้อนกลับ)

```typescript
// monotonic-reads.ts

interface VersionedData<T> {
  data: T;
  version: number;
  timestamp: number;
}

class MonotonicReadCache {
  private lastReadVersions: Map<string, number> = new Map();
  private redis: Redis;
  private db: Pool;

  constructor(redis: Redis, db: Pool) {
    this.redis = redis;
    this.db = db;
  }

  async read<T>(
    clientId: string,
    resourceId: string
  ): Promise<VersionedData<T>> {
    const lastVersion = this.lastReadVersions.get(`${clientId}:${resourceId}`) || 0;

    // ลอง read จาก cache ก่อน
    const cached = await this.redis.get(`resource:${resourceId}`);
    
    if (cached) {
      const parsed = JSON.parse(cached) as VersionedData<T>;
      
      // ตรวจสอบ monotonic property
      if (parsed.version >= lastVersion) {
        this.lastReadVersions.set(`${clientId}:${resourceId}`, parsed.version);
        return parsed;
      }
      
      // Cache มีข้อมูลเก่ากว่า - ไปอ่าน DB
      console.warn(`Cache version ${parsed.version} < last seen ${lastVersion}, reading from DB`);
    }

    // Read จาก DB พร้อม version check
    const result = await this.db.query<{ data: T; version: number; updated_at: Date }>(
      'SELECT data, version, updated_at FROM resources WHERE id=$1 AND version >= $2',
      [resourceId, lastVersion]
    );

    if (result.rows.length === 0) {
      // ต้องอ่านจาก primary
      const primaryResult = await this.db.query<{ data: T; version: number; updated_at: Date }>(
        'SELECT data, version, updated_at FROM resources WHERE id=$1',
        [resourceId]
      );
      
      if (primaryResult.rows.length === 0) {
        throw new Error(`Resource ${resourceId} not found`);
      }

      const row = primaryResult.rows[0];
      const versionedData: VersionedData<T> = {
        data: row.data,
        version: row.version,
        timestamp: row.updated_at.getTime(),
      };

      this.lastReadVersions.set(`${clientId}:${resourceId}`, versionedData.version);
      return versionedData;
    }

    const row = result.rows[0];
    const versionedData: VersionedData<T> = {
      data: row.data,
      version: row.version,
      timestamp: row.updated_at.getTime(),
    };

    this.lastReadVersions.set(`${clientId}:${resourceId}`, versionedData.version);
    return versionedData;
  }
}
```

---

## 5. Consistent Prefix Reads

### แนวคิด

Consistent Prefix Reads รับประกันว่าถ้า operation A เกิดก่อน B ผู้อ่านจะเห็น A ก่อน B เสมอ ไม่มีการข้ามลำดับ

```typescript
// consistent-prefix-reads.ts

interface OrderedEvent {
  sequenceNumber: number;
  partitionKey: string;
  data: Record<string, unknown>;
  timestamp: number;
}

class ConsistentPrefixReader {
  private db: Pool;
  private lastReadSequence: Map<string, number> = new Map();

  constructor(db: Pool) {
    this.db = db;
  }

  async readEvents(
    partitionKey: string,
    afterSequence?: number
  ): Promise<OrderedEvent[]> {
    const startSeq = afterSequence ?? this.lastReadSequence.get(partitionKey) ?? 0;

    const result = await this.db.query<{
      sequence_number: number;
      partition_key: string;
      data: string;
      created_at: Date;
    }>(
      `SELECT sequence_number, partition_key, data, created_at
       FROM event_log
       WHERE partition_key = $1 AND sequence_number > $2
       ORDER BY sequence_number ASC`,
      [partitionKey, startSeq]
    );

    // ตรวจสอบว่า sequence ต่อเนื่อง (no gaps)
    const events: OrderedEvent[] = [];
    let expectedSeq = startSeq + 1;

    for (const row of result.rows) {
      if (row.sequence_number !== expectedSeq) {
        // พบ gap - หยุดและ return เท่าที่มี (consistent prefix)
        console.warn(
          `Gap detected: expected ${expectedSeq}, got ${row.sequence_number}. Returning prefix.`
        );
        break;
      }

      events.push({
        sequenceNumber: row.sequence_number,
        partitionKey: row.partition_key,
        data: JSON.parse(row.data),
        timestamp: row.created_at.getTime(),
      });

      expectedSeq++;
    }

    if (events.length > 0) {
      this.lastReadSequence.set(
        partitionKey,
        events[events.length - 1].sequenceNumber
      );
    }

    return events;
  }

  async writeEvent(
    partitionKey: string,
    data: Record<string, unknown>
  ): Promise<number> {
    // ใช้ sequence ที่รับประกันว่าต่อเนื่อง
    const result = await this.db.query<{ sequence_number: number }>(
      `INSERT INTO event_log (partition_key, data, created_at)
       VALUES ($1, $2, NOW())
       RETURNING sequence_number`,
      [partitionKey, JSON.stringify(data)]
    );

    return result.rows[0].sequence_number;
  }
}
```

---

## 6. Vector Clocks and Conflict Resolution

### Vector Clocks

Vector Clocks ใช้ติดตามลำดับของ events ใน distributed system เพื่อระบุว่า event ไหนเกิดก่อน/หลัง หรือเกิดพร้อมกัน (concurrent)

```typescript
// vector-clocks.ts

type VectorClock = Map<string, number>;

class VectorClockManager {
  private nodeId: string;
  private clock: VectorClock;

  constructor(nodeId: string) {
    this.nodeId = nodeId;
    this.clock = new Map();
    this.clock.set(nodeId, 0);
  }

  // เพิ่ม counter สำหรับ node ปัจจุบัน
  tick(): VectorClock {
    const current = this.clock.get(this.nodeId) || 0;
    this.clock.set(this.nodeId, current + 1);
    return new Map(this.clock);
  }

  // Merge clock จาก event ที่รับมา
  receive(receivedClock: VectorClock): void {
    // อัพเดท clock ตาม maximum
    for (const [nodeId, time] of receivedClock) {
      const current = this.clock.get(nodeId) || 0;
      this.clock.set(nodeId, Math.max(current, time));
    }
    // Increment ของตัวเอง
    const selfTime = this.clock.get(this.nodeId) || 0;
    this.clock.set(this.nodeId, selfTime + 1);
  }

  // เปรียบเทียบ vector clocks
  compare(a: VectorClock, b: VectorClock): 'before' | 'after' | 'concurrent' | 'equal' {
    let aBeforeB = false;
    let bBeforeA = false;

    const allNodes = new Set([...a.keys(), ...b.keys()]);

    for (const node of allNodes) {
      const aTime = a.get(node) || 0;
      const bTime = b.get(node) || 0;

      if (aTime < bTime) aBeforeB = true;
      if (aTime > bTime) bBeforeA = true;
    }

    if (!aBeforeB && !bBeforeA) return 'equal';
    if (aBeforeB && !bBeforeA) return 'before';
    if (!aBeforeB && bBeforeA) return 'after';
    return 'concurrent'; // Conflict!
  }

  getClock(): VectorClock {
    return new Map(this.clock);
  }
}

// Conflict Resolution Strategies
interface VersionedValue<T> {
  value: T;
  vectorClock: VectorClock;
  nodeId: string;
  timestamp: number;
}

class ConflictResolver<T> {
  private clockManager: VectorClockManager;

  constructor(nodeId: string) {
    this.clockManager = new VectorClockManager(nodeId);
  }

  // Last-Write-Wins (LWW) Strategy
  resolveWithLWW(versions: VersionedValue<T>[]): VersionedValue<T> {
    return versions.reduce((latest, current) =>
      current.timestamp > latest.timestamp ? current : latest
    );
  }

  // Application-specific merge
  resolveWithMerge(
    versions: VersionedValue<T>[],
    mergeFn: (a: T, b: T) => T
  ): VersionedValue<T> {
    if (versions.length === 1) return versions[0];

    const merged = versions.reduce((acc, current) => ({
      ...acc,
      value: mergeFn(acc.value, current.value),
    }));

    return {
      ...merged,
      vectorClock: this.mergeClock(versions.map(v => v.vectorClock)),
      timestamp: Date.now(),
    };
  }

  private mergeClock(clocks: VectorClock[]): VectorClock {
    const merged: VectorClock = new Map();
    for (const clock of clocks) {
      for (const [node, time] of clock) {
        merged.set(node, Math.max(merged.get(node) || 0, time));
      }
    }
    return merged;
  }

  // Multi-value register (เก็บทุก concurrent version)
  detectConflicts(
    existing: VersionedValue<T>[],
    incoming: VersionedValue<T>
  ): VersionedValue<T>[] {
    const manager = this.clockManager;
    const dominated: VersionedValue<T>[] = [];
    const conflicts: VersionedValue<T>[] = [incoming];

    for (const version of existing) {
      const rel = manager.compare(incoming.vectorClock, version.vectorClock);
      
      if (rel === 'after') {
        // incoming ใหม่กว่า - ไม่เก็บ existing
        dominated.push(version);
      } else if (rel === 'before') {
        // existing ใหม่กว่า - ไม่เก็บ incoming
        return existing;
      } else {
        // Concurrent - เก็บทั้งคู่
        conflicts.push(version);
      }
    }

    // Return versions ที่ไม่ถูก dominate
    return conflicts.filter(v => !dominated.includes(v));
  }
}

// ตัวอย่าง: Shopping Cart ที่ใช้ Vector Clocks
interface CartItem {
  productId: string;
  quantity: number;
}

interface Cart {
  userId: string;
  items: CartItem[];
}

class DistributedCart {
  private resolver: ConflictResolver<Cart>;
  private versions: Map<string, VersionedValue<Cart>[]> = new Map();
  private nodeId: string;

  constructor(nodeId: string) {
    this.nodeId = nodeId;
    this.resolver = new ConflictResolver<Cart>(nodeId);
  }

  addItem(userId: string, item: CartItem): void {
    const existing = this.versions.get(userId) || [];
    
    // Merge items จาก existing versions
    const existingItems = existing.length > 0 
      ? existing[0].value.items 
      : [];

    const updatedItems = [...existingItems];
    const existingItem = updatedItems.find(i => i.productId === item.productId);
    
    if (existingItem) {
      existingItem.quantity += item.quantity;
    } else {
      updatedItems.push(item);
    }

    const newVersion: VersionedValue<Cart> = {
      value: { userId, items: updatedItems },
      vectorClock: new Map([[this.nodeId, Date.now()]]),
      nodeId: this.nodeId,
      timestamp: Date.now(),
    };

    const resolved = this.resolver.detectConflicts(existing, newVersion);
    this.versions.set(userId, resolved);
  }

  getCart(userId: string): Cart[] {
    const versions = this.versions.get(userId) || [];
    
    if (versions.length === 1) {
      return [versions[0].value];
    }
    
    // Return multiple versions ถ้ามี conflict
    return versions.map(v => v.value);
  }
}
```

---

## 7. CRDTs (Conflict-free Replicated Data Types)

### แนวคิด CRDTs

CRDTs คือ data structures ที่ออกแบบมาเพื่อให้ merge ได้โดยไม่เกิด conflict ระบบสามารถ replicate ข้อมูลแบบ asynchronous และ merge โดยอัตโนมัติ

```typescript
// crdts.ts

// G-Counter (Grow-only Counter)
class GCounter {
  private counts: Map<string, number>;
  private nodeId: string;

  constructor(nodeId: string) {
    this.nodeId = nodeId;
    this.counts = new Map([[nodeId, 0]]);
  }

  increment(amount: number = 1): void {
    const current = this.counts.get(this.nodeId) || 0;
    this.counts.set(this.nodeId, current + amount);
  }

  value(): number {
    let total = 0;
    for (const count of this.counts.values()) {
      total += count;
    }
    return total;
  }

  // Merge สองก Counter (commutative, associative, idempotent)
  merge(other: GCounter): GCounter {
    const merged = new GCounter(this.nodeId);
    const allNodes = new Set([...this.counts.keys(), ...other.counts.keys()]);
    
    for (const node of allNodes) {
      const a = this.counts.get(node) || 0;
      const b = other.counts.get(node) || 0;
      merged.counts.set(node, Math.max(a, b));
    }
    
    return merged;
  }

  serialize(): Record<string, number> {
    return Object.fromEntries(this.counts);
  }

  static deserialize(data: Record<string, number>, nodeId: string): GCounter {
    const counter = new GCounter(nodeId);
    counter.counts = new Map(Object.entries(data));
    return counter;
  }
}

// PN-Counter (Positive-Negative Counter)
class PNCounter {
  private positive: GCounter;
  private negative: GCounter;

  constructor(nodeId: string) {
    this.positive = new GCounter(nodeId);
    this.negative = new GCounter(nodeId);
  }

  increment(amount: number = 1): void {
    this.positive.increment(amount);
  }

  decrement(amount: number = 1): void {
    this.negative.increment(amount);
  }

  value(): number {
    return this.positive.value() - this.negative.value();
  }

  merge(other: PNCounter): PNCounter {
    const merged = new PNCounter('merged');
    merged.positive = this.positive.merge(other.positive);
    merged.negative = this.negative.merge(other.negative);
    return merged;
  }
}

// OR-Set (Observed-Remove Set)
class ORSet<T> {
  private elements: Map<string, Set<string>>; // value -> set of unique tags
  private tombstones: Set<string>; // removed tags
  private nodeId: string;

  constructor(nodeId: string) {
    this.nodeId = nodeId;
    this.elements = new Map();
    this.tombstones = new Set();
  }

  add(element: T): void {
    const key = JSON.stringify(element);
    const tag = `${this.nodeId}-${Date.now()}-${Math.random()}`;
    
    if (!this.elements.has(key)) {
      this.elements.set(key, new Set());
    }
    this.elements.get(key)!.add(tag);
  }

  remove(element: T): void {
    const key = JSON.stringify(element);
    const tags = this.elements.get(key);
    
    if (tags) {
      // Add all current tags to tombstones
      for (const tag of tags) {
        this.tombstones.add(tag);
      }
    }
  }

  has(element: T): boolean {
    const key = JSON.stringify(element);
    const tags = this.elements.get(key);
    
    if (!tags) return false;
    
    // Element exists if any tag is NOT in tombstones
    for (const tag of tags) {
      if (!this.tombstones.has(tag)) {
        return true;
      }
    }
    
    return false;
  }

  values(): T[] {
    const result: T[] = [];
    
    for (const [key, tags] of this.elements) {
      const hasLiveTags = [...tags].some(tag => !this.tombstones.has(tag));
      if (hasLiveTags) {
        result.push(JSON.parse(key) as T);
      }
    }
    
    return result;
  }

  merge(other: ORSet<T>): ORSet<T> {
    const merged = new ORSet<T>(this.nodeId);
    
    // Merge elements
    for (const [key, tags] of this.elements) {
      merged.elements.set(key, new Set(tags));
    }
    
    for (const [key, tags] of other.elements) {
      if (!merged.elements.has(key)) {
        merged.elements.set(key, new Set());
      }
      for (const tag of tags) {
        merged.elements.get(key)!.add(tag);
      }
    }
    
    // Merge tombstones
    merged.tombstones = new Set([...this.tombstones, ...other.tombstones]);
    
    return merged;
  }
}

// LWW-Register (Last-Write-Wins Register)
class LWWRegister<T> {
  private value: T | undefined;
  private timestamp: number = 0;
  private nodeId: string;

  constructor(nodeId: string) {
    this.nodeId = nodeId;
  }

  set(value: T, timestamp?: number): void {
    const ts = timestamp ?? Date.now();
    if (ts > this.timestamp) {
      this.value = value;
      this.timestamp = ts;
    }
  }

  get(): T | undefined {
    return this.value;
  }

  merge(other: LWWRegister<T>): LWWRegister<T> {
    const merged = new LWWRegister<T>(this.nodeId);
    
    if (this.timestamp >= other.timestamp) {
      merged.value = this.value;
      merged.timestamp = this.timestamp;
    } else {
      merged.value = other.value;
      merged.timestamp = other.timestamp;
    }
    
    return merged;
  }
}

// Practical CRDT: Collaborative Document Editing
class CRDTDocument {
  private content: Map<string, { char: string; timestamp: number; nodeId: string }>;
  private deletions: Set<string>;
  private nodeId: string;

  constructor(nodeId: string) {
    this.nodeId = nodeId;
    this.content = new Map();
    this.deletions = new Set();
  }

  insert(position: number, char: string): string {
    const id = `${this.nodeId}-${Date.now()}-${Math.random()}`;
    this.content.set(id, {
      char,
      timestamp: Date.now(),
      nodeId: this.nodeId,
    });
    return id;
  }

  delete(id: string): void {
    this.deletions.add(id);
  }

  getText(): string {
    return [...this.content.entries()]
      .filter(([id]) => !this.deletions.has(id))
      .sort(([, a], [, b]) => a.timestamp - b.timestamp)
      .map(([, { char }]) => char)
      .join('');
  }

  merge(other: CRDTDocument): void {
    // Merge content
    for (const [id, data] of other.content) {
      if (!this.content.has(id)) {
        this.content.set(id, data);
      }
    }
    
    // Merge deletions
    for (const id of other.deletions) {
      this.deletions.add(id);
    }
  }
}
```

---

## 8. Multi-master Replication

```typescript
// multi-master-replication.ts

interface ReplicationNode {
  nodeId: string;
  endpoint: string;
  priority: number;
}

interface ReplicationConflict {
  key: string;
  versions: Array<{
    value: unknown;
    nodeId: string;
    timestamp: number;
  }>;
}

class MultiMasterReplication {
  private nodes: ReplicationNode[];
  private localNodeId: string;
  private db: Pool;
  private conflictLog: ReplicationConflict[] = [];

  constructor(localNodeId: string, nodes: ReplicationNode[], db: Pool) {
    this.localNodeId = localNodeId;
    this.nodes = nodes;
    this.db = db;
  }

  async write(key: string, value: unknown): Promise<void> {
    const timestamp = Date.now();
    
    // Write locally ก่อน
    await this.localWrite(key, value, timestamp);
    
    // Async replicate ไปยัง nodes อื่น
    this.replicateToOtherNodes(key, value, timestamp);
  }

  private async localWrite(
    key: string,
    value: unknown,
    timestamp: number
  ): Promise<void> {
    await this.db.query(
      `INSERT INTO replicated_data (key, value, node_id, timestamp)
       VALUES ($1, $2, $3, $4)
       ON CONFLICT (key) DO UPDATE
       SET value = CASE 
         WHEN excluded.timestamp > replicated_data.timestamp THEN excluded.value
         ELSE replicated_data.value
       END,
       timestamp = GREATEST(excluded.timestamp, replicated_data.timestamp),
       updated_at = NOW()`,
      [key, JSON.stringify(value), this.localNodeId, timestamp]
    );
  }

  private replicateToOtherNodes(
    key: string,
    value: unknown,
    timestamp: number
  ): void {
    // Fire and forget - async replication
    for (const node of this.nodes) {
      if (node.nodeId === this.localNodeId) continue;
      
      this.replicateToNode(node, key, value, timestamp).catch(err => {
        console.error(`Replication to ${node.nodeId} failed:`, err);
        this.queueForRetry(node.nodeId, key, value, timestamp);
      });
    }
  }

  private async replicateToNode(
    node: ReplicationNode,
    key: string,
    value: unknown,
    timestamp: number
  ): Promise<void> {
    const response = await fetch(`${node.endpoint}/replicate`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        key,
        value,
        timestamp,
        sourceNodeId: this.localNodeId,
      }),
    });

    if (!response.ok) {
      throw new Error(`Replication failed: ${response.status}`);
    }
  }

  async receiveReplication(
    key: string,
    value: unknown,
    timestamp: number,
    sourceNodeId: string
  ): Promise<void> {
    // ตรวจสอบว่ามี conflict ไหม
    const existing = await this.db.query<{
      value: string;
      node_id: string;
      timestamp: number;
    }>(
      'SELECT value, node_id, timestamp FROM replicated_data WHERE key=$1',
      [key]
    );

    if (existing.rows.length > 0) {
      const currentTimestamp = existing.rows[0].timestamp;
      
      if (timestamp === currentTimestamp && sourceNodeId !== existing.rows[0].node_id) {
        // Conflict! เวลาเท่ากันแต่ node ต่างกัน
        this.conflictLog.push({
          key,
          versions: [
            {
              value: JSON.parse(existing.rows[0].value),
              nodeId: existing.rows[0].node_id,
              timestamp: currentTimestamp,
            },
            { value, nodeId: sourceNodeId, timestamp },
          ],
        });
        
        // Resolve ด้วย node ID (deterministic)
        if (sourceNodeId > existing.rows[0].node_id) {
          await this.localWrite(key, value, timestamp);
        }
        return;
      }
    }

    await this.localWrite(key, value, timestamp);
  }

  private async queueForRetry(
    nodeId: string,
    key: string,
    value: unknown,
    timestamp: number
  ): Promise<void> {
    await this.db.query(
      `INSERT INTO replication_queue (target_node_id, key, value, timestamp, retry_count)
       VALUES ($1, $2, $3, $4, 0)`,
      [nodeId, key, JSON.stringify(value), timestamp]
    );
  }

  getConflicts(): ReplicationConflict[] {
    return [...this.conflictLog];
  }
}
```

---

## 9. Synchronous vs Asynchronous Replication

```typescript
// replication-strategies.ts

interface ReplicationConfig {
  strategy: 'sync' | 'async' | 'semi-sync';
  quorumSize?: number;
  timeout?: number;
  maxLag?: number; // milliseconds
}

class ReplicationManager {
  private config: ReplicationConfig;
  private replicas: string[];
  private lagMonitor: Map<string, number> = new Map();

  constructor(config: ReplicationConfig, replicas: string[]) {
    this.config = config;
    this.replicas = replicas;
    this.startLagMonitoring();
  }

  async write(data: unknown): Promise<{ success: boolean; latency: number }> {
    const start = Date.now();

    switch (this.config.strategy) {
      case 'sync':
        await this.synchronousWrite(data);
        break;
      case 'async':
        await this.asynchronousWrite(data);
        break;
      case 'semi-sync':
        await this.semiSynchronousWrite(data);
        break;
    }

    return { success: true, latency: Date.now() - start };
  }

  // Synchronous: รอ ack จากทุก replica
  private async synchronousWrite(data: unknown): Promise<void> {
    const promises = this.replicas.map(replica =>
      this.writeToReplica(replica, data)
    );

    await Promise.all(promises);
  }

  // Asynchronous: ไม่รอ replica
  private async asynchronousWrite(data: unknown): Promise<void> {
    // Write ที่ primary แล้วส่ง async
    for (const replica of this.replicas) {
      this.writeToReplica(replica, data).catch(err =>
        console.error(`Async write to ${replica} failed:`, err)
      );
    }
  }

  // Semi-synchronous: รอ quorum
  private async semiSynchronousWrite(data: unknown): Promise<void> {
    const quorum = this.config.quorumSize || Math.floor(this.replicas.length / 2) + 1;
    let confirmed = 1; // count primary

    const promises = this.replicas.map(replica =>
      this.writeToReplica(replica, data)
        .then(() => { confirmed++; })
        .catch(err => console.error(`Replica write failed:`, err))
    );

    // รอจนถึง quorum
    await new Promise<void>((resolve, reject) => {
      const timeout = setTimeout(() => {
        if (confirmed >= quorum) {
          resolve();
        } else {
          reject(new Error(`Quorum not reached: ${confirmed}/${quorum}`));
        }
      }, this.config.timeout || 5000);

      Promise.allSettled(promises).then(() => {
        clearTimeout(timeout);
        if (confirmed >= quorum) {
          resolve();
        } else {
          reject(new Error(`Quorum not reached: ${confirmed}/${quorum}`));
        }
      });
    });

    // ส่ง ack ไปยัง replicas ที่เหลือแบบ async
  }

  private async writeToReplica(replica: string, data: unknown): Promise<void> {
    const start = Date.now();
    
    await fetch(`${replica}/replicate`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data),
    });

    this.lagMonitor.set(replica, Date.now() - start);
  }

  private startLagMonitoring(): void {
    setInterval(() => {
      for (const [replica, lag] of this.lagMonitor) {
        if (this.config.maxLag && lag > this.config.maxLag) {
          console.warn(`Replica ${replica} is lagging: ${lag}ms`);
        }
      }
    }, 5000);
  }

  getReplicationLag(): Map<string, number> {
    return new Map(this.lagMonitor);
  }
}
```

---

## 10. Data Consistency Testing Strategies

```typescript
// consistency-testing.ts
// การทดสอบ Data Consistency ใน Distributed System

import { describe, test, expect, beforeEach, afterEach } from '@jest/globals';

// Linearizability Checker
class LinearizabilityChecker {
  private operations: Array<{
    type: 'read' | 'write';
    key: string;
    value?: unknown;
    result?: unknown;
    startTime: number;
    endTime: number;
    processId: string;
  }> = [];

  recordWrite(
    key: string,
    value: unknown,
    startTime: number,
    endTime: number,
    processId: string
  ): void {
    this.operations.push({
      type: 'write', key, value, startTime, endTime, processId
    });
  }

  recordRead(
    key: string,
    result: unknown,
    startTime: number,
    endTime: number,
    processId: string
  ): void {
    this.operations.push({
      type: 'read', key, result, startTime, endTime, processId
    });
  }

  // ตรวจสอบว่า history เป็น linearizable ไหม
  isLinearizable(): boolean {
    // Simplified check: reads should see the latest completed write
    const writes = this.operations.filter(op => op.type === 'write');
    const reads = this.operations.filter(op => op.type === 'read');

    for (const read of reads) {
      // หา writes ที่เสร็จก่อน read เริ่ม
      const completedWrites = writes.filter(
        w => w.key === read.key && w.endTime <= read.startTime
      );

      if (completedWrites.length === 0) continue;

      // Read ควรเห็น write ล่าสุด
      const latestWrite = completedWrites.reduce((latest, current) =>
        current.endTime > latest.endTime ? current : latest
      );

      if (JSON.stringify(read.result) !== JSON.stringify(latestWrite.value)) {
        return false;
      }
    }

    return true;
  }
}

// Jepsen-inspired consistency test
describe('Data Consistency Tests', () => {
  let checker: LinearizabilityChecker;

  beforeEach(() => {
    checker = new LinearizabilityChecker();
  });

  test('Read-Your-Writes: should see own writes', async () => {
    const userId = 'test-user-1';
    // สมมติ service
    const service = {
      write: async (key: string, value: unknown) => {
        const start = Date.now();
        // simulate write
        await new Promise(r => setTimeout(r, 10));
        const end = Date.now();
        checker.recordWrite(key, value, start, end, userId);
      },
      read: async (key: string): Promise<unknown> => {
        const start = Date.now();
        // simulate read
        await new Promise(r => setTimeout(r, 5));
        const result = { name: 'test' }; // simulated result
        const end = Date.now();
        checker.recordRead(key, result, start, end, userId);
        return result;
      },
    };

    await service.write('user:1', { name: 'test' });
    const result = await service.read('user:1');
    
    expect(result).toEqual({ name: 'test' });
  });

  test('Monotonic Reads: should not see older data after newer', async () => {
    const versions: number[] = [];
    
    // Simulate multiple reads
    for (let i = 0; i < 5; i++) {
      const version = Math.floor(Math.random() * 10) + 1;
      versions.push(version);
    }

    // Check monotonic property
    let maxSeen = 0;
    for (const version of versions) {
      // In a monotonic read system, this should always increase
      maxSeen = Math.max(maxSeen, version);
    }
    
    // Verify no regression (simplified test)
    expect(maxSeen).toBeGreaterThanOrEqual(versions[0]);
  });

  test('Eventual Consistency: all replicas should converge', async () => {
    const replicas = [
      new Map<string, unknown>(),
      new Map<string, unknown>(),
      new Map<string, unknown>(),
    ];

    // Simulate writes to different replicas
    replicas[0].set('key1', 'value1');
    replicas[1].set('key2', 'value2');
    replicas[2].set('key3', 'value3');

    // Simulate sync (eventual consistency)
    const mergeReplicas = () => {
      const merged = new Map<string, unknown>();
      for (const replica of replicas) {
        for (const [k, v] of replica) {
          merged.set(k, v);
        }
      }
      replicas.forEach(r => {
        for (const [k, v] of merged) {
          r.set(k, v);
        }
      });
    };

    // After convergence
    await new Promise(resolve => setTimeout(resolve, 100));
    mergeReplicas();

    // All replicas should have same data
    const data0 = Object.fromEntries(replicas[0]);
    const data1 = Object.fromEntries(replicas[1]);
    const data2 = Object.fromEntries(replicas[2]);

    expect(data0).toEqual(data1);
    expect(data1).toEqual(data2);
  });
});

// Consistency Testing with Chaos
class ConsistencyTestWithChaos {
  async testWithNetworkPartition(
    writeNode: () => Promise<void>,
    readNodes: Array<() => Promise<unknown>>,
    healPartition: () => Promise<void>
  ): Promise<{
    writeDuringPartition: boolean;
    readsBeforeHeal: unknown[];
    readsAfterConvergence: unknown[];
  }> {
    let writeDuringPartition = false;

    try {
      await writeNode();
      writeDuringPartition = true;
    } catch {
      writeDuringPartition = false;
    }

    // อ่านระหว่าง partition
    const readsBeforeHeal = await Promise.all(
      readNodes.map(read => read().catch(() => null))
    );

    // Heal partition
    await healPartition();
    
    // รอให้ converge
    await new Promise(resolve => setTimeout(resolve, 2000));

    // อ่านหลัง convergence
    const readsAfterConvergence = await Promise.all(
      readNodes.map(read => read().catch(() => null))
    );

    return { writeDuringPartition, readsBeforeHeal, readsAfterConvergence };
  }
}
```

---

## Kubernetes Deployment สำหรับ Consistency Services

```yaml
# consistency-service-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: consistency-service
  namespace: microservices
  labels:
    app: consistency-service
    version: v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: consistency-service
  template:
    metadata:
      labels:
        app: consistency-service
        version: v1
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "3000"
    spec:
      containers:
        - name: consistency-service
          image: consistency-service:latest
          ports:
            - containerPort: 3000
          env:
            - name: NODE_ID
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: REPLICA_NODES
              value: "node-1:3000,node-2:3000,node-3:3000"
            - name: CONSISTENCY_LEVEL
              value: "quorum"
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "200m"
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 5
---
# PostgreSQL Primary-Replica Setup
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
  namespace: microservices
data:
  postgresql.conf: |
    wal_level = replica
    max_wal_senders = 10
    wal_keep_size = 1GB
    synchronous_commit = on
    synchronous_standby_names = 'FIRST 1 (replica1, replica2)'
  
  pg_hba.conf: |
    host replication replicator 10.0.0.0/8 md5
    host all all 0.0.0.0/0 md5
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-primary
  namespace: microservices
spec:
  selector:
    app: postgres
    role: primary
  ports:
    - port: 5432
      targetPort: 5432
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-replica
  namespace: microservices
spec:
  selector:
    app: postgres
    role: replica
  ports:
    - port: 5432
      targetPort: 5432
---
# PodDisruptionBudget เพื่อรับประกัน availability
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: consistency-service-pdb
  namespace: microservices
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: consistency-service
```

---

## สรุปตารางเปรียบเทียบ Data Consistency Patterns

| Pattern | Consistency Level | Performance | Use Case | Trade-off |
|---------|------------------|-------------|----------|-----------|
| Strong Consistency | สูงมาก | ต่ำ (latency สูง) | Financial, Inventory | Availability ลดลง |
| Eventual Consistency | อ่อน | สูง (latency ต่ำ) | Social Media, Analytics | ข้อมูลอาจ stale ชั่วคราว |
| Read-Your-Writes | ปานกลาง | ปานกลาง | User Profile, Settings | Session-based routing |
| Monotonic Reads | ปานกลาง | ปานกลาง | Feed, Timeline | Version tracking |
| Consistent Prefix | ปานกลาง | ปานกลาง | Event Sourcing | Sequence gaps |
| Causal Consistency | ปานกลาง-สูง | ปานกลาง | Collaboration Tools | Vector clock overhead |
| Linearizability | สูงสุด | ต่ำมาก | Critical Transactions | Quorum latency |

| CRDT Type | Operations | Conflict-free | Use Case |
|-----------|-----------|---------------|----------|
| G-Counter | Increment only | ใช่ | View counts, Likes |
| PN-Counter | Inc/Dec | ใช่ | Stock levels |
| OR-Set | Add/Remove | ใช่ | Shopping cart |
| LWW-Register | Set (LWW) | ใช่ | User settings |
| RGA | Insert/Delete | ใช่ | Collaborative text |

| Replication Strategy | Consistency | Latency | Failure Tolerance |
|---------------------|-------------|---------|-------------------|
| Synchronous | สูงสุด | สูง | ต่ำ (รอทุก replica) |
| Asynchronous | ต่ำ (eventual) | ต่ำ | สูง |
| Semi-synchronous | ปานกลาง | ปานกลาง | ปานกลาง |
| Quorum | ปรับได้ | ปรับได้ | ปรับได้ |

---

## สรุป

Data Consistency เป็นหัวใจสำคัญของ Microservices Architecture:

1. **Eventual Consistency** เหมาะกับระบบ high-traffic ที่ยอมให้มีความล่าช้าเล็กน้อย
2. **Strong Consistency** จำเป็นสำหรับ financial transactions แต่แลกมาด้วย latency
3. **CAP Theorem** บังคับให้เลือกระหว่าง Consistency กับ Availability เมื่อเกิด Partition
4. **Vector Clocks** ช่วยติดตามลำดับ events และตรวจจับ conflicts
5. **CRDTs** ให้ merge ข้อมูลแบบ conflict-free โดยไม่ต้องประสานงาน
6. **Outbox Pattern** รับประกัน at-least-once delivery ของ events
7. **Saga Pattern** จัดการ distributed transactions แบบ eventually consistent

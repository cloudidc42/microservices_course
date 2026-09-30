# Part 62: Distributed Lock

## บทนำ

Distributed Lock เป็น Mechanism สำหรับการ synchronize การทำงานของ Process หลายๆ ตัว
ที่รันบน Node ต่างๆ ใน Distributed System เพื่อป้องกัน Race Condition และ Concurrent Access
ปัญหาหลักคือต้องมั่นใจว่า Lock จะ release เสมอ แม้ว่า Process จะ crash

---

## 1. Redis-Based Distributed Lock (Redlock Algorithm)

### หลักการทำงานของ Redlock

```
Redlock Algorithm (ใช้กับ Redis Cluster หลาย nodes):
1. Get current timestamp T1
2. Attempt to acquire lock on N/2+1 nodes
3. If acquired on quorum with total time < TTL: lock acquired
4. Otherwise: release all locks and retry
```

### Implementation ด้วย ioredis

```typescript
// src/distributed-lock/redis-lock.service.ts
import { Injectable, Logger } from '@nestjs/common';
import Redis from 'ioredis';
import * as crypto from 'crypto';

export interface LockOptions {
  ttlMs: number;           // Time-to-live ของ lock
  retryCount?: number;     // จำนวน retry
  retryDelayMs?: number;   // Delay ระหว่าง retry
  retryJitter?: number;    // Random jitter สำหรับ retry
}

export interface LockHandle {
  key: string;
  value: string;  // Unique value สำหรับระบุ owner
  expiresAt: Date;
}

@Injectable()
export class RedisDistributedLockService {
  private readonly logger = new Logger(RedisDistributedLockService.name);

  // Lua script สำหรับ atomic SET NX PX
  private readonly SET_LOCK_SCRIPT = `
    if redis.call("SET", KEYS[1], ARGV[1], "NX", "PX", ARGV[2]) then
      return 1
    else
      return 0
    end
  `;

  // Lua script สำหรับ atomic release (ตรวจสอบ owner ก่อน delete)
  private readonly RELEASE_LOCK_SCRIPT = `
    if redis.call("GET", KEYS[1]) == ARGV[1] then
      return redis.call("DEL", KEYS[1])
    else
      return 0
    end
  `;

  // Lua script สำหรับ extend lock TTL
  private readonly EXTEND_LOCK_SCRIPT = `
    if redis.call("GET", KEYS[1]) == ARGV[1] then
      return redis.call("PEXPIRE", KEYS[1], ARGV[2])
    else
      return 0
    end
  `;

  constructor(private readonly redis: Redis) {}

  async acquire(key: string, options: LockOptions): Promise<LockHandle | null> {
    const lockKey = `lock:${key}`;
    const lockValue = crypto.randomUUID();
    const { ttlMs, retryCount = 0, retryDelayMs = 100, retryJitter = 50 } = options;

    for (let attempt = 0; attempt <= retryCount; attempt++) {
      try {
        const result = await this.redis.eval(
          this.SET_LOCK_SCRIPT,
          1,
          lockKey,
          lockValue,
          ttlMs.toString()
        );

        if (result === 1) {
          const handle: LockHandle = {
            key: lockKey,
            value: lockValue,
            expiresAt: new Date(Date.now() + ttlMs),
          };

          this.logger.debug(`Lock acquired: ${key} (expires at ${handle.expiresAt.toISOString()})`);
          return handle;
        }

        if (attempt < retryCount) {
          const jitter = Math.random() * retryJitter;
          const delay = retryDelayMs + jitter;
          this.logger.debug(`Lock ${key} busy, retrying in ${delay}ms (attempt ${attempt + 1}/${retryCount})`);
          await this.sleep(delay);
        }
      } catch (error) {
        this.logger.error(`Error acquiring lock ${key}:`, error);
        if (attempt === retryCount) throw error;
      }
    }

    this.logger.warn(`Failed to acquire lock: ${key} after ${retryCount + 1} attempts`);
    return null;
  }

  async release(handle: LockHandle): Promise<boolean> {
    try {
      const result = await this.redis.eval(
        this.RELEASE_LOCK_SCRIPT,
        1,
        handle.key,
        handle.value
      );

      const released = result === 1;
      if (released) {
        this.logger.debug(`Lock released: ${handle.key}`);
      } else {
        this.logger.warn(`Lock ${handle.key} was already released or expired`);
      }

      return released;
    } catch (error) {
      this.logger.error(`Error releasing lock ${handle.key}:`, error);
      return false;
    }
  }

  async extend(handle: LockHandle, additionalTtlMs: number): Promise<boolean> {
    try {
      const result = await this.redis.eval(
        this.EXTEND_LOCK_SCRIPT,
        1,
        handle.key,
        handle.value,
        additionalTtlMs.toString()
      );

      if (result === 1) {
        handle.expiresAt = new Date(Date.now() + additionalTtlMs);
        return true;
      }

      return false;
    } catch (error) {
      this.logger.error(`Error extending lock ${handle.key}:`, error);
      return false;
    }
  }

  // Convenience method: ทำงานกับ lock อัตโนมัติ
  async withLock<T>(
    key: string,
    options: LockOptions,
    fn: (handle: LockHandle) => Promise<T>
  ): Promise<T> {
    const handle = await this.acquire(key, options);

    if (!handle) {
      throw new Error(`Unable to acquire lock: ${key}`);
    }

    try {
      return await fn(handle);
    } finally {
      await this.release(handle);
    }
  }

  async isLocked(key: string): Promise<boolean> {
    const exists = await this.redis.exists(`lock:${key}`);
    return exists === 1;
  }

  async getLockInfo(key: string): Promise<{ owner: string; ttl: number } | null> {
    const lockKey = `lock:${key}`;
    const [value, ttl] = await Promise.all([
      this.redis.get(lockKey),
      this.redis.pttl(lockKey),
    ]);

    if (!value || ttl < 0) return null;

    return { owner: value, ttl };
  }

  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}
```

### Redlock Multi-Node Implementation

```typescript
// src/distributed-lock/redlock.service.ts
import Redis from 'ioredis';
import * as crypto from 'crypto';

interface RedlockOptions {
  driftFactor?: number;  // Clock drift factor
  retryCount?: number;
  retryDelay?: number;
  retryJitter?: number;
}

export class RedlockService {
  private readonly driftFactor: number;
  private readonly retryCount: number;
  private readonly retryDelay: number;
  private readonly retryJitter: number;
  private readonly quorum: number;

  private readonly RELEASE_SCRIPT = `
    if redis.call("GET", KEYS[1]) == ARGV[1] then
      return redis.call("DEL", KEYS[1])
    else
      return 0
    end
  `;

  constructor(
    private readonly clients: Redis[],  // หลาย Redis nodes
    options: RedlockOptions = {}
  ) {
    this.driftFactor = options.driftFactor ?? 0.01;
    this.retryCount = options.retryCount ?? 3;
    this.retryDelay = options.retryDelay ?? 200;
    this.retryJitter = options.retryJitter ?? 100;
    this.quorum = Math.floor(clients.length / 2) + 1;
  }

  async acquire(resource: string, ttlMs: number): Promise<string | null> {
    const value = crypto.randomBytes(20).toString('hex');

    for (let attempt = 0; attempt < this.retryCount; attempt++) {
      const startTime = Date.now();
      let locksAcquired = 0;

      // Try to acquire lock on each node
      const results = await Promise.allSettled(
        this.clients.map(client =>
          this.acquireOnNode(client, resource, value, ttlMs)
        )
      );

      for (const result of results) {
        if (result.status === 'fulfilled' && result.value) {
          locksAcquired++;
        }
      }

      const elapsedTime = Date.now() - startTime;
      const drift = Math.floor(ttlMs * this.driftFactor) + 2;
      const validityTime = ttlMs - elapsedTime - drift;

      if (locksAcquired >= this.quorum && validityTime > 0) {
        return value;
      }

      // ไม่ได้ quorum - release ทุก lock ที่ acquire ได้
      await this.releaseOnAllNodes(resource, value);

      if (attempt < this.retryCount - 1) {
        const delay = this.retryDelay + Math.random() * this.retryJitter;
        await new Promise(resolve => setTimeout(resolve, delay));
      }
    }

    return null;
  }

  async release(resource: string, value: string): Promise<void> {
    await this.releaseOnAllNodes(resource, value);
  }

  private async acquireOnNode(
    client: Redis,
    resource: string,
    value: string,
    ttlMs: number
  ): Promise<boolean> {
    try {
      const result = await client.set(resource, value, 'NX', 'PX', ttlMs);
      return result === 'OK';
    } catch {
      return false;
    }
  }

  private async releaseOnAllNodes(resource: string, value: string): Promise<void> {
    await Promise.allSettled(
      this.clients.map(client =>
        client.eval(this.RELEASE_SCRIPT, 1, resource, value)
      )
    );
  }
}
```

---

## 2. PostgreSQL Advisory Locks

```typescript
// src/distributed-lock/postgres-advisory-lock.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { DataSource, EntityManager } from 'typeorm';
import * as crypto from 'crypto';

@Injectable()
export class PostgresAdvisoryLockService {
  private readonly logger = new Logger(PostgresAdvisoryLockService.name);

  constructor(private readonly dataSource: DataSource) {}

  // แปลง string key เป็น bigint สำหรับ advisory lock
  private keyToBigInt(key: string): bigint {
    const hash = crypto.createHash('sha256').update(key).digest('hex');
    // ใช้แค่ 8 bytes แรก
    return BigInt('0x' + hash.substring(0, 16));
  }

  // Session-level lock (ต้อง release เอง)
  async acquireSessionLock(key: string): Promise<boolean> {
    const lockId = this.keyToBigInt(key);
    
    const result = await this.dataSource.query(
      'SELECT pg_try_advisory_lock($1) as acquired',
      [lockId.toString()]
    );
    
    const acquired = result[0].acquired as boolean;
    if (acquired) {
      this.logger.debug(`Session lock acquired: ${key}`);
    }
    
    return acquired;
  }

  async releaseSessionLock(key: string): Promise<void> {
    const lockId = this.keyToBigInt(key);
    
    await this.dataSource.query(
      'SELECT pg_advisory_unlock($1)',
      [lockId.toString()]
    );
    
    this.logger.debug(`Session lock released: ${key}`);
  }

  // Transaction-level lock (auto-release เมื่อ transaction สิ้นสุด)
  async acquireTransactionLock(
    manager: EntityManager,
    key: string
  ): Promise<void> {
    const lockId = this.keyToBigInt(key);
    
    await manager.query(
      'SELECT pg_advisory_xact_lock($1)',
      [lockId.toString()]
    );
    
    this.logger.debug(`Transaction lock acquired: ${key}`);
  }

  async tryAcquireTransactionLock(
    manager: EntityManager,
    key: string
  ): Promise<boolean> {
    const lockId = this.keyToBigInt(key);
    
    const result = await manager.query(
      'SELECT pg_try_advisory_xact_lock($1) as acquired',
      [lockId.toString()]
    );
    
    return result[0].acquired as boolean;
  }

  // Helper: ทำงานกับ session lock
  async withSessionLock<T>(
    key: string,
    fn: () => Promise<T>,
    timeoutMs = 10000
  ): Promise<T> {
    const acquired = await this.acquireSessionLock(key);
    
    if (!acquired) {
      throw new Error(`Unable to acquire advisory lock: ${key}`);
    }

    try {
      // Set statement timeout
      await this.dataSource.query(`SET LOCAL statement_timeout = ${timeoutMs}`);
      return await fn();
    } finally {
      await this.releaseSessionLock(key);
    }
  }

  // Helper: ทำงานกับ transaction lock
  async withTransactionLock<T>(
    key: string,
    fn: (manager: EntityManager) => Promise<T>
  ): Promise<T> {
    return this.dataSource.transaction(async (manager) => {
      await this.acquireTransactionLock(manager, key);
      return fn(manager);
    });
  }
}
```

---

## 3. Fencing Token

```typescript
// src/distributed-lock/fencing-token.ts

// Fencing Token แก้ปัญหา "stale lock" ที่ GC pause หรือ network partition
// ทุก lock จะได้รับ monotonically increasing token
// Resource ต้องตรวจสอบว่า token ใหม่กว่า token ที่เคยเห็น

export class FencingTokenService {
  private readonly TOKEN_KEY = 'fencing:token:counter';

  constructor(private readonly redis: Redis) {}

  async acquireLockWithToken(key: string, ttlMs: number): Promise<{
    lockHandle: LockHandle;
    token: bigint;
  } | null> {
    const lockKey = `lock:${key}`;
    const lockValue = crypto.randomUUID();

    // Lua script สำหรับ atomic lock acquire + token increment
    const ACQUIRE_WITH_TOKEN_SCRIPT = `
      local acquired = redis.call("SET", KEYS[1], ARGV[1], "NX", "PX", ARGV[2])
      if acquired then
        local token = redis.call("INCR", KEYS[2])
        return {1, token}
      else
        return {0, 0}
      end
    `;

    const result = await this.redis.eval(
      ACQUIRE_WITH_TOKEN_SCRIPT,
      2,
      lockKey,
      this.TOKEN_KEY,
      lockValue,
      ttlMs.toString()
    ) as [number, number];

    if (result[0] === 1) {
      return {
        lockHandle: {
          key: lockKey,
          value: lockValue,
          expiresAt: new Date(Date.now() + ttlMs),
        },
        token: BigInt(result[1]),
      };
    }

    return null;
  }
}

// Storage Resource ที่ใช้ Fencing Token
export class FencedStorage {
  private lastSeenToken: bigint = 0n;

  async write(
    key: string,
    value: unknown,
    fencingToken: bigint
  ): Promise<{ success: boolean; reason?: string }> {
    // ตรวจสอบว่า token ใหม่กว่า
    if (fencingToken <= this.lastSeenToken) {
      return {
        success: false,
        reason: `Stale lock: token ${fencingToken} <= last seen ${this.lastSeenToken}`,
      };
    }

    this.lastSeenToken = fencingToken;
    // Write to storage...
    return { success: true };
  }
}
```

---

## 4. TypeScript DistributedLock Class พร้อม Auto-Extend

```typescript
// src/distributed-lock/distributed-lock.class.ts
import { Logger } from '@nestjs/common';

export interface DistributedLockConfig {
  key: string;
  ttlMs: number;
  autoExtend?: boolean;
  extendIntervalMs?: number;
  maxDurationMs?: number;
}

export class DistributedLock {
  private readonly logger = new Logger(DistributedLock.name);
  private lockHandle: LockHandle | null = null;
  private extendTimer?: NodeJS.Timeout;
  private readonly startTime: number;

  constructor(
    private readonly lockService: RedisDistributedLockService,
    private readonly config: DistributedLockConfig
  ) {
    this.startTime = Date.now();
  }

  async acquire(): Promise<boolean> {
    this.lockHandle = await this.lockService.acquire(this.config.key, {
      ttlMs: this.config.ttlMs,
      retryCount: 3,
      retryDelayMs: 200,
    });

    if (this.lockHandle && this.config.autoExtend) {
      this.startAutoExtend();
    }

    return this.lockHandle !== null;
  }

  private startAutoExtend(): void {
    const interval = this.config.extendIntervalMs || Math.floor(this.config.ttlMs * 0.7);

    this.extendTimer = setInterval(async () => {
      if (!this.lockHandle) {
        clearInterval(this.extendTimer);
        return;
      }

      // ตรวจสอบว่าถึง maxDuration หรือยัง
      if (this.config.maxDurationMs) {
        const elapsed = Date.now() - this.startTime;
        if (elapsed >= this.config.maxDurationMs) {
          this.logger.warn(`Lock ${this.config.key} reached max duration, releasing`);
          await this.release();
          return;
        }
      }

      const extended = await this.lockService.extend(
        this.lockHandle,
        this.config.ttlMs
      );

      if (extended) {
        this.logger.debug(`Lock ${this.config.key} extended`);
      } else {
        this.logger.error(`Failed to extend lock ${this.config.key} - lock may have expired`);
        clearInterval(this.extendTimer);
        this.lockHandle = null;
      }
    }, interval);
  }

  async release(): Promise<void> {
    if (this.extendTimer) {
      clearInterval(this.extendTimer);
      this.extendTimer = undefined;
    }

    if (this.lockHandle) {
      await this.lockService.release(this.lockHandle);
      this.lockHandle = null;
    }
  }

  isHeld(): boolean {
    return this.lockHandle !== null;
  }

  // Static factory method
  static async execute<T>(
    lockService: RedisDistributedLockService,
    config: DistributedLockConfig,
    fn: () => Promise<T>
  ): Promise<T> {
    const lock = new DistributedLock(lockService, config);
    const acquired = await lock.acquire();

    if (!acquired) {
      throw new Error(`Cannot acquire distributed lock: ${config.key}`);
    }

    try {
      return await fn();
    } finally {
      await lock.release();
    }
  }
}
```

---

## 5. Leader Election Pattern

```typescript
// src/distributed-lock/leader-election.service.ts
import { Injectable, Logger, OnModuleInit, OnModuleDestroy } from '@nestjs/common';
import { EventEmitter } from 'events';
import Redis from 'ioredis';

@Injectable()
export class LeaderElectionService extends EventEmitter implements OnModuleInit, OnModuleDestroy {
  private readonly logger = new Logger(LeaderElectionService.name);
  private isLeader = false;
  private electionTimer?: NodeJS.Timeout;
  private heartbeatTimer?: NodeJS.Timeout;
  private readonly instanceId: string;

  private readonly LEADER_KEY = 'leader:election';
  private readonly LEADER_TTL_MS = 10000;  // 10 seconds
  private readonly ELECTION_INTERVAL_MS = 5000;  // Try every 5 seconds
  private readonly HEARTBEAT_INTERVAL_MS = 3000;  // Heartbeat every 3 seconds

  constructor(private readonly redis: Redis) {
    super();
    this.instanceId = `${process.env.POD_NAME || 'instance'}-${Date.now()}`;
  }

  async onModuleInit(): Promise<void> {
    await this.startElection();
  }

  async onModuleDestroy(): Promise<void> {
    await this.stepDown();
  }

  private async startElection(): Promise<void> {
    await this.tryBecomeLeader();

    this.electionTimer = setInterval(
      () => this.tryBecomeLeader(),
      this.ELECTION_INTERVAL_MS
    );
  }

  private async tryBecomeLeader(): Promise<void> {
    try {
      if (this.isLeader) return;

      const result = await this.redis.set(
        this.LEADER_KEY,
        this.instanceId,
        'NX',
        'PX',
        this.LEADER_TTL_MS
      );

      if (result === 'OK') {
        await this.onBecomeLeader();
      }
    } catch (error) {
      this.logger.error('Error during leader election:', error);
    }
  }

  private async onBecomeLeader(): Promise<void> {
    this.isLeader = true;
    this.logger.log(`Instance ${this.instanceId} became leader`);
    this.emit('leader:gained');

    // Start heartbeat
    this.heartbeatTimer = setInterval(
      () => this.sendHeartbeat(),
      this.HEARTBEAT_INTERVAL_MS
    );
  }

  private async sendHeartbeat(): Promise<void> {
    try {
      const HEARTBEAT_SCRIPT = `
        if redis.call("GET", KEYS[1]) == ARGV[1] then
          return redis.call("PEXPIRE", KEYS[1], ARGV[2])
        else
          return 0
        end
      `;

      const result = await this.redis.eval(
        HEARTBEAT_SCRIPT,
        1,
        this.LEADER_KEY,
        this.instanceId,
        this.LEADER_TTL_MS.toString()
      );

      if (result === 0) {
        await this.onLostLeadership();
      }
    } catch (error) {
      this.logger.error('Heartbeat failed:', error);
      await this.onLostLeadership();
    }
  }

  private async onLostLeadership(): Promise<void> {
    if (!this.isLeader) return;

    this.isLeader = false;
    this.logger.warn(`Instance ${this.instanceId} lost leadership`);
    this.emit('leader:lost');

    if (this.heartbeatTimer) {
      clearInterval(this.heartbeatTimer);
      this.heartbeatTimer = undefined;
    }
  }

  async stepDown(): Promise<void> {
    if (!this.isLeader) return;

    const STEP_DOWN_SCRIPT = `
      if redis.call("GET", KEYS[1]) == ARGV[1] then
        return redis.call("DEL", KEYS[1])
      else
        return 0
      end
    `;

    await this.redis.eval(
      STEP_DOWN_SCRIPT,
      1,
      this.LEADER_KEY,
      this.instanceId
    );

    await this.onLostLeadership();
  }

  getIsLeader(): boolean {
    return this.isLeader;
  }

  getInstanceId(): string {
    return this.instanceId;
  }

  async getCurrentLeader(): Promise<string | null> {
    return this.redis.get(this.LEADER_KEY);
  }

  // Decorator สำหรับ leader-only methods
  leaderOnly() {
    return (target: unknown, propertyKey: string, descriptor: PropertyDescriptor) => {
      const originalMethod = descriptor.value;

      descriptor.value = async function (...args: unknown[]) {
        if (!this.leaderElectionService?.isLeader) {
          this.logger.debug(`Skipping ${propertyKey}: not the leader`);
          return;
        }
        return originalMethod.apply(this, args);
      };

      return descriptor;
    };
  }
}
```

---

## 6. Mutex vs Semaphore

```typescript
// src/distributed-lock/semaphore.service.ts
import Redis from 'ioredis';

// Semaphore: อนุญาตให้ N processes ทำงานพร้อมกัน
export class DistributedSemaphore {
  private readonly logger = console;

  constructor(
    private readonly redis: Redis,
    private readonly key: string,
    private readonly maxPermits: number,
    private readonly ttlMs: number
  ) {}

  async acquire(permitId: string): Promise<boolean> {
    const ACQUIRE_SCRIPT = `
      local key = KEYS[1]
      local id = ARGV[1]
      local max = tonumber(ARGV[2])
      local ttl = tonumber(ARGV[3])
      local now = tonumber(ARGV[4])
      
      -- ลบ expired permits
      redis.call("ZREMRANGEBYSCORE", key, "-inf", now)
      
      local count = redis.call("ZCARD", key)
      if count < max then
        redis.call("ZADD", key, now + ttl, id)
        redis.call("PEXPIRE", key, ttl)
        return 1
      else
        return 0
      end
    `;

    const result = await this.redis.eval(
      ACQUIRE_SCRIPT,
      1,
      this.key,
      permitId,
      this.maxPermits.toString(),
      this.ttlMs.toString(),
      Date.now().toString()
    );

    return result === 1;
  }

  async release(permitId: string): Promise<void> {
    await this.redis.zrem(this.key, permitId);
  }

  async getAvailablePermits(): Promise<number> {
    await this.redis.zremrangebyscore(this.key, '-inf', Date.now());
    const used = await this.redis.zcard(this.key);
    return Math.max(0, this.maxPermits - used);
  }

  async withPermit<T>(fn: () => Promise<T>): Promise<T> {
    const permitId = crypto.randomUUID();
    const acquired = await this.acquire(permitId);

    if (!acquired) {
      throw new Error(`Semaphore ${this.key} is exhausted`);
    }

    try {
      return await fn();
    } finally {
      await this.release(permitId);
    }
  }
}

// Mutex: อนุญาตแค่ 1 process เท่านั้น (เหมือน Semaphore ที่ max=1)
export class DistributedMutex extends DistributedSemaphore {
  constructor(redis: Redis, key: string, ttlMs: number) {
    super(redis, key, 1, ttlMs);
  }
}
```

---

## 7. Lock Metrics and Monitoring

```typescript
// src/distributed-lock/lock-metrics.service.ts
import { Injectable } from '@nestjs/common';
import { Counter, Histogram, Gauge, register } from 'prom-client';

@Injectable()
export class LockMetricsService {
  private readonly lockAcquireAttempts: Counter;
  private readonly lockAcquireSuccess: Counter;
  private readonly lockAcquireFailed: Counter;
  private readonly lockHoldDuration: Histogram;
  private readonly lockWaitTime: Histogram;
  private readonly activeLocks: Gauge;

  constructor() {
    this.lockAcquireAttempts = new Counter({
      name: 'distributed_lock_acquire_attempts_total',
      help: 'Total number of lock acquire attempts',
      labelNames: ['lock_key', 'service'],
    });

    this.lockAcquireSuccess = new Counter({
      name: 'distributed_lock_acquire_success_total',
      help: 'Total number of successful lock acquisitions',
      labelNames: ['lock_key', 'service'],
    });

    this.lockAcquireFailed = new Counter({
      name: 'distributed_lock_acquire_failed_total',
      help: 'Total number of failed lock acquisitions',
      labelNames: ['lock_key', 'service', 'reason'],
    });

    this.lockHoldDuration = new Histogram({
      name: 'distributed_lock_hold_duration_seconds',
      help: 'Duration of lock being held',
      labelNames: ['lock_key', 'service'],
      buckets: [0.01, 0.05, 0.1, 0.5, 1, 5, 10, 30, 60],
    });

    this.lockWaitTime = new Histogram({
      name: 'distributed_lock_wait_seconds',
      help: 'Time waiting to acquire lock',
      labelNames: ['lock_key', 'service'],
      buckets: [0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1, 5],
    });

    this.activeLocks = new Gauge({
      name: 'distributed_lock_active',
      help: 'Number of currently held locks',
      labelNames: ['lock_key', 'service'],
    });
  }

  recordAcquireAttempt(key: string, service: string): void {
    this.lockAcquireAttempts.inc({ lock_key: key, service });
  }

  recordAcquireSuccess(key: string, service: string, waitTimeMs: number): void {
    this.lockAcquireSuccess.inc({ lock_key: key, service });
    this.lockWaitTime.observe({ lock_key: key, service }, waitTimeMs / 1000);
    this.activeLocks.inc({ lock_key: key, service });
  }

  recordAcquireFailed(key: string, service: string, reason: string): void {
    this.lockAcquireFailed.inc({ lock_key: key, service, reason });
  }

  recordRelease(key: string, service: string, holdDurationMs: number): void {
    this.lockHoldDuration.observe({ lock_key: key, service }, holdDurationMs / 1000);
    this.activeLocks.dec({ lock_key: key, service });
  }
}
```

---

## 8. Kubernetes Leader Election ผ่าน API

```yaml
# kubernetes-leader-election.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: leader-election-sa
  namespace: production
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: leader-election-role
  namespace: production
rules:
  - apiGroups: ["coordination.k8s.io"]
    resources: ["leases"]
    verbs: ["get", "watch", "list", "create", "update", "patch", "delete"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: leader-election-binding
  namespace: production
subjects:
  - kind: ServiceAccount
    name: leader-election-sa
roleRef:
  kind: Role
  name: leader-election-role
  apiGroup: rbac.authorization.k8s.io
```

```typescript
// src/distributed-lock/k8s-leader-election.ts
import * as k8s from '@kubernetes/client-node';

export class KubernetesLeaderElection {
  private readonly logger = console;
  private isLeader = false;
  private electionInterval?: NodeJS.Timeout;

  constructor(
    private readonly k8sClient: k8s.CoordinationV1Api,
    private readonly namespace: string,
    private readonly leaseName: string,
    private readonly identity: string,
    private readonly leaseDurationSeconds = 15,
    private readonly renewDeadlineSeconds = 10,
    private readonly retryPeriodSeconds = 2
  ) {}

  async start(
    onStartedLeading: () => Promise<void>,
    onStoppedLeading: () => void
  ): Promise<void> {
    this.electionInterval = setInterval(async () => {
      const wasLeader = this.isLeader;
      const nowLeader = await this.tryAcquireLease();

      if (!wasLeader && nowLeader) {
        this.isLeader = true;
        this.logger.log(`${this.identity} became leader`);
        await onStartedLeading();
      } else if (wasLeader && !nowLeader) {
        this.isLeader = false;
        this.logger.warn(`${this.identity} lost leadership`);
        onStoppedLeading();
      }
    }, this.retryPeriodSeconds * 1000);
  }

  private async tryAcquireLease(): Promise<boolean> {
    const now = new Date();
    const renewTime = now.toISOString();
    const acquireTime = renewTime;

    try {
      // Try to get existing lease
      const existing = await this.k8sClient.readNamespacedLease(
        this.leaseName,
        this.namespace
      );

      const lease = existing.body;
      const spec = lease.spec!;
      const currentHolder = spec.holderIdentity;
      const renewTimeDate = spec.renewTime ? new Date(spec.renewTime as string) : null;
      const leaseDurationMs = (spec.leaseDurationSeconds || this.leaseDurationSeconds) * 1000;
      const isExpired = !renewTimeDate || 
        (Date.now() - renewTimeDate.getTime()) > leaseDurationMs;

      if (currentHolder === this.identity || isExpired) {
        // Update lease
        await this.k8sClient.replaceNamespacedLease(this.leaseName, this.namespace, {
          metadata: lease.metadata,
          spec: {
            holderIdentity: this.identity,
            leaseDurationSeconds: this.leaseDurationSeconds,
            acquireTime: currentHolder !== this.identity ? acquireTime as unknown as k8s.V1MicroTime : spec.acquireTime,
            renewTime: renewTime as unknown as k8s.V1MicroTime,
            leaseTransitions: currentHolder !== this.identity 
              ? (spec.leaseTransitions || 0) + 1 
              : spec.leaseTransitions,
          },
        });

        return true;
      }

      return false;
    } catch (error: unknown) {
      const k8sError = error as { response?: { statusCode?: number } };
      if (k8sError?.response?.statusCode === 404) {
        // Create new lease
        try {
          await this.k8sClient.createNamespacedLease(this.namespace, {
            metadata: {
              name: this.leaseName,
              namespace: this.namespace,
            },
            spec: {
              holderIdentity: this.identity,
              leaseDurationSeconds: this.leaseDurationSeconds,
              acquireTime: acquireTime as unknown as k8s.V1MicroTime,
              renewTime: renewTime as unknown as k8s.V1MicroTime,
              leaseTransitions: 0,
            },
          });

          return true;
        } catch {
          return false;
        }
      }

      return false;
    }
  }

  stop(): void {
    if (this.electionInterval) {
      clearInterval(this.electionInterval);
    }
  }
}
```

---

## สรุป

| Mechanism | Library/Tool | เหมาะกับ | ข้อดี | ข้อเสีย |
|-----------|-------------|---------|-------|---------|
| Redis Single-node | ioredis | Development, Simple setups | เร็ว, ง่าย | Single point of failure |
| Redlock | redlock npm | Production clusters | HA, กระจาย | Complex, Network overhead |
| PostgreSQL Advisory | TypeORM | เมื่อมี PostgreSQL อยู่แล้ว | ACID, No extra service | Slower than Redis |
| Zookeeper | zookeeper npm | Enterprise | Proven, Strong consistency | Complex setup |
| K8s Lease API | @kubernetes/client-node | K8s deployments | Native K8s | K8s only |
| Fencing Token | Custom | Critical resources | ป้องกัน stale locks | Requires resource cooperation |
| Semaphore | Custom Redis | Rate limiting | Flexible concurrency | More complex than Mutex |
| Leader Election | Redis/K8s | Singleton jobs | HA without downtime | Election latency |

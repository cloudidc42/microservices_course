# Part 62: Distributed Lock Patterns

## บทนำ

ใน Microservices Architecture ที่มีหลาย Instance ทำงานพร้อมกัน การจัดการ Concurrent Access ต่อ Shared Resources เป็นปัญหาสำคัญ Distributed Lock ช่วยให้มั่นใจได้ว่า Operation ที่ต้องการ Mutual Exclusion จะทำงานได้ถูกต้อง บทนี้ครอบคลุม Redis SETNX, Redlock Algorithm, Database Advisory Locks, Fencing Tokens, Leader Election และการป้องกัน Deadlock

## 1. Redis SETNX-based Lock

### 1.1 Simple Redis Lock

```typescript
// src/locks/redis-lock.ts
import Redis from 'ioredis';
import { logger } from '../utils/logger';
import crypto from 'crypto';

interface LockOptions {
  ttlMs: number;           // Time-to-live ของ Lock
  retryCount?: number;     // จำนวนครั้งที่ลองใหม่
  retryDelayMs?: number;   // ช่วงเวลารอระหว่างการลอง
  retryJitter?: number;    // Random jitter สำหรับป้องกัน Thundering Herd
}

interface AcquiredLock {
  key: string;
  token: string;
  expiresAt: number;
  release: () => Promise<boolean>;
  extend: (additionalMs: number) => Promise<boolean>;
}

export class RedisLock {
  private unlockScript: string;
  private extendScript: string;

  constructor(private redis: Redis) {
    // Lua Script สำหรับ Atomic Unlock (ตรวจสอบ token ก่อน unlock)
    this.unlockScript = `
      if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("del", KEYS[1])
      else
        return 0
      end
    `;

    // Lua Script สำหรับ Atomic Extend
    this.extendScript = `
      if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("pexpire", KEYS[1], ARGV[2])
      else
        return 0
      end
    `;
  }

  async acquire(
    resource: string,
    options: LockOptions
  ): Promise<AcquiredLock | null> {
    const key = `lock:${resource}`;
    const token = crypto.randomBytes(16).toString('hex');
    const retryCount = options.retryCount ?? 0;
    const retryDelay = options.retryDelayMs ?? 100;
    const jitter = options.retryJitter ?? 50;

    for (let attempt = 0; attempt <= retryCount; attempt++) {
      if (attempt > 0) {
        const delay = retryDelay + Math.random() * jitter;
        await this.sleep(delay);
      }

      // NX = Only set if Not Exists, PX = Expire in milliseconds
      const result = await this.redis.set(
        key,
        token,
        'NX',
        'PX',
        options.ttlMs
      );

      if (result === 'OK') {
        logger.debug('Lock acquired', { resource, token: token.substring(0, 8) });
        
        const expiresAt = Date.now() + options.ttlMs;
        
        return {
          key,
          token,
          expiresAt,
          release: () => this.release(key, token),
          extend: (additionalMs: number) => this.extend(key, token, additionalMs),
        };
      }
    }

    logger.warn('Failed to acquire lock', { resource, attempts: retryCount + 1 });
    return null;
  }

  private async release(key: string, token: string): Promise<boolean> {
    const result = await this.redis.eval(
      this.unlockScript,
      1,
      key,
      token
    ) as number;

    const released = result === 1;
    
    if (released) {
      logger.debug('Lock released', { key, token: token.substring(0, 8) });
    } else {
      logger.warn('Failed to release lock (expired or stolen)', { key });
    }
    
    return released;
  }

  private async extend(
    key: string,
    token: string,
    additionalMs: number
  ): Promise<boolean> {
    const result = await this.redis.eval(
      this.extendScript,
      1,
      key,
      token,
      additionalMs.toString()
    ) as number;

    return result === 1;
  }

  // Helper สำหรับใช้งานแบบ with-lock pattern
  async withLock<T>(
    resource: string,
    options: LockOptions,
    fn: () => Promise<T>
  ): Promise<T> {
    const lock = await this.acquire(resource, options);
    
    if (!lock) {
      throw new Error(`Could not acquire lock for resource: ${resource}`);
    }

    try {
      const result = await fn();
      return result;
    } finally {
      await lock.release();
    }
  }

  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

// ตัวอย่างการใช้งาน: ป้องกัน Double Spending ในระบบ Payment ไทย
class PaymentProcessor {
  constructor(
    private redisLock: RedisLock,
    private db: any
  ) {}

  async processPayment(userId: string, amount: number, orderId: string) {
    // Lock บน User เพื่อป้องกัน Concurrent Payment
    return this.redisLock.withLock(
      `payment:user:${userId}`,
      {
        ttlMs: 30000,       // Lock 30 วินาที
        retryCount: 3,      // ลองใหม่ 3 ครั้ง
        retryDelayMs: 200,  // รอ 200ms
        retryJitter: 100,   // Random jitter 0-100ms
      },
      async () => {
        // ตรวจสอบ Balance
        const balance = await this.db.getUserBalance(userId);
        
        if (balance < amount) {
          throw new Error('Insufficient balance');
        }
        
        // ตรวจสอบว่า Order ยังไม่ถูกชำระ
        const existingPayment = await this.db.findPaymentByOrderId(orderId);
        if (existingPayment) {
          throw new Error('Order already paid');
        }
        
        // ดำเนินการชำระเงิน
        await this.db.deductBalance(userId, amount);
        const payment = await this.db.createPayment({ userId, amount, orderId });
        
        return payment;
      }
    );
  }
}
```

## 2. Redlock Algorithm

Redlock เป็น Algorithm ที่ Redis เองแนะนำสำหรับ Distributed Lock ที่ใช้ Redis Nodes หลาย Node

### 2.1 Redlock Implementation

```typescript
// src/locks/redlock.ts
import Redis from 'ioredis';
import crypto from 'crypto';
import { logger } from '../utils/logger';

interface RedlockConfig {
  retryCount: number;
  retryDelay: number;
  retryJitter: number;
  driftFactor: number;  // Clock drift factor (default 0.01 = 1%)
}

interface RedlockInstance {
  resource: string;
  value: string;
  validityTime: number;  // เวลาที่ Lock ยังใช้ได้ (ms)
  release: () => Promise<void>;
}

export class Redlock {
  private clients: Redis[];
  private quorum: number;
  private config: RedlockConfig;
  
  private unlockScript = `
    if redis.call("get", KEYS[1]) == ARGV[1] then
      return redis.call("del", KEYS[1])
    else
      return 0
    end
  `;

  constructor(clients: Redis[], config?: Partial<RedlockConfig>) {
    if (clients.length < 3) {
      throw new Error('Redlock requires at least 3 Redis instances for safety');
    }
    
    this.clients = clients;
    this.quorum = Math.floor(clients.length / 2) + 1;
    this.config = {
      retryCount: config?.retryCount ?? 3,
      retryDelay: config?.retryDelay ?? 200,
      retryJitter: config?.retryJitter ?? 100,
      driftFactor: config?.driftFactor ?? 0.01,
    };
  }

  async acquire(resource: string, ttlMs: number): Promise<RedlockInstance> {
    const value = crypto.randomBytes(20).toString('hex');
    
    for (let attempt = 0; attempt < this.config.retryCount; attempt++) {
      if (attempt > 0) {
        const delay = this.config.retryDelay + 
          Math.random() * this.config.retryJitter;
        await this.sleep(delay);
      }

      const startTime = Date.now();
      let successCount = 0;

      // ลองล็อคใน Redis ทุก Node พร้อมกัน
      const lockPromises = this.clients.map(client =>
        this.lockInstance(client, resource, value, ttlMs)
      );

      const results = await Promise.allSettled(lockPromises);
      
      for (const result of results) {
        if (result.status === 'fulfilled' && result.value) {
          successCount++;
        }
      }

      const elapsedTime = Date.now() - startTime;
      const drift = Math.ceil(this.config.driftFactor * ttlMs) + 2;
      const validityTime = ttlMs - elapsedTime - drift;

      // ต้องได้ Quorum (majority) และ Lock ยังไม่หมดอายุ
      if (successCount >= this.quorum && validityTime > 0) {
        logger.debug('Redlock acquired', {
          resource,
          successCount,
          quorum: this.quorum,
          validityTime,
        });

        return {
          resource,
          value,
          validityTime,
          release: () => this.release(resource, value),
        };
      }

      // ไม่ได้ Quorum - ปล่อย Lock ทั้งหมดที่ได้มา
      await this.releaseAll(resource, value);
      
      logger.debug('Redlock acquisition failed', {
        resource,
        attempt: attempt + 1,
        successCount,
        quorum: this.quorum,
      });
    }

    throw new Error(`Unable to acquire Redlock for resource: ${resource}`);
  }

  private async lockInstance(
    client: Redis,
    resource: string,
    value: string,
    ttlMs: number
  ): Promise<boolean> {
    try {
      const result = await Promise.race([
        client.set(resource, value, 'NX', 'PX', ttlMs),
        this.sleep(ttlMs / 3).then(() => null), // Timeout
      ]);
      
      return result === 'OK';
    } catch (error) {
      return false;
    }
  }

  private async release(resource: string, value: string): Promise<void> {
    await this.releaseAll(resource, value);
  }

  private async releaseAll(resource: string, value: string): Promise<void> {
    const releasePromises = this.clients.map(client =>
      client.eval(this.unlockScript, 1, resource, value).catch(() => 0)
    );
    
    await Promise.allSettled(releasePromises);
  }

  async withLock<T>(
    resource: string,
    ttlMs: number,
    fn: (lock: RedlockInstance) => Promise<T>
  ): Promise<T> {
    const lock = await this.acquire(resource, ttlMs);
    
    try {
      return await fn(lock);
    } finally {
      await lock.release();
    }
  }

  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

// ตัวอย่างการใช้งาน Redlock ใน E-commerce ไทย
class InventoryService {
  constructor(
    private redlock: Redlock,
    private db: any
  ) {}

  async reserveStock(productId: string, quantity: number, orderId: string) {
    // ใช้ Redlock เพื่อป้องกัน Race Condition ใน Stock Reservation
    return this.redlock.withLock(
      `inventory:product:${productId}`,
      10000, // TTL 10 วินาที
      async (lock) => {
        logger.info('Lock acquired for inventory reservation', {
          productId,
          validityTime: lock.validityTime,
        });

        const stock = await this.db.getStock(productId);
        
        if (stock.available < quantity) {
          throw new Error(`Insufficient stock. Available: ${stock.available}, Requested: ${quantity}`);
        }
        
        // จอง Stock
        await this.db.updateStock(productId, {
          available: stock.available - quantity,
          reserved: stock.reserved + quantity,
        });
        
        // สร้าง Reservation Record
        const reservation = await this.db.createReservation({
          productId,
          quantity,
          orderId,
          expiresAt: new Date(Date.now() + 900000), // หมดอายุใน 15 นาที
        });
        
        return reservation;
      }
    );
  }
}
```

## 3. Database Advisory Locks

PostgreSQL Advisory Locks เป็น Lock ที่เบากว่า Row Lock และใช้งานง่าย

### 3.1 PostgreSQL Advisory Lock Implementation

```typescript
// src/locks/pg-advisory-lock.ts
import { Pool, PoolClient } from 'pg';
import { logger } from '../utils/logger';

export class PostgresAdvisoryLock {
  constructor(private pool: Pool) {}

  // Session-level Lock (คงอยู่ตลอด Connection)
  async acquireSessionLock(lockId: number): Promise<PoolClient> {
    const client = await this.pool.connect();
    
    try {
      await client.query('SELECT pg_advisory_lock($1)', [lockId]);
      logger.debug('Session advisory lock acquired', { lockId });
      return client;
    } catch (error) {
      client.release();
      throw error;
    }
  }

  async releaseSessionLock(client: PoolClient, lockId: number): Promise<void> {
    try {
      await client.query('SELECT pg_advisory_unlock($1)', [lockId]);
      logger.debug('Session advisory lock released', { lockId });
    } finally {
      client.release();
    }
  }

  // Transaction-level Lock (ปล่อยอัตโนมัติเมื่อ Transaction จบ)
  async withTransactionLock<T>(
    lockId: number,
    fn: (client: PoolClient) => Promise<T>
  ): Promise<T> {
    const client = await this.pool.connect();
    
    try {
      await client.query('BEGIN');
      
      // pg_advisory_xact_lock จะปล่อยเมื่อ Transaction จบ
      await client.query('SELECT pg_advisory_xact_lock($1)', [lockId]);
      logger.debug('Transaction advisory lock acquired', { lockId });
      
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

  // Try Lock (ไม่ Block - Return false ทันทีถ้าได้ไม่ได้ Lock)
  async tryAcquireLock(lockId: number): Promise<boolean> {
    const result = await this.pool.query(
      'SELECT pg_try_advisory_lock($1) as acquired',
      [lockId]
    );
    
    return result.rows[0].acquired;
  }

  // Shared Lock (หลาย Reader, หนึ่ง Writer)
  async acquireSharedLock(lockId: number): Promise<void> {
    await this.pool.query('SELECT pg_advisory_lock_shared($1)', [lockId]);
  }

  // สร้าง Lock ID จาก String (ต้องการ bigint)
  static hashToLockId(str: string): number {
    let hash = 0;
    for (let i = 0; i < str.length; i++) {
      const char = str.charCodeAt(i);
      hash = ((hash << 5) - hash) + char;
      hash = hash & hash;
    }
    return Math.abs(hash);
  }
}

// ตัวอย่างการใช้งาน: ป้องกัน Duplicate Payment Processing
class TransactionProcessor {
  constructor(
    private pgLock: PostgresAdvisoryLock,
    private db: Pool
  ) {}

  async processWithdrawal(
    accountId: string,
    amount: number,
    transactionRef: string
  ) {
    const lockId = PostgresAdvisoryLock.hashToLockId(`withdrawal:${accountId}`);
    
    return this.pgLock.withTransactionLock(lockId, async (client) => {
      // ตรวจสอบ Duplicate Transaction
      const existing = await client.query(
        'SELECT id FROM transactions WHERE reference = $1 FOR UPDATE',
        [transactionRef]
      );
      
      if (existing.rows.length > 0) {
        throw new Error(`Duplicate transaction: ${transactionRef}`);
      }
      
      // ตรวจสอบ Balance ด้วย SELECT FOR UPDATE เพื่อ Lock Row
      const accountResult = await client.query(
        'SELECT balance FROM accounts WHERE id = $1 FOR UPDATE',
        [accountId]
      );
      
      if (accountResult.rows.length === 0) {
        throw new Error(`Account not found: ${accountId}`);
      }
      
      const currentBalance = parseFloat(accountResult.rows[0].balance);
      
      if (currentBalance < amount) {
        throw new Error(`Insufficient funds. Balance: ${currentBalance}, Required: ${amount}`);
      }
      
      // หักยอดเงิน
      await client.query(
        'UPDATE accounts SET balance = balance - $1 WHERE id = $2',
        [amount, accountId]
      );
      
      // บันทึก Transaction
      const result = await client.query(
        `INSERT INTO transactions (account_id, amount, type, reference, created_at)
         VALUES ($1, $2, 'withdrawal', $3, NOW())
         RETURNING *`,
        [accountId, amount, transactionRef]
      );
      
      return result.rows[0];
    });
  }
}
```

## 4. Fencing Tokens

Fencing Token ป้องกัน Zombie Lock ที่เกิดจาก Process ที่ช้าเกินไปมาแก้ไขข้อมูลหลัง Lock หมดอายุ

### 4.1 Fencing Token Implementation

```typescript
// src/locks/fencing-token-lock.ts
import Redis from 'ioredis';
import { logger } from '../utils/logger';

interface FencedLock {
  token: number;       // Monotonically increasing token
  resource: string;
  expiresAt: number;
  release: () => Promise<void>;
}

export class FencingTokenLock {
  constructor(private redis: Redis) {}

  async acquire(resource: string, ttlMs: number): Promise<FencedLock | null> {
    const lockKey = `fenced_lock:${resource}`;
    const counterKey = `fenced_lock:counter:${resource}`;
    
    const script = `
      local lockKey = KEYS[1]
      local counterKey = KEYS[2]
      local ttlMs = tonumber(ARGV[1])
      local now = tonumber(ARGV[2])
      
      -- ตรวจสอบว่า Lock ว่างหรือไม่
      if redis.call('EXISTS', lockKey) == 1 then
        return nil
      end
      
      -- เพิ่ม Counter เพื่อสร้าง Fencing Token
      local token = redis.call('INCR', counterKey)
      
      -- ตั้งค่า Lock พร้อม Token
      redis.call('HMSET', lockKey, 'token', token, 'acquired_at', now)
      redis.call('PEXPIRE', lockKey, ttlMs)
      
      return token
    `;
    
    const result = await this.redis.eval(
      script,
      2,
      lockKey,
      counterKey,
      ttlMs.toString(),
      Date.now().toString()
    ) as number | null;
    
    if (result === null) {
      return null;
    }
    
    return {
      token: result,
      resource,
      expiresAt: Date.now() + ttlMs,
      release: async () => {
        await this.redis.del(lockKey);
      },
    };
  }

  async validateToken(resource: string, token: number): Promise<boolean> {
    const lockKey = `fenced_lock:${resource}`;
    const lockToken = await this.redis.hget(lockKey, 'token');
    
    if (lockToken === null) {
      logger.warn('Lock expired when validating fencing token', { resource, token });
      return false;
    }
    
    const currentToken = parseInt(lockToken);
    
    if (token !== currentToken) {
      logger.warn('Invalid fencing token (stale lock)', {
        resource,
        providedToken: token,
        currentToken,
      });
      return false;
    }
    
    return true;
  }
}

// Storage Service ที่ใช้ Fencing Token
class StorageService {
  constructor(
    private fencingLock: FencingTokenLock,
    private storage: any
  ) {}

  async updateWithFencing(
    resource: string,
    data: any,
    fencingToken: number
  ): Promise<void> {
    // ตรวจสอบ Fencing Token ก่อนเขียนข้อมูล
    const isValid = await this.fencingLock.validateToken(resource, fencingToken);
    
    if (!isValid) {
      throw new Error(
        `Fencing token validation failed. Token ${fencingToken} is stale or invalid.`
      );
    }
    
    // เขียนข้อมูลพร้อม Token เพื่อตรวจสอบในฝั่ง Storage
    await this.storage.write(resource, {
      ...data,
      _fencingToken: fencingToken,
    });
  }
}
```

## 5. Leader Election

Leader Election ใช้สำหรับเลือก Primary Node ในระบบ Distributed

### 5.1 Leader Election Service

```typescript
// src/locks/leader-election.ts
import Redis from 'ioredis';
import { EventEmitter } from 'events';
import { logger } from '../utils/logger';
import os from 'os';

interface LeaderElectionConfig {
  electionKey: string;
  ttlMs: number;
  renewalIntervalMs: number;
  nodeId?: string;
}

type LeaderElectionEvent = 'elected' | 'deposed' | 'error';

export class LeaderElection extends EventEmitter {
  private isLeader = false;
  private renewalTimer: NodeJS.Timeout | null = null;
  private nodeId: string;
  private renewalScript: string;

  constructor(
    private redis: Redis,
    private config: LeaderElectionConfig
  ) {
    super();
    this.nodeId = config.nodeId ?? `${os.hostname()}-${process.pid}-${Date.now()}`;
    
    this.renewalScript = `
      if redis.call('get', KEYS[1]) == ARGV[1] then
        return redis.call('pexpire', KEYS[1], ARGV[2])
      else
        return 0
      end
    `;
  }

  async start(): Promise<void> {
    logger.info('Starting leader election', {
      nodeId: this.nodeId,
      key: this.config.electionKey,
    });
    
    await this.elect();
  }

  async stop(): Promise<void> {
    if (this.renewalTimer) {
      clearInterval(this.renewalTimer);
      this.renewalTimer = null;
    }
    
    if (this.isLeader) {
      await this.resign();
    }
    
    logger.info('Leader election stopped', { nodeId: this.nodeId });
  }

  private async elect(): Promise<void> {
    try {
      const result = await this.redis.set(
        this.config.electionKey,
        this.nodeId,
        'NX',
        'PX',
        this.config.ttlMs
      );

      if (result === 'OK') {
        this.becomeLeader();
      } else {
        this.isLeader = false;
        
        // ดูว่าใครเป็น Leader อยู่
        const currentLeader = await this.redis.get(this.config.electionKey);
        logger.info('Not elected as leader', {
          nodeId: this.nodeId,
          currentLeader,
        });
        
        // ลองใหม่หลังจาก TTL หมด
        const ttl = await this.redis.pttl(this.config.electionKey);
        setTimeout(() => this.elect(), Math.max(ttl, 1000));
      }
    } catch (error) {
      logger.error('Leader election error', { error });
      this.emit('error', error);
      setTimeout(() => this.elect(), 5000);
    }
  }

  private becomeLeader(): void {
    this.isLeader = true;
    logger.info('Elected as leader', { nodeId: this.nodeId });
    this.emit('elected', this.nodeId);
    
    // เริ่ม Renewal Loop
    this.renewalTimer = setInterval(
      () => this.renewLeadership(),
      this.config.renewalIntervalMs
    );
  }

  private async renewLeadership(): Promise<void> {
    if (!this.isLeader) return;
    
    try {
      const result = await this.redis.eval(
        this.renewalScript,
        1,
        this.config.electionKey,
        this.nodeId,
        this.config.ttlMs.toString()
      ) as number;

      if (result !== 1) {
        // ล้มเหลวในการ Renew - สูญเสีย Leadership
        this.isLeader = false;
        
        if (this.renewalTimer) {
          clearInterval(this.renewalTimer);
          this.renewalTimer = null;
        }
        
        logger.warn('Lost leadership', { nodeId: this.nodeId });
        this.emit('deposed', this.nodeId);
        
        // ลอง Elect ใหม่
        setTimeout(() => this.elect(), 1000);
      }
    } catch (error) {
      logger.error('Leadership renewal error', { error });
    }
  }

  private async resign(): Promise<void> {
    const script = `
      if redis.call('get', KEYS[1]) == ARGV[1] then
        return redis.call('del', KEYS[1])
      else
        return 0
      end
    `;
    
    await this.redis.eval(script, 1, this.config.electionKey, this.nodeId);
    this.isLeader = false;
    logger.info('Resigned from leadership', { nodeId: this.nodeId });
    this.emit('deposed', this.nodeId);
  }

  getIsLeader(): boolean {
    return this.isLeader;
  }

  getNodeId(): string {
    return this.nodeId;
  }

  async getCurrentLeader(): Promise<string | null> {
    return this.redis.get(this.config.electionKey);
  }
}

// ตัวอย่าง: Scheduled Job ที่ต้องทำงานบน Leader เท่านั้น
class ScheduledJobRunner {
  private leaderElection: LeaderElection;

  constructor(redis: Redis) {
    this.leaderElection = new LeaderElection(redis, {
      electionKey: 'leader:scheduled-jobs',
      ttlMs: 30000,
      renewalIntervalMs: 10000,
    });
    
    this.leaderElection.on('elected', async (nodeId: string) => {
      logger.info(`Node ${nodeId} became leader, starting scheduled jobs`);
      await this.startJobs();
    });
    
    this.leaderElection.on('deposed', async (nodeId: string) => {
      logger.info(`Node ${nodeId} lost leadership, stopping scheduled jobs`);
      await this.stopJobs();
    });
  }

  async start(): Promise<void> {
    await this.leaderElection.start();
  }

  private async startJobs(): Promise<void> {
    // เริ่ม Scheduled Jobs เช่น Daily Report, Cleanup, etc.
    setInterval(() => this.runDailyReport(), 86400000);
    setInterval(() => this.cleanupExpiredSessions(), 3600000);
  }

  private async stopJobs(): Promise<void> {
    // หยุด Scheduled Jobs
    logger.info('Stopping scheduled jobs (lost leadership)');
  }

  private async runDailyReport(): Promise<void> {
    if (!this.leaderElection.getIsLeader()) return;
    logger.info('Running daily report (leader only)');
    // Generate daily sales report
  }

  private async cleanupExpiredSessions(): Promise<void> {
    if (!this.leaderElection.getIsLeader()) return;
    logger.info('Cleaning up expired sessions (leader only)');
    // Cleanup logic
  }
}
```

## 6. Mutex Pattern

### 6.1 Async Mutex สำหรับ In-Process Locking

```typescript
// src/locks/async-mutex.ts

interface QueueItem<T> {
  resolve: (value: T) => void;
  reject: (error: Error) => void;
  fn: () => Promise<T>;
  timeoutMs?: number;
}

export class AsyncMutex {
  private locked = false;
  private queue: Array<QueueItem<any>> = [];

  async withLock<T>(fn: () => Promise<T>, timeoutMs?: number): Promise<T> {
    return new Promise<T>((resolve, reject) => {
      const item: QueueItem<T> = { resolve, reject, fn, timeoutMs };
      this.queue.push(item);
      
      if (!this.locked) {
        this.processQueue();
      }
    });
  }

  private async processQueue(): Promise<void> {
    if (this.queue.length === 0) {
      this.locked = false;
      return;
    }

    this.locked = true;
    const item = this.queue.shift()!;
    
    let timeoutHandle: NodeJS.Timeout | undefined;
    
    try {
      let result: any;
      
      if (item.timeoutMs) {
        const timeoutPromise = new Promise<never>((_, reject) => {
          timeoutHandle = setTimeout(() => {
            reject(new Error(`Mutex operation timed out after ${item.timeoutMs}ms`));
          }, item.timeoutMs);
        });
        
        result = await Promise.race([item.fn(), timeoutPromise]);
      } else {
        result = await item.fn();
      }
      
      item.resolve(result);
    } catch (error) {
      item.reject(error as Error);
    } finally {
      if (timeoutHandle) {
        clearTimeout(timeoutHandle);
      }
      
      // Process next item in queue
      setImmediate(() => this.processQueue());
    }
  }

  get isLocked(): boolean {
    return this.locked;
  }

  get queueLength(): number {
    return this.queue.length;
  }
}

// Semaphore สำหรับจำกัดจำนวน Concurrent Operations
export class Semaphore {
  private current = 0;
  private queue: Array<() => void> = [];

  constructor(private maxConcurrent: number) {}

  async acquire(): Promise<void> {
    if (this.current < this.maxConcurrent) {
      this.current++;
      return;
    }

    return new Promise(resolve => {
      this.queue.push(resolve);
    });
  }

  release(): void {
    this.current--;
    
    if (this.queue.length > 0 && this.current < this.maxConcurrent) {
      this.current++;
      const next = this.queue.shift()!;
      next();
    }
  }

  async withSemaphore<T>(fn: () => Promise<T>): Promise<T> {
    await this.acquire();
    
    try {
      return await fn();
    } finally {
      this.release();
    }
  }
}

// ตัวอย่าง: จำกัด Concurrent External API Calls
class ThaiPaymentGatewayClient {
  // จำกัด 5 Concurrent requests ไป Payment Gateway
  private semaphore = new Semaphore(5);
  
  async chargeCard(cardToken: string, amount: number): Promise<any> {
    return this.semaphore.withSemaphore(async () => {
      const response = await fetch('https://payment-gateway.th/v1/charge', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ cardToken, amount }),
      });
      return response.json();
    });
  }
}
```

## 7. Deadlock Prevention

### 7.1 Deadlock Detection และ Prevention

```typescript
// src/locks/deadlock-prevention.ts
import { logger } from '../utils/logger';

interface LockRequest {
  requestId: string;
  resourceId: string;
  requestedAt: number;
  timeoutMs: number;
}

interface WaitForGraph {
  [holderId: string]: string[]; // holder -> [waiter1, waiter2, ...]
}

export class DeadlockPrevention {
  // Lock Ordering: กำหนดลำดับในการขอ Lock เพื่อป้องกัน Deadlock
  private lockOrder = new Map<string, number>();
  private resourceLockOrder = 0;

  registerResource(resourceId: string, priority?: number): void {
    if (!this.lockOrder.has(resourceId)) {
      this.lockOrder.set(resourceId, priority ?? this.resourceLockOrder++);
    }
  }

  // ตรวจสอบว่าการขอ Lock ตามลำดับที่กำหนดหรือไม่
  validateLockOrder(
    currentHeldLocks: string[],
    requestedResource: string
  ): void {
    const requestedOrder = this.lockOrder.get(requestedResource);
    
    if (requestedOrder === undefined) {
      logger.warn(`Resource ${requestedResource} not registered for lock ordering`);
      return;
    }
    
    for (const heldResource of currentHeldLocks) {
      const heldOrder = this.lockOrder.get(heldResource);
      
      if (heldOrder !== undefined && heldOrder > requestedOrder) {
        throw new Error(
          `Lock ordering violation: Cannot acquire ${requestedResource} (order: ${requestedOrder}) ` +
          `while holding ${heldResource} (order: ${heldOrder}). ` +
          `Always acquire locks in ascending order to prevent deadlocks.`
        );
      }
    }
  }
}

// Timeout-based Deadlock Detection
export class LockManager {
  private activeLocks = new Map<string, LockRequest>();
  private waitingRequests = new Map<string, LockRequest[]>();
  
  async acquireWithDeadlockDetection(
    requestId: string,
    resourceId: string,
    timeoutMs: number,
    acquireFn: () => Promise<boolean>
  ): Promise<boolean> {
    const request: LockRequest = {
      requestId,
      resourceId,
      requestedAt: Date.now(),
      timeoutMs,
    };
    
    // ตรวจสอบ Circular Wait
    if (this.detectDeadlock(requestId, resourceId)) {
      throw new Error(
        `Potential deadlock detected: ${requestId} waiting for ${resourceId}`
      );
    }
    
    // เพิ่มเข้า Waiting Queue
    const waiting = this.waitingRequests.get(resourceId) ?? [];
    waiting.push(request);
    this.waitingRequests.set(resourceId, waiting);
    
    try {
      const acquired = await Promise.race([
        acquireFn(),
        this.createTimeout(timeoutMs, resourceId),
      ]);
      
      if (acquired) {
        this.activeLocks.set(resourceId, request);
      }
      
      return acquired;
    } finally {
      // ลบออกจาก Waiting Queue
      const remaining = (this.waitingRequests.get(resourceId) ?? [])
        .filter(r => r.requestId !== requestId);
      
      if (remaining.length === 0) {
        this.waitingRequests.delete(resourceId);
      } else {
        this.waitingRequests.set(resourceId, remaining);
      }
    }
  }
  
  private detectDeadlock(requestId: string, resourceId: string): boolean {
    // สร้าง Wait-for Graph และตรวจหา Cycle
    const visited = new Set<string>();
    const path = new Set<string>();
    
    const hasCycle = (node: string): boolean => {
      if (path.has(node)) return true;
      if (visited.has(node)) return false;
      
      visited.add(node);
      path.add(node);
      
      const waiting = this.waitingRequests.get(node) ?? [];
      for (const req of waiting) {
        const heldBy = this.activeLocks.get(req.resourceId);
        if (heldBy && hasCycle(heldBy.requestId)) {
          return true;
        }
      }
      
      path.delete(node);
      return false;
    };
    
    return hasCycle(requestId);
  }

  releaseLock(resourceId: string): void {
    this.activeLocks.delete(resourceId);
  }

  private createTimeout(ms: number, resourceId: string): Promise<never> {
    return new Promise((_, reject) =>
      setTimeout(
        () => reject(new Error(`Lock acquisition timeout for resource: ${resourceId}`)),
        ms
      )
    );
  }
}

// ตัวอย่าง: ป้องกัน Deadlock ในการโอนเงินระหว่าง Accounts
class SafeTransferService {
  private deadlockPrevention = new DeadlockPrevention();
  
  constructor(private redisLock: any, private db: any) {
    // กำหนด Lock Order สำหรับ Accounts
    // จะ Lock Account ที่มี ID น้อยกว่าก่อนเสมอ
  }

  async transfer(
    fromAccountId: string,
    toAccountId: string,
    amount: number
  ): Promise<void> {
    // กำหนดลำดับ Lock: Lock Account ID ที่น้อยกว่าก่อนเสมอ
    // เพื่อป้องกัน Deadlock ที่เกิดจากการ Lock สลับกัน
    const [firstId, secondId] = fromAccountId < toAccountId
      ? [fromAccountId, toAccountId]
      : [toAccountId, fromAccountId];
    
    const firstLock = await this.redisLock.acquire(
      `account:${firstId}`,
      { ttlMs: 10000, retryCount: 3 }
    );
    
    if (!firstLock) {
      throw new Error(`Cannot acquire lock for account: ${firstId}`);
    }
    
    try {
      const secondLock = await this.redisLock.acquire(
        `account:${secondId}`,
        { ttlMs: 10000, retryCount: 3 }
      );
      
      if (!secondLock) {
        throw new Error(`Cannot acquire lock for account: ${secondId}`);
      }
      
      try {
        // ดำเนินการโอนเงิน
        await this.db.transfer(fromAccountId, toAccountId, amount);
      } finally {
        await secondLock.release();
      }
    } finally {
      await firstLock.release();
    }
  }
}
```

## สรุป

| Pattern | Use Case | ข้อดี | ข้อเสีย | เหมาะกับ |
|---------|----------|-------|---------|---------|
| Redis SETNX | Single Redis Node | ง่าย, เร็ว | Single Point of Failure | Development, Low-criticality |
| Redlock | Multiple Redis Nodes | High Availability | ซับซ้อนขึ้น, ต้องมี 3+ Nodes | Production Payment Systems |
| PG Advisory Lock | Database-centric | Integrate กับ Transaction | ต้อง Connect DB | Financial Transactions |
| Fencing Token | Prevent Stale Writes | ป้องกัน Zombie Processes | ต้องมี Storage Support | Storage Systems |
| Leader Election | Singleton Operations | ป้องกัน Duplicate Jobs | Failover ต้องรอ TTL | Scheduled Jobs |
| Mutex (In-process) | Single Process | ไม่ต้อง External Dependency | ใช้ได้แค่ใน Process เดียว | In-memory Operations |
| Deadlock Prevention | Complex Lock Graphs | ป้องกัน Circular Wait | ต้องออกแบบ Lock Order | Multi-resource Transactions |

การเลือก Lock Pattern ที่ถูกต้องสำคัญมากสำหรับระบบ Financial ไทย ควรใช้ Redlock สำหรับ Critical Operations อย่างการชำระเงินและการโอนเงิน และใช้ PostgreSQL Advisory Lock เมื่อต้องการ Integration กับ Database Transaction เพื่อความ Atomic สูงสุด

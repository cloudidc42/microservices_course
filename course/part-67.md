# Part 67: Database Transaction Patterns

## บทนำ

Transaction patterns เป็นหัวใจสำคัญของ data consistency ใน microservices การเข้าใจ ACID properties, locking mechanisms และ pattern ต่างๆ จะช่วยให้เราออกแบบระบบที่ถูกต้องและมีประสิทธิภาพ

---

## 1. ACID Properties

ACID ย่อมาจาก Atomicity, Consistency, Isolation, Durability

```typescript
// ตัวอย่าง ACID transaction ใน PostgreSQL
import { Pool, PoolClient } from "pg";

class TransactionManager {
  constructor(private readonly pool: Pool) {}

  async runTransaction<T>(
    work: (client: PoolClient) => Promise<T>
  ): Promise<T> {
    const client = await this.pool.connect();

    try {
      await client.query("BEGIN");
      const result = await work(client);
      await client.query("COMMIT");
      return result;
    } catch (error) {
      await client.query("ROLLBACK");
      throw error;
    } finally {
      client.release();
    }
  }
}

// ตัวอย่าง: Transfer funds (ต้องการ Atomicity)
async function transferFunds(
  tm: TransactionManager,
  fromAccountId: string,
  toAccountId: string,
  amount: number
): Promise<void> {
  await tm.runTransaction(async (client) => {
    // Atomicity: ทั้ง debit และ credit ต้องสำเร็จหรือล้มเหลวพร้อมกัน
    const fromAccount = await client.query(
      "SELECT balance FROM accounts WHERE id = $1 FOR UPDATE",
      [fromAccountId]
    );

    if (fromAccount.rows[0].balance < amount) {
      throw new Error("Insufficient funds");
    }

    await client.query(
      "UPDATE accounts SET balance = balance - $1 WHERE id = $2",
      [amount, fromAccountId]
    );

    await client.query(
      "UPDATE accounts SET balance = balance + $1 WHERE id = $2",
      [amount, toAccountId]
    );

    // Log transaction
    await client.query(
      `INSERT INTO transactions (from_account, to_account, amount, created_at)
       VALUES ($1, $2, $3, NOW())`,
      [fromAccountId, toAccountId, amount]
    );
  });
}
```

---

## 2. Isolation Levels

PostgreSQL รองรับ isolation levels 4 ระดับ:

```typescript
// isolation-levels.ts
type IsolationLevel =
  | "READ UNCOMMITTED"
  | "READ COMMITTED"
  | "REPEATABLE READ"
  | "SERIALIZABLE";

async function runWithIsolation<T>(
  pool: Pool,
  level: IsolationLevel,
  work: (client: PoolClient) => Promise<T>
): Promise<T> {
  const client = await pool.connect();
  try {
    await client.query(`BEGIN ISOLATION LEVEL ${level}`);
    const result = await work(client);
    await client.query("COMMIT");
    return result;
  } catch (error) {
    await client.query("ROLLBACK");
    throw error;
  } finally {
    client.release();
  }
}

// READ COMMITTED (default) — เห็น committed data จาก transaction อื่น
async function readCommittedExample(pool: Pool) {
  await runWithIsolation(pool, "READ COMMITTED", async (client) => {
    const r1 = await client.query("SELECT balance FROM accounts WHERE id = $1", ["acc-1"]);
    // ... อาจเห็นค่าต่างกันถ้า transaction อื่น commit ระหว่างนี้
    const r2 = await client.query("SELECT balance FROM accounts WHERE id = $1", ["acc-1"]);
    // r1 และ r2 อาจต่างกัน (non-repeatable read)
  });
}

// REPEATABLE READ — ป้องกัน non-repeatable reads
async function repeatableReadExample(pool: Pool) {
  await runWithIsolation(pool, "REPEATABLE READ", async (client) => {
    const r1 = await client.query("SELECT balance FROM accounts WHERE id = $1", ["acc-1"]);
    // ... แม้ transaction อื่น commit ระหว่างนี้ ก็ยังเห็นค่าเดิม
    const r2 = await client.query("SELECT balance FROM accounts WHERE id = $1", ["acc-1"]);
    // r1 === r2 รับประกัน
  });
}

// SERIALIZABLE — ป้องกัน phantom reads, serialization anomalies
async function serializableExample(pool: Pool) {
  // ใช้สำหรับ operations ที่ต้องการ global consistency สูงสุด
  await runWithIsolation(pool, "SERIALIZABLE", async (client) => {
    // ต้องระวัง serialization failures และ retry
    const inventory = await client.query(
      "SELECT * FROM inventory WHERE product_id = $1",
      ["prod-1"]
    );
    // ...
  });
}

// Helper: auto-retry on serialization failure
async function serializableWithRetry<T>(
  pool: Pool,
  work: (client: PoolClient) => Promise<T>,
  maxRetries: number = 3
): Promise<T> {
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await runWithIsolation(pool, "SERIALIZABLE", work);
    } catch (error: any) {
      // PostgreSQL error code 40001 = serialization_failure
      if (error.code === "40001" && attempt < maxRetries) {
        const delay = Math.pow(2, attempt) * 100;
        await new Promise((resolve) => setTimeout(resolve, delay));
        continue;
      }
      throw error;
    }
  }
  throw new Error("Max retries exceeded");
}
```

---

## 3. Optimistic Locking

Optimistic locking ใช้เมื่อ conflicts เกิดน้อย — ไม่ lock rows แต่ตรวจสอบ version เมื่อ update

```typescript
// entities/product.entity.ts
interface ProductRow {
  id: string;
  name: string;
  price: number;
  stock_quantity: number;
  version: number;
  updated_at: Date;
}

class Product {
  constructor(
    public readonly id: string,
    public name: string,
    public price: number,
    public stockQuantity: number,
    public readonly version: number
  ) {}

  static fromRow(row: ProductRow): Product {
    return new Product(
      row.id,
      row.name,
      row.price,
      row.stock_quantity,
      row.version
    );
  }
}

// repositories/product.repository.ts
class ProductRepository {
  constructor(private readonly pool: Pool) {}

  async findById(id: string): Promise<Product | null> {
    const result = await this.pool.query(
      "SELECT * FROM products WHERE id = $1",
      [id]
    );
    return result.rows[0] ? Product.fromRow(result.rows[0]) : null;
  }

  async updateWithOptimisticLock(product: Product): Promise<void> {
    const result = await this.pool.query(
      `UPDATE products
       SET name = $1,
           price = $2,
           stock_quantity = $3,
           version = version + 1,
           updated_at = NOW()
       WHERE id = $4 AND version = $5`,
      [
        product.name,
        product.price,
        product.stockQuantity,
        product.id,
        product.version, // ตรวจสอบ version
      ]
    );

    if (result.rowCount === 0) {
      throw new OptimisticLockError(
        `Product ${product.id} was modified by another transaction`
      );
    }
  }
}

class OptimisticLockError extends Error {
  constructor(message: string) {
    super(message);
    this.name = "OptimisticLockError";
  }
}

// service/inventory.service.ts — พร้อม retry logic
class InventoryService {
  constructor(
    private readonly productRepo: ProductRepository,
    private readonly maxRetries: number = 3
  ) {}

  async decreaseStock(productId: string, quantity: number): Promise<void> {
    for (let attempt = 0; attempt <= this.maxRetries; attempt++) {
      try {
        const product = await this.productRepo.findById(productId);
        if (!product) throw new Error(`Product ${productId} not found`);

        if (product.stockQuantity < quantity) {
          throw new InsufficientStockError(productId, quantity, product.stockQuantity);
        }

        const updated = new Product(
          product.id,
          product.name,
          product.price,
          product.stockQuantity - quantity,
          product.version
        );

        await this.productRepo.updateWithOptimisticLock(updated);
        return; // success

      } catch (error) {
        if (error instanceof OptimisticLockError && attempt < this.maxRetries) {
          // Retry with exponential backoff
          await new Promise((resolve) =>
            setTimeout(resolve, Math.pow(2, attempt) * 50)
          );
          continue;
        }
        throw error;
      }
    }
  }
}
```

---

## 4. Pessimistic Locking

Pessimistic locking ล็อก rows ก่อนอ่าน เหมาะเมื่อมี high contention

```typescript
// Pessimistic locking ด้วย SELECT FOR UPDATE
class SeatBookingService {
  constructor(private readonly pool: Pool) {}

  async bookSeat(eventId: string, seatId: string, userId: string): Promise<void> {
    const client = await this.pool.connect();

    try {
      await client.query("BEGIN");

      // Lock the seat row
      const seatResult = await client.query(
        `SELECT * FROM seats
         WHERE event_id = $1 AND seat_id = $2
         FOR UPDATE`, // ล็อก row นี้
        [eventId, seatId]
      );

      const seat = seatResult.rows[0];
      if (!seat) throw new Error("Seat not found");
      if (seat.status !== "available") {
        throw new SeatAlreadyBookedError(seatId);
      }

      // Safe to update — no other transaction can read this row
      await client.query(
        `UPDATE seats
         SET status = 'booked', user_id = $1, booked_at = NOW()
         WHERE event_id = $2 AND seat_id = $3`,
        [userId, eventId, seatId]
      );

      await client.query("COMMIT");
    } catch (error) {
      await client.query("ROLLBACK");
      throw error;
    } finally {
      client.release();
    }
  }

  // SKIP LOCKED — สำหรับ queue processing
  async processNextPendingOrder(): Promise<Order | null> {
    const client = await this.pool.connect();

    try {
      await client.query("BEGIN");

      // Lock first available order, skip locked ones
      const result = await client.query(
        `SELECT * FROM orders
         WHERE status = 'pending'
         ORDER BY created_at ASC
         LIMIT 1
         FOR UPDATE SKIP LOCKED`
      );

      if (result.rows.length === 0) {
        await client.query("COMMIT");
        return null;
      }

      const order = result.rows[0];

      await client.query(
        "UPDATE orders SET status = 'processing', started_at = NOW() WHERE id = $1",
        [order.id]
      );

      await client.query("COMMIT");
      return Order.fromRow(order);
    } catch (error) {
      await client.query("ROLLBACK");
      throw error;
    } finally {
      client.release();
    }
  }
}
```

---

## 5. Two-Phase Commit (2PC) และข้อจำกัด

2PC ใช้เพื่อประสาน transaction ข้ามหลาย databases แต่มีข้อจำกัดมาก

```typescript
// two-phase-commit.ts
// ⚠️ 2PC มีปัญหาหลายอย่างใน distributed systems:
// - Blocking protocol: ถ้า coordinator ล่ม ทุก participant ค้าง
// - Low throughput: ต้อง 2 round trips
// - Not suitable for microservices

// PostgreSQL PREPARE TRANSACTION
class TwoPhaseCommit {
  async phase1Prepare(
    client: PoolClient,
    transactionId: string
  ): Promise<void> {
    // Phase 1: Prepare — ทำทุกอย่างแต่ยังไม่ commit
    await client.query(`PREPARE TRANSACTION '${transactionId}'`);
  }

  async phase2Commit(pool: Pool, transactionId: string): Promise<void> {
    // Phase 2: Commit ถ้าทุก participant พร้อม
    await pool.query(`COMMIT PREPARED '${transactionId}'`);
  }

  async rollbackPrepared(pool: Pool, transactionId: string): Promise<void> {
    await pool.query(`ROLLBACK PREPARED '${transactionId}'`);
  }
}

// ❌ ทำไมไม่ควรใช้ 2PC ใน Microservices
// 1. Coordinator failure → all participants blocked indefinitely
// 2. Network partition → uncertainty about commit/rollback
// 3. Performance bottleneck — synchronous coordination
// 4. Tight coupling between services

// ✅ ใช้ Saga pattern แทน (ดูตอนถัดไป)
```

---

## 6. Compensating Transactions (Saga Pattern)

Saga pattern แก้ปัญหา distributed transactions โดยใช้ compensating transactions

```typescript
// saga/order-saga.ts
interface SagaStep<T = unknown> {
  name: string;
  execute: (context: SagaContext) => Promise<T>;
  compensate: (context: SagaContext, result: T) => Promise<void>;
}

interface SagaContext {
  orderId: string;
  userId: string;
  amount: number;
  items: OrderItem[];
  executedSteps: Array<{ step: string; result: unknown }>;
}

class SagaOrchestrator {
  async execute(steps: SagaStep[], initialContext: SagaContext): Promise<void> {
    const context: SagaContext = { ...initialContext, executedSteps: [] };
    const executedSteps: Array<{ step: SagaStep; result: unknown }> = [];

    for (const step of steps) {
      try {
        console.log(`Executing step: ${step.name}`);
        const result = await step.execute(context);
        executedSteps.push({ step, result });
        context.executedSteps.push({ step: step.name, result });
      } catch (error) {
        console.error(`Step ${step.name} failed, compensating...`);

        // Compensate in reverse order
        for (const { step: executedStep, result } of executedSteps.reverse()) {
          try {
            await executedStep.compensate(context, result);
            console.log(`Compensated: ${executedStep.name}`);
          } catch (compensateError) {
            // Log and continue — compensation errors need manual intervention
            console.error(
              `Compensation failed for ${executedStep.name}:`,
              compensateError
            );
          }
        }

        throw new SagaError(`Saga failed at step: ${step.name}`, error as Error);
      }
    }
  }
}

// Concrete saga steps
const reserveInventoryStep: SagaStep<{ reservationId: string }> = {
  name: "ReserveInventory",
  async execute(ctx) {
    const reservationId = await inventoryService.reserve({
      orderId: ctx.orderId,
      items: ctx.items,
    });
    return { reservationId };
  },
  async compensate(ctx, result) {
    await inventoryService.cancelReservation(result.reservationId);
  },
};

const processPaymentStep: SagaStep<{ paymentId: string }> = {
  name: "ProcessPayment",
  async execute(ctx) {
    const paymentId = await paymentService.charge({
      userId: ctx.userId,
      amount: ctx.amount,
      orderId: ctx.orderId,
    });
    return { paymentId };
  },
  async compensate(ctx, result) {
    await paymentService.refund(result.paymentId);
  },
};

const createShipmentStep: SagaStep<{ shipmentId: string }> = {
  name: "CreateShipment",
  async execute(ctx) {
    const shipmentId = await shippingService.createShipment({
      orderId: ctx.orderId,
      userId: ctx.userId,
    });
    return { shipmentId };
  },
  async compensate(ctx, result) {
    await shippingService.cancelShipment(result.shipmentId);
  },
};

const confirmOrderStep: SagaStep = {
  name: "ConfirmOrder",
  async execute(ctx) {
    await orderService.confirm(ctx.orderId);
  },
  async compensate(ctx) {
    await orderService.cancel(ctx.orderId);
  },
};

// Run the saga
const orchestrator = new SagaOrchestrator();
await orchestrator.execute(
  [
    reserveInventoryStep,
    processPaymentStep,
    createShipmentStep,
    confirmOrderStep,
  ],
  {
    orderId: "order-123",
    userId: "user-456",
    amount: 999.99,
    items: [{ productId: "prod-1", quantity: 2 }],
    executedSteps: [],
  }
);
```

---

## 7. Savepoints

Savepoints ช่วยให้ rollback ได้บางส่วน โดยไม่ต้อง rollback ทั้ง transaction

```typescript
// savepoints.ts
class SavepointManager {
  private savepointCounter = 0;

  async withSavepoint<T>(
    client: PoolClient,
    work: () => Promise<T>
  ): Promise<T> {
    const savepointName = `sp_${++this.savepointCounter}`;

    await client.query(`SAVEPOINT ${savepointName}`);

    try {
      const result = await work();
      await client.query(`RELEASE SAVEPOINT ${savepointName}`);
      return result;
    } catch (error) {
      await client.query(`ROLLBACK TO SAVEPOINT ${savepointName}`);
      throw error;
    }
  }
}

// ตัวอย่างการใช้งาน: Bulk import ที่ skip invalid records
async function bulkImportProducts(
  pool: Pool,
  products: ProductImportData[]
): Promise<BulkImportResult> {
  const client = await pool.connect();
  const result: BulkImportResult = {
    succeeded: [],
    failed: [],
  };

  try {
    await client.query("BEGIN");
    const spManager = new SavepointManager();

    for (const product of products) {
      try {
        await spManager.withSavepoint(client, async () => {
          await client.query(
            `INSERT INTO products (id, name, price, sku)
             VALUES ($1, $2, $3, $4)`,
            [product.id, product.name, product.price, product.sku]
          );

          // Additional validations that might fail
          await validateProductConstraints(client, product);
        });

        result.succeeded.push(product.id);
      } catch (error) {
        // This product failed, but others can continue
        result.failed.push({
          id: product.id,
          error: (error as Error).message,
        });
      }
    }

    await client.query("COMMIT");
  } catch (error) {
    await client.query("ROLLBACK");
    throw error;
  } finally {
    client.release();
  }

  return result;
}
```

---

## 8. Deadlock Detection and Resolution

```typescript
// deadlock-handler.ts
const POSTGRES_DEADLOCK_ERROR_CODE = "40P01";
const POSTGRES_LOCK_TIMEOUT_ERROR_CODE = "55P03";
const POSTGRES_SERIALIZATION_FAILURE_CODE = "40001";

interface RetryConfig {
  maxAttempts: number;
  baseDelayMs: number;
  maxDelayMs: number;
  jitter: boolean;
}

const defaultRetryConfig: RetryConfig = {
  maxAttempts: 5,
  baseDelayMs: 100,
  maxDelayMs: 5000,
  jitter: true,
};

function calculateDelay(attempt: number, config: RetryConfig): number {
  const exponential = config.baseDelayMs * Math.pow(2, attempt);
  const bounded = Math.min(exponential, config.maxDelayMs);
  if (config.jitter) {
    return bounded * (0.5 + Math.random() * 0.5);
  }
  return bounded;
}

function isRetryableError(error: any): boolean {
  return [
    POSTGRES_DEADLOCK_ERROR_CODE,
    POSTGRES_SERIALIZATION_FAILURE_CODE,
  ].includes(error.code);
}

async function withDeadlockRetry<T>(
  pool: Pool,
  work: (client: PoolClient) => Promise<T>,
  config: RetryConfig = defaultRetryConfig
): Promise<T> {
  for (let attempt = 0; attempt < config.maxAttempts; attempt++) {
    const client = await pool.connect();

    try {
      await client.query("BEGIN");
      // Set lock timeout to prevent waiting forever
      await client.query("SET LOCAL lock_timeout = '5s'");
      const result = await work(client);
      await client.query("COMMIT");
      return result;
    } catch (error: any) {
      await client.query("ROLLBACK").catch(() => {});

      if (isRetryableError(error) && attempt < config.maxAttempts - 1) {
        const delay = calculateDelay(attempt, config);
        console.warn(
          `Retryable error (${error.code}) on attempt ${attempt + 1}/${config.maxAttempts}. ` +
          `Retrying in ${delay.toFixed(0)}ms...`
        );
        await new Promise((resolve) => setTimeout(resolve, delay));
        continue;
      }

      throw error;
    } finally {
      client.release();
    }
  }

  throw new Error("Max retry attempts exceeded");
}

// Deadlock prevention: consistent lock ordering
// Always lock accounts in the same order to prevent deadlocks
async function transferWithDeadlockPrevention(
  pool: Pool,
  fromId: string,
  toId: string,
  amount: number
): Promise<void> {
  await withDeadlockRetry(pool, async (client) => {
    // Sort IDs to ensure consistent locking order
    const [firstId, secondId] = [fromId, toId].sort();

    // Lock in consistent order
    await client.query(
      "SELECT id FROM accounts WHERE id = ANY($1) FOR UPDATE",
      [[firstId, secondId]]
    );

    // Perform transfer
    await client.query(
      "UPDATE accounts SET balance = balance - $1 WHERE id = $2",
      [amount, fromId]
    );
    await client.query(
      "UPDATE accounts SET balance = balance + $1 WHERE id = $2",
      [amount, toId]
    );
  });
}

// Monitoring for deadlocks
async function monitorDeadlocks(pool: Pool): Promise<void> {
  const result = await pool.query(`
    SELECT
      blocked_locks.pid AS blocked_pid,
      blocked_activity.usename AS blocked_user,
      blocking_locks.pid AS blocking_pid,
      blocking_activity.usename AS blocking_user,
      blocked_activity.query AS blocked_statement,
      blocking_activity.query AS current_statement_in_blocking_process
    FROM pg_catalog.pg_locks blocked_locks
    JOIN pg_catalog.pg_stat_activity blocked_activity
      ON blocked_activity.pid = blocked_locks.pid
    JOIN pg_catalog.pg_locks blocking_locks
      ON blocking_locks.locktype = blocked_locks.locktype
      AND blocking_locks.DATABASE IS NOT DISTINCT FROM blocked_locks.DATABASE
      AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
      AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
      AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
      AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
      AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
      AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
      AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
      AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
      AND blocking_locks.pid != blocked_locks.pid
    JOIN pg_catalog.pg_stat_activity blocking_activity
      ON blocking_activity.pid = blocking_locks.pid
    WHERE NOT blocked_locks.GRANTED
  `);

  if (result.rows.length > 0) {
    console.warn("Active lock waits detected:", result.rows);
    metrics.gauge("db.lock_waits", result.rows.length);
  }
}
```

---

## 9. Connection Pool Management

```typescript
// database/connection-pool.ts
import { Pool, PoolConfig } from "pg";
import { createClient } from "redis";

interface ConnectionPoolConfig {
  min: number;
  max: number;
  idleTimeoutMs: number;
  connectionTimeoutMs: number;
  statementTimeoutMs?: number;
}

class DatabaseConnectionPool {
  private readonly pool: Pool;
  private readonly healthCheckInterval: NodeJS.Timer;

  constructor(
    connectionString: string,
    config: ConnectionPoolConfig
  ) {
    const poolConfig: PoolConfig = {
      connectionString,
      min: config.min,
      max: config.max,
      idleTimeoutMillis: config.idleTimeoutMs,
      connectionTimeoutMillis: config.connectionTimeoutMs,
    };

    this.pool = new Pool(poolConfig);

    // Event handlers
    this.pool.on("connect", (client) => {
      if (config.statementTimeoutMs) {
        client.query(
          `SET statement_timeout = ${config.statementTimeoutMs}`
        );
      }
      metrics.increment("db.connections.opened");
    });

    this.pool.on("remove", () => {
      metrics.increment("db.connections.closed");
    });

    this.pool.on("error", (err) => {
      logger.error("Unexpected DB pool error", err);
      metrics.increment("db.pool.errors");
    });

    // Health check
    this.healthCheckInterval = setInterval(
      () => this.checkHealth(),
      30000
    );
  }

  async getClient(): Promise<PoolClient> {
    const client = await this.pool.connect();
    metrics.gauge("db.pool.waiting", this.pool.waitingCount);
    metrics.gauge("db.pool.idle", this.pool.idleCount);
    metrics.gauge("db.pool.total", this.pool.totalCount);
    return client;
  }

  private async checkHealth(): Promise<void> {
    try {
      await this.pool.query("SELECT 1");
      metrics.gauge("db.health", 1);
    } catch (error) {
      logger.error("Database health check failed", error as Error);
      metrics.gauge("db.health", 0);
    }
  }

  async close(): Promise<void> {
    clearInterval(this.healthCheckInterval);
    await this.pool.end();
  }

  get stats() {
    return {
      total: this.pool.totalCount,
      idle: this.pool.idleCount,
      waiting: this.pool.waitingCount,
    };
  }
}

// Read replica routing
class DatabaseRouter {
  constructor(
    private readonly primary: DatabaseConnectionPool,
    private readonly replicas: DatabaseConnectionPool[]
  ) {}

  getPrimary(): DatabaseConnectionPool {
    return this.primary;
  }

  getReplica(): DatabaseConnectionPool {
    // Round-robin across replicas
    const index = Math.floor(Math.random() * this.replicas.length);
    return this.replicas[index];
  }

  getForQuery(isReadOnly: boolean): DatabaseConnectionPool {
    return isReadOnly ? this.getReplica() : this.getPrimary();
  }
}
```

---

## 10. Database Migration Strategy

```typescript
// migrations/migration-runner.ts
import { readdir, readFile } from "fs/promises";
import { join } from "path";
import { Pool } from "pg";
import crypto from "crypto";

interface Migration {
  version: string;
  name: string;
  up: string;
  down: string;
  checksum: string;
}

class MigrationRunner {
  constructor(
    private readonly pool: Pool,
    private readonly migrationsDir: string
  ) {}

  async setup(): Promise<void> {
    await this.pool.query(`
      CREATE TABLE IF NOT EXISTS schema_migrations (
        version VARCHAR(255) PRIMARY KEY,
        name VARCHAR(255) NOT NULL,
        checksum VARCHAR(64) NOT NULL,
        applied_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
      )
    `);
  }

  async loadMigrations(): Promise<Migration[]> {
    const files = await readdir(this.migrationsDir);
    const sqlFiles = files
      .filter((f) => f.endsWith(".sql"))
      .sort();

    const migrations: Migration[] = [];

    for (const file of sqlFiles) {
      const content = await readFile(join(this.migrationsDir, file), "utf-8");
      const [up, down] = content.split("-- Down");

      migrations.push({
        version: file.split("_")[0],
        name: file.replace(".sql", ""),
        up: up.replace("-- Up", "").trim(),
        down: (down ?? "").trim(),
        checksum: crypto.createHash("sha256").update(content).digest("hex"),
      });
    }

    return migrations;
  }

  async migrate(): Promise<void> {
    await this.setup();

    const migrations = await this.loadMigrations();
    const applied = await this.getAppliedMigrations();

    for (const migration of migrations) {
      if (applied.has(migration.version)) {
        // Verify checksum hasn't changed
        const appliedChecksum = applied.get(migration.version)!.checksum;
        if (appliedChecksum !== migration.checksum) {
          throw new Error(
            `Migration ${migration.version} has been modified after application!`
          );
        }
        continue;
      }

      console.log(`Applying migration: ${migration.name}`);

      const client = await this.pool.connect();
      try {
        await client.query("BEGIN");
        await client.query(migration.up);
        await client.query(
          `INSERT INTO schema_migrations (version, name, checksum)
           VALUES ($1, $2, $3)`,
          [migration.version, migration.name, migration.checksum]
        );
        await client.query("COMMIT");
        console.log(`✓ Applied: ${migration.name}`);
      } catch (error) {
        await client.query("ROLLBACK");
        throw new Error(
          `Migration ${migration.name} failed: ${(error as Error).message}`
        );
      } finally {
        client.release();
      }
    }
  }

  private async getAppliedMigrations(): Promise<Map<string, { checksum: string }>> {
    const result = await this.pool.query(
      "SELECT version, checksum FROM schema_migrations ORDER BY version"
    );
    return new Map(result.rows.map((r) => [r.version, { checksum: r.checksum }]));
  }
}

// Example migration file: 0001_create_users.sql
/*
-- Up
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR(255) UNIQUE NOT NULL,
  name VARCHAR(255) NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  email_verified BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  deleted_at TIMESTAMP WITH TIME ZONE
);

CREATE INDEX idx_users_email ON users(email) WHERE deleted_at IS NULL;
CREATE INDEX idx_users_created_at ON users(created_at);

-- Down
DROP TABLE IF EXISTS users;
*/
```

---

## 11. Database Monitoring และ Observability

```typescript
// monitoring/db-metrics.ts
import { Pool } from "pg";
import { register, Gauge, Histogram, Counter } from "prom-client";

export function setupDatabaseMetrics(pool: Pool, prefix: string = "db"): void {
  const poolSizeGauge = new Gauge({
    name: `${prefix}_pool_size`,
    help: "Current pool size",
    labelNames: ["state"],
  });

  const queryDuration = new Histogram({
    name: `${prefix}_query_duration_seconds`,
    help: "Query execution time",
    labelNames: ["operation", "table"],
    buckets: [0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1, 5],
  });

  const queryErrors = new Counter({
    name: `${prefix}_query_errors_total`,
    help: "Total query errors",
    labelNames: ["error_code"],
  });

  const transactionCounter = new Counter({
    name: `${prefix}_transactions_total`,
    help: "Total transactions",
    labelNames: ["status"],
  });

  // Update pool metrics periodically
  setInterval(() => {
    poolSizeGauge.labels("total").set(pool.totalCount);
    poolSizeGauge.labels("idle").set(pool.idleCount);
    poolSizeGauge.labels("waiting").set(pool.waitingCount);
  }, 5000);

  // Wrap pool.query to collect metrics
  const originalQuery = pool.query.bind(pool);
  (pool as any).query = async (...args: any[]) => {
    const timer = queryDuration.startTimer();
    try {
      const result = await originalQuery(...args);
      timer({ operation: "query", table: "unknown" });
      return result;
    } catch (error: any) {
      queryErrors.labels(error.code ?? "unknown").inc();
      throw error;
    }
  };
}

// Slow query logging
async function setupSlowQueryLogging(pool: Pool, thresholdMs: number = 1000): Promise<void> {
  await pool.query(`
    ALTER SYSTEM SET log_min_duration_statement = ${thresholdMs};
    SELECT pg_reload_conf();
  `);

  console.log(`Slow query logging enabled (>${thresholdMs}ms)`);
}
```

---

## สรุป

ใน Part นี้เราได้เรียนรู้ Database Transaction Patterns ที่สำคัญ:

1. **ACID Properties** — Atomicity, Consistency, Isolation, Durability คือรากฐานของ database transactions
2. **Isolation Levels** — Read Committed, Repeatable Read, Serializable มีระดับความปลอดภัยและ performance ต่างกัน
3. **Optimistic Locking** — เหมาะสำหรับ low-contention scenarios ใช้ version column ตรวจสอบ conflicts
4. **Pessimistic Locking** — เหมาะสำหรับ high-contention scenarios ใช้ SELECT FOR UPDATE
5. **Two-Phase Commit** — มีข้อจำกัดมากใน distributed systems ควรหลีกเลี่ยง
6. **Compensating Transactions (Saga)** — วิธีที่ดีกว่าสำหรับ distributed transactions
7. **Savepoints** — Partial rollback ภายใน transaction
8. **Deadlock Prevention** — Consistent lock ordering, retry with backoff
9. **Connection Pool Management** — Monitor pool health, route reads to replicas
10. **Database Migrations** — Version control สำหรับ schema changes

Key takeaway: ใน microservices ควรหลีกเลี่ยง distributed transactions และใช้ Saga pattern แทน โดยยอมรับ eventual consistency

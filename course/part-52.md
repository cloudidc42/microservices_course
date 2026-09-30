# Part 52: Saga Pattern — Choreography vs Orchestration

## บทนำ

Saga Pattern เป็นหนึ่งในวิธีการจัดการ **Distributed Transaction** ใน Microservices Architecture ที่ได้รับความนิยมมากที่สุด เนื่องจาก Microservices แต่ละตัวมี Database ของตัวเอง การทำ ACID Transaction แบบ Traditional ข้าม Service จึงทำไม่ได้โดยตรง Saga จึงเป็นทางออกที่ใช้ sequence ของ Local Transaction และ Compensation Transaction แทน

### ทำไมต้องใช้ Saga?

ใน E-commerce system การสั่งซื้อสินค้าต้องผ่านหลายขั้นตอน:
1. สร้าง Order
2. ตัด Stock สินค้า
3. ตัดเงินจาก Payment
4. จัดส่งสินค้า (Fulfillment)

ถ้าขั้นตอนที่ 3 ล้มเหลว เราต้องย้อนกลับขั้นตอนที่ 1 และ 2 — นี่คือสิ่งที่ Saga ทำ

---

## 1. สองแนวทางของ Saga Pattern

### 1.1 Choreography-based Saga

ใน Choreography แต่ละ Service จะฟัง Event และตัดสินใจเอง ไม่มี Central Coordinator

```
OrderService → publishes OrderCreated
  → InventoryService listens → reserves stock → publishes StockReserved
    → PaymentService listens → charges card → publishes PaymentCharged
      → FulfillmentService listens → ships order → publishes OrderFulfilled
```

**ข้อดี:**
- ไม่มี Single Point of Failure
- Service แต่ละตัว Loose Coupling

**ข้อเสีย:**
- ยากต่อการ Debug (ต้องตามดู Event ทั่วระบบ)
- Logic กระจายอยู่ใน Service ต่างๆ

### 1.2 Orchestration-based Saga

ใน Orchestration มี Saga Orchestrator ที่เป็น Central Controller บอกแต่ละ Service ว่าต้องทำอะไร

```
SagaOrchestrator
  → calls OrderService.createOrder()
  → calls InventoryService.reserveStock()
  → calls PaymentService.chargeCard()
  → calls FulfillmentService.shipOrder()
  → on failure: calls compensating transactions in reverse order
```

**ข้อดี:**
- มองเห็น Flow ชัดเจน
- ง่ายต่อการ Debug
- Centralized Error Handling

**ข้อเสีย:**
- Orchestrator กลายเป็น God Object ได้
- Service ต้องรู้จัก Orchestrator

---

## 2. Project Structure

```
saga-pattern/
├── packages/
│   ├── saga-orchestrator/
│   │   ├── src/
│   │   │   ├── saga.engine.ts
│   │   │   ├── saga.definition.ts
│   │   │   ├── saga.log.ts
│   │   │   └── sagas/
│   │   │       └── create-order.saga.ts
│   ├── order-service/
│   │   └── src/
│   │       ├── order.service.ts
│   │       └── compensation/
│   │           └── cancel-order.handler.ts
│   ├── inventory-service/
│   │   └── src/
│   │       ├── inventory.service.ts
│   │       └── compensation/
│   │           └── release-stock.handler.ts
│   └── payment-service/
│       └── src/
│           ├── payment.service.ts
│           └── compensation/
│               └── refund-payment.handler.ts
├── docker-compose.yml
└── db/
    └── saga-schema.sql
```

---

## 3. Database Schema สำหรับ Saga Log

```sql
-- db/saga-schema.sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE saga_instances (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    saga_type       VARCHAR(100) NOT NULL,
    correlation_id  VARCHAR(255) NOT NULL UNIQUE,
    status          VARCHAR(50) NOT NULL DEFAULT 'STARTED',
    -- STARTED | RUNNING | COMPLETED | COMPENSATING | COMPENSATED | FAILED
    current_step    INTEGER NOT NULL DEFAULT 0,
    context         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at    TIMESTAMPTZ,
    error_message   TEXT,
    CONSTRAINT chk_status CHECK (status IN (
        'STARTED', 'RUNNING', 'COMPLETED',
        'COMPENSATING', 'COMPENSATED', 'FAILED'
    ))
);

CREATE INDEX idx_saga_correlation_id ON saga_instances(correlation_id);
CREATE INDEX idx_saga_status ON saga_instances(status);
CREATE INDEX idx_saga_type_status ON saga_instances(saga_type, status);

CREATE TABLE saga_step_logs (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    saga_id         UUID NOT NULL REFERENCES saga_instances(id) ON DELETE CASCADE,
    step_number     INTEGER NOT NULL,
    step_name       VARCHAR(100) NOT NULL,
    action          VARCHAR(50) NOT NULL, -- EXECUTE | COMPENSATE
    status          VARCHAR(50) NOT NULL, -- PENDING | RUNNING | COMPLETED | FAILED | SKIPPED
    input_data      JSONB,
    output_data     JSONB,
    error_details   JSONB,
    started_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at    TIMESTAMPTZ,
    duration_ms     INTEGER,
    retry_count     INTEGER NOT NULL DEFAULT 0,
    CONSTRAINT chk_action CHECK (action IN ('EXECUTE', 'COMPENSATE')),
    CONSTRAINT chk_step_status CHECK (status IN (
        'PENDING', 'RUNNING', 'COMPLETED', 'FAILED', 'SKIPPED'
    ))
);

CREATE INDEX idx_step_logs_saga_id ON saga_step_logs(saga_id);
CREATE INDEX idx_step_logs_step ON saga_step_logs(saga_id, step_number);

-- View สำหรับดู Saga ที่กำลัง Stuck
CREATE VIEW stuck_sagas AS
SELECT
    s.id,
    s.saga_type,
    s.correlation_id,
    s.status,
    s.current_step,
    s.created_at,
    NOW() - s.updated_at AS stuck_duration,
    s.context
FROM saga_instances s
WHERE s.status IN ('RUNNING', 'COMPENSATING')
  AND s.updated_at < NOW() - INTERVAL '5 minutes';

-- Function สำหรับ cleanup old sagas
CREATE OR REPLACE FUNCTION cleanup_old_sagas(retention_days INTEGER DEFAULT 30)
RETURNS INTEGER AS $$
DECLARE
    deleted_count INTEGER;
BEGIN
    DELETE FROM saga_instances
    WHERE status IN ('COMPLETED', 'COMPENSATED', 'FAILED')
      AND completed_at < NOW() - (retention_days || ' days')::INTERVAL;
    
    GET DIAGNOSTICS deleted_count = ROW_COUNT;
    RETURN deleted_count;
END;
$$ LANGUAGE plpgsql;
```

---

## 4. Core Types และ Interfaces

```typescript
// packages/saga-orchestrator/src/types.ts

export interface SagaContext {
  correlationId: string;
  [key: string]: unknown;
}

export interface StepResult {
  success: boolean;
  data?: unknown;
  error?: Error;
}

export interface SagaStep<TContext extends SagaContext> {
  name: string;
  execute: (context: TContext) => Promise<StepResult>;
  compensate: (context: TContext) => Promise<StepResult>;
  timeout?: number; // milliseconds
  retryPolicy?: RetryPolicy;
}

export interface RetryPolicy {
  maxAttempts: number;
  backoffMs: number;
  backoffMultiplier: number;
  maxBackoffMs: number;
}

export interface SagaDefinition<TContext extends SagaContext> {
  name: string;
  steps: SagaStep<TContext>[];
  onComplete?: (context: TContext) => Promise<void>;
  onCompensated?: (context: TContext) => Promise<void>;
  onFailed?: (context: TContext, error: Error) => Promise<void>;
}

export type SagaStatus =
  | 'STARTED'
  | 'RUNNING'
  | 'COMPLETED'
  | 'COMPENSATING'
  | 'COMPENSATED'
  | 'FAILED';

export interface SagaInstance {
  id: string;
  sagaType: string;
  correlationId: string;
  status: SagaStatus;
  currentStep: number;
  context: SagaContext;
  createdAt: Date;
  updatedAt: Date;
  completedAt?: Date;
  errorMessage?: string;
}
```

---

## 5. Saga Log Repository

```typescript
// packages/saga-orchestrator/src/saga.log.ts

import { Pool } from 'pg';
import { SagaContext, SagaInstance, SagaStatus } from './types';

export class SagaLogRepository {
  constructor(private readonly pool: Pool) {}

  async createSaga(
    sagaType: string,
    correlationId: string,
    context: SagaContext
  ): Promise<SagaInstance> {
    const result = await this.pool.query(
      `INSERT INTO saga_instances 
         (saga_type, correlation_id, status, current_step, context)
       VALUES ($1, $2, 'STARTED', 0, $3)
       RETURNING *`,
      [sagaType, correlationId, JSON.stringify(context)]
    );

    return this.mapToInstance(result.rows[0]);
  }

  async updateSagaStatus(
    sagaId: string,
    status: SagaStatus,
    currentStep: number,
    context: SagaContext,
    errorMessage?: string
  ): Promise<void> {
    const completedAt =
      status === 'COMPLETED' || status === 'COMPENSATED' || status === 'FAILED'
        ? 'NOW()'
        : 'NULL';

    await this.pool.query(
      `UPDATE saga_instances
       SET status = $1,
           current_step = $2,
           context = $3,
           updated_at = NOW(),
           completed_at = ${completedAt},
           error_message = $4
       WHERE id = $5`,
      [status, currentStep, JSON.stringify(context), errorMessage ?? null, sagaId]
    );
  }

  async logStep(
    sagaId: string,
    stepNumber: number,
    stepName: string,
    action: 'EXECUTE' | 'COMPENSATE',
    status: 'PENDING' | 'RUNNING' | 'COMPLETED' | 'FAILED',
    inputData?: unknown,
    outputData?: unknown,
    errorDetails?: unknown,
    durationMs?: number,
    retryCount?: number
  ): Promise<string> {
    const result = await this.pool.query(
      `INSERT INTO saga_step_logs 
         (saga_id, step_number, step_name, action, status,
          input_data, output_data, error_details, duration_ms, retry_count)
       VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10)
       ON CONFLICT (saga_id, step_number, action) 
       DO UPDATE SET
         status = EXCLUDED.status,
         output_data = EXCLUDED.output_data,
         error_details = EXCLUDED.error_details,
         completed_at = CASE WHEN EXCLUDED.status IN ('COMPLETED', 'FAILED') THEN NOW() END,
         duration_ms = EXCLUDED.duration_ms,
         retry_count = EXCLUDED.retry_count
       RETURNING id`,
      [
        sagaId,
        stepNumber,
        stepName,
        action,
        status,
        inputData ? JSON.stringify(inputData) : null,
        outputData ? JSON.stringify(outputData) : null,
        errorDetails ? JSON.stringify(errorDetails) : null,
        durationMs ?? null,
        retryCount ?? 0,
      ]
    );

    return result.rows[0].id;
  }

  async findByCorrelationId(correlationId: string): Promise<SagaInstance | null> {
    const result = await this.pool.query(
      `SELECT * FROM saga_instances WHERE correlation_id = $1`,
      [correlationId]
    );

    if (result.rows.length === 0) return null;
    return this.mapToInstance(result.rows[0]);
  }

  async findStuckSagas(): Promise<SagaInstance[]> {
    const result = await this.pool.query(
      `SELECT * FROM stuck_sagas ORDER BY stuck_duration DESC LIMIT 100`
    );

    return result.rows.map(this.mapToInstance);
  }

  async findById(sagaId: string): Promise<SagaInstance | null> {
    const result = await this.pool.query(
      `SELECT * FROM saga_instances WHERE id = $1`,
      [sagaId]
    );

    if (result.rows.length === 0) return null;
    return this.mapToInstance(result.rows[0]);
  }

  private mapToInstance(row: Record<string, unknown>): SagaInstance {
    return {
      id: row.id as string,
      sagaType: row.saga_type as string,
      correlationId: row.correlation_id as string,
      status: row.status as SagaStatus,
      currentStep: row.current_step as number,
      context: row.context as SagaContext,
      createdAt: row.created_at as Date,
      updatedAt: row.updated_at as Date,
      completedAt: row.completed_at as Date | undefined,
      errorMessage: row.error_message as string | undefined,
    };
  }
}
```

---

## 6. Saga Engine (Orchestration Core)

```typescript
// packages/saga-orchestrator/src/saga.engine.ts

import { Logger } from './logger';
import { SagaLogRepository } from './saga.log';
import {
  RetryPolicy,
  SagaContext,
  SagaDefinition,
  SagaInstance,
  StepResult,
} from './types';

const DEFAULT_RETRY_POLICY: RetryPolicy = {
  maxAttempts: 3,
  backoffMs: 1000,
  backoffMultiplier: 2,
  maxBackoffMs: 30000,
};

const DEFAULT_STEP_TIMEOUT_MS = 30000; // 30 seconds

export class SagaEngine<TContext extends SagaContext> {
  private readonly logger: Logger;

  constructor(
    private readonly definition: SagaDefinition<TContext>,
    private readonly repository: SagaLogRepository
  ) {
    this.logger = new Logger(`SagaEngine:${definition.name}`);
  }

  async execute(context: TContext): Promise<SagaInstance> {
    // ตรวจสอบว่า Saga นี้ถูกรันไปแล้วหรือยัง (idempotency)
    const existing = await this.repository.findByCorrelationId(context.correlationId);
    if (existing) {
      this.logger.warn(`Saga already exists for correlationId: ${context.correlationId}`, {
        sagaId: existing.id,
        status: existing.status,
      });

      if (existing.status === 'COMPLETED') {
        return existing;
      }

      // ถ้า Saga ค้างอยู่ ลองทำใหม่
      if (existing.status === 'RUNNING') {
        return this.resume(existing, context);
      }

      return existing;
    }

    // สร้าง Saga Instance ใหม่
    const sagaInstance = await this.repository.createSaga(
      this.definition.name,
      context.correlationId,
      context
    );

    this.logger.info(`Starting saga: ${this.definition.name}`, {
      sagaId: sagaInstance.id,
      correlationId: context.correlationId,
    });

    return this.runSteps(sagaInstance, context, 0);
  }

  private async resume(
    instance: SagaInstance,
    context: TContext
  ): Promise<SagaInstance> {
    this.logger.info(`Resuming saga from step ${instance.currentStep}`, {
      sagaId: instance.id,
    });

    return this.runSteps(instance, context, instance.currentStep);
  }

  private async runSteps(
    instance: SagaInstance,
    context: TContext,
    startStep: number
  ): Promise<SagaInstance> {
    const { steps } = this.definition;

    await this.repository.updateSagaStatus(
      instance.id,
      'RUNNING',
      startStep,
      context
    );

    for (let i = startStep; i < steps.length; i++) {
      const step = steps[i];

      this.logger.info(`Executing step ${i}: ${step.name}`, {
        sagaId: instance.id,
      });

      await this.repository.logStep(
        instance.id,
        i,
        step.name,
        'EXECUTE',
        'RUNNING',
        context
      );

      const startTime = Date.now();
      let result: StepResult;
      let retryCount = 0;

      try {
        result = await this.executeWithTimeoutAndRetry(
          () => step.execute(context),
          step.timeout ?? DEFAULT_STEP_TIMEOUT_MS,
          step.retryPolicy ?? DEFAULT_RETRY_POLICY,
          (attempt) => {
            retryCount = attempt;
          }
        );
      } catch (error) {
        const duration = Date.now() - startTime;
        const err = error as Error;

        await this.repository.logStep(
          instance.id,
          i,
          step.name,
          'EXECUTE',
          'FAILED',
          context,
          undefined,
          { message: err.message, stack: err.stack },
          duration,
          retryCount
        );

        this.logger.error(`Step ${i} (${step.name}) failed`, { error: err });

        // เริ่ม Compensation
        return this.compensate(instance, context, i - 1);
      }

      const duration = Date.now() - startTime;

      if (!result.success) {
        await this.repository.logStep(
          instance.id,
          i,
          step.name,
          'EXECUTE',
          'FAILED',
          context,
          result.data,
          result.error
            ? { message: result.error.message }
            : { message: 'Step returned failure' },
          duration,
          retryCount
        );

        this.logger.warn(`Step ${i} (${step.name}) returned failure`, {
          sagaId: instance.id,
        });

        return this.compensate(instance, context, i - 1);
      }

      // อัพเดต context ด้วยผลลัพธ์จาก step
      if (result.data) {
        Object.assign(context, result.data);
      }

      await this.repository.logStep(
        instance.id,
        i,
        step.name,
        'EXECUTE',
        'COMPLETED',
        context,
        result.data,
        undefined,
        duration,
        retryCount
      );

      await this.repository.updateSagaStatus(
        instance.id,
        'RUNNING',
        i + 1,
        context
      );
    }

    // Saga สำเร็จ
    await this.repository.updateSagaStatus(
      instance.id,
      'COMPLETED',
      steps.length,
      context
    );

    const finalInstance = await this.repository.findById(instance.id);

    if (this.definition.onComplete) {
      try {
        await this.definition.onComplete(context);
      } catch (error) {
        this.logger.error('onComplete hook failed', { error });
      }
    }

    this.logger.info(`Saga completed successfully`, {
      sagaId: instance.id,
      correlationId: context.correlationId,
    });

    return finalInstance!;
  }

  private async compensate(
    instance: SagaInstance,
    context: TContext,
    fromStep: number
  ): Promise<SagaInstance> {
    const { steps } = this.definition;

    this.logger.warn(`Starting compensation from step ${fromStep}`, {
      sagaId: instance.id,
    });

    await this.repository.updateSagaStatus(
      instance.id,
      'COMPENSATING',
      fromStep,
      context
    );

    // ทำ Compensation ย้อนกลับ
    for (let i = fromStep; i >= 0; i--) {
      const step = steps[i];

      this.logger.info(`Compensating step ${i}: ${step.name}`, {
        sagaId: instance.id,
      });

      await this.repository.logStep(
        instance.id,
        i,
        step.name,
        'COMPENSATE',
        'RUNNING',
        context
      );

      const startTime = Date.now();

      try {
        const result = await this.executeWithTimeoutAndRetry(
          () => step.compensate(context),
          step.timeout ?? DEFAULT_STEP_TIMEOUT_MS,
          step.retryPolicy ?? DEFAULT_RETRY_POLICY
        );

        const duration = Date.now() - startTime;

        if (!result.success) {
          // Compensation ล้มเหลว — บันทึกและดำเนินการต่อ (ต้อง Manual intervention)
          await this.repository.logStep(
            instance.id,
            i,
            step.name,
            'COMPENSATE',
            'FAILED',
            context,
            undefined,
            { message: 'Compensation returned failure' },
            duration
          );
          this.logger.error(
            `Compensation for step ${i} (${step.name}) failed — requires manual intervention`,
            { sagaId: instance.id }
          );
        } else {
          await this.repository.logStep(
            instance.id,
            i,
            step.name,
            'COMPENSATE',
            'COMPLETED',
            context,
            result.data,
            undefined,
            duration
          );
        }
      } catch (error) {
        const duration = Date.now() - startTime;
        const err = error as Error;

        await this.repository.logStep(
          instance.id,
          i,
          step.name,
          'COMPENSATE',
          'FAILED',
          context,
          undefined,
          { message: err.message, stack: err.stack },
          duration
        );

        this.logger.error(
          `Compensation step ${i} threw exception`,
          { error: err, sagaId: instance.id }
        );
        // ดำเนินการ compensate ต่อแม้จะ error
      }
    }

    await this.repository.updateSagaStatus(
      instance.id,
      'COMPENSATED',
      0,
      context
    );

    const finalInstance = await this.repository.findById(instance.id);

    if (this.definition.onCompensated) {
      try {
        await this.definition.onCompensated(context);
      } catch (error) {
        this.logger.error('onCompensated hook failed', { error });
      }
    }

    return finalInstance!;
  }

  private async executeWithTimeoutAndRetry(
    fn: () => Promise<StepResult>,
    timeoutMs: number,
    retryPolicy: RetryPolicy,
    onRetry?: (attempt: number) => void
  ): Promise<StepResult> {
    let lastError: Error | undefined;
    let backoffMs = retryPolicy.backoffMs;

    for (let attempt = 1; attempt <= retryPolicy.maxAttempts; attempt++) {
      try {
        const result = await Promise.race([
          fn(),
          new Promise<never>((_, reject) =>
            setTimeout(
              () => reject(new Error(`Step timed out after ${timeoutMs}ms`)),
              timeoutMs
            )
          ),
        ]);

        return result;
      } catch (error) {
        lastError = error as Error;

        if (attempt < retryPolicy.maxAttempts) {
          this.logger.warn(
            `Attempt ${attempt} failed, retrying in ${backoffMs}ms`,
            { error: lastError }
          );

          onRetry?.(attempt);

          await sleep(backoffMs);
          backoffMs = Math.min(
            backoffMs * retryPolicy.backoffMultiplier,
            retryPolicy.maxBackoffMs
          );
        }
      }
    }

    throw lastError;
  }
}

function sleep(ms: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, ms));
}
```

---

## 7. Create Order Saga Definition

```typescript
// packages/saga-orchestrator/src/sagas/create-order.saga.ts

import axios from 'axios';
import { SagaDefinition, StepResult } from '../types';

export interface CreateOrderContext {
  correlationId: string;
  // Input
  userId: string;
  items: Array<{ productId: string; quantity: number; price: number }>;
  shippingAddress: string;
  paymentMethodId: string;
  // Results from steps
  orderId?: string;
  reservationId?: string;
  paymentId?: string;
  fulfillmentId?: string;
  totalAmount?: number;
}

const ORDER_SERVICE_URL = process.env.ORDER_SERVICE_URL ?? 'http://order-service:3001';
const INVENTORY_SERVICE_URL = process.env.INVENTORY_SERVICE_URL ?? 'http://inventory-service:3002';
const PAYMENT_SERVICE_URL = process.env.PAYMENT_SERVICE_URL ?? 'http://payment-service:3003';
const FULFILLMENT_SERVICE_URL = process.env.FULFILLMENT_SERVICE_URL ?? 'http://fulfillment-service:3004';

export const createOrderSaga: SagaDefinition<CreateOrderContext> = {
  name: 'CreateOrder',

  steps: [
    {
      name: 'CreateOrder',
      timeout: 10000,
      retryPolicy: {
        maxAttempts: 3,
        backoffMs: 500,
        backoffMultiplier: 2,
        maxBackoffMs: 5000,
      },
      async execute(ctx): Promise<StepResult> {
        try {
          const totalAmount = ctx.items.reduce(
            (sum, item) => sum + item.price * item.quantity,
            0
          );

          const response = await axios.post(`${ORDER_SERVICE_URL}/orders`, {
            correlationId: ctx.correlationId,
            userId: ctx.userId,
            items: ctx.items,
            totalAmount,
            status: 'PENDING',
          });

          return {
            success: true,
            data: {
              orderId: response.data.orderId,
              totalAmount,
            },
          };
        } catch (error) {
          return { success: false, error: error as Error };
        }
      },

      async compensate(ctx): Promise<StepResult> {
        if (!ctx.orderId) return { success: true }; // Nothing to compensate

        try {
          await axios.post(`${ORDER_SERVICE_URL}/orders/${ctx.orderId}/cancel`, {
            reason: 'Saga compensation',
            correlationId: ctx.correlationId,
          });

          return { success: true };
        } catch (error) {
          return { success: false, error: error as Error };
        }
      },
    },

    {
      name: 'ReserveInventory',
      timeout: 15000,
      retryPolicy: {
        maxAttempts: 3,
        backoffMs: 1000,
        backoffMultiplier: 2,
        maxBackoffMs: 10000,
      },
      async execute(ctx): Promise<StepResult> {
        try {
          const response = await axios.post(`${INVENTORY_SERVICE_URL}/reservations`, {
            correlationId: ctx.correlationId,
            orderId: ctx.orderId,
            items: ctx.items,
          });

          return {
            success: true,
            data: { reservationId: response.data.reservationId },
          };
        } catch (error) {
          const axiosError = error as { response?: { status: number } };
          if (axiosError.response?.status === 409) {
            // Out of stock — ไม่ retry
            return {
              success: false,
              error: new Error('Insufficient stock for one or more items'),
            };
          }
          return { success: false, error: error as Error };
        }
      },

      async compensate(ctx): Promise<StepResult> {
        if (!ctx.reservationId) return { success: true };

        try {
          await axios.delete(
            `${INVENTORY_SERVICE_URL}/reservations/${ctx.reservationId}`,
            { data: { correlationId: ctx.correlationId } }
          );

          return { success: true };
        } catch (error) {
          return { success: false, error: error as Error };
        }
      },
    },

    {
      name: 'ProcessPayment',
      timeout: 20000,
      retryPolicy: {
        maxAttempts: 2, // Payment retry ต้องระวัง double charge
        backoffMs: 2000,
        backoffMultiplier: 2,
        maxBackoffMs: 10000,
      },
      async execute(ctx): Promise<StepResult> {
        try {
          const response = await axios.post(`${PAYMENT_SERVICE_URL}/charges`, {
            correlationId: ctx.correlationId, // Idempotency key
            orderId: ctx.orderId,
            userId: ctx.userId,
            paymentMethodId: ctx.paymentMethodId,
            amount: ctx.totalAmount,
            currency: 'THB',
          });

          return {
            success: true,
            data: { paymentId: response.data.paymentId },
          };
        } catch (error) {
          const axiosError = error as { response?: { status: number; data: { code: string } } };
          // Payment declined — ไม่ควร retry
          if (
            axiosError.response?.status === 402 ||
            axiosError.response?.data?.code === 'PAYMENT_DECLINED'
          ) {
            return {
              success: false,
              error: new Error('Payment declined'),
            };
          }
          return { success: false, error: error as Error };
        }
      },

      async compensate(ctx): Promise<StepResult> {
        if (!ctx.paymentId) return { success: true };

        try {
          await axios.post(`${PAYMENT_SERVICE_URL}/refunds`, {
            correlationId: ctx.correlationId,
            paymentId: ctx.paymentId,
            reason: 'Order saga compensation',
          });

          return { success: true };
        } catch (error) {
          return { success: false, error: error as Error };
        }
      },
    },

    {
      name: 'CreateFulfillment',
      timeout: 10000,
      retryPolicy: {
        maxAttempts: 3,
        backoffMs: 1000,
        backoffMultiplier: 2,
        maxBackoffMs: 15000,
      },
      async execute(ctx): Promise<StepResult> {
        try {
          const response = await axios.post(`${FULFILLMENT_SERVICE_URL}/fulfillments`, {
            correlationId: ctx.correlationId,
            orderId: ctx.orderId,
            reservationId: ctx.reservationId,
            shippingAddress: ctx.shippingAddress,
            items: ctx.items,
          });

          return {
            success: true,
            data: { fulfillmentId: response.data.fulfillmentId },
          };
        } catch (error) {
          return { success: false, error: error as Error };
        }
      },

      async compensate(ctx): Promise<StepResult> {
        if (!ctx.fulfillmentId) return { success: true };

        try {
          await axios.post(
            `${FULFILLMENT_SERVICE_URL}/fulfillments/${ctx.fulfillmentId}/cancel`,
            { correlationId: ctx.correlationId }
          );

          return { success: true };
        } catch (error) {
          return { success: false, error: error as Error };
        }
      },
    },
  ],

  async onComplete(ctx) {
    console.log(`[Saga] Order ${ctx.orderId} created successfully`, {
      orderId: ctx.orderId,
      paymentId: ctx.paymentId,
      fulfillmentId: ctx.fulfillmentId,
    });
  },

  async onCompensated(ctx) {
    console.log(`[Saga] Order ${ctx.orderId} saga compensated`, {
      orderId: ctx.orderId,
    });
  },
};
```

---

## 8. Choreography-based Saga ด้วย RabbitMQ

```typescript
// packages/choreography/src/choreography.saga.ts

import amqp, { Channel, Connection } from 'amqplib';

interface Event {
  type: string;
  correlationId: string;
  timestamp: string;
  payload: Record<string, unknown>;
}

// Base class สำหรับ Choreography Participant
abstract class SagaParticipant {
  protected channel!: Channel;
  protected connection!: Connection;

  abstract get serviceName(): string;
  abstract get listenEvents(): string[];
  abstract handleEvent(event: Event): Promise<void>;

  async connect(amqpUrl: string): Promise<void> {
    this.connection = await amqp.connect(amqpUrl);
    this.channel = await this.connection.createChannel();

    // Exchange สำหรับ Saga Events
    await this.channel.assertExchange('saga.events', 'topic', { durable: true });

    // Queue ของ Service นี้
    const queueName = `${this.serviceName}.saga.queue`;
    await this.channel.assertQueue(queueName, {
      durable: true,
      arguments: {
        'x-dead-letter-exchange': 'saga.events.dlx',
        'x-dead-letter-routing-key': `${this.serviceName}.dead`,
      },
    });

    // Bind queue กับ events ที่ต้องการ
    for (const eventType of this.listenEvents) {
      await this.channel.bindQueue(queueName, 'saga.events', eventType);
    }

    // DLX setup
    await this.channel.assertExchange('saga.events.dlx', 'topic', { durable: true });

    this.channel.prefetch(10);
    await this.channel.consume(queueName, async (msg) => {
      if (!msg) return;

      try {
        const event: Event = JSON.parse(msg.content.toString());
        await this.handleEvent(event);
        this.channel.ack(msg);
      } catch (error) {
        console.error(`[${this.serviceName}] Error handling event:`, error);
        // Nack และส่งไป DLQ หลังจาก 3 ครั้ง
        const retryCount = (msg.properties.headers?.['x-retry-count'] ?? 0) as number;
        if (retryCount < 3) {
          this.channel.nack(msg, false, false); // requeue=false → ไป DLX
        } else {
          this.channel.nack(msg, false, false);
        }
      }
    });

    console.log(`[${this.serviceName}] Connected and listening for events`);
  }

  protected async publishEvent(eventType: string, payload: Record<string, unknown>, correlationId: string): Promise<void> {
    const event: Event = {
      type: eventType,
      correlationId,
      timestamp: new Date().toISOString(),
      payload,
    };

    this.channel.publish(
      'saga.events',
      eventType,
      Buffer.from(JSON.stringify(event)),
      {
        persistent: true,
        contentType: 'application/json',
        headers: {
          'x-origin-service': this.serviceName,
        },
      }
    );
  }
}

// InventoryService เป็น Choreography Participant
export class InventoryChoreographyParticipant extends SagaParticipant {
  get serviceName() { return 'inventory-service'; }
  get listenEvents() {
    return ['order.created', 'payment.failed', 'saga.order.cancel'];
  }

  async handleEvent(event: Event): Promise<void> {
    switch (event.type) {
      case 'order.created':
        await this.handleOrderCreated(event);
        break;
      case 'payment.failed':
      case 'saga.order.cancel':
        await this.handleReleaseStock(event);
        break;
    }
  }

  private async handleOrderCreated(event: Event): Promise<void> {
    const { orderId, items } = event.payload as {
      orderId: string;
      items: Array<{ productId: string; quantity: number }>;
    };

    try {
      // ลอง Reserve Stock
      const reservationId = await this.reserveStock(items);

      await this.publishEvent('inventory.reserved', {
        orderId,
        reservationId,
        items,
      }, event.correlationId);

    } catch (error) {
      await this.publishEvent('inventory.reservation.failed', {
        orderId,
        reason: (error as Error).message,
      }, event.correlationId);
    }
  }

  private async handleReleaseStock(event: Event): Promise<void> {
    const { reservationId } = event.payload as { reservationId: string };
    if (!reservationId) return;

    await this.releaseStock(reservationId);

    await this.publishEvent('inventory.released', {
      reservationId,
    }, event.correlationId);
  }

  private async reserveStock(
    items: Array<{ productId: string; quantity: number }>
  ): Promise<string> {
    // TODO: Implement actual stock reservation logic
    console.log('Reserving stock for items:', items);
    return `reservation-${Date.now()}`;
  }

  private async releaseStock(reservationId: string): Promise<void> {
    console.log('Releasing stock for reservation:', reservationId);
  }
}
```

---

## 9. Saga HTTP API

```typescript
// packages/saga-orchestrator/src/api.ts

import express from 'express';
import { Pool } from 'pg';
import { SagaEngine } from './saga.engine';
import { SagaLogRepository } from './saga.log';
import { CreateOrderContext, createOrderSaga } from './sagas/create-order.saga';

export function createApp(pool: Pool) {
  const app = express();
  app.use(express.json());

  const repository = new SagaLogRepository(pool);

  // POST /sagas/orders — เริ่ม Create Order Saga
  app.post('/sagas/orders', async (req, res) => {
    const {
      correlationId,
      userId,
      items,
      shippingAddress,
      paymentMethodId,
    } = req.body;

    if (!correlationId || !userId || !items?.length || !shippingAddress || !paymentMethodId) {
      return res.status(400).json({ error: 'Missing required fields' });
    }

    const engine = new SagaEngine<CreateOrderContext>(createOrderSaga, repository);

    const context: CreateOrderContext = {
      correlationId,
      userId,
      items,
      shippingAddress,
      paymentMethodId,
    };

    try {
      const instance = await engine.execute(context);

      return res.status(202).json({
        sagaId: instance.id,
        correlationId: instance.correlationId,
        status: instance.status,
        orderId: (instance.context as CreateOrderContext).orderId,
      });
    } catch (error) {
      return res.status(500).json({
        error: 'Saga execution failed',
        message: (error as Error).message,
      });
    }
  });

  // GET /sagas/:correlationId — ดูสถานะ Saga
  app.get('/sagas/:correlationId', async (req, res) => {
    const instance = await repository.findByCorrelationId(req.params.correlationId);

    if (!instance) {
      return res.status(404).json({ error: 'Saga not found' });
    }

    return res.json(instance);
  });

  // GET /sagas/stuck — ดู Saga ที่ค้าง
  app.get('/admin/sagas/stuck', async (_req, res) => {
    const stuckSagas = await repository.findStuckSagas();
    return res.json({ count: stuckSagas.length, sagas: stuckSagas });
  });

  return app;
}

// Main entry point
async function main() {
  const pool = new Pool({
    host: process.env.DB_HOST ?? 'localhost',
    port: parseInt(process.env.DB_PORT ?? '5432'),
    database: process.env.DB_NAME ?? 'saga_db',
    user: process.env.DB_USER ?? 'postgres',
    password: process.env.DB_PASSWORD ?? 'postgres',
    max: 20,
  });

  const app = createApp(pool);
  const port = parseInt(process.env.PORT ?? '3000');

  app.listen(port, () => {
    console.log(`Saga Orchestrator running on port ${port}`);
  });

  // Periodic check สำหรับ stuck sagas
  setInterval(async () => {
    const repository = new SagaLogRepository(pool);
    const stuck = await repository.findStuckSagas();
    if (stuck.length > 0) {
      console.warn(`[Alert] Found ${stuck.length} stuck sagas`);
    }
  }, 60000);
}

main().catch(console.error);
```

---

## 10. Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: saga_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./db/saga-schema.sql:/docker-entrypoint-initdb.d/01-schema.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  rabbitmq:
    image: rabbitmq:3.13-management-alpine
    environment:
      RABBITMQ_DEFAULT_USER: guest
      RABBITMQ_DEFAULT_PASS: guest
    ports:
      - "5672:5672"
      - "15672:15672"
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  saga-orchestrator:
    build:
      context: ./packages/saga-orchestrator
      dockerfile: Dockerfile
    environment:
      DB_HOST: postgres
      DB_PORT: 5432
      DB_NAME: saga_db
      DB_USER: postgres
      DB_PASSWORD: postgres
      ORDER_SERVICE_URL: http://order-service:3001
      INVENTORY_SERVICE_URL: http://inventory-service:3002
      PAYMENT_SERVICE_URL: http://payment-service:3003
      FULFILLMENT_SERVICE_URL: http://fulfillment-service:3004
      PORT: 3000
    ports:
      - "3000:3000"
    depends_on:
      postgres:
        condition: service_healthy

  order-service:
    build:
      context: ./packages/order-service
      dockerfile: Dockerfile
    environment:
      DB_HOST: postgres
      PORT: 3001
    ports:
      - "3001:3001"
    depends_on:
      postgres:
        condition: service_healthy

  inventory-service:
    build:
      context: ./packages/inventory-service
      dockerfile: Dockerfile
    environment:
      DB_HOST: postgres
      RABBITMQ_URL: amqp://guest:guest@rabbitmq:5672
      PORT: 3002
    ports:
      - "3002:3002"
    depends_on:
      postgres:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy

  payment-service:
    build:
      context: ./packages/payment-service
      dockerfile: Dockerfile
    environment:
      DB_HOST: postgres
      STRIPE_SECRET_KEY: ${STRIPE_SECRET_KEY}
      PORT: 3003
    ports:
      - "3003:3003"

  fulfillment-service:
    build:
      context: ./packages/fulfillment-service
      dockerfile: Dockerfile
    environment:
      DB_HOST: postgres
      PORT: 3004
    ports:
      - "3004:3004"

volumes:
  postgres_data:
```

---

## 11. Saga Recovery Job

```typescript
// packages/saga-orchestrator/src/recovery.job.ts
// งาน Cron ที่ตรวจหา Stuck Sagas และลอง Resume

import { Pool } from 'pg';
import { SagaLogRepository } from './saga.log';
import { SagaEngine } from './saga.engine';
import { CreateOrderContext, createOrderSaga } from './sagas/create-order.saga';

export class SagaRecoveryJob {
  private readonly repository: SagaLogRepository;

  constructor(private readonly pool: Pool) {
    this.repository = new SagaLogRepository(pool);
  }

  async run(): Promise<void> {
    console.log('[Recovery] Checking for stuck sagas...');

    const stuckSagas = await this.repository.findStuckSagas();

    if (stuckSagas.length === 0) {
      console.log('[Recovery] No stuck sagas found');
      return;
    }

    console.log(`[Recovery] Found ${stuckSagas.length} stuck sagas`);

    for (const saga of stuckSagas) {
      try {
        await this.recoverSaga(saga.correlationId, saga.sagaType, saga.context as CreateOrderContext);
      } catch (error) {
        console.error(`[Recovery] Failed to recover saga ${saga.id}:`, error);
      }
    }
  }

  private async recoverSaga(
    correlationId: string,
    sagaType: string,
    context: CreateOrderContext
  ): Promise<void> {
    console.log(`[Recovery] Attempting to recover saga: ${correlationId} (${sagaType})`);

    switch (sagaType) {
      case 'CreateOrder': {
        const engine = new SagaEngine<CreateOrderContext>(
          createOrderSaga,
          this.repository
        );
        await engine.execute(context);
        break;
      }
      default:
        console.warn(`[Recovery] Unknown saga type: ${sagaType}`);
    }
  }
}

// Schedule recovery job
if (require.main === module) {
  const pool = new Pool({
    host: process.env.DB_HOST ?? 'localhost',
    database: process.env.DB_NAME ?? 'saga_db',
    user: process.env.DB_USER ?? 'postgres',
    password: process.env.DB_PASSWORD ?? 'postgres',
  });

  const job = new SagaRecoveryJob(pool);

  // รันทุก 5 นาที
  setInterval(() => {
    job.run().catch(console.error);
  }, 5 * 60 * 1000);

  // รันทันทีเมื่อเริ่ม
  job.run().catch(console.error);
}
```

---

## 12. Testing Saga

```typescript
// packages/saga-orchestrator/src/__tests__/create-order.saga.test.ts

import { Pool } from 'pg';
import axios from 'axios';
import MockAdapter from 'axios-mock-adapter';
import { SagaEngine } from '../saga.engine';
import { SagaLogRepository } from '../saga.log';
import { CreateOrderContext, createOrderSaga } from '../sagas/create-order.saga';

const mock = new MockAdapter(axios);

describe('CreateOrder Saga', () => {
  let pool: Pool;
  let repository: SagaLogRepository;
  let engine: SagaEngine<CreateOrderContext>;

  beforeAll(async () => {
    pool = new Pool({
      host: 'localhost',
      database: 'saga_test',
      user: 'postgres',
      password: 'postgres',
    });
    repository = new SagaLogRepository(pool);
    engine = new SagaEngine(createOrderSaga, repository);
  });

  afterAll(async () => {
    await pool.end();
  });

  beforeEach(() => {
    mock.reset();
  });

  it('should complete successfully when all services respond correctly', async () => {
    const correlationId = `test-${Date.now()}`;

    mock.onPost('/orders').reply(201, { orderId: 'order-123' });
    mock.onPost('/reservations').reply(201, { reservationId: 'res-456' });
    mock.onPost('/charges').reply(201, { paymentId: 'pay-789' });
    mock.onPost('/fulfillments').reply(201, { fulfillmentId: 'ful-012' });

    const context: CreateOrderContext = {
      correlationId,
      userId: 'user-1',
      items: [{ productId: 'prod-1', quantity: 2, price: 100 }],
      shippingAddress: '123 Main St',
      paymentMethodId: 'pm-1',
    };

    const result = await engine.execute(context);

    expect(result.status).toBe('COMPLETED');
    expect((result.context as CreateOrderContext).orderId).toBe('order-123');
    expect((result.context as CreateOrderContext).paymentId).toBe('pay-789');
  });

  it('should compensate when payment fails', async () => {
    const correlationId = `test-fail-${Date.now()}`;

    mock.onPost('/orders').reply(201, { orderId: 'order-999' });
    mock.onPost('/reservations').reply(201, { reservationId: 'res-999' });
    mock.onPost('/charges').reply(402, { code: 'PAYMENT_DECLINED' });
    mock.onPost('/orders/order-999/cancel').reply(200);
    mock.onDelete('/reservations/res-999').reply(200);

    const context: CreateOrderContext = {
      correlationId,
      userId: 'user-2',
      items: [{ productId: 'prod-2', quantity: 1, price: 500 }],
      shippingAddress: '456 Other St',
      paymentMethodId: 'pm-declined',
    };

    const result = await engine.execute(context);

    expect(result.status).toBe('COMPENSATED');
    // ตรวจสอบว่า cancel order ถูกเรียก
    expect(mock.history.post.some(r => r.url?.includes('/cancel'))).toBe(true);
    // ตรวจสอบว่า release stock ถูกเรียก
    expect(mock.history.delete.some(r => r.url?.includes('reservations'))).toBe(true);
  });

  it('should be idempotent — running same correlationId twice returns same result', async () => {
    const correlationId = `idempotent-${Date.now()}`;

    mock.onPost('/orders').reply(201, { orderId: 'order-idem' });
    mock.onPost('/reservations').reply(201, { reservationId: 'res-idem' });
    mock.onPost('/charges').reply(201, { paymentId: 'pay-idem' });
    mock.onPost('/fulfillments').reply(201, { fulfillmentId: 'ful-idem' });

    const context: CreateOrderContext = {
      correlationId,
      userId: 'user-3',
      items: [{ productId: 'prod-3', quantity: 1, price: 200 }],
      shippingAddress: '789 Another Ave',
      paymentMethodId: 'pm-2',
    };

    const first = await engine.execute(context);
    const second = await engine.execute(context);

    expect(first.id).toBe(second.id);
    expect(second.status).toBe('COMPLETED');
  });
});
```

---

## 13. Logger Utility

```typescript
// packages/saga-orchestrator/src/logger.ts

export class Logger {
  constructor(private readonly context: string) {}

  info(message: string, meta?: Record<string, unknown>): void {
    console.log(JSON.stringify({
      level: 'info',
      context: this.context,
      message,
      ...meta,
      timestamp: new Date().toISOString(),
    }));
  }

  warn(message: string, meta?: Record<string, unknown>): void {
    console.warn(JSON.stringify({
      level: 'warn',
      context: this.context,
      message,
      ...meta,
      timestamp: new Date().toISOString(),
    }));
  }

  error(message: string, meta?: Record<string, unknown>): void {
    console.error(JSON.stringify({
      level: 'error',
      context: this.context,
      message,
      ...meta,
      timestamp: new Date().toISOString(),
    }));
  }
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Choreography vs Orchestration** — Choreography เหมาะกับ simple flow ที่ไม่มี complex logic, Orchestration เหมาะกับ business process ที่ซับซ้อนและต้องการ central visibility

2. **Compensation Transactions** — ทุก Step ต้องมี Compensate function ที่สามารถ undo การกระทำได้ โดยต้องออกแบบให้เป็น Idempotent

3. **Saga Log ใน PostgreSQL** — บันทึกทุก Step เพื่อให้ Resume ได้และ Debug ได้ง่าย

4. **Timeout Handling** — แต่ละ Step มี timeout และ retry policy ที่กำหนดได้เพื่อป้องกัน saga ค้าง

5. **Recovery Job** — Background job ตรวจหา stuck sagas และลอง resume อัตโนมัติ

6. **Idempotency** — ใช้ correlationId เพื่อให้การรัน saga ซ้ำไม่ก่อให้เกิด side effects

**Key Principle:** Saga ไม่ใช่ silver bullet — ต้องออกแบบ Compensation ให้ดี โดยเฉพาะกรณีที่ Compensation ล้มเหลวซึ่งต้องการ Manual Intervention

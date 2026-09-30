# Part 23: Saga Pattern - Distributed Transaction Management

## ภาพรวม

ใน Part นี้เราจะเรียนรู้ Saga Pattern อย่างละเอียด:
- **Choreography Saga** - Services communicate via events
- **Orchestration Saga** - Central orchestrator controls flow
- **Compensation Transactions** - ยกเลิกเมื่อเกิดข้อผิดพลาด
- **Idempotency** - ทำซ้ำได้อย่างปลอดภัย
- **Saga State Machine** - Track saga progress

---

## 1. ทำไมต้องใช้ Saga?

```
ปัญหาของ Distributed Transactions:
- ไม่สามารถใช้ ACID transaction ข้ามหลาย services ได้
- 2PC (Two-Phase Commit) ทำให้ระบบช้าและ availability ต่ำ

Saga Solution:
- แบ่ง transaction ใหญ่เป็น local transactions เล็กๆ
- แต่ละ step มี compensating transaction
- เมื่อเกิดข้อผิดพลาด ให้ rollback ผ่าน compensations

Order Creation Saga:
Step 1: Create Order (order-service)
Step 2: Reserve Inventory (inventory-service)  
Step 3: Process Payment (payment-service)
Step 4: Send Confirmation (notification-service)

If Step 3 fails:
→ Compensate Step 2: Release Inventory
→ Compensate Step 1: Cancel Order
```

---

## 2. Choreography Saga

```javascript
// sagas/order-creation-choreography.js

// ===== Order Service =====
class OrderService {
  async createOrder(data) {
    const order = await this.orderRepo.create({
      ...data,
      status: 'pending'
    });

    // Publish event to start saga
    await this.eventBus.publish({
      type: 'OrderCreated',
      orderId: order.id,
      userId: order.userId,
      items: order.items,
      totalAmount: order.totalAmount
    });

    return order;
  }

  // Listen for success/failure events
  async handleInventoryReserved(event) {
    await this.orderRepo.updateStatus(event.orderId, 'inventory_reserved');
  }

  async handleInventoryReservationFailed(event) {
    // Compensate: cancel the order
    await this.orderRepo.updateStatus(event.orderId, 'cancelled');
    await this.eventBus.publish({
      type: 'OrderCancelled',
      orderId: event.orderId,
      reason: 'inventory_unavailable'
    });
  }

  async handlePaymentProcessed(event) {
    await this.orderRepo.updateStatus(event.orderId, 'paid');
  }

  async handlePaymentFailed(event) {
    // Compensate: cancel order + release inventory
    await this.orderRepo.updateStatus(event.orderId, 'payment_failed');
    await this.eventBus.publish({
      type: 'OrderPaymentFailed',
      orderId: event.orderId,
      // This triggers inventory service to release reservation
    });
  }
}

// ===== Inventory Service =====
class InventoryService {
  async handleOrderCreated(event) {
    try {
      // Reserve items
      await this.reserveItems(event.orderId, event.items);
      
      await this.eventBus.publish({
        type: 'InventoryReserved',
        orderId: event.orderId,
        items: event.items
      });
    } catch (err) {
      await this.eventBus.publish({
        type: 'InventoryReservationFailed',
        orderId: event.orderId,
        reason: err.message
      });
    }
  }

  async handleOrderPaymentFailed(event) {
    // Compensate: release reservation
    await this.releaseReservation(event.orderId);
    
    await this.eventBus.publish({
      type: 'InventoryReleased',
      orderId: event.orderId
    });
  }

  async handleOrderCancelled(event) {
    await this.releaseReservation(event.orderId);
  }

  async reserveItems(orderId, items) {
    for (const item of items) {
      const result = await this.db.query(
        `UPDATE products 
         SET reserved_stock = reserved_stock + $1
         WHERE id = $2 AND (available_stock - reserved_stock) >= $1
         RETURNING id`,
        [item.quantity, item.productId]
      );

      if (result.rows.length === 0) {
        // Rollback previous reservations in this saga step
        await this.releasePartialReservation(orderId, items, item);
        throw new Error(`Insufficient stock for product ${item.productId}`);
      }

      // Track reservation
      await this.db.query(
        `INSERT INTO inventory_reservations (order_id, product_id, quantity)
         VALUES ($1, $2, $3)
         ON CONFLICT (order_id, product_id) DO UPDATE SET quantity = $3`,
        [orderId, item.productId, item.quantity]
      );
    }
  }

  async releaseReservation(orderId) {
    const reservations = await this.db.query(
      'SELECT * FROM inventory_reservations WHERE order_id = $1',
      [orderId]
    );

    for (const res of reservations.rows) {
      await this.db.query(
        `UPDATE products 
         SET reserved_stock = reserved_stock - $1
         WHERE id = $2`,
        [res.quantity, res.product_id]
      );
    }

    await this.db.query(
      'DELETE FROM inventory_reservations WHERE order_id = $1',
      [orderId]
    );
  }
}

// ===== Payment Service =====
class PaymentService {
  async handleInventoryReserved(event) {
    try {
      const order = await this.orderService.getOrder(event.orderId);
      const payment = await this.processPayment(order);
      
      await this.eventBus.publish({
        type: 'PaymentProcessed',
        orderId: event.orderId,
        paymentId: payment.id,
        amount: payment.amount
      });
    } catch (err) {
      await this.eventBus.publish({
        type: 'PaymentFailed',
        orderId: event.orderId,
        reason: err.message
      });
    }
  }
}
```

---

## 3. Orchestration Saga

### 3.1 Saga Orchestrator

```javascript
// sagas/order-saga-orchestrator.js
const { v4: uuidv4 } = require('uuid');

class OrderSagaOrchestrator {
  constructor(services, sagaRepository) {
    this.services = services;
    this.sagaRepo = sagaRepository;
    this.steps = this.defineSteps();
  }

  defineSteps() {
    return [
      {
        name: 'CreateOrder',
        execute: async (ctx) => {
          return this.services.orderService.createOrder({
            userId: ctx.userId,
            items: ctx.items,
            shippingAddress: ctx.shippingAddress
          });
        },
        compensate: async (ctx) => {
          await this.services.orderService.cancelOrder(ctx.orderId, 'saga_rollback');
        }
      },
      {
        name: 'ReserveInventory',
        execute: async (ctx) => {
          return this.services.inventoryService.reserveItems(ctx.orderId, ctx.items);
        },
        compensate: async (ctx) => {
          await this.services.inventoryService.releaseReservation(ctx.orderId);
        }
      },
      {
        name: 'ProcessPayment',
        execute: async (ctx) => {
          return this.services.paymentService.charge({
            orderId: ctx.orderId,
            userId: ctx.userId,
            amount: ctx.totalAmount,
            currency: ctx.currency,
            paymentMethodId: ctx.paymentMethodId
          });
        },
        compensate: async (ctx) => {
          if (ctx.paymentId) {
            await this.services.paymentService.refund({
              paymentId: ctx.paymentId,
              amount: ctx.totalAmount,
              reason: 'order_cancelled'
            });
          }
        }
      },
      {
        name: 'ConfirmOrder',
        execute: async (ctx) => {
          return this.services.orderService.confirmOrder(ctx.orderId);
        },
        compensate: async (ctx) => {
          // Order confirmation doesn't need compensation (already cancelling)
        }
      },
      {
        name: 'SendNotification',
        execute: async (ctx) => {
          return this.services.notificationService.sendOrderConfirmation({
            userId: ctx.userId,
            orderId: ctx.orderId,
            items: ctx.items
          });
        },
        compensate: async (ctx) => {
          // Optionally send cancellation notification
          await this.services.notificationService.sendOrderCancellation({
            userId: ctx.userId,
            orderId: ctx.orderId
          });
        }
      }
    ];
  }

  async start(input) {
    const sagaId = uuidv4();
    const context = {
      sagaId,
      ...input,
      completedSteps: [],
      status: 'running'
    };

    // Persist saga state
    await this.sagaRepo.create(sagaId, 'order_creation', context);

    try {
      for (let i = 0; i < this.steps.length; i++) {
        const step = this.steps[i];
        
        await this.sagaRepo.updateStep(sagaId, step.name, 'running');
        
        try {
          const result = await step.execute(context);
          
          // Merge result into context
          Object.assign(context, result);
          context.completedSteps.push(step.name);
          
          await this.sagaRepo.updateStep(sagaId, step.name, 'completed', result);
        } catch (stepError) {
          await this.sagaRepo.updateStep(sagaId, step.name, 'failed', {
            error: stepError.message
          });
          
          // Compensate
          await this.compensate(sagaId, context, i - 1);
          
          await this.sagaRepo.updateStatus(sagaId, 'failed');
          throw new SagaError(sagaId, step.name, stepError);
        }
      }

      await this.sagaRepo.updateStatus(sagaId, 'completed');
      return { sagaId, ...context };

    } catch (err) {
      if (err instanceof SagaError) throw err;
      
      await this.sagaRepo.updateStatus(sagaId, 'failed');
      throw err;
    }
  }

  async compensate(sagaId, context, fromStepIndex) {
    console.log(`Compensating saga ${sagaId} from step ${fromStepIndex}`);
    
    // Execute compensations in reverse order
    for (let i = fromStepIndex; i >= 0; i--) {
      const step = this.steps[i];
      
      if (!context.completedSteps.includes(step.name)) continue;
      
      try {
        await this.sagaRepo.updateStep(sagaId, step.name, 'compensating');
        await step.compensate(context);
        await this.sagaRepo.updateStep(sagaId, step.name, 'compensated');
      } catch (compensationError) {
        console.error(
          `Compensation failed for step ${step.name}:`,
          compensationError
        );
        
        // Log for manual intervention
        await this.sagaRepo.updateStep(sagaId, step.name, 'compensation_failed', {
          error: compensationError.message
        });
        
        // Notify ops team
        await this.notifyCompensationFailure(sagaId, step.name, compensationError);
        
        // Continue with other compensations
      }
    }
  }

  async resume(sagaId) {
    const saga = await this.sagaRepo.findById(sagaId);
    
    if (!saga) throw new Error('Saga not found');
    if (saga.status === 'completed') return saga.context;
    if (saga.status !== 'running') throw new Error(`Cannot resume saga in ${saga.status} status`);

    const lastCompletedIndex = this.steps.findIndex(
      s => !saga.context.completedSteps.includes(s.name)
    );

    // Resume from where we left off
    return this.start({ ...saga.context, _resumeFrom: lastCompletedIndex });
  }
}

class SagaError extends Error {
  constructor(sagaId, stepName, cause) {
    super(`Saga ${sagaId} failed at step ${stepName}: ${cause.message}`);
    this.sagaId = sagaId;
    this.stepName = stepName;
    this.cause = cause;
  }
}
```

### 3.2 Saga State Repository

```javascript
// repositories/saga-repository.js
class SagaRepository {
  constructor(db) {
    this.db = db;
  }

  async create(sagaId, sagaType, context) {
    await this.db.query(
      `INSERT INTO sagas (id, type, status, context, steps, created_at)
       VALUES ($1, $2, 'running', $3, $4, NOW())`,
      [sagaId, sagaType, JSON.stringify(context), JSON.stringify({})]
    );
  }

  async findById(sagaId) {
    const result = await this.db.query(
      'SELECT * FROM sagas WHERE id = $1',
      [sagaId]
    );
    
    if (result.rows.length === 0) return null;
    
    const row = result.rows[0];
    return {
      id: row.id,
      type: row.type,
      status: row.status,
      context: row.context,
      steps: row.steps,
      createdAt: row.created_at,
      updatedAt: row.updated_at
    };
  }

  async updateStatus(sagaId, status) {
    await this.db.query(
      'UPDATE sagas SET status = $2, updated_at = NOW() WHERE id = $1',
      [sagaId, status]
    );
  }

  async updateStep(sagaId, stepName, stepStatus, data = {}) {
    await this.db.query(
      `UPDATE sagas
       SET steps = steps || $3::jsonb,
           updated_at = NOW()
       WHERE id = $1`,
      [
        sagaId,
        stepName,
        JSON.stringify({
          [stepName]: {
            status: stepStatus,
            data,
            updatedAt: new Date().toISOString()
          }
        })
      ]
    );
  }

  // Find stuck sagas for recovery
  async findStuckSagas(olderThanMinutes = 30) {
    const result = await this.db.query(
      `SELECT * FROM sagas
       WHERE status = 'running'
         AND updated_at < NOW() - INTERVAL '$1 minutes'
       ORDER BY created_at ASC`,
      [olderThanMinutes]
    );
    
    return result.rows;
  }
}
```

---

## 4. Saga Recovery

```javascript
// sagas/saga-recovery.js
class SagaRecovery {
  constructor(orchestrators, sagaRepository) {
    this.orchestrators = orchestrators;
    this.sagaRepo = sagaRepository;
    
    // Run recovery every 5 minutes
    setInterval(() => this.recoverStuckSagas(), 5 * 60 * 1000);
  }

  async recoverStuckSagas() {
    const stuckSagas = await this.sagaRepo.findStuckSagas(30);
    
    console.log(`Found ${stuckSagas.length} stuck sagas`);

    for (const saga of stuckSagas) {
      try {
        const orchestrator = this.orchestrators[saga.type];
        if (!orchestrator) {
          console.error(`No orchestrator for saga type: ${saga.type}`);
          continue;
        }

        console.log(`Recovering saga: ${saga.id}`);
        await orchestrator.resume(saga.id);
      } catch (err) {
        console.error(`Failed to recover saga ${saga.id}:`, err);
        
        // After max retries, mark as failed
        const retryCount = saga.context.retryCount || 0;
        if (retryCount >= 3) {
          await this.sagaRepo.updateStatus(saga.id, 'failed');
          await this.escalateForManualReview(saga);
        } else {
          await this.sagaRepo.updateContext(saga.id, {
            ...saga.context,
            retryCount: retryCount + 1
          });
        }
      }
    }
  }

  async escalateForManualReview(saga) {
    // Send to ops team
    await alerting.send('critical', 'Saga requires manual intervention', {
      sagaId: saga.id,
      sagaType: saga.type,
      context: saga.context,
      steps: saga.steps
    });
  }
}
```

---

## 5. Idempotency - ทำซ้ำได้อย่างปลอดภัย

### 5.1 Idempotency Key Pattern

```javascript
// middleware/idempotency.js
class IdempotencyHandler {
  constructor(redis) {
    this.redis = redis;
    this.TTL = 86400; // 24 hours
  }

  middleware() {
    return async (req, res, next) => {
      // Only for mutating operations
      if (!['POST', 'PUT', 'PATCH'].includes(req.method)) {
        return next();
      }

      const idempotencyKey = req.headers['idempotency-key'];
      
      if (!idempotencyKey) {
        // Optional: require idempotency key for certain endpoints
        if (req.requiresIdempotencyKey) {
          return res.status(400).json({
            error: 'Idempotency-Key header required'
          });
        }
        return next();
      }

      // Validate key format (UUID)
      if (!/^[0-9a-f-]{36}$/.test(idempotencyKey)) {
        return res.status(400).json({ error: 'Invalid Idempotency-Key format' });
      }

      const cacheKey = `idempotency:${req.user.id}:${idempotencyKey}`;

      // Check if request was already processed
      const cached = await this.redis.get(cacheKey);
      
      if (cached) {
        const stored = JSON.parse(cached);
        
        // Return cached response
        res.status(stored.status)
          .set('X-Idempotent-Replayed', 'true')
          .json(stored.body);
        return;
      }

      // Mark as processing (prevent concurrent duplicates)
      const acquired = await this.redis.set(
        `${cacheKey}:lock`,
        '1',
        'EX', 30,
        'NX'
      );
      
      if (!acquired) {
        return res.status(409).json({
          error: 'Request is being processed',
          message: 'A request with this idempotency key is already in progress'
        });
      }

      // Intercept response to cache it
      const originalJson = res.json.bind(res);
      res.json = async function(body) {
        if (res.statusCode < 500) {
          // Cache successful and client error responses (not server errors)
          await this.redis.setex(
            cacheKey,
            this.TTL,
            JSON.stringify({ status: res.statusCode, body })
          );
        }
        
        await this.redis.del(`${cacheKey}:lock`);
        return originalJson(body);
      }.bind(this);

      next();
    };
  }
}
```

---

## 6. Workshop: Complete Order Saga

```javascript
// workshop/complete-order-saga.js

// Setup
const sagaRepo = new SagaRepository(db);
const orderOrchestrator = new OrderSagaOrchestrator(
  {
    orderService,
    inventoryService,
    paymentService,
    notificationService
  },
  sagaRepo
);

// API endpoint
app.post('/orders', idempotency.middleware(), async (req, res) => {
  const { items, shippingAddress, paymentMethodId } = req.body;
  
  try {
    const result = await orderOrchestrator.start({
      userId: req.user.id,
      items,
      shippingAddress,
      paymentMethodId,
      totalAmount: calculateTotal(items),
      currency: 'THB'
    });

    res.status(201).json({
      orderId: result.orderId,
      sagaId: result.sagaId,
      status: result.status
    });
  } catch (err) {
    if (err instanceof SagaError) {
      return res.status(422).json({
        error: 'Order creation failed',
        failedStep: err.stepName,
        sagaId: err.sagaId,
        message: err.message
      });
    }
    throw err;
  }
});

// Test the saga
async function testOrderSaga() {
  // Happy path
  const result = await orderOrchestrator.start({
    userId: 'user-1',
    items: [
      { productId: 'prod-1', productName: 'iPhone', quantity: 1, unitPrice: 35000 }
    ],
    shippingAddress: {
      street: '123 Main St',
      city: 'Bangkok',
      postalCode: '10100',
      country: 'TH'
    },
    paymentMethodId: 'pm-1',
    totalAmount: 35000,
    currency: 'THB'
  });

  console.log('Saga completed:', result);

  // Failure path (inventory unavailable)
  try {
    await orderOrchestrator.start({
      userId: 'user-2',
      items: [
        { productId: 'out-of-stock-prod', quantity: 100, unitPrice: 100 }
      ],
      totalAmount: 10000,
      paymentMethodId: 'pm-2'
    });
  } catch (err) {
    console.log('Saga failed (expected):', err.message);
    // Verify compensation was executed
    const order = await orderService.getOrder(err.context?.orderId);
    console.log('Order status after saga failure:', order?.status);
    // Should be 'cancelled'
  }
}
```

---

## สรุป

```
Choreography vs Orchestration:

Choreography:
✓ ไม่มี central point of failure
✓ Services loosely coupled
✓ ง่ายต่อการ add services
✗ ยาก trace ว่า saga อยู่ที่ไหน
✗ ยาก debug เมื่อเกิดปัญหา

Orchestration:
✓ มองเห็นภาพรวมชัดเจน
✓ ง่าย debug และ monitor
✓ ง่าย implement compensation
✗ Orchestrator เป็น central point
✗ Services ต้องรู้จัก orchestrator

Best Practice:
- ใช้ Choreography สำหรับ simple, independent flows
- ใช้ Orchestration สำหรับ complex, multi-step transactions
- เสมอ implement compensation transactions
- เสมอ implement idempotency
- Monitor and alert on stuck sagas
```

**Next:** Part 24 - Multi-tenancy Architecture

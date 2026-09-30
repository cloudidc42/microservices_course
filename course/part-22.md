# Part 22: Event Sourcing & CQRS Deep Dive

## ภาพรวม

ใน Part นี้เราจะเรียนรู้ Event Sourcing และ CQRS อย่างละเอียด:
- **Event Sourcing** - เก็บ state เป็น sequence of events
- **CQRS** - แยก Command และ Query
- **Projections** - สร้าง Read Models จาก Events
- **Snapshots** - ลด replay time
- **Event Store** ด้วย PostgreSQL และ EventStoreDB

---

## 1. Event Sourcing Fundamentals

### 1.1 ทำความเข้าใจ Event Sourcing

```
Traditional State Storage:
User { id: 1, balance: 850, updatedAt: 2024-01-15 }
(เราไม่รู้ว่า balance เป็น 850 ได้อย่างไร)

Event Sourcing:
Event 1: AccountOpened    { balance: 1000 }
Event 2: WithdrawalMade   { amount: 200 }
Event 3: DepositMade      { amount: 100 }
Event 4: WithdrawalMade   { amount: 50 }
→ Current State = 1000 - 200 + 100 - 50 = 850

Benefits:
✓ Complete audit trail
✓ Time travel (replay to any point)
✓ Event-driven integration
✓ Debug production issues easily
✓ Business insights from events

Challenges:
✗ Eventual consistency
✗ Event schema evolution
✗ Query complexity
✗ Replay performance (solved with snapshots)
```

### 1.2 Domain Events

```javascript
// events/domain-events.js

// Base Event
class DomainEvent {
  constructor(aggregateId, data) {
    this.eventId = crypto.randomUUID();
    this.aggregateId = aggregateId;
    this.aggregateType = this.constructor.aggregateType;
    this.eventType = this.constructor.eventType;
    this.data = data;
    this.occurredAt = new Date().toISOString();
    this.version = this.constructor.version || 1;
    this.metadata = {};
  }

  withMetadata(metadata) {
    this.metadata = { ...this.metadata, ...metadata };
    return this;
  }
}

// Order Events
class OrderCreated extends DomainEvent {
  static aggregateType = 'Order';
  static eventType = 'OrderCreated';
  static version = 1;
}

class OrderItemAdded extends DomainEvent {
  static aggregateType = 'Order';
  static eventType = 'OrderItemAdded';
  static version = 1;
}

class OrderItemRemoved extends DomainEvent {
  static aggregateType = 'Order';
  static eventType = 'OrderItemRemoved';
  static version = 1;
}

class OrderSubmitted extends DomainEvent {
  static aggregateType = 'Order';
  static eventType = 'OrderSubmitted';
  static version = 1;
}

class OrderConfirmed extends DomainEvent {
  static aggregateType = 'Order';
  static eventType = 'OrderConfirmed';
  static version = 1;
}

class OrderPaymentReceived extends DomainEvent {
  static aggregateType = 'Order';
  static eventType = 'OrderPaymentReceived';
  static version = 1;
}

class OrderShipped extends DomainEvent {
  static aggregateType = 'Order';
  static eventType = 'OrderShipped';
  static version = 1;
}

class OrderDelivered extends DomainEvent {
  static aggregateType = 'Order';
  static eventType = 'OrderDelivered';
  static version = 1;
}

class OrderCancelled extends DomainEvent {
  static aggregateType = 'Order';
  static eventType = 'OrderCancelled';
  static version = 1;
}

module.exports = {
  DomainEvent,
  OrderCreated,
  OrderItemAdded,
  OrderItemRemoved,
  OrderSubmitted,
  OrderConfirmed,
  OrderPaymentReceived,
  OrderShipped,
  OrderDelivered,
  OrderCancelled
};
```

---

## 2. Aggregate สำหรับ Event Sourcing

### 2.1 AggregateRoot Base Class

```javascript
// domain/aggregate-root.js
class AggregateRoot {
  constructor(id) {
    this.id = id;
    this._version = 0;          // Current version in event store
    this._uncommittedEvents = []; // Events raised but not yet persisted
  }

  // Apply event and track as uncommitted
  raise(event) {
    this._applyEvent(event);
    this._uncommittedEvents.push(event);
  }

  // Apply event to update state (idempotent)
  _applyEvent(event) {
    const handlerName = `on${event.eventType}`;
    if (typeof this[handlerName] === 'function') {
      this[handlerName](event.data, event);
    } else {
      throw new Error(`No handler for event: ${event.eventType}`);
    }
    this._version++;
  }

  // Rebuild state from event history
  static reconstitute(EventClass, id, events) {
    const aggregate = new EventClass(id);
    
    for (const event of events) {
      aggregate._applyEvent(event);
    }
    
    return aggregate;
  }

  getUncommittedEvents() {
    return [...this._uncommittedEvents];
  }

  clearUncommittedEvents() {
    this._uncommittedEvents = [];
  }

  get version() {
    return this._version;
  }
}

module.exports = AggregateRoot;
```

### 2.2 Order Aggregate

```javascript
// domain/order.js
const AggregateRoot = require('./aggregate-root');
const events = require('../events/domain-events');

class Order extends AggregateRoot {
  constructor(id) {
    super(id);
    this.status = null;
    this.userId = null;
    this.items = [];
    this.totalAmount = 0;
    this.currency = 'THB';
    this.shippingAddress = null;
    this.paymentId = null;
    this.trackingNumber = null;
  }

  // ===== COMMANDS =====

  static create(orderId, userId, shippingAddress) {
    const order = new Order(orderId);
    
    if (!userId) throw new Error('UserId is required');
    if (!shippingAddress) throw new Error('ShippingAddress is required');
    
    order.raise(new events.OrderCreated(orderId, {
      userId,
      shippingAddress,
      currency: 'THB'
    }));
    
    return order;
  }

  addItem(productId, productName, quantity, unitPrice) {
    if (this.status !== 'draft') {
      throw new Error('Can only add items to draft orders');
    }
    if (quantity <= 0) throw new Error('Quantity must be positive');
    if (unitPrice < 0) throw new Error('Price must be non-negative');
    
    const existingItem = this.items.find(i => i.productId === productId);
    
    this.raise(new events.OrderItemAdded(this.id, {
      productId,
      productName,
      quantity,
      unitPrice,
      lineTotal: quantity * unitPrice
    }));
  }

  removeItem(productId) {
    if (this.status !== 'draft') {
      throw new Error('Can only remove items from draft orders');
    }
    
    const item = this.items.find(i => i.productId === productId);
    if (!item) throw new Error('Item not found in order');
    
    this.raise(new events.OrderItemRemoved(this.id, { productId }));
  }

  submit() {
    if (this.status !== 'draft') {
      throw new Error('Only draft orders can be submitted');
    }
    if (this.items.length === 0) {
      throw new Error('Cannot submit empty order');
    }
    
    this.raise(new events.OrderSubmitted(this.id, {
      submittedAt: new Date().toISOString(),
      itemCount: this.items.length,
      totalAmount: this.totalAmount
    }));
  }

  confirm(confirmedBy) {
    if (this.status !== 'submitted') {
      throw new Error('Only submitted orders can be confirmed');
    }
    
    this.raise(new events.OrderConfirmed(this.id, {
      confirmedBy,
      confirmedAt: new Date().toISOString()
    }));
  }

  receivePayment(paymentId, amount) {
    if (this.status !== 'confirmed') {
      throw new Error('Only confirmed orders can receive payment');
    }
    if (amount !== this.totalAmount) {
      throw new Error(`Payment amount ${amount} doesn't match order total ${this.totalAmount}`);
    }
    
    this.raise(new events.OrderPaymentReceived(this.id, {
      paymentId,
      amount,
      receivedAt: new Date().toISOString()
    }));
  }

  ship(trackingNumber, carrier) {
    if (this.status !== 'paid') {
      throw new Error('Only paid orders can be shipped');
    }
    
    this.raise(new events.OrderShipped(this.id, {
      trackingNumber,
      carrier,
      shippedAt: new Date().toISOString()
    }));
  }

  cancel(reason, cancelledBy) {
    if (['delivered', 'cancelled'].includes(this.status)) {
      throw new Error(`Cannot cancel ${this.status} order`);
    }
    
    this.raise(new events.OrderCancelled(this.id, {
      reason,
      cancelledBy,
      cancelledAt: new Date().toISOString(),
      refundAmount: this.status === 'paid' ? this.totalAmount : 0
    }));
  }

  // ===== EVENT HANDLERS (Apply to state) =====

  onOrderCreated(data) {
    this.status = 'draft';
    this.userId = data.userId;
    this.shippingAddress = data.shippingAddress;
    this.currency = data.currency;
    this.items = [];
    this.totalAmount = 0;
  }

  onOrderItemAdded(data) {
    const existingIndex = this.items.findIndex(i => i.productId === data.productId);
    
    if (existingIndex >= 0) {
      this.items[existingIndex].quantity += data.quantity;
      this.items[existingIndex].lineTotal += data.lineTotal;
    } else {
      this.items.push({
        productId: data.productId,
        productName: data.productName,
        quantity: data.quantity,
        unitPrice: data.unitPrice,
        lineTotal: data.lineTotal
      });
    }
    
    this.totalAmount = this.items.reduce((sum, item) => sum + item.lineTotal, 0);
  }

  onOrderItemRemoved(data) {
    this.items = this.items.filter(i => i.productId !== data.productId);
    this.totalAmount = this.items.reduce((sum, item) => sum + item.lineTotal, 0);
  }

  onOrderSubmitted() {
    this.status = 'submitted';
  }

  onOrderConfirmed() {
    this.status = 'confirmed';
  }

  onOrderPaymentReceived(data) {
    this.status = 'paid';
    this.paymentId = data.paymentId;
  }

  onOrderShipped(data) {
    this.status = 'shipped';
    this.trackingNumber = data.trackingNumber;
  }

  onOrderDelivered() {
    this.status = 'delivered';
  }

  onOrderCancelled() {
    this.status = 'cancelled';
  }
}

module.exports = Order;
```

---

## 3. Event Store

### 3.1 PostgreSQL Event Store

```sql
-- event-store/migrations/001_create_event_store.sql

CREATE TABLE IF NOT EXISTS events (
  id            UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  aggregate_id  UUID NOT NULL,
  aggregate_type VARCHAR(100) NOT NULL,
  event_type    VARCHAR(200) NOT NULL,
  event_version INT NOT NULL DEFAULT 1,
  sequence_num  BIGSERIAL,         -- Global ordering
  aggregate_seq INT NOT NULL,      -- Per-aggregate ordering
  data          JSONB NOT NULL,
  metadata      JSONB DEFAULT '{}',
  occurred_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  created_by    UUID,              -- User who triggered the event
  correlation_id UUID,             -- For tracking related events
  causation_id  UUID               -- Event that caused this event
);

-- Prevent duplicate events (optimistic concurrency)
CREATE UNIQUE INDEX idx_events_aggregate_seq 
ON events(aggregate_id, aggregate_seq);

-- Fast lookup by aggregate
CREATE INDEX idx_events_aggregate 
ON events(aggregate_id, aggregate_seq ASC);

-- Event streaming by type
CREATE INDEX idx_events_type 
ON events(event_type, sequence_num ASC);

-- Global stream ordering
CREATE INDEX idx_events_sequence 
ON events(sequence_num ASC);

-- Snapshots table
CREATE TABLE IF NOT EXISTS snapshots (
  id              UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  aggregate_id    UUID NOT NULL UNIQUE,
  aggregate_type  VARCHAR(100) NOT NULL,
  aggregate_seq   INT NOT NULL,
  state           JSONB NOT NULL,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_snapshots_aggregate 
ON snapshots(aggregate_id, aggregate_seq DESC);
```

### 3.2 Event Store Repository

```javascript
// event-store/event-store.js
const { Pool } = require('pg');
const EventEmitter = require('events');

class PostgresEventStore extends EventEmitter {
  constructor(pool) {
    super();
    this.pool = pool;
  }

  async appendEvents(aggregateId, aggregateType, events, expectedVersion) {
    const client = await this.pool.connect();
    
    try {
      await client.query('BEGIN');
      
      // Optimistic concurrency check
      const versionResult = await client.query(
        'SELECT MAX(aggregate_seq) as current_version FROM events WHERE aggregate_id = $1',
        [aggregateId]
      );
      
      const currentVersion = versionResult.rows[0].current_version || 0;
      
      if (expectedVersion !== undefined && currentVersion !== expectedVersion) {
        throw new ConcurrencyError(
          `Expected version ${expectedVersion} but got ${currentVersion} for ${aggregateId}`
        );
      }

      // Insert events
      let nextSeq = currentVersion;
      const insertedEvents = [];
      
      for (const event of events) {
        nextSeq++;
        
        const result = await client.query(
          `INSERT INTO events (
            aggregate_id, aggregate_type, event_type, event_version,
            aggregate_seq, data, metadata, occurred_at,
            created_by, correlation_id, causation_id
          ) VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11)
          RETURNING *`,
          [
            aggregateId,
            aggregateType,
            event.eventType,
            event.version || 1,
            nextSeq,
            JSON.stringify(event.data),
            JSON.stringify(event.metadata || {}),
            event.occurredAt || new Date().toISOString(),
            event.metadata?.createdBy || null,
            event.metadata?.correlationId || null,
            event.metadata?.causationId || null
          ]
        );
        
        insertedEvents.push(result.rows[0]);
      }
      
      await client.query('COMMIT');
      
      // Notify subscribers (after commit)
      for (const event of insertedEvents) {
        this.emit('event', event);
        this.emit(event.event_type, event);
      }
      
      return insertedEvents;
    } catch (err) {
      await client.query('ROLLBACK');
      throw err;
    } finally {
      client.release();
    }
  }

  async getEvents(aggregateId, options = {}) {
    const { fromVersion = 0, toVersion = null } = options;
    
    let query = `
      SELECT * FROM events
      WHERE aggregate_id = $1
        AND aggregate_seq > $2
    `;
    const params = [aggregateId, fromVersion];
    
    if (toVersion !== null) {
      query += ` AND aggregate_seq <= $3`;
      params.push(toVersion);
    }
    
    query += ' ORDER BY aggregate_seq ASC';
    
    const result = await this.pool.query(query, params);
    return result.rows.map(this.deserializeEvent);
  }

  async getEventsByType(eventType, options = {}) {
    const { fromSequence = 0, limit = 100 } = options;
    
    const result = await this.pool.query(
      `SELECT * FROM events
       WHERE event_type = $1
         AND sequence_num > $2
       ORDER BY sequence_num ASC
       LIMIT $3`,
      [eventType, fromSequence, limit]
    );
    
    return result.rows.map(this.deserializeEvent);
  }

  async getAllEvents(options = {}) {
    const { fromSequence = 0, limit = 100, eventTypes } = options;
    
    let query = `
      SELECT * FROM events
      WHERE sequence_num > $1
    `;
    const params = [fromSequence];
    
    if (eventTypes && eventTypes.length > 0) {
      query += ` AND event_type = ANY($${params.length + 1})`;
      params.push(eventTypes);
    }
    
    query += ` ORDER BY sequence_num ASC LIMIT $${params.length + 1}`;
    params.push(limit);
    
    const result = await this.pool.query(query, params);
    return result.rows.map(this.deserializeEvent);
  }

  async saveSnapshot(aggregateId, aggregateType, aggregateSeq, state) {
    await this.pool.query(
      `INSERT INTO snapshots (aggregate_id, aggregate_type, aggregate_seq, state)
       VALUES ($1, $2, $3, $4)
       ON CONFLICT (aggregate_id) DO UPDATE
       SET aggregate_seq = $3, state = $4, created_at = NOW()`,
      [aggregateId, aggregateType, aggregateSeq, JSON.stringify(state)]
    );
  }

  async getSnapshot(aggregateId) {
    const result = await this.pool.query(
      'SELECT * FROM snapshots WHERE aggregate_id = $1',
      [aggregateId]
    );
    
    if (result.rows.length === 0) return null;
    
    const snapshot = result.rows[0];
    return {
      aggregateId: snapshot.aggregate_id,
      aggregateType: snapshot.aggregate_type,
      aggregateSeq: snapshot.aggregate_seq,
      state: snapshot.state
    };
  }

  deserializeEvent(row) {
    return {
      eventId: row.id,
      aggregateId: row.aggregate_id,
      aggregateType: row.aggregate_type,
      eventType: row.event_type,
      version: row.event_version,
      sequenceNum: row.sequence_num,
      aggregateSeq: row.aggregate_seq,
      data: row.data,
      metadata: row.metadata,
      occurredAt: row.occurred_at,
      correlationId: row.correlation_id,
      causationId: row.causation_id
    };
  }
}

class ConcurrencyError extends Error {
  constructor(message) {
    super(message);
    this.name = 'ConcurrencyError';
    this.statusCode = 409;
  }
}

module.exports = { PostgresEventStore, ConcurrencyError };
```

---

## 4. Repository with Snapshots

```javascript
// repositories/order-repository.js
const Order = require('../domain/order');

class OrderRepository {
  constructor(eventStore) {
    this.eventStore = eventStore;
    this.SNAPSHOT_THRESHOLD = 50;  // Snapshot every 50 events
  }

  async findById(orderId) {
    // Try to load from snapshot first
    const snapshot = await this.eventStore.getSnapshot(orderId);
    
    let order;
    let fromVersion = 0;
    
    if (snapshot) {
      // Restore from snapshot
      order = new Order(orderId);
      Object.assign(order, snapshot.state);
      order._version = snapshot.aggregateSeq;
      fromVersion = snapshot.aggregateSeq;
    } else {
      order = new Order(orderId);
    }

    // Load events after snapshot
    const events = await this.eventStore.getEvents(orderId, { fromVersion });
    
    if (events.length === 0 && !snapshot) {
      return null;
    }

    // Apply events
    for (const event of events) {
      order._applyEvent(event);
    }

    return order;
  }

  async save(order) {
    const uncommittedEvents = order.getUncommittedEvents();
    
    if (uncommittedEvents.length === 0) return;

    // Append events to store (with optimistic concurrency)
    const expectedVersion = order.version - uncommittedEvents.length;
    
    await this.eventStore.appendEvents(
      order.id,
      'Order',
      uncommittedEvents,
      expectedVersion
    );

    order.clearUncommittedEvents();

    // Create snapshot if needed
    if (order.version % this.SNAPSHOT_THRESHOLD === 0) {
      await this.createSnapshot(order);
    }
  }

  async createSnapshot(order) {
    const state = {
      status: order.status,
      userId: order.userId,
      items: order.items,
      totalAmount: order.totalAmount,
      currency: order.currency,
      shippingAddress: order.shippingAddress,
      paymentId: order.paymentId,
      trackingNumber: order.trackingNumber
    };

    await this.eventStore.saveSnapshot(
      order.id,
      'Order',
      order.version,
      state
    );
  }
}

module.exports = OrderRepository;
```

---

## 5. CQRS - Command Side

### 5.1 Command Handlers

```javascript
// commands/order-commands.js

// Command classes
class CreateOrderCommand {
  constructor(data) {
    this.orderId = data.orderId || crypto.randomUUID();
    this.userId = data.userId;
    this.shippingAddress = data.shippingAddress;
    this.type = 'CreateOrder';
  }
}

class AddItemToOrderCommand {
  constructor(data) {
    this.orderId = data.orderId;
    this.productId = data.productId;
    this.productName = data.productName;
    this.quantity = data.quantity;
    this.unitPrice = data.unitPrice;
    this.type = 'AddItemToOrder';
  }
}

class SubmitOrderCommand {
  constructor(orderId) {
    this.orderId = orderId;
    this.type = 'SubmitOrder';
  }
}

// Command Handler
class OrderCommandHandler {
  constructor(orderRepository, eventBus) {
    this.orderRepository = orderRepository;
    this.eventBus = eventBus;
  }

  async handle(command) {
    const handlerName = `handle${command.type}`;
    
    if (typeof this[handlerName] !== 'function') {
      throw new Error(`No handler for command: ${command.type}`);
    }
    
    return this[handlerName](command);
  }

  async handleCreateOrder(command) {
    // Check idempotency
    const existing = await this.orderRepository.findById(command.orderId);
    if (existing) {
      return { orderId: command.orderId, status: 'already_exists' };
    }

    const order = Order.create(
      command.orderId,
      command.userId,
      command.shippingAddress
    );

    await this.orderRepository.save(order);

    // Publish to event bus for projections
    for (const event of order.getUncommittedEvents()) {
      await this.eventBus.publish(event);
    }

    return { orderId: order.id, status: 'created' };
  }

  async handleAddItemToOrder(command) {
    const order = await this.orderRepository.findById(command.orderId);
    if (!order) throw new NotFoundError('Order not found');

    order.addItem(
      command.productId,
      command.productName,
      command.quantity,
      command.unitPrice
    );

    await this.orderRepository.save(order);
    
    return { orderId: order.id, totalAmount: order.totalAmount };
  }

  async handleSubmitOrder(command) {
    const order = await this.orderRepository.findById(command.orderId);
    if (!order) throw new NotFoundError('Order not found');

    order.submit();
    await this.orderRepository.save(order);
    
    return { orderId: order.id, status: order.status };
  }
}
```

---

## 6. CQRS - Query Side (Projections)

### 6.1 Projection สำหรับ Read Models

```javascript
// projections/order-list-projection.js
const EventEmitter = require('events');

class OrderListProjection {
  constructor(readDb, eventStore) {
    this.readDb = readDb;
    this.eventStore = eventStore;
    this.lastProcessedSequence = 0;
  }

  async initialize() {
    // Create read model tables
    await this.readDb.query(`
      CREATE TABLE IF NOT EXISTS order_list_view (
        order_id    UUID PRIMARY KEY,
        user_id     UUID NOT NULL,
        status      VARCHAR(50) NOT NULL,
        item_count  INT NOT NULL DEFAULT 0,
        total_amount DECIMAL(15, 2) NOT NULL DEFAULT 0,
        currency    CHAR(3) NOT NULL DEFAULT 'THB',
        created_at  TIMESTAMPTZ NOT NULL,
        updated_at  TIMESTAMPTZ NOT NULL
      )
    `);

    await this.readDb.query(`
      CREATE INDEX IF NOT EXISTS idx_order_list_user 
      ON order_list_view(user_id, created_at DESC)
    `);

    // Get last processed sequence
    const result = await this.readDb.query(
      "SELECT value FROM projection_checkpoints WHERE projection_name = 'order_list'"
    );
    
    if (result.rows.length > 0) {
      this.lastProcessedSequence = parseInt(result.rows[0].value);
    }
  }

  async process(event) {
    const handler = this[`on${event.eventType}`];
    
    if (handler) {
      await handler.call(this, event);
    }
    
    // Update checkpoint
    await this.updateCheckpoint(event.sequenceNum);
  }

  async onOrderCreated(event) {
    await this.readDb.query(
      `INSERT INTO order_list_view 
       (order_id, user_id, status, item_count, total_amount, currency, created_at, updated_at)
       VALUES ($1, $2, 'draft', 0, 0, 'THB', $3, $3)
       ON CONFLICT (order_id) DO NOTHING`,
      [event.aggregateId, event.data.userId, event.occurredAt]
    );
  }

  async onOrderItemAdded(event) {
    await this.readDb.query(
      `UPDATE order_list_view
       SET item_count = item_count + 1,
           total_amount = total_amount + $2,
           updated_at = $3
       WHERE order_id = $1`,
      [event.aggregateId, event.data.lineTotal, event.occurredAt]
    );
  }

  async onOrderItemRemoved(event) {
    // Need to recalculate from events or maintain line items
    // For simplicity, we'll rebuild from DB
    await this.rebuildOrderTotals(event.aggregateId);
  }

  async onOrderSubmitted(event) {
    await this.readDb.query(
      `UPDATE order_list_view
       SET status = 'submitted', updated_at = $2
       WHERE order_id = $1`,
      [event.aggregateId, event.occurredAt]
    );
  }

  async onOrderConfirmed(event) {
    await this.updateStatus(event.aggregateId, 'confirmed', event.occurredAt);
  }

  async onOrderPaymentReceived(event) {
    await this.updateStatus(event.aggregateId, 'paid', event.occurredAt);
  }

  async onOrderShipped(event) {
    await this.updateStatus(event.aggregateId, 'shipped', event.occurredAt);
  }

  async onOrderDelivered(event) {
    await this.updateStatus(event.aggregateId, 'delivered', event.occurredAt);
  }

  async onOrderCancelled(event) {
    await this.updateStatus(event.aggregateId, 'cancelled', event.occurredAt);
  }

  async updateStatus(orderId, status, updatedAt) {
    await this.readDb.query(
      'UPDATE order_list_view SET status = $2, updated_at = $3 WHERE order_id = $1',
      [orderId, status, updatedAt]
    );
  }

  async updateCheckpoint(sequenceNum) {
    await this.readDb.query(
      `INSERT INTO projection_checkpoints (projection_name, value, updated_at)
       VALUES ('order_list', $1, NOW())
       ON CONFLICT (projection_name) DO UPDATE
       SET value = $1, updated_at = NOW()`,
      [sequenceNum]
    );
    this.lastProcessedSequence = sequenceNum;
  }

  // Rebuild projection from scratch
  async rebuild() {
    console.log('Rebuilding order_list projection...');
    
    await this.readDb.query('TRUNCATE order_list_view');
    this.lastProcessedSequence = 0;
    
    let processed = 0;
    let hasMore = true;
    
    while (hasMore) {
      const events = await this.eventStore.getAllEvents({
        fromSequence: this.lastProcessedSequence,
        limit: 1000,
        eventTypes: [
          'OrderCreated', 'OrderItemAdded', 'OrderItemRemoved',
          'OrderSubmitted', 'OrderConfirmed', 'OrderPaymentReceived',
          'OrderShipped', 'OrderDelivered', 'OrderCancelled'
        ]
      });
      
      for (const event of events) {
        await this.process(event);
        processed++;
      }
      
      hasMore = events.length === 1000;
    }
    
    console.log(`Rebuilt order_list projection: ${processed} events processed`);
  }
}
```

### 6.2 Query Service

```javascript
// queries/order-queries.js

class OrderQueryService {
  constructor(readDb) {
    this.db = readDb;
  }

  async getUserOrders(userId, options = {}) {
    const {
      status,
      cursor,
      limit = 20
    } = options;

    let conditions = ['user_id = $1'];
    const params = [userId];

    if (status) {
      params.push(status);
      conditions.push(`status = $${params.length}`);
    }

    if (cursor) {
      const decodedCursor = JSON.parse(Buffer.from(cursor, 'base64url').toString());
      params.push(decodedCursor.createdAt, decodedCursor.orderId);
      conditions.push(
        `(created_at, order_id) < ($${params.length - 1}, $${params.length})`
      );
    }

    const whereClause = `WHERE ${conditions.join(' AND ')}`;

    const result = await this.db.query(
      `SELECT * FROM order_list_view
       ${whereClause}
       ORDER BY created_at DESC, order_id DESC
       LIMIT $${params.length + 1}`,
      [...params, limit + 1]
    );

    const rows = result.rows;
    const hasMore = rows.length > limit;
    const items = hasMore ? rows.slice(0, limit) : rows;

    const nextCursor = hasMore
      ? Buffer.from(JSON.stringify({
          createdAt: items[items.length - 1].created_at,
          orderId: items[items.length - 1].order_id
        })).toString('base64url')
      : null;

    return {
      orders: items,
      pageInfo: { hasMore, nextCursor }
    };
  }

  async getOrderDetail(orderId, userId) {
    const result = await this.db.query(
      `SELECT 
        o.*,
        json_agg(
          json_build_object(
            'productId', oi.product_id,
            'productName', oi.product_name,
            'quantity', oi.quantity,
            'unitPrice', oi.unit_price,
            'lineTotal', oi.line_total
          ) ORDER BY oi.added_at
        ) as items
       FROM order_detail_view o
       LEFT JOIN order_items_view oi ON oi.order_id = o.order_id
       WHERE o.order_id = $1 AND o.user_id = $2
       GROUP BY o.order_id, o.user_id, o.status, o.total_amount, 
                o.currency, o.shipping_address, o.created_at, o.updated_at`,
      [orderId, userId]
    );

    if (result.rows.length === 0) return null;
    return result.rows[0];
  }

  async getOrderStatsByUser(userId) {
    const result = await this.db.query(
      `SELECT
        COUNT(*) FILTER (WHERE status = 'delivered') as completed_count,
        COUNT(*) FILTER (WHERE status = 'cancelled') as cancelled_count,
        COUNT(*) FILTER (WHERE status NOT IN ('delivered', 'cancelled')) as active_count,
        SUM(total_amount) FILTER (WHERE status = 'delivered') as total_spent
       FROM order_list_view
       WHERE user_id = $1`,
      [userId]
    );

    return result.rows[0];
  }
}

module.exports = OrderQueryService;
```

---

## 7. Projection Runner (Event Subscription)

```javascript
// projections/projection-runner.js
class ProjectionRunner {
  constructor(eventStore, projections) {
    this.eventStore = eventStore;
    this.projections = projections;
    this.running = false;
    this.pollInterval = 1000; // 1 second
  }

  async start() {
    // Initialize all projections
    for (const projection of this.projections) {
      await projection.initialize();
    }

    this.running = true;
    this.poll();

    console.log('Projection runner started');
  }

  async poll() {
    while (this.running) {
      try {
        await this.processNewEvents();
      } catch (err) {
        console.error('Projection error:', err);
      }
      
      await new Promise(resolve => setTimeout(resolve, this.pollInterval));
    }
  }

  async processNewEvents() {
    // Get minimum sequence across all projections
    const fromSequence = Math.min(
      ...this.projections.map(p => p.lastProcessedSequence)
    );

    const events = await this.eventStore.getAllEvents({
      fromSequence,
      limit: 500
    });

    if (events.length === 0) return;

    // Process events for each projection
    for (const event of events) {
      for (const projection of this.projections) {
        if (event.sequenceNum > projection.lastProcessedSequence) {
          await projection.process(event);
        }
      }
    }
  }

  stop() {
    this.running = false;
  }
}
```

---

## สรุป

```
Event Sourcing + CQRS Pattern:

Write Side (Command):
User Request → Command → Command Handler
                              ↓
                        Domain Logic (Aggregate)
                              ↓
                        Events → Event Store

Read Side (Query):
Event Store → Projection Runner → Read Model (PostgreSQL/Redis/Elasticsearch)
                                        ↓
                               Query → Read Model → User Response

Benefits:
✓ Complete audit trail (ทุก state change ถูกบันทึก)
✓ Time travel debugging
✓ Multiple read models สำหรับ use cases ต่างกัน
✓ Eventual consistency allows better performance
✓ Event-driven architecture ได้ฟรี
```

**Next:** Part 23 - Saga Pattern Deep Dive: Managing Distributed Transactions

# Part 51: Event Sourcing และ CQRS ขั้นสูง

## บทนำ

Event Sourcing เก็บ state ของ application เป็น sequence ของ events แทนการเก็บ current state โดยตรง ร่วมกับ CQRS (Command Query Responsibility Segregation) ที่แยก write model ออกจาก read model

## หัวข้อที่ครอบคลุม

1. Event Store Implementation
2. Aggregate กับ Event Replay
3. Projections และ Read Models
4. Snapshot Pattern
5. Process Manager (Saga)
6. Eventual Consistency Handling
7. Event Versioning
8. Integration กับ Kafka

---

## 1. Event Store

```typescript
// event-store/event-store.ts
import { Pool } from 'pg';
import { v4 as uuidv4 } from 'uuid';

export interface StoredEvent {
  eventId: string;
  streamId: string;
  eventType: string;
  version: number;
  data: Record<string, unknown>;
  metadata: {
    correlationId: string;
    causationId?: string;
    userId?: string;
    timestamp: string;
  };
}

export interface AppendToStreamOptions {
  expectedVersion: number | 'any' | 'no_stream';
}

export class EventStore {
  constructor(private db: Pool) {}
  
  async appendToStream(
    streamId: string,
    events: Array<Omit<StoredEvent, 'eventId' | 'version'>>,
    options: AppendToStreamOptions = { expectedVersion: 'any' }
  ): Promise<StoredEvent[]> {
    const client = await this.db.connect();
    
    try {
      await client.query('BEGIN');
      
      // ตรวจสอบ current version
      const currentVersionResult = await client.query(
        'SELECT COALESCE(MAX(version), -1) as version FROM events WHERE stream_id = $1',
        [streamId]
      );
      const currentVersion = parseInt(currentVersionResult.rows[0].version);
      
      // Optimistic concurrency check
      if (options.expectedVersion !== 'any') {
        if (options.expectedVersion === 'no_stream' && currentVersion !== -1) {
          throw new Error(`Stream ${streamId} already exists`);
        } else if (typeof options.expectedVersion === 'number' &&
                   currentVersion !== options.expectedVersion) {
          throw new Error(
            `Concurrency conflict: expected version ${options.expectedVersion}, ` +
            `but current is ${currentVersion}`
          );
        }
      }
      
      // Append events
      const storedEvents: StoredEvent[] = [];
      let nextVersion = currentVersion + 1;
      
      for (const event of events) {
        const storedEvent: StoredEvent = {
          eventId: uuidv4(),
          streamId,
          eventType: event.eventType,
          version: nextVersion++,
          data: event.data,
          metadata: {
            ...event.metadata,
            timestamp: new Date().toISOString(),
          },
        };
        
        await client.query(
          `INSERT INTO events 
           (event_id, stream_id, event_type, version, data, metadata, created_at)
           VALUES ($1, $2, $3, $4, $5, $6, NOW())`,
          [
            storedEvent.eventId,
            storedEvent.streamId,
            storedEvent.eventType,
            storedEvent.version,
            JSON.stringify(storedEvent.data),
            JSON.stringify(storedEvent.metadata),
          ]
        );
        
        storedEvents.push(storedEvent);
      }
      
      await client.query('COMMIT');
      
      // Publish events for subscriptions
      await this.publishEvents(storedEvents);
      
      return storedEvents;
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }
  
  async readStreamForward(
    streamId: string,
    fromVersion: number = 0,
    maxCount: number = 1000
  ): Promise<StoredEvent[]> {
    const result = await this.db.query(
      `SELECT event_id, stream_id, event_type, version, data, metadata
       FROM events
       WHERE stream_id = $1 AND version >= $2
       ORDER BY version ASC
       LIMIT $3`,
      [streamId, fromVersion, maxCount]
    );
    
    return result.rows.map(row => ({
      eventId: row.event_id,
      streamId: row.stream_id,
      eventType: row.event_type,
      version: row.version,
      data: row.data,
      metadata: row.metadata,
    }));
  }
  
  async readAllEvents(
    fromPosition: number = 0,
    maxCount: number = 1000
  ): Promise<StoredEvent[]> {
    const result = await this.db.query(
      `SELECT event_id, stream_id, event_type, version, data, metadata
       FROM events
       WHERE global_position > $1
       ORDER BY global_position ASC
       LIMIT $2`,
      [fromPosition, maxCount]
    );
    
    return result.rows.map(row => ({
      eventId: row.event_id,
      streamId: row.stream_id,
      eventType: row.event_type,
      version: row.version,
      data: row.data,
      metadata: row.metadata,
    }));
  }
  
  private async publishEvents(events: StoredEvent[]): Promise<void> {
    // ใช้ PostgreSQL NOTIFY สำหรับ real-time notifications
    for (const event of events) {
      await this.db.query(
        `SELECT pg_notify('new_event', $1)`,
        [JSON.stringify({ streamId: event.streamId, eventType: event.eventType })]
      );
    }
  }
  
  subscribe(callback: (event: StoredEvent) => void): () => void {
    // ใช้ separate connection สำหรับ LISTEN
    const listener = new Pool({ /* config */ });
    listener.connect().then(client => {
      client.query('LISTEN new_event');
      client.on('notification', async (msg) => {
        if (msg.channel === 'new_event' && msg.payload) {
          const payload = JSON.parse(msg.payload);
          const events = await this.readStreamForward(payload.streamId, 0);
          const latestEvent = events[events.length - 1];
          if (latestEvent) callback(latestEvent);
        }
      });
    });
    
    return () => listener.end();
  }
}

// Database schema
const SCHEMA = `
CREATE TABLE IF NOT EXISTS events (
  id BIGSERIAL,
  event_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  stream_id VARCHAR(255) NOT NULL,
  event_type VARCHAR(255) NOT NULL,
  version INTEGER NOT NULL,
  data JSONB NOT NULL,
  metadata JSONB NOT NULL DEFAULT '{}',
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  global_position BIGSERIAL,
  UNIQUE(stream_id, version)
);

CREATE INDEX IF NOT EXISTS idx_events_stream_id ON events(stream_id, version);
CREATE INDEX IF NOT EXISTS idx_events_event_type ON events(event_type);
CREATE INDEX IF NOT EXISTS idx_events_global_position ON events(global_position);
`;
```

---

## 2. Aggregate กับ Event Replay

```typescript
// domain/order-aggregate.ts

// Domain Events
export type OrderEvent =
  | { type: 'OrderCreated'; data: { orderId: string; customerId: string; items: OrderItem[] } }
  | { type: 'ItemAdded'; data: { orderId: string; item: OrderItem } }
  | { type: 'ItemRemoved'; data: { orderId: string; productId: string } }
  | { type: 'OrderConfirmed'; data: { orderId: string; confirmedAt: string } }
  | { type: 'PaymentProcessed'; data: { orderId: string; paymentId: string; amount: number } }
  | { type: 'OrderShipped'; data: { orderId: string; trackingNumber: string } }
  | { type: 'OrderCancelled'; data: { orderId: string; reason: string } };

export interface OrderItem {
  productId: string;
  name: string;
  quantity: number;
  price: number;
}

export type OrderStatus = 'pending' | 'confirmed' | 'paid' | 'shipped' | 'cancelled';

export interface OrderState {
  orderId: string;
  customerId: string;
  items: OrderItem[];
  status: OrderStatus;
  totalAmount: number;
  paymentId?: string;
  trackingNumber?: string;
  version: number;
}

export class OrderAggregate {
  private state: OrderState;
  private uncommittedEvents: OrderEvent[] = [];
  
  private constructor(state: OrderState) {
    this.state = state;
  }
  
  // Factory: สร้าง Order ใหม่
  static create(customerId: string, items: OrderItem[]): OrderAggregate {
    const orderId = uuidv4();
    const aggregate = new OrderAggregate({
      orderId: '',
      customerId: '',
      items: [],
      status: 'pending',
      totalAmount: 0,
      version: -1,
    });
    
    aggregate.apply({
      type: 'OrderCreated',
      data: { orderId, customerId, items },
    });
    
    return aggregate;
  }
  
  // Factory: ประกอบ aggregate จาก events
  static rehydrate(events: OrderEvent[]): OrderAggregate {
    const aggregate = new OrderAggregate({
      orderId: '',
      customerId: '',
      items: [],
      status: 'pending',
      totalAmount: 0,
      version: -1,
    });
    
    for (const event of events) {
      aggregate.applyEvent(event);
      aggregate.state.version++;
    }
    
    return aggregate;
  }
  
  // Commands
  addItem(item: OrderItem): void {
    this.assertStatus('pending');
    this.apply({
      type: 'ItemAdded',
      data: { orderId: this.state.orderId, item },
    });
  }
  
  removeItem(productId: string): void {
    this.assertStatus('pending');
    const item = this.state.items.find(i => i.productId === productId);
    if (!item) throw new Error(`Item ${productId} not found`);
    
    this.apply({
      type: 'ItemRemoved',
      data: { orderId: this.state.orderId, productId },
    });
  }
  
  confirm(): void {
    this.assertStatus('pending');
    if (this.state.items.length === 0) {
      throw new Error('Cannot confirm empty order');
    }
    
    this.apply({
      type: 'OrderConfirmed',
      data: {
        orderId: this.state.orderId,
        confirmedAt: new Date().toISOString(),
      },
    });
  }
  
  processPayment(paymentId: string, amount: number): void {
    this.assertStatus('confirmed');
    if (amount !== this.state.totalAmount) {
      throw new Error(`Payment amount ${amount} doesn't match total ${this.state.totalAmount}`);
    }
    
    this.apply({
      type: 'PaymentProcessed',
      data: { orderId: this.state.orderId, paymentId, amount },
    });
  }
  
  ship(trackingNumber: string): void {
    this.assertStatus('paid');
    this.apply({
      type: 'OrderShipped',
      data: { orderId: this.state.orderId, trackingNumber },
    });
  }
  
  cancel(reason: string): void {
    if (!['pending', 'confirmed'].includes(this.state.status)) {
      throw new Error(`Cannot cancel order in status: ${this.state.status}`);
    }
    
    this.apply({
      type: 'OrderCancelled',
      data: { orderId: this.state.orderId, reason },
    });
  }
  
  // Event application (pure function: state transition)
  private applyEvent(event: OrderEvent): void {
    switch (event.type) {
      case 'OrderCreated':
        this.state.orderId = event.data.orderId;
        this.state.customerId = event.data.customerId;
        this.state.items = [...event.data.items];
        this.state.status = 'pending';
        this.state.totalAmount = this.calculateTotal(event.data.items);
        break;
        
      case 'ItemAdded':
        this.state.items.push(event.data.item);
        this.state.totalAmount = this.calculateTotal(this.state.items);
        break;
        
      case 'ItemRemoved':
        this.state.items = this.state.items.filter(
          i => i.productId !== event.data.productId
        );
        this.state.totalAmount = this.calculateTotal(this.state.items);
        break;
        
      case 'OrderConfirmed':
        this.state.status = 'confirmed';
        break;
        
      case 'PaymentProcessed':
        this.state.status = 'paid';
        this.state.paymentId = event.data.paymentId;
        break;
        
      case 'OrderShipped':
        this.state.status = 'shipped';
        this.state.trackingNumber = event.data.trackingNumber;
        break;
        
      case 'OrderCancelled':
        this.state.status = 'cancelled';
        break;
    }
  }
  
  private apply(event: OrderEvent): void {
    this.applyEvent(event);
    this.state.version++;
    this.uncommittedEvents.push(event);
  }
  
  private assertStatus(expectedStatus: OrderStatus): void {
    if (this.state.status !== expectedStatus) {
      throw new Error(
        `Invalid operation: order is ${this.state.status}, expected ${expectedStatus}`
      );
    }
  }
  
  private calculateTotal(items: OrderItem[]): number {
    return items.reduce((sum, item) => sum + item.price * item.quantity, 0);
  }
  
  getState(): Readonly<OrderState> {
    return { ...this.state };
  }
  
  getUncommittedEvents(): OrderEvent[] {
    return [...this.uncommittedEvents];
  }
  
  clearUncommittedEvents(): void {
    this.uncommittedEvents = [];
  }
  
  get id(): string {
    return this.state.orderId;
  }
  
  get version(): number {
    return this.state.version;
  }
}
```

---

## 3. Order Repository

```typescript
// infrastructure/order-repository.ts
import { EventStore, StoredEvent } from '../event-store/event-store';
import { OrderAggregate, OrderEvent } from '../domain/order-aggregate';
import { SnapshotStore, OrderSnapshot } from './snapshot-store';

export class OrderRepository {
  constructor(
    private eventStore: EventStore,
    private snapshotStore: SnapshotStore
  ) {}
  
  async save(order: OrderAggregate): Promise<void> {
    const uncommittedEvents = order.getUncommittedEvents();
    if (uncommittedEvents.length === 0) return;
    
    const streamId = `order-${order.id}`;
    const storedEvents = uncommittedEvents.map(event => ({
      streamId,
      eventType: event.type,
      data: event.data as Record<string, unknown>,
      metadata: {
        correlationId: uuidv4(),
        timestamp: new Date().toISOString(),
      },
    }));
    
    await this.eventStore.appendToStream(streamId, storedEvents, {
      expectedVersion: order.version - uncommittedEvents.length,
    });
    
    order.clearUncommittedEvents();
    
    // Create snapshot every 50 events
    if (order.version > 0 && order.version % 50 === 0) {
      await this.snapshotStore.save({
        streamId,
        version: order.version,
        state: order.getState(),
        createdAt: new Date().toISOString(),
      });
    }
  }
  
  async findById(orderId: string): Promise<OrderAggregate | null> {
    const streamId = `order-${orderId}`;
    
    // Check for snapshot
    const snapshot = await this.snapshotStore.findLatest(streamId);
    
    let fromVersion = 0;
    let initialEvents: OrderEvent[] = [];
    
    if (snapshot) {
      // Start from snapshot
      fromVersion = snapshot.version + 1;
      // เริ่มต้นจาก snapshot state
      const order = OrderAggregate.fromSnapshot(snapshot.state);
      
      // Load events after snapshot
      const events = await this.eventStore.readStreamForward(streamId, fromVersion);
      if (events.length === 0 && fromVersion > 0) {
        return order;
      }
      
      const domainEvents = events.map(e => this.toDomainEvent(e));
      return OrderAggregate.rehydrateFromSnapshot(snapshot.state, domainEvents);
    }
    
    // Load all events
    const events = await this.eventStore.readStreamForward(streamId);
    if (events.length === 0) return null;
    
    const domainEvents = events.map(e => this.toDomainEvent(e));
    return OrderAggregate.rehydrate(domainEvents);
  }
  
  private toDomainEvent(stored: StoredEvent): OrderEvent {
    return {
      type: stored.eventType as OrderEvent['type'],
      data: stored.data as any,
    };
  }
}

// Import missing
declare function uuidv4(): string;
```

---

## 4. Projections (Read Models)

```typescript
// projections/order-projection.ts
import { Pool } from 'pg';
import { StoredEvent } from '../event-store/event-store';

// Projection เป็น denormalized read model สำหรับ queries
export class OrderProjection {
  constructor(private readDb: Pool) {}
  
  async project(event: StoredEvent): Promise<void> {
    switch (event.eventType) {
      case 'OrderCreated':
        await this.handleOrderCreated(event);
        break;
      case 'ItemAdded':
        await this.handleItemAdded(event);
        break;
      case 'ItemRemoved':
        await this.handleItemRemoved(event);
        break;
      case 'OrderConfirmed':
        await this.handleOrderConfirmed(event);
        break;
      case 'PaymentProcessed':
        await this.handlePaymentProcessed(event);
        break;
      case 'OrderShipped':
        await this.handleOrderShipped(event);
        break;
      case 'OrderCancelled':
        await this.handleOrderCancelled(event);
        break;
    }
  }
  
  private async handleOrderCreated(event: StoredEvent): Promise<void> {
    const { orderId, customerId, items } = event.data as any;
    const totalAmount = items.reduce(
      (sum: number, item: any) => sum + item.price * item.quantity, 0
    );
    
    await this.readDb.query(
      `INSERT INTO order_summaries 
       (order_id, customer_id, status, total_amount, item_count, created_at, version)
       VALUES ($1, $2, 'pending', $3, $4, $5, $6)`,
      [orderId, customerId, totalAmount, items.length, event.metadata.timestamp, event.version]
    );
    
    // Insert items
    for (const item of items) {
      await this.readDb.query(
        `INSERT INTO order_items_view 
         (order_id, product_id, name, quantity, price)
         VALUES ($1, $2, $3, $4, $5)`,
        [orderId, item.productId, item.name, item.quantity, item.price]
      );
    }
  }
  
  private async handleOrderConfirmed(event: StoredEvent): Promise<void> {
    const { orderId } = event.data as any;
    await this.readDb.query(
      `UPDATE order_summaries 
       SET status = 'confirmed', confirmed_at = $2, version = $3
       WHERE order_id = $1`,
      [orderId, event.data.confirmedAt, event.version]
    );
  }
  
  private async handlePaymentProcessed(event: StoredEvent): Promise<void> {
    const { orderId, paymentId } = event.data as any;
    await this.readDb.query(
      `UPDATE order_summaries 
       SET status = 'paid', payment_id = $2, paid_at = $3, version = $4
       WHERE order_id = $1`,
      [orderId, paymentId, event.metadata.timestamp, event.version]
    );
  }
  
  private async handleOrderShipped(event: StoredEvent): Promise<void> {
    const { orderId, trackingNumber } = event.data as any;
    await this.readDb.query(
      `UPDATE order_summaries 
       SET status = 'shipped', tracking_number = $2, shipped_at = $3, version = $4
       WHERE order_id = $1`,
      [orderId, trackingNumber, event.metadata.timestamp, event.version]
    );
  }
  
  private async handleOrderCancelled(event: StoredEvent): Promise<void> {
    const { orderId } = event.data as any;
    await this.readDb.query(
      `UPDATE order_summaries 
       SET status = 'cancelled', cancelled_at = $2, version = $3
       WHERE order_id = $1`,
      [orderId, event.metadata.timestamp, event.version]
    );
  }
  
  private async handleItemAdded(event: StoredEvent): Promise<void> {
    const { orderId, item } = event.data as any;
    
    await this.readDb.query(
      `INSERT INTO order_items_view 
       (order_id, product_id, name, quantity, price)
       VALUES ($1, $2, $3, $4, $5)
       ON CONFLICT (order_id, product_id) DO UPDATE
       SET quantity = EXCLUDED.quantity, price = EXCLUDED.price`,
      [orderId, item.productId, item.name, item.quantity, item.price]
    );
    
    // Update total
    await this.readDb.query(
      `UPDATE order_summaries
       SET total_amount = (
         SELECT SUM(quantity * price) FROM order_items_view WHERE order_id = $1
       ), item_count = (
         SELECT COUNT(*) FROM order_items_view WHERE order_id = $1
       ), version = $2
       WHERE order_id = $1`,
      [orderId, event.version]
    );
  }
  
  private async handleItemRemoved(event: StoredEvent): Promise<void> {
    const { orderId, productId } = event.data as any;
    
    await this.readDb.query(
      'DELETE FROM order_items_view WHERE order_id = $1 AND product_id = $2',
      [orderId, productId]
    );
    
    await this.readDb.query(
      `UPDATE order_summaries
       SET total_amount = (
         SELECT COALESCE(SUM(quantity * price), 0) FROM order_items_view WHERE order_id = $1
       ), item_count = (
         SELECT COUNT(*) FROM order_items_view WHERE order_id = $1
       ), version = $2
       WHERE order_id = $1`,
      [orderId, event.version]
    );
  }
}

// Projection Runner: subscribe to event store and update projections
export class ProjectionRunner {
  private position: number = 0;
  private isRunning = false;
  
  constructor(
    private eventStore: EventStore,
    private projections: Array<{ project: (event: StoredEvent) => Promise<void> }>,
    private db: Pool
  ) {}
  
  async start(): Promise<void> {
    // Load checkpoint position
    const result = await this.db.query(
      'SELECT position FROM projection_checkpoints WHERE projection_name = $1',
      ['order_projection']
    );
    this.position = result.rows[0]?.position || 0;
    
    this.isRunning = true;
    this.runLoop();
  }
  
  private async runLoop(): Promise<void> {
    while (this.isRunning) {
      try {
        const events = await this.eventStore.readAllEvents(this.position, 100);
        
        for (const event of events) {
          await this.processEvent(event);
          this.position++;
        }
        
        if (events.length === 0) {
          // ไม่มี events ใหม่ รอแล้ว retry
          await new Promise(resolve => setTimeout(resolve, 100));
        }
      } catch (err) {
        console.error('Projection runner error:', err);
        await new Promise(resolve => setTimeout(resolve, 1000));
      }
    }
  }
  
  private async processEvent(event: StoredEvent): Promise<void> {
    for (const projection of this.projections) {
      await projection.project(event);
    }
    
    // Save checkpoint
    await this.db.query(
      `INSERT INTO projection_checkpoints (projection_name, position)
       VALUES ('order_projection', $1)
       ON CONFLICT (projection_name) DO UPDATE SET position = $1`,
      [this.position]
    );
  }
  
  stop(): void {
    this.isRunning = false;
  }
}

// Missing imports/declarations
declare class EventStore {
  readAllEvents(from: number, max: number): Promise<StoredEvent[]>;
}
declare class Pool {}
```

---

## 5. Snapshot Store

```typescript
// infrastructure/snapshot-store.ts
import { Pool } from 'pg';
import { OrderState } from '../domain/order-aggregate';

export interface OrderSnapshot {
  streamId: string;
  version: number;
  state: OrderState;
  createdAt: string;
}

export class SnapshotStore {
  constructor(private db: Pool) {}
  
  async save(snapshot: OrderSnapshot): Promise<void> {
    await this.db.query(
      `INSERT INTO snapshots (stream_id, version, state, created_at)
       VALUES ($1, $2, $3, $4)
       ON CONFLICT (stream_id) DO UPDATE
       SET version = $2, state = $3, created_at = $4`,
      [snapshot.streamId, snapshot.version, JSON.stringify(snapshot.state), snapshot.createdAt]
    );
  }
  
  async findLatest(streamId: string): Promise<OrderSnapshot | null> {
    const result = await this.db.query(
      'SELECT * FROM snapshots WHERE stream_id = $1 ORDER BY version DESC LIMIT 1',
      [streamId]
    );
    
    if (result.rows.length === 0) return null;
    
    const row = result.rows[0];
    return {
      streamId: row.stream_id,
      version: row.version,
      state: row.state,
      createdAt: row.created_at,
    };
  }
}
```

---

## 6. CQRS Command Handler

```typescript
// application/order-command-handler.ts
import { OrderRepository } from '../infrastructure/order-repository';
import { OrderAggregate } from '../domain/order-aggregate';
import { EventBus } from '../infrastructure/event-bus';

export interface CreateOrderCommand {
  customerId: string;
  items: Array<{
    productId: string;
    name: string;
    quantity: number;
    price: number;
  }>;
}

export interface ConfirmOrderCommand {
  orderId: string;
}

export interface ProcessPaymentCommand {
  orderId: string;
  paymentId: string;
  amount: number;
}

export interface CancelOrderCommand {
  orderId: string;
  reason: string;
}

export class OrderCommandHandler {
  constructor(
    private repository: OrderRepository,
    private eventBus: EventBus
  ) {}
  
  async handleCreateOrder(command: CreateOrderCommand): Promise<string> {
    const order = OrderAggregate.create(command.customerId, command.items);
    await this.repository.save(order);
    
    // Publish integration event
    await this.eventBus.publish('OrderCreated', {
      orderId: order.id,
      customerId: command.customerId,
      totalAmount: order.getState().totalAmount,
    });
    
    return order.id;
  }
  
  async handleConfirmOrder(command: ConfirmOrderCommand): Promise<void> {
    const order = await this.repository.findById(command.orderId);
    if (!order) throw new Error(`Order ${command.orderId} not found`);
    
    order.confirm();
    await this.repository.save(order);
    
    await this.eventBus.publish('OrderConfirmed', {
      orderId: command.orderId,
    });
  }
  
  async handleProcessPayment(command: ProcessPaymentCommand): Promise<void> {
    const order = await this.repository.findById(command.orderId);
    if (!order) throw new Error(`Order ${command.orderId} not found`);
    
    order.processPayment(command.paymentId, command.amount);
    await this.repository.save(order);
    
    await this.eventBus.publish('OrderPaid', {
      orderId: command.orderId,
      paymentId: command.paymentId,
      amount: command.amount,
    });
  }
  
  async handleCancelOrder(command: CancelOrderCommand): Promise<void> {
    const order = await this.repository.findById(command.orderId);
    if (!order) throw new Error(`Order ${command.orderId} not found`);
    
    order.cancel(command.reason);
    await this.repository.save(order);
    
    await this.eventBus.publish('OrderCancelled', {
      orderId: command.orderId,
      reason: command.reason,
    });
  }
}

// CQRS Query Handler
export class OrderQueryHandler {
  constructor(private readDb: Pool) {}
  
  async getOrderById(orderId: string) {
    const [orderResult, itemsResult] = await Promise.all([
      this.readDb.query(
        'SELECT * FROM order_summaries WHERE order_id = $1',
        [orderId]
      ),
      this.readDb.query(
        'SELECT * FROM order_items_view WHERE order_id = $1',
        [orderId]
      ),
    ]);
    
    if (orderResult.rows.length === 0) return null;
    
    return {
      ...orderResult.rows[0],
      items: itemsResult.rows,
    };
  }
  
  async getOrdersByCustomer(
    customerId: string,
    status?: string,
    page: number = 1,
    limit: number = 20
  ) {
    const offset = (page - 1) * limit;
    const conditions = ['customer_id = $1'];
    const params: unknown[] = [customerId];
    
    if (status) {
      conditions.push(`status = $${params.length + 1}`);
      params.push(status);
    }
    
    const where = conditions.join(' AND ');
    
    const [countResult, dataResult] = await Promise.all([
      this.readDb.query(
        `SELECT COUNT(*) FROM order_summaries WHERE ${where}`,
        params
      ),
      this.readDb.query(
        `SELECT * FROM order_summaries WHERE ${where}
         ORDER BY created_at DESC LIMIT $${params.length + 1} OFFSET $${params.length + 2}`,
        [...params, limit, offset]
      ),
    ]);
    
    return {
      total: parseInt(countResult.rows[0].count),
      page,
      limit,
      data: dataResult.rows,
    };
  }
  
  async getOrderHistory(orderId: string) {
    const result = await this.readDb.query(
      `SELECT event_type, data, metadata, created_at
       FROM events
       WHERE stream_id = $1
       ORDER BY version ASC`,
      [`order-${orderId}`]
    );
    
    return result.rows;
  }
}

declare class Pool {}
declare class EventBus {
  publish(event: string, data: any): Promise<void>;
}
```

---

## 7. Event Versioning

```typescript
// event-store/event-versioning.ts

// Event Upcaster: แปลง event เวอร์ชันเก่าเป็นเวอร์ชันใหม่
interface EventUpcaster {
  canUpcast(eventType: string, version: number): boolean;
  upcast(event: StoredEvent): StoredEvent;
}

// V1 → V2: เพิ่ม field ใหม่
class OrderCreatedV1ToV2Upcaster implements EventUpcaster {
  canUpcast(eventType: string, version: number): boolean {
    return eventType === 'OrderCreated' && version === 1;
  }
  
  upcast(event: StoredEvent): StoredEvent {
    const data = event.data as any;
    return {
      ...event,
      data: {
        ...data,
        // เพิ่ม field ใหม่ที่ไม่มีใน V1
        currency: data.currency || 'THB',
        source: data.source || 'web',
        metadata: {
          ...event.metadata,
        }
      },
      eventType: 'OrderCreated', // ยังเป็น event type เดิม แต่ schema ใหม่
    };
  }
}

export class EventUpcasterChain {
  private upcasters: EventUpcaster[] = [];
  
  register(upcaster: EventUpcaster): void {
    this.upcasters.push(upcaster);
  }
  
  upcast(event: StoredEvent): StoredEvent {
    let current = event;
    
    // Apply upcasters จนกว่าจะไม่มี upcaster ที่ match อีก
    let changed = true;
    while (changed) {
      changed = false;
      for (const upcaster of this.upcasters) {
        const version = (current.metadata as any).schemaVersion || 1;
        if (upcaster.canUpcast(current.eventType, version)) {
          current = upcaster.upcast(current);
          changed = true;
        }
      }
    }
    
    return current;
  }
}
```

---

## 8. Integration กับ Kafka

```typescript
// infrastructure/kafka-event-publisher.ts
import { Kafka, Producer, CompressionTypes } from 'kafkajs';
import { StoredEvent } from '../event-store/event-store';

export class KafkaEventPublisher {
  private producer: Producer;
  
  constructor(brokers: string[]) {
    const kafka = new Kafka({
      clientId: 'event-sourcing-service',
      brokers,
    });
    
    this.producer = kafka.producer({
      idempotent: true, // Exactly-once semantics
      maxInFlightRequests: 5,
    });
  }
  
  async connect(): Promise<void> {
    await this.producer.connect();
  }
  
  async publishEvent(event: StoredEvent): Promise<void> {
    const topic = `events.${event.eventType.toLowerCase().replace(/_/g, '.')}`;
    
    await this.producer.send({
      topic,
      compression: CompressionTypes.GZIP,
      messages: [
        {
          key: event.streamId,
          value: JSON.stringify(event),
          headers: {
            'event-type': event.eventType,
            'correlation-id': event.metadata.correlationId,
            'stream-id': event.streamId,
            'version': String(event.version),
          },
        }
      ],
    });
  }
  
  async publishEvents(events: StoredEvent[]): Promise<void> {
    // Group events by topic
    const byTopic = new Map<string, StoredEvent[]>();
    for (const event of events) {
      const topic = `events.${event.eventType.toLowerCase().replace(/_/g, '.')}`;
      const topicEvents = byTopic.get(topic) || [];
      topicEvents.push(event);
      byTopic.set(topic, topicEvents);
    }
    
    // Publish to each topic
    const sends = [...byTopic.entries()].map(([topic, topicEvents]) =>
      this.producer.send({
        topic,
        compression: CompressionTypes.GZIP,
        messages: topicEvents.map(event => ({
          key: event.streamId,
          value: JSON.stringify(event),
          headers: {
            'event-type': event.eventType,
            'correlation-id': event.metadata.correlationId,
          },
        })),
      })
    );
    
    await Promise.all(sends);
  }
  
  async disconnect(): Promise<void> {
    await this.producer.disconnect();
  }
}
```

---

## 9. Database Schema

```sql
-- migrations/001_event_store.sql

-- Events table
CREATE TABLE IF NOT EXISTS events (
  global_position BIGSERIAL,
  event_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  stream_id VARCHAR(255) NOT NULL,
  event_type VARCHAR(255) NOT NULL,
  version INTEGER NOT NULL,
  data JSONB NOT NULL,
  metadata JSONB NOT NULL DEFAULT '{}',
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  CONSTRAINT events_stream_version_unique UNIQUE (stream_id, version)
);

CREATE INDEX idx_events_stream_id_version ON events(stream_id, version);
CREATE INDEX idx_events_event_type ON events(event_type);
CREATE INDEX idx_events_global_position ON events(global_position);
CREATE INDEX idx_events_created_at ON events(created_at);

-- Snapshots table
CREATE TABLE IF NOT EXISTS snapshots (
  stream_id VARCHAR(255) PRIMARY KEY,
  version INTEGER NOT NULL,
  state JSONB NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Projection checkpoints
CREATE TABLE IF NOT EXISTS projection_checkpoints (
  projection_name VARCHAR(255) PRIMARY KEY,
  position BIGINT NOT NULL DEFAULT 0,
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Read model tables
CREATE TABLE IF NOT EXISTS order_summaries (
  order_id UUID PRIMARY KEY,
  customer_id UUID NOT NULL,
  status VARCHAR(50) NOT NULL,
  total_amount DECIMAL(10,2) NOT NULL,
  item_count INTEGER NOT NULL DEFAULT 0,
  payment_id VARCHAR(255),
  tracking_number VARCHAR(255),
  created_at TIMESTAMPTZ NOT NULL,
  confirmed_at TIMESTAMPTZ,
  paid_at TIMESTAMPTZ,
  shipped_at TIMESTAMPTZ,
  cancelled_at TIMESTAMPTZ,
  version INTEGER NOT NULL DEFAULT 0
);

CREATE INDEX idx_order_summaries_customer_id ON order_summaries(customer_id);
CREATE INDEX idx_order_summaries_status ON order_summaries(status);
CREATE INDEX idx_order_summaries_created_at ON order_summaries(created_at);

CREATE TABLE IF NOT EXISTS order_items_view (
  id SERIAL PRIMARY KEY,
  order_id UUID NOT NULL REFERENCES order_summaries(order_id),
  product_id UUID NOT NULL,
  name VARCHAR(255) NOT NULL,
  quantity INTEGER NOT NULL,
  price DECIMAL(10,2) NOT NULL,
  UNIQUE(order_id, product_id)
);

CREATE INDEX idx_order_items_view_order_id ON order_items_view(order_id);
```

---

## 10. REST API สำหรับ CQRS

```typescript
// api/order-router.ts
import express from 'express';
import { OrderCommandHandler, OrderQueryHandler } from '../application/order-command-handler';

export function createOrderRouter(
  commandHandler: OrderCommandHandler,
  queryHandler: OrderQueryHandler
): express.Router {
  const router = express.Router();
  
  // Commands (Write side)
  router.post('/orders', async (req, res) => {
    try {
      const orderId = await commandHandler.handleCreateOrder(req.body);
      res.status(201).json({ orderId });
    } catch (err) {
      res.status(400).json({ error: (err as Error).message });
    }
  });
  
  router.post('/orders/:id/confirm', async (req, res) => {
    try {
      await commandHandler.handleConfirmOrder({ orderId: req.params.id });
      res.status(200).json({ success: true });
    } catch (err) {
      res.status(400).json({ error: (err as Error).message });
    }
  });
  
  router.post('/orders/:id/process-payment', async (req, res) => {
    try {
      await commandHandler.handleProcessPayment({
        orderId: req.params.id,
        ...req.body,
      });
      res.status(200).json({ success: true });
    } catch (err) {
      res.status(400).json({ error: (err as Error).message });
    }
  });
  
  router.post('/orders/:id/cancel', async (req, res) => {
    try {
      await commandHandler.handleCancelOrder({
        orderId: req.params.id,
        reason: req.body.reason,
      });
      res.status(200).json({ success: true });
    } catch (err) {
      res.status(400).json({ error: (err as Error).message });
    }
  });
  
  // Queries (Read side)
  router.get('/orders/:id', async (req, res) => {
    const order = await queryHandler.getOrderById(req.params.id);
    if (!order) return res.status(404).json({ error: 'Order not found' });
    res.json(order);
  });
  
  router.get('/orders/:id/history', async (req, res) => {
    const history = await queryHandler.getOrderHistory(req.params.id);
    res.json(history);
  });
  
  router.get('/customers/:customerId/orders', async (req, res) => {
    const orders = await queryHandler.getOrdersByCustomer(
      req.params.customerId,
      req.query.status as string,
      parseInt(req.query.page as string) || 1,
      parseInt(req.query.limit as string) || 20
    );
    res.json(orders);
  });
  
  return router;
}
```

---

## สรุป

Event Sourcing + CQRS ให้ประโยชน์สำคัญ:

| ด้าน | ประโยชน์ |
|------|----------|
| Audit Trail | ทุก state change ถูกบันทึกเป็น event |
| Time Travel | สามารถ replay กลับไปดู state ในอดีตได้ |
| Scalability | Write/Read model แยกกัน scale ได้อิสระ |
| Debugging | ดู sequence ของ events ช่วย debug ได้ดี |
| Integration | Events เป็น integration points ที่ชัดเจน |
| Flexibility | สร้าง read model ใหม่ได้โดย replay events |

เมื่อใช้:
- Domain มี complex business rules
- ต้องการ audit trail
- ต้องการ temporal queries
- Event-driven architecture
- Distributed systems ที่ต้องการ eventual consistency

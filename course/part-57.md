# Part 57: Asynchronous Patterns and Event-Driven Architecture

## บทนำ

Event-Driven Architecture (EDA) เป็น Architecture Pattern ที่ Services สื่อสารกัน
ผ่าน Events แทนการเรียกตรง ทำให้ระบบ Loosely Coupled, Scalable และ Resilient
บทนี้จะครอบคลุม Patterns ขั้นสูงที่ใช้ใน Production

---

## 1. Async Patterns Overview

### Fire-and-Forget

```typescript
// src/patterns/fire-and-forget.ts
import { Injectable, Logger } from '@nestjs/common';
import { InjectQueue } from '@nestjs/bull';
import { Queue } from 'bull';

@Injectable()
export class FireAndForgetService {
  private readonly logger = new Logger(FireAndForgetService.name);

  constructor(
    @InjectQueue('email') private readonly emailQueue: Queue,
    @InjectQueue('analytics') private readonly analyticsQueue: Queue
  ) {}

  // Fire-and-forget: ส่งแล้วไม่รอผล
  async sendWelcomeEmail(userId: string, email: string): Promise<void> {
    // ส่ง job ไปยัง queue โดยไม่รอผล
    await this.emailQueue.add(
      'welcome-email',
      { userId, email },
      {
        removeOnComplete: true,
        removeOnFail: false,
        attempts: 3,
        backoff: {
          type: 'exponential',
          delay: 2000,
        },
      }
    );
    
    this.logger.log(`Welcome email queued for user ${userId}`);
    // Return ทันทีโดยไม่รอ email ถูกส่ง
  }

  // Track analytics events แบบ fire-and-forget
  trackEvent(event: {
    type: string;
    userId?: string;
    properties: Record<string, unknown>;
  }): void {
    // Intentionally not awaiting
    this.analyticsQueue
      .add('track-event', event, { removeOnComplete: true })
      .catch(err => this.logger.error('Failed to queue analytics event:', err));
  }
}
```

### Request-Reply Pattern

```typescript
// src/patterns/request-reply.ts
import { Injectable, Logger } from '@nestjs/common';
import { RabbitMQClient } from '../messaging/rabbitmq.client';

@Injectable()
export class RequestReplyService {
  private readonly logger = new Logger(RequestReplyService.name);
  private pendingRequests = new Map<string, {
    resolve: (value: unknown) => void;
    reject: (error: Error) => void;
    timer: NodeJS.Timeout;
  }>();

  constructor(private readonly rabbitmq: RabbitMQClient) {}

  async initialize(): Promise<void> {
    // Setup reply queue
    const replyQueue = await this.rabbitmq.assertQueue('', {
      exclusive: true,
      autoDelete: true,
    });

    const channel = this.rabbitmq.getChannel();
    await channel.consume(replyQueue.queue, (msg) => {
      if (!msg) return;
      
      const correlationId = msg.properties.correlationId;
      const pending = this.pendingRequests.get(correlationId);
      
      if (pending) {
        clearTimeout(pending.timer);
        this.pendingRequests.delete(correlationId);
        
        const response = JSON.parse(msg.content.toString());
        if (response.success) {
          pending.resolve(response.data);
        } else {
          pending.reject(new Error(response.error || 'Request failed'));
        }
        
        channel.ack(msg);
      }
    });
  }

  async request<T>(
    service: string,
    operation: string,
    payload: unknown,
    timeoutMs = 30000
  ): Promise<T> {
    const correlationId = `${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
    const queue = `${service}.rpc`;

    return new Promise((resolve, reject) => {
      const timer = setTimeout(() => {
        this.pendingRequests.delete(correlationId);
        reject(new Error(`Request to ${service}.${operation} timed out after ${timeoutMs}ms`));
      }, timeoutMs);

      this.pendingRequests.set(correlationId, {
        resolve: resolve as (value: unknown) => void,
        reject,
        timer,
      });

      const channel = this.rabbitmq.getChannel();
      channel.sendToQueue(
        queue,
        Buffer.from(JSON.stringify({ operation, payload })),
        {
          correlationId,
          replyTo: 'amq.rabbitmq.reply-to',
          contentType: 'application/json',
        }
      );
    });
  }
}
```

---

## 2. Transactional Outbox Pattern (Full Implementation)

### Outbox Table Schema

```sql
-- migrations/create-outbox-table.sql
CREATE TABLE IF NOT EXISTS outbox_events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  aggregate_type VARCHAR(100) NOT NULL,
  aggregate_id VARCHAR(255) NOT NULL,
  event_type VARCHAR(200) NOT NULL,
  payload JSONB NOT NULL,
  headers JSONB DEFAULT '{}',
  status VARCHAR(20) NOT NULL DEFAULT 'pending',
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  processed_at TIMESTAMPTZ,
  failed_at TIMESTAMPTZ,
  retry_count INTEGER NOT NULL DEFAULT 0,
  max_retries INTEGER NOT NULL DEFAULT 3,
  error_message TEXT,
  scheduled_for TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_outbox_status_scheduled 
  ON outbox_events(status, scheduled_for) 
  WHERE status IN ('pending', 'failed');

CREATE INDEX idx_outbox_aggregate 
  ON outbox_events(aggregate_type, aggregate_id);
```

### Outbox Repository

```typescript
// src/outbox/outbox.repository.ts
import { Injectable } from '@nestjs/common';
import { DataSource, EntityManager } from 'typeorm';
import { OutboxEvent } from './outbox.entity';

@Injectable()
export class OutboxRepository {
  constructor(private readonly dataSource: DataSource) {}

  // บันทึก event พร้อมกับ transaction ของ business operation
  async saveEventInTransaction(
    manager: EntityManager,
    event: Partial<OutboxEvent>
  ): Promise<OutboxEvent> {
    const outboxEvent = manager.create(OutboxEvent, {
      ...event,
      status: 'pending',
      createdAt: new Date(),
    });
    
    return manager.save(OutboxEvent, outboxEvent);
  }

  // ดึง pending events สำหรับ processing
  async getPendingEvents(limit = 100): Promise<OutboxEvent[]> {
    return this.dataSource
      .getRepository(OutboxEvent)
      .createQueryBuilder('event')
      .where('event.status IN (:...statuses)', { statuses: ['pending', 'failed'] })
      .andWhere('event.scheduledFor <= :now', { now: new Date() })
      .andWhere('event.retryCount < event.maxRetries')
      .orderBy('event.scheduledFor', 'ASC')
      .limit(limit)
      .getMany();
  }

  async markAsProcessed(id: string): Promise<void> {
    await this.dataSource.getRepository(OutboxEvent).update(id, {
      status: 'processed',
      processedAt: new Date(),
    });
  }

  async markAsFailed(id: string, error: string, nextRetryAt?: Date): Promise<void> {
    const repo = this.dataSource.getRepository(OutboxEvent);
    const event = await repo.findOneBy({ id });
    
    if (!event) return;
    
    const newRetryCount = event.retryCount + 1;
    const shouldRetry = newRetryCount < event.maxRetries;
    
    await repo.update(id, {
      status: shouldRetry ? 'failed' : 'dead',
      failedAt: new Date(),
      retryCount: newRetryCount,
      errorMessage: error,
      scheduledFor: shouldRetry ? (nextRetryAt || this.calculateNextRetry(newRetryCount)) : undefined,
    });
  }

  private calculateNextRetry(retryCount: number): Date {
    // Exponential backoff: 2^n * 1000ms
    const delayMs = Math.min(Math.pow(2, retryCount) * 1000, 300000); // max 5 min
    return new Date(Date.now() + delayMs);
  }
}
```

### Outbox Processor

```typescript
// src/outbox/outbox.processor.ts
import { Injectable, Logger } from '@nestjs/common';
import { Cron } from '@nestjs/schedule';
import { DataSource } from 'typeorm';
import { OutboxRepository } from './outbox.repository';
import { EventPublisher } from '../events/event-publisher.service';
import { OutboxEvent } from './outbox.entity';

@Injectable()
export class OutboxProcessor {
  private readonly logger = new Logger(OutboxProcessor.name);
  private isProcessing = false;

  constructor(
    private readonly outboxRepository: OutboxRepository,
    private readonly eventPublisher: EventPublisher,
    private readonly dataSource: DataSource
  ) {}

  @Cron('*/5 * * * * *') // ทุก 5 วินาที
  async processOutbox(): Promise<void> {
    if (this.isProcessing) {
      this.logger.debug('Outbox processing already in progress, skipping');
      return;
    }

    this.isProcessing = true;

    try {
      const events = await this.outboxRepository.getPendingEvents(50);
      
      if (events.length === 0) return;
      
      this.logger.debug(`Processing ${events.length} outbox events`);
      
      for (const event of events) {
        await this.processEvent(event);
      }
    } catch (error) {
      this.logger.error('Error processing outbox:', error);
    } finally {
      this.isProcessing = false;
    }
  }

  private async processEvent(event: OutboxEvent): Promise<void> {
    try {
      await this.eventPublisher.publish({
        exchange: this.getExchangeForAggregate(event.aggregateType),
        routingKey: event.eventType,
        content: {
          id: event.id,
          aggregateType: event.aggregateType,
          aggregateId: event.aggregateId,
          eventType: event.eventType,
          payload: event.payload,
          occurredAt: event.createdAt,
        },
        options: {
          headers: event.headers,
          messageId: event.id,
        },
      });
      
      await this.outboxRepository.markAsProcessed(event.id);
      this.logger.debug(`Outbox event ${event.id} processed successfully`);
    } catch (error) {
      const errorMessage = error instanceof Error ? error.message : 'Unknown error';
      this.logger.warn(`Failed to process outbox event ${event.id}: ${errorMessage}`);
      await this.outboxRepository.markAsFailed(event.id, errorMessage);
    }
  }

  private getExchangeForAggregate(aggregateType: string): string {
    const exchangeMap: Record<string, string> = {
      Order: 'orders.events',
      Payment: 'payments.events',
      User: 'users.events',
      Product: 'products.events',
    };
    
    return exchangeMap[aggregateType] || 'events';
  }
}
```

### Order Service กับ Outbox Pattern

```typescript
// src/orders/order.service.ts
import { Injectable } from '@nestjs/common';
import { DataSource } from 'typeorm';
import { OutboxRepository } from '../outbox/outbox.repository';
import { Order } from './order.entity';

@Injectable()
export class OrderService {
  constructor(
    private readonly dataSource: DataSource,
    private readonly outboxRepository: OutboxRepository
  ) {}

  async createOrder(createOrderDto: {
    customerId: string;
    items: Array<{ productId: string; quantity: number; price: number }>;
  }): Promise<Order> {
    // Transaction เดียวสำหรับทั้ง business operation และ outbox event
    return this.dataSource.transaction(async (manager) => {
      // 1. สร้าง Order
      const total = createOrderDto.items.reduce(
        (sum, item) => sum + item.quantity * item.price,
        0
      );
      
      const order = manager.create(Order, {
        customerId: createOrderDto.customerId,
        items: createOrderDto.items,
        total,
        status: 'pending',
      });
      
      const savedOrder = await manager.save(Order, order);
      
      // 2. บันทึก Outbox Event ในTransaction เดียวกัน
      await this.outboxRepository.saveEventInTransaction(manager, {
        aggregateType: 'Order',
        aggregateId: savedOrder.id,
        eventType: 'order.created',
        payload: {
          orderId: savedOrder.id,
          customerId: savedOrder.customerId,
          items: savedOrder.items,
          total: savedOrder.total,
          status: savedOrder.status,
        },
        headers: {
          'content-type': 'application/json',
          'schema-version': '1.0',
        },
      });
      
      return savedOrder;
      // ถ้า transaction fail ทั้ง Order และ Outbox event จะถูก rollback พร้อมกัน
    });
  }
}
```

---

## 3. Inbox Pattern สำหรับ Idempotent Consumers

```typescript
// src/inbox/inbox.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { DataSource } from 'typeorm';
import { InboxEvent } from './inbox.entity';

@Injectable()
export class InboxService {
  private readonly logger = new Logger(InboxService.name);

  constructor(private readonly dataSource: DataSource) {}

  async processIdempotent(
    messageId: string,
    eventType: string,
    handler: () => Promise<void>
  ): Promise<{ processed: boolean; alreadyProcessed: boolean }> {
    const repo = this.dataSource.getRepository(InboxEvent);
    
    // ตรวจสอบว่า message นี้เคย process แล้วหรือยัง
    const existing = await repo.findOneBy({ messageId });
    
    if (existing) {
      if (existing.status === 'processed') {
        this.logger.debug(`Message ${messageId} already processed, skipping`);
        return { processed: false, alreadyProcessed: true };
      }
      
      if (existing.status === 'processing') {
        this.logger.warn(`Message ${messageId} is being processed by another instance`);
        return { processed: false, alreadyProcessed: false };
      }
    }

    // สร้าง inbox record เพื่อ "lock" message นี้
    try {
      await repo.insert({
        messageId,
        eventType,
        status: 'processing',
        receivedAt: new Date(),
      });
    } catch (e) {
      // Duplicate key error - message already being processed
      this.logger.warn(`Race condition detected for message ${messageId}`);
      return { processed: false, alreadyProcessed: false };
    }

    // Process message
    try {
      await handler();
      
      await repo.update({ messageId }, {
        status: 'processed',
        processedAt: new Date(),
      });
      
      return { processed: true, alreadyProcessed: false };
    } catch (error) {
      await repo.update({ messageId }, {
        status: 'failed',
        errorMessage: error instanceof Error ? error.message : 'Unknown error',
      });
      
      throw error;
    }
  }
}
```

---

## 4. Choreography vs Orchestration Sagas

### Choreography Saga

```typescript
// src/sagas/choreography/order-saga.events.ts
export const OrderSagaEvents = {
  ORDER_CREATED: 'order.created',
  PAYMENT_INITIATED: 'payment.initiated',
  PAYMENT_COMPLETED: 'payment.completed',
  PAYMENT_FAILED: 'payment.failed',
  INVENTORY_RESERVED: 'inventory.reserved',
  INVENTORY_FAILED: 'inventory.failed',
  ORDER_CONFIRMED: 'order.confirmed',
  ORDER_CANCELLED: 'order.cancelled',
} as const;

// Order Service
// src/sagas/choreography/order-choreography.service.ts
@Injectable()
export class OrderChoreographyService {
  constructor(
    private readonly eventBus: EventBusService,
    private readonly orderRepository: OrderRepository
  ) {}

  // Listen สำหรับ payment events
  @EventHandler(OrderSagaEvents.PAYMENT_COMPLETED)
  async onPaymentCompleted(event: { orderId: string; transactionId: string }): Promise<void> {
    const order = await this.orderRepository.findById(event.orderId);
    
    if (!order || order.status !== 'payment_pending') return;
    
    await this.orderRepository.updateStatus(event.orderId, 'payment_completed');
    
    // Publish event เพื่อให้ Inventory Service ทำงานต่อ
    await this.eventBus.publish(OrderSagaEvents.ORDER_CONFIRMED, {
      orderId: event.orderId,
      customerId: order.customerId,
      items: order.items,
    });
  }

  @EventHandler(OrderSagaEvents.PAYMENT_FAILED)
  async onPaymentFailed(event: { orderId: string; reason: string }): Promise<void> {
    await this.orderRepository.updateStatus(event.orderId, 'cancelled');
    
    await this.eventBus.publish(OrderSagaEvents.ORDER_CANCELLED, {
      orderId: event.orderId,
      reason: `Payment failed: ${event.reason}`,
    });
  }

  @EventHandler(OrderSagaEvents.INVENTORY_FAILED)
  async onInventoryFailed(event: { orderId: string; reason: string }): Promise<void> {
    await this.orderRepository.updateStatus(event.orderId, 'cancelled');
    
    // Trigger compensation: คืนเงิน
    await this.eventBus.publish('payment.refund.requested', {
      orderId: event.orderId,
      reason: event.reason,
    });
  }
}
```

### Orchestration Saga

```typescript
// src/sagas/orchestration/order-orchestrator.ts
import { Injectable, Logger } from '@nestjs/common';

type SagaState = 
  | 'started'
  | 'payment_pending'
  | 'payment_completed'
  | 'inventory_pending'
  | 'completed'
  | 'compensating'
  | 'cancelled';

interface SagaData {
  orderId: string;
  customerId: string;
  items: Array<{ productId: string; quantity: number; price: number }>;
  total: number;
  paymentTransactionId?: string;
  inventoryReservationId?: string;
}

@Injectable()
export class OrderSagaOrchestrator {
  private readonly logger = new Logger(OrderSagaOrchestrator.name);

  constructor(
    private readonly sagaRepository: SagaRepository,
    private readonly paymentService: PaymentServiceClient,
    private readonly inventoryService: InventoryServiceClient,
    private readonly notificationService: NotificationServiceClient
  ) {}

  async startOrderSaga(data: SagaData): Promise<void> {
    const sagaId = `order-saga-${data.orderId}`;
    
    // บันทึก saga state
    await this.sagaRepository.save({
      id: sagaId,
      type: 'OrderSaga',
      state: 'started',
      data,
    });

    await this.executeNextStep(sagaId, 'started', data);
  }

  private async executeNextStep(
    sagaId: string,
    currentState: SagaState,
    data: SagaData
  ): Promise<void> {
    try {
      switch (currentState) {
        case 'started':
          await this.step1_initiatePayment(sagaId, data);
          break;
          
        case 'payment_completed':
          await this.step2_reserveInventory(sagaId, data);
          break;
          
        case 'inventory_pending':
          await this.step3_completeOrder(sagaId, data);
          break;
          
        default:
          this.logger.warn(`Unknown saga state: ${currentState}`);
      }
    } catch (error) {
      this.logger.error(`Saga ${sagaId} failed at state ${currentState}:`, error);
      await this.startCompensation(sagaId, currentState, data);
    }
  }

  private async step1_initiatePayment(sagaId: string, data: SagaData): Promise<void> {
    await this.sagaRepository.updateState(sagaId, 'payment_pending');
    
    const result = await this.paymentService.initiatePayment({
      orderId: data.orderId,
      customerId: data.customerId,
      amount: data.total,
    });
    
    await this.sagaRepository.updateData(sagaId, {
      ...data,
      paymentTransactionId: result.transactionId,
    });
    
    await this.sagaRepository.updateState(sagaId, 'payment_completed');
    await this.executeNextStep(sagaId, 'payment_completed', {
      ...data,
      paymentTransactionId: result.transactionId,
    });
  }

  private async step2_reserveInventory(sagaId: string, data: SagaData): Promise<void> {
    await this.sagaRepository.updateState(sagaId, 'inventory_pending');
    
    const result = await this.inventoryService.reserveItems({
      orderId: data.orderId,
      items: data.items,
    });
    
    await this.sagaRepository.updateData(sagaId, {
      ...data,
      inventoryReservationId: result.reservationId,
    });
    
    await this.executeNextStep(sagaId, 'inventory_pending', data);
  }

  private async step3_completeOrder(sagaId: string, data: SagaData): Promise<void> {
    await this.sagaRepository.updateState(sagaId, 'completed');
    
    await this.notificationService.sendOrderConfirmation({
      orderId: data.orderId,
      customerId: data.customerId,
    });
    
    this.logger.log(`Saga ${sagaId} completed successfully`);
  }

  private async startCompensation(
    sagaId: string,
    failedState: SagaState,
    data: SagaData
  ): Promise<void> {
    await this.sagaRepository.updateState(sagaId, 'compensating');
    
    this.logger.warn(`Starting compensation for saga ${sagaId} at state ${failedState}`);

    switch (failedState) {
      case 'inventory_pending':
        // Inventory failed - refund payment
        if (data.paymentTransactionId) {
          await this.paymentService.refund({
            transactionId: data.paymentTransactionId,
            reason: 'Inventory reservation failed',
          });
        }
        break;
        
      case 'payment_completed':
        // Nothing to compensate if payment just completed and inventory not yet started
        break;
    }
    
    await this.sagaRepository.updateState(sagaId, 'cancelled');
  }
}
```

---

## 5. Event-Carried State Transfer

```typescript
// src/patterns/event-carried-state.ts
// แทนที่จะ call API เพื่อดึงข้อมูล User เราส่งข้อมูลที่จำเป็นไปใน Event

export interface OrderCreatedEvent {
  // Identity fields
  eventId: string;
  eventType: 'order.created';
  occurredAt: string;
  
  // Order data
  orderId: string;
  
  // Embedded customer data (Event-Carried State Transfer)
  customer: {
    id: string;
    email: string;
    name: string;
    phone?: string;
    address: {
      street: string;
      city: string;
      postalCode: string;
      country: string;
    };
    tier: 'standard' | 'premium' | 'vip';
  };
  
  // Embedded product data
  items: Array<{
    productId: string;
    productName: string;
    productSku: string;
    quantity: number;
    unitPrice: number;
    totalPrice: number;
    category: string;
  }>;
  
  total: number;
  currency: string;
  
  // Shipping info
  shippingMethod: string;
  estimatedDelivery: string;
}
```

---

## 6. Change Data Capture (CDC) with Debezium

### Debezium Configuration

```yaml
# debezium/postgres-connector.json
{
  "name": "orders-postgres-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "postgres",
    "database.port": "5432",
    "database.user": "debezium",
    "database.password": "debezium_password",
    "database.dbname": "orders_db",
    "database.server.name": "orders",
    "table.include.list": "public.orders,public.order_items,public.customers",
    "plugin.name": "pgoutput",
    "slot.name": "debezium_orders",
    "publication.name": "debezium_publication",
    "topic.prefix": "cdc.orders",
    "transforms": "unwrap,route",
    "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
    "transforms.unwrap.drop.tombstones": "false",
    "transforms.unwrap.delete.handling.mode": "rewrite",
    "transforms.route.type": "org.apache.kafka.connect.transforms.ReplaceField$Value",
    "key.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "key.converter.schemas.enable": "false",
    "value.converter.schemas.enable": "false"
  }
}
```

```yaml
# docker-compose.debezium.yml
version: '3.8'

services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1

  debezium:
    image: debezium/connect:2.4
    depends_on:
      - kafka
    ports:
      - "8083:8083"
    environment:
      BOOTSTRAP_SERVERS: kafka:29092
      GROUP_ID: debezium-connect
      CONFIG_STORAGE_TOPIC: debezium.configs
      OFFSET_STORAGE_TOPIC: debezium.offsets
      STATUS_STORAGE_TOPIC: debezium.status
```

### CDC Event Consumer ใน TypeScript

```typescript
// src/cdc/cdc-consumer.ts
import { Injectable, Logger, OnModuleInit } from '@nestjs/common';
import { Kafka, Consumer, KafkaMessage } from 'kafkajs';

interface DebeziumEvent<T> {
  before: T | null;
  after: T | null;
  op: 'c' | 'u' | 'd' | 'r';  // create, update, delete, read
  ts_ms: number;
  source: {
    db: string;
    table: string;
  };
}

@Injectable()
export class CDCConsumerService implements OnModuleInit {
  private readonly logger = new Logger(CDCConsumerService.name);
  private consumer: Consumer;

  constructor(private readonly orderProjectionService: OrderProjectionService) {
    const kafka = new Kafka({
      clientId: 'order-cdc-consumer',
      brokers: ['kafka:9092'],
    });
    
    this.consumer = kafka.consumer({ groupId: 'order-projections' });
  }

  async onModuleInit(): Promise<void> {
    await this.consumer.connect();
    
    await this.consumer.subscribe({
      topics: [
        'cdc.orders.public.orders',
        'cdc.orders.public.order_items',
      ],
      fromBeginning: false,
    });
    
    await this.consumer.run({
      eachMessage: async ({ topic, message }) => {
        await this.processMessage(topic, message);
      },
    });
  }

  private async processMessage(topic: string, message: KafkaMessage): Promise<void> {
    if (!message.value) return;
    
    try {
      const event = JSON.parse(message.value.toString()) as DebeziumEvent<Record<string, unknown>>;
      
      const tableName = topic.split('.').pop();
      
      switch (tableName) {
        case 'orders':
          await this.handleOrderChange(event);
          break;
        case 'order_items':
          await this.handleOrderItemChange(event);
          break;
      }
    } catch (error) {
      this.logger.error(`Failed to process CDC message from ${topic}:`, error);
    }
  }

  private async handleOrderChange(event: DebeziumEvent<Record<string, unknown>>): Promise<void> {
    switch (event.op) {
      case 'c':
        await this.orderProjectionService.onCreate(event.after!);
        break;
      case 'u':
        await this.orderProjectionService.onUpdate(event.before!, event.after!);
        break;
      case 'd':
        await this.orderProjectionService.onDelete(event.before!);
        break;
    }
  }

  private async handleOrderItemChange(event: DebeziumEvent<Record<string, unknown>>): Promise<void> {
    switch (event.op) {
      case 'c':
        await this.orderProjectionService.onItemAdded(event.after!);
        break;
      case 'u':
        await this.orderProjectionService.onItemUpdated(event.before!, event.after!);
        break;
      case 'd':
        await this.orderProjectionService.onItemRemoved(event.before!);
        break;
    }
  }
}
```

---

## 7. AsyncAPI Specification

```yaml
# asyncapi.yaml
asyncapi: '2.6.0'
info:
  title: Order Service Events API
  version: '1.0.0'
  description: Event definitions สำหรับ Order Service
  contact:
    name: Platform Team
    email: platform@company.com

servers:
  production:
    url: rabbitmq.production.svc.cluster.local:5672
    protocol: amqp
    description: Production RabbitMQ
    security:
      - userPassword: []
  
  staging:
    url: rabbitmq.staging.svc.cluster.local:5672
    protocol: amqp
    description: Staging RabbitMQ

defaultContentType: application/json

channels:
  order/created:
    description: Published เมื่อมีการสร้าง Order ใหม่
    subscribe:
      summary: Order Created Event
      operationId: onOrderCreated
      message:
        $ref: '#/components/messages/OrderCreated'
    bindings:
      amqp:
        is: routingKey
        exchange:
          name: orders.events
          type: topic
          durable: true

  order/cancelled:
    description: Published เมื่อ Order ถูก cancel
    subscribe:
      summary: Order Cancelled Event
      operationId: onOrderCancelled
      message:
        $ref: '#/components/messages/OrderCancelled'

  payment/requested:
    description: Published เพื่อ request payment processing
    publish:
      summary: Request Payment Processing
      operationId: requestPayment
      message:
        $ref: '#/components/messages/PaymentRequested'

components:
  messages:
    OrderCreated:
      name: OrderCreated
      title: Order Created
      summary: สร้าง Order ใหม่เรียบร้อยแล้ว
      contentType: application/json
      headers:
        type: object
        properties:
          correlationId:
            description: Unique ID สำหรับ tracing
            type: string
            format: uuid
          schemaVersion:
            description: Version ของ schema
            type: string
            default: "1.0"
      payload:
        $ref: '#/components/schemas/OrderCreatedPayload'
        
    OrderCancelled:
      name: OrderCancelled
      payload:
        $ref: '#/components/schemas/OrderCancelledPayload'
        
    PaymentRequested:
      name: PaymentRequested
      payload:
        $ref: '#/components/schemas/PaymentRequestedPayload'
        
  schemas:
    OrderCreatedPayload:
      type: object
      required:
        - orderId
        - customerId
        - items
        - total
        - createdAt
      properties:
        orderId:
          type: string
          format: uuid
          description: Unique identifier ของ Order
        customerId:
          type: string
          format: uuid
        items:
          type: array
          items:
            $ref: '#/components/schemas/OrderItem'
        total:
          type: number
          format: float
          minimum: 0
        currency:
          type: string
          default: THB
        createdAt:
          type: string
          format: date-time
          
    OrderItem:
      type: object
      required:
        - productId
        - quantity
        - unitPrice
      properties:
        productId:
          type: string
          format: uuid
        productName:
          type: string
        quantity:
          type: integer
          minimum: 1
        unitPrice:
          type: number
          minimum: 0
        totalPrice:
          type: number
          minimum: 0
          
    OrderCancelledPayload:
      type: object
      required:
        - orderId
        - reason
        - cancelledAt
      properties:
        orderId:
          type: string
          format: uuid
        reason:
          type: string
        cancelledAt:
          type: string
          format: date-time
          
    PaymentRequestedPayload:
      type: object
      required:
        - orderId
        - amount
        - currency
        - customerId
      properties:
        orderId:
          type: string
          format: uuid
        amount:
          type: number
          minimum: 0
        currency:
          type: string
          default: THB
        customerId:
          type: string
          format: uuid
        paymentMethod:
          type: string
          enum: [credit_card, bank_transfer, qr_code]
          
  securitySchemes:
    userPassword:
      type: userPassword
```

---

## 8. Compensating Transactions

```typescript
// src/sagas/compensating-transactions.ts
export interface CompensationStep {
  name: string;
  compensate: () => Promise<void>;
}

export class CompensationManager {
  private readonly logger = console;
  private executedSteps: CompensationStep[] = [];

  async execute(
    step: CompensationStep,
    operation: () => Promise<void>
  ): Promise<void> {
    try {
      await operation();
      this.executedSteps.push(step);
    } catch (error) {
      this.logger.error(`Step ${step.name} failed:`, error);
      await this.compensateAll();
      throw error;
    }
  }

  private async compensateAll(): Promise<void> {
    this.logger.warn(`Starting compensation for ${this.executedSteps.length} steps`);
    
    // Compensate ในลำดับย้อนกลับ
    const stepsToCompensate = [...this.executedSteps].reverse();
    
    for (const step of stepsToCompensate) {
      try {
        this.logger.warn(`Compensating step: ${step.name}`);
        await step.compensate();
      } catch (error) {
        this.logger.error(`Compensation failed for step ${step.name}:`, error);
        // Log and continue with other compensations
      }
    }
    
    this.executedSteps = [];
  }
}

// การใช้งาน
async function createOrderWithCompensation(orderData: {
  customerId: string;
  items: Array<{ productId: string; quantity: number; price: number }>;
  total: number;
}) {
  const compensation = new CompensationManager();
  let orderId: string | undefined;
  let paymentId: string | undefined;

  await compensation.execute(
    {
      name: 'create-order',
      compensate: async () => {
        if (orderId) await cancelOrder(orderId);
      },
    },
    async () => {
      orderId = await createOrder(orderData);
    }
  );

  await compensation.execute(
    {
      name: 'process-payment',
      compensate: async () => {
        if (paymentId) await refundPayment(paymentId);
      },
    },
    async () => {
      paymentId = await processPayment({ orderId: orderId!, amount: orderData.total });
    }
  );

  await compensation.execute(
    {
      name: 'reserve-inventory',
      compensate: async () => {
        if (orderId) await releaseInventory(orderId);
      },
    },
    async () => {
      await reserveInventory(orderId!, orderData.items);
    }
  );
}

// Placeholder functions
async function cancelOrder(id: string): Promise<void> { console.log('Cancelling order', id); }
async function refundPayment(id: string): Promise<void> { console.log('Refunding payment', id); }
async function releaseInventory(id: string): Promise<void> { console.log('Releasing inventory', id); }
async function createOrder(data: unknown): Promise<string> { return 'order-123'; }
async function processPayment(data: unknown): Promise<string> { return 'payment-123'; }
async function reserveInventory(orderId: string, items: unknown): Promise<void> { }
```

---

## สรุป

| Pattern | รูปแบบ | เหมาะกับ |
|---------|--------|---------|
| Fire-and-Forget | ส่งแล้วไม่รอผล | Email, Notifications, Analytics |
| Request-Reply | ส่งและรอผลตอบกลับ | Cross-service queries |
| Choreography Saga | Services ตัดสินใจเอง | Simple workflows, loose coupling |
| Orchestration Saga | Central coordinator | Complex workflows, ง่ายต่อ debugging |
| Transactional Outbox | At-least-once delivery | Critical business events |
| Inbox Pattern | Idempotent processing | ป้องกัน duplicate processing |
| CDC | Database change capture | Real-time data sync |
| Event-Carried State | ส่งข้อมูลใน event | ลด API calls, improve decoupling |
| Compensating Transactions | Rollback distributed ops | Long-running transactions |
| AsyncAPI | API documentation | Cross-team communication |

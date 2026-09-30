# Part 57: Asynchronous Patterns and Event-Driven Architecture

ในบทนี้เราจะเรียนรู้ Async Communication Patterns ขั้นสูงสำหรับ Microservices ครอบคลุม Request-Reply over RabbitMQ, Correlation ID Tracking, Message Ordering, Idempotent Consumer, At-least-once Delivery, Outbox Pattern, Inbox Pattern, และ Message Schema Versioning

---

## 1. Request-Reply Pattern over RabbitMQ

Request-Reply ช่วยให้ services สามารถสื่อสารแบบ synchronous-like ผ่าน message broker โดยไม่ต้องใช้ HTTP

```typescript
// src/messaging/request-reply/rpc-client.ts
import { Channel, ConsumeMessage } from 'amqplib';
import { randomUUID } from 'crypto';

interface PendingRequest {
  resolve: (value: unknown) => void;
  reject: (reason: Error) => void;
  timeoutId: NodeJS.Timeout;
}

export class RpcClient {
  private readonly replyQueue: string;
  private readonly pendingRequests = new Map<string, PendingRequest>();
  private initialized = false;
  
  constructor(
    private readonly channel: Channel,
    private readonly defaultTimeoutMs: number = 30000
  ) {
    this.replyQueue = `rpc.reply.${process.env.SERVICE_NAME}.${randomUUID()}`;
  }
  
  async initialize(): Promise<void> {
    if (this.initialized) return;
    
    // Create exclusive reply queue
    await this.channel.assertQueue(this.replyQueue, {
      exclusive: true,
      autoDelete: true,
      durable: false,
    });
    
    // Start consuming replies
    this.channel.consume(
      this.replyQueue,
      (msg) => this.handleReply(msg),
      { noAck: true }
    );
    
    this.initialized = true;
    log.info('RPC client initialized', { replyQueue: this.replyQueue });
  }
  
  async call<TRequest, TResponse>(
    targetQueue: string,
    request: TRequest,
    options: {
      timeoutMs?: number;
      correlationId?: string;
      headers?: Record<string, string>;
    } = {}
  ): Promise<TResponse> {
    await this.initialize();
    
    const correlationId = options.correlationId || randomUUID();
    const timeoutMs = options.timeoutMs || this.defaultTimeoutMs;
    
    return new Promise<TResponse>((resolve, reject) => {
      const timeoutId = setTimeout(() => {
        this.pendingRequests.delete(correlationId);
        reject(new Error(`RPC timeout after ${timeoutMs}ms for ${targetQueue}`));
      }, timeoutMs);
      
      this.pendingRequests.set(correlationId, {
        resolve: resolve as (value: unknown) => void,
        reject,
        timeoutId,
      });
      
      this.channel.sendToQueue(
        targetQueue,
        Buffer.from(JSON.stringify(request)),
        {
          correlationId,
          replyTo: this.replyQueue,
          contentType: 'application/json',
          messageId: randomUUID(),
          timestamp: Math.floor(Date.now() / 1000),
          expiration: timeoutMs.toString(),
          headers: {
            'x-source-service': process.env.SERVICE_NAME,
            ...options.headers,
          },
        }
      );
      
      log.info('RPC request sent', {
        targetQueue,
        correlationId,
        timeout: timeoutMs,
      });
    });
  }
  
  private handleReply(msg: ConsumeMessage | null): void {
    if (!msg) return;
    
    const correlationId = msg.properties.correlationId;
    const pending = this.pendingRequests.get(correlationId);
    
    if (!pending) {
      log.warn('Received reply for unknown correlation ID', { correlationId });
      return;
    }
    
    this.pendingRequests.delete(correlationId);
    clearTimeout(pending.timeoutId);
    
    try {
      const response = JSON.parse(msg.content.toString());
      
      if (response.error) {
        pending.reject(new RpcError(response.error.message, response.error.code));
      } else {
        pending.resolve(response.data);
      }
    } catch (error) {
      pending.reject(new Error('Failed to parse RPC response'));
    }
  }
  
  async close(): Promise<void> {
    // Reject all pending requests
    for (const [correlationId, pending] of this.pendingRequests) {
      clearTimeout(pending.timeoutId);
      pending.reject(new Error('RPC client closed'));
    }
    this.pendingRequests.clear();
  }
}

export class RpcError extends Error {
  constructor(message: string, public readonly code: string) {
    super(message);
    this.name = 'RpcError';
  }
}

// RPC Server: handles incoming requests and sends replies
export class RpcServer {
  constructor(
    private readonly channel: Channel,
    private readonly queueName: string
  ) {}
  
  async registerHandler<TRequest, TResponse>(
    handler: (request: TRequest, meta: { correlationId: string; messageId: string }) => Promise<TResponse>
  ): Promise<void> {
    await this.channel.assertQueue(this.queueName, {
      durable: true,
      arguments: {
        'x-message-ttl': 60000, // Don't process stale requests
      },
    });
    
    await this.channel.prefetch(1); // Process one request at a time
    
    this.channel.consume(this.queueName, async (msg) => {
      if (!msg) return;
      
      const correlationId = msg.properties.correlationId;
      const replyTo = msg.properties.replyTo;
      
      if (!correlationId || !replyTo) {
        log.warn('RPC request missing correlationId or replyTo', {
          messageId: msg.properties.messageId,
        });
        this.channel.ack(msg);
        return;
      }
      
      const startTime = Date.now();
      
      try {
        const request = JSON.parse(msg.content.toString()) as TRequest;
        
        const response = await handler(request, {
          correlationId,
          messageId: msg.properties.messageId,
        });
        
        this.channel.sendToQueue(
          replyTo,
          Buffer.from(JSON.stringify({ data: response })),
          {
            correlationId,
            contentType: 'application/json',
            headers: {
              'x-processing-time-ms': Date.now() - startTime,
              'x-processed-by': process.env.SERVICE_NAME,
            },
          }
        );
        
        this.channel.ack(msg);
      } catch (error) {
        const err = error as Error;
        
        // Send error response
        this.channel.sendToQueue(
          replyTo,
          Buffer.from(JSON.stringify({
            error: {
              message: err.message,
              code: (error as any).code || 'INTERNAL_ERROR',
            },
          })),
          {
            correlationId,
            contentType: 'application/json',
          }
        );
        
        this.channel.ack(msg);
      }
    });
    
    log.info('RPC server registered', { queue: this.queueName });
  }
}

// Usage example
const rpcClient = new RpcClient(channel);

// Order service calling inventory service
const availability = await rpcClient.call<
  { productIds: string[] },
  { available: Record<string, number> }
>(
  'inventory.check-availability',
  { productIds: ['prod-1', 'prod-2'] },
  { timeoutMs: 5000 }
);
```

---

## 2. Correlation ID Tracking

```typescript
// src/messaging/correlation-tracking.ts
import { AsyncLocalStorage } from 'async_hooks';
import { Request, Response, NextFunction } from 'express';
import { randomUUID } from 'crypto';

interface TraceContext {
  correlationId: string;
  requestId: string;
  traceId?: string;
  userId?: string;
  sessionId?: string;
}

const asyncStorage = new AsyncLocalStorage<TraceContext>();

// Middleware to set correlation context
export function correlationMiddleware() {
  return (req: Request, res: Response, next: NextFunction) => {
    const correlationId =
      (req.headers['x-correlation-id'] as string) ||
      (req.headers['x-request-id'] as string) ||
      randomUUID();
    
    const context: TraceContext = {
      correlationId,
      requestId: randomUUID(),
      traceId: req.headers['traceparent'] as string,
      userId: req.user?.id,
    };
    
    res.setHeader('x-correlation-id', correlationId);
    res.setHeader('x-request-id', context.requestId);
    
    asyncStorage.run(context, () => next());
  };
}

// Get current correlation context
export function getTraceContext(): TraceContext | null {
  return asyncStorage.getStore() ?? null;
}

export function getCorrelationId(): string {
  return getTraceContext()?.correlationId || randomUUID();
}

// Inject correlation ID into all outgoing messages
export function enrichMessageWithContext(
  payload: Record<string, unknown>,
  additionalHeaders?: Record<string, string>
): { payload: Record<string, unknown>; headers: Record<string, string> } {
  const context = getTraceContext();
  
  return {
    payload: {
      ...payload,
      _context: {
        correlationId: context?.correlationId,
        requestId: context?.requestId,
        userId: context?.userId,
        timestamp: new Date().toISOString(),
      },
    },
    headers: {
      'x-correlation-id': context?.correlationId || '',
      'x-request-id': context?.requestId || '',
      'x-user-id': context?.userId || '',
      'x-trace-id': context?.traceId || '',
      ...additionalHeaders,
    },
  };
}

// Extract correlation context from incoming message
export function extractContextFromMessage(
  msg: { properties: { headers?: Record<string, string> }; content: Buffer }
): TraceContext {
  const headers = msg.properties.headers || {};
  const payload = JSON.parse(msg.content.toString());
  
  return {
    correlationId:
      headers['x-correlation-id'] ||
      payload._context?.correlationId ||
      randomUUID(),
    requestId:
      headers['x-request-id'] ||
      payload._context?.requestId ||
      randomUUID(),
    traceId: headers['x-trace-id'],
    userId: headers['x-user-id'] || payload._context?.userId,
  };
}

// Run message handler with correlation context
export async function withMessageContext<T>(
  msg: { properties: any; content: Buffer },
  handler: () => Promise<T>
): Promise<T> {
  const context = extractContextFromMessage(msg);
  return asyncStorage.run(context, handler);
}
```

---

## 3. Message Ordering Guarantees

```typescript
// src/messaging/ordering.ts
import { Channel, ConsumeMessage } from 'amqplib';
import { Redis } from 'ioredis';

// Pattern: Sequence number + ordering buffer
interface OrderedMessage<T> {
  sequenceNumber: number;
  partitionKey: string;
  payload: T;
  timestamp: string;
}

export class OrderingBuffer<T> {
  private readonly buffer = new Map<number, OrderedMessage<T>>();
  private expectedSequence: number;
  private readonly maxBufferSize: number;
  private gapTimer: NodeJS.Timeout | null = null;
  
  constructor(
    private readonly startSequence: number,
    private readonly onDelivered: (msg: OrderedMessage<T>) => Promise<void>,
    private readonly onGapTimeout: (gap: number[]) => void,
    maxBufferSize = 1000,
    private readonly gapTimeoutMs = 5000
  ) {
    this.expectedSequence = startSequence;
    this.maxBufferSize = maxBufferSize;
  }
  
  async receive(msg: OrderedMessage<T>): Promise<void> {
    if (msg.sequenceNumber < this.expectedSequence) {
      // Duplicate - already processed
      log.warn('Received duplicate message', {
        received: msg.sequenceNumber,
        expected: this.expectedSequence,
      });
      return;
    }
    
    if (msg.sequenceNumber === this.expectedSequence) {
      // In-order message
      await this.deliverAndDrain(msg);
    } else {
      // Out-of-order: buffer it
      if (this.buffer.size >= this.maxBufferSize) {
        throw new Error(`Ordering buffer overflow at sequence ${msg.sequenceNumber}`);
      }
      
      this.buffer.set(msg.sequenceNumber, msg);
      this.startGapTimer();
    }
  }
  
  private async deliverAndDrain(msg: OrderedMessage<T>): Promise<void> {
    await this.onDelivered(msg);
    this.expectedSequence++;
    
    // Drain buffered in-order messages
    while (this.buffer.has(this.expectedSequence)) {
      const next = this.buffer.get(this.expectedSequence)!;
      this.buffer.delete(this.expectedSequence);
      await this.onDelivered(next);
      this.expectedSequence++;
    }
    
    if (this.buffer.size === 0 && this.gapTimer) {
      clearTimeout(this.gapTimer);
      this.gapTimer = null;
    }
  }
  
  private startGapTimer(): void {
    if (this.gapTimer) return;
    
    this.gapTimer = setTimeout(() => {
      const gaps: number[] = [];
      const bufferKeys = [...this.buffer.keys()].sort((a, b) => a - b);
      
      // Find missing sequence numbers
      for (let seq = this.expectedSequence; seq < bufferKeys[0]; seq++) {
        gaps.push(seq);
      }
      
      if (gaps.length > 0) {
        log.warn('Sequence gap detected', { gaps, expectedSequence: this.expectedSequence });
        this.onGapTimeout(gaps);
      }
    }, this.gapTimeoutMs);
  }
}

// Kafka-style partitioned ordering with RabbitMQ
export class PartitionedConsumer<T> {
  private readonly buffers = new Map<string, OrderingBuffer<T>>();
  
  async processMessage(
    channel: Channel,
    msg: ConsumeMessage,
    handler: (payload: T) => Promise<void>
  ): Promise<void> {
    const partitionKey = msg.properties.headers?.['x-partition-key'] as string;
    const sequenceNumber = parseInt(msg.properties.headers?.['x-sequence-number'] as string);
    
    if (!partitionKey || isNaN(sequenceNumber)) {
      // No ordering guarantee needed
      const payload = JSON.parse(msg.content.toString()) as T;
      await handler(payload);
      channel.ack(msg);
      return;
    }
    
    let buffer = this.buffers.get(partitionKey);
    if (!buffer) {
      buffer = new OrderingBuffer<T>(
        sequenceNumber,
        async (orderedMsg) => {
          await handler(orderedMsg.payload);
        },
        (gaps) => {
          log.warn('Gap in message sequence', { partitionKey, gaps });
        }
      );
      this.buffers.set(partitionKey, buffer);
    }
    
    const ordered: OrderedMessage<T> = {
      sequenceNumber,
      partitionKey,
      payload: JSON.parse(msg.content.toString()),
      timestamp: new Date().toISOString(),
    };
    
    await buffer.receive(ordered);
    channel.ack(msg);
  }
}
```

---

## 4. Idempotent Consumer Implementation

```typescript
// src/messaging/idempotent-consumer.ts
import { Channel, ConsumeMessage } from 'amqplib';
import { Redis } from 'ioredis';
import { db } from '../database';

const redis = new Redis(process.env.REDIS_URL!);

export class IdempotentConsumer {
  private readonly idempotencyKeyTTL: number;
  
  constructor(
    private readonly channel: Channel,
    ttlSeconds: number = 86400 // 24 hours
  ) {
    this.idempotencyKeyTTL = ttlSeconds;
  }
  
  async processOnce<T>(
    msg: ConsumeMessage,
    handler: (payload: T) => Promise<unknown>,
    options: {
      idempotencyKey?: string; // Custom key, defaults to messageId
      lockTimeoutMs?: number;
    } = {}
  ): Promise<boolean> {
    const messageId = options.idempotencyKey ||
      msg.properties.messageId ||
      this.hashMessage(msg.content);
    
    const idempotencyKey = `idempotent:${messageId}`;
    const processingKey = `processing:${messageId}`;
    const lockTimeout = options.lockTimeoutMs || 60000;
    
    // Try to acquire processing lock
    const locked = await redis.set(
      processingKey,
      process.env.POD_NAME || 'processor',
      'PX', lockTimeout,
      'NX' // Only set if not exists
    );
    
    if (!locked) {
      // Another instance is processing this message
      log.warn('Message already being processed, skipping', { messageId });
      this.channel.ack(msg); // Ack to remove from queue
      return false;
    }
    
    try {
      // Check if already processed successfully
      const alreadyProcessed = await redis.exists(idempotencyKey);
      if (alreadyProcessed) {
        log.info('Message already processed (idempotent skip)', { messageId });
        this.channel.ack(msg);
        return false;
      }
      
      // Process the message
      const payload = JSON.parse(msg.content.toString()) as T;
      await handler(payload);
      
      // Mark as processed
      await redis.setex(idempotencyKey, this.idempotencyKeyTTL, JSON.stringify({
        processedAt: new Date().toISOString(),
        processedBy: process.env.POD_NAME,
      }));
      
      this.channel.ack(msg);
      return true;
    } catch (error) {
      // Don't mark as processed on error - allow retry
      this.channel.nack(msg, false, true);
      throw error;
    } finally {
      await redis.del(processingKey);
    }
  }
  
  private hashMessage(content: Buffer): string {
    return require('crypto')
      .createHash('sha256')
      .update(content)
      .digest('hex');
  }
}

// Database-backed idempotency (stronger guarantee)
export class DatabaseIdempotentConsumer {
  async processOnce<T>(
    messageId: string,
    handler: (payload: T) => Promise<unknown>,
    payload: T
  ): Promise<boolean> {
    return db.transaction(async (trx) => {
      // Try to insert idempotency record
      try {
        await trx('processed_messages').insert({
          message_id: messageId,
          processed_at: new Date(),
          status: 'processing',
          processor: process.env.POD_NAME,
        });
      } catch (error) {
        // Unique constraint violation = already processed or in progress
        const existing = await trx('processed_messages')
          .where({ message_id: messageId })
          .first();
        
        if (existing?.status === 'completed') {
          log.info('Message already processed', { messageId });
          return false;
        }
        
        if (existing?.status === 'processing') {
          // Another instance is handling it
          throw new Error('Message is being processed by another instance');
        }
        
        throw error;
      }
      
      try {
        await handler(payload);
        
        await trx('processed_messages')
          .where({ message_id: messageId })
          .update({ status: 'completed', completed_at: new Date() });
        
        return true;
      } catch (error) {
        await trx('processed_messages')
          .where({ message_id: messageId })
          .update({ status: 'failed', failed_at: new Date(), error_message: (error as Error).message });
        
        throw error;
      }
    });
  }
}
```

---

## 5. Outbox Pattern กับ PostgreSQL Triggers

```typescript
// src/messaging/outbox/outbox-pattern.ts
import { Knex } from 'knex';
import { Channel } from 'amqplib';

// Database schema
/*
CREATE TABLE outbox_events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  aggregate_type VARCHAR(100) NOT NULL,
  aggregate_id VARCHAR(100) NOT NULL,
  event_type VARCHAR(200) NOT NULL,
  payload JSONB NOT NULL,
  exchange VARCHAR(200),
  routing_key VARCHAR(200),
  headers JSONB DEFAULT '{}',
  status VARCHAR(20) NOT NULL DEFAULT 'pending',
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  processed_at TIMESTAMPTZ,
  retry_count INTEGER NOT NULL DEFAULT 0,
  last_error TEXT,
  scheduled_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_outbox_pending ON outbox_events (status, scheduled_at)
  WHERE status IN ('pending', 'failed');
*/

// PostgreSQL trigger to capture domain events atomically
const OUTBOX_TRIGGER_SQL = `
CREATE OR REPLACE FUNCTION capture_order_events()
RETURNS TRIGGER AS $$
BEGIN
  IF TG_OP = 'INSERT' THEN
    INSERT INTO outbox_events (
      aggregate_type, aggregate_id, event_type, payload, exchange, routing_key
    ) VALUES (
      'Order', NEW.id::text,
      'OrderCreated',
      jsonb_build_object(
        'orderId', NEW.id,
        'userId', NEW.user_id,
        'status', NEW.status,
        'totalAmount', NEW.total_amount,
        'currency', NEW.currency,
        'createdAt', NEW.created_at
      ),
      'orders.events',
      'order.created'
    );
  ELSIF TG_OP = 'UPDATE' THEN
    IF OLD.status != NEW.status THEN
      INSERT INTO outbox_events (
        aggregate_type, aggregate_id, event_type, payload, exchange, routing_key
      ) VALUES (
        'Order', NEW.id::text,
        'OrderStatusChanged',
        jsonb_build_object(
          'orderId', NEW.id,
          'previousStatus', OLD.status,
          'newStatus', NEW.status,
          'updatedAt', NOW()
        ),
        'orders.events',
        'order.status_changed'
      );
    END IF;
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER order_events_trigger
  AFTER INSERT OR UPDATE ON orders
  FOR EACH ROW EXECUTE FUNCTION capture_order_events();
`;

// Outbox processor: polls and publishes
export class OutboxProcessor {
  private isRunning = false;
  private intervalId: NodeJS.Timeout | null = null;
  
  constructor(
    private readonly db: Knex,
    private readonly channel: Channel,
    private readonly config: {
      pollIntervalMs?: number;
      batchSize?: number;
      maxRetries?: number;
      retryDelayMs?: number;
    } = {}
  ) {
    this.config = {
      pollIntervalMs: 1000,
      batchSize: 100,
      maxRetries: 5,
      retryDelayMs: 5000,
      ...config,
    };
  }
  
  start(): void {
    if (this.isRunning) return;
    this.isRunning = true;
    
    // Initial run immediately
    this.processOutbox().catch(err => log.error('Outbox processing error', err));
    
    this.intervalId = setInterval(() => {
      this.processOutbox().catch(err => log.error('Outbox processing error', err));
    }, this.config.pollIntervalMs);
    
    log.info('Outbox processor started');
  }
  
  stop(): void {
    this.isRunning = false;
    if (this.intervalId) {
      clearInterval(this.intervalId);
      this.intervalId = null;
    }
    log.info('Outbox processor stopped');
  }
  
  private async processOutbox(): Promise<void> {
    const events = await this.db('outbox_events')
      .where('status', 'pending')
      .orWhere((builder) => {
        builder
          .where('status', 'failed')
          .where('retry_count', '<', this.config.maxRetries!)
          .where('scheduled_at', '<=', new Date());
      })
      .orderBy('created_at', 'asc')
      .limit(this.config.batchSize!)
      .forUpdate()
      .skipLocked(); // Skip rows locked by other processes (PostgreSQL)
    
    if (events.length === 0) return;
    
    log.info(`Processing ${events.length} outbox events`);
    
    for (const event of events) {
      await this.publishEvent(event);
    }
  }
  
  private async publishEvent(event: any): Promise<void> {
    try {
      // Mark as processing
      await this.db('outbox_events')
        .where({ id: event.id })
        .update({ status: 'processing' });
      
      const payload = Buffer.from(JSON.stringify({
        ...event.payload,
        _outbox: {
          eventId: event.id,
          eventType: event.event_type,
          aggregateType: event.aggregate_type,
          aggregateId: event.aggregate_id,
          occurredAt: event.created_at,
        },
      }));
      
      const headers: Record<string, string> = {
        ...(event.headers || {}),
        'x-event-type': event.event_type,
        'x-aggregate-type': event.aggregate_type,
        'x-aggregate-id': event.aggregate_id,
        'x-outbox-id': event.id,
      };
      
      // Publish with confirm
      await new Promise<void>((resolve, reject) => {
        const published = this.channel.publish(
          event.exchange || '',
          event.routing_key,
          payload,
          {
            persistent: true,
            contentType: 'application/json',
            messageId: event.id,
            type: event.event_type,
            headers,
          }
        );
        
        if (!published) {
          reject(new Error('Channel write buffer full'));
        } else {
          resolve();
        }
      });
      
      // Wait for broker confirmation
      await this.channel.waitForConfirms();
      
      // Mark as published
      await this.db('outbox_events')
        .where({ id: event.id })
        .update({
          status: 'published',
          processed_at: new Date(),
        });
      
      log.info('Outbox event published', {
        eventId: event.id,
        eventType: event.event_type,
        aggregateId: event.aggregate_id,
      });
    } catch (error) {
      const retryCount = (event.retry_count || 0) + 1;
      const status = retryCount >= this.config.maxRetries! ? 'dead' : 'failed';
      const nextScheduledAt = new Date(
        Date.now() + this.config.retryDelayMs! * Math.pow(2, retryCount - 1)
      );
      
      await this.db('outbox_events')
        .where({ id: event.id })
        .update({
          status,
          retry_count: retryCount,
          last_error: (error as Error).message,
          scheduled_at: nextScheduledAt,
        });
      
      log.error('Failed to publish outbox event', error as Error, {
        eventId: event.id,
        retryCount,
        status,
      });
    }
  }
}

// Helper to publish events as part of a database transaction
export class TransactionalOutbox {
  constructor(private readonly trx: Knex.Transaction) {}
  
  async publish(
    aggregateType: string,
    aggregateId: string,
    eventType: string,
    payload: Record<string, unknown>,
    routing: { exchange?: string; routingKey: string }
  ): Promise<void> {
    await this.trx('outbox_events').insert({
      aggregate_type: aggregateType,
      aggregate_id: aggregateId,
      event_type: eventType,
      payload: JSON.stringify(payload),
      exchange: routing.exchange || '',
      routing_key: routing.routingKey,
      status: 'pending',
      created_at: new Date(),
    });
  }
}

// Usage in service
async function createOrderWithOutbox(orderData: CreateOrderDto): Promise<Order> {
  return db.transaction(async (trx) => {
    // 1. Create order in database
    const [order] = await trx('orders').insert({
      user_id: orderData.userId,
      status: 'pending',
      total_amount: orderData.totalAmount,
      currency: orderData.currency,
    }).returning('*');
    
    // 2. Record outbox event in same transaction
    const outbox = new TransactionalOutbox(trx);
    await outbox.publish(
      'Order',
      order.id,
      'OrderCreated',
      {
        orderId: order.id,
        userId: order.user_id,
        totalAmount: order.total_amount,
        currency: order.currency,
      },
      { routingKey: 'order.created' }
    );
    
    // Both operations are atomic - either both succeed or both fail
    return order;
  });
}
```

---

## 6. Inbox Pattern สำหรับ Deduplication

```typescript
// src/messaging/inbox/inbox-pattern.ts
import { Knex } from 'knex';
import { Channel, ConsumeMessage } from 'amqplib';

/*
CREATE TABLE inbox_messages (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  message_id VARCHAR(255) UNIQUE NOT NULL,  -- RabbitMQ messageId
  correlation_id VARCHAR(255),
  event_type VARCHAR(200) NOT NULL,
  payload JSONB NOT NULL,
  received_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  processed_at TIMESTAMPTZ,
  status VARCHAR(20) NOT NULL DEFAULT 'received',
  error_message TEXT,
  processing_attempts INTEGER NOT NULL DEFAULT 0
);

CREATE UNIQUE INDEX idx_inbox_message_id ON inbox_messages (message_id);
CREATE INDEX idx_inbox_unprocessed ON inbox_messages (status, received_at)
  WHERE status IN ('received', 'failed');
*/

export class InboxConsumer {
  constructor(
    private readonly db: Knex,
    private readonly channel: Channel
  ) {}
  
  async consume(
    queueName: string,
    handlers: Map<string, (payload: unknown) => Promise<void>>
  ): Promise<void> {
    await this.channel.prefetch(10);
    
    this.channel.consume(queueName, async (msg) => {
      if (!msg) return;
      
      const messageId = msg.properties.messageId;
      const eventType = msg.properties.type ||
        msg.properties.headers?.['x-event-type'];
      
      if (!messageId) {
        log.warn('Message missing messageId, cannot use inbox deduplication');
        this.channel.ack(msg);
        return;
      }
      
      try {
        // Attempt to insert into inbox (unique constraint prevents duplicates)
        const inserted = await this.tryInsertInbox(msg, messageId, eventType);
        
        if (!inserted) {
          // Already in inbox (duplicate)
          this.channel.ack(msg);
          return;
        }
        
        // Process the message
        const handler = handlers.get(eventType);
        if (!handler) {
          log.warn('No handler for event type', { eventType, messageId });
          await this.markInboxStatus(messageId, 'skipped');
          this.channel.ack(msg);
          return;
        }
        
        const payload = JSON.parse(msg.content.toString());
        await handler(payload);
        
        await this.markInboxStatus(messageId, 'processed');
        this.channel.ack(msg);
      } catch (error) {
        await this.markInboxError(messageId, (error as Error).message);
        this.channel.nack(msg, false, false); // Send to DLQ
      }
    });
  }
  
  private async tryInsertInbox(
    msg: ConsumeMessage,
    messageId: string,
    eventType: string
  ): Promise<boolean> {
    try {
      await this.db('inbox_messages').insert({
        message_id: messageId,
        correlation_id: msg.properties.correlationId,
        event_type: eventType || 'unknown',
        payload: msg.content.toString(),
        received_at: new Date(),
        status: 'received',
      });
      return true;
    } catch (error: any) {
      if (error.code === '23505') { // PostgreSQL unique violation
        log.info('Duplicate message detected', { messageId });
        return false;
      }
      throw error;
    }
  }
  
  private async markInboxStatus(messageId: string, status: string): Promise<void> {
    await this.db('inbox_messages')
      .where({ message_id: messageId })
      .update({
        status,
        processed_at: new Date(),
        processing_attempts: this.db.raw('processing_attempts + 1'),
      });
  }
  
  private async markInboxError(messageId: string, errorMessage: string): Promise<void> {
    await this.db('inbox_messages')
      .where({ message_id: messageId })
      .update({
        status: 'failed',
        error_message: errorMessage,
        processing_attempts: this.db.raw('processing_attempts + 1'),
      });
  }
}
```

---

## 7. Message Schema Versioning

```typescript
// src/messaging/schema-versioning.ts

// Version 1 schema
interface OrderCreatedV1 {
  _schema_version: 1;
  orderId: string;
  userId: string;
  total: number; // Before: single total field
  items: Array<{
    productId: string;
    quantity: number;
    price: number;
  }>;
  createdAt: string;
}

// Version 2 schema (breaking change: total split into subtotal + tax)
interface OrderCreatedV2 {
  _schema_version: 2;
  orderId: string;
  userId: string;
  subtotal: number;  // Changed from total
  tax: number;       // New field
  total: number;     // Kept for backward compat
  currency: string;  // New required field
  items: Array<{
    productId: string;
    productName: string; // New field
    quantity: number;
    unitPrice: number;   // Renamed from price
    totalPrice: number;  // New field
  }>;
  createdAt: string;
  updatedAt: string; // New field
}

type OrderCreatedEvent = OrderCreatedV1 | OrderCreatedV2;

// Schema migrator
export class MessageMigrator {
  private readonly migrations = new Map<
    `${number}->${number}`,
    (msg: any) => any
  >();
  
  registerMigration(
    fromVersion: number,
    toVersion: number,
    migrator: (msg: any) => any
  ): void {
    this.migrations.set(`${fromVersion}->${toVersion}`, migrator);
  }
  
  migrateToLatest<T>(message: { _schema_version?: number } & Record<string, unknown>): T {
    const currentVersion = message._schema_version || 1;
    const LATEST_VERSION = 2;
    
    let current: any = message;
    
    for (let v = currentVersion; v < LATEST_VERSION; v++) {
      const migration = this.migrations.get(`${v}->${v + 1}`);
      if (!migration) {
        throw new Error(`No migration from v${v} to v${v + 1}`);
      }
      current = migration(current);
    }
    
    return current as T;
  }
}

// Register migrations
export const orderEventMigrator = new MessageMigrator();

// Migration: V1 → V2
orderEventMigrator.registerMigration(1, 2, (v1: OrderCreatedV1): OrderCreatedV2 => ({
  _schema_version: 2,
  orderId: v1.orderId,
  userId: v1.userId,
  subtotal: v1.total,       // Estimate: assume no tax in V1
  tax: 0,                   // Default: no tax
  total: v1.total,
  currency: 'USD',          // Default: assume USD for old events
  items: v1.items.map(item => ({
    productId: item.productId,
    productName: item.productId, // Use productId as placeholder
    quantity: item.quantity,
    unitPrice: item.price,        // Renamed
    totalPrice: item.price * item.quantity,
  })),
  createdAt: v1.createdAt,
  updatedAt: v1.createdAt,        // Default to createdAt
}));

// Schema registry for validation
import Ajv from 'ajv';

const ajv = new Ajv({ allErrors: true });

const schemas: Record<number, object> = {
  1: {
    type: 'object',
    required: ['_schema_version', 'orderId', 'userId', 'total'],
    properties: {
      _schema_version: { type: 'integer', enum: [1] },
      orderId: { type: 'string', format: 'uuid' },
      userId: { type: 'string', format: 'uuid' },
      total: { type: 'number', minimum: 0 },
    },
  },
  2: {
    type: 'object',
    required: ['_schema_version', 'orderId', 'userId', 'subtotal', 'total', 'currency'],
    properties: {
      _schema_version: { type: 'integer', enum: [2] },
      orderId: { type: 'string', format: 'uuid' },
      userId: { type: 'string', format: 'uuid' },
      subtotal: { type: 'number', minimum: 0 },
      tax: { type: 'number', minimum: 0 },
      total: { type: 'number', minimum: 0 },
      currency: { type: 'string', pattern: '^[A-Z]{3}$' },
    },
  },
};

export class SchemaRegistry {
  private readonly validators = new Map<number, ReturnType<typeof ajv.compile>>();
  
  constructor() {
    for (const [version, schema] of Object.entries(schemas)) {
      this.validators.set(parseInt(version), ajv.compile(schema));
    }
  }
  
  validate(message: unknown, version?: number): void {
    const v = version || (message as any)?._schema_version || 1;
    const validator = this.validators.get(v);
    
    if (!validator) {
      throw new Error(`Unknown schema version: ${v}`);
    }
    
    if (!validator(message)) {
      throw new Error(
        `Schema validation failed (v${v}): ${ajv.errorsText(validator.errors)}`
      );
    }
  }
}

// Consumer that handles multiple schema versions
export function createVersionedConsumer(channel: Channel, queueName: string) {
  const migrator = orderEventMigrator;
  const registry = new SchemaRegistry();
  
  channel.consume(queueName, async (msg) => {
    if (!msg) return;
    
    try {
      const rawPayload = JSON.parse(msg.content.toString());
      const version = rawPayload._schema_version || 1;
      
      // Validate against claimed schema version
      registry.validate(rawPayload, version);
      
      // Migrate to latest version if needed
      const payload = migrator.migrateToLatest<OrderCreatedV2>(rawPayload);
      
      log.info('Processing order event', {
        orderId: payload.orderId,
        schemaVersion: version,
        migratedTo: payload._schema_version,
      });
      
      await processOrder(payload);
      channel.ack(msg);
    } catch (error) {
      log.error('Failed to process message', error as Error);
      channel.nack(msg, false, false);
    }
  });
}

async function processOrder(event: OrderCreatedV2): Promise<void> {
  // Process always works with V2 schema
}
```

---

## 8. At-least-once Delivery กับ Business Logic Idempotency

```typescript
// src/messaging/at-least-once-delivery.ts
import { Channel, ConsumeMessage } from 'amqplib';
import { db } from '../database';

// Business logic ต้องเป็น idempotent เสมอเมื่อใช้ at-least-once delivery
export class OrderPaymentProcessor {
  async processPaymentSuccess(
    channel: Channel,
    msg: ConsumeMessage
  ): Promise<void> {
    const event = JSON.parse(msg.content.toString());
    const { orderId, paymentId, amount } = event;
    
    // Idempotent: ถ้า order already paid, ไม่ทำซ้ำ
    const order = await db('orders')
      .where({ id: orderId })
      .first();
    
    if (!order) {
      log.warn('Order not found for payment event', { orderId, paymentId });
      channel.ack(msg); // Ack: ไม่ retry event ที่ไม่มี order
      return;
    }
    
    if (order.status === 'paid') {
      // Already processed - idempotent success
      log.info('Order already paid, skipping', { orderId, paymentId });
      channel.ack(msg);
      return;
    }
    
    if (order.status !== 'pending') {
      log.warn('Cannot process payment for order in status', {
        orderId,
        status: order.status,
      });
      channel.ack(msg); // Ack to prevent infinite retry
      return;
    }
    
    try {
      await db.transaction(async (trx) => {
        // Update order status
        await trx('orders')
          .where({ id: orderId, status: 'pending' }) // Optimistic lock via status check
          .update({
            status: 'paid',
            payment_id: paymentId,
            paid_at: new Date(),
            updated_at: new Date(),
          });
        
        // Verify update happened (another instance might have beaten us)
        const updated = await trx('orders')
          .where({ id: orderId, status: 'paid', payment_id: paymentId })
          .first();
        
        if (!updated) {
          throw new Error('Concurrent update detected');
        }
        
        // Record payment history
        await trx('payment_history').insert({
          order_id: orderId,
          payment_id: paymentId,
          amount,
          status: 'completed',
          processed_at: new Date(),
        });
      });
      
      channel.ack(msg);
      log.info('Payment processed successfully', { orderId, paymentId });
    } catch (error) {
      if ((error as Error).message === 'Concurrent update detected') {
        // Another instance processed it first, ack to avoid re-delivery
        channel.ack(msg);
      } else {
        channel.nack(msg, false, true); // Retry
        throw error;
      }
    }
  }
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้ Async Communication Patterns ขั้นสูงครอบคลุม:

1. **Request-Reply over RabbitMQ** — RpcClient/RpcServer ด้วย reply queues, correlation IDs, timeout handling, และ error propagation

2. **Correlation ID Tracking** — AsyncLocalStorage สำหรับ propagate context ข้าม async boundaries, automatic injection ใน outgoing messages

3. **Message Ordering** — OrderingBuffer สำหรับ reorder out-of-sequence messages, gap detection, partition-based ordering

4. **Idempotent Consumer** — Redis-based distributed locking, database unique constraint, business logic idempotency ด้วย status checks

5. **Outbox Pattern** — Database trigger สำหรับ automatic event capture, polling processor พร้อม exponential backoff retry, PostgreSQL `FOR UPDATE SKIP LOCKED`

6. **Inbox Pattern** — Database deduplication ด้วย unique constraint, message handler registry, status tracking

7. **Schema Versioning** — Version field, AJV validation per version, migration chain, consumer ที่ handle multiple versions

8. **At-least-once Delivery** — Business logic idempotency ด้วย optimistic locking, concurrent update detection

Key takeaways:
- Outbox + Inbox patterns ร่วมกันให้ exactly-once semantics ได้โดยไม่ต้องใช้ distributed transactions
- ทุก consumer ต้องเป็น idempotent เสมอเมื่อใช้ at-least-once delivery
- Schema versioning ทำให้ services deploy independently ได้โดยไม่ต้องประสานงาน
- Request-Reply ผ่าน message broker มีข้อดีเรื่อง decoupling และ backpressure แต่ latency สูงกว่า HTTP ตรง
- Correlation ID เป็น essential สำหรับ debugging distributed systems

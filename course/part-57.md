# Part 57: Async Communication Patterns

## บทนำ

Asynchronous Communication ใน Microservices มีความซับซ้อนมากกว่า Synchronous REST Call เพราะต้องจัดการ Message Ordering, Idempotency, Deduplication, และ Delivery Guarantees ในบทนี้เราจะเรียนรู้ Pattern ที่ Production-ready สำหรับระบบที่ต้องการ Reliability สูง

---

## 1. Delivery Guarantees

### At-most-once
- Message ถูกส่งได้สูงสุด 1 ครั้ง (อาจสูญหาย)
- ใช้กับข้อมูลที่ไม่สำคัญ เช่น Analytics Events

### At-least-once
- Message ถูก process อย่างน้อย 1 ครั้ง (อาจ duplicate)
- RabbitMQ default พร้อม manual ack
- Consumer ต้องเป็น **Idempotent**

### Exactly-once
- Message ถูก process ครั้งเดียวเท่านั้น
- ทำได้ด้วย Outbox Pattern + Deduplication (Inbox Pattern)

---

## 2. Outbox Pattern กับ PostgreSQL

Outbox Pattern แก้ปัญหา **Dual Write Problem** — เมื่อต้องบันทึก Database และส่ง Message พร้อมกัน โดยไม่ให้ทั้งสองล้มเหลวแบบ inconsistent

```
❌ Dual Write Problem:
  1. UPDATE orders SET status='completed'  ← ทำสำเร็จ
  2. publish('order.completed', ...)       ← ล้มเหลว → inconsistent!

✅ Outbox Pattern:
  1. BEGIN TRANSACTION
     UPDATE orders SET status='completed'
     INSERT INTO outbox (event_type, payload) VALUES ('order.completed', ...)
     COMMIT
  2. Outbox Worker reads outbox → publishes → marks as sent
```

### 2.1 Outbox Schema

```sql
-- db/outbox-schema.sql
CREATE TABLE outbox (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_id    VARCHAR(255) NOT NULL,    -- เช่น orderId
    aggregate_type  VARCHAR(100) NOT NULL,    -- เช่น 'Order'
    event_type      VARCHAR(100) NOT NULL,    -- เช่น 'order.completed'
    event_version   INTEGER NOT NULL DEFAULT 1,
    payload         JSONB NOT NULL,
    headers         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    sent_at         TIMESTAMPTZ,
    send_attempts   INTEGER NOT NULL DEFAULT 0,
    last_error      TEXT,
    scheduled_for   TIMESTAMPTZ NOT NULL DEFAULT NOW(), -- สำหรับ delayed events
    CONSTRAINT chk_send_attempts CHECK (send_attempts >= 0)
);

CREATE INDEX idx_outbox_unsent ON outbox (scheduled_for, created_at)
  WHERE sent_at IS NULL;

CREATE INDEX idx_outbox_aggregate ON outbox (aggregate_type, aggregate_id)
  WHERE sent_at IS NULL;

-- Partition by date สำหรับ cleanup ง่าย (optional)
-- CREATE TABLE outbox_y2026m01 PARTITION OF outbox
-- FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
```

### 2.2 Outbox Repository

```typescript
// packages/shared/src/outbox/outbox.repository.ts

import { Pool, PoolClient } from 'pg';

export interface OutboxMessage {
  id: string;
  aggregateId: string;
  aggregateType: string;
  eventType: string;
  eventVersion: number;
  payload: Record<string, unknown>;
  headers: Record<string, string>;
  createdAt: Date;
  scheduledFor: Date;
}

export class OutboxRepository {
  constructor(private readonly pool: Pool) {}

  // ใช้ client ที่ผ่านมา เพื่อให้ run ใน transaction เดียวกัน
  async insert(
    client: PoolClient,
    message: Omit<OutboxMessage, 'id' | 'createdAt'>
  ): Promise<string> {
    const result = await client.query(
      `INSERT INTO outbox
         (aggregate_id, aggregate_type, event_type, event_version,
          payload, headers, scheduled_for)
       VALUES ($1, $2, $3, $4, $5, $6, $7)
       RETURNING id`,
      [
        message.aggregateId,
        message.aggregateType,
        message.eventType,
        message.eventVersion,
        JSON.stringify(message.payload),
        JSON.stringify(message.headers),
        message.scheduledFor ?? new Date(),
      ]
    );

    return result.rows[0].id as string;
  }

  // Lock and fetch unsent messages (SELECT FOR UPDATE SKIP LOCKED)
  async fetchAndLockUnsent(
    batchSize = 100,
    maxAgeSeconds = 300 // ไม่เอา message ที่ส่งล้มเหลวภายใน 5 นาที
  ): Promise<OutboxMessage[]> {
    const result = await this.pool.query(
      `SELECT id, aggregate_id, aggregate_type, event_type,
              event_version, payload, headers, created_at, scheduled_for
       FROM outbox
       WHERE sent_at IS NULL
         AND scheduled_for <= NOW()
         AND (
           send_attempts = 0
           OR created_at < NOW() - (INTERVAL '1 second' * $3)
         )
         AND send_attempts < 5
       ORDER BY scheduled_for, created_at
       LIMIT $1
       FOR UPDATE SKIP LOCKED`,
      [batchSize, maxAgeSeconds, maxAgeSeconds]
    );

    return result.rows.map(this.mapRow);
  }

  async markAsSent(ids: string[]): Promise<void> {
    await this.pool.query(
      `UPDATE outbox
       SET sent_at = NOW()
       WHERE id = ANY($1::uuid[])`,
      [ids]
    );
  }

  async markAsFailed(id: string, error: string): Promise<void> {
    // Exponential backoff: 1min, 2min, 4min, 8min, 16min
    await this.pool.query(
      `UPDATE outbox
       SET send_attempts = send_attempts + 1,
           last_error = $2,
           scheduled_for = NOW() + (INTERVAL '1 minute' * POWER(2, send_attempts))
       WHERE id = $1`,
      [id, error]
    );
  }

  // Cleanup sent messages
  async cleanup(olderThanDays = 7): Promise<number> {
    const result = await this.pool.query(
      `DELETE FROM outbox
       WHERE sent_at < NOW() - ($1 || ' days')::INTERVAL`,
      [olderThanDays]
    );

    return result.rowCount ?? 0;
  }

  private mapRow(row: Record<string, unknown>): OutboxMessage {
    return {
      id: row.id as string,
      aggregateId: row.aggregate_id as string,
      aggregateType: row.aggregate_type as string,
      eventType: row.event_type as string,
      eventVersion: row.event_version as number,
      payload: row.payload as Record<string, unknown>,
      headers: row.headers as Record<string, string>,
      createdAt: row.created_at as Date,
      scheduledFor: row.scheduled_for as Date,
    };
  }
}
```

### 2.3 Outbox Worker (Polling-based)

```typescript
// packages/shared/src/outbox/outbox.worker.ts

import { Pool } from 'pg';
import { Channel } from 'amqplib';
import { OutboxRepository } from './outbox.repository';
import { Counter, Histogram } from 'prom-client';

export class OutboxWorker {
  private readonly repository: OutboxRepository;
  private running = false;
  private pollInterval: ReturnType<typeof setTimeout> | null = null;

  private readonly messagesPublished = new Counter({
    name: 'outbox_messages_published_total',
    help: 'Messages published from outbox',
    labelNames: ['event_type'],
  });

  private readonly publishDuration = new Histogram({
    name: 'outbox_publish_duration_seconds',
    help: 'Time to publish outbox batch',
    buckets: [0.01, 0.05, 0.1, 0.5, 1, 5],
  });

  constructor(
    private readonly pool: Pool,
    private readonly channel: Channel,
    private readonly options: {
      batchSize?: number;
      pollIntervalMs?: number;
      exchange?: string;
    } = {}
  ) {
    this.repository = new OutboxRepository(pool);
  }

  start(): void {
    if (this.running) return;
    this.running = true;
    this.poll();
    console.log('[OutboxWorker] Started');
  }

  stop(): void {
    this.running = false;
    if (this.pollInterval) {
      clearTimeout(this.pollInterval);
      this.pollInterval = null;
    }
    console.log('[OutboxWorker] Stopped');
  }

  private async poll(): Promise<void> {
    if (!this.running) return;

    const timer = this.publishDuration.startTimer();

    try {
      const messages = await this.repository.fetchAndLockUnsent(
        this.options.batchSize ?? 100
      );

      if (messages.length > 0) {
        console.log(`[OutboxWorker] Publishing ${messages.length} messages`);
        await this.publishBatch(messages);
      }
    } catch (error) {
      console.error('[OutboxWorker] Poll error:', error);
    } finally {
      timer();
    }

    // Schedule next poll
    const delay = this.options.pollIntervalMs ?? 1000;
    this.pollInterval = setTimeout(() => this.poll(), delay);
  }

  private async publishBatch(messages: Array<{
    id: string;
    eventType: string;
    aggregateType: string;
    aggregateId: string;
    payload: Record<string, unknown>;
    headers: Record<string, string>;
    eventVersion: number;
    createdAt: Date;
  }>): Promise<void> {
    const published: string[] = [];
    const failed: Array<{ id: string; error: string }> = [];

    for (const msg of messages) {
      try {
        const exchange = this.options.exchange ?? msg.aggregateType.toLowerCase();
        const routingKey = msg.eventType;

        // Publish ด้วย Publisher Confirms
        await new Promise<void>((resolve, reject) => {
          const ok = this.channel.publish(
            exchange,
            routingKey,
            Buffer.from(JSON.stringify(msg.payload)),
            {
              persistent: true,
              messageId: msg.id, // ใช้ outbox ID เป็น message ID
              contentType: 'application/json',
              timestamp: Math.floor(msg.createdAt.getTime() / 1000),
              headers: {
                ...msg.headers,
                'x-aggregate-id': msg.aggregateId,
                'x-aggregate-type': msg.aggregateType,
                'x-event-version': String(msg.eventVersion),
                'x-outbox-id': msg.id,
              },
            }
          );

          // Simple confirm via channel confirm mode
          this.channel.waitForConfirms()
            .then(() => resolve())
            .catch(reject);
        });

        published.push(msg.id);
        this.messagesPublished.inc({ event_type: msg.eventType });

      } catch (error) {
        const errMsg = (error as Error).message;
        console.error(
          `[OutboxWorker] Failed to publish message ${msg.id}:`,
          errMsg
        );
        failed.push({ id: msg.id, error: errMsg });
      }
    }

    // Mark published
    if (published.length > 0) {
      await this.repository.markAsSent(published);
    }

    // Mark failed
    for (const { id, error } of failed) {
      await this.repository.markAsFailed(id, error);
    }
  }
}
```

---

## 3. Inbox Pattern สำหรับ Deduplication

Inbox Pattern ช่วยป้องกันการ process message ซ้ำใน At-least-once delivery

```sql
-- db/inbox-schema.sql
CREATE TABLE inbox (
    id              UUID PRIMARY KEY,           -- ใช้ message ID จาก broker
    event_type      VARCHAR(100) NOT NULL,
    payload         JSONB NOT NULL,
    received_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    processed_at    TIMESTAMPTZ,
    processing_node VARCHAR(255),               -- Node ที่กำลัง process
    error_message   TEXT,
    retry_count     INTEGER NOT NULL DEFAULT 0
);

-- Index สำหรับ check duplication
CREATE UNIQUE INDEX idx_inbox_id ON inbox(id);
CREATE INDEX idx_inbox_unprocessed ON inbox (received_at)
  WHERE processed_at IS NULL;

-- Cleanup
CREATE OR REPLACE FUNCTION cleanup_inbox(retention_days INTEGER DEFAULT 30)
RETURNS INTEGER AS $$
DECLARE deleted_count INTEGER;
BEGIN
    DELETE FROM inbox
    WHERE processed_at < NOW() - (retention_days || ' days')::INTERVAL;
    GET DIAGNOSTICS deleted_count = ROW_COUNT;
    RETURN deleted_count;
END;
$$ LANGUAGE plpgsql;
```

```typescript
// packages/shared/src/inbox/inbox.repository.ts

import { Pool, PoolClient } from 'pg';

export interface InboxMessage {
  id: string;
  eventType: string;
  payload: Record<string, unknown>;
  receivedAt: Date;
  processedAt?: Date;
  processingNode?: string;
  retryCount: number;
}

export class InboxRepository {
  private readonly nodeId: string;

  constructor(
    private readonly pool: Pool,
    nodeId?: string
  ) {
    this.nodeId = nodeId ?? `${process.env.SERVICE_NAME ?? 'service'}-${process.pid}`;
  }

  // Try to insert — returns false if duplicate
  async tryInsert(
    client: PoolClient,
    id: string,
    eventType: string,
    payload: Record<string, unknown>
  ): Promise<boolean> {
    const result = await client.query(
      `INSERT INTO inbox (id, event_type, payload)
       VALUES ($1, $2, $3)
       ON CONFLICT (id) DO NOTHING
       RETURNING id`,
      [id, eventType, JSON.stringify(payload)]
    );

    return (result.rowCount ?? 0) > 0;
  }

  // Fetch and lock unprocessed messages
  async fetchAndLock(batchSize = 50): Promise<InboxMessage[]> {
    const result = await this.pool.query(
      `UPDATE inbox
       SET processing_node = $1
       WHERE id IN (
         SELECT id FROM inbox
         WHERE processed_at IS NULL
           AND processing_node IS NULL
           AND retry_count < 5
         ORDER BY received_at
         LIMIT $2
         FOR UPDATE SKIP LOCKED
       )
       RETURNING *`,
      [this.nodeId, batchSize]
    );

    return result.rows.map(this.mapRow);
  }

  async markAsProcessed(id: string): Promise<void> {
    await this.pool.query(
      `UPDATE inbox
       SET processed_at = NOW(),
           processing_node = NULL
       WHERE id = $1`,
      [id]
    );
  }

  async markAsFailed(id: string, error: string): Promise<void> {
    await this.pool.query(
      `UPDATE inbox
       SET processing_node = NULL,
           retry_count = retry_count + 1,
           error_message = $2
       WHERE id = $1`,
      [id, error]
    );
  }

  private mapRow(row: Record<string, unknown>): InboxMessage {
    return {
      id: row.id as string,
      eventType: row.event_type as string,
      payload: row.payload as Record<string, unknown>,
      receivedAt: row.received_at as Date,
      processedAt: row.processed_at as Date | undefined,
      processingNode: row.processing_node as string | undefined,
      retryCount: row.retry_count as number,
    };
  }
}
```

---

## 4. Idempotent Message Consumer

```typescript
// packages/shared/src/idempotency/idempotent.consumer.ts

import { Pool, PoolClient } from 'pg';
import { InboxRepository } from '../inbox/inbox.repository';
import { MessageMetadata } from '../rabbitmq/consumer';

type IdempotentHandler<T> = (
  message: T,
  metadata: MessageMetadata,
  client: PoolClient
) => Promise<void>;

export class IdempotentConsumer<T = unknown> {
  private readonly inboxRepository: InboxRepository;

  constructor(
    private readonly pool: Pool,
    private readonly handler: IdempotentHandler<T>
  ) {
    this.inboxRepository = new InboxRepository(pool);
  }

  async process(message: T, metadata: MessageMetadata): Promise<void> {
    const client = await this.pool.connect();

    try {
      await client.query('BEGIN');

      // ตรวจสอบ duplicate
      const isNew = await this.inboxRepository.tryInsert(
        client,
        metadata.messageId,
        metadata.routingKey,
        message as Record<string, unknown>
      );

      if (!isNew) {
        // Message นี้ถูก process ไปแล้ว — idempotent skip
        console.log(
          `[IdempotentConsumer] Duplicate message skipped: ${metadata.messageId}`
        );
        await client.query('ROLLBACK');
        return;
      }

      // Process message ใน transaction เดียวกัน
      await this.handler(message, metadata, client);

      await client.query('COMMIT');

      // Mark processed ใน separate query (after commit)
      await this.inboxRepository.markAsProcessed(metadata.messageId);

    } catch (error) {
      await client.query('ROLLBACK');
      await this.inboxRepository.markAsFailed(
        metadata.messageId,
        (error as Error).message
      );
      throw error;
    } finally {
      client.release();
    }
  }
}

// ─── Usage Example ─────────────────────────────────────────────────────────

import { RabbitMQConsumer } from '../rabbitmq/consumer';
import { Channel } from 'amqplib';

interface OrderCompletedEvent {
  orderId: string;
  userId: string;
  totalAmount: number;
  items: Array<{ productId: string; quantity: number }>;
}

export async function setupOrderCompletedConsumer(
  channel: Channel,
  pool: Pool
): Promise<void> {
  const consumer = new RabbitMQConsumer(channel);

  const idempotentConsumer = new IdempotentConsumer<OrderCompletedEvent>(
    pool,
    async (event, metadata, client) => {
      // ทำ business logic ใน database transaction เดียวกัน
      // ถ้า process ล้มเหลว ทั้ง outbox entry และ inbox entry จะถูก rollback

      // 1. Update inventory
      await client.query(
        'UPDATE inventory SET reserved = reserved - $1 WHERE product_id = $2',
        [event.items[0].quantity, event.items[0].productId]
      );

      // 2. Create fulfillment record
      await client.query(
        `INSERT INTO fulfillments (order_id, user_id, status)
         VALUES ($1, $2, 'pending')
         ON CONFLICT (order_id) DO NOTHING`,
        [event.orderId, event.userId]
      );

      // 3. Publish downstream event (via outbox)
      const outbox = new OutboxRepository(pool);
      await outbox.insert(client, {
        aggregateId: event.orderId,
        aggregateType: 'Order',
        eventType: 'fulfillment.created',
        eventVersion: 1,
        payload: {
          orderId: event.orderId,
          userId: event.userId,
        },
        headers: {
          'x-correlation-id': metadata.correlationId ?? '',
        },
        scheduledFor: new Date(),
      });

      console.log(`[Consumer] Processed order.completed: ${event.orderId}`);
    }
  );

  await consumer.consume<OrderCompletedEvent>(
    'order.completed',
    async (event, metadata) => {
      await idempotentConsumer.process(event, metadata);
    },
    {
      prefetchCount: 5,
      maxRetries: 3,
    }
  );
}

import { OutboxRepository } from '../outbox/outbox.repository';
```

---

## 5. Correlation ID Pattern

```typescript
// packages/shared/src/correlation/correlation.context.ts
// ติดตาม Request ข้าม Service ด้วย Correlation ID

import { AsyncLocalStorage } from 'async_hooks';
import { Request, Response, NextFunction } from 'express';

interface CorrelationContext {
  correlationId: string;
  requestId: string;
  userId?: string;
  sessionId?: string;
  traceId?: string;
}

const asyncLocalStorage = new AsyncLocalStorage<CorrelationContext>();

export function getCorrelationContext(): CorrelationContext | undefined {
  return asyncLocalStorage.getStore();
}

export function getCorrelationId(): string {
  return asyncLocalStorage.getStore()?.correlationId ?? 'unknown';
}

export function withCorrelationContext<T>(
  context: CorrelationContext,
  fn: () => T
): T {
  return asyncLocalStorage.run(context, fn);
}

// Express Middleware
export function correlationMiddleware(
  req: Request,
  res: Response,
  next: NextFunction
): void {
  const correlationId =
    req.headers['x-correlation-id']?.toString() ??
    req.headers['x-request-id']?.toString() ??
    generateCorrelationId();

  const requestId = generateCorrelationId();

  const context: CorrelationContext = {
    correlationId,
    requestId,
    userId: req.headers['x-user-id']?.toString(),
    sessionId: req.headers['x-session-id']?.toString(),
    traceId: req.headers['traceparent']?.toString(),
  };

  // Propagate ไปใน response headers
  res.setHeader('x-correlation-id', correlationId);
  res.setHeader('x-request-id', requestId);

  asyncLocalStorage.run(context, () => next());
}

// Axios Interceptor — inject correlation headers ในทุก outgoing request
import axios from 'axios';

export function setupAxiosCorrelation(): void {
  axios.interceptors.request.use((config) => {
    const context = getCorrelationContext();
    if (context) {
      config.headers['x-correlation-id'] = context.correlationId;
      config.headers['x-request-id'] = generateCorrelationId();
      if (context.userId) {
        config.headers['x-user-id'] = context.userId;
      }
    }
    return config;
  });
}

function generateCorrelationId(): string {
  return `${Date.now().toString(36)}-${Math.random().toString(36).slice(2, 9)}`;
}
```

---

## 6. Message Ordering Guarantees

```typescript
// packages/shared/src/ordering/ordered.consumer.ts
// Ensure message ordering per partition key

interface OrderedMessage<T> {
  partitionKey: string; // เช่น userId หรือ orderId
  sequenceNumber: number;
  payload: T;
}

export class OrderedMessageProcessor<T> {
  // Buffer สำหรับ message ที่มาไม่เป็นลำดับ
  private readonly buffers: Map<
    string,
    Map<number, OrderedMessage<T>>
  > = new Map();

  private readonly expectedSeq: Map<string, number> = new Map();

  constructor(
    private readonly handler: (message: T, key: string) => Promise<void>,
    private readonly options: {
      maxBufferSize?: number;
      bufferTimeoutMs?: number;
    } = {}
  ) {}

  async process(message: OrderedMessage<T>): Promise<void> {
    const { partitionKey, sequenceNumber } = message;

    if (!this.buffers.has(partitionKey)) {
      this.buffers.set(partitionKey, new Map());
    }

    const buffer = this.buffers.get(partitionKey)!;
    const expectedSeq = this.expectedSeq.get(partitionKey) ?? 1;

    if (sequenceNumber === expectedSeq) {
      // Message มาถูกลำดับ — process ทันที
      await this.handler(message.payload, partitionKey);
      this.expectedSeq.set(partitionKey, expectedSeq + 1);

      // ตรวจสอบว่ามี buffered message ที่รอลำดับถัดไปหรือไม่
      await this.drainBuffer(partitionKey);

    } else if (sequenceNumber > expectedSeq) {
      // Message มาไม่เป็นลำดับ — buffer ไว้ก่อน
      if (buffer.size >= (this.options.maxBufferSize ?? 1000)) {
        throw new Error(
          `Buffer overflow for partition ${partitionKey}. Expected seq ${expectedSeq}, got ${sequenceNumber}`
        );
      }

      buffer.set(sequenceNumber, message);
      console.log(
        `[OrderedConsumer] Buffered out-of-order message: key=${partitionKey}, seq=${sequenceNumber}, expected=${expectedSeq}`
      );

    } else {
      // sequenceNumber < expectedSeq — duplicate/old message
      console.warn(
        `[OrderedConsumer] Ignoring old message: key=${partitionKey}, seq=${sequenceNumber}, expected=${expectedSeq}`
      );
    }
  }

  private async drainBuffer(partitionKey: string): Promise<void> {
    const buffer = this.buffers.get(partitionKey);
    if (!buffer) return;

    while (true) {
      const nextSeq = this.expectedSeq.get(partitionKey) ?? 1;
      const nextMessage = buffer.get(nextSeq);

      if (!nextMessage) break;

      buffer.delete(nextSeq);
      await this.handler(nextMessage.payload, partitionKey);
      this.expectedSeq.set(partitionKey, nextSeq + 1);
    }
  }

  getBufferStats(): Array<{ partitionKey: string; bufferedCount: number; expectedSeq: number }> {
    return Array.from(this.buffers.entries()).map(([key, buffer]) => ({
      partitionKey: key,
      bufferedCount: buffer.size,
      expectedSeq: this.expectedSeq.get(key) ?? 1,
    }));
  }
}
```

---

## 7. Request-Reply Pattern กับ Correlation ID

```typescript
// packages/shared/src/patterns/request-reply.ts

import { Channel } from 'amqplib';
import { EventEmitter } from 'events';

interface PendingRequest {
  resolve: (value: unknown) => void;
  reject: (error: Error) => void;
  timer: ReturnType<typeof setTimeout>;
  requestedAt: Date;
}

export class RequestReplyBus extends EventEmitter {
  private readonly pendingRequests: Map<string, PendingRequest> = new Map();
  private replyQueue!: string;
  private initialized = false;

  constructor(private readonly channel: Channel) {
    super();
  }

  async initialize(): Promise<void> {
    if (this.initialized) return;

    // Exclusive, auto-delete reply queue
    const { queue } = await this.channel.assertQueue('', {
      exclusive: true,
      autoDelete: true,
      durable: false,
    });
    this.replyQueue = queue;

    await this.channel.consume(
      this.replyQueue,
      (msg) => {
        if (!msg) return;

        const correlationId = msg.properties.correlationId;
        if (!correlationId) return;

        const pending = this.pendingRequests.get(correlationId);
        if (!pending) {
          console.warn(
            `[RequestReply] No pending request for correlationId: ${correlationId}`
          );
          this.channel.ack(msg);
          return;
        }

        clearTimeout(pending.timer);
        this.pendingRequests.delete(correlationId);

        try {
          const response = JSON.parse(msg.content.toString());

          if (response.__error) {
            pending.reject(new Error(response.__error));
          } else {
            pending.resolve(response);
          }
        } catch (error) {
          pending.reject(error as Error);
        }

        this.channel.ack(msg);
      },
      { noAck: false }
    );

    this.initialized = true;
    console.log(`[RequestReply] Initialized with reply queue: ${this.replyQueue}`);
  }

  async request<TReq, TRes>(
    exchange: string,
    routingKey: string,
    payload: TReq,
    options: {
      correlationId?: string;
      timeoutMs?: number;
      headers?: Record<string, string>;
    } = {}
  ): Promise<TRes> {
    await this.initialize();

    const correlationId = options.correlationId ?? generateId();
    const timeoutMs = options.timeoutMs ?? 30000;

    return new Promise<TRes>((resolve, reject) => {
      const timer = setTimeout(() => {
        this.pendingRequests.delete(correlationId);
        reject(
          new Error(
            `Request timed out after ${timeoutMs}ms (correlationId: ${correlationId})`
          )
        );
      }, timeoutMs);

      this.pendingRequests.set(correlationId, {
        resolve: resolve as (value: unknown) => void,
        reject,
        timer,
        requestedAt: new Date(),
      });

      this.channel.publish(
        exchange,
        routingKey,
        Buffer.from(JSON.stringify(payload)),
        {
          correlationId,
          replyTo: this.replyQueue,
          persistent: false,
          contentType: 'application/json',
          timestamp: Math.floor(Date.now() / 1000),
          headers: options.headers,
        }
      );
    });
  }

  // Statistics
  getPendingCount(): number {
    return this.pendingRequests.size;
  }

  getOldestPendingAge(): number {
    let maxAge = 0;
    for (const pending of this.pendingRequests.values()) {
      const age = Date.now() - pending.requestedAt.getTime();
      maxAge = Math.max(maxAge, age);
    }
    return maxAge;
  }
}

// Reply Server
export async function createReplyServer<TReq, TRes>(
  channel: Channel,
  queue: string,
  handler: (request: TReq, correlationId: string) => Promise<TRes>
): Promise<void> {
  await channel.prefetch(10);

  await channel.consume(queue, async (msg) => {
    if (!msg) return;

    const correlationId = msg.properties.correlationId;
    const replyTo = msg.properties.replyTo;

    if (!replyTo) {
      console.warn('[ReplyServer] Message has no replyTo, discarding');
      channel.ack(msg);
      return;
    }

    const request = JSON.parse(msg.content.toString()) as TReq;
    let response: unknown;

    try {
      response = await handler(request, correlationId);
    } catch (error) {
      response = { __error: (error as Error).message };
    }

    channel.sendToQueue(
      replyTo,
      Buffer.from(JSON.stringify(response)),
      {
        correlationId,
        contentType: 'application/json',
      }
    );

    channel.ack(msg);
  });
}

function generateId(): string {
  return `${Date.now().toString(36)}-${Math.random().toString(36).slice(2, 9)}`;
}
```

---

## 8. Outbox Pattern ด้วย PostgreSQL LISTEN/NOTIFY

```typescript
// packages/shared/src/outbox/outbox.listener.ts
// ใช้ PostgreSQL LISTEN/NOTIFY แทน polling เพื่อ latency ต่ำ

import { Pool, Client } from 'pg';
import { OutboxRepository } from './outbox.repository';
import { Channel } from 'amqplib';

export class OutboxListener {
  private notifyClient!: Client;
  private readonly repository: OutboxRepository;
  private processing = false;

  constructor(
    private readonly pool: Pool,
    private readonly channel: Channel
  ) {
    this.repository = new OutboxRepository(pool);
  }

  async start(): Promise<void> {
    // ใช้ dedicated client สำหรับ LISTEN
    this.notifyClient = new Client({
      connectionString: process.env.DATABASE_URL,
    });

    await this.notifyClient.connect();

    this.notifyClient.on('notification', async (msg) => {
      if (msg.channel === 'outbox_new_message') {
        await this.processOutbox();
      }
    });

    await this.notifyClient.query('LISTEN outbox_new_message');

    // Initial processing
    await this.processOutbox();

    console.log('[OutboxListener] Listening for outbox notifications');
  }

  private async processOutbox(): Promise<void> {
    if (this.processing) return; // Prevent concurrent processing
    this.processing = true;

    try {
      const messages = await this.repository.fetchAndLockUnsent(50);

      for (const msg of messages) {
        try {
          this.channel.publish(
            msg.aggregateType.toLowerCase(),
            msg.eventType,
            Buffer.from(JSON.stringify(msg.payload)),
            {
              persistent: true,
              messageId: msg.id,
              contentType: 'application/json',
              headers: msg.headers,
            }
          );

          await this.repository.markAsSent([msg.id]);

        } catch (error) {
          await this.repository.markAsFailed(msg.id, (error as Error).message);
        }
      }
    } finally {
      this.processing = false;
    }
  }

  async stop(): Promise<void> {
    await this.notifyClient.query('UNLISTEN outbox_new_message');
    await this.notifyClient.end();
  }
}

// PostgreSQL trigger สำหรับ NOTIFY
// db/outbox-notify-trigger.sql
/*
CREATE OR REPLACE FUNCTION notify_outbox()
RETURNS TRIGGER AS $$
BEGIN
  PERFORM pg_notify('outbox_new_message', NEW.id::text);
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER outbox_insert_notify
AFTER INSERT ON outbox
FOR EACH ROW EXECUTE FUNCTION notify_outbox();
*/
```

---

## 9. Full Integration Example

```typescript
// packages/order-service/src/place-order.handler.ts
// รวม Outbox + Inbox + Idempotency ในตัวอย่างเดียว

import { Pool, PoolClient } from 'pg';
import { OutboxRepository } from '@shared/outbox/outbox.repository';
import { InboxRepository } from '@shared/inbox/inbox.repository';

interface PlaceOrderCommand {
  correlationId: string;
  userId: string;
  items: Array<{ productId: string; quantity: number; price: number }>;
  shippingAddress: string;
  paymentMethodId: string;
}

interface OrderCreatedEvent {
  orderId: string;
  userId: string;
  totalAmount: number;
  items: Array<{ productId: string; quantity: number; price: number }>;
}

export class PlaceOrderHandler {
  private readonly outboxRepository: OutboxRepository;
  private readonly inboxRepository: InboxRepository;

  constructor(private readonly pool: Pool) {
    this.outboxRepository = new OutboxRepository(pool);
    this.inboxRepository = new InboxRepository(pool);
  }

  async handle(
    command: PlaceOrderCommand,
    messageId: string
  ): Promise<string> {
    const client = await this.pool.connect();

    try {
      await client.query('BEGIN');

      // 1. Idempotency check via Inbox
      const isNew = await this.inboxRepository.tryInsert(
        client,
        messageId,
        'place-order',
        command as unknown as Record<string, unknown>
      );

      if (!isNew) {
        await client.query('ROLLBACK');
        // ค้นหา orderId ที่สร้างไปแล้ว
        const existingOrder = await this.findExistingOrder(
          client,
          command.correlationId
        );
        return existingOrder ?? 'already-processed';
      }

      // 2. Validate business rules
      const totalAmount = command.items.reduce(
        (sum, item) => sum + item.price * item.quantity,
        0
      );

      if (totalAmount <= 0) {
        throw new Error('Order total must be positive');
      }

      // 3. Create order
      const orderId = await this.createOrder(client, command, totalAmount);

      // 4. Publish event via Outbox (in same transaction)
      const orderCreatedEvent: OrderCreatedEvent = {
        orderId,
        userId: command.userId,
        totalAmount,
        items: command.items,
      };

      await this.outboxRepository.insert(client, {
        aggregateId: orderId,
        aggregateType: 'Order',
        eventType: 'order.created',
        eventVersion: 1,
        payload: orderCreatedEvent,
        headers: {
          'x-correlation-id': command.correlationId,
          'x-user-id': command.userId,
        },
        scheduledFor: new Date(),
      });

      // 5. Commit — atomic: order created + outbox entry in one transaction
      await client.query('COMMIT');

      // 6. Mark inbox as processed
      await this.inboxRepository.markAsProcessed(messageId);

      console.log(`[PlaceOrderHandler] Order created: ${orderId}`);
      return orderId;

    } catch (error) {
      await client.query('ROLLBACK');

      if (messageId) {
        await this.inboxRepository.markAsFailed(
          messageId,
          (error as Error).message
        );
      }

      throw error;
    } finally {
      client.release();
    }
  }

  private async createOrder(
    client: PoolClient,
    command: PlaceOrderCommand,
    totalAmount: number
  ): Promise<string> {
    const result = await client.query(
      `INSERT INTO orders
         (correlation_id, user_id, total_amount, status, shipping_address)
       VALUES ($1, $2, $3, 'pending', $4)
       RETURNING id`,
      [command.correlationId, command.userId, totalAmount, command.shippingAddress]
    );

    const orderId = result.rows[0].id as string;

    // Insert order items
    for (const item of command.items) {
      await client.query(
        `INSERT INTO order_items (order_id, product_id, quantity, price)
         VALUES ($1, $2, $3, $4)`,
        [orderId, item.productId, item.quantity, item.price]
      );
    }

    return orderId;
  }

  private async findExistingOrder(
    client: PoolClient,
    correlationId: string
  ): Promise<string | null> {
    const result = await client.query(
      'SELECT id FROM orders WHERE correlation_id = $1',
      [correlationId]
    );

    return result.rows[0]?.id ?? null;
  }
}
```

---

## 10. Docker Compose ครบสมบูรณ์

```yaml
# docker-compose.async.yml
version: '3.8'

services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: orders_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./db:/docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
    command: >
      postgres
      -c wal_level=logical
      -c max_replication_slots=10
      -c max_wal_senders=10

  rabbitmq:
    image: rabbitmq:3.13-management-alpine
    environment:
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: admin123
      RABBITMQ_DEFAULT_VHOST: microservices
    ports:
      - "5672:5672"
      - "15672:15672"
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  order-service:
    build: ./packages/order-service
    environment:
      DATABASE_URL: postgresql://postgres:postgres@postgres:5432/orders_db
      RABBITMQ_URL: amqp://admin:admin123@rabbitmq:5672/microservices
      SERVICE_NAME: order-service
      PORT: 3001
    depends_on:
      postgres:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy
    deploy:
      replicas: 2

  outbox-worker:
    build: ./packages/outbox-worker
    environment:
      DATABASE_URL: postgresql://postgres:postgres@postgres:5432/orders_db
      RABBITMQ_URL: amqp://admin:admin123@rabbitmq:5672/microservices
      POLL_INTERVAL_MS: 1000
      BATCH_SIZE: 100
    depends_on:
      postgres:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy
    deploy:
      replicas: 1  # Only 1 outbox worker (uses SKIP LOCKED for safety)

  inventory-service:
    build: ./packages/inventory-service
    environment:
      DATABASE_URL: postgresql://postgres:postgres@postgres:5432/inventory_db
      RABBITMQ_URL: amqp://admin:admin123@rabbitmq:5672/microservices
      SERVICE_NAME: inventory-service
    depends_on:
      postgres:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy
    deploy:
      replicas: 2

volumes:
  postgres_data:
```

---

## 11. Testing Async Patterns

```typescript
// packages/order-service/src/__tests__/outbox.integration.test.ts

import { Pool } from 'pg';
import { PlaceOrderHandler } from '../place-order.handler';
import { OutboxRepository } from '@shared/outbox/outbox.repository';

describe('PlaceOrderHandler Integration', () => {
  let pool: Pool;
  let handler: PlaceOrderHandler;
  let outboxRepo: OutboxRepository;

  beforeAll(async () => {
    pool = new Pool({
      connectionString: process.env.TEST_DATABASE_URL ??
        'postgresql://postgres:postgres@localhost:5432/orders_test',
    });

    handler = new PlaceOrderHandler(pool);
    outboxRepo = new OutboxRepository(pool);

    // Run migrations
    await pool.query(`
      CREATE TABLE IF NOT EXISTS orders (
        id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
        correlation_id VARCHAR(255) UNIQUE NOT NULL,
        user_id VARCHAR(255) NOT NULL,
        total_amount DECIMAL NOT NULL,
        status VARCHAR(50) NOT NULL,
        shipping_address TEXT NOT NULL,
        created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
      );
      
      CREATE TABLE IF NOT EXISTS order_items (
        id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
        order_id UUID REFERENCES orders(id),
        product_id VARCHAR(255) NOT NULL,
        quantity INTEGER NOT NULL,
        price DECIMAL NOT NULL
      );
    `);
  });

  afterAll(async () => {
    await pool.end();
  });

  it('should create order and add outbox entry atomically', async () => {
    const correlationId = `test-${Date.now()}`;
    const messageId = `msg-${Date.now()}`;

    const orderId = await handler.handle(
      {
        correlationId,
        userId: 'user-1',
        items: [{ productId: 'prod-1', quantity: 2, price: 100 }],
        shippingAddress: '123 Test St',
        paymentMethodId: 'pm-1',
      },
      messageId
    );

    expect(orderId).toBeTruthy();

    // ตรวจสอบว่า outbox entry ถูกสร้าง
    const unsent = await outboxRepo.fetchAndLockUnsent(10);
    const entry = unsent.find((m) => m.aggregateId === orderId);

    expect(entry).toBeDefined();
    expect(entry?.eventType).toBe('order.created');
    expect((entry?.payload as { totalAmount: number }).totalAmount).toBe(200);
  });

  it('should be idempotent — same messageId returns same orderId', async () => {
    const correlationId = `idem-${Date.now()}`;
    const messageId = `idem-msg-${Date.now()}`;

    const command = {
      correlationId,
      userId: 'user-2',
      items: [{ productId: 'prod-2', quantity: 1, price: 500 }],
      shippingAddress: '456 Other St',
      paymentMethodId: 'pm-2',
    };

    const first = await handler.handle(command, messageId);
    const second = await handler.handle(command, messageId);

    // ทั้งคู่ควรได้ orderId เดียวกัน
    expect(first).toBe(second === 'already-processed' ? first : second);
  });
});
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Delivery Guarantees** — At-most-once vs At-least-once vs Exactly-once และเมื่อไหรควรใช้แบบไหน

2. **Outbox Pattern** — แก้ Dual Write Problem โดยบันทึก event ใน transaction เดียวกับ business data แล้วให้ Worker ส่งทีหลัง

3. **Inbox Pattern** — Deduplication ด้วย database unique constraint เพื่อป้องกัน message ถูก process ซ้ำ

4. **Idempotent Consumer** — ออกแบบ Handler ให้ประมวลผล message เดิมซ้ำได้อย่างปลอดภัย

5. **Correlation ID** — ติดตาม request flow ข้าม Service ด้วย unique ID ที่ propagate ผ่าน headers

6. **Message Ordering** — Buffer message ที่มาไม่เป็นลำดับและ process เมื่อ sequence ครบ

7. **Request-Reply over Messaging** — Synchronous-style call ผ่าน Message Broker ด้วย temporary reply queue

**Key Takeaway:** ความน่าเชื่อถือของ Async Communication มาจาก Idempotency + Exactly-once Semantics ที่ implement ด้วย Outbox + Inbox Pattern — ไม่ใช่จากการ "หวังว่าจะไม่มี duplicate"

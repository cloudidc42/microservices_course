# Part 56: Message Queue Patterns ด้วย RabbitMQ

## บทนำ

RabbitMQ เป็น Message Broker ที่ได้รับความนิยมสูงในระบบ Microservices เพราะรองรับหลาย Exchange Type และ Pattern ที่ซับซ้อน การเข้าใจ Exchange Types, Dead Letter Queues, Priority Queues, และ Consumer Patterns อย่างถ่องแท้จะช่วยให้ระบบมีความ Reliable และ Scalable

---

## 1. RabbitMQ Architecture Overview

```
Producer → Exchange → Binding → Queue → Consumer
              ↓ (ถ้าส่งไม่ได้หรือ TTL หมด)
          Dead Letter Exchange → Dead Letter Queue → DLQ Consumer
```

Exchange Types:
- **Direct**: Route ด้วย exact routing key
- **Topic**: Route ด้วย wildcard patterns (`*` = one word, `#` = zero or more)
- **Fanout**: Broadcast ทุก Queue ที่ bind
- **Headers**: Route ด้วย message headers

---

## 2. RabbitMQ Setup และ Configuration

```yaml
# docker-compose.rabbitmq.yml
version: '3.8'

services:
  rabbitmq:
    image: rabbitmq:3.13-management-alpine
    hostname: rabbitmq-1
    environment:
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: ${RABBITMQ_PASSWORD:-admin123}
      RABBITMQ_DEFAULT_VHOST: microservices
      # เพิ่ม plugins
      RABBITMQ_ENABLED_PLUGINS_FILE: /etc/rabbitmq/enabled_plugins
    ports:
      - "5672:5672"    # AMQP
      - "15672:15672"  # Management UI
      - "15692:15692"  # Prometheus metrics
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
      - ./rabbitmq/rabbitmq.conf:/etc/rabbitmq/rabbitmq.conf
      - ./rabbitmq/enabled_plugins:/etc/rabbitmq/enabled_plugins
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    ulimits:
      nofile:
        soft: 65536
        hard: 65536

volumes:
  rabbitmq_data:
```

```ini
# rabbitmq/rabbitmq.conf
# Memory high watermark — เริ่ม block producers เมื่อใช้ RAM ถึง 70%
vm_memory_high_watermark.relative = 0.7

# Disk free limit — block เมื่อ disk < 2GB
disk_free_limit.absolute = 2GB

# Max message size (20MB)
max_message_size = 20971520

# Consumer timeout
consumer_timeout = 1800000

# Heartbeat
heartbeat = 60

# Prometheus plugin
prometheus.path = /metrics

# Management
management.tcp.port = 15672
management.load_definitions = /etc/rabbitmq/definitions.json
```

```json
// rabbitmq/definitions.json
{
  "vhosts": [{ "name": "microservices" }],
  "users": [
    {
      "name": "admin",
      "password_hash": "...",
      "tags": "administrator"
    },
    {
      "name": "app",
      "password_hash": "...",
      "tags": ""
    }
  ],
  "permissions": [
    {
      "user": "app",
      "vhost": "microservices",
      "configure": "^(amq\\.gen.*|app\\..*)",
      "write": ".*",
      "read": ".*"
    }
  ],
  "exchanges": [
    {
      "name": "orders",
      "vhost": "microservices",
      "type": "topic",
      "durable": true,
      "auto_delete": false
    },
    {
      "name": "orders.dlx",
      "vhost": "microservices",
      "type": "direct",
      "durable": true,
      "auto_delete": false
    }
  ]
}
```

---

## 3. RabbitMQ Connection Manager

```typescript
// packages/shared/src/rabbitmq/connection.manager.ts

import amqplib, {
  Channel,
  ConfirmChannel,
  Connection,
  Options,
} from 'amqplib';
import { EventEmitter } from 'events';

interface ConnectionManagerOptions {
  url: string;
  heartbeat?: number;
  reconnectDelay?: number;
  maxReconnectAttempts?: number;
}

export class RabbitMQConnectionManager extends EventEmitter {
  private connection: Connection | null = null;
  private reconnectAttempts = 0;
  private isShuttingDown = false;

  constructor(private readonly options: ConnectionManagerOptions) {
    super();
  }

  async connect(): Promise<Connection> {
    try {
      this.connection = await amqplib.connect(this.options.url, {
        heartbeat: this.options.heartbeat ?? 60,
      });

      this.connection.on('error', (err) => {
        console.error('[RabbitMQ] Connection error:', err.message);
        this.emit('error', err);
      });

      this.connection.on('close', () => {
        if (!this.isShuttingDown) {
          console.warn('[RabbitMQ] Connection closed, reconnecting...');
          this.scheduleReconnect();
        }
      });

      this.reconnectAttempts = 0;
      this.emit('connected');
      console.log('[RabbitMQ] Connected successfully');

      return this.connection;
    } catch (error) {
      console.error('[RabbitMQ] Connection failed:', (error as Error).message);
      this.scheduleReconnect();
      throw error;
    }
  }

  async createChannel(): Promise<Channel> {
    if (!this.connection) {
      throw new Error('Not connected to RabbitMQ');
    }

    const channel = await this.connection.createChannel();
    channel.on('error', (err) => {
      console.error('[RabbitMQ] Channel error:', err.message);
    });

    return channel;
  }

  async createConfirmChannel(): Promise<ConfirmChannel> {
    if (!this.connection) {
      throw new Error('Not connected to RabbitMQ');
    }

    const channel = await this.connection.createConfirmChannel();
    channel.on('error', (err) => {
      console.error('[RabbitMQ] Confirm channel error:', err.message);
    });

    return channel;
  }

  private scheduleReconnect(): void {
    const maxAttempts = this.options.maxReconnectAttempts ?? -1; // -1 = infinite
    if (maxAttempts !== -1 && this.reconnectAttempts >= maxAttempts) {
      console.error('[RabbitMQ] Max reconnect attempts reached');
      this.emit('max-reconnect-exceeded');
      return;
    }

    this.reconnectAttempts++;
    const delay = Math.min(
      (this.options.reconnectDelay ?? 5000) * Math.pow(2, this.reconnectAttempts - 1),
      60000 // max 60s
    );

    console.log(
      `[RabbitMQ] Reconnecting in ${delay}ms (attempt ${this.reconnectAttempts})`
    );

    setTimeout(() => {
      this.connect().catch(() => {
        // Error handled in connect()
      });
    }, delay);
  }

  async close(): Promise<void> {
    this.isShuttingDown = true;
    if (this.connection) {
      await this.connection.close();
      this.connection = null;
    }
  }
}
```

---

## 4. Exchange Types ตัวอย่าง

```typescript
// packages/shared/src/rabbitmq/exchanges.setup.ts

import { Channel } from 'amqplib';

export async function setupExchanges(channel: Channel): Promise<void> {
  // ─── 1. Direct Exchange ─────────────────────────────────────────────────
  // ส่ง message ไปยัง queue ที่ routing key ตรงกัน
  await channel.assertExchange('orders.direct', 'direct', {
    durable: true,
    autoDelete: false,
  });

  // ─── 2. Topic Exchange ──────────────────────────────────────────────────
  // ส่ง message ด้วย wildcard pattern
  // routing key: "order.created.thailand" → ตรงกับ "order.*.thailand", "order.#"
  await channel.assertExchange('orders.topic', 'topic', {
    durable: true,
    autoDelete: false,
  });

  // ─── 3. Fanout Exchange ─────────────────────────────────────────────────
  // Broadcast ไปทุก Queue ที่ bind (ไม่สนใจ routing key)
  await channel.assertExchange('notifications.fanout', 'fanout', {
    durable: true,
    autoDelete: false,
  });

  // ─── 4. Headers Exchange ────────────────────────────────────────────────
  // Route ด้วย message headers
  await channel.assertExchange('orders.headers', 'headers', {
    durable: true,
    autoDelete: false,
  });

  // ─── Dead Letter Exchange ───────────────────────────────────────────────
  await channel.assertExchange('orders.dlx', 'direct', {
    durable: true,
    autoDelete: false,
  });

  console.log('[RabbitMQ] Exchanges set up');
}

export async function setupQueues(channel: Channel): Promise<void> {
  // ─── Queue with Dead Letter Exchange ────────────────────────────────────
  await channel.assertQueue('order.processing', {
    durable: true,
    arguments: {
      'x-dead-letter-exchange': 'orders.dlx',
      'x-dead-letter-routing-key': 'order.processing.dead',
      'x-message-ttl': 300000,      // 5 minutes TTL
      'x-max-length': 10000,         // Max 10k messages
      'x-max-length-bytes': 100 * 1024 * 1024, // 100MB max
      'x-overflow': 'reject-publish', // Reject new messages when full
    },
  });

  // DLQ
  await channel.assertQueue('order.processing.dlq', {
    durable: true,
    arguments: {
      'x-message-ttl': 7 * 24 * 60 * 60 * 1000, // Keep 7 days
    },
  });

  // ─── Priority Queue ─────────────────────────────────────────────────────
  await channel.assertQueue('order.priority', {
    durable: true,
    arguments: {
      'x-max-priority': 10, // Priority 0-10 (10 = highest)
      'x-dead-letter-exchange': 'orders.dlx',
      'x-dead-letter-routing-key': 'order.priority.dead',
    },
  });

  // ─── Bindings ────────────────────────────────────────────────────────────

  // Direct binding
  await channel.bindQueue('order.processing', 'orders.direct', 'new-order');

  // Topic bindings
  await channel.bindQueue(
    'order.processing',
    'orders.topic',
    'order.created.*' // ตรงกับ order.created.anything
  );
  await channel.bindQueue(
    'order.priority',
    'orders.topic',
    'order.vip.#' // ตรงกับ order.vip.anything.or.more
  );

  // DLQ binding
  await channel.bindQueue(
    'order.processing.dlq',
    'orders.dlx',
    'order.processing.dead'
  );

  // Fanout binding (ไม่ต้องระบุ routing key)
  await channel.bindQueue('notifications.email', 'notifications.fanout', '');
  await channel.bindQueue('notifications.sms', 'notifications.fanout', '');
  await channel.bindQueue('notifications.push', 'notifications.fanout', '');

  // Headers binding
  await channel.bindQueue(
    'order.processing',
    'orders.headers',
    '', // routing key ไม่สำคัญ
    {
      'x-match': 'all',         // ต้อง match ทุก header
      'region': 'thailand',
      'order-type': 'express',
    }
  );

  console.log('[RabbitMQ] Queues and bindings set up');
}
```

---

## 5. Publisher พร้อม Publisher Confirms

```typescript
// packages/shared/src/rabbitmq/publisher.ts

import { ConfirmChannel, Options } from 'amqplib';

export interface PublishOptions {
  persistent?: boolean;
  priority?: number; // 0-10 สำหรับ priority queues
  expiration?: string; // TTL เป็น milliseconds string
  messageId?: string;
  correlationId?: string;
  replyTo?: string;
  headers?: Record<string, string | number | boolean>;
  contentType?: string;
  mandatory?: boolean; // ส่ง basic.return ถ้า unroutable
}

export interface PublishResult {
  success: boolean;
  messageId: string;
  exchange: string;
  routingKey: string;
  error?: Error;
}

export class RabbitMQPublisher {
  private channel!: ConfirmChannel;
  private pendingConfirms: Map<
    number,
    { resolve: () => void; reject: (err: Error) => void }
  > = new Map();
  private deliveryTag = 0;

  constructor(private readonly channelProvider: () => Promise<ConfirmChannel>) {}

  async initialize(): Promise<void> {
    this.channel = await this.channelProvider();

    // Publisher Confirms
    this.channel.on('ack', (seqNum: number, multiple: boolean) => {
      if (multiple) {
        // Confirm ทุก message ที่ seqNum <= seqNum
        for (const [tag, { resolve }] of this.pendingConfirms) {
          if (tag <= seqNum) {
            resolve();
            this.pendingConfirms.delete(tag);
          }
        }
      } else {
        this.pendingConfirms.get(seqNum)?.resolve();
        this.pendingConfirms.delete(seqNum);
      }
    });

    this.channel.on('nack', (seqNum: number, multiple: boolean) => {
      const error = new Error('Message nacked by broker');
      if (multiple) {
        for (const [tag, { reject }] of this.pendingConfirms) {
          if (tag <= seqNum) {
            reject(error);
            this.pendingConfirms.delete(tag);
          }
        }
      } else {
        this.pendingConfirms.get(seqNum)?.reject(error);
        this.pendingConfirms.delete(seqNum);
      }
    });

    // Handle unroutable messages
    this.channel.on('return', (msg) => {
      console.warn(
        '[Publisher] Message returned (unroutable):',
        msg.fields.routingKey
      );
    });
  }

  async publish(
    exchange: string,
    routingKey: string,
    message: unknown,
    options: PublishOptions = {}
  ): Promise<PublishResult> {
    const messageId = options.messageId ?? generateMessageId();
    const content = Buffer.from(JSON.stringify(message));
    const seqNum = this.channel.getNextPublishSeqNo();

    const publishOptions: Options.Publish = {
      persistent: options.persistent ?? true,
      messageId,
      correlationId: options.correlationId,
      replyTo: options.replyTo,
      contentType: options.contentType ?? 'application/json',
      timestamp: Math.floor(Date.now() / 1000),
      headers: {
        ...options.headers,
        'x-origin-service': process.env.SERVICE_NAME ?? 'unknown',
        'x-publish-time': new Date().toISOString(),
      },
      priority: options.priority,
      expiration: options.expiration,
      mandatory: options.mandatory ?? false,
    };

    return new Promise<PublishResult>((resolve, reject) => {
      this.pendingConfirms.set(seqNum, {
        resolve: () =>
          resolve({
            success: true,
            messageId,
            exchange,
            routingKey,
          }),
        reject: (err) =>
          reject({
            success: false,
            messageId,
            exchange,
            routingKey,
            error: err,
          }),
      });

      const ok = this.channel.publish(
        exchange,
        routingKey,
        content,
        publishOptions
      );

      if (!ok) {
        // Channel buffer is full — wait for drain
        this.channel.once('drain', () => {
          console.log('[Publisher] Channel drained, ready to publish again');
        });
      }
    });
  }

  // Batch publish
  async publishBatch(
    messages: Array<{
      exchange: string;
      routingKey: string;
      message: unknown;
      options?: PublishOptions;
    }>
  ): Promise<PublishResult[]> {
    return Promise.all(
      messages.map(({ exchange, routingKey, message, options }) =>
        this.publish(exchange, routingKey, message, options)
      )
    );
  }

  // Publish to queue directly (default exchange)
  async sendToQueue(
    queue: string,
    message: unknown,
    options: PublishOptions = {}
  ): Promise<PublishResult> {
    return this.publish('', queue, message, options);
  }
}

function generateMessageId(): string {
  return `msg-${Date.now()}-${Math.random().toString(36).slice(2, 9)}`;
}
```

---

## 6. Consumer พร้อม Prefetch และ Acknowledgment

```typescript
// packages/shared/src/rabbitmq/consumer.ts

import { Channel, ConsumeMessage, Options } from 'amqplib';
import { Histogram, Counter } from 'prom-client';

export type MessageHandler<T = unknown> = (
  message: T,
  metadata: MessageMetadata
) => Promise<void>;

export interface MessageMetadata {
  messageId: string;
  correlationId?: string;
  routingKey: string;
  exchange: string;
  headers: Record<string, unknown>;
  deliveryTag: number;
  redelivered: boolean;
  timestamp?: Date;
  retryCount: number;
}

export interface ConsumerOptions {
  prefetchCount?: number;  // Flow control — max unacked messages
  prefetchGlobal?: boolean; // Apply prefetch to channel or consumer
  requeue?: boolean;       // Requeue on failure
  maxRetries?: number;
}

export class RabbitMQConsumer {
  private readonly processingDuration: Histogram<string>;
  private readonly messagesConsumed: Counter<string>;
  private readonly messagesFailed: Counter<string>;

  constructor(
    private readonly channel: Channel,
    metrics?: {
      processingDuration: Histogram<string>;
      messagesConsumed: Counter<string>;
      messagesFailed: Counter<string>;
    }
  ) {
    this.processingDuration = metrics?.processingDuration ?? new Histogram({
      name: 'mq_message_processing_duration_seconds',
      help: 'Message processing duration',
      labelNames: ['queue'],
      buckets: [0.01, 0.05, 0.1, 0.5, 1, 5, 10],
    });
    this.messagesConsumed = metrics?.messagesConsumed ?? new Counter({
      name: 'mq_messages_consumed_total',
      help: 'Messages consumed',
      labelNames: ['queue', 'status'],
    });
    this.messagesFailed = metrics?.messagesFailed ?? new Counter({
      name: 'mq_messages_failed_total',
      help: 'Messages failed',
      labelNames: ['queue'],
    });
  }

  async consume<T = unknown>(
    queue: string,
    handler: MessageHandler<T>,
    options: ConsumerOptions = {}
  ): Promise<string> {
    const {
      prefetchCount = 10,
      prefetchGlobal = false,
      requeue = false,
      maxRetries = 3,
    } = options;

    // ตั้ง prefetch — Flow control
    await this.channel.prefetch(prefetchCount, prefetchGlobal);

    const { consumerTag } = await this.channel.consume(queue, async (msg) => {
      if (!msg) return; // Consumer cancelled

      const timer = this.processingDuration.startTimer({ queue });
      const metadata = extractMetadata(msg);

      try {
        const body = JSON.parse(msg.content.toString()) as T;

        // ตรวจสอบ retry count
        if (metadata.retryCount > maxRetries) {
          console.warn(
            `[Consumer] Message exceeded max retries (${maxRetries}), sending to DLQ`,
            { messageId: metadata.messageId, queue }
          );
          this.channel.nack(msg, false, false); // requeue=false → DLQ
          this.messagesConsumed.inc({ queue, status: 'dlq' });
          timer();
          return;
        }

        await handler(body, metadata);

        this.channel.ack(msg);
        this.messagesConsumed.inc({ queue, status: 'success' });

      } catch (error) {
        const err = error as Error;
        console.error(
          `[Consumer] Error processing message`,
          {
            messageId: metadata.messageId,
            queue,
            error: err.message,
            retryCount: metadata.retryCount,
          }
        );

        this.messagesFailed.inc({ queue });

        // ตัดสินใจ: requeue หรือ DLQ
        const shouldRequeue = requeue && metadata.retryCount < maxRetries;
        this.channel.nack(msg, false, shouldRequeue);
        this.messagesConsumed.inc({
          queue,
          status: shouldRequeue ? 'requeued' : 'failed',
        });

      } finally {
        timer();
      }
    });

    console.log(`[Consumer] Started consuming from ${queue} (tag: ${consumerTag})`);
    return consumerTag;
  }

  // Competing Consumers Pattern — หลาย consumer process จาก queue เดียวกัน
  async consumeWithConcurrency<T = unknown>(
    queue: string,
    handler: MessageHandler<T>,
    concurrency: number,
    options: Omit<ConsumerOptions, 'prefetchCount'> = {}
  ): Promise<string[]> {
    const tags: string[] = [];

    for (let i = 0; i < concurrency; i++) {
      const tag = await this.consume(queue, handler, {
        ...options,
        prefetchCount: 1, // Each consumer processes 1 at a time
      });
      tags.push(tag);
    }

    return tags;
  }
}

function extractMetadata(msg: ConsumeMessage): MessageMetadata {
  const headers = msg.properties.headers ?? {};
  const retryCount = typeof headers['x-death'] === 'object' && headers['x-death'] !== null
    ? (headers['x-death'] as Array<{ count: number }>)[0]?.count ?? 0
    : 0;

  return {
    messageId: msg.properties.messageId ?? `unknown-${msg.fields.deliveryTag}`,
    correlationId: msg.properties.correlationId,
    routingKey: msg.fields.routingKey,
    exchange: msg.fields.exchange,
    headers: headers as Record<string, unknown>,
    deliveryTag: msg.fields.deliveryTag,
    redelivered: msg.fields.redelivered,
    timestamp: msg.properties.timestamp
      ? new Date(msg.properties.timestamp * 1000)
      : undefined,
    retryCount: Number(retryCount),
  };
}
```

---

## 7. Dead Letter Queue Handler

```typescript
// packages/shared/src/rabbitmq/dlq.handler.ts

import { Channel } from 'amqplib';

interface DeadLetterMessage {
  originalQueue: string;
  originalExchange: string;
  originalRoutingKey: string;
  reason: string;
  deathCount: number;
  content: unknown;
  timestamp: Date;
  messageId: string;
}

export class DeadLetterHandler {
  constructor(
    private readonly channel: Channel,
    private readonly storage: DLQStorage
  ) {}

  async startProcessing(dlqName: string): Promise<void> {
    await this.channel.prefetch(5);

    await this.channel.consume(dlqName, async (msg) => {
      if (!msg) return;

      try {
        const xDeath = msg.properties.headers?.['x-death'];
        const deaths = Array.isArray(xDeath) ? xDeath : [];

        const dlm: DeadLetterMessage = {
          originalQueue: deaths[0]?.queue ?? 'unknown',
          originalExchange: deaths[0]?.exchange ?? 'unknown',
          originalRoutingKey: Array.isArray(deaths[0]?.['routing-keys'])
            ? deaths[0]['routing-keys'][0]
            : 'unknown',
          reason: deaths[0]?.reason ?? 'unknown',
          deathCount: deaths[0]?.count ?? 1,
          content: JSON.parse(msg.content.toString()),
          timestamp: new Date(),
          messageId: msg.properties.messageId ?? 'unknown',
        };

        // บันทึกลง Storage สำหรับ manual inspection
        await this.storage.save(dlm);

        // Notify operations team
        await this.notifyOperations(dlm);

        this.channel.ack(msg);

        console.log(`[DLQ] Processed dead letter: ${dlm.messageId}`, {
          originalQueue: dlm.originalQueue,
          reason: dlm.reason,
          deathCount: dlm.deathCount,
        });

      } catch (error) {
        console.error('[DLQ] Error processing dead letter:', error);
        this.channel.nack(msg, false, false); // ไม่ requeue
      }
    });

    console.log(`[DLQ] Monitoring dead letters from: ${dlqName}`);
  }

  // Retry dead letter — ส่งกลับไป original queue
  async retry(messageId: string): Promise<boolean> {
    const dlm = await this.storage.findById(messageId);
    if (!dlm) return false;

    this.channel.publish(
      dlm.originalExchange,
      dlm.originalRoutingKey,
      Buffer.from(JSON.stringify(dlm.content)),
      {
        persistent: true,
        messageId: dlm.messageId,
        headers: {
          'x-retry-from-dlq': true,
          'x-retry-timestamp': new Date().toISOString(),
        },
      }
    );

    await this.storage.markRetried(messageId);
    return true;
  }

  private async notifyOperations(dlm: DeadLetterMessage): Promise<void> {
    // Send to Slack, PagerDuty, etc.
    console.error(`[DLQ ALERT] Dead letter from ${dlm.originalQueue}:`, {
      messageId: dlm.messageId,
      reason: dlm.reason,
      deathCount: dlm.deathCount,
    });
  }
}

interface DLQStorage {
  save(message: DeadLetterMessage): Promise<void>;
  findById(messageId: string): Promise<DeadLetterMessage | null>;
  markRetried(messageId: string): Promise<void>;
}
```

---

## 8. Priority Queue ตัวอย่าง

```typescript
// packages/order-service/src/priority.producer.ts

import { RabbitMQPublisher } from '@shared/rabbitmq/publisher';

interface OrderMessage {
  orderId: string;
  userId: string;
  items: Array<{ productId: string; quantity: number }>;
  totalAmount: number;
  priority: 'vip' | 'premium' | 'standard';
}

export class OrderPriorityProducer {
  constructor(private readonly publisher: RabbitMQPublisher) {}

  async publishOrder(order: OrderMessage): Promise<void> {
    // Map priority ไปยัง RabbitMQ priority number (0-10)
    const priorityMap = {
      vip: 9,
      premium: 6,
      standard: 3,
    };

    const mqPriority = priorityMap[order.priority];

    await this.publisher.publish(
      'orders.topic',
      `order.created.${order.priority}`,
      order,
      {
        priority: mqPriority,
        messageId: `order-${order.orderId}`,
        headers: {
          'order-priority': order.priority,
          'user-id': order.userId,
        },
      }
    );

    console.log(`[OrderProducer] Published order ${order.orderId} with priority ${mqPriority}`);
  }
}

// ตัวอย่างการส่ง VIP order ที่มี priority สูง
// order.vip.# → order.priority queue (priority 9)
// order.standard.* → order.processing queue (priority 3)
```

---

## 9. Work Queue Pattern

```typescript
// packages/notification-service/src/work.queue.ts
// Work Queue: หลาย Worker แย่ง consume จาก queue เดียวกัน

import { Channel } from 'amqplib';

interface EmailTask {
  to: string;
  subject: string;
  templateId: string;
  data: Record<string, unknown>;
}

export class EmailWorker {
  private readonly workerId: string;

  constructor(
    private readonly channel: Channel,
    workerIndex: number
  ) {
    this.workerId = `worker-${workerIndex}`;
  }

  async start(): Promise<void> {
    // prefetch=1 ทำให้แต่ละ worker รับงานทีละ 1 ชิ้น
    // Worker ที่ทำงานเสร็จเร็วจะรับงานใหม่ได้เร็วกว่า (fair dispatch)
    await this.channel.prefetch(1);

    await this.channel.consume('email.tasks', async (msg) => {
      if (!msg) return;

      const task: EmailTask = JSON.parse(msg.content.toString());

      console.log(`[${this.workerId}] Processing email task:`, {
        to: task.to,
        subject: task.subject,
      });

      try {
        await this.sendEmail(task);
        this.channel.ack(msg);
        console.log(`[${this.workerId}] Email sent successfully`);
      } catch (error) {
        console.error(`[${this.workerId}] Failed to send email:`, error);
        // Nack — ส่งไป DLQ
        this.channel.nack(msg, false, false);
      }
    });

    console.log(`[${this.workerId}] Email worker started`);
  }

  private async sendEmail(task: EmailTask): Promise<void> {
    // Simulate email sending
    await new Promise((resolve) => setTimeout(resolve, 100 + Math.random() * 500));
    if (Math.random() < 0.01) {
      throw new Error('Email service temporarily unavailable');
    }
  }
}

// ─── Start multiple workers ────────────────────────────────────────────────

export async function startEmailWorkerPool(
  channel: Channel,
  workerCount: number
): Promise<EmailWorker[]> {
  const workers: EmailWorker[] = [];

  for (let i = 0; i < workerCount; i++) {
    const worker = new EmailWorker(channel, i);
    await worker.start();
    workers.push(worker);
  }

  console.log(`[WorkerPool] Started ${workerCount} email workers`);
  return workers;
}
```

---

## 10. Message TTL และ Per-Message TTL

```typescript
// packages/shared/src/rabbitmq/ttl.example.ts

import { Channel } from 'amqplib';

export async function setupTTLQueues(channel: Channel): Promise<void> {
  // ─── Queue-level TTL: ทุก message ใน queue มี TTL เดียวกัน ─────────────
  await channel.assertQueue('session.events', {
    durable: true,
    arguments: {
      'x-message-ttl': 30 * 60 * 1000, // 30 minutes
      'x-dead-letter-exchange': 'session.dlx',
    },
  });

  // ─── Per-Message TTL: แต่ละ message มี TTL ต่างกัน ─────────────────────
  // ต้องตั้ง x-message-ttl บน queue ก่อน หรือส่ง expiration ใน message
}

// ส่ง message พร้อม TTL เฉพาะของมันเอง
export async function publishWithTTL(
  channel: Channel,
  queue: string,
  message: unknown,
  ttlMs: number
): Promise<void> {
  channel.sendToQueue(
    queue,
    Buffer.from(JSON.stringify(message)),
    {
      persistent: true,
      expiration: String(ttlMs), // เป็น string!
      messageId: `msg-${Date.now()}`,
    }
  );
}

// ตัวอย่าง: OTP message หมดอายุใน 5 นาที
// publishWithTTL(channel, 'otp.requests', { userId, otp }, 5 * 60 * 1000);
```

---

## 11. Request-Reply Pattern ใน RabbitMQ

```typescript
// packages/shared/src/rabbitmq/rpc.ts
// RPC over RabbitMQ — synchronous-style request-reply

import { Channel, ConsumeMessage } from 'amqplib';

export class RabbitMQRPC {
  private pendingRequests: Map<
    string,
    { resolve: (value: unknown) => void; reject: (error: Error) => void; timer: ReturnType<typeof setTimeout> }
  > = new Map();
  private replyQueue!: string;

  constructor(private readonly channel: Channel) {}

  async initialize(): Promise<void> {
    // สร้าง exclusive reply queue
    const { queue } = await this.channel.assertQueue('', {
      exclusive: true,
      autoDelete: true,
    });
    this.replyQueue = queue;

    // รอ reply messages
    await this.channel.consume(
      this.replyQueue,
      (msg) => {
        if (!msg) return;

        const correlationId = msg.properties.correlationId;
        const pending = this.pendingRequests.get(correlationId);

        if (pending) {
          clearTimeout(pending.timer);
          this.pendingRequests.delete(correlationId);

          try {
            const response = JSON.parse(msg.content.toString());
            if (response.error) {
              pending.reject(new Error(response.error));
            } else {
              pending.resolve(response.data);
            }
          } catch (error) {
            pending.reject(error as Error);
          }

          this.channel.ack(msg);
        }
      },
      { noAck: false }
    );

    console.log(`[RPC] Reply queue: ${this.replyQueue}`);
  }

  async call<TRequest, TResponse>(
    exchange: string,
    routingKey: string,
    request: TRequest,
    timeoutMs = 30000
  ): Promise<TResponse> {
    const correlationId = `rpc-${Date.now()}-${Math.random().toString(36).slice(2)}`;

    return new Promise<TResponse>((resolve, reject) => {
      const timer = setTimeout(() => {
        this.pendingRequests.delete(correlationId);
        reject(new Error(`RPC timeout after ${timeoutMs}ms`));
      }, timeoutMs);

      this.pendingRequests.set(correlationId, {
        resolve: resolve as (value: unknown) => void,
        reject,
        timer,
      });

      this.channel.publish(
        exchange,
        routingKey,
        Buffer.from(JSON.stringify(request)),
        {
          correlationId,
          replyTo: this.replyQueue,
          persistent: false, // RPC requests are usually ephemeral
        }
      );
    });
  }
}

// RPC Server side
export async function createRPCServer<TRequest, TResponse>(
  channel: Channel,
  queue: string,
  handler: (request: TRequest) => Promise<TResponse>
): Promise<void> {
  await channel.prefetch(1);

  await channel.consume(queue, async (msg) => {
    if (!msg) return;

    const request = JSON.parse(msg.content.toString()) as TRequest;
    let response: { data?: TResponse; error?: string };

    try {
      const data = await handler(request);
      response = { data };
    } catch (error) {
      response = { error: (error as Error).message };
    }

    if (msg.properties.replyTo) {
      channel.sendToQueue(
        msg.properties.replyTo,
        Buffer.from(JSON.stringify(response)),
        { correlationId: msg.properties.correlationId }
      );
    }

    channel.ack(msg);
  });
}
```

---

## 12. Full Setup Example

```typescript
// packages/order-service/src/messaging.setup.ts

import { RabbitMQConnectionManager } from '@shared/rabbitmq/connection.manager';
import { setupExchanges, setupQueues } from '@shared/rabbitmq/exchanges.setup';
import { RabbitMQPublisher } from '@shared/rabbitmq/publisher';
import { RabbitMQConsumer } from '@shared/rabbitmq/consumer';
import { DeadLetterHandler } from '@shared/rabbitmq/dlq.handler';
import { startEmailWorkerPool } from './work.queue';

export async function setupMessaging() {
  const connectionManager = new RabbitMQConnectionManager({
    url: process.env.RABBITMQ_URL ?? 'amqp://admin:admin123@rabbitmq:5672/microservices',
    heartbeat: 60,
    reconnectDelay: 5000,
    maxReconnectAttempts: -1, // infinite
  });

  const connection = await connectionManager.connect();

  // Setup channel สำหรับ setup (ไม่ใช้ consume)
  const setupChannel = await connectionManager.createChannel();
  await setupExchanges(setupChannel);
  await setupQueues(setupChannel);
  await setupChannel.close();

  // Publisher channel (confirm channel)
  const publishChannel = await connectionManager.createConfirmChannel();
  const publisher = new RabbitMQPublisher(() =>
    Promise.resolve(publishChannel)
  );
  await publisher.initialize();

  // Consumer channel
  const consumeChannel = await connectionManager.createChannel();
  const consumer = new RabbitMQConsumer(consumeChannel);

  // Start consuming order events
  await consumer.consume<OrderCreatedEvent>(
    'order.processing',
    async (event, metadata) => {
      console.log(`Processing order: ${event.orderId}`, {
        messageId: metadata.messageId,
        retryCount: metadata.retryCount,
      });

      // Process order...
      await processOrder(event);
    },
    {
      prefetchCount: 10,
      maxRetries: 3,
    }
  );

  // DLQ monitoring
  const dlqChannel = await connectionManager.createChannel();
  const dlqHandler = new DeadLetterHandler(dlqChannel, createDLQStorage());
  await dlqHandler.startProcessing('order.processing.dlq');

  // Email workers (work queue pattern)
  const emailChannel = await connectionManager.createChannel();
  await startEmailWorkerPool(emailChannel, 3);

  console.log('[Messaging] All messaging components started');

  return { publisher, consumer, connectionManager };
}

interface OrderCreatedEvent {
  orderId: string;
  userId: string;
  items: Array<{ productId: string; quantity: number }>;
  totalAmount: number;
}

async function processOrder(event: OrderCreatedEvent): Promise<void> {
  // Business logic here
  console.log(`Processing order: ${event.orderId}`);
}

function createDLQStorage() {
  // Simple in-memory storage (use database in production)
  const storage = new Map<string, unknown>();
  return {
    async save(msg: unknown) {
      const m = msg as { messageId: string };
      storage.set(m.messageId, msg);
    },
    async findById(id: string) {
      return storage.get(id) ?? null;
    },
    async markRetried(id: string) {
      const msg = storage.get(id) as Record<string, unknown> | undefined;
      if (msg) {
        storage.set(id, { ...msg, retried: true, retriedAt: new Date() });
      }
    },
  };
}
```

---

## 13. Docker Compose ครบสมบูรณ์

```yaml
# docker-compose.full-messaging.yml
version: '3.8'

services:
  rabbitmq:
    image: rabbitmq:3.13-management-alpine
    hostname: rabbitmq-1
    environment:
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: admin123
      RABBITMQ_DEFAULT_VHOST: microservices
    ports:
      - "5672:5672"
      - "15672:15672"
      - "15692:15692"
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
      - ./rabbitmq/rabbitmq.conf:/etc/rabbitmq/rabbitmq.conf
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    ulimits:
      nofile:
        soft: 65536
        hard: 65536

  order-service:
    build: ./packages/order-service
    environment:
      RABBITMQ_URL: amqp://admin:admin123@rabbitmq:5672/microservices
      DB_HOST: postgres
    depends_on:
      rabbitmq:
        condition: service_healthy
    deploy:
      replicas: 2

  notification-service:
    build: ./packages/notification-service
    environment:
      RABBITMQ_URL: amqp://admin:admin123@rabbitmq:5672/microservices
      SMTP_HOST: mailhog
      SMTP_PORT: 1025
    depends_on:
      rabbitmq:
        condition: service_healthy
    deploy:
      replicas: 3  # Competing consumers

  # Dev SMTP
  mailhog:
    image: mailhog/mailhog:latest
    ports:
      - "1025:1025"
      - "8025:8025"

volumes:
  rabbitmq_data:
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Exchange Types** — Direct (exact match), Topic (wildcard), Fanout (broadcast), Headers (attribute-based routing)

2. **Dead Letter Queues** — Route failed messages ไปยัง DLQ เพื่อ inspection และ retry manual

3. **Message TTL** — Queue-level TTL และ Per-message TTL สำหรับ time-sensitive messages

4. **Priority Queues** — Queue ที่ process high-priority messages ก่อน (ใช้กับ VIP customers)

5. **Publisher Confirms** — ยืนยันว่า Broker รับ message แล้วก่อน return success

6. **Consumer Prefetch** — Control flow ด้วย prefetch count เพื่อป้องกัน consumer รับงานมากเกินไป

7. **Work Queue Pattern** — Competing consumers สำหรับ parallel processing

8. **RPC over RabbitMQ** — Request-reply pattern ด้วย correlation ID และ reply queue

**Best Practice:** ตั้งค่า DLQ ทุก queue, ใช้ Publisher Confirms ใน production, และ monitor queue depth อย่างต่อเนื่อง

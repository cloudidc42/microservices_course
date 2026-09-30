# Part 56: RabbitMQ Message Patterns — Dead Letter Queues, Exchange Types, Priority Queues, และ Connection Recovery

ในบทนี้เราจะเรียนรู้ RabbitMQ Message Patterns ขั้นสูงอย่างละเอียด ครอบคลุม Dead Letter Queues, Exchange types ทั้งหมด, Priority Queues, Consumer Acknowledgment, Publisher Confirms, และ Connection Recovery

---

## 1. RabbitMQ TypeScript Client Setup

```typescript
// src/messaging/rabbitmq-client.ts
import amqplib, { Connection, Channel, ConfirmChannel, Options } from 'amqplib';
import { EventEmitter } from 'events';
import { log } from '../telemetry/logger';

interface RabbitMQConfig {
  url: string;
  prefetchCount?: number;
  heartbeat?: number;
  reconnectDelay?: number;
  maxReconnectAttempts?: number;
}

export class RabbitMQClient extends EventEmitter {
  private connection: Connection | null = null;
  private channel: Channel | null = null;
  private confirmChannel: ConfirmChannel | null = null;
  private readonly config: Required<RabbitMQConfig>;
  private reconnectAttempts = 0;
  private isShuttingDown = false;
  private reconnectTimer: NodeJS.Timeout | null = null;
  
  constructor(config: RabbitMQConfig) {
    super();
    this.config = {
      prefetchCount: 10,
      heartbeat: 60,
      reconnectDelay: 5000,
      maxReconnectAttempts: Infinity,
      ...config,
    };
  }
  
  async connect(): Promise<void> {
    try {
      this.connection = await amqplib.connect(this.config.url, {
        heartbeat: this.config.heartbeat,
      });
      
      this.connection.on('error', (err) => {
        log.error('RabbitMQ connection error', err);
        this.emit('connection:error', err);
        this.scheduleReconnect();
      });
      
      this.connection.on('close', () => {
        if (!this.isShuttingDown) {
          log.error('RabbitMQ connection closed unexpectedly');
          this.emit('connection:closed');
          this.scheduleReconnect();
        }
      });
      
      this.channel = await this.connection.createChannel();
      this.channel.on('error', (err) => {
        log.error('RabbitMQ channel error', err);
        this.emit('channel:error', err);
      });
      
      await this.channel.prefetch(this.config.prefetchCount);
      
      this.confirmChannel = await this.connection.createConfirmChannel();
      await this.confirmChannel.prefetch(this.config.prefetchCount);
      
      this.reconnectAttempts = 0;
      log.info('Connected to RabbitMQ');
      this.emit('connected');
    } catch (error) {
      log.error('Failed to connect to RabbitMQ', error as Error);
      this.scheduleReconnect();
    }
  }
  
  private scheduleReconnect(): void {
    if (this.isShuttingDown) return;
    if (this.reconnectAttempts >= this.config.maxReconnectAttempts) {
      log.error('Max reconnect attempts reached');
      this.emit('reconnect:failed');
      return;
    }
    
    this.reconnectAttempts++;
    const delay = Math.min(
      this.config.reconnectDelay * Math.pow(2, Math.min(this.reconnectAttempts - 1, 5)),
      60000 // Max 60 second backoff
    );
    
    log.info(`Reconnecting to RabbitMQ in ${delay}ms (attempt ${this.reconnectAttempts})`);
    
    this.reconnectTimer = setTimeout(() => {
      this.connection = null;
      this.channel = null;
      this.confirmChannel = null;
      this.connect();
    }, delay);
  }
  
  async disconnect(): Promise<void> {
    this.isShuttingDown = true;
    
    if (this.reconnectTimer) {
      clearTimeout(this.reconnectTimer);
    }
    
    try {
      await this.channel?.close();
      await this.confirmChannel?.close();
      await this.connection?.close();
      log.info('Disconnected from RabbitMQ');
    } catch (error) {
      log.error('Error disconnecting from RabbitMQ', error as Error);
    }
  }
  
  getChannel(): Channel {
    if (!this.channel) {
      throw new Error('RabbitMQ channel not available');
    }
    return this.channel;
  }
  
  getConfirmChannel(): ConfirmChannel {
    if (!this.confirmChannel) {
      throw new Error('RabbitMQ confirm channel not available');
    }
    return this.confirmChannel;
  }
  
  isConnected(): boolean {
    return this.connection !== null && !this.isShuttingDown;
  }
}

export const rabbitMQClient = new RabbitMQClient({
  url: process.env.RABBITMQ_URL!,
  prefetchCount: parseInt(process.env.RABBITMQ_PREFETCH || '10'),
});
```

---

## 2. Exchange Types พร้อม TypeScript Examples

### 2.1 Direct Exchange

```typescript
// src/messaging/exchanges/direct-exchange.ts
import { Channel } from 'amqplib';

const EXCHANGE_NAME = 'orders.direct';

export async function setupDirectExchange(channel: Channel) {
  await channel.assertExchange(EXCHANGE_NAME, 'direct', {
    durable: true,
    autoDelete: false,
  });
  
  // Queues with routing keys
  const queues = [
    { name: 'orders.new', routingKey: 'order.created' },
    { name: 'orders.paid', routingKey: 'order.paid' },
    { name: 'orders.shipped', routingKey: 'order.shipped' },
    { name: 'orders.cancelled', routingKey: 'order.cancelled' },
  ];
  
  for (const { name, routingKey } of queues) {
    await channel.assertQueue(name, {
      durable: true,
      arguments: {
        'x-dead-letter-exchange': 'orders.dlx',
        'x-dead-letter-routing-key': `dlq.${name}`,
        'x-message-ttl': 86400000, // 24 hours
      },
    });
    
    await channel.bindQueue(name, EXCHANGE_NAME, routingKey);
  }
}

export async function publishToDirectExchange(
  channel: Channel,
  routingKey: string,
  payload: unknown,
  options: { messageId?: string; correlationId?: string } = {}
): Promise<void> {
  const message = Buffer.from(JSON.stringify(payload));
  
  channel.publish(EXCHANGE_NAME, routingKey, message, {
    persistent: true,
    contentType: 'application/json',
    messageId: options.messageId || crypto.randomUUID(),
    correlationId: options.correlationId,
    timestamp: Math.floor(Date.now() / 1000),
    appId: process.env.SERVICE_NAME,
    headers: {
      'x-service': process.env.SERVICE_NAME,
      'x-version': process.env.SERVICE_VERSION,
    },
  });
}
```

### 2.2 Topic Exchange

```typescript
// src/messaging/exchanges/topic-exchange.ts
import { Channel } from 'amqplib';

// Topic exchange ใช้ routing key แบบ pattern matching
// * = exactly one word, # = zero or more words
// Example: "order.created.us" matches "order.*.us" and "order.#"

const EXCHANGE_NAME = 'events.topic';

export async function setupTopicExchange(channel: Channel) {
  await channel.assertExchange(EXCHANGE_NAME, 'topic', { durable: true });
  
  // Analytics service: wants ALL order events
  await channel.assertQueue('analytics.all-orders', { durable: true });
  await channel.bindQueue('analytics.all-orders', EXCHANGE_NAME, 'order.#');
  
  // US payment service: wants US payment events only
  await channel.assertQueue('payments.us', { durable: true });
  await channel.bindQueue('payments.us', EXCHANGE_NAME, 'payment.*.us');
  
  // Notifications: wants created and completed events for any region
  await channel.assertQueue('notifications.lifecycle', { durable: true });
  await channel.bindQueue('notifications.lifecycle', EXCHANGE_NAME, 'order.created.*');
  await channel.bindQueue('notifications.lifecycle', EXCHANGE_NAME, 'order.completed.*');
  
  // Audit: wants all events from all services
  await channel.assertQueue('audit.all', { durable: true });
  await channel.bindQueue('audit.all', EXCHANGE_NAME, '#');
}

export async function publishToTopicExchange(
  channel: Channel,
  routingKey: string, // e.g., "order.created.us", "payment.failed.eu"
  payload: unknown,
  options: Record<string, unknown> = {}
) {
  const buffer = Buffer.from(JSON.stringify({
    ...payload,
    _meta: {
      service: process.env.SERVICE_NAME,
      timestamp: new Date().toISOString(),
      routingKey,
    },
  }));
  
  channel.publish(EXCHANGE_NAME, routingKey, buffer, {
    persistent: true,
    contentType: 'application/json',
    messageId: crypto.randomUUID(),
    ...options,
  });
}

// Usage:
// publishToTopicExchange(channel, 'order.created.th', orderData);  // Thailand
// publishToTopicExchange(channel, 'order.created.us', orderData);  // USA
// publishToTopicExchange(channel, 'payment.failed.eu', paymentData); // Europe
```

### 2.3 Fanout Exchange

```typescript
// src/messaging/exchanges/fanout-exchange.ts
import { Channel } from 'amqplib';

// Fanout exchange ส่ง message ไปยังทุก queue ที่ bind อยู่โดยไม่สนใจ routing key
// ใช้สำหรับ broadcast events

const EXCHANGE_NAME = 'cache.invalidation';

export async function setupFanoutExchange(channel: Channel) {
  await channel.assertExchange(EXCHANGE_NAME, 'fanout', { durable: false });
  
  // Each service instance creates its own temporary queue
  const q = await channel.assertQueue('', {
    exclusive: true,  // Delete when connection closes
    autoDelete: true,
  });
  
  // Bind to fanout exchange (routing key ignored)
  await channel.bindQueue(q.queue, EXCHANGE_NAME, '');
  
  return q.queue;
}

export async function broadcastCacheInvalidation(
  channel: Channel,
  keys: string[]
): Promise<void> {
  const payload = Buffer.from(JSON.stringify({
    type: 'CACHE_INVALIDATION',
    keys,
    timestamp: new Date().toISOString(),
    sourceService: process.env.SERVICE_NAME,
  }));
  
  // Fanout: routing key is ignored
  channel.publish(EXCHANGE_NAME, '', payload, {
    persistent: false, // Fanout events don't need persistence
    contentType: 'application/json',
    expiration: '10000', // 10 second TTL
  });
}

// Consumer for cache invalidation
export async function consumeCacheInvalidations(
  channel: Channel,
  onInvalidate: (keys: string[]) => Promise<void>
) {
  const queueName = await setupFanoutExchange(channel);
  
  channel.consume(queueName, async (msg) => {
    if (!msg) return;
    
    const event = JSON.parse(msg.content.toString());
    
    // Don't process own messages
    if (event.sourceService === process.env.SERVICE_NAME) {
      channel.ack(msg);
      return;
    }
    
    try {
      await onInvalidate(event.keys);
      channel.ack(msg);
    } catch (error) {
      channel.nack(msg, false, false); // Don't requeue
    }
  }, { noAck: false });
}
```

### 2.4 Headers Exchange

```typescript
// src/messaging/exchanges/headers-exchange.ts
import { Channel } from 'amqplib';

// Headers exchange ใช้ message headers แทน routing key สำหรับ routing
// ยืดหยุ่นกว่า topic แต่ overhead สูงกว่า

const EXCHANGE_NAME = 'notifications.headers';

export async function setupHeadersExchange(channel: Channel) {
  await channel.assertExchange(EXCHANGE_NAME, 'headers', { durable: true });
  
  // Email notifications queue
  // x-match: all = ต้องตรงทุก header (AND)
  const emailQueue = await channel.assertQueue('notifications.email', { durable: true });
  await channel.bindQueue(emailQueue.queue, EXCHANGE_NAME, '', {
    'x-match': 'all',
    'channel': 'email',
    'priority': 'high',
  });
  
  // SMS notifications queue
  const smsQueue = await channel.assertQueue('notifications.sms', { durable: true });
  await channel.bindQueue(smsQueue.queue, EXCHANGE_NAME, '', {
    'x-match': 'any', // any = ตรงแค่ header เดียวก็พอ (OR)
    'channel': 'sms',
    'urgent': 'true',
  });
  
  // Push notification queue
  const pushQueue = await channel.assertQueue('notifications.push', { durable: true });
  await channel.bindQueue(pushQueue.queue, EXCHANGE_NAME, '', {
    'x-match': 'all',
    'channel': 'push',
  });
}

export async function publishNotification(
  channel: Channel,
  payload: {
    userId: string;
    title: string;
    body: string;
    channel: 'email' | 'sms' | 'push';
    priority: 'low' | 'medium' | 'high';
    urgent?: boolean;
  }
) {
  const buffer = Buffer.from(JSON.stringify(payload));
  
  channel.publish(EXCHANGE_NAME, '', buffer, {
    persistent: true,
    contentType: 'application/json',
    headers: {
      'channel': payload.channel,
      'priority': payload.priority,
      'urgent': payload.urgent ? 'true' : 'false',
      'user-id': payload.userId,
    },
  });
}
```

---

## 3. Dead Letter Queue (DLQ) Configuration

```typescript
// src/messaging/dead-letter-queue.ts
import { Channel, ConsumeMessage } from 'amqplib';

interface DLQConfig {
  exchangeName: string;
  queueName: string;
  maxRetries: number;
  retryDelay: number; // milliseconds
}

export async function setupDeadLetterInfrastructure(
  channel: Channel,
  mainExchange: string,
  mainQueue: string
) {
  // 1. Setup Dead Letter Exchange
  await channel.assertExchange('orders.dlx', 'direct', { durable: true });
  
  // 2. Setup Dead Letter Queue
  await channel.assertQueue('orders.dlq', {
    durable: true,
    arguments: {
      // Messages in DLQ expire after 7 days
      'x-message-ttl': 7 * 24 * 60 * 60 * 1000,
      // After 7 days, move to archive exchange
      'x-dead-letter-exchange': 'orders.archive',
    },
  });
  
  // 3. Bind DLQ to DLX
  await channel.bindQueue('orders.dlq', 'orders.dlx', `dlq.${mainQueue}`);
  
  // 4. Setup Retry Exchange (delayed retry pattern)
  // Using x-delayed-message plugin
  await channel.assertExchange('orders.retry', 'x-delayed-message', {
    durable: true,
    arguments: { 'x-delayed-type': 'direct' },
  });
  
  // 5. Setup Retry Queue
  await channel.assertQueue('orders.retry', {
    durable: true,
    arguments: {
      'x-dead-letter-exchange': mainExchange,
      'x-dead-letter-routing-key': 'order.retry',
    },
  });
  
  await channel.bindQueue('orders.retry', 'orders.retry', 'order.retry');
  
  // 6. Main queue with DLX configured
  await channel.assertQueue(mainQueue, {
    durable: true,
    arguments: {
      'x-dead-letter-exchange': 'orders.dlx',
      'x-dead-letter-routing-key': `dlq.${mainQueue}`,
      'x-message-ttl': 3600000, // 1 hour
    },
  });
}

// Retry mechanism with exponential backoff
export class RetryableConsumer {
  constructor(
    private readonly channel: Channel,
    private readonly maxRetries: number = 3,
    private readonly baseDelay: number = 1000
  ) {}
  
  async processWithRetry(
    msg: ConsumeMessage,
    handler: (payload: unknown) => Promise<void>
  ): Promise<void> {
    const retryCount = this.getRetryCount(msg);
    
    try {
      const payload = JSON.parse(msg.content.toString());
      await handler(payload);
      this.channel.ack(msg);
      
      if (retryCount > 0) {
        log.info('Message processed successfully after retry', { retryCount });
      }
    } catch (error) {
      if (retryCount < this.maxRetries) {
        await this.retryMessage(msg, retryCount, error as Error);
      } else {
        log.error('Message failed after max retries, sending to DLQ', error as Error, {
          retryCount,
          messageId: msg.properties.messageId,
        });
        
        // Reject without requeue → sends to DLX → DLQ
        this.channel.nack(msg, false, false);
      }
    }
  }
  
  private async retryMessage(
    msg: ConsumeMessage,
    currentRetry: number,
    error: Error
  ): Promise<void> {
    const delay = this.calculateDelay(currentRetry);
    
    log.warn('Retrying message', {
      messageId: msg.properties.messageId,
      retryCount: currentRetry + 1,
      delay,
      error: error.message,
    });
    
    // Publish to retry exchange with delay header
    this.channel.publish('orders.retry', 'order.retry', msg.content, {
      ...msg.properties,
      headers: {
        ...msg.properties.headers,
        'x-retry-count': currentRetry + 1,
        'x-delay': delay,
        'x-original-error': error.message,
        'x-first-failed-at': msg.properties.headers?.['x-first-failed-at'] || new Date().toISOString(),
      },
    });
    
    // Acknowledge original message
    this.channel.ack(msg);
  }
  
  private getRetryCount(msg: ConsumeMessage): number {
    const deathHeader = msg.properties.headers?.['x-retry-count'];
    return typeof deathHeader === 'number' ? deathHeader : 0;
  }
  
  private calculateDelay(retryCount: number): number {
    // Exponential backoff: 1s, 2s, 4s, 8s, ...
    return Math.min(
      this.baseDelay * Math.pow(2, retryCount),
      30000 // Max 30 second delay
    );
  }
}

// DLQ Monitor: alerts when messages pile up in DLQ
export class DLQMonitor {
  async checkDLQDepth(channel: Channel, queueName: string): Promise<number> {
    const queue = await channel.checkQueue(queueName);
    return queue.messageCount;
  }
  
  async reprocessDLQMessages(
    channel: Channel,
    dlqName: string,
    mainExchange: string,
    routingKey: string,
    limit: number = 100
  ): Promise<number> {
    let processed = 0;
    
    while (processed < limit) {
      const msg = await channel.get(dlqName, { noAck: false });
      
      if (!msg) break;
      
      // Reset retry count for manual reprocessing
      const cleanedHeaders = { ...msg.properties.headers };
      delete cleanedHeaders['x-retry-count'];
      delete cleanedHeaders['x-original-error'];
      
      channel.publish(mainExchange, routingKey, msg.content, {
        ...msg.properties,
        headers: {
          ...cleanedHeaders,
          'x-reprocessed-from-dlq': new Date().toISOString(),
        },
      });
      
      channel.ack(msg);
      processed++;
    }
    
    log.info(`Reprocessed ${processed} messages from DLQ ${dlqName}`);
    return processed;
  }
}
```

---

## 4. Message TTL Patterns

```typescript
// src/messaging/ttl-patterns.ts
import { Channel } from 'amqplib';

export async function setupTTLQueues(channel: Channel) {
  // Queue-level TTL: ทุก message หมดอายุใน 30 นาที
  await channel.assertQueue('session.events', {
    durable: false,
    arguments: {
      'x-message-ttl': 30 * 60 * 1000, // 30 minutes in ms
    },
  });
  
  // Per-message TTL (ใช้ expiration field ใน publish)
  await channel.assertQueue('flash.notifications', { durable: false });
  
  // TTL + DLX combo: ใช้ทำ delayed execution
  await channel.assertExchange('delayed.execution', 'direct', { durable: true });
  
  // Delay queue: messages stay here until TTL expires, then move to actual queue
  await channel.assertQueue('delayed.5min', {
    durable: true,
    arguments: {
      'x-message-ttl': 5 * 60 * 1000,     // 5 minute delay
      'x-dead-letter-exchange': 'orders.direct', // Go here after TTL
      'x-dead-letter-routing-key': 'order.retry',
    },
  });
  
  await channel.bindQueue('delayed.5min', 'delayed.execution', '5min');
}

// Send message with per-message TTL
export function publishWithTTL(
  channel: Channel,
  exchange: string,
  routingKey: string,
  payload: unknown,
  ttlMs: number
): void {
  channel.publish(
    exchange,
    routingKey,
    Buffer.from(JSON.stringify(payload)),
    {
      persistent: true,
      contentType: 'application/json',
      expiration: ttlMs.toString(), // Per-message TTL in milliseconds
      messageId: crypto.randomUUID(),
      timestamp: Math.floor(Date.now() / 1000),
    }
  );
}

// Schedule delayed message via TTL trick
export function scheduleMessage(
  channel: Channel,
  payload: unknown,
  delayMs: number,
  targetExchange: string,
  targetRoutingKey: string
): void {
  // Publish to delay queue with per-message TTL = desired delay
  channel.publish(
    'delayed.execution',
    '5min', // Route to 5min delay queue
    Buffer.from(JSON.stringify(payload)),
    {
      persistent: true,
      contentType: 'application/json',
      expiration: delayMs.toString(),
      headers: {
        'x-target-exchange': targetExchange,
        'x-target-routing-key': targetRoutingKey,
        'x-scheduled-at': new Date().toISOString(),
        'x-deliver-at': new Date(Date.now() + delayMs).toISOString(),
      },
    }
  );
}
```

---

## 5. Priority Queues

```typescript
// src/messaging/priority-queue.ts
import { Channel, ConsumeMessage } from 'amqplib';

// Priority queue: 0 (lowest) to 10 (highest)
export async function setupPriorityQueue(channel: Channel) {
  await channel.assertExchange('orders.priority', 'direct', { durable: true });
  
  await channel.assertQueue('orders.processing', {
    durable: true,
    arguments: {
      'x-max-priority': 10, // Enable priority queue with max priority 10
      'x-dead-letter-exchange': 'orders.dlx',
    },
  });
  
  await channel.bindQueue(
    'orders.processing',
    'orders.priority',
    'order.process'
  );
}

type OrderPriority = 'vip' | 'premium' | 'standard' | 'bulk';

function getPriorityValue(priority: OrderPriority): number {
  const priorityMap: Record<OrderPriority, number> = {
    vip: 10,      // VIP customers: highest priority
    premium: 7,   // Premium subscribers
    standard: 5,  // Regular customers
    bulk: 2,      // Bulk/wholesale orders
  };
  return priorityMap[priority];
}

export function publishPriorityOrder(
  channel: Channel,
  orderData: {
    orderId: string;
    userId: string;
    priority: OrderPriority;
    items: Array<{ productId: string; quantity: number }>;
  }
): void {
  channel.publish(
    'orders.priority',
    'order.process',
    Buffer.from(JSON.stringify(orderData)),
    {
      persistent: true,
      contentType: 'application/json',
      messageId: orderData.orderId,
      priority: getPriorityValue(orderData.priority),
      headers: {
        'x-order-priority': orderData.priority,
        'x-user-id': orderData.userId,
      },
    }
  );
}

// Consumer with priority awareness
export async function consumePriorityOrders(
  channel: Channel,
  handler: (order: any, priority: number) => Promise<void>
) {
  // Set low prefetch to ensure VIP orders aren't stuck behind bulk orders
  await channel.prefetch(1);
  
  channel.consume('orders.processing', async (msg: ConsumeMessage | null) => {
    if (!msg) return;
    
    const priority = msg.properties.priority || 0;
    const order = JSON.parse(msg.content.toString());
    
    log.info('Processing order', {
      orderId: order.orderId,
      priority,
      priorityLabel: order.priority,
    });
    
    try {
      await handler(order, priority);
      channel.ack(msg);
    } catch (error) {
      log.error('Failed to process priority order', error as Error, {
        orderId: order.orderId,
        priority,
      });
      channel.nack(msg, false, false); // Send to DLQ
    }
  });
}
```

---

## 6. Consumer Acknowledgment Patterns

```typescript
// src/messaging/acknowledgment-patterns.ts
import { Channel, ConsumeMessage } from 'amqplib';

// Pattern 1: Simple ack/nack
export async function simpleAckPattern(
  channel: Channel,
  queueName: string,
  handler: (payload: unknown) => Promise<void>
) {
  channel.consume(queueName, async (msg) => {
    if (!msg) return;
    
    try {
      await handler(JSON.parse(msg.content.toString()));
      channel.ack(msg); // Single message ack
    } catch (error) {
      // nack(msg, allUpTo, requeue)
      channel.nack(msg, false, true); // Requeue for retry
    }
  });
}

// Pattern 2: Batch acknowledgment
export class BatchConsumer {
  private pendingMessages: ConsumeMessage[] = [];
  private batchTimer: NodeJS.Timeout | null = null;
  
  constructor(
    private readonly channel: Channel,
    private readonly batchSize: number = 100,
    private readonly batchTimeoutMs: number = 1000
  ) {}
  
  async startConsuming(
    queueName: string,
    batchHandler: (messages: ConsumeMessage[]) => Promise<void>
  ) {
    this.channel.consume(queueName, async (msg) => {
      if (!msg) return;
      
      this.pendingMessages.push(msg);
      
      if (this.pendingMessages.length >= this.batchSize) {
        await this.processBatch(batchHandler);
      } else if (!this.batchTimer) {
        // Start timer to flush partial batch
        this.batchTimer = setTimeout(
          () => this.processBatch(batchHandler),
          this.batchTimeoutMs
        );
      }
    }, { noAck: false });
  }
  
  private async processBatch(
    handler: (messages: ConsumeMessage[]) => Promise<void>
  ) {
    if (this.batchTimer) {
      clearTimeout(this.batchTimer);
      this.batchTimer = null;
    }
    
    const batch = this.pendingMessages.splice(0, this.batchSize);
    if (batch.length === 0) return;
    
    try {
      await handler(batch);
      
      // Ack all in batch with allUpTo=true (more efficient)
      this.channel.ack(batch[batch.length - 1], true);
    } catch (error) {
      log.error('Batch processing failed', error as Error, { batchSize: batch.length });
      
      // Nack all messages individually for DLQ routing
      for (const msg of batch) {
        this.channel.nack(msg, false, false);
      }
    }
  }
}

// Pattern 3: Manual ack with transaction support
export async function transactionalConsumer(
  channel: Channel,
  queueName: string,
  handler: (payload: unknown) => Promise<void>
) {
  channel.consume(queueName, async (msg) => {
    if (!msg) return;
    
    const payload = JSON.parse(msg.content.toString());
    
    // Process in database transaction
    const trx = await db.transaction();
    
    try {
      await handler(payload);
      
      // Commit DB transaction first
      await trx.commit();
      
      // Then ack message
      channel.ack(msg);
    } catch (error) {
      await trx.rollback();
      
      // Don't requeue if it's a business logic error
      const shouldRequeue = error instanceof TransientError;
      channel.nack(msg, false, shouldRequeue);
    }
  }, { noAck: false });
}

class TransientError extends Error {}
```

---

## 7. Publisher Confirms

```typescript
// src/messaging/publisher-confirms.ts
import { ConfirmChannel } from 'amqplib';

export class ReliablePublisher {
  private readonly pendingConfirms = new Map<number, {
    resolve: () => void;
    reject: (err: Error) => void;
    timestamp: number;
  }>();
  
  constructor(private readonly channel: ConfirmChannel) {
    // Handle ack/nack from broker
    this.channel.on('ack', (seqNo: number, multiple: boolean) => {
      this.handleConfirm(seqNo, multiple, true);
    });
    
    this.channel.on('nack', (seqNo: number, multiple: boolean) => {
      this.handleConfirm(seqNo, multiple, false);
    });
  }
  
  async publish(
    exchange: string,
    routingKey: string,
    payload: unknown,
    options: Record<string, unknown> = {}
  ): Promise<void> {
    return new Promise((resolve, reject) => {
      const seqNo = this.channel.publish(
        exchange,
        routingKey,
        Buffer.from(JSON.stringify(payload)),
        {
          persistent: true,
          contentType: 'application/json',
          messageId: crypto.randomUUID(),
          timestamp: Math.floor(Date.now() / 1000),
          ...options,
        }
      );
      
      if (seqNo === false) {
        reject(new Error('Channel write buffer is full'));
        return;
      }
      
      // The channel.publish returns void for confirm channels when seqNo is tracked
      // Track pending confirm
      const actualSeqNo = (this.channel as any)._seqNum || 0;
      
      this.pendingConfirms.set(actualSeqNo, {
        resolve,
        reject,
        timestamp: Date.now(),
      });
      
      // Timeout after 30 seconds
      setTimeout(() => {
        const pending = this.pendingConfirms.get(actualSeqNo);
        if (pending) {
          this.pendingConfirms.delete(actualSeqNo);
          pending.reject(new Error('Publisher confirm timeout'));
        }
      }, 30000);
    });
  }
  
  // Batch publish with single waitForConfirms
  async publishBatch(
    messages: Array<{
      exchange: string;
      routingKey: string;
      payload: unknown;
    }>
  ): Promise<void> {
    for (const msg of messages) {
      this.channel.publish(
        msg.exchange,
        msg.routingKey,
        Buffer.from(JSON.stringify(msg.payload)),
        {
          persistent: true,
          contentType: 'application/json',
        }
      );
    }
    
    // Wait for broker to confirm all messages in batch
    await this.channel.waitForConfirms();
  }
  
  private handleConfirm(seqNo: number, multiple: boolean, isAck: boolean) {
    if (multiple) {
      // Confirm all messages up to seqNo
      for (const [seq, pending] of this.pendingConfirms) {
        if (seq <= seqNo) {
          this.pendingConfirms.delete(seq);
          if (isAck) {
            pending.resolve();
          } else {
            pending.reject(new Error('Message was nacked by broker'));
          }
        }
      }
    } else {
      const pending = this.pendingConfirms.get(seqNo);
      if (pending) {
        this.pendingConfirms.delete(seqNo);
        if (isAck) {
          pending.resolve();
        } else {
          pending.reject(new Error('Message was nacked by broker'));
        }
      }
    }
  }
}
```

---

## 8. Consumer Prefetch Count Tuning

```typescript
// src/messaging/prefetch-tuning.ts
import { Channel } from 'amqplib';

interface PrefetchConfig {
  global: number;     // Channel-level prefetch
  consumer: number;   // Per-consumer prefetch
}

function calculateOptimalPrefetch(
  avgProcessingTimeMs: number,
  targetThroughputPerSec: number
): number {
  // Little's Law: N = λ × W
  // N = prefetch count, λ = throughput, W = processing time
  const processingTimeSec = avgProcessingTimeMs / 1000;
  const optimal = Math.ceil(targetThroughputPerSec * processingTimeSec);
  
  // Add 20% buffer
  return Math.ceil(optimal * 1.2);
}

export async function setupOptimizedConsumer(
  channel: Channel,
  config: {
    queueName: string;
    avgProcessingTimeMs: number;
    targetThroughputPerSec: number;
    handler: (msg: any) => Promise<void>;
  }
) {
  const prefetch = calculateOptimalPrefetch(
    config.avgProcessingTimeMs,
    config.targetThroughputPerSec
  );
  
  log.info('Setting prefetch count', {
    queue: config.queueName,
    prefetch,
    avgProcessingTimeMs: config.avgProcessingTimeMs,
    targetThroughputPerSec: config.targetThroughputPerSec,
  });
  
  // global=false: per-consumer prefetch (recommended)
  await channel.prefetch(prefetch, false);
  
  channel.consume(config.queueName, async (msg) => {
    if (!msg) return;
    
    const start = Date.now();
    
    try {
      await config.handler(JSON.parse(msg.content.toString()));
      channel.ack(msg);
      
      const duration = Date.now() - start;
      
      // Record for metrics
      messageProcessingDuration.labels(
        config.queueName,
        msg.properties.type || 'unknown',
        'true'
      ).observe(duration / 1000);
    } catch (error) {
      channel.nack(msg, false, false);
      
      messageProcessingErrors.labels(
        config.queueName,
        msg.properties.type || 'unknown',
        (error as Error).constructor.name
      ).inc();
    }
  });
}

// Different prefetch for different message types
export async function setupMultiConsumer(channel: Channel) {
  // CPU-intensive tasks: low prefetch (prevent overwhelming workers)
  await channel.prefetch(2, false);
  
  channel.consume('orders.image-processing', async (msg) => {
    if (!msg) return;
    // Intensive processing...
    channel.ack(msg!);
  });
  
  // Fast I/O tasks: higher prefetch
  await channel.prefetch(20, false);
  
  channel.consume('orders.notifications', async (msg) => {
    if (!msg) return;
    // Quick operations...
    channel.ack(msg!);
  });
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้ RabbitMQ Message Patterns ขั้นสูงครอบคลุม:

1. **RabbitMQ Client** — Resilient connection management พร้อม exponential backoff reconnect, separate channels สำหรับ consume/publish/confirm

2. **Exchange Types**:
   - **Direct**: Exact routing key matching สำหรับ point-to-point
   - **Topic**: Wildcard pattern matching (*, #) สำหรับ content-based routing
   - **Fanout**: Broadcast ไปทุก bound queues สำหรับ cache invalidation
   - **Headers**: Header-based routing สำหรับ complex routing logic

3. **Dead Letter Queues** — DLX + DLQ infrastructure, retry with exponential backoff, DLQ monitoring และ manual reprocessing

4. **Message TTL** — Queue-level TTL, per-message TTL, delayed execution pattern ด้วย TTL trick

5. **Priority Queues** — x-max-priority configuration, priority values by customer tier, low prefetch สำหรับ fair processing

6. **Consumer Acknowledgment** — Simple ack/nack, batch acknowledgment, transactional consumer

7. **Publisher Confirms** — ConfirmChannel พร้อม per-message confirm tracking, batch publish, timeout handling

8. **Prefetch Tuning** — Little's Law สำหรับคำนวณ optimal prefetch, per-consumer vs global prefetch

Key takeaways:
- ใช้ ConfirmChannel สำหรับทุก critical message เพื่อ guarantee at-least-once delivery
- DLQ ต้องมี monitoring และ alerting เสมอ เพื่อรู้เมื่อมี messages stuck
- Prefetch count มีผลต่อ throughput มาก ควร tune ตาม processing time และ target throughput
- Topic exchange เป็น choice ที่ดีสำหรับ event-driven architectures ที่ต้องการ flexibility

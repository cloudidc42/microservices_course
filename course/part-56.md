# Part 56: RabbitMQ and Message Broker Patterns

## บทนำ

RabbitMQ เป็น Message Broker ที่ทรงพลังและยืดหยุ่นสูง ใช้ AMQP protocol
ในบทนี้จะเรียนรู้ Pattern ขั้นสูงที่ใช้จริงใน Production รวมถึงการจัดการ Error,
Dead Letter Queue, Priority Queue และ Monitoring

---

## 1. RabbitMQ Concepts พื้นฐาน

### Exchange Types

```
Producer → Exchange → Queue → Consumer

Exchange Types:
- direct:  routing key ต้องตรงกับ binding key
- fanout:  broadcast ไปทุก queue
- topic:   routing key แบบ wildcard (*, #)
- headers: ใช้ message headers แทน routing key
```

### การติดตั้ง RabbitMQ ด้วย Docker

```yaml
# docker-compose.rabbitmq.yml
version: '3.8'

services:
  rabbitmq:
    image: rabbitmq:3.12-management
    hostname: rabbitmq
    environment:
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: adminpassword
      RABBITMQ_DEFAULT_VHOST: /
    ports:
      - "5672:5672"    # AMQP port
      - "15672:15672"  # Management UI
      - "15692:15692"  # Prometheus metrics
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
      - ./rabbitmq.conf:/etc/rabbitmq/rabbitmq.conf
      - ./rabbitmq-definitions.json:/etc/rabbitmq/definitions.json
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "-q", "ping"]
      interval: 30s
      timeout: 10s
      retries: 5

  rabbitmq-exporter:
    image: kbudde/rabbitmq-exporter:latest
    environment:
      RABBIT_URL: http://rabbitmq:15672
      RABBIT_USER: admin
      RABBIT_PASSWORD: adminpassword
    ports:
      - "9419:9419"
    depends_on:
      - rabbitmq

volumes:
  rabbitmq_data:
```

```ini
# rabbitmq.conf
# การตั้งค่า RabbitMQ สำหรับ Production
default_vhost = /
default_user = admin
default_pass = adminpassword

# Memory threshold
vm_memory_high_watermark.relative = 0.6
vm_memory_high_watermark_paging_ratio = 0.5

# Disk free space
disk_free_limit.relative = 1.0

# Logging
log.console = true
log.console.level = info
log.file = /var/log/rabbitmq/rabbit.log
log.file.level = info

# Management plugin
management.tcp.port = 15672
management.tcp.ip = 0.0.0.0

# Prometheus metrics
prometheus.tcp.port = 15692
```

---

## 2. TypeScript AMQP Wrapper Library

### การสร้าง RabbitMQ Client

```typescript
// src/messaging/rabbitmq.client.ts
import * as amqplib from 'amqplib';
import { EventEmitter } from 'events';
import { Logger } from '@nestjs/common';

export interface RabbitMQConfig {
  url: string;
  reconnectDelay?: number;
  maxReconnectAttempts?: number;
  heartbeat?: number;
  prefetch?: number;
}

export interface PublishOptions {
  exchange: string;
  routingKey: string;
  content: unknown;
  options?: amqplib.Options.Publish;
}

export interface ConsumeOptions {
  queue: string;
  handler: (message: amqplib.ConsumeMessage) => Promise<void>;
  options?: amqplib.Options.Consume;
}

export class RabbitMQClient extends EventEmitter {
  private readonly logger = new Logger(RabbitMQClient.name);
  private connection?: amqplib.Connection;
  private channel?: amqplib.Channel;
  private confirmChannel?: amqplib.ConfirmChannel;
  private isConnected = false;
  private reconnectAttempts = 0;

  constructor(private readonly config: RabbitMQConfig) {
    super();
  }

  async connect(): Promise<void> {
    try {
      this.connection = await amqplib.connect(this.config.url, {
        heartbeat: this.config.heartbeat || 60,
      });
      
      this.connection.on('error', (err) => {
        this.logger.error('RabbitMQ connection error:', err);
        this.handleDisconnect();
      });
      
      this.connection.on('close', () => {
        this.logger.warn('RabbitMQ connection closed');
        this.handleDisconnect();
      });
      
      // สร้าง channel ปกติ
      this.channel = await this.connection.createChannel();
      await this.channel.prefetch(this.config.prefetch || 10);
      
      // สร้าง confirm channel สำหรับ publisher confirms
      this.confirmChannel = await this.connection.createConfirmChannel();
      
      this.isConnected = true;
      this.reconnectAttempts = 0;
      this.emit('connected');
      
      this.logger.log('RabbitMQ connected successfully');
    } catch (error) {
      this.logger.error('Failed to connect to RabbitMQ:', error);
      await this.scheduleReconnect();
    }
  }

  private async handleDisconnect(): Promise<void> {
    this.isConnected = false;
    this.channel = undefined;
    this.confirmChannel = undefined;
    this.connection = undefined;
    
    this.emit('disconnected');
    await this.scheduleReconnect();
  }

  private async scheduleReconnect(): Promise<void> {
    const maxAttempts = this.config.maxReconnectAttempts || Infinity;
    
    if (this.reconnectAttempts >= maxAttempts) {
      this.logger.error(`Max reconnect attempts (${maxAttempts}) reached`);
      this.emit('max_reconnects_reached');
      return;
    }
    
    this.reconnectAttempts++;
    const delay = Math.min(
      (this.config.reconnectDelay || 5000) * Math.pow(2, this.reconnectAttempts - 1),
      30000  // Maximum 30 seconds
    );
    
    this.logger.log(`Reconnecting in ${delay}ms (attempt ${this.reconnectAttempts})...`);
    
    await new Promise(resolve => setTimeout(resolve, delay));
    await this.connect();
  }

  async assertExchange(
    name: string,
    type: 'direct' | 'fanout' | 'topic' | 'headers',
    options?: amqplib.Options.AssertExchange
  ): Promise<void> {
    if (!this.channel) throw new Error('Channel not available');
    
    await this.channel.assertExchange(name, type, {
      durable: true,
      ...options,
    });
  }

  async assertQueue(
    name: string,
    options?: amqplib.Options.AssertQueue
  ): Promise<amqplib.Replies.AssertQueue> {
    if (!this.channel) throw new Error('Channel not available');
    
    return this.channel.assertQueue(name, {
      durable: true,
      ...options,
    });
  }

  async bindQueue(
    queue: string,
    exchange: string,
    routingKey: string,
    args?: Record<string, unknown>
  ): Promise<void> {
    if (!this.channel) throw new Error('Channel not available');
    await this.channel.bindQueue(queue, exchange, routingKey, args);
  }

  async publish(opts: PublishOptions): Promise<boolean> {
    if (!this.confirmChannel) throw new Error('Confirm channel not available');
    
    const content = Buffer.from(JSON.stringify(opts.content));
    
    return new Promise((resolve, reject) => {
      this.confirmChannel!.publish(
        opts.exchange,
        opts.routingKey,
        content,
        {
          persistent: true,
          contentType: 'application/json',
          timestamp: Date.now(),
          ...opts.options,
        },
        (err) => {
          if (err) {
            reject(err);
          } else {
            resolve(true);
          }
        }
      );
    });
  }

  async consume(opts: ConsumeOptions): Promise<void> {
    if (!this.channel) throw new Error('Channel not available');
    
    await this.channel.consume(
      opts.queue,
      async (msg) => {
        if (!msg) return;
        
        try {
          await opts.handler(msg);
          this.channel?.ack(msg);
        } catch (error) {
          this.logger.error(`Error processing message from ${opts.queue}:`, error);
          // Nack และส่งไป DLQ
          this.channel?.nack(msg, false, false);
        }
      },
      opts.options
    );
  }

  async close(): Promise<void> {
    try {
      await this.channel?.close();
      await this.confirmChannel?.close();
      await this.connection?.close();
    } catch (error) {
      this.logger.error('Error closing RabbitMQ connection:', error);
    }
  }

  getChannel(): amqplib.Channel {
    if (!this.channel) throw new Error('Channel not available');
    return this.channel;
  }

  isReady(): boolean {
    return this.isConnected && !!this.channel;
  }
}
```

---

## 3. Exchange Patterns

### Direct Exchange

```typescript
// src/messaging/patterns/direct-exchange.ts
import { RabbitMQClient } from '../rabbitmq.client';

export class DirectExchangeExample {
  constructor(private readonly client: RabbitMQClient) {}

  async setup(): Promise<void> {
    // สร้าง exchange
    await this.client.assertExchange('orders', 'direct', { durable: true });
    
    // สร้าง queues สำหรับแต่ละ routing key
    await this.client.assertQueue('orders.created', { durable: true });
    await this.client.assertQueue('orders.updated', { durable: true });
    await this.client.assertQueue('orders.cancelled', { durable: true });
    
    // Bind queues กับ exchange
    await this.client.bindQueue('orders.created', 'orders', 'order.created');
    await this.client.bindQueue('orders.updated', 'orders', 'order.updated');
    await this.client.bindQueue('orders.cancelled', 'orders', 'order.cancelled');
  }

  async publishOrderCreated(order: { id: string; total: number; customerId: string }): Promise<void> {
    await this.client.publish({
      exchange: 'orders',
      routingKey: 'order.created',
      content: {
        eventType: 'order.created',
        payload: order,
        timestamp: new Date().toISOString(),
        correlationId: crypto.randomUUID(),
      },
    });
  }
}
```

### Fanout Exchange

```typescript
// src/messaging/patterns/fanout-exchange.ts
export class FanoutExchangeExample {
  constructor(private readonly client: RabbitMQClient) {}

  async setup(): Promise<void> {
    // Fanout exchange - broadcast ไปทุก queue
    await this.client.assertExchange('notifications', 'fanout', { durable: true });
    
    // สร้าง queues สำหรับ subscribers ต่างๆ
    const subscribers = ['email-service', 'sms-service', 'push-notification-service', 'audit-log-service'];
    
    for (const subscriber of subscribers) {
      await this.client.assertQueue(`notifications.${subscriber}`, { durable: true });
      // Fanout ไม่ต้องการ routing key (ใช้ '' แทน)
      await this.client.bindQueue(`notifications.${subscriber}`, 'notifications', '');
    }
  }

  async broadcastNotification(notification: {
    type: string;
    message: string;
    userId: string;
  }): Promise<void> {
    await this.client.publish({
      exchange: 'notifications',
      routingKey: '',  // Fanout ไม่ใช้ routing key
      content: notification,
    });
  }
}
```

### Topic Exchange

```typescript
// src/messaging/patterns/topic-exchange.ts
export class TopicExchangeExample {
  constructor(private readonly client: RabbitMQClient) {}

  async setup(): Promise<void> {
    await this.client.assertExchange('events', 'topic', { durable: true });
    
    // Queue สำหรับ order events ทั้งหมด
    await this.client.assertQueue('events.orders.all', { durable: true });
    await this.client.bindQueue('events.orders.all', 'events', 'order.#');
    
    // Queue สำหรับ payment events
    await this.client.assertQueue('events.payment', { durable: true });
    await this.client.bindQueue('events.payment', 'events', 'payment.*');
    
    // Queue สำหรับ critical events ทั้งหมด
    await this.client.assertQueue('events.critical', { durable: true });
    await this.client.bindQueue('events.critical', 'events', '*.critical.*');
    
    // Queue สำหรับทุก event ของ user-service
    await this.client.assertQueue('events.user-service', { durable: true });
    await this.client.bindQueue('events.user-service', 'events', 'user-service.#');
  }

  async publishEvent(domain: string, action: string, data: unknown): Promise<void> {
    const routingKey = `${domain}.${action}`;
    
    await this.client.publish({
      exchange: 'events',
      routingKey,
      content: {
        domain,
        action,
        data,
        timestamp: new Date().toISOString(),
      },
    });
  }
}
```

### Headers Exchange

```typescript
// src/messaging/patterns/headers-exchange.ts
export class HeadersExchangeExample {
  constructor(private readonly client: RabbitMQClient) {}

  async setup(): Promise<void> {
    await this.client.assertExchange('reports', 'headers', { durable: true });
    
    // Queue สำหรับ PDF reports ภาษาไทย
    await this.client.assertQueue('reports.pdf.th', { durable: true });
    await this.client.bindQueue('reports.pdf.th', 'reports', '', {
      'x-match': 'all',  // ต้องตรงทั้งหมด
      format: 'pdf',
      language: 'th',
    });
    
    // Queue สำหรับ Excel reports ทุกภาษา
    await this.client.assertQueue('reports.excel.all', { durable: true });
    await this.client.bindQueue('reports.excel.all', 'reports', '', {
      'x-match': 'any',  // ตรงอย่างน้อย 1
      format: 'xlsx',
    });
  }

  async requestReport(format: string, language: string, data: unknown): Promise<void> {
    await this.client.publish({
      exchange: 'reports',
      routingKey: '',  // Headers exchange ไม่ใช้ routing key
      content: data,
      options: {
        headers: {
          format,
          language,
          requestedAt: new Date().toISOString(),
        },
      },
    });
  }
}
```

---

## 4. Dead Letter Queue Pattern

### การตั้งค่า DLQ

```typescript
// src/messaging/patterns/dead-letter-queue.ts
import { RabbitMQClient } from '../rabbitmq.client';
import * as amqplib from 'amqplib';

export interface RetryConfig {
  maxRetries: number;
  retryDelays: number[];  // Milliseconds สำหรับแต่ละ retry
}

export class DeadLetterQueuePattern {
  private readonly logger = console;

  constructor(
    private readonly client: RabbitMQClient,
    private readonly retryConfig: RetryConfig = {
      maxRetries: 3,
      retryDelays: [1000, 5000, 30000],  // 1s, 5s, 30s
    }
  ) {}

  async setup(serviceName: string): Promise<void> {
    const mainExchange = `${serviceName}.exchange`;
    const dlxExchange = `${serviceName}.dlx`;
    const mainQueue = `${serviceName}.queue`;
    const dlqQueue = `${serviceName}.dlq`;
    const retryQueue = `${serviceName}.retry`;

    // 1. สร้าง Dead Letter Exchange
    await this.client.assertExchange(dlxExchange, 'direct', { durable: true });
    
    // 2. สร้าง Dead Letter Queue
    await this.client.assertQueue(dlqQueue, {
      durable: true,
      arguments: {
        'x-queue-type': 'quorum',
      },
    });
    await this.client.bindQueue(dlqQueue, dlxExchange, 'dead-letter');

    // 3. สร้าง Retry Queue พร้อม delay
    for (let i = 0; i < this.retryConfig.maxRetries; i++) {
      const retryQueueName = `${retryQueue}.${i + 1}`;
      const delayMs = this.retryConfig.retryDelays[i];
      
      await this.client.assertQueue(retryQueueName, {
        durable: true,
        arguments: {
          'x-message-ttl': delayMs,
          'x-dead-letter-exchange': mainExchange,
          'x-dead-letter-routing-key': 'main',
        },
      });
    }

    // 4. สร้าง Main Exchange และ Queue
    await this.client.assertExchange(mainExchange, 'direct', { durable: true });
    await this.client.assertQueue(mainQueue, {
      durable: true,
      arguments: {
        'x-dead-letter-exchange': dlxExchange,
        'x-dead-letter-routing-key': 'dead-letter',
      },
    });
    await this.client.bindQueue(mainQueue, mainExchange, 'main');
  }

  async consume(
    serviceName: string,
    handler: (content: unknown) => Promise<void>
  ): Promise<void> {
    const mainQueue = `${serviceName}.queue`;
    const retryQueue = `${serviceName}.retry`;
    const dlxExchange = `${serviceName}.dlx`;
    const mainExchange = `${serviceName}.exchange`;

    await this.client.consume({
      queue: mainQueue,
      handler: async (msg: amqplib.ConsumeMessage) => {
        const channel = this.client.getChannel();
        
        try {
          const content = JSON.parse(msg.content.toString());
          await handler(content);
          channel.ack(msg);
        } catch (error) {
          const retryCount = (msg.properties.headers?.['x-retry-count'] || 0) as number;
          
          if (retryCount < this.retryConfig.maxRetries) {
            const nextRetryQueue = `${retryQueue}.${retryCount + 1}`;
            
            this.logger.warn(`Retry ${retryCount + 1}/${this.retryConfig.maxRetries} for message`);
            
            // ส่งไป retry queue
            channel.sendToQueue(
              nextRetryQueue,
              msg.content,
              {
                persistent: true,
                headers: {
                  ...msg.properties.headers,
                  'x-retry-count': retryCount + 1,
                  'x-original-error': error instanceof Error ? error.message : 'Unknown error',
                  'x-failed-at': new Date().toISOString(),
                },
              }
            );
            
            channel.ack(msg);
          } else {
            this.logger.error(`Message failed after ${this.retryConfig.maxRetries} retries, sending to DLQ`);
            channel.nack(msg, false, false);  // ส่งไป DLQ
          }
        }
      },
    });
  }
}
```

---

## 5. Priority Queue

```typescript
// src/messaging/patterns/priority-queue.ts
export enum MessagePriority {
  CRITICAL = 10,
  HIGH = 8,
  NORMAL = 5,
  LOW = 2,
  BACKGROUND = 0,
}

export class PriorityQueuePattern {
  constructor(private readonly client: RabbitMQClient) {}

  async setup(queueName: string, maxPriority = 10): Promise<void> {
    await this.client.assertQueue(queueName, {
      durable: true,
      arguments: {
        'x-max-priority': maxPriority,
      },
    });
  }

  async publishWithPriority(
    queueName: string,
    content: unknown,
    priority: MessagePriority
  ): Promise<void> {
    const channel = this.client.getChannel();
    
    channel.sendToQueue(
      queueName,
      Buffer.from(JSON.stringify(content)),
      {
        persistent: true,
        priority,
        timestamp: Date.now(),
      }
    );
  }

  async publishOrderWithPriority(order: {
    id: string;
    type: 'express' | 'standard' | 'economy';
    amount: number;
  }): Promise<void> {
    let priority: MessagePriority;
    
    if (order.type === 'express' || order.amount > 10000) {
      priority = MessagePriority.HIGH;
    } else if (order.type === 'standard') {
      priority = MessagePriority.NORMAL;
    } else {
      priority = MessagePriority.LOW;
    }
    
    await this.publishWithPriority('orders.processing', order, priority);
  }
}
```

---

## 6. RPC over RabbitMQ

```typescript
// src/messaging/patterns/rpc.ts
import * as amqplib from 'amqplib';
import { v4 as uuidv4 } from 'uuid';

export class RpcPattern {
  private pendingRequests = new Map<string, {
    resolve: (value: unknown) => void;
    reject: (error: Error) => void;
    timeout: NodeJS.Timeout;
  }>();

  private replyQueue?: string;

  constructor(
    private readonly client: RabbitMQClient,
    private readonly timeoutMs = 30000
  ) {}

  async setupClient(): Promise<void> {
    // สร้าง exclusive queue สำหรับ replies
    const q = await this.client.assertQueue('', {
      exclusive: true,
      autoDelete: true,
    });
    
    this.replyQueue = q.queue;
    
    // เริ่ม consume replies
    const channel = this.client.getChannel();
    await channel.consume(
      this.replyQueue,
      (msg) => {
        if (!msg) return;
        
        const correlationId = msg.properties.correlationId;
        const pending = this.pendingRequests.get(correlationId);
        
        if (pending) {
          clearTimeout(pending.timeout);
          this.pendingRequests.delete(correlationId);
          
          try {
            const response = JSON.parse(msg.content.toString());
            if (response.error) {
              pending.reject(new Error(response.error));
            } else {
              pending.resolve(response.data);
            }
          } catch (e) {
            pending.reject(new Error('Failed to parse RPC response'));
          }
          
          channel.ack(msg);
        }
      },
      { noAck: false }
    );
  }

  async call<T>(
    queue: string,
    payload: unknown,
    timeoutMs?: number
  ): Promise<T> {
    if (!this.replyQueue) {
      throw new Error('RPC client not initialized. Call setupClient() first.');
    }
    
    const correlationId = uuidv4();
    const timeout = timeoutMs || this.timeoutMs;
    
    return new Promise((resolve, reject) => {
      const timeoutHandle = setTimeout(() => {
        this.pendingRequests.delete(correlationId);
        reject(new Error(`RPC call to ${queue} timed out after ${timeout}ms`));
      }, timeout);
      
      this.pendingRequests.set(correlationId, {
        resolve: resolve as (value: unknown) => void,
        reject,
        timeout: timeoutHandle,
      });
      
      const channel = this.client.getChannel();
      channel.sendToQueue(
        queue,
        Buffer.from(JSON.stringify(payload)),
        {
          correlationId,
          replyTo: this.replyQueue!,
          persistent: false,  // RPC ไม่ต้องการ persistence
        }
      );
    });
  }

  async setupServer(
    queue: string,
    handler: (payload: unknown) => Promise<unknown>
  ): Promise<void> {
    await this.client.assertQueue(queue, { durable: true });
    
    const channel = this.client.getChannel();
    await channel.prefetch(1);  // Process one request at a time
    
    await channel.consume(queue, async (msg) => {
      if (!msg) return;
      
      try {
        const payload = JSON.parse(msg.content.toString());
        const result = await handler(payload);
        
        if (msg.properties.replyTo && msg.properties.correlationId) {
          channel.sendToQueue(
            msg.properties.replyTo,
            Buffer.from(JSON.stringify({ data: result })),
            {
              correlationId: msg.properties.correlationId,
            }
          );
        }
        
        channel.ack(msg);
      } catch (error) {
        if (msg.properties.replyTo && msg.properties.correlationId) {
          channel.sendToQueue(
            msg.properties.replyTo,
            Buffer.from(JSON.stringify({ 
              error: error instanceof Error ? error.message : 'Unknown error' 
            })),
            { correlationId: msg.properties.correlationId }
          );
        }
        
        channel.ack(msg);  // Ack แม้จะ error เพื่อไม่ให้ retry
      }
    });
  }
}

// Example การใช้งาน RPC
async function rpcExample() {
  const client = new RabbitMQClient({ url: 'amqp://localhost' });
  await client.connect();
  
  const rpc = new RpcPattern(client);
  
  // Server side
  await rpc.setupServer('user.getById', async (payload: unknown) => {
    const { userId } = payload as { userId: string };
    // ดึงข้อมูล user จาก database
    return { id: userId, name: 'John Doe', email: 'john@example.com' };
  });
  
  // Client side
  await rpc.setupClient();
  
  const user = await rpc.call<{ id: string; name: string; email: string }>(
    'user.getById',
    { userId: '123' },
    5000  // timeout 5 seconds
  );
  
  console.log('User:', user);
}
```

---

## 7. Publisher Confirms และ Consumer Acknowledgements

```typescript
// src/messaging/reliable-publisher.ts
export class ReliablePublisher {
  private unconfirmedMessages = new Map<number, {
    resolve: () => void;
    reject: (error: Error) => void;
    timeout: NodeJS.Timeout;
  }>();
  
  private deliveryTag = 0;

  constructor(
    private readonly client: RabbitMQClient,
    private readonly confirmTimeoutMs = 5000
  ) {}

  async publishWithConfirm(opts: {
    exchange: string;
    routingKey: string;
    content: unknown;
  }): Promise<void> {
    return this.client.publish({
      exchange: opts.exchange,
      routingKey: opts.routingKey,
      content: opts.content,
      options: {
        persistent: true,
        mandatory: true,  // Return ถ้า route ไม่ได้
      },
    }).then(() => void 0);
  }

  async publishBatch(messages: Array<{
    exchange: string;
    routingKey: string;
    content: unknown;
  }>): Promise<void> {
    // ส่ง messages พร้อมกัน
    const promises = messages.map(msg => this.publishWithConfirm(msg));
    await Promise.all(promises);
  }
}
```

---

## 8. Competing Consumers Pattern

```typescript
// src/messaging/patterns/competing-consumers.ts
export class CompetingConsumersPattern {
  constructor(
    private readonly client: RabbitMQClient,
    private readonly consumerCount: number = 5
  ) {}

  async setup(queueName: string): Promise<void> {
    // สร้าง queue เพียงอันเดียว
    await this.client.assertQueue(queueName, { durable: true });
    
    // ตั้งค่า prefetch เพื่อกระจาย load อย่างเป็นธรรม
    const channel = this.client.getChannel();
    await channel.prefetch(1);
  }

  async startConsumers(
    queueName: string,
    handler: (message: unknown) => Promise<void>
  ): Promise<void> {
    const consumers: Promise<void>[] = [];
    
    for (let i = 0; i < this.consumerCount; i++) {
      consumers.push(
        this.client.consume({
          queue: queueName,
          handler: async (msg) => {
            const content = JSON.parse(msg.content.toString());
            console.log(`Consumer ${i + 1} processing message`);
            await handler(content);
          },
        })
      );
    }
    
    await Promise.all(consumers);
  }
}
```

---

## 9. Message TTL และ Queue Expiry

```typescript
// src/messaging/patterns/ttl-expiry.ts
export class TTLAndExpiryPattern {
  constructor(private readonly client: RabbitMQClient) {}

  async createTemporaryQueue(
    name: string,
    ttlSeconds: number
  ): Promise<void> {
    await this.client.assertQueue(name, {
      durable: false,
      arguments: {
        // Queue หมดอายุหลังจากไม่มีคนใช้นาน 30 วินาที
        'x-expires': ttlSeconds * 1000,
        // Message หมดอายุหลัง 5 นาที
        'x-message-ttl': 300000,
      },
    });
  }

  async publishWithTTL(
    exchange: string,
    routingKey: string,
    content: unknown,
    ttlMs: number
  ): Promise<void> {
    await this.client.publish({
      exchange,
      routingKey,
      content,
      options: {
        expiration: ttlMs.toString(),  // Per-message TTL
      },
    });
  }

  async createSessionQueue(sessionId: string): Promise<void> {
    const queueName = `session.${sessionId}`;
    
    await this.client.assertQueue(queueName, {
      durable: false,
      exclusive: false,
      autoDelete: false,
      arguments: {
        'x-expires': 3600000,  // Queue หมดอายุหลัง 1 ชั่วโมง
        'x-message-ttl': 1800000,  // Message หมดอายุหลัง 30 นาที
      },
    });
  }
}
```

---

## 10. Kubernetes Deployment สำหรับ RabbitMQ Cluster

```yaml
# rabbitmq-cluster.yaml
apiVersion: rabbitmq.com/v1beta1
kind: RabbitmqCluster
metadata:
  name: rabbitmq-production
  namespace: messaging
spec:
  replicas: 3
  image: rabbitmq:3.12-management
  
  service:
    type: ClusterIP
    
  persistence:
    storageClassName: standard
    storage: 20Gi
    
  resources:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: 2000m
      memory: 2Gi
      
  rabbitmq:
    additionalConfig: |
      cluster_formation.peer_discovery_backend = rabbit_peer_discovery_k8s
      cluster_formation.k8s.host = kubernetes.default.svc.cluster.local
      cluster_formation.k8s.address_type = hostname
      vm_memory_high_watermark_paging_ratio = 0.5
      vm_memory_high_watermark.relative = 0.6
      disk_free_limit.relative = 1.0
      collect_statistics_interval = 10000
      
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app.kubernetes.io/name: rabbitmq-production
          topologyKey: kubernetes.io/hostname
          
  override:
    statefulSet:
      spec:
        template:
          spec:
            containers:
              - name: rabbitmq
                env:
                  - name: RABBITMQ_SERVER_ADDITIONAL_ERL_ARGS
                    value: "-rabbit consumer_timeout 36000000"
```

---

## 11. Prometheus Monitoring

```yaml
# prometheus-rabbitmq-config.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: rabbitmq-monitor
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: rabbitmq
  endpoints:
    - port: prometheus
      interval: 15s
      path: /metrics
---
# prometheus-rules-rabbitmq.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: rabbitmq-alerts
  namespace: monitoring
spec:
  groups:
    - name: rabbitmq
      interval: 30s
      rules:
        - alert: RabbitMQHighQueueMessages
          expr: rabbitmq_queue_messages > 10000
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "RabbitMQ queue has too many messages"
            description: "Queue {{ $labels.queue }} has {{ $value }} messages"
            
        - alert: RabbitMQConsumerCountLow
          expr: rabbitmq_queue_consumers == 0
          for: 1m
          labels:
            severity: critical
          annotations:
            summary: "RabbitMQ queue has no consumers"
            description: "Queue {{ $labels.queue }} has no active consumers"
            
        - alert: RabbitMQConnectionsHigh
          expr: rabbitmq_connections > 1000
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "RabbitMQ has too many connections"
```

---

## สรุป

| Pattern | ประโยชน์ | Use Case |
|---------|---------|---------|
| Direct Exchange | Routing ตาม key ที่แน่นอน | Order processing, routing by type |
| Fanout Exchange | Broadcast ไปทุก subscriber | Notifications, cache invalidation |
| Topic Exchange | Pattern matching routing | Event routing แบบ flexible |
| Headers Exchange | Route ตาม message headers | Complex routing rules |
| Dead Letter Queue | จัดการ failed messages | Error handling, auditing |
| Priority Queue | จัดลำดับความสำคัญ | Urgent vs normal requests |
| RPC Pattern | Request-Reply แบบ async | Cross-service communication |
| Publisher Confirms | Guaranteed delivery | Critical messages |
| Competing Consumers | Load balancing | High throughput processing |
| Message TTL | Auto-expire messages | Session data, real-time events |
